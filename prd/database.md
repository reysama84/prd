# Database Product Requirements Document (PRD)
## Sales Point — VisioNet Mini ATM Agent Acquisition App

---

## 1. Overview

This document defines the complete database schema for the **Sales Point** mobile application — a field-sales tool used by agents to acquire Mini ATM merchant prospects, track daily attendance via GPS+Selfie, and manage visit documentation with photo evidence.

**Target Database:** PostgreSQL 14+  
**Naming Convention:** `snake_case` for all identifiers  
**Character Encoding:** UTF-8  
**Timezone:** All timestamps stored as `TIMESTAMPTZ` in UTC; display-layer converts to WIB (Asia/Jakarta, UTC+7)

---

## 2. Design Conventions

| Convention | Rule |
|---|---|
| Primary Keys | `id BIGSERIAL` (BIGINT, auto-increment) |
| Foreign Keys | `{entity}_id BIGINT` referencing parent `id` |
| Timestamps | `created_at` / `updated_at` as `TIMESTAMPTZ DEFAULT now()` |
| Soft Deletes | Not used in v1.0; data is retained for audit |
| Status Fields | `VARCHAR` with `CHECK` constraint enumerating allowed values |
| Monetary/Precision | `NUMERIC(10,2)` for distances, `NUMERIC(10,7)` for coordinates |
| Photo Storage | File paths stored in DB; binary files stored on object storage (S3/MinIO) |
| Indexes | Every FK column indexed; composite indexes on common query patterns |

---

## 3. Schema Overview (Entity Relationship)

```
branches (1) ───────────< (N) users
                              │
                              │ (1)
                              │
              ┌───────────────┼───────────────┐
              │ (1)           │ (1)           │ (1)
              ▼               ▼               ▼
    user_shift_assignments  attendances    prospects
              │ (N)           │ (1)           │ (1)
              │               │               │
      shifts (1)             │               ▼
                              │       prospect_status_history (N)
                              │
                              │
                          users (1)
                              │
                              ├──< notifications (N)
                              ├──< login_sessions (N)
                              └──< prospect_status_history (changed_by)

app_config (standalone)
```

---

## 4. Table Definitions

### 4.1 `branches` — Office Branches / Work Areas

Stores physical office branches where agents are assigned and where attendance GPS geofencing is validated.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `code` | VARCHAR(20) | NOT NULL, UNIQUE | Branch code (e.g. `JKT-SEL`) |
| `name` | VARCHAR(100) | NOT NULL | Display name (e.g. "Kantor Cabang Jakarta Selatan") |
| `address` | TEXT | NOT NULL | Full street address |
| `latitude` | NUMERIC(10,7) | NOT NULL, CHECK (lat BETWEEN -90 AND 90) | Branch GPS latitude |
| `longitude` | NUMERIC(10,7) | NOT NULL, CHECK (lng BETWEEN -180 AND 180) | Branch GPS longitude |
| `radius_meters` | INTEGER | NOT NULL DEFAULT 100 | Geofence radius for valid clock-in (meters) |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | Soft-deactivation flag |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `uk_branches_code` | `code` | UNIQUE (btree) | Ensure unique branch codes |
| `ix_branches_active` | `is_active` | btree | Filter active branches |

---

### 4.2 `users` — Sales Agents & Staff

Stores all application users. Agents use the mobile app; supervisors/admins may use a web dashboard.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `username` | VARCHAR(50) | NOT NULL, UNIQUE | Login username (e.g. `rizky.pratama`) |
| `password_hash` | VARCHAR(255) | NOT NULL | Bcrypt/Argon2 hash |
| `full_name` | VARCHAR(100) | NOT NULL | Legal full name (e.g. "Rizky Pratama Nugroho") |
| `email` | VARCHAR(150) | NOT NULL, UNIQUE | Corporate email |
| `referral_code` | VARCHAR(20) | NOT NULL, UNIQUE | Agent referral code (e.g. `SP-RZK2041`) |
| `role` | VARCHAR(20) | NOT NULL DEFAULT 'agent', CHECK (role IN ('agent','supervisor','admin')) | User role |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active', CHECK (status IN ('active','inactive','suspended')) | Account status |
| `branch_id` | BIGINT | FK → branches(id) ON DELETE SET NULL, nullable | Assigned branch |
| `avatar_initials` | VARCHAR(4) | nullable | Cached initials for UI avatar (e.g. "RP") |
| `last_login_at` | TIMESTAMPTZ | nullable | Last successful login timestamp |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `uk_users_username` | `username` | UNIQUE (btree) | Login lookup |
| `uk_users_email` | `email` | UNIQUE (btree) | Email lookup |
| `uk_users_referral_code` | `referral_code` | UNIQUE (btree) | Referral code lookup |
| `ix_users_branch_id` | `branch_id` | btree | Filter users by branch |
| `ix_users_status` | `status` | btree | Filter active users |

