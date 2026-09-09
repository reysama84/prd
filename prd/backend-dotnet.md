# Product Requirements Document (PRD) — Sales Point Backend (.NET)

> **Produk**: VisioNet Mini ATM — Sales Point
> **Platform Backend**: ASP.NET Core Web API (.NET 8)
> **Target Audience**: Field Sales Agents (akuisisi agen Mini ATM)
> **Versi Dokumen**: 1.0.0

---

## 1. Overview

### 1.1 Tujuan
Menyediakan REST API backend untuk aplikasi mobile **Sales Point** yang digunakan oleh agen lapangan VisioNet Mini ATM. Backend menangani autentikasi agen, absensi berbasis GPS+selfie, pencatatan prospek toko, manajemen dokumentasi foto, notifikasi, serta pelaporan performa agen.

### 1.2 Lingkup
PRD ini hanya mencakup **backend**. Frontend/mobile, infrastruktur deployment, dan integrasi pihak ketiga di luar lingkup dokumen ini.

### 1.3 Asumsi Teknis
- **Framework**: ASP.NET Core 8 Web API
- **ORM**: Entity Framework Core 8 (SQL Server / PostgreSQL)
- **Auth**: JWT Bearer Token + Refresh Token
- **File Storage**: Azure Blob Storage / AWS S3 (abstraksi via `IFileStorageService`)
- **Time Zone**: WIB (UTC+7), disimpan sebagai UTC di database
- **API Convention**: RESTful, versioning via URL `/api/v1/...`
- **Response Format**: `application/json`, snake_case atau camelCase (System.Text.Json default camelCase)

---

## 2. Arsitektur Backend

### 2.1 Layered Architecture
```
SalesPoint.API          → Controllers, Middleware, Filters
SalesPoint.Application  → Services, DTOs, Interfaces, Validators
SalesPoint.Domain       → Entities, Enums, Domain Events
SalesPoint.Infrastructure → EF Core DbContext, Repositories, External Services
```

### 2.2 Cross-Cutting Concerns
| Concern | Implementation |
|---|---|
| Logging | Serilog (structured logging) |
| Validation | FluentValidation |
| Error Handling | Global exception middleware → RFC 7807 ProblemDetails |
| Rate Limiting | `AspNetCoreRateLimit` (per-endpoint) |
| Caching | `IDistributedCache` (Redis) untuk lookup master & session |
| Audit Trail | `AuditableEntity` base (CreatedAt, CreatedBy, UpdatedAt, UpdatedBy) |

### 2.3 Dependency Injection
Semua services terdaftar di `Program.cs` dengan lifetime berikut:
- **Scoped**: `DbContext`, services per-request (Application Services, Repositories)
- **Singleton**: `ILogger`, `IConfiguration`, `IMemoryCache`
- **Transient**: `IValidator<T>`, mappers

---

## 3. Data Model (Domain Entities)

### 3.1 Entities

#### Agent (User)
```csharp
public class Agent : AuditableEntity
{
    public Guid Id { get; set; }
    public string Username { get; set; }          // unique, max 50
    public string PasswordHash { get; set; }
    public string FullName { get; set; }           // max 150
    public string Email { get; set; }              // unique
    public string PhoneNumber { get; set; }        // E.164
    public string ReferralCode { get; set; }       // "SP-RZK2041"
    public Guid? BranchId { get; set; }
    public AgentStatus Status { get; set; }         // Active, Suspended, Inactive
    public DateTimeOffset? LastLoginAt { get; set; }
    public ICollection<Attendance> Attendances { get; set; }
    public ICollection<Prospect> Prospects { get; set; }
    public ICollection<RefreshToken> RefreshTokens { get; set; }
}
```

#### Branch (Kantor Cabang)
```csharp
public class Branch : AuditableEntity
{
    public Guid Id { get; set; }
    public string Name { get; set; }                // "Jakarta Selatan"
    public string Address { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public double GeoFenceRadiusMeters { get; set; } // default 200m
    public TimeSpan ShiftStart { get; set; }         // 08:00
    public TimeSpan ShiftEnd { get; set; }            // 17:00
}
```

