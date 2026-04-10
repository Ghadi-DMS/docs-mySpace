---
read_when:
    - تنفيذ عملاء Gateway WS أو تحديثهم
    - تصحيح عدم تطابق البروتوكول أو حالات فشل الاتصال
    - إعادة توليد مخطط/نماذج البروتوكول
summary: 'بروتوكول Gateway WebSocket: المصافحة، الإطارات، وإدارة الإصدارات'
title: بروتوكول Gateway
x-i18n:
    generated_at: "2026-04-10T07:17:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83c820c46d4803d571c770468fd6782619eaa1dca253e156e8087dec735c127f
    source_path: gateway/protocol.md
    workflow: 15
---

# بروتوكول Gateway ‏(WebSocket)

بروتوكول Gateway WS هو **مستوى التحكم الوحيد + نقل العقد** في
OpenClaw. تتصل جميع العملاء (CLI، وواجهة الويب، وتطبيق macOS، وعقد iOS/Android،
والعقد عديمة الواجهة) عبر WebSocket وتُعلن **الدور** + **النطاق**
في وقت المصافحة.

## النقل

- WebSocket، إطارات نصية مع حمولات JSON.
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

عند إصدار رمز جهاز مميز، يتضمن `hello-ok` أيضًا:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

أثناء تسليم التمهيد الموثوق، قد يتضمن `hello-ok.auth` أيضًا إدخالات دور إضافية
مقيدة ضمن `deviceTokens`:

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

في تدفق التمهيد المدمج للعقدة/المشغّل، يظل رمز العقدة الأساسي
`scopes: []` وأي رمز مشغّل يتم تسليمه يظل مقيدًا بقائمة السماح الخاصة
بمشغّل التمهيد (`operator.approvals`، `operator.read`،
`operator.talk.secrets`، `operator.write`). تظل فحوصات نطاق التمهيد
مسبوقة بالدور: إدخالات المشغّل تلبّي فقط طلبات المشغّل، والأدوار غير المشغّل
تظل بحاجة إلى نطاقات ضمن بادئة دورها الخاصة.

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

تتطلب الأساليب ذات التأثيرات الجانبية **مفاتيح idempotency** (راجع المخطط).

## الأدوار + النطاقات

### الأدوار

- `operator` = عميل مستوى التحكم (CLI/واجهة المستخدم/الأتمتة).
- `node` = مضيف الإمكانات (camera/screen/canvas/system.run).

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

قد تطلب أساليب Gateway RPC المسجلة من الإضافات نطاق مشغّل خاصًا بها، لكن
البوادئ الإدارية الأساسية المحجوزة (`config.*`، `exec.approvals.*`، `wizard.*`،
`update.*`) تُحل دائمًا إلى `operator.admin`.

نطاق الأسلوب هو البوابة الأولى فقط. بعض أوامر الشرطة المائلة التي يتم الوصول إليها عبر
`chat.send` تطبق فحوصات أكثر صرامة على مستوى الأمر فوق ذلك. على سبيل المثال،
تتطلب عمليات الكتابة المستمرة في `/config set` و`/config unset`
النطاق `operator.admin`.

يحتوي `node.pair.approve` أيضًا على فحص نطاق إضافي في وقت الموافقة فوق
نطاق الأسلوب الأساسي:

- الطلبات بلا أوامر: `operator.pairing`
- الطلبات التي تحتوي على أوامر عقدة غير تنفيذية: `operator.pairing` + `operator.write`
- الطلبات التي تتضمن `system.run` أو `system.run.prepare` أو `system.which`:
  `operator.pairing` + `operator.admin`

### `caps`/`commands`/`permissions` (`node`)

تُعلن العقد مطالبات الإمكانات عند وقت الاتصال:

- `caps`: فئات إمكانات عالية المستوى.
- `commands`: قائمة سماح الأوامر للاستدعاء.
- `permissions`: مفاتيح تبديل دقيقة (مثل `screen.record` و`camera.capture`).

يتعامل Gateway مع هذه القيم على أنها **مطالبات** ويفرض قوائم السماح من جهة الخادم.

## الوجود

