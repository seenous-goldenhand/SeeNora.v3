# ارتباط Android و Desktop براساس PRD 03

## پیشنهاد کوتاه

برای انتقال Tree از این روش استفاده شود:

1. کاربر در Desktop یک Branch یا چند Direction را انتخاب می‌کند.
2. Desktop یک Assignment نسخه‌دار می‌سازد.
3. Android با اسکن QR به Desktop متصل می‌شود.
4. Android بسته Assignment را با HTTPS دریافت می‌کند.
5. بسته در یک تراکنش محلی Import می‌شود.
6. UUIDهای قبلی Update و UUIDهای جدید Insert می‌شوند.
7. تنظیمات محلی `enabled/order` دست‌نخورده باقی می‌مانند.

از دید کاربر انتقال «Desktop به Mobile» است، اما بهتر است اتصال شبکه را Android آغاز کند. این کار مشکل IP موبایل، Firewall و اتصال ورودی به گوشی را حذف می‌کند.

## مقایسه پروتکل‌ها

| روش | نتیجه |
|---|---|
| HTTPS REST روی LAN | پیشنهاد اصلی؛ ساده، قابل تست و مناسب Retry |
| WebSocket | فقط برای نمایش لحظه‌ای وضعیت؛ برای Tree لازم نیست |
| MQTT | برای این انتقال کوچک Broker اضافه می‌کند و ارزش کافی ندارد |
| Bluetooth | برای Data Collector مناسب‌تر است؛ Sync Desktop را پیچیده می‌کند |
| USB Package | مسیر جایگزین رسمی برای سایت بدون شبکه |
| Cloud Relay | فقط اگر انتقال از اینترنت همراه لازم باشد |

## Pairing

Desktop یک QR کوتاه‌عمر نشان می‌دهد که شامل این اطلاعات است:

```json
{
  "authority_id": "desktop-uuid",
  "address": "https://192.168.10.20:7443",
  "pairing_code": "one-time-code"
}
```

QR نباید Tree، رمز دائمی یا Raw Data داشته باشد. Android پس از Pairing یک شناسه نصب و credential محدود به همان کارخانه دریافت می‌کند.

اگر Certificate عمومی در LAN ممکن نیست، Installer باید Certificate محلی بسازد و اثرانگشت آن از طریق QR تأیید شود. خاموش‌کردن TLS validation راه‌حل قابل قبول نیست.

## API ساده نسخه اول

| درخواست | کاربرد |
|---|---|
| `GET /api/v1/capabilities` | نسخه پروتکل و قابلیت‌های Desktop |
| `POST /api/v1/pair` | ثبت موبایل با کد یک‌بارمصرف |
| `GET /api/v1/assignments` | فهرست مأموریت‌های آماده این موبایل |
| `GET /api/v1/assignments/{id}` | دریافت Tree، Spec و تنظیمات مأموریت |
| `POST /api/v1/assignments/{id}/ack` | اعلام Import موفق یا خطا |

همین Assignment در USB نیز به‌صورت فایل `.seenora-tree` Export می‌شود. شبکه و USB باید از یک Importer مشترک استفاده کنند.

## محتوای Assignment

```json
{
  "schema_version": 1,
  "assignment_id": "uuid",
  "factory_id": "uuid",
  "authority_id": "uuid",
  "tree_revision": 42,
  "created_at": "2026-10-01T08:00:00Z",
  "nodes": [
    {
      "id": "node-uuid",
      "parent_id": "parent-uuid",
      "type": "MEASUREMENT_DIRECTION",
      "name": "Horizontal",
      "revision": 7,
      "deleted_at": null
    }
  ],
  "specs": [],
  "package_sha256": "..."
}
```

تمام Parentهای لازم برای نمایش مسیر باید داخل بسته باشند، حتی اگر فقط یک Machine انتخاب شده باشد. تصاویر بزرگ می‌توانند جداگانه و اختیاری منتقل شوند.

## Import روی Android

Android ابتدا بسته را در Staging نگه می‌دارد و سپس این مراحل را انجام می‌دهد:

1. بررسی نسخه Schema، Factory، Authority و hash بسته.
2. بررسی وجود Parentها و نبود چرخه.
3. Upsert تمام Nodeها با UUID اصلی Desktop.
4. اعمال Tombstoneهای صریح.
5. ساخت یا Update مأموریت و ترتیب Directionها.
6. Commit همه تغییرات در یک تراکنش Room/SQLite.
7. ثبت `tree_revision` فقط پس از Commit موفق.

اگر Import قطع شود، نسخه فعال قبلی باقی می‌ماند. Import نیمه‌کاره نباید در UI دیده شود.

## حفظ تنظیم کاربر

تنظیم محلی در جدولی جدا و با کلید `mobile + measurement_node_id` ذخیره می‌شود:

```text
is_enabled
local_sort_order
last_measured_at
```

این ردیف با Update Node جایگزین نمی‌شود. اگر Node واقعاً حذف شده باشد، Preference می‌تواند برای مدتی باقی بماند تا Measurementهای قدیمی همچنان قابل نمایش باشند.

## Full Package یا Delta؟

برای MVP، ارسال کامل Branch انتخاب‌شده پیشنهاد می‌شود. Tree نسبت به Raw Signal کوچک است و پیاده‌سازی Full Package ساده‌تر و مطمئن‌تر است.

Delta زمانی اضافه شود که اندازه Tree یا زمان Import واقعاً مشکل ایجاد کند. Delta شامل `base_revision`، Nodeهای تغییرکرده و Tombstoneهای صریح است. نبودن یک Node در بسته به معنی حذف آن نیست.

## خطاهای مهم

| خطا | رفتار |
|---|---|
| قطع شبکه | بسته ناقص فعال نشود؛ Retry از ابتدا برای بسته کوچک |
| دریافت دوباره همان Assignment | Upsert؛ بدون Duplicate |
| نسخه قدیمی Android | پیام ناسازگاری Schema؛ Tree قبلی حفظ شود |
| فضای ناکافی | Import انجام نشود و مقدار فضای لازم نمایش داده شود |
| Parent ناموجود | کل بسته رد شود |
| Node حذف‌شده با Measurement قدیمی | Soft Delete؛ Measurement حفظ شود |
| Factory یا Authority اشتباه | بسته رد شود |

## UX پیشنهادی

Desktop:

```text
Select Branch -> Show machine/point counts -> Choose Mobile -> Create Assignment -> Show QR/status
```

Android:

```text
Scan/Connect -> Show assignment summary -> Download -> Import -> Ready
```

وضعیت‌های قابل نمایش: `Waiting`, `Downloading`, `Validating`, `Imported`, `Failed`. پیام خطا باید اقدام بعدی مثل Retry، آزادکردن فضا یا Update برنامه را بگوید.

## نتیجه پذیرش PRD 03

این طراحی تضمین می‌کند Parent/Child و UUID حفظ شوند، Branch محدود منتقل شود، Import مجدد Duplicate نسازد، Tree بعد از Restart موجود باشد، Update داده قبلی را حذف نکند و تنظیم فعال/غیرفعال تکنسین با Sync بعدی تغییر نکند.
