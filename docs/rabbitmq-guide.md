# راهنمای RabbitMQ — راه‌اندازی، پنل مدیریت و تمرین عملی

> این راهنما همراه تسک «افزودن RabbitMQ» است (`docs/__logic/add-rabbitmq.md`، `docs/__plan/add-rabbitmq.md`).
> همهٔ دستورها از داخل پوشهٔ همین پروژه (devops) اجرا می‌شوند. همهٔ دستورها و تمرین‌ها روی RabbitMQ 4.1 واقعاً اجرا و آزموده شده‌اند.

---

## ۱. راه‌اندازی و ورود به پنل

```bash
# اولین بار (اگر شبکهٔ مشترک هنوز ساخته نشده)
docker network create airplus-network

# بالا آوردن فقط RabbitMQ (یا بدون نام سرویس، کل مجموعه)
docker compose up -d s-rabbitmq-service-fastapi

# وضعیت — باید (healthy) نشان دهد؛ اولین بار حدود ۲۰ تا ۳۰ ثانیه طول می‌کشد
docker compose ps s-rabbitmq-service-fastapi

# لاگ‌ها
docker compose logs -f s-rabbitmq-service-fastapi
```

| چیز | مقدار (از `.env`) |
|---|---|
| پنل مدیریت (مرورگر) | http://localhost:45683 |
| اتصال برنامه‌ها از روی سیستم خودتان (AMQP) | `localhost:45684` |
| اتصال برنامه‌ها از داخل شبکهٔ داکر | `s-rabbitmq-service-fastapi:5672` |
| نام کاربری | مقدار `RABBITMQ_USER` (پیش‌فرض: `airplus`) |
| رمز عبور | مقدار `RABBITMQ_PASSWORD` |
| فضای مجازی (vhost) | مقدار `RABBITMQ_VHOST` (پیش‌فرض: `accounting`) |

> ⚠️ رمز داخل `.env` فقط برای محیط توسعه است؛ این فایل در گیت ثبت شده و مخزن عمومی است.

---

## ۲. مفاهیم پایه — با مثال از سیستم خودمان

تصور کنید «موتور اسناد» می‌خواهد به «دفترداری» بگوید «سند ۱۰۱ موقت شد، آن را بساز»، و دفترداری جواب بدهد «انجام شد».

| مفهوم | یعنی چه | در مثال ما |
|---|---|---|
| **Producer** (تولیدکننده) | برنامه‌ای که پیام می‌فرستد | موتور اسناد |
| **Message** (پیام) | یک بستهٔ داده + مشخصات (properties) | `{"document_id":101}` |
| **Exchange** (مبادله‌گر) | «ادارهٔ پست»؛ پیام را می‌گیرد و تصمیم می‌گیرد به کدام صف‌ها برود. **پیام هیچ‌وقت مستقیم به صف فرستاده نمی‌شود، همیشه به یک exchange.** | `demo.documents` |
| **Routing key** (کلید مسیریابی) | برچسبی که فرستنده روی پیام می‌زند | `document.temporary.created` |
| **Queue** (صف) | صندوق پستی؛ پیام‌ها آنجا می‌مانند تا کسی برشان دارد | `demo.ledger.inbox` (صندوق دفترداری) |
| **Binding** (اتصال) | قانون «اگر کلید مسیریابی شبیه X بود، به این صف بفرست» | `document.temporary.*` ← صندوق دفترداری |
| **Consumer** (مصرف‌کننده) | برنامه‌ای که از صف پیام برمی‌دارد | دفترداری |
| **Ack** (تأیید دریافت) | مصرف‌کننده می‌گوید «پیام را پردازش کردم، پاکش کن». اگر قبل از Ack قطع شود، پیام به صف برمی‌گردد — هیچ پیامی گم نمی‌شود. | دفترداری بعد از ساختن سند، Ack می‌دهد |
| **Reject / Nack** | «این پیام را نمی‌توانم پردازش کنم» — یا دوباره به صف برمی‌گردد (requeue) یا دور انداخته/به صف مرده می‌رود | پیام خراب |
| **Dead-letter exchange** (DLX) | exchange‌ای که پیام‌های ردشده/منقضی‌شده به آن می‌روند تا گم نشوند و بعداً بررسی شوند | `demo.dlx` ← `demo.dead-letters` |
| **Durable** (ماندگار) — برای صف/exchange | بعد از ری‌استارت RabbitMQ خود صف/exchange باقی می‌ماند | همهٔ صف‌های ما |
| **Persistent** (ماندگار) — برای پیام | پیام روی دیسک نوشته می‌شود و بعد از ری‌استارت باقی می‌ماند (`delivery_mode: 2`) | همهٔ پیام‌های مالی |
| **vhost** | یک «RabbitMQ مجازی» جدا؛ صف‌ها و کاربران هر vhost از بقیه جدا هستند | `accounting` |

