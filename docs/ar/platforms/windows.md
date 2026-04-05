---
read_when:
    - تثبيت OpenClaw على Windows
    - الاختيار بين Windows الأصلية وWSL2
    - البحث عن حالة التطبيق المرافق لـ Windows
summary: 'دعم Windows: مسارات التثبيت الأصلية وWSL2، وdaemon، والمحاذير الحالية'
title: Windows
x-i18n:
    generated_at: "2026-04-05T12:50:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7d9819206bdd65cf03519c1bc73ed0c7889b0ab842215ea94343262300adfd14
    source_path: platforms/windows.md
    workflow: 15
---

# Windows

يدعم OpenClaw كلاً من **Windows الأصلية** و**WSL2**. ويُعد WSL2 المسار الأكثر
استقرارًا والموصى به للحصول على التجربة الكاملة — حيث تعمل CLI وGateway
والأدوات داخل Linux مع توافق كامل. أما Windows الأصلية فتعامل بشكل جيد
مع CLI الأساسية واستخدامات Gateway، مع بعض المحاذير المذكورة أدناه.

من المخطط توفير تطبيقات مرافقة أصلية لـ Windows.

## WSL2 (موصى به)

- [Getting Started](/ar/start/getting-started) (استخدمه داخل WSL)
- [Install & updates](/install/updating)
- دليل WSL2 الرسمي (Microsoft): [https://learn.microsoft.com/windows/wsl/install](https://learn.microsoft.com/windows/wsl/install)

## حالة Windows الأصلية

تتحسن تدفقات CLI الأصلية على Windows، لكن WSL2 ما تزال المسار الموصى به.

ما الذي يعمل جيدًا على Windows الأصلية اليوم:

- مثبّت الموقع عبر `install.ps1`
- استخدام CLI المحلي مثل `openclaw --version` و`openclaw doctor` و`openclaw plugins list --json`
- اختبارات smoke المحلية للوكيل/المزوّد المضمّنين مثل:

```powershell
openclaw agent --local --agent main --thinking low -m "Reply with exactly WINDOWS-HATCH-OK."
```

المحاذير الحالية:

- ما يزال `openclaw onboard --non-interactive` يتوقع gateway محلية قابلة للوصول ما لم تمرر `--skip-health`
- يحاول `openclaw onboard --non-interactive --install-daemon` و`openclaw gateway install` استخدام Windows Scheduled Tasks أولًا
- إذا تم رفض إنشاء Scheduled Task، يعود OpenClaw إلى عنصر تسجيل دخول لكل مستخدم في مجلد Startup ويبدأ gateway فورًا
- إذا تعطلت `schtasks` نفسها أو توقفت عن الاستجابة، يوقف OpenClaw هذا المسار بسرعة الآن ويعود إلى الرجوع الاحتياطي بدلًا من التعليق إلى الأبد
- ما تزال Scheduled Tasks هي المفضلة عندما تكون متاحة لأنها توفّر حالة مشرف أفضل

إذا كنت تريد CLI الأصلية فقط، من دون تثبيت خدمة gateway، فاستخدم أحد هذين:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

إذا كنت تريد بدءًا مُدارًا على Windows الأصلية:

```powershell
openclaw gateway install
openclaw gateway status --json
```

إذا تم حظر إنشاء Scheduled Task، فسيظل وضع الخدمة الاحتياطية يبدأ تلقائيًا بعد تسجيل الدخول عبر مجلد Startup الخاص بالمستخدم الحالي.

## Gateway

- [Gateway runbook](/gateway)
- [Configuration](/gateway/configuration)

## تثبيت خدمة Gateway ‏(CLI)

داخل WSL2:

```
openclaw onboard --install-daemon
```

أو:

```
openclaw gateway install
```

أو:

```
openclaw configure
```

اختر **Gateway service** عند مطالبتك.

الإصلاح/الترحيل:

```
openclaw doctor
```

## التشغيل التلقائي لـ Gateway قبل تسجيل الدخول إلى Windows

بالنسبة إلى الإعدادات بدون واجهة، تأكد من أن سلسلة الإقلاع الكاملة تعمل حتى عندما لا يسجّل أحد الدخول إلى
Windows.

### 1) إبقاء خدمات المستخدم تعمل من دون تسجيل الدخول

داخل WSL:

```bash
sudo loginctl enable-linger "$(whoami)"
```

### 2) تثبيت خدمة مستخدم OpenClaw gateway

داخل WSL:

```bash
openclaw gateway install
```

### 3) بدء WSL تلقائيًا عند إقلاع Windows

في PowerShell بصلاحية المسؤول:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec /bin/true" /sc onstart /ru SYSTEM
```

استبدل `Ubuntu` باسم التوزيعة لديك من:

```powershell
wsl --list --verbose
```

### التحقق من سلسلة بدء التشغيل

بعد إعادة التشغيل (وقبل تسجيل الدخول إلى Windows)، تحقق من داخل WSL:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## متقدم: كشف خدمات WSL عبر LAN ‏(portproxy)

تملك WSL شبكتها الافتراضية الخاصة. إذا كان جهاز آخر يحتاج إلى الوصول إلى خدمة
تعمل **داخل WSL** ‏(SSH أو خادم TTS محلي أو Gateway)، فيجب
تمرير منفذ من Windows إلى عنوان IP الحالي الخاص بـ WSL. ويتغير عنوان IP الخاص بـ WSL بعد إعادة التشغيل،
لذا قد تحتاج إلى تحديث قاعدة التمرير.

مثال (PowerShell **بصلاحية المسؤول**):

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP not found." }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort
```

اسمح بالمنفذ عبر Windows Firewall ‏(مرة واحدة):

```powershell
New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

حدّث portproxy بعد إعادة تشغيل WSL:

```powershell
netsh interface portproxy delete v4tov4 listenport=$ListenPort listenaddress=0.0.0.0 | Out-Null
netsh interface portproxy add v4tov4 listenport=$ListenPort listenaddress=0.0.0.0 `
  connectaddress=$WslIp connectport=$TargetPort | Out-Null
```

ملاحظات:

- يستهدف SSH من جهاز آخر **عنوان IP الخاص بمضيف Windows** ‏(مثال: `ssh user@windows-host -p 2222`).
- يجب أن تشير العقد البعيدة إلى عنوان URL لـ Gateway **يمكن الوصول إليه** (وليس `127.0.0.1`)؛ استخدم
  `openclaw status --all` للتأكيد.
- استخدم `listenaddress=0.0.0.0` للوصول عبر LAN؛ أما `127.0.0.1` فيبقيه محليًا فقط.
- إذا كنت تريد تنفيذ ذلك تلقائيًا، فسجّل Scheduled Task لتشغيل
  خطوة التحديث عند تسجيل الدخول.

## تثبيت WSL2 خطوة بخطوة

### 1) تثبيت WSL2 + Ubuntu

افتح PowerShell ‏(المسؤول):

```powershell
wsl --install
# Or pick a distro explicitly:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

أعد التشغيل إذا طلب Windows ذلك.

### 2) تمكين systemd ‏(مطلوب لتثبيت gateway)

في طرفية WSL لديك:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

ثم من PowerShell:

```powershell
wsl --shutdown
```

أعد فتح Ubuntu، ثم تحقق:

```bash
systemctl --user status
```

### 3) تثبيت OpenClaw ‏(داخل WSL)

اتبع تدفق Linux Getting Started داخل WSL:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
openclaw onboard
```

الدليل الكامل: [Getting Started](/ar/start/getting-started)

## التطبيق المرافق لـ Windows

ليس لدينا تطبيق مرافق لـ Windows حتى الآن. المساهمات مرحب بها إذا كنت تريد
المساعدة في تحقيق ذلك.
