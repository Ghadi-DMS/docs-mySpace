---
read_when:
    - ضبط وتيرة heartbeat أو الرسائل
    - اتخاذ قرار بين heartbeat وcron للمهام المجدولة
summary: رسائل استطلاع Heartbeat وقواعد الإشعارات
title: Heartbeat
x-i18n:
    generated_at: "2026-04-05T12:43:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: f417b0d4453bed9022144d364521a59dec919d44cca8f00f0def005cd38b146f
    source_path: gateway/heartbeat.md
    workflow: 15
---

# Heartbeat ‏(Gateway)

> **Heartbeat أم Cron؟** راجع [Automation & Tasks](/automation) للحصول على إرشادات حول وقت استخدام كل منهما.

يشغّل Heartbeat **أدوار وكيل دورية** في الجلسة الرئيسية حتى يتمكن النموذج من
إظهار أي شيء يحتاج إلى انتباه من دون إزعاجك برسائل متكررة.

يُعد Heartbeat دورًا مجدولًا في الجلسة الرئيسية — وهو **لا** ينشئ سجلات [background task](/automation/tasks).
فسجلات المهام مخصصة للعمل المنفصل (تشغيلات ACP، والوكلاء الفرعيون، ومهام cron المعزولة).

استكشاف الأخطاء وإصلاحها: [Scheduled Tasks](/automation/cron-jobs#troubleshooting)

## البدء السريع (للمبتدئين)

1. اترك heartbeats مفعلة (الافتراضي هو `30m`، أو `1h` لمصادقة Anthropic OAuth/token، بما في ذلك إعادة استخدام Claude CLI) أو عيّن وتيرتك الخاصة.
2. أنشئ قائمة تحقق صغيرة في `HEARTBEAT.md` أو كتلة `tasks:` في مساحة عمل الوكيل (اختياري لكنه موصى به).
3. قرر أين يجب أن تذهب رسائل heartbeat ‏(`target: "none"` هو الافتراضي؛ اضبط `target: "last"` للتوجيه إلى آخر جهة اتصال).
4. اختياري: فعّل تسليم reasoning الخاص بـ heartbeat من أجل الشفافية.
5. اختياري: استخدم سياق bootstrap خفيفًا إذا كانت تشغيلات heartbeat تحتاج فقط إلى `HEARTBEAT.md`.
6. اختياري: فعّل الجلسات المعزولة لتجنب إرسال السجل الكامل للمحادثة في كل heartbeat.
7. اختياري: قيد heartbeats بالساعات النشطة (التوقيت المحلي).

مثال على التكوين:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // تسليم صريح إلى آخر جهة اتصال (الافتراضي هو "none")
        directPolicy: "allow", // الافتراضي: السماح بأهداف direct/DM؛ اضبط "block" لمنعها
        lightContext: true, // اختياري: حقن HEARTBEAT.md فقط من ملفات bootstrap
        isolatedSession: true, // اختياري: جلسة جديدة لكل تشغيل (من دون سجل المحادثة)
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // اختياري: إرسال رسالة `Reasoning:` منفصلة أيضًا
      },
    },
  },
}
```

## القيم الافتراضية

- الفاصل الزمني: `30m` ‏(أو `1h` عندما يكون وضع مصادقة Anthropic OAuth/token هو وضع المصادقة المكتشف، بما في ذلك إعادة استخدام Claude CLI). اضبط `agents.defaults.heartbeat.every` أو `agents.list[].heartbeat.every` لكل وكيل؛ واستخدم `0m` للتعطيل.
- نص المطالبة (قابل للتكوين عبر `agents.defaults.heartbeat.prompt`):
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- تُرسل مطالبة heartbeat **حرفيًا** كرسالة مستخدم. وتتضمن مطالبة
  النظام قسم “Heartbeat” ويتم تعليم التشغيل داخليًا.
- يتم التحقق من الساعات النشطة (`heartbeat.activeHours`) في المنطقة الزمنية المكوّنة.
  وخارج هذه النافذة، يتم تخطي heartbeats حتى التوقيت التالي داخل النافذة.

## الغرض من مطالبة heartbeat

المطالبة الافتراضية واسعة عمدًا:

- **المهام الخلفية**: عبارة “Consider outstanding tasks” تدفع الوكيل إلى مراجعة
  المتابعات (البريد الوارد، والتقويم، والتذكيرات، والعمل في قائمة الانتظار) وإظهار أي شيء عاجل.
- **الاطمئنان على الإنسان**: عبارة “Checkup sometimes on your human during day time” تدفع إلى
  رسالة خفيفة أحيانًا مثل “هل تحتاج شيئًا؟”، لكنها تتجنب الإزعاج الليلي
  باستخدام منطقتك الزمنية المحلية المكوّنة (راجع [/concepts/timezone](/concepts/timezone)).

يمكن لـ Heartbeat التفاعل مع [background tasks](/automation/tasks) المكتملة، لكن تشغيل heartbeat نفسه لا ينشئ سجل مهمة.

إذا كنت تريد من heartbeat أن يقوم بشيء محدد جدًا (مثل “check Gmail PubSub
stats” أو “verify gateway health”)، فاضبط `agents.defaults.heartbeat.prompt` (أو
`agents.list[].heartbeat.prompt`) على نص مخصص (يُرسل حرفيًا).

## عقد الاستجابة

- إذا لم يكن هناك ما يحتاج إلى انتباه، فردّ بـ **`HEARTBEAT_OK`**.
- أثناء تشغيلات heartbeat، يعامل OpenClaw الرمز `HEARTBEAT_OK` كإقرار عندما يظهر
  في **بداية أو نهاية** الرد. تتم إزالة الرمز ويتم إسقاط الرد إذا كان
  المحتوى المتبقي **≤ `ackMaxChars`** ‏(الافتراضي: 300).
- إذا ظهر `HEARTBEAT_OK` في **وسط** الرد، فلا تتم معاملته
  بشكل خاص.
- بالنسبة إلى التنبيهات، **لا** تضمن `HEARTBEAT_OK`؛ وأعد نص التنبيه فقط.

خارج heartbeats، تتم إزالة أي `HEARTBEAT_OK` عارض في بداية/نهاية الرسالة
ويتم تسجيله؛ وتُسقط الرسالة إذا كانت تحتوي فقط على `HEARTBEAT_OK`.

## التكوين

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // الافتراضي: 30m (0m يعطّل)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // الافتراضي: false (تسليم رسالة Reasoning: منفصلة عندما تكون متاحة)
        lightContext: false, // الافتراضي: false؛ قيمة true تُبقي HEARTBEAT.md فقط من ملفات bootstrap لمساحة العمل
        isolatedSession: false, // الافتراضي: false؛ قيمة true تشغّل كل heartbeat في جلسة جديدة (من دون سجل محادثة)
        target: "last", // الافتراضي: none | الخيارات: last | none | <channel id> (أساسي أو plugin، مثل "bluebubbles")
        to: "+15551234567", // تجاوز اختياري خاص بالقناة
        accountId: "ops-bot", // معرّف قناة متعدد الحسابات اختياري
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // الحد الأقصى للأحرف المسموح بها بعد HEARTBEAT_OK
      },
    },
  },
}
```

