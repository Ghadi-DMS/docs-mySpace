---
read_when:
    - إعداد Mattermost
    - تصحيح أخطاء توجيه Mattermost
summary: إعداد بوت Mattermost وتكوين OpenClaw
title: Mattermost
x-i18n:
    generated_at: "2026-04-05T12:36:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: f21dc7543176fda0b38b00fab60f0daae38dffcf68fa1cf7930a9f14ec57cb5a
    source_path: channels/mattermost.md
    workflow: 15
---

# Mattermost

الحالة: plugin مضمّن (رمز bot المميز + أحداث WebSocket). القنوات والمجموعات والرسائل الخاصة مدعومة.
Mattermost هو منصة مراسلة جماعية قابلة للاستضافة الذاتية؛ راجع الموقع الرسمي على
[mattermost.com](https://mattermost.com) للحصول على تفاصيل المنتج والتنزيلات.

## plugin المضمّن

يأتي Mattermost كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك لا تحتاج
الإصدارات المعبأة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن Mattermost،
فثبّته يدويًا:

التثبيت عبر CLI ‏(سجل npm):

```bash
openclaw plugins install @openclaw/mattermost
```

نسخة محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/mattermost-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## الإعداد السريع

1. تأكد من أن plugin Mattermost متاح.
   - تتضمن إصدارات OpenClaw المعبأة الحالية هذا plugin بالفعل.
   - يمكن للتثبيتات الأقدم/المخصصة إضافته يدويًا باستخدام الأوامر أعلاه.
2. أنشئ حساب bot في Mattermost وانسخ **رمز bot المميز**.
3. انسخ **عنوان URL الأساسي** لـ Mattermost (مثل `https://chat.example.com`).
4. قم بتكوين OpenClaw وابدأ gateway.

الحد الأدنى من التكوين:

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
    },
  },
}
```

## أوامر الشرطة المائلة الأصلية

أوامر الشرطة المائلة الأصلية اختيارية. عند تمكينها، يسجل OpenClaw أوامر الشرطة المائلة `oc_*` عبر
واجهة Mattermost API ويتلقى طلبات POST الراجعة على خادم HTTP الخاص بـ gateway.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // استخدم هذا عندما لا يتمكن Mattermost من الوصول إلى gateway مباشرة (reverse proxy/عنوان URL عام).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

ملاحظات:

- تكون القيمة الافتراضية لـ `native: "auto"` هي التعطيل في Mattermost. عيّن `native: true` للتمكين.
- إذا تم حذف `callbackUrl`، فسيشتق OpenClaw قيمة من مضيف/منفذ gateway + `callbackPath`.
- في إعدادات الحسابات المتعددة، يمكن تعيين `commands` في المستوى الأعلى أو تحت
  `channels.mattermost.accounts.<id>.commands` (تتجاوز قيم الحساب حقول المستوى الأعلى).
- يتم التحقق من عمليات callback الخاصة بالأوامر باستخدام الرموز المميزة لكل أمر التي يعيدها
  Mattermost عندما يسجل OpenClaw أوامر `oc_*`.
- تفشل عمليات callback الخاصة بأوامر الشرطة المائلة بشكل مغلق عندما يفشل التسجيل، أو يكون بدء التشغيل جزئيًا، أو
  لا يطابق رمز callback أحد الأوامر المسجلة.
- متطلب إمكانية الوصول: يجب أن تكون نقطة نهاية callback قابلة للوصول من خادم Mattermost.
  - لا تعيّن `callbackUrl` إلى `localhost` إلا إذا كان Mattermost يعمل على نفس المضيف/namespace الشبكة مثل OpenClaw.
  - لا تعيّن `callbackUrl` إلى عنوان URL الأساسي لـ Mattermost إلا إذا كان هذا العنوان يستخدم reverse-proxy للمسار `/api/channels/mattermost/command` إلى OpenClaw.
  - فحص سريع هو `curl https://<gateway-host>/api/channels/mattermost/command`; يجب أن يعيد طلب GET قيمة `405 Method Not Allowed` من OpenClaw، وليس `404`.
- متطلب قائمة السماح لخروج Mattermost:
  - إذا كان callback لديك يستهدف عناوين خاصة/tailnet/داخلية، فعيّن
    `ServiceSettings.AllowedUntrustedInternalConnections` في Mattermost ليشمل مضيف/نطاق callback.
  - استخدم إدخالات المضيف/النطاق، وليس عناوين URL كاملة.
    - جيد: `gateway.tailnet-name.ts.net`
    - سيئ: `https://gateway.tailnet-name.ts.net`

