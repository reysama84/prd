# Product Requirements Document (PRD) — Database Schema

**Project:** Sales Point — Field Sales Tracking Mobile Application  
**Scope:** Database design only (entities, tables, columns, data types, relationships, indexes, constraints)  
**Target Database:** PostgreSQL 14+ (syntax-compatible; types map to MySQL 8 / MariaDB with minor adjustments)  
**Document Version:** 1.0

---

## 1. Overview

The application is a field-sales tracking system. Core functional areas visible in the mockup:

| Area | Key Capabilities |
|---|---|
| **Authentication** | Login with credentials, session management, remember-me |
| **Attendance / Clock** | Daily clock-in/out with GPS coordinates, location label, two verification photos |
| **Prospects** | CRUD for sales prospects; store location, status pipeline, notes, two attached photos |
| **Visits** | Logged visits to prospects with outcome and location |
| **Notifications** | In-app notifications with typed severity (info, success, warning, critical) |
| **Referrals** | Unique referral code per user; track referrals |
| **Targets / Stats** | Daily/monthly targets powering the dashboard stat cards and progress meter |

---

## 2. Entity-Relationship Summary

```
users 1───∞ user_sessions
users 1───∞ attendances ───∞ attendance_photos
users 1───∞ prospects ───∞ prospect_photos
users 1───∞ visits ───1 prospects
users 1───∞ notifications
users 1───1 referral_codes  (unique per user)
users 1───∞ sales_targets
users 1───∞ referrals  (referrer → referred user)
```

---

## 3. Tables — Detailed Definitions

### 3.1 `users`

Sales agents, supervisors, and administrators.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `name` | `VARCHAR(120)` | NO | — | Full display name |
| `email` | `VARCHAR(255)` | NO | — | Unique; login identifier |
| `phone` | `VARCHAR(32)` | NO | — | E.164 or local format |
| `password_hash` | `VARCHAR(255)` | NO | — | bcrypt/argon2 hash |
| `avatar_url` | `VARCHAR(512)` | YES | NULL | Profile image URL |
| `role` | `VARCHAR(20)` | NO | `'sales_agent'` | `sales_agent`, `supervisor`, `admin` |
| `status` | `VARCHAR(20)` | NO | `'active'` | `active`, `inactive`, `suspended` |
| `remember_token` | `VARCHAR(255)` | YES | NULL | For "remember me" feature |
| `last_login_at` | `TIMESTAMPTZ` | YES | NULL | |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK users (id)`
- `UNIQUE users (email)`
- `UNIQUE users (phone)`
- `INDEX idx_users_role (role)`
- `INDEX idx_users_status (status)`

---

### 3.2 `user_sessions`

Active session tokens for authentication.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` |
| `token_hash` | `VARCHAR(255)` | NO | — | Hashed session token |
| `device_info` | `VARCHAR(500)` | YES | NULL | User-agent / device model |
| `ip_address` | `VARCHAR(45)` | YES | NULL | IPv4 or IPv6 |
| `expires_at` | `TIMESTAMPTZ` | NO | — | Token expiry |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `revoked_at` | `TIMESTAMPTZ` | YES | NULL | Set when session is logged out |

**Constraints & Indexes:**
- `PK user_sessions (id)`
- `FK fk_sessions_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE user_sessions (token_hash)`
- `INDEX idx_sessions_user (user_id)`
- `INDEX idx_sessions_expires (expires_at)`

---

### 3.3 `attendances`