#### Attendance
```csharp
public class Attendance : AuditableEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public DateTimeOffset Date { get; set; }       // date only (WIB)
    public DateTimeOffset? ClockInAt { get; set; }
    public DateTimeOffset? ClockOutAt { get; set; }
    public double? ClockInLat { get; set; }
    public double? ClockInLng { get; set; }
    public double? ClockOutLat { get; set; }
    public double? ClockOutLng { get; set; }
    public string ClockInPhotoUrl { get; set; }    // selfie URL
    public string ClockOutPhotoUrl { get; set; }
    public double? GpsAccuracyMeters { get; set; }
    public AttendanceStatus Status { get; set; }   // OnTime, Late, Izin, Missed
    public TimeSpan? Duration => ClockOutAt - ClockInAt;
    public Agent Agent { get; set; }
}
```

#### Prospect (Prospek Toko)
```csharp
public class Prospect : AuditableEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string StoreName { get; set; }
    public string Address { get; set; }
    public string PicName { get; set; }
    public string PicPhone { get; set; }            // E.164
    public string Notes { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public double GpsAccuracyMeters { get; set; }
    public DateTimeOffset VisitedAt { get; set; }
    public string PlangPhotoUrl { get; set; }       // foto papan nama
    public string SelfiePhotoUrl { get; set; }      // selfie + PIC
    public ProspectStatus Status { get; set; }       // New, Verified, Rejected, Installed
    public Agent Agent { get; set; }
}
```

#### Notification
```csharp
public class Notification : AuditableEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string Title { get; set; }
    public string Body { get; set; }
    public NotificationType Type { get; set; }       // ProspectSaved, ClockReminder, BonusPaid, etc.
    public bool IsRead { get; set; }
    public DateTimeOffset? ReadAt { get; set; }
    public string PayloadJson { get; set; }          // additional context
}
```

#### RefreshToken
```csharp
public class RefreshToken : AuditableEntity
{
    public Guid Id { get; set; }
    public Guid AgentId { get; set; }
    public string Token { get; set; }                 // hashed
    public DateTimeOffset ExpiresAt { get; set; }
    public DateTimeOffset? RevokedAt { get; set; }
    public string ReplacedByToken { get; set; }
    public string CreatedByIp { get; set; }
    public bool IsActive => RevokedAt == null && DateTime.UtcNow <= ExpiresAt;
}
```

### 3.2 Enums
```csharp
public enum AgentStatus { Active, Suspended, Inactive }
public enum AttendanceStatus { OnTime, Late, Izin, Missed, Completed }
public enum ProspectStatus { New, Verified, Rejected, Installed }
public enum NotificationType { ProspectSaved, ClockReminder, ClockIn, ClockOut, BonusPaid, AppUpdate, Briefing }
```

---

## 4. Authentication & Authorization

### 4.1 Strategy
- **JWT Bearer Token** untuk autentikasi stateless (access token, TTL 15 menit)
- **Refresh Token** (TTL 7 hari) disimpan hashed di DB, rotation on use
- **Password Policy**: BCrypt hash (cost 11), min 8 karakter, lockout setelah 5 percobaan gagal
- **Claim-based authorization**: `AgentId`, `BranchId`, `Role` (`Agent`, `Supervisor`, `Admin`)

### 4.2 Endpoints Auth

#### `POST /api/v1/auth/login`
**Request DTO**:
```csharp
public record LoginRequest(
    [property: Required] string Username,
    [property: Required] string Password,
    bool RememberMe);
```
**Business Rules**:
1. Cari agent by `Username` (case-insensitive) dengan `Status == Active`
2. Verifikasi password via `BCrypt.Verify`
3. Jika gagal → increment `FailedLoginAttempts`; jika ≥5 → `Status = Suspended` selama 15 menit
4. Jika sukses → reset attempts, update `LastLoginAt`, generate access+refresh token
5. Return token pair

**Response DTO**:
```csharp
public record LoginResponse(
    Guid AgentId,
    string AccessToken,
    string RefreshToken,
    DateTimeOffset ExpiresAt,
    string ReferralCode,
    string FullName);
```
**Errors**: `401 InvalidCredentials`, `423 AccountLocked`

#### `POST /api/v1/auth/refresh`
```csharp
public record RefreshRequest(string AccessToken, string RefreshToken);
```
Validasi refresh token aktif, rotate, return token baru. Invalidasi token lama.

