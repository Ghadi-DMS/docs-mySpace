---
read_when:
    - العمل على ميزات قناة Tlon/Urbit
summary: حالة دعم Tlon/Urbit، والقدرات، والتكوين
title: Tlon
x-i18n:
    generated_at: "2026-04-05T12:36:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 289cffb3c1b2d450a5f41e0d67117dfb5c192cec956d82039caac9df9f07496d
    source_path: channels/tlon.md
    workflow: 15
---

# Tlon

Tlon هو تطبيق مراسلة لامركزي مبني على Urbit. يتصل OpenClaw بسفينة Urbit الخاصة بك ويمكنه
الرد على الرسائل الخاصة ورسائل الدردشة الجماعية. تتطلب الردود في المجموعات إشارة @ افتراضيًا ويمكن
تقييدها بشكل إضافي عبر قوائم السماح.

الحالة: plugin مضمّن. الرسائل الخاصة، والإشارات في المجموعات، والردود ضمن السلاسل، وتنسيق النص المنسق،
ورفع الصور مدعومة. أما التفاعلات واستطلاعات الرأي فليست مدعومة بعد.

## plugin المضمّن

يأتي Tlon كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك لا تحتاج الإصدارات المعبأة
العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن Tlon، فثبّته
يدويًا:

التثبيت عبر CLI ‏(سجل npm):

```bash
openclaw plugins install @openclaw/tlon
```

نسخة محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## الإعداد

1. تأكد من أن plugin Tlon متاح.
   - تتضمن إصدارات OpenClaw المعبأة الحالية هذا plugin بالفعل.
   - يمكن للتثبيتات الأقدم/المخصصة إضافته يدويًا باستخدام الأوامر أعلاه.
2. اجمع عنوان URL الخاص بالسفينة ورمز تسجيل الدخول.
3. قم بتكوين `channels.tlon`.
4. أعد تشغيل gateway.
5. أرسل رسالة خاصة إلى bot أو أشر إليه في قناة جماعية.

الحد الأدنى من التكوين (حساب واحد):

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // موصى به: سفينتك، ومسموح بها دائمًا
    },
  },
}
```

## السفن الخاصة/على LAN

افتراضيًا، يمنع OpenClaw أسماء المضيفين ونطاقات IP الخاصة/الداخلية للحماية من SSRF.
إذا كانت سفينتك تعمل على شبكة خاصة (localhost أو عنوان IP على LAN أو اسم مضيف داخلي)،
فيجب عليك الاشتراك صراحةً:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      allowPrivateNetwork: true,
    },
  },
}
```

ينطبق هذا على عناوين URL مثل:

- `http://localhost:8080`
- `http://192.168.x.x:8080`
- `http://my-ship.local:8080`

⚠️ فعّل هذا فقط إذا كنت تثق بشبكتك المحلية. يعطّل هذا الإعداد حمايات SSRF
للطلبات المرسلة إلى عنوان URL الخاص بسفينتك.

## قنوات المجموعات

يكون الاكتشاف التلقائي ممكّنًا افتراضيًا. يمكنك أيضًا تثبيت القنوات يدويًا:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
    },
  },
}
```

تعطيل الاكتشاف التلقائي:

```json5
{
  channels: {
    tlon: {
      autoDiscoverChannels: false,
    },
  },
}
```

## التحكم في الوصول

قائمة السماح للرسائل الخاصة (فارغة = لا يُسمح بأي رسائل خاصة، استخدم `ownerShip` لتدفق الموافقة):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

تفويض المجموعات (مقيّد افتراضيًا):

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

## نظام المالك والموافقة

عيّن سفينة مالك لتلقي طلبات الموافقة عندما يحاول مستخدمون غير مصرح لهم التفاعل:

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

تكون سفينة المالك **مصرحًا لها تلقائيًا في كل مكان** — تتم الموافقة تلقائيًا على دعوات الرسائل الخاصة،
وتُسمح رسائل القنوات دائمًا. لا تحتاج إلى إضافة المالك إلى `dmAllowlist` أو
`defaultAuthorizedShips`.

عند تعيينه، يتلقى المالك إشعارات عبر الرسائل الخاصة بشأن:

- طلبات الرسائل الخاصة من سفن غير موجودة في قائمة السماح
- الإشارات في القنوات من دون تفويض
- طلبات دعوات المجموعات

## إعدادات القبول التلقائي

القبول التلقائي لدعوات الرسائل الخاصة (للسفن الموجودة في dmAllowlist):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

القبول التلقائي لدعوات المجموعات:

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
    },
  },
}
```

## أهداف التسليم (CLI/cron)

استخدم هذه مع `openclaw message send` أو تسليم cron:

- رسالة خاصة: `~sampel-palnet` أو `dm/~sampel-palnet`
- مجموعة: `chat/~host-ship/channel` أو `group:~host-ship/channel`

