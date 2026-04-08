---
read_when:
    - إعداد Matrix في OpenClaw
    - تكوين E2EE والتحقق في Matrix
summary: حالة دعم Matrix وإعداده وأمثلة التكوين
title: Matrix
x-i18n:
    generated_at: "2026-04-08T02:16:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec926df79a41fa296d63f0ec7219d0f32e075628d76df9ea490e93e4c5030f83
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix هو إضافة القناة المضمّنة Matrix في OpenClaw.
يستخدم `matrix-js-sdk` الرسمي ويدعم الرسائل المباشرة، والغرف، والخيوط، والوسائط، والتفاعلات، والاستطلاعات، والموقع، وE2EE.

## الإضافة المضمّنة

يأتي Matrix كإضافة مضمّنة في إصدارات OpenClaw الحالية، لذلك لا تحتاج
الإصدارات المعبأة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستثني Matrix، فقم بتثبيته
يدويًا:

التثبيت من npm:

```bash
openclaw plugins install @openclaw/matrix
```

التثبيت من نسخة سحب محلية:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

راجع [الإضافات](/ar/tools/plugin) للاطلاع على سلوك الإضافات وقواعد التثبيت.

## الإعداد

1. تأكد من أن إضافة Matrix متاحة.
   - إصدارات OpenClaw المعبأة الحالية تتضمنها بالفعل.
   - يمكن للتثبيتات الأقدم/المخصصة إضافتها يدويًا باستخدام الأوامر أعلاه.
2. أنشئ حساب Matrix على الخادم المنزلي الخاص بك.
3. قم بتكوين `channels.matrix` باستخدام أحد الخيارين التاليين:
   - `homeserver` + `accessToken`، أو
   - `homeserver` + `userId` + `password`.
4. أعد تشغيل البوابة.
5. ابدأ رسالة مباشرة مع الروبوت أو ادعه إلى غرفة.
   - لا تعمل دعوات Matrix الجديدة إلا عندما يسمح `channels.matrix.autoJoin` بها.

مسارات الإعداد التفاعلية:

```bash
openclaw channels add
openclaw configure --section channels
```

ما الذي يسأل عنه معالج Matrix فعليًا:

- عنوان URL للخادم المنزلي
- طريقة المصادقة: رمز وصول أو كلمة مرور
- معرّف المستخدم فقط عند اختيار المصادقة بكلمة المرور
- اسم جهاز اختياري
- ما إذا كان يجب تمكين E2EE
- ما إذا كان يجب تكوين وصول غرفة Matrix الآن
- ما إذا كان يجب تكوين الانضمام التلقائي لدعوات Matrix الآن
- عند تمكين الانضمام التلقائي للدعوات، ما إذا كان يجب أن يكون `allowlist` أو `always` أو `off`

سلوك المعالج المهم:

- إذا كانت متغيرات البيئة الخاصة بمصادقة Matrix موجودة بالفعل للحساب المحدد، ولم يكن هذا الحساب محفوظًا له مصادقة في التكوين بعد، فسيعرض المعالج اختصارًا للبيئة بحيث يمكن للإعداد إبقاء المصادقة في متغيرات البيئة بدلًا من نسخ الأسرار إلى التكوين.
- عند إضافة حساب Matrix آخر بشكل تفاعلي، يُطبَّع اسم الحساب المُدخل إلى معرّف الحساب المستخدم في التكوين ومتغيرات البيئة. على سبيل المثال، يتحول `Ops Bot` إلى `ops-bot`.
- تقبل مطالبات قائمة السماح للرسائل المباشرة قيم `@user:server` الكاملة مباشرة. تعمل أسماء العرض فقط عندما يعثر البحث المباشر في الدليل على تطابق واحد دقيق؛ وإلا فسيطلب منك المعالج إعادة المحاولة باستخدام معرّف Matrix كامل.
- تقبل مطالبات قائمة السماح للغرف معرّفات الغرف والأسماء المستعارة مباشرة. ويمكنها أيضًا حل أسماء الغرف المنضم إليها بشكل مباشر، لكن الأسماء غير المحلولة تُحتفظ بها كما أُدخلت أثناء الإعداد فقط ويتم تجاهلها لاحقًا بواسطة حل قائمة السماح وقت التشغيل. فضّل استخدام `!room:server` أو `#alias:server`.
- يعرض المعالج الآن تحذيرًا صريحًا قبل خطوة الانضمام التلقائي للدعوات لأن القيمة الافتراضية لـ `channels.matrix.autoJoin` هي `off`؛ لن تنضم الوكلاء إلى الغرف المدعوّة أو الدعوات الجديدة الشبيهة بالرسائل المباشرة ما لم تقم بتعيينه.
- في وضع قائمة السماح للانضمام التلقائي للدعوات، استخدم فقط أهداف دعوات مستقرة: `!roomId:server` أو `#alias:server` أو `*`. تُرفض أسماء الغرف العادية.
- تستخدم هوية الغرفة/الجلسة وقت التشغيل معرّف غرفة Matrix المستقر. لا تُستخدم الأسماء المستعارة المعلنة في الغرفة إلا كمدخلات للبحث، وليس كمفتاح جلسة طويل الأمد أو هوية مجموعة مستقرة.
- لحل أسماء الغرف قبل حفظها، استخدم `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
القيمة الافتراضية لـ `channels.matrix.autoJoin` هي `off`.

إذا تركته غير معيّن، فلن ينضم الروبوت إلى الغرف المدعوّة أو الدعوات الجديدة الشبيهة بالرسائل المباشرة، لذلك لن يظهر في المجموعات الجديدة أو الرسائل المباشرة المدعو إليها ما لم تنضم يدويًا أولًا.

عيّن `autoJoin: "allowlist"` مع `autoJoinAllowlist` لتقييد الدعوات التي يقبلها، أو عيّن `autoJoin: "always"` إذا كنت تريد منه الانضمام إلى كل دعوة.

في وضع `allowlist`، لا يقبل `autoJoinAllowlist` إلا `!roomId:server` أو `#alias:server` أو `*`.
</Warning>

مثال على قائمة السماح:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

الانضمام إلى كل دعوة:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

إعداد أدنى يعتمد على الرمز:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

إعداد يعتمد على كلمة المرور (يتم تخزين الرمز مؤقتًا بعد تسجيل الدخول):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

يخزن Matrix بيانات الاعتماد المخزنة مؤقتًا في `~/.openclaw/credentials/matrix/`.
يستخدم الحساب الافتراضي `credentials.json`؛ وتستخدم الحسابات المسماة `credentials-<account>.json`.
عند وجود بيانات اعتماد مخزنة مؤقتًا هناك، يتعامل OpenClaw مع Matrix على أنه مُكوَّن من أجل الإعداد، وdoctor، واكتشاف حالة القناة حتى إذا لم تكن المصادقة الحالية معيّنة مباشرة في التكوين.

مكافئات متغيرات البيئة (تُستخدم عندما لا يكون مفتاح التكوين معيّنًا):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

للحسابات غير الافتراضية، استخدم متغيرات البيئة ذات النطاق الخاص بالحساب:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

