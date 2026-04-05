---
read_when: You want multiple isolated agents (workspaces + auth) in one gateway process.
status: active
summary: 'التوجيه متعدد الوكلاء: وكلاء معزولون، وحسابات القنوات، والارتباطات'
title: التوجيه متعدد الوكلاء
x-i18n:
    generated_at: "2026-04-05T12:41:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7e8bc48f229d01aa793ca4137e5a59f2a5ceb0ba65841710aaf69f53a672be60
    source_path: concepts/multi-agent.md
    workflow: 15
---

# التوجيه متعدد الوكلاء

الهدف: عدة وكلاء _معزولون_ (مساحة عمل + `agentDir` + جلسات منفصلة)، بالإضافة إلى عدة حسابات قنوات (مثل حسابي WhatsApp) ضمن Gateway واحد قيد التشغيل. يتم توجيه الوارد إلى وكيل عبر الارتباطات.

## ما المقصود بـ "وكيل واحد"؟

**الوكيل** هو عقل محدد النطاق بالكامل وله ما يلي:

- **مساحة عمل** (ملفات، وAGENTS.md/SOUL.md/USER.md، وملاحظات محلية، وقواعد الشخصية).
- **دليل الحالة** (`agentDir`) لملفات تعريف المصادقة، وسجل النماذج، وتكوين كل وكيل.
- **مخزن الجلسات** (سجل الدردشة + حالة التوجيه) ضمن `~/.openclaw/agents/<agentId>/sessions`.

ملفات تعريف المصادقة هي **لكل وكيل**. ويقرأ كل وكيل من ملفه الخاص:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

يعد `sessions_history` أيضًا المسار الأكثر أمانًا للاسترجاع عبر الجلسات هنا: فهو يعيد
عرضًا محدودًا ومنقحًا، وليس تفريغًا خامًا للنصوص. ويزيل استرجاع المساعد
وسوم التفكير، وبنية `<relevant-memories>`، وحمولات XML النصية الصريحة لاستدعاءات الأدوات
(بما في ذلك `<tool_call>...</tool_call>`،
و`<function_call>...</function_call>`،
و`<tool_calls>...</tool_calls>`،
و`<function_calls>...</function_calls>`،
وكتل استدعاء الأدوات المقتطعة)، وبنية استدعاء الأدوات المخفّضة،
ورموز التحكم بالنموذج المسرّبة بتنسيق ASCII/العرض الكامل،
وXML المشوه لاستدعاء أدوات MiniMax قبل التنقيح/الاقتطاع.

لا تتم مشاركة بيانات اعتماد الوكيل الرئيسي **تلقائيًا**. لا تعِد استخدام `agentDir`
بين الوكلاء مطلقًا (فهذا يسبب تعارضات في المصادقة/الجلسات). وإذا كنت تريد مشاركة بيانات الاعتماد،
فانسخ `auth-profiles.json` إلى `agentDir` الخاص بالوكيل الآخر.

