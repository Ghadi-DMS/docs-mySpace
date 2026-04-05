---
read_when:
    - تصميم أو إعادة هيكلة فهم الوسائط الواردة
    - ضبط المعالجة المسبقة للصوت/الفيديو/الصور الواردة
summary: فهم الصور/الصوت/الفيديو الواردة (اختياري) مع مسارات احتياطية عبر الموفّرين وCLI
title: فهم الوسائط
x-i18n:
    generated_at: "2026-04-05T12:49:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: fe36bd42250d48d12f4ff549e8644afa7be8e42ee51f8aff4f21f81b7ff060f4
    source_path: nodes/media-understanding.md
    workflow: 15
---

# فهم الوسائط - الوارد (2026-01-17)

يمكن لـ OpenClaw **تلخيص الوسائط الواردة** (الصور/الصوت/الفيديو) قبل تشغيل مسار الرد. وهو يكتشف تلقائيًا متى تكون الأدوات المحلية أو مفاتيح الموفّرين متاحة، ويمكن تعطيله أو تخصيصه. وإذا كان الفهم معطلًا، فلا تزال النماذج تتلقى الملفات/عناوين URL الأصلية كالمعتاد.

يتم تسجيل سلوك الوسائط الخاص بكل جهة مصنّعة بواسطة إضافات الجهات المصنّعة، بينما
تملك core في OpenClaw الإعداد المشترك `tools.media`، وترتيب المسارات الاحتياطية،
والتكامل مع مسار الرد.

## الأهداف

- اختياري: هضم أولي للوسائط الواردة إلى نص قصير لتوجيه أسرع + تحليل أفضل للأوامر.
- الحفاظ دائمًا على تسليم الوسائط الأصلية إلى النموذج.
- دعم **واجهات API الخاصة بالموفّرين** و**المسارات الاحتياطية عبر CLI**.
- السماح بعدة نماذج مع تراجع احتياطي مرتب (خطأ/حجم/مهلة).

## السلوك عالي المستوى

1. جمع المرفقات الواردة (`MediaPaths` و`MediaUrls` و`MediaTypes`).
2. لكل قدرة مفعلة (صورة/صوت/فيديو)، تحديد المرفقات وفق السياسة (الافتراضي: **الأول**).
3. اختيار أول إدخال نموذج مؤهل (الحجم + القدرة + المصادقة).
4. إذا فشل نموذج أو كانت الوسائط كبيرة جدًا، **يتم الرجوع إلى الإدخال التالي**.
5. عند النجاح:
   - تصبح `Body` كتلة `[Image]` أو `[Audio]` أو `[Video]`.
   - يضبط الصوت `{{Transcript}}`؛ ويستخدم تحليل الأوامر نص التسمية التوضيحية عند وجوده،
     وإلا يستخدم النص المفرغ.
   - تُحفَظ التسميات التوضيحية بوصفها `User text:` داخل الكتلة.

إذا فشل الفهم أو كان معطلًا، **يستمر تدفق الرد** باستخدام النص + المرفقات الأصلية.

## نظرة عامة على الإعداد

يدعم `tools.media` **نماذج مشتركة** بالإضافة إلى تجاوزات لكل قدرة:

- `tools.media.models`: قائمة النماذج المشتركة (استخدم `capabilities` للبوابة).
- `tools.media.image` / `tools.media.audio` / `tools.media.video`:
  - القيم الافتراضية (`prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`)
  - تجاوزات الموفّر (`baseUrl` و`headers` و`providerOptions`)
  - خيارات Deepgram للصوت عبر `tools.media.audio.providerOptions.deepgram`
  - عناصر تحكم صدى النص المفرغ للصوت (`echoTranscript`، الافتراضي `false`؛ و`echoFormat`)
  - قائمة `models` اختيارية **لكل قدرة** (وتُفضَّل قبل النماذج المشتركة)
  - سياسة `attachments` ‏(`mode` و`maxAttachments` و`prefer`)
  - `scope` ‏(بوابة اختيارية حسب القناة/نوع الدردشة/مفتاح الجلسة)
- `tools.media.concurrency`: الحد الأقصى لتشغيلات القدرات المتزامنة (الافتراضي **2**).

```json5
{
  tools: {
    media: {
      models: [
        /* قائمة مشتركة */
      ],
      image: {
        /* تجاوزات اختيارية */
      },
      audio: {
        /* تجاوزات اختيارية */
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
      video: {
        /* تجاوزات اختيارية */
      },
    },
  },
}
```