### النطاق والأولوية

- يعيّن `agents.defaults.heartbeat` سلوك heartbeat العام.
- يتم دمج `agents.list[].heartbeat` فوقه؛ وإذا كان لأي وكيل كتلة `heartbeat`، فإن **هؤلاء الوكلاء فقط** هم الذين يشغّلون heartbeats.
- يضبط `channels.defaults.heartbeat` القيم الافتراضية للظهور لكل القنوات.
- يتجاوز `channels.<channel>.heartbeat` القيم الافتراضية للقناة.
- يتجاوز `channels.<channel>.accounts.<id>.heartbeat` ‏(للقنوات متعددة الحسابات) الإعدادات لكل قناة.

### Heartbeats لكل وكيل

إذا تضمّن أي إدخال `agents.list[]` كتلة `heartbeat`، فإن **هؤلاء الوكلاء فقط**
هم من يشغّلون heartbeats. ويتم دمج الكتلة لكل وكيل فوق `agents.defaults.heartbeat`
(بحيث يمكنك تعيين القيم المشتركة مرة واحدة ثم تجاوزها لكل وكيل).

مثال: وكيلان، والوكيل الثاني فقط هو الذي يشغّل heartbeats.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // تسليم صريح إلى آخر جهة اتصال (الافتراضي هو "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### مثال على الساعات النشطة

