---
read_when:
    - تريد استخدام Together AI مع OpenClaw
    - تحتاج إلى متغير البيئة لمفتاح API أو خيار مصادقة CLI
summary: إعداد Together AI (المصادقة + اختيار النموذج)
title: Together AI
x-i18n:
    generated_at: "2026-04-06T03:11:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: b68fdc15bfcac8d59e3e0c06a39162abd48d9d41a9a64a0ac622cd8e3f80a595
    source_path: providers/together.md
    workflow: 15
---

# Together AI

يوفر [Together AI](https://together.ai) الوصول إلى أبرز النماذج مفتوحة المصدر، بما في ذلك Llama وDeepSeek وKimi وغيرها، من خلال API موحدة.

- الموفّر: `together`
- المصادقة: `TOGETHER_API_KEY`
- API: متوافق مع OpenAI
- عنوان URL الأساسي: `https://api.together.xyz/v1`

## البدء السريع

1. عيّن مفتاح API (يوصى بتخزينه من أجل Gateway):

```bash
openclaw onboard --auth-choice together-api-key
```

2. عيّن نموذجًا افتراضيًا:

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

سيؤدي هذا إلى تعيين `together/moonshotai/Kimi-K2.5` كنموذج افتراضي.

## ملاحظة البيئة

إذا كان Gateway يعمل كخدمة daemon ‏(`launchd/systemd`)، فتأكد من أن `TOGETHER_API_KEY`
متاح لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## الفهرس المضمن

يشحن OpenClaw حاليًا فهرس Together المضمن التالي:

| مرجع النموذج | الاسم | الإدخال | السياق | ملاحظات |
| ------------------------------------------------------------ | -------------------------------------- | ----------- | ---------- | -------------------------------- |
| `together/moonshotai/Kimi-K2.5`                              | Kimi K2.5                              | نص، صورة | 262,144    | النموذج الافتراضي؛ الاستدلال مفعّل |
| `together/zai-org/GLM-4.7`                                   | GLM 4.7 Fp8                            | نص | 202,752    | نموذج نص عام الغرض |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`           | Llama 3.3 70B Instruct Turbo           | نص | 131,072    | نموذج تعليمات سريع |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`         | Llama 4 Scout 17B 16E Instruct         | نص، صورة | 10,000,000 | متعدد الوسائط |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | Llama 4 Maverick 17B 128E Instruct FP8 | نص، صورة | 20,000,000 | متعدد الوسائط |
| `together/deepseek-ai/DeepSeek-V3.1`                         | DeepSeek V3.1                          | نص | 131,072    | نموذج نص عام |
| `together/deepseek-ai/DeepSeek-R1`                           | DeepSeek R1                            | نص | 131,072    | نموذج استدلال |
| `together/moonshotai/Kimi-K2-Instruct-0905`                  | Kimi K2-Instruct 0905                  | نص | 262,144    | نموذج نص Kimi ثانوي |

يضبط preset الخاص بـ onboarding النموذج `together/moonshotai/Kimi-K2.5` كنموذج افتراضي.

## إنشاء الفيديو

يسجل plugin المضمن `together` أيضًا إنشاء الفيديو عبر الأداة المشتركة
`video_generate`.

- نموذج الفيديو الافتراضي: `together/Wan-AI/Wan2.2-T2V-A14B`
- الأوضاع: نص إلى فيديو وتدفقات مرجعية لصورة واحدة
- يدعم `aspectRatio` و`resolution`

لاستخدام Together بوصفه موفّر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

راجع [إنشاء الفيديو](/tools/video-generation) للاطلاع على معلمات
الأداة المشتركة، واختيار الموفّر، وسلوك التحويل الاحتياطي.
