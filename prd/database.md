# Product Requirements Document (PRD) — Database Design

**Project:** Sales Point — Field Agent Acquisition App (VisioNet Mini ATM)
**Document Scope:** Database layer only (entities, tables, columns, data types, relationships, indexes, constraints)
**Source of Truth:** UI Mockup (HTML)

---

## 1. Overview

The database supports a mobile field-agent application used by "Sales Point Agents" to:
- Authenticate and manage sessions
- Clock in/out with GPS location verification
- Acquire new merchant/store prospects (toko) with photo documentation
- Track visit history and verification status
- Receive notifications
- View personal performance analytics (monthly prospect count)
- Manage referral codes

**Target DBMS:** PostgreSQL 15+ (or equivalent relational DBMS)
**Charset:** UTF-8
**Timezone:** All timestamps stored in UTC; application converts to Asia/Jakarta (WIB) for display.

---

## 2. Entity Relationship Summary

| # | Entity | Description | Primary Relationships |
|---|--------|-------------|----------------------|
| 1 | `users` | Sales agent accounts | Has many attendances, prospects, notifications; belongs to one branch |
| 2 | `branches` | Office/branch locations (Kantor Cabang) | Has many users, attendances |
| 3 | `shifts` | Work shift definitions | Has many attendances |
| 4 | `attendances` | Daily clock in/out records | Belongs to user, branch, shift |
| 5 | `prospects` | Store/merchant acquisition records | Belongs to user; has many photos |
| 6 | `prospect_photos` | Photo documentation for prospects | Belongs to prospect |
| 7 | `prospect_statuses` | Lookup: verification status of prospects | Referenced by prospects |
| 8 | `notifications` | In-app notification messages | Belongs to user |
| 9 | `monthly_prospect_stats` | Materialized/aggregated monthly counts | Belongs to user |
| 10 | `app_sessions` | Active session/token tracking | Belongs to user |

### ERD (Textual)

```
branches 1───∞ users 1───∞ attendances ∞───1 shifts
                     │              ∞───1 branches
                     │
                     ├───∞ prospects 1───∞ prospect_photos
                     │              ∞───1 prospect_statuses
                     │
                     ├───∞ notifications
                     └───∞ monthly_prospect_stats
                     └───∞ app_sessions
```

---

## 3. Table Definitions

### 3.1 `branches`
Stores office/branch locations used for attendance geofencing.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `code` | `VARCHAR(20)` | NO | — | Unique branch code (e.g. `JKT-SEL`) |
| `name` | `VARCHAR(150)` | NO | — | Branch display name (e.g. "Kantor Cabang Jakarta Selatan") |
| `address` | `TEXT` | NO | — | Full street address |
| `latitude` | `DECIMAL(10,7)` | NO | — | Branch geocoordinate (lat) |
| `longitude` | `DECIMAL(10,7)` | NO | — | Branch geocoordinate (lng) |
| `geofence_radius_m` | `INTEGER` | NO | `100` | Allowed attendance radius in meters |
| `is_active` | `BOOLEAN` | NO | `TRUE` | Soft-delete flag |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK branches (id)`
- `UNIQUE branches (code)`

**Indexes:**
- `idx_branches_code` ON `(code)` — login lookup
- `idx_branches_active` ON `(is_active)`

---

### 3.2 `users`
Sales Point agent accounts.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `username` | `VARCHAR(50)` | NO | — | Unique login username (e.g. `rizky.pratama`) |
| `password_hash` | `VARCHAR(255)` | NO | — | Bcrypt/argon2 hash |
| `full_name` | `VARCHAR(150)` | NO | — | Full legal name |
| `email` | `VARCHAR(255)` | NO | — | Corporate email |
| `referral_code` | `VARCHAR(20)` | NO | — | Unique agent referral code (e.g. `SP-RZK2041`) |
| `avatar_initials` | `VARCHAR(4)` | NO | — | Derived initials (e.g. `RP`) |
| `role` | `VARCHAR(20)` | NO | `'agent'` | `agent`, `supervisor`, `admin` |
| `status` | `VARCHAR(20)` | NO | `'active'` | `active`, `suspended`, `inactive` |
| `branch_id` | `BIGINT` | YES | `NULL` | FK → branches(id) |
| `remember_token` | `VARCHAR(255)` | YES | `NULL` | "Ingat saya" persistent token |
| `last_login_at` | `TIMESTAMPTZ` | YES | `NULL` | |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK users (id)`
- `UNIQUE users (username)`
- `UNIQUE users (email)`
- `UNIQUE users (referral_code)`
- `FK users_branch_id FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE SET NULL`
- `CHECK users.role IN ('agent','supervisor','admin')`
- `CHECK users.status IN ('active','suspended','inactive')`
- `CHECK length(username) >= 3`
- `CHECK length(referral_code) BETWEEN 5 AND 20`

