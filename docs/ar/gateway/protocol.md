---
read_when:
    - تنفيذ أو تحديث عملاء gateway عبر WS
    - تصحيح أخطاء عدم تطابق البروتوكول أو فشل الاتصال
    - إعادة توليد schema/models الخاصة بالبروتوكول
summary: 'بروتوكول Gateway عبر WebSocket: المصافحة، والإطارات، والإصدارات'
title: بروتوكول Gateway
x-i18n:
    generated_at: "2026-04-05T12:44:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: c37f5b686562dda3ba3516ac6982ad87b2f01d8148233284e9917099c6e96d87
    source_path: gateway/protocol.md
    workflow: 15
---

# بروتوكول Gateway ‏(WebSocket)

بروتوكول Gateway WS هو **سطح التحكم الوحيد + نقل العقد** لـ
OpenClaw. فجميع العملاء (CLI وواجهة الويب وتطبيق macOS وعقد iOS/Android والعقد
عديمة الرأس) يتصلون عبر WebSocket ويعلنون **الدور** + **النطاق** الخاصين بهم وقت
المصافحة.

## النقل

- WebSocket، وإطارات نصية بحمولات JSON.
- **يجب** أن يكون الإطار الأول طلب `connect`.

## المصافحة (`connect`)

Gateway → العميل (تحدي ما قبل الاتصال):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

العميل → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → العميل:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

عند إصدار device token، تتضمن `hello-ok` أيضًا:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

أثناء تسليم bootstrap الموثوق، قد تتضمن `hello-ok.auth` أيضًا إدخالات
أدوار إضافية محدودة ضمن `deviceTokens`:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

في تدفق bootstrap المضمّن للعقدة/المشغّل، يبقى token الأساسي للعقدة
`scopes: []` وتبقى أي token للمشغّل مسلّمة ضمن الحدود
المقيدة بقائمة سماح bootstrap للمشغّل (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). وتظل فحوصات نطاق bootstrap
مرتبطة ببادئة الدور: إذ لا تلبّي إدخالات المشغّل إلا طلبات المشغّل، وما تزال
الأدوار غير المشغّلة تحتاج إلى نطاقات تحت بادئة دورها الخاصة.

### مثال عقدة

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## التأطير

- **طلب**: `{type:"req", id, method, params}`
- **استجابة**: `{type:"res", id, ok, payload|error}`
- **حدث**: `{type:"event", event, payload, seq?, stateVersion?}`

تتطلب الطرق ذات الآثار الجانبية **مفاتيح idempotency** ‏(راجع schema).

## الأدوار + النطاقات

### الأدوار

- `operator` = عميل سطح التحكم (CLI/UI/الأتمتة).
- `node` = مضيف الإمكانات (`camera/screen/canvas/system.run`).

### النطاقات (`operator`)

النطاقات الشائعة:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

يتطلب `talk.config` مع `includeSecrets: true` النطاق `operator.talk.secrets`
(أو `operator.admin`).

قد تطلب طرق Gateway RPC المسجلة من المكونات الإضافية نطاق operator خاصًا بها، لكن
بادئات الإدارة الأساسية المحجوزة (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) تُحل دائمًا إلى `operator.admin`.

يمثل نطاق الطريقة البوابة الأولى فقط. إذ تطبق بعض الأوامر المائلة التي تصل عبر
`chat.send` فحوصات أشد على مستوى الأمر فوق ذلك. فعلى سبيل المثال، تتطلب
عمليات الكتابة الدائمة عبر `/config set` و`/config unset` النطاق `operator.admin`.

كما يملك `node.pair.approve` فحص نطاق إضافيًا وقت الموافقة فوق
نطاق الطريقة الأساسي:

- الطلبات من دون أوامر: `operator.pairing`
- الطلبات التي تحتوي على أوامر عقدة غير `exec`: ‏`operator.pairing` + `operator.write`
- الطلبات التي تتضمن `system.run`, `system.run.prepare`, أو `system.which`:
  ‏`operator.pairing` + `operator.admin`

