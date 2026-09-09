# Product Requirements Document (PRD) — Backend
## Sales Point — VisioNet Mini ATM Agent Acquisition System

**Document Version:** 1.0.0  
**Platform:** .NET 8 Web API  
**Architecture:** Clean Architecture / N-Tier  
**Last Updated:** 2026-09-04

---

## 1. Overview

### 1.1 Purpose
The Sales Point backend is a .NET 8 Web API that powers a mobile field application used by VisioNet Mini ATM sales agents. It handles agent authentication, daily attendance (clock in/out) with GPS validation, prospect store acquisition (with photo documentation), notifications, and performance analytics.

### 1.2 Scope
This PRD covers ONLY backend concerns:
- REST API endpoints
- Controllers, Services, Repositories
- Data Transfer Objects (DTOs)
- Authentication & Authorization
- Business logic & validation rules
- Data persistence and file storage

### 1.3 Out of Scope
- Frontend implementation (mobile app)
- Infrastructure provisioning (handled by DevOps)
- Third-party integrations with core banking (future phase)

---

## 2. System Architecture

### 2.1 Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | .NET 8 LTS |
| Web Framework | ASP.NET Core Web API |
| ORM | Entity Framework Core 8 |
| Database | PostgreSQL 16 |
| Cache | Redis 7 |
| Auth | JWT Bearer + Refresh Tokens |
| File Storage | AWS S3 / MinIO |
| Logging | Serilog → Elasticsearch |
| API Docs | Swashbuckle (OpenAPI 3) |
| Validation | FluentValidation |
| Mapping | AutoMapper |

### 2.2 Solution Structure

```
SalesPoint.sln
├── src/
│   ├── SalesPoint.Api/              → Controllers, Middleware, DI
│   ├── SalesPoint.Application/     → Services, DTOs, Validators, Interfaces
│   ├── SalesPoint.Domain/          → Entities, Enums, Domain Events
│   ├── SalesPoint.Infrastructure/ → EF Core, Repositories, External Services
│   └── SalesPoint.Shared/          → Common utilities, Constants, Exceptions
├── tests/
│   ├── SalesPoint.UnitTests/
│   └── SalesPoint.IntegrationTests/
```

### 2.3 Layered Dependencies

```
Api → Application → Domain
Api → Infrastructure → Application → Domain
Infrastructure → Domain (EF Core configurations)
```

---

## 3. Domain Entities

### 3.1 Agent (User)

```csharp
public class Agent : BaseEntity
{
    public Guid Id { get; set; }
    public string Username { get; set; }          // unique, max 50
    public string PasswordHash { get; set; }
    public string FullName { get; set; }           // max 150
    public string Email { get; set; }              // unique
    public string PhoneNumber { get; set; }        // E.164 format
    public string ReferralCode { get; set; }       // unique, format: SP-XXX####  (e.g., SP-RZK2041)
    public Guid BranchId { get; set; }             // FK → Branch
    public AgentStatus Status { get; set; }        // Active, Suspended, Inactive
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? LastLoginAt { get; set; }

    // Navigation
    public Branch Branch { get; set; }
    public ICollection<Attendance> Attendances { get; set; }
    public ICollection<Prospect> Prospects { get; set; }
    public ICollection<Notification> Notifications { get; set; }
    public ICollection<RefreshToken> RefreshTokens { get; set; }
}
```

### 3.2 Branch

```csharp
public class Branch : BaseEntity
{
    public Guid Id { get; set; }
    public string Name { get; set; }               // e.g., "Kantor Cabang Jakarta Selatan"
    public string Address { get; set; }
    public decimal Latitude { get; set; }
    public decimal Longitude { get; set; }
    public decimal GeoFenceRadiusMeters { get; set; } = 200;
    public bool IsActive { get; set; }
}
```

### 3.3 Attendance

