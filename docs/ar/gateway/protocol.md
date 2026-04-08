---
read_when:
    - تنفيذ عملاء WS للبوابة أو تحديثهم
    - تصحيح حالات عدم تطابق البروتوكول أو فشل الاتصال
    - إعادة توليد مخطط/نماذج البروتوكول
summary: 'بروتوكول Gateway عبر WebSocket: المصافحة، والإطارات، والإصدارات'
title: بروتوكول Gateway
x-i18n:
    generated_at: "2026-04-08T02:16:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8635e3ac1dd311dbd3a770b088868aa1495a8d53b3ebc1eae0dfda3b2bf4694a
    source_path: gateway/protocol.md
    workflow: 15
---

# بروتوكول Gateway (WebSocket)

يُعد بروتوكول Gateway عبر WS **مستوى التحكم الوحيد + ناقل العقد** في
OpenClaw. جميع العملاء (CLI، وواجهة الويب، وتطبيق macOS، وعقد iOS/Android، والعقد
عديمة الواجهة) يتصلون عبر WebSocket ويعلنون **الدور** + **النطاق**
في وقت المصافحة.

## النقل

- WebSocket، وإطارات نصية بحمولات JSON.
- **يجب** أن يكون الإطار الأول طلب `connect`.

## المصافحة (connect)

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

عند إصدار رمز جهاز، يتضمن `hello-ok` أيضًا:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

أثناء تسليم bootstrap الموثوق، قد يتضمن `hello-ok.auth` أيضًا
إدخالات أدوار إضافية مقيّدة في `deviceTokens`:

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

في تدفق bootstrap المضمّن للعقدة/المشغّل، يظل رمز العقدة الأساسي
`scopes: []` وأي رمز مشغّل تم تسليمه يظل مقيّدًا بقائمة السماح الخاصة بمشغّل bootstrap
(`operator.approvals`، و`operator.read`،
و`operator.talk.secrets`، و`operator.write`). تظل فحوصات نطاق bootstrap
مسبوقة بالدور: إدخالات المشغّل لا تلبّي إلا طلبات المشغّل، وما زالت الأدوار
غير المشغّلة تحتاج إلى نطاقات ضمن بادئة دورها الخاصة.

### مثال على عقدة

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

تتطلب الأساليب ذات الآثار الجانبية **مفاتيح idempotency** (راجع المخطط).

## الأدوار + النطاقات

### الأدوار

- `operator` = عميل مستوى التحكم (CLI/واجهة المستخدم/الأتمتة).
- `node` = مضيف الإمكانات (camera/screen/canvas/system.run).

### النطاقات (operator)

النطاقات الشائعة:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

يتطلب `talk.config` مع `includeSecrets: true` النطاق `operator.talk.secrets`
(أو `operator.admin`).

يمكن لأساليب RPC الخاصة بالبوابة والمسجَّلة من الإضافات أن تطلب نطاق operator خاصًا بها، لكن
بادئات الإدارة الأساسية المحجوزة (`config.*`، و`exec.approvals.*`، و`wizard.*`،
و`update.*`) تُحل دائمًا إلى `operator.admin`.

نطاق الأسلوب ليس سوى البوابة الأولى. بعض أوامر الشرطة المائلة التي يتم الوصول إليها
عبر `chat.send` تطبق فحوصات على مستوى الأمر أكثر صرامة فوق ذلك. على سبيل المثال،
تتطلب عمليات الكتابة الدائمة في `/config set` و`/config unset` النطاق `operator.admin`.

يحتوي `node.pair.approve` أيضًا على فحص نطاق إضافي في وقت الموافقة فوق نطاق
الأسلوب الأساسي:

- الطلبات بلا أوامر: `operator.pairing`
- الطلبات التي تتضمن أوامر عقدة غير تنفيذية: `operator.pairing` + `operator.write`
- الطلبات التي تتضمن `system.run` أو `system.run.prepare` أو `system.which`:
  `operator.pairing` + `operator.admin`

### caps/commands/permissions (node)

