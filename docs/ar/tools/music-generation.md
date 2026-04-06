---
read_when:
    - إنشاء موسيقى أو صوت عبر الوكيل
    - إعداد مزودي ونماذج توليد الموسيقى
    - فهم معلمات أداة music_generate
summary: أنشئ موسيقى باستخدام مزودين مشتركين، بما في ذلك الإضافات المعتمدة على workflows
title: توليد الموسيقى
x-i18n:
    generated_at: "2026-04-06T03:13:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: a03de8aa75cfb7248eb0c1d969fb2a6da06117967d097e6f6e95771d0f017ae1
    source_path: tools/music-generation.md
    workflow: 15
---

# توليد الموسيقى

تتيح أداة `music_generate` للوكيل إنشاء موسيقى أو صوت عبر
إمكانات توليد الموسيقى المشتركة باستخدام مزودين مُعدّين مثل Google،
وMiniMax، وComfyUI المُعد عبر workflows.

بالنسبة إلى جلسات الوكيل المعتمدة على مزودين مشتركين، يبدأ OpenClaw توليد الموسيقى
كمهمة في الخلفية، ويتتبعها في سجل المهام، ثم يوقظ الوكيل مرة أخرى عندما
يصبح المقطع جاهزًا حتى يتمكن الوكيل من نشر الصوت النهائي مرة أخرى في
القناة الأصلية.

<Note>
لا تظهر الأداة المشتركة المدمجة إلا عند توفر مزود واحد على الأقل لتوليد الموسيقى. إذا لم ترَ `music_generate` ضمن أدوات وكيلك، فقم بإعداد `agents.defaults.musicGenerationModel` أو اضبط مفتاح API لمزود.
</Note>

## بدء سريع

### التوليد المعتمد على مزودين مشتركين

1. اضبط مفتاح API لمزود واحد على الأقل، مثل `GEMINI_API_KEY` أو
   `MINIMAX_API_KEY`.
2. اختياريًا، اضبط النموذج المفضل لديك:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

3. اطلب من الوكيل: _"أنشئ مقطع synthpop حيويًا عن قيادة ليلية
   عبر مدينة نيون."_

سيستدعي الوكيل `music_generate` تلقائيًا. لا حاجة إلى قائمة سماح للأدوات.

في السياقات التزامنية المباشرة من دون تشغيل وكيل مدعوم بجلسة، تظل
الأداة المدمجة تعود إلى التوليد المضمن وتعيد مسار الوسائط النهائي في
نتيجة الأداة.

أمثلة على المطالبات:

```text
Generate a cinematic piano track with soft strings and no vocals.
```

```text
Generate an energetic chiptune loop about launching a rocket at sunrise.
```

### توليد Comfy المعتمد على workflow

تتصل إضافة `comfy` المضمنة بأداة `music_generate` المشتركة عبر
سجل مزودي توليد الموسيقى.

1. اضبط `models.providers.comfy.music` باستخدام workflow JSON و
   عقد prompt/output.
2. إذا كنت تستخدم Comfy Cloud، فاضبط `COMFY_API_KEY` أو `COMFY_CLOUD_API_KEY`.
3. اطلب من الوكيل إنشاء موسيقى أو استدعِ الأداة مباشرة.

مثال:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

## دعم المزودين المضمنين المشتركين

