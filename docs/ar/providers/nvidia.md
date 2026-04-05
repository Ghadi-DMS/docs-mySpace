---
read_when:
    - أنت تريد استخدام نماذج NVIDIA في OpenClaw
    - أنت تحتاج إلى إعداد NVIDIA_API_KEY
summary: استخدم API المتوافقة مع OpenAI من NVIDIA في OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-04-05T12:53:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: a24c5e46c0cf0fbc63bf09c772b486dd7f8f4b52e687d3b835bb54a1176b28da
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

توفر NVIDIA واجهة API متوافقة مع OpenAI عند `https://integrate.api.nvidia.com/v1` لنماذج Nemotron وNeMo. قم بالمصادقة باستخدام مفتاح API من [NVIDIA NGC](https://catalog.ngc.nvidia.com/).

## إعداد CLI

قم بتصدير المفتاح مرة واحدة، ثم شغّل onboarding واضبط نموذج NVIDIA:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/llama-3.1-nemotron-70b-instruct
```

إذا كنت لا تزال تمرر `--token`، فتذكر أنه سيظهر في سجل shell ومخرجات `ps`؛ لذا فضّل متغير البيئة عندما يكون ذلك ممكنًا.

## مقتطف إعدادات

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
      model: { primary: "nvidia/nvidia/llama-3.1-nemotron-70b-instruct" },
    },
  },
}
```

## معرّفات النماذج

| مرجع النموذج                                          | الاسم                                      | السياق  | الحد الأقصى للخرج |
| ---------------------------------------------------- | ----------------------------------------- | ------- | ----------------- |
| `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`      | NVIDIA Llama 3.1 Nemotron 70B Instruct    | 131,072 | 4,096             |
| `nvidia/meta/llama-3.3-70b-instruct`                 | Meta Llama 3.3 70B Instruct               | 131,072 | 4,096             |
| `nvidia/nvidia/mistral-nemo-minitron-8b-8k-instruct` | NVIDIA Mistral NeMo Minitron 8B Instruct  | 8,192   | 2,048             |

## ملاحظات

- نقطة نهاية `/v1` متوافقة مع OpenAI؛ استخدم مفتاح API من NVIDIA NGC.
- يتم تمكين المزوّد تلقائيًا عند ضبط `NVIDIA_API_KEY`.
- يكون الفهرس المضمّن ثابتًا؛ وتكون التكاليف الافتراضية في المصدر هي `0`.