قيد heartbeats بساعات العمل في منطقة زمنية محددة:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // تسليم صريح إلى آخر جهة اتصال (الافتراضي هو "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // اختياري؛ يستخدم userTimezone إن كانت معيّنة، وإلا المنطقة الزمنية للمضيف
        },
      },
    },
  },
}
```

خارج هذه النافذة (قبل 9 صباحًا أو بعد 10 مساءً بتوقيت الشرق)، يتم تخطي heartbeats. وسيعمل التوقيت المجدول التالي داخل النافذة بشكل طبيعي.

### إعداد 24/7

إذا كنت تريد تشغيل heartbeats طوال اليوم، فاستخدم أحد هذين النمطين:

- احذف `activeHours` بالكامل (من دون تقييد بنافذة زمنية؛ وهذا هو السلوك الافتراضي).
- عيّن نافذة يوم كامل: `activeHours: { start: "00:00", end: "24:00" }`.

لا تعيّن الوقت نفسه لكل من `start` و`end` (مثلًا `08:00` إلى `08:00`).
تُعامل هذه الحالة على أنها نافذة بعرض صفري، لذلك يتم دائمًا تخطي heartbeats.

### مثال متعدد الحسابات

استخدم `accountId` لاستهداف حساب محدد على القنوات متعددة الحسابات مثل Telegram:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // اختياري: التوجيه إلى topic/thread محدد
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### ملاحظات الحقول

- `every`: الفاصل الزمني لـ heartbeat ‏(سلسلة مدة؛ وحدة القياس الافتراضية = دقائق).
- `model`: تجاوز اختياري للنموذج في تشغيلات heartbeat ‏(`provider/model`).
- `includeReasoning`: عند التمكين، يسلّم أيضًا رسالة `Reasoning:` منفصلة عندما تكون متاحة (بنفس شكل `/reasoning on`).
- `lightContext`: عندما تكون القيمة true، تستخدم تشغيلات heartbeat سياق bootstrap خفيفًا وتحتفظ فقط بـ `HEARTBEAT.md` من ملفات bootstrap الخاصة بمساحة العمل.
- `isolatedSession`: عندما تكون القيمة true، تعمل كل heartbeat في جلسة جديدة من دون سجل محادثة سابق. وتستخدم نفس نمط العزل الذي يستخدمه cron مع `sessionTarget: "isolated"`. وهذا يقلل بشكل كبير تكلفة الرموز لكل heartbeat. اجمعها مع `lightContext: true` لتحقيق أقصى توفير. ويظل توجيه التسليم يستخدم سياق الجلسة الرئيسية.
- `session`: مفتاح جلسة اختياري لتشغيلات heartbeat.
  - `main` ‏(الافتراضي): الجلسة الرئيسية للوكيل.
  - مفتاح جلسة صريح (انسخه من `openclaw sessions --json` أو من [sessions CLI](/cli/sessions)).
  - تنسيقات مفاتيح الجلسات: راجع [Sessions](/concepts/session) و[Groups](/channels/groups).
- `target`:
  - `last`: التسليم إلى آخر قناة خارجية مستخدمة.
  - قناة صريحة: أي قناة مكوّنة أو معرّف plugin، مثل `discord` أو `matrix` أو `telegram` أو `whatsapp`.
  - `none` ‏(الافتراضي): تشغيل heartbeat لكن **من دون تسليم** خارجي.
- `directPolicy`: يتحكم في سلوك التسليم المباشر/DM:
  - `allow` ‏(الافتراضي): السماح بتسليم heartbeat إلى direct/DM.
  - `block`: منع التسليم إلى direct/DM ‏(`reason=dm-blocked`).
- `to`: تجاوز اختياري للمستلم (معرّف خاص بالقناة، مثل E.164 لـ WhatsApp أو معرّف دردشة Telegram). وبالنسبة إلى Telegram topics/threads، استخدم `<chatId>:topic:<messageThreadId>`.
- `accountId`: معرّف حساب اختياري للقنوات متعددة الحسابات. وعندما تكون `target: "last"`، ينطبق معرّف الحساب على آخر قناة محلولة إذا كانت تدعم الحسابات؛ وإلا فيتم تجاهله. وإذا لم يطابق معرّف الحساب حسابًا مكوّنًا للقناة المحلولة، يتم تخطي التسليم.
- `prompt`: يتجاوز نص المطالبة الافتراضي (ولا يتم دمجه).
- `ackMaxChars`: الحد الأقصى للأحرف المسموح بها بعد `HEARTBEAT_OK` قبل التسليم.
- `suppressToolErrorWarnings`: عندما تكون القيمة true، يتم إخفاء حمولات تحذير أخطاء الأدوات أثناء تشغيلات heartbeat.
- `activeHours`: تقييد تشغيلات heartbeat بنافذة زمنية. كائن يحتوي على `start` ‏(HH:MM، شامل؛ استخدم `00:00` لبداية اليوم)، و`end` ‏(HH:MM غير شامل؛ ويُسمح بـ `24:00` لنهاية اليوم)، و`timezone` اختيارية.
  - إذا حُذفت أو كانت `"user"`: تستخدم `agents.defaults.userTimezone` إن كانت معيّنة، وإلا تعود إلى المنطقة الزمنية لنظام المضيف.
  - `"local"`: تستخدم دائمًا المنطقة الزمنية لنظام المضيف.
  - أي معرّف IANA ‏(مثل `America/New_York`): يُستخدم مباشرة؛ وإذا كان غير صالح، يعود إلى سلوك `"user"` أعلاه.
  - يجب ألا يكون `start` و`end` متساويين لنافذة نشطة؛ فالقيم المتساوية تُعامل على أنها عرض صفري (دائمًا خارج النافذة).
  - خارج النافذة النشطة، يتم تخطي heartbeats حتى التوقيت التالي داخل النافذة.

## سلوك التسليم

- تعمل Heartbeats في الجلسة الرئيسية للوكيل افتراضيًا (`agent:<id>:<mainKey>`)،
  أو `global` عندما تكون `session.scope = "global"`. اضبط `session` للتجاوز إلى
  جلسة قناة محددة (Discord/WhatsApp/إلخ).
- يؤثر `session` في سياق التشغيل فقط؛ أما التسليم فيتحكم فيه `target` و`to`.
- للتسليم إلى قناة/مستلم محدد، اضبط `target` + `to`. ومع
  `target: "last"`، يستخدم التسليم آخر قناة خارجية لتلك الجلسة.
- تسمح عمليات تسليم Heartbeat بأهداف direct/DM افتراضيًا. اضبط `directPolicy: "block"` لمنع الإرسال إلى الأهداف المباشرة مع الاستمرار في تشغيل دور heartbeat.
- إذا كانت الطوابير الرئيسية مشغولة، يتم تخطي heartbeat وإعادة المحاولة لاحقًا.
- إذا جرى حل `target` إلى عدم وجود وجهة خارجية، يظل التشغيل يحدث ولكن لا
  يتم إرسال أي رسالة صادرة.
- إذا كانت `showOk` و`showAlerts` و`useIndicator` كلها معطلة، فيتم تخطي التشغيل مسبقًا على أنه `reason=alerts-disabled`.
- إذا كان تسليم التنبيهات فقط معطلًا، فلا يزال بإمكان OpenClaw تشغيل heartbeat، وتحديث الطوابع الزمنية للمهام المستحقة، واستعادة الطابع الزمني للخمول للجلسة، ومنع حمولة التنبيه الخارجية.
- الردود الخاصة بـ heartbeat فقط لا **تبقي الجلسة نشطة**؛ إذ تتم استعادة قيمة `updatedAt`
  الأخيرة بحيث يعمل انتهاء الصلاحية عند الخمول بشكل طبيعي.
- يمكن أن تقوم [background tasks](/automation/tasks) المنفصلة بصف نظام حدث وتوقظ heartbeat عندما ينبغي أن تلاحظ الجلسة الرئيسية شيئًا بسرعة. ولا يجعل هذا الإيقاظ تشغيل heartbeat مهمة خلفية.

## عناصر التحكم في الظهور

افتراضيًا، يتم إخفاء إشعارات `HEARTBEAT_OK` بينما يتم
تسليم محتوى التنبيه. يمكنك ضبط ذلك لكل قناة أو لكل حساب:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # إخفاء HEARTBEAT_OK (الافتراضي)
      showAlerts: true # إظهار رسائل التنبيه (الافتراضي)
      useIndicator: true # إصدار أحداث المؤشر (الافتراضي)
  telegram:
    heartbeat:
      showOk: true # إظهار إشعارات OK على Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # منع تسليم التنبيهات لهذا الحساب
```