Daily clock-in / clock-out records.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` |
| `attendance_date` | `DATE` | NO | — | The working date |
| `check_in_at` | `TIMESTAMPTZ` | YES | NULL | Actual check-in timestamp |
| `check_out_at` | `TIMESTAMPTZ` | YES | NULL | Actual check-out timestamp |
| `check_in_lat` | `NUMERIC(10,7)` | YES | NULL | GPS latitude at check-in |
| `check_in_lng` | `NUMERIC(10,7)` | YES | NULL | GPS longitude at check-in |
| `check_out_lat` | `NUMERIC(10,7)` | YES | NULL | GPS latitude at check-out |
| `check_out_lng` | `NUMERIC(10,7)` | YES | NULL | GPS longitude at check-out |
| `check_in_location` | `VARCHAR(255)` | YES | NULL | Reverse-geocoded label |
| `check_out_location` | `VARCHAR(255)` | YES | NULL | Reverse-geocoded label |
| `status` | `VARCHAR(20)` | NO | `'pending'` | `pending`, `clocked_in`, `completed`, `absent`, `late` |
| `duration_minutes` | `INTEGER` | YES | NULL | Computed on checkout |
| `notes` | `TEXT` | YES | NULL | Optional agent notes |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK attendances (id)`
- `FK fk_att_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE uq_att_user_date (user_id, attendance_date)` — one record per user per day
- `INDEX idx_att_user_date (user_id, attendance_date DESC)`
- `INDEX idx_att_status (status)`
- `INDEX idx_att_date (attendance_date DESC)`
- `CHECK (check_out_at IS NULL OR check_out_at > check_in_at)`
- `CHECK (status IN ('pending','clocked_in','completed','absent','late'))`

---

### 3.4 `attendance_photos`

Verification photographs attached to attendance records (selfie + location/store-front).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `attendance_id` | `BIGINT` | NO | — | FK → `attendances(id)` |
| `photo_type` | `VARCHAR(30)` | NO | — | `selfie`, `location`, `store_front` |
| `photo_url` | `VARCHAR(512)` | NO | — | Stored/CDN URL |
| `thumbnail_url` | `VARCHAR(512)` | YES | NULL | Optional smaller variant |
| `metadata` | `JSONB` | YES | NULL | EXIF GPS, file size, mime type |
| `taken_at` | `TIMESTAMPTZ` | YES | NULL | When the photo was captured |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK attendance_photos (id)`
- `FK fk_attphoto_att FOREIGN KEY (attendance_id) REFERENCES attendances(id) ON DELETE CASCADE`
- `INDEX idx_attphoto_att (attendance_id)`
- `INDEX idx_attphoto_type (photo_type)`
- `CHECK (photo_type IN ('selfie','location','store_front'))`

---

### 3.5 `prospects`

Sales prospect / lead records assigned to an agent.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` owning agent |
| `name` | `VARCHAR(200)` | NO | — | Prospect / store name |
| `phone` | `VARCHAR(32)` | YES | NULL | Contact phone |
| `email` | `VARCHAR(255)` | YES | NULL | Contact email |
| `address` | `TEXT` | YES | NULL | Street address |
| `lat` | `NUMERIC(10,7)` | YES | NULL | GPS latitude |
| `lng` | `NUMERIC(10,7)` | YES | NULL | GPS longitude |
| `status` | `VARCHAR(20)` | NO | `'new'` | `new`, `contacted`, `visited`, `interested`, `won`, `lost` |
| `thumbnail_url` | `VARCHAR(512)` | YES | NULL | Primary thumbnail shown in list |
| `last_visit_at` | `TIMESTAMPTZ` | YES | NULL | Denormalized from latest visit |
| `notes` | `TEXT` | YES | NULL | Free-text notes |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK prospects (id)`
- `FK fk_prosp_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT`
- `INDEX idx_prosp_user (user_id)`
- `INDEX idx_prosp_status (status)`
- `INDEX idx_prosp_user_created (user_id, created_at DESC)`
- `INDEX idx_prosp_name_trgm ON prospects USING gin (name gin_trgm_ops)` — for search bar fuzzy matching
- `INDEX idx_prosp_latlng ON prospects USING gist (point(lng, lat))` — for proximity queries
- `CHECK (status IN ('new','contacted','visited','interested','won','lost'))`

---

### 3.6 `prospect_photos`

Photographs attached to a prospect (e.g., store front, product display).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `prospect_id` | `BIGINT` | NO | — | FK → `prospects(id)` |
| `photo_type` | `VARCHAR(30)` | NO | `'general'` | `store_front`, `product`, `general` |
| `photo_url` | `VARCHAR(512)` | NO | — | |
| `thumbnail_url` | `VARCHAR(512)` | YES | NULL | |
| `caption` | `VARCHAR(255)` | YES | NULL | Optional caption |
| `sort_order` | `SMALLINT` | NO | `0` | Display ordering |
| `taken_at` | `TIMESTAMPTZ` | YES | NULL | |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK prospect_photos (id)`
- `FK fk_prosp_photo FOREIGN KEY (prospect_id) REFERENCES prospects(id) ON DELETE CASCADE`
- `INDEX idx_prosp_photo_prosp (prospect_id, sort_order)`
- `CHECK (photo_type IN ('store_front','product','general'))`