#### `POST /api/v1/auth/logout`
Revoke aktif refresh token berdasarkan `AgentId` dari claim. Idempotent.

#### `POST /api/v1/auth/forgot-password`
Trigger workflow reset password (kirim OTP ke email/phone). Di luar lingkup penuh PRD ini — endpoint stub.

### 4.3 Authorization Policies
```csharp
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("AgentOnly", p => p.RequireRole("Agent"));
    opts.AddPolicy("SupervisorOrAdmin", p => p.RequireRole("Supervisor", "Admin"));
    opts.AddPolicy("OwnerOrSupervisor", p => p.Requirements.Add(new OwnerOrSupervisorRequirement()));
});
```
Custom handler `OwnerOrSupervisorHandler` memeriksa apakah `AgentId` di route sama dengan claim `AgentId`, atau role `Supervisor`/`Admin`.

---

## 5. API Endpoints

> Semua endpoint (kecuali `/auth/login`, `/auth/refresh`) memerlukan header `Authorization: Bearer {token}`.
> Response sukses menggunakan `200 OK` / `201 Created`. Error menggunakan RFC 7807.

### 5.1 Agent / Profile

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `GET` | `/api/v1/agents/me` | Profil agen login + referral code | Agent |
| `PUT` | `/api/v1/agents/me` | Update profil (fullName, email, phone) | Agent |
| `GET` | `/api/v1/agents/me/stats` | Statistik bulan berjalan (jumlah prospek, kehadiran %) | Agent |
| `GET` | `/api/v1/agents/me/prospects/monthly?year={year}` | Agregat prospek per bulan untuk chart | Agent |
| `POST` | `/api/v1/agents/me/referral/copy` | Log event copy referral (no state change) | Agent |

#### `GET /api/v1/agents/me` Response
```csharp
public record AgentProfileDto(
    Guid Id,
    string Username,
    string FullName,
    string Email,
    string PhoneNumber,
    string ReferralCode,
    string Role,
    string Status,
    string BranchName);
```

#### `GET /api/v1/agents/me/prospects/monthly?year=2026` Response
```csharp
public record MonthlyProspectDto(string Month, int Count, bool IsCurrentMonth);
public record MonthlyProspectChartResponse(int Year, int Total, List<MonthlyProspectDto> Months);
```
**Logic**: Group `Prospects` by `VisitedAt` (month, WIB), hitung count per bulan, tandai bulan berjalan. Cache 5 menit per `AgentId`.

### 5.2 Attendance (Absensi)

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `GET` | `/api/v1/attendance/today` | Status absen hari ini | Agent |
| `POST` | `/api/v1/attendance/clock-in` | Clock in (GPS + selfie) | Agent |
| `POST` | `/api/v1/attendance/clock-out` | Clock out | Agent |
| `GET` | `/api/v1/attendance/history?from={date}&to={date}&page={n}` | Riwayat absen (paginated) | Agent |
| `GET` | `/api/v1/attendance/{id}` | Detail absen | OwnerOrSupervisor |

#### `POST /api/v1/attendance/clock-in`
**Request DTO** (`multipart/form-data`):
```csharp
public class ClockInRequest
{
    [Required] public double Latitude { get; set; }
    [Required] public double Longitude { get; set; }
    [Required] public IFormFile SelfiePhoto { get; set; }
    public double? GpsAccuracyMeters { get; set; }
}
```
**Business Rules (Service: `AttendanceService.ClockInAsync`)**:
1. Cek apakah sudah ada `Attendance` hari ini untuk agent → jika sudah clock-in → `409 Conflict`
2. Ambil `Branch` agent, hitung jarak haversine `agent location ↔ branch coordinate`
3. Jika jarak > `GeoFenceRadiusMeters` (default 200m) → `400 OutsideGeoFence`
4. Jika `GpsAccuracyMeters > 100m` → `400 LowGpsAccuracy`
5. Upload selfie ke blob storage → dapat URL
6. Tentukan status: `OnTime` jika `ClockInAt <= ShiftStart + 5m`, sebaliknya `Late`
7. Persist `Attendance`, return DTO

