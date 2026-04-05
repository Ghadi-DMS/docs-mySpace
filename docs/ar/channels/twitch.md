---
read_when:
    - إعداد تكامل دردشة Twitch لـ OpenClaw
summary: تكوين وإعداد روبوت دردشة Twitch
title: Twitch
x-i18n:
    generated_at: "2026-04-05T12:37:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47af9fb6edb1f462c5919850ee9d05e500a1914ddd0d64a41608fbe960e77cd6
    source_path: channels/twitch.md
    workflow: 15
---

# Twitch

دعم دردشة Twitch عبر اتصال IRC. يتصل OpenClaw كمستخدم Twitch (حساب روبوت) لاستقبال الرسائل وإرسالها في القنوات.

## Plugin مضمّن

يأتي Twitch كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذا فإن
النسخ المجمعة العادية لا تحتاج إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن Twitch، فقم
بتثبيته يدويًا:

التثبيت عبر CLI (سجل npm):

```bash
openclaw plugins install @openclaw/twitch
```

نسخة محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/twitch-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## إعداد سريع (للمبتدئين)

1. تأكد من أن plugin الخاص بـ Twitch متاح.
   - تتضمنه إصدارات OpenClaw المجمعة الحالية بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافته يدويًا بالأوامر أعلاه.
2. أنشئ حساب Twitch مخصصًا للروبوت (أو استخدم حسابًا موجودًا).
3. أنشئ بيانات الاعتماد: [Twitch Token Generator](https://twitchtokengenerator.com/)
   - اختر **Bot Token**
   - تحقق من تحديد النطاقين `chat:read` و`chat:write`
   - انسخ **Client ID** و**Access Token**
4. اعثر على معرّف مستخدم Twitch الخاص بك: [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/)
5. اضبط الرمز المميز:
   - متغير البيئة: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (للحساب الافتراضي فقط)
   - أو التكوين: `channels.twitch.accessToken`
   - إذا تم تعيين الاثنين، تكون أولوية التكوين أعلى (والرجوع إلى env للحساب الافتراضي فقط).
6. ابدأ gateway.

**⚠️ مهم:** أضف عنصر تحكم في الوصول (`allowFrom` أو `allowedRoles`) لمنع المستخدمين غير المصرح لهم من تشغيل الروبوت. تكون `requireMention` مضبوطة افتراضيًا على `true`.

الحد الأدنى من التكوين:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Bot's Twitch account
      accessToken: "oauth:abc123...", // OAuth Access Token (or use OPENCLAW_TWITCH_ACCESS_TOKEN env var)
      clientId: "xyz789...", // Client ID from Token Generator
      channel: "vevisk", // Which Twitch channel's chat to join (required)
      allowFrom: ["123456789"], // (recommended) Your Twitch user ID only - get it from https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/
    },
  },
}
```

## ما هو

- قناة Twitch مملوكة لـ Gateway.
- توجيه حتمي: تعود الردود دائمًا إلى Twitch.
- يُطابق كل حساب مفتاح جلسة معزولًا `agent:<agentId>:twitch:<accountName>`.
- `username` هو حساب الروبوت (الذي يصادق)، بينما `channel` هي غرفة الدردشة المطلوب الانضمام إليها.

## الإعداد (تفصيلي)

### إنشاء بيانات الاعتماد

استخدم [Twitch Token Generator](https://twitchtokengenerator.com/):

- اختر **Bot Token**
- تحقق من تحديد النطاقين `chat:read` و`chat:write`
- انسخ **Client ID** و**Access Token**

لا حاجة إلى تسجيل تطبيق يدويًا. تنتهي صلاحية الرموز المميزة بعد عدة ساعات.

### ضبط الروبوت

**متغير البيئة (للحساب الافتراضي فقط):**

```bash
OPENCLAW_TWITCH_ACCESS_TOKEN=oauth:abc123...
```

**أو التكوين:**

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
    },
  },
}
```

إذا تم تعيين env والتكوين معًا، تكون أولوية التكوين أعلى.

### التحكم في الوصول (موصى به)

```json5
{
  channels: {
    twitch: {
      allowFrom: ["123456789"], // (recommended) Your Twitch user ID only
    },
  },
}
```

فضّل `allowFrom` للحصول على قائمة سماح صارمة. استخدم `allowedRoles` بدلًا من ذلك إذا كنت تريد وصولًا قائمًا على الأدوار.

**الأدوار المتاحة:** `"moderator"` و`"owner"` و`"vip"` و`"subscriber"` و`"all"`.

**لماذا معرّفات المستخدمين؟** يمكن أن تتغير أسماء المستخدمين، مما يسمح بانتحال الهوية. أما معرّفات المستخدمين فهي دائمة.

اعثر على معرّف مستخدم Twitch الخاص بك: [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) (تحويل اسم مستخدم Twitch إلى معرّف)

## تحديث الرمز المميز (اختياري)

لا يمكن تحديث الرموز المميزة من [Twitch Token Generator](https://twitchtokengenerator.com/) تلقائيًا — أعد إنشاءها عند انتهاء صلاحيتها.

للحصول على تحديث تلقائي للرمز المميز، أنشئ تطبيق Twitch خاصًا بك في [Twitch Developer Console](https://dev.twitch.tv/console) وأضف إلى التكوين:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

يحدّث الروبوت الرموز المميزة تلقائيًا قبل انتهاء صلاحيتها ويسجل أحداث التحديث.

## دعم الحسابات المتعددة

استخدم `channels.twitch.accounts` مع رموز مميزة لكل حساب. راجع [`gateway/configuration`](/gateway/configuration) للنمط المشترك.

مثال (حساب روبوت واحد في قناتين):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "vevisk",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

**ملاحظة:** يحتاج كل حساب إلى رمزه المميز الخاص به (رمز واحد لكل قناة).

## التحكم في الوصول

### قيود قائمة على الأدوار

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator", "vip"],
        },
      },
    },
  },
}
```

### قائمة سماح حسب معرّف المستخدم (الأكثر أمانًا)

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowFrom: ["123456789", "987654321"],
        },
      },
    },
  },
}
```