---

### 3.7 `visits`

Each logged visit by an agent to a prospect.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `prospect_id` | `BIGINT` | NO | — | FK → `prospects(id)` |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` visiting agent |
| `visited_at` | `TIMESTAMPTZ` | NO | `now()` | When visit occurred |
| `lat` | `NUMERIC(10,7)` | YES | NULL | GPS at time of visit |
| `lng` | `NUMERIC(10,7)` | YES | NULL | GPS at time of visit |
| `location_label` | `VARCHAR(255)` | YES | NULL | Reverse-geocoded label |
| `outcome` | `VARCHAR(20)` | NO | `'pending'` | `pending`, `interested`, `not_interested`, `follow_up`, `closed_won`, `closed_lost` |
| `notes` | `TEXT` | YES | NULL | |
| `photo_url` | `VARCHAR(512)` | YES | NULL | Visit evidence photo |
| `duration_minutes` | `INTEGER` | YES | NULL | Visit duration |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK visits (id)`
- `FK fk_visit_prosp FOREIGN KEY (prospect_id) REFERENCES prospects(id) ON DELETE CASCADE`
- `FK fk_visit_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT`
- `INDEX idx_visit_user_date (user_id, visited_at DESC)` — powers "today's visits" stat
- `INDEX idx_visit_prosp (prospect_id)`
- `INDEX idx_visit_outcome (outcome)`
- `CHECK (outcome IN ('pending','interested','not_interested','follow_up','closed_won','closed_lost'))`

---

### 3.8 `notifications`

In-app notification messages per user.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` recipient |
| `type` | `VARCHAR(40)` | NO | — | `visit_reminder`, `prospect_update`, `attendance_alert`, `referral`, `system` |
| `severity` | `VARCHAR(10)` | NO | `'info'` | `info`, `success`, `warning`, `critical` — maps to mockup badge colors |
| `title` | `VARCHAR(200)` | NO | — | Notification title |
| `body` | `TEXT` | NO | — | Notification body text |
| `data` | `JSONB` | YES | NULL | Payload (prospect_id, visit_id, etc.) |
| `is_read` | `BOOLEAN` | NO | `false` | |
| `read_at` | `TIMESTAMPTZ` | YES | NULL | |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK notifications (id)`
- `FK fk_notif_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `INDEX idx_notif_user_unread (user_id, is_read, created_at DESC)` — primary query for notification list + unread badge count
- `INDEX idx_notif_created (created_at DESC)`
- `CHECK (severity IN ('info','success','warning','critical'))`

---

### 3.9 `referral_codes`

Unique referral code per user (one-to-one).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` |
| `code` | `VARCHAR(20)` | NO | — | Human-readable code (e.g., `AGN7X3K`) |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK referral_codes (id)`
- `FK fk_refcode_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE uq_refcode_code (code)` — case-insensitive uniqueness enforced via `UPPER(code)`
- `UNIQUE uq_refcode_user (user_id)` — one code per user
- `INDEX idx_refcode_code_upper ON referral_codes (UPPER(code))`

---

### 3.10 `referrals`

Tracks when a user's referral code is used by a new registrant.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `referrer_id` | `BIGINT` | NO | — | FK → `users(id)` who owns the code |
| `referred_id` | `BIGINT` | NO | — | FK → `users(id)` who used the code |
| `code_used` | `VARCHAR(20)` | NO | — | Snapshot of code at signup |
| `status` | `VARCHAR(20)` | NO | `'pending'` | `pending`, `confirmed`, `rewarded`, `rejected` |
| `reward_amount` | `NUMERIC(12,2)` | YES | NULL | Bonus given to referrer |
| `confirmed_at` | `TIMESTAMPTZ` | YES | NULL | |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK referrals (id)`
- `FK fk_ref_referrer FOREIGN KEY (referrer_id) REFERENCES users(id) ON DELETE RESTRICT`
- `FK fk_ref_referred FOREIGN KEY (referred_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE uq_ref_referred (referred_id)` — each user can only be referred once
- `INDEX idx_ref_referrer (referrer_id)`

