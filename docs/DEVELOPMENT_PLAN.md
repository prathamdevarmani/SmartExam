# SmartExam – Development Plan

**Document Version:** 1.0  
**Date:** 2026-11-18  
**Status:** Module 0 Complete  
**Author:** SmartExam Development Team

---

## Table of Contents

1. [Development Philosophy](#1-development-philosophy)
2. [Module Overview](#2-module-overview)
3. [Module Status Tracker](#3-module-status-tracker)
4. [Detailed Module Plans](#4-detailed-module-plans)
5. [Definition of Done](#5-definition-of-done)
6. [Prerequisites Checklist](#6-prerequisites-checklist)
7. [Risk Register](#7-risk-register)
8. [Technology Version Reference](#8-technology-version-reference)

---

## 1. Development Philosophy

### Core Rules

1. **Module by module** — One module at a time. The next module does not begin until the current one is reviewed and approved.
2. **Inspect before modifying** — Always read existing code before changing it. No blind overwrites.
3. **No full rewrites** — Existing working architecture is never replaced without discussion and approval.
4. **Document decisions** — Every significant design or implementation choice is recorded in `ARCHITECTURE.md`.
5. **Security first** — Security requirements are implemented from day one, not added later.
6. **Tests before done** — Automated tests are written and passing before any module is declared complete.
7. **Clean code** — PEP 8 (Python), ESLint (TypeScript), ktlint (Kotlin). No commented-out dead code.
8. **Appropriate error handling** — All failure paths are handled. No silent exceptions.
9. **Logging** — Structured logs at appropriate levels (DEBUG/INFO/WARNING/ERROR).
10. **Never commit secrets** — `.env` files, keys, certificates, passwords stay out of Git.

### Approval Gate

```
Implement Module N
       │
       ▼
Run tests → All pass
       │
       ▼
Present summary to developer
       │
       ▼
Developer reviews and approves
       │
       ▼
Begin Module N+1
```

---

## 2. Module Overview

| Module | Title | Depends On |
|---|---|---|
| **0** | Project Initialization | — |
| **1** | Authentication and RBAC | Module 0 |
| **2** | Student Management | Module 1 |
| **3** | Classroom and Seating Management | Module 1 |
| **4** | Android Device Management and Pairing | Module 1 |
| **5** | Examination Management | Modules 2, 3, 4 |
| **6** | Question Paper Upload and Security | Module 5 |
| **7** | Android Student Display Application | Module 4 |
| **8** | Local Paper Distribution and Exam Control | Modules 6, 7 |
| **9** | Monitoring, Reports and Audit Logs | Module 8 |
| **10** | Testing and Local Server Deployment | All previous |

---

## 3. Module Status Tracker

| Module | Status | Completion Date | Notes |
|---|---|---|---|
| **0** – Project Initialization | ✅ **Complete** | 2026-11-18 | Approved by developer |
| **1** – Authentication and RBAC | 🔜 Pending | — | Awaiting Module 0 approval |
| **2** – Student Management | 🔜 Pending | — | — |
| **3** – Classroom and Seating | 🔜 Pending | — | — |
| **4** – Android Device Management | 🔜 Pending | — | — |
| **5** – Examination Management | 🔜 Pending | — | — |
| **6** – Question Paper Upload | 🔜 Pending | — | — |
| **7** – Android Display App | 🔜 Pending | — | — |
| **8** – Distribution and Exam Control | 🔜 Pending | — | — |
| **9** – Monitoring, Reports, Audit | 🔜 Pending | — | — |
| **10** – Testing and Deployment | 🔜 Pending | — | — |

> **Update this table** each time a module is approved. Record the exact date.

---

## 4. Detailed Module Plans

---

### Module 0 – Project Initialization ✅

**Objective:** Set up the repository structure, documentation foundation, and Git version control.

**Deliverables:**
- [x] Folder structure: `backend/`, `web/`, `android/`, `database/`, `docs/`
- [x] `.gitignore` covering secrets, builds, IDE files, virtual environments
- [x] `README.md` explaining the project
- [x] `docs/SRS.md` — Software Requirements Specification
- [x] `docs/ARCHITECTURE.md` — System and security architecture
- [x] `docs/DATABASE_DESIGN.md` — 15-table schema design
- [x] `docs/API_DESIGN.md` — REST and WebSocket API specification
- [x] `docs/DEVELOPMENT_PLAN.md` — This file
- [x] Git repository initialized with initial commit
- [x] Environment scan report provided to developer

**Tests:** N/A (documentation-only module)

---

### Module 1 – Authentication and RBAC 🔜

**Objective:** Build the foundation of the backend — database setup, User model, login/logout, JWT, RBAC middleware. Build the login UI for the web portal.

**Backend Deliverables:**
- [ ] Python 3.12 virtual environment configured (`backend/.venv`)
- [ ] `requirements.txt` with: FastAPI, uvicorn, SQLAlchemy, Alembic, PyMySQL, python-jose, passlib[bcrypt], pydantic-settings, python-multipart
- [ ] Database connection configured via `.env`
- [ ] Alembic initialized; first migration creates `users`, `refresh_tokens`, `system_config`
- [ ] `users` table seeded with one bootstrap Admin account
- [ ] `POST /api/v1/auth/login` — bcrypt verify, return JWT
- [ ] `POST /api/v1/auth/refresh` — token rotation
- [ ] `POST /api/v1/auth/logout` — revoke refresh token
- [ ] `GET /api/v1/users/me` — current user profile
- [ ] RBAC dependency injector: `require_role(["admin"])`, `require_role(["admin", "invigilator"])`
- [ ] Account lockout after 5 failed attempts
- [ ] Audit logging middleware wired (logs every request)

**Web Portal Deliverables:**
- [ ] React + TypeScript + Vite project scaffolded in `web/`
- [ ] Login page with form validation
- [ ] JWT token storage (access in memory, refresh in HttpOnly cookie)
- [ ] Axios interceptor for automatic token refresh
- [ ] Protected route wrapper (redirects to login if unauthenticated)
- [ ] Basic dashboard shell with role-based navigation

**Tests:**
- [ ] `test_auth.py`: login success, login failure, locked account, token refresh, token revocation
- [ ] `test_rbac.py`: admin-only endpoint rejects invigilator, invigilator endpoint rejects unauthenticated

**Definition of Done:** All tests pass. Admin can log in via browser. Invalid credentials return 401. Invigilator cannot reach admin-only endpoints.

---

### Module 2 – Student Management 🔜

**Objective:** Admin can create, list, update, and deactivate students. CSV bulk import supported.

**Backend Deliverables:**
- [ ] Alembic migration adds `students` table
- [ ] `POST /api/v1/students`
- [ ] `GET /api/v1/students` (paginated, searchable)
- [ ] `GET /api/v1/students/{uuid}`
- [ ] `PATCH /api/v1/students/{uuid}`
- [ ] `DELETE /api/v1/students/{uuid}` (soft delete)
- [ ] `POST /api/v1/students/import` (CSV upload, row-level error reporting)

**Web Portal Deliverables:**
- [ ] Students list page with search/filter
- [ ] Add/edit student form
- [ ] CSV import interface with error report display
- [ ] Soft-delete with confirmation dialog

**Tests:**
- [ ] CRUD operations
- [ ] Duplicate `student_code` rejected
- [ ] CSV import with mixed valid/invalid rows
- [ ] Soft-deleted students not returned in active list

---

### Module 3 – Classroom and Seating Management 🔜

**Objective:** Admin defines classrooms and seats. Seat capacity is enforced.

**Backend Deliverables:**
- [ ] Alembic migration adds `classrooms`, `seats` tables
- [ ] Full CRUD for classrooms and seats
- [ ] Prevent classroom deletion if seats exist

**Web Portal Deliverables:**
- [ ] Classroom management page with seat grid visualization
- [ ] Add/remove seats from a classroom

**Tests:**
- [ ] Duplicate seat label in same classroom rejected
- [ ] Classroom with seats cannot be deleted

---

### Module 4 – Android Device Management and Pairing 🔜

**Objective:** Android tablets can self-register. Admin approves, enables, and pairs them to exam seats.

**Backend Deliverables:**
- [ ] Alembic migration adds `devices` table
- [ ] `POST /api/v1/devices/register` (unauthenticated, creates pending device)
- [ ] `GET /api/v1/devices` (admin)
- [ ] `PATCH /api/v1/devices/{uuid}/approve`
- [ ] `PATCH /api/v1/devices/{uuid}/disable`
- [ ] `GET /api/v1/devices/{uuid}/status` (device-authenticated)
- [ ] Device token hashing (bcrypt) on registration

**Android Deliverables (Pairing Flow only):**
- [ ] Android project created in `android/` using Kotlin + Jetpack Compose
- [ ] Pairing screen: enter server IP, generate and store token in Keystore, POST to register
- [ ] Polling loop on `GET /devices/{uuid}/status` until approved
- [ ] Waiting for assignment screen

**Tests:**
- [ ] Registration creates pending device
- [ ] Duplicate `android_id` rejected
- [ ] Token hash stored (not plaintext)
- [ ] Only approved devices can authenticate

---

### Module 5 – Examination Management 🔜

**Objective:** Admin creates and configures examinations, assigns seats, devices, invigilators.

**Backend Deliverables:**
- [ ] Alembic migration adds `examinations`, `exam_seat_assignments` tables
- [ ] Full exam CRUD
- [ ] Status transition enforcement (draft → scheduled → active → ended)
- [ ] Bulk seat assignment endpoint
- [ ] Prevent starting without paper assigned

**Web Portal Deliverables:**
- [ ] Examination creation form
- [ ] Seat assignment matrix (student ↔ seat ↔ device)
- [ ] Status badge and lifecycle controls

**Tests:**
- [ ] Duplicate student/seat/device in same exam rejected
- [ ] Cannot transition to `active` without paper assigned
- [ ] Cannot assign more students than classroom capacity

---

### Module 6 – Question Paper Upload and Security 🔜

**Objective:** Admin uploads PDF. Encrypted at rest. Assigned to exam. Hash stored.

**Backend Deliverables:**
- [ ] Alembic migration adds `question_papers`, `paper_assignments` tables
- [ ] PDF upload: MIME + magic byte validation, 50 MB size limit
- [ ] AES-256-GCM encryption on upload
- [ ] Key stored in `/srv/smartexam/keys/<exam_uuid>.key` (chmod 600)
- [ ] SHA-256 hash of plaintext stored in DB
- [ ] Encrypted file stored as `<uuid>.enc`
- [ ] Plaintext PDF deleted after encryption
- [ ] `POST /api/v1/papers/{uuid}/assign/{exam_uuid}`

**Web Portal Deliverables:**
- [ ] File upload form with drag-and-drop
- [ ] Upload progress indicator
- [ ] Paper-to-exam assignment UI

**Tests:**
- [ ] Non-PDF file rejected
- [ ] SHA-256 of original verified post-encryption
- [ ] Key file not stored in database
- [ ] Plaintext file not on disk after upload
- [ ] Encrypted file is different from plaintext

---

### Module 7 – Android Student Display Application 🔜

**Objective:** Tablet displays assigned exam paper. Zoom, navigation. Read-only. WebSocket state machine.

**Android Deliverables:**
- [ ] WebSocket client using OkHttp
- [ ] Exam state machine: `WAITING → DOWNLOADING → DISPLAYING → ENDED`
- [ ] Pre-download encrypted paper when exam is `scheduled`
- [ ] Receive decryption key via WebSocket on `EXAM_STARTED`
- [ ] Decrypt paper in memory (never write plaintext to disk)
- [ ] SHA-256 integrity verification before display
- [ ] PDF display using `android.graphics.pdf.PdfRenderer`
- [ ] Pinch-to-zoom, page navigation
- [ ] No text selection, no annotation, no screenshot capability (best effort)
- [ ] Clear paper from memory on `EXAM_ENDED`
- [ ] "Waiting" and "Exam Ended" screens

**Tests:**
- [ ] Integrity check failure prevents display
- [ ] Exam state transitions function correctly
- [ ] Paper is cleared from memory on exam end

---

### Module 8 – Local Paper Distribution and Exam Control 🔜

**Objective:** Invigilator starts and ends exam. WebSocket push to all tablets. Monitoring dashboard.

**Backend Deliverables:**
- [ ] Alembic migration adds `paper_distribution_log`, `device_connections`, `device_exam_state`
- [ ] WebSocket hub for exam channels
- [ ] `GET /api/v1/distribution/{exam_uuid}/download` (device-auth, encrypted file)
- [ ] `GET /api/v1/distribution/{exam_uuid}/key` (device-auth, only when active)
- [ ] Broadcast `EXAM_STARTED` / `EXAM_ENDED` to all connected tablets
- [ ] Accept and store device state events
- [ ] `GET /api/v1/distribution/{exam_uuid}/monitor`

**Web Portal Deliverables:**
- [ ] Invigilator exam control panel
- [ ] "Start Exam" and "End Exam" buttons with confirmation
- [ ] Real-time device state grid (shows each tablet's status)
- [ ] WebSocket connection from portal to monitor channel

**Tests:**
- [ ] Key endpoint returns 403 when exam not active
- [ ] Broadcast reaches all connected tablets
- [ ] Offline tablets pick up paper from local cache

---

### Module 9 – Monitoring, Reports and Audit Logs 🔜

**Objective:** Admin views audit logs, exports CSV, views post-exam reports.

**Backend Deliverables:**
- [ ] `GET /api/v1/audit-logs` with filters
- [ ] `GET /api/v1/audit-logs/export` CSV
- [ ] `GET /api/v1/reports/exam/{uuid}` summary
- [ ] Confirm audit_logs table is append-only (no UPDATE/DELETE paths in codebase)

**Web Portal Deliverables:**
- [ ] Audit log viewer with date/actor/action filters
- [ ] CSV export button
- [ ] Post-exam report page

**Tests:**
- [ ] Filters return correct subsets
- [ ] CSV export is well-formed
- [ ] Audit log cannot be modified via API

---

### Module 10 – Testing and Local Server Deployment 🔜

**Objective:** Full integration testing on local Ubuntu server. Emergency fallback verified. Deployment documented.

**Deliverables:**
- [ ] Integration test suite covering full exam workflow end-to-end
- [ ] Load test: 30 simultaneous tablet connections
- [ ] Ubuntu Linux deployment guide (`docs/DEPLOYMENT.md`)
- [ ] nginx + systemd configuration files
- [ ] TLS certificate generation (self-signed for LAN)
- [ ] MySQL production configuration
- [ ] Emergency fallback procedure documented and tested
- [ ] Pre-exam checklist for invigilator and admin

**Tests:**
- [ ] All 30 tablets connect and display paper simultaneously
- [ ] Server restart during exam — tablets continue from local cache
- [ ] Wi-Fi drop for 2 minutes — tablet recovers when reconnected
- [ ] Emergency fallback procedure executed successfully

---

## 5. Definition of Done

A module is **done** when ALL of the following are true:

1. ✅ All planned deliverables are implemented.
2. ✅ Automated tests are written and all pass.
3. ✅ No regressions in previously passing tests.
4. ✅ Code follows style conventions (PEP 8 / ESLint / ktlint).
5. ✅ `.env.example` updated if new environment variables are introduced.
6. ✅ Relevant documentation (`ARCHITECTURE.md`, `DATABASE_DESIGN.md`, `API_DESIGN.md`) updated for any changes.
7. ✅ Audit log captures the module's significant events.
8. ✅ No secrets or keys are committed.
9. ✅ Developer has reviewed and explicitly approved.

---

## 6. Prerequisites Checklist

| Prerequisite | Status | Action |
|---|---|---|
| Git 2.x | ✅ Installed (2.50.1) | None |
| Python 3.12 | ❌ Not installed | Install from python.org before Module 1 |
| Node.js 22.x | ✅ Installed (22.17.1) | None |
| npm 10.x | ✅ Installed (10.9.2) | None |
| MySQL 8.4 | ✅ Installed | Add `mysql` to PATH before Module 1 |
| Android Studio | ✅ Installed | None |
| Android SDK | ✅ Detected | None |
| GitHub account | ⏳ Not pushed yet | Create repo and push when approved |

---

## 7. Risk Register

| ID | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Python 3.12 not installed before Module 1 | Medium | High | Install before Module 1 starts |
| R-02 | Wi-Fi drops during exam | Medium | High | Tablets buffer encrypted paper locally |
| R-03 | Server power failure during exam | Low | High | UPS on server + emergency fallback with printed papers |
| R-04 | Tablet rooted/compromised | Low | Medium | Cannot guarantee full tamper-proof; rely on physical supervision |
| R-05 | MySQL password in code | Low | Critical | `.env` for all credentials; `.gitignore` enforced |
| R-06 | Paper leaked before exam | Low | Critical | Key only sent after Invigilator confirms start; download ≠ decrypt |
| R-07 | Only 1 Admin trained | Medium | Medium | Train at least 2 staff before pilot |
| R-08 | Tablet battery dies during exam | Medium | Medium | Ensure tablets are fully charged before each session |

---

## 8. Technology Version Reference

| Technology | Version Locked | Notes |
|---|---|---|
| Python | 3.12.x | Specified. 3.13 may work but not tested. |
| FastAPI | Latest 0.11x | Pin in `requirements.txt` |
| SQLAlchemy | 2.x (async) | Major version matters |
| Alembic | Latest | Pin in `requirements.txt` |
| MySQL | 8.4 | Use InnoDB engine only |
| Node.js | 22.x LTS | As installed |
| React | 18.x | Vite-based project |
| TypeScript | 5.x | Strict mode enabled |
| Kotlin | As bundled with Android Studio | — |
| Jetpack Compose | As bundled with Android Studio | — |
| Android SDK | API 29 minimum (Android 10) | Covers >90% of modern tablets |

> **Do not change these versions without discussion and approval.**
