---
read_when:
    - إعداد Zalo Personal لـ OpenClaw
    - تصحيح مشكلات تسجيل الدخول أو تدفق الرسائل في Zalo Personal
summary: دعم حساب Zalo الشخصي عبر `zca-js` الأصلي (تسجيل دخول QR)، والقدرات، والإعدادات
title: Zalo Personal
x-i18n:
    generated_at: "2026-04-05T12:37:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 331b95041463185472d242cb0a944972f0a8e99df8120bda6350eca86ad5963f
    source_path: channels/zalouser.md
    workflow: 15
---

# Zalo Personal (غير رسمي)

الحالة: تجريبي. يقوم هذا التكامل بأتمتة **حساب Zalo شخصي** عبر `zca-js` الأصلي داخل OpenClaw.

> **تحذير:** هذا تكامل غير رسمي وقد يؤدي إلى تعليق الحساب أو حظره. استخدمه على مسؤوليتك الخاصة.

## plugin المضمّن

يأتي Zalo Personal كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك لا تحتاج
البنيات المعبأة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا يستبعد Zalo Personal،
فثبّته يدويًا:

- ثبّت عبر CLI: `openclaw plugins install @openclaw/zalouser`
- أو من نسخة checkout للمصدر: `openclaw plugins install ./path/to/local/zalouser-plugin`
- التفاصيل: [Plugins](/tools/plugin)

لا يلزم وجود ملف CLI تنفيذي خارجي لـ `zca`/`openzca`.

## إعداد سريع (للمبتدئين)

1. تأكد من أن plugin الخاص بـ Zalo Personal متاح.
   - تتضمنه إصدارات OpenClaw المعبأة الحالية بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافته يدويًا باستخدام الأوامر أعلاه.
2. سجّل الدخول (QR، على جهاز Gateway):
   - `openclaw channels login --channel zalouser`
   - امسح رمز QR باستخدام تطبيق Zalo على الهاتف المحمول.
3. فعّل القناة:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. أعد تشغيل Gateway (أو أكمل الإعداد).
5. يكون الوصول إلى الرسائل المباشرة مضبوطًا افتراضيًا على الاقتران؛ وافق على رمز الاقتران عند أول تواصل.

## ما هو

- يعمل بالكامل داخل العملية عبر `zca-js`.
- يستخدم مستمعي أحداث أصليين لاستقبال الرسائل الواردة.
- يرسل الردود مباشرة عبر JS API ‏(نص/وسائط/رابط).
- مصمم لحالات استخدام "الحساب الشخصي" حيث لا تكون Zalo Bot API متاحة.

## التسمية

معرّف القناة هو `zalouser` لتوضيح أن هذا التكامل يؤتمت **حساب مستخدم Zalo شخصي** (غير رسمي). ونحتفظ بالاسم `zalo` لتكامل رسمي محتمل مستقبلاً مع Zalo API.

## العثور على المعرّفات (الدليل)

استخدم CLI الخاص بالدليل لاكتشاف الجهات النظيرة/المجموعات ومعرّفاتها:

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## الحدود

- يُقسَّم النص الصادر إلى أجزاء بحوالي 2000 حرف (حدود عميل Zalo).
- يكون البث المتدفق محظورًا افتراضيًا.

## التحكم في الوصول (الرسائل المباشرة)

يدعم `channels.zalouser.dmPolicy` القيم التالية: `pairing | allowlist | open | disabled` (الافتراضي: `pairing`).

يقبل `channels.zalouser.allowFrom` معرّفات المستخدمين أو الأسماء. أثناء الإعداد، تُحوَّل الأسماء إلى معرّفات باستخدام البحث عن جهات الاتصال داخل العملية في plugin.

للموافقة:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## الوصول إلى المجموعات (اختياري)

