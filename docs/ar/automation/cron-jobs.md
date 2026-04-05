---
read_when:
    - عند جدولة المهام الخلفية أو عمليات الإيقاظ
    - عند توصيل المشغلات الخارجية (webhooks وGmail) إلى OpenClaw
    - عند اتخاذ قرار بين heartbeat وcron للمهام المجدولة
summary: المهام المجدولة وwebhooks ومشغلات Gmail PubSub لجدولة Gateway
title: المهام المجدولة
x-i18n:
    generated_at: "2026-04-05T12:34:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 43b906914461aba9af327e7e8c22aa856f65802ec2da37ed0c4f872d229cfde6
    source_path: automation/cron-jobs.md
    workflow: 15
---

# المهام المجدولة (Cron)

Cron هو المجدول المدمج في Gateway. فهو يحتفظ بالمهام، ويوقظ الوكيل في الوقت المناسب، ويمكنه إعادة تسليم المخرجات إلى قناة دردشة أو نقطة نهاية webhook.

## بدء سريع

```bash
# Add a one-shot reminder
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the cron docs draft" \
  --wake now \
  --delete-after-run

# Check your jobs
openclaw cron list

# See run history
openclaw cron runs --id <job-id>
```

## كيف يعمل cron

- يعمل Cron **داخل** عملية Gateway (وليس داخل النموذج).
- تُحفَظ المهام في `~/.openclaw/cron/jobs.json` بحيث لا تؤدي عمليات إعادة التشغيل إلى فقدان الجداول.
- تنشئ جميع عمليات تنفيذ cron سجلات [المهام الخلفية](/automation/tasks).
- تُحذف المهام ذات التشغيل الواحد (`--at`) تلقائيًا بعد النجاح افتراضيًا.
- تحاول عمليات cron المعزولة، بأفضل جهد ممكن، إغلاق علامات تبويب/عمليات المتصفح المتتبعة الخاصة بجلسة `cron:<jobId>` عند اكتمال التشغيل، حتى لا تترك أتمتة المتصفح المفصولة عمليات يتيمة وراءها.
- تحمي عمليات cron المعزولة أيضًا من ردود الإقرار القديمة. إذا كانت
  النتيجة الأولى مجرد تحديث حالة مؤقت (`on it` و`pulling everything
together` وتلميحات مشابهة) ولم يعد أي تشغيل تابع منحدر لا يزال
  مسؤولًا عن الإجابة النهائية، فإن OpenClaw يعيد المطالبة مرة واحدة للحصول على
  النتيجة الفعلية قبل التسليم.

تسوية المهام لـ cron مملوكة لوقت التشغيل: تبقى مهمة cron النشطة قيد التشغيل ما دام
وقت تشغيل cron لا يزال يتتبع تلك المهمة على أنها قيد التشغيل، حتى إذا كان صف جلسة فرعي قديم لا يزال موجودًا.
وبمجرد أن يتوقف وقت التشغيل عن امتلاك المهمة وتنتهي مهلة السماح البالغة 5 دقائق، يمكن للصيانة
وضع علامة `lost` على المهمة.

## أنواع الجداول

| النوع    | علم CLI  | الوصف                                                    |
| ------- | --------- | ------------------------------------------------------- |
| `at`    | `--at`    | طابع زمني لتشغيل واحد (ISO 8601 أو قيمة نسبية مثل `20m`) |
| `every` | `--every` | فاصل زمني ثابت                                           |
| `cron`  | `--cron`  | تعبير cron من 5 أو 6 حقول مع `--tz` اختياري              |

تُعامل الطوابع الزمنية التي لا تحتوي على منطقة زمنية على أنها UTC. أضف `--tz America/New_York` للجدولة حسب التوقيت المحلي الفعلي.

تُوزَّع تعبيرات التكرار أعلى كل ساعة تلقائيًا بفارق يصل إلى 5 دقائق لتقليل ارتفاعات الحمل. استخدم `--exact` لفرض توقيت دقيق أو `--stagger 30s` لنافذة صريحة.

## أنماط التنفيذ

| النمط           | قيمة `--session`   | يعمل في                  | الأنسب لـ                      |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| الجلسة الرئيسية    | `main`              | دورة heartbeat التالية      | التذكيرات، وأحداث النظام        |
| معزول          | `isolated`          | `cron:<jobId>` مخصصة      | التقارير، والمهام الخلفية       |
| الجلسة الحالية | `current`           | يُربط وقت الإنشاء         | الأعمال الدورية المعتمدة على السياق |
| جلسة مخصصة  | `session:custom-id` | جلسة مسماة دائمة         | سير العمل الذي يعتمد على السجل   |

