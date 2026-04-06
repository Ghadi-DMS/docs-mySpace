---
read_when:
    - عند تغيير سلوك الدردشات الجماعية أو تقييد الإشارات
summary: سلوك الدردشات الجماعية عبر الأسطح المختلفة (Discord/iMessage/Matrix/Microsoft Teams/Signal/Slack/Telegram/WhatsApp/Zalo)
title: المجموعات
x-i18n:
    generated_at: "2026-04-06T03:06:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8620de6f7f0b866bf43a307fdbec3399790f09f22a87703704b0522caba80b18
    source_path: channels/groups.md
    workflow: 15
---

# المجموعات

يتعامل OpenClaw مع الدردشات الجماعية بشكل متسق عبر الأسطح المختلفة: Discord وiMessage وMatrix وMicrosoft Teams وSignal وSlack وTelegram وWhatsApp وZalo.

## مقدمة للمبتدئين (دقيقتان)

يعمل OpenClaw من خلال حسابات المراسلة الخاصة بك. لا يوجد مستخدم bot منفصل على WhatsApp.
إذا كنت **أنت** ضمن مجموعة، يمكن لـ OpenClaw رؤية تلك المجموعة والرد فيها.

السلوك الافتراضي:

- المجموعات مقيّدة (`groupPolicy: "allowlist"`).
- تتطلب الردود إشارة ما لم تقم بتعطيل تقييد الإشارات صراحةً.

بمعنى آخر: يمكن للمرسلين المدرجين في قائمة السماح تشغيل OpenClaw عبر الإشارة إليه.

> الخلاصة
>
> - يتم التحكم في **الوصول إلى الرسائل الخاصة** بواسطة `*.allowFrom`.
> - يتم التحكم في **الوصول إلى المجموعات** بواسطة `*.groupPolicy` + قوائم السماح (`*.groups` و`*.groupAllowFrom`).
> - يتم التحكم في **تشغيل الردود** بواسطة تقييد الإشارات (`requireMention` و`/activation`).

التدفق السريع (ما الذي يحدث لرسالة مجموعة):

```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

## إظهار السياق وقوائم السماح

يتضمن أمان المجموعات عنصرَي تحكم مختلفين:

- **تفويض التشغيل**: من يمكنه تشغيل الوكيل (`groupPolicy` و`groups` و`groupAllowFrom` وقوائم السماح الخاصة بكل قناة).
- **إظهار السياق**: ما السياق الإضافي الذي يتم حقنه في النموذج (نص الرد، والاقتباسات، وسجل السلسلة، وبيانات إعادة التوجيه).

بشكل افتراضي، يعطي OpenClaw الأولوية لسلوك الدردشة الطبيعي ويُبقي السياق في الغالب كما تم استلامه. وهذا يعني أن قوائم السماح تحدد أساسًا من يمكنه تشغيل الإجراءات، وليست حدًا شاملًا لإخفاء كل مقتطف مقتبس أو تاريخي.

السلوك الحالي يعتمد على القناة:

- بعض القنوات تطبق بالفعل تصفية قائمة على المرسل للسياق الإضافي في مسارات محددة (على سبيل المثال تهيئة سلاسل Slack، وعمليات البحث عن الردود/السلاسل في Matrix).
- قنوات أخرى ما زالت تمرر سياق الاقتباس/الرد/إعادة التوجيه كما تم استلامه.

اتجاه التحصين (مخطط له):

- `contextVisibility: "all"` (الافتراضي) يُبقي السلوك الحالي كما تم استلامه.
- `contextVisibility: "allowlist"` يفلتر السياق الإضافي ليقتصر على المرسلين المدرجين في قائمة السماح.
- `contextVisibility: "allowlist_quote"` هو `allowlist` مع استثناء صريح واحد للاقتباس/الرد.

إلى أن يتم تنفيذ نموذج التحصين هذا بشكل متسق عبر القنوات، توقّع وجود اختلافات بحسب السطح.

![تدفق رسائل المجموعة](/images/groups-flow.svg)

إذا كنت تريد...

| الهدف | ما الذي يجب ضبطه |
| -------------------------------------------- | ---------------------------------------------------------- |
| السماح بكل المجموعات ولكن الرد فقط عند @mentions | `groups: { "*": { requireMention: true } }` |
| تعطيل جميع الردود في المجموعات | `groupPolicy: "disabled"` |
| مجموعات محددة فقط | `groups: { "<group-id>": { ... } }` (من دون مفتاح `"*"`) |
| أنت فقط يمكنك التشغيل داخل المجموعات | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |

## مفاتيح الجلسات

- تستخدم جلسات المجموعات مفاتيح جلسات بصيغة `agent:<agentId>:<channel>:group:<id>` (وتستخدم الغرف/القنوات `agent:<agentId>:<channel>:channel:<id>`).
- تضيف موضوعات منتديات Telegram اللاحقة `:topic:<threadId>` إلى معرّف المجموعة بحيث يكون لكل موضوع جلسته الخاصة.
- تستخدم الدردشات المباشرة الجلسة الرئيسية (أو جلسة لكل مرسل إذا تم إعداد ذلك).
- يتم تخطي Heartbeats لجلسات المجموعات.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## النمط: الرسائل الخاصة الشخصية + المجموعات العامة (وكيل واحد)

نعم — يعمل هذا جيدًا إذا كانت حركة المرور **الشخصية** لديك هي **الرسائل الخاصة** وكانت حركة المرور **العامة** لديك هي **المجموعات**.

السبب: في وضع الوكيل الواحد، تصل الرسائل الخاصة عادةً إلى مفتاح الجلسة **الرئيسي** (`agent:main:main`)، بينما تستخدم المجموعات دائمًا مفاتيح جلسات **غير رئيسية** (`agent:main:<channel>:group:<id>`). إذا فعّلت العزل باستخدام `mode: "non-main"`، فستعمل جلسات المجموعات داخل Docker بينما تبقى جلسة الرسائل الخاصة الرئيسية على المضيف.

يمنحك هذا “عقل” وكيل واحد (مساحة عمل + ذاكرة مشتركتان)، ولكن بوضعيتَي تنفيذ:

- **الرسائل الخاصة**: أدوات كاملة (على المضيف)
- **المجموعات**: صندوق عزل + أدوات مقيدة (Docker)

> إذا كنت تحتاج إلى مساحات عمل/شخصيات منفصلة تمامًا (بحيث لا يجب أن تختلط “الشخصية” و“العامة” أبدًا)، فاستخدم وكيلًا ثانيًا + bindings. راجع [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent).

مثال (الرسائل الخاصة على المضيف، والمجموعات داخل صندوق العزل + أدوات مراسلة فقط):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // groups/channels are non-main -> sandboxed
        scope: "session", // strongest isolation (one container per group/channel)
        workspaceAccess: "none",
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // If allow is non-empty, everything else is blocked (deny still wins).
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
      },
    },
  },
}
```

