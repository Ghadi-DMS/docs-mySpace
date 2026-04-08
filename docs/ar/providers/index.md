---
read_when:
    - تريد اختيار موفر نموذج
    - تحتاج إلى نظرة عامة سريعة على الواجهات الخلفية المدعومة لـ LLM
summary: موفرو النماذج (LLMs) الذين يدعمهم OpenClaw
title: دليل الموفرين
x-i18n:
    generated_at: "2026-04-08T02:17:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: e7bee5528b7fc9a982b3d0eaa4930cb77f7bded19a47aec00572b6fcbd823a70
    source_path: providers/index.md
    workflow: 15
---

# موفرو النماذج

يمكن لـ OpenClaw استخدام العديد من موفري LLM. اختر موفرًا، ثم أكمل المصادقة، ثم اضبط
النموذج الافتراضي بصيغة `provider/model`.

هل تبحث عن وثائق قنوات الدردشة (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/إلخ)؟ راجع [القنوات](/ar/channels).

## بداية سريعة

1. أكمل المصادقة مع الموفر (عادةً عبر `openclaw onboard`).
2. اضبط النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## وثائق الموفرين

- [Alibaba Model Studio](/ar/providers/alibaba)
- [Amazon Bedrock](/ar/providers/bedrock)
- [Anthropic (API + Claude CLI)](/ar/providers/anthropic)
- [Arcee AI (نماذج Trinity)](/ar/providers/arcee)
- [BytePlus (الدولي)](/ar/concepts/model-providers#byteplus-international)
- [Chutes](/ar/providers/chutes)
- [ComfyUI](/ar/providers/comfy)
- [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
- [DeepSeek](/ar/providers/deepseek)
- [fal](/ar/providers/fal)
- [Fireworks](/ar/providers/fireworks)
- [GitHub Copilot](/ar/providers/github-copilot)
- [نماذج GLM](/ar/providers/glm)
- [Google (Gemini)](/ar/providers/google)
- [Groq (استدلال LPU)](/ar/providers/groq)
- [Hugging Face (Inference)](/ar/providers/huggingface)
- [inferrs (نماذج محلية)](/ar/providers/inferrs)
- [Kilocode](/ar/providers/kilocode)
- [LiteLLM (بوابة موحدة)](/ar/providers/litellm)
- [MiniMax](/ar/providers/minimax)
- [Mistral](/ar/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot)
- [NVIDIA](/ar/providers/nvidia)
- [Ollama (نماذج سحابية + محلية)](/ar/providers/ollama)
- [OpenAI (API + Codex)](/ar/providers/openai)
- [OpenCode](/ar/providers/opencode)
- [OpenCode Go](/ar/providers/opencode-go)
- [OpenRouter](/ar/providers/openrouter)
- [Perplexity (بحث الويب)](/ar/providers/perplexity-provider)
- [Qianfan](/ar/providers/qianfan)
- [Qwen Cloud](/ar/providers/qwen)
- [Runway](/ar/providers/runway)
- [SGLang (نماذج محلية)](/ar/providers/sglang)
- [StepFun](/ar/providers/stepfun)
- [Synthetic](/ar/providers/synthetic)
- [Together AI](/ar/providers/together)
- [Venice (Venice AI، مع التركيز على الخصوصية)](/ar/providers/venice)
- [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
- [Vydra](/ar/providers/vydra)
- [vLLM (نماذج محلية)](/ar/providers/vllm)
- [Volcengine (Doubao)](/ar/providers/volcengine)
- [xAI](/ar/providers/xai)
- [Xiaomi](/ar/providers/xiaomi)
- [Z.AI](/ar/providers/zai)

## صفحات النظرة العامة المشتركة

- [المتغيرات المضمنة الإضافية](/ar/providers/models#additional-bundled-provider-variants) - Anthropic Vertex وCopilot Proxy وGemini CLI OAuth
- [إنشاء الصور](/ar/tools/image-generation) - الأداة المشتركة `image_generate`، واختيار الموفر، وسلوك التبديل الاحتياطي
- [إنشاء الموسيقى](/ar/tools/music-generation) - الأداة المشتركة `music_generate`، واختيار الموفر، وسلوك التبديل الاحتياطي
- [إنشاء الفيديو](/ar/tools/video-generation) - الأداة المشتركة `video_generate`، واختيار الموفر، وسلوك التبديل الاحتياطي

## موفرو النسخ الصوتي

- [Deepgram (نسخ الصوت)](/ar/providers/deepgram)

## أدوات المجتمع

- [Claude Max API Proxy](/ar/providers/claude-max-api-proxy) - وكيل مجتمعي لبيانات اعتماد اشتراك Claude (تحقق من سياسة Anthropic/الشروط قبل الاستخدام)

للاطلاع على فهرس الموفرين الكامل (xAI وGroq وMistral وما إلى ذلك) والإعدادات المتقدمة،
راجع [موفرو النماذج](/ar/concepts/model-providers).
