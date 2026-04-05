---
read_when:
    - تريد نماذج StepFun في OpenClaw
    - تحتاج إلى إرشادات إعداد StepFun
summary: استخدم نماذج StepFun مع OpenClaw
title: StepFun
x-i18n:
    generated_at: "2026-04-05T12:54:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3154852556577b4cfb387a2de281559f2b173c774bfbcaea996abe5379ae684a
    source_path: providers/stepfun.md
    workflow: 15
---

# StepFun

يتضمن OpenClaw plugin مضمنة لمزوّد StepFun مع معرّفي مزوّدين:

- `stepfun` لنقطة النهاية القياسية
- `stepfun-plan` لنقطة النهاية Step Plan

تختلف الفهارس المضمنة الحالية بحسب السطح:

- القياسي: `step-3.5-flash`
- Step Plan: ‏`step-3.5-flash`، و`step-3.5-flash-2603`

## نظرة عامة على المنطقة ونقطة النهاية

- نقطة النهاية القياسية في الصين: `https://api.stepfun.com/v1`
- نقطة النهاية القياسية العالمية: `https://api.stepfun.ai/v1`
- نقطة النهاية Step Plan في الصين: `https://api.stepfun.com/step_plan/v1`
- نقطة النهاية Step Plan العالمية: `https://api.stepfun.ai/step_plan/v1`
- متغير البيئة للمصادقة: `STEPFUN_API_KEY`

استخدم مفتاح الصين مع نقاط النهاية `.com` واستخدم المفتاح العالمي مع نقاط النهاية
`.ai`.

## إعداد CLI

الإعداد التفاعلي:

```bash
openclaw onboard
```

اختر أحد خيارات المصادقة التالية:

- `stepfun-standard-api-key-cn`
- `stepfun-standard-api-key-intl`
- `stepfun-plan-api-key-cn`
- `stepfun-plan-api-key-intl`

أمثلة غير تفاعلية:

```bash
openclaw onboard --auth-choice stepfun-standard-api-key-intl --stepfun-api-key "$STEPFUN_API_KEY"
openclaw onboard --auth-choice stepfun-plan-api-key-intl --stepfun-api-key "$STEPFUN_API_KEY"
```

## مراجع النماذج

- النموذج الافتراضي القياسي: `stepfun/step-3.5-flash`
- النموذج الافتراضي لـ Step Plan: ‏`stepfun-plan/step-3.5-flash`
- النموذج البديل لـ Step Plan: ‏`stepfun-plan/step-3.5-flash-2603`

## الفهارس المضمنة

القياسي (`stepfun`):

| مرجع النموذج                | السياق | الحد الأقصى للإخراج | ملاحظات                  |
| ------------------------ | ------- | ---------- | ---------------------- |
| `stepfun/step-3.5-flash` | 262,144 | 65,536     | النموذج القياسي الافتراضي |

Step Plan (`stepfun-plan`):

| مرجع النموذج                          | السياق | الحد الأقصى للإخراج | ملاحظات                      |
| ---------------------------------- | ------- | ---------- | -------------------------- |
| `stepfun-plan/step-3.5-flash`      | 262,144 | 65,536     | نموذج Step Plan الافتراضي    |
| `stepfun-plan/step-3.5-flash-2603` | 262,144 | 65,536     | نموذج Step Plan إضافي |

## مقتطفات التكوين

المزوّد القياسي:

```json5
{
  env: { STEPFUN_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "stepfun/step-3.5-flash" } } },
  models: {
    mode: "merge",
    providers: {
      stepfun: {
        baseUrl: "https://api.stepfun.ai/v1",
        api: "openai-completions",
        apiKey: "${STEPFUN_API_KEY}",
        models: [
          {
            id: "step-3.5-flash",
            name: "Step 3.5 Flash",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

مزوّد Step Plan:

```json5
{
  env: { STEPFUN_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "stepfun-plan/step-3.5-flash" } } },
  models: {
    mode: "merge",
    providers: {
      "stepfun-plan": {
        baseUrl: "https://api.stepfun.ai/step_plan/v1",
        api: "openai-completions",
        apiKey: "${STEPFUN_API_KEY}",
        models: [
          {
            id: "step-3.5-flash",
            name: "Step 3.5 Flash",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
          {
            id: "step-3.5-flash-2603",
            name: "Step 3.5 Flash 2603",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## ملاحظات

- المزوّد مضمن مع OpenClaw، لذلك لا توجد خطوة منفصلة لتثبيت plugin.
- يتم حاليًا كشف `step-3.5-flash-2603` على `stepfun-plan` فقط.
- يكتب تدفق مصادقة واحد ملفات تعريف مطابقة للمنطقة لكل من `stepfun` و`stepfun-plan`، بحيث يمكن اكتشاف السطحين معًا.
- استخدم `openclaw models list` و`openclaw models set <provider/model>` لفحص النماذج أو تبديلها.
- للحصول على النظرة العامة الأوسع حول المزوّدين، راجع [موفرو النماذج](/concepts/model-providers).