**Response**:
```csharp
public record ClockInResponse(
    Guid AttendanceId,
    DateTimeOffset ClockInAt,
    string BranchName,
    string Status,             // "OnTime" | "Late"
    string SelfiePhotoUrl);
```

#### `POST /api/v1/attendance/clock-out`
Mirror clock-in; validasi: harus sudah clock-in, belum clock-out. Jika `ClockOutAt > ShiftEnd + 30m` → tampilkan warning (non-blocking).

#### `GET /api/v1/attendance/today`
```csharp
public record TodayAttendanceDto(
    bool HasClockIn,
    bool HasClockOut,
    DateTimeOffset? ClockInAt,
    DateTimeOffset? ClockOutAt,
    TimeSpan? DurationRunning,
    string ShiftLabel,           // "Reguler · 08:00–17:00 WIB"
    string Status);
```

#### `GET /api/v1/attendance/history`
**Query**: `?from=2026-08-01&to=2026-09-30&page=1&pageSize=20`
**Response**: `PagedResult<AttendanceHistoryItemDto>` dengan field `Date`, `ClockIn`, `ClockOut`, `Duration`, `Status`, `StatusClass`.

### 5.3 Prospects (Prospek Toko)

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `GET` | `/api/v1/prospects?date={date}&page={n}&search={q}` | List prospek (filter & search) | Agent (own) / Supervisor (all) |
| `GET` | `/api/v1/prospects/today` | List prospek hari ini + count | Agent |
| `GET` | `/api/v1/prospects/{id}` | Detail prospek | OwnerOrSupervisor |
| `POST` | `/api/v1/prospects` | Buat prospek baru | Agent |
| `PUT` | `/api/v1/prospects/{id}` | Update prospek (jika status = New) | Owner |
| `DELETE` | `/api/v1/prospects/{id}` | Hapus prospek (soft delete, jika New) | Owner |
| `GET` | `/api/v1/prospects/{id}/photos/{kind}` | Stream foto (`plang`/`selfie`) | OwnerOrSupervisor |
| `GET` | `/api/v1/prospects/{id}/photos/{kind}/download` | Force-download foto | OwnerOrSupervisor |

#### `POST /api/v1/prospects` (`multipart/form-data`)
**Request DTO**:
```csharp
public class CreateProspectRequest
{
    [Required, MaxLength(150)] public string StoreName { get; set; }
    [Required, MaxLength(500)] public string Address { get; set; }
    [Required, MaxLength(100)] public string PicName { get; set; }
    [Required, Phone] public string PicPhone { get; set; }
    [MaxLength(1000)] public string Notes { get; set; }
    [Required] public double Latitude { get; set; }
    [Required] public double Longitude { get; set; }
    public double? GpsAccuracyMeters { get; set; }
    [Required] public IFormFile PlangPhoto { get; set; }
    [Required] public IFormFile SelfiePhoto { get; set; }
}
```
**Business Rules (`ProspectService.CreateAsync`)**:
1. **Prerequisite check**: agent harus sudah `ClockIn` hari ini (cek `Attendance.Today`). Jika belum → `409 AttendanceRequired` (sesuai mockup: "Absen wajib dilakukan sebelum memulai kunjungan prospek.")
2. **Validation**: FluentValidation — phone minimal 9 digit, file size ≤5MB, MIME `image/jpeg`|`image/png`
3. **Upload**: kedua foto ke blob dengan path `prospects/{agentId}/{yyyy-MM-dd}/{guid}_{kind}.jpg`. Generate SAS URL read-only.
4. **Persist**: simpan record dengan `VisitedAt = UtcNow`, `Status = New`, `AgentId` dari claim
5. **Notification**: enqueue `ProspectSaved` notification + push (jika terdaftar)
6. **Return** `201 Created` dengan lokasi detail

**Response**:
```csharp
public record ProspectDto(
    Guid Id,
    string StoreName,
    string Address,
    string PicName,
    string PicPhone,
    string Notes,
    double Latitude,
    double Longitude,
    DateTimeOffset VisitedAt,
    string PlangPhotoUrl,
    string SelfiePhotoUrl,
    string Status,
    string AgentName);
```

