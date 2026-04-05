---
read_when:
    - تريد استخدام نماذج OSS المستضافة على Bedrock Mantle مع OpenClaw
    - تحتاج إلى نقطة نهاية Mantle المتوافقة مع OpenAI لكل من GPT-OSS وQwen وKimi أو GLM
summary: استخدم نماذج Amazon Bedrock Mantle ‏(المتوافقة مع OpenAI) مع OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-05T12:52:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2efe61261fbb430f63be9f5025c0654c44b191dbe96b3eb081d7ccbe78458907
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

يتضمن OpenClaw مزود **Amazon Bedrock Mantle** مضمّنًا يتصل
بنقطة النهاية Mantle المتوافقة مع OpenAI. تستضيف Mantle نماذج مفتوحة المصدر
ونماذج تابعة لجهات خارجية (GPT-OSS وQwen وKimi وGLM وما شابه) عبر
سطح قياسي ` /v1/chat/completions` مدعوم ببنية Bedrock التحتية.

## ما الذي يدعمه OpenClaw

- المزوّد: `amazon-bedrock-mantle`
- API: ‏`openai-completions` ‏(متوافق مع OpenAI)
- المصادقة: bearer token عبر `AWS_BEARER_TOKEN_BEDROCK`
- المنطقة: `AWS_REGION` أو `AWS_DEFAULT_REGION` ‏(الافتراضي: `us-east-1`)

## الاكتشاف التلقائي للنماذج

عند ضبط `AWS_BEARER_TOKEN_BEDROCK`، يكتشف OpenClaw تلقائيًا
نماذج Mantle المتاحة عبر الاستعلام عن نقطة النهاية `/v1/models` الخاصة بالمنطقة.
وتُخزَّن نتائج الاكتشاف مؤقتًا لمدة ساعة واحدة.

المناطق المدعومة: `us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## الإعداد الأولي

1. اضبط bearer token على **مضيف gateway**:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# اختياري (الافتراضي هو us-east-1):
export AWS_REGION="us-west-2"
```

2. تحقق من اكتشاف النماذج:

```bash
openclaw models list
```

تظهر النماذج المكتشفة تحت المزوّد `amazon-bedrock-mantle`. ولا
يلزم أي إعداد إضافي إلا إذا كنت تريد تجاوز القيم الافتراضية.

## الإعداد اليدوي

إذا كنت تفضّل إعدادًا صريحًا بدل الاكتشاف التلقائي:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

## ملاحظات

- تتطلب Mantle حاليًا bearer token. ولا تكفي بيانات اعتماد IAM العادية (أدوار المثيلات،
  وSSO، ومفاتيح الوصول) من دون token.
- bearer token هي نفسها `AWS_BEARER_TOKEN_BEDROCK` المستخدمة من قبل
  مزود [Amazon Bedrock](/providers/bedrock) القياسي.
- يُستدل على دعم الاستدلال من معرّفات النماذج التي تحتوي على أنماط مثل
  `thinking` أو `reasoner` أو `gpt-oss-120b`.
- إذا كانت نقطة نهاية Mantle غير متاحة أو لم تُرجع أي نماذج، فسيُتخطى المزوّد
  بصمت.
