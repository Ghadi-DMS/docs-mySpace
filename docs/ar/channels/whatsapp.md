---
read_when:
    - العمل على سلوك قناة WhatsApp/web أو توجيه inbox
summary: دعم قناة WhatsApp، وعناصر التحكم في الوصول، وسلوك التسليم، والعمليات
title: WhatsApp
x-i18n:
    generated_at: "2026-04-05T12:37:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: c16a468b3f47fdf7e4fc3fd745b5c49c7ccebb7af0e8c87c632b78b04c583e49
    source_path: channels/whatsapp.md
    workflow: 15
---

# WhatsApp (قناة Web)

الحالة: جاهزة للإنتاج عبر WhatsApp Web ‏(Baileys). يمتلك gateway الجلسة (الجلسات) المرتبطة.

## التثبيت (عند الطلب)

- يطلب كل من الإعداد التفاعلي (`openclaw onboard`) و`openclaw channels add --channel whatsapp`
  تثبيت plugin WhatsApp في أول مرة تحددها فيها.
- يوفّر `openclaw channels login --channel whatsapp` أيضًا تدفق التثبيت عندما
  لا يكون plugin موجودًا بعد.
- قناة التطوير + نسخة git محلية: تكون القيمة الافتراضية هي مسار plugin المحلي.
- Stable/Beta: القيمة الافتراضية هي حزمة npm ‏`@openclaw/whatsapp`.

يبقى التثبيت اليدوي متاحًا:

```bash
openclaw plugins install @openclaw/whatsapp
```

<CardGroup cols={3}>
  <Card title="الاقتران" icon="link" href="/channels/pairing">
    سياسة الرسائل الخاصة الافتراضية هي الاقتران للمرسلين غير المعروفين.
  </Card>
  <Card title="استكشاف أخطاء القنوات وإصلاحها" icon="wrench" href="/channels/troubleshooting">
    أدوات التشخيص وخطط الإصلاح عبر القنوات.
  </Card>
  <Card title="تكوين Gateway" icon="settings" href="/gateway/configuration">
    أنماط وأمثلة تكوين القنوات الكاملة.
  </Card>
</CardGroup>

## الإعداد السريع

<Steps>
  <Step title="تكوين سياسة الوصول إلى WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="ربط WhatsApp ‏(QR)">

```bash
openclaw channels login --channel whatsapp
```

    لحساب محدد:

```bash
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="بدء تشغيل gateway">

```bash
openclaw gateway
```

  </Step>

  <Step title="الموافقة على أول طلب اقتران (إذا كنت تستخدم وضع الاقتران)">

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    تنتهي صلاحية طلبات الاقتران بعد ساعة واحدة. الحد الأقصى للطلبات المعلقة هو 3 لكل قناة.

  </Step>
</Steps>

<Note>
يوصي OpenClaw بتشغيل WhatsApp على رقم منفصل كلما أمكن ذلك. (تم تحسين بيانات تعريف القناة وتدفق الإعداد لهذا الإعداد، لكن الإعدادات التي تستخدم الرقم الشخصي مدعومة أيضًا.)
</Note>

## أنماط النشر

<AccordionGroup>
  <Accordion title="رقم مخصص (موصى به)">
    هذا هو وضع التشغيل الأنظف:

    - هوية WhatsApp منفصلة لـ OpenClaw
    - حدود أوضح لقوائم السماح بالرسائل الخاصة والتوجيه
    - احتمال أقل لحدوث ارتباك في الدردشة الذاتية

    نمط السياسة الأدنى:

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="بديل الرقم الشخصي">
    يدعم الإعداد التفاعلي وضع الرقم الشخصي ويكتب خط أساس مناسبًا للدردشة الذاتية:

    - `dmPolicy: "allowlist"`
    - تتضمن `allowFrom` رقمك الشخصي
    - `selfChatMode: true`

    في وقت التشغيل، تعتمد وسائل الحماية من الدردشة الذاتية على الرقم الذاتي المرتبط و`allowFrom`.

  </Accordion>

  <Accordion title="نطاق قناة WhatsApp Web فقط">
    قناة منصة المراسلة في بنية قنوات OpenClaw الحالية تستند إلى WhatsApp Web ‏(`Baileys`).

    لا توجد قناة مراسلة WhatsApp منفصلة عبر Twilio ضمن سجل قنوات الدردشة المدمج.

  </Accordion>
</AccordionGroup>

## نموذج وقت التشغيل

- يمتلك gateway مقبس WhatsApp وحلقة إعادة الاتصال.
- تتطلب عمليات الإرسال الصادرة وجود مستمع WhatsApp نشط للحساب الهدف.
- يتم تجاهل محادثات الحالة والبث (`@status` و`@broadcast`).
- تستخدم المحادثات المباشرة قواعد جلسات الرسائل الخاصة (`session.dmScope`؛ القيمة الافتراضية `main` تؤدي إلى طي الرسائل الخاصة ضمن الجلسة الرئيسية للوكيل).
- تكون جلسات المجموعات معزولة (`agent:<agentId>:whatsapp:group:<jid>`).

## التحكم في الوصول والتفعيل

<Tabs>
  <Tab title="سياسة الرسائل الخاصة">
    يتحكم `channels.whatsapp.dmPolicy` في الوصول إلى الدردشة المباشرة:

    - `pairing` (الافتراضي)
    - `allowlist`
    - `open` (يتطلب أن تتضمن `allowFrom` القيمة `"*"`)
    - `disabled`

    تقبل `allowFrom` أرقامًا بنمط E.164 (ويتم تطبيعها داخليًا).

    تجاوز الحسابات المتعددة: تكون `channels.whatsapp.accounts.<id>.dmPolicy` (و`allowFrom`) لها الأولوية على القيم الافتراضية على مستوى القناة لذلك الحساب.

    تفاصيل سلوك وقت التشغيل:

    - يتم الاحتفاظ بعمليات الاقتران في مخزن السماح الخاص بالقناة ودمجها مع `allowFrom` المكوّنة
    - إذا لم يتم تكوين أي قائمة سماح، يُسمح بالرقم الذاتي المرتبط افتراضيًا
    - لا يتم أبدًا إقران الرسائل الخاصة الصادرة `fromMe` تلقائيًا

  </Tab>

  <Tab title="سياسة المجموعات + قوائم السماح">
    يتكون الوصول إلى المجموعات من طبقتين:

    1. **قائمة سماح عضوية المجموعة** (`channels.whatsapp.groups`)
       - إذا تم حذف `groups`، تكون كل المجموعات مؤهلة
       - إذا وُجدت `groups`، فإنها تعمل كقائمة سماح للمجموعات (مع السماح بـ `"*"`)

    2. **سياسة مرسل المجموعة** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`)
       - `open`: يتم تجاوز قائمة سماح المرسلين
       - `allowlist`: يجب أن يطابق المرسل `groupAllowFrom` (أو `*`)
       - `disabled`: حظر كل الرسائل الواردة من المجموعات

    بديل قائمة سماح المرسلين:

    - إذا لم يتم تعيين `groupAllowFrom`، يعود وقت التشغيل إلى `allowFrom` عند توفرها
    - يتم تقييم قوائم سماح المرسلين قبل تفعيل الإشارة/الرد

    ملاحظة: إذا لم توجد كتلة `channels.whatsapp` إطلاقًا، فإن بديل سياسة المجموعات في وقت التشغيل يكون `allowlist` (مع سجل تحذير)، حتى لو كانت `channels.defaults.groupPolicy` معيّنة.

  </Tab>

  <Tab title="الإشارات + /activation">
    تتطلب الردود في المجموعات الإشارة افتراضيًا.

    يتضمن اكتشاف الإشارة:

    - إشارات WhatsApp الصريحة لهوية bot
    - أنماط regex المكوّنة للإشارة (`agents.list[].groupChat.mentionPatterns`، والبديل `messages.groupChat.mentionPatterns`)
    - اكتشاف الرد الضمني على bot ‏(يطابق مرسل الرد هوية bot)

    ملاحظة أمنية:

    - الاقتباس/الرد يفي فقط بضبط الإشارة؛ ولا يمنح تفويض المرسل
    - مع `groupPolicy: "allowlist"`، يظل المرسلون غير المدرجين في قائمة السماح محظورين حتى إذا ردوا على رسالة من مستخدم مدرج في قائمة السماح

    أمر التفعيل على مستوى الجلسة:

    - `/activation mention`
    - `/activation always`

    يقوم `activation` بتحديث حالة الجلسة (وليس التكوين العام). وهو مقيّد بالمالك.

  </Tab>
