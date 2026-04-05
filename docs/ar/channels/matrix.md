---
read_when:
    - إعداد Matrix في OpenClaw
    - تهيئة E2EE والتحقق في Matrix
summary: حالة دعم Matrix، والإعداد، وأمثلة التهيئة
title: Matrix
x-i18n:
    generated_at: "2026-04-05T12:37:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: ba5c49ad2125d97adf66b5517f8409567eff8b86e20224a32fcb940a02cb0659
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix هي إضافة القناة المدمجة Matrix في OpenClaw.
وهي تستخدم `matrix-js-sdk` الرسمي وتدعم الرسائل المباشرة، والغرف، وسلاسل الرسائل، والوسائط، والتفاعلات، والاستطلاعات، والموقع، وE2EE.

## الإضافة المدمجة

تأتي Matrix كإضافة مدمجة في إصدارات OpenClaw الحالية، لذلك لا تحتاج
البنيات المجمعة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستبعد Matrix، فقم بتثبيته
يدويًا:

التثبيت من npm:

```bash
openclaw plugins install @openclaw/matrix
```

التثبيت من نسخة محلية:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

راجع [Plugins](/tools/plugin) لمعرفة سلوك الإضافات وقواعد التثبيت.

## الإعداد

1. تأكد من توفر إضافة Matrix.
   - إصدارات OpenClaw المجمعة الحالية تتضمنها بالفعل.
   - يمكن للتثبيتات الأقدم/المخصصة إضافتها يدويًا باستخدام الأوامر أعلاه.
2. أنشئ حساب Matrix على خادمك المنزلي.
3. هيئ `channels.matrix` باستخدام أحد الخيارين:
   - `homeserver` + `accessToken`، أو
   - `homeserver` + `userId` + `password`.
4. أعد تشغيل Gateway.
5. ابدأ رسالة مباشرة مع البوت أو ادعه إلى غرفة.

مسارات الإعداد التفاعلي:

```bash
openclaw channels add
openclaw configure --section channels
```

ما الذي يطلبه معالج Matrix فعليًا:

- عنوان URL للخادم المنزلي
- طريقة المصادقة: access token أو كلمة مرور
- معرّف المستخدم فقط عند اختيار المصادقة بكلمة المرور
- اسم الجهاز اختياريًا
- ما إذا كنت تريد تمكين E2EE
- ما إذا كنت تريد تهيئة وصول غرفة Matrix الآن

سلوك المعالج المهم:

- إذا كانت متغيرات بيئة مصادقة Matrix موجودة بالفعل للحساب المحدد، وكان هذا الحساب لا يحتوي بالفعل على بيانات مصادقة محفوظة في الإعداد، فإن المعالج يعرض اختصار env ويكتب فقط `enabled: true` لذلك الحساب.
- عند إضافة حساب Matrix آخر بشكل تفاعلي، يتم تطبيع اسم الحساب الذي أدخلته إلى معرّف الحساب المستخدم في الإعداد ومتغيرات env. على سبيل المثال، تصبح `Ops Bot` هي `ops-bot`.
- تقبل مطالبات قائمة السماح للرسائل المباشرة قيم `@user:server` الكاملة مباشرة. ولا تعمل أسماء العرض إلا عندما يعثر البحث الحي في الدليل على تطابق واحد دقيق؛ وإلا يطلب منك المعالج إعادة المحاولة باستخدام معرّف Matrix كامل.
- تقبل مطالبات قائمة السماح للغرف معرّفات الغرف والأسماء المستعارة مباشرة. ويمكنها أيضًا حل أسماء الغرف المنضم إليها حيًا، لكن الأسماء غير المحلولة لا يتم الاحتفاظ بها إلا كما كُتبت أثناء الإعداد ويتم تجاهلها لاحقًا بواسطة حل قائمة السماح في وقت التشغيل. فضّل `!room:server` أو `#alias:server`.
- تستخدم هوية الغرفة/الجلسة في وقت التشغيل معرّف غرفة Matrix الثابت. ولا تُستخدم الأسماء المستعارة المعلنة للغرفة إلا كمدخلات بحث، وليس كمفتاح جلسة طويل الأمد أو هوية مجموعة ثابتة.
- لحل أسماء الغرف قبل حفظها، استخدم `openclaw channels resolve --channel matrix "Project Room"`.

إعداد أساسي قائم على الرمز المميز:

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

إعداد قائم على كلمة المرور (يتم تخزين الرمز المميز مؤقتًا بعد تسجيل الدخول):

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

تخزن Matrix بيانات الاعتماد المؤقتة في `~/.openclaw/credentials/matrix/`.
ويستخدم الحساب الافتراضي `credentials.json`؛ وتستخدم الحسابات المسماة `credentials-<account>.json`.

المكافئات في متغيرات البيئة (تُستخدم عندما لا يكون مفتاح الإعداد مضبوطًا):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

للحسابات غير الافتراضية، استخدم متغيرات env ضمن نطاق الحساب:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

