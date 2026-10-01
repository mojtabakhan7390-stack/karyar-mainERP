<div class="cover">
<h1 class="title">معماری Karyar</h1>
<p class="subtitle">ERPNext + Karyar Agent Platform</p>
<p class="meta">سند معماری — نسخه ۱٫۰ (بازبینی و به‌روزرسانی نهایی) — مهر ۱۴۰۵ / اکتبر ۲۰۲۶</p>
<p class="meta">این نسخه همه تصمیم‌های CR-01 تا CR-04 و تصمیم‌های قطعی بازبینی نهایی را یکپارچه کرده است.</p>
</div>

## فهرست

- بخش صفر: خلاصه اجرایی، وضعیت مخزن، تصمیم‌های قطعی
- بخش ۱ تا ۳۰: معماری
- بخش ۳۱: مثال‌های سرتاسری
- بخش ۳۲: جدول تصمیم‌های معماری
- بخش ۳۳: تعارض‌های حل‌شده و تنش‌های باقی‌مانده
- بخش ۳۴: خروجی نهایی و نقشه راه

---

# بخش صفر — خلاصه اجرایی

## ۰.۱ Karyar در یک پاراگراف

Karyar یک پلتفرم **ERP + AI Agents** است.

**ERPNext موتور اصلی منطق و داده کسب‌وکار است** و منبع نهایی حقیقت (Final Source of Truth) باقی می‌ماند. Karyar **ERP دوم نیست.** نقش آن لایه هوشمند، کنترل‌کننده و رابط است و این بخش‌ها را در بر می‌گیرد:

- ایجنت‌ها و اجرای آن‌ها (Agent Runtime)
- گفت‌وگوی زنده
- Workspace مشترک انسان و ایجنت
- کنترل، مجوز، ممیزی و ثبت مصرف AI
- نگهداری وضعیت فرایند

**n8n فقط هماهنگی گام‌ها (Orchestration) و اجرای Integrationها را انجام می‌دهد.** هر تصمیم کسب‌وکاری، مجوز و عملیات نهایی از Karyar می‌گذرد و در نهایت با منطق استاندارد خود ERPNext ثبت می‌شود.

**اصل راهنمای کل سند:**

> هر کاری که ERPNext به‌صورت امن، استاندارد و قابل اتکا انجام می‌دهد، به ERPNext سپرده می‌شود. Karyar فقط در جایی منطق اضافه می‌کند که برای Agent، AI یا کنترل واقعاً لازم است.

## ۰.۲ وضعیت فعلی مخزن

| مورد | یافته |
|---|---|
| مخزن | `karyar-mainERP`؛ fork مستقیم `frappe/erpnext` |
| شاخه پایه | `develop` (نسخه `17.0.0-dev`، ناپایدار) |
| وابستگی‌ها | Frappe ‏`>=17.0.0-dev,<18`، Python ‏3.14 یا بالاتر، Node ‏24 |
| سفارشی‌سازی Karyar | هیچ؛ فقط مستندات در `docs/karyar/` |
| حجم | حدود ۴۳۰ هزار خط Python و ۸۷ هزار خط JS؛ ۲۱ ماژول |

**نقاط توسعه استانداردی که Karyar از آن‌ها استفاده می‌کند:**

- `doc_events`
- DocType ‏Webhook با امضای HMAC
- `scheduler_events` و صف‌های سفارشی RQ
- `frappe.db.after_commit`
- `has_permission`، `permission_query_conditions`، User Permission و permlevel
- `extend_doctype_class`
- Custom Field و Property Setter به‌صورت fixture
- `override_doctype_dashboards`
- `regional_overrides` و `naming_series_variables`
- `ToDo` و `assign_to`، Assignment Rule، Notification
- Email Account، SMS Settings، `Communication` (با medium‌های Chat، SMS و Phone)
- `Version` (track changes)
- OAuth Client
- `publish_realtime`

**محدودیت‌هایی که بر طراحی اثر دارند:**

1. **Frappe Workflow ماشین حالتِ تک‌سند است.** پیش از وجود سند کار نمی‌کند، تأیید «M از N» ندارد و تأیید را به hash محتوا گره نمی‌زند.
2. **`frappe.get_all` مجوزها را نادیده می‌گیرد.**
3. **سرور Frappe همگام (WSGI) است.** فراخوانی‌های طولانی مدل AI باید در Worker اجرا شوند.
4. **چارت حساب فقط از پوشه ERPNext بارگذاری می‌شود.** راه حل: `create_charts(custom_chart=…)`.
5. **برخی گزارش‌ها SQL خام دارند و مجوزشان فقط بر اساس Role است.** پس فقط فهرست گزینش‌شده‌ای از گزارش‌ها در دسترس ایجنت قرار می‌گیرد.
6. **ERPNext نسخه ۱۶ روی PostgreSQL پشتیبانی رسمی ندارد.**
7. **Contact در v16 فقط فیلدهای تلفن و ایمیل دارد** (`phone_nos`، `email_ids`، `mobile_no`، `unsubscribed`). Lead فیلد `whatsapp_no` دارد. برای Telegram، Instagram و بله فیلد استانداردی وجود ندارد.

> **نکته ساختاری:** کد Karyar نباید داخل این fork نوشته شود. بخش ۲۳ ساختار درست مخازن را آورده است. این سند موقتاً در `docs/karyar/` نگهداری می‌شود.

## ۰.۳ تصمیم‌های قطعی (مبنای این نسخه)

| حوزه | تصمیم |
|---|---|
| نقش ERPNext | موتور اصلی منطق و داده کسب‌وکار و منبع نهایی حقیقت. **ERPNext-first:** هرجا قابلیت استاندارد دارد، همان استفاده می‌شود |
| نقش Karyar | ایجنت، گفت‌وگو، Workspace انسانی، کنترل، مجوز، ممیزی، مصرف AI، و وضعیت فرایند |
| نقش n8n | **فقط** هماهنگی گام‌ها و اجرای Automation و Integration |
| در کنترل Karyar | گفت‌وگوی زنده، اجرای ایجنت، تصمیم کسب‌وکاری، مجوز، Human Task، عملیات نهایی |
| Human Task | Workspace مشترک انسان و ایجنت. مبتنی بر گفت‌وگو. انسان می‌تواند تغییر دهد، Pause/Resume کند، واگذار کند، جلو/عقب ببرد و از ایجنت‌های دیگر کمک بگیرد |
| تغییرهای حساس | **پیشنهاد تغییر (Change Proposal) با Diff**؛ ثبت فقط پس از تأیید صریح انسان |
| نسخه‌داری و ممیزی | همه تغییرها، تأییدها، تصمیم‌ها و واگذاری‌ها |
| تأیید (Approval) | از فاز ۱: تک‌نفره، چندنفره، M از N، ترتیبی و موازی |
| واگذاری بین ایجنت‌ها | داخل Karyar. اختیار کار واگذارشده هرگز بیشتر از Principal اولیه نیست |
| مجوز | در کد، با زنجیره Tenant → Principal/User → Agent → Responsibility → Capability → Workflow Step → ERPNext Permission. **Prompt هرگز مجوز یا تأیید ایجاد نمی‌کند** |
| داده | بدون دسترسی خام به پایگاه داده یا SQL دلخواه. گزارش‌ها از ERPNext Report و ابزارهای کنترل‌شده |
| مصرف AI | ثبت برای Tenant، Agent، Workflow، User و Task |
| چندمستأجری | جداسازی Tenantها. Credential و Integrationها Tenant-aware و امن |
| RPA | فقط برای سیستم‌های بدون API. **هرگز برای ERPNext.** کپچا و OTP به Human Task می‌روند |
| کانال‌های مشتری | در CRM/ERP. برای کانال‌هایی که فیلد استاندارد ندارند، جدول فرزند سفارشی `Contact Channel` روی Contact. **`Contact Identity` حذف شد** |
| اجرای خودکار | فقط بر اساس Policy از پیش تعریف‌شده و قابل ممیزی. ایجنت تصمیم یا اختیار جدید نمی‌سازد |
| پیش‌ثبت (Draft) | به‌صورت پیش‌فرض فقط در Karyar. سند نهایی کسب‌وکار قبل از تأیید در ERPNext ساخته نمی‌شود. پیش‌نویس در ERPNext فقط به‌عنوان قابلیت اختیاری و کنترل‌شده |
| Preview و Re-Approval | hash پیش‌نمایش، تأیید، و تأیید مجدد در صورت تغییر داده مهم |
| پس از ثبت نهایی | داده اصلی فقط از ERPNext خوانده می‌شود. Karyar فقط فرایند، Context و ممیزی خود را نگه می‌دارد |
| مرز Tenant | Tenant همان مرز کسب‌وکار یا پروژه است. محدوده‌هایی مثل شعبه با User Permission کنترل می‌شوند |

---

# ۱. نمای کلی معماری

Karyar از **چهار لایه** و یک **لایه عمودی** ساخته می‌شود:

1. **لایه تعامل:**
   - کانال‌های مشتری: ویجت وب، پیام‌رسان‌ها، و در آینده صوت
   - **Karyar Workspace** برای کارکنان
   - همه از طریق **درگاه کانال (Channel Gateway)** داخل Karyar
2. **لایه هوشمندی و کار انسانی** (داخل Karyar):
   - اجرای ایجنت (Agent Runtime)
   - گفت‌وگو (Conversation)
   - Human Task Workspace
   - پیشنهاد تغییر (Change Proposal)
   - درخواست تأیید (Approval Request)
   - کار واگذارشده (Agent Task)
   - پرونده (Karyar Case)
3. **لایه کنترل** (داخل Karyar):
   - موتور سیاست و مجوز (Policy/Permission Engine)
   - اجراکننده ابزار (Tool Executor)
   - رجیستری توانمندی و ابزار
   - گام ثبت نهایی (Promote)
   - Karyar API v1
   - انتشار رویداد
   - درگاه AI (AI Gateway)
4. **لایه داده و منطق کسب‌وکار:** ERPNext، HRMS و `karyar_iran` روی Frappe. این لایه **منبع نهایی حقیقت** است.

- **لایه عمودی:** ممیزی، نسخه‌داری، مصرف AI و اسرار.
- **بیرون از هسته:**
  - **n8n**: ترتیب مراحل فرایند، و اجرای Integrationها (پیامک انبوه، APIهای بیرونی، و در آینده RPA)
  - ارائه‌دهندگان AI
  - سرویس‌های بیرونی

**تقسیم کار در یک جمله:**

- **ERPNext** کسب‌وکار را ثبت و محاسبه می‌کند.
- **Karyar** می‌فهمد، کنترل می‌کند، پیشنهاد می‌دهد و پس از تصمیم انسان (یا Policy مصوب) اجرا می‌کند.
- **n8n** ترتیب گام‌ها را جلو می‌برد و Integrationها را اجرا می‌کند.

# ۲. اصول بنیادین

| # | اصل | پیامد فنی |
|---|---|---|
| P1 | ERPNext منبع نهایی حقیقت و موتور منطق کسب‌وکار است | محاسبه، اعتبارسنجی و ثبت با کنترلرهای ERPNext انجام می‌شود |
| P2 | **ERPNext-first** | منطق موازی ساخته نمی‌شود. اگر ERPNext قابلیت استاندارد دارد، Karyar از همان استفاده می‌کند (بخش ۱۲.۱) |
| P3 | Karyar ERP دوم نیست | Karyar فقط پیش‌ثبت، Context، فرایند و ممیزی نگه می‌دارد |
| P4 | بدون دسترسی خام به پایگاه داده یا SQL دلخواه | فقط ابزارهای تایپ‌شده و ثبت‌شده، و گزارش‌های فهرست‌شده |
| P5 | مجوز در کد اعمال می‌شود، نه در Prompt | زنجیره مجوز (بخش ۱۱). Prompt نه مجوز می‌سازد، نه تأیید |
| P6 | مفاهیم از هم جدا هستند: ایجنت، نقش کاری (Responsibility)، توانمندی (Capability)، مجوز و گردش‌کار | گام فرایند به نقش کاری اشاره می‌کند، نه به ایجنت مشخص |
| P7 | تصمیم شخصی ایجنت وجود ندارد | هر اقدام نهایی به یک تصمیم انسانی قابل ردیابی وصل است: تأیید لحظه‌ای یا Policy مصوب |
| P8 | پیش‌ثبت در Karyar، ثبت نهایی در ERPNext | سند نهایی کسب‌وکار قبل از تأیید ساخته نمی‌شود. پیش‌نویس ERPNext فقط به‌صورت اختیاری و قفل‌شده |
| P9 | آنچه تأیید شد همان است که اجرا می‌شود | تأیید به hash پیش‌ثبت و پیش‌نمایش گره می‌خورد. هر تغییر مهم، تأیید مجدد لازم دارد |
| P10 | هر تغییر حساس، پیشنهاد تغییر با Diff است | ثبت فقط پس از تأیید صریح انسانِ احرازشده |
| P11 | واگذاری کنترل‌شده | ایجنت فقط طبق Delegation Policy کار می‌سپارد. اختیار هرگز افزایش نمی‌یابد |
| P12 | n8n فقط هماهنگی و اجراست | n8n مجوز، تصمیم، وضعیت پرونده و داده پیش‌ثبت ندارد |
| P13 | مستقل از ارائه‌دهنده AI | AI Gateway با نام مستعار مدل (alias) |
| P14 | همه‌چیز قابل ردیابی است | `correlation_id`، Audit Log معنایی، نسخه‌ها |
| P15 | Tenant مرز جداسازی است | سایت و پایگاه داده جدا، Credential جدا، n8n و Integrationهای Tenant-aware |
| P16 | هسته ERPNext دست‌نخورده می‌ماند | Hook، Custom Field، اپ جدا. وصله ضروری مستند می‌شود |
| P17 | ساده شروع کن، درست مرز بکش | یک اپ Frappe با ماژول‌ها. سرویس جدا فقط هنگام نیاز |