- يعيد `system-presence` إدخالات مفهرسة بحسب هوية الجهاز.
- تتضمن إدخالات الوجود `deviceId` و`roles` و`scopes` حتى تتمكن واجهات المستخدم من عرض صف واحد لكل جهاز
  حتى عندما يتصل بصفته **operator** و**node** معًا.

## عائلات أساليب RPC الشائعة

هذه الصفحة ليست تفريغًا مولدًا كاملًا، لكن سطح WS العام أوسع
من أمثلة المصافحة/المصادقة أعلاه. هذه هي عائلات الأساليب الرئيسية التي يعرضها
Gateway اليوم.

`hello-ok.features.methods` هي قائمة اكتشاف متحفظة مبنية من
`src/gateway/server-methods-list.ts` بالإضافة إلى صادرات أساليب الإضافات/القنوات المحملة.
تعامل معها على أنها لاكتشاف الميزات، وليس كتفريغ مولد لكل مساعد قابل للاستدعاء
منفذ في `src/gateway/server-methods/*.ts`.

### النظام والهوية

- يعيد `health` لقطة سلامة Gateway المخزنة مؤقتًا أو التي تم فحصها حديثًا.
- يعيد `status` ملخص Gateway بأسلوب `/status`؛ وتُضمن الحقول الحساسة
  فقط لعملاء المشغّل ذوي نطاق الإدارة.
- يعيد `gateway.identity.get` هوية جهاز Gateway المستخدمة في تدفقات
  الترحيل والاقتران.
- يعيد `system-presence` لقطة الوجود الحالية للأجهزة المتصلة من نوع
  operator/node.
- يُلحق `system-event` حدث نظام ويمكنه تحديث/بث سياق الوجود.
- يعيد `last-heartbeat` أحدث حدث نبضة قلب محفوظ.
- يبدّل `set-heartbeats` معالجة نبضات القلب على Gateway.

### النماذج والاستخدام

- يعيد `models.list` فهرس النماذج المسموح بها وقت التشغيل.
- يعيد `usage.status` نوافذ استخدام المزود/ملخصات الحصة المتبقية.
- يعيد `usage.cost` ملخصات تكلفة الاستخدام المجمعة لنطاق تاريخ.
- يعيد `doctor.memory.status` جاهزية الذاكرة المتجهية / التضمين لمساحة عمل
  الوكيل الافتراضي النشطة.
- يعيد `sessions.usage` ملخصات الاستخدام لكل جلسة.
- يعيد `sessions.usage.timeseries` استخدامًا كسلسلة زمنية لجلسة واحدة.
- يعيد `sessions.usage.logs` إدخالات سجل الاستخدام لجلسة واحدة.

### القنوات ومساعدات تسجيل الدخول

- يعيد `channels.status` ملخصات حالة القنوات/الإضافات المدمجة والمجمعة.
- يسجل `channels.logout` الخروج من قناة/حساب محدد حيث تدعم القناة
  تسجيل الخروج.
- يبدأ `web.login.start` تدفق تسجيل دخول QR/الويب لمزود قناة الويب الحالي
  القادر على QR.
- ينتظر `web.login.wait` اكتمال تدفق تسجيل دخول QR/الويب هذا ويبدأ
  القناة عند النجاح.
- يرسل `push.test` إشعار APNs تجريبيًا إلى عقدة iOS مسجلة.
- يعيد `voicewake.get` محفزات كلمة التنبيه المخزنة.
- يحدّث `voicewake.set` محفزات كلمة التنبيه ويبث التغيير.

### المراسلة والسجلات

- `send` هو RPC التسليم الصادر المباشر لعمليات الإرسال الموجهة إلى
  قناة/حساب/سلسلة محادثة خارج مشغّل الدردشة.
- يعيد `logs.tail` ذيل سجل ملفات Gateway المكوّن مع عناصر تحكم
  بالمؤشر/الحد والحد الأقصى للبايتات.

### Talk وTTS

- يعيد `talk.config` حمولة إعداد Talk الفعالة؛ ويتطلب `includeSecrets`
  النطاق `operator.talk.secrets` (أو `operator.admin`).
- يضبط `talk.mode` حالة وضع Talk الحالية لعملاء WebChat/Control UI
  ويبثها.
