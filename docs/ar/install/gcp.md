---
read_when:
    - أنت تريد تشغيل OpenClaw على GCP على مدار الساعة طوال أيام الأسبوع
    - أنت تريد Gateway دائم التشغيل وبمستوى إنتاجي على جهاز VM الخاص بك
    - أنت تريد تحكمًا كاملًا في الاستمرارية، والثنائيات، وسلوك إعادة التشغيل
summary: شغّل OpenClaw Gateway على مدار الساعة طوال أيام الأسبوع على جهاز GCP Compute Engine VM (باستخدام Docker) مع حالة دائمة
title: GCP
x-i18n:
    generated_at: "2026-04-05T12:47:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 73daaee3de71dad5175f42abf3e11355f2603b2f9e2b2523eac4d4c7015e3ebc
    source_path: install/gcp.md
    workflow: 15
---

# OpenClaw على GCP Compute Engine (Docker، دليل VPS للإنتاج)

## الهدف

تشغيل OpenClaw Gateway دائم على جهاز GCP Compute Engine VM باستخدام Docker، مع حالة دائمة، وثنائيات مضمنة في الصورة، وسلوك آمن لإعادة التشغيل.

إذا كنت تريد "OpenClaw يعمل 24/7 مقابل نحو 5 إلى 12 دولارًا شهريًا"، فهذا إعداد موثوق على Google Cloud.
تختلف الأسعار حسب نوع الجهاز والمنطقة؛ اختر أصغر جهاز VM يناسب حمل العمل لديك وقم بالترقية إذا واجهت حالات OOM.

## ماذا نفعل هنا (بعبارات بسيطة)؟

- إنشاء مشروع GCP وتمكين الفوترة
- إنشاء جهاز Compute Engine VM
- تثبيت Docker (بيئة تشغيل تطبيق معزولة)
- تشغيل OpenClaw Gateway داخل Docker
- جعل `~/.openclaw` + `~/.openclaw/workspace` دائمة على المضيف (لتبقى بعد إعادة التشغيل/إعادة البناء)
- الوصول إلى Control UI من جهازك المحمول عبر نفق SSH

تتضمن حالة `~/.openclaw` المركبة هذه `openclaw.json`، وملف
`agents/<agentId>/agent/auth-profiles.json` لكل وكيل، و`.env`.

يمكن الوصول إلى Gateway عبر:

- إعادة توجيه المنافذ عبر SSH من جهازك المحمول
- كشف المنفذ مباشرة إذا كنت تدير الجدار الناري والرموز المميزة بنفسك

يستخدم هذا الدليل Debian على GCP Compute Engine.
كما يعمل Ubuntu أيضًا؛ فقط طابِق الحزم وفقًا لذلك.
ولمسار Docker العام، راجع [Docker](/install/docker).

---

## المسار السريع (للمشغلين المتمرسين)

1. أنشئ مشروع GCP + فعّل Compute Engine API
2. أنشئ جهاز Compute Engine VM (‏e2-small، ‏Debian 12، ‏20GB)
3. اتصل عبر SSH إلى الجهاز الافتراضي
4. ثبّت Docker
5. انسخ مستودع OpenClaw
6. أنشئ أدلة مضيف دائمة
7. اضبط `.env` و`docker-compose.yml`
8. ضمّن الثنائيات المطلوبة، وابنِ، ثم شغّل

---

## ما الذي تحتاجه

- حساب GCP (طبقة مجانية مؤهلة لـ e2-micro)
- تثبيت gcloud CLI (أو استخدام Cloud Console)
- وصول SSH من جهازك المحمول
- إلمام أساسي بـ SSH + النسخ/اللصق
- نحو 20 إلى 30 دقيقة
- Docker وDocker Compose
- بيانات اعتماد auth الخاصة بالنماذج
- بيانات اعتماد موفّرين اختيارية
  - QR الخاص بـ WhatsApp
  - رمز بوت Telegram المميز
  - Gmail OAuth

---

