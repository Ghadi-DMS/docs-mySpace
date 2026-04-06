---
read_when:
    - إضافة أو تعديل CLI النماذج (`models list/set/scan/aliases/fallbacks`)
    - تغيير سلوك البدائل الاحتياطية للنموذج أو تجربة اختيار النموذج
    - تحديث مجسات فحص النموذج (الأدوات/الصور)
summary: 'CLI النماذج: عرض، تعيين، أسماء مستعارة، بدائل احتياطية، فحص، حالة'
title: CLI النماذج
x-i18n:
    generated_at: "2026-04-06T03:07:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299602ccbe0c3d6bbdb2deab22bc60e1300ef6843ed0b8b36be574cc0213c155
    source_path: concepts/models.md
    workflow: 15
---

# CLI النماذج

راجع [/concepts/model-failover](/ar/concepts/model-failover) لمعرفة تدوير ملفات
تعريف المصادقة، وفترات التهدئة، وكيفية تفاعل ذلك مع البدائل الاحتياطية.
وللاطلاع على نظرة عامة سريعة على الموفرين مع أمثلة: [/concepts/model-providers](/ar/concepts/model-providers).

## كيف يعمل اختيار النموذج

يختار OpenClaw النماذج بهذا الترتيب:

1. النموذج **الأساسي** (`agents.defaults.model.primary` أو `agents.defaults.model`).
2. **البدائل الاحتياطية** في `agents.defaults.model.fallbacks` (بالترتيب).
3. يحدث **التحويل الاحتياطي لمصادقة الموفر** داخل الموفر قبل الانتقال إلى
   النموذج التالي.

ذو صلة:

- `agents.defaults.models` هو قائمة السماح/الفهرس للنماذج التي يمكن لـ OpenClaw استخدامها (بالإضافة إلى الأسماء المستعارة).
- يُستخدم `agents.defaults.imageModel` **فقط عندما** لا يستطيع النموذج الأساسي قبول الصور.
- يُستخدم `agents.defaults.pdfModel` بواسطة أداة `pdf`. وإذا لم يتم تعيينه، فستعود الأداة إلى `agents.defaults.imageModel`، ثم إلى النموذج المحلول للجلسة/الافتراضي.
- يُستخدم `agents.defaults.imageGenerationModel` بواسطة إمكانية إنشاء الصور المشتركة. وإذا لم يتم تعيينه، يظل `image_generate` قادرًا على استنتاج افتراضي موفر مدعوم بالمصادقة. ويحاول أولًا موفر الافتراضي الحالي، ثم بقية موفري إنشاء الصور المسجلين بترتيب معرّف الموفر. وإذا عيّنت موفرًا/نموذجًا محددًا، فعليك أيضًا تهيئة مصادقة/مفتاح API لذلك الموفر.
- يُستخدم `agents.defaults.musicGenerationModel` بواسطة إمكانية إنشاء الموسيقى المشتركة. وإذا لم يتم تعيينه، يظل `music_generate` قادرًا على استنتاج افتراضي موفر مدعوم بالمصادقة. ويحاول أولًا موفر الافتراضي الحالي، ثم بقية موفري إنشاء الموسيقى المسجلين بترتيب معرّف الموفر. وإذا عيّنت موفرًا/نموذجًا محددًا، فعليك أيضًا تهيئة مصادقة/مفتاح API لذلك الموفر.
- يُستخدم `agents.defaults.videoGenerationModel` بواسطة إمكانية إنشاء الفيديو المشتركة. وإذا لم يتم تعيينه، يظل `video_generate` قادرًا على استنتاج افتراضي موفر مدعوم بالمصادقة. ويحاول أولًا موفر الافتراضي الحالي، ثم بقية موفري إنشاء الفيديو المسجلين بترتيب معرّف الموفر. وإذا عيّنت موفرًا/نموذجًا محددًا، فعليك أيضًا تهيئة مصادقة/مفتاح API لذلك الموفر.
- يمكن للإعدادات الافتراضية الخاصة بكل وكيل تجاوز `agents.defaults.model` عبر `agents.list[].model` بالإضافة إلى الربط (راجع [/concepts/multi-agent](/ar/concepts/multi-agent)).

## سياسة سريعة للنموذج

- اضبط نموذجك الأساسي على أقوى نموذج من أحدث جيل متاح لك.
- استخدم البدائل الاحتياطية للمهام الحساسة من ناحية التكلفة/زمن الاستجابة ولمحادثات الأقل أهمية.
- بالنسبة إلى الوكلاء المفعلة لهم الأدوات أو المدخلات غير الموثوقة، تجنب مستويات النماذج الأقدم/الأضعف.

## الإعداد الأولي (موصى به)

إذا كنت لا تريد تعديل الإعدادات يدويًا، فشغّل الإعداد الأولي:

```bash
openclaw onboard
```

يمكنه إعداد النموذج + المصادقة للموفرين الشائعين، بما في ذلك **اشتراك OpenAI Code (Codex)**
(OAuth) و**Anthropic** (مفتاح API أو Claude CLI).

## مفاتيح الإعدادات (نظرة عامة)

- `agents.defaults.model.primary` و`agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` و`agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` و`agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` و`agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` و`agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models` (قائمة السماح + الأسماء المستعارة + معاملات الموفر)
- `models.providers` (موفرون مخصصون يُكتبون إلى `models.json`)

تُطبّع مراجع النماذج إلى أحرف صغيرة. كما تُطبّع الأسماء المستعارة للموفر مثل `z.ai/*`
إلى `zai/*`.

توجد أمثلة إعدادات الموفرين (بما في ذلك OpenCode) في
[/providers/opencode](/ar/providers/opencode).

## "Model is not allowed" (ولماذا تتوقف الردود)

إذا تم تعيين `agents.defaults.models`، فإنه يصبح **قائمة السماح** لأمر `/model` ولتجاوزات
الجلسة. وعندما يختار المستخدم نموذجًا غير موجود في قائمة السماح هذه،
فإن OpenClaw يعيد:

```
Model "provider/model" is not allowed. Use /model to list available models.
```

يحدث هذا **قبل** إنشاء رد عادي، لذلك قد تبدو الرسالة وكأنها
"لم ترد". والحل هو أحد ما يلي:

- إضافة النموذج إلى `agents.defaults.models`، أو
- مسح قائمة السماح (إزالة `agents.defaults.models`)، أو
- اختيار نموذج من `/model list`.