## متغيرات البيئة (الحساب الافتراضي)

اضبط هذه المتغيرات على مضيف gateway إذا كنت تفضل استخدام متغيرات البيئة:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

تنطبق متغيرات البيئة فقط على الحساب **الافتراضي** (`default`). يجب أن تستخدم الحسابات الأخرى قيم التكوين.

## أوضاع الدردشة

يرد Mattermost على الرسائل الخاصة تلقائيًا. يتم التحكم في سلوك القنوات بواسطة `chatmode`:

- `oncall` (الافتراضي): الرد فقط عند الإشارة إليه بـ @ في القنوات.
- `onmessage`: الرد على كل رسالة في القناة.
- `onchar`: الرد عندما تبدأ الرسالة ببادئة تشغيل.

مثال على التكوين:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"],
    },
  },
}
```

ملاحظات:

- ما زال `onchar` يرد على إشارات @ الصريحة.
- يتم احترام `channels.mattermost.requireMention` في التكوينات القديمة لكن `chatmode` هو المفضل.

## سلاسل الرسائل والجلسات

استخدم `channels.mattermost.replyToMode` للتحكم في ما إذا كانت الردود في القنوات والمجموعات تبقى في
القناة الرئيسية أو تبدأ سلسلة رسائل تحت المنشور المُشغِّل.

- `off` (الافتراضي): لا يتم الرد في سلسلة رسائل إلا عندما يكون المنشور الوارد داخل سلسلة بالفعل.
- `first`: بالنسبة لمنشورات القنوات/المجموعات ذات المستوى الأعلى، ابدأ سلسلة رسائل تحت ذلك المنشور ووجّه
  المحادثة إلى جلسة ضمن نطاق السلسلة.
- `all`: نفس سلوك `first` في Mattermost حاليًا.
- تتجاهل الرسائل الخاصة هذا الإعداد وتبقى بدون سلاسل.

مثال على التكوين:

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
    },
  },
}
```

ملاحظات:

- تستخدم الجلسات ضمن نطاق السلسلة معرّف المنشور المُشغِّل كجذر للسلسلة.
- `first` و`all` متكافئان حاليًا لأنه بمجرد أن يملك Mattermost جذر سلسلة،
  تستمر الأجزاء اللاحقة والوسائط في نفس السلسلة.

## التحكم في الوصول (الرسائل الخاصة)

- الافتراضي: `channels.mattermost.dmPolicy = "pairing"` ‏(يحصل المرسلون غير المعروفين على رمز اقتران).
- وافق عبر:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- الرسائل الخاصة العامة: `channels.mattermost.dmPolicy="open"` مع `channels.mattermost.allowFrom=["*"]`.

## القنوات (المجموعات)

- الافتراضي: `channels.mattermost.groupPolicy = "allowlist"` ‏(مقيّد بالإشارة).
- اسمح للمرسلين عبر `channels.mattermost.groupAllowFrom` (يوصى باستخدام معرّفات المستخدمين).
- توجد تجاوزات الإشارة لكل قناة تحت `channels.mattermost.groups.<channelId>.requireMention`
  أو `channels.mattermost.groups["*"].requireMention` كقيمة افتراضية.
- المطابقة باستخدام `@username` قابلة للتغيير ولا يتم تمكينها إلا عندما تكون `channels.mattermost.dangerouslyAllowNameMatching: true`.
- القنوات المفتوحة: `channels.mattermost.groupPolicy="open"` ‏(مقيّد بالإشارة).
- ملاحظة وقت التشغيل: إذا كانت `channels.mattermost` مفقودة بالكامل، فإن وقت التشغيل يعود إلى `groupPolicy="allowlist"` لفحوصات المجموعات (حتى لو كانت `channels.defaults.groupPolicy` معيّنة).

مثال:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## الأهداف للتسليم الصادر

استخدم صيغ الأهداف التالية مع `openclaw message send` أو cron/webhooks:

- `channel:<id>` لقناة
- `user:<id>` لرسالة خاصة
- `@username` لرسالة خاصة (يتم حلها عبر Mattermost API)

المعرّفات المعتمة المجردة (مثل `64ifufp...`) **ملتبسة** في Mattermost (معرّف مستخدم أم معرّف قناة).

يقوم OpenClaw بحلها **كمستخدم أولًا**:

- إذا كان المعرّف موجودًا كمستخدم (`GET /api/v4/users/<id>` ينجح)، يرسل OpenClaw **رسالة خاصة** عبر حل القناة المباشرة باستخدام `/api/v4/channels/direct`.
- بخلاف ذلك، يُعامل المعرّف على أنه **معرّف قناة**.

إذا كنت تحتاج إلى سلوك حتمي، فاستخدم دائمًا البوادئ الصريحة (`user:<id>` / `channel:<id>`).

## إعادة محاولة قناة الرسائل الخاصة

عندما يرسل OpenClaw إلى هدف رسالة خاصة في Mattermost ويحتاج إلى حل القناة المباشرة أولًا، فإنه
يعيد محاولة إخفاقات إنشاء القناة المباشرة العابرة افتراضيًا.

استخدم `channels.mattermost.dmChannelRetry` لضبط هذا السلوك عمومًا لـ plugin Mattermost،
أو `channels.mattermost.accounts.<id>.dmChannelRetry` لحساب واحد.

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

ملاحظات:

- ينطبق هذا فقط على إنشاء قناة الرسائل الخاصة (`/api/v4/channels/direct`)، وليس على كل استدعاء Mattermost API.
- تنطبق إعادة المحاولات على الإخفاقات العابرة مثل حدود المعدل، واستجابات 5xx، وأخطاء الشبكة أو انتهاء المهلة.
- تُعامل أخطاء العميل 4xx بخلاف `429` على أنها دائمة ولا تتم إعادة محاولتها.

## التفاعلات (أداة الرسائل)

- استخدم `message action=react` مع `channel=mattermost`.
- `messageId` هو معرّف منشور Mattermost.
- تقبل `emoji` أسماء مثل `thumbsup` أو `:+1:` ‏(النقطتان اختيارية).
- عيّن `remove=true` ‏(منطقي) لإزالة تفاعل.
- يتم تمرير أحداث إضافة/إزالة التفاعل كأحداث نظام إلى جلسة الوكيل الموجّهة.

أمثلة:

```
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

التكوين:

- `channels.mattermost.actions.reactions`: تمكين/تعطيل إجراءات التفاعل (الافتراضي true).
- تجاوز لكل حساب: `channels.mattermost.accounts.<id>.actions.reactions`.

## الأزرار التفاعلية (أداة الرسائل)

أرسل رسائل تحتوي على أزرار قابلة للنقر. عندما ينقر المستخدم زرًا، يتلقى الوكيل
الاختيار ويمكنه الرد.

قم بتمكين الأزرار بإضافة `inlineButtons` إلى قدرات القناة:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

استخدم `message action=send` مع معلمة `buttons`. الأزرار عبارة عن مصفوفة ثنائية الأبعاد (صفوف من الأزرار):

```
message action=send channel=mattermost target=channel:<channelId> buttons=[[{"text":"Yes","callback_data":"yes"},{"text":"No","callback_data":"no"}]]
```

حقول الزر:

- `text` (مطلوب): تسمية العرض.
- `callback_data` (مطلوب): القيمة التي تُرسل مرة أخرى عند النقر (تُستخدم كمعرّف الإجراء).
- `style` (اختياري): `"default"` أو `"primary"` أو `"danger"`.

عندما ينقر المستخدم زرًا:

1. تُستبدل كل الأزرار بسطر تأكيد (مثل "✓ **Yes** selected by @user").
2. يتلقى الوكيل الاختيار كرسالة واردة ويرد.

ملاحظات:

- تستخدم عمليات callback الخاصة بالأزرار تحقق HMAC-SHA256 ‏(تلقائي، ولا حاجة إلى تكوين).
- يزيل Mattermost بيانات callback من استجابات API الخاصة به (ميزة أمان)، لذلك تتم
  إزالة جميع الأزرار عند النقر — ولا يمكن الإزالة الجزئية.
- يتم تنظيف معرّفات الإجراءات التي تحتوي على شرطات أو شرطات سفلية تلقائيًا
  (بسبب قيد في توجيه Mattermost).

التكوين:

- `channels.mattermost.capabilities`: مصفوفة سلاسل للقدرات. أضف `"inlineButtons"` إلى
  تمكين وصف أداة الأزرار في مطالبة نظام الوكيل.
- `channels.mattermost.interactions.callbackBaseUrl`: عنوان URL أساسي خارجي اختياري لعمليات callback
  الخاصة بالأزرار (مثل `https://gateway.example.com`). استخدمه عندما يتعذر على Mattermost
  الوصول إلى gateway عند مضيف الربط مباشرة.