## Skill المضمّنة

يتضمن plugin Tlon Skill مضمّنة ([`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill))
توفر وصولًا عبر CLI إلى عمليات Tlon:

- **جهات الاتصال**: الحصول على الملفات الشخصية/تحديثها، وسرد جهات الاتصال
- **القنوات**: السرد، والإنشاء، ونشر الرسائل، وجلب السجل
- **المجموعات**: السرد، والإنشاء، وإدارة الأعضاء
- **الرسائل الخاصة**: إرسال الرسائل، والتفاعل مع الرسائل
- **التفاعلات**: إضافة/إزالة تفاعلات emoji إلى المنشورات والرسائل الخاصة
- **الإعدادات**: إدارة أذونات plugin عبر أوامر الشرطة المائلة

تتوفر Skill تلقائيًا عند تثبيت plugin.

## القدرات

| الميزة          | الحالة                                  |
| --------------- | --------------------------------------- |
| الرسائل الخاصة  | ✅ مدعومة                               |
| المجموعات/القنوات | ✅ مدعومة (مقيّدة بالإشارة افتراضيًا) |
| السلاسل         | ✅ مدعومة (الردود التلقائية داخل السلسلة) |
| النص المنسق     | ✅ يتم تحويل Markdown إلى تنسيق Tlon    |
| الصور           | ✅ يتم رفعها إلى تخزين Tlon             |
| التفاعلات       | ✅ عبر [Skill المضمّنة](#bundled-skill) |
| استطلاعات الرأي | ❌ غير مدعومة بعد                       |
| الأوامر الأصلية | ✅ مدعومة (للمالك فقط افتراضيًا)        |

## استكشاف الأخطاء وإصلاحها

شغّل هذا التسلسل أولًا:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

الإخفاقات الشائعة:

- **يتم تجاهل الرسائل الخاصة**: المرسل غير موجود في `dmAllowlist` ولم يتم تكوين `ownerShip` لتدفق الموافقة.
- **يتم تجاهل رسائل المجموعات**: لم يتم اكتشاف القناة أو أن المرسل غير مصرح له.
- **أخطاء الاتصال**: تحقق من أن عنوان URL الخاص بالسفينة قابل للوصول؛ فعّل `allowPrivateNetwork` للسفن المحلية.
- **أخطاء المصادقة**: تحقق من أن رمز تسجيل الدخول الحالي ما زال صالحًا (تتغير الرموز دوريًا).

## مرجع التكوين

التكوين الكامل: [التكوين](/gateway/configuration)

خيارات المزوّد:

- `channels.tlon.enabled`: تمكين/تعطيل بدء تشغيل القناة.
- `channels.tlon.ship`: اسم سفينة Urbit الخاصة بـ bot ‏(مثل `~sampel-palnet`).
- `channels.tlon.url`: عنوان URL الخاص بالسفينة (مثل `https://sampel-palnet.tlon.network`).
- `channels.tlon.code`: رمز تسجيل الدخول للسفينة.
- `channels.tlon.allowPrivateNetwork`: السماح بعناوين localhost/LAN ‏(تجاوز SSRF).
- `channels.tlon.ownerShip`: سفينة المالك لنظام الموافقة (مصرح لها دائمًا).
- `channels.tlon.dmAllowlist`: السفن المسموح لها بإرسال رسائل خاصة (فارغة = لا شيء).
- `channels.tlon.autoAcceptDmInvites`: القبول التلقائي للرسائل الخاصة من السفن الموجودة في قائمة السماح.
- `channels.tlon.autoAcceptGroupInvites`: القبول التلقائي لجميع دعوات المجموعات.
- `channels.tlon.autoDiscoverChannels`: الاكتشاف التلقائي لقنوات المجموعات (الافتراضي: true).
- `channels.tlon.groupChannels`: قنوات مثبتة يدويًا.
- `channels.tlon.defaultAuthorizedShips`: السفن المصرح لها في جميع القنوات.
- `channels.tlon.authorization.channelRules`: قواعد التفويض لكل قناة.
- `channels.tlon.showModelSignature`: إلحاق اسم النموذج بالرسائل.

## ملاحظات

- تتطلب الردود في المجموعات إشارة (مثل `~your-bot-ship`) للرد.
- الردود ضمن السلاسل: إذا كانت الرسالة الواردة داخل سلسلة، يرد OpenClaw داخل السلسلة.
- النص المنسق: يتم تحويل تنسيق Markdown ‏(الغامق، والمائل، والكود، والعناوين، والقوائم) إلى تنسيق Tlon الأصلي.
- الصور: يتم رفع عناوين URL إلى تخزين Tlon وتضمينها ككتل صور.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل الخاصة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وضبط الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
