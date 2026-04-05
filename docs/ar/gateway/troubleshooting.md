---
read_when:
    - وجّهك مركز استكشاف الأخطاء هنا لتشخيص أعمق
    - تحتاج إلى أقسام دليل تشغيل مستقرة حسب الأعراض مع أوامر دقيقة
summary: دليل استكشاف الأخطاء العميق لـ gateway والقنوات والأتمتة والعقد والمتصفح
title: استكشاف الأخطاء وإصلاحها
x-i18n:
    generated_at: "2026-04-05T12:45:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 028226726e6adc45ca61d41510a953c4e21a3e85f3082af9e8085745c6ac3ec1
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# استكشاف أخطاء Gateway

هذه الصفحة هي دليل التشغيل العميق.
ابدأ من [/help/troubleshooting](/help/troubleshooting) إذا كنت تريد تدفق الفرز السريع أولًا.

## سلم الأوامر

شغّل هذه الأوامر أولًا، بهذا الترتيب:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

الإشارات الصحية المتوقعة:

- يعرض `openclaw gateway status` القيمتين `Runtime: running` و`RPC probe: ok`.
- يبلّغ `openclaw doctor` عن عدم وجود مشكلات حظر في التكوين/الخدمة.
- يعرض `openclaw channels status --probe` حالة النقل المباشرة لكل حساب،
  وعند الدعم، نتائج probe/audit مثل `works` أو `audit ok`.

## Anthropic 429 extra usage required for long context

استخدم هذا عندما تتضمن السجلات/الأخطاء:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

ابحث عن:

- أن نموذج Anthropic Opus/Sonnet المحدد يحتوي على `params.context1m: true`.
- أن بيانات اعتماد Anthropic الحالية غير مؤهلة لاستخدام السياق الطويل.
- أن الطلبات تفشل فقط على الجلسات الطويلة/تشغيلات النماذج التي تحتاج إلى مسار 1M beta.

خيارات الإصلاح:

1. عطّل `context1m` لذلك النموذج للرجوع إلى نافذة السياق العادية.
2. استخدم مفتاح Anthropic API مع الفوترة، أو فعّل Anthropic Extra Usage على حساب Anthropic OAuth/الاشتراك.
3. اضبط نماذج fallback بحيث تستمر التشغيلات عندما تُرفض طلبات Anthropic ذات السياق الطويل.

ذو صلة:

- [/providers/anthropic](/providers/anthropic)
- [/reference/token-use](/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## لا توجد ردود

إذا كانت القنوات تعمل لكن لا شيء يجيب، فتحقق من التوجيه والسياسة قبل إعادة توصيل أي شيء.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

ابحث عن:

- pairing معلّق لمرسلي الرسائل المباشرة.
- تقييد الإشارة في المجموعات (`requireMention`, `mentionPatterns`).
- عدم تطابق قوائم السماح للقناة/المجموعة.

التوقيعات الشائعة:

- `drop guild message (mention required` → تم تجاهل رسالة المجموعة حتى حدوث إشارة.
- `pairing request` → يحتاج المرسل إلى موافقة.
- `blocked` / `allowlist` → تمت تصفية المرسل/القناة بواسطة السياسة.

ذو صلة:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/pairing](/channels/pairing)
- [/channels/groups](/channels/groups)

## اتصال dashboard control ui

عندما لا يتمكن dashboard/control UI من الاتصال، تحقّق من عنوان URL ووضع المصادقة وافتراضات السياق الآمن.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

ابحث عن:

- عنوان probe الصحيح وعنوان dashboard الصحيح.
- عدم تطابق وضع المصادقة/الرمز بين العميل وgateway.
- استخدام HTTP عندما تكون هوية الجهاز مطلوبة.

التوقيعات الشائعة:

- `device identity required` → سياق غير آمن أو مصادقة جهاز مفقودة.
- `origin not allowed` → قيمة `Origin` في المتصفح ليست ضمن `gateway.controlUi.allowedOrigins`
  (أو أنك تتصل من `Origin` متصفح غير loopback بدون
  قائمة سماح صريحة).