#### `GET /api/v1/prospects?search=...&date=...`
**Search fields**: `StoreName`, `Address`, `PicName` (case-insensitive contains)
**Filter**: by date (`VisitedAt` WIB), optional
**Pagination**: `page=1&pageSize=20`, max 100
**Response**: `PagedResult<ProspectListItemDto>` dengan thumbnail URLs.

#### `GET /api/v1/prospects/{id}/photos/{kind}`
`kind` ∈ {`plang`, `selfie`}. Return `FileStreamResult` dengan `Content-Type: image/jpeg`. Validasi ownership sebelum stream.

#### `GET /api/v1/prospects/{id}/photos/{kind}/download`
Set `Content-Disposition: attachment; filename="Prospek_{StoreName}_{kind}.jpg"`. Stream blob.

### 5.4 Notifications

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `GET` | `/api/v1/notifications?page={n}` | List notifikasi (paginated, group by date) | Agent |
| `GET` | `/api/v1/notifications/unread-count` | Hitung belum dibaca | Agent |
| `PUT` | `/api/v1/notifications/{id}/read` | Tandai satu dibaca | Agent |
| `PUT` | `/api/v1/notifications/read-all` | Tandai semua dibaca | Agent |

#### `GET /api/v1/notifications` Response
```csharp
public record NotificationDto(
    Guid Id,
    string Title,
    string Body,
    string Type,             // string dari enum
    string IconColor,        // "green" | "orange" | "blue" | "red"
    bool IsUnread,
    DateTimeOffset CreatedAt,
    string CreatedAtLabel);  // "12:05 WIB" / "Kemarin 09:15 WIB"
public record NotificationGroupDto(string DateLabel, List<NotificationDto> Items);
```
**Logic**: Group by `CreatedAt` (WIB) → "Hari Ini", "Kemarin", atau tanggal lengkap.

### 5.5 Master Data

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `GET` | `/api/v1/branches/current` | Branch agent login (untuk gate screen) | Agent |
| `GET` | `/api/v1/branches/{id}` | Detail branch | Any authenticated |

### 5.6 Health & Misc

| Method | Endpoint | Deskripsi |
|---|---|---|
| `GET` | `/health` | Liveness probe |
| `GET` | `/health/ready` | Readiness (DB + Blob + Redis) |
| `GET` | `/api/v1/version` | `{ "version": "1.0.0", "build": "2026.09.04" }` |

---

## 6. Controllers

### 6.1 Struktur
```csharp
[ApiController]
[Route("api/v1/[controller]")]
[Authorize]
public class ProspectsController : ControllerBase
{
    private readonly IProspectService _service;
    private readonly ILogger<ProspectsController> _logger;

    [HttpGet]
    public async Task<ActionResult<PagedResult<ProspectListItemDto>>> Get(
        [FromQuery] ProspectQuery query, CancellationToken ct) { ... }

    [HttpGet("today")]
    public async Task<ActionResult<TodayProspectsDto>> GetToday(CancellationToken ct) { ... }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ProspectDto>> GetById(Guid id, CancellationToken ct) { ... }

    [HttpPost]
    [Consumes("multipart/form-data")]
    [RequestSizeLimit(20_000_000)]  // 20MB total
    public async Task<ActionResult<ProspectDto>> Create(
        [FromForm] CreateProspectRequest req, CancellationToken ct) { ... }

    [HttpGet("{id:guid}/photos/{kind}")]
    public async Task<IActionResult> GetPhoto(Guid id, string kind, CancellationToken ct) { ... }

    [HttpGet("{id:guid}/photos/{kind}/download")]
    public async Task<IActionResult> DownloadPhoto(Guid id, string kind, CancellationToken ct) { ... }
}
```

### 6.2 Daftar Controllers
1. `AuthController` — login, refresh, logout, forgot-password
2. `AgentsController` — `/me`, stats, monthly chart, referral
3. `AttendanceController` — today, clock-in/out, history
4. `ProspectsController` — CRUD + photos
5. `NotificationsController` — list, read, read-all, unread-count
6. `BranchesController` — current, by-id
7. `HealthController` — health/ready/version

---

## 7. Services (Application Layer)

