---
read_when:
    - أنت تريد تثبيت أو إدارة plugins البوابة أو الحزم المتوافقة
    - أنت تريد تصحيح إخفاقات تحميل plugin
summary: مرجع CLI للأمر `openclaw plugins` (السرد، والتثبيت، وmarketplace، وإلغاء التثبيت، والتفعيل/التعطيل، وdoctor)
title: plugins
x-i18n:
    generated_at: "2026-04-05T12:39:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8c35ccf68cd7be1af5fee175bd1ce7de88b81c625a05a23887e5780e790df925
    source_path: cli/plugins.md
    workflow: 15
---

# `openclaw plugins`

إدارة plugins/امتدادات البوابة، وحزم الخطافات، والحزم المتوافقة.

ذو صلة:

- نظام plugin: [Plugins](/tools/plugin)
- توافق الحزم: [حزم Plugin](/plugins/bundles)
- manifest ومخطط plugin: [manifest الـ Plugin](/plugins/manifest)
- التحصين الأمني: [الأمان](/gateway/security)

## الأوامر

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins install <path-or-spec>
openclaw plugins inspect <id>
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
openclaw plugins info <id>
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id>
openclaw plugins doctor
openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins marketplace list <marketplace>
openclaw plugins marketplace list <marketplace> --json
```

تأتي plugins المضمّنة مع OpenClaw. ويكون بعضها مفعّلًا افتراضيًا (على سبيل المثال
موفرو النماذج المضمّنون، وموفرو الكلام المضمّنون، وplugin المتصفح
المضمّن)؛ بينما يتطلب بعضها الآخر `plugins enable`.

يجب أن تتضمن plugins OpenClaw الأصلية ملف `openclaw.plugin.json` مع JSON
Schema مضمّن (`configSchema`، حتى لو كان فارغًا). أما الحزم المتوافقة فتستخدم manifests الحزم الخاصة بها بدلًا من ذلك.

يعرض `plugins list` القيمة `Format: openclaw` أو `Format: bundle`. كما يعرض إخراج
السرد/المعلومات المفصل أيضًا النوع الفرعي للحزمة (`codex` أو `claude` أو `cursor`) بالإضافة إلى
إمكانات الحزمة المكتشفة.

### التثبيت

```bash
openclaw plugins install <package>                      # ClawHub أولًا، ثم npm
openclaw plugins install clawhub:<package>              # ClawHub فقط
openclaw plugins install <package> --force              # الكتابة فوق التثبيت الموجود
openclaw plugins install <package> --pin                # تثبيت الإصدار
openclaw plugins install <package> --dangerously-force-unsafe-install
openclaw plugins install <path>                         # مسار محلي
openclaw plugins install <plugin>@<marketplace>         # marketplace
openclaw plugins install <plugin> --marketplace <name>  # marketplace (صريح)
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
```

يتم التحقق من أسماء الحزم المجردة في ClawHub أولًا، ثم npm. ملاحظة أمنية:
تعامل مع تثبيت plugins كما لو كنت تشغّل شيفرة. ويُفضَّل استخدام الإصدارات المثبتة.

إذا كان الإعداد غير صالح، فإن `plugins install` يفشل عادةً بشكل مغلق ويطلب منك
تشغيل `openclaw doctor --fix` أولًا. والاستثناء الموثق الوحيد هو مسار استرداد ضيق
لـ plugin مضمّن بالنسبة إلى plugins التي تختار صراحةً
`openclaw.install.allowInvalidConfigRecovery`.

يعيد `--force` استخدام هدف التثبيت الحالي ويكتب فوق plugin أو حزمة خطافات
مثبّتة بالفعل في مكانها. استخدمه عندما تكون تعيد عمدًا تثبيت
المعرّف نفسه من مسار محلي جديد، أو أرشيف، أو حزمة ClawHub، أو أثر npm.

ينطبق `--pin` على تثبيتات npm فقط. وهو غير مدعوم مع `--marketplace`،
لأن تثبيتات marketplace تحتفظ ببيانات وصفية عن مصدر marketplace بدلًا من
مواصفة npm.

الخيار `--dangerously-force-unsafe-install` هو خيار طوارئ للحالات الإيجابية الكاذبة
في ماسح الشيفرة الخطرة المضمّن. فهو يسمح بمتابعة التثبيت حتى
عندما يبلغ الماسح المضمّن عن نتائج `critical`، لكنه **لا** يتجاوز حظر سياسة خطاف
plugin ‏`before_install`، كما أنه **لا** يتجاوز
إخفاقات الفحص.

ينطبق هذا الخيار في CLI على تدفقات تثبيت/تحديث plugin. أما تثبيتات تبعيات
Skills المدعومة بالبوابة فتستخدم التجاوز المطابق للطلب `dangerouslyForceUnsafeInstall`، بينما يظل `openclaw skills install` تدفق تنزيل/تثبيت Skills منفصلًا عبر ClawHub.

يُعد `plugins install` أيضًا سطح التثبيت لحزم الخطافات التي تكشف
`openclaw.hooks` في `package.json`. استخدم `openclaw hooks` للحصول على رؤية
الخطافات المصفّاة والتفعيل لكل خطاف، وليس لتثبيت الحزم.

تكون مواصفات npm **خاصة بالسجل فقط** (اسم الحزمة + **إصدار دقيق** اختياري أو
**dist-tag**). ويتم رفض مواصفات Git/URL/file ونطاقات semver. كما تُشغَّل
عمليات تثبيت التبعيات باستخدام `--ignore-scripts` لأسباب تتعلق بالأمان.

تبقى المواصفات المجردة و`@latest` على المسار المستقر. وإذا قام npm بحل أيٍّ من
هذين إلى إصدار prerelease، فسيتوقف OpenClaw ويطلب منك الاشتراك صراحةً باستخدام
وسم prerelease مثل `@beta`/`@rc` أو إصدار prerelease دقيق مثل
`@1.2.3-beta.4`.

إذا طابقت مواصفة تثبيت مجردة معرّف plugin مضمّنًا (على سبيل المثال `diffs`)،
فسيقوم OpenClaw بتثبيت plugin المضمّن مباشرةً. ولتثبيت حزمة npm بالاسم
نفسه، استخدم مواصفة ذات نطاق صريح (على سبيل المثال `@scope/diffs`).

الأرشيفات المدعومة: `.zip` و`.tgz` و`.tar.gz` و`.tar`.

كما أن تثبيتات Claude marketplace مدعومة أيضًا.

تستخدم تثبيتات ClawHub محدِّد `clawhub:<package>` الصريح:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

يفضّل OpenClaw الآن أيضًا ClawHub بالنسبة إلى مواصفات plugin المجردة والصالحة لـ npm. ولا
يرجع إلى npm إلا إذا لم يكن ClawHub يملك تلك الحزمة أو ذلك الإصدار:

```bash
openclaw plugins install openclaw-codex-app-server
```

يقوم OpenClaw بتنزيل أرشيف الحزمة من ClawHub، والتحقق من توافق
plugin API المُعلن / الحد الأدنى لتوافق البوابة، ثم يثبّتها عبر مسار
الأرشيف العادي. وتحتفظ التثبيتات المسجلة ببياناتها الوصفية الخاصة بمصدر ClawHub من أجل التحديثات اللاحقة.

استخدم الصيغة المختصرة `plugin@marketplace` عندما يكون اسم marketplace موجودًا في
ذاكرة السجل المحلية الخاصة بـ Claude في `~/.claude/plugins/known_marketplaces.json`:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

استخدم `--marketplace` عندما تريد تمرير مصدر marketplace بشكل صريح:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

يمكن أن تكون مصادر marketplace:

- اسم known-marketplace خاص بـ Claude من `~/.claude/plugins/known_marketplaces.json`
- جذر marketplace محلي أو مسار `marketplace.json`
- صيغة مختصرة لمستودع GitHub مثل `owner/repo`
- عنوان URL لمستودع GitHub مثل `https://github.com/owner/repo`
- عنوان git URL

