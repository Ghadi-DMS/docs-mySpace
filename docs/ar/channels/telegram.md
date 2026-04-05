---
read_when:
    - تعمل على ميزات Telegram أو webhooks
summary: حالة دعم Telegram bot، والإمكانات، والتكوين
title: Telegram
x-i18n:
    generated_at: "2026-04-05T12:38:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39fbf328375fbc5d08ec2e3eed58b19ee0afa102010ecbc02e074a310ced157e
    source_path: channels/telegram.md
    workflow: 15
---

# Telegram (Bot API)

الحالة: جاهز للإنتاج لرسائل bot الخاصة والمجموعات عبر grammY. يكون long polling هو الوضع الافتراضي؛ ووضع webhook اختياري.

<CardGroup cols={3}>
  <Card title="الإقران" icon="link" href="/channels/pairing">
    سياسة الرسائل الخاصة الافتراضية في Telegram هي pairing.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/channels/troubleshooting">
    أدلة تشخيص وإصلاح عبر القنوات.
  </Card>
  <Card title="تكوين البوابة" icon="settings" href="/gateway/configuration">
    أنماط وأمثلة كاملة لتكوين القنوات.
  </Card>
</CardGroup>

## الإعداد السريع

<Steps>
  <Step title="أنشئ token البوت في BotFather">
    افتح Telegram وتحدث مع **@BotFather** (تأكد من أن اسم المستخدم هو `@BotFather` تمامًا).

    نفّذ `/newbot`، واتبع التعليمات، واحفظ token.

  </Step>

  <Step title="كوّن token وسياسة الرسائل الخاصة">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    الرجوع إلى متغيرات البيئة: `TELEGRAM_BOT_TOKEN=...` (للحساب الافتراضي فقط).
    لا يستخدم Telegram الأمر `openclaw channels login telegram`؛ كوّن token في التكوين/البيئة، ثم ابدأ البوابة.

  </Step>

  <Step title="ابدأ البوابة ووافق على أول رسالة خاصة">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    تنتهي صلاحية رموز pairing بعد ساعة واحدة.

  </Step>

  <Step title="أضف البوت إلى مجموعة">
    أضف البوت إلى مجموعتك، ثم اضبط `channels.telegram.groups` و`groupPolicy` بما يتوافق مع نموذج الوصول لديك.
  </Step>
</Steps>

<Note>
ترتيب حل token يراعي الحسابات. عمليًا، تفوز قيم التكوين على الرجوع إلى البيئة، و`TELEGRAM_BOT_TOKEN` ينطبق فقط على الحساب الافتراضي.
</Note>

## إعدادات جانب Telegram

<AccordionGroup>
  <Accordion title="وضع الخصوصية ورؤية المجموعات">
    تعمل bots Telegram افتراضيًا في **Privacy Mode**، ما يحد من رسائل المجموعات التي تتلقاها.

    إذا كان يجب أن يرى البوت كل رسائل المجموعة، فإما أن:

    - تعطل وضع الخصوصية عبر `/setprivacy`، أو
    - تجعل البوت مشرفًا على المجموعة.

    عند تبديل وضع الخصوصية، أزل البوت ثم أعد إضافته في كل مجموعة حتى يطبّق Telegram التغيير.

  </Accordion>

  <Accordion title="أذونات المجموعة">
    يتم التحكم في حالة المشرف من إعدادات مجموعة Telegram.

    تتلقى bots المشرفة جميع رسائل المجموعة، وهذا مفيد لسلوك المجموعات الدائم التشغيل.

  </Accordion>

  <Accordion title="خيارات مفيدة في BotFather">

    - `/setjoingroups` للسماح/منع الإضافة إلى المجموعات
    - `/setprivacy` لسلوك الرؤية داخل المجموعات

  </Accordion>
</AccordionGroup>

## التحكم في الوصول والتفعيل