مثال على إعداد قائمة السماح:

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-6" },
    models: {
      "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## تبديل النماذج في الدردشة (`/model`)

يمكنك تبديل النماذج للجلسة الحالية من دون إعادة التشغيل:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

ملاحظات:

- `/model` (وكذلك `/model list`) عبارة عن أداة اختيار مدمجة ومرقمة (عائلة النموذج + الموفرون المتاحون).
- على Discord، يفتح `/model` و`/models` أداة اختيار تفاعلية مع قوائم منسدلة للموفر والنموذج بالإضافة إلى خطوة Submit.
- يختار `/model <#>` من أداة الاختيار هذه.
- يحفظ `/model` اختيار الجلسة الجديد فورًا.
- إذا كان الوكيل في وضع الخمول، فسيستخدم التشغيل التالي النموذج الجديد مباشرة.
- إذا كان التشغيل نشطًا بالفعل، يضع OpenClaw علامة على التبديل المباشر على أنه معلّق ولا يعيد التشغيل إلى النموذج الجديد إلا عند نقطة إعادة محاولة نظيفة.
- إذا كان نشاط الأداة أو إخراج الرد قد بدأ بالفعل، فقد يظل التبديل المعلّق في قائمة الانتظار حتى فرصة إعادة محاولة لاحقة أو حتى دور المستخدم التالي.
- `/model status` هو العرض التفصيلي (مرشحو المصادقة، وعند التهيئة، `baseUrl` لنقطة نهاية الموفر + وضع `api`).
- تُحلل مراجع النماذج عبر التقسيم عند **أول** `/`. استخدم `provider/model` عند كتابة `/model <ref>`.
- إذا كان معرّف النموذج نفسه يحتوي على `/` (بنمط OpenRouter)، فيجب تضمين بادئة الموفر (مثال: `/model openrouter/moonshotai/kimi-k2`).
- إذا حذفت الموفر، فسيحل OpenClaw الإدخال بهذا الترتيب:
  1. مطابقة الاسم المستعار
  2. مطابقة فريدة لموفر مُهيأ لذلك مع معرّف النموذج الدقيق غير المسبوق ببادئة
  3. الرجوع الاحتياطي القديم إلى الموفر الافتراضي المُهيأ
     إذا لم يعد ذلك الموفر يعرض النموذج الافتراضي المُهيأ، فسيرجع OpenClaw
     بدلًا من ذلك إلى أول موفر/نموذج مُهيأ لتجنب
     إظهار افتراضي قديم لموفر تمت إزالته.

السلوك الكامل للأمر/الإعدادات: [أوامر الشرطة المائلة](/ar/tools/slash-commands).

## أوامر CLI

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models` (من دون أمر فرعي) هو اختصار لـ `models status`.

### `models list`

يعرض النماذج المُهيأة افتراضيًا. العلامات المفيدة:

- `--all`: الفهرس الكامل
- `--local`: الموفّرون المحليون فقط
- `--provider <name>`: التصفية حسب الموفر
- `--plain`: نموذج واحد في كل سطر
- `--json`: مخرجات قابلة للقراءة آليًا

### `models status`

يعرض النموذج الأساسي المحلول، والبدائل الاحتياطية، ونموذج الصور، ونظرة عامة على مصادقة
الموفرين المُهيئين. كما يُظهر حالة انتهاء صلاحية OAuth لملفات التعريف الموجودة
في مخزن المصادقة (ويُحذر ضمن 24 ساعة افتراضيًا). يطبع `--plain` النموذج
الأساسي المحلول فقط.
تُعرض حالة OAuth دائمًا (وتُضمّن في مخرجات `--json`). وإذا لم يكن لدى موفر
مُهيأ أي بيانات اعتماد، فسيطبع `models status` قسم **Missing auth**.
يتضمن JSON `auth.oauth` (نافذة التحذير + ملفات التعريف) و`auth.providers`
(المصادقة الفعالة لكل موفر، بما في ذلك بيانات الاعتماد المعتمدة على env). ويقتصر `auth.oauth`
على سلامة ملفات تعريف مخزن المصادقة فقط؛ ولا تظهر الموفّرات التي تعتمد على env فقط هناك.
استخدم `--check` للأتمتة (الخروج بـ `1` عند الفقدان/انتهاء الصلاحية، و`2` عند قرب الانتهاء).
واستخدم `--probe` لفحوصات المصادقة الحية؛ ويمكن أن تأتي صفوف الفحص من ملفات تعريف المصادقة أو بيانات اعتماد env
أو `models.json`.
إذا أغفل `auth.order.<provider>` الصريح ملف تعريف مخزنًا، فإن الفحص يبلغ عن
`excluded_by_auth_order` بدلًا من تجربته. وإذا كانت المصادقة موجودة ولكن لم يمكن حل نموذج
قابل للفحص لذلك الموفر، فإن الفحص يبلغ عن `status: no_model`.

يعتمد اختيار المصادقة على الموفر/الحساب. وبالنسبة إلى مضيفي Gateway
الدائمين، تكون مفاتيح API عادةً الأكثر قابلية للتنبؤ؛ كما أن إعادة استخدام Claude CLI
وملفات تعريف OAuth/token الحالية لـ Anthropic مدعومة أيضًا.

مثال (Claude CLI):

```bash
claude auth login
openclaw models status
```

## الفحص (نماذج OpenRouter المجانية)

يفحص `openclaw models scan` **فهرس النماذج المجانية** في OpenRouter، ويمكنه
اختياريًا فحص دعم النماذج للأدوات والصور.

العلامات الأساسية:

- `--no-probe`: تخطي الفحوصات الحية (البيانات الوصفية فقط)
- `--min-params <b>`: الحد الأدنى لحجم المعلمات (بالمليارات)
- `--max-age-days <days>`: تخطي النماذج الأقدم
- `--provider <name>`: مرشح بادئة الموفر
- `--max-candidates <n>`: حجم قائمة البدائل الاحتياطية
- `--set-default`: تعيين `agents.defaults.model.primary` إلى أول اختيار
- `--set-image`: تعيين `agents.defaults.imageModel.primary` إلى أول اختيار للصورة

يتطلب الفحص مفتاح API لـ OpenRouter (من ملفات تعريف المصادقة أو
`OPENROUTER_API_KEY`). ومن دون مفتاح، استخدم `--no-probe` لعرض المرشحين فقط.

تُرتب نتائج الفحص حسب:

1. دعم الصور
2. زمن استجابة الأدوات
3. حجم السياق
4. عدد المعلمات

الإدخال

- قائمة OpenRouter `/models` (مرشح `:free`)
- يتطلب مفتاح API لـ OpenRouter من ملفات تعريف المصادقة أو `OPENROUTER_API_KEY` (راجع [/environment](/ar/help/environment))
- مرشحات اختيارية: `--max-age-days` و`--min-params` و`--provider` و`--max-candidates`
- عناصر التحكم في الفحص: `--timeout` و`--concurrency`

عند التشغيل في TTY، يمكنك اختيار البدائل الاحتياطية بشكل تفاعلي. وفي الوضع غير التفاعلي،
مرّر `--yes` لقبول القيم الافتراضية.

## سجل النماذج (`models.json`)

تُكتب الموفّرات المخصصة في `models.providers` إلى `models.json` تحت
دليل الوكيل (الافتراضي `~/.openclaw/agents/<agentId>/agent/models.json`). ويجري دمج هذا الملف
افتراضيًا ما لم يتم تعيين `models.mode` إلى `replace`.

أولوية وضع الدمج لمعرّفات الموفّرين المتطابقة:

- يفوز `baseUrl` غير الفارغ الموجود بالفعل في `models.json` الخاص بالوكيل.
- يفوز `apiKey` غير الفارغ في `models.json` الخاص بالوكيل فقط عندما لا يكون ذلك الموفر مُدارًا بواسطة SecretRef في سياق الإعدادات/ملف تعريف المصادقة الحالي.
- تُحدّث قيم `apiKey` للموفر المُدار بواسطة SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec) بدلًا من الاستمرار في حفظ الأسرار المحلولة.
- تُحدّث قيم ترويسة الموفر المُدار بواسطة SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec).
- تعود قيم `apiKey`/`baseUrl` الفارغة أو المفقودة في الوكيل إلى `models.providers` في الإعدادات.
- تُحدّث حقول الموفر الأخرى من الإعدادات وبيانات الفهرس المُطبّعة.

استمرار العلامات يعتمد على المصدر بوصفه المرجع المعتمد: يكتب OpenClaw العلامات من لقطة إعدادات المصدر النشطة (قبل الحل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.
وينطبق هذا كلما أعاد OpenClaw إنشاء `models.json`، بما في ذلك المسارات التي تقودها الأوامر مثل `openclaw agent`.

## ذو صلة

- [موفرو النماذج](/ar/concepts/model-providers) — توجيه الموفر والمصادقة
- [التحويل الاحتياطي للنموذج](/ar/concepts/model-failover) — سلاسل البدائل الاحتياطية
- [إنشاء الصور](/ar/tools/image-generation) — إعداد نموذج الصور
- [إنشاء الموسيقى](/tools/music-generation) — إعداد نموذج الموسيقى
- [إنشاء الفيديو](/tools/video-generation) — إعداد نموذج الفيديو
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults) — مفاتيح إعدادات النموذج