الأولوية: لكل حساب ← لكل قناة ← القيم الافتراضية للقنوات ← القيم المدمجة الافتراضية.

### ما الذي يفعله كل علم

- `showOk`: يرسل إشعار `HEARTBEAT_OK` عندما يعيد النموذج ردًا يتضمن OK فقط.
- `showAlerts`: يرسل محتوى التنبيه عندما يعيد النموذج ردًا غير OK.
- `useIndicator`: يصدر أحداث مؤشرات لأسطح حالة واجهة المستخدم.

إذا كانت **الثلاثة كلها** false، فإن OpenClaw يتخطى تشغيل heartbeat بالكامل (من دون استدعاء للنموذج).

### أمثلة لكل قناة مقابل لكل حساب

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # كل حسابات Slack
    accounts:
      ops:
        heartbeat:
          showAlerts: false # منع التنبيهات لحساب ops فقط
  telegram:
    heartbeat:
      showOk: true
```

### الأنماط الشائعة

| الهدف                                    | التكوين                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| السلوك الافتراضي (OK صامت، والتنبيهات مفعلة) | _(لا حاجة إلى تكوين)_                                                                    |
| صامت بالكامل (لا رسائل، ولا مؤشر)        | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| مؤشر فقط (من دون رسائل)                  | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| رسائل OK في قناة واحدة فقط                | `channels.telegram.heartbeat: { showOk: true }`                                           |

## `HEARTBEAT.md` ‏(اختياري)

إذا كان ملف `HEARTBEAT.md` موجودًا في مساحة العمل، فإن المطالبة الافتراضية تطلب من
الوكيل قراءته. فكّر فيه باعتباره “قائمة التحقق الخاصة بـ heartbeat”: صغيرة، وثابتة،
وآمنة للإدراج كل 30 دقيقة.

إذا كان `HEARTBEAT.md` موجودًا لكنه فارغ فعليًا (أسطر فارغة فقط وعناوين markdown
مثل `# Heading`)، فإن OpenClaw يتخطى تشغيل heartbeat لتوفير استدعاءات API.
ويتم الإبلاغ عن هذا التجاوز على أنه `reason=empty-heartbeat-file`.
أما إذا كان الملف مفقودًا، فسيظل heartbeat يعمل ويقرر النموذج ما يجب فعله.