مثال للحساب `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

وبالنسبة إلى معرّف الحساب المطبع `ops-bot`، استخدم:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

تهرب Matrix علامات الترقيم في معرّفات الحسابات للحفاظ على متغيرات env ضمن النطاق بدون تعارضات.
على سبيل المثال، تتحول `-` إلى `_X2D_`، لذا يُحوَّل `ops-prod` إلى `MATRIX_OPS_X2D_PROD_*`.

لا يعرض المعالج التفاعلي اختصار متغيرات env إلا عندما تكون متغيرات env الخاصة بالمصادقة موجودة بالفعل ولا يكون الحساب المحدد قد حفظ بالفعل مصادقة Matrix في الإعداد.

## مثال على التهيئة

هذه تهيئة أساسية عملية مع اقتران الرسائل المباشرة، وقائمة سماح للغرف، وE2EE مفعّل:

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

بث الردود في Matrix اختياري.

اضبط `channels.matrix.streaming` على `"partial"` عندما تريد أن يرسل OpenClaw مسودة رد واحدة،
ويعدل هذه المسودة في مكانها أثناء توليد النموذج للنص، ثم ينهيها عند اكتمال
الرد:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- القيمة الافتراضية هي `streaming: "off"`. ينتظر OpenClaw الرد النهائي ثم يرسله مرة واحدة.
- ينشئ `streaming: "partial"` رسالة معاينة واحدة قابلة للتعديل لكتلة المساعد الحالية بدلًا من إرسال عدة رسائل جزئية.
- يفعّل `blockStreaming: true` رسائل تقدم Matrix منفصلة. ومع `streaming: "partial"`، تحتفظ Matrix بالمسودة الحية للكتلة الحالية وتحافظ على الكتل المكتملة كرسائل منفصلة.
- عندما تكون `streaming: "partial"` و`blockStreaming` معطلة، لا تقوم Matrix إلا بتعديل المسودة الحية وترسل الرد المكتمل عند انتهاء تلك الكتلة أو الدورة.
- إذا لم تعد المعاينة تتسع لحدث Matrix واحد، يوقف OpenClaw بث المعاينة ويرجع إلى الإرسال النهائي العادي.
- تواصل ردود الوسائط إرسال المرفقات بشكل طبيعي. وإذا تعذر إعادة استخدام معاينة قديمة بأمان، يقوم OpenClaw بتنقيحها قبل إرسال رد الوسائط النهائي.
- تعديلات المعاينة تستهلك استدعاءات إضافية إلى Matrix API. اترك البث معطلًا إذا كنت تريد السلوك الأكثر تحفظًا فيما يتعلق بحدود المعدل.

لا يقوم `blockStreaming` وحده بتمكين معاينات المسودات.
استخدم `streaming: "partial"` لتعديلات المعاينة؛ ثم أضف `blockStreaming: true` فقط إذا كنت تريد أيضًا أن تظل كتل المساعد المكتملة مرئية كرسائل تقدم منفصلة.

## التشفير والتحقق

في الغرف المشفرة (E2EE)، تستخدم أحداث الصور الصادرة `thumbnail_file` بحيث تكون معاينات الصور مشفرة إلى جانب المرفق الكامل. أما الغرف غير المشفرة فما زالت تستخدم `thumbnail_url` العادي. لا يلزم أي إعداد — تكتشف الإضافة حالة E2EE تلقائيًا.

### غرف بوت إلى بوت

افتراضيًا، يتم تجاهل رسائل Matrix القادمة من حسابات Matrix OpenClaw أخرى مهيأة.

استخدم `allowBots` عندما تريد عمدًا حركة Matrix بين الوكلاء:

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

- يقبل `allowBots: true` الرسائل من حسابات Matrix bot أخرى مهيأة في الغرف المسموح بها والرسائل المباشرة.
- يقبل `allowBots: "mentions"` تلك الرسائل فقط عندما تشير بوضوح إلى هذا البوت في الغرف. وتظل الرسائل المباشرة مسموحًا بها.
- تتجاوز `groups.<room>.allowBots` الإعداد على مستوى الحساب لغرفة واحدة.
- يواصل OpenClaw تجاهل الرسائل من معرّف مستخدم Matrix نفسه لتجنب حلقات الرد الذاتي.
- لا تكشف Matrix هنا عن علامة bot أصلية؛ ويتعامل OpenClaw مع "منشأ بواسطة bot" على أنه "أرسلته حساب Matrix آخر مهيأ على Gateway OpenClaw هذا".

استخدم قوائم سماح صارمة للغرف ومتطلبات الإشارة عند تمكين حركة bot-to-bot في الغرف المشتركة.

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

التحقق من حالة التحقق:

```bash
openclaw matrix verify status
```

حالة مفصلة (تشخيصات كاملة):

```bash
openclaw matrix verify status --verbose
```

تضمين مفتاح الاسترداد المخزن في مخرجات قابلة للقراءة آليًا:

```bash
openclaw matrix verify status --include-recovery-key --json
```

تهيئة حالة cross-signing والتحقق:

```bash
openclaw matrix verify bootstrap
```

دعم الحسابات المتعددة: استخدم `channels.matrix.accounts` مع بيانات اعتماد لكل حساب و`name` اختياري. راجع [مرجع التهيئة](/gateway/configuration-reference#multi-account-all-channels) للنمط المشترك.

تشخيصات bootstrap مفصلة:

```bash
openclaw matrix verify bootstrap --verbose
```

فرض إعادة تعيين هوية cross-signing جديدة قبل bootstrap:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

تحقق من هذا الجهاز باستخدام مفتاح استرداد:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

تفاصيل مفصلة للتحقق من الجهاز:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

التحقق من سلامة النسخ الاحتياطي لمفاتيح الغرف:

```bash
openclaw matrix verify backup status
```

تشخيصات مفصلة لسلامة النسخ الاحتياطي:

```bash
openclaw matrix verify backup status --verbose
```

استعادة مفاتيح الغرف من النسخ الاحتياطي على الخادم:

```bash
openclaw matrix verify backup restore
```

تشخيصات مفصلة للاستعادة:

```bash
openclaw matrix verify backup restore --verbose
```

احذف النسخة الاحتياطية الحالية على الخادم وأنشئ خط أساس جديدًا للنسخ الاحتياطي. وإذا تعذر تحميل
مفتاح النسخ الاحتياطي المخزن بشكل نظيف، يمكن لإعادة التعيين هذه أيضًا إعادة إنشاء التخزين السري لكي
تتمكن عمليات البدء البارد المستقبلية من تحميل مفتاح النسخ الاحتياطي الجديد:

```bash
openclaw matrix verify backup reset --yes
```

تكون جميع أوامر `verify` موجزة افتراضيًا (بما في ذلك تسجيلات SDK الداخلية الهادئة) ولا تعرض تشخيصات مفصلة إلا مع `--verbose`.
استخدم `--json` للحصول على مخرجات كاملة قابلة للقراءة آليًا عند استخدام البرامج النصية.

في الإعدادات متعددة الحسابات، تستخدم أوامر Matrix في CLI الحساب الافتراضي الضمني لـ Matrix ما لم تمرر `--account <id>`.
وإذا هيأت عدة حسابات مسماة، فاضبط أولًا `channels.matrix.defaultAccount` وإلا فإن عمليات CLI الضمنية تلك ستتوقف وتطلب منك اختيار حساب صراحة.
استخدم `--account` كلما أردت أن تستهدف عمليات التحقق أو الأجهزة حسابًا مسمى صراحة:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

عندما يكون التشفير معطلًا أو غير متاح لحساب مسمى، تشير تحذيرات Matrix وأخطاء التحقق إلى مفتاح إعداد ذلك الحساب، مثل `channels.matrix.accounts.assistant.encryption`.

### ما معنى "تم التحقق"

يتعامل OpenClaw مع جهاز Matrix هذا على أنه تم التحقق منه فقط عندما تتحقق منه هوية cross-signing الخاصة بك.
عمليًا، يكشف `openclaw matrix verify status --verbose` ثلاث إشارات ثقة:

- `Locally trusted`: هذا الجهاز موثوق به من قبل العميل الحالي فقط
- `Cross-signing verified`: يفيد SDK بأن الجهاز تم التحقق منه عبر cross-signing
- `Signed by owner`: تم توقيع الجهاز بواسطة مفتاح self-signing الخاص بك

تصبح قيمة `Verified by owner` هي `yes` فقط عند وجود تحقق cross-signing أو توقيع من المالك.
ولا تكفي الثقة المحلية وحدها لكي يتعامل OpenClaw مع الجهاز على أنه متحقق منه بالكامل.

### ماذا يفعل bootstrap

الأمر `openclaw matrix verify bootstrap` هو أمر الإصلاح والإعداد لحسابات Matrix المشفرة.
وهو ينفذ كل ما يلي بالترتيب:

- يهيئ التخزين السري، مع إعادة استخدام مفتاح استرداد موجود متى أمكن
- يهيئ cross-signing ويرفع مفاتيح cross-signing العامة الناقصة
- يحاول وسم الجهاز الحالي وتوقيعه عبر cross-signing
- ينشئ نسخة احتياطية جديدة لمفاتيح الغرف على الخادم إذا لم تكن موجودة بالفعل

إذا كان الخادم المنزلي يتطلب مصادقة تفاعلية لرفع مفاتيح cross-signing، فإن OpenClaw يحاول الرفع أولًا بلا مصادقة، ثم باستخدام `m.login.dummy`، ثم باستخدام `m.login.password` عندما تكون `channels.matrix.password` مهيأة.

استخدم `--force-reset-cross-signing` فقط عندما تريد عمدًا تجاهل هوية cross-signing الحالية وإنشاء هوية جديدة.

إذا كنت تريد عمدًا تجاهل النسخة الاحتياطية الحالية لمفاتيح الغرف وبدء
خط أساس جديد للنسخ الاحتياطي للرسائل المستقبلية، فاستخدم `openclaw matrix verify backup reset --yes`.
ولا تفعل ذلك إلا إذا كنت تقبل أن السجل المشفر القديم غير القابل للاسترداد سيظل
غير متاح، وأن OpenClaw قد يعيد إنشاء التخزين السري إذا تعذر تحميل
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

عندما تكون `encryption: true`، تضبط Matrix القيمة الافتراضية لـ `startupVerification` على `"if-unverified"`.
عند بدء التشغيل، إذا ظل هذا الجهاز غير متحقق منه، فستطلب Matrix تحققًا ذاتيًا في عميل Matrix آخر،
وستتخطى الطلبات المكررة ما دام هناك طلب قيد الانتظار بالفعل، وستطبق فترة تهدئة محلية قبل إعادة المحاولة بعد إعادة التشغيل.
وتُعاد محاولة الطلبات الفاشلة بشكل أسرع من إنشاء الطلبات الناجح افتراضيًا.
اضبط `startupVerification: "off"` لتعطيل طلبات بدء التشغيل التلقائية، أو عدّل `startupVerificationCooldownHours`
إذا كنت تريد نافذة إعادة محاولة أقصر أو أطول.

ينفذ بدء التشغيل أيضًا تمريرة bootstrap تشفيرية محافظة تلقائيًا.
تحاول هذه التمريرة أولًا إعادة استخدام التخزين السري الحالي وهوية cross-signing الحالية، وتتجنب إعادة تعيين cross-signing ما لم تشغّل مسار إصلاح bootstrap صريحًا.

إذا اكتشف بدء التشغيل حالة bootstrap معطلة وكانت `channels.matrix.password` مهيأة، يمكن لـ OpenClaw محاولة مسار إصلاح أكثر صرامة.
وإذا كان الجهاز الحالي موقعًا بالفعل من المالك، فإن OpenClaw يحافظ على تلك الهوية بدلًا من إعادة تعيينها تلقائيًا.

الترقية من إضافة Matrix العامة السابقة:

- يعيد OpenClaw تلقائيًا استخدام حساب Matrix نفسه، وaccess token نفسه، وهوية الجهاز نفسها متى أمكن.
- قبل تشغيل أي تغييرات ترحيل Matrix قابلة للتنفيذ، ينشئ OpenClaw أو يعيد استخدام لقطة استرداد ضمن `~/Backups/openclaw-migrations/`.
- إذا كنت تستخدم عدة حسابات Matrix، فاضبط `channels.matrix.defaultAccount` قبل الترقية من تخطيط التخزين المسطح القديم لكي يعرف OpenClaw أي حساب يجب أن يستقبل هذه الحالة القديمة المشتركة.
- إذا كانت الإضافة السابقة تخزن محليًا مفتاح فك تشفير النسخ الاحتياطي لمفاتيح غرف Matrix، فسيقوم بدء التشغيل أو `openclaw doctor --fix` باستيراده إلى تدفق مفتاح الاسترداد الجديد تلقائيًا.
- إذا تغير access token الخاص بـ Matrix بعد إعداد الترحيل، فإن بدء التشغيل يفحص الآن جذور تخزين تجزئة الرموز المجاورة بحثًا عن حالة استعادة قديمة معلقة قبل التخلي عن الاستعادة التلقائية للنسخ الاحتياطي.
- إذا تغير access token الخاص بـ Matrix لاحقًا للحساب نفسه والخادم المنزلي نفسه والمستخدم نفسه، فإن OpenClaw يفضل الآن إعادة استخدام جذر تخزين تجزئة الرمز الأكثر اكتمالًا بدلًا من البدء من دليل حالة Matrix فارغ.
- عند بدء تشغيل Gateway التالي، تتم استعادة مفاتيح الغرف المنسوخة احتياطيًا تلقائيًا إلى مخزن التشفير الجديد.
- إذا كانت الإضافة القديمة تحتوي على مفاتيح غرف محلية فقط ولم تُنسخ احتياطيًا مطلقًا، فسيحذر OpenClaw بوضوح. ولا يمكن تصدير هذه المفاتيح تلقائيًا من مخزن rust crypto السابق، لذلك قد يظل بعض السجل المشفر القديم غير متاح حتى تتم استعادته يدويًا.
- راجع [ترحيل Matrix](/install/migrating-matrix) للتدفق الكامل للترقية، والقيود، وأوامر الاسترداد، ورسائل الترحيل الشائعة.

تُنظم حالة وقت التشغيل المشفرة ضمن جذور لكل حساب، ولكل مستخدم، ولكل token-hash في
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
ويحتوي هذا الدليل على مخزن المزامنة (`bot-storage.json`) ومخزن التشفير (`crypto/`)،
وملف مفتاح الاسترداد (`recovery-key.json`)، ولقطة IndexedDB (`crypto-idb-snapshot.json`)،
وروابط سلاسل الرسائل (`thread-bindings.json`)، وحالة التحقق عند بدء التشغيل (`startup-verification.json`)
عندما تكون هذه الميزات قيد الاستخدام.
وعندما يتغير الرمز المميز مع بقاء هوية الحساب نفسها، يعيد OpenClaw استخدام أفضل
جذر موجود لذلك الثلاثي account/homeserver/user بحيث تظل حالة المزامنة السابقة، وحالة التشفير، وروابط سلاسل الرسائل،
وحالة التحقق عند بدء التشغيل مرئية.

### نموذج مخزن التشفير في Node

تستخدم Matrix E2EE في هذه الإضافة مسار Rust crypto الرسمي الخاص بـ `matrix-js-sdk` في Node.
ويتوقع هذا المسار تخزينًا دائمًا قائمًا على IndexedDB عندما تريد أن تستمر حالة التشفير بعد إعادة التشغيل.

يوفر OpenClaw ذلك حاليًا في Node من خلال:

- استخدام `fake-indexeddb` كطبقة واجهة IndexedDB المتوقعة من SDK
- استعادة محتويات IndexedDB الخاصة بـ Rust crypto من `crypto-idb-snapshot.json` قبل `initRustCrypto`
- حفظ محتويات IndexedDB المحدثة مرة أخرى إلى `crypto-idb-snapshot.json` بعد التهيئة وأثناء وقت التشغيل
- تسلسل استعادة اللقطة وحفظها مقابل `crypto-idb-snapshot.json` باستخدام قفل ملف استشاري حتى لا يتسابق حفظ وقت تشغيل Gateway وصيانة CLI على ملف اللقطة نفسه

هذا هو منطق توافق/تخزين، وليس تنفيذًا مخصصًا للتشفير.
ملف اللقطة هو حالة وقت تشغيل حساسة ويتم تخزينه بأذونات ملفات مقيّدة.
وبموجب نموذج أمان OpenClaw، فإن مضيف Gateway ودليل حالة OpenClaw المحلي موجودان بالفعل داخل حدود المشغل الموثوق، لذلك فهذه في الأساس مسألة متانة تشغيلية وليست حد ثقة بعيدًا مستقلًا.

تحسين مخطط له:

- إضافة دعم SecretRef لمواد مفاتيح Matrix الدائمة بحيث يمكن جلب مفاتيح الاسترداد والأسرار ذات الصلة بتشفير المخزن من موفري أسرار OpenClaw بدلًا من الملفات المحلية فقط

## إدارة الملف الشخصي

حدّث الملف الشخصي الذاتي لـ Matrix للحساب المحدد باستخدام:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

أضف `--account <id>` عندما تريد استهداف حساب Matrix مسمى صراحة.

تقبل Matrix عناوين URL للصور الرمزية بصيغة `mxc://` مباشرة. وعندما تمرر عنوان URL للصورة الرمزية من نوع `http://` أو `https://`، يقوم OpenClaw أولًا برفعه إلى Matrix ثم يخزن عنوان `mxc://` المحلول مرة أخرى في `channels.matrix.avatarUrl` (أو تجاوز الحساب المحدد).

