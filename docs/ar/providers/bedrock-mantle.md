---
read_when:
    - تريد استخدام نماذج OSS المستضافة على Bedrock Mantle مع OpenClaw
    - تحتاج إلى نقطة نهاية Mantle المتوافقة مع OpenAI لـ GPT-OSS أو Qwen أو Kimi أو GLM
summary: استخدم نماذج Amazon Bedrock Mantle ‏(المتوافقة مع OpenAI) مع OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-06T03:10:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e5b33ede4067fb7de02a046f3e375cbd2af4bf68e7751c8dd687447f1a78c86
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

يتضمن OpenClaw مزود **Amazon Bedrock Mantle** مدمجًا يتصل بنقطة نهاية Mantle المتوافقة مع OpenAI. يستضيف Mantle نماذج مفتوحة المصدر
ونماذج من جهات خارجية (مثل GPT-OSS وQwen وKimi وGLM وغيرها) عبر واجهة قياسية
`/v1/chat/completions` مدعومة ببنية Bedrock التحتية.

## ما الذي يدعمه OpenClaw

- المزود: `amazon-bedrock-mantle`
- API: ‏`openai-completions` (متوافق مع OpenAI)
- المصادقة: `AWS_BEARER_TOKEN_BEDROCK` صريح أو إنشاء bearer token من سلسلة بيانات اعتماد IAM
- المنطقة: `AWS_REGION` أو `AWS_DEFAULT_REGION` (الافتراضي: `us-east-1`)

## الاكتشاف التلقائي للنماذج

عندما يتم ضبط `AWS_BEARER_TOKEN_BEDROCK`، يستخدمه OpenClaw مباشرة. وإلا،
يحاول OpenClaw إنشاء bearer token لـ Mantle من سلسلة بيانات الاعتماد
الافتراضية لـ AWS، بما في ذلك ملفات بيانات الاعتماد/الإعدادات المشتركة، وSSO، وweb
identity، وأدوار المثيل أو المهام. ثم يكتشف نماذج Mantle المتاحة
عن طريق الاستعلام من نقطة النهاية `/v1/models` الخاصة بالمنطقة. تُخزَّن نتائج الاكتشاف
مؤقتًا لمدة ساعة واحدة، ويتم تحديث bearer tokens المشتقة من IAM كل ساعة.

المناطق المدعومة: `us-east-1` و`us-east-2` و`us-west-2` و`ap-northeast-1`،
`ap-south-1` و`ap-southeast-3` و`eu-central-1` و`eu-west-1` و`eu-west-2`،
`eu-south-1` و`eu-north-1` و`sa-east-1`.

## الإعداد الأولي

1. اختر مسار مصادقة واحدًا على **gateway host**:

Bearer token صريح:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# اختياري (الافتراضي هو us-east-1):
export AWS_REGION="us-west-2"
```

بيانات اعتماد IAM:

```bash
# أي مصدر مصادقة متوافق مع AWS SDK يعمل هنا، على سبيل المثال:
export AWS_PROFILE="default"
export AWS_REGION="us-west-2"
```

2. تحقق من اكتشاف النماذج:

```bash
openclaw models list
```

تظهر النماذج المكتشفة تحت المزود `amazon-bedrock-mantle`. ولا يلزم
أي إعداد إضافي إلا إذا كنت تريد تجاوز القيم الافتراضية.

## الإعداد اليدوي

إذا كنت تفضّل الإعداد الصريح بدلًا من الاكتشاف التلقائي:

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

- يمكن لـ OpenClaw إنشاء bearer token الخاص بـ Mantle نيابةً عنك من بيانات اعتماد IAM
  المتوافقة مع AWS SDK عندما لا يكون `AWS_BEARER_TOKEN_BEDROCK` مضبوطًا.
- bearer token هو نفسه `AWS_BEARER_TOKEN_BEDROCK` المستخدم من قبل مزود
  [Amazon Bedrock](/ar/providers/bedrock) القياسي.
- يتم استنتاج دعم الاستدلال من معرّفات النماذج التي تحتوي على أنماط مثل
  `thinking` أو `reasoner` أو `gpt-oss-120b`.
- إذا كانت نقطة نهاية Mantle غير متاحة أو لم تُرجع أي نماذج، فسيتم
  تخطي المزود بصمت.
