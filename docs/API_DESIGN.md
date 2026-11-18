# SmartExam – API Design

**Document Version:** 1.0  
**Date:** 2026-11-18  
**Status:** Approved – Module 0  
**Author:** SmartExam Development Team

---

## Table of Contents

1. [API Overview](#1-api-overview)
2. [Base URL and Versioning](#2-base-url-and-versioning)
3. [Authentication and Authorization](#3-authentication-and-authorization)
4. [Common Conventions](#4-common-conventions)
5. [REST API Endpoints](#5-rest-api-endpoints)
   - [Auth](#51-auth-endpoints)
   - [Users](#52-user-management-endpoints)
   - [Students](#53-student-endpoints)
   - [Classrooms and Seats](#54-classroom-and-seat-endpoints)
   - [Devices](#55-device-endpoints)
   - [Examinations](#56-examination-endpoints)
   - [Question Papers](#57-question-paper-endpoints)
   - [Paper Distribution](#58-paper-distribution-endpoints)
   - [Reports and Audit Logs](#59-reports-and-audit-log-endpoints)
6. [WebSocket API](#6-websocket-api)
7. [Error Response Format](#7-error-response-format)
8. [HTTP Status Code Reference](#8-http-status-code-reference)
9. [Rate Limiting](#9-rate-limiting)
10. [API Security](#10-api-security)

---

## 1. API Overview

SmartExam exposes a **REST API** for all administrative and invigilator portal operations, and a **WebSocket API** for real-time examination control communications between the server and Android tablets (and the invigilator portal).

| Concern | Technology |
|---|---|
| Protocol | HTTPS (TLS) for REST; WSS for WebSocket |
| Data Format | JSON (UTF-8) |
| Authentication | JWT Bearer tokens |
| API Versioning | URI-based (`/api/v1/`) |
| Documentation | Auto-generated via FastAPI/OpenAPI at `/api/docs` (dev only) |

---

## 2. Base URL and Versioning

```
Production:  https://192.168.1.1/api/v1/
Development: http://localhost:8000/api/v1/
```

All REST endpoints are prefixed with `/api/v1/`. When a breaking change is required in the future, a `/api/v2/` prefix will be introduced.

---

## 3. Authentication and Authorization

### 3.1 JWT Token Usage

All protected endpoints require an `Authorization` header:

```
Authorization: Bearer <access_token>
```

### 3.2 Token Lifecycle

```
POST /api/v1/auth/login
  → Returns: { access_token (60 min), refresh_token (7 days) }

POST /api/v1/auth/refresh
  → Accepts: { refresh_token }
  → Returns: { access_token (new), refresh_token (rotated) }

POST /api/v1/auth/logout
  → Revokes the refresh token
```

### 3.3 Role-Based Access

| Endpoint Group | Admin | Invigilator |
|---|---|---|
| Auth (login/logout/refresh) | ✅ | ✅ |
| User Management | ✅ | ❌ |
| Student Management | ✅ | ❌ |
| Classroom/Seat Management | ✅ | ❌ |
| Device Management | ✅ | ❌ |
| Examination Management | ✅ | ❌ (read-only for assigned) |
| Question Paper Upload | ✅ | ❌ |
| Exam Control (start/end) | ✅ | ✅ (assigned exams only) |
| Monitoring Dashboard | ✅ | ✅ (assigned exams only) |
| Audit Logs | ✅ | ❌ |

### 3.4 Device Authentication

Android tablets authenticate using a device token (not a human JWT). The device token is sent in the `X-Device-Token` header for device-specific endpoints.

```
X-Device-Token: <device_token>
```

---

## 4. Common Conventions

### Request Headers

```
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>       (all protected endpoints)
X-Device-Token: <token>             (device endpoints only)
```

### Pagination

All list endpoints support pagination:

```
GET /api/v1/students?page=1&page_size=20&search=john
```

Response envelope:

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total_items": 85,
    "total_pages": 5
  }
}
```

### Timestamps

All timestamps are in **ISO 8601 UTC format**: `2026-11-18T03:30:00Z`

### UUIDs

All public-facing IDs use UUID strings (`uuid` field). Internal auto-increment IDs are not exposed in responses.

---

## 5. REST API Endpoints

---

### 5.1 Auth Endpoints

#### `POST /api/v1/auth/login`

Authenticate a user (Admin or Invigilator).

**Request:**
```json
{
  "email": "admin@school.edu",
  "password": "SecurePassword123"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "token_type": "bearer",
  "expires_in": 3600,
  "user": {
    "uuid": "123e4567-e89b-12d3-a456-426614174000",
    "full_name": "Admin User",
    "email": "admin@school.edu",
    "role": "admin"
  }
}
```

**Errors:** `401 Unauthorized` (invalid credentials), `423 Locked` (account locked after 5 failures)

---

#### `POST /api/v1/auth/refresh`

Rotate access and refresh tokens.

**Request:**
```json
{ "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..." }
```

**Response `200 OK`:**
```json
{
  "access_token": "new_access_token...",
  "refresh_token": "new_refresh_token...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

---

#### `POST /api/v1/auth/logout`

Revoke refresh token and end session.

**Request:**
```json
{ "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..." }
```

**Response `204 No Content`**

---

### 5.2 User Management Endpoints

> **Admin only**

#### `POST /api/v1/users`
Create an Admin or Invigilator account.

**Request:**
```json
{
  "full_name": "Jane Invigilator",
  "email": "jane@school.edu",
  "password": "TempPassword!23",
  "role": "invigilator"
}
```

**Response `201 Created`:** User object (without password_hash)

---

#### `GET /api/v1/users`
List all users. Supports `?role=invigilator&search=jane`.

**Response `200 OK`:** Paginated list of user objects.

---

#### `GET /api/v1/users/{user_uuid}`
Get single user details.

---

#### `PATCH /api/v1/users/{user_uuid}`
Update user fields (name, email, role, is_active).

---

#### `POST /api/v1/users/{user_uuid}/reset-password`
Admin resets another user's password.

**Request:**
```json
{ "new_password": "NewSecurePassword!45" }
```

---

### 5.3 Student Endpoints

> **Admin only**

#### `POST /api/v1/students`
Create a student.

**Request:**
```json
{
  "student_code": "STU2026001",
  "full_name": "Rahul Kumar",
  "class_name": "10-A",
  "section": "A",
  "roll_number": "101"
}
```

---

#### `GET /api/v1/students`
List students. Supports `?class_name=10-A&search=rahul&is_active=true`.

---

#### `GET /api/v1/students/{student_uuid}`
Get single student.

---

#### `PATCH /api/v1/students/{student_uuid}`
Update student details.

---

#### `DELETE /api/v1/students/{student_uuid}`
Soft-delete (sets `is_active = false`). Returns `204 No Content`.

---

#### `POST /api/v1/students/import`
Bulk import from CSV file.

**Request:** `multipart/form-data` with field `file` (CSV).

**Response `200 OK`:**
```json
{
  "imported": 28,
  "skipped": 2,
  "errors": [
    { "row": 5, "reason": "Duplicate student_code" }
  ]
}
```

---

### 5.4 Classroom and Seat Endpoints

> **Admin only**

#### `POST /api/v1/classrooms`
Create a classroom.

**Request:**
```json
{
  "name": "Lab A",
  "capacity": 30,
  "location": "Block 2, Ground Floor"
}
```

---

#### `GET /api/v1/classrooms`
List classrooms.

---

#### `GET /api/v1/classrooms/{classroom_uuid}`
Get classroom with its seats.

---

#### `POST /api/v1/classrooms/{classroom_uuid}/seats`
Add seats to a classroom.

**Request:**
```json
{
  "seats": ["A1", "A2", "A3", "B1", "B2", "B3"]
}
```

---

#### `DELETE /api/v1/classrooms/{classroom_uuid}/seats/{seat_uuid}`
Deactivate a seat (soft delete).

---

### 5.5 Device Endpoints

#### `POST /api/v1/devices/register`
Android tablet self-registers. No auth required, but pending Admin approval.

**Request (from tablet):**
```json
{
  "device_name": "Tablet-01",
  "android_id": "a1b2c3d4e5f6a7b8",
  "device_token": "randomly_generated_secure_token_stored_in_keystore",
  "model": "Samsung Galaxy Tab A8",
  "android_version": "13"
}
```

**Response `201 Created`:**
```json
{
  "device_uuid": "uuid-of-new-device",
  "status": "pending",
  "message": "Registration submitted. Awaiting Admin approval."
}
```

---

#### `GET /api/v1/devices` *(Admin)*
List all devices with status.

---

#### `PATCH /api/v1/devices/{device_uuid}/approve` *(Admin)*
Approve a pending device registration.

---

#### `PATCH /api/v1/devices/{device_uuid}/disable` *(Admin)*
Disable an active device.

---

#### `GET /api/v1/devices/{device_uuid}/status` *(Tablet)*
Tablet polls this to check if it has been approved and get exam assignment.

**Response `200 OK`:**
```json
{
  "status": "active",
  "assigned_exam": {
    "exam_uuid": "...",
    "exam_name": "Mathematics Final Exam 2026",
    "scheduled_date": "2026-11-25",
    "start_time": "09:00",
    "student_name": "Rahul Kumar",
    "seat_label": "A1"
  }
}
```

---

### 5.6 Examination Endpoints

#### `POST /api/v1/examinations` *(Admin)*
Create an examination.

**Request:**
```json
{
  "name": "Mathematics Final Exam 2026",
  "subject": "Mathematics",
  "classroom_uuid": "...",
  "invigilator_uuid": "...",
  "scheduled_date": "2026-11-25",
  "start_time": "09:00",
  "duration_minutes": 180
}
```

---

#### `GET /api/v1/examinations` *(Admin)*
List exams. Supports `?status=scheduled&date=2026-11-25`.

---

#### `GET /api/v1/examinations/{exam_uuid}` *(Admin, Invigilator)*
Get exam details with seat assignments and paper assignment.

---

#### `POST /api/v1/examinations/{exam_uuid}/assignments` *(Admin)*
Bulk-assign students to seats and devices.

**Request:**
```json
{
  "assignments": [
    {
      "student_uuid": "...",
      "seat_uuid": "...",
      "device_uuid": "..."
    }
  ]
}
```

---

#### `PATCH /api/v1/examinations/{exam_uuid}/status` *(Admin, Invigilator)*
Transition exam status.

**Request:**
```json
{ "status": "active" }
```

Valid transitions: `draft→scheduled`, `scheduled→active`, `active→ended`, `any→cancelled`

---

### 5.7 Question Paper Endpoints

> **Admin only**

#### `POST /api/v1/papers`
Upload a question paper PDF. Encrypted immediately on receipt.

**Request:** `multipart/form-data`
- `file`: PDF file
- `title`: string
- `subject`: string

**Response `201 Created`:**
```json
{
  "paper_uuid": "...",
  "title": "Mathematics Final Exam Paper",
  "subject": "Mathematics",
  "page_count": 4,
  "file_size_bytes": 2048576,
  "sha256_hash": "a3f1...",
  "upload_status": "ready"
}
```

---

#### `GET /api/v1/papers`
List uploaded papers.

---

#### `POST /api/v1/papers/{paper_uuid}/assign/{exam_uuid}`
Assign a paper to an examination.

---

#### `DELETE /api/v1/papers/{paper_uuid}`
Soft-delete a paper (marks inactive). Cannot delete if assigned to a scheduled/active exam.

---

### 5.8 Paper Distribution Endpoints

#### `GET /api/v1/distribution/{exam_uuid}/download` *(Tablet — authenticated device)*
Download the encrypted question paper file.

- Only available to devices assigned to this exam.
- Only available when exam status is `scheduled` or `active`.
- Returns the raw encrypted binary file (`.enc`).
- Decryption key is **not** included here.

**Response `200 OK`:** Binary stream of encrypted PDF.

---

#### `GET /api/v1/distribution/{exam_uuid}/key` *(Tablet — authenticated device)*
Retrieve the decryption key for the assigned paper.

- Only available when exam status is `active` (exam has been started by Invigilator).
- Only available to the specific device assigned to this exam.

**Response `200 OK`:**
```json
{
  "key_b64": "base64_encoded_aes256_key",
  "iv_b64": "base64_encoded_iv",
  "auth_tag_b64": "base64_encoded_auth_tag"
}
```

---

#### `GET /api/v1/distribution/{exam_uuid}/monitor` *(Admin, Invigilator)*
Get real-time state of all devices in this exam.

**Response `200 OK`:**
```json
{
  "exam_uuid": "...",
  "exam_status": "active",
  "devices": [
    {
      "device_name": "Tablet-01",
      "student_name": "Rahul Kumar",
      "seat_label": "A1",
      "state": "paper_displayed",
      "last_seen": "2026-11-25T09:03:12Z"
    }
  ],
  "summary": {
    "total": 30,
    "connected": 29,
    "paper_displayed": 28,
    "offline": 1
  }
}
```

---

### 5.9 Reports and Audit Log Endpoints

> **Admin only**

#### `GET /api/v1/audit-logs`
List audit logs with filters.

Query params: `?action=exam.started&actor_type=user&from=2026-11-01&to=2026-11-30&page=1`

---

#### `GET /api/v1/audit-logs/export`
Export audit logs as CSV download.

**Response:** `text/csv` file attachment.

---

#### `GET /api/v1/reports/exam/{exam_uuid}`
Get a post-exam summary report (device states, distribution events, timeline).

---

## 6. WebSocket API

### 6.1 Connection

```
wss://<server>/ws/exam/{exam_uuid}?device_token=<token>
```

Authentication: Device token in query param (validated on connect). After handshake, token is validated and connection is registered.

### 6.2 Server → Tablet Events

```json
// Sent on connection
{ "event": "EXAM_WAITING", "exam_id": "uuid", "student_name": "Rahul Kumar", "seat": "A1" }

// Sent when paper is ready to pre-download
{ "event": "PAPER_READY", "download_url": "/api/v1/distribution/uuid/download", "file_size": 2048576 }

// Sent when Invigilator starts exam
{ "event": "EXAM_STARTED", "exam_duration_minutes": 180, "server_time": "2026-11-25T09:00:00Z" }

// Sent when time is extended
{ "event": "EXAM_EXTENDED", "new_end_time": "2026-11-25T12:30:00Z" }

// Sent when Invigilator ends exam
{ "event": "EXAM_ENDED", "timestamp": "2026-11-25T12:00:00Z" }

// Heartbeat (every 30s)
{ "event": "PING" }
```

### 6.3 Tablet → Server Events

```json
{ "event": "PAPER_DOWNLOADED", "device_id": "uuid", "checksum": "sha256hex" }
{ "event": "PAPER_DISPLAYED", "device_id": "uuid", "timestamp": "2026-11-25T09:00:05Z" }
{ "event": "PONG" }
```

### 6.4 Invigilator Portal WebSocket

```
wss://<server>/ws/monitor/{exam_uuid}
Authorization: Bearer <jwt>
```

The portal subscribes to real-time device state updates:

```json
// Server broadcasts to all invigilator portal connections
{
  "event": "DEVICE_STATE_CHANGED",
  "device_name": "Tablet-05",
  "student_name": "Priya Sharma",
  "state": "paper_displayed",
  "timestamp": "2026-11-25T09:00:08Z"
}
```

---

## 7. Error Response Format

All errors return a consistent JSON structure:

```json
{
  "error": {
    "code": "EXAM_NOT_ACTIVE",
    "message": "The examination is not currently active. Papers can only be accessed during an active exam.",
    "detail": null
  }
}
```

**Validation errors (422):**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "detail": [
      { "field": "email", "message": "Invalid email format." },
      { "field": "password", "message": "Password must be at least 8 characters." }
    ]
  }
}
```

---

## 8. HTTP Status Code Reference

| Code | Meaning | Usage |
|---|---|---|
| `200` | OK | Successful GET, PATCH |
| `201` | Created | Successful POST that creates a resource |
| `204` | No Content | Successful DELETE, logout |
| `400` | Bad Request | Invalid request body structure |
| `401` | Unauthorized | Missing or invalid token |
| `403` | Forbidden | Authenticated but insufficient role |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Duplicate resource (e.g. same email) |
| `422` | Unprocessable Entity | Validation failure |
| `423` | Locked | Account locked after too many failures |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server error |

---

## 9. Rate Limiting

| Endpoint | Limit | Window |
|---|---|---|
| `POST /auth/login` | 10 requests | per minute per IP |
| `POST /devices/register` | 5 requests | per minute per IP |
| All other endpoints | 120 requests | per minute per token |

Rate limit headers returned on every response:
```
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 115
X-RateLimit-Reset: 1700000060
```

---

## 10. API Security

| Concern | Implementation |
|---|---|
| **Transport** | All endpoints HTTPS only. HTTP returns 301 redirect. |
| **Auth** | JWT Bearer (access token 60 min). No API keys stored in code. |
| **RBAC** | Role checked server-side per endpoint. Not just on client. |
| **Device Auth** | Separate `X-Device-Token` header validated against bcrypt hash. |
| **Input validation** | Pydantic schemas validate all request bodies before handler. |
| **SQL injection** | SQLAlchemy ORM with parameterized queries. No raw SQL from user input. |
| **File upload** | MIME type validation + magic byte check + size limit (50 MB max). |
| **OpenAPI docs** | Disabled in production (`/api/docs` returns 404 in prod). |
| **CORS** | Restricted to local network IP ranges in production. |
| **Paper endpoint** | Key endpoint `GET /distribution/{exam_uuid}/key` checks: (1) device is assigned, (2) exam is active, (3) device is registered and approved. All three must pass. |