### `caps`/`commands`/`permissions` ‏(`node`)

تعلن العقد مطالبات الإمكانات وقت الاتصال:

- `caps`: فئات إمكانات عالية المستوى.
- `commands`: قائمة سماح الأوامر للاستدعاء.
- `permissions`: مفاتيح تبديل دقيقة (مثل `screen.record`, `camera.capture`).

يتعامل Gateway مع هذه القيم على أنها **مطالبات** ويفرض قوائم سماح من جانب الخادم.

## الحضور

- تعيد `system-presence` إدخالات مفاتيحها حسب هوية الجهاز.
- تتضمن إدخالات الحضور `deviceId`, `roles`, و`scopes` لكي تتمكن واجهات المستخدم من عرض صف واحد لكل جهاز
  حتى عندما يتصل بصفته **operator** و**node** معًا.

## عائلات طرق RPC الشائعة

ليست هذه الصفحة تفريغًا مولّدًا كاملًا، لكن السطح العام لـ WS أوسع
من أمثلة المصافحة/المصادقة أعلاه. وهذه هي عائلات الطرق الرئيسية التي يعرضها
Gateway اليوم.

`hello-ok.features.methods` هي قائمة اكتشاف متحفظة مبنية من
`src/gateway/server-methods-list.ts` بالإضافة إلى الطرق المصدّرة من المكونات الإضافية/القنوات المحمّلة.
تعامل معها على أنها اكتشاف للميزات، لا على أنها تفريغ مولّد لكل المساعدات القابلة للاستدعاء
المطبقة في `src/gateway/server-methods/*.ts`.

### النظام والهوية

- تعيد `health` لقطة السلامة المخزنة مؤقتًا أو المفحوصة حديثًا للـ gateway.
- تعيد `status` ملخص gateway بنمط `/status`؛ وتُضمّن الحقول الحساسة
  فقط لعملاء operator ذوي النطاق الإداري.
- تعيد `gateway.identity.get` هوية جهاز gateway المستخدمة في relay
  وتدفقات الاقتران.
- تعيد `system-presence` لقطة الحضور الحالية للأجهزة المتصلة
  operator/node.
- يضيف `system-event` حدث نظام ويمكنه تحديث/بث
  سياق الحضور.
- تعيد `last-heartbeat` أحدث حدث heartbeat محفوظ.
- يبدّل `set-heartbeats` معالجة heartbeat على الـ gateway.

### النماذج والاستخدام

- تعيد `models.list` فهرس النماذج المسموح بها وقت التشغيل.
- تعيد `usage.status` ملخصات نوافذ استخدام المزوّد/الحصة المتبقية.
- تعيد `usage.cost` ملخصات مجمعة لاستخدام التكلفة لنطاق زمني.
- تعيد `doctor.memory.status` جاهزية الذاكرة المتجهية / embedding لمساحة عمل
  الوكيل الافتراضي النشطة.
- تعيد `sessions.usage` ملخصات استخدام لكل جلسة.
- تعيد `sessions.usage.timeseries` سلسلة زمنية للاستخدام لجلسة واحدة.
- تعيد `sessions.usage.logs` إدخالات سجل الاستخدام لجلسة واحدة.

### القنوات ومساعدات تسجيل الدخول

- تعيد `channels.status` ملخصات حالة القنوات/المكونات الإضافية
  المضمّنة والمجمّعة.
- تسجّل `channels.logout` الخروج من قناة/حساب محدد حيث تدعم القناة
  تسجيل الخروج.
- تبدأ `web.login.start` تدفق تسجيل دخول QR/ويب للقناة الحالية
  القابلة لتسجيل دخول QR على الويب.
- تنتظر `web.login.wait` اكتمال تدفق تسجيل دخول QR/الويب ذاك وتبدأ
  القناة عند النجاح.
