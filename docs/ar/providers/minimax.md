---
read_when:
    - تريد استخدام نماذج MiniMax في OpenClaw
    - تحتاج إلى إرشادات إعداد MiniMax
summary: استخدم نماذج MiniMax في OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-04-06T03:12:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ca35c43cdde53f6f09d9e12d48ce09e4c099cf8cbe1407ac6dbb45b1422507e
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

يستخدم موفر MiniMax في OpenClaw افتراضيًا **MiniMax M2.7**.

كما يوفر MiniMax أيضًا:

- توليف كلام مجمّع عبر T2A v2
- فهم صور مجمّع عبر `MiniMax-VL-01`
- توليد موسيقى مجمّع عبر `music-2.5+`
- `web_search` مجمّعًا عبر MiniMax Coding Plan search API

تقسيم الموفر:

- `minimax`: موفر نصوص قائم على API key، بالإضافة إلى توليد الصور، وفهم الصور، والكلام، والبحث في الويب المجمّعة
- `minimax-portal`: موفر نصوص عبر OAuth، بالإضافة إلى توليد الصور وفهم الصور المجمّعين

## مجموعة النماذج

- `MiniMax-M2.7`: نموذج الاستدلال المستضاف الافتراضي.
- `MiniMax-M2.7-highspeed`: مستوى استدلال M2.7 أسرع.
- `image-01`: نموذج توليد الصور (التوليد والتحرير من صورة إلى صورة).

## توليد الصور

تسجل إضافة MiniMax النموذج `image-01` لأداة `image_generate`. وهو يدعم:

- **توليد الصور من النص** مع التحكم في نسبة الأبعاد.
- **التحرير من صورة إلى صورة** (مرجع الموضوع) مع التحكم في نسبة الأبعاد.
- حتى **9 صور ناتجة** لكل طلب.
- حتى **صورة مرجعية واحدة** لكل طلب تحرير.
- نسب الأبعاد المدعومة: `1:1` و`16:9` و`4:3` و`3:2` و`2:3` و`3:4` و`9:16` و`21:9`.

لاستخدام MiniMax لتوليد الصور، اضبطه كموفر توليد الصور:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

تستخدم الإضافة نفس `MINIMAX_API_KEY` أو مصادقة OAuth المستخدمة لنماذج النص. لا حاجة إلى أي تكوين إضافي إذا كان MiniMax مُعدًا بالفعل.

يسجل كل من `minimax` و`minimax-portal` أداة `image_generate` باستخدام النموذج نفسه
`image-01`. تستخدم إعدادات API key القيمة `MINIMAX_API_KEY`؛ ويمكن لإعدادات OAuth استخدام
مسار المصادقة المجمّع `minimax-portal` بدلًا من ذلك.

عندما تكتب عملية onboarding أو إعداد API key إدخالات صريحة تحت `models.providers.minimax`،
يقوم OpenClaw بإظهار `MiniMax-M2.7` و
`MiniMax-M2.7-highspeed` مع `input: ["text", "image"]`.

أما كتالوج نصوص MiniMax المجمّع المدمج نفسه فيبقى بيانات وصفية خاصة بالنص فقط حتى
يظهر تكوين موفر صريح. ويتم كشف فهم الصور بشكل منفصل
عبر موفر الوسائط المملوك للإضافة `MiniMax-VL-01`.

راجع [Image Generation](/ar/tools/image-generation) للاطلاع على
معلمات الأداة المشتركة، واختيار الموفّر، وسلوك الفشل الاحتياطي.

## توليد الموسيقى

تسجل الإضافة المجمّعة `minimax` أيضًا توليد الموسيقى عبر الأداة المشتركة
`music_generate`.

- نموذج الموسيقى الافتراضي: `minimax/music-2.5+`
- يدعم أيضًا `minimax/music-2.5` و`minimax/music-2.0`
- عناصر التحكم في المطالبة: `lyrics` و`instrumental` و`durationSeconds`
- تنسيق الإخراج: `mp3`
- يتم فصل التشغيلات المدعومة بالجلسة عبر تدفق المهمة/الحالة المشترك، بما في ذلك `action: "status"`

لاستخدام MiniMax كموفر الموسيقى الافتراضي:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "minimax/music-2.5+",
      },
    },
  },
}
```

راجع [Music Generation](/tools/music-generation) للاطلاع على
معلمات الأداة المشتركة، واختيار الموفّر، وسلوك الفشل الاحتياطي.

## توليد الفيديو

تسجل الإضافة المجمّعة `minimax` أيضًا توليد الفيديو عبر الأداة المشتركة
`video_generate`.

- نموذج الفيديو الافتراضي: `minimax/MiniMax-Hailuo-2.3`
- الأوضاع: تحويل النص إلى فيديو وتدفقات مرجعية بصورة واحدة
- يدعم `aspectRatio` و`resolution`

لاستخدام MiniMax كموفر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "minimax/MiniMax-Hailuo-2.3",
      },
    },
  },
}
```

