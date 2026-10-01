
---

## ۱. `factories` — کارخانه

**مسئولیت:** مرز داده (Data Boundary) و نسخه‌بندی درخت تجهیزات یک کارخانه.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `name` | varchar(200) | نام کارخانه |
| `code` | varchar(80) | کد یکتای کارخانه |
| `authority_id` | uuid | شناسه Desktop/Site فعلی که مرجع است |
| `authority_epoch` | bigint | شماره دوره مرجعیت (برای حل تعارض) |
| `tree_revision` | bigint | نسخه درخت تجهیزات (هر تغییر +۱) |
| `created_at` / `updated_at` | timestamptz | زمان ایجاد و به‌روزرسانی |

**نکته:** `authority_id` و `authority_epoch` برای همگام‌سازی بین Mobile و Desktop استفاده می‌شوند تا مشخص شود کدام نسخه معتبر است.

---

## ۲. `asset_nodes` — گره‌های درخت تجهیزات

**مسئولیت:** نمایش کل سلسله‌مراتب ثابت کارخانه در **یک جدول خودارجاع (Self-Referencing)**.

ساختار درخت:
```
LINE → DEPARTMENT → MACHINE → MACHINE_PART → MEASUREMENT_POINT → MEASUREMENT_DIRECTION
```

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `parent_id` | uuid | ارجاع به گره والد (خودارجاع) |
| `node_type` | enum | نوع گره (LINE, DEPARTMENT, ...) |
| `name` | varchar(200) | نام گره |
| `code` | varchar(80) | کد اختیاری |
| `sort_order` | int | ترتیب نمایش بین خواهر/برادرها |
| `is_active` | boolean | فعال/غیرفعال |
| `revision` | bigint | نسخه گره |
| `deleted_at` | timestamptz | حذف نرم (Soft Delete) |

**ایندکس‌ها:**
- `(factory_id, parent_id, sort_order)` → واکشی فرزندان یک گره
- `(factory_id, node_type)` → فیلتر بر اساس نوع
- `(parent_id, node_type, name)` یکتا → جلوگیری از تکرار نام در یک والد

**نکته مهم:** `MEASUREMENT_DIRECTION` همان **Measurement Node پایدار** است. UUID آن بین Desktop و Android تغییر نمی‌کند و مبنای ارجاع Signalها است.

**قواعدی که باید با Trigger کنترل شوند:**
- ترتیب مجاز Parent/Child
- جلوگیری از Cycle
- گره‌های دارای History فقط Soft Delete می‌شوند

---

## ۳. `machines` — ماشین‌ها

**مسئولیت:** نگهداری اطلاعات قابل جست‌وجو و پرتکرار ماشین به‌صورت ستون‌های رابطه‌ای.

| ستون | نوع | توضیح |
|---|---|---|
| `node_id` | uuid | کلید اصلی + ارجاع به `asset_nodes` (باید از نوع MACHINE باشد) |
| `factory_id` | uuid | ارجاع به کارخانه |
| `tag` | varchar(120) | تگ ماشین (یکتا در هر کارخانه) |
| `priority` | smallint | اولویت |
| `power_kw` | decimal | توان |
| `foundation_type` | varchar | نوع فونداسیون |
| `machine_group` | varchar | گروه ماشین |
| `speed_mode` | enum | FIXED / VARIABLE / UNKNOWN |
| `nominal_rpm` | decimal | دور نامی |
| `default_collection_duration_ms` | int | مدت پیش‌فرض جمع‌آوری |
| `image_uri` | text | آدرس تصویر |
| `technical_data` | jsonb | اطلاعات فنی متغیر |
| `notes` | text | یادداشت |

**ایندکس‌ها:**
- `(factory_id, tag)` یکتا
- `(machine_group, priority)`

**نکته:** `points_count` مشتق‌شده است و ذخیره نمی‌شود.

---

## ۴. `asset_spec_revisions` — تاریخچه مشخصات فنی

**مسئولیت:** نگهداری تاریخچه Spec ماشین یا Measurement Point به‌صورت نسخه‌بندی‌شده.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `asset_node_id` | uuid | ارجاع به گره |
| `revision_no` | int | شماره نسخه |
| `spec_data` | jsonb | اطلاعات فنی (یاتاقان، RPM، دندانه چرخ‌دنده، ...) |
| `source` | varchar | منبع (پیش‌فرض MANUAL) |
| `effective_from` | timestamptz | شروع اعتبار |
| `effective_to` | timestamptz | پایان اعتبار |

**ایندکس‌ها:**
- `(asset_node_id, revision_no)` یکتا
- `(asset_node_id, effective_from)`