- ترسل `push.test` دفعة APNs اختبارية إلى عقدة iOS مسجّلة.
- تعيد `voicewake.get` محفزات كلمة التنبيه المخزنة.
- تحدّث `voicewake.set` محفزات كلمة التنبيه وتبث التغيير.

### المراسلة والسجلات

- `send` هو RPC التسليم الصادر المباشر للإرسال الموجّه إلى قناة/حساب/سلسلة
  خارج مشغّل الدردشة.
- تعيد `logs.tail` ذيل سجل ملف gateway المكوَّن مع أدوات
  المؤشر/الحدود والتحكم في أقصى عدد من البايتات.

### Talk وTTS

- تعيد `talk.config` حمولة إعداد Talk الفعالة؛ ويتطلب `includeSecrets`
  النطاق `operator.talk.secrets` ‏(أو `operator.admin`).
- تضبط `talk.mode`/تبث حالة وضع Talk الحالية لعملاء WebChat/Control UI.
- تولد `talk.speak` كلامًا من خلال مزود الكلام النشط في Talk.
- تعيد `tts.status` حالة تفعيل TTS، والمزوّد النشط، ومزوّدي fallback،
  وحالة إعداد المزوّد.
- تعيد `tts.providers` جرد مزوّدي TTS المرئيين.
- تبدّل `tts.enable` و`tts.disable` حالة تفضيلات TTS.
- تحدّث `tts.setProvider` المزوّد المفضّل لـ TTS.
- تشغّل `tts.convert` تحويل text-to-speech لمرة واحدة.

### الأسرار، والإعداد، والتحديث، والمعالج

- تعيد `secrets.reload` حل SecretRefs النشطة وتبدّل حالة الأسرار وقت التشغيل
  فقط عند النجاح الكامل.
- تحل `secrets.resolve` إسنادات الأسرار المستهدفة بالأوامر لمجموعة
  أوامر/أهداف محددة.
- تعيد `config.get` لقطة الإعداد الحالية وhash الخاص بها.
- تكتب `config.set` حمولة إعداد مُتحققًا من صحتها.
- تدمج `config.patch` تحديثًا جزئيًا للإعداد.
- تتحقق `config.apply` + تستبدل حمولة الإعداد الكاملة.
- تعيد `config.schema` حمولة schema الحية للإعداد المستخدمة بواسطة Control UI و
  أدوات CLI: ‏schema، و`uiHints`، والإصدار، وبيانات التوليد الوصفية، بما في ذلك
  بيانات schema الوصفية للمكونات الإضافية + القنوات عندما يستطيع وقت التشغيل تحميلها. وتتضمن schema
  بيانات `title` / `description` الوصفية للحقول المشتقة من التسميات نفسها
  ونصوص المساعدة المستخدمة في UI، بما في ذلك فروع
  التركيب الخاصة بالكائنات المتداخلة، والرمز الشامل، وعناصر المصفوفات،
  و`anyOf` / `oneOf` / `allOf` عندما توجد وثائق حقول مطابقة.
- تعيد `config.schema.lookup` حمولة lookup محددة بالمسار لمسار إعداد
  واحد: المسار المطبع، وعقدة schema سطحية، وhint مطابق + `hintPath`، و
  ملخصات الأبناء المباشرين للتعمق في UI/CLI.
  - تحتفظ عقد schema الخاصة بالlookup بالوثائق الموجهة للمستخدم
    وحقول التحقق الشائعة:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    وحدود الأعداد/السلاسل/المصفوفات/الكائنات، والأعلام المنطقية مثل
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - تعرض ملخصات الأبناء `key`, و`path` المطبع، و`type`, و`required`,
    و`hasChildren`، بالإضافة إلى `hint` / `hintPath` المطابقين.
- تشغّل `update.run` تدفق تحديث gateway وتجدول إعادة التشغيل فقط عندما
  ينجح التحديث نفسه.