بالنسبة إلى marketplaces البعيدة المحمّلة من GitHub أو git، يجب أن تبقى إدخالات plugin
داخل مستودع marketplace المستنسخ. ويقبل OpenClaw مصادر المسارات النسبية من
ذلك المستودع، ويرفض HTTP(S) والمسارات المطلقة وgit وGitHub وغيرها من
مصادر plugin غير المعتمدة على المسار من manifests البعيدة.

بالنسبة إلى المسارات والأرشيفات المحلية، يكتشف OpenClaw تلقائيًا:

- plugins OpenClaw الأصلية (`openclaw.plugin.json`)
- حزم متوافقة مع Codex (`.codex-plugin/plugin.json`)
- حزم متوافقة مع Claude (`.claude-plugin/plugin.json` أو تخطيط
  مكونات Claude الافتراضي)
- حزم متوافقة مع Cursor (`.cursor-plugin/plugin.json`)

يتم تثبيت الحزم المتوافقة في جذر الامتدادات العادي وتشارك في التدفق نفسه الخاص
بالسرد/المعلومات/التفعيل/التعطيل. وحاليًا، فإن bundle Skills، وClaude
command-skills، وقيم `settings.json` الافتراضية الخاصة بـ Claude، وقيم Claude
الافتراضية الخاصة بـ `.lsp.json` / `lspServers` المعلنة في manifest، وCursor command-skills، وأدلة خطافات Codex المتوافقة مدعومة؛ أما إمكانات الحزم الأخرى المكتشفة فتظهر في التشخيصات/المعلومات لكنها لم تُوصَل بعد بتنفيذ وقت التشغيل.

### السرد

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

استخدم `--enabled` لعرض plugins المحمّلة فقط. واستخدم `--verbose` للتبديل من
عرض الجدول إلى أسطر تفاصيل لكل plugin مع بيانات
المصدر/المنشأ/الإصدار/التفعيل. واستخدم `--json` للحصول على مخزون مقروء آليًا بالإضافة إلى
تشخيصات السجل.

استخدم `--link` لتجنب نسخ دليل محلي (يضيفه إلى `plugins.load.paths`):

```bash
openclaw plugins install -l ./my-plugin
```

