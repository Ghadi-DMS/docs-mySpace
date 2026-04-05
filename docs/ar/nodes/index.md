---
read_when:
    - إقران عقد iOS/Android مع gateway
    - استخدام canvas/camera الخاصة بالعقدة لسياق الوكيل
    - إضافة أوامر node جديدة أو مساعدات CLI
summary: 'Nodes: الاقتران، والقدرات، والأذونات، ومساعدات CLI لـ canvas/camera/screen/device/notifications/system'
title: Nodes
x-i18n:
    generated_at: "2026-04-05T12:49:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 201be0e13cb6d39608f0bbd40fd02333f68bd44f588538d1016fe864db7e038e
    source_path: nodes/index.md
    workflow: 15
---

# Nodes

**العقدة** هي جهاز مرافق (macOS/iOS/Android/بدون واجهة) يتصل بـ Gateway **WebSocket** ‏(المنفذ نفسه الذي يستخدمه المشغلون) مع `role: "node"` ويعرض سطح أوامر (مثل `canvas.*` و`camera.*` و`device.*` و`notifications.*` و`system.*`) عبر `node.invoke`. تفاصيل البروتوكول: [بروتوكول Gateway](/gateway/protocol).

وسيلة النقل القديمة: [بروتوكول Bridge](/gateway/bridge-protocol) ‏(TCP JSONL؛
لأغراض تاريخية فقط بالنسبة إلى العقد الحالية).

يمكن لـ macOS أيضًا العمل في **وضع node**: حيث يتصل تطبيق شريط القوائم بخادم WS الخاص بـ Gateway ويعرض أوامر canvas/camera المحلية كعقدة (لذلك يعمل `openclaw nodes …` على هذا الـ Mac).

ملاحظات:

- Nodes هي **أجهزة طرفية** وليست بوابات. فهي لا تشغّل خدمة gateway.
- تصل رسائل Telegram/WhatsApp/إلخ إلى **gateway**، وليس إلى nodes.
- دليل استكشاف الأخطاء وإصلاحها: [/nodes/troubleshooting](/nodes/troubleshooting)

## الاقتران + الحالة

**تستخدم WS nodes اقتران الأجهزة.** تعرض العقد هويتها أثناء `connect`؛ ويقوم Gateway
بإنشاء طلب اقتران جهاز لـ `role: node`. وافق عليه عبر CLI الخاص بالأجهزة (أو عبر واجهة المستخدم).

CLI سريع:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

إذا أعادت عقدة المحاولة مع تفاصيل مصادقة متغيرة (role/scopes/public key)، فسيتم
استبدال الطلب المعلق السابق وإنشاء `requestId` جديد. أعد تشغيل
`openclaw devices list` قبل الموافقة.

ملاحظات:

- يعلّم `nodes status` العقدة على أنها **مقترنة** عندما تتضمن أدوار اقتران الجهاز لديها قيمة `node`.
- سجل اقتران الجهاز هو العقد الدائم للأدوار المعتمدة. ويبقى
  تدوير الرموز داخل ذلك العقد؛ ولا يمكنه ترقية عقدة مقترنة إلى
  دور مختلف لم تمنحه الموافقة أصلًا.
- يُعد `node.pair.*` ‏(CLI: ‏`openclaw nodes pending/approve/reject/rename`) مخزن اقتران عقد منفصلًا مملوكًا للـ gateway؛
  وهو لا يتحكم في مصافحة WS ‏`connect`.
