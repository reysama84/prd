# Product Requirements Document (PRD) — Backend

## Sales Point — Mini ATM Field Acquisition App

**Version:** 1.0.0 | **Platform:** .NET 8 Web API | **Date:** 2026-09-04

---

## 1. Overview

### 1.1 Purpose

The backend provides RESTful APIs to power the **Sales Point** mobile web application, used by field sales agents to acquire Mini ATM merchant locations. Core capabilities include agent authentication, GPS-based attendance (clock-in/out), prospect (store visit) management with photo documentation, notifications, and performance dashboards.

### 1.2 Tech Stack

| Layer | Technology |
|---|---|
| Framework | .NET 8 ASP.NET Core Web API |
| ORM | Entity Framework Core 8 (Code-First) |
| Database | SQL Server 2022 |
| Cache | Redis (sessions, rate-limit) |
| Auth | JWT Bearer tokens + Refresh tokens |
| File Storage | Azure Blob Storage (photos) |
| Real-time | SignalR (notifications, optional) |
| Docs | Swashbuckle (OpenAPI 3.1) |
| Logging | Serilog → Seq / Application Insights |
| Validation | FluentValidation |
| Mapping | AutoMapper |

### 1.3 Solution Structure

```
SalesPoint.sln
├── src/
│   ├── SalesPoint.Api/                 # Controllers, Middleware, Program.cs
│   ├── SalesPoint.Application/        # Services, DTOs, Validators, Interfaces
│   ├── SalesPoint.Domain/             # Entities, Enums, Domain Events
│   ├── SalesPoint.Infrastructure/     # EF Core, Repositories, External services
│   └── SalesPoint.Shared/             # Common, Constants, Results
└── tests/
    ├── SalesPoint.UnitTests/
    └── SalesPoint.IntegrationTests/
```

---

## 2. Authentication & Authorization

### 2.1 Roles

| Role | Key | Description |
|---|---|---|
| Agent | `agent` | Field sales agent (clock-in, create prospects) |
| Supervisor | `supervisor` | Area supervisor (view team data, approve) |
| Admin | `admin` | Full system access |

### 2.2 JWT Configuration

```json
{
  "Jwt": {
    "Issuer": "SalesPoint.Api",
    "Audience": "SalesPoint.App",
    "AccessTokenMinutes": 60,
    "RefreshTokenDays": 30,
    "SigningKey": "<256-bit-secret>"
  }
}
```

### 2.3 Auth Endpoints

#### POST `/api/auth/login`
Authenticates agent credentials and returns access + refresh tokens.

**Request Body:**
```json
{
  "username": "rizky.pratama",
  "password": "salespoint",
  "rememberMe": true
}
```

**Response 200:**
```json
{
  "accessToken": "eyJhbG...",
  "refreshToken": "d2f4a8...",
  "expiresIn": 3600,
  "user": {
    "id": "usr_abc123",
    "username": "rizky.pratama",
    "fullName": "Rizky Pratama Nugroho",
    "email": "rizky.pratama@visionet.co.id",
    "role": "agent",
    "referralCode": "SP-RZK2041",
    "branchName": "Kantor Cabang Jakarta Selatan",
    "avatarInitials": "RP"
  }
}
```

**Response 401:** `{ "message": "Username atau kata sandi salah." }`

#### POST `/api/auth/refresh`
#### POST `/api/auth/logout`
#### GET `/api/auth/me`
Returns the current authenticated user's profile snapshot (used after splash/loading to hydrate the app).

---

## 3. Controllers

### 3.1 Controller Summary

| Controller | Route Prefix | Auth |
|---|---|---|
| `AuthController` | `/api/auth` | Public (login), JWT (refresh, logout, me) |
| `AttendanceController` | `/api/attendance` | JWT |
| `ProspectController` | `/api/prospects` | JWT |
| `NotificationController` | `/api/notifications` | JWT |
| `DashboardController` | `/api/dashboard` | JWT |
| `FileController` | `/api/files` | JWT |
| `UserController` | `/api/users` | JWT + Admin/Supervisor |

