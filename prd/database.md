# Database Product Requirements Document (PRD)
**Project:** Sales Point Mobile Application
**Document Version:** 1.0
**Role:** Senior Database Architect
**Target RDBMS:** PostgreSQL (Recommended for robust spatial, JSON, and relational support)

## 1. Overview
This document outlines the database architecture for the "Sales Point" mobile application. Based on the application mockup, the system serves as a field-sales and attendance tracking tool. It includes features for employee clock-in/out with geolocation, prospect/customer management with photo documentation (KTP/storefront), notifications, and core sales point metrics.

## 2. Database Design Principles
- **Normalization:** The schema is normalized to 3NF to reduce data redundancy, with selective denormalization for reporting metrics.
- **Data Integrity:** Strict use of Foreign Keys, CHECK constraints, and NOT NULL flags.
- **Auditability:** All primary entities include `created_at` and `updated_at` timestamps.
- **Security:** Personally Identifiable Information (PII) such as phone numbers and ID card photos are stored securely; access is role-based.

---

## 3. Entity Relationship Diagram (Textual Representation)
- **Users** (1) ──── (M) **Attendances**
- **Users** (1) ──── (M) **Prospects**
- **Prospects** (1) ──── (M) **Prospect_Media**
- **Prospects** (1) ──── (M) **Transactions**
- **Transactions** (1) ──── (M) **Transaction_Items**
- **Products** (1) ──── (M) **Transaction_Items**
- **Users** (1) ──── (M) **Notifications**

---

## 4. Detailed Schema Definitions

### 4.1 `users`
Stores salesperson/staff account information and tracking preferences.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `user_id` | UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `username` | VARCHAR(50) | UNIQUE, NOT NULL | Login username |
| `password_hash` | VARCHAR(255) | NOT NULL | Hashed password |
| `full_name` | VARCHAR(100) | NOT NULL | Full legal name |
| `email` | VARCHAR(255) | UNIQUE | Email address |
| `phone` | VARCHAR(20) | | Phone number |
| `role` | VARCHAR(20) | NOT NULL, DEFAULT 'sales' | 'admin', 'sales' |
| `referral_code`| VARCHAR(15) | UNIQUE, NOT NULL | User's unique referral code |
| `avatar_url` | VARCHAR(255) | | Path to profile avatar |
| `is_active` | BOOLEAN | DEFAULT TRUE | Account status |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.2 `attendances`
Tracks daily clock-in and clock-out events with geographic data.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `attendance_id`| UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `user_id` | UUID | FK -> users(user_id), NOT NULL | Salesperson |
| `clock_in_time` | TIMESTAMP | NOT NULL | Check-in timestamp |
| `clock_out_time`| TIMESTAMP | | Check-out timestamp (null if active) |
| `clock_in_lat` | DECIMAL(10,8)| NOT NULL | Check-in latitude |
| `clock_in_lng` | DECIMAL(11,8)| NOT NULL | Check-in longitude |
| `clock_out_lat` | DECIMAL(10,8)| | Check-out latitude |
| `clock_out_lng` | DECIMAL(11,8)| | Check-out longitude |
| `duration_mins` | INTEGER | | Calculated duration in minutes |
| `status` | VARCHAR(15) | NOT NULL, DEFAULT 'in_progress' | 'in_progress', 'completed' |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.3 `prospects`
Represents potential or existing customers visited by sales staff.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `prospect_id` | UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `salesperson_id`| UUID | FK -> users(user_id), NOT NULL | Assigned sales rep |
| `name` | VARCHAR(150) | NOT NULL | Prospect/Store name |
| `address` | TEXT | | Store/address location |
| `phone` | VARCHAR(20) | | Contact number |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'new' | 'new', 'contacted', 'won', 'lost' |
| `notes` | TEXT | | Meeting notes |
| `last_visit` | TIMESTAMP | | Last interaction date |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.4 `prospect_media`
Stores metadata for photos captured during prospecting (e.g., KTP, Storefront).
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `media_id` | UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `prospect_id` | UUID | FK -> prospects(prospect_id), NOT NULL | Linked prospect |
| `file_path` | VARCHAR(255) | NOT NULL | S3 / Storage bucket URL |
| `media_type` | VARCHAR(20) | NOT NULL, CHECK IN (...) | 'ktp', 'storefront', 'selfie', 'other' |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.5 `products`
Inventory items available for sale.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `product_id` | UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `sku` | VARCHAR(50) | UNIQUE, NOT NULL | Stock Keeping Unit |
| `name` | VARCHAR(150) | NOT NULL | Product name |
| `description` | TEXT | | Product details |
| `price` | DECIMAL(12,2)| NOT NULL, CHECK >= 0 | Selling price |
| `cost` | DECIMAL(12,2)| NOT NULL, CHECK >= 0 | Cost of goods |
| `stock_qty` | INTEGER | NOT NULL, DEFAULT 0 | Current inventory |
| `is_active` | BOOLEAN | DEFAULT TRUE | Product availability |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.6 `transactions`
Records sales transactions (orders) linked to prospects and sales reps.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `transaction_id`| UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `invoice_no` | VARCHAR(30) | UNIQUE, NOT NULL | Human-readable invoice ID |
| `salesperson_id`| UUID | FK -> users(user_id), NOT NULL | Who made the sale |
| `prospect_id` | UUID | FK -> prospects(prospect_id) | Converted prospect (nullable if direct sale) |
| `total_amount` | DECIMAL(12,2)| NOT NULL, CHECK >= 0 | Total sale value |
| `status` | VARCHAR(15) | NOT NULL, DEFAULT 'pending' | 'pending', 'completed', 'cancelled' |
| `transaction_date`| TIMESTAMP | NOT NULL, DEFAULT NOW() | Sale timestamp |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

