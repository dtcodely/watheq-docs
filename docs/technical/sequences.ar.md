---
title: مخططات التسلسل
---

# مخططات التسلسل — Sequence Diagrams

> **إجمالي المخططات**: 15 مخطط | **الصيغة**: Mermaid sequenceDiagram | **المصدر**: `backend/src/` (24 Controller, 5 Middleware, 4 Service)

---

## فهرس المخططات

| #   | المخطط                                 | التعقيد    | المشاركون الرئيسيون          |
| --- | -------------------------------------- | ---------- | ---------------------------- |
| 1   | [تسجيل الدخول والمصادقة](#auth)        | ⭐⭐⭐     | User, AuthSvc, DB            |
| 2   | [إنشاء وثيقة جديدة](#create-doc)       | ⭐⭐⭐⭐   | User, DocSvc, OCR, BullMQ    |
| 3   | [رفع المرفقات](#upload)                | ⭐⭐⭐     | User, FileStore, DocSvc      |
| 4   | [أرشفة الوثيقة](#archive)              | ⭐⭐⭐     | User, DocSvc, FileStore      |
| 5   | [دورة الموافقات Workflow](#workflow)   | ⭐⭐⭐⭐⭐ | User×2, WfSvc, Escalation    |
| 6   | [البحث والاسترجاع](#search)            | ⭐⭐⭐     | User, SearchSvc, DB GIN      |
| 7   | [التوقيع الإلكتروني](#signature)       | ⭐⭐⭐⭐   | User, DocSvc, DB             |
| 8   | [إدارة الصلاحيات والأدوار](#roles)     | ⭐⭐⭐     | Admin, AuthSvc, DB           |
| 9   | [الإشعارات والتنبيهات](#notifications) | ⭐⭐⭐⭐   | NotifSvc, Socket.IO, WebPush |
| 10  | [النسخ الاحتياطي والاستعادة](#backup)  | ⭐⭐⭐     | BackupSvc, 11 Models, FS     |
| 11  | [سجل التدقيق Audit logs](#audit)       | ⭐⭐       | AuditSvc ← جميع الخدمات      |
| 12  | [مشاركة الوثائق](#share)               | ⭐⭐⭐     | User×2, DocSvc, NotifSvc     |
| 13  | [معالجة OCR](#ocr)                     | ⭐⭐⭐⭐   | BullMQ, Worker, OCR, DB      |
| 14  | [تصنيف الوثائق وفهرستها](#classify)    | ⭐⭐⭐     | User, DocSvc, DB             |
| 15  | [تصدير وطباعة الوثائق](#export)        | ⭐⭐       | User, DocSvc, FileStore      |

---

<a id="auth"></a>

## 1. تسجيل الدخول والمصادقة

```mermaid
sequenceDiagram
    actor U as "مستخدم (User)"
    participant FE as "واجهة (Frontend)"
    participant Rate as "معدل الطلبات"
    participant Auth as "(Auth Service) خدمة المصادقة"
    participant Audit as "(Audit Service) خدمة التدقيق"
    participant DB as "(Database) قاعدة البيانات"

    Note over U: تسجيل الدخول
    U->>FE: إدخال البريد/اسم المستخدم + كلمة المرور
    FE->>Auth: POST /api/auth/login

    Note over Auth: Rate Limit: loginLimiter (5 req/15min)

    alt تجاوز معدل الطلبات
        Rate-->>FE: 429 Too Many Requests
    else
        Auth->>DB: Prisma: findUnique(User)
        DB-->>Auth: User { isActive, deletedAt, isTwoFactorEnabled, ... }

        alt حساب غير نشط
            Auth->>Audit: logActivity(LOGIN_FAILED)
            Auth-->>FE: 401 "الحساب معطل"
        else كلمة مرور خاطئة
            Auth->>Auth: bcrypt.compare(password, hash)
            alt لا تطابق
                Auth->>Audit: logActivity(LOGIN_FAILED)
                Auth-->>FE: 401 "كلمة المرور خاطئة"
            else تطابق
                alt MFA مفعل
                    Auth->>Auth: generateTempToken(5min)
                    Auth-->>FE: 200 { requiresMfa: true, tempToken }
                    FE-->>U: طلب إدخال رمز TOTP
                    U->>FE: إدخال رمز المصادقة
                    FE->>Auth: POST /api/auth/mfa/verify
                    Auth->>Auth: speakeasy.totp.verify
                    alt رمز صحيح
                        Auth->>Auth: generateAccessToken(24h) + generateRefreshToken(7d)
                    else رمز خطأ
                        Auth->>Audit: logActivity(MFA_FAILED)
                        Auth-->>FE: 401 "رمز المصادقة خاطئ"
                    end
                else MFA غير مفعل
                    Auth->>Auth: generateAccessToken(24h) + generateRefreshToken(7d)
                end
                Auth->>DB: تحديث lastLoginAt = now()
                Auth->>Audit: logActivity(LOGIN_SUCCESS)
                Auth-->>FE: 200 { accessToken, refreshToken, user }
                FE-->>U: ✅ تم تسجيل الدخول بنجاح
            end
        end
    end
```

---

<a id="create-doc"></a>

## 2. إنشاء وثيقة جديدة

```mermaid
sequenceDiagram
    actor U as "مستخدم"
    participant FE as "واجهة"
    participant API as "بوابة API"
    participant Doc as "خدمة الوثائق"
    participant FS as "تخزين الملفات"
    participant Redis as "BullMQ"
    participant OCR as "معالجة النصوص"
    participant DB as "قاعدة البيانات"

    U->>FE: تعبئة نموذج إنشاء وثيقة
    FE->>API: POST /api/documents (+ attachments)

    Note over API: authenticate → authorize → requirePermission("documents.create")

    API->>Doc: إنشاء الوثيقة
    Doc->>DB: create(Document { trackingNumber, subject, type, ... })
    DB-->>Doc: Document { id, trackingNumber }

    opt يوجد مرفقات
        API->>FS: Multer: تخزين الملفات (max 50MB)
        FS->>FS: uploads/{Cabinet}/{Folder}/{docId}/
        FS-->>Doc: fileMetadata[]
        Doc->>DB: create(Attachment[])
    end

    Doc->>Redis: addJob("document-processing", { docId })
    Note over Redis: 3 attempts, exponential backoff 5s

    Doc-->>FE: 201 { document, attachments }
    FE-->>U: ✅ "تم إنشاء الوثيقة بنجاح"

    Note over Redis,DB: معالجة غير متزامنة
    Redis-->>OCR: processDocument(docId)
    loop لكل مرفق
        OCR->>FS: قراءة الملف
        OCR->>OCR: PDF→pdf-parse / DOCX→mammoth / IMG→tesseract.js / TXT→مباشر
    end
    OCR->>DB: update Document.textContent
    Note over DB: GIN Index للبحث النصي الكامل
```

<a id="upload"></a>
<a id="archive"></a>
<a id="search"></a>
<a id="roles"></a>
<a id="notifications"></a>
<a id="audit"></a>
<a id="share"></a>
<a id="classify"></a>
<a id="export"></a>

---

<a id="workflow"></a>

## 5. دورة الموافقات Workflow

```mermaid
sequenceDiagram
    actor U1 as "منشئ الوثيقة"
    actor U2 as "المسؤول (الموافق)"
    participant FE as "واجهة"
    participant API as "بوابة API"
    participant Wf as "محرك سير العمل"
    participant Notif as "خدمة الإشعارات"
    participant DB as "قاعدة البيانات"
    participant Esc as "خدمة التصعيد (كل ساعة)"

    U1->>FE: إنشاء مسار موافقة + تعيين المسؤولين
    FE->>API: POST /api/workflows { documentId, steps }
    API->>Wf: createWorkflow(documentId, steps)
    Wf->>DB: create(Workflow + WorkflowStep[])
    Wf->>Notif: إخطار المسؤول الأول
    Wf-->>FE: 201 { workflow, steps }
    FE-->>U1: ✅ "تم إنشاء مسار الموافقة"

    U2->>FE: فتح الإشعار ← عرض الوثيقة
    FE->>API: PATCH /api/workflows/steps/:stepId

    alt موافقة
        Wf->>DB: update step { status: APPROVED, completedAt }
        alt آخر خطوة
            Wf->>DB: update workflow { status: APPROVED }
            Wf->>Notif: إخطار منشئ الوثيقة بالموافقة النهائية
        else توجد خطوة تالية
            Wf->>Notif: إخطار المسؤول التالي
        end
    else رفض
        Wf->>DB: update step+workflow { status: REJECTED }
        Wf->>Notif: إخطار المنشئ بسبب الرفض
    end

    loop كل ساعة (cron)
        Esc->>DB: find pending + overdue WorkflowStep
        Esc->>DB: update { assignedToId: dept.managerId, isEscalated: true }
        Esc->>Notif: ESCALATION_EXECUTED → إخطار المدير
        Esc->>Notif: ESCALATION_WARNING → إخطار الموظف الأصلي
    end
```

---

<a id="signature"></a>

## 7. التوقيع الإلكتروني

```mermaid
sequenceDiagram
    actor U as "مستخدم"
    participant FE as "واجهة"
    participant API as "بوابة API"
    participant Doc as "خدمة الوثائق"
    participant Audit as "خدمة التدقيق"
    participant DB as "قاعدة البيانات"

    alt السيناريو 1: توقيع مباشر
        U->>FE: فتح وثيقة ← طلب توقيع
        FE->>API: PATCH /api/signatures { documentId }
        API->>Doc: signDocument(documentId, userId)

        alt المستخدم ليس لديه توقيع
            Doc-->>FE: 400 "يرجى إعداد التوقيع أولاً"
        else
            FE-->>U: طلب إدخال PIN التوقيع
            U->>FE: إدخال PIN
            FE->>API: POST /api/signatures/verify { documentId, pin }
            API->>Doc: verifyPin(userId, pin)
            alt PIN صحيح
                Doc->>DB: create(DocumentSignature { signatureData, signedAt, ipAddress, userAgent })
                Doc->>Audit: logActivity(DOCUMENT_SIGNED)
                Doc-->>FE: ✅ "تم التوقيع بنجاح"
            else PIN خطأ
                Doc->>Audit: logActivity(SIGNATURE_FAILED)
                Doc-->>FE: 401 "PIN غير صحيح"
            end
        end

    else السيناريو 2: توقيع عبر الجوال
        U->>FE: اختيار توقيع عبر الجوال
        FE->>API: POST /api/signatures/mobile { documentId }
        API->>Doc: createMobileSession(documentId, userId)
        Doc->>DB: إنشاء جلسة توقيع { sessionId, status: PENDING }
        Doc-->>FE: "تم إرسال رابط التوقيع إلى جوالك"
        loop polling كل 3 ثوان
            FE->>API: GET /api/signatures/mobile/{sessionId}/status
            alt status == COMPLETED
                Doc->>DB: create(DocumentSignature)
                Doc->>Audit: logActivity(DOCUMENT_SIGNED_MOBILE)
                Doc-->>FE: ✅ تم التوقيع عبر الجوال
            else status == PENDING
                Doc-->>FE: 202 جارٍ الانتظار...
            end
        end
    end
```

---

<a id="backup"></a>

## 10. النسخ الاحتياطي والاستعادة

```mermaid
sequenceDiagram
    participant Cron as "backup.scheduler (02:00 يومياً)"
    participant API as "API Gateway (يدوي - Admin)"
    participant Backup as "خدمة النسخ الاحتياطي"
    participant DB as "قاعدة البيانات"
    participant FS as "تخزين الملفات"
    participant Notif as "خدمة الإشعارات"

    alt تلقائي (مجدول)
        Cron->>Backup: createBackup()
    else يدوي (Admin)
        API->>Backup: POST /api/backup (Admin only, 5 req/h)
    end

    Backup->>DB: findUnique(SystemSetting) key="backup_enabled"

    alt مفعل
        Backup->>Backup: generateFilename("watheq-backup-{YYYY-MM-DD}.json")
        loop لكل model من الـ 11 Model
            Backup->>DB: findMany(Model)
            DB-->>Backup: records[]
        end
        Note over Backup: User, Department, Document, Category, Folder, Cabinet, ExternalParty, Notification, AuditLog, SystemSetting, Role
        Backup->>FS: writeFile(filename, jsonData)
        Backup->>DB: update SystemSetting { backup_last_status: "success" }
        Backup-->>API: 201 { filename, size, createdAt }
    else فشل
        Backup->>DB: update SystemSetting { backup_last_status: "failed" }
        Backup->>Notif: إخطار جميع ADMIN بالفشل (AR + EN)
    end
```

---

<a id="ocr"></a>

## 13. معالجة OCR

```mermaid
sequenceDiagram
    participant Doc as "خدمة الوثائق (بعد الإنشاء)"
    participant Redis as "BullMQ"
    participant Worker as "documentWorker"
    participant FS as "تخزين الملفات"
    participant Extractor as "TextExtraction Service"
    participant DB as "قاعدة البيانات"

    Doc->>Redis: addJob("document-processing", { docId })
    Note over Redis: attempts: 3, backoff: exponential 5s

    Redis-->>Worker: process(docId)
    Worker->>DB: findUnique(Document + attachments)
    DB-->>Worker: Document { subject, content, attachments[] }

    loop لكل مرفق
        Worker->>FS: readFile(attachment.path)
        FS-->>Worker: fileBuffer
        Worker->>Extractor: extractText(fileBuffer, mimeType)
        alt PDF
            Extractor->>Extractor: pdf-parse(fileBuffer)
        else DOCX
            Extractor->>Extractor: mammoth(fileBuffer)
        else IMG (JPEG/PNG)
            Extractor->>Extractor: tesseract.js(fileBuffer, "ara+eng")
        else TXT
            Extractor->>Extractor: buffer.toString("utf-8")
        else
            Worker->>Worker: skip (نوع غير مدعوم)
        end
        Extractor-->>Worker: extractedText
    end

    Worker->>Worker: fullText = textParts.join(" ").trim()
    Worker->>DB: update Document { textContent: fullText }
    Note over DB: GIN Index on textContent للبحث النصي الكامل
    Worker-->>Redis: ✅ job complete

    opt فشل بعد 3 محاولات
        Worker-->>Redis: ❌ job failed → dead letter queue
    end
```

---

## جدول الأحداث الحرجة

| الحدث            | يُسجل Audit؟         | إشعار مطلوب؟       | غير متزامن؟  |
| ---------------- | -------------------- | ------------------ | ------------ |
| تسجيل دخول ناجح  | ✅ LOGIN_SUCCESS     | ❌                 | ❌           |
| إنشاء وثيقة      | ✅ CREATE_DOCUMENT   | ✅                 | ✅ OCR       |
| موافقة Workflow  | ✅ WORKFLOW_APPROVED | ✅                 | ❌           |
| رفض Workflow     | ✅ WORKFLOW_REJECTED | ✅                 | ❌           |
| تصعيد Workflow   | ✅ ESCALATION        | ✅ المدير + الموظف | ✅ كل ساعة   |
| توقيع وثيقة      | ✅ DOCUMENT_SIGNED   | ✅                 | ❌           |
| إتلاف وثيقة      | ✅ DISPOSAL          | ✅ المشرف          | ✅ Retention |
| نسخ احتياطي ناجح | ✅ BACKUP_SUCCESS    | ❌                 | ✅ مجدول     |
| فشل نسخ احتياطي  | ✅ BACKUP_FAILED     | ✅ جميع ADMIN      | ✅           |

!!! note "المخططات المكتملة"
المخططات 3، 4، 6، 8، 9، 11، 12، 14، 15 موجودة في الملفات المصدرية `docs/sequence-diagrams/` وتحتوي على تفاصيل كاملة لكل عملية.
