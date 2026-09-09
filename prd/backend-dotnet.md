# Product Requirements Document (PRD) — Backend
## Sales Point (VisioNet Mini ATM) — .NET API

**Version:** 1.0.0 | **Last Updated:** 2026-09-04 | **Status:** Draft

---

## 1. Overview

### 1.1 Purpose
This document defines the backend API specification and business logic for the **Sales Point** mobile web application — a field-agent tool used by VisioNet Mini ATM sales agents to log daily attendance (clock-in/out), record store acquisition prospects with photo documentation, and track performance.

### 1.2 Tech Stack
| Layer | Technology |
|---|---|
| Framework | .NET 8 Web API |
| Language | C# 12 |
| ORM | Entity Framework Core 8 |
| Database | SQL Server / PostgreSQL |
| Auth | JWT Bearer Tokens |
| File Storage | Local / Azure Blob / AWS S3 (abstracted) |
| Logging | Serilog |
| Validation | FluentValidation |
| Docs | Swagger / OpenAPI 3.0 |

### 1.3 Architecture Pattern
Clean Architecture — **Controllers → Application Services → Repositories / Domain**.

---

## 2. Authentication & Authorization

### 2.1 Auth Flow
- **Method:** JWT Bearer Token
- **Token Lifetime:** Access = 8 hours, Refresh = 7 days
- **Claim Payload:** `agentId`, `username`, `fullName`, `role`, `branchId`
- **Header:** `Authorization: Bearer {token}`

### 2.2 Roles
| Role | Scope |
|---|---|
| `Agent` | Own data only (attendance, prospects, profile) |
| `Supervisor` | View agents in assigned branch, manage notifications |
| `Admin` | Full system access |

### 2.3 Endpoints — Auth

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | POST | `/api/auth/login` | No | Authenticate agent credentials |
| 2 | POST | `/api/auth/refresh` | Refresh Token | Refresh expired access token |
| 3 | POST | `/api/auth/logout` | Bearer | Revoke refresh token |
| 4 | GET | `/api/auth/me` | Bearer | Get current authenticated user profile |

### 2.4 DTOs

```csharp
// LoginRequestDto
public record LoginRequestDto
{
    [Required] public string Username { get; init; }
    [Required] public string Password { get; init; }
    public bool RememberMe { get; init; }
}

// LoginResponseDto
public record LoginResponseDto(
    string AccessToken,
    string RefreshToken,
    DateTime ExpiresAt,
    AgentProfileDto Profile
);

// AgentProfileDto
public record AgentProfileDto(
    Guid AgentId,
    string Username,
    string FullName,
    string Email,
    string ReferralCode,
    string Role,
    string BranchName,
    string Status
);
```

### 2.5 Business Rules
- Login attempt rate limit: **5 failed attempts per 5 minutes** per IP+username, then lock for 15 minutes.
- Passwords hashed using **BCrypt** (cost 12).
- Refresh tokens stored in DB with `RevokedAt` nullable; rotation on every refresh.

---

## 3. Controllers & API Endpoints

### 3.1 Endpoint Summary

| Controller | Route Prefix | Endpoints |
|---|---|---|
| `AuthController` | `/api/auth` | login, refresh, logout, me |
| `AttendanceController` | `/api/attendance` | clock-in, clock-out, today, history, history-older |
| `ProspectController` | `/api/prospects` | create, list, detail, search, older |
| `PhotoController` | `/api/photos` | upload, download, download-all |
| `NotificationController` | `/api/notifications` | list, mark-read, mark-all-read, unread-count |
| `ProfileController` | `/api/profile` | get, referral, monthly-stats |
| `ReferenceController` | `/api/reference` | branches, shifts |

---

### 3.2 AttendanceController

#### Endpoints

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | POST | `/api/attendance/clock-in` | Bearer | Record clock-in with GPS + selfie ref |
| 2 | POST | `/api/attendance/clock-out` | Bearer | Record clock-out with GPS |
| 3 | GET | `/api/attendance/today` | Bearer | Get today's attendance record |
| 4 | GET | `/api/attendance/history?page=1&size=10` | Bearer | Paginated attendance history |
| 5 | GET | `/api/attendance/history/older?month={yyyy-MM}` | Bearer | Older month records |