### انواع exchange
- **direct** — کلید مسیریابی باید **دقیقاً** برابر کلید اتصال باشد.
- **topic** — کلید اتصال می‌تواند الگو باشد؛ کلیدها با نقطه تکه‌تکه می‌شوند:
  - `*` یعنی **دقیقاً یک** تکه: `document.temporary.*` با `document.temporary.created` جور است، با `document.temporary.a.b` نه.
  - `#` یعنی **صفر یا چند** تکه: `ledger.reply.#` با `ledger.reply.done` و `ledger.reply.rejected.closed_period` جور است.
- **fanout** — کلید را نادیده می‌گیرد و به **همهٔ** صف‌های وصل‌شده می‌فرستد (برای DLX مناسب است).

### انواع صف
- **quorum** — صف مدرن و امن؛ همیشه ماندگار؛ برای داده‌های مهم (مثل ما) پیشنهاد می‌شود.
- **classic** — صف قدیمی‌تر و سبک‌تر.

---

## ۳. جریان آینده — موتور اسناد ↔ دفترداری (فقط برای فهم)

```
موتور اسناد ──(document.temporary.created)──► [exchange: documents] ──► [صف: ledger.inbox] ──► دفترداری
                                                                                   │ رد شد و قابل پردازش نیست
                                                                                   ▼
                                                                         [DLX] ──► [صف: dead-letters]

دفترداری ──(ledger.reply.done / ledger.reply.rejected)──► [exchange: documents] ──► [صف: document.replies] ──► موتور اسناد
```

- موتور اسناد پیام «سند موقت شد» را می‌فرستد و **منتظر جواب** می‌ماند (قانون هم‌زمانی فاز ۱).
- دفترداری پیام را برمی‌دارد، سند را می‌سازد، و **فقط بعد از commit** جواب «انجام شد» را می‌فرستد و Ack می‌دهد.
- اگر همان پیام دوبار برسد (مثلاً بعد از قطعی)، دفترداری با شناسهٔ یکتای پیام تشخیص می‌دهد و سند دوم نمی‌سازد (idempotency).

> این فقط برای فهم است؛ طراحی نهایی نام‌ها و صف‌ها در تسک «موتور اسناد» انجام می‌شود. نام‌های تمرین‌های زیر همه با `demo.` شروع می‌شوند.

---

## ۴. گشت در پنل

بعد از ورود، بالای صفحه این زبانه‌ها هست:

| زبانه | چه نشان می‌دهد |
|---|---|
| **Overview** | وضعیت کلی: تعداد پیام‌های آماده/در انتظار Ack، نمودار نرخ ورود و خروج پیام، وضعیت نود (حافظه، دیسک). |
| **Connections** | برنامه‌هایی که الان به RabbitMQ وصل‌اند (فعلاً خالی؛ بعداً سرویس حسابداری اینجا دیده می‌شود). |
| **Channels** | کانال‌های داخل هر اتصال — هر برنامه معمولاً روی یک اتصال چند کانال باز می‌کند. |
| **Exchanges** | فهرست exchangeها؛ با کلیک روی هرکدام: اتصال‌ها (Bindings) و بخش **Publish message** برای فرستادن پیام دستی. |
| **Queues and Streams** | فهرست صف‌ها با تعداد پیام‌ها (Ready = آماده، Unacked = برداشته‌شده ولی هنوز Ack نشده)؛ با کلیک روی هر صف: نمودار، اتصال‌ها، **Get messages**، **Purge**، **Delete**. |
| **Admin** | کاربران، vhostها، دسترسی‌ها، سیاست‌ها (Policies) و محدودیت‌ها. |

