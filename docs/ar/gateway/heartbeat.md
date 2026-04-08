---
read_when:
    - تعديل وتيرة Heartbeat أو رسائله
    - اتخاذ قرار بين Heartbeat وcron للمهام المجدولة
summary: رسائل استطلاع Heartbeat وقواعد الإشعارات
title: Heartbeat
x-i18n:
    generated_at: "2026-04-08T02:15:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8021d747637060eacb91ec5f75904368a08790c19f4fca32acda8c8c0a25e41
    source_path: gateway/heartbeat.md
    workflow: 15
---

# Heartbeat (البوابة)

> **Heartbeat أم Cron؟** راجع [الأتمتة والمهام](/ar/automation) للحصول على إرشادات حول وقت استخدام كل منهما.

يشغّل Heartbeat **أدوار وكيل دورية** في الجلسة الرئيسية حتى يتمكن النموذج من
إبراز أي شيء يحتاج إلى انتباه دون إغراقك بالرسائل.

Heartbeat هو دور مجدول في الجلسة الرئيسية — وهو **لا** ينشئ سجلات [المهام الخلفية](/ar/automation/tasks).
سجلات المهام مخصصة للعمل المنفصل (تشغيلات ACP، والوكلاء الفرعيون، ووظائف cron المعزولة).

