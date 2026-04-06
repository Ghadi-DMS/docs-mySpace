---
read_when:
    - عند إعداد Slack أو تصحيح أخطاء وضع socket/HTTP في Slack
summary: إعداد Slack وسلوك وقت التشغيل (Socket Mode + عناوين URL لطلبات HTTP)
title: Slack
x-i18n:
    generated_at: "2026-04-06T07:18:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 471421a34e5ff20dfc46dab85422d2e814524068c8466d0738448cd5d64d415b
    source_path: channels/slack.md
    workflow: 15
---

# Slack

الحالة: جاهز للإنتاج للرسائل المباشرة والقنوات عبر تكاملات تطبيق Slack. الوضع الافتراضي هو Socket Mode؛ كما أن عناوين URL لطلبات HTTP مدعومة أيضًا.

<CardGroup cols={3}>
  <Card title="الإقران" icon="link" href="/ar/channels/pairing">
    تكون الرسائل المباشرة في Slack في وضع الإقران افتراضيًا.
  </Card>
  <Card title="أوامر الشرطة المائلة" icon="terminal" href="/ar/tools/slash-commands">
    سلوك الأوامر الأصلي وكتالوج الأوامر.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/ar/channels/troubleshooting">
    تشخيصات عبر القنوات وأدلة الإصلاح.
  </Card>
</CardGroup>

## الإعداد السريع

