---
read_when:
    - إعداد دعم iMessage
    - تصحيح مشكلات الإرسال/الاستقبال في iMessage
summary: دعم iMessage القديم عبر imsg ‏(JSON-RPC عبر stdio). يجب أن تستخدم الإعدادات الجديدة BlueBubbles.
title: iMessage
x-i18n:
    generated_at: "2026-04-05T12:35:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 086d85bead49f75d12ae6b14ac917af52375b6afd28f6af1a0dcbbc7fcb628a0
    source_path: channels/imessage.md
    workflow: 15
---

# iMessage (قديم: imsg)

<Warning>
بالنسبة إلى عمليات نشر iMessage الجديدة، استخدم <a href="/channels/bluebubbles">BlueBubbles</a>.

تكامل `imsg` قديم وقد تتم إزالته في إصدار مستقبلي.
</Warning>

الحالة: تكامل CLI خارجي قديم. يقوم Gateway بتشغيل `imsg rpc` والتواصل عبر JSON-RPC على stdio (من دون daemon/port منفصل).

<CardGroup cols={3}>
  <Card title="BlueBubbles (موصى به)" icon="message-circle" href="/channels/bluebubbles">
    مسار iMessage المفضل للإعدادات الجديدة.
  </Card>
  <Card title="الاقتران" icon="link" href="/channels/pairing">
    تستخدم الرسائل المباشرة في iMessage وضع الاقتران افتراضيًا.
  </Card>
  <Card title="مرجع الإعدادات" icon="settings" href="/gateway/configuration-reference#imessage">
    مرجع كامل لحقول iMessage.
  </Card>
</CardGroup>

## إعداد سريع

<Tabs>
  <Tab title="Mac محلي (المسار السريع)">
    <Steps>
      <Step title="تثبيت imsg والتحقق منه">

```bash
brew install steipete/tap/imsg
imsg rpc --help
```

      </Step>

      <Step title="إعداد OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/<you>/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="بدء gateway">

```bash
openclaw gateway
```

      </Step>

      <Step title="الموافقة على أول اقتران رسالة مباشرة (سياسة dmPolicy الافتراضية)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        تنتهي صلاحية طلبات الاقتران بعد ساعة واحدة.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Mac بعيد عبر SSH">
    يحتاج OpenClaw فقط إلى `cliPath` متوافق مع stdio، لذلك يمكنك توجيه `cliPath` إلى نص wrapper برمجي يستخدم SSH إلى Mac بعيد ويشغّل `imsg`.

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

    الإعداد الموصى به عند تمكين المرفقات:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // يُستخدم لجلب المرفقات عبر SCP
      includeAttachments: true,
      // اختياري: تجاوز الجذور المسموح بها للمرفقات.
      // تتضمن القيم الافتراضية /Users/*/Library/Messages/Attachments
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    إذا لم يتم ضبط `remoteHost`، يحاول OpenClaw اكتشافه تلقائيًا من خلال تحليل نص wrapper البرمجي لـ SSH.
    يجب أن يكون `remoteHost` بصيغة `host` أو `user@host` (من دون مسافات أو خيارات SSH).
    يستخدم OpenClaw تحققًا صارمًا من مفتاح المضيف لـ SCP، لذلك يجب أن يكون مفتاح مضيف relay موجودًا مسبقًا في `~/.ssh/known_hosts`.
    يتم التحقق من مسارات المرفقات مقابل الجذور المسموح بها (`attachmentRoots` / `remoteAttachmentRoots`).

  </Tab>
</Tabs>

## المتطلبات والأذونات (macOS)

- يجب أن يكون Messages مسجّل الدخول على Mac الذي يشغّل `imsg`.
- يلزم Full Disk Access لسياق العملية التي تشغّل OpenClaw/`imsg` (للوصول إلى قاعدة بيانات Messages).
- يلزم إذن Automation لإرسال الرسائل عبر Messages.app.

<Tip>
يتم منح الأذونات لكل سياق عملية. إذا كان gateway يعمل بلا واجهة (LaunchAgent/SSH)، فشغّل أمرًا تفاعليًا لمرة واحدة في السياق نفسه لتفعيل مطالبات الأذونات:

```bash
imsg chats --limit 1
# or
imsg send <handle> "test"
```

</Tip>

## التحكم في الوصول والتوجيه