---

### 4.3 `shifts` — Work Shift Definitions

Defines work shift schedules used for attendance validation.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `name` | VARCHAR(50) | NOT NULL | Shift name (e.g. "Reguler") |
| `start_time` | TIME | NOT NULL | Shift start (e.g. `08:00`) |
| `end_time` | TIME | NOT NULL | Shift end (e.g. `17:00`) |
| `grace_period_minutes` | INTEGER | NOT NULL DEFAULT 15 | Minutes after start_time before flagged "terlambat" |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `ix_shifts_active` | `is_active` | btree | Filter active shifts |

---

### 4.4 `user_shift_assignments` — Shift Assignment History

Tracks which shift is assigned to a user over time, allowing historical accuracy.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `user_id` | BIGINT | NOT NULL, FK → users(id) ON DELETE CASCADE | Assigned user |
| `shift_id` | BIGINT | NOT NULL, FK → shifts(id) ON DELETE RESTRICT | Assigned shift |
| `effective_from` | DATE | NOT NULL | Start date of assignment |
| `effective_to` | DATE | nullable | End date (NULL = currently active) |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraints:**
- CHECK (`effective_to` IS NULL OR `effective_to` >= `effective_from`)
- Only one active assignment per user (enforced via partial unique index)

**Indexes:**
| Name | Columns / Condition | Type | Purpose |
|---|---|---|---|
| `ix_usa_user_id` | `user_id` | btree | Lookup by user |
| `ix_usa_user_effective` | `user_id, effective_from DESC` | btree | Find current assignment |
| `uk_usa_active_per_user` | `user_id` WHERE `effective_to IS NULL` | UNIQUE (partial) | One active shift per user |

---

### 4.5 `attendances` — Daily Attendance Records

One row per user per day. Created on clock-in; updated on clock-out.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `user_id` | BIGINT | NOT NULL, FK → users(id) ON DELETE CASCADE | |
| `attendance_date` | DATE | NOT NULL | The work date (not timestamp) |
| `shift_id` | BIGINT | NOT NULL, FK → shifts(id) ON DELETE RESTRICT | Shift applicable on that date |
| `clock_in_at` | TIMESTAMPTZ | nullable | Actual clock-in time |
| `clock_out_at` | TIMESTAMPTZ | nullable | Actual clock-out time |
| `clock_in_latitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -90 AND 90) | GPS latitude at clock-in |
| `clock_in_longitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -180 AND 180) | GPS longitude at clock-in |
| `clock_in_accuracy_m` | NUMERIC(8,2) | nullable, CHECK (>= 0) | GPS accuracy in meters (e.g. 8.0) |
| `clock_in_address` | TEXT | nullable | Reverse-geocoded address at clock-in |
| `clock_in_photo_path` | VARCHAR(500) | nullable | Selfie photo path on object storage |
| `clock_out_latitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -90 AND 90) | GPS latitude at clock-out |
| `clock_out_longitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -180 AND 180) | GPS longitude at clock-out |
| `clock_out_accuracy_m` | NUMERIC(8,2) | nullable, CHECK (>= 0) | GPS accuracy at clock-out |
| `clock_out_address` | TEXT | nullable | Reverse-geocoded address at clock-out |
| `clock_out_photo_path` | VARCHAR(500) | nullable | Selfie photo path at clock-out |
| `duration_minutes` | INTEGER | nullable, CHECK (>= 0) | Computed: clock_out - clock_in in minutes |
| `status` | VARCHAR(20) | nullable, CHECK (status IN ('tepat_waktu','terlambat','izin','sakit','alpha')) | Attendance classification |
| `method` | VARCHAR(20) | NOT NULL DEFAULT 'gps_selfie', CHECK (method IN ('gps_selfie','manual')) | Verification method |
| `notes` | TEXT | nullable | Free-text notes (e.g. "Izin sakit") |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraints:**
- UNIQUE (`user_id`, `attendance_date`) — one attendance record per user per day
- CHECK (`clock_out_at` IS NULL OR `clock_out_at` > `clock_in_at`)
- CHECK (`duration_minutes` IS NULL OR (`clock_in_at` IS NOT NULL AND `clock_out_at` IS NOT NULL))