---

### 3.11 `sales_targets`

Monthly/daily targets per agent for the dashboard stat cards and progress meter.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | `BIGSERIAL` | NO | — | Primary key |
| `user_id` | `BIGINT` | NO | — | FK → `users(id)` |
| `period_year` | `SMALLINT` | NO | — | e.g., `2025` |
| `period_month` | `SMALLINT` | NO | — | 1–12 |
| `target_visits` | `INTEGER` | NO | `0` | Monthly visit target |
| `target_prospects` | `INTEGER` | NO | `0` | Monthly new-prospect target |
| `target_conversion_pct` | `NUMERIC(5,2)` | YES | NULL | Expected conversion rate % |
| `target_revenue` | `NUMERIC(14,2)` | YES | NULL | Optional revenue target |
| `created_at` | `TIMESTAMPTZ` | NO | `now()` | |
| `updated_at` | `TIMESTAMPTZ` | NO | `now()` | |

**Constraints & Indexes:**
- `PK sales_targets (id)`
- `FK fk_target_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`
- `UNIQUE uq_target_user_period (user_id, period_year, period_month)` — one target set per user per month
- `INDEX idx_target_period (period_year, period_month)`
- `CHECK (period_month BETWEEN 1 AND 12)`

---

## 4. View: `v_dashboard_stats`

A computed view to support the home screen stat card and meter, avoiding repeated aggregation at the application layer.

```sql
CREATE OR REPLACE VIEW v_dashboard_stats AS
SELECT
    u.id                    AS user_id,
    CURRENT_DATE            AS stat_date,
    COALESCE(t.target_visits, 0)          AS target_visits_month,
    COALESCE(t.target_prospects, 0)       AS target_prospects_month,
    COALESCE(t.target_conversion_pct, 0)  AS target_conversion_pct,
    (
        SELECT COUNT(*) FROM visits v
        WHERE v.user_id = u.id
          AND v.visited_at::date = CURRENT_DATE
    )                                    AS today_visits,
    (
        SELECT COUNT(*) FROM visits v
        WHERE v.user_id = u.id
          AND DATE_TRUNC('month', v.visited_at) = DATE_TRUNC('month', CURRENT_DATE)
    )                                    AS month_visits,
    (
        SELECT COUNT(*) FROM prospects p
        WHERE p.user_id = u.id
    )                                    AS total_prospects,
    (
        SELECT COUNT(*) FROM prospects p
        WHERE p.user_id = u.id
          AND p.status = 'won'
    )                                    AS won_prospects,
    CASE
        WHEN COUNT(p.id) OVER (PARTITION BY u.id) = 0 THEN 0
        ELSE ROUND(
            100.0 * SUM(CASE WHEN p.status = 'won' THEN 1 ELSE 0 END)
                  OVER (PARTITION BY u.id)
            / COUNT(p.id) OVER (PARTITION BY u.id), 2)
    END                                  AS conversion_pct,
    (
        SELECT a.status FROM attendances a
        WHERE a.user_id = u.id AND a.attendance_date = CURRENT_DATE
    )                                    AS today_attendance_status
FROM users u
LEFT JOIN sales_targets t
       ON t.user_id = u.id
      AND t.period_year  = EXTRACT(YEAR  FROM CURRENT_DATE)::int
      AND t.period_month = EXTRACT(MONTH FROM CURRENT_DATE)::int
LEFT JOIN prospects p ON p.user_id = u.id;
```