**نکته:** فقط **یک Spec Revision باز (Open-Ended)** برای هر گره مجاز است. Revision قبلی حذف نمی‌شود تا Measurement تاریخی قابل تفسیر بماند.

---

## ۵. `asset_templates` — قالب‌های تجهیزات

**مسئولیت:** قالب‌سازی برای ساخت سریع Machine/Point/Branch.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `name` | varchar(200) | نام قالب |
| `template_type` | varchar(40) | MACHINE / POINT / BRANCH |
| `template_data` | jsonb | داده قالب |
| `version` | int | نسخه |
| `is_active` | boolean | فعال |

**ایندکس:** `(factory_id, name, version)` یکتا

---

## ۶. `devices` — دستگاه‌ها

**مسئولیت:** ثبت Mobile و Data Collector.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `device_type` | enum | MOBILE / DATA_COLLECTOR |
| `device_key` | varchar(160) | شناسه نصب Mobile یا سریال Collector |
| `display_name` | varchar | نام نمایشی |
| `axis_count` | smallint | تعداد محور (۱ یا ۳) |
| `app_or_firmware_version` | varchar | نسخه نرم‌افزار/فریمور |
| `calibration_revision` | varchar | نسخه کالیبراسیون |
| `capabilities` | jsonb | قابلیت‌ها |
| `last_seen_at` | timestamptz | آخرین مشاهده |
| `revoked_at` | timestamptz | زمان ابطال |

**ایندکس:** `(factory_id, device_type, device_key)` یکتا

**نکته:** `axis_count` فقط ۱ یا ۳ است. نقش دستگاه باید با Foreign Key استفاده‌شده مطابقت داشته باشد.

---

## ۷. `collection_assignments` — مأموریت‌های جمع‌آوری

**مسئولیت:** مأموریتی که به Mobile ارسال می‌شود.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `mobile_device_id` | uuid | ارجاع به دستگاه Mobile |
| `name` | varchar(200) | نام مأموریت |
| `tree_revision` | bigint | نسخه درخت در زمان ایجاد |
| `status` | varchar(40) | پیش‌فرض ACTIVE |
| `created_at` | timestamptz | زمان ایجاد |
| `completed_at` | timestamptz | زمان تکمیل |

**ایندکس:** `(mobile_device_id, status)`

**نکته:** `mobile_device_id` باید به یک دستگاه از نوع MOBILE اشاره کند.

---

## ۸. `assignment_nodes` — گره‌های مأموریت

**مسئولیت:** Directionهای موجود در یک مأموریت و ترتیب آن‌ها.

| ستون | نوع | توضیح |
|---|---|---|
| `assignment_id` | uuid | ارجاع به مأموریت |
| `measurement_node_id` | uuid | ارجاع به گره Direction |
| `sort_order` | int | ترتیب |
| `is_enabled` | boolean | فعال/غیرفعال |

**کلید اصلی مرکب:** `(assignment_id, measurement_node_id)`
**ایندکس:** `(assignment_id, sort_order)`

**نکته:** `measurement_node_id` باید به یک گره `MEASUREMENT_DIRECTION` اشاره کند.

---

## ۹. `measurements` — اندازه‌گیری‌ها

**مسئولیت:** اطلاعات اصلی هر برداشت (Start → Finish).

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `assignment_id` | uuid | ارجاع به مأموریت |
| `measurement_point_id` | uuid | ارجاع به Measurement Point |
| `mobile_device_id` | uuid | دستگاه Mobile |
| `data_collector_id` | uuid | Data Collector |
| `mode` | enum | ROOT / OFF_ROOT_FREE |
| `measured_at` | timestamptz | زمان اندازه‌گیری |
| `received_at` | timestamptz | زمان دریافت |
| `actual_rpm` | decimal | دور واقعی |
| `sample_rate_hz` | int | نرخ نمونه‌برداری |
| `expected_sample_count` | int | تعداد نمونه مورد انتظار |
| `duration_ms` | int | مدت |
| `acquisition_settings` | jsonb | تنظیمات جمع‌آوری |
| `off_root_machine_tag` | varchar | تگ ماشین در حالت Off-Root |
| `spec_snapshot` | jsonb | تصویر Spec در زمان جمع‌آوری (تغییرناپذیر) |
| `context_hash` | char(64) | هش زمینه |
| `manifest_hash` | char(64) | هش محتوا |
| `quality` | enum | PENDING / VALID / INCOMPLETE / INVALID |
| `quality_details` | jsonb | جزئیات کیفیت |
| `transfer_status` | enum | PENDING / SELECTED / UPLOADING / FAILED / COMMITTED |
| `processing_status` | enum | PENDING / RUNNING / READY / FAILED |
| `temperature_value` | decimal | دمای اندازه‌گیری‌شده |
| `temperature_unit` | varchar | پیش‌فرض C |
| `temperature_status` | enum | VALID / UNAVAILABLE / INVALID / OUT_OF_RANGE |