مثال للحساب `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

بالنسبة إلى معرّف الحساب المُطبّع `ops-bot`، استخدم:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

يهرب Matrix علامات الترقيم في معرّفات الحسابات للحفاظ على متغيرات البيئة ذات النطاق الخاص خالية من التعارضات.
على سبيل المثال، تتحول `-` إلى `_X2D_`، لذا يُطابق `ops-prod` النمط `MATRIX_OPS_X2D_PROD_*`.

لا يعرض المعالج التفاعلي اختصار متغيرات البيئة إلا عندما تكون متغيرات بيئة المصادقة هذه موجودة بالفعل ولا تكون مصادقة Matrix للحساب المحدد محفوظة بالفعل في التكوين.

## مثال على التكوين

هذا تكوين أساسي عملي يتضمن إقران الرسائل المباشرة، وقائمة سماح للغرف، وتمكين E2EE:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

ينطبق `autoJoin` على دعوات Matrix عمومًا، وليس فقط على دعوات الغرف/المجموعات.
ويشمل ذلك الدعوات الجديدة الشبيهة بالرسائل المباشرة. في وقت الدعوة، لا يعرف OpenClaw بشكل موثوق ما إذا كانت
الغرفة المدعو إليها ستُعامل في النهاية كرسالة مباشرة أم كمجموعة، لذلك تمر جميع الدعوات عبر نفس
قرار `autoJoin` أولًا. يظل `dm.policy` مطبقًا بعد انضمام الروبوت وتصنيف الغرفة
كرسالة مباشرة، لذا يتحكم `autoJoin` في سلوك الانضمام بينما يتحكم `dm.policy` في سلوك
الرد/الوصول.

## معاينات البث

بث الردود في Matrix اختياري.

عيّن `channels.matrix.streaming` إلى `"partial"` عندما تريد من OpenClaw إرسال
رد معاينة مباشر واحد، وتحرير هذه المعاينة في مكانها بينما يولد النموذج النص، ثم إنهاءها عندما
يكتمل الرد:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` هو الافتراضي. ينتظر OpenClaw الرد النهائي ويرسله مرة واحدة.
- ينشئ `streaming: "partial"` رسالة معاينة واحدة قابلة للتحرير للكتلة الحالية من رد المساعد باستخدام رسائل Matrix النصية العادية. يحافظ هذا على سلوك الإشعارات القديم في Matrix القائم على المعاينة أولًا، لذا قد ترسل البرامج العميلة القياسية إشعارًا عند نص المعاينة الأول المبثوث بدلًا من الكتلة النهائية.
- ينشئ `streaming: "quiet"` إشعار معاينة هادئ واحد قابل للتحرير للكتلة الحالية من رد المساعد. استخدم هذا فقط عندما تضبط أيضًا قواعد دفع للمستلمين من أجل تعديلات المعاينة النهائية.
- يفعّل `blockStreaming: true` رسائل تقدم منفصلة في Matrix. عند تمكين بث المعاينة، يحتفظ Matrix بالمسودة المباشرة للكتلة الحالية ويحافظ على الكتل المكتملة كرسائل منفصلة.
- عندما يكون بث المعاينة قيد التشغيل ويكون `blockStreaming` معطّلًا، يحرر Matrix المسودة المباشرة في مكانها وينهي نفس الحدث عند اكتمال الكتلة أو الدور.
- إذا لم تعد المعاينة تتسع لحدث Matrix واحد، يوقف OpenClaw بث المعاينة ويعود إلى التسليم النهائي العادي.
- لا تزال ردود الوسائط ترسل المرفقات بشكل طبيعي. إذا تعذر إعادة استخدام معاينة قديمة بأمان، يقوم OpenClaw بحذفها قبل إرسال رد الوسائط النهائي.
- تكلف تعديلات المعاينة استدعاءات إضافية إلى Matrix API. اترك البث معطّلًا إذا كنت تريد أكثر سلوك محافظ فيما يتعلق بحدود المعدل.

لا يفعّل `blockStreaming` معاينات المسودة بحد ذاته.
استخدم `streaming: "partial"` أو `streaming: "quiet"` من أجل تعديلات المعاينة؛ ثم أضف `blockStreaming: true` فقط إذا كنت تريد أيضًا أن تظل كتل المساعد المكتملة مرئية كرسائل تقدم منفصلة.

إذا كنت تحتاج إلى إشعارات Matrix القياسية دون قواعد دفع مخصصة، فاستخدم `streaming: "partial"` لسلوك المعاينة أولًا أو اترك `streaming` معطّلًا للتسليم النهائي فقط. مع `streaming: "off"`:

- يرسل `blockStreaming: true` كل كتلة مكتملة كرسالة Matrix عادية ترسل إشعارًا.
- يرسل `blockStreaming: false` الرد النهائي المكتمل فقط كرسالة Matrix عادية ترسل إشعارًا.

### قواعد دفع مستضافة ذاتيًا للمعاينات النهائية الهادئة

إذا كنت تشغّل بنية Matrix الخاصة بك وتريد أن ترسل المعاينات الهادئة إشعارًا فقط عند اكتمال كتلة أو
الرد النهائي، فعيّن `streaming: "quiet"` وأضف قاعدة دفع لكل مستخدم لمعاينات التعديل النهائية.

يكون هذا عادة إعدادًا على مستوى المستخدم المستلم، وليس تغييرًا عامًا في تكوين الخادم المنزلي:

خريطة سريعة قبل أن تبدأ:

- المستخدم المستلم = الشخص الذي يجب أن يتلقى الإشعار
- مستخدم الروبوت = حساب OpenClaw Matrix الذي يرسل الرد
- استخدم رمز وصول المستخدم المستلم في استدعاءات API أدناه
- طابق `sender` في قاعدة الدفع مع MXID الكامل لمستخدم الروبوت

1. قم بتكوين OpenClaw لاستخدام المعاينات الهادئة:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. تأكد من أن حساب المستلم يتلقى بالفعل إشعارات دفع Matrix العادية. قواعد المعاينة الهادئة
   لا تعمل إلا إذا كان لدى هذا المستخدم بالفعل أجهزة/خدمات دفع عاملة.

3. احصل على رمز وصول المستخدم المستلم.
   - استخدم رمز المستخدم المستقبل، وليس رمز الروبوت.
   - عادةً ما تكون إعادة استخدام رمز جلسة عميل موجودة هي الأسهل.
   - إذا احتجت إلى إصدار رمز جديد، يمكنك تسجيل الدخول عبر Matrix Client-Server API القياسي:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. تحقق من أن حساب المستلم يحتوي بالفعل على pushers:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

إذا لم يُرجع هذا أي pushers/أجهزة نشطة، فأصلح إشعارات Matrix العادية أولًا قبل إضافة
قاعدة OpenClaw أدناه.

يضع OpenClaw علامة على تعديلات المعاينة النهائية النصية فقط باستخدام:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. أنشئ قاعدة دفع override لكل حساب مستلم يجب أن يتلقى هذه الإشعارات:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

استبدل هذه القيم قبل تشغيل الأمر:

- `https://matrix.example.org`: عنوان URL الأساسي للخادم المنزلي الخاص بك
- `$USER_ACCESS_TOKEN`: رمز وصول المستخدم المستلم
- `openclaw-finalized-preview-botname`: معرّف قاعدة فريد لهذا الروبوت لهذا المستخدم المستلم
- `@bot:example.org`: MXID روبوت OpenClaw Matrix لديك، وليس MXID المستخدم المستلم

مهم لإعدادات الروبوتات المتعددة:

- تُفهرس قواعد الدفع بواسطة `ruleId`. تؤدي إعادة تشغيل `PUT` على نفس معرّف القاعدة إلى تحديث تلك القاعدة.
- إذا كان يجب على مستخدم مستلم واحد تلقي إشعارات من عدة حسابات روبوت OpenClaw Matrix، فأنشئ قاعدة واحدة لكل روبوت مع معرّف قاعدة فريد لكل مطابقة `sender`.
- نمط بسيط هو `openclaw-finalized-preview-<botname>`، مثل `openclaw-finalized-preview-ops` أو `openclaw-finalized-preview-support`.

يتم تقييم القاعدة مقابل مُرسل الحدث:

- قم بالمصادقة باستخدام رمز المستخدم المستلم
- طابق `sender` مع MXID روبوت OpenClaw

6. تحقق من وجود القاعدة:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. اختبر ردًا مبثوثًا. في الوضع الهادئ، يجب أن تعرض الغرفة معاينة مسودة هادئة وأن
   يرسل التعديل النهائي في المكان إشعارًا واحدًا عند اكتمال الكتلة أو الدور.

إذا احتجت إلى إزالة القاعدة لاحقًا، فاحذف معرّف القاعدة نفسه باستخدام رمز المستخدم المستلم:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

ملاحظات:

- أنشئ القاعدة باستخدام رمز وصول المستخدم المستلم، وليس رمز الروبوت.
- تُدرج قواعد `override` الجديدة المعرفة من قبل المستخدم قبل قواعد الكتم الافتراضية، لذلك لا حاجة إلى معلمة ترتيب إضافية.
- يؤثر هذا فقط في تعديلات المعاينة النصية فقط التي يستطيع OpenClaw إنهاءها بأمان في مكانها. ما تزال بدائل الوسائط وبدائل المعاينة القديمة تستخدم تسليم Matrix العادي.
- إذا أظهر `GET /_matrix/client/v3/pushers` عدم وجود pushers، فهذا يعني أن المستخدم لا يملك بعد تسليم دفع Matrix عاملًا لهذا الحساب/الجهاز.

