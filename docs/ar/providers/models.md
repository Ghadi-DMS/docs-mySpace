---
read_when:
    - أنت تريد اختيار موفّر نموذج
    - أنت تريد أمثلة إعداد سريعة لمصادقة LLM + اختيار النموذج
summary: موفرو النماذج (LLMs) المدعومون في OpenClaw
title: البدء السريع لموفري النماذج
x-i18n:
    generated_at: "2026-04-05T12:53:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83e372193b476c7cee6eb9f5c443b03563d863043f47c633ac0096bca642cc6f
    source_path: providers/models.md
    workflow: 15
---

# موفرو النماذج

يمكن لـ OpenClaw استخدام العديد من موفري LLM. اختر واحدًا، وصادِق معه، ثم اضبط
النموذج الافتراضي بصيغة `provider/model`.

## البدء السريع (خطوتان)

1. صادِق مع الموفّر (عادةً عبر `openclaw onboard`).
2. اضبط النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## الموفّرون المدعومون (مجموعة البداية)

- [Anthropic ‏(API + Claude CLI)](/providers/anthropic)
- [Amazon Bedrock](/providers/bedrock)
- [BytePlus ‏(دولي)](/concepts/model-providers#byteplus-international)
- [Chutes](/providers/chutes)
- [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
- [Fireworks](/providers/fireworks)
- [نماذج GLM](/providers/glm)
- [MiniMax](/providers/minimax)
- [Mistral](/providers/mistral)
- [Moonshot AI ‏(Kimi + Kimi Coding)](/providers/moonshot)
- [OpenAI ‏(API + Codex)](/providers/openai)
- [OpenCode ‏(Zen + Go)](/providers/opencode)
- [OpenRouter](/providers/openrouter)
- [Qianfan](/providers/qianfan)
- [Qwen](/providers/qwen)
- [StepFun](/providers/stepfun)
- [Synthetic](/providers/synthetic)
- [Vercel AI Gateway](/providers/vercel-ai-gateway)
- [Venice ‏(Venice AI)](/providers/venice)
- [xAI](/providers/xai)
- [Z.AI](/providers/zai)

## متغيرات الموفّرين المضمّنة الإضافية

- `anthropic-vertex` - دعم Anthropic على Google Vertex ضمنيًا عندما تكون بيانات اعتماد Vertex متاحة؛ ولا يوجد خيار مصادقة منفصل في الإعداد الأولي
- `copilot-proxy` - جسر Copilot Proxy محلي لـ VS Code؛ استخدم `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - تدفق OAuth غير رسمي لـ Gemini CLI؛ يتطلب تثبيت `gemini` محليًا (`brew install gemini-cli` أو `npm install -g @google/gemini-cli`)؛ النموذج الافتراضي هو `google-gemini-cli/gemini-3.1-pro-preview`؛ استخدم `openclaw onboard --auth-choice google-gemini-cli` أو `openclaw models auth login --provider google-gemini-cli --set-default`

للحصول على كتالوج الموفّرين الكامل (xAI وGroq وMistral وغيرهم) والإعداد المتقدم،
راجع [موفرو النماذج](/concepts/model-providers).
