---
read_when:
    - تريد توصيل بوت Feishu/Lark
    - أنت تقوم بإعداد قناة Feishu
summary: نظرة عامة على بوت Feishu وميزاته وإعداده
title: Feishu
x-i18n:
    generated_at: "2026-04-05T12:35:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e39b6dfe3a3aa4ebbdb992975e570e4f1b5e79f3b400a555fc373a0d1889952
    source_path: channels/feishu.md
    workflow: 15
---

# بوت Feishu

Feishu ‏(Lark) هو نظام دردشة جماعي تستخدمه الشركات للمراسلة والتعاون. يربط هذا الـ plugin بين OpenClaw وبوت Feishu/Lark باستخدام اشتراك أحداث WebSocket الخاص بالمنصة، بحيث يمكن استقبال الرسائل من دون تعريض عنوان URL عام لـ webhook.

---

## الـ plugin المضمّن

يأتي Feishu مضمّنًا مع إصدارات OpenClaw الحالية، لذلك لا يلزم تثبيت plugin منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن Feishu المضمّن، فقم بتثبيته يدويًا:

```bash
openclaw plugins install @openclaw/feishu
```

---

## البدء السريع

هناك طريقتان لإضافة قناة Feishu:

### الطريقة 1: الإعداد الأولي (مستحسن)

إذا كنت قد ثبّت OpenClaw للتو، فشغّل الإعداد الأولي:

```bash
openclaw onboard
```

سيرشدك المعالج خلال ما يلي:

1. إنشاء تطبيق Feishu وجمع بيانات الاعتماد
2. إعداد بيانات اعتماد التطبيق في OpenClaw
3. بدء تشغيل البوابة

✅ **بعد الإعداد**، تحقّق من حالة البوابة:

- `openclaw gateway status`
- `openclaw logs --follow`

### الطريقة 2: الإعداد عبر CLI

إذا كنت قد أكملت التثبيت الأولي بالفعل، فأضف القناة عبر CLI:

```bash
openclaw channels add
```

اختر **Feishu**، ثم أدخل App ID وApp Secret.

✅ **بعد الإعداد**، أدر البوابة:

- `openclaw gateway status`
- `openclaw gateway restart`
- `openclaw logs --follow`

---

## الخطوة 1: إنشاء تطبيق Feishu

### 1. افتح Feishu Open Platform

