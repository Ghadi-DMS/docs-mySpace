---
read_when:
    - تحتاج إلى تثبيت Node.js قبل تثبيت OpenClaw
    - قمت بتثبيت OpenClaw لكن `openclaw` غير موجود كأمر
    - يفشل `npm install -g` بسبب الأذونات أو مشكلات PATH
summary: تثبيت وإعداد Node.js لـ OpenClaw — متطلبات الإصدار وخيارات التثبيت واستكشاف أخطاء PATH وإصلاحها
title: Node.js
x-i18n:
    generated_at: "2026-04-05T12:48:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5e880f6132359dba8720638669df2d71cf857d516cbf5df2589ffeed269b5120
    source_path: install/node.md
    workflow: 15
---

# Node.js

يتطلب OpenClaw **Node 22.14 أو أحدث**. **Node 24 هو وقت التشغيل الافتراضي والموصى به** للتثبيتات وCI وتدفقات الإصدار. وما يزال Node 22 مدعومًا عبر خط LTS النشط. سيكتشف [نص التثبيت](/install#alternative-install-methods) Node ويثبتها تلقائيًا — وهذه الصفحة مخصصة للحالات التي تريد فيها إعداد Node بنفسك والتأكد من أن كل شيء مضبوط بشكل صحيح (الإصدارات، وPATH، والتثبيتات العامة).

## التحقق من الإصدار

```bash
node -v
```

إذا طبع هذا `v24.x.x` أو أعلى، فأنت تستخدم الإصدار الافتراضي الموصى به. وإذا طبع `v22.14.x` أو أعلى، فأنت على مسار Node 22 LTS المدعوم، لكننا ما زلنا نوصي بالترقية إلى Node 24 عندما يكون ذلك مناسبًا. وإذا لم تكن Node مثبتة أو كان الإصدار قديمًا جدًا، فاختر إحدى طرق التثبيت أدناه.

## تثبيت Node

<Tabs>
  <Tab title="macOS">
    **Homebrew** (موصى به):

    ```bash
    brew install node
    ```

    أو نزّل مثبّت macOS من [nodejs.org](https://nodejs.org/).

  </Tab>
  <Tab title="Linux">
    **Ubuntu / Debian:**

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

    **Fedora / RHEL:**

    ```bash
    sudo dnf install nodejs
    ```

    أو استخدم مدير إصدارات (انظر أدناه).

  </Tab>
  <Tab title="Windows">
    **winget** (موصى به):

    ```powershell
    winget install OpenJS.NodeJS.LTS
    ```

    **Chocolatey:**

    ```powershell
    choco install nodejs-lts
    ```

    أو نزّل مثبّت Windows من [nodejs.org](https://nodejs.org/).

  </Tab>
</Tabs>

<Accordion title="استخدام مدير إصدارات (nvm، fnm، mise، asdf)">
  تتيح لك مدراء الإصدارات التبديل بين إصدارات Node بسهولة. من الخيارات الشائعة:

- [**fnm**](https://github.com/Schniz/fnm) — سريع ومتعدد المنصات
- [**nvm**](https://github.com/nvm-sh/nvm) — واسع الاستخدام على macOS/Linux
- [**mise**](https://mise.jdx.dev/) — متعدد اللغات (Node وPython وRuby وغيرها)

مثال باستخدام fnm:

```bash
fnm install 24
fnm use 24
```

  <Warning>
  تأكد من تهيئة مدير الإصدارات في ملف بدء تشغيل shell لديك (`~/.zshrc` أو `~/.bashrc`). إذا لم تفعل ذلك، فقد لا يتم العثور على `openclaw` في جلسات الطرفية الجديدة لأن PATH لن تتضمن دليل bin الخاص بـ Node.
  </Warning>
</Accordion>

## استكشاف الأخطاء وإصلاحها

### `openclaw: command not found`

يعني هذا في الغالب أن دليل bin العام الخاص بـ npm غير موجود على PATH.

<Steps>
  <Step title="اعثر على npm prefix العام لديك">
    ```bash
    npm prefix -g
    ```
  </Step>
  <Step title="تحقق مما إذا كان موجودًا على PATH">
    ```bash
    echo "$PATH"
    ```

    ابحث عن `<npm-prefix>/bin` ‏(macOS/Linux) أو `<npm-prefix>` ‏(Windows) في الخرج.

  </Step>
  <Step title="أضِفه إلى ملف بدء تشغيل shell لديك">
    <Tabs>
      <Tab title="macOS / Linux">
        أضف إلى `~/.zshrc` أو `~/.bashrc`:

        ```bash
        export PATH="$(npm prefix -g)/bin:$PATH"
        ```

        ثم افتح طرفية جديدة (أو شغّل `rehash` في zsh / `hash -r` في bash).
      </Tab>
      <Tab title="Windows">
        أضف خرج `npm prefix -g` إلى PATH النظام عبر Settings → System → Environment Variables.
      </Tab>
    </Tabs>

  </Step>
</Steps>

### أخطاء الأذونات في `npm install -g` ‏(Linux)

إذا رأيت أخطاء `EACCES`، فحوّل npm global prefix إلى دليل قابل للكتابة من قبل المستخدم:

```bash
mkdir -p "$HOME/.npm-global"
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
```

أضف سطر `export PATH=...` إلى `~/.bashrc` أو `~/.zshrc` لجعله دائمًا.

## ذو صلة

- [Install Overview](/install) — جميع طرق التثبيت
- [Updating](/install/updating) — إبقاء OpenClaw محدثًا
- [Getting Started](/ar/start/getting-started) — الخطوات الأولى بعد التثبيت