**Indexes:**
- `idx_users_username` ON `(username)` — login
- `idx_users_referral` ON `(referral_code)`
- `idx_users_branch` ON `(branch_id)`

---

### 3.3 `shifts`
Work shift definitions (e.g. Reguler 08:00–17:00).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `name` | `VARCHAR(50)` | NO | — | e.g. "Reguler" |
| `start_time` | `TIME` | NO | — | Shift start (e.g. `08:00`) |
| `end_time` | `TIME` | NO | — | Shift end (e.g. `17:00`) |
| `grace_minutes` | `INTEGER` | NO | `15` | Late tolerance in minutes |
| `is_active` | `BOOLEAN` | NO | `TRUE` | |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK shifts (id)`
- `UNIQUE shifts (name, start_time, end_time)`
- `CHECK end_time > start_time`

**Indexes:**
- `idx_shifts_active` ON `(is_active)`

---

### 3.4 `attendances`
Daily clock in/out records with GPS.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `user_id` | `BIGINT` | NO | — | FK → users(id) |
| `branch_id` | `BIGINT` | NO | — | FK → branches(id) — where attendance recorded |
| `shift_id` | `BIGINT` | YES | `NULL` | FK → shifts(id) |
| `attendance_date` | `DATE` | NO | — | The working date (WIB) |
| `clock_in_at` | `TIMESTAMPTZ` | YES | `NULL` | NULL = not yet clocked in |
| `clock_out_at` | `TIMESTAMPTZ` | YES | `NULL` | NULL = not yet clocked out |
| `clock_in_lat` | `DECIMAL(10,7)` | YES | `NULL` | GPS lat at clock in |
| `clock_in_lng` | `DECIMAL(10,7)` | YES | `NULL` | GPS lng at clock in |
| `clock_in_accuracy_m` | `INTEGER` | YES | `NULL` | GPS accuracy meters (e.g. ±8 m) |
| `clock_out_lat` | `DECIMAL(10,7)` | YES | `NULL` | |
| `clock_out_lng` | `DECIMAL(10,7)` | YES | `NULL` | |
| `clock_out_accuracy_m` | `INTEGER` | YES | `NULL` | |
| `method` | `VARCHAR(20)` | NO | `'gps_selfie'` | `gps_selfie`, `gps_only`, `manual` |
| `duration_minutes` | `INTEGER` | YES | `NULL` | Computed: clock_out - clock_in |
| `status` | `VARCHAR(20)` | NO | `'pending'` | `pending`, `on_time`, `late`, `early_leave`, `complete`, `absent`, `leave` |
| `status_label` | `VARCHAR(50)` | YES | `NULL` | Display label (e.g. "Terlambat 12 m", "Izin sakit") |
| `status_class` | `VARCHAR(10)` | YES | `NULL` | UI class: `good`, `warn`, `crit`, `info` |
| `note` | `TEXT` | YES | `NULL` | e.g. "Izin sakit" |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK attendances (id)`
- `FK attend_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `FK attend_branch FOREIGN KEY (branch_id) REFERENCES branches(id)`
- `FK attend_shift FOREIGN KEY (shift_id) REFERENCES shifts(id) ON DELETE SET NULL`
- `UNIQUE attendances (user_id, attendance_date)` — one record per user per day
- `CHECK method IN ('gps_selfie','gps_only','manual')`
- `CHECK status IN ('pending','on_time','late','early_leave','complete','absent','leave')`
- `CHECK status_class IS NULL OR status_class IN ('good','warn','crit','info')`
- `CHECK (clock_in_at IS NULL) OR (clock_in_lat IS NOT NULL AND clock_in_lng IS NOT NULL)`
- `CHECK (clock_out_at IS NULL) OR (clock_out_at > clock_in_at)`

**Indexes:**
- `idx_attend_user_date` ON `(user_id, attendance_date DESC)` — primary query pattern
- `idx_attend_branch_date` ON `(branch_id, attendance_date)`
- `idx_attend_status` ON `(status)`
- `idx_attend_date` ON `(attendance_date)`

---

### 3.5 `prospect_statuses`
Lookup table for prospect verification status.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `code` | `VARCHAR(30)` | NO | — | e.g. `verified`, `pending_verification`, `rejected`, `visit_scheduled` |
| `label` | `VARCHAR(100)` | NO | — | Display label (e.g. "Terverifikasi") |
| `ui_class` | `VARCHAR(10)` | NO | `'info'` | `good`, `warn`, `crit`, `info` |
| `sort_order` | `INTEGER` | NO | `0` | Display ordering |
| `is_active` | `BOOLEAN` | NO | `TRUE` | |

**Seed Data:**

| code | label | ui_class |
|------|-------|----------|
| `verified` | Terverifikasi | good |
| `pending_verification` | Menunggu verifikasi | warn |
| `rejected` | Ditolak | crit |
| `visit_scheduled` | Kunjungan ulang dijadwalkan | warn |
| `installed` | Sudah setuju pasang | info |

**Constraints:**
- `PK prospect_statuses (id)`
- `UNIQUE prospect_statuses (code)`

**Indexes:**
- `idx_pstatus_code` ON `(code)`

---

### 3.6 `prospects`
Store/merchant acquisition records created by agents during field visits.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `user_id` | `BIGINT` | NO | — | FK → users(id) — agent who created |
| `store_name` | `VARCHAR(200)` | NO | — | Nama Toko (e.g. "Toko Berkah Jaya") |
| `store_address` | `TEXT` | NO | — | Full address |
| `pic_name` | `VARCHAR(150)` | NO | — | Person in charge name |
| `pic_phone` | `VARCHAR(20)` | NO | — | E.164 or local format (digits only ≥9) |
| `visit_time` | `TIMESTAMPTZ` | NO | `NOW()` | When the visit occurred |
| `visit_date` | `DATE` | NO | — | Derived from visit_time (WIB date) for grouping |
| `latitude` | `DECIMAL(10,7)` | NO | — | GPS lat at prospect location |
| `longitude` | `DECIMAL(10,7)` | NO | — | GPS lng at prospect location |
| `location_accuracy_m` | `INTEGER` | YES | `NULL` | GPS accuracy |
| `location_label` | `VARCHAR(255)` | YES | `NULL` | Reverse-geocoded label (e.g. "Jl. Bangka Raya, Mampang Prapatan") |
| `note` | `TEXT` | YES | `NULL` | Visit notes (optional) |
| `status_id` | `BIGINT` | YES | `NULL` | FK → prospect_statuses(id); NULL for today's new |
| `status_label` | `VARCHAR(100)` | YES | `NULL` | Denormalized for history list |
| `status_class` | `VARCHAR(10)` | YES | `NULL` | Denormalized UI class |
| `visit_sequence` | `INTEGER` | NO | `1` | Visit number for the day (e.g. 5th visit today) |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK prospects (id)`
- `FK prospect_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `FK prospect_status FOREIGN KEY (status_id) REFERENCES prospect_statuses(id) ON DELETE SET NULL`
- `CHECK length(store_name) >= 2`
- `CHECK length(pic_name) >= 2`
- `CHECK length(regexp_replace(pic_phone, '[^0-9]', '', 'g')) >= 9`
- `CHECK latitude BETWEEN -90 AND 90`
- `CHECK longitude BETWEEN -180 AND 180`

**Indexes:**
- `idx_prospect_user_date` ON `(user_id, visit_date DESC)` — agent's prospect list
- `idx_prospect_date` ON `(visit_date DESC)` — today's count / dashboard
- `idx_prospect_status` ON `(status_id)`
- `idx_prospect_store_name` ON `(store_name)` — search by store name
- `idx_prospect_search` ON `(store_name, store_address, pic_name)` — combined search (GIN trigram optional)
- `idx_prospect_phone` ON `(pic_phone)`

---

### 3.7 `prospect_photos`
Photo documentation for each prospect (plang + selfie).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `prospect_id` | `BIGINT` | NO | — | FK → prospects(id) |
| `photo_type` | `VARCHAR(20)` | NO | — | `plang` (signboard) or `selfie` (with PIC) |
| `file_path` | `VARCHAR(500)` | NO | — | Object storage path / data URL reference |
| `file_size_bytes` | `BIGINT` | YES | `NULL` | |
| `mime_type` | `VARCHAR(50)` | NO | `'image/jpeg'` | |
| `width_px` | `INTEGER` | YES | `NULL` | |
| `height_px` | `INTEGER` | YES | `NULL` | |
| `captured_lat` | `DECIMAL(10,7)` | YES | `NULL` | GPS stamped in photo |
| `captured_lng` | `DECIMAL(10,7)` | YES | `NULL` | |
| `captured_at` | `TIMESTAMPTZ` | NO | `NOW()` | When photo was taken |
| `location_label` | `VARCHAR(255)` | YES | `NULL` | Geocoded label stamped in photo |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK prospect_photos (id)`
- `FK pp_prospect FOREIGN KEY (prospect_id) REFERENCES prospects(id) ON DELETE CASCADE`
- `CHECK photo_type IN ('plang','selfie')`
- `UNIQUE prospect_photos (prospect_id, photo_type)` — exactly one plang + one selfie per prospect

