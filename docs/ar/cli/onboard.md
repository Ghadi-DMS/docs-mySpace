---
read_when:
    - تريد إعدادًا موجّهًا لـ gateway ومساحة العمل والمصادقة والقنوات وSkills
summary: مرجع CLI للأمر `openclaw onboard` (الإعداد التفاعلي الأولي)
title: onboard
x-i18n:
    generated_at: "2026-04-05T12:39:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6db61c8002c9e82e48ff44f72e176b58ad85fad5cb8434687455ed40add8cc2a
    source_path: cli/onboard.md
    workflow: 15
---

# `openclaw onboard`

الإعداد التفاعلي الأولي لإعداد Gateway محلي أو بعيد.

## أدلة ذات صلة

- مركز إعداد CLI الأولي: [Onboarding (CLI)](/ar/start/wizard)
- نظرة عامة على الإعداد الأولي: [Onboarding Overview](/start/onboarding-overview)
- مرجع إعداد CLI الأولي: [CLI Setup Reference](/start/wizard-cli-reference)
- أتمتة CLI: [CLI Automation](/start/wizard-cli-automation)
- الإعداد الأولي على macOS: [Onboarding (macOS App)](/start/onboarding)

## أمثلة

```bash
openclaw onboard
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

بالنسبة إلى أهداف `ws://` ذات النص الصريح على الشبكات الخاصة (الشبكات الموثوقة فقط)، اضبط
`OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` في بيئة عملية الإعداد الأولي.

مزود مخصص غير تفاعلي:

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai
```

تكون `--custom-api-key` اختيارية في الوضع غير التفاعلي. إذا لم تُمرَّر، يتحقق الإعداد الأولي من `CUSTOM_API_KEY`.

Ollama غير التفاعلي:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

القيمة الافتراضية لـ `--custom-base-url` هي `http://127.0.0.1:11434`. وتكون `--custom-model-id` اختيارية؛ وإذا لم تُمرَّر، يستخدم الإعداد الأولي القيم الافتراضية المقترحة من Ollama. كما تعمل هنا أيضًا معرّفات النماذج السحابية مثل `kimi-k2.5:cloud`.

خزّن مفاتيح المزوّد كمراجع بدلًا من نص صريح:

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

مع `--secret-input-mode ref`، يكتب الإعداد الأولي مراجع مدعومة بـ env بدلًا من قيم المفاتيح النصية الصريحة.
وبالنسبة إلى المزوّدين المدعومين بملفات تعريف المصادقة، يكتب هذا إدخالات `keyRef`؛ أما بالنسبة إلى المزوّدين المخصصين، فيكتب `models.providers.<id>.apiKey` كمرجع env (على سبيل المثال `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`).

عقد وضع `ref` غير التفاعلي:

- اضبط متغير env الخاص بالمزوّد في بيئة عملية الإعداد الأولي (على سبيل المثال `OPENAI_API_KEY`).
- لا تمرر علامات المفاتيح المضمنة (مثل `--openai-api-key`) إلا إذا كان متغير env هذا مضبوطًا أيضًا.
- إذا مُرِّرت علامة مفتاح مضمن من دون متغير env المطلوب، يفشل الإعداد الأولي بسرعة مع إرشادات.

خيارات token الخاصة بـ gateway في الوضع غير التفاعلي:

- `--gateway-auth token --gateway-token <token>` يخزن token نصيًا صريحًا.
- `--gateway-auth token --gateway-token-ref-env <name>` يخزن `gateway.auth.token` كمرجع env SecretRef.
- لا يمكن استخدام `--gateway-token` و`--gateway-token-ref-env` معًا.
- يتطلب `--gateway-token-ref-env` متغير env غير فارغ في بيئة عملية الإعداد الأولي.
- مع `--install-daemon`، عندما تتطلب مصادقة token وجود token، يتم التحقق من tokens الخاصة بـ gateway المُدارة بواسطة SecretRef ولكن لا تُحفَظ كنص صريح محلول ضمن بيانات تعريف بيئة خدمة المشرف.
- مع `--install-daemon`، إذا كان وضع token يتطلب token وكان المرجع SecretRef المكوَّن لـ token غير قابل للحل، يفشل الإعداد الأولي بإغلاق آمن مع إرشادات للمعالجة.
- مع `--install-daemon`، إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مكوّنين وكان `gateway.auth.mode` غير مضبوط، يمنع الإعداد الأولي التثبيت حتى يُضبط الوضع صراحةً.
- يكتب الإعداد الأولي المحلي `gateway.mode="local"` في الإعداد. وإذا كان ملف إعداد لاحق يفتقد `gateway.mode`، فاعتبر ذلك تلفًا في الإعداد أو تعديلًا يدويًا غير مكتمل، وليس اختصارًا صالحًا للوضع المحلي.
- يمثل `--allow-unconfigured` منفذ هروب منفصلًا لوقت تشغيل gateway. وهو لا يعني أن الإعداد الأولي يمكنه حذف `gateway.mode`.