### وصول قائم على الأدوار (بديل)

`allowFrom` هي قائمة سماح صارمة. عند تعيينها، يُسمح فقط لمعرّفات المستخدمين تلك.
إذا كنت تريد وصولًا قائمًا على الأدوار، فاترك `allowFrom` غير معيّنة واضبط `allowedRoles` بدلًا من ذلك:

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

### تعطيل شرط @mention

افتراضيًا، تكون `requireMention` هي `true`. لتعطيلها والرد على جميع الرسائل:

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          requireMention: false,
        },
      },
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

أولًا، شغّل أوامر التشخيص:

```bash
openclaw doctor
openclaw channels status --probe
```

### الروبوت لا يرد على الرسائل

**تحقق من التحكم في الوصول:** تأكد من أن معرّف المستخدم الخاص بك موجود في `allowFrom`، أو أزل
`allowFrom` مؤقتًا واضبط `allowedRoles: ["all"]` للاختبار.

**تحقق من وجود الروبوت في القناة:** يجب أن ينضم الروبوت إلى القناة المحددة في `channel`.

### مشكلات الرمز المميز

**"Failed to connect" أو أخطاء المصادقة:**

- تحقق من أن `accessToken` هو قيمة OAuth access token (ويبدأ عادةً بالبادئة `oauth:`)
- تحقق من أن الرمز المميز يملك النطاقين `chat:read` و`chat:write`
- إذا كنت تستخدم تحديث الرمز المميز، فتأكد من تعيين `clientSecret` و`refreshToken`

### تحديث الرمز المميز لا يعمل

**تحقق من السجلات بحثًا عن أحداث التحديث:**

```
Using env token source for mybot
Access token refreshed for user 123456 (expires in 14400s)
```

إذا رأيت "token refresh disabled (no refresh token)":

- تأكد من توفير `clientSecret`
- تأكد من توفير `refreshToken`

## التكوين

**تكوين الحساب:**

- `username` - اسم مستخدم الروبوت
- `accessToken` - OAuth access token مع `chat:read` و`chat:write`
- `clientId` - Twitch Client ID (من Token Generator أو من تطبيقك)
- `channel` - القناة المطلوب الانضمام إليها (مطلوب)
- `enabled` - تمكين هذا الحساب (الافتراضي: `true`)
- `clientSecret` - اختياري: لتحديث الرمز المميز تلقائيًا
- `refreshToken` - اختياري: لتحديث الرمز المميز تلقائيًا
- `expiresIn` - انتهاء صلاحية الرمز المميز بالثواني
- `obtainmentTimestamp` - الطابع الزمني للحصول على الرمز المميز
- `allowFrom` - قائمة سماح معرّفات المستخدمين
- `allowedRoles` - التحكم في الوصول القائم على الأدوار (`"moderator" | "owner" | "vip" | "subscriber" | "all"`)
- `requireMention` - طلب @mention (الافتراضي: `true`)

**خيارات المزوّد:**

- `channels.twitch.enabled` - تمكين/تعطيل بدء تشغيل القناة
- `channels.twitch.username` - اسم مستخدم الروبوت (تكوين مبسط لحساب واحد)
- `channels.twitch.accessToken` - OAuth access token (تكوين مبسط لحساب واحد)
- `channels.twitch.clientId` - Twitch Client ID (تكوين مبسط لحساب واحد)
- `channels.twitch.channel` - القناة المطلوب الانضمام إليها (تكوين مبسط لحساب واحد)
- `channels.twitch.accounts.<accountName>` - تكوين متعدد الحسابات (جميع حقول الحساب أعلاه)

مثال كامل:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      allowedRoles: ["moderator", "vip"],
      accounts: {
        default: {
          username: "mybot",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "your_channel",
          enabled: true,
          clientSecret: "secret123...",
          refreshToken: "refresh456...",
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowFrom: ["123456789", "987654321"],
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## إجراءات الأداة

يمكن للوكيل استدعاء `twitch` مع الإجراء:

- `send` - إرسال رسالة إلى قناة

مثال:

```json5
{
  action: "twitch",
  params: {
    message: "Hello Twitch!",
    to: "#mychannel",
  },
}
```

## السلامة والعمليات

- **تعامل مع الرموز المميزة كأنها كلمات مرور** - لا تُدرج الرموز المميزة في git أبدًا
- **استخدم تحديث الرمز المميز التلقائي** للروبوتات طويلة التشغيل
- **استخدم قوائم سماح معرّفات المستخدمين** بدلًا من أسماء المستخدمين للتحكم في الوصول
- **راقب السجلات** لأحداث تحديث الرموز المميزة وحالة الاتصال
- **قلّل نطاقات الرموز المميزة لأدنى حد** - اطلب فقط `chat:read` و`chat:write`
- **إذا علقت**: أعد تشغيل gateway بعد التأكد من عدم وجود عملية أخرى تملك الجلسة

## الحدود

- **500 حرف** لكل رسالة (مع تجزئة تلقائية عند حدود الكلمات)
- تتم إزالة Markdown قبل التجزئة
- لا يوجد تحديد لمعدل الطلبات (يستخدم حدود المعدل المضمنة في Twitch)

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق pairing
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