**Indexes:**
- `idx_pp_prospect` ON `(prospect_id, photo_type)` — fetch photos for detail view

---

### 3.8 `notifications`
In-app notifications per user.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `user_id` | `BIGINT` | NO | — | FK → users(id) |
| `category` | `VARCHAR(20)` | NO | — | `green`, `orange`, `blue`, `red` (UI color) |
| `title` | `VARCHAR(200)` | NO | — | Notification title |
| `body` | `TEXT` | NO | — | Notification body text |
| `sent_at` | `TIMESTAMPTZ` | NO | `NOW()` | When notification was sent |
| `is_read` | `BOOLEAN` | NO | `FALSE` | Read status |
| `read_at` | `TIMESTAMPTZ` | YES | `NULL` | |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK notifications (id)`
- `FK notif_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `CHECK category IN ('green','orange','blue','red')`
- `CHECK (is_read = FALSE) OR (read_at IS NOT NULL)`

**Indexes:**
- `idx_notif_user_unread` ON `(user_id, is_read, sent_at DESC)` — badge count + list
- `idx_notif_sent` ON `(sent_at DESC)`

---

### 3.9 `monthly_prospect_stats`
Aggregated monthly prospect counts for profile chart.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `user_id` | `BIGINT` | NO | — | FK → users(id) |
| `year` | `INTEGER` | NO | — | e.g. 2026 |
| `month` | `INTEGER` | NO | — | 1–12 |
| `month_label` | `VARCHAR(5)` | NO | — | Short label (Jan, Feb, ..., Sep, Ags) |
| `prospect_count` | `INTEGER` | NO | `0` | Total prospects that month |
| `is_current` | `BOOLEAN` | NO | `FALSE` | Whether this is the ongoing month |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `NOW()` | |