- في إعدادات الحسابات المتعددة، يمكنك أيضًا تعيين الحقل نفسه تحت
  `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl`.
- إذا تم حذف `interactions.callbackBaseUrl`، يشتق OpenClaw عنوان URL الخاص بـ callback من
  `gateway.customBindHost` + `gateway.port`، ثم يعود إلى `http://localhost:<port>`.
- قاعدة إمكانية الوصول: يجب أن يكون عنوان URL الخاص بـ callback للأزرار قابلاً للوصول من خادم Mattermost.
  لا يعمل `localhost` إلا عندما يعمل Mattermost وOpenClaw على نفس المضيف/namespace الشبكة.
- إذا كان هدف callback لديك خاصًا/tailnet/داخليًا، فأضف مضيفه/نطاقه إلى
  `ServiceSettings.AllowedUntrustedInternalConnections` في Mattermost.

### تكامل API مباشر (scripts خارجية)

يمكن لـ scripts وwebhooks الخارجية نشر الأزرار مباشرة عبر Mattermost REST API
بدلًا من المرور عبر أداة `message` الخاصة بالوكيل. استخدم `buildButtonAttachments()` من
الامتداد عندما يكون ذلك ممكنًا؛ وإذا كنت تنشر JSON خامًا، فاتبع هذه القواعد:

**بنية الحمولة:**

```json5
{
  channel_id: "<channelId>",
  message: "Choose an option:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // أبجدي رقمي فقط — راجع أدناه
            type: "button", // مطلوب، وإلا فسيتم تجاهل النقرات بصمت
            name: "Approve", // تسمية العرض
            style: "primary", // اختياري: "default" أو "primary" أو "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // يجب أن يطابق id الخاص بالزر (لاستخدامه في البحث عن الاسم)
                action: "approve",
                // ... أي حقول مخصصة ...
                _token: "<hmac>", // راجع قسم HMAC أدناه
              },
            },
          },
        ],
      },
    ],
  },
}
```

**القواعد الحرجة:**

1. توضع المرفقات في `props.attachments`، وليس في `attachments` على المستوى الأعلى (وإلا يتم تجاهلها بصمت).
2. يحتاج كل إجراء إلى `type: "button"` — من دونه، يتم ابتلاع النقرات بصمت.
3. يحتاج كل إجراء إلى حقل `id` — يتجاهل Mattermost الإجراءات من دون معرّفات.
4. يجب أن يكون `id` الخاص بالإجراء **أبجديًا رقميًا فقط** (`[a-zA-Z0-9]`). تؤدي الشرطات والشرطات السفلية إلى كسر
   توجيه الإجراءات من جهة خادم Mattermost (ويُرجع 404). أزلها قبل الاستخدام.
5. يجب أن يطابق `context.action_id` قيمة `id` الخاصة بالزر حتى تعرض رسالة التأكيد
   اسم الزر (مثل "Approve") بدلًا من معرّف خام.
6. `context.action_id` مطلوب — يعيد معالج التفاعل 400 من دونه.

**توليد رمز HMAC:**

يتحقق gateway من نقرات الأزرار باستخدام HMAC-SHA256. يجب على scripts الخارجية توليد رموز
تطابق منطق التحقق في gateway:

1. اشتق السر من رمز bot المميز:
   `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`
2. ابنِ كائن السياق بجميع الحقول **باستثناء** `_token`.
3. قم بالتسلسل باستخدام **مفاتيح مرتبة** و**من دون مسافات** (يستخدم gateway ‏`JSON.stringify`
   مع مفاتيح مرتبة، ما ينتج خرجًا مضغوطًا).
4. وقّع: `HMAC-SHA256(key=secret, data=serializedContext)`
5. أضف الملخص السداسي الناتج كقيمة `_token` في السياق.

مثال Python:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

المشكلات الشائعة في HMAC:

- يضيف `json.dumps` في Python مسافات افتراضيًا (`{"key": "val"}`). استخدم
  `separators=(",", ":")` لمطابقة الخرج المضغوط في JavaScript ‏(`{"key":"val"}`).
- وقّع دائمًا **كل** حقول السياق (باستثناء `_token`). يزيل gateway قيمة `_token` ثم
  يوقّع كل ما تبقى. يؤدي توقيع مجموعة فرعية إلى فشل صامت في التحقق.