<Tabs>
  <Tab title="سياسة الرسائل الخاصة">
    يتحكم `channels.telegram.dmPolicy` في الوصول إلى الرسائل الخاصة:

    - `pairing` (الافتراضي)
    - `allowlist` (يتطلب وجود معرّف مرسل واحد على الأقل في `allowFrom`)
    - `open` (يتطلب أن تتضمن `allowFrom` القيمة `"*"`)
    - `disabled`

    يقبل `channels.telegram.allowFrom` معرّفات مستخدمي Telegram الرقمية. وتُقبل البادئات `telegram:` و`tg:` ويتم توحيدها.
    يؤدي `dmPolicy: "allowlist"` مع `allowFrom` فارغة إلى حظر جميع الرسائل الخاصة ويتم رفضه عبر التحقق من صحة التكوين.
    يقبل الإعداد التفاعلي إدخال `@username` ويحوّله إلى معرّفات رقمية.
    إذا قمت بالترقية وكان التكوين لديك يحتوي على إدخالات allowlist بصيغة `@username`، فشغّل `openclaw doctor --fix` لتحويلها (بأفضل جهد؛ ويتطلب token Telegram bot).
    إذا كنت تعتمد سابقًا على ملفات allowlist في pairing-store، فيمكن لـ `openclaw doctor --fix` استعادة الإدخالات إلى `channels.telegram.allowFrom` في تدفقات allowlist (على سبيل المثال عندما يكون `dmPolicy: "allowlist"` بلا معرّفات صريحة بعد).

    بالنسبة إلى bots ذات المالك الواحد، فضّل `dmPolicy: "allowlist"` مع معرّفات `allowFrom` رقمية صريحة للحفاظ على سياسة الوصول بشكل مستدام في التكوين (بدلًا من الاعتماد على موافقات pairing السابقة).

    التباس شائع: الموافقة على pairing في الرسائل الخاصة لا تعني أن "هذا المرسل مصرح له في كل مكان".
    يمنح pairing الوصول إلى الرسائل الخاصة فقط. أما تفويض مرسل المجموعة فما زال يأتي من allowlists الصريحة في التكوين.
    إذا أردت "أن أُصرَّح مرة واحدة وأن تعمل الرسائل الخاصة وأوامر المجموعة معًا"، فضع معرّف مستخدم Telegram الرقمي الخاص بك في `channels.telegram.allowFrom`.

    ### العثور على معرّف مستخدم Telegram الخاص بك

    الطريقة الأكثر أمانًا (من دون bot تابع لجهة خارجية):

    1. أرسل رسالة خاصة إلى البوت.
    2. نفّذ `openclaw logs --follow`.
    3. اقرأ `from.id`.

    طريقة Bot API الرسمية:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    طريقة تابعة لجهة خارجية (خصوصية أقل): `@userinfobot` أو `@getidsbot`.

  </Tab>

  <Tab title="سياسة المجموعات وallowlists">
    يُطبَّق عنصران للتحكم معًا:

    1. **أي المجموعات مسموح بها** (`channels.telegram.groups`)
       - بدون تكوين `groups`:
         - مع `groupPolicy: "open"`: يمكن لأي مجموعة اجتياز فحوصات معرّف المجموعة
         - مع `groupPolicy: "allowlist"` (الافتراضي): تُحظر المجموعات حتى تضيف إدخالات في `groups` (أو `"*"`)
       - عند تكوين `groups`: تعمل كـ allowlist (معرّفات صريحة أو `"*"`)

    2. **أي المرسلين مسموح بهم داخل المجموعات** (`channels.telegram.groupPolicy`)
       - `open`
       - `allowlist` (الافتراضي)
       - `disabled`

    يُستخدم `groupAllowFrom` لتصفية مرسلي المجموعات. وإذا لم يُضبط، فسيعود Telegram إلى `allowFrom`.
    يجب أن تكون إدخالات `groupAllowFrom` معرّفات مستخدمي Telegram رقمية (وتُوحَّد البادئات `telegram:` / `tg:`).
    لا تضع معرّفات دردشات مجموعات Telegram أو المجموعات الفائقة في `groupAllowFrom`. فمعرّفات الدردشة السالبة يجب أن تكون ضمن `channels.telegram.groups`.
    تُتجاهل الإدخالات غير الرقمية عند تفويض المرسل.
    الحد الأمني (`2026.2.25+`): لا يرث تفويض مرسل المجموعة موافقات pairing-store الخاصة بالرسائل الخاصة.
    يظل pairing خاصًا بالرسائل الخاصة فقط. وبالنسبة إلى المجموعات، اضبط `groupAllowFrom` أو `allowFrom` لكل مجموعة/موضوع.
    إذا لم يُضبط `groupAllowFrom`، يعود Telegram إلى `allowFrom` في التكوين، وليس إلى pairing store.
    النمط العملي للـ bots ذات المالك الواحد: اضبط معرّف المستخدم الخاص بك في `channels.telegram.allowFrom`، واترك `groupAllowFrom` غير مضبوط، واسمح للمجموعات المستهدفة ضمن `channels.telegram.groups`.
    ملاحظة وقت التشغيل: إذا كان `channels.telegram` مفقودًا بالكامل، فستكون القيم الافتراضية وقت التشغيل fail-closed مع `groupPolicy="allowlist"` ما لم يتم ضبط `channels.defaults.groupPolicy` صراحةً.

    مثال: السماح لأي عضو في مجموعة محددة واحدة:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    مثال: السماح لمستخدمين محددين فقط داخل مجموعة محددة واحدة:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      خطأ شائع: `groupAllowFrom` ليست allowlist لمجموعات Telegram.

      - ضع معرّفات دردشات مجموعات Telegram أو المجموعات الفائقة السالبة مثل `-1001234567890` ضمن `channels.telegram.groups`.
      - ضع معرّفات مستخدمي Telegram مثل `8734062810` ضمن `groupAllowFrom` عندما تريد تقييد الأشخاص داخل مجموعة مسموح بها الذين يمكنهم تشغيل البوت.
      - استخدم `groupAllowFrom: ["*"]` فقط عندما تريد أن يتمكن أي عضو في مجموعة مسموح بها من التحدث إلى البوت.
    </Warning>

  </Tab>

  <Tab title="سلوك الإشارات">
    تتطلب الردود في المجموعات وجود إشارة افتراضيًا.

    يمكن أن تأتي الإشارة من:

    - إشارة أصلية `@botusername`، أو
    - أنماط الإشارة في:
      - `agents.list[].groupChat.mentionPatterns`
      - `messages.groupChat.mentionPatterns`

    مفاتيح تبديل الأوامر على مستوى الجلسة:

    - `/activation always`
    - `/activation mention`

    تحدّث هذه الأوامر حالة الجلسة فقط. استخدم التكوين للاستمرارية.

    مثال على تكوين دائم:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    الحصول على معرّف دردشة المجموعة:

    - أعد توجيه رسالة من المجموعة إلى `@userinfobot` / `@getidsbot`
    - أو اقرأ `chat.id` من `openclaw logs --follow`
    - أو افحص `getUpdates` في Bot API

  </Tab>
</Tabs>

## سلوك وقت التشغيل

- يملك Telegram عملية البوابة.
- التوجيه حتمي: ترد رسائل Telegram الواردة إلى Telegram (ولا يختار النموذج القنوات).
- تُوحَّد الرسائل الواردة داخل غلاف القنوات المشترك مع بيانات الرد الوصفية والعناصر النائبة للوسائط.
- تُعزل جلسات المجموعات بحسب معرّف المجموعة. وتُلحق موضوعات المنتدى `:topic:<threadId>` للحفاظ على عزل الموضوعات.
- يمكن أن تحمل الرسائل الخاصة `message_thread_id`؛ ويقوم OpenClaw بتوجيهها باستخدام مفاتيح جلسة تراعي thread ويحافظ على معرّف thread في الردود.
- يستخدم long polling مشغّل grammY مع تسلسل لكل دردشة/لكل thread. ويستخدم تزامن sink العام للمشغّل `agents.defaults.maxConcurrent`.
- لا يحتوي Telegram Bot API على دعم لإيصالات القراءة (`sendReadReceipts` غير منطبق).

## مرجع الميزات

