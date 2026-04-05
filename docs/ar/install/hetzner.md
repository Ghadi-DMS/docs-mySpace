---
read_when:
    - تريد تشغيل OpenClaw طوال الوقت على VPS سحابي (وليس على حاسوبك المحمول)
    - تريد Gateway جاهزة للإنتاج وتعمل دائمًا على VPS خاص بك
    - تريد تحكمًا كاملًا في الاستمرارية، والملفات التنفيذية، وسلوك إعادة التشغيل
    - أنت تشغّل OpenClaw داخل Docker على Hetzner أو مزود مشابه
summary: شغّل OpenClaw Gateway على Hetzner VPS (باستخدام Docker) طوال الوقت مع حالة مستدامة وملفات تنفيذية مضمّنة
title: Hetzner
x-i18n:
    generated_at: "2026-04-05T12:47:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: d859e4c0943040b022835f320708f879a11eadef70f2816cf0f2824eaaf165ef
    source_path: install/hetzner.md
    workflow: 15
---

# OpenClaw على Hetzner (Docker، دليل VPS للإنتاج)

## الهدف

تشغيل OpenClaw Gateway دائمة على Hetzner VPS باستخدام Docker، مع حالة مستدامة، وملفات تنفيذية مضمّنة، وسلوك آمن لإعادة التشغيل.

إذا كنت تريد "OpenClaw تعمل 24/7 مقابل حوالي 5 دولارات"، فهذا هو أبسط إعداد موثوق.
تتغير أسعار Hetzner؛ اختر أصغر VPS بنظام Debian/Ubuntu ثم وسّع الموارد إذا واجهت حالات OOM.

تذكير بنموذج الأمان:

- الوكلاء المشتركون على مستوى الشركة مناسبون عندما يكون الجميع ضمن حد ثقة واحد ويكون وقت التشغيل مخصصًا للأعمال فقط.
- حافظ على فصل صارم: VPS/وقت تشغيل مخصص + حسابات مخصصة؛ لا تستخدم ملفات تعريف Apple/Google/browser/password-manager الشخصية على ذلك المضيف.
- إذا كان المستخدمون خصومًا لبعضهم البعض، فقسّم حسب gateway/المضيف/مستخدم نظام التشغيل.

راجع [الأمان](/gateway/security) و[VPS hosting](/vps).

## ماذا نفعل هنا (بكلمات بسيطة)؟

- استئجار خادم Linux صغير (Hetzner VPS)
- تثبيت Docker (وقت تشغيل تطبيق معزول)
- تشغيل OpenClaw Gateway داخل Docker
- الإبقاء على `~/.openclaw` + `~/.openclaw/workspace` على المضيف (لتبقى بعد إعادة التشغيل/إعادة البناء)
- الوصول إلى Control UI من حاسوبك المحمول عبر نفق SSH

تتضمن حالة `~/.openclaw` المركبة هذه الملفات `openclaw.json` و
`agents/<agentId>/agent/auth-profiles.json` لكل وكيل، و`.env`.

يمكن الوصول إلى Gateway عبر:

- إعادة توجيه منفذ SSH من حاسوبك المحمول
- كشف المنفذ مباشرة إذا كنت تدير الجدار الناري وtokens بنفسك

يفترض هذا الدليل استخدام Ubuntu أو Debian على Hetzner.  
إذا كنت تستخدم Linux VPS أخرى، فطابق الحزم وفقًا لذلك.
وللتدفق العام الخاص بـ Docker، راجع [Docker](/install/docker).

---

## المسار السريع (للمشغلين ذوي الخبرة)

1. جهّز Hetzner VPS
2. ثبّت Docker
3. انسخ مستودع OpenClaw
4. أنشئ أدلة مضيف مستدامة
5. كوّن `.env` و`docker-compose.yml`
6. ضمّن الملفات التنفيذية المطلوبة داخل الصورة
7. نفّذ `docker compose up -d`
8. تحقّق من الاستمرارية والوصول إلى Gateway

---

## ما الذي تحتاجه

- Hetzner VPS مع وصول root
- وصول SSH من حاسوبك المحمول
- راحة أساسية في استخدام SSH + النسخ/اللصق
- حوالي 20 دقيقة
- Docker وDocker Compose
- بيانات اعتماد مصادقة النموذج
- بيانات اعتماد المزوّدين اختياريًا
  - WhatsApp QR
  - Telegram bot token
  - Gmail OAuth

---