#### Synapse

بالنسبة إلى Synapse، يكون الإعداد أعلاه كافيًا عادةً بحد ذاته:

- لا يلزم إجراء تغيير خاص في `homeserver.yaml` لإشعارات معاينة OpenClaw النهائية.
- إذا كانت عملية نشر Synapse لديك ترسل بالفعل إشعارات دفع Matrix العادية، فإن رمز المستخدم + استدعاء `pushrules` أعلاه هما خطوة الإعداد الأساسية.
- إذا كنت تشغّل Synapse خلف وكيل عكسي أو workers، فتأكد من أن `/_matrix/client/.../pushrules/` يصل إلى Synapse بشكل صحيح.
- إذا كنت تشغّل Synapse workers، فتأكد من سلامة pushers. تتم معالجة تسليم الدفع بواسطة العملية الرئيسية أو `synapse.app.pusher` / عمال pusher المهيئين.

#### Tuwunel

بالنسبة إلى Tuwunel، استخدم نفس تدفق الإعداد واستدعاء Push-rule API الموضح أعلاه:

- لا يلزم تكوين خاص بـ Tuwunel لعلامة المعاينة النهائية نفسها.
- إذا كانت إشعارات Matrix العادية تعمل بالفعل لذلك المستخدم، فإن رمز المستخدم + استدعاء `pushrules` أعلاه هما خطوة الإعداد الأساسية.
- إذا بدا أن الإشعارات تختفي بينما يكون المستخدم نشطًا على جهاز آخر، فتحقق مما إذا كان `suppress_push_when_active` ممكّنًا. أضاف Tuwunel هذا الخيار في Tuwunel 1.4.2 بتاريخ 12 سبتمبر 2025، ويمكنه عمدًا كتم الإشعارات إلى الأجهزة الأخرى بينما يكون أحد الأجهزة نشطًا.

## التشفير والتحقق

في الغرف المشفرة (E2EE)، تستخدم أحداث الصور الصادرة `thumbnail_file` بحيث تُشفَّر معاينات الصور إلى جانب المرفق الكامل. ولا تزال الغرف غير المشفرة تستخدم `thumbnail_url` العادي. لا حاجة إلى أي تكوين — تكتشف الإضافة حالة E2EE تلقائيًا.

### غرف روبوت إلى روبوت

بشكل افتراضي، يتم تجاهل رسائل Matrix الواردة من حسابات OpenClaw Matrix الأخرى المكوّنة.

استخدم `allowBots` عندما تريد عمدًا حركة مرور Matrix بين الوكلاء:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- يقبل `allowBots: true` الرسائل من حسابات روبوت Matrix الأخرى المكوّنة في الغرف والرسائل المباشرة المسموح بها.
- يقبل `allowBots: "mentions"` هذه الرسائل فقط عندما تذكر هذا الروبوت بوضوح في الغرف. ولا تزال الرسائل المباشرة مسموحًا بها.
- يتجاوز `groups.<room>.allowBots` الإعداد على مستوى الحساب لغرفة واحدة.
- لا يزال OpenClaw يتجاهل الرسائل من نفس معرّف مستخدم Matrix لتجنب حلقات الرد الذاتي.
- لا يوفّر Matrix هنا علامة روبوت أصلية؛ يتعامل OpenClaw مع "المرسل من روبوت" على أنه "مرسل بواسطة حساب Matrix مكوَّن آخر على بوابة OpenClaw هذه".

استخدم قوائم سماح صارمة للغرف ومتطلبات الذكر عند تمكين حركة المرور من روبوت إلى روبوت في الغرف المشتركة.

تمكين التشفير:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

تحقق من حالة التحقق:

```bash
openclaw matrix verify status
```

حالة مفصلة (تشخيصات كاملة):

```bash
openclaw matrix verify status --verbose
```

تضمين مفتاح الاسترداد المخزن في المخرجات القابلة للقراءة آليًا:

```bash
openclaw matrix verify status --include-recovery-key --json
```

تهيئة cross-signing وحالة التحقق:

```bash
openclaw matrix verify bootstrap
```

