---
read_when:
    - إعداد OpenClaw على Raspberry Pi
    - تشغيل OpenClaw على أجهزة ARM
    - بناء ذكاء اصطناعي شخصي دائم التشغيل ورخيص
summary: OpenClaw على Raspberry Pi ‏(إعداد ذاتي الاستضافة منخفض التكلفة)
title: Raspberry Pi ‏(المنصة)
x-i18n:
    generated_at: "2026-04-05T12:51:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 07f34e91899b7e0a31d9b944f3cb0cfdd4ecdeba58b619ae554379abdbf37eaf
    source_path: platforms/raspberry-pi.md
    workflow: 15
---

# OpenClaw على Raspberry Pi

## الهدف

تشغيل OpenClaw Gateway دائمة ودائمة التشغيل على Raspberry Pi بتكلفة لمرة واحدة **تقارب 35-80 دولارًا** (من دون رسوم شهرية).

مثالية من أجل:

- مساعد ذكاء اصطناعي شخصي يعمل على مدار الساعة
- محور أتمتة منزلية
- bot منخفضة الاستهلاك ومتاحة دائمًا لـ Telegram/WhatsApp

## متطلبات العتاد

| طراز Pi          | الذاكرة | هل يعمل؟   | ملاحظات                             |
| ---------------- | ------- | ---------- | ----------------------------------- |
| **Pi 5**         | 4GB/8GB | ✅ الأفضل  | الأسرع، وموصى به                   |
| **Pi 4**         | 4GB     | ✅ جيد     | الخيار المتوازن لمعظم المستخدمين    |
| **Pi 4**         | 2GB     | ✅ مقبول   | يعمل، أضف swap                      |
| **Pi 4**         | 1GB     | ⚠️ ضيق     | ممكن مع swap وتكوين أدنى           |
| **Pi 3B+**       | 1GB     | ⚠️ بطيء    | يعمل لكنه بطيء                     |
| **Pi Zero 2 W**  | 512MB   | ❌         | غير موصى به                        |

**الحد الأدنى للمواصفات:** 1GB RAM، نواة واحدة، 500MB قرص  
**الموصى به:** 2GB+ RAM، ونظام تشغيل 64-bit، وبطاقة SD بسعة 16GB+ ‏(أو USB SSD)

## ما الذي تحتاج إليه

- Raspberry Pi 4 أو 5 ‏(يوصى بـ 2GB+)
- بطاقة MicroSD ‏(16GB+) أو USB SSD ‏(أداء أفضل)
- مزود طاقة (يفضّل PSU الرسمي لـ Pi)
- اتصال بالشبكة (Ethernet أو WiFi)
- نحو 30 دقيقة

## 1) كتابة نظام التشغيل

استخدم **Raspberry Pi OS Lite ‏(64-bit)** — لا حاجة إلى سطح مكتب لخادم بدون واجهة.