- يتبع نطاق الموافقة الأوامر المعلنة في الطلب المعلق:
  - طلب بلا أوامر: ‏`operator.pairing`
  - أوامر node غير التنفيذية: ‏`operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: ‏`operator.pairing` + `operator.admin`

## مضيف node بعيد (`system.run`)

استخدم **مضيف node** عندما تعمل Gateway على جهاز وتريد تنفيذ الأوامر
على جهاز آخر. يظل النموذج يتحدث إلى **gateway**؛ وتقوم gateway
بتمرير استدعاءات `exec` إلى **مضيف node** عندما يتم اختيار `host=node`.

### ما الذي يعمل وأين

- **مضيف Gateway**: يستقبل الرسائل، ويشغّل النموذج، ويوجّه استدعاءات الأدوات.
- **مضيف Node**: ينفذ `system.run`/`system.which` على جهاز العقدة.
- **الموافقات**: تُفرض على مضيف node عبر `~/.openclaw/exec-approvals.json`.

ملاحظة الموافقات:

- تربط عمليات تشغيل العقدة المدعومة بالموافقة سياق الطلب الدقيق.
- بالنسبة إلى عمليات التنفيذ المباشر لملفات shell/runtime، يربط OpenClaw أيضًا على أساس أفضل جهد
  مُعامل ملف محلي فعلي واحد، ويرفض التشغيل إذا تغيّر ذلك الملف قبل التنفيذ.
- إذا لم يتمكن OpenClaw من تحديد ملف محلي فعلي واحد بالضبط لأمر interpreter/runtime،
  فسيتم رفض التنفيذ المدعوم بالموافقة بدلًا من الادعاء بتغطية كاملة لوقت التشغيل. استخدم sandboxing،
  أو مضيفات منفصلة، أو قائمة سماح موثوقة صريحة/تدفقًا كاملاً من أجل دلالات interpreter الأوسع.

### ابدأ مضيف node ‏(مقدمة)

على جهاز node:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

### Gateway بعيدة عبر نفق SSH ‏(ربط loopback)

إذا كانت Gateway ترتبط بـ loopback ‏(`gateway.bind=loopback`، وهو الافتراضي في الوضع المحلي)،
فلا يمكن لمضيفات node البعيدة الاتصال مباشرة. أنشئ نفق SSH ووجّه
مضيف node إلى الطرف المحلي من النفق.

مثال (مضيف node -> مضيف gateway):

```bash
# الطرفية A (اتركها قيد التشغيل): مرّر المنفذ المحلي 18790 -> gateway 127.0.0.1:18789
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# الطرفية B: صدّر رمز gateway واتصل عبر النفق
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

ملاحظات:

- يدعم `openclaw node run` مصادقة الرمز أو كلمة المرور.
- يُفضَّل استخدام متغيرات البيئة: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- بديل التكوين هو `gateway.auth.token` / `gateway.auth.password`.
- في الوضع المحلي، يتجاهل مضيف node عمدًا القيم `gateway.remote.token` / `gateway.remote.password`.
- في الوضع البعيد، تكون `gateway.remote.token` / `gateway.remote.password` مؤهلة وفق قواعد الأولوية البعيدة.
- إذا كانت `gateway.auth.*` المحلية الفعالة مكوّنة كـ SecretRefs ولكنها غير محلولة، فإن مصادقة مضيف node تفشل بشكل مغلق.
- لا يحترم حل مصادقة مضيف node إلا متغيرات البيئة `OPENCLAW_GATEWAY_*`.

### ابدأ مضيف node ‏(كخدمة)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node restart
```

### اقترن + سمِّ

على مضيف gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

إذا أعادت العقدة المحاولة مع تغيّر تفاصيل المصادقة، فأعد تشغيل `openclaw devices list`
ووافق على `requestId` الحالية.

خيارات التسمية:

- `--display-name` على `openclaw node run` / `openclaw node install` ‏(تُحفَظ في `~/.openclaw/node.json` على العقدة).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` ‏(تجاوز من gateway).

### أضف الأوامر إلى قائمة السماح

تكون موافقات exec **لكل مضيف node**. أضف إدخالات قائمة السماح من gateway:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

توجد الموافقات على مضيف node في `~/.openclaw/exec-approvals.json`.

### وجّه exec إلى node

كوّن القيم الافتراضية (تكوين gateway):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.security allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

أو لكل جلسة:

```
/exec host=node security=allowlist node=<id-or-name>
```

بمجرد التعيين، فإن أي استدعاء `exec` مع `host=node` يعمل على مضيف node ‏(مع مراعاة
قائمة السماح/الموافقات الخاصة بالعقدة).

لن يختار `host=auto` مضيف node ضمنيًا من تلقاء نفسه، لكن يُسمح بطلب صريح لكل استدعاء من `host=node` من وضع `auto`. وإذا كنت تريد أن يكون تنفيذ node هو الافتراضي للجلسة، فعيّن `tools.exec.host=node` أو `/exec host=node ...` صراحةً.

ذو صلة:

- [CLI مضيف node](/cli/node)
- [أداة Exec](/tools/exec)
- [موافقات Exec](/tools/exec-approvals)

## استدعاء الأوامر

منخفض المستوى (RPC خام):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

توجد مساعدات أعلى مستوى لسير العمل الشائعة من نوع “إعطاء الوكيل مرفق MEDIA”.

## لقطات الشاشة (لقطات canvas)

إذا كانت العقدة تعرض Canvas ‏(WebView)، فإن `canvas.snapshot` تعيد `{ format, base64 }`.

مساعد CLI ‏(يكتب إلى ملف مؤقت ويطبع `MEDIA:<path>`):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### عناصر تحكم Canvas

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

ملاحظات:

- يقبل `canvas present` عناوين URL أو مسارات ملفات محلية (`--target`)، بالإضافة إلى `--x/--y/--width/--height` اختياريًا لتحديد الموضع.
- يقبل `canvas eval` JavaScript مضمنة (`--js`) أو وسيطة موضعية.

### A2UI ‏(Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