<Steps>
  <Step title="تثبيت gcloud CLI (أو استخدام Console)">
    **الخيار A: gcloud CLI** (موصى به للأتمتة)

    ثبّته من [https://cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)

    نفّذ التهيئة والمصادقة:

    ```bash
    gcloud init
    gcloud auth login
    ```

    **الخيار B: Cloud Console**

    يمكن تنفيذ كل الخطوات عبر واجهة الويب على [https://console.cloud.google.com](https://console.cloud.google.com)

  </Step>

  <Step title="إنشاء مشروع GCP">
    **CLI:**

    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    ```

    فعّل الفوترة على [https://console.cloud.google.com/billing](https://console.cloud.google.com/billing) (مطلوب من أجل Compute Engine).

    فعّل Compute Engine API:

    ```bash
    gcloud services enable compute.googleapis.com
    ```

    **Console:**

    1. انتقل إلى IAM & Admin > Create Project
    2. سمِّ المشروع ثم أنشئه
    3. فعّل الفوترة للمشروع
    4. انتقل إلى APIs & Services > Enable APIs > وابحث عن "Compute Engine API" > ثم Enable

  </Step>

  <Step title="إنشاء الجهاز الافتراضي">
    **أنواع الأجهزة:**

    | النوع | المواصفات | التكلفة | الملاحظات |
    | ----- | --------- | ------- | ---------- |
    | e2-medium | 2 vCPU، ‏4GB RAM | نحو 25 دولارًا شهريًا | الأكثر موثوقية لعمليات بناء Docker المحلية |
    | e2-small | 2 vCPU، ‏2GB RAM | نحو 12 دولارًا شهريًا | الحد الأدنى الموصى به لبناء Docker |
    | e2-micro | ‏2 vCPU (مشتركة)، ‏1GB RAM | مؤهل للطبقة المجانية | يفشل كثيرًا مع OOM أثناء بناء Docker (الخروج 137) |

    **CLI:**

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

    **Console:**

    1. انتقل إلى Compute Engine > VM instances > Create instance
    2. الاسم: `openclaw-gateway`
    3. المنطقة: `us-central1`، والمنطقة الفرعية: `us-central1-a`
    4. نوع الجهاز: `e2-small`
    5. قرص الإقلاع: Debian 12، ‏20GB
    6. أنشئه

  </Step>

  <Step title="الاتصال عبر SSH إلى الجهاز الافتراضي">
    **CLI:**

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    **Console:**

    انقر زر "SSH" بجوار جهازك الافتراضي في لوحة تحكم Compute Engine.

    ملاحظة: قد يستغرق نشر مفتاح SSH من دقيقة إلى دقيقتين بعد إنشاء الجهاز الافتراضي. إذا رُفض الاتصال، فانتظر ثم أعد المحاولة.

  </Step>

  <Step title="تثبيت Docker (على الجهاز الافتراضي)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    سجّل الخروج ثم الدخول مجددًا حتى يسري تغيير المجموعة:

    ```bash
    exit
    ```

    ثم اتصل عبر SSH مرة أخرى:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    تحقّق:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="استنساخ مستودع OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    يفترض هذا الدليل أنك ستبني صورة مخصصة لضمان استمرارية الثنائيات.

  </Step>

  <Step title="إنشاء أدلة المضيف الدائمة">
    حاويات Docker مؤقتة بطبيعتها.
    يجب أن تعيش كل حالة طويلة الأمد على المضيف.

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="إعداد متغيرات البيئة">
    أنشئ `.env` في جذر المستودع.

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=change-me-now
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=change-me-now
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    أنشئ أسرارًا قوية:

    ```bash
    openssl rand -hex 32
    ```

    **لا تقم بعمل commit لهذا الملف.**

    هذا الملف `.env` مخصص لمتغيرات بيئة الحاوية/وقت التشغيل مثل `OPENCLAW_GATEWAY_TOKEN`.
    أما auth المخزن الخاص بـ OAuth/API key للمزوّدين فيعيش في
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` المركب.

  </Step>

  <Step title="إعداد Docker Compose">
    أنشئ أو حدّث `docker-compose.yml`.

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # الموصى به: أبقِ Gateway مقصورة على loopback على الجهاز الافتراضي؛ وادخل إليها عبر نفق SSH.
          # لكشفها علنًا، أزل السابقة `127.0.0.1:` واضبط الجدار الناري وفقًا لذلك.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    إن `--allow-unconfigured` مخصص فقط لتسهيل التهيئة الأولية، وليس بديلًا عن إعداد gateway صحيح. ما زال عليك تعيين auth (`gateway.auth.token` أو كلمة مرور) واستخدام إعدادات bind آمنة لبيئة النشر الخاصة بك.

  </Step>

  <Step title="خطوات وقت تشغيل Docker المشتركة على الجهاز الافتراضي">
    استخدم دليل وقت التشغيل المشترك للتدفق العام لمضيف Docker:

    - [ضمّن الثنائيات المطلوبة في الصورة](/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [البناء والتشغيل](/install/docker-vm-runtime#build-and-launch)
    - [ما الذي يستمر وأين](/install/docker-vm-runtime#what-persists-where)
    - [التحديثات](/install/docker-vm-runtime#updates)

  </Step>

  <Step title="ملاحظات تشغيل خاصة بـ GCP">
    على GCP، إذا فشل البناء مع `Killed` أو `exit code 137` أثناء `pnpm install --frozen-lockfile`، فهذا يعني أن الجهاز الافتراضي نفدت ذاكرته. استخدم `e2-small` كحد أدنى، أو `e2-medium` لبناء أولي أكثر موثوقية.

    عند الربط على LAN (`OPENCLAW_GATEWAY_BIND=lan`)، اضبط origin موثوقًا للمتصفح قبل المتابعة:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    إذا غيّرت منفذ البوابة، فاستبدل `18789` بالمنفذ الذي ضبطته.

  </Step>

  <Step title="الوصول من جهازك المحمول">
    أنشئ نفق SSH لإعادة توجيه منفذ Gateway:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    افتحه في المتصفح:

    `http://127.0.0.1:18789/`

    أعد طباعة رابط dashboard نظيف:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    إذا طلبت واجهة المستخدم مصادقة السر المشترك، فألصق الرمز المميز أو
    كلمة المرور المهيأة في إعدادات Control UI. يكتب تدفق Docker هذا رمزًا مميزًا
    افتراضيًا؛ وإذا بدّلت إعداد الحاوية إلى مصادقة كلمة مرور، فاستخدم تلك
    الكلمة بدلًا من ذلك.

    إذا عرضت Control UI الرسالة `unauthorized` أو `disconnected (1008): pairing required`، فوافق على جهاز المتصفح:

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    هل تحتاج إلى مرجع الاستمرارية والتحديثات المشتركة مرة أخرى؟
    راجع [Docker VM Runtime](/install/docker-vm-runtime#what-persists-where) و[تحديثات Docker VM Runtime](/install/docker-vm-runtime#updates).

  </Step>
</Steps>

---

## استكشاف الأخطاء وإصلاحها

**تم رفض اتصال SSH**

قد يستغرق نشر مفتاح SSH من دقيقة إلى دقيقتين بعد إنشاء الجهاز الافتراضي. انتظر ثم أعد المحاولة.

**مشكلات OS Login**

تحقق من ملف تعريف OS Login الخاص بك:

```bash
gcloud compute os-login describe-profile
```

تأكد من أن حسابك يملك أذونات IAM المطلوبة (‏Compute OS Login أو ‏Compute OS Admin Login).

**نفاد الذاكرة (OOM)**

إذا فشل بناء Docker مع `Killed` و`exit code 137`، فهذا يعني أن النظام أنهى العملية بسبب نفاد الذاكرة. قم بالترقية إلى e2-small (الحد الأدنى) أو e2-medium (موصى به للبناءات المحلية الموثوقة):

```bash
# أوقف الجهاز الافتراضي أولًا
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# غيّر نوع الجهاز
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# ابدأ الجهاز الافتراضي
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

---

## حسابات الخدمة (أفضل ممارسة أمنية)

للاستخدام الشخصي، يعمل حساب المستخدم الافتراضي لديك بشكل جيد.

بالنسبة إلى الأتمتة أو مسارات CI/CD، أنشئ حساب خدمة مخصصًا بأقل قدر من الأذونات:

1. أنشئ حساب خدمة:

   ```bash
   gcloud iam service-accounts create openclaw-deploy \
     --display-name="OpenClaw Deployment"
   ```

2. امنحه دور Compute Instance Admin (أو دورًا مخصصًا أضيق):

   ```bash
   gcloud projects add-iam-policy-binding my-openclaw-project \
     --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
     --role="roles/compute.instanceAdmin.v1"
   ```

تجنب استخدام دور Owner للأتمتة. استخدم مبدأ أقل قدر من الامتيازات.

راجع [https://cloud.google.com/iam/docs/understanding-roles](https://cloud.google.com/iam/docs/understanding-roles) للحصول على تفاصيل أدوار IAM.

---

## الخطوات التالية

- إعداد قنوات المراسلة: [القنوات](/channels)
- إقران الأجهزة المحلية كعُقد: [العُقد](/nodes)
- إعداد Gateway: [إعداد Gateway](/gateway/configuration)