- استخدم `sort_keys=True` — يقوم gateway بترتيب المفاتيح قبل التوقيع، وقد يعيد Mattermost
  ترتيب حقول السياق عند تخزين الحمولة.
- اشتق السر من رمز bot المميز (حتمي)، وليس من بايتات عشوائية. يجب أن يكون السر
  نفسه في كل من العملية التي تنشئ الأزرار وgateway الذي يتحقق منها.

## مهايئ الدليل

يتضمن plugin Mattermost مهايئ دليل يحل أسماء القنوات والمستخدمين
عبر Mattermost API. يتيح هذا استخدام أهداف `#channel-name` و`@username` في
`openclaw message send` وعمليات التسليم عبر cron/webhook.

لا يلزم أي تكوين — يستخدم المهايئ رمز bot المميز من تكوين الحساب.

## حسابات متعددة

يدعم Mattermost حسابات متعددة تحت `channels.mattermost.accounts`:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

- لا توجد ردود في القنوات: تأكد من وجود bot في القناة واذكره (oncall)، أو استخدم بادئة تشغيل (onchar)، أو عيّن `chatmode: "onmessage"`.
- أخطاء المصادقة: تحقق من رمز bot المميز وعنوان URL الأساسي وما إذا كان الحساب ممكّنًا.
- مشكلات الحسابات المتعددة: تنطبق متغيرات البيئة فقط على الحساب `default`.
- تعيد أوامر الشرطة المائلة الأصلية `Unauthorized: invalid command token.`: لم
  يقبل OpenClaw رمز callback المميز. الأسباب المعتادة:
  - فشل تسجيل أوامر الشرطة المائلة أو اكتمل جزئيًا فقط عند بدء التشغيل
  - يصل callback إلى gateway/الحساب الخطأ
  - ما زال Mattermost يملك أوامر قديمة تشير إلى هدف callback سابق
  - أُعيد تشغيل gateway من دون إعادة تنشيط أوامر الشرطة المائلة
- إذا توقفت أوامر الشرطة المائلة الأصلية عن العمل، فتحقق من السجلات بحثًا عن
  `mattermost: failed to register slash commands` أو
  `mattermost: native slash commands enabled but no commands could be registered`.
- إذا تم حذف `callbackUrl` وكانت السجلات تحذر من أن callback تم حله إلى
  `http://127.0.0.1:18789/...`، فمن المحتمل أن يكون عنوان URL هذا قابلاً للوصول فقط عندما
  يعمل Mattermost على نفس المضيف/namespace الشبكة مثل OpenClaw. عيّن
  `commands.callbackUrl` صريحًا وقابلًا للوصول من الخارج بدلًا من ذلك.
- تظهر الأزرار كمربعات بيضاء: ربما يرسل الوكيل بيانات أزرار غير صحيحة. تحقق من أن كل زر يحتوي على الحقلين `text` و`callback_data`.
- تُعرض الأزرار لكن النقرات لا تفعل شيئًا: تحقق من أن `AllowedUntrustedInternalConnections` في تكوين خادم Mattermost يتضمن `127.0.0.1 localhost`، وأن `EnablePostActionIntegration` يساوي `true` في ServiceSettings.
- تُرجع الأزرار 404 عند النقر: من المحتمل أن يحتوي `id` الخاص بالزر على شرطات أو شرطات سفلية. يتعطل موجه إجراءات Mattermost مع المعرّفات غير الأبجدية الرقمية. استخدم `[a-zA-Z0-9]` فقط.
- تُظهر سجلات gateway الرسالة `invalid _token`: عدم تطابق HMAC. تحقق من أنك توقّع كل حقول السياق (وليس مجموعة فرعية)، وتستخدم مفاتيح مرتبة، وJSON مضغوطًا (من دون مسافات). راجع قسم HMAC أعلاه.
- تُظهر سجلات gateway الرسالة `missing _token in context`: الحقل `_token` غير موجود في سياق الزر. تأكد من تضمينه عند إنشاء حمولة التكامل.
- يُظهر التأكيد معرّفًا خامًا بدلًا من اسم الزر: `context.action_id` لا يطابق `id` الخاص بالزر. عيّن الاثنين إلى القيمة المنظفة نفسها.
- الوكيل لا يعرف الأزرار: أضف `capabilities: ["inlineButtons"]` إلى تكوين قناة Mattermost.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل الخاصة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وضبط الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
