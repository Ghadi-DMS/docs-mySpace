---
read_when:
    - تريد استخدام نماذج Xiaomi MiMo في OpenClaw
    - تحتاج إلى إعداد `XIAOMI_API_KEY`
summary: استخدام نماذج Xiaomi MiMo مع OpenClaw
title: Xiaomi MiMo
x-i18n:
    generated_at: "2026-04-05T12:54:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: a2533fa99b29070e26e0e1fbde924e1291c89b1fbc2537451bcc0eb677ea6949
    source_path: providers/xiaomi.md
    workflow: 15
---

# Xiaomi MiMo

Xiaomi MiMo هي منصة API الخاصة بنماذج **MiMo**. يستخدم OpenClaw
نقطة النهاية المتوافقة مع OpenAI الخاصة بـ Xiaomi مع مصادقة مفتاح API. أنشئ مفتاح API الخاص بك من
[وحدة تحكم Xiaomi MiMo](https://platform.xiaomimimo.com/#/console/api-keys)، ثم هيّئ
الموفّر المدمج `xiaomi` باستخدام هذا المفتاح.

## الفهرس المدمج

- Base URL: ‏`https://api.xiaomimimo.com/v1`
- API: ‏`openai-completions`
- المصادقة: ‏`Bearer $XIAOMI_API_KEY`

| مرجع النموذج | الإدخال | السياق | الحد الأقصى للإخراج | ملاحظات |
| ---------------------- | ----------- | --------- | ---------- | ---------------------------- |
| `xiaomi/mimo-v2-flash` | نص | 262,144 | 8,192 | النموذج الافتراضي |
| `xiaomi/mimo-v2-pro` | نص | 1,048,576 | 32,000 | مفعّل للاستدلال |
| `xiaomi/mimo-v2-omni` | نص، صورة | 262,144 | 32,000 | متعدد الوسائط ومفعّل للاستدلال |

## الإعداد عبر CLI

```bash
openclaw onboard --auth-choice xiaomi-api-key
# أو بشكل غير تفاعلي
openclaw onboard --auth-choice xiaomi-api-key --xiaomi-api-key "$XIAOMI_API_KEY"
```

## مقتطف إعداد

```json5
{
  env: { XIAOMI_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "xiaomi/mimo-v2-flash" } } },
  models: {
    mode: "merge",
    providers: {
      xiaomi: {
        baseUrl: "https://api.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_API_KEY",
        models: [
          {
            id: "mimo-v2-flash",
            name: "Xiaomi MiMo V2 Flash",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 8192,
          },
          {
            id: "mimo-v2-pro",
            name: "Xiaomi MiMo V2 Pro",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 1048576,
            maxTokens: 32000,
          },
          {
            id: "mimo-v2-omni",
            name: "Xiaomi MiMo V2 Omni",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

## ملاحظات

- مرجع النموذج الافتراضي: `xiaomi/mimo-v2-flash`.
- النماذج المدمجة الإضافية: `xiaomi/mimo-v2-pro` و`xiaomi/mimo-v2-omni`.
- يُحقن الموفّر تلقائيًا عندما يكون `XIAOMI_API_KEY` مضبوطًا (أو يوجد ملف تعريف مصادقة).
- راجع [/concepts/model-providers](/concepts/model-providers) للاطلاع على قواعد الموفّرين.