<AccordionGroup>
  <Accordion title="معاينة البث المباشر (تعديلات الرسائل)">
    يمكن لـ OpenClaw بث الردود الجزئية في الوقت الفعلي:

    - الدردشات الخاصة: رسالة معاينة + `editMessageText`
    - المجموعات/الموضوعات: رسالة معاينة + `editMessageText`

    المتطلب:

    - أن تكون `channels.telegram.streaming` إحدى القيم `off | partial | block | progress` (الافتراضي: `partial`)
    - تُحوَّل `progress` إلى `partial` على Telegram (للتوافق مع التسمية عبر القنوات)
    - تُحوَّل تلقائيًا قيم `channels.telegram.streamMode` القديمة وقيم `streaming` المنطقية

    بالنسبة إلى الردود النصية فقط:

    - الرسائل الخاصة: يحتفظ OpenClaw برسالة المعاينة نفسها ويجري تعديلًا نهائيًا في المكان نفسه (من دون رسالة ثانية)
    - المجموعة/الموضوع: يحتفظ OpenClaw برسالة المعاينة نفسها ويجري تعديلًا نهائيًا في المكان نفسه (من دون رسالة ثانية)

    بالنسبة إلى الردود المعقدة (مثل حمولات الوسائط)، يعود OpenClaw إلى التسليم النهائي العادي ثم ينظف رسالة المعاينة.

    بث المعاينة منفصل عن block streaming. عندما يتم تمكين block streaming صراحةً في Telegram، يتخطى OpenClaw بث المعاينة لتجنب البث المزدوج.

    إذا كان نقل المسودة الأصلي غير متاح/مرفوض، يعود OpenClaw تلقائيًا إلى `sendMessage` + `editMessageText`.

    بث الاستدلال الخاص بـ Telegram:

    - `/reasoning stream` يرسل الاستدلال إلى المعاينة المباشرة أثناء التوليد
    - يُرسل الجواب النهائي من دون نص الاستدلال

  </Accordion>

  <Accordion title="التنسيق والرجوع إلى HTML">
    يستخدم النص الصادر `parse_mode: "HTML"` في Telegram.

    - يُعرَض النص الشبيه بـ Markdown إلى HTML آمن لـ Telegram.
    - يُهَرب HTML الخام من النموذج لتقليل حالات فشل التحليل في Telegram.
    - إذا رفض Telegram HTML المحلل، يعيد OpenClaw المحاولة كنص عادي.

    تُفعَّل معاينات الروابط افتراضيًا ويمكن تعطيلها باستخدام `channels.telegram.linkPreview: false`.

  </Accordion>

  <Accordion title="الأوامر الأصلية والأوامر المخصصة">
    يتم تسجيل قائمة أوامر Telegram عند بدء التشغيل عبر `setMyCommands`.

    الإعدادات الافتراضية للأوامر الأصلية:

    - `commands.native: "auto"` يفعّل الأوامر الأصلية لـ Telegram

    أضف إدخالات أوامر مخصصة إلى القائمة:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
    },
  },
}
```

    القواعد:

    - تُوحَّد الأسماء (تُزال الشرطة المائلة `/` من البداية، وتحوَّل إلى أحرف صغيرة)
    - النمط الصحيح: `a-z` و`0-9` و`_`، والطول `1..32`
    - لا يمكن للأوامر المخصصة تجاوز الأوامر الأصلية
    - تُتخطى التعارضات/التكرارات ويُسجل ذلك

    ملاحظات:

    - الأوامر المخصصة هي إدخالات في القائمة فقط؛ ولا تنفذ السلوك تلقائيًا
    - يمكن أن تستمر أوامر plugin/Skills في العمل عند كتابتها حتى إذا لم تظهر في قائمة Telegram

    إذا عُطلت الأوامر الأصلية، فستزال الأوامر المضمنة. وقد تستمر أوامر custom/plugin في التسجيل إذا كانت مكوّنة.

    حالات فشل إعداد شائعة:

    - فشل `setMyCommands failed` مع `BOT_COMMANDS_TOO_MUCH` يعني أن قائمة Telegram ما زالت زائدة بعد التقليص؛ قلّل أوامر plugin/Skills/الأوامر المخصصة أو عطّل `channels.telegram.commands.native`.
    - فشل `setMyCommands failed` مع أخطاء الشبكة/fetch يعني عادةً أن DNS/HTTPS الصادر إلى `api.telegram.org` محجوب.

    ### أوامر إقران الأجهزة (`device-pair` plugin)

    عند تثبيت plugin `device-pair`:

    1. يولد `/pair` رمز الإعداد
    2. الصق الرمز في تطبيق iOS
    3. يسرد `/pair pending` الطلبات المعلقة (بما في ذلك الدور/النطاقات)
    4. وافق على الطلب:
       - `/pair approve <requestId>` للموافقة الصريحة
       - `/pair approve` عندما يكون هناك طلب معلق واحد فقط
       - `/pair approve latest` للأحدث

    يحمل رمز الإعداد bootstrap token قصير العمر. ويحافظ bootstrap handoff المضمن على token العقدة الأساسية عند `scopes: []`؛ وأي operator token يتم تسليمه يبقى محصورًا في `operator.approvals` و`operator.read` و`operator.talk.secrets` و`operator.write`. وتكون فحوصات نطاق bootstrap مسبوقة بالدور، لذلك فإن allowlist الخاصة بالمشغل لا تلبّي إلا طلبات المشغل؛ أما الأدوار غير المشغلة فما زالت تحتاج إلى scopes تحت بادئة دورها الخاص.

    إذا أعاد جهاز المحاولة مع تفاصيل مصادقة متغيرة (مثل الدور/النطاقات/المفتاح العام)، فسيتم استبدال الطلب المعلق السابق وسيستخدم الطلب الجديد `requestId` مختلفًا. أعد تنفيذ `/pair pending` قبل الموافقة.

    مزيد من التفاصيل: [الإقران](/channels/pairing#pair-via-telegram-recommended-for-ios).

  </Accordion>

  <Accordion title="الأزرار المضمنة">
    كوّن نطاق لوحة المفاتيح المضمنة:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    تجاوز لكل حساب:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    النطاقات:

    - `off`
    - `dm`
    - `group`
    - `all`
    - `allowlist` (الافتراضي)

    تتحول `capabilities: ["inlineButtons"]` القديمة إلى `inlineButtons: "all"`.

    مثال على إجراء رسالة:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose an option:",
  buttons: [
    [
      { text: "Yes", callback_data: "yes" },
      { text: "No", callback_data: "no" },
    ],
    [{ text: "Cancel", callback_data: "cancel" }],
  ],
}
```

    تُمرر نقرات callback إلى الوكيل كنص:
    `callback_data: <value>`

  </Accordion>

  <Accordion title="إجراءات رسائل Telegram للوكلاء والأتمتة">
    تتضمن إجراءات أداة Telegram ما يلي:

    - `sendMessage` (`to`, `content`, واختياريًا `mediaUrl` و`replyToMessageId` و`messageThreadId`)
    - `react` (`chatId`, `messageId`, `emoji`)
    - `deleteMessage` (`chatId`, `messageId`)
    - `editMessage` (`chatId`, `messageId`, `content`)
    - `createForumTopic` (`chatId`, `name`, واختياريًا `iconColor` و`iconCustomEmojiId`)

    تعرض إجراءات رسائل القنوات أسماء بديلة عملية (`send` و`react` و`delete` و`edit` و`sticker` و`sticker-search` و`topic-create`).

    عناصر التحكم في التقييد:

    - `channels.telegram.actions.sendMessage`
    - `channels.telegram.actions.deleteMessage`
    - `channels.telegram.actions.reactions`
    - `channels.telegram.actions.sticker` (الافتراضي: معطل)

    ملاحظة: `edit` و`topic-create` مفعّلان افتراضيًا حاليًا ولا يملكان مفاتيح تبديل منفصلة ضمن `channels.telegram.actions.*`.
    تستخدم الإرسالات وقت التشغيل snapshot النشط للتكوين/الأسرار (بدء التشغيل/إعادة التحميل)، لذلك لا تنفّذ مسارات الإجراء إعادة حل SecretRef مخصصة لكل إرسال.

    دلالات إزالة التفاعل: [/tools/reactions](/tools/reactions)

  </Accordion>

  <Accordion title="وسوم ربط الردود">
    يدعم Telegram وسوم ربط الردود الصريحة في المخرجات المولدة:

    - `[[reply_to_current]]` يرد على الرسالة المُشغِّلة
    - `[[reply_to:<id>]]` يرد على معرّف رسالة Telegram محدد

    يتحكم `channels.telegram.replyToMode` في المعالجة:

    - `off` (الافتراضي)
    - `first`
    - `all`

    ملاحظة: `off` يعطّل ربط الردود الضمني. لكن ما زالت وسوم `[[reply_to_*]]` الصريحة محترمة.

  </Accordion>

  <Accordion title="موضوعات المنتدى وسلوك thread">
    في المجموعات الفائقة من نوع المنتدى:

    - تُلحق مفاتيح جلسات الموضوعات بـ `:topic:<threadId>`
    - تستهدف الردود وإشارات الكتابة thread الخاص بالموضوع
    - مسار تكوين الموضوع:
      `channels.telegram.groups.<chatId>.topics.<threadId>`

    الحالة الخاصة للموضوع العام (`threadId=1`):

    - تُرسل الرسائل من دون `message_thread_id` (يرفض Telegram `sendMessage(...thread_id=1)`)
    - ما زالت إجراءات الكتابة تتضمن `message_thread_id`

    وراثة الموضوعات: ترث إدخالات الموضوع إعدادات المجموعة ما لم يتم تجاوزها (`requireMention` و`allowFrom` و`skills` و`systemPrompt` و`enabled` و`groupPolicy`).
    `agentId` خاص بالموضوع فقط ولا يرث من الإعدادات الافتراضية للمجموعة.

    **توجيه الوكيل لكل موضوع**: يمكن لكل موضوع التوجيه إلى وكيل مختلف عبر ضبط `agentId` في تكوين الموضوع. وهذا يمنح كل موضوع مساحة عمل وذاكرة وجلسة معزولة خاصة به. مثال:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // الموضوع العام → الوكيل الرئيسي
                "3": { agentId: "zu" },        // موضوع التطوير → الوكيل zu
                "5": { agentId: "coder" }      // مراجعة الشيفرة → الوكيل coder
              }
            }
          }
        }
      }
    }
    ```

    يمتلك كل موضوع عندئذٍ مفتاح جلسة خاصًا به: `agent:zu:telegram:group:-1001234567890:topic:3`

    **ربط ACP دائم للموضوع**: يمكن لموضوعات المنتدى تثبيت جلسات harness الخاصة بـ ACP عبر عمليات ربط ACP مكتوبة على المستوى الأعلى:

    - `bindings[]` مع `type: "acp"` و`match.channel: "telegram"`

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
            channel: "telegram",
            accountId: "default",
            peer: { kind: "group", id: "-1001234567890:topic:42" },
          },
        },
      ],
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "42": {
                  requireMention: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    يقتصر هذا حاليًا على موضوعات المنتدى في المجموعات والمجموعات الفائقة.

    **إنشاء ACP مرتبط بـ thread من الدردشة**:

    - يمكن للأمر `/acp spawn <agent> --thread here|auto` ربط موضوع Telegram الحالي بجلسة ACP جديدة.
    - تُوجَّه رسائل الموضوع اللاحقة مباشرة إلى جلسة ACP المرتبطة (من دون الحاجة إلى `/acp steer`).
    - يثبت OpenClaw رسالة تأكيد الإنشاء داخل الموضوع بعد نجاح الربط.
    - يتطلب `channels.telegram.threadBindings.spawnAcpSessions=true`.

    يتضمن سياق القالب ما يلي:

    - `MessageThreadId`
    - `IsForum`

    سلوك thread في الرسائل الخاصة:

    - تحافظ الدردشات الخاصة التي تحتوي على `message_thread_id` على توجيه الرسائل الخاصة لكنها تستخدم مفاتيح جلسة/أهداف رد تراعي thread.

  </Accordion>

  <Accordion title="الصوت والفيديو والملصقات">
    ### الرسائل الصوتية

    يميز Telegram بين الملاحظات الصوتية وملفات الصوت.

    - الافتراضي: سلوك ملف صوتي
    - الوسم `[[audio_as_voice]]` في رد الوكيل لفرض الإرسال كملاحظة صوتية

    مثال على إجراء رسالة:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### رسائل الفيديو

    يميز Telegram بين ملفات الفيديو وملاحظات الفيديو.

    مثال على إجراء رسالة:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    لا تدعم ملاحظات الفيديو التعليقات التوضيحية؛ ويُرسل نص الرسالة المقدم بشكل منفصل.

    ### الملصقات

    معالجة الملصقات الواردة:

    - WEBP ثابت: يتم تنزيله ومعالجته (عنصر نائب `<media:sticker>`)
    - TGS متحرك: يتم تخطيه
    - WEBM فيديو: يتم تخطيه

    حقول سياق الملصق:

    - `Sticker.emoji`
    - `Sticker.setName`
    - `Sticker.fileId`
    - `Sticker.fileUniqueId`
    - `Sticker.cachedDescription`

    ملف cache الخاص بالملصقات:

    - `~/.openclaw/telegram/sticker-cache.json`

    تُوصف الملصقات مرة واحدة (عند الإمكان) وتُخزَّن في cache لتقليل استدعاءات الرؤية المتكررة.

    فعّل إجراءات الملصقات:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    إجراء إرسال ملصق:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    ابحث في الملصقات المخزنة في cache:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="إشعارات التفاعلات">
    تصل تفاعلات Telegram كتحديثات `message_reaction` (منفصلة عن حمولات الرسائل).

    عند التمكين، يضع OpenClaw أحداث النظام التالية في قائمة الانتظار مثلًا:

    - `Telegram reaction added: 👍 by Alice (@alice) on msg 42`

    التكوين:

    - `channels.telegram.reactionNotifications`: `off | own | all` (الافتراضي: `own`)
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` (الافتراضي: `minimal`)

    ملاحظات:

    - تعني `own` تفاعلات المستخدمين على الرسائل التي أرسلها البوت فقط (بأفضل جهد عبر cache الرسائل المرسلة).
    - ما زالت أحداث التفاعل تراعي عناصر التحكم في الوصول في Telegram (`dmPolicy` و`allowFrom` و`groupPolicy` و`groupAllowFrom`)؛ ويُسقَط المرسلون غير المصرح لهم.
    - لا يوفّر Telegram معرّفات thread في تحديثات التفاعل.
      - تُوجَّه المجموعات غير التابعة للمنتدى إلى جلسة دردشة المجموعة
      - وتُوجَّه مجموعات المنتدى إلى جلسة الموضوع العام للمجموعة (`:topic:1`)، وليس إلى الموضوع الأصلي بالضبط

    تتضمن `allowed_updates` الخاصة بـ polling/webhook قيمة `message_reaction` تلقائيًا.

  </Accordion>

  <Accordion title="تفاعلات التأكيد">
    يرسل `ackReaction` رمز emoji للتأكيد بينما يعالج OpenClaw رسالة واردة.

    ترتيب الحل:

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - الرجوع إلى emoji هوية الوكيل (`agents.list[].identity.emoji`، وإلا "👀")

    ملاحظات:

    - يتوقع Telegram emoji موحدًا من Unicode (مثل "👀").
    - استخدم `""` لتعطيل التفاعل لقناة أو حساب.

  </Accordion>

  <Accordion title="كتابات التكوين من أحداث وأوامر Telegram">
    تكون كتابات تكوين القناة مفعلة افتراضيًا (`configWrites !== false`).

    تشمل الكتابات التي يطلقها Telegram ما يلي:

    - أحداث ترحيل المجموعات (`migrate_to_chat_id`) لتحديث `channels.telegram.groups`
    - `/config set` و`/config unset` (يتطلب تمكين الأوامر)

    التعطيل:

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Long polling مقابل webhook">
    الافتراضي: long polling.

    وضع webhook:

    - اضبط `channels.telegram.webhookUrl`
    - اضبط `channels.telegram.webhookSecret` (مطلوب عند ضبط webhook URL)
    - اختياريًا `channels.telegram.webhookPath` (الافتراضي `/telegram-webhook`)
    - اختياريًا `channels.telegram.webhookHost` (الافتراضي `127.0.0.1`)
    - اختياريًا `channels.telegram.webhookPort` (الافتراضي `8787`)

    يرتبط مستمع webhook المحلي الافتراضي في `127.0.0.1:8787`.

    إذا كانت نقطة النهاية العامة لديك مختلفة، فضع reverse proxy في الأمام ووجّه `webhookUrl` إلى الرابط العام.
    اضبط `webhookHost` (مثل `0.0.0.0`) عندما تحتاج عمدًا إلى ingress خارجي.

  </Accordion>

  <Accordion title="الحدود وإعادة المحاولة وأهداف CLI">
    - القيمة الافتراضية لـ `channels.telegram.textChunkLimit` هي 4000.
    - يفضّل `channels.telegram.chunkMode="newline"` حدود الفقرات (الأسطر الفارغة) قبل التقسيم حسب الطول.
    - يحدد `channels.telegram.mediaMaxMb` (الافتراضي 100) الحد الأقصى لحجم الوسائط الواردة والصادرة في Telegram.
    - يتجاوز `channels.telegram.timeoutSeconds` مهلة عميل Telegram API (إذا لم يُضبط، تُطبَّق القيمة الافتراضية لـ grammY).
    - يستخدم سجل سياق المجموعات `channels.telegram.historyLimit` أو `messages.groupChat.historyLimit` (الافتراضي 50)؛ و`0` يعطّل الميزة.
    - يُمرَّر حاليًا سياق الرد/الاقتباس/إعادة التوجيه التكميلي كما تم استلامه.
    - تعمل allowlists في Telegram أساسًا على تقييد من يمكنه تشغيل الوكيل، وليست حدًا كاملًا لتنقيح السياق التكميلي.
    - عناصر التحكم في سجل الرسائل الخاصة:
      - `channels.telegram.dmHistoryLimit`
      - `channels.telegram.dms["<user_id>"].historyLimit`
    - ينطبق تكوين `channels.telegram.retry` على مساعدات الإرسال الخاصة بـ Telegram (CLI/tools/actions) في أخطاء Telegram API الصادرة القابلة للاسترداد.

    يمكن أن يكون هدف الإرسال في CLI معرّف دردشة رقميًا أو اسم مستخدم:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