- `device nonce required` / `device nonce mismatch` → العميل لا يُكمل
  تدفق مصادقة الجهاز القائم على التحدي (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → وقّع العميل حمولة خاطئة
  (أو بطابع زمني قديم) لعملية المصافحة الحالية.
- `AUTH_TOKEN_MISMATCH` مع `canRetryWithDeviceToken=true` → يمكن للعميل إجراء إعادة محاولة موثوقة واحدة باستخدام رمز جهاز مخزن مؤقتًا.
- تعيد إعادة المحاولة باستخدام الرمز المخزن مؤقتًا استخدام مجموعة النطاقات المخزنة مع
  رمز الجهاز المقترن. أما المستدعون الصريحون لـ `deviceToken` / `scopes` فيحتفظون
  بمجموعة النطاقات المطلوبة الخاصة بهم.
- خارج مسار إعادة المحاولة هذا، تكون أولوية مصادقة الاتصال هي
  الرمز/كلمة المرور المشتركة الصريحة أولًا، ثم `deviceToken` الصريح، ثم رمز الجهاز المخزن، ثم رمز bootstrap.
- في مسار Control UI غير المتزامن عبر Tailscale Serve، يتم تسلسل المحاولات الفاشلة للزوج `{scope, ip}` نفسه قبل أن يسجل المحدِّد الفشل. لذلك قد تُظهر محاولتا إعادة سيئتان متزامنتان من العميل نفسه رسالة `retry later` في المحاولة الثانية بدلًا من حالتي عدم تطابق عاديتين.
- `too many failed authentication attempts (retry later)` من عميل loopback ذي أصل متصفح → تُقفل الإخفاقات المتكررة من `Origin` المطبع نفسه مؤقتًا؛ ويستخدم أصل localhost آخر سلة منفصلة.
- تكرار `unauthorized` بعد إعادة المحاولة تلك → انحراف في الرمز المشترك/رمز الجهاز؛ حدّث تكوين الرمز وأعد الموافقة على رمز الجهاز/دوّره إذا لزم.
- `gateway connect failed:` → هدف مضيف/منفذ/عنوان URL خاطئ.

### خريطة سريعة لتفاصيل رموز المصادقة

استخدم `error.details.code` من استجابة `connect` الفاشلة لاختيار الإجراء التالي:

| رمز التفاصيل                | المعنى                                                   | الإجراء الموصى به                                                                                                                                                                                                                                                                      |
| --------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`        | لم يرسل العميل رمزًا مشتركًا مطلوبًا.                  | الصق/اضبط الرمز في العميل ثم أعد المحاولة. بالنسبة إلى مسارات dashboard: شغّل `openclaw config get gateway.auth.token` ثم الصقه في إعدادات Control UI.                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`       | لم يطابق الرمز المشترك رمز مصادقة gateway.             | إذا كانت `canRetryWithDeviceToken=true`، اسمح بإعادة محاولة موثوقة واحدة. تعيد محاولات الرمز المخزن مؤقتًا استخدام النطاقات الموافق عليها المخزنة؛ أما مستدعو `deviceToken` / `scopes` الصريحون فيحتفظون بالنطاقات المطلوبة. إذا استمر الفشل، شغّل [قائمة التحقق من استعادة انحراف الرمز](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | رمز كل جهاز المخزن مؤقتًا قديم أو تم إلغاؤه.          | دوّر/أعد الموافقة على رمز الجهاز باستخدام [CLI الخاص بـ devices](/cli/devices)، ثم أعد الاتصال.                                                                                                                                                                                      |
| `PAIRING_REQUIRED`          | هوية الجهاز معروفة لكنها غير موافق عليها لهذا الدور.   | وافق على الطلب المعلّق: `openclaw devices list` ثم `openclaw devices approve <requestId>`.                                                                                                                                                                                            |

فحص ترحيل device auth v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

إذا أظهرت السجلات أخطاء nonce/signature، فحدّث العميل المتصل وتأكد من أنه:

1. ينتظر `connect.challenge`
2. يوقّع الحمولة المرتبطة بالتحدي
3. يرسل `connect.params.device.nonce` مع nonce التحدي نفسها

إذا تم رفض `openclaw devices rotate` / `revoke` / `remove` بشكل غير متوقع:

- يمكن لجلسات الرموز الخاصة بالأجهزة المقترنة إدارة **أجهزتها الخاصة فقط** ما لم يكن
  لدى المستدعي أيضًا `operator.admin`
- لا يستطيع `openclaw devices rotate --scope ...` طلب نطاقات operator إلا إذا كانت
  جلسة المستدعي تملكها بالفعل

ذو صلة:

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/gateway/configuration) (أوضاع مصادقة gateway)
- [/gateway/trusted-proxy-auth](/gateway/trusted-proxy-auth)
- [/gateway/remote](/gateway/remote)
- [/cli/devices](/cli/devices)

## خدمة Gateway لا تعمل

استخدم هذا عندما تكون الخدمة مثبتة لكن العملية لا تبقى قيد التشغيل.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # also scan system-level services
```

ابحث عن:

- `Runtime: stopped` مع تلميحات خروج.
- عدم تطابق تكوين الخدمة (`Config (cli)` مقابل `Config (service)`).
- تعارضات المنفذ/المستمع.
- تثبيتات launchd/systemd/schtasks إضافية عند استخدام `--deep`.
- تلميحات تنظيف `Other gateway-like services detected (best effort)`.

التوقيعات الشائعة:

- `Gateway start blocked: set gateway.mode=local` أو `existing config is missing gateway.mode` → لم يتم تمكين وضع gateway المحلي، أو تم العبث بملف التكوين وفقد `gateway.mode`. الإصلاح: اضبط `gateway.mode="local"` في التكوين، أو أعد تشغيل `openclaw onboard --mode local` / `openclaw setup` لإعادة ختم تكوين الوضع المحلي المتوقع. إذا كنت تشغّل OpenClaw عبر Podman، فمسار التكوين الافتراضي هو `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → ربط غير loopback من دون مسار مصادقة gateway صالح (رمز/كلمة مرور، أو trusted-proxy عند تكوينه).
- `another gateway instance is already listening` / `EADDRINUSE` → تعارض منفذ.
- `Other gateway-like services detected (best effort)` → توجد وحدات launchd/systemd/schtasks قديمة أو متوازية. يجب أن تحتفظ معظم الإعدادات بـ gateway واحدة لكل جهاز؛ وإذا كنت تحتاج بالفعل إلى أكثر من واحدة، فاعزل المنافذ + التكوين/الحالة/مساحة العمل. راجع [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host).

ذو صلة:

- [/gateway/background-process](/gateway/background-process)
- [/gateway/configuration](/gateway/configuration)
- [/gateway/doctor](/gateway/doctor)

## تحذيرات Gateway probe

استخدم هذا عندما يصل `openclaw gateway probe` إلى شيء ما، لكنه ما يزال يطبع كتلة تحذير.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

ابحث عن:

- `warnings[].code` و`primaryTargetId` في إخراج JSON.
- ما إذا كان التحذير يتعلق برجوع SSH الاحتياطي، أو تعدد gateways، أو النطاقات المفقودة، أو مراجع المصادقة غير المحلولة.

التوقيعات الشائعة:

- `SSH tunnel failed to start; falling back to direct probes.` → فشل إعداد SSH، لكن الأمر ما زال يحاول الأهداف المباشرة المكوّنة/loopback.
- `multiple reachable gateways detected` → استجاب أكثر من هدف. يعني هذا عادةً إعدادًا مقصودًا متعدد gateways أو مستمعين قدامى/مكررين.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → نجح الاتصال، لكن تفاصيل RPC محدودة بالنطاق؛ قم بربط هوية الجهاز أو استخدم بيانات اعتماد تملك `operator.read`.
- نص تحذير SecretRef غير المحلولة لـ `gateway.auth.*` / `gateway.remote.*` → لم تكن مادة المصادقة متاحة في مسار هذا الأمر للهدف الفاشل.

ذو صلة:

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host)
- [/gateway/remote](/gateway/remote)