يتم تحميل Skills من مساحة عمل كل وكيل بالإضافة إلى الجذور المشتركة مثل
`~/.openclaw/skills`، ثم تُرشَّح بحسب قائمة السماح الفعالة لمهارات الوكيل عند
تكوينها. استخدم `agents.defaults.skills` كأساس مشترك و
`agents.list[].skills` كبديل لكل وكيل. راجع
[Skills: لكل وكيل مقابل المشتركة](/tools/skills#per-agent-vs-shared-skills) و
[Skills: قوائم السماح لمهارات الوكيل](/tools/skills#agent-skill-allowlists).

يمكن لـ Gateway استضافة **وكيل واحد** (الافتراضي) أو **عدة وكلاء** جنبًا إلى جنب.

**ملاحظة مساحة العمل:** مساحة عمل كل وكيل هي **cwd الافتراضي**، وليست
sandbox صارمًا. تُحل المسارات النسبية داخل مساحة العمل، لكن يمكن للمسارات المطلقة
الوصول إلى مواقع أخرى على المضيف ما لم يتم تمكين sandboxing. راجع
[Sandboxing](/gateway/sandboxing).

## المسارات (خريطة سريعة)

- التكوين: `~/.openclaw/openclaw.json` (أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة: `~/.openclaw` (أو `OPENCLAW_STATE_DIR`)
- مساحة العمل: `~/.openclaw/workspace` (أو `~/.openclaw/workspace-<agentId>`)
- دليل الوكيل: `~/.openclaw/agents/<agentId>/agent` (أو `agents.list[].agentDir`)
- الجلسات: `~/.openclaw/agents/<agentId>/sessions`

### وضع الوكيل الواحد (الافتراضي)

إذا لم تفعل شيئًا، فسيشغّل OpenClaw وكيلًا واحدًا:

- تكون قيمة `agentId` الافتراضية هي **`main`**.
- تُفهرس الجلسات على شكل `agent:main:<mainKey>`.
- تكون مساحة العمل الافتراضية هي `~/.openclaw/workspace` (أو `~/.openclaw/workspace-<profile>` عند تعيين `OPENCLAW_PROFILE`).
- تكون الحالة الافتراضية هي `~/.openclaw/agents/main/agent`.

## مساعد الوكيل

استخدم معالج الوكلاء لإضافة وكيل جديد معزول:

```bash
openclaw agents add work
```

ثم أضف `bindings` (أو دع المعالج يفعل ذلك) لتوجيه الرسائل الواردة.

تحقق باستخدام:

```bash
openclaw agents list --bindings
```

## البدء السريع

<Steps>
  <Step title="أنشئ مساحة عمل كل وكيل">

استخدم المعالج أو أنشئ مساحات العمل يدويًا:

```bash
openclaw agents add coding
openclaw agents add social
```

يحصل كل وكيل على مساحة العمل الخاصة به مع `SOUL.md` و`AGENTS.md` و`USER.md` الاختيارية، بالإضافة إلى `agentDir` مخصص ومخزن جلسات ضمن `~/.openclaw/agents/<agentId>`.

  </Step>

  <Step title="أنشئ حسابات القنوات">

أنشئ حسابًا واحدًا لكل وكيل على القنوات التي تفضلها:

- Discord: bot واحد لكل وكيل، فعّل Message Content Intent، وانسخ كل رمز مميز.
- Telegram: bot واحد لكل وكيل عبر BotFather، وانسخ كل رمز مميز.
- WhatsApp: اربط كل رقم هاتف لكل حساب.

```bash
openclaw channels login --channel whatsapp --account work
```

راجع أدلة القنوات: [Discord](/channels/discord) و[Telegram](/channels/telegram) و[WhatsApp](/channels/whatsapp).

  </Step>

  <Step title="أضف الوكلاء والحسابات والارتباطات">

أضف الوكلاء تحت `agents.list`، وحسابات القنوات تحت `channels.<channel>.accounts`، ثم صِل بينها باستخدام `bindings` (الأمثلة أدناه).

  </Step>

  <Step title="أعد التشغيل وتحقق">

```bash
openclaw gateway restart
openclaw agents list --bindings
openclaw channels status --probe
```

  </Step>
</Steps>

## عدة وكلاء = عدة أشخاص، عدة شخصيات

مع **عدة وكلاء**، يصبح كل `agentId` **شخصية معزولة بالكامل**:

- **أرقام هواتف/حسابات مختلفة** (لكل `accountId` في القناة).
- **شخصيات مختلفة** (لكل وكيل ملفات مساحة عمل مثل `AGENTS.md` و`SOUL.md`).
- **مصادقة + جلسات منفصلة** (من دون تداخل إلا إذا تم تمكين ذلك صراحةً).

وهذا يتيح لـ **عدة أشخاص** مشاركة خادم Gateway واحد مع إبقاء "عقول" الذكاء الاصطناعي والبيانات الخاصة بهم معزولة.

## بحث ذاكرة QMD عبر الوكلاء

إذا كان ينبغي لوكيل ما أن يبحث في نصوص جلسات QMD الخاصة بوكيل آخر، فأضف
مجموعات إضافية تحت `agents.list[].memorySearch.qmd.extraCollections`.
استخدم `agents.defaults.memorySearch.qmd.extraCollections` فقط عندما
يجب أن يرث كل وكيل مجموعات النصوص المشتركة نفسها.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
      memorySearch: {
        qmd: {
          extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
        },
      },
    },
    list: [
      {
        id: "main",
        workspace: "~/workspaces/main",
        memorySearch: {
          qmd: {
            extraCollections: [{ path: "notes" }], // يُحل داخل مساحة العمل -> مجموعة باسم "notes-main"
          },
        },
      },
      { id: "family", workspace: "~/workspaces/family" },
    ],
  },
  memory: {
    backend: "qmd",
    qmd: { includeDefaultMemory: false },
  },
}
```

يمكن مشاركة مسار المجموعة الإضافية بين الوكلاء، لكن اسم المجموعة
يبقى صريحًا عندما يكون المسار خارج مساحة عمل الوكيل. وتبقى المسارات داخل
مساحة العمل ضمن نطاق الوكيل بحيث يحتفظ كل وكيل بمجموعة البحث في النصوص الخاصة به.

## رقم WhatsApp واحد، عدة أشخاص (تقسيم الرسائل الخاصة)

يمكنك توجيه **رسائل WhatsApp الخاصة المختلفة** إلى وكلاء مختلفين مع البقاء على **حساب WhatsApp واحد**. طابق على E.164 للمرسل (مثل `+15551234567`) باستخدام `peer.kind: "direct"`. ستظل الردود تخرج من رقم WhatsApp نفسه (ولا توجد هوية مرسل لكل وكيل).

تفصيل مهم: تنهار المحادثات المباشرة إلى **مفتاح الجلسة الرئيسي** للوكيل، لذا يتطلب العزل الحقيقي **وكيلًا واحدًا لكل شخص**.

مثال:

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

ملاحظات:

- التحكم في الوصول إلى الرسائل الخاصة **عام لكل حساب WhatsApp** (الاقتران/قائمة السماح)، وليس لكل وكيل.
- بالنسبة إلى المجموعات المشتركة، اربط المجموعة بوكيل واحد أو استخدم [Broadcast groups](/channels/broadcast-groups).

## قواعد التوجيه (كيف تختار الرسائل وكيلًا)

الارتباطات **حتمية** و**الأكثر تحديدًا يفوز**:

1. مطابقة `peer` ‏(معرّف الدردشة الخاصة/المجموعة/القناة الدقيق)
2. مطابقة `parentPeer` ‏(وراثة السلسلة)
3. `guildId + roles` ‏(توجيه الأدوار في Discord)
4. `guildId` ‏(Discord)
5. `teamId` ‏(Slack)
6. مطابقة `accountId` لقناة
7. مطابقة على مستوى القناة (`accountId: "*"`)
8. الرجوع إلى الوكيل الافتراضي (`agents.list[].default`، وإلا أول إدخال في القائمة، الافتراضي: `main`)

إذا طابقت عدة ارتباطات في المستوى نفسه، يفوز أول ارتباط حسب ترتيب التكوين.
إذا ضبط ارتباط ما عدة حقول مطابقة (مثل `peer` + `guildId`)، فكل الحقول المحددة مطلوبة (دلالات `AND`).

تفصيل مهم حول نطاق الحساب:

- الارتباط الذي يحذف `accountId` يطابق الحساب الافتراضي فقط.
- استخدم `accountId: "*"` كبديل على مستوى القناة عبر جميع الحسابات.
- إذا أضفت لاحقًا الارتباط نفسه للوكيل نفسه مع معرّف حساب صريح، فسيقوم OpenClaw بترقية الارتباط القائم على مستوى القناة إلى نطاق الحساب بدلًا من تكراره.

## عدة حسابات / عدة أرقام هواتف

تستخدم القنوات التي تدعم **عدة حسابات** (مثل WhatsApp) قيمة `accountId` لتحديد
كل تسجيل دخول. ويمكن توجيه كل `accountId` إلى وكيل مختلف، بحيث يمكن لخادم واحد استضافة
عدة أرقام هواتف من دون خلط الجلسات.

إذا كنت تريد حسابًا افتراضيًا على مستوى القناة عند حذف `accountId`، فاضبط
`channels.<channel>.defaultAccount` (اختياري). وعند عدم تعيينه، يعود OpenClaw
إلى `default` إذا كانت موجودة، وإلا إلى أول معرّف حساب مكوَّن (مرتب).

تشمل القنوات الشائعة التي تدعم هذا النمط ما يلي:

- `whatsapp` و`telegram` و`discord` و`slack` و`signal` و`imessage`
- `irc` و`line` و`googlechat` و`mattermost` و`matrix` و`nextcloud-talk`
- `bluebubbles` و`zalo` و`zalouser` و`nostr` و`feishu`

## المفاهيم

- `agentId`: "عقل" واحد (مساحة عمل، ومصادقة لكل وكيل، ومخزن جلسات لكل وكيل).
- `accountId`: مثيل حساب قناة واحد (مثل حساب WhatsApp ‏`"personal"` مقابل `"biz"`).
- `binding`: يوجّه الرسائل الواردة إلى `agentId` بواسطة `(channel, accountId, peer)` ومعرّفات guild/team اختياريًا.
- تنهار المحادثات المباشرة إلى `agent:<agentId>:<mainKey>` (الحالة "الرئيسية" لكل وكيل؛ `session.mainKey`).

## أمثلة المنصات

### Discord bots لكل وكيل

يُطابق كل حساب Discord bot قيمة `accountId` فريدة. اربط كل حساب بوكيل واحتفظ بقوائم السماح لكل bot.

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "coding", workspace: "~/.openclaw/workspace-coding" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "discord", accountId: "default" } },
    { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
  ],
  channels: {
    discord: {
      groupPolicy: "allowlist",
      accounts: {
        default: {
          token: "DISCORD_BOT_TOKEN_MAIN",
          guilds: {
            "123456789012345678": {
              channels: {
                "222222222222222222": { allow: true, requireMention: false },
              },
            },
          },
        },
        coding: {
          token: "DISCORD_BOT_TOKEN_CODING",
          guilds: {
            "123456789012345678": {
              channels: {
                "333333333333333333": { allow: true, requireMention: false },
              },
            },
          },
        },
      },
    },
  },
}
```