## إشعارات التحقق التلقائية

تنشر Matrix الآن إشعارات دورة حياة التحقق مباشرة في غرفة التحقق الصارمة للرسائل المباشرة كرسائل `m.notice`.
ويشمل ذلك:

- إشعارات طلب التحقق
- إشعارات جاهزية التحقق (مع إرشادات صريحة "تحقق عبر emoji")
- إشعارات بدء التحقق واكتماله
- تفاصيل SAS (emoji والأرقام العشرية) عند توفرها

تُتبع طلبات التحقق الواردة من عميل Matrix آخر وتُقبل تلقائيًا بواسطة OpenClaw.
وبالنسبة إلى تدفقات التحقق الذاتي، يبدأ OpenClaw أيضًا تدفق SAS تلقائيًا عندما يصبح تحقق emoji متاحًا ويؤكد جانبه هو.
أما طلبات التحقق من مستخدم/جهاز Matrix آخر، فيقبل OpenClaw الطلب تلقائيًا ثم ينتظر استمرار تدفق SAS بشكل طبيعي.
ولا يزال عليك مقارنة SAS المكوّن من emoji أو الأرقام العشرية في عميل Matrix لديك وتأكيد "إنها متطابقة" هناك لإكمال التحقق.

لا يقبل OpenClaw تلقائيًا التدفقات المكررة التي بدأها بنفسه بشكل أعمى. ويتخطى بدء التشغيل إنشاء طلب جديد عندما يكون طلب التحقق الذاتي معلقًا بالفعل.

