# SmartExam – System Architecture

**Document Version:** 1.0  
**Date:** 2026-11-18  
**Status:** Approved – Module 0  
**Author:** SmartExam Development Team

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Component Architecture](#2-component-architecture)
3. [Backend Architecture (FastAPI)](#3-backend-architecture-fastapi)
4. [Frontend Architecture (React)](#4-frontend-architecture-react)
5. [Android Architecture (MVVM)](#5-android-architecture-mvvm)
6. [Network Architecture](#6-network-architecture)
7. [Security Architecture](#7-security-architecture)
8. [Question Paper Lifecycle](#8-question-paper-lifecycle)
9. [WebSocket Communication](#9-websocket-communication)
10. [Data Flow Diagrams](#10-data-flow-diagrams)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Architecture Decision Log](#12-architecture-decision-log)

---

## 1. Architecture Overview

SmartExam uses a **three-tier client-server architecture** operating entirely on a private local area network.

```
┌─────────────────────────────────────────────────────────────────┐
│                     LOCAL WI-FI NETWORK (LAN)                   │
│                   No internet access required                    │
│                                                                  │
│  ┌──────────────────┐    HTTPS     ┌──────────────────────────┐ │
│  │   Web Portal     │◄────────────►│     FastAPI Backend      │ │
│  │  (React + TS)    │              │     Python 3.12          │ │
│  │  Admin /         │   WebSocket  │                          │ │
│  │  Invigilator     │◄────────────►│     MySQL 8.4            │ │
│  └──────────────────┘              │                          │ │
│                                    │  Encrypted PDF Store     │ │
│  ┌──────────────────┐    HTTPS     │  (filesystem)            │ │
│  │  Android Tablet  │◄────────────►│                          │ │
│  │  (Kotlin/Compose)│   WebSocket  │  Key Store               │ │
│  │  × 30 devices    │◄────────────►│  (separate from DB)      │ │
│  └──────────────────┘              └──────────────────────────┘ │
│                                           │                      │
│                                    Ubuntu Linux Server           │
└─────────────────────────────────────────────────────────────────┘
```

### Core Design Principles

1. **Network isolation**: The system operates fully offline. No external API calls during exams.
2. **Defense in depth**: Security applied at transport (TLS), storage (AES-256-GCM), and authorization (RBAC) layers.
3. **Fail-safe display**: Tablets buffer encrypted papers locally so a Wi-Fi interruption does not disrupt an active exam.
4. **Modularity**: Each layer (backend, web, Android) is independently deployable and testable.
5. **Audit by default**: All significant operations are logged before they complete.

---

## 2. Component Architecture

### Component Responsibilities

| Component | Technology | Responsibility |
|---|---|---|
| **FastAPI Backend** | Python 3.12, FastAPI, SQLAlchemy | Business logic, auth, REST API, WebSocket hub, file encryption |
| **MySQL Database** | MySQL 8.4 | Persistent storage for all application data |
| **Encrypted File Store** | Server filesystem | Stores AES-256-GCM encrypted PDF files |
| **Key Store** | Server filesystem (restricted path) | Stores encryption keys, isolated from main DB |
| **React Web Portal** | React 18, TypeScript, Vite | Admin and Invigilator browser interface |
| **Android App** | Kotlin, Jetpack Compose | Student tablet — PDF display, WebSocket client |

---

## 3. Backend Architecture (FastAPI)

### Directory Structure (Module 1 onwards)

```
backend/
├── app/
│   ├── main.py                  # FastAPI app entry point
│   ├── config.py                # Settings loaded from environment
│   ├── database.py              # SQLAlchemy engine and session factory
│   │
│   ├── api/                     # Route handlers (controllers)
│   │   ├── v1/
│   │   │   ├── auth.py          # Login, logout, token refresh
│   │   │   ├── users.py         # Admin/Invigilator management
│   │   │   ├── students.py      # Student CRUD and CSV import
│   │   │   ├── classrooms.py    # Classroom and seat management
│   │   │   ├── devices.py       # Android device registration and pairing
│   │   │   ├── examinations.py  # Exam lifecycle management
│   │   │   ├── papers.py        # PDF upload, encryption, assignment
│   │   │   ├── distribution.py  # Paper release and exam control
│   │   │   └── reports.py       # Audit logs and monitoring
│   │   └── ws/
│   │       └── exam_hub.py      # WebSocket connection manager
│   │
│   ├── models/                  # SQLAlchemy ORM models
│   │   ├── user.py
│   │   ├── student.py
│   │   ├── classroom.py
│   │   ├── device.py
│   │   ├── examination.py
│   │   ├── paper.py
│   │   └── audit_log.py
│   │
│   ├── schemas/                 # Pydantic request/response schemas
│   │
│   ├── services/                # Business logic layer
│   │   ├── auth_service.py
│   │   ├── paper_service.py     # Encryption, decryption, integrity
│   │   ├── distribution_service.py
│   │   └── audit_service.py
│   │
│   ├── core/                    # Cross-cutting concerns
│   │   ├── security.py          # JWT, password hashing, AES-256-GCM
│   │   ├── dependencies.py      # FastAPI dependency injectors
│   │   └── keys/                # Encryption keys (gitignored)
│   │
│   └── middleware/
│       ├── audit_middleware.py  # Logs every request/response
│       └── rate_limit.py
│
├── tests/
│   ├── test_auth.py
│   ├── test_papers.py
│   └── ...
├── alembic/                     # Database migration scripts
├── requirements.txt
└── .env.example                 # Template — no real secrets
```

### Request Lifecycle

```
HTTP Request
     │
     ▼
 TLS Termination (nginx/uvicorn)
     │
     ▼
 Rate Limit Middleware
     │
     ▼
 JWT Authentication Dependency
     │
     ▼
 RBAC Authorization Check
     │
     ▼
 Route Handler (api/v1/...)
     │
     ▼
 Service Layer (business logic)
     │
     ├──► SQLAlchemy ORM (MySQL)
     │
     └──► Filesystem (encrypted PDFs / keys)
     │
     ▼
 Audit Log Write
     │
     ▼
 Pydantic Response Serialization
     │
     ▼
 HTTP Response
```

---

## 4. Frontend Architecture (React)

### Directory Structure (Module 1 onwards)

```
web/
├── src/
│   ├── main.tsx                 # Entry point
│   ├── App.tsx                  # Router root
│   │
│   ├── pages/                   # Route-level page components
│   │   ├── Login/
│   │   ├── Dashboard/
│   │   ├── Students/
│   │   ├── Classrooms/
│   │   ├── Devices/
│   │   ├── Examinations/
│   │   ├── Papers/
│   │   ├── ExamControl/         # Invigilator live control panel
│   │   └── AuditLogs/
│   │
│   ├── components/              # Reusable UI components
│   │
│   ├── hooks/                   # Custom React hooks (useAuth, useWebSocket)
│   │
│   ├── api/                     # Axios API client, typed endpoints
│   │
│   ├── store/                   # State management (Zustand or React Context)
│   │
│   └── types/                   # TypeScript interfaces and enums
│
├── public/
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

### Authentication Flow (Web)

```
User submits login form
       │
       ▼
POST /api/v1/auth/login
       │
       ▼
Server returns { access_token, refresh_token }
       │
       ▼
Tokens stored in memory (access) + HttpOnly cookie (refresh)
       │
       ▼
Axios interceptor attaches Bearer token to all requests
       │
       ▼
On 401, interceptor uses refresh token → new access token
```

---

## 5. Android Architecture (MVVM)

### Package Structure (Module 7 onwards)

```
android/app/src/main/
└── java/com/smartexam/
    ├── MainActivity.kt          # Single Activity host
    │
    ├── ui/                      # Jetpack Compose UI layer
    │   ├── screens/
    │   │   ├── SplashScreen.kt
    │   │   ├── PairingScreen.kt
    │   │   ├── WaitingScreen.kt
    │   │   ├── ExamScreen.kt    # PDF viewer
    │   │   └── EndScreen.kt
    │   └── theme/
    │
    ├── viewmodel/               # ViewModel layer
    │   ├── PairingViewModel.kt
    │   └── ExamViewModel.kt
    │
    ├── data/                    # Data layer
    │   ├── repository/
    │   │   ├── DeviceRepository.kt
    │   │   └── ExamRepository.kt
    │   ├── remote/              # Retrofit API client
    │   └── local/               # Encrypted local cache (EncryptedFile)
    │
    ├── domain/                  # Use cases
    │   ├── DownloadPaperUseCase.kt
    │   ├── DecryptPaperUseCase.kt
    │   └── VerifyIntegrityUseCase.kt
    │
    ├── security/
    │   ├── KeystoreManager.kt   # Android Keystore operations
    │   └── PaperCryptoManager.kt
    │
    └── websocket/
        └── ExamWebSocketClient.kt
```

### MVVM Data Flow

```
UI (Compose Screen)
        │
        │ observes StateFlow / LiveData
        ▼
  ViewModel
        │
        │ calls use cases
        ▼
  Domain (Use Cases)
        │
        ├── Remote: Retrofit → FastAPI
        └── Local: EncryptedFile (Android Jetpack Security)
```

---

## 6. Network Architecture

### Network Topology

```
┌─────────────────────────────────────────────────┐
│          Dedicated Exam Wi-Fi (isolated)         │
│                                                   │
│  [Server: 192.168.1.1]                           │
│       │                                           │
│       ├── [Admin laptop]      192.168.1.10x      │
│       ├── [Invigilator laptop] 192.168.1.10x     │
│       ├── [Tablet 1..30]      192.168.1.2xx      │
│                                                   │
│  No gateway to internet during exams             │
└─────────────────────────────────────────────────┘
```

### Port Allocations

| Port | Service | Protocol |
|---|---|---|
| 443 | FastAPI (via nginx reverse proxy) | HTTPS |
| 80 | HTTP → redirect to 443 | HTTP |
| 8000 | FastAPI dev server (dev only, not prod) | HTTP |
| 3306 | MySQL (localhost only, never exposed) | TCP |

> MySQL shall **never** be accessible from the network. It listens on `127.0.0.1` only.

---

## 7. Security Architecture

### Layered Security Model

```
Layer 1 – Transport:    TLS (HTTPS + WSS) for all connections
Layer 2 – Auth:         JWT with short expiry + refresh tokens
Layer 3 – AuthZ:        RBAC enforced server-side per endpoint
Layer 4 – Storage:      AES-256-GCM encrypted PDF files
Layer 5 – Key Mgmt:     Keys stored separately from application DB
Layer 6 – Device:       Android Keystore for tablet credentials
Layer 7 – Integrity:    SHA-256 hash verified before paper display
Layer 8 – Audit:        All significant actions logged immutably
```

### Question Paper Key Management

```
Upload time:
  PDF → AES-256-GCM encrypt → Store as .enc file
  Encryption key → Stored in /secure/keys/<exam_id>.key (chmod 600)
  IV + Auth tag → Stored with encrypted file header

Release time:
  Server authenticates tablet device token
  Server verifies exam status = Active
  Server sends key over TLS to authorized tablet only
  Tablet decrypts in memory, renders PDF
  Tablet does NOT persist decrypted content to disk
```

### What We Do NOT Claim

- We do not claim the Android app is fully tamper-proof on rooted devices.
- Security is proportional to the threat model: prevention of casual unauthorized access, not nation-state attacks.
- Physical security of tablets during the exam is an operational responsibility.

---

## 8. Question Paper Lifecycle

```
[Admin] Uploads PDF
        │
        ▼
Backend validates file (PDF signature, size)
        │
        ▼
AES-256-GCM encryption
        │
        ├── Encrypted file → /srv/smartexam/papers/<uuid>.enc
        └── Key → /srv/smartexam/keys/<exam_id>.key (restricted)
        │
        ▼
SHA-256 hash stored in database (papers table)
        │
        ▼
Admin assigns paper to examination
        │
        ▼
        ◄── PAPER IS LOCKED. TABLETS CANNOT ACCESS IT YET ──►
        │
        ▼
Exam status = Scheduled
        │
        ▼
Tablets connect, pair, enter WAITING state
        │
        ▼
        ◄── Tablets may pre-download encrypted file (still cannot decrypt) ──►
        │
        ▼
Invigilator clicks "Start Exam"
        │
        ▼
Server: exam status = Active
Server: WebSocket push → EXAM_STARTED + decryption_key (over TLS)
        │
        ▼
Each Tablet:
  - Receives key
  - Decrypts paper in memory
  - Verifies SHA-256 integrity hash
  - Renders PDF via PdfRenderer API
  - Displays to student
        │
        ▼
Invigilator clicks "End Exam"
        │
        ▼
Server: exam status = Ended
Server: WebSocket push → EXAM_ENDED
        │
        ▼
Each Tablet:
  - Clears decrypted PDF from memory
  - Displays "Examination has ended" screen
  - Removes cached encrypted file
```

---

## 9. WebSocket Communication

### Event Types (Server → Tablet)

| Event | Payload | Trigger |
|---|---|---|
| `EXAM_WAITING` | `{ exam_id, student_name, seat }` | Tablet connects and pairs |
| `PAPER_READY` | `{ download_url, file_size }` | Paper is assigned and exam is scheduled |
| `EXAM_STARTED` | `{ decryption_key, exam_duration_minutes }` | Invigilator starts exam |
| `EXAM_EXTENDED` | `{ new_end_time }` | Invigilator extends exam time |
| `EXAM_ENDED` | `{ timestamp }` | Invigilator ends exam |
| `PING` | `{}` | Heartbeat (every 30 seconds) |

### Event Types (Tablet → Server)

| Event | Payload | Trigger |
|---|---|---|
| `DEVICE_CONNECTED` | `{ device_id, exam_id }` | Tablet opens WebSocket |
| `PAPER_DOWNLOADED` | `{ device_id, checksum }` | Tablet finishes downloading encrypted paper |
| `PAPER_DISPLAYED` | `{ device_id, timestamp }` | Tablet successfully displays PDF |
| `PONG` | `{}` | Response to server PING |

### WebSocket Endpoint

```
wss://<server>/ws/exam/{exam_id}?device_token=<token>
```

---

## 10. Data Flow Diagrams

### Exam Day Flow (Simplified)

```
Admin                Server               Invigilator            Tablet (×30)
  │                    │                      │                      │
  │── Upload PDF ──────►│                      │                      │
  │   (encrypted)       │                      │                      │
  │── Assign to Exam ──►│                      │                      │
  │                     │                      │                      │
  │                     │◄── Login ────────────│                      │
  │                     │─── JWT Token ────────►│                      │
  │                     │                      │                      │
  │                     │◄────────────────────────── WebSocket Connect ─│
  │                     │───────────── EXAM_WAITING ──────────────────►│
  │                     │                      │                      │
  │                     │◄─────── Start Exam ──│                      │
  │                     │── Status = Active     │                      │
  │                     │───────────────────────────── EXAM_STARTED ──►│
  │                     │                      │                      │
  │                     │                      │            Decrypt PDF│
  │                     │                      │            Display PDF│
  │                     │                      │                      │
  │                     │◄─────── End Exam ────│                      │
  │                     │── Status = Ended      │                      │
  │                     │───────────────────────────── EXAM_ENDED ────►│
  │                     │                      │            Clear PDF  │
```

---

## 11. Deployment Architecture

### Production (Ubuntu Linux Server)

```
Ubuntu Linux Server
├── nginx                    # Reverse proxy, TLS termination
│   ├── Serves React build (static files)
│   └── Proxies /api/ and /ws/ → uvicorn
│
├── uvicorn                  # ASGI server for FastAPI
│   └── 4 worker processes
│
├── MySQL 8.4                # Database (localhost only)
│
└── Filesystem
    ├── /srv/smartexam/papers/   # Encrypted PDF files
    └── /srv/smartexam/keys/     # Encryption keys (chmod 700, root only)
```

### Process Management

- **uvicorn** managed by `systemd` service for auto-restart.
- **nginx** managed by `systemd`.
- **MySQL** managed by `systemd`.

### Development (Windows 11)

```
Windows 11 (Developer Machine)
├── uvicorn backend (py -3.12)    → localhost:8000
├── Vite dev server (npm run dev) → localhost:5173
├── MySQL (local instance)        → localhost:3306
└── Android Studio Emulator       → connects to 10.0.2.2:8000
```

---

## 12. Architecture Decision Log

| ID | Decision | Reason | Date |
|---|---|---|---|
| ADR-01 | FastAPI over Django/Flask | Async-native, automatic OpenAPI docs, Pydantic validation, WebSocket built-in | 2026-11-18 |
| ADR-02 | MySQL over PostgreSQL | Institution familiarity, MySQL Workbench availability, MySQL 8.4 JSON support sufficient | 2026-11-18 |
| ADR-03 | React over Vue/Angular | Wider developer pool, mature ecosystem, TypeScript first-class | 2026-11-18 |
| ADR-04 | Jetpack Compose over XML layouts | Modern Android UI toolkit, declarative, consistent with MVVM | 2026-11-18 |
| ADR-05 | AES-256-GCM for paper encryption | Authenticated encryption prevents tampering; hardware-accelerated on modern Android | 2026-11-18 |
| ADR-06 | Keys stored separately from MySQL | If DB is compromised, encrypted papers remain unreadable | 2026-11-18 |
| ADR-07 | WebSocket for exam control | Enables real-time server-push to all tablets without polling | 2026-11-18 |
| ADR-08 | Local paper caching on tablet | Tolerates Wi-Fi interruptions during active exam | 2026-11-18 |
| ADR-09 | nginx as reverse proxy | TLS termination, static file serving, single entry point | 2026-11-18 |
| ADR-10 | Android Keystore for device credentials | Hardware-backed security, tamper-resistant on supported devices | 2026-11-18 |
