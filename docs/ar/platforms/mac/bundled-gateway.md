---
read_when:
    - تغليف OpenClaw.app
    - تصحيح أخطاء خدمة gateway الخاصة بـ launchd على macOS
    - تثبيت CLI الخاص بـ gateway على macOS
summary: وقت تشغيل Gateway على macOS ‏(خدمة launchd خارجية)
title: Gateway على macOS
x-i18n:
    generated_at: "2026-04-05T12:49:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 69e41528b35d69c13608cb9a34b39a7f02e1134204d1b496cbdd191798f39607
    source_path: platforms/mac/bundled-gateway.md
    workflow: 15
---

# Gateway على macOS ‏(launchd خارجي)

لم يعد OpenClaw.app يضم Node/Bun أو وقت تشغيل Gateway. يتوقع تطبيق macOS
وجود تثبيت **خارجي** لـ CLI ‏`openclaw`، ولا يشغّل Gateway كعملية
تابعة، بل يدير خدمة launchd لكل مستخدم للإبقاء على Gateway
قيد التشغيل (أو يتصل بـ Gateway محلية موجودة بالفعل إذا كانت تعمل مسبقًا).

## تثبيت CLI ‏(مطلوب للوضع المحلي)

يُعد Node 24 وقت التشغيل الافتراضي على Mac. وما زال Node 22 LTS، حاليًا `22.14+`، يعمل من أجل التوافق. ثم ثبّت `openclaw` عالميًا:

```bash
npm install -g openclaw@<version>
```

يقوم زر **Install CLI** في تطبيق macOS بتشغيل تدفق التثبيت العام نفسه الذي
يستخدمه التطبيق داخليًا: فهو يفضل npm أولًا، ثم pnpm، ثم bun إذا كان
مدير الحزم الوحيد المكتشف. ويظل Node هو وقت تشغيل Gateway الموصى به.

## Launchd ‏(Gateway كـ LaunchAgent)

التسمية:

- `ai.openclaw.gateway` ‏(أو `ai.openclaw.<profile>`؛ وقد تبقى التسمية القديمة `com.openclaw.*`)

موقع Plist ‏(لكل مستخدم):

- `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
  ‏(أو `~/Library/LaunchAgents/ai.openclaw.<profile>.plist`)

المدير:

- يملك تطبيق macOS تثبيت/تحديث LaunchAgent في الوضع المحلي.
- يمكن أيضًا لـ CLI تثبيته: `openclaw gateway install`.

السلوك:

- يقوم خيار “OpenClaw Active” بتمكين/تعطيل LaunchAgent.
- لا يؤدي إنهاء التطبيق إلى إيقاف gateway ‏(إذ يبقيها launchd قيد التشغيل).
- إذا كانت Gateway تعمل بالفعل على المنفذ المكوَّن، فإن التطبيق يتصل
  بها بدلًا من بدء واحدة جديدة.

التسجيل:

- stdout/err الخاص بـ launchd: ‏`/tmp/openclaw/openclaw-gateway.log`

## توافق الإصدارات

يتحقق تطبيق macOS من إصدار gateway مقارنة بإصداره. وإذا كانا
غير متوافقين، فحدّث CLI العام ليتوافق مع إصدار التطبيق.

## فحص سريع

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

ثم:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```
