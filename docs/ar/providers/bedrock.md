---
read_when:
    - تريد استخدام نماذج Amazon Bedrock مع OpenClaw
    - تحتاج إلى إعداد بيانات اعتماد/منطقة AWS لاستدعاءات النماذج
summary: استخدم نماذج Amazon Bedrock ‏(Converse API) مع OpenClaw
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-04-06T03:11:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 70bb29fe9199084b1179ced60935b5908318f5b80ced490bf44a45e0467c4929
    source_path: providers/bedrock.md
    workflow: 15
---

# Amazon Bedrock

يمكن لـ OpenClaw استخدام نماذج **Amazon Bedrock** عبر مزود البث **Bedrock Converse**
من pi‑ai. تستخدم مصادقة Bedrock **سلسلة بيانات الاعتماد الافتراضية لـ AWS SDK**،
وليس مفتاح API.

## ما الذي يدعمه pi-ai

- المزود: `amazon-bedrock`
- API: ‏`bedrock-converse-stream`
- المصادقة: بيانات اعتماد AWS ‏(متغيرات البيئة أو الإعدادات المشتركة أو دور المثيل)
- المنطقة: `AWS_REGION` أو `AWS_DEFAULT_REGION` (الافتراضي: `us-east-1`)

## الاكتشاف التلقائي للنماذج

يمكن لـ OpenClaw اكتشاف نماذج Bedrock التي تدعم **البث**
و**إخراج النص** تلقائيًا. يستخدم الاكتشاف `bedrock:ListFoundationModels` و
`bedrock:ListInferenceProfiles`، وتُخزَّن النتائج مؤقتًا (الافتراضي: ساعة واحدة).

كيفية تمكين المزود الضمني:

- إذا كانت `plugins.entries.amazon-bedrock.config.discovery.enabled` تساوي `true`،
  فسيحاول OpenClaw الاكتشاف حتى عند عدم وجود أي مؤشر بيئة AWS.
- إذا لم تكن `plugins.entries.amazon-bedrock.config.discovery.enabled` مضبوطة،
  فإن OpenClaw يضيف تلقائيًا مزود Bedrock
  الضمني فقط عندما يرى أحد مؤشرات مصادقة AWS التالية:
  `AWS_BEARER_TOKEN_BEDROCK`، أو `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`، أو `AWS_PROFILE`.
- ما يزال مسار مصادقة Bedrock الفعلي في وقت التشغيل يستخدم السلسلة الافتراضية لـ AWS SDK، لذا
  يمكن أن تعمل الإعدادات المشتركة وSSO ومصادقة دور المثيل عبر IMDS حتى عندما يحتاج الاكتشاف
  إلى `enabled: true` للاشتراك الصريح.

توجد خيارات الإعدادات تحت `plugins.entries.amazon-bedrock.config.discovery`:

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

- تكون `enabled` افتراضيًا في وضع auto. وفي وضع auto، لا يفعّل OpenClaw
  مزود Bedrock الضمني إلا عندما يرى مؤشر بيئة AWS مدعومًا.
- تكون `region` افتراضيًا من `AWS_REGION` أو `AWS_DEFAULT_REGION`، ثم `us-east-1`.
- تطابق `providerFilter` أسماء مزودي Bedrock (على سبيل المثال `anthropic`).
- `refreshInterval` بالثواني؛ اضبطه على `0` لتعطيل التخزين المؤقت.
- تُستخدم `defaultContextWindow` (الافتراضي: `32000`) و`defaultMaxTokens` (الافتراضي: `4096`)
  للنماذج المكتشفة (يمكنك تجاوزهما إذا كنت تعرف حدود نموذجك).
- بالنسبة إلى إدخالات `models.providers["amazon-bedrock"]` الصريحة، لا يزال OpenClaw قادرًا على
  حل مصادقة مؤشرات بيئة Bedrock مبكرًا من مؤشرات بيئة AWS مثل
  `AWS_BEARER_TOKEN_BEDROCK` دون فرض تحميل كامل لمصادقة وقت التشغيل. ولا يزال
  مسار مصادقة استدعاء النموذج الفعلي يستخدم السلسلة الافتراضية لـ AWS SDK.

## الإعداد الأولي

