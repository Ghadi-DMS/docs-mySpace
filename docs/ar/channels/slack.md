---
read_when:
    - إعداد Slack أو تصحيح أخطاء وضع Socket/HTTP في Slack
summary: إعداد Slack وسلوك وقت التشغيل (Socket Mode + HTTP Events API)
title: Slack
x-i18n:
    generated_at: "2026-04-06T03:07:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7e4ff2ce7d92276d62f4f3d3693ddb56ca163d5fdc2f1082ff7ba3421fada69c
    source_path: channels/slack.md
    workflow: 15
---

# Slack

الحالة: جاهز للإنتاج للرسائل الخاصة + القنوات عبر تكاملات تطبيق Slack. الوضع الافتراضي هو Socket Mode؛ كما أن وضع HTTP Events API مدعوم أيضًا.

<CardGroup cols={3}>
  <Card title="الاقتران" icon="link" href="/ar/channels/pairing">
    تستخدم الرسائل الخاصة في Slack وضع الاقتران بشكل افتراضي.
  </Card>
  <Card title="أوامر الشرطة المائلة" icon="terminal" href="/ar/tools/slash-commands">
    سلوك الأوامر الأصلي وفهرس الأوامر.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/ar/channels/troubleshooting">
    تشخيصات متعددة القنوات وأدلة الإصلاح.
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

        متغيرات البيئة الاحتياطية (للحساب الافتراضي فقط):

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

        فعّل أيضًا App Home **Messages Tab** للرسائل الخاصة.
      </Step>

      <Step title="ابدأ البوابة">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="وضع HTTP Events API">
    <Steps>
      <Step title="اضبط تطبيق Slack لاستخدام HTTP">

        - اضبط الوضع على HTTP (`channels.slack.mode="http"`)
        - انسخ **Signing Secret** الخاص بـ Slack
        - اضبط عنوان URL للطلبات لكل من Event Subscriptions وInteractivity وأمر الشرطة المائلة إلى مسار webhook نفسه (الافتراضي `/slack/events`)

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

      <Step title="استخدم مسارات webhook فريدة لتعدد الحسابات عبر HTTP">
        وضع HTTP لكل حساب مدعوم.

        امنح كل حساب `webhookPath` مميزًا حتى لا تتعارض التسجيلات.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## قائمة manifest والنطاقات

<AccordionGroup>
  <Accordion title="مثال على manifest تطبيق Slack" defaultOpen>

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
    إذا قمت بضبط `channels.slack.userToken`، فإن نطاقات القراءة المعتادة هي:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (إذا كنت تعتمد على قراءات البحث في Slack)

  </Accordion>
</AccordionGroup>

## نموذج الرمز المميز

- `botToken` + `appToken` مطلوبان لـ Socket Mode.
- يتطلب وضع HTTP `botToken` + `signingSecret`.
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية
  عادية أو كائنات SecretRef.
- تتجاوز الرموز المميزة في الإعدادات القيم الاحتياطية من البيئة.
- تنطبق القيم الاحتياطية من متغيرات البيئة `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` على الحساب الافتراضي فقط.
- `userToken` (`xoxp-...`) خاص بالإعدادات فقط (لا يوجد بديل من البيئة) ويستخدم سلوك القراءة فقط افتراضيًا (`userTokenReadOnly: true`).
- اختياري: أضف `chat:write.customize` إذا كنت تريد أن تستخدم الرسائل الصادرة هوية الوكيل النشط (اسم مستخدم مخصص وأيقونة). يستخدم `icon_emoji` الصيغة `:emoji_name:`.

سلوك لقطة الحالة:

- يتتبع فحص حساب Slack حقول `*Source` و`*Status`
  لكل بيانات اعتماد (`botToken` و`appToken` و`signingSecret` و`userToken`).
- تكون الحالة `available` أو `configured_unavailable` أو `missing`.
- تعني `configured_unavailable` أن الحساب مضبوط عبر SecretRef
  أو مصدر أسرار غير مضمن آخر، لكن مسار الأمر/وقت التشغيل الحالي
  لم يتمكن من حل القيمة الفعلية.