تضيف مهام **الجلسة الرئيسية** حدث نظام إلى قائمة الانتظار، ويمكنها اختياريًا إيقاظ heartbeat (`--wake now` أو `--wake next-heartbeat`). تعمل المهام **المعزولة** بدورة وكيل مخصصة مع جلسة جديدة. تحتفظ **الجلسات المخصصة** (`session:xxx`) بالسياق عبر مرات التشغيل، مما يتيح سير عمل مثل الاجتماعات اليومية المختصرة التي تعتمد على الملخصات السابقة.

بالنسبة للمهام المعزولة، يتضمن التفكيك في وقت التشغيل الآن تنظيف المتصفح، بأفضل جهد، لتلك الجلسة الخاصة بـ cron. ويتم تجاهل إخفاقات التنظيف بحيث تبقى نتيجة cron الفعلية هي المعتمدة.

عندما تقوم عمليات cron المعزولة بتنسيق وكلاء فرعيين، يفضّل التسليم أيضًا
المخرجات النهائية التابعة بدلًا من النص المؤقت القديم للأصل. وإذا كانت العمليات التابعة لا تزال
قيد التشغيل، فإن OpenClaw يحجب هذا التحديث الجزئي من الأصل بدلًا من إعلانه.

### خيارات الحمولة للمهام المعزولة

- `--message`: نص المطالبة (مطلوب للمهام المعزولة)
- `--model` / `--thinking`: تجاوزات النموذج ومستوى التفكير
- `--light-context`: تخطي حقن ملف تهيئة مساحة العمل
- `--tools exec,read`: تقييد الأدوات التي يمكن للمهمة استخدامها

يستخدم `--model` النموذج المسموح المحدد لتلك المهمة. إذا كان النموذج المطلوب
غير مسموح به، يسجل cron تحذيرًا ويعود إلى اختيار نموذج الوكيل/الافتراضي
لتلك المهمة بدلًا من ذلك. وتظل سلاسل التراجع المهيأة مطبقة، لكن تجاوز
النموذج العادي من دون قائمة تراجع صريحة لكل مهمة لم يعد يضيف النموذج
الأساسي للوكيل كهدف إعادة محاولة إضافي مخفي.

أولوية اختيار النموذج للمهام المعزولة هي:

1. تجاوز نموذج hook الخاص بـ Gmail (عندما يكون التشغيل قادمًا من Gmail ويكون هذا التجاوز مسموحًا)
2. `model` في حمولة كل مهمة
3. تجاوز النموذج المخزن لجلسة cron
4. اختيار نموذج الوكيل/الافتراضي

يتبع الوضع السريع اختيار التشغيل المباشر الذي تم حله أيضًا. إذا كان إعداد النموذج المحدد
يحتوي على `params.fastMode`، فإن cron المعزول يستخدم ذلك افتراضيًا. ويظل تجاوز
`fastMode` المخزن للجلسة متقدمًا على الإعداد في كلا الاتجاهين.

إذا واجه تشغيل معزول تسليمًا مباشرًا لتبديل النموذج، يعيد cron المحاولة باستخدام
المزوّد/النموذج المُبدَّل ويحفظ هذا الاختيار المباشر قبل إعادة المحاولة. وعندما
يحمل التبديل أيضًا ملف تعريف مصادقة جديدًا، فإن cron يحفظ تجاوز ملف تعريف المصادقة هذا أيضًا.
إعادات المحاولة محدودة: بعد المحاولة الأولية بالإضافة إلى محاولتي تبديل،
يُجهِض cron بدلًا من الدخول في حلقة لا نهائية.

## التسليم والمخرجات

| الوضع       | ما الذي يحدث                                             |
| ---------- | -------------------------------------------------------- |
| `announce` | تسليم الملخص إلى القناة المستهدفة (الافتراضي للمهام المعزولة) |
| `webhook`  | إرسال حمولة حدث الاكتمال إلى URL عبر POST                |
| `none`     | داخلي فقط، بلا تسليم                                      |

استخدم `--announce --channel telegram --to "-1001234567890"` للتسليم إلى قناة. بالنسبة لموضوعات منتدى Telegram، استخدم `-1001234567890:topic:123`. يجب أن تستخدم أهداف Slack/Discord/Mattermost بادئات صريحة (`channel:<id>` و`user:<id>`).

بالنسبة للمهام المعزولة المملوكة لـ cron، يمتلك المشغّل مسار التسليم النهائي. تتم
مطالبة الوكيل بإرجاع ملخص نصي عادي، ثم يُرسل هذا الملخص عبر
`announce` أو `webhook` أو يُحتفظ به داخليًا عند `none`. لا يعيد `--no-deliver`
التسليم إلى الوكيل؛ بل يُبقي التشغيل داخليًا.

إذا كانت المهمة الأصلية تنص صراحةً على إرسال رسالة إلى مستلم خارجي ما، فيجب على
الوكيل أن يذكر في مخرجاته إلى من/أين ينبغي أن تذهب تلك الرسالة بدلًا من
محاولة إرسالها مباشرةً.

