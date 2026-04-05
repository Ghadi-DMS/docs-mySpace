---
read_when:
    - إضافة Skills أو تعديلها
    - تغيير قواعد تقييد Skills أو تحميلها
summary: 'Skills: المُدارة مقابل مساحة العمل، وقواعد التقييد، وربط التهيئة/البيئة'
title: Skills
x-i18n:
    generated_at: "2026-04-05T13:00:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6bb0e2e7c2ff50cf19c759ea1da1fd1886dc11f94adc77cbfd816009f75d93ee
    source_path: tools/skills.md
    workflow: 15
---

# Skills ‏(OpenClaw)

يستخدم OpenClaw مجلدات Skills متوافقة مع **[AgentSkills](https://agentskills.io)** لتعليم الوكيل كيفية استخدام الأدوات. كل Skill هو دليل يحتوي على `SKILL.md` يتضمن YAML frontmatter وتعليمات. يحمّل OpenClaw **Skills المجمّعة** بالإضافة إلى تجاوزات محلية اختيارية، ويقوم بتصفيتها وقت التحميل استنادًا إلى البيئة، والتهيئة، ووجود الملفات التنفيذية.

## المواقع والأولوية

يحمّل OpenClaw Skills من هذه المصادر:

1. **مجلدات Skills إضافية**: تُهيَّأ عبر `skills.load.extraDirs`
2. **Skills المجمّعة**: تُشحن مع التثبيت (حزمة npm أو OpenClaw.app)
3. **Skills المُدارة/المحلية**: `~/.openclaw/skills`
4. **Skills الوكيل الشخصية**: `~/.agents/skills`
5. **Skills وكيل المشروع**: `<workspace>/.agents/skills`
6. **Skills مساحة العمل**: `<workspace>/skills`

إذا تعارض اسم Skill، تكون الأولوية كالتالي:

`<workspace>/skills` (الأعلى) ← `<workspace>/.agents/skills` ← `~/.agents/skills` ← `~/.openclaw/skills` ← Skills المجمّعة ← `skills.load.extraDirs` (الأدنى)

## Skills لكل وكيل مقابل Skills المشتركة

في إعدادات **الوكلاء المتعددين**، يمتلك كل وكيل مساحة عمل خاصة به. وهذا يعني:

- توجد **Skills الخاصة بكل وكيل** في `<workspace>/skills` لذلك الوكيل فقط.
- توجد **Skills وكيل المشروع** في `<workspace>/.agents/skills` وتُطبَّق على
  مساحة العمل تلك قبل مجلد `skills/` العادي الخاص بمساحة العمل.
- توجد **Skills الوكيل الشخصية** في `~/.agents/skills` وتُطبَّق عبر
  مساحات العمل على ذلك الجهاز.
- توجد **Skills المشتركة** في `~/.openclaw/skills` (المُدارة/المحلية) وتكون مرئية
  إلى **جميع الوكلاء** على الجهاز نفسه.
- يمكن أيضًا إضافة **مجلدات مشتركة** عبر `skills.load.extraDirs` (بأدنى
  أولوية) إذا كنت تريد حزمة Skills مشتركة يستخدمها عدة وكلاء.

إذا وُجد اسم Skill نفسه في أكثر من مكان، فتنطبق الأولوية المعتادة:
تفوز مساحة العمل، ثم Skills وكيل المشروع، ثم Skills الوكيل الشخصية،
ثم المُدارة/المحلية، ثم المجمّعة، ثم الأدلة الإضافية.

## قوائم السماح الخاصة بـ Skills لكل وكيل

**موقع** Skill و**إمكانية رؤيته** عنصران منفصلان للتحكم.

- يحدد الموقع/الأولوية أي نسخة من Skill بالاسم نفسه تفوز.
- وتحدد قوائم السماح الخاصة بالوكيل أي Skills مرئية يمكن للوكيل استخدامها فعليًا.

استخدم `agents.defaults.skills` كأساس مشترك، ثم تجاوز لكل وكيل عبر
`agents.list[].skills`:

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

القواعد:

- احذف `agents.defaults.skills` إذا أردت Skills غير مقيّدة افتراضيًا.
- احذف `agents.list[].skills` للوراثة من `agents.defaults.skills`.
- عيّن `agents.list[].skills: []` لعدم استخدام أي Skills.
- تمثل القائمة غير الفارغة في `agents.list[].skills` المجموعة النهائية لذلك الوكيل؛
  ولا تندمج مع القيم الافتراضية.

يطبّق OpenClaw مجموعة Skills الفعّالة الخاصة بالوكيل عبر بناء prompt،
واكتشاف أوامر الشرطة المائلة الخاصة بـ Skill، والمزامنة في sandbox، ولقطات Skills.

## Plugins + Skills

يمكن أن تشحن Plugins Skills خاصة بها عن طريق إدراج أدلة `skills` في
`openclaw.plugin.json` (مسارات نسبةً إلى جذر plugin). يتم تحميل Skills الخاصة بالـ Plugin
عند تمكين plugin. حاليًا، تُدمج هذه الأدلة في نفس المسار منخفض الأولوية الخاص بـ `skills.load.extraDirs`، لذلك فإن Skillًا مجمّعًا أو مُدارًا أو خاصًا بالوكيل أو بمساحة العمل يحمل الاسم نفسه سيتجاوزها.
يمكنك تقييدها عبر `metadata.openclaw.requires.config` في إدخال تهيئة الـ plugin.
راجع [Plugins](/tools/plugin) للاكتشاف/التهيئة و[الأدوات](/tools) لسطح
الأدوات الذي تعلّمه تلك Skills.

## ClawHub ‏(التثبيت + المزامنة)

ClawHub هو سجل Skills العام لـ OpenClaw. تصفحه على
[https://clawhub.ai](https://clawhub.ai). استخدم أوامر `openclaw skills`
الأصلية لاكتشاف Skills أو تثبيتها أو تحديثها، أو استخدم CLI المنفصل `clawhub` عندما
تحتاج إلى مسارات عمل النشر/المزامنة.
الدليل الكامل: [ClawHub](/tools/clawhub).

المسارات الشائعة:

- تثبيت Skill في مساحة عملك:
  - `openclaw skills install <skill-slug>`
- تحديث جميع Skills المثبتة:
  - `openclaw skills update --all`
- المزامنة (الفحص + نشر التحديثات):
  - `clawhub sync --all`

يقوم `openclaw skills install` الأصلي بالتثبيت في دليل `skills/` الخاص بمساحة العمل النشطة. كما يقوم CLI المنفصل `clawhub` بالتثبيت أيضًا في `./skills` تحت
دليل العمل الحالي (أو يعود إلى مساحة عمل OpenClaw المهيأة).
وسيلتقط OpenClaw ذلك باعتباره `<workspace>/skills` في الجلسة التالية.

## ملاحظات الأمان

- تعامل مع Skills التابعة لجهات خارجية باعتبارها **تعليمة برمجية غير موثوق بها**. اقرأها قبل تمكينها.
- فضّل التشغيل داخل sandbox للمدخلات غير الموثوق بها والأدوات الخطِرة. راجع [العزل](/ar/gateway/sandboxing).
- لا يقبل اكتشاف Skills في مساحة العمل والأدلة الإضافية إلا جذور Skills وملفات `SKILL.md` التي يبقى `realpath` المحلول الخاص بها داخل الجذر المهيأ.
- تقوم عمليات تثبيت تبعيات Skills المدعومة من Gateway (`skills.install`، والإعداد الأولي، وواجهة إعدادات Skills) بتشغيل ماسح التعليمة البرمجية الخطِرة المدمج قبل تنفيذ بيانات التثبيت الوصفية. وتحظر النتائج `critical` افتراضيًا ما لم يعيّن المستدعي صراحة تجاوز الخطورة؛ أما النتائج المريبة فتعرض تحذيرات فقط.
- يختلف `openclaw skills install <slug>` عن ذلك: فهو ينزّل مجلد Skill من ClawHub إلى مساحة العمل ولا يستخدم مسار بيانات التثبيت الوصفية المذكور أعلاه.
- يقوم `skills.entries.*.env` و`skills.entries.*.apiKey` بحقن الأسرار في عملية **المضيف** لذلك الدور الخاص بالوكيل
  (وليس في sandbox). أبقِ الأسرار خارج prompts والسجلات.
- للحصول على نموذج تهديد أشمل وقوائم تحقق، راجع [الأمان](/ar/gateway/security).

## التنسيق (متوافق مع AgentSkills وPi)

يجب أن يتضمن `SKILL.md` على الأقل:

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
---
```

ملاحظات:

- نتبع مواصفة AgentSkills من حيث التخطيط والغرض.
- يدعم المحلل المستخدم بواسطة الوكيل المضمّن مفاتيح frontmatter **أحادية السطر** فقط.
- يجب أن تكون `metadata` **كائن JSON أحادي السطر**.
- استخدم `{baseDir}` داخل التعليمات للإشارة إلى مسار مجلد Skill.
- مفاتيح frontmatter الاختيارية:
  - `homepage` — عنوان URL يظهر بوصفه “Website” في واجهة Skills في macOS (ومدعوم أيضًا عبر `metadata.openclaw.homepage`).
  - `user-invocable` — `true|false` (الافتراضي: `true`). عندما تكون `true`، يُعرَض Skill كأمر شرطة مائلة للمستخدم.
  - `disable-model-invocation` — `true|false` (الافتراضي: `false`). عندما تكون `true`، يُستبعد Skill من prompt الخاص بالنموذج (مع بقائه متاحًا عبر استدعاء المستخدم).
  - `command-dispatch` — `tool` (اختياري). عند تعيينه إلى `tool`، يتجاوز أمر الشرطة المائلة النموذج ويُرسَل مباشرة إلى أداة.
  - `command-tool` — اسم الأداة المطلوب استدعاؤها عند تعيين `command-dispatch: tool`.
  - `command-arg-mode` — `raw` (الافتراضي). بالنسبة إلى إرسال الأداة، يمرّر سلسلة الوسائط الخام إلى الأداة (من دون تحليل في core).

    تُستدعى الأداة بالمعلمات التالية:
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.

## التقييد (مرشحات وقت التحميل)

يقوم OpenClaw **بتصفية Skills وقت التحميل** باستخدام `metadata` ‏(JSON أحادي السطر):

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

الحقول تحت `metadata.openclaw`:

- `always: true` — تضمين Skill دائمًا (وتخطي بقية القيود).
- `emoji` — رمز تعبيري اختياري تستخدمه واجهة Skills في macOS.
- `homepage` — عنوان URL اختياري يُعرض بصفة “Website” في واجهة Skills في macOS.
- `os` — قائمة اختيارية بالمنصات (`darwin` و`linux` و`win32`). إذا تم تعيينها، يكون Skill مؤهلًا فقط على أنظمة التشغيل تلك.
- `requires.bins` — قائمة؛ يجب أن يوجد كل عنصر منها على `PATH`.
- `requires.anyBins` — قائمة؛ يجب أن يوجد عنصر واحد منها على الأقل على `PATH`.
- `requires.env` — قائمة؛ يجب أن يوجد متغير البيئة **أو** أن يُوفَّر في التهيئة.
- `requires.config` — قائمة بمسارات `openclaw.json` التي يجب أن تكون truthy.
- `primaryEnv` — اسم متغير البيئة المرتبط بـ `skills.entries.<name>.apiKey`.
- `install` — مصفوفة اختيارية من مواصفات التثبيت تستخدمها واجهة Skills في macOS (brew/node/go/uv/download).

ملاحظة حول sandboxing:

- يتم التحقق من `requires.bins` على **المضيف** عند تحميل Skill.
- إذا كان الوكيل يعمل داخل sandbox، فيجب أن يكون الملف التنفيذي موجودًا أيضًا **داخل الحاوية**.
  قم بتثبيته عبر `agents.defaults.sandbox.docker.setupCommand` (أو صورة مخصصة).
  يتم تشغيل `setupCommand` مرة واحدة بعد إنشاء الحاوية.
  تتطلب عمليات تثبيت الحزم أيضًا خروجًا للشبكة، ونظام ملفات جذر قابلًا للكتابة، ومستخدم root داخل sandbox.
  مثال: يحتاج Skill ‏`summarize` ‏(`skills/summarize/SKILL.md`) إلى CLI ‏`summarize`
  داخل حاوية sandbox لكي يعمل هناك.

مثال على التثبيت:

```markdown
---
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Install Gemini CLI (brew)",
            },
          ],
      },
  }
---
```

ملاحظات:

- إذا أُدرجت عدة أدوات تثبيت، يختار gateway **خيارًا مفضلًا واحدًا** (brew عند توفره، وإلا node).
- إذا كانت جميع أدوات التثبيت من نوع `download`، يعرض OpenClaw كل إدخال لتتمكن من رؤية العناصر المتاحة.
- يمكن أن تتضمن مواصفات أداة التثبيت `os: ["darwin"|"linux"|"win32"]` لتصفية الخيارات حسب المنصة.
- تراعي عمليات تثبيت Node القيمة `skills.install.nodeManager` في `openclaw.json` (الافتراضي: npm؛ والخيارات: npm/pnpm/yarn/bun).
  يؤثر هذا فقط في **تثبيت Skills**؛ لكن Runtime الخاص بـ Gateway يجب أن يبقى Node
  (لا يُنصح باستخدام Bun مع WhatsApp/Telegram).
- يعتمد اختيار أداة التثبيت المدعوم من Gateway على التفضيل، وليس على node فقط:
  فعندما تمزج مواصفات التثبيت بين الأنواع، يفضّل OpenClaw Homebrew عندما
  تكون `skills.install.preferBrew` ممكّنة ويكون `brew` موجودًا، ثم `uv`، ثم
  مدير node المهيأ، ثم البدائل الأخرى مثل `go` أو `download`.
- إذا كانت كل مواصفات التثبيت من نوع `download`، فسيعرض OpenClaw جميع خيارات التنزيل
  بدلًا من طيّها في أداة تثبيت مفضلة واحدة.
- عمليات تثبيت Go: إذا كان `go` مفقودًا وكان `brew` متاحًا، يقوم gateway بتثبيت Go عبر Homebrew أولًا ويضبط `GOBIN` على دليل `bin` الخاص بـ Homebrew عندما يكون ذلك ممكنًا.
- عمليات تثبيت التنزيل: `url` ‏(مطلوب)، و`archive` ‏(`tar.gz` | `tar.bz2` | `zip`)، و`extract` ‏(الافتراضي: تلقائي عند اكتشاف الأرشيف)، و`stripComponents`، و`targetDir` ‏(الافتراضي: `~/.openclaw/tools/<skillKey>`).

إذا لم توجد `metadata.openclaw`، يكون Skill مؤهلًا دائمًا (ما لم
يُعطّل في التهيئة أو يُحظر بواسطة `skills.allowBundled` بالنسبة إلى Skills المجمّعة).

## تجاوزات التهيئة (`~/.openclaw/openclaw.json`)

يمكن تبديل Skills المجمّعة/المُدارة وتزويدها بقيم env:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

ملاحظة: إذا احتوى اسم Skill على شرطات، فضع المفتاح بين علامتي اقتباس (يدعم JSON5 المفاتيح المقتبسة).

إذا أردت توليد/تحرير صور قياسيًا داخل OpenClaw نفسه، فاستخدم
الأداة الأساسية `image_generate` مع `agents.defaults.imageGenerationModel` بدلًا من
Skill مجمّع. أمثلة Skills هنا مخصصة لمسارات عمل مخصصة أو تابعة لجهات خارجية.

لتحليل الصور الأصلي، استخدم أداة `image` مع `agents.defaults.imageModel`.
ولتوليد/تحرير الصور الأصلي، استخدم `image_generate` مع
`agents.defaults.imageGenerationModel`. إذا اخترت `openai/*` أو `google/*`
أو `fal/*` أو نموذج صور خاصًا بموفر آخر، فأضف أيضًا المصادقة/مفتاح API الخاص
بذلك الموفر.

تطابق مفاتيح التهيئة **اسم Skill** افتراضيًا. وإذا عرّف Skill
`metadata.openclaw.skillKey`، فاستخدم ذلك المفتاح تحت `skills.entries`.

القواعد:

- تؤدي `enabled: false` إلى تعطيل Skill حتى لو كان مجمّعًا/مثبتًا.
- `env`: يُحقن **فقط إذا** لم يكن المتغير مضبوطًا بالفعل في العملية.
- `apiKey`: وسيلة ملائمة للـ Skills التي تعلن `metadata.openclaw.primaryEnv`.
  وتدعم سلسلة نصية واضحة أو كائن SecretRef ‏(`{ source, provider, id }`).
- `config`: حاوية اختيارية لحقول مخصصة لكل Skill؛ ويجب أن توجد المفاتيح المخصصة هنا.
- `allowBundled`: قائمة سماح اختيارية لـ **Skills المجمّعة** فقط. إذا تم تعيينها، لا تكون مؤهلة
  إلا Skills المجمّعة الموجودة في القائمة (ولا تتأثر Skills المُدارة/مساحة العمل).

## حقن البيئة (لكل تشغيل وكيل)

عند بدء تشغيل وكيل، يقوم OpenClaw بما يلي:

1. قراءة بيانات Skill الوصفية.
2. تطبيق أي قيم `skills.entries.<key>.env` أو `skills.entries.<key>.apiKey` على
   `process.env`.
3. بناء system prompt باستخدام Skills **المؤهلة**.
4. استعادة البيئة الأصلية بعد انتهاء التشغيل.

هذا **مقيّد بتشغيل الوكيل**، وليس بيئة shell عامة.

## لقطة الجلسة (الأداء)

يلتقط OpenClaw Snapshot لـ Skills المؤهلة **عند بدء الجلسة** ويعيد استخدام تلك القائمة في الأدوار اللاحقة داخل الجلسة نفسها. تسري التغييرات على Skills أو التهيئة في الجلسة الجديدة التالية.

يمكن أيضًا تحديث Skills في منتصف الجلسة عند تمكين مراقب Skills أو عند ظهور node بعيد جديد مؤهل (انظر أدناه). فكّر في هذا على أنه **إعادة تحميل فورية**: تُلتقط القائمة المحدّثة في دور الوكيل التالي.

إذا تغيّرت قائمة السماح الفعّالة الخاصة بـ Skills لذلك الوكيل في تلك الجلسة، فسيقوم OpenClaw
بتحديث Snapshot حتى تبقى Skills المرئية متوافقة مع الوكيل الحالي.

## عُقد macOS البعيدة (Gateway على Linux)

إذا كان Gateway يعمل على Linux لكن **عقدة macOS** متصلة **ومسموح لها باستخدام `system.run`** (ولم تُضبط أمانات موافقات Exec على `deny`)، فيمكن لـ OpenClaw التعامل مع Skills الخاصة بـ macOS فقط على أنها مؤهلة عندما تكون الملفات التنفيذية المطلوبة موجودة على تلك العقدة. ينبغي للوكيل تنفيذ تلك Skills عبر أداة `exec` مع `host=node`.

يعتمد ذلك على أن تُبلغ العقدة عن دعم الأوامر الخاصة بها وعلى فحص bin عبر `system.run`. وإذا أصبحت عقدة macOS غير متصلة لاحقًا، فستبقى Skills مرئية؛ وقد تفشل عمليات الاستدعاء حتى تعيد العقدة الاتصال.

## مراقب Skills ‏(التحديث التلقائي)

افتراضيًا، يراقب OpenClaw مجلدات Skills ويزيد Snapshot الخاص بـ Skills عند تغيّر ملفات `SKILL.md`. هيّئ هذا تحت `skills.load`:

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250,
    },
  },
}
```

## أثر الرموز (قائمة Skills)

عندما تكون Skills مؤهلة، يقوم OpenClaw بحقن قائمة XML مضغوطة بالـ Skills المتاحة داخل system prompt (عبر `formatSkillsForPrompt` في `pi-coding-agent`). وتكون التكلفة حتمية:

- **العبء الأساسي (فقط عند وجود Skill واحد أو أكثر):** 195 حرفًا.
- **لكل Skill:** ‏97 حرفًا + طول القيم `<name>` و`<description>` و`<location>` بعد تهريب XML.

الصيغة (بالحروف):

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

ملاحظات:

- يوسّع تهريب XML الأحرف `& < > " '` إلى كيانات (`&amp;` و`&lt;` وما إلى ذلك)، مما يزيد الطول.
- يختلف عدد الرموز باختلاف tokenizer الخاص بالنموذج. كتقدير تقريبي على نمط OpenAI، يساوي الرمز الواحد نحو 4 أحرف، لذا فإن **97 حرفًا ≈ 24 رمزًا** لكل Skill إضافة إلى أطوال الحقول الفعلية.

## دورة حياة Skills المُدارة

يشحن OpenClaw مجموعة أساسية من Skills على أنها **Skills مجمّعة** كجزء من
التثبيت (حزمة npm أو OpenClaw.app). ويُستخدم `~/.openclaw/skills` من أجل
التجاوزات المحلية (مثل تثبيت إصدار معيّن أو ترقيع Skill من دون تغيير
النسخة المجمّعة). أما Skills الخاصة بمساحة العمل فهي مملوكة للمستخدم وتتجاوز كليهما عند تعارض الأسماء.

## مرجع التهيئة

راجع [تهيئة Skills](/tools/skills-config) للاطلاع على مخطط التهيئة الكامل.

## هل تبحث عن المزيد من Skills؟

تصفح [https://clawhub.ai](https://clawhub.ai).

---

## ذو صلة

- [إنشاء Skills](/tools/creating-skills) — بناء Skills مخصصة
- [تهيئة Skills](/tools/skills-config) — مرجع تهيئة Skills
- [أوامر الشرطة المائلة](/tools/slash-commands) — جميع أوامر الشرطة المائلة المتاحة
- [Plugins](/tools/plugin) — نظرة عامة على نظام plugin