تعلن العقد مطالبات الإمكانات في وقت الاتصال:

- `caps`: فئات الإمكانات عالية المستوى.
- `commands`: قائمة سماح الأوامر للاستدعاء.
- `permissions`: مفاتيح تبديل دقيقة (مثل `screen.record` و`camera.capture`).

يتعامل Gateway مع هذه القيم على أنها **مطالبات** ويفرض قوائم سماح من جهة الخادم.

## الحضور

- يعيد `system-presence` إدخالات مفهرسة بحسب هوية الجهاز.
- تتضمن إدخالات الحضور `deviceId`، و`roles`، و`scopes` حتى تتمكن واجهات المستخدم من عرض صف واحد لكل جهاز
  حتى عندما يتصل بوصفه **operator** و**node** معًا.

## عائلات أساليب RPC الشائعة

هذه الصفحة ليست تفريغًا كاملاً مولدًا، لكن سطح WS العام أوسع
من أمثلة المصافحة/المصادقة أعلاه. هذه هي عائلات الأساليب الرئيسية التي
يكشفها Gateway اليوم.

تمثل `hello-ok.features.methods` قائمة اكتشاف متحفظة مبنية من
`src/gateway/server-methods-list.ts` بالإضافة إلى صادرات أساليب الإضافات/القنوات المحمَّلة.
تعامل معها بوصفها لاكتشاف الميزات، لا بوصفها تفريغًا مولدًا لكل دالة قابلة للاستدعاء
تم تنفيذها في `src/gateway/server-methods/*.ts`.

### النظام والهوية

- يعيد `health` لقطة صحة البوابة المخزنة مؤقتًا أو التي تم فحصها حديثًا.
- يعيد `status` ملخص البوابة على نمط `/status`؛ ويتم
  تضمين الحقول الحساسة فقط لعملاء operator ذوي نطاق الإدارة.
- يعيد `gateway.identity.get` هوية جهاز البوابة المستخدمة في
  تدفقات relay والاقتران.
- يعيد `system-presence` لقطة الحضور الحالية للأجهزة المتصلة
  بوصفها operator/node.
- يضيف `system-event` حدث نظام ويمكنه تحديث/بث
  سياق الحضور.
- يعيد `last-heartbeat` أحدث حدث heartbeat محفوظ.
- يقوم `set-heartbeats` بتبديل معالجة heartbeat على البوابة.

### النماذج والاستخدام

- يعيد `models.list` فهرس النماذج المسموح بها في بيئة التشغيل.
- يعيد `usage.status` ملخصات نوافذ استخدام الموفر/الحصة المتبقية.
- يعيد `usage.cost` ملخصات استخدام التكلفة المجمعة لنطاق تاريخ.
- يعيد `doctor.memory.status` جاهزية الذاكرة المتجهية / التضمين
  لمساحة عمل الوكيل الافتراضي النشط.
- يعيد `sessions.usage` ملخصات الاستخدام لكل جلسة.
- يعيد `sessions.usage.timeseries` سلسلة زمنية للاستخدام لجلسة واحدة.
- يعيد `sessions.usage.logs` إدخالات سجل الاستخدام لجلسة واحدة.

### القنوات ومساعدات تسجيل الدخول

- يعيد `channels.status` ملخصات حالة القنوات/الإضافات المضمّنة والمجمعة.
- يقوم `channels.logout` بتسجيل الخروج من قناة/حساب محدد حيث
  تدعم القناة تسجيل الخروج.
- يبدأ `web.login.start` تدفق تسجيل دخول QR/ويب للموفر
  الحالي لقناة الويب القادرة على QR.
- ينتظر `web.login.wait` اكتمال تدفق تسجيل دخول QR/ويب ويبدأ
  القناة عند النجاح.
- يرسل `push.test` إشعار APNs تجريبيًا إلى عقدة iOS مسجّلة.
- يعيد `voicewake.get` محفزات كلمة التنبيه المخزنة.
- يحدّث `voicewake.set` محفزات كلمة التنبيه ويبث التغيير.