```csharp
public class Attendance : BaseEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public DateOnly ShiftDate { get; set; }         // local date
    public ShiftType ShiftType { get; set; }       // Reguler, Shift_1, Shift_2
    public TimeOnly ExpectedIn { get; set; }       // e.g., 08:00
    public TimeOnly ExpectedOut { get; set; }      // e.g., 17:00

    public DateTimeOffset? ClockInAt { get; set; }
    public decimal? ClockInLatitude { get; set; }
    public decimal? ClockInLongitude { get; set; }
    public decimal? ClockInAccuracyMeters { get; set; }
    public string? ClockInPhotoUrl { get; set; }   // selfie at clock-in
    public Guid? ClockInBranchId { get; set; }

    public DateTimeOffset? ClockOutAt { get; set; }
    public decimal? ClockOutLatitude { get; set; }
    public decimal? ClockOutLongitude { get; set; }
    public decimal? ClockOutAccuracyMeters { get; set; }
    public string? ClockOutPhotoUrl { get; set; }
    public Guid? ClockOutBranchId { get; set; }

    public AttendanceStatus Status { get; set; }  // Pending, Present, Late, Absent, OnLeave
    public string? Notes { get; set; }
}
```

### 3.4 Prospect (Store Acquisition)

```csharp
public class Prospect : BaseEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string StoreName { get; set; }          // max 200
    public string Address { get; set; }            // max 500
    public string PicName { get; set; }            // PIC = Person in Charge
    public string PicPhoneNumber { get; set; }     // E.164
    public decimal Latitude { get; set; }
    public decimal Longitude { get; set; }
    public decimal GeoAccuracyMeters { get; set; }
    public string? Notes { get; set; }             // optional, max 1000
    public DateTimeOffset VisitedAt { get; set; }
    public ProspectStatus Status { get; set; }     // PendingVerification, Verified, Rejected

    // Photos
    public string PlangPhotoUrl { get; set; }      // storefront sign photo
    public string SelfiePhotoUrl { get; set; }     // selfie with PIC

    // Verification
    public Guid? VerifiedBy { get; set; }
    public DateTimeOffset? VerifiedAt { get; set; }
    public string? RejectionReason { get; set; }
}
```

### 3.5 Notification

```csharp
public class Notification : BaseEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public NotificationType Type { get; set; }     // ProspectSaved, ClockInReminder, ClockInRecorded, ScheduleVisit, BonusDisbursed, AppUpdate, Briefing
    public NotificationIconColor IconColor { get; set; }  // Green, Orange, Blue, Red
    public string Title { get; set; }
    public string Body { get; set; }
    public bool IsRead { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? ReadAt { get; set; }
}
```

### 3.6 RefreshToken

```csharp
public class RefreshToken : BaseEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string TokenHash { get; set; }          // SHA-256 hashed
    public DateTimeOffset ExpiresAt { get; set; }
    public DateTimeOffset? RevokedAt { get; set; }
    public string? ReplacedByToken { get; set; }
    public string? RemoteIp { get; set; }
    public string? UserAgent { get; set; }
}
```

### 3.7 Enums

```csharp
public enum AgentStatus { Active, Suspended, Inactive }
public enum ShiftType { Reguler, Shift_1, Shift_2 }
public enum AttendanceStatus { Pending, Present, Late, Absent, OnLeave }
public enum ProspectStatus { PendingVerification, Verified, Rejected }
public enum NotificationType { ProspectSaved, ClockInReminder, ClockInRecorded, ScheduleVisit, BonusDisbursed, AppUpdate, Briefing }
public enum NotificationIconColor { Green, Orange, Blue, Red }
```

---

## 4. Authentication & Authorization

### 4.1 Authentication Mechanism

| Aspect | Value |
|--------|-------|
| Token Type | JWT (access) + Refresh Token |
| Access Token TTL | 15 minutes |
| Refresh Token TTL | 7 days |
| Algorithm | HS256 |
| Claim `sub` | AgentId (GUID) |
| Claim `username` | Agent username |
| Claim `branch_id` | BranchId |
| Claim `role` | "Agent" (future: "Supervisor", "Admin") |

### 4.2 Password Policy
- Hashing: **BCrypt** with work factor 12
- Min length: 8 characters
- Lockout: 5 failed attempts → 15-minute lock

### 4.3 Authorization Policies

```csharp
// Policy: "AgentOnly" → requires role "Agent"
// Policy: "SupervisorOnly" → requires role "Supervisor"
// Policy: "AdminOnly" → requires role "Admin"
```

Resource-level authorization: agents can only access their own attendance, prospects, and notifications.

---