</Tabs>

## سلوك الرقم الشخصي والدردشة الذاتية

عندما يكون الرقم الذاتي المرتبط موجودًا أيضًا في `allowFrom`، يتم تفعيل ضمانات الدردشة الذاتية في WhatsApp:

- تخطي إيصالات القراءة في أدوار الدردشة الذاتية
- تجاهل سلوك التشغيل التلقائي لإشارة JID الذي قد يؤدي بخلاف ذلك إلى تنبيهك لنفسك
- إذا لم تكن `messages.responsePrefix` معيّنة، فإن ردود الدردشة الذاتية تستخدم افتراضيًا `[{identity.name}]` أو `[openclaw]`

## تطبيع الرسائل والسياق

<AccordionGroup>
  <Accordion title="الغلاف الوارد + سياق الرد">
    تُغلَّف رسائل WhatsApp الواردة في الغلاف الوارد المشترك.

    إذا وُجد رد مقتبس، يُضاف السياق بهذا الشكل:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    كما تتم تعبئة حقول بيانات تعريف الرد عند توفرها (`ReplyToId` و`ReplyToBody` و`ReplyToSender` وJID/E.164 الخاص بالمرسل).

  </Accordion>

  <Accordion title="عناصر الوسائط النائبة واستخراج الموقع/جهة الاتصال">
    يتم تطبيع الرسائل الواردة التي تحتوي على وسائط فقط باستخدام عناصر نائبة مثل:

    - `<media:image>`
    - `<media:video>`
    - `<media:audio>`
    - `<media:document>`
    - `<media:sticker>`

    يتم تطبيع حمولات الموقع وجهات الاتصال إلى سياق نصي قبل التوجيه.

  </Accordion>

  <Accordion title="حقن سجل المجموعات المعلق">
    بالنسبة إلى المجموعات، يمكن تخزين الرسائل غير المعالجة مؤقتًا وحقنها كسياق عندما يتم تشغيل bot أخيرًا.

    - الحد الافتراضي: `50`
    - التكوين: `channels.whatsapp.historyLimit`
    - البديل: `messages.groupChat.historyLimit`
    - `0` يعطّل الميزة

    علامات الحقن:

    - `[Chat messages since your last reply - for context]`
    - `[Current message - respond to this]`

  </Accordion>

  <Accordion title="إيصالات القراءة">
    تكون إيصالات القراءة ممكّنة افتراضيًا لرسائل WhatsApp الواردة المقبولة.

    التعطيل عالميًا:

    ```json5
    {
      channels: {
        whatsapp: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    تجاوز لكل حساب:

    ```json5
    {
      channels: {
        whatsapp: {
          accounts: {
            work: {
              sendReadReceipts: false,
            },
          },
        },
      },
    }
    ```

    تتخطى أدوار الدردشة الذاتية إيصالات القراءة حتى عند تمكينها عالميًا.

  </Accordion>
</AccordionGroup>

## التسليم والتجزئة والوسائط

<AccordionGroup>
  <Accordion title="تجزئة النص">
    - حد التجزئة الافتراضي: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.chunkMode = "length" | "newline"`
    - يفضل وضع `newline` حدود الفقرات (الأسطر الفارغة)، ثم يعود إلى التجزئة الآمنة حسب الطول
  </Accordion>

  <Accordion title="سلوك الوسائط الصادرة">
    - يدعم حمولات الصور والفيديو والصوت (ملاحظة صوتية PTT) والمستندات
    - تتم إعادة كتابة `audio/ogg` إلى `audio/ogg; codecs=opus` للتوافق مع الملاحظات الصوتية
    - يتم دعم تشغيل GIF المتحرك عبر `gifPlayback: true` عند إرسال الفيديو
    - تُطبَّق التسميات التوضيحية على أول عنصر وسائط عند إرسال حمولات رد متعددة الوسائط
    - يمكن أن يكون مصدر الوسائط هو HTTP(S) أو `file://` أو مسارات محلية
  </Accordion>

  <Accordion title="حدود حجم الوسائط وسلوك البديل">
    - حد حفظ الوسائط الواردة: `channels.whatsapp.mediaMaxMb` (الافتراضي `50`)
    - حد إرسال الوسائط الصادرة: `channels.whatsapp.mediaMaxMb` (الافتراضي `50`)
    - تستخدم تجاوزات كل حساب `channels.whatsapp.accounts.<accountId>.mediaMaxMb`
    - يتم تحسين الصور تلقائيًا (تغيير الحجم/اجتياز الجودة) لتلائم الحدود
    - عند فشل إرسال الوسائط، يرسل البديل الخاص بأول عنصر تحذيرًا نصيًا بدلًا من إسقاط الرد بصمت
  </Accordion>