استكشاف الأخطاء وإصلاحها: [المهام المجدولة](/ar/automation/cron-jobs#troubleshooting)

## بداية سريعة (للمبتدئين)

1. اترك Heartbeat مفعّلًا (القيمة الافتراضية هي `30m`، أو `1h` لمصادقة Anthropic OAuth/الرمز، بما في ذلك إعادة استخدام Claude CLI) أو اضبط وتيرتك الخاصة.
2. أنشئ قائمة تحقق صغيرة في `HEARTBEAT.md` أو كتلة `tasks:` في مساحة عمل الوكيل (اختياري لكنه مستحسن).
3. قرر إلى أين يجب أن تذهب رسائل Heartbeat (القيمة الافتراضية هي `target: "none"`؛ اضبط `target: "last"` للتوجيه إلى آخر جهة اتصال).
4. اختياري: فعّل تسليم الاستدلال الخاص بـ Heartbeat لمزيد من الشفافية.
5. اختياري: استخدم سياق تمهيد خفيفًا إذا كانت تشغيلات Heartbeat تحتاج فقط إلى `HEARTBEAT.md`.
6. اختياري: فعّل الجلسات المعزولة لتجنب إرسال سجل المحادثة الكامل في كل Heartbeat.
7. اختياري: قيّد Heartbeats بساعات النشاط (بالتوقيت المحلي).

مثال على الإعدادات:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // توصيل صريح إلى آخر جهة اتصال (الافتراضي هو "none")
        directPolicy: "allow", // الافتراضي: السماح بالجهات المباشرة/الرسائل الخاصة؛ اضبط "block" لمنعها
        lightContext: true, // اختياري: حقن `HEARTBEAT.md` فقط من ملفات التمهيد
        isolatedSession: true, // اختياري: جلسة جديدة في كل تشغيل (من دون سجل محادثة)
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // اختياري: إرسال رسالة `Reasoning:` منفصلة أيضًا
      },
    },
  },
}
```

## القيم الافتراضية

- الفاصل الزمني: `30m` (أو `1h` عندما تكون مصادقة Anthropic OAuth/الرمز هي وضع المصادقة المكتشف، بما في ذلك إعادة استخدام Claude CLI). اضبط `agents.defaults.heartbeat.every` أو `agents.list[].heartbeat.every` لكل وكيل؛ استخدم `0m` للتعطيل.
- نص الموجّه (قابل للتهيئة عبر `agents.defaults.heartbeat.prompt`):
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- يُرسَل موجّه Heartbeat **حرفيًا** كرسالة مستخدم. يتضمن
  موجّه النظام قسم “Heartbeat” فقط عندما تكون Heartbeats مفعّلة للوكيل
  الافتراضي، ويكون التشغيل معلّمًا داخليًا.
- عندما يتم تعطيل Heartbeats باستخدام `0m`، تحذف التشغيلات العادية أيضًا `HEARTBEAT.md`
  من سياق التمهيد حتى لا يرى النموذج تعليمات Heartbeat فقط.
- يتم التحقق من ساعات النشاط (`heartbeat.activeHours`) في المنطقة الزمنية المهيأة.
  خارج النافذة، يتم تخطي Heartbeats حتى النبضة التالية داخل النافذة.

## ما الغرض من موجّه Heartbeat

الموجّه الافتراضي عام عمدًا:

- **المهام الخلفية**: عبارة “ضع في الاعتبار المهام المعلّقة” تدفع الوكيل إلى مراجعة
  المتابعات (البريد الوارد، والتقويم، والتذكيرات، والعمل المصطف في الطابور) وإبراز أي شيء عاجل.
- **التحقق من المستخدم**: عبارة “تحقق من إنسانك أحيانًا أثناء النهار” تدفع إلى
  رسالة خفيفة من حين لآخر مثل “هل تحتاج إلى شيء؟”، لكنها تتجنب الإزعاج الليلي
  باستخدام منطقتك الزمنية المحلية المهيأة (راجع [/concepts/timezone](/ar/concepts/timezone)).

يمكن لـ Heartbeat التفاعل مع [المهام الخلفية](/ar/automation/tasks) المكتملة، لكن تشغيل Heartbeat نفسه لا ينشئ سجل مهمة.

إذا كنت تريد من Heartbeat أن يفعل شيئًا محددًا جدًا (مثل “تحقق من إحصاءات Gmail PubSub”
أو “تحقق من صحة البوابة”)، فاضبط `agents.defaults.heartbeat.prompt` (أو
`agents.list[].heartbeat.prompt`) على نص مخصص (يُرسَل حرفيًا).

## عقد الاستجابة

- إذا لم يكن هناك ما يحتاج إلى انتباه، فردّ بـ **`HEARTBEAT_OK`**.
- أثناء تشغيلات Heartbeat، يتعامل OpenClaw مع `HEARTBEAT_OK` كإقرار عندما يظهر
  في **بداية أو نهاية** الرد. تتم إزالة الرمز ويُسقَط الرد
  إذا كان المحتوى المتبقي **≤ `ackMaxChars`** (الافتراضي: 300).
- إذا ظهر `HEARTBEAT_OK` في **منتصف** الرد، فلا تتم معاملته
  معاملة خاصة.
- للتنبيهات، **لا** تُضمّن `HEARTBEAT_OK`؛ أعد نص التنبيه فقط.

خارج Heartbeats، تتم إزالة أي `HEARTBEAT_OK` عارض في بداية/نهاية الرسالة
ويُسجَّل؛ وتُسقَط الرسالة إذا كانت فقط `HEARTBEAT_OK`.

## الإعدادات

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // الافتراضي: 30m (يعطَّل عند 0m)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // الافتراضي: false (تسليم رسالة Reasoning: منفصلة عند توفرها)
        lightContext: false, // الافتراضي: false؛ true يبقي فقط `HEARTBEAT.md` من ملفات تمهيد مساحة العمل
        isolatedSession: false, // الافتراضي: false؛ true يشغّل كل Heartbeat في جلسة جديدة (من دون سجل المحادثة)
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

### النطاق والأسبقية

- يضبط `agents.defaults.heartbeat` سلوك Heartbeat العام.
- يُدمَج `agents.list[].heartbeat` فوقه؛ إذا كان لدى أي وكيل كتلة `heartbeat`، فإن **هؤلاء الوكلاء فقط** هم من يشغّلون Heartbeats.
- يضبط `channels.defaults.heartbeat` القيم الافتراضية للظهور لجميع القنوات.
- يتجاوز `channels.<channel>.heartbeat` القيم الافتراضية للقناة.
- يتجاوز `channels.<channel>.accounts.<id>.heartbeat` (القنوات متعددة الحسابات) إعدادات كل قناة.

### Heartbeats لكل وكيل

إذا تضمّن أي إدخال في `agents.list[]` كتلة `heartbeat`، فإن **هؤلاء الوكلاء فقط**
هم من يشغّلون Heartbeats. وتُدمَج الكتلة الخاصة بكل وكيل فوق `agents.defaults.heartbeat`
(حتى تتمكن من ضبط القيم المشتركة مرة واحدة ثم تجاوزها لكل وكيل).

مثال: وكيلان، والوكيل الثاني فقط هو من يشغّل Heartbeats.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // توصيل صريح إلى آخر جهة اتصال (الافتراضي هو "none")
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

### مثال على ساعات النشاط

قيّد Heartbeats بساعات العمل في منطقة زمنية محددة:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // توصيل صريح إلى آخر جهة اتصال (الافتراضي هو "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // اختياري؛ يستخدم `userTimezone` إذا كان مضبوطًا، وإلا يستخدم المنطقة الزمنية للمضيف
        },
      },
    },
  },
}
```