<Steps>
  <Step title="جهّز VPS">
    أنشئ Ubuntu أو Debian VPS في Hetzner.

    اتصل كمستخدم root:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    يفترض هذا الدليل أن VPS تحتفظ بالحالة.
    لا تتعامل معها على أنها بنية تحتية قابلة للاستبدال.

  </Step>

  <Step title="ثبّت Docker (على VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    تحقّق:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="انسخ مستودع OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    يفترض هذا الدليل أنك ستبني صورة مخصصة لضمان استمرارية الملفات التنفيذية.

  </Step>

  <Step title="أنشئ أدلة مضيف مستدامة">
    حاويات Docker مؤقتة.
    يجب أن تعيش كل حالة طويلة الأمد على المضيف.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # اضبط الملكية على مستخدم الحاوية (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="كوّن متغيرات البيئة">
    أنشئ ملف `.env` في جذر المستودع.

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=change-me-now
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=change-me-now
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    أنشئ أسرارًا قوية:

    ```bash
    openssl rand -hex 32
    ```

    **لا تلتزم بهذا الملف في المستودع.**

    ملف `.env` هذا مخصص لبيئة الحاوية/وقت التشغيل مثل `OPENCLAW_GATEWAY_TOKEN`.
    أما مصادقة OAuth/API-key المخزنة للمزوّدين فتوجد في
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` المركبة.

  </Step>

  <Step title="تكوين Docker Compose">
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
          # الموصى به: أبقِ Gateway مقتصرة على loopback على VPS؛ وادخل إليها عبر نفق SSH.
          # لكشفها علنًا، احذف السابقة `127.0.0.1:` واضبط الجدار الناري وفقًا لذلك.
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

    إن `--allow-unconfigured` مخصص فقط لتسهيل bootstrap، وليس بديلًا عن تكوين gateway صحيح. ما زال عليك ضبط المصادقة (`gateway.auth.token` أو كلمة مرور) واستخدام إعدادات bind آمنة لعملية النشر لديك.

  </Step>

  <Step title="خطوات وقت التشغيل المشتركة لـ Docker VM">
    استخدم دليل وقت التشغيل المشترك لتدفق مضيف Docker الشائع:

    - [ضمّن الملفات التنفيذية المطلوبة داخل الصورة](/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [البناء والتشغيل](/install/docker-vm-runtime#build-and-launch)
    - [ما الذي يبقى وأين](/install/docker-vm-runtime#what-persists-where)
    - [التحديثات](/install/docker-vm-runtime#updates)

  </Step>

  <Step title="الوصول الخاص بـ Hetzner">
    بعد تنفيذ خطوات البناء والتشغيل المشتركة، أنشئ نفقًا من حاسوبك المحمول:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    افتح:

    `http://127.0.0.1:18789/`

    ثم الصق السر المشترك المكوَّن. يستخدم هذا الدليل gateway token
    افتراضيًا؛ وإذا بدّلت إلى مصادقة password، فاستخدم تلك الكلمة بدلًا من ذلك.

  </Step>
</Steps>

توجد خريطة الاستمرارية المشتركة في [Docker VM Runtime](/install/docker-vm-runtime#what-persists-where).

## البنية التحتية كتعريف برمجي (Terraform)

بالنسبة إلى الفرق التي تفضل سير عمل البنية التحتية كتعريف برمجي، يوفر إعداد Terraform تُديره المجتمع ما يلي:

- تكوين Terraform معياري مع إدارة للحالة البعيدة
- تهيئة آلية عبر cloud-init
- scripts للنشر (bootstrap، وdeploy، وbackup/restore)
- تقوية أمنية (الجدار الناري، وUFW، ووصول SSH فقط)
- تكوين نفق SSH للوصول إلى gateway

**المستودعات:**

- البنية التحتية: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- تكوين Docker: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

يكمل هذا النهج إعداد Docker أعلاه عبر عمليات نشر قابلة لإعادة الإنتاج، وبنية تحتية مضبوطة بالإصدارات، واستعادة كوارث آلية.

> **ملاحظة:** تتم صيانته من المجتمع. وبالنسبة إلى المشكلات أو المساهمات، راجع روابط المستودعات أعلاه.

## الخطوات التالية

- إعداد قنوات المراسلة: [القنوات](/channels)
- تكوين Gateway: [تكوين Gateway](/gateway/configuration)
- الحفاظ على تحديث OpenClaw: [التحديث](/install/updating)
