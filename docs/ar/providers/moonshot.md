---
read_when:
    - تريد إعداد Moonshot K2 (منصة Moonshot Open Platform) مقابل Kimi Coding
    - تحتاج إلى فهم نقاط النهاية والمفاتيح ومراجع النماذج المنفصلة
    - تريد تكوينًا جاهزًا للنسخ واللصق لأي من المزوّدين
summary: تكوين Moonshot K2 مقابل Kimi Coding (مزودون ومفاتيح منفصلة)
title: Moonshot AI
x-i18n:
    generated_at: "2026-04-05T12:53:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: a80c71ef432b778e296bd60b7d9ec7c72d025d13fd9bdae474b3d58436d15695
    source_path: providers/moonshot.md
    workflow: 15
---

# Moonshot AI (Kimi)

توفّر Moonshot واجهة Kimi API بنقاط نهاية متوافقة مع OpenAI. قم بتكوين
المزوّد واضبط النموذج الافتراضي على `moonshot/kimi-k2.5`، أو استخدم
Kimi Coding مع `kimi/kimi-code`.

معرّفات نماذج Kimi K2 الحالية:

[//]: # "moonshot-kimi-k2-ids:start"

- `kimi-k2.5`
- `kimi-k2-thinking`
- `kimi-k2-thinking-turbo`
- `kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-ids:end"

```bash
openclaw onboard --auth-choice moonshot-api-key
# or
openclaw onboard --auth-choice moonshot-api-key-cn
```

Kimi Coding:

```bash
openclaw onboard --auth-choice kimi-code-api-key
```

ملاحظة: Moonshot وKimi Coding مزوّدان منفصلان. المفاتيح غير قابلة للتبادل، ونقاط النهاية مختلفة، ومراجع النماذج مختلفة (تستخدم Moonshot الصيغة `moonshot/...`، بينما تستخدم Kimi Coding الصيغة `kimi/...`).

يستخدم بحث الويب في Kimi أيضًا plugin ‏Moonshot:

```bash
openclaw configure --section web
```

اختر **Kimi** في قسم بحث الويب لتخزين
`plugins.entries.moonshot.config.webSearch.*`.

## مقتطف التكوين (Moonshot API)

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: {
        // moonshot-kimi-k2-aliases:start
        "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
        "moonshot/kimi-k2-thinking": { alias: "Kimi K2 Thinking" },
        "moonshot/kimi-k2-thinking-turbo": { alias: "Kimi K2 Thinking Turbo" },
        "moonshot/kimi-k2-turbo": { alias: "Kimi K2 Turbo" },
        // moonshot-kimi-k2-aliases:end
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          // moonshot-kimi-k2-models:start
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-thinking",
            name: "Kimi K2 Thinking",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-thinking-turbo",
            name: "Kimi K2 Thinking Turbo",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-turbo",
            name: "Kimi K2 Turbo",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 256000,
            maxTokens: 16384,
          },
          // moonshot-kimi-k2-models:end
        ],
      },
    },
  },
}
```

## Kimi Coding

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: {
        "kimi/kimi-code": { alias: "Kimi" },
      },
    },
  },
}
```

## بحث الويب في Kimi

يشحن OpenClaw أيضًا **Kimi** كمزوّد `web_search`، ومدعومًا ببحث الويب من Moonshot.

يمكن أن يطلب الإعداد التفاعلي ما يلي:

- منطقة Moonshot API:
  - `https://api.moonshot.ai/v1`
  - `https://api.moonshot.cn/v1`
- نموذج بحث الويب الافتراضي لـ Kimi (الافتراضي `kimi-k2.5`)

يعيش التكوين تحت `plugins.entries.moonshot.config.webSearch`:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.5",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## ملاحظات

- تستخدم مراجع نماذج Moonshot الصيغة `moonshot/<modelId>`. وتستخدم مراجع نماذج Kimi Coding الصيغة `kimi/<modelId>`.
- مرجع نموذج Kimi Coding الافتراضي الحالي هو `kimi/kimi-code`. وما زال المرجع القديم `kimi/k2p5` مقبولًا كمعرّف نموذج للتوافق.
- يستخدم بحث الويب في Kimi المفتاح `KIMI_API_KEY` أو `MOONSHOT_API_KEY`، ويستخدم افتراضيًا `https://api.moonshot.ai/v1` مع النموذج `kimi-k2.5`.
- تعلن نقاط النهاية الأصلية لـ Moonshot (`https://api.moonshot.ai/v1` و
  `https://api.moonshot.cn/v1`) عن توافق استخدام البث على
  وسيلة النقل المشتركة `openai-completions`. ويعتمد OpenClaw الآن على قدرات نقطة النهاية،
  لذا ترث معرّفات المزوّدات المخصصة المتوافقة التي تستهدف مضيفات Moonshot الأصلية نفسها
  سلوك استخدام البث نفسه.
- تجاوز بيانات التسعير والسياق الوصفية في `models.providers` عند الحاجة.
- إذا نشرت Moonshot حدود سياق مختلفة لنموذج ما، فعدّل
  `contextWindow` وفقًا لذلك.
- استخدم `https://api.moonshot.ai/v1` لنقطة النهاية الدولية، و`https://api.moonshot.cn/v1` لنقطة نهاية الصين.
- خيارات onboarding:
  - `moonshot-api-key` من أجل `https://api.moonshot.ai/v1`
  - `moonshot-api-key-cn` من أجل `https://api.moonshot.cn/v1`

## وضع التفكير الأصلي (Moonshot)

تدعم Moonshot Kimi التفكير الأصلي الثنائي:

- `thinking: { type: "enabled" }`
- `thinking: { type: "disabled" }`

قم بتكوينه لكل نموذج عبر `agents.defaults.models.<provider/model>.params`:

```json5
{
  agents: {
    defaults: {
      models: {
        "moonshot/kimi-k2.5": {
          params: {
            thinking: { type: "disabled" },
          },
        },
      },
    },
  },
}
```

كما يعيّن OpenClaw مستويات `/think` في وقت التشغيل لـ Moonshot:

- `/think off` -> `thinking.type=disabled`
- أي مستوى تفكير غير off -> `thinking.type=enabled`

عند تمكين التفكير في Moonshot، يجب أن تكون `tool_choice` هي `auto` أو `none`. ويقوم OpenClaw بتوحيد قيم `tool_choice` غير المتوافقة إلى `auto` من أجل التوافق.