خارج هذه النافذة (قبل التاسعة صباحًا أو بعد العاشرة مساءً بالتوقيت الشرقي)، يتم تخطي Heartbeats. وستعمل النبضة المجدولة التالية داخل النافذة بشكل طبيعي.

### إعداد 24/7

إذا كنت تريد تشغيل Heartbeats طوال اليوم، فاستخدم أحد هذين النمطين:

- احذف `activeHours` بالكامل (من دون تقييد بنافذة زمنية؛ وهذا هو السلوك الافتراضي).
- اضبط نافذة يوم كامل: `activeHours: { start: "00:00", end: "24:00" }`.

لا تضبط القيمتين `start` و`end` على الوقت نفسه (مثل `08:00` إلى `08:00`).
تُعامَل هذه الحالة على أنها نافذة بعرض صفري، لذلك يتم دائمًا تخطي Heartbeats.

### مثال على الحسابات المتعددة

استخدم `accountId` لاستهداف حساب محدد في القنوات متعددة الحسابات مثل Telegram:

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

- `every`: الفاصل الزمني لـ Heartbeat (سلسلة مدة؛ وحدة القياس الافتراضية = دقائق).
- `model`: تجاوز اختياري للنموذج في تشغيلات Heartbeat (`provider/model`).
- `includeReasoning`: عند تفعيله، يسلّم أيضًا رسالة `Reasoning:` المنفصلة عند توفرها (بالصيغة نفسها مثل `/reasoning on`).
- `lightContext`: عندما تكون قيمته true، تستخدم تشغيلات Heartbeat سياق تمهيد خفيفًا وتُبقي فقط `HEARTBEAT.md` من ملفات تمهيد مساحة العمل.
- `isolatedSession`: عندما تكون قيمته true، يعمل كل Heartbeat في جلسة جديدة من دون أي سجل محادثة سابق. ويستخدم نمط العزل نفسه مثل cron `sessionTarget: "isolated"`. ويقلل بدرجة كبيرة تكلفة الرموز لكل Heartbeat. اجمعه مع `lightContext: true` لتحقيق أكبر قدر من التوفير. يظل توجيه التسليم يستخدم سياق الجلسة الرئيسية.
- `session`: مفتاح جلسة اختياري لتشغيلات Heartbeat.
  - `main` (الافتراضي): الجلسة الرئيسية للوكيل.
  - مفتاح جلسة صريح (انسخه من `openclaw sessions --json` أو من [CLI الجلسات](/cli/sessions)).
  - صيغ مفاتيح الجلسة: راجع [الجلسات](/ar/concepts/session) و[المجموعات](/ar/channels/groups).
- `target`:
  - `last`: التسليم إلى آخر قناة خارجية مستخدمة.
  - قناة صريحة: أي قناة أو معرّف plugin مهيأ، مثل `discord` أو `matrix` أو `telegram` أو `whatsapp`.
  - `none` (الافتراضي): شغّل Heartbeat لكن **لا تُسلِّمه** خارجيًا.
- `directPolicy`: يتحكم في سلوك التسليم المباشر/الرسائل الخاصة:
  - `allow` (الافتراضي): السماح بتسليم Heartbeat إلى الأهداف المباشرة/الرسائل الخاصة.
  - `block`: منع التسليم المباشر/الرسائل الخاصة (`reason=dm-blocked`).
