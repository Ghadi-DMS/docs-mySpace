---
read_when:
    - تشغيل Gateway من CLI (للتطوير أو الخوادم)
    - تصحيح مصادقة Gateway وأوضاع bind والاتصال
    - اكتشاف Gateways عبر Bonjour ‏(local + wide-area DNS-SD)
summary: CLI الخاص بـ OpenClaw Gateway (`openclaw gateway`) — تشغيل Gateways والاستعلام عنها واكتشافها
title: gateway
x-i18n:
    generated_at: "2026-04-05T12:38:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: e311ded0dbad84b8212f0968f3563998d49c5e0eb292a0dc4b3bd3c22d4fa7f2
    source_path: cli/gateway.md
    workflow: 15
---

# CLI الخاص بـ Gateway

Gateway هو خادم WebSocket الخاص بـ OpenClaw (القنوات، والعقد، والجلسات، وhooks).

توجد الأوامر الفرعية في هذه الصفحة تحت `openclaw gateway …`.

المستندات ذات الصلة:

- [/gateway/bonjour](/gateway/bonjour)
- [/gateway/discovery](/gateway/discovery)
- [/gateway/configuration](/gateway/configuration)

## تشغيل Gateway

شغّل عملية Gateway محلية:

```bash
openclaw gateway
```

الاسم المستعار للتشغيل في الواجهة الأمامية:

```bash
openclaw gateway run
```

ملاحظات:

- يرفض Gateway افتراضيًا بدء التشغيل ما لم يتم ضبط `gateway.mode=local` في `~/.openclaw/openclaw.json`. استخدم `--allow-unconfigured` للتشغيلات المخصصة/التطويرية.
- من المتوقع أن يكتب `openclaw onboard --mode local` و`openclaw setup` القيمة `gateway.mode=local`. وإذا كان الملف موجودًا لكن `gateway.mode` مفقودًا، فاعتبر ذلك تهيئة معطوبة أو تم العبث بها وأصلحها بدلًا من افتراض الوضع المحلي ضمنيًا.
- إذا كان الملف موجودًا وكانت `gateway.mode` مفقودة، فإن Gateway يتعامل مع ذلك على أنه تلف مريب في التهيئة ويرفض "افتراض local" نيابةً عنك.
- يتم حظر الربط خارج loopback بدون مصادقة (حاجز أمان).
- يؤدي `SIGUSR1` إلى إعادة تشغيل داخل العملية عند السماح بذلك (`commands.restart` مفعّل افتراضيًا؛ اضبط `commands.restart: false` لحظر إعادة التشغيل اليدوية، مع بقاء أداة gateway وتطبيق/تحديث التهيئة مسموحين).
- تؤدي معالجات `SIGINT`/`SIGTERM` إلى إيقاف عملية gateway، لكنها لا تستعيد أي حالة طرفية مخصصة. إذا كنت تغلف CLI بواجهة TUI أو إدخال raw-mode، فاستعد حالة الطرفية قبل الخروج.

### الخيارات

- `--port <port>`: منفذ WebSocket (تأتي القيمة الافتراضية من config/env؛ وعادةً `18789`).
- `--bind <loopback|lan|tailnet|auto|custom>`: وضع ربط المستمع.
- `--auth <token|password>`: تجاوز وضع المصادقة.
- `--token <token>`: تجاوز token (ويضبط أيضًا `OPENCLAW_GATEWAY_TOKEN` للعملية).
- `--password <password>`: تجاوز كلمة المرور. تحذير: قد تظهر كلمات المرور المضمنة في قوائم العمليات المحلية.
- `--password-file <path>`: قراءة كلمة مرور gateway من ملف.
- `--tailscale <off|serve|funnel>`: كشف Gateway عبر Tailscale.
- `--tailscale-reset-on-exit`: إعادة تعيين تهيئة Tailscale serve/funnel عند الإغلاق.
- `--allow-unconfigured`: السماح ببدء gateway دون `gateway.mode=local` في الإعداد. يتجاوز هذا حاجز بدء التشغيل للتمهيد المخصص/التطويري فقط؛ ولا يكتب ملف التهيئة أو يصلحه.
- `--dev`: إنشاء تهيئة + workspace للتطوير إذا كانت مفقودة (يتخطى `BOOTSTRAP.md`).
- `--reset`: إعادة تعيين تهيئة التطوير + بيانات الاعتماد + الجلسات + workspace (يتطلب `--dev`).
- `--force`: إنهاء أي مستمع موجود على المنفذ المحدد قبل البدء.
- `--verbose`: سجلات مفصلة.
- `--cli-backend-logs`: عرض سجلات الواجهة الخلفية لـ CLI فقط في الطرفية (مع تمكين stdout/stderr).
- `--claude-cli-logs`: اسم مستعار مهمل لـ `--cli-backend-logs`.
- `--ws-log <auto|full|compact>`: نمط سجل websocket (الافتراضي `auto`).
- `--compact`: اسم مستعار لـ `--ws-log compact`.
- `--raw-stream`: تسجيل أحداث تدفق النموذج الخام إلى jsonl.
- `--raw-stream-path <path>`: مسار jsonl للتدفق الخام.

