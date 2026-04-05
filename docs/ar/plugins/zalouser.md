---
read_when:
    - تريد دعم Zalo Personal ‏(غير الرسمي) في OpenClaw
    - أنت تضبط أو تطور plugin الخاصة بـ zalouser
summary: 'Plugin خاص بـ Zalo Personal: تسجيل دخول QR + مراسلة عبر `zca-js` الأصلي (تثبيت plugin + تكوين القناة + الأداة)'
title: Plugin الخاص بـ Zalo Personal
x-i18n:
    generated_at: "2026-04-05T12:52:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3218c3ee34f36466d952aec1b479d451a6235c7c46918beb28698234a7fd0968
    source_path: plugins/zalouser.md
    workflow: 15
---

# Zalo Personal (plugin)

دعم Zalo Personal لـ OpenClaw عبر plugin، باستخدام `zca-js` الأصلي لأتمتة حساب مستخدم Zalo عادي.

> **تحذير:** قد تؤدي الأتمتة غير الرسمية إلى تعليق الحساب/حظره. استخدمها على مسؤوليتك الخاصة.

## التسمية

معرّف القناة هو `zalouser` لتوضيح أن هذه الأتمتة تخص **حساب مستخدم Zalo شخصي** (غير رسمي). ونحتفظ بالاسم `zalo` لتكامل رسمي محتمل مع Zalo API في المستقبل.

## مكان التشغيل

تعمل هذه الـ plugin **داخل عملية Gateway**.

إذا كنت تستخدم Gateway بعيدة، فقم بتثبيتها/تكوينها على **الجهاز الذي يشغّل Gateway**، ثم أعد تشغيل Gateway.

لا حاجة إلى ثنائي CLI خارجي لـ `zca`/`openzca`.

## التثبيت

### الخيار A: التثبيت من npm

```bash
openclaw plugins install @openclaw/zalouser
```

أعد تشغيل Gateway بعد ذلك.

### الخيار B: التثبيت من مجلد محلي (للتطوير)

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

أعد تشغيل Gateway بعد ذلك.

## التكوين

يعيش تكوين القناة تحت `channels.zalouser` ‏(وليس `plugins.entries.*`):

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

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hello from OpenClaw"
openclaw directory peers list --channel zalouser --query "name"
```

## أداة الوكيل

اسم الأداة: `zalouser`

الإجراءات: `send`, `image`, `link`, `friends`, `groups`, `me`, `status`

كما تدعم إجراءات رسائل القناة أيضًا `react` لتفاعلات الرسائل.