### إدخالات النماذج

يمكن أن يكون كل إدخال `models[]` من نوع **provider** أو **CLI**:

```json5
{
  type: "provider", // الافتراضي إذا تم حذفه
  provider: "openai",
  model: "gpt-5.4-mini",
  prompt: "Describe the image in <= 500 chars.",
  maxChars: 500,
  maxBytes: 10485760,
  timeoutSeconds: 60,
  capabilities: ["image"], // اختياري، ويُستخدم للإدخالات متعددة الوسائط
  profile: "vision-profile",
  preferredProfile: "vision-fallback",
}
```

```json5
{
  type: "cli",
  command: "gemini",
  args: [
    "-m",
    "gemini-3-flash",
    "--allowed-tools",
    "read_file",
    "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
  ],
  maxChars: 500,
  maxBytes: 52428800,
  timeoutSeconds: 120,
  capabilities: ["video", "image"],
}
```

يمكن لقوالب CLI أيضًا استخدام:

- `{{MediaDir}}` ‏(الدليل الذي يحتوي على ملف الوسائط)
- `{{OutputDir}}` ‏(دليل مؤقت أُنشئ لهذا التشغيل)
- `{{OutputBase}}` ‏(المسار الأساسي للملف المؤقت، من دون امتداد)

## القيم الافتراضية والحدود

القيم الافتراضية الموصى بها:

- `maxChars`: ‏**500** للصورة/الفيديو (قصير وملائم للأوامر)
- `maxChars`: ‏**غير مضبوطة** للصوت (النص المفرغ الكامل ما لم تضبط حدًا)
- `maxBytes`:
  - الصورة: **10MB**
  - الصوت: **20MB**
  - الفيديو: **50MB**

القواعد:

- إذا تجاوزت الوسائط `maxBytes`، يتم تخطي ذلك النموذج وتجربة **النموذج التالي**.
- تُعامَل الملفات الصوتية الأصغر من **1024 بايت** على أنها فارغة/تالفة ويتم تخطيها قبل النسخ عبر الموفّر/CLI.
- إذا أعاد النموذج أكثر من `maxChars`، يتم اقتطاع الإخراج.
- تفترض `prompt` افتراضيًا عبارة بسيطة من نوع “Describe the {media}.” بالإضافة إلى توجيه `maxChars` ‏(للصورة/الفيديو فقط).
- إذا كان نموذج الصورة الأساسي النشط يدعم الرؤية أصلًا، فإن OpenClaw
  يتخطى كتلة الملخص `[Image]` ويمرر الصورة الأصلية مباشرة إلى
  النموذج.
- إذا كانت `<capability>.enabled: true` لكن لا توجد نماذج مهيأة، يحاول OpenClaw
  **نموذج الرد النشط** عندما يدعم موفّره تلك القدرة.

### الاكتشاف التلقائي لفهم الوسائط (الافتراضي)

إذا لم يتم ضبط `tools.media.<capability>.enabled` على **`false`** ولم تكن قد
هيأت نماذج، فإن OpenClaw يكتشف تلقائيًا بهذا الترتيب ويتوقف **عند أول
خيار يعمل**:

1. **نموذج الرد النشط** عندما يدعم موفّره تلك القدرة.
2. مراجع `agents.defaults.imageModel` ‏primary/fallback ‏(للصورة فقط).
3. **CLI محلية** ‏(للصوت فقط؛ إذا كانت مثبتة)
   - `sherpa-onnx-offline` ‏(يتطلب `SHERPA_ONNX_MODEL_DIR` مع encoder/decoder/joiner/tokens)
   - `whisper-cli` ‏(`whisper-cpp`; يستخدم `WHISPER_CPP_MODEL` أو النموذج tiny المضمّن)
   - `whisper` ‏(Python CLI؛ ينزّل النماذج تلقائيًا)
4. **Gemini CLI** ‏(`gemini`) باستخدام `read_many_files`
5. **مصادقة الموفّر**
   - تتم تجربة إدخالات `models.providers.*` المهيأة التي تدعم القدرة
     قبل ترتيب التراجع الاحتياطي المضمّن.
   - يتم تسجيل موفّري الإعداد ذوي الصور فقط مع نموذج قادر على الصور تلقائيًا
     لفهم الوسائط حتى عندما لا يكونون إضافة vendor مدمجة.
   - ترتيب التراجع الاحتياطي المضمّن:
     - الصوت: OpenAI → Groq → Deepgram → Google → Mistral
     - الصورة: OpenAI → Anthropic → Google → MiniMax → MiniMax Portal → Z.AI
     - الفيديو: Google → Qwen → Moonshot