**Constraints:**
- `PK monthly_prospect_stats (id)`
- `FK mps_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE monthly_prospect_stats (user_id, year, month)`
- `CHECK month BETWEEN 1 AND 12`
- `CHECK prospect_count >= 0`

**Indexes:**
- `idx_mps_user_year` ON `(user_id, year, month)` — chart data

---

### 3.10 `app_sessions`
Active session tracking for authenticated users.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGSERIAL` | NO | — | PK |
| `user_id` | `BIGINT` | NO | — | FK → users(id) |
| `session_token` | `VARCHAR(255)` | NO | — | Opaque token |
| `device_info` | `VARCHAR(255)` | YES | `NULL` | User-agent / device string |
| `ip_address` | `INET` | YES | `NULL` | |
| `expires_at` | `TIMESTAMPTZ` | NO | — | Token expiry |
| `is_active` | `BOOLEAN` | NO | `TRUE` | |
| `created_at` | `TIMESTAMPTZ` | NO | `NOW()` | |
| `revoked_at` | `TIMESTAMPTZ` | YES | `NULL` | |

**Constraints:**
- `PK app_sessions (id)`
- `FK sess_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE app_sessions (session_token)`

**Indexes:**
- `idx_sess_token` ON `(session_token)` — auth middleware lookup
- `idx_sess_user_active` ON `(user_id, is_active)`

---

## 4. Key Relationships (Summary)

| Parent | Child | Cardinality | FK Column | ON DELETE |
|--------|-------|-------------|-----------|----------|
| `branches` | `users` | 1 : N | `users.branch_id` | SET NULL |
| `branches` | `attendances` | 1 : N | `attendances.branch_id` | RESTRICT |
| `shifts` | `attendances` | 1 : N | `attendances.shift_id` | SET NULL |
| `users` | `attendances` | 1 : N | `attendances.user_id` | CASCADE |
| `users` | `prospects` | 1 : N | `prospects.user_id` | CASCADE |
| `prospect_statuses` | `prospects` | 1 : N | `prospects.status_id` | SET NULL |
| `prospects` | `prospect_photos` | 1 : N | `prospect_photos.prospect_id` | CASCADE |
| `users` | `notifications` | 1 : N | `notifications.user_id` | CASCADE |
| `users` | `monthly_prospect_stats` | 1 : N | `monthly_prospect_stats.user_id` | CASCADE |
| `users` | `app_sessions` | 1 : N | `app_sessions.user_id` | CASCADE |

---

## 5. Index Strategy Summary

### 5.1 Primary Lookup Indexes (high-frequency queries)

| Index | Table | Columns | Use Case |
|-------|-------|---------|----------|
| `idx_users_username` | users | `(username)` | Login authentication |
| `idx_attend_user_date` | attendances | `(user_id, attendance_date DESC)` | "Absen hari ini", attendance history |
| `idx_prospect_user_date` | prospects | `(user_id, visit_date DESC)` | Agent's prospect list, today count |
| `idx_notif_user_unread` | notifications | `(user_id, is_read, sent_at DESC)` | Unread badge, notification list |
| `idx_mps_user_year` | monthly_prospect_stats | `(user_id, year, month)` | Profile chart |
| `idx_pp_prospect` | prospect_photos | `(prospect_id, photo_type)` | Detail view photo fetch |
| `idx_sess_token` | app_sessions | `(session_token)` | Auth middleware |

### 5.2 Search Indexes

| Index | Table | Columns | Use Case |
|-------|-------|---------|----------|
| `idx_prospect_store_name` | prospects | `(store_name)` | History search |
| `idx_prospect_search` | prospects | `(store_name, store_address, pic_name)` | Combined search (mockup search bar) |
| `idx_prospect_phone` | prospects | `(pic_phone)` | Call PIC feature, dedup |

### 5.3 Optional Full-Text / Trigram (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_prospect_search_trgm
  ON prospects USING GIN (store_name gin_trgm_ops, pic_name gin_trgm_ops);
```
Supports the mockup's search placeholder: *"Cari nama toko, alamat, atau PIC"*.