### 7.1 Interfaces
```csharp
public interface IAuthService
{
    Task<LoginResponse> LoginAsync(LoginRequest req, string ip, CancellationToken ct);
    Task<RefreshResponse> RefreshAsync(RefreshRequest req, string ip, CancellationToken ct);
    Task LogoutAsync(Guid agentId, CancellationToken ct);
}

public interface IAttendanceService
{
    Task<TodayAttendanceDto> GetTodayAsync(Guid agentId, CancellationToken ct);
    Task<ClockInResponse> ClockInAsync(Guid agentId, ClockInRequest req, CancellationToken ct);
    Task<ClockOutResponse> ClockOutAsync(Guid agentId, CancellationToken ct);
    Task<PagedResult<AttendanceHistoryItemDto>> GetHistoryAsync(
        Guid agentId, DateOnly from, DateOnly to, int page, int pageSize, CancellationToken ct);
}

public interface IProspectService
{
    Task<PagedResult<ProspectListItemDto>> SearchAsync(
        Guid agentId, ProspectQuery query, CancellationToken ct);
    Task<TodayProspectsDto> GetTodayAsync(Guid agentId, CancellationToken ct);
    Task<ProspectDto> GetByIdAsync(Guid id, Guid requesterId, string role, CancellationToken ct);
    Task<ProspectDto> CreateAsync(Guid agentId, CreateProspectRequest req, CancellationToken ct);
    Task<ProspectDto> UpdateAsync(Guid id, Guid agentId, UpdateProspectRequest req, CancellationToken ct);
    Task DeleteAsync(Guid id, Guid agentId, CancellationToken ct);
}

public interface INotificationService
{
    Task<PagedResult<NotificationGroupDto>> GetAsync(Guid agentId, int page, int pageSize, CancellationToken ct);
    Task<int> GetUnreadCountAsync(Guid agentId, CancellationToken ct);
    Task MarkAsReadAsync(Guid id, Guid agentId, CancellationToken ct);
    Task MarkAllAsReadAsync(Guid agentId, CancellationToken ct);
    Task EnqueueAsync(Guid agentId, NotificationType type, string title, string body, string payloadJson = null);
}

public interface IFileStorageService
{
    Task<string> UploadAsync(Stream stream, string blobName, string contentType, CancellationToken ct);
    Task<Stream> DownloadAsync(string blobName, CancellationToken ct);
    Task<string> GetSignedReadUrlAsync(string blobName, TimeSpan ttl);
    Task DeleteAsync(string blobName, CancellationToken ct);
}

public interface IGeoService
{
    double HaversineMeters(double lat1, double lng1, double lat2, double lng2);
    bool IsWithinRadius(double lat1, double lng1, double lat2, double lng2, double radiusMeters);
}

public interface IAgentStatsService
{
    Task<MonthlyProspectChartResponse> GetMonthlyAsync(Guid agentId, int year, CancellationToken ct);
    Task<AgentStatsDto> GetCurrentMonthStatsAsync(Guid agentId, CancellationToken ct);
}
```

### 7.2 Business Logic Highlights

#### `AttendanceService.ClockInAsync`
1. Load `agent` with `Branch`
2. `attendance = await repo.GetTodayAsync(agentId)` → if exists & `ClockInAt != null` → throw `ConflictException("AlreadyClockedIn")`
3. `distance = _geo.HaversineMeters(req.Latitude, req.Longitude, branch.Latitude, branch.Longitude)`
4. if `distance > branch.GeoFenceRadiusMeters` → throw `DomainException("OutsideGeoFence", Details: { distance, allowed })`
5. if `req.GpsAccuracyMeters > 100` → throw `DomainException("LowGpsAccuracy")`
6. `selfieUrl = await _file.UploadAsync(req.SelfiePhoto.OpenReadStream(), $"attendance/{agentId}/{date}/{guid}.jpg", "image/jpeg")`
7. `attendance.ClockInAt = DateTimeOffset.UtcNow`
8. `attendance.Status = (localTime > shiftStart.AddMinutes(5)) ? Late : OnTime`
9. `await repo.SaveChangesAsync(ct)`
10. `await _notification.EnqueueAsync(agentId, NotificationType.ClockIn, "Clock in tercatat", ...)`
11. Return mapped DTO

