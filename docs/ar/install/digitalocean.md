---
read_when:
    - إعداد OpenClaw على DigitalOcean
    - البحث عن VPS مدفوع بسيط لـ OpenClaw
summary: استضافة OpenClaw على DigitalOcean Droplet
title: DigitalOcean
x-i18n:
    generated_at: "2026-04-05T12:46:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4b161db8ec643d8313938a2453ce6242fc1ee8ea1fd2069916276f1aadeb71f1
    source_path: install/digitalocean.md
    workflow: 15
---

# DigitalOcean

شغّل OpenClaw Gateway دائمة على DigitalOcean Droplet.

## المتطلبات الأساسية

- حساب DigitalOcean ‏([التسجيل](https://cloud.digitalocean.com/registrations/new))
- زوج مفاتيح SSH (أو الاستعداد لاستخدام مصادقة كلمة المرور)
- نحو 20 دقيقة

## الإعداد

<Steps>
  <Step title="إنشاء Droplet">
    <Warning>
    استخدم صورة أساسية نظيفة (Ubuntu 24.04 LTS). تجنب صور Marketplace الجاهزة بنقرة واحدة من جهات خارجية ما لم تكن قد راجعت نصوص بدء التشغيل الافتراضية وقواعد جدار الحماية الخاصة بها.
    </Warning>

    1. سجّل الدخول إلى [DigitalOcean](https://cloud.digitalocean.com/).
    2. انقر **Create > Droplets**.
    3. اختر:
       - **Region:** الأقرب إليك
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic, Regular, 1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** مفتاح SSH (موصى به) أو كلمة مرور
    4. انقر **Create Droplet** ودوّن عنوان IP.

  </Step>

  <Step title="الاتصال والتثبيت">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # Install Node.js 24
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # Install OpenClaw
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw --version
    ```

  </Step>

  <Step title="تشغيل التهيئة الأولية">
    ```bash
    openclaw onboard --install-daemon
    ```

    يرشدك المعالج خلال مصادقة النموذج، وإعداد القنوات، وإنشاء رمز gateway، وتثبيت daemon ‏(systemd).

  </Step>

  <Step title="إضافة swap (موصى به لـ 1 GB Droplets)">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="التحقق من gateway">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="الوصول إلى Control UI">
    ترتبط gateway بعنوان loopback افتراضيًا. اختر أحد هذه الخيارات.

    **الخيار A: نفق SSH (الأبسط)**

    ```bash
    # From your local machine
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    ثم افتح `http://localhost:18789`.

    **الخيار B: Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    ثم افتح `https://<magicdns>/` من أي جهاز على tailnet الخاصة بك.

    **الخيار C: ربط tailnet (من دون Serve)**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    ثم افتح `http://<tailscale-ip>:18789` (الرمز مطلوب).

  </Step>
</Steps>

## استكشاف الأخطاء وإصلاحها

**لا تبدأ Gateway** -- شغّل `openclaw doctor --non-interactive` وتحقق من السجلات باستخدام `journalctl --user -u openclaw-gateway.service -n 50`.

**المنفذ مستخدم بالفعل** -- شغّل `lsof -i :18789` للعثور على العملية، ثم أوقفها.

**نفاد الذاكرة** -- تحقق من أن swap نشطة باستخدام `free -h`. وإذا استمر OOM، فاستخدم نماذج تعتمد على API ‏(Claude، GPT) بدلًا من النماذج المحلية، أو قم بالترقية إلى Droplet بسعة 2 GB.

## الخطوات التالية

- [Channels](/channels) -- صِل Telegram وWhatsApp وDiscord والمزيد
- [Gateway configuration](/gateway/configuration) -- جميع خيارات التكوين
- [Updating](/install/updating) -- حافظ على OpenClaw محدّثًا