## 5. API Endpoints

Base URL: `/api/v1`

### 5.1 Authentication

| # | Method | Endpoint | Description | Auth |
|---|--------|----------|-------------|------|
| 1.1 | POST | `/auth/login` | Agent login | No |
| 1.2 | POST | `/auth/refresh` | Refresh access token | Refresh Token |
| 1.3 | POST | `/auth/logout` | Revoke refresh token | Bearer |
| 1.4 | GET | `/auth/me` | Get current agent profile | Bearer |

#### 5.1.1 POST `/auth/login`

**Request DTO:**
```csharp
public record LoginRequest(
    [property: Required] string Username,
    [property: Required] string Password,
    bool RememberMe
);
```

**Response DTO:**
```csharp
public record LoginResponse(
    string AccessToken,
    DateTimeOffset ExpiresAt,
    string RefreshToken,
    AgentProfileDto Profile
);

public record AgentProfileDto(
    Guid Id,
    string Username,
    string FullName,
    string Email,
    string PhoneNumber,
    string ReferralCode,
    string BranchName,
    string Status,
    bool HasClockedInToday,
    bool HasClockedOutToday,
    int TodayProspectCount
);
```

**Business Logic:**
- Validate credentials against BCrypt hash
- Check `AgentStatus == Active`
- Enforce lockout policy
- Generate JWT + refresh token (stored hashed in DB)
- Update `LastLoginAt`
- Return profile summary including today's attendance state

**Errors:**
- `401 Unauthorized` — invalid credentials
- `423 Locked` — account locked

#### 5.1.2 POST `/auth/refresh`

```csharp
public record RefreshRequest(string RefreshToken);
public record RefreshResponse(string AccessToken, DateTimeOffset ExpiresAt, string NewRefreshToken);
```

Implements **refresh token rotation**: old token revoked, new issued.

### 5.2 Attendance

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 2.1 | GET | `/attendance/today` | Today's attendance state |
| 2.2 | GET | `/attendance/shift` | Today's shift info |
| 2.3 | POST | `/attendance/clock-in` | Clock in |
| 2.4 | POST | `/attendance/clock-out` | Clock out |
| 2.5 | GET | `/attendance/history` | Recent attendance (paginated) |
| 2.6 | GET | `/attendance/history/older` | Load older records |

#### 5.2.1 GET `/attendance/today`

**Response:**
```csharp
public record TodayAttendanceDto(
    DateOnly Date,
    string DayName,            // "Jumat"
    string ShiftName,          // "Reguler"
    TimeOnly ExpectedIn,
    TimeOnly ExpectedOut,
    TimeOnly? ClockInAt,
    TimeOnly? ClockOutAt,
    string? ClockInBranchName,
    string? ClockOutBranchName,
    decimal? ClockInAccuracy,
    string? ClockInPhotoUrl,
    AttendanceStatus Status,
    string? DurationText       // "5 jam 34 menit berjalan"
);
```

#### 5.2.2 POST `/attendance/clock-in`

**Request DTO:**
```csharp
public record ClockInRequest(
    decimal Latitude,
    decimal Longitude,
    decimal AccuracyMeters,
    IFormFile SelfiePhoto
);
```

**Response:**
```csharp
public record ClockInResponse(
    TimeOnly ClockInTime,
    string BranchName,
    decimal DistanceToBranchMeters,
    bool WithinGeoFence,
    AttendanceStatus Status     // Present or Late
);
```

**Business Logic:**
1. Reject if already clocked in today → `409 Conflict`
2. Determine branch by nearest geo-fence
3. Validate GPS accuracy ≤ 50 m; if > 50 m → `400 BadRequest` with message
4. Validate within geo-fence (≤ 200 m from branch). If outside → `422 UnprocessableEntity`
5. Determine status: `Present` if `ClockInAt ≤ ExpectedIn`, else `Late`
6. Save selfie to S3: `attendance/{agentId}/{yyyy-MM-dd}/clockin.jpg`
7. Persist attendance record
8. Create notification `ClockInRecorded`
9. Schedule `ClockInReminder` for 17:00 WIB (if not clocked out by then)

#### 5.2.3 POST `/attendance/clock-out`