#### DTOs

```csharp
public record ClockInRequestDto(
    [Required] decimal Latitude,
    [Required] decimal Longitude,
    [Required] decimal AccuracyMeters,
    [Required] string BranchName,
    string Address,
    string SelfiePhotoId  // references uploaded photo
);

public record ClockOutRequestDto(
    [Required] decimal Latitude,
    [Required] decimal Longitude,
    [Required] decimal AccuracyMeters
);

public record AttendanceDto(
    Guid Id,
    DateTime? ClockInTime,
    DateTime? ClockOutTime,
    string BranchName,
    string Address,
    decimal? Latitude,
    decimal? Longitude,
    decimal? AccuracyMeters,
    string Method,       // "GPS + Selfie"
    string Status,       // "Tepat waktu", "Terlambat", "Izin"
    TimeSpan? Duration,
    string ShiftStart,
    string ShiftEnd
);

public record AttendanceListDto(
    List<AttendanceDto> Items,
    int TotalCount,
    int Page,
    int PageSize
);
```

#### Business Rules
- One clock-in per agent per calendar day; reject duplicates with `409 Conflict`.
- Clock-in time vs shift start (`08:00`) determines status:
  - `< 08:00` → `On Time` / `Tepat Waktu`
  - `>= 08:00` → `Late` / `Terlambat` (store delta minutes)
- Clock-out only allowed if clock-in exists for today; reject if already clocked out (`409`).
- GPS geofence: reject clock-in if agent distance > **500m** from assigned branch coordinates (return `400` with `ERR_GEOFENCE_VIOLATION`).
- Duration auto-calculated on clock-out: `ClockOutTime - ClockInTime`.

---

### 3.3 ProspectController

#### Endpoints

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | POST | `/api/prospects` | Bearer | Create new prospect visit |
| 2 | GET | `/api/prospects/today` | Bearer | List today's prospects |
| 3 | GET | `/api/prospects?page=1&size=10` | Bearer | Paginated list |
| 4 | GET | `/api/prospects/search?q={query}` | Bearer | Search by name/address/PIC |
| 5 | GET | `/api/prospects/{id}` | Bearer | Prospect detail with photos |
| 6 | GET | `/api/prospects/older?day={date}` | Bearer | Prospects from prior dates |

#### DTOs

```csharp
public record CreateProspectRequestDto(
    [Required] string StoreName,
    [Required] string Address,
    [Required] string PicName,
    [Required][Phone] string PicPhone,
    [Required] string PhotoPlangId,   // uploaded photo ID
    [Required] string PhotoSelfieId,  // uploaded photo ID
    string? Notes,
    decimal? Latitude,
    decimal? Longitude,
    decimal? AccuracyMeters
);

public record ProspectDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string PicPhone,
    string VisitTime,    // "HH:mm"
    DateTime VisitDate,
    string? Notes,
    string Status,       // "Terverifikasi", "Menunggu verifikasi", "Ditolak"
    PhotoDto PhotoPlang,
    PhotoDto PhotoSelfie,
    string Coordinates   // "-6.26412, 106.79931"
);

public record ProspectListItemDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string VisitTime,
    DateTime VisitDate,
    string Status,
    string StatusClass   // "good", "warn", "crit"
);

public record ProspectListDto(
    List<ProspectGroupDto> Groups,
    int TotalCount
);

public record ProspectGroupDto(
    string DateLabel,    // "Hari Ini — Jumat, 4 September 2026"
    List<ProspectListItemDto> Items
);
```

#### Business Rules
- Prospect creation **requires** clock-in to exist for the current day (`403 Forbidden` if not clocked in).
- `PicPhone` validated: numeric, **min 9 digits**, normalize to `08xxxxxxxxxx`.
- Both photo IDs (`PhotoPlangId`, `PhotoSelfieId`) must exist and belong to the uploading agent (`400` if missing).
- New prospects default to status `Menunggu verifikasi`.
- Duplicate detection: same `StoreName + Address` within same day → warn but allow (`409 Conflict` optional).
- Auto-stamp photo metadata (GPS coordinates, timestamp) on upload.

---

### 3.4 PhotoController