---

## 4. Attendance Module

### 4.1 Entities

```csharp
public class Attendance
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public DateTime ClockInTime { get; set; }       // UTC
    public DateTime? ClockOutTime { get; set; }      // UTC
    public DateTimeOffset ClockInLocal { get; set; } // WIB display
    public DateTimeOffset? ClockOutLocal { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public double GpsAccuracyMeters { get; set; }
    public string BranchName { get; set; }
    public string BranchAddress { get; set; }
    public string ShiftName { get; set; }              // "Reguler"
    public TimeSpan ShiftStart { get; set; }           // 08:00
    public TimeSpan ShiftEnd { get; set; }             // 17:00
    public string ClockInPhotoUrl { get; set; }        // selfie
    public string ClockOutPhotoUrl { get; set; }
    public AttendanceStatus Status { get; set; }
    public Guid UserId { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum AttendanceStatus
{
    ClockedIn,
    ClockedOut,
    Late,
    OnLeave
}
```

### 4.2 DTOs

```csharp
public record ClockInRequest(
    double Latitude,
    double Longitude,
    double GpsAccuracyMeters,
    string SelfiePhotoBase64   // or multipart/form-data
);

public record ClockOutRequest(
    double Latitude,
    double Longitude,
    double GpsAccuracyMeters,
    string? SelfiePhotoBase64
);

public record AttendanceDto(
    Guid Id,
    string ClockInTime,     // "07:48"
    string? ClockOutTime,
    string DateLabel,       // "Jumat, 4 September 2026"
    string ShiftLabel,      // "Reguler · 08:00 – 17:00 WIB"
    string BranchName,
    string BranchAddress,
    string Method,          // "GPS + Selfie · akurasi ±8 m"
    string DurationLabel,   // "5 jam 34 menit berjalan"
    string Status,          // "Tepat waktu" | "Terlambat" | "Izin"
    string StatusClass      // "good" | "warn" | "crit" | "info"
);

public record AttendanceHistoryDto(
    string DayShort,        // "Kamis"
    string DateNumber,      // "03"
    string MonthShort,      // "Sep"
    string ClockIn,         // "07:52"
    string ClockOut,        // "17:10"
    string Duration,       // "9j 18m"
    string Status,
    string StatusClass
);
```

### 4.3 Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/api/attendance/today` | Get today's attendance record for the current user |
| GET | `/api/attendance/status` | Lightweight check: `{ "clockedIn": true, "clockedOut": false, "clockInTime": "07:48" }` |
| POST | `/api/attendance/clock-in` | Submit clock-in with GPS + selfie photo |
| POST | `/api/attendance/clock-out` | Submit clock-out |
| GET | `/api/attendance/history?monthOffset=0` | Paginated attendance history (20 per page) |

### 4.4 Business Rules

1. **Geofence validation:** Clock-in coordinates must be within 150 meters of the assigned branch. If outside, return `400 GeofenceViolation`.
2. **One clock-in per day:** If user already has a clock-in for `DateTime.UtcNow.Date`, return `409 AlreadyClockedIn`.
3. **Clock-out requires clock-in:** If no clock-in exists, return `400 NoClockInFound`.
4. **Late detection:** If `ClockInTime` > `ShiftStart + 5 minutes`, set `Status = Late`.
5. **GPS accuracy threshold:** If `GpsAccuracyMeters > 50`, return `400 LowGpsAccuracy`.
6. **Branch resolution:** The system matches the agent's GPS to the nearest registered branch within the geofence radius.

### 4.5 Attendance Service

```csharp
public interface IAttendanceService
{
    Task<Result<AttendanceDto>> ClockInAsync(Guid userId, ClockInRequest req, CancellationToken ct);
    Task<Result<AttendanceDto>> ClockOutAsync(Guid userId, ClockOutRequest req, CancellationToken ct);
    Task<Result<AttendanceDto?>> GetTodayAsync(Guid userId, CancellationToken ct);
    Task<Result<AttendanceStatusDto>> GetStatusAsync(Guid userId, CancellationToken ct);
    Task<Result<PagedResult<AttendanceHistoryDto>>> GetHistoryAsync(
        Guid userId, int monthOffset, int page, int pageSize, CancellationToken ct);
}
```

