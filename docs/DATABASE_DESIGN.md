# SmartExam – Database Design

**Document Version:** 1.0  
**Date:** 2026-11-18  
**Status:** Approved – Module 0  
**Author:** SmartExam Development Team

---

## Table of Contents

1. [Design Overview](#1-design-overview)
2. [Database Technology](#2-database-technology)
3. [Core Tables (15 Tables)](#3-core-tables-15-tables)
   - [01. users](#table-01-users)
   - [02. students](#table-02-students)
   - [03. classrooms](#table-03-classrooms)
   - [04. seats](#table-04-seats)
   - [05. devices](#table-05-devices)
   - [06. examinations](#table-06-examinations)
   - [07. exam_seat_assignments](#table-07-exam_seat_assignments)
   - [08. question_papers](#table-08-question_papers)
   - [09. paper_assignments](#table-09-paper_assignments)
   - [10. paper_distribution_log](#table-10-paper_distribution_log)
   - [11. device_connections](#table-11-device_connections)
   - [12. audit_logs](#table-12-audit_logs)
   - [13. refresh_tokens](#table-13-refresh_tokens)
   - [14. device_exam_state](#table-14-device_exam_state)
   - [15. system_config](#table-15-system_config)
4. [Entity Relationship Diagram](#4-entity-relationship-diagram)
5. [Relationship Summary](#5-relationship-summary)
6. [Index Strategy](#6-index-strategy)
7. [Data Integrity Rules](#7-data-integrity-rules)
8. [Security Considerations](#8-security-considerations)
9. [Migration Strategy](#9-migration-strategy)

---

## 1. Design Overview

SmartExam uses a relational database schema designed around the examination delivery workflow:

```
Users (admin/invigilator) → manage →
Students → assigned to →
Seats (in Classrooms) → linked to →
Devices (Android tablets) → for →
Examinations → use →
Question Papers → distributed via →
Paper Distribution Log + Device Exam State
```

All tables use surrogate primary keys (UUID or auto-increment BIGINT). Soft deletion is used throughout — no records are permanently deleted; they are deactivated.

### Design Principles

1. **Soft deletes only** — `is_active` / `deleted_at` columns instead of physical row deletion.
2. **Full audit trail** — `created_at`, `updated_at`, `created_by` on every table.
3. **No plaintext secrets** — no passwords, keys, or PINs in any table.
4. **Foreign key enforcement** — all relationships enforced at the database level.
5. **UTF8MB4 charset** — supports full Unicode including emoji (future-proof for student names).

---

## 2. Database Technology

| Item | Detail |
|---|---|
| Engine | MySQL 8.4 |
| Character Set | `utf8mb4` |
| Collation | `utf8mb4_unicode_ci` |
| Storage Engine | InnoDB (all tables — required for FK constraints) |
| ORM | SQLAlchemy 2.x (async) |
| Migration Tool | Alembic |
| Connection | localhost only (127.0.0.1:3306) |

---

## 3. Core Tables (15 Tables)

---

### Table 01: `users`

**Purpose:** Stores Admin and Invigilator accounts. No student login accounts are stored here.

```sql
CREATE TABLE users (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid          CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    full_name     VARCHAR(255) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,       -- bcrypt, never plaintext
    role          ENUM('admin', 'invigilator') NOT NULL,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by    BIGINT UNSIGNED NULL,        -- NULL for first admin (bootstrap)
    last_login_at DATETIME NULL,

    CONSTRAINT fk_users_created_by FOREIGN KEY (created_by)
        REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- `password_hash`: Always bcrypt. Application layer enforces this — column is named to prevent accidental plaintext storage.
- `role`: Only two roles exist in MVP. ENUM is appropriate. Future roles would require an ALTER.
- `created_by`: Self-referencing FK. The first Admin account has NULL (created by bootstrap script).

---

### Table 02: `students`

**Purpose:** Stores student records managed by Admin. Students do not log in.

```sql
CREATE TABLE students (
    id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid         CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    student_code VARCHAR(50) NOT NULL UNIQUE,  -- Institution's student ID number
    full_name    VARCHAR(255) NOT NULL,
    class_name   VARCHAR(100) NOT NULL,         -- e.g. "10-A", "12-B"
    section      VARCHAR(50) NULL,
    roll_number  VARCHAR(50) NULL,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by   BIGINT UNSIGNED NOT NULL,

    CONSTRAINT fk_students_created_by FOREIGN KEY (created_by)
        REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- `student_code`: The institution's own student ID (roll number, admission number). Unique.
- Soft delete via `is_active`: Historical exam records referencing this student remain valid.
- No `password_hash` column: Students have zero digital credentials in this system.

---

### Table 03: `classrooms`

**Purpose:** Physical examination rooms where exams are conducted.

```sql
CREATE TABLE classrooms (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid        CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    name        VARCHAR(100) NOT NULL UNIQUE,   -- e.g. "Lab A", "Hall 1"
    capacity    SMALLINT UNSIGNED NOT NULL,      -- Maximum seats
    location    VARCHAR(255) NULL,              -- Optional: building, floor
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by  BIGINT UNSIGNED NOT NULL,

    CONSTRAINT fk_classrooms_created_by FOREIGN KEY (created_by)
        REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### Table 04: `seats`

**Purpose:** Individual physical seats within a classroom. Each seat can host one tablet.

```sql
CREATE TABLE seats (
    id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid         CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    classroom_id BIGINT UNSIGNED NOT NULL,
    seat_label   VARCHAR(50) NOT NULL,           -- e.g. "A1", "Row-2-Seat-3"
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uq_classroom_seat (classroom_id, seat_label),
    CONSTRAINT fk_seats_classroom FOREIGN KEY (classroom_id)
        REFERENCES classrooms(id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- `seat_label` is unique within a classroom (composite unique key).
- `ON DELETE RESTRICT`: Cannot delete a classroom that has seats.

---

### Table 05: `devices`

**Purpose:** Registered Android tablets. Each device has a unique token used for authentication.

```sql
CREATE TABLE devices (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    device_name     VARCHAR(100) NOT NULL,         -- Human label: "Tablet-01"
    android_id      VARCHAR(100) NOT NULL UNIQUE,  -- Android device fingerprint
    device_token_hash VARCHAR(255) NOT NULL,        -- bcrypt hash of token stored in Android Keystore
    model           VARCHAR(100) NULL,             -- Device model (e.g. Samsung Galaxy Tab)
    android_version VARCHAR(50) NULL,
    status          ENUM('pending', 'active', 'disabled', 'deregistered') NOT NULL DEFAULT 'pending',
    registered_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    approved_at     DATETIME NULL,
    approved_by     BIGINT UNSIGNED NULL,
    last_seen_at    DATETIME NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    CONSTRAINT fk_devices_approved_by FOREIGN KEY (approved_by)
        REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- `device_token_hash`: The actual token is stored in Android Keystore. We store only the hash to verify it on login.
- `android_id`: Hardware fingerprint to prevent duplicate registrations.
- `status = pending`: Devices require Admin approval before they can connect.
- No plaintext token stored in DB.

---

### Table 06: `examinations`

**Purpose:** Examination sessions managed by Admin, controlled by Invigilator.

```sql
CREATE TABLE examinations (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    name            VARCHAR(255) NOT NULL,          -- e.g. "Mathematics Final Exam 2026"
    subject         VARCHAR(100) NOT NULL,
    classroom_id    BIGINT UNSIGNED NOT NULL,
    invigilator_id  BIGINT UNSIGNED NOT NULL,
    scheduled_date  DATE NOT NULL,
    start_time      TIME NOT NULL,
    duration_minutes SMALLINT UNSIGNED NOT NULL,
    status          ENUM('draft', 'scheduled', 'active', 'ended', 'cancelled') NOT NULL DEFAULT 'draft',
    actual_start_at DATETIME NULL,                  -- Recorded when Invigilator actually starts
    actual_end_at   DATETIME NULL,
    notes           TEXT NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by      BIGINT UNSIGNED NOT NULL,

    CONSTRAINT fk_exam_classroom FOREIGN KEY (classroom_id)
        REFERENCES classrooms(id),
    CONSTRAINT fk_exam_invigilator FOREIGN KEY (invigilator_id)
        REFERENCES users(id),
    CONSTRAINT fk_exam_created_by FOREIGN KEY (created_by)
        REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Status Lifecycle:**
```
draft → scheduled → active → ended
                           → cancelled
```

---

### Table 07: `exam_seat_assignments`

**Purpose:** Links one student, one seat, and one device for one specific examination.

```sql
CREATE TABLE exam_seat_assignments (
    id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid           CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    examination_id BIGINT UNSIGNED NOT NULL,
    student_id     BIGINT UNSIGNED NOT NULL,
    seat_id        BIGINT UNSIGNED NOT NULL,
    device_id      BIGINT UNSIGNED NOT NULL,
    created_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by     BIGINT UNSIGNED NOT NULL,

    -- One student per exam
    UNIQUE KEY uq_exam_student (examination_id, student_id),
    -- One seat per exam
    UNIQUE KEY uq_exam_seat (examination_id, seat_id),
    -- One device per exam
    UNIQUE KEY uq_exam_device (examination_id, device_id),

    CONSTRAINT fk_esa_exam    FOREIGN KEY (examination_id) REFERENCES examinations(id),
    CONSTRAINT fk_esa_student FOREIGN KEY (student_id)     REFERENCES students(id),
    CONSTRAINT fk_esa_seat    FOREIGN KEY (seat_id)        REFERENCES seats(id),
    CONSTRAINT fk_esa_device  FOREIGN KEY (device_id)      REFERENCES devices(id),
    CONSTRAINT fk_esa_creator FOREIGN KEY (created_by)     REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- Three separate UNIQUE constraints prevent double-booking at the database level.
- This is the central junction table of the entire system.

---

### Table 08: `question_papers`

**Purpose:** Metadata about uploaded question papers. The encrypted file lives on the filesystem.

```sql
CREATE TABLE question_papers (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid              CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    title             VARCHAR(255) NOT NULL,
    subject           VARCHAR(100) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    encrypted_filename VARCHAR(255) NOT NULL UNIQUE, -- UUID-based .enc filename
    file_size_bytes   BIGINT UNSIGNED NOT NULL,       -- Size of original PDF
    sha256_hash       CHAR(64) NOT NULL,              -- SHA-256 of original PDF (hex)
    page_count        SMALLINT UNSIGNED NULL,
    upload_status     ENUM('uploading', 'ready', 'failed') NOT NULL DEFAULT 'uploading',
    uploaded_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    uploaded_by       BIGINT UNSIGNED NOT NULL,
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,

    CONSTRAINT fk_qp_uploaded_by FOREIGN KEY (uploaded_by)
        REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- `encrypted_filename`: Stored as UUID-based name so the original filename is never exposed on the filesystem.
- `sha256_hash`: Computed from the original plaintext PDF. Tablets verify this after decryption.
- Encryption key is NOT stored here — it lives in the separate key store directory.
- File content is NOT stored in the database — only the filesystem path metadata.

---

### Table 09: `paper_assignments`

**Purpose:** Links a question paper to an examination.

```sql
CREATE TABLE paper_assignments (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    examination_id  BIGINT UNSIGNED NOT NULL UNIQUE, -- One paper per exam
    paper_id        BIGINT UNSIGNED NOT NULL,
    assigned_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    assigned_by     BIGINT UNSIGNED NOT NULL,

    CONSTRAINT fk_pa_exam  FOREIGN KEY (examination_id) REFERENCES examinations(id),
    CONSTRAINT fk_pa_paper FOREIGN KEY (paper_id)       REFERENCES question_papers(id),
    CONSTRAINT fk_pa_user  FOREIGN KEY (assigned_by)    REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### Table 10: `paper_distribution_log`

**Purpose:** Records every paper download event for each device during an exam. Audit trail for distribution.

```sql
CREATE TABLE paper_distribution_log (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    examination_id  BIGINT UNSIGNED NOT NULL,
    device_id       BIGINT UNSIGNED NOT NULL,
    student_id      BIGINT UNSIGNED NOT NULL,
    event_type      ENUM(
                        'download_started',
                        'download_completed',
                        'integrity_check_passed',
                        'integrity_check_failed',
                        'paper_displayed',
                        'paper_cleared'
                    ) NOT NULL,
    event_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    details         JSON NULL,                   -- Optional extra context

    CONSTRAINT fk_pdl_exam    FOREIGN KEY (examination_id) REFERENCES examinations(id),
    CONSTRAINT fk_pdl_device  FOREIGN KEY (device_id)      REFERENCES devices(id),
    CONSTRAINT fk_pdl_student FOREIGN KEY (student_id)     REFERENCES students(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### Table 11: `device_connections`

**Purpose:** Records WebSocket connection and disconnection events for each device in each exam.

```sql
CREATE TABLE device_connections (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    device_id       BIGINT UNSIGNED NOT NULL,
    examination_id  BIGINT UNSIGNED NOT NULL,
    connected_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    disconnected_at DATETIME NULL,
    disconnect_reason VARCHAR(255) NULL,         -- 'clean_close', 'timeout', 'exam_ended'
    ip_address      VARCHAR(45) NULL,            -- IPv4 or IPv6

    CONSTRAINT fk_dc_device FOREIGN KEY (device_id)      REFERENCES devices(id),
    CONSTRAINT fk_dc_exam   FOREIGN KEY (examination_id) REFERENCES examinations(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### Table 12: `audit_logs`

**Purpose:** Immutable record of all significant system events. Append-only by design.

```sql
CREATE TABLE audit_logs (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid          CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    actor_type    ENUM('user', 'device', 'system') NOT NULL,
    actor_id      BIGINT UNSIGNED NULL,          -- user.id or device.id; NULL for system events
    actor_name    VARCHAR(255) NULL,             -- Denormalized: name at time of event
    action        VARCHAR(100) NOT NULL,         -- e.g. 'exam.started', 'paper.uploaded'
    entity_type   VARCHAR(100) NULL,             -- e.g. 'examination', 'question_paper'
    entity_id     BIGINT UNSIGNED NULL,          -- ID of affected entity
    outcome       ENUM('success', 'failure', 'warning') NOT NULL,
    ip_address    VARCHAR(45) NULL,
    details       JSON NULL,                     -- Structured extra context (no paper content)
    logged_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP

    -- No UPDATE or DELETE allowed on this table (enforced at application layer)
    -- No foreign keys: audit log must remain even if referenced records are deleted
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Key Design Decisions:**
- No foreign keys: So that deleting a device or user does not cascade into the audit log.
- `actor_name` is denormalized: Preserves the name at the time of the event.
- No `updated_at`: This table is append-only. Application layer must never issue UPDATE or DELETE against it.

---

### Table 13: `refresh_tokens`

**Purpose:** Stores refresh token metadata for session management. Actual token value is hashed.

```sql
CREATE TABLE refresh_tokens (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid        CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    user_id     BIGINT UNSIGNED NOT NULL,
    token_hash  CHAR(64) NOT NULL UNIQUE,         -- SHA-256 hash of the token
    issued_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at  DATETIME NOT NULL,
    revoked_at  DATETIME NULL,                    -- NULL = still valid
    revoked_by  BIGINT UNSIGNED NULL,
    ip_address  VARCHAR(45) NULL,
    user_agent  VARCHAR(500) NULL,

    CONSTRAINT fk_rt_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### Table 14: `device_exam_state`

**Purpose:** Tracks the last known state of each device within an examination. Used for monitoring dashboard and emergency recovery.

```sql
CREATE TABLE device_exam_state (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            CHAR(36) NOT NULL UNIQUE DEFAULT (UUID()),
    device_id       BIGINT UNSIGNED NOT NULL,
    examination_id  BIGINT UNSIGNED NOT NULL,
    state           ENUM(
                        'offline',
                        'connected',
                        'paper_downloaded',
                        'paper_displayed',
                        'exam_ended'
                    ) NOT NULL DEFAULT 'offline',
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uq_device_exam (device_id, examination_id),

    CONSTRAINT fk_des_device FOREIGN KEY (device_id)      REFERENCES devices(id),
    CONSTRAINT fk_des_exam   FOREIGN KEY (examination_id) REFERENCES examinations(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Purpose:** This table is the source of truth for the Invigilator's real-time monitoring dashboard. It shows which tablets are ready, displaying the paper, or offline.

---

### Table 15: `system_config`

**Purpose:** Key-value configuration table for runtime-adjustable system parameters.

```sql
CREATE TABLE system_config (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    config_key  VARCHAR(100) NOT NULL UNIQUE,
    config_value TEXT NOT NULL,
    description VARCHAR(500) NULL,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by  BIGINT UNSIGNED NULL,

    CONSTRAINT fk_sc_updated_by FOREIGN KEY (updated_by)
        REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Initial Config Rows:**

| `config_key` | `config_value` | Description |
|---|---|---|
| `jwt_access_expiry_minutes` | `60` | JWT access token lifetime |
| `jwt_refresh_expiry_days` | `7` | Refresh token lifetime |
| `max_login_attempts` | `5` | Before account lockout |
| `max_devices` | `30` | Maximum registered tablets |
| `exam_buffer_minutes` | `10` | How early tablets can connect before exam start |
| `paper_cache_hours` | `24` | How long to retain encrypted paper on tablet |

---

## 4. Entity Relationship Diagram

```
users ──────────────── creates ──────────────► students
  │                                                │
  │ creates                                        │ assigned_to
  ▼                                                ▼
examinations ──── assigned_invigilator ──── users  exam_seat_assignments
  │                                              ▲       ▲         ▲
  │ held_in                                      │       │         │
  ▼                                              │       │         │
classrooms ──── contains ──► seats ─────────────┘       │         │
                                                         │         │
                                        devices ─────────┘         │
                                            │                      │
                                            └── student_id ────────┘
                                                         
examinations ◄──── paper_assignments ────► question_papers
  │
  ├──► paper_distribution_log
  ├──► device_connections
  ├──► device_exam_state
  └──► audit_logs

users ──► refresh_tokens
users ──► audit_logs (as actor)
devices ──► audit_logs (as actor)
```

---

## 5. Relationship Summary

| Relationship | Type | Table |
|---|---|---|
| User creates Students | One-to-Many | `students.created_by → users.id` |
| Classroom has Seats | One-to-Many | `seats.classroom_id → classrooms.id` |
| Exam held in Classroom | Many-to-One | `examinations.classroom_id → classrooms.id` |
| Exam assigned Invigilator | Many-to-One | `examinations.invigilator_id → users.id` |
| Exam has one Paper | One-to-One | `paper_assignments.examination_id UNIQUE` |
| Student ↔ Seat ↔ Device per Exam | Many-to-Many (resolved) | `exam_seat_assignments` |
| Device has distribution events | One-to-Many | `paper_distribution_log.device_id` |
| Device has connection events | One-to-Many | `device_connections.device_id` |
| Device has one state per exam | One-to-One | `device_exam_state` UNIQUE |
| User has refresh tokens | One-to-Many | `refresh_tokens.user_id` |

---

## 6. Index Strategy

| Table | Index | Type | Reason |
|---|---|---|---|
| `users` | `email` | UNIQUE | Login lookup |
| `students` | `student_code` | UNIQUE | Search by institution ID |
| `students` | `class_name` | INDEX | Filter by class |
| `devices` | `android_id` | UNIQUE | Prevent duplicate registration |
| `devices` | `status` | INDEX | Filter active devices |
| `examinations` | `status` | INDEX | Filter active exams |
| `examinations` | `scheduled_date` | INDEX | Upcoming exam queries |
| `exam_seat_assignments` | `examination_id, student_id` | UNIQUE | Prevent double assignment |
| `audit_logs` | `logged_at` | INDEX | Time-range queries |
| `audit_logs` | `actor_id, actor_type` | INDEX | Filter by actor |
| `audit_logs` | `action` | INDEX | Filter by action type |
| `paper_distribution_log` | `examination_id, device_id` | INDEX | Per-exam per-device lookup |
| `device_exam_state` | `device_id, examination_id` | UNIQUE | Real-time state lookup |
| `refresh_tokens` | `token_hash` | UNIQUE | Token validation |
| `refresh_tokens` | `user_id, revoked_at` | INDEX | Active tokens per user |

---

## 7. Data Integrity Rules

| Rule | Enforcement |
|---|---|
| A device can only be assigned to one exam at a time | Application-layer check before INSERT into `exam_seat_assignments` |
| An exam cannot start without a paper assigned | Application-layer check on status transition |
| An exam cannot start before scheduled date/time | Application-layer check on status transition |
| `audit_logs` is append-only | Application-layer: no UPDATE/DELETE queries issued against this table |
| Soft delete only | Application-layer: `is_active = FALSE` or `deleted_at = NOW()` instead of DELETE |
| Paper key never stored in DB | Application-layer: key file written to filesystem only |
| Decrypted PDF never persisted | Application-layer: decryption in-memory, never written to disk |

---

## 8. Security Considerations

| Item | Implementation |
|---|---|
| `users.password_hash` | bcrypt minimum cost factor 12 |
| `devices.device_token_hash` | bcrypt hashed before storage |
| `refresh_tokens.token_hash` | SHA-256 of random 256-bit token |
| `question_papers.encrypted_filename` | UUID filename — no path traversal risk |
| `audit_logs.details` | JSON must never contain paper content, full decrypted data, or key material |
| MySQL connection | `127.0.0.1` only, never exposed on network |
| DB credentials | Loaded from `.env` file, never hardcoded |

---

## 9. Migration Strategy

All schema changes are managed through **Alembic** versioned migration scripts.

```
database/
└── migrations/
    ├── 001_initial_schema.sql     # Base tables (created Module 1)
    ├── 002_seed_system_config.sql # Initial system_config rows
    └── ...                        # Future changes
```

### Migration Rules

1. Never edit existing migration files after they have been applied.
2. Each migration must include a rollback (downgrade) script.
3. Schema changes require discussion and approval before implementation.
4. Migrations are applied in numeric order.
5. The `alembic_version` table (managed by Alembic) tracks the current schema version.