ملاحظات:

- يتم دعم A2UI v0.8 JSONL فقط (يتم رفض v0.9/createSurface).

## الصور + الفيديوهات (كاميرا العقدة)

الصور (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # الافتراضي: كلا الاتجاهين (سطران MEDIA)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
```

مقاطع الفيديو (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

ملاحظات:

- يجب أن تكون العقدة **في المقدمة** من أجل `canvas.*` و`camera.*` ‏(تعيد الاستدعاءات في الخلفية `NODE_BACKGROUND_UNAVAILABLE`).
- يتم تقييد مدة المقطع (حاليًا `<= 60s`) لتجنب الحمولات الكبيرة جدًا بتنسيق base64.
- سيطلب Android أذونات `CAMERA`/`RECORD_AUDIO` متى أمكن؛ وتفشل الأذونات المرفوضة برسالة `*_PERMISSION_REQUIRED`.

## تسجيلات الشاشة (nodes)

تعرض العقد المدعومة الأمر `screen.record` ‏(`mp4`). مثال:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

ملاحظات:

- يعتمد توفر `screen.record` على منصة العقدة.
- يتم تقييد تسجيلات الشاشة إلى `<= 60s`.
- يعطّل `--no-audio` التقاط الميكروفون على المنصات المدعومة.
- استخدم `--screen <index>` لاختيار شاشة عندما تتوفر شاشات متعددة.

## الموقع (nodes)

تعرض Nodes الأمر `location.get` عند تمكين Location في الإعدادات.

مساعد CLI:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

ملاحظات:

- يكون Location **معطلًا افتراضيًا**.
- تتطلب قيمة “Always” إذنًا من النظام؛ ويكون الجلب في الخلفية على أساس أفضل جهد.
- تتضمن الاستجابة lat/lon والدقة (بالمتر) والطابع الزمني.

## SMS ‏(عقد Android)

يمكن لعقد Android عرض `sms.send` عندما يمنح المستخدم إذن **SMS** ويدعم الجهاز الاتصال الخلوي.

استدعاء منخفض المستوى:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

ملاحظات:

- يجب قبول مطالبة الإذن على جهاز Android قبل الإعلان عن هذه القدرة.
- لن تعلن الأجهزة التي تعمل عبر Wi‑Fi فقط ومن دون اتصال خلوي عن `sms.send`.

## أوامر جهاز Android + البيانات الشخصية

يمكن لعقد Android الإعلان عن عائلات أوامر إضافية عند تمكين القدرات المقابلة.

العائلات المتاحة:

- `device.status` و`device.info` و`device.permissions` و`device.health`
- `notifications.list` و`notifications.actions`
- `photos.latest`
- `contacts.search` و`contacts.add`
- `calendar.events` و`calendar.add`
- `callLog.search`
- `sms.search`
- `motion.activity` و`motion.pedometer`

أمثلة على الاستدعاء:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

ملاحظات:

- يتم تقييد أوامر الحركة بالقدرات المتاحة من المستشعرات.

## أوامر النظام (مضيف node / عقدة mac)

تعرض عقدة macOS الأوامر `system.run` و`system.notify` و`system.execApprovals.get/set`.
ويعرض مضيف node بدون واجهة الأوامر `system.run` و`system.which` و`system.execApprovals.get/set`.

أمثلة:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway ready"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"name":"git"}'
```

ملاحظات:

- يعيد `system.run` قيمة stdout/stderr/exit code ضمن الحمولة.
- يمر تنفيذ shell الآن عبر أداة `exec` مع `host=node`؛ بينما تبقى `nodes` سطح RPC مباشرًا لأوامر العقدة الصريحة.
- لا تعرض `nodes invoke` الأمرين `system.run` أو `system.run.prepare`؛ فهذان يبقيان على مسار exec فقط.
- يقوم مسار exec بتحضير `systemRunPlan` معتمد قبل الموافقة. وبمجرد
  منح الموافقة، تقوم gateway بتمرير تلك الخطة المخزنة، وليس أي حقول أمر/cwd/session
  معدّلة لاحقًا من قبل المستدعي.
- يحترم `system.notify` حالة إذن الإشعارات في تطبيق macOS.
- تستخدم بيانات تعريف `platform` / `deviceFamily` غير المعروفة للعقدة قائمة سماح افتراضية محافظة تستبعد `system.run` و`system.which`. وإذا كنت تحتاج عمدًا إلى هذه الأوامر على منصة غير معروفة، فأضفها صراحةً عبر `gateway.nodes.allowCommands`.
- يدعم `system.run` الخيارات `--cwd` و`--env KEY=VAL` و`--command-timeout` و`--needs-screen-recording`.
- بالنسبة إلى مغلفات shell ‏(`bash|sh|zsh ... -c/-lc`)، يتم تقليص قيم `--env` الخاصة بنطاق الطلب إلى قائمة سماح صريحة (`TERM` و`LANG` و`LC_*` و`COLORTERM` و`NO_COLOR` و`FORCE_COLOR`).
- بالنسبة إلى قرارات السماح الدائم في وضع allowlist، تقوم مغلفات الإرسال المعروفة (`env` و`nice` و`nohup` و`stdbuf` و`timeout`) بحفظ مسارات الملفات التنفيذية الداخلية بدلًا من مسارات المغلف. وإذا لم يكن فك المغلف آمنًا، فلن يتم حفظ أي إدخال لقائمة السماح تلقائيًا.
- على مضيفات node التي تعمل بنظام Windows في وضع allowlist، تتطلب عمليات تشغيل مغلفات shell عبر `cmd.exe /c` موافقة (ولا يؤدي إدخال allowlist وحده إلى السماح التلقائي بصيغة المغلف).
- يدعم `system.notify` الخيارين `--priority <passive|active|timeSensitive>` و`--delivery <system|overlay|auto>`.
- تتجاهل مضيفات node تجاوزات `PATH` وتزيل مفاتيح بدء التشغيل/shell الخطرة (`DYLD_*` و`LD_*` و`NODE_OPTIONS` و`PYTHON*` و`PERL*` و`RUBYOPT` و`SHELLOPTS` و`PS4`). وإذا كنت تحتاج إلى إدخالات PATH إضافية، فقم بتكوين بيئة خدمة مضيف node ‏(أو بتثبيت الأدوات في مواقع قياسية) بدلًا من تمرير `PATH` عبر `--env`.
- في وضع node على macOS، يتم التحكم في `system.run` بواسطة موافقات exec في تطبيق macOS ‏(Settings → Exec approvals).
  وتتصرف الأوضاع ask/allowlist/full بالطريقة نفسها كما في مضيف node بدون واجهة؛ وتعيد المطالبات المرفوضة `SYSTEM_RUN_DENIED`.
- على مضيف node بدون واجهة، يتم التحكم في `system.run` بواسطة موافقات exec ‏(`~/.openclaw/exec-approvals.json`).

## ربط exec بالعقدة

عند توفر عدة nodes، يمكنك ربط exec بعقدة محددة.
ويؤدي هذا إلى تعيين العقدة الافتراضية لـ `exec host=node` ‏(ويمكن تجاوزها لكل وكيل).

الافتراضي العام:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

تجاوز لكل وكيل:

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

أزل التعيين للسماح بأي عقدة:

```bash
openclaw config unset tools.exec.node
openclaw config unset agents.list[0].tools.exec.node
```

## خريطة الأذونات

قد تتضمن Nodes خريطة `permissions` في `node.list` / `node.describe`، مفهرسة باسم الإذن (مثل `screenRecording` و`accessibility`) مع قيم منطقية (`true` = ممنوح).

## مضيف node بدون واجهة (متعدد المنصات)

يمكن لـ OpenClaw تشغيل **مضيف node بدون واجهة** (من دون UI) يتصل بـ Gateway
WebSocket ويعرض `system.run` / `system.which`. وهذا مفيد على Linux/Windows
أو لتشغيل عقدة مصغّرة إلى جانب خادم.

ابدأ تشغيله:

```bash
openclaw node run --host <gateway-host> --port 18789
```

ملاحظات:

- ما يزال الاقتران مطلوبًا (سيعرض Gateway مطالبة اقتران جهاز).
- يخزن مضيف node معرّف العقدة الخاص به، ورمزه، واسم العرض، ومعلومات اتصال gateway في `~/.openclaw/node.json`.
- تُفرض موافقات exec محليًا عبر `~/.openclaw/exec-approvals.json`
  (راجع [موافقات Exec](/tools/exec-approvals)).
- على macOS، ينفذ مضيف node بدون واجهة الأمر `system.run` محليًا افتراضيًا. اضبط
  `OPENCLAW_NODE_EXEC_HOST=app` لتوجيه `system.run` عبر مضيف exec في التطبيق المرافق؛ وأضف
  `OPENCLAW_NODE_EXEC_FALLBACK=0` لفرض استخدام مضيف التطبيق والفشل بشكل مغلق إذا لم يكن متاحًا.
- أضف `--tls` / `--tls-fingerprint` عندما تستخدم Gateway WS بروتوكول TLS.

## وضع node على Mac

- يتصل تطبيق شريط القوائم في macOS بخادم Gateway WS كعقدة (لذلك تعمل `openclaw nodes …` على هذا الـ Mac).
- في الوضع البعيد، يفتح التطبيق نفق SSH لمنفذ Gateway ويتصل بـ `localhost`.