لتعطيل الاكتشاف التلقائي، اضبط:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

ملاحظة: يكون اكتشاف الملفات التنفيذية بأفضل جهد عبر macOS/Linux/Windows؛ تأكد من أن CLI موجودة على `PATH` ‏(نحن نوسّع `~`)، أو اضبط نموذج CLI صريحًا مع مسار أمر كامل.

### دعم بيئة Proxy ‏(نماذج الموفّر)

عندما تكون نماذج فهم الوسائط **الصوتية** و**الفيديو** المعتمدة على الموفّر مفعلة، فإن OpenClaw
يحترم متغيرات بيئة proxy الصادرة القياسية لاستدعاءات HTTP الخاصة بالموفّر:

- `HTTPS_PROXY`
- `HTTP_PROXY`
- `https_proxy`
- `http_proxy`

إذا لم تُضبط أي متغيرات بيئة proxy، فإن فهم الوسائط يستخدم اتصالًا مباشرًا.
إذا كانت قيمة proxy غير صالحة، يسجل OpenClaw تحذيرًا ثم يعود إلى الجلب
المباشر.

## القدرات (اختياري)

إذا ضبطت `capabilities`، فلن يعمل الإدخال إلا مع أنواع الوسائط تلك. وبالنسبة إلى القوائم
المشتركة، يستطيع OpenClaw استنتاج القيم الافتراضية:

- `openai` و`anthropic` و`minimax`: **صورة**
- `minimax-portal`: **صورة**
- `moonshot`: **صورة + فيديو**
- `openrouter`: **صورة**
- `google` ‏(Gemini API): **صورة + صوت + فيديو**
- `qwen`: **صورة + فيديو**
- `mistral`: **صوت**
- `zai`: **صورة**
- `groq`: **صوت**
- `deepgram`: **صوت**
- أي فهرس `models.providers.<id>.models[]` يحتوي على نموذج قادر على الصور:
  **صورة**

بالنسبة إلى إدخالات CLI، **اضبط `capabilities` صراحة** لتجنب المطابقات المفاجئة.
إذا حذفت `capabilities`، فسيكون الإدخال مؤهلًا للقائمة التي يظهر فيها.

## مصفوفة دعم الموفّر (تكاملات OpenClaw)

| القدرة | تكامل الموفّر | ملاحظات |
| ---------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| الصورة | OpenAI وOpenRouter وAnthropic وGoogle وMiniMax وMoonshot وQwen وZ.AI وموفّرو الإعداد | تسجل إضافات vendor دعم الصور؛ ويستخدم كل من MiniMax وMiniMax OAuth الموفّر `MiniMax-VL-01`؛ ويتم تسجيل موفّري الإعداد القادرين على الصور تلقائيًا. |
| الصوت | OpenAI وGroq وDeepgram وGoogle وMistral | نسخ عبر الموفّر (Whisper/Deepgram/Gemini/Voxtral). |
| الفيديو | Google وQwen وMoonshot | فهم فيديو عبر الموفّر بواسطة إضافات vendor؛ ويستخدم فهم الفيديو في Qwen نقاط نهاية DashScope Standard. |

ملاحظة MiniMax:

- يأتي فهم الصور في `minimax` و`minimax-portal` من
  موفّر الوسائط `MiniMax-VL-01` المملوك للإضافة.
- لا يزال فهرس النصوص المضمّن لـ MiniMax يبدأ كنصي فقط؛ وتُنشئ
  إدخالات `models.providers.minimax` الصريحة مراجع دردشة M2.7 القادرة على الصور.

## إرشادات اختيار النموذج

- فضّل أقوى نموذج من أحدث الأجيال متاح لكل قدرة وسائط عندما تكون الجودة والسلامة مهمتين.
- بالنسبة إلى الوكلاء المفعلة لديهم الأدوات ويتعاملون مع مدخلات غير موثوقة، تجنب نماذج الوسائط الأقدم/الأضعف.
- احتفظ بمسار احتياطي واحد على الأقل لكل قدرة من أجل التوافر (نموذج عالي الجودة + نموذج أسرع/أرخص).
- تُعد المسارات الاحتياطية عبر CLI ‏(`whisper-cli` و`whisper` و`gemini`) مفيدة عندما لا تتوفر واجهات API الخاصة بالموفّرين.
- ملاحظة `parakeet-mlx`: مع `--output-dir`، يقرأ OpenClaw الملف `<output-dir>/<media-basename>.txt` عندما يكون تنسيق الإخراج هو `txt` ‏(أو غير محدد)؛ أما التنسيقات غير `txt` فتعود احتياطيًا إلى stdout.

