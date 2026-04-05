---
read_when:
    - إضافة أو تعديل CLI النماذج (models list/set/scan/aliases/fallbacks)
    - تغيير سلوك الرجوع إلى النموذج أو تجربة اختيار النموذج
    - تحديث فحوصات مسح النماذج (tools/images)
summary: 'CLI النماذج: عرض، وتعيين، وأسماء مستعارة، وبدائل، وفحص، وحالة'
title: CLI النماذج
x-i18n:
    generated_at: "2026-04-05T12:41:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: e08f7e50da263895dae2bd2b8dc327972ea322615f8d1918ddbd26bb0fb24840
    source_path: concepts/models.md
    workflow: 15
---

# CLI النماذج

راجع [/concepts/model-failover](/concepts/model-failover) لمعرفة تناوب
ملف تعريف auth، وفترات التهدئة، وكيفية تفاعل ذلك مع البدائل.
وللحصول على نظرة عامة سريعة على الموفّرين مع أمثلة: [/concepts/model-providers](/concepts/model-providers).

## كيف يعمل اختيار النموذج

يختار OpenClaw النماذج بهذا الترتيب:

1. النموذج **الأساسي** (`agents.defaults.model.primary` أو `agents.defaults.model`).
2. **البدائل** في `agents.defaults.model.fallbacks` (بالترتيب).
3. يحدث **الرجوع إلى auth الخاص بالموفّر** داخل الموفّر قبل الانتقال إلى
   النموذج التالي.

ذو صلة:

- `agents.defaults.models` هي قائمة السماح/الفهرس للنماذج التي يمكن لـ OpenClaw استخدامها (بالإضافة إلى الأسماء المستعارة).
- يُستخدم `agents.defaults.imageModel` **فقط عندما** يتعذر على النموذج الأساسي قبول الصور.
- يُستخدم `agents.defaults.pdfModel` بواسطة أداة `pdf`. وإذا تم حذفه، فسترجع الأداة
  إلى `agents.defaults.imageModel`، ثم إلى النموذج المحلول للجلسة/الافتراضي.
- يُستخدم `agents.defaults.imageGenerationModel` بواسطة الإمكانية المشتركة لتوليد الصور. وإذا تم حذفه، فلا يزال بإمكان `image_generate` استنتاج موفّر افتراضي مدعوم بـ auth. يحاول أولًا موفّر الخدمة الافتراضي الحالي، ثم بقية موفّري توليد الصور المسجلين بترتيب معرّف الموفّر. إذا عيّنت موفّرًا/نموذجًا محددًا، فقم أيضًا بإعداد auth/API key لذلك الموفّر.
- يُستخدم `agents.defaults.videoGenerationModel` بواسطة الإمكانية المشتركة لتوليد الفيديو. وعلى خلاف توليد الصور، لا يستنتج هذا موفّرًا افتراضيًا اليوم. عيّن `provider/model` صريحًا مثل `qwen/wan2.6-t2v`، واضبط أيضًا auth/API key لذلك الموفّر.
- يمكن للإعدادات الافتراضية لكل وكيل تجاوز `agents.defaults.model` عبر `agents.list[].model` بالإضافة إلى عمليات الربط (راجع [/concepts/multi-agent](/concepts/multi-agent)).

## سياسة سريعة للنموذج

- اضبط النموذج الأساسي على أقوى نموذج من أحدث جيل متاح لك.
- استخدم البدائل للمهام الحساسة للتكلفة/الكمون ولمحادثات الرهانات المنخفضة.
- بالنسبة إلى الوكلاء الممكّنين بالأدوات أو المدخلات غير الموثوقة، تجنب مستويات النماذج الأقدم/الأضعف.

## onboarding (موصى به)

إذا كنت لا تريد تعديل الإعدادات يدويًا، فشغّل onboarding:

```bash
openclaw onboard
```

يمكنه إعداد النموذج + auth لموفّرين شائعين، بما في ذلك **اشتراك
OpenAI Code (Codex)** ‏(OAuth) و**Anthropic** ‏(API key أو Claude CLI).

## مفاتيح الإعدادات (نظرة عامة)

- `agents.defaults.model.primary` و`agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` و`agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` و`agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` و`agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` و`agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models` (قائمة السماح + الأسماء المستعارة + معلمات الموفّر)
- `models.providers` (موفّرون مخصصون يُكتبون في `models.json`)

تُطبَّع مراجع النماذج إلى أحرف صغيرة. كما تُطبَّع الأسماء المستعارة للموفّرين مثل `z.ai/*`
إلى `zai/*`.

