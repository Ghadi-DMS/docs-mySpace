---
read_when:
    - العمل على ميزات قناة Discord
summary: حالة دعم بوت Discord وإمكاناته وإعداده
title: Discord
x-i18n:
    generated_at: "2026-04-06T03:08:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 54af2176a1b4fa1681e3f07494def0c652a2730165058848000e71a59e2a9d08
    source_path: channels/discord.md
    workflow: 15
---

# Discord (Bot API)

الحالة: جاهز للرسائل المباشرة والقنوات الجماعية عبر بوابة Discord الرسمية.

<CardGroup cols={3}>
  <Card title="الإقران" icon="link" href="/ar/channels/pairing">
    تستخدم الرسائل المباشرة في Discord وضع الإقران افتراضيًا.
  </Card>
  <Card title="أوامر الشرطة المائلة" icon="terminal" href="/ar/tools/slash-commands">
    سلوك الأوامر الأصلي وكتالوج الأوامر.
  </Card>
  <Card title="استكشاف أخطاء القناة وإصلاحها" icon="wrench" href="/ar/channels/troubleshooting">
    التشخيص والتدفق الإصلاحي عبر القنوات.
  </Card>
</CardGroup>

## إعداد سريع

ستحتاج إلى إنشاء تطبيق جديد يحتوي على بوت، وإضافة البوت إلى خادمك، ثم إقرانه مع OpenClaw. نوصي بإضافة البوت إلى خادمك الخاص. إذا لم يكن لديك واحد بعد، [فأنشئ واحدًا أولًا](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (اختر **Create My Own > For me and my friends**).

<Steps>
  <Step title="إنشاء تطبيق Discord وبوت">
    انتقل إلى [Discord Developer Portal](https://discord.com/developers/applications) وانقر على **New Application**. سمّه مثل "OpenClaw".

    انقر على **Bot** في الشريط الجانبي. اضبط **Username** على الاسم الذي تطلقه على وكيل OpenClaw الخاص بك.

  </Step>

  <Step title="تفعيل المقاصد ذات الامتيازات">
    ما زلت في صفحة **Bot**، مرر إلى أسفل حتى **Privileged Gateway Intents** وفعّل:

    - **Message Content Intent** (مطلوب)
    - **Server Members Intent** (موصى به؛ مطلوب لقوائم السماح المستندة إلى الأدوار ولمطابقة الاسم مع المعرّف)
    - **Presence Intent** (اختياري؛ مطلوب فقط لتحديثات الحالة)

  </Step>

  <Step title="نسخ رمز البوت المميز">
    مرر مرة أخرى إلى أعلى صفحة **Bot** وانقر على **Reset Token**.

    <Note>
    رغم الاسم، يؤدي هذا إلى إنشاء أول رمز مميز لك — لا تتم "إعادة تعيين" أي شيء.
    </Note>

    انسخ الرمز المميز واحفظه في مكان ما. هذا هو **Bot Token** الخاص بك وستحتاج إليه بعد قليل.

  </Step>

  <Step title="إنشاء رابط دعوة وإضافة البوت إلى خادمك">
    انقر على **OAuth2** في الشريط الجانبي. ستنشئ رابط دعوة بالأذونات الصحيحة لإضافة البوت إلى خادمك.

    مرر إلى أسفل حتى **OAuth2 URL Generator** وفعّل:

    - `bot`
    - `applications.commands`

    سيظهر قسم **Bot Permissions** أدناه. فعّل:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (اختياري)

    انسخ الرابط المُنشأ في الأسفل، والصقه في المتصفح، واختر خادمك، ثم انقر **Continue** للاتصال. يجب أن ترى الآن البوت في خادم Discord.

  </Step>

  <Step title="تفعيل Developer Mode وجمع المعرّفات الخاصة بك">
    بالعودة إلى تطبيق Discord، تحتاج إلى تفعيل Developer Mode حتى تتمكن من نسخ المعرّفات الداخلية.

    1. انقر على **User Settings** (أيقونة الترس بجانب صورتك الرمزية) → **Advanced** → فعّل **Developer Mode**
    2. انقر بزر الماوس الأيمن على **أيقونة الخادم** في الشريط الجانبي → **Copy Server ID**
    3. انقر بزر الماوس الأيمن على **صورتك الرمزية** → **Copy User ID**

    احفظ **Server ID** و**User ID** إلى جانب Bot Token — سترسل الثلاثة جميعًا إلى OpenClaw في الخطوة التالية.

  </Step>

  <Step title="السماح بالرسائل المباشرة من أعضاء الخادم">
    لكي يعمل الإقران، يجب أن يسمح Discord للبوت بإرسال رسالة مباشرة إليك. انقر بزر الماوس الأيمن على **أيقونة الخادم** → **Privacy Settings** → فعّل **Direct Messages**.

    يتيح هذا لأعضاء الخادم (بما في ذلك البوتات) إرسال رسائل مباشرة إليك. أبقِ هذا مفعّلًا إذا كنت تريد استخدام الرسائل المباشرة في Discord مع OpenClaw. إذا كنت تخطط فقط لاستخدام القنوات الجماعية، يمكنك تعطيل الرسائل المباشرة بعد الإقران.

  </Step>

  <Step title="اضبط رمز البوت المميز بأمان (لا ترسله في الدردشة)">
    رمز بوت Discord المميز هو سر (مثل كلمة المرور). اضبطه على الجهاز الذي يشغّل OpenClaw قبل مراسلة وكيلك.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    إذا كان OpenClaw يعمل بالفعل كخدمة في الخلفية، فأعد تشغيله عبر تطبيق OpenClaw Mac أو عن طريق إيقاف عملية `openclaw gateway run` ثم إعادة تشغيلها.

  </Step>

  <Step title="إعداد OpenClaw والإقران">

    <Tabs>
      <Tab title="اسأل وكيلك">
        تحدث مع وكيل OpenClaw الخاص بك على أي قناة موجودة (مثل Telegram) وأخبره بذلك. إذا كانت Discord هي قناتك الأولى، فاستخدم تبويب CLI / config بدلًا من ذلك.

        > "لقد قمت بالفعل بضبط رمز بوت Discord المميز في الإعدادات. من فضلك أكمل إعداد Discord باستخدام User ID `<user_id>` وServer ID `<server_id>`."
      </Tab>
      <Tab title="CLI / config">
        إذا كنت تفضّل الإعداد المعتمد على الملفات، فاضبط:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        بديل env للحساب الافتراضي:

```bash
DISCORD_BOT_TOKEN=...
```

        القيم النصية الصريحة لـ `token` مدعومة. كما أن قيم SecretRef مدعومة أيضًا لـ `channels.discord.token` عبر موفري env/file/exec. راجع [إدارة الأسرار](/ar/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="الموافقة على أول إقران عبر رسالة مباشرة">
    انتظر حتى تعمل البوابة، ثم أرسل رسالة مباشرة إلى البوت في Discord. سيرد برمز إقران.

    <Tabs>
      <Tab title="اسأل وكيلك">
        أرسل رمز الإقران إلى وكيلك على قناتك الحالية:

        > "وافق على رمز إقران Discord هذا: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    تنتهي صلاحية رموز الإقران بعد ساعة واحدة.

    يجب أن تتمكن الآن من الدردشة مع وكيلك في Discord عبر الرسائل المباشرة.

  </Step>
</Steps>

<Note>
حل الرمز المميز يعتمد على الحساب. قيم الرمز المميز في الإعدادات لها أولوية على بديل env. لا يُستخدم `DISCORD_BOT_TOKEN` إلا للحساب الافتراضي.
بالنسبة إلى الاستدعاءات الصادرة المتقدمة (أداة الرسائل/إجراءات القناة)، يُستخدم `token` صريح لكل استدعاء لذلك الاستدعاء. ينطبق هذا على إجراءات الإرسال والقراءة/الفحص من نوع send وread/probe (مثل read/search/fetch/thread/pins/permissions). ولا تزال إعدادات سياسة الحساب/إعادة المحاولة تأتي من الحساب المحدد في اللقطة النشطة لوقت التشغيل.
</Note>

## موصى به: إعداد مساحة عمل جماعية

بمجرد أن تعمل الرسائل المباشرة، يمكنك إعداد خادم Discord الخاص بك كمساحة عمل كاملة حيث تحصل كل قناة على جلسة وكيل خاصة بها مع سياقها الخاص. يُوصى بهذا للخوادم الخاصة التي تضمك أنت وبوتك فقط.

<Steps>
  <Step title="أضف خادمك إلى قائمة السماح الجماعية">
    يتيح هذا لوكيلك الرد في أي قناة على خادمك، وليس فقط في الرسائل المباشرة.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "أضف Server ID الخاص بـ Discord وهو `<server_id>` إلى قائمة السماح الجماعية"
      </Tab>
      <Tab title="الإعدادات">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="السماح بالردود بدون @mention">
    افتراضيًا، لا يرد وكيلك في القنوات الجماعية إلا عند الإشارة إليه بـ @. بالنسبة إلى خادم خاص، من المحتمل أنك تريد منه الرد على كل رسالة.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "اسمح لوكيلي بالرد على هذا الخادم دون الحاجة إلى الإشارة إليه بـ @"
      </Tab>
      <Tab title="الإعدادات">
        اضبط `requireMention: false` في إعدادات الخادم الجماعي:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="خطّط للذاكرة في القنوات الجماعية">
    افتراضيًا، لا يتم تحميل الذاكرة طويلة المدى (MEMORY.md) إلا في جلسات الرسائل المباشرة. لا يتم تحميل MEMORY.md تلقائيًا في القنوات الجماعية.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "عندما أطرح أسئلة في قنوات Discord، استخدم memory_search أو memory_get إذا احتجت إلى سياق طويل المدى من MEMORY.md."
      </Tab>
      <Tab title="يدويًا">
        إذا كنت تحتاج إلى سياق مشترك في كل قناة، فضع التعليمات الثابتة في `AGENTS.md` أو `USER.md` (يتم حقنهما في كل جلسة). واحتفظ بالملاحظات طويلة المدى في `MEMORY.md` وادخل إليها عند الطلب باستخدام أدوات الذاكرة.
      </Tab>
    </Tabs>

  </Step>
</Steps>

أنشئ الآن بعض القنوات على خادم Discord وابدأ الدردشة. يمكن لوكيلك رؤية اسم القناة، وتحصل كل قناة على جلسة معزولة خاصة بها — لذا يمكنك إعداد `#coding` أو `#home` أو `#research` أو أي شيء يناسب سير عملك.

## نموذج وقت التشغيل

- البوابة تتولى اتصال Discord.
- توجيه الردود حتمي: ترد الرسائل الواردة من Discord إلى Discord.
- افتراضيًا (`session.dmScope=main`)، تشارك الدردشات المباشرة الجلسة الرئيسية للوكيل (`agent:main:main`).
- القنوات الجماعية مفاتيح جلسات معزولة (`agent:<agentId>:discord:channel:<channelId>`).
- يتم تجاهل الرسائل المباشرة الجماعية افتراضيًا (`channels.discord.dm.groupEnabled=false`).
- تعمل أوامر الشرطة المائلة الأصلية في جلسات أوامر معزولة (`agent:<agentId>:discord:slash:<userId>`)، مع الاستمرار في حمل `CommandTargetSessionKey` إلى جلسة المحادثة الموجّهة.

## قنوات المنتدى

لا تقبل قنوات المنتدى والوسائط في Discord إلا منشورات الخيوط. يدعم OpenClaw طريقتين لإنشائها:

- أرسل رسالة إلى أصل المنتدى (`channel:<forumId>`) لإنشاء خيط تلقائيًا. يستخدم عنوان الخيط أول سطر غير فارغ من رسالتك.
- استخدم `openclaw message thread create` لإنشاء خيط مباشرة. لا تمرر `--message-id` لقنوات المنتدى.

مثال: الإرسال إلى أصل المنتدى لإنشاء خيط

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

مثال: إنشاء خيط منتدى بشكل صريح

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

لا تقبل أصول المنتديات مكونات Discord. إذا كنت تحتاج إلى مكونات، فأرسل إلى الخيط نفسه (`channel:<threadId>`).

## المكونات التفاعلية

يدعم OpenClaw حاويات Discord components v2 لرسائل الوكيل. استخدم أداة الرسائل مع حمولة `components`. يتم توجيه نتائج التفاعل مرة أخرى إلى الوكيل كرسائل واردة عادية وتتبع إعدادات Discord `replyToMode` الحالية.

الكتل المدعومة:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- تسمح صفوف الإجراءات بما يصل إلى 5 أزرار أو قائمة اختيار واحدة
- أنواع الاختيار: `string`, `user`, `role`, `mentionable`, `channel`

افتراضيًا، تكون المكونات للاستخدام مرة واحدة. اضبط `components.reusable=true` للسماح باستخدام الأزرار وقوائم الاختيار والنماذج عدة مرات حتى تنتهي صلاحيتها.

لتقييد من يمكنه النقر على زر، اضبط `allowedUsers` على ذلك الزر (معرّفات مستخدمي Discord أو العلامات أو `*`). عند الإعداد، يتلقى المستخدمون غير المطابقين رفضًا مؤقتًا.

يفتح الأمران المائلان `/model` و`/models` منتقي نماذج تفاعليًا مع قوائم منسدلة لمزود النموذج والنموذج نفسه بالإضافة إلى خطوة Submit. يكون رد المنتقي مؤقتًا، ولا يمكن استخدامه إلا من قبل المستخدم الذي استدعاه.

مرفقات الملفات:

- يجب أن تشير كتل `file` إلى مرجع مرفق (`attachment://<filename>`)
- وفّر المرفق عبر `media`/`path`/`filePath` (ملف واحد)؛ استخدم `media-gallery` لعدة ملفات
- استخدم `filename` لتجاوز اسم الرفع عندما يجب أن يطابق مرجع المرفق

النماذج المنبثقة:

- أضف `components.modal` مع ما يصل إلى 5 حقول
- أنواع الحقول: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- يضيف OpenClaw زر تشغيل تلقائيًا

مثال:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## التحكم في الوصول والتوجيه

<Tabs>
  <Tab title="سياسة الرسائل المباشرة">
    يتحكم `channels.discord.dmPolicy` في الوصول إلى الرسائل المباشرة (قديمًا: `channels.discord.dm.policy`):

    - `pairing` (افتراضي)
    - `allowlist`
    - `open` (يتطلب أن يتضمن `channels.discord.allowFrom` القيمة `"*"`؛ قديمًا: `channels.discord.dm.allowFrom`)
    - `disabled`

    إذا لم تكن سياسة الرسائل المباشرة مفتوحة، فسيتم حظر المستخدمين غير المعروفين (أو مطالبتهم بالإقران في وضع `pairing`).

    أولوية الحسابات المتعددة:

    - ينطبق `channels.discord.accounts.default.allowFrom` على الحساب `default` فقط.
    - ترث الحسابات المسماة `channels.discord.allowFrom` عندما لا يكون `allowFrom` الخاص بها مضبوطًا.
    - لا ترث الحسابات المسماة `channels.discord.accounts.default.allowFrom`.

    تنسيق هدف الرسائل المباشرة للتسليم:

    - `user:<id>`
    - الإشارة `<@id>`

    المعرّفات الرقمية المجردة ملتبسة ويتم رفضها ما لم يُوفَّر نوع هدف مستخدم/قناة صريح.

  </Tab>

  <Tab title="سياسة الخادم الجماعي">
    يتم التحكم في التعامل مع الخوادم الجماعية بواسطة `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    خط الأساس الآمن عند وجود `channels.discord` هو `allowlist`.

    سلوك `allowlist`:

    - يجب أن يطابق الخادم الجماعي `channels.discord.guilds` (`id` مفضّل، وslug مقبول)
    - قوائم سماح اختيارية للمرسلين: `users` (يوصى بالمعرّفات الثابتة) و`roles` (معرّفات الأدوار فقط)؛ إذا كان أيٌّ منهما مضبوطًا، فيُسمح للمرسلين عندما يطابقون `users` أو `roles`
    - مطابقة الاسم/العلامة المباشرة معطلة افتراضيًا؛ فعّل `channels.discord.dangerouslyAllowNameMatching: true` فقط كوضع توافق طارئ
    - الأسماء/العلامات مدعومة لـ `users`، لكن المعرّفات أكثر أمانًا؛ ويحذّر `openclaw security audit` عند استخدام إدخالات الاسم/العلامة
    - إذا كان لدى خادم جماعي إعداد `channels`، فسيتم رفض القنوات غير المدرجة
    - إذا لم يكن لدى خادم جماعي كتلة `channels`، فستكون كل القنوات في ذلك الخادم الجماعي المدرج في قائمة السماح مسموحة

    مثال:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    إذا قمت فقط بضبط `DISCORD_BOT_TOKEN` ولم تنشئ كتلة `channels.discord`، فسيكون بديل وقت التشغيل هو `groupPolicy="allowlist"` (مع تحذير في السجلات)، حتى لو كانت `channels.defaults.groupPolicy` هي `open`.

  </Tab>

  <Tab title="الإشارات والرسائل المباشرة الجماعية">
    رسائل الخوادم الجماعية مقيّدة بالإشارة افتراضيًا.

    يشمل اكتشاف الإشارة ما يلي:

    - إشارة صريحة إلى البوت
    - أنماط الإشارة المضبوطة (`agents.list[].groupChat.mentionPatterns`، مع بديل `messages.groupChat.mentionPatterns`)
    - سلوك الرد الضمني على البوت في الحالات المدعومة

    يتم ضبط `requireMention` لكل خادم جماعي/قناة (`channels.discord.guilds...`).
    ويؤدي `ignoreOtherMentions` اختياريًا إلى إسقاط الرسائل التي تشير إلى مستخدم/دور آخر ولكن ليس إلى البوت (باستثناء @everyone/@here).

    الرسائل المباشرة الجماعية:

    - الافتراضي: يتم تجاهلها (`dm.groupEnabled=false`)
    - قائمة سماح اختيارية عبر `dm.groupChannels` (معرّفات القنوات أو الـ slug)

  </Tab>
</Tabs>

### توجيه الوكيل المستند إلى الأدوار

استخدم `bindings[].match.roles` لتوجيه أعضاء خوادم Discord الجماعية إلى وكلاء مختلفين حسب معرّف الدور. تقبل عمليات الربط المستندة إلى الأدوار معرّفات الأدوار فقط، ويتم تقييمها بعد عمليات ربط النظير أو نظير الأصل وقبل عمليات الربط الخاصة بالخادم الجماعي فقط. إذا كان الربط يضبط أيضًا حقول مطابقة أخرى (مثل `peer` + `guildId` + `roles`)، فيجب أن تتطابق كل الحقول المضبوطة.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## إعداد Developer Portal

<AccordionGroup>
  <Accordion title="إنشاء التطبيق والبوت">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. انسخ رمز البوت المميز

  </Accordion>

  <Accordion title="المقاصد ذات الامتيازات">
    في **Bot -> Privileged Gateway Intents**، فعّل:

    - Message Content Intent
    - Server Members Intent (موصى به)

    Intent الحالة اختياري ومطلوب فقط إذا كنت تريد تلقي تحديثات الحالة. لا يتطلب ضبط حالة البوت (`setPresence`) تفعيل تحديثات حالة الأعضاء.

  </Accordion>

  <Accordion title="نطاقات OAuth والأذونات الأساسية">
    مولّد رابط OAuth:

    - النطاقات: `bot`, `applications.commands`

    الأذونات الأساسية النموذجية:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (اختياري)

    تجنب `Administrator` إلا عند الحاجة الصريحة.

  </Accordion>

  <Accordion title="نسخ المعرّفات">
    فعّل Discord Developer Mode، ثم انسخ:

    - معرّف الخادم
    - معرّف القناة
    - معرّف المستخدم

    فضّل المعرّفات الرقمية في إعدادات OpenClaw للحصول على عمليات تدقيق وفحص أكثر موثوقية.

  </Accordion>
</AccordionGroup>

## الأوامر الأصلية ومصادقة الأوامر

- يكون `commands.native` افتراضيًا `"auto"` ومفعّلًا لـ Discord.
- تجاوز لكل قناة: `channels.discord.commands.native`.
- يؤدي `commands.native=false` إلى مسح أوامر Discord الأصلية المسجلة مسبقًا صراحةً.
- تستخدم مصادقة الأوامر الأصلية قوائم السماح/السياسات نفسها في Discord كما في معالجة الرسائل العادية.
- قد تظل الأوامر مرئية في واجهة Discord للمستخدمين غير المصرح لهم؛ ومع ذلك يفرض التنفيذ مصادقة OpenClaw ويعيد "غير مخول".

راجع [أوامر الشرطة المائلة](/ar/tools/slash-commands) للاطلاع على كتالوج الأوامر والسلوك.

إعدادات أوامر الشرطة المائلة الافتراضية:

- `ephemeral: true`

## تفاصيل الميزة

<AccordionGroup>
  <Accordion title="وسوم الردود والردود الأصلية">
    يدعم Discord وسوم الرد في مخرجات الوكيل:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    يتحكم فيها `channels.discord.replyToMode`:

    - `off` (افتراضي)
    - `first`
    - `all`
    - `batched`

    ملاحظة: يؤدي `off` إلى تعطيل سلاسل الرد الضمنية. ولا تزال وسوم `[[reply_to_*]]` الصريحة محترمة.
    يقوم `first` دائمًا بإرفاق مرجع الرد الأصلي الضمني لأول رسالة Discord صادرة في الدور.
    ويقوم `batched` بإرفاق مرجع الرد الأصلي الضمني في Discord فقط عندما
    يكون الدور الوارد دفعة مؤجلة من عدة رسائل. وهذا مفيد
    عندما تريد الردود الأصلية أساسًا للمحادثات المندفعة والغامضة، وليس لكل
    دور رسالة واحدة.

    تظهر معرّفات الرسائل في السياق/السجل حتى تتمكن الوكلاء من استهداف رسائل محددة.

  </Accordion>

  <Accordion title="معاينة البث المباشر">
    يمكن لـ OpenClaw بث مسودات الردود بإرسال رسالة مؤقتة وتحريرها مع وصول النص.

    - يتحكم `channels.discord.streaming` في بث المعاينة (`off` | `partial` | `block` | `progress`، الافتراضي: `off`).
    - يبقى الافتراضي `off` لأن تعديلات معاينة Discord قد تصطدم بحدود المعدل بسرعة، خاصةً عندما تشترك عدة بوتات أو بوابات في الحساب نفسه أو حركة الخادم الجماعي نفسها.
    - يتم قبول `progress` لتحقيق الاتساق عبر القنوات ويُطابَق مع `partial` على Discord.
    - `channels.discord.streamMode` اسم بديل قديم وتتم هجرته تلقائيًا.
    - يقوم `partial` بتحرير رسالة معاينة واحدة مع وصول الرموز.
    - يقوم `block` بإخراج مقاطع بحجم المسودة (استخدم `draftChunk` لضبط الحجم ونقاط التقسيم).

    مثال:

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    إعدادات تقطيع وضع `block` الافتراضية (مقيدة ضمن `channels.discord.textChunkLimit`):

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    بث المعاينة خاص بالنص فقط؛ تعود الردود التي تحتوي على وسائط إلى التسليم العادي.

    ملاحظة: بث المعاينة منفصل عن بث الكتل. عندما يكون block streaming مفعّلًا صراحةً
    لـ Discord، يتجاوز OpenClaw بث المعاينة لتجنب البث المزدوج.

  </Accordion>

  <Accordion title="السجل والسياق وسلوك الخيوط">
    سياق سجل الخادم الجماعي:

    - `channels.discord.historyLimit` الافتراضي `20`
    - البديل: `messages.groupChat.historyLimit`
    - `0` يعطل

    عناصر التحكم في سجل الرسائل المباشرة:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    سلوك الخيوط:

    - يتم توجيه خيوط Discord كجلسات قنوات
    - يمكن استخدام بيانات تعريف خيط الأصل لربط جلسة الأصل
    - يرث إعداد الخيط إعداد القناة الأصل ما لم يوجد إدخال خاص بالخيط

    يتم حقن مواضيع القنوات كسياق **غير موثوق** (وليس كموجّه نظام).
    يظل سياق الرد والرسائل المقتبسة حاليًا كما تم استلامه.
    تعمل قوائم سماح Discord أساسًا على تقييد من يمكنه تشغيل الوكيل، وليست حدًا كاملًا لتنقيح السياق الإضافي.

  </Accordion>

  <Accordion title="جلسات مرتبطة بالخيط للوكلاء الفرعيين">
    يمكن لـ Discord ربط خيط بهدف جلسة بحيث تستمر الرسائل اللاحقة في ذلك الخيط في التوجيه إلى الجلسة نفسها (بما في ذلك جلسات الوكلاء الفرعيين).

    الأوامر:

    - `/focus <target>` ربط الخيط الحالي/الجديد بهدف وكيل فرعي/جلسة
    - `/unfocus` إزالة ربط الخيط الحالي
    - `/agents` عرض عمليات التشغيل النشطة وحالة الربط
    - `/session idle <duration|off>` فحص/تحديث إلغاء التركيز التلقائي لعدم النشاط لعمليات الربط المركزة
    - `/session max-age <duration|off>` فحص/تحديث الحد الأقصى الصارم للعمر لعمليات الربط المركزة

    الإعدادات:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // اشتراك اختياري
      },
    },
  },
}
```

    ملاحظات:

    - يضبط `session.threadBindings.*` الإعدادات الافتراضية العامة.
    - يتجاوز `channels.discord.threadBindings.*` سلوك Discord.
    - يجب أن تكون `spawnSubagentSessions` على true لإنشاء/ربط الخيوط تلقائيًا لـ `sessions_spawn({ thread: true })`.
    - يجب أن تكون `spawnAcpSessions` على true لإنشاء/ربط الخيوط تلقائيًا لـ ACP (`/acp spawn ... --thread ...` أو `sessions_spawn({ runtime: "acp", thread: true })`).
    - إذا كانت عمليات ربط الخيوط معطلة لحساب ما، فلن تتوفر `/focus` وعمليات ربط الخيوط ذات الصلة.

    راجع [الوكلاء الفرعيون](/ar/tools/subagents) و[وكلاء ACP](/ar/tools/acp-agents) و[مرجع الإعدادات](/ar/gateway/configuration-reference).

  </Accordion>

  <Accordion title="عمليات ربط قنوات ACP المستمرة">
    لمساحات عمل ACP ثابتة و"دائمة التشغيل"، اضبط عمليات ربط ACP مكتوبة على المستوى الأعلى تستهدف محادثات Discord.

    مسار الإعدادات:

    - `bindings[]` مع `type: "acp"` و`match.channel: "discord"`

    مثال:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    ملاحظات:

    - يقوم `/acp spawn codex --bind here` بربط قناة Discord الحالية أو الخيط الحالي في مكانه ويحافظ على توجيه الرسائل المستقبلية إلى جلسة ACP نفسها.
    - ما يزال هذا قد يعني "بدء جلسة Codex ACP جديدة"، لكنه لا ينشئ خيط Discord جديدًا بحد ذاته. تبقى القناة الحالية هي سطح الدردشة.
    - قد يستمر Codex في العمل ضمن `cwd` الخاص به أو مساحة عمل backend الخاصة به على القرص. مساحة العمل هذه هي حالة وقت تشغيل، وليست خيط Discord.
    - يمكن أن ترث رسائل الخيط ربط ACP الخاص بالقناة الأصل.
    - في قناة أو خيط مرتبط، يعيد `/new` و`/reset` تعيين جلسة ACP نفسها في مكانها.
    - لا تزال عمليات ربط الخيوط المؤقتة تعمل ويمكنها تجاوز حل الهدف أثناء نشاطها.
    - تكون `spawnAcpSessions` مطلوبة فقط عندما يحتاج OpenClaw إلى إنشاء/ربط خيط فرعي عبر `--thread auto|here`. ولا تكون مطلوبة لـ `/acp spawn ... --bind here` في القناة الحالية.

    راجع [وكلاء ACP](/ar/tools/acp-agents) للحصول على تفاصيل سلوك الربط.

  </Accordion>

  <Accordion title="إشعارات التفاعل">
    وضع إشعارات التفاعل لكل خادم جماعي:

    - `off`
    - `own` (افتراضي)
    - `all`
    - `allowlist` (يستخدم `guilds.<id>.users`)

    تتحول أحداث التفاعل إلى أحداث نظام وتُرفق بجلسة Discord الموجّهة.

  </Accordion>

  <Accordion title="تفاعلات التأكيد">
    يرسل `ackReaction` رمزًا تعبيريًا للتأكيد أثناء معالجة OpenClaw لرسالة واردة.

    ترتيب الحل:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - بديل الرمز التعبيري لهوية الوكيل (`agents.list[].identity.emoji`، وإلا "👀")

    ملاحظات:

    - يقبل Discord الرموز التعبيرية الموحدة أو أسماء الرموز التعبيرية المخصصة.
    - استخدم `""` لتعطيل التفاعل لقناة أو حساب.

  </Accordion>

  <Accordion title="كتابات الإعدادات">
    تكون كتابات الإعدادات التي تبدأ من القناة مفعّلة افتراضيًا.

    يؤثر هذا في تدفقات `/config set|unset` (عند تمكين ميزات الأوامر).

    التعطيل:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="وكيل البوابة">
    وجّه حركة WebSocket الخاصة ببوابة Discord وعمليات بحث REST عند البدء (معرّف التطبيق + حل قائمة السماح) عبر وكيل HTTP(S) باستخدام `channels.discord.proxy`.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    تجاوز لكل حساب:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="دعم PluralKit">
    فعّل حل PluralKit لربط الرسائل المُمررة بهوية عضو النظام:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // اختياري؛ مطلوب للأنظمة الخاصة
      },
    },
  },
}
```

    ملاحظات:

    - يمكن لقوائم السماح استخدام `pk:<memberId>`
    - تتم مطابقة أسماء عرض الأعضاء بالاسم/slug فقط عندما تكون `channels.discord.dangerouslyAllowNameMatching: true`
    - تستخدم عمليات البحث معرّف الرسالة الأصلي وتكون مقيّدة بنافذة زمنية
    - إذا فشل البحث، تُعامل الرسائل المُمررة كرسائل بوت ويتم إسقاطها ما لم يكن `allowBots=true`

  </Accordion>

  <Accordion title="إعدادات الحالة">
    يتم تطبيق تحديثات الحالة عندما تضبط حقل حالة أو نشاط، أو عندما تفعّل الحالة التلقائية.

    مثال للحالة فقط:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    مثال للنشاط (الحالة المخصصة هي نوع النشاط الافتراضي):

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    مثال للبث:

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    خريطة نوع النشاط:

    - 0: Playing
    - 1: Streaming (يتطلب `activityUrl`)
    - 2: Listening
    - 3: Watching
    - 4: Custom (يستخدم نص النشاط كحالة؛ والرمز التعبيري اختياري)
    - 5: Competing

    مثال على الحالة التلقائية (إشارة صحة وقت التشغيل):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    تطابق الحالة التلقائية مدى توفر وقت التشغيل مع حالة Discord: healthy => online، وdegraded أو unknown => idle، وexhausted أو unavailable => dnd. تجاوزات النص الاختيارية:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (يدعم العنصر النائب `{reason}`)

  </Accordion>

  <Accordion title="الموافقات في Discord">
    يدعم Discord معالجة الموافقات المستندة إلى الأزرار في الرسائل المباشرة ويمكنه اختياريًا نشر مطالبات الموافقة في القناة الأصلية.

    مسار الإعدادات:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (اختياري؛ يعود إلى `commands.ownerAllowFrom` عندما يكون ذلك ممكنًا)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    يفعّل Discord موافقات التنفيذ الأصلية تلقائيًا عندما تكون `enabled` غير مضبوطة أو تساوي `"auto"` وعندما يمكن حل معتمد واحد على الأقل، إما من `execApprovals.approvers` أو من `commands.ownerAllowFrom`. لا يستنتج Discord معتمدي التنفيذ من `allowFrom` الخاصة بالقناة أو `dm.allowFrom` القديمة أو `defaultTo` الخاصة بالرسائل المباشرة. اضبط `enabled: false` لتعطيل Discord كعميل موافقة أصلي بشكل صريح.

    عندما تكون `target` هي `channel` أو `both`، تكون مطالبة الموافقة مرئية في القناة. لا يمكن استخدام الأزرار إلا من قبل المعتمدين الذين تم حلهم؛ ويتلقى المستخدمون الآخرون رفضًا مؤقتًا. تتضمن مطالبات الموافقة نص الأمر، لذلك فعّل التسليم عبر القناة فقط في القنوات الموثوقة. إذا تعذر اشتقاق معرّف القناة من مفتاح الجلسة، يعود OpenClaw إلى التسليم عبر الرسائل المباشرة.

    يعرض Discord أيضًا أزرار الموافقة المشتركة المستخدمة من قنوات الدردشة الأخرى. يضيف مهايئ Discord الأصلي أساسًا توجيه الرسائل المباشرة للمعتمدين والنشر في القناة.
    عندما تكون هذه الأزرار موجودة، فهي تجربة المستخدم الأساسية للموافقة؛ ويجب على OpenClaw
    تضمين أمر `/approve` يدوي فقط عندما تشير نتيجة الأداة إلى أن
    موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.

    تستخدم مصادقة البوابة لهذا المعالج عقد حل بيانات الاعتماد المشتركة نفسه كما في عملاء البوابة الآخرين:

    - مصادقة محلية من env أولًا (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` ثم `gateway.auth.*`)
    - في الوضع المحلي، يمكن استخدام `gateway.remote.*` كبديل فقط عندما لا يكون `gateway.auth.*` مضبوطًا؛ تفشل SecretRefs المحلية المضبوطة ولكن غير المحلولة بشكل مغلق
    - دعم الوضع البعيد عبر `gateway.remote.*` عندما ينطبق ذلك
    - تجاوزات URL آمنة للتجاوز: لا تعيد تجاوزات CLI استخدام بيانات اعتماد ضمنية، وتستخدم تجاوزات env بيانات اعتماد env فقط

    سلوك حل الموافقة:

    - يتم حل المعرّفات المسبوقة بـ `plugin:` عبر `plugin.approval.resolve`.
    - يتم حل المعرّفات الأخرى عبر `exec.approval.resolve`.
    - لا يقوم Discord هنا بقفزة بديلة إضافية من exec إلى plugin؛ تحدد
      بادئة المعرّف طريقة البوابة التي يستدعيها.

    تنتهي موافقات التنفيذ بعد 30 دقيقة افتراضيًا. إذا فشلت الموافقات مع
    معرّفات موافقة غير معروفة، فتحقق من حل المعتمدين وتمكين الميزة وأن
    نوع معرّف الموافقة المُسلَّم يطابق الطلب المعلّق.

    الوثائق ذات الصلة: [موافقات التنفيذ](/ar/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## الأدوات وبوابات الإجراءات

تتضمن إجراءات رسائل Discord المراسلة، وإدارة القناة، والإشراف، والحالة، وإجراءات البيانات الوصفية.

أمثلة أساسية:

- المراسلة: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- التفاعلات: `react`, `reactions`, `emojiList`
- الإشراف: `timeout`, `kick`, `ban`
- الحالة: `setPresence`

توجد بوابات الإجراءات تحت `channels.discord.actions.*`.

سلوك البوابة الافتراضي:

| مجموعة الإجراءات                                                                                                                                                             | الافتراضي |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | مفعّل  |
| roles                                                                                                                                                                    | معطّل |
| moderation                                                                                                                                                               | معطّل |
| presence                                                                                                                                                                 | معطّل |

## واجهة Components v2

يستخدم OpenClaw Discord components v2 لموافقات التنفيذ وعلامات السياق المتقاطع. يمكن لإجراءات رسائل Discord أيضًا قبول `components` لواجهة مستخدم مخصصة (متقدم؛ يتطلب إنشاء حمولة مكونات عبر أداة discord)، بينما تظل `embeds` القديمة متاحة ولكن غير موصى بها.

- يضبط `channels.discord.ui.components.accentColor` لون التمييز المستخدم بواسطة حاويات مكونات Discord (hex).
- اضبطه لكل حساب باستخدام `channels.discord.accounts.<id>.ui.components.accentColor`.
- يتم تجاهل `embeds` عند وجود components v2.

مثال:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## القنوات الصوتية

يمكن لـ OpenClaw الانضمام إلى القنوات الصوتية في Discord لإجراء محادثات آنية ومستمرة. وهذا منفصل عن مرفقات الرسائل الصوتية.

المتطلبات:

- فعّل الأوامر الأصلية (`commands.native` أو `channels.discord.commands.native`).
- اضبط `channels.discord.voice`.
- يحتاج البوت إلى أذونات Connect + Speak في القناة الصوتية المستهدفة.

استخدم أمر Discord الأصلي فقط `/vc join|leave|status` للتحكم في الجلسات. يستخدم الأمر الوكيل الافتراضي للحساب ويتبع قواعد قائمة السماح وسياسة المجموعة نفسها كما في أوامر Discord الأخرى.

مثال على الانضمام التلقائي:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
    },
  },
}
```

ملاحظات:

- يتجاوز `voice.tts` القيمة `messages.tts` لتشغيل الصوت فقط.
- تستمد أدوار مالك نُسخ المحادثات الصوتية من Discord `allowFrom` (أو `dm.allowFrom`)؛ لا يمكن للمتحدثين غير المالكين الوصول إلى الأدوات المخصصة للمالك فقط (مثل `gateway` و`cron`).
- يكون الصوت مفعّلًا افتراضيًا؛ اضبط `channels.discord.voice.enabled=false` لتعطيله.
- يتم تمرير `voice.daveEncryption` و`voice.decryptionFailureTolerance` إلى خيارات الانضمام في `@discordjs/voice`.
- القيم الافتراضية لـ `@discordjs/voice` هي `daveEncryption=true` و`decryptionFailureTolerance=24` إذا لم يتم ضبطها.
- يراقب OpenClaw أيضًا إخفاقات فك التشفير عند الاستقبال ويقوم بالاسترداد التلقائي عبر المغادرة/إعادة الانضمام إلى القناة الصوتية بعد تكرار الإخفاقات في نافذة زمنية قصيرة.
- إذا أظهرت سجلات الاستقبال مرارًا `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`، فقد يكون هذا هو خلل الاستقبال upstream في `@discordjs/voice` المتعقَّب في [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419).

## الرسائل الصوتية

تعرض الرسائل الصوتية في Discord معاينة شكل موجي وتتطلب صوت OGG/Opus بالإضافة إلى بيانات وصفية. ينشئ OpenClaw الشكل الموجي تلقائيًا، لكنه يحتاج إلى توفر `ffmpeg` و`ffprobe` على مضيف البوابة لفحص ملفات الصوت وتحويلها.

المتطلبات والقيود:

- قدّم **مسار ملف محلي** (يتم رفض عناوين URL).
- احذف المحتوى النصي (لا يسمح Discord بنص + رسالة صوتية في الحمولة نفسها).
- يُقبل أي تنسيق صوتي؛ ويحوّل OpenClaw التنسيق إلى OGG/Opus عند الحاجة.

مثال:

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="تم استخدام intents غير مسموح بها أو أن البوت لا يرى رسائل الخوادم الجماعية">

    - فعّل Message Content Intent
    - فعّل Server Members Intent عندما تعتمد على حل المستخدم/العضو
    - أعد تشغيل البوابة بعد تغيير intents

  </Accordion>

  <Accordion title="تم حظر رسائل الخوادم الجماعية بشكل غير متوقع">

    - تحقّق من `groupPolicy`
    - تحقّق من قائمة السماح للخادم الجماعي ضمن `channels.discord.guilds`
    - إذا كانت خريطة `channels` موجودة للخادم الجماعي، فسيُسمح فقط بالقنوات المدرجة
    - تحقّق من سلوك `requireMention` وأنماط الإشارة

    فحوصات مفيدة:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="require mention على false لكنه ما يزال محظورًا">
    الأسباب الشائعة:

    - `groupPolicy="allowlist"` بدون قائمة سماح مطابقة للخادم الجماعي/القناة
    - تم ضبط `requireMention` في المكان الخطأ (يجب أن تكون تحت `channels.discord.guilds` أو إدخال القناة)
    - المرسل محظور بواسطة قائمة السماح `users` للخادم الجماعي/القناة

  </Accordion>

  <Accordion title="تنتهي مهلة المعالجات طويلة التشغيل أو تحدث ردود مكررة">

    سجلات نموذجية:

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    مقبض ميزانية المستمع:

    - حساب واحد: `channels.discord.eventQueue.listenerTimeout`
    - حسابات متعددة: `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    مقبض مهلة تشغيل العامل:

    - حساب واحد: `channels.discord.inboundWorker.runTimeoutMs`
    - حسابات متعددة: `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - الافتراضي: `1800000` (30 دقيقة)؛ اضبط `0` للتعطيل

    خط أساس موصى به:

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    استخدم `eventQueue.listenerTimeout` لإعداد المستمع البطيء و`inboundWorker.runTimeoutMs`
    فقط إذا كنت تريد صمام أمان منفصلًا لأدوار الوكيل الموضوعة في قائمة الانتظار.

  </Accordion>

  <Accordion title="عدم تطابق تدقيق الأذونات">
    تعمل فحوصات الأذونات في `channels status --probe` فقط مع معرّفات القنوات الرقمية.

    إذا كنت تستخدم مفاتيح slug، فقد تستمر المطابقة وقت التشغيل في العمل، لكن الفحص لا يمكنه التحقق الكامل من الأذونات.

  </Accordion>

  <Accordion title="مشكلات الرسائل المباشرة والإقران">

    - الرسائل المباشرة معطلة: `channels.discord.dm.enabled=false`
    - سياسة الرسائل المباشرة معطلة: `channels.discord.dmPolicy="disabled"` (قديمًا: `channels.discord.dm.policy`)
    - بانتظار الموافقة على الإقران في وضع `pairing`

  </Accordion>

  <Accordion title="حلقات بوت إلى بوت">
    افتراضيًا، يتم تجاهل الرسائل التي يكتبها البوت.

    إذا قمت بضبط `channels.discord.allowBots=true`، فاستخدم قواعد صارمة للإشارة وقائمة السماح لتجنب سلوك الحلقات.
    فضّل `channels.discord.allowBots="mentions"` لقبول رسائل البوت التي تشير إلى البوت فقط.

  </Accordion>

  <Accordion title="سقوط Voice STT مع DecryptionFailed(...)">

    - حافظ على تحديث OpenClaw (`openclaw update`) حتى يكون منطق استرداد استقبال الصوت في Discord موجودًا
    - أكّد أن `channels.discord.voice.daveEncryption=true` (افتراضي)
    - ابدأ من `channels.discord.voice.decryptionFailureTolerance=24` (القيمة الافتراضية upstream) وعدّل فقط عند الحاجة
    - راقب السجلات بحثًا عن:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - إذا استمرت الإخفاقات بعد إعادة الانضمام التلقائية، فاجمع السجلات وقارنها مع [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## مؤشرات مرجع الإعدادات

المرجع الأساسي:

- [مرجع الإعدادات - Discord](/ar/gateway/configuration-reference#discord)

حقول Discord عالية الإشارة:

- البدء/المصادقة: `enabled`, `token`, `accounts.*`, `allowBots`
- السياسة: `groupPolicy`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- الأمر: `commands.native`, `commands.useAccessGroups`, `configWrites`, `slashCommand.*`
- قائمة انتظار الأحداث: `eventQueue.listenerTimeout` (ميزانية المستمع)، `eventQueue.maxQueueSize`, `eventQueue.maxConcurrency`
- العامل الوارد: `inboundWorker.runTimeoutMs`
- الرد/السجل: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- التسليم: `textChunkLimit`, `chunkMode`, `maxLinesPerMessage`
- البث: `streaming` (اسم بديل قديم: `streamMode`)، `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`
- الوسائط/إعادة المحاولة: `mediaMaxMb`, `retry`
  - يحد `mediaMaxMb` من عمليات رفع Discord الصادرة (الافتراضي: `100MB`)
- الإجراءات: `actions.*`
- الحالة: `activity`, `status`, `activityType`, `activityUrl`
- واجهة المستخدم: `ui.components.accentColor`
- الميزات: `threadBindings`, المستوى الأعلى `bindings[]` (`type: "acp"`)، `pluralkit`, `execApprovals`, `intents`, `agentComponents`, `heartbeat`, `responsePrefix`

## السلامة والعمليات

- تعامل مع رموز البوت المميزة كأسرار (يفضّل `DISCORD_BOT_TOKEN` في البيئات الخاضعة للإشراف).
- امنح أذونات Discord بأقل قدر لازم من الصلاحيات.
- إذا كانت حالة نشر الأوامر/الحالة قديمة، فأعد تشغيل البوابة وأعد التحقق باستخدام `openclaw channels status --probe`.

## ذو صلة

- [الإقران](/ar/channels/pairing)
- [المجموعات](/ar/channels/groups)
- [توجيه القنوات](/ar/channels/channel-routing)
- [الأمان](/ar/gateway/security)
- [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)
- [استكشاف الأخطاء وإصلاحها](/ar/channels/troubleshooting)
- [أوامر الشرطة المائلة](/ar/tools/slash-commands)