لا يتم تمرير إشعارات بروتوكول/نظام التحقق إلى مسار دردشة الوكيل، لذلك لا تنتج `NO_REPLY`.

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

إذا خرجت حالة الرسائل المباشرة عن المزامنة، فقد ينتهي الأمر بـ OpenClaw مع روابط `m.direct` قديمة تشير إلى غرف فردية قديمة بدلًا من الرسالة المباشرة الحية. افحص الرابط الحالي لنظير باستخدام:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

وأصلحه باستخدام:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

يبقي تدفق الإصلاح منطق Matrix الخاص داخل الإضافة:

- يفضل رسالة مباشرة 1:1 صارمة تم ربطها بالفعل في `m.direct`
- وإلا يعود إلى أي رسالة مباشرة 1:1 صارمة منضم إليها حاليًا مع ذلك المستخدم
- وإذا لم توجد رسالة مباشرة سليمة، فإنه ينشئ غرفة مباشرة جديدة ويعيد كتابة `m.direct` للإشارة إليها

لا يحذف تدفق الإصلاح الغرف القديمة تلقائيًا. إنه يختار فقط الرسالة المباشرة السليمة ويحدث الربط بحيث تستهدف عمليات إرسال Matrix الجديدة، وإشعارات التحقق، وتدفقات الرسائل المباشرة الأخرى الغرفة الصحيحة مرة أخرى.