**Business Rules (enforced via trigger):**
- On `clock_out_at` insert/update → compute `duration_minutes` = `EXTRACT(EPOCH FROM (clock_out_at - clock_in_at)) / 60`
- On `clock_in_at` insert → determine `status` = 'tepat_waktu' if clock_in <= shift_start + grace_period, else 'terlambat'

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `uk_attendances_user_date` | `user_id, attendance_date` | UNIQUE (btree) | Prevent duplicate daily records |
| `ix_attendances_user_date_desc` | `user_id, attendance_date DESC` | btree | History list query |
| `ix_attendances_date` | `attendance_date` | btree | Date-range queries |
| `ix_attendances_status` | `status` | btree | Filter by status |

---

### 4.6 `prospects` — Store Prospect Visit Records

Each row represents a single prospect store visit by an agent during a field day.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `user_id` | BIGINT | NOT NULL, FK → users(id) ON DELETE CASCADE | Visiting agent |
| `store_name` | VARCHAR(200) | NOT NULL | Name of the store (e.g. "Toko Berkah Jaya") |
| `store_address` | TEXT | NOT NULL | Full store address |
| `pic_name` | VARCHAR(100) | NOT NULL | Person in charge at store |
| `pic_phone` | VARCHAR(20) | NOT NULL, CHECK (LENGTH(REGEXP_REPLACE(pic_phone, '\D', '', 'g')) >= 9) | PIC phone number |
| `notes` | TEXT | nullable | Visit notes (e.g. "Pemilik tertarik, minta dihubungi...") |
| `latitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -90 AND 90) | GPS latitude at visit |
| `longitude` | NUMERIC(10,7) | nullable, CHECK (BETWEEN -180 AND 180) | GPS longitude at visit |
| `location_accuracy_m` | NUMERIC(8,2) | nullable, CHECK (>= 0) | GPS accuracy in meters |
| `location_address` | TEXT | nullable | Reverse-geocoded address |
| `visit_number` | INTEGER | nullable, CHECK (>= 1) | Sequential visit number for the day (e.g. 5th visit) |
| `visited_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | Actual timestamp of visit |
| `visit_date` | DATE | NOT NULL | Date-only for grouping (derived from visited_at) |
| `plang_photo_path` | VARCHAR(500) | NOT NULL | Object-storage path for store-sign photo |
| `selfie_photo_path` | VARCHAR(500) | NOT NULL | Object-storage path for selfie-with-PIC photo |
| `plang_photo_metadata` | JSONB | nullable | Stamp data embedded on photo: `{date, time, coords, address}` |
| `selfie_photo_metadata` | JSONB | nullable | Stamp data for selfie photo |
| `status` | VARCHAR(30) | NOT NULL DEFAULT 'menunggu_verifikasi', CHECK (status IN ('menunggu_verifikasi','terverifikasi','ditolak')) | Verification status |
| `verified_at` | TIMESTAMPTZ | nullable | When status was set to verified/rejected |
| `verified_by` | BIGINT | nullable, FK → users(id) ON DELETE SET NULL | Supervisor who verified |
| `rejection_reason` | TEXT | nullable | Required when status = 'ditolak' |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraints:**
- CHECK (`status` = 'ditolak' AND `rejection_reason` IS NOT NULL) OR (`status` != 'ditolak')
  — i.e., rejection_reason mandatory when status is 'ditolak'

