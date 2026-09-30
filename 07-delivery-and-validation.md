# برنامه اجرا، کیفیت و منابع فنی

## برنامه مطابق سه Gate پروژه

این برنامه جزئیات فنی پیشنهادی برای Milestone Plan موجود است؛ تعهد زمانی قطعی بدون شناخت ظرفیت تیم نیست. مسیر بحرانی همان Foundation → Acquisition → Storage → Transfer → Trend/Analysis/Screening باقی می‌ماند. Support و RAG نسخه چهارم نباید Gateهای v3 را متوقف کنند.

| مرحله | خروجی قابل نمایش | وابستگی/مالک پیشنهادی | شاهد پذیرش |
|---|---|---|---|
| آغاز ماه اول | واژه‌نامه، Data Contract، Identity و سیاست Off-root | Technical Owner با نمایندگان چهار تیم و Product Owner | قرارداد نمونه و تصمیم ابهام‌های سند ۱ |
| ماه اول، Desktop | Tree/Spec، UUID، persistence، snapshot export | تیم Desktop | بازگشایی برنامه بدون تغییر ID |
| ماه اول، IoT/Android | اتصال، capability، سه محور/دما، storage، modeهای Free و Live جدا | IoT و Android | fixture واقعی و قطع اتصال قابل تشخیص |
| ماه اول، AI/DSP | الگوریتم نسخه‌دار و Reference Dataset | تیم AI با Desktop | خروجی‌های عددی و tolerance مصوب |
| ماه دوم | شبکه + USB، Import مشترک، Receipt، selection، retry، Delta | Android و Desktop | حلقه Root بسته و duplicate صفر در آزمون |
| ماه دوم | Quality و Screening rules نسخه‌دار | AI و Desktop | Normal/Warning/Critical/Insufficient Data با علت |
| ماه سوم | Trend/Analysis، performance، recovery، installer و field test | همه تیم‌ها | گزارش پذیرش و بسته نصب |

Cloud Relay اگر روز اول الزام Cellular داشته باشیم بخشی از ماه‌های اول/دوم است؛ نمی‌توان آن را هم ضروری دانست و هم بی‌هزینه به پایان Beta موکول کرد. B نیز اگر انتخاب شود، نصب و بهره‌برداری سرور سایت باید deliverable رسمی باشد.

## PoCهای تصمیم‌ساز پیش از تعهد بزرگ

1. **انتقال واقعی سنسور:** نرخ مؤثر سه محور، افت packet، دما، battery و محدودیت firmware با data واقعی؛ خروجی انتخاب transport سخت‌افزار است.
2. **Sync شبکه‌ای شکننده:** یک Measurement بزرگ در چند قطع ارتباط، kill برنامه و Retry؛ hash و شمار رکورد مقصد بررسی شوند.
3. **USB رفت و برگشت:** روی Android و OS واقعی مشتری، Export، Import و Receipt بدون ADB؛ صرف نمایش فایل روی PC کافی نیست.
4. **پردازش و نمودار:** dataset مشترک Mobile/Desktop، اختلاف عددی و latency روی ضعیف‌ترین سخت‌افزار پایلوت.
5. **موتور آماده، فقط در صورت بررسی F:** scoped replication، conflict، Blob بزرگ، USB و خروج داده از vendor.
6. **Multi-master، فقط در صورت الزام E:** دو edit هم‌زمان، cycle، duplicate Tag، delete-vs-edit و بازگشت replica قدیمی؛ خروجی باید هم convergence و هم invariantهای دامنه را نشان دهد.

## آزمون‌های پذیرش و تزریق خطا