**Request:**
```csharp
public record ClockOutRequest(
    decimal Latitude,
    decimal Longitude,
    decimal AccuracyMeters,
    IFormFile SelfiePhoto
);
```

**Business Logic:**
- Reject if not clocked in today → `409 Conflict`
- Reject if already clocked out → `409 Conflict`
- Same GPS validation as clock-in
- Update existing attendance record

#### 5.2.4 GET `/attendance/history`

**Query Parameters:**
- `page` (default 1)
- `pageSize` (default 5, max 20)

**Response:**
```csharp
public record AttendanceHistoryResponse(
    List<AttendanceHistoryItemDto> Items,
    bool HasOlder
);

public record AttendanceHistoryItemDto(
    DateOnly Date,
    string DayLabel,          // "Kamis, 3 September 2026"
    string DayShort,          // "Kam"
    string DateLabel,         // "03"
    string MonthShort,        // "Sep"
    TimeOnly? ClockIn,
    TimeOnly? ClockOut,
    string DurationText,     // "9j 18m" or "Izin sakit"
    string StatusLabel,       // "Tepat waktu", "Terlambat 12 m", "Izin"
    string StatusClass        // "good", "warn", "info"
);
```

### 5.3 Prospects

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 3.1 | GET | `/prospects/today` | Today's prospects |
| 3.2 | GET | `/prospects/history` | Prospects grouped by day (with search) |
| 3.3 | GET | `/prospects/history/older` | Load older day groups |
| 3.4 | GET | `/prospects/{id}` | Prospect detail |
| 3.5 | POST | `/prospects` | Create new prospect |
| 3.6 | GET | `/prospects/{id}/photos/{kind}` | Get photo (plang/selfie) |
| 3.7 | GET | `/prospects/monthly-stats` | Monthly chart data |

#### 5.3.1 GET `/prospects/today`

```csharp
public record ProspectListResponse(List<ProspectSummaryDto> Items);

public record ProspectSummaryDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string VisitedTimeLabel,    // "08:52"
    string ThumbnailUrl,        // plang photo URL or null
    string Initials             // "TB" if no thumbnail
);
```

#### 5.3.2 GET `/prospects/history`

**Query Parameters:**
- `search` (optional, matches store name/address/PIC)
- `page` (default 1, for pagination of day groups)

**Response:**
```csharp
public record ProspectHistoryResponse(
    List<ProspectDayGroupDto> Groups,
    bool HasOlder
);

public record ProspectDayGroupDto(
    string DayLabel,               // "Kamis, 3 September 2026"
    List<ProspectSummaryWithStatusDto> Items
);

public record ProspectSummaryWithStatusDto : ProspectSummaryDto
{
    public string StatusLabel { get; init; }   // "Terverifikasi", "Menunggu verifikasi", "Ditolak"
    public string StatusClass { get; init; }    // "good", "warn", "crit"
    public string PhoneNumber { get; init; }
    public string VisitDateLabel { get; init; }
}
```

#### 5.3.3 GET `/prospects/{id}`

```csharp
public record ProspectDetailDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string PicPhoneNumber,
    string Latitude,
    string Longitude,
    string VisitedTimeLabel,
    string VisitDateLabel,
    string PlangPhotoUrl,
    string SelfiePhotoUrl,
    string Notes,
    string StatusLabel,
    string StatusClass
);
```

#### 5.3.4 POST `/prospects`

**Request (multipart/form-data):**

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `StoreName` | string | Yes | 1–200 chars |
| `Address` | string | Yes | 1–500 chars |
| `PicName` | string | Yes | 1–100 chars |
| `PicPhoneNumber` | string | Yes | Min 9 digits, E.164 or local format |
| `Latitude` | decimal | Yes | -90 to 90 |
| `Longitude` | decimal | Yes | -180 to 180 |
| `GeoAccuracyMeters` | decimal | Yes | ≤ 50 m |
| `PlangPhoto` | IFormFile | Yes | image/jpeg or image/png, max 5 MB |
| `SelfiePhoto` | IFormFile | Yes | image/jpeg or image/png, max 5 MB |
| `Notes` | string | No | Max 1000 chars |