- الافتراضي: `channels.zalouser.groupPolicy = "open"` ‏(المجموعات مسموح بها). استخدم `channels.defaults.groupPolicy` لتجاوز القيمة الافتراضية عند عدم ضبطها.
- للتقييد إلى قائمة سماح:
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups` ‏(يجب أن تكون المفاتيح معرّفات مجموعات مستقرة؛ وتُحوَّل الأسماء إلى معرّفات عند بدء التشغيل عندما يكون ذلك ممكنًا)
  - `channels.zalouser.groupAllowFrom` ‏(يتحكم في المرسلين داخل المجموعات المسموح بها الذين يمكنهم تشغيل البوت)
- لحظر جميع المجموعات: `channels.zalouser.groupPolicy = "disabled"`.
- يمكن لمعالج الإعداد المطالبة بقوائم سماح المجموعات.
- عند بدء التشغيل، يحوّل OpenClaw أسماء المجموعات/المستخدمين في قوائم السماح إلى معرّفات ويسجل هذا الربط.
- تكون مطابقة قائمة سماح المجموعات بحسب المعرّف فقط افتراضيًا. ويتم تجاهل الأسماء التي لم يتم حلها لأغراض التخويل ما لم يتم تمكين `channels.zalouser.dangerouslyAllowNameMatching: true`.
- يُعد `channels.zalouser.dangerouslyAllowNameMatching: true` وضع توافق احتياطي يعيد تمكين المطابقة باستخدام أسماء المجموعات القابلة للتغيير.
- إذا لم يتم ضبط `groupAllowFrom`، فإن وقت التشغيل يعود إلى `allowFrom` لفحوصات مرسلي المجموعات.
- تنطبق فحوصات المرسل على كل من رسائل المجموعات العادية وأوامر التحكم (على سبيل المثال `/new` و`/reset`).

مثال:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { allow: true },
        "Work Chat": { allow: true },
      },
    },
  },
}
```

### بوابة الإشارات في المجموعات

- يتحكم `channels.zalouser.groups.<group>.requireMention` فيما إذا كانت الردود في المجموعات تتطلب إشارة.
- ترتيب الحل: معرّف/اسم المجموعة المطابق تمامًا -> slug المجموعة المطبع -> `*` -> الافتراضي (`true`).
- ينطبق هذا على كل من المجموعات المدرجة في قائمة السماح ووضع المجموعات المفتوح.
- يمكن لأوامر التحكم المصرّح بها (على سبيل المثال `/new`) تجاوز بوابة الإشارات.
- عندما يتم تخطي رسالة مجموعة لأن الإشارة مطلوبة، يخزنها OpenClaw كسجل مجموعة معلّق ويضمّنها في رسالة المجموعة التالية التي تتم معالجتها.
- يكون حد سجل المجموعات افتراضيًا هو `messages.groupChat.historyLimit` (مع تراجع إلى `50`). ويمكنك تجاوزه لكل حساب عبر `channels.zalouser.historyLimit`.

مثال:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { allow: true, requireMention: true },
        "Work Chat": { allow: true, requireMention: false },
      },
    },
  },
}
```

## حسابات متعددة

ترتبط الحسابات بملفات `zalouser` الشخصية في حالة OpenClaw. مثال:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## الكتابة والتفاعلات وإقرارات التسليم

- يرسل OpenClaw حدث كتابة قبل إرسال الرد (بأفضل جهد ممكن).
- الإجراء `react` لتفاعلات الرسائل مدعوم لـ `zalouser` في إجراءات القنوات.
  - استخدم `remove: true` لإزالة رمز emoji تفاعلي محدد من رسالة.
  - دلالات التفاعلات: [Reactions](/tools/reactions)
- بالنسبة إلى الرسائل الواردة التي تتضمن بيانات تعريف الحدث، يرسل OpenClaw إقرارات بالتسليم والرؤية (بأفضل جهد ممكن).

## استكشاف الأخطاء وإصلاحها

**تسجيل الدخول لا يستمر:**

- `openclaw channels status --probe`
- أعد تسجيل الدخول: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**لم يتم حل اسم قائمة السماح/المجموعة:**

- استخدم المعرفات الرقمية في `allowFrom`/`groupAllowFrom`/`groups`، أو أسماء الأصدقاء/المجموعات المطابقة تمامًا.

**تمت الترقية من إعداد قديم قائم على CLI:**

- أزل أي افتراضات قديمة عن عملية `zca` خارجية.
- تعمل القناة الآن بالكامل داخل OpenClaw من دون ملفات CLI تنفيذية خارجية.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وبوابة الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