راجع [Video Generation](/tools/video-generation) للاطلاع على
معلمات الأداة المشتركة، واختيار الموفّر، وسلوك الفشل الاحتياطي.

## فهم الصور

تسجل إضافة MiniMax فهم الصور بشكل منفصل عن
كتالوج النصوص:

- `minimax`: نموذج الصور الافتراضي `MiniMax-VL-01`
- `minimax-portal`: نموذج الصور الافتراضي `MiniMax-VL-01`

لهذا السبب يمكن لتوجيه الوسائط التلقائي استخدام فهم الصور في MiniMax حتى
عندما يظل كتالوج موفر النصوص المجمّع يعرض مراجع محادثة M2.7 كبيانات وصفية نصية فقط.

## البحث في الويب

تسجل إضافة MiniMax أيضًا `web_search` عبر MiniMax Coding Plan
search API.

- معرّف الموفّر: `minimax`
- نتائج مهيكلة: عناوين، وعناوين URL، ومقتطفات، وعمليات بحث ذات صلة
- متغير البيئة المفضل: `MINIMAX_CODE_PLAN_KEY`
- الاسم المستعار المقبول في env: `MINIMAX_CODING_API_KEY`
- رجوع التوافق: `MINIMAX_API_KEY` عندما يكون يشير بالفعل إلى token لخطة البرمجة
- إعادة استخدام المنطقة: `plugins.entries.minimax.config.webSearch.region`، ثم `MINIMAX_API_HOST`، ثم عناوين base URLs لموفر MiniMax
- يبقى البحث على معرّف الموفّر `minimax`؛ ويمكن لإعدادات OAuth الصينية/العالمية مع ذلك توجيه المنطقة بشكل غير مباشر عبر `models.providers.minimax-portal.baseUrl`

يوجد التكوين تحت `plugins.entries.minimax.config.webSearch.*`.
راجع [MiniMax Search](/ar/tools/minimax-search).

## اختر إعدادًا

### MiniMax OAuth (Coding Plan) - موصى به

**الأفضل لـ:** إعداد سريع مع MiniMax Coding Plan عبر OAuth، من دون الحاجة إلى API key.

قم بالمصادقة باستخدام خيار OAuth الإقليمي الصريح:

```bash
openclaw onboard --auth-choice minimax-global-oauth
# أو
openclaw onboard --auth-choice minimax-cn-oauth
```

تعيين الخيارات:

- `minimax-global-oauth`: للمستخدمين الدوليين (`api.minimax.io`)
- `minimax-cn-oauth`: للمستخدمين في الصين (`api.minimaxi.com`)

راجع README الخاصة بحزمة إضافة MiniMax في مستودع OpenClaw للتفاصيل.

### MiniMax M2.7 ‏(API key)

**الأفضل لـ:** MiniMax المستضاف مع API متوافق مع Anthropic.

قم بالتكوين عبر CLI:

- onboarding تفاعلي:

```bash
openclaw onboard --auth-choice minimax-global-api
# أو
openclaw onboard --auth-choice minimax-cn-api
```