**Response:**
```csharp
public record CreateProspectResponse(
    Guid Id,
    string StoreName,
    string PicName,
    string PicPhoneNumber,
    string VisitedTimeLabel,
    string PhotoCountLabel     // "2 foto terlampir"
);
```

**Business Logic:**
1. Validate agent has clocked in today (unless supervisor override) → `403 Forbidden` if not
2. Validate all required fields via FluentValidation
3. Validate GPS accuracy ≤ 50 m
4. Validate photo content type and size
5. Upload photos to S3:
   - `prospects/{agentId}/{prospectId}/plang_{timestamp}.jpg`
   - `prospects/{agentId}/{prospectId}/selfie_{timestamp}.jpg`
6. Stamp photo metadata (date/time, coordinates) server-side using `ImageSharp`
7. Persist prospect with `Status = PendingVerification`
8. Create notification `ProspectSaved`
9. Update cached `TodayProspectCount`
10. Return created prospect summary

#### 5.3.5 GET `/prospects/monthly-stats`

Returns last 9 months of prospect counts for the profile chart.

```csharp
public record MonthlyStatsResponse(
    List<MonthlyStatDto> Months,
    int TotalCount
);

public record MonthlyStatDto(
    string MonthLabel,    // "Jan", "Feb", ..., "Sep"
    int Count,
    bool IsCurrentMonth
);
```

### 5.4 Notifications

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 4.1 | GET | `/notifications` | Grouped notifications |
| 4.2 | PUT | `/notifications/{id}/read` | Mark single as read |
| 4.3 | PUT | `/notifications/read-all` | Mark all as read |
| 4.4 | GET | `/notifications/unread-count` | Badge count |

#### 5.4.1 GET `/notifications`

```csharp
public record NotificationsResponse(
    List<NotificationDayGroupDto> Groups,
    int UnreadCount
);

public record NotificationDayGroupDto(
    string DayLabel,   // "Hari Ini", "Kemarin", "Rabu, 2 September 2026"
    List<NotificationItemDto> Items
);

public record NotificationItemDto(
    Guid Id,
    string IconColor,   // "green", "orange", "blue", "red"
    string Title,
    string Body,
    string TimeLabel,   // "12:05 WIB"
    bool IsRead
);
```

### 5.5 Profile / Agent

| # | Method | Endpoint | Description |
|---|--------|----------|-------------|
| 5.1 | GET | `/agents/profile` | Full profile |
| 5.2 | POST | `/agents/referral-code/copy` | Track referral copy event (analytics) |
| 5.3 | GET | `/agents/dashboard` | Home dashboard summary |

#### 5.5.1 GET `/agents/profile`

```csharp
public record AgentFullProfileDto(
    Guid Id,
    string Username,
    string FullName,
    string Email,
    string PhoneNumber,
    string ReferralCode,
    string StatusLabel,
    string RoleLabel,
    MonthlyStatsResponse MonthlyProspectStats,
    int CurrentMonthProspectCount,
    decimal MonthOverMonthChangePercent,   // +12%
    int AttendanceThisMonth,
    decimal AttendancePercent,
    string AttendanceStatusLabel           // "Baik"
);
```

#### 5.5.2 GET `/agents/dashboard`

```csharp
public record HomeDashboardDto(
    string AgentName,
    string ReferralCode,
    int TodayProspectCount,
    string TodayDateLabel,         // "Jumat, 4 September 2026"
    AttendanceWidgetDto Attendance,
    int UnreadNotificationCount
);

public record AttendanceWidgetDto(
    bool HasClockedIn,
    bool HasClockedOut,
    string? ClockInTimeLabel,
    string? ClockOutTimeLabel,
    string ButtonAction,   // "clock-in" | "clock-out" | "done"
    string ButtonLabel    // "Clock In Sekarang" | "Clock Out Sekarang"
);
```

---

## 6. Controllers

### 6.1 Controller List

```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class AuthController : ControllerBase { ... }

[ApiController]
[Authorize]
[Route("api/v1/attendance")]
public class AttendanceController : ControllerBase { ... }

[ApiController]
[Authorize]
[Route("api/v1/prospects")]
public class ProspectsController : ControllerBase { ... }

[ApiController]
[Authorize]
[Route("api/v1/notifications")]
public class NotificationsController : ControllerBase { ... }

[ApiController]
[Authorize]
[Route("api/v1/agents")]
public class AgentsController : ControllerBase { ... }
```