توجد أمثلة إعداد الموفّرين (بما في ذلك OpenCode) في
[/providers/opencode](/providers/opencode).

## "النموذج غير مسموح به" (ولماذا تتوقف الردود)

إذا تم تعيين `agents.defaults.models`، فسيصبح **قائمة السماح** لـ `/model` و
لتجاوزات الجلسة. وعندما يختار المستخدم نموذجًا غير موجود في قائمة السماح تلك،
يعيد OpenClaw:

```
Model "provider/model" is not allowed. Use /model to list available models.
```

يحدث هذا **قبل** إنشاء رد عادي، لذلك قد يبدو أن الرسالة
"لم تستجب". والحل هو أحد ما يلي:

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

يمكنك تبديل النماذج للجلسة الحالية دون إعادة التشغيل:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

ملاحظات:

- يكون `/model` (و`/model list`) منتقيًا مدمجًا ومرقمًا (عائلة النموذج + الموفّرون المتاحون).
- على Discord، يفتح `/model` و`/models` منتقيًا تفاعليًا مع قوائم منسدلة للموفّر والنموذج بالإضافة إلى خطوة Submit.
- يختار `/model <#>` من ذلك المنتقي.
- يحفظ `/model` اختيار الجلسة الجديد فورًا.
- إذا كان الوكيل في وضع الخمول، فسيستخدم التشغيل التالي النموذج الجديد مباشرةً.
- إذا كان هناك تشغيل نشط بالفعل، يعلّم OpenClaw التبديل المباشر كمعلّق ولا يعيد التشغيل إلى النموذج الجديد إلا عند نقطة إعادة محاولة نظيفة.
- إذا كان نشاط الأداة أو مخرجات الرد قد بدأ بالفعل، فقد يظل التبديل المعلّق في قائمة الانتظار حتى فرصة إعادة محاولة لاحقة أو الدور التالي للمستخدم.
- يمثل `/model status` العرض التفصيلي (مرشحو auth و، عند الإعداد، `baseUrl` الخاص بنقطة نهاية الموفّر + وضع `api`).
- يتم تحليل مراجع النماذج بالتقسيم عند **أول** `/`. استخدم `provider/model` عند كتابة `/model <ref>`.
- إذا كان معرّف النموذج نفسه يحتوي على `/` (بنمط OpenRouter)، فيجب عليك تضمين بادئة الموفّر (مثال: `/model openrouter/moonshotai/kimi-k2`).
- إذا حذفت الموفّر، فسيحل OpenClaw الإدخال بهذا الترتيب:
  1. مطابقة الاسم المستعار
  2. مطابقة موفّر مهيأ فريدة لذلك المعرّف غير المسبوق ببادئة تمامًا
  3. رجوع قديم إلى الموفّر الافتراضي المهيأ
     إذا لم يعد ذلك الموفّر يكشف النموذج الافتراضي المهيأ، فإن OpenClaw
     يرجع بدلًا من ذلك إلى أول موفّر/نموذج مهيأ لتجنب
     إظهار افتراضي قديم تمت إزالته من الموفّر.

سلوك/إعداد الأمر الكامل: [Slash commands](/tools/slash-commands).

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

يُعد `openclaw models` (من دون أمر فرعي) اختصارًا لـ `models status`.

### `models list`

يعرض النماذج المهيأة افتراضيًا. العلامات المفيدة:

- `--all`: الفهرس الكامل
- `--local`: الموفّرون المحليون فقط
- `--provider <name>`: التصفية حسب الموفّر
- `--plain`: نموذج واحد في كل سطر
- `--json`: مخرجات قابلة للقراءة آليًا

### `models status`

يعرض النموذج الأساسي المحلول، والبدائل، ونموذج الصور، ونظرة عامة على auth
للموفّرين المهيئين. كما يعرض حالة انتهاء صلاحية OAuth لملفات التعريف الموجودة
في مخزن auth (ويحذر خلال 24 ساعة افتراضيًا). ويطبع `--plain` فقط
النموذج الأساسي المحلول.
تُعرض حالة OAuth دائمًا (وتُضمن في مخرجات `--json`). وإذا كان الموفّر المهيأ
لا يملك بيانات اعتماد، فإن `models status` يطبع قسم **Missing auth**.
ويتضمن JSON كلًا من `auth.oauth` (نافذة التحذير + ملفات التعريف) و`auth.providers`
(‏auth الفعّال لكل موفّر).
استخدم `--check` للأتمتة (خروج `1` عند الغياب/الانتهاء، و`2` عند قرب الانتهاء).
واستخدم `--probe` لفحوصات auth الحية؛ ويمكن أن تأتي صفوف الفحص من ملفات تعريف auth، أو بيانات اعتماد env،
أو `models.json`.
إذا كان `auth.order.<provider>` الصريح يحذف ملف تعريف مخزنًا، فإن تقرير الفحص يعرض
`excluded_by_auth_order` بدلًا من محاولة استخدامه. وإذا كان auth موجودًا ولكن لا يمكن حل
أي نموذج قابل للفحص لذلك الموفّر، فإن الفحص يعرض `status: no_model`.

