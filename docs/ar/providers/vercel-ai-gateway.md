---
read_when:
    - تريد استخدام Vercel AI Gateway مع OpenClaw
    - تحتاج إلى متغير البيئة الخاص بمفتاح API أو خيار مصادقة CLI
summary: إعداد Vercel AI Gateway (المصادقة + اختيار النموذج)
title: Vercel AI Gateway
x-i18n:
    generated_at: "2026-04-05T12:54:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: f30768dc3db49708b25042d317906f7ad9a2c72b0fa03263bc04f5eefbf7a507
    source_path: providers/vercel-ai-gateway.md
    workflow: 15
---

# Vercel AI Gateway

توفّر [Vercel AI Gateway](https://vercel.com/ai-gateway) واجهة API موحدة للوصول إلى مئات النماذج عبر نقطة نهاية واحدة.

- المزوّد: `vercel-ai-gateway`
- المصادقة: `AI_GATEWAY_API_KEY`
- API: متوافقة مع Anthropic Messages
- يكتشف OpenClaw تلقائيًا فهرس `/v1/models` الخاص بـ Gateway، لذا فإن `/models vercel-ai-gateway`
  يتضمن مراجع النماذج الحالية مثل `vercel-ai-gateway/openai/gpt-5.4`.

## البدء السريع

1. اضبط مفتاح API (موصى به: خزّنه لصالح Gateway):

```bash
openclaw onboard --auth-choice ai-gateway-api-key
```

2. اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "vercel-ai-gateway/anthropic/claude-opus-4.6" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY"
```

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كخدمة daemon ‏(launchd/systemd)، فتأكد من أن `AI_GATEWAY_API_KEY`
متاح لهذه العملية (على سبيل المثال في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## الاختصار في معرّف النموذج

يقبل OpenClaw مراجع نماذج Claude المختصرة الخاصة بـ Vercel ويقوم بتوحيدها
في وقت التشغيل:

- `vercel-ai-gateway/claude-opus-4.6` -> `vercel-ai-gateway/anthropic/claude-opus-4.6`
- `vercel-ai-gateway/opus-4.6` -> `vercel-ai-gateway/anthropic/claude-opus-4-6`
