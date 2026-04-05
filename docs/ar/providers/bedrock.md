---
read_when:
    - تريد استخدام نماذج Amazon Bedrock مع OpenClaw
    - تحتاج إلى إعداد بيانات اعتماد/منطقة AWS لاستدعاءات النماذج
summary: استخدام نماذج Amazon Bedrock ‏(Converse API) مع OpenClaw
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-04-05T12:53:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: a751824b679a9340db714ee5227e8d153f38f6c199ca900458a4ec092b4efe54
    source_path: providers/bedrock.md
    workflow: 15
---

# Amazon Bedrock

يمكن لـ OpenClaw استخدام نماذج **Amazon Bedrock** عبر مزوّد البث **Bedrock Converse**
من pi‑ai. تستخدم مصادقة Bedrock **سلسلة بيانات الاعتماد الافتراضية لـ AWS SDK**،
وليس مفتاح API.

## ما الذي يدعمه pi-ai

- المزوّد: `amazon-bedrock`
- API: ‏`bedrock-converse-stream`
- المصادقة: بيانات اعتماد AWS ‏(متغيرات env أو التكوين المشترك أو دور المثيل)
- المنطقة: `AWS_REGION` أو `AWS_DEFAULT_REGION` ‏(الافتراضي: `us-east-1`)

## اكتشاف النماذج تلقائيًا

يمكن لـ OpenClaw اكتشاف نماذج Bedrock التي تدعم **البث**
و**الإخراج النصي** تلقائيًا. يستخدم الاكتشاف `bedrock:ListFoundationModels` و
`bedrock:ListInferenceProfiles`، ويتم تخزين النتائج مؤقتًا (الافتراضي: ساعة واحدة).

كيفية تمكين المزوّد الضمني:

- إذا كانت `plugins.entries.amazon-bedrock.config.discovery.enabled` تساوي `true`،
  فسيحاول OpenClaw الاكتشاف حتى عند عدم وجود علامة AWS env.
- إذا كانت `plugins.entries.amazon-bedrock.config.discovery.enabled` غير مضبوطة،
  فلن يضيف OpenClaw
  مزوّد Bedrock الضمني تلقائيًا إلا عندما يرى إحدى علامات مصادقة AWS التالية:
  `AWS_BEARER_TOKEN_BEDROCK` أو `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY` أو `AWS_PROFILE`.
- ما يزال مسار مصادقة Bedrock الفعلي في وقت التشغيل يستخدم سلسلة AWS SDK الافتراضية، لذا
  يمكن أن يعمل التكوين المشترك وSSO ومصادقة دور المثيل IMDS حتى عندما يحتاج الاكتشاف
  إلى `enabled: true` للاشتراك الصريح.

توجد خيارات التكوين تحت `plugins.entries.amazon-bedrock.config.discovery`:

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          discovery: {
            enabled: true,
            region: "us-east-1",
            providerFilter: ["anthropic", "amazon"],
            refreshInterval: 3600,
            defaultContextWindow: 32000,
            defaultMaxTokens: 4096,
          },
        },
      },
    },
  },
}
```

ملاحظات:

- القيمة الافتراضية لـ `enabled` هي الوضع التلقائي. وفي هذا الوضع، لا يمكّن OpenClaw
  مزوّد Bedrock الضمني إلا عندما يرى علامة env مدعومة من AWS.
- تكون القيمة الافتراضية لـ `region` هي `AWS_REGION` أو `AWS_DEFAULT_REGION`، ثم `us-east-1`.
- يطابق `providerFilter` أسماء مزوّدي Bedrock ‏(مثل `anthropic`).
- يكون `refreshInterval` بالثواني؛ اضبطه على `0` لتعطيل التخزين المؤقت.
- يُستخدم `defaultContextWindow` (الافتراضي: `32000`) و`defaultMaxTokens` (الافتراضي: `4096`)
  للنماذج المكتشفة (تجاوزهما إذا كنت تعرف حدود النموذج لديك).
- بالنسبة إلى الإدخالات الصريحة `models.providers["amazon-bedrock"]`، يستطيع OpenClaw أيضًا
  حل مصادقة علامات env الخاصة بـ Bedrock مبكرًا من علامات AWS env مثل
  `AWS_BEARER_TOKEN_BEDROCK` من دون فرض تحميل المصادقة الكامل وقت التشغيل. أمّا
  مسار المصادقة الفعلي لاستدعاء النموذج فما يزال يستخدم سلسلة AWS SDK الافتراضية.

## التهيئة الأولية

1. تأكد من توفر بيانات اعتماد AWS على **مضيف gateway**:

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
# Optional:
export AWS_SESSION_TOKEN="..."
export AWS_PROFILE="your-profile"
# Optional (Bedrock API key/bearer token):
export AWS_BEARER_TOKEN_BEDROCK="..."
```

2. أضف مزوّد Bedrock ونموذجًا إلى التكوين لديك (لا حاجة إلى `apiKey`):

```json5
{
  models: {
    providers: {
      "amazon-bedrock": {
        baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
        api: "bedrock-converse-stream",
        auth: "aws-sdk",
        models: [
          {
            id: "us.anthropic.claude-opus-4-6-v1:0",
            name: "Claude Opus 4.6 (Bedrock)",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0" },
    },
  },
}
```

## أدوار مثيل EC2

عند تشغيل OpenClaw على مثيل EC2 مرفق به دور IAM، يستطيع AWS SDK
استخدام خدمة بيانات تعريف المثيل (IMDS) للمصادقة. وبالنسبة إلى اكتشاف نماذج Bedrock، لا يمكّن OpenClaw المزوّد الضمني تلقائيًا من علامات AWS env إلا إذا قمت بضبط
`plugins.entries.amazon-bedrock.config.discovery.enabled: true` صراحةً.

