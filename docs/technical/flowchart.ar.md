---
title: المخططات التشغيلية
---

# المخططات التشغيلية — دورة التشغيل الكاملة

> **المصدر**: `backend/src/` — 24 Controller، 5 Middleware، 4 Service، 20 Utility، 2 Worker، 4 Scheduler

---

## نظرة عامة — المخطط العام للنظام

```mermaid
flowchart LR
    classDef core fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px

    O0>"🔄 Watheq System\nدورة التشغيل المتكاملة"]:::core

    O1[① الأمن والصلاحيات\nJWT + MFA + Rate Limit]:::process
    O2[② التحكم بالوصول\nRole + Permission Guards]:::process
    O3[③ إنشاء الوثيقة\nOCR + BullMQ + Attachments]:::process
    O4[④ سير العمل والموافقات\nWorkflow + Escalation]:::process
    O5[⑤ التوقيع الإلكتروني\nPIN + Signature + Certificate]:::process
    O6[⑥ الأرشفة والتخزين\nCabinet + Folder + Encryption]:::process
    O7[⑦ البحث والاسترجاع\nFTS + Filters + GIN Index]:::process
    O8[⑧ دورة حياة الوثيقة\nRetention + Legal Hold + Disposal]:::process
    O9[⑨ النسخ الاحتياطي\nScheduler + 11 Models Export]:::process
    O10[⑩ الإشعارات وسجل التدقيق\nSocket.IO + Web Push + AuditLog]:::process

    O0 --> O1 --> O2 --> O3 --> O4
    O4 --> O5 --> O6
    O3 -.-> O7
    O6 --> O8
    O8 -.-> O9
    O2 -.-> O10
    O3 -.-> O10
    O4 -.-> O10
    O5 -.-> O10
    O8 -.-> O10
```

---

## جدول المراحل التشغيلية

| # | المرحلة | الملفات المصدرية | الوصف |
|---|---------|-----------------|-------|
| ① | **الأمن والصلاحيات** 🔐 | `auth.controller.ts`, `mfa.controller.ts` | JWT + MFA + bcrypt |
| ② | **التحكم بالوصول** 🛡️ | `middleware/auth.ts`, `middleware/rateLimit.middleware.ts` | RBAC + Rate Limit |
| ③ | **إنشاء الوثيقة** 📄 | `document.controller.ts`, `workers/documentWorker.ts` | OCR + BullMQ + FTS |
| ④ | **سير العمل والموافقات** 📋 | `workflow.controller.ts`, `services/escalation.service.ts` | Workflow + Escalation |
| ⑤ | **التوقيع الإلكتروني** ✍️ | `signature.controller.ts` | PIN + Signature + Certificate |
| ⑥ | **الأرشفة والتخزين** 🗂️ | `cabinet.controller.ts`, `folder.controller.ts` | Cabinet + Folder + AES-256-GCM |
| ⑦ | **البحث والاسترجاع** 🔍 | `search.controller.ts` | FTS GIN + 15 Filters |
| ⑧ | **دورة حياة الوثيقة** 🔄 | `retention.controller.ts`, `disposal.controller.ts` | Retention + Legal Hold + Disposal |
| ⑨ | **النسخ الاحتياطي** 💾 | `backup.controller.ts`, `utils/backup.scheduler.ts` | Scheduler + 11 Models JSON |
| ⑩ | **الإشعارات وسجل التدقيق** 📢 | `notification.controller.ts`, `audit.controller.ts` | Socket.IO + Web Push + AuditLog |

---

## المرحلة ①: الأمن والصلاحيات

```mermaid
flowchart TD
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    S1[طلب تسجيل الدخول\nPOST /api/auth/login]:::process
    S2[البحث عن المستخدم\nPrisma: email/username]:::process
    S3{isActive = true؟}:::decision
    S4{deletedAt IS NULL؟}:::decision
    S5[التحقق من كلمة المرور\nbcrypt.compare]:::process
    S6{مطابقة؟}:::decision
    S7{isTwoFactorEnabled؟}:::decision
    S8[إنشاء tempToken\nصلاحية 5 دقائق + MFA]:::process
    S9[التحقق من TOTP\nspeakeasy.totp.verify]:::decision
    S13[إنشاء JWT Tokens\naccessToken: 24h\nrefreshToken: 7d]:::process
    S14[(تسجيل الدخول بنجاح)]:::terminal
    S15[(محاولة فاشلة → AuditLog)]:::terminal

    S1 --> S2 --> S3
    S3 -->|لا| S15
    S3 -->|نعم| S4
    S4 -->|محذوف| S15
    S4 -->|لا| S5 --> S6
    S6 -->|لا| S15
    S6 -->|نعم| S7
    S7 -->|MFA| S8 --> S9
    S9 -->|صحيح| S13
    S9 -->|لا| S15
    S7 -->|لا MFA| S13 --> S14
```

---

## المرحلة ②: التحكم بالوصول (RBAC)

```mermaid
flowchart TD
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    A1[استقبال طلب API]:::process
    A2[authenticate middleware\nاستخراج Bearer Token]:::process
    A3{التوكن صالح؟}:::decision
    A4[authorize middleware\nالتحقق من الدور]:::process
    A5{الدور مسموح؟}:::decision
    A6[requirePermission middleware]:::process
    A7{يمتلك الصلاحية؟}:::decision
    A8[Rate Limit Check\n6 Limiters]:::decision
    A9{ضمن الحد؟}:::decision
    A10[(تنفيذ الطلب + AuditLog)]:::terminal

    A1 --> A2 --> A3
    A3 -->|لا| ERR1["401 Unauthorized"]:::terminal
    A3 -->|نعم| A4 --> A5
    A5 -->|لا| ERR2["403 Forbidden"]:::terminal
    A5 -->|نعم| A6 --> A7
    A7 -->|لا| ERR2
    A7 -->|نعم| A8 --> A9
    A9 -->|لا| ERR3["429 Too Many Requests"]:::terminal
    A9 -->|نعم| A10
```