- في وضع HTTP، يتم تضمين `signingSecretStatus`؛ أما في Socket Mode،
  فالزوج المطلوب هو `botTokenStatus` + `appTokenStatus`.

<Tip>
بالنسبة للإجراءات/قراءات الدليل، يمكن تفضيل user token عند ضبطه. أما في عمليات الكتابة، فيبقى bot token هو المفضل؛ ولا يُسمح بعمليات الكتابة عبر user token إلا عندما تكون `userTokenReadOnly: false` ويكون bot token غير متاح.
</Tip>

## الإجراءات والبوابات

تُتحكم إجراءات Slack بواسطة `channels.slack.actions.*`.

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
  <Tab title="سياسة الرسائل الخاصة">
    يتحكم `channels.slack.dmPolicy` في الوصول إلى الرسائل الخاصة (القديم: `channels.slack.dm.policy`):

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن يتضمن `channels.slack.allowFrom` القيمة `"*"`؛ القديم: `channels.slack.dm.allowFrom`)
    - `disabled`

    علامات الرسائل الخاصة:

    - `dm.enabled` (الافتراضي true)
    - `channels.slack.allowFrom` (المفضل)
    - `dm.allowFrom` (قديم)
    - `dm.groupEnabled` (الافتراضي للرسائل الجماعية الخاصة false)
    - `dm.groupChannels` (قائمة سماح MPIM اختيارية)

    أولوية تعدد الحسابات:

    - ينطبق `channels.slack.accounts.default.allowFrom` على الحساب `default` فقط.
    - ترث الحسابات المسماة `channels.slack.allowFrom` عندما لا تكون `allowFrom` الخاصة بها مضبوطة.
    - لا ترث الحسابات المسماة `channels.slack.accounts.default.allowFrom`.

    يستخدم الاقتران في الرسائل الخاصة `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="سياسة القنوات">
    يتحكم `channels.slack.groupPolicy` في التعامل مع القنوات:

    - `open`
    - `allowlist`
    - `disabled`

    توجد قائمة سماح القنوات ضمن `channels.slack.channels` ويجب أن تستخدم معرّفات القنوات الثابتة.

    ملاحظة وقت التشغيل: إذا كان `channels.slack` مفقودًا بالكامل (إعداد عبر البيئة فقط)، يعود وقت التشغيل إلى `groupPolicy="allowlist"` ويسجل تحذيرًا (حتى لو كانت `channels.defaults.groupPolicy` مضبوطة).

    تحليل الاسم/المعرّف:

    - يتم حل عناصر قائمة سماح القنوات وعناصر قائمة سماح الرسائل الخاصة عند بدء التشغيل عندما يسمح الوصول إلى الرمز المميز بذلك
    - يتم الاحتفاظ بعناصر أسماء القنوات غير المحلولة كما هي في الإعدادات، لكن يتم تجاهلها افتراضيًا في التوجيه
    - يعتمد التفويض الوارد وتوجيه القنوات على المعرّف أولًا بشكل افتراضي؛ ويتطلب المطابقة المباشرة لاسم المستخدم/slug تعيين `channels.slack.dangerouslyAllowNameMatching: true`

  </Tab>

  <Tab title="الإشارات ومستخدمي القنوات">
    تكون رسائل القنوات مقيدة بالإشارة بشكل افتراضي.

    مصادر الإشارة:

    - إشارة صريحة للتطبيق (`<@botId>`)
    - أنماط regex للإشارة (`agents.list[].groupChat.mentionPatterns`، والبديل `messages.groupChat.mentionPatterns`)
    - سلوك ضمني للرد ضمن سلسلة رسائل البوت

    عناصر التحكم لكل قناة (`channels.slack.channels.<id>`؛ الأسماء فقط عبر الحل عند بدء التشغيل أو `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `users` (قائمة سماح)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - صيغة مفتاح `toolsBySender`: ‏`id:` أو `e164:` أو `username:` أو `name:` أو wildcard `"*"`
      (لا تزال المفاتيح القديمة غير المسبوقة تُطابق `id:` فقط)

  </Tab>
</Tabs>