## سلاسل الرسائل

تدعم Matrix سلاسل Matrix الأصلية لكل من الردود التلقائية ورسائل أدوات الإرسال.

- يبقي `threadReplies: "off"` الردود في المستوى الأعلى ويبقي الرسائل الواردة المترابطة على جلسة الأصل.
- يرد `threadReplies: "inbound"` داخل سلسلة فقط عندما تكون الرسالة الواردة بالفعل في تلك السلسلة.
- يبقي `threadReplies: "always"` ردود الغرفة داخل سلسلة متجذرة في الرسالة المشغلة ويوجه تلك المحادثة عبر الجلسة المطابقة ذات النطاق الخاص بالسلسلة من أول رسالة مشغلة.
- تتجاوز `dm.threadReplies` الإعداد الأعلى مستوى للرسائل المباشرة فقط. على سبيل المثال، يمكنك إبقاء سلاسل الغرف معزولة مع إبقاء الرسائل المباشرة مسطحة.
- تتضمن الرسائل الواردة ضمن السلاسل رسالة جذر السلسلة كسياق إضافي للوكيل.
- ترث رسائل أدوات الإرسال الآن سلسلة Matrix الحالية تلقائيًا عندما يكون الهدف هو الغرفة نفسها، أو هدف مستخدم الرسائل المباشرة نفسه، ما لم يتم توفير `threadId` صراحة.
- الروابط التشغيلية لسلاسل الرسائل مدعومة في Matrix. تعمل الآن `/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` و`/acp spawn` المرتبطة بالسلسلة في غرف Matrix والرسائل المباشرة.
- ينشئ `/focus` في غرفة/رسالة مباشرة Matrix من المستوى الأعلى سلسلة Matrix جديدة ويربطها بالجلسة الهدف عندما تكون `threadBindings.spawnSubagentSessions=true`.
- يؤدي تشغيل `/focus` أو `/acp spawn --thread here` داخل سلسلة Matrix موجودة إلى ربط تلك السلسلة الحالية بدلًا من ذلك.

## روابط محادثات ACP

يمكن تحويل غرف Matrix والرسائل المباشرة وسلاسل Matrix الموجودة إلى مساحات عمل ACP دائمة دون تغيير واجهة الدردشة.

تدفق مشغل سريع:

- شغّل `/acp spawn codex --bind here` داخل رسالة Matrix المباشرة، أو الغرفة، أو السلسلة الموجودة التي تريد الاستمرار في استخدامها.
- في رسالة Matrix مباشرة أو غرفة من المستوى الأعلى، تظل الرسالة المباشرة/الغرفة الحالية هي واجهة الدردشة وتوجَّه الرسائل المستقبلية إلى جلسة ACP التي تم إنشاؤها.
- داخل سلسلة Matrix موجودة، يربط `--bind here` تلك السلسلة الحالية في مكانها.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الربط.

ملاحظات:

- لا ينشئ `--bind here` سلسلة Matrix فرعية.
- لا تكون `threadBindings.spawnAcpSessions` مطلوبة إلا لـ `/acp spawn --thread auto|here`، حيث يحتاج OpenClaw إلى إنشاء سلسلة Matrix فرعية أو ربطها.

### تهيئة ربط السلاسل

ترث Matrix القيم الافتراضية العامة من `session.threadBindings`، كما تدعم تجاوزات لكل قناة:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

علامات إنشاء العمليات المرتبطة بالسلاسل في Matrix اختيارية:

- اضبط `threadBindings.spawnSubagentSessions: true` للسماح للأمر `/focus` في المستوى الأعلى بإنشاء سلاسل Matrix جديدة وربطها.
- اضبط `threadBindings.spawnAcpSessions: true` للسماح للأمر `/acp spawn --thread auto|here` بربط جلسات ACP بسلاسل Matrix.

## التفاعلات

تدعم Matrix إجراءات التفاعل الصادرة، وإشعارات التفاعل الواردة، وتفاعلات الإقرار الواردة.

- تخضع أدوات التفاعل الصادر للبوابة `channels["matrix"].actions.reactions`.
- يضيف `react` تفاعلًا إلى حدث Matrix محدد.
- تسرد `reactions` ملخص التفاعلات الحالي لحدث Matrix محدد.
- تؤدي `emoji=""` إلى إزالة تفاعلات حساب البوت نفسه على ذلك الحدث.
- تؤدي `remove: true` إلى إزالة تفاعل emoji المحدد فقط من حساب البوت.

يستخدم نطاق تفاعلات الإقرار ترتيب حل OpenClaw القياسي:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- الرجوع الاحتياطي إلى emoji الخاص بهوية الوكيل

يُحل نطاق تفاعلات الإقرار بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

يُحل وضع إشعارات التفاعل بهذا الترتيب:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- الافتراضي: `own`

السلوك الحالي:

- يؤدي `reactionNotifications: "own"` إلى تمرير أحداث `m.reaction` المضافة عندما تستهدف رسائل Matrix التي أنشأها البوت.
- يؤدي `reactionNotifications: "off"` إلى تعطيل أحداث نظام التفاعل.
- لا تزال إزالة التفاعلات غير تُركّب كأحداث نظام لأن Matrix تعرضها كتنقيحات، وليس كعمليات إزالة `m.reaction` مستقلة.

## سياق السجل