أبقِه صغيرًا (قائمة تحقق قصيرة أو تذكيرات) لتجنب تضخم المطالبة.

مثال على `HEARTBEAT.md`:

```md
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it’s daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

### كتل `tasks:`

يدعم `HEARTBEAT.md` أيضًا كتلة `tasks:` مهيكلة صغيرة للفحوصات
المعتمدة على الفواصل داخل heartbeat نفسه.

مثال:

```md
tasks:

- name: inbox-triage
  interval: 30m
  prompt: "Check for urgent unread emails and flag anything time sensitive."
- name: calendar-scan
  interval: 2h
  prompt: "Check for upcoming meetings that need prep or follow-up."

# Additional instructions

- Keep alerts short.
- If nothing needs attention after all due tasks, reply HEARTBEAT_OK.
```

السلوك:

- يقوم OpenClaw بتحليل كتلة `tasks:` والتحقق من كل مهمة مقابل `interval` الخاص بها.
- يتم تضمين المهام **المستحقة** فقط في مطالبة heartbeat لذلك التوقيت.
- إذا لم تكن هناك مهام مستحقة، يتم تخطي heartbeat بالكامل (`reason=no-tasks-due`) لتجنب استدعاء نموذج بلا فائدة.
- يتم الاحتفاظ بالمحتوى غير المتعلق بالمهام في `HEARTBEAT.md` وإلحاقه كسياق إضافي بعد قائمة المهام المستحقة.
- تُخزَّن الطوابع الزمنية لآخر تشغيل للمهام في حالة الجلسة (`heartbeatTaskState`)، لذا تبقى الفواصل الزمنية محفوظة عبر عمليات إعادة التشغيل العادية.
- لا يتم تقديم الطوابع الزمنية للمهام إلا بعد أن يكمل تشغيل heartbeat مسار الرد الطبيعي. ولا تضع التشغيلات المتخطاة بسبب `empty-heartbeat-file` / `no-tasks-due` علامة على المهام على أنها مكتملة.

يفيد وضع المهام عندما تريد أن يحتوي ملف heartbeat واحد على عدة فحوصات دورية من دون أن تدفع تكلفة جميعها في كل توقيت.

### هل يمكن للوكيل تحديث `HEARTBEAT.md`؟

نعم — إذا طلبت منه ذلك.

إن `HEARTBEAT.md` مجرد ملف عادي في مساحة عمل الوكيل، لذلك يمكنك أن تطلب من
الوكيل (في دردشة عادية) شيئًا مثل:

- “حدّث `HEARTBEAT.md` لإضافة فحص يومي للتقويم.”
- “أعد كتابة `HEARTBEAT.md` ليكون أقصر وأكثر تركيزًا على متابعات البريد الوارد.”

إذا كنت تريد أن يحدث ذلك بشكل استباقي، فيمكنك أيضًا تضمين سطر صريح في
مطالبة heartbeat مثل: “If the checklist becomes stale, update HEARTBEAT.md
with a better one.”

ملاحظة أمان: لا تضع أسرارًا (مفاتيح API، أو أرقام هواتف، أو رموزًا خاصة) في
`HEARTBEAT.md` — لأنه يصبح جزءًا من سياق المطالبة.

## إيقاظ يدوي (عند الطلب)

يمكنك صف نظام حدث وتشغيل heartbeat فوري باستخدام:

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

إذا كان عدة وكلاء لديهم `heartbeat` مكوّنًا، فإن الإيقاظ اليدوي يشغّل heartbeats
الخاصة بكل واحد من هؤلاء الوكلاء فورًا.

استخدم `--mode next-heartbeat` للانتظار حتى التوقيت المجدول التالي.

## تسليم Reasoning ‏(اختياري)

افتراضيًا، تسلّم heartbeats حمولة “الإجابة” النهائية فقط.

إذا كنت تريد الشفافية، ففعّل:

- `agents.defaults.heartbeat.includeReasoning: true`

عند التمكين، ستسلّم heartbeats أيضًا رسالة منفصلة مسبوقة بـ
`Reasoning:` ‏(بنفس شكل `/reasoning on`). وقد يكون هذا مفيدًا عندما يكون الوكيل
يدير جلسات/codexes متعددة وتريد معرفة سبب قراره بتنبيهك —
لكنه قد يكشف أيضًا تفاصيل داخلية أكثر مما تريد. ويفضل إبقاؤه
معطّلًا في الدردشات الجماعية.

## الوعي بالتكلفة

تشغّل Heartbeats أدوار وكيل كاملة. وكلما كانت الفواصل أقصر زادت تكلفة الرموز. لتقليل التكلفة:

- استخدم `isolatedSession: true` لتجنب إرسال السجل الكامل للمحادثة (تقريبًا من ~100K رمز إلى ~2-5K لكل تشغيل).
- استخدم `lightContext: true` لحصر ملفات bootstrap في `HEARTBEAT.md` فقط.
- عيّن `model` أرخص (مثل `ollama/llama3.2:1b`).
- أبقِ `HEARTBEAT.md` صغيرًا.
- استخدم `target: "none"` إذا كنت تريد فقط تحديثات الحالة الداخلية.

## ذو صلة

- [Automation & Tasks](/automation) — جميع آليات الأتمتة في لمحة
- [Background Tasks](/automation/tasks) — كيفية تتبع العمل المنفصل
- [Timezone](/concepts/timezone) — كيف تؤثر المنطقة الزمنية في جدولة heartbeat
- [Troubleshooting](/automation/cron-jobs#troubleshooting) — تصحيح مشكلات الأتمتة