---

## 5. Prospect Module

### 5.1 Entities

```csharp
public class Prospect
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string StoreName { get; set; }              // "Toko Berkah Jaya"
    public string Address { get; set; }
    public string PicName { get; set; }                // "Ibu Sri Wahyuni"
    public string PicPhone { get; set; }               // "081288452190"
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public double GpsAccuracyMeters { get; set; }
    public string Notes { get; set; }                  // optional visit notes
    public string PlangPhotoUrl { get; set; }          // storefront photo
    public string SelfiePhotoUrl { get; set; }         // selfie with PIC
    public string VisitTimeLabel { get; set; }         // "08:52" (HH:mm local)
    public DateTime VisitTimeUtc { get; set; }
    public string DateLabel { get; set; }               // "4 September 2026"
    public DateTime CreatedAt { get; set; }
    public ProspectStatus Status { get; set; }
    public int VisitOrderOfDay { get; set; }           // 1st, 2nd, 3rd visit
}

public enum ProspectStatus
{
    Pending,
    Verified,
    Rejected,
    FollowUpRequired
}
```

### 5.2 DTOs

```csharp
public record CreateProspectRequest(
    string StoreName,
    string Address,
    string PicName,
    string PicPhone,             // E.164 or local format, validated ≥9 digits
    double Latitude,
    double Longitude,
    double GpsAccuracyMeters,
    string? Notes,
    string PlangPhotoBase64,     // JPEG base64 with geo+time stamp applied by client
    string SelfiePhotoBase64
);

public record ProspectListItemDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string VisitTimeLabel,       // "08:52"
    string? PhotoThumbUrl,       // plang photo thumbnail
    string DateLabel,
    string StatusLabel,         // "Terverifikasi" | "Menunggu" | "Ditolak"
    string StatusClass,         // "good" | "warn" | "crit"
    int VisitOrderOfDay
);

public record ProspectDetailDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string PicPhone,
    string VisitTimeLabel,
    string DateLabel,
    string PlangPhotoUrl,
    string SelfiePhotoUrl,
    string? Notes,
    string Coordinates,         // "-6.26412, 106.79931"
    string GpsAccuracyLabel,   // "akurasi ±6 m"
    string StatusLabel,
    string StatusClass
);

public record ProspectGroupDto(
    string DateLabel,          // "Hari Ini — Jumat, 4 September 2026"
    int Count,
    List<ProspectListItemDto> Items
);

public record PagedProspectsDto(
    List<ProspectGroupDto> Groups,
    bool HasMore
);
```

### 5.3 Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/api/prospects/today` | List today's prospects for current agent |
| GET | `/api/prospects/today/count` | Returns `{ "count": 4, "nextVisitOrder": 5 }` |
| GET | `/api/prospects/history?page=1&search=` | Paginated + searchable history grouped by day |
| GET | `/api/prospects/{id}` | Prospect detail with full-res photos |
| POST | `/api/prospects` | Create new prospect (form-data with photos or JSON base64) |
| DELETE | `/api/prospects/{id}` | Soft-delete (admin/supervisor only) |

### 5.4 Validators (FluentValidation)

```csharp
public class CreateProspectRequestValidator : AbstractValidator<CreateProspectRequest>
{
    public CreateProspectRequestValidator()
    {
        RuleFor(x => x.StoreName).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Address).NotEmpty().MaximumLength(500);
        RuleFor(x => x.PicName).NotEmpty().MaximumLength(150);
        RuleFor(x => x.PicPhone).NotEmpty()
            .Matches(@"^(\+62|62|0)8[1-9]\d{6,11}$")
            .WithMessage("Nomor telepon PIC tidak valid (minimal 9 digit).");
        RuleFor(x => x.Latitude).InclusiveBetween(-90, 90);
        RuleFor(x => x.Longitude).InclusiveBetween(-180, 180);
        RuleFor(x => x.GpsAccuracyMeters).LessThanOrEqualTo(50)
            .WithMessage("Akurasi GPS terlalu rendah (maks ±50 m).");
        RuleFor(x => x.PlangPhotoBase64).NotEmpty();
        RuleFor(x => x.SelfiePhotoBase64).NotEmpty();
        RuleFor(x => x.Notes).MaximumLength(2000);
    }
}
```

