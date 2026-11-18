# SmartExam

**Local Network-Based Digital Examination Question Paper Distribution System**

SmartExam is a paperless examination delivery system designed to run entirely on a school or institution's local Wi-Fi network — with no internet connection required during examinations. Each student reads their question paper on a dedicated Android tablet while writing answers on physical answer sheets.

---

## 📋 Project Overview

| Item | Detail |
|---|---|
| **Purpose** | Reduce paper consumption during written examinations |
| **Deployment** | Local institution network (no internet required) |
| **Pilot capacity** | 30 student tablets |
| **Server OS** | Ubuntu Linux (local server) |
| **Development OS** | Windows 11 (this machine) |

---

## 🏗️ Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                   Local Wi-Fi Network                   │
│                                                         │
│  ┌────────────────┐        ┌──────────────────────────┐ │
│  │  Admin/        │  HTTPS │  SmartExam Server        │ │
│  │  Invigilator   │◄──────►│  Python 3.12 + FastAPI   │ │
│  │  Web Portal    │        │  MySQL 8.4               │ │
│  │  React + TS    │        │  Ubuntu Linux            │ │
│  └────────────────┘        └──────────┬───────────────┘ │
│                                       │                  │
│                            HTTPS + WebSocket            │
│                                       │                  │
│         ┌─────────┬──────────┬────────┘                 │
│         ▼         ▼          ▼                           │
│    [Tablet 1] [Tablet 2] ... [Tablet 30]                │
│    Kotlin + Jetpack Compose                             │
│    Android PDF Renderer                                 │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Technical Stack

| Layer | Technology |
|---|---|
| **Backend** | Python 3.12, FastAPI |
| **Database** | MySQL 8.4 |
| **Admin & Invigilator Portal** | React, TypeScript |
| **Android Application** | Kotlin, Jetpack Compose, MVVM |
| **API Communication** | REST (HTTPS) |
| **Real-Time Control** | WebSocket |
| **PDF Rendering** | Android PdfRenderer API |
| **Local Server** | Ubuntu Linux |

---

## 📁 Repository Structure

```
SmartExam/
│
├── backend/              # Python 3.12 + FastAPI server
│   ├── app/              # Application package (created in Module 1)
│   └── tests/            # Automated tests
│
├── web/                  # React + TypeScript portal
│   └── (scaffolded in Module 1)
│
├── android/              # Kotlin + Jetpack Compose student app
│   └── (created in Module 7)
│
├── database/             # Versioned SQL migration scripts
│   └── migrations/
│
├── docs/                 # Project documentation
│   ├── SRS.md            # Software Requirements Specification
│   ├── ARCHITECTURE.md   # Architecture decisions and diagrams
│   ├── DATABASE_DESIGN.md# Schema design and ER relationships
│   ├── API_DESIGN.md     # REST + WebSocket API specification
│   └── DEVELOPMENT_PLAN.md # Module roadmap and status tracker
│
├── .gitignore            # Excludes secrets, builds, IDE files
└── README.md             # This file
```

---

## 🚀 MVP Feature Summary

### Admin Portal
- Manage students, classrooms, seats, devices, examinations
- Upload question papers (PDF)
- View audit logs and reports

### Invigilator Portal
- Start and end examinations
- Monitor connected devices in real time
- Trigger emergency fallback if required

### Android Tablet App
- Register and pair with server over local network
- Receive assigned question paper after authorized release
- Display PDF with zoom and navigation
- Read-only: students cannot type, edit, or submit digitally

### Security
- TLS encrypted communication on the local network
- AES-256-GCM encrypted question paper storage
- Papers released only at authorized examination start
- Audit log of all critical operations
- Android Keystore for device credentials

---

## ⚙️ Prerequisites

| Tool | Version |
|---|---|
| Python | 3.12.x |
| Node.js | 22.x LTS |
| MySQL | 8.4 |
| Android Studio | Latest stable |
| Git | 2.x |

---

## 🔧 Setup Instructions

> Full setup is covered module by module. See [`docs/DEVELOPMENT_PLAN.md`](docs/DEVELOPMENT_PLAN.md).

### Backend (Module 1+)
```bash
cd backend
py -3.12 -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
```

### Web Portal (Module 1+)
```bash
cd web
npm install
npm run dev
```

### Android App (Module 7+)
Open `android/` in Android Studio and sync Gradle.

---

## 🔐 Security Notice

- **Never commit** `.env` files, certificates, private keys, or keystores.
- Question papers are encrypted at rest and distributed only on authorized release.
- The system is designed for local network use only. Do not expose the server to the internet without additional hardening.

---

## 📄 Documentation

| Document | Purpose |
|---|---|
| [`docs/SRS.md`](docs/SRS.md) | Complete functional and non-functional requirements |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System and security architecture |
| [`docs/DATABASE_DESIGN.md`](docs/DATABASE_DESIGN.md) | Database schema with 15 core tables |
| [`docs/API_DESIGN.md`](docs/API_DESIGN.md) | REST and WebSocket API endpoints |
| [`docs/DEVELOPMENT_PLAN.md`](docs/DEVELOPMENT_PLAN.md) | Module-by-module plan with status tracking |

---

## 📌 Development Modules

| Module | Title | Status |
|---|---|---|
| 0 | Project Initialization | ✅ Complete |
| 1 | Authentication & RBAC | 🔜 Pending |
| 2 | Student Management | 🔜 Pending |
| 3 | Classroom & Seating Management | 🔜 Pending |
| 4 | Android Device Management & Pairing | 🔜 Pending |
| 5 | Examination Management | 🔜 Pending |
| 6 | Question Paper Upload & Security | 🔜 Pending |
| 7 | Android Student Display Application | 🔜 Pending |
| 8 | Local Paper Distribution & Exam Control | 🔜 Pending |
| 9 | Monitoring, Reports & Audit Logs | 🔜 Pending |
| 10 | Testing & Local Server Deployment | 🔜 Pending |

---

## 🤝 Contributing

This is a guided development project. Each module requires explicit review and approval before the next module begins. See [`docs/DEVELOPMENT_PLAN.md`](docs/DEVELOPMENT_PLAN.md) for the approval workflow.

---

## 📝 License

Proprietary — Institution use only. Not for public distribution.
