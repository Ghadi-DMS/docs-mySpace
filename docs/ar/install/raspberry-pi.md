---
read_when:
    - إعداد OpenClaw على Raspberry Pi
    - تشغيل OpenClaw على أجهزة ARM
    - بناء ذكاء اصطناعي شخصي منخفض التكلفة ودائم التشغيل
summary: استضافة OpenClaw على Raspberry Pi للاستضافة الذاتية الدائمة التشغيل
title: Raspberry Pi
x-i18n:
    generated_at: "2026-04-05T12:48:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 222ccbfb18a8dcec483adac6f5647dcb455c84edbad057e0ba2589a6da570b4c
    source_path: install/raspberry-pi.md
    workflow: 15
---

# Raspberry Pi

شغّل Gateway دائمًا ومستمرًا لـ OpenClaw على Raspberry Pi. وبما أن Pi يعمل فقط كبوابة gateway (بينما تعمل النماذج في السحابة عبر API)، فإن حتى جهاز Pi متواضع يمكنه التعامل مع الحمل بشكل جيد.

## المتطلبات المسبقة

- Raspberry Pi 4 أو 5 مع ذاكرة RAM بسعة 2 GB أو أكثر (يوصى بـ 4 GB)
- بطاقة MicroSD ‏(16 GB أو أكثر) أو USB SSD ‏(أداء أفضل)
- مزود طاقة رسمي لـ Pi
- اتصال شبكة (Ethernet أو WiFi)
- نظام Raspberry Pi OS ‏64-bit ‏(مطلوب -- لا تستخدم 32-bit)
- حوالي 30 دقيقة

## الإعداد

<Steps>
  <Step title="نسخ نظام التشغيل">
    استخدم **Raspberry Pi OS Lite (64-bit)** -- لا حاجة إلى واجهة سطح مكتب لخادم بلا واجهة.

    1. نزّل [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
    2. اختر نظام التشغيل: **Raspberry Pi OS Lite (64-bit)**.
    3. في مربع حوار الإعدادات، قم بالتهيئة المسبقة لما يلي:
       - اسم المضيف: `gateway-host`
       - تمكين SSH
       - تعيين اسم المستخدم وكلمة المرور
       - إعداد WiFi ‏(إذا كنت لا تستخدم Ethernet)
    4. انسخه إلى بطاقة SD أو محرك USB، ثم أدخله وشغّل Pi.

  </Step>

  <Step title="الاتصال عبر SSH">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="تحديث النظام">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # Set timezone (important for cron and reminders)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="تثبيت Node.js 24">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="إضافة swap (مهم للأجهزة ذات 2 GB أو أقل)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Reduce swappiness for low-RAM devices
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="تثبيت OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="تشغيل onboarding">
    ```bash
    openclaw onboard --install-daemon
    ```

    اتبع المعالج. يوصى باستخدام مفاتيح API بدلًا من OAuth للأجهزة بلا واجهة. ويُعد Telegram أسهل قناة للبدء.

  </Step>

  <Step title="التحقق">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="الوصول إلى Control UI">
    على جهاز الكمبيوتر لديك، احصل على عنوان URL للوحة التحكم من Pi:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    ثم أنشئ نفق SSH في طرفية أخرى:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    افتح عنوان URL المطبوع في متصفحك المحلي. وللوصول البعيد الدائم، راجع [تكامل Tailscale](/gateway/tailscale).

  </Step>
</Steps>

## نصائح الأداء

**استخدم USB SSD** -- بطاقات SD بطيئة وتتعرض للاهتراء. ويحسن USB SSD الأداء بشكل كبير. راجع [دليل الإقلاع من USB لـ Pi](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot).

**فعّل module compile cache** -- يسرّع استدعاءات CLI المتكررة على أجهزة Pi الأقل قدرة:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

**تقليل استخدام الذاكرة** -- في الإعدادات بلا واجهة، حرر ذاكرة GPU وعطّل الخدمات غير المستخدمة:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

## استكشاف الأخطاء وإصلاحها

**نفاد الذاكرة** -- تحقّق من أن swap نشط باستخدام `free -h`. عطّل الخدمات غير المستخدمة (`sudo systemctl disable cups bluetooth avahi-daemon`). استخدم النماذج المعتمدة على API فقط.

**أداء بطيء** -- استخدم USB SSD بدلًا من بطاقة SD. تحقّق من اختناق CPU باستخدام `vcgencmd get_throttled` ‏(يجب أن يعيد `0x0`).

**الخدمة لا تبدأ** -- تحقّق من السجلات باستخدام `journalctl --user -u openclaw-gateway.service --no-pager -n 100` وشغّل `openclaw doctor --non-interactive`. وإذا كان هذا Pi بلا واجهة، فتحقق أيضًا من تمكين lingering: ‏`sudo loginctl enable-linger "$(whoami)"`.

**مشكلات ملف ARM التنفيذي** -- إذا فشلت Skill برسالة "exec format error"، فتحقق مما إذا كان الملف التنفيذي يملك إصدار ARM64. وتحقق من البنية باستخدام `uname -m` ‏(يجب أن يعرض `aarch64`).

**انقطاع WiFi** -- عطّل إدارة طاقة WiFi: ‏`sudo iwconfig wlan0 power off`.

## الخطوات التالية

- [القنوات](/channels) -- صِل Telegram وWhatsApp وDiscord والمزيد
- [إعدادات Gateway](/gateway/configuration) -- جميع خيارات الإعدادات
- [التحديث](/install/updating) -- حافظ على تحديث OpenClaw