## الاستعلام عن Gateway يعمل

تستخدم جميع أوامر الاستعلام WebSocket RPC.

أوضاع الإخراج:

- الافتراضي: قابل للقراءة البشرية (ملون في TTY).
- `--json`: JSON قابل للقراءة آليًا (من دون تنسيق/مؤشر دوران).
- `--no-color` (أو `NO_COLOR=1`): تعطيل ANSI مع إبقاء التخطيط البشري.

الخيارات المشتركة (حيثما كانت مدعومة):

- `--url <url>`: عنوان URL لـ Gateway WebSocket.
- `--token <token>`: token الخاص بـ Gateway.
- `--password <password>`: كلمة مرور Gateway.
- `--timeout <ms>`: المهلة/الميزانية (تختلف حسب الأمر).
- `--expect-final`: انتظار استجابة "نهائية" (استدعاءات الوكيل).

ملاحظة: عندما تضبط `--url`، لا يعود CLI إلى بيانات الاعتماد الموجودة في config أو environment.
مرّر `--token` أو `--password` صراحة. وغياب بيانات اعتماد صريحة يُعد خطأً.

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

### `gateway usage-cost`

اجلب ملخصات تكلفة الاستخدام من سجلات الجلسات.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --json
```

الخيارات:

- `--days <days>`: عدد الأيام المطلوب تضمينها (الافتراضي `30`).

### `gateway status`

يعرض `gateway status` خدمة Gateway ‏(launchd/systemd/schtasks) بالإضافة إلى فحص RPC اختياري.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

الخيارات:

- `--url <url>`: إضافة هدف فحص صريح. لا يزال يتم فحص configured remote + localhost.
- `--token <token>`: مصادقة token للفحص.
- `--password <password>`: مصادقة كلمة المرور للفحص.
- `--timeout <ms>`: مهلة الفحص (الافتراضي `10000`).
- `--no-probe`: تخطي فحص RPC (عرض الخدمة فقط).
- `--deep`: فحص خدمات على مستوى النظام أيضًا.
- `--require-rpc`: الخروج بقيمة غير صفرية عندما يفشل فحص RPC. لا يمكن دمجه مع `--no-probe`.

ملاحظات:

- يظل `gateway status` متاحًا للتشخيص حتى عندما تكون تهيئة CLI المحلية مفقودة أو غير صالحة.
- يقوم `gateway status` بحل SecretRefs الخاصة بالمصادقة المهيأة لمصادقة الفحص عند الإمكان.
- إذا كان SecretRef مطلوبًا للمصادقة وغير محلول في مسار هذا الأمر، فإن `gateway status --json` يبلّغ عن `rpc.authWarning` عندما يفشل اتصال/مصادقة probe؛ مرّر `--token`/`--password` صراحة أو أصلح مصدر السر أولًا.
- إذا نجح الفحص، يتم كتم تحذيرات auth-ref غير المحلولة لتجنب الإيجابيات الكاذبة.
- استخدم `--require-rpc` في البرامج النصية والأتمتة عندما لا يكفي وجود خدمة تستمع وتحتاج إلى أن تكون Gateway RPC نفسها سليمة.
- يضيف `--deep` فحصًا بأفضل جهد لعمليات تثبيت إضافية في launchd/systemd/schtasks. وعند اكتشاف عدة خدمات شبيهة بـ gateway، يطبع الإخراج البشري تلميحات للتنظيف ويحذر من أن معظم الإعدادات يجب أن تشغّل gateway واحدًا لكل جهاز.
- يتضمن الإخراج البشري مسار سجل الملف المحلول بالإضافة إلى لقطة من مسارات/صلاحية التهيئة بين CLI والخدمة للمساعدة في تشخيص انجراف profile أو state-dir.
- في عمليات تثبيت Linux systemd، تقرأ فحوصات انجراف مصادقة الخدمة قيم `Environment=` و`EnvironmentFile=` من الوحدة (بما في ذلك `%h` والمسارات المقتبسة والملفات المتعددة وملفات `-` الاختيارية).
- تحل فحوصات الانجراف `gateway.auth.token` SecretRefs باستخدام merged runtime env ‏(بيئة أمر الخدمة أولًا، ثم process env كاحتياطي).
- إذا لم تكن مصادقة token مفعّلة فعليًا (وضع `gateway.auth.mode` صريح بقيمة `password`/`none`/`trusted-proxy`، أو وضع غير مضبوط حيث يمكن أن تفوز كلمة المرور ولا يوجد مرشح token يمكنه الفوز)، فإن فحوصات انجراف token تتخطى حل token من التهيئة.

### `gateway probe`

الأمر `gateway probe` هو أمر "تصحيح كل شيء". وهو يفحص دائمًا:

- الـ remote gateway المهيأ لديك (إن وُجد)، و
- localhost ‏(loopback) **حتى إذا كان remote مهيأ**.

إذا مرّرت `--url`، يُضاف هذا الهدف الصريح قبلهما معًا. ويُسمّي الإخراج البشري
الأهداف كما يلي:

- `URL (explicit)`
- `Remote (configured)` أو `Remote (configured, inactive)`
- `Local loopback`

إذا أمكن الوصول إلى عدة Gateways، فسيطبعها جميعًا. وتُدعَم عدة Gateways عند استخدام profiles/mنافذ معزولة (مثل rescue bot)، لكن معظم عمليات التثبيت ما زالت تشغّل gateway واحدًا.

```bash
openclaw gateway probe
openclaw gateway probe --json
```

التفسير:

- `Reachable: yes` تعني أن هدفًا واحدًا على الأقل قبل اتصال WebSocket.
- `RPC: ok` تعني أن استدعاءات RPC التفصيلية (`health`/`status`/`system-presence`/`config.get`) نجحت أيضًا.
- `RPC: limited - missing scope: operator.read` تعني أن الاتصال نجح لكن RPC التفصيلي كان محدود النطاق. ويتم الإبلاغ عن هذا على أنه وصول **متدهور**، وليس فشلًا كاملًا.
- تكون قيمة الخروج غير صفرية فقط عندما لا يكون أي هدف تم فحصه قابلًا للوصول.

ملاحظات JSON ‏(`--json`):

- المستوى الأعلى:
  - `ok`: يمكن الوصول إلى هدف واحد على الأقل.
  - `degraded`: كان لدى هدف واحد على الأقل RPC تفصيلي محدود النطاق.
  - `primaryTargetId`: أفضل هدف يُعامل على أنه الفائز النشط بهذا الترتيب: URL صريح، نفق SSH، configured remote، ثم Local loopback.
  - `warnings[]`: سجلات تحذير بأفضل جهد تحتوي على `code` و`message` و`targetIds` الاختيارية.
  - `network`: تلميحات URL لـ Local loopback/tailnet مشتقة من التهيئة الحالية وشبكات المضيف.
  - `discovery.timeoutMs` و`discovery.count`: ميزانية/عدد نتائج الاكتشاف الفعليان المستخدمان في تمريرة الفحص هذه.
- لكل هدف (`targets[].connect`):
  - `ok`: إمكانية الوصول بعد الاتصال + تصنيف التدهور.
  - `rpcOk`: نجاح RPC التفصيلي الكامل.
  - `scopeLimited`: فشل RPC التفصيلي بسبب غياب نطاق operator.

رموز التحذير الشائعة:

- `ssh_tunnel_failed`: فشل إعداد نفق SSH؛ وعاد الأمر إلى الفحوصات المباشرة.
- `multiple_gateways`: أمكن الوصول إلى أكثر من هدف واحد؛ وهذا غير معتاد إلا إذا كنت تشغّل عمدًا profiles معزولة، مثل rescue bot.
- `auth_secretref_unresolved`: تعذر حل SecretRef مصادقة مهيأ لهدف فاشل.
- `probe_scope_limited`: نجح اتصال WebSocket، لكن RPC التفصيلي كان محدودًا بسبب غياب `operator.read`.

#### Remote عبر SSH ‏(مماثل لتطبيق Mac)

يستخدم وضع "Remote over SSH" في تطبيق macOS إعادة توجيه منفذ محلية بحيث يصبح الـ remote gateway (الذي قد يكون مرتبطًا بـ loopback فقط) قابلاً للوصول على `ws://127.0.0.1:<port>`.