| شناسه | سناریو | نتیجه مورد انتظار | نیازمندی |
|---|---|---|---|
| T01 | بازگشایی برنامه پس از Saved | Raw و Metadata و ID یکسان | PRD07، PRD11 |
| T02 | قطع سنسور در میانه محور دوم | Incomplete، عدم ورود به History معتبر | PRD04–05 |
| T03 | خرابی سنسور دما | TemperatureUnavailable؛ ارتعاش سالم حفظ شود | PRD06 |
| T04 | نصب با Axis Mapping متفاوت | جهت درست و mapping قابل مشاهده در مقصد | PRD05 |
| T05 | kill موبایل وسط ذخیره فایل | staging ناقص قابل تشخیص؛ Saved کاذب وجود نداشته باشد | PRD11 |
| T06 | قطع شبکه پس از چند chunk | resume و فقط chunkهای لازم؛ Raw موبایل باقی | PRD12 |
| T07 | commit مقصد موفق و ACK گم شده | query Receipt/Retry همان نتیجه؛ یک رکورد | PRD12 |
| T08 | دو import هم‌زمان همان ID | یک Measurement و نتیجه idempotent | PRD12 |
| T09 | همان ID با hash متفاوت | reject conflict، بدون overwrite | PRD07، PRD12 |
| T10 | بخشی از batch خراب | Receipt فقط برای موارد موفق؛ Cleanup محدود به همان‌ها | PRD12 |
| T11 | فضای Desktop هنگام commit تمام شود | عدم success کاذب؛ حفظ موبایل | PRD12 |
| T12 | خاموشی در مرز rename فایل و DB commit | بدون committed record با Blob ناقص؛ orphan قابل GC | PRD11–12 |
| T13 | Import مجدد از USB پس از LAN | هیچ Duplicate جدید | PRD12 |
| T14 | Receipt USB جعلی یا برای مقصد دیگر | رد و ممنوعیت Cleanup | حفاظت طراحی |
| T15 | Tree جدید روی موبایل با داده ارسال‌نشده | ID و Measurement و override حفظ شوند | PRD03 |
| T16 | حذف/جابجایی Node بعد از مأموریت | حفظ Raw و انتساب بر ID؛ archive/quarantine روشن | PRD01، PRD12 |
| T17 | Snapshot چندصفحه‌ای هم‌زمان با تغییر Tree | نسخه سازگار و Delta بدون gap | PRD03 |
| T18 | cursor قدیمی‌تر از retention | Full Sync بدون حذف Outbox و preferences | PRD03 |
| T19 | گوشی با ساعت یک روز اشتباه | ID تکراری نشود؛ clock quality آشکار؛ no LWW destructive | طراحی زمان |
| T20 | یک هفته آفلاین با حجم هدف | برداشت تا ظرفیت توافق‌شده؛ هشدار قبل از پرشدن | PRD11 |
| T21 | RelayStored، Desktop خاموش | عدم مجوز Cleanup؛ وضعیت pending قابل فهم | PRD12 |
| T22 | حذف تمام اعلان‌های Push | Pull بعدی کامل بازیابی کند | قرارداد Sync |
| T23 | Spec/RPM/Threshold عوض شود | Raw و Alert تاریخی ثابت؛ تحلیل مجدد نسخه تازه | PRD02، PRD15 |
| T24 | نبود Context/History معتبر | Insufficient Data، بدون fault قطعی | PRD14–15 |
| T25 | Live Off-root پایان یابد | داده نهایی در آرشیو/کش پایدار نباشد | PRD09 |
| T26 | Free Off-root و restart | ماندگاری مطابق سیاست مصوب، مستقل از Live | PRD08 |
| T27 | credential سایت A به داده B دست بزند | منع دسترسی، بدون افشای وجود Blob | امنیت |
| T28 | Restore کامل روی سیستم دیگر | ID، Blob، Feature، Receipt و کلید بازیابی سازگار | تداوم کار |
| T29 | v1 موبایل با schema ناسازگار مقصد | خطای قابل اقدام، بدون حذف یا تبدیل خاموش واحد | سازگاری |
| T30 | archive مخرب/بسیار بزرگ | رد امن بدون نوشتن خارج از staging یا پرکردن دیسک | USB |

برای E علاوه بر این‌ها: permutation ترتیب عملیات، پیام تکراری، partition طولانی، بازگشت actor retired، reset شمارنده، merge هم‌زمان و invariantهای Tree آزموده شوند. تست property-based باید بررسی کند پس از تحویل تمام عملیات مجاز، replicaها همگرا هستند و هیچ Measurement ثبت‌شده بی‌علت ناپدید نشده است.

## معیارهای کیفیت پیشنهادی برای توافق

