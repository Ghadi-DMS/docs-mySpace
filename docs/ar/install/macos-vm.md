---
read_when:
    - أنت تريد عزل OpenClaw عن بيئة macOS الرئيسية لديك
    - أنت تريد تكامل iMessage ‏(BlueBubbles) داخل بيئة معزولة
    - أنت تريد بيئة macOS قابلة لإعادة التعيين ويمكنك استنساخها
    - أنت تريد مقارنة خيارات أجهزة macOS الافتراضية المحلية مقابل المستضافة
summary: شغّل OpenClaw داخل جهاز macOS VM معزول (محلي أو مستضاف) عندما تحتاج إلى العزل أو iMessage
title: أجهزة macOS الافتراضية
x-i18n:
    generated_at: "2026-04-05T12:47:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: b1f7c5691fd2686418ee25f2c38b1f9badd511daeef2906d21ad30fb523b013f
    source_path: install/macos-vm.md
    workflow: 15
---

# OpenClaw على أجهزة macOS الافتراضية (العزل)

## الإعداد الافتراضي الموصى به (لمعظم المستخدمين)

- **VPS صغير بنظام Linux** لتشغيل Gateway دائمًا وبتكلفة منخفضة. راجع [استضافة VPS](/vps).
- **عتاد مخصص** (Mac mini أو جهاز Linux) إذا كنت تريد تحكمًا كاملًا و**عنوان IP سكنيًا** لأتمتة المتصفح. تحظر كثير من المواقع عناوين IP الخاصة بمراكز البيانات، لذلك يعمل التصفح المحلي غالبًا بشكل أفضل.
- **إعداد هجين:** أبقِ Gateway على VPS رخيص، ووصل جهاز Mac لديك كـ **عقدة** عندما تحتاج إلى أتمتة المتصفح/واجهة المستخدم. راجع [العُقد](/nodes) و[Gateway remote](/gateway/remote).

استخدم جهاز macOS VM عندما تحتاج تحديدًا إلى إمكانات خاصة بـ macOS فقط (iMessage/BlueBubbles) أو عندما تريد عزلًا صارمًا عن جهاز Mac اليومي الخاص بك.

## خيارات أجهزة macOS الافتراضية

### جهاز VM محلي على جهاز Apple Silicon Mac لديك (Lume)

شغّل OpenClaw داخل جهاز macOS VM معزول على جهاز Apple Silicon Mac الحالي لديك باستخدام [Lume](https://cua.ai/docs/lume).

يمنحك هذا:

- بيئة macOS كاملة داخل عزل (ويبقى جهازك المضيف نظيفًا)
- دعم iMessage عبر BlueBubbles ‏(وهو أمر مستحيل على Linux/Windows)
- إعادة تعيين فورية عبر استنساخ الأجهزة الافتراضية
- من دون عتاد إضافي أو تكاليف سحابية

### موفرو أجهزة Mac المستضافة (السحابة)

إذا كنت تريد macOS في السحابة، فموفرو أجهزة Mac المستضافة يعملون أيضًا:

- [MacStadium](https://www.macstadium.com/) ‏(أجهزة Mac مستضافة)
- يعمل أيضًا مزودو macOS المستضافون الآخرون؛ اتبع وثائق VM + SSH الخاصة بهم

بمجرد أن يصبح لديك وصول SSH إلى جهاز macOS VM، تابع من الخطوة 6 أدناه.

---

## المسار السريع (Lume، للمستخدمين المتمرسين)

1. ثبّت Lume
2. ‏`lume create openclaw --os macos --ipsw latest`
3. أكمل Setup Assistant، وفعّل Remote Login ‏(SSH)
4. ‏`lume run openclaw --no-display`
5. اتصل عبر SSH، وثبّت OpenClaw، واضبط القنوات
6. انتهى

---

## ما الذي تحتاجه (Lume)

- جهاز Apple Silicon Mac ‏(M1/M2/M3/M4)
- macOS Sequoia أو أحدث على الجهاز المضيف
- نحو 60 GB من المساحة الحرة لكل جهاز VM
- نحو 20 دقيقة

---

## 1) تثبيت Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

إذا لم يكن `~/.local/bin` ضمن PATH لديك:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

تحقق:

```bash
lume --version
```

الوثائق: [تثبيت Lume](https://cua.ai/docs/lume/guide/getting-started/installation)

---

## 2) إنشاء جهاز macOS VM

```bash
lume create openclaw --os macos --ipsw latest
```

يقوم هذا بتنزيل macOS وإنشاء الجهاز الافتراضي. تفتح نافذة VNC تلقائيًا.

ملاحظة: قد يستغرق التنزيل بعض الوقت حسب سرعة اتصالك.

---

## 3) إكمال Setup Assistant

داخل نافذة VNC:

1. اختر اللغة والمنطقة
2. تخطَّ Apple ID ‏(أو سجّل الدخول إذا كنت تريد iMessage لاحقًا)
3. أنشئ حساب مستخدم (وتذكر اسم المستخدم وكلمة المرور)
4. تخطَّ كل الميزات الاختيارية

بعد اكتمال الإعداد، فعّل SSH:

1. افتح System Settings → General → Sharing
2. فعّل "Remote Login"

---

## 4) الحصول على عنوان IP الخاص بالجهاز الافتراضي

```bash
lume get openclaw
```

ابحث عن عنوان IP ‏(غالبًا `192.168.64.x`).

---

## 5) الاتصال عبر SSH إلى الجهاز الافتراضي

```bash
ssh youruser@192.168.64.X
```

استبدل `youruser` بالحساب الذي أنشأته، واستبدل عنوان IP بعنوان جهازك الافتراضي.

---

## 6) تثبيت OpenClaw

داخل الجهاز الافتراضي:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

اتبع مطالبات onboarding لإعداد موفّر النموذج لديك (Anthropic أو OpenAI أو غيرهما).

---

## 7) إعداد القنوات

حرّر ملف الإعدادات:

```bash
nano ~/.openclaw/openclaw.json
```

أضف قنواتك:

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
  },
}
```

ثم سجّل الدخول إلى WhatsApp ‏(امسح QR):

```bash
openclaw channels login
```

---

## 8) تشغيل الجهاز الافتراضي من دون واجهة

أوقف الجهاز الافتراضي ثم أعد تشغيله من دون عرض:

```bash
lume stop openclaw
lume run openclaw --no-display
```

سيعمل الجهاز الافتراضي في الخلفية. وسيُبقي daemon الخاص بـ OpenClaw البوابة قيد التشغيل.

للتحقق من الحالة:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

---

## إضافة: تكامل iMessage

هذه هي الميزة الأهم للتشغيل على macOS. استخدم [BlueBubbles](https://bluebubbles.app) لإضافة iMessage إلى OpenClaw.

داخل الجهاز الافتراضي:

1. نزّل BlueBubbles من bluebubbles.app
2. سجّل الدخول باستخدام Apple ID
3. فعّل Web API واضبط كلمة مرور
4. وجّه webhooks الخاصة بـ BlueBubbles إلى بوابتك (مثال: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`)