```

    تستخدم استطلاعات Telegram الأمر `openclaw message poll` وتدعم موضوعات المنتدى:

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    أعلام الاستطلاع الخاصة بـ Telegram فقط:

    - `--poll-duration-seconds` (من 5 إلى 600)
    - `--poll-anonymous`
    - `--poll-public`
    - `--thread-id` لموضوعات المنتدى (أو استخدم هدف `:topic:`)

    يدعم إرسال Telegram أيضًا:

    - `--buttons` للوحات المفاتيح المضمنة عندما يسمح `channels.telegram.capabilities.inlineButtons` بذلك
    - `--force-document` لإرسال الصور وملفات GIF الصادرة كمستندات بدلًا من تحميلات الصور المضغوطة أو الوسائط المتحركة

    تقييد الإجراءات:

    - يعطّل `channels.telegram.actions.sendMessage=false` رسائل Telegram الصادرة، بما في ذلك الاستطلاعات
    - يعطّل `channels.telegram.actions.poll=false` إنشاء استطلاعات Telegram مع إبقاء الإرسالات العادية مفعلة

  </Accordion>

  <Accordion title="موافقات exec في Telegram">
    يدعم Telegram موافقات exec في الرسائل الخاصة للموافِقين، ويمكنه اختياريًا نشر مطالبات الموافقة في الدردشة أو الموضوع الأصلي.

    مسار التكوين:

    - `channels.telegram.execApprovals.enabled`
    - `channels.telegram.execApprovals.approvers` (اختياري؛ ويرجع إلى معرّفات المالك الرقمية المستنتجة من `allowFrom` و`defaultTo` المباشر عند الإمكان)
    - `channels.telegram.execApprovals.target` (`dm` | `channel` | `both`، الافتراضي: `dm`)
    - `agentFilter` و`sessionFilter`

    يجب أن يكون الموافقون معرّفات مستخدمي Telegram رقمية. يفعّل Telegram موافقات exec الأصلية تلقائيًا عندما يكون `enabled` غير مضبوط أو `"auto"` ويمكن حل موافق واحد على الأقل، إما من `execApprovals.approvers` أو من تكوين المالك الرقمي للحساب (`allowFrom` و`defaultTo` للرسائل الخاصة المباشرة). اضبط `enabled: false` لتعطيل Telegram كعميل موافقات أصلي بشكل صريح. وإلا فستعود طلبات الموافقة إلى مسارات الموافقة الأخرى المكوّنة أو إلى سياسة الرجوع لموافقات exec.

    يعرض Telegram أيضًا أزرار الموافقة المشتركة المستخدمة بواسطة قنوات الدردشة الأخرى. ويضيف المهايئ الأصلي لـ Telegram أساسًا توجيه الرسائل الخاصة للموافقين، والتوزيع على القنوات/الموضوعات، وتلميحات الكتابة قبل التسليم.
    عندما تكون هذه الأزرار موجودة، فهي واجهة الموافقة الأساسية؛ ويجب على OpenClaw
    أن يضمّن أمر `/approve` اليدوي فقط عندما تقول نتيجة الأداة إن موافقات الدردشة غير متاحة
    أو عندما تكون الموافقة اليدوية هي المسار الوحيد.

    قواعد التسليم:

    - يرسل `target: "dm"` مطالبات الموافقة فقط إلى الرسائل الخاصة للموافقين الذين تم حلهم
    - يرسل `target: "channel"` المطالبة مرة أخرى إلى دردشة/موضوع Telegram الأصلي
    - يرسل `target: "both"` إلى الرسائل الخاصة للموافقين وإلى الدردشة/الموضوع الأصلي

    لا يمكن الموافقة أو الرفض إلا للموافقين الذين تم حلهم. لا يمكن لغير الموافقين استخدام `/approve` ولا أزرار الموافقة في Telegram.

    سلوك حل الموافقة:

    - تُحل دائمًا معرّفات الموافقة المسبوقة بـ `plugin:` عبر موافقات plugin.
    - تحاول معرّفات الموافقة الأخرى أولًا `exec.approval.resolve`.
    - إذا كان Telegram مصرحًا له أيضًا بموافقات plugin وقالت البوابة
      إن موافقة exec غير معروفة/منتهية الصلاحية، يعيد Telegram المحاولة مرة واحدة عبر
      `plugin.approval.resolve`.
    - لا تسقط حالات الرفض/الأخطاء الحقيقية الخاصة بـ exec بصمت إلى حل
      موافقات plugin.

    يُظهر التسليم إلى القناة نص الأمر في الدردشة، لذا فعّل `channel` أو `both` فقط في المجموعات/الموضوعات الموثوقة. عندما تصل المطالبة إلى موضوع منتدى، يحافظ OpenClaw على الموضوع لكل من مطالبة الموافقة والمتابعة بعد الموافقة. وتنتهي صلاحية موافقات exec بعد 30 دقيقة افتراضيًا.

    تعتمد أزرار الموافقة المضمنة أيضًا على سماح `channels.telegram.capabilities.inlineButtons` بالسطح المستهدف (`dm` أو `group` أو `all`).

    مستندات ذات صلة: [موافقات Exec](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## عناصر التحكم في ردود الأخطاء

عندما يواجه الوكيل خطأ في التسليم أو خطأ من المزوّد، يمكن لـ Telegram إما الرد بنص الخطأ أو كتمه. يتحكم مفتاحا تكوين في هذا السلوك:

| المفتاح                             | القيم             | الافتراضي | الوصف                                                                                     |
| ----------------------------------- | ----------------- | --------- | ----------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy`     | `reply`, `silent` | `reply`   | يرسل `reply` رسالة خطأ ودية إلى الدردشة. ويكتم `silent` ردود الأخطاء بالكامل. |
| `channels.telegram.errorCooldownMs` | number (ms)       | `60000`   | الحد الأدنى للوقت بين ردود الأخطاء على الدردشة نفسها. يمنع سيلان الأخطاء أثناء الانقطاعات.        |

