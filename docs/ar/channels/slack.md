---
read_when:
    - عند إعداد Slack أو تصحيح وضع socket/HTTP في Slack
summary: إعداد Slack وسلوك وقت التشغيل (Socket Mode + HTTP Events API)
title: Slack
x-i18n:
    generated_at: "2026-04-05T12:37:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: efb37e1f04e1ac8ac3786c36ffc20013dacdc654bfa61e7f6e8df89c4902d2ab
    source_path: channels/slack.md
    workflow: 15
---

# Slack

الحالة: جاهز للإنتاج للرسائل المباشرة + القنوات عبر تكاملات تطبيق Slack. الوضع الافتراضي هو Socket Mode؛ كما أن وضع HTTP Events API مدعوم أيضًا.

<CardGroup cols={3}>
  <Card title="الاقتران" icon="link" href="/channels/pairing">
    تستخدم الرسائل المباشرة في Slack وضع الاقتران افتراضيًا.
  </Card>
  <Card title="أوامر slash" icon="terminal" href="/tools/slash-commands">
    سلوك الأوامر الأصلي وكتالوج الأوامر.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/channels/troubleshooting">
    تشخيصات عبر القنوات وأدلة إصلاح.
  </Card>
</CardGroup>

## الإعداد السريع

<Tabs>
  <Tab title="Socket Mode (الافتراضي)">
    <Steps>
      <Step title="أنشئ تطبيق Slack والرموز">
        في إعدادات تطبيق Slack:

        - فعّل **Socket Mode**
        - أنشئ **App Token** (`xapp-...`) مع `connections:write`
        - ثبّت التطبيق وانسخ **Bot Token** (`xoxb-...`)
      </Step>

      <Step title="إعداد OpenClaw">

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

        الرجوع إلى البيئة (للحساب الافتراضي فقط):

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

        فعّل أيضًا App Home **Messages Tab** للرسائل المباشرة.
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
      <Step title="إعداد تطبيق Slack لـ HTTP">

        - اضبط الوضع على HTTP (`channels.slack.mode="http"`)
        - انسخ **Signing Secret** الخاص بـ Slack
        - اضبط عنوان Request URL لاشتراكات الأحداث + التفاعلية + أوامر Slash على مسار webhook نفسه (الافتراضي `/slack/events`)

      </Step>

      <Step title="إعداد وضع HTTP في OpenClaw">

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

      <Step title="استخدم مسارات webhook فريدة للحسابات المتعددة عبر HTTP">
        وضع HTTP لكل حساب مدعوم.

        امنح كل حساب `webhookPath` مختلفًا حتى لا تتصادم التسجيلات.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## قائمة التحقق من manifest والنطاقات

<AccordionGroup>
  <Accordion title="مثال على manifest لتطبيق Slack" defaultOpen>

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
    إذا قمت بإعداد `channels.slack.userToken`، فإن نطاقات القراءة النموذجية هي:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (إذا كنت تعتمد على قراءات البحث في Slack)

  </Accordion>
</AccordionGroup>

## نموذج الرموز

- `botToken` + `appToken` مطلوبان لـ Socket Mode.
- يتطلب وضع HTTP كلاً من `botToken` + `signingSecret`.
- يقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية صريحة
  أو كائنات SecretRef.
- تتجاوز الرموز في الإعداد قيم الرجوع إلى البيئة.
- ينطبق الرجوع إلى متغيرات البيئة `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` على الحساب الافتراضي فقط.
- `userToken` (`xoxp-...`) للإعداد فقط (لا يوجد رجوع إلى البيئة) ويستخدم سلوك القراءة فقط افتراضيًا (`userTokenReadOnly: true`).
- اختياري: أضف `chat:write.customize` إذا كنت تريد أن تستخدم الرسائل الصادرة هوية الوكيل النشطة (اسم مستخدم `username` مخصص وأيقونة). يستخدم `icon_emoji` صيغة `:emoji_name:`.

سلوك لقطة الحالة:

- يتتبع فحص حساب Slack حقول `*Source` و`*Status`
  لكل بيانات اعتماد (`botToken` و`appToken` و`signingSecret` و`userToken`).