- يقوم `talk.speak` بتركيب الكلام عبر مزود كلام Talk النشط.
- يعيد `tts.status` حالة تمكين TTS، والمزود النشط، والمزودين الاحتياطيين،
  وحالة إعداد المزود.
- يعيد `tts.providers` مخزون مزودي TTS المرئي.
- يبدّل `tts.enable` و`tts.disable` حالة تفضيلات TTS.
- يحدّث `tts.setProvider` مزود TTS المفضل.
- يشغّل `tts.convert` تحويل نص إلى كلام لمرة واحدة.

### الأسرار والإعداد والتحديث والمعالج

- يعيد `secrets.reload` حل SecretRefs النشطة ويبدّل حالة الأسرار وقت التشغيل
  فقط عند النجاح الكامل.
- يحل `secrets.resolve` تعيينات الأسرار المستهدفة بالأوامر لمجموعة
  أوامر/أهداف محددة.
- يعيد `config.get` لقطة الإعداد الحالية وتجزئتها.
- يكتب `config.set` حمولة إعداد تم التحقق من صحتها.
- يدمج `config.patch` تحديث إعداد جزئي.
- يتحقق `config.apply` من حمولة الإعداد الكاملة ويستبدلها.
- يعيد `config.schema` حمولة مخطط الإعداد الحي المستخدمة بواسطة Control UI وأدوات
  CLI: المخطط، و`uiHints`، والإصدار، وبيانات التوليد الوصفية، بما في ذلك
  بيانات مخطط الإضافات + القنوات الوصفية عندما يكون وقت التشغيل قادرًا على تحميلها. يتضمن المخطط
  بيانات `title` / `description` الوصفية للحقول المشتقة من نفس التسميات
  ونصوص المساعدة التي تستخدمها واجهة المستخدم، بما في ذلك الفروع المتداخلة
  للكائنات، والرموز العامة، وعناصر المصفوفات، وتركيبات
  `anyOf` / `oneOf` / `allOf` عندما تكون وثائق الحقول المطابقة موجودة.
- يعيد `config.schema.lookup` حمولة بحث مقيّدة بالمسار لمسار إعداد واحد:
  المسار الموحّد، وعقدة مخطط سطحية، و`hint` + `hintPath` المطابقين،
  وملخصات الأبناء المباشرين للتعمق في واجهة المستخدم/CLI.
  - تحتفظ عقد مخطط البحث بوثائق المستخدم والحقول الشائعة للتحقق:
    `title` و`description` و`type` و`enum` و`const` و`format` و`pattern`،
    وحدود الأرقام/السلاسل/المصفوفات/الكائنات، وأعلام منطقية مثل
    `additionalProperties` و`deprecated` و`readOnly` و`writeOnly`.
  - تعرض ملخصات الأبناء `key` و`path` الموحّد و`type` و`required`،
    و`hasChildren`، بالإضافة إلى `hint` / `hintPath` المطابقين.
- يشغّل `update.run` تدفق تحديث Gateway ويجدول إعادة تشغيل فقط عندما
  ينجح التحديث نفسه.
- تعرض `wizard.start` و`wizard.next` و`wizard.status` و`wizard.cancel`
  معالج الإعداد الأولي عبر WS RPC.

### العائلات الرئيسية الحالية

#### مساعدات الوكيل ومساحة العمل

- يعيد `agents.list` إدخالات الوكلاء المكوّنة.
- تدير `agents.create` و`agents.update` و`agents.delete` سجلات الوكلاء
  وربط مساحة العمل.
- تدير `agents.files.list` و`agents.files.get` و`agents.files.set`
  ملفات مساحة العمل الأولية المعروضة لوكيل.
- يعيد `agent.identity.get` هوية المساعد الفعالة لوكيل أو
  جلسة.
- ينتظر `agent.wait` انتهاء التشغيل ويعيد اللقطة النهائية عند
  توفرها.

#### التحكم في الجلسة

#### التحكم في الجلسة

- يعيد `sessions.list` فهرس الجلسات الحالي.
- يبدّل `sessions.subscribe` و`sessions.unsubscribe` اشتراكات
  أحداث تغيّر الجلسة لعميل WS الحالي.
