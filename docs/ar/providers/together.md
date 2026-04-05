---
read_when:
    - تريد استخدام Together AI مع OpenClaw
    - تحتاج إلى متغير env الخاص بـ API key أو خيار مصادقة CLI
summary: إعداد Together AI ‏(المصادقة + اختيار النموذج)
title: Together AI
x-i18n:
    generated_at: "2026-04-05T12:54:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 22aacbaadf860ce8245bba921dcc5ede9da8fd6fa1bc3cc912551aecc1ba0d71
    source_path: providers/together.md
    workflow: 15
---

# Together AI

يوفر [Together AI](https://together.ai) الوصول إلى نماذج مفتوحة المصدر رائدة تشمل Llama وDeepSeek وKimi وغير ذلك عبر API موحدة.

- المزوّد: `together`
- المصادقة: `TOGETHER_API_KEY`
- API: متوافق مع OpenAI
- Base URL: `https://api.together.xyz/v1`

## بدء سريع

1. اضبط API key ‏(الموصى به: تخزينها من أجل Gateway):

```bash
openclaw onboard --auth-choice together-api-key
```

2. اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

سيؤدي هذا إلى ضبط `together/moonshotai/Kimi-K2.5` كنموذج افتراضي.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن `TOGETHER_API_KEY`
متاحة لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## الفهرس المضمّن

يشحن OpenClaw حاليًا فهرس Together المضمّن التالي:

| مرجع النموذج                                                   | الاسم                                   | الإدخال      | السياق     | ملاحظات                           |
| -------------------------------------------------------------- | --------------------------------------- | ------------ | ---------- | --------------------------------- |
| `together/moonshotai/Kimi-K2.5`                                | Kimi K2.5                               | text, image  | 262,144    | النموذج الافتراضي؛ مع تفعيل reasoning |
| `together/zai-org/GLM-4.7`                                     | GLM 4.7 Fp8                             | text         | 202,752    | نموذج نصي عام الأغراض             |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`             | Llama 3.3 70B Instruct Turbo            | text         | 131,072    | نموذج سريع للتعليمات              |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`           | Llama 4 Scout 17B 16E Instruct          | text, image  | 10,000,000 | متعدد الوسائط                     |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8`   | Llama 4 Maverick 17B 128E Instruct FP8  | text, image  | 20,000,000 | متعدد الوسائط                     |
| `together/deepseek-ai/DeepSeek-V3.1`                           | DeepSeek V3.1                           | text         | 131,072    | نموذج نص عام                      |
| `together/deepseek-ai/DeepSeek-R1`                             | DeepSeek R1                             | text         | 131,072    | نموذج reasoning                   |
| `together/moonshotai/Kimi-K2-Instruct-0905`                    | Kimi K2-Instruct 0905                   | text         | 262,144    | نموذج Kimi نصي ثانوي              |

يضبط إعداد onboarding المسبق `together/moonshotai/Kimi-K2.5` كنموذج افتراضي.