هل تريد أن “تتمكن المجموعات من رؤية المجلد X فقط” بدلًا من “عدم الوصول إلى المضيف”؟ أبقِ `workspaceAccess: "none"` كما هو، وقم بربط المسارات المدرجة في قائمة السماح فقط داخل صندوق العزل:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        docker: {
          binds: [
            // hostPath:containerPath:mode
            "/home/user/FriendsShared:/data:ro",
          ],
        },
      },
    },
  },
}
```

ذو صلة:

- مفاتيح الإعداد والقيم الافتراضية: [إعدادات البوابة](/ar/gateway/configuration-reference#agentsdefaultssandbox)
- تصحيح سبب حظر أداة: [Sandbox مقابل Tool Policy مقابل Elevated](/ar/gateway/sandbox-vs-tool-policy-vs-elevated)
- تفاصيل bind mounts: [العزل](/ar/gateway/sandboxing#custom-bind-mounts)

## تسميات العرض

- تستخدم تسميات واجهة المستخدم `displayName` عند توفره، وتُنسق بصيغة `<channel>:<token>`.
- الرمز `#room` محجوز للغرف/القنوات؛ وتستخدم الدردشات الجماعية `g-<slug>` (أحرف صغيرة، وتحويل المسافات إلى `-`، مع الإبقاء على `#@+._-`).

## سياسة المجموعات