### المراسلة والسجلات

- `send` هو RPC التسليم الصادر المباشر للرسائل
  الموجهة إلى قناة/حساب/سلسلة خارج مشغّل الدردشة.
- يعيد `logs.tail` ذيل سجل ملفات البوابة المضبوط مع المؤشر/الحد
  وعناصر التحكم في الحد الأقصى للبايتات.

### Talk وTTS

- يعيد `talk.config` حمولة إعداد Talk الفعالة؛ ويتطلب `includeSecrets`
  النطاق `operator.talk.secrets` (أو `operator.admin`).
- يضبط `talk.mode` حالة وضع Talk الحالية ويبثها لعملاء
  WebChat/Control UI.
- يقوم `talk.speak` بتوليف الكلام عبر موفر كلام Talk النشط.
- يعيد `tts.status` حالة تمكين TTS، والموفر النشط، وموفري البدائل،
  وحالة إعداد الموفر.
- يعيد `tts.providers` قائمة موفري TTS المرئية.
- يقوم `tts.enable` و`tts.disable` بتبديل حالة تفضيلات TTS.
- يحدّث `tts.setProvider` موفر TTS المفضل.
- يشغّل `tts.convert` تحويل نص إلى كلام لمرة واحدة.

### الأسرار، والإعداد، والتحديث، والمعالج

- يعيد `secrets.reload` حل SecretRefs النشطة ويبدّل حالة الأسرار في وقت التشغيل
  فقط عند النجاح الكامل.
- يقوم `secrets.resolve` بحل تعيينات الأسرار المستهدفة للأوامر لمجموعة
  أمر/هدف محددة.
- يعيد `config.get` لقطة الإعداد الحالية وتجزئتها.
- يكتب `config.set` حمولة إعداد تم التحقق من صحتها.
- يدمج `config.patch` تحديث إعداد جزئيًا.
- يتحقق `config.apply` من حمولة الإعداد الكاملة ويستبدلها.
- يعيد `config.schema` حمولة مخطط الإعداد المباشر المستخدمة بواسطة Control UI و
  أدوات CLI: المخطط، و`uiHints`، والإصدار، وبيانات التوليد الوصفية، بما في ذلك
  بيانات مخطط الإضافات + القنوات الوصفية عندما تستطيع بيئة التشغيل تحميلها. يتضمن
  المخطط بيانات الحقل `title` / `description`
  المشتقة من التسميات نفسها ونصوص المساعدة التي تستخدمها واجهة المستخدم، بما في ذلك
  الكائنات المتداخلة، والبدائل العامة، وعناصر المصفوفات،
  وفروع التركيب `anyOf` / `oneOf` / `allOf` عندما توجد
  وثائق حقل مطابقة.
- يعيد `config.schema.lookup` حمولة بحث محصورة بمسار لإعداد واحد
  لمسار إعداد واحد: مسار مُطبَّع، وعقدة مخطط سطحية، وتلميح مطابق + `hintPath`،
  وملخصات الأبناء المباشرين للتعمق عبر واجهة المستخدم/CLI.
  - تحتفظ عقد مخطط البحث بوثائق المستخدم والحقول الشائعة للتحقق:
    `title`، و`description`، و`type`، و`enum`، و`const`، و`format`، و`pattern`،
    وحدود الأرقام/السلاسل/المصفوفات/الكائنات، والأعلام المنطقية مثل
    `additionalProperties`، و`deprecated`، و`readOnly`، و`writeOnly`.
  - تكشف ملخصات الأبناء `key`، و`path` المُطبَّع، و`type`، و`required`،
    و`hasChildren`، بالإضافة إلى `hint` / `hintPath` المطابقين.
- يشغّل `update.run` تدفق تحديث البوابة ويجدول إعادة تشغيل فقط عندما
  ينجح التحديث نفسه.
- تكشف `wizard.start`، و`wizard.next`، و`wizard.status`، و`wizard.cancel`
  معالج الإعداد الأولي عبر WS RPC.

### العائلات الرئيسية الموجودة

