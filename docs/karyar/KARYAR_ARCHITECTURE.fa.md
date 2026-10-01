<div class="cover">
<h1 class="title">معماری پایه Karyar</h1>
<p class="subtitle">ERPNext + Karyar Agent Platform</p>
<p class="meta">سند طراحی معماری — نسخه ۰٫۱ (پیش‌نویس برای بازبینی) — مهر ۱۴۰۵ / اکتبر ۲۰۲۶</p>
<p class="meta">وضعیت: <b>پیشنهادی</b>. هیچ تصمیمی در این سند نهایی نیست، مگر در بخش «اصول تأییدشده».</p>
</div>

## فهرست

- بخش صفر: خلاصه اجرایی و وضعیت فعلی مخزن
- بخش ۱ تا ۳۰: معماری
- بخش ۳۱: شش مثال سرتاسری
- بخش ۳۲: جدول تصمیم‌های معماری
- بخش ۳۳: تعارض‌ها و محدودیت‌هایی که باید صریح گفته شوند
- بخش ۳۴: خروجی نهایی و نقشه راه

---

# بخش صفر — خلاصه اجرایی و وضعیت فعلی مخزن

## ۰.۱ خلاصه در یک پاراگراف

Karyar یک **سکوی ایجنت (Agent Platform)** است که روی ERPNext قرار می‌گیرد. ERPNext منبع حقیقت (Source of Truth) داده‌های کسب‌وکار است: مشتری، سفارش، انبار و حسابداری. Karyar چهار کار انجام می‌دهد:

- **پیام‌ها را می‌گیرد** و آن‌ها را به گفت‌وگوهای استاندارد تبدیل می‌کند.
- **پرونده‌های کاری را جلو می‌برد**: هر پرونده (Workflow Run) طبق یک «تعریف فرایند» قابل‌پیکربندی حرکت می‌کند.
- **ایجنت‌ها را داخل این پرونده‌ها به کار می‌گیرد**: ایجنت‌ها فقط از طریق «ابزارهای کنترل‌شده» با ERPNext کار می‌کنند.
- **انسان را در نقاط تعریف‌شده وارد می‌کند**: گفت‌وگومحور و ثبت‌شده.

انتقال کار بین ایجنت‌ها **همیشه** از راه رویداد و تغییر وضعیت پرونده است، نه گفت‌وگوی آزاد بین هوش‌های مصنوعی.

دسترسی‌ها در لایه‌ای **مستقل از مدل هوش مصنوعی** اعمال می‌شوند و نهایتاً به همان سیستم مجوز Frappe می‌رسند.

## ۰.۲ وضعیت فعلی مخزن (یافته‌های بررسی)

| مورد | یافته |
|---|---|
| مخزن | `karyar-mainERP`، یک fork مستقیم از `frappe/erpnext` |
| شاخه و commit | `claude/keen-hypatia-07r3hv`، برابر با `develop` در commit ‏`3257b70` |
| نسخه ERPNext | `17.0.0-dev`، یعنی شاخه توسعه و **ناپایدار** |
| نسخه Frappe مورد نیاز | `>=17.0.0-dev,<18` (خود Frappe در مخزن نیست) |
| Python / Node | Python ‏3.14 یا بالاتر، Node ‏24 |
| اپ‌های موجود | فقط `erpnext` و یک SPA بانکداری با React (`banking/`). اپ‌های frappe، hrms و Karyar در مخزن نیستند |
| سفارشی‌سازی Karyar | **هیچ‌چیز.** کلمه «karyar» در کد وجود ندارد و همه commitها از Frappe هستند |
| حجم کد | حدود ۴۳۰ هزار خط Python و ۸۷ هزار خط JS، در ۲۱ ماژول |
| وابستگی‌های Python | `rapidfuzz`، `holidays`، `pdfplumber`، `mt-940`، `plaid-python` و چند مورد دیگر |

**نقاط توسعه موجود** (یعنی جاهایی که Karyar می‌تواند بدون دست زدن به هسته وصل شود):

| نقطه توسعه | کاربرد برای Karyar |
|---|---|
| `doc_events` (از جمله `"*"` برای همه DocTypeها) | گرفتن رویدادهای ERPNext |
| `scheduler_events` و صف‌های سفارشی RQ (کلید `workers` در `common_site_config`) | اجرای ایجنت‌ها و کارهای پس‌زمینه |
| `frappe.db.after_commit` و `enqueue(..., enqueue_after_commit=True)` | انتشار مطمئن رویداد، **فقط بعد از** commit تراکنش |
| `has_permission`، `permission_query_conditions`، User Permission و permlevel | کنترل دسترسی در سطح سند، رکورد و فیلد |
| `extend_doctype_class` و `override_doctype_class` | توسعه رفتار DocTypeهای ERPNext بدون تغییر هسته |
| `regional_overrides` و `naming_series_variables` | بومی‌سازی ایران، مثلاً شماره‌گذاری جلالی |
| `setup_wizard_stages`، `boot_session` و `jinja` | نصب، بارگذاری اولیه و قالب‌های چاپ |
| Frappe Workflow (در v16 با Transition Task)، Assignment Rule، Notification | امکانات آماده برای گردش سند و تخصیص کار |
| Webhook با امضای HMAC (`X-Frappe-Webhook-Signature`)، OAuth Client، API نسخه ۱ و ۲ | اتصال به سیستم‌های بیرونی |
| Version، Access Log، Error Log، RQ Job | پایه ردیابی و ممیزی |
| Socket.io و `publish_realtime` | پخش پاسخ‌های بلادرنگ در رابط کاربری |

**محدودیت‌هایی که روی طراحی اثر می‌گذارند:**

1. **Frappe Workflow فقط ماشین حالتِ یک سند است.** گردش بین چند سند، تایمر، تأیید موازی «۲ از ۳» و گام ایجنت ندارد.
2. **`frappe.get_all` مجوزها را نادیده می‌گیرد.** ابزارهای ایجنت **نباید** از آن استفاده کنند.
3. **سرور Frappe همگام (WSGI) است.** فراخوانی‌های طولانی مدل زبانی نباید در پروسه وب اجرا شوند و باید به Workerها بروند.
4. **چارت حساب فقط از پوشه خود ERPNext بارگذاری می‌شود.** برای چارت ایرانی باید از `create_charts(custom_chart=...)` استفاده شود.
5. **برخی گزارش‌های استاندارد با SQL خام نوشته شده‌اند** و کنترل دسترسی آن‌ها فقط در سطح نقش (Role) است. پس فقط فهرستی گزینش‌شده از گزارش‌ها باید در اختیار ایجنت قرار گیرد.
6. **ERPNext نسخه ۱۶ روی PostgreSQL پشتیبانی رسمی ندارد.**

> **تعارض با چشم‌انداز:** این مخزن خودِ ERPNext است. کد Karyar نباید داخل آن نوشته شود، چون این کار عملاً تغییر هسته است و هر ارتقا را دردناک می‌کند. بخش ۲۳ ساختار درست مخزن‌ها را پیشنهاد می‌دهد. این سند فعلاً موقتاً در همین مخزن (`docs/karyar/`) قرار گرفته است.

---

# ۱. نمای کلی معماری

Karyar از **چهار لایه منطقی** ساخته می‌شود. یک لایه **عمودی** هم از همه آن‌ها عبور می‌کند:

1. **لایه تعامل:** کانال‌ها (وب‌سایت، تلگرام، بله، واتس‌اپ و…) به‌علاوه «میز کار کاریار» (Karyar Workspace) برای کارکنان. همه این‌ها به «درگاه کانال» (Channel Gateway) وصل می‌شوند. وظیفه درگاه این است که پیام هر کانال را به یک پیام استاندارد تبدیل کند.
2. **لایه هوشمندی و فرایند:**
   - **موتور گردش‌کار** (Workflow Engine): تعیین می‌کند **چه کاری** و **کِی** انجام شود.
   - **اجراکننده ایجنت** (Agent Runtime): تعیین می‌کند **چطور** آن کار با زبان طبیعی و به‌صورت هوشمند انجام شود.
   - **سرویس کار انسانی** (Human Task Service): مدیریت تعامل انسان در گردش‌کار.
   - **سرویس گفت‌وگو** (Conversation Service).
3. **لایه کنترل:**
   - **رجیستری توانمندی و ابزار:** فهرست کارهایی که ایجنت مجاز است انجام دهد.
   - **موتور سیاست/مجوز** (Policy Engine).
   - **اجراکننده ابزار** (Tool Executor).
   - **API پایدار کاریار** (Karyar API v1).
   - **گذرگاه رویداد** (Event Bus).
   - **درگاه مدل هوش مصنوعی** (AI Gateway).
4. **لایه داده و عملیات:** ERPNext، HRMS و `karyar_iran` روی Frappe. این تنها منبع حقیقت داده‌های کسب‌وکار است.
- **لایه عمودی:** ممیزی، ردیابی، مصرف هوش مصنوعی و امنیت.
- **بیرون از هسته:** n8n برای یکپارچه‌سازی‌ها، Workerهای RPA و سرویس‌های بیرونی.

**ایده محوری:** «Karyar فرایند را تعریف می‌کند؛ هوش مصنوعی درون فرایند هوشمندانه کار می‌کند.» گردش‌کار **قطعی (deterministic)** و قابل ممیزی است. ایجنت **احتمالاتی (probabilistic)** است و فقط در قالب گام‌ها و ابزارهای مجاز عمل می‌کند.

# ۲. اصول بنیادین

| # | اصل | پیامد فنی |
|---|---|---|
| P1 | ERPNext منبع حقیقت داده‌های کسب‌وکار است | Karyar فقط «وضعیت فرایند» را نگه می‌دارد، نه داده موازی |
| P2 | هیچ ایجنتی به پایگاه داده دسترسی خام ندارد | فقط ابزارهای تایپ‌شده و از پیش ثبت‌شده. SQL آزاد وجود ندارد |
| P3 | مجوز مستقل از مدل هوش مصنوعی اعمال می‌شود | بررسی در Tool Executor و Frappe انجام می‌شود، نه در prompt |
| P4 | فرایندها پیکربندی‌اند، نه کد | Workflow Definition نسخه‌دار. کد مشتری‌محور فقط از راه «نقاط توسعه» |
| P5 | ایجنت، نقش کاری، توانمندی، مجوز و گردش‌کار از هم جدا هستند | گردش‌کار به **نقش کاری** اشاره می‌کند، نه به ایجنت مشخص |
| P6 | دخالت انسان (HITL) یک گام یا سیاست قابل‌پیکربندی است | هیچ قانون ثابتی مثل «Arman همیشه تأیید می‌خواهد» در کد نیست |
| P7 | انتقال کار بین ایجنت‌ها فقط از راه رویداد و وضعیت | چت آزاد ایجنت با ایجنت وجود ندارد. داده ساخت‌یافته منتقل می‌شود |
| P8 | نوشتن در ERPNext ابتدا به‌صورت پیش‌نویس است | ثبت نهایی (Submit) یک اقدام جداگانه است که سیاست‌ها کنترلش می‌کنند |
| P9 | آنچه تأیید شده، دقیقاً همان است که اجرا می‌شود | تأیید به hash محتوا گره می‌خورد و هر تغییری تأیید را باطل می‌کند |
| P10 | به هیچ ارائه‌دهنده هوش مصنوعی گره نخوردن | AI Gateway با نام مستعار مدل (alias) کار می‌کند |
| P11 | هر کاری قابل ردیابی است | `correlation_id` در همه لایه‌ها و Audit Log معنایی |
| P12 | هسته ERPNext دست‌نخورده می‌ماند | Hooks، اپ سفارشی و API. هر وصله ضروری مستند می‌شود |
| P13 | ساده شروع کن، درست مرز بکش | یک اپ Frappe با ماژول‌های جدا. جداسازی به سرویس مستقل فقط هنگام نیاز |

# ۳. نمودار معماری سیستم

<div class="arch">
  <div class="arch-main">
    <div class="layer l-channel"><div class="lname">کانال‌ها و رابط‌ها</div>
      <div class="boxes"><span>وب‌سایت (ویجت)</span><span>تلگرام</span><span>بله / ایتا</span><span>واتس‌اپ</span><span>اینستاگرام</span><span>صوت (آینده)</span><span class="hl">میز کار کاریار (کارکنان)</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-gw"><div class="lname">درگاه کانال — Channel Gateway</div>
      <div class="boxes"><span>Adapterهای کانال</span><span>تشخیص هویت مخاطب</span><span>پیام استاندارد</span><span>تأیید امضای Webhook</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-brain"><div class="lname">هوشمندی و فرایند</div>
      <div class="boxes"><span class="hl">موتور گردش‌کار<br><small>Workflow Engine</small></span><span class="hl">اجراکننده ایجنت<br><small>Agent Runtime</small></span><span>کار انسانی<br><small>Human Task</small></span><span>گفت‌وگو<br><small>Conversation</small></span></div></div>
    <div class="arrow">▼</div>
    <div class="layer l-control"><div class="lname">کنترل — هیچ مسیری از کنار این لایه رد نمی‌شود</div>
      <div class="boxes"><span class="hl">Policy Engine<br><small>مجوز و ریسک</small></span><span class="hl">Tool Executor</span><span>رجیستری توانمندی و ابزار</span><span>Karyar API v1</span><span>Event Bus<br><small>Outbox + RQ</small></span><span>AI Gateway</span></div></div>
    <div class="arrow">▼ ▲</div>
    <div class="layer l-erp"><div class="lname">منبع حقیقت — Frappe Site (یک سایت برای هر Tenant)</div>
      <div class="boxes"><span class="hl">ERPNext</span><span>HRMS</span><span>karyar_iran</span><span>Frappe: مجوز، Workflow، Version، Webhook</span></div></div>
    <div class="arrow">▼</div>
    <div class="layer l-infra"><div class="lname">زیرساخت</div>
      <div class="boxes"><span>MariaDB (پایگاه داده جدا برای هر Tenant)</span><span>Redis (کش و صف)</span><span>فایل‌ها / S3</span><span>Workerها و Scheduler</span></div></div>
  </div>
  <div class="arch-side">
    <div class="side s-audit"><div class="lname">عمودی</div><span>Audit Log</span><span>Trace / Correlation</span><span>AI Usage Record</span><span>متریک و لاگ</span><span>مدیریت اسرار</span></div>
    <div class="side s-ext"><div class="lname">بیرون از هسته</div><span>n8n (یکپارچه‌سازی)</span><span>RPA Workers</span><span>ارائه‌دهندگان AI</span><span>APIهای بیرونی (مودیان، پیامک، بانک)</span></div>
  </div>
</div>

**جهت جریان کنترل:**

- **کانال ← گردش‌کار:** کانال پیام را به درگاه می‌دهد. درگاه آن را به یک Conversation وصل می‌کند و یک رویداد `channel.message.received` منتشر می‌کند. موتور گردش‌کار تصمیم می‌گیرد پیام متعلق به کدام پرونده است و کدام ایجنت باید پاسخ دهد.
- **ایجنت ← ابزار ← ERPNext:** ایجنت فقط «ابزار» صدا می‌زند. Tool Executor سیاست‌ها را بررسی می‌کند و ابزار با هویت مجاز روی ERPNext اجرا می‌شود.
- **ERPNext ← رویداد ← گردش‌کار:** تغییرات ERPNext رویداد تولید می‌کنند و موتور گردش‌کار گام بعدی را فعال می‌کند. این «تحویل خودکار» بین ایجنت‌هاست.

# ۴. معماری اجزا

برای هر جزء: چرا وجود دارد، چه مسئله‌ای حل می‌کند، کِی لازم است و چه جایگزین‌هایی بررسی شد.

