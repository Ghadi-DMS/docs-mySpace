---
read_when:
    - استخدام أداة exec أو تعديلها
    - تصحيح سلوك stdin أو TTY
summary: استخدام أداة Exec، وأوضاع stdin، ودعم TTY
title: أداة Exec
x-i18n:
    generated_at: "2026-04-06T03:13:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28388971c627292dba9bf65ae38d7af8cde49a33bb3b5fc8b20da4f0e350bedd
    source_path: tools/exec.md
    workflow: 15
---

# أداة Exec

شغّل أوامر shell في مساحة العمل. تدعم التنفيذ في المقدمة + الخلفية عبر `process`.
إذا كان `process` غير مسموح، تعمل `exec` بشكل متزامن وتتجاهل `yieldMs`/`background`.
جلسات الخلفية تكون ضمن نطاق كل وكيل؛ ولا يرى `process` إلا الجلسات الخاصة بالوكيل نفسه.

## المعلمات

- `command` (مطلوب)
- `workdir` (الافتراضي هو cwd)
- `env` (تجاوزات key/value)
- `yieldMs` (الافتراضي 10000): نقل تلقائي إلى الخلفية بعد التأخير
- `background` (قيمة منطقية): التشغيل في الخلفية فورًا
- `timeout` (بالثواني، الافتراضي 1800): إنهاء عند انتهاء المهلة
- `pty` (قيمة منطقية): التشغيل في pseudo-terminal عند توفره (CLI التي تعمل فقط مع TTY، ووكلاء البرمجة، وواجهات الطرفية)
- `host` (`auto | sandbox | gateway | node`): مكان التنفيذ
- `security` (`deny | allowlist | full`): وضع الفرض لـ `gateway`/`node`
- `ask` (`off | on-miss | always`): مطالبات الموافقة لـ `gateway`/`node`
- `node` (سلسلة): معرّف/اسم العقدة لـ `host=node`
- `elevated` (قيمة منطقية): طلب وضع مرتفع الصلاحية (الخروج من sandbox إلى مسار المضيف المكوَّن)؛ ولا يُفرَض `security=full` إلا عندما يُحل elevated إلى `full`

ملاحظات:

- الإعداد الافتراضي لـ `host` هو `auto`: ‏sandbox عندما يكون وقت تشغيل sandbox نشطًا للجلسة، وإلا gateway.
- `auto` هو استراتيجية التوجيه الافتراضية، وليس wildcard. يُسمح باستخدام `host=node` لكل استدعاء من `auto`؛ أما `host=gateway` لكل استدعاء فلا يُسمح به إلا عندما لا يكون وقت تشغيل sandbox نشطًا.
- بدون أي تكوين إضافي، يظل `host=auto` "يعمل ببساطة": إذا لم توجد sandbox فسيُحل إلى `gateway`؛ وإذا كانت هناك sandbox نشطة فسيبقى داخلها.
- يؤدي `elevated` إلى الخروج من sandbox إلى مسار المضيف المكوَّن: `gateway` افتراضيًا، أو `node` عندما يكون `tools.exec.host=node` (أو تكون الجلسة افتراضيًا `host=node`). وهو متاح فقط عندما يكون الوصول المرتفع الصلاحية مفعّلًا للجلسة/الموفّر الحالي.
- تتحكم `~/.openclaw/exec-approvals.json` في موافقات `gateway`/`node`.
- يتطلب `node` عقدة مقترنة (تطبيقًا مرافقًا أو مضيف عقدة بدون واجهة).
- إذا كانت هناك عدة عقد متاحة، فاضبط `exec.node` أو `tools.exec.node` لاختيار واحدة.
- `exec host=node` هو مسار تنفيذ shell الوحيد للعقد؛ وقد أزيل الغلاف القديم `nodes.run`.
- على المضيفات غير Windows، تستخدم exec القيمة `SHELL` عند تعيينها؛ وإذا كانت `SHELL` هي `fish`، فإنها تفضّل `bash` (أو `sh`)
  من `PATH` لتجنّب السكربتات غير المتوافقة مع fish، ثم تعود إلى `SHELL` إذا لم يوجد أي منهما.
- على مضيفات Windows، تفضّل exec اكتشاف PowerShell 7 (`pwsh`) ‏(Program Files، ثم ProgramW6432، ثم PATH)،
  ثم تعود إلى Windows PowerShell 5.1.
- يرفض التنفيذ على المضيف (`gateway`/`node`) تجاوزات `env.PATH` وتجاوزات loader ‏(`LD_*`/`DYLD_*`) من أجل
  منع اختطاف الثنائيات أو حقن الشيفرة.