- يبدّل `sessions.messages.subscribe` و`sessions.messages.unsubscribe`
  اشتراكات أحداث النصوص/الرسائل لجلسة واحدة.
- يعيد `sessions.preview` معاينات نصوص مقيّدة لمفاتيح جلسات محددة.
- يحل `sessions.resolve` هدف الجلسة أو يحوله إلى الصيغة القياسية.
- ينشئ `sessions.create` إدخال جلسة جديدًا.
- يرسل `sessions.send` رسالة إلى جلسة موجودة.
- `sessions.steer` هو متغير المقاطعة وإعادة التوجيه لجلسة نشطة.
- يوقف `sessions.abort` العمل النشط لجلسة.
- يحدّث `sessions.patch` بيانات الجلسة الوصفية/التجاوزات.
- تنفذ `sessions.reset` و`sessions.delete` و`sessions.compact`
  صيانة الجلسة.
- يعيد `sessions.get` صف الجلسة المخزن بالكامل.
- لا يزال تنفيذ الدردشة يستخدم `chat.history` و`chat.send` و`chat.abort` و
  `chat.inject`.
- يتم تطبيع `chat.history` للعرض لعملاء واجهة المستخدم: تُزال وسوم التوجيه المضمّنة من
  النص المرئي، وتُزال حمولات XML النصية العادية لاستدعاءات الأدوات
  (بما في ذلك `<tool_call>...</tool_call>` و`<function_call>...</function_call>`،
  و`<tool_calls>...</tool_calls>` و`<function_calls>...</function_calls>`،
  وكتل استدعاء الأدوات المقتطعة) ورموز التحكم الخاصة بالنموذج
  المسرّبة بنمط ASCII/العرض الكامل، وتُحذف صفوف المساعد التي تحتوي فقط على
  رموز صامتة مثل `NO_REPLY` / `no_reply` تمامًا، ويمكن استبدال
  الصفوف الكبيرة جدًا بعناصر نائبة.

#### اقتران الأجهزة ورموز الأجهزة المميزة

- يعيد `device.pair.list` الأجهزة المقترنة المعلقة والموافق عليها.
- تدير `device.pair.approve` و`device.pair.reject` و`device.pair.remove`
  سجلات اقتران الأجهزة.
- يقوم `device.token.rotate` بتدوير رمز جهاز مقترن مميز ضمن حدود
  الدور والنطاق الموافق عليهما.
- يلغي `device.token.revoke` رمز جهاز مميز مقترن.

#### اقتران العقدة والاستدعاء والعمل المعلّق

- تغطي `node.pair.request` و`node.pair.list` و`node.pair.approve` و
  `node.pair.reject` و`node.pair.verify` اقتران العقدة والتحقق من
  التمهيد.
- يعيد `node.list` و`node.describe` حالة العقدة المعروفة/المتصلة.
- يحدّث `node.rename` تسمية عقدة مقترنة.
- يمرر `node.invoke` أمرًا إلى عقدة متصلة.
- يعيد `node.invoke.result` النتيجة لطلب استدعاء.
- يحمل `node.event` الأحداث الصادرة من العقدة مرة أخرى إلى Gateway.
- يحدّث `node.canvas.capability.refresh` رموز إمكانات canvas المميزة
  المقيّدة بالنطاق.
- `node.pending.pull` و`node.pending.ack` هما واجهتا API لطابور
  العقدة المتصلة.
- تدير `node.pending.enqueue` و`node.pending.drain` العمل المعلّق الدائم
  للعقد غير المتصلة/غير المتاحة.

#### عائلات الموافقات

- تغطي `exec.approval.request` و`exec.approval.get` و`exec.approval.list` و
  `exec.approval.resolve` طلبات موافقة التنفيذ لمرة واحدة بالإضافة إلى
  البحث/إعادة التشغيل للموافقات المعلقة.
- ينتظر `exec.approval.waitDecision` موافقة تنفيذ معلقة واحدة ويعيد
  القرار النهائي (أو `null` عند انتهاء المهلة).
- تدير `exec.approvals.get` و`exec.approvals.set` لقطات سياسة موافقة
  التنفيذ في Gateway.