#### `ProspectService.CreateAsync`
1. `attendance = await _attendance.GetTodayAsync(agentId)` → if null or `ClockInAt == null` → throw `ConflictException("AttendanceRequired")`
2. Validate via `CreateProspectValidator`
3. Generate blob names: `prospects/{agentId}/{yyyyMMdd}/{guid}_plang.jpg` and `..._selfie.jpg`
4. Upload both in parallel (`Task.WhenAll`)
5. Persist `Prospect` entity
6. `await _notification.EnqueueAsync(agentId, ProspectSaved, "Prospek tersimpan", $"Data {req.StoreName} berhasil masuk ke sistem.")`
7. Return `ProspectDto`

#### `AgentStatsService.GetMonthlyAsync`
```csharp
var data = await _repo.Prospects
    .Where(p => p.AgentId == agentId && p.VisitedAt.Year == year)
    .GroupBy(p => p.VisitedAt.Month)
    .Select(g => new { Month = g.Key, Count = g.Count() })
    .ToListAsync(ct);

// Map to month names (Jan-Sep ...), fill 0 for missing months, mark current
```

---

## 8. DTOs (Lengkap)

### 8.1 Auth
```csharp
public record LoginRequest(string Username, string Password, bool RememberMe);
public record LoginResponse(Guid AgentId, string AccessToken, string RefreshToken,
                            DateTimeOffset ExpiresAt, string ReferralCode, string FullName);
public record RefreshRequest(string AccessToken, string RefreshToken);
public record RefreshResponse(string AccessToken, string RefreshToken, DateTimeOffset ExpiresAt);
```

### 8.2 Attendance
```csharp
public record ClockInRequest(double Latitude, double Longitude, IFormFile SelfiePhoto, double? GpsAccuracyMeters);
public record ClockOutRequest(double Latitude, double Longitude, IFormFile? SelfiePhoto, double? GpsAccuracyMeters);
public record ClockInResponse(Guid AttendanceId, DateTimeOffset ClockInAt, string BranchName,
                              string Status, string SelfiePhotoUrl);
public record TodayAttendanceDto(bool HasClockIn, bool HasClockOut,
                                  DateTimeOffset? ClockInAt, DateTimeOffset? ClockOutAt,
                                  TimeSpan? DurationRunning, string ShiftLabel, string Status);
public record AttendanceHistoryItemDto(DateOnly Date, string DayName, string ClockIn,
                                       string ClockOut, string Duration, string Status, string StatusClass);
```

### 8.3 Prospect
```csharp
public record CreateProspectRequest(string StoreName, string Address, string PicName,
                                     string PicPhone, string Notes, double Latitude,
                                     double Longitude, double? GpsAccuracyMeters,
                                     IFormFile PlangPhoto, IFormFile SelfiePhoto);
public record ProspectListItemDto(Guid Id, string StoreName, string Address, string PicName,
                                  string VisitedAtLabel, string ThumbnailUrl, string Status,
                                  string StatusClass);
public record ProspectDto(Guid Id, string StoreName, string Address, string PicName,
                          string PicPhone, string Notes, double Latitude, double Longitude,
                          DateTimeOffset VisitedAt, string PlangPhotoUrl, string SelfiePhotoUrl,
                          string Status, string AgentName);
public record TodayProspectsDto(int Count, List<ProspectListItemDto> Items);
```

### 8.4 Notifications & Stats
```csharp
public record NotificationDto(Guid Id, string Title, string Body, string Type,
                              string IconColor, bool IsUnread, DateTimeOffset CreatedAt, string CreatedAtLabel);
public record NotificationGroupDto(string DateLabel, List<NotificationDto> Items);
public record MonthlyProspectDto(string Month, int Count, bool IsCurrentMonth);
public record MonthlyProspectChartResponse(int Year, int Total, List<MonthlyProspectDto> Months);
public record AgentStatsDto(int ProspectsThisMonth, double AttendanceRate, double GrowthPercent);
```

### 8.5 Generic
```csharp
public record PagedResult<T>(List<T> Items, int Page, int PageSize, int TotalCount, int TotalPages);
public record ErrorResponse(string Type, string Title, int Status, string Detail, string Instance, Dictionary<string,object> Errors);
```

---

## 9. Validation (FluentValidation)