### 5.5 Business Rules

1. **Authentication required:** Only authenticated agents can create prospects.
2. **Clock-in prerequisite:** Agent must have a valid clock-in for the current day before creating a prospect. If not, return `403 ClockInRequired` with message: *"Absen wajib dilakukan sebelum memulai kunjungan prospek."*
3. **Photo requirements:** Both plang (storefront) and selfie photos are mandatory. Each must be:
   - JPEG format, base64-encoded
   - Min size: 50 KB, Max size: 8 MB each
   - Client must embed geolocation + timestamp watermark before upload
4. **Phone normalization:** `PicPhone` stored in E.164 format (`+6281288452190`), but returned in display format (`0812 8845 2190`).
5. **Visit ordering:** `VisitOrderOfDay` auto-increments per agent per day.
6. **Search scope:** `search` query param matches against `StoreName`, `Address`, `PicName` (case-insensitive, full-text).
7. **Pagination:** 20 items per page. `HasMore` flag triggers the "Muat data sebelumnya" button.

### 5.6 Prospect Service

```csharp
public interface IProspectService
{
    Task<Result<ProspectDetailDto>> CreateAsync(Guid agentId, CreateProspectRequest req, CancellationToken ct);
    Task<Result<List<ProspectListItemDto>>> GetTodayAsync(Guid agentId, CancellationToken ct);
    Task<Result<TodayCountDto>> GetTodayCountAsync(Guid agentId, CancellationToken ct);
    Task<Result<PagedProspectsDto>> GetHistoryAsync(
        Guid agentId, int page, string? search, CancellationToken ct);
    Task<Result<ProspectDetailDto>> GetByIdAsync(Guid id, Guid agentId, CancellationToken ct);
}
```

---

## 6. File / Photo Module

### 6.1 Endpoints

| Method | Route | Description |
|---|---|---|
| POST | `/api/files/upload` | Upload a single photo (multipart/form-data), returns URL |
| GET | `/api/files/{fileId}/download` | Download original photo (authenticated) |
| GET | `/api/files/{fileId}/thumbnail` | Download thumbnail (400px wide) |

### 6.2 Storage Strategy

- **Container:** `prospect-photos` (Azure Blob Storage)
- **Path pattern:** `{userId}/{prospectId}/{photoType}_{timestamp}.jpg`
  - Example: `usr_abc/prospect_xyz/plang_20260904085200.jpg`
- **Blob metadata:** `userId`, `prospectId`, `photoType`, `latitude`, `longitude`, `capturedAt`
- **SAS tokens:** Read-only SAS URLs with 24-hour expiry for `download`/`thumbnail` endpoints.
- **Thumbnails:** Generated server-side using `SixLabors.ImageSharp` at upload time (400px wide, 75% JPEG quality).

### 6.3 File DTOs

```csharp
public record FileUploadResponse(
    string FileId,
    string Url,           // CDN/SAS URL for display
    string ThumbnailUrl,
    long SizeBytes,
    string ContentType    // "image/jpeg"
);

public record PhotoDownloadDto(
    string FileName,      // "Prospek_Toko_Berkah_Jaya_plang.jpg"
    string ContentType,
    byte[] Data            // or stream
);
```

---

## 7. Notification Module

### 7.1 Entities

