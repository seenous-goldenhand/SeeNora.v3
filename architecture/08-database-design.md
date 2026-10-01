# دیتابیس ساده SeeNora v3

## فایل‌ها

- `dbdiagram.txt`: مدل پیشنهادی MVP با ۱۵ جدول.
- `dbdiagram-full.txt`: مدل ۲۶ جدولی برای توسعه‌های آینده.

مدل ساده برای شروع پیشنهاد می‌شود. ویژگی‌های اصلی حذف نشده‌اند؛ جدول‌های مشابه با هم ادغام شده‌اند.

## انتخاب Storage

| محل | انتخاب |
|---|---|
| Android | Room روی SQLite |
| Desktop یا Site Server | PostgreSQL |
| Raw Signal | فایل یا Object Storage |
| Metadata و Feature | PostgreSQL |

## جدول‌ها به زبان ساده

| جدول | مسئولیت |
|---|---|
| `factories` | مرز داده و نسخه Tree یک کارخانه |
| `asset_nodes` | تمام سطوح Tree با Parent/Child |
| `machines` | Tag و اطلاعات قابل جست‌وجوی Machine |
| `asset_spec_revisions` | تاریخچه Spec ماشین یا Point |
| `asset_templates` | Template ساخت Machine/Point/Branch |
| `devices` | Mobile و Data Collector |
| `collection_assignments` | مأموریت ارسال‌شده به موبایل |
| `assignment_nodes` | Directionهای مأموریت و ترتیب آن‌ها |
| `measurements` | اطلاعات اصلی هر برداشت |
| `measurement_signals` | X/Y/Z و آدرس فایل Raw |
| `import_receipts` | تأیید ثبت بادوام در Desktop |
| `derived_features` | RMS، Overall، Crest و خروجی DSP |
| `threshold_rules` | قوانین ISO، دستی، Trend و Similar Machine |
| `screening_results` | نتیجه و دلیل Screening |
| `audit_events` | سابقه تغییرات حساس |

## مدل درخت

همه سطوح در یک جدول قرار دارند:

```text
LINE
  DEPARTMENT
    MACHINE
      MACHINE_PART
        MEASUREMENT_POINT
          MEASUREMENT_DIRECTION
```

این مدل Sync و Query فرزندان را ساده می‌کند. در مقابل، ترتیب صحیح نوع Parent/Child باید با Trigger یا Service کنترل شود.

Direction همان Measurement Node پایدار است. UUID آن بین Desktop و Android تغییر نمی‌کند.

## Machine و Spec

اطلاعات پرتکرار مانند Tag، Priority و Nominal RPM در `machines` ستون مستقل دارند. اطلاعات متغیر یا وابسته به نوع تجهیز در JSONB نگهداری می‌شوند.

هر تغییر فنی مهم یک ردیف جدید در `asset_spec_revisions` می‌سازد. Revision قبلی حذف نمی‌شود تا Measurement تاریخی قابل تفسیر باقی بماند.

## Measurement و Signal

`measurements` اطلاعات مشترک برداشت را نگه می‌دارد. `measurement_signals` برای هر محور X/Y/Z یک ردیف دارد.

```text
Measurement
  Signal X -> Direction H -> raw file
  Signal Y -> Direction V -> raw file
  Signal Z -> Direction A -> raw file
  Temperature
```

Raw Sampleها داخل دیتابیس قرار نمی‌گیرند. دیتابیس `storage_key`، SHA-256، اندازه، encoding و sample count را نگه می‌دارد.

## سه وضعیت مستقل

| وضعیت | مثال | کاربرد |
|---|---|---|
| Quality | VALID یا INCOMPLETE | سلامت برداشت |
| Transfer | PENDING یا COMMITTED | تحویل به Desktop |
| Processing | PENDING یا READY | آماده‌بودن Featureها |

ممکن است داده `VALID` و `COMMITTED` باشد ولی پردازش آن `FAILED` شود. در این حالت Raw امن است و فقط DSP دوباره اجرا می‌شود.

## جلوگیری از Duplicate

Measurement با UUID شناخته می‌شود. `manifest_hash` محتوای آن را مشخص می‌کند:

- ID و hash تکراری: همان رکورد و Receipt قبلی.
- ID تکراری با hash متفاوت: Conflict و توقف Import.
- ID جدید: Measurement جدید.

## Receipt

`import_receipts` بعد از کنترل و ثبت کامل Metadata و Raw ساخته می‌شود. موبایل قبل از Receipt نباید داده Root را حذف کند. Receipt دریافتی باید با Measurement ID، hash و Authority مورد انتظار تطبیق داده شود.

## Constraintهای لازم در PostgreSQL

DBML همه قواعد را بیان نمی‌کند. Migration باید این موارد را اضافه کند:

1. ترتیب مجاز Parent/Child و نبود Cycle در Tree.
2. نوع Node مربوط به Machine، Point و Direction.
3. Mobile و Data Collector در نقش صحیح.
4. Root حتماً Point داشته باشد.
5. Directionهای Signal متعلق به همان Point باشند.
6. تعداد محور Collector فقط ۱ یا ۳ باشد.
7. Rate، Count، Duration، Byte Size و RPM منفی نباشند.
8. فقط یک Spec Revision فعال برای هر Node وجود داشته باشد.
9. Node دارای History فقط Soft Delete شود.
10. Measurement ثبت‌شده بعد از Receipt تغییر نکند.

## Indexهای اصلی

| Query | Index |
|---|---|
| فرزندان یک Node | factory + parent + sort |
| پیدا کردن Machine | factory + tag |
| Trend یک Point | point + measured_at |
| صف ارسال موبایل | mobile + transfer_status |
| History یک Direction | direction + measurement |
| Alertهای اخیر | direction/time و severity/time |

## چه زمانی مدل کامل لازم می‌شود؟

- مدیریت چند Organization و Site
- Cloud Relay و Transfer Batch
- Dedup فیزیکی Blobها
- چرخه عمر جدا و پیچیده Mobile و Collector
- Audit و امضای امنیتی گسترده‌تر

تا قبل از این نیازها، مدل ساده هزینه پیاده‌سازی و تست کمتری دارد و برای MVP کافی است.