سند Quality Planning خالی است و هدف عددی رسمی از آن قابل استخراج نیست. اعداد زیر **هدف پیشنهادی آزمایشگاهی** هستند و پس از baseline سخت‌افزار و داده واقعی باید بازبینی شوند.

| معیار | پیشنهاد اولیه | روش اندازه‌گیری |
|---|---|---|
| Duplicate بر اثر Retry | صفر در ۱۰٬۰۰۰ ارسال تکراری کنترل‌شده | شمار IDهای یکتا و hash در DB |
| حذف Root تأییدنشده | صفر در تمام آزمون‌های خرابی | audit و inventory فایل موبایل |
| موفقیت Import داده معتبر | حداقل ۹۹٫۵٪ بدون اقدام اصلاحی در شبکه پایدار آزمون | denominator و نوع خطا مشخص باشد |
| بازیابی Retry | تمام موارد transient آزمون پس از رفع علت قابل تکمیل | محدود به ماتریس مشخص، نه تضمین مطلق |
| Snapshot درخت | زمان و RAM با ۱۰هزار Node آزمایش شود | p50/p95 روی دستگاه هدف؛ threshold پس از baseline |
| بازشدن Trend | هدف p95 زیر ۲ ثانیه برای بازه و حجم مصوب | query feature، cold/warm جدا |
| ذخیره پس از پایان برداشت | هدف p95 زیر ۲ ثانیه برای payload مرجع | شامل flush و ثبت Metadata؛ وابسته به دیسک |
| RPO آرشیو | پیشنهاد حداکثر یک شیفت، نیازمند تصمیم کسب‌وکار | فاصله backupهای قابل Restore |
| RTO استقرار A | پیشنهاد حداکثر ۴ ساعت | تمرین نصب و Restore روی سیستم جایگزین |

«داده از دست نرود» باید محدوده خرابی داشته باشد. پس از Cleanup موبایل، خرابی کامل دیسک Desktop بدون backup می‌تواند داده را از بین ببرد. اگر RPO صفر برای چنین خرابی لازم است، Receipt باید پس از replication بادوام مستقل یا backup هم‌زمان صادر شود و هزینه/دسترس‌پذیری آن صریحاً پذیرفته شود. حفظ محلی طولانی‌تر هم گزینه محصول است، ولی با الزام پاک‌سازی باید هماهنگ شود.

## امنیت متناسب با معماری

- مجوزهای حداقلی: technician برای مأموریت و ارسال؛ analyst برای تحلیل؛ asset editor برای Tree؛ admin برای Pairing و backup. یک نقش می‌تواند چند مجوز بگیرد، ولی endpoint باید authorization مستقل داشته باشد.
- رمزنگاری انتقال و کنترل کلید برای شبکه و بسته USB؛ raw sensor checksum فقط corruption را تشخیص می‌دهد و احراز هویت نیست.
- رمزنگاری storage طبق سیاست مشتری و امکان سیستم‌عامل، با کلید در secure storage؛ recovery key و تعویض دستگاه تمرین شود.
- هر attachment و package ورودی نامطمئن است؛ محدودیت حجم، schema validation، مسیر فایل و resource budget اعمال شود.
- credential و Raw کامل در log ثبت نشوند. support export با انتخاب کاربر و redaction داده حساس باشد.
- revoke دستگاه آفلاین تا زمان اطلاع دستگاه، نسخه دانلودشده را از راه دور نابود نمی‌کند. authorization مقصد فوراً می‌تواند تحویل جدید را رد کند؛ این دو تضمین متفاوت‌اند.
- نصب/به‌روزرسانی برنامه و firmware باید اصالت قابل بررسی داشته باشند؛ rollback schema DB بدون برنامه مهاجرت انجام نشود.

## مشاهده‌پذیری بدون Cloud اجباری

برای هر انتقال `batch_id`، `transfer_id`، `measurement_id` و `receipt_id` correlation بسازید. metricهای مهم: سن قدیمی‌ترین Outbox، حجم pending، retries به تفکیک علت، mismatch hash، تعداد quarantine، فضای آزاد، lag درخت، مدت ingest و backlog پردازش.