## سلاسل الرسائل والجلسات وعلامات الرد

- تُوجَّه الرسائل الخاصة كـ `direct`؛ والقنوات كـ `channel`؛ وMPIMs كـ `group`.
- مع الإعداد الافتراضي `session.dmScope=main`، تُدمج الرسائل الخاصة في Slack في الجلسة الرئيسية للوكيل.
- جلسات القنوات: `agent:<agentId>:slack:channel:<channelId>`.
- يمكن أن تنشئ الردود داخل السلسلة لاحقات لجلسة السلسلة (`:thread:<threadTs>`) عند الاقتضاء.
- القيمة الافتراضية لـ `channels.slack.thread.historyScope` هي `thread`؛ والقيمة الافتراضية لـ `thread.inheritParent` هي `false`.
- يتحكم `channels.slack.thread.initialHistoryLimit` في عدد رسائل السلسلة الموجودة التي يتم جلبها عند بدء جلسة سلسلة جديدة (الافتراضي `20`؛ اضبطه على `0` للتعطيل).

عناصر التحكم في سلاسل الردود:

- `channels.slack.replyToMode`: ‏`off|first|all|batched` (الافتراضي `off`)
- `channels.slack.replyToModeByChatType`: لكل من `direct|group|channel`
- البديل القديم للمحادثات المباشرة: `channels.slack.dm.replyToMode`

علامات الرد اليدوية مدعومة:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

ملاحظة: يعطّل `replyToMode="off"` **جميع** سلاسل الردود في Slack، بما في ذلك علامات `[[reply_to_*]]` الصريحة. يختلف هذا عن Telegram، حيث لا تزال العلامات الصريحة تُحترم في وضع `"off"`. يعكس هذا الاختلاف نماذج السلاسل في المنصات: إذ تُخفي سلاسل Slack الرسائل عن القناة، بينما تبقى ردود Telegram مرئية في تدفق الدردشة الرئيسي.

## تفاعلات التأكيد

يرسل `ackReaction` رمزًا تعبيريًا للتأكيد بينما يعالج OpenClaw رسالة واردة.

ترتيب الحل:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- بديل رمز هوية الوكيل التعبيري (`agents.list[].identity.emoji`، وإلا `"👀"`)

ملاحظات:

- يتوقع Slack shortcodes (مثل `"eyes"`).
- استخدم `""` لتعطيل التفاعل لهذا الحساب في Slack أو بشكل عام.

## بث النص

يتحكم `channels.slack.streaming` في سلوك المعاينة المباشرة:

- `off`: تعطيل بث المعاينة المباشرة.
- `partial` (الافتراضي): استبدال نص المعاينة بأحدث مخرجات جزئية.
- `block`: إلحاق تحديثات معاينة مجزأة.
- `progress`: عرض نص حالة التقدم أثناء التوليد، ثم إرسال النص النهائي.

يتحكم `channels.slack.nativeStreaming` في البث النصي الأصلي في Slack عندما تكون قيمة `streaming` هي `partial` (الافتراضي: `true`).

- يجب أن تكون سلسلة رد متاحة حتى يظهر البث النصي الأصلي. ولا يزال اختيار السلسلة يتبع `replyToMode`. وبدونها، تُستخدم معاينة المسودة العادية.
- تعود الوسائط والحمولات غير النصية إلى التسليم العادي.
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

## بديل تفاعل الكتابة

يضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة بينما يعالج OpenClaw ردًا، ثم يزيله عند انتهاء التشغيل. يكون هذا مفيدًا بشكل خاص خارج ردود السلاسل، التي تستخدم مؤشر حالة افتراضيًا "is typing...".

ترتيب الحل:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

ملاحظات:

- يتوقع Slack shortcodes (مثل `"hourglass_flowing_sand"`).
- يُعد التفاعل أفضل جهد ممكن، وتتم محاولة تنظيفه تلقائيًا بعد الرد أو بعد اكتمال مسار الفشل.

## الوسائط والتجزئة والتسليم