#### مساعدات الوكيل ومساحة العمل

- يعيد `agents.list` إدخالات الوكلاء المضبوطة.
- تدير `agents.create`، و`agents.update`، و`agents.delete` سجلات الوكلاء و
  ربط مساحة العمل.
- تدير `agents.files.list`، و`agents.files.get`، و`agents.files.set`
  ملفات مساحة عمل bootstrap المعروضة لوكيل.
- يعيد `agent.identity.get` هوية المساعد الفعالة لوكيل أو
  جلسة.
- ينتظر `agent.wait` انتهاء التشغيل ويعيد اللقطة النهائية عند
  توفرها.

#### التحكم في الجلسات

- يعيد `sessions.list` فهرس الجلسات الحالي.
- تقوم `sessions.subscribe` و`sessions.unsubscribe` بتبديل اشتراكات
  أحداث تغيّر الجلسات لعميل WS الحالي.
- تقوم `sessions.messages.subscribe` و`sessions.messages.unsubscribe` بتبديل
  اشتراكات أحداث السجل/الرسائل لجلسة واحدة.
- يعيد `sessions.preview` معاينات سجلات محصورة لمفاتيح جلسات
  محددة.
- يقوم `sessions.resolve` بحل هدف جلسة أو جعله قياسيًا.
- تنشئ `sessions.create` إدخال جلسة جديدًا.
- ترسل `sessions.send` رسالة إلى جلسة موجودة.
- تمثل `sessions.steer` نسخة المقاطعة والتوجيه لجلسة نشطة.
- تقوم `sessions.abort` بإلغاء العمل النشط لجلسة.
- تحدّث `sessions.patch` البيانات الوصفية/التجاوزات الخاصة بالجلسة.
- تنفّذ `sessions.reset`، و`sessions.delete`، و`sessions.compact`
  صيانة الجلسة.
- يعيد `sessions.get` صف الجلسة المخزَّن بالكامل.
- ما زال تنفيذ الدردشة يستخدم `chat.history`، و`chat.send`، و`chat.abort`، و
  `chat.inject`.
- تمت تهيئة `chat.history` للعرض لعملاء UI: تتم
  إزالة وسوم التوجيه المضمنة من النص المرئي، كما تُزال حمولات XML النصية العادية
  لاستدعاءات الأدوات (بما في ذلك
  `<tool_call>...</tool_call>`، و`<function_call>...</function_call>`،
  و`<tool_calls>...</tool_calls>`، و`<function_calls>...</function_calls>`،
  وكتل استدعاءات الأدوات المقتطعة) ورموز التحكم بالنموذج المسرّبة بصيغة ASCII/العرض الكامل،
  كما تُحذف صفوف المساعد المؤلفة فقط من رموز صامتة مثل `NO_REPLY` /
  `no_reply`، ويمكن استبدال الصفوف كبيرة الحجم بعناصر نائبة.

#### اقتران الأجهزة ورموز الأجهزة

- يعيد `device.pair.list` الأجهزة المقترنة المعلقة والمعتمدة.
- تدير `device.pair.approve`، و`device.pair.reject`، و`device.pair.remove`
  سجلات اقتران الأجهزة.
- يقوم `device.token.rotate` بتدوير رمز جهاز مقترن ضمن حدود الدور
  والنطاق المعتمدة له.
- يقوم `device.token.revoke` بإبطال رمز جهاز مقترن.

#### اقتران العقد، والاستدعاء، والعمل المعلّق

- تغطي `node.pair.request`، و`node.pair.list`، و`node.pair.approve`،
  و`node.pair.reject`، و`node.pair.verify` اقتران العقد والتحقق من
  bootstrap.
- تعيد `node.list` و`node.describe` حالة العقد المعروفة/المتصلة.
- يحدّث `node.rename` تسمية عقدة مقترنة.
- يمرّر `node.invoke` أمرًا إلى عقدة متصلة.
- يعيد `node.invoke.result` النتيجة لطلب استدعاء.
- يحمل `node.event` الأحداث الصادرة من العقدة عائدًا إلى البوابة.
- يقوم `node.canvas.capability.refresh` بتحديث رموز إمكانات canvas
  المحصورة بالنطاق.