زر [Feishu Open Platform](https://open.feishu.cn/app) وسجّل الدخول.

يجب على مستأجري Lark (العالمي) استخدام [https://open.larksuite.com/app](https://open.larksuite.com/app) وضبط `domain: "lark"` في إعدادات Feishu.

### 2. أنشئ تطبيقًا

1. انقر **Create enterprise app**
2. املأ اسم التطبيق والوصف
3. اختر أيقونة للتطبيق

![Create enterprise app](/images/feishu-step2-create-app.png)

### 3. انسخ بيانات الاعتماد

من **Credentials & Basic Info**، انسخ:

- **App ID** (بالتنسيق: `cli_xxx`)
- **App Secret**

❗ **مهم:** احتفظ بـ App Secret بشكل خاص.

![Get credentials](/images/feishu-step3-credentials.png)

### 4. إعداد الأذونات

في **Permissions**، انقر **Batch import** والصق:

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": ["aily:file:read", "aily:file:write", "im:chat.access_event.bot_p2p_chat:read"]
  }
}
```

![Configure permissions](/images/feishu-step4-permissions.png)

### 5. فعّل إمكانية البوت

في **App Capability** > **Bot**:

1. فعّل إمكانية البوت
2. عيّن اسم البوت

![Enable bot capability](/images/feishu-step5-bot-capability.png)

### 6. إعداد اشتراك الأحداث

⚠️ **مهم:** قبل إعداد اشتراك الأحداث، تأكد من:

1. أنك شغّلت بالفعل `openclaw channels add` لـ Feishu
2. أن البوابة قيد التشغيل (`openclaw gateway status`)

في **Event Subscription**:

1. اختر **Use long connection to receive events** ‏(WebSocket)
2. أضف الحدث: `im.message.receive_v1`
3. (اختياري) لمهام تعليقات Drive، أضف أيضًا: `drive.notice.comment_add_v1`

⚠️ إذا لم تكن البوابة قيد التشغيل، فقد يفشل حفظ إعداد الاتصال الطويل.

![Configure event subscription](/images/feishu-step6-event-subscription.png)

### 7. انشر التطبيق

1. أنشئ إصدارًا في **Version Management & Release**
2. أرسله للمراجعة وانشره
3. انتظر موافقة المسؤول (تطبيقات المؤسسات تُوافق عادةً تلقائيًا)

---

## الخطوة 2: إعداد OpenClaw

### الإعداد باستخدام المعالج (مستحسن)

```bash
openclaw channels add
```

اختر **Feishu** والصق App ID وApp Secret.

### الإعداد عبر ملف التهيئة

حرّر `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    feishu: {
      enabled: true,
      dmPolicy: "pairing",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "My AI assistant",
        },
      },
    },
  },
}
```

إذا كنت تستخدم `connectionMode: "webhook"`، فاضبط كلًا من `verificationToken` و`encryptKey`. يرتبط خادم webhook الخاص بـ Feishu على `127.0.0.1` افتراضيًا؛ لا تضبط `webhookHost` إلا إذا كنت تحتاج عمدًا إلى عنوان ربط مختلف.

#### Verification Token وEncrypt Key ‏(وضع webhook)

عند استخدام وضع webhook، اضبط كلًا من `channels.feishu.verificationToken` و`channels.feishu.encryptKey` في إعداداتك. للحصول على القيم:

1. في Feishu Open Platform، افتح تطبيقك
2. انتقل إلى **Development** → **Events & Callbacks** ‏(开发配置 → 事件与回调)
3. افتح تبويب **Encryption** ‏(加密策略)
4. انسخ **Verification Token** و**Encrypt Key**

توضح الصورة التالية مكان العثور على **Verification Token**. أما **Encrypt Key** فهو مدرج في القسم نفسه **Encryption**.

![Verification Token location](/images/feishu-verification-token.png)

### الإعداد عبر متغيرات البيئة

```bash
export FEISHU_APP_ID="cli_xxx"
export FEISHU_APP_SECRET="xxx"
```

### نطاق Lark (العالمي)

إذا كان المستأجر الخاص بك على Lark (الدولي)، فاضبط النطاق إلى `lark` (أو سلسلة نطاق كاملة). يمكنك ضبطه في `channels.feishu.domain` أو لكل حساب على حدة (`channels.feishu.accounts.<id>.domain`).

```json5
{
  channels: {
    feishu: {
      domain: "lark",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
        },
      },
    },
  },
}
```

### علامات تحسين الحصة

يمكنك تقليل استخدام Feishu API باستخدام علامتين اختياريتين:

- `typingIndicator` (الافتراضي `true`): عند ضبطه على `false`، يتم تخطي استدعاءات تفاعل الكتابة.
- `resolveSenderNames` (الافتراضي `true`): عند ضبطه على `false`، يتم تخطي استدعاءات البحث عن ملف المُرسِل الشخصي.

اضبطهما على المستوى الأعلى أو لكل حساب:

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          typingIndicator: true,
          resolveSenderNames: false,
        },
      },
    },
  },
}
```

---

## الخطوة 3: البدء + الاختبار

### 1. شغّل البوابة

```bash
openclaw gateway
```

### 2. أرسل رسالة اختبار

في Feishu، ابحث عن البوت وأرسل رسالة.

### 3. وافق على الاقتران

بشكل افتراضي، يرد البوت برمز اقتران. وافق عليه:

```bash
openclaw pairing approve feishu <CODE>
```