تحكم في كيفية التعامل مع رسائل المجموعات/الغرف لكل قناة:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numeric Telegram user id (wizard can resolve @username)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { allow: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { allow: true },
        "#alias:example.org": { allow: true },
      },
    },
  },
}
```

| السياسة | السلوك |
| ------------- | ------------------------------------------------------------ |
| `"open"` | تتجاوز المجموعات قوائم السماح؛ وما زال تقييد الإشارات مطبقًا. |
| `"disabled"` | حظر جميع رسائل المجموعات بالكامل. |
| `"allowlist"` | السماح فقط للمجموعات/الغرف التي تطابق قائمة السماح المكوّنة. |

ملاحظات:

- `groupPolicy` منفصلة عن تقييد الإشارات (الذي يتطلب @mentions).
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: استخدم `groupAllowFrom` (والبديل الاحتياطي: `allowFrom` الصريح).
- تنطبق موافقات اقتران الرسائل الخاصة (إدخالات التخزين `*-allowFrom`) على وصول الرسائل الخاصة فقط؛ أما تفويض مرسل المجموعة فيبقى صريحًا عبر قوائم سماح المجموعات.
- Discord: تستخدم قائمة السماح `channels.discord.guilds.<id>.channels`.
- Slack: تستخدم قائمة السماح `channels.slack.channels`.
- Matrix: تستخدم قائمة السماح `channels.matrix.groups`. يُفضَّل استخدام معرّفات الغرف أو الأسماء المستعارة؛ كما أن البحث عن اسم الغرفة المنضم إليها يتم بأفضل جهد، ويتم تجاهل الأسماء غير المحلولة وقت التشغيل. استخدم `channels.matrix.groupAllowFrom` لتقييد المرسلين؛ كما أن قوائم السماح `users` لكل غرفة مدعومة أيضًا.
- يتم التحكم في Group DMs بشكل منفصل (`channels.discord.dm.*` و`channels.slack.dm.*`).
- يمكن لقائمة سماح Telegram مطابقة معرّفات المستخدمين (`"123456789"` و`"telegram:123456789"` و`"tg:123456789"`) أو أسماء المستخدمين (`"@alice"` أو `"alice"`); prefixes are case-insensitive.
- القيمة الافتراضية هي `groupPolicy: "allowlist"`؛ وإذا كانت قائمة سماح مجموعاتك فارغة، فسيتم حظر رسائل المجموعات.
- أمان وقت التشغيل: عندما تكون كتلة المزود مفقودة بالكامل (`channels.<provider>` غير موجودة)، تعود سياسة المجموعات إلى وضع آمن مغلق افتراضيًا (عادةً `allowlist`) بدلًا من وراثة `channels.defaults.groupPolicy`.

نموذج ذهني سريع (ترتيب التقييم لرسائل المجموعات):

1. `groupPolicy` ‏(open/disabled/allowlist)
2. قوائم سماح المجموعات (`*.groups` و`*.groupAllowFrom` وقائمة السماح الخاصة بالقناة)
3. تقييد الإشارات (`requireMention` و`/activation`)

## تقييد الإشارات (الافتراضي)

تتطلب رسائل المجموعات إشارة ما لم يتم تجاوز ذلك لكل مجموعة. توجد القيم الافتراضية لكل نظام فرعي ضمن `*.groups."*"`.

يُحسب الرد على رسالة bot كإشارة ضمنية (عندما تدعم القناة بيانات تعريف الرد). ينطبق هذا على Telegram وWhatsApp وSlack وDiscord وMicrosoft Teams.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    ],
  },
}
```

ملاحظات:

- إن `mentionPatterns` هي أنماط regex آمنة وغير حساسة لحالة الأحرف؛ ويتم تجاهل الأنماط غير الصالحة وأشكال التكرار المتداخل غير الآمنة.
- الأسطح التي توفر إشارات صريحة ستظل تمر؛ وهذه الأنماط مجرد بديل احتياطي.
- تجاوز لكل وكيل: `agents.list[].groupChat.mentionPatterns` (وهذا مفيد عندما يشارك عدة وكلاء مجموعة واحدة).
- لا يتم فرض تقييد الإشارات إلا عندما يكون اكتشاف الإشارات ممكنًا (إشارات أصلية أو عند تكوين `mentionPatterns`).
- توجد القيم الافتراضية لـ Discord ضمن `channels.discord.guilds."*"` (وقابلة للتجاوز لكل guild/channel).
- يتم تغليف سياق سجل المجموعة بشكل موحّد عبر القنوات ويكون **pending-only** (أي الرسائل التي تم تخطيها بسبب تقييد الإشارات)؛ استخدم `messages.groupChat.historyLimit` للقيمة الافتراضية العامة و`channels.<channel>.historyLimit` (أو `channels.<channel>.accounts.*.historyLimit`) للتجاوزات. اضبطها على `0` للتعطيل.

## قيود أدوات المجموعة/القناة (اختياري)

بعض إعدادات القنوات تدعم تقييد الأدوات المتاحة **داخل مجموعة/غرفة/قناة محددة**.