<Tabs>
  <Tab title="سياسة الرسائل المباشرة">
    يتحكم `channels.imessage.dmPolicy` في الرسائل المباشرة:

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن تتضمن `allowFrom` القيمة `"*"`)
    - `disabled`

    حقل قائمة السماح: `channels.imessage.allowFrom`.

    يمكن أن تكون إدخالات قائمة السماح handles أو أهداف دردشة (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`).

  </Tab>

  <Tab title="سياسة المجموعات + الإشارات">
    يتحكم `channels.imessage.groupPolicy` في التعامل مع المجموعات:

    - `allowlist` (الافتراضي عند الإعداد)
    - `open`
    - `disabled`

    قائمة السماح لمرسلي المجموعات: `channels.imessage.groupAllowFrom`.

    التراجع في وقت التشغيل: إذا لم يتم ضبط `groupAllowFrom`، فإن فحوصات مرسلي مجموعات iMessage تعود إلى `allowFrom` عند توفره.
    ملاحظة وقت التشغيل: إذا كان `channels.imessage` مفقودًا بالكامل، يعود وقت التشغيل إلى `groupPolicy="allowlist"` ويسجل تحذيرًا (حتى إذا كان `channels.defaults.groupPolicy` مضبوطًا).

    بوابة الإشارات للمجموعات:

    - لا يحتوي iMessage على بيانات تعريف أصلية للإشارات
    - يستخدم اكتشاف الإشارات أنماط regex ‏(`agents.list[].groupChat.mentionPatterns`، مع تراجع إلى `messages.groupChat.mentionPatterns`)
    - من دون أنماط مهيأة، لا يمكن فرض بوابة الإشارات

    يمكن لأوامر التحكم الواردة من مرسلين مخولين تجاوز بوابة الإشارات في المجموعات.

  </Tab>

  <Tab title="الجلسات والردود الحتمية">
    - تستخدم الرسائل المباشرة التوجيه المباشر؛ وتستخدم المجموعات توجيه المجموعات.
    - مع الإعداد الافتراضي `session.dmScope=main`، تندمج الرسائل المباشرة في iMessage ضمن الجلسة الرئيسية للوكيل.
    - تكون جلسات المجموعات معزولة (`agent:<agentId>:imessage:group:<chat_id>`).
    - تُوجَّه الردود مرة أخرى إلى iMessage باستخدام بيانات تعريف القناة/الهدف الأصلية.

    سلوك السلاسل الشبيهة بالمجموعات:

    قد تصل بعض سلاسل iMessage متعددة المشاركين مع `is_group=false`.
    إذا كان هذا `chat_id` مهيأً صراحةً ضمن `channels.imessage.groups`، فإن OpenClaw يعامله كحركة مرور مجموعة (بوابة مجموعات + عزل جلسة المجموعة).

  </Tab>
</Tabs>

## ارتباطات محادثات ACP

يمكن أيضًا ربط محادثات iMessage القديمة بجلسات ACP.

تدفق المشغل السريع:

- شغّل `/acp spawn codex --bind here` داخل الرسالة المباشرة أو دردشة المجموعة المسموح بها.
- تُوجَّه الرسائل المستقبلية في محادثة iMessage نفسها إلى جلسة ACP التي تم إنشاؤها.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` جلسة ACP ويزيل الارتباط.

يتم دعم الارتباطات الدائمة المهيأة عبر إدخالات `bindings[]` من المستوى الأعلى مع `type: "acp"` و`match.channel: "imessage"`.

يمكن أن يستخدم `match.peer.id`:

- handle رسالة مباشرة مطبّعًا مثل `+15555550123` أو `user@example.com`
- `chat_id:<id>` (موصى به للارتباطات المستقرة للمجموعات)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

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
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

راجع [وكلاء ACP](/tools/acp-agents) لمعرفة سلوك ارتباط ACP المشترك.

## أنماط النشر

<AccordionGroup>
  <Accordion title="مستخدم macOS مخصص للبوت (هوية iMessage منفصلة)">
    استخدم Apple ID ومستخدم macOS مخصصين بحيث تكون حركة مرور البوت معزولة عن ملف Messages الشخصي الخاص بك.

    التدفق المعتاد:

    1. أنشئ/سجّل الدخول إلى مستخدم macOS مخصص.
    2. سجّل الدخول إلى Messages باستخدام Apple ID الخاص بالبوت في ذلك المستخدم.
    3. ثبّت `imsg` لذلك المستخدم.
    4. أنشئ wrapper لـ SSH حتى يتمكن OpenClaw من تشغيل `imsg` في سياق ذلك المستخدم.
    5. وجّه `channels.imessage.accounts.<id>.cliPath` و`.dbPath` إلى ملف ذلك المستخدم.

    قد يتطلب التشغيل الأول موافقات GUI ‏(Automation + Full Disk Access) في جلسة مستخدم البوت تلك.

  </Accordion>

  <Accordion title="Mac بعيد عبر Tailscale (مثال)">
    بنية شائعة:

    - يعمل gateway على Linux/VM
    - يعمل iMessage و`imsg` على Mac داخل tailnet لديك
    - يستخدم wrapper الخاص بـ `cliPath` ‏SSH لتشغيل `imsg`
    - يتيح `remoteHost` جلب المرفقات عبر SCP

    مثال:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
      includeAttachments: true,
      dbPath: "/Users/bot/Library/Messages/chat.db",
    },
  },
}
```

```bash
#!/usr/bin/env bash
exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
```

    استخدم مفاتيح SSH بحيث يكون كل من SSH وSCP غير تفاعليين.
    تأكد أولًا من الوثوق بمفتاح المضيف (على سبيل المثال `ssh bot@mac-mini.tailnet-1234.ts.net`) حتى تتم تعبئة `known_hosts`.

  </Accordion>

  <Accordion title="نمط الحسابات المتعددة">
    يدعم iMessage إعدادات لكل حساب ضمن `channels.imessage.accounts`.

    يمكن لكل حساب تجاوز حقول مثل `cliPath` و`dbPath` و`allowFrom` و`groupPolicy` و`mediaMaxMb` وإعدادات السجل وقوائم السماح لجذور المرفقات.

  </Accordion>
</AccordionGroup>

## الوسائط والتقسيم وأهداف التسليم

<AccordionGroup>
  <Accordion title="المرفقات والوسائط">
    - إدخال المرفقات الواردة اختياري: `channels.imessage.includeAttachments`
    - يمكن جلب مسارات المرفقات البعيدة عبر SCP عند ضبط `remoteHost`
    - يجب أن تطابق مسارات المرفقات الجذور المسموح بها:
      - `channels.imessage.attachmentRoots` (محلي)
      - `channels.imessage.remoteAttachmentRoots` (وضع SCP البعيد)
      - نمط الجذر الافتراضي: `/Users/*/Library/Messages/Attachments`
    - يستخدم SCP تحققًا صارمًا من مفتاح المضيف (`StrictHostKeyChecking=yes`)
    - يستخدم حجم الوسائط الصادرة `channels.imessage.mediaMaxMb` (الافتراضي 16 MB)
  </Accordion>

  <Accordion title="تقسيم الرسائل الصادرة">
    - حد تقسيم النص: `channels.imessage.textChunkLimit` (الافتراضي 4000)
    - وضع التقسيم: `channels.imessage.chunkMode`
      - `length` (الافتراضي)
      - `newline` (التقسيم حسب الفقرة أولًا)
  </Accordion>

  <Accordion title="تنسيقات العنونة">
    الأهداف الصريحة المفضلة:

    - `chat_id:123` (موصى به للتوجيه المستقر)
    - `chat_guid:...`
    - `chat_identifier:...`

    أهداف handle مدعومة أيضًا:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

```bash
imsg chats --limit 20
```

  </Accordion>
</AccordionGroup>

## عمليات كتابة الإعدادات

يسمح iMessage افتراضيًا بعمليات كتابة الإعدادات التي تبدأها القناة (لأوامر `/config set|unset` عندما تكون `commands.config: true`).

للتعطيل:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لم يتم العثور على imsg أو أن RPC غير مدعوم">
    تحقّق من الملف التنفيذي ودعم RPC:

```bash
imsg rpc --help
openclaw channels status --probe
```

    إذا أبلغ الفحص أن RPC غير مدعوم، فقم بتحديث `imsg`.

  </Accordion>

  <Accordion title="يتم تجاهل الرسائل المباشرة">
    تحقّق من:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - موافقات الاقتران (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="يتم تجاهل رسائل المجموعات">
    تحقّق من:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - سلوك قائمة السماح لـ `channels.imessage.groups`
    - إعداد أنماط الإشارة (`agents.list[].groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="فشل المرفقات البعيدة">
    تحقّق من:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - مصادقة مفاتيح SSH/SCP من مضيف gateway
    - وجود مفتاح المضيف في `~/.ssh/known_hosts` على مضيف gateway
    - قابلية قراءة المسار البعيد على Mac الذي يشغّل Messages

  </Accordion>

  <Accordion title="تم تفويت مطالبات أذونات macOS">
    أعد التشغيل في طرفية GUI تفاعلية ضمن سياق المستخدم/الجلسة نفسه ووافق على المطالبات:

```bash
imsg chats --limit 1
imsg send <handle> "test"
```

    أكّد منح Full Disk Access وAutomation لسياق العملية الذي يشغّل OpenClaw/`imsg`.

  </Accordion>
</AccordionGroup>

## مؤشرات إلى مرجع الإعدادات

- [مرجع الإعدادات - iMessage](/gateway/configuration-reference#imessage)
- [إعدادات Gateway](/gateway/configuration)
- [الاقتران](/channels/pairing)
- [BlueBubbles](/channels/bluebubbles)

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك دردشات المجموعات وبوابة الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