**ایندکس‌ها:**
- `(factory_id, id)` یکتا
- `(measurement_point_id, measured_at)` → Trend
- `(mobile_device_id, transfer_status)` → صف ارسال
- `(data_collector_id, measured_at)`

**نکات مهم:**
- حالت `OFF_ROOT_LIVE` هرگز ذخیره نمی‌شود (Ring Buffer پس از Live View دور ریخته می‌شود).
- حالت `ROOT` نیازمند Point و Direction نگاشت‌شده است.
- حالت `OFF_ROOT` بدون نگاشت خودکار باقی می‌ماند مگر کاربر صریحاً نگاشت کند.

---

## ۱۰. `measurement_signals` — سیگنال‌های اندازه‌گیری

**مسئولیت:** یک ردیف برای هر کانال فیزیکی X/Y/Z.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `measurement_id` | uuid | ارجاع به اندازه‌گیری |
| `sensor_axis` | varchar(8) | محور سنسور |
| `measurement_node_id` | uuid | نگاشت به Direction (در حالت Root) |
| `storage_key` | text | آدرس فایل Raw (یکتا) |
| `sha256` | char(64) | هش فایل |
| `byte_size` | bigint | اندازه فایل |
| `encoding` | varchar(60) | مثال: float32-le |
| `compression` | varchar(40) | فشرده‌سازی |
| `sample_count` | int | تعداد نمونه |
| `acceleration_unit` | varchar(20) | پیش‌فرض m/s2 |
| `scale_factor` | decimal | ضریب مقیاس |
| `quality` | enum | VALID / MISSING / CLIPPED / CORRUPTED / OUT_OF_RANGE |
| `quality_details` | jsonb | جزئیات |

**ایندکس‌ها:**
- `(measurement_id, sensor_axis)` یکتا
- `(measurement_node_id, measurement_id)` → History
- `sha256`

**نکته:** داده خام (Raw Bytes) در دیتابیس ذخیره نمی‌شود؛ فقط متادیتا و آدرس فایل.

---

## ۱۱. `import_receipts` — رسیدهای ثبت

**مسئولیت:** تأیید ثبت بادوام (Durable) در Desktop.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `measurement_id` | uuid | ارجاع یک‌به‌یک به اندازه‌گیری |
| `authority_id` | uuid | شناسه مرجع |
| `authority_epoch` | bigint | دوره مرجعیت |
| `manifest_hash` | char(64) | هش محتوا |
| `commit_sequence` | bigint | شماره ترتیب ثبت |
| `committed_at` | timestamptz | زمان ثبت |
| `signature` | text | امضا |

**ایندکس:** `(authority_id, authority_epoch, commit_sequence)` یکتا

**نکته حیاتی:** فقط این رسید به Mobile اجازه می‌دهد داده Root را پاک کند. تا قبل از دریافت Receipt، Mobile نباید داده را حذف کند.

---

## ۱۲. `derived_features` — ویژگی‌های مشتق‌شده

**مسئولیت:** خروجی DSP مانند RMS، P2P، Crest، Kurtosis، Overall.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `measurement_signal_id` | uuid | ارجاع به سیگنال |
| `algorithm_name` | varchar(120) | نام الگوریتم |
| `algorithm_version` | varchar(80) | نسخه الگوریتم |
| `parameters_hash` | char(64) | هش پارامترها |
| `input_hash` | char(64) | هش ورودی |
| `feature_values` | jsonb | مقادیر ویژگی |
| `created_at` | timestamptz | زمان ایجاد |

**ایندکس یکتا:** `(measurement_signal_id, algorithm_name, algorithm_version, parameters_hash)`

---

## ۱۳. `threshold_rules` — قوانین آستانه

**مسئولیت:** قوانین ISO، دستی، Trend، Historical، Similar Machine.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `target_node_id` | uuid | گره هدف |
| `kind` | enum | ISO / USER_DEFINED / TREND_DEVIATION / HISTORICAL / SIMILAR_MACHINE |
| `name` | varchar(200) | نام قانون |
| `ruleset_version` | varchar(80) | نسخه Ruleset |
| `rule_config` | jsonb | پیکربندی |
| `is_active` | boolean | فعال |
| `effective_from` / `effective_to` | timestamptz | بازه اعتبار |

**ایندکس‌ها:**
- `(factory_id, kind, is_active)`
- `(target_node_id, is_active)`

---