### 4.7 `transaction_items`
Line items for each transaction.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `item_id` | UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `transaction_id`| UUID | FK -> transactions(transaction_id), NOT NULL | Parent transaction |
| `product_id` | UUID | FK -> products(product_id), NOT NULL | Product sold |
| `quantity` | INTEGER | NOT NULL, CHECK > 0 | Items sold |
| `unit_price` | DECIMAL(12,2)| NOT NULL | Price at time of sale |
| `subtotal` | DECIMAL(12,2)| NOT NULL | quantity * unit_price |

### 4.8 `notifications`
In-app notifications for sales targets, alerts, and updates.
| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `notification_id`| UUID | PK, DEFAULT gen_random_uuid() | Unique identifier |
| `user_id` | UUID | FK -> users(user_id), NOT NULL | Recipient |
| `title` | VARCHAR(100) | NOT NULL | Alert title |
| `body` | TEXT | NOT NULL | Alert message |
| `type` | VARCHAR(15) | NOT NULL, DEFAULT 'info' | 'info', 'success', 'warning', 'error' |
| `is_read` | BOOLEAN | DEFAULT FALSE | Read status |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

---

## 5. Indexing Strategy

To optimize query performance for the UI views (Home dashboard, Prospects list, Attendance history), the following indexes are defined:

### Primary & Foreign Key Indexes (Automatic via constraints)
- All Primary Keys (`user_id`, `prospect_id`, etc.)
- All Foreign Keys (`salesperson_id`, `prospect_id`, `transaction_id`, etc.)

### Performance Indexes
```sql
-- Users / Login
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_referral_code ON users(referral_code);

-- Attendances (Home dashboard today's status, history lists)
CREATE INDEX idx_attendances_user_date ON attendances(user_id, clock_in_time DESC);
CREATE INDEX idx_attendances_status ON attendances(status);

-- Prospects (List views, filtering by salesperson and status badges)
CREATE INDEX idx_prospects_salesperson ON prospects(salesperson_id);
CREATE INDEX idx_prospects_status ON prospects(status);
CREATE INDEX idx_prospects_created_at ON prospects(created_at DESC);

-- Transactions (Reporting and Stats)
CREATE INDEX idx_transactions_salesperson_date ON transactions(salesperson_id, transaction_date DESC);
CREATE INDEX idx_transactions_prospect ON transactions(prospect_id);
CREATE INDEX idx_transactions_status ON transactions(status);

-- Notifications
CREATE INDEX idx_notifications_user_read ON notifications(user_id, is_read, created_at DESC);
```

## 6. Constraints & Data Integrity Rules

1. **Check Constraints:**
   - `CHECK (products.price >= 0)` prevents negative pricing.
   - `CHECK (transaction_items.quantity > 0)` prevents zero-quantity sales.
   - `CHECK (attendances.duration_mins >= 0)` ensures logical time tracking.
2. **Unique Constraints:**
   - `users.username` and `users.referral_code` must be globally unique to prevent login collisions and referral fraud.
   - `products.sku` must be unique.
3. **Referential Actions:**
   - `ON DELETE RESTRICT` for `transaction_items` referencing `transactions` and `products` (prevents deleting products with historical sales).
   - `ON DELETE CASCADE` for `prospect_media` referencing `prospects` (if a prospect is purged, their uploaded KTP/Storefront photos are also purged).
4. **Geospatial Validation:**
   - While standard DECIMAL types are used for lat/lng, application logic must validate ranges (Lat: -90 to 90, Lng: -180 to 180) before insertion.