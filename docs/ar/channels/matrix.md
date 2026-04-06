---
read_when:
    - إعداد Matrix في OpenClaw
    - تكوين التشفير من طرف إلى طرف (E2EE) والتحقق في Matrix
summary: حالة دعم Matrix وإعداده وأمثلة التكوين
title: Matrix
x-i18n:
    generated_at: "2026-04-06T03:08:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3e2d84c08d7d5b96db14b914e54f08d25334401cdd92eb890bc8dfb37b0ca2dc
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix هو إضافة القناة المجمعة الخاصة بـ Matrix في OpenClaw.
ويستخدم `matrix-js-sdk` الرسمي ويدعم الرسائل الخاصة، والغرف، والخيوط، والوسائط، والتفاعلات، والاستطلاعات، والموقع، والتشفير من طرف إلى طرف (E2EE).

## الإضافة المجمعة

يأتي Matrix كإضافة مجمعة في إصدارات OpenClaw الحالية، لذلك فإن
البنيات المعبأة العادية لا تحتاج إلى تثبيت منفصل.

إذا كنت تستخدم بنية أقدم أو تثبيتًا مخصصًا يستبعد Matrix، فقم بتثبيته
يدويًا:

التثبيت من npm:

```bash
openclaw plugins install @openclaw/matrix
```

التثبيت من نسخة محلية:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

راجع [Plugins](/ar/tools/plugin) لمعرفة سلوك الإضافات وقواعد التثبيت.

## الإعداد

1. تأكد من أن إضافة Matrix متاحة.
   - إصدارات OpenClaw المعبأة الحالية تتضمنها بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافتها يدويًا باستخدام الأوامر أعلاه.
2. أنشئ حساب Matrix على الخادم المنزلي الخاص بك.
3. قم بتكوين `channels.matrix` بأحد الخيارين:
   - `homeserver` + `accessToken`، أو
   - `homeserver` + `userId` + `password`.
4. أعد تشغيل البوابة.
5. ابدأ رسالة خاصة مع البوت أو قم بدعوته إلى غرفة.

مسارات الإعداد التفاعلية:

```bash
openclaw channels add
openclaw configure --section channels
```

ما الذي يطلبه معالج Matrix فعليًا:

- عنوان URL للخادم المنزلي
- طريقة المصادقة: access token أو كلمة المرور
- معرّف المستخدم فقط عند اختيار مصادقة كلمة المرور
- اسم الجهاز الاختياري
- ما إذا كنت تريد تفعيل E2EE
- ما إذا كنت تريد تكوين الوصول إلى غرف Matrix الآن

سلوك المعالج المهم:

- إذا كانت متغيرات بيئة مصادقة Matrix موجودة بالفعل للحساب المحدد، ولم يكن هذا الحساب قد حفظ المصادقة في التكوين بالفعل، فسيعرض المعالج اختصارًا لمتغيرات البيئة ويكتب فقط `enabled: true` لذلك الحساب.
- عند إضافة حساب Matrix آخر بشكل تفاعلي، يُطبَّع اسم الحساب المُدخل إلى معرّف الحساب المستخدم في التكوين ومتغيرات البيئة. على سبيل المثال، يتحول `Ops Bot` إلى `ops-bot`.
- تقبل مطالبات قائمة السماح للرسائل الخاصة قيم `@user:server` الكاملة مباشرة. لا تعمل أسماء العرض إلا عندما يعثر البحث المباشر في الدليل على تطابق واحد دقيق؛ وإلا يطلب منك المعالج إعادة المحاولة باستخدام معرّف Matrix كامل.
- تقبل مطالبات قائمة السماح للغرف معرّفات الغرف والأسماء المستعارة مباشرة. ويمكنها أيضًا حل أسماء الغرف المنضم إليها مباشرة، لكن الأسماء غير المحلولة لا تُحتفَظ إلا كما كُتبت أثناء الإعداد ويتم تجاهلها لاحقًا بواسطة حل قائمة السماح وقت التشغيل. فضّل استخدام `!room:server` أو `#alias:server`.
- تستخدم هوية الغرفة/الجلسة وقت التشغيل معرّف غرفة Matrix الثابت. تُستخدم الأسماء المستعارة المعلنة للغرفة فقط كمدخلات للبحث، وليس كمفتاح جلسة طويل الأمد أو هوية مجموعة ثابتة.
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

إعداد قائم على كلمة المرور (يتم تخزين token مؤقتًا بعد تسجيل الدخول):

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

يخزن Matrix بيانات الاعتماد المؤقتة في `~/.openclaw/credentials/matrix/`.
يستخدم الحساب الافتراضي `credentials.json`؛ وتستخدم الحسابات المسماة `credentials-<account>.json`.
عندما توجد بيانات اعتماد مخزنة مؤقتًا هناك، يعتبر OpenClaw أن Matrix مُكوَّن لأغراض الإعداد وdoctor واكتشاف حالة القناة حتى إذا لم تكن المصادقة الحالية مضبوطة مباشرة في التكوين.

مكافئات متغيرات البيئة (تُستخدم عندما لا يكون مفتاح التكوين مضبوطًا):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

بالنسبة إلى الحسابات غير الافتراضية، استخدم متغيرات بيئة مقيّدة بالحساب:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