#### Endpoints

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | POST | `/api/photos` | Bearer | Upload photo (multipart/form-data) |
| 2 | GET | `/api/photos/{id}` | Bearer | Download photo (returns image/jpeg) |
| 3 | GET | `/api/photos/{id}/metadata` | Bearer | Get photo metadata only |
| 4 | GET | `/api/photos/prospect/{prospectId}/all` | Bearer | Get all photos for a prospect |

#### DTOs

```csharp
public record PhotoUploadRequestDto(
    IFormFile File,
    string PhotoType,    // "plang" | "selfie"
    decimal? Latitude,
    decimal? Longitude,
    decimal? AccuracyMeters
);

public record PhotoDto(
    Guid Id,
    string Url,
    string PhotoType,
    DateTime CapturedAt,
    string? Latitude,
    string? Longitude,
    string? Address,
    string FileName
);
```

#### Business Rules
- Max file size: **10 MB**.
- Allowed types: `image/jpeg`, `image/png`, `image/webp`.
- Store original + generate thumbnail (320px width) for list views.
- Inject metadata watermark (timestamp + GPS) server-side on upload.
- Files stored in abstracted `IPhotoStorage` (local disk / blob / S3).

---

### 3.5 NotificationController

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | GET | `/api/notifications` | Bearer | Grouped notification list |
| 2 | POST | `/api/notifications/{id}/read` | Bearer | Mark single as read |
| 3 | POST | `/api/notifications/read-all` | Bearer | Mark all as read |
| 4 | GET | `/api/notifications/unread-count` | Bearer | Badge count only |

#### DTOs

```csharp
public record NotificationDto(
    Guid Id,
    string Title,
    string Body,
    string IconType,   // "green", "orange", "blue", "red"
    DateTime CreatedAt,
    string TimeLabel,  // "12:05 WIB"
    bool IsRead
);

public record NotificationGroupDto(
    string DateLabel,  // "Hari Ini", "Kemarin", or full date
    List<NotificationDto> Items
);
```

#### Business Rules
- Notifications auto-generated on events:
  - Prospect created → `green` "Prospek tersimpan"
  - Clock-in recorded → `blue` "Clock in tercatat"
  - Approaching clock-out deadline (16:00 WIB) → `orange` "Jangan lupa clock out"
  - Prospect verified/rejected by supervisor → `green`/`red`
- Grouped by date: Today, Yesterday, then full dates.

---

### 3.6 ProfileController

| # | Method | Endpoint | Auth | Description |
|---|---|---|---|---|
| 1 | GET | `/api/profile` | Bearer | Full agent profile |
| 2 | GET | `/api/profile/referral` | Bearer | Referral code + copy |
| 3 | GET | `/api/profile/monthly-stats?year=2026` | Bearer | Bar chart data |
| 4 | GET | `/api/profile/summary` | Bearer | Current month prospek + attendance % |

#### DTOs

```csharp
public record ProfileDetailDto(
    Guid AgentId,
    string Username,
    string FullName,
    string Email,
    string ReferralCode,
    string Role,
    string Status,
    string BranchName,
    string AvatarInitials
);

public record MonthlyStatItemDto(
    string Month,       // "Jan", "Feb", ...
    int Count,
    bool IsCurrentMonth
);

public record MonthlyStatsDto(
    List<MonthlyStatItemDto> Months,
    int TotalProspects,
    int CurrentMonthProspects,
    decimal MonthOverMonthChange  // +12%, -5%, etc.
);

public record ProfileSummaryDto(
    int CurrentMonthProspects,
    decimal AttendanceRate,      // percentage
    int AttendanceDays,
    string Trend                 // "+12%"
);
```

#### Business Rules
- Referral code format: `SP-{INITIALS}{AGENT_NUMBER_4_DIGITS}` (e.g., `SP-RZK2041`), generated once at agent registration.
- Monthly stats aggregated from `Prospects` table grouped by month; current month flagged `IsCurrentMonth = true`.

---

## 4. Service Layer

### 4.1 Service Interfaces

