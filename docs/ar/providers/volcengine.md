---
read_when:
    - تريد استخدام Volcano Engine أو نماذج Doubao مع OpenClaw
    - تحتاج إلى إعداد Volcengine API key
summary: إعداد Volcengine ‏(نماذج Doubao، ونقاط النهاية العامة + الخاصة بالبرمجة)
title: Volcengine ‏(Doubao)
x-i18n:
    generated_at: "2026-04-05T12:54:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 85d9e737e906cd705fb31479d6b78d92b68c9218795ea9667516c1571dcaaf3a
    source_path: providers/volcengine.md
    workflow: 15
---

# Volcengine ‏(Doubao)

يمنح مزود Volcengine الوصول إلى نماذج Doubao والنماذج التابعة لجهات خارجية
المستضافة على Volcano Engine، مع نقاط نهاية منفصلة لأحمال العمل
العامة وأحمال البرمجة.

- المزوّدون: `volcengine` ‏(عام) + `volcengine-plan` ‏(للبرمجة)
- المصادقة: `VOLCANO_ENGINE_API_KEY`
- API: متوافقة مع OpenAI

## بدء سريع

1. اضبط API key:

```bash
openclaw onboard --auth-choice volcengine-api-key
```

2. اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "volcengine-plan/ark-code-latest" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

## المزوّدون ونقاط النهاية

| المزوّد            | نقطة النهاية                              | حالة الاستخدام   |
| ------------------ | ----------------------------------------- | ---------------- |
| `volcengine`       | `ark.cn-beijing.volces.com/api/v3`        | النماذج العامة   |
| `volcengine-plan`  | `ark.cn-beijing.volces.com/api/coding/v3` | نماذج البرمجة    |

يُضبط كلا المزوّدين باستخدام API key واحدة. ويقوم الإعداد بتسجيلهما
تلقائيًا.

## النماذج المتاحة

المزوّد العام (`volcengine`):

| مرجع النموذج                               | الاسم                           | الإدخال      | السياق  |
| ----------------------------------------- | ------------------------------- | ------------ | ------- |
| `volcengine/doubao-seed-1-8-251228`       | Doubao Seed 1.8                 | text, image  | 256,000 |
| `volcengine/doubao-seed-code-preview-251028` | doubao-seed-code-preview-251028 | text, image  | 256,000 |
| `volcengine/kimi-k2-5-260127`             | Kimi K2.5                       | text, image  | 256,000 |
| `volcengine/glm-4-7-251222`               | GLM 4.7                         | text, image  | 200,000 |
| `volcengine/deepseek-v3-2-251201`         | DeepSeek V3.2                   | text, image  | 128,000 |

مزوّد البرمجة (`volcengine-plan`):

| مرجع النموذج                                    | الاسم                    | الإدخال | السياق  |
| ------------------------------------------------ | ------------------------ | ------- | ------- |
| `volcengine-plan/ark-code-latest`                | Ark Coding Plan          | text    | 256,000 |
| `volcengine-plan/doubao-seed-code`               | Doubao Seed Code         | text    | 256,000 |
| `volcengine-plan/glm-4.7`                        | GLM 4.7 Coding           | text    | 200,000 |
| `volcengine-plan/kimi-k2-thinking`               | Kimi K2 Thinking         | text    | 256,000 |
| `volcengine-plan/kimi-k2.5`                      | Kimi K2.5 Coding         | text    | 256,000 |
| `volcengine-plan/doubao-seed-code-preview-251028`| Doubao Seed Code Preview | text    | 256,000 |

يضبط `openclaw onboard --auth-choice volcengine-api-key` حاليًا
`volcengine-plan/ark-code-latest` كنموذج افتراضي، مع تسجيل
فهرس `volcengine` العام أيضًا.

أثناء اختيار النموذج في الإعداد الأولي/configure، يفضّل خيار مصادقة Volcengine
كلًا من الصفوف `volcengine/*` و`volcengine-plan/*`. وإذا لم تكن هذه النماذج
محملة بعد، يعود OpenClaw إلى الفهرس غير المصفّى بدل عرض
منتقٍ فارغ محدد بالمزوّد.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن
`VOLCANO_ENGINE_API_KEY` متاحة لتلك العملية (على سبيل المثال، في
`~/.openclaw/.env` أو عبر `env.shellEnv`).