- يتحكم `channels.matrix.historyLimit` في عدد رسائل الغرف الحديثة التي تُضمَّن كـ `InboundHistory` عندما تشغّل رسالة غرفة Matrix الوكيل.
- ويرجع احتياطيًا إلى `messages.groupChat.historyLimit`. اضبطه على `0` للتعطيل.
- يكون سجل غرف Matrix خاصًا بالغرفة فقط. وتستمر الرسائل المباشرة باستخدام سجل الجلسة العادي.
- سجل غرف Matrix يكون للمعلّق فقط: يخزّن OpenClaw رسائل الغرفة التي لم تشغّل ردًا بعد، ثم يلتقط هذه النافذة عندما تصل إشارة أو مشغّل آخر.
- لا تُضمَّن رسالة المشغّل الحالية في `InboundHistory`؛ بل تبقى في نص الإدخال الرئيسي لتلك الدورة.
- تعيد محاولات الحدث Matrix نفسه استخدام لقطة السجل الأصلية بدلًا من الانجراف إلى رسائل غرفة أحدث.

## ظهور السياق

تدعم Matrix عنصر التحكم المشترك `contextVisibility` للسياق الإضافي للغرفة مثل نص الرد المجتلب، وجذور السلاسل، والسجل المعلّق.

- `contextVisibility: "all"` هي القيمة الافتراضية. ويتم الاحتفاظ بالسياق الإضافي كما ورد.
- يقوم `contextVisibility: "allowlist"` بتصفية السياق الإضافي إلى المرسلين المسموح لهم بواسطة فحوصات قائمة سماح الغرفة/المستخدم النشطة.
- يتصرف `contextVisibility: "allowlist_quote"` مثل `allowlist`، لكنه يحتفظ مع ذلك برد مقتبس صريح واحد.

يؤثر هذا الإعداد في ظهور السياق الإضافي، وليس في ما إذا كانت الرسالة الواردة نفسها يمكن أن تشغّل ردًا.
ولا يزال تفويض التشغيل يأتي من إعدادات `groupPolicy` و`groups` و`groupAllowFrom` وسياسات الرسائل المباشرة.

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

راجع [Groups](/channels/groups) لمعرفة سلوك بوابة الإشارات وقائمة السماح.

مثال اقتران لرسائل Matrix المباشرة:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

إذا استمر مستخدم Matrix غير معتمد في مراسلتك قبل الموافقة، يعيد OpenClaw استخدام رمز الاقتران المعلق نفسه وقد يرسل رد تذكير مرة أخرى بعد فترة تهدئة قصيرة بدلًا من إنشاء رمز جديد.

راجع [Pairing](/channels/pairing) للتدفق المشترك لاقتران الرسائل المباشرة وتخطيط التخزين.

## موافقات exec