</AccordionGroup>

## مستوى التفاعل

يتحكم `channels.whatsapp.reactionLevel` في مدى استخدام الوكيل لتفاعلات emoji على WhatsApp:

| المستوى       | تفاعلات الإقرار | التفاعلات التي يبدأها الوكيل | الوصف                                            |
| ------------- | --------------- | ---------------------------- | ------------------------------------------------ |
| `"off"`       | لا              | لا                           | لا توجد تفاعلات إطلاقًا                          |
| `"ack"`       | نعم             | لا                           | تفاعلات الإقرار فقط (إيصال ما قبل الرد)         |
| `"minimal"`   | نعم             | نعم (بشكل محافظ)            | تفاعلات الإقرار + تفاعلات الوكيل مع توجيه محافظ |
| `"extensive"` | نعم             | نعم (مشجعة)                 | تفاعلات الإقرار + تفاعلات الوكيل مع توجيه مشجع  |

الافتراضي: `"minimal"`.

تستخدم تجاوزات كل حساب `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{
  channels: {
    whatsapp: {
      reactionLevel: "ack",
    },
  },
}
```

## تفاعلات الإقرار

يدعم WhatsApp تفاعلات الإقرار الفورية عند الاستلام الوارد عبر `channels.whatsapp.ackReaction`.
تخضع تفاعلات الإقرار إلى `reactionLevel` — ويتم إخفاؤها عندما تكون قيمة `reactionLevel` هي `"off"`.

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

ملاحظات السلوك:

- تُرسَل فورًا بعد قبول الرسالة الواردة (قبل الرد)
- يتم تسجيل الإخفاقات لكنها لا تمنع تسليم الرد العادي
- في وضع المجموعة `mentions`، يتم التفاعل في الأدوار التي يتم تشغيلها بالإشارة؛ ويعمل تفعيل المجموعة `always` كتجاوز لهذا الفحص
- يستخدم WhatsApp الإعداد `channels.whatsapp.ackReaction` (ولا يُستخدم هنا الإعداد القديم `messages.ackReaction`)

## الحسابات المتعددة وبيانات الاعتماد

<AccordionGroup>
  <Accordion title="اختيار الحساب والقيم الافتراضية">
    - تأتي معرّفات الحسابات من `channels.whatsapp.accounts`
    - اختيار الحساب الافتراضي: `default` إذا كان موجودًا، وإلا أول معرّف حساب مكوَّن (مرتب)
    - يتم تطبيع معرّفات الحسابات داخليًا لأغراض البحث
  </Accordion>

  <Accordion title="مسارات بيانات الاعتماد والتوافق مع الأنظمة القديمة">
    - مسار المصادقة الحالي: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
    - ملف النسخة الاحتياطية: `creds.json.bak`
    - ما زالت مصادقة النظام القديم الافتراضية في `~/.openclaw/credentials/` معرّفًا بها/تُنقل لتدفقات الحساب الافتراضي
  </Accordion>

  <Accordion title="سلوك تسجيل الخروج">
    يؤدي `openclaw channels logout --channel whatsapp [--account <id>]` إلى مسح حالة مصادقة WhatsApp لذلك الحساب.

    في أدلة المصادقة القديمة، يتم الاحتفاظ بـ `oauth.json` بينما تتم إزالة ملفات مصادقة Baileys.

  </Accordion>