- تكون الحالة إحدى القيم: `available` أو `configured_unavailable` أو `missing`.
- تعني `configured_unavailable` أن الحساب مُعدّ عبر SecretRef
  أو مصدر أسرار آخر غير مضمن، ولكن مسار الأمر/وقت التشغيل الحالي
  لم يتمكن من حل القيمة الفعلية.
- في وضع HTTP، يتم تضمين `signingSecretStatus`؛ وفي Socket Mode، يكون
  الزوج المطلوب هو `botTokenStatus` + `appTokenStatus`.

<Tip>
بالنسبة إلى الإجراءات/قراءات الدليل، يمكن تفضيل user token عند إعداده. أما بالنسبة إلى عمليات الكتابة، فيظل bot token هو المفضل؛ ولا يُسمح بعمليات الكتابة عبر user-token إلا عندما يكون `userTokenReadOnly: false` وbot token غير متاح.
</Tip>

## الإجراءات والبوابات

تتحكم `channels.slack.actions.*` في إجراءات Slack.

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
    تتحكم `channels.slack.dmPolicy` في الوصول إلى الرسائل المباشرة (القديم: `channels.slack.dm.policy`):

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن يتضمن `channels.slack.allowFrom` القيمة `"*"`؛ القديم: `channels.slack.dm.allowFrom`)
    - `disabled`

    علامات الرسائل المباشرة:

    - `dm.enabled` (الافتراضي true)
    - `channels.slack.allowFrom` (المفضل)
    - `dm.allowFrom` (قديم)
    - `dm.groupEnabled` (الرسائل المباشرة الجماعية default false)
    - `dm.groupChannels` (قائمة سماح MPIM اختيارية)

    أسبقية الحسابات المتعددة:

    - ينطبق `channels.slack.accounts.default.allowFrom` على الحساب `default` فقط.
    - ترث الحسابات المسماة `channels.slack.allowFrom` عندما لا تكون `allowFrom` الخاصة بها مضبوطة.
    - لا ترث الحسابات المسماة `channels.slack.accounts.default.allowFrom`.

    يستخدم الاقتران في الرسائل المباشرة `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="سياسة القنوات">
    تتحكم `channels.slack.groupPolicy` في معالجة القنوات:

    - `open`
    - `allowlist`
    - `disabled`

    توجد قائمة السماح للقنوات ضمن `channels.slack.channels` ويجب أن تستخدم معرّفات قنوات ثابتة.

    ملاحظة وقت التشغيل: إذا كان `channels.slack` مفقودًا بالكامل (إعداد يعتمد على البيئة فقط)، فسيعود وقت التشغيل إلى `groupPolicy="allowlist"` ويسجل تحذيرًا (حتى إذا كان `channels.defaults.groupPolicy` مضبوطًا).

    تحليل الاسم/المعرّف:

    - يتم تحليل مدخلات قائمة السماح للقنوات ومدخلات قائمة السماح للرسائل المباشرة عند بدء التشغيل عندما يتيح وصول الرمز ذلك
    - تُحتفظ بمدخلات أسماء القنوات غير المحلولة كما هي في الإعداد، لكنها تُتجاهل افتراضيًا في التوجيه
    - يعتمد التفويض الوارد وتوجيه القنوات على المعرّف أولاً افتراضيًا؛ وتتطلب مطابقة اسم المستخدم/الاسم المختصر مباشرة ضبط `channels.slack.dangerouslyAllowNameMatching: true`

  </Tab>

  <Tab title="الإشارات ومستخدمي القنوات">
    رسائل القنوات مقيّدة بالإشارة افتراضيًا.

    مصادر الإشارة:

    - إشارة صريحة إلى التطبيق (`<@botId>`)
    - أنماط regex للإشارة (`agents.list[].groupChat.mentionPatterns`، والرجوع إلى `messages.groupChat.mentionPatterns`)
    - سلوك الرد الضمني على سلسلة خاصة بالبوت

    عناصر التحكم لكل قناة (`channels.slack.channels.<id>`؛ الأسماء فقط عبر التحليل عند بدء التشغيل أو `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `users` (قائمة سماح)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - صيغة مفتاح `toolsBySender`: ‏`id:` أو `e164:` أو `username:` أو `name:` أو wildcard `"*"`
      (لا تزال المفاتيح القديمة غير المسبوقة تُربط إلى `id:` فقط)

  </Tab>