> گوشهٔ بالا-راست یک انتخاب‌گر **Virtual host** هست؛ آن را روی `accounting` بگذارید تا فهرست‌ها شلوغ نشوند.
> آمار پنل هر ~۵ ثانیه به‌روز می‌شود؛ اگر عددی بلافاصله عوض نشد، چند ثانیه صبر کنید.

---

## ۵. تمرین الف — فقط با پنل، بدون کد

### گام ۱ — صف پیام‌های مرده
1. زبانهٔ **Exchanges** ← بخش **Add a new exchange**:
   Virtual host = `accounting`، Name = `demo.dlx`، Type = `fanout`، Durability = `Durable` ← **Add exchange**.
2. زبانهٔ **Queues and Streams** ← **Add a new queue**:
   Virtual host = `accounting`، Type = `Quorum`، Name = `demo.dead-letters` ← **Add queue**.
3. روی `demo.dlx` در Exchanges کلیک کنید ← بخش **Bindings** ← To queue = `demo.dead-letters`، Routing key خالی ← **Bind**.

### گام ۲ — exchange اصلی و دو صف
1. Exchange جدید: Name = `demo.documents`، Type = `topic`، Durable.
2. صف جدید: Type = `Quorum`، Name = `demo.ledger.inbox`؛ در قسمت **Arguments** روی **Dead letter exchange** کلیک کنید تا `x-dead-letter-exchange` اضافه شود و مقدارش را `demo.dlx` بگذارید ← **Add queue**.
3. صف جدید: Type = `Quorum`، Name = `demo.document.replies`.
4. روی `demo.documents` کلیک کنید و دو Binding بسازید:
   - To queue = `demo.ledger.inbox`، Routing key = `document.temporary.*`
   - To queue = `demo.document.replies`، Routing key = `ledger.reply.#`

### گام ۳ — فرستادن پیام
روی `demo.documents` ← بخش **Publish message**:
1. Routing key = `document.temporary.created`، Delivery mode = `2 - Persistent`، Payload = `{"document_id":101,"branch_id":1}` ← **Publish message**. باید پیغام سبز «Message published» ببینید.
2. یک پیام دیگر با همان کلید و `{"document_id":102,"branch_id":1}`.
3. Routing key = `ledger.reply.done`، Payload = `{"document_id":101,"result":"done"}`.
4. Routing key = `nothing.matches`، Payload = `lost` ← پیغام زرد **«Message published, but not routed»**: هیچ اتصالی با این کلید جور نبود و پیام **دور ریخته شد**. (درس: همیشه مطمئن شوید پیام مسیر دارد.)

به زبانهٔ **Queues and Streams** بروید: `demo.ledger.inbox` = ۲ پیام، `demo.document.replies` = ۱ پیام، `demo.dead-letters` = ۰.

### گام ۴ — برداشتن پیام (نقش مصرف‌کننده)
روی `demo.ledger.inbox` کلیک کنید ← بخش **Get messages**:
1. Ack Mode = **Automatic ack**، Messages = `1` ← **Get Message(s)**. پیام ۱۰۱ نمایش داده و از صف حذف می‌شود (یعنی «پردازش کردم»).
2. دوباره، این بار Ack Mode = **Reject requeue false** ← پیام ۱۰۲ رد می‌شود و چون صف DLX دارد، **به `demo.dead-letters` می‌رود** (نه این‌که گم شود). در Queues ببینید: `demo.dead-letters` = ۱.
3. (اختیاری) Ack Mode = **Nack message requeue true** روی `demo.document.replies`: پیام را می‌بینید ولی **به صف برمی‌گردد** — همان اتفاقی که وقتی یک مصرف‌کننده قبل از Ack قطع شود می‌افتد.

### گام ۵ — ماندگاری
```bash
docker compose restart s-rabbitmq-service-fastapi
```
بعد از healthy شدن، پنل را تازه کنید (دوباره وارد شوید): صف‌ها و پیام‌های Persistent سر جایشان هستند.

### گام ۶ — نمودارها
در **Overview** و صفحهٔ هر صف، نمودار **Message rates** اوج‌های publish/deliver/ack کارهای بالا را نشان می‌دهد.

---