**Business Rules (enforced via trigger):**
- On insert → compute `visit_date` = DATE(`visited_at`)
- On insert → compute `visit_number` = COUNT(prospects WHERE user_id = NEW.user_id AND visit_date = NEW.visit_date) + 1
- On status change to 'terverifikasi' or 'ditolak' → set `verified_at` = now()

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `ix_prospects_user_id` | `user_id` | btree | Filter by agent |
| `ix_prospects_visit_date` | `visit_date` | btree | Date-range queries |
| `ix_prospects_user_date_desc` | `user_id, visit_date DESC` | btree | History list per agent |
| `ix_prospects_status` | `status` | btree | Filter by verification status |
| `ix_prospects_store_name` | `store_name` | btree (pattern) | Search by store name (LIKE) |
| `ix_prospects_pic_name` | `pic_name` | btree (pattern) | Search by PIC name |
| `ix_prospects_pic_phone` | `pic_phone` | btree | Search by phone |
| `ix_prospects_store_address_gin` | `to_tsvector('indonesian', store_address)` | GIN | Full-text search on address |
| `ix_prospects_verified_by` | `verified_by` | btree | Lookup by verifier |

---

### 4.7 `prospect_status_history` — Status Change Audit Log

Records every status transition for a prospect, providing an audit trail.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `prospect_id` | BIGINT | NOT NULL, FK → prospects(id) ON DELETE CASCADE | Related prospect |
| `previous_status` | VARCHAR(30) | nullable, CHECK (IN same enum as prospects.status) | Status before change |
| `new_status` | VARCHAR(30) | NOT NULL, CHECK (IN same enum as prospects.status) | Status after change |
| `changed_by` | BIGINT | NOT NULL, FK → users(id) ON DELETE RESTRICT | User who made the change |
| `notes` | TEXT | nullable | Optional context (e.g. "Toko tutup permanen") |
| `changed_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | Timestamp of change |

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `ix_psh_prospect_id` | `prospect_id` | btree | Get history for a prospect |
| `ix_psh_changed_by` | `changed_by` | btree | Filter by who made changes |
| `ix_psh_changed_at` | `changed_at DESC` | btree | Chronological sort |

---

### 4.8 `notifications` — In-App Notifications

Stores notifications displayed in the app's notification center.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `user_id` | BIGINT | NOT NULL, FK → users(id) ON DELETE CASCADE | Recipient user |
| `type` | VARCHAR(20) | NOT NULL DEFAULT 'info', CHECK (type IN ('success','reminder','info','warning')) | Notification category |
| `icon_category` | VARCHAR(10) | NOT NULL DEFAULT 'blue', CHECK (icon_category IN ('green','orange','blue','red')) | UI icon color mapping |
| `title` | VARCHAR(200) | NOT NULL | Notification title (e.g. "Prospek tersimpan") |
| `body` | TEXT | NOT NULL | Full notification text |
| `is_read` | BOOLEAN | NOT NULL DEFAULT FALSE | Read/unread flag |
| `read_at` | TIMESTAMPTZ | nullable | When user opened the notification |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | Display timestamp |

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `ix_notif_user_unread` | `user_id, is_read, created_at DESC` | btree | Fetch unread notifications for badge count |
| `ix_notif_user_created` | `user_id, created_at DESC` | btree | Full notification list pagination |
| `ix_notif_created_at` | `created_at DESC` | btree | Cron cleanup of old notifications |

---

### 4.9 `login_sessions` — Authentication Sessions

Manages "remember me" tokens and active session tracking.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `user_id` | BIGINT | NOT NULL, FK → users(id) ON DELETE CASCADE | Session owner |
| `token_hash` | VARCHAR(255) | NOT NULL, UNIQUE | SHA-256 hash of issued JWT/session token |
| `device_info` | TEXT | nullable | User-Agent / device model string |
| `ip_address` | INET | nullable | Login IP address |
| `remember_me` | BOOLEAN | NOT NULL DEFAULT FALSE | Extended expiry if TRUE |
| `expires_at` | TIMESTAMPTZ | NOT NULL | Token expiry (24h normal, 30d if remember_me) |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | Login timestamp |
| `revoked_at` | TIMESTAMPTZ | nullable | Set on logout/token revocation |

**Constraints:**
- CHECK (`expires_at` > `created_at`)

**Indexes:**
| Name | Columns | Type | Purpose |
|---|---|---|---|
| `uk_sessions_token` | `token_hash` | UNIQUE (btree) | Token lookup at auth |
| `ix_sessions_user_id` | `user_id` | btree | List/expire sessions for a user |
| `ix_sessions_expires_at` | `expires_at` WHERE `revoked_at IS NULL` | btree (partial) | Cron cleanup of expired sessions |

---

### 4.10 `app_config` — Application Configuration

Key-value store for application-wide settings.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGSERIAL | PK | Surrogate key |
| `key` | VARCHAR(100) | NOT NULL, UNIQUE | Config key (e.g. `app_version`, `min_client_version`) |
| `value` | TEXT | NOT NULL | Config value |
| `data_type` | VARCHAR(20) | NOT NULL DEFAULT 'string', CHECK (data_type IN ('string','integer','boolean','json')) | Value type hint |
| `description` | TEXT | nullable | Human-readable explanation |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Seed Values:**

| key | value | data_type | description |
|---|---|---|---|
| `app_version` | `1.0.0` | string | Current app version |
| `app_build` | `2026.09.04` | string | Build date identifier |
| `min_client_version` | `1.0.0` | string | Minimum allowed client version |
| `clock_in_early_minutes` | `120` | integer | Minutes before shift start when clock-in is allowed |
| `clock_out_early_minutes` | `30` | integer | Minutes before shift end when clock-out is allowed |
| `default_geofence_radius_m` | `100` | integer | Default branch geofence radius |
| `photo_quality_jpeg` | `0.86` | string | JPEG compression quality for photos |

---

## 5. Views

### 5.1 `v_monthly_prospect_stats` — Monthly Prospect Chart Data

Feeds the bar chart on the Profile screen (Jan–Dec prospect counts per month).

```sql
CREATE VIEW v_monthly_prospect_stats AS
SELECT
    user_id,
    EXTRACT(YEAR FROM visit_date)  AS year,
    EXTRACT(MONTH FROM visit_date) AS month,
    TO_CHAR(visit_date, 'Mon')     AS month_label,
    COUNT(*)                        AS prospect_count,
    CASE
        WHEN EXTRACT(YEAR FROM visit_date)  = EXTRACT(YEAR FROM CURRENT_DATE)
         AND EXTRACT(MONTH FROM visit_date) = EXTRACT(MONTH FROM CURRENT_DATE)
        THEN TRUE ELSE FALSE
    END                             AS is_current_month