---

## 6. Constraints Summary

### 6.1 Not-Null Constraints
All `id`, foreign-key columns, and business-critical fields (store_name, address, pic_name, pic_phone, visit_time, coordinates, timestamps) are `NOT NULL`.

### 6.2 Unique Constraints
- `users.username` — login identity
- `users.email`
- `users.referral_code` — agent referral identity
- `branches.code`
- `attendances (user_id, attendance_date)` — one attendance record per agent per day
- `prospect_photos (prospect_id, photo_type)` — exactly one plang + one selfie per prospect
- `prospect_statuses.code`
- `monthly_prospect_stats (user_id, year, month)`
- `app_sessions.session_token`

### 6.3 Check Constraints
- **Users:** `role ∈ {agent, supervisor, admin}`, `status ∈ {active, suspended, inactive}`, username ≥3 chars, referral_code 5–20 chars
- **Attendances:** method ∈ valid set; status ∈ valid set; status_class ∈ valid set or NULL; clock_out > clock_in; GPS required when clock_in present
- **Shifts:** end_time > start_time
- **Prospects:** store_name ≥2 chars; pic_name ≥2 chars; pic_phone digits ≥9; lat/lng range validation
- **Prospect_photos:** photo_type ∈ {plang, selfie}
- **Notifications:** category ∈ {green, orange, blue, red}; read_at must be set if is_read=TRUE
- **Monthly stats:** month ∈ 1–12; prospect_count ≥ 0