## ۱۴. `screening_results` — نتایج Screening

**مسئولیت:** نتیجه و دلیل Screening هر اندازه‌گیری.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `measurement_id` | uuid | ارجاع به اندازه‌گیری |
| `measurement_node_id` | uuid | گره اندازه‌گیری |
| `ruleset_version` | varchar(80) | نسخه Ruleset |
| `context_hash` | char(64) | هش زمینه |
| `severity` | enum | NORMAL / WARNING / CRITICAL / INSUFFICIENT_DATA |
| `reasons` | jsonb | دلایل |
| `created_at` | timestamptz | زمان ایجاد |

**ایندکس‌ها:**
- `(measurement_id, measurement_node_id, ruleset_version, context_hash)` یکتا
- `(measurement_node_id, created_at)` → History
- `(severity, created_at)` → Alertهای اخیر

---

## ۱۵. `audit_events` — رویدادهای حسابرسی

**مسئولیت:** سابقه تغییرات حساس.

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `actor_id` | uuid | شناسه کاربر |
| `entity_type` | varchar(80) | نوع موجودیت |
| `entity_id` | uuid | شناسه موجودیت |
| `operation` | varchar(80) | عملیات |
| `revision` | bigint | نسخه |
| `change_data` | jsonb | داده تغییر |
| `reason` | text | دلیل |
| `occurred_at` | timestamptz | زمان وقوع |

**ایندکس:** `(factory_id, entity_type, entity_id, occurred_at)`

---

## سه وضعیت مستقل (Very Important)

| وضعیت | مقادیر | کاربرد |
|---|---|---|
| **Quality** | VALID / INCOMPLETE / INVALID | سلامت برداشت |
| **Transfer** | PENDING / COMMITTED / FAILED | تحویل به Desktop |
| **Processing** | PENDING / READY / FAILED | آماده‌بودن Featureها |

ممکن است داده `VALID` و `COMMITTED` باشد ولی پردازش آن `FAILED` شود. در این حالت Raw امن است و فقط DSP دوباره اجرا می‌شود.

---

## جلوگیری از Duplicate (Manifest Hash)

Measurement با UUID شناخته می‌شود و `manifest_hash` محتوای آن را مشخص می‌کند:

| حالت | نتیجه |
|---|---|
| ID و hash تکراری | همان رکورد و Receipt قبلی |
| ID تکراری با hash متفاوت | Conflict و توقف Import |
| ID جدید | Measurement جدید |

---

## Constraintهای لازم در PostgreSQL (فراتر از DBML)

1. ترتیب مجاز Parent/Child و نبود Cycle در Tree
2. نوع Node مربوط به Machine، Point و Direction
3. Mobile و Data Collector در نقش صحیح
4. Root حتماً Point داشته باشد
5. Directionهای Signal متعلق به همان Point باشند
6. تعداد محور Collector فقط ۱ یا ۳ باشد
7. Rate، Count، Duration، Byte Size و RPM منفی نباشند
8. فقط یک Spec Revision فعال برای هر Node وجود داشته باشد
9. Node دارای History فقط Soft Delete شود
10. Measurement ثبت‌شده بعد از Receipt تغییر نکند

---

## انتخاب Storage

| محل | انتخاب |
|---|---|
| Android | Room روی SQLite |
| Desktop یا Site Server | PostgreSQL |
| Raw Signal | فایل یا Object Storage |
| Metadata و Feature | PostgreSQL |

---

## Indexهای اصلی (خلاصه Query → Index)

| Query | Index |
|---|---|
| فرزندان یک Node | `factory + parent + sort` |
| پیدا کردن Machine | `factory + tag` |
| Trend یک Point | `point + measured_at` |
| صف ارسال موبایل | `mobile + transfer_status` |
| History یک Direction | `direction + measurement` |
| Alertهای اخیر | `direction/time` و `severity/time` |

---

## چه زمانی مدل کامل (۲۶ جدولی) لازم می‌شود؟

- مدیریت چند Organization و Site
- Cloud Relay و Transfer Batch
- Dedup فیزیکی Blobها
- چرخه عمر جدا و پیچیده Mobile و Collector
- Audit و امضای امنیتی گسترده‌تر

تا قبل از این نیازها، **مدل ساده ۱۵ جدولی** هزینه پیاده‌سازی و تست کمتری دارد و برای MVP کافی است.

---

## جمع‌بندی جریان داده

```
Mobile (Room/SQLite)
   │
   │ جمع‌آوری بر اساس Assignment
   ▼
Measurement + Signals (فایل Raw خارج از DB)
   │
   │ انتقال با manifest_hash
   ▼
Desktop (PostgreSQL)
   │
   │ ثبت اتمیک + ساخت import_receipt
   ▼
Mobile داده Root را پاک می‌کند
   │
   ▼
DSP → derived_features
   │
   ▼
Screening → screening_results
   │
   ▼
Threshold Rules → Alert
```