- تدير `exec.approvals.node.get` و`exec.approvals.node.set` سياسة موافقة
  التنفيذ المحلية للعقدة عبر أوامر ترحيل العقدة.
- تغطي `plugin.approval.request` و`plugin.approval.list` و
  `plugin.approval.waitDecision` و`plugin.approval.resolve`
  تدفقات الموافقة المعرفة بواسطة الإضافات.

#### عائلات رئيسية أخرى

- الأتمتة:
  - يجدول `wake` حقن نص استيقاظ فوري أو عند نبضة القلب التالية
  - `cron.list` و`cron.status` و`cron.add` و`cron.update` و`cron.remove`،
    و`cron.run` و`cron.runs`
- Skills/الأدوات: `commands.list` و`skills.*` و`tools.catalog` و`tools.effective`

### عائلات الأحداث الشائعة

- `chat`: تحديثات دردشة واجهة المستخدم مثل `chat.inject` وأحداث
  الدردشة الأخرى الخاصة بالنص فقط.
- `session.message` و`session.tool`: تحديثات النص/تدفق الأحداث
  لجلسة مشترَك فيها.
- `sessions.changed`: تغيّر فهرس الجلسات أو بياناتها الوصفية.
- `presence`: تحديثات لقطة وجود النظام.
- `tick`: حدث دوري للإبقاء على النشاط / التحقق من الحيوية.
- `health`: تحديث لقطة سلامة Gateway.
- `heartbeat`: تحديث تدفق أحداث نبضات القلب.
- `cron`: حدث تغيير تشغيل/مهمة cron.
- `shutdown`: إشعار إيقاف Gateway.
- `node.pair.requested` / `node.pair.resolved`: دورة حياة اقتران العقدة.
- `node.invoke.request`: بث طلب استدعاء العقدة.
- `device.pair.requested` / `device.pair.resolved`: دورة حياة الجهاز المقترن.
- `voicewake.changed`: تغيّر إعداد محفز كلمة التنبيه.
- `exec.approval.requested` / `exec.approval.resolved`: دورة حياة
  موافقة التنفيذ.
- `plugin.approval.requested` / `plugin.approval.resolved`: دورة حياة
  موافقة الإضافة.

### أساليب مساعدة العقدة

- يمكن للعقد استدعاء `skills.bins` لجلب القائمة الحالية لملفات Skills التنفيذية
  لفحوصات السماح التلقائي.

### أساليب مساعدة المشغّل

- يمكن للمشغّلين استدعاء `commands.list` (`operator.read`) لجلب
  مخزون الأوامر وقت التشغيل لوكيل.
  - `agentId` اختياري؛ احذفه لقراءة مساحة عمل الوكيل الافتراضي.
  - يتحكم `scope` في السطح الذي يستهدفه `name` الأساسي:
    - يعيد `text` رمز أمر النص الأساسي بدون `/` البادئة
    - يعيد `native` ومسار `both` الافتراضي أسماء أصلية مدركة للمزوّد
      عند توفرها
  - يحمل `textAliases` الأسماء المستعارة الدقيقة للشرطة المائلة مثل `/model` و`/m`.
  - يحمل `nativeName` اسم الأمر الأصلي المدرك للمزوّد عند وجوده.
  - `provider` اختياري ويؤثر فقط في التسمية الأصلية بالإضافة إلى
    توفر أوامر الإضافات الأصلية.
  - يحذف `includeArgs=false` بيانات الوسائط التسلسلية من الاستجابة.
- يمكن للمشغّلين استدعاء `tools.catalog` (`operator.read`) لجلب فهرس الأدوات وقت التشغيل لوكيل.
  تتضمن الاستجابة أدوات مجمعة وبيانات وصفية لمصدرها:
  - `source`: `core` أو `plugin`
  - `pluginId`: مالك الإضافة عندما يكون `source="plugin"`
  - `optional`: ما إذا كانت أداة الإضافة اختيارية
