---
read_when:
    - عند تقديم ClawHub إلى المستخدمين الجدد
    - عند تثبيت Skills أو plugins أو البحث عنها أو نشرها
    - عند شرح أعلام ClawHub CLI وسلوك المزامنة
summary: 'دليل ClawHub: السجل العام، وتدفقات التثبيت الأصلية في OpenClaw، وتدفقات عمل ClawHub CLI'
title: ClawHub
x-i18n:
    generated_at: "2026-04-05T12:58:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: e65b3fd770ca96a5dd828dce2dee4ef127268f4884180a912f43d7744bc5706f
    source_path: tools/clawhub.md
    workflow: 15
---

# ClawHub

ClawHub هو السجل العام لـ **Skills وplugins الخاصة بـ OpenClaw**.

- استخدم أوامر `openclaw` الأصلية للبحث عن Skills وتثبيتها وتحديثها، وتثبيت
  plugins من ClawHub.
- استخدم CLI المنفصل `clawhub` عندما تحتاج إلى مصادقة السجل، أو النشر، أو الحذف،
  أو التراجع عن الحذف، أو تدفقات المزامنة.

الموقع: [clawhub.ai](https://clawhub.ai)

## تدفقات OpenClaw الأصلية

Skills:

```bash
openclaw skills search "calendar"
openclaw skills install <skill-slug>
openclaw skills update --all
```

Plugins:

```bash
openclaw plugins install clawhub:<package>
openclaw plugins update --all
```

تُجرَّب أيضًا مواصفات plugin العارية الآمنة مع npm على ClawHub قبل npm:

```bash
openclaw plugins install openclaw-codex-app-server
```

تُثبِّت أوامر `openclaw` الأصلية في مساحة العمل النشطة لديك وتحفظ بيانات
التعريف الخاصة بالمصدر بحيث تتمكن استدعاءات `update` اللاحقة من البقاء على ClawHub.

تتحقق عمليات تثبيت plugin من توافق `pluginApi` و`minGatewayVersion`
المعلنَين قبل تشغيل تثبيت الأرشيف، بحيث تفشل المضيفات غير المتوافقة بشكل مغلق
مبكرًا بدلًا من تثبيت الحزمة جزئيًا.

لا يقبل `openclaw plugins install clawhub:...` إلا عائلات plugins القابلة للتثبيت.
إذا كانت حزمة ClawHub في الواقع Skill، فسيتوقف OpenClaw ويوجهك إلى
`openclaw skills install <slug>` بدلًا من ذلك.

## ما هو ClawHub

- سجل عام لـ Skills وplugins الخاصة بـ OpenClaw.
- مخزن بإصدارات لحِزم Skills وبياناتها الوصفية.
- واجهة اكتشاف للبحث، والوسوم، وإشارات الاستخدام.

## كيف يعمل

1. ينشر المستخدم حزمة Skill ‏(ملفات + بيانات وصفية).
2. يخزن ClawHub الحزمة، ويحلل البيانات الوصفية، ويُسنِد إصدارًا.
3. يفهرس السجل Skill من أجل البحث والاكتشاف.
4. يتصفح المستخدمون Skills وينزّلونها ويثبتونها في OpenClaw.

## ما الذي يمكنك فعله

- نشر Skills جديدة وإصدارات جديدة من Skills الحالية.
- اكتشاف Skills بالاسم أو الوسوم أو البحث.
- تنزيل حِزم Skills وفحص ملفاتها.
- الإبلاغ عن Skills المسيئة أو غير الآمنة.
- إذا كنت مشرفًا، يمكنك إخفاء العناصر أو إلغاء إخفائها أو حذفها أو حظرها.

## لمن هذا الدليل (سهل للمبتدئين)

إذا كنت تريد إضافة قدرات جديدة إلى وكيل OpenClaw الخاص بك، فإن ClawHub هو أسهل طريقة للعثور على Skills وتثبيتها. لا تحتاج إلى معرفة كيفية عمل الواجهة الخلفية. يمكنك:

- البحث عن Skills بلغة بسيطة.
- تثبيت Skill في مساحة العمل الخاصة بك.
- تحديث Skills لاحقًا بأمر واحد.
- نسخ Skills الخاصة بك احتياطيًا عن طريق نشرها.

## البداية السريعة (غير التقنية)

1. ابحث عن شيء تحتاج إليه:
   - `openclaw skills search "calendar"`
2. ثبّت Skill:
   - `openclaw skills install <skill-slug>`
3. ابدأ جلسة OpenClaw جديدة حتى يلتقط Skill الجديد.
4. إذا كنت تريد النشر أو إدارة مصادقة السجل، فثبّت CLI المنفصل
   `clawhub` أيضًا.

## تثبيت ClawHub CLI

لن تحتاج إليه إلا لتدفقات العمل التي تتطلب مصادقة السجل مثل النشر/المزامنة:

```bash
npm i -g clawhub
```

```bash
pnpm add -g clawhub
```

## كيف يندمج مع OpenClaw

يقوم `openclaw skills install` الأصلي بالتثبيت داخل دليل `skills/`
في مساحة العمل النشطة. ويسجل `openclaw plugins install clawhub:...` عملية
تثبيت plugin مُدارة عادية بالإضافة إلى بيانات تعريف مصدر ClawHub للتحديثات.

كما أن عمليات تثبيت plugin المجهولة من ClawHub تفشل بشكل مغلق للحزم الخاصة.
ولا يزال بإمكان القنوات المجتمعية أو غير الرسمية الأخرى التثبيت، لكن OpenClaw يحذر
حتى يتمكن المشغلون من مراجعة المصدر والتحقق قبل تمكينها.

كما يقوم CLI المنفصل `clawhub` أيضًا بتثبيت Skills في `./skills` داخل
دليل العمل الحالي. وإذا كانت مساحة عمل OpenClaw مهيأة، فإن `clawhub`
يعود إلى استخدام تلك المساحة ما لم تتجاوز ذلك باستخدام `--workdir` (أو
`CLAWHUB_WORKDIR`). يحمّل OpenClaw Skills مساحة العمل من `<workspace>/skills`
وسيلتقطها في الجلسة **التالية**. وإذا كنت تستخدم بالفعل
`~/.openclaw/skills` أو Skills المضمنة، فإن Skills مساحة العمل لها أولوية.

لمزيد من التفاصيل حول كيفية تحميل Skills ومشاركتها والتحكم فيها، راجع
[Skills](/tools/skills).

## نظرة عامة على نظام Skills

Skill هي حزمة ملفات ذات إصدار تعلّم OpenClaw كيفية تنفيذ
مهمة محددة. وينشئ كل نشر إصدارًا جديدًا، ويحتفظ السجل
بسجل الإصدارات بحيث يتمكن المستخدمون من تدقيق التغييرات.

تتضمن Skill النموذجية ما يلي:

- ملف `SKILL.md` يحتوي على الوصف الرئيسي وطريقة الاستخدام.
- إعدادات أو سكربتات أو ملفات مساعدة اختيارية تستخدمها Skill.
- بيانات وصفية مثل الوسوم، والملخص، ومتطلبات التثبيت.

يستخدم ClawHub البيانات الوصفية لتشغيل الاكتشاف وعرض قدرات Skills بأمان.
كما يتتبع السجل أيضًا إشارات الاستخدام (مثل النجوم وعمليات التنزيل) لتحسين
الترتيب وإمكانية الظهور.

## ما الذي توفره الخدمة (الميزات)

- **تصفح عام** لـ Skills ومحتوى `SKILL.md` الخاص بها.
- **بحث** مدعوم بالتضمينات (بحث متجهي)، وليس بالكلمات المفتاحية فقط.
- **إصدار** باستخدام semver، وسجلات التغيير، والوسوم (بما في ذلك `latest`).
- **تنزيلات** على هيئة zip لكل إصدار.
- **النجوم والتعليقات** للحصول على ملاحظات المجتمع.
- **خطافات الإشراف** للموافقات وعمليات التدقيق.
- **API مناسب لـ CLI** للأتمتة والسكربتات.

## الأمان والإشراف

ClawHub مفتوح افتراضيًا. يمكن لأي شخص رفع Skills، لكن يجب أن يكون حساب GitHub
عمره أسبوعًا واحدًا على الأقل حتى يتمكن من النشر. ويساعد هذا على إبطاء الإساءة دون حظر
المساهمين الشرعيين.

الإبلاغ والإشراف:

- يمكن لأي مستخدم مسجل الدخول الإبلاغ عن Skill.
- أسباب الإبلاغ مطلوبة وتُسجَّل.
- يمكن لكل مستخدم أن يملك ما يصل إلى 20 بلاغًا نشطًا في الوقت نفسه.
- تُخفى Skills التي لديها أكثر من 3 بلاغات فريدة تلقائيًا افتراضيًا.
- يمكن للمشرفين عرض Skills المخفية، أو إلغاء إخفائها، أو حذفها، أو حظر المستخدمين.
- يمكن أن يؤدي إساءة استخدام ميزة الإبلاغ إلى حظر الحساب.

هل تهتم بأن تصبح مشرفًا؟ اسأل في Discord الخاص بـ OpenClaw وتواصل مع
أحد المشرفين أو المشرفين المسؤولين.

## أوامر CLI والمعلمات

الخيارات العامة (تنطبق على جميع الأوامر):

- `--workdir <dir>`: دليل العمل (الافتراضي: الدليل الحالي؛ ويعود إلى مساحة عمل OpenClaw).
- `--dir <dir>`: دليل Skills، نسبي إلى workdir ‏(الافتراضي: `skills`).
- `--site <url>`: عنوان URL الأساسي للموقع (تسجيل الدخول عبر المتصفح).
- `--registry <url>`: عنوان URL الأساسي لـ API السجل.
- `--no-input`: تعطيل المطالبات (غير تفاعلي).
- `-V, --cli-version`: طباعة إصدار CLI.

المصادقة:

- `clawhub login` ‏(تدفق المتصفح) أو `clawhub login --token <token>`
- `clawhub logout`
- `clawhub whoami`

الخيارات:

- `--token <token>`: الصق رمز API مميزًا.
- `--label <label>`: التسمية المخزنة لرموز تسجيل الدخول عبر المتصفح (الافتراضي: `CLI token`).
- `--no-browser`: عدم فتح متصفح (يتطلب `--token`).

البحث:

- `clawhub search "query"`
- `--limit <n>`: الحد الأقصى للنتائج.

التثبيت:

- `clawhub install <slug>`
- `--version <version>`: تثبيت إصدار محدد.
- `--force`: الكتابة فوق المجلد إذا كان موجودًا بالفعل.

التحديث:

- `clawhub update <slug>`
- `clawhub update --all`
- `--version <version>`: التحديث إلى إصدار محدد (لـ slug واحد فقط).
- `--force`: الكتابة فوق الملفات عندما لا تتطابق الملفات المحلية مع أي إصدار منشور.

القائمة:

- `clawhub list` ‏(يقرأ `.clawhub/lock.json`)

نشر Skills:

- `clawhub skill publish <path>`
- `--slug <slug>`: معرّف Skill.
- `--name <name>`: اسم العرض.
- `--version <version>`: إصدار semver.
- `--changelog <text>`: نص سجل التغيير (يمكن أن يكون فارغًا).
- `--tags <tags>`: وسوم مفصولة بفواصل (الافتراضي: `latest`).

نشر plugins:

- `clawhub package publish <source>`
- يمكن أن يكون `<source>` مجلدًا محليًا، أو `owner/repo`، أو `owner/repo@ref`، أو عنوان URL على GitHub.
- `--dry-run`: أنشئ خطة النشر الدقيقة دون رفع أي شيء.
- `--json`: أخرج مخرجات قابلة للقراءة آليًا لـ CI.
- `--source-repo`, `--source-commit`, `--source-ref`: تجاوزات اختيارية عندما لا يكون الاكتشاف التلقائي كافيًا.

الحذف/التراجع عن الحذف (للمالك/المشرف فقط):

- `clawhub delete <slug> --yes`
- `clawhub undelete <slug> --yes`

المزامنة (فحص Skills المحلية + نشر الجديدة/المحدّثة):

- `clawhub sync`
- `--root <dir...>`: جذور فحص إضافية.
- `--all`: رفع كل شيء دون مطالبات.
- `--dry-run`: عرض ما سيتم رفعه.
- `--bump <type>`: ‏`patch|minor|major` للتحديثات (الافتراضي: `patch`).
- `--changelog <text>`: سجل التغيير للتحديثات غير التفاعلية.
- `--tags <tags>`: وسوم مفصولة بفواصل (الافتراضي: `latest`).
- `--concurrency <n>`: فحوصات السجل (الافتراضي: 4).

## تدفقات العمل الشائعة للوكلاء

### البحث عن Skills

```bash
clawhub search "postgres backups"
```

### تنزيل Skills جديدة

```bash
clawhub install my-skill-pack
```

### تحديث Skills المثبتة

```bash
clawhub update --all
```

### نسخ Skills الخاصة بك احتياطيًا (نشر أو مزامنة)

لمجلد Skill واحد:

```bash
clawhub skill publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0 --tags latest
```

لفحص عدد كبير من Skills ونسخها احتياطيًا دفعة واحدة:

```bash
clawhub sync --all
```

### نشر plugin من GitHub

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
clawhub package publish https://github.com/your-org/your-plugin
```

يجب أن تتضمن plugins البرمجية بيانات OpenClaw الوصفية المطلوبة في `package.json`:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

## تفاصيل متقدمة (تقنية)

### الإصدارات والوسوم

- ينشئ كل نشر `SkillVersion` جديدًا باستخدام **semver**.
- تشير الوسوم (مثل `latest`) إلى إصدار؛ ويتيح لك نقل الوسوم التراجع.
- تُرفق سجلات التغيير بكل إصدار، ويمكن أن تكون فارغة عند المزامنة أو نشر التحديثات.

### التغييرات المحلية مقابل إصدارات السجل

تقارن التحديثات محتويات Skill المحلية بإصدارات السجل باستخدام تجزئة محتوى. إذا لم تتطابق الملفات المحلية مع أي إصدار منشور، يسأل CLI قبل الكتابة فوقها (أو يتطلب `--force` في عمليات التشغيل غير التفاعلية).

### فحص المزامنة والجذور الاحتياطية

يفحص `clawhub sync` دليل العمل الحالي أولًا. وإذا لم يعثر على أي Skills، فإنه يعود إلى مواقع قديمة معروفة (على سبيل المثال `~/openclaw/skills` و`~/.openclaw/skills`). وقد صُمم ذلك للعثور على عمليات تثبيت Skills الأقدم دون الحاجة إلى أعلام إضافية.

### التخزين وملف القفل

- تُسجَّل Skills المثبتة في `.clawhub/lock.json` ضمن workdir الخاص بك.
- تُخزَّن رموز المصادقة المميزة في ملف تكوين ClawHub CLI ‏(يمكن التجاوز عبر `CLAWHUB_CONFIG_PATH`).

### القياس عن بُعد (عدادات التثبيت)

عندما تشغّل `clawhub sync` أثناء تسجيل الدخول، يرسل CLI لقطة دنيا لحساب أعداد التثبيت. يمكنك تعطيل ذلك بالكامل:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

## متغيرات البيئة

- `CLAWHUB_SITE`: تجاوز عنوان URL الخاص بالموقع.
- `CLAWHUB_REGISTRY`: تجاوز عنوان URL الخاص بـ API السجل.
- `CLAWHUB_CONFIG_PATH`: تجاوز مكان تخزين CLI للرمز/التكوين.
- `CLAWHUB_WORKDIR`: تجاوز workdir الافتراضي.
- `CLAWHUB_DISABLE_TELEMETRY=1`: تعطيل القياس عن بُعد في `sync`.