- يضبط OpenClaw القيمة `OPENCLAW_SHELL=exec` في بيئة الأمر المشغَّل (بما في ذلك تنفيذ PTY وsandbox) حتى تتمكن قواعد shell/profile من اكتشاف سياق أداة exec.
- مهم: يكون sandboxing **معطلًا افتراضيًا**. إذا كان sandboxing معطلًا، فإن `host=auto`
  الضمني يُحل إلى `gateway`. أما `host=sandbox` الصريح فيفشل بشكل مغلق بدلًا من التشغيل بصمت
  على مضيف gateway. فعّل sandboxing أو استخدم `host=gateway` مع الموافقات.
- لا تفحص عمليات preflight للسكربتات (لأخطاء صياغة shell الشائعة في Python/Node) إلا الملفات داخل
  حدود `workdir` الفعالة. إذا كان مسار السكربت يُحل خارج `workdir`، يتم تخطي preflight لذلك
  الملف.
- بالنسبة إلى الأعمال طويلة التشغيل التي تبدأ الآن، ابدأها مرة واحدة واعتمد على
  تنبيه الاكتمال التلقائي عندما يكون مفعّلًا ويصدر الأمر مخرجات أو يفشل.
  استخدم `process` للسجلات أو الحالة أو الإدخال أو التدخل؛ ولا تحاكِ
  الجدولة باستخدام حلقات sleep أو timeout أو polling متكرر.
- للأعمال التي يجب أن تحدث لاحقًا أو وفق جدول زمني، استخدم cron بدلًا من
  أنماط sleep/delay في `exec`.

## التكوين