مثال:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

سلامة gateway المحلي في الوضع غير التفاعلي:

- ما لم تمرر `--skip-health`، ينتظر الإعداد الأولي وصول gateway محليًا قبل أن يخرج بنجاح.
- يبدأ `--install-daemon` أولًا مسار تثبيت gateway المُدار. ومن دونه، يجب أن يكون لديك gateway محلي يعمل بالفعل، مثل `openclaw gateway run`.
- إذا كنت تريد فقط كتابة الإعداد/مساحة العمل/تهيئة البداية ضمن الأتمتة، فاستخدم `--skip-health`.
- على Windows الأصلي، يحاول `--install-daemon` أولًا استخدام Scheduled Tasks ثم يعود إلى عنصر تسجيل دخول لكل مستخدم في مجلد Startup إذا رُفض إنشاء المهمة.

سلوك الإعداد الأولي التفاعلي مع وضع المرجع:

- اختر **Use secret reference** عندما يُطلب منك ذلك.
- ثم اختر أحد الخيارين:
  - متغير بيئة
  - مزود أسرار مُعد (`file` أو `exec`)
- يُجري الإعداد الأولي تحققًا تمهيديًا سريعًا قبل حفظ المرجع.
  - إذا فشل التحقق، يعرض الإعداد الأولي الخطأ ويتيح لك إعادة المحاولة.

خيارات نقطة نهاية Z.AI غير التفاعلية:

ملاحظة: يقوم `--auth-choice zai-api-key` الآن بالكشف التلقائي عن أفضل نقطة نهاية Z.AI لمفتاحك (ويفضّل واجهة API العامة مع `zai/glm-5`).
وإذا كنت تريد تحديدًا نقاط نهاية GLM Coding Plan، فاختر `zai-coding-global` أو `zai-coding-cn`.

```bash
# اختيار نقطة النهاية دون مطالبة
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# خيارات نقاط نهاية Z.AI الأخرى:
# --auth-choice zai-coding-cn
# --auth-choice zai-global
# --auth-choice zai-cn
```

مثال Mistral غير تفاعلي:

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

ملاحظات التدفق:

- `quickstart`: أقل عدد من المطالبات، ويولّد gateway token تلقائيًا.
- `manual`: مطالبات كاملة للمنفذ/الربط/المصادقة (اسم بديل لـ `advanced`).
- عندما يشير خيار المصادقة إلى مزوّد مفضّل، يقوم الإعداد الأولي بتصفية
  منتقيات النموذج الافتراضي وقائمة السماح مسبقًا إلى ذلك المزوّد. وبالنسبة إلى Volcengine و
  BytePlus، يشمل ذلك أيضًا متغيرات coding-plan
  (`volcengine-plan/*`, `byteplus-plan/*`).
- إذا لم ينتج عن مرشح المزوّد المفضّل أي نماذج محمّلة بعد، يعود الإعداد الأولي
  إلى الكتالوج غير المصفّى بدلًا من ترك المنتقي فارغًا.
- في خطوة البحث على الويب، يمكن لبعض المزوّدين تشغيل
  مطالبات متابعة خاصة بالمزوّد:
  - يمكن لـ **Grok** تقديم إعداد `x_search` اختياري باستخدام `XAI_API_KEY`
    نفسه واختيار نموذج `x_search`.
  - يمكن لـ **Kimi** السؤال عن منطقة Moonshot API ‏(`api.moonshot.ai` مقابل
    `api.moonshot.cn`) ونموذج البحث على الويب الافتراضي لـ Kimi.
- سلوك نطاق الرسائل المباشرة في الإعداد الأولي المحلي: [CLI Setup Reference](/start/wizard-cli-reference#outputs-and-internals).
- أسرع طريقة لأول دردشة: `openclaw dashboard` ‏(Control UI، من دون إعداد قناة).
- Custom Provider: صِل أي نقطة نهاية متوافقة مع OpenAI أو Anthropic،
  بما في ذلك المزوّدين المستضافين غير المدرجين. استخدم Unknown للكشف التلقائي.

## أوامر متابعة شائعة

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
لا يعني `--json` الوضع غير التفاعلي تلقائيًا. استخدم `--non-interactive` في النصوص البرمجية.
</Note>