</Tabs>

## سلاسل المحادثات والجلسات وعلامات الرد

- يتم توجيه الرسائل المباشرة كـ `direct`؛ والقنوات كـ `channel`؛ وMPIMs كـ `group`.
- مع الإعداد الافتراضي `session.dmScope=main`، تُدمج الرسائل المباشرة في Slack ضمن الجلسة الرئيسية للوكيل.
- جلسات القنوات: `agent:<agentId>:slack:channel:<channelId>`.
- يمكن أن تنشئ ردود السلاسل لاحقات جلسة للسلسلة (`:thread:<threadTs>`) عند الاقتضاء.
- القيمة الافتراضية لـ `channels.slack.thread.historyScope` هي `thread`؛ والقيمة الافتراضية لـ `thread.inheritParent` هي `false`.
- يتحكم `channels.slack.thread.initialHistoryLimit` في عدد رسائل السلسلة الموجودة التي تُجلب عند بدء جلسة سلسلة جديدة (الافتراضي `20`؛ اضبطه إلى `0` للتعطيل).

عناصر التحكم في سلاسل الرد:

- `channels.slack.replyToMode`: ‏`off|first|all` (الافتراضي `off`)
- `channels.slack.replyToModeByChatType`: لكل من `direct|group|channel`
- الرجوع القديم للدردشات المباشرة: `channels.slack.dm.replyToMode`

علامات الرد اليدوية مدعومة:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

ملاحظة: يعطّل `replyToMode="off"` **كل** سلاسل الرد في Slack، بما في ذلك علامات `[[reply_to_*]]` الصريحة. وهذا يختلف عن Telegram، حيث تظل العلامات الصريحة محترمة في وضع `"off"`. ويعكس هذا الاختلاف نماذج السلاسل في المنصات: تخفي سلاسل Slack الرسائل عن القناة، بينما تظل ردود Telegram مرئية في تدفق الدردشة الرئيسي.

## تفاعلات التأكيد

يرسل `ackReaction` رمزًا تعبيريًا للتأكيد بينما يعالج OpenClaw رسالة واردة.

ترتيب التحليل:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- الرجوع إلى emoji هوية الوكيل (`agents.list[].identity.emoji`، وإلا "👀")

ملاحظات:

- يتوقع Slack shortcodes (على سبيل المثال `"eyes"`).
- استخدم `""` لتعطيل التفاعل لحساب Slack أو على مستوى عام.

## بث النص

يتحكم `channels.slack.streaming` في سلوك المعاينة المباشرة:

- `off`: تعطيل بث المعاينة المباشرة.
- `partial` (الافتراضي): استبدال نص المعاينة بآخر مخرجات جزئية.
- `block`: إلحاق تحديثات معاينة مجزأة.
- `progress`: إظهار نص حالة التقدم أثناء التوليد، ثم إرسال النص النهائي.

يتحكم `channels.slack.nativeStreaming` في البث النصي الأصلي في Slack عندما تكون `streaming` هي `partial` (الافتراضي: `true`).

- يجب أن تكون سلسلة رد متاحة حتى يظهر البث النصي الأصلي. ولا يزال اختيار السلسلة يتبع `replyToMode`. ومن دونها، تُستخدم معاينة المسودة العادية.
- تعود الوسائط والحمولات غير النصية إلى التسليم العادي.
- إذا فشل البث في منتصف الرد، يعود OpenClaw إلى التسليم العادي للحمولات المتبقية.

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

- تتم الترحيل التلقائي لـ `channels.slack.streamMode` (`replace | status_final | append`) إلى `channels.slack.streaming`.
- تتم الترحيل التلقائي لـ boolean `channels.slack.streaming` إلى `channels.slack.nativeStreaming`.

## الرجوع إلى تفاعل الكتابة

يضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة بينما يعالج OpenClaw ردًا، ثم يزيله عند انتهاء التشغيل. وهذا مفيد أكثر خارج ردود السلاسل، التي تستخدم مؤشر حالة افتراضي "is typing...".

ترتيب التحليل:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

ملاحظات:

- يتوقع Slack shortcodes (على سبيل المثال `"hourglass_flowing_sand"`).
- التفاعل best-effort وتتم محاولة تنظيفه تلقائيًا بعد اكتمال الرد أو مسار الفشل.

## الوسائط والتجزئة والتسليم

<AccordionGroup>
  <Accordion title="المرفقات الواردة">
    تُنزَّل مرفقات ملفات Slack من عناوين URL خاصة مستضافة لدى Slack (تدفق طلبات موثَّق بالرمز) وتُكتب في مخزن الوسائط عند نجاح الجلب والسماح بحدود الحجم.

    الحد الأقصى للحجم الوارد في وقت التشغيل يكون افتراضيًا `20MB` ما لم يتم تجاوزه عبر `channels.slack.mediaMaxMb`.

  </Accordion>

  <Accordion title="النصوص والملفات الصادرة">
    - تستخدم مقاطع النص `channels.slack.textChunkLimit` (الافتراضي 4000)
    - يفعّل `channels.slack.chunkMode="newline"` التقسيم بحسب الفقرات أولًا
    - تستخدم عمليات إرسال الملفات واجهات Slack upload API ويمكن أن تتضمن ردود سلاسل (`thread_ts`)
    - يتبع الحد الأقصى للوسائط الصادرة `channels.slack.mediaMaxMb` عند إعداده؛ وإلا فتستخدم عمليات الإرسال في القنوات القيم الافتراضية حسب نوع MIME من media pipeline
  </Accordion>

  <Accordion title="أهداف التسليم">
    الأهداف الصريحة المفضلة:

    - `user:<id>` للرسائل المباشرة
    - `channel:<id>` للقنوات

    تُفتح الرسائل المباشرة في Slack عبر Slack conversation APIs عند الإرسال إلى أهداف المستخدمين.

  </Accordion>
</AccordionGroup>

## الأوامر وسلوك slash

- الوضع التلقائي للأوامر الأصلية **معطّل** في Slack (`commands.native: "auto"` لا يفعّل أوامر Slack الأصلية).
- فعّل معالجات أوامر Slack الأصلية باستخدام `channels.slack.commands.native: true` (أو الإعداد العام `commands.native: true`).
- عند تفعيل الأوامر الأصلية، سجّل أوامر slash المطابقة في Slack (أسماء `/<command>`)، مع استثناء واحد:
  - سجّل `/agentstatus` لأمر الحالة (يحجز Slack الاسم `/status`)
- إذا لم تكن الأوامر الأصلية مفعّلة، فيمكنك تشغيل أمر slash واحد مُعدّ عبر `channels.slack.slashCommand`.
- تتكيف قوائم الوسائط الأصلية للوسائط مع إستراتيجية العرض:
  - حتى 5 خيارات: كتل أزرار
  - من 6 إلى 100 خيار: قائمة اختيار ثابتة
  - أكثر من 100 خيار: اختيار خارجي مع تصفية خيارات غير متزامنة عند توفر معالجات خيارات التفاعلية
  - إذا تجاوزت قيم الخيارات المشفّرة حدود Slack، يعود التدفق إلى الأزرار
- بالنسبة إلى حمولات الخيارات الطويلة، تستخدم قوائم وسائط معاملات أوامر Slash مربع حوار تأكيد قبل إرسال القيمة المحددة.

إعدادات أوامر slash الافتراضية:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

تستخدم جلسات Slash مفاتيح معزولة:

- `agent:<agentId>:slack:slash:<userId>`

ومع ذلك تظل توجّه تنفيذ الأمر إلى جلسة المحادثة المستهدفة (`CommandTargetSessionKey`).

## الردود التفاعلية

يمكن لـ Slack عرض عناصر تحكم تفاعلية للردود التي أنشأها الوكيل، لكن هذه الميزة معطلة افتراضيًا.

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

