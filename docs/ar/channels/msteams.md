---
read_when:
    - العمل على ميزات قناة Microsoft Teams
summary: حالة دعم بوت Microsoft Teams وإمكاناته وإعداده
title: Microsoft Teams
x-i18n:
    generated_at: "2026-04-05T12:37:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: 99fc6e136893ec65dc85d3bc0c0d92134069a2f3b8cb4fcf66c14674399b3eaf
    source_path: channels/msteams.md
    workflow: 15
---

# Microsoft Teams

> "اتركوا كل أمل، يا من تدخلون هنا."

تم التحديث: 2026-01-21

الحالة: النص + مرفقات الرسائل المباشرة مدعومان؛ ويتطلب إرسال الملفات في القنوات/المجموعات `sharePointSiteId` + أذونات Graph (راجع [إرسال الملفات في الدردشات الجماعية](#sending-files-in-group-chats)). تُرسل الاستطلاعات عبر Adaptive Cards. وتعرض إجراءات الرسائل الإجراء الصريح `upload-file` لعمليات الإرسال التي تبدأ بالملف.

## المكون الإضافي المضمّن

يأتي Microsoft Teams كمكون إضافي مضمّن في إصدارات OpenClaw الحالية، لذلك لا يلزم
أي تثبيت منفصل في البنية المعبأة العادية.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستبعد Teams المضمّن،
فقم بتثبيته يدويًا:

```bash
openclaw plugins install @openclaw/msteams
```

نسخة checkout محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## إعداد سريع (للمبتدئين)

1. تأكد من أن المكون الإضافي لـ Microsoft Teams متاح.
   - إصدارات OpenClaw المعبأة الحالية تتضمنه بالفعل.
   - يمكن للإصدارات/التثبيتات الأقدم أو المخصصة إضافته يدويًا بالأوامر أعلاه.
2. أنشئ **Azure Bot** ‏(App ID + client secret + tenant ID).
3. اضبط OpenClaw باستخدام بيانات الاعتماد هذه.
4. اكشف `/api/messages` ‏(المنفذ 3978 افتراضيًا) عبر URL عام أو نفق.
5. ثبّت حزمة تطبيق Teams وابدأ gateway.

الحد الأدنى من الإعداد:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

ملاحظة: تُحظر الدردشات الجماعية افتراضيًا (`channels.msteams.groupPolicy: "allowlist"`). للسماح بالردود في المجموعات، اضبط `channels.msteams.groupAllowFrom` (أو استخدم `groupPolicy: "open"` للسماح لأي عضو، مع تقييد الإشارات).

## الأهداف

- التحدث إلى OpenClaw عبر الرسائل المباشرة في Teams أو الدردشات الجماعية أو القنوات.
- إبقاء التوجيه حتميًا: تعود الردود دائمًا إلى القناة التي وصلت منها.
- الاعتماد افتراضيًا على سلوك آمن للقناة (الإشارات مطلوبة ما لم يتم تكوين خلاف ذلك).

## عمليات كتابة الإعداد

يُسمح افتراضيًا لـ Microsoft Teams بكتابة تحديثات الإعداد الناتجة عن `/config set|unset` (يتطلب `commands.config: true`).

عطّل ذلك باستخدام:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## التحكم في الوصول (الرسائل المباشرة + المجموعات)

**الوصول عبر الرسائل المباشرة**

- الافتراضي: `channels.msteams.dmPolicy = "pairing"`. يتم تجاهل المرسلين غير المعروفين حتى تتم الموافقة عليهم.
- يجب أن يستخدم `channels.msteams.allowFrom` معرّفات كائنات AAD الثابتة.
- أسماء UPN/العرض قابلة للتغيير؛ والمطابقة المباشرة معطلة افتراضيًا ولا تُفعّل إلا باستخدام `channels.msteams.dangerouslyAllowNameMatching: true`.
- يمكن للمعالج تحويل الأسماء إلى معرّفات عبر Microsoft Graph عندما تسمح بيانات الاعتماد بذلك.

**الوصول إلى المجموعات**

- الافتراضي: `channels.msteams.groupPolicy = "allowlist"` (محظور ما لم تضف `groupAllowFrom`). استخدم `channels.defaults.groupPolicy` لتجاوز القيمة الافتراضية عند عدم ضبطها.
- يتحكم `channels.msteams.groupAllowFrom` في المرسلين الذين يمكنهم التشغيل في الدردشات/القنوات الجماعية (ويعود إلى `channels.msteams.allowFrom` عند الحاجة).
- اضبط `groupPolicy: "open"` للسماح لأي عضو (مع بقاء تقييد الإشارات افتراضيًا).
- لعدم السماح **بأي قنوات**، اضبط `channels.msteams.groupPolicy: "disabled"`.

مثال:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["user@org.com"],
    },
  },
}
```

**Teams + قائمة سماح القنوات**

- حدّد نطاق الردود في المجموعات/القنوات عبر إدراج الفرق والقنوات ضمن `channels.msteams.teams`.
- يجب أن تستخدم المفاتيح معرّفات الفرق الثابتة ومعرّفات محادثات القنوات.
- عندما يكون `groupPolicy="allowlist"` وتوجد قائمة سماح للفرق، لا تُقبل إلا الفرق/القنوات المُدرجة (مع تقييد الإشارات).
- يقبل معالج الإعداد إدخالات `Team/Channel` ويخزنها لك.
- عند بدء التشغيل، يحل OpenClaw أسماء الفرق/القنوات وأسماء المستخدمين في قوائم السماح إلى معرّفات (عندما تسمح أذونات Graph بذلك)
  ويسجل هذا الربط؛ وتُحتفظ بأسماء الفرق/القنوات غير المحلولة كما أُدخلت ولكن يتم تجاهلها للتوجيه افتراضيًا ما لم يكن `channels.msteams.dangerouslyAllowNameMatching: true` مفعّلًا.

مثال:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

## كيف يعمل

1. تأكد من أن المكون الإضافي لـ Microsoft Teams متاح.
   - إصدارات OpenClaw المعبأة الحالية تتضمنه بالفعل.
   - يمكن للإصدارات/التثبيتات الأقدم أو المخصصة إضافته يدويًا بالأوامر أعلاه.
2. أنشئ **Azure Bot** ‏(App ID + secret + tenant ID).
3. أنشئ **حزمة تطبيق Teams** تشير إلى البوت وتتضمن أذونات RSC أدناه.
4. ارفع/ثبّت تطبيق Teams داخل فريق (أو نطاق شخصي للرسائل المباشرة).
5. اضبط `msteams` في `~/.openclaw/openclaw.json` (أو متغيرات البيئة) وابدأ gateway.
6. يستمع gateway إلى حركة webhook الخاصة بـ Bot Framework على `/api/messages` افتراضيًا.

## إعداد Azure Bot (المتطلبات المسبقة)

قبل إعداد OpenClaw، تحتاج إلى إنشاء مورد Azure Bot.

### الخطوة 1: إنشاء Azure Bot

1. انتقل إلى [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot)
2. املأ علامة التبويب **Basics**:

   | الحقل              | القيمة                                                    |
   | ------------------ | --------------------------------------------------------- |
   | **Bot handle**     | اسم البوت الخاص بك، مثل `openclaw-msteams` (يجب أن يكون فريدًا) |
   | **Subscription**   | اختر اشتراك Azure الخاص بك                                |
   | **Resource group** | أنشئ مجموعة جديدة أو استخدم مجموعة موجودة                |
   | **Pricing tier**   | **Free** للتطوير/الاختبار                                 |
   | **Type of App**    | **Single Tenant** (موصى به - راجع الملاحظة أدناه)         |
   | **Creation type**  | **Create new Microsoft App ID**                           |

> **إشعار إيقاف:** تم إيقاف إنشاء البوتات الجديدة متعددة المستأجرين بعد 2025-07-31. استخدم **Single Tenant** للبوتات الجديدة.

3. انقر **Review + create** → **Create** (انتظر نحو دقيقة إلى دقيقتين)

### الخطوة 2: الحصول على بيانات الاعتماد

1. انتقل إلى مورد Azure Bot الخاص بك → **Configuration**
2. انسخ **Microsoft App ID** → هذا هو `appId` الخاص بك
3. انقر **Manage Password** → انتقل إلى App Registration
4. ضمن **Certificates & secrets** → **New client secret** → انسخ **Value** → هذا هو `appPassword` الخاص بك
5. انتقل إلى **Overview** → انسخ **Directory (tenant) ID** → هذا هو `tenantId` الخاص بك

### الخطوة 3: إعداد نقطة نهاية المراسلة

1. في Azure Bot → **Configuration**
2. اضبط **Messaging endpoint** على webhook URL الخاص بك:
   - الإنتاج: `https://your-domain.com/api/messages`
   - التطوير المحلي: استخدم نفقًا (راجع [التطوير المحلي](#local-development-tunneling) أدناه)

### الخطوة 4: تفعيل قناة Teams

1. في Azure Bot → **Channels**
2. انقر **Microsoft Teams** → Configure → Save
3. اقبل شروط الخدمة

## التطوير المحلي (الأنفاق)

لا يستطيع Teams الوصول إلى `localhost`. استخدم نفقًا للتطوير المحلي:

**الخيار A: ngrok**

```bash
ngrok http 3978
# انسخ URL الذي يبدأ بـ https، مثل https://abc123.ngrok.io
# اضبط نقطة نهاية المراسلة على: https://abc123.ngrok.io/api/messages
```

**الخيار B: Tailscale Funnel**

```bash
tailscale funnel 3978
# استخدم URL الخاص بـ Tailscale funnel كنقطة نهاية للمراسلة
```

## Teams Developer Portal (بديل)

بدلًا من إنشاء manifest ZIP يدويًا، يمكنك استخدام [Teams Developer Portal](https://dev.teams.microsoft.com/apps):

1. انقر **+ New app**
2. املأ المعلومات الأساسية (الاسم، الوصف، معلومات المطور)
3. انتقل إلى **App features** → **Bot**
4. اختر **Enter a bot ID manually** والصق Azure Bot App ID
5. فعّل النطاقات: **Personal** و**Team** و**Group Chat**
6. انقر **Distribute** → **Download app package**
7. في Teams: **Apps** → **Manage your apps** → **Upload a custom app** → اختر ملف ZIP

غالبًا ما يكون هذا أسهل من تعديل ملفات manifest JSON يدويًا.

## اختبار البوت

**الخيار A: Azure Web Chat (تحقق من webhook أولًا)**

1. في Azure Portal → مورد Azure Bot الخاص بك → **Test in Web Chat**
2. أرسل رسالة - يجب أن ترى ردًا
3. وهذا يؤكد أن نقطة نهاية webhook تعمل قبل إعداد Teams

**الخيار B: Teams (بعد تثبيت التطبيق)**

1. ثبّت تطبيق Teams (تحميل جانبي أو كتالوج المؤسسة)
2. اعثر على البوت في Teams وأرسل رسالة مباشرة
3. تحقق من سجلات gateway لرؤية النشاط الوارد

## الإعداد (الحد الأدنى للنص فقط)

1. **تأكد من أن المكون الإضافي لـ Microsoft Teams متاح**
   - إصدارات OpenClaw المعبأة الحالية تتضمنه بالفعل.
   - يمكن للإصدارات/التثبيتات الأقدم أو المخصصة إضافته يدويًا:
     - من npm: `openclaw plugins install @openclaw/msteams`
     - من نسخة checkout محلية: `openclaw plugins install ./path/to/local/msteams-plugin`

2. **تسجيل البوت**
   - أنشئ Azure Bot (راجع أعلاه) ودوّن:
     - App ID
     - Client secret ‏(App password)
     - Tenant ID ‏(single-tenant)

3. **manifest تطبيق Teams**
   - أضف إدخال `bot` حيث `botId = <App ID>`.
   - النطاقات: `personal`, `team`, `groupChat`.
   - `supportsFiles: true` ‏(مطلوب لمعالجة الملفات في النطاق الشخصي).
   - أضف أذونات RSC (أدناه).
   - أنشئ الأيقونات: `outline.png` ‏(32x32) و`color.png` ‏(192x192).
   - اضغط الملفات الثلاثة معًا في zip: ‏`manifest.json`, `outline.png`, `color.png`.

4. **اضبط OpenClaw**

   ```json5
   {
     channels: {
       msteams: {
         enabled: true,
         appId: "<APP_ID>",
         appPassword: "<APP_PASSWORD>",
         tenantId: "<TENANT_ID>",
         webhook: { port: 3978, path: "/api/messages" },
       },
     },
   }
   ```

   يمكنك أيضًا استخدام متغيرات البيئة بدلًا من مفاتيح الإعداد:
   - `MSTEAMS_APP_ID`
   - `MSTEAMS_APP_PASSWORD`
   - `MSTEAMS_TENANT_ID`

5. **نقطة نهاية البوت**
   - اضبط Azure Bot Messaging Endpoint على:
     - `https://<host>:3978/api/messages` ‏(أو المسار/المنفذ الذي اخترته).

6. **شغّل gateway**
   - تبدأ قناة Teams تلقائيًا عندما يكون المكون الإضافي المضمّن أو المثبّت يدويًا متاحًا وتوجد إعدادات `msteams` مع بيانات الاعتماد.

## إجراء معلومات العضو

يعرض OpenClaw إجراء `member-info` مدعومًا من Graph لـ Microsoft Teams حتى تتمكن الوكلاء وعمليات الأتمتة من حل تفاصيل أعضاء القناة (اسم العرض والبريد الإلكتروني والدور) مباشرةً من Microsoft Graph.

المتطلبات:

- إذن RSC ‏`Member.Read.Group` (موجود بالفعل في manifest الموصى به)
- لعمليات البحث عبر الفرق: إذن تطبيق Graph ‏`User.Read.All` مع موافقة المسؤول

يخضع الإجراء لـ `channels.msteams.actions.memberInfo` (الافتراضي: مفعّل عند توفر بيانات اعتماد Graph).

## سياق السجل

- يتحكم `channels.msteams.historyLimit` في عدد رسائل القناة/المجموعة الأخيرة التي تُغلَّف داخل الموجّه.
- يعود إلى `messages.groupChat.historyLimit`. اضبطه إلى `0` للتعطيل (الافتراضي 50).
- يُرشّح سجل السلسلة الذي يتم جلبه بواسطة قوائم سماح المرسلين (`allowFrom` / `groupAllowFrom`)، لذا فإن تهيئة سياق السلسلة تتضمن فقط الرسائل من المرسلين المسموح لهم.
- يتم تمرير سياق المرفقات المقتبسة (`ReplyTo*` المشتق من HTML الرد في Teams) حاليًا كما تم استلامه.
- بمعنى آخر، تتحكم قوائم السماح في من يمكنه تشغيل الوكيل؛ ولا تُرشّح اليوم إلا بعض مسارات السياق التكميلي المحددة.
- يمكن تقييد سجل الرسائل المباشرة عبر `channels.msteams.dmHistoryLimit` (أدوار المستخدم). والتجاوزات لكل مستخدم: `channels.msteams.dms["<user_id>"].historyLimit`.

## أذونات Teams RSC الحالية (Manifest)

هذه هي أذونات **resourceSpecific** الحالية في manifest تطبيق Teams لدينا. وهي لا تنطبق إلا داخل الفريق/الدردشة التي ثُبّت فيها التطبيق.

**للقنوات (نطاق الفريق):**

- `ChannelMessage.Read.Group` ‏(Application) - تلقي جميع رسائل القناة بدون @mention
- `ChannelMessage.Send.Group` ‏(Application)
- `Member.Read.Group` ‏(Application)
- `Owner.Read.Group` ‏(Application)
- `ChannelSettings.Read.Group` ‏(Application)
- `TeamMember.Read.Group` ‏(Application)
- `TeamSettings.Read.Group` ‏(Application)

**للدردشات الجماعية:**

- `ChatMessage.Read.Chat` ‏(Application) - تلقي جميع رسائل الدردشة الجماعية بدون @mention

## مثال على Teams Manifest (منقّح)

مثال أدنى صالح مع الحقول المطلوبة. استبدل المعرّفات وعناوين URL.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Your Org",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### ملاحظات manifest المهمة (حقول لا غنى عنها)

- يجب أن يطابق `bots[].botId` **تمامًا** Azure Bot App ID.
- يجب أن يطابق `webApplicationInfo.id` **تمامًا** Azure Bot App ID.
- يجب أن يتضمن `bots[].scopes` الأسطح التي تخطط لاستخدامها (`personal`, `team`, `groupChat`).
- يتطلب `bots[].supportsFiles: true` لمعالجة الملفات في النطاق الشخصي.
- يجب أن يتضمن `authorization.permissions.resourceSpecific` أذونات قراءة/إرسال القنوات إذا كنت تريد حركة مرور القنوات.

### تحديث تطبيق موجود

لتحديث تطبيق Teams مثبّت بالفعل (مثلًا لإضافة أذونات RSC):

1. حدّث `manifest.json` بالإعدادات الجديدة
2. **زد الحقل `version`** ‏(مثلًا `1.0.0` → `1.1.0`)
3. **أعد ضغط** manifest مع الأيقونات (`manifest.json`, `outline.png`, `color.png`)
4. ارفع ملف zip الجديد:
   - **الخيار A (Teams Admin Center):** Teams Admin Center → Teams apps → Manage apps → اعثر على تطبيقك → Upload new version
   - **الخيار B (Sideload):** في Teams → Apps → Manage your apps → Upload a custom app
5. **لقنوات الفرق:** أعد تثبيت التطبيق في كل فريق حتى تدخل الأذونات الجديدة حيز التنفيذ
6. **أغلق Teams تمامًا ثم أعد تشغيله** (وليس مجرد إغلاق النافذة) لمسح بيانات تعريف التطبيق المخزنة مؤقتًا

## الإمكانات: RSC فقط مقابل Graph

### مع **Teams RSC فقط** (التطبيق مثبت، من دون أذونات Microsoft Graph API)

يعمل:

- قراءة محتوى **النص** لرسائل القنوات.
- إرسال محتوى **نص** رسائل القنوات.
- استقبال مرفقات الملفات في **النطاق الشخصي (DM)**.

لا يعمل:

- **الصور أو محتويات الملفات** في القنوات/المجموعات (تتضمن الحمولة مجرد HTML stub).
- تنزيل المرفقات المخزنة في SharePoint/OneDrive.
- قراءة سجل الرسائل (فيما يتجاوز حدث webhook المباشر).

### مع **Teams RSC + أذونات تطبيق Microsoft Graph**

يضيف:

- تنزيل المحتويات المستضافة (الصور الملصقة داخل الرسائل).
- تنزيل مرفقات الملفات المخزنة في SharePoint/OneDrive.
- قراءة سجل رسائل القنوات/الدردشات عبر Graph.

### RSC مقابل Graph API

| الإمكانية               | أذونات RSC            | Graph API                            |
| ----------------------- | --------------------- | ------------------------------------ |
| **الرسائل الآنية**      | نعم (عبر webhook)     | لا (استطلاع فقط)                     |
| **الرسائل التاريخية**   | لا                    | نعم (يمكن الاستعلام عن السجل)        |
| **تعقيد الإعداد**       | manifest التطبيق فقط  | يتطلب موافقة المسؤول + تدفق token    |
| **يعمل دون اتصال**      | لا (يجب أن يكون قيد التشغيل) | نعم (يمكن الاستعلام في أي وقت) |

**الخلاصة:** يستخدم RSC للاستماع الآني؛ ويُستخدم Graph API للوصول إلى السجل التاريخي. وللحاق بالرسائل الفائتة أثناء عدم الاتصال، تحتاج إلى Graph API مع `ChannelMessage.Read.All` ‏(يتطلب موافقة المسؤول).

## الوسائط + السجل المفعّلان عبر Graph (مطلوبان للقنوات)

إذا كنت تحتاج إلى الصور/الملفات في **القنوات** أو تريد جلب **سجل الرسائل**، فيجب تفعيل أذونات Microsoft Graph ومنح موافقة المسؤول.

1. في Entra ID ‏(Azure AD) **App Registration**، أضف أذونات **Application** لـ Microsoft Graph:
   - `ChannelMessage.Read.All` ‏(مرفقات القنوات + السجل)
   - `Chat.Read.All` أو `ChatMessage.Read.All` ‏(للدردشات الجماعية)
2. **امنح موافقة المسؤول** للمستأجر.
3. زِد **إصدار manifest** لتطبيق Teams، وأعد رفعه، و**أعد تثبيت التطبيق في Teams**.
4. **أغلق Teams تمامًا ثم أعد تشغيله** لمسح بيانات تعريف التطبيق المخزنة مؤقتًا.

**إذن إضافي لإشارات المستخدمين:** تعمل إشارات @ للمستخدمين مباشرةً للمستخدمين الموجودين في المحادثة. ولكن إذا أردت البحث ديناميكيًا عن مستخدمين والإشارة إليهم وهم **ليسوا في المحادثة الحالية**، فأضف إذن `User.Read.All` ‏(Application) وامنح موافقة المسؤول.

## القيود المعروفة

### مهلات webhook

يسلّم Teams الرسائل عبر HTTP webhook. وإذا استغرقت المعالجة وقتًا طويلًا جدًا (مثل بطء استجابات LLM)، فقد ترى:

- مهلات gateway
- إعادة Teams محاولة إرسال الرسالة (مما يسبب التكرار)
- إسقاط الردود

يتعامل OpenClaw مع ذلك عبر إعادة الاستجابة بسرعة وإرسال الردود بشكل استباقي، ولكن قد تستمر المشكلات مع الاستجابات البطيئة جدًا.

### التنسيق

Markdown في Teams أكثر محدودية من Slack أو Discord:

- يعمل التنسيق الأساسي: **غامق** و_مائل_ و`code` والروابط
- قد لا تُعرض Markdown المعقدة (الجداول والقوائم المتداخلة) بشكل صحيح
- تُدعم Adaptive Cards للاستطلاعات وإرسال البطاقات العامة (راجع أدناه)

## الإعداد

الإعدادات الأساسية (راجع `/gateway/configuration` لأنماط القنوات المشتركة):

- `channels.msteams.enabled`: تفعيل/تعطيل القناة.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: بيانات اعتماد البوت.
- `channels.msteams.webhook.port` ‏(الافتراضي `3978`)
- `channels.msteams.webhook.path` ‏(الافتراضي `/api/messages`)
- `channels.msteams.dmPolicy`: ‏`pairing | allowlist | open | disabled` ‏(الافتراضي: pairing)
- `channels.msteams.allowFrom`: قائمة السماح للرسائل المباشرة (يوصى بمعرّفات كائنات AAD). يحل المعالج الأسماء إلى معرّفات أثناء الإعداد عندما يكون وصول Graph متاحًا.
- `channels.msteams.dangerouslyAllowNameMatching`: مفتاح طوارئ لإعادة تفعيل مطابقة UPN/أسماء العرض القابلة للتغيير والتوجيه المباشر بأسماء الفرق/القنوات.
- `channels.msteams.textChunkLimit`: حجم تقسيم النص الصادر.
- `channels.msteams.chunkMode`: ‏`length` ‏(الافتراضي) أو `newline` للتقسيم على الأسطر الفارغة (حدود الفقرات) قبل التقسيم حسب الطول.
- `channels.msteams.mediaAllowHosts`: قائمة السماح لمضيفي المرفقات الواردة (الافتراضي نطاقات Microsoft/Teams).
- `channels.msteams.mediaAuthAllowHosts`: قائمة السماح لإرفاق رؤوس Authorization عند إعادة محاولة الوسائط (الافتراضي مضيفو Graph + Bot Framework).
- `channels.msteams.requireMention`: طلب @mention في القنوات/المجموعات (الافتراضي true).
- `channels.msteams.replyStyle`: ‏`thread | top-level` ‏(راجع [نمط الرد](#reply-style-threads-vs-posts)).
- `channels.msteams.teams.<teamId>.replyStyle`: تجاوز لكل فريق.
- `channels.msteams.teams.<teamId>.requireMention`: تجاوز لكل فريق.
- `channels.msteams.teams.<teamId>.tools`: تجاوزات سياسة الأدوات الافتراضية لكل فريق (`allow`/`deny`/`alsoAllow`) تُستخدم عندما يكون تجاوز القناة مفقودًا.
- `channels.msteams.teams.<teamId>.toolsBySender`: تجاوزات سياسة الأدوات الافتراضية لكل فريق ولكل مرسل (الرمز الشامل `"*"` مدعوم).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: تجاوز لكل قناة.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: تجاوز لكل قناة.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: تجاوزات سياسة الأدوات لكل قناة (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: تجاوزات سياسة الأدوات لكل قناة ولكل مرسل (الرمز الشامل `"*"` مدعوم).
- يجب أن تستخدم مفاتيح `toolsBySender` بادئات صريحة:
  `id:`, `e164:`, `username:`, `name:` (أما المفاتيح القديمة غير المسبوقة فما تزال تُطابق `id:` فقط).
- `channels.msteams.actions.memberInfo`: تفعيل أو تعطيل إجراء معلومات العضو المدعوم من Graph (الافتراضي: مفعّل عند توفر بيانات اعتماد Graph).
- `channels.msteams.sharePointSiteId`: معرّف موقع SharePoint لرفع الملفات في الدردشات/القنوات الجماعية (راجع [إرسال الملفات في الدردشات الجماعية](#sending-files-in-group-chats)).

## التوجيه والجلسات

- تتبع مفاتيح الجلسات صيغة الوكيل القياسية (راجع [/concepts/session](/concepts/session)):
  - تشترك الرسائل المباشرة في الجلسة الرئيسية (`agent:<agentId>:<mainKey>`).
  - تستخدم رسائل القنوات/المجموعات معرّف المحادثة:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## نمط الرد: Threads مقابل Posts

قدّم Teams مؤخرًا نمطي واجهة مستخدم للقنوات فوق نموذج البيانات الأساسي نفسه:

| النمط                    | الوصف                                                    | `replyStyle` الموصى به |
| ------------------------ | -------------------------------------------------------- | ---------------------- |
| **Posts** ‏(الكلاسيكي)   | تظهر الرسائل كبطاقات مع ردود مترابطة تحتها              | `thread` ‏(الافتراضي)  |
| **Threads** ‏(على نمط Slack) | تتدفق الرسائل خطيًا، بشكل أقرب إلى Slack              | `top-level`            |

**المشكلة:** لا تعرض Teams API نمط واجهة المستخدم الذي تستخدمه القناة. إذا استخدمت `replyStyle` خاطئًا:

- `thread` في قناة بنمط Threads → تظهر الردود متداخلة بشكل غير ملائم
- `top-level` في قناة بنمط Posts → تظهر الردود كمنشورات مستقلة على المستوى الأعلى بدلًا من أن تكون داخل السلسلة

**الحل:** اضبط `replyStyle` لكل قناة بحسب إعداد القناة:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

## المرفقات والصور

**القيود الحالية:**

- **الرسائل المباشرة:** تعمل الصور ومرفقات الملفات عبر Teams bot file APIs.
- **القنوات/المجموعات:** تعيش المرفقات في تخزين M365 ‏(SharePoint/OneDrive). لا تتضمن حمولة webhook إلا HTML stub، وليس بايتات الملف الفعلية. **أذونات Graph API مطلوبة** لتنزيل مرفقات القنوات.
- لعمليات الإرسال الصريحة التي تبدأ بالملف، استخدم `action=upload-file` مع `media` / `filePath` / `path`؛ وتصبح `message` الاختيارية هي النص/التعليق المصاحب، بينما يتجاوز `filename` الاسم المرفوع.

من دون أذونات Graph، ستُستقبل رسائل القنوات التي تحتوي على صور كنص فقط (ولا يمكن للبوت الوصول إلى محتوى الصورة).
وبشكل افتراضي، ينزّل OpenClaw الوسائط فقط من أسماء مضيفي Microsoft/Teams. ويمكنك تجاوز ذلك عبر `channels.msteams.mediaAllowHosts` ‏(استخدم `["*"]` للسماح بأي مضيف).
ولا تُرفق رؤوس Authorization إلا للمضيفين المدرجين في `channels.msteams.mediaAuthAllowHosts` ‏(الافتراضي مضيفو Graph + Bot Framework). أبقِ هذه القائمة صارمة (وتجنب اللواحق متعددة المستأجرين).

## إرسال الملفات في الدردشات الجماعية

يمكن للبوتات إرسال الملفات في الرسائل المباشرة باستخدام تدفق FileConsentCard ‏(مدمج). لكن **إرسال الملفات في الدردشات/القنوات الجماعية** يتطلب إعدادًا إضافيًا:

| السياق                   | كيفية إرسال الملفات                       | الإعداد المطلوب                                 |
| ------------------------ | ----------------------------------------- | ----------------------------------------------- |
| **الرسائل المباشرة**     | FileConsentCard → يقبل المستخدم → يرفع البوت | يعمل مباشرةً                                    |
| **الدردشات/القنوات الجماعية** | رفع إلى SharePoint → رابط مشاركة        | يتطلب `sharePointSiteId` + أذونات Graph         |
| **الصور (أي سياق)**      | مضمنة داخل الرسالة بترميز Base64          | تعمل مباشرةً                                    |

### لماذا تحتاج الدردشات الجماعية إلى SharePoint

لا تمتلك البوتات محرك OneDrive شخصيًا (ونقطة نهاية Graph API ‏`/me/drive` لا تعمل لهويات التطبيقات). ولإرسال الملفات في الدردشات/القنوات الجماعية، يرفع البوت الملف إلى **موقع SharePoint** وينشئ رابط مشاركة.

### الإعداد

1. **أضف أذونات Graph API** في Entra ID ‏(Azure AD) → App Registration:
   - `Sites.ReadWrite.All` ‏(Application) - رفع الملفات إلى SharePoint
   - `Chat.Read.All` ‏(Application) - اختياري، يفعّل روابط المشاركة لكل مستخدم

2. **امنح موافقة المسؤول** للمستأجر.

3. **احصل على معرّف موقع SharePoint الخاص بك:**

   ```bash
   # عبر Graph Explorer أو curl باستخدام token صالح:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # مثال: لموقع على "contoso.sharepoint.com/sites/BotFiles"
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # تتضمن الاستجابة: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **اضبط OpenClaw:**

   ```json5
   {
     channels: {
       msteams: {
         // ... إعداد آخر ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### سلوك المشاركة

| الإذن                                    | سلوك المشاركة                                             |
| ---------------------------------------- | ---------------------------------------------------------- |
| `Sites.ReadWrite.All` فقط                | رابط مشاركة على مستوى المؤسسة (يمكن لأي شخص في المؤسسة الوصول) |
| `Sites.ReadWrite.All` + `Chat.Read.All`  | رابط مشاركة لكل مستخدم (يمكن فقط لأعضاء الدردشة الوصول)   |

تكون المشاركة لكل مستخدم أكثر أمانًا لأن المشاركين في الدردشة فقط يمكنهم الوصول إلى الملف. وإذا كان إذن `Chat.Read.All` مفقودًا، يعود البوت إلى المشاركة على مستوى المؤسسة.

### سلوك الرجوع الاحتياطي

| السيناريو                                         | النتيجة                                             |
| ------------------------------------------------- | --------------------------------------------------- |
| دردشة جماعية + ملف + تم إعداد `sharePointSiteId` | رفع إلى SharePoint وإرسال رابط مشاركة              |
| دردشة جماعية + ملف + لا يوجد `sharePointSiteId`  | محاولة رفع إلى OneDrive ‏(قد تفشل) وإرسال نص فقط   |
| دردشة شخصية + ملف                                | تدفق FileConsentCard ‏(يعمل بدون SharePoint)       |
| أي سياق + صورة                                   | مضمنة داخل الرسالة بترميز Base64 ‏(تعمل بدون SharePoint) |

### موقع تخزين الملفات

تُخزَّن الملفات المرفوعة في مجلد `/OpenClawShared/` داخل مكتبة المستندات الافتراضية لموقع SharePoint المُعد.

## الاستطلاعات (Adaptive Cards)

يرسل OpenClaw استطلاعات Teams على شكل Adaptive Cards ‏(لا توجد Teams poll API أصلية).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> ...`
- تُسجَّل الأصوات بواسطة gateway في `~/.openclaw/msteams-polls.json`.
- يجب أن يبقى gateway متصلًا لتسجيل الأصوات.
- لا تنشر الاستطلاعات تلقائيًا ملخصات النتائج بعدُ (افحص ملف التخزين إذا لزم الأمر).

## Adaptive Cards (عامة)

أرسل أي JSON خاص بـ Adaptive Card إلى مستخدمي Teams أو المحادثات باستخدام أداة `message` أو CLI.

تقبل المعلمة `card` كائن JSON خاص بـ Adaptive Card. وعند توفير `card` يصبح نص الرسالة اختياريًا.

**أداة الوكيل:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  card: {
    type: "AdaptiveCard",
    version: "1.5",
    body: [{ type: "TextBlock", text: "Hello!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Hello!"}]}'
```

راجع [Adaptive Cards documentation](https://adaptivecards.io/) للحصول على مخطط البطاقات والأمثلة. ولتفاصيل تنسيق الأهداف، راجع [تنسيقات الأهداف](#target-formats) أدناه.

## تنسيقات الأهداف

تستخدم أهداف MSTeams بادئات للتمييز بين المستخدمين والمحادثات:

| نوع الهدف              | التنسيق                          | المثال                                              |
| ---------------------- | -------------------------------- | --------------------------------------------------- |
| مستخدم (حسب المعرّف)   | `user:<aad-object-id>`           | `user:40a1a0ed-4ff2-4164-a219-55518990c197`         |
| مستخدم (حسب الاسم)     | `user:<display-name>`            | `user:John Smith` ‏(يتطلب Graph API)               |
| مجموعة/قناة            | `conversation:<conversation-id>` | `conversation:19:abc123...@thread.tacv2`            |
| مجموعة/قناة (خام)      | `<conversation-id>`              | `19:abc123...@thread.tacv2` ‏(إذا كان يتضمن `@thread`) |

**أمثلة CLI:**

```bash
# الإرسال إلى مستخدم حسب المعرّف
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "Hello"

# الإرسال إلى مستخدم حسب اسم العرض (يُفعّل بحث Graph API)
openclaw message send --channel msteams --target "user:John Smith" --message "Hello"

# الإرسال إلى دردشة جماعية أو قناة
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "Hello"

# إرسال Adaptive Card إلى محادثة
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Hello"}]}'
```

**أمثلة أداة الوكيل:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "Hello!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  card: {
    type: "AdaptiveCard",
    version: "1.5",
    body: [{ type: "TextBlock", text: "Hello" }],
  },
}
```

ملاحظة: من دون البادئة `user:`، تُفسَّر الأسماء افتراضيًا على أنها حل لمجموعة/فريق. استخدم دائمًا `user:` عند استهداف أشخاص باسم العرض.

## المراسلة الاستباقية

- لا يمكن إرسال الرسائل الاستباقية إلا **بعد** أن يتفاعل المستخدم، لأننا نخزن مراجع المحادثات في تلك اللحظة.
- راجع `/gateway/configuration` لمعرفة `dmPolicy` وتقييد قائمة السماح.

## معرّفات الفريق والقناة (مشكلة شائعة)

إن معلمة الاستعلام `groupId` في عناوين URL الخاصة بـ Teams **ليست** معرّف الفريق المستخدم في الإعداد. استخرج المعرّفات من مسار URL بدلًا من ذلك:

**URL الفريق:**

```
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    معرّف الفريق (نفّذ URL-decode لهذا)
```

**URL القناة:**

```
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      معرّف القناة (نفّذ URL-decode لهذا)
```

**للإعداد:**

- معرّف الفريق = مقطع المسار بعد `/team/` ‏(بعد URL-decoded، مثل `19:Bk4j...@thread.tacv2`)
- معرّف القناة = مقطع المسار بعد `/channel/` ‏(بعد URL-decoded)
- **تجاهل** معلمة الاستعلام `groupId`

## القنوات الخاصة

تملك البوتات دعمًا محدودًا في القنوات الخاصة:

| الميزة                       | القنوات القياسية   | القنوات الخاصة        |
| ---------------------------- | ------------------ | --------------------- |
| تثبيت البوت                  | نعم                | محدود                 |
| الرسائل الآنية (webhook)     | نعم                | قد لا تعمل            |
| أذونات RSC                   | نعم                | قد تتصرف بشكل مختلف   |
| @mentions                    | نعم                | إذا كان البوت متاحًا   |
| سجل Graph API                | نعم                | نعم (مع الأذونات)     |

**حلول بديلة إذا لم تعمل القنوات الخاصة:**

1. استخدم القنوات القياسية لتفاعلات البوت
2. استخدم الرسائل المباشرة - يمكن للمستخدمين دائمًا مراسلة البوت مباشرةً
3. استخدم Graph API للوصول إلى السجل التاريخي (يتطلب `ChannelMessage.Read.All`)

## استكشاف الأخطاء وإصلاحها

### مشكلات شائعة

- **الصور لا تظهر في القنوات:** أذونات Graph أو موافقة المسؤول مفقودة. أعد تثبيت تطبيق Teams وأغلق Teams تمامًا ثم أعد فتحه.
- **لا توجد ردود في القناة:** الإشارات مطلوبة افتراضيًا؛ اضبط `channels.msteams.requireMention=false` أو قم بالإعداد لكل فريق/قناة.
- **عدم تطابق الإصدار (ما يزال Teams يعرض manifest قديمًا):** أزل التطبيق ثم أعد إضافته وأغلق Teams تمامًا لتحديثه.
- **401 Unauthorized من webhook:** هذا متوقع عند الاختبار اليدوي من دون Azure JWT - ويعني أن نقطة النهاية قابلة للوصول لكن المصادقة فشلت. استخدم Azure Web Chat للاختبار بشكل صحيح.

### أخطاء رفع manifest

- **"Icon file cannot be empty":** يشير manifest إلى ملفات أيقونات حجمها 0 بايت. أنشئ أيقونات PNG صالحة (`outline.png` بحجم 32x32 و`color.png` بحجم 192x192).
- **"webApplicationInfo.Id already in use":** لا يزال التطبيق مثبتًا في فريق/دردشة أخرى. اعثر عليه وأزل تثبيته أولًا، أو انتظر 5-10 دقائق لانتشار التغيير.
- **"Something went wrong" on upload:** ارفع عبر [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com) بدلًا من ذلك، وافتح أدوات المطور في المتصفح (F12) → علامة Network، ثم تحقق من جسم الاستجابة لرؤية الخطأ الفعلي.
- **فشل التحميل الجانبي:** جرّب "Upload an app to your org's app catalog" بدلًا من "Upload a custom app" - فهذا يتجاوز غالبًا قيود التحميل الجانبي.

### أذونات RSC لا تعمل

1. تحقق من أن `webApplicationInfo.id` يطابق App ID الخاص بالبوت تمامًا
2. أعد رفع التطبيق وأعد تثبيته في الفريق/الدردشة
3. تحقق مما إذا كان مسؤول المؤسسة قد حظر أذونات RSC
4. تأكد من أنك تستخدم النطاق الصحيح: `ChannelMessage.Read.Group` للفرق، و`ChatMessage.Read.Chat` للدردشات الجماعية

## المراجع

- [Create Azure Bot](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - دليل إعداد Azure Bot
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - إنشاء/إدارة تطبيقات Teams
- [Teams app manifest schema](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [Receive channel messages with RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [RSC permissions reference](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Teams bot file handling](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) ‏(القناة/المجموعة تتطلب Graph)
- [Proactive messaging](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الإقران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الإقران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