تتوفر تجاوزات لكل حساب ولكل مجموعة ولكل موضوع (بنفس الوراثة الخاصة بمفاتيح تكوين Telegram الأخرى).

```json5
{
  channels: {
    telegram: {
      errorPolicy: "reply",
      errorCooldownMs: 120000,
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // كتم الأخطاء في هذه المجموعة
        },
      },
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لا يستجيب البوت لرسائل المجموعات التي لا تحتوي على إشارة">

    - إذا كان `requireMention=false`، فيجب أن يسمح وضع الخصوصية في Telegram بالرؤية الكاملة.
      - BotFather: `/setprivacy` -> Disable
      - ثم أزل البوت وأعد إضافته إلى المجموعة
    - يحذر `openclaw channels status` عندما يتوقع التكوين رسائل مجموعات من دون إشارة.
    - يستطيع `openclaw channels status --probe` فحص معرّفات المجموعات الرقمية الصريحة؛ أما `"*"` فلا يمكن فحص العضوية لها.
    - اختبار جلسة سريع: `/activation always`.

  </Accordion>

  <Accordion title="البوت لا يرى رسائل المجموعة إطلاقًا">

    - عندما تكون `channels.telegram.groups` موجودة، يجب أن تكون المجموعة مدرجة (أو أن تتضمن `"*"`)
    - تحقق من عضوية البوت في المجموعة
    - راجع السجلات: `openclaw logs --follow` لمعرفة أسباب التخطي

  </Accordion>

  <Accordion title="تعمل الأوامر جزئيًا أو لا تعمل إطلاقًا">

    - صرّح هوية المرسل الخاص بك (pairing و/أو `allowFrom` رقمية)
    - ما زال تفويض الأوامر يطبّق حتى عندما تكون سياسة المجموعة `open`
    - يعني `setMyCommands failed` مع `BOT_COMMANDS_TOO_MUCH` أن القائمة الأصلية تحتوي على عدد كبير جدًا من الإدخالات؛ قلّل أوامر plugin/Skills/الأوامر المخصصة أو عطّل القوائم الأصلية
    - يشير `setMyCommands failed` مع أخطاء الشبكة/fetch عادةً إلى مشكلات في الوصول إلى DNS/HTTPS الخاص بـ `api.telegram.org`

  </Accordion>

  <Accordion title="عدم استقرار polling أو الشبكة">

    - يمكن أن يؤدي Node 22+ مع fetch/proxy مخصص إلى سلوك إجهاض فوري إذا لم تتطابق أنواع AbortSignal.
    - تحل بعض المضيفات `api.telegram.org` إلى IPv6 أولًا؛ وقد يؤدي خروج IPv6 المعطوب إلى أعطال متقطعة في Telegram API.
    - إذا تضمنت السجلات `TypeError: fetch failed` أو `Network request for 'getUpdates' failed!`، فإن OpenClaw يعيد الآن محاولة هذه الحالات كأخطاء شبكة قابلة للاسترداد.
    - على مضيفات VPS ذات الخروج/TLS المباشر غير المستقر، وجّه استدعاءات Telegram API عبر `channels.telegram.proxy`:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - يستخدم Node 22+ افتراضيًا `autoSelectFamily=true` (باستثناء WSL2) و`dnsResultOrder=ipv4first`.
    - إذا كان مضيفك WSL2 أو يعمل صراحةً بشكل أفضل مع سلوك IPv4 فقط، فافرض اختيار family:

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - تُسمح بالفعل افتراضيًا إجابات نطاق القياس RFC 2544 (`198.18.0.0/15`)
      لتنزيلات وسائط Telegram. وإذا كانت fake-IP موثوقة أو
      transparent proxy تعيد كتابة `api.telegram.org` إلى عنوان آخر
      خاص/داخلي/ذو استخدام خاص أثناء تنزيلات الوسائط، فيمكنك تفعيل
      تجاوز Telegram فقط:

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - يتوفر التفعيل نفسه لكل حساب في
      `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork`.
    - إذا كان proxy لديك يحل مضيفات وسائط Telegram إلى `198.18.x.x`، فاترك
      الخيار الخطير معطلًا أولًا. إذ تسمح وسائط Telegram بالفعل افتراضيًا
      بنطاق القياس RFC 2544.

    <Warning>
      يضعف `channels.telegram.network.dangerouslyAllowPrivateNetwork` وسائل حماية
      Telegram media SSRF. استخدمه فقط في بيئات proxy موثوقة
      يتحكم فيها المشغل، مثل توجيه fake-IP في Clash أو Mihomo أو Surge عندما
      تولّد هذه الأدوات إجابات خاصة أو ذات استخدام خاص خارج
      نطاق القياس RFC 2544 الافتراضي. اتركه معطلًا عند الوصول العادي العام إلى Telegram.
    </Warning>

    - تجاوزات البيئة (مؤقتة):
      - `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`
    - تحقّق من إجابات DNS:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

مزيد من المساعدة: [استكشاف أخطاء القنوات وإصلاحها](/channels/troubleshooting).

## مؤشرات مرجع تكوين Telegram

المرجع الأساسي:

- `channels.telegram.enabled`: تمكين/تعطيل بدء تشغيل القناة.
- `channels.telegram.botToken`: bot token (BotFather).
- `channels.telegram.tokenFile`: قراءة token من مسار ملف عادي. تُرفض الروابط الرمزية.
- `channels.telegram.dmPolicy`: ‏`pairing | allowlist | open | disabled` (الافتراضي: pairing).
- `channels.telegram.allowFrom`: allowlist الرسائل الخاصة (معرّفات مستخدمي Telegram الرقمية). تتطلب `allowlist` معرّف مرسل واحدًا على الأقل. وتتطلب `open` القيمة `"*"`. يمكن لـ `openclaw doctor --fix` تحويل إدخالات `@username` القديمة إلى معرّفات، كما يمكنه استعادة إدخالات allowlist من ملفات pairing-store في تدفقات ترحيل allowlist.
- `channels.telegram.actions.poll`: تمكين أو تعطيل إنشاء استطلاعات Telegram (مفعّل افتراضيًا؛ وما زال يتطلب `sendMessage`).
- `channels.telegram.defaultTo`: هدف Telegram الافتراضي الذي يستخدمه CLI مع `--deliver` عندما لا يكون `--reply-to` صريحًا.
- `channels.telegram.groupPolicy`: ‏`open | allowlist | disabled` (الافتراضي: allowlist).
- `channels.telegram.groupAllowFrom`: allowlist مرسلي المجموعات (معرّفات مستخدمي Telegram الرقمية). يمكن لـ `openclaw doctor --fix` تحويل إدخالات `@username` القديمة إلى معرّفات. تُتجاهل الإدخالات غير الرقمية وقت التفويض. لا يستخدم تفويض المجموعات الرجوع إلى DM pairing-store (`2026.2.25+`).
- أولوية الحسابات المتعددة:
  - عندما يتم تكوين معرّفي حسابين أو أكثر، اضبط `channels.telegram.defaultAccount` (أو ضمّن `channels.telegram.accounts.default`) لجعل التوجيه الافتراضي صريحًا.
  - إذا لم يُضبط أي منهما، يعود OpenClaw إلى أول معرّف حساب مُوحَّد ويصدر `openclaw doctor` تحذيرًا.
  - ينطبق `channels.telegram.accounts.default.allowFrom` و`channels.telegram.accounts.default.groupAllowFrom` على حساب `default` فقط.
  - ترث الحسابات المسماة `channels.telegram.allowFrom` و`channels.telegram.groupAllowFrom` عندما تكون قيم مستوى الحساب غير مضبوطة.
  - لا ترث الحسابات المسماة `channels.telegram.accounts.default.allowFrom` / `groupAllowFrom`.
- `channels.telegram.groups`: الإعدادات الافتراضية لكل مجموعة + allowlist (استخدم `"*"` للإعدادات العامة الافتراضية).
  - `channels.telegram.groups.<id>.groupPolicy`: تجاوز لكل مجموعة لـ groupPolicy (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.requireMention`: تقييد الإشارات الافتراضي.
  - `channels.telegram.groups.<id>.skills`: مرشح Skills (الحذف = كل Skills، والفارغ = لا شيء).
  - `channels.telegram.groups.<id>.allowFrom`: تجاوز allowlist مرسلين لكل مجموعة.
  - `channels.telegram.groups.<id>.systemPrompt`: system prompt إضافي للمجموعة.
  - `channels.telegram.groups.<id>.enabled`: تعطيل المجموعة عندما تكون `false`.
  - `channels.telegram.groups.<id>.topics.<threadId>.*`: تجاوزات لكل موضوع (حقول المجموعة + `agentId` الخاص بالموضوع فقط).
  - `channels.telegram.groups.<id>.topics.<threadId>.agentId`: توجيه هذا الموضوع إلى وكيل محدد (يتجاوز توجيه مستوى المجموعة وbindings).