```csharp
public class Notification
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public string Title { get; set; }       // "Prospek tersimpan"
    public string Body { get; set; }        // "Data Toko Cahaya Elektronik..."
    public NotificationType Type { get; set; }
    public string IconColor { get; set; }   // "green" | "orange" | "blue" | "red"
    public string IconSvgKey { get; set; }  // "check" | "clock" | "location" | "warning"
    public string TimeLabel { get; set; }   // "12:05 WIB"
    public DateTime CreatedAtUtc { get; set; }
    public string DateGroupLabel { get; set; }  // "Hari Ini" | "Kemarin" | "Rabu, 2 September 2026"
    public bool IsRead { get; set; }
}

public enum NotificationType
{
    ProspectSaved,
    ClockInRecorded,
    ClockOutReminder,
    FollowUpScheduled,
    BonusCredited,
    AppUpdate,
    MeetingScheduled
}
```

### 7.2 DTOs

```csharp
public record NotificationDto(
    Guid Id,
    string Title,
    string Body,
    string IconColor,
    string IconSvgMarkup,   // raw SVG path content for client rendering
    string TimeLabel,
    string DateGroupLabel,
    bool IsRead
);

public record NotificationGroupDto(
    string DateLabel,
    int UnreadCount,
    List<NotificationDto> Items
);

public record NotificationSummaryDto(
    int UnreadCount,
    List<NotificationGroupDto> Groups
);
```

### 7.3 Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/api/notifications` | All notifications grouped by date |
| GET | `/api/notifications/unread-count` | Returns `{ "unreadCount": 3 }` |
| PUT | `/api/notifications/{id}/read` | Mark single notification as read |
| PUT | `/api/notifications/read-all` | Mark all as read |

### 7.4 Business Rules

1. Notifications are generated by **domain events** (see § 10.1):
   - `ProspectCreatedEvent` → "Prospek tersimpan" notification
   - `AttendanceClockedInEvent` → "Clock in tercatat" notification
   - Scheduled job triggers "Jangan lupa clock out" at 11:00 WIB if not yet clocked out
2. Date group labels computed server-side based on `CreatedAtUtc` converted to WIB:
   - Same calendar day → `"Hari Ini"`
   - Previous day → `"Kemarin"`
   - Else → `"Wednesday, 2 September 2026"` (Indonesian day/month names)
3. Unread count feeds the bottom-nav badge.

---

## 8. Dashboard Module

### 8.1 Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/api/dashboard/home` | Home screen aggregate data |
| GET | `/api/dashboard/profile` | Profile screen aggregate data |
| GET | `/api/dashboard/monthly-chart?year=2026` | Monthly prospect bar chart data |

### 8.2 Home Dashboard DTO

```csharp
public record HomeDashboardDto(
    string WelcomeName,          // "Rizky Pratama"
    string DateLabel,           // "Jumat, 4 September 2026"
    string ReferralCode,        // "SP-RZK2041"
    int TodayProspectCount,     // 4
    int NextVisitOrder,         // 5
    AttendanceStatusDto Attendance,
    TodaySummaryDto Stats
);

public record TodaySummaryDto(
    int TotalProspects,       // 4
    int Verified,              // 1
    int Pending,               // 2
    int Rejected,              // 1
    int Target,                // 10
    double CompletionPercent   // 40.0
);
```

### 8.3 Profile Dashboard DTO

```csharp
public record ProfileDashboardDto(
    string FullName,
    string Username,
    string Email,
    string ReferralCode,
    string RoleLabel,           // "Sales Point Agent"
    bool IsActive,
    MonthlyChartDto MonthlyChart,
    ProfileStatsDto Stats
);

public record MonthlyChartDto(
    int TotalProspects,         // 677
    List<MonthlyChartItemDto> Items
);

public record MonthlyChartItemDto(
    string MonthLabel,          // "Jan"
    int Count,                  // 62
    bool IsCurrentMonth         // false (Sep is current)
);

public record ProfileStatsDto(
    int ThisMonthProspects,     // 38
    double MonthGrowthPercent,  // 12.0
    int AttendanceDaysThisMonth, // 4
    double AttendancePercent,    // 100.0
    string AttendanceLabel      // "Baik"
);
```

