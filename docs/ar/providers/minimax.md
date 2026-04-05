---
read_when:
    - تريد نماذج MiniMax في OpenClaw
    - تحتاج إلى إرشادات إعداد MiniMax
summary: استخدام نماذج MiniMax في OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-04-05T12:53:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 353e1d9ce1b48c90ccaba6cc0109e839c473ca3e65d0c5d8ba744e9011c2bf45
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

يستخدم مزوّد MiniMax في OpenClaw افتراضيًا **MiniMax M2.7**.

وتوفّر MiniMax أيضًا:

- speech synthesis مضمّنة عبر T2A v2
- فهم صور مضمّن عبر `MiniMax-VL-01`
- `web_search` مضمّنة عبر MiniMax Coding Plan search API

تقسيم المزوّد:

- `minimax`: مزوّد نصي بمفتاح API، بالإضافة إلى image generation وفهم الصور وspeech وweb search المضمّنة
- `minimax-portal`: مزوّد نصي عبر OAuth، بالإضافة إلى image generation وفهم الصور المضمّنة

## تشكيلة النماذج

- `MiniMax-M2.7`: نموذج الاستدلال المستضاف الافتراضي.
- `MiniMax-M2.7-highspeed`: فئة استدلال M2.7 أسرع.
- `image-01`: نموذج image generation ‏(الإنشاء والتحرير من صورة إلى صورة).

## Image generation

تسجّل plugin الخاصة بـ MiniMax النموذج `image-01` لأداة `image_generate`. وهو يدعم:

- **إنشاء الصور من النص** مع التحكم في نسبة الأبعاد.
- **التحرير من صورة إلى صورة** (مرجع الموضوع) مع التحكم في نسبة الأبعاد.
- حتى **9 صور مخرجة** لكل طلب.
- حتى **صورة مرجعية واحدة** لكل طلب تحرير.
- نسب الأبعاد المدعومة: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`.

لاستخدام MiniMax في image generation، اضبطها كمزوّد image generation:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

تستخدم plugin نفس مصادقة `MINIMAX_API_KEY` أو OAuth الخاصة بالنماذج النصية. ولا تحتاج إلى أي تكوين إضافي إذا كانت MiniMax معدّة بالفعل.

يسجّل كل من `minimax` و`minimax-portal` الأداة `image_generate` باستخدام
النموذج نفسه `image-01`. تستخدم إعدادات مفتاح API القيمة `MINIMAX_API_KEY`؛ ويمكن لإعدادات OAuth استخدام
مسار المصادقة المضمّن `minimax-portal` بدلًا من ذلك.

عندما تكتب onboarding أو إعداد مفتاح API إدخالات صريحة لـ `models.providers.minimax`,
فإن OpenClaw يُجسّد النموذجين `MiniMax-M2.7` و
`MiniMax-M2.7-highspeed` مع `input: ["text", "image"]`.

أما فهرس النصوص المضمّن والمبني في MiniMax نفسه فيبقى بيانات تعريف نصية فقط إلى أن يوجد تكوين مزوّد صريح. ويُكشف فهم الصور بشكل منفصل عبر مزوّد الوسائط `MiniMax-VL-01` المملوك للـ plugin.

## Image understanding

تسجّل plugin الخاصة بـ MiniMax فهم الصور بشكل منفصل عن
فهرس النصوص:

- `minimax`: نموذج الصور الافتراضي `MiniMax-VL-01`
- `minimax-portal`: نموذج الصور الافتراضي `MiniMax-VL-01`

ولهذا السبب يمكن أن يستخدم التوجيه التلقائي للوسائط فهم صور MiniMax حتى
عندما يظل فهرس المزوّد النصي المضمّن يعرض مراجع دردشة M2.7 النصية فقط.

## Web search

تسجّل plugin الخاصة بـ MiniMax أيضًا `web_search` عبر MiniMax Coding Plan
search API.

- معرّف المزوّد: `minimax`
- نتائج مهيكلة: عناوين، وعناوين URL، ومقتطفات، واستعلامات ذات صلة
- متغير env المفضل: `MINIMAX_CODE_PLAN_KEY`
- الاسم المستعار المقبول في env: `MINIMAX_CODING_API_KEY`
- الرجوع الاحتياطي التوافقي: `MINIMAX_API_KEY` عندما يشير بالفعل إلى رمز coding-plan
- إعادة استخدام المنطقة: `plugins.entries.minimax.config.webSearch.region`، ثم `MINIMAX_API_HOST`، ثم base URLs الخاصة بمزوّد MiniMax
- يبقى البحث على معرّف المزوّد `minimax`؛ ويمكن لإعداد OAuth CN/global أن يوجّه المنطقة بشكل غير مباشر عبر `models.providers.minimax-portal.baseUrl`

يعيش التكوين تحت `plugins.entries.minimax.config.webSearch.*`.
راجع [MiniMax Search](/tools/minimax-search).

## اختر إعدادًا

### MiniMax OAuth ‏(Coding Plan) - موصى به

**الأفضل لـ:** إعداد سريع مع MiniMax Coding Plan عبر OAuth، من دون الحاجة إلى مفتاح API.

نفّذ المصادقة باستخدام اختيار OAuth الإقليمي الصريح:

```bash
openclaw onboard --auth-choice minimax-global-oauth
# or
openclaw onboard --auth-choice minimax-cn-oauth
```

ربط الاختيارات:

- `minimax-global-oauth`: للمستخدمين الدوليين (`api.minimax.io`)
- `minimax-cn-oauth`: للمستخدمين في الصين (`api.minimaxi.com`)

راجع README الخاصة بحزمة MiniMax plugin في مستودع OpenClaw للتفاصيل.

### MiniMax M2.7 ‏(مفتاح API)

**الأفضل لـ:** MiniMax مستضافة مع API متوافقة مع Anthropic.

اضبطها عبر CLI:

- التهيئة الأولية التفاعلية:

```bash
openclaw onboard --auth-choice minimax-global-api
# or
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