!!! info "معدلات تحديد الطلبات (Rate Limiters)"
    | الـ Limiter | الحد |
    |------------|------|
    | `generalLimiter` | 1000 req/15min |
    | `authLimiter` | 100 req/15min |
    | `loginLimiter` | 5 req/15min |
    | `createLimiter` | 200 req/min |
    | `searchLimiter` | 300 req/min |
    | `backupLimiter` | 5 req/hour |

---

## المرحلة ③: إنشاء الوثيقة

```mermaid
flowchart TD
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef storage fill:#e8f5e9,color:#000,stroke:#2e7d32,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    D1[POST /api/documents]:::process
    D2[التحقق من الصلاحية\ndocuments.create]:::process
    D3[تعبئة البيانات الأساسية\nالموضوع + النوع + القسم]:::process
    D8{وجود مرفقات؟}:::decision
    D9[رفع الملفات\nMulter 50MB]:::process
    D13[إنشاء Document في DB\ntrackingNumber تلقائي فريد]:::storage
    D14[إرسال مهمة إلى BullMQ\ndocument-processing queue]:::process
    D15[Worker: OCR + استخراج النص\nPDF + IMG + DOCX + TXT]:::process
    D21[تخزين textContent\nفي Document.textContent]:::storage
    D22[GIN Index للبحث النصي]:::storage
    D23[إشعار فوري Socket.IO]:::process
    D26[(تم الإنشاء بنجاح)]:::terminal

    D1 --> D2 --> D3
    D3 --> D8
    D8 -->|نعم| D9 --> D13
    D8 -->|لا| D13
    D13 --> D14 --> D15 --> D21 --> D22 --> D23 --> D26
```

---

## المرحلة ④: سير العمل والموافقات

```mermaid
flowchart TD
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    W1[إنشاء مسار موافقة\nWorkflow + Steps]:::process
    W4[إخطار المسؤول الأول]:::process
    W7{قرار المسؤول}:::decision
    W8[موافقة: completedAt\nstatus → APPROVED]:::process
    W9[رفض: comment\nstatus → REJECTED]:::process
    W10{آخر خطوة؟}:::decision
    W11[(Workflow → APPROVED)]:::terminal
    W15[إخطار المسؤول التالي]:::process
    W16[(AuditLog)]:::terminal
    ESC[EscalationScheduler\nكل ساعة — تصعيد تلقائي\nللمراجعات المتأخرة]:::process

    W1 --> W4 --> W7
    W7 -->|قبول| W8 --> W10
    W10 -->|نعم| W11 --> W16
    W10 -->|لا| W15 --> W7
    W7 -->|رفض| W9 --> W16
    ESC -.-> W4
```

---

## المرحلة ⑧: دورة حياة الوثيقة

```mermaid
flowchart TD
    classDef process fill:#ede9f5,color:#000,stroke:#5c2d91,stroke-width:2px
    classDef decision fill:#fff3e0,color:#000,stroke:#e65100,stroke-width:2px
    classDef terminal fill:#5c2d91,color:#fff,stroke:#fff,stroke-width:2px

    L2[RetentionScheduler\nكل يوم 04:00]:::process
    L6{Legal Hold؟}:::decision
    L7[إخطار المدير\nبوثائق محمية]:::process
    L9{RetentionAction}:::decision
    L10[DISPOSE → إتلاف آمن\n+ شهادة إتلاف]:::process
    L11[ARCHIVE → أرشفة تلقائية]:::process
    L12[REVIEW → مراجعة يدوية]:::process
    L13[PERMANENT → احتفاظ دائم]:::process
    L23[(AuditLog: DISPOSAL)]:::terminal

    L2 --> L6
    L6 -->|نعم| L7
    L6 -->|لا| L9
    L9 -->|DISPOSE| L10 --> L23
    L9 -->|ARCHIVE| L11
    L9 -->|REVIEW| L12
    L9 -->|PERMANENT| L13
```

---

## الجداول الزمنية للمهام الخلفية

| المهمة | الوقت | المدة المتوقعة |
|--------|-------|----------------|
| النسخ الاحتياطي | 02:00 يومياً | ~2-5 دقائق |
| فحص الاحتفاظ | 04:00 يومياً | ~1-3 دقائق |
| تصعيد سير العمل | كل ساعة | ~30 ثانية |
| تقارير الأداء | الأحد 00:00 | ~5-10 دقائق |

---

## الـ 7 Modules المقترحة

| # | الـ Module | المحتوى | التبعيات |
|---|-----------|---------|---------|
| 1 | **Core** | User, Department, Organization, Settings | — |
| 2 | **Security** | Auth, MFA, RBAC, Rate Limit | Core |
| 3 | **Documents** | Document, Version, Signature, Forward | Core + Security |
| 4 | **Filing** | Cabinet, Folder, Physical, Category | Core + Documents |
| 5 | **Workflow** | Workflow, Escalation, Notification, Retention | Documents + Core |
| 6 | **Search** | FTS, OCR, AI, Filters | Documents |
| 7 | **Operations** | Audit, Backup, Analytics | All |