کاربر باید علت قابل اقدام ببیند: «۳ مورد ثبت شد، ۱ مورد به علت Node ناشناخته باقی ماند» از پیام مبهم «Sync failed» بهتر است. پیشرفت ارسال bytes از مرحله ثبت نهایی تفکیک شود. زمان آخرین Sync موفق و مقصد آن مشخص باشد.

در A، log و diagnostics محلی با export اختیاری کافی است. در B/C/D پایش سرویس‌ها و alert عملیاتی لازم می‌شود. موفقیت DSP، upload و commit سه metric جدا هستند.

## Backup و Restore

Backup باید Metadata، Blob، Receipt/dedup ledger، نسخه الگوریتم و Rule، audit و اطلاعات لازم بازیابی کلید را پوشش دهد. Snapshot سازگار DB و inventory Blob تهیه شود؛ فایل‌های در حال Import از snapshot نهایی جدا باشند. نسخه backup در failure domain دیگری نگهداری و صحت آن با restore واقعی سنجیده شود.

Restore قدیمی پس از Cleanup موبایل ممکن است Measurementهایی را که قبلاً Receipt داشته‌اند نداشته باشد. باید از backup جدیدتر، replica یا آرشیو مستقل بازیابی شوند؛ پروتکل نمی‌تواند داده حذف‌شده همه نسخه‌ها را بازسازی کند. این محدودیت باید در انتخاب RPO و زمان Cleanup دیده شود.

پس از Clone دیتابیس، اگر هر دو نسخه قرار است فعال شوند، شناسه authority/epoch و مالکیت دوباره تعیین شود تا دو مرجع مستقل با هویت یکسان به موبایل Receipt ندهند. برای E، actor incarnation و شمارنده نیز بخشی از recovery است.

## منابع فنی بررسی‌شده

این منابع برای راستی‌آزمایی مفاهیم فنی استفاده شده‌اند؛ طرح، APIها و اعداد پیشنهادی این مجموعه تصمیم اختصاصی SeeNora هستند. منابع محصول با شناسه S01 تا S10 در [سند نیازمندی‌ها](01-requirements-and-assumptions.md) آمده‌اند.

| منبع اولیه | کاربرد در این مجموعه |
|---|---|
| [Android Offline-first Architecture](https://developer.android.com/topic/architecture/data-layer/offline-first) | Local data source و صف کار پایدار |
| [RFC 9110 HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) | idempotency، conditional request و If-Match |
| [RFC 6455 WebSocket](https://www.rfc-editor.org/info/rfc6455/) | کانال دوطرفه؛ مستقل از Receipt دامنه |
| [Dynamo، مقاله اصلی SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) | version causality و vector clock |
| [Shapiro درباره CRDT](https://www.microsoft.com/en-us/research/video/strong-eventual-consistency-and-conflict-free-replicated-data-types/) | همگرایی نوع‌های داده تکثیرشده |
| [Apache CouchDB Conflicts](https://docs.couchdb.org/en/stable/replication/conflicts.html) | revision conflict و تفاوت winner با حل دامنه |
| [Apache CouchDB Replication Protocol](https://docs.couchdb.org/en/stable/replication/protocol.html) | ارزیابی موتور replication آماده |
| [SQLite Appropriate Uses](https://www.sqlite.org/whentouse.html) | مرز SQLite محلی و موتور client/server |
| [SQLite WAL](https://www.sqlite.org/wal.html) | محدودیت اشتراک WAL بین ماشین‌ها |
| [FCM Message Lifespan](https://firebase.google.com/docs/cloud-messaging/customize-messages/setting-message-lifespan) | اعلان best-effort در مقابل دریافت داده مرجع |

## خروجی جلسه تصمیم معماری

جلسه باید یک گزینه استقرار، authority نهایی، سیاست Free/Live Off-root، کانال‌های الزامی Beta، حجم و دوره آفلاین هدف، و سیاست Backup/Cleanup را تعیین کند. پس از این تصمیم‌ها، ADRهای سند ۶ از «پیشنهادی» به «پذیرفته‌شده» تغییر کنند و نسخه ۱ قرارداد بین Android، Desktop، IoT و AI امضا شود. خروجی جلسه صرف انتخاب نام دیتابیس یا پروتکل نباشد؛ سناریوی خطا و شاهد پذیرش نیز جزو تصمیم است.