الإعداد الموصى به للمضيفات المدعومة بـ IMDS:

- اضبط `plugins.entries.amazon-bedrock.config.discovery.enabled` على `true`.
- اضبط `plugins.entries.amazon-bedrock.config.discovery.region` (أو صدّر `AWS_REGION`).
- لا تحتاج إلى مفتاح API زائف.
- تحتاج إلى `AWS_PROFILE=default` فقط إذا كنت تريد تحديدًا علامة env
  للوضع التلقائي أو لأسطح الحالة.

```bash
# Recommended: explicit discovery enable + region
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# Optional: add an env marker if you want auto mode without explicit enable
export AWS_PROFILE=default
export AWS_REGION=us-east-1
```

**أذونات IAM المطلوبة** لدور مثيل EC2:

- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- `bedrock:ListFoundationModels` ‏(للاكتشاف التلقائي)
- `bedrock:ListInferenceProfiles` ‏(لاكتشاف inference profile)

أو أرفق السياسة المُدارة `AmazonBedrockFullAccess`.

## إعداد سريع (مسار AWS)

```bash
# 1. Create IAM role and instance profile
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. Attach to your EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. On the EC2 instance, enable discovery explicitly
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. Optional: add an env marker if you want auto mode without explicit enable
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. Verify models are discovered
openclaw models list
```

## Inference profiles

يكتشف OpenClaw **inference profiles الإقليمية والعالمية** إلى جانب
foundation models. وعندما تطابق profile نموذج foundation معروفًا، فإن
profile ترث إمكانات ذلك النموذج (نافذة السياق، والحد الأقصى للرموز،
والاستدلال، والرؤية)، كما يتم حقن منطقة طلب Bedrock الصحيحة
تلقائيًا. وهذا يعني أن ملفات Claude عبر المناطق تعمل من دون تجاوزات يدوية للمزوّد.

تبدو معرّفات inference profile مثل `us.anthropic.claude-opus-4-6-v1:0` ‏(إقليمية)
أو `anthropic.claude-opus-4-6-v1:0` ‏(عالمية). وإذا كان النموذج الخلفي موجودًا بالفعل
في نتائج الاكتشاف، فإن profile ترث مجموعة إمكاناته الكاملة؛
وإلا تُستخدم افتراضيات آمنة.

لا حاجة إلى أي تكوين إضافي. ما دام الاكتشاف مفعّلًا وكان لدى IAM
principal الإذن `bedrock:ListInferenceProfiles`، فستظهر profiles إلى جانب
foundation models في `openclaw models list`.

## ملاحظات

- تتطلب Bedrock **وصول النموذج** مفعّلًا في حساب/منطقة AWS لديك.
- يتطلب الاكتشاف التلقائي الأذونات `bedrock:ListFoundationModels` و
  `bedrock:ListInferenceProfiles`.
- إذا كنت تعتمد على الوضع التلقائي، فاضبط إحدى علامات AWS auth env المدعومة على
  مضيف gateway. وإذا كنت تفضّل مصادقة IMDS/shared-config من دون علامات env، فاضبط
  `plugins.entries.amazon-bedrock.config.discovery.enabled: true`.
- يعرض OpenClaw مصدر بيانات الاعتماد بهذا الترتيب: `AWS_BEARER_TOKEN_BEDROCK`,
  ثم `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`، ثم `AWS_PROFILE`، ثم
  سلسلة AWS SDK الافتراضية.
- يعتمد دعم الاستدلال على النموذج؛ تحقق من بطاقة نموذج Bedrock لمعرفة
  الإمكانات الحالية.
- إذا كنت تفضّل تدفق مفاتيح مُدارًا، فيمكنك أيضًا وضع
  proxy متوافق مع OpenAI أمام Bedrock وتكوينه كمزوّد OpenAI بدلًا من ذلك.

## Guardrails

يمكنك تطبيق [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
على جميع استدعاءات نماذج Bedrock بإضافة كائن `guardrail` إلى
تكوين plugin الخاصة بـ `amazon-bedrock`. تتيح لك Guardrails فرض تصفية المحتوى،
ورفض الموضوعات، ومرشحات الكلمات، ومرشحات المعلومات الحساسة، وفحوصات
الاستناد السياقي.

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          guardrail: {
            guardrailIdentifier: "abc123", // guardrail ID or full ARN
            guardrailVersion: "1", // version number or "DRAFT"
            streamProcessingMode: "sync", // optional: "sync" or "async"
            trace: "enabled", // optional: "enabled", "disabled", or "enabled_full"
          },
        },
      },
    },
  },
}
```

- يقبل `guardrailIdentifier` ‏(مطلوب) معرّف guardrail ‏(مثل `abc123`) أو
  ARN كاملًا (مثل `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`).
- يحدد `guardrailVersion` ‏(مطلوب) أي إصدار منشور يجب استخدامه، أو
  `"DRAFT"` للمسودة العاملة.
- يتحكم `streamProcessingMode` ‏(اختياري) في ما إذا كان تقييم guardrail يعمل
  بشكل متزامن (`"sync"`) أو غير متزامن (`"async"`) أثناء البث. وإذا
  حُذف، تستخدم Bedrock سلوكها الافتراضي.
- يفعّل `trace` ‏(اختياري) مخرجات trace الخاصة بـ guardrail في استجابة API. اضبطه على
  `"enabled"` أو `"enabled_full"` لأغراض التصحيح، أو احذفه/اضبطه على `"disabled"` في
  الإنتاج.

يجب أن يملك IAM principal الذي تستخدمه gateway الإذن `bedrock:ApplyGuardrail`
بالإضافة إلى أذونات الاستدعاء القياسية.
