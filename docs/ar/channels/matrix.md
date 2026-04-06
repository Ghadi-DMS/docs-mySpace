---
read_when:
    - إعداد Matrix في OpenClaw
    - تكوين Matrix E2EE والتحقق
summary: حالة دعم Matrix، والإعداد، وأمثلة التكوين
title: Matrix
x-i18n:
    generated_at: "2026-04-06T07:20:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 06f833bf0ede81bad69f140994c32e8cc5d1635764f95fc5db4fc5dc25f2b85e
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix هو plugin القناة المضمّن Matrix لـ OpenClaw.
يستخدم `matrix-js-sdk` الرسمي ويدعم الرسائل المباشرة والغرف والخيوط والوسائط والتفاعلات واستطلاعات الرأي والموقع وE2EE.

## plugin المضمّن

يأتي Matrix كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك لا تحتاج
البنيات المجمّعة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستبعد Matrix، فثبّته
يدويًا:

ثبّت من npm:

```bash
openclaw plugins install @openclaw/matrix
```

ثبّت من نسخة محلية:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

راجع [Plugins](/ar/tools/plugin) لمعرفة سلوك plugin وقواعد التثبيت.

## الإعداد

1. تأكد من أن plugin Matrix متاح.
   - إصدارات OpenClaw المجمّعة الحالية تتضمنه بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافته يدويًا باستخدام الأوامر أعلاه.
2. أنشئ حساب Matrix على الخادم المنزلي الخاص بك.
3. كوّن `channels.matrix` باستخدام أحد الخيارين:
   - `homeserver` + `accessToken`، أو
   - `homeserver` + `userId` + `password`.
4. أعد تشغيل البوابة.
5. ابدأ رسالة مباشرة مع الروبوت أو ادعه إلى غرفة.

مسارات الإعداد التفاعلية:

```bash
openclaw channels add
openclaw configure --section channels
```

ما الذي يسأل عنه معالج Matrix فعليًا:

- عنوان URL للخادم المنزلي
- طريقة المصادقة: access token أو كلمة مرور
- معرّف المستخدم فقط عندما تختار مصادقة كلمة المرور
- اسم جهاز اختياري
- ما إذا كنت تريد تفعيل E2EE
- ما إذا كنت تريد تكوين الوصول إلى غرف Matrix الآن

سلوك المعالج المهم:

- إذا كانت متغيرات بيئة مصادقة Matrix موجودة بالفعل للحساب المحدد، ولم يكن هذا الحساب محفوظًا له مصادقة في التكوين بالفعل، فإن المعالج يعرض اختصارًا عبر البيئة ويكتب فقط `enabled: true` لذلك الحساب.
- عند إضافة حساب Matrix آخر بشكل تفاعلي، يُطبَّع اسم الحساب المُدخل إلى معرّف الحساب المستخدم في التكوين ومتغيرات البيئة. على سبيل المثال، `Ops Bot` تصبح `ops-bot`.
- تقبل مطالبات قائمة السماح للرسائل المباشرة قيم `@user:server` الكاملة مباشرة. أسماء العرض تعمل فقط عندما يعثر البحث المباشر في الدليل على تطابق واحد دقيق؛ وإلا يطلب منك المعالج إعادة المحاولة باستخدام معرّف Matrix كامل.
- تقبل مطالبات قائمة السماح للغرف معرّفات الغرف والأسماء المستعارة مباشرة. ويمكنها أيضًا حل أسماء الغرف المنضم إليها مباشرة، لكن الأسماء غير المحلولة تُحتفظ بها كما أُدخلت أثناء الإعداد فقط ويتم تجاهلها لاحقًا بواسطة حل قائمة السماح وقت التشغيل. فضّل `!room:server` أو `#alias:server`.
- تستخدم هوية الغرفة/الجلسة وقت التشغيل معرّف غرفة Matrix الثابت. لا تُستخدم الأسماء المستعارة المعلنة في الغرفة إلا كمدخلات للبحث، وليس كمفتاح جلسة طويل الأمد أو هوية مجموعة ثابتة.
- لحل أسماء الغرف قبل حفظها، استخدم `openclaw channels resolve --channel matrix "Project Room"`.

إعداد بسيط قائم على token:

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

إعداد قائم على كلمة المرور (يُخزَّن token مؤقتًا بعد تسجيل الدخول):

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
يستخدم الحساب الافتراضي `credentials.json`؛ وتستخدم الحسابات المسمّاة `credentials-<account>.json`.
عندما توجد بيانات اعتماد مخزنة مؤقتًا هناك، يتعامل OpenClaw مع Matrix على أنه مُكوَّن لأغراض الإعداد وdoctor واكتشاف حالة القناة، حتى إذا لم تكن المصادقة الحالية مضبوطة مباشرة في التكوين.

المكافئات عبر متغيرات البيئة (تُستخدم عندما لا يكون مفتاح التكوين مضبوطًا):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

للحسابات غير الافتراضية، استخدم متغيرات البيئة المقيّدة بنطاق الحساب:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

مثال للحساب `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

لمعرّف الحساب المطبع `ops-bot`، استخدم:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

يهرب Matrix علامات الترقيم في معرّفات الحسابات للحفاظ على عدم تعارض متغيرات البيئة المقيّدة بالنطاق.
على سبيل المثال، تتحول `-` إلى `_X2D_`، لذا يتحول `ops-prod` إلى `MATRIX_OPS_X2D_PROD_*`.

لا يعرض المعالج التفاعلي اختصار متغير البيئة إلا عندما تكون متغيرات بيئة المصادقة هذه موجودة بالفعل ولم يكن الحساب المحدد قد حفظ مصادقة Matrix بالفعل في التكوين.

## مثال على التكوين

هذا تكوين أساسي عملي يتضمن اقتران الرسائل المباشرة وقائمة سماح للغرف وE2EE مفعّلًا:

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
وهذا يشمل دعوات جديدة بنمط الرسائل المباشرة. في وقت الدعوة، لا يعرف OpenClaw بشكل موثوق ما إذا كانت
الغرفة المدعو إليها ستُعامل في النهاية كرسالة مباشرة أو كمجموعة، لذا تمر كل الدعوات أولًا عبر قرار
`autoJoin` نفسه. يظل `dm.policy` منطبقًا بعد انضمام الروبوت وتصنيف الغرفة
كرسالة مباشرة، لذا يتحكم `autoJoin` في سلوك الانضمام بينما يتحكم `dm.policy` في سلوك
الرد/الوصول.

## معاينات البث

بث الردود في Matrix هو ميزة اختيارية.

اضبط `channels.matrix.streaming` على `"partial"` عندما تريد من OpenClaw إرسال معاينة مباشرة واحدة
للرد، وتعديل هذه المعاينة في مكانها بينما يولد النموذج النص، ثم إنهاءها عندما
ينتهي الرد:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` هو الوضع الافتراضي. ينتظر OpenClaw الرد النهائي ويرسله مرة واحدة.
- `streaming: "partial"` ينشئ رسالة معاينة واحدة قابلة للتعديل لكتلة المساعد الحالية باستخدام رسائل Matrix النصية العادية. يحافظ هذا على سلوك إشعارات Matrix القديم القائم على المعاينة أولًا، لذا قد ترسل التطبيقات العميلة القياسية إشعارًا عند أول نص معاينة متدفق بدلًا من الكتلة النهائية.
- `streaming: "quiet"` ينشئ إشعار معاينة هادئًا واحدًا قابلًا للتعديل لكتلة المساعد الحالية. استخدم هذا فقط عندما تضبط أيضًا قواعد push للمستلمين من أجل تعديلات المعاينة النهائية.
- `blockStreaming: true` يفعّل رسائل تقدم منفصلة في Matrix. عند تفعيل بث المعاينة، يحتفظ Matrix بالمسودة الحية للكتلة الحالية ويحافظ على الكتل المكتملة كرسائل منفصلة.
- عندما يكون بث المعاينة مفعّلًا ويكون `blockStreaming` معطلًا، يعدّل Matrix المسودة الحية في مكانها وينهي الحدث نفسه عندما تنتهي الكتلة أو الدور.
- إذا لم تعد المعاينة تتسع داخل حدث Matrix واحد، يوقف OpenClaw بث المعاينة ويعود إلى التسليم النهائي العادي.
- تستمر ردود الوسائط في إرسال المرفقات بشكل طبيعي. إذا لم يعد من الممكن إعادة استخدام معاينة قديمة بأمان، يحذفها OpenClaw قبل إرسال رد الوسائط النهائي.
- تعديلات المعاينة تستهلك استدعاءات إضافية إلى Matrix API. اترك البث معطلًا إذا كنت تريد أكثر سلوك تحفظًا تجاه حدود المعدل.

لا يفعّل `blockStreaming` معاينات المسودة بمفرده.
استخدم `streaming: "partial"` أو `streaming: "quiet"` لتعديلات المعاينة؛ ثم أضف `blockStreaming: true` فقط إذا كنت تريد أيضًا أن تبقى كتل المساعد المكتملة مرئية كرسائل تقدم منفصلة.

إذا كنت تحتاج إلى إشعارات Matrix القياسية بدون قواعد push مخصصة، فاستخدم `streaming: "partial"` لسلوك المعاينة أولًا أو اترك `streaming` معطلًا للتسليم النهائي فقط. مع `streaming: "off"`:

- `blockStreaming: true` يرسل كل كتلة مكتملة كرسالة Matrix عادية مُشعِرة.
- `blockStreaming: false` يرسل فقط الرد المكتمل النهائي كرسالة Matrix عادية مُشعِرة.

### قواعد push ذاتية الاستضافة للمعاينات الهادئة المنتهية

إذا كنت تشغّل بنية Matrix الخاصة بك وتريد أن تُشعِر المعاينات الهادئة فقط عند اكتمال كتلة أو
الرد النهائي، فاضبط `streaming: "quiet"` وأضف قاعدة push لكل مستخدم لتعديلات المعاينة النهائية.

عادةً ما يكون هذا إعدادًا على مستوى المستخدم المستلم، وليس تغييرًا عامًا في تكوين الخادم المنزلي:

خريطة سريعة قبل أن تبدأ:

- المستخدم المستلم = الشخص الذي يجب أن يتلقى الإشعار
- المستخدم الروبوت = حساب Matrix الخاص بـ OpenClaw الذي يرسل الرد
- استخدم access token الخاص بالمستخدم المستلم في استدعاءات API أدناه
- طابق `sender` في قاعدة push مع MXID الكامل للمستخدم الروبوت

1. كوّن OpenClaw لاستخدام المعاينات الهادئة:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. تأكد من أن حساب المستلم يتلقى بالفعل إشعارات push عادية من Matrix. قواعد المعاينة الهادئة
   لا تعمل إلا إذا كان لدى هذا المستخدم pushers/أجهزة عاملة بالفعل.

3. احصل على access token الخاص بالمستخدم المستلم.
   - استخدم token الخاص بالمستخدم المتلقي، وليس token الروبوت.
   - غالبًا ما تكون إعادة استخدام token جلسة عميل موجودة هي الأسهل.
   - إذا كنت بحاجة إلى إصدار token جديد، يمكنك تسجيل الدخول عبر Matrix Client-Server API القياسي:

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

4. تحقق من أن حساب المستلم لديه pushers بالفعل:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

إذا أعاد هذا الاستدعاء عدم وجود pushers/أجهزة نشطة، فأصلح أولًا إشعارات Matrix العادية قبل إضافة
قاعدة OpenClaw أدناه.

يضع OpenClaw علامة على تعديلات المعاينة النصية النهائية باستخدام:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. أنشئ قاعدة push من نوع override لكل حساب مستلم يجب أن يتلقى هذه الإشعارات:

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

- `https://matrix.example.org`: عنوان URL الأساسي لخادمك المنزلي
- `$USER_ACCESS_TOKEN`: access token الخاص بالمستخدم المتلقي
- `openclaw-finalized-preview-botname`: معرّف قاعدة فريد لهذا الروبوت لهذا المستخدم المتلقي
- `@bot:example.org`: MXID روبوت Matrix الخاص بـ OpenClaw لديك، وليس MXID المستخدم المتلقي

مهم لإعدادات الروبوتات المتعددة:

- تُفهرس قواعد push بواسطة `ruleId`. تؤدي إعادة تشغيل `PUT` على معرّف القاعدة نفسه إلى تحديث هذه القاعدة نفسها.
- إذا كان يجب أن يتلقى مستخدم واحد إشعارات من عدة حسابات روبوت Matrix لـ OpenClaw، فأنشئ قاعدة واحدة لكل روبوت مع معرّف قاعدة فريد لكل تطابق `sender`.
- نمط بسيط هو `openclaw-finalized-preview-<botname>`، مثل `openclaw-finalized-preview-ops` أو `openclaw-finalized-preview-support`.

تُقيّم القاعدة مقابل مرسل الحدث:

- صادق باستخدام token المستخدم المتلقي
- طابق `sender` مع MXID روبوت OpenClaw

6. تحقق من وجود القاعدة:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. اختبر ردًا متدفقًا. في الوضع الهادئ، يجب أن تُظهر الغرفة معاينة مسودة هادئة وأن
   يُشعِر التعديل النهائي في مكانه مرة واحدة عند انتهاء الكتلة أو الدور.

إذا احتجت إلى إزالة القاعدة لاحقًا، فاحذف معرّف القاعدة نفسه باستخدام token المستخدم المتلقي:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

ملاحظات:

- أنشئ القاعدة باستخدام access token للمستخدم المتلقي، وليس token الروبوت.
- تُدرج قواعد `override` الجديدة المعرفة من قبل المستخدم قبل قواعد الكبت الافتراضية، لذلك لا حاجة إلى معلمة ترتيب إضافية.
- يؤثر هذا فقط في تعديلات المعاينة النصية النهائية التي يمكن لـ OpenClaw إنهاؤها بأمان في مكانها. ما زالت بدائل الوسائط وبدائل المعاينات القديمة تستخدم تسليم Matrix العادي.
- إذا أظهر `GET /_matrix/client/v3/pushers` عدم وجود pushers، فهذا يعني أن المستخدم ليس لديه بعد تسليم push عاملاً من Matrix لهذا الحساب/الجهاز.

#### Synapse

بالنسبة إلى Synapse، يكون الإعداد أعلاه كافيًا عادةً بمفرده:

- لا يلزم إجراء تغيير خاص في `homeserver.yaml` لإشعارات معاينة OpenClaw النهائية.
- إذا كانت عملية نشر Synapse لديك ترسل بالفعل إشعارات push عادية من Matrix، فإن token المستخدم + استدعاء `pushrules` أعلاه هو خطوة الإعداد الرئيسية.
- إذا كنت تشغّل Synapse خلف وكيل عكسي أو workers، فتأكد من أن `/_matrix/client/.../pushrules/` يصل إلى Synapse بشكل صحيح.
- إذا كنت تشغّل Synapse workers، فتأكد من أن pushers سليمة. تتم معالجة تسليم push بواسطة العملية الرئيسية أو `synapse.app.pusher` / workers المكوّنة لـ pusher.

#### Tuwunel

بالنسبة إلى Tuwunel، استخدم مسار الإعداد نفسه واستدعاء API الخاص بـ push-rule الموضحين أعلاه:

- لا يلزم أي تكوين خاص بـ Tuwunel لعلامة المعاينة النهائية نفسها.
- إذا كانت إشعارات Matrix العادية تعمل بالفعل لهذا المستخدم، فإن token المستخدم + استدعاء `pushrules` أعلاه هو خطوة الإعداد الرئيسية.
- إذا بدا أن الإشعارات تختفي بينما يكون المستخدم نشطًا على جهاز آخر، فتحقق مما إذا كان `suppress_push_when_active` مفعّلًا. أضاف Tuwunel هذا الخيار في Tuwunel 1.4.2 بتاريخ 12 سبتمبر 2025، ويمكنه عمدًا كبت عمليات push إلى الأجهزة الأخرى بينما يكون أحد الأجهزة نشطًا.

## التشفير والتحقق

في الغرف المشفرة (E2EE)، تستخدم أحداث الصور الصادرة `thumbnail_file` بحيث تُشفَّر معاينات الصور جنبًا إلى جنب مع المرفق الكامل. أما الغرف غير المشفرة فما زالت تستخدم `thumbnail_url` العادي. لا يلزم أي تكوين — يكتشف plugin حالة E2EE تلقائيًا.

### غرف الروبوت إلى الروبوت

افتراضيًا، يتم تجاهل رسائل Matrix القادمة من حسابات Matrix أخرى مُكوّنة في OpenClaw.

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

- `allowBots: true` يقبل الرسائل من حسابات روبوت Matrix أخرى مُكوّنة في الغرف والرسائل المباشرة المسموح بها.
- `allowBots: "mentions"` يقبل هذه الرسائل فقط عندما تذكر هذا الروبوت صراحةً في الغرف. وتبقى الرسائل المباشرة مسموحًا بها.
- `groups.<room>.allowBots` يتجاوز الإعداد على مستوى الحساب لغرفة واحدة.
- ما زال OpenClaw يتجاهل الرسائل من معرّف مستخدم Matrix نفسه لتجنب حلقات الرد الذاتي.
- لا يكشف Matrix هنا عن علامة روبوت أصلية؛ ويتعامل OpenClaw مع "مكتوب بواسطة روبوت" على أنه "مرسل بواسطة حساب Matrix آخر مُكوَّن على بوابة OpenClaw هذه".

استخدم قوائم سماح صارمة للغرف ومتطلبات الذكر عند تفعيل حركة المرور بين الروبوتات في الغرف المشتركة.

فعّل التشفير:

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

حالة تفصيلية (تشخيص كامل):

```bash
openclaw matrix verify status --verbose
```

تضمين مفتاح الاسترداد المخزن في مخرجات قابلة للقراءة آليًا:

```bash
openclaw matrix verify status --include-recovery-key --json
```

تهيئة التوقيع المتقاطع وحالة التحقق:

```bash
openclaw matrix verify bootstrap
```

دعم الحسابات المتعددة: استخدم `channels.matrix.accounts` مع بيانات اعتماد لكل حساب و`name` اختياري. راجع [المرجع الخاص بالتكوين](/ar/gateway/configuration-reference#multi-account-all-channels) للنمط المشترك.

تشخيص bootstrap التفصيلي:

```bash
openclaw matrix verify bootstrap --verbose
```

فرض إعادة تعيين هوية توقيع متقاطع جديدة قبل bootstrap:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

تحقق من هذا الجهاز باستخدام مفتاح استرداد:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

تفاصيل التحقق من الجهاز بشكل تفصيلي:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

تحقق من سلامة النسخة الاحتياطية لمفاتيح الغرف:

```bash
openclaw matrix verify backup status
```

تشخيص سلامة النسخة الاحتياطية بشكل تفصيلي:

```bash
openclaw matrix verify backup status --verbose
```

استعد مفاتيح الغرف من النسخة الاحتياطية على الخادم:

```bash
openclaw matrix verify backup restore
```

تشخيص الاستعادة بشكل تفصيلي:

```bash
openclaw matrix verify backup restore --verbose
```

احذف النسخة الاحتياطية الحالية على الخادم وأنشئ خط أساس جديدًا للنسخة الاحتياطية. إذا تعذر تحميل
مفتاح النسخة الاحتياطية المخزن بشكل نظيف، فقد تؤدي إعادة الضبط هذه أيضًا إلى إعادة إنشاء التخزين السري بحيث
تستطيع عمليات البدء الباردة المستقبلية تحميل مفتاح النسخة الاحتياطية الجديد:

```bash
openclaw matrix verify backup reset --yes
```

جميع أوامر `verify` موجزة افتراضيًا (بما في ذلك التسجيل الداخلي الهادئ لـ SDK) وتعرض تشخيصًا مفصلًا فقط باستخدام `--verbose`.
استخدم `--json` للحصول على مخرجات كاملة قابلة للقراءة آليًا عند كتابة السكربتات.

في إعدادات الحسابات المتعددة، تستخدم أوامر Matrix في CLI حساب Matrix الافتراضي الضمني ما لم تمرر `--account <id>`.
إذا كوّنت عدة حسابات مسمّاة، فاضبط `channels.matrix.defaultAccount` أولًا وإلا ستتوقف عمليات CLI الضمنية تلك وتطلب منك اختيار حساب صراحةً.
استخدم `--account` كلما أردت أن تستهدف عمليات التحقق أو الجهاز حسابًا مسمّى بشكل صريح:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

عندما يكون التشفير معطلًا أو غير متاح لحساب مسمّى، تشير تحذيرات Matrix وأخطاء التحقق إلى مفتاح تكوين ذلك الحساب، مثل `channels.matrix.accounts.assistant.encryption`.

### ما معنى "تم التحقق"

يعامل OpenClaw جهاز Matrix هذا على أنه مُتحقَّق منه فقط عندما يكون متحققًا منه بواسطة هوية التوقيع المتقاطع الخاصة بك.
عمليًا، يكشف `openclaw matrix verify status --verbose` عن ثلاث إشارات ثقة:

- `Locally trusted`: هذا الجهاز موثوق من قبل العميل الحالي فقط
- `Cross-signing verified`: يبلّغ SDK أن الجهاز متحقق منه عبر التوقيع المتقاطع
- `Signed by owner`: الجهاز موقّع بواسطة مفتاح التوقيع الذاتي الخاص بك

تصبح `Verified by owner` مساوية لـ `yes` فقط عند وجود تحقق بالتوقيع المتقاطع أو توقيع من المالك.
الثقة المحلية وحدها لا تكفي لكي يتعامل OpenClaw مع الجهاز على أنه متحقق منه بالكامل.

### ماذا يفعل bootstrap

الأمر `openclaw matrix verify bootstrap` هو أمر الإصلاح والإعداد لحسابات Matrix المشفرة.
وهو ينفذ كل ما يلي بالترتيب:

- يهيئ التخزين السري، مع إعادة استخدام مفتاح استرداد موجود إن أمكن
- يهيئ التوقيع المتقاطع ويرفع مفاتيح التوقيع المتقاطع العامة الناقصة
- يحاول تعليم الجهاز الحالي وتوقيعه توقيعًا متقاطعًا
- ينشئ نسخة احتياطية جديدة لمفاتيح الغرف على الخادم إذا لم تكن موجودة بالفعل

إذا كان الخادم المنزلي يتطلب مصادقة تفاعلية لرفع مفاتيح التوقيع المتقاطع، فإن OpenClaw يحاول الرفع أولًا دون مصادقة، ثم باستخدام `m.login.dummy`، ثم باستخدام `m.login.password` عندما يكون `channels.matrix.password` مضبوطًا.

استخدم `--force-reset-cross-signing` فقط عندما تريد عمدًا تجاهل هوية التوقيع المتقاطع الحالية وإنشاء هوية جديدة.

إذا كنت تريد عمدًا تجاهل النسخة الاحتياطية الحالية لمفاتيح الغرف وبدء
خط أساس جديد للنسخة الاحتياطية للرسائل المستقبلية، فاستخدم `openclaw matrix verify backup reset --yes`.
افعل ذلك فقط عندما تقبل أن السجل المشفر القديم غير القابل للاسترداد سيبقى
غير متاح وأن OpenClaw قد يعيد إنشاء التخزين السري إذا تعذر تحميل
سر النسخة الاحتياطية الحالية بأمان.

### خط أساس جديد للنسخة الاحتياطية

إذا كنت تريد إبقاء الرسائل المشفرة المستقبلية عاملة وتقبل فقدان السجل القديم غير القابل للاسترداد، فشغّل هذه الأوامر بالترتيب:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

أضف `--account <id>` إلى كل أمر عندما تريد استهداف حساب Matrix مسمّى بشكل صريح.

### سلوك بدء التشغيل

عندما يكون `encryption: true`، يضبط Matrix القيمة الافتراضية لـ `startupVerification` إلى `"if-unverified"`.
عند بدء التشغيل، إذا كان هذا الجهاز لا يزال غير متحقق منه، فسيطلب Matrix التحقق الذاتي في عميل Matrix آخر،
ويتجاوز الطلبات المكررة عندما يكون أحدها قيد الانتظار بالفعل، ويطبق فترة تهدئة محلية قبل إعادة المحاولة بعد إعادة التشغيل.
تعاد محاولة الطلبات الفاشلة في وقت أقرب من الإنشاء الناجح للطلبات افتراضيًا.
اضبط `startupVerification: "off"` لتعطيل طلبات بدء التشغيل التلقائية، أو عدّل `startupVerificationCooldownHours`
إذا كنت تريد نافذة إعادة محاولة أقصر أو أطول.

ينفذ بدء التشغيل أيضًا تمريرة bootstrap مشفرة محافظة تلقائيًا.
وتحاول هذه التمريرة إعادة استخدام التخزين السري الحالي وهوية التوقيع المتقاطع الحالية أولًا، وتتجنب إعادة تعيين التوقيع المتقاطع ما لم تشغّل مسار إصلاح bootstrap صريحًا.

إذا وجد بدء التشغيل حالة bootstrap معطلة وكان `channels.matrix.password` مضبوطًا، فيمكن لـ OpenClaw محاولة مسار إصلاح أكثر صرامة.
إذا كان الجهاز الحالي موقّعًا بالفعل من المالك، فإن OpenClaw يحافظ على تلك الهوية بدلًا من إعادة تعيينها تلقائيًا.

الترقية من plugin Matrix العام السابق:

- يعيد OpenClaw تلقائيًا استخدام حساب Matrix نفسه وaccess token وهوية الجهاز متى أمكن.
- قبل تنفيذ أي تغييرات ترحيل Matrix قابلة للتنفيذ، ينشئ OpenClaw أو يعيد استخدام لقطة استرداد ضمن `~/Backups/openclaw-migrations/`.
- إذا كنت تستخدم عدة حسابات Matrix، فاضبط `channels.matrix.defaultAccount` قبل الترقية من تخطيط التخزين المسطح القديم حتى يعرف OpenClaw أي حساب يجب أن يتلقى هذه الحالة القديمة المشتركة.
- إذا كان plugin السابق قد خزن محليًا مفتاح فك تشفير نسخة Matrix الاحتياطية لمفاتيح الغرف، فإن بدء التشغيل أو `openclaw doctor --fix` سيستوردانه تلقائيًا إلى مسار مفتاح الاسترداد الجديد.
- إذا تغيّر access token الخاص بـ Matrix بعد تجهيز الترحيل، فإن بدء التشغيل يفحص الآن جذور تخزين تجزئة token المجاورة بحثًا عن حالة استعادة قديمة معلّقة قبل التخلي عن الاستعادة التلقائية للنسخة الاحتياطية.
- إذا تغيّر access token الخاص بـ Matrix لاحقًا للحساب نفسه والخادم المنزلي نفسه والمستخدم نفسه، فإن OpenClaw يفضّل الآن إعادة استخدام جذر تخزين تجزئة token الأكثر اكتمالًا بدلًا من البدء من دليل حالة Matrix فارغ.
- في تشغيل البوابة التالي، ستُستعاد مفاتيح الغرف المنسوخة احتياطيًا تلقائيًا إلى مخزن التشفير الجديد.
- إذا كان plugin القديم يحتوي على مفاتيح غرف محلية فقط لم تُنسخ احتياطيًا أبدًا، فسيحذر OpenClaw بوضوح. لا يمكن تصدير هذه المفاتيح تلقائيًا من مخزن التشفير rust السابق، لذلك قد يبقى بعض السجل المشفر القديم غير متاح حتى يُستعاد يدويًا.
- راجع [ترحيل Matrix](/ar/install/migrating-matrix) لمعرفة مسار الترقية الكامل والقيود وأوامر الاسترداد ورسائل الترحيل الشائعة.

تُنظم حالة وقت التشغيل المشفرة ضمن جذور لكل حساب ولكل مستخدم ولكل تجزئة token في
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
يحتوي هذا الدليل على مخزن المزامنة (`bot-storage.json`) ومخزن التشفير (`crypto/`)،
وملف مفتاح الاسترداد (`recovery-key.json`) ولقطة IndexedDB (`crypto-idb-snapshot.json`)،
وربط الخيوط (`thread-bindings.json`) وحالة التحقق عند بدء التشغيل (`startup-verification.json`)
عند استخدام تلك الميزات.
عندما يتغير token لكن تبقى هوية الحساب كما هي، يعيد OpenClaw استخدام أفضل جذر موجود
لهذه الثلاثية account/homeserver/user بحيث تبقى حالة المزامنة السابقة وحالة التشفير وربط الخيوط
وحالة التحقق عند بدء التشغيل مرئية.

### نموذج مخزن تشفير Node

يستخدم Matrix E2EE في هذا plugin مسار التشفير Rust الرسمي الخاص بـ `matrix-js-sdk` في Node.
ويتوقع هذا المسار وجود تخزين دائم مدعوم بـ IndexedDB عندما تريد أن تستمر حالة التشفير بعد إعادة التشغيل.

يوفر OpenClaw ذلك حاليًا في Node من خلال:

- استخدام `fake-indexeddb` كطبقة توافق لـ IndexedDB API التي يتوقعها SDK
- استعادة محتويات IndexedDB الخاصة بتشفير Rust من `crypto-idb-snapshot.json` قبل `initRustCrypto`
- حفظ محتويات IndexedDB المحدّثة مرة أخرى إلى `crypto-idb-snapshot.json` بعد التهيئة وأثناء وقت التشغيل
- إجراء تسلسل لعمليات استعادة اللقطة وحفظها مقابل `crypto-idb-snapshot.json` باستخدام قفل ملفات استشاري حتى لا تتسابق عملية حفظ وقت تشغيل البوابة وصيانة CLI على ملف اللقطة نفسه

هذا تجهيز توافق/تخزين، وليس تنفيذًا مخصصًا للتشفير.
ملف اللقطة حالة وقت تشغيل حساسة ويُخزَّن مع أذونات ملفات مقيدة.
وبموجب نموذج أمان OpenClaw، فإن مضيف البوابة ودليل حالة OpenClaw المحلي داخل حدود المشغل الموثوق بالفعل، لذا فهذه في الأساس مسألة متانة تشغيلية وليست حد ثقة بعيد منفصل.

تحسين مخطط له:

- إضافة دعم SecretRef لمواد مفاتيح Matrix الدائمة بحيث يمكن الحصول على مفاتيح الاسترداد وأسرار تشفير التخزين ذات الصلة من موفري أسرار OpenClaw بدلًا من الملفات المحلية فقط

## إدارة الملف الشخصي

حدّث الملف الشخصي الذاتي في Matrix للحساب المحدد باستخدام:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

أضف `--account <id>` عندما تريد استهداف حساب Matrix مسمّى بشكل صريح.

يقبل Matrix عناوين URL للصورة الرمزية من نوع `mxc://` مباشرة. عندما تمرر عنوان URL للصورة الرمزية من نوع `http://` أو `https://`، يرفعه OpenClaw أولًا إلى Matrix ثم يخزن عنوان `mxc://` المحلول مرة أخرى في `channels.matrix.avatarUrl` (أو تجاوز الحساب المحدد).

## إشعارات التحقق التلقائية

ينشر Matrix الآن إشعارات دورة حياة التحقق مباشرة في غرفة الرسائل المباشرة الصارمة الخاصة بالتحقق كرسائل `m.notice`.
ويتضمن ذلك:

- إشعارات طلب التحقق
- إشعارات جاهزية التحقق (مع إرشاد صريح "تحقق عبر الرموز التعبيرية")
- إشعارات بدء التحقق واكتماله
- تفاصيل SAS (الرموز التعبيرية والأرقام العشرية) عند توفرها

تُتتبع طلبات التحقق الواردة من عميل Matrix آخر ويقبلها OpenClaw تلقائيًا.
وبالنسبة إلى مسارات التحقق الذاتي، يبدأ OpenClaw أيضًا تدفق SAS تلقائيًا عندما يصبح التحقق بالرموز التعبيرية متاحًا ويؤكد جانبه الخاص.
أما بالنسبة إلى طلبات التحقق من مستخدم/جهاز Matrix آخر، فيقبل OpenClaw الطلب تلقائيًا ثم ينتظر استمرار تدفق SAS بشكل طبيعي.
ما زلت بحاجة إلى مقارنة SAS بالرموز التعبيرية أو الأرقام العشرية في عميل Matrix الخاص بك وتأكيد "إنها متطابقة" هناك لإكمال التحقق.

لا يقبل OpenClaw تلقائيًا التدفقات المكررة التي بدأها بنفسه بشكل أعمى. يتجاوز بدء التشغيل إنشاء طلب جديد عندما يكون طلب التحقق الذاتي معلقًا بالفعل.

لا تُمرر إشعارات النظام/البروتوكول الخاصة بالتحقق إلى مسار دردشة الوكيل، لذا لا تنتج `NO_REPLY`.

### نظافة الأجهزة

يمكن أن تتراكم أجهزة Matrix القديمة التي يديرها OpenClaw على الحساب وتجعل الثقة في الغرف المشفرة أصعب في الفهم.
اعرضها باستخدام:

```bash
openclaw matrix devices list
```

أزل أجهزة Matrix القديمة التي يديرها OpenClaw باستخدام:

```bash
openclaw matrix devices prune-stale
```

### إصلاح الغرفة المباشرة

إذا خرجت حالة الرسائل المباشرة عن المزامنة، فقد ينتهي الأمر بـ OpenClaw إلى امتلاك تعيينات `m.direct` قديمة تشير إلى غرف فردية قديمة بدلًا من الرسالة المباشرة الحية. افحص التعيين الحالي لنظير باستخدام:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

وأصلحه باستخدام:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

يبقي الإصلاح المنطق الخاص بـ Matrix داخل plugin:

- يفضّل رسالة مباشرة صارمة 1:1 تم تعيينها بالفعل في `m.direct`
- وإلا يعود إلى أي رسالة مباشرة صارمة 1:1 منضم إليها حاليًا مع ذلك المستخدم
- وإذا لم توجد رسالة مباشرة سليمة، فإنه ينشئ غرفة مباشرة جديدة ويعيد كتابة `m.direct` للإشارة إليها

لا يحذف مسار الإصلاح الغرف القديمة تلقائيًا. فهو يختار فقط الرسالة المباشرة السليمة ويحدث التعيين بحيث تستهدف عمليات الإرسال الجديدة في Matrix وإشعارات التحقق وغيرها من تدفقات الرسائل المباشرة الغرفة الصحيحة مرة أخرى.

## الخيوط

يدعم Matrix خيوط Matrix الأصلية لكل من الردود التلقائية وعمليات الإرسال عبر أداة الرسائل.

- `dm.sessionScope: "per-user"` (الافتراضي) يُبقي توجيه الرسائل المباشرة في Matrix ضمن نطاق المرسل، بحيث يمكن لعدة غرف رسائل مباشرة مشاركة جلسة واحدة عندما تُحل إلى النظير نفسه.
- `dm.sessionScope: "per-room"` يعزل كل غرفة رسائل مباشرة في Matrix داخل مفتاح جلسة خاص بها مع الاستمرار في استخدام فحوصات المصادقة وقائمة السماح العادية للرسائل المباشرة.
- ما زالت ربطيات محادثات Matrix الصريحة تتغلب على `dm.sessionScope`، لذا تحافظ الغرف والخيوط المرتبطة على الجلسة المستهدفة التي اختارتها.
- `threadReplies: "off"` يبقي الردود في المستوى الأعلى ويبقي الرسائل الواردة داخل خيط على جلسة الرسالة الأصلية.
- `threadReplies: "inbound"` يرد داخل خيط فقط عندما تكون الرسالة الواردة أصلًا في هذا الخيط.
- `threadReplies: "always"` يبقي ردود الغرف داخل خيط متجذر في الرسالة المُطلِقة ويوجه تلك المحادثة عبر الجلسة المطابقة ذات نطاق الخيط من أول رسالة مُطلِقة.
- يتجاوز `dm.threadReplies` الإعداد العام للرسائل المباشرة فقط. على سبيل المثال، يمكنك إبقاء خيوط الغرف معزولة مع إبقاء الرسائل المباشرة مسطحة.
- تتضمن الرسائل الواردة ضمن خيوط رسالة جذر الخيط كسياق إضافي للوكيل.
- ترث عمليات الإرسال عبر أداة الرسائل الآن خيط Matrix الحالي تلقائيًا عندما يكون الهدف هو الغرفة نفسها، أو هدف مستخدم الرسائل المباشرة نفسه، ما لم يُقدَّم `threadId` صريح.
- لا يبدأ إعادة استخدام هدف مستخدم الرسائل المباشرة ضمن الجلسة نفسها إلا عندما تثبت بيانات وصف الجلسة الحالية النظير نفسه في الرسائل المباشرة وعلى حساب Matrix نفسه؛ وإلا يعود OpenClaw إلى التوجيه العادي ضمن نطاق المستخدم.
- عندما يرى OpenClaw أن غرفة رسائل مباشرة في Matrix تتصادم مع غرفة رسائل مباشرة أخرى على جلسة الرسائل المباشرة المشتركة نفسها في Matrix، فإنه ينشر `m.notice` لمرة واحدة في تلك الغرفة مع مخرج `/focus` عند تفعيل ربط الخيوط وتلميح `dm.sessionScope`.
- تُدعَم ربطيات خيوط وقت التشغيل في Matrix. تعمل الآن `/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` و`/acp spawn` المرتبطة بالخيط في غرف Matrix والرسائل المباشرة.
- ينشئ `/focus` على مستوى الغرفة/الرسالة المباشرة في Matrix خيط Matrix جديدًا ويربطه بالجلسة المستهدفة عندما يكون `threadBindings.spawnSubagentSessions=true`.
- يؤدي تشغيل `/focus` أو `/acp spawn --thread here` داخل خيط Matrix موجود إلى ربط هذا الخيط الحالي بدلًا من ذلك.

## ربط محادثات ACP

يمكن تحويل غرف Matrix والرسائل المباشرة وخيوط Matrix الموجودة إلى مساحات عمل ACP دائمة دون تغيير سطح الدردشة.

تدفق تشغيل سريع:

- شغّل `/acp spawn codex --bind here` داخل رسالة Matrix المباشرة أو الغرفة أو الخيط الحالي الذي تريد مواصلة استخدامه.
- في رسالة Matrix مباشرة أو غرفة على المستوى الأعلى، يبقى سطح الدردشة هو الرسالة المباشرة/الغرفة الحالية وتُوجَّه الرسائل المستقبلية إلى جلسة ACP المنشأة.
- داخل خيط Matrix موجود، يربط `--bind here` هذا الخيط الحالي في مكانه.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الربط.

ملاحظات:

- `--bind here` لا ينشئ خيط Matrix فرعيًا.
- لا يكون `threadBindings.spawnAcpSessions` مطلوبًا إلا لـ `/acp spawn --thread auto|here`، حيث يحتاج OpenClaw إلى إنشاء خيط Matrix فرعي أو ربطه.

### تكوين ربط الخيوط

يرث Matrix القيم الافتراضية العامة من `session.threadBindings`، ويدعم أيضًا تجاوزات لكل قناة:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

علامات الإنشاء المرتبطة بالخيوط في Matrix هي ميزات اختيارية:

- اضبط `threadBindings.spawnSubagentSessions: true` للسماح لأمر `/focus` على المستوى الأعلى بإنشاء خيوط Matrix جديدة وربطها.
- اضبط `threadBindings.spawnAcpSessions: true` للسماح للأمر `/acp spawn --thread auto|here` بربط جلسات ACP بخيوط Matrix.

## التفاعلات

يدعم Matrix إجراءات التفاعل الصادرة وإشعارات التفاعل الواردة وتفاعلات الإقرار الواردة.

- تُقيَّد أدوات التفاعل الصادرة بواسطة `channels["matrix"].actions.reactions`.
- يضيف `react` تفاعلًا إلى حدث Matrix محدد.
- يسرد `reactions` ملخص التفاعلات الحالي لحدث Matrix محدد.
- تؤدي `emoji=""` إلى إزالة تفاعلات حساب الروبوت نفسه على ذلك الحدث.
- تؤدي `remove: true` إلى إزالة تفاعل الرمز التعبيري المحدد فقط من حساب الروبوت.

يُحل نطاق تفاعلات الإقرار باستخدام ترتيب الحل القياسي في OpenClaw:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- الرجوع إلى الرمز التعبيري لهوية الوكيل

يُحل نطاق تفاعلات الإقرار بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

يُحل وضع إشعارات التفاعل بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- الافتراضي: `own`

السلوك الحالي:

- `reactionNotifications: "own"` يمرر أحداث `m.reaction` المضافة عندما تستهدف رسائل Matrix مكتوبة بواسطة الروبوت.
- `reactionNotifications: "off"` يعطل أحداث نظام التفاعل.
- ما زالت إزالة التفاعلات لا تُركَّب إلى أحداث نظام لأن Matrix يعرضها كعمليات حذف redactions، وليس كإزالة مستقلة لـ `m.reaction`.

## سياق السجل

- يتحكم `channels.matrix.historyLimit` في عدد رسائل الغرفة الحديثة التي تُدرج كـ `InboundHistory` عندما تُطلق رسالة غرفة Matrix الوكيل.
- ويعود إلى `messages.groupChat.historyLimit`. إذا لم يُضبط أي منهما، فالقيمة الافتراضية الفعلية هي `0`، لذا لا تُخزن رسائل الغرف المقيّدة بالذكر مؤقتًا. اضبط `0` للتعطيل.
- سجل غرف Matrix خاص بالغرف فقط. أما الرسائل المباشرة فتستمر في استخدام سجل الجلسة العادي.
- سجل غرف Matrix خاص بالرسائل المعلقة فقط: يخزن OpenClaw رسائل الغرف التي لم تُطلق ردًا بعد، ثم يلتقط تلك النافذة عندما يصل ذكر أو مُشغِّل آخر.
- لا تُدرج رسالة التشغيل الحالية ضمن `InboundHistory`؛ بل تبقى في النص الوارد الرئيسي لذلك الدور.
- تعيد محاولات الحدث نفسه في Matrix استخدام لقطة السجل الأصلية بدلًا من الانجراف إلى رسائل أحدث في الغرفة.

## إظهار السياق

يدعم Matrix عنصر التحكم المشترك `contextVisibility` للسياق التكميلي للغرفة مثل نص الرد المُجلب وجذور الخيوط والسجل المعلق.

- `contextVisibility: "all"` هو الوضع الافتراضي. يُحتفظ بالسياق التكميلي كما ورد.
- `contextVisibility: "allowlist"` يرشح السياق التكميلي إلى المرسلين المسموح لهم وفقًا لفحوصات قائمة السماح النشطة للغرفة/المستخدم.
- `contextVisibility: "allowlist_quote"` يعمل مثل `allowlist`، لكنه يحتفظ أيضًا برد مقتبس صريح واحد.

يؤثر هذا الإعداد في إظهار السياق التكميلي، وليس في ما إذا كانت الرسالة الواردة نفسها يمكن أن تُطلق ردًا.
ويظل تخويل المُشغِّل قادمًا من إعدادات `groupPolicy` و`groups` و`groupAllowFrom` وسياسة الرسائل المباشرة.

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

راجع [Groups](/ar/channels/groups) لمعرفة سلوك تقييد الذكر وقائمة السماح.

مثال اقتران لرسائل Matrix المباشرة:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

إذا استمر مستخدم Matrix غير معتمد في مراسلتك قبل الموافقة، فسيعيد OpenClaw استخدام رمز الاقتران المعلق نفسه وقد يرسل رد تذكير مرة أخرى بعد فترة تهدئة قصيرة بدلًا من إصدار رمز جديد.

راجع [Pairing](/ar/channels/pairing) لمعرفة تدفق اقتران الرسائل المباشرة والتخطيط التخزيني المشترك.

## موافقات Exec

يمكن أن يعمل Matrix كعميل موافقة exec لحساب Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (اختياري؛ يعود إلى `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

يجب أن يكون الموافقون معرّفات مستخدمي Matrix مثل `@owner:example.org`. يفعّل Matrix موافقات exec الأصلية تلقائيًا عندما تكون `enabled` غير مضبوطة أو تساوي `"auto"` وعندما يمكن حل موافِق واحد على الأقل، سواء من `execApprovals.approvers` أو من `channels.matrix.dm.allowFrom`. اضبط `enabled: false` لتعطيل Matrix كعميل موافقة أصلي صراحةً. وإلا تعود طلبات الموافقة إلى مسارات الموافقة الأخرى المضبوطة أو إلى سياسة fallback لموافقة exec.

التوجيه الأصلي في Matrix مخصص لـ exec فقط حاليًا:

- يتحكم `channels.matrix.execApprovals.*` في التوجيه الأصلي للرسائل المباشرة/القناة لموافقات exec فقط.
- ما زالت موافقات plugin تستخدم `/approve` المشترك في الدردشة نفسها بالإضافة إلى أي إعادة توجيه مضبوطة في `approvals.plugin`.
- ما زال Matrix قادرًا على إعادة استخدام `channels.matrix.dm.allowFrom` لتخويل موافقة plugin عندما يستطيع استنتاج الموافقين بأمان، لكنه لا يوفّر مسار توسيع أصلي منفصل لموافقات plugin عبر الرسائل المباشرة/القناة.

قواعد التسليم:

- `target: "dm"` يرسل مطالبات الموافقة إلى الرسائل المباشرة الخاصة بالموافقين
- `target: "channel"` يعيد إرسال المطالبة إلى غرفة Matrix الأصلية أو الرسالة المباشرة الأصلية
- `target: "both"` يرسل إلى الرسائل المباشرة للموافقين وإلى غرفة Matrix الأصلية أو الرسالة المباشرة الأصلية

تزرع مطالبات الموافقة في Matrix اختصارات تفاعلية على رسالة الموافقة الأساسية:

- `✅` = السماح مرة واحدة
- `❌` = الرفض
- `♾️` = السماح دائمًا عندما يكون هذا القرار مسموحًا به وفق سياسة exec الفعلية

يمكن للموافقين التفاعل على تلك الرسالة أو استخدام أوامر slash الاحتياطية: `/approve <id> allow-once` أو `/approve <id> allow-always` أو `/approve <id> deny`.

لا يستطيع الموافقة أو الرفض إلا الموافقون الذين تم حلهم. ويتضمن التسليم إلى القناة نص الأمر، لذا لا تفعّل `channel` أو `both` إلا في الغرف الموثوقة.

تعيد مطالبات الموافقة في Matrix استخدام مخطط الموافقات الأساسي المشترك. والسطح الأصلي الخاص بـ Matrix هو طبقة نقل فقط لموافقات exec: توجيه الغرف/الرسائل المباشرة وسلوك الإرسال/التحديث/الحذف للرسائل.

تجاوز لكل حساب:

- `channels.matrix.accounts.<account>.execApprovals`

الوثائق ذات الصلة: [موافقات Exec](/ar/tools/exec-approvals)

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

تعمل القيم ذات المستوى الأعلى في `channels.matrix` كقيم افتراضية للحسابات المسمّاة ما لم يتجاوزها حساب ما.
يمكنك تقييد إدخالات الغرف الموروثة إلى حساب Matrix واحد باستخدام `groups.<room>.account` (أو `rooms.<room>.account` القديم).
تبقى الإدخالات التي لا تحتوي على `account` مشتركة عبر جميع حسابات Matrix، وما زالت الإدخالات التي تحتوي على `account: "default"` تعمل عندما يكون الحساب الافتراضي مُكوَّنًا مباشرة على المستوى الأعلى في `channels.matrix.*`.
لا تنشئ القيم الافتراضية الجزئية المشتركة للمصادقة حسابًا افتراضيًا ضمنيًا منفصلًا بمفردها. لا يقوم OpenClaw بتركيب الحساب الأعلى `default` إلا عندما تكون لهذا الافتراضي مصادقة حديثة (`homeserver` مع `accessToken`، أو `homeserver` مع `userId` و`password`)؛ ويمكن للحسابات المسمّاة أن تبقى قابلة للاكتشاف من `homeserver` مع `userId` عندما تلبّي بيانات الاعتماد المخزنة مؤقتًا المصادقة لاحقًا.
إذا كان Matrix يحتوي بالفعل على حساب مسمّى واحد بالضبط، أو كان `defaultAccount` يشير إلى مفتاح حساب مسمّى موجود، فإن ترقية الإصلاح/الإعداد من حساب واحد إلى حسابات متعددة تحافظ على ذلك الحساب بدلًا من إنشاء إدخال جديد `accounts.default`. ولا تُنقل إلى ذلك الحساب المُرقّى إلا مفاتيح مصادقة/bootstrap الخاصة بـ Matrix؛ أما مفاتيح سياسة التسليم المشتركة فتظل في المستوى الأعلى.
اضبط `defaultAccount` عندما تريد أن يفضّل OpenClaw حساب Matrix مسمّى واحدًا للتوجيه الضمني والفحص وعمليات CLI.
إذا كوّنت عدة حسابات مسمّاة، فاضبط `defaultAccount` أو مرر `--account <id>` لأوامر CLI التي تعتمد على اختيار حساب ضمني.
مرر `--account <id>` إلى `openclaw matrix verify ...` و`openclaw matrix devices ...` عندما تريد تجاوز هذا الاختيار الضمني لأمر واحد.

## الخوادم المنزلية الخاصة/ضمن LAN

افتراضيًا، يحظر OpenClaw الخوادم المنزلية الخاصة/الداخلية لـ Matrix للحماية من SSRF ما لم
تفعّل ذلك صراحةً لكل حساب.

إذا كان الخادم المنزلي يعمل على localhost أو عنوان IP ضمن LAN/Tailscale أو اسم مضيف داخلي، ففعّل
`allowPrivateNetwork` لذلك الحساب في Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      allowPrivateNetwork: true,
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

يسمح هذا التفعيل الاختياري فقط بالأهداف الخاصة/الداخلية الموثوقة. أما الخوادم المنزلية العامة غير المشفرة مثل
`http://matrix.example.org:8008` فتبقى محظورة. فضّل `https://` كلما أمكن.

## توجيه حركة Matrix عبر وكيل

إذا كانت عملية نشر Matrix لديك تحتاج إلى وكيل HTTP(S) صادر صريح، فاضبط `channels.matrix.proxy`:

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

يمكن للحسابات المسمّاة تجاوز القيمة الافتراضية في المستوى الأعلى باستخدام `channels.matrix.accounts.<id>.proxy`.
يستخدم OpenClaw إعداد الوكيل نفسه لكل من حركة Matrix وقت التشغيل وفحوصات حالة الحساب.

## حل الأهداف

يقبل Matrix صيغ الأهداف التالية في أي مكان يطلب منك فيه OpenClaw هدف غرفة أو مستخدم:

- المستخدمون: `@user:server` أو `user:@user:server` أو `matrix:user:@user:server`
- الغرف: `!room:server` أو `room:!room:server` أو `matrix:room:!room:server`
- الأسماء المستعارة: `#alias:server` أو `channel:#alias:server` أو `matrix:channel:#alias:server`

يستخدم البحث المباشر في الدليل حساب Matrix المسجل الدخول:

- تستعلم عمليات البحث عن المستخدمين من دليل مستخدمي Matrix على ذلك الخادم المنزلي.
- تقبل عمليات البحث عن الغرف معرّفات الغرف والأسماء المستعارة الصريحة مباشرة، ثم تعود إلى البحث في أسماء الغرف المنضم إليها لذلك الحساب.
- البحث في أسماء الغرف المنضم إليها هو أفضل جهد. إذا تعذر حل اسم غرفة إلى معرّف أو اسم مستعار، فيتم تجاهله عند حل قائمة السماح وقت التشغيل.

## المرجع الخاص بالتكوين

- `enabled`: تفعيل القناة أو تعطيلها.
- `name`: تسمية اختيارية للحساب.
- `defaultAccount`: معرّف الحساب المفضل عند تكوين عدة حسابات Matrix.
- `homeserver`: عنوان URL للخادم المنزلي، مثل `https://matrix.example.org`.
- `allowPrivateNetwork`: السماح لحساب Matrix هذا بالاتصال بخوادم منزلية خاصة/داخلية. فعّل هذا عندما يُحل الخادم المنزلي إلى `localhost` أو عنوان IP ضمن LAN/Tailscale أو مضيف داخلي مثل `matrix-synapse`.
- `proxy`: عنوان URL اختياري لوكيل HTTP(S) لحركة Matrix. يمكن للحسابات المسمّاة تجاوز القيمة الافتراضية في المستوى الأعلى باستخدام `proxy` الخاص بها.
- `userId`: معرّف مستخدم Matrix الكامل، مثل `@bot:example.org`.
- `accessToken`: access token للمصادقة المستندة إلى token. تُدعَم القيم النصية الصريحة وقيم SecretRef لكل من `channels.matrix.accessToken` و`channels.matrix.accounts.<id>.accessToken` عبر موفري env/file/exec. راجع [إدارة الأسرار](/ar/gateway/secrets).
- `password`: كلمة المرور لتسجيل الدخول المستند إلى كلمة المرور. تُدعَم القيم النصية الصريحة وقيم SecretRef.
- `deviceId`: معرّف جهاز Matrix صريح.
- `deviceName`: اسم عرض الجهاز لتسجيل الدخول بكلمة المرور.
- `avatarUrl`: عنوان URL المخزن للصورة الرمزية الذاتية لمزامنة الملف الشخصي وتحديثات `set-profile`.
- `initialSyncLimit`: حد أحداث المزامنة عند بدء التشغيل.
- `encryption`: تفعيل E2EE.
- `allowlistOnly`: فرض سلوك قائمة السماح فقط للرسائل المباشرة والغرف.
- `allowBots`: السماح برسائل من حسابات Matrix أخرى مُكوّنة في OpenClaw (`true` أو `"mentions"`).
- `groupPolicy`: `open` أو `allowlist` أو `disabled`.
- `contextVisibility`: وضع إظهار سياق الغرفة التكميلي (`all` أو `allowlist` أو `allowlist_quote`).
- `groupAllowFrom`: قائمة سماح لمعرّفات المستخدمين لحركة الغرف.
- يجب أن تكون إدخالات `groupAllowFrom` معرّفات مستخدمي Matrix كاملة. تُتجاهل الأسماء غير المحلولة وقت التشغيل.
- `historyLimit`: الحد الأقصى لرسائل الغرفة التي تُضمّن كسياق لسجل المجموعة. ويعود إلى `messages.groupChat.historyLimit`؛ وإذا لم يُضبط أي منهما، فالقيمة الافتراضية الفعلية هي `0`. اضبط `0` للتعطيل.
- `replyToMode`: `off` أو `first` أو `all`.
- `markdown`: تكوين اختياري لتصيير Markdown لنص Matrix الصادر.
- `streaming`: `off` (الافتراضي) أو `partial` أو `quiet` أو `true` أو `false`. يفعّل `partial` و`true` تحديثات المسودة القائمة على المعاينة أولًا باستخدام رسائل Matrix النصية العادية. ويستخدم `quiet` إشعارات معاينة غير مُشعِرة لإعدادات قواعد push ذاتية الاستضافة.
- `blockStreaming`: يؤدي `true` إلى تفعيل رسائل تقدم منفصلة لكتل المساعد المكتملة أثناء نشاط بث معاينة المسودة.
- `threadReplies`: `off` أو `inbound` أو `always`.
- `threadBindings`: تجاوزات لكل قناة لتوجيه الجلسات المرتبطة بالخيوط ودورة حياتها.
- `startupVerification`: وضع طلب التحقق الذاتي التلقائي عند بدء التشغيل (`if-unverified` أو `off`).
- `startupVerificationCooldownHours`: فترة التهدئة قبل إعادة محاولة طلبات التحقق التلقائية عند بدء التشغيل.
- `textChunkLimit`: حجم تقطيع الرسائل الصادرة.
- `chunkMode`: `length` أو `newline`.
- `responsePrefix`: بادئة رسائل اختيارية للردود الصادرة.
- `ackReaction`: تجاوز اختياري لتفاعل الإقرار لهذه القناة/الحساب.
- `ackReactionScope`: تجاوز اختياري لنطاق تفاعل الإقرار (`group-mentions` أو `group-all` أو `direct` أو `all` أو `none` أو `off`).
- `reactionNotifications`: وضع إشعارات التفاعل الواردة (`own` أو `off`).
- `mediaMaxMb`: الحد الأقصى لحجم الوسائط بالميغابايت لمعالجة وسائط Matrix. وينطبق على الإرسال الصادر ومعالجة الوسائط الواردة.
- `autoJoin`: سياسة الانضمام التلقائي للدعوات (`always` أو `allowlist` أو `off`). الافتراضي: `off`. ينطبق هذا على دعوات Matrix عمومًا، بما في ذلك الدعوات بنمط الرسائل المباشرة، وليس فقط دعوات الغرف/المجموعات. يتخذ OpenClaw هذا القرار في وقت الدعوة، قبل أن يتمكن من تصنيف الغرفة المنضم إليها بشكل موثوق كرسالة مباشرة أو مجموعة.
- `autoJoinAllowlist`: الغرف/الأسماء المستعارة المسموح بها عندما يكون `autoJoin` هو `allowlist`. تُحل إدخالات الأسماء المستعارة إلى معرّفات غرف أثناء معالجة الدعوة؛ ولا يثق OpenClaw بحالة الاسم المستعار التي تعلنها الغرفة المدعو إليها.
- `dm`: كتلة سياسة الرسائل المباشرة (`enabled` و`policy` و`allowFrom` و`sessionScope` و`threadReplies`).
- `dm.policy`: يتحكم في الوصول إلى الرسائل المباشرة بعد انضمام OpenClaw إلى الغرفة وتصنيفها كرسالة مباشرة. ولا يغير ما إذا كانت الدعوة ستُضم تلقائيًا.
- يجب أن تكون إدخالات `dm.allowFrom` معرّفات مستخدمي Matrix كاملة ما لم تكن قد حللتها بالفعل عبر البحث المباشر في الدليل.
- `dm.sessionScope`: `per-user` (الافتراضي) أو `per-room`. استخدم `per-room` عندما تريد أن يحتفظ كل غرفة رسائل مباشرة في Matrix بسياق منفصل حتى لو كان النظير هو نفسه.
- `dm.threadReplies`: تجاوز سياسة الخيوط للرسائل المباشرة فقط (`off` أو `inbound` أو `always`). وهو يتجاوز إعداد `threadReplies` على المستوى الأعلى لكل من موضع الرد وعزل الجلسة في الرسائل المباشرة.
- `execApprovals`: تسليم موافقات exec الأصلي في Matrix (`enabled` و`approvers` و`target` و`agentFilter` و`sessionFilter`).
- `execApprovals.approvers`: معرّفات مستخدمي Matrix المسموح لهم بالموافقة على طلبات exec. وهو اختياري عندما يكون `dm.allowFrom` قد حدد الموافقين بالفعل.
- `execApprovals.target`: `dm | channel | both` (الافتراضي: `dm`).
- `accounts`: تجاوزات مسماة لكل حساب. تعمل القيم ذات المستوى الأعلى في `channels.matrix` كقيم افتراضية لهذه الإدخالات.
- `groups`: خريطة سياسات لكل غرفة. فضّل معرّفات الغرف أو الأسماء المستعارة؛ تُتجاهل أسماء الغرف غير المحلولة وقت التشغيل. تستخدم هوية الجلسة/المجموعة معرّف الغرفة الثابت بعد الحل، بينما تبقى التسميات المقروءة للبشر قادمة من أسماء الغرف.
- `groups.<room>.account`: قصر إدخال غرفة موروث واحد على حساب Matrix محدد في إعدادات الحسابات المتعددة.
- `groups.<room>.allowBots`: تجاوز على مستوى الغرفة للمرسلين من الروبوتات المكوّنة (`true` أو `"mentions"`).
- `groups.<room>.users`: قائمة سماح للمرسلين لكل غرفة.
- `groups.<room>.tools`: تجاوزات السماح/المنع للأدوات لكل غرفة.
- `groups.<room>.autoReply`: تجاوز على مستوى الغرفة لتقييد الذكر. تؤدي `true` إلى تعطيل متطلبات الذكر لتلك الغرفة؛ وتؤدي `false` إلى فرضها مجددًا.
- `groups.<room>.skills`: مرشح Skills اختياري على مستوى الغرفة.
- `groups.<room>.systemPrompt`: مقتطف `systemPrompt` اختياري على مستوى الغرفة.
- `rooms`: اسم مستعار قديم لـ `groups`.
- `actions`: تقييد الأدوات لكل إجراء (`messages` و`reactions` و`pins` و`profile` و`memberInfo` و`channelInfo` و`verification`).

## ذو صلة

- [نظرة عامة على القنوات](/ar/channels) — جميع القنوات المدعومة
- [Pairing](/ar/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [Groups](/ar/channels/groups) — سلوك الدردشة الجماعية وتقييد الذكر
- [توجيه القنوات](/ar/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/ar/gateway/security) — نموذج الوصول والتقوية