تتبع إشعارات الفشل مسار وجهة منفصلًا:

- يعيّن `cron.failureDestination` القيمة الافتراضية العامة لإشعارات الفشل.
- يتجاوز `job.delivery.failureDestination` ذلك لكل مهمة.
- إذا لم يُضبط أي منهما وكانت المهمة تسلّم بالفعل عبر `announce`، فإن إشعارات الفشل تعود الآن إلى هدف announce الأساسي هذا.
- لا يكون `delivery.failureDestination` مدعومًا إلا في المهام ذات `sessionTarget="isolated"` ما لم يكن وضع التسليم الأساسي هو `webhook`.

## أمثلة CLI

تذكير بتشغيل واحد (الجلسة الرئيسية):

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

مهمة معزولة متكررة مع تسليم:

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

مهمة معزولة مع تجاوز للنموذج والتفكير:

```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly deep analysis of project progress." \
  --model "opus" \
  --thinking high \
  --announce
```

## Webhooks

يمكن لـ Gateway كشف نقاط نهاية HTTP webhook للمشغلات الخارجية. فعّل ذلك في الإعدادات:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### المصادقة

يجب أن يتضمن كل طلب رمز hook عبر ترويسة:

- `Authorization: Bearer <token>` (موصى به)
- `x-openclaw-token: <token>`

يتم رفض الرموز في سلسلة الاستعلام.

### POST /hooks/wake

أضف حدث نظام إلى قائمة انتظار الجلسة الرئيسية:

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"New email received","mode":"now"}'
```

- `text` (مطلوب): وصف الحدث
- `mode` (اختياري): `now` (الافتراضي) أو `next-heartbeat`

### POST /hooks/agent

شغّل دورة وكيل معزولة:

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.4-mini"}'
```

الحقول: `message` (مطلوب)، `name`، `agentId`، `wakeMode`، `deliver`، `channel`، `to`، `model`، `thinking`، `timeoutSeconds`.

### Hooks المعينة (POST /hooks/\<name\>)

تُحل أسماء hooks المخصصة عبر `hooks.mappings` في الإعدادات. ويمكن للتعيينات تحويل الحمولات العشوائية إلى إجراءات `wake` أو `agent` باستخدام قوالب أو تحويلات برمجية.

### الأمان

- أبقِ نقاط نهاية hook خلف local loopback أو tailnet أو reverse proxy موثوق.
- استخدم رمز hook مخصصًا؛ لا تعِد استخدام رموز مصادقة gateway.
- أبقِ `hooks.path` على مسار فرعي مخصص؛ يتم رفض `/`.
- اضبط `hooks.allowedAgentIds` لتقييد التوجيه الصريح لـ `agentId`.
- أبقِ `hooks.allowRequestSessionKey=false` ما لم تكن تحتاج إلى جلسات يحددها المتصل.
- إذا فعّلت `hooks.allowRequestSessionKey`، فاضبط أيضًا `hooks.allowedSessionKeyPrefixes` لتقييد أشكال مفاتيح الجلسات المسموح بها.
- تُغلَّف حمولات hook بحدود أمان افتراضيًا.

## تكامل Gmail PubSub

اربط مشغلات صندوق وارد Gmail بـ OpenClaw عبر Google PubSub.

**المتطلبات المسبقة**: CLI `gcloud`، و`gog` (`gogcli`)، وhooks مفعلة في OpenClaw، وTailscale لنقطة النهاية العامة عبر HTTPS.

### إعداد المعالج (موصى به)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

يكتب هذا إعداد `hooks.gmail`، ويفعّل الإعداد المسبق لـ Gmail، ويستخدم Tailscale Funnel لنقطة نهاية الدفع.

### التشغيل التلقائي لـ Gateway

عندما يكون `hooks.enabled=true` و`hooks.gmail.account` مضبوطًا، يبدأ Gateway تشغيل `gog gmail watch serve` عند الإقلاع ويجدد المراقبة تلقائيًا. اضبط `OPENCLAW_SKIP_GMAIL_WATCHER=1` لإلغاء الاشتراك.

### إعداد يدوي لمرة واحدة

1. حدّد مشروع GCP الذي يملك عميل OAuth المستخدم بواسطة `gog`:

```bash
gcloud auth login
gcloud config set project <project-id>
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

2. أنشئ topic وامنح Gmail صلاحية دفع النشر:

```bash
gcloud pubsub topics create gog-gmail-watch
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

3. ابدأ المراقبة:

```bash
gog gmail watch start \
  --account openclaw@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

### تجاوز نموذج Gmail

```json5
{
  hooks: {
    gmail: {
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

## إدارة المهام

```bash
# List all jobs
openclaw cron list

# Edit a job
openclaw cron edit <jobId> --message "Updated prompt" --model "opus"

# Force run a job now
openclaw cron run <jobId>

# Run only if due
openclaw cron run <jobId> --due

# View run history
openclaw cron runs --id <jobId> --limit 50

# Delete a job
openclaw cron remove <jobId>

# Agent selection (multi-agent setups)
openclaw cron add --name "Ops sweep" --cron "0 6 * * *" --session isolated --message "Check ops queue" --agent ops
openclaw cron edit <jobId> --clear-agent
```

ملاحظة حول تجاوز النموذج:

- يغيّر `openclaw cron add|edit --model ...` النموذج المحدد للمهمة.
- إذا كان النموذج مسموحًا، يصل هذا المزوّد/النموذج المحدد نفسه إلى تشغيل الوكيل المعزول.
- إذا لم يكن مسموحًا، يحذّر cron ويعود إلى اختيار نموذج الوكيل/الافتراضي للمهمة.
- تظل سلاسل التراجع المهيأة مطبقة، لكن تجاوز `--model` العادي من دون قائمة تراجع صريحة لكل مهمة لم يعد ينتقل إلى النموذج الأساسي للوكيل باعتباره هدف إعادة محاولة إضافيًا صامتًا.

## الإعدادات

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1,
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
    runLog: { maxBytes: "2mb", keepLines: 2000 },
  },
}
```

تعطيل cron: `cron.enabled: false` أو `OPENCLAW_SKIP_CRON=1`.

**إعادة محاولة التشغيل الواحد**: تتم إعادة المحاولة للأخطاء المؤقتة (حد المعدل، والتحميل الزائد، والشبكة، وخطأ الخادم) حتى 3 مرات مع تراجع أُسّي. أما الأخطاء الدائمة فتؤدي إلى التعطيل فورًا.

**إعادة محاولة المهام المتكررة**: تراجع أُسّي (من 30 ثانية إلى 60 دقيقة) بين المحاولات. ويُعاد ضبط التراجع بعد التشغيل الناجح التالي.

**الصيانة**: يقوم `cron.sessionRetention` (الافتراضي `24h`) بتقليم إدخالات جلسات التشغيل المعزولة. كما يقوم `cron.runLog.maxBytes` / `cron.runLog.keepLines` بتقليم ملفات سجل التشغيل تلقائيًا.

## استكشاف الأخطاء وإصلاحها

### تسلسل الأوامر

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

### Cron لا يعمل

- تحقّق من `cron.enabled` ومتغير البيئة `OPENCLAW_SKIP_CRON`.
- أكّد أن Gateway يعمل بشكل مستمر.
- بالنسبة لجداول `cron`، تحقّق من المنطقة الزمنية (`--tz`) مقارنة بالمنطقة الزمنية للمضيف.
- تعني `reason: not-due` في مخرجات التشغيل أنه تم التحقق من التشغيل اليدوي باستخدام `openclaw cron run <jobId> --due` وأن المهمة لم يحن موعدها بعد.

### تم تشغيل Cron لكن لم يحدث تسليم

- يعني وضع التسليم `none` أنه لا يُتوقع إرسال أي رسالة خارجية.
- يعني غياب/عدم صحة هدف التسليم (`channel`/`to`) أنه تم تخطي الإرسال الخارجي.
- تعني أخطاء مصادقة القناة (`unauthorized` و`Forbidden`) أن التسليم تم حجبه بواسطة بيانات الاعتماد.
- إذا أعاد التشغيل المعزول فقط الرمز الصامت (`NO_REPLY` / `no_reply`)،
  فإن OpenClaw يحجب التسليم الخارجي المباشر ويحجب أيضًا مسار
  الملخص الاحتياطي الموضوع في قائمة الانتظار، لذلك لا يُنشر أي شيء إلى الدردشة.
- بالنسبة للمهام المعزولة المملوكة لـ cron، لا تتوقع من الوكيل استخدام أداة الرسائل
  كحل احتياطي. فالمشغّل يملك التسليم النهائي؛ ويبقي `--no-deliver` التشغيل
  داخليًا بدلًا من السماح بإرسال مباشر.

### ملاحظات مهمة حول المنطقة الزمنية

- يستخدم Cron بدون `--tz` المنطقة الزمنية لمضيف gateway.
- تُعامل جداول `at` التي لا تحتوي على منطقة زمنية على أنها UTC.
- يستخدم `activeHours` في heartbeat تحليل المنطقة الزمنية المهيأ.

## ذو صلة

- [الأتمتة والمهام](/automation) — لمحة سريعة عن جميع آليات الأتمتة
- [المهام الخلفية](/automation/tasks) — سجل المهام لتنفيذات cron
- [Heartbeat](/gateway/heartbeat) — دورات الجلسة الرئيسية الدورية
- [المنطقة الزمنية](/concepts/timezone) — إعداد المنطقة الزمنية