- `to`: تجاوز اختياري للمستلم (معرّف خاص بالقناة، مثل E.164 لـ WhatsApp أو معرّف دردشة Telegram). بالنسبة إلى topics/threads في Telegram، استخدم `<chatId>:topic:<messageThreadId>`.
- `accountId`: معرّف حساب اختياري للقنوات متعددة الحسابات. عندما تكون `target: "last"`، يُطبَّق معرّف الحساب على آخر قناة محلولة إذا كانت تدعم الحسابات؛ وإلا يتم تجاهله. وإذا لم يطابق معرّف الحساب حسابًا مهيأً للقناة المحلولة، يتم تخطي التسليم.
- `prompt`: يتجاوز نص الموجّه الافتراضي (من دون دمج).
- `ackMaxChars`: الحد الأقصى للأحرف المسموح بها بعد `HEARTBEAT_OK` قبل التسليم.
- `suppressToolErrorWarnings`: عندما تكون قيمته true، يمنع حمولات تحذير أخطاء الأدوات أثناء تشغيلات Heartbeat.
- `activeHours`: يقيّد تشغيلات Heartbeat بنافذة زمنية. وهو كائن يحتوي على `start` (HH:MM، شامل؛ استخدم `00:00` لبداية اليوم) و`end` (HH:MM غير شامل؛ ويسمح بـ `24:00` لنهاية اليوم) و`timezone` اختياري.
  - إذا حُذف أو كانت قيمته `"user"`: يستخدم `agents.defaults.userTimezone` إذا كان مضبوطًا، وإلا يعود إلى المنطقة الزمنية لنظام المضيف.
  - `"local"`: يستخدم دائمًا المنطقة الزمنية لنظام المضيف.
  - أي معرّف IANA (مثل `America/New_York`): يُستخدم مباشرة؛ وإذا كان غير صالح، يعود إلى سلوك `"user"` أعلاه.
  - يجب ألا تكون `start` و`end` متساويتين في النافذة النشطة؛ إذ تُعامَل القيم المتساوية على أنها عرض صفري (أي دائمًا خارج النافذة).
  - خارج النافذة النشطة، يتم تخطي Heartbeats حتى النبضة التالية داخل النافذة.

## سلوك التسليم

- تعمل Heartbeats افتراضيًا في الجلسة الرئيسية للوكيل (`agent:<id>:<mainKey>`)،
  أو `global` عندما تكون `session.scope = "global"`. اضبط `session` للتجاوز إلى
  جلسة قناة محددة (Discord/WhatsApp/إلخ).
- يؤثر `session` فقط في سياق التشغيل؛ ويتحكم `target` و`to` في التسليم.
- للتسليم إلى قناة/مستلم محدد، اضبط `target` + `to`. مع
  `target: "last"`، يستخدم التسليم آخر قناة خارجية لتلك الجلسة.
- تسمح تسليمات Heartbeat بالأهداف المباشرة/الرسائل الخاصة افتراضيًا. اضبط `directPolicy: "block"` لمنع الإرسال إلى الأهداف المباشرة مع الاستمرار في تشغيل دور Heartbeat.
- إذا كان الطابور الرئيسي مشغولًا، يتم تخطي Heartbeat وإعادة المحاولة لاحقًا.
- إذا لم يُحلّ `target` إلى أي وجهة خارجية، فسيحدث التشغيل لكن لن
  تُرسَل أي رسالة صادرة.
- إذا كانت `showOk` و`showAlerts` و`useIndicator` كلها معطلة، فيتم تخطي التشغيل مسبقًا بسبب `reason=alerts-disabled`.
- إذا كان تسليم التنبيهات فقط معطلًا، فلا يزال بإمكان OpenClaw تشغيل Heartbeat، وتحديث الطوابع الزمنية للمهام المستحقة، واستعادة الطابع الزمني لخمول الجلسة، ومنع حمولة التنبيه الخارجية.
- الردود الخاصة بـ Heartbeat فقط **لا** تُبقي الجلسة نشطة؛ بل تتم استعادة `updatedAt`
  الأخيرة حتى يعمل انتهاء صلاحية الخمول بشكل طبيعي.
- يمكن لـ [المهام الخلفية](/ar/automation/tasks) المنفصلة إدراج حدث نظام وإيقاظ Heartbeat عندما ينبغي أن تلاحظ الجلسة الرئيسية شيئًا بسرعة. وهذا الإيقاظ لا يجعل تشغيل Heartbeat مهمة خلفية.

## عناصر التحكم في الظهور