| جزء | مسئولیت | چرا | الان یا بعد | جایگزین‌های بررسی‌شده |
|---|---|---|---|---|
| **Channel Gateway** | نرمال‌سازی پیام‌ها، شناسایی مخاطب، ارسال پاسخ، تأیید امضا | جدا کردن کانال از منطق ایجنت | **الان** (۱ تا ۲ کانال) | درون Frappe یا سرویس جدا. فعلاً درون Frappe و در صورت حجم بالا جدا شود |
| **Conversation Service** | نگهداری گفت‌وگو و پیام و اتصال آن به پرونده | حافظه کوتاه‌مدت و ردیابی | **الان** | استفاده از Communication در Frappe؛ برای چت ساخت‌یافته مناسب نیست |
| **Agent Runtime** | اجرای یک «نوبت» ایجنت: ساخت context، فراخوانی مدل، اجرای ابزارها، اعتبارسنجی خروجی | هسته هوشمندی | **الان** (ساده) | LangGraph، CrewAI، Agents SDK. تصمیم: runtime سبک خودمان با رابط تمیز (بخش ۵) |
| **Workflow Engine** | اجرای پایدار گراف فرایند و نگهداری وضعیت پرونده | تحویل خودکار، HITL، تایمر | **الان** (حداقلی) | Frappe Workflow، Temporal، n8n، Camunda (بخش ۷) |
| **Human Task Service** | کار انسانی، تأیید، ویرایش گفت‌وگومحور | HITL قابل‌پیکربندی | **الان** | ToDo و Workflow Action در Frappe؛ ناکافی‌اند |
| **Capability & Tool Registry** | تعریف ابزارهای تایپ‌شده و بسته‌بندی آن‌ها در توانمندی‌ها | جلوگیری از دسترسی خام | **الان** | MCP به‌عنوان تنها لایه؛ MCP فقط یک «نمایش» از همین رجیستری است |
| **Policy Engine** | ترکیب سطوح مجوز و سیاست ریسک | امنیت مستقل از مدل | **الان** (پایه) | OPA یا Cedar؛ برای شروع زیادی است |
| **Tool Executor** | تنها مسیر اجرای ابزار، همراه با ممیزی، idempotency و فیلتر خروجی | نقطه واحد کنترل | **الان** | — |
| **Event Bus** | Outbox در پایگاه داده + dispatcher روی RQ | تحویل مطمئن رویداد | **الان** | Kafka، RabbitMQ، NATS؛ فعلاً لازم نیست |
| **Karyar API v1** | API پایدار و نسخه‌دار برای n8n، اپ‌ها و سیستم‌های بیرونی | محافظت از مصرف‌کننده‌ها در برابر تغییرات ERPNext | **الان** (کوچک) | دسترسی مستقیم به `/api/resource`؛ شکننده است |
| **AI Gateway** | انتزاع ارائه‌دهنده، نام مستعار مدل، fallback، ثبت مصرف | مستقل از ارائه‌دهنده و آماده ثبت هزینه | **الان** (کتابخانه) | LiteLLM Proxy (گزینه بعدی) |
| **Audit & Telemetry** | Audit Log معنایی، Trace، متریک | ممیزی و عیب‌یابی | **الان** (پایه) | OpenTelemetry کامل (بعد) |
| **Karyar Workspace** | رابط کارکنان: صندوق کار، گفت‌وگو، پرونده‌ها | کاربر نباید مجبور به کار با ERPNext Desk باشد | **الان** (حداقلی) | فقط تلگرام یا بله برای کارکنان؛ برای کار جدی کافی نیست |
| **n8n** | یکپارچه‌سازی و اعلان | اتصال سریع به سرویس‌های بیرونی | **فاز ۲** | کدنویسی مستقیم هر اتصال |
| **RPA Service** | تعامل با سیستم‌های بدون API | پورتال‌ها | **فاز ۳ به بعد** | — |
| **Control Plane** | ثبت Tenant، راه‌اندازی، بسته‌ها، اندازه‌گیری مصرف مرکزی | چندمستأجری در مقیاس | **بعد** (اول با اسکریپت) | Frappe Press (AGPL) |

# ۵. معماری ایجنت

## ۵.۱ مفاهیم جداشده

یکی از مهم‌ترین تصمیم‌های این سند، **جدا کردن پنج مفهوم** از هم است:

| مفهوم | تعریف | مثال |
|---|---|---|
| **Capability (توانمندی)** | یک توانایی اتمی که به یک یا چند ابزار نگاشت می‌شود و سطح ریسک دارد | `crm.lead.draft`، `report.sales_summary.read`، `accounts.journal.draft` |
| **Responsibility (نقش کاری)** | بسته‌ای از توانمندی‌ها به‌علاوه دستورالعمل حرفه‌ای | «فروش»، «CRM»، «حسابداری»، «پذیرش مشتری» |
| **Agent (ایجنت)** | یک شخصیت اجرایی شامل نام، لحن، کانال‌ها، مدل و یک یا چند نقش کاری | Ava، Hanna، Arman، «سارا» (چند نقش در یک ایجنت) |
| **Role Assignment (انتساب نقش)** | تعیین اینکه در هر Tenant، هر نقش کاری را کدام ایجنت(ها) یا کاربر(ها) انجام می‌دهند | شرکت C: فروش + CRM + حسابداری ← «سارا» |
| **Frappe Role / Permission** | مجوز واقعی سیستم روی DocTypeها و رکوردها | Sales User، Accounts Manager |

**قاعده کلیدی:** گام‌های گردش‌کار به **نقش کاری** اشاره می‌کنند، نه به ایجنت. یعنی گام می‌گوید «انجام توسط: نقش حسابداری» و جدول انتساب در هر Tenant مشخص می‌کند این نقش با کیست.

نتیجه: اگر یک ایجنت چند نقش داشته باشد یا یک نقش بین چند ایجنت یا انسان تقسیم شود، **هیچ تعریف گردش‌کاری عوض نمی‌شود**.

> **نکته نام‌گذاری (نیاز به تأیید):** کلمه Role در Frappe معنای «نقش دسترسی» دارد. برای جلوگیری از اشتباه، در کد از `Responsibility` استفاده می‌کنیم و در رابط کاربری فارسی از «نقش کاری».

## ۵.۲ تعریف ایجنت (DocType: `Karyar Agent`)

- **شناسه:** نام، شخصیت و لحن، زبان، آواتار
- **نقش‌های کاری:** یک یا چند Responsibility. توانمندی‌ها از اجتماع (union) آن‌ها به دست می‌آیند و قابل محدودسازی‌اند
- **کانال‌های مجاز** و **مخاطب مجاز:** مشتری بیرونی، کارمند یا هر دو
- **پروفایل مدل:** یک نام مستعار مثل `smart` یا `fast` که در AI Gateway به مدل واقعی نگاشت می‌شود
- **قالب دستورالعمل:** نسخه‌دار. دستورالعمل از چند لایه ترکیب می‌شود: پایه سکو، نقش کاری، تنظیمات Tenant، و گام گردش‌کار
- **هویت اجرایی:** یک کاربر Frappe از نوع سرویس (بدون امکان ورود) برای کارهای خودکار
- **سیاست حافظه:** فعلاً فقط گفت‌وگو و context پرونده. حافظه بلندمدت در فاز بعد
- **محدودیت‌ها:** حداکثر تعداد فراخوانی ابزار در هر نوبت، حداکثر token، و سقف هزینه (جای خالی برای فاز بعد)

## ۵.۳ حالت‌های اجرای ایجنت

1. **نوبت گفت‌وگو:** پیامی از کاربر یا مشتری می‌رسد و ایجنت پاسخ می‌دهد. ممکن است ابزار بخواند یا پیشنهاد اقدام بدهد.
2. **گام گردش‌کار:** موتور گردش‌کار به ایجنت یک «دستور کار» ساخت‌یافته می‌دهد (هدف، داده پرونده، توانمندی‌های مجاز در همین گام، و Schema خروجی). ایجنت باید خروجی معتبر طبق Schema تحویل دهد.
3. **دستیار بازبینی:** در گام انسانی، ایجنت به کارمند کمک می‌کند داده را بررسی و اصلاح کند (بخش ۸).
4. **(آینده)** کارهای زمان‌بندی‌شده و تحلیلی، مثل «هر صبح پیگیری‌های عقب‌افتاده را بررسی کن». حتی این‌ها هم یک گردش‌کار تعریف‌شده‌اند، نه ابتکار ایجنت.

## ۵.۴ چرخه یک نوبت ایجنت

<pre class="ltr">
1. Load: Agent config + Responsibilities → effective Capabilities (∩ step scope)
2. Build context: conversation window + run context (structured) + task brief
3. AI Gateway.complete(alias, messages, tools=allowed_tool_schemas, response_schema?)
4. For each tool call → Tool Executor:
      policy.check(principal, capability, scope, risk) → execute → filter output → audit
5. Loop (bounded: max_steps, max_tokens, timeout)
6. Validate final output against schema (retry once on failure → else escalate to human)
7. Persist: messages, step output, usage record, audit; emit events
</pre>

- **محل اجرا:** Workerهای RQ با صف اختصاصی `karyar_agent`، نه پروسه وب.
- **پخش پاسخ:** با `publish_realtime` در Socket.io انجام می‌شود.
- **چرا runtime سبک خودمان، نه چارچوب آماده؟** ما به سه چیز احتیاج داریم:
  - یکپارچگی عمیق با هویت و مجوز Frappe و تراکنش‌های آن
  - چندمستأجری در سطح Site
  - کنترل کامل روی ممیزی

  حلقه tool-calling هم پیچیده نیست. چارچوب‌هایی مثل LangGraph می‌توانند **داخل** یک گام برای استدلال‌های پیچیده به کار بروند، ولی **مالک وضعیت فرایند نیستند**.

## ۵.۵ پیشنهاد اقدام (Action Proposal)

ایجنت مستقیماً «کار حساس» انجام نمی‌دهد. وقتی ابزاری از نوع `commit-write` یا `external` باشد و سیاست بگوید «نیاز به تأیید دارد»، روند این است:

1. Tool Executor به‌جای اجرا، یک **Action Proposal** می‌سازد که شامل ابزار، ورودی، خلاصه انسانی و hash است.
2. این پیشنهاد به گام انسانی می‌رود.
3. بعد از تأیید، **همان ورودی دقیق** اجرا می‌شود.

به این ترتیب HITL دو منبع دارد:

- **صریح:** یک گام انسانی که در گراف گردش‌کار تعریف شده
- **ضمنی:** سیاست ریسک Tenant، مثلاً «هر ثبت نهایی سند حسابداری بالای ۵۰ میلیون ریال تأیید مدیر مالی می‌خواهد»

هر دو قابل‌پیکربندی‌اند.

# ۶. معماری سازنده ایجنت (Agent Builder)

**فاز ۱: بدون رابط گرافیکی.** پیکربندی با DocTypeهای Frappe انجام می‌شود (فرم‌های Desk برای تیم Karyar) و به‌صورت **بسته (Pack)** نسخه‌دار ذخیره می‌شود.

**ساختار بسته:**