اگر نیاز به توضیح عمیق‌تر روی هر بخش (مثلاً Sync، Conflict Resolution یا DSP Pipeline) داری، بگو تا جزئیات بیشتری ارائه کنم.


# توضیح دقیق‌تر چهار جدول اصلی جریان جمع‌آوری داده

این چهار جدول **قلب جریان کاری SeeNora** هستند: از لحظه‌ای که یک مأموریت به Mobile فرستاده می‌شود، تا لحظه‌ای که داده خام به سیگنال‌های X/Y/Z تبدیل و ذخیره می‌شود.

---

## ۱. `collection_assignments` — مأموریت جمع‌آوری

### مسئولیت

نماینده یک **مأموریت کاری** که از Desktop/Site به یک دستگاه Mobile فرستاده می‌شود. این جدول می‌گوید:

> «کاربر X باید با موبایل Y، این لیست از Directionها را در کارخانه Z اندازه‌گیری کند.»

### ساختار

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `mobile_device_id` | uuid | ارجاع به دستگاه Mobile (باید از نوع MOBILE باشد) |
| `name` | varchar(200) | نام مأموریت (مثلاً «روتین هفتگی خط ۳») |
| `tree_revision` | bigint | نسخه درخت در لحظه ایجاد مأموریت |
| `status` | varchar(40) | پیش‌فرض ACTIVE |
| `created_at` | timestamptz | زمان ایجاد |
| `completed_at` | timestamptz | زمان تکمیل (اگر تکمیل شده) |

### چرا `tree_revision` مهم است؟

فرض کن مأموریت در تاریخ ۱ فروردین ساخته شده و `tree_revision = 5` بوده. در ۱۵ فروردین، یک Point جدید به درخت اضافه می‌شود و `tree_revision = 6` می‌شود.

اگر Mobile نسخه ۵ را داشته باشد، مأموریت همچنان معتبر است چون بر اساس همان نسخه ساخته شده. این باعث می‌شود:

- Mobile بداند کدام Directionها را باید اندازه‌گیری کند
- اگر درخت در Desktop تغییر کرد، مأموریت قدیمی باطل نشود
- Sync بتواند تصمیم بگیرد که آیا Mobile باید درخت را به‌روزرسانی کند یا نه

### چرخه عمر (Status)

```
ACTIVE → COMPLETED
ACTIVE → CANCELLED
ACTIVE → EXPIRED
```

معمولاً:
- `ACTIVE`: مأموریت در حال اجراست
- `COMPLETED`: همه Directionها اندازه‌گیری شده‌اند
- `CANCELLED`: کاربر لغو کرده
- `EXPIRED`: تاریخ انقضا گذشته

### نکته مهم

`mobile_device_id` **باید** به یک دستگاه از نوع `MOBILE` اشاره کند. اگر به `DATA_COLLECTOR` اشاره کند، Constraint باید خطا بدهد (مورد ۳ در لیست Constraintها).

---

## ۲. `assignment_nodes` — گره‌های مأموریت

### مسئولیت

این جدول **لیست Directionهایی** است که در یک مأموریت باید اندازه‌گیری شوند، به همراه ترتیب آن‌ها.

اگر `collection_assignments` بگوید «چه مأموریتی»، `assignment_nodes` می‌گوید «دقیقاً کدام Directionها و به چه ترتیبی».

### ساختار

| ستون | نوع | توضیح |
|---|---|---|
| `assignment_id` | uuid | ارجاع به مأموریت |
| `measurement_node_id` | uuid | ارجاع به گره `MEASUREMENT_DIRECTION` |
| `sort_order` | int | ترتیب اندازه‌گیری |
| `is_enabled` | boolean | فعال/غیرفعال |

**کلید اصلی مرکب:** `(assignment_id, measurement_node_id)`
**ایندکس:** `(assignment_id, sort_order)`

### چرا `sort_order` مهم است؟

اپراتور موبایل باید بداند به چه ترتیبی به سراغ Directionها برود. مثلاً:

```
sort_order | measurement_node_id
───────────┼────────────────────
1          | Motor-A-DE-H
2          | Motor-A-DE-V
3          | Motor-A-DE-A
4          | Motor-A-NDE-H
5          | Motor-A-NDE-V
6          | Motor-A-NDE-A
```

این ترتیب معمولاً بر اساس **مسیر فیزیکی حرکت اپراتور** در کارخانه تعیین می‌شود تا وقت تلف نشود.