- تمثل `node.pending.pull` و`node.pending.ack` واجهات API الخاصة بطابور
  العقد المتصلة.
- تدير `node.pending.enqueue` و`node.pending.drain` العمل المعلّق الدائم
  للعقد غير المتصلة/المنفصلة.

#### عائلات الموافقات

- تغطي `exec.approval.request`، و`exec.approval.get`، و`exec.approval.list`، و
  `exec.approval.resolve` طلبات موافقة exec لمرة واحدة بالإضافة إلى
  البحث/إعادة التشغيل للموافقات المعلقة.
- ينتظر `exec.approval.waitDecision` قرار موافقة exec واحدًا معلقًا ويعيد
  القرار النهائي (أو `null` عند انتهاء المهلة).
- تدير `exec.approvals.get` و`exec.approvals.set` لقطات سياسة موافقة
  exec على البوابة.
- تدير `exec.approvals.node.get` و`exec.approvals.node.set` سياسة موافقة exec
  المحلية للعقدة عبر أوامر relay للعقدة.
- تغطي `plugin.approval.request`، و`plugin.approval.list`،
  و`plugin.approval.waitDecision`، و`plugin.approval.resolve`
  تدفقات الموافقة المعرفة من الإضافات.

#### عائلات رئيسية أخرى

- الأتمتة:
  - يقوم `wake` بجدولة حقن نص تنبيه فوري أو عند heartbeat التالي
  - `cron.list`، و`cron.status`، و`cron.add`، و`cron.update`، و`cron.remove`،
    و`cron.run`، و`cron.runs`
- Skills/tools: `skills.*`، و`tools.catalog`، و`tools.effective`

### عائلات الأحداث الشائعة

- `chat`: تحديثات دردشة واجهة المستخدم مثل `chat.inject` وغيرها من
  أحداث الدردشة الخاصة بالسجل فقط.
- `session.message` و`session.tool`: تحديثات السجل/تيار الأحداث لجلسة
  مشترَك بها.
- `sessions.changed`: تغيّر فهرس الجلسات أو بياناتها الوصفية.
- `presence`: تحديثات لقطة حضور النظام.
- `tick`: حدث دوري للإبقاء على الحياة / الحيوية.
- `health`: تحديث لقطة صحة البوابة.
- `heartbeat`: تحديث تيار أحداث heartbeat.
- `cron`: حدث تغير تشغيل/مهمة cron.
- `shutdown`: إشعار إيقاف تشغيل البوابة.
- `node.pair.requested` / `node.pair.resolved`: دورة حياة اقتران العقدة.
- `node.invoke.request`: بث طلب استدعاء العقدة.
- `device.pair.requested` / `device.pair.resolved`: دورة حياة الجهاز المقترن.
- `voicewake.changed`: تغيّر إعداد محفز كلمة التنبيه.
- `exec.approval.requested` / `exec.approval.resolved`: دورة حياة
  موافقة exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: دورة حياة موافقة
  الإضافة.

### أساليب مساعدة العقدة

- يمكن للعقد استدعاء `skills.bins` لجلب القائمة الحالية لملفات Skills التنفيذية
  لفحوصات السماح التلقائي.

### أساليب مساعدة operator

- يمكن للمشغّلين استدعاء `tools.catalog` (`operator.read`) لجلب فهرس الأدوات في وقت التشغيل
  لوكيل. تتضمن الاستجابة أدوات مجمعة وبيانات وصفية للمصدر:
  - `source`: `core` أو `plugin`
  - `pluginId`: مالك الإضافة عندما يكون `source="plugin"`
  - `optional`: ما إذا كانت أداة الإضافة اختيارية
