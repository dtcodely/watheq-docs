---
title: دليل التثبيت والتشغيل
---

# دليل التثبيت والتشغيل (Deployment Guide)

يرشدك هذا الدليل إلى كيفية تشغيل النظام محلياً لأغراض التطوير، وكيفية إعداد البيئة للإنتاج.

![اقتراح صورة: رسم توضيحي يحتوي على شعارات أدوات التشغيل مثل Docker، Node.js، و PM2]

## المتطلبات الأساسية

لضمان تشغيل النظام دون مشاكل، يجب أن تتوفر البرمجيات التالية على خادمك:
- **Node.js** بإصدار 18 فما فوق.
- **Docker Desktop** (أو Docker Engine) لتشغيل قاعدة البيانات، الخبيئة (Redis)، وإدارة القواعد (pgAdmin).
- **npm** بإصدار 9 فما فوق.

---

## 1. تشغيل الخدمات الأساسية (Infrastructure) عبر Docker

يحتوي مجلد المشروع على ملف `docker-compose.yml` يجمع خدمات البنية التحتية المطلوبة لتشغيل النظام.

```bash
# تشغيل كل من (قاعدة البيانات PostgreSQL، رسائل مهام Redis، واجهة إدارة pgAdmin)
docker-compose up -d

# التحقق من حالة الحاويات النشطة
docker-compose ps
```

### تفاصيل الخدمات الافتراضية:
| الخدمة | المنفذ (Port) | بيانات الدخول الافتراضية |
|---|---|---|
| **PostgreSQL** | 5454 | user: postgres / pass: postgres |
| **Redis** | 6379 | — |
| **pgAdmin** | 5051 | admin@watheq.com / admin |

---

## 2. تشغيل الواجهة الخلفية والتحكم (Backend)

بمجرد عمل خدمات Docker، قم بالانتقال لمجلد الخادم لتثبيت الحزم وتشغيله:

```bash
cd backend
npm install
npx prisma generate
npx prisma migrate deploy
npm run dev
```

> **ملاحظة:** سيبدأ الخادم العمل بشكل افتراضي على الرابط `http://localhost:3000`.

---

## 3. تشغيل واجهة المستخدم (Web Frontend)

لتشغيل واجهة المتصفح الأمامية المبنية بـ React و Vite:

```bash
cd web
npm install
npm run dev
```

> **ملاحظة:** ستعمل الواجهة بشكل افتراضي وتفتح المتصفح على الرابط `http://localhost:5173`.

---

## 4. متغيرات البيئة الأساسية (`backend/.env`)

تأكد من وجود ملف `.env` في مجلد الـ `backend` يحتوي على الأقل على القيم الآتية لربط الخدمات:
```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5454/watheq"
JWT_SECRET="your-secret-key"
REDIS_URL="redis://localhost:6379"
NODE_ENV="development"
PORT=3000
```

---

## 5. خطوة ضرورية: تطبيق فهرست GIN للبحث (المرحلة الخامسة)

نظراً لأن نظام البحث المتقدم يعتمد على مؤشرات خاصة بالـ PostgreSQL لا يدعمها Prisma محلياً بشكل مباشر، يجب تطبيق ملف الـ SQL الخاص بها يدوياً في أول تشغيل، أو بعد مسح قاعدة البيانات.

```bash
# 1. نسخ ملف الـ SQL الخاص بتسريع البحث من بيئة التطوير إلى حاوية الدوكر
docker cp "prisma\migrations\20260509225603_add_gin_index_text_content\migration.sql" watheq-db:/tmp/gin_index.sql

# 2. تنفيذ الأوامر مباشرة داخل قواعد بيانات الحاوية لتبني الفهارس
docker exec watheq-db psql -U postgres -d watheq -f /tmp/gin_index.sql
```

![اقتراح صورة: نافذة الطرفية (Terminal) أو واجهة pgAdmin توضح الأوامر أو الفهارس المضافة]

---

## 6. الوصول إلى مدير قواعد البيانات (pgAdmin)

إذا كنت ترغب بإدارة الجداول واكتشاف البيانات بصرياً:
- **الرابط:** `http://localhost:5051`
- **البريد:** `admin@watheq.com`
- **كلمة المرور:** `admin`

لإضافة خادم الاتصال (Server Setup) في pgAdmin، استخدم القيم: `Host: watheq-db`، `Port: 5432`، `DB: watheq`.

---

## 7. البناء والإعداد لبيئة الإنتاج (Production)

لكي يعمل النظام على خوادم الإنتاج بشكل عالي الأداء:

```bash
# 1. بناء ملفات واجهة المستخدم (تنتج مجلد dist سيتم تقديمه عبر Nginx مثلاً)
cd web && npm run build

# 2. بناء ملفات الخادم المترجمة لـ Javascript
cd backend && npm run build

# 3. تشغيل الخادم بشكل دائم باستخدام مدير المهام PM2
pm2 start dist/server.js --name watheq-backend
```

---

## 8. تشغيل موقع التوثيق الداخلي (MkDocs) الحالي

تم وضع هذا التوثيق التفاعلي لمساعدة المطورين ومدراء التقنية، وتم بناؤه بـ `MkDocs` والمظهر المطور `Material for MkDocs`.

```bash
# العمل من المجلد الجذري للمشروع
cd docs-site
.\venv\Scripts\mkdocs serve
```
بعدها سيعمل التوثيق (العربي والإنجليزي) على الرابط المحلي: `http://localhost:8000`.