```csharp
public interface IAuthService
{
    Task<LoginResponseDto> LoginAsync(LoginRequestDto req, string ipAddress);
    Task<LoginResponseDto> RefreshAsync(string refreshToken);
    Task LogoutAsync(Guid agentId);
    Task<AgentProfileDto> GetMeAsync(Guid agentId);
}

public interface IAttendanceService
{
    Task<AttendanceDto> ClockInAsync(Guid agentId, ClockInRequestDto req);
    Task<AttendanceDto> ClockOutAsync(Guid agentId, ClockOutRequestDto req);
    Task<AttendanceDto?> GetTodayAsync(Guid agentId);
    Task<AttendanceListDto> GetHistoryAsync(Guid agentId, int page, int size);
    Task<AttendanceListDto> GetOlderHistoryAsync(Guid agentId, string month);
}

public interface IProspectService
{
    Task<ProspectDto> CreateAsync(Guid agentId, CreateProspectRequestDto req);
    Task<List<ProspectListItemDto>> GetTodayAsync(Guid agentId);
    Task<ProspectListDto> GetListAsync(Guid agentId, int page, int size);
    Task<ProspectListDto> SearchAsync(Guid agentId, string query);
    Task<ProspectDto> GetByIdAsync(Guid agentId, Guid prospectId);
    Task<ProspectListDto> GetOlderAsync(Guid agentId, DateTime date);
}

public interface IPhotoService
{
    Task<PhotoDto> UploadAsync(Guid agentId, PhotoUploadRequestDto req);
    Task<(byte[] Data, string ContentType)> DownloadAsync(Guid agentId, Guid photoId);
    Task<List<PhotoDto>> GetByProspectAsync(Guid agentId, Guid prospectId);
}

public interface INotificationService
{
    Task<List<NotificationGroupDto>> GetAllAsync(Guid agentId);
    Task MarkReadAsync(Guid agentId, Guid notificationId);
    Task MarkAllReadAsync(Guid agentId);
    Task<int> GetUnreadCountAsync(Guid agentId);
    Task CreateAsync(Guid agentId, string title, string body, string iconType);
}

public interface IProfileService
{
    Task<ProfileDetailDto> GetProfileAsync(Guid agentId);
    Task<MonthlyStatsDto> GetMonthlyStatsAsync(Guid agentId, int year);
    Task<ProfileSummaryDto> GetSummaryAsync(Guid agentId);
}
```

### 4.2 Key Business Logic

#### Attendance — Geofence Validation
```csharp
public async Task<AttendanceDto> ClockInAsync(Guid agentId, ClockInRequestDto req)
{
    // 1. Check if already clocked in today
    var existing = await _repo.GetTodayAsync(agentId);
    if (existing?.ClockInTime != null)
        throw new ConflictException("Already clocked in today.");

    // 2. Validate geofence (distance to branch)
    var branch = await _branchRepo.GetByAgentAsync(agentId);
    var distance = GeoHelper.HaversineDistance(
        req.Latitude, req.Longitude,
        branch.Latitude, branch.Longitude);
    
    if (distance > 500m)
        throw new BadRequestException("ERR_GEOFENCE_VIOLATION",
            $"You are {distance:F0}m from your branch. Max allowed: 500m.");

    // 3. Determine status based on shift
    var clockInTime = DateTime.UtcNow.ToWIB();
    var shiftStart = TimeSpan.Parse("08:00");
    var isLate = clockInTime.TimeOfDay > shiftStart;
    var status = isLate
        ? $"Terlambat {(clockInTime.TimeOfDay - shiftStart).TotalMinutes:F0} m"
        : "Tepat waktu";

    // 4. Save
    var attendance = new Attendance { /* ... */ };
    await _repo.AddAsync(attendance);

    // 5. Auto-create notification
    await _notifService.CreateAsync(agentId, "Clock in tercatat",
        $"Absen masuk pukul {clockInTime:HH:mm} WIB di {req.BranchName} berhasil disimpan.",
        "blue");

    return MapToDto(attendance);
}
```

