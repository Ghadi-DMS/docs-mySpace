---
read_when:
    - إضافة أو تعديل إجراءات الرسائل في CLI
    - تغيير سلوك القنوات الصادر
summary: مرجع CLI للأمر `openclaw message` ‏(send + إجراءات القنوات)
title: message
x-i18n:
    generated_at: "2026-04-05T12:38:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: b70f36189d028d59db25cd8b39d7c67883eaea71bea2358ee6314eec6cd2fa51
    source_path: cli/message.md
    workflow: 15
---

# `openclaw message`

أمر صادر واحد لإرسال الرسائل وإجراءات القنوات
(Discord/Google Chat/iMessage/Matrix/Mattermost (plugin)/Microsoft Teams/Signal/Slack/Telegram/WhatsApp).

## الاستخدام

```
openclaw message <subcommand> [flags]
```

اختيار القناة:

- يكون `--channel` مطلوبًا إذا كانت هناك أكثر من قناة واحدة مهيأة.
- إذا كانت هناك قناة واحدة فقط مهيأة، فإنها تصبح القيمة الافتراضية.
- القيم: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp` ‏(يتطلب Mattermost وجود plugin)

تنسيقات الهدف (`--target`):

- WhatsApp: ‏E.164 أو group JID
- Telegram: ‏chat id أو `@username`
- Discord: ‏`channel:<id>` أو `user:<id>` ‏(أو الإشارة `<@id>`؛ وتُعامل المعرّفات الرقمية الخام على أنها قنوات)
- Google Chat: ‏`spaces/<spaceId>` أو `users/<userId>`
- Slack: ‏`channel:<id>` أو `user:<id>` ‏(يُقبل معرّف القناة الخام)
- Mattermost (plugin): ‏`channel:<id>` أو `user:<id>` أو `@username` ‏(تُعامل المعرّفات المجردة على أنها قنوات)
- Signal: ‏`+E.164` أو `group:<id>` أو `signal:+E.164` أو `signal:group:<id>` أو `username:<name>`/`u:<name>`
- iMessage: ‏handle أو `chat_id:<id>` أو `chat_guid:<guid>` أو `chat_identifier:<id>`
- Matrix: ‏`@user:server` أو `!room:server` أو `#alias:server`
- Microsoft Teams: معرّف المحادثة (`19:...@thread.tacv2`) أو `conversation:<id>` أو `user:<aad-object-id>`

البحث بالاسم:

- بالنسبة إلى المزوّدين المدعومين (Discord/Slack/إلخ)، يتم حل أسماء القنوات مثل `Help` أو `#help` عبر ذاكرة التخزين المؤقت للدليل.
- عند غياب الإدخال من الذاكرة المؤقتة، سيحاول OpenClaw إجراء بحث مباشر في الدليل عندما يدعم المزوّد ذلك.

## الأعلام الشائعة

- `--channel <name>`
- `--account <id>`
- `--target <dest>` ‏(القناة أو المستخدم المستهدف للإرسال/poll/read/إلخ)
- `--targets <name>` ‏(قابل للتكرار؛ للبث فقط)
- `--json`
- `--dry-run`
- `--verbose`

## سلوك SecretRef

- يقوم `openclaw message` بحل SecretRefs المدعومة للقنوات قبل تنفيذ الإجراء المحدد.
- يتم حصر الحل في هدف الإجراء النشط عندما يكون ذلك ممكنًا:
  - على نطاق القناة عند ضبط `--channel` ‏(أو عند استنتاجه من الأهداف ذات البادئة مثل `discord:...`)
  - على نطاق الحساب عند ضبط `--account` ‏(أسطح القناة العامة + أسطح الحساب المحدد)
  - عند حذف `--account`، لا يفرض OpenClaw نطاق SecretRef لحساب `default`
- لا تمنع SecretRefs غير المحلولة في القنوات غير المرتبطة إجراء رسالة مستهدفًا.
- إذا كان SecretRef الخاص بالقناة/الحساب المحدد غير محلول، يفشل الأمر بشكل مغلق لذلك الإجراء.

## الإجراءات

### الأساسية