### 6.2 Example: ProspectsController

```csharp
[ApiController]
[Authorize]
[Route("api/v1/prospects")]
public class ProspectsController : ControllerBase
{
    private readonly IProspectService _prospectService;
    private readonly ICurrentUserService _currentUser;

    public ProspectsController(
        IProspectService prospectService,
        ICurrentUserService currentUser)
    {
        _prospectService = prospectService;
        _currentUser = currentUser;
    }

    [HttpGet("today")]
    [ProducesResponseType(typeof(ProspectListResponse), 200)]
    public async Task<IActionResult> GetToday()
    {
        var result = await _prospectService.GetTodayAsync(_currentUser.AgentId);
        return Ok(result);
    }

    [HttpGet("history")]
    [ProducesResponseType(typeof(ProspectHistoryResponse), 200)]
    public async Task<IActionResult> GetHistory(
        [FromQuery] string? search,
        [FromQuery] int page = 1)
    {
        var result = await _prospectService.GetHistoryAsync(
            _currentUser.AgentId, search, page);
        return Ok(result);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ProspectDetailDto), 200)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> GetDetail(Guid id)
    {
        var result = await _prospectService.GetDetailAsync(id, _currentUser.AgentId);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpPost]
    [RequestSizeLimit(20_000_000)] // 20 MB total
    [ProducesResponseType(typeof(CreateProspectResponse), 201)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    [ProducesResponseType(403)]
    public async Task<IActionResult> Create([FromForm] CreateProspectRequest request)
    {
        var result = await _prospectService.CreateAsync(_currentUser.AgentId, request);
        return CreatedAtAction(nameof(GetDetail), new { id = result.Id }, result);
    }

    [HttpGet("monthly-stats")]
    [ProducesResponseType(typeof(MonthlyStatsResponse), 200)]
    public async Task<IActionResult> GetMonthlyStats()
    {
        var result = await _prospectService.GetMonthlyStatsAsync(_currentUser.AgentId);
        return Ok(result);
    }
}
```

---

## 7. Application Services

### 7.1 Service Interfaces

```csharp
public interface IAuthService
{
    Task<LoginResponse> LoginAsync(LoginRequest request, string? remoteIp, string? userAgent);
    Task<RefreshResponse> RefreshAsync(RefreshRequest request);
    Task LogoutAsync(Guid agentId, string refreshToken);
    Task<AgentFullProfileDto> GetProfileAsync(Guid agentId);
}

public interface IAttendanceService
{
    Task<TodayAttendanceDto?> GetTodayAsync(Guid agentId);
    Task<ShiftInfoDto> GetTodayShiftAsync(Guid agentId);
    Task<ClockInResponse> ClockInAsync(Guid agentId, ClockInRequest request);
    Task ClockOutAsync(Guid agentId, ClockOutRequest request);
    Task<AttendanceHistoryResponse> GetHistoryAsync(Guid agentId, int page, int pageSize);
    Task<AttendanceHistoryResponse> GetOlderAsync(Guid agentId, int page);
}

public interface IProspectService
{
    Task<ProspectListResponse> GetTodayAsync(Guid agentId);
    Task<ProspectHistoryResponse> GetHistoryAsync(Guid agentId, string? search, int page);
    Task<ProspectDetailDto?> GetDetailAsync(Guid prospectId, Guid agentId);
    Task<CreateProspectResponse> CreateAsync(Guid agentId, CreateProspectRequest request);
    Task<MonthlyStatsResponse> GetMonthlyStatsAsync(Guid agentId);
    Task<(Stream Content, string ContentType)> GetPhotoAsync(Guid prospectId, string kind, Guid agentId);
}

public interface INotificationService
{
    Task<NotificationsResponse> GetAllAsync(Guid agentId);
    Task MarkAsReadAsync(Guid notificationId, Guid agentId);
    Task MarkAllAsReadAsync(Guid agentId);
    Task<int> GetUnreadCountAsync(Guid agentId);
}

public interface IAgentService
{
    Task<HomeDashboardDto> GetDashboardAsync(Guid agentId);
    Task<AgentFullProfileDto> GetFullProfileAsync(Guid agentId);
    Task TrackReferralCopyAsync(Guid agentId);
}

public interface IFileStorageService
{
    Task<string> UploadAsync(Stream content, string contentType, string key);
    Task<Stream> DownloadAsync(string key);
    Task DeleteAsync(string key);
    string GetPresignedUrl(string key, TimeSpan expiry);
}

public interface IGeoService
{
    double HaversineDistance(decimal lat1, decimal lon1, decimal lat2, decimal lon2);
    (Branch? Nearest, double DistanceMeters) FindNearestBranch(decimal lat, decimal lon);
    bool IsWithinGeoFence(decimal lat, decimal lon, Guid branchId);
}
```