يعتمد اختيار auth على الموفّر/الحساب. وبالنسبة إلى مضيفي البوابة الدائمين،
تكون API keys عادةً الأكثر قابلية للتنبؤ؛ كما أن إعادة استخدام Claude CLI
وملفات تعريف OAuth/token الحالية لـ Anthropic مدعومة أيضًا.

مثال (Claude CLI):

```bash
claude auth login
openclaw models status
```

## المسح (نماذج OpenRouter المجانية)

يفحص `openclaw models scan` **فهرس النماذج المجانية** في OpenRouter ويمكنه
اختياريًا فحص النماذج للتأكد من دعم الأدوات والصور.

العلامات الأساسية:

- `--no-probe`: تخطي الفحوصات الحية (البيانات الوصفية فقط)
- `--min-params <b>`: الحد الأدنى لحجم المعلمات (بالمليارات)
- `--max-age-days <days>`: تخطي النماذج الأقدم
- `--provider <name>`: تصفية حسب بادئة الموفّر
- `--max-candidates <n>`: حجم قائمة البدائل
- `--set-default`: تعيين `agents.defaults.model.primary` إلى أول اختيار
- `--set-image`: تعيين `agents.defaults.imageModel.primary` إلى أول اختيار للصورة

يتطلب الفحص OpenRouter API key (من ملفات تعريف auth أو
`OPENROUTER_API_KEY`). ومن دون مفتاح، استخدم `--no-probe` لإدراج المرشحين فقط.

يتم ترتيب نتائج المسح حسب:

1. دعم الصور
2. كمون الأدوات
3. حجم السياق
4. عدد المعلمات

الإدخال

- قائمة OpenRouter `/models` (تصفية `:free`)
- يتطلب OpenRouter API key من ملفات تعريف auth أو `OPENROUTER_API_KEY` (راجع [/environment](/help/environment))
- المرشحات الاختيارية: `--max-age-days` و`--min-params` و`--provider` و`--max-candidates`
- عناصر التحكم في الفحص: `--timeout` و`--concurrency`

عند التشغيل في TTY، يمكنك اختيار البدائل تفاعليًا. وفي الوضع غير التفاعلي،
مرر `--yes` لقبول الإعدادات الافتراضية.

## سجل النماذج (`models.json`)

تُكتب الموفّرات المخصصة في `models.providers` إلى `models.json` ضمن
دليل الوكيل (الافتراضي `~/.openclaw/agents/<agentId>/agent/models.json`). ويتم دمج هذا الملف
افتراضيًا ما لم يتم تعيين `models.mode` إلى `replace`.

أسبقية وضع الدمج عند تطابق معرّفات الموفّر:

- يفوز `baseUrl` غير الفارغ الموجود بالفعل في `models.json` الخاص بالوكيل.
- يفوز `apiKey` غير الفارغ في `models.json` الخاص بالوكيل فقط عندما لا يكون ذلك الموفّر مُدارًا عبر SecretRef في سياق config/auth-profile الحالي.
- يتم تحديث قيم `apiKey` الخاصة بالموفّر المُدار عبر SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec) بدلًا من حفظ الأسرار المحلولة.
- يتم تحديث قيم ترويسات الموفّر المُدار عبر SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec).
- ترجع قيم `apiKey`/`baseUrl` الفارغة أو المفقودة في الوكيل إلى `models.providers` في config.
- يتم تحديث حقول الموفّر الأخرى من config وبيانات الفهرس المطبّعة.

تكون استمرارية العلامات مرجعية المصدر: يكتب OpenClaw العلامات من لقطة config المصدر النشطة (قبل الحل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.
وينطبق هذا كلما أعاد OpenClaw إنشاء `models.json`، بما في ذلك المسارات التي تقودها الأوامر مثل `openclaw agent`.

## ذو صلة

- [Model Providers](/concepts/model-providers) — توجيه الموفّرين وauth
- [Model Failover](/concepts/model-failover) — سلاسل البدائل
- [Image Generation](/tools/image-generation) — إعداد نموذج الصور
- [Configuration Reference](/gateway/configuration-reference#agent-defaults) — مفاتيح إعدادات النموذج
