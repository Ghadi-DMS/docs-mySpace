---
read_when:
    - أنت تريد استخدام النماذج المفتوحة في OpenClaw مجانًا
    - تحتاج إلى إعداد NVIDIA_API_KEY
summary: استخدم API المتوافق مع OpenAI من NVIDIA في OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-04-08T02:18:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: b00f8cedaf223a33ba9f6a6dd8cf066d88cebeea52d391b871e435026182228a
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

توفر NVIDIA واجهة API متوافقة مع OpenAI على `https://integrate.api.nvidia.com/v1` للنماذج المفتوحة مجانًا. قم بالمصادقة باستخدام مفتاح API من [build.nvidia.com](https://build.nvidia.com/settings/api-keys).

## إعداد CLI

صدّر المفتاح مرة واحدة، ثم شغّل التهيئة واضبط نموذج NVIDIA:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/nemotron-3-super-120b-a12b
```

إذا كنت ما تزال تمرر `--token`، فتذكر أنه يُحفَظ في سجل الصدفة ومخرجات `ps`؛ ويفضل استخدام متغير البيئة عند الإمكان.

## مقتطف التكوين

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-super-120b-a12b" },
    },
  },
}
```

## معرّفات النماذج

| مرجع النموذج | الاسم | السياق | الحد الأقصى للإخراج |
| ------------------------------------------ | ---------------------------- | ------- | ---------- |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | NVIDIA Nemotron 3 Super 120B | 262,144 | 8,192      |
| `nvidia/moonshotai/kimi-k2.5`              | Kimi K2.5                    | 262,144 | 8,192      |
| `nvidia/minimaxai/minimax-m2.5`            | Minimax M2.5                 | 196,608 | 8,192      |
| `nvidia/z-ai/glm5`                         | GLM 5                        | 202,752 | 8,192      |

## ملاحظات

- نقطة نهاية `/v1` متوافقة مع OpenAI؛ استخدم مفتاح API من [build.nvidia.com](https://build.nvidia.com/).
- يتم تمكين الموفّر تلقائيًا عند تعيين `NVIDIA_API_KEY`.
- الفهرس المجمّع ثابت؛ وتكون التكاليف مضبوطة افتراضيًا على `0` في المصدر.
