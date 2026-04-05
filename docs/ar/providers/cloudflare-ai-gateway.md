---
read_when:
    - تريد استخدام Cloudflare AI Gateway مع OpenClaw
    - تحتاج إلى معرّف الحساب أو معرّف gateway أو متغير env الخاص بمفتاح API
summary: إعداد Cloudflare AI Gateway ‏(المصادقة + اختيار النموذج)
title: Cloudflare AI Gateway
x-i18n:
    generated_at: "2026-04-05T12:52:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: db77652c37652ca20f7c50f32382dbaeaeb50ea5bdeaf1d4fd17dc394e58950c
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 15
---

# Cloudflare AI Gateway

تعمل Cloudflare AI Gateway أمام واجهات API الخاصة بالموفّرين وتتيح لك إضافة التحليلات، والتخزين المؤقت، وعناصر التحكم. وبالنسبة إلى Anthropic، يستخدم OpenClaw واجهة Anthropic Messages API عبر نقطة نهاية Gateway الخاصة بك.

- الموفّر: `cloudflare-ai-gateway`
- Base URL: `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`
- النموذج الافتراضي: `cloudflare-ai-gateway/claude-sonnet-4-5`
- مفتاح API: ‏`CLOUDFLARE_AI_GATEWAY_API_KEY` ‏(مفتاح API الخاص بالموفّر لطلبات المرور عبر Gateway)

بالنسبة إلى نماذج Anthropic، استخدم مفتاح API الخاص بك من Anthropic.

## بدء سريع

1. اضبط مفتاح API الخاص بالموفّر وتفاصيل Gateway:

```bash
openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
```

2. اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-5" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## Gateways المصادق عليها

إذا فعّلت مصادقة Gateway في Cloudflare، فأضف الترويسة `cf-aig-authorization` ‏(وهذا بالإضافة إلى مفتاح API الخاص بالموفّر).

```json5
{
  models: {
    providers: {
      "cloudflare-ai-gateway": {
        headers: {
          "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
        },
      },
    },
  },
}
```

## ملاحظة حول البيئة

إذا كانت Gateway تعمل بوصفها daemon ‏(`launchd/systemd`)، فتأكد من أن `CLOUDFLARE_AI_GATEWAY_API_KEY` متاح لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر `env.shellEnv`).