بعد الموافقة، يمكنك الدردشة بشكل طبيعي.

---

## نظرة عامة

- **قناة بوت Feishu**: بوت Feishu تديره البوابة
- **توجيه حتمي**: تعود الردود دائمًا إلى Feishu
- **عزل الجلسات**: تشترك الرسائل الخاصة في جلسة رئيسية؛ أما المجموعات فمعزولة
- **اتصال WebSocket**: اتصال طويل عبر Feishu SDK، ولا حاجة إلى عنوان URL عام

---

## التحكم في الوصول

### الرسائل المباشرة

- **الافتراضي**: `dmPolicy: "pairing"` ‏(يحصل المستخدمون غير المعروفين على رمز اقتران)
- **الموافقة على الاقتران**:

  ```bash
  openclaw pairing list feishu
  openclaw pairing approve feishu <CODE>
  ```

- **وضع قائمة السماح**: اضبط `channels.feishu.allowFrom` باستخدام Open IDs المسموح بها

### دردشات المجموعات

**1. سياسة المجموعات** (`channels.feishu.groupPolicy`):

- `"open"` = السماح للجميع في المجموعات
- `"allowlist"` = السماح فقط لما هو موجود في `groupAllowFrom`
- `"disabled"` = تعطيل رسائل المجموعات

الافتراضي: `allowlist`

**2. شرط الإشارة** (`channels.feishu.requireMention`، ويمكن تجاوزه عبر `channels.feishu.groups.<chat_id>.requireMention`):

- `true` صراحةً = يتطلب @mention
- `false` صراحةً = يرد من دون إشارات
- عند عدم ضبطه ووجود `groupPolicy: "open"` = الافتراضي `false`
- عند عدم ضبطه وكون `groupPolicy` ليس `"open"` = الافتراضي `true`

---

## أمثلة على إعداد المجموعات