<AccordionGroup>
  <Accordion title="المرفقات الواردة">
    يتم تنزيل مرفقات ملفات Slack من عناوين URL خاصة مستضافة على Slack (تدفق طلبات موثّق بالرمز المميز) وكتابتها إلى مخزن الوسائط عندما ينجح الجلب وتسمح حدود الحجم بذلك.

    الحد الأقصى الافتراضي للحجم الوارد في وقت التشغيل هو `20MB` ما لم يتم تجاوزه عبر `channels.slack.mediaMaxMb`.

  </Accordion>

  <Accordion title="النصوص والملفات الصادرة">
    - تستخدم أجزاء النص `channels.slack.textChunkLimit` (الافتراضي 4000)
    - يفعّل `channels.slack.chunkMode="newline"` التقسيم المعتمد على الفقرات أولًا
    - تستخدم عمليات إرسال الملفات واجهات رفع Slack ويمكن أن تتضمن ردود السلاسل (`thread_ts`)
    - يتبع الحد الأقصى للوسائط الصادرة القيمة `channels.slack.mediaMaxMb` عند ضبطها؛ وإلا فإن إرسال القنوات يستخدم القيم الافتراضية حسب نوع MIME من مسار الوسائط
  </Accordion>

  <Accordion title="أهداف التسليم">
    الأهداف الصريحة المفضلة:

    - `user:<id>` للرسائل الخاصة
    - `channel:<id>` للقنوات

    تُفتح الرسائل الخاصة في Slack عبر واجهات محادثات Slack عند الإرسال إلى أهداف المستخدمين.

  </Accordion>
</AccordionGroup>

## الأوامر وسلوك الشرطة المائلة

- وضع الأوامر الأصلية التلقائي **معطّل** في Slack (`commands.native: "auto"` لا يفعّل الأوامر الأصلية في Slack).
- فعّل معالجات أوامر Slack الأصلية باستخدام `channels.slack.commands.native: true` (أو الإعداد العام `commands.native: true`).
- عند تفعيل الأوامر الأصلية، سجّل أوامر الشرطة المائلة المطابقة في Slack (أسماء `/<command>`)، مع استثناء واحد:
  - سجّل `/agentstatus` لأمر الحالة (يحجز Slack الأمر `/status`)
- إذا لم تكن الأوامر الأصلية مفعّلة، يمكنك تشغيل أمر شرطة مائلة واحد مضبوط عبر `channels.slack.slashCommand`.
- تتكيف قوائم الوسائط الأصلية للوسائط الآن مع استراتيجية العرض:
  - حتى 5 خيارات: كتل أزرار
  - من 6 إلى 100 خيار: قائمة اختيار ثابتة
  - أكثر من 100 خيار: اختيار خارجي مع تصفية خيارات غير متزامنة عندما تكون معالجات خيارات interactivity متاحة
  - إذا تجاوزت قيم الخيارات المرمزة حدود Slack، يعود التدفق إلى الأزرار
- بالنسبة لحمولات الخيارات الطويلة، تستخدم قوائم وسائط وسيطات أوامر الشرطة المائلة مربع تأكيد قبل إرسال القيمة المحددة.

إعدادات أمر الشرطة المائلة الافتراضية:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

تستخدم جلسات الشرطة المائلة مفاتيح معزولة:

- `agent:<agentId>:slack:slash:<userId>`

ومع ذلك، لا يزال توجيه تنفيذ الأوامر يتم إلى جلسة المحادثة المستهدفة (`CommandTargetSessionKey`).

## الردود التفاعلية

يمكن لـ Slack عرض عناصر تحكم تفاعلية في الردود التي ينشئها الوكيل، لكن هذه الميزة معطّلة افتراضيًا.

فعّلها على مستوى النظام:

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

عند التفعيل، يمكن للوكلاء إصدار توجيهات رد خاصة بـ Slack فقط:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

تُترجم هذه التوجيهات إلى Slack Block Kit وتُوجّه النقرات أو التحديدات مرة أخرى عبر مسار أحداث تفاعل Slack الحالي.

ملاحظات:

- هذه واجهة مستخدم خاصة بـ Slack. لا تترجم القنوات الأخرى توجيهات Slack Block Kit إلى أنظمة الأزرار الخاصة بها.
- قيم نداءات التفاعل هي رموز مبهمة يولدها OpenClaw، وليست قيمًا خامًا أنشأها الوكيل.
- إذا تجاوزت الكتل التفاعلية المولدة حدود Slack Block Kit، يعود OpenClaw إلى الرد النصي الأصلي بدلًا من إرسال حمولة كتل غير صالحة.

## موافقات exec في Slack

يمكن أن يعمل Slack كعميل موافقة أصلي باستخدام الأزرار التفاعلية والتفاعلات، بدلًا من الرجوع إلى واجهة الويب أو الطرفية.

- تستخدم موافقات Exec المسار `channels.slack.execApprovals.*` للتوجيه الأصلي إلى الرسائل الخاصة/القنوات.
- لا تزال موافقات الإضافات قادرة على الحل عبر سطح الأزرار الأصلي نفسه في Slack عندما يصل الطلب أصلًا إلى Slack ويكون نوع معرّف الموافقة هو `plugin:`.
- لا يزال تفويض الموافقين مفروضًا: يمكن فقط للمستخدمين المحددين كموافقين الموافقة على الطلبات أو رفضها عبر Slack.

يستخدم هذا سطح أزرار الموافقة المشتركة نفسه مثل القنوات الأخرى. عند تفعيل `interactivity` في إعدادات تطبيق Slack لديك، تُعرض مطالبات الموافقة كأزرار Block Kit مباشرة داخل المحادثة.
عند وجود هذه الأزرار، فإنها تكون تجربة الموافقة الأساسية؛ ويجب على OpenClaw
أن يتضمن أمر `/approve` اليدوي فقط عندما تشير نتيجة الأداة إلى أن
الموافقات عبر الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.

مسار الإعداد:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (اختياري؛ يعود إلى `commands.ownerAllowFrom` عندما يكون ذلك ممكنًا)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `agentFilter`, `sessionFilter`

يفعّل Slack موافقات exec الأصلية تلقائيًا عندما تكون قيمة `enabled` غير مضبوطة أو `"auto"` وعند حل موافق واحد على الأقل. اضبط `enabled: false` لتعطيل Slack كعميل موافقة أصلي بشكل صريح.
واضبط `enabled: true` لفرض تفعيل الموافقات الأصلية عند حل الموافقين.

السلوك الافتراضي من دون إعداد صريح لموافقة exec في Slack:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

لا تكون إعدادات Slack الأصلية الصريحة مطلوبة إلا عندما تريد تجاوز الموافقين أو إضافة عوامل تصفية أو
اختيار التسليم إلى دردشة المصدر:

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

يكون التوجيه المشترك لـ `approvals.exec` منفصلًا. استخدمه فقط عندما يجب أيضًا
توجيه مطالبات موافقة exec إلى دردشات أخرى أو إلى أهداف صريحة خارج النطاق. كما أن التوجيه المشترك لـ `approvals.plugin` منفصل أيضًا؛
ولا تزال أزرار Slack الأصلية قادرة على حل موافقات الإضافات عندما تصل هذه الطلبات أصلًا
إلى Slack.

يعمل `/approve` في الدردشة نفسها أيضًا في قنوات Slack والرسائل الخاصة التي تدعم الأوامر بالفعل. راجع [موافقات Exec](/ar/tools/exec-approvals) للاطلاع على نموذج توجيه الموافقات الكامل.

## الأحداث والسلوك التشغيلي

