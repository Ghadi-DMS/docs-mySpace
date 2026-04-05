---
read_when:
    - تحتاج إلى طريقة تثبيت غير البدء السريع في Getting Started
    - تريد النشر على منصة سحابية
    - تحتاج إلى التحديث أو الترحيل أو إلغاء التثبيت
summary: تثبيت OpenClaw — script التثبيت، وnpm/pnpm/bun، ومن المصدر، وDocker، وغير ذلك
title: التثبيت
x-i18n:
    generated_at: "2026-04-05T12:47:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: eca17c76a2a66166b3d8cda9dc3144ab920d30ad0ed2a220eb9389d7a383ba5d
    source_path: install/index.md
    workflow: 15
---

# التثبيت

## الموصى به: script التثبيت

أسرع طريقة للتثبيت. يكتشف نظام التشغيل لديك، ويثبت Node عند الحاجة، ويثبت OpenClaw، ثم يشغّل onboarding.

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
</Tabs>

للتثبيت من دون تشغيل onboarding:

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

للاطلاع على جميع العلامات وخيارات CI/الأتمتة، راجع [الجوانب الداخلية للمثبّت](/install/installer).

## متطلبات النظام

- **Node 24** ‏(موصى به) أو Node 22.14+ — يتولى script التثبيت ذلك تلقائيًا
- **macOS أو Linux أو Windows** — يدعم كلًا من Windows الأصلي وWSL2؛ ويكون WSL2 أكثر استقرارًا. راجع [Windows](/platforms/windows).
- يلزم `pnpm` فقط إذا كنت ستبني من المصدر

## طرق التثبيت البديلة

### مثبّت local prefix ‏(`install-cli.sh`)

استخدمه عندما تريد الاحتفاظ بـ OpenClaw وNode تحت local prefix مثل
`~/.openclaw`، من دون الاعتماد على تثبيت Node على مستوى النظام:

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

يدعم هذا المسار تثبيتات npm افتراضيًا، بالإضافة إلى تثبيتات نسخة git المحلية ضمن
تدفق prefix نفسه. المرجع الكامل: [الجوانب الداخلية للمثبّت](/install/installer#install-clish).

### npm أو pnpm أو bun

إذا كنت تدير Node بنفسك بالفعل:

<Tabs>
  <Tab title="npm">
    ```bash
    npm install -g openclaw@latest
    openclaw onboard --install-daemon
    ```
  </Tab>
  <Tab title="pnpm">
    ```bash
    pnpm add -g openclaw@latest
    pnpm approve-builds -g
    openclaw onboard --install-daemon
    ```

    <Note>
    يتطلب pnpm موافقة صريحة على الحزم التي تحتوي على scripts بناء. شغّل `pnpm approve-builds -g` بعد أول تثبيت.
    </Note>

  </Tab>
  <Tab title="bun">
    ```bash
    bun add -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    يتم دعم Bun لمسار تثبيت CLI العام فقط. أما بالنسبة إلى وقت تشغيل Gateway، فما يزال Node هو بيئة daemon الموصى بها.
    </Note>

  </Tab>
</Tabs>

<Accordion title="استكشاف الأخطاء وإصلاحها: أخطاء بناء sharp ‏(npm)">
  إذا فشل `sharp` بسبب libvips مثبّتة على مستوى النظام:

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

</Accordion>

### من المصدر

للمساهمين أو لأي شخص يريد التشغيل من نسخة محلية:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install && pnpm ui:build && pnpm build
pnpm link --global
openclaw onboard --install-daemon
```

أو تخطَّ الربط واستخدم `pnpm openclaw ...` من داخل المستودع. راجع [الإعداد](/start/setup) للحصول على تدفقات التطوير الكاملة.

### التثبيت من GitHub main

```bash
npm install -g github:openclaw/openclaw#main
```

### الحاويات ومديرو الحزم

<CardGroup cols={2}>
  <Card title="Docker" href="/install/docker" icon="container">
    عمليات نشر معبأة أو بلا واجهة.
  </Card>
  <Card title="Podman" href="/install/podman" icon="container">
    بديل حاويات rootless لـ Docker.
  </Card>
  <Card title="Nix" href="/install/nix" icon="snowflake">
    تثبيت تصريحي عبر Nix flake.
  </Card>
  <Card title="Ansible" href="/install/ansible" icon="server">
    تجهيز آلي للأساطيل.
  </Card>
  <Card title="Bun" href="/install/bun" icon="zap">
    استخدام CLI فقط عبر وقت تشغيل Bun.
  </Card>
</CardGroup>

## التحقق من التثبيت

```bash
openclaw --version      # تأكيد توفر CLI
openclaw doctor         # التحقق من مشكلات التكوين
openclaw gateway status # التحقق من أن Gateway تعمل
```

إذا كنت تريد بدء تشغيل مُدارًا بعد التثبيت:

- macOS: ‏LaunchAgent عبر `openclaw onboard --install-daemon` أو `openclaw gateway install`
- Linux/WSL2: خدمة مستخدم systemd عبر الأوامر نفسها
- Windows الأصلي: ‏Scheduled Task أولًا، مع بديل عنصر تسجيل دخول لكل مستخدم في Startup folder إذا تم رفض إنشاء المهمة

## الاستضافة والنشر

انشر OpenClaw على خادم سحابي أو VPS:

<CardGroup cols={3}>
  <Card title="VPS" href="/vps">أي Linux VPS</Card>
  <Card title="Docker VM" href="/install/docker-vm-runtime">خطوات Docker المشتركة</Card>
  <Card title="Kubernetes" href="/install/kubernetes">K8s</Card>
  <Card title="Fly.io" href="/install/fly">Fly.io</Card>
  <Card title="Hetzner" href="/install/hetzner">Hetzner</Card>
  <Card title="GCP" href="/install/gcp">Google Cloud</Card>
  <Card title="Azure" href="/install/azure">Azure</Card>
  <Card title="Railway" href="/install/railway">Railway</Card>
  <Card title="Render" href="/install/render">Render</Card>
  <Card title="Northflank" href="/install/northflank">Northflank</Card>
</CardGroup>

## التحديث أو الترحيل أو إلغاء التثبيت

<CardGroup cols={3}>
  <Card title="التحديث" href="/install/updating" icon="refresh-cw">
    حافظ على تحديث OpenClaw.
  </Card>
  <Card title="الترحيل" href="/install/migrating" icon="arrow-right">
    الانتقال إلى جهاز جديد.
  </Card>
  <Card title="إلغاء التثبيت" href="/install/uninstall" icon="trash-2">
    إزالة OpenClaw بالكامل.
  </Card>
</CardGroup>

## استكشاف الأخطاء وإصلاحها: تعذر العثور على `openclaw`

إذا نجح التثبيت لكن تعذر العثور على `openclaw` في طرفيتك:

```bash
node -v           # هل Node مثبت؟
npm prefix -g     # أين توجد الحزم العامة؟
echo "$PATH"      # هل يوجد دليل bin العام في PATH؟
```

إذا لم يكن `$(npm prefix -g)/bin` موجودًا في `$PATH` لديك، فأضفه إلى ملف بدء shell ‏(`~/.zshrc` أو `~/.bashrc`):

```bash
export PATH="$(npm prefix -g)/bin:$PATH"
```

ثم افتح طرفية جديدة. راجع [إعداد Node](/install/node) لمزيد من التفاصيل.
