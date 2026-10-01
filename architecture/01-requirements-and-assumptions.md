# نیازمندی‌های معماری

## مسئله اصلی

کارشناس در Desktop درخت تجهیزات را تعریف می‌کند. تکنسین بخشی از این درخت را روی Android می‌گیرد، در کارخانه داده‌برداری می‌کند و Measurementها را به همان Desktop برمی‌گرداند. Desktop داده را نگه می‌دارد و Trend، Analysis و Screening را نمایش می‌دهد.

```text
Desktop -> Tree selected for a mission -> Android
Android -> Measurement and raw signals -> Desktop
```

## نیازهای قطعی

- ساختار درخت: Line → Department → Machine → Part → Point → Direction.
- هر Node یک UUID ثابت داشته باشد.
- Rename یا Move باعث تغییر UUID نشود.
- موبایل فقط Branchهای لازم را دریافت کند.
- Tree و Measurementها روی موبایل آفلاین باقی بمانند.
- انتقال تکراری Node یا Measurement رکورد تکراری نسازد.
- به‌روزرسانی Tree، Measurement قدیمی را حذف یا جابه‌جا نکند.
- کاربر بتواند Directionهای مأموریت را روی موبایل غیرفعال کند.
- تنظیم محلی کاربر با Update بعدی Tree پاک نشود.
- Measurement ناقص یا خراب از داده معتبر قابل تشخیص باشد.
- پاک‌سازی موبایل فقط بعد از ثبت موفق در Desktop و تأیید کاربر انجام شود.

## تقسیم مسئولیت

| Desktop | Android |
|---|---|
| تعریف Tree و Machine Spec | نگهداری نسخه محلی Tree |
| انتخاب Branch مأموریت | اجرای Root و Off-root |
| آرشیو دائمی Raw | ذخیره موقت تا انتقال موفق |
| Trend، Analysis و Screening | نمایش سریع و Re-measurement |
| گزارش و Backup | صف Retry انتقال |

## تفاوت Root و Off-root

در Root، Measurement از ابتدا به Point و Directionهای شناخته‌شده وصل است. در Off-root، کاربر بدون Tree داده می‌گیرد و Tag فقط یک متن کمکی است. سیستم نباید Off-root را خودکار و فقط با تشابه نام به یک Machine متصل کند.

Off-root Live فقط نمایش زنده دارد و بعد از پایان در آرشیو ذخیره نمی‌شود. Off-root Free می‌تواند موقت ذخیره شود.

## خارج از مسیر اصلی v3

- جایگزینی کامل Data Collector حرفه‌ای
- مانیتورینگ آنلاین دائمی کارخانه
- CMMS کامل
- Diagnosis نهایی توسط AI
- ویرایش مستقل و آفلاین یک Tree توسط چند Desktop

این موارد می‌توانند بعداً اضافه شوند، ولی نباید تحویل حلقه اصلی Root را عقب بیندازند.