## ۶. تمرین ب — همان کارها از خط فرمان

برنامه‌ها همین کارها را با کد انجام می‌دهند؛ اینجا با ابزار `rabbitmqadmin` (نسخهٔ ۲، داخل خود کانتینر) و رابط HTTP. ابتدا یک میان‌بر بسازید (رمز را از `.env` بردارید):

```bash
R() { docker compose exec -T s-rabbitmq-service-fastapi \
      rabbitmqadmin -u airplus -p airplus-dev-only-change-me -V accounting "$@"; }
```

### ساختن توپولوژی
```bash
R declare exchange --name demo.dlx --type fanout --durable true
R declare queue    --name demo.dead-letters --type quorum --durable true
R declare binding  --source demo.dlx --destination-type queue --destination demo.dead-letters --routing-key ""

R declare exchange --name demo.documents --type topic --durable true
R declare queue    --name demo.ledger.inbox --type quorum --durable true \
                   --arguments '{"x-dead-letter-exchange":"demo.dlx"}'
R declare queue    --name demo.document.replies --type quorum --durable true
R declare binding  --source demo.documents --destination-type queue --destination demo.ledger.inbox     --routing-key "document.temporary.*"
R declare binding  --source demo.documents --destination-type queue --destination demo.document.replies --routing-key "ledger.reply.#"
```

### فرستادن پیام
```bash
R publish message -e demo.documents -k document.temporary.created \
  -m '{"document_id":101,"branch_id":1}' -p '{"delivery_mode":2,"content_type":"application/json"}'
# → Message published and routed successfully

R publish message -e demo.documents -k nothing.matches -m 'lost'
# → Message published but NOT routed
```

### دیدن و برداشتن
```bash
R list queues                                                    # همهٔ صف‌ها با تعداد پیام
R get messages -q demo.ledger.inbox -a ack_requeue_false         # = Automatic ack
R get messages -q demo.ledger.inbox -a reject_requeue_false      # = Reject requeue false → dead-letter
R list bindings
```

### رابط HTTP (همان چیزی که پنل و ابزارها زیر کاپوت صدا می‌زنند)
```bash
A="-s -u airplus:airplus-dev-only-change-me -H content-type:application/json"

curl $A http://localhost:45683/api/overview
curl $A http://localhost:45683/api/queues/accounting

curl $A -X POST http://localhost:45683/api/exchanges/accounting/demo.documents/publish \
  -d '{"properties":{"delivery_mode":2},"routing_key":"document.temporary.updated","payload":"{\"document_id\":103}","payload_encoding":"string"}'
# → {"routed":true}

curl $A -X POST http://localhost:45683/api/queues/accounting/demo.ledger.inbox/get \
  -d '{"count":1,"ackmode":"ack_requeue_false","encoding":"auto"}'
```

> `publish`/`get` از طریق HTTP و `rabbitmqadmin` فقط برای آزمایش است. برنامه‌های واقعی با پروتکل AMQP (پورت 5672) و یک کتابخانهٔ کلاینت وصل می‌شوند و به‌جای «برداشتن دستی»، **مشترک** صف می‌شوند (consume) تا پیام‌ها خودکار برسند.

### پاک‌کردن تمرین
```bash
for q in demo.ledger.inbox demo.document.replies demo.dead-letters; do R delete queue --name $q; done
for e in demo.documents demo.dlx; do R delete exchange --name $e; done
```
(یا از پنل: صفحهٔ هر صف/exchange ← بخش **Delete**.)

---

## ۷. عیب‌یابی رایج