#### Prospect — Creation Guard
```csharp
public async Task<ProspectDto> CreateAsync(Guid agentId, CreateProspectRequestDto req)
{
    // Must be clocked in today
    var today = await _attendanceService.GetTodayAsync(agentId);
    if (today?.ClockInTime == null)
        throw new ForbiddenException("ERR_NOT_CLOCKED_IN",
            "You must clock in before creating prospects.");

    // Validate photos belong to agent
    await _photoService.ValidateOwnershipAsync(agentId, req.PhotoPlangId, req.PhotoSelfieId);

    // Normalize phone
    var normalizedPhone = PhoneNormalizer.Normalize(req.PicPhone);

    // Create
    var prospect = new Prospect { /* ... */ };
    await _repo.AddAsync(prospect);

    // Auto notification
    await _notifService.CreateAsync(agentId, "Prospek tersimpan",
        $"Data {req.StoreName} berhasil masuk ke sistem Mini ATM.", "green");

    return MapToDto(prospect);
}
```

---

## 5. Data Model (EF Core Entities)

```csharp
public class Agent
{
    public Guid Id { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; }
    public string FullName { get; set; }
    public string Email { get; set; }
    public string ReferralCode { get; set; }
    public string Role { get; set; }        // "Agent", "Supervisor", "Admin"
    public string Status { get; set; }      // "Aktif", "Nonaktif"
    public Guid BranchId { get; set; }
    public Branch? Branch { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LockedUntil { get; set; }
    public int FailedLoginAttempts { get; set; }

    public ICollection<Attendance> Attendances { get; set; }
    public ICollection<Prospect> Prospects { get; set; }
    public ICollection<RefreshToken> RefreshTokens { get; set; }
}

public class Branch
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
    public decimal Latitude { get; set; }
    public decimal Longitude { get; set; }
    public string ShiftStart { get; set; }   // "08:00"
    public string ShiftEnd { get; set; }     // "17:00"
    public ICollection<Agent> Agents { get; set; }
}

public class Attendance
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public DateTime Date { get; set; }
    public DateTime? ClockInTime { get; set; }
    public DateTime? ClockOutTime { get; set; }
    public decimal? Latitude { get; set; }
    public decimal? Longitude { get; set; }
    public decimal? AccuracyMeters { get; set; }
    public string BranchName { get; set; }
    public string Address { get; set; }
    public string Method { get; set; }       // "GPS + Selfie"
    public string Status { get; set; }
    public TimeSpan? Duration { get; set; }
    public Agent? Agent { get; set; }
}

public class Prospect
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string StoreName { get; set; }
    public string Address { get; set; }
    public string PicName { get; set; }
    public string PicPhone { get; set; }
    public DateTime VisitDate { get; set; }
    public string VisitTime { get; set; }    // "HH:mm"
    public string? Notes { get; set; }
    public string Status { get; set; }        // "Menunggu verifikasi", "Terverifikasi", "Ditolak"
    public decimal? Latitude { get; set; }
    public decimal? Longitude { get; set; }
    public Guid? PhotoPlangId { get; set; }
    public Guid? PhotoSelfieId { get; set; }
    public Photo? PhotoPlang { get; set; }
    public Photo? PhotoSelfie { get; set; }
    public Agent? Agent { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class Photo
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string FileName { get; set; }
    public string FilePath { get; set; }
    public string ThumbnailPath { get; set; }
    public string PhotoType { get; set; }    // "plang", "selfie"
    public DateTime CapturedAt { get; set; }
    public decimal? Latitude { get; set; }
    public decimal? Longitude { get; set; }
    public string? Address { get; set; }
    public Agent? Agent { get; set; }
}

public class Notification
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string Title { get; set; }
    public string Body { get; set; }
    public string IconType { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ReadAt { get; set; }
    public Agent? Agent { get; set; }
}

public class RefreshToken
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string Token { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime? RevokedAt { get; set; }
    public string? ReplacedByToken { get; set; }
    public string? IpAddress { get; set; }
}
```

---

## 6. API Response Standards

### 6.1 Success Response
```json
{
  "success": true,
  "data": { ... },
  "message": null
}
```