- `tools.exec.notifyOnExit` (الافتراضي: true): عندما تكون true، تقوم جلسات exec التي انتقلت إلى الخلفية بوضع حدث نظام في قائمة الانتظار وتطلب heartbeat عند الخروج.
- `tools.exec.approvalRunningNoticeMs` (الافتراضي: 10000): إصدار إشعار واحد "قيد التشغيل" عندما تستغرق exec المحكومة بالموافقة مدة أطول من هذا (القيمة 0 تعطل ذلك).
- `tools.exec.host` (الافتراضي: `auto`؛ ويُحل إلى `sandbox` عندما يكون وقت تشغيل sandbox نشطًا، وإلى `gateway` خلاف ذلك)
- `tools.exec.security` (الافتراضي: `deny` لـ sandbox، و`full` لـ gateway + node عندما لا يكون مضبوطًا)
- `tools.exec.ask` (الافتراضي: `off`)
- تنفيذ المضيف بدون موافقة هو الوضع الافتراضي لـ gateway + node. إذا كنت تريد سلوك الموافقات/allowlist، فشدّد كلًا من `tools.exec.*` وملف سياسة المضيف `~/.openclaw/exec-approvals.json`؛ راجع [Exec approvals](/ar/tools/exec-approvals#no-approval-yolo-mode).
- يأتي وضع YOLO من إعدادات سياسة المضيف الافتراضية (`security=full`, `ask=off`)، وليس من `host=auto`. إذا كنت تريد فرض التوجيه إلى gateway أو node، فاضبط `tools.exec.host` أو استخدم `/exec host=...`.
- في وضع `security=full` مع `ask=off`، يتبع تنفيذ المضيف السياسة المكوّنة مباشرة؛ ولا توجد مصفاة heuristic إضافية مسبقة لتعتيم الأوامر.
- `tools.exec.node` (الافتراضي: غير مضبوط)
- `tools.exec.strictInlineEval` (الافتراضي: false): عندما تكون true، تتطلب صيغ eval المضمنة للمفسرات مثل `python -c` و`node -e` و`ruby -e` و`perl -e` و`php -r` و`lua -e` و`osascript -e` موافقة صريحة دائمًا. ويمكن مع `allow-always` الاستمرار في حفظ استدعاءات المفسر/السكربت الآمنة، لكن صيغ inline-eval ستطلب موافقة كل مرة.
- `tools.exec.pathPrepend`: قائمة أدلة تُضاف في بداية `PATH` لتشغيلات exec ‏(gateway + sandbox فقط).
- `tools.exec.safeBins`: ثنائيات آمنة تعمل عبر stdin فقط ويمكنها العمل دون إدخالات allowlist صريحة. لتفاصيل السلوك، راجع [Safe bins](/ar/tools/exec-approvals#safe-bins-stdin-only).
- `tools.exec.safeBinTrustedDirs`: أدلة إضافية صريحة موثوق بها لفحوصات مسارات ملفات safeBins التنفيذية. لا تُعد إدخالات `PATH` موثوقًا بها تلقائيًا مطلقًا. الإعدادات المدمجة الافتراضية هي `/bin` و`/usr/bin`.
- `tools.exec.safeBinProfiles`: سياسة argv مخصصة اختيارية لكل safe bin ‏(`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).

مثال:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### التعامل مع PATH

- `host=gateway`: يدمج `PATH` الخاص بـ login-shell في بيئة exec. تُرفض تجاوزات `env.PATH`
  لتنفيذ المضيف. ومع ذلك، يظل daemon نفسه يعمل بحد أدنى من `PATH`:
  - macOS: ‏`/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: ‏`/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: يشغّل `sh -lc` ‏(login shell) داخل الحاوية، لذلك قد يعيد `/etc/profile` ضبط `PATH`.
  يضيف OpenClaw قيمة `env.PATH` في البداية بعد تحميل profile عبر متغير env داخلي (من دون shell interpolation)؛ وينطبق `tools.exec.pathPrepend` هنا أيضًا.
- `host=node`: تُرسل فقط تجاوزات env غير المحظورة التي تمررها إلى node. تُرفض تجاوزات `env.PATH`
  لتنفيذ المضيف ويتم تجاهلها بواسطة مضيفات node. إذا كنت بحاجة إلى إدخالات PATH إضافية على عقدة،
  فقم بتكوين بيئة خدمة مضيف العقدة (systemd/launchd) أو ثبّت الأدوات في مواقع قياسية.

ربط العقدة لكل وكيل (استخدم فهرس قائمة الوكلاء في التكوين):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI: تتضمن علامة تبويب Nodes لوحة صغيرة باسم “Exec node binding” للإعدادات نفسها.

## تجاوزات الجلسة (`/exec`)

استخدم `/exec` لضبط القيم الافتراضية **لكل جلسة** لـ `host` و`security` و`ask` و`node`.
أرسل `/exec` من دون أي معاملات لعرض القيم الحالية.

مثال:

```
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

## نموذج التفويض

لا يُطبَّق `/exec` إلا على **المرسلين المصرح لهم** (قوائم السماح/الاقتران في القناة بالإضافة إلى `commands.useAccessGroups`).
وهو يحدّث **حالة الجلسة فقط** ولا يكتب إلى التكوين. ولتعطيل exec بشكل صارم، امنعه عبر
سياسة الأداة (`tools.deny: ["exec"]` أو لكل وكيل). وما تزال موافقات المضيف مطبقة ما لم تضبط صراحة
`security=full` و`ask=off`.

## موافقات Exec ‏(التطبيق المرافق / مضيف العقدة)

يمكن للوكلاء داخل sandbox أن يطلبوا موافقة لكل طلب قبل تشغيل `exec` على مضيف gateway أو node.
راجع [Exec approvals](/ar/tools/exec-approvals) لمعرفة السياسة، وallowlist، وتدفق واجهة المستخدم.

عندما تكون الموافقات مطلوبة، تعيد أداة exec فورًا
`status: "approval-pending"` ومعه معرّف موافقة. وبمجرد الموافقة (أو الرفض / انتهاء المهلة)،
تصدر البوابة أحداث نظام (`Exec finished` / `Exec denied`). وإذا كان الأمر ما يزال
قيد التشغيل بعد `tools.exec.approvalRunningNoticeMs`، يصدر إشعار واحد `Exec running`.
في القنوات التي تحتوي على بطاقات/أزرار موافقة أصلية، يجب على الوكيل الاعتماد على
واجهة المستخدم الأصلية هذه أولًا وعدم تضمين أمر `/approve` اليدوي إلا عندما
تشير نتيجة الأداة صراحة إلى أن موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي
المسار الوحيد.

## Allowlist + Safe bins

تطابق عملية فرض allowlist اليدوية **مسارات الثنائيات المحلولة فقط** (من دون مطابقة اسم الأساس). عندما
يكون `security=allowlist`، تُسمح أوامر shell تلقائيًا فقط إذا كان كل مقطع في pipeline
ضمن allowlist أو safe bin. وتُرفض عمليات التسلسل (`;`, `&&`, `||`) وعمليات إعادة التوجيه في
وضع allowlist ما لم يستوفِ كل مقطع من المستوى الأعلى allowlist (بما في ذلك safe bins).
وتظل عمليات إعادة التوجيه غير مدعومة.
ولا تتجاوز الثقة الدائمة عبر `allow-always` هذه القاعدة: إذ يظل الأمر المتسلسل يتطلب أن يطابق كل
مقطع من المستوى الأعلى.

يمثل `autoAllowSkills` مسار تسهيل منفصلًا في موافقات exec. وهو ليس مطابقًا
لإدخالات allowlist اليدوية للمسارات. وإذا كنت تريد ثقة صريحة صارمة، فأبقِ `autoAllowSkills` معطلًا.

استخدم وسيلتي التحكم للمهام المختلفة:

- `tools.exec.safeBins`: مرشحات تدفق صغيرة تعمل عبر stdin فقط.
- `tools.exec.safeBinTrustedDirs`: أدلة إضافية صريحة موثوق بها لمسارات ملفات safe-bin التنفيذية.
- `tools.exec.safeBinProfiles`: سياسة argv صريحة لـ safe bins المخصصة.
- allowlist: ثقة صريحة لمسارات الملفات التنفيذية.

لا تتعامل مع `safeBins` على أنه allowlist عامة، ولا تضف ثنائيات المفسرات/بيئات التشغيل (مثل `python3` أو `node` أو `ruby` أو `bash`). إذا كنت بحاجة إليها، فاستخدم إدخالات allowlist صريحة وأبقِ مطالبات الموافقة مفعلة.
يحذر `openclaw security audit` عندما تكون إدخالات `safeBins` الخاصة بالمفسرات/بيئات التشغيل تفتقد ملفات تعريف صريحة، ويمكن لـ `openclaw doctor --fix` إنشاء إدخالات `safeBinProfiles` المخصصة المفقودة.
كما يحذر `openclaw security audit` و`openclaw doctor` أيضًا عندما تضيف صراحة ثنائيات واسعة السلوك مثل `jq` مرة أخرى إلى `safeBins`.
إذا أضفت المفسرات صراحة إلى allowlist، ففعّل `tools.exec.strictInlineEval` حتى تظل صيغ تقييم الشيفرة المضمنة تتطلب موافقة جديدة.

للحصول على تفاصيل السياسة الكاملة والأمثلة، راجع [Exec approvals](/ar/tools/exec-approvals#safe-bins-stdin-only) و[Safe bins versus allowlist](/ar/tools/exec-approvals#safe-bins-versus-allowlist).

## أمثلة

في المقدمة:

```json
{ "tool": "exec", "command": "ls -la" }
```

في الخلفية + استطلاع:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

يُستخدم الاستطلاع للحالة عند الطلب، وليس لحلقات الانتظار. وإذا كان تنبيه
الاكتمال التلقائي مفعّلًا، يمكن للأمر إيقاظ الجلسة عندما يصدر مخرجات أو يفشل.

إرسال مفاتيح (بنمط tmux):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

إرسال التنفيذ (إرسال CR فقط):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

لصق (مع bracketed افتراضيًا):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` أداة فرعية من `exec` لإجراء تعديلات منظمة على عدة ملفات.
وهي مفعّلة افتراضيًا لنماذج OpenAI وOpenAI Codex. استخدم التكوين فقط
عندما تريد تعطيلها أو تقييدها على نماذج محددة:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.4"] },
    },
  },
}
```

ملاحظات:

- متاحة فقط لنماذج OpenAI/OpenAI Codex.
- ما تزال سياسة الأداة مطبقة؛ إذ إن `allow: ["write"]` تسمح ضمنيًا بـ `apply_patch`.
- يوجد التكوين تحت `tools.exec.applyPatch`.
- يكون `tools.exec.applyPatch.enabled` مضبوطًا افتراضيًا على `true`؛ اضبطه على `false` لتعطيل الأداة لنماذج OpenAI.
- يكون `tools.exec.applyPatch.workspaceOnly` مضبوطًا افتراضيًا على `true` ‏(محصور داخل مساحة العمل). اضبطه على `false` فقط إذا كنت تريد عمدًا أن يكتب `apply_patch`/يحذف خارج دليل مساحة العمل.

## ذو صلة

- [Exec Approvals](/ar/tools/exec-approvals) — بوابات الموافقة لأوامر shell
- [Sandboxing](/ar/gateway/sandboxing) — تشغيل الأوامر في بيئات sandbox
- [Background Process](/ar/gateway/background-process) — exec طويل التشغيل وأداة process
- [Security](/ar/gateway/security) — سياسة الأداة والوصول المرتفع الصلاحية