# ۳. نمودار معماری سیستم

<div class="arch">
  <div class="arch-main">
    <div class="layer l-channel"><div class="lname">کانال‌ها و رابط‌ها</div>
      <div class="boxes"><span>ویجت وب</span><span>بله / تلگرام</span><span>واتس‌اپ / اینستاگرام</span><span>صوت (آینده)</span><span class="hl">Karyar Workspace (کارکنان)</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-gw"><div class="lname">Channel Gateway (داخل Karyar) — گفت‌وگوی زنده</div>
      <div class="boxes"><span>Adapterها</span><span>شناسایی مخاطب از Contact Channel</span><span>پیام استاندارد</span><span>تأیید امضا</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-brain"><div class="lname">هوشمندی و کار انسانی (Karyar)</div>
      <div class="boxes"><span class="hl">Agent Runtime</span><span class="hl">Human Task Workspace</span><span>Change Proposal</span><span>Approval Request</span><span>Agent Task (واگذاری)</span><span>Karyar Case (پیش‌ثبت)</span></div></div>
    <div class="arrow">▼</div>
    <div class="layer l-control"><div class="lname">کنترل (Karyar) — هیچ مسیری از کنار این لایه رد نمی‌شود</div>
      <div class="boxes"><span class="hl">Policy / Permission</span><span class="hl">Tool Executor</span><span class="hl">Promote</span><span>رجیستری ابزار</span><span>Karyar API v1</span><span>emit_event</span><span>AI Gateway</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-erp"><div class="lname">منطق و داده کسب‌وکار — Frappe Site (یک سایت برای هر Tenant)</div>
      <div class="boxes"><span class="hl">ERPNext (Final Truth)</span><span>HRMS</span><span>karyar_iran</span><span>Contact Channel</span><span>Frappe: مجوز، Version، Webhook، ToDo، Email، SMS</span></div></div>
    <div class="arrow">▼</div>
    <div class="layer l-infra"><div class="lname">زیرساخت</div>
      <div class="boxes"><span>MariaDB (پایگاه داده جدا برای هر Tenant)</span><span>Redis</span><span>فایل‌ها / S3</span><span>Workerها و Scheduler</span></div></div>
  </div>
  <div class="arch-side">
    <div class="side s-audit"><div class="lname">عمودی</div><span>Audit Log</span><span>نسخه‌های پیش‌ثبت</span><span>AI Usage</span><span>correlation_id</span><span>مدیریت اسرار</span></div>
    <div class="side s-ext"><div class="lname">بیرون از هسته</div><span>n8n (برای هر Tenant): هماهنگی گام‌ها و اجرای Integration</span><span>RPA Worker (آینده، فقط بدون API)</span><span>ارائه‌دهندگان AI</span><span>APIهای بیرونی (پیامک، مودیان، بانک)</span></div>
  </div>
</div>

**جهت جریان:**

- **مشتری ← Karyar:** مشتری با ایجنت در Karyar گفت‌وگو می‌کند. Karyar داده را به‌صورت **پیش‌ثبت** در Case نگه می‌دارد.
- **Karyar ← n8n:** نقطه‌های عطف فرایند به‌صورت رویداد به n8n می‌رسند.
- **n8n ← Karyar API:** ‏n8n با Karyar API ‏Human Task می‌سازد، ایجنت را فرا می‌خواند یا Promote را درخواست می‌کند.
- **Karyar ← ERPNext:** ‏Karyar پس از بررسی مجوز و تأیید، سند را با منطق خود ERPNext ثبت می‌کند.
- **ERPNext ← n8n:** رویدادهای ERPNext (Webhook استاندارد Frappe) دوباره به n8n می‌روند تا مرحله بعد شروع شود.

# ۴. معماری اجزا

| جزء | مسئولیت | چرا در Karyar است (و نه ERPNext) | فاز |
|---|---|---|---|
| **Channel Gateway** | گفت‌وگوی زنده، Adapter کانال‌ها، شناسایی مخاطب از `Contact Channel` | ERPNext چت زنده و ایجنت ندارد | ۱ |
| **Conversation** | پیام‌ها و خلاصه تعامل. خلاصه پس از اتصال به مخاطب در Timeline خود ERPNext ثبت می‌شود | Context ایجنت. تاریخچه رسمی در ERPNext (`Communication`) | ۱ |
| **Agent Runtime** | اجرای یک نوبت ایجنت: Context، مدل، ابزارها، اعتبارسنجی خروجی | مخصوص AI | ۱ |
| **Karyar Case** | پرونده: مرحله، پیش‌ثبت (Draft) و نسخه‌های آن، ارجاع به اسناد ERPNext | پیش‌ثبتِ قبل از وجود سند ERPNext | ۱ |
| **Human Task Workspace** | فضای مشترک انسان و ایجنت: گفت‌وگو، پنل، اقدام‌ها | Frappe Workflow پیش از وجود سند و به‌صورت گفت‌وگویی کار نمی‌کند. **تخصیص و اعلان از `ToDo` و Notification استاندارد Frappe** | ۱ |
| **Change Proposal** | پیشنهاد تغییر با Diff و تأیید صریح | کنترل AI | ۱ |
| **Approval Request** | تأیید تک‌نفره، چندنفره، M از N، ترتیبی، موازی، و گره خوردن به hash | Frappe Workflow ‏M از N و hash ندارد و پیش از سند کار نمی‌کند | ۱ |
| **Agent Task** | واگذاری بین ایجنت‌ها و به Integrationها، و پیگیری آن | کنترل AI | ۱ |
| **Policy / Permission Engine** | زنجیره مجوز، Approval Policy، Auto-Execution Policy، Delegation Policy | کنترل AI. **در انتها به مجوز Frappe ختم می‌شود** | ۱ |
| **Tool Executor + Registry** | تنها مسیر اجرای ابزار: بررسی مجوز، idempotency، فیلتر خروجی، ممیزی | کنترل AI | ۱ |
| **Promote** | تنها مسیر ثبت نهایی در ERPNext با کنترلرهای ERPNext | دروازه کنترل | ۱ |
| **Karyar API v1** | API پایدار برای n8n، Workspace و اپ‌ها | جدا کردن مصرف‌کننده‌ها از جزئیات ERPNext | ۱ |
| **emit_event** | انتشار رویدادهای Karyar به n8n پس از commit. رویدادهای ERPNext با Webhook استاندارد Frappe | سبک؛ Outbox اختصاصی موکول به آینده | ۱ |
| **AI Gateway** | انتزاع ارائه‌دهنده و ثبت مصرف و هزینه | مخصوص AI | ۱ (به‌صورت کتابخانه) |
| **Audit** | ممیزی معنایی (چه کسی، از طرف چه کسی، چه پیشنهادی، چه تأییدی). تغییرات میدانی اسناد ERPNext با `Version` استاندارد | مکمل `Version`، نه تکرار آن | ۱ |
| **n8n** | ترتیب مراحل فرایند و اجرای Integration | — | ۱ |
| **RPA Worker** | سیستم‌های بدون API | — | آینده |
| **Control Plane** | ثبت Tenant، راه‌اندازی، تجمیع مصرف | — | آینده (ابتدا با اسکریپت) |

# ۵. معماری ایجنت

## ۵.۱ مفاهیم جداشده

| مفهوم | تعریف | مثال |
|---|---|---|
| **Capability (توانمندی)** | یک توانایی اتمی که به یک یا چند ابزار نگاشت می‌شود و سطح ریسک دارد | `crm.lead.draft`، `report.sales_summary.read` |
| **Responsibility (نقش کاری)** | بسته‌ای از توانمندی‌ها و دستورالعمل حرفه‌ای. **هم به ایجنت و هم به انسان** قابل انتساب است | «پذیرش مشتری»، «فروش»، «حسابداری»، «تأییدکننده مالی» |
| **Agent (ایجنت)** | یک شخصیت اجرایی با یک یا چند Responsibility، کانال‌ها و مدل | Ava، Hanna، Arman، «سارا» (چند نقش) |
| **Role Assignment** | تعیین اینکه در هر Tenant، هر Responsibility با کدام ایجنت(ها) و کدام انسان(ها) است | شرکت C: فروش، CRM و حسابداری با «سارا»؛ تأیید مالی با «خانم احمدی» |
| **Frappe Role / Permission** | مجوز واقعی روی داده (DocType، رکورد، فیلد) | Sales User، Accounts Manager |

- **نام‌گذاری:** در کد `Responsibility` و در رابط کاربری «نقش کاری». این انتخاب برای جلوگیری از اشتباه با Role در Frappe است.
- **گام‌های فرایند** به Responsibility اشاره می‌کنند، نه به ایجنت. پس ترکیب یا تفکیک نقش‌ها هیچ تعریف فرایندی را تغییر نمی‌دهد.

## ۵.۲ تعریف ایجنت (`Karyar Agent`)

- **شخصیت و زبان**
- **Responsibilityها:** توانمندی‌ها اجتماع (union) آن‌هاست و قابل محدودسازی است.
- **کانال‌های مجاز و نوع مخاطب**
- **پروفایل مدل:** یک نام مستعار مثل `fast` یا `smart`.
- **دستورالعمل نسخه‌دار:** از چند لایه ترکیب می‌شود: پایه سکو، نقش کاری، Tenant و گام فرایند.
- **هویت اجرایی:** یک کاربر سرویس Frappe بدون امکان ورود.
- **محدودیت‌ها:** حداکثر گام در هر نوبت، حداکثر token و سقف هزینه.
- **حافظه:**
  - خلاصه گفت‌وگوی جاری
  - خلاصه چند گفت‌وگوی آخر **همان مخاطب**
  - **واقعیت‌های کسب‌وکاری همیشه از ERPNext خوانده می‌شوند** (بخش ۲۲)

## ۵.۳ حالت‌های اجرا

1. **گفت‌وگوی زنده با مشتری:** درون Karyar. نتیجه به Case و رویداد تبدیل می‌شود.
2. **کار فرایندی:** n8n با `agents.run_task` آن را آغاز می‌کند، به‌صورت ناهمگام. پایان کار با رویداد `agent.task.completed` اعلام می‌شود.
3. **دستیار Workspace:** در Human Task، از طرف کاربر مسئول عمل می‌کند (بخش ۸).
4. **کار واگذارشده:** یک `Agent Task` که ایجنت یا انسان دیگری طبق Policy ساخته است (بخش ۱۰).

## ۵.۴ چرخه یک نوبت ایجنت

<pre class="ltr">
1. Load agent config → effective capabilities = Responsibilities ∩ step/task scope ∩ principal permissions
2. Build context: conversation window + contact summaries + case refs (business facts via ERPNext tools)
3. AI Gateway.complete(alias, messages, allowed tool schemas, response schema?)  → UsageRecord
4. Each tool call → Tool Executor: policy check → execute (as principal) → filter output → audit
5. Bounded loop (max steps / tokens / time)
6. Validate output schema (one retry, else escalate to a Human Task)
7. Persist messages / proposals / audit; emit events
</pre>

- **محل اجرا:** Workerهای RQ با صف `karyar_agent`.
- **پخش زنده پاسخ:** با `publish_realtime`.
- **چرا runtime سبک خودمان:** یکپارچگی عمیق با هویت، مجوز و چندمستأجری Frappe، و کنترل کامل روی ممیزی.

## ۵.۵ ایجنت پیشنهاد می‌دهد؛ تصمیم متعلق به انسان است

| ایجنت **می‌تواند** | ایجنت **نمی‌تواند** |
|---|---|
| فهمیدن درخواست، خواندن داده مجاز، پیشنهاد تغییر (Change Proposal)، پیشنهاد تصمیم، آماده کردن پیش‌ثبت، اجرای تغییرِ **تأییدشده**، هماهنگی، و واگذاری طبق Policy | تأیید تغییر، تأیید مجوز (Approval)، رد، لغو، Pause، تغییر مسیر، Promote بدون تأیید یا Policy، و ساختن مجوز یا اختیار جدید |

**اجرای خودکار** (بدون انسان در لحظه) فقط وقتی مجاز است که یک **Auto-Execution Policy** مصوب آن را مجاز کرده باشد. ممیزی هر اجرای خودکار شامل نسخه Policy و کاربری است که آن را تعریف یا فعال کرده است.

# ۶. معماری Agent Builder و بسته‌ها (Pack)

**فاز ۱:** پیکربندی با فرم‌های Frappe انجام می‌شود (توسط تیم Karyar) و در قالب **Pack**های نسخه‌دار ذخیره می‌شود.