افتراضيًا، يتم منع إقرارات `HEARTBEAT_OK` بينما يتم
تسليم محتوى التنبيهات. يمكنك ضبط هذا لكل قناة أو لكل حساب:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # إخفاء HEARTBEAT_OK (الافتراضي)
      showAlerts: true # إظهار رسائل التنبيه (الافتراضي)
      useIndicator: true # إصدار أحداث المؤشر (الافتراضي)
  telegram:
    heartbeat:
      showOk: true # إظهار إقرارات OK على Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # منع تسليم التنبيهات لهذا الحساب
```

الأسبقية: لكل حساب ← لكل قناة ← القيم الافتراضية للقناة ← القيم الافتراضية المدمجة.

### ماذا يفعل كل علم

- `showOk`: يرسل إقرار `HEARTBEAT_OK` عندما يعيد النموذج ردًا يحتوي فقط على OK.
- `showAlerts`: يرسل محتوى التنبيه عندما يعيد النموذج ردًا غير OK.
- `useIndicator`: يصدر أحداث مؤشر لواجهات حالة UI.

إذا كانت **الثلاثة جميعًا** false، يتخطى OpenClaw تشغيل Heartbeat بالكامل (من دون استدعاء للنموذج).

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
      showOk: true # جميع حسابات Slack
    accounts:
      ops:
        heartbeat:
          showAlerts: false # منع التنبيهات لحساب ops فقط
  telegram:
    heartbeat:
      showOk: true
```

### الأنماط الشائعة

| الهدف                                     | الإعدادات                                                                                 |
| ----------------------------------------- | ----------------------------------------------------------------------------------------- |
| السلوك الافتراضي (إشعارات OK صامتة، والتنبيهات مفعّلة) | _(لا حاجة إلى إعدادات)_                                                                   |
| صامت بالكامل (من دون رسائل، ومن دون مؤشر) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| المؤشر فقط (من دون رسائل)                 | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| إظهار OK في قناة واحدة فقط                | `channels.telegram.heartbeat: { showOk: true }`                                           |

## `HEARTBEAT.md` (اختياري)

إذا كان ملف `HEARTBEAT.md` موجودًا في مساحة العمل، فإن الموجّه الافتراضي يطلب من
الوكيل قراءته. فكّر فيه باعتباره “قائمة تحقق Heartbeat” الخاصة بك: صغيرة، وثابتة،
وآمنة للإدراج كل 30 دقيقة.

في التشغيلات العادية، لا يتم حقن `HEARTBEAT.md` إلا عندما تكون إرشادات Heartbeat
مفعّلة للوكيل الافتراضي. ويؤدي تعطيل وتيرة Heartbeat باستخدام `0m` أو
ضبط `includeSystemPromptSection: false` إلى حذفه من سياق التمهيد
العادي.

إذا كان `HEARTBEAT.md` موجودًا لكنه فارغ فعليًا (يحتوي فقط على أسطر فارغة وعناوين markdown
مثل `# Heading`)، فإن OpenClaw يتخطى تشغيل Heartbeat لتوفير استدعاءات API.
ويتم الإبلاغ عن هذا التخطي باعتباره `reason=empty-heartbeat-file`.
أما إذا كان الملف مفقودًا، فلا يزال Heartbeat يعمل ويقرر النموذج ما الذي ينبغي فعله.

اجعله صغيرًا جدًا (قائمة تحقق قصيرة أو تذكيرات) لتجنب تضخم الموجّه.

مثال على `HEARTBEAT.md`:

```md
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it’s daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

### كتل `tasks:`

يدعم `HEARTBEAT.md` أيضًا كتلة `tasks:` منظَّمة صغيرة لإجراء
فحوصات قائمة على الفواصل الزمنية داخل Heartbeat نفسه.

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

- يحلل OpenClaw كتلة `tasks:` ويفحص كل مهمة وفق `interval` الخاص بها.
- لا يتم تضمين سوى المهام **المستحقة** في موجّه Heartbeat لتلك النبضة.
- إذا لم تكن هناك مهام مستحقة، يتم تخطي Heartbeat بالكامل (`reason=no-tasks-due`) لتجنب استدعاء نموذج غير ضروري.
- يتم الاحتفاظ بالمحتوى غير المتعلق بالمهام في `HEARTBEAT.md` وإلحاقه كسياق إضافي بعد قائمة المهام المستحقة.
- تُخزَّن الطوابع الزمنية لآخر تشغيل للمهمة في حالة الجلسة (`heartbeatTaskState`)، لذلك تظل الفواصل الزمنية محفوظة عبر عمليات إعادة التشغيل العادية.
- لا تُقدَّم الطوابع الزمنية للمهام إلا بعد أن يكمل Heartbeat مسار رده العادي. أما تشغيلات `empty-heartbeat-file` / `no-tasks-due` المتخطاة فلا تضع علامة على المهام باعتبارها مكتملة.

يفيد وضع المهام عندما تريد أن يحتوي ملف Heartbeat واحد على عدة فحوصات دورية من دون أن تدفع تكلفة جميعها في كل نبضة.

### هل يمكن للوكيل تحديث `HEARTBEAT.md`؟

نعم — إذا طلبت منه ذلك.

إن `HEARTBEAT.md` مجرد ملف عادي في مساحة عمل الوكيل، لذا يمكنك أن تطلب من
الوكيل (في دردشة عادية) شيئًا مثل:

- “حدّث `HEARTBEAT.md` لإضافة فحص يومي للتقويم.”
- “أعد كتابة `HEARTBEAT.md` ليصبح أقصر ويركز على متابعات البريد الوارد.”

إذا كنت تريد أن يحدث هذا بشكل استباقي، فيمكنك أيضًا تضمين سطر صريح في
موجّه Heartbeat مثل: “إذا أصبحت قائمة التحقق قديمة، فحدّث HEARTBEAT.md
بنسخة أفضل.”

ملاحظة أمان: لا تضع أسرارًا (مفاتيح API أو أرقام الهواتف أو الرموز الخاصة) داخل
`HEARTBEAT.md` — لأنه يصبح جزءًا من سياق الموجّه.

## إيقاظ يدوي (عند الطلب)

يمكنك إدراج حدث نظام وتشغيل Heartbeat فوري باستخدام:

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

إذا كان لدى عدة وكلاء إعداد `heartbeat`، فإن الإيقاظ اليدوي يشغّل Heartbeats الخاصة بكل واحد منهم
فورًا.

استخدم `--mode next-heartbeat` للانتظار حتى النبضة المجدولة التالية.

## تسليم الاستدلال (اختياري)

افتراضيًا، تسلّم Heartbeats حمولة “الإجابة” النهائية فقط.

إذا كنت تريد مزيدًا من الشفافية، ففعّل:

- `agents.defaults.heartbeat.includeReasoning: true`

عند التفعيل، ستسلّم Heartbeats أيضًا رسالة منفصلة مسبوقة بـ
`Reasoning:` (بالصيغة نفسها مثل `/reasoning on`). وقد يكون هذا مفيدًا عندما يكون الوكيل
يدير عدة جلسات/نسخ Codex وتريد أن ترى لماذا قرر أن ينبهك —
لكنه قد يكشف أيضًا تفاصيل داخلية أكثر مما تريد. لذا يُفضَّل إبقاؤه
معطلًا في الدردشات الجماعية.

## الوعي بالتكلفة

تشغّل Heartbeats أدوار وكيل كاملة. الفواصل الأقصر تستهلك رموزًا أكثر. لتقليل التكلفة:

- استخدم `isolatedSession: true` لتجنب إرسال سجل المحادثة الكامل (من ~100 ألف رمز إلى ~2-5 آلاف لكل تشغيل).
- استخدم `lightContext: true` لقصر ملفات التمهيد على `HEARTBEAT.md` فقط.
- اضبط `model` أرخص (مثل `ollama/llama3.2:1b`).
- اجعل `HEARTBEAT.md` صغيرًا.
- استخدم `target: "none"` إذا كنت تريد فقط تحديثات الحالة الداخلية.

## ذو صلة

- [الأتمتة والمهام](/ar/automation) — نظرة عامة على جميع آليات الأتمتة
- [المهام الخلفية](/ar/automation/tasks) — كيفية تتبع العمل المنفصل
- [المنطقة الزمنية](/ar/concepts/timezone) — كيف تؤثر المنطقة الزمنية في جدولة Heartbeat
- [استكشاف الأخطاء وإصلاحها](/ar/automation/cron-jobs#troubleshooting) — تصحيح أخطاء مشكلات الأتمتة