### 7.2 Example: ProspectService.CreateAsync

```csharp
public async Task<CreateProspectResponse> CreateAsync(
    Guid agentId, CreateProspectRequest request)
{
    // 1. Validate agent has clocked in
    var hasClockedIn = await _attendanceRepo.HasClockedInTodayAsync(agentId);
    if (!hasClockedIn)
        throw new ForbiddenException("Anda belum clock in hari ini.");

    // 2. Validate GPS accuracy
    if (request.GeoAccuracyMeters > 50m)
        throw new ValidationException("Akurasi GPS terlalu rendah (max 50 m).");

    // 3. Generate prospect ID
    var prospectId = Guid.NewGuid();
    var now = DateTimeOffset.Now;   // WIB

    // 4. Upload & stamp photos
    using var plangStream = request.PlangPhoto.OpenReadStream();
    using var stampedPlang = _photoStampService.Stamp(plangStream, now, request);
    var plangKey = $"prospects/{agentId}/{prospectId}/plang_{now:yyyyMMddHHmmss}.jpg";
    await _fileStorage.UploadAsync(stampedPlang, "image/jpeg", plangKey);

    using var selfieStream = request.SelfiePhoto.OpenReadStream();
    using var stampedSelfie = _photoStampService.Stamp(selfieStream, now, request);
    var selfieKey = $"prospects/{agentId}/{prospectId}/selfie_{now:yyyyMMddHHmmss}.jpg";
    await _fileStorage.UploadAsync(stampedSelfie, "image/jpeg", selfieKey);

    // 5. Persist
    var prospect = new Prospect
    {
        Id = prospectId,
        AgentId = agentId,
        StoreName = request.StoreName.Trim(),
        Address = request.Address.Trim(),
        PicName = request.PicName.Trim(),
        PicPhoneNumber = NormalizePhone(request.PicPhoneNumber),
        Latitude = request.Latitude,
        Longitude = request.Longitude,
        GeoAccuracyMeters = request.GeoAccuracyMeters,
        Notes = request.Notes?.Trim(),
        VisitedAt = now,
        PlangPhotoUrl = plangKey,
        SelfiePhotoUrl = selfieKey,
        Status = ProspectStatus.PendingVerification
    };

    await _prospectRepo.AddAsync(prospect);

    // 6. Create notification
    await _notificationService.CreateAsync(agentId, NotificationType.ProspectSaved,
        NotificationIconColor.Green, "Prospek tersimpan",
        $"Data {request.StoreName} beserta dokumentasi foto berhasil masuk ke sistem Mini ATM.");

    // 7. Invalidate caches
    await _cache.RemoveAsync($"prospects:today:{agentId}");
    await _cache.RemoveAsync($"dashboard:{agentId}");

    return new CreateProspectResponse(
        prospect.Id,
        prospect.StoreName,
        prospect.PicName,
        prospect.PicPhoneNumber,
        now.ToString("HH:mm"),
        "2 foto terlampir");
}
```

---

## 8. DTOs Summary

### 8.1 Request DTOs

| DTO | Used By |
|-----|---------|
| `LoginRequest` | POST `/auth/login` |
| `RefreshRequest` | POST `/auth/refresh` |
| `ClockInRequest` | POST `/attendance/clock-in` |
| `ClockOutRequest` | POST `/attendance/clock-out` |
| `CreateProspectRequest` | POST `/prospects` |

### 8.2 Response DTOs