- `send`
  - القنوات: WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Matrix/Microsoft Teams
  - المطلوب: `--target`، بالإضافة إلى `--message` أو `--media`
  - الاختياري: `--media` و`--interactive` و`--buttons` و`--components` و`--card` و`--reply-to` و`--thread-id` و`--gif-playback` و`--force-document` و`--silent`
  - الحمولات التفاعلية المشتركة: يرسل `--interactive` حمولة JSON تفاعلية أصلية للقناة عندما تكون مدعومة
  - Telegram فقط: `--buttons` ‏(يتطلب أن يسمح `channels.telegram.capabilities.inlineButtons` بذلك)
  - Telegram فقط: `--force-document` ‏(إرسال الصور وملفات GIF كمستندات لتجنب ضغط Telegram)
  - Telegram فقط: `--thread-id` ‏(معرّف موضوع المنتدى)
  - Slack فقط: `--thread-id` ‏(الطابع الزمني للسلسلة؛ ويستخدم `--reply-to` الحقل نفسه)
  - Discord فقط: حمولة JSON لـ `--components`
  - القنوات التي تدعم Adaptive Card: حمولة JSON لـ `--card` عندما تكون مدعومة
  - Telegram + Discord: ‏`--silent`
  - WhatsApp فقط: `--gif-playback`

- `poll`
  - القنوات: WhatsApp/Telegram/Discord/Matrix/Microsoft Teams
  - المطلوب: `--target` و`--poll-question` و`--poll-option` ‏(قابل للتكرار)
  - الاختياري: `--poll-multi`
  - Discord فقط: `--poll-duration-hours` و`--silent` و`--message`
  - Telegram فقط: `--poll-duration-seconds` ‏(5-600) و`--silent` و`--poll-anonymous` / `--poll-public` و`--thread-id`

- `react`
  - القنوات: Discord/Google Chat/Slack/Telegram/WhatsApp/Signal/Matrix
  - المطلوب: `--message-id` و`--target`
  - الاختياري: `--emoji` و`--remove` و`--participant` و`--from-me` و`--target-author` و`--target-author-uuid`
  - ملاحظة: يتطلب `--remove` وجود `--emoji` ‏(احذف `--emoji` لمسح تفاعلاتك الخاصة عند الدعم؛ راجع /tools/reactions)
  - WhatsApp فقط: `--participant` و`--from-me`
  - تفاعلات مجموعات Signal: يتطلب `--target-author` أو `--target-author-uuid`

- `reactions`
  - القنوات: Discord/Google Chat/Slack/Matrix
  - المطلوب: `--message-id` و`--target`
  - الاختياري: `--limit`

- `read`
  - القنوات: Discord/Slack/Matrix
  - المطلوب: `--target`
  - الاختياري: `--limit` و`--before` و`--after`
  - Discord فقط: `--around`

- `edit`
  - القنوات: Discord/Slack/Matrix
  - المطلوب: `--message-id` و`--message` و`--target`

- `delete`
  - القنوات: Discord/Slack/Telegram/Matrix
  - المطلوب: `--message-id` و`--target`

- `pin` / `unpin`
  - القنوات: Discord/Slack/Matrix
  - المطلوب: `--message-id` و`--target`

- `pins` ‏(عرض القائمة)
  - القنوات: Discord/Slack/Matrix
  - المطلوب: `--target`

- `permissions`
  - القنوات: Discord/Matrix
  - المطلوب: `--target`
  - Matrix فقط: متاح عند تمكين تشفير Matrix والسماح بإجراءات التحقق

- `search`
  - القنوات: Discord
  - المطلوب: `--guild-id` و`--query`
  - الاختياري: `--channel-id` و`--channel-ids` ‏(قابل للتكرار) و`--author-id` و`--author-ids` ‏(قابل للتكرار) و`--limit`

### السلاسل

- `thread create`
  - القنوات: Discord
  - المطلوب: `--thread-name` و`--target` ‏(معرّف القناة)
  - الاختياري: `--message-id` و`--message` و`--auto-archive-min`

- `thread list`
  - القنوات: Discord
  - المطلوب: `--guild-id`
  - الاختياري: `--channel-id` و`--include-archived` و`--before` و`--limit`

- `thread reply`
  - القنوات: Discord
  - المطلوب: `--target` ‏(معرّف السلسلة) و`--message`
  - الاختياري: `--media` و`--reply-to`

### الرموز التعبيرية

- `emoji list`
  - Discord: ‏`--guild-id`
  - Slack: لا توجد أعلام إضافية