ملاحظات:

- ادعُ كل bot إلى guild وفعّل Message Content Intent.
- توجد الرموز المميزة ضمن `channels.discord.accounts.<id>.token` (يمكن للحساب الافتراضي استخدام `DISCORD_BOT_TOKEN`).

### Telegram bots لكل وكيل

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "telegram", accountId: "default" } },
    { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
  ],
  channels: {
    telegram: {
      accounts: {
        default: {
          botToken: "123456:ABC...",
          dmPolicy: "pairing",
        },
        alerts: {
          botToken: "987654:XYZ...",
          dmPolicy: "allowlist",
          allowFrom: ["tg:123456789"],
        },
      },
    },
  },
}
```

ملاحظات:

- أنشئ bot واحدًا لكل وكيل باستخدام BotFather وانسخ كل رمز مميز.
- توجد الرموز المميزة ضمن `channels.telegram.accounts.<id>.botToken` (يمكن للحساب الافتراضي استخدام `TELEGRAM_BOT_TOKEN`).

### أرقام WhatsApp لكل وكيل

اربط كل حساب قبل بدء تشغيل gateway:

```bash
openclaw channels login --channel whatsapp --account personal
openclaw channels login --channel whatsapp --account biz
```

`~/.openclaw/openclaw.json` ‏(JSON5):

```js
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  // توجيه حتمي: أول تطابق يفوز (الأكثر تحديدًا أولًا).
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // تجاوز اختياري لكل peer (مثال: إرسال مجموعة معينة إلى وكيل العمل).
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: { kind: "group", id: "1203630...@g.us" },
      },
    },
  ],

  // معطّل افتراضيًا: يجب تمكين المراسلة بين الوكلاء وقوائم السماح بها صراحةً.
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },

  channels: {
    whatsapp: {
      accounts: {
        personal: {
          // تجاوز اختياري. الافتراضي: ~/.openclaw/credentials/whatsapp/personal
          // authDir: "~/.openclaw/credentials/whatsapp/personal",
        },
        biz: {
          // تجاوز اختياري. الافتراضي: ~/.openclaw/credentials/whatsapp/biz
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

## مثال: دردشة يومية على WhatsApp + عمل عميق على Telegram

قسّم حسب القناة: وجّه WhatsApp إلى وكيل سريع يومي وTelegram إلى وكيل Opus.

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-6",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

ملاحظات:

- إذا كانت لديك عدة حسابات لقناة ما، فأضف `accountId` إلى الارتباط (مثل `{ channel: "whatsapp", accountId: "personal" }`).
- لتوجيه رسالة خاصة/مجموعة واحدة إلى Opus مع إبقاء الباقي على chat، أضف ارتباط `match.peer` لذلك الـ peer؛ إذ تفوز مطابقة peer دائمًا على القواعد الواسعة على مستوى القناة.

## مثال: القناة نفسها، وpeer واحد إلى Opus

أبقِ WhatsApp على الوكيل السريع، لكن وجّه رسالة خاصة واحدة إلى Opus:

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-6",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    {
      agentId: "opus",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551234567" } },
    },
    { agentId: "chat", match: { channel: "whatsapp" } },
  ],
}
```

تفوز ارتباطات peer دائمًا، لذا أبقها فوق القاعدة الواسعة على مستوى القناة.

## وكيل عائلي مرتبط بمجموعة WhatsApp

اربط وكيلًا عائليًا مخصصًا بمجموعة WhatsApp واحدة، مع ضبط بالإشارة
وسياسة أدوات أكثر إحكامًا:

```json5
{
  agents: {
    list: [
      {
        id: "family",
        name: "Family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" },
        groupChat: {
          mentionPatterns: ["@family", "@familybot", "@Family Bot"],
        },
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: [
            "exec",
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
        },
      },
    ],
  },
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "120363999999999999@g.us" },
      },
    },
  ],
}
```

ملاحظات:

- قوائم السماح/المنع للأدوات تخص **الأدوات**، لا Skills. وإذا احتاجت Skill إلى تشغيل
  ملف تنفيذي، فتأكد من السماح بـ `exec` وأن الملف التنفيذي موجود داخل sandbox.
- لمزيد من الضبط الصارم، عيّن `agents.list[].groupChat.mentionPatterns` وأبقِ
  قوائم السماح للمجموعات مفعلة للقناة.

## تكوين Sandbox والأدوات لكل وكيل

يمكن أن يكون لكل وكيل sandbox وقيود أدوات خاصة به:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // لا يوجد sandbox للوكيل الشخصي
        },
        // لا توجد قيود على الأدوات - كل الأدوات متاحة
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // دائمًا داخل sandbox
          scope: "agent",  // حاوية واحدة لكل وكيل
          docker: {
            // إعداد اختياري لمرة واحدة بعد إنشاء الحاوية
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // أداة read فقط
          deny: ["exec", "write", "edit", "apply_patch"],    // منع الآخرين
        },
      },
    ],
  },
}
```

