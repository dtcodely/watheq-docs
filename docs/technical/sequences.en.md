---
title: Sequence Diagrams
---

# Sequence Diagrams — System Interaction Flows

> **Total Diagrams**: 15 | **Format**: Mermaid sequenceDiagram | **Source**: `backend/src/` (24 Controllers, 5 Middleware, 4 Services)

---

## Diagram Index

| # | Diagram | Complexity | Key Participants |
|---|---------|-----------|----------------|
| 1 | [Login & Authentication](#auth) | ⭐⭐⭐ | User, AuthSvc, DB |
| 2 | [Create Document](#create-doc) | ⭐⭐⭐⭐ | User, DocSvc, OCR, BullMQ |
| 3 | Upload Attachments | ⭐⭐⭐ | User, FileStore, DocSvc |
| 4 | Archive Document | ⭐⭐⭐ | User, DocSvc, FileStore |
| 5 | [Workflow Approval](#workflow) | ⭐⭐⭐⭐⭐ | User×2, WfSvc, Escalation |
| 6 | Search & Retrieval | ⭐⭐⭐ | User, SearchSvc, DB GIN |
| 7 | [Electronic Signature](#signature) | ⭐⭐⭐⭐ | User, DocSvc, DB |
| 8 | Roles & Permissions | ⭐⭐⭐ | Admin, AuthSvc, DB |
| 9 | Notifications & Alerts | ⭐⭐⭐⭐ | NotifSvc, Socket.IO, WebPush |
| 10 | [Backup & Restore](#backup) | ⭐⭐⭐ | BackupSvc, 11 Models, FS |
| 11 | Audit Log | ⭐⭐ | AuditSvc ← All Services |
| 12 | Document Sharing | ⭐⭐⭐ | User×2, DocSvc, NotifSvc |
| 13 | [OCR Processing](#ocr) | ⭐⭐⭐⭐ | BullMQ, Worker, OCR, DB |
| 14 | Document Classification | ⭐⭐⭐ | User, DocSvc, DB |
| 15 | Export & Print | ⭐⭐ | User, DocSvc, FileStore |

---

<a id="auth"></a>
## 1. Login & Authentication

```mermaid
sequenceDiagram
    actor U as "User"
    participant FE as "Frontend"
    participant Auth as "Auth Service"
    participant Audit as "Audit Service"
    participant DB as "Database"

    U->>FE: Enter email/username + password
    FE->>Auth: POST /api/auth/login
    Note over Auth: loginLimiter (5 req/15min)

    Auth->>DB: findUnique(User) by email/username
    DB-->>Auth: User { isActive, deletedAt, isTwoFactorEnabled, ... }

    alt Account inactive or deleted
        Auth->>Audit: logActivity(LOGIN_FAILED)
        Auth-->>FE: 401 Account disabled
    else Wrong password
        Auth->>Auth: bcrypt.compare(password, hash)
        alt No match
            Auth->>Audit: logActivity(LOGIN_FAILED)
            Auth-->>FE: 401 Wrong password
        else Match
            alt MFA enabled
                Auth-->>FE: 200 { requiresMfa: true, tempToken }
                FE-->>U: Enter TOTP code
                U->>FE: Enter code
                FE->>Auth: POST /api/auth/mfa/verify
                Auth->>Auth: speakeasy.totp.verify
                alt Valid code
                    Auth->>Auth: generateAccessToken(24h) + generateRefreshToken(7d)
                else Invalid
                    Auth->>Audit: logActivity(MFA_FAILED)
                    Auth-->>FE: 401 Invalid MFA code
                end
            else No MFA
                Auth->>Auth: generateAccessToken(24h) + generateRefreshToken(7d)
            end
            Auth->>DB: update lastLoginAt = now()
            Auth->>Audit: logActivity(LOGIN_SUCCESS)
            Auth-->>FE: 200 { accessToken, refreshToken, user }
            FE-->>U: ✅ Login successful
        end
    end
```

---

<a id="create-doc"></a>
## 2. Create Document

```mermaid
sequenceDiagram
    actor U as "User"
    participant FE as "Frontend"
    participant API as "API Gateway"
    participant Doc as "Document Service"
    participant FS as "File Storage"
    participant Redis as "BullMQ"
    participant OCR as "OCR Service"
    participant DB as "Database"

    U->>FE: Fill document creation form
    FE->>API: POST /api/documents (+ attachments)
    Note over API: authenticate → authorize → requirePermission("documents.create")

    API->>Doc: Create document
    Doc->>DB: create(Document { trackingNumber, subject, type, ... })
    DB-->>Doc: Document { id, trackingNumber }

    opt Has attachments
        API->>FS: Multer: store files (max 50MB each)
        FS->>FS: uploads/{Cabinet}/{Folder}/{docId}/
        FS-->>Doc: fileMetadata[]
        Doc->>DB: create(Attachment[])
    end

    Doc->>Redis: addJob("document-processing", { docId })
    Note over Redis: 3 attempts, exponential backoff 5s

    Doc-->>FE: 201 { document, attachments }
    FE-->>U: ✅ Document created successfully

    Note over Redis,DB: Async background processing
    Redis-->>OCR: processDocument(docId)
    loop For each attachment
        OCR->>FS: Read file
        OCR->>OCR: PDF→pdf-parse / DOCX→mammoth / IMG→tesseract.js / TXT→direct
    end
    OCR->>DB: update Document.textContent
    Note over DB: GIN Index on textContent for Full-Text Search
```

---

<a id="workflow"></a>
## 5. Workflow Approval

```mermaid
sequenceDiagram
    actor U1 as "Document Creator"
    actor U2 as "Approver"
    participant FE as "Frontend"
    participant API as "API Gateway"
    participant Wf as "Workflow Engine"
    participant Notif as "Notification Service"
    participant DB as "Database"
    participant Esc as "Escalation Service (every hour)"

    U1->>FE: Create approval workflow + assign reviewers
    FE->>API: POST /api/workflows { documentId, steps }
    API->>Wf: createWorkflow(documentId, steps)
    Wf->>DB: create(Workflow + WorkflowStep[])
    Wf->>Notif: Notify first reviewer (U2)
    Wf-->>FE: 201 { workflow, steps }
    FE-->>U1: ✅ Workflow created

    U2->>FE: Open notification → review document
    FE->>API: PATCH /api/workflows/steps/:stepId

    alt Approve
        Wf->>DB: update step { status: APPROVED, completedAt }
        alt Last step
            Wf->>DB: update workflow { status: APPROVED }
            Wf->>Notif: Notify U1 of final approval
        else Has next step
            Wf->>Notif: Notify next reviewer
        end
    else Reject
        Wf->>DB: update step+workflow { status: REJECTED }
        Wf->>Notif: Notify U1 of rejection with reason
    end

    loop Every hour (cron job)
        Esc->>DB: Find pending + overdue WorkflowSteps
        Esc->>DB: update { assignedToId: dept.managerId, isEscalated: true }
        Esc->>Notif: ESCALATION_EXECUTED → notify manager
        Esc->>Notif: ESCALATION_WARNING → notify original employee
    end
```

---

<a id="signature"></a>
## 7. Electronic Signature

```mermaid
sequenceDiagram
    actor U as "User"
    participant FE as "Frontend"
    participant API as "API Gateway"
    participant Doc as "Document Service"
    participant Audit as "Audit Service"
    participant DB as "Database"

    alt Scenario 1: Direct signature
        U->>FE: Open document → request signature
        FE->>API: PATCH /api/signatures { documentId }
        API->>Doc: signDocument(documentId, userId)

        alt User has no signature set up
            Doc-->>FE: 400 "Please configure your signature first"
        else
            FE-->>U: Enter signature PIN
            U->>FE: Enter PIN
            FE->>API: POST /api/signatures/verify { documentId, pin }
            API->>Doc: verifyPin(userId, pin)
            alt PIN correct
                Doc->>DB: create(DocumentSignature { signatureData, signedAt, ipAddress, userAgent })
                Doc->>Audit: logActivity(DOCUMENT_SIGNED)
                Doc-->>FE: ✅ Signed successfully
            else PIN incorrect
                Doc->>Audit: logActivity(SIGNATURE_FAILED)
                Doc-->>FE: 401 Incorrect PIN
            end
        end

    else Scenario 2: Mobile signature
        U->>FE: Select mobile signature
        FE->>API: POST /api/signatures/mobile { documentId }
        API->>Doc: createMobileSession(documentId, userId)
        Doc->>DB: create mobile session { sessionId, status: PENDING }
        Doc-->>FE: "Signature link sent to your phone"
        loop Poll every 3 seconds
            FE->>API: GET /api/signatures/mobile/{sessionId}/status
            alt COMPLETED
                Doc->>DB: create(DocumentSignature)
                Doc->>Audit: logActivity(DOCUMENT_SIGNED_MOBILE)
                Doc-->>FE: ✅ Mobile signature completed
            else PENDING
                Doc-->>FE: 202 Still waiting...
            end
        end
    end
```

---

<a id="backup"></a>
## 10. Backup & Restore

```mermaid
sequenceDiagram
    participant Cron as "backup.scheduler (daily 02:00)"
    participant API as "API Gateway (manual - Admin)"
    participant Backup as "Backup Service"
    participant DB as "Database"
    participant FS as "File Storage"
    participant Notif as "Notification Service"

    alt Automatic (scheduled)
        Cron->>Backup: createBackup()
    else Manual (Admin)
        API->>Backup: POST /api/backup (Admin only, 5 req/h)
    end

    Backup->>DB: findUnique(SystemSetting) key="backup_enabled"

    alt Enabled
        Backup->>Backup: generateFilename("watheq-backup-{YYYY-MM-DD}.json")
        loop For each of 11 Models
            Backup->>DB: findMany(Model)
            DB-->>Backup: records[]
        end
        Note over Backup: User, Department, Document, Category, Folder, Cabinet, ExternalParty, Notification, AuditLog, SystemSetting, Role
        Backup->>FS: writeFile(filename, jsonData)
        Backup->>DB: update SystemSetting { backup_last_status: "success" }
        Backup-->>API: 201 { filename, size, createdAt }
    else Write failure
        Backup->>DB: update SystemSetting { backup_last_status: "failed" }
        Backup->>Notif: Notify all ADMINs of failure (AR + EN)
    end
```

---

<a id="ocr"></a>
## 13. OCR Processing

```mermaid
sequenceDiagram
    participant Doc as "Document Service"
    participant Redis as "BullMQ"
    participant Worker as "documentWorker"
    participant FS as "File Storage"
    participant Extractor as "TextExtraction Service"
    participant DB as "Database"

    Doc->>Redis: addJob("document-processing", { docId })
    Note over Redis: attempts: 3, backoff: exponential 5s

    Redis-->>Worker: process(docId)
    Worker->>DB: findUnique(Document + attachments)
    DB-->>Worker: Document { subject, content, attachments[] }

    loop For each attachment
        Worker->>FS: readFile(attachment.path)
        FS-->>Worker: fileBuffer
        Worker->>Extractor: extractText(fileBuffer, mimeType)
        alt PDF
            Extractor->>Extractor: pdf-parse(fileBuffer)
        else DOCX
            Extractor->>Extractor: mammoth(fileBuffer)
        else Image (JPEG/PNG)
            Extractor->>Extractor: tesseract.js(fileBuffer, "ara+eng")
        else TXT
            Extractor->>Extractor: buffer.toString("utf-8")
        else
            Worker->>Worker: skip (unsupported type)
        end
        Extractor-->>Worker: extractedText
    end

    Worker->>Worker: fullText = textParts.join(" ").trim()
    Worker->>DB: update Document { textContent: fullText }
    Note over DB: GIN Index on textContent for Full-Text Search
    Worker-->>Redis: ✅ job complete

    opt Failed after 3 attempts
        Worker-->>Redis: ❌ job failed → dead letter queue
    end
```

---

## Critical Events Reference

| Event | Audit Log | Notification | Async? |
|-------|-----------|-------------|--------|
| Successful Login | ✅ LOGIN_SUCCESS | ❌ | ❌ |
| Document Creation | ✅ CREATE_DOCUMENT | ✅ | ✅ OCR |
| Workflow Approval | ✅ WORKFLOW_APPROVED | ✅ | ❌ |
| Workflow Rejection | ✅ WORKFLOW_REJECTED | ✅ | ❌ |
| Workflow Escalation | ✅ ESCALATION | ✅ Manager + Employee | ✅ hourly |
| Document Signing | ✅ DOCUMENT_SIGNED | ✅ | ❌ |
| Document Disposal | ✅ DISPOSAL | ✅ Supervisor | ✅ Retention |
| Backup Success | ✅ BACKUP_SUCCESS | ❌ | ✅ scheduled |
| Backup Failure | ✅ BACKUP_FAILED | ✅ All ADMINs | ✅ |
