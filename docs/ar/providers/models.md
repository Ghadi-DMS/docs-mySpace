---
read_when:
    - تريد اختيار موفّر نموذج
    - تريد أمثلة إعداد سريعة لمصادقة LLM + اختيار النموذج
summary: موفّرات النماذج (LLMs) التي يدعمها OpenClaw
title: البدء السريع لموفّر النموذج
x-i18n:
    generated_at: "2026-04-08T02:18:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 59ee4c2f993fe0ae05fe34f52bc6f3e0fc9a76b10760f56b20ad251e25ee9f20
    source_path: providers/models.md
    workflow: 15
---

# موفّرات النماذج

يمكن لـ OpenClaw استخدام العديد من موفّرات LLM. اختر واحدًا، وقم بالمصادقة، ثم عيّن
النموذج الافتراضي بصيغة `provider/model`.

## البدء السريع (خطوتان)

1. قم بالمصادقة مع الموفّر (عادةً عبر `openclaw onboard`).
2. عيّن النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## الموفّرون المدعومون (مجموعة البداية)

- [Alibaba Model Studio](/ar/providers/alibaba)
- [Anthropic (API + Claude CLI)](/ar/providers/anthropic)
- [Amazon Bedrock](/ar/providers/bedrock)
- [BytePlus (International)](/ar/concepts/model-providers#byteplus-international)
- [Chutes](/ar/providers/chutes)
- [ComfyUI](/ar/providers/comfy)
- [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
- [fal](/ar/providers/fal)
- [Fireworks](/ar/providers/fireworks)
- [GLM models](/ar/providers/glm)
- [MiniMax](/ar/providers/minimax)
- [Mistral](/ar/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot)
- [OpenAI (API + Codex)](/ar/providers/openai)
- [OpenCode (Zen + Go)](/ar/providers/opencode)
- [OpenRouter](/ar/providers/openrouter)
- [Qianfan](/ar/providers/qianfan)
- [Qwen](/ar/providers/qwen)
- [Runway](/ar/providers/runway)
- [StepFun](/ar/providers/stepfun)
- [Synthetic](/ar/providers/synthetic)
- [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/ar/providers/venice)
- [xAI](/ar/providers/xai)
- [Z.AI](/ar/providers/zai)

## متغيرات موفّرات إضافية مضمّنة

- `anthropic-vertex` - دعم Anthropic ضمني على Google Vertex عند توفر بيانات اعتماد Vertex؛ لا يوجد خيار مصادقة منفصل أثناء الإعداد الأولي
- `copilot-proxy` - جسر Copilot Proxy محلي لـ VS Code؛ استخدم `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - تدفق OAuth غير رسمي لـ Gemini CLI؛ يتطلب تثبيت `gemini` محليًا (`brew install gemini-cli` أو `npm install -g @google/gemini-cli`)؛ النموذج الافتراضي هو `google-gemini-cli/gemini-3-flash-preview`؛ استخدم `openclaw onboard --auth-choice google-gemini-cli` أو `openclaw models auth login --provider google-gemini-cli --set-default`

للاطلاع على كتالوج الموفّرات الكامل (xAI وGroq وMistral وغيرها) والتكوين المتقدم،
راجع [موفّرات النماذج](/ar/concepts/model-providers).
