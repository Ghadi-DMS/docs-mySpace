---
read_when:
    - تريد توصيل OpenClaw بـ QQ
    - تحتاج إلى إعداد بيانات اعتماد QQ Bot
    - تريد دعم QQ Bot للمجموعات أو الدردشة الخاصة
summary: إعداد QQ Bot وتكوينه واستخدامه
title: QQ Bot
x-i18n:
    generated_at: "2026-04-05T12:36:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e58fb7b07c59ecbf80a1276368c4a007b45d84e296ed40cffe9845e0953696c
    source_path: channels/qqbot.md
    workflow: 15
---

# QQ Bot

يتصل QQ Bot بـ OpenClaw عبر QQ Bot API الرسمي (بوابة WebSocket). تدعم
الإضافة الدردشة الخاصة C2C، ورسائل @ في المجموعات، ورسائل قنوات guild مع
وسائط غنية (الصور، والصوت، والفيديو، والملفات).

الحالة: plugin مضمّنة. الرسائل الخاصة، ودردشات المجموعات، وقنوات guild،
والوسائط مدعومة. لا يتم دعم التفاعلات والخيوط.

## plugin المضمّنة

تتضمن إصدارات OpenClaw الحالية QQ Bot، لذلك لا تحتاج الإصدارات المجمعة العادية
إلى خطوة `openclaw plugins install` منفصلة.

## الإعداد

1. انتقل إلى [QQ Open Platform](https://q.qq.com/) وامسح رمز QR باستخدام
   تطبيق QQ على هاتفك للتسجيل / تسجيل الدخول.
2. انقر على **Create Bot** لإنشاء QQ bot جديد.
3. اعثر على **AppID** و**AppSecret** في صفحة إعدادات البوت وانسخهما.

> لا يتم تخزين AppSecret كنص عادي — إذا غادرت الصفحة دون حفظه،
> فسيتعين عليك إنشاء واحد جديد.

4. أضف القناة:

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. أعد تشغيل Gateway.

مسارات الإعداد التفاعلي:

```bash
openclaw channels add
openclaw configure --section channels
```

## التكوين

الحد الأدنى من التكوين:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

متغيرات البيئة للحساب الافتراضي:

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

AppSecret من ملف:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

ملاحظات:

- ينطبق الرجوع إلى متغيرات البيئة على حساب QQ Bot الافتراضي فقط.
- يوفّر `openclaw channels add --channel qqbot --token-file ...`
  AppSecret فقط؛ ويجب أن يكون AppID مضبوطًا مسبقًا في التكوين أو في `QQBOT_APP_ID`.
- يقبل `clientSecret` أيضًا إدخال SecretRef، وليس فقط سلسلة نصية عادية.

### إعداد متعدد الحسابات

شغّل عدة حسابات QQ bot ضمن مثيل OpenClaw واحد:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

يشغّل كل حساب اتصال WebSocket خاصًا به ويحافظ على ذاكرة تخزين مؤقت مستقلة
للرمز المميز (معزولة بحسب `appId`).

أضف بوتًا ثانيًا عبر CLI:

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### الصوت (STT / TTS)

يدعم STT وTTS تكوينًا على مستويين مع رجوع بحسب الأولوية:

| الإعداد | خاص بـ plugin         | رجوع الإطار العام              |
| ------- | --------------------- | ------------------------------ |
| STT     | `channels.qqbot.stt`  | `tools.media.audio.models[0]`  |
| TTS     | `channels.qqbot.tts`  | `messages.tts`                 |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
    },
  },
}
```

اضبط `enabled: false` على أي منهما للتعطيل.

يمكن أيضًا ضبط سلوك رفع/تحويل الصوت الصادر عبر
`channels.qqbot.audioFormatPolicy`:

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## التنسيقات المستهدفة

| التنسيق                   | الوصف              |
| ------------------------- | ------------------ |
| `qqbot:c2c:OPENID`        | دردشة خاصة (C2C)   |
| `qqbot:group:GROUP_OPENID` | دردشة جماعية       |
| `qqbot:channel:CHANNEL_ID` | قناة guild         |

> لكل بوت مجموعة OpenID خاصة به للمستخدمين. ولا يمكن
> استخدام OpenID المستلم بواسطة Bot A **لإرسال** رسائل عبر Bot B.

## أوامر الشرطة المائلة

الأوامر المضمنة التي يتم اعتراضها قبل قائمة انتظار الذكاء الاصطناعي:

| الأمر          | الوصف                                 |
| -------------- | ------------------------------------- |
| `/bot-ping`    | اختبار زمن الاستجابة                  |
| `/bot-version` | عرض إصدار إطار OpenClaw              |
| `/bot-help`    | عرض جميع الأوامر                      |
| `/bot-upgrade` | عرض رابط دليل ترقية QQBot            |
| `/bot-logs`    | تصدير سجلات Gateway الحديثة كملف      |

أضف `?` إلى أي أمر للحصول على مساعدة الاستخدام (على سبيل المثال `/bot-upgrade ?`).

## استكشاف الأخطاء وإصلاحها

- **يرد البوت بعبارة "gone to Mars":** بيانات الاعتماد غير مكوّنة أو لم يتم تشغيل Gateway.
- **لا توجد رسائل واردة:** تحقّق من صحة `appId` و`clientSecret`، ومن
  تمكين البوت في QQ Open Platform.
- **لا يزال الإعداد باستخدام `--token-file` يظهر على أنه غير مكوّن:** يضبط `--token-file`
  AppSecret فقط. ما زلت بحاجة إلى `appId` في التكوين أو `QQBOT_APP_ID`.
- **لا تصل الرسائل الاستباقية:** قد يعترض QQ الرسائل التي يبدأها البوت إذا
  لم يتفاعل المستخدم مؤخرًا.
- **لا يتم تفريغ الصوت إلى نص:** تأكد من تكوين STT وإمكانية الوصول إلى المزوّد.