- يمكن للمشغّلين استدعاء `tools.effective` (`operator.read`) لجلب جرد الأدوات الفعّال في وقت التشغيل
  لجلسة.
  - `sessionKey` مطلوب.
  - تستخلص البوابة سياق وقت التشغيل الموثوق من الجلسة من جهة الخادم بدلًا من قبول
    سياق المصادقة أو التسليم المورَّد من المستدعي.
  - تكون الاستجابة محصورة بالجلسة وتعكس ما يمكن للمحادثة النشطة استخدامه الآن،
    بما في ذلك أدوات core، وplugin، والقنوات.
- يمكن للمشغّلين استدعاء `skills.status` (`operator.read`) لجلب
  جرد Skills المرئي لوكيل.
  - `agentId` اختياري؛ احذفه لقراءة مساحة عمل الوكيل الافتراضية.
  - تتضمن الاستجابة الأهلية، والمتطلبات المفقودة، وفحوصات الإعداد، و
    خيارات التثبيت المنقّحة من دون كشف قيم الأسرار الخام.
- يمكن للمشغّلين استدعاء `skills.search` و`skills.detail` (`operator.read`) للحصول على
  بيانات اكتشاف ClawHub الوصفية.
- يمكن للمشغّلين استدعاء `skills.install` (`operator.admin`) في وضعين:
  - وضع ClawHub: `{ source: "clawhub", slug, version?, force? }` يثبت
    مجلد Skill داخل الدليل `skills/` لمساحة عمل الوكيل الافتراضية.
  - وضع مُثبّت البوابة: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    يشغّل إجراء `metadata.openclaw.install` معلنًا على مضيف البوابة.
- يمكن للمشغّلين استدعاء `skills.update` (`operator.admin`) في وضعين:
  - يقوم وضع ClawHub بتحديث slug متعقّب واحد أو جميع تثبيتات ClawHub المتعقبة في
    مساحة عمل الوكيل الافتراضية.
  - يقوم وضع الإعداد بترقيع قيم `skills.entries.<skillKey>` مثل `enabled`،
    و`apiKey`، و`env`.

## موافقات exec

- عندما يحتاج طلب exec إلى موافقة، تبث البوابة `exec.approval.requested`.
- يحسم عملاء operator ذلك عبر استدعاء `exec.approval.resolve` (يتطلب النطاق `operator.approvals`).
- بالنسبة إلى `host=node`، يجب أن يتضمن `exec.approval.request` القيمة `systemRunPlan` (الحقول القياسية `argv`/`cwd`/`rawCommand`/بيانات تعريف الجلسة). تُرفض الطلبات التي تفتقد `systemRunPlan`.
- بعد الموافقة، تعيد استدعاءات `node.invoke system.run` المُمرَّرة استخدام
  `systemRunPlan` القياسي هذا بوصفه سياق الأمر/`cwd`/الجلسة المعتمد.
- إذا قام مستدعٍ بتعديل `command` أو`rawCommand` أو`cwd` أو`agentId` أو
  `sessionKey` بين التحضير وتمرير `system.run` الموافق عليه النهائي، فإن
  البوابة ترفض التشغيل بدلًا من الوثوق بالحمولة المعدلة.

## بديل تسليم الوكيل

- يمكن أن تتضمن طلبات `agent` القيمة `deliver=true` لطلب تسليم صادر.
- يحافظ `bestEffortDeliver=false` على السلوك الصارم: أهداف التسليم غير المحلولة أو الداخلية فقط تعيد `INVALID_REQUEST`.
- يسمح `bestEffortDeliver=true` بالرجوع إلى تنفيذ على مستوى الجلسة فقط عندما لا يمكن حل مسار خارجي قابل للتسليم (على سبيل المثال جلسات داخلية/WebChat أو إعدادات متعددة القنوات ملتبسة).

## الإصدارات

- توجد `PROTOCOL_VERSION` في `src/gateway/protocol/schema.ts`.
- يرسل العملاء `minProtocol` + `maxProtocol`؛ ويرفض الخادم حالات عدم التطابق.
- يتم توليد المخططات + النماذج من تعريفات TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## المصادقة

