---
title: Operational Flowcharts
---

# Operational Flowcharts — System Processing Cycle

> **Source**: `backend/src/` — 24 Controllers, 5 Middleware, 4 Services, 20 Utilities, 2 Workers, 4 Schedulers

---

## System Overview Diagram

```mermaid
flowchart LR
    classDef core fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px
    classDef process fill:#f5f0ff,color:#000,stroke:#5c2d91,stroke-width:2px

    O0>"🔄 Watheq System\nFull Processing Cycle"]:::core

    O1[① Security & Auth\nJWT + MFA + Rate Limit]:::process
    O2[② Access Control\nRole + Permission Guards]:::process
    O3[③ Document Creation\nOCR + BullMQ + Attachments]:::process
    O4[④ Workflow & Approvals\nWorkflow + Escalation]:::process
    O5[⑤ Electronic Signature\nPIN + Signature + Certificate]:::process
    O6[⑥ Archiving & Storage\nCabinet + Folder + Encryption]:::process
    O7[⑦ Search & Retrieval\nFTS + Filters + GIN Index]:::process
    O8[⑧ Document Lifecycle\nRetention + Legal Hold + Disposal]:::process
    O9[⑨ Backup\nScheduler + 11 Models Export]:::process
    O10[⑩ Notifications & Audit\nSocket.IO + Web Push + AuditLog]:::process

    O0 --> O1 --> O2 --> O3 --> O4
    O4 --> O5 --> O6
    O3 -.-> O7
    O6 --> O8
    O8 -.-> O9
    O3 -.-> O10
    O4 -.-> O10
    O8 -.-> O10
```

---

## 10 Operational Phases

| # | Phase | Source Files | Description |
|---|-------|-------------|-------------|
| ① | **Security & Auth** 🔐 | `auth.controller.ts`, `mfa.controller.ts` | JWT + MFA + bcrypt |
| ② | **Access Control** 🛡️ | `middleware/auth.ts`, `rateLimit.middleware.ts` | RBAC + Rate Limiting |
| ③ | **Document Creation** 📄 | `document.controller.ts`, `workers/documentWorker.ts` | OCR + BullMQ + FTS |
| ④ | **Workflow & Approvals** 📋 | `workflow.controller.ts`, `escalation.service.ts` | Multi-step approval + Auto-escalation |
| ⑤ | **Electronic Signature** ✍️ | `signature.controller.ts` | PIN + Signature + Certificate |
| ⑥ | **Archiving & Storage** 🗂️ | `cabinet.controller.ts`, `folder.controller.ts` | Cabinet + Folder + AES-256-GCM |
| ⑦ | **Search & Retrieval** 🔍 | `search.controller.ts` | FTS GIN + 15 Filters |
| ⑧ | **Document Lifecycle** 🔄 | `retention.controller.ts`, `disposal.controller.ts` | Retention + Legal Hold + Disposal |
| ⑨ | **Backup** 💾 | `backup.controller.ts`, `backup.scheduler.ts` | Daily scheduler + 11 Models JSON export |
| ⑩ | **Notifications & Audit** 📢 | `notification.controller.ts`, `audit.controller.ts` | Socket.IO + Web Push + AuditLog |

---

## Phase ①: Security & Authentication

```mermaid
flowchart TD
    classDef process fill:#f5f0ff,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    S1[Login Request\nPOST /api/auth/login]:::process
    S2[Lookup User\nPrisma: email/username]:::process
    S3{isActive = true?}:::decision
    S4{deletedAt IS NULL?}:::decision
    S5[Verify Password\nbcrypt.compare]:::process
    S6{Match?}:::decision
    S7{isTwoFactorEnabled?}:::decision
    S8[Generate tempToken\n5 min expiry + MFA required]:::process
    S9[Verify TOTP\nspeakeasy.totp.verify]:::decision
    S13[Generate JWT Tokens\naccessToken: 24h\nrefreshToken: 7d]:::process
    S14[(Login Success → AuditLog)]:::terminal
    S15[(Failed Attempt → AuditLog)]:::terminal

    S1 --> S2 --> S3
    S3 -->|No| S15
    S3 -->|Yes| S4
    S4 -->|Deleted| S15
    S4 -->|No| S5 --> S6
    S6 -->|No| S15
    S6 -->|Yes| S7
    S7 -->|MFA| S8 --> S9
    S9 -->|Valid| S13
    S9 -->|Invalid| S15
    S7 -->|No MFA| S13 --> S14
```

---

## Phase ②: Access Control (RBAC)

```mermaid
flowchart TD
    classDef process fill:#f5f0ff,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    A1[Incoming API Request]:::process
    A2[authenticate middleware\nExtract Bearer Token]:::process
    A3{Token Valid?}:::decision
    A4[authorize middleware\nRole Check]:::process
    A5{Role Allowed?}:::decision
    A6[requirePermission middleware]:::process
    A7{Has Permission?}:::decision
    A8[Rate Limit Check\n6 Limiters]:::decision
    A9{Within Limit?}:::decision
    A10[(Execute Request + AuditLog)]:::terminal

    A1 --> A2 --> A3
    A3 -->|No| ERR1["401 Unauthorized"]:::terminal
    A3 -->|Yes| A4 --> A5
    A5 -->|No| ERR2["403 Forbidden"]:::terminal
    A5 -->|Yes| A6 --> A7
    A7 -->|No| ERR2
    A7 -->|Yes| A8 --> A9
    A9 -->|No| ERR3["429 Too Many Requests"]:::terminal
    A9 -->|Yes| A10
```

!!! info "Rate Limiter Configuration"
    | Limiter | Limit |
    |---------|-------|
    | `generalLimiter` | 1000 req/15min |
    | `authLimiter` | 100 req/15min |
    | `loginLimiter` | 5 req/15min |
    | `createLimiter` | 200 req/min |
    | `searchLimiter` | 300 req/min |
    | `backupLimiter` | 5 req/hour |

---

## Background Job Schedule

| Job | Schedule | Expected Duration |
|-----|----------|------------------|
| Backup | Daily at 02:00 | ~2-5 minutes |
| Retention Check | Daily at 04:00 | ~1-3 minutes |
| Workflow Escalation | Every hour | ~30 seconds |
| Performance Reports | Sunday 00:00 | ~5-10 minutes |

---

## Proposed Module Architecture

| # | Module | Contents | Dependencies |
|---|--------|---------|--------------|
| 1 | **Core** | User, Department, Organization, Settings | — |
| 2 | **Security** | Auth, MFA, RBAC, Rate Limit | Core |
| 3 | **Documents** | Document, Version, Signature, Forward | Core + Security |
| 4 | **Filing** | Cabinet, Folder, Physical, Category | Core + Documents |
| 5 | **Workflow** | Workflow, Escalation, Notification, Retention | Documents + Core |
| 6 | **Search** | FTS, OCR, AI, Filters | Documents |
| 7 | **Operations** | Audit, Backup, Analytics | All |
