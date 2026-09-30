# همگام‌سازی، علیت و حل تعارض

## تفکیک سه محور تصمیم

Centralized/Distributed درباره محل مرجع و هماهنگی است؛ Pull/Push درباره آغازگر انتقال؛ و Revision/Vector Clock درباره تشخیص نسخه و علیت. یک سامانه مرکزی می‌تواند Push داشته باشد و سامانه توزیع‌شده می‌تواند فقط Pull کند. خود پروتکل WebSocket، MQTT یا HTTP سیاست حل تعارض را تعیین نمی‌کند.

در این سند «Push داده» یعنی تولیدکننده داده ارسال را آغاز می‌کند. «Push notification» فقط اعلان تغییر است. «Pull» یعنی دریافت‌کننده درخواست داده می‌دهد. حتی در Push سرور به موبایل، اتصال شبکه غالباً از طرف موبایل باز می‌شود؛ سرور لازم نیست بتواند به IP گوشی اتصال ورودی بزند.

## مقایسه روش‌های انتقال

| روش | نمونه در SeeNora | مزیت | محدودیت | بازیابی الزامی |
|---|---|---|---|---|
| Manual Pull | تکنسین قبل از مأموریت Sync می‌زند | قابل فهم، کم‌مصرف | وابسته به اقدام کاربر | نمایش نسخه و آخرین Sync |
| Periodic Pull | پرس‌وجوی دوره‌ای تغییرات درخت | ساده و مقاوم به از دست رفتن اعلان | تأخیر و درخواست خالی | cursor بادوام، backoff و jitter |
| Long polling | درخواست منتظر تغییر می‌ماند | اعلان سریع‌تر بدون WebSocket | timeout و reconnect | از cursor آخر ادامه دهد |
| Push داده از موبایل | ارسال Measurement منتخب | مناسب Data Producer | خطا یا retry تکراری | Outbox و idempotent ingest |
| Push داده از مرجع | ارسال Delta روی کانال باز | تأخیر کم | replay، backpressure و موبایل خوابیده | log بادوام و ACK کاربردی |
| Push hint + Pull | اعلان tree.changed سپس Pull | تأخیر کم با بازیابی ساده | دو مکانیسم | Pull در شروع، reconnect و Sync دستی |
| USB بسته‌ای | Export و Import با فایل | مستقل از شبکه و Cloud | تحویل و Receipt انسانی | Package ID، manifest و Import مشترک |

**پیشنهاد:** Tree و Receipt با Pull، Measurement با Push، و اعلان تغییر اختیاری. برای ۲۰ موبایل و polling هر ۶۰ ثانیه، حدود ۰٫۳۳ درخواست بر ثانیه داریم؛ این فقط مثال هزینه polling است، نه تنظیم تضمین‌شده Android. به‌روزرسانی چند میلی‌ثانیه‌ای درخت در میدان از اسناد لازم نمی‌شود.

