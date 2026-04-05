---
read_when:
    - تريد تغيير النماذج الافتراضية أو عرض حالة مصادقة المزوّد
    - تريد فحص النماذج/المزوّدات المتاحة وتصحيح ملفات تعريف المصادقة
summary: مرجع CLI للأمر `openclaw models` (status/list/set/scan، والأسماء البديلة، والبدائل الاحتياطية، والمصادقة)
title: models
x-i18n:
    generated_at: "2026-04-05T12:38:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 04ba33181d49b6bbf3b5d5fa413aa6b388c9f29fb9d4952055d68c79f7bcfea0
    source_path: cli/models.md
    workflow: 15
---

# `openclaw models`

اكتشاف النماذج وفحصها وإعدادها (النموذج الافتراضي، والبدائل الاحتياطية، وملفات تعريف المصادقة).

ذو صلة:

- المزوّدون + النماذج: [النماذج](/providers/models)
- إعداد مصادقة المزوّد: [البدء](/ar/start/getting-started)

## الأوامر الشائعة

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models scan
```

يعرض `openclaw models status` القيم الافتراضية/الاحتياطية المحلولة بالإضافة إلى نظرة عامة على المصادقة.
وعندما تكون لقطات استخدام المزوّد متاحة، يتضمن قسم حالة OAuth/API key
نوافذ استخدام المزوّد ولقطات الحصة.
مزوّدو نوافذ الاستخدام الحاليون: Anthropic وGitHub Copilot وGemini CLI وOpenAI
Codex وMiniMax وXiaomi وz.ai. تأتي مصادقة الاستخدام من خطافات خاصة بالمزوّد
عند توفرها؛ وإلا يعود OpenClaw إلى مطابقة بيانات اعتماد OAuth/API-key
من ملفات تعريف المصادقة أو البيئة أو الإعداد.
أضف `--probe` لتشغيل مجسات مصادقة مباشرة مقابل كل ملف تعريف مزوّد مُعد.
المجسات هي طلبات حقيقية (وقد تستهلك tokens وتؤدي إلى حدود معدل).
استخدم `--agent <id>` لفحص حالة النموذج/المصادقة لوكيل مُعد. وعند عدم تحديده،
يستخدم الأمر `OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR` إذا كان مضبوطًا، وإلا
يستخدم الوكيل الافتراضي المُعد.
يمكن أن تأتي صفوف المجسات من ملفات تعريف المصادقة أو بيانات اعتماد البيئة أو `models.json`.

ملاحظات:

- يقبل `models set <model-or-alias>` التنسيق `provider/model` أو اسمًا بديلًا.
- تُحلّل مراجع النماذج عبر التقسيم عند **أول** `/`. وإذا كان معرّف النموذج يتضمن `/` (بنمط OpenRouter)، فأدرج بادئة المزوّد (مثال: `openrouter/moonshotai/kimi-k2`).
- إذا حذفت المزوّد، فسيحل OpenClaw الإدخال أولًا على أنه اسم بديل، ثم
  على أنه تطابق فريد لمزوّد مُعد لذلك المعرّف الدقيق للنموذج، وبعدها فقط
  يعود إلى المزوّد الافتراضي المُعد مع تحذير إيقاف.
  وإذا لم يعد ذلك المزوّد يعرض النموذج الافتراضي المُعد، فسيعود OpenClaw
  إلى أول مزوّد/نموذج مُعد بدلًا من إظهار
  قيمة افتراضية قديمة من مزوّد محذوف.
- قد يعرض `models status` ‏`marker(<value>)` في مخرجات المصادقة للعناصر النائبة غير السرية (مثل `OPENAI_API_KEY` و`secretref-managed` و`minimax-oauth` و`oauth:chutes` و`ollama-local`) بدلًا من إخفائها كأسرار.

### `models status`

الخيارات:

- `--json`
- `--plain`
- `--check` ‏(رمز الخروج 1=منتهي/مفقود، 2=قارب على الانتهاء)
- `--probe` ‏(مجس مباشر لملفات تعريف المصادقة المُعدة)
- `--probe-provider <name>` ‏(فحص مزوّد واحد)
- `--probe-profile <id>` ‏(قابل للتكرار أو مفصول بفواصل لمعرّفات الملفات)
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`
- `--agent <id>` ‏(معرّف وكيل مُعد؛ يتجاوز `OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR`)