### 8.4 Dashboard Endpoints Detail

#### GET `/api/dashboard/monthly-chart?year=2026`

```json
{
  "totalProspects": 677,
  "items": [
    { "monthLabel": "Jan", "count": 62, "isCurrentMonth": false },
    { "monthLabel": "Feb", "count": 71, "isCurrentMonth": false },
    { "monthLabel": "Mar", "count": 58, "isCurrentMonth": false },
    { "monthLabel": "Apr", "count": 83, "isCurrentMonth": false },
    { "monthLabel": "Mei", "count": 90, "isCurrentMonth": false },
    { "monthLabel": "Jun", "count": 76, "isCurrentMonth": false },
    { "monthLabel": "Jul", "count": 95, "isCurrentMonth": false },
    { "monthLabel": "Ags", "count": 104, "isCurrentMonth": false },
    { "monthLabel": "Sep", "count": 38, "isCurrentMonth": true }
  ]
}
```

---

## 9. User Module (Admin/Supervisor)

### 9.1 Endpoints

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/api/users` | Admin | Paginated list of agents |
| POST | `/api/users` | Admin | Create new agent (generates referral code) |
| PUT | `/api/users/{id}` | Admin | Update agent info |
| PUT | `/api/users/{id}/reset-password` | Admin/Supervisor | Reset password |
| GET | `/api/users/{id}/performance` | Supervisor | Agent performance summary |

---

## 10. Cross-Cutting Concerns

### 10.1 Domain Events

```csharp
public record ProspectCreatedEvent(Guid ProspectId, Guid AgentId, string StoreName) : IDomainEvent;
public record AttendanceClockedInEvent(Guid UserId, string TimeLabel, string BranchName) : IDomainEvent;
public record AttendanceClockedOutEvent(Guid UserId, string TimeLabel) : IDomainEvent;
```

**Notification handler example:**
```csharp
public class ProspectCreatedNotificationHandler : INotificationHandler<ProspectCreatedEvent>
{
    public async Task Handle(ProspectCreatedEvent e, CancellationToken ct)
    {
        var notif = new Notification
        {
            UserId = e.AgentId,
            Title = "Prospek tersimpan",
            Body = $"Data {e.StoreName} beserta dokumentasi foto berhasil masuk ke sistem Mini ATM.",
            Type = NotificationType.ProspectSaved,
            IconColor = "green",
            IconSvgKey = "check",
            TimeLabel = DateTime.UtcNow.ToWibLabel(), // "12:05 WIB"
            DateGroupLabel = "Hari Ini"
        };
        await _notifRepo.AddAsync(notif, ct);
    }
}
```

### 10.2 Global Exception Handling

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var problem = ex switch
        {
            ValidationException ve => new ProblemDetails
            {
                Title = "Validation Error",
                Status = 400,
                Detail = ve.Message,
                Extensions = { ["errors"] = ve.Errors }
            },
            NotFoundException => new ProblemDetails { Title = "Not Found", Status = 404 },
            GeofenceViolationException => new ProblemDetails
            {
                Title = "Geofence Violation",
                Status = 400,
                Detail = "Lokasi Anda terlalu jauh dari kantor cabang."
            },
            _ => new ProblemDetails { Title = "Internal Server Error", Status = 500 }
        };
        // serialize + write
        return true;
    }
}
```

### 10.3 Response Envelope

All successful API responses use a consistent envelope:

```json
{
  "success": true,
  "data": { ... },
  "message": null,
  "timestamp": "2026-09-04T08:52:00Z"
}
```

Error responses:
```json
{
  "success": false,
  "data": null,
  "message": "Absen wajib dilakukan sebelum memulai kunjungan prospek.",
  "errorCode": "CLOCK_IN_REQUIRED",
  "timestamp": "2026-09-04T08:52:00Z"
}
```