- تستخدم مصادقة البوابة بالسر المشترك `connect.params.auth.token` أو
  `connect.params.auth.password`، حسب وضع المصادقة المضبوط.
- أوضاع حمل الهوية مثل Tailscale Serve
  (`gateway.auth.allowTailscale: true`) أو
  `gateway.auth.mode: "trusted-proxy"` غير loopback
  تحقق فحص مصادقة الاتصال من ترويسات الطلب بدلًا من `connect.params.auth.*`.
- يتجاوز الوضع `gateway.auth.mode: "none"` للولوج الخاص مصادقة الاتصال بالسر المشترك
  بالكامل؛ لا تعرّض هذا الوضع على منافذ عامة/غير موثوقة.
- بعد الاقتران، تصدر البوابة **رمز جهاز** محصورًا بدور الاتصال +
  النطاقات. ويُعاد في `hello-ok.auth.deviceToken` ويجب أن يحفظه العميل
  للاتصالات المستقبلية.
- يجب على العملاء حفظ `hello-ok.auth.deviceToken` الأساسي بعد أي
  اتصال ناجح.
- يجب أن تؤدي إعادة الاتصال باستخدام رمز الجهاز **المخزَّن** هذا أيضًا إلى إعادة استخدام
  مجموعة النطاقات المعتمدة والمخزنة لذلك الرمز. وهذا يحافظ على وصول
  القراءة/الفحص/الحالة الذي مُنح سابقًا ويمنع تقليص الاتصالات المعاد إنشاؤها
  بصمت إلى نطاق إدارة ضمني أضيق.
- تكون أولوية مصادقة الاتصال العادية: الرمز/كلمة المرور المشتركة الصريحة أولًا، ثم
  `deviceToken` الصريح، ثم الرمز المخزَّن لكل جهاز، ثم رمز bootstrap.
- تمثل إدخالات `hello-ok.auth.deviceTokens` الإضافية رموز تسليم bootstrap.
  احفظها فقط عندما يستخدم الاتصال مصادقة bootstrap على ناقل موثوق
  مثل `wss://` أو الاقتران المحلي/loopback.
- إذا قدّم عميل `deviceToken` **صريحًا** أو نطاقات `scopes` صريحة، فإن
  مجموعة النطاقات المطلوبة من المستدعي تظل هي المعتمدة؛ ولا يعاد استخدام النطاقات المخزنة
  إلا عندما يعيد العميل استخدام الرمز المخزَّن لكل جهاز.
- يمكن تدوير/إبطال رموز الأجهزة عبر `device.token.rotate` و
  `device.token.revoke` (يتطلب النطاق `operator.pairing`).
- يظل إصدار/تدوير الرموز محصورًا ضمن مجموعة الأدوار المعتمدة والمسجلة في
  إدخال اقتران ذلك الجهاز؛ ولا يمكن لتدوير رمز أن يوسّع الجهاز إلى
  دور لم تمنحه موافقة الاقتران أصلًا.
- بالنسبة إلى جلسات رموز الأجهزة المقترنة، تكون إدارة الجهاز محصورة ذاتيًا ما لم
  يكن لدى المستدعي أيضًا `operator.admin`: لا يمكن للمستدعين غير الإداريين إزالة/إبطال/تدوير
  إلا إدخال جهازهم **الخاص**.
- يتحقق `device.token.rotate` أيضًا من مجموعة نطاقات operator المطلوبة في مقابل
  نطاقات جلسة المستدعي الحالية. لا يمكن للمستدعين غير الإداريين تدوير رمز
  إلى مجموعة نطاقات operator أوسع مما يملكونه بالفعل.
- تتضمن حالات فشل المصادقة `error.details.code` بالإضافة إلى تلميحات الاسترداد:
  - `error.details.canRetryWithDeviceToken` (منطقي)
  - `error.details.recommendedNextStep` (`retry_with_device_token`، `update_auth_configuration`، `update_auth_credentials`، `wait_then_retry`، `review_auth_configuration`)