<pre class="ltr">
Karyar Pack (versioned, e.g. "clinic-basic@1.0.0")
├── responsibilities/*.json      capabilities + instructions
├── agents/*.json                persona, channels, model alias, responsibilities
├── role_assignments.json        tenant-overridable defaults (agents + humans)
├── policies/                    approval, auto-execution, delegation, retention
├── human_task_templates/*.json
├── reports.json                 report catalog entries
├── integration_targets.json     names + schemas (credentials NOT included)
├── n8n/                         workflow JSON + README per workflow
└── erp_customizations/          custom fields, property setters, print formats (fixtures)
</pre>

- **اعمال روی Tenant:** با `bench --site X karyar apply-pack <pack@version>`. تنظیمات اختصاصی هر Tenant به‌صورت override ادغام می‌شود. این همان سازوکار ۸۰/۲۰ است.
- **نقاط توسعه** برای اپ‌های دیگر: `karyar_tools`، `karyar_channel_adapters`، `karyar_ai_providers`، `karyar_report_providers`.
- **ساخت ابزار جدید همیشه کد است.** بازبینی می‌شود و نه از رابط کاربری ساخته می‌شود، نه توسط AI.

# ۷. Workflow و Orchestration (فاز ۱: n8n)

## ۷.۱ تصمیم

در فاز ۱، **Workflow Engine اختصاصی ساخته نمی‌شود.** ترتیب و مسیر مراحل فرایند در **n8n خودمیزبان** ساخته و اجرا می‌شود.

**اصل:** «n8n ترتیب کارها را تعیین می‌کند. Karyar تصمیم می‌گیرد، کنترل می‌کند و اجرا می‌کند. ERPNext ثبت می‌کند.»

## ۷.۲ مرز مسئولیت

| n8n | Karyar | ERPNext |
|---|---|---|
| Trigger (رویداد Webhook، Cron) | گفت‌وگوی زنده و اجرای ایجنت | منطق، محاسبه و اعتبارسنجی کسب‌وکار |
| ترتیب، انشعاب و مسیر بین مراحل | Human Task، Change Proposal، Approval | ثبت نهایی و وضعیت سند |
| فراخوانی Karyar API | مجوز، Policy، Promote | گزارش‌های استاندارد |
| اجرای Integration (پیامک انبوه، APIهای بیرونی، RPA در آینده) | وضعیت پرونده (`Karyar Case`) و پیش‌ثبت | `Version`، `Communication`، `ToDo` |
| retry فراخوانی بیرونی و Error Workflow | ممیزی و مصرف AI | — |

## ۷.۳ قواعد سخت n8n

1. n8n **فقط** با Karyar API v1 کار می‌کند. دسترسی مستقیم به ERPNext (نود ERPNext یا `/api/resource`) و پایگاه داده **ممنوع** است.
2. **منطق کسب‌وکار در Code Node ممنوع است.** Code Node فقط برای نگاشت و قالب‌بندی است.
3. n8n روی **پرچم‌هایی که Karyar برمی‌گرداند** شرط می‌گذارد، مثل `requires_approval` یا `decision`. خودش محاسبه کسب‌وکاری نمی‌کند.
4. **نودهای AI Agent در n8n ممنوع‌اند.** همه ایجنت‌ها در Karyar هستند.
5. **Workflowها کوتاه و زنجیره‌ای‌اند.** Wait چندروزه ممنوع است. وضعیت پرونده در `Karyar Case` است.
6. n8n **پیش‌ثبت، مجوز و داده کسب‌وکاری نگه نمی‌دارد.** فقط `case_id` و حداقل فیلدها را منتقل می‌کند. ذخیره داده اجرای جریان‌های حساس خاموش است و داده‌های اجرا به‌صورت خودکار حذف می‌شوند (pruning).
7. **`case_id` و `correlation_id` در همه فراخوانی‌ها منتقل می‌شوند.** شناسه execution در n8n روی Case ثبت می‌شود.
8. **Karyar API برای پرونده‌های `paused` و `cancelled` هر اقدام و Promote را رد می‌کند.** پس ادامه اشتباهی یک Workflow اثری در ERPNext ندارد.

## ۷.۴ نگهداری و انتقال‌پذیری

- **export:** هر Workflow به JSON در `karyar-deploy/n8n/workflows/` منتقل می‌شود، همراه با یک `README` (Trigger، گام‌ها و endpointها). اعتبارنامه‌ها در git نیستند.
- **محیط‌ها:** n8n جدا برای test و production. نام‌گذاری: `KY | <process> | <segment>`.
- **انتقال‌پذیری:** چون منطق، وضعیت و مجوز در Karyar است، انتقال در آینده به موتور داخلی یعنی بازنویسی **ترتیب** از روی READMEها، نه بازسازی Karyar.

## ۷.۵ Frappe Workflow

- برای اسنادی که از مسیر Karyar ثبت می‌شوند، **Frappe Workflow فعال نمی‌شود.** تأیید پیش از وجود سند در Karyar انجام شده است و دو ماشین حالت روی یک سند تعارض ایجاد می‌کنند.
- Frappe Workflow برای اسنادی که کارکنان **مستقیم در Desk** می‌سازند، همچنان قابل استفاده است (ERPNext-first).

# ۸. Human-in-the-Loop: Workspace مشترک انسان و ایجنت

## ۸.۱ تعریف

Human Task یک **Workspace مشترک** است، نه یک دکمه تأیید یا رد:

- **تعامل اصلی گفت‌وگوست.** انسان با ایجنت صحبت می‌کند، وضعیت می‌پرسد، دستور می‌دهد، پیشنهاد تغییر می‌گیرد و تأیید می‌کند، کار را به ایجنت دیگر می‌سپارد، و پرونده را در محدوده اختیار خودش هدایت می‌کند.
- **پنل کاری** در کنار گفت‌وگو شامل این‌هاست:
  - داده ساخت‌یافته و Diff نسخه‌ها
  - پیش‌نمایش ERPNext
  - کارت‌های Change Proposal
  - وضعیت Approval
  - خط زمانی
  - کارهای واگذارشده

**دکمه‌ها فقط میانبرهای اختیاری‌اند.** محل ساخت Task را Workflow در n8n تعیین می‌کند، با `human_tasks.create(template, case_id)`. محتوا و قواعد Task در Karyar است.

## ۸.۲ اختیار انسان

**اختیار مؤثر = Identity ∩ Frappe Roles/Permissions ∩ Responsibility ∩ Human Task Template ∩ Approval Policy**

| لایه | تعیین می‌کند |
|---|---|
| Identity | چه کسی است. Session احرازشده و 2FA برای سطح `strong` |
| Frappe Roles و User Permission | به **چه داده‌ای** دسترسی دارد. محدوده‌هایی مثل شعبه هم همین‌جا تعیین می‌شود |
| Responsibility (انسانی) | چه **نقش کاری** در فرایند دارد. Role Assignment در هر Tenant |
| Human Task Template | در **این مرحله** چه اقدام‌ها و فیلدهایی مجاز است |
| Approval Policy | آیا واجد شرایط تأیید این عملیات است |

**درخواست خارج از اختیار:** ایجنت صریح می‌گوید خارج از حوزه اختیار کاربر است، افراد صاحب صلاحیت را معرفی می‌کند (از Role Assignment)، و در صورت مجاز بودن واگذاری را پیشنهاد می‌دهد. رویداد `security.out_of_scope_request` ثبت می‌شود.

## ۸.۳ اقدام‌ها

| اقدام | قاعده |
|---|---|
| مشاهده | فقط فیلدهای مجاز. بقیه پوشانده می‌شوند |
| پرسش وضعیت، تاریخچه، آخرین گفت‌وگو، آخرین تغییر و «چه کسی چه کرد» | ابزارهای فقط‌خواندنی `case.status`، `case.timeline`، `task.history`، `draft.diff` و `conversation.last_summary`، فیلترشده با مجوز همان کاربر |
| افزودن، ویرایش یا حذف (پیش‌ثبت یا اطلاعات مشتری) | **Change Proposal با Diff، سپس تأیید صریح، سپس ثبت** (۸.۴) |
| درخواست اطلاعات جدید از مشتری | واگذاری به ایجنت مسئول ارتباط با مشتری (مثلاً Ava) با `Agent Task`. پاسخ مشتری یک نسخه جدید پیش‌ثبت با منبع «مشتری» می‌سازد |
| کمک گرفتن از ایجنت دیگر | `Agent Task` طبق Delegation Policy (بخش ۱۰) |
| واگذاری به فرد دیگر (Assign) | فقط به فردی که Responsibility واجد شرایط دارد. از `assign_to` و `ToDo` استاندارد Frappe استفاده می‌شود |
| جلو بردن (`approved`/`forwarded`) | فقط اگر پیش‌شرط‌ها و Approval کامل باشد |
| برگرداندن به مرحله قبل (`returned`) | دلیل الزامی است |
| تغییر مسیر | فقط بین `allowed_routes` تعریف‌شده در Template |
| Pause و Resume | `Karyar Case.status`. اقدام‌ها و Promote در وضعیت Pause مسدودند |
| لغو (Cancel) | پایانی است، با سطح تأیید `confirm`. **اسناد قبلاً ثبت‌شده در ERPNext خودکار لغو نمی‌شوند.** لغو یا اصلاح آن‌ها اقدامی جدا و تحت Approval Policy است |

**اقدام‌های سطح فرایند** (جلو، عقب، مسیر، Pause/Resume، لغو) به‌صورت رویداد `human_task.decided` (با فیلدهای `decision`، `route` و `reason`) یا `case.*` به n8n می‌روند. **اقدام‌های داخل Task** (Assign، Change، واگذاری) در Karyar انجام می‌شوند.

## ۸.۴ پیشنهاد تغییر (Change Proposal)

<pre class="ltr">
Human: "Change the customer's mobile to 0935…"
 → agent interprets → ChangeProposal{target, field, before, after, interpretation}
 → shown as a diff: "Mobile: 0912… → 0935… — confirm?"
 → authenticated assignee confirms explicitly (in-session "yes" or button)
 → apply:  pre-registration → new draft revision
           finalized ERPNext data → update through a business action (+ Approval Policy floor)
 → audit: request(raw) · interpretation · proposal(before/after) · confirmation(who/when/how) · result · version
</pre>

- **تغییر حساس:** هر تغییر در داده پیش‌ثبت، داده کسب‌وکاری ERPNext، مسئول Task، مسیر یا وضعیت پرونده. یادداشت‌ها و نظرهای داخلی حساس نیستند.
- **ذخیره در پنل فرم**، پس از نمایش Diff، خودش یک تأیید صریح است.
- **چند تغییر** را می‌توان در یک Proposal دسته‌بندی کرد تا یک‌جا تأیید شوند. این خستگی ناشی از تأییدهای زیاد را کم می‌کند.
- **تأیید تغییر ≠ تأیید مجوز:**
  - **Change Confirmation:** درخواست‌کننده تأیید می‌کند که ایجنت درست فهمیده است.
  - **Approval:** فرد یا افراد واجد شرایط اجازه ثبت نهایی می‌دهند.
  - یک نفر ممکن است هر دو را انجام دهد، ولی دو رکورد جدا ثبت می‌شود.

## ۸.۵ مدل تأیید (Approval) — از فاز ۱

<pre class="ltr">
Approval Policy (per operation / workflow, versioned)
  stages: [                                   ordered stages = Sequential
    { eligible: users | roles | responsibilities | expression,
      required: M,                            1 = Single · M of N = M-of-N · all N = Parallel/Multi
      exclude_requester: true }               no self-approval
  ]
  invalidate_on_change: all | later_stages    important change after approval → re-approval
  reject_rule: any_reject_rejects | count_based
  confirmation_level: none | confirm | strong
</pre>

| ساختار | بیان |
|---|---|
| Single | یک مرحله با `required=1` |
| Multi-person / Parallel | یک مرحله با N نفر و `required=N`؛ همه همزمان |
| M-of-N | یک مرحله با `required=M` |
| Sequential | چند مرحله پشت سر هم |
| ترکیبی | مثلاً مرحله ۱: یک سرپرست؛ مرحله ۲: دو نفر از سه مدیر |

- **`Approval Request`:** مراحل، تأییدهای جمع‌شده، `approved_hash` و وضعیت را نگه می‌دارد. هر تأییدکننده یک Human Task از نوع `approval` در Workspace خودش دارد.
- **تأیید به hash گره می‌خورد:** hash پیش‌ثبت به‌اضافه hash پیش‌نمایش ERPNext. هر تغییر مهم پس از تأیید، طبق `invalidate_on_change` تأییدها را باطل می‌کند و تأیید مجدد لازم است.
- **کف ایمنی:** Promote فقط وقتی انجام می‌شود که Approval Request **با همان hash** کامل باشد، یا یک Auto-Execution Policy معتبر عملیات را مجاز کرده باشد. این کف حتی اگر Workflow در n8n گام تأیید را جا انداخته باشد اعمال می‌شود. در غیر این صورت `security.approval_missing` ثبت می‌شود.
- **آینده:** مهلت، escalation خودکار و جانشین تأییدکننده.

## ۸.۶ Human Task Template (پیکربندی برای هر Workflow)

| تنظیم | توضیح |
|---|---|
| `assignee_rule` | کاربر، Frappe Role یا Responsibility. در صف، Task توسط یک نفر برداشته (claim) و قفل می‌شود |
| `visible_fields` / `masked_fields` / `editable_fields` / `required_fields` | محدوده داده |
| `allowed_actions` / `allowed_routes` | اقدام‌ها و مسیرهای مجاز |
| `approval_policy` | ارجاع به Policy |
| `can_delegate_to` | Responsibilityها یا Integration Targetهای قابل واگذاری (در محدوده Delegation Policy) |

**تنظیمات در Karyar است.** n8n هنگام ساخت Task فقط می‌تواند آن‌ها را **محدودتر** کند، هرگز گسترده‌تر.

## ۸.۷ امنیت تصمیم

- **اختیار تصمیم** فقط از Session کاربر احرازشده و مسئول همان Task می‌آید.
- **متن پیام‌ها و دستورهای مشتری** هیچ اختیاری ایجاد نمی‌کنند. این دفاع اصلی در برابر Prompt Injection است.
- **تغییرهای مشتری** پس از بازبینی، Task را «تغییرکرده» علامت می‌زنند و تأییدهای مرتبط را باطل می‌کنند.

## ۸.۸ رابط کاربری

- **بخش اصلی:** گفت‌وگو.
- **پنل کناری:** داده، Diff، Proposalها، Approvalها، Timeline و کارهای واگذارشده.
- **فاز ۱:** فقط Workspace وب. انجام Task از پیام‌رسان برای کارکنان به آینده موکول شده است.
- **پیاده‌سازی:** یک SPA داخل Frappe، به الگوی `erpnext/banking` (React و `frappe-react-sdk`) یا frappe-ui. انتخاب فناوری باز است.

# ۹. معماری رویداد

| منبع | سازوکار (فاز ۱) |
|---|---|
| ERPNext | **DocType ‏Webhook استاندارد Frappe** (شرط‌گذاری، امضای HMAC و لاگ) به n8n. مثال: `erp.lead.created` و `erp.sales_order.submitted` |
| Karyar | `emit_event()` پس از commit. نمونه‌ها: `intake.completed`، `agent.task.completed`، `human_task.decided`، `approval.completed`، `case.paused`، `case.resumed`، `case.cancelled` |
| بیرونی | callback از n8n به Karyar API (امضاشده) |
| زمان | Cron در n8n. Scheduler در Frappe برای نگهداری داده و پاک‌سازی |

**پوشش رویداد:** `event_id`، `type`، `tenant`، `occurred_at`، `subject` (doctype/name یا case)، `actor`، `correlation_id`، و یک `payload` حداقلی. مصرف‌کننده جزئیات را **با مجوز خودش** از API می‌خواند.

**قابلیت اطمینان:** Webhook در Frappe و `emit_event` شکست‌ها را لاگ می‌کنند. یک Workflow زمان‌بندی‌شده در n8n «پرونده‌های بی‌حرکت» را از Karyar می‌پرسد. Outbox اختصاصی به آینده موکول شده است.

# ۱۰. واگذاری و ارتباط ایجنت با ایجنت

**سه شکل ارتباط:**

1. **حرکت بین مراحل فرایند:** با رویداد و n8n.
2. **واگذاری کنترل‌شده (Delegation):** **داخل Karyar.** ایجنت یا انسان یک `Agent Task` برای ایجنت دیگر یا یک Integration Target می‌سازد.
3. **ارجاع از طریق ERPNext:** سندی ثبت می‌شود و رویداد آن فرایند دیگری را آغاز می‌کند.

**چت آزاد ایجنت با ایجنت وجود ندارد.**

- **Delegation Policy** (در `Karyar Settings`) مشخص می‌کند:
  - کدام Responsibility به کدام Responsibility یا Integration Target کار بسپارد
  - با کدام توانمندی‌ها
  - آیا تأیید انسانی لازم است
  - دامنه کار (مثلاً فقط مخاطبانِ همین پرونده)
  - محدودیت تعداد
- **`Agent Task`** این‌ها را نگه می‌دارد:
  - والد (Case یا Task)
  - درخواست‌دهنده: ایجنت، **از طرف کدام انسان**، و بر اساس کدام Policy
  - مقصد
  - دستور کار ساخت‌یافته (ارجاع، نه کپی)
  - وضعیت: `queued`، `running`، `waiting_customer`، `completed`، `failed` یا `cancelled`
  - نتیجه و زنجیره واگذاری
- **عدم افزایش اختیار:** اختیار Agent Task برابر است با **اشتراکِ** اختیار Principal اولیه، توانمندی ایجنت واگذارکننده، توانمندی مقصد و Delegation Policy.
- **محدودیت‌ها:** عمق زنجیره حداکثر ۲ (پیش‌فرض). چرخه ممنوع است.
- **بازگشت نتیجه:** رویداد `agent.task.completed` منتشر می‌شود. نتیجه در Workspace نمایش داده می‌شود. اگر داده را تغییر دهد، به‌صورت **Change Proposal** به انسان ارائه می‌شود.
- **Integration Target:** یک فهرست حداقلی شامل نام، Workflow مربوط در n8n، Schema ورودی و خروجی، توانمندی لازم، سیاست تأیید و محدودیت‌های کانال. Karyar آغاز، مجوز و ثبت را انجام می‌دهد و n8n اجرا می‌کند و نتیجه را با callback برمی‌گرداند.
- **Task Brief:** ایجنت مقصد یک دستور کار ساخت‌یافته دریافت می‌کند، نه متن خام گفت‌وگو. این از انتقال Prompt Injection جلوگیری می‌کند و داده کمتری منتقل می‌شود.

# ۱۱. معماری مجوز

## ۱۱.۱ زنجیره مجوز (همیشه در کد)

<pre class="ltr">
Tenant (site)          → enabled packs, capabilities, integrations
 └ Principal / User    → authenticated human | agent service user | channel contact | integration (n8n)
    └ Agent            → which Responsibilities it holds
       └ Responsibility → which Capabilities
          └ Capability  → which tools / reports / integration targets
             └ Workflow step / Task template → allowed actions, fields, records in THIS step
                └ ERPNext permission → DocType perms, User Permissions (e.g. branch), permlevel, has_permission
Effective = intersection of all levels.  Delegation: also ∩ the initiating principal.
A prompt can never add a permission or an approval.
</pre>

## ۱۱.۲ از طرفِ چه کسی؟ (On-behalf-of)

| موقعیت | هویت اجرایی | نتیجه |
|---|---|---|
| کارمند در Workspace با ایجنت | **کاربر همان کارمند** (`frappe.set_user`)، به‌علاوه فیلتر توانمندی و گام | ایجنت هرگز بیش از کارمند نمی‌بیند |
| کار فرایندی خودکار | کاربر سرویس ایجنت، محدود به رکوردهای همان پرونده | دسترسی حداقلی |
| کار واگذارشده | اشتراک اختیار Principal اولیه و همه حلقه‌های زنجیره | بدون افزایش اختیار |
| مشتری در کانال | Contact Principal با ابزارهایی که فقط داده همان مخاطب را برمی‌گردانند | مشتری فقط داده خودش |
| n8n | کاربر یکپارچه‌سازی **هر Tenant** با کلید API مجزا و توانمندی‌های محدود | فقط Karyar API |

## ۱۱.۳ قواعد پیاده‌سازی

- ابزارها فقط از APIهای مجوزدار استفاده می‌کنند: `get_list`، `check_permission`، `has_permission`، `query_report.run`.
- **`frappe.get_all`، `ignore_permissions=True` و `frappe.db.sql` در ابزارها ممنوع است.** این با قانون semgrep در CI بررسی می‌شود.
- **Schema خروجی** هر ابزار تعیین می‌کند چه فیلدهایی برگردد. فیلدهای بالاتر از permlevel کاربر پوشانده می‌شوند.
- **مثال:** کارمند فروش «سود کل» را می‌پرسد. نقش او به گزارش دسترسی ندارد، پس نتیجه `PERMISSION_DENIED` است و **هیچ عددی به مدل نمی‌رسد**.
- **منع تأیید توسط خود:** درخواست‌کننده نمی‌تواند درخواست خودش را تأیید کند (`exclude_requester`).

# ۱۲. یکپارچگی با ERPNext (ERPNext-first)

## ۱۲.۱ چه چیزی به ERPNext سپرده می‌شود

| نیاز | قابلیت استاندارد ERPNext/Frappe | کار Karyar |
|---|---|---|
| ثبت و اعتبارسنجی اسناد | کنترلرها، `insert` و `submit`، توابع `make_*` | فقط فراخوانی از طریق Promote |
| محاسبه مالیات، جمع و دفتر کل | `calculate_taxes_and_totals`، پیش‌نمایش دفتر کل (`get_accounting_ledger_preview`) | پیش‌نمایش **بدون ذخیره** و hash |
| اعتبار مشتری، قیمت، موجودی | Credit Limit، Price List، Projected Qty | خواندن از طریق ابزار؛ بدون قاعده موازی |
| سفارش مجدد خودکار | Reorder Level و Material Request خودکار | فقط واکنش به رویداد |
| تاریخچه تغییرات اسناد | `Version` (track changes) | ممیزی **معنایی** مکمل؛ تغییرات میدانی تکرار نمی‌شود |
| تخصیص و اعلان | `ToDo`، `assign_to`، Assignment Rule، Notification | Human Task به‌جای ساختن سیستم اعلان جدا از این‌ها استفاده می‌کند |
| ایمیل | Email Account، Email Queue، ارسال سند با Print Format | ایجنت درخواست ارسال می‌کند و ERPNext می‌فرستد |
| پیامک تراکنشی ساده | SMS Settings | برای پیامک ساده. پیامک انبوه و APIهای خاص با n8n |
| تاریخچه تعامل با مشتری | `Communication` (medium‌های Chat، SMS و Phone) و Timeline | خلاصه گفت‌وگو پس از اتصال به مخاطب در Timeline ثبت می‌شود |
| اطلاعات تماس | Contact (`phone_nos`، `email_ids`، `mobile_no`)، Lead (`whatsapp_no`) | فقط کانال‌های بدون فیلد استاندارد در `Contact Channel` |
| کمپین | `Campaign` (ماژول CRM) | رکورد کسب‌وکاری کمپین در ERPNext؛ اجرا با Karyar و n8n |
| گزارش | Query/Script Reports، Prepared Report | Report Catalog و فقط گزارش‌های مکمل |
| بومی‌سازی | Tax Template، Tax Withholding Category، `regional_overrides`، `create_charts` | `karyar_iran` روی همین‌ها |
| تأیید اسنادی که در Desk ساخته می‌شوند | Frappe Workflow | دخالت نمی‌کند |
| نمایش پرونده‌های در جریان در ERPNext | Connections dashboard (`override_doctype_dashboards`) | `Karyar Case` در Connections مشتری، Lead و غیره |

## ۱۲.۲ پیش‌ثبت (Pre-Registration) و Promote

- **پیش‌فرض:** داده در حال تکمیل و پیش‌نویس‌ها **فقط در Karyar** (`Karyar Case.draft_data`) هستند. **قبل از تأیید هیچ سند کسب‌وکاری در ERPNext ساخته نمی‌شود.**
  - چون Karyar و ERPNext روی یک Site هستند، پرونده در جریان در Desk و در Connections اسناد مرتبط **قابل مشاهده** است.
  - اطلاعات پایه (Lead، Customer، Contact) قبل از تأیید ساخته نمی‌شوند. **استثنا:** گزینه «Lead ناقص» که برای هر Workflow قابل تنظیم و پیش‌فرض آن خاموش است.
- **فقط دلتا:** برای موجودیت‌های موجود، پیش‌ثبت فقط ارجاع به‌علاوه داده جدید یا تغییرکرده را نگه می‌دارد.
- **پیش‌نمایش:** محاسبات متعلق به ERPNext روی سندِ **در حافظه** و بدون ذخیره اجرا می‌شود. hash آن جزو چیزی است که تأیید می‌شود.
  - تابع عمومی پیش‌نمایش دفتر کل در v16 به سند ذخیره‌شده نیاز دارد. پس از تابع داخلی روی سندِ در حافظه استفاده می‌شود.
  - **راه جایگزین:** savepoint و سپس rollback.
  - **کار فنی فاز ۱:** آزمایش کوتاه (spike) برای Sales Invoice و Journal Entry.
- **Promote** (تنها مسیر ثبت نهایی) به ترتیب این کارها را انجام می‌دهد:
  1. بررسی کامل بودن طبق Schema
  2. بررسی مجوز و وضعیت پرونده (نباید paused یا cancelled باشد)
  3. بررسی Approval Request با **همان hash**، یا Auto-Execution Policy معتبر
  4. محاسبه دوباره پیش‌نمایش. اگر متفاوت بود، توقف و درخواست تأیید مجدد
  5. ثبت با کنترلرهای ERPNext و `idempotency_key`
  6. ذخیره ارجاع در Case، و قفل کردن پیش‌ثبت به‌عنوان «منتقل‌شده»
- **پیش‌نویس اختیاری در ERPNext** (`erp_draft_mode`، **پیش‌فرض خاموش**):
  - فقط برای اسناد ثبت‌شدنی و فقط وقتی یک Workflow واقعاً به آن نیاز دارد.
  - Karyar یک سند با `docstatus=0` می‌سازد که `karyar_case` و `karyar_locked=1` دارد.
  - ویرایش آن در Desk با `validate` و `has_permission` مسدود است. فقط Karyar پس از تغییرِ تأییدشده آن را به‌روز می‌کند.
  - در Promote همان سند Submit می‌شود.
  - منبع ویرایش همچنان یکی است (Karyar)، پس Karyar ERP دوم نمی‌شود.
- **پس از ثبت نهایی:** داده اصلی فقط از ERPNext خوانده می‌شود.
  - اصلاح بعدی از طریق Change Proposal و یک اقدام کسب‌وکاری روی ERPNext انجام می‌شود.
  - لغو یا اصلاح سند (Amend) با سازوکار استاندارد ERPNext و تحت Approval Policy است.
  - snapshot تأییدشده فقط برای ممیزی نگه داشته می‌شود.

## ۱۲.۳ شناسه‌های کانال مشتری

**جدول فرزند سفارشی `Contact Channel` روی Contact** (به‌صورت fixture و بدون تغییر هسته). فیلدها:

- `channel`: telegram، instagram، whatsapp، bale، eitaa، web، sms، phone، email
- `identifier`: برای کانال‌های بدون فیلد استاندارد
- `display_handle`
- `verified`
- `consent_marketing`
- `opted_out_at`
- `last_inbound_at`: برای پنجره زمانی پاسخ در برخی کانال‌ها
- `is_primary`
- `source_case`

**قواعد:**

- مقدار تلفن و ایمیل فقط در فیلدهای استاندارد Contact می‌ماند. ردیف `Contact Channel` برای این کانال‌ها فقط رضایت، انصراف و آخرین تماس را نگه می‌دارد.
- Lead و Customer از طریق `links` در Contact به این جدول دسترسی دارند. `whatsapp_no` در Lead هم استاندارد است و حفظ می‌شود.
- **پیش از ثبت نهایی**، شناسه کانال در پیش‌ثبت و Conversation است. در Promote به Contact منتقل می‌شود.
- **`Contact Identity` در Karyar حذف شد.** Channel Gateway برای مسیریابی پیام‌های ورودی فقط یک index قابل بازسازی از روی `Contact Channel` دارد.

## ۱۲.۴ قرارداد ابزار

<pre class="ltr">
@karyar_tool(name="crm.lead.propose", capability="crm.lead.draft",
             side_effect="draft", risk="low",
             input_schema=LeadDraftIn, output_schema=CaseRef,
             required_perms=[("Lead", "create")], idempotent=True)
</pre>

انواع اثر ابزار (`side_effect`):

- `read`
- `draft` (فقط پیش‌ثبت در Karyar)
- `promote` (ثبت نهایی؛ همیشه از دروازه Promote)
- `external` (Integration)

# ۱۳. Karyar API

- **منطق یکتا، چند نمایش:** سرویس‌های Karyar از سه راه در دسترس‌اند:
  1. ابزارهای ایجنت
  2. REST نسخه ۱ (`/api/method/karyar.api.v1.*`)
  3. MCP در آینده
- **احراز هویت:**
  - کلید API برای n8n، **جدا برای هر Tenant**
  - Session برای Workspace
  - OAuth2 برای اپ‌های بیرونی
- **idempotency:** همه endpointهای تغییردهنده `idempotency_key` می‌پذیرند.
- **امضا:** همه callbackها امضای HMAC دارند.

**endpointهای فاز ۱:**

| گروه | endpointها |
|---|---|
| Case | `cases.create`، `get`، `set_stage`، `pause`، `resume`، `cancel`، `timeline` |
| Agent | `agents.run_task` (ناهمگام)، `agent_tasks.create`، `get` |
| Human Task | `human_tasks.create(template)`، `get`، `claim`، `assign`، `decide` |
| Change | `changes.propose`، `confirm`، `reject` |
| Approval | `approvals.create`، `status`، `grant`، `deny` |
| Promote | `promote(case, operation)` (تنها مسیر ثبت نهایی) |
| Integration | `integrations.callback(agent_task)`، `integrations.needs_input` (برای OTP و CAPTCHA در آینده) |

# ۱۴. معماری گزارش‌گیری

| اولویت | منبع | سازوکار |
|---|---|---|
| ۱ | **گزارش‌های استاندارد ERPNext** | **Report Catalog** شامل شناسه، Schema فیلترهای مجاز و ستون‌های مجاز. اجرا با `query_report.run` و هویت Principal |
| ۲ | **گزارش‌ها و Queryهای کنترل‌شده Karyar** | تابع Python از پیش نوشته‌شده با Schema، **فقط وقتی** ERPNext گزارش معادل ندارد. مثال: «مشتریان دارای خرید در بازه X همراه با رضایت کانال» |
| ۳ (آینده) | Frappe Insights | داشبورد برای انسان |
| ❌ | SQL ساخته‌شده توسط AI | ممنوع |

- **اعداد همیشه از نتیجه گزارش می‌آیند**، و پاسخ منبع خود را نشان می‌دهد: نام گزارش، فیلترها و زمان.
- **پرونده‌های در جریان** فقط در داشبورد عملیاتی Karyar دیده می‌شوند. در گزارش‌های کسب‌وکاری ERPNext نمی‌آیند.
- **گزارش‌های سنگین** با Prepared Report و در صورت نیاز Read Replica اجرا می‌شوند.

# ۱۵. معماری مدل هوش مصنوعی

<pre class="ltr">
Agent Runtime → AIGateway.complete(request) → ProviderAdapter → Provider/Model
     alias → (provider, model, params, fallback)  per tenant
     JSON-schema enforcement · redaction by data classification · timeouts/retries
     → UsageSink.record(UsageRecord)
</pre>

- **آداپتورها:** «سازگار با OpenAI» (که ارائه‌دهندگان داخلی و مدل‌های خودمیزبان را هم پوشش می‌دهد)، به‌علاوه آداپتورهای اختصاصی از طریق `karyar_ai_providers`.
- **Model Profile** در هر Tenant: نام مستعار را به مدل واقعی نگاشت می‌کند.
- **Prompt Template** نسخه‌دار است.
- **کلید API** در فیلد Password سایت Tenant ذخیره می‌شود.
- **LiteLLM Proxy** گزینه آینده است.
- ⚠️ **ایران:** دسترسی به برخی ارائه‌دهندگان بین‌المللی محدود است و ارسال داده حساس به خارج پیامد حقوقی دارد. پشتیبانی از مدل خودمیزبان یا داخلی از روز اول لازم است، همراه با ارزیابی کیفیت فارسی هر مدل.

# ۱۶. مصرف و هزینه AI

**`UsageRecord` در هر فراخوانی:**

<pre class="ltr">
request_id, tenant, agent, responsibility, user (on-behalf-of), case, task (human/agent task),
workflow (n8n process + segment), conversation, provider, model, model_alias,
input_tokens, output_tokens, cached_tokens, total_tokens, latency_ms, status,
price_version, estimated_cost, currency, actual_cost (nullable), timestamp
</pre>

- **فاز ۱:**
  - ذخیره در `AI Usage Log` با گزارش بر اساس Tenant، Agent، Workflow، User و Task
  - **سقف سخت** مصرف روزانه برای هر Tenant و هر Agent
  - جدول نسخه‌دار `Model Price`
- **آینده:**
  - `UsageSink` مرکزی در Control Plane برای تجمیع بین Tenantها
  - تطبیق `actual_cost` با صورت‌حساب ارائه‌دهنده
  - صورت‌حساب‌دهی به مشتری

# ۱۷. n8n

- **نقش:** فقط هماهنگی گام‌ها و اجرای Integration (بخش ۷).
- **جداسازی Tenant:** **یک نمونه n8n برای هر Tenant** (کانتینر جدا با پایگاه داده Postgres مخصوص). دلیل این تصمیم:
  - نسخه Community در n8n جداسازی چندمستأجری ندارد.
  - Credentialهای Tenantها نباید کنار هم باشند.
  - هزینه این مدل در §۲۸ (R5) ثبت شده است.
- **Credentialها:**
  - Credential سرویس‌های بیرونی (پیامک و…) در n8n همان Tenant ذخیره می‌شوند.
  - Karyar فقط **نام** Integration Target را می‌شناسد.
  - کلید Karyar API و secret مربوط به HMAC جدا برای هر Tenant است.
- **شبکه:** n8n در شبکه خصوصی است و فقط مسیرهای Webhook از طریق Reverse Proxy در دسترس‌اند.
- **داده:** pruning داده‌های اجرا فعال است و ذخیره داده اجرای جریان‌های حساس خاموش است.
- **ساختار:** زیرWorkflow مشترک «Call Karyar» برای احراز هویت، ارسال `case_id` و مدیریت خطا، و یک Error Workflow برای هشدار.
- **مجوز n8n:** مجوز Sustainable Use پیش از فروش تجاری باید بررسی حقوقی شود. جایگزین‌ها: Activepieces یا Node-RED.

# ۱۸. معماری RPA (آینده)

- **فقط** برای سیستم‌های بدون API: پورتال‌های دولتی، بانکی، و نرم‌افزارهای قدیمی.
- **هرگز برای ERPNext.** برای ERPNext همیشه ابزار و API استفاده می‌شود.
- **اجرا:** یک RPA Worker جدا (Python و Playwright، در کانتینر ایزوله، بدون دسترسی به پایگاه داده). آغاز آن با Karyar به‌صورت یک `Agent Task` به یک Integration Target است. هماهنگی اجرا با n8n است.
- **CAPTCHA و OTP:**
  1. Worker اعلام `needs_input` می‌کند.
  2. Karyar یک **Human Task** گفت‌وگومحور می‌سازد.
  3. Karyar پاسخ کاربر را به Worker برمی‌گرداند.
- **اعتبارنامه پورتال‌ها** در فیلد Password سایت Tenant است و فقط برای همان کار و برای مدت کوتاه به Worker داده می‌شود.
- **شواهد** (اسکرین‌شات و لاگ) به‌صورت فایل خصوصی به Agent Task پیوست می‌شوند.

# ۱۹. معماری چندمستأجری

| مدل | جداسازی | مناسب برای |
|---|---|---|
| A. یک Site و چند Company | ضعیف | فقط یک گروه تجاری. **❌ برای کسب‌وکارهای مستقل** |
| **B. یک Site برای هر Tenant روی Bench مشترک** ✅ | پایگاه داده، کاربر پایگاه داده، پوشه فایل و کلید رمزنگاری جدا. پروسه‌ها و Redis مشترک ولی با فضای نام سایت | **پیش‌فرض** |
| C. یک Bench یا Stack برای هر Tenant | کامل | مشتریان بزرگ یا حساس (مثلاً پزشکی) |

- **Tenant = مرز کسب‌وکار یا پروژه.** محدوده‌های داخلی (شعبه، واحد، شرکت زیرمجموعه) با User Permission کنترل می‌شوند.
- **همه اجزای بیرونی Tenant-aware هستند:** n8n جدا برای هر Tenant، کلیدهای API و HMAC جدا، Integration Target و Credential در محدوده Tenant، و AI Gateway با Model Profile و کلید جدا.
- **قواعد Bench مشترک:**
  - به مشتری Server Script، Administrator، bench یا console داده نمی‌شود.
  - کد اختصاصی یک مشتری بدون بازبینی روی Bench مشترک نصب نمی‌شود.
- **Control Plane** (ثبت Tenant، راه‌اندازی، تجمیع مصرف): در فاز ۱ با اسکریپت `karyar-deploy`. سیستم کامل در آینده.

# ۲۰. معماری امنیت

| حوزه | تصمیم |
|---|---|
| جداسازی Tenant | مدل B یا C. هر سایت کلید رمزنگاری خودش را دارد. n8n و Credentialها جدا برای هر Tenant |
| احراز هویت انسان | ورود Frappe. 2FA برای نقش‌های مالی و مدیریتی و برای تأیید با سطح `strong` |
| احراز هویت مشتری | هویت کانال از `Contact Channel`. برای داده حساس، OTP |
| احراز هویت ماشین | کلید API یا OAuth2 جدا برای هر Tenant و هر سیستم، با چرخش کلید |
| مجوز | زنجیره بخش ۱۱. ممنوعیت `ignore_permissions` و SQL در ابزارها. **Prompt مجوز یا تأیید نمی‌سازد** |
| تصمیم و تأیید | فقط از Session کاربر احرازشده و مسئول. گره خوردن به hash. منع تأیید توسط خود. ثبت تلاش‌های غیرمجاز |
| واگذاری | اشتراک اختیار (جلوگیری از Confused Deputy)، عمق محدود، منع چرخه |
| Prompt Injection | محتوای مشتری، فایل‌ها و وب «داده نامطمئن» برچسب می‌خورند. Task Brief به‌جای متن خام. ابزارهای پرریسک در گفت‌وگوی مشتری در دسترس نیستند |
| اسرار | فیلد Password در Frappe (رمزنگاری با کلید سایت). کلیدهای سکو در متغیر محیطی یا Vault. **هرگز** در Prompt، لاگ، Pack یا git |
| ورودی‌های بیرونی | HMAC، جلوگیری از replay با timestamp و nonce، و محدودیت نرخ |
| پیام‌رسانی | رعایت رضایت و انصراف (`Contact Channel`)، قواعد هر کانال، و قواعد پیامک تبلیغاتی |
| داده به AI | Schema خروجی ابزار، طبقه‌بندی داده، و گزینه مدل خودمیزبان برای داده حساس |
| RPA | ایزوله، بدون دسترسی به پایگاه داده، اعتبارنامه کوتاه‌مدت |
| زنجیره تأمین | پین نسخه‌ها، mirror داخلی PyPI و npm، اسکن وابستگی‌ها |

# ۲۱. ممیزی، نسخه‌داری و مشاهده‌پذیری

| لایه | محتوا | ابزار |
|---|---|---|
| تغییرات میدانی اسناد ERPNext | چه فیلدی در کدام سند تغییر کرد | **`Version` استاندارد Frappe** |
| ممیزی معنایی Karyar | چه کسی (انسان، ایجنت، مشتری، سیستم یا Policy)، از طرف چه کسی، چه درخواستی، برداشت ایجنت، پیشنهاد (قبل و بعد)، تأیید (چه کسی، کِی، چطور)، اجرای واقعی، سند اثرپذیرفته، Case، Task، Workflow، نسخه داده، خطا | `Karyar Audit Log` (فقط‌افزودنی) |
| نسخه‌های پیش‌ثبت | شماره نسخه، Diff، hash قبل و بعد، منبع | `draft.revised` در Audit Log |
| فنی | زمان‌ها، retryها، مصرف AI، خطاها | AI Usage Log، Error Log، RQ Job، شناسه execution در n8n |

**انواع رویداد ممیزی:**

- `change.requested`، `interpreted`، `proposed`، `confirmed`، `rejected`، `applied`
- `approval.requested`، `granted`، `denied`، `completed`، `invalidated`
- `delegation.created`، `completed`، `denied_by_policy`
- `task.created`، `claimed`، `assigned`، `decided`
- `case.paused`، `resumed`، `cancelled`
- `promote.succeeded`، `promote.blocked`
- `auto_execution.by_policy`، همراه با نسخه Policy و تعریف‌کننده
- `security.out_of_scope_request`، `security.approval_missing`، `security.unauthorized_decision_attempt`

**یکپارچگی و نگهداری:**

- هیچ نقشی، حتی System Manager، نمی‌تواند Audit Log را ویرایش یا حذف کند.
- برای عملیات حساس، **زنجیره hash** در آینده اضافه می‌شود.
- ذخیره Prompt و پاسخ مدل با سطح قابل تنظیم است: `off`، `summary` یا `full`.

**مشاهده‌پذیری:**

- **فاز ۱:** `correlation_id` در همه لاگ‌ها، لاگ ساخت‌یافته JSON، و داشبورد Workspace شامل پرونده‌های بی‌حرکت، کارهای معوق، Approvalهای ناقص، خطاها و مصرف AI.
- **آینده:** OpenTelemetry، Prometheus و Grafana، و Sentry.

# ۲۲. جریان و مالکیت داده

| دسته | چه چیزی | کجا | عمر |
|---|---|---|---|
| **پیش‌ثبت** (Draft / Pre-Registration) | داده ناقص، پیش‌نویس، تغییرات پیشنهادی (فقط دلتا) | `Karyar Case.draft_data` و نسخه‌های آن | تا Promote یا رها شدن |
| **Context تعامل** | پیام‌ها و خلاصه‌ها | `Conversation` و `Message` در Karyar. **خلاصه نهایی در Timeline ‏ERPNext** (`Communication`) | طبق سیاست نگهداری |
| **فرایند و ممیزی** | مرحله پرونده، ارجاع به اسناد ERPNext، Proposalها، Approvalها، Agent Taskها، ممیزی و مصرف AI | Karyar | دائمی |
| **داده نهایی کسب‌وکار** | Lead، Customer، Contact (و `Contact Channel`)، سفارش، فاکتور، اسناد حسابداری و انبار، Campaign | **ERPNext** | دائمی (Final Truth) |

<pre class="ltr">
conversation → Karyar draft (revisions) → [customer confirmation?] → [Approval Request?] → Promote → ERPNext (final)
      ↑ change proposals (confirmed)                    (configurable per workflow)              │
      └────────────────────────────────────────────────────────────────────────────────────────── │
                            Karyar Case keeps only: refs (doctype/name) + approved snapshot (audit-only)
</pre>

**قواعد:**

1. پیش‌ثبت فقط **دلتا** و ارجاع نگه می‌دارد. کپی از داده پایه ERPNext ممنوع است.
2. **پس از Promote**، ایجنت داده را فقط از ERPNext می‌خواند. پیش‌ثبتِ قفل‌شده و snapshot در دسترس ابزارهای ایجنت نیستند.
3. **خلاصه گفت‌وگو** قصد، ترجیحات، سؤال‌های باز و قول‌های داده‌شده را نگه می‌دارد. واقعیت‌های کسب‌وکاری در خلاصه با برچسب «طبق گفته مشتری در تاریخ X» ثبت می‌شوند و منبع معتبر نیستند.
4. **پیش‌ثبت رهاشده:** پس از مدت نگهداری Workflow (پیش‌فرض ۳۰ روز، قابل تنظیم)، پرونده با وضعیت `abandoned` بسته و داده شخصی پاک می‌شود. اگر گزینه «Lead ناقص» برای آن Workflow روشن باشد، قبل از پاک‌سازی Lead ناقص ساخته می‌شود (این گزینه پیش‌فرض خاموش است و تحت Policy قرار دارد).
5. **n8n هیچ‌وقت پیش‌ثبت نگه نمی‌دارد.**

# ۲۳. ساختار مخازن و اپ

| مخزن | محتوا | فاز |
|---|---|---|
| `karyar-mainERP` (همین) | fork ERPNext **با پایه `version-16`** و فقط برای وصله‌های اجتناب‌ناپذیر (`PATCHES.md`). مستندات موقتاً در `docs/karyar/` | ۰ |
| `karyar` | اپ اصلی | ۱ |
| `karyar_iran` | بومی‌سازی ایران | ۱ (مبانی) و ۲ |
| `karyar_hr_ir` | حقوق و بیمه ایران روی HRMS | آینده |
| `karyar-deploy` | `apps.json` پین‌شده، Docker و compose، اسکریپت راه‌اندازی Tenant (سایت، اپ‌ها، Pack، n8n)، Workflowهای n8n، Packها، runbookها | ۰ تا ۱ |
| `karyar-rpa-worker` | RPA | آینده |

<pre class="ltr">
karyar/karyar/
├── hooks.py              doc_events, scheduler, fixtures, dashboards, extension-point hooks
├── core/                 framework-light logic: policy, tools, agents, ai, promote, approvals, delegation
├── karyar_agents/        Karyar Agent, Responsibility, Capability, Role Assignment, Prompt Template, Model Profile
├── karyar_cases/         Karyar Case, Human Task, Human Task Template, Approval Request, Agent Task
├── karyar_conversations/ Conversation, Message, Channel Account
├── karyar_audit/         Karyar Audit Log, AI Usage Log, Model Price
├── karyar_settings/      Karyar Settings (+ policy tables: approval, auto-execution, delegation,
│                         retention, integration targets)
├── fixtures/             Custom Field: Contact Channel (child of Contact), idempotency/karyar_case fields
├── tools/ channels/ api/v1/ packs/ workspace/ (SPA) commands/ locale/fa.po tests/
</pre>

- **حذف‌شده نسبت به نسخه ۰٫۱:** `Contact Identity`، و ماژول‌های Workflow Engine و Event (`Workflow Definition`، `Workflow Run`، `Step Run`، `Timer`، `Karyar Event`).
- **DocTypeهای Karyar:** Agent، Responsibility، Capability، Role Assignment، Prompt Template، Model Profile، Case، Human Task، Human Task Template، Approval Request، Agent Task، Conversation، Message، Channel Account، Audit Log، AI Usage Log، Model Price، Settings.

# ۲۴. معماری استقرار

**فاز ۱** (برای هر محیط؛ مبتنی بر `frappe_docker`):

<pre class="ltr">
nginx (TLS, per-tenant hostnames)
 ├─ frappe-web (gunicorn) · socketio
 ├─ workers: short, default, long, karyar_agent (concurrency-limited)
 ├─ scheduler
 ├─ MariaDB (one DB per site) · redis-cache · redis-queue
 ├─ n8n per tenant (+ its Postgres), private network, webhooks via reverse proxy
 └─ encrypted backups → S3-compatible storage
</pre>

- **آینده:** چند Bench، Kubernetes، replica برای MariaDB، RPA Worker، و در صورت نیاز سرویس جدای Channel Gateway و LiteLLM.
- **محل میزبانی** (داخل ایران، خارج، یا ترکیبی) تصمیم باز است (§۲۹).

# ۲۵. راهبرد مقیاس‌پذیری

- **گلوگاه اصلی** تأخیر و نرخ API مدل است. راه‌ها:
  - صف `karyar_agent` جدا
  - سقف همزمانی برای هر Tenant
  - پخش زنده پاسخ
  - کش کوتاه‌مدت برای ابزارهای خواندنی
- **افقی:** Worker و gunicorn بیشتر. Workerهای ایجنت مستقل مقیاس می‌گیرند.
- **پایگاه داده:** Prepared Report، replica برای گزارش، و انتقال Tenant پرمصرف به Bench جدید.
- **n8n جدا برای هر Tenant** به‌صورت طبیعی افقی است. هزینه آن در §۲۸.
- **سرویس جدا** فقط وقتی ساخته می‌شود که اندازه‌گیری نشان دهد لازم است.

# ۲۶. راهبرد ارتقا

1. ERPNext و Frappe دست‌نخورده، با نسخه پین‌شده. ارتقای minor ماهانه: اول staging، بعد production.
2. **آزمون قرارداد ابزارها و Promote** روی نسخه هدف ERPNext. شکست در این آزمون یعنی توقف ارتقا.
3. **Karyar API v1** مصرف‌کننده‌ها (n8n و Workspace) را از تغییرات ERPNext محافظت می‌کند.
4. **نسخه‌داری:** Policyها، Templateها، Packها، دستورالعمل‌ها، و Workflowهای n8n (export در git).
5. **ارتقای v16 به v17:** حدود ۶ ماه پس از انتشار پایدار v17. فرصت ارزیابی PostgreSQL.
6. **بدون Monkey-patch.**

# ۲۷. بومی‌سازی ایران (`karyar_iran`)

| بخش | طراحی (روی قابلیت‌های استاندارد ERPNext) | فاز |
|---|---|---|
| زبان و RTL | ترجمه موجود Frappe (۹۶٪) و ERPNext (۸۱٪). تکمیل `fa.po` | ۱ |
| نرمال‌سازی | ارقام فارسی و عربی، «ی/ک»، نیم‌فاصله، شماره موبایل. **ابزار قطعی** | ۱ |
| اعتبارسنج‌ها | کد ملی، شناسه ملی، کد اقتصادی، شبا، کد پستی | ۱ |
| تاریخ جلالی | ذخیره میلادی و نمایش جلالی. `parse_persian_date` قطعی. `naming_series_variables` | ۱ تا ۲ |
| کانال‌های ایرانی | Adapterهای بله و ایتا، و ردیف آن‌ها در `Contact Channel` | ۱ تا ۲ |
| پیامک | SMS Settings برای پیامک تراکنشی. پیامک انبوه با Integration Target در n8n. رعایت قواعد خطوط تبلیغاتی و انصراف | ۱ تا ۲ |
| حسابداری | چارت حساب با `create_charts(custom_chart=…)`. سطح تفصیلی با Party و Accounting Dimension. دقت ریال برابر صفر | ۲ |
| مالیات | Tax Template و Tax Withholding Category، با نرخ قابل‌پیکربندی | ۲ |
| مودیان | ماژول جدا: صف ارسال، استعلام، HITL برای خطا | آینده |
| اسناد فارسی | Print Formatهای RTL | ۲ |
| حقوق و بیمه | `karyar_hr_ir` روی HRMS | آینده |

# ۲۸. ریسک‌ها و توازن‌ها

| # | ریسک | شدت | کاهش |
|---|---|---|---|
| R1 | دسترسی و قانونی بودن ارائه‌دهندگان AI از ایران | بالا | AI Gateway، مدل خودمیزبان، طبقه‌بندی داده، بررسی حقوقی |
| R2 | محدودیت کانال‌ها: فیلتر شدن؛ API ‏Meta برای کسب‌وکار ایرانی؛ قالب پیام و رضایت در WhatsApp؛ پنجره پاسخ Instagram؛ ربات Telegram و بله فقط برای کاربرانی که ربات را آغاز کرده‌اند | بالا | شروع با وب و بله. اعلام محدودیت‌ها در هر Integration Target. داده رضایت در `Contact Channel` |
| R3 | منطق ترتیبی در n8n سخت‌تر تست و بازبینی می‌شود | متوسط | export به git، README، منطق در Karyar، n8n جدا برای test |
| R4 | رویداد گم‌شده (Webhook در Frappe retry خودکار ندارد) | متوسط | لاگ شکست، جست‌وجوی پرونده‌های بی‌حرکت، و در آینده Outbox |
| R5 | هزینه عملیاتی **n8n جدا برای هر Tenant** | متوسط | کانتینر سبک. ارزیابی دوباره پس از چند مشتری (n8n Enterprise، یا موتور داخلی) |
| R6 | برداشت اشتباه ایجنت در دستورهای گفت‌وگومحور | بالا | Change Proposal با Diff و تأیید صریح؛ hash |
| R7 | خستگی کاربر از تأییدهای زیاد | متوسط | دسته‌بندی تغییرها در یک Proposal. تعریف دقیق «تغییر حساس». ذخیره فرم به‌عنوان تأیید |
| R8 | زنجیره واگذاری و افزایش اختیار | بالا | اشتراک اختیار، عمق محدود، Delegation Policy، ممیزی |
| R9 | پیچیدگی Workspace (گفت‌وگو، پنل، Approval، واگذاری) | متوسط | پیاده‌سازی تدریجی، و شروع با یک فرایند پایلوت |
| R10 | پیش‌نمایش بدون ذخیره در ERPNext (تابع داخلی) | متوسط | spike در فاز ۱؛ راه جایگزین savepoint و rollback |
| R11 | Prompt Injection | بالا | مجوز مستقل از مدل؛ داده نامطمئن؛ Task Brief |
| R12 | هزینه AI از کنترل خارج شود | متوسط | سقف سخت و محدودیت گام |
| R13 | مجوزها: GPL و AGPL، و SUL در n8n | متوسط | بررسی حقوقی |
| R14 | داده پزشکی و حساس | بالا | مدل C برای Tenant، مدل خودمیزبان، پوشاندن فیلدها |
| R15 | ارتقای ERPNext ابزارها یا Promote را بشکند | متوسط | آزمون قرارداد، پین نسخه |

# ۲۹. تصمیم‌های باز (کسب‌وکاری و زیرساختی؛ مانع معماری نیستند)

1. **پایه نسخه:** پیشنهاد ERPNext و Frappe v16 به‌جای develop، با MariaDB.
2. **محل میزبانی** و سیاست ارسال داده به ارائه‌دهندگان AI.
3. **ارائه‌دهندگان AI مجاز** و مدل تجاری (کلید سکو یا BYOK).
4. **صنعت و مشتری پایلوت** (تعیین‌کننده اولین Pack).
5. **کانال‌های نسخه اول** (پیشنهاد: ویجت وب و بله).
6. **فناوری Workspace:** React (الگوی banking) یا frappe-ui.
7. **مجوز Karyar** و ادامه با n8n یا جایگزین آن (SUL).
8. **سیاست داده پزشکی** (در صورت پایلوت کلینیک).
9. **دسترسی کارکنان مشتری به Desk:** کدام نقش‌ها.

# ۳۰. اجزای موکول‌شده به آینده

| جزء | جای خالی در معماری فعلی |
|---|---|
| Workflow Engine داخلی و Automation Gateway (جایگزین n8n) | منطق، وضعیت و مجوز در Karyar است. READMEهای Workflowها. فهرست Integration Target |
| Outbox و Event Bus اختصاصی | `emit_event` و پوشش استاندارد رویداد |
| مهلت، escalation و جانشین تأییدکننده | `Approval Request` و Template |
| انجام Human Task از پیام‌رسان برای کارکنان | Channel Gateway |
| سازنده گرافیکی ایجنت، Policy و Template | DocTypeها و Packها |
| حافظه بلندمدت و RAG | Context Builder |
| صورت‌حساب‌دهی و مدیریت کامل هزینه | `UsageRecord`، `Model Price`، سقف سخت |
| صوت و MCP | Channel Adapter و رجیستری ابزار |
| RPA | `Agent Task`، Integration Target، `needs_input` |
| مودیان، حقوق ایران، جلالی کامل | `karyar_iran` و `karyar_hr_ir` |
| Control Plane، تجمیع مصرف، و Observability کامل | اسکریپت‌های deploy، `UsageSink`، `correlation_id` |
| PostgreSQL، Temporal، LiteLLM | کد مستقل از نوع پایگاه داده. AI Gateway |

# ۳۱. مثال‌های سرتاسری

> در همه مثال‌ها نام ایجنت‌ها فقط **انتساب** است. «n8n» یعنی Workflow کوتاهی که فقط Karyar API را صدا می‌زند.

## مثال ۱ — مشتری ← Ava ← تأیید مشتری ← بازبینی کارمند ← Lead در ERPNext ← Hanna (کلینیک)

<div class="flow"><span>مشتری در Karyar</span><span>Ava: پیش‌ثبت</span><span>تأیید مشتری</span><span>Workspace کارمند</span><span>Promote: Lead</span><span>Webhook</span><span>n8n</span><span>Hanna</span></div>

| مورد | جزئیات |
|---|---|
| **Trigger** | پیام مشتری در ویجت وب، بله یا اینستاگرام. گفت‌وگوی زنده **در Karyar** است و پرونده `clinic_intake` ساخته می‌شود |
| **Agent** | Ava (نقش `intake`). ابزارها: نرمال‌سازی، اعتبارسنجی موبایل و کد ملی. **بدون** ابزار Promote |
| **Data** | پیش‌ثبت در `Karyar Case`: نام، موبایل، خدمت، سن، شناسه اینستاگرام. **هنوز در ERPNext نیست** |
| **Event** | `intake.completed` (پس از تأیید مشتری، به hash ‏h1 گره خورده) |
| **Workflow steps** | n8n: ‏`human_tasks.create(template=staff_review)`، پایان. سپس رویداد `human_task.decided`. n8n: ‏`promote(case, crm.lead)`، سپس `agents.run_task(sales)` |
| **Human interaction** | کارمند پذیرش: «سن اشتباه است، ۳۸ کن». ایجنت Diff را نشان می‌دهد («سن: ۳۶ ← ۳۸؛ تأیید؟») و کارمند تأیید می‌کند. نسخه ۲ ثبت می‌شود. «شماره ثابت ندارد؛ از مشتری بگیر». یک Agent Task برای Ava ساخته می‌شود، Ava از مشتری می‌پرسد و نسخه ۳ با منبع «مشتری» ثبت می‌شود. کارمند: «بفرست مرحله بعد». Approval Policy عملیات `crm.lead` تک‌نفره است، پس تصمیم `approved` با hash ‏h3 ثبت می‌شود |
| **ERPNext action** | Promote: ‏Lead و Contact با کنترلرهای ERPNext، و ردیف `Contact Channel` (instagram) با رضایت. خلاصه گفت‌وگو در Timeline ‏Lead (`Communication` با medium برابر Chat) |
| **Next agent** | Webhook استاندارد `erp.lead.created`، سپس n8n، سپس `agents.run_task(sales)` برای **Hanna** |
| **Final result** | Lead در ERPNext. Hanna پیگیری را آغاز می‌کند و یک ToDo استاندارد برای مشاور می‌سازد. هیچ‌کس پرونده را دستی منتقل نکرد |

## مثال ۲ — Hanna ← سفارش ← تأیید پرداخت ← Arman ← تأیید چندمرحله‌ای ← سند حسابداری

| مورد | جزئیات |
|---|---|
| **Trigger** | ادامه `sales_followup`. مشتری پیش‌فاکتور را می‌پذیرد |
| **Agent** | Hanna (نقش `sales`)، سپس Arman (نقش `accounting`) |
| **Data** | پیش‌ثبت سفارش در Karyar، همراه با **پیش‌نمایش ERPNext بدون ذخیره** (قیمت، مالیات، جمع) و hash آن |
| **Workflow steps** | **(الف)** n8n: ‏`promote(case, selling.sales_order)`. یک **Auto-Execution Policy** («سفارش تا سقف X برای مشتری عادی»، نسخه ۳، تعریف‌شده توسط مدیر فروش) آن را مجاز می‌کند. بعد `human_tasks.create(payment_confirm)` برای صندوق‌دار. **(ب)** ‏`human_task.decided`، سپس n8n: ‏`agents.run_task(accounting)` |
| **Human interaction** | صندوق‌دار اطلاعات پرداخت را به‌صورت گفت‌وگو وارد می‌کند (Change Proposal و تأیید). Arman پیش‌ثبت Payment Entry و Sales Invoice را با **پیش‌نمایش دفتر کل** آماده می‌کند. **Approval Request ترتیبی:** حسابدار، سپس مدیر مالی (`strong`). حسابدار: «حساب بانک را ملت بگذار». با Diff و تأیید، **تأییدهای قبلی باطل می‌شوند** و تأیید مجدد گرفته می‌شود |
| **ERPNext action** | Promote: ‏Payment Entry و Sales Invoice با Submit. ثبت‌های دفتر کل را **خود ERPNext** می‌سازد. فاکتور با ایمیل استاندارد ERPNext و Print Format برای مشتری ارسال می‌شود |
| **Event** | `erp.sales_invoice.submitted` |
| **Next agent** | (آینده) فرایند مودیان در `karyar_iran` |
| **Final result** | سند مالی فقط پس از کامل شدن Approval با همان hash ثبت شد. ممیزی کامل است |

## مثال ۳ — یک ایجنت با چند نقش (شرکت کوچک)

| مورد | جزئیات |
|---|---|
| **Trigger** | همان Workflowهای مثال ۱ و ۲ (از همان Pack) |
| **Agent** | «سارا». انتساب نقش: ‏`intake`، `sales` و `accounting`، همه با سارا. تأیید مالی با مالک (انسان) |
| **Data / Workflow** | بدون تغییر تعریف. گام بازبینی کارمند با یک override در Tenant حذف شده است |
| **Human interaction** | فقط مالک: Approval تک‌نفره برای اسناد مالی |
| **نکته معماری** | ترکیب نقش‌ها **مجوزها را ادغام نمی‌کند.** در گفت‌وگو با مشتری فقط توانمندی‌های `intake` فعال است (محدوده گام). ممیزی ثبت می‌کند «سارا در نقش X در گام Y» عمل کرده است |
| **Final result** | همان نتیجه مثال‌های ۱ و ۲ با یک ایجنت و یک تأییدکننده |

## مثال ۴ — تأیید ترتیبی و M از N (خرید)

| مورد | جزئیات |
|---|---|
| **Trigger** | ERPNext بر اساس **سطح سفارش مجدد استاندارد** یک Material Request می‌سازد. Webhook ‏`erp.material_request.submitted` به n8n می‌رود |
| **Agent** | ایجنت نقش `inventory` |
| **Data** | پیش‌ثبت Purchase Order در Karyar، با پیش‌نمایش. تأمین‌کننده و قیمت از گزارش‌های استاندارد خرید |
| **Workflow steps** | n8n: ‏`agents.run_task(inventory)`، سپس `approvals.create(policy=po_over_threshold)` |
| **Human interaction** | **مرحله ۱:** یکی از دو سرپرست انبار. **مرحله ۲:** ۲ از ۳ عضو کمیته مالی، به‌صورت موازی. یکی از اعضای کمیته: «۲۰ عدد کم کن». با Diff و تأیید، **تأییدهای قبلی باطل می‌شوند** و هر دو مرحله دوباره تأیید می‌دهند |
| **ERPNext action** | Promote: ‏Purchase Order با Submit. ارسال برای تأمین‌کننده با **ایمیل استاندارد ERPNext و Print Format** (بدون n8n) |
| **Event** | `erp.purchase_order.submitted` |
| **Final result** | سفارش خرید فقط پس از کامل شدن هر دو مرحله با همان hash ثبت شد |

## مثال ۵ — ترکیب کاملاً متفاوت: تولیدکننده مواد غذایی

ایجنت‌ها: **Ava** (سفارش عمده از بله)، **نیما** (انبار و تولید)، **کیفیت**، و **مدیریت**. Hanna و Arman در این شرکت وجود ندارند.

| مورد | جزئیات |
|---|---|
| **Trigger** | پیام پخش‌کننده در بله. مخاطب از روی `Contact Channel` (bale) شناسایی می‌شود |
| **Agent** | Ava (`b2b_order_intake`)، سپس نیما، سپس کیفیت، سپس Ava (اعلان) |
| **Data** | پیش‌ثبت Sales Order. قیمت از Price List و اعتبار از Credit Limit استاندارد |
| **Workflow steps** | Auto-Execution Policy برای «مشتری معتبر و درون سقف اعتبار»، سپس Promote ‏Sales Order، سپس Webhook و n8n، سپس نیما (موجودی پیش‌بینی‌شده استاندارد). در صورت کمبود: پیش‌ثبت Work Order و Approval سرپرست تولید، سپس Promote. پس از رویداد Stock Entry ‏Manufacture: ایجنت کیفیت، سپس Quality Inspection |
| **Human interaction** | سرپرست تولید: «بگذار برای شیفت فردا صبح» (Change Proposal و تأیید). تکنسین کیفیت: «pH ‏۴٫۲، بریکس ۲۸». ایجنت مقادیر را با **قالب بازرسی کیفیت استاندارد ERPNext** مقایسه می‌کند و تکنسین تأیید می‌کند. انباردار: «بار زده شد» |
| **ERPNext action** | Sales Order، Work Order، Stock Entry، Quality Inspection و Delivery Note. **همه با کنترلرهای ERPNext** |
| **Next agent** | Ava از Channel Gateway در بله به مشتری اطلاع می‌دهد. ایجنت مدیریت با Cron در n8n هر روز گزارش خلاصه را برای مدیر می‌فرستد و پرسش‌ها را **با مجوز خود مدیر** پاسخ می‌دهد |
| **Final result** | همان موتور و همان ابزارها، با ترکیب کاملاً متفاوت |

## مثال ۶ — درخواست غیرمجاز

1. کارمند فروش: «سود کل شرکت امسال چقدر است؟»
2. ابزار `report.run(profit_and_loss)` بررسی می‌کند. نقش او به گزارش دسترسی ندارد، پس نتیجه `PERMISSION_DENIED` است و **هیچ عددی به مدل نمی‌رسد**.
3. ایجنت افراد صاحب صلاحیت را معرفی می‌کند.
4. رویداد `security.out_of_scope_request` ثبت می‌شود.

Prompt Injection نتیجه را تغییر نمی‌دهد.

## مثال ۷ — کمپین پیام با واگذاری به Integration

<div class="flow"><span>مدیر در Workspace</span><span>ایجنت مدیریت</span><span>گزارش کنترل‌شده</span><span>متن + Diff</span><span>Approval</span><span>Promote: Campaign</span><span>Agent Task</span><span>n8n: پیامک</span></div>

| مورد | جزئیات |
|---|---|
| **Trigger** | مدیر: «به همه مشتریانی که ماه گذشته خرید داشته‌اند پیام تبلیغاتی بفرست» |
| **Agent** | ایجنت مدیریت (از طرف مدیر) |
| **Data** | گزارش کنترل‌شده «مشتریان دارای فاکتور در بازه X به‌همراه رضایت کانال»، روی Sales Invoice و `Contact Channel`. خروجی: **ارجاع به Contactها**، نه کپی. حذف کسانی که انصراف داده‌اند |
| **Human interaction** | متن پیام پیشنهاد می‌شود و هر اصلاح آن Change Proposal است. **Approval ‏M از N** (مدیر و مسئول بازاریابی) به hash «فهرست و متن» گره می‌خورد |
| **ERPNext action** | Promote: رکورد **`Campaign` استاندارد ERPNext** |
| **Delegation** | طبق Delegation Policy (مدیریت به `messaging.sms.bulk_send`)، یک `Agent Task` ساخته می‌شود. n8n ارسال را **دسته‌ای** و با رعایت نرخ و انصراف انجام می‌دهد. وضعیت هر گیرنده با callback به Karyar برمی‌گردد |
| **Final result** | گزارش ارسال در Workspace. خلاصه در Timeline رکورد Campaign. ممیزی کامل. همین الگو برای WhatsApp، Telegram، Instagram و بله با Integration Target دیگر کار می‌کند (با محدودیت‌های R2) |

# ۳۲. جدول تصمیم‌های معماری

## الف) تصمیم‌های قطعی (تأییدشده)

| تصمیم | دلیل | جایگزین ردشده | قابل تغییر؟ |
|---|---|---|---|
| ERPNext موتور منطق و منبع نهایی حقیقت؛ ERPNext-first | جلوگیری از ERP دوم | منطق موازی در Karyar | خیر |
| Karyar: ایجنت، گفت‌وگو، Workspace، کنترل، مجوز، ممیزی، مصرف AI، فرایند | مرز روشن | — | خیر |
| n8n فقط برای هماهنگی و اجرای Integration | منطق و مجوز متمرکز در Karyar | منطق در n8n | با موتور داخلی در آینده |
| گفت‌وگوی زنده، اجرای ایجنت، تصمیم، مجوز، Human Task و عملیات نهایی در Karyar | کنترل | در n8n | خیر |
| Human Task به‌صورت Workspace مشترک و گفت‌وگومحور | تجربه کاربری غیر ERP | فقط دکمه تأیید و رد | خیر |
| Change Proposal با Diff و تأیید صریح برای تغییر حساس | جلوگیری از برداشت اشتباه AI | اعمال مستقیم | خیر |
| تأیید تک‌نفره، چندنفره، M از N، ترتیبی و موازی از فاز ۱ | نیاز کسب‌وکار | فقط تک‌نفره | خیر |
| واگذاری در Karyar با اشتراک اختیار | امنیت | واگذاری آزاد | خیر |
| زنجیره مجوز در کد؛ Prompt مجوز یا تأیید نمی‌سازد | امنیت واقعی | محدودیت در Prompt | خیر |
| بدون SQL و دسترسی خام به پایگاه داده؛ گزارش از ERPNext و Queryهای کنترل‌شده | امنیت و صحت | Text-to-SQL | خیر |
| مصرف AI به تفکیک Tenant، Agent، Workflow، User و Task | هزینه و شفافیت | — | خیر |
| جداسازی Tenant و Integrationهای Tenant-aware | امنیت | — | خیر |
| RPA فقط برای سیستم‌های بدون API؛ هرگز برای ERPNext؛ OTP و CAPTCHA به Human Task | پایداری | RPA روی ERPNext | خیر |
| `Contact Channel` روی Contact؛ حذف `Contact Identity` | داده تماس در ERP | جدول در Karyar | خیر |
| اجرای خودکار فقط با Policy قابل ممیزی | «تصمیم شخصی ایجنت» وجود ندارد | خودمختاری ایجنت | خیر |
| پیش‌ثبت پیش‌فرض فقط در Karyar؛ پیش‌نویس ERPNext فقط اختیاری و قفل‌شده | بدون ERP دوم و بدون دو محل ویرایش | پیش‌نویس بومی همیشگی | استثنا برای هر Workflow |
| hash پیش‌نمایش، Approval، و Re-Approval | «آنچه تأیید شد همان اجرا می‌شود» | — | خیر |
| پس از ثبت نهایی، خواندن فقط از ERPNext | یک منبع حقیقت | کش در Karyar | خیر |
| Tenant = مرز کسب‌وکار یا پروژه؛ شعبه با User Permission | ساده و استاندارد | پروژه درون Tenant | خیر |

## ب) پیشنهادی (هنوز تأیید نشده)

| تصمیم | دلیل | جایگزین | قابل تغییر؟ |
|---|---|---|---|
| پایه ERPNext و Frappe v16 با MariaDB | پایدار؛ Postgres در v16 رسمی نیست | develop، v15 | بله (v17) |
| مدل B (سایت برای هر Tenant) و C برای حساس‌ها | جداسازی پایگاه داده با هزینه معقول | A | بله |
| n8n جدا برای هر Tenant | n8n Community جداسازی ندارد | n8n مشترک | بله (Enterprise یا موتور داخلی) |
| چند مخزن و `karyar-deploy`؛ یک اپ `karyar` با ماژول‌ها | سازوکار bench و سادگی | تک‌مخزن یا چند اپ | دشوار |
| `Responsibility` برای ایجنت و انسان | یک مفهوم برای هر دو | جدا | بله |
| `ToDo`، Email، SMS Settings و `Communication` استاندارد | ERPNext-first | ساختن سیستم موازی | بله |
| Workspace به‌صورت SPA درون Frappe | بدون سرور جدا | سرویس جدا | بله |
| AI Gateway به‌صورت کتابخانه و سقف سخت هزینه | ساده و امن | LiteLLM از ابتدا | بله |
| پیش‌نمایش با تابع داخلی ERPNext | بدون بازنویسی محاسبات | savepoint و rollback | بله (spike) |

## ج) پرسش‌های باز

میزبانی و اقامت داده، ارائه‌دهندگان AI و BYOK، صنعت پایلوت، کانال‌های نسخه اول، فناوری Workspace، مجوز Karyar و n8n، سیاست داده پزشکی، دسترسی به Desk (§۲۹).

## د) آینده

همان فهرست §۳۰.

# ۳۳. تعارض‌ها: حل‌شده و باقی‌مانده

**حل‌شده در این نسخه:**

| تعارض (از v0.1 و CRها) | حل |
|---|---|
| Workflow Engine اختصاصی (v0.1 §۷) در برابر n8n | n8n فقط برای هماهنگی. منطق، وضعیت و مجوز در Karyar. موتور اختصاصی موکول به آینده |
| Automation Gateway (CR-01a) | حذف. فقط فهرست حداقلی Integration Target |
| «اول پیش‌نویس در ERPNext» (P8 در v0.1) در برابر «پیش‌ثبت در Karyar» (CR-02) | پیش‌فرض Karyar. پیش‌نویس ERPNext فقط اختیاری و قفل‌شده |
| پیش‌ثبت قابل مشاهده در هر دو سیستم (CR-04) در برابر «فقط Karyar» (CR-02) | یک Site مشترک. Case در Desk و Connections دیده می‌شود. آینه اختیاری |
| ویرایش مستقیم ایجنت (CR-03) در برابر تأیید صریح (CR-04) | Change Proposal برای هر تغییر حساس |
| «M از N» موکول به آینده (CR-01 و CR-03) در برابر الزام فاز ۱ | مدل یکپارچه Approval در فاز ۱ |
| «فقط رویداد بین ایجنت‌ها» (v0.1 §۱۰) در برابر واگذاری | واگذاری کنترل‌شده در Karyar؛ حرکت مراحل با n8n |
| `Contact Identity` در Karyar در برابر داده کانال در ERP | `Contact Channel` روی Contact |
| «اجرای خودکار» در برابر «تصمیم شخصی ایجنت وجود ندارد» | اجرای خودکار تصمیم انسانیِ از پیش گرفته‌شده در Policy است |
| تأییدهای Frappe Workflow و Karyar روی یک سند | Frappe Workflow فقط برای اسناد ساخته‌شده در Desk |
| ساختن سیستم اعلان، تخصیص و تاریخچه موازی | `ToDo`، Notification، Email، SMS Settings، `Communication` و `Version` |

**باقی‌مانده:** **هیچ تعارض فنی بازی** که بدون تصمیم جدید حل نشود وجود ندارد. تنش‌های باقی‌مانده به هزینه یا حقوق مربوط‌اند، نه به معماری: هزینه n8n جدا برای هر Tenant (R5)، مجوز SUL در n8n (R13)، و محدودیت کانال‌ها (R2).

# ۳۴. خروجی نهایی

## ۳۴.۱ معماری در یک نگاه

- **ERPNext (v16):** منطق و حقیقت نهایی کسب‌وکار.
- **Karyar:** لایه هوشمند، کنترل‌کننده و رابط. شامل ایجنت، گفت‌وگو، Workspace، Change Proposal، Approval، واگذاری، مجوز، Promote، ممیزی و مصرف AI.
- **n8n (برای هر Tenant):** هماهنگی گام‌ها و اجرای Integration.

## ۳۴.۲ توضیح ساده

- **ERPNext دفتر رسمی و حسابدار شرکت است.** فقط آنچه آنجا ثبت شود واقعی است و محاسبات را خودش انجام می‌دهد.
- **Karyar میز کار هوشمند است.** ایجنت‌ها با مشتری و کارکنان حرف می‌زنند، پیش‌نویس آماده می‌کنند و تغییر پیشنهاد می‌دهند.
- **انسان در Workspace گفت‌وگو می‌کند، تأیید می‌کند و کار را هدایت می‌کند.** فقط بعد از تأیید (یا Policy مصوب) و فقط از دروازه Promote، سند در ERPNext ثبت می‌شود.
- **n8n زنگ مرحله بعد را می‌زند و پیامک‌ها را می‌فرستد**، ولی تصمیم نمی‌گیرد.

## ۳۴.۳ نمودار اصلی

بخش ۳ را ببینید.

## ۳۴.۴ ساختار مخازن

بخش ۲۳ را ببینید.

## ۳۴.۵ ریسک‌های بزرگ

R1 (AI در ایران)، R2 (کانال‌ها)، R6 و R8 (برداشت اشتباه و افزایش اختیار)، R5 (هزینه n8n)، R11 (Prompt Injection).

## ۳۴.۶ نقشه راه

> زمان‌ها **تقریبی** و برای تیم ۲ تا ۳ نفره است.

| فاز | محتوا | معیار خروج |
|---|---|---|
| **۰. پایه** (۱ تا ۲ هفته) | پاسخ به تصمیم‌های باز فوری (§۲۹: ۱ تا ۵). انتقال fork به `version-16`. ساخت `karyar` و `karyar-deploy`. bench توسعه (v16 و HRMS). n8n برای Tenant توسعه. CI با semgrep | اپ خالی نصب می‌شود و CI سبز است |
| **۱. هسته** (۶ تا ۸ هفته) | DocTypeهای §۲۳. زنجیره مجوز و Tool Executor. AI Gateway با UsageRecord و سقف سخت. Agent Runtime. Channel Gateway (وب و بله). Case و پیش‌ثبت و نسخه‌ها. Change Proposal. **Approval یکپارچه**. Agent Task و Delegation Policy. Promote به‌همراه spike پیش‌نمایش. API v1. `emit_event` و Webhookهای Frappe. Workspace نسخه ۰ (گفت‌وگو و پنل). `Contact Channel`. مبانی `karyar_iran` | گردش نمونه سرتاسری؛ تست‌های مجوز، واگذاری و Approval پاس می‌شوند |
| **۲. پایلوت** (۴ تا ۶ هفته) | Pack صنعت پایلوت. مثال ۱ سرتاسری. ۳ تا ۵ Workflow در n8n. Report Catalog. پیامک | پایلوت داخلی و بازبینی امنیتی |
| **۳. مالی و استحکام** (۴ تا ۶ هفته) | مثال ۲ (Approval ترتیبی و `strong`). داشبورد پرونده‌های بی‌حرکت. پشتیبان‌گیری و بازیابی. تست بار. staging. Print Formatهای فارسی | چک‌لیست راه‌اندازی |
| **۴. اولین مشتری** | Pack به‌همراه ۲۰٪ پیکربندی. پشتیبانی فشرده. اندازه‌گیری هزینه و زمان‌ها | مشتری فعال |
| **بعد** | §۳۰ | به ترتیب نیاز |

---

<p class="endnote">نسخه ۱٫۰ — جایگزین نسخه ۰٫۱ و همه تغییرات ثبت‌شده در PENDING_CHANGES (CR-01 تا CR-04). منابع: بررسی مستقیم کد این مخزن و کد Frappe و ERPNext نسخه ۱۶، مستندات n8n، issue شماره 56241 در frappe/erpnext، مخزن frappe/mcp.</p>
