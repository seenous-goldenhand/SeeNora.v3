# تصمیم‌های معماری

## انتخاب پیشنهادی برای MVP

| بخش | انتخاب |
|---|---|
| مرجع Tree و History | Desktop Service |
| Android | Offline-first با Room/SQLite |
| دیتابیس Desktop/Site | PostgreSQL؛ SQLite فقط برای نصب کاملاً تک‌ماشین |
| Raw Signal | فایل مدیریت‌شده با SHA-256 |
| Tree Transfer | HTTPS Pull از Android + USB fallback |
| Measurement Transfer | HTTPS Push از Android + USB fallback |
| Sync | UUID + Revision + Upsert + Tombstone |
| Conflict | یک نویسنده Tree؛ بدون Vector Clock در v3 |
| ساختار برنامه | Modular Monolith، نه Microserviceهای متعدد |
| استقرار ویندوز تک‌PC | Installer و Windows Service |

## چرا PostgreSQL؟

Tree، Spec، Tag یکتا، Measurement، Receipt و Audit روابط و Constraintهای مشخص دارند. PostgreSQL این قواعد را مستقیم‌تر از دیتابیس‌های query-driven مانند ScyllaDB اجرا می‌کند.

ScyllaDB زمانی بررسی شود که نرخ نوشتن یا حجم Query واقعاً بسیار بزرگ شده باشد. در آن مرحله نیز می‌تواند Read Store یا Measurement Store ثانویه باشد و PostgreSQL مرجع اصلی باقی بماند.

## چه زمانی معماری عوض شود؟

| نشانه | تغییر پیشنهادی |
|---|---|
| چند Desktop هم‌زمان به داده مشترک نیاز دارند | انتقال مرجع به Site Server |
| Desktop معمولاً خاموش است | Site Server همیشه‌روشن |
| ارسال از اینترنت همراه لازم است | افزودن Cloud Relay |
| Tree بسیار بزرگ و Full Sync کند است | Delta Sync |
| حجم Trend از PostgreSQL عبور می‌کند | Read Model یا Store تحلیلی جدا |
| چند Desktop آفلاین Tree را ویرایش می‌کنند | طراحی چندمرجعی و Conflict UI |

## مواردی که فعلاً لازم نیست

- Kubernetes
- Kafka
- Event Sourcing کامل
- Vector Clock
- Graph Database
- ذخیره Raw داخل PostgreSQL
- Microservice جدا برای هر قابلیت

این ابزارها بد نیستند، ولی نیاز فعلی محصول هزینه آن‌ها را توجیه نمی‌کند.