| DTO | Used By |
|-----|---------|
| `LoginResponse` | POST `/auth/login` |
| `RefreshResponse` | POST `/auth/refresh` |
| `AgentProfileDto` | embedded in LoginResponse |
| `TodayAttendanceDto` | GET `/attendance/today` |
| `ShiftInfoDto` | GET `/attendance/shift` |
| `ClockInResponse` | POST `/attendance/clock-in` |
| `AttendanceHistoryResponse` | GET `/attendance/history` |
| `ProspectListResponse` | GET `/prospects/today` |
| `ProspectHistoryResponse` | GET `/prospects/history` |
| `ProspectDetailDto` | GET `/prospects/{id}` |
| `CreateProspectResponse` | POST `/prospects` |
| `MonthlyStatsResponse` | GET `/prospects/monthly-stats` |
| `NotificationsResponse` | GET `/notifications` |
| `HomeDashboardDto` | GET `/agents/dashboard` |
| `AgentFullProfileDto` | GET `/agents/profile` |

---

## 9. Validation Rules

### 9.1 FluentValidation Profiles

```csharp
public class CreateProspectRequestValidator : AbstractValidator<CreateProspectRequest>
{
    public CreateProspectRequestValidator()
    {
        RuleFor(x => x.StoreName)
            .NotEmpty().WithMessage("Nama toko wajib diisi.")
            .MaximumLength(200);

        RuleFor(x => x.Address)
            .NotEmpty().WithMessage("Alamat toko wajib diisi.")
            .MaximumLength(500);

        RuleFor(x => x.PicName)
            .NotEmpty().WithMessage("Nama PIC wajib diisi.")
            .MaximumLength(100);

        RuleFor(x => x.PicPhoneNumber)
            .NotEmpty().WithMessage("Nomor telepon wajib diisi.")
            .Must(BeValidPhone).WithMessage("Nomor telepon minimal 9 digit.");

        RuleFor(x => x.Latitude).InclusiveBetween(-90m, 90m);
        RuleFor(x => x.Longitude).InclusiveBetween(-180m, 180m);
        RuleFor(x => x.GeoAccuracyMeters).LessThanOrEqualTo(50m)
            .WithMessage("Akurasi GPS terlalu rendah (max 50 m).");

        RuleFor(x => x.PlangPhoto)
            .NotNull().WithMessage("Foto plang wajib diambil.")
            .Must(BeImage).WithMessage("Format file harus JPEG/PNG.")
            .Must(BeUnder5Mb).WithMessage("Ukuran foto maksimal 5 MB.");

        RuleFor(x => x.SelfiePhoto)
            .NotNull().WithMessage("Foto selfie wajib diambil.")
            .Must(BeImage).WithMessage("Format file harus JPEG/PNG.")
            .Must(BeUnder5Mb).WithMessage("Ukuran foto maksimal 5 MB.");

        RuleFor(x => x.Notes).MaximumLength(1000).When(x => !string.IsNullOrEmpty(x.Notes));
    }
}
```

---

## 10. Business Logic Rules

### 10.1 Attendance Rules

| Rule | Description |
|------|-------------|
| BR-A01 | An agent cannot clock in twice on the same date. |
| BR-A02 | An agent cannot clock out without clocking in. |
| BR-A03 | An agent cannot clock out twice on the same date. |
| BR-A04 | GPS accuracy must be ≤ 50 meters. |
| BR-A05 | Agent must be within 200 m of registered branch geo-fence. |
| BR-A06 | Status `Late` if clock-in > `ExpectedIn + 5 min`. |
| BR-A07 | `ClockInReminder` notification scheduled at `ExpectedOut - 1h`. |
| BR-A08 | If agent hasn't clocked out by `ExpectedOut`, system logs `Absent` at end of day (23:59). |
| BR-A09 | Attendance records are immutable after clock-out; corrections require supervisor approval. |

### 10.2 Prospect Rules

| Rule | Description |
|------|-------------|
| BR-P01 | Agent must have clocked in today to create prospects. |
| BR-P02 | Both photos (plang + selfie) are mandatory. |
| BR-P03 | Photo metadata (timestamp + coordinates) stamped server-side. |
| BR-P04 | New prospects are `PendingVerification` by default. |
| BR-P05 | Verified prospects are immutable by agents. |
| BR-P06 | Agent