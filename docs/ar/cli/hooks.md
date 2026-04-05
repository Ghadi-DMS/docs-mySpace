---
read_when:
    - أنت تريد إدارة خطافات الوكيل
    - أنت تريد فحص توفر الخطافات أو تفعيل خطافات مساحة العمل
summary: مرجع CLI للأمر `openclaw hooks` (خطافات الوكيل)
title: hooks
x-i18n:
    generated_at: "2026-04-05T12:38:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8dc9144e9844e9c3cdef2514098eb170543746fcc55ca5a1cc746c12d80209e7
    source_path: cli/hooks.md
    workflow: 15
---

# `openclaw hooks`

إدارة خطافات الوكيل (عمليات أتمتة مدفوعة بالأحداث لأوامر مثل `/new` و`/reset` وبدء تشغيل البوابة).

يُعادل تشغيل `openclaw hooks` من دون أمر فرعي تشغيل `openclaw hooks list`.

ذو صلة:

- الخطافات: [Hooks](/automation/hooks)
- خطافات plugin: [خطافات Plugin](/plugins/architecture#provider-runtime-hooks)

## سرد جميع الخطافات

```bash
openclaw hooks list
```

يسرد جميع الخطافات المكتشفة من أدلة مساحة العمل، والمدارة، والإضافية، والمضمنة.

**الخيارات:**

- `--eligible`: عرض الخطافات المؤهلة فقط (التي استوفت المتطلبات)
- `--json`: إخراج بصيغة JSON
- `-v, --verbose`: عرض معلومات تفصيلية بما في ذلك المتطلبات المفقودة

**مثال على الإخراج:**

```
Hooks (4/4 ready)

Ready:
  🚀 boot-md ✓ - Run BOOT.md on gateway startup
  📎 bootstrap-extra-files ✓ - Inject extra workspace bootstrap files during agent bootstrap
  📝 command-logger ✓ - Log all command events to a centralized audit file
  💾 session-memory ✓ - Save session context to memory when /new or /reset command is issued
```

**مثال (verbose):**

```bash
openclaw hooks list --verbose
```

يعرض المتطلبات المفقودة للخطافات غير المؤهلة.

**مثال (JSON):**

```bash
openclaw hooks list --json
```

يعيد JSON منظمًا للاستخدام البرمجي.

## الحصول على معلومات الخطاف

```bash
openclaw hooks info <name>
```

يعرض معلومات تفصيلية عن خطاف محدد.

**الوسائط:**

- `<name>`: اسم الخطاف أو مفتاح الخطاف (مثل `session-memory`)

**الخيارات:**

- `--json`: إخراج بصيغة JSON

**مثال:**

```bash
openclaw hooks info session-memory
```

**الإخراج:**

```
💾 session-memory ✓ Ready

Save session context to memory when /new or /reset command is issued

Details:
  Source: openclaw-bundled
  Path: /path/to/openclaw/hooks/bundled/session-memory/HOOK.md
  Handler: /path/to/openclaw/hooks/bundled/session-memory/handler.ts
  Homepage: https://docs.openclaw.ai/automation/hooks#session-memory
  Events: command:new, command:reset

Requirements:
  Config: ✓ workspace.dir
```

## التحقق من أهلية الخطافات

```bash
openclaw hooks check
```

يعرض ملخصًا لحالة أهلية الخطافات (عدد الخطافات الجاهزة مقابل غير الجاهزة).

**الخيارات:**

- `--json`: إخراج بصيغة JSON

**مثال على الإخراج:**

```
Hooks Status

Total hooks: 4
Ready: 4
Not ready: 0
```

## تفعيل خطاف

```bash
openclaw hooks enable <name>
```

يفعّل خطافًا محددًا بإضافته إلى إعداداتك (`~/.openclaw/openclaw.json` افتراضيًا).

**ملاحظة:** تكون خطافات مساحة العمل معطلة افتراضيًا حتى يتم تفعيلها هنا أو في الإعدادات. وتظهر الخطافات التي تديرها plugins على شكل `plugin:<id>` في `openclaw hooks list` ولا يمكن تفعيلها/تعطيلها هنا. قم بتفعيل/تعطيل plugin نفسه بدلًا من ذلك.

**الوسائط:**

- `<name>`: اسم الخطاف (مثل `session-memory`)

**مثال:**

```bash
openclaw hooks enable session-memory
```

**الإخراج:**

```
✓ Enabled hook: 💾 session-memory
```

**ما الذي يفعله:**

- يتحقق من وجود الخطاف وأنه مؤهل
- يحدّث `hooks.internal.entries.<name>.enabled = true` في إعداداتك
- يحفظ الإعدادات على القرص

إذا كان الخطاف قد جاء من `<workspace>/hooks/`، فإن خطوة الاشتراك هذه مطلوبة قبل
أن تقوم البوابة بتحميله.

**بعد التفعيل:**

- أعد تشغيل البوابة حتى تعيد الخطافات التحميل (إعادة تشغيل تطبيق شريط القوائم على macOS، أو أعد تشغيل عملية البوابة في وضع التطوير).

## تعطيل خطاف

```bash
openclaw hooks disable <name>
```

يعطّل خطافًا محددًا عبر تحديث إعداداتك.

**الوسائط:**

- `<name>`: اسم الخطاف (مثل `command-logger`)

**مثال:**

```bash
openclaw hooks disable command-logger
```

**الإخراج:**

```
⏸ Disabled hook: 📝 command-logger
```

**بعد التعطيل:**

- أعد تشغيل البوابة حتى تعيد الخطافات التحميل

## ملاحظات

- تكتب الأوامر `openclaw hooks list --json` و`info --json` و`check --json` JSON منظمًا مباشرة إلى stdout.
- لا يمكن تفعيل أو تعطيل الخطافات التي تديرها plugins هنا؛ قم بتفعيل أو تعطيل plugin المالك بدلًا من ذلك.

## تثبيت حزم الخطافات

```bash
openclaw plugins install <package>        # ClawHub أولًا، ثم npm
openclaw plugins install <package> --pin  # تثبيت الإصدار
openclaw plugins install <path>           # مسار محلي
```

ثبّت حزم الخطافات من خلال مثبّت plugins الموحد.

لا يزال `openclaw hooks install` يعمل كاسم مستعار للتوافق، لكنه يطبع
تحذير إهمال ويعيد التوجيه إلى `openclaw plugins install`.

تكون مواصفات npm **خاصة بالسجل فقط** (اسم الحزمة + **إصدار دقيق** اختياري أو
**dist-tag**). ويتم رفض مواصفات Git/URL/file ونطاقات semver. وتُشغَّل
عمليات تثبيت التبعيات باستخدام `--ignore-scripts` لأسباب تتعلق بالأمان.

تبقى المواصفات المجردة و`@latest` على المسار المستقر. وإذا قام npm بحل أيٍّ من
هذين إلى إصدار prerelease، فسيتوقف OpenClaw ويطلب منك الاشتراك صراحةً باستخدام
وسم prerelease مثل `@beta`/`@rc` أو إصدار prerelease دقيق.

**ما الذي يفعله:**

- ينسخ حزمة الخطافات إلى `~/.openclaw/hooks/<id>`
- يفعّل الخطافات المثبتة في `hooks.internal.entries.*`
- يسجل التثبيت ضمن `hooks.internal.installs`

**الخيارات:**

- `-l, --link`: ربط دليل محلي بدلًا من نسخه (يضيفه إلى `hooks.internal.load.extraDirs`)
- `--pin`: تسجيل عمليات تثبيت npm على شكل `name@version` محلول ودقيق في `hooks.internal.installs`

**الأرشيفات المدعومة:** `.zip`, `.tgz`, `.tar.gz`, `.tar`

**أمثلة:**

```bash
# دليل محلي
openclaw plugins install ./my-hook-pack

# أرشيف محلي
openclaw plugins install ./my-hook-pack.zip

# حزمة NPM
openclaw plugins install @openclaw/my-hook-pack

# ربط دليل محلي من دون نسخه
openclaw plugins install -l ./my-hook-pack
```

تُعامل حزم الخطافات المرتبطة على أنها خطافات مُدارة من دليل
مُعد من المشغّل، وليست خطافات مساحة عمل.

## تحديث حزم الخطافات

```bash
openclaw plugins update <id>
openclaw plugins update --all
```

حدّث حزم الخطافات المعتمدة على npm والمتعقبة من خلال محدّث plugins الموحد.

لا يزال `openclaw hooks update` يعمل كاسم مستعار للتوافق، لكنه يطبع
تحذير إهمال ويعيد التوجيه إلى `openclaw plugins update`.

**الخيارات:**

- `--all`: تحديث جميع حزم الخطافات المتعقبة
- `--dry-run`: عرض ما سيتغير من دون كتابة

عند وجود قيمة hash سلامة مخزنة وتغيّر hash الأثر الذي تم جلبه،
يطبع OpenClaw تحذيرًا ويطلب التأكيد قبل المتابعة. استخدم
القيمة العامة `--yes` لتجاوز المطالبات في تشغيلات CI/غير التفاعلية.

## الخطافات المضمنة

### session-memory

يحفظ سياق الجلسة في الذاكرة عند إصدار `/new` أو `/reset`.

**التفعيل:**

```bash
openclaw hooks enable session-memory
```

**الإخراج:** `~/.openclaw/workspace/memory/YYYY-MM-DD-slug.md`

**راجع:** [توثيق session-memory](/automation/hooks#session-memory)

### bootstrap-extra-files

يحقن ملفات bootstrap إضافية (على سبيل المثال `AGENTS.md` / `TOOLS.md` محلية في monorepo) أثناء `agent:bootstrap`.

**التفعيل:**

```bash
openclaw hooks enable bootstrap-extra-files
```

**راجع:** [توثيق bootstrap-extra-files](/automation/hooks#bootstrap-extra-files)

### command-logger

يسجل جميع أحداث الأوامر في ملف تدقيق مركزي.

**التفعيل:**

```bash
openclaw hooks enable command-logger
```

**الإخراج:** `~/.openclaw/logs/commands.log`

**عرض السجلات:**

```bash
# الأوامر الحديثة
tail -n 20 ~/.openclaw/logs/commands.log

# تنسيق جميل
cat ~/.openclaw/logs/commands.log | jq .

# التصفية حسب الإجراء
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .
```

**راجع:** [توثيق command-logger](/automation/hooks#command-logger)

### boot-md

يشغّل `BOOT.md` عند بدء البوابة (بعد بدء القنوات).

**الأحداث**: `gateway:startup`

**التفعيل**:

```bash
openclaw hooks enable boot-md
```

**راجع:** [توثيق boot-md](/automation/hooks#boot-md)