- `tools`: السماح/منع الأدوات للمجموعة كلها.
- `toolsBySender`: تجاوزات لكل مرسل داخل المجموعة.
  استخدم بادئات المفاتيح الصريحة:
  `id:<senderId>` و`e164:<phone>` و`username:<handle>` و`name:<displayName>` ورمز `"*"` العام.
  لا تزال المفاتيح القديمة غير المسبوقة ببادئة مقبولة وتتم مطابقتها على أنها `id:` فقط.

ترتيب الحل (الأكثر تحديدًا يفوز):

1. مطابقة `toolsBySender` للمجموعة/القناة
2. `tools` للمجموعة/القناة
3. مطابقة `toolsBySender` الافتراضية (`"*"`).
4. `tools` الافتراضية (`"*"`)

مثال (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

ملاحظات:

- يتم تطبيق قيود أدوات المجموعة/القناة بالإضافة إلى سياسة الأدوات العامة/الخاصة بالوكيل (ويظل المنع هو الغالب).
- تستخدم بعض القنوات تعشيقًا مختلفًا للغرف/القنوات (مثل Discord `guilds.*.channels.*` وSlack `channels.*` وMicrosoft Teams `teams.*.channels.*`).

## قوائم سماح المجموعات

عند تكوين `channels.whatsapp.groups` أو `channels.telegram.groups` أو `channels.imessage.groups`، تعمل المفاتيح كقائمة سماح للمجموعات. استخدم `"*"` للسماح بكل المجموعات مع الاستمرار في ضبط سلوك الإشارات الافتراضي.

التباس شائع: موافقة اقتران الرسائل الخاصة ليست هي نفسها تفويض المجموعة.
بالنسبة إلى القنوات التي تدعم اقتران الرسائل الخاصة، فإن مخزن الاقتران يفتح الرسائل الخاصة فقط. أما أوامر المجموعات فتظل تتطلب تفويضًا صريحًا لمرسل المجموعة من قوائم السماح في الإعدادات مثل `groupAllowFrom` أو البديل الاحتياطي الموثق لتلك القناة.

النيات الشائعة (نسخ/لصق):

1. تعطيل جميع الردود في المجموعات

```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } },
}
```

2. السماح بمجموعات محددة فقط (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "123@g.us": { requireMention: true },
        "456@g.us": { requireMention: false },
      },
    },
  },
}
```

3. السماح بكل المجموعات ولكن اشتراط الإشارة (بشكل صريح)

```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } },
    },
  },
}
```

4. يمكن للمالك فقط التشغيل داخل المجموعات (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## التفعيل (للمالك فقط)

يمكن لمالكي المجموعات تبديل التفعيل لكل مجموعة:

- `/activation mention`
- `/activation always`

يتم تحديد المالك بواسطة `channels.whatsapp.allowFrom` (أو E.164 الخاص بالـ bot نفسه إذا لم يتم تعيينه). أرسل الأمر كرسالة مستقلة. تتجاهل الأسطح الأخرى حاليًا `/activation`.

## حقول السياق

تضبط الحمولات الواردة من المجموعات ما يلي:

- `ChatType=group`
- `GroupSubject` (إذا كان معروفًا)
- `GroupMembers` (إذا كانوا معروفين)
- `WasMentioned` (نتيجة تقييد الإشارات)
- تتضمن موضوعات منتديات Telegram أيضًا `MessageThreadId` و`IsForum`.

ملاحظات خاصة بالقنوات:

- يمكن لـ BlueBubbles اختياريًا إثراء المشاركين غير المسمّين في مجموعات macOS من قاعدة بيانات جهات الاتصال المحلية قبل تعبئة `GroupMembers`. يكون هذا معطّلًا افتراضيًا ولا يعمل إلا بعد اجتياز تقييد المجموعة العادي.

يتضمن system prompt الخاص بالوكيل مقدمة خاصة بالمجموعة في الدور الأول من جلسة مجموعة جديدة. وهي تذكّر النموذج بأن يرد مثل إنسان، وأن يتجنب جداول Markdown، وأن يقلل الأسطر الفارغة ويتبع التباعد الطبيعي للدردشة، وأن يتجنب كتابة تسلسلات `\n` الحرفية.

## تفاصيل iMessage

- يُفضَّل استخدام `chat_id:<id>` عند التوجيه أو الإدراج في قائمة السماح.
- لعرض الدردشات: `imsg chats --limit 20`.
- تعود ردود المجموعات دائمًا إلى `chat_id` نفسه.

## تفاصيل WhatsApp

راجع [رسائل المجموعات](/ar/channels/group-messages) لمعرفة السلوك الخاص بـ WhatsApp فقط (حقن السجل، وتفاصيل التعامل مع الإشارات).