دعم الحسابات المتعددة: استخدم `channels.matrix.accounts` مع بيانات اعتماد لكل حساب و`name` اختياري. راجع [مرجع التكوين](/ar/gateway/configuration-reference#multi-account-all-channels) للاطلاع على النمط المشترك.

تشخيصات bootstrap مفصلة:

```bash
openclaw matrix verify bootstrap --verbose
```

فرض إعادة تعيين جديدة لهوية cross-signing قبل التهيئة:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

تحقق من هذا الجهاز باستخدام مفتاح استرداد:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

تفاصيل التحقق من الجهاز بشكل مفصل:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

تحقق من سلامة النسخ الاحتياطي لمفاتيح الغرف:

```bash
openclaw matrix verify backup status
```

تشخيصات سلامة النسخ الاحتياطي بشكل مفصل:

```bash
openclaw matrix verify backup status --verbose
```

استعادة مفاتيح الغرف من النسخة الاحتياطية على الخادم:

```bash
openclaw matrix verify backup restore
```

تشخيصات الاستعادة بشكل مفصل:

```bash
openclaw matrix verify backup restore --verbose
```

احذف النسخة الاحتياطية الحالية على الخادم وأنشئ خط أساس جديدًا للنسخ الاحتياطي. إذا تعذر تحميل
مفتاح النسخة الاحتياطية المخزّن بشكل نظيف، فإن إعادة التعيين هذه يمكنها أيضًا إعادة إنشاء التخزين السري بحيث
تتمكن عمليات البدء البارد المستقبلية من تحميل مفتاح النسخة الاحتياطية الجديد:

```bash
openclaw matrix verify backup reset --yes
```

تكون جميع أوامر `verify` موجزة افتراضيًا (بما في ذلك تسجيل SDK الداخلي الهادئ) وتعرض تشخيصات مفصلة فقط مع `--verbose`.
استخدم `--json` للحصول على مخرجات كاملة قابلة للقراءة آليًا عند كتابة البرامج النصية.

في إعدادات الحسابات المتعددة، تستخدم أوامر Matrix CLI حساب Matrix الافتراضي الضمني ما لم تمرر `--account <id>`.
إذا قمت بتكوين عدة حسابات مسماة، فاضبط `channels.matrix.defaultAccount` أولًا وإلا فستتوقف عمليات CLI الضمنية هذه وتطلب منك اختيار حساب بشكل صريح.
استخدم `--account` كلما أردت أن تستهدف عمليات التحقق أو الجهاز حسابًا مسمّى بشكل صريح:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

عندما يكون التشفير معطّلًا أو غير متاح لحساب مسمّى، تشير تحذيرات Matrix وأخطاء التحقق إلى مفتاح تكوين ذلك الحساب، مثل `channels.matrix.accounts.assistant.encryption`.

### ما معنى "تم التحقق"

يعتبر OpenClaw جهاز Matrix هذا متحققًا منه فقط عندما يتم التحقق منه بواسطة هوية cross-signing الخاصة بك.
عمليًا، يكشف `openclaw matrix verify status --verbose` عن ثلاث إشارات ثقة:

- `Locally trusted`: هذا الجهاز موثوق به من قبل العميل الحالي فقط
- `Cross-signing verified`: يفيد SDK بأن الجهاز تم التحقق منه عبر cross-signing
- `Signed by owner`: هذا الجهاز موقّع بواسطة مفتاح self-signing الخاص بك

تصبح `Verified by owner` مساوية لـ `yes` فقط عند وجود تحقق cross-signing أو توقيع المالك.
الثقة المحلية وحدها ليست كافية لكي يعامل OpenClaw الجهاز على أنه متحقق بالكامل.

### ماذا يفعل bootstrap

الأمر `openclaw matrix verify bootstrap` هو أمر الإصلاح والإعداد لحسابات Matrix المشفرة.
ويقوم بكل ما يلي بالترتيب:

- يهيئ التخزين السري، مع إعادة استخدام مفتاح استرداد موجود عندما يكون ذلك ممكنًا
- يهيئ cross-signing ويرفع مفاتيح cross-signing العامة المفقودة
- يحاول تعليم الجهاز الحالي وتوقيعه عبر cross-signing
- ينشئ نسخة احتياطية جديدة على جانب الخادم لمفاتيح الغرف إذا لم تكن موجودة بالفعل

إذا كان الخادم المنزلي يتطلب مصادقة تفاعلية لرفع مفاتيح cross-signing، فسيحاول OpenClaw الرفع أولًا دون مصادقة، ثم باستخدام `m.login.dummy`، ثم باستخدام `m.login.password` عندما يكون `channels.matrix.password` مُكوَّنًا.

استخدم `--force-reset-cross-signing` فقط عندما تريد عمدًا تجاهل هوية cross-signing الحالية وإنشاء هوية جديدة.

إذا كنت تريد عمدًا تجاهل النسخة الاحتياطية الحالية لمفاتيح الغرف والبدء بخط أساس جديد
للنسخ الاحتياطي للرسائل المستقبلية، فاستخدم `openclaw matrix verify backup reset --yes`.
افعل هذا فقط عندما تقبل أن يظل السجل القديم المشفر غير القابل للاسترداد
غير متاح وأن OpenClaw قد يعيد إنشاء التخزين السري إذا تعذر تحميل
سر النسخة الاحتياطية الحالي بأمان.

### خط أساس جديد للنسخ الاحتياطي

إذا كنت تريد الإبقاء على الرسائل المشفرة المستقبلية تعمل وتقبل فقدان السجل القديم غير القابل للاسترداد، فشغّل هذه الأوامر بالترتيب:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

أضف `--account <id>` إلى كل أمر عندما تريد استهداف حساب Matrix مسمى بشكل صريح.

### سلوك بدء التشغيل

عندما يكون `encryption: true`، يضبط Matrix القيمة الافتراضية لـ `startupVerification` على `"if-unverified"`.
عند بدء التشغيل، إذا كان هذا الجهاز لا يزال غير متحقق منه، فسيطلب Matrix التحقق الذاتي في عميل Matrix آخر،
ويتخطى الطلبات المكررة طالما أن أحدها قيد الانتظار بالفعل، ويطبق فترة تهدئة محلية قبل إعادة المحاولة بعد عمليات إعادة التشغيل.
تحاول الطلبات الفاشلة مرة أخرى في وقت أقرب من إنشاء الطلبات الناجح افتراضيًا.
عيّن `startupVerification: "off"` لتعطيل طلبات بدء التشغيل التلقائية، أو اضبط `startupVerificationCooldownHours`
إذا أردت نافذة إعادة محاولة أقصر أو أطول.

ينفذ بدء التشغيل أيضًا بشكل تلقائي تمريرة bootstrap محافظة للتشفير.
تحاول هذه التمريرة أولًا إعادة استخدام التخزين السري الحالي وهوية cross-signing الحالية، وتتجنب إعادة تعيين cross-signing ما لم تُشغِّل تدفق إصلاح bootstrap صريحًا.

إذا وجد بدء التشغيل حالة bootstrap تالفة وكان `channels.matrix.password` مُكوَّنًا، فيمكن لـ OpenClaw محاولة مسار إصلاح أكثر صرامة.
إذا كان الجهاز الحالي موقّعًا بالفعل من قبل المالك، فسيحافظ OpenClaw على تلك الهوية بدلًا من إعادة تعيينها تلقائيًا.

الترقية من إضافة Matrix العامة السابقة:

- يعيد OpenClaw تلقائيًا استخدام نفس حساب Matrix، ورمز الوصول، وهوية الجهاز متى أمكن.
- قبل تشغيل أي تغييرات ترحيل Matrix قابلة للتنفيذ، ينشئ OpenClaw أو يعيد استخدام لقطة استرداد ضمن `~/Backups/openclaw-migrations/`.
- إذا كنت تستخدم عدة حسابات Matrix، فاضبط `channels.matrix.defaultAccount` قبل الترقية من تخطيط التخزين المسطح القديم حتى يعرف OpenClaw أي حساب يجب أن يتلقى تلك الحالة القديمة المشتركة.
- إذا كانت الإضافة السابقة تخزّن مفتاح فك تشفير نسخة احتياطية لمفاتيح غرف Matrix محليًا، فسيقوم بدء التشغيل أو `openclaw doctor --fix` باستيراده تلقائيًا إلى تدفق مفتاح الاسترداد الجديد.
- إذا تغيّر رمز وصول Matrix بعد تحضير الترحيل، فإن بدء التشغيل يفحص الآن جذور تخزين تجزئة الرمز الشقيقة بحثًا عن حالة استعادة قديمة معلّقة قبل التخلي عن الاستعادة التلقائية للنسخ الاحتياطي.
- إذا تغيّر رمز وصول Matrix لاحقًا لنفس الحساب والخادم المنزلي والمستخدم، فإن OpenClaw يفضّل الآن إعادة استخدام جذر تخزين تجزئة الرمز الأكثر اكتمالًا بدلًا من البدء من دليل حالة Matrix فارغ.
- عند بدء تشغيل البوابة التالي، تُستعاد مفاتيح الغرف المنسوخة احتياطيًا تلقائيًا إلى مخزن التشفير الجديد.
- إذا كانت الإضافة القديمة تحتوي على مفاتيح غرف محلية فقط لم تُنسخ احتياطيًا مطلقًا، فسيحذر OpenClaw بوضوح. لا يمكن تصدير هذه المفاتيح تلقائيًا من مخزن التشفير rust السابق، لذا قد يظل بعض السجل القديم المشفر غير متاح حتى تتم استعادته يدويًا.
- راجع [ترحيل Matrix](/ar/install/migrating-matrix) للاطلاع على تدفق الترقية الكامل، والقيود، وأوامر الاسترداد، ورسائل الترحيل الشائعة.

تُنظَّم حالة وقت التشغيل المشفرة ضمن جذور تجزئة رموز لكل حساب ولكل مستخدم في
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
يحتوي هذا الدليل على مخزن المزامنة (`bot-storage.json`)، ومخزن التشفير (`crypto/`)،
وملف مفتاح الاسترداد (`recovery-key.json`)، ولقطة IndexedDB (`crypto-idb-snapshot.json`)،
وروابط الخيوط (`thread-bindings.json`)، وحالة التحقق عند بدء التشغيل (`startup-verification.json`)
عندما تكون هذه الميزات قيد الاستخدام.
عندما يتغير الرمز لكن تبقى هوية الحساب نفسها، يعيد OpenClaw استخدام أفضل
جذر موجود لذلك الثلاثي الحساب/الخادم المنزلي/المستخدم بحيث تظل حالة المزامنة السابقة، وحالة التشفير، وروابط الخيوط،
وحالة التحقق عند بدء التشغيل مرئية.

### نموذج مخزن تشفير Node

تستخدم E2EE الخاصة بـ Matrix في هذه الإضافة مسار تشفير Rust الرسمي من `matrix-js-sdk` في Node.
ويتوقع هذا المسار وجود استمرارية مدعومة بـ IndexedDB عندما تريد لحالة التشفير أن تبقى بعد إعادة التشغيل.

يوفّر OpenClaw ذلك حاليًا في Node من خلال:

- استخدام `fake-indexeddb` كطبقة API لـ IndexedDB المتوقعة من قبل SDK
- استعادة محتويات Rust crypto IndexedDB من `crypto-idb-snapshot.json` قبل `initRustCrypto`
- حفظ محتويات IndexedDB المحدّثة مرة أخرى إلى `crypto-idb-snapshot.json` بعد التهيئة وأثناء وقت التشغيل
- تسلسل استعادة اللقطة والحفظ مقابل `crypto-idb-snapshot.json` باستخدام قفل ملف استشاري بحيث لا يتسابق حفظ وقت تشغيل البوابة وصيانة CLI على ملف اللقطة نفسه

هذا تكييف للتوافق/التخزين، وليس تنفيذ تشفير مخصصًا.
ملف اللقطة هو حالة وقت تشغيل حساسة ويتم تخزينه بأذونات ملفات مقيدة.
في نموذج أمان OpenClaw، يكون مضيف البوابة ودليل حالة OpenClaw المحلي داخل حدود المشغّل الموثوق بها بالفعل، لذا فهذه في المقام الأول مسألة متانة تشغيلية وليست حد ثقة بعيد منفصل.

تحسين مخطط له:

- إضافة دعم SecretRef لمواد مفاتيح Matrix المستمرة بحيث يمكن توفير مفاتيح الاسترداد وأسرار تشفير المخزن ذات الصلة من مزودي أسرار OpenClaw بدلًا من الملفات المحلية فقط

## إدارة الملف الشخصي

حدّث الملف الشخصي الذاتي في Matrix للحساب المحدد باستخدام:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

أضف `--account <id>` عندما تريد استهداف حساب Matrix مسمى بشكل صريح.

يقبل Matrix عناوين URL للصورة الرمزية `mxc://` مباشرة. عندما تمرر عنوان URL للصورة الرمزية من نوع `http://` أو `https://`، يرفعها OpenClaw إلى Matrix أولًا ثم يخزن عنوان `mxc://` المحلول مرة أخرى في `channels.matrix.avatarUrl` (أو التجاوز المحدد للحساب).

## إشعارات التحقق التلقائية

ينشر Matrix الآن إشعارات دورة حياة التحقق مباشرة داخل غرفة التحقق الصارمة للرسائل المباشرة كرسائل `m.notice`.
ويشمل ذلك:

- إشعارات طلب التحقق
- إشعارات جاهزية التحقق (مع إرشاد صريح "تحقق بواسطة emoji")
- إشعارات بدء التحقق واكتماله
- تفاصيل SAS (emoji والأرقام العشرية) عندما تكون متاحة

تتم متابعة طلبات التحقق الواردة من عميل Matrix آخر وقبولها تلقائيًا بواسطة OpenClaw.
وبالنسبة إلى تدفقات التحقق الذاتي، يبدأ OpenClaw أيضًا تدفق SAS تلقائيًا عندما يصبح التحقق بواسطة emoji متاحًا ويؤكد جانبه الخاص.
أما بالنسبة إلى طلبات التحقق من مستخدم/جهاز Matrix آخر، فيقبل OpenClaw الطلب تلقائيًا ثم ينتظر استمرار تدفق SAS بشكل طبيعي.
لا يزال يتعين عليك مقارنة emoji أو SAS العشري في عميل Matrix لديك وتأكيد "إنها متطابقة" هناك لإكمال التحقق.

لا يقبل OpenClaw تلقائيًا التدفقات الذاتية المكررة التي تم بدؤها ذاتيًا بشكل أعمى. يتخطى بدء التشغيل إنشاء طلب جديد عندما يكون طلب التحقق الذاتي معلقًا بالفعل.

لا تُمرَّر إشعارات بروتوكول/نظام التحقق إلى مسار دردشة الوكيل، لذلك لا تنتج `NO_REPLY`.

### نظافة الأجهزة

يمكن أن تتراكم أجهزة Matrix القديمة التي يديرها OpenClaw على الحساب وتجعل الثقة في الغرف المشفرة أصعب في الفهم.
اعرضها باستخدام:

```bash
openclaw matrix devices list
```

أزل أجهزة OpenClaw Matrix القديمة غير المستخدمة باستخدام:

```bash
openclaw matrix devices prune-stale
```

### إصلاح الغرفة المباشرة

إذا أصبحت حالة الرسائل المباشرة غير متزامنة، فقد ينتهي الأمر بـ OpenClaw إلى امتلاك تعيينات `m.direct` قديمة تشير إلى غرف فردية قديمة بدلًا من الرسالة المباشرة النشطة. افحص التعيين الحالي لنظير باستخدام:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

أصلحه باستخدام:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

يبقي الإصلاح المنطق الخاص بـ Matrix داخل الإضافة:

- يفضّل رسالة مباشرة صارمة 1:1 معيّنة بالفعل في `m.direct`
- وإلا فإنه يعود إلى أي رسالة مباشرة صارمة 1:1 منضم إليها حاليًا مع ذلك المستخدم
- إذا لم توجد رسالة مباشرة سليمة، فإنه ينشئ غرفة مباشرة جديدة ويعيد كتابة `m.direct` لتشير إليها

لا يحذف تدفق الإصلاح الغرف القديمة تلقائيًا. فهو يختار فقط الرسالة المباشرة السليمة ويحدّث التعيين بحيث تستهدف عمليات الإرسال الجديدة في Matrix، وإشعارات التحقق، وتدفقات الرسائل المباشرة الأخرى الغرفة الصحيحة مرة أخرى.

## الخيوط

يدعم Matrix خيوط Matrix الأصلية لكل من الردود التلقائية وعمليات الإرسال بأداة الرسائل.

- يحافظ `dm.sessionScope: "per-user"` (الافتراضي) على توجيه الرسائل المباشرة في Matrix في نطاق المرسل، بحيث يمكن لعدة غرف رسائل مباشرة مشاركة جلسة واحدة عندما تُحل إلى النظير نفسه.
- يعزل `dm.sessionScope: "per-room"` كل غرفة رسائل مباشرة في Matrix في مفتاح جلسة خاص بها مع الاستمرار في استخدام فحوصات المصادقة وقائمة السماح العادية الخاصة بالرسائل المباشرة.
- تظل روابط محادثات Matrix الصريحة لها الأولوية على `dm.sessionScope`، لذلك تحتفظ الغرف والخيوط المرتبطة بجلسة الهدف المختارة.
- يبقي `threadReplies: "off"` الردود على المستوى الأعلى ويبقي الرسائل الواردة داخل الخيوط على جلسة الأصل.
- يرد `threadReplies: "inbound"` داخل خيط فقط عندما تكون الرسالة الواردة موجودة بالفعل في ذلك الخيط.
- يحافظ `threadReplies: "always"` على ردود الغرف داخل خيط متجذر في الرسالة المُحفِّزة ويوجه تلك المحادثة عبر الجلسة المطابقة ذات النطاق الخاص بالخيط من أول رسالة محفِّزة.
- يتجاوز `dm.threadReplies` الإعداد الأعلى مستوى للرسائل المباشرة فقط. على سبيل المثال، يمكنك إبقاء خيوط الغرف معزولة مع إبقاء الرسائل المباشرة مسطحة.
- تتضمن الرسائل الواردة داخل الخيوط الرسالة الجذرية للخيط كسياق إضافي للوكيل.
- ترث عمليات الإرسال بأداة الرسائل الآن خيط Matrix الحالي تلقائيًا عندما يكون الهدف هو الغرفة نفسها أو هدف مستخدم الرسالة المباشرة نفسه، ما لم يتم توفير `threadId` صراحةً.
- لا يُفعَّل إعادة استخدام هدف مستخدم الرسالة المباشرة في الجلسة نفسها إلا عندما تثبت بيانات تعريف الجلسة الحالية النظير نفسه في الرسالة المباشرة على حساب Matrix نفسه؛ وإلا يعود OpenClaw إلى التوجيه العادي في نطاق المستخدم.
- عندما يرى OpenClaw أن غرفة رسالة مباشرة في Matrix تتصادم مع غرفة رسالة مباشرة أخرى على جلسة Matrix DM المشتركة نفسها، فإنه ينشر `m.notice` لمرة واحدة في تلك الغرفة مع مخرج الهروب `/focus` عندما تكون روابط الخيوط ممكّنة ومع تلميح `dm.sessionScope`.
- تُدعَم روابط الخيوط وقت التشغيل في Matrix. تعمل الآن `/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` و`/acp spawn` المرتبط بالخيط في غرف Matrix ورسائلها المباشرة.
- ينشئ `/focus` على مستوى الغرفة/الرسالة المباشرة الأعلى في Matrix خيط Matrix جديدًا ويربطه بالجلسة المستهدفة عندما يكون `threadBindings.spawnSubagentSessions=true`.
- يؤدي تشغيل `/focus` أو `/acp spawn --thread here` داخل خيط Matrix موجود إلى ربط ذلك الخيط الحالي بدلًا من ذلك.

## روابط محادثات ACP

يمكن تحويل غرف Matrix والرسائل المباشرة وخيوط Matrix الموجودة إلى مساحات عمل ACP دائمة دون تغيير سطح الدردشة.

تدفق سريع للمشغّل:

- شغّل `/acp spawn codex --bind here` داخل رسالة Matrix المباشرة أو الغرفة أو الخيط الموجود الذي تريد الاستمرار في استخدامه.
- في رسالة Matrix مباشرة أو غرفة على المستوى الأعلى، يبقى سطح الدردشة هو الرسالة المباشرة/الغرفة الحالية، وتُوجَّه الرسائل المستقبلية إلى جلسة ACP التي تم إنشاؤها.
- داخل خيط Matrix موجود، يربط `--bind here` ذلك الخيط الحالي في مكانه.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الرابط.

ملاحظات:

- لا ينشئ `--bind here` خيط Matrix فرعيًا.
- لا يكون `threadBindings.spawnAcpSessions` مطلوبًا إلا مع `/acp spawn --thread auto|here`، عندما يحتاج OpenClaw إلى إنشاء خيط Matrix فرعي أو ربطه.

### تكوين ربط الخيط

يرث Matrix القيم الافتراضية العامة من `session.threadBindings`، ويدعم أيضًا تجاوزات لكل قناة:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

إشارات إنشاء الجلسات المرتبطة بالخيط في Matrix اختيارية:

- عيّن `threadBindings.spawnSubagentSessions: true` للسماح للأمر `/focus` على المستوى الأعلى بإنشاء خيوط Matrix جديدة وربطها.
- عيّن `threadBindings.spawnAcpSessions: true` للسماح للأمر `/acp spawn --thread auto|here` بربط جلسات ACP بخيوط Matrix.

## التفاعلات

يدعم Matrix إجراءات التفاعل الصادرة، وإشعارات التفاعل الواردة، وتفاعلات الإقرار الواردة.

- يتم تقييد أدوات التفاعل الصادرة بواسطة `channels["matrix"].actions.reactions`.
- تضيف `react` تفاعلًا إلى حدث Matrix محدد.
- تسرد `reactions` الملخص الحالي للتفاعلات لحدث Matrix محدد.
- يؤدي `emoji=""` إلى إزالة تفاعلات حساب الروبوت نفسه على ذلك الحدث.
- يؤدي `remove: true` إلى إزالة تفاعل emoji المحدد فقط من حساب الروبوت.

يُحل نطاق تفاعلات الإقرار وفق ترتيب OpenClaw القياسي:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- الرجوع إلى emoji هوية الوكيل

يُحل نطاق تفاعل الإقرار بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

يُحل وضع إشعارات التفاعل بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- الافتراضي: `own`

السلوك الحالي:

- يمرر `reactionNotifications: "own"` أحداث `m.reaction` المضافة عندما تستهدف رسائل Matrix التي أنشأها الروبوت.
- يعطّل `reactionNotifications: "off"` أحداث نظام التفاعل.
- لا تزال عمليات إزالة التفاعل غير مُصنَّعة كأحداث نظام لأن Matrix يعرضها كعمليات حذف redactions، وليس كإزالات مستقلة لـ `m.reaction`.

## سياق السجل

- يتحكم `channels.matrix.historyLimit` في عدد رسائل الغرف الأخيرة المضمّنة كـ `InboundHistory` عندما تؤدي رسالة غرفة Matrix إلى تشغيل الوكيل.
- يرجع إلى `messages.groupChat.historyLimit`. إذا لم يكن أي منهما معيّنًا، فالقيمة الافتراضية الفعالة هي `0`، لذلك لا يتم تخزين رسائل الغرف المقيّدة بالذكر مؤقتًا. عيّن `0` للتعطيل.
- يقتصر سجل غرف Matrix على الغرف فقط. وتستمر الرسائل المباشرة في استخدام سجل الجلسة العادي.
- سجل غرف Matrix هو سجل معلق فقط: يخزن OpenClaw رسائل الغرف التي لم تفعّل ردًا بعد، ثم يلتقط تلك النافذة عندما يصل ذكر أو مُشغِّل آخر.
- لا تُضمَّن رسالة التشغيل الحالية في `InboundHistory`؛ بل تبقى في الجسم الرئيسي الوارد لذلك الدور.
- تعيد محاولات الحدث نفسه في Matrix استخدام لقطة السجل الأصلية بدلًا من الانجراف إلى رسائل غرف أحدث.

## رؤية السياق

يدعم Matrix عنصر التحكم المشترك `contextVisibility` للسياق الإضافي للغرفة مثل نصوص الردود المجلبة، وجذور الخيوط، والسجل المعلّق.

- `contextVisibility: "all"` هو الافتراضي. يتم الاحتفاظ بالسياق الإضافي كما تم استلامه.
- يرشح `contextVisibility: "allowlist"` السياق الإضافي إلى المرسلين المسموح بهم بواسطة فحوصات قائمة السماح النشطة للغرفة/المستخدم.
- يتصرف `contextVisibility: "allowlist_quote"` مثل `allowlist`، لكنه يحتفظ أيضًا برد مقتبس صريح واحد.

يؤثر هذا الإعداد في رؤية السياق الإضافي، وليس في ما إذا كانت الرسالة الواردة نفسها يمكن أن تؤدي إلى رد.
ولا يزال تفويض التشغيل يأتي من `groupPolicy` و`groups` و`groupAllowFrom` وإعدادات سياسة الرسائل المباشرة.

## مثال على سياسة الرسائل المباشرة والغرف

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

راجع [المجموعات](/ar/channels/groups) لمعرفة سلوك تقييد الذكر وقائمة السماح.

مثال على الإقران لرسائل Matrix المباشرة:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

إذا استمر مستخدم Matrix غير معتمد في مراسلتك قبل الموافقة، فسيعيد OpenClaw استخدام رمز الإقران المعلّق نفسه وقد يرسل رد تذكير مرة أخرى بعد فترة تهدئة قصيرة بدلًا من إصدار رمز جديد.

راجع [الإقران](/ar/channels/pairing) لمعرفة تدفق إقران الرسائل المباشرة المشترك وتخطيط التخزين.

## موافقات Exec

يمكن أن يعمل Matrix كعميل موافقة أصلي لحساب Matrix. ولا تزال
عناصر التحكم الأصلية في توجيه الرسائل المباشرة/القنوات موجودة ضمن تكوين موافقة exec:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (اختياري؛ يرجع إلى `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

يجب أن يكون الموافقون معرّفات مستخدمي Matrix مثل `@owner:example.org`. يفعّل Matrix الموافقات الأصلية تلقائيًا عندما تكون `enabled` غير معيّنة أو تساوي `"auto"` ويمكن حل موافق واحد على الأقل. تستخدم موافقات Exec أولًا مجموعة الموافقين من `execApprovals.approvers` ويمكن أن ترجع إلى `channels.matrix.dm.allowFrom`. وتفوّض موافقات الإضافة عبر `channels.matrix.dm.allowFrom`. عيّن `enabled: false` لتعطيل Matrix كعميل موافقة أصلي بشكل صريح. وإلا فسترجع طلبات الموافقة إلى مسارات الموافقة المكوّنة الأخرى أو إلى سياسة الرجوع للموافقة.

يدعم التوجيه الأصلي في Matrix الآن نوعي الموافقة:

- تتحكم `channels.matrix.execApprovals.*` في وضع التوزيع الأصلي للرسائل المباشرة/القنوات لمطالبات موافقة Matrix.
- تستخدم موافقات Exec مجموعة الموافقين الخاصة بالتنفيذ من `execApprovals.approvers` أو `channels.matrix.dm.allowFrom`.
- تستخدم موافقات الإضافة قائمة سماح الرسائل المباشرة في Matrix من `channels.matrix.dm.allowFrom`.
- تنطبق اختصارات التفاعل في Matrix وتحديثات الرسائل على كل من موافقات exec وموافقات الإضافة.

قواعد التسليم:

- يرسل `target: "dm"` مطالبات الموافقة إلى الرسائل المباشرة للموافقين
- يرسل `target: "channel"` المطالبة مرة أخرى إلى غرفة Matrix أو الرسالة المباشرة الأصلية
- يرسل `target: "both"` إلى الرسائل المباشرة للموافقين وإلى غرفة Matrix أو الرسالة المباشرة الأصلية

تزرع مطالبات الموافقة في Matrix اختصارات التفاعل على رسالة الموافقة الأساسية:

- `✅` = السماح مرة واحدة
- `❌` = الرفض
- `♾️` = السماح دائمًا عندما يكون هذا القرار مسموحًا به بواسطة سياسة exec الفعالة

يمكن للموافقين التفاعل على تلك الرسالة أو استخدام أوامر الشرطة المائلة الاحتياطية: `/approve <id> allow-once` أو `/approve <id> allow-always` أو `/approve <id> deny`.

لا يمكن سوى للموافقين الذين تم حلهم أن يوافقوا أو يرفضوا. بالنسبة إلى موافقات exec، يتضمن التسليم عبر القناة نص الأمر، لذا فعّل `channel` أو `both` فقط في الغرف الموثوق بها.

تعيد مطالبات الموافقة في Matrix استخدام مخطط الموافقة الأساسي المشترك. ويتولى السطح الأصلي الخاص بـ Matrix التوجيه بين الغرف/الرسائل المباشرة، والتفاعلات، وسلوك إرسال/تحديث/حذف الرسائل لكل من موافقات exec وموافقات الإضافة.

تجاوز لكل حساب:

- `channels.matrix.accounts.<account>.execApprovals`

المستندات ذات الصلة: [موافقات Exec](/ar/tools/exec-approvals)

## مثال على الحسابات المتعددة

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

تعمل قيم `channels.matrix` ذات المستوى الأعلى كقيم افتراضية للحسابات المسماة ما لم يتجاوزها الحساب.
يمكنك تحديد نطاق إدخالات الغرف الموروثة لحساب Matrix واحد باستخدام `groups.<room>.account` (أو `rooms.<room>.account` القديم).
تبقى الإدخالات التي لا تحتوي على `account` مشتركة بين جميع حسابات Matrix، ولا تزال الإدخالات التي تحتوي على `account: "default"` تعمل عندما يكون الحساب الافتراضي مُكوَّنًا مباشرة في `channels.matrix.*` ذات المستوى الأعلى.
لا تنشئ قيم المصادقة الافتراضية الجزئية المشتركة حسابًا افتراضيًا ضمنيًا منفصلًا بحد ذاتها. لا يقوم OpenClaw بتركيب الحساب `default` ذي المستوى الأعلى إلا عندما يملك ذلك الافتراضي مصادقة حديثة (`homeserver` مع `accessToken`، أو `homeserver` مع `userId` و`password`)؛ ويمكن للحسابات المسماة أن تظل قابلة للاكتشاف من `homeserver` مع `userId` عندما تلبّي بيانات الاعتماد المخزنة مؤقتًا المصادقة لاحقًا.
إذا كان Matrix يحتوي بالفعل على حساب مسمى واحد فقط تمامًا، أو كان `defaultAccount` يشير إلى مفتاح حساب مسمى موجود، فإن ترقية الإصلاح/الإعداد من حساب واحد إلى حسابات متعددة تحافظ على ذلك الحساب بدلًا من إنشاء إدخال `accounts.default` جديد. لا تنتقل إلى ذلك الحساب المُرقّى إلا مفاتيح مصادقة/تهيئة Matrix؛ وتبقى مفاتيح سياسات التسليم المشتركة في المستوى الأعلى.
عيّن `defaultAccount` عندما تريد من OpenClaw أن يفضّل حساب Matrix مسمى واحدًا للتوجيه الضمني، والفحص، وعمليات CLI.
إذا قمت بتكوين عدة حسابات مسماة، فعيّن `defaultAccount` أو مرر `--account <id>` لأوامر CLI التي تعتمد على اختيار الحساب الضمني.
مرر `--account <id>` إلى `openclaw matrix verify ...` و`openclaw matrix devices ...` عندما تريد تجاوز ذلك الاختيار الضمني لأمر واحد.

## الخوادم المنزلية الخاصة/الشبكة المحلية

بشكل افتراضي، يمنع OpenClaw الخوادم المنزلية الخاصة/الداخلية لـ Matrix للحماية من SSRF ما لم
توافق صراحةً على ذلك لكل حساب.

إذا كان الخادم المنزلي يعمل على localhost أو عنوان IP لشبكة LAN/Tailscale أو اسم مضيف داخلي، فقم بتمكين
`network.dangerouslyAllowPrivateNetwork` لذلك الحساب في Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

مثال إعداد عبر CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

يسمح هذا الاختيار فقط بالأهداف الخاصة/الداخلية الموثوق بها. أما الخوادم المنزلية العامة غير المشفرة مثل
`http://matrix.example.org:8008` فتبقى محظورة. فضّل `https://` كلما أمكن.

## تمرير حركة مرور Matrix عبر وكيل

إذا كانت عملية نشر Matrix لديك تحتاج إلى وكيل HTTP(S) صادر صريح، فعيّن `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

يمكن للحسابات المسماة تجاوز الافتراضي ذي المستوى الأعلى باستخدام `channels.matrix.accounts.<id>.proxy`.
يستخدم OpenClaw إعداد الوكيل نفسه لكل من حركة مرور Matrix وقت التشغيل وفحوصات حالة الحساب.

## حل الأهداف

يقبل Matrix صيغ الأهداف التالية في أي مكان يطلب منك OpenClaw فيه هدف غرفة أو مستخدم:

- المستخدمون: `@user:server` أو `user:@user:server` أو `matrix:user:@user:server`
- الغرف: `!room:server` أو `room:!room:server` أو `matrix:room:!room:server`
- الأسماء المستعارة: `#alias:server` أو `channel:#alias:server` أو `matrix:channel:#alias:server`

يستخدم البحث المباشر في الدليل حساب Matrix الذي تم تسجيل الدخول به:

- تستعلم عمليات البحث عن المستخدمين دليل مستخدمي Matrix على ذلك الخادم المنزلي.
- تقبل عمليات البحث عن الغرف معرّفات الغرف والأسماء المستعارة الصريحة مباشرة، ثم ترجع إلى البحث في أسماء الغرف المنضم إليها لذلك الحساب.
- البحث في أسماء الغرف المنضم إليها هو أفضل جهد. إذا تعذر حل اسم الغرفة إلى معرّف أو اسم مستعار، يتم تجاهله بواسطة حل قائمة السماح وقت التشغيل.

## مرجع التكوين

- `enabled`: تمكين القناة أو تعطيلها.
- `name`: تسمية اختيارية للحساب.
- `defaultAccount`: معرّف الحساب المفضل عند تكوين عدة حسابات Matrix.
- `homeserver`: عنوان URL للخادم المنزلي، على سبيل المثال `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: يسمح لهذا الحساب في Matrix بالاتصال بالخوادم المنزلية الخاصة/الداخلية. قم بتمكين هذا عندما يُحل الخادم المنزلي إلى `localhost` أو عنوان IP لشبكة LAN/Tailscale أو مضيف داخلي مثل `matrix-synapse`.
- `proxy`: عنوان URL اختياري لوكيل HTTP(S) لحركة مرور Matrix. يمكن للحسابات المسماة تجاوز الافتراضي ذي المستوى الأعلى باستخدام `proxy` خاص بها.
- `userId`: معرّف مستخدم Matrix الكامل، على سبيل المثال `@bot:example.org`.
- `accessToken`: رمز وصول للمصادقة المعتمدة على الرموز. تُدعم القيم النصية الصريحة وقيم SecretRef لكل من `channels.matrix.accessToken` و`channels.matrix.accounts.<id>.accessToken` عبر مزودي env/file/exec. راجع [إدارة الأسرار](/ar/gateway/secrets).
- `password`: كلمة مرور لتسجيل الدخول المعتمد على كلمة المرور. تُدعم القيم النصية الصريحة وقيم SecretRef.
- `deviceId`: معرّف جهاز Matrix صريح.
- `deviceName`: اسم عرض الجهاز لتسجيل الدخول بكلمة المرور.
- `avatarUrl`: عنوان URL للصورة الرمزية الذاتية المخزّن لمزامنة الملف الشخصي وتحديثات `set-profile`.
- `initialSyncLimit`: حد أحداث المزامنة عند بدء التشغيل.
- `encryption`: تمكين E2EE.
- `allowlistOnly`: فرض سلوك قائمة السماح فقط للرسائل المباشرة والغرف.
- `allowBots`: السماح برسائل من حسابات OpenClaw Matrix الأخرى المكوّنة (`true` أو `"mentions"`).
- `groupPolicy`: ‏`open` أو `allowlist` أو `disabled`.
- `contextVisibility`: وضع رؤية سياق الغرفة الإضافي (`all` أو `allowlist` أو `allowlist_quote`).
- `groupAllowFrom`: قائمة سماح لمعرّفات المستخدمين لحركة مرور الغرف.
- يجب أن تكون إدخالات `groupAllowFrom` معرّفات مستخدمي Matrix كاملة. تُتجاهل الأسماء غير المحلولة وقت التشغيل.
- `historyLimit`: الحد الأقصى لرسائل الغرف التي تُضمَّن كسياق سجل للمجموعة. يرجع إلى `messages.groupChat.historyLimit`؛ وإذا لم يكن أي منهما معيّنًا، فالقيمة الافتراضية الفعالة هي `0`. عيّن `0` للتعطيل.
- `replyToMode`: ‏`off` أو `first` أو `all` أو `batched`.
- `markdown`: تكوين اختياري لعرض Markdown للنص الصادر في Matrix.
- `streaming`: ‏`off` (الافتراضي) أو `partial` أو `quiet` أو `true` أو `false`. يفعّل `partial` و`true` تحديثات المسودة بنمط المعاينة أولًا باستخدام رسائل Matrix النصية العادية. ويستخدم `quiet` إشعارات معاينة غير مُنبِّهة لإعدادات قواعد الدفع المستضافة ذاتيًا.
- `blockStreaming`: يفعّل `true` رسائل تقدم منفصلة للكتل المكتملة من المساعد بينما يكون بث مسودة المعاينة نشطًا.
- `threadReplies`: ‏`off` أو `inbound` أو `always`.
- `threadBindings`: تجاوزات لكل قناة لتوجيه الجلسات المرتبطة بالخيط ودورة حياتها.
- `startupVerification`: وضع طلب التحقق الذاتي التلقائي عند بدء التشغيل (`if-unverified` أو `off`).
- `startupVerificationCooldownHours`: فترة التهدئة قبل إعادة محاولة طلبات التحقق التلقائية عند بدء التشغيل.
- `textChunkLimit`: حجم تجزئة الرسائل الصادرة.
- `chunkMode`: ‏`length` أو `newline`.
- `responsePrefix`: بادئة رسالة اختيارية للردود الصادرة.
- `ackReaction`: تجاوز اختياري لتفاعل الإقرار لهذه القناة/الحساب.
- `ackReactionScope`: تجاوز اختياري لنطاق تفاعل الإقرار (`group-mentions` أو `group-all` أو `direct` أو `all` أو `none` أو `off`).
- `reactionNotifications`: وضع إشعارات التفاعل الواردة (`own` أو `off`).
- `mediaMaxMb`: حد حجم الوسائط بالميغابايت لمعالجة وسائط Matrix. ينطبق على الإرسال الصادر ومعالجة الوسائط الواردة.
- `autoJoin`: سياسة الانضمام التلقائي للدعوات (`always` أو `allowlist` أو `off`). الافتراضي: `off`. ينطبق هذا على دعوات Matrix عمومًا، بما في ذلك الدعوات الشبيهة بالرسائل المباشرة، وليس فقط دعوات الغرف/المجموعات. يتخذ OpenClaw هذا القرار في وقت الدعوة، قبل أن يتمكن من تصنيف الغرفة المنضم إليها بشكل موثوق كرسالة مباشرة أو مجموعة.
- `autoJoinAllowlist`: الغرف/الأسماء المستعارة المسموح بها عندما يكون `autoJoin` هو `allowlist`. تُحل إدخالات الأسماء المستعارة إلى معرّفات غرف أثناء معالجة الدعوة؛ ولا يثق OpenClaw في حالة الاسم المستعار التي تدّعيها الغرفة المدعو إليها.
- `dm`: كتلة سياسة الرسائل المباشرة (`enabled` أو `policy` أو `allowFrom` أو `sessionScope` أو `threadReplies`).
- `dm.policy`: يتحكم في وصول الرسائل المباشرة بعد انضمام OpenClaw إلى الغرفة وتصنيفها كرسالة مباشرة. ولا يغير ما إذا كانت الدعوة ستنضم تلقائيًا.
- يجب أن تكون إدخالات `dm.allowFrom` معرّفات مستخدمي Matrix كاملة ما لم تكن قد حللتها بالفعل عبر البحث المباشر في الدليل.
- `dm.sessionScope`: ‏`per-user` (الافتراضي) أو `per-room`. استخدم `per-room` عندما تريد أن تحتفظ كل غرفة رسائل مباشرة في Matrix بسياق منفصل حتى لو كان النظير هو نفسه.
- `dm.threadReplies`: تجاوز لسياسة الخيوط خاص بالرسائل المباشرة (`off` أو `inbound` أو `always`). وهو يتجاوز إعداد `threadReplies` ذي المستوى الأعلى لكل من موضع الرد وعزل الجلسة في الرسائل المباشرة.
- `execApprovals`: تسليم موافقات exec الأصلية في Matrix (`enabled` أو `approvers` أو `target` أو `agentFilter` أو `sessionFilter`).
- `execApprovals.approvers`: معرّفات مستخدمي Matrix المسموح لهم بالموافقة على طلبات exec. وهو اختياري عندما تكون `dm.allowFrom` تحدد الموافقين بالفعل.
- `execApprovals.target`: ‏`dm | channel | both` (الافتراضي: `dm`).
- `accounts`: تجاوزات مسماة لكل حساب. تعمل قيم `channels.matrix` ذات المستوى الأعلى كقيم افتراضية لهذه الإدخالات.
- `groups`: خريطة سياسة لكل غرفة. فضّل معرّفات الغرف أو الأسماء المستعارة؛ تُتجاهل أسماء الغرف غير المحلولة وقت التشغيل. تستخدم هوية الجلسة/المجموعة معرّف الغرفة المستقر بعد الحل، بينما تستمر التسميات المقروءة بشريًا في المجيء من أسماء الغرف.
- `groups.<room>.account`: يقصر إدخال غرفة موروث واحدًا على حساب Matrix محدد في إعدادات الحسابات المتعددة.
- `groups.<room>.allowBots`: تجاوز على مستوى الغرفة للمرسلين من الروبوتات المكوّنة (`true` أو `"mentions"`).
- `groups.<room>.users`: قائمة سماح للمرسلين لكل غرفة.
- `groups.<room>.tools`: تجاوزات السماح/الرفض للأدوات لكل غرفة.
- `groups.<room>.autoReply`: تجاوز على مستوى الغرفة لتقييد الذكر. يؤدي `true` إلى تعطيل متطلبات الذكر لتلك الغرفة؛ ويعيد `false` فرضها.
- `groups.<room>.skills`: مرشح Skills اختياري على مستوى الغرفة.
- `groups.<room>.systemPrompt`: مقتطف system prompt اختياري على مستوى الغرفة.
- `rooms`: اسم مستعار قديم لـ `groups`.
- `actions`: تقييد الأدوات لكل إجراء (`messages` أو `reactions` أو `pins` أو `profile` أو `memberInfo` أو `channelInfo` أو `verification`).

## ذو صلة

- [نظرة عامة على القنوات](/ar/channels) — جميع القنوات المدعومة
- [الإقران](/ar/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الإقران
- [المجموعات](/ar/channels/groups) — سلوك الدردشة الجماعية وتقييد الذكر
- [توجيه القنوات](/ar/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/ar/gateway/security) — نموذج الوصول والتقوية
