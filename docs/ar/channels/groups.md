---
read_when:
    - تغيير سلوك الدردشة الجماعية أو تقييد الإشارات
summary: سلوك الدردشة الجماعية عبر الأسطح المختلفة (Discord/iMessage/Matrix/Microsoft Teams/Signal/Slack/Telegram/WhatsApp/Zalo)
title: المجموعات
x-i18n:
    generated_at: "2026-04-05T12:35:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39d066e0542b468c6f8b384b463e2316590ea09a00ecb2065053e1e2ce55bd5f
    source_path: channels/groups.md
    workflow: 15
---

# المجموعات

يتعامل OpenClaw مع الدردشات الجماعية بشكل متسق عبر الأسطح المختلفة: Discord وiMessage وMatrix وMicrosoft Teams وSignal وSlack وTelegram وWhatsApp وZalo.

## مقدمة للمبتدئين (دقيقتان)

يعمل OpenClaw من خلال حسابات المراسلة الخاصة بك. لا يوجد مستخدم بوت منفصل على WhatsApp.
إذا كنت **أنت** في مجموعة، يمكن لـ OpenClaw رؤية تلك المجموعة والرد فيها.

السلوك الافتراضي:

- المجموعات مقيّدة (`groupPolicy: "allowlist"`).
- تتطلب الردود إشارة، إلا إذا عطّلت تقييد الإشارات صراحةً.

المعنى: يمكن للمرسلين المدرجين في قائمة السماح تشغيل OpenClaw عبر الإشارة إليه.

> خلاصة سريعة
>
> - يتم التحكم في **الوصول عبر الرسائل المباشرة** بواسطة `*.allowFrom`.
> - يتم التحكم في **الوصول إلى المجموعات** بواسطة `*.groupPolicy` + قوائم السماح (`*.groups`, `*.groupAllowFrom`).
> - يتم التحكم في **تشغيل الردود** بواسطة تقييد الإشارات (`requireMention`, `/activation`).

التدفق السريع (ما الذي يحدث لرسالة مجموعة):