در Android، Local Storage مبنای خواندن UI باشد و صف پایدار Sync با محدودیت‌های سیستم‌عامل هماهنگ شود. WorkManager برای کار پایدار قابل بررسی است، اما timer دقیق یا تضمین اجرای فوری نیست. انتقال کاربرمحور بزرگ باید با API و سیاست نسخه هدف Android پیاده شود. [راهنمای رسمی Offline-first](https://developer.android.com/topic/architecture/data-layer/offline-first)

Push ابری نباید شرط درستی باشد؛ پیام‌ها عمر و سیاست نگهداری دارند. در طراحی پیشنهادی، حتی حذف تمام اعلان‌ها فقط تأخیر ایجاد می‌کند و Sync بعدی وضعیت را بازیابی می‌کند. [طول عمر پیام در FCM](https://firebase.google.com/docs/cloud-messaging/customize-messages/setting-message-lifespan)

## داده‌ها را یکسان Sync نکنید

| دسته | نویسنده | مدل | سیاست تعارض |
|---|---|---|---|
| Tree و Machine Spec | مرجع Desktop/Site | revision و Delta | optimistic concurrency یا بازبینی |
| Raw Measurement | دستگاه تولیدکننده تا نهایی‌شدن | immutable insert | ID یکسان/hash متفاوت = reject |
| وضعیت انتقال | هر سمت برای state خودش | state machine و Receipt | ACK نهایی مرجع دریافت |
| ترتیب مسیر و فعال‌بودن محلی | کاربر/دستگاه موبایل | LocalOverride | update درخت آن را overwrite نکند |
| Threshold رسمی | کارشناس مجاز در مرجع | نسخه‌دار و auditable | تأیید صریح، بدون LWW خاموش |
| Feature و Plot cache | Processing Engine | مشتق و قابل بازسازی | کلید نسخه الگوریتم و input hash |
| Annotation آینده | چند کاربر احتمالی | append یا CRDT مناسب | وابسته به نوع داده |

## Delta و Snapshot قابل اعتماد

1. مرجع برای هر تغییر پذیرفته‌شده در یک scope، change sequence بادوام می‌سازد.
2. اولین Sync یک Snapshot سازگار با `snapshot_watermark` می‌گیرد. صفحات باید متعلق به همان Snapshot باشند.
3. موبایل صفحه‌ها را در staging ذخیره می‌کند؛ پس از validation، Snapshot را اتمی فعال می‌کند.
4. Pull بعدی فقط تغییرات پس از watermark را می‌گیرد. ذخیره Delta و cursor در یک تراکنش محلی انجام شود.
5. اعلان Push هرگز cursor را جلو نمی‌برد؛ فقط درخواست Pull ایجاد می‌کند.
6. اگر cursor منقضی یا scope عوض شده باشد، Full Snapshot جدید دریافت شود؛ Outbox، Measurementها و LocalOverride جدا باقی بمانند.

cursor زمان دستگاه نیست. حذف رکورد با tombstone منتقل می‌شود؛ نبودن رکورد در یک صفحه نشانه حذف نیست. `removed_from_scope` با `deleted_in_authority` فرق دارد. انتقال یک Node به بیرون شاخه، History قبلی موبایل را قطع نمی‌کند.

برای Sync جزیی درخت، Parentهای لازم به عنوان structural context همراه Snapshot بیایند؛ دسترسی به فرزند نباید به‌اشتباه کل محتوای شاخه‌های دیگر را افشا کند. Scope هم مرز عملکردی است و هم باید با مجوز سرور کنترل شود.

## از نسخه ساده تا Vector Clock

| ابزار | چه چیزی می‌دهد؟ | چه چیزی نمی‌دهد؟ | تناسب SeeNora |
|---|---|---|---|
| Wall clock / updated_at | زمان انسانی تقریبی | علیت معتبر با ساعت ناهماهنگ | نمایش زمان، نه تشخیص تعارض |
| Scalar revision + CAS | رد تغییر مبتنی بر نسخه کهنه در یک مرجع | علیت چند نویسنده مستقل | انتخاب A/B/C با ownership روشن |
| Lamport clock | ترتیب سازگار با happened-before در یک جهت | مقایسه اعداد، concurrency را اثبات نمی‌کند | مرتب‌سازی log، نه حل تعارض کامل |
| HLC | ترکیب مؤلفه منطقی و زمان برای ترتیب عملی | تشخیص کامل هم‌زمانی مستقل | اختیاری در سامانه بزرگ‌تر |
| Vector Clock | تشخیص causally-before و concurrent | حل معنایی یا دوام داده | مدل E |
| Dotted Version Vector | dot عملیات + context علّی | حذف نیاز به قواعد دامنه | برای log چندمرجعی پیشرفته |
| CRDT | همگرایی برای نوع و عملیات تعریف‌شده | صحت تمام invariantهای دامنه | مجموعه/annotation مناسب، نه Tree دلخواه |

در مرجع مرکزی، `If-Match` با ETag یا `expected_revision` باعث می‌شود ویرایش براساس نسخه قدیمی بی‌صدا تغییر جدید را از بین نبرد. استفاده از Conditional Request مطابق معنای HTTP است؛ قواعد Merge متعلق به SeeNora هستند. [RFC 9110، If-Match](https://www.rfc-editor.org/rfc/rfc9110.html#name-if-match)

## مثال عددی Vector Clock

فرض کنید دو دسکتاپ A و B اجازه ویرایش آفلاین Spec یک Point را دارند. نسخه مشترک:

```text
V0 = {A: 2, B: 1}
A changes bearing -> VA = {A: 3, B: 1}
B changes bearing -> VB = {A: 2, B: 2}
```

برای `X <= Y` باید تمام مؤلفه‌های X کوچک‌تر یا مساوی Y باشند. `X < Y` یعنی علاوه بر این، حداقل یک مؤلفه کوچک‌تر باشد. مؤلفه غایب در یک مجموعه actor شناخته‌شده صفر فرض می‌شود.

در مثال، `VA[A] > VB[A]` ولی `VA[B] < VB[B]`؛ هیچ‌کدام دیگری را dominate نمی‌کند. پس تغییرها concurrent هستند. ساعت فیزیکی یا «آخرین فایل رسیده» این حقیقت را تغییر نمی‌دهد.

اگر کارشناس روی A پس از دیدن هر دو نسخه مقدار نهایی را انتخاب کند:

```text
causal_context = max(VA, VB) = {A: 3, B: 2}
resolved_version_on_A = {A: 4, B: 2}
```

نسخه حل‌شده هر دو را dominate می‌کند. صرفاً ذخیره max بدون ثبت رویداد حل تعارض و بدون مقدار تصمیم‌گرفته‌شده کافی نیست. تا پیش از حل، دو sibling حفظ می‌شوند و Screening وابسته به Spec متعارض باید محدود یا بر snapshot معتبر قبلی متکی شود.

Vector clock سابقه وابستگی نسخه‌ها را حمل می‌کند و مثال شناخته‌شده آن در Dynamo توضیح داده شده است؛ این مجموعه الگوریتم Dynamo را عیناً برای SeeNora تجویز نمی‌کند. [مقاله اصلی Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

## سیاست Merge برای دامنه محصول

| وضعیت | سیاست پیشنهادی |
|---|---|
| دو موبایل از یک Node برداشت می‌کنند | دو Measurement؛ حذف خودکار یکی خطاست |
| Retry همان Measurement و همان hash | همان Receipt قبلی؛ بدون رکورد جدید |
| ID یکسان و محتوا متفاوت | Conflict سخت، حفظ نسخه فرستنده، گزارش علت |
| Desktop نام Node را عوض کرده، موبایل برداشت قدیمی دارد | دریافت با ID ثابت و snapshot زمان برداشت |
| Node غیرفعال/soft-deleted شده | Raw حفظ شود؛ در History آرشیوی یا Quarantine؛ بدون ساخت Node جدید خودکار |
| ترتیب موبایل و Tree دسکتاپ تغییر کرده | merge لایه مرجع + override محلی؛ تعارض هم‌فیلد نداریم |
| دو کارشناس فیلدهای مستقل Spec را عوض کرده‌اند | در E، merge سه‌طرفه فقط با base و invariant معتبر |
| دو مقدار برای Bearing، RPM مرجع یا Threshold | بازبینی انسانی، audit و نسخه حل تعارض |
| دو Move هم‌زمان چرخه در Tree می‌سازند | رد/تعلیق تغییر ساختاری؛ CRDT ساده کافی نیست |
| حذف هم‌زمان با ویرایش | tombstone و conflict قابل مشاهده؛ restore باید عملیات صریح باشد |
| دو Tag یکسان در نوشتن مستقل آفلاین | تعارض یکتایی پس از reconnect؛ انتشار نهایی تا حل متوقف شود |

**LWW** فقط برای فیلد کم‌اهمیت مانند آخرین tab بازشده قابل بررسی است. برای سیگنال خام، انتساب Node، تنظیمات پردازش یا Threshold باعث از دست رفتن معنی داده می‌شود. حتی LWW مبتنی بر HLC نیز انتخاب یک winner است، نه اثبات درستی مقدار.

**CRDT** برای set برچسب‌ها یا یادداشت مشترک می‌تواند هزینه Merge را کم کند، اما انتخاب نوع مهم است: OR-Set برای حذف هم‌زمان با افزودن semantics مشخص دارد؛ register چندمقداری تعارض را نگه می‌دارد. همگرایی CRDT در محدوده عملیات و فرض‌های همان نوع است. [ارائه پژوهشی Shapiro درباره CRDT](https://www.microsoft.com/en-us/research/video/strong-eventual-consistency-and-conflict-free-replicated-data-types/)

## خطرهای عملی Vector Clock و replicaها

- `actor_id` شناسه نصب replica است؛ شناسه کاربر یا Data Collector جای آن نیست.
- شمارنده و mutation باید در یک تراکنش بادوام ذخیره شوند. Restore قدیمی نباید باعث بازاستفاده از dot شود؛ پس از reset، incarnation جدید لازم است.
- vector به تعداد actorها رشد می‌کند. برای v3 اگر تنها یک نویسنده دارید، این هزینه بی‌فایده است.
- retirement یک replica مجوز حذف فوری component یا tombstone نیست. باید replicaهای فعال تغییر را دیده باشند یا replica قدیمی در بازگشت ملزم به bootstrap شود.
- log compaction و tombstone retention براساس بیشترین مدت آفلاین و قرارداد membership تعیین شود؛ صرف گذشت چند روز proof ایمنی نیست.
- در replicated operation log، anti-entropy و تشخیص gap لازم‌اند؛ مرتب رسیدن شبکه را فرض نکنید.

## تضمین واقعی تحویل

### سازگاری در زمان قطع شبکه

برای Root Acquisition، دسترس‌پذیری محلی و read-your-writes لازم است: تکنسین باید برداشت ذخیره‌شده خودش را فوراً ببیند. دید مشترک Desktopها بعد از Sync به‌روز می‌شود. در A/B، تغییرات مرجع Tree در یک محل serial می‌شوند و موبایل ممکن است snapshot کهنه داشته باشد؛ «کهنه بودن مجاز» با «گم‌شدن تغییر» فرق دارد.

اگر در E هنگام partition هر دو سمت مستقل تغییر ساختاری را بپذیرند، نمی‌توان هم‌زمان یک دید خطی و فوری مشترک از همان داده تضمین کرد. این trade-off را در سطح هر operation تعیین کنید؛ برچسب ساده CP یا AP برای تمام محصول، رفتار واقعی Measurement، Tree و تحلیل را توضیح نمی‌دهد.

برای HA مرجع در B، primary/standby یا consensus با quorum می‌تواند یک writer معتبر انتخاب کند؛ گوشی آفلاین عضو quorum نباشد. اقلیت جداشده نباید بی‌قید همچنان Receipt مرجع بدهد. lease و lock نیز باید fencing داشته باشند؛ timer محلی یا expire شدن قفل روی یک PC به‌تنهایی مانع split-brain نیست. قفل شبکه‌ای برای کل مدت مأموریت میدانی راه‌حل مناسبی برای تغییر Tree نیست؛ snapshot نسخه‌دار و اعتبارسنجی بازگشت بهتر است.

در E، anti-entropy می‌تواند ابتدا high-watermark هر actor، سپس rangeهای مفقود عملیات را مقایسه کند. Merkle tree برای مقایسه مجموعه بزرگ hashها یک بهینه‌سازی احتمالی است، نه پیش‌نیاز v3؛ checksum manifest و cursor برای مقیاس اولیه ساده‌ترند. همچنین content hash اختلاف bytes را نشان می‌دهد، نه اینکه کدام تغییر از نظر دامنه معتبرتر است.

### اثر یک‌باره در مقصد

طراحی پیشنهادی **at-least-once delivery با اثر idempotent** است. از ادعای «exactly-once در تمام شبکه، دیسک، صف و UI» استفاده نکنید. کلید یکتا و تراکنش باعث یک‌بار اثر در مقصد می‌شوند، در حالی که پیام می‌تواند چند بار برسد.

Outbox باید با داده مرجع در یک commit ایجاد شود. در مقصد، Inbox/Measurement و Receipt باید با هم ثبت شوند. Event پردازش نیز در همان commit در Outbox مقصد قرار گیرد تا crash بین ذخیره Measurement و ساخت job باعث جاافتادن تحلیل نشود. وضعیت پردازش مستقل از وضعیت انتقال باقی می‌ماند.