في مسار البث المتوافق مع Anthropic، يعطّل OpenClaw الآن
thinking في MiniMax افتراضيًا ما لم تقم بتعيين `thinking` بنفسك صراحةً. ذلك أن
نقطة النهاية الخاصة بالبث في MiniMax تُصدر `reasoning_content` ضمن أجزاء delta
بأسلوب OpenAI بدلًا من كتل thinking الأصلية في Anthropic، ما قد يؤدي إلى تسرب الاستدلال الداخلي
إلى المخرجات المرئية إذا تُرك مفعّلًا ضمنيًا.

### MiniMax M2.7 كرجوع احتياطي (مثال)

**الأفضل لـ:** الإبقاء على أقوى نموذج حديث لديك كنموذج أساسي، مع الرجوع إلى MiniMax M2.7 عند الفشل.
يستخدم المثال أدناه Opus كنموذج أساسي ملموس؛ استبدله بالنموذج الحديث الأساسي الذي تفضله.

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

## الضبط عبر `openclaw configure`

استخدم معالج التكوين التفاعلي لضبط MiniMax من دون تحرير JSON:

1. شغّل `openclaw configure`.
2. اختر **Model/auth**.
3. اختر خيار مصادقة **MiniMax**.
4. اختر نموذجك الافتراضي عند المطالبة.

اختيارات مصادقة MiniMax الحالية في المعالج/CLI:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## خيارات التكوين

- `models.providers.minimax.baseUrl`: فضّل `https://api.minimax.io/anthropic` ‏(متوافق مع Anthropic)؛ ويظل `https://api.minimax.io/v1` اختياريًا للحمولات المتوافقة مع OpenAI.
- `models.providers.minimax.api`: فضّل `anthropic-messages`؛ وتظل `openai-completions` اختيارية للحمولات المتوافقة مع OpenAI.
- `models.providers.minimax.apiKey`: مفتاح MiniMax API ‏(`MINIMAX_API_KEY`).
- `models.providers.minimax.models`: عرّف `id` و`name` و`reasoning` و`contextWindow` و`maxTokens` و`cost`.
- `agents.defaults.models`: ضع أسماء مستعارة للنماذج التي تريدها في allowlist.
- `models.mode`: أبقِها `merge` إذا كنت تريد إضافة MiniMax إلى جانب المزوّدات المضمّنة.

## ملاحظات