### 6.4 Foreign Key Constraints
See Section 4. All FKs use explicit `ON DELETE` policies: cascade for child-owned data (prospects, photos, notifications, sessions), SET NULL for optional references (branch, shift, status).

---

## 7. Data Mapping: Mockup → Database

| Mockup Element | Table | Column(s) |
|----------------|-------|-----------|
| Login username/password | `users` | `username`, `password_hash` |
| "Ingat saya" checkbox | `users` | `remember_token` |
| Clock In time (07:48) | `attendances` | `clock_in_at` |
| Clock Out time | `attendances` | `clock_out_at` |
| GPS accuracy ±8 m | `attendances` | `clock_in_accuracy_m` |
| Branch name (Jakarta Selatan) | `branches` | `name`, `address` |
| Shift (Reguler 08:00–17:00) | `shifts` | `name`, `start_time`, `end_time` |
| Referral code (SP-RZK2041) | `users` | `referral_code` |
| Prospek hari ini count (4) | `prospects` | `WHERE user_id=? AND visit_date=CURRENT_DATE` |
| Store name, address, PIC, phone | `prospects` | `store_name`, `store_address`, `pic_name`, `pic_phone` |
| Visit time (08:52 WIB) | `prospects` | `visit_time` |
| Foto Plang / Selfie | `prospect_photos` | `photo_type`, `file_path`, `captured_at`, `captured_lat/lng` |
| Photo timestamp stamp | `prospect_photos` | `captured_at`, `location_label` |
| Status badge (Terverifikasi/Ditolak) | `prospect_statuses` | `label`, `ui_class` |
| Notifications (3 unread) | `notifications` | `is_read = FALSE` count |
| Monthly chart (Jan–Sep 2026) | `monthly_prospect_stats` | rows for `year=2026` |
| Attendance history (Kamis, Rabu...) | `attendances` | `attendance_date`, `clock_in_at`, `clock_out_at`, `duration_minutes`, `status_label` |
| Visit sequence (ke-5 hari ini) | `prospects` | `visit_sequence` |
| Note/Catatan kunjungan | `prospects` | `note` |
| Agent full name, email | `users` | `full_name`, `email` |

---

## 8. Naming Conventions

- **Tables:** snake_case, plural (`users`, `prospects`, `attendances`)
- **Primary keys:** `id` (BIGSERIAL) on all tables
- **Foreign keys:** `{singular_table}_id` (e.g. `user_id`, `branch_id`, `prospect_id`)
- **Timestamps:** `*_at` suffix, `TIMESTAMPTZ`, default `NOW()`
- **Boolean flags:** `is_*` prefix (e.g. `is_active`, `is_read`, `is_current`)
- **Status enums:** `VARCHAR(20)` with CHECK constraint
- **Indexes:** `idx_{table}_{purpose}`

---

## 9. Migration / Seed Order

1. `branches` (no FK dependencies)
2. `shifts` (no FK dependencies)
3. `prospect_statuses` (no FK dependencies)
4. `users` (depends on `branches`)
5. `attendances` (depends on `users`, `branches`, `shifts`)
6. `prospects` (depends on `users`, `prospect_statuses`)
7. `prospect_photos` (depends on `prospects`)
8. `notifications` (depends on `users`)
9. `monthly_prospect_stats` (depends on `users`)
10. `app_sessions` (depends on `users`)

Seed data required for `prospect_statuses` (5 rows) and at least one `branch` and `shift` record before creating test `users`.

---

*End of Database PRD.*