عند التفعيل، يمكن للوكلاء إصدار توجيهات رد خاصة بـ Slack فقط:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

تُترجم هذه التوجيهات إلى Slack Block Kit وتعيد توجيه النقرات أو التحديدات عبر مسار أحداث تفاعل Slack الحالي.

ملاحظات:

- هذه واجهة مستخدم خاصة بـ Slack. لا تترجم القنوات الأخرى توجيهات Slack Block Kit إلى أنظمة الأزرار الخاصة بها.
- قيم ردود النداء التفاعلية هي رموز opaque مولدة من OpenClaw، وليست قيمًا خام كتبها الوكيل.
- إذا كانت الكتل التفاعلية المولدة ستتجاوز حدود Slack Block Kit، فسيعود OpenClaw إلى الرد النصي الأصلي بدلًا من إرسال حمولة blocks غير صالحة.

## موافقات exec في Slack

يمكن أن يعمل Slack كعميل موافقة أصلي مع أزرار وتفاعلات تفاعلية، بدلًا من الرجوع إلى Web UI أو الطرفية.

- تستخدم موافقات exec المسار `channels.slack.execApprovals.*` للتوجيه الأصلي في الرسائل المباشرة/القنوات.
- لا تزال موافقات plugin تُحل عبر سطح أزرار Slack الأصلي نفسه عندما يصل الطلب بالفعل إلى Slack ويكون نوع معرّف الموافقة هو `plugin:`.
- لا يزال تفويض الموافقين مفروضًا: لا يمكن سوى للمستخدمين المحددين كموافقين الموافقة على الطلبات أو رفضها عبر Slack.

يستخدم هذا سطح أزرار الموافقة المشتركة نفسه كما في القنوات الأخرى. عندما تكون `interactivity` مفعّلة في إعدادات تطبيق Slack لديك، تُعرض مطالبات الموافقة كأزرار Block Kit مباشرة في المحادثة.
عند وجود هذه الأزرار، تكون هي تجربة الاستخدام الأساسية للموافقة؛ ويجب على OpenClaw
أن يدرج أمر `/approve` يدويًا فقط عندما تشير نتيجة الأداة إلى أن
الموافقات عبر الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.

مسار الإعداد:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (اختياري؛ يعود إلى `commands.ownerAllowFrom` عندما يكون ممكنًا)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
- `agentFilter`, `sessionFilter`

يفعّل Slack موافقات exec الأصلية تلقائيًا عندما تكون `enabled` غير مضبوطة أو `"auto"` ويوجد موافق واحد محلول على الأقل.
اضبط `enabled: false` لتعطيل Slack كعميل موافقة أصلي صراحةً.
واضبط `enabled: true` لفرض تشغيل الموافقات الأصلية عند حل الموافقين.

السلوك الافتراضي من دون إعداد صريح لموافقات exec في Slack:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

يلزم إعداد Slack أصلي صريح فقط عندما تريد تجاوز الموافقين أو إضافة عوامل تصفية أو
تفعيل التسليم إلى دردشة المنشأ:

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

إن إعادة توجيه `approvals.exec` المشتركة منفصلة. استخدمها فقط عندما يجب أن تُوجَّه مطالبات موافقة exec أيضًا
إلى دردشات أخرى أو أهداف صريحة خارج النطاق. كما أن إعادة توجيه `approvals.plugin` المشتركة منفصلة أيضًا؛
ولا تزال أزرار Slack الأصلية قادرة على حل موافقات plugin عندما تصل هذه الطلبات بالفعل
إلى Slack.

كما يعمل `/approve` في الدردشة نفسها داخل قنوات Slack والرسائل المباشرة التي تدعم الأوامر بالفعل. راجع [موافقات Exec](/tools/exec-approvals) للحصول على نموذج إعادة توجيه الموافقات الكامل.

## الأحداث والسلوك التشغيلي