- يمكن للمشغّلين استدعاء `tools.effective` (`operator.read`) لجلب
  مخزون الأدوات الفعّال وقت التشغيل لجلسة.
  - `sessionKey` مطلوب.
  - يستمد Gateway سياق وقت التشغيل الموثوق من الجلسة من جهة الخادم بدلًا من قبول
    سياق المصادقة أو التسليم المورّد من المستدعي.
  - تكون الاستجابة مقيّدة بالجلسة وتعكس ما يمكن للمحادثة النشطة استخدامه الآن،
    بما في ذلك أدوات core والإضافات والقنوات.
- يمكن للمشغّلين استدعاء `skills.status` (`operator.read`) لجلب
  مخزون Skills المرئي لوكيل.
  - `agentId` اختياري؛ احذفه لقراءة مساحة عمل الوكيل الافتراضي.
  - تتضمن الاستجابة الأهلية، والمتطلبات المفقودة، وفحوصات الإعداد،
    وخيارات التثبيت المنقحة دون كشف قيم الأسرار الخام.
- يمكن للمشغّلين استدعاء `skills.search` و`skills.detail` (`operator.read`)
  لبيانات اكتشاف ClawHub الوصفية.
- يمكن للمشغّلين استدعاء `skills.install` (`operator.admin`) في وضعين:
  - وضع ClawHub: يقوم `{ source: "clawhub", slug, version?, force? }` بتثبيت
    مجلد Skill في دليل `skills/` لمساحة عمل الوكيل الافتراضية.
  - وضع مُثبّت Gateway: يشغّل `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    إجراء `metadata.openclaw.install` معلنًا على مضيف Gateway.
- يمكن للمشغّلين استدعاء `skills.update` (`operator.admin`) في وضعين:
  - يحدّث وضع ClawHub slugًا واحدًا متتبعًا أو جميع تثبيتات ClawHub المتتبعة في
    مساحة عمل الوكيل الافتراضية.
  - يقوم وضع الإعداد بترقيع قيم `skills.entries.<skillKey>` مثل `enabled`،
    و`apiKey`، و`env`.

## موافقات التنفيذ

- عندما يحتاج طلب تنفيذ إلى موافقة، يبث Gateway الحدث `exec.approval.requested`.
- يحل عملاء المشغّل ذلك عبر استدعاء `exec.approval.resolve` (يتطلب النطاق `operator.approvals`).
- بالنسبة إلى `host=node`، يجب أن يتضمن `exec.approval.request` القيمة `systemRunPlan`
  (`argv`/`cwd`/`rawCommand`/بيانات الجلسة الوصفية بصيغتها القياسية). تُرفض الطلبات
  التي تفتقد `systemRunPlan`.
- بعد الموافقة، تعيد استدعاءات `node.invoke system.run` الممررة استخدام
  `systemRunPlan` القياسي هذا بوصفه السياق الموثوق للأمر/دليل العمل/الجلسة.
- إذا غيّر مستدعٍ `command` أو `rawCommand` أو `cwd` أو `agentId` أو
  `sessionKey` بين التحضير وتمرير `system.run` النهائي الموافق عليه،
  يرفض Gateway التشغيل بدلًا من الوثوق بالحمولة المعدّلة.

## الاحتياط لتسليم الوكيل

- يمكن أن تتضمن طلبات `agent` القيمة `deliver=true` لطلب تسليم صادر.
- يحافظ `bestEffortDeliver=false` على السلوك الصارم: أهداف التسليم غير المحلولة أو الداخلية فقط تعيد `INVALID_REQUEST`.
- يسمح `bestEffortDeliver=true` بالاحتياط إلى تنفيذ داخل الجلسة فقط عندما يتعذر حل مسار خارجي قابل للتسليم (مثل جلسات webchat الداخلية أو إعدادات القنوات المتعددة الملتبسة).

## إدارة الإصدارات

- يوجد `PROTOCOL_VERSION` في `src/gateway/protocol/schema.ts`.
- يرسل العملاء `minProtocol` و`maxProtocol`؛ ويرفض الخادم حالات عدم التطابق.
- تُولد المخططات + النماذج من تعريفات TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## المصادقة

- تستخدم مصادقة Gateway ذات السر المشترك `connect.params.auth.token` أو
  `connect.params.auth.password`، بحسب وضع المصادقة المُكوَّن.
- أوضاع حمل الهوية مثل Tailscale Serve
  (`gateway.auth.allowTailscale: true`) أو
  `gateway.auth.mode: "trusted-proxy"` على منافذ غير loopback
  تستوفي فحص مصادقة الاتصال من رؤوس الطلب بدلًا من `connect.params.auth.*`.
- يتخطى `gateway.auth.mode: "none"` على الإدخال الخاص
  مصادقة الاتصال ذات السر المشترك بالكامل؛ لا تعرّض هذا الوضع على إدخال عام/غير موثوق.
- بعد الاقتران، يصدر Gateway **رمز جهاز مميزًا** مقيّدًا بدور الاتصال + النطاقات.
  ويُعاد في `hello-ok.auth.deviceToken` ويجب أن يحتفظ به العميل
  للاتصالات المستقبلية.
- يجب على العملاء الاحتفاظ برمز الجهاز المميز الأساسي `hello-ok.auth.deviceToken` بعد
  أي اتصال ناجح.
- يجب أن تؤدي إعادة الاتصال باستخدام **رمز الجهاز المميز المخزن** هذا أيضًا إلى إعادة استخدام
  مجموعة النطاقات الموافق عليها المخزنة لهذا الرمز. يحافظ هذا على وصول
  القراءة/الفحص/الحالة الذي مُنح بالفعل ويتجنب تضييق إعادة الاتصال
  بصمت إلى نطاق إداري ضمني أضيق فقط.
- أولوية مصادقة الاتصال العادية هي أولًا الرمز/كلمة المرور المشتركة الصريحة، ثم
  `deviceToken` الصريح، ثم الرمز المخزن لكل جهاز، ثم رمز التمهيد.
- الإدخالات الإضافية في `hello-ok.auth.deviceTokens` هي رموز تسليم تمهيد.
  احتفظ بها فقط عندما يستخدم الاتصال مصادقة التمهيد على نقل موثوق
  مثل `wss://` أو loopback/الاقتران المحلي.
