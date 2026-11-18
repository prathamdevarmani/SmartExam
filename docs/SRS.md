# SmartExam – Software Requirements Specification (SRS)

**Document Version:** 1.0  
**Date:** 2026-11-18  
**Status:** Approved – Module 0  
**Author:** SmartExam Development Team

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Overall Description](#2-overall-description)
3. [Stakeholders and User Roles](#3-stakeholders-and-user-roles)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [Security Requirements](#6-security-requirements)
7. [System Constraints](#7-system-constraints)
8. [Emergency Fallback Requirements](#8-emergency-fallback-requirements)
9. [Audit and Compliance Requirements](#9-audit-and-compliance-requirements)
10. [Assumptions and Dependencies](#10-assumptions-and-dependencies)
11. [Out of Scope](#11-out-of-scope)

---

## 1. Introduction

### 1.1 Purpose

This document defines the complete software requirements for **SmartExam**, a local network-based digital examination question paper distribution system. It is the authoritative reference for all design, development, and testing decisions.

### 1.2 Project Scope

SmartExam digitizes question paper delivery during written examinations. Students receive their question papers on Android tablets connected to a dedicated local Wi-Fi network. Students continue to write their answers on physical answer sheets. The system does not support digital answer submission.

### 1.3 Definitions and Abbreviations

| Term | Definition |
|---|---|
| **Admin** | Super-user who configures the system, manages all data, and uploads question papers |
| **Invigilator** | Examination supervisor who controls exam start/end via the portal |
| **Device** | An Android tablet registered in the system |
| **Pairing** | The process of associating a physical tablet with a registered student seat |
| **Release** | The authorized act of making a question paper available to tablets |
| **LAN** | Local Area Network — the private Wi-Fi network used during exams |
| **TLS** | Transport Layer Security — encrypted communication protocol |
| **AES-256-GCM** | Authenticated encryption algorithm used for question paper files |
| **RBAC** | Role-Based Access Control |
| **MVP** | Minimum Viable Product |
| **PDF** | Portable Document Format — the file format for question papers |

### 1.4 Document Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-11-18 | Initial version — Module 0 approved |

---

## 2. Overall Description

### 2.1 Product Perspective

SmartExam is a standalone local-network application. It has no dependency on the internet during examination operation. It consists of:

- A central **Ubuntu Linux server** running the FastAPI backend and MySQL database.
- A **web portal** accessible from the Admin and Invigilator's devices over the LAN.
- Up to **30 Android tablets**, each displaying one student's question paper.

### 2.2 Pilot Deployment Parameters

| Parameter | Value |
|---|---|
| Maximum simultaneous students | 30 |
| Network type | Dedicated Wi-Fi (isolated from internet) |
| Server hardware | Institution-owned Ubuntu Linux machine |
| Tablet OS | Android (version to be confirmed in Module 7) |
| Connectivity | Local network only during exams |

### 2.3 Product Functions (Summary)

- Admin manages users, devices, classrooms, seats, and examinations.
- Admin uploads encrypted question papers.
- Invigilator starts and ends examinations.
- System distributes the correct question paper to each paired tablet upon authorized release.
- Tablets display the PDF; students read and write on physical answer sheets.
- Audit logs record every significant action.

---

## 3. Stakeholders and User Roles

### 3.1 Role Definitions

| Role | Access Level | Primary Responsibilities |
|---|---|---|
| **Admin** | Full system access | Configure system; manage all entities; upload papers; view all logs |
| **Invigilator** | Examination control only | Monitor exam room; start/end exams; trigger emergency fallback |
| **Android Tablet** | Assigned student device | Receive and display question paper (not a human role) |

### 3.2 Role Hierarchy

```
Admin (superuser)
  └── Invigilator (assigned per exam session)
        └── Android Device (registered and paired per seat)
```

> **Note:** Students have no login credentials. They interact only with the tablet's read-only PDF display.

---

## 4. Functional Requirements

### 4.1 Authentication and Role-Based Access Control (Module 1)

| ID | Requirement |
|---|---|
| FR-AUTH-01 | The system shall provide a login interface for Admin and Invigilator roles. |
| FR-AUTH-02 | Passwords shall be stored as bcrypt hashes. Plaintext passwords shall never be stored. |
| FR-AUTH-03 | The system shall issue JWT access tokens upon successful login. |
| FR-AUTH-04 | JWT tokens shall expire after a configurable duration (default: 60 minutes). |
| FR-AUTH-05 | Refresh tokens shall be supported to extend sessions without re-login. |
| FR-AUTH-06 | Access to each API endpoint shall be enforced based on the user's role. |
| FR-AUTH-07 | An Invigilator shall not access admin-only functions (user management, paper upload). |
| FR-AUTH-08 | Failed login attempts shall be logged with timestamp and source IP. |
| FR-AUTH-09 | Account lockout shall be implemented after 5 consecutive failed login attempts. |

### 4.2 Student Management (Module 2)

| ID | Requirement |
|---|---|
| FR-STU-01 | Admin shall be able to create, read, update, and deactivate student records. |
| FR-STU-02 | Each student record shall store: student ID, full name, class/section, roll number. |
| FR-STU-03 | Admin shall be able to import students in bulk via CSV. |
| FR-STU-04 | Student records shall not be permanently deleted; only deactivated. |
| FR-STU-05 | Admin shall be able to search and filter students by name, ID, or class. |

### 4.3 Classroom and Seating Management (Module 3)

| ID | Requirement |
|---|---|
| FR-CLASS-01 | Admin shall define classrooms with a name and seating capacity. |
| FR-CLASS-02 | Admin shall define individual seats within a classroom. |
| FR-CLASS-03 | Admin shall assign students to specific seats for a specific examination. |
| FR-CLASS-04 | Each seat assignment shall link one student to one tablet for one examination. |
| FR-CLASS-05 | The system shall prevent duplicate student or tablet assignments in the same exam. |

### 4.4 Android Device Management (Module 4)

| ID | Requirement |
|---|---|
| FR-DEV-01 | Each Android tablet shall complete a one-time authenticated registration with the server. |
| FR-DEV-02 | The server shall assign a unique device ID to each registered tablet. |
| FR-DEV-03 | Admin shall be able to view, enable, disable, and deregister devices. |
| FR-DEV-04 | Devices shall authenticate to the server using a device token stored in Android Keystore. |
| FR-DEV-05 | The registration process shall require Admin approval before a device is active. |
| FR-DEV-06 | Admin shall pair a device to a seat for a given examination. |

### 4.5 Examination Management (Module 5)

| ID | Requirement |
|---|---|
| FR-EXAM-01 | Admin shall create examinations with: name, subject, date, start time, duration, assigned classroom. |
| FR-EXAM-02 | Each examination shall have a lifecycle status: Draft, Scheduled, Active, Ended, Cancelled. |
| FR-EXAM-03 | Admin shall assign an Invigilator to each examination. |
| FR-EXAM-04 | Admin shall assign a question paper to an examination. |
| FR-EXAM-05 | The system shall prevent starting an examination if the question paper is not uploaded and assigned. |
| FR-EXAM-06 | The system shall prevent starting an examination before the scheduled date/time window. |

### 4.6 Question Paper Upload and Security (Module 6)

| ID | Requirement |
|---|---|
| FR-QP-01 | Admin shall upload question papers in PDF format only. |
| FR-QP-02 | Uploaded PDFs shall be validated (file type, minimum size). |
| FR-QP-03 | The PDF shall be encrypted at rest using AES-256-GCM immediately upon upload. |
| FR-QP-04 | The encryption key shall not be stored in the main database. It shall be stored in a separate secure key store. |
| FR-QP-05 | The encrypted file shall be stored on the server filesystem, not as a database BLOB. |
| FR-QP-06 | The unencrypted PDF shall not persist on disk after encryption completes. |
| FR-QP-07 | A SHA-256 integrity hash of the original PDF shall be stored and verified before distribution. |
| FR-QP-08 | Tablets shall not be able to download the question paper before the Invigilator authorizes release. |

### 4.7 Android Student Display Application (Module 7)

| ID | Requirement |
|---|---|
| FR-APP-01 | The app shall display the assigned question paper PDF using the Android PdfRenderer API. |
| FR-APP-02 | Students shall be able to zoom and navigate pages within the PDF. |
| FR-APP-03 | The app shall be read-only. Students shall not be able to annotate, edit, or export the PDF. |
| FR-APP-04 | The app shall maintain the exam state (waiting, active, ended) via WebSocket connection. |
| FR-APP-05 | The app shall download and locally cache the encrypted paper before the exam starts, so a brief Wi-Fi drop does not interrupt display. |
| FR-APP-06 | The app shall verify the SHA-256 hash of the decrypted paper before displaying it. |
| FR-APP-07 | The app shall clear the decrypted paper from memory when the exam ends. |
| FR-APP-08 | The app shall display a "Waiting for exam to start" screen until the Invigilator releases the paper. |

### 4.8 Local Paper Distribution and Examination Control (Module 8)

| ID | Requirement |
|---|---|
| FR-DIST-01 | The Invigilator shall initiate exam start from the web portal. |
| FR-DIST-02 | Upon start, the server shall push a release signal to all paired tablets via WebSocket. |
| FR-DIST-03 | Each tablet shall decrypt and display its paper only after receiving the authorized release signal. |
| FR-DIST-04 | The Invigilator shall be able to end the examination, triggering all tablets to clear the paper. |
| FR-DIST-05 | The portal shall display real-time connection status for each registered tablet. |
| FR-DIST-06 | If a tablet loses Wi-Fi after downloading its paper, it shall continue displaying the paper for the examination duration. |

### 4.9 Monitoring, Reports and Audit Logs (Module 9)

| ID | Requirement |
|---|---|
| FR-LOG-01 | The system shall write an audit log entry for every significant action. |
| FR-LOG-02 | Log entries shall record: action type, actor (user or device), timestamp, affected entity, outcome. |
| FR-LOG-03 | Admin shall be able to view and filter audit logs by date, actor, and action type. |
| FR-LOG-04 | Admin shall be able to export audit logs as CSV. |
| FR-LOG-05 | The monitoring dashboard shall show: total devices, connected devices, exam status per session. |

---

## 5. Non-Functional Requirements

### 5.1 Performance

| ID | Requirement |
|---|---|
| NFR-PERF-01 | The system shall support 30 simultaneous tablet connections without degradation. |
| NFR-PERF-02 | PDF distribution to all 30 tablets shall complete within 2 minutes of release. |
| NFR-PERF-03 | API response time for standard requests shall be under 500 ms on the local network. |
| NFR-PERF-04 | The web portal shall load within 3 seconds on the local network. |

### 5.2 Reliability

| ID | Requirement |
|---|---|
| NFR-REL-01 | The server shall recover from an unplanned restart within 2 minutes. |
| NFR-REL-02 | Tablets shall buffer the downloaded paper locally to tolerate up to 5 minutes of Wi-Fi interruption. |
| NFR-REL-03 | The system shall provide an emergency fallback procedure when the server fails during an exam. |

### 5.3 Usability

| ID | Requirement |
|---|---|
| NFR-USE-01 | The web portal shall function on a modern browser (Chrome, Firefox, Edge) without additional plugins. |
| NFR-USE-02 | Invigilator controls shall be accessible within 3 navigation actions from login. |
| NFR-USE-03 | The Android app UI shall require no training for exam-day display. |

### 5.4 Maintainability

| ID | Requirement |
|---|---|
| NFR-MNT-01 | Backend code shall follow PEP 8 style conventions. |
| NFR-MNT-02 | Database changes shall be managed through versioned migration scripts. |
| NFR-MNT-03 | All modules shall have automated test coverage before being declared complete. |

### 5.5 Scalability

| ID | Requirement |
|---|---|
| NFR-SCALE-01 | The database schema shall support increasing capacity beyond 30 students without redesign. |
| NFR-SCALE-02 | The architecture shall support multiple exam classrooms if the institution grows. |

---

## 6. Security Requirements

| ID | Requirement |
|---|---|
| SEC-01 | All communication between server and clients (web and Android) shall use TLS. |
| SEC-02 | Passwords shall be hashed using bcrypt with a minimum cost factor of 12. |
| SEC-03 | Question papers shall be encrypted at rest using AES-256-GCM. |
| SEC-04 | Encryption keys shall not be stored in the main MySQL database. |
| SEC-05 | Android device credentials shall be stored in Android Keystore, not in SharedPreferences. |
| SEC-06 | A tablet shall not be able to download another student's paper. |
| SEC-07 | The decryption key shall be delivered to the tablet only after authorized release. |
| SEC-08 | The API shall reject all unauthenticated requests with HTTP 401. |
| SEC-09 | RBAC shall be enforced server-side; client-side role hiding is UI convenience only. |
| SEC-10 | No question paper content shall appear in server logs. |
| SEC-11 | The system shall not claim to be fully tamper-proof on rooted Android devices. |
| SEC-12 | Sensitive operations shall require re-authentication or confirmation. |

---

## 7. System Constraints

| Constraint | Detail |
|---|---|
| **No internet** | The system shall operate without internet during examinations. |
| **Local network only** | All communication is confined to the institution's private Wi-Fi. |
| **Fixed tech stack** | Python 3.12, FastAPI, MySQL, React/TS, Kotlin/Jetpack Compose. Changes require approval. |
| **PDF only** | Question papers must be uploaded in PDF format. No other format is supported. |
| **Read-only display** | Students cannot interact with the question paper beyond reading. |
| **Physical answer sheets** | Digital answer collection is out of scope. |
| **Ubuntu server** | Production server runs Ubuntu Linux. Development may use Windows. |

---

## 8. Emergency Fallback Requirements

| ID | Requirement |
|---|---|
| EMG-01 | If the server fails during an active examination, tablets already displaying the paper shall continue to show it without server contact. |
| EMG-02 | The institution shall maintain printed backup copies of question papers in a sealed envelope. |
| EMG-03 | The Invigilator shall have a documented manual fallback procedure to distribute printed copies if fewer than 80% of tablets are functioning. |
| EMG-04 | The fallback procedure shall be documented and trained before each examination session. |
| EMG-05 | The system shall log the last known state of all tablets before server failure. |

---

## 9. Audit and Compliance Requirements

The following actions shall be recorded in the audit log:

| Event Category | Events Logged |
|---|---|
| **Authentication** | Login success, login failure, logout, token refresh, account lockout |
| **Question Papers** | Upload, assignment to exam, release, integrity check pass/fail |
| **Examinations** | Create, schedule, start, end, cancel |
| **Device Actions** | Registration, approval, pairing, connection, disconnection, deregistration |
| **User Management** | Admin created, Invigilator created, password reset, role change, deactivation |
| **System** | Server start, server stop, error events, emergency fallback invocation |

---

## 10. Assumptions and Dependencies

| Item | Assumption |
|---|---|
| **Network infrastructure** | The institution provides a stable dedicated Wi-Fi access point covering the exam room. |
| **Hardware** | 30 Android tablets are procured and charged before each exam session. |
| **Device management** | Tablets are pre-configured and app is pre-installed before exam day. |
| **Admin training** | At least one Admin is trained to operate the system before the pilot. |
| **Server stability** | The local Ubuntu server has a UPS (uninterruptible power supply). |

---

## 11. Out of Scope

The following features are explicitly **not** part of the SmartExam MVP:

- Digital answer collection or automated marking.
- Online/internet-based delivery.
- Student login credentials on tablets.
- Multi-institution or cloud deployment.
- Video surveillance integration.
- Biometric authentication.
- Native iOS or Windows tablet support.
- Printing functionality from tablets.
- Question paper authoring or editing inside the system.