مثال للحساب `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

بالنسبة إلى معرّف الحساب المطبع `ops-bot`، استخدم:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

يهرب Matrix علامات الترقيم في معرّفات الحسابات للحفاظ على خلو متغيرات البيئة المقيّدة من التعارضات.
على سبيل المثال، تتحول `-` إلى `_X2D_`، لذلك تتحول `ops-prod` إلى `MATRIX_OPS_X2D_PROD_*`.

لا يعرض المعالج التفاعلي اختصار متغيرات البيئة إلا عندما تكون متغيرات بيئة المصادقة هذه موجودة بالفعل ولم يكن الحساب المحدد قد حفظ مصادقة Matrix في التكوين مسبقًا.

## مثال التكوين

هذا تكوين أساسي عملي مع اقتران الرسائل الخاصة، وقائمة سماح للغرف، وتمكين E2EE:

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

## معاينات البث

بث الردود في Matrix يتطلب التفعيل الاختياري.

اضبط `channels.matrix.streaming` على `"partial"` عندما تريد أن يرسل OpenClaw معاينة مباشرة واحدة
للرد، ويعدّل تلك المعاينة في مكانها بينما يقوم النموذج بإنشاء النص، ثم ينهيها عندما
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

- `streaming: "off"` هو الإعداد الافتراضي. ينتظر OpenClaw الرد النهائي ويرسله مرة واحدة.
- `streaming: "partial"` ينشئ رسالة معاينة واحدة قابلة للتعديل لكتلة المساعد الحالية باستخدام رسائل Matrix النصية العادية. يحافظ هذا على سلوك الإشعارات القديم في Matrix القائم على المعاينة أولًا، لذلك قد ترسل التطبيقات القياسية إشعارًا عند أول نص معاينة متدفق بدلًا من الكتلة المكتملة.
- `streaming: "quiet"` ينشئ إشعار معاينة هادئًا واحدًا قابلًا للتعديل لكتلة المساعد الحالية. استخدم هذا فقط عندما تقوم أيضًا بتكوين قواعد push للمستلمين من أجل تعديلات المعاينة النهائية.
- `blockStreaming: true` يفعّل رسائل تقدم Matrix منفصلة. مع تفعيل بث المعاينة، يحتفظ Matrix بالمسودة الحية للكتلة الحالية ويحافظ على الكتل المكتملة كرسائل منفصلة.
- عندما يكون بث المعاينة قيد التشغيل و`blockStreaming` متوقفًا، يعدّل Matrix المسودة الحية في مكانها وينهي الحدث نفسه عند انتهاء الكتلة أو الدور.
- إذا لم تعد المعاينة مناسبة ضمن حدث Matrix واحد، يوقف OpenClaw بث المعاينة ويعود إلى التسليم النهائي العادي.
- ما تزال ردود الوسائط ترسل المرفقات بشكل عادي. وإذا تعذر إعادة استخدام معاينة قديمة بأمان، فسيقوم OpenClaw بحذفها قبل إرسال رد الوسائط النهائي.
- تتطلب تعديلات المعاينة استدعاءات إضافية إلى Matrix API. اترك البث متوقفًا إذا كنت تريد السلوك الأكثر تحفظًا فيما يتعلق بحدود المعدل.

لا يقوم `blockStreaming` بتمكين معاينات المسودة بمفرده.
استخدم `streaming: "partial"` أو `streaming: "quiet"` لتعديلات المعاينة؛ ثم أضف `blockStreaming: true` فقط إذا كنت تريد أيضًا أن تبقى كتل المساعد المكتملة مرئية كرسائل تقدم منفصلة.

إذا كنت تحتاج إلى إشعارات Matrix القياسية من دون قواعد push مخصصة، فاستخدم `streaming: "partial"` لسلوك المعاينة أولًا أو اترك `streaming` متوقفًا للتسليم النهائي فقط. مع `streaming: "off"`:

- `blockStreaming: true` يرسل كل كتلة مكتملة كرسالة Matrix عادية ترسل إشعارًا.
- `blockStreaming: false` يرسل فقط الرد النهائي المكتمل كرسالة Matrix عادية ترسل إشعارًا.

### قواعد push مستضافة ذاتيًا للمعاينات النهائية الهادئة

إذا كنت تدير بنية Matrix التحتية الخاصة بك وتريد أن ترسل المعاينات الهادئة إشعارًا فقط عند اكتمال كتلة أو
رد نهائي، فاضبط `streaming: "quiet"` وأضف قاعدة push لكل مستخدم من أجل تعديلات المعاينة النهائية.

يكون هذا عادة إعدادًا على مستوى المستخدم المستلم، وليس تغييرًا عامًا في تكوين الخادم المنزلي:

خريطة سريعة قبل أن تبدأ:

- المستخدم المستلم = الشخص الذي يجب أن يتلقى الإشعار
- مستخدم البوت = حساب OpenClaw Matrix الذي يرسل الرد
- استخدم access token الخاص بالمستخدم المستلم لاستدعاءات API أدناه
- طابق `sender` في قاعدة push مع MXID الكامل لمستخدم البوت

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

2. تأكد من أن الحساب المستلم يتلقى بالفعل إشعارات push العادية من Matrix. تعمل قواعد
   المعاينة الهادئة فقط إذا كان لدى ذلك المستخدم pushers/أجهزة عاملة بالفعل.

3. احصل على access token الخاص بالمستخدم المستلم.
   - استخدم token الخاص بالمستخدم المستلم، وليس token الخاص بالبوت.
   - تكون إعادة استخدام token جلسة عميل موجودة أسهل عادةً.
   - إذا كنت بحاجة إلى إنشاء token جديد، يمكنك تسجيل الدخول عبر Matrix Client-Server API القياسي:

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

4. تحقق من أن الحساب المستلم لديه pushers بالفعل:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

إذا أعاد هذا الاستدعاء عدم وجود pushers/أجهزة نشطة، فأصلح إشعارات Matrix العادية أولًا قبل إضافة
قاعدة OpenClaw أدناه.

يضع OpenClaw علامة على تعديلات المعاينة النصية النهائية فقط باستخدام:

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

- `https://matrix.example.org`: عنوان URL الأساسي للخادم المنزلي لديك
- `$USER_ACCESS_TOKEN`: access token الخاص بالمستخدم المستلم
- `openclaw-finalized-preview-botname`: معرّف قاعدة فريد لهذا البوت لهذا المستخدم المستلم
- `@bot:example.org`: MXID بوت OpenClaw Matrix لديك، وليس MXID الخاص بالمستخدم المستلم

مهم لعمليات الإعداد متعددة البوتات:

- تُفهرس قواعد push بواسطة `ruleId`. تؤدي إعادة تشغيل `PUT` على معرّف القاعدة نفسه إلى تحديث تلك القاعدة.
- إذا كان يجب أن يتلقى مستخدم مستلم واحد إشعارات من عدة حسابات بوت OpenClaw Matrix، فأنشئ قاعدة واحدة لكل بوت مع معرّف قاعدة فريد لكل تطابق `sender`.
- النمط البسيط هو `openclaw-finalized-preview-<botname>`، مثل `openclaw-finalized-preview-ops` أو `openclaw-finalized-preview-support`.

تُقيَّم القاعدة مقابل مرسل الحدث:

- قم بالمصادقة باستخدام token الخاص بالمستخدم المستلم
- طابق `sender` مع MXID الخاص ببوت OpenClaw

6. تحقق من وجود القاعدة:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. اختبر ردًا متدفقًا. في الوضع الهادئ، يجب أن تعرض الغرفة معاينة مسودة هادئة، ويجب أن يرسل
   التعديل النهائي في مكانه إشعارًا واحدًا عند اكتمال الكتلة أو الدور.

إذا احتجت إلى إزالة القاعدة لاحقًا، فاحذف معرّف القاعدة نفسه باستخدام token الخاص بالمستخدم المستلم:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

ملاحظات:

- أنشئ القاعدة باستخدام access token الخاص بالمستخدم المستلم، وليس الخاص بالبوت.
- تُدرج قواعد `override` الجديدة المعرّفة من قبل المستخدم قبل قواعد الكبت الافتراضية، لذلك لا حاجة إلى معلمة ترتيب إضافية.
- يؤثر هذا فقط في تعديلات المعاينة النصية التي يستطيع OpenClaw إنهاءها بأمان في مكانها. ما تزال بدائل الوسائط وبدائل المعاينات القديمة تستخدم تسليم Matrix العادي.
- إذا أظهر `GET /_matrix/client/v3/pushers` عدم وجود pushers، فهذا يعني أن المستخدم لا يملك بعد تسليم push عاملًا من Matrix لهذا الحساب/الجهاز.

#### Synapse

بالنسبة إلى Synapse، يكون الإعداد أعلاه كافيًا عادةً بمفرده:

- لا يلزم أي تغيير خاص في `homeserver.yaml` لإشعارات معاينة OpenClaw النهائية.
- إذا كان نشر Synapse لديك يرسل بالفعل إشعارات Matrix push العادية، فإن token المستخدم + استدعاء `pushrules` أعلاه هما خطوة الإعداد الأساسية.
- إذا كنت تشغّل Synapse خلف reverse proxy أو workers، فتأكد من أن `/_matrix/client/.../pushrules/` يصل إلى Synapse بشكل صحيح.
- إذا كنت تستخدم Synapse workers، فتأكد من أن pushers بحالة جيدة. يتولى تسليم push العملية الرئيسية أو `synapse.app.pusher` / workers المكوّنة لـ pusher.

#### Tuwunel

بالنسبة إلى Tuwunel، استخدم تدفق الإعداد نفسه واستدعاء push-rule API نفسه الموضح أعلاه:

- لا يلزم أي تكوين خاص بـ Tuwunel لوسم المعاينة النهائية نفسه.
- إذا كانت إشعارات Matrix العادية تعمل بالفعل لذلك المستخدم، فإن token المستخدم + استدعاء `pushrules` أعلاه هما خطوة الإعداد الأساسية.
- إذا بدا أن الإشعارات تختفي بينما يكون المستخدم نشطًا على جهاز آخر، فتحقق مما إذا كان `suppress_push_when_active` مفعّلًا. أضاف Tuwunel هذا الخيار في Tuwunel 1.4.2 بتاريخ 12 سبتمبر 2025، ويمكنه عمدًا كبت push إلى الأجهزة الأخرى بينما يكون أحد الأجهزة نشطًا.

## التشفير والتحقق

في الغرف المشفرة (E2EE)، تستخدم أحداث الصور الصادرة `thumbnail_file` بحيث تُشفَّر معاينات الصور إلى جانب المرفق الكامل. وما تزال الغرف غير المشفرة تستخدم `thumbnail_url` العادي. لا يلزم أي تكوين — فالإضافة تكتشف حالة E2EE تلقائيًا.

### غرف بوت إلى بوت

افتراضيًا، يتم تجاهل رسائل Matrix القادمة من حسابات OpenClaw Matrix الأخرى المكوّنة.

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

- `allowBots: true` يقبل الرسائل من حسابات Matrix bot الأخرى المكوّنة في الغرف والرسائل الخاصة المسموح بها.
- `allowBots: "mentions"` يقبل تلك الرسائل فقط عندما تذكر هذا البوت صراحة في الغرف. وتبقى الرسائل الخاصة مسموحًا بها.
- `groups.<room>.allowBots` يتجاوز الإعداد على مستوى الحساب لغرفة واحدة.
- ما يزال OpenClaw يتجاهل الرسائل من معرّف مستخدم Matrix نفسه لتجنب حلقات الرد على الذات.
- لا يكشف Matrix عن علامة bot أصلية هنا؛ يعتبر OpenClaw أن "مكتوبة بواسطة بوت" تعني "مرسلة من حساب Matrix آخر مكوّن على بوابة OpenClaw هذه".

استخدم قوائم سماح صارمة للغرف ومتطلبات الذكر عند تمكين حركة المرور بين البوتات في الغرف المشتركة.

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

حالة مطولة (تشخيصات كاملة):

```bash
openclaw matrix verify status --verbose
```

تضمين مفتاح الاسترداد المخزن في مخرجات قابلة للقراءة آليًا:

```bash
openclaw matrix verify status --include-recovery-key --json
```

تهيئة cross-signing وحالة التحقق:

```bash
openclaw matrix verify bootstrap
```