```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

## رؤية السياق وقوائم السماح

هناك عنصران مختلفان يتحكمان في أمان المجموعات:

- **تفويض التشغيل**: من يمكنه تشغيل الوكيل (`groupPolicy`, `groups`, `groupAllowFrom`, وقوائم السماح الخاصة بالقناة).
- **رؤية السياق**: ما السياق التكميلي الذي يُحقن في النموذج (نص الرد، والاقتباسات، وسجل السلسلة، وبيانات إعادة التوجيه).

بشكل افتراضي، يعطي OpenClaw الأولوية لسلوك الدردشة العادي ويحتفظ بالسياق كما تم استلامه في الغالب. وهذا يعني أن قوائم السماح تحدد أساسًا من يمكنه تشغيل الإجراءات، وليست حدًا عامًا لإخفاء كل مقتطف مقتبس أو تاريخي.

السلوك الحالي خاص بكل قناة:

- تطبق بعض القنوات بالفعل تصفية قائمة على المرسل للسياق التكميلي في مسارات محددة (مثل تهيئة سلاسل Slack وعمليات بحث الرد/السلسلة في Matrix).
- ما تزال قنوات أخرى تمرر سياق الاقتباس/الرد/إعادة التوجيه كما تم استلامه.

اتجاه التقوية (مخطط له):

- `contextVisibility: "all"` (الافتراضي) يبقي السلوك الحالي كما تم استلامه.
- `contextVisibility: "allowlist"` يرشّح السياق التكميلي إلى المرسلين المسموح لهم.
- `contextVisibility: "allowlist_quote"` هو `allowlist` مع استثناء صريح واحد للاقتباس/الرد.

إلى أن يُنفَّذ نموذج التقوية هذا بشكل متسق عبر القنوات، توقّع وجود فروق حسب السطح.

![تدفق رسائل المجموعات](/images/groups-flow.svg)

إذا كنت تريد...

| الهدف                                        | ما الذي يجب ضبطه                                         |
| -------------------------------------------- | -------------------------------------------------------- |
| السماح بكل المجموعات لكن الرد فقط عند @mentions | `groups: { "*": { requireMention: true } }`              |
| تعطيل كل ردود المجموعات                      | `groupPolicy: "disabled"`                                |
| مجموعات محددة فقط                            | `groups: { "<group-id>": { ... } }` (من دون مفتاح `"*"`) |
| أنت وحدك من يمكنه التشغيل في المجموعات       | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |

## مفاتيح الجلسات

- تستخدم جلسات المجموعات مفاتيح جلسات بصيغة `agent:<agentId>:<channel>:group:<id>` (أما الغرف/القنوات فتستخدم `agent:<agentId>:<channel>:channel:<id>`).
- تضيف موضوعات منتدى Telegram اللاحقة `:topic:<threadId>` إلى معرّف المجموعة بحيث يكون لكل موضوع جلسته الخاصة.
- تستخدم الدردشات المباشرة الجلسة الرئيسية (أو جلسة لكل مرسل إذا تم تكوين ذلك).
- يتم تخطي heartbeat لجلسات المجموعات.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## النمط: الرسائل المباشرة الشخصية + المجموعات العامة (وكيل واحد)

نعم — هذا يعمل جيدًا إذا كانت الحركة “الشخصية” لديك هي **الرسائل المباشرة** وكانت الحركة “العامة” لديك هي **المجموعات**.

السبب: في وضع الوكيل الواحد، تصل الرسائل المباشرة عادةً إلى مفتاح الجلسة **الرئيسية** (`agent:main:main`)، بينما تستخدم المجموعات دائمًا مفاتيح جلسات **غير رئيسية** (`agent:main:<channel>:group:<id>`). إذا فعّلت العزل باستخدام `mode: "non-main"`، فستعمل جلسات المجموعات تلك داخل Docker بينما تبقى جلسة الرسائل المباشرة الرئيسية على المضيف.

وهذا يمنحك “عقل” وكيل واحدًا (مساحة عمل + ذاكرة مشتركتان)، ولكن وضعيتين مختلفتين للتنفيذ:

- **الرسائل المباشرة**: أدوات كاملة (على المضيف)
- **المجموعات**: صندوق عزل + أدوات مراسلة مقيّدة (Docker)

> إذا كنت تحتاج إلى مساحات عمل/شخصيات منفصلة تمامًا (“الشخصي” و”العام” يجب ألا يختلطا أبدًا)، فاستخدم وكيلًا ثانيًا + ربطًا. راجع [Multi-Agent Routing](/concepts/multi-agent).

مثال (الرسائل المباشرة على المضيف، والمجموعات داخل صندوق عزل + أدوات مراسلة فقط):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // المجموعات/القنوات غير رئيسية -> داخل صندوق عزل
        scope: "session", // أقوى عزل (حاوية واحدة لكل مجموعة/قناة)
        workspaceAccess: "none",
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // إذا كانت allow غير فارغة، فسيُحظر كل شيء آخر (وdeny ما زال له الأولوية).
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
      },
    },
  },
}
```

هل تريد أن “تستطيع المجموعات رؤية المجلد X فقط” بدلًا من “عدم الوصول إلى المضيف”؟ أبقِ `workspaceAccess: "none"` وقم بتركيب المسارات المسموح بها فقط داخل صندوق العزل:

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