- `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`: تجاوز لكل موضوع لـ groupPolicy (`open | allowlist | disabled`).
- `channels.telegram.groups.<id>.topics.<threadId>.requireMention`: تجاوز تقييد الإشارات لكل موضوع.
- `bindings[]` على المستوى الأعلى مع `type: "acp"` ومعرّف الموضوع القياسي `chatId:topic:topicId` في `match.peer.id`: حقول ربط ACP الدائم للموضوع (راجع [وكلاء ACP](/tools/acp-agents#channel-specific-settings)).
- `channels.telegram.direct.<id>.topics.<threadId>.agentId`: توجيه موضوعات الرسائل الخاصة إلى وكيل محدد (السلوك نفسه مثل موضوعات المنتدى).
- `channels.telegram.execApprovals.enabled`: تمكين Telegram كعميل موافقات exec قائم على الدردشة لهذا الحساب.
- `channels.telegram.execApprovals.approvers`: معرّفات مستخدمي Telegram المسموح لهم بالموافقة على طلبات exec أو رفضها. اختياري عندما يعرّف `channels.telegram.allowFrom` أو `channels.telegram.defaultTo` المباشر المالك مسبقًا.
- `channels.telegram.execApprovals.target`: ‏`dm | channel | both` (الافتراضي: `dm`). يحافظ `channel` و`both` على موضوع Telegram الأصلي عند وجوده.
- `channels.telegram.execApprovals.agentFilter`: مرشح اختياري لمعرّف الوكيل لمطالبات الموافقة المعاد توجيهها.
- `channels.telegram.execApprovals.sessionFilter`: مرشح اختياري لمفتاح الجلسة (سلسلة فرعية أو regex) لمطالبات الموافقة المعاد توجيهها.
- `channels.telegram.accounts.<account>.execApprovals`: تجاوز لكل حساب لتوجيه موافقات exec وتفويض الموافقين في Telegram.
- `channels.telegram.capabilities.inlineButtons`: ‏`off | dm | group | all | allowlist` (الافتراضي: allowlist).
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`: تجاوز لكل حساب.
- `channels.telegram.commands.nativeSkills`: تمكين/تعطيل أوامر Skills الأصلية في Telegram.
- `channels.telegram.replyToMode`: ‏`off | first | all` (الافتراضي: `off`).
- `channels.telegram.textChunkLimit`: حجم الأجزاء الصادرة (أحرف).
- `channels.telegram.chunkMode`: ‏`length` (الافتراضي) أو `newline` للتقسيم على الأسطر الفارغة (حدود الفقرات) قبل التقسيم بحسب الطول.
- `channels.telegram.linkPreview`: تبديل معاينات الروابط للرسائل الصادرة (الافتراضي: true).
- `channels.telegram.streaming`: ‏`off | partial | block | progress` (معاينة البث المباشر؛ الافتراضي: `partial`؛ تتحول `progress` إلى `partial`؛ و`block` للتوافق مع وضع المعاينة القديم). يستخدم بث المعاينة في Telegram رسالة معاينة واحدة يتم تعديلها في مكانها.
- `channels.telegram.mediaMaxMb`: الحد الأقصى لوسائط Telegram الواردة/الصادرة (ميغابايت، الافتراضي: 100).
- `channels.telegram.retry`: سياسة إعادة المحاولة لمساعدات الإرسال في Telegram (CLI/tools/actions) عند أخطاء API الصادرة القابلة للاسترداد (المحاولات وminDelayMs وmaxDelayMs وjitter).
- `channels.telegram.network.autoSelectFamily`: تجاوز autoSelectFamily في Node (true=تمكين، false=تعطيل). يكون مفعّلًا افتراضيًا على Node 22+، مع تعطيل افتراضي في WSL2.
- `channels.telegram.network.dnsResultOrder`: تجاوز ترتيب نتائج DNS (`ipv4first` أو `verbatim`). يكون `ipv4first` افتراضيًا على Node 22+.
- `channels.telegram.network.dangerouslyAllowPrivateNetwork`: تفعيل خطير لبيئات fake-IP أو transparent-proxy الموثوقة حيث تُحل تنزيلات وسائط Telegram لـ `api.telegram.org` إلى عناوين خاصة/داخلية/ذات استخدام خاص خارج السماح الافتراضي لنطاق RFC 2544.
- `channels.telegram.proxy`: URL للـ proxy لاستدعاءات Bot API (SOCKS/HTTP).
- `channels.telegram.webhookUrl`: تمكين وضع webhook (يتطلب `channels.telegram.webhookSecret`).
- `channels.telegram.webhookSecret`: سر webhook (مطلوب عند ضبط webhookUrl).
- `channels.telegram.webhookPath`: مسار webhook المحلي (الافتراضي `/telegram-webhook`).
- `channels.telegram.webhookHost`: مضيف ربط webhook المحلي (الافتراضي `127.0.0.1`).
- `channels.telegram.webhookPort`: منفذ ربط webhook المحلي (الافتراضي `8787`).
- `channels.telegram.actions.reactions`: تقييد تفاعلات أداة Telegram.
- `channels.telegram.actions.sendMessage`: تقييد إرسال رسائل أداة Telegram.
- `channels.telegram.actions.deleteMessage`: تقييد حذف رسائل أداة Telegram.
- `channels.telegram.actions.sticker`: تقييد إجراءات ملصقات Telegram — الإرسال والبحث (الافتراضي: false).
- `channels.telegram.reactionNotifications`: ‏`off | own | all` — التحكم في التفاعلات التي تطلق أحداث النظام (الافتراضي: `own` عندما لا تُضبط).
- `channels.telegram.reactionLevel`: ‏`off | ack | minimal | extensive` — التحكم في قدرة الوكيل على التفاعل (الافتراضي: `minimal` عندما لا تُضبط).
- `channels.telegram.errorPolicy`: ‏`reply | silent` — التحكم في سلوك ردود الأخطاء (الافتراضي: `reply`). وتُدعم تجاوزات لكل حساب/مجموعة/موضوع.
- `channels.telegram.errorCooldownMs`: الحد الأدنى بالميلي ثانية بين ردود الأخطاء للدردشة نفسها (الافتراضي: `60000`). يمنع سيلان الأخطاء أثناء الانقطاعات.

- [مرجع التكوين - Telegram](/gateway/configuration-reference#telegram)

حقول Telegram عالية الأهمية:

- بدء التشغيل/المصادقة: `enabled` و`botToken` و`tokenFile` و`accounts.*` (`tokenFile` يجب أن يشير إلى ملف عادي؛ وتُرفض الروابط الرمزية)
- التحكم في الوصول: `dmPolicy` و`allowFrom` و`groupPolicy` و`groupAllowFrom` و`groups` و`groups.*.topics.*` و`bindings[]` على المستوى الأعلى (`type: "acp"`)
- موافقات exec: ‏`execApprovals` و`accounts.*.execApprovals`
- الأوامر/القائمة: `commands.native` و`commands.nativeSkills` و`customCommands`
- threading/الردود: `replyToMode`
- البث: `streaming` (المعاينة) و`blockStreaming`
- التنسيق/التسليم: `textChunkLimit` و`chunkMode` و`linkPreview` و`responsePrefix`
- الوسائط/الشبكة: `mediaMaxMb` و`timeoutSeconds` و`retry` و`network.autoSelectFamily` و`network.dangerouslyAllowPrivateNetwork` و`proxy`
- webhook: ‏`webhookUrl` و`webhookSecret` و`webhookPath` و`webhookHost`
- الإجراءات/القدرات: `capabilities.inlineButtons` و`actions.sendMessage|editMessage|deleteMessage|reactions|sticker`
- التفاعلات: `reactionNotifications` و`reactionLevel`
- الأخطاء: `errorPolicy` و`errorCooldownMs`
- الكتابات/السجل: `configWrites` و`historyLimit` و`dmHistoryLimit` و`dms.*.historyLimit`

## ذو صلة

- [الإقران](/channels/pairing)
- [المجموعات](/channels/groups)
- [الأمان](/gateway/security)
- [توجيه القنوات](/channels/channel-routing)
- [التوجيه متعدد الوكلاء](/concepts/multi-agent)
- [استكشاف الأخطاء وإصلاحها](/channels/troubleshooting)