أضف إلى إعدادات OpenClaw:

```json5
{
  channels: {
    bluebubbles: {
      serverUrl: "http://localhost:1234",
      password: "your-api-password",
      webhookPath: "/bluebubbles-webhook",
    },
  },
}
```

أعد تشغيل البوابة. والآن يمكن لوكيلك إرسال iMessages واستقبالها.

تفاصيل الإعداد الكاملة: [قناة BlueBubbles](/channels/bluebubbles)

---

## احفظ صورة ذهبية

قبل تخصيص المزيد، التقط Snapshot لحالتك النظيفة:

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

أعد التعيين في أي وقت:

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

---

## التشغيل على مدار الساعة

أبقِ الجهاز الافتراضي قيد التشغيل عبر:

- إبقاء جهاز Mac موصولًا بالكهرباء
- تعطيل السكون في System Settings → Energy Saver
- استخدام `caffeinate` عند الحاجة

ولتشغيل دائم حقيقي، فكّر في Mac mini مخصص أو VPS صغير. راجع [استضافة VPS](/vps).

---

## استكشاف الأخطاء وإصلاحها

| المشكلة | الحل |
| ------- | ---- |
| تعذر الاتصال عبر SSH إلى الجهاز الافتراضي | تحقق من أن "Remote Login" مفعّل في System Settings داخل الجهاز الافتراضي |
| لا يظهر عنوان IP الخاص بالجهاز الافتراضي | انتظر حتى يكتمل إقلاع الجهاز الافتراضي، ثم شغّل `lume get openclaw` مرة أخرى |
| الأمر Lume غير موجود | أضف `~/.local/bin` إلى PATH |
| لا يتم مسح QR الخاص بـ WhatsApp | تأكد من أنك مسجّل الدخول داخل الجهاز الافتراضي (وليس المضيف) عند تشغيل `openclaw channels login` |

---

## وثائق ذات صلة

- [استضافة VPS](/vps)
- [العُقد](/nodes)
- [Gateway remote](/gateway/remote)
- [قناة BlueBubbles](/channels/bluebubbles)
- [البدء السريع مع Lume](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [مرجع CLI الخاص بـ Lume](https://cua.ai/docs/lume/reference/cli-reference)
- [إعداد VM غير تفاعلي](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup) (متقدم)
- [العزل باستخدام Docker](/install/docker) (نهج عزل بديل)
