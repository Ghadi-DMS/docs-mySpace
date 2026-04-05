---
read_when:
    - عند العمل على ميزات قناة Discord
summary: حالة دعم بوت Discord، وإمكاناته، وإعداده
title: Discord
x-i18n:
    generated_at: "2026-04-05T12:36:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: e757d321d80d05642cd9e24b51fb47897bacaf8db19df83bd61a49a8ce51ed3a
    source_path: channels/discord.md
    workflow: 15
---

# Discord (Bot API)

الحالة: جاهز للرسائل الخاصة والقنوات داخل الخوادم عبر بوابة Discord الرسمية.

<CardGroup cols={3}>
  <Card title="الاقتران" icon="link" href="/channels/pairing">
    تستخدم رسائل Discord الخاصة وضع الاقتران افتراضيًا.
  </Card>
  <Card title="أوامر الشرطة المائلة" icon="terminal" href="/tools/slash-commands">
    سلوك الأوامر الأصلي وفهرس الأوامر.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/channels/troubleshooting">
    التشخيصات متعددة القنوات وتدفق الإصلاح.
  </Card>
</CardGroup>

## إعداد سريع

ستحتاج إلى إنشاء تطبيق جديد مع بوت، وإضافة البوت إلى خادمك، ثم إقرانه مع OpenClaw. نوصي بإضافة البوت إلى خادمك الخاص. إذا لم يكن لديك خادم بعد، [فأنشئ واحدًا أولًا](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (اختر **Create My Own > For me and my friends**).

<Steps>
  <Step title="إنشاء تطبيق Discord وبوت">
    انتقل إلى [Discord Developer Portal](https://discord.com/developers/applications) وانقر على **New Application**. سمِّه بشيء مثل "OpenClaw".

    انقر على **Bot** في الشريط الجانبي. اضبط **Username** على الاسم الذي تطلقه على وكيل OpenClaw الخاص بك.

  </Step>

  <Step title="تمكين الأذونات المميّزة">
    لا تزال في صفحة **Bot**، مرّر إلى أسفل حتى **Privileged Gateway Intents** وفعّل:

    - **Message Content Intent** (مطلوب)
    - **Server Members Intent** (موصى به؛ ومطلوب لقوائم السماح المستندة إلى الأدوار ولمطابقة الاسم بالمعرّف)
    - **Presence Intent** (اختياري؛ مطلوب فقط لتحديثات الحالة)

  </Step>

  <Step title="نسخ رمز البوت المميز">
    مرّر مجددًا إلى أعلى صفحة **Bot** وانقر على **Reset Token**.

    <Note>
    على الرغم من الاسم، فإن هذا يولّد أول رمز مميز لك — لا تتم "إعادة تعيين" أي شيء.
    </Note>

    انسخ الرمز المميز واحفظه في مكان ما. هذا هو **Bot Token** الخاص بك وستحتاج إليه بعد قليل.

  </Step>

  <Step title="إنشاء رابط دعوة وإضافة البوت إلى خادمك">
    انقر على **OAuth2** في الشريط الجانبي. ستنشئ رابط دعوة بالأذونات الصحيحة لإضافة البوت إلى خادمك.

    مرّر إلى أسفل حتى **OAuth2 URL Generator** وفعّل:

    - `bot`
    - `applications.commands`

    سيظهر قسم **Bot Permissions** بالأسفل. فعّل:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (اختياري)

    انسخ الرابط الذي تم إنشاؤه في الأسفل، والصقه في متصفحك، وحدد خادمك، ثم انقر **Continue** للاتصال. يجب أن ترى الآن البوت الخاص بك داخل خادم Discord.

  </Step>

  <Step title="تمكين Developer Mode وجمع المعرّفات الخاصة بك">
    بالعودة إلى تطبيق Discord، تحتاج إلى تمكين Developer Mode حتى تتمكن من نسخ المعرّفات الداخلية.

    1. انقر على **User Settings** (أيقونة الترس بجوار صورتك الرمزية) → **Advanced** → فعّل **Developer Mode**
    2. انقر بزر الفأرة الأيمن على **أيقونة الخادم** في الشريط الجانبي → **Copy Server ID**
    3. انقر بزر الفأرة الأيمن على **صورتك الرمزية** → **Copy User ID**

    احفظ **Server ID** و**User ID** مع Bot Token — سترسل الثلاثة جميعًا إلى OpenClaw في الخطوة التالية.

  </Step>

  <Step title="السماح بالرسائل الخاصة من أعضاء الخادم">
    لكي يعمل الاقتران، يجب أن يسمح Discord للبوت بإرسال رسالة خاصة إليك. انقر بزر الفأرة الأيمن على **أيقونة الخادم** → **Privacy Settings** → فعّل **Direct Messages**.

    يتيح هذا لأعضاء الخادم (بمن فيهم البوتات) إرسال رسائل خاصة إليك. أبقِ هذا الخيار مفعّلًا إذا كنت تريد استخدام رسائل Discord الخاصة مع OpenClaw. إذا كنت تخطط لاستخدام قنوات الخادم فقط، يمكنك تعطيل الرسائل الخاصة بعد الاقتران.

  </Step>

  <Step title="تعيين رمز البوت المميز بأمان (لا ترسله في الدردشة)">
    رمز بوت Discord المميز الخاص بك هو سر (مثل كلمة المرور). اضبطه على الجهاز الذي يشغّل OpenClaw قبل مراسلة وكيلك.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    إذا كان OpenClaw يعمل بالفعل كخدمة في الخلفية، فأعد تشغيله عبر تطبيق OpenClaw على Mac أو عبر إيقاف عملية `openclaw gateway run` وتشغيلها مجددًا.

  </Step>

  <Step title="إعداد OpenClaw والاقتران">

    <Tabs>
      <Tab title="اسأل وكيلك">
        تحدّث مع وكيل OpenClaw الخاص بك على أي قناة موجودة بالفعل (مثل Telegram) وأخبره بذلك. إذا كان Discord هو قناتك الأولى، فاستخدم علامة تبويب CLI / config بدلًا من ذلك.

        > "لقد قمت بالفعل بتعيين رمز بوت Discord المميز في الإعدادات. يُرجى إكمال إعداد Discord باستخدام User ID `<user_id>` وServer ID `<server_id>`."
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

        الرجوع إلى env للحساب الافتراضي:

```bash
DISCORD_BOT_TOKEN=...
```

        القيم النصية الصريحة `token` مدعومة. كما أن قيم SecretRef مدعومة أيضًا لـ `channels.discord.token` عبر موفري env/file/exec. راجع [Secrets Management](/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="الموافقة على أول اقتران عبر الرسائل الخاصة">
    انتظر حتى تعمل البوابة، ثم أرسل رسالة خاصة إلى البوت في Discord. سيرد عليك برمز اقتران.

    <Tabs>
      <Tab title="اسأل وكيلك">
        أرسل رمز الاقتران إلى وكيلك على قناتك الحالية:

        > "وافق على رمز اقتران Discord هذا: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    تنتهي صلاحية رموز الاقتران بعد ساعة واحدة.

    يجب أن تتمكن الآن من الدردشة مع وكيلك في Discord عبر الرسائل الخاصة.

  </Step>
</Steps>

<Note>
يأخذ حل الرمز المميز في الاعتبار الحساب. تتغلب قيم الرمز المميز في الإعدادات على الرجوع إلى env. لا يُستخدم `DISCORD_BOT_TOKEN` إلا للحساب الافتراضي.
بالنسبة إلى الاستدعاءات الصادرة المتقدمة (أداة الرسائل/إجراءات القناة)، يتم استخدام `token` صريح لكل استدعاء لتلك العملية. ينطبق هذا على إجراءات الإرسال وإجراءات القراءة/الفحص من النمط نفسه (مثل read/search/fetch/thread/pins/permissions). تظل إعدادات سياسة الحساب/إعادة المحاولة تأتي من الحساب المحدد في اللقطة النشطة لوقت التشغيل.
</Note>

## موصى به: إعداد مساحة عمل للخادم

بمجرد أن تعمل الرسائل الخاصة، يمكنك إعداد خادم Discord الخاص بك كمساحة عمل كاملة بحيث تحصل كل قناة على جلسة وكيل خاصة بها وسياقها الخاص. يوصى بذلك للخوادم الخاصة التي تكون لك ولبوتك فقط.

<Steps>
  <Step title="إضافة خادمك إلى قائمة السماح الخاصة بالخوادم">
    يتيح ذلك لوكيلك الرد في أي قناة على خادمك، وليس فقط في الرسائل الخاصة.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "أضف Server ID الخاص بخادم Discord `<server_id>` إلى قائمة السماح الخاصة بالخوادم"
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

  <Step title="السماح بالردود دون @mention">
    افتراضيًا، لا يرد وكيلك في قنوات الخادم إلا عند الإشارة إليه باستخدام @mention. بالنسبة إلى خادم خاص، فغالبًا ستريده أن يرد على كل رسالة.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "اسمح لوكيلي بالرد على هذا الخادم من دون الحاجة إلى الإشارة إليه باستخدام @mention"
      </Tab>
      <Tab title="الإعدادات">
        اضبط `requireMention: false` في إعدادات الخادم:

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

  <Step title="خطط للذاكرة في قنوات الخادم">
    افتراضيًا، لا يتم تحميل الذاكرة طويلة الأمد (`MEMORY.md`) إلا في جلسات الرسائل الخاصة. قنوات الخادم لا تحمّل `MEMORY.md` تلقائيًا.

    <Tabs>
      <Tab title="اسأل وكيلك">
        > "عندما أطرح أسئلة في قنوات Discord، استخدم memory_search أو memory_get إذا كنت بحاجة إلى سياق طويل الأمد من `MEMORY.md`."
      </Tab>
      <Tab title="يدوي">
        إذا كنت بحاجة إلى سياق مشترك في كل قناة، فضع التعليمات الثابتة في `AGENTS.md` أو `USER.md` (إذ يتم حقنهما في كل جلسة). واحتفظ بالملاحظات طويلة الأمد في `MEMORY.md` واطلب الوصول إليها عند الحاجة باستخدام أدوات الذاكرة.
      </Tab>
    </Tabs>

  </Step>
</Steps>

أنشئ الآن بعض القنوات على خادم Discord وابدأ الدردشة. يمكن لوكيلك رؤية اسم القناة، وتحصل كل قناة على جلسة معزولة خاصة بها — بحيث يمكنك إعداد `#coding` و`#home` و`#research` أو أي شيء يناسب سير عملك.

## نموذج وقت التشغيل

- تتولى البوابة اتصال Discord.
- يكون توجيه الردود حتميًا: تدخل الردود الواردة من Discord وتعود إلى Discord.
- افتراضيًا (`session.dmScope=main`)، تشارك المحادثات المباشرة الجلسة الرئيسية للوكيل (`agent:main:main`).
- قنوات الخادم عبارة عن مفاتيح جلسات معزولة (`agent:<agentId>:discord:channel:<channelId>`).
- يتم تجاهل الرسائل الخاصة الجماعية افتراضيًا (`channels.discord.dm.groupEnabled=false`).
- تعمل أوامر الشرطة المائلة الأصلية في جلسات أوامر معزولة (`agent:<agentId>:discord:slash:<userId>`)، مع الاستمرار في حمل `CommandTargetSessionKey` إلى جلسة المحادثة الموجَّهة.

## قنوات المنتدى

لا تقبل قنوات المنتدى والوسائط في Discord إلا المشاركات ضمن سلاسل. يدعم OpenClaw طريقتين لإنشائها:

- أرسل رسالة إلى أصل المنتدى (`channel:<forumId>`) لإنشاء سلسلة تلقائيًا. يستخدم عنوان السلسلة أول سطر غير فارغ من رسالتك.
- استخدم `openclaw message thread create` لإنشاء سلسلة مباشرة. لا تمرر `--message-id` لقنوات المنتدى.

مثال: الإرسال إلى أصل المنتدى لإنشاء سلسلة

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

مثال: إنشاء سلسلة منتدى بشكل صريح

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

لا تقبل أصول المنتدى مكونات Discord. إذا كنت بحاجة إلى مكونات، فأرسل إلى السلسلة نفسها (`channel:<threadId>`).

## المكونات التفاعلية

يدعم OpenClaw حاويات Discord components v2 لرسائل الوكيل. استخدم أداة الرسائل مع حمولة `components`. تُوجَّه نتائج التفاعل مرة أخرى إلى الوكيل باعتبارها رسائل واردة عادية، وتتبع إعدادات Discord `replyToMode` الحالية.

الكتل المدعومة:

- `text` و`section` و`separator` و`actions` و`media-gallery` و`file`
- تسمح صفوف الإجراءات بما يصل إلى 5 أزرار أو قائمة تحديد واحدة
- أنواع التحديد: `string` و`user` و`role` و`mentionable` و`channel`

افتراضيًا، تكون المكونات للاستخدام مرة واحدة. اضبط `components.reusable=true` للسماح باستخدام الأزرار وعناصر التحديد والنماذج عدة مرات حتى تنتهي صلاحيتها.

لتقييد من يمكنه النقر على زر، اضبط `allowedUsers` على ذلك الزر (معرّفات مستخدمي Discord أو الوسوم أو `*`). عند الإعداد، يتلقى المستخدمون غير المطابقين رفضًا مؤقتًا.

تفتح أوامر الشرطة المائلة `/model` و`/models` منتقي نماذج تفاعليًا مع قوائم منسدلة لموفر الخدمة والنموذج إضافة إلى خطوة Submit. يكون رد المنتقي مؤقتًا ويمكن فقط للمستخدم الذي استدعاه استخدامه.

مرفقات الملفات:

- يجب أن تشير كتل `file` إلى مرجع مرفق (`attachment://<filename>`)
- قدّم المرفق عبر `media`/`path`/`filePath` (ملف واحد)؛ استخدم `media-gallery` لعدة ملفات
- استخدم `filename` لتجاوز اسم الرفع عندما يجب أن يطابق مرجع المرفق

نماذج Modal:

- أضف `components.modal` بما يصل إلى 5 حقول
- أنواع الحقول: `text` و`checkbox` و`radio` و`select` و`role-select` و`user-select`
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
  <Tab title="سياسة الرسائل الخاصة">
    يتحكم `channels.discord.dmPolicy` في الوصول إلى الرسائل الخاصة (الاسم القديم: `channels.discord.dm.policy`):

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن يحتوي `channels.discord.allowFrom` على `"*"`؛ الاسم القديم: `channels.discord.dm.allowFrom`)
    - `disabled`

    إذا لم تكن سياسة الرسائل الخاصة مفتوحة، فسيتم حظر المستخدمين غير المعروفين (أو مطالبتهم بالاقتران في وضع `pairing`).

    أسبقية الحسابات المتعددة:

    - ينطبق `channels.discord.accounts.default.allowFrom` على الحساب `default` فقط.
    - ترث الحسابات المسماة `channels.discord.allowFrom` عندما لا يتم تعيين `allowFrom` الخاص بها.
    - لا ترث الحسابات المسماة `channels.discord.accounts.default.allowFrom`.

    صيغة هدف الرسائل الخاصة للتسليم:

    - `user:<id>`
    - إشارة `<@id>`

    تكون المعرّفات الرقمية المجردة ملتبسة ويتم رفضها ما لم يتم توفير نوع هدف user/channel صريح.

  </Tab>

  <Tab title="سياسة الخوادم">
    يتم التحكم في التعامل مع الخوادم بواسطة `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    خط الأساس الآمن عند وجود `channels.discord` هو `allowlist`.

    سلوك `allowlist`:

    - يجب أن يطابق الخادم `channels.discord.guilds` (يفضل `id`، ويقبل slug)
    - قوائم السماح الاختيارية للمرسلين: `users` (يوصى بالمعرّفات الثابتة) و`roles` (معرّفات الأدوار فقط)؛ إذا تم إعداد أي منهما، يُسمح للمرسلين عندما يطابقون `users` أو `roles`
    - تكون مطابقة الاسم/الوسم المباشرة معطلة افتراضيًا؛ فعّل `channels.discord.dangerouslyAllowNameMatching: true` فقط كوضع توافقية طارئ
    - الأسماء/الوسوم مدعومة في `users`، لكن المعرّفات أكثر أمانًا؛ ويحذّر `openclaw security audit` عند استخدام إدخالات الاسم/الوسم
    - إذا كان لدى خادم ما `channels` مهيأة، فسيتم رفض القنوات غير المدرجة
    - إذا لم يكن لدى الخادم كتلة `channels`، فسيُسمح بكل القنوات في ذلك الخادم المدرج في قائمة السماح

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

    إذا قمت فقط بتعيين `DISCORD_BOT_TOKEN` ولم تُنشئ كتلة `channels.discord`، فسيكون الرجوع في وقت التشغيل هو `groupPolicy="allowlist"` (مع تحذير في السجلات)، حتى إذا كان `channels.defaults.groupPolicy` يساوي `open`.

  </Tab>

  <Tab title="الإشارات والرسائل الخاصة الجماعية">
    تكون رسائل الخوادم مقيدة بالإشارة افتراضيًا.

    يشمل اكتشاف الإشارة ما يلي:

    - إشارة صريحة إلى البوت
    - أنماط الإشارة المهيأة (`agents.list[].groupChat.mentionPatterns`، والرجوع إلى `messages.groupChat.mentionPatterns`)
    - سلوك الرد الضمني على البوت في الحالات المدعومة

    يتم إعداد `requireMention` لكل خادم/قناة (`channels.discord.guilds...`).
    ويمكن لـ `ignoreOtherMentions` اختياريًا إسقاط الرسائل التي تشير إلى مستخدم/دور آخر ولكن ليس إلى البوت (باستثناء @everyone/@here).

    الرسائل الخاصة الجماعية:

    - الافتراضي: يتم تجاهلها (`dm.groupEnabled=false`)
    - قائمة السماح الاختيارية عبر `dm.groupChannels` (معرّفات القنوات أو slug)

  </Tab>
</Tabs>

### توجيه الوكلاء المستند إلى الأدوار

استخدم `bindings[].match.roles` لتوجيه أعضاء خوادم Discord إلى وكلاء مختلفين بحسب معرّف الدور. تقبل عمليات الربط المستندة إلى الأدوار معرّفات الأدوار فقط، ويتم تقييمها بعد عمليات الربط peer أو parent-peer وقبل عمليات الربط الخاصة بالخادم فقط. إذا كانت عملية الربط تضبط أيضًا حقول مطابقة أخرى (مثل `peer` + `guildId` + `roles`)، فيجب أن تتطابق جميع الحقول المهيأة.

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

  <Accordion title="الأذونات المميّزة">
    في **Bot -> Privileged Gateway Intents**، فعّل:

    - Message Content Intent
    - Server Members Intent (موصى به)

    Presence intent اختياري ولا يلزم إلا إذا كنت تريد تلقي تحديثات الحالة. لا يتطلب تعيين حالة البوت (`setPresence`) تمكين تحديثات الحالة للأعضاء.

  </Accordion>

  <Accordion title="نطاقات OAuth والأذونات الأساسية">
    مولد عنوان URL لـ OAuth:

    - النطاقات: `bot` و`applications.commands`

    الأذونات الأساسية الشائعة:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (اختياري)

    تجنب `Administrator` ما لم تكن هناك حاجة صريحة إليه.

  </Accordion>

  <Accordion title="نسخ المعرّفات">
    فعّل Discord Developer Mode، ثم انسخ:

    - معرّف الخادم
    - معرّف القناة
    - معرّف المستخدم

    فضّل المعرّفات الرقمية في إعدادات OpenClaw لإجراء التدقيقات وعمليات الفحص بشكل موثوق.

  </Accordion>
</AccordionGroup>

## الأوامر الأصلية ومصادقة الأوامر

- الإعداد الافتراضي لـ `commands.native` هو `"auto"` وهو مفعّل لـ Discord.
- التجاوز لكل قناة: `channels.discord.commands.native`.
- يؤدي `commands.native=false` إلى مسح أوامر Discord الأصلية المسجلة سابقًا صراحةً.
- تستخدم مصادقة الأوامر الأصلية قوائم السماح/السياسات نفسها في Discord كما في معالجة الرسائل العادية.
- قد تظل الأوامر مرئية في واجهة Discord للمستخدمين غير المصرح لهم؛ لكن التنفيذ يظل يفرض مصادقة OpenClaw ويعيد "not authorized".

راجع [Slash commands](/tools/slash-commands) للاطلاع على فهرس الأوامر وسلوكها.

إعدادات أوامر الشرطة المائلة الافتراضية:

- `ephemeral: true`

## تفاصيل الميزات

<AccordionGroup>
  <Accordion title="وسوم الردود والردود الأصلية">
    يدعم Discord وسوم الردود في مخرجات الوكيل:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    يتحكم فيها `channels.discord.replyToMode`:

    - `off` (الافتراضي)
    - `first`
    - `all`

    ملاحظة: يؤدي `off` إلى تعطيل التسلسل الضمني للردود. لكن لا يزال يتم احترام وسوم `[[reply_to_*]]` الصريحة.

    يتم إظهار معرّفات الرسائل في السياق/السجل بحيث يمكن للوكلاء استهداف رسائل محددة.

  </Accordion>

  <Accordion title="معاينة البث المباشر">
    يمكن لـ OpenClaw بث مسودات الردود عبر إرسال رسالة مؤقتة وتحريرها مع وصول النص.

    - يتحكم `channels.discord.streaming` في بث المعاينة (`off` | `partial` | `block` | `progress`، الافتراضي: `off`).
    - يظل الافتراضي `off` لأن تعديلات معاينة Discord قد تصطدم بسرعة بحدود المعدل، خاصة عندما تشترك عدة بوتات أو بوابات في الحساب نفسه أو حركة مرور الخادم نفسها.
    - يتم قبول `progress` لتحقيق الاتساق بين القنوات ويتم تعيينه إلى `partial` على Discord.
    - `channels.discord.streamMode` اسم قديم بديل ويتم ترحيله تلقائيًا.
    - يقوم `partial` بتحرير رسالة معاينة واحدة مع وصول الرموز.
    - يخرج `block` أجزاء بحجم المسودة (استخدم `draftChunk` لضبط الحجم ونقاط الفصل).

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

    القيم الافتراضية لتقسيم الأجزاء في وضع `block` (مقيدة بـ `channels.discord.textChunkLimit`):

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

    يكون بث المعاينة للنص فقط؛ وتعود الردود الإعلامية إلى التسليم العادي.

    ملاحظة: بث المعاينة منفصل عن بث الكتل. عندما يكون بث الكتل مفعّلًا صراحةً
    لـ Discord، يتخطى OpenClaw بث المعاينة لتجنب البث المزدوج.

  </Accordion>

  <Accordion title="السجل والسياق وسلوك السلاسل">
    سياق سجل الخادم:

    - `channels.discord.historyLimit` الافتراضي `20`
    - الرجوع: `messages.groupChat.historyLimit`
    - القيمة `0` تعني التعطيل

    عناصر التحكم في سجل الرسائل الخاصة:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    سلوك السلاسل:

    - يتم توجيه سلاسل Discord كجلسات قنوات
    - يمكن استخدام بيانات تعريف السلسلة الأصلية لربط الجلسة بالأصل
    - يرث إعداد السلسلة إعداد القناة الأصلية ما لم توجد إدخالة خاصة بالسلسلة

    يتم حقن مواضيع القنوات كسياق **غير موثوق** (وليس كـ system prompt).
    يظل سياق الرد والرسائل المقتبسة حاليًا كما تم استلامه.
    تعمل قوائم السماح في Discord أساسًا على ضبط من يمكنه تشغيل الوكيل، لا كحد كامل لتنقيح السياق التكميلي.

  </Accordion>

  <Accordion title="جلسات مرتبطة بالسلاسل للوكلاء الفرعيين">
    يمكن لـ Discord ربط سلسلة بهدف جلسة بحيث تستمر الرسائل اللاحقة في تلك السلسلة بالتوجيه إلى الجلسة نفسها (بما في ذلك جلسات الوكلاء الفرعيين).

    الأوامر:

    - `/focus <target>` لربط السلسلة الحالية/الجديدة بهدف وكيل فرعي/جلسة
    - `/unfocus` لإزالة ربط السلسلة الحالية
    - `/agents` لعرض عمليات التشغيل النشطة وحالة الربط
    - `/session idle <duration|off>` لفحص/تحديث إلغاء التركيز التلقائي بعد عدم النشاط للروابط المركزة
    - `/session max-age <duration|off>` لفحص/تحديث الحد الأقصى الصلب للعمر للروابط المركزة

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
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    ملاحظات:

    - يضبط `session.threadBindings.*` الإعدادات الافتراضية العامة.
    - يتجاوز `channels.discord.threadBindings.*` سلوك Discord.
    - يجب أن تكون `spawnSubagentSessions` مساوية لـ true لإنشاء/ربط السلاسل تلقائيًا لـ `sessions_spawn({ thread: true })`.
    - يجب أن تكون `spawnAcpSessions` مساوية لـ true لإنشاء/ربط السلاسل تلقائيًا لـ ACP (`/acp spawn ... --thread ...` أو `sessions_spawn({ runtime: "acp", thread: true })`).
    - إذا كانت روابط السلاسل معطلة لحساب ما، فلن تكون `/focus` وعمليات ربط السلاسل ذات الصلة متاحة.

    راجع [Sub-agents](/tools/subagents) و[ACP Agents](/tools/acp-agents) و[Configuration Reference](/gateway/configuration-reference).

  </Accordion>

  <Accordion title="عمليات ربط قنوات ACP المستمرة">
    لمساحات عمل ACP الثابتة "الدائمة"، قم بإعداد روابط ACP مكتوبة على المستوى الأعلى تستهدف محادثات Discord.

    مسار الإعداد:

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

    - يقوم `/acp spawn codex --bind here` بربط قناة Discord الحالية أو السلسلة الحالية في مكانها ويحافظ على توجيه الرسائل المستقبلية إلى جلسة ACP نفسها.
    - قد يعني ذلك أيضًا "بدء جلسة Codex ACP جديدة"، لكنه لا ينشئ سلسلة Discord جديدة بحد ذاته. تظل القناة الحالية هي سطح الدردشة.
    - قد يستمر Codex بالعمل ضمن `cwd` الخاص به أو ضمن مساحة عمل الخلفية الخاصة به على القرص. مساحة العمل هذه هي حالة وقت تشغيل وليست سلسلة Discord.
    - يمكن أن ترث رسائل السلاسل ربط ACP للقناة الأصلية.
    - في قناة أو سلسلة مرتبطة، يعيد `/new` و`/reset` تعيين جلسة ACP نفسها في مكانها.
    - لا تزال روابط السلاسل المؤقتة تعمل ويمكنها تجاوز حل الهدف أثناء نشاطها.
    - لا تكون `spawnAcpSessions` مطلوبة إلا عندما يحتاج OpenClaw إلى إنشاء/ربط سلسلة فرعية عبر `--thread auto|here`. وهي غير مطلوبة لـ `/acp spawn ... --bind here` في القناة الحالية.

    راجع [ACP Agents](/tools/acp-agents) للاطلاع على تفاصيل سلوك الربط.

  </Accordion>

  <Accordion title="إشعارات التفاعلات">
    وضع إشعارات التفاعلات لكل خادم:

    - `off`
    - `own` (الافتراضي)
    - `all`
    - `allowlist` (يستخدم `guilds.<id>.users`)

    يتم تحويل أحداث التفاعل إلى أحداث نظام وربطها بجلسة Discord الموجَّهة.

  </Accordion>

  <Accordion title="تفاعلات الإقرار">
    يرسل `ackReaction` رمزًا تعبيريًا للإقرار بينما يعالج OpenClaw رسالة واردة.

    ترتيب الحل:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - الرجوع إلى الرمز التعبيري لهوية الوكيل (`agents.list[].identity.emoji`، وإلا "👀")

    ملاحظات:

    - يقبل Discord الرموز التعبيرية الموحدة أو أسماء الرموز التعبيرية المخصصة.
    - استخدم `""` لتعطيل التفاعل لقناة أو حساب.

  </Accordion>

  <Accordion title="عمليات الكتابة في الإعدادات">
    تكون عمليات الكتابة في الإعدادات التي تبدأ من القناة مفعّلة افتراضيًا.

    يؤثر ذلك على تدفقات `/config set|unset` (عندما تكون ميزات الأوامر مفعلة).

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

  <Accordion title="وكيل Gateway">
    وجّه حركة مرور Discord gateway WebSocket وعمليات البحث REST عند بدء التشغيل (معرّف التطبيق + حل قائمة السماح) عبر وكيل HTTP(S) باستخدام `channels.discord.proxy`.

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
    فعّل حل PluralKit لربط الرسائل الممررة بهوية عضو النظام:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // optional; needed for private systems
      },
    },
  },
}
```

    ملاحظات:

    - يمكن لقوائم السماح استخدام `pk:<memberId>`
    - تتم مطابقة أسماء عرض الأعضاء بالاسم/slug فقط عندما تكون `channels.discord.dangerouslyAllowNameMatching: true`
    - تستخدم عمليات البحث معرّف الرسالة الأصلي وتكون مقيدة بنافذة زمنية
    - إذا فشل البحث، تُعامل الرسائل الممررة كرسائل بوت ويتم إسقاطها ما لم يكن `allowBots=true`

  </Accordion>

  <Accordion title="إعداد الحالة">
    تُطبّق تحديثات الحالة عند تعيين حقل status أو activity، أو عند تمكين auto presence.

    مثال على الحالة فقط:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    مثال على النشاط (الحالة المخصصة هي نوع النشاط الافتراضي):

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

    مثال على البث:

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

    خريطة أنواع النشاط:

    - 0: Playing
    - 1: Streaming (يتطلب `activityUrl`)
    - 2: Listening
    - 3: Watching
    - 4: Custom (يستخدم نص النشاط باعتباره حالة status؛ والرمز التعبيري اختياري)
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

    تربط الحالة التلقائية توفر وقت التشغيل بحالة Discord: healthy => online، وdegraded أو unknown => idle، وexhausted أو unavailable => dnd. تجاوزات النص الاختيارية:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (يدعم العنصر النائب `{reason}`)

  </Accordion>

  <Accordion title="الموافقات في Discord">
    يدعم Discord معالجة الموافقات المستندة إلى الأزرار في الرسائل الخاصة ويمكنه اختياريًا نشر مطالبات الموافقة في القناة الأصلية.

    مسار الإعداد:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (اختياري؛ يرجع إلى `commands.ownerAllowFrom` عندما يكون ذلك ممكنًا)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
    - `agentFilter` و`sessionFilter` و`cleanupAfterResolve`

    يفعّل Discord موافقات exec الأصلية تلقائيًا عندما يكون `enabled` غير معيّن أو `"auto"` ويمكن حل معتمد واحد على الأقل، سواء من `execApprovals.approvers` أو من `commands.ownerAllowFrom`. لا يستنتج Discord معتمدي exec من `allowFrom` الخاصة بالقناة أو الاسم القديم `dm.allowFrom` أو `defaultTo` الخاصة بالرسائل المباشرة. اضبط `enabled: false` لتعطيل Discord كعميل موافقة أصلي بشكل صريح.

    عندما تكون `target` مساوية لـ `channel` أو `both`، تكون مطالبة الموافقة مرئية في القناة. لا يمكن إلا للمعتمدين الذين تم حلهم استخدام الأزرار؛ ويتلقى المستخدمون الآخرون رفضًا مؤقتًا. تتضمن مطالبات الموافقة نص الأمر، لذا فعّل التسليم إلى القناة فقط في القنوات الموثوقة. إذا تعذر اشتقاق معرّف القناة من مفتاح الجلسة، يعود OpenClaw إلى التسليم عبر الرسائل الخاصة.

    يعرض Discord أيضًا أزرار الموافقة المشتركة التي تستخدمها قنوات دردشة أخرى. يضيف مكيّف Discord الأصلي أساسًا توجيه الرسائل الخاصة للمعتمدين والتوزيع على القنوات.
    عندما تكون هذه الأزرار موجودة، تكون هي تجربة الموافقة الأساسية؛ ويجب على OpenClaw
    تضمين أمر `/approve` يدوي فقط عندما تشير نتيجة الأداة إلى أن
    موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.

    تستخدم مصادقة Gateway لهذا المعالج عقد حل بيانات الاعتماد المشتركة نفسه الذي تستخدمه عملاء Gateway الآخرون:

    - مصادقة محلية مع أولوية env (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` ثم `gateway.auth.*`)
    - في الوضع المحلي، يمكن استخدام `gateway.remote.*` كرجوع فقط عندما لا يكون `gateway.auth.*` معيّنًا؛ تفشل SecretRef المحلية المهيأة ولكن غير المحلولة بشكل مغلق
    - دعم الوضع البعيد عبر `gateway.remote.*` عند الاقتضاء
    - تكون تجاوزات URL آمنة ضد التجاوز: لا تعيد تجاوزات CLI استخدام بيانات اعتماد ضمنية، وتستخدم تجاوزات env بيانات اعتماد env فقط

    سلوك حل الموافقة:

    - يتم حل المعرّفات التي تبدأ بـ `plugin:` عبر `plugin.approval.resolve`.
    - يتم حل المعرّفات الأخرى عبر `exec.approval.resolve`.
    - لا يجري Discord هنا خطوة رجوع إضافية من exec إلى plugin؛ فبادئة
      المعرّف هي التي تحدد أي طريقة gateway يستدعيها.

    تنتهي صلاحية موافقات exec بعد 30 دقيقة افتراضيًا. إذا فشلت الموافقات مع
    معرّفات موافقة غير معروفة، فتحقق من حل المعتمدين، وتمكين الميزات،
    ومن أن نوع معرّف الموافقة الذي تم تسليمه يطابق الطلب المعلق.

    الوثائق ذات الصلة: [Exec approvals](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## الأدوات وبوابات الإجراءات

تتضمن إجراءات رسائل Discord الرسائل، وإدارة القنوات، والإشراف، والحضور، وإجراءات البيانات الوصفية.

أمثلة أساسية:

- المراسلة: `sendMessage` و`readMessages` و`editMessage` و`deleteMessage` و`threadReply`
- التفاعلات: `react` و`reactions` و`emojiList`
- الإشراف: `timeout` و`kick` و`ban`
- الحضور: `setPresence`

توجد بوابات الإجراءات ضمن `channels.discord.actions.*`.

سلوك البوابة الافتراضي:

| مجموعة الإجراءات                                                                                                                                                         | الافتراضي |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | مفعّل     |
| roles                                                                                                                                                                    | معطّل     |
| moderation                                                                                                                                                               | معطّل     |
| presence                                                                                                                                                                 | معطّل     |

## واجهة Components v2

يستخدم OpenClaw Discord components v2 لموافقات exec وعلامات السياق المتقاطع. يمكن لإجراءات رسائل Discord أيضًا قبول `components` لواجهة مستخدم مخصصة (متقدم؛ ويتطلب إنشاء حمولة مكونات عبر أداة discord)، بينما تظل `embeds` القديمة متاحة ولكن لا يُنصح بها.

- يضبط `channels.discord.ui.components.accentColor` لون التمييز المستخدم من قبل حاويات مكونات Discord (hex).
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

يمكن لـ OpenClaw الانضمام إلى قنوات Discord الصوتية لإجراء محادثات آنية ومستمرة. هذا منفصل عن مرفقات الرسائل الصوتية.

المتطلبات:

- فعّل الأوامر الأصلية (`commands.native` أو `channels.discord.commands.native`).
- اضبط `channels.discord.voice`.
- يحتاج البوت إلى أذونات Connect + Speak في القناة الصوتية المستهدفة.

استخدم أمر Discord الأصلي فقط `/vc join|leave|status` للتحكم في الجلسات. يستخدم الأمر الوكيل الافتراضي للحساب ويتبع قواعد قائمة السماح وسياسة المجموعات نفسها كما في أوامر Discord الأخرى.

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
- تستمد أدوار النسخ الصوتي حالة المالك من `allowFrom` في Discord (أو `dm.allowFrom`)؛ لا يمكن للمتحدثين غير المالكين الوصول إلى الأدوات المخصصة للمالك فقط (مثل `gateway` و`cron`).
- يكون الصوت مفعّلًا افتراضيًا؛ اضبط `channels.discord.voice.enabled=false` لتعطيله.
- يمرر `voice.daveEncryption` و`voice.decryptionFailureTolerance` إلى خيارات الانضمام في `@discordjs/voice`.
- القيم الافتراضية لـ `@discordjs/voice` هي `daveEncryption=true` و`decryptionFailureTolerance=24` إذا لم يتم تعيينها.
- يراقب OpenClaw أيضًا فشل فك التشفير عند الاستقبال ويتعافى تلقائيًا عبر مغادرة القناة الصوتية ثم إعادة الانضمام بعد فشل متكرر خلال نافذة قصيرة.
- إذا كانت سجلات الاستقبال تعرض مرارًا `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`، فقد يكون هذا هو خطأ الاستقبال في `@discordjs/voice` المتتبع في [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419).

## الرسائل الصوتية

تعرض الرسائل الصوتية في Discord معاينة للموجة الصوتية وتتطلب صوت OGG/Opus مع بيانات وصفية. ينشئ OpenClaw الموجة الصوتية تلقائيًا، لكنه يحتاج إلى توفر `ffmpeg` و`ffprobe` على مضيف البوابة لفحص الملفات الصوتية وتحويلها.

المتطلبات والقيود:

- قدّم **مسار ملف محلي** (يتم رفض عناوين URL).
- احذف المحتوى النصي (لا يسمح Discord بالنص + رسالة صوتية في الحمولة نفسها).
- يُقبل أي تنسيق صوتي؛ ويحوّله OpenClaw إلى OGG/Opus عند الحاجة.

مثال:

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="تم استخدام أذونات غير مسموح بها أو أن البوت لا يرى رسائل الخادم">

    - فعّل Message Content Intent
    - فعّل Server Members Intent عندما تعتمد على حل المستخدم/العضو
    - أعد تشغيل البوابة بعد تغيير الأذونات

  </Accordion>

  <Accordion title="يتم حظر رسائل الخادم بشكل غير متوقع">

    - تحقق من `groupPolicy`
    - تحقق من قائمة سماح الخادم ضمن `channels.discord.guilds`
    - إذا كانت خريطة `channels` الخاصة بالخادم موجودة، فلا يُسمح إلا بالقنوات المدرجة
    - تحقق من سلوك `requireMention` وأنماط الإشارة

    فحوصات مفيدة:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="تم تعيين require mention إلى false لكنه لا يزال محظورًا">
    الأسباب الشائعة:

    - `groupPolicy="allowlist"` من دون قائمة سماح مطابقة للخادم/القناة
    - تم إعداد `requireMention` في المكان الخاطئ (يجب أن يكون ضمن `channels.discord.guilds` أو إدخال القناة)
    - تم حظر المرسل بواسطة قائمة السماح `users` للخادم/القناة

  </Accordion>

  <Accordion title="تنتهي مهلة المعالجات طويلة التشغيل أو تتكرر الردود">

    السجلات النموذجية:

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    مقبض ميزانية المستمع:

    - حساب واحد: `channels.discord.eventQueue.listenerTimeout`
    - حسابات متعددة: `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    مقبض مهلة تشغيل العامل:

    - حساب واحد: `channels.discord.inboundWorker.runTimeoutMs`
    - حسابات متعددة: `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - الافتراضي: `1800000` (30 دقيقة)؛ اضبطه إلى `0` للتعطيل

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
    لا تعمل عمليات فحص الأذونات في `channels status --probe` إلا مع معرّفات القنوات الرقمية.

    إذا كنت تستخدم مفاتيح slug، فقد يظل التطابق في وقت التشغيل يعمل، لكن الفحص لا يمكنه التحقق بالكامل من الأذونات.

  </Accordion>

  <Accordion title="مشكلات الرسائل الخاصة والاقتران">

    - الرسائل الخاصة معطلة: `channels.discord.dm.enabled=false`
    - سياسة الرسائل الخاصة معطلة: `channels.discord.dmPolicy="disabled"` (الاسم القديم: `channels.discord.dm.policy`)
    - بانتظار الموافقة على الاقتران في وضع `pairing`

  </Accordion>

  <Accordion title="حلقات بوت إلى بوت">
    افتراضيًا، يتم تجاهل الرسائل التي كتبها البوت.

    إذا قمت بتعيين `channels.discord.allowBots=true`، فاستخدم قواعد صارمة للإشارة وقائمة السماح لتجنب سلوك الحلقات.
    ويفضّل استخدام `channels.discord.allowBots="mentions"` لقبول رسائل البوت التي تشير إلى البوت فقط.

  </Accordion>

  <Accordion title="انقطاع STT الصوتي مع DecryptionFailed(...)">

    - حافظ على تحديث OpenClaw (`openclaw update`) حتى يكون منطق التعافي من استقبال Discord الصوتي موجودًا
    - أكّد أن `channels.discord.voice.daveEncryption=true` (الافتراضي)
    - ابدأ من `channels.discord.voice.decryptionFailureTolerance=24` (الافتراضي upstream) واضبط فقط عند الحاجة
    - راقب السجلات بحثًا عن:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - إذا استمرت الأعطال بعد إعادة الانضمام التلقائية، فاجمع السجلات وقارنها مع [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## مؤشرات مرجع الإعدادات

المرجع الأساسي:

- [مرجع الإعدادات - Discord](/gateway/configuration-reference#discord)

حقول Discord عالية الأهمية:

- بدء التشغيل/المصادقة: `enabled` و`token` و`accounts.*` و`allowBots`
- السياسة: `groupPolicy` و`dm.*` و`guilds.*` و`guilds.*.channels.*`
- الأوامر: `commands.native` و`commands.useAccessGroups` و`configWrites` و`slashCommand.*`
- قائمة انتظار الأحداث: `eventQueue.listenerTimeout` (ميزانية المستمع) و`eventQueue.maxQueueSize` و`eventQueue.maxConcurrency`
- العامل الوارد: `inboundWorker.runTimeoutMs`
- الرد/السجل: `replyToMode` و`historyLimit` و`dmHistoryLimit` و`dms.*.historyLimit`
- التسليم: `textChunkLimit` و`chunkMode` و`maxLinesPerMessage`
- البث: `streaming` (الاسم القديم البديل: `streamMode`) و`draftChunk` و`blockStreaming` و`blockStreamingCoalesce`
- الوسائط/إعادة المحاولة: `mediaMaxMb` و`retry`
  - يحدد `mediaMaxMb` الحد الأقصى للتحميلات الصادرة إلى Discord (الافتراضي: `8MB`)
- الإجراءات: `actions.*`
- الحالة: `activity` و`status` و`activityType` و`activityUrl`
- واجهة المستخدم: `ui.components.accentColor`
- الميزات: `threadBindings` و`bindings[]` على المستوى الأعلى (`type: "acp"`) و`pluralkit` و`execApprovals` و`intents` و`agentComponents` و`heartbeat` و`responsePrefix`

## السلامة والعمليات

- تعامل مع رموز البوت المميزة على أنها أسرار (ويُفضّل `DISCORD_BOT_TOKEN` في البيئات المُدارة).
- امنح أقل قدر لازم من أذونات Discord.
- إذا كانت حالة نشر/حالة الأوامر قديمة، فأعد تشغيل البوابة وأعد التحقق باستخدام `openclaw channels status --probe`.

## ذو صلة

- [الاقتران](/channels/pairing)
- [المجموعات](/channels/groups)
- [توجيه القنوات](/channels/channel-routing)
- [الأمان](/gateway/security)
- [التوجيه متعدد الوكلاء](/concepts/multi-agent)
- [استكشاف الأخطاء وإصلاحها](/channels/troubleshooting)
- [أوامر الشرطة المائلة](/tools/slash-commands)