- مفاتيح الإعداد والقيم الافتراضية: [Gateway configuration](/gateway/configuration-reference#agentsdefaultssandbox)
- تصحيح سبب حظر أداة: [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)
- تفاصيل bind mounts: [Sandboxing](/gateway/sandboxing#custom-bind-mounts)

## تسميات العرض

- تستخدم تسميات واجهة المستخدم `displayName` عندما تكون متاحة، وتُنسَّق بالشكل `<channel>:<token>`.
- `#room` محجوزة للغرف/القنوات؛ وتستخدم الدردشات الجماعية `g-<slug>` (أحرف صغيرة، والمسافات تتحول إلى `-`، مع الاحتفاظ بـ `#@+._-`).

## سياسة المجموعات

تحكّم في كيفية التعامل مع رسائل المجموعات/الغرف لكل قناة:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // معرّف مستخدم Telegram رقمي (يمكن للمعالج حل @username)
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

| السياسة      | السلوك                                                       |
| ------------ | ------------------------------------------------------------ |
| `"open"`     | تتجاوز المجموعات قوائم السماح؛ وما يزال تقييد الإشارات مطبقًا. |
| `"disabled"` | حظر كل رسائل المجموعات بالكامل.                              |
| `"allowlist"`| السماح فقط للمجموعات/الغرف التي تطابق قائمة السماح المكوّنة.   |

ملاحظات:

- `groupPolicy` منفصلة عن تقييد الإشارات (الذي يتطلب @mentions).
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: استخدم `groupAllowFrom` (والبديل: `allowFrom` الصريح).
- تنطبق موافقات إقران الرسائل المباشرة (مدخلات المخزن `*-allowFrom`) على الوصول عبر الرسائل المباشرة فقط؛ أما تفويض مرسلي المجموعات فيبقى صريحًا عبر قوائم سماح المجموعات.
- Discord: تستخدم قائمة السماح `channels.discord.guilds.<id>.channels`.
- Slack: تستخدم قائمة السماح `channels.slack.channels`.
- Matrix: تستخدم قائمة السماح `channels.matrix.groups`. يُفضَّل استخدام معرّفات الغرف أو الأسماء المستعارة؛ فالبحث عن أسماء الغرف المنضم إليها يتم على أساس أفضل جهد، وتُتجاهل الأسماء غير المحلولة أثناء التشغيل. استخدم `channels.matrix.groupAllowFrom` لتقييد المرسلين؛ كما أن قوائم السماح `users` لكل غرفة مدعومة أيضًا.
- يتم التحكم في الرسائل المباشرة الجماعية بشكل منفصل (`channels.discord.dm.*`, `channels.slack.dm.*`).
- يمكن أن تطابق قائمة سماح Telegram معرّفات المستخدمين (`"123456789"`, `"telegram:123456789"`, `"tg:123456789"`) أو أسماء المستخدمين (`"@alice"` أو `"alice"`); وتكون البوادئ غير حساسة لحالة الأحرف.
- القيمة الافتراضية هي `groupPolicy: "allowlist"`؛ وإذا كانت قائمة سماح المجموعات فارغة، فستُحظر رسائل المجموعات.
- أمان وقت التشغيل: عندما تكون كتلة الموفر مفقودة بالكامل (`channels.<provider>` غير موجودة)، تعود سياسة المجموعات إلى وضع مغلق افتراضيًا عند الفشل (عادةً `allowlist`) بدلًا من وراثة `channels.defaults.groupPolicy`.

النموذج الذهني السريع (ترتيب التقييم لرسائل المجموعات):

1. `groupPolicy` ‏(open/disabled/allowlist)
2. قوائم سماح المجموعات (`*.groups`, `*.groupAllowFrom`, قائمة السماح الخاصة بالقناة)
3. تقييد الإشارات (`requireMention`, `/activation`)

## تقييد الإشارات (الافتراضي)

تتطلب رسائل المجموعات إشارة ما لم يتم تجاوز ذلك لكل مجموعة على حدة. تعيش القيم الافتراضية لكل نظام فرعي تحت `*.groups."*"`.

يُحتسب الرد على رسالة البوت كإشارة ضمنية (عندما تدعم القناة بيانات الرد الوصفية). ينطبق هذا على Telegram وWhatsApp وSlack وDiscord وMicrosoft Teams.

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

- `mentionPatterns` هي أنماط regex آمنة وغير حساسة لحالة الأحرف؛ ويتم تجاهل الأنماط غير الصالحة وأشكال التكرار المتداخل غير الآمنة.
- الأسطح التي توفّر إشارات صريحة ما تزال تمر؛ والأنماط هي بديل احتياطي.
- تجاوز لكل وكيل: `agents.list[].groupChat.mentionPatterns` (مفيد عندما تشترك عدة وكلاء في مجموعة واحدة).
- لا يُفرض تقييد الإشارات إلا عندما يكون اكتشاف الإشارات ممكنًا (إشارات أصلية أو تكوين `mentionPatterns`).
- تعيش القيم الافتراضية لـ Discord في `channels.discord.guilds."*"` (وقابلة للتجاوز لكل خادم/قناة).
- يُغلَّف سياق سجل المجموعات بشكل موحّد عبر القنوات وهو **معلّق فقط** (الرسائل التي تم تخطيها بسبب تقييد الإشارات)؛ استخدم `messages.groupChat.historyLimit` للقيمة الافتراضية العامة و`channels.<channel>.historyLimit` (أو `channels.<channel>.accounts.*.historyLimit`) للتجاوزات. اضبط القيمة إلى `0` للتعطيل.

## قيود أدوات المجموعات/القنوات (اختياري)

تدعم بعض إعدادات القنوات تقييد الأدوات المتاحة **داخل مجموعة/غرفة/قناة محددة**.

- `tools`: السماح/المنع للأدوات على مستوى المجموعة كلها.
- `toolsBySender`: تجاوزات لكل مرسل داخل المجموعة.
  استخدم بادئات مفاتيح صريحة:
  `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>`, والرمز الشامل `"*"`.
  ما تزال المفاتيح القديمة غير المسبوقة مقبولة وتُطابَق على أنها `id:` فقط.

ترتيب الحل (الأكثر تحديدًا يفوز):

1. مطابقة `toolsBySender` للمجموعة/القناة
2. `tools` للمجموعة/القناة
3. مطابقة `toolsBySender` الافتراضية (`"*"` )
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

- تُطبَّق قيود أدوات المجموعات/القنوات بالإضافة إلى سياسة الأدوات العامة/الخاصة بالوكيل (وما يزال المنع له الأولوية).
- تستخدم بعض القنوات تداخلًا مختلفًا للغرف/القنوات (مثل Discord `guilds.*.channels.*` وSlack `channels.*` وMicrosoft Teams `teams.*.channels.*`).

## قوائم سماح المجموعات

عند تكوين `channels.whatsapp.groups` أو `channels.telegram.groups` أو `channels.imessage.groups`، تعمل المفاتيح كقائمة سماح للمجموعات. استخدم `"*"` للسماح بكل المجموعات مع الاستمرار في ضبط سلوك الإشارات الافتراضي.

التباس شائع: موافقة إقران الرسائل المباشرة ليست هي نفسها تفويض المجموعات.
بالنسبة إلى القنوات التي تدعم إقران الرسائل المباشرة، يفتح مخزن الإقران الرسائل المباشرة فقط. أما أوامر المجموعات فما تزال تتطلب تفويضًا صريحًا لمرسل المجموعة من قوائم السماح في الإعداد، مثل `groupAllowFrom` أو البديل الموثق في الإعداد لتلك القناة.

النوايا الشائعة (نسخ/لصق):

1. تعطيل كل ردود المجموعات

```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } },
}
```

2. السماح فقط بمجموعات محددة (WhatsApp)

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

3. السماح بكل المجموعات لكن مع طلب الإشارة (بشكل صريح)

```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } },
    },
  },
}
```

4. المالك فقط يمكنه التشغيل في المجموعات (WhatsApp)

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

يُحدَّد المالك بواسطة `channels.whatsapp.allowFrom` (أو E.164 الخاص بالبوت نفسه عند عدم ضبطه). أرسل الأمر كرسالة مستقلة. وتتجاهل الأسطح الأخرى حاليًا `/activation`.

## حقول السياق

تضبط الحمولات الواردة من المجموعات ما يلي:

- `ChatType=group`
- `GroupSubject` (إذا كان معروفًا)
- `GroupMembers` (إذا كانوا معروفين)
- `WasMentioned` (نتيجة تقييد الإشارات)
- تتضمن موضوعات منتدى Telegram أيضًا `MessageThreadId` و`IsForum`.

ملاحظات خاصة بالقنوات:

- يمكن لـ BlueBubbles اختياريًا إثراء مشاركي مجموعات macOS غير المسمّين من قاعدة بيانات جهات الاتصال المحلية قبل تعبئة `GroupMembers`. وهذا معطّل افتراضيًا ولا يعمل إلا بعد اجتياز تقييد المجموعات العادي.

يتضمن موجه نظام الوكيل مقدمة عن المجموعات في الدور الأول من جلسة مجموعة جديدة. وهي تذكّر النموذج بالرد مثل إنسان، وتجنب جداول Markdown، وتجنب كتابة سلاسل `\n` الحرفية.

## تفاصيل iMessage

- يُفضَّل استخدام `chat_id:<id>` عند التوجيه أو الإدراج في قائمة السماح.
- عرض الدردشات: `imsg chats --limit 20`.
- تعود الردود الجماعية دائمًا إلى `chat_id` نفسه.

## تفاصيل WhatsApp

راجع [رسائل المجموعات](/channels/group-messages) للاطلاع على السلوك الخاص بـ WhatsApp فقط (حقن السجل وتفاصيل معالجة الإشارات).
