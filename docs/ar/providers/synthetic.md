---
read_when:
    - تريد استخدام Synthetic كمزوّد نماذج
    - تحتاج إلى مفتاح API أو إعداد base URL لـ Synthetic
summary: استخدام API المتوافقة مع Anthropic من Synthetic في OpenClaw
title: Synthetic
x-i18n:
    generated_at: "2026-04-05T12:54:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3495bca5cb134659cf6c54e31fa432989afe0cc04f53cf3e3146ce80a5e8af49
    source_path: providers/synthetic.md
    workflow: 15
---

# Synthetic

توفّر Synthetic نقاط نهاية متوافقة مع Anthropic. ويسجّلها OpenClaw على أنها
المزوّد `synthetic` ويستخدم Anthropic Messages API.

## إعداد سريع

1. اضبط `SYNTHETIC_API_KEY` (أو شغّل المعالج أدناه).
2. شغّل التهيئة الأولية:

```bash
openclaw onboard --auth-choice synthetic-api-key
```

يُضبط النموذج الافتراضي على:

```
synthetic/hf:MiniMaxAI/MiniMax-M2.5
```

## مثال على التكوين

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

ملاحظة: يضيف عميل Anthropic في OpenClaw اللاحقة `/v1` إلى base URL، لذا استخدم
`https://api.synthetic.new/anthropic` ‏(وليس `/anthropic/v1`). وإذا غيّرت Synthetic
عنوان base URL، فتجاوز `models.providers.synthetic.baseUrl`.

## فهرس النماذج

تستخدم جميع النماذج أدناه تكلفة `0` ‏(الإدخال/الإخراج/الذاكرة المؤقتة).

| معرّف النموذج                                          | نافذة السياق | الحد الأقصى للرموز | الاستدلال | الإدخال       |
| ------------------------------------------------------ | ------------ | ------------------ | --------- | ------------- |
| `hf:MiniMaxAI/MiniMax-M2.5`                            | 192000       | 65536              | false     | نص            |
| `hf:moonshotai/Kimi-K2-Thinking`                       | 256000       | 8192               | true      | نص            |
| `hf:zai-org/GLM-4.7`                                   | 198000       | 128000             | false     | نص            |
| `hf:deepseek-ai/DeepSeek-R1-0528`                      | 128000       | 8192               | false     | نص            |
| `hf:deepseek-ai/DeepSeek-V3-0324`                      | 128000       | 8192               | false     | نص            |
| `hf:deepseek-ai/DeepSeek-V3.1`                         | 128000       | 8192               | false     | نص            |
| `hf:deepseek-ai/DeepSeek-V3.1-Terminus`                | 128000       | 8192               | false     | نص            |
| `hf:deepseek-ai/DeepSeek-V3.2`                         | 159000       | 8192               | false     | نص            |
| `hf:meta-llama/Llama-3.3-70B-Instruct`                 | 128000       | 8192               | false     | نص            |
| `hf:meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | 524000       | 8192               | false     | نص            |
| `hf:moonshotai/Kimi-K2-Instruct-0905`                  | 256000       | 8192               | false     | نص            |
| `hf:moonshotai/Kimi-K2.5`                              | 256000       | 8192               | true      | نص + صورة     |
| `hf:openai/gpt-oss-120b`                               | 128000       | 8192               | false     | نص            |
| `hf:Qwen/Qwen3-235B-A22B-Instruct-2507`                | 256000       | 8192               | false     | نص            |
| `hf:Qwen/Qwen3-Coder-480B-A35B-Instruct`               | 256000       | 8192               | false     | نص            |
| `hf:Qwen/Qwen3-VL-235B-A22B-Instruct`                  | 250000       | 8192               | false     | نص + صورة     |
| `hf:zai-org/GLM-4.5`                                   | 128000       | 128000             | false     | نص            |
| `hf:zai-org/GLM-4.6`                                   | 198000       | 128000             | false     | نص            |
| `hf:zai-org/GLM-5`                                     | 256000       | 128000             | true      | نص + صورة     |
| `hf:deepseek-ai/DeepSeek-V3`                           | 128000       | 8192               | false     | نص            |
| `hf:Qwen/Qwen3-235B-A22B-Thinking-2507`                | 256000       | 8192               | true      | نص            |

## ملاحظات

- تستخدم مراجع النماذج الصيغة `synthetic/<modelId>`.
- إذا فعّلت allowlist للنماذج (`agents.defaults.models`)، فأضف كل نموذج
  تخطط لاستخدامه.
- راجع [Model providers](/concepts/model-providers) لمعرفة قواعد المزوّد.