| مشکل | علت و راه‌حل |
|---|---|
| پنل باز نمی‌شود | `docker compose ps` — آیا سرویس `healthy` است؟ اولین بار ۲۰–۳۰ ثانیه صبر کنید. پورت اشغال است؟ `RABBITMQ_MANAGEMENT_PORT` را در `.env` عوض کنید و دوباره `up -d` بزنید. |
| ورود با `guest` / `guest` رد می‌شود | عمدی است؛ کاربر `guest` ساخته نمی‌شود (و در هر حال فقط از داخل خود کانتینر کار می‌کند). با `RABBITMQ_USER` وارد شوید. |
| رمز را در `.env` عوض کردم ولی رمز جدید کار نمی‌کند | کاربر و vhost فقط **اولین بار** (روی حجم دادهٔ خالی) ساخته می‌شوند. یا رمز را از پنل عوض کنید (Admin ← Users ← کاربر ← Update this user)، یا در محیط توسعه دادهٔ آزمایشی را پاک کنید: `docker compose rm -sf s-rabbitmq-service-fastapi && docker volume rm <پروژه>_fastapi_rabbitmq_data` (همهٔ صف‌ها و پیام‌ها پاک می‌شوند). نام دقیق حجم: `docker volume ls`. |
| پیام فرستادم ولی به هیچ صفی نرسید | پیغام «published, but not routed»: کلید مسیریابی با هیچ Binding جور نیست، یا به exchange اشتباه/vhost اشتباه فرستاده‌اید. Bindingهای exchange را در پنل ببینید. |
| بعد از ری‌استارت پیام‌ها گم شدند | صف `Durable` نبوده یا پیام `Persistent` (delivery mode 2) نبوده. صف‌های quorum همیشه ماندگارند؛ پیام را هم Persistent بفرستید. |
| بعد از بازسازی کانتینر همهٔ داده‌ها رفت | اگر `hostname` سرویس عوض شود، RabbitMQ دادهٔ قبلی را (که با نام نود قبلی ذخیره شده) نمی‌بیند. `hostname: rabbitmq` در فایل ارکستراسیون نباید تغییر کند. |
| پیام در Unacked گیر کرده | مصرف‌کننده پیام را برداشته ولی Ack نداده؛ وقتی اتصالش بسته شود، پیام خودکار به Ready برمی‌گردد. |

---

## ۸. صف‌های واقعی سیستم و کارگرها

از این به بعد، خود سیستم (نه تمرین‌ها) این‌ها را **خودکار** در vhost ‏`accounting` می‌سازد — دستی از پنل نسازید و پاک نکنید:

| نام | نوع | کاربرد |
|---|---|---|
| `acc.commands` | exchange (direct) | فرمان‌ها؛ کلید = سرویس گیرنده |
| `acc.replies` | exchange (direct) | پاسخ فرمان‌ها؛ کلید = سرویس فرستندهٔ فرمان |
| `acc.events` | exchange (topic) | رویدادها؛ کلید = نوع پیام |
| `acc.dlx` | exchange (direct) | پیام‌های مرده |
| `ledger.commands` | صف quorum | فرمان‌های دفترداری (فقط یک مصرف‌کنندهٔ فعال) |
| `diagnostics.replies` | صف quorum | پاسخ‌های پینگ آزمایشی |
| `<صف>.retry` | صف classic | انتظار برای تلاش مجدد با تأخیر؛ بعد خودکار به صف اصلی برمی‌گردد |
| `<صف>.dlq` | صف quorum | پیام‌هایی که پردازش نشدند (خراب، ناشناخته، یا بعد از چند تلاش ناموفق) |

دو کانتینر کارگر کنار سرویس حسابداری اجرا می‌شوند:
- `s-messaging-relay-fastapi` — پیام‌های «صندوق خروجی» پایگاه داده را به RabbitMQ می‌فرستد.
- `s-messaging-consumer-fastapi` — پیام‌های صف‌ها را برمی‌دارد و پردازش می‌کند.

```bash
docker compose logs -f s-messaging-relay-fastapi s-messaging-consumer-fastapi
docker compose restart s-messaging-relay-fastapi s-messaging-consumer-fastapi   # بعد از تغییر کد (خودکار ری‌لود نمی‌شوند)
```

**آزمایش از کنسول تست:** زبانهٔ «پیام‌رسانی» ← «ارسال پینگ». حالت `ok` پاسخ «انجام شد» می‌گیرد، `reject` پاسخ «رد شد»، و `fail` بعد از ۳ تلاش مجدد (در پنل: `ledger.commands.retry`) به `ledger.commands.dlq` می‌رود. «ارسال تکراری» نشان می‌دهد همان پیام دوباره اجرا نمی‌شود. با `docker compose stop s-messaging-consumer-fastapi` و ارسال پینگ، پیام را در صف `ledger.commands` پنل می‌بینید؛ با `start` پاسخ می‌گیرد.