### 10.4 Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("auth", ctx => RateLimitPartition.GetFixedWindowLimiter(
        ctx.Connection.RemoteIpAddress?.ToString(),
        _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 5,
            Window = TimeSpan.FromMinutes(1)
        }));

    options.AddPolicy("api", ctx => RateLimitPartition.GetTokenBucketLimiter(
        ctx.User.Identity?.Name,
        _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,
            TokensPerPeriod = 50,
            ReplenishmentPeriod = TimeSpan.FromSeconds(10)
        }));
});
```

### 10.5 Time Zone Handling

- All `DateTime` stored as **UTC** in the database.
- Conversion to **WIB (UTC+7)** for display happens in the Application layer via a `ITimeZoneService`:

```csharp
public interface ITimeZoneService
{
    DateTimeOffset ToWib(DateTime utc);
    string ToWibLabel(DateTime utc);          // "07:48 WIB"
    string ToWibDateLabel(DateTime utc);      // "Jumat, 4 September 2026"
    string ToWibTimeOnly(DateTime utc);       // "07:48"
    string ToWibDateGroupLabel(DateTime utc); // "Hari Ini" | "Kemarin" | "Rabu, 2 September 2026"
}
```

Indonesian culture info configured:
```csharp
var idCulture = new CultureInfo("id-ID");
```

### 10.6 Logging

```csharp
builder.Host.UseSerilog((ctx, cfg) => cfg
    .Enrich.FromLogContext()
    .Enrich.WithProperty("App", "SalesPoint.Api")
    .WriteTo.Console()
    .WriteTo.Seq("http://seq:5341")
    .WriteTo.ApplicationInsights(telemetryConfig, TelemetryConverter.Traces));
```

Key log events:
- `POST /api/auth/login` → `{ "userId": "...", "ip": "...", "success": true }`
- `POST /api/attendance/clock-in` → `{ "userId": "...", "lat": -6.26, "lng": 106.79, "branch": "..." }`
- `POST /api/prospects` → `{ "agentId": "...", "prospectId": "...", "storeName": "..." }`

---

## 11. Database Schema (Summary)

```sql
-- Users
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Username NVARCHAR(100) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(500) NOT NULL,
    FullName NVARCHAR(200) NOT NULL,
    Email NVARCHAR(256) NOT NULL,
    Role NVARCHAR(20) NOT NULL DEFAULT 'agent',
    ReferralCode NVARCHAR(20) NOT NULL UNIQUE,
    BranchId UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Branches(Id),
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- Branches
CREATE TABLE Branches (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(200) NOT NULL,
    Address NVARCHAR(500) NOT NULL,
    Latitude DECIMAL(10,7) NOT NULL,
    Longitude DECIMAL(10,7) NOT NULL,
    GeofenceRadiusMeters INT NOT NULL DEFAULT 150
);

-- Attendances
CREATE TABLE Attendances (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL FOREIGN KEY REFERENCES Users(Id),
    ClockInTimeUtc DATETIME2 NOT NULL,
    ClockOutTimeUtc DATETIME2 NULL,
    Latitude DECIMAL(10,7) NOT NULL,
    Longitude DECIMAL(10,7) NOT NULL,
    GpsAccuracyMeters DECIMAL(6,1) NOT NULL,
    BranchId UNIQUEIDENTIFIER NOT NULL FOREIGN KEY REFERENCES Branches(Id),
    ClockInPhotoUrl NVARCHAR(500) NULL,
    ClockOutPhotoUrl NVARCHAR(500) NULL,
    Status NVARCHAR(20) NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT UQ_Attendance_User_Date UNIQUE (UserId, CAST(ClockInTimeUtc AS DATE))
);