1. نزّل [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. اختر نظام التشغيل: **Raspberry Pi OS Lite ‏(64-bit)**
3. انقر على أيقونة الترس (⚙️) لإجراء التكوين المسبق:
   - عيّن اسم المضيف: `gateway-host`
   - فعّل SSH
   - عيّن اسم المستخدم/كلمة المرور
   - كوّن WiFi ‏(إذا كنت لا تستخدم Ethernet)
4. اكتب الصورة إلى بطاقة SD / محرك USB
5. أدخلها وأقلع Pi

## 2) الاتصال عبر SSH

```bash
ssh user@gateway-host
# أو استخدم عنوان IP
ssh user@192.168.x.x
```

## 3) إعداد النظام

```bash
# تحديث النظام
sudo apt update && sudo apt upgrade -y

# تثبيت الحزم الأساسية
sudo apt install -y git curl build-essential

# تعيين المنطقة الزمنية (مهم لـ cron/reminders)
sudo timedatectl set-timezone America/Chicago  # غيّرها إلى منطقتك الزمنية
```

## 4) تثبيت Node.js 24 ‏(ARM64)

```bash
# تثبيت Node.js عبر NodeSource
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

# التحقق
node --version  # يجب أن يعرض v24.x.x
npm --version
```

## 5) إضافة Swap ‏(مهم للذاكرة 2GB أو أقل)

تمنع Swap أعطال نفاد الذاكرة:

```bash
# إنشاء ملف swap بحجم 2GB
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# جعله دائمًا
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# تحسين للأجهزة ذات الذاكرة المنخفضة (تقليل swappiness)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 6) تثبيت OpenClaw

### الخيار A: تثبيت قياسي (موصى به)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

### الخيار B: تثبيت قابل للتعديل (للتجريب)

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
npm install
npm run build
npm link
```

يمنحك التثبيت القابل للتعديل وصولًا مباشرًا إلى السجلات والشفرة — وهو مفيد لتصحيح المشكلات الخاصة بـ ARM.

## 7) تشغيل Onboarding

```bash
openclaw onboard --install-daemon
```

اتبع المعالج:

1. **وضع Gateway:** محلي
2. **المصادقة:** يوصى بمفاتيح API ‏(قد تكون OAuth مزعجة على Pi بدون واجهة)
3. **القنوات:** Telegram هي الأسهل للبدء
4. **Daemon:** نعم (systemd)

## 8) التحقق من التثبيت

```bash
# التحقق من الحالة
openclaw status

# التحقق من الخدمة (التثبيت القياسي = وحدة systemd للمستخدم)
systemctl --user status openclaw-gateway.service

# عرض السجلات
journalctl --user -u openclaw-gateway.service -f
```

## 9) الوصول إلى لوحة OpenClaw

استبدل `user@gateway-host` باسم مستخدم Pi لديك واسم المضيف أو عنوان IP.

على جهاز الكمبيوتر الخاص بك، اطلب من Pi طباعة عنوان URL جديد للوحة التحكم:

```bash
ssh user@gateway-host 'openclaw dashboard --no-open'
```

يطبع الأمر `Dashboard URL:`. واعتمادًا على كيفية تكوين `gateway.auth.token`,
قد يكون عنوان URL رابطًا عاديًا على `http://127.0.0.1:18789/` أو رابطًا
يتضمن `#token=...`.

في طرفية أخرى على جهاز الكمبيوتر لديك، أنشئ نفق SSH:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

ثم افتح عنوان Dashboard URL المطبوع في متصفحك المحلي.

إذا طلبت واجهة المستخدم مصادقة بسر مشترك، فألصق الرمز أو كلمة المرور المكوّنة
في إعدادات Control UI. وبالنسبة إلى مصادقة الرمز، استخدم `gateway.auth.token` ‏(أو
`OPENCLAW_GATEWAY_TOKEN`).

للوصول البعيد الدائم، راجع [Tailscale](/gateway/tailscale).

---

## تحسينات الأداء

### استخدم USB SSD ‏(تحسن كبير)

بطاقات SD بطيئة وتتعرض للتلف. ويوفر USB SSD تحسنًا كبيرًا في الأداء:

```bash
# تحقق مما إذا كان الإقلاع يتم من USB
lsblk
```

راجع [دليل إقلاع Pi من USB](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot) من أجل الإعداد.

### تسريع بدء CLI ‏(ذاكرة تخزين مؤقت لترجمة الوحدات)

على مضيفات Pi ذات الطاقة المنخفضة، فعّل ذاكرة التخزين المؤقت لترجمة الوحدات في Node حتى تكون تشغيلات CLI المتكررة أسرع:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

ملاحظات:

- يسرّع `NODE_COMPILE_CACHE` التشغيلات اللاحقة (`status` و`health` و`--help`).
- يبقى `/var/tmp` بعد إعادة التشغيل أكثر من `/tmp`.
- يتجنب `OPENCLAW_NO_RESPAWN=1` تكلفة بدء إضافية من إعادة تشغيل CLI لنفسها.
- يقوم التشغيل الأول بتسخين الذاكرة المؤقتة؛ بينما تستفيد التشغيلات اللاحقة أكثر.

### ضبط بدء تشغيل systemd ‏(اختياري)

إذا كان هذا الـ Pi يشغّل OpenClaw في الأساس، فأضف drop-in للخدمة لتقليل
اهتزاز إعادة التشغيل والحفاظ على استقرار بيئة البدء:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

ثم طبّق ذلك:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway.service
```

إذا أمكن، فأبقِ حالة/ذاكرة التخزين المؤقت الخاصة بـ OpenClaw على تخزين مدعوم بـ SSD لتجنب
اختناقات الإدخال/الإخراج العشوائية الخاصة ببطاقات SD أثناء البدء البارد.

إذا كان هذا Pi بدون واجهة، ففعّل lingering مرة واحدة حتى تبقى خدمة المستخدم
بعد تسجيل الخروج:

```bash
sudo loginctl enable-linger "$(whoami)"
```

كيف تساعد سياسات `Restart=` في الاسترداد الآلي:
[يمكن لـ systemd أتمتة استرداد الخدمة](https://www.redhat.com/en/blog/systemd-automate-recovery).

### تقليل استخدام الذاكرة

```bash
# تعطيل تخصيص ذاكرة GPU (بدون واجهة)
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt

# تعطيل Bluetooth إذا لم يكن مطلوبًا
sudo systemctl disable bluetooth
```

### مراقبة الموارد

```bash
# التحقق من الذاكرة
free -h

# التحقق من درجة حرارة CPU
vcgencmd measure_temp

# المراقبة الحية
htop
```

---

## ملاحظات خاصة بـ ARM

### توافق الثنائيات

تعمل معظم ميزات OpenClaw على ARM64، لكن بعض الثنائيات الخارجية قد تحتاج إلى إصدارات ARM:

| الأداة              | حالة ARM64 | ملاحظات                            |
| ------------------- | ---------- | ---------------------------------- |
| Node.js             | ✅         | يعمل بشكل ممتاز                    |
| WhatsApp ‏(Baileys) | ✅         | JavaScript خالص، لا مشكلات         |
| Telegram            | ✅         | JavaScript خالص، لا مشكلات         |
| gog ‏(Gmail CLI)    | ⚠️         | تحقق من وجود إصدار ARM             |
| Chromium ‏(browser) | ✅         | `sudo apt install chromium-browser` |

إذا فشلت Skill ما، فتحقق مما إذا كان للثنائي الخاص بها إصدار ARM. الكثير من أدوات Go/Rust تملكه؛ وبعضها لا يملكه.

### 32-bit مقابل 64-bit

**استخدم دائمًا نظام تشغيل 64-bit.** فـ Node.js والعديد من الأدوات الحديثة تتطلب ذلك. تحقق باستخدام:

```bash
uname -m
# يجب أن يعرض: aarch64 ‏(64-bit) وليس armv7l ‏(32-bit)
```

---

## إعداد النموذج الموصى به

بما أن Pi هي مجرد Gateway ‏(والنماذج تعمل في السحابة)، فاستخدم النماذج المعتمدة على API:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openai/gpt-5.4-mini"]
      }
    }
  }
}
```

**لا تحاول تشغيل LLMs محليًا على Pi** — حتى النماذج الصغيرة بطيئة جدًا. دع Claude/GPT يقومان بالعمل الثقيل.

---

## البدء التلقائي عند الإقلاع

يقوم Onboarding بإعداد ذلك، لكن للتحقق:

```bash
# تحقق من أن الخدمة مفعلة
systemctl --user is-enabled openclaw-gateway.service

# فعّلها إذا لم تكن كذلك
systemctl --user enable openclaw-gateway.service

# ابدأ التشغيل عند الإقلاع
systemctl --user start openclaw-gateway.service
```

---

## استكشاف الأخطاء وإصلاحها

### نفاد الذاكرة (OOM)

```bash
# تحقق من الذاكرة
free -h

# أضف المزيد من swap ‏(راجع الخطوة 5)
# أو قلّل عدد الخدمات العاملة على Pi
```

### بطء الأداء

- استخدم USB SSD بدلًا من بطاقة SD
- عطّل الخدمات غير المستخدمة: `sudo systemctl disable cups bluetooth avahi-daemon`
- تحقق من خنق CPU: ‏`vcgencmd get_throttled` ‏(يجب أن يعيد `0x0`)

### الخدمة لا تبدأ

```bash
# تحقق من السجلات
journalctl --user -u openclaw-gateway.service --no-pager -n 100

# إصلاح شائع: إعادة البناء
cd ~/openclaw  # إذا كنت تستخدم التثبيت القابل للتعديل
npm run build
systemctl --user restart openclaw-gateway.service
```

### مشكلات ثنائيات ARM

إذا فشلت Skill برسالة "exec format error":

1. تحقق مما إذا كان للثنائي إصدار ARM64
2. حاول البناء من المصدر
3. أو استخدم حاوية Docker مع دعم ARM

### انقطاع WiFi

بالنسبة إلى أجهزة Pi بدون واجهة التي تعمل عبر WiFi:

```bash
# تعطيل إدارة طاقة WiFi
sudo iwconfig wlan0 power off

# جعل ذلك دائمًا
echo 'wireless-power off' | sudo tee -a /etc/network/interfaces
```

---

## مقارنة التكلفة

| الإعداد          | تكلفة لمرة واحدة | التكلفة الشهرية | ملاحظات                      |
| ---------------- | ---------------- | --------------- | ---------------------------- |
| **Pi 4 ‏(2GB)**  | ~$45             | $0              | + الكهرباء (~$5/السنة)       |
| **Pi 4 ‏(4GB)**  | ~$55             | $0              | موصى به                      |
| **Pi 5 ‏(4GB)**  | ~$60             | $0              | أفضل أداء                    |
| **Pi 5 ‏(8GB)**  | ~$80             | $0              | مبالغ فيه لكن مستقبلي        |
| DigitalOcean     | $0               | $6/شهر          | $72/السنة                    |
| Hetzner          | $0               | €3.79/شهر       | ~$50/السنة                   |

**نقطة التعادل:** يدفع Pi تكلفته لنفسه خلال نحو 6-12 شهرًا مقارنةً بـ cloud VPS.

---

## راجع أيضًا

- [دليل Linux](/platforms/linux) — إعداد Linux عام
- [دليل DigitalOcean](/platforms/digitalocean) — بديل سحابي
- [دليل Hetzner](/install/hetzner) — إعداد Docker
- [Tailscale](/gateway/tailscale) — الوصول عن بُعد
- [Nodes](/nodes) — إقران الكمبيوتر المحمول/الهاتف مع Pi gateway
