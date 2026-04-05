---
read_when:
    - إعداد Synology Chat مع OpenClaw
    - تصحيح مسار توجيه webhook في Synology Chat
summary: إعداد webhook لـ Synology Chat وإعدادات OpenClaw
title: Synology Chat
x-i18n:
    generated_at: "2026-04-05T12:36:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: ddb25fc6b53f896f15f43b4936d69ea071a29a91838a5b662819377271e89d81
    source_path: channels/synology-chat.md
    workflow: 15
---

# Synology Chat

الحالة: قناة رسائل مباشرة عبر plugin مضمّن باستخدام webhooks لـ Synology Chat.
يقبل plugin الرسائل الواردة من webhooks الصادرة في Synology Chat ويرسل الردود
عبر webhook وارد في Synology Chat.

## plugin المضمّن

يأتي Synology Chat كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك لا تحتاج
البنيات المعبأة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستبعد Synology Chat،
فثبّته يدويًا:

ثبّت من نسخة checkout محلية:

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## إعداد سريع

1. تأكد من أن plugin الخاص بـ Synology Chat متاح.
   - تتضمنه إصدارات OpenClaw المعبأة الحالية بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافته يدويًا من نسخة checkout للمصدر باستخدام الأمر أعلاه.
   - يعرض `openclaw onboard` الآن Synology Chat في قائمة إعداد القنوات نفسها الموجودة في `openclaw channels add`.
   - إعداد غير تفاعلي: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
2. في عمليات التكامل الخاصة بـ Synology Chat:
   - أنشئ webhook واردًا وانسخ عنوان URL الخاص به.
   - أنشئ webhook صادرًا باستخدام رمزك السري.
3. وجّه عنوان URL الخاص بـ webhook الصادر إلى OpenClaw gateway:
   - `https://gateway-host/webhook/synology` افتراضيًا.
   - أو `channels.synology-chat.webhookPath` المخصص لديك.
4. أكمل الإعداد في OpenClaw.
   - بإرشاد: `openclaw onboard`
   - مباشرة: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. أعد تشغيل gateway وأرسل رسالة مباشرة إلى بوت Synology Chat.

تفاصيل مصادقة webhook:

- يقبل OpenClaw رمز webhook الصادر من `body.token`، ثم
  `?token=...`، ثم الترويسات.
- صيغ الترويسات المقبولة:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- تفشل الرموز الفارغة أو المفقودة بشكل مغلق.

إعداد أدنى:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## متغيرات البيئة

بالنسبة إلى الحساب الافتراضي، يمكنك استخدام متغيرات البيئة:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (مفصولة بفواصل)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

تتجاوز قيم الإعدادات متغيرات البيئة.

## سياسة الرسائل المباشرة والتحكم في الوصول

- `dmPolicy: "allowlist"` هو الإعداد الافتراضي الموصى به.
- يقبل `allowedUserIds` قائمة بمعرّفات مستخدمي Synology (أو سلسلة مفصولة بفواصل).
- في وضع `allowlist`، تُعامل قائمة `allowedUserIds` الفارغة على أنها خطأ في الإعداد، ولن يبدأ مسار webhook (استخدم `dmPolicy: "open"` للسماح للجميع).
- `dmPolicy: "open"` يسمح لأي مرسل.
- `dmPolicy: "disabled"` يحظر الرسائل المباشرة.
- يبقى ربط مستلم الرد على `user_id` الرقمي المستقر افتراضيًا. ويعد `channels.synology-chat.dangerouslyAllowNameMatching: true` وضع توافق احتياطي يعيد تمكين البحث باسم المستخدم/الاسم المستعار القابل للتغيير لتسليم الردود.
- تعمل موافقات الاقتران باستخدام:
  - `openclaw pairing list synology-chat`
  - `openclaw pairing approve synology-chat <CODE>`

## التسليم الصادر

استخدم معرّفات مستخدمي Synology Chat الرقمية كأهداف.

أمثلة:

```bash
openclaw message send --channel synology-chat --target 123456 --text "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --text "Hello again"
```

إرسال الوسائط مدعوم عبر تسليم الملفات المستند إلى URL.

## حسابات متعددة

يتم دعم عدة حسابات Synology Chat ضمن `channels.synology-chat.accounts`.
يمكن لكل حساب تجاوز الرمز، وعنوان URL الوارد، ومسار webhook، وسياسة الرسائل المباشرة، والحدود.
تكون جلسات الرسائل المباشرة معزولة لكل حساب ولكل مستخدم، لذلك فإن `user_id` الرقمي نفسه
في حسابين مختلفين من Synology لا يشارك حالة السجل.
أعطِ كل حساب مفعّل قيمة `webhookPath` مختلفة. يرفض OpenClaw الآن المسارات المتطابقة الدقيقة
ويرفض بدء الحسابات المسماة التي ترث فقط مسار webhook مشتركًا في إعدادات الحسابات المتعددة.
إذا كنت تحتاج عمدًا إلى الوراثة القديمة لحساب مسمى، فاضبط
`dangerouslyAllowInheritedWebhookPath: true` على ذلك الحساب أو على `channels.synology-chat`،
لكن المسارات المتطابقة الدقيقة لا تزال تُرفض بشكل مغلق. فضّل المسارات الصريحة لكل حساب.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## ملاحظات الأمان

- احتفظ بسرية `token` وبدّله إذا تسرّب.
- أبقِ `allowInsecureSsl: false` ما لم تكن تثق صراحةً في شهادة NAS محلية موقعة ذاتيًا.
- يتم التحقق من طلبات webhook الواردة بواسطة الرمز وتطبيق حد معدل عليها لكل مرسل.
- تستخدم فحوصات الرمز غير الصالح مقارنة أسرار بزمن ثابت وتفشل بشكل مغلق.
- فضّل `dmPolicy: "allowlist"` في بيئات الإنتاج.
- أبقِ `dangerouslyAllowNameMatching` معطّلًا ما لم تكن تحتاج صراحةً إلى تسليم الردود القديم المستند إلى اسم المستخدم.
- أبقِ `dangerouslyAllowInheritedWebhookPath` معطّلًا ما لم تكن تقبل صراحةً مخاطر التوجيه عبر المسار المشترك في إعداد حسابات متعددة.

## استكشاف الأخطاء وإصلاحها

- `Missing required fields (token, user_id, text)`:
  - حمولة webhook الصادر تفتقد أحد الحقول المطلوبة
  - إذا كان Synology يرسل الرمز في الترويسات، فتأكد من أن gateway/proxy يحافظ على تلك الترويسات
- `Invalid token`:
  - سر webhook الصادر لا يطابق `channels.synology-chat.token`
  - الطلب يصل إلى الحساب أو مسار webhook الخاطئ
  - قام reverse proxy بإزالة ترويسة الرمز قبل أن يصل الطلب إلى OpenClaw
- `Rate limit exceeded`:
  - قد يؤدي عدد كبير جدًا من محاولات الرمز غير الصالح من المصدر نفسه إلى حظر ذلك المصدر مؤقتًا
  - يملك المرسلون الموثقون أيضًا حد معدل رسائل منفصلًا لكل مستخدم
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open.`:
  - تم تمكين `dmPolicy="allowlist"` لكن لم يتم إعداد أي مستخدمين
- `User not authorized`:
  - `user_id` الرقمي للمرسل غير موجود في `allowedUserIds`

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وبوابة الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