- تتبع مراجع النموذج مسار المصادقة:
  - إعداد مفتاح API: ‏`minimax/<model>`
  - إعداد OAuth: ‏`minimax-portal/<model>`
- نموذج الدردشة الافتراضي: `MiniMax-M2.7`
- نموذج الدردشة البديل: `MiniMax-M2.7-highspeed`
- في `api: "anthropic-messages"`، يحقن OpenClaw
  `thinking: { type: "disabled" }` ما لم تكن thinking مضبوطة بالفعل صراحةً في
  params/config.
- تعيد `/fast on` أو `params.fastMode: true` كتابة `MiniMax-M2.7` إلى
  `MiniMax-M2.7-highspeed` على مسار البث المتوافق مع Anthropic.
- تكتب onboarding وإعداد مفتاح API المباشر تعريفات نماذج صريحة مع
  `input: ["text", "image"]` لكلا متغيرَي M2.7
- يكشف فهرس المزوّد المضمّن حاليًا مراجع الدردشة كبيانات تعريف
  نصية فقط إلى أن يوجد تكوين مزوّد MiniMax صريح
- واجهة Coding Plan usage API: ‏`https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains` ‏(تتطلب مفتاح coding plan).
- يطبّع OpenClaw استخدام MiniMax coding-plan إلى العرض نفسه `% left`
  المستخدم مع المزوّدات الأخرى. وتمثل الحقول الخام في MiniMax ‏`usage_percent` / `usagePercent`
  الحصة المتبقية، لا الحصة المستهلكة، لذا يعكسها OpenClaw.
  وتعطي الحقول المعتمدة على العد الأولوية عند وجودها. وعندما تعيد API القيمة `model_remains`,
  يفضّل OpenClaw إدخال chat-model، ويشتق تسمية النافذة من
  `start_time` / `end_time` عند الحاجة، ويضمّن اسم النموذج المختار
  في تسمية الخطة بحيث يصبح من الأسهل التمييز بين نوافذ coding-plan.
- تعامل لقطات الاستخدام `minimax` و`minimax-cn` و`minimax-portal` على أنها
  سطح الحصة نفسه في MiniMax، وتفضّل MiniMax OAuth المخزنة قبل الرجوع
  إلى متغيرات env الخاصة بمفاتيح Coding Plan.
- حدّث قيم التسعير في `models.json` إذا كنت تحتاج إلى تتبع دقيق للتكلفة.
- رابط الإحالة إلى MiniMax Coding Plan ‏(خصم 10%): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- راجع [/concepts/model-providers](/concepts/model-providers) لمعرفة قواعد المزوّد.
- استخدم `openclaw models list` لتأكيد معرّف المزوّد الحالي، ثم بدّل باستخدام
  `openclaw models set minimax/MiniMax-M2.7` أو
  `openclaw models set minimax-portal/MiniMax-M2.7`.

## استكشاف الأخطاء وإصلاحها

### "Unknown model: minimax/MiniMax-M2.7"

يعني هذا عادةً أن **مزوّد MiniMax غير مكوَّن** (لا يوجد
إدخال مزوّد مطابق ولا ملف تعريف/مفتاح env لمصادقة MiniMax). وقد تم تضمين إصلاح لهذا
الاكتشاف في **2026.1.12**. ويمكن الإصلاح عبر:

- الترقية إلى **2026.1.12** ‏(أو التشغيل من المصدر `main`)، ثم إعادة تشغيل gateway.
- تشغيل `openclaw configure` واختيار خيار مصادقة **MiniMax**، أو
- إضافة كتلة `models.providers.minimax` أو
  `models.providers.minimax-portal` المطابقة يدويًا، أو
- ضبط `MINIMAX_API_KEY` أو `MINIMAX_OAUTH_TOKEN` أو ملف تعريف مصادقة MiniMax
  حتى يمكن حقن المزوّد المطابق.

تأكد من أن معرّف النموذج **حساس لحالة الأحرف**:

- مسار مفتاح API: ‏`minimax/MiniMax-M2.7` أو `minimax/MiniMax-M2.7-highspeed`
- مسار OAuth: ‏`minimax-portal/MiniMax-M2.7` أو
  `minimax-portal/MiniMax-M2.7-highspeed`

ثم أعد التحقق باستخدام:

```bash
openclaw models list
```
