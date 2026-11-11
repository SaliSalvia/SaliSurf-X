# استقرار بک‌اند SaliSurf-X در رانفلر (Runflare)

راهنمای کامل فارسی برای دیپلوی `backend/` روی پلتفرم ابری رانفلر.
منبع رسمی: https://docs.runflare.com (بخش FastAPI).

---

## گام ۰ — پیش‌نیازها

- حساب رانفلر (ایمیل + شماره موبایل، تأیید‌شده) → https://runflare.com
- فایل‌های بک‌اند (همین ریپو، پوشه `backend/`) شامل:
  - `requirements.lock` ← وابستگی‌های کاملاً قفل‌شده با hash (fastapi، uvicorn، numpy، …)؛ `requirements.txt` هم به همان اشاره می‌کند
  - `requirements-build.lock` ← setuptools قفل‌شده برای ساخت wheel (بدون حل وابستگی در زمان بیلد)
  - `Dockerfile` ← به `$PORT` احترام می‌گذارد + health check داخلی
  - پکیج `subsurface_api/` (شامل ماژول پردازش عمیق `deep/`)

---

## روش ۱ — دیپلوی با CLI (سریع‌ترین راه)

```bash
# نصب CLI (لینوکس/مک/Termux بدون proot)
/bin/bash -c "$(curl -fsSL https://get.runflare.com/install.sh)"

# ورود به حساب
runflare login

# از داخل پوشه backend (مسیر دیپلوی همان محتوای این پوشه است)
cd backend
runflare deploy          # در پایان سوال y/n لاگ زنده می‌پرسد
# دفعه‌های بعد بدون سوال:
# runflare deploy -y
```

## روش ۲ — دیپلوی از GitHub (پیشنهادی برای CI/CD)

1. وارد پنل رانفلر شوید → **مدیریت پروژه** → پروژه مورد نظر.
2. در صفحه جزئیات سرویس، روی **«اتصال به مخزن گیت»** کلیک کنید.
3. گزینه **«گیت هاب oAuth app»** → **«افزودن حساب»** → در GitHub روی
   **Authorize runflare cloud** → انتخاب دسترسی (همه ریپوها یا فقط
   `SaliSalvia/Sali-Subsurf`) → **Install & Authorize**.
4. برگردید به رانفلر → دوباره **oAuth app** → دکمه **اتصال** →
   ریپو و برنچ (`main`) را انتخاب و **ذخیره** کنید.
5. جلوی کامیت موردنظر **revert** بزنید تا همان کامیت مستقر شود.
6. تیک **«دریافت خودکار» (auto pull)** را بزنید — از این پس هر
   `git push` به‌صورت خودکار روی رانفلر مستقر می‌شود.

> روش کلاسیک (توکن): GitHub → Settings → Developer settings →
> Personal access tokens (classic) → فقط دسترسی `repo` → آدرس HTTPS ریپو +
> توکن را در «گیت هاب کلاسیک» رانفلر وارد کنید.

---

## تنظیمات سرویس (حیاتی)

### پورت

رانفلر روی Docker/Kubernetes است؛ در **تنظیمات شبکه** سرویس، پورت ورودی
دامنه (۸۰/۴۴۳) را به پورت برنامه داخل کانتینر وصل کنید:

| ورودی (دامنه) | پورت برنامه |
|---|---|
| 80 | 8000 (یا هر مقدار `$PORT` که رانفلر تزریق می‌کند — Dockerfile ما خودکار می‌خواند) |

دستور اجرا (اگر از نوع سرویس FastAPI/Python استفاده می‌کنید به‌جای Docker):

```
pip install -r requirements.lock && pip install -r requirements-build.lock && \
  SOURCE_DATE_EPOCH=1794182400 pip install --no-build-isolation --no-deps . && \
  uvicorn subsurface_api.main:app --host 0.0.0.0 --port $PORT
```

### متغیرهای محیطی و Secrets

در جزئیات سرویس → **Secrets / متغیر محیطی**:

| متغیر | الزامی؟ | توضیح |
|---|---|---|
| `PORT` | خودکار | پورتی که رانفلر به برنامه می‌دهد |
| `COPERNICUS_USERNAME` | برای حالت زنده | حساب رایگان https://dataspace.copernicus.eu |
| `COPERNICUS_PASSWORD` | برای حالت زنده | رمز همان حساب |
| `COPERNICUS_CLIENT_ID` / `COPERNICUS_CLIENT_SECRET` | جایگزین بالا | OAuth Client ساخته‌شده در پروفایل CDSE |
| `DEEP_API_KEY` | اختیاری | اگر تنظیم شود همه endpointهای deep هدر `X-API-Key` می‌خواهند (مقایسه constant-time) |
| `CORS_ORIGINS` | اختیاری | پیش‌فرض `*` ؛ برای استقرار عمومی دامنه‌های مجاز را با ویرگول بگذارید |
| `DEEP_DATA_DIR` | اختیاری | مسیر ذخیره خروجی‌ها (پیش‌فرض `./deep_jobs`) |

> ⚠️ **بدون اعتبارنامه Copernicus** فقط `mode: selftest` کار می‌کند؛
> `mode: live` خطای 503 با پیام راهنما برمی‌گرداند.

### دیسک (اختیاری ولی توصیه‌شده)

خروجی‌های deep (PNG/GeoTIFF) روی دیسک محلی نوشته می‌شوند. برای ماندگاری
بین ری‌استارت‌ها، از بخش **«ساخت دیسک جدید»** یک دیسک وصل کنید و
`DEEP_DATA_DIR` را روی مسیر همان دیسک بگذارید.

---

## تست بعد از دیپلوی

```bash
BASE="https://<دامنه-سرویس-شما>"

# 1) سلامت
curl -s "$BASE/api/v1/health"

# 2) قابلیت‌ها (live_mode_available باید true شود اگر اعتبارنامه دادید)
curl -s "$BASE/api/v1/deep/capabilities"

# 3) خودآزمایی کامل پایپ‌لاین بدون نیاز به حساب (چند ثانیه)
curl -s -X POST "$BASE/api/v1/deep/selftest?grid=160"

# 4) شروع یک تحلیل واقعی
curl -s -X POST "$BASE/api/v1/deep/analyze" \
  -H 'Content-Type: application/json' \
  -d '{"polygon":[[52.50,29.55],[52.70,29.55],[52.70,29.70],[52.50,29.70]],
       "date_from":"2026-06-01","date_to":"2026-11-07",
       "grid":512,"model":"pca","mode":"live"}'
# → job_id بگیرید، بعد:
#   GET  $BASE/api/v1/deep/jobs/<job_id>          (وضعیت/درصد)
#   GET  $BASE/api/v1/deep/jobs/<job_id>/result   (گزارش)
#   GET  $BASE/api/v1/deep/jobs/<job_id>/files/overlay.png

# 5) کنسول مرورگر:  https://<دامنه>/deep-console
```

## اتصال اپ اندروید

اپ → **تنظیمات** → «آدرس API» → آدرس رانفلر را وارد کنید
(مثلاً `https://subsurf.runflare.app`) → دکمه تست اتصال.
آدرس در `SharedPreferences` ذخیره می‌شود (`SettingsStore.apiBaseUrl`).

> منابع پیشنهادی: حداقل 512MB RAM برای grid=512؛ برای grid=512 + مدل
> autoencoder بهتر است 1GB بگیرید. حالت `resolution_m=10` (کاشی‌کاری بومی)
> هم با 512MB کار می‌کند (هر کاشی ≤512px) ولی برای سرعت بهتر 1GB بگیرید.
> وابستگی جدید: `scipy` (در requirements/Dockerfile هست، کاری لازم نیست).
> هزینه ساعتی است؛ بعد از تست می‌توانید سرویس را **متوقف** کنید و بعداً
> دوباره روشن کنید (`runflare start`).