- سلوك العميل مع `AUTH_TOKEN_MISMATCH`:
  - يمكن للعملاء الموثوقين محاولة إعادة واحدة محدودة باستخدام رمز
    مخزَّن لكل جهاز.
  - إذا فشلت إعادة المحاولة تلك، يجب على العملاء إيقاف حلقات إعادة الاتصال التلقائية
    وإظهار إرشادات لإجراء من المشغّل.

## هوية الجهاز + الاقتران

- يجب أن تتضمن العقد هوية جهاز مستقرة (`device.id`) مشتقة من
  بصمة زوج مفاتيح.
- تصدر البوابات رموزًا لكل جهاز + دور.
- تتطلب معتمَدات الاقتران لمعرّفات الأجهزة الجديدة موافقات اقتران ما لم يكن
  التلقائي المحلي مفعّلًا.
- يتمحور الاعتماد التلقائي للاقتران حول اتصالات loopback المحلية المباشرة.
- لدى OpenClaw أيضًا مسار اتصال ذاتي ضيق على مستوى الواجهة الخلفية/الحاوية من أجل
  تدفقات المساعد الموثوقة ذات السر المشترك.
- ما تزال اتصالات tailnet أو LAN على المضيف نفسه تُعامل باعتبارها بعيدة من أجل الاقتران
  وتتطلب موافقة.
- يجب أن تتضمن جميع عملاء WS هوية `device` أثناء `connect` (operator + node).
  يمكن لـ Control UI حذفها فقط في هذه الأوضاع:
  - `gateway.controlUi.allowInsecureAuth=true` من أجل توافق HTTP غير الآمن على localhost فقط.
  - نجاح مصادقة Control UI الخاصة بـ operator مع `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (وضع طوارئ، خفض أمني شديد).
- يجب على جميع الاتصالات توقيع nonce المقدَّم من الخادم في `connect.challenge`.

### تشخيصات ترحيل مصادقة الجهاز

بالنسبة إلى العملاء القدامى الذين ما زالوا يستخدمون سلوك التوقيع السابق للتحدي، يعيد `connect` الآن
أكواد تفاصيل `DEVICE_AUTH_*` ضمن `error.details.code` مع `error.details.reason` ثابت.

إخفاقات الترحيل الشائعة:

| الرسالة                     | details.code                     | details.reason           | المعنى                                             |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | حذف العميل `device.nonce` (أو أرسله فارغًا).      |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | وقّع العميل باستخدام nonce قديم/خاطئ.             |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | حمولة التوقيع لا تطابق حمولة v2.                  |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | الطابع الزمني الموقَّع خارج الانحراف المسموح به.  |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | لا يطابق `device.id` بصمة المفتاح العام.          |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | فشل تنسيق/توحيد المفتاح العام.                    |

هدف الترحيل:

- انتظر دائمًا `connect.challenge`.
- وقّع حمولة v2 التي تتضمن nonce الخاص بالخادم.
- أرسل nonce نفسه في `connect.params.device.nonce`.
- حمولة التوقيع المفضلة هي `v3`، التي تربط `platform` و`deviceFamily`
  بالإضافة إلى حقول الجهاز/العميل/الدور/النطاقات/الرمز/nonce.
- ما تزال توقيعات `v2` القديمة مقبولة من أجل التوافق، لكن تثبيت بيانات
  الجهاز المقترن الوصفية ما يزال يتحكم بسياسة الأوامر عند إعادة الاتصال.

## TLS + التثبيت

- دعم TLS متاح لاتصالات WS.
- يمكن للعملاء تثبيت بصمة شهادة البوابة اختياريًا (راجع إعداد `gateway.tls`
  بالإضافة إلى `gateway.remote.tlsFingerprint` أو علم CLI `--tls-fingerprint`).

## النطاق

يكشف هذا البروتوكول **واجهة API الكاملة للبوابة** (الحالة، والقنوات، والنماذج، والدردشة،
والوكيل، والجلسات، والعقد، والموافقات، وغير ذلك). ويتم تعريف السطح الدقيق بواسطة
مخططات TypeBox في `src/gateway/protocol/schema.ts`.