- تُحوَّل عمليات تعديل/حذف الرسائل وبث سلاسل الرسائل إلى أحداث نظام.
- تُحوَّل أحداث إضافة/إزالة التفاعلات إلى أحداث نظام.
- تُحوَّل أحداث انضمام/مغادرة الأعضاء وإنشاء/إعادة تسمية القنوات وإضافة/إزالة التثبيتات إلى أحداث نظام.
- يمكن لـ `channel_id_changed` ترحيل مفاتيح إعدادات القنوات عندما يكون `configWrites` مفعّلًا.
- تُعامل بيانات موضوع/غرض القناة الوصفية كسياق غير موثوق ويمكن حقنها في سياق التوجيه.
- تتم تصفية بادئ السلسلة والتهيئة الأولية لسجل السلسلة حسب قوائم سماح المرسلين المضبوطة عند الاقتضاء.
- تُصدر إجراءات الكتل وتفاعلات النوافذ المنبثقة أحداث نظام منظمة بالشكل `Slack interaction: ...` مع حقول حمولة غنية:
  - إجراءات الكتل: القيم المحددة والتسميات وقيم المحدد وبيانات `workflow_*` الوصفية
  - أحداث `view_submission` و`view_closed` للنوافذ المنبثقة مع بيانات القناة الموجهة ومدخلات النموذج

## مؤشرات مرجعية للإعدادات

المرجع الأساسي:

- [مرجع الإعدادات - Slack](/ar/gateway/configuration-reference#slack)

  حقول Slack عالية الإشارة:
  - الوضع/المصادقة: `mode`, `botToken`, `appToken`, `signingSecret`, `webhookPath`, `accounts.*`
  - الوصول إلى الرسائل الخاصة: `dm.enabled`, `dmPolicy`, `allowFrom` (قديم: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
  - مفتاح التوافق: `dangerouslyAllowNameMatching` (للطوارئ فقط؛ اتركه معطّلًا ما لم تكن بحاجة إليه)
  - الوصول إلى القنوات: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`
  - السلاسل/السجل: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
  - التسليم: `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `streaming`, `nativeStreaming`
  - التشغيل/الميزات: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لا توجد ردود في القنوات">
    تحقّق، بالترتيب، من:

    - `groupPolicy`
    - قائمة سماح القنوات (`channels.slack.channels`)
    - `requireMention`
    - قائمة السماح `users` لكل قناة

    أوامر مفيدة:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="يتم تجاهل رسائل DM">
    تحقّق من:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (أو القديم `channels.slack.dm.policy`)
    - موافقات الاقتران / عناصر قائمة السماح

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="وضع Socket لا يتصل">
    تحقّق من صحة رمزي bot وapp ومن تفعيل Socket Mode في إعدادات تطبيق Slack.

    إذا أظهر `openclaw channels status --probe --json` القيمة `botTokenStatus` أو
    `appTokenStatus: "configured_unavailable"`، فهذا يعني أن حساب Slack
    مضبوط، لكن وقت التشغيل الحالي لم يتمكن من حل القيمة
    المدعومة بواسطة SecretRef.

  </Accordion>

  <Accordion title="وضع HTTP لا يستقبل الأحداث">
    تحقّق من:

    - signing secret
    - مسار webhook
    - عناوين URL لطلبات Slack ‏(Events + Interactivity + Slash Commands)
    - `webhookPath` فريد لكل حساب HTTP

    إذا ظهر `signingSecretStatus: "configured_unavailable"` في
    لقطات الحساب، فهذا يعني أن حساب HTTP مضبوط لكن وقت التشغيل الحالي لم يتمكن
    من حل signing secret المدعوم بواسطة SecretRef.

  </Accordion>

  <Accordion title="الأوامر الأصلية/أوامر الشرطة المائلة لا تعمل">
    تحقّق مما إذا كنت تقصد:

    - وضع الأوامر الأصلية (`channels.slack.commands.native: true`) مع تسجيل أوامر الشرطة المائلة المطابقة في Slack
    - أو وضع أمر الشرطة المائلة الفردي (`channels.slack.slashCommand.enabled: true`)

    تحقّق أيضًا من `commands.useAccessGroups` وقوائم سماح القنوات/المستخدمين.

  </Accordion>
</AccordionGroup>

## ذو صلة

- [الاقتران](/ar/channels/pairing)
- [المجموعات](/ar/channels/groups)
- [الأمان](/ar/gateway/security)
- [توجيه القنوات](/ar/channels/channel-routing)
- [استكشاف الأخطاء وإصلاحها](/ar/channels/troubleshooting)
- [الإعدادات](/ar/gateway/configuration)
- [أوامر الشرطة المائلة](/ar/tools/slash-commands)
