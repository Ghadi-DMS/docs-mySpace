---
read_when:
    - أحالَك مركز استكشاف الأخطاء وإصلاحها إلى هنا للتشخيص الأعمق
    - تحتاج إلى أقسام دليل تشغيل مستقرة قائمة على الأعراض مع أوامر دقيقة
summary: دليل متعمق لاستكشاف الأخطاء وإصلاحها للبوابة والقنوات والأتمتة والعُقد والمتصفح
title: استكشاف الأخطاء وإصلاحها
x-i18n:
    generated_at: "2026-04-08T02:16:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 02c9537845248db0c9d315bf581338a93215fe6fe3688ed96c7105cbb19fe6ba
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# استكشاف أخطاء البوابة وإصلاحها

هذه الصفحة هي دليل التشغيل المتعمق.
ابدأ من [/help/troubleshooting](/ar/help/troubleshooting) إذا كنت تريد تدفق الفرز السريع أولًا.

## تسلسل الأوامر

شغّل هذه الأوامر أولًا، بهذا الترتيب:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

الإشارات السليمة المتوقعة:

- يعرض `openclaw gateway status` القيمة `Runtime: running` و `RPC probe: ok`.
- يُبلغ `openclaw doctor` عن عدم وجود مشكلات حظر في الإعدادات/الخدمة.
- يعرض `openclaw channels status --probe` حالة النقل المباشرة لكل حساب،
  وحيثما كان ذلك مدعومًا، نتائج الفحص/التدقيق مثل `works` أو `audit ok`.

## خطأ Anthropic 429: استخدام إضافي مطلوب للسياق الطويل

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
- أن الطلبات تفشل فقط في الجلسات الطويلة/تشغيلات النماذج التي تحتاج إلى مسار 1M التجريبي.

خيارات الإصلاح:

1. عطّل `context1m` لذلك النموذج للرجوع إلى نافذة السياق العادية.
2. استخدم بيانات اعتماد Anthropic مؤهلة لطلبات السياق الطويل، أو انتقل إلى مفتاح Anthropic API.
3. اضبط نماذج بديلة حتى تستمر التشغيلات عندما تُرفض طلبات Anthropic ذات السياق الطويل.

ذو صلة:

- [/providers/anthropic](/ar/providers/anthropic)
- [/reference/token-use](/ar/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/ar/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## الواجهة الخلفية المحلية المتوافقة مع OpenAI تجتاز الفحوصات المباشرة لكن تشغيلات الوكيل تفشل

استخدم هذا عندما:

- ينجح `curl ... /v1/models`
- تنجح استدعاءات `/v1/chat/completions` المباشرة الصغيرة
- تفشل تشغيلات نماذج OpenClaw فقط في أدوار الوكيل العادية

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

ابحث عن:

- أن الاستدعاءات الصغيرة المباشرة تنجح، لكن تشغيلات OpenClaw تفشل فقط مع الموجّهات الأكبر
- أخطاء في الواجهة الخلفية تشير إلى أن `messages[].content` يتوقع سلسلة نصية
- أعطال الواجهة الخلفية التي تظهر فقط مع أعداد أكبر من رموز الموجّه أو موجّهات
  وقت تشغيل الوكيل الكاملة

الأنماط الشائعة:

- `messages[...].content: invalid type: sequence, expected a string` ← الواجهة الخلفية
  ترفض أجزاء محتوى Chat Completions المهيكلة. الإصلاح: اضبط
  `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- تنجح الطلبات المباشرة الصغيرة، لكن تشغيلات وكيل OpenClaw تفشل مع أعطال
  الواجهة الخلفية/النموذج (مثلًا Gemma على بعض إصدارات `inferrs`) ← من
  المرجح أن نقل OpenClaw صحيح بالفعل؛ الواجهة الخلفية هي التي تفشل مع شكل
  موجّه وقت تشغيل الوكيل الأكبر.
- تتراجع حالات الفشل بعد تعطيل الأدوات لكنها لا تختفي ← كانت مخططات الأدوات
  جزءًا من الضغط، لكن المشكلة المتبقية لا تزال في سعة النموذج/الخادم upstream
  أو في خطأ بالواجهة الخلفية.

خيارات الإصلاح:

1. اضبط `compat.requiresStringContent: true` لواجهات Chat Completions الخلفية التي تقبل السلاسل النصية فقط.
2. اضبط `compat.supportsTools: false` للنماذج/الواجهات الخلفية التي لا تستطيع التعامل
   مع سطح مخطط أدوات OpenClaw بشكل موثوق.
3. خفّض ضغط الموجّه حيثما أمكن: تهيئة أولية أصغر لمساحة العمل، وسجل جلسة أقصر،
   أو نموذج محلي أخف، أو واجهة خلفية ذات دعم أقوى للسياق الطويل.
4. إذا استمرت الطلبات المباشرة الصغيرة بالنجاح بينما ما زالت أدوار وكيل OpenClaw تتعطل
   داخل الواجهة الخلفية، فاعتبر ذلك قيدًا في الخادم/النموذج upstream وقدّم
   بلاغ إعادة إنتاج هناك مع شكل الحمولة المقبول.

ذو صلة:

- [/gateway/local-models](/ar/gateway/local-models)
- [/gateway/configuration#models](/ar/gateway/configuration#models)
- [/gateway/configuration-reference#openai-compatible-endpoints](/ar/gateway/configuration-reference#openai-compatible-endpoints)

## لا توجد ردود

إذا كانت القنوات تعمل لكن لا يوجد أي رد، فتحقق من التوجيه والسياسة قبل إعادة توصيل أي شيء.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

ابحث عن:

- اقتران معلّق لمرسلي الرسائل المباشرة.
- تقييد الإشارة في المجموعات (`requireMention`، `mentionPatterns`).
- عدم تطابق قائمة السماح للقناة/المجموعة.

الأنماط الشائعة:

- `drop guild message (mention required` ← تم تجاهل رسالة المجموعة حتى حدوث إشارة.
- `pairing request` ← يحتاج المرسل إلى موافقة.
- `blocked` / `allowlist` ← تمت تصفية المرسل/القناة بواسطة السياسة.

ذو صلة:

- [/channels/troubleshooting](/ar/channels/troubleshooting)
- [/channels/pairing](/ar/channels/pairing)
- [/channels/groups](/ar/channels/groups)

## اتصال واجهة dashboard/control UI

عندما لا يتصل dashboard/control UI، تحقّق من عنوان URL ووضع المصادقة وافتراضات السياق الآمن.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

ابحث عن:

- عنوان URL الصحيح للفحص ولوحة التحكم.
- عدم تطابق وضع/رمز المصادقة بين العميل والبوابة.
- استخدام HTTP حيث تكون هوية الجهاز مطلوبة.

الأنماط الشائعة:

- `device identity required` ← سياق غير آمن أو مصادقة جهاز مفقودة.
- `origin not allowed` ← قيمة `Origin` في المتصفح غير موجودة في `gateway.controlUi.allowedOrigins`
  (أو أنك تتصل من مصدر متصفح غير loopback من دون
  قائمة سماح صريحة).
- `device nonce required` / `device nonce mismatch` ← العميل لا يُكمل
  تدفق مصادقة الجهاز المعتمد على التحدي (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` ← وقّع العميل الحمولة الخطأ
  (أو بطابع زمني قديم) للمصافحة الحالية.
- `AUTH_TOKEN_MISMATCH` مع `canRetryWithDeviceToken=true` ← يمكن للعميل تنفيذ
  إعادة محاولة موثوقة واحدة باستخدام رمز الجهاز المخزن مؤقتًا.
- تعيد إعادة المحاولة باستخدام الرمز المخزن مؤقتًا استخدام مجموعة النطاقات المخزنة مع
  رمز الجهاز الموافق عليه. أما مستدعيات `deviceToken` / `scopes` الصريحة فتحافظ على
  مجموعة النطاقات المطلوبة بدلًا من ذلك.
- خارج مسار إعادة المحاولة هذا، تكون أولوية مصادقة الاتصال: الرمز
  أو كلمة المرور المشتركة الصريحة أولًا، ثم `deviceToken` الصريح، ثم رمز الجهاز المخزن،
  ثم رمز التهيئة الأولية.
- في مسار Control UI غير المتزامن عبر Tailscale Serve، تتم
  سلسلة المحاولات الفاشلة لنفس `{scope, ip}` قبل أن يسجل المحدِّد الفشل.
  لذا قد تظهر في المحاولة الثانية من محاولتين خاطئتين متزامنتين من العميل نفسه رسالة `retry later`
  بدلًا من حالتي عدم تطابق عاديتين.
- `too many failed authentication attempts (retry later)` من عميل loopback ذي أصل متصفح
  ← تُحظر الإخفاقات المتكررة من `Origin` المُطبَّع نفسه مؤقتًا؛ بينما يستخدم أصل localhost آخر حاوية منفصلة.
- تكرار `unauthorized` بعد إعادة المحاولة تلك ← يوجد انجراف في الرمز المشترك/رمز الجهاز؛ حدّث إعدادات الرمز وأعِد الموافقة/دوّر رمز الجهاز عند الحاجة.
- `gateway connect failed:` ← المضيف/المنفذ/هدف عنوان URL غير صحيح.

### خريطة سريعة لرموز تفاصيل المصادقة

استخدم `error.details.code` من استجابة `connect` الفاشلة لاختيار الإجراء التالي:

| رمز التفاصيل                  | المعنى                                                  | الإجراء الموصى به                                                                                                                                                                                                                                                                        |
| ---------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | لم يرسل العميل رمزًا مشتركًا مطلوبًا.             | الصق/اضبط الرمز في العميل وأعد المحاولة. لمسارات dashboard: `openclaw config get gateway.auth.token` ثم الصقه في إعدادات Control UI.                                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | لم يطابق الرمز المشترك رمز مصادقة البوابة.           | إذا كانت `canRetryWithDeviceToken=true`، اسمح بإعادة محاولة موثوقة واحدة. تعيد محاولات الرمز المخزن مؤقتًا استخدام النطاقات الموافق عليها والمخزنة؛ أما مستدعيات `deviceToken` / `scopes` الصريحة فتحافظ على النطاقات المطلوبة. وإذا استمر الفشل، شغّل [قائمة التحقق من استرداد انجراف الرمز](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | رمز كل جهاز المخزن مؤقتًا قديم أو أُلغي.             | دوّر/أعِد الموافقة على رمز الجهاز باستخدام [CLI الأجهزة](/cli/devices)، ثم أعد الاتصال.                                                                                                                                                                                               |
| `PAIRING_REQUIRED`           | هوية الجهاز معروفة لكنها غير معتمدة لهذا الدور. | وافق على الطلب المعلّق: `openclaw devices list` ثم `openclaw devices approve <requestId>`.                                                                                                                                                                                            |

فحص ترحيل مصادقة الجهاز v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

إذا أظهرت السجلات أخطاء nonce/signature، فحدّث العميل المتصل وتحقق من أنه:

1. ينتظر `connect.challenge`
2. يوقّع الحمولة المرتبطة بالتحدي
3. يرسل `connect.params.device.nonce` باستخدام nonce التحدي نفسه

إذا رُفض `openclaw devices rotate` / `revoke` / `remove` بشكل غير متوقع:

- يمكن لجلسات رموز الأجهزة المقترنة إدارة **أجهزتها الخاصة فقط** ما لم يكن لدى
  الجهة المستدعية أيضًا `operator.admin`
- يمكن للأمر `openclaw devices rotate --scope ...` طلب نطاقات مشغّل فقط
  التي تمتلكها جلسة الجهة المستدعية بالفعل

ذو صلة:

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/ar/gateway/configuration) (أوضاع مصادقة البوابة)
- [/gateway/trusted-proxy-auth](/ar/gateway/trusted-proxy-auth)
- [/gateway/remote](/ar/gateway/remote)
- [/cli/devices](/cli/devices)

## خدمة البوابة لا تعمل

استخدم هذا عندما تكون الخدمة مثبتة لكن العملية لا تبقى قيد التشغيل.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # يفحص أيضًا الخدمات على مستوى النظام
```

ابحث عن:

- `Runtime: stopped` مع تلميحات عن الخروج.
- عدم تطابق إعدادات الخدمة (`Config (cli)` مقابل `Config (service)`).
- تعارضات المنفذ/المستمع.
- تثبيتات launchd/systemd/schtasks إضافية عند استخدام `--deep`.
- تلميحات تنظيف `Other gateway-like services detected (best effort)`.

الأنماط الشائعة:

- `Gateway start blocked: set gateway.mode=local` أو `existing config is missing gateway.mode` ← لم يتم تمكين وضع البوابة المحلية، أو تم العبث بملف الإعدادات وفقد `gateway.mode`. الإصلاح: اضبط `gateway.mode="local"` في إعداداتك، أو أعد تشغيل `openclaw onboard --mode local` / `openclaw setup` لإعادة ختم إعدادات الوضع المحلي المتوقعة. إذا كنت تشغّل OpenClaw عبر Podman، فمسار الإعدادات الافتراضي هو `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` ← ربط غير loopback من دون مسار مصادقة صالح للبوابة (رمز/كلمة مرور، أو trusted-proxy حيث يكون مضبوطًا).
- `another gateway instance is already listening` / `EADDRINUSE` ← تعارض منفذ.
- `Other gateway-like services detected (best effort)` ← توجد وحدات launchd/systemd/schtasks قديمة أو متوازية. معظم الإعدادات يجب أن تحتفظ ببوابة واحدة لكل جهاز؛ وإذا كنت تحتاج فعلًا إلى أكثر من واحدة، فاعزل المنافذ + الإعدادات/الحالة/مساحة العمل. راجع [/gateway#multiple-gateways-same-host](/ar/gateway#multiple-gateways-same-host).

ذو صلة:

- [/gateway/background-process](/ar/gateway/background-process)
- [/gateway/configuration](/ar/gateway/configuration)
- [/gateway/doctor](/ar/gateway/doctor)

## تحذيرات فحص البوابة

استخدم هذا عندما يصل `openclaw gateway probe` إلى شيء ما، لكنه لا يزال يطبع كتلة تحذير.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

ابحث عن:

- `warnings[].code` و `primaryTargetId` في مخرجات JSON.
- ما إذا كان التحذير متعلقًا بالرجوع إلى SSH، أو بوجود عدة بوابات، أو بنطاقات مفقودة، أو بمراجع مصادقة غير محلولة.

الأنماط الشائعة:

- `SSH tunnel failed to start; falling back to direct probes.` ← فشل إعداد SSH، لكن الأمر ما زال يحاول الأهداف المباشرة المضبوطة/loopback.
- `multiple reachable gateways detected` ← استجابت أكثر من جهة هدف. عادة يعني ذلك إعدادًا مقصودًا لعدة بوابات أو مستمعين قدامى/مكررين.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` ← نجح الاتصال، لكن تفاصيل RPC محدودة بالنطاقات؛ قم بإقران هوية الجهاز أو استخدم بيانات اعتماد تحتوي على `operator.read`.
- نص تحذير SecretRef غير المحلول لـ `gateway.auth.*` / `gateway.remote.*` ← لم تكن مواد المصادقة متاحة في مسار هذا الأمر للهدف الفاشل.

ذو صلة:

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/ar/gateway#multiple-gateways-same-host)
- [/gateway/remote](/ar/gateway/remote)

## القناة متصلة لكن الرسائل لا تتدفق

إذا كانت حالة القناة متصلة لكن تدفق الرسائل متوقف، فركّز على السياسة والأذونات وقواعد التسليم الخاصة بالقناة.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

ابحث عن:

- سياسة الرسائل المباشرة (`pairing`، `allowlist`، `open`، `disabled`).
- قائمة السماح للمجموعة ومتطلبات الإشارة.
- أذونات/نطاقات API المفقودة الخاصة بالقناة.

الأنماط الشائعة:

- `mention required` ← تم تجاهل الرسالة بسبب سياسة الإشارة في المجموعة.
- `pairing` / آثار موافقة معلقة ← المرسل غير معتمد.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` ← مشكلة في مصادقة/أذونات القناة.

ذو صلة:

- [/channels/troubleshooting](/ar/channels/troubleshooting)
- [/channels/whatsapp](/ar/channels/whatsapp)
- [/channels/telegram](/ar/channels/telegram)
- [/channels/discord](/ar/channels/discord)

## تسليم cron والنبضات

إذا لم يتم تشغيل cron أو النبضات أو لم يتم تسليمهما، فتحقق أولًا من حالة المجدول، ثم من جهة التسليم المستهدفة.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

ابحث عن:

- أن cron مفعّل وأن وقت التنبيه التالي موجود.
- حالة سجل تشغيل المهمة (`ok`، `skipped`، `error`).
- أسباب تخطي النبضات (`quiet-hours`، `requests-in-flight`، `alerts-disabled`، `empty-heartbeat-file`، `no-tasks-due`).

الأنماط الشائعة:

- `cron: scheduler disabled; jobs will not run automatically` ← تم تعطيل cron.
- `cron: timer tick failed` ← فشلت نبضة المؤقت للمجدول؛ تحقق من أخطاء الملفات/السجلات/وقت التشغيل.
- `heartbeat skipped` مع `reason=quiet-hours` ← خارج نافذة الساعات النشطة.
- `heartbeat skipped` مع `reason=empty-heartbeat-file` ← يوجد `HEARTBEAT.md` لكنه يحتوي فقط على أسطر فارغة / عناوين markdown، لذلك يتجاوز OpenClaw استدعاء النموذج.
- `heartbeat skipped` مع `reason=no-tasks-due` ← يحتوي `HEARTBEAT.md` على كتلة `tasks:`، لكن لا توجد مهام مستحقة في هذه النبضة.
- `heartbeat: unknown accountId` ← معرّف حساب غير صالح لجهة تسليم النبضات المستهدفة.
- `heartbeat skipped` مع `reason=dm-blocked` ← تم حل هدف النبضات إلى وجهة بأسلوب الرسائل المباشرة بينما `agents.defaults.heartbeat.directPolicy` (أو تجاوز كل وكيل) مضبوط على `block`.

ذو صلة:

- [/automation/cron-jobs#troubleshooting](/ar/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/ar/automation/cron-jobs)
- [/gateway/heartbeat](/ar/gateway/heartbeat)

## فشل أداة العقدة المقترنة

إذا كانت العقدة مقترنة لكن الأدوات تفشل، فاعزل حالة المقدمة والأذونات والموافقات.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

ابحث عن:

- أن العقدة متصلة بالإنترنت وبالقدرات المتوقعة.
- منح أذونات نظام التشغيل للكاميرا/المايكروفون/الموقع/الشاشة.
- حالة موافقات exec وقائمة السماح.

الأنماط الشائعة:

- `NODE_BACKGROUND_UNAVAILABLE` ← يجب أن يكون تطبيق العقدة في المقدمة.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` ← إذن نظام تشغيل مفقود.
- `SYSTEM_RUN_DENIED: approval required` ← موافقة exec معلقة.
- `SYSTEM_RUN_DENIED: allowlist miss` ← تم حظر الأمر بواسطة قائمة السماح.

ذو صلة:

- [/nodes/troubleshooting](/ar/nodes/troubleshooting)
- [/nodes/index](/ar/nodes/index)
- [/tools/exec-approvals](/ar/tools/exec-approvals)

## فشل أداة المتصفح

استخدم هذا عندما تفشل إجراءات أداة المتصفح رغم أن البوابة نفسها سليمة.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

ابحث عن:

- ما إذا كان `plugins.allow` مضبوطًا ويتضمن `browser`.
- مسار صالح لملف المتصفح التنفيذي.
- إمكانية الوصول إلى ملف تعريف CDP.
- توفر Chrome محليًا لملفات تعريف `existing-session` / `user`.

الأنماط الشائعة:

- `unknown command "browser"` أو `unknown command 'browser'` ← تم استبعاد إضافة المتصفح المضمّنة بواسطة `plugins.allow`.
- أداة المتصفح مفقودة / غير متاحة بينما `browser.enabled=true` ← يستبعد `plugins.allow` المكوّن `browser`، لذا لم تُحمّل الإضافة أصلًا.
- `Failed to start Chrome CDP on port` ← فشلت عملية المتصفح في التشغيل.
- `browser.executablePath not found` ← المسار المضبوط غير صالح.
- `browser.cdpUrl must be http(s) or ws(s)` ← يستخدم عنوان CDP URL المضبوط مخططًا غير مدعوم مثل `file:` أو `ftp:`.
- `browser.cdpUrl has invalid port` ← يحتوي CDP URL المضبوط على منفذ سيئ أو خارج النطاق.
- `No Chrome tabs found for profile="user"` ← لا توجد علامات تبويب Chrome محلية مفتوحة لملف تعريف إرفاق Chrome MCP.
- `Remote CDP for profile "<name>" is not reachable` ← نقطة نهاية CDP البعيدة المضبوطة غير قابلة للوصول من مضيف البوابة.
- `Browser attachOnly is enabled ... not reachable` أو `Browser attachOnly is enabled and CDP websocket ... is not reachable` ← لا يوجد هدف يمكن الوصول إليه لملف التعريف attach-only، أو أن نقطة نهاية HTTP استجابت لكن لم يمكن بعد ذلك فتح CDP WebSocket.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` ← يفتقر تثبيت البوابة الحالي إلى حزمة Playwright الكاملة؛ ولا تزال لقطات ARIA ولقطات الشاشة الأساسية للصفحات تعمل، لكن يظل التنقل ولقطات AI ولقطات عناصر محددات CSS وتصدير PDF غير متاحة.
- `fullPage is not supported for element screenshots` ← طلب لقطة الشاشة خلط `--full-page` مع `--ref` أو `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` ← يجب أن تستخدم استدعاءات لقطة الشاشة لـ Chrome MCP / `existing-session` التقاط الصفحة أو `--ref` من اللقطة، وليس CSS `--element`.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` ← تحتاج خطافات رفع الملفات في Chrome MCP إلى مراجع لقطات، وليس محددات CSS.
- `existing-session file uploads currently support one file at a time.` ← أرسل ملف رفع واحدًا لكل استدعاء في ملفات تعريف Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` ← لا تدعم خطافات مربعات الحوار في ملفات تعريف Chrome MCP تجاوزات المهلة.
- `response body is not supported for existing-session profiles yet.` ← ما يزال `responsebody` يتطلب متصفحًا مُدارًا أو ملف تعريف CDP خامًا.
- تجاوزات قديمة لمنفذ العرض / الوضع الداكن / اللغة المحلية / عدم الاتصال في ملفات تعريف attach-only أو CDP البعيدة ← شغّل `openclaw browser stop --browser-profile <name>` لإغلاق جلسة التحكم النشطة وتحرير حالة محاكاة Playwright/CDP من دون إعادة تشغيل البوابة كاملة.

ذو صلة:

- [/tools/browser-linux-troubleshooting](/ar/tools/browser-linux-troubleshooting)
- [/tools/browser](/ar/tools/browser)

## إذا قمت بالترقية وتعطل شيء فجأة

معظم الأعطال بعد الترقية تكون بسبب انجراف الإعدادات أو فرض إعدادات افتراضية أكثر صرامة الآن.

### 1) تغيّر سلوك المصادقة وتجاوز عنوان URL

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

ما الذي يجب التحقق منه:

- إذا كان `gateway.mode=remote`، فقد تستهدف استدعاءات CLI البيئة البعيدة بينما تكون خدمتك المحلية سليمة.
- الاستدعاءات الصريحة `--url` لا تعود إلى بيانات الاعتماد المخزنة.

الأنماط الشائعة:

- `gateway connect failed:` ← هدف URL غير صحيح.
- `unauthorized` ← يمكن الوصول إلى نقطة النهاية لكن المصادقة خاطئة.

### 2) أصبحت ضوابط الربط والمصادقة أكثر صرامة

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

ما الذي يجب التحقق منه:

- تحتاج عمليات الربط غير loopback (`lan`، `tailnet`، `custom`) إلى مسار مصادقة صالح للبوابة: مصادقة برمز/كلمة مرور مشتركة، أو نشر `trusted-proxy` غير loopback مضبوط بشكل صحيح.
- المفاتيح القديمة مثل `gateway.token` لا تحل محل `gateway.auth.token`.

الأنماط الشائعة:

- `refusing to bind gateway ... without auth` ← ربط غير loopback من دون مسار مصادقة صالح للبوابة.
- `RPC probe: failed` بينما وقت التشغيل يعمل ← البوابة حيّة لكن يتعذر الوصول إليها باستخدام إعدادات المصادقة/URL الحالية.

### 3) تغيّرت حالة الاقتران وهوية الجهاز

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

ما الذي يجب التحقق منه:

- موافقات الأجهزة المعلقة لـ dashboard/nodes.
- موافقات اقتران الرسائل المباشرة المعلقة بعد تغييرات السياسة أو الهوية.

الأنماط الشائعة:

- `device identity required` ← لم تُستوفَ مصادقة الجهاز.
- `pairing required` ← يجب الموافقة على المرسل/الجهاز.

إذا استمر عدم اتفاق إعدادات الخدمة ووقت التشغيل بعد عمليات التحقق، فأعد تثبيت بيانات تعريف الخدمة من الدليل نفسه للملف الشخصي/الحالة:

```bash
openclaw gateway install --force
openclaw gateway restart
```

ذو صلة:

- [/gateway/pairing](/ar/gateway/pairing)
- [/gateway/authentication](/ar/gateway/authentication)
- [/gateway/background-process](/ar/gateway/background-process)