فئات حالة المجس:

- `ok`
- `auth`
- `rate_limit`
- `billing`
- `timeout`
- `format`
- `unknown`
- `no_model`

حالات التفاصيل/رموز السبب المتوقعة في المجس:

- `excluded_by_auth_order`: يوجد ملف تعريف مخزّن، لكن
  `auth.order.<provider>` الصريح استبعده، لذلك يبلغ المجس عن هذا الاستبعاد
  بدلًا من تجربته.
- `missing_credential`, `invalid_expires`, `expired`, `unresolved_ref`:
  ملف التعريف موجود لكنه غير مؤهل/غير قابل للحل.
- `no_model`: مصادقة المزوّد موجودة، لكن OpenClaw لم يتمكن من حل مرشح نموذج قابل للفحص لذلك المزوّد.

## الأسماء البديلة + البدائل الاحتياطية

```bash
openclaw models aliases list
openclaw models fallbacks list
```

## ملفات تعريف المصادقة

```bash
openclaw models auth add
openclaw models auth login --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token
```

`models auth add` هو مساعد المصادقة التفاعلي. ويمكنه تشغيل تدفق مصادقة مزوّد
(OAuth/API key) أو إرشادك إلى لصق token يدويًا، حسب
المزوّد الذي تختاره.

يشغّل `models auth login` تدفق مصادقة لمكون إضافي خاص بالمزوّد
(OAuth/API key). استخدم `openclaw plugins list` لمعرفة المزوّدين المثبتين.

أمثلة:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
openclaw models auth login --provider openai-codex --set-default
```

ملاحظات:

- يعيد `login --provider anthropic --method cli --set-default` استخدام تسجيل
  دخول Claude CLI محلي، ويعيد كتابة مسار النموذج الافتراضي الرئيسي لـ Anthropic إلى مرجع
  قانوني `claude-cli/claude-*`.
- يظل `setup-token` و`paste-token` أوامر tokens عامة للمزوّدين
  الذين يوفّرون أساليب مصادقة معتمدة على token.
- يتطلب `setup-token` طرفية تفاعلية ويشغّل أسلوب مصادقة token الخاص بالمزوّد
  (ويفترض افتراضيًا أسلوب `setup-token` لذلك المزوّد عندما يوفّره).
- يقبل `paste-token` سلسلة token مُولَّدة في مكان آخر أو من عملية أتمتة.
- يتطلب `paste-token` الخيار `--provider`، ويطلب قيمة token، ويكتبها
  إلى معرّف ملف التعريف الافتراضي `<provider>:manual` ما لم تمرر
  `--profile-id`.
- يخزن `paste-token --expires-in <duration>` وقت انتهاء token مطلقًا انطلاقًا من
  مدة نسبية مثل `365d` أو `12h`.
- ملاحظة فوترة Anthropic: نعتقد أن Claude Code CLI الاحتياطي مسموح على الأرجح لعمليات الأتمتة المحلية التي يديرها المستخدم استنادًا إلى وثائق CLI العامة من Anthropic. ومع ذلك، فإن سياسة Anthropic الخاصة بأحزمة الأطراف الثالثة تخلق قدرًا كافيًا من الغموض بشأن الاستخدام المدعوم بالاشتراك في المنتجات الخارجية، لذلك لا نوصي به للإنتاج. كما أبلغت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن مسار تسجيل الدخول إلى Claude ضمن **OpenClaw** يُعد استخدامًا لحزمة طرف ثالث ويتطلب **Extra Usage** تُحتسب بشكل منفصل عن الاشتراك.
- أصبح `setup-token` / `paste-token` في Anthropic متاحين مرة أخرى كمسار OpenClaw قديم/يدوي. استخدمهما مع توقع أن Anthropic أبلغت مستخدمي OpenClaw أن هذا المسار يتطلب **Extra Usage**.