- تعرض `wizard.start`, `wizard.next`, `wizard.status`, و`wizard.cancel`
  معالج الإعداد الأولي عبر WS RPC.

### العائلات الرئيسية الموجودة

#### مساعدات الوكيل ومساحة العمل

- تعيد `agents.list` إدخالات الوكلاء المكوّنة.
- تدير `agents.create`, `agents.update`, و`agents.delete` سجلات الوكلاء
  وربط مساحة العمل.
- تدير `agents.files.list`, `agents.files.get`, و`agents.files.set` ملفات
  bootstrap الخاصة بمساحة العمل المعروضة لوكيل ما.
- تعيد `agent.identity.get` هوية المساعد الفعالة لوكيل أو
  جلسة.
- تنتظر `agent.wait` انتهاء تشغيل ما وتعيد اللقطة النهائية عند
  توفرها.

#### التحكم في الجلسة

- تعيد `sessions.list` فهرس الجلسات الحالي.
- تبدّل `sessions.subscribe` و`sessions.unsubscribe` اشتراكات
  تغييرات الجلسات لعميل WS الحالي.
- تبدّل `sessions.messages.subscribe` و`sessions.messages.unsubscribe`
  اشتراكات أحداث النصوص/الرسائل لجلسة واحدة.
- تعيد `sessions.preview` معاينات نصوص محدودة لمفاتيح جلسات محددة.
- تحل `sessions.resolve` أو تحول إلى الصيغة القانونية هدف جلسة.
- تنشئ `sessions.create` إدخال جلسة جديدًا.
- ترسل `sessions.send` رسالة إلى جلسة موجودة.
- `sessions.steer` هو متغير المقاطعة والتوجيه لجلسة نشطة.
- توقف `sessions.abort` العمل النشط لجلسة.
- تحدّث `sessions.patch` بيانات الجلسة الوصفية/التجاوزات.
- تنفذ `sessions.reset`, `sessions.delete`, و`sessions.compact` صيانة
  الجلسات.
- تعيد `sessions.get` صف الجلسة المخزن كاملًا.
- ما يزال تنفيذ الدردشة يستخدم `chat.history`, `chat.send`, `chat.abort`, و
  `chat.inject`.
