# داده و پردازش سیگنال

## چه چیزی کجا ذخیره می‌شود؟

| داده | محل |
|---|---|
| Tree، Spec، وضعیت و Feature | دیتابیس |
| Raw X/Y/Z | فایل یا Object Storage |
| آدرس، اندازه و SHA-256 فایل | دیتابیس |
| نمودار و Feature قابل بازسازی | دیتابیس یا Cache |

ذخیره Raw داخل ردیف اصلی دیتابیس Backup، انتقال و Query را سخت‌تر می‌کند. فایل با hash ثابت نگهداری می‌شود و Metadata لازم برای تفسیر آن کنار Measurement ثبت می‌شود.

## ساختار Measurement

هر Measurement شامل این موارد است:

- UUID، Mode و زمان برداشت
- Mobile ID و Data Collector ID
- Point در Root یا Tag اختیاری در Off-root
- RPM واقعی در صورت دور متغیر
- Sample Rate، Duration و Sample Count
- وضعیت کیفیت برداشت
- دما و وضعیت سنسور دما
- Context فنی زمان برداشت
- یک تا سه Signal برای X/Y/Z

Axis سنسور با Direction ماشین فرق دارد. جدول Signal مشخص می‌کند مثلاً X سنسور در این برداشت معادل Direction افقی کدام Point بوده است.

## Context تاریخی

تغییر Bearing یا RPM شناسنامه نباید تحلیل گذشته را عوض کند. هنگام برداشت، Spec لازم داخل `spec_snapshot` ذخیره می‌شود. Raw Signal و Snapshot بعد از Commit تغییر نمی‌کنند.

## پردازش

```text
Raw validation
  -> Time Signal / FFT / Envelope
  -> RMS / Overall / Crest / Kurtosis
  -> Trend
  -> Screening
```

هر خروجی پردازش نام و نسخه الگوریتم دارد. اگر الگوریتم تغییر کرد، خروجی جدید ساخته می‌شود و نتیجه قدیمی حذف نمی‌شود.

## Trend

Trend از Featureهای آماده خوانده می‌شود، نه از محاسبه دوباره Raw در هر بار بازکردن صفحه. Pointهایی با تنظیمات یا شرایط غیرقابل‌مقایسه باید در UI علامت داشته باشند.

## Screening

خروجی Screening یکی از `Normal`, `Warning`, `Critical` یا `Insufficient Data` است و دلیل Rule را نگه می‌دارد. Screening تشخیص نهایی خرابی نیست.

## نگهداری محلی Android

- Root تا Receipt موفق Desktop حذف نمی‌شود.
- Measurement ناقص از معتبر جدا نگه داشته می‌شود.
- Free Off-root طبق سیاست محصول موقت ذخیره می‌شود.
- Live Off-root فقط Ring Buffer حافظه دارد.
- کمبود فضا قبل از شروع برداشت هشدار داده می‌شود.