<Tabs>
  <Tab title="Socket Mode (الافتراضي)">
    <Steps>
      <Step title="أنشئ تطبيق Slack والرموز المميزة">
        في إعدادات تطبيق Slack:

        - فعّل **Socket Mode**
        - أنشئ **App Token** (`xapp-...`) مع `connections:write`
        - ثبّت التطبيق وانسخ **Bot Token** (`xoxb-...`)
      </Step>

      <Step title="اضبط OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

        الرجوع إلى متغيرات البيئة (للحساب الافتراضي فقط):

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      </Step>

      <Step title="اشترك في أحداث التطبيق">
        اشترك في أحداث البوت التالية:

        - `app_mention`
        - `message.channels`, `message.groups`, `message.im`, `message.mpim`
        - `reaction_added`, `reaction_removed`
        - `member_joined_channel`, `member_left_channel`
        - `channel_rename`
        - `pin_added`, `pin_removed`

        فعّل أيضًا **Messages Tab** في App Home للرسائل المباشرة.
      </Step>

      <Step title="ابدأ البوابة">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="عناوين URL لطلبات HTTP">
    <Steps>
      <Step title="اضبط تطبيق Slack لـ HTTP">

        - عيّن الوضع إلى HTTP (`channels.slack.mode="http"`)
        - انسخ **Signing Secret** الخاص بـ Slack
        - عيّن عنوان URL للطلب لكل من اشتراكات الأحداث والتفاعلية وأمر الشرطة المائلة إلى مسار webhook نفسه (الافتراضي `/slack/events`)

      </Step>

      <Step title="اضبط وضع HTTP في OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events",
    },
  },
}
```

      </Step>

      <Step title="استخدم مسارات webhook فريدة لـ HTTP متعدد الحسابات">
        وضع HTTP لكل حساب مدعوم.

        امنح كل حساب `webhookPath` مميزًا حتى لا تتعارض عمليات التسجيل.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## قائمة التحقق من البيان والنطاقات

<AccordionGroup>
  <Accordion title="مثال على بيان تطبيق Slack" defaultOpen>

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": true
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

  </Accordion>

  <Accordion title="نطاقات user-token الاختيارية (عمليات القراءة)">
    إذا قمت بضبط `channels.slack.userToken`، فإن نطاقات القراءة الشائعة هي:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (إذا كنت تعتمد على قراءات بحث Slack)

  </Accordion>
</AccordionGroup>

## نموذج الرموز المميزة

- `botToken` + `appToken` مطلوبان لـ Socket Mode.
- يتطلب وضع HTTP `botToken` + `signingSecret`.
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل
  نصية عادية أو كائنات SecretRef.
- تتجاوز الرموز المميزة في الإعدادات الرجوع إلى متغيرات البيئة.
- ينطبق الرجوع إلى متغيرات البيئة `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` على الحساب الافتراضي فقط.
- `userToken` (`xoxp-...`) متاح في الإعدادات فقط (من دون رجوع إلى متغيرات البيئة) ويكون افتراضيًا بسلوك للقراءة فقط (`userTokenReadOnly: true`).
- اختياري: أضف `chat:write.customize` إذا أردت أن تستخدم الرسائل الصادرة هوية العامل النشط (مع `username` وأيقونة مخصصين). يستخدم `icon_emoji` صيغة `:emoji_name:`.

سلوك لقطة الحالة:

- يتتبع فحص حساب Slack حقول `*Source` و`*Status` لكل بيانات اعتماد
  (`botToken`, `appToken`, `signingSecret`, `userToken`).
- تكون الحالة `available` أو `configured_unavailable` أو `missing`.
- تعني `configured_unavailable` أن الحساب مضبوط عبر SecretRef
  أو مصدر أسرار آخر غير مضمن، لكن مسار الأمر/وقت التشغيل الحالي
  لم يتمكن من حل القيمة الفعلية.
- في وضع HTTP، يتم تضمين `signingSecretStatus`؛ وفي Socket Mode،
  يكون الزوج المطلوب هو `botTokenStatus` + `appTokenStatus`.

<Tip>
بالنسبة إلى الإجراءات/قراءات الدليل، يمكن تفضيل user token عند ضبطه. بالنسبة إلى عمليات الكتابة، يظل bot token هو المفضل؛ ولا يُسمح بعمليات الكتابة عبر user-token إلا عندما تكون `userTokenReadOnly: false` وكان bot token غير متاح.
</Tip>

## الإجراءات والقيود

يتم التحكم في إجراءات Slack بواسطة `channels.slack.actions.*`.

مجموعات الإجراءات المتاحة في أدوات Slack الحالية:

| المجموعة | الافتراضي |
| ---------- | ------- |
| messages   | مفعّل |
| reactions  | مفعّل |
| pins       | مفعّل |
| memberInfo | مفعّل |
| emojiList  | مفعّل |

تتضمن إجراءات رسائل Slack الحالية `send` و`upload-file` و`download-file` و`read` و`edit` و`delete` و`pin` و`unpin` و`list-pins` و`member-info` و`emoji-list`.

## التحكم في الوصول والتوجيه

<Tabs>
  <Tab title="سياسة الرسائل المباشرة">
    يتحكم `channels.slack.dmPolicy` في وصول الرسائل المباشرة (قديمًا: `channels.slack.dm.policy`):

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن يتضمن `channels.slack.allowFrom` القيمة `"*"`؛ قديمًا: `channels.slack.dm.allowFrom`)
    - `disabled`

    علامات الرسائل المباشرة:

    - `dm.enabled` (الافتراضي true)
    - `channels.slack.allowFrom` (المفضل)
    - `dm.allowFrom` (قديم)
    - `dm.groupEnabled` (الرسائل المباشرة الجماعية افتراضيًا false)
    - `dm.groupChannels` (قائمة سماح MPIM اختيارية)

    أولوية تعدد الحسابات:

    - ينطبق `channels.slack.accounts.default.allowFrom` على الحساب `default` فقط.
    - ترث الحسابات المسماة `channels.slack.allowFrom` عندما لا تكون قيمة `allowFrom` الخاصة بها مضبوطة.
    - لا ترث الحسابات المسماة `channels.slack.accounts.default.allowFrom`.

    يستخدم الإقران في الرسائل المباشرة `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="سياسة القنوات">
    يتحكم `channels.slack.groupPolicy` في التعامل مع القنوات:

    - `open`
    - `allowlist`
    - `disabled`

    توجد قائمة السماح الخاصة بالقنوات تحت `channels.slack.channels` ويجب أن تستخدم معرّفات قنوات مستقرة.

    ملاحظة وقت التشغيل: إذا كان `channels.slack` مفقودًا بالكامل (إعداد يعتمد على البيئة فقط)، فإن وقت التشغيل يرجع إلى `groupPolicy="allowlist"` ويسجل تحذيرًا (حتى إذا كانت `channels.defaults.groupPolicy` مضبوطة).

    تحليل الاسم/المعرّف:

    - يتم تحليل إدخالات قائمة سماح القنوات وإدخالات قائمة سماح الرسائل المباشرة عند بدء التشغيل عندما يسمح الوصول إلى الرمز المميز بذلك
    - يتم الاحتفاظ بإدخالات أسماء القنوات غير المحلولة كما تم ضبطها ولكن يتم تجاهلها افتراضيًا في التوجيه
    - يكون التفويض الوارد وتوجيه القنوات معتمدين على المعرّف أولًا افتراضيًا؛ وتتطلب المطابقة المباشرة لاسم المستخدم/slug القيمة `channels.slack.dangerouslyAllowNameMatching: true`

  </Tab>

  <Tab title="الإشارات ومستخدمو القنوات">
    تكون رسائل القنوات مقيّدة بالإشارة افتراضيًا.

    مصادر الإشارة:

    - إشارة صريحة للتطبيق (`<@botId>`)
    - أنماط regex للإشارة (`agents.list[].groupChat.mentionPatterns`، والرجوع إلى `messages.groupChat.mentionPatterns`)
    - سلوك ضمني لسلسلة الرد على البوت

    عناصر التحكم لكل قناة (`channels.slack.channels.<id>`؛ الأسماء فقط عبر التحليل عند بدء التشغيل أو `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `users` (قائمة سماح)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - صيغة مفتاح `toolsBySender`: ‏`id:` أو `e164:` أو `username:` أو `name:` أو حرف البدل `"*"`
      (المفاتيح القديمة غير المسبوقة تُطابق `id:` فقط)

  </Tab>
</Tabs>

## التسلسل الخيطي والجلسات وعلامات الرد

- يتم توجيه الرسائل المباشرة كـ `direct`؛ والقنوات كـ `channel`؛ وMPIMs كـ `group`.
- مع الافتراضي `session.dmScope=main`، تنهار الرسائل المباشرة في Slack إلى الجلسة الرئيسية للعامل.
- جلسات القنوات: `agent:<agentId>:slack:channel:<channelId>`.
- يمكن أن تنشئ ردود السلاسل لاحقة جلسة سلسلة (`:thread:<threadTs>`) عند الاقتضاء.
- القيمة الافتراضية لـ `channels.slack.thread.historyScope` هي `thread`؛ والقيمة الافتراضية لـ `thread.inheritParent` هي `false`.
- يتحكم `channels.slack.thread.initialHistoryLimit` في عدد رسائل السلسلة الموجودة التي يتم جلبها عند بدء جلسة سلسلة جديدة (الافتراضي `20`؛ اضبطه إلى `0` للتعطيل).

عناصر التحكم في تسلسل الردود:

- `channels.slack.replyToMode`: ‏`off|first|all|batched` (الافتراضي `off`)
- `channels.slack.replyToModeByChatType`: لكل من `direct|group|channel`
- الرجوع القديم للمحادثات المباشرة: `channels.slack.dm.replyToMode`

علامات الرد اليدوية مدعومة:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

ملاحظة: يؤدي `replyToMode="off"` إلى تعطيل **كل** تسلسل الردود في Slack، بما في ذلك العلامات الصريحة `[[reply_to_*]]`. يختلف هذا عن Telegram، حيث لا تزال العلامات الصريحة محترمة في وضع `"off"`. يعكس هذا الاختلاف نماذج التسلسل الخيطي في المنصتين: تخفي سلاسل Slack الرسائل عن القناة، بينما تظل ردود Telegram مرئية في تدفق المحادثة الرئيسي.

## تفاعلات الإقرار

يرسل `ackReaction` رمزًا تعبيريًا للإقرار بينما يعالج OpenClaw رسالة واردة.

ترتيب التحليل:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- الرجوع إلى رمز هوية العامل التعبيري (`agents.list[].identity.emoji`، وإلا "👀")

ملاحظات:

- يتوقع Slack shortcodes (على سبيل المثال `"eyes"`).
- استخدم `""` لتعطيل التفاعل لهذا الحساب في Slack أو بشكل عام.

## بث النص

يتحكم `channels.slack.streaming` في سلوك المعاينة المباشرة:

- `off`: تعطيل بث المعاينة المباشرة.
- `partial` (الافتراضي): استبدال نص المعاينة بأحدث إخراج جزئي.
- `block`: إلحاق تحديثات معاينة مجزأة.
- `progress`: إظهار نص حالة التقدم أثناء الإنشاء، ثم إرسال النص النهائي.

يتحكم `channels.slack.nativeStreaming` في البث النصي الأصلي في Slack عندما تكون قيمة `streaming` هي `partial` (الافتراضي: `true`).

- يجب أن يكون تسلسل الرد متاحًا حتى يظهر البث النصي الأصلي. ولا يزال اختيار التسلسل يتبع `replyToMode`. وبدونه، تُستخدم معاينة المسودة العادية.
- ترجع الوسائط والحمولات غير النصية إلى التسليم العادي.
- إذا فشل البث أثناء الرد، يعود OpenClaw إلى التسليم العادي للحمولات المتبقية.

استخدم معاينة المسودة بدلًا من البث النصي الأصلي في Slack:

```json5
{
  channels: {
    slack: {
      streaming: "partial",
      nativeStreaming: false,
    },
  },
}
```

المفاتيح القديمة:

- يتم ترحيل `channels.slack.streamMode` (`replace | status_final | append`) تلقائيًا إلى `channels.slack.streaming`.
- يتم ترحيل القيمة المنطقية `channels.slack.streaming` تلقائيًا إلى `channels.slack.nativeStreaming`.

## الرجوع إلى تفاعل الكتابة

يضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة بينما يعالج OpenClaw ردًا، ثم يزيله عند انتهاء التشغيل. وهذا مفيد أكثر خارج ردود السلاسل، التي تستخدم مؤشر حالة افتراضيًا من نوع "يكتب الآن...".

ترتيب التحليل:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

ملاحظات:

- يتوقع Slack shortcodes (على سبيل المثال `"hourglass_flowing_sand"`).
- يكون التفاعل على أساس أفضل جهد، وتتم محاولة التنظيف تلقائيًا بعد اكتمال الرد أو مسار الفشل.

## الوسائط والتقسيم والتسليم

<AccordionGroup>
  <Accordion title="المرفقات الواردة">
    يتم تنزيل مرفقات ملفات Slack من عناوين URL خاصة مستضافة على Slack (تدفق طلبات موثّق بالرمز المميز) وكتابتها إلى مخزن الوسائط عند نجاح الجلب والسماح بحدود الحجم.

    يكون الحد الأقصى الافتراضي للحجم الوارد أثناء التشغيل `20MB` ما لم يتم تجاوزه بواسطة `channels.slack.mediaMaxMb`.

  </Accordion>

  <Accordion title="النصوص والملفات الصادرة">
    - تستخدم مقاطع النص `channels.slack.textChunkLimit` (الافتراضي 4000)
    - يمكّن `channels.slack.chunkMode="newline"` التقسيم أولًا حسب الفقرات
    - تستخدم عمليات إرسال الملفات واجهات رفع Slack API ويمكن أن تتضمن ردود السلاسل (`thread_ts`)
    - يتبع الحد الأقصى للوسائط الصادرة `channels.slack.mediaMaxMb` عند ضبطه؛ وإلا تستخدم عمليات الإرسال عبر القناة القيم الافتراضية حسب نوع MIME من مسار الوسائط
  </Accordion>

  <Accordion title="أهداف التسليم">
    الأهداف الصريحة المفضلة:

    - `user:<id>` للرسائل المباشرة
    - `channel:<id>` للقنوات

    يتم فتح الرسائل المباشرة في Slack عبر واجهات Slack conversation API عند الإرسال إلى أهداف المستخدمين.

  </Accordion>
</AccordionGroup>

## الأوامر وسلوك الشرطة المائلة

- يكون الوضع التلقائي للأوامر الأصلية **معطّلًا** في Slack (`commands.native: "auto"` لا يفعّل أوامر Slack الأصلية).
- فعّل معالجات أوامر Slack الأصلية باستخدام `channels.slack.commands.native: true` (أو عامًّا `commands.native: true`).
- عند تمكين الأوامر الأصلية، سجّل أوامر الشرطة المائلة المطابقة في Slack (أسماء `/<command>`)، مع استثناء واحد:
  - سجّل `/agentstatus` لأمر الحالة (يحجز Slack الأمر `/status`)
- إذا لم تكن الأوامر الأصلية مفعلة، يمكنك تشغيل أمر شرطة مائلة واحد مضبوط عبر `channels.slack.slashCommand`.
- تتكيف قوائم الوسائط الأصلية للوسائط الآن في استراتيجية العرض:
  - حتى 5 خيارات: كتل أزرار
  - من 6 إلى 100 خيار: قائمة تحديد ثابتة
  - أكثر من 100 خيار: تحديد خارجي مع تصفية خيارات غير متزامنة عندما تكون معالجات خيارات التفاعلية متاحة
  - إذا تجاوزت قيم الخيارات المرمّزة حدود Slack، يعود التدفق إلى الأزرار
- بالنسبة إلى حمولات الخيارات الطويلة، تستخدم قوائم وسائط أوامر الشرطة المائلة مربع حوار تأكيد قبل إرسال القيمة المحددة.

إعدادات أمر الشرطة المائلة الافتراضية:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

تستخدم جلسات أوامر الشرطة المائلة مفاتيح معزولة:

- `agent:<agentId>:slack:slash:<userId>`

ومع ذلك لا يزال توجيه تنفيذ الأوامر يتم على جلسة المحادثة المستهدفة (`CommandTargetSessionKey`).

## الردود التفاعلية

يمكن لـ Slack عرض عناصر تحكم للردود التفاعلية التي ينشئها العامل، لكن هذه الميزة معطلة افتراضيًا.

فعّلها على مستوى عام:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

أو فعّلها لحساب Slack واحد فقط:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

عند التمكين، يمكن للعوامل إصدار توجيهات رد خاصة بـ Slack فقط:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

تُجمَّع هذه التوجيهات إلى Slack Block Kit وتعيد تمرير النقرات أو التحديدات عبر مسار حدث تفاعل Slack الحالي.

ملاحظات:

- هذه واجهة مستخدم خاصة بـ Slack. لا تترجم القنوات الأخرى توجيهات Slack Block Kit إلى أنظمة الأزرار الخاصة بها.
- قيم ردود النداء التفاعلية هي رموز معتمة ينشئها OpenClaw، وليست قيمًا خامًا أنشأها العامل.
- إذا كانت الكتل التفاعلية المولدة ستتجاوز حدود Slack Block Kit، فإن OpenClaw يعود إلى الرد النصي الأصلي بدلًا من إرسال حمولة كتل غير صالحة.

## موافقات exec في Slack

يمكن أن يعمل Slack كعميل موافقات أصلي مع أزرار وتفاعلات تفاعلية، بدلًا من الرجوع إلى واجهة الويب أو الطرفية.

- تستخدم موافقات exec المسار `channels.slack.execApprovals.*` للتوجيه الأصلي في الرسائل المباشرة/القنوات.
- لا تزال موافقات plugin قادرة على الحل عبر سطح أزرار Slack الأصلي نفسه عندما يصل الطلب بالفعل إلى Slack ويكون نوع معرّف الموافقة هو `plugin:`.
- لا يزال تفويض الموافقين مفروضًا: يمكن فقط للمستخدمين المحددين كموافقين الموافقة على الطلبات أو رفضها عبر Slack.

يستخدم هذا سطح أزرار الموافقة المشتركة نفسه المستخدم في القنوات الأخرى. عندما تكون `interactivity` مفعلة في إعدادات تطبيق Slack، يتم عرض مطالبات الموافقة كأزرار Block Kit مباشرة داخل المحادثة.
وعندما تكون هذه الأزرار موجودة، فإنها تكون تجربة الموافقة الأساسية؛ ويجب على OpenClaw
ألا يضمّن أمر `/approve` يدويًا إلا عندما تشير نتيجة الأداة إلى أن
الموافقات داخل المحادثة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.

مسار الإعداد:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (اختياري؛ يرجع إلى `commands.ownerAllowFrom` عندما يكون ذلك ممكنًا)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `agentFilter`, `sessionFilter`

يفعّل Slack موافقات exec الأصلية تلقائيًا عندما تكون `enabled` غير مضبوطة أو `"auto"` وعندما يتم تحليل موافق واحد على الأقل. اضبط `enabled: false` لتعطيل Slack كعميل موافقات أصلي بشكل صريح.
واضبط `enabled: true` لفرض تفعيل الموافقات الأصلية عند تحليل الموافقين.

السلوك الافتراضي من دون إعداد Slack صريح لموافقات exec:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

يلزم الإعداد الأصلي الصريح لـ Slack فقط عندما تريد تجاوز الموافقين أو إضافة عوامل تصفية أو
اختيار التسليم إلى محادثة المصدر:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

يكون تمرير `approvals.exec` المشترك منفصلًا. استخدمه فقط عندما يجب أيضًا
توجيه مطالبات موافقة exec إلى محادثات أخرى أو أهداف صريحة خارج النطاق. كما أن تمرير `approvals.plugin` المشترك
منفصل أيضًا؛ ولا تزال الأزرار الأصلية في Slack قادرة على حل موافقات plugin عندما تصل هذه الطلبات بالفعل
إلى Slack.

يعمل `/approve` ضمن المحادثة نفسها أيضًا في قنوات Slack والرسائل المباشرة التي تدعم الأوامر بالفعل. راجع [موافقات Exec](/ar/tools/exec-approvals) للحصول على نموذج تمرير الموافقات الكامل.

## الأحداث والسلوك التشغيلي

- يتم تعيين تعديلات الرسائل/حذفها/بث السلاسل إلى أحداث نظام.
- يتم تعيين أحداث إضافة/إزالة التفاعلات إلى أحداث نظام.
- يتم تعيين أحداث انضمام/مغادرة الأعضاء، وإنشاء/إعادة تسمية القنوات، وإضافة/إزالة التثبيت إلى أحداث نظام.
- يمكن لـ `channel_id_changed` ترحيل مفاتيح إعداد القناة عندما تكون `configWrites` مفعلة.
- يتم التعامل مع بيانات وصف/غرض القناة الوصفية على أنها سياق غير موثوق ويمكن حقنها في سياق التوجيه.
- تتم تصفية بادئ السلسلة وسياق تهيئة سجل السلسلة الأولي بواسطة قوائم السماح المكوّنة للمرسلين عند الاقتضاء.
- تُصدر إجراءات الكتل وتفاعلات النوافذ المشروطة أحداث نظام مهيكلة بصيغة `Slack interaction: ...` مع حقول حمولة غنية:
  - إجراءات الكتل: القيم المحددة، والتسميات، وقيم أدوات الاختيار، وبيانات `workflow_*` الوصفية
  - أحداث `view_submission` و`view_closed` للنوافذ المشروطة مع بيانات القناة الموجّهة ومدخلات النماذج

## مؤشرات مرجعية للإعدادات

المرجع الأساسي:

- [مرجع الإعدادات - Slack](/ar/gateway/configuration-reference#slack)

  حقول Slack عالية الإشارة:
  - الوضع/المصادقة: `mode`, `botToken`, `appToken`, `signingSecret`, `webhookPath`, `accounts.*`
  - وصول الرسائل المباشرة: `dm.enabled`, `dmPolicy`, `allowFrom` (قديمًا: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
  - مفتاح التوافق: `dangerouslyAllowNameMatching` (للحالات الطارئة؛ اتركه معطلًا ما لم تكن هناك حاجة)
  - وصول القنوات: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`
  - التسلسل/السجل: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
  - التسليم: `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `streaming`, `nativeStreaming`
  - التشغيل/الميزات: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لا توجد ردود في القنوات">
    تحقّق، بالترتيب، من:

    - `groupPolicy`
    - قائمة سماح القنوات (`channels.slack.channels`)
    - `requireMention`
    - قائمة سماح `users` لكل قناة

    أوامر مفيدة:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="يتم تجاهل رسائل الرسائل المباشرة">
    تحقّق من:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (أو القديم `channels.slack.dm.policy`)
    - موافقات الإقران / إدخالات قائمة السماح

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="وضع Socket لا يتصل">
    تحقّق من صحة bot token وapp token ومن تمكين Socket Mode في إعدادات تطبيق Slack.

    إذا أظهر `openclaw channels status --probe --json` القيمة `botTokenStatus` أو
    `appTokenStatus: "configured_unavailable"`، فهذا يعني أن حساب Slack
    مضبوط، لكن وقت التشغيل الحالي لم يتمكن من حل القيمة
    المدعومة بـ SecretRef.

  </Accordion>

  <Accordion title="وضع HTTP لا يستقبل الأحداث">
    تحقّق من:

    - signing secret
    - مسار webhook
    - عناوين URL لطلبات Slack (الأحداث + التفاعلية + أوامر الشرطة المائلة)
    - `webhookPath` فريد لكل حساب HTTP

    إذا ظهر `signingSecretStatus: "configured_unavailable"` في لقطات
    الحساب، فهذا يعني أن حساب HTTP مضبوط، لكن وقت التشغيل الحالي لم يتمكن من
    حل signing secret المدعوم بـ SecretRef.

  </Accordion>

  <Accordion title="الأوامر الأصلية/أوامر الشرطة المائلة لا تعمل">
    تحقّق مما إذا كنت تقصد:

    - وضع الأوامر الأصلية (`channels.slack.commands.native: true`) مع تسجيل أوامر الشرطة المائلة المطابقة في Slack
    - أو وضع أمر الشرطة المائلة الواحد (`channels.slack.slashCommand.enabled: true`)

    تحقّق أيضًا من `commands.useAccessGroups` وقوائم السماح الخاصة بالقنوات/المستخدمين.

  </Accordion>
</AccordionGroup>

## ذو صلة

- [الإقران](/ar/channels/pairing)
- [المجموعات](/ar/channels/groups)
- [الأمان](/ar/gateway/security)
- [توجيه القنوات](/ar/channels/channel-routing)
- [استكشاف الأخطاء وإصلاحها](/ar/channels/troubleshooting)
- [الإعدادات](/ar/gateway/configuration)
- [أوامر الشرطة المائلة](/ar/tools/slash-commands)
