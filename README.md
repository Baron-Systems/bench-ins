# تشغيل Frappe/ERPNext مع Cloudflare Tunnel

## 1) تجهيز ملف `.env`

افتح `.env` وعدّل:

```bash
# كلمة مرور قوية لقاعدة البيانات
DB_ROOT_PASSWORD=كلمة_مرور_قوية_هنا

# Token من Cloudflare (تحصل عليه من الخطوة التالية)
CLOUDFLARE_TUNNEL_TOKEN=الـ_Token_الحقيقي
```

## 2) إنشاء Cloudflare Tunnel والحصول على الـ Token

1. ادخل إلى [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. من القائمة: **Networks** → **Tunnels** → **Create a tunnel**
3. اختر **Cloudflared**
4. سمّ التunnel (مثلاً `frappe-prod`) ثم **Save tunnel**
5. في **Public Hostname**:
   - **Subdomain**: مثلاً `erp` أو اتركه فارغاً للنطاق الرئيسي
   - **Domain**: اختر نطاقك (مثلاً `company.com`)
   - **Service type**: `HTTP`
   - **URL**: اكتب `frontend:80` (لأن الـ tunnel يتصل بحاوية الـ frontend على البورت 80)
6. انسخ **Install and run** → اختر **Docker** وانسخ الـ **Tunnel token** (يبدأ عادة بـ `eyJ...`)
7. الصق هذا الـ token في `.env` في السطر:
   ```bash
   CLOUDFLARE_TUNNEL_TOKEN=eyJ...
   ```

## 3) تشغيل الحاويات

من مجلد المشروع (حيث يوجد `docker-compose.yml` و`.env`):

```bash
cd /home/manager-pc/Desktop/bench-ins
docker compose up -d
```

انتظر حتى تكتمل جميع الخدمات (خاصة `configurator` ثم `backend` و`frontend`). يمكنك متابعة السجلات:

```bash
docker compose logs -f
```

للمتابعة حتى يهدأ التشغيل اضغط `Ctrl+C` ثم استمر بالخطوة التالية.

## 4) إنشاء موقع (Site) في Frappe

هذا الإعداد **بدون Sites** مسبقة، لذلك تحتاج إنشاء الموقع يدوياً مرة واحدة:

```bash
docker compose exec backend bench new-site SITE_NAME --no-mount
```

- استبدل `SITE_NAME` باسم الموقع، ويجب أن يطابق الـ host الذي سيدخل منه المستخدم، مثلاً:
  - إذا كان الرابط سيكون `erp.company.com` فاستخدم: `erp.company.com`
  - أو إذا كان `company.com` فاستخدم: `company.com`

مثال:

```bash
docker compose exec backend bench new-site erp.company.com --no-mount
```

سيُطلب منك كلمة مرور لـ Administrator. اخترها واحفظها.

ثم ربط تطبيق ERPNext بالموقع (إن كنت تستخدم صورة ERPNext):

```bash
docker compose exec backend bench --site erp.company.com install-app erpnext
```

## 5) الدخول عبر المتصفح

افتح الرابط الذي حددته في Cloudflare، مثلاً:

- `https://erp.company.com`

يجب أن تظهر واجهة تسجيل الدخول لـ Frappe/ERPNext.

---

## تثبيت تطبيق (انزال تطبيق)

### تطبيق موجود في الصورة (مثل ERPNext)

التطبيق موجود في الصورة، تحتاج فقط ربطه بالموقع:

```bash
docker compose exec backend bench --site SITE_NAME install-app اسم_التطبيق
```

مثال (تطبيق الدفع payments):

```bash
docker compose exec backend bench --site erp.company.com install-app payments
```

### انزال تطبيق جديد من الإنترنت (get-app)

1. **إنزال التطبيق** داخل حاوية الـ backend:

```bash
docker compose exec backend bash -lc "cd /home/frappe/frappe-bench && bench get-app رابط_المستودع"
```

مثال (تطبيق من GitHub):

```bash
docker compose exec backend bash -lc "cd /home/frappe/frappe-bench && bench get-app https://github.com/frappe/wiki"
```

2. **تثبيت التطبيق على الموقع**:

```bash
docker compose exec backend bench --site SITE_NAME install-app اسم_التطبيق
```

مثال:

```bash
docker compose exec backend bench --site erp.company.com install-app wiki
```

3. **إعادة تشغيل الخدمات** حتى يقرأ الـ frontend التطبيق الجديد:

```bash
docker compose restart backend frontend websocket
```

**ملاحظة:** التطبيقات التي تنزلها بـ `get-app` داخل الحاوية قد لا تبقى بعد إعادة إنشاء الحاوية (`docker compose down` ثم `up`) لأن مجلد `apps` غير مخزّن على volume. إذا أردت إبقاءها دائمياً إما تضيف volume لـ `apps` في الـ compose أو تبني صورة Docker مخصصة تحتوي التطبيق.

---

## من وين أشغّل الأوامر؟

**كل أوامر الـ bench تشغّلها من حاوية الـ backend فقط:**

- `new-site`، `install-app`، `get-app`، `migrate`، `clear-cache`، إلخ → كلها داخل **backend**.

حاويات **frontend** و **queue-short** و **queue-long** و **scheduler** و **websocket** تشتغل تلقائياً على نفس الـ sites والـ apps (عبر الـ volumes المشتركة)، ما تحتاج تشغّل فيها أي أوامر يدوية.

```bash
# الصيغة العامة
docker compose exec backend bench ...
docker compose exec backend bench --site SITE_NAME ...
```

---

## أوامر مفيدة

| الأمر | الوصف |
|--------|--------|
| `docker compose ps` | عرض حالة الخدمات |
| `docker compose logs -f frontend` | متابعة سجلات الـ frontend |
| `docker compose logs -f cloudflared` | متابعة سجلات الـ tunnel |
| `docker compose down` | إيقاف كل الحاويات |
| `docker compose up -d` | تشغيل الخدمات من جديد |

---

## ملاحظات

- **FRAPPE_SITE_NAME_HEADER**: الإعداد الحالي `$$host` يجعل Frappe يحدد الموقع تلقائياً من عنوان الطلب (Host). تأكد أن اسم الـ Site الذي أنشأته يطابق الـ host (مثل `erp.company.com`).
- لا تحتاج فتح أي port على السيرفر؛ كل الاتصال يتم عبر Cloudflare Tunnel.
- احفظ نسخة من `.env` في مكان آمن ولا ترفعها إلى Git (عادةً `.env` مضاف في `.gitignore`).