1. تأكد من توفر بيانات اعتماد AWS على **gateway host**:

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
# اختياري:
export AWS_SESSION_TOKEN="..."
export AWS_PROFILE="your-profile"
# اختياري (مفتاح API / bearer token لـ Bedrock):
export AWS_BEARER_TOKEN_BEDROCK="..."
```

2. أضف مزود Bedrock ونموذجًا إلى إعداداتك (لا حاجة إلى `apiKey`):

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

عند تشغيل OpenClaw على مثيل EC2 مع إرفاق دور IAM، يمكن لـ AWS SDK
استخدام خدمة بيانات تعريف المثيل (IMDS) للمصادقة. بالنسبة إلى اكتشاف نماذج Bedrock،
لا يفعّل OpenClaw المزود الضمني تلقائيًا إلا من خلال مؤشرات بيئة AWS
ما لم تقم صراحةً بضبط
`plugins.entries.amazon-bedrock.config.discovery.enabled: true`.

الإعداد الموصى به للمضيفين المعتمدين على IMDS:

- اضبط `plugins.entries.amazon-bedrock.config.discovery.enabled` على `true`.
- اضبط `plugins.entries.amazon-bedrock.config.discovery.region` (أو صدّر `AWS_REGION`).
- **لا** تحتاج إلى مفتاح API وهمي.
- تحتاج إلى `AWS_PROFILE=default` فقط إذا كنت تريد تحديدًا مؤشر بيئة
  لوضع auto أو لأسطح الحالة.

```bash
# موصى به: تمكين صريح للاكتشاف + المنطقة
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# اختياري: أضف مؤشر بيئة إذا كنت تريد وضع auto بدون تمكين صريح
export AWS_PROFILE=default
export AWS_REGION=us-east-1
```

**أذونات IAM المطلوبة** لدور مثيل EC2:

- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- `bedrock:ListFoundationModels` (للاكتشاف التلقائي)
- `bedrock:ListInferenceProfiles` (لاكتشاف ملفات تعريف الاستدلال)

أو قم بإرفاق السياسة المُدارة `AmazonBedrockFullAccess`.

## إعداد سريع (مسار AWS)

```bash
# 1. أنشئ دور IAM وملف تعريف مثيل
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

# 2. أرفقه بمثيل EC2 الخاص بك
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. على مثيل EC2، فعّل الاكتشاف صراحةً
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. اختياري: أضف مؤشر بيئة إذا كنت تريد وضع auto بدون تمكين صريح
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. تحقق من اكتشاف النماذج
openclaw models list
```

## ملفات تعريف الاستدلال

يكتشف OpenClaw **ملفات تعريف الاستدلال الإقليمية والعالمية** إلى جانب
النماذج الأساسية. عندما يطابق ملف تعريف نموذجًا أساسيًا معروفًا، فإن
ملف التعريف يرث قدرات ذلك النموذج (نافذة السياق، الحد الأقصى للرموز،
الاستدلال، الرؤية) وتُحقن منطقة طلب Bedrock الصحيحة
تلقائيًا. وهذا يعني أن ملفات تعريف Claude عبر المناطق تعمل دون تجاوزات
يدوية للمزود.

تبدو معرّفات ملفات تعريف الاستدلال مثل `us.anthropic.claude-opus-4-6-v1:0` (إقليمي)
أو `anthropic.claude-opus-4-6-v1:0` (عالمي). إذا كان النموذج الخلفي موجودًا بالفعل
في نتائج الاكتشاف، فإن ملف التعريف يرث مجموعة قدراته الكاملة؛
وإلا فستُطبق قيم افتراضية آمنة.

لا حاجة إلى أي إعداد إضافي. ما دام الاكتشاف مفعّلًا وكان كيان IAM
يملك الإذن `bedrock:ListInferenceProfiles`، فستظهر ملفات التعريف إلى جانب
النماذج الأساسية في `openclaw models list`.

## ملاحظات

- يتطلب Bedrock تفعيل **الوصول إلى النموذج** في حسابك/منطقتك على AWS.
- يحتاج الاكتشاف التلقائي إلى الإذنين `bedrock:ListFoundationModels` و
  `bedrock:ListInferenceProfiles`.
- إذا كنت تعتمد على وضع auto، فاضبط أحد مؤشرات بيئة مصادقة AWS المدعومة على
  gateway host. وإذا كنت تفضل مصادقة IMDS/الإعدادات المشتركة بدون مؤشرات بيئة، فاضبط
  `plugins.entries.amazon-bedrock.config.discovery.enabled: true`.
- يعرض OpenClaw مصدر بيانات الاعتماد بهذا الترتيب: `AWS_BEARER_TOKEN_BEDROCK`،
  ثم `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`، ثم `AWS_PROFILE`، ثم
  السلسلة الافتراضية لـ AWS SDK.
- يعتمد دعم الاستدلال على النموذج؛ تحقّق من بطاقة نموذج Bedrock للاطلاع على
  الإمكانات الحالية.
- إذا كنت تفضّل تدفقًا مُدارًا للمفاتيح، فيمكنك أيضًا وضع وكيل
  متوافق مع OpenAI أمام Bedrock وتهيئته بدلًا من ذلك كمزود OpenAI.

## الحواجز الوقائية

يمكنك تطبيق [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
على جميع استدعاءات نماذج Bedrock بإضافة كائن `guardrail` إلى
إعدادات إضافة `amazon-bedrock`. تتيح لك Guardrails فرض تصفية المحتوى،
ورفض الموضوعات، وفلاتر الكلمات، وفلاتر المعلومات الحساسة، وفحوصات
الاستناد السياقي.

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          guardrail: {
            guardrailIdentifier: "abc123", // معرّف guardrail أو ARN كامل
            guardrailVersion: "1", // رقم الإصدار أو "DRAFT"
            streamProcessingMode: "sync", // اختياري: "sync" أو "async"
            trace: "enabled", // اختياري: "enabled" أو "disabled" أو "enabled_full"
          },
        },
      },
    },
  },
}
```

