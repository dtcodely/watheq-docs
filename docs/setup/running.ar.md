# طريقة التشغيل

لتشغيل نظام **واثق (Watheq)** في بيئة التطوير المحلية (Local Development)، يرجى تباع الخطوات التالية بصلاحيات موجه الأوامر (Terminal).

## 1. تشغيل قاعدة البيانات
قم بالدخول إلى المجلد الرئيسي للمشروع وتشغيل حاوية قاعدة البيانات عبر Docker:
```bash
docker-compose up -d pgadmin
```

## 2. تشغيل الخادم الخلفي (Backend)
انتقل إلى مجلد الـ `backend` وثبّت الحزم ثم ابدأ التشغيل:
```bash
cd backend
npm install
npx prisma generate
npx prisma db push
npm run dev
```

## 3. تشغيل الواجهة (Frontend)
افتح نافذة Terminal جديدة وانتقل لمجلد الـ `web`:
```bash
cd web
npm install
npm run dev
```

وبعدها ستحصل على رابط الدخول (غالباً ما يكون `http://localhost:5173`).

![لقطة شاشة النظام](assets/placeholder.png)
*(سيتم استبدال الصورة بلقطة من واجهة الدخول لاحقاً)*
