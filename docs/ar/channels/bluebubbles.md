---
read_when:
    - إعداد قناة BlueBubbles
    - استكشاف أخطاء اقتران webhook وإصلاحها
    - تكوين iMessage على macOS
summary: iMessage عبر خادم BlueBubbles على macOS ‏(إرسال/استقبال REST، الكتابة، التفاعلات، الاقتران، الإجراءات المتقدمة).
title: BlueBubbles
x-i18n:
    generated_at: "2026-04-05T12:35:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: ed8e59a165bdfb8fd794ee2ad6e4dacd44aa02d512312c5f2fd7d15f863380bb
    source_path: channels/bluebubbles.md
    workflow: 15
---

# BlueBubbles (REST على macOS)

الحالة: plugin مضمّن يتصل بخادم BlueBubbles على macOS عبر HTTP. **موصى به لتكامل iMessage** بسبب واجهة API الأغنى وسهولة الإعداد مقارنة بقناة imsg القديمة.

## plugin المضمّن

تتضمن إصدارات OpenClaw الحالية BlueBubbles، لذلك لا تحتاج الإصدارات المعبأة المعتادة إلى خطوة `openclaw plugins install` منفصلة.

## نظرة عامة

- يعمل على macOS عبر تطبيق BlueBubbles المساعد ([bluebubbles.app](https://bluebubbles.app)).
- الموصى به/المختبر: macOS Sequoia ‏(15). يعمل macOS Tahoe ‏(26)، لكن التحرير معطل حاليًا على Tahoe، وقد تُبلغ تحديثات أيقونات المجموعات عن النجاح لكنها لا تتزامن.
- يتصل OpenClaw به عبر واجهة REST API الخاصة به (`GET /api/v1/ping` و`POST /message/text` و`POST /chat/:id/*`).
- تصل الرسائل الواردة عبر webhooks؛ أما الردود الصادرة ومؤشرات الكتابة وإيصالات القراءة وtapbacks فهي استدعاءات REST.
- يتم استيعاب المرفقات والملصقات كوسائط واردة (وتُعرَض للوكيل عندما يكون ذلك ممكنًا).
- يعمل الاقتران/قائمة السماح بالطريقة نفسها كما في القنوات الأخرى (`/channels/pairing` وما إلى ذلك) باستخدام `channels.bluebubbles.allowFrom` + رموز الاقتران.
- تُعرَض التفاعلات كأحداث نظام تمامًا مثل Slack/Telegram بحيث يمكن للوكلاء "الإشارة" إليها قبل الرد.
- الميزات المتقدمة: التحرير، وإلغاء الإرسال، وسلاسل الردود، وتأثيرات الرسائل، وإدارة المجموعات.

## البدء السريع

1. ثبّت خادم BlueBubbles على جهاز Mac لديك (اتبع التعليمات على [bluebubbles.app/install](https://bluebubbles.app/install)).
2. في إعدادات BlueBubbles، فعّل web API وحدد كلمة مرور.
3. شغّل `openclaw onboard` وحدد BlueBubbles، أو قم بالتكوين يدويًا:

   ```json5
   {
     channels: {
       bluebubbles: {
         enabled: true,
         serverUrl: "http://192.168.1.100:1234",
         password: "example-password",
         webhookPath: "/bluebubbles-webhook",
       },
     },
   }
   ```

4. وجّه webhooks الخاصة بـ BlueBubbles إلى gateway لديك (مثال: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`).
5. ابدأ تشغيل gateway؛ سيقوم بتسجيل معالج webhook وبدء الاقتران.

ملاحظة أمنية:

- احرص دائمًا على تعيين كلمة مرور لـ webhook.
- تكون مصادقة webhook مطلوبة دائمًا. يرفض OpenClaw طلبات BlueBubbles webhook ما لم تتضمن كلمة مرور/guid يطابق `channels.bluebubbles.password` (مثل `?password=<password>` أو `x-password`) بغض النظر عن بنية loopback/proxy.
- يتم التحقق من مصادقة كلمة المرور قبل قراءة/تحليل أجسام webhook الكاملة.

## إبقاء Messages.app نشطًا (إعدادات VM / بدون واجهة)

قد تنتهي بعض إعدادات macOS VM / التشغيل الدائم إلى أن يصبح Messages.app "خاملًا" (تتوقف الأحداث الواردة حتى يتم فتح التطبيق/إحضاره إلى المقدمة). أحد الحلول البسيطة هو **تحفيز Messages كل 5 دقائق** باستخدام AppleScript + LaunchAgent.

### 1) احفظ AppleScript

احفظ هذا باسم:

- `~/Scripts/poke-messages.scpt`

مثال على script ‏(غير تفاعلي؛ لا يسرق التركيز):

```applescript
try
  tell application "Messages"
    if not running then
      launch
    end if

    -- Touch the scripting interface to keep the process responsive.
    set _chatCount to (count of chats)
  end tell
on error
  -- Ignore transient failures (first-run prompts, locked session, etc).
end try
```

### 2) ثبّت LaunchAgent

احفظ هذا باسم:

- `~/Library/LaunchAgents/com.user.poke-messages.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.user.poke-messages</string>

    <key>ProgramArguments</key>
    <array>
      <string>/bin/bash</string>
      <string>-lc</string>
      <string>/usr/bin/osascript &quot;$HOME/Scripts/poke-messages.scpt&quot;</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>StartInterval</key>
    <integer>300</integer>

    <key>StandardOutPath</key>
    <string>/tmp/poke-messages.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/poke-messages.err</string>
  </dict>
</plist>
```

ملاحظات:

- يعمل هذا **كل 300 ثانية** و**عند تسجيل الدخول**.
- قد يؤدي التشغيل الأول إلى ظهور مطالبات macOS **Automation** ‏(`osascript` → Messages). وافق عليها في جلسة المستخدم نفسها التي تشغّل LaunchAgent.

قم بتحميله:

```bash
launchctl unload ~/Library/LaunchAgents/com.user.poke-messages.plist 2>/dev/null || true
launchctl load ~/Library/LaunchAgents/com.user.poke-messages.plist
```

## الإعداد التفاعلي

يتوفر BlueBubbles في الإعداد التفاعلي:

```
openclaw onboard
```

يطلب المعالج ما يلي:

- **Server URL** ‏(مطلوب): عنوان خادم BlueBubbles ‏(مثلًا، `http://192.168.1.100:1234`)
- **Password** ‏(مطلوب): كلمة مرور API من إعدادات BlueBubbles Server
- **Webhook path** ‏(اختياري): القيمة الافتراضية هي `/bluebubbles-webhook`
- **DM policy**: ‏pairing أو allowlist أو open أو disabled
- **Allow list**: أرقام الهواتف أو عناوين البريد الإلكتروني أو أهداف الدردشة

يمكنك أيضًا إضافة BlueBubbles عبر CLI:

```
openclaw channels add bluebubbles --http-url http://192.168.1.100:1234 --password <password>
```

## التحكم في الوصول (الرسائل الخاصة + المجموعات)

الرسائل الخاصة:

- الافتراضي: `channels.bluebubbles.dmPolicy = "pairing"`.
- يتلقى المرسلون غير المعروفين رمز اقتران؛ ويتم تجاهل الرسائل حتى الموافقة عليها (تنتهي صلاحية الرموز بعد ساعة واحدة).
- وافق عبر:
  - `openclaw pairing list bluebubbles`
  - `openclaw pairing approve bluebubbles <CODE>`
- الاقتران هو تبادل الرموز الافتراضي. التفاصيل: [الاقتران](/channels/pairing)

المجموعات:

- `channels.bluebubbles.groupPolicy = open | allowlist | disabled` ‏(الافتراضي: `allowlist`).
- يتحكم `channels.bluebubbles.groupAllowFrom` في من يمكنه التفعيل في المجموعات عندما يتم تعيين `allowlist`.

### إثراء أسماء جهات الاتصال (macOS، اختياري)

غالبًا ما تتضمن webhooks الخاصة بمجموعات BlueBubbles عناوين المشاركين الخام فقط. إذا كنت تريد أن يُظهر سياق `GroupMembers` أسماء جهات الاتصال المحلية بدلًا من ذلك، فيمكنك الاشتراك في إثراء Contacts المحلي على macOS:

- يؤدي `channels.bluebubbles.enrichGroupParticipantsFromContacts = true` إلى تمكين البحث. الافتراضي: `false`.
- لا تعمل عمليات البحث إلا بعد أن يسمح وصول المجموعة وتفويض الأوامر وضبط الإشارات بمرور الرسالة.
- يتم إثراء المشاركين الهاتفيين غير المسمّين فقط.
- تبقى أرقام الهواتف الخام هي البديل عند عدم العثور على تطابق محلي.

```json5
{
  channels: {
    bluebubbles: {
      enrichGroupParticipantsFromContacts: true,
    },
  },
}
```

### ضبط الإشارات (المجموعات)

يدعم BlueBubbles ضبط الإشارات للمحادثات الجماعية، بما يتطابق مع سلوك iMessage/WhatsApp:

- يستخدم `agents.list[].groupChat.mentionPatterns` (أو `messages.groupChat.mentionPatterns`) لاكتشاف الإشارات.
- عند تمكين `requireMention` لمجموعة، لا يرد الوكيل إلا عند الإشارة إليه.
- تتجاوز أوامر التحكم من المرسلين المصرح لهم ضبط الإشارات.

تكوين لكل مجموعة:

```json5
{
  channels: {
    bluebubbles: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true }, // الافتراضي لجميع المجموعات
        "iMessage;-;chat123": { requireMention: false }, // تجاوز لمجموعة محددة
      },
    },
  },
}
```

### ضبط الأوامر

- تتطلب أوامر التحكم (مثل `/config` و`/model`) تفويضًا.
- يستخدم `allowFrom` و`groupAllowFrom` لتحديد تفويض الأوامر.
- يمكن للمرسلين المصرح لهم تشغيل أوامر التحكم حتى بدون إشارة في المجموعات.

## ارتباطات محادثات ACP

يمكن تحويل محادثات BlueBubbles إلى مساحات عمل ACP دائمة من دون تغيير طبقة النقل.

تدفق المشغل السريع:

- شغّل `/acp spawn codex --bind here` داخل الرسالة الخاصة أو الدردشة الجماعية المسموح بها.
- تُوجَّه الرسائل المستقبلية في محادثة BlueBubbles نفسها إلى جلسة ACP المنشأة.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الارتباط.

يتم أيضًا دعم الارتباطات الدائمة المكوّنة من خلال إدخالات `bindings[]` على المستوى الأعلى مع `type: "acp"` و`match.channel: "bluebubbles"`.

يمكن أن يستخدم `match.peer.id` أي صيغة هدف BlueBubbles مدعومة:

- مقبض DM مُطبَّع مثل `+15555550123` أو `user@example.com`
- `chat_id:<id>`
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

للحصول على ارتباطات مجموعات مستقرة، استخدم `chat_id:*` أو `chat_identifier:*` ويفضل ذلك.

مثال:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "bluebubbles",
        accountId: "default",
        peer: { kind: "dm", id: "+15555550123" },
      },
      acp: { label: "codex-imessage" },
    },
  ],
}
```

راجع [وكلاء ACP](/tools/acp-agents) للاطلاع على سلوك ارتباط ACP المشترك.

## الكتابة + إيصالات القراءة

- **مؤشرات الكتابة**: يتم إرسالها تلقائيًا قبل وأثناء توليد الرد.
- **إيصالات القراءة**: يتحكم فيها `channels.bluebubbles.sendReadReceipts` ‏(الافتراضي: `true`).
- **مؤشرات الكتابة**: يرسل OpenClaw أحداث بدء الكتابة؛ ويقوم BlueBubbles بمسح حالة الكتابة تلقائيًا عند الإرسال أو انتهاء المهلة (الإيقاف اليدوي عبر DELETE غير موثوق).

```json5
{
  channels: {
    bluebubbles: {
      sendReadReceipts: false, // تعطيل إيصالات القراءة
    },
  },
}
```

## الإجراءات المتقدمة

يدعم BlueBubbles إجراءات الرسائل المتقدمة عند تمكينها في التكوين:

```json5
{
  channels: {
    bluebubbles: {
      actions: {
        reactions: true, // tapbacks (الافتراضي: true)
        edit: true, // تحرير الرسائل المرسلة (macOS 13+، معطل على macOS 26 Tahoe)
        unsend: true, // إلغاء إرسال الرسائل (macOS 13+)
        reply: true, // سلاسل الردود حسب GUID الرسالة
        sendWithEffect: true, // تأثيرات الرسائل (slam وloud وما إلى ذلك)
        renameGroup: true, // إعادة تسمية الدردشات الجماعية
        setGroupIcon: true, // تعيين أيقونة/صورة الدردشة الجماعية (غير مستقر على macOS 26 Tahoe)
        addParticipant: true, // إضافة مشاركين إلى المجموعات
        removeParticipant: true, // إزالة مشاركين من المجموعات
        leaveGroup: true, // مغادرة الدردشات الجماعية
        sendAttachment: true, // إرسال المرفقات/الوسائط
      },
    },
  },
}
```

الإجراءات المتاحة:

- **react**: إضافة/إزالة تفاعلات tapback ‏(`messageId` و`emoji` و`remove`)
- **edit**: تحرير رسالة مرسلة (`messageId` و`text`)
- **unsend**: إلغاء إرسال رسالة (`messageId`)
- **reply**: الرد على رسالة محددة (`messageId` و`text` و`to`)
- **sendWithEffect**: الإرسال مع تأثير iMessage ‏(`text` و`to` و`effectId`)
- **renameGroup**: إعادة تسمية دردشة جماعية (`chatGuid` و`displayName`)
- **setGroupIcon**: تعيين أيقونة/صورة دردشة جماعية (`chatGuid` و`media`) — غير مستقر على macOS 26 Tahoe ‏(قد تُرجع API نجاحًا لكن الأيقونة لا تتزامن).
- **addParticipant**: إضافة شخص إلى مجموعة (`chatGuid` و`address`)
- **removeParticipant**: إزالة شخص من مجموعة (`chatGuid` و`address`)
- **leaveGroup**: مغادرة دردشة جماعية (`chatGuid`)
- **upload-file**: إرسال وسائط/ملفات (`to` و`buffer` و`filename` و`asVoice`)
  - المذكرات الصوتية: اضبط `asVoice: true` مع صوت **MP3** أو **CAF** للإرسال كرسالة صوتية في iMessage. يقوم BlueBubbles بتحويل MP3 → CAF عند إرسال المذكرات الصوتية.
- الاسم المستعار القديم: ما زال `sendAttachment` يعمل، لكن `upload-file` هو اسم الإجراء المعتمد.

### معرفات الرسائل (القصيرة مقابل الكاملة)

قد يعرض OpenClaw معرفات رسائل _قصيرة_ (مثل `1` و`2`) لتوفير الرموز.

- يمكن أن تكون `MessageSid` / `ReplyToId` معرفات قصيرة.
- تحتوي `MessageSidFull` / `ReplyToIdFull` على المعرفات الكاملة للمزوّد.
- المعرفات القصيرة موجودة في الذاكرة؛ وقد تنتهي صلاحيتها عند إعادة التشغيل أو إخلاء ذاكرة التخزين المؤقت.
- تقبل الإجراءات `messageId` القصير أو الكامل، لكن المعرفات القصيرة ستؤدي إلى خطأ إذا لم تعد متاحة.

استخدم المعرفات الكاملة للأتمتة والتخزين الدائمين:

- القوالب: `{{MessageSidFull}}` و`{{ReplyToIdFull}}`
- السياق: `MessageSidFull` / `ReplyToIdFull` في الحمولات الواردة

راجع [التكوين](/gateway/configuration) للاطلاع على متغيرات القوالب.

## تدفق الكتل

تحكم فيما إذا كانت الردود تُرسل كرسالة واحدة أو تُبث على شكل كتل:

```json5
{
  channels: {
    bluebubbles: {
      blockStreaming: true, // تمكين تدفق الكتل (معطل افتراضيًا)
    },
  },
}
```

## الوسائط + الحدود

- يتم تنزيل المرفقات الواردة وتخزينها في ذاكرة التخزين المؤقت للوسائط.
- حد الوسائط عبر `channels.bluebubbles.mediaMaxMb` للوسائط الواردة والصادرة (الافتراضي: 8 ميجابايت).
- يتم تجزئة النص الصادر وفق `channels.bluebubbles.textChunkLimit` ‏(الافتراضي: 4000 حرف).

## مرجع التكوين

التكوين الكامل: [التكوين](/gateway/configuration)

خيارات المزوّد:

- `channels.bluebubbles.enabled`: تمكين/تعطيل القناة.
- `channels.bluebubbles.serverUrl`: عنوان URL الأساسي لـ BlueBubbles REST API.
- `channels.bluebubbles.password`: كلمة مرور API.
- `channels.bluebubbles.webhookPath`: مسار نقطة نهاية webhook ‏(الافتراضي: `/bluebubbles-webhook`).
- `channels.bluebubbles.dmPolicy`: ‏`pairing | allowlist | open | disabled` ‏(الافتراضي: `pairing`).
- `channels.bluebubbles.allowFrom`: قائمة السماح للرسائل الخاصة (المقابض، والبريد الإلكتروني، وأرقام E.164، و`chat_id:*`، و`chat_guid:*`).
- `channels.bluebubbles.groupPolicy`: ‏`open | allowlist | disabled` ‏(الافتراضي: `allowlist`).
- `channels.bluebubbles.groupAllowFrom`: قائمة السماح لمرسلي المجموعات.
- `channels.bluebubbles.enrichGroupParticipantsFromContacts`: على macOS، إثراء المشاركين غير المسمّين في المجموعة اختياريًا من Contacts المحلية بعد اجتياز الضبط. الافتراضي: `false`.
- `channels.bluebubbles.groups`: تكوين لكل مجموعة (`requireMention` وما إلى ذلك).
- `channels.bluebubbles.sendReadReceipts`: إرسال إيصالات القراءة (الافتراضي: `true`).
- `channels.bluebubbles.blockStreaming`: تمكين تدفق الكتل (الافتراضي: `false`؛ مطلوب للردود المتدفقة).
- `channels.bluebubbles.textChunkLimit`: حجم التجزئة الصادرة بالأحرف (الافتراضي: 4000).
- `channels.bluebubbles.chunkMode`: يؤدي `length` ‏(الافتراضي) إلى التقسيم فقط عند تجاوز `textChunkLimit`؛ بينما يقسم `newline` عند الأسطر الفارغة (حدود الفقرات) قبل التجزئة حسب الطول.
- `channels.bluebubbles.mediaMaxMb`: حد الوسائط الواردة/الصادرة بالميجابايت (الافتراضي: 8).
- `channels.bluebubbles.mediaLocalRoots`: قائمة سماح صريحة بالأدلة المحلية المطلقة المسموح بها لمسارات الوسائط المحلية الصادرة. يتم رفض الإرسال عبر المسارات المحلية افتراضيًا ما لم يتم تكوين هذا الخيار. تجاوز لكل حساب: `channels.bluebubbles.accounts.<accountId>.mediaLocalRoots`.
- `channels.bluebubbles.historyLimit`: الحد الأقصى لرسائل المجموعة للسياق (يعطل عند 0).
- `channels.bluebubbles.dmHistoryLimit`: حد سجل الرسائل الخاصة.
- `channels.bluebubbles.actions`: تمكين/تعطيل إجراءات محددة.
- `channels.bluebubbles.accounts`: تكوين متعدد الحسابات.

الخيارات العامة ذات الصلة:

- `agents.list[].groupChat.mentionPatterns` (أو `messages.groupChat.mentionPatterns`).
- `messages.responsePrefix`.

## الاستهداف / أهداف التسليم

يفضل استخدام `chat_guid` للتوجيه المستقر:

- `chat_guid:iMessage;-;+15555550123` ‏(مفضل للمجموعات)
- `chat_id:123`
- `chat_identifier:...`
- المقابض المباشرة: `+15555550123` و`user@example.com`
  - إذا لم يكن للمقبض المباشر دردشة DM موجودة، فسيقوم OpenClaw بإنشاء واحدة عبر `POST /api/v1/chat/new`. وهذا يتطلب تمكين BlueBubbles Private API.

## الأمان

- تتم مصادقة طلبات webhook عبر مقارنة معاملات الاستعلام `guid`/`password` أو الرؤوس مع `channels.bluebubbles.password`.
- حافظ على سرية كلمة مرور API ونقطة نهاية webhook (تعامل معها على أنها بيانات اعتماد).
- لا يوجد تجاوز localhost لمصادقة BlueBubbles webhook. إذا كنت تمرر حركة webhook عبر proxy، فاحتفظ بكلمة مرور BlueBubbles داخل الطلب من الطرف إلى الطرف. لا يستبدل `gateway.trustedProxies` قيمة `channels.bluebubbles.password` هنا. راجع [أمان Gateway](/gateway/security#reverse-proxy-configuration).
- فعّل HTTPS + قواعد الجدار الناري على خادم BlueBubbles إذا كنت ستعرضه خارج شبكتك المحلية.

## استكشاف الأخطاء وإصلاحها

- إذا توقفت أحداث الكتابة/القراءة عن العمل، فتحقق من سجلات BlueBubbles webhook وتأكد من أن مسار gateway يطابق `channels.bluebubbles.webhookPath`.
- تنتهي صلاحية رموز الاقتران بعد ساعة واحدة؛ استخدم `openclaw pairing list bluebubbles` و`openclaw pairing approve bluebubbles <code>`.
- تتطلب التفاعلات BlueBubbles private API ‏(`POST /api/v1/message/react`)؛ تأكد من أن إصدار الخادم يوفّرها.
- يتطلب التحرير/إلغاء الإرسال macOS 13+ وإصدار BlueBubbles Server متوافقًا. في macOS 26 ‏(Tahoe)، يكون التحرير معطلًا حاليًا بسبب تغييرات في private API.
- قد تكون تحديثات أيقونات المجموعات غير مستقرة على macOS 26 ‏(Tahoe): قد تُرجع API نجاحًا لكن الأيقونة الجديدة لا تتزامن.
- يُخفي OpenClaw تلقائيًا الإجراءات المعروفة بأنها معطلة بناءً على إصدار macOS الخاص بخادم BlueBubbles. إذا استمر ظهور التحرير على macOS 26 ‏(Tahoe)، فعطّله يدويًا باستخدام `channels.bluebubbles.actions.edit=false`.
- لمعلومات الحالة/السلامة: `openclaw status --all` أو `openclaw status --deep`.

للاطلاع على مرجع عام لتدفق القنوات، راجع [القنوات](/channels) ودليل [Plugins](/tools/plugin).

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل الخاصة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وضبط الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
