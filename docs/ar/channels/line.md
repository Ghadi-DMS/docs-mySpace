---
read_when:
    - تريد توصيل OpenClaw بـ LINE
    - تحتاج إلى إعداد webhook وبيانات الاعتماد الخاصة بـ LINE
    - تريد خيارات رسائل خاصة بـ LINE
summary: إعداد وتكوين واستخدام plugin الخاص بـ LINE Messaging API
title: LINE
x-i18n:
    generated_at: "2026-04-05T12:35:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: b4782b2aa3e8654505d7f1fd6fc112adf125b5010fc84d655d033688ded37414
    source_path: channels/line.md
    workflow: 15
---

# LINE

يتصل LINE بـ OpenClaw عبر LINE Messaging API. يعمل plugin كمستقبل
webhook على gateway ويستخدم channel access token وchannel secret
الخاصين بك للمصادقة.

الحالة: plugin مضمّن. الرسائل المباشرة، ودردشات المجموعات، والوسائط، والمواقع، ورسائل Flex،
ورسائل القوالب، والردود السريعة مدعومة. التفاعلات والخيوط
غير مدعومة.

## Plugin مضمّن

يأتي LINE كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذا فإن
النسخ المجمعة العادية لا تحتاج إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن LINE، فقم بتثبيته
يدويًا:

```bash
openclaw plugins install @openclaw/line
```

نسخة محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## الإعداد

1. أنشئ حساب LINE Developers وافتح Console:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. أنشئ Provider (أو اختر واحدًا) وأضف قناة **Messaging API**.
3. انسخ **Channel access token** و**Channel secret** من إعدادات القناة.
4. فعّل **Use webhook** في إعدادات Messaging API.
5. اضبط عنوان URL الخاص بـ webhook على نقطة نهاية gateway لديك (يتطلب HTTPS):

```
https://gateway-host/line/webhook
```

يستجيب gateway لعملية التحقق من webhook الخاصة بـ LINE ‏(GET) وللأحداث الواردة ‏(POST).
إذا كنت بحاجة إلى مسار مخصص، فاضبط `channels.line.webhookPath` أو
`channels.line.accounts.<id>.webhookPath` وحدّث عنوان URL وفقًا لذلك.

ملاحظة أمنية:

- يعتمد التحقق من توقيع LINE على جسم الطلب (HMAC على الجسم الخام)، لذلك يطبق OpenClaw حدودًا صارمة على حجم جسم ما قبل المصادقة ومهلة زمنية قبل التحقق.
- يعالج OpenClaw أحداث webhook من بايتات الطلب الخام التي تم التحقق منها. يتم تجاهل قيم `req.body` التي حوّلتها برمجيات middleware في الطبقة العليا حفاظًا على سلامة التوقيع.

## التكوين

الحد الأدنى من التكوين:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

متغيرات البيئة (للحساب الافتراضي فقط):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

ملفات token/secret:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

يجب أن يشير `tokenFile` و`secretFile` إلى ملفات عادية. يتم رفض الروابط الرمزية.

حسابات متعددة:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## التحكم في الوصول

تستخدم الرسائل المباشرة pairing افتراضيًا. يحصل المرسلون غير المعروفين على رمز pairing ويتم
تجاهل رسائلهم حتى تتم الموافقة عليها.

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

قوائم السماح والسياسات:

- `channels.line.dmPolicy`: ‏`pairing | allowlist | open | disabled`
- `channels.line.allowFrom`: معرّفات مستخدمي LINE المسموح لهم في الرسائل المباشرة
- `channels.line.groupPolicy`: ‏`allowlist | open | disabled`
- `channels.line.groupAllowFrom`: معرّفات مستخدمي LINE المسموح لهم في المجموعات
- عمليات تجاوز لكل مجموعة: `channels.line.groups.<groupId>.allowFrom`
- ملاحظة وقت التشغيل: إذا كان `channels.line` مفقودًا بالكامل، يعود وقت التشغيل إلى `groupPolicy="allowlist"` لفحوصات المجموعات (حتى إذا كان `channels.defaults.groupPolicy` معيّنًا).

معرّفات LINE حساسة لحالة الأحرف. تبدو المعرّفات الصالحة كما يلي:

- مستخدم: `U` + 32 حرفًا سداسيًا عشريًا
- مجموعة: `C` + 32 حرفًا سداسيًا عشريًا
- غرفة: `R` + 32 حرفًا سداسيًا عشريًا

## سلوك الرسائل

- يتم تقسيم النص إلى أجزاء عند 5000 حرف.
- تتم إزالة تنسيق Markdown؛ وتتحول كتل التعليمات البرمجية والجداول إلى بطاقات Flex
  عندما يكون ذلك ممكنًا.
- يتم تخزين الردود المتدفقة مؤقتًا؛ ويتلقى LINE أجزاء كاملة مع
  رسم متحرك للتحميل أثناء عمل الوكيل.
- يتم تقييد تنزيلات الوسائط بواسطة `channels.line.mediaMaxMb` (الافتراضي 10).

## بيانات القناة (الرسائل الغنية)

استخدم `channelData.line` لإرسال الردود السريعة أو المواقع أو بطاقات Flex أو
رسائل القوالب.

```json5
{
  text: "Here you go",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Office",
        address: "123 Main St",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Status card",
        contents: {
          /* Flex payload */
        },
      },
      templateMessage: {
        type: "confirm",
        text: "Proceed?",
        confirmLabel: "Yes",
        confirmData: "yes",
        cancelLabel: "No",
        cancelData: "no",
      },
    },
  },
}
```

يشحن plugin الخاص بـ LINE أيضًا أمر `/card` لإعدادات رسائل Flex المسبقة:

```
/card info "Welcome" "Thanks for joining!"
```

## دعم ACP

يدعم LINE ربط المحادثات عبر ACP ‏(Agent Communication Protocol):

- `/acp spawn <agent> --bind here` يربط دردشة LINE الحالية بجلسة ACP من دون إنشاء خيط فرعي.
- تعمل ارتباطات ACP المكوّنة وجلسات ACP النشطة المرتبطة بالمحادثة على LINE كما هو الحال في قنوات المحادثة الأخرى.

راجع [ACP agents](/tools/acp-agents) للحصول على التفاصيل.

## الوسائط الصادرة

يدعم plugin الخاص بـ LINE إرسال الصور ومقاطع الفيديو والملفات الصوتية عبر أداة رسائل الوكيل. يتم إرسال الوسائط عبر مسار التسليم الخاص بـ LINE مع معالجة مناسبة للمعاينة والتتبع:

- **الصور**: تُرسل كرسائل صور LINE مع إنشاء معاينة تلقائيًا.
- **مقاطع الفيديو**: تُرسل مع معالجة صريحة للمعاينة ونوع المحتوى.
- **الصوت**: يُرسل كرسائل صوتية LINE.

تعود عمليات إرسال الوسائط العامة إلى مسار الصور فقط الحالي عندما لا يتوفر مسار خاص بـ LINE.

## استكشاف الأخطاء وإصلاحها

- **فشل التحقق من webhook:** تأكد من أن عنوان URL الخاص بـ webhook يستخدم HTTPS وأن
  `channelSecret` يطابق ما في LINE Console.
- **لا توجد أحداث واردة:** تأكد من أن مسار webhook يطابق `channels.line.webhookPath`
  وأن gateway يمكن الوصول إليه من LINE.
- **أخطاء تنزيل الوسائط:** ارفع قيمة `channels.line.mediaMaxMb` إذا تجاوزت الوسائط
  الحد الافتراضي.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق pairing
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