### چرا `is_enabled`؟

گاهی اپراتور می‌خواهد یک Direction را موقتاً رد کند (مثلاً دسترسی سخت است یا ماشین خاموش است). به جای حذف ردیف، فقط `is_enabled = false` می‌کند. این کار:

- Audit را حفظ می‌کند
- امکان فعال‌سازی مجدد را می‌دهد
- Sync را ساده‌تر می‌کند

### نکته مهم

`measurement_node_id` **باید** به یک گره از نوع `MEASUREMENT_DIRECTION` اشاره کند. این Direction پایدار است و UUID آن بین Desktop و Mobile تغییر نمی‌کند.

### رابطه با `collection_assignments`

```
collection_assignments (1)
        │
        │ یک مأموریت
        ▼
assignment_nodes (N)
        │
        │ N تا Direction
        ▼
asset_nodes (MEASUREMENT_DIRECTION)
```

---

## ۳. `measurements` — اندازه‌گیری‌ها

### مسئولیت

نماینده **یک برداشت کامل** از یک Point. این جدول اطلاعات مشترک بین همه سیگنال‌های X/Y/Z را نگه می‌دارد.

هر Measurement یک **Start → Finish** دارد. یعنی اپراتور دکمه Start را زده، داده جمع‌آوری شده، و دکمه Finish را زده.

### ساختار (دسته‌بندی شده)

#### الف) شناسایی و ارتباط

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `factory_id` | uuid | ارجاع به کارخانه |
| `assignment_id` | uuid | ارجاع به مأموریت (اگر از مأموریت آمده) |
| `measurement_point_id` | uuid | ارجاع به Measurement Point |
| `mobile_device_id` | uuid | دستگاه Mobile |
| `data_collector_id` | uuid | Data Collector |

#### ب) زمان و حالت

| ستون | نوع | توضیح |
|---|---|---|
| `mode` | enum | ROOT / OFF_ROOT_FREE |
| `measured_at` | timestamptz | زمان اندازه‌گیری |
| `received_at` | timestamptz | زمان دریافت در Desktop |

#### پ) پارامترهای فنی

| ستون | نوع | توضیح |
|---|---|---|
| `actual_rpm` | decimal | دور واقعی در لحظه اندازه‌گیری |
| `sample_rate_hz` | int | نرخ نمونه‌برداری |
| `expected_sample_count` | int | تعداد نمونه مورد انتظار |
| `duration_ms` | int | مدت اندازه‌گیری |
| `acquisition_settings` | jsonb | تنظیمات کامل جمع‌آوری |

#### ت) زمینه (Context) و یکپارچگی

| ستون | نوع | توضیح |
|---|---|---|
| `spec_snapshot` | jsonb | تصویر Spec در زمان جمع‌آوری (تغییرناپذیر) |
| `context_hash` | char(64) | هش زمینه |
| `manifest_hash` | char(64) | هش محتوا |

#### ث) وضعیت‌ها

| ستون | نوع | توضیح |
|---|---|---|
| `quality` | enum | PENDING / VALID / INCOMPLETE / INVALID |
| `quality_details` | jsonb | جزئیات کیفیت |
| `transfer_status` | enum | PENDING / SELECTED / UPLOADING / FAILED / COMMITTED |
| `processing_status` | enum | PENDING / RUNNING / READY / FAILED |

#### ج) دما

| ستون | نوع | توضیح |
|---|---|---|
| `temperature_value` | decimal | مقدار دما |
| `temperature_unit` | varchar | پیش‌فرض C |
| `temperature_status` | enum | VALID / UNAVAILABLE / INVALID / OUT_OF_RANGE |

### حالت‌های اندازه‌گیری (mode)

| حالت | معنی | نیاز به Point؟ | نیاز به Direction؟ |
|---|---|---|---|
| `ROOT` | اندازه‌گیری روی Point مشخص با Directionهای نگاشت‌شده | ✅ بله | ✅ بله |
| `OFF_ROOT_FREE` | اندازه‌گیری آزاد روی ماشینی که Point مشخصی ندارد | ❌ خیر | ❌ خیر |

**نکته حیاتی:** حالت `OFF_ROOT_LIVE` در این جدول **وجود ندارد**. چرا؟ چون در آن حالت، داده در یک Ring Buffer موقت نگه داشته می‌شود و پس از پایان Live View دور ریخته می‌شود. فقط اگر کاربر تصمیم بگیرد آن را ذخیره کند، به `OFF_ROOT_FREE` تبدیل می‌شود.

### سه وضعیت مستقل (بسیار مهم)

این سه وضعیت **مستقل از هم** هستند و می‌توانند ترکیب‌های مختلف بسازند:

```
Quality:    VALID
Transfer:   COMMITTED
Processing: FAILED
```

این ترکیب یعنی: «داده سالم است، به Desktop رسیده، ولی DSP شکست خورده.» در این حالت Raw امن است و فقط باید DSP دوباره اجرا شود.

### چرا `spec_snapshot` و `context_hash` مهم‌اند؟

`spec_snapshot` یک **تصویر تغییرناپذیر** از مشخصات فنی ماشین در لحظه اندازه‌گیری است. اگر فردا کسی Spec ماشین را عوض کند، Measurement تاریخی همچنان با Spec قدیمی تفسیر می‌شود. این باعث می‌شود:

- Measurementهای تاریخی همیشه قابل تفسیر باشند
- Screening با Ruleset درست اجرا شود
- مقایسه Trend معنی‌دار باشد

`context_hash` هش همه این زمینه است تا بتوانی بفهمی آیا دو Measurement در شرایط یکسان گرفته شده‌اند یا نه.

### چرا `manifest_hash` مهم است؟

`manifest_hash` هش کل محتوای Measurement است (شامل Metadata و سیگنال‌ها). این برای **جلوگیری از Duplicate** استفاده می‌شود:

| حالت | نتیجه |
|---|---|
| ID و hash تکراری | همان رکورد و Receipt قبلی |
| ID تکراری با hash متفاوت | Conflict و توقف Import |
| ID جدید | Measurement جدید |

### ایندکس‌ها و کاربردشان

| ایندکس | کاربرد |
|---|---|
| `(factory_id, id)` یکتا | شناسایی سریع |
| `(measurement_point_id, measured_at)` | Trend یک Point |
| `(mobile_device_id, transfer_status)` | صف ارسال موبایل |
| `(data_collector_id, measured_at)` | History یک Collector |

---

## ۴. `measurement_signals` — سیگنال‌های اندازه‌گیری

### مسئولیت

برای هر **کانال فیزیکی X/Y/Z** یک ردیف نگه می‌دارد. داده خام در دیتابیس نیست؛ فقط متادیتا و آدرس فایل.

### ساختار

| ستون | نوع | توضیح |
|---|---|---|
| `id` | uuid | کلید اصلی |
| `measurement_id` | uuid | ارجاع به Measurement |
| `sensor_axis` | varchar(8) | محور سنسور (X, Y, Z یا H, V, A) |
| `measurement_node_id` | uuid | نگاشت به Direction (در حالت Root) |
| `storage_key` | text | آدرس فایل Raw (یکتا) |
| `sha256` | char(64) | هش فایل |
| `byte_size` | bigint | اندازه فایل |
| `encoding` | varchar(60) | مثال: float32-le |
| `compression` | varchar(40) | فشرده‌سازی |
| `sample_count` | int | تعداد نمونه |
| `acceleration_unit` | varchar(20) | پیش‌فرض m/s2 |
| `scale_factor` | decimal | ضریب مقیاس |
| `quality` | enum | VALID / MISSING / CLIPPED / CORRUPTED / OUT_OF_RANGE |
| `quality_details` | jsonb | جزئیات |

### چرا جدا از `measurements`؟

چون یک Measurement می‌تواند **چند سیگنال** داشته باشد. معمولاً:

```
Measurement (1)
   ├── Signal X  → Direction H (افقی)
   ├── Signal Y  → Direction V (عمودی)
   └── Signal Z  → Direction A (محوری)
```

اگر Data Collector سه‌محوره باشد، سه ردیف در `measurement_signals` داریم. اگر تک‌محوره باشد، یک ردیف.

### نگاشت `measurement_node_id`

در حالت `ROOT`، هر سیگنال باید به یک Direction نگاشت شود:

| sensor_axis | measurement_node_id |
|---|---|
| X | Motor-A-DE-H |
| Y | Motor-A-DE-V |
| Z | Motor-A-DE-A |

در حالت `OFF_ROOT_FREE`، این ستون `NULL` است چون Point مشخصی وجود ندارد.

**قاعده مهم:** همه Directionهای یک Measurement باید متعلق به **همان Point** باشند (مورد ۵ در لیست Constraintها).

### داده خام کجاست؟

داده خام **در دیتابیس نیست**. فقط:

- `storage_key`: آدرس فایل (مثلاً `s3://seenora/raw/2024/01/01/abc.bin`)
- `sha256`: هش برای بررسی یکپارچگی
- `byte_size`: اندازه فایل
- `encoding`: فرمت داده
- `sample_count`: تعداد نمونه

این طراحی چند مزیت دارد:

1. **دیتابیس کوچک می‌ماند** — فایل‌های خام می‌توانند گیگابایت باشند
2. **Object Storage ارزان‌تر است** — S3 یا MinIO
3. **Backup سریع‌تر است** — فقط Metadata
4. **Dedup ممکن است** — اگر دو Measurement یک فایل داشته باشند، `sha256` یکی است

### `sha256` و یکپارچگی

`sha256` هش فایل خام است. هنگام Import:

1. فایل دریافت می‌شود
2. هش محاسبه می‌شود
3. با `sha256` در Metadata مقایسه می‌شود
4. اگر مطابقت نداشت → `quality = CORRUPTED`

### `quality` در سطح سیگنال

برخلاف `measurements.quality` که کل برداشت را توصیف می‌کند، `measurement_signals.quality` وضعیت **هر محور** را جداگانه می‌گوید:

| مقدار | معنی |
|---|---|
| `VALID` | سیگنال سالم |
| `MISSING` | سیگنال وجود ندارد |
| `CLIPPED` | سیگنال اشباع شده (بیش از حد بزرگ) |
| `CORRUPTED` | فایل خراب است |
| `OUT_OF_RANGE` | خارج از بازه مجاز |

ممکن است Measurement کلی `VALID` باشد ولی یک محور `CLIPPED` باشد.

### ایندکس‌ها و کاربردشان

| ایندکس | کاربرد |
|---|---|
| `(measurement_id, sensor_axis)` یکتا | هر محور فقط یک بار |
| `(measurement_node_id, measurement_id)` | History یک Direction |
| `sha256` | Dedup و بررسی یکپارچگی |

---

## جریان کامل داده در این چهار جدول

```
Desktop
   │
   │ ۱. ساخت مأموریت
   ▼
collection_assignments
   │  (id = A1, mobile = M1, tree_revision = 5)
   │
   │ ۲. افزودن Directionها
   ▼
assignment_nodes
   │  (A1, Direction-DE-H, sort=1)
   │  (A1, Direction-DE-V, sort=2)
   │  (A1, Direction-DE-A, sort=3)
   │
   │ ۳. Sync به Mobile
   ▼
Mobile
   │
   │ ۴. اپراتور Start می‌زند
   ▼
measurements
   │  (id = M-1, point = P1, mode = ROOT, ...)
   │
   │ ۵. سه سیگنال جمع‌آوری می‌شود
   ▼
measurement_signals
   │  (M-1, X, Direction-DE-H, storage_key=..., sha256=...)
   │  (M-1, Y, Direction-DE-V, storage_key=..., sha256=...)
   │  (M-1, Z, Direction-DE-A, storage_key=..., sha256=...)
   │
   │ ۶. انتقال به Desktop
   ▼
Desktop
   │
   │ ۷. بررسی hash و ثبت
   ▼
import_receipts
   │  (M-1, authority, manifest_hash, ...)
   │
   │ ۸. Mobile داده Root را پاک می‌کند
   ▼
DSP → derived_features → screening_results
```

---

## جمع‌بندی

| جدول | نقش | کلید فهم |
|---|---|---|
| `collection_assignments` | «چه مأموریتی؟» | به Mobile فرستاده می‌شود |
| `assignment_nodes` | «کدام Directionها و به چه ترتیبی؟» | لیست کاری اپراتور |
| `measurements` | «چه برداشتی؟» | اطلاعات مشترک + Context |
| `measurement_signals` | «کدام محورها و کجا ذخیره شده‌اند؟» | یک ردیف per axis |

### نکات کلیدی که نباید فراموش کنی

1. **`tree_revision`** در مأموریت، مأموریت را در برابر تغییرات درخت مقاوم می‌کند.
2. **`is_enabled`** در `assignment_nodes` اجازه رد کردن موقت را می‌دهد بدون از دست دادن Audit.
3. **سه وضعیت مستقل** (`quality`, `transfer_status`, `processing_status`) هر کدام جداگانه معنی دارند.
4. **`spec_snapshot`** و **`context_hash`** Measurement را در برابر تغییرات آینده Spec مقاوم می‌کنند.
5. **`manifest_hash`** برای جلوگیری از Duplicate و تشخیص Conflict است.
6. **داده خام در دیتابیس نیست** — فقط `storage_key`، `sha256` و متادیتا.
7. **`OFF_ROOT_LIVE` هرگز ذخیره نمی‌شود** — فقط `ROOT` و `OFF_ROOT_FREE`.
8. **`measurement_node_id`** در `measurement_signals` فقط در حالت `ROOT` پر می‌شود.

اگر خواستی، می‌توانم یک **مثال کامل End-to-End** با داده‌های واقعی بنویسم که این چهار جدول را پر کند.