- يتم تطبيع `chat.history` للعرض لعملاء UI: حيث تُزال وسوم التعليمات المضمنة من النص المرئي،
  وتُزال حمولات XML النصية العادية لاستدعاءات الأدوات (بما في ذلك
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>`, و
  كتل استدعاءات الأدوات المقتطعة)، كما تُزال رموز التحكم بالنموذج المسرّبة بصيغة ASCII/العرض الكامل،
  وتُحذف صفوف المساعد الصامتة البحتة مثل `NO_REPLY` /
  `no_reply` المطابقة تمامًا، ويمكن استبدال الصفوف كبيرة الحجم بعناصر نائبة.

#### اقتران الأجهزة وdevice tokens

- تعيد `device.pair.list` الأجهزة المقترنة المعلقة والموافق عليها.
- تدير `device.pair.approve`, `device.pair.reject`, و`device.pair.remove`
  سجلات اقتران الأجهزة.
- تدير `device.token.rotate` تدوير token جهاز مقترن ضمن حدود دوره
  ونطاقاته المعتمدة.
- تلغي `device.token.revoke` device token.

#### اقتران العقد، والاستدعاء، والعمل المعلّق

- تغطي `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject`, و`node.pair.verify` اقتران العقد والتحقق من
  bootstrap.
- تعيد `node.list` و`node.describe` حالة العقد المعروفة/المتصلة.
- تحدّث `node.rename` تسمية عقدة مقترنة.
- تمرر `node.invoke` أمرًا إلى عقدة متصلة.
- تعيد `node.invoke.result` النتيجة لطلب invoke.
- تحمل `node.event` الأحداث الصادرة من العقدة عودةً إلى الـ gateway.
- تحدّث `node.canvas.capability.refresh` رموز canvas-capability المحددة النطاق.
- تمثل `node.pending.pull` و`node.pending.ack` واجهات queue الخاصة بالعقد المتصلة.
- تدير `node.pending.enqueue` و`node.pending.drain` العمل المعلّق المتين
  للعقد غير المتصلة/المنفصلة.

#### عائلات الموافقات

- تغطي `exec.approval.request` و`exec.approval.resolve` طلبات
  موافقة exec لمرة واحدة.
- تنتظر `exec.approval.waitDecision` قرار exec pending واحدًا وتعيد
  القرار النهائي (أو `null` عند انتهاء المهلة).
- تدير `exec.approvals.get` و`exec.approvals.set` لقطات سياسة موافقات exec الخاصة بالـ gateway.
- تدير `exec.approvals.node.get` و`exec.approvals.node.set` سياسة موافقات exec المحلية للعقدة
  عبر أوامر relay الخاصة بالعقدة.
- تغطي `plugin.approval.request`, `plugin.approval.waitDecision`, و
  `plugin.approval.resolve` تدفقات الموافقات المعرّفة من المكونات الإضافية.

#### عائلات رئيسية أخرى

- الأتمتة:
  - تجدول `wake` حقن نص تنبيه فوري أو في heartbeat التالية
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- المهارات/الأدوات: `skills.*`, `tools.catalog`, `tools.effective`

### عائلات الأحداث الشائعة

- `chat`: تحديثات دردشة UI مثل `chat.inject` وغيرها من
  أحداث الدردشة الخاصة بالنصوص فقط.
- `session.message` و`session.tool`: تحديثات النصوص/تدفق الأحداث
  لجلسة مشتركة.
- `sessions.changed`: تغيّر فهرس الجلسات أو بياناتها الوصفية.
- `presence`: تحديثات لقطة حضور النظام.
- `tick`: حدث keepalive / liveness دوري.
- `health`: تحديث لقطة سلامة gateway.
- `heartbeat`: تحديث تدفق أحداث heartbeat.
- `cron`: حدث تغيّر تشغيل/وظيفة cron.
- `shutdown`: إشعار إيقاف gateway.
- `node.pair.requested` / `node.pair.resolved`: دورة حياة اقتران العقد.
- `node.invoke.request`: بث طلب استدعاء عقدة.
- `device.pair.requested` / `device.pair.resolved`: دورة حياة الأجهزة المقترنة.
- `voicewake.changed`: تغيّر إعداد محفزات كلمة التنبيه.
- `exec.approval.requested` / `exec.approval.resolved`: دورة حياة
  موافقة exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: دورة حياة موافقة
  المكون الإضافي.

### طرق مساعدة العقد

- يمكن للعقد استدعاء `skills.bins` لجلب القائمة الحالية للملفات التنفيذية
  الخاصة بالمهارات لفحوصات allow التلقائية.

### طرق مساعدة المشغّل

- يمكن للمشغّلين استدعاء `tools.catalog` ‏(`operator.read`) لجلب فهرس الأدوات وقت التشغيل لوكيل
  ما. وتتضمن الاستجابة أدوات مجمّعة وبيانات وصفية للمصدر:
  - `source`: ‏`core` أو `plugin`
  - `pluginId`: مالك المكون الإضافي عندما يكون `source="plugin"`
  - `optional`: ما إذا كانت أداة المكون الإضافي اختيارية
- يمكن للمشغّلين استدعاء `tools.effective` ‏(`operator.read`) لجلب جرد الأدوات
  الفعّال وقت التشغيل لجلسة ما.
  - `sessionKey` مطلوب.
  - يستمد gateway سياق وقت التشغيل الموثوق من الجلسة على جانب الخادم بدل قبول
    سياق مصادقة أو تسليم يقدمه المستدعي.
  - تكون الاستجابة محددة بالجسلة وتعكس ما الذي تستطيع المحادثة النشطة استخدامه الآن،
    بما في ذلك الأدوات الأساسية وأدوات المكونات الإضافية وأدوات القنوات.
- يمكن للمشغّلين استدعاء `skills.status` ‏(`operator.read`) لجلب جرد
  المهارات المرئي لوكيل ما.
  - `agentId` اختياري؛ اتركه فارغًا لقراءة مساحة عمل الوكيل الافتراضي.
  - تتضمن الاستجابة الأهلية، والمتطلبات المفقودة، وفحوصات الإعداد، و
    خيارات التثبيت المنقحة من دون كشف القيم السرية الخام.
- يمكن للمشغّلين استدعاء `skills.search` و`skills.detail` ‏(`operator.read`) للحصول على
  بيانات اكتشاف ClawHub الوصفية.
- يمكن للمشغّلين استدعاء `skills.install` ‏(`operator.admin`) في وضعين:
  - وضع ClawHub: ‏`{ source: "clawhub", slug, version?, force? }` يثبّت
    مجلد مهارة داخل دليل `skills/` في مساحة عمل الوكيل الافتراضي.
  - وضع مثبّت Gateway: ‏`{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    يشغّل إجراء `metadata.openclaw.install` مصرّحًا به على مضيف gateway.
- يمكن للمشغّلين استدعاء `skills.update` ‏(`operator.admin`) في وضعين:
  - يحدّث وضع ClawHub slug متتبعًا واحدًا أو جميع عمليات تثبيت ClawHub المتتبعة في
    مساحة عمل الوكيل الافتراضي.
  - يقوم وضع config بترقيع قيم `skills.entries.<skillKey>` مثل `enabled`,
    `apiKey`, و`env`.

## موافقات exec

- عندما يحتاج طلب exec إلى موافقة، يبث gateway الحدث `exec.approval.requested`.
- يحل عملاء operator ذلك عبر استدعاء `exec.approval.resolve` ‏(يتطلب النطاق `operator.approvals`).
- بالنسبة إلى `host=node`، يجب أن يتضمن `exec.approval.request` القيمة `systemRunPlan` ‏(`argv`/`cwd`/`rawCommand`/بيانات الجلسة الوصفية القانونية). وتُرفض الطلبات التي تفتقد `systemRunPlan`.
- بعد الموافقة، تعيد استدعاءات `node.invoke system.run` المُمررة استخدام
  `systemRunPlan` القانوني ذاك بوصفه السياق الموثوق للأمر/cwd/الجلسة.
- إذا عدّل مستدعٍ القيم `command`, `rawCommand`, `cwd`, `agentId`, أو
  `sessionKey` بين التحضير والتمرير النهائي الموافق عليه لـ `system.run`، فإن
  gateway يرفض التشغيل بدل الوثوق بالحمولة المعدلة.

## fallback تسليم الوكيل

- يمكن لطلبات `agent` أن تتضمن `deliver=true` لطلب تسليم صادر.
- يبقي `bestEffortDeliver=false` السلوك صارمًا: إذ تعيد أهداف التسليم غير المحلولة أو الداخلية فقط الخطأ `INVALID_REQUEST`.
- يسمح `bestEffortDeliver=true` بالرجوع الاحتياطي إلى التنفيذ داخل الجلسة فقط عندما لا يمكن حل أي مسار خارجي قابل للتسليم (مثل جلسات internal/webchat أو إعدادات متعددة القنوات ملتبسة).

## الإصدارات

- يعيش `PROTOCOL_VERSION` في `src/gateway/protocol/schema.ts`.
- يرسل العملاء `minProtocol` + `maxProtocol`؛ ويرفض الخادم عدم التطابق.
- تُولَّد schemas + models من تعريفات TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## المصادقة

- تستخدم مصادقة gateway بالسر المشترك إما `connect.params.auth.token` أو
  `connect.params.auth.password`، بحسب وضع المصادقة المكوَّن.
- تلبّي الأوضاع الحاملة للهوية مثل Tailscale Serve
  (`gateway.auth.allowTailscale: true`) أو `gateway.auth.mode: "trusted-proxy"`
  غير المعتمد على loopback فحص مصادقة الاتصال من
  ترويسات الطلب بدل `connect.params.auth.*`.
- يتجاوز `gateway.auth.mode: "none"` الخاص بالدخول الخاص فحص المصادقة بالسر المشترك
  بالكامل؛ ولا تعرض هذا الوضع على دخول عام/غير موثوق.
- بعد الاقتران، يصدر Gateway **device token** محددة بالدور + النطاقات الخاصة بالاتصال.
  وتُعاد في `hello-ok.auth.deviceToken` ويجب أن يحتفظ بها العميل
  للاتصالات المستقبلية.
- يجب على العملاء الاحتفاظ بالـ `hello-ok.auth.deviceToken` الأساسية بعد أي
  اتصال ناجح.
- كما يجب عند إعادة الاتصال باستخدام **device token** هذه المخزنة
  إعادة استخدام مجموعة النطاقات المعتمدة المخزنة الخاصة بتلك الـ token. وهذا يحافظ على
  وصول read/probe/status الذي مُنح بالفعل ويتجنب انهيار إعادة الاتصال بصمت إلى
  نطاق ضمني أضيق خاص بالإدارة فقط.
- تكون أسبقية مصادقة الاتصال العادية: token/password المشتركة الصريحة أولًا، ثم
  `deviceToken` الصريحة، ثم token المخزنة لكل جهاز، ثم bootstrap token.
- إن إدخالات `hello-ok.auth.deviceTokens` الإضافية هي tokens مسلّمة ضمن bootstrap.
  ولا تحتفظ بها إلا عندما يستخدم الاتصال مصادقة bootstrap عبر نقل موثوق
  مثل `wss://` أو loopback/local pairing.
- إذا قدّم عميل **`deviceToken` صريحة** أو `scopes` صريحة، فإن
  مجموعة النطاقات المطلوبة من المستدعي تبقى هي المرجع؛ ولا يُعاد استخدام النطاقات المخبأة إلا
  عندما يعيد العميل استخدام token المخزنة لكل جهاز.
- يمكن تدوير/إلغاء device tokens عبر `device.token.rotate` و
  `device.token.revoke` ‏(يتطلب `operator.pairing`).
- يبقى إصدار/تدوير token مقيدًا بمجموعة الأدوار المعتمدة المسجلة في
  إدخال اقتران ذلك الجهاز؛ ولا يمكن لتدوير token توسيع الجهاز إلى
  دور لم تمنحه موافقة الاقتران أصلًا.
- بالنسبة إلى جلسات token الخاصة بالأجهزة المقترنة، تكون إدارة الأجهزة محددة بالذات ما لم يكن
  للمستدعي أيضًا `operator.admin`: إذ لا يستطيع غير المسؤولين إزالة/إلغاء/تدوير
  إلا إدخال الجهاز **الخاص بهم**.
- كما يتحقق `device.token.rotate` من مجموعة نطاقات operator المطلوبة مقابل
  نطاقات جلسة المستدعي الحالية. ولا يستطيع غير المسؤولين تدوير token إلى
  مجموعة نطاقات operator أوسع مما يملكونه بالفعل.
- تتضمن إخفاقات المصادقة `error.details.code` بالإضافة إلى تلميحات الاستعادة:
  - `error.details.canRetryWithDeviceToken` ‏(قيمة منطقية)
  - `error.details.recommendedNextStep` ‏(`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- سلوك العميل مع `AUTH_TOKEN_MISMATCH`:
  - يمكن للعملاء الموثوقين محاولة إعادة محاولة واحدة محدودة باستخدام token مخزنة لكل جهاز.
  - إذا فشلت إعادة المحاولة تلك، فيجب على العملاء إيقاف حلقات إعادة الاتصال التلقائية وإظهار إرشادات لاتخاذ إجراء من المشغّل.

## هوية الجهاز + الاقتران

- يجب على العقد تضمين هوية جهاز ثابتة (`device.id`) مشتقة من
  بصمة زوج مفاتيح.
- تصدر Gateways tokens لكل جهاز + دور.
- موافقات الاقتران مطلوبة لمعرّفات الأجهزة الجديدة ما لم يكن تفعيل الموافقة التلقائية
  المحلية مفعّلًا.
- تتمحور الموافقة التلقائية للاقتران حول اتصالات loopback المحلية المباشرة.
- يملك OpenClaw أيضًا مسار self-connect ضيقًا محليًا على مستوى backend/container
  لتدفقات المساعدات الموثوقة ذات الأسرار المشتركة.
- ما تزال اتصالات tailnet أو LAN على المضيف نفسه تُعامل على أنها بعيدة لأغراض الاقتران وتتطلب موافقة.
- يجب أن تتضمن جميع عملاء WS هوية `device` أثناء `connect` ‏(operator + node).
  ويمكن لـ Control UI حذفها فقط في هذه الأوضاع:
  - `gateway.controlUi.allowInsecureAuth=true` لتوافق HTTP غير الآمن على localhost فقط.
  - نجاح مصادقة operator في Control UI ضمن `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` ‏(وضع طوارئ، خفض أمني شديد).
- يجب أن توقّع جميع الاتصالات `connect.challenge` nonce التي يوفرها الخادم.

### تشخيصات ترحيل مصادقة الجهاز

بالنسبة إلى العملاء القدامى الذين ما زالوا يستخدمون سلوك التوقيع السابق للتحدي، يعيد `connect` الآن
رموز تفاصيل `DEVICE_AUTH_*` ضمن `error.details.code` مع قيمة ثابتة `error.details.reason`.

إخفاقات الترحيل الشائعة:

| الرسالة                    | details.code                     | details.reason           | المعنى                                             |
| -------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`    | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | حذف العميل `device.nonce` (أو أرسل قيمة فارغة).   |
| `device nonce mismatch`    | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | وقّع العميل باستخدام nonce قديمة/خاطئة.           |
| `device signature invalid` | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | لا تطابق حمولة التوقيع حمولة v2.                  |
| `device signature expired` | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | يقع الطابع الزمني الموقَّع خارج الانحراف المسموح. |
| `device identity mismatch` | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | لا يطابق `device.id` بصمة المفتاح العام.          |
| `device public key invalid`| `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | فشل تنسيق/تحويل المفتاح العام إلى الصيغة القانونية. |

هدف الترحيل:

- انتظر دائمًا `connect.challenge`.
- وقّع حمولة v2 التي تتضمن server nonce.
- أرسل nonce نفسها ضمن `connect.params.device.nonce`.
- حمولة التوقيع المفضلة هي `v3`، التي تربط `platform` و`deviceFamily`
  بالإضافة إلى حقول الجهاز/العميل/الدور/النطاقات/token/nonce.
- ما تزال توقيعات `v2` القديمة مقبولة للتوافق، لكن
  تثبيت البيانات الوصفية للأجهزة المقترنة ما يزال يتحكم في سياسة الأوامر عند إعادة الاتصال.

## TLS + التثبيت

- TLS مدعوم لاتصالات WS.
- يمكن للعملاء اختياريًا تثبيت بصمة شهادة gateway (راجع إعداد `gateway.tls`
  بالإضافة إلى `gateway.remote.tlsFingerprint` أو CLI `--tls-fingerprint`).

## النطاق

يعرض هذا البروتوكول **واجهة gateway API الكاملة** (الحالة، والقنوات، والنماذج، والدردشة،
والوكيل، والجلسات، والعقد، والموافقات، وغير ذلك). ويُعرَّف السطح الدقيق بواسطة
TypeBox schemas في `src/gateway/protocol/schema.ts`.
