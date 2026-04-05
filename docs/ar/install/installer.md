---
read_when:
    - تريد فهم `openclaw.ai/install.sh`
    - تريد أتمتة عمليات التثبيت (CI / headless)
    - تريد التثبيت من GitHub checkout
summary: كيفية عمل scripts التثبيت (`install.sh` و`install-cli.sh` و`install.ps1`)، والعلامات، والأتمتة
title: الآليات الداخلية لبرنامج التثبيت
x-i18n:
    generated_at: "2026-04-05T12:48:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: eced891572b8825b1f8a26ccc9d105ae8a38bd8ad89baef2f1927e27d4619e04
    source_path: install/installer.md
    workflow: 15
---

# الآليات الداخلية لبرنامج التثبيت

يشحن OpenClaw ثلاثة scripts للتثبيت، يتم تقديمها من `openclaw.ai`.

| Script                             | المنصة               | ما الذي يفعله                                                                                                   |
| ---------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| [`install.sh`](#installsh)         | macOS / Linux / WSL  | يثبت Node عند الحاجة، ويثبت OpenClaw عبر npm (افتراضيًا) أو git، ويمكنه تشغيل الإعداد التفاعلي.                   |
| [`install-cli.sh`](#install-clish) | macOS / Linux / WSL  | يثبت Node + OpenClaw داخل بادئة محلية (`~/.openclaw`) باستخدام npm أو أوضاع git checkout. لا يتطلب root. |
| [`install.ps1`](#installps1)       | Windows (PowerShell) | يثبت Node عند الحاجة، ويثبت OpenClaw عبر npm (افتراضيًا) أو git، ويمكنه تشغيل الإعداد التفاعلي.                   |

## أوامر سريعة

<Tabs>
  <Tab title="install.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install-cli.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install.ps1">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag beta -NoOnboard -DryRun
    ```

  </Tab>
</Tabs>

<Note>
إذا نجح التثبيت لكن لم يتم العثور على `openclaw` في طرفية جديدة، فراجع [استكشاف أخطاء Node.js وإصلاحها](/install/node#troubleshooting).
</Note>

---

<a id="installsh"></a>

## install.sh

<Tip>
موصى به لمعظم عمليات التثبيت التفاعلية على macOS/Linux/WSL.
</Tip>

### التدفق (install.sh)

<Steps>
  <Step title="اكتشاف نظام التشغيل">
    يدعم macOS وLinux (بما في ذلك WSL). وإذا تم اكتشاف macOS، فسيقوم بتثبيت Homebrew إذا كان مفقودًا.
  </Step>
  <Step title="ضمان Node.js 24 افتراضيًا">
    يتحقق من إصدار Node ويثبت Node 24 إذا لزم الأمر (Homebrew على macOS، وscripts إعداد NodeSource على Linux apt/dnf/yum). وما زال OpenClaw يدعم Node 22 LTS، حاليًا `22.14+`، من أجل التوافق.
  </Step>
  <Step title="ضمان Git">
    يثبت Git إذا كان مفقودًا.
  </Step>
  <Step title="تثبيت OpenClaw">
    - طريقة `npm` (الافتراضية): تثبيت npm عالمي
    - طريقة `git`: استنساخ/تحديث المستودع، وتثبيت التبعيات باستخدام pnpm، والبناء، ثم تثبيت wrapper في `~/.local/bin/openclaw`
  </Step>
  <Step title="مهام ما بعد التثبيت">
    - يحدّث خدمة gateway المحمّلة على أفضل جهد (`openclaw gateway install --force`، ثم إعادة التشغيل)
    - يشغّل `openclaw doctor --non-interactive` عند الترقيات وتثبيتات git (على أفضل جهد)
    - يحاول تنفيذ الإعداد التفاعلي عند الاقتضاء (توفر TTY، وعدم تعطيل onboarding، واجتياز فحوصات bootstrap/config)
    - يضبط `SHARP_IGNORE_GLOBAL_LIBVIPS=1` افتراضيًا
  </Step>
</Steps>

### اكتشاف source checkout

إذا تم تشغيله داخل OpenClaw checkout (`package.json` + `pnpm-workspace.yaml`)، فإن script يعرض:

- استخدام checkout (`git`)، أو
- استخدام التثبيت العالمي (`npm`)

إذا لم تكن هناك TTY متاحة ولم يتم تحديد طريقة التثبيت، فسيستخدم `npm` افتراضيًا ويعرض تحذيرًا.

يخرج script بالرمز `2` عند اختيار طريقة غير صالحة أو قيم `--install-method` غير صالحة.

### أمثلة (install.sh)

<Tabs>
  <Tab title="الافتراضي">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="تخطي الإعداد التفاعلي">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="تثبيت git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```
  </Tab>
  <Tab title="GitHub main عبر npm">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --version main
    ```
  </Tab>
  <Tab title="تشغيل تجريبي">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --dry-run
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="مرجع العلامات">

| العلامة                                | الوصف                                                       |
| ------------------------------------- | ---------------------------------------------------------- |
| `--install-method npm\|git`           | اختيار طريقة التثبيت (الافتراضي: `npm`). الاسم المستعار: `--method`  |
| `--npm`                               | اختصار لطريقة npm                                    |
| `--git`                               | اختصار لطريقة git. الاسم المستعار: `--github`                 |
| `--version <version\|dist-tag\|spec>` | إصدار npm أو dist-tag أو package spec (الافتراضي: `latest`) |
| `--beta`                              | استخدام beta dist-tag إذا كانت متاحة، وإلا الرجوع إلى `latest`  |
| `--git-dir <path>`                    | دليل checkout (الافتراضي: `~/openclaw`). الاسم المستعار: `--dir` |
| `--no-git-update`                     | تخطي `git pull` لـ checkout الموجودة                      |
| `--no-prompt`                         | تعطيل prompts                                            |
| `--no-onboard`                        | تخطي onboarding                                            |
| `--onboard`                           | تمكين onboarding                                          |
| `--dry-run`                           | طباعة الإجراءات من دون تطبيق التغييرات                     |
| `--verbose`                           | تمكين إخراج التصحيح (`set -x`، وسجلات npm بمستوى notice)      |
| `--help`                              | عرض الاستخدام (`-h`)                                          |

  </Accordion>

  <Accordion title="مرجع متغيرات البيئة">

| المتغير                                                | الوصف                                   |
| ------------------------------------------------------- | --------------------------------------------- |
| `OPENCLAW_INSTALL_METHOD=git\|npm`                      | طريقة التثبيت                                |
| `OPENCLAW_VERSION=latest\|next\|main\|<semver>\|<spec>` | إصدار npm أو dist-tag أو package spec        |
| `OPENCLAW_BETA=0\|1`                                    | استخدام beta إذا كانت متاحة                         |
| `OPENCLAW_GIT_DIR=<path>`                               | دليل checkout                            |
| `OPENCLAW_GIT_UPDATE=0\|1`                              | تبديل تحديثات git                            |
| `OPENCLAW_NO_PROMPT=1`                                  | تعطيل prompts                               |
| `OPENCLAW_NO_ONBOARD=1`                                 | تخطي onboarding                               |
| `OPENCLAW_DRY_RUN=1`                                    | وضع التشغيل التجريبي                                  |
| `OPENCLAW_VERBOSE=1`                                    | وضع التصحيح                                    |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice`             | مستوى سجل npm                                 |
| `SHARP_IGNORE_GLOBAL_LIBVIPS=0\|1`                      | التحكم في سلوك sharp/libvips (الافتراضي: `1`) |

  </Accordion>
</AccordionGroup>

---

<a id="install-clish"></a>

## install-cli.sh

<Info>
مصمم للبيئات التي تريد فيها كل شيء تحت بادئة محلية
(الافتراضي `~/.openclaw`) ومن دون تبعية Node على مستوى النظام. يدعم تثبيتات npm
افتراضيًا، بالإضافة إلى تثبيتات git-checkout ضمن تدفق البادئة نفسه.
</Info>

### التدفق (install-cli.sh)

<Steps>
  <Step title="تثبيت runtime محلي لـ Node">
    ينزّل tarball مثبتًا لإصدار Node LTS مدعوم (الإصدار مضمّن في script ويتم تحديثه بشكل مستقل) إلى `<prefix>/tools/node-v<version>` ويتحقق من SHA-256.
  </Step>
  <Step title="ضمان Git">
    إذا كان Git مفقودًا، فإنه يحاول تثبيته عبر apt/dnf/yum على Linux أو Homebrew على macOS.
  </Step>
  <Step title="تثبيت OpenClaw تحت البادئة">
    - طريقة `npm` (الافتراضية): تثبيت تحت البادئة باستخدام npm، ثم كتابة wrapper إلى `<prefix>/bin/openclaw`
    - طريقة `git`: استنساخ/تحديث checkout (الافتراضي `~/openclaw`) ومع ذلك يكتب wrapper إلى `<prefix>/bin/openclaw`
  </Step>
  <Step title="تحديث خدمة gateway المحمّلة">
    إذا كانت خدمة gateway محمّلة بالفعل من تلك البادئة نفسها، فإن script يشغّل
    `openclaw gateway install --force`، ثم `openclaw gateway restart`، ثم
    يتحقق من صحة gateway على أفضل جهد.
  </Step>
</Steps>

### أمثلة (install-cli.sh)

<Tabs>
  <Tab title="الافتراضي">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```
  </Tab>
  <Tab title="بادئة مخصصة + إصدار">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --prefix /opt/openclaw --version latest
    ```
  </Tab>
  <Tab title="تثبيت git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --install-method git --git-dir ~/openclaw
    ```
  </Tab>
  <Tab title="إخراج JSON للأتمتة">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="تشغيل الإعداد التفاعلي">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --onboard
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="مرجع العلامات">

| العلامة                        | الوصف                                                                     |
| --------------------------- | ------------------------------------------------------------------------------- |
| `--prefix <path>`           | بادئة التثبيت (الافتراضي: `~/.openclaw`)                                         |
| `--install-method npm\|git` | اختيار طريقة التثبيت (الافتراضي: `npm`). الاسم المستعار: `--method`                       |
| `--npm`                     | اختصار لطريقة npm                                                         |
| `--git`, `--github`         | اختصار لطريقة git                                                         |
| `--git-dir <path>`          | دليل Git checkout (الافتراضي: `~/openclaw`). الاسم المستعار: `--dir`                  |
| `--version <ver>`           | إصدار OpenClaw أو dist-tag (الافتراضي: `latest`)                                |
| `--node-version <ver>`      | إصدار Node (الافتراضي: `22.22.0`)                                               |
| `--json`                    | إخراج أحداث NDJSON                                                              |
| `--onboard`                 | تشغيل `openclaw onboard` بعد التثبيت                                            |
| `--no-onboard`              | تخطي onboarding (الافتراضي)                                                       |
| `--set-npm-prefix`          | على Linux، فرض بادئة npm إلى `~/.npm-global` إذا كانت البادئة الحالية غير قابلة للكتابة |
| `--help`                    | عرض الاستخدام (`-h`)                                                               |

  </Accordion>

  <Accordion title="مرجع متغيرات البيئة">

| المتغير                                    | الوصف                                   |
| ------------------------------------------- | --------------------------------------------- |
| `OPENCLAW_PREFIX=<path>`                    | بادئة التثبيت                                |
| `OPENCLAW_INSTALL_METHOD=git\|npm`          | طريقة التثبيت                                |
| `OPENCLAW_VERSION=<ver>`                    | إصدار OpenClaw أو dist-tag                  |
| `OPENCLAW_NODE_VERSION=<ver>`               | إصدار Node                                  |
| `OPENCLAW_GIT_DIR=<path>`                   | دليل Git checkout لتثبيتات git       |
| `OPENCLAW_GIT_UPDATE=0\|1`                  | تبديل تحديثات git لـ checkouts الموجودة     |
| `OPENCLAW_NO_ONBOARD=1`                     | تخطي onboarding                               |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice` | مستوى سجل npm                                 |
| `SHARP_IGNORE_GLOBAL_LIBVIPS=0\|1`          | التحكم في سلوك sharp/libvips (الافتراضي: `1`) |

  </Accordion>
</AccordionGroup>

---

<a id="installps1"></a>

## install.ps1

### التدفق (install.ps1)

<Steps>
  <Step title="ضمان بيئة PowerShell + Windows">
    يتطلب PowerShell 5+.
  </Step>
  <Step title="ضمان Node.js 24 افتراضيًا">
    إذا كان مفقودًا، يحاول التثبيت عبر winget، ثم Chocolatey، ثم Scoop. وما زال Node 22 LTS، حاليًا `22.14+`، مدعومًا من أجل التوافق.
  </Step>
  <Step title="تثبيت OpenClaw">
    - طريقة `npm` (الافتراضية): تثبيت npm عالمي باستخدام `-Tag` المحدد
    - طريقة `git`: استنساخ/تحديث المستودع، والتثبيت/البناء باستخدام pnpm، وتثبيت wrapper في `%USERPROFILE%\.local\bin\openclaw.cmd`
  </Step>
  <Step title="مهام ما بعد التثبيت">
    - يضيف دليل bin المطلوب إلى PATH الخاص بالمستخدم عند الإمكان
    - يحدّث خدمة gateway المحمّلة على أفضل جهد (`openclaw gateway install --force`، ثم إعادة التشغيل)
    - يشغّل `openclaw doctor --non-interactive` عند الترقيات وتثبيتات git (على أفضل جهد)
  </Step>
</Steps>

### أمثلة (install.ps1)

<Tabs>
  <Tab title="الافتراضي">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
  <Tab title="تثبيت git">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git
    ```
  </Tab>
  <Tab title="GitHub main عبر npm">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag main
    ```
  </Tab>
  <Tab title="دليل git مخصص">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -GitDir "C:\openclaw"
    ```
  </Tab>
  <Tab title="تشغيل تجريبي">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -DryRun
    ```
  </Tab>
  <Tab title="أثر التصحيح">
    ```powershell
    # لا يحتوي install.ps1 بعد على علامة -Verbose مخصصة.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="مرجع العلامات">

| العلامة                        | الوصف                                                |
| --------------------------- | ---------------------------------------------------------- |
| `-InstallMethod npm\|git`   | طريقة التثبيت (الافتراضي: `npm`)                            |
| `-Tag <tag\|version\|spec>` | npm dist-tag أو إصدار أو package spec (الافتراضي: `latest`) |
| `-GitDir <path>`            | دليل checkout (الافتراضي: `%USERPROFILE%\openclaw`)     |
| `-NoOnboard`                | تخطي onboarding                                            |
| `-NoGitUpdate`              | تخطي `git pull`                                            |
| `-DryRun`                   | طباعة الإجراءات فقط                                         |

  </Accordion>

  <Accordion title="مرجع متغيرات البيئة">

| المتغير                           | الوصف        |
| ---------------------------------- | ------------------ |
| `OPENCLAW_INSTALL_METHOD=git\|npm` | طريقة التثبيت     |
| `OPENCLAW_GIT_DIR=<path>`          | دليل checkout |
| `OPENCLAW_NO_ONBOARD=1`            | تخطي onboarding    |
| `OPENCLAW_GIT_UPDATE=0`            | تعطيل git pull   |
| `OPENCLAW_DRY_RUN=1`               | وضع التشغيل التجريبي       |

  </Accordion>
</AccordionGroup>

<Note>
إذا تم استخدام `-InstallMethod git` وكان Git مفقودًا، فإن script ينهي التنفيذ ويطبع رابط Git for Windows.
</Note>

---

## CI والأتمتة

استخدم العلامات/متغيرات البيئة غير التفاعلية للحصول على عمليات تشغيل متوقعة.

<Tabs>
  <Tab title="install.sh (npm غير تفاعلي)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-prompt --no-onboard
    ```
  </Tab>
  <Tab title="install.sh (git غير تفاعلي)">
    ```bash
    OPENCLAW_INSTALL_METHOD=git OPENCLAW_NO_PROMPT=1 \
      curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="install-cli.sh (JSON)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="install.ps1 (تخطي onboarding)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

---

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="لماذا يُطلب Git؟">
    Git مطلوب لطريقة التثبيت `git`. أما في تثبيتات `npm`، فما زال يتم التحقق من Git/تثبيته لتجنب أعطال `spawn git ENOENT` عندما تستخدم التبعيات عناوين URL خاصة بـ git.
  </Accordion>

  <Accordion title="لماذا تواجه npm خطأ EACCES على Linux؟">
    تشير بعض إعدادات Linux إلى أن البادئة العالمية لـ npm تقع في مسارات مملوكة لـ root. ويمكن لـ `install.sh` تبديل البادئة إلى `~/.npm-global` وإلحاق تصديرات PATH بملفات rc الخاصة بـ shell (عندما تكون هذه الملفات موجودة).
  </Accordion>

  <Accordion title="مشكلات sharp/libvips">
    تضبط scripts افتراضيًا `SHARP_IGNORE_GLOBAL_LIBVIPS=1` لتجنب بناء sharp مقابل libvips الخاص بالنظام. وللتجاوز:

    ```bash
    SHARP_IGNORE_GLOBAL_LIBVIPS=0 curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

  </Accordion>

  <Accordion title='Windows: "npm error spawn git / ENOENT"'>
    ثبّت Git for Windows، وأعد فتح PowerShell، ثم أعد تشغيل برنامج التثبيت.
  </Accordion>

  <Accordion title='Windows: "openclaw is not recognized"'>
    شغّل `npm config get prefix` وأضف ذلك الدليل إلى PATH الخاص بالمستخدم لديك (من دون لاحقة `\bin` على Windows)، ثم أعد فتح PowerShell.
  </Accordion>

  <Accordion title="Windows: كيفية الحصول على إخراج verbose لبرنامج التثبيت">
    لا يوفّر `install.ps1` حاليًا مفتاح `-Verbose`.
    استخدم تتبع PowerShell لتشخيصات على مستوى script:

    ```powershell
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

  </Accordion>

  <Accordion title="لم يتم العثور على openclaw بعد التثبيت">
    تكون المشكلة عادة في PATH. راجع [استكشاف أخطاء Node.js وإصلاحها](/install/node#troubleshooting).
  </Accordion>
</AccordionGroup>