ملاحظة: يوجد `setupCommand` تحت `sandbox.docker` ويعمل مرة واحدة عند إنشاء الحاوية.
يتم تجاهل تجاوزات `sandbox.docker.*` لكل وكيل عندما يكون النطاق المحلول هو `"shared"`.

**الفوائد:**

- **عزل أمني**: تقييد الأدوات للوكلاء غير الموثوقين
- **التحكم في الموارد**: وضع بعض الوكلاء داخل sandbox مع إبقاء آخرين على المضيف
- **سياسات مرنة**: أذونات مختلفة لكل وكيل

ملاحظة: `tools.elevated` **عام** ويعتمد على المرسل؛ وهو غير قابل للتكوين لكل وكيل.
إذا كنت تحتاج إلى حدود لكل وكيل، فاستخدم `agents.list[].tools` لمنع `exec`.
وبالنسبة إلى استهداف المجموعات، استخدم `agents.list[].groupChat.mentionPatterns` بحيث تُطابق إشارات @ الوكيل المقصود بوضوح.

راجع [Sandbox والأدوات متعددة الوكلاء](/tools/multi-agent-sandbox-tools) للحصول على أمثلة مفصلة.

## ذو صلة

- [توجيه القنوات](/channels/channel-routing) — كيف تُوجَّه الرسائل إلى الوكلاء
- [الوكلاء الفرعيون](/tools/subagents) — إنشاء تشغيلات وكلاء في الخلفية
- [وكلاء ACP](/tools/acp-agents) — تشغيل بيئات برمجة خارجية
- [الحضور](/concepts/presence) — حضور الوكيل وتوفره
- [الجلسة](/concepts/session) — عزل الجلسات وتوجيهها
