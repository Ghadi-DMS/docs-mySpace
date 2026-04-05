---
read_when:
    - تريد استخدام Hugging Face Inference مع OpenClaw
    - تحتاج إلى متغير env الخاص برمز HF أو خيار مصادقة CLI
summary: إعداد Hugging Face Inference ‏(المصادقة + اختيار النموذج)
title: Hugging Face ‏(Inference)
x-i18n:
    generated_at: "2026-04-05T12:53:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 692d2caffbaf991670260da393c67ae7e6349b9e1e3ed5cb9a514f8a77192e86
    source_path: providers/huggingface.md
    workflow: 15
---

# Hugging Face ‏(Inference)

توفّر [Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) واجهات chat completions متوافقة مع OpenAI عبر API موجهة واحدة. وهذا يمنحك الوصول إلى نماذج كثيرة (DeepSeek وLlama وغير ذلك) باستخدام token واحدة. يستخدم OpenClaw **نقطة النهاية المتوافقة مع OpenAI** ‏(chat completions فقط)؛ أما text-to-image أو embeddings أو speech فاستخدم [HF inference clients](https://huggingface.co/docs/api-inference/quicktour) مباشرة.

- المزوّد: `huggingface`
- المصادقة: `HUGGINGFACE_HUB_TOKEN` أو `HF_TOKEN` ‏(fine-grained token مع إذن **Make calls to Inference Providers**)
- API: متوافق مع OpenAI ‏(`https://router.huggingface.co/v1`)
- الفوترة: token واحدة من HF؛ وتتبع [الأسعار](https://huggingface.co/docs/inference-providers/pricing) أسعار المزوّد مع طبقة مجانية.

## بدء سريع

1. أنشئ fine-grained token في [Hugging Face → Settings → Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) مع إذن **Make calls to Inference Providers**.
2. شغّل الإعداد الأولي واختر **Hugging Face** في قائمة المزوّد، ثم أدخل API key عندما يُطلب منك ذلك:

```bash
openclaw onboard --auth-choice huggingface-api-key
```

3. من قائمة **Default Hugging Face model** المنسدلة، اختر النموذج الذي تريده (تُحمَّل القائمة من Inference API عندما يكون لديك token صالحة؛ وإلا تُعرض قائمة مضمّنة). ويُحفظ اختيارك كنموذج افتراضي.
4. يمكنك أيضًا ضبط النموذج الافتراضي أو تغييره لاحقًا في الإعداد:

```json5
{
  agents: {
    defaults: {
      model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

سيضبط هذا `huggingface/deepseek-ai/DeepSeek-R1` كنموذج افتراضي.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن `HUGGINGFACE_HUB_TOKEN` أو `HF_TOKEN`
متاحان لتلك العملية (مثلًا في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## اكتشاف النماذج وقائمة الإعداد الأولي المنسدلة

يكتشف OpenClaw النماذج عبر استدعاء **Inference endpoint مباشرة**:

```bash
GET https://router.huggingface.co/v1/models
```

(اختياري: أرسل `Authorization: Bearer $HUGGINGFACE_HUB_TOKEN` أو `$HF_TOKEN` للحصول على القائمة الكاملة؛ فبعض نقاط النهاية تعيد مجموعة فرعية من دون مصادقة.) تكون الاستجابة بنمط OpenAI على الشكل `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`.

عندما تهيّئ Hugging Face API key ‏(عبر الإعداد الأولي، أو `HUGGINGFACE_HUB_TOKEN`، أو `HF_TOKEN`)، يستخدم OpenClaw هذا الطلب GET لاكتشاف نماذج chat-completion المتاحة. أثناء **الإعداد التفاعلي**، بعد إدخال token ترى قائمة منسدلة باسم **Default Hugging Face model** مملوءة من تلك القائمة (أو من الفهرس المضمّن إذا فشل الطلب). وأثناء وقت التشغيل (مثل بدء Gateway)، وعندما يكون المفتاح موجودًا، يستدعي OpenClaw مرة أخرى **GET** ‏`https://router.huggingface.co/v1/models` لتحديث الفهرس. وتُدمج القائمة مع فهرس مضمّن (لبيانات وصفية مثل نافذة السياق والتكلفة). وإذا فشل الطلب أو لم يُضبط مفتاح، فلن يُستخدم إلا الفهرس المضمّن.

## أسماء النماذج والخيارات القابلة للتحرير

- **الاسم القادم من API:** يُشتق اسم عرض النموذج من **GET /v1/models** عندما تعيد API أحد الحقول `name` أو `title` أو `display_name`؛ وإلا يُشتق من معرّف النموذج نفسه (مثل `deepseek-ai/DeepSeek-R1` → “DeepSeek R1”).
- **تجاوز اسم العرض:** يمكنك ضبط تسمية مخصصة لكل نموذج بحيث تظهر بالطريقة التي تريدها في CLI وUI:

```json5
{
  agents: {
    defaults: {
      models: {
        "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
        "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
      },
    },
  },
}
```

- **لواحق السياسة:** تتعامل وثائق ومساعدات Hugging Face المضمّنة في OpenClaw حاليًا مع هاتين اللاحقتين على أنهما متغيرا السياسة المضمّنان:
  - **`:fastest`** — أعلى معدل throughput.
  - **`:cheapest`** — أقل تكلفة لكل token خرج.

  يمكنك إضافة هاتين الصيغتين كإدخالات منفصلة في `models.providers.huggingface.models` أو ضبط `model.primary` مع اللاحقة. كما يمكنك أيضًا ضبط ترتيب المزوّد الافتراضي في [إعدادات Inference Provider](https://hf.co/settings/inference-providers) ‏(من دون لاحقة = استخدام ذلك الترتيب).

- **دمج الإعداد:** تُحتفظ بالإدخالات الموجودة في `models.providers.huggingface.models` ‏(مثلًا في `models.json`) عند دمج الإعداد. لذا فإن أي `name` أو `alias` أو خيارات نموذج مخصصة تضبطها هناك ستظل محفوظة.

## معرّفات النماذج وأمثلة الإعداد

تستخدم مراجع النماذج الصيغة `huggingface/<org>/<model>` ‏(معرّفات بأسلوب Hub). وتأتي القائمة أدناه من **GET** ‏`https://router.huggingface.co/v1/models`؛ وقد يتضمن فهرسك المزيد.

**أمثلة على المعرّفات (من نقطة نهاية inference):**

| النموذج                 | المرجع (أضف البادئة `huggingface/`)   |
| ---------------------- | ------------------------------------- |
| DeepSeek R1            | `deepseek-ai/DeepSeek-R1`             |
| DeepSeek V3.2          | `deepseek-ai/DeepSeek-V3.2`           |
| Qwen3 8B               | `Qwen/Qwen3-8B`                       |
| Qwen2.5 7B Instruct    | `Qwen/Qwen2.5-7B-Instruct`            |
| Qwen3 32B              | `Qwen/Qwen3-32B`                      |
| Llama 3.3 70B Instruct | `meta-llama/Llama-3.3-70B-Instruct`   |
| Llama 3.1 8B Instruct  | `meta-llama/Llama-3.1-8B-Instruct`    |
| GPT-OSS 120B           | `openai/gpt-oss-120b`                 |
| GLM 4.7                | `zai-org/GLM-4.7`                     |
| Kimi K2.5              | `moonshotai/Kimi-K2.5`                |

يمكنك إلحاق `:fastest` أو `:cheapest` بمعرّف النموذج. واضبط ترتيبك الافتراضي في [إعدادات Inference Provider](https://hf.co/settings/inference-providers)؛ وراجع [Inference Providers](https://huggingface.co/docs/inference-providers) و**GET** ‏`https://router.huggingface.co/v1/models` للحصول على القائمة الكاملة.

### أمثلة إعداد كاملة

**DeepSeek R1 أساسي مع fallback إلى Qwen:**

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "huggingface/deepseek-ai/DeepSeek-R1",
        fallbacks: ["huggingface/Qwen/Qwen3-8B"],
      },
      models: {
        "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
        "huggingface/Qwen/Qwen3-8B": { alias: "Qwen3 8B" },
      },
    },
  },
}
```

**Qwen كافتراضي، مع متغيرَي :cheapest و:fastest:**

```json5
{
  agents: {
    defaults: {
      model: { primary: "huggingface/Qwen/Qwen3-8B" },
      models: {
        "huggingface/Qwen/Qwen3-8B": { alias: "Qwen3 8B" },
        "huggingface/Qwen/Qwen3-8B:cheapest": { alias: "Qwen3 8B (cheapest)" },
        "huggingface/Qwen/Qwen3-8B:fastest": { alias: "Qwen3 8B (fastest)" },
      },
    },
  },
}
```

**DeepSeek + Llama + GPT-OSS مع أسماء بديلة:**

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "huggingface/deepseek-ai/DeepSeek-V3.2",
        fallbacks: [
          "huggingface/meta-llama/Llama-3.3-70B-Instruct",
          "huggingface/openai/gpt-oss-120b",
        ],
      },
      models: {
        "huggingface/deepseek-ai/DeepSeek-V3.2": { alias: "DeepSeek V3.2" },
        "huggingface/meta-llama/Llama-3.3-70B-Instruct": { alias: "Llama 3.3 70B" },
        "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
      },
    },
  },
}
```

**عدة نماذج من Qwen وDeepSeek مع لواحق السياسة:**

```json5
{
  agents: {
    defaults: {
      model: { primary: "huggingface/Qwen/Qwen2.5-7B-Instruct:cheapest" },
      models: {
        "huggingface/Qwen/Qwen2.5-7B-Instruct": { alias: "Qwen2.5 7B" },
        "huggingface/Qwen/Qwen2.5-7B-Instruct:cheapest": { alias: "Qwen2.5 7B (cheap)" },
        "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fast)" },
        "huggingface/meta-llama/Llama-3.1-8B-Instruct": { alias: "Llama 3.1 8B" },
      },
    },
  },
}
```