---

## 5. Database-Level Constraints Summary

| Constraint Type | Location | Rule |
|---|---|---|
| **PK** | All tables | `id BIGSERIAL PRIMARY KEY` |
| **UNIQUE** | `users.email` | No duplicate login emails |
| **UNIQUE** | `users.phone` | No duplicate phones |
| **UNIQUE** | `attendances(user_id, attendance_date)` | One attendance row per user per day |
| **UNIQUE** | `referral_codes.user_id` | One referral code per user |
| **UNIQUE** | `referral_codes.code` (case-insensitive) | Globally unique referral codes |
| **UNIQUE** | `referrals.referred_id` | A user can only be referred once |
| **UNIQUE** | `sales_targets(user_id, period_year, period_month)` | One target per user per month |
| **FK CASCADE** | Sessions, attendances, photos, notifications → user/parent | Child rows deleted with parent |
| **FK RESTRICT** | Prospects → users, Visits → user | Cannot delete agent with assigned prospects/visits |
| **CHECK** | `attendances.check_out_at > check_in_at` | Logical time ordering |
| **CHECK** | Enum-like columns (`status`, `severity`, `role`, `outcome`, `photo_type`) | Validates allowed values |

---

## 6. Index Strategy

| Index | Table(s) | Purpose |
|---|---|---|
| `idx_sessions_user` | `user_sessions` | Look up active sessions by user |
| `idx_att_user_date` | `attendances` | Fetch today's / historical attendance |
| `idx_att_status` | `attendances` | Filter pending/completed attendances |
| `idx_attphoto_att` | `attendance_photos` | Load photos per attendance record |
| `idx_prosp_user_created` | `prospects` | Prospect list ordered by recency (grouped by day) |
| `idx_prosp_status` | `prospects` | Pipeline filter by status |
| `idx_prosp_name_trgm` (GIN) | `prospects` | Search-bar fuzzy search |
| `idx_prosp_latlng` (GiST) | `prospects` | Nearby-prospect / map queries |
| `idx_prosp_photo_prosp` | `prospect_photos` | Load photos for prospect detail |
| `idx_visit_user_date` | `visits` | "Today's visits" stat + visit history |
| `idx_visit_prosp` | `visits` | All visits for a prospect |
| `idx_notif_user_unread` | `notifications` | Notification list + unread badge count |
| `idx_refcode_code_upper` | `referral_codes` | Referral-code lookup at signup |

---

## 7. Data Volume & Scaling Notes

| Table | Expected Growth | Retention Policy |
|---|---|---|
| `attendances` | ~1 row/user/day | Retain 2 years; archive to cold storage |
| `attendance_photos` | ~2 rows/attendance | Retain 1 year; offload to object storage |
| `visits` | ~5–20 rows/user/day | Retain 2 years |
| `prospects` | ~50–200/user | Permanent |
| `prospect_photos` | ~2 per prospect | Permanent (object storage) |
| `notifications` | ~5–20/user/day | Auto-purge read notifications after 90 days |
| `user_sessions` | High turnover | Purge expired/revoked sessions nightly |

---

## 8. Future Considerations (Out of Current Scope)

- **`sales_orders`** / **`transactions`** table — when POS/order-taking is added.
- **`visit_checklists`** — structured checklist per visit type.
- **`geofences`** — geofence validation for clock-in/out (currently only raw GPS stored).
- **`audit_log`** — generic audit trail for compliance.
- **Partitioning** on `visits` and `attendances` by month once row counts exceed ~10M.

---

*End of Database PRD.*