- يقبل `guardrailIdentifier` (مطلوب) معرّف guardrail (مثل `abc123`) أو
  ARN كاملًا (مثل `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`).
- يحدد `guardrailVersion` (مطلوب) أي إصدار منشور يجب استخدامه، أو
  `"DRAFT"` للمسودة العاملة.
- يتحكم `streamProcessingMode` (اختياري) في ما إذا كان تقييم guardrail يُنفّذ
  بشكل متزامن (`"sync"`) أو غير متزامن (`"async"`) أثناء البث. وإذا
  أُهمل، يستخدم Bedrock سلوكه الافتراضي.
- يفعّل `trace` (اختياري) مخرجات تتبع guardrail في استجابة API. اضبطه على
  `"enabled"` أو `"enabled_full"` لأغراض التصحيح؛ واحذفه أو اضبطه على `"disabled"` في
  الإنتاج.

يجب أن يملك كيان IAM الذي تستخدمه البوابة الإذن `bedrock:ApplyGuardrail`
بالإضافة إلى أذونات الاستدعاء القياسية.

## التضمينات لبحث الذاكرة

يمكن أن يعمل Bedrock أيضًا كمزود تضمينات لـ
[بحث الذاكرة](/ar/concepts/memory-search). ويتم إعداد هذا بشكل منفصل عن
مزود الاستدلال — اضبط `agents.defaults.memorySearch.provider` على `"bedrock"`:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0", // الافتراضي
      },
    },
  },
}
```

تستخدم تضمينات Bedrock السلسلة نفسها لبيانات الاعتماد في AWS SDK كما في الاستدلال (أدوار
المثيل، وSSO، ومفاتيح الوصول، والإعدادات المشتركة، وweb identity). لا
توجد حاجة إلى مفتاح API. عندما تكون `provider` هي `"auto"`، يتم
اكتشاف Bedrock تلقائيًا إذا تم حل سلسلة بيانات الاعتماد هذه بنجاح.

تشمل نماذج التضمين المدعومة Amazon Titan Embed ‏(v1 وv2)، وAmazon Nova
Embed، وCohere Embed ‏(v3 وv4)، وTwelveLabs Marengo. راجع
[مرجع إعدادات الذاكرة — Bedrock](/ar/reference/memory-config#bedrock-embedding-config)
للحصول على قائمة النماذج الكاملة وخيارات الأبعاد.