## القناة متصلة لكن الرسائل لا تتدفق

إذا كانت حالة القناة متصلة لكن تدفق الرسائل ميتًا، فركز على السياسة والأذونات وقواعد التسليم الخاصة بالقناة.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

ابحث عن:

- سياسة الرسائل المباشرة (`pairing`, `allowlist`, `open`, `disabled`).
- قائمة سماح المجموعات ومتطلبات الإشارة.
- أذونات/نطاقات API الخاصة بالقناة المفقودة.

التوقيعات الشائعة:

- `mention required` → تم تجاهل الرسالة بسبب سياسة الإشارة في المجموعة.
- آثار `pairing` / الموافقة المعلّقة → المرسل غير موافق عليه.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → مشكلة في مصادقة/أذونات القناة.

ذو صلة:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/whatsapp](/channels/whatsapp)
- [/channels/telegram](/channels/telegram)
- [/channels/discord](/channels/discord)

## تسليم Cron وheartbeat

إذا لم تعمل cron أو heartbeat أو لم تسلّما، فتحقق من حالة المجدول أولًا، ثم من هدف التسليم.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

ابحث عن:

- أن cron مفعلة وأن wake التالية موجودة.
- حالة سجل تشغيل المهمة (`ok`, `skipped`, `error`).
- أسباب تخطي heartbeat (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

التوقيعات الشائعة:

- `cron: scheduler disabled; jobs will not run automatically` → cron معطلة.
- `cron: timer tick failed` → فشل tick للمجدول؛ تحقق من أخطاء الملف/السجل/وقت التشغيل.
- `heartbeat skipped` مع `reason=quiet-hours` → خارج نافذة الساعات النشطة.
- `heartbeat skipped` مع `reason=empty-heartbeat-file` → يوجد `HEARTBEAT.md` لكنه يحتوي فقط على أسطر فارغة / عناوين markdown، لذلك يتخطى OpenClaw استدعاء النموذج.
- `heartbeat skipped` مع `reason=no-tasks-due` → يحتوي `HEARTBEAT.md` على كتلة `tasks:`، لكن لا توجد أي مهام مستحقة في هذه tick.
- `heartbeat: unknown accountId` → معرّف حساب غير صالح لهدف تسليم heartbeat.
- `heartbeat skipped` مع `reason=dm-blocked` → تم حل هدف heartbeat إلى وجهة بنمط الرسائل المباشرة بينما تم ضبط `agents.defaults.heartbeat.directPolicy` (أو التجاوز لكل وكيل) على `block`.

ذو صلة:

- [/automation/cron-jobs#troubleshooting](/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/automation/cron-jobs)
- [/gateway/heartbeat](/gateway/heartbeat)

## فشل أداة عقدة مقترنة

إذا كانت العقدة مقترنة لكن الأدوات تفشل، فاعزل حالة الواجهة الأمامية، والأذونات، والموافقة.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

ابحث عن:

- أن العقدة متصلة بالإمكانات المتوقعة.
- منح أذونات نظام التشغيل للكاميرا/الميكروفون/الموقع/الشاشة.
- حالة موافقات exec وقائمة السماح.

التوقيعات الشائعة:

- `NODE_BACKGROUND_UNAVAILABLE` → يجب أن يكون تطبيق العقدة في الواجهة الأمامية.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → إذن نظام تشغيل مفقود.
- `SYSTEM_RUN_DENIED: approval required` → موافقة exec معلّقة.
- `SYSTEM_RUN_DENIED: allowlist miss` → تم حظر الأمر بواسطة قائمة السماح.

ذو صلة:

- [/nodes/troubleshooting](/nodes/troubleshooting)
- [/nodes/index](/nodes/index)
- [/tools/exec-approvals](/tools/exec-approvals)

## فشل أداة Browser

استخدم هذا عندما تفشل إجراءات أداة Browser حتى مع سلامة gateway نفسها.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

ابحث عن:

- ما إذا كان `plugins.allow` مضبوطًا ويتضمن `browser`.
- صحة مسار الملف التنفيذي للمتصفح.
- إمكانية الوصول إلى ملف CDP الشخصي.
- توفر Chrome المحلي لملفات `existing-session` / `user`.

التوقيعات الشائعة:

- `unknown command "browser"` أو `unknown command 'browser'` → تم استبعاد plugin المتصفح المضمّنة بواسطة `plugins.allow`.
- غياب/تعذر توفر أداة browser مع `browser.enabled=true` → يستبعد `plugins.allow` القيمة `browser`، لذلك لم يتم تحميل plugin مطلقًا.
- `Failed to start Chrome CDP on port` → فشلت عملية المتصفح في التشغيل.
- `browser.executablePath not found` → المسار المكوَّن غير صالح.
- `browser.cdpUrl must be http(s) or ws(s)` → يستخدم عنوان CDP URL المكوّن مخططًا غير مدعوم مثل `file:` أو `ftp:`.
- `browser.cdpUrl has invalid port` → يحتوي عنوان CDP URL المكوّن على منفذ سيئ أو خارج النطاق.
- `No Chrome tabs found for profile="user"` → لا يحتوي ملف الإرفاق في Chrome MCP على علامات تبويب Chrome محلية مفتوحة.
- `Remote CDP for profile "<name>" is not reachable` → لا يمكن الوصول إلى نقطة نهاية CDP البعيدة المكوّنة من مضيف gateway.
- `Browser attachOnly is enabled ... not reachable` أو `Browser attachOnly is enabled and CDP websocket ... is not reachable` → لا يحتوي ملف attach-only على هدف يمكن الوصول إليه، أو أن نقطة نهاية HTTP استجابت لكن تعذّر مع ذلك فتح WebSocket الخاصة بـ CDP.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → يفتقر تثبيت gateway الحالي إلى حزمة Playwright الكاملة؛ لا تزال لقطات ARIA ولقطات الشاشة الأساسية للصفحة ممكنة، لكن التنقل ولقطات AI ولقطات العناصر عبر محدد CSS وتصدير PDF تبقى غير متاحة.
- `fullPage is not supported for element screenshots` → خلط طلب لقطة الشاشة بين `--full-page` و`--ref` أو `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → يجب أن تستخدم استدعاءات لقطات Chrome MCP / `existing-session` التقاط الصفحة أو `--ref` من snapshot، وليس CSS `--element`.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → تحتاج hooks رفع Chrome MCP إلى مراجع snapshot، وليس محددات CSS.
- `existing-session file uploads currently support one file at a time.` → أرسل عملية رفع واحدة لكل استدعاء في ملفات Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` → لا تدعم hooks الحوارات في ملفات Chrome MCP تجاوزات المهلة.
- `response body is not supported for existing-session profiles yet.` → لا تزال `responsebody` تتطلب متصفحًا مُدارًا أو ملف CDP خامًا.
- تجاوزات viewport / الوضع الداكن / locale / عدم الاتصال القديمة على ملفات attach-only أو CDP البعيدة → شغّل `openclaw browser stop --browser-profile <name>` لإغلاق جلسة التحكم النشطة وتحرير حالة محاكاة Playwright/CDP من دون إعادة تشغيل gateway بالكامل.

ذو صلة:

- [/tools/browser-linux-troubleshooting](/tools/browser-linux-troubleshooting)
- [/tools/browser](/tools/browser)

## إذا قمت بالترقية وانكسر شيء فجأة

يكون معظم العطل بعد الترقية ناتجًا عن انحراف التكوين أو فرض إعدادات افتراضية أشد صرامة الآن.

### 1) تغيّر سلوك المصادقة وتجاوز URL

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

ما الذي يجب التحقق منه:

- إذا كانت `gateway.mode=remote`، فقد تستهدف استدعاءات CLI هدفًا بعيدًا بينما تكون خدمتك المحلية سليمة.
- لا تعود استدعاءات `--url` الصريحة إلى بيانات الاعتماد المخزنة.

التوقيعات الشائعة:

- `gateway connect failed:` → هدف URL خاطئ.
- `unauthorized` → يمكن الوصول إلى نقطة النهاية لكن المصادقة خاطئة.

### 2) أصبحت ضوابط bind والمصادقة أشد صرامة

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

ما الذي يجب التحقق منه:

- تحتاج الروابط غير loopback ‏(`lan`, `tailnet`, `custom`) إلى مسار مصادقة gateway صالح: مصادقة رمز/كلمة مرور مشتركة، أو نشر `trusted-proxy` غير loopback ومكوّن بشكل صحيح.
- المفاتيح القديمة مثل `gateway.token` لا تحل محل `gateway.auth.token`.

التوقيعات الشائعة:

- `refusing to bind gateway ... without auth` → ربط غير loopback من دون مسار مصادقة gateway صالح.
- `RPC probe: failed` بينما وقت التشغيل يعمل → gateway حية لكن لا يمكن الوصول إليها باستخدام المصادقة/URL الحالية.

### 3) تغيّرت حالة pairing وهوية الجهاز

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

ما الذي يجب التحقق منه:

- موافقات الأجهزة المعلّقة لـ dashboard/nodes.
- موافقات pairing المعلّقة للرسائل المباشرة بعد تغييرات السياسة أو الهوية.

التوقيعات الشائعة:

- `device identity required` → لم تتحقق مصادقة الجهاز.
- `pairing required` → يجب الموافقة على المرسل/الجهاز.

إذا ظل تكوين الخدمة ووقت التشغيل مختلفين بعد الفحوصات، فأعد تثبيت بيانات تعريف الخدمة من دليل profile/state نفسه:

```bash
openclaw gateway install --force
openclaw gateway restart
```

ذو صلة:

- [/gateway/pairing](/gateway/pairing)
- [/gateway/authentication](/gateway/authentication)
- [/gateway/background-process](/gateway/background-process)
