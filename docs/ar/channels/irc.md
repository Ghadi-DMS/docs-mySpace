---
read_when:
    - تريد توصيل OpenClaw بقنوات IRC أو الرسائل الخاصة
    - أنت تقوم بتكوين قوائم السماح في IRC أو سياسة المجموعات أو تقييد الإشارات
summary: إعداد إضافة IRC، وعناصر التحكم في الوصول، واستكشاف الأخطاء وإصلاحها
title: IRC
x-i18n:
    generated_at: "2026-04-05T12:35:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: fceab2979db72116689c6c774d6736a8a2eee3559e3f3cf8969e673d317edd94
    source_path: channels/irc.md
    workflow: 15
---

# IRC

استخدم IRC عندما تريد OpenClaw في القنوات التقليدية (`#room`) والرسائل الخاصة.
يُشحن IRC كإضافة plugin، لكنه يُكوَّن في التكوين الرئيسي ضمن `channels.irc`.

## البدء السريع

1. فعّل تكوين IRC في `~/.openclaw/openclaw.json`.
2. اضبط على الأقل:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

يفضَّل استخدام خادم IRC خاص لتنسيق البوتات. وإذا كنت تستخدم عمدًا شبكة IRC عامة، فمن الخيارات الشائعة Libera.Chat وOFTC وSnoonet. تجنب القنوات العامة المتوقعة لحركة مرور القناة الخلفية الخاصة بالبوت أو السرب.

3. ابدأ/أعد تشغيل البوابة:

```bash
openclaw gateway run
```

## الإعدادات الأمنية الافتراضية

- القيمة الافتراضية لـ `channels.irc.dmPolicy` هي `"pairing"`.
- القيمة الافتراضية لـ `channels.irc.groupPolicy` هي `"allowlist"`.
- عند استخدام `groupPolicy="allowlist"`، اضبط `channels.irc.groups` لتعريف القنوات المسموح بها.
- استخدم TLS (`channels.irc.tls=true`) ما لم تكن تقبل عمدًا النقل بنص واضح.

## التحكم في الوصول

هناك "بوابتان" منفصلتان لقنوات IRC:

1. **الوصول إلى القناة** (`groupPolicy` + `groups`): ما إذا كان البوت يقبل الرسائل من القناة أصلًا.
2. **وصول المرسل** (`groupAllowFrom` / `groups["#channel"].allowFrom` لكل قناة): من المسموح له بتشغيل البوت داخل تلك القناة.

مفاتيح التكوين:

- قائمة السماح للرسائل الخاصة (وصول مرسل الرسائل الخاصة): `channels.irc.allowFrom`
- قائمة السماح لمرسلي المجموعات (وصول مرسل القناة): `channels.irc.groupAllowFrom`
- عناصر التحكم لكل قناة (القناة + المرسل + قواعد الإشارة): `channels.irc.groups["#channel"]`
- يتيح `channels.irc.groupPolicy="open"` القنوات غير المكوّنة (**مع استمرار تقييدها بالإشارات افتراضيًا**)

يجب أن تستخدم إدخالات قائمة السماح هويات مرسل مستقرة (`nick!user@host`).
مطابقة الاسم المستعار المجرد قابلة للتغيير ولا يتم تمكينها إلا عند `channels.irc.dangerouslyAllowNameMatching: true`.

### مشكلة شائعة: `allowFrom` مخصص للرسائل الخاصة، وليس للقنوات

إذا رأيت سجلات مثل:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

…فهذا يعني أن المرسل غير مسموح له برسائل **المجموعة/القناة**. أصلح ذلك بإحدى الطريقتين:

- ضبط `channels.irc.groupAllowFrom` (عام لكل القنوات)، أو
- ضبط قوائم السماح للمرسلين لكل قناة: `channels.irc.groups["#channel"].allowFrom`

مثال (السماح لأي شخص في `#tuirc-dev` بالتحدث إلى البوت):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": { allowFrom: ["*"] },
      },
    },
  },
}
```

## تشغيل الردود (الإشارات)

حتى إذا كانت القناة مسموحًا بها (عبر `groupPolicy` + `groups`) وكان المرسل مسموحًا به، فإن OpenClaw يطبّق افتراضيًا **تقييدًا بالإشارات** في سياقات المجموعات.

هذا يعني أنك قد ترى سجلات مثل `drop channel … (missing-mention)` ما لم تتضمن الرسالة نمط إشارة يطابق البوت.

لجعل البوت يرد في قناة IRC **من دون الحاجة إلى إشارة**، عطّل تقييد الإشارات لتلك القناة:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

أو للسماح **بكل** قنوات IRC (من دون قائمة سماح لكل قناة) مع الاستمرار في الرد من دون إشارات:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## ملاحظة أمنية (موصى بها للقنوات العامة)

إذا سمحت بـ `allowFrom: ["*"]` في قناة عامة، يمكن لأي شخص إرسال مطالبات إلى البوت.
لتقليل المخاطر، قيّد الأدوات لتلك القناة.

### الأدوات نفسها للجميع في القناة

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### أدوات مختلفة لكل مرسل (المالك يحصل على صلاحيات أكبر)

استخدم `toolsBySender` لتطبيق سياسة أكثر صرامة على `"*"` وسياسة أقل صرامة على اسمك المستعار:

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:eigen": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

ملاحظات:

- يجب أن تستخدم مفاتيح `toolsBySender` البادئة `id:` لقيم هوية مرسل IRC:
  `id:eigen` أو `id:eigen!~eigen@174.127.248.171` لمطابقة أقوى.
- لا تزال المفاتيح القديمة غير المسبوقة ببادئة مقبولة وتُطابَق على أنها `id:` فقط.
- تفوز أول سياسة مرسل مطابقة؛ و`"*"` هي البديل العام.

للمزيد حول الوصول إلى المجموعة مقابل التقييد بالإشارات (وكيفية تفاعلهما)، راجع: [/channels/groups](/channels/groups).

## NickServ

للتعريف باستخدام NickServ بعد الاتصال:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

تسجيل اختياري لمرة واحدة عند الاتصال:

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

عطّل `register` بعد تسجيل الاسم المستعار لتجنب محاولات REGISTER المتكررة.

## متغيرات البيئة

يدعم الحساب الافتراضي ما يلي:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (مفصولة بفواصل)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

## استكشاف الأخطاء وإصلاحها

- إذا اتصل البوت لكنه لا يرد أبدًا في القنوات، فتحقق من `channels.irc.groups` **وأيضًا** مما إذا كان تقييد الإشارات يسقط الرسائل (`missing-mention`). إذا كنت تريده أن يرد من دون تنبيهات، فاضبط `requireMention:false` للقناة.
- إذا فشل تسجيل الدخول، فتحقق من توفر الاسم المستعار وكلمة مرور الخادم.
- إذا فشل TLS على شبكة مخصصة، فتحقق من إعدادات المضيف/المنفذ والشهادة.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الإقران](/channels/pairing) — مصادقة الرسائل الخاصة وتدفق الإقران
- [المجموعات](/channels/groups) — سلوك دردشة المجموعات وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
