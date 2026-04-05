---
read_when:
    - تريد تشغيل تدقيق أمني سريع على التهيئة/الحالة
    - تريد تطبيق اقتراحات “fix” الآمنة (الأذونات، وتشديد الإعدادات الافتراضية)
summary: مرجع CLI للأمر `openclaw security` (تدقيق مشكلات الأمان الشائعة وإصلاحها)
title: security
x-i18n:
    generated_at: "2026-04-05T12:39:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: e5a3e4ab8e0dfb6c10763097cb4483be2431985f16de877523eb53e2122239ae
    source_path: cli/security.md
    workflow: 15
---

# `openclaw security`

أدوات الأمان (تدقيق + إصلاحات اختيارية).

ذو صلة:

- دليل الأمان: [الأمان](/gateway/security)

## التدقيق

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --fix
openclaw security audit --json
```

يحذر التدقيق عندما يتشارك عدة مرسلين للرسائل المباشرة الجلسة الرئيسية، ويوصي باستخدام **وضع الرسائل المباشرة الآمن**: ‏`session.dmScope="per-channel-peer"` (أو `per-account-channel-peer` للقنوات متعددة الحسابات) لصناديق الوارد المشتركة.
وهذا مخصص لتقوية صناديق الوارد التعاونية/المشتركة. ولا يُعد Gateway واحد مشترك بين مشغلين غير موثوقين/عدائيين إعدادًا موصى به؛ افصل حدود الثقة باستخدام Gateways منفصلة (أو مستخدمي/مضيفي نظام تشغيل منفصلين).
كما أنه يصدر `security.trust_model.multi_user_heuristic` عندما توحي التهيئة بوجود إدخال محتمل لمستخدمين مشتركين (على سبيل المثال سياسة رسائل مباشرة/مجموعات مفتوحة، أو أهداف مجموعات مهيأة، أو قواعد مرسلين عامة)، ويذكرك بأن OpenClaw يعتمد افتراضيًا نموذج ثقة المساعد الشخصي.
وبالنسبة إلى الإعدادات المقصودة للمستخدمين المشتركين، فإن إرشادات التدقيق هي وضع جميع الجلسات في sandbox، والإبقاء على وصول نظام الملفات ضمن نطاق workspace، وإبقاء الهويات أو بيانات الاعتماد الشخصية/الخاصة خارج وقت التشغيل هذا.
كما يحذر أيضًا عند استخدام نماذج صغيرة (`<=300B`) من دون sandbox ومع تمكين أدوات الويب/المتصفح.
وبالنسبة إلى إدخال webhook، فإنه يحذر عندما يعيد `hooks.token` استخدام token الخاص بـ Gateway، أو عندما يكون `hooks.token` قصيرًا، أو عندما تكون `hooks.path="/"`، أو عندما لا تكون `hooks.defaultSessionKey` مضبوطة، أو عندما تكون `hooks.allowedAgentIds` غير مقيّدة، أو عندما تكون تجاوزات `sessionKey` للطلبات مفعّلة، أو عندما تكون التجاوزات مفعلة من دون `hooks.allowedSessionKeyPrefixes`.
كما يحذر أيضًا عند تهيئة إعدادات sandbox Docker بينما وضع sandbox معطّل، وعندما يستخدم `gateway.nodes.denyCommands` إدخالات غير فعالة شبيهة بالأنماط/غير معروفة (مطابقة دقيقة لاسم أمر العقدة فقط، وليس تصفية نص shell)، وعندما يفعّل `gateway.nodes.allowCommands` صراحة أوامر عقدة خطيرة، وعندما يتم تجاوز `tools.profile="minimal"` العام بواسطة ملفات تعريف أدوات الوكيل، وعندما تكشف المجموعات المفتوحة أدوات وقت التشغيل/نظام الملفات من دون حواجز sandbox/workspace، وعندما يمكن الوصول إلى أدوات extension plugin المثبتة في ظل سياسة أدوات متساهلة.
كما يشير أيضًا إلى `gateway.allowRealIpFallback=true` (خطر انتحال الترويسات إذا أسيء ضبط الوكلاء) و`discovery.mdns.mode="full"` (تسرّب البيانات الوصفية عبر سجلات mDNS TXT).
كما يحذر أيضًا عندما يستخدم متصفح sandbox شبكة Docker من نوع `bridge` من دون `sandbox.browser.cdpSourceRange`.
كما يشير أيضًا إلى أوضاع شبكة sandbox Docker الخطيرة (بما في ذلك انضمامات مساحة الأسماء `host` و`container:*`).
كما يحذر أيضًا عندما تكون حاويات Docker الحالية لمتصفح sandbox تحتوي على تسميات hash مفقودة/قديمة (على سبيل المثال الحاويات السابقة للترحيل التي تفتقد `openclaw.browserConfigEpoch`) ويوصي باستخدام `openclaw sandbox recreate --browser --all`.
كما يحذر أيضًا عندما تكون سجلات تثبيت plugin/hook المعتمدة على npm غير مثبّتة بإصدار محدد، أو تفتقد بيانات integrity الوصفية، أو تنحرف عن إصدارات الحزم المثبتة حاليًا.
كما يحذر عندما تعتمد قوائم السماح الخاصة بالقنوات على أسماء/بريد إلكتروني/وسوم قابلة للتغيير بدلًا من معرّفات ثابتة (Discord وSlack وGoogle Chat وMicrosoft Teams وMattermost ونطاقات IRC عند الاقتضاء).
كما يحذر عندما يترك `gateway.auth.mode="none"` واجهات Gateway HTTP API قابلة للوصول من دون سر مشترك (`/tools/invoke` بالإضافة إلى أي نقطة نهاية `/v1/*` مفعّلة).
وتُعد الإعدادات التي تبدأ بـ `dangerous`/`dangerously` تجاوزات صريحة من المشغل لكسر الزجاج عند الطوارئ؛ ولا يُعد تمكين أحدها، بحد ذاته، تقرير ثغرة أمنية.
للحصول على الجرد الكامل للمعلمات الخطيرة، راجع قسم "ملخص العلامات غير الآمنة أو الخطيرة" في [الأمان](/gateway/security).

سلوك SecretRef:

- يقوم `security audit` بحل SecretRefs المدعومة في وضع القراءة فقط للمسارات المستهدفة الخاصة به.
- إذا كان SecretRef غير متاح في مسار الأمر الحالي، يستمر التدقيق ويبلّغ عن `secretDiagnostics` (بدلًا من التعطل).
- لا يؤدي `--token` و`--password` إلا إلى تجاوز مصادقة deep-probe لهذا الاستدعاء من الأمر؛ وهما لا يعيدان كتابة التهيئة أو تعيينات SecretRef.

## إخراج JSON

استخدم `--json` لفحوصات CI/السياسات:

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

إذا جرى دمج `--fix` و`--json`، فسيتضمن الإخراج إجراءات الإصلاح والتقرير النهائي معًا:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## ما الذي يغيره `--fix`

يطبق `--fix` معالجات آمنة وحتمية:

- يحول القيم الشائعة `groupPolicy="open"` إلى `groupPolicy="allowlist"` (بما في ذلك متغيرات الحسابات في القنوات المدعومة)
- عندما تتحول سياسة مجموعة WhatsApp إلى `allowlist`، فإنه يزرع `groupAllowFrom` من
  ملف `allowFrom` المخزن عندما تكون تلك القائمة موجودة ولا تكون التهيئة قد عرّفت بالفعل
  `allowFrom`
- يضبط `logging.redactSensitive` من `"off"` إلى `"tools"`
- يشدد الأذونات للحالة/التهيئة والملفات الحساسة الشائعة
  (`credentials/*.json` و`auth-profiles.json` و`sessions.json` وملفات الجلسات
  `*.jsonl`)
- كما يشدد أيضًا ملفات تضمين التهيئة المشار إليها من `openclaw.json`
- يستخدم `chmod` على مضيفات POSIX وعمليات إعادة تعيين `icacls` على Windows

لا يقوم `--fix` بما يلي:

- تدوير tokens أو كلمات المرور أو مفاتيح API
- تعطيل الأدوات (`gateway` أو `cron` أو `exec` وما إلى ذلك)
- تغيير خيارات ربط/auth/تعرض الشبكة الخاصة بـ gateway
- إزالة أو إعادة كتابة plugins/Skills