- تُربط تعديلات الرسائل/حذفها/بث السلاسل بأحداث النظام.
- تُربط أحداث إضافة/إزالة التفاعلات بأحداث النظام.
- تُربط أحداث انضمام/مغادرة الأعضاء، وإنشاء/إعادة تسمية القنوات، وإضافة/إزالة التثبيتات بأحداث النظام.
- يمكن لـ `channel_id_changed` ترحيل مفاتيح إعداد القنوات عندما يكون `configWrites` مفعّلًا.
- تُعامل بيانات topic/purpose الخاصة بالقناة كسياق غير موثوق ويمكن حقنها في سياق التوجيه.
- تُصفّى بادئات السلاسل وسياق تهيئة سجل السلسلة الأولي بحسب قوائم السماح المكونة للمرسلين عند الاقتضاء.
- تصدر إجراءات الكتل وتفاعلات النوافذ المنظمة أحداث نظام منظمة باسم `Slack interaction: ...` مع حقول حمولة غنية:
  - إجراءات الكتل: القيم المحددة، والتسميات، وقيم المحددات، وبيانات `workflow_*`
  - أحداث `view_submission` و`view_closed` الخاصة بالنوافذ مع بيانات القنوات الموجهة ومدخلات النماذج

## مؤشرات مرجع الإعداد

المرجع الأساسي:

- [مرجع الإعداد - Slack](/gateway/configuration-reference#slack)

  الحقول الأبرز في Slack:
  - الوضع/المصادقة: `mode`, `botToken`, `appToken`, `signingSecret`, `webhookPath`, `accounts.*`
  - الوصول إلى الرسائل المباشرة: `dm.enabled`, `dmPolicy`, `allowFrom` (القديم: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
  - مفتاح التوافق: `dangerouslyAllowNameMatching` (للطوارئ؛ اتركه معطّلًا ما لم تكن بحاجة إليه)
  - الوصول إلى القنوات: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`
  - السلاسل/السجل: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
  - التسليم: `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `streaming`, `nativeStreaming`
  - العمليات/الميزات: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لا توجد ردود في القنوات">
    تحقّق، بالترتيب، من:

    - `groupPolicy`
    - قائمة السماح للقنوات (`channels.slack.channels`)
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
    - موافقات الاقتران / إدخالات قائمة السماح

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="وضع Socket لا يتصل">
    تحقّق من bot token + app token ومن تفعيل Socket Mode في إعدادات تطبيق Slack.

    إذا أظهر `openclaw channels status --probe --json` أن `botTokenStatus` أو
    `appTokenStatus: "configured_unavailable"`، فهذا يعني أن حساب Slack
    مُعدّ لكنه لم يتمكن وقت التشغيل الحالي من حل القيمة
    المدعومة بـ SecretRef.

  </Accordion>

  <Accordion title="وضع HTTP لا يستقبل الأحداث">
    تحقّق من:

    - signing secret
    - مسار webhook
    - عناوين Slack Request URL ‏(الأحداث + التفاعلية + أوامر Slash)
    - `webhookPath` فريد لكل حساب HTTP

    إذا ظهر `signingSecretStatus: "configured_unavailable"` في لقطات
    الحساب، فهذا يعني أن حساب HTTP مُعدّ لكن وقت التشغيل الحالي لم يتمكن
    من حل signing secret المدعوم بـ SecretRef.

  </Accordion>

  <Accordion title="الأوامر الأصلية/slash لا تعمل">
    تحقّق مما إذا كنت تقصد:

    - وضع الأوامر الأصلية (`channels.slack.commands.native: true`) مع تسجيل أوامر slash المطابقة في Slack
    - أو وضع أمر slash واحد (`channels.slack.slashCommand.enabled: true`)

    وتحقق أيضًا من `commands.useAccessGroups` وقوائم سماح القنوات/المستخدمين.

  </Accordion>
</AccordionGroup>

## ذو صلة

- [الاقتران](/channels/pairing)
- [المجموعات](/channels/groups)
- [الأمان](/gateway/security)
- [توجيه القنوات](/channels/channel-routing)
- [استكشاف الأخطاء وإصلاحها](/channels/troubleshooting)
- [الإعداد](/gateway/configuration)
- [أوامر slash](/tools/slash-commands)