FROM prospects
GROUP BY user_id, year, month, month_label, visit_date
ORDER BY user_id, year, month;
```

### 5.2 `v_attendance_summary` — Monthly Attendance Summary

Feeds the profile screen's attendance stat (e.g. "4 hari · 100%").

```sql
CREATE VIEW v_attendance_summary AS
SELECT
    user_id,
    DATE_TRUNC('month', attendance_date) AS month,
    COUNT(*) FILTER (WHERE status IN ('tepat_waktu','terlambat')) AS days_present,
    COUNT(*) FILTER (WHERE status = 'tepat_waktu')                AS days_on_time,
    COUNT(*) FILTER (WHERE status IN ('izin','sakit'))           AS days_leave,
    COUNT(*)                                                      AS total_records,
    ROUND(
        COUNT(*) FILTER (WHERE status = 'tepat_waktu')::NUMERIC
        / NULLIF(COUNT(*) FILTER (WHERE status IN ('tepat_waktu','terlambat')), 0) * 100,
        1
    ) AS on_time_percentage
FROM attendances
GROUP BY user_id, DATE_TRUNC('month', attendance_date);
```

### 5.3 `v_prospect_search` — Full-Text Prospect Search

Optimized view for the History Prospek search bar (search by store name, address, or PIC).

```sql
CREATE VIEW v_prospect_search AS
SELECT
    p.*,
    to_tsvector('indonesian',
        coalesce(p.store_name, '') || ' ' ||
        coalesce(p.store_address, '') || ' ' ||
        coalesce(p.pic_name, '')
    ) AS search_vector