### 9.1 `CreateProspectValidator`
```csharp
public class CreateProspectValidator : AbstractValidator<CreateProspectRequest>
{
    public CreateProspectValidator()
    {
        RuleFor(x => x.StoreName).NotEmpty().MaximumLength(150);
        RuleFor(x => x.Address).NotEmpty().MaximumLength(500);
        RuleFor(x => x.PicName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.PicPhone).NotEmpty().Matches(@"^(\+62|62|0)8[1-9]\d{6,11}$")
            .WithMessage("Nomor telepon PIC tidak valid.");
        RuleFor(x => x.Notes).MaximumLength(1000);
        RuleFor(x => x.Latitude).InclusiveBetween(-90, 90);
        RuleFor(x => x.Longitude).InclusiveBetween(-180, 180);
        RuleFor(x => x.PlangPhoto).NotNull()
            .Must(f => f.Length <= 5_000_000).WithMessage("Ukuran foto plang maksimal 5MB.")
            .Must(f => f.ContentType is "image/jpeg" or "image/png");
        RuleFor(x => x.SelfiePhoto).NotNull()
            .Must(f => f.Length <= 5_000_000).WithMessage("Ukuran foto selfie maksimal 5MB.")
            .Must(f => f.ContentType is "image/jpeg" or "image/png");
    }
}
```

### 9.2 `ClockInValidator`
```csharp
public class ClockInValidator : AbstractValidator<ClockInRequest>
{
    public ClockInValidator()
    {
        RuleFor(x => x.Latitude).InclusiveBetween(-90, 90);
        RuleFor(x => x.Longitude).InclusiveBetween(-180, 180);
        RuleFor(x => x.SelfiePhoto).NotNull().Must(f => f.Length <= 5_000_000);
        RuleFor(x => x.GpsAccuracyMeters).LessThanOrEqualTo(100).When(x => x.GpsAccuracyMeters.HasValue);
    }
}
```

---

## 10. Middleware & Error Handling

### 10.1 Pipeline Order (`Program.cs`)
```csharp
app.UseExceptionHandler(errApp => errApp.UseExceptionHandlerMiddleware());
app.UseHttpsRedirection();
app.UseSerilogRequestLogging();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

### 10.2 Exception → HTTP Mapping
| Exception | HTTP | Code |
|---|---|---|
| `NotFoundException` | 404 | `NotFound` |
| `ConflictException` | 409 | `Conflict` |
| `DomainException` | 400 | custom code |
| `ValidationException` | 422 | `ValidationFailed` |
| `UnauthorizedException` | 401 | `Unauthorized` |
| `ForbiddenException` | 403 | `Forbidden` |
| `RateLimitExceededException` | 429 | `RateLimited` |
| Unhandled | 500 | `InternalError` |

### 10.3 Sample Error Body
```json
{
  "type": "https://salespoint.visionet.co.id/errors/attendance-required",
  "title": "Attendance required",
  "status": 409,
  "detail": "Absen wajib dilakukan sebelum memulai kunjungan prospek.",
  "instance": "/api/v1/prospects",
  "code": "AttendanceRequired",
  "errors": {}
}
```

---

## 11. Security

### 11.1 Transport & Headers
- Force HTTPS; redirect HTTP
- HSTS di production
- Security headers via `NetEscapades.AspNetCore.SecurityHeaders`: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Content-Security-Policy: default-src 'none'`

### 11.2 Rate Limiting
| Endpoint | Limit |
|---|---|
| `POST /auth/login` | 10 / menit / IP |
| `POST /auth/refresh` | 30 / menit / IP |
| `POST /prospects` | 60 / jam / agent |
| `POST /attendance/clock-in` | 10 / hari / agent |
| Other read endpoints | 300 / menit / agent |

### 11.3 File Upload Security
- Validate MIME via content sniffing (`MimeDetective`), bukan hanya `ContentType` header
- Max file size 5MB per foto, 20MB total per request
- Strip EXIF untuk photo plang (privacy), retain untuk selfie (timestamp + GPS untuk audit)
- Generate SAS URL read-only (TTL 1 jam) — tidak ekspos blob storage langsung

### 11.4 PII Handling
- `PicPhone` disimpan sebagai E.164, di-mask (`0812 **** 7890`) di response publik/list