الخيار `--force` غير مدعوم مع `--link` لأن التثبيتات المرتبطة تعيد استخدام
المسار المصدر بدلًا من النسخ فوق هدف تثبيت مُدار.

استخدم `--pin` في تثبيتات npm لحفظ المواصفة الدقيقة المحلولة (`name@version`) في
`plugins.installs` مع إبقاء السلوك الافتراضي غير مثبت.

### إلغاء التثبيت

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
```

يقوم `uninstall` بإزالة سجلات plugin من `plugins.entries` و`plugins.installs`،
وقائمة سماح plugin، وإدخالات `plugins.load.paths` المرتبطة عند الاقتضاء.
وبالنسبة إلى plugins الذاكرة النشطة، تتم إعادة تعيين خانة الذاكرة إلى `memory-core`.

افتراضيًا، يزيل إلغاء التثبيت أيضًا دليل تثبيت plugin الموجود تحت جذر plugins
لدليل الحالة النشط. استخدم
`--keep-files` للإبقاء على الملفات على القرص.

يُدعَم `--keep-config` كاسم مستعار مهمل لصالح `--keep-files`.

### التحديث

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call@beta
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

تنطبق التحديثات على التثبيتات المتعقبة في `plugins.installs` وعلى
تثبيتات حزم الخطافات المتعقبة في `hooks.internal.installs`.

عندما تمرر معرّف plugin، يعيد OpenClaw استخدام مواصفة التثبيت المسجلة لذلك
plugin. وهذا يعني أن dist-tags المخزنة سابقًا مثل `@beta` والإصدارات
الدقيقة المثبتة تظل مستخدمة في تشغيلات `update <id>` اللاحقة.

بالنسبة إلى تثبيتات npm، يمكنك أيضًا تمرير مواصفة صريحة لحزمة npm تتضمن dist-tag
أو إصدارًا دقيقًا. ويقوم OpenClaw بحل اسم الحزمة هذا مرة أخرى إلى سجل plugin
المتعقب، ويحدّث ذلك plugin المثبّت، ويسجل مواصفة npm الجديدة من أجل التحديثات
المستقبلية المعتمدة على المعرّف.

عند وجود قيمة hash سلامة مخزنة وتغيّر hash الأثر الذي تم جلبه،
يطبع OpenClaw تحذيرًا ويطلب التأكيد قبل المتابعة. استخدم
القيمة العامة `--yes` لتجاوز المطالبات في تشغيلات CI/غير التفاعلية.

يتوفر أيضًا `--dangerously-force-unsafe-install` في `plugins update` كتجاوز
طوارئ للحالات الإيجابية الكاذبة في فحص الشيفرة الخطرة المضمّن أثناء
تحديثات plugin. ومع ذلك، فهو لا يزال لا يتجاوز حظر سياسات `before_install` الخاصة بـ plugin
أو حظر فشل الفحص، كما أنه ينطبق فقط على تحديثات plugin وليس على تحديثات
حزم الخطافات.

### الفحص

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --json
```

فحص عميق لـ plugin واحد. يعرض الهوية، وحالة التحميل، والمصدر،
والإمكانات المسجلة، والخطافات، والأدوات، والأوامر، والخدمات، وطرق البوابة،
ومسارات HTTP، وأعلام السياسات، والتشخيصات، وبيانات التثبيت الوصفية، وإمكانات الحزم،
وأي دعم مكتشف لخوادم MCP أو LSP.

يتم تصنيف كل plugin بحسب ما يسجله فعليًا في وقت التشغيل:

- **plain-capability** — نوع إمكانية واحد (مثل plugin يوفّر provider فقط)
- **hybrid-capability** — أنواع إمكانيات متعددة (مثل النص + الكلام + الصور)
- **hook-only** — خطافات فقط، من دون إمكانيات أو أسطح
- **non-capability** — أدوات/أوامر/خدمات ولكن من دون إمكانيات

راجع [أشكال Plugin](/plugins/architecture#plugin-shapes) لمعرفة المزيد عن نموذج الإمكانيات.

يُخرج الخيار `--json` تقريرًا مقروءًا آليًا مناسبًا للبرمجة النصية
والتدقيق.

يعرض `inspect --all` جدولًا على مستوى الأسطول يتضمن الشكل، وأنواع
الإمكانيات، وملاحظات التوافق، وإمكانات الحزم، وأعمدة ملخص الخطافات.

`info` هو اسم مستعار لـ `inspect`.

### Doctor

```bash
openclaw plugins doctor
```

يعرض `doctor` أخطاء تحميل plugin، وتشخيصات manifest/الاكتشاف، و
ملاحظات التوافق. وعندما يكون كل شيء سليمًا فإنه يطبع `No plugin issues
detected.`

### Marketplace

```bash
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
```

يقبل أمر marketplace list مسار marketplace محلي، أو مسار `marketplace.json`، أو
صيغة GitHub المختصرة مثل `owner/repo`، أو عنوان URL لمستودع GitHub، أو عنوان git URL. ويقوم `--json`
بطباعة تسمية المصدر المحلولة بالإضافة إلى manifest الـ marketplace المحلّل
وإدخالات plugin.