### السماح لجميع المجموعات، من دون الحاجة إلى @mention ‏(الافتراضي للمجموعات المفتوحة)

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
    },
  },
}
```

### السماح لجميع المجموعات، مع الاستمرار في طلب @mention

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### السماح بمجموعات محددة فقط

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // Feishu group IDs (chat_id) look like: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

### تقييد المرسلين المسموح لهم بالمراسلة داخل مجموعة (قائمة سماح للمرسلين)

بالإضافة إلى السماح للمجموعة نفسها، فإن **كل الرسائل** داخل تلك المجموعة يتم ضبطها حسب `open_id` الخاص بالمرسل: فقط المستخدمون المدرجون في `groups.<chat_id>.allowFrom` تتم معالجة رسائلهم؛ أما الرسائل من الأعضاء الآخرين فيتم تجاهلها (هذا ضبط كامل على مستوى المرسل، وليس فقط لأوامر التحكم مثل `/reset` أو `/new`).

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // Feishu user IDs (open_id) look like: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

---

<a id="get-groupuser-ids"></a>

## الحصول على معرّفات المجموعة/المستخدم

### معرّفات المجموعات (chat_id)

تبدو معرّفات المجموعات بهذا الشكل `oc_xxx`.

**الطريقة 1 (مستحسنة)**

1. شغّل البوابة واذكر البوت باستخدام @mention داخل المجموعة
2. شغّل `openclaw logs --follow` وابحث عن `chat_id`

**الطريقة 2**

استخدم مصحح Feishu API لسرد دردشات المجموعات.

### معرّفات المستخدمين (open_id)

تبدو معرّفات المستخدمين بهذا الشكل `ou_xxx`.

**الطريقة 1 (مستحسنة)**

1. شغّل البوابة وأرسل رسالة مباشرة إلى البوت
2. شغّل `openclaw logs --follow` وابحث عن `open_id`

**الطريقة 2**

تحقق من طلبات الاقتران للحصول على Open IDs الخاصة بالمستخدمين:

```bash
openclaw pairing list feishu
```

---

## الأوامر الشائعة

| الأمر | الوصف |
| --------- | ----------------- |
| `/status` | عرض حالة البوت |
| `/reset`  | إعادة تعيين الجلسة |
| `/model`  | عرض/تبديل النموذج |

> ملاحظة: لا يدعم Feishu قوائم الأوامر الأصلية حتى الآن، لذلك يجب إرسال الأوامر كنص.

## أوامر إدارة البوابة

| الأمر | الوصف |
| -------------------------- | ----------------------------- |
| `openclaw gateway status`  | عرض حالة البوابة |
| `openclaw gateway install` | تثبيت/بدء خدمة البوابة |
| `openclaw gateway stop`    | إيقاف خدمة البوابة |
| `openclaw gateway restart` | إعادة تشغيل خدمة البوابة |
| `openclaw logs --follow`   | تتبع سجلات البوابة |

---

## استكشاف الأخطاء وإصلاحها

### البوت لا يستجيب في دردشات المجموعات

1. تأكد من إضافة البوت إلى المجموعة
2. تأكد من أنك تشير إلى البوت باستخدام @mention ‏(السلوك الافتراضي)
3. تحقق من أن `groupPolicy` ليس مضبوطًا على `"disabled"`
4. تحقق من السجلات: `openclaw logs --follow`

### البوت لا يستقبل الرسائل

1. تأكد من أن التطبيق منشور ومعتمد
2. تأكد من أن اشتراك الأحداث يتضمن `im.message.receive_v1`
3. تأكد من تفعيل **long connection**
4. تأكد من اكتمال أذونات التطبيق
5. تأكد من أن البوابة قيد التشغيل: `openclaw gateway status`
6. تحقق من السجلات: `openclaw logs --follow`

### تسرّب App Secret

1. أعد تعيين App Secret في Feishu Open Platform
2. حدّث App Secret في إعداداتك
3. أعد تشغيل البوابة

### فشل إرسال الرسائل

1. تأكد من أن التطبيق يملك إذن `im:message:send_as_bot`
2. تأكد من أن التطبيق منشور
3. تحقق من السجلات للحصول على أخطاء مفصلة

---

## الإعداد المتقدم

### حسابات متعددة

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primary bot",
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

يتحكم `defaultAccount` في حساب Feishu المستخدم عندما لا تحدد واجهات API الصادرة `accountId` بشكل صريح.

### حدود الرسائل

- `textChunkLimit`: حجم مقطع النص الصادر (الافتراضي: 2000 حرف)
- `mediaMaxMb`: حد رفع/تنزيل الوسائط (الافتراضي: 30MB)

### البث

يدعم Feishu الردود المتدفقة عبر البطاقات التفاعلية. عند التفعيل، يحدّث البوت بطاقة أثناء توليد النص.

```json5
{
  channels: {
    feishu: {
      streaming: true, // enable streaming card output (default true)
      blockStreaming: true, // enable block-level streaming (default true)
    },
  },
}
```

اضبط `streaming: false` للانتظار حتى يكتمل الرد بالكامل قبل الإرسال.

### جلسات ACP

يدعم Feishu ‏ACP في:

- الرسائل المباشرة
- محادثات موضوعات المجموعات

يعتمد ACP في Feishu على أوامر نصية. لا توجد قوائم أوامر slash أصلية، لذا استخدم رسائل `/acp ...` مباشرة في المحادثة.

#### ارتباطات ACP الدائمة

استخدم ارتباطات ACP المكتوبة على المستوى الأعلى لتثبيت رسالة مباشرة أو محادثة موضوع في Feishu إلى جلسة ACP دائمة.

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
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### إنشاء ACP مرتبط بسلسلة محادثة من الدردشة

في رسالة مباشرة أو محادثة موضوع في Feishu، يمكنك إنشاء جلسة ACP وربطها في مكانها:

```text
/acp spawn codex --thread here
```

ملاحظات:

- يعمل `--thread here` مع الرسائل المباشرة وموضوعات Feishu.
- يتم توجيه الرسائل اللاحقة في الرسالة المباشرة/الموضوع المرتبط مباشرة إلى جلسة ACP تلك.
- لا يستهدف الإصدار v1 دردشات المجموعات العامة غير المرتبطة بموضوع.

### توجيه متعدد الوكلاء

استخدم `bindings` لتوجيه الرسائل المباشرة أو المجموعات في Feishu إلى وكلاء مختلفين.

```json5
{
  agents: {
    list: [
      { id: "main" },
      {
        id: "clawd-fan",
        workspace: "/home/user/clawd-fan",
        agentDir: "/home/user/.openclaw/agents/clawd-fan/agent",
      },
      {
        id: "clawd-xi",
        workspace: "/home/user/clawd-xi",
        agentDir: "/home/user/.openclaw/agents/clawd-xi/agent",
      },
    ],
  },
  bindings: [
    {
      agentId: "main",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "clawd-fan",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_yyy" },
      },
    },
    {
      agentId: "clawd-xi",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

حقول التوجيه:

- `match.channel`: ‏`"feishu"`
- `match.peer.kind`: ‏`"direct"` أو `"group"`
- `match.peer.id`: معرّف Open للمستخدم (`ou_xxx`) أو معرّف المجموعة (`oc_xxx`)

راجع [الحصول على معرّفات المجموعة/المستخدم](#get-groupuser-ids) للحصول على نصائح البحث.

---

## مرجع الإعداد

الإعداد الكامل: [إعدادات البوابة](/gateway/configuration)

الخيارات الأساسية:

| الإعداد | الوصف | الافتراضي |
| ------------------------------------------------- | --------------------------------------- | ---------------- |
| `channels.feishu.enabled`                         | تفعيل/تعطيل القناة | `true` |
| `channels.feishu.domain`                          | نطاق API ‏(`feishu` أو `lark`) | `feishu` |
| `channels.feishu.connectionMode`                  | وضع نقل الأحداث | `websocket` |
| `channels.feishu.defaultAccount`                  | معرّف الحساب الافتراضي للتوجيه الصادر | `default` |
| `channels.feishu.verificationToken`               | مطلوب لوضع webhook | - |
| `channels.feishu.encryptKey`                      | مطلوب لوضع webhook | - |
| `channels.feishu.webhookPath`                     | مسار route لـ webhook | `/feishu/events` |
| `channels.feishu.webhookHost`                     | مضيف الربط لـ webhook | `127.0.0.1` |
| `channels.feishu.webhookPort`                     | منفذ الربط لـ webhook | `3000` |
| `channels.feishu.accounts.<id>.appId`             | App ID | - |
| `channels.feishu.accounts.<id>.appSecret`         | App Secret | - |
| `channels.feishu.accounts.<id>.domain`            | تجاوز نطاق API لكل حساب | `feishu` |
| `channels.feishu.dmPolicy`                        | سياسة الرسائل المباشرة | `pairing` |
| `channels.feishu.allowFrom`                       | قائمة السماح للرسائل المباشرة (قائمة `open_id`) | - |
| `channels.feishu.groupPolicy`                     | سياسة المجموعات | `allowlist` |
| `channels.feishu.groupAllowFrom`                  | قائمة السماح للمجموعات | - |
| `channels.feishu.requireMention`                  | طلب @mention افتراضيًا | مشروط |
| `channels.feishu.groups.<chat_id>.requireMention` | تجاوز طلب @mention لكل مجموعة | موروث |
| `channels.feishu.groups.<chat_id>.enabled`        | تفعيل المجموعة | `true` |
| `channels.feishu.textChunkLimit`                  | حجم مقطع الرسالة | `2000` |
| `channels.feishu.mediaMaxMb`                      | حد حجم الوسائط | `30` |
| `channels.feishu.streaming`                       | تفعيل خرج البطاقات المتدفقة | `true` |
| `channels.feishu.blockStreaming`                  | تفعيل البث على مستوى الكتل | `true` |

---

## مرجع dmPolicy

| القيمة | السلوك |
| ------------- | --------------------------------------------------------------- |
| `"pairing"`   | **الافتراضي.** يحصل المستخدمون غير المعروفين على رمز اقتران؛ ويجب اعتمادهم |
| `"allowlist"` | فقط المستخدمون الموجودون في `allowFrom` يمكنهم الدردشة |
| `"open"`      | السماح لجميع المستخدمين (يتطلب `"*"` في `allowFrom`) |
| `"disabled"`  | تعطيل الرسائل المباشرة |

---

## أنواع الرسائل المدعومة

### الاستقبال

- ✅ نص
- ✅ نص منسق (post)
- ✅ صور
- ✅ ملفات
- ✅ صوت
- ✅ فيديو/وسائط
- ✅ ملصقات

### الإرسال

- ✅ نص
- ✅ صور
- ✅ ملفات
- ✅ صوت
- ✅ فيديو/وسائط
- ✅ بطاقات تفاعلية
- ⚠️ نص منسق (تنسيق بنمط post وبطاقات، وليس ميزات التأليف العشوائية في Feishu)

### سلاسل المحادثات والردود

- ✅ ردود مضمّنة
- ✅ ردود سلسلة الموضوع عندما يوفّر Feishu ‏`reply_in_thread`
- ✅ تظل ردود الوسائط مدركة للسلسلة عند الرد على رسالة سلسلة/موضوع

## تعليقات Drive

يمكن لـ Feishu تشغيل الوكيل عندما يضيف شخص ما تعليقًا على مستند Feishu Drive ‏(Docs وSheets وغيرها). يتلقى الوكيل نص التعليق وسياق المستند وسلسلة التعليق حتى يتمكن من الرد داخل السلسلة أو إجراء تعديلات على المستند.

المتطلبات:

- الاشتراك في `drive.notice.comment_add_v1` ضمن إعدادات اشتراك أحداث تطبيق Feishu
  (إلى جانب `im.message.receive_v1` الموجود)
- أداة Drive مفعلة افتراضيًا؛ عطّلها باستخدام `channels.feishu.tools.drive: false`

تكشف أداة `feishu_drive` عن إجراءات التعليقات التالية:

| الإجراء | الوصف |
| ---------------------- | ----------------------------------- |
| `list_comments`        | سرد التعليقات على مستند |
| `list_comment_replies` | سرد الردود في سلسلة تعليق |
| `add_comment`          | إضافة تعليق جديد على المستوى الأعلى |
| `reply_comment`        | الرد على سلسلة تعليق موجودة |

عندما يتعامل الوكيل مع حدث تعليق Drive، فإنه يتلقى:

- نص التعليق والمرسل
- بيانات المستند الوصفية (العنوان والنوع وURL)
- سياق سلسلة التعليق للردود داخل السلسلة

بعد إجراء تعديلات على المستند، يتم توجيه الوكيل لاستخدام `feishu_drive.reply_comment` لإخطار
المعلّق ثم إخراج الرمز الصامت المطابق تمامًا `NO_REPLY` / `no_reply` لتجنب
الإرسال المكرر.

## سطح إجراءات وقت التشغيل

يكشف Feishu حاليًا عن إجراءات وقت التشغيل التالية:

- `send`
- `read`
- `edit`
- `thread-reply`
- `pin`
- `list-pins`
- `unpin`
- `member-info`
- `channel-info`
- `channel-list`
- `react` و`reactions` عندما تكون التفاعلات مفعّلة في الإعدادات
- إجراءات تعليقات `feishu_drive`: ‏`list_comments` و`list_comment_replies` و`add_comment` و`reply_comment`

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك دردشات المجموعات وضبط الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتحصين
