---
read_when:
    - إضافة تهيئة Skills أو تعديلها
    - ضبط allowlist المضمّنة أو سلوك التثبيت
summary: مخطط تهيئة Skills والأمثلة
title: تهيئة Skills
x-i18n:
    generated_at: "2026-04-05T12:59:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7839f39f68c1442dcf4740b09886e0ef55762ce0d4b9f7b4f493a8c130c84579
    source_path: tools/skills-config.md
    workflow: 15
---

# تهيئة Skills

توجد معظم إعدادات تحميل/تثبيت Skills ضمن `skills` في
`~/.openclaw/openclaw.json`. وتوجد إمكانية رؤية Skills الخاصة بكل وكيل ضمن
`agents.defaults.skills` و`agents.list[].skills`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills", "~/Projects/oss/some-skill-pack/skills"],
      watch: true,
      watchDebounceMs: 250,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun (ما زال Gateway runtime هو Node؛ ولا يُنصح باستخدام bun)
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // أو سلسلة نصية plaintext
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

بالنسبة إلى إنشاء/تحرير الصور المضمّن، استخدم `agents.defaults.imageGenerationModel`
مع أداة `image_generate` الأساسية. يُستخدم `skills.entries.*` فقط لتدفقات عمل Skills
المخصصة أو الخارجية.

إذا اخترت موفر/نموذج صور محددًا، فقم أيضًا بتهيئة
المصادقة/مفتاح API الخاص بذلك الموفّر. من الأمثلة الشائعة: `GEMINI_API_KEY` أو `GOOGLE_API_KEY` من أجل
`google/*`، و`OPENAI_API_KEY` من أجل `openai/*`، و`FAL_KEY` من أجل `fal/*`.

أمثلة:

- إعداد أصلي على نمط Nano Banana: `agents.defaults.imageGenerationModel.primary: "google/gemini-3.1-flash-image-preview"`
- إعداد fal أصلي: `agents.defaults.imageGenerationModel.primary: "fal/fal-ai/flux/dev"`

## قوائم سماح Skills الخاصة بالوكلاء

استخدم تهيئة الوكيل عندما تريد جذور Skills نفسها على مستوى الجهاز/مساحة العمل، لكن
مع مجموعة Skills مرئية مختلفة لكل وكيل.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // يرث القيم الافتراضية -> github, weather
      { id: "docs", skills: ["docs-search"] }, // يستبدل القيم الافتراضية
      { id: "locked-down", skills: [] }, // بلا Skills
    ],
  },
}
```

القواعد:

- `agents.defaults.skills`: allowlist أساسي مشترك للوكلاء الذين يهملون
  `agents.list[].skills`.
- احذف `agents.defaults.skills` لترك Skills غير مقيّدة افتراضيًا.
- `agents.list[].skills`: مجموعة Skills النهائية الصريحة لذلك الوكيل؛ وهي لا
  تُدمج مع القيم الافتراضية.
- `agents.list[].skills: []`: لا تعرض أي Skills لذلك الوكيل.

## الحقول

- تتضمن جذور Skills المضمّنة دائمًا `~/.openclaw/skills` و`~/.agents/skills`،
  و`<workspace>/.agents/skills`، و`<workspace>/skills`.
- `allowBundled`: allowlist اختيارية لـ Skills **المضمّنة** فقط. عند تعيينها، تكون
  Skills المضمّنة الموجودة في القائمة فقط مؤهلة (لا تتأثر Skills المُدارة، وSkills الوكيل، وSkills مساحة العمل).
- `load.extraDirs`: أدلة Skills إضافية للفحص (الأولوية الأدنى).
- `load.watch`: مراقبة مجلدات Skills وتحديث لقطة Skills (الافتراضي: true).
- `load.watchDebounceMs`: مهلة debounce لأحداث مراقب Skills بالمللي ثانية (الافتراضي: 250).
- `install.preferBrew`: تفضيل أدوات تثبيت brew عند توفرها (الافتراضي: true).
- `install.nodeManager`: تفضيل أداة تثبيت Node (`npm` | `pnpm` | `yarn` | `bun`، والافتراضي: npm).
  يؤثر هذا فقط في **تثبيتات Skills**؛ ويجب أن يظل Gateway runtime هو Node
  (ولا يُنصح باستخدام Bun مع WhatsApp/Telegram).
  - الخيار `openclaw setup --node-manager` أضيق نطاقًا ويقبل حاليًا `npm`،
    أو `pnpm`، أو `bun`. اضبط `skills.install.nodeManager: "yarn"` يدويًا إذا كنت
    تريد تثبيتات Skills مدعومة بـ Yarn.
- `entries.<skillKey>`: تجاوزات لكل Skill.
- `agents.defaults.skills`: allowlist افتراضية اختيارية لـ Skills يرثها الوكلاء
  الذين يهملون `agents.list[].skills`.
- `agents.list[].skills`: allowlist نهائية اختيارية لكل وكيل؛ حيث تستبدل
  القوائم الصريحة القيم الافتراضية الموروثة بدلًا من دمجها.

الحقول الخاصة بكل Skill:

- `enabled`: اضبطها على `false` لتعطيل Skill حتى لو كانت مضمّنة/مثبتة.
- `env`: متغيرات بيئة تُحقن لتشغيل الوكيل (فقط إذا لم تكن معيّنة بالفعل).
- `apiKey`: وسيلة راحة اختيارية لـ Skills التي تعلن عن متغير env أساسي.
  يدعم سلسلة plaintext أو كائن SecretRef (`{ source, provider, id }`).

## ملاحظات

- تُطابِق المفاتيح ضمن `entries` اسم Skill افتراضيًا. إذا عرّفت Skill
  `metadata.openclaw.skillKey`، فاستخدم ذلك المفتاح بدلًا منه.
- أولوية التحميل هي `<workspace>/skills` → `<workspace>/.agents/skills` →
  `~/.agents/skills` → `~/.openclaw/skills` → Skills المضمّنة →
  `skills.load.extraDirs`.
- تُلتقط التغييرات على Skills في دور الوكيل التالي عندما تكون المراقبة مفعّلة.

### Skills المعزولة + متغيرات البيئة

عندما تكون الجلسة **معزولة**، تعمل عمليات Skills داخل Docker. ولا ترث
البيئة المعزولة `process.env` الخاصة بالمضيف.

استخدم أحد الخيارين التاليين:

- `agents.defaults.sandbox.docker.env` (أو `agents.list[].sandbox.docker.env` لكل وكيل)
- تضمين env داخل صورة البيئة المعزولة المخصصة الخاصة بك

ينطبق `env` العام و`skills.entries.<skill>.env/apiKey` على عمليات **المضيف** فقط.