| المزود | النموذج الافتراضي | المدخلات المرجعية | عناصر التحكم المدعومة | مفتاح API |
| -------- | ---------------------- | ---------------- | --------------------------------------------------------- | -------------------------------------- |
| ComfyUI  | `workflow`             | حتى صورة واحدة    | موسيقى أو صوت معرّفان بواسطة workflow                           | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google   | `lyria-3-clip-preview` | حتى 10 صور        | `lyrics`, `instrumental`, `format`                        | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax  | `music-2.5+`           | لا شيء             | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY`                      |

استخدم `action: "list"` لفحص المزودين والنماذج المشتركة المتاحة في
وقت التشغيل:

```text
/tool music_generate action=list
```

استخدم `action: "status"` لفحص مهمة الموسيقى النشطة المدعومة بالجلسة:

```text
/tool music_generate action=status
```

مثال على التوليد المباشر:

```text
/tool music_generate prompt="Dreamy lo-fi hip hop with vinyl texture and gentle rain" instrumental=true
```

## معلمات الأداة المدمجة

| المعلمة | النوع | الوصف |
| ----------------- | -------- | ------------------------------------------------------------------------------------------------- |
| `prompt`          | string   | مطالبة توليد الموسيقى (مطلوبة عند `action: "generate"`) |
| `action`          | string   | `"generate"` (الافتراضي)، أو `"status"` لمهمة الجلسة الحالية، أو `"list"` لفحص المزودين |
| `model`           | string   | تجاوز المزود/النموذج، مثل `google/lyria-3-pro-preview` أو `comfy/workflow` |
| `lyrics`          | string   | كلمات أغنية اختيارية عندما يدعم المزود إدخال كلمات صريحًا |
| `instrumental`    | boolean  | طلب إخراج آلي فقط عندما يدعم المزود ذلك |
| `image`           | string   | مسار صورة مرجعية واحدة أو URL |
| `images`          | string[] | صور مرجعية متعددة (حتى 10) |
| `durationSeconds` | number   | المدة المستهدفة بالثواني عندما يدعم المزود تلميحات المدة |
| `format`          | string   | تلميح تنسيق الإخراج (`mp3` أو `wav`) عندما يدعم المزود ذلك |
| `filename`        | string   | تلميح اسم ملف الإخراج |

ليست كل المعلمات مدعومة من جميع المزودين. لا يزال OpenClaw يتحقق من الحدود
القصوى الصارمة مثل عدد المدخلات قبل الإرسال، لكن التلميحات الاختيارية غير المدعومة
يتم تجاهلها مع تحذير عندما لا يستطيع المزود أو النموذج المحدد احترامها.

## السلوك غير المتزامن للمسار المعتمد على المزودين المشتركين

- تشغيلات الوكيل المدعومة بالجلسة: تنشئ `music_generate` مهمة في الخلفية، وتعيد استجابة بدء/مهمة فورًا، ثم تنشر المقطع النهائي لاحقًا في رسالة متابعة من الوكيل.
- منع التكرار: طالما أن مهمة الخلفية لا تزال في حالة `queued` أو `running`، فإن استدعاءات `music_generate` اللاحقة في الجلسة نفسها تعيد حالة المهمة بدلًا من بدء توليد آخر.
- البحث عن الحالة: استخدم `action: "status"` لفحص مهمة الموسيقى النشطة المدعومة بالجلسة من دون بدء مهمة جديدة.
- تتبع المهام: استخدم `openclaw tasks list` أو `openclaw tasks show <taskId>` لفحص الحالات queued وrunning والحالات النهائية للتوليد.
- إيقاظ الإكمال: يحقن OpenClaw حدث إكمال داخليًا مرة أخرى في الجلسة نفسها حتى يتمكن النموذج من كتابة رسالة المتابعة الموجهة للمستخدم بنفسه.
- تلميح الموجّه: تحصل الأدوار اللاحقة للمستخدم/اليدوية في الجلسة نفسها على تلميح صغير في وقت التشغيل عندما تكون مهمة موسيقى قيد التنفيذ بالفعل حتى لا يستدعي النموذج `music_generate` مرة أخرى بشكل أعمى.
- بديل عدم وجود جلسة: ما تزال السياقات المباشرة/المحلية من دون جلسة وكيل فعلية تعمل بشكل مضمن وتعيد نتيجة الصوت النهائية في الدور نفسه.

## الإعدادات

### اختيار النموذج

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["minimax/music-2.5+"],
      },
    },
  },
}
```

### ترتيب اختيار المزود

عند توليد الموسيقى، يحاول OpenClaw المزودين بهذا الترتيب:

1. معلمة `model` من استدعاء الأداة، إذا حدد الوكيل واحدة
2. `musicGenerationModel.primary` من الإعدادات
3. `musicGenerationModel.fallbacks` بالترتيب
4. الاكتشاف التلقائي باستخدام القيم الافتراضية للمزودين المدعومة بالمصادقة فقط:
   - المزود الافتراضي الحالي أولًا
   - مزودو توليد الموسيقى المسجلون الباقون بترتيب معرّف المزود

إذا فشل أحد المزودين، تُجرَّب المحاولة التالية تلقائيًا. وإذا فشل الجميع، فإن
الخطأ يتضمن تفاصيل من كل محاولة.

## ملاحظات حول المزودين

- يستخدم Google توليد Lyria 3 الدفعي. يدعم التدفق المضمن الحالي
  prompt، ونص كلمات اختياري، وصورًا مرجعية اختيارية.
- يستخدم MiniMax نقطة النهاية الدفعية `music_generation`. يدعم التدفق المضمن الحالي
  prompt، وكلمات اختيارية، ووضع الآلات فقط، وتوجيه المدة، و
  إخراج mp3.
- دعم ComfyUI معتمد على workflow ويعتمد على الرسم البياني المكوَّن بالإضافة إلى
  ربط العقد لحقول prompt/output.

## اختيار المسار المناسب

- استخدم المسار المعتمد على المزودين المشتركين عندما تريد اختيار النموذج، وتجاوز فشل المزود، والتدفق المدمج للمهام/الحالة غير المتزامنة.
- استخدم مسار إضافة مثل ComfyUI عندما تحتاج إلى رسم workflow مخصص أو مزود غير موجود ضمن إمكانات الموسيقى المضمنة المشتركة.
- إذا كنت تستكشف سلوكًا خاصًا بـ ComfyUI، فراجع [ComfyUI](/providers/comfy). وإذا كنت تستكشف سلوك المزود المشترك، فابدأ بـ [Google (Gemini)](/ar/providers/google) أو [MiniMax](/ar/providers/minimax).

## الاختبارات الحية

تغطية حية اختيارية للمزودين المشتركين المضمنين:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

تغطية حية اختيارية لمسار الموسيقى ComfyUI المضمن:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

يغطي ملف Comfy الحي أيضًا workflows الخاصة بالصور والفيديو عند إعداد
تلك الأقسام.

## ذو صلة

- [المهام في الخلفية](/ar/automation/tasks) - تتبع المهام لتشغيلات `music_generate` المنفصلة
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults) - إعدادات `musicGenerationModel`
- [ComfyUI](/providers/comfy)
- [Google (Gemini)](/ar/providers/google)
- [MiniMax](/ar/providers/minimax)
- [النماذج](/ar/concepts/models) - إعداد النموذج وتجاوز الفشل
- [نظرة عامة على الأدوات](/ar/tools)