<pre class="ltr">
Karyar Pack (versioned, e.g. "clinic-basic@1.2.0")
├── responsibilities/*.json   (capabilities + instructions)
├── agents/*.json             (persona, channels, model alias, responsibilities)
├── workflows/*.json          (versioned graph definitions)
├── role_assignments.json     (tenant-overridable defaults)
├── policies.json             (risk thresholds, HITL defaults)
├── reports.json              (report catalog entries)
└── erp_customizations/       (custom fields, property setters — Frappe fixtures)
</pre>

- **اعمال روی Tenant:** دستور `bench --site X karyar apply-pack clinic-basic@1.2.0` اجرا می‌شود و تفاوت‌ها با تنظیمات خود Tenant ادغام می‌شوند.
- **چرا Pack:** این همان سازوکار ۸۰/۲۰ است. ۸۰٪ مشترک در Pack قرار می‌گیرد و ۲۰٪ اختصاصی هر مشتری به‌صورت override در سایت خودش. هیچ clone کردن کدی لازم نیست.
- **فاز بعد:** رابط گرافیکی سازنده ایجنت (فرم ساده) و پیش‌نمایش رفتار ایجنت با «سناریوی تست».
- **چیزی که هرگز قابل پیکربندی نیست:** ساخت ابزار جدید از طریق رابط کاربری یا توسط هوش مصنوعی. ابزار جدید **کد** است، بازبینی می‌شود و از راه `hooks` ثبت می‌شود.

**نقاط توسعه برای اپ‌های دیگر** (مثل Packها، `karyar_iran` یا اپ اختصاصی یک مشتری):

<pre class="ltr">
# hooks.py of any app
karyar_tools            = ["karyar_iran.tools.validators", "karyar_iran.tools.moadian"]
karyar_step_types       = ["my_app.steps.custom_step"]
karyar_channel_adapters = ["karyar_iran.channels.bale"]
karyar_ai_providers     = ["my_app.ai.local_vllm"]
karyar_report_providers = ["my_app.reports"]
</pre>

# ۷. معماری موتور گردش‌کار

## ۷.۱ چرا موتور جداگانه؟ (مقایسه گزینه‌ها)

| گزینه | مزیت | عیب | نتیجه |
|---|---|---|---|
| **Frappe Workflow** | آماده، ادغام کامل با سند و Role | فقط یک DocType. بدون گام ایجنت، تایمر، موازی یا زیرفرایند | فقط برای «وضعیت سند» در Desk، و اختیاری |
| **n8n** | سریع و بصری | منطق تراکنشی و وضعیت، خارج از ERP و پراکنده؛ مجوز ضعیف | ❌ برای هسته؛ ✅ برای یکپارچه‌سازی |
| **Temporal** | اجرای پایدار حرفه‌ای، تایمر و retry | زیرساخت سنگین (سرور، پایگاه داده جدا)؛ تیم باید یاد بگیرد؛ چندمستأجری دردسر دارد | گزینه **بعدی** اگر بار و پیچیدگی رشد کرد |
| **Camunda / BPMN** | استاندارد | سنگین، Java، برای فرایندهای گفت‌وگومحور زیادی رسمی | ❌ |
| **موتور سبک در Frappe** ✅ | هم‌تراکنش با ERPNext، همان مجوز و چندمستأجری، بدون زیرساخت جدید | باید خودمان بسازیم و با دقت محدودش کنیم | **پیشنهاد فاز ۱** |

**تصمیم پیشنهادی:** یک **ماشین حالت پایدار** داخل اپ `karyar` ساخته شود. «تعریف» از «اجراکننده» جدا باشد تا در آینده اگر لازم شد، اجراکننده با Temporal عوض شود و تعریف‌ها دست نخورند.

## ۷.۲ مدل داده

| DocType | توضیح |
|---|---|
| `Karyar Workflow Definition` | نام، نسخه، وضعیت (draft/active/retired)، trigger و `graph` به‌صورت JSON اعتبارسنجی‌شده |
| `Karyar Workflow Run` (پرونده) | تعریف و نسخه آن (هر پرونده به نسخه‌ای که با آن شروع شده **سنجاق** است)، وضعیت، `context` ساخت‌یافته، اسناد ERPNext مرتبط، `correlation_id` |
| `Karyar Step Run` | هر اجرای یک گام: نوع، ورودی و خروجی، مجری، تلاش‌ها، خطا، زمان‌ها |
| `Karyar Human Task` | کار انسانی (بخش ۸) |
| `Karyar Timer` | زمان‌های سررسید برای timeout و escalation. یک job در scheduler هر دقیقه آن‌ها را بررسی می‌کند |

## ۷.۳ انواع گام (قابل گسترش با `karyar_step_types`)

| نوع گام | کاربرد | فاز |
|---|---|---|
| `trigger` | شروع با رویداد، پیام، زمان‌بندی یا دستی | ۱ |
| `agent_task` | گام ایجنت با هدف، نقش کاری، توانمندی‌های مجاز و Schema خروجی | ۱ |
| `human_task` | تعامل یا تأیید انسانی (بخش ۸) | ۱ |
| `condition` | شاخه شرطی با عبارت امن روی `context`، **نه** کد آزاد | ۱ |
| `erp_action` | اقدام از پیش تعریف‌شده روی ERPNext از طریق ابزار، مثل «ثبت نهایی Sales Order» | ۱ |
| `wait_event` | انتظار برای رویداد مشخص، مثل تأیید پرداخت | ۱ |
| `validate` / `transform` | اعتبارسنجی Schema و قواعد، و نگاشت داده | ۱ |
| `notify` | اعلان به انسان یا مشتری | ۱ |
| `end` | پایان (موفق، ردشده یا لغوشده) | ۱ |
| `timer` / `escalate` | مهلت و ارجاع به سطح بالاتر | ۲ |
| `integration_call` | فراخوانی n8n یا سرویس بیرونی با callback | ۲ |
| `parallel` / `join` | شاخه‌های موازی (مثلاً تأیید ۲ از ۳) | ۲ تا ۳ |
| `sub_workflow` | فراخوانی گردش‌کار دیگر | ۳ |
| `rpa_job` | کار RPA | ۳ به بعد |

## ۷.۴ معناشناسی اجرا

- **پیشروی رویدادمحور:** هر رویداد یا تکمیل گام، یک job به نام «پیشروی پرونده» (`advance_run`) را با **قفل پرونده** اجرا می‌کند. قفل با `SELECT … FOR UPDATE` روی ردیف پرونده یا قفل Redis گرفته می‌شود تا دو worker همزمان یک پرونده را جلو نبرند.
- **Idempotency:** هر گام یک `idempotency_key` دارد. اقدام‌های ERPNext با کلید یکتا ثبت می‌شوند تا تکرار رویداد به ایجاد سند تکراری منجر نشود.
- **Retry:** هر نوع گام سیاست retry خودش را دارد. ایجنت و API بیرونی retry می‌شوند، ولی `erp_action` فقط اگر idempotent باشد. بعد از اتمام تلاش‌ها، پرونده به حالت `needs_attention` می‌رود و یک Human Task «رفع خطا» ساخته می‌شود.
- **نسخه‌داری:** پرونده‌های در جریان با همان نسخه تعریفی که شروع شده‌اند ادامه می‌دهند. فعال کردن نسخه جدید فقط روی پرونده‌های جدید اثر دارد.
- **عبارت‌های شرط:** با یک زبان عبارت امن و محدود نوشته می‌شوند، مثلاً `safe_eval` در Frappe یا JSONLogic. **هرگز** کد Python آزاد در تعریف گردش‌کار نوشته نمی‌شود.

# ۸. معماری دخالت انسان (Human-in-the-Loop)

## ۸.۱ HITL یک گام است، نه یک قانون

هیچ ایجنتی ذاتاً «نیازمند تأیید» نیست. HITL از دو جا وارد پرونده می‌شود:

1. **گام `human_task` در گراف گردش‌کار:** صریح و طراحی‌شده.
2. **سیاست ریسک (Policy Gate):** وقتی ایجنت اقدامی پیشنهاد می‌کند که سیاست Tenant برایش تأیید لازم می‌داند.

## ۸.۲ پیکربندی یک گام انسانی

| تنظیم | گزینه‌ها |
|---|---|
| مسئول انجام (Assignee) | کاربر مشخص، Frappe Role، نقش کاری (Responsibility)، عبارت پویا مثل «مدیر شعبه همین مشتری»، یا Assignment Rule در Frappe |
| تعداد و الگوی تأیید | ۱ نفر، N نفر، «M از N»، ترتیبی یا موازی |
| داده قابل نمایش | فهرست فیلدها، همراه با پوشاندن فیلدهای حساس برای این گام |
| داده قابل ویرایش | فهرست فیلدهای قابل اصلاح توسط انسان. بقیه فقط‌خواندنی‌اند |
| پیش‌شرط تکمیل | فیلدهای اجباری یا قواعد اعتبارسنجی قبل از امکان تأیید |
| اقدام‌های مجاز | تأیید، رد، بازگشت برای اصلاح، درخواست اطلاعات از مشتری، توقف، ارجاع |
| پس از ویرایش | ادامه، یا بازبینی مجدد توسط همان نفر یا نفر دیگر |
| پس از رد | شاخه رد: پایان، بازگشت به گام قبل یا مسیر جایگزین |
| سطح تأیید صریح | `none` (پیام عادی کافی است)، `confirm` (نمایش خلاصه نهایی و تأیید صریح)، `strong` (تأیید صریح به‌علاوه OTP یا رمز برای عملیات مالی) |
| مهلت | timeout، یادآوری، ارجاع (فاز ۲) |
| کانال | Karyar Workspace، تلگرام/بله کارمند، یا ایمیل (لینک) |

## ۸.۳ تعامل گفت‌وگومحور، با تصمیم‌های ساخت‌یافته

**مشکل:** تعامل آزاد («سن را ۳۸ کن»، «بفرست مرحله بعد») با ممیزی و امنیت در تعارض است. اگر مدل «باشه» را اشتباهاً «تأیید» تفسیر کند چه می‌شود؟

**راه‌حل:** گفت‌وگو **رابط** است، اما وضعیت و تصمیم **ساخت‌یافته** است:

- Human Task یک **payload ساخت‌یافته** با Schema دارد که شامل داده، فیلدهای الزامی و فیلدهای قابل ویرایش است.
- ایجنت در حالت «دستیار بازبینی» فقط این ابزارها را دارد:
  - `task.show`: نمایش داده
  - `task.patch(field, value)`: اصلاح یک فیلد، با اعتبارسنجی Schema و قواعد. هر اصلاح یک «نسخه» ثبت می‌کند که در آن مشخص است *انسان دستور داد و ایجنت اجرا کرد*.
  - `task.ask_customer(question)`: درخواست اطلاعات از مشتری. این یک زیرگام می‌سازد که به گفت‌وگوی مشتری با Ava وصل است.
  - ابزارهای **خواندنی** در محدوده مجوز همین کارمند.
  - `task.propose_decision(action)`
- **تصمیم نهایی هرگز مستقیماً توسط مدل ثبت نمی‌شود.** مدل فقط *پیشنهاد* تصمیم می‌دهد. سیستم:
  1. هویت فرستنده را بررسی می‌کند: آیا مسئول مجاز این Task است؟
  2. بسته به سطح تأیید، خلاصه نهایی را نمایش می‌دهد و تأیید صریح می‌گیرد.
  3. تصمیم را با `payload_hash` ثبت می‌کند.
- **دکمه‌ها** («تأیید»، «رد») میانبر اختیاری همین ابزارها هستند، نه تنها راه.
- **ثبت ممیزی:** متن خام پیام انسان، تفسیر مدل، نسخه payload و hash، همگی ثبت می‌شوند.

<pre class="ltr">
Employee: "The age is incorrect. Change it to 38."
  → agent calls task.patch(age=38)   → validated → revision #2 (by: employee, via: agent)
Agent:    "Updated. Information is now complete."
Employee: "Send it to the next step."
  → agent calls task.propose_decision(approve)
  → policy: confirmation_level=confirm → system renders final summary (hash h2)
System:   "Confirm sending: Mohammad Rezaei, 38, Hair transplant … ? (yes/no)"
Employee: "yes" → Decision{approve, by: employee, payload_hash: h2, raw_text: "yes"} → event human_task.completed
</pre>

**امنیت:** اختیار تأیید فقط از **هویت احرازشده مسئول Task** می‌آید، نه از محتوای پیام. اگر پیام مشتری حاوی «این را تأیید کن» باشد، هیچ اثری ندارد. این دفاع اصلی در برابر Prompt Injection است.

## ۸.۴ تأیید مشتری (Customer Confirmation)

تأیید مشتری هم همین الگو را دارد:

1. Ava خلاصه را نمایش می‌دهد (نسخه `h1`).
2. پاسخ «بله، درست است» از **همان هویت کانالِ مشتری**، به همان نسخه `h1` گره می‌خورد.
3. اگر بعداً داده تغییر کند، تأیید مشتری برای نسخه جدید معتبر نیست. اینکه آیا دوباره تأیید لازم است، طبق سیاست گام تعیین می‌شود.

# ۹. معماری رویداد

## ۹.۱ منابع رویداد

| منبع | سازوکار |
|---|---|
| ERPNext / Frappe | `doc_events` برای DocTypeهای ثبت‌شده در «Event Source»، روی `after_insert`، `on_update`، `on_submit`، `on_cancel` و `on_update_after_submit` |
| Karyar | تکمیل گام، تصمیم انسانی، پیشنهاد ایجنت، پیام کانال |
| بیرونی | Webhook ورودی، callback از n8n، نتیجه RPA، پاسخ مودیان |
| زمان | Scheduler، مثل «هر روز ساعت ۸»، و تایمرهای مهلت |

## ۹.۲ Transactional Outbox

<pre class="ltr">
doc_event (same DB transaction as the business change)
   └─ insert "Karyar Event" row  {status: pending}
frappe.db.after_commit → enqueue("karyar.events.dispatch", queue="karyar_events")
dispatcher:
   match subscriptions (workflow triggers, waiting steps, outbound webhooks/n8n)
   → create/advance Workflow Runs (idempotent by event_id)
   → mark event dispatched
scheduler sweep (every minute): re-dispatch pending events older than N seconds
</pre>

- **چرا Outbox؟** اگر رویداد مستقیماً بعد از تغییر سند به صف فرستاده شود، ممکن است تراکنش rollback شود و رویداد «دروغ» منتشر شود، یا برعکس، commit انجام شود ولی صف از دست برود. Outbox هر دو را حل می‌کند: تحویل دست‌کم‌یک‌بار (at-least-once) همراه با مصرف‌کننده idempotent.
- **پوشش Wildcard:** استفاده از `"*"` در `doc_events` روی همه DocTypeها پرهزینه است. فقط DocTypeهایی که در پیکربندی «Event Source» فعال شده‌اند (با کش) رویداد تولید می‌کنند.

## ۹.۳ پوشش استاندارد رویداد

<pre class="ltr">
{ "event_id": "uuid", "type": "erp.sales_order.submitted", "schema_version": 1,
  "tenant": "acme.karyar.ir", "occurred_at": "...",
  "subject": {"doctype": "Sales Order", "name": "SO-0042"},
  "actor": {"type": "human|agent|system|customer", "id": "..."},
  "correlation_id": "run/RUN-0007", "causation_id": "event/...",
  "payload": { minimal, non-sensitive fields } }
</pre>

payload عمداً حداقلی است. مصرف‌کننده برای جزئیات، داده را با ابزار و **با مجوز خودش** از ERPNext می‌خواند. این کار از نشت داده از طریق رویدادها جلوگیری می‌کند.

## ۹.۴ چرا Kafka یا RabbitMQ نه (فعلاً)؟

Redis Queue (RQ) و Outbox مبتنی بر MariaDB همین حالا در Frappe وجود دارند و برای بار یک Tenant کافی‌اند. وقتی به جریان رویداد **بین Tenantها** نیاز شد، مثلاً برای اندازه‌گیری مرکزی مصرف یا تحلیل، می‌شود یک **Relay** از Outbox به NATS، Redis Streams یا Kafka اضافه کرد. هیچ تولیدکننده‌ای لازم نیست تغییر کند.

# ۱۰. ارتباط ایجنت با ایجنت

**اصل:** ایجنت‌ها با هم «چت» نمی‌کنند. ارتباط فقط از سه راه است:

1. **تحویل از راه گردش‌کار (اصلی):**
   1. ایجنت A گام خود را با **خروجی ساخت‌یافته** (طبق Schema) تمام می‌کند و خروجی در `context` پرونده ذخیره می‌شود.
   2. رویداد `workflow.step.completed` منتشر می‌شود.
   3. گام بعدی به نقش کاری X تعلق دارد. جدول انتساب می‌گوید X با ایجنت B است.
   4. ایجنت B یک **دستور کار (Task Brief)** دریافت می‌کند که از context ساخته شده است، **نه** متن خام گفت‌وگوی A.
2. **تحویل از راه ERPNext:** ایجنت A سندی می‌سازد (مثلاً Lead)، رویداد ERPNext منتشر می‌شود و گردش‌کار دیگری که trigger آن «Lead ایجاد شد» است برای ایجنت B شروع می‌شود.
3. **مشاوره کنترل‌شده (آینده):** ابزار `consult(responsibility, question)` که فقط اطلاعات **فقط‌خواندنی** و در محدوده مجوز برمی‌گرداند و ثبت می‌شود. در فاز ۱ وجود ندارد.

**چرا Task Brief به‌جای متن گفت‌وگو؟**

- **جلوگیری از انتقال Prompt Injection:** متن مشتری ممکن است دستور مخرب داشته باشد.
- **کاهش هزینه token.**
- **کمینه‌سازی داده:** ایجنت حسابداری لازم نیست کل گفت‌وگوی پزشکی را ببیند.
- **قابلیت ممیزی:** مشخص است دقیقاً چه چیزی تحویل شده است.

# ۱۱. معماری مجوز

## ۱۱.۱ زنجیره مجوز

<pre class="ltr">
Tenant (site)         → which packs/capabilities/integrations are licensed & enabled
  └ Principal         → who is acting: Human User | Agent Service User | Channel Contact | Integration
     └ Agent          → which Responsibilities → Capabilities it may use
        └ Step scope  → which Capabilities & which records are allowed in THIS workflow step
           └ Frappe   → DocType perms, User Permissions, permlevel, has_permission hooks
              └ Output filter → field whitelist per tool + masking by data classification
Effective = Tenant ∩ Agent ∩ Step ∩ Principal(Frappe)    — evaluated in code, never in the prompt
</pre>

## ۱۱.۲ اصل «از طرفِ چه کسی؟» (On-behalf-of)

| موقعیت | هویت اجرایی | نتیجه |
|---|---|---|
| کارمند با ایجنت گفت‌وگو می‌کند | **کاربر همان کارمند** (`frappe.set_user`)، به‌علاوه فیلتر توانمندی ایجنت | ایجنت هرگز بیش از خود کارمند نمی‌بیند |
| گام خودکار گردش‌کار (بدون انسان) | کاربر سرویس ایجنت، با Frappe Roleهای مشتق از توانمندی‌ها، **محدود به رکوردهای همین پرونده** | دسترسی حداقلی |
| مشتری بیرونی در کانال | «Contact Principal» بدون دسترسی Desk. ابزارهای ویژه که فقط داده همان مخاطب را برمی‌گردانند | مشتری فقط داده خودش را می‌بیند |
| n8n یا سیستم بیرونی | کاربر یکپارچه‌سازی با توانمندی‌های تعریف‌شده و کلید API یا OAuth | محدود به Karyar API |

**مثال سؤال «سود کل شرکت چقدر است؟» از کارمند عادی:**

1. ایجنت ممکن است ابزار `report.profit_and_loss` را صدا بزند.
2. Tool Executor بررسی می‌کند: آیا این کارمند به گزارش «Profit and Loss Statement» دسترسی Frappe دارد؟ جواب منفی است.
3. نتیجه `PERMISSION_DENIED` برمی‌گردد و **هیچ داده‌ای به مدل نمی‌رسد**.
4. ایجنت فقط توضیح می‌دهد که دسترسی ندارد.

حتی اگر prompt دستکاری شود، داده‌ای برای نشت وجود ندارد.

## ۱۱.۳ قواعد پیاده‌سازی (قابل بررسی خودکار)

- ابزارها فقط از APIهای مجوزدار استفاده می‌کنند: `frappe.get_list`، `doc.check_permission`، `frappe.has_permission`، `query_report.run`.
- **استفاده از `frappe.get_all`، `ignore_permissions=True` و `frappe.db.sql` در ماژول ابزارها ممنوع است.** این قاعده با یک قانون semgrep در CI بررسی می‌شود (ERPNext خودش semgrep دارد). استثنا فقط با بازبینی امنیتی و توضیح مکتوب مجاز است.
- هر ابزار یک **Schema خروجی** دارد. فیلدهای خارج از Schema حذف می‌شوند. فیلدهای دارای permlevel بالاتر از سطح دسترسی principal پوشانده می‌شوند.
- **طبقه‌بندی داده** (فاز ۲): فیلدها با برچسب‌هایی مثل `pii`، `financial`، `medical` علامت‌گذاری می‌شوند. سیاست Tenant تعیین می‌کند کدام برچسب‌ها اجازه دارند به ارائه‌دهنده بیرونی AI فرستاده شوند (بخش ۱۹).

## ۱۱.۴ محدودیت Frappe که باید صریح گفته شود

- مدل مجوز Frappe **نقش‌محور (RBAC) به‌علاوه User Permission** است، نه ABAC کامل. منطق ظریف‌تر از راه hookهای `has_permission` و `permission_query_conditions` پیاده می‌شود.
- **گزارش‌های Query/Script** ممکن است فقط در سطح نقش کنترل شوند و User Permission را کامل اعمال نکنند. بنابراین ایجنت فقط به **فهرست گزینش‌شده گزارش‌ها** (Report Catalog) دسترسی دارد. هر گزارش قبل از ورود به فهرست، از نظر مجوز بازبینی می‌شود.

# ۱۲. معماری یکپارچگی با ERPNext

| موضوع | تصمیم |
|---|---|
| خواندن | از ابزارها، با `get_list` یا `get_doc` و مجوز principal |
| نوشتن | فقط از ابزارهای `draft-write` (سند در وضعیت `docstatus=0`) و `commit-write` (submit یا cancel) |
| منطق کسب‌وکار | **از کنترلرهای ERPNext استفاده می‌شود** (`doc.insert()`، `doc.submit()`، توابع `make_*` مثل `make_sales_invoice`)، نه نوشتن مستقیم جدول. همه اعتبارسنجی‌های ERPNext حفظ می‌شوند |
| فیلدهای اضافه | Custom Field و Property Setter به‌صورت fixtures در اپ Karyar یا Pack. **هرگز** تغییر JSON DocTypeهای ERPNext |
| رفتار اضافه | `doc_events` و `extend_doctype_class`. از `override_doctype_class` فقط در صورت اجبار و با مستندسازی استفاده می‌شود. Monkey-patch ممنوع است |
| وضعیت فرایند | در Karyar Run نگه داشته می‌شود. وضعیت سند (`docstatus` و `status`) در ERPNext است. Frappe Workflow روی DocTypeهایی که Karyar مدیریت می‌کند **فعال نمی‌شود** تا دو ماشین حالت روی یک سند نداشته باشیم (بخش ۳۳) |
| ارجاع | Karyar Run فقط **ارجاع** (`doctype`/`name`) به سند ERPNext نگه می‌دارد، نه کپی داده. تنها استثنا snapshot لازم برای ممیزی تأیید است |
| Idempotency | فیلد سفارشی `karyar_idempotency_key` روی DocTypeهایی که ابزارها می‌سازند، همراه با بررسی یکتایی |

**نمایه ابزار (Tool Contract):**

<pre class="ltr">
@karyar_tool(
  name="crm.lead.create_draft", capability="crm.lead.draft",
  side_effect="draft-write", risk="low",
  input_schema=LeadDraftIn, output_schema=LeadRef,
  required_perms=[("Lead", "create")], idempotent=True)
def create_lead_draft(inp: LeadDraftIn, ctx: ToolContext) -> LeadRef: ...
</pre>

# ۱۳. معماری Karyar API

- **یک لایه سرویس، سه نمایش.** منطق در «سرویس‌های توانمندی» نوشته می‌شود و از سه راه در دسترس قرار می‌گیرد:
  1. ابزارهای ایجنت
  2. REST: ‏`/api/method/karyar.api.v1.<resource>.<action>` یا مسیرهای v2 در Frappe
  3. (فاز بعد) MCP با کتابخانه `frappe-mcp` برای ایجنت‌های بیرونی
- **نسخه‌داری:** `v1` پایدار است. تغییرات فقط افزایشی‌اند و تغییر ناسازگار یعنی `v2`. DTOها با Schema صریح تعریف می‌شوند، **نه** ساختار خام DocType. به این ترتیب تغییر نام فیلد در ERPNext نسخه ۱۷ مصرف‌کننده‌ها را نمی‌شکند.
- **احراز هویت:**
  - کلید API/Secret برای سیستم‌ها (n8n)
  - OAuth2 برای اپ‌ها و ایجنت‌های بیرونی (Frappe خودش OAuth Provider است)
  - Session برای Workspace
- **محدودیت نرخ و اندازه** برای هر principal.
- **منابع اولیه v1 (حداقلی):** `conversations`، `messages` (ورودی کانال‌ها)، `runs` (شروع و وضعیت پرونده)، `tasks` (صندوق کار انسانی)، `events/callback` (callback از n8n و RPA)، و `tools/invoke` (فقط برای principalهای مجاز).
- **Webhook خروجی:** با سازوکار Webhook در Frappe و امضای HMAC، یا از دیسپچر رویداد Karyar.

# ۱۴. معماری گزارش‌گیری

| اولویت | منبع | سازوکار |
|---|---|---|
| ۱ | گزارش‌های استاندارد ERPNext | ورود به **Report Catalog** با شناسه، توضیح، Schema فیلترهای مجاز و ستون‌های مجاز. اجرا با `frappe.desk.query_report.run` با هویت principal |
| ۲ | گزارش‌های Karyar | تابع Python از پیش نوشته‌شده با Schema ورودی و خروجی که در کد بازبینی شده است. ترکیبی‌ها هم همین‌جا هستند، مثل «خلاصه مدیریتی = فروش + دریافتنی + موجودی بحرانی» |
| ۳ (بعد) | Frappe Insights | برای داشبوردهای انسانی، نه برای پرسش آزاد ایجنت |
| ❌ | SQL ساخته‌شده توسط هوش مصنوعی | ممنوع |

**نقش هوش مصنوعی در گزارش:**

1. فهم درخواست
2. انتخاب گزارش از فهرست مجاز
3. پر کردن پارامترها (با اعتبارسنجی Schema)
4. تفسیر و توضیح

**اعداد همیشه از نتیجه گزارش می‌آیند.** مدل محاسبه نمی‌کند. اگر نسبت یا جمع لازم است، گزارش Karyar آن را محاسبه می‌کند. پاسخ ایجنت **منبع** را هم نشان می‌دهد: نام گزارش، فیلترها و زمان اجرا.

**(فاز ۲) بررسی عدد:** یک بررسی خودکار کنترل می‌کند هر عدد در متن پاسخ، در داده نتیجه وجود داشته باشد.

**عملکرد:** گزارش‌های سنگین به‌صورت **Prepared Report** اجرا می‌شوند. در صورت نیاز از **Read Replica** پایگاه داده استفاده می‌شود که Frappe پشتیبانی می‌کند.

# ۱۵. معماری مدل هوش مصنوعی

<pre class="ltr">
Agent Runtime ──► AIGateway.complete(ModelRequest) ──► ProviderAdapter ──► Provider/Model
                     │  alias → (provider, model, params, fallback chain)  [per tenant]
                     │  structured-output enforcement (JSON Schema validate + 1 retry)
                     │  redaction hook (data classification policy)
                     │  timeouts / retries / circuit breaker
                     └► UsageSink.record(UsageRecord)   ← extension point (section 16)
</pre>

- **آداپتورها:**
  - **«سازگار با OpenAI»:** بیشتر ارائه‌دهندگان، درگاه‌های داخلی و مدل‌های خودمیزبان (vLLM و Ollama) را پوشش می‌دهد.
  - آداپتورهای اختصاصی، مثلاً Anthropic و Google، در صورت نیاز.
  - همه آداپتورها از راه hook ‏`karyar_ai_providers` قابل افزودن‌اند.
- **پیکربندی مدل:**
  - `Model Profile` در هر Tenant، نام مستعار (`fast`، `smart`، `local-private`) را به مدل واقعی نگاشت می‌کند.
  - ایجنت فقط نام مستعار را می‌شناسد. تعویض ارائه‌دهنده یعنی تغییر یک رکورد.
- **دستورالعمل‌ها:** `Prompt Template` نسخه‌دار. هر نوبت ایجنت نسخه دستورالعمل را ثبت می‌کند تا بشود رفتار را بازتولید کرد.
- **کلیدها:** «کلید سکو» (Karyar هزینه AI را می‌فروشد) یا «کلید مشتری» (BYOK). هر دو در فیلد Password سایت ذخیره می‌شوند. **تصمیم تجاری لازم است.**
- **LiteLLM Proxy:** گزینه فاز ۲ یا ۳، وقتی بودجه‌بندی، کلیدهای مجازی و مسیریابی مرکزی لازم شد. چون همه ترافیک از `AIGateway` می‌گذرد، جایگزینی بدون تغییر ایجنت‌هاست.
- **⚠️ واقعیت ایران:**
  - دسترسی به برخی ارائه‌دهندگان بین‌المللی از ایران به دلیل تحریم و سیاست ارائه‌دهندگان محدود است.
  - فرستادن داده مشتری (به‌ویژه داده پزشکی یا مالی) به خارج پیامد حقوقی دارد.
  - معماری باید از روز اول **مدل خودمیزبان یا داخلی** را به‌عنوان گزینه درجه‌یک پشتیبانی کند.
  - کیفیت زبان فارسی هر مدل باید با **مجموعه ارزیابی فارسی** سنجیده شود.

# ۱۶. نقطه توسعه هزینه و token (AI Cost / Token Tracking)

**الان ساخته نمی‌شود**؛ فقط جای تمیز آن گذاشته می‌شود:

**`UsageRecord` در هر فراخوانی AI Gateway:**

<pre class="ltr">
request_id, tenant, agent, responsibility, workflow, run, step, conversation, api_principal,
provider, model, model_alias, input_tokens, output_tokens, cached_tokens, total_tokens,
latency_ms, status, price_version, estimated_cost, currency, actual_cost (nullable), timestamp
</pre>

- **`UsageSink`:** یک رابط است. در فاز ۱ دو پیاده‌سازی دارد:
  1. DocType ‏`Karyar AI Usage` در همان سایت
  2. یک خط لاگ JSON ساخت‌یافته

  بعداً پیاده‌سازی «ارسال به سرویس اندازه‌گیری مرکزی» در Control Plane اضافه می‌شود، که برای جمع بین Tenantها لازم است چون هر Tenant پایگاه داده جدا دارد.
- **قیمت‌ها:** جدول `Model Price` نسخه‌دار با تاریخ اعتبار. **در کد ثابت نیستند.** `actual_cost` بعداً با صورت‌حساب ارائه‌دهنده تطبیق داده می‌شود.
- **حداقل لازم از روز اول:** یک **سقف سخت** مصرف روزانه برای هر Tenant و هر ایجنت (قطع‌کننده مدار). یک حلقه معیوب ایجنت می‌تواند در چند ساعت هزینه سنگین بسازد.

# ۱۷. معماری n8n

| کجا؟ | چه چیزی؟ |
|---|---|
| **ERPNext** | داده و قواعد کسب‌وکار: سند، حسابداری، انبار، اعتبارسنجی‌ها، وضعیت سند، مجوز پایه |
| **Karyar** | ایجنت‌ها، گردش‌کارهای کسب‌وکار، HITL، رویداد، سیاست، ابزار، API، ممیزی، AI Gateway، و **هر تصمیم یا تراکنش حساس** |
| **n8n** | اتصال به سرویس‌های ثالث (پیامک، ایمیل مارکتینگ، Google Sheets، CRMهای دیگر)، اعلان‌ها، همگام‌سازی‌های زمان‌بندی‌شده، خروجی داده، چسب‌های کم‌ریسک |

**قواعد سخت:**

1. n8n **هرگز** مستقیم در ERPNext نمی‌نویسد. فقط از **Karyar API** و با کاربر یکپارچه‌سازی و توانمندی‌های محدود کار می‌کند.
2. n8n وضعیت کسب‌وکار نگه نمی‌دارد. اگر چیزی باید یادش بماند، جای آن در Karyar یا ERPNext است.
3. Karyar از گام `integration_call` با یک callback امضاشده، n8n را صدا می‌زند و پرونده تا رسیدن callback یا timeout منتظر می‌ماند.
4. **callbackهای حساس (مثل تأیید پرداخت درگاه بانکی) مستقیم به Karyar API می‌آیند**، نه از مسیر n8n.
5. workflowهای n8n به‌صورت JSON در مخزن `karyar-deploy` نسخه‌داری می‌شوند.

**چندمستأجری:** n8n نسخه Community چندمستأجری واقعی ندارد. پیشنهاد:

- یک n8n مشترک برای جریان‌های سکو که پارامتر Tenant می‌گیرند و اعتبارنامه‌ها را از Karyar می‌گیرند.
- n8n اختصاصی فقط برای مشتریانی که جریان سفارشی دارند.

> **⚠️ مجوز:** n8n مجوز Sustainable Use دارد. اگر مشتریان شما با اعتبارنامه‌های خودشان از n8n (حتی غیرمستقیم) استفاده کنند، احتمالاً مجوز Embed لازم است. قبل از تعهد تجاری بررسی حقوقی شود. جایگزین‌های متن‌باز: Activepieces (MIT در هسته) یا Node-RED (Apache-2.0).

# ۱۸. معماری RPA

<pre class="ltr">
Workflow step "rpa_job" ──► Karyar RPA Job (doctype: queued)
                                  ▲            │ pull (HTTPS, Karyar API, worker token)
                                  │            ▼
                          result/screenshots   RPA Worker (separate container: Python + Playwright)
                                  │            │  needs CAPTCHA/OTP?
                                  │            ▼
                                  └── Human Task "provide OTP" (conversational) ──► resume job
</pre>

| اصل | توضیح |
|---|---|
| **اول API** | RPA فقط برای سیستم‌هایی است که API ندارند، مثل پورتال بانک، تأمین اجتماعی، سامانه‌های دولتی و نرم‌افزارهای قدیمی |
| **هرگز روی رابط ERPNext** | برای کار داخل ERPNext همیشه ابزار و API استفاده می‌شود |
| **سرویس جدا** | Worker در کانتینر ایزوله و بدون دسترسی به پایگاه داده اجرا می‌شود. فقط با Karyar API ارتباط دارد |
| **مدل Pull** | Worker خودش کار را برمی‌دارد. لازم نیست ورودی شبکه به Worker باز باشد |
| **اسرار** | اعتبارنامه پورتال در فیلد Password سایت Tenant ذخیره می‌شود و **فقط برای همان Job** و برای مدت کوتاه به Worker داده می‌شود |
| **انسان در حلقه** | کپچا، OTP و تأیید دستی تبدیل به Human Task می‌شوند. کارمند به‌صورت گفت‌وگویی کد را وارد می‌کند و Job ادامه پیدا می‌کند |
| **شواهد** | اسکرین‌شات و لاگ هر گام به‌صورت فایل خصوصی پیوست Job می‌شود |
| **زمان** | فاز ۳ به بعد. **الان لازم نیست** |

# ۱۹. معماری چندمستأجری (Multi-Tenant)

## ۱۹.۱ گزینه‌ها

| مدل | جداسازی | هزینه و عملیات | ارتقا | مناسب برای |
|---|---|---|---|---|
| **A. یک Site، چند Company در ERPNext** | ضعیف. با یک خطای User Permission، داده نشت می‌کند. داده پایه مثل Item و Customer مشترک است. سفارشی‌سازی‌ها مشترک‌اند | ارزان‌ترین | با هم | فقط **یک گروه تجاری** با چند شخصیت حقوقی. **❌ برای کسب‌وکارهای مستقل** |
| **B. یک Site برای هر Tenant روی Bench مشترک** ✅ | پایگاه داده جدا با کاربر جدا، پوشه فایل جدا، کلید رمزنگاری جدا. پروسه‌ها، Workerها و Redis مشترک‌اند (کلیدها با نام سایت جدا می‌شوند) | متوسط. هر Bench می‌تواند ده‌ها سایت داشته باشد | همه سایت‌های یک Bench با هم ارتقا می‌یابند | **پیش‌فرض Karyar**. همان مدلی که Frappe Cloud استفاده می‌کند |
| **C. یک Bench یا Stack برای هر Tenant** | قوی‌ترین: کانتینر، Worker و حتی سرور پایگاه داده جدا | گران و پرعملیات | مستقل | مشتریان بزرگ، حساس (پزشکی) یا نیازمند نسخه خاص |
| **D. Row-Level Security در پایگاه داده** | — | — | — | در Frappe و ERPNext **قابل استفاده نیست** |

## ۱۹.۲ پیشنهاد

- **پیش‌فرض: مدل B** با «گروه‌های Bench» (مثلاً `bench-standard-01` و `bench-standard-02`).
- **برای مشتریان حساس: مدل C.**
- **مدل A برای کسب‌وکارهای جدا هرگز استفاده نشود.**

**نکات صریح:**

- «جدول جدا» یا حتی «پایگاه داده جدا» به‌تنهایی کافی نیست. در مدل B، کد همه Tenantها در یک پروسه اجرا می‌شود. پس:
  - (۱) هیچ کد مخصوص یک مشتری بدون بازبینی روی Bench مشترک نصب نمی‌شود.
  - (۲) **Server Script** و دسترسی Administrator، bench و console به مشتری داده نمی‌شود.
  - (۳) Workerها و کش Redis همیشه با `frappe.local.site` کار می‌کنند.
- **همه اجزای بیرون از Frappe باید Tenant-aware باشند.** n8n، RPA، AI Gateway (در صورت جدا شدن) و Channel Gateway (در صورت جدا شدن) هر درخواست را با شناسه سایت و اعتبارنامه همان سایت انجام می‌دهند.
- **محدودیت مقیاس مدل B:** Scheduler روی همه سایت‌ها می‌چرخد و Workerها مشترک‌اند، پس هر Bench سقفی دارد. مشتری پرمصرف به Bench دیگری منتقل می‌شود (پشتیبان‌گیری و بازیابی سایت).
- **Control Plane (بعد):** شامل ثبت Tenantها، Bench هر Tenant، Packها، پلن، سقف مصرف و جمع مصرف. می‌تواند یک سایت Frappe جداگانه با اپ `karyar_platform` باشد. Frappe Press (AGPL) را می‌توان بررسی کرد، ولی برای شروع **سنگین** است. اول با اسکریپت راه‌اندازی شود: `new-site`، سپس `install-app`، سپس `apply-pack`.

# ۲۰. معماری امنیت

| حوزه | تصمیم |
|---|---|
| **جداسازی Tenant** | مدل B یا C (بخش ۱۹). هر سایت کلید رمزنگاری خودش را دارد (`encryption_key`) |
| **احراز هویت انسان** | ورود Frappe همراه با 2FA (موجود در Frappe) برای نقش‌های مالی و مدیریتی. برای کانال‌ها: اتصال حساب کانال به کاربر از راه کد یک‌بارمصرف |
| **احراز هویت مشتری** | هویت کانال (شناسه تلگرام یا شماره واتس‌اپ). برای دسترسی به داده حساس، تأیید با OTP پیامکی |
| **احراز هویت ماشین** | API Key/Secret یا OAuth2 برای هر سیستم بیرونی، با چرخش کلید |
| **مجوز** | بخش ۱۱: زنجیره کامل در کد و ممنوعیت `ignore_permissions` در ابزارها |
| **اسرار** | فیلدهای Password در Frappe (رمزنگاری با کلید سایت). کلیدهای سکو در متغیر محیطی یا Vault. **هرگز** در prompt، لاگ یا Pack |
| **ورودی‌های بیرونی** | تأیید امضای HMAC روی همه Webhookها، جلوگیری از replay با timestamp و nonce، محدودیت نرخ |
| **Prompt Injection** | محتوای مشتری، فایل‌ها و صفحات وب «داده نامطمئن» برچسب می‌خورند. اختیار اقدام فقط از principal و سیاست می‌آید. ابزارهای پرریسک بدون HITL در دسترس گفت‌وگوی مشتری نیستند |
| **امنیت تأیید** | گره خوردن تأیید به `payload_hash`، احراز هویت تأییدکننده، و تأیید قوی برای عملیات مالی (بخش ۸) |
| **کمینه‌سازی داده به AI** | Schema خروجی ابزار، طبقه‌بندی داده، و امکان «مدل خودمیزبان» برای داده حساس |
| **RPA** | ایزوله، بدون دسترسی به پایگاه داده، اعتبارنامه کوتاه‌مدت |
| **ممیزی** | Audit Log فقط‌افزودنی (بخش ۲۱) |
| **زنجیره تأمین** | پین کردن نسخه‌ها، mirror داخلی PyPI و npm (به دلیل تحریم)، اسکن وابستگی‌ها |

# ۲۱. معماری ممیزی و مشاهده‌پذیری

## ۲۱.۱ سه لایه ردیابی

| لایه | محتوا | ابزار |
|---|---|---|
| **تغییرات داده** | چه فیلدی در کدام سند تغییر کرد | `Version` در Frappe (track changes) — موجود |
| **ممیزی معنایی** | چه کسی (انسان، ایجنت، سیستم یا مشتری)، از طرف چه کسی، کدام توانمندی یا ابزار، ورودی (خلاصه یا hash)، پیشنهاد در برابر اجرای واقعی، تأییدکننده، سند اثرپذیرفته، پرونده و گام، نتیجه و خطا | DocType جدید `Karyar Audit Log` |
| **ردیابی فنی** | زمان‌ها، retryها، مصرف AI، خطاها | Step Run، AI Usage، Error Log و RQ Job، با `correlation_id` مشترک |

- **فقط‌افزودنی:** هیچ نقشی، حتی System Manager، اجازه ویرایش یا حذف Audit Log را ندارد. این در کنترلر اعمال می‌شود.
- **محدودیت صریح:** مدیر پایگاه داده همچنان می‌تواند با SQL تغییر دهد. برای عملیات حساس در فاز بعد، **زنجیره hash** (هر ردیف hash ردیف قبل را دارد) برای تشخیص دستکاری اضافه می‌شود.
- **ذخیره prompt و پاسخ مدل:** با سطح‌های قابل تنظیم `off`، `summary` و `full`، به‌علاوه سیاست نگهداری. این توازنی بین عیب‌یابی و حریم خصوصی است.

## ۲۱.۲ مشاهده‌پذیری

**الان:**

- `correlation_id` در همه لاگ‌ها
- لاگ ساخت‌یافته JSON
- یک داشبورد ساده در Workspace: پرونده‌های گیرکرده، کارهای انسانی معوق، خطاها، صف‌ها، و مصرف AI امروز

**بعد:**

- OpenTelemetry برای trace بین Frappe، Worker، n8n و RPA
- Prometheus و Grafana برای متریک‌ها
- Sentry یا GlitchTip برای خطاها
- هشدارها

# ۲۲. جریان داده

<pre class="ltr">
[Customer msg] → Channel Gateway → Conversation(+Message) → event channel.message.received
   → Workflow Engine: find/create Run → agent_task(Ava, responsibility=intake)
      → Ava: tools (validate_phone, validate_national_id, …) → Run.context.intake (draft, NOT ERP yet)
      → customer confirmation (hash h1) → human_task(review) → decision approve(h2)
      → erp_action: crm.lead.create (idempotent) → ERPNext Lead (SOURCE OF TRUTH)
         → doc_event → Karyar Event erp.lead.created (outbox)
            → trigger workflow "sales_followup" → agent_task(responsibility=sales → Hanna)
</pre>

**مرز داده:**

- Karyar فقط **وضعیت فرایند** را نگه می‌دارد: گفت‌وگو، context پرونده، پیش‌نویس‌های قبل از تأیید، تصمیم‌ها و ممیزی.
- بعد از تأیید، **حقیقت کسب‌وکار** وارد ERPNext می‌شود و Karyar فقط به آن ارجاع می‌دهد.
- اگر کسی Lead را مستقیم در ERPNext تغییر دهد، ERPNext معتبر است. ایجنت‌ها همیشه داده تازه را از ERPNext می‌خوانند، **نه از context کهنه پرونده**.

# ۲۳. ساختار مخزن‌ها

## ۲۳.۱ تعارض فعلی

مخزن `karyar-mainERP` خودِ ERPNext است و روی `develop` قرار دارد. اگر Karyar داخل آن نوشته شود:

- (۱) عملاً هسته تغییر کرده است.
- (۲) هر ادغام با upstream تعارض می‌دهد.
- (۳) bench اپ‌ها را **یک مخزن برای هر اپ** نصب می‌کند.

## ۲۳.۲ پیشنهاد: چند مخزن (Polyrepo) و یک مخزن استقرار

| مخزن | محتوا | فاز |
|---|---|---|
| `karyar-mainERP` (همین) | fork ERPNext، **با پایه `version-16`** (شاخه `karyar/version-16`). فقط وصله‌های اجتناب‌ناپذیر، همراه با فهرست `PATCHES.md` و تلاش برای ارسال به upstream | ۰ |
| `karyar` | اپ اصلی سکو (ساختار پایین) | ۱ |
| `karyar_iran` | بومی‌سازی ایران و مودیان | ۲ |
| `karyar_hr_ir` | حقوق، بیمه و مالیات حقوق ایران (روی HRMS) | ۳ به بعد |
| `karyar-deploy` | `apps.json` با نسخه‌های پین‌شده، Dockerfile و compose (بعدها Helm)، اسکریپت راه‌اندازی Tenant، workflowهای n8n، Packها، runbookها | ۰ تا ۱ |
| `karyar-rpa-worker` | سرویس RPA | ۳ به بعد |
| `karyar-packs` (اختیاری) | اگر Packها زیاد شدند، به‌صورت اپ یا مخزن جدا | بعد |

**چرا اپ `karyar` یکی است، نه چند اپ از روز اول؟**

- جدا کردن اپ‌ها هزینه نسخه‌داری و وابستگی دارد.
- مرز را با **ماژول‌ها و پکیج‌های Python** و قاعده import می‌کشیم.
- **ریسک:** انتقال DocType بین اپ‌ها در آینده دردسر دارد. پس مرزهای ماژول باید از اول تمیز باشند.

<pre class="ltr">
karyar/                              (Frappe app repo)
├── pyproject.toml
└── karyar/
    ├── hooks.py                     # doc_events, scheduler, extension-point hook names, fixtures
    ├── modules.txt                  # Karyar Agents, Karyar Workflow, Karyar Conversations,
    │                                # Karyar Events, Karyar Integrations, Karyar Audit, Karyar AI
    ├── core/                        # framework-light domain logic (unit-testable)
    │   ├── workflow/  (graph model, executor, step types registry)
    │   ├── policy/    (permission chain, risk gates)
    │   ├── tools/     (registry, executor, contracts)
    │   ├── agents/    (runtime loop, context builder)
    │   ├── ai/        (gateway, provider adapters, usage sink)
    │   └── events/    (envelope, outbox, dispatcher)
    ├── karyar_agents/doctype/       # Karyar Agent, Responsibility, Capability, Role Assignment,
    │                                # Prompt Template, Model Profile
    ├── karyar_workflow/doctype/     # Workflow Definition, Workflow Run, Step Run, Human Task, Timer
    ├── karyar_conversations/doctype/# Conversation, Message, Channel Account, Contact Identity
    ├── karyar_events/doctype/       # Karyar Event, Event Source, Event Subscription
    ├── karyar_integrations/doctype/ # Integration Endpoint, RPA Job
    ├── karyar_audit/doctype/        # Audit Log, AI Usage, Model Price
    ├── tools/                       # built-in tool packs: crm, selling, accounts, stock, reports, task
    ├── channels/                    # adapters: web widget, telegram, (bale via karyar_iran)
    ├── api/v1/                      # stable REST facade
    ├── mcp.py                       # (later) frappe-mcp exposure of the same tools
    ├── packs/                       # base packs (fixtures/JSON)
    ├── workspace/                   # Karyar Workspace SPA (React + frappe-react-sdk, like erpnext/banking)
    ├── commands/                    # bench karyar apply-pack, etc.
    ├── patches/ , patches.txt
    ├── locale/fa.po
    └── tests/                       # unit + integration (against ERPNext) + tool contract tests
</pre>

**الگوی موجود:** ERPNext خودش یک SPA با React برای بانکداری دارد که با `frappe-react-sdk` در `/banking` سرو می‌شود. Karyar Workspace می‌تواند دقیقاً همین الگو را دنبال کند، یعنی بدون سرور جداگانه فرانت‌اند. گزینه دیگر **frappe-ui** (Vue) است که اپ‌های CRM و Helpdesk استفاده می‌کنند. **(تصمیم لازم است.)**

# ۲۴. معماری استقرار

**فاز ۱ (یک سرور یا Docker Compose، بر پایه `frappe_docker`):**

<pre class="ltr">
nginx (TLS, per-tenant hostnames)
 ├─ frappe-web (gunicorn)        ├─ socketio (realtime)
 ├─ workers: short, default, long, karyar_events, karyar_agent (concurrency-limited)
 ├─ scheduler
 ├─ MariaDB (one DB per site)    ├─ redis-cache   ├─ redis-queue
 ├─ n8n (+ its own Postgres)     [phase 2]
 └─ backups → S3-compatible object storage (encrypted)
</pre>

**فاز ۲ و بعد:**

- چند Bench، Kubernetes (Helm chart رسمی Frappe وجود دارد)
- MariaDB با replica
- RPA Workerها
- (در صورت نیاز) سرویس جدای Channel Gateway و AI Gateway (LiteLLM)

**⚠️ تصمیم محل میزبانی** (بزرگ‌ترین تصمیم زیرساختی):

| گزینه | مزیت | عیب |
|---|---|---|
| داخل ایران | اقامت داده، دسترسی مشتریان، مودیان و بانک‌ها | دسترسی به ارائه‌دهندگان AI بین‌المللی و APIهای تلگرام و Meta محدود است |
| خارج از ایران | دسترسی به AI و کانال‌های بین‌المللی | ریسک حقوقی و اقامت داده، و دسترسی سامانه‌های داخلی |
| **ترکیبی** (پیشنهاد اولیه) | ERP و داده در ایران. یک «Egress Gateway» کم‌حجم بیرون، فقط برای AI و کانال‌های بین‌المللی، همراه با کمینه‌سازی داده | پیچیدگی بیشتر و نیاز به بررسی حقوقی |

# ۲۵. راهبرد مقیاس‌پذیری

- **گلوگاه اصلی** تأخیر و نرخ API مدل است، نه پایگاه داده. پس:
  - صف جدا برای ایجنت‌ها
  - **سقف همزمانی برای هر Tenant** (عدالت بین Tenantها)
  - پخش پاسخ (streaming)
  - کش پاسخ ابزارهای خواندنی پرتکرار برای مدت کوتاه
- **افقی:** Worker و gunicorn بیشتر. Workerهای `karyar_agent` مستقل از Workerهای ERPNext مقیاس می‌گیرند.
- **عمودی پایگاه داده:** MariaDB قوی‌تر، replica برای گزارش‌ها، و Prepared Report.
- **بین Tenantها:** انتقال Tenant پرمصرف به Bench جدید و سقف تعداد سایت در هر Bench.
- **رویداد:** Outbox و RQ تا هزاران رویداد در دقیقه برای هر Bench کافی است. بعد از آن Relay به Streams یا NATS.
- **قبل از استخراج سرویس:** ابتدا اندازه‌گیری و پروفایل‌گیری. سرویس جدا فقط وقتی ساخته می‌شود که داده نشان دهد لازم است.

# ۲۶. راهبرد ارتقا

1. **ERPNext و Frappe دست‌نخورده یا تقریباً دست‌نخورده**، با نسخه‌های پین‌شده در `apps.json`. ارتقاهای minor نسخه v16 به‌صورت ماهانه انجام می‌شوند: اول staging، بعد production.
2. **آزمون قرارداد ابزارها (Tool Contract Tests):** هر ابزار روی نسخه هدف ERPNext تست می‌شود. شکست در این تست‌ها یعنی ارتقا متوقف می‌شود.
3. **Karyar API v1** مصرف‌کننده‌ها را از تغییر نام‌ها در ERPNext محافظت می‌کند.
4. **نسخه‌داری** تعریف گردش‌کار، Pack و دستورالعمل‌ها. پرونده‌های در جریان روی نسخه خود باقی می‌مانند.
5. **ارتقای اصلی (v16 به v17):** حدود ۶ ماه بعد از انتشار پایدار v17، با شاخه آزمایشی و اجرای کل تست‌ها. احتمالاً همزمان فرصت ارزیابی PostgreSQL هم پیش می‌آید.
6. **بدون Monkey-patch.** درس از اپ‌های جامعه ایرانی: وصله‌های زمان اجرا بزرگ‌ترین مانع ارتقا هستند.

# ۲۷. معماری بومی‌سازی ایران

**اپ `karyar_iran`** (بدون هیچ وابستگی از سمت هسته `karyar` به آن):

| بخش | طراحی | فاز |
|---|---|---|
| زبان و RTL | ترجمه Frappe (۹۶٪) و ERPNext (۸۱٪) موجود است. تکمیل ترجمه‌های Karyar در `fa.po` | ۱ |
| نرمال‌سازی متن | ارقام فارسی و عربی به لاتین، «ي/ك» به «ی/ک»، نیم‌فاصله، شماره موبایل (`+98`/`09`). به‌صورت **ابزار قطعی** که ایجنت‌ها و جست‌وجو استفاده می‌کنند | ۱ |
| اعتبارسنج‌ها | کد ملی، شناسه ملی، کد اقتصادی، شبا (چک‌سام)، کد پستی. به‌صورت ابزار برای اعتبارسنجی Ava | ۱ |
| تاریخ جلالی | ذخیره میلادی و نمایش جلالی (datepicker، formatter، فیلتر Jinja). **تفسیر تاریخ‌های زبان طبیعی** مثل «پنجشنبه هفته بعد» یا «۱۵ مهر» با یک ابزار قطعی `parse_persian_date`، نه حدس مدل. شماره‌گذاری اسناد با `naming_series_variables` | ۱ تا ۲ |
| کانال‌های ایرانی | آداپتور بله (API شبیه تلگرام)، و در صورت نیاز ایتا یا روبیکا، از راه `karyar_channel_adapters` | ۲ |
| حسابداری | چارت حساب ایرانی با `create_charts(custom_chart=…)`. سطح «تفصیلی» با Party و Accounting Dimension. ریال با دقت صفر | ۲ |
| مالیات | ارزش افزوده قابل‌پیکربندی (نه ثابت در کد)، کالاهای معاف، کسر از قرارداد (Tax Withholding Category) | ۲ |
| مودیان | ماژول جدا با کلاینت مستقل، صف ارسال، استعلام وضعیت، و HITL برای خطاها | ۳ |
| اسناد فارسی | قالب‌های چاپ RTL فاکتور، پیش‌فاکتور و رسید | ۲ |
| حقوق و بیمه | اپ `karyar_hr_ir` روی HRMS | ۳ به بعد |

**بین‌المللی‌سازی بیش از حد ممنوع است.** فقط دو کار از الان انجام می‌شود:

1. هسته `karyar` به ایران وابسته نباشد.
2. متن‌ها قابل ترجمه باشند.

چیزهایی مثل چندارزی پیچیده یا قواعد مالیاتی چندکشوری الان طراحی نمی‌شوند.

# ۲۸. ریسک‌ها و توازن‌ها

| # | ریسک | شدت | کاهش |
|---|---|---|---|
| R1 | دسترسی به ارائه‌دهندگان AI از ایران، و حقوقی بودن ارسال داده به خارج | **بالا** | AI Gateway چندارائه‌دهنده‌ای، مدل خودمیزبان، طبقه‌بندی داده، بررسی حقوقی |
| R2 | کانال‌های تلگرام، واتس‌اپ و اینستاگرام در ایران فیلتر هستند و API رسمی Meta برای کسب‌وکار ایرانی در دسترس نیست | **بالا** | شروع با ویجت وب و بله. Channel Gateway جداشده تا کانال جایگزین شود |
| R3 | ساختن موتور گردش‌کار، کار سختی است و می‌تواند دامنه پروژه را منفجر کند | **بالا** | مجموعه گام‌های حداقلی، تست‌های جدی، جدا کردن تعریف از اجراکننده (مسیر فرار به Temporal) |
| R4 | برداشت اشتباه مدل در تأیید گفت‌وگومحور | بالا | تصمیم ساخت‌یافته، تأیید صریح بر اساس ریسک، `payload_hash` |
| R5 | Prompt Injection از پیام مشتری یا فایل | بالا | مجوز مستقل از مدل، داده نامطمئن، Task Brief به‌جای متن خام |
| R6 | خلأ مجوز در گزارش‌های SQL خام ERPNext | متوسط | فقط Report Catalog بازبینی‌شده |
| R7 | هزینه AI از کنترل خارج شود (حلقه ایجنت) | متوسط | سقف سخت از روز اول و محدودیت گام در هر نوبت |
| R8 | پیکربندی بیش از حد، محصول را برای مشتری پیچیده کند | متوسط | Packهای آماده با پیش‌فرض‌های معقول. سازنده گرافیکی بعداً |
| R9 | ارتقای ERPNext ابزارها را بشکند | متوسط | آزمون قرارداد، API facade، پین نسخه |
| R10 | مجوزها: GPL و AGPL برای توزیع، و SUL برای n8n | متوسط | تصمیم مجوز Karyar و بررسی حقوقی |
| R11 | داده حساس پزشکی (مثال کلینیک) | بالا (برای آن بخش بازار) | مدل C برای Tenant، مدل خودمیزبان، پوشاندن فیلدها |
| R12 | تخصص کم تیم در Frappe | متوسط | شروع کوچک، الگوبرداری از کد ERPNext، آموزش |

# ۲۹. تصمیم‌هایی که نیاز به تأیید دارند

1. **پایه:** ERPNext و Frappe v16 به‌جای develop، و جابه‌جایی fork به `version-16`.
2. **ساختار مخازن:** چند مخزن (`karyar`، `karyar_iran`، `karyar-deploy`).
3. **چندمستأجری:** مدل B به‌عنوان پیش‌فرض و C برای حساس‌ها.
4. **محل میزبانی** و سیاست ارسال داده به ارائه‌دهندگان AI.
5. **ارائه‌دهندگان AI مجاز** و مدل تجاری: کلید سکو یا BYOK.
6. **اولین مشتری و صنعت پایلوت** (کلینیک؟ پخش؟). این تعیین می‌کند اولین Pack چه باشد.
7. **کانال‌های نسخه اول** (پیشنهاد: ویجت وب و بله و/یا تلگرام).
8. **فناوری Karyar Workspace:** React (الگوی banking) یا frappe-ui (Vue).
9. **نام‌گذاری:** `Responsibility` و «نقش کاری» به‌جای Role.
10. **سطح پیش‌فرض تأیید** برای عملیات مالی (`confirm` یا `strong`).
11. **مجوز Karyar** (GPLv3 متن‌باز یا فقط SaaS) و ادامه با n8n یا جایگزین.
12. **زبان کد و مستندات** (پیشنهاد: کد و شناسه‌ها انگلیسی، مستندات کاربر فارسی).

# ۳۰. اجزایی که عمداً برای فازهای بعد منعطف مانده‌اند

| جزء | الان | جای خالی در معماری |
|---|---|---|
| سازنده گرافیکی گردش‌کار و ایجنت | JSON و فرم Desk | Schema گراف نسخه‌دار. UI فقط یک ویرایشگر روی همین Schema است |
| حافظه بلندمدت ایجنت | ندارد | `context builder` قابل افزودن منبع، مثل Memory Store یا RAG روی اسناد |
| مدیریت هزینه و صورت‌حساب | فقط UsageRecord و سقف سخت | `UsageSink`، `Model Price` و Control Plane |
| اعلان چندکاناله | `notify` ساده | Notification Router با ترجیحات کاربر |
| صف پیشرفته و اولویت | صف‌های RQ | Relay رویداد و صف‌های اولویت‌دار |
| صوت | ندارد | Adapter کانال صوتی (STT/TTS) پشت Channel Gateway |
| MCP | ندارد | نمایش همان رجیستری ابزار با `frappe-mcp` |
| مشاوره ایجنت با ایجنت | ندارد | ابزار `consult` |
| تأیید موازی «M از N»، escalation و timeout | پایه | انواع گام `parallel`، `join` و `escalate` |
| RPA، مودیان، حقوق ایران | ندارد | انواع گام، `karyar_iran` و `karyar_hr_ir` |
| Observability کامل | لاگ و داشبورد ساده | OpenTelemetry |
| PostgreSQL | MariaDB | کد مستقل از نوع پایگاه داده و ابزار بررسی `postgres_compat` در CI |
| اجراکننده Temporal | موتور داخلی | جدایی تعریف از اجراکننده |

# ۳۱. مثال‌های سرتاسری

> در همه مثال‌ها نام ایجنت‌ها فقط **انتساب** است. تعریف‌های گردش‌کار به نقش کاری اشاره می‌کنند.

## مثال ۱ — مشتری ← Ava ← تأیید مشتری ← بازبینی کارمند ← Lead در ERPNext ← Hanna (کلینیک کاشت مو)

<div class="flow"><span>پیام مشتری</span><span>Ava جمع‌آوری</span><span>تأیید مشتری</span><span>بازبینی کارمند</span><span>Lead در ERPNext</span><span>رویداد</span><span>Hanna</span></div>

| مورد | جزئیات |
|---|---|
| **Trigger** | `channel.message.received` از مخاطب جدید در اینستاگرام یا ویجت وب. پرونده بازی وجود ندارد، پس گردش‌کار `clinic_intake@v1` شروع می‌شود |
| **Agent** | Ava، با نقش کاری `intake`. توانمندی‌ها: گفت‌وگو با مشتری، `validate_phone`، `normalize_text`، `parse_persian_date`. **بدون** هیچ توانمندی نوشتن در ERPNext در این گام |
| **Data** | Schema ‏`IntakeData`: نام، موبایل، خدمت، سن، شهر، زمان ترجیحی. در `context` پرونده ذخیره می‌شود و **هنوز در ERPNext نیست** |
| **Workflow steps** | `agent_task(intake)` ← `validate` ← `human_task(customer_confirm, level=confirm)` ← `human_task(staff_review)` ← `erp_action(crm.lead.create)` ← `end` |
| **Human interaction** | (۱) مشتری: «بله، اطلاعاتم درسته، بفرستید». تأیید به hash ‏h1 گره می‌خورد. (۲) کارمند پذیرش در Workspace: «سن اشتباه است، ۳۸ کن». نتیجه `task.patch` و نسخه ۲ است. بعد: «بفرست مرحله بعد». سیستم خلاصه نهایی را نشان می‌دهد و کارمند «بله» می‌گوید. تصمیم approve با hash ‏h2 ثبت می‌شود. سیاست گام می‌گوید «اصلاح سن توسط کارمند، تأیید مجدد مشتری لازم ندارد» و این در ممیزی ثبت می‌شود |
| **ERPNext action** | ایجاد `Lead` با فیلدهای سفارشی (`service`، `age`) و `source=Instagram`، با کلید idempotency برابر شناسه پرونده |
| **Event** | `erp.lead.created` از Outbox |
| **Next agent** | گردش‌کار `sales_followup` که trigger آن ‏`erp.lead.created` با شرط `service in clinic_services` است، شروع می‌شود. گام اول `agent_task(sales)` است و انتساب می‌گوید با **Hanna** |
| **Final result** | Lead در ERPNext ثبت شده است. Hanna دستور کار ساخت‌یافته دارد و برای مشاور یک ToDo تعیین وقت می‌سازد. مشتری پیام «درخواست شما ثبت شد» را دریافت کرده است. **هیچ انسانی پرونده را دستی منتقل نکرد** |

## مثال ۲ — Hanna ← سفارش ← تأیید پرداخت توسط انسان ← رویداد ← Arman ← تأیید ← سند حسابداری

<div class="flow"><span>Hanna: سفارش</span><span>تأیید پرداخت (انسان)</span><span>Payment Entry پیش‌نویس</span><span>رویداد</span><span>Arman</span><span>تأیید حسابدار</span><span>ثبت نهایی</span></div>

| مورد | جزئیات |
|---|---|
| **Trigger** | ادامه پرونده `sales_followup`. مشتری پیش‌فاکتور را می‌پذیرد |
| **Agent** | Hanna (نقش `sales`) و سپس Arman (نقش `accounting`) |
| **Data** | Quotation و Sales Order. اطلاعات پرداخت: مبلغ، روش، شماره پیگیری و تصویر رسید |
| **Workflow steps** | (الف) `sales_followup`: ‏`erp_action(quotation.create_draft)` ← تأیید مشتری ← `erp_action(sales_order.create_and_submit)` (سیاست Tenant: ثبت خودکار سفارش مجاز است) ← `human_task(payment_confirm)` برای نقش «صندوق» ← `erp_action(payment_entry.create_draft)`. (ب) `accounting_posting`، با trigger ‏`erp.payment_entry.created` و شرط `docstatus=0 and karyar_source=sales_followup` |
| **Human interaction** | (۱) صندوق‌دار: «۲۰۰ میلیون ریال کارت‌به‌کارت شد، رسید پیوست». Hanna فیلدها را پر می‌کند و صندوق‌دار تأیید می‌کند. (۲) حسابدار: Arman خلاصه را نشان می‌دهد: «پرداخت پیش، مطابق سفارش؛ فاکتور فروش پیش‌نویس آماده است». حسابدار: «حساب بانک را ملت بگذار». نتیجه `patch` است. تأیید با سطح `strong` (تأیید صریح به‌علاوه OTP) |
| **ERPNext action** | Arman: ‏`make_sales_invoice` به‌صورت پیش‌نویس، تطبیق مبلغ‌ها، اتصال Payment Entry. بعد از تأیید: ‏`submit` هر دو سند و ایجاد ثبت‌های دفتر کل توسط خود ERPNext |
| **Event** | `erp.payment_entry.created` و بعد `erp.sales_invoice.submitted` |
| **Next agent** | (اختیاری) گردش‌کار `moadian_submit` در `karyar_iran` که سیستمی است و ایجنت ندارد. اعلان فاکتور به مشتری از کانال |
| **Final result** | سند حسابداری با تأیید حسابدار ثبت شد. Audit Log شامل این‌هاست: پیشنهاد Arman، اصلاح حسابدار، hash تأییدشده، و اسناد اثرپذیرفته |

## مثال ۳ — یک ایجنت با چند نقش (CRM + فروش + حسابداری) در شرکت کوچک

| مورد | جزئیات |
|---|---|
| **Trigger** | همان گردش‌کارهای مثال ۱ و ۲ (از همان Pack) |
| **Agent** | «سارا». انتساب نقش در این Tenant: `intake`، `sales` و `accounting`، همه با سارا |
| **Data** | همان Schemaها |
| **Workflow steps** | بدون تغییر در تعریف. Override در Tenant: گام `staff_review` با شرط `tenant.policy.staff_review=false` رد می‌شود |
| **Human interaction** | فقط یک نفر (مالک): تأیید قبل از ثبت حسابداری، با سطح `confirm` |
| **ERPNext action** | مثل مثال ۲ |
| **Event** | همان رویدادها. **تحویل هنوز با رویداد انجام می‌شود**، حتی اگر فرستنده و گیرنده هر دو سارا باشند |
| **Next agent** | سارا در نقش بعدی |
| **Final result** | نکته معماری: ترکیب نقش‌ها **مجوزها را در هم ادغام نمی‌کند**. در گفت‌وگو با مشتری، سارا فقط توانمندی‌های گام `intake` را دارد و ابزار حسابداری برایش در دسترس نیست (Step Scope). در ممیزی هم مشخص است «سارا در نقش حسابداری در گام X» عمل کرده است |

## مثال ۴ — دو گام تأیید انسانی متفاوت (خرید)

<div class="flow"><span>ERPNext: درخواست مواد</span><span>ایجنت انبار</span><span>تأیید ۱: سرپرست انبار</span><span>تأیید ۲: مدیر مالی</span><span>ثبت سفارش خرید</span><span>n8n: ارسال به تأمین‌کننده</span></div>

| مورد | جزئیات |
|---|---|
| **Trigger** | ERPNext بر اساس «سطح سفارش مجدد» خودش Material Request ثبت می‌کند. رویداد `erp.material_request.submitted` گردش‌کار `procurement@v2` را شروع می‌کند |
| **Agent** | ایجنت نقش `inventory` |
| **Data** | اقلام، مقدار، تأمین‌کنندگان قبلی و آخرین قیمت‌ها (از گزارش فهرست‌شده «Item-wise Purchase History») |
| **Workflow steps** | `agent_task(inventory: draft PO)` ← `condition(total > threshold)` ← `human_task(approval_1: role=Stock Manager, editable=[qty, supplier])` ← `human_task(approval_2: role=Accounts Manager, level=strong, show=budget_report)` ← `erp_action(purchase_order.submit)` ← `integration_call(n8n: send_po)` |
| **Human interaction** | سرپرست انبار: «۲۰ تا کم کن، تأمین‌کننده B ارزان‌تر است». ایجنت اصلاح می‌کند و تأیید ۱ ثبت می‌شود. مدیر مالی: «با این بودجه، فعلاً نصفش را بخر» و **رد همراه با دلیل**. شاخه رد پرونده را به گام ایجنت برمی‌گرداند و پیش‌نویس جدید ساخته می‌شود. سیاست `re_review_on_change=all_previous` می‌گوید هر دو تأیید باید دوباره گرفته شوند |
| **ERPNext action** | Purchase Order پیش‌نویس و سپس ثبت نهایی |
| **Event** | `erp.purchase_order.submitted` و سپس callback از n8n |
| **Next agent** | ندارد. بعدها رسیدن کالا (`erp.purchase_receipt.submitted`) گردش‌کار تحویل را شروع می‌کند |
| **Final result** | سفارش خرید با دو تأیید مستقل ثبت و برای تأمین‌کننده ارسال شد. هر دو دور تأیید در ممیزی موجود است |

## مثال ۵ — ترکیب کاملاً متفاوت: تولیدکننده مواد غذایی، بدون Hanna و Arman

ایجنت‌ها: **Ava** (سفارش عمده از بله)، **نیما** (انبار + تولید)، **کیفیت**، و **مدیریت**.

| مورد | جزئیات |
|---|---|
| **Trigger** | پیام پخش‌کننده در بله: «۵۰ کارتن رب ۸۰۰ گرمی برای فروشگاه شعبه ۲» |
| **Agent** | Ava (نقش `b2b_order_intake`). مشتری از طریق `Contact Identity` شناسایی می‌شود |
| **Data** | قلم کالا (تطبیق نام محاوره‌ای با Item از راه ابزار جست‌وجو)، مقدار، آدرس، اعتبار مشتری |
| **Workflow steps** | `agent_task(intake)` ← `validate(price_list, credit_limit)` ← `condition(customer_group=trusted and amount<credit)`. اگر درست بود، **بدون انسان** به `erp_action(sales_order.submit)`. اگر نه، `human_task(sales_manager)`. سپس `fulfillment`: ‏`agent_task(nima: stock check)` ← در صورت کمبود `erp_action(work_order.draft)` ← `human_task(production_supervisor)` ← `wait_event(erp.stock_entry.submitted: Manufacture)` ← `agent_task(quality)` ← `human_task(qc_technician)` ← `erp_action(quality_inspection.submit)` ← `erp_action(delivery_note.draft)` ← `human_task(warehouse_loading)` ← `erp_action(delivery_note.submit)` ← `notify(customer via Bale)` |
| **Human interaction** | سرپرست تولید: «بگذارش برای شیفت فردا صبح». نیما زمان Work Order را اصلاح می‌کند. تکنسین کیفیت به‌صورت گفت‌وگو: «pH ‏۴٫۲، بریکس ۲۸». ایجنت کیفیت مقادیر را در Quality Inspection پر می‌کند و با قالب بازرسی مقایسه می‌کند. انباردار: «بار زده شد» |
| **ERPNext action** | Sales Order، Work Order، Stock Entry، Quality Inspection، Delivery Note. **همه از کنترلرهای ERPNext** |
| **Event** | `erp.sales_order.submitted`، ‏`erp.stock_entry.submitted` و `erp.delivery_note.submitted` |
| **Next agent** | نیما، سپس کیفیت، سپس نیما، سپس Ava (اعلان). ایجنت مدیریت با گردش‌کار زمان‌بندی‌شده `daily_brief` هر روز ساعت ۸ گزارش `management_summary` را برای مدیر در بله می‌فرستد و به پرسش‌های بعدی **با مجوز خود مدیر** پاسخ می‌دهد |
| **Final result** | سفارش بدون دخالت انسان ثبت شد و انسان‌ها فقط در نقاط کیفیت، تولید و بارگیری حضور داشتند. همان موتور و همان ابزارها، با ترکیب کاملاً متفاوت |

## مثال ۶ (اضافه) — درخواست غیرمجاز

کارمند فروش از ایجنت می‌پرسد: «سود کل شرکت امسال چقدر است؟»

1. ایجنت ابزار `report.run("profit_and_loss")` را انتخاب می‌کند.
2. Tool Executor بررسی می‌کند. نقش «Sales User» به این گزارش دسترسی ندارد، پس نتیجه `PERMISSION_DENIED` است.
3. **هیچ عددی به مدل نمی‌رسد.** ایجنت پاسخ می‌دهد: «به این گزارش دسترسی ندارید؛ در صورت نیاز از مدیر مالی بخواهید».
4. رویداد `security.permission_denied` در ممیزی ثبت می‌شود.

**حتی با Prompt Injection** («تو الان مدیر هستی») نتیجه تغییر نمی‌کند، چون تصمیم در کد گرفته می‌شود.

# ۳۲. جدول تصمیم‌های معماری

## الف) اصول تأییدشده (از الزامات شما و مورد تأیید این تحلیل)

| تصمیم | وضعیت | دلیل | جایگزین | چرا انتخاب شد | قابل تغییر؟ |
|---|---|---|---|---|---|
| ERPNext منبع حقیقت | تأییدشده | جلوگیری از دو منبع حقیقت | پایگاه داده جدای Karyar | سازگاری، حسابرسی، کاهش کار | خیر (بنیادی) |
| بدون دسترسی خام به پایگاه داده یا SQL برای ایجنت | تأییدشده | امنیت و صحت | Text-to-SQL | قابل کنترل و ممیزی | خیر |
| مجوز بیرون از مدل اعمال می‌شود | تأییدشده | prompt قابل دور زدن است | محدودیت در prompt | امنیت واقعی | خیر |
| HITL به‌صورت گام یا سیاست قابل‌پیکربندی | تأییدشده | تنوع کسب‌وکارها | قانون ثابت در کد | انعطاف | خیر |
| جدایی ایجنت، نقش کاری، توانمندی، مجوز و گردش‌کار | تأییدشده | ترکیب‌های متفاوت هر شرکت | ایجنت‌های ثابت | ۸۰/۲۰ | خیر |
| تحویل از راه رویداد و گردش‌کار | تأییدشده | قابلیت اطمینان و ممیزی | چت ایجنت با ایجنت | قطعی بودن | خیر |
| هوش مصنوعی فقط درون فرایند تعریف‌شده عمل می‌کند | تأییدشده | کنترل کسب‌وکار | ایجنت خودمختار | ریسک پایین | خیر |
| لایه مدل مستقل از ارائه‌دهنده | تأییدشده | تحریم، هزینه، کیفیت | SDK یک ارائه‌دهنده | انعطاف | خیر |
| بدون تغییر هسته ERPNext | تأییدشده | ارتقاپذیری | fork سنگین | هزینه نگهداری | فقط با دلیل مستند |
| n8n فقط برای یکپارچه‌سازی و RPA فقط برای سیستم‌های بدون API | تأییدشده | جلوگیری از پراکندگی منطق | منطق در n8n | نگهداری‌پذیری | خیر |

## ب) پیشنهادی، هنوز تأییدنشده

| تصمیم | وضعیت | دلیل | جایگزین | چرا انتخاب شد | قابل تغییر؟ |
|---|---|---|---|---|---|
| پایه ERPNext و Frappe v16 | پیشنهادی | پایدار و پشتیبانی‌شده | develop (v17)، v15 | ثبات به‌علاوه عمر پشتیبانی | بله (ارتقا به v17) |
| MariaDB | پیشنهادی | تنها گزینه رسمی v16 | PostgreSQL | پشتیبانی | بله در v17 |
| مدل B (سایت برای هر Tenant) و C برای حساس‌ها | پیشنهادی | جداسازی پایگاه داده با هزینه معقول | A یا C برای همه | توازن | بله (انتقال سایت) |
| چند مخزن و `karyar-deploy` | پیشنهادی | سازوکار bench و جدایی از هسته | تک‌مخزن داخل fork | ارتقاپذیری | بله |
| یک اپ `karyar` با ماژول‌های جدا | پیشنهادی | سادگی | چند اپ از ابتدا | سربار کمتر | سخت‌تر (انتقال DocType) |
| موتور گردش‌کار سبک داخل Frappe | پیشنهادی | هم‌تراکنش و هم‌مجوز | Temporal، n8n، Frappe Workflow | بدون زیرساخت جدید | بله (تعریف از اجراکننده جداست) |
| Outbox و RQ برای رویداد | پیشنهادی | موجود و قابل اعتماد | Kafka، RabbitMQ | سادگی | بله (Relay) |
| اجرای ایجنت در Workerهای RQ | پیشنهادی | چندمستأجری و مجوز طبیعی | سرویس async جدا | سادگی | بله (رابط تمیز) |
| حلقه ایجنت سبک خودمان | پیشنهادی | کنترل کامل | LangGraph یا CrewAI به‌عنوان مالک | ممیزی و مجوز | بله |
| `Responsibility` به‌عنوان نقش کاری | پیشنهادی | جلوگیری از اشتباه با Role در Frappe | Role | وضوح | بله (نام) |
| نوشتن پیش‌نویس‌محور و Action Proposal | پیشنهادی | ایمنی | نوشتن مستقیم | قابل تأیید | خیر (توصیه) |
| تصمیم ساخت‌یافته، `payload_hash` و سطح تأیید | پیشنهادی | گفت‌وگو و ممیزی با هم | فقط دکمه، یا فقط تفسیر مدل | هر دو هدف | سطوح قابل تنظیم |
| یک لایه سرویس با سه نمایش (ابزار، REST، MCP) | پیشنهادی | منطق یکتا | APIهای جدا | سازگاری | خیر (توصیه) |
| AI Gateway به‌صورت کتابخانه و بعداً LiteLLM | پیشنهادی | شروع ساده | LiteLLM از روز اول | سربار کمتر | بله |
| Pack برای ۸۰/۲۰ | پیشنهادی | پیکربندی قابل حمل | clone کد | نگهداری‌پذیری | بله |
| Karyar Workspace به‌صورت SPA درون Frappe | پیشنهادی | تجربه کاربری غیر ERP | فقط Desk، یا فقط پیام‌رسان | کارایی کارکنان | بله |
| سقف سخت هزینه AI از روز اول | پیشنهادی | حفاظت مالی | بدون سقف | ریسک | خیر (توصیه) |
| میزبانی ترکیبی | پیشنهاد اولیه | AI و کانال‌ها در برابر اقامت داده | داخل یا خارج کامل | توازن | بله |

## ج) پرسش‌های باز

| پرسش | چرا مهم است | چه کسی تصمیم می‌گیرد |
|---|---|---|
| محل میزبانی و اقامت داده | قانونی، دسترسی AI و کانال‌ها | شما و مشاور حقوقی |
| ارائه‌دهندگان AI و کلید سکو یا BYOK | هزینه، کیفیت فارسی، قانون | شما |
| صنعت و مشتری پایلوت | اولین Pack | شما |
| کانال‌های نسخه اول | دسترسی در ایران | شما |
| React یا frappe-ui برای Workspace | مهارت تیم | تیم فنی |
| مجوز Karyar و جایگزین n8n | مدل کسب‌وکار | شما و حقوقی |
| سطح تأیید پیش‌فرض مالی | امنیت در برابر سرعت | شما |
| سیاست داده پزشکی | قانونی | شما و حقوقی |
| آیا مشتری به Desk دسترسی دارد؟ کدام نقش‌ها؟ | امنیت چندمستأجری | شما |
| Control Plane: سفارشی یا Press | مقیاس | بعداً |

## د) ویژگی‌های فازهای آینده

| ویژگی | پیش‌نیازی که از الان در معماری هست |
|---|---|
| سازنده گرافیکی گردش‌کار و ایجنت | Schema گراف نسخه‌دار |
| تأیید موازی، escalation و timeout | رجیستری نوع گام و `Karyar Timer` |
| مدیریت هزینه، صورت‌حساب و سهمیه | `UsageRecord`، `UsageSink` و `Model Price` |
| حافظه بلندمدت و RAG | Context Builder قابل افزودن |
| صوت | آداپتور کانال |
| MCP | رجیستری ابزار |
| RPA | گام `rpa_job` و Human Task |
| مودیان، چارت حساب، مالیات و جلالی کامل | `karyar_iran` و `regional_overrides` |
| حقوق ایران | `karyar_hr_ir` روی HRMS |
| مشاوره ایجنت با ایجنت | ابزار `consult` |
| مشاهده‌پذیری کامل | `correlation_id` و لاگ ساخت‌یافته |
| اجراکننده Temporal | جدایی تعریف و اجراکننده |
| PostgreSQL | کد مستقل از نوع پایگاه داده |
| Control Plane و راه‌اندازی خودکار | اسکریپت‌های `karyar-deploy` |

# ۳۳. تعارض‌ها و محدودیت‌هایی که صریح گفته می‌شوند

1. **«کاربر نیازی به ERPNext ندارد» در برابر واقعیت.** حسابدار برای بستن دوره، مغایرت‌گیری و گزارش‌های پیچیده همچنان به Desk نیاز دارد. Karyar کار روزمره را از Desk بیرون می‌آورد، نه کار تخصصی را. **پیشنهاد:** Workspace برای همه، Desk برای کاربران حرفه‌ای.
2. **«بدون داده موازی» در برابر نیاز فرایند.** Ava قبل از ثبت Lead باید داده را جایی نگه دارد. این داده «وضعیت فرایند» است، نه «حقیقت کسب‌وکار». مرز در بخش ۲۲ تعریف شده است.
3. **گفت‌وگوی آزاد در برابر ممیزی.** با تصمیم ساخت‌یافته و تأیید صریح حل شده است (بخش ۸). تأیید «کاملاً آزاد و بدون تأیید صریح» برای عملیات مالی **پیشنهاد نمی‌شود**.
4. **دو ماشین حالت روی یک سند.** اگر Frappe Workflow و Karyar هر دو روی Sales Order حالت نگه دارند، تعارض پیش می‌آید. **قاعده:** روی DocTypeهایی که Karyar مدیریت می‌کند، Frappe Workflow فعال نشود، یا Karyar از `apply_workflow` استفاده کند.
5. **«همه‌چیز قابل‌پیکربندی» در برابر «پیچیده نکن».** در فاز ۱ پیکربندی را تیم Karyar با JSON و فرم انجام می‌دهد. سازنده گرافیکی برای کاربر نهایی بعداً می‌آید.
6. **ابر مشترک در برابر اقامت داده و داده پزشکی.** ممکن است بعضی مشتریان مدل C یا میزبانی اختصاصی بخواهند. معماری این را پشتیبانی می‌کند، اما هزینه‌اش بالاتر است.
7. **کانال‌های پیشنهادی در برابر واقعیت ایران.** اینستاگرام، واتس‌اپ و تلگرام فیلتر هستند و API رسمی Meta برای کسب‌وکار ایرانی در دسترس نیست. شروع با ویجت وب و بله منطقی‌تر است.
8. **ایجنت مدیریت با «دید کلی» در برابر حداقل دسترسی.** ایجنت مدیریت با مجوز **خود مدیر** عمل می‌کند و دسترسی ویژه‌ای ندارد.
9. **«یک نقش توسط چند ایجنت یا انسان».** این به منطق مسیریابی (تقسیم بار، نوبتی) نیاز دارد. در فاز ۱ هر نقش در هر Tenant یک ایجنت دارد، و برای انسان‌ها Assignment Rule در Frappe استفاده می‌شود.
10. **زنجیره خودکار در برابر شکست خاموش.** وقتی «هیچ‌کس دستی منتقل نمی‌کند»، پرونده گیرکرده دیده نمی‌شود. پس وضعیت `needs_attention`، داشبورد پرونده‌های گیرکرده و هشدار **الزامی‌اند**.
11. **«خلاصه‌سازی توسط ایجنت» در برابر «AI منبع عدد نیست».** خلاصه مجاز است، اما اعداد فقط از نتیجه گزارش می‌آیند و منبع نمایش داده می‌شود.

# ۳۴. خروجی نهایی

## ۳۴.۱ معماری پیشنهادی (در یک نگاه)

- **پایه:** ERPNext و Frappe **v16** با **MariaDB**. هسته دست‌نخورده می‌ماند و fork فقط به‌عنوان آینه با پایه `version-16` نگه داشته می‌شود.
- **اپ‌ها:**
  - **`karyar`** (سکوی ایجنت: ایجنت، گردش‌کار، HITL، رویداد، سیاست، ابزار، API، AI Gateway، ممیزی، Workspace)
  - **`karyar_iran`** (بومی‌سازی ایران)
  - **`karyar_hr_ir`** (بعداً)
  - **`karyar-deploy`** (استقرار و Packها)
- **چندمستأجری:** یک Site برای هر Tenant روی Benchهای مشترک. Bench اختصاصی برای مشتریان حساس.
- **هسته رفتاری:**
  - گردش‌کار قطعی و نسخه‌دار، به‌علاوه ایجنت‌های هوشمند درون گام‌ها
  - تحویل با رویداد (Outbox)
  - HITL به‌صورت گام یا سیاست
  - تأیید ساخت‌یافته با `payload_hash`
  - مجوز چندلایه در کد که به Frappe ختم می‌شود
- **یکپارچه‌سازی:** n8n فقط از Karyar API. RPA به‌صورت سرویس جدا و بعداً.

## ۳۴.۲ توضیح ساده: کل سیستم چطور کار می‌کند؟

Karyar را مثل یک **شرکت با کارمندان دیجیتال** تصور کنید:

- **ERPNext دفتر رسمی شرکت است.** فقط چیزی که آنجا ثبت شده «واقعی» است.
- **گردش‌کارها دستورالعمل‌های اداری شرکت‌اند.** مشخص می‌کنند هر پرونده از کدام میزها و به چه ترتیبی عبور کند، و کجا امضای انسان لازم است.
- **ایجنت‌ها کارمندان دیجیتال‌اند.** هرکدام یک یا چند «سمت» (نقش کاری) دارد و فقط به «کشوهایی» (ابزارهایی) دسترسی دارد که سمتش اجازه می‌دهد.
- **وقتی کار یک میز تمام می‌شود، پرونده خودکار به میز بعدی می‌رود.** چون دفتر رسمی تغییر کرده یا مرحله تمام شده، نه چون کسی آن را دستی برده.
- **انسان‌ها فقط جایی وارد می‌شوند که دستورالعمل گفته.** آن هم با گفت‌وگوی معمولی: «سن را اصلاح کن»، «بفرست مرحله بعد». ولی امضای نهایی همیشه روشن، ثبت‌شده و به همان متنی که دیده‌اند گره خورده است.
- **نگهبان (Policy Engine) پیش از باز شدن هر کشو بررسی می‌کند** که چه کسی، از طرف چه کسی و در کدام مرحله می‌خواهد آن را باز کند. هوش مصنوعی نمی‌تواند نگهبان را قانع کند.

## ۳۴.۳ نمودار اصلی

بخش ۳ را ببینید. نسخه فشرده:

<pre class="ltr">
 Users / Customers / Employees
   │  Web · Telegram/Bale · WhatsApp · Instagram · Voice(later) · Karyar Workspace
   ▼
 Channel Gateway ──► Conversation ──► Event Bus (Outbox+RQ) ◄── ERPNext doc_events
                                        │
                     ┌──────────────────┴──────────────────┐
                     ▼                                     ▼
              Workflow Engine  ◄──── Human Task (conversational, hash-bound decisions)
                     │ assigns step → Responsibility → Agent
                     ▼
               Agent Runtime ──► AI Gateway ──► Providers (intl / local / self-hosted)
                     │                 └► UsageRecord (cost extension point)
                     ▼
     Tool Executor + Policy Engine (Tenant ∩ Agent ∩ Step ∩ Frappe perms)
                     │                     ▲
                     ▼                     │ Karyar API v1 (n8n, apps, MCP later)
        ERPNext / HRMS / karyar_iran  (SOURCE OF TRUTH, one site per tenant)
                     │
           MariaDB · Redis · Files          ⟂ Audit Log · Traces · Metrics (cross-cutting)
</pre>

## ۳۴.۴ ساختار مخازن

بخش ۲۳ را ببینید. خلاصه: `karyar-mainERP` (fork روی v16 و فقط آینه)، `karyar`، `karyar_iran`، `karyar-deploy`، و بعدها `karyar_hr_ir` و `karyar-rpa-worker`.

## ۳۴.۵ اجزای اصلی و مسئولیت‌ها

| جزء | مسئولیت در یک جمله |
|---|---|
| Channel Gateway | هر کانال را به پیام استاندارد تبدیل می‌کند و مخاطب را می‌شناسد |
| Conversation | تاریخچه گفت‌وگو و اتصال آن به پرونده |
| Workflow Engine | پرونده را طبق تعریف نسخه‌دار، پایدار و idempotent جلو می‌برد |
| Agent Runtime | یک نوبت هوشمند را با ابزارهای مجاز و خروجی معتبر اجرا می‌کند |
| Human Task | تعامل و تأیید انسانی، گفت‌وگومحور و ثبت‌شده |
| Capability & Tool Registry | فهرست بسته و تایپ‌شده کارهایی که ایجنت می‌تواند بکند |
| Policy Engine + Tool Executor | تنها دروازه اجرا: مجوز، ریسک، idempotency، فیلتر خروجی، ممیزی |
| Event Bus | رویداد مطمئن از ERPNext و Karyar به گردش‌کارها |
| Karyar API v1 | قرارداد پایدار برای دنیای بیرون |
| AI Gateway | انتزاع ارائه‌دهنده و مدل، و ثبت مصرف |
| Audit & Telemetry | چه کسی، چه کاری، به اجازه چه کسی، روی کدام سند، و چرا شکست خورد |
| Karyar Workspace | میز کار کارکنان: صندوق کار، گفت‌وگو، پرونده‌ها |
| karyar_iran | فارسی، جلالی، اعتبارسنج‌ها، حسابداری و مالیات، مودیان |
| n8n / RPA | اتصال به بیرون، بدون منطق اصلی کسب‌وکار |

## ۳۴.۶ بزرگ‌ترین ریسک‌ها

1. **دسترسی و قانونی بودن ارائه‌دهندگان AI از ایران**، و ارسال داده حساس به بیرون.
2. **دسترسی کانال‌ها در ایران** (فیلتر، و API رسمی Meta).
3. **دامنه موتور گردش‌کار:** خطر ساختن یک BPMN کامل به‌جای یک هسته کوچک.
4. **تأیید گفت‌وگومحور و Prompt Injection.**
5. **هزینه AI** بدون سقف سخت.

جزئیات در بخش ۲۸ آمده است.

## ۳۴.۷ تصمیم‌هایی که قبل از پیاده‌سازی باید بگیرید

بخش ۲۹ را ببینید. پنج مورد فوری:

1. **v16** به‌عنوان پایه
2. **محل میزبانی و سیاست داده**
3. **ارائه‌دهنده یا ارائه‌دهندگان AI**
4. **صنعت و مشتری پایلوت**
5. **کانال‌های نسخه اول**

## ۳۴.۸ بخش‌هایی که عمداً منعطف می‌مانند

بخش ۳۰ را ببینید: سازنده گرافیکی، حافظه، مدیریت هزینه، صوت، MCP، تأیید موازی، RPA، مودیان، مشاهده‌پذیری کامل، اجراکننده Temporal و PostgreSQL.

## ۳۴.۹ نقشه راه پیاده‌سازی (از مخزن فعلی تا اولین مشتری)

> زمان‌ها **تقریبی** و برای تیم ۲ تا ۳ نفره توسعه‌دهنده آشنا با Python هستند. بعد از تصمیم‌های بخش ۲۹ دقیق‌تر می‌شوند.

| فاز | محتوا | معیار خروج |
|---|---|---|
| **۰. پایه‌گذاری** (حدود ۱ تا ۲ هفته) | تصمیم‌های فوری. انتقال fork به `version-16`. ساخت مخازن `karyar` و `karyar-deploy`. bench توسعه با Docker (v16 و HRMS). CI شامل ruff، تست‌ها، و قانون semgrep «ممنوعیت `ignore_permissions` و `get_all` و `db.sql` در ابزارها». پوشه ADR (ثبت تصمیم‌ها) | اپ خالی `karyar` روی سایت v16 نصب می‌شود و CI سبز است |
| **۱. هسته سکو** (حدود ۴ تا ۶ هفته) | DocTypeهای اصلی (ایجنت، نقش کاری، توانمندی، انتساب، تعریف و اجرای گردش‌کار، گام، Human Task، رویداد، Audit، AI Usage، گفت‌وگو). رجیستری، اجراکننده ابزار و Policy. Outbox و Dispatcher. AI Gateway با یک ارائه‌دهنده سازگار با OpenAI، UsageRecord و سقف سخت. Agent Runtime محدود. موتور گردش‌کار با گام‌های `trigger`، `agent_task`، `human_task`، `condition`، `validate`، `erp_action`، `wait_event`، `notify` و `end`. API v1 حداقلی. ویجت وب. Workspace نسخه ۰ (صندوق کار و گفت‌وگو) | یک گردش‌کار نمونه سرتاسری اجرا می‌شود و **تست‌های مجوز** (از جمله مثال ۶) پاس می‌شوند |
| **۲. جریان پایلوت** (حدود ۴ تا ۶ هفته) | Pack نسخه ۱ برای صنعت پایلوت. مثال ۱ (Ava، بازبینی، Lead، Hanna) روی وب و یک پیام‌رسان (بله یا تلگرام). مبانی `karyar_iran` (نرمال‌سازی، اعتبارسنج‌ها، نمایش جلالی، `parse_persian_date`). HITL گفت‌وگومحور با سطوح تأیید. Report Catalog با ۵ تا ۱۰ گزارش. n8n برای پیامک | پایلوت داخلی با مشتریان شبیه‌سازی‌شده، و بازبینی امنیتی زنجیره مجوز |
| **۳. جریان مالی و استحکام** (حدود ۴ تا ۶ هفته) | مثال ۲ (Arman) با تأیید `strong`. تایمر و escalation. داشبورد پرونده‌های گیرکرده. پشتیبان‌گیری و تمرین بازیابی. تست بار صف ایجنت. محیط staging. اسکریپت راه‌اندازی Tenant. قالب‌های چاپ فارسی و قالب مالیات | چک‌لیست راه‌اندازی تکمیل شده است |
| **۴. اولین مشتری** | راه‌اندازی با Pack به‌علاوه ۲۰٪ پیکربندی اختصاصی. پشتیبانی فشرده ۲ تا ۴ هفته. اندازه‌گیری هزینه AI برای هر پرونده، زمان تأییدها و خطاها | مشتری در حال استفاده و متریک‌ها جمع‌آوری می‌شوند |
| **بعد** | مودیان، چارت حساب ایرانی، سازنده گرافیکی، تأیید موازی، RPA، Control Plane، مدیریت هزینه، حافظه، صوت، MCP | به ترتیب نیاز بازار |

**قدم بعدی پیشنهادی:** پاسخ به پنج تصمیم فوری (۳۴.۷). بعد از آن، فاز ۰ را با یک PR کوچک شروع می‌کنیم: انتقال پایه به v16 و اسکلت مخزن `karyar`.

---

<p class="endnote">این سند پیش‌نویس است و برای بازبینی و تکرار مشترک تهیه شده است. منابع اصلی: بررسی مستقیم کد این مخزن و کد Frappe و ERPNext نسخه ۱۶ در GitHub، مستندات n8n، issue شماره 56241 ‏(PostgreSQL) در frappe/erpnext، مخزن frappe/mcp، و بررسی گزارش قبلی همین گفت‌وگو.</p>
