---
read_when:
    - تريد اختيار مزود نموذج
    - تحتاج إلى نظرة عامة سريعة على واجهات LLM الخلفية المدعومة
summary: مزودو النماذج (LLMs) الذين يدعمهم OpenClaw
title: دليل المزودين
x-i18n:
    generated_at: "2026-04-06T03:11:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7271157a6ab5418672baff62bfd299572fd010f75aad529267095c6e55903882
    source_path: providers/index.md
    workflow: 15
---

# مزودو النماذج

يمكن لـ OpenClaw استخدام العديد من مزودي LLM. اختر مزودًا، ثم قم بالمصادقة، ثم اضبط
النموذج الافتراضي بالشكل `provider/model`.

هل تبحث عن وثائق قنوات الدردشة (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/إلخ)؟ راجع [القنوات](/ar/channels).

## بدء سريع

1. قم بالمصادقة مع المزود (عادةً عبر `openclaw onboard`).
2. اضبط النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## وثائق المزودين

- [Alibaba Model Studio](/providers/alibaba)
- [Amazon Bedrock](/ar/providers/bedrock)
- [Anthropic (API + Claude CLI)](/ar/providers/anthropic)
- [BytePlus (International)](/ar/concepts/model-providers#byteplus-international)
- [Chutes](/ar/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
- [DeepSeek](/ar/providers/deepseek)
- [fal](/providers/fal)
- [Fireworks](/ar/providers/fireworks)
- [GitHub Copilot](/ar/providers/github-copilot)
- [GLM models](/ar/providers/glm)
- [Google (Gemini)](/ar/providers/google)
- [Groq (LPU inference)](/ar/providers/groq)
- [Hugging Face (Inference)](/ar/providers/huggingface)
- [Kilocode](/ar/providers/kilocode)
- [LiteLLM (unified gateway)](/ar/providers/litellm)
- [MiniMax](/ar/providers/minimax)
- [Mistral](/ar/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot)
- [NVIDIA](/ar/providers/nvidia)
- [Ollama (cloud + local models)](/ar/providers/ollama)
- [OpenAI (API + Codex)](/ar/providers/openai)
- [OpenCode](/ar/providers/opencode)
- [OpenCode Go](/ar/providers/opencode-go)
- [OpenRouter](/ar/providers/openrouter)
- [Perplexity (web search)](/ar/providers/perplexity-provider)
- [Qianfan](/ar/providers/qianfan)
- [Qwen Cloud](/ar/providers/qwen)
- [Runway](/providers/runway)
- [SGLang (local models)](/ar/providers/sglang)
- [StepFun](/ar/providers/stepfun)
- [Synthetic](/ar/providers/synthetic)
- [Together AI](/ar/providers/together)
- [Venice (Venice AI, privacy-focused)](/ar/providers/venice)
- [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
- [Vydra](/providers/vydra)
- [vLLM (local models)](/ar/providers/vllm)
- [Volcengine (Doubao)](/ar/providers/volcengine)
- [xAI](/ar/providers/xai)
- [Xiaomi](/ar/providers/xiaomi)
- [Z.AI](/ar/providers/zai)

## صفحات النظرة العامة المشتركة

- [إصدارات مضمّنة إضافية](/ar/providers/models#additional-bundled-provider-variants) - Anthropic Vertex وCopilot Proxy وGemini CLI OAuth
- [توليد الصور](/ar/tools/image-generation) - أداة `image_generate` المشتركة، واختيار المزود، وتجاوز الفشل
- [توليد الموسيقى](/tools/music-generation) - أداة `music_generate` المشتركة، واختيار المزود، وتجاوز الفشل
- [توليد الفيديو](/tools/video-generation) - أداة `video_generate` المشتركة، واختيار المزود، وتجاوز الفشل

## مزودو التفريغ

- [Deepgram (audio transcription)](/ar/providers/deepgram)

## أدوات المجتمع

- [Claude Max API Proxy](/ar/providers/claude-max-api-proxy) - وكيل مجتمع لبيانات اعتماد اشتراك Claude (تحقق من سياسة/شروط Anthropic قبل الاستخدام)

للاطلاع على فهرس المزودين الكامل (xAI وGroq وMistral وما إلى ذلك) والإعدادات المتقدمة،
راجع [مزودو النماذج](/ar/concepts/model-providers).
