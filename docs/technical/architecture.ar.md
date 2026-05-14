---
title: معمارية النظام
---

# معمارية النظام (System Architecture)

يعتمد نظام واثق على هندسة برمجية حديثة تتبنى فصل الواجهات (Client Layer) عن طبقة المعالجة (API Layer) لضمان الأمان، قابلية التوسع، وسلاسة التطوير.

!!! info "روابط سريعة"
    - 📊 [المخططات التشغيلية الكاملة — 10 مراحل](flowchart.md)
    - 🔁 [مخططات التسلسل — 15 مخطط](sequences.md)
    - 🗄️ [مخطط ERD التفصيلي](erd.md)


## المكونات الرئيسية

يتكون المشروع من الهيكلية التالية:

```txt
WATHEQ/
├── backend/          # Node.js + Express + Prisma ORM
├── web/              # React + Vite + TypeScript + TailwindCSS
├── mobile/           # React Native (Expo)
├── desktop/          # Electron
├── database/         # PostgreSQL (عبر Docker)
└── docs-site/        # التوثيق الشامل (MkDocs)
```

## المصفوفة التقنية (Tech Stack)

### خادم المعالجة (Backend)
| التقنية | الدور |
|---|---|
| **Node.js + Express** | خادم API REST |
| **Prisma ORM** | التواصل مع قاعدة البيانات |
| **PostgreSQL** | قاعدة البيانات الرئيسية |
| **Redis + BullMQ** | طابور المهام الخلفية (OCR، فهرسة البحث) |
| **JWT** | المصادقة وإدارة الجلسات |
| **Multer** | رفع الملفات |
| **Tesseract.js** | التعرف البصري على الحروف (OCR) للصور |
| **pdf-parse / mammoth** | استخراج النص من PDF و DOCX |
| **Socket.io** | الإشعارات الفورية |
| **bcryptjs** | تشفير كلمات المرور |

### واجهة المستخدم (Frontend - Web)
| التقنية | الدور |
|---|---|
| **React 18 + TypeScript** | إطار عمل الواجهة |
| **Vite** | بناء وتطوير الواجهة |
| **TailwindCSS** | التصميم والتنسيق المرن |
| **react-i18next** | التوطين ودعم اللغات (AR/EN) |
| **lucide-react** | أيقونات النظام |



## مخطط المعمارية وتدفق البيانات

```txt
┌─────────────────────────────────────────────────┐
│                   Client Layer                  │
│  Web (React/Vite)  │  Mobile (Expo)  │ Desktop  │
└────────────────────┬────────────────────────────┘
                     │ HTTP REST + WebSocket
┌────────────────────▼────────────────────────────┐
│              API Layer (Express.js)             │
│  Auth  │  Documents  │  Users  │  Reports  ...  │
└───────────┬─────────────────────────┬───────────┘
            │ Prisma ORM              │ BullMQ
┌───────────▼──────────┐  ┌──────────▼───────────┐
│  PostgreSQL Database │  │   Redis (Job Queue)  │
│  + GIN Indexes       │  │   + Document Worker  │
└──────────────────────┘  └──────────────────────┘
```

## هيكلة مجلد الخادم (Backend src/)

```txt
src/
├── controllers/        # نقاط النهاية (Endpoints) ومعالجة طلبات HTTP
├── services/           # منطق الأعمال (Business Logic)
│   ├── textExtraction.service.ts
│   ├── disposalCertificate.service.ts
│   └── ...
├── middleware/         # وسطاء الطلبات (Auth, Upload)
├── queues/             # طوابير BullMQ للمهام الثقيلة
├── workers/            # معالجات الطوابير في الخلفية
├── routes/             # تعريف المسارات وتوجيهها للـ Controllers
├── utils/              # أدوات مساعدة (Prisma, Redis, AI)
└── prisma/             # مخططات القاعدة (Schema & Migrations)
```
