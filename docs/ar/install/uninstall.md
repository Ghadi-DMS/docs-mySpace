---
read_when:
    - تريد إزالة OpenClaw من جهاز
    - لا تزال خدمة gateway تعمل بعد إلغاء التثبيت
summary: إزالة OpenClaw بالكامل (CLI والخدمة والحالة ومساحة العمل)
title: إلغاء التثبيت
x-i18n:
    generated_at: "2026-04-05T12:48:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 34c7d3e4ad17333439048dfda739fc27db47e7f9e4212fe17db0e4eb3d3ab258
    source_path: install/uninstall.md
    workflow: 15
---

# إلغاء التثبيت

هناك مساران:

- **المسار السهل** إذا كانت `openclaw` لا تزال مثبتة.
- **إزالة الخدمة يدويًا** إذا اختفى CLI لكن الخدمة ما تزال تعمل.

## المسار السهل (CLI ما تزال مثبتة)

الموصى به: استخدم أداة إلغاء التثبيت المضمّنة:

```bash
openclaw uninstall
```

بشكل غير تفاعلي (للأتمتة / npx):

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

الخطوات اليدوية (النتيجة نفسها):

1. أوقف خدمة gateway:

```bash
openclaw gateway stop
```

2. أزل خدمة gateway ‏(launchd/systemd/schtasks):

```bash
openclaw gateway uninstall
```

3. احذف الحالة + التكوين:

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

إذا كنت قد ضبطت `OPENCLAW_CONFIG_PATH` على موقع مخصص خارج دليل الحالة، فاحذف ذلك الملف أيضًا.

4. احذف مساحة العمل الخاصة بك (اختياري، يزيل ملفات الوكيل):

```bash
rm -rf ~/.openclaw/workspace
```

5. أزل تثبيت CLI ‏(اختر ما استخدمته):

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. إذا كنت قد ثبّت تطبيق macOS:

```bash
rm -rf /Applications/OpenClaw.app
```

ملاحظات:

- إذا كنت قد استخدمت profiles ‏(`--profile` / `OPENCLAW_PROFILE`)، فأعد تنفيذ الخطوة 3 لكل دليل حالة (القيم الافتراضية هي `~/.openclaw-<profile>`).
- في الوضع البعيد، يعيش دليل الحالة على **مضيف gateway**، لذا شغّل الخطوات 1-4 هناك أيضًا.

## إزالة الخدمة يدويًا (CLI غير مثبتة)

استخدم هذا إذا استمرت خدمة gateway في العمل لكن `openclaw` غير موجودة.

### macOS ‏(launchd)

تكون التسمية الافتراضية هي `ai.openclaw.gateway` ‏(أو `ai.openclaw.<profile>`؛ وقد تظل التسمية القديمة `com.openclaw.*` موجودة):

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

إذا كنت قد استخدمت profile، فاستبدل التسمية واسم plist بالقيمة `ai.openclaw.<profile>`. وأزل أي ملفات plist قديمة من نوع `com.openclaw.*` إذا كانت موجودة.

### Linux ‏(وحدة systemd للمستخدم)

اسم الوحدة الافتراضي هو `openclaw-gateway.service` ‏(أو `openclaw-gateway-<profile>.service`):

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows ‏(Scheduled Task)

اسم المهمة الافتراضي هو `OpenClaw Gateway` ‏(أو `OpenClaw Gateway (<profile>)`).
يعيش script المهمة تحت دليل الحالة الخاص بك.

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd"
```

إذا كنت قد استخدمت profile، فاحذف اسم المهمة المطابق والملف `~\.openclaw-<profile>\gateway.cmd`.

## التثبيت العادي مقابل نسخة المصدر

### التثبيت العادي (`install.sh` / npm / pnpm / bun)

إذا كنت قد استخدمت `https://openclaw.ai/install.sh` أو `install.ps1`، فقد تم تثبيت CLI باستخدام `npm install -g openclaw@latest`.
أزله باستخدام `npm rm -g openclaw` ‏(أو `pnpm remove -g` / `bun remove -g` إذا ثبّتّه بهذه الطريقة).

### نسخة المصدر (`git clone`)

إذا كنت تشغّل التطبيق من نسخة مستودع (`git clone` + `openclaw ...` / `bun run openclaw ...`):

1. أزل خدمة gateway **قبل** حذف المستودع (استخدم المسار السهل أعلاه أو إزالة الخدمة يدويًا).
2. احذف دليل المستودع.
3. أزل الحالة + مساحة العمل كما هو موضح أعلاه.
