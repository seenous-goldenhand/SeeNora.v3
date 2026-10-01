# همگام‌سازی و حل تعارض

## Push و Pull در SeeNora

- موبایل Tree و Receiptهای جدید را از Desktop دریافت می‌کند: Pull.
- موبایل Measurementهای انتخاب‌شده را به Desktop می‌فرستد: Push.
- اعلان Push یا WebSocket فقط می‌تواند بگوید «تغییر جدید وجود دارد»؛ داده اصلی با API دریافت می‌شود.

این ترکیب ساده است و اگر اعلان از دست برود، Sync بعدی اطلاعات را بازیابی می‌کند.

## نسخه‌بندی درخت

هر Factory یک `tree_revision` و هر Node یک `revision` دارد. موبایل آخرین نسخه دریافت‌شده را نگه می‌دارد.

برای MVP می‌توان Branch انتخاب‌شده را کامل ارسال کرد و موبایل آن را براساس UUID به‌صورت Upsert وارد کند. در مقیاس بزرگ‌تر، Desktop فقط تغییرات بعد از آخرین Revision را می‌فرستد.

قواعد Import:

- UUID موجود: Update همان Node.
- UUID جدید: Insert.
- Node حذف‌شده: Tombstone یا `deleted_at`؛ حذف فیزیکی ممنوع اگر History دارد.
- نام یکسان با UUID متفاوت: دو Node متفاوت؛ ادغام خودکار ممنوع.

## تنظیم محلی موبایل

فیلدهایی مانند «در این مأموریت فعال است» و ترتیب دلخواه تکنسین باید جدا از Tree مرجع نگهداری شوند:

```text
Node from Desktop: name, parent, identity, technical data
Local preference: enabled, local order, last used
```

Update درخت فقط بخش اول را تغییر می‌دهد و تنظیم محلی را پاک نمی‌کند.

## انتقال بدون Duplicate

هر Measurement یک UUID و `manifest_hash` دارد:

- ID و hash یکسان: همان Measurement قبلی؛ Receipt قبلی برگردد.
- ID یکسان و hash متفاوت: خطای جدی؛ overwrite ممنوع.
- ID متفاوت: Measurement جدید، حتی اگر زمان و Node مشابه باشد.

ارسال ممکن است چند بار تکرار شود، ولی اثر آن در Desktop باید یک بار باشد.

## چه زمانی Vector Clock لازم است؟

Vector Clock زمانی لازم است که دو Desktop بتوانند یک Machine Spec را مستقل و آفلاین تغییر دهند. برای طراحی پیشنهادی که فقط Desktop مرجع Tree را می‌نویسد، Revision ساده کافی است.

Vector Clock خودش Conflict را حل نمی‌کند؛ فقط نشان می‌دهد دو تغییر هم‌زمان بوده‌اند. انتخاب RPM، Bearing یا Parent درست همچنان به Rule یا کاربر نیاز دارد.
