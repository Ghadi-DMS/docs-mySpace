---
x-i18n:
    generated_at: "2026-04-05T12:54:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1066a1d0acebe4ae3500d18c21f7de07f43b9766daf3d13b098936734e9e7a2b
    source_path: providers/qwen_modelstudio.md
    workflow: 15
---

title: "Qwen / Model Studio"
summary: "تفاصيل نقطة النهاية لمزوّد qwen المضمّن وسطح التوافق القديم الخاص به مع modelstudio"
read_when:

- تريد تفاصيل على مستوى نقطة النهاية لـ Qwen Cloud / Alibaba DashScope
- تحتاج إلى فهم التوافق الخاص بمتغيرات البيئة لمزوّد qwen
- تريد استخدام نقطة نهاية الخطة Standard ‏(الدفع حسب الاستخدام) أو Coding Plan

---

# Qwen / Model Studio ‏(Alibaba Cloud)

توثق هذه الصفحة تعيين نقاط النهاية خلف مزوّد `qwen`
المضمّن في OpenClaw. يحافظ المزوّد على معرّفات مزوّد `modelstudio`، ومعرّفات auth-choice،
وmodel refs كأسماء مستعارة توافقية بينما يصبح `qwen` هو
السطح المعتمد.

<Info>

إذا كنت تحتاج إلى **`qwen3.6-plus`**، ففضّل **Standard ‏(الدفع حسب الاستخدام)**. قد يتأخر
توفر Coding Plan عن كتالوج Model Studio العام،
وقد ترفض API الخاصة بـ Coding Plan نموذجًا ما حتى يظهر في قائمة النماذج المدعومة
ضمن خطتك.

</Info>

- المزوّد: `qwen` ‏(اسم مستعار قديم: `modelstudio`)
- المصادقة: `QWEN_API_KEY`
- المقبول أيضًا: `MODELSTUDIO_API_KEY`، `DASHSCOPE_API_KEY`
- API: متوافقة مع OpenAI

## البدء السريع

### Standard ‏(الدفع حسب الاستخدام)

```bash
# نقطة نهاية الصين
openclaw onboard --auth-choice qwen-standard-api-key-cn

# نقطة النهاية العالمية/الدولية
openclaw onboard --auth-choice qwen-standard-api-key
```

### Coding Plan ‏(اشتراك)

```bash
# نقطة نهاية الصين
openclaw onboard --auth-choice qwen-api-key-cn

# نقطة النهاية العالمية/الدولية
openclaw onboard --auth-choice qwen-api-key
```

ما تزال معرّفات auth-choice القديمة `modelstudio-*` تعمل كأسماء مستعارة توافقية، لكن
معرّفات onboarding المعتمدة هي خيارات `qwen-*` المعروضة أعلاه.

بعد onboarding، عيّن نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## أنواع الخطط ونقاط النهاية

| الخطة                       | المنطقة | Auth choice                | نقطة النهاية                                      |
| --------------------------- | ------- | -------------------------- | ------------------------------------------------- |
| Standard ‏(الدفع حسب الاستخدام) | الصين   | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard ‏(الدفع حسب الاستخدام) | عالمي   | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan ‏(اشتراك)       | الصين   | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan ‏(اشتراك)       | عالمي   | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

يختار المزوّد نقطة النهاية تلقائيًا بناءً على auth choice الخاصة بك. وتستخدم الخيارات
المعتمدة عائلة `qwen-*`؛ أما `modelstudio-*` فتبقى للتوافق فقط.
ويمكنك
التجاوز باستخدام `baseUrl` مخصصة في التكوين.

تعلن نقاط نهاية Model Studio الأصلية عن توافق استخدام البث على
وسيلة النقل المشتركة `openai-completions`. ويعتمد OpenClaw الآن على إمكانات
نقطة النهاية لهذا الغرض، لذلك فإن معرّفات المزوّدات المخصصة المتوافقة مع DashScope والتي تستهدف
المضيفات الأصلية نفسها ترث سلوك استخدام البث نفسه بدلًا من
اشتراط معرّف المزوّد المضمّن `qwen` تحديدًا.

## احصل على مفتاح API الخاص بك

- **إدارة المفاتيح**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **الوثائق**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## الكتالوج المضمّن

يشحن OpenClaw حاليًا كتالوج Qwen المضمّن التالي:

| مرجع النموذج                | الإدخال      | السياق    | ملاحظات                                              |
| --------------------------- | ------------ | --------- | ---------------------------------------------------- |
| `qwen/qwen3.5-plus`         | نص، صورة     | 1,000,000 | النموذج الافتراضي                                    |
| `qwen/qwen3.6-plus`         | نص، صورة     | 1,000,000 | فضّل نقاط نهاية Standard عندما تحتاج إلى هذا النموذج |
| `qwen/qwen3-max-2026-01-23` | نص           | 262,144   | سلسلة Qwen Max                                       |
| `qwen/qwen3-coder-next`     | نص           | 262,144   | للبرمجة                                              |
| `qwen/qwen3-coder-plus`     | نص           | 1,000,000 | للبرمجة                                              |
| `qwen/MiniMax-M2.5`         | نص           | 1,000,000 | يدعم الاستدلال                                       |
| `qwen/glm-5`                | نص           | 202,752   | GLM                                                  |
| `qwen/glm-4.7`              | نص           | 202,752   | GLM                                                  |
| `qwen/kimi-k2.5`            | نص، صورة     | 262,144   | Moonshot AI عبر Alibaba                              |

قد يظل التوفر يختلف حسب نقطة النهاية وخطة الفوترة حتى عندما يكون النموذج
موجودًا في الكتالوج المضمّن.

ينطبق توافق استخدام البث الأصلي على كل من مضيفات Coding Plan
ومضيفات Standard المتوافقة مع DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## توفر Qwen 3.6 Plus

يتوفر `qwen3.6-plus` على نقاط نهاية Model Studio من نوع Standard ‏(الدفع حسب الاستخدام):

- الصين: `dashscope.aliyuncs.com/compatible-mode/v1`
- عالمي: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

إذا أعادت نقاط نهاية Coding Plan خطأ "unsupported model" بالنسبة إلى
`qwen3.6-plus`، فانتقل إلى Standard ‏(الدفع حسب الاستخدام) بدلًا من زوج
نقطة النهاية/المفتاح الخاص بـ Coding Plan.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن
`QWEN_API_KEY` متاح لتلك العملية (مثلًا في
`~/.openclaw/.env` أو عبر `env.shellEnv`).