- `emoji upload`
  - القنوات: Discord
  - المطلوب: `--guild-id` و`--emoji-name` و`--media`
  - الاختياري: `--role-ids` ‏(قابل للتكرار)

### الملصقات

- `sticker send`
  - القنوات: Discord
  - المطلوب: `--target` و`--sticker-id` ‏(قابل للتكرار)
  - الاختياري: `--message`

- `sticker upload`
  - القنوات: Discord
  - المطلوب: `--guild-id` و`--sticker-name` و`--sticker-desc` و`--sticker-tags` و`--media`

### الأدوار / القنوات / الأعضاء / الصوت

- `role info` ‏(Discord): ‏`--guild-id`
- `role add` / `role remove` ‏(Discord): ‏`--guild-id` و`--user-id` و`--role-id`
- `channel info` ‏(Discord): ‏`--target`
- `channel list` ‏(Discord): ‏`--guild-id`
- `member info` ‏(Discord/Slack): ‏`--user-id` ‏(+ `--guild-id` لـ Discord)
- `voice status` ‏(Discord): ‏`--guild-id` و`--user-id`

### الأحداث

- `event list` ‏(Discord): ‏`--guild-id`
- `event create` ‏(Discord): ‏`--guild-id` و`--event-name` و`--start-time`
  - الاختياري: `--end-time` و`--desc` و`--channel-id` و`--location` و`--event-type`

### الإشراف (Discord)

- `timeout`: ‏`--guild-id` و`--user-id` ‏(اختياري `--duration-min` أو `--until`؛ احذف كليهما لإزالة timeout)
- `kick`: ‏`--guild-id` و`--user-id` ‏(+ `--reason`)
- `ban`: ‏`--guild-id` و`--user-id` ‏(+ `--delete-days` و`--reason`)
  - يدعم `timeout` أيضًا `--reason`

### البث

- `broadcast`
  - القنوات: أي قناة مهيأة؛ استخدم `--channel all` لاستهداف جميع المزوّدين
  - المطلوب: `--targets <target...>`
  - الاختياري: `--message` و`--media` و`--dry-run`

## أمثلة

إرسال رد في Discord:

```
openclaw message send --channel discord \
  --target channel:123 --message "hi" --reply-to 456
```

إرسال رسالة Discord مع components:

```
openclaw message send --channel discord \
  --target channel:123 --message "Choose:" \
  --components '{"text":"Choose a path","blocks":[{"type":"actions","buttons":[{"label":"Approve","style":"success"},{"label":"Decline","style":"danger"}]}]}'
```

راجع [Discord components](/channels/discord#interactive-components) للحصول على المخطط الكامل.

إرسال حمولة تفاعلية مشتركة:

```bash
openclaw message send --channel googlechat --target spaces/AAA... \
  --message "Choose:" \
  --interactive '{"text":"Choose a path","blocks":[{"type":"actions","buttons":[{"label":"Approve"},{"label":"Decline"}]}]}'
```

إنشاء poll في Discord:

```
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "Snack?" \
  --poll-option Pizza --poll-option Sushi \
  --poll-multi --poll-duration-hours 48
```

إنشاء poll في Telegram ‏(إغلاق تلقائي بعد دقيقتين):

```
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "Lunch?" \
  --poll-option Pizza --poll-option Sushi \
  --poll-duration-seconds 120 --silent
```

إرسال رسالة استباقية في Teams:

```
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 --message "hi"
```

إنشاء poll في Teams:

```
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "Lunch?" \
  --poll-option Pizza --poll-option Sushi
```

إضافة تفاعل في Slack:

```
openclaw message react --channel slack \
  --target C123 --message-id 456 --emoji "✅"
```

إضافة تفاعل في مجموعة Signal:

```
openclaw message react --channel signal \
  --target signal:group:abc123 --message-id 1737630212345 \
  --emoji "✅" --target-author-uuid 123e4567-e89b-12d3-a456-426614174000
```

إرسال أزرار inline في Telegram:

```
openclaw message send --channel telegram --target @mychat --message "Choose:" \
  --buttons '[ [{"text":"Yes","callback_data":"cmd:yes"}], [{"text":"No","callback_data":"cmd:no"}] ]'
```

إرسال Adaptive Card في Teams:

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Status update"}]}'
```

إرسال صورة Telegram كمستند لتجنب الضغط:

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```