FROM prospects p;
```

---

## 6. Relationships Summary

| Parent Entity | Child Entity | FK Column | Cardinality | ON DELETE |
|---|---|---|---|---|
| `branches` | `users` | `users.branch_id` | 1 : N | SET NULL |
| `users` | `user_shift_assignments` | `user_shift_assignments.user_id` | 1 : N | CASCADE |
| `shifts` | `user_shift_assignments` | `user_shift_assignments.shift_id` | 1 : N | RESTRICT |
| `users` | `attendances` | `attendances.user_id` | 1 : N | CASCADE |
| `shifts` | `attendances` | `attendances.shift_id` | 1 : N | RESTRICT |
| `users` | `prospects` | `prospects.user_id` | 1 : N | CASCADE |
| `users` | `prospects` | `prospects.verified_by` | 1 : N | SET NULL |
| `prospects` | `prospect_status_history` | `prospect_status_history.prospect_id` | 1 : N | CASCADE |
| `users` | `prospect_status_history` | `prospect_status_history.changed_by` | 1 : N | RESTRICT |
| `users` | `notifications` | `notifications.user_id` | 1 : N | CASCADE |
| `users` | `login_sessions` | `login_sessions.user_id` | 1 : N | CASCADE |

---

## 7. Complete Index Summary

| # | Table | Index Name | Columns | Type | Purpose |
|---|---|---|---|---|---|
| 1 | branches | `branches_pkey` | `id` | UNIQUE (btree) | PK |
| 2 | branches | `uk_branches_code` | `code` | UNIQUE (btree) | Unique branch code |
| 3 | branches | `ix_branches_active` | `is_active` | btree | Active filter |
| 4 | users | `users_pkey` | `id` | UNIQUE (btree) | PK |
| 5 | users | `uk_users_username` | `username` | UNIQUE (btree) | Login lookup |
| 6 | users | `uk_users_email` | `email` | UNIQUE (btree) | Email uniqueness |
| 7 | users | `uk_users_referral_code` | `referral_code` | UNIQUE (btree) | Referral lookup |
| 8 | users | `ix_users_branch_id` | `branch_id` | btree | Branch filter |
| 9 | users | `ix_users_status` | `status` | btree | Status filter |
| 10 | shifts | `shifts_pkey` | `id` | UNIQUE (btree) | PK |
| 11 | shifts | `ix_shifts_active` | `is_active` | btree | Active filter |
| 12 | user_shift_assignments | `user_shift_assignments_pkey` | `id` | UNIQUE (btree) | PK |
| 13 | user_shift_assignments | `ix_usa_user_id` | `user_id` | btree | FK lookup |
| 14 | user_shift_assignments | `ix_usa_user_effective` | `user_id, effective_from DESC` | btree | Current shift lookup |
| 15 | user_shift_assignments | `uk_usa_active_per_user` | `user_id` WHERE `effective_to IS NULL` | UNIQUE (partial) | One active shift per user |
| 16 | attendances | `attendances_pkey` | `id` | UNIQUE (btree) | PK |
| 17 | attendances | `uk_attendances_user_date` | `user_id, attendance_date` | UNIQUE (btree) | One record per day |
| 18 | attendances | `ix_attendances_user_date_desc` | `user_id, attendance_date DESC` | btree | History query |
| 19 | attendances | `ix_attendances_date` | `attendance_date` | btree | Date range filter |
| 20 | attendances | `ix_attendances_status` | `status` | btree | Status filter |
| 21 | prospects | `prospects_pkey` | `id` | UNIQUE (btree) | PK |
| 22 | prospects | `ix_prospects_user_id` | `user_id` | btree | FK lookup |
| 23 | prospects | `ix_prospects_visit_date` | `visit_date` | btree | Date filter |
| 24 | prospects | `ix_prospects_user_date_desc` | `user_id, visit_date DESC` | btree | Agent history list |
| 25 | prospects | `ix_prospects_status` | `status` | btree | Status filter |
| 26 | prospects | `ix_prospects_store_name` | `store_name` | btree | Name search |
| 27 | prospects | `ix_prospects_pic_name` | `pic_name` | btree | PIC search |
| 28 | prospects | `ix_prospects_pic_phone` | `pic_phone` | btree | Phone search |
| 29 | prospects | `ix_prospects_store_address_gin` | `to_tsvector('indonesian', store_address)` | GIN | Full-text address search |
| 30 | prospects | `ix_prospects_verified_by` | `verified_by` | btree | Verifier lookup |
| 31 | prospect_status_history | `prospect_status_history_pkey` | `id` | UNIQUE (btree) | PK |
| 32 | prospect_status_history | `ix_psh_prospect_id` | `prospect_id` | btree | FK lookup |
| 33 | prospect_status_history | `ix_psh_changed_by` | `changed_by` | btree | FK lookup |
| 34 | prospect_status_history | `ix_psh_changed_at` | `changed_at DESC` | btree | Chronological sort |
| 35 | notifications | `notifications_pkey` | `id` | UNIQUE (btree) | PK |
| 36 | notifications | `ix_notif_user_unread` | `user_id, is_read, created_at DESC` | btree | Unread badge + list |
| 37 | notifications | `ix_notif_user_created` | `user_id, created_at DESC` | btree | Full list pagination |
| 38 | notifications | `ix_notif_created_at` | `created_at DESC` | btree | Cleanup job |
| 39 | login_sessions | `login_sessions_pkey` | `id` | UNIQUE (btree) | PK |
| 40 | login_sessions | `uk_sessions_token` | `token_hash` | UNIQUE (btree) | Auth token lookup |
| 41 | login_sessions | `ix_sessions_user_id` | `user_id` | btree | List user sessions |
| 42 | login_sessions | `ix_sessions_expires_at` | `expires_at` WHERE `revoked_at IS NULL` | btree (partial) | Expiry cleanup |
| 43 | app_config | `app_config_pkey` | `id` | UNIQUE (btree) | PK |
| 44 | app_config | `uk_app_config_key` | `key` | UNIQUE (btree) | Key lookup |

---

## 8. Constraints Summary

### 8.1 Primary Keys

All tables use `id BIGSERIAL PRIMARY KEY`.

### 8.2 Unique Constraints (excluding PKs)

| Table | Constraint | Columns |
|---|---|---|
| branches | `uk_branches_code` | `code` |
| users | `uk_users_username` | `username` |
| users | `uk_users_email` | `email` |
| users | `uk_users_referral_code` | `referral_code` |
| user_shift_assignments | `uk_usa_active_per_user` | `user_id` WHERE `effective_to IS NULL` |
| attendances | `uk_attendances_user_date` | `user_id, attendance_date` |
| login_sessions | `uk_sessions_token` | `token_hash` |
| app_config | `uk_app_config_key` | `key` |

### 8.3 Check Constraints

| Table | Column(s) | Rule |
|---|---|---|
| branches | `latitude` | BETWEEN -90 AND 90 |
| branches | `longitude` | BETWEEN -180 AND 180 |
| users | `role` | IN ('agent','supervisor','admin') |
| users | `status` | IN ('active','inactive','suspended') |
| user_shift_assignments | `effective_to, effective_from` | `effective_to IS NULL OR effective_to >= effective_from` |
| attendances | `clock_in_latitude` | BETWEEN -90 AND 90 |
| attendances | `clock_in_longitude` | BETWEEN -180 AND 180 |
| attendances | `clock_out_latitude` | BETWEEN -90 AND 90 |
| attendances | `clock_out_longitude` | BETWEEN -180 AND 180 |
| attendances | `clock_in_accuracy_m` | >= 0 |
| attendances | `clock_out_accuracy_m` | >= 0 |
| attendances | `duration_minutes` | >= 0 |
| attendances | `clock_in_at, clock_out_at` | `clock_out_at IS NULL OR clock_out_at > clock_in_at` |
| attendances | `duration_minutes, clock_in_at, clock_out_at` | `duration_minutes IS NULL OR (clock_in_at IS NOT NULL AND clock_out_at IS NOT NULL)` |
| attendances | `status` | IN ('tepat_waktu','terlambat','izin','sakit','alpha') |
| attendances | `method` | IN ('gps_selfie','manual') |
| prospects | `latitude` | BETWEEN -90 AND 90 |
| prospects | `longitude` | BETWEEN -180 AND 180 |
| prospects | `location_accuracy_m` | >= 0 |
| prospects | `visit_number` | >= 1 |
| prospects | `pic_phone` | `LENGTH(REGEXP_REPLACE(pic_phone, '\D', '', 'g')) >= 9` |
| prospects | `status` | IN ('menunggu_verifikasi','terverifikasi','ditolak') |
| prospects | `status, rejection_reason` | `status != 'ditolak' OR rejection_reason IS NOT NULL