-- Prospects
CREATE TABLE Prospects (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    AgentId UNIQUEIDENTIFIER NOT NULL FOREIGN KEY REFERENCES Users(Id),
    StoreName NVARCHAR(200) NOT NULL,
    Address NVARCHAR(500) NOT NULL,
    PicName NVARCHAR(150) NOT NULL,
    PicPhone NVARCHAR(20) NOT NULL,
    Latitude DECIMAL(10,7) NOT NULL,
    Longitude DECIMAL(10,7) NOT NULL,
    GpsAccuracyMeters DECIMAL(6,1) NOT NULL,
    Notes NVARCHAR(2000) NULL,
    PlangPhotoUrl NVARCHAR(500) NOT NULL,
    SelfiePhotoUrl NVARCHAR(500) NOT NULL,
    VisitTimeUtc DATETIME2 NOT NULL,
    Status NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    VisitOrderOfDay INT NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- Notifications
CREATE TABLE Notifications (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL FOREIGN KEY REFERENCES Users(Id),
    Title NVARCHAR(200) NOT NULL,
    Body NVARCHAR(1000) NOT NULL,
    Type NVARCHAR(50) NOT NULL,
    IconColor NVARCHAR(20) NOT NULL,
    IconSvgKey NVARCHAR(50) NOT NULL,
    CreatedAtUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    IsRead BIT NOT NULL DEFAULT 0
);

-- Refresh Tokens
CREATE TABLE RefreshTokens (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL FOREIGN KEY REFERENCES Users(Id),
    TokenHash NVARCHAR(500) NOT NULL,
    ExpiresAtUtc DATETIME2 NOT NULL,
    RevokedAtUtc DATETIME2 NULL,
    CreatedAtUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

**Key indexes:**
```sql
CREATE INDEX IX_Prospects_AgentId_CreatedAt ON Prospects(AgentId, CreatedAt DESC);
CREATE INDEX IX_Prospects_AgentId_StoreName ON Prospects(AgentId, StoreName);
CREATE INDEX IX_Attendances_UserId_ClockInTime ON Attendances(UserId, ClockInTimeUtc DESC);
CREATE INDEX IX_Notifications_UserId_CreatedAt ON Notifications(UserId, CreatedAtUtc DESC);
```

---

## 12. API Conventions

### 12.1 Route & Naming

- Base URL: `https://api.salespoint.visionet.co.id/v1`
- All routes are lowercase kebab-case: `/api/prospects/{id}/photos`
- Query params are camelCase: `?monthOffset=0&pageSize=20`

### 12.2 Pagination

```json
{
  "data": [ ... ],
  "page": 1,
  "pageSize": 20,
  "totalItems": 47,
  "totalPages": 3,
  "hasMore": true
}
```

### 12.3 Standard HTTP Status Codes

| Code | Usage |
|---|---|
| 200 | Successful GET, PUT |
| 201 | Successful POST (create) |
| 204 | Successful DELETE |
| 400 | Validation error, bad request |
| 401 | Missing/invalid token |
| 403 | Authorized but forbidden (e.g., agent accessing another's prospect) |
| 404 | Resource not found |
| 409 | Conflict (e.g., already clocked in) |
| 429 | Rate limited |
| 500 | Unhandled server error |

---

## 13. Configuration

### `appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=SalesPointDb;...",
    "Redis": "redis:6379"
  },
  "Jwt": {
    "Issuer": "SalesPoint.Api",
    "Audience": "SalesPoint.App",
    "AccessTokenMinutes": 60,
    "RefreshTokenDays": 30,
    "SigningKey": "<from-secrets>"
  },
  "AzureBlob": {
    "ConnectionString": "<from-secrets>",
    "ContainerName": "prospect-photos"
  },
  "Geofence": {
    "DefaultRadiusMeters": 150,
    "MaxGpsAccuracyMeters": 50
  },
  "TimeZone": {
    "DisplayTimeZoneId": "SE Asia Standard Time"
  },
  "Cors": {
    "AllowedOrigins": [ "https://app.salespoint.visionet.co.id" ]
  }
}
```

---

## 14. Security Requirements

1. **Password hashing:** BCrypt with cost factor 12.
2. **JWT signing:** HMAC-SHA256, 256-bit key from Azure Key Vault.
3. **Refresh token rotation:** On refresh, the old token is revoked and a new pair issued.
4. **Data isolation:** All prospect/attendance queries filter by `AgentId == currentUserId` unless role is `supervisor`/`admin`.
5. **