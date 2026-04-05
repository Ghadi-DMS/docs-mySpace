---
read_when:
    - تريد توجيه OpenClaw عبر وكيل LiteLLM
    - تحتاج إلى تتبع التكلفة أو التسجيل أو توجيه النماذج عبر LiteLLM
summary: تشغيل OpenClaw عبر LiteLLM Proxy للوصول الموحد إلى النماذج وتتبع التكلفة
title: LiteLLM
x-i18n:
    generated_at: "2026-04-05T12:53:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e8ca73458186285bc06967b397b8a008791dc58eea1159d6c358e1a794982d1
    source_path: providers/litellm.md
    workflow: 15
---

# LiteLLM

يُعد [LiteLLM](https://litellm.ai) بوابة LLM مفتوحة المصدر توفّر واجهة API موحدة لأكثر من 100 موفّر نماذج. مرّر OpenClaw عبر LiteLLM للحصول على تتبع مركزي للتكلفة، والتسجيل، والمرونة في تبديل الواجهات الخلفية من دون تغيير إعداد OpenClaw.

## لماذا تستخدم LiteLLM مع OpenClaw؟

- **تتبع التكلفة** — اعرف بالضبط ما الذي ينفقه OpenClaw عبر جميع النماذج
- **توجيه النماذج** — بدّل بين Claude وGPT-4 وGemini وBedrock من دون تغييرات في الإعداد
- **المفاتيح الافتراضية** — أنشئ مفاتيح مع حدود إنفاق لـ OpenClaw
- **التسجيل** — سجلات كاملة للطلب/الاستجابة لأغراض التصحيح
- **المسارات الاحتياطية** — تراجع احتياطي تلقائي إذا كان موفّرك الأساسي متوقفًا

## بدء سريع

### عبر onboarding

```bash
openclaw onboard --auth-choice litellm-api-key
```

### الإعداد اليدوي

1. ابدأ LiteLLM Proxy:

```bash
pip install 'litellm[proxy]'
litellm --model claude-opus-4-6
```

2. وجّه OpenClaw إلى LiteLLM:

```bash
export LITELLM_API_KEY="your-litellm-key"

openclaw
```

هذا كل شيء. أصبح OpenClaw الآن يوجّه الطلبات عبر LiteLLM.

## الإعداد

### متغيرات البيئة

```bash
export LITELLM_API_KEY="sk-litellm-key"
```

### ملف الإعداد

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

## المفاتيح الافتراضية

أنشئ مفتاحًا مخصصًا لـ OpenClaw مع حدود للإنفاق:

```bash
curl -X POST "http://localhost:4000/key/generate" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "key_alias": "openclaw",
    "max_budget": 50.00,
    "budget_duration": "monthly"
  }'
```

استخدم المفتاح المولّد بوصفه `LITELLM_API_KEY`.

## توجيه النماذج

يمكن لـ LiteLLM توجيه طلبات النماذج إلى واجهات خلفية مختلفة. اضبط ذلك في `config.yaml` الخاص بـ LiteLLM:

```yaml
model_list:
  - model_name: claude-opus-4-6
    litellm_params:
      model: claude-opus-4-6
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: gpt-4o
    litellm_params:
      model: gpt-4o
      api_key: os.environ/OPENAI_API_KEY
```

يواصل OpenClaw طلب `claude-opus-4-6` — ويتولى LiteLLM عملية التوجيه.

## عرض الاستخدام

تحقق من لوحة تحكم LiteLLM أو من واجهة API:

```bash
# معلومات المفتاح
curl "http://localhost:4000/key/info" \
  -H "Authorization: Bearer sk-litellm-key"

# سجلات الإنفاق
curl "http://localhost:4000/spend/logs" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

## ملاحظات

- يعمل LiteLLM افتراضيًا على `http://localhost:4000`
- يتصل OpenClaw عبر نقطة النهاية المتوافقة مع OpenAI ذات نمط proxy في LiteLLM ‏`/v1`
- لا تنطبق صياغة الطلبات الأصلية الخاصة بـ OpenAI فقط عند المرور عبر LiteLLM:
  لا `service_tier`، ولا `store` الخاصة بـ Responses، ولا تلميحات prompt-cache، ولا
  تشكيل الحمولات الخاصة بتوافق reasoning في OpenAI
- لا يتم حقن ترويسات الإسناد المخفية الخاصة بـ OpenClaw ‏(`originator` و`version` و`User-Agent`)
  على base URLs المخصصة لـ LiteLLM

## راجع أيضًا

- [وثائق LiteLLM](https://docs.litellm.ai)
- [موفرو النماذج](/concepts/model-providers)