### 6.2 Error Response
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "ERR_GEOFENCE_VIOLATION",
    "message": "You are 1200m from your branch. Max allowed: 500m."
  }
}
```

### 6.3 Error Codes

| Code | HTTP | Description |
|---|---|---|
| `ERR_INVALID_CREDENTIALS` | 401 | Wrong username/password |
| `ERR_ACCOUNT_LOCKED` | 423 | Too many failed attempts |
| `ERR_TOKEN_EXPIRED` | 401 | Access token expired |
| `ERR_TOKEN_INVALID` | 401 | Malformed/revoked token |
| `ERR_NOT_CLOCKED_IN` | 403 | Prospect creation without clock-in |
| `ERR_GEOFENCE_VIOLATION` | 400 | Agent too far from branch |
| `ERR_ALREADY_CLOCKED_IN` | 409 | Duplicate clock-in |
| `ERR_ALREADY_CLOCKED_OUT` | 409 | Duplicate clock-out |
| `ERR_NOT_CLOCKED_OUT` | 400 | Clock-out without clock-in |
| `ERR_PHOTO_MISSING` | 400 | Required photos not provided |
| `ERR_PHOTO_TOO_LARGE` | 413 | File exceeds 10MB |
| `ERR_VALIDATION` | 422 | Model validation failed |
| `ERR_NOT_FOUND` | 404 | Resource not found |

---

## 7. Cross-Cutting Concerns

### 7.1 Middleware Pipeline (order matters)
1. ExceptionHandlerMiddleware (global try-catch → standardized error JSON)
2. SerilogRequestLogging
3. RateLimitingMiddleware (per-IP + per-user)
4. JwtAuthenticationMiddleware
5. AuthorizationMiddleware
6. Swagger (Dev only)
7. EndpointRouting
8. Controllers

### 7.2 Rate Limiting

| Endpoint Group | Limit | Window |
|---|---|---|
| `/api/auth/login` | 5 | 5 minutes per IP+username |
| All authenticated endpoints | 100 | 1 minute per agent |
| Photo upload | 20 | 1 minute per agent |

### 7.3 Validation
- All DTOs validated via **FluentValidation**.
- `CreateProspectRequestDto`:
  - `StoreName`: not empty, max 200
  - `Address`: not empty, max 500
  - `PicName`: not empty, max 100
  - `PicPhone`: numeric, 9–15 digits
  - `PhotoPlangId`, `PhotoSelfieId`: required, must exist

### 7.4 Timezone
- All timestamps stored in **UTC**.
- API accepts/returns **WIB (UTC+7)** formatted strings for display: `HH:mm WIB`, `dd MMMM yyyy`.
- Helper: `DateTimeExtensions.ToWIB()`.

### 7.5 Pagination
- Query params: `?page=1&size=10`
- Response includes: `Items`, `TotalCount`, `Page`, `PageSize`
- Max page size: 50.

---

## 8. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Response time (p95) | < 300ms for list endpoints, < 150ms for single resource |
| Photo upload (10MB) | < 5 seconds |
| Availability | 99.9% during business hours (06:00–20:00 WIB) |
| Database connection pooling | Min 5, Max 50 |
| Concurrent agents | 500 active sessions |
| Photo storage retention | 2 years |
| Audit log | All clock-in/out, prospect create/delete, login events |

---

## 9. Configuration (`appsettings.json`)

```json
{
  "Jwt": {
    "Issuer": "SalesPoint.API",
    "Audience": "SalesPoint.App",
    "AccessTokenMinutes": 480,
    "RefreshTokenDays": 7,
    "Secret": "{from-secrets}"
  },
  "Geofence": {
    "MaxDistanceMeters": 500,
    "ShiftStart": "08:00",
    "ShiftEnd": "17:00",
    "ClockOutReminderHour": 16
  },
  "PhotoStorage": {
    "Provider": "Blob",
    "MaxFileSizeMB": 10,
    "AllowedTypes": ["image/jpeg", "image/png", "image/webp"],
    "ThumbnailWidth": 320
  },
  "RateLimit": {
    "LoginMaxAttempts": 5,
    "LoginWindowMinutes": 5,
    "LockDurationMinutes": 15,
    "ApiPerMinute": 100,
    "UploadPerMinute": 20
  },
  "ConnectionStrings": {
    "DefaultConnection": "{from-secrets}"
  }
}
```

---

## 10. Future Considerations (Out of Scope v1.0)
- Push notifications (FCM/APNS) for real-time alerts
- Offline sync (PWA service worker / mobile SDK)
- Supervisor dashboard API (approve/reject prospects)
- Bulk prospect export (Excel/CSV)
- Agent leaderboard & gamification
- Integration with external CRM (Salesforce/HubSpot)

---

**End of Document**