- إذا قدّم عميل **`deviceToken` صريحًا** أو `scopes` صريحة، فستظل
  مجموعة النطاقات التي طلبها المستدعي هي المعتمدة؛ لا يُعاد استخدام النطاقات المخزنة
  إلا عندما يعيد العميل استخدام الرمز المخزن لكل جهاز.
- يمكن تدوير/إلغاء رموز الأجهزة المميزة عبر `device.token.rotate` و
  `device.token.revoke` (يتطلب النطاق `operator.pairing`).
- يظل إصدار/تدوير الرموز المميزة مقيّدًا بمجموعة الأدوار الموافق عليها والمُسجَّلة في
  إدخال اقتران ذلك الجهاز؛ ولا يمكن أن يؤدي تدوير رمز مميز إلى توسيع الجهاز إلى
  دور لم تمنحه موافقة الاقتران أصلًا.
- بالنسبة إلى جلسات رموز الأجهزة المميزة للأجهزة المقترنة، تكون إدارة الأجهزة مقيّدة
  ذاتيًا ما لم يكن لدى المستدعي أيضًا `operator.admin`: لا يمكن للمستدعين غير الإداريين
  الإزالة/الإلغاء/التدوير إلا لإدخال أجهزتهم **هم** فقط.
- يتحقق `device.token.rotate` أيضًا من مجموعة نطاقات المشغّل المطلوبة مقابل
  نطاقات جلسة المستدعي الحالية. لا يمكن للمستدعين غير الإداريين تدوير رمز مميز إلى
  مجموعة نطاقات مشغّل أوسع مما لديهم بالفعل.
- تتضمن حالات فشل المصادقة `error.details.code` بالإضافة إلى تلميحات
  للاسترداد:
  - `error.details.canRetryWithDeviceToken` (قيمة منطقية)
  - `error.details.recommendedNextStep` (`retry_with_device_token`، `update_auth_configuration`، `update_auth_credentials`، `wait_then_retry`، `review_auth_configuration`)
- سلوك العميل عند `AUTH_TOKEN_MISMATCH`:
  - يمكن للعملاء الموثوقين محاولة إعادة واحدة مقيّدة باستخدام رمز مميز مخزن لكل جهاز.
  - إذا فشلت إعادة المحاولة تلك، فيجب على العملاء إيقاف حلقات إعادة الاتصال التلقائية وعرض إرشادات إجراء المشغّل.

## هوية الجهاز + الاقتران

- يجب أن تتضمن العقد هوية جهاز مستقرة (`device.id`) مشتقة من
  بصمة زوج مفاتيح.
- تصدر بوابات Gateway رموزًا مميزة لكل جهاز + دور.
- يلزم الحصول على موافقات الاقتران لمعرّفات الأجهزة الجديدة ما لم يكن
  القبول التلقائي المحلي مفعّلًا.
- يتركز القبول التلقائي للاقتران على اتصالات local loopback المباشرة.
- يحتوي OpenClaw أيضًا على مسار اتصال ذاتي ضيق على مستوى الخلفية/الحاوية
  لتدفقات المساعد الموثوق ذات السر المشترك.
- لا تزال اتصالات tailnet أو LAN على المضيف نفسه تُعامَل على أنها بعيدة لأغراض الاقتران
  وتتطلب موافقة.
- يجب على جميع عملاء WS تضمين هوية `device` أثناء `connect` (operator + node).
  يمكن لـ Control UI حذفها فقط في هذه الأوضاع:
  - `gateway.controlUi.allowInsecureAuth=true` للتوافق مع HTTP غير الآمن على localhost فقط.
  - نجاح مصادقة Control UI للمشغّل عبر `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (خيار طارئ، تخفيض أمني شديد).
- يجب على جميع الاتصالات توقيع قيمة `connect.challenge` nonce التي يوفّرها الخادم.

### تشخيصات ترحيل مصادقة الجهاز

بالنسبة إلى العملاء القدامى الذين ما زالوا يستخدمون سلوك التوقيع السابق للتحدي، يعيد `connect` الآن
رموز التفاصيل `DEVICE_AUTH_*` ضمن `error.details.code` مع قيمة `error.details.reason` مستقرة.

إخفاقات الترحيل الشائعة:

| الرسالة | details.code | details.reason | المعنى |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required` | `DEVICE_AUTH_NONCE_REQUIRED` | `device-nonce-missing` | أغفل العميل `device.nonce` (أو أرسله فارغًا). |
| `device nonce mismatch` | `DEVICE_AUTH_NONCE_MISMATCH` | `device-nonce-mismatch` | وقّع العميل باستخدام nonce قديم/خاطئ. |
| `device signature invalid` | `DEVICE_AUTH_SIGNATURE_INVALID` | `device-signature` | لا تطابق حمولة التوقيع حمولة v2. |
| `device signature expired` | `DEVICE_AUTH_SIGNATURE_EXPIRED` | `device-signature-stale` | الطابع الزمني الموقّع خارج حدود الانحراف المسموح بها. |
| `device identity mismatch` | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch` | لا يطابق `device.id` بصمة المفتاح العام. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key` | فشل تنسيق/توحيد المفتاح العام. |

هدف الترحيل:

- انتظر دائمًا `connect.challenge`.
- وقّع حمولة v2 التي تتضمن nonce الخادم.
- أرسل nonce نفسه في `connect.params.device.nonce`.
- حمولة التوقيع المفضلة هي `v3`، التي تربط `platform` و`deviceFamily`
  بالإضافة إلى حقول device/client/role/scopes/token/nonce.
- تظل تواقيع `v2` القديمة مقبولة للتوافق، لكن تثبيت البيانات الوصفية
  للجهاز المقترن يظل يتحكم في سياسة الأوامر عند إعادة الاتصال.

## TLS + التثبيت

- يدعم TLS اتصالات WS.
- يمكن للعملاء اختياريًا تثبيت بصمة شهادة Gateway (راجع إعداد
  `gateway.tls` بالإضافة إلى `gateway.remote.tlsFingerprint` أو وسيط CLI `--tls-fingerprint`).

## النطاق

يكشف هذا البروتوكول **واجهة API الكاملة لـ Gateway** (الحالة، والقنوات، والنماذج، والدردشة،
والوكيل، والجلسات، والعقد، والموافقات، وغير ذلك). ويُعرَّف السطح الدقيق بواسطة
مخططات TypeBox في `src/gateway/protocol/schema.ts`.
