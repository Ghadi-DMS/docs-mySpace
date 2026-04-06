---
read_when:
    - تريد اختيار مزوّد نموذج
    - تريد أمثلة إعداد سريعة لمصادقة LLM + اختيار النموذج
summary: مزوّدو النماذج (LLMs) الذين يدعمهم OpenClaw
title: البدء السريع لمزوّد النموذج
x-i18n:
    generated_at: "2026-04-06T03:11:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: c0314fb1c754171e5fc252d30f7ba9bb6acdbb978d97e9249264d90351bac2e7
    source_path: providers/models.md
    workflow: 15
---

# مزوّدو النماذج

يمكن لـ OpenClaw استخدام العديد من مزوّدي LLM. اختر واحدًا، ثم صادِق عليه، ثم عيّن
النموذج الافتراضي بصيغة `provider/model`.

## بداية سريعة (خطوتان)

1. صادِق مع المزوّد (عادةً عبر `openclaw onboard`).
2. عيّن النموذج الافتراضي:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## المزوّدون المدعومون (المجموعة الأساسية)

- [Alibaba Model Studio](/providers/alibaba)
- [Anthropic (API + Claude CLI)](/ar/providers/anthropic)
- [Amazon Bedrock](/ar/providers/bedrock)
- [BytePlus (الدولي)](/ar/concepts/model-providers#byteplus-international)
- [Chutes](/ar/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
- [fal](/providers/fal)
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
- [Runway](/providers/runway)
- [StepFun](/ar/providers/stepfun)
- [Synthetic](/ar/providers/synthetic)
- [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/ar/providers/venice)
- [xAI](/ar/providers/xai)
- [Z.AI](/ar/providers/zai)

## متغيرات مزوّدين إضافية مجمّعة

- `anthropic-vertex` - دعم Anthropic الضمني على Google Vertex عندما تتوفر بيانات اعتماد Vertex؛ لا يوجد خيار مصادقة منفصل في onboarding
- `copilot-proxy` - جسر VS Code Copilot Proxy محلي؛ استخدم `openclaw onboard --auth-choice copilot-proxy`

للاطلاع على فهرس المزوّدين الكامل (xAI وGroq وMistral وغيرها) والإعدادات المتقدمة،
راجع [مزوّدو النماذج](/ar/concepts/model-providers).