يمكن لـ Matrix أن تعمل كعميل موافقات exec لحساب Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (اختياري؛ يرجع احتياطيًا إلى `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

يجب أن يكون الموافقون معرّفات مستخدمي Matrix مثل `@owner:example.org`. تقوم Matrix بتمكين موافقات exec الأصلية تلقائيًا عندما تكون `enabled` غير مضبوطة أو `"auto"` وعندما يمكن حل موافق واحد على الأقل، إما من `execApprovals.approvers` أو من `channels.matrix.dm.allowFrom`. اضبط `enabled: false` لتعطيل Matrix كعميل موافقات أصلي صراحة. وبخلاف ذلك، تعود طلبات الموافقة احتياطيًا إلى مسارات الموافقة المهيأة الأخرى أو إلى سياسة الرجوع الاحتياطي لموافقات exec.

التوجيه الأصلي في Matrix يقتصر اليوم على exec فقط:

- تتحكم `channels.matrix.execApprovals.*` في التوجيه الأصلي للرسائل المباشرة/القنوات لموافقات exec فقط.
- لا تزال موافقات الإضافات تستخدم `/approve` المشترك في الدردشة نفسها بالإضافة إلى أي إعادة توجيه مهيأة في `approvals.plugin`.
- لا يزال بإمكان Matrix إعادة استخدام `channels.matrix.dm.allowFrom` لتفويض موافقات الإضافات عندما تتمكن من استنتاج الموافقين بأمان، لكنها لا تعرض مسار توزيع أصلي منفصل لموافقات الإضافات عبر الرسائل المباشرة/القنوات.

قواعد التسليم:

- يؤدي `target: "dm"` إلى إرسال مطالبات الموافقة إلى الرسائل المباشرة للموافقين
- يؤدي `target: "channel"` إلى إعادة إرسال المطالبة إلى غرفة أو رسالة Matrix المباشرة الأصلية
- يؤدي `target: "both"` إلى الإرسال إلى الرسائل المباشرة للموافقين وإلى غرفة أو رسالة Matrix المباشرة الأصلية

تستخدم Matrix اليوم مطالبات موافقة نصية. ويعالجها الموافقون باستخدام `/approve <id> allow-once` أو `/approve <id> allow-always` أو `/approve <id> deny`.

يمكن فقط للموافقين الذين تم حلهم الموافقة أو الرفض. ويتضمن التسليم عبر القنوات نص الأمر، لذلك لا تمكّن `channel` أو `both` إلا في الغرف الموثوقة.

تعيد مطالبات موافقة Matrix استخدام مخطط الموافقات الأساسي المشترك. أما الواجهة الأصلية الخاصة بـ Matrix فهي مجرد وسيلة نقل لموافقات exec: توجيه الغرف/الرسائل المباشرة وسلوك إرسال/تحديث/حذف الرسائل.

تجاوز لكل حساب:

- `channels.matrix.accounts.<account>.execApprovals`

مستندات ذات صلة: [Exec approvals](/tools/exec-approvals)

## مثال على تعدد الحسابات

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

تعمل القيم في `channels.matrix` من المستوى الأعلى كقيم افتراضية للحسابات المسماة ما لم يتجاوزها حساب ما.
يمكنك تقييد إدخالات الغرف الموروثة إلى حساب Matrix واحد باستخدام `groups.<room>.account` (أو `rooms.<room>.account` القديم).
وتظل الإدخالات التي لا تحتوي على `account` مشتركة بين جميع حسابات Matrix، كما أن الإدخالات التي تحتوي على `account: "default"` تظل تعمل عندما يكون الحساب الافتراضي مهيأ مباشرة على `channels.matrix.*` من المستوى الأعلى.
ولا تنشئ القيم الافتراضية المشتركة الجزئية للمصادقة حسابًا افتراضيًا ضمنيًا منفصلًا بمفردها. لا يقوم OpenClaw بتوليف الحساب `default` في المستوى الأعلى إلا عندما يمتلك ذلك الافتراضي مصادقة حديثة (`homeserver` مع `accessToken`، أو `homeserver` مع `userId` و`password`)؛ ويمكن للحسابات المسماة أن تظل قابلة للاكتشاف من `homeserver` مع `userId` عندما تلبّي بيانات الاعتماد المخزنة مؤقتًا المصادقة لاحقًا.
إذا كانت Matrix تحتوي بالفعل على حساب مسمى واحد بالضبط، أو كانت `defaultAccount` تشير إلى مفتاح حساب مسمى موجود، فإن ترقية الإصلاح/الإعداد من حساب واحد إلى عدة حسابات تحافظ على ذلك الحساب بدلًا من إنشاء إدخال جديد `accounts.default`. ولا تنتقل إلى ذلك الحساب المُرقّى إلا مفاتيح Matrix الخاصة بالمصادقة/bootstrap؛ أما مفاتيح سياسات التسليم المشتركة فتبقى في المستوى الأعلى.
اضبط `defaultAccount` عندما تريد أن يفضّل OpenClaw حساب Matrix مسمى واحدًا للتوجيه الضمني، والفحص، وعمليات CLI.
إذا هيأت عدة حسابات مسماة، فاضبط `defaultAccount` أو مرر `--account <id>` لأوامر CLI التي تعتمد على اختيار الحساب الضمني.
مرر `--account <id>` إلى `openclaw matrix verify ...` و`openclaw matrix devices ...` عندما تريد تجاوز هذا الاختيار الضمني لأمر واحد.

## الخوادم المنزلية الخاصة/الشبكات المحلية

افتراضيًا، يمنع OpenClaw خوادم Matrix المنزلية الخاصة/الداخلية للحماية من SSRF ما لم
توافق صراحة لكل حساب.

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

مثال إعداد عبر CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

تسمح هذه الموافقة فقط بالأهداف الخاصة/الداخلية الموثوقة. أما الخوادم المنزلية العامة غير المشفرة مثل
`http://matrix.example.org:8008` فتظل محظورة. ويفضل استخدام `https://` كلما أمكن.

## تمرير حركة Matrix عبر وكيل

إذا كان نشر Matrix لديك يحتاج إلى وكيل HTTP(S) صادر صريح، فاضبط `channels.matrix.proxy`:

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

يمكن للحسابات المسماة تجاوز الافتراضي في المستوى الأعلى باستخدام `channels.matrix.accounts.<id>.proxy`.
ويستخدم OpenClaw إعداد الوكيل نفسه لحركة Matrix في وقت التشغيل ولفحوصات حالة الحساب.

## حل الأهداف

تقبل Matrix صيغ الأهداف التالية أينما طلب منك OpenClaw هدف غرفة أو مستخدم:

- المستخدمون: `@user:server` أو `user:@user:server` أو `matrix:user:@user:server`
- الغرف: `!room:server` أو `room:!room:server` أو `matrix:room:!room:server`
- الأسماء المستعارة: `#alias:server` أو `channel:#alias:server` أو `matrix:channel:#alias:server`

يستخدم البحث الحي في الدليل حساب Matrix المسجل دخوله:

- تستعلم عمليات البحث عن المستخدمين دليل مستخدمي Matrix على ذلك الخادم المنزلي.
- تقبل عمليات البحث عن الغرف معرّفات الغرف والأسماء المستعارة الصريحة مباشرة، ثم تعود احتياطيًا إلى البحث في أسماء الغرف المنضم إليها لذلك الحساب.
- يكون البحث في أسماء الغرف المنضم إليها بأفضل جهد. وإذا تعذر حل اسم الغرفة إلى معرّف أو اسم مستعار، فسيتم تجاهله بواسطة حل قائمة السماح في وقت التشغيل.

## مرجع التهيئة

- `enabled`: تمكين القناة أو تعطيلها.
- `name`: تسمية اختيارية للحساب.
- `defaultAccount`: معرّف الحساب المفضل عند تهيئة عدة حسابات Matrix.
- `homeserver`: عنوان URL للخادم المنزلي، مثل `https://matrix.example.org`.
- `allowPrivateNetwork`: السماح لهذا الحساب في Matrix بالاتصال بخوادم منزلية خاصة/داخلية. فعّل هذا عندما يُحل الخادم المنزلي إلى `localhost` أو عنوان IP ضمن LAN/Tailscale أو مضيف داخلي مثل `matrix-synapse`.
- `proxy`: عنوان URL اختياري لوكيل HTTP(S) لحركة Matrix. ويمكن للحسابات المسماة تجاوز الافتراضي في المستوى الأعلى عبر `proxy` خاص بها.
- `userId`: معرّف مستخدم Matrix كامل، مثل `@bot:example.org`.
- `accessToken`: access token للمصادقة القائمة على الرمز المميز. تُدعم القيم النصية الصريحة وقيم SecretRef لكل من `channels.matrix.accessToken` و`channels.matrix.accounts.<id>.accessToken` عبر موفري env/file/exec. راجع [Secrets Management](/gateway/secrets).
- `password`: كلمة المرور لتسجيل الدخول القائم على كلمة المرور. تُدعم القيم النصية الصريحة وقيم SecretRef.
- `deviceId`: معرّف جهاز Matrix صريح.
- `deviceName`: اسم عرض الجهاز لتسجيل الدخول بكلمة المرور.
- `avatarUrl`: عنوان URL للصورة الرمزية الذاتية المخزن لمزامنة الملف الشخصي وتحديثات `set-profile`.
- `initialSyncLimit`: حد أحداث المزامنة عند بدء التشغيل.
- `encryption`: تمكين E2EE.
- `allowlistOnly`: فرض سلوك قائمة السماح فقط على الرسائل المباشرة والغرف.
- `allowBots`: السماح بالرسائل من حسابات Matrix OpenClaw أخرى مهيأة (`true` أو `"mentions"`).
- `groupPolicy`: `open` أو `allowlist` أو `disabled`.
- `contextVisibility`: وضع ظهور السياق الإضافي للغرفة (`all` أو `allowlist` أو `allowlist_quote`).
- `groupAllowFrom`: قائمة سماح لمعرّفات المستخدمين لحركة الغرف.
- يجب أن تكون إدخالات `groupAllowFrom` معرّفات مستخدم Matrix كاملة. ويتم تجاهل الأسماء غير المحلولة في وقت التشغيل.
- `historyLimit`: الحد الأقصى لرسائل الغرف المضمّنة كسياق سجل للمجموعات. يرجع احتياطيًا إلى `messages.groupChat.historyLimit`. اضبط `0` للتعطيل.
- `replyToMode`: `off` أو `first` أو `all`.
- `markdown`: تهيئة Markdown اختيارية لتصيير نص Matrix الصادر.
- `streaming`: `off` (الافتراضي) أو `partial` أو `true` أو `false`. تؤدي `partial` و`true` إلى تمكين معاينات مسودة أحادية الرسالة مع تحديثات تعديل في المكان.
- `blockStreaming`: يؤدي `true` إلى تمكين رسائل تقدم منفصلة للكتل المكتملة للمساعد أثناء تفعيل بث معاينة المسودة.
- `threadReplies`: `off` أو `inbound` أو `always`.
- `threadBindings`: تجاوزات لكل قناة لتوجيه الجلسات المرتبطة بالسلاسل ودورة حياتها.
- `startupVerification`: وضع طلب التحقق الذاتي التلقائي عند بدء التشغيل (`if-unverified` أو `off`).
- `startupVerificationCooldownHours`: فترة التهدئة قبل إعادة محاولة طلبات التحقق التلقائية عند بدء التشغيل.
- `textChunkLimit`: حجم تجزئة الرسائل الصادرة.
- `chunkMode`: `length` أو `newline`.
- `responsePrefix`: بادئة رسائل اختيارية للردود الصادرة.
- `ackReaction`: تجاوز اختياري لتفاعل الإقرار لهذه القناة/الحساب.
- `ackReactionScope`: تجاوز اختياري لنطاق تفاعل الإقرار (`group-mentions` أو `group-all` أو `direct` أو `all` أو `none` أو `off`).
- `reactionNotifications`: وضع إشعارات التفاعل الواردة (`own` أو `off`).
- `mediaMaxMb`: الحد الأقصى لحجم الوسائط بالميغابايت لمعالجة وسائط Matrix. ويُطبّق على الإرسال الصادر ومعالجة الوسائط الواردة.
- `autoJoin`: سياسة الانضمام التلقائي للدعوات (`always` أو `allowlist` أو `off`). الافتراضي: `off`.
- `autoJoinAllowlist`: الغرف/الأسماء المستعارة المسموح بها عندما تكون `autoJoin` هي `allowlist`. تُحل إدخالات الأسماء المستعارة إلى معرّفات غرف أثناء معالجة الدعوات؛ ولا يثق OpenClaw في حالة الاسم المستعار التي تدعيها الغرفة المدعو إليها.
- `dm`: كتلة سياسة الرسائل المباشرة (`enabled` أو `policy` أو `allowFrom` أو `threadReplies`).
- يجب أن تكون إدخالات `dm.allowFrom` معرّفات مستخدم Matrix كاملة ما لم تكن قد قمت بحلها بالفعل عبر البحث الحي في الدليل.
- `dm.threadReplies`: تجاوز لسياسة السلاسل في الرسائل المباشرة فقط (`off` أو `inbound` أو `always`). ويتجاوز إعداد `threadReplies` الأعلى مستوى لكل من موضع الرد وعزل الجلسة في الرسائل المباشرة.
- `execApprovals`: تسليم موافقات exec الأصلية في Matrix (`enabled` أو `approvers` أو `target` أو `agentFilter` أو `sessionFilter`).
- `execApprovals.approvers`: معرّفات مستخدمي Matrix المسموح لهم بالموافقة على طلبات exec. وهو اختياري عندما تكون `dm.allowFrom` تحدد الموافقين بالفعل.
- `execApprovals.target`: `dm | channel | both` (الافتراضي: `dm`).
- `accounts`: تجاوزات مسماة لكل حساب. وتعمل القيم العليا في `channels.matrix` كقيم افتراضية لهذه الإدخالات.
- `groups`: خريطة سياسات لكل غرفة. فضّل معرّفات الغرف أو الأسماء المستعارة؛ ويتم تجاهل أسماء الغرف غير المحلولة في وقت التشغيل. تستخدم هوية الجلسة/المجموعة معرّف الغرفة الثابت بعد الحل، بينما تظل التسميات المقروءة للبشر مأخوذة من أسماء الغرف.
- `groups.<room>.account`: يقيّد إدخال غرفة موروثًا واحدًا إلى حساب Matrix محدد في الإعدادات متعددة الحسابات.
- `groups.<room>.allowBots`: تجاوز على مستوى الغرفة للمرسلين من البوتات المهيأة (`true` أو `"mentions"`).
- `groups.<room>.users`: قائمة سماح للمرسلين لكل غرفة.
- `groups.<room>.tools`: تجاوزات السماح/المنع للأدوات لكل غرفة.
- `groups.<room>.autoReply`: تجاوز على مستوى الغرفة لبوابة الإشارات. تؤدي `true` إلى تعطيل متطلبات الإشارة لتلك الغرفة؛ وتفرض `false` إعادتها.
- `groups.<room>.skills`: مرشح Skills اختياري على مستوى الغرفة.
- `groups.<room>.systemPrompt`: مقتطف system prompt اختياري على مستوى الغرفة.
- `rooms`: اسم مستعار قديم لـ `groups`.
- `actions`: بوابات الأدوات لكل إجراء (`messages` أو `reactions` أو `pins` أو `profile` أو `memberInfo` أو `channelInfo` أو `verification`).

## ذو صلة

- [Channels Overview](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [Groups](/channels/groups) — سلوك الدردشة الجماعية وبوابة الإشارات
- [Channel Routing](/channels/channel-routing) — توجيه الجلسات للرسائل
- [Security](/gateway/security) — نموذج الوصول والتقوية