## سياسة المرفقات

يتحكم `attachments` لكل قدرة في المرفقات التي تتم معالجتها:

- `mode`: ‏`first` ‏(الافتراضي) أو `all`
- `maxAttachments`: حد أقصى للعدد المعالَج (الافتراضي **1**)
- `prefer`: ‏`first` أو `last` أو `path` أو `url`

عندما تكون `mode: "all"`، يتم وسم المخرجات بالشكل `[Image 1/2]` و`[Audio 2/2]`، وهكذا.

سلوك استخراج مرفقات الملفات:

- يُغلَّف النص المستخرج من الملفات بوصفه **محتوى خارجيًا غير موثوق** قبل
  إلحاقه بموجّه الوسائط.
- تستخدم الكتلة المحقونة علامات حدود صريحة مثل
  `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` /
  `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` وتتضمن سطر بيانات وصفية
  `Source: External`.
- يتعمد مسار استخراج المرفقات هذا حذف لافتة
  `SECURITY NOTICE:` الطويلة لتجنب تضخيم موجّه الوسائط؛ لكن تظل علامات
  الحدود والبيانات الوصفية موجودة.
- إذا لم يكن للملف نص قابل للاستخراج، يحقن OpenClaw القيمة `[No extractable text]`.
- إذا عاد PDF في هذا المسار احتياطيًا إلى صور صفحات معروضة، فإن موجّه الوسائط يحتفظ
  بالنص النائب `[PDF content rendered to images; images not forwarded to model]`
  لأن خطوة استخراج المرفقات هذه تمرر كتل نصية، وليس صور PDF المعروضة.

## أمثلة على الإعداد

### 1) قائمة نماذج مشتركة + تجاوزات

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-5.4-mini", capabilities: ["image"] },
        {
          provider: "google",
          model: "gemini-3-flash-preview",
          capabilities: ["image", "audio", "video"],
        },
        {
          type: "cli",
          command: "gemini",
          args: [
            "-m",
            "gemini-3-flash",
            "--allowed-tools",
            "read_file",
            "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
          ],
          capabilities: ["image", "video"],
        },
      ],
      audio: {
        attachments: { mode: "all", maxAttachments: 2 },
      },
      video: {
        maxChars: 500,
      },
    },
  },
}
```

### 2) الصوت + الفيديو فقط (الصورة معطلة)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          {
            type: "cli",
            command: "whisper",
            args: ["--model", "base", "{{MediaPath}}"],
          },
        ],
      },
      video: {
        enabled: true,
        maxChars: 500,
        models: [
          { provider: "google", model: "gemini-3-flash-preview" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
            ],
          },
        ],
      },
    },
  },
}
```

### 3) فهم الصور الاختياري

```json5
{
  tools: {
    media: {
      image: {
        enabled: true,
        maxBytes: 10485760,
        maxChars: 500,
        models: [
          { provider: "openai", model: "gpt-5.4-mini" },
          { provider: "anthropic", model: "claude-opus-4-6" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
            ],
          },
        ],
      },
    },
  },
}
```

### 4) إدخال واحد متعدد الوسائط (قدرات صريحة)

```json5
{
  tools: {
    media: {
      image: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      audio: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      video: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
    },
  },
}
```

## إخراج الحالة

عندما يعمل فهم الوسائط، يتضمن `/status` سطر ملخص قصيرًا:

```
📎 Media: image ok (openai/gpt-5.4-mini) · audio skipped (maxBytes)
```

ويعرض هذا نتائج كل قدرة والموفّر/النموذج المختار عند الاقتضاء.

## ملاحظات

- يكون الفهم **بأفضل جهد**. ولا تمنع الأخطاء الردود.
- تظل المرفقات تُمرَّر إلى النماذج حتى عندما يكون الفهم معطلًا.
- استخدم `scope` لتقييد الأماكن التي يعمل فيها الفهم (مثل الرسائل المباشرة فقط).

## مستندات ذات صلة

- [التهيئة](/gateway/configuration)
- [دعم الصور والوسائط](/nodes/images)