دعم الحسابات المتعددة: استخدم `channels.matrix.accounts` مع بيانات اعتماد لكل حساب و`name` اختياري. راجع [Configuration reference](/ar/gateway/configuration-reference#multi-account-all-channels) للنمط المشترك.

تشخيصات bootstrap مطولة:

```bash
openclaw matrix verify bootstrap --verbose
```

فرض إعادة تعيين هوية cross-signing جديدة قبل التهيئة:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

تحقق من هذا الجهاز باستخدام مفتاح استرداد:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

تفاصيل تحقق الجهاز المطولة:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

تحقق من سلامة النسخ الاحتياطي لمفاتيح الغرفة:

```bash
openclaw matrix verify backup status
```

تشخيصات مطولة لسلامة النسخ الاحتياطي:

```bash
openclaw matrix verify backup status --verbose
```

استعد مفاتيح الغرف من النسخة الاحتياطية على الخادم:

```bash
openclaw matrix verify backup restore
```

تشخيصات الاستعادة المطولة:

```bash
openclaw matrix verify backup restore --verbose
```

احذف النسخة الاحتياطية الحالية على الخادم وأنشئ خط أساس جديدًا للنسخ الاحتياطي. إذا تعذر تحميل
مفتاح النسخ الاحتياطي المخزن بشكل سليم، فيمكن أن تؤدي إعادة التعيين هذه أيضًا إلى إعادة إنشاء التخزين السري بحيث
تستطيع عمليات البدء البارد المستقبلية تحميل مفتاح النسخ الاحتياطي الجديد:

```bash
openclaw matrix verify backup reset --yes
```

جميع أوامر `verify` موجزة افتراضيًا (بما في ذلك تسجيل SDK الداخلي الهادئ) وتعرض تشخيصات مفصلة فقط مع `--verbose`.
استخدم `--json` للحصول على مخرجات كاملة قابلة للقراءة آليًا عند كتابة السكربتات.

في إعدادات الحسابات المتعددة، تستخدم أوامر Matrix CLI الحساب الافتراضي الضمني لـ Matrix ما لم تمرر `--account <id>`.
إذا قمت بتكوين عدة حسابات مسماة، فاضبط `channels.matrix.defaultAccount` أولًا وإلا ستتوقف عمليات CLI الضمنية هذه وتطلب منك اختيار حساب صراحة.
استخدم `--account` كلما أردت أن تستهدف عمليات التحقق أو الجهاز حسابًا مسمى صراحة:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

عندما يكون التشفير معطلًا أو غير متاح لحساب مسمى، تشير تحذيرات Matrix وأخطاء التحقق إلى مفتاح تكوين ذلك الحساب، على سبيل المثال `channels.matrix.accounts.assistant.encryption`.

### ما معنى "verified"

يعتبر OpenClaw جهاز Matrix هذا متحققًا فقط عندما يكون متحققًا بواسطة هوية cross-signing الخاصة بك.
عمليًا، يكشف `openclaw matrix verify status --verbose` عن ثلاث إشارات ثقة:

- `Locally trusted`: هذا الجهاز موثوق من قبل العميل الحالي فقط
- `Cross-signing verified`: يبلغ SDK أن الجهاز متحقق منه عبر cross-signing
- `Signed by owner`: الجهاز موقّع بواسطة مفتاح self-signing الخاص بك

تصبح `Verified by owner` = `yes` فقط عندما يكون تحقق cross-signing أو توقيع المالك موجودًا.
الثقة المحلية وحدها لا تكفي لكي يعتبر OpenClaw الجهاز متحققًا بالكامل.

### ما الذي يفعله bootstrap

الأمر `openclaw matrix verify bootstrap` هو أمر الإصلاح والإعداد لحسابات Matrix المشفرة.
ويقوم بكل ما يلي بالترتيب:

- يهيئ التخزين السري، مع إعادة استخدام مفتاح استرداد موجود عندما يكون ذلك ممكنًا
- يهيئ cross-signing ويرفع مفاتيح cross-signing العامة الناقصة
- يحاول وضع علامة على الجهاز الحالي وتوقيعه عبر cross-signing
- ينشئ نسخة احتياطية جديدة لمفاتيح الغرف على الخادم إذا لم تكن موجودة بالفعل

إذا كان الخادم المنزلي يتطلب مصادقة تفاعلية لرفع مفاتيح cross-signing، يحاول OpenClaw الرفع أولًا بدون مصادقة، ثم باستخدام `m.login.dummy`، ثم باستخدام `m.login.password` عندما يكون `channels.matrix.password` مضبوطًا.

استخدم `--force-reset-cross-signing` فقط عندما تريد عمدًا تجاهل هوية cross-signing الحالية وإنشاء واحدة جديدة.

إذا كنت تريد عمدًا تجاهل النسخة الاحتياطية الحالية لمفاتيح الغرف والبدء بخط
أساس نسخ احتياطي جديد للرسائل المستقبلية، فاستخدم `openclaw matrix verify backup reset --yes`.
افعل ذلك فقط عندما تقبل أن السجل المشفر القديم غير القابل للاسترداد سيظل
غير متاح وأن OpenClaw قد يعيد إنشاء التخزين السري إذا تعذر تحميل
سر النسخ الاحتياطي الحالي بأمان.

### خط أساس جديد للنسخ الاحتياطي

إذا كنت تريد الحفاظ على عمل الرسائل المشفرة المستقبلية وتقبل فقدان السجل القديم غير القابل للاسترداد، فشغّل هذه الأوامر بالترتيب:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

أضف `--account <id>` إلى كل أمر عندما تريد استهداف حساب Matrix مسمى صراحة.

### سلوك بدء التشغيل

عندما يكون `encryption: true`، يعيّن Matrix افتراضيًا `startupVerification` إلى `"if-unverified"`.
عند بدء التشغيل، إذا ظل هذا الجهاز غير متحقق، فسيطلب Matrix التحقق الذاتي في عميل Matrix آخر،
ويتخطى الطلبات المكررة عندما يكون أحدها معلقًا بالفعل، ويطبّق مهلة تهدئة محلية قبل إعادة المحاولة بعد عمليات إعادة التشغيل.
تعاد محاولة الطلبات الفاشلة أسرع افتراضيًا من عمليات إنشاء الطلبات الناجحة.
اضبط `startupVerification: "off"` لتعطيل طلبات بدء التشغيل التلقائية، أو عدّل `startupVerificationCooldownHours`
إذا كنت تريد نافذة إعادة محاولة أقصر أو أطول.

ينفذ بدء التشغيل أيضًا تمريرة bootstrap محافظة للتشفير تلقائيًا.
تحاول هذه التمريرة أولًا إعادة استخدام التخزين السري الحالي وهوية cross-signing الحالية، وتتجنب إعادة تعيين cross-signing ما لم تشغّل تدفق إصلاح bootstrap صريحًا.

إذا وجد بدء التشغيل حالة bootstrap معطلة وكان `channels.matrix.password` مضبوطًا، فيمكن لـ OpenClaw محاولة مسار إصلاح أكثر صرامة.
إذا كان الجهاز الحالي موقّعًا من المالك بالفعل، فسيحافظ OpenClaw على تلك الهوية بدلًا من إعادة تعيينها تلقائيًا.

الترقية من إضافة Matrix العامة السابقة:

- يعيد OpenClaw تلقائيًا استخدام حساب Matrix نفسه وaccess token وهوية الجهاز كلما أمكن.
- قبل تشغيل أي تغييرات ترحيل قابلة للتنفيذ في Matrix، ينشئ OpenClaw أو يعيد استخدام لقطة استرداد ضمن `~/Backups/openclaw-migrations/`.
- إذا كنت تستخدم عدة حسابات Matrix، فاضبط `channels.matrix.defaultAccount` قبل الترقية من تخطيط التخزين المسطح القديم حتى يعرف OpenClaw أي حساب يجب أن يتلقى تلك الحالة القديمة المشتركة.
- إذا كانت الإضافة السابقة قد خزنت محليًا مفتاح فك تشفير النسخ الاحتياطي لمفاتيح غرف Matrix، فسيقوم بدء التشغيل أو `openclaw doctor --fix` باستيراده إلى تدفق مفتاح الاسترداد الجديد تلقائيًا.
- إذا تغيّر access token الخاص بـ Matrix بعد تحضير الترحيل، فإن بدء التشغيل يفحص الآن جذور التخزين الشقيقة ذات تجزئة token بحثًا عن حالة استعادة قديمة معلقة قبل التخلي عن الاستعادة التلقائية للنسخ الاحتياطي.
- إذا تغيّر access token الخاص بـ Matrix لاحقًا للحساب نفسه والخادم المنزلي نفسه والمستخدم نفسه، يفضّل OpenClaw الآن إعادة استخدام جذر تخزين تجزئة token الأكثر اكتمالًا بدلًا من البدء من دليل حالة Matrix فارغ.
- عند بدء تشغيل البوابة التالي، تُستعاد مفاتيح الغرف المنسوخة احتياطيًا تلقائيًا إلى مخزن التشفير الجديد.
- إذا كانت الإضافة القديمة تملك مفاتيح غرف محلية فقط ولم تُنسخ احتياطيًا مطلقًا، فسيحذر OpenClaw بوضوح. لا يمكن تصدير هذه المفاتيح تلقائيًا من مخزن rust crypto السابق، لذلك قد يظل بعض السجل المشفر القديم غير متاح إلى أن يُستعاد يدويًا.
- راجع [Matrix migration](/ar/install/migrating-matrix) لمعرفة تدفق الترقية الكامل، والقيود، وأوامر الاسترداد، ورسائل الترحيل الشائعة.

تُنظَّم حالة وقت التشغيل المشفرة تحت جذور حسب الحساب، والمستخدم، وتجزئة token ضمن
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
يحتوي هذا الدليل على مخزن المزامنة (`bot-storage.json`) ومخزن التشفير (`crypto/`)،
وملف مفتاح الاسترداد (`recovery-key.json`) ولقطة IndexedDB (`crypto-idb-snapshot.json`)،
وروابط الخيوط (`thread-bindings.json`) وحالة التحقق عند بدء التشغيل (`startup-verification.json`)
عند استخدام هذه الميزات.
عندما يتغير token لكن تبقى هوية الحساب نفسها، يعيد OpenClaw استخدام أفضل
جذر موجود لذلك الثلاثي account/homeserver/user بحيث تبقى حالة المزامنة السابقة، وحالة التشفير، وروابط الخيوط،
وحالة التحقق عند بدء التشغيل مرئية.

### نموذج مخزن Node crypto

يستخدم Matrix E2EE في هذه الإضافة مسار Rust crypto الرسمي الخاص بـ `matrix-js-sdk` في Node.
ويتوقع هذا المسار استخدام تخزين مستمر قائم على IndexedDB عندما تريد أن تبقى حالة التشفير بعد إعادة التشغيل.

يوفر OpenClaw ذلك حاليًا في Node عبر:

- استخدام `fake-indexeddb` كطبقة API لـ IndexedDB كما يتوقعها SDK
- استعادة محتويات Rust crypto IndexedDB من `crypto-idb-snapshot.json` قبل `initRustCrypto`
- حفظ محتويات IndexedDB المحدّثة مرة أخرى إلى `crypto-idb-snapshot.json` بعد التهيئة وأثناء وقت التشغيل
- إجراء restore وpersist للّقطة بشكل تسلسلي مقابل `crypto-idb-snapshot.json` باستخدام قفل ملف استشاري حتى لا تتسابق استمرارية وقت تشغيل البوابة وصيانة CLI على ملف اللقطة نفسه

هذا مجرد توافق/بنية تخزين، وليس تنفيذ تشفير مخصصًا.
ملف اللقطة حالة وقت تشغيل حساسة ويُخزَّن بأذونات ملفات مقيّدة.
ضمن نموذج أمان OpenClaw، يكون مضيف البوابة ودليل حالة OpenClaw المحلي داخل حدود المشغل الموثوق بها بالفعل، لذلك فهذه في المقام الأول مسألة متانة تشغيلية وليست حد ثقة بعيد منفصل.

تحسين مخطط له:

- إضافة دعم SecretRef لمواد مفاتيح Matrix المستمرة بحيث يمكن الحصول على مفاتيح الاسترداد وأسرار تشفير المخزن ذات الصلة من مزودي الأسرار في OpenClaw بدلًا من الملفات المحلية فقط

## إدارة الملف الشخصي

حدّث الملف الشخصي الذاتي في Matrix للحساب المحدد باستخدام:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

أضف `--account <id>` عندما تريد استهداف حساب Matrix مسمى صراحة.

يقبل Matrix عناوين URL للصور الرمزية من نوع `mxc://` مباشرة. وعندما تمرر عنوان URL لصورة رمزية من نوع `http://` أو `https://`، يرفعها OpenClaw إلى Matrix أولًا ويخزن عنوان `mxc://` الذي تم حله مرة أخرى في `channels.matrix.avatarUrl` (أو التجاوز الخاص بالحساب المحدد).

## إشعارات التحقق التلقائية

ينشر Matrix الآن إشعارات دورة حياة التحقق مباشرة في غرفة الرسائل الخاصة الصارمة الخاصة بالتحقق كرسائل `m.notice`.
ويتضمن ذلك:

- إشعارات طلب التحقق
- إشعارات جاهزية التحقق (مع إرشاد صريح "تحقق باستخدام الرموز التعبيرية")
- إشعارات بدء التحقق واكتماله
- تفاصيل SAS (الرموز التعبيرية والأرقام) عندما تكون متاحة

يتم تتبع طلبات التحقق الواردة من عميل Matrix آخر وقبولها تلقائيًا بواسطة OpenClaw.
وبالنسبة إلى تدفقات التحقق الذاتي، يبدأ OpenClaw أيضًا تدفق SAS تلقائيًا عندما يصبح التحقق بالرموز التعبيرية متاحًا ويؤكد جانبه الخاص.
أما بالنسبة إلى طلبات التحقق من مستخدم/جهاز Matrix آخر، فيقبل OpenClaw الطلب تلقائيًا ثم ينتظر متابعة تدفق SAS بشكل طبيعي.
وما يزال عليك مقارنة الرموز التعبيرية أو أرقام SAS في عميل Matrix لديك وتأكيد "إنها متطابقة" هناك لإكمال التحقق.

لا يقبل OpenClaw تلقائيًا التدفقات المكررة التي بدأها ذاتيًا بشكل أعمى. يتخطى بدء التشغيل إنشاء طلب جديد عندما يكون طلب تحقق ذاتي معلقًا بالفعل.

لا تُمرَّر إشعارات نظام/بروتوكول التحقق إلى مسار دردشة الوكيل، لذا لا تنتج `NO_REPLY`.

### نظافة الأجهزة

قد تتراكم أجهزة Matrix القديمة التي يديرها OpenClaw على الحساب وتجعل الثقة في الغرف المشفرة أصعب في الفهم.
اعرضها باستخدام:

```bash
openclaw matrix devices list
```

أزل أجهزة OpenClaw-managed القديمة باستخدام:

```bash
openclaw matrix devices prune-stale
```

### إصلاح الغرفة المباشرة

إذا خرجت حالة الرسائل الخاصة عن التزامن، فقد ينتهي الأمر بـ OpenClaw إلى امتلاك تعيينات `m.direct` قديمة تشير إلى غرف فردية قديمة بدلًا من الرسائل الخاصة الحية. افحص التعيين الحالي لنظير باستخدام:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

وأصلحه باستخدام:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

يبقي الإصلاح المنطق الخاص بـ Matrix داخل الإضافة:

- يفضّل رسالة خاصة صارمة 1:1 معيّنة بالفعل في `m.direct`
- وإلا يعود إلى أي رسالة خاصة صارمة 1:1 منضم إليها حاليًا مع ذلك المستخدم
- وإذا لم توجد رسالة خاصة سليمة، فإنه ينشئ غرفة مباشرة جديدة ويعيد كتابة `m.direct` للإشارة إليها

لا يقوم تدفق الإصلاح بحذف الغرف القديمة تلقائيًا. بل يختار فقط الرسالة الخاصة السليمة ويحدث التعيين حتى تستهدف عمليات إرسال Matrix الجديدة، وإشعارات التحقق، وتدفقات الرسائل الخاصة الأخرى الغرفة الصحيحة مرة أخرى.

## الخيوط

يدعم Matrix خيوط Matrix الأصلية لكل من الردود التلقائية وإرسال أدوات الرسائل.

- يبقي `dm.sessionScope: "per-user"` (الافتراضي) توجيه الرسائل الخاصة في Matrix محصورًا بالمرسل، بحيث يمكن لعدة غرف رسائل خاصة مشاركة جلسة واحدة عندما تُحل إلى النظير نفسه.
- يعزل `dm.sessionScope: "per-room"` كل غرفة رسائل خاصة في Matrix في مفتاح جلسة خاص بها مع الاستمرار في استخدام فحوصات المصادقة وقائمة السماح العادية للرسائل الخاصة.
- ما تزال روابط المحادثات الصريحة في Matrix تتفوق على `dm.sessionScope`، لذلك تبقي الغرف والخيوط المرتبطة هدف الجلسة الذي اختارته.
- يحافظ `threadReplies: "off"` على الردود في المستوى الأعلى ويحافظ على الرسائل الواردة ضمن خيط على جلسة الأصل.
- يرد `threadReplies: "inbound"` داخل خيط فقط عندما تكون الرسالة الواردة موجودة أصلًا في ذلك الخيط.
- يحافظ `threadReplies: "always"` على ردود الغرف داخل خيط متجذر في الرسالة المشغِّلة ويوجه تلك المحادثة عبر الجلسة المطابقة المحصورة بالخيط من أول رسالة مشغِّلة.
- يتجاوز `dm.threadReplies` الإعداد ذي المستوى الأعلى للرسائل الخاصة فقط. على سبيل المثال، يمكنك إبقاء خيوط الغرف معزولة مع إبقاء الرسائل الخاصة مسطحة.
- تتضمن الرسائل الواردة ضمن خيط رسالة جذر الخيط كسياق إضافي للوكيل.
- أصبحت عمليات الإرسال عبر أدوات الرسائل ترث الآن خيط Matrix الحالي تلقائيًا عندما يكون الهدف الغرفة نفسها، أو الهدف نفسه لمستخدم الرسائل الخاصة، ما لم يُوفَّر `threadId` صراحة.
- لا يعمل إعادة استخدام هدف مستخدم الرسائل الخاصة للجلسة نفسها إلا عندما تثبت بيانات الجلسة الوصفية الحالية وجود النظير نفسه في الرسائل الخاصة على حساب Matrix نفسه؛ وإلا يعود OpenClaw إلى التوجيه العادي المحصور بالمستخدم.
- عندما يكتشف OpenClaw أن غرفة رسائل Matrix خاصة تتصادم مع غرفة رسائل خاصة أخرى على جلسة Matrix DM المشتركة نفسها، فإنه ينشر `m.notice` لمرة واحدة في تلك الغرفة يتضمن مخرج `/focus` عند تفعيل روابط الخيوط وتلميح `dm.sessionScope`.
- روابط الخيوط وقت التشغيل مدعومة في Matrix. أوامر `/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` و`/acp spawn` المرتبطة بالخيط تعمل الآن في غرف Matrix والرسائل الخاصة.
- ينشئ `/focus` على مستوى أعلى في غرفة/رسالة خاصة Matrix خيط Matrix جديدًا ويربطه بجلسة الهدف عندما يكون `threadBindings.spawnSubagentSessions=true`.
- يؤدي تشغيل `/focus` أو `/acp spawn --thread here` داخل خيط Matrix موجود إلى ربط ذلك الخيط الحالي بدلًا من ذلك.

## روابط محادثات ACP

يمكن تحويل غرف Matrix والرسائل الخاصة وخيوط Matrix الموجودة إلى مساحات عمل ACP دائمة من دون تغيير سطح الدردشة.

تدفق سريع للمشغل:

- شغّل `/acp spawn codex --bind here` داخل رسالة Matrix الخاصة أو الغرفة أو الخيط الموجود الذي تريد الاستمرار في استخدامه.
- في رسالة Matrix خاصة أو غرفة من المستوى الأعلى، يبقى سطح الدردشة هو الرسالة الخاصة/الغرفة الحالية، وتُوجَّه الرسائل المستقبلية إلى جلسة ACP التي تم إنشاؤها.
- داخل خيط Matrix موجود، يقوم `--bind here` بربط ذلك الخيط الحالي في مكانه.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الربط.

ملاحظات:

- لا يؤدي `--bind here` إلى إنشاء خيط Matrix فرعي.
- لا يلزم `threadBindings.spawnAcpSessions` إلا مع `/acp spawn --thread auto|here`، عندما يحتاج OpenClaw إلى إنشاء خيط Matrix فرعي أو ربطه.

### تكوين ربط الخيوط

يرث Matrix القيم الافتراضية العامة من `session.threadBindings`، ويدعم أيضًا تجاوزات لكل قناة:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

علامات الإنشاء المرتبط بالخيط في Matrix اختيارية التفعيل:

- اضبط `threadBindings.spawnSubagentSessions: true` للسماح لـ `/focus` من المستوى الأعلى بإنشاء خيوط Matrix جديدة وربطها.
- اضبط `threadBindings.spawnAcpSessions: true` للسماح لـ `/acp spawn --thread auto|here` بربط جلسات ACP بخيوط Matrix.

## التفاعلات

يدعم Matrix إجراءات التفاعل الصادرة، وإشعارات التفاعل الواردة، وتفاعلات التأكيد الواردة.

- يتم تقييد أدوات التفاعل الصادرة بواسطة `channels["matrix"].actions.reactions`.
- يضيف `react` تفاعلًا إلى حدث Matrix محدد.
- يسرد `reactions` ملخص التفاعلات الحالي لحدث Matrix محدد.
- يؤدي `emoji=""` إلى إزالة تفاعلات حساب البوت نفسه على ذلك الحدث.
- يؤدي `remove: true` إلى إزالة تفاعل الرمز التعبيري المحدد فقط من حساب البوت.

يُحل نطاق تفاعلات التأكيد باستخدام ترتيب OpenClaw القياسي:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- الرجوع إلى الرمز التعبيري لهوية الوكيل

يُحل نطاق تفاعلات التأكيد بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

يُحل وضع إشعارات التفاعل بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- الافتراضي: `own`

السلوك الحالي:

- `reactionNotifications: "own"` يمرّر أحداث `m.reaction` المضافة عندما تستهدف رسائل Matrix المكتوبة بواسطة البوت.
- `reactionNotifications: "off"` يعطل أحداث نظام التفاعلات.
- ما تزال إزالة التفاعلات غير مُركّبة كأحداث نظام لأن Matrix يعرضها كحالات حذف redactions، وليس كعمليات إزالة `m.reaction` مستقلة.

## سياق السجل

- يتحكم `channels.matrix.historyLimit` في عدد رسائل الغرفة الحديثة التي تُدرج باعتبارها `InboundHistory` عندما تؤدي رسالة غرفة Matrix إلى تشغيل الوكيل.
- ويعود إلى `messages.groupChat.historyLimit`. اضبطه على `0` للتعطيل.
- يقتصر سجل غرف Matrix على الغرف فقط. تستمر الرسائل الخاصة في استخدام سجل الجلسة العادي.
- سجل غرف Matrix معلق فقط: يضع OpenClaw رسائل الغرفة في مخزن مؤقت إذا لم تؤد بعد إلى رد، ثم يلتقط تلك النافذة عند وصول ذكر أو مشغّل آخر.
- لا تُدرج رسالة المشغّل الحالية ضمن `InboundHistory`؛ بل تبقى في جسم الإدخال الرئيسي لذلك الدور.
- تعيد محاولات الحدث نفسه في Matrix استخدام لقطة السجل الأصلية بدلًا من الانجراف إلى رسائل الغرفة الأحدث.

## ظهور السياق

يدعم Matrix عنصر التحكم المشترك `contextVisibility` للسياق الإضافي للغرفة مثل نص الرد الذي تم جلبه، وجذور الخيوط، والسجل المعلق.

- `contextVisibility: "all"` هو الإعداد الافتراضي. يُحتفظ بالسياق الإضافي كما تم استلامه.
- يقوم `contextVisibility: "allowlist"` بتصفية السياق الإضافي إلى المرسلين المسموح لهم وفق فحوصات قائمة السماح النشطة للغرفة/المستخدم.
- يعمل `contextVisibility: "allowlist_quote"` مثل `allowlist`، لكنه يحتفظ مع ذلك برد مقتبس صريح واحد.

يؤثر هذا الإعداد في ظهور السياق الإضافي، وليس في ما إذا كانت الرسالة الواردة نفسها يمكنها تشغيل رد.
ويظل تفويض المشغّلات صادرًا من إعدادات `groupPolicy` و`groups` و`groupAllowFrom` وسياسة الرسائل الخاصة.

## مثال سياسة الرسائل الخاصة والغرف

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

مثال اقتران لرسائل Matrix الخاصة:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

إذا استمر مستخدم Matrix غير معتمد في مراسلتك قبل الموافقة، يعيد OpenClaw استخدام رمز الاقتران المعلق نفسه وقد يرسل رد تذكير مرة أخرى بعد مهلة قصيرة بدلًا من إنشاء رمز جديد.

راجع [Pairing](/ar/channels/pairing) لمعرفة تدفق اقتران الرسائل الخاصة المشترك وتخطيط التخزين.

## موافقات Exec

يمكن لـ Matrix أن يعمل كعميل موافقة exec لحساب Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (اختياري؛ ويعود إلى `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

يجب أن يكون الموافقون معرّفات مستخدمي Matrix مثل `@owner:example.org`. يقوم Matrix بتمكين موافقات exec الأصلية تلقائيًا عندما يكون `enabled` غير مضبوط أو `"auto"` ويكون من الممكن حل معتمِد واحد على الأقل، إما من `execApprovals.approvers` أو من `channels.matrix.dm.allowFrom`. اضبط `enabled: false` لتعطيل Matrix كعميل موافقة أصلي صراحة. وبخلاف ذلك، تعود طلبات الموافقة إلى مسارات الموافقة المكوّنة الأخرى أو سياسة الرجوع لموافقات exec.

يقتصر التوجيه الأصلي في Matrix حاليًا على exec فقط:

- تتحكم `channels.matrix.execApprovals.*` في التوجيه الأصلي للرسائل الخاصة/القنوات لموافقات exec فقط.
- ما تزال موافقات الإضافات تستخدم `/approve` المشترك في الدردشة نفسها بالإضافة إلى أي إعادة توجيه مكوّن في `approvals.plugin`.
- لا يزال بإمكان Matrix إعادة استخدام `channels.matrix.dm.allowFrom` لتفويض موافقات الإضافات عندما يتمكن من استنتاج الموافقين بأمان، لكنه لا يوفّر مسار إرسال أصليًا منفصلًا لموافقات الإضافات عبر الرسائل الخاصة/القنوات.

قواعد التسليم:

- `target: "dm"` يرسل مطالبات الموافقة إلى الرسائل الخاصة للموافقين
- `target: "channel"` يعيد إرسال المطالبة إلى غرفة Matrix أو الرسالة الخاصة الأصلية
- `target: "both"` يرسل إلى الرسائل الخاصة للموافقين وإلى غرفة Matrix أو الرسالة الخاصة الأصلية

تزرع مطالبات الموافقة في Matrix اختصارات التفاعل على رسالة الموافقة الأساسية:

- `✅` = السماح مرة واحدة
- `❌` = الرفض
- `♾️` = السماح دائمًا عندما يكون هذا القرار مسموحًا به بموجب سياسة exec الفعلية

يمكن للموافقين التفاعل على تلك الرسالة أو استخدام أوامر الشرطة المائلة البديلة: `/approve <id> allow-once` أو `/approve <id> allow-always` أو `/approve <id> deny`.

لا يمكن إلا للموافقين الذين تم حلهم الموافقة أو الرفض. ويتضمن التسليم عبر القناة نص الأمر، لذلك لا تفعّل `channel` أو `both` إلا في الغرف الموثوقة.

تعيد مطالبات الموافقة في Matrix استخدام مخطط الموافقة المشترك في النواة. والسطح الأصلي الخاص بـ Matrix مخصّص للنقل فقط لموافقات exec: توجيه الغرف/الرسائل الخاصة وسلوك إرسال/تحديث/حذف الرسائل.

تجاوز لكل حساب:

- `channels.matrix.accounts.<account>.execApprovals`

مستندات ذات صلة: [Exec approvals](/ar/tools/exec-approvals)

## مثال حسابات متعددة

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

تعمل القيم ذات المستوى الأعلى في `channels.matrix` كقيم افتراضية للحسابات المسماة ما لم يتجاوزها حساب ما.
يمكنك تقييد إدخالات الغرف الموروثة بحساب Matrix واحد باستخدام `groups.<room>.account` (أو `rooms.<room>.account` القديم).
تظل الإدخالات التي لا تحتوي على `account` مشتركة بين جميع حسابات Matrix، وما تزال الإدخالات ذات `account: "default"` تعمل عندما يُكوَّن الحساب الافتراضي مباشرة على المستوى الأعلى `channels.matrix.*`.
لا تنشئ القيم الافتراضية الجزئية المشتركة للمصادقة حسابًا افتراضيًا ضمنيًا منفصلًا بمفردها. لا يقوم OpenClaw بتوليف الحساب ذي المستوى الأعلى `default` إلا عندما يمتلك ذلك الافتراضي مصادقة حديثة (`homeserver` مع `accessToken`، أو `homeserver` مع `userId` و`password`)؛ ويمكن للحسابات المسماة أن تظل قابلة للاكتشاف من `homeserver` مع `userId` عندما تلبي بيانات الاعتماد المخزنة مؤقتًا متطلبات المصادقة لاحقًا.
إذا كان Matrix يحتوي بالفعل على حساب مسمى واحد بالضبط، أو كان `defaultAccount` يشير إلى مفتاح حساب مسمى موجود، فإن ترقية الإصلاح/الإعداد من حساب واحد إلى حسابات متعددة تحافظ على ذلك الحساب بدلًا من إنشاء إدخال جديد `accounts.default`. تنتقل فقط مفاتيح المصادقة/التهيئة الخاصة بـ Matrix إلى ذلك الحساب المُرقّى؛ وتبقى مفاتيح سياسة التسليم المشتركة في المستوى الأعلى.
اضبط `defaultAccount` عندما تريد أن يفضّل OpenClaw حساب Matrix مسمى واحدًا للتوجيه الضمني، والاستقصاء، وعمليات CLI.
إذا قمت بتكوين عدة حسابات مسماة، فاضبط `defaultAccount` أو مرّر `--account <id>` لأوامر CLI التي تعتمد على اختيار الحساب الضمني.
مرّر `--account <id>` إلى `openclaw matrix verify ...` و`openclaw matrix devices ...` عندما تريد تجاوز هذا الاختيار الضمني لأمر واحد.

## الخوادم المنزلية الخاصة/ضمن LAN

افتراضيًا، يمنع OpenClaw الخوادم المنزلية الخاصة/الداخلية في Matrix من أجل الحماية من SSRF ما لم
تفعّل ذلك صراحة لكل حساب.

إذا كان خادمك المنزلي يعمل على localhost أو عنوان IP ضمن LAN/Tailscale أو اسم مضيف داخلي، فقم بتمكين
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

مثال إعداد CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

يسمح هذا التفعيل الاختياري فقط بالأهداف الخاصة/الداخلية الموثوقة. وتظل الخوادم المنزلية العامة غير المشفرة مثل
`http://matrix.example.org:8008` محظورة. فضّل `https://` كلما أمكن.

## تمرير حركة Matrix عبر proxy

إذا كان نشر Matrix لديك يحتاج إلى HTTP(S) proxy صريح للخروج، فاضبط `channels.matrix.proxy`:

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

يمكن للحسابات المسماة تجاوز الإعداد الافتراضي ذي المستوى الأعلى باستخدام `channels.matrix.accounts.<id>.proxy`.
ويستخدم OpenClaw إعداد proxy نفسه لحركة Matrix وقت التشغيل ولمجسات حالة الحساب.

## حل الأهداف

يقبل Matrix صيغ الأهداف هذه في أي موضع يطلب منك فيه OpenClaw هدف غرفة أو مستخدم:

- المستخدمون: `@user:server` أو `user:@user:server` أو `matrix:user:@user:server`
- الغرف: `!room:server` أو `room:!room:server` أو `matrix:room:!room:server`
- الأسماء المستعارة: `#alias:server` أو `channel:#alias:server` أو `matrix:channel:#alias:server`

يستخدم البحث المباشر في الدليل حساب Matrix المسجل دخوله:

- تستعلم عمليات بحث المستخدمين دليل مستخدمي Matrix على ذلك الخادم المنزلي.
- تقبل عمليات بحث الغرف معرّفات الغرف والأسماء المستعارة الصريحة مباشرة، ثم تعود إلى البحث في أسماء الغرف المنضم إليها لذلك الحساب.
- يكون البحث في أسماء الغرف المنضم إليها بأفضل جهد. إذا تعذر حل اسم غرفة إلى معرّف أو اسم مستعار، فسيتم تجاهله بواسطة حل قائمة السماح وقت التشغيل.

## مرجع التكوين

- `enabled`: تفعيل القناة أو تعطيلها.
- `name`: تسمية اختيارية للحساب.
- `defaultAccount`: معرّف الحساب المفضل عند تكوين عدة حسابات Matrix.
- `homeserver`: عنوان URL للخادم المنزلي، مثل `https://matrix.example.org`.
- `allowPrivateNetwork`: السماح لحساب Matrix هذا بالاتصال بالخوادم المنزلية الخاصة/الداخلية. فعّل هذا عندما يُحل الخادم المنزلي إلى `localhost` أو عنوان IP ضمن LAN/Tailscale أو مضيف داخلي مثل `matrix-synapse`.
- `proxy`: عنوان URL اختياري لـ HTTP(S) proxy لحركة Matrix. يمكن للحسابات المسماة تجاوز الإعداد الافتراضي ذي المستوى الأعلى باستخدام `proxy` الخاص بها.
- `userId`: معرّف مستخدم Matrix الكامل، مثل `@bot:example.org`.
- `accessToken`: access token للمصادقة القائمة على token. تُدعَم القيم النصية الصريحة وقيم SecretRef لكل من `channels.matrix.accessToken` و`channels.matrix.accounts.<id>.accessToken` عبر موفري env/file/exec. راجع [Secrets Management](/ar/gateway/secrets).
- `password`: كلمة المرور لتسجيل الدخول القائم على كلمة المرور. تُدعَم القيم النصية الصريحة وقيم SecretRef.
- `deviceId`: معرّف جهاز Matrix صريح.
- `deviceName`: اسم عرض الجهاز لتسجيل الدخول بكلمة المرور.
- `avatarUrl`: عنوان URL للصورة الرمزية الذاتية المخزنة لمزامنة الملف الشخصي وتحديثات `set-profile`.
- `initialSyncLimit`: حد أحداث المزامنة عند بدء التشغيل.
- `encryption`: تمكين E2EE.
- `allowlistOnly`: فرض سلوك قائمة السماح فقط للرسائل الخاصة والغرف.
- `allowBots`: السماح بالرسائل من حسابات OpenClaw Matrix الأخرى المكوّنة (`true` أو `"mentions"`).
- `groupPolicy`: `open` أو `allowlist` أو `disabled`.
- `contextVisibility`: وضع ظهور سياق الغرفة الإضافي (`all` أو `allowlist` أو `allowlist_quote`).
- `groupAllowFrom`: قائمة سماح لمعرّفات المستخدمين لحركة الغرف.
- يجب أن تكون إدخالات `groupAllowFrom` معرّفات مستخدم Matrix كاملة. وتُتجاهل الأسماء غير المحلولة وقت التشغيل.
- `historyLimit`: الحد الأقصى لرسائل الغرفة التي تُدرج كسياق سجل المجموعة. ويعود إلى `messages.groupChat.historyLimit`. اضبطه على `0` للتعطيل.
- `replyToMode`: `off` أو `first` أو `all`.
- `markdown`: تكوين اختياري لتصيير Markdown للنص الصادر في Matrix.
- `streaming`: `off` (الافتراضي) أو `partial` أو `quiet` أو `true` أو `false`. يؤدي `partial` و`true` إلى تمكين تحديثات المسودة القائمة على المعاينة أولًا باستخدام رسائل Matrix النصية العادية. ويستخدم `quiet` إشعارات معاينة غير مرسلة للإشعار لعمليات إعداد push-rule المستضافة ذاتيًا.
- `blockStreaming`: تؤدي القيمة `true` إلى تمكين رسائل تقدم منفصلة لكتل المساعد المكتملة أثناء نشاط بث معاينة المسودة.
- `threadReplies`: `off` أو `inbound` أو `always`.
- `threadBindings`: تجاوزات لكل قناة لتوجيه الجلسات المرتبط بالخيوط ودورة حياتها.
- `startupVerification`: وضع طلب التحقق الذاتي التلقائي عند بدء التشغيل (`if-unverified` أو `off`).
- `startupVerificationCooldownHours`: مهلة التهدئة قبل إعادة محاولة طلبات التحقق التلقائي عند بدء التشغيل.
- `textChunkLimit`: حجم تقسيم الرسالة الصادرة.
- `chunkMode`: `length` أو `newline`.
- `responsePrefix`: بادئة رسالة اختيارية للردود الصادرة.
- `ackReaction`: تجاوز اختياري لتفاعل التأكيد لهذه القناة/الحساب.
- `ackReactionScope`: تجاوز اختياري لنطاق تفاعل التأكيد (`group-mentions` أو `group-all` أو `direct` أو `all` أو `none` أو `off`).
- `reactionNotifications`: وضع إشعارات التفاعل الواردة (`own` أو `off`).
- `mediaMaxMb`: الحد الأقصى لحجم الوسائط بالميغابايت لمعالجة وسائط Matrix. وينطبق على الإرسال الصادر ومعالجة الوسائط الواردة.
- `autoJoin`: سياسة الانضمام التلقائي إلى الدعوات (`always` أو `allowlist` أو `off`). الافتراضي: `off`.
- `autoJoinAllowlist`: الغرف/الأسماء المستعارة المسموح بها عندما تكون `autoJoin` هي `allowlist`. تُحل إدخالات الأسماء المستعارة إلى معرّفات غرف أثناء معالجة الدعوات؛ ولا يثق OpenClaw بحالة الاسم المستعار التي تدّعيها الغرفة المدعو إليها.
- `dm`: كتلة سياسة الرسائل الخاصة (`enabled` و`policy` و`allowFrom` و`sessionScope` و`threadReplies`).
- يجب أن تكون إدخالات `dm.allowFrom` معرّفات مستخدم Matrix كاملة ما لم تكن قد حللتها بالفعل عبر البحث المباشر في الدليل.
- `dm.sessionScope`: `per-user` (الافتراضي) أو `per-room`. استخدم `per-room` عندما تريد أن تحتفظ كل غرفة رسائل خاصة في Matrix بسياق منفصل حتى لو كان النظير نفسه.
- `dm.threadReplies`: تجاوز لسياسة الخيوط خاص بالرسائل الخاصة (`off` أو `inbound` أو `always`). وهو يتجاوز الإعداد ذي المستوى الأعلى `threadReplies` لكل من موضع الرد وعزل الجلسة في الرسائل الخاصة.
- `execApprovals`: تسليم موافقات exec الأصلية في Matrix (`enabled` و`approvers` و`target` و`agentFilter` و`sessionFilter`).
- `execApprovals.approvers`: معرّفات مستخدمي Matrix المسموح لهم بالموافقة على طلبات exec. وهو اختياري عندما يحدد `dm.allowFrom` الموافقين بالفعل.
- `execApprovals.target`: `dm | channel | both` (الافتراضي: `dm`).
- `accounts`: تجاوزات مسماة لكل حساب. تعمل القيم ذات المستوى الأعلى في `channels.matrix` كقيم افتراضية لهذه الإدخالات.
- `groups`: خريطة سياسة لكل غرفة. فضّل معرّفات الغرف أو الأسماء المستعارة؛ وتُتجاهل أسماء الغرف غير المحلولة وقت التشغيل. تستخدم هوية الجلسة/المجموعة معرّف الغرفة الثابت بعد الحل، بينما تبقى التسميات المقروءة للبشر مستمدة من أسماء الغرف.
- `groups.<room>.account`: قصر إدخال غرفة موروث واحد على حساب Matrix محدد في إعدادات الحسابات المتعددة.
- `groups.<room>.allowBots`: تجاوز على مستوى الغرفة للمرسلين المكوّنين كبوت (`true` أو `"mentions"`).
- `groups.<room>.users`: قائمة سماح للمرسلين لكل غرفة.
- `groups.<room>.tools`: تجاوزات السماح/المنع للأدوات لكل غرفة.
- `groups.<room>.autoReply`: تجاوز على مستوى الغرفة لتقييد الذكر. يؤدي `true` إلى تعطيل متطلبات الذكر لتلك الغرفة؛ ويعيد `false` فرضها.
- `groups.<room>.skills`: عامل تصفية Skills اختياري على مستوى الغرفة.
- `groups.<room>.systemPrompt`: مقتطف system prompt اختياري على مستوى الغرفة.
- `rooms`: اسم مستعار قديم لـ `groups`.
- `actions`: تقييد الأدوات لكل إجراء (`messages` و`reactions` و`pins` و`profile` و`memberInfo` و`channelInfo` و`verification`).

## ذو صلة

- [Channels Overview](/ar/channels) — جميع القنوات المدعومة
- [Pairing](/ar/channels/pairing) — مصادقة الرسائل الخاصة وتدفق الاقتران
- [Groups](/ar/channels/groups) — سلوك الدردشة الجماعية وتقييد الذكر
- [Channel Routing](/ar/channels/channel-routing) — توجيه الجلسات للرسائل
- [Security](/ar/gateway/security) — نموذج الوصول والتقوية