</AccordionGroup>

## الأدوات والإجراءات وعمليات كتابة التكوين

- يتضمن دعم أدوات الوكيل إجراء تفاعل WhatsApp ‏(`react`).
- بوابات الإجراءات:
  - `channels.whatsapp.actions.reactions`
  - `channels.whatsapp.actions.polls`
- تكون عمليات كتابة التكوين التي تبدأها القناة ممكّنة افتراضيًا (عطّلها عبر `channels.whatsapp.configWrites=false`).

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="غير مرتبط (يلزم QR)">
    العَرَض: تشير حالة القناة إلى أنها غير مرتبطة.

    الإصلاح:

    ```bash
    openclaw channels login --channel whatsapp
    openclaw channels status
    ```

  </Accordion>

  <Accordion title="مرتبط لكن غير متصل / حلقة إعادة اتصال">
    العَرَض: حساب مرتبط مع انقطاعات متكررة أو محاولات إعادة اتصال.

    الإصلاح:

    ```bash
    openclaw doctor
    openclaw logs --follow
    ```

    إذا لزم الأمر، أعد الربط باستخدام `channels login`.

  </Accordion>

  <Accordion title="لا يوجد مستمع نشط عند الإرسال">
    تفشل عمليات الإرسال الصادرة بسرعة عندما لا يوجد مستمع gateway نشط للحساب الهدف.

    تأكد من أن gateway يعمل وأن الحساب مرتبط.

  </Accordion>

  <Accordion title="يتم تجاهل رسائل المجموعات بشكل غير متوقع">
    تحقق بالترتيب التالي:

    - `groupPolicy`
    - `groupAllowFrom` / `allowFrom`
    - إدخالات قائمة سماح `groups`
    - ضبط الإشارة (`requireMention` + أنماط الإشارة)
    - المفاتيح المكررة في `openclaw.json` ‏(JSON5): تتجاوز الإدخالات اللاحقة الإدخالات السابقة، لذا احتفظ بقيمة `groupPolicy` واحدة لكل نطاق

  </Accordion>

  <Accordion title="تحذير وقت تشغيل Bun">
    يجب أن يستخدم وقت تشغيل WhatsApp gateway بيئة Node. تُصنَّف Bun على أنها غير متوافقة مع التشغيل المستقر لـ WhatsApp/Telegram gateway.
  </Accordion>
</AccordionGroup>

## مؤشرات مرجع التكوين

المرجع الأساسي:

- [مرجع التكوين - WhatsApp](/gateway/configuration-reference#whatsapp)

حقول WhatsApp عالية الأهمية:

- الوصول: `dmPolicy` و`allowFrom` و`groupPolicy` و`groupAllowFrom` و`groups`
- التسليم: `textChunkLimit` و`chunkMode` و`mediaMaxMb` و`sendReadReceipts` و`ackReaction` و`reactionLevel`
- الحسابات المتعددة: `accounts.<id>.enabled` و`accounts.<id>.authDir` والتجاوزات على مستوى الحساب
- العمليات: `configWrites` و`debounceMs` و`web.enabled` و`web.heartbeatSeconds` و`web.reconnect.*`
- سلوك الجلسة: `session.dmScope` و`historyLimit` و`dmHistoryLimit` و`dms.<id>.historyLimit`

## ذو صلة

- [الاقتران](/channels/pairing)
- [المجموعات](/channels/groups)
- [الأمان](/gateway/security)
- [توجيه القنوات](/channels/channel-routing)
- [التوجيه متعدد الوكلاء](/concepts/multi-agent)
- [استكشاف الأخطاء وإصلاحها](/channels/troubleshooting)