المكافئ في CLI:

```bash
openclaw gateway probe --ssh user@gateway-host
```

الخيارات:

- `--ssh <target>`: ‏`user@host` أو `user@host:port` (يكون المنفذ افتراضيًا `22`).
- `--ssh-identity <path>`: ملف الهوية.
- `--ssh-auto`: اختيار أول مضيف gateway مكتشف كهدف SSH من نقطة نهاية
  الاكتشاف المحلولة (`local.` بالإضافة إلى wide-area domain المهيأ، إن وجد). ويتم تجاهل
  التلميحات المعتمدة على TXT فقط.

التهيئة (اختيارية، وتُستخدم كقيم افتراضية):

- `gateway.remote.sshTarget`
- `gateway.remote.sshIdentity`

### `gateway call <method>`

مساعد RPC منخفض المستوى.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"sinceMs": 60000}'
```

الخيارات:

- `--params <json>`: سلسلة كائن JSON للمعلمات (الافتراضي `{}`)
- `--url <url>`
- `--token <token>`
- `--password <password>`
- `--timeout <ms>`
- `--expect-final`
- `--json`

ملاحظات:

- يجب أن تكون `--params` قيمة JSON صالحة.
- يُستخدم `--expect-final` بشكل أساسي مع RPCs على نمط الوكيل التي تبث أحداثًا وسيطة قبل payload نهائي.

## إدارة خدمة Gateway

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

خيارات الأوامر:

- `gateway status`: ‏`--url`، ‏`--token`، ‏`--password`، ‏`--timeout`، ‏`--no-probe`، ‏`--require-rpc`، ‏`--deep`، ‏`--json`
- `gateway install`: ‏`--port`، ‏`--runtime <node|bun>`، ‏`--token`، ‏`--force`، ‏`--json`
- `gateway uninstall|start|stop|restart`: ‏`--json`

ملاحظات:

- يدعم `gateway install` الخيارات `--port` و`--runtime` و`--token` و`--force` و`--json`.
- عندما تتطلب مصادقة token وجود token وكان `gateway.auth.token` مُدارًا عبر SecretRef، فإن `gateway install` يتحقق من أن SecretRef قابل للحل لكنه لا يخزن token المحلول في بيانات تعريف بيئة الخدمة.
- إذا كانت مصادقة token تتطلب token وكان SecretRef الخاص بالـ token المهيأ غير محلول، فإن التثبيت يفشل بشكل مغلق بدلًا من حفظ نص صريح احتياطي.
- بالنسبة إلى مصادقة كلمة المرور في `gateway run`، ففضّل `OPENCLAW_GATEWAY_PASSWORD` أو `--password-file` أو `gateway.auth.password` المدعوم بـ SecretRef بدلًا من `--password` المضمن.
- في وضع المصادقة المستنتج، لا يخفف `OPENCLAW_GATEWAY_PASSWORD` الموجود في shell فقط متطلبات token الخاصة بالتثبيت؛ استخدم تهيئة دائمة (`gateway.auth.password` أو config `env`) عند تثبيت خدمة مُدارة.
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مهيأين وكانت `gateway.auth.mode` غير مضبوطة، فسيتم حظر التثبيت حتى يُضبط الوضع صراحة.
- تقبل أوامر دورة الحياة `--json` للاستخدام في البرامج النصية.

## اكتشاف Gateways ‏(Bonjour)

يقوم `gateway discover` بفحص beacons الخاصة بـ Gateway ‏(`_openclaw-gw._tcp`).

- Multicast DNS-SD: ‏`local.`
- Unicast DNS-SD ‏(Wide-Area Bonjour): اختر domain (مثال: `openclaw.internal.`) وأعد إعداد split DNS + خادم DNS؛ راجع [/gateway/bonjour](/gateway/bonjour)

تعلن فقط الـ Gateways التي يكون فيها اكتشاف Bonjour مفعّلًا (افتراضيًا) عن الـ beacon.

تتضمن سجلات الاكتشاف Wide-Area ‏(TXT):

- `role` (تلميح دور gateway)
- `transport` (تلميح النقل، مثل `gateway`)
- `gatewayPort` (منفذ WebSocket، وعادةً `18789`)
- `sshPort` (اختياري؛ تستخدم العملاء افتراضيًا أهداف SSH على `22` عند غيابه)
- `tailnetDns` (اسم مضيف MagicDNS، عند توفره)
- `gatewayTls` / `gatewayTlsSha256` (تمكين TLS + بصمة الشهادة)
- `cliPath` (تلميح تثبيت remote مكتوب إلى wide-area zone)

### `gateway discover`

```bash
openclaw gateway discover
```

الخيارات:

- `--timeout <ms>`: المهلة لكل أمر (browse/resolve)؛ الافتراضي `2000`.
- `--json`: إخراج قابل للقراءة آليًا (ويعطل أيضًا التنسيق/مؤشر الدوران).

أمثلة:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

ملاحظات:

- يفحص CLI كلاً من `local.` وwide-area domain المهيأ عندما يكون أحدها مفعّلًا.
- يتم اشتقاق `wsUrl` في إخراج JSON من نقطة نهاية الخدمة المحلولة، وليس من
  التلميحات المعتمدة على TXT فقط مثل `lanHost` أو `tailnetDns`.
- في mDNS الخاص بـ `local.`، لا يتم بث `sshPort` و`cliPath` إلا عندما
  تكون `discovery.mdns.mode` هي `full`. أما Wide-area DNS-SD فلا يزال يكتب `cliPath`؛
  ويبقى `sshPort` اختياريًا هناك أيضًا.