- `minimax-global-api`: للمستخدمين الدوليين (`api.minimax.io`)
- `minimax-cn-api`: للمستخدمين في الصين (`api.minimaxi.com`)

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
          {
            id: "MiniMax-M2.7-highspeed",
            name: "MiniMax M2.7 Highspeed",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

في مسار البث المتوافق مع Anthropic، يعطّل OpenClaw الآن تفكير MiniMax
افتراضيًا ما لم تضبط `thinking` بنفسك صراحة. تُصدر نقطة نهاية البث في MiniMax
`reasoning_content` ضمن مقاطع delta بأسلوب OpenAI
بدلًا من كتل التفكير الأصلية الخاصة بـ Anthropic، ما قد يؤدي إلى تسريب الاستدلال الداخلي
إلى الإخراج الظاهر إذا تُرك مفعّلًا ضمنيًا.

### MiniMax M2.7 كخيار احتياطي (مثال)

**الأفضل لـ:** الاحتفاظ بأقوى نموذج من الجيل الأحدث كنموذج أساسي، مع الرجوع إلى MiniMax M2.7 عند الفشل.
يستخدم المثال أدناه Opus كنموذج أساسي ملموس؛ استبدله بالنموذج الأساسي الأحدث الذي تفضله.

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "primary" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

## التكوين عبر `openclaw configure`

استخدم معالج التكوين التفاعلي لضبط MiniMax من دون تحرير JSON:

1. شغّل `openclaw configure`.
2. اختر **Model/auth**.
3. اختر أحد خيارات مصادقة **MiniMax**.
4. اختر نموذجك الافتراضي عندما يُطلب منك ذلك.

خيارات مصادقة MiniMax الحالية في المعالج/CLI:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## خيارات التكوين

- `models.providers.minimax.baseUrl`: يُفضَّل `https://api.minimax.io/anthropic` (متوافق مع Anthropic)؛ و`https://api.minimax.io/v1` اختياري لحمولات متوافقة مع OpenAI.
- `models.providers.minimax.api`: يُفضَّل `anthropic-messages`؛ و`openai-completions` اختياري لحمولات متوافقة مع OpenAI.
- `models.providers.minimax.apiKey`: API key الخاص بـ MiniMax (`MINIMAX_API_KEY`).
- `models.providers.minimax.models`: عرّف `id` و`name` و`reasoning` و`contextWindow` و`maxTokens` و`cost`.
- `agents.defaults.models`: أعطِ أسماء مستعارة للنماذج التي تريدها في allowlist.
- `models.mode`: أبقه على `merge` إذا كنت تريد إضافة MiniMax إلى جانب المدمج.

## ملاحظات

- تتبع مراجع النماذج مسار المصادقة:
  - إعداد API key: `minimax/<model>`
  - إعداد OAuth: `minimax-portal/<model>`
- نموذج الدردشة الافتراضي: `MiniMax-M2.7`
- نموذج الدردشة البديل: `MiniMax-M2.7-highspeed`
- مع `api: "anthropic-messages"`، يقوم OpenClaw بحقن
  `thinking: { type: "disabled" }` ما لم يكن التفكير مضبوطًا بالفعل صراحة في
  المعلمات/التكوين.
- يعيد `/fast on` أو `params.fastMode: true` كتابة `MiniMax-M2.7` إلى
  `MiniMax-M2.7-highspeed` على مسار البث المتوافق مع Anthropic.
- يكتب onboarding وإعداد API key المباشر تعريفات نماذج صريحة مع
  `input: ["text", "image"]` لكلا متغيري M2.7
- يكشف كتالوج الموفّر المجمّع حاليًا عن مراجع الدردشة كبيانات وصفية
  نصية فقط إلى أن يوجد تكوين موفر MiniMax صريح
- API استخدام Coding Plan: ‏`https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains` (يتطلب مفتاح Coding Plan).
- يطبّع OpenClaw استخدام MiniMax coding-plan إلى عرض `% المتبقي` نفسه
  المستخدم مع الموفّرين الآخرين. إن الحقول الخام `usage_percent` / `usagePercent`
  في MiniMax تمثل الحصة المتبقية، لا الحصة المستهلكة، لذا يعكسها OpenClaw.
  تحظى الحقول القائمة على العد بالأولوية عند وجودها. وعندما تعيد API القيمة `model_remains`,
  يفضّل OpenClaw إدخال نموذج الدردشة، ويشتق تسمية النافذة من
  `start_time` / `end_time` عند الحاجة، ويتضمن اسم النموذج المحدد
  في تسمية الخطة حتى تصبح نوافذ coding-plan أسهل في التمييز.
- تتعامل لقطات الاستخدام مع `minimax` و`minimax-cn` و`minimax-portal` باعتبارها
  سطح الحصة نفسه الخاص بـ MiniMax، وتفضّل MiniMax OAuth المخزّن قبل
  الرجوع إلى متغيرات env الخاصة بمفتاح Coding Plan.
- حدّث قيم التسعير في `models.json` إذا كنت تحتاج إلى تتبع تكلفة دقيق.
- رابط الإحالة لـ MiniMax Coding Plan (خصم 10%): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- راجع [/concepts/model-providers](/ar/concepts/model-providers) للاطلاع على قواعد الموفّرين.
- استخدم `openclaw models list` لتأكيد معرّف الموفّر الحالي، ثم بدّل باستخدام
  `openclaw models set minimax/MiniMax-M2.7` أو
  `openclaw models set minimax-portal/MiniMax-M2.7`.

## استكشاف الأخطاء وإصلاحها

### "Unknown model: minimax/MiniMax-M2.7"

يعني هذا عادة أن **موفّر MiniMax غير مكوّن** (لا يوجد
إدخال موفّر مطابق ولا ملف مصادقة/مفتاح env خاص بـ MiniMax). يوجد إصلاح لهذا
الاكتشاف في **2026.1.12**. أصلح المشكلة عبر:

- الترقية إلى **2026.1.12** (أو التشغيل من المصدر `main`) ثم إعادة تشغيل البوابة.
- تشغيل `openclaw configure` واختيار أحد خيارات مصادقة **MiniMax**، أو
- إضافة الكتلة المطابقة `models.providers.minimax` أو
  `models.providers.minimax-portal` يدويًا، أو
- ضبط `MINIMAX_API_KEY` أو `MINIMAX_OAUTH_TOKEN` أو ملف مصادقة MiniMax
  بحيث يمكن حقن الموفّر المطابق.

تأكد من أن معرّف النموذج **حساس لحالة الأحرف**:

- مسار API key: ‏`minimax/MiniMax-M2.7` أو `minimax/MiniMax-M2.7-highspeed`
- مسار OAuth: ‏`minimax-portal/MiniMax-M2.7` أو
  `minimax-portal/MiniMax-M2.7-highspeed`

ثم تحقق مجددًا باستخدام:

```bash
openclaw models list
```
