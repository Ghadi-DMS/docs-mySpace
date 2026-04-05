---
read_when:
    - تريد اختيار مزوّد نماذج
    - تحتاج إلى نظرة عامة سريعة على الواجهات الخلفية المدعومة لـ LLM
summary: موفرو النماذج (LLMs) الذين يدعمهم OpenClaw
title: دليل المزوّدين
x-i18n:
    generated_at: "2026-04-05T12:53:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 690d17c14576d454ea3cd3dcbc704470da10a2a34adfe681dab7048438f2e193
    source_path: providers/index.md
    workflow: 15
---

# موفرو النماذج

يمكن لـ OpenClaw استخدام العديد من موفري LLM. اختر مزودًا، وقم بالمصادقة، ثم اضبط
النموذج الافتراضي بصيغة `provider/model`.

هل تبحث عن مستندات قنوات الدردشة (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/إلخ)؟ راجع [القنوات](/channels).

## البدء السريع

1. صادق مع المزوّد (عادةً عبر `openclaw onboard`).
2. اضبط النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## مستندات المزوّدين

- [Amazon Bedrock](/providers/bedrock)
- [Anthropic (API + Claude CLI)](/providers/anthropic)
- [BytePlus (دولي)](/concepts/model-providers#byteplus-international)
- [Chutes](/providers/chutes)
- [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
- [DeepSeek](/providers/deepseek)
- [Fireworks](/providers/fireworks)
- [GitHub Copilot](/providers/github-copilot)
- [نماذج GLM](/providers/glm)
- [Google (Gemini)](/providers/google)
- [Groq (استدلال LPU)](/providers/groq)
- [Hugging Face (Inference)](/providers/huggingface)
- [Kilocode](/providers/kilocode)
- [LiteLLM (بوابة موحدة)](/providers/litellm)
- [MiniMax](/providers/minimax)
- [Mistral](/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot)
- [NVIDIA](/providers/nvidia)
- [Ollama (نماذج سحابية + محلية)](/providers/ollama)
- [OpenAI (API + Codex)](/providers/openai)
- [OpenCode](/providers/opencode)
- [OpenCode Go](/providers/opencode-go)
- [OpenRouter](/providers/openrouter)
- [Perplexity (بحث الويب)](/providers/perplexity-provider)
- [Qianfan](/providers/qianfan)
- [Qwen Cloud](/providers/qwen)
- [Qwen / Model Studio (تفاصيل نقطة النهاية؛ `qwen-*` هو الاسم القياسي، و`modelstudio-*` قديم)](/providers/qwen_modelstudio)
- [SGLang (نماذج محلية)](/providers/sglang)
- [StepFun](/providers/stepfun)
- [Synthetic](/providers/synthetic)
- [Together AI](/providers/together)
- [Venice (Venice AI، يركز على الخصوصية)](/providers/venice)
- [Vercel AI Gateway](/providers/vercel-ai-gateway)
- [vLLM (نماذج محلية)](/providers/vllm)
- [Volcengine (Doubao)](/providers/volcengine)
- [xAI](/providers/xai)
- [Xiaomi](/providers/xiaomi)
- [Z.AI](/providers/zai)

## صفحات النظرة العامة المشتركة

- [متغيرات مضمنة إضافية](/providers/models#additional-bundled-provider-variants) - Anthropic Vertex، وCopilot Proxy، وGemini CLI OAuth

## موفرو النسخ الصوتي

- [Deepgram (نسخ صوتي)](/providers/deepgram)

## أدوات المجتمع

- [Claude Max API Proxy](/providers/claude-max-api-proxy) - proxy مجتمعي لبيانات اعتماد اشتراك Claude (تحقق من سياسة/شروط Anthropic قبل الاستخدام)

للحصول على فهرس المزوّدين الكامل (xAI وGroq وMistral وغيرهم) والتكوين المتقدم،
راجع [موفرو النماذج](/concepts/model-providers).
