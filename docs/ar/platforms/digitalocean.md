---
read_when:
    - إعداد OpenClaw على DigitalOcean
    - البحث عن استضافة VPS رخيصة لـ OpenClaw
summary: OpenClaw على DigitalOcean (خيار VPS مدفوع بسيط)
title: DigitalOcean (المنصة)
x-i18n:
    generated_at: "2026-04-05T12:49:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6ee4ad84c421f87064534a4fb433df1f70304502921841ec618318ed862d4092
    source_path: platforms/digitalocean.md
    workflow: 15
---

# OpenClaw على DigitalOcean

## الهدف

تشغيل بوابة OpenClaw دائمة على DigitalOcean مقابل **6 دولارات شهريًا** (أو 4 دولارات شهريًا مع التسعير المحجوز).

إذا كنت تريد خيارًا بقيمة 0 دولار شهريًا ولا تمانع ARM + إعدادًا خاصًا بالموفر، فراجع [دليل Oracle Cloud](/platforms/oracle).

## مقارنة التكلفة (2026)

| الموفّر | الخطة | المواصفات | السعر/شهريًا | ملاحظات |
| ------------ | --------------- | ---------------------- | ----------- | ------------------------------------- |
| Oracle Cloud | Always Free ARM | حتى 4 OCPU و24GB RAM | $0 | ARM، سعة محدودة / تعقيدات في التسجيل |
| Hetzner      | CX22            | 2 vCPU، 4GB RAM        | €3.79 (~$4) | أرخص خيار مدفوع |
| DigitalOcean | Basic           | 1 vCPU، 1GB RAM        | $6          | واجهة سهلة، ووثائق جيدة |
| Vultr        | Cloud Compute   | 1 vCPU، 1GB RAM        | $6          | مواقع كثيرة |
| Linode       | Nanode          | 1 vCPU، 1GB RAM        | $5          | أصبحت الآن جزءًا من Akamai |

**اختيار الموفّر:**

- DigitalOcean: أبسط تجربة استخدام + إعداد متوقع (هذا الدليل)
- Hetzner: سعر/أداء جيد (راجع [دليل Hetzner](/install/hetzner))
- Oracle Cloud: قد يكون بسعر 0 دولار شهريًا، لكنه أكثر حساسية ويعتمد على ARM فقط (راجع [دليل Oracle](/platforms/oracle))

---

## المتطلبات الأساسية

- حساب DigitalOcean ‏([التسجيل مع رصيد مجاني 200 دولار](https://m.do.co/c/signup))
- زوج مفاتيح SSH (أو الاستعداد لاستخدام مصادقة كلمة المرور)
- حوالي 20 دقيقة

## 1) إنشاء Droplet

<Warning>
استخدم صورة أساسية نظيفة (Ubuntu 24.04 LTS). تجنب صور Marketplace بنقرة واحدة التابعة لجهات خارجية ما لم تكن قد راجعت نصوص بدء التشغيل والإعدادات الافتراضية للجدار الناري الخاصة بها.
</Warning>

1. سجّل الدخول إلى [DigitalOcean](https://cloud.digitalocean.com/)
2. انقر **Create → Droplets**
3. اختر:
   - **المنطقة:** الأقرب إليك (أو إلى مستخدميك)
   - **الصورة:** Ubuntu 24.04 LTS
   - **الحجم:** Basic → Regular → **6 دولارات/شهريًا** (1 vCPU و1GB RAM و25GB SSD)
   - **المصادقة:** مفتاح SSH (مستحسن) أو كلمة مرور
4. انقر **Create Droplet**
5. دوّن عنوان IP

## 2) الاتصال عبر SSH

```bash
ssh root@YOUR_DROPLET_IP
```

## 3) تثبيت OpenClaw

```bash
# تحديث النظام
apt update && apt upgrade -y

# تثبيت Node.js 24
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt install -y nodejs

# تثبيت OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# التحقق
openclaw --version
```

## 4) تشغيل الإعداد الأولي

```bash
openclaw onboard --install-daemon
```

سيرشدك المعالج خلال:

- مصادقة النموذج (مفاتيح API أو OAuth)
- إعداد القناة (Telegram، وWhatsApp، وDiscord، وغير ذلك)
- رمز البوابة (يتم توليده تلقائيًا)
- تثبيت daemon ‏(systemd)

## 5) التحقق من البوابة

```bash
# التحقق من الحالة
openclaw status

# التحقق من الخدمة
systemctl --user status openclaw-gateway.service

# عرض السجلات
journalctl --user -u openclaw-gateway.service -f
```

## 6) الوصول إلى لوحة التحكم

ترتبط البوابة بـ loopback افتراضيًا. للوصول إلى Control UI:

**الخيار A: نفق SSH (مستحسن)**

```bash
# من جهازك المحلي
ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP

# ثم افتح: http://localhost:18789
```

**الخيار B: Tailscale Serve ‏(HTTPS، loopback-only)**

```bash
# على droplet
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# إعداد البوابة لاستخدام Tailscale Serve
openclaw config set gateway.tailscale.mode serve
openclaw gateway restart
```

افتح: `https://<magicdns>/`

ملاحظات:

- يُبقي Serve البوابة في وضع loopback-only ويصادق حركة Control UI/WebSocket عبر رؤوس هوية Tailscale (تفترض المصادقة من دون رمز مضيف بوابة موثوقًا؛ ولا تستخدم HTTP APIs رؤوس Tailscale هذه، بل تتبع وضع HTTP auth العادي للبوابة).
- لطلب بيانات اعتماد صريحة تعتمد على سر مشترك بدلًا من ذلك، اضبط `gateway.auth.allowTailscale: false` واستخدم `gateway.auth.mode: "token"` أو `"password"`.

**الخيار C: ربط tailnet ‏(من دون Serve)**

```bash
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

افتح: `http://<tailscale-ip>:18789` ‏(الرمز مطلوب).

## 7) توصيل القنوات

### Telegram

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### WhatsApp

```bash
openclaw channels login whatsapp
# امسح رمز QR
```

راجع [القنوات](/channels) بالنسبة إلى الموفّرين الآخرين.

---

## تحسينات لذاكرة 1GB RAM

لا يملك Droplet بقيمة 6 دولارات سوى 1GB RAM. ولإبقاء الأمور تعمل بسلاسة:

### أضف swap ‏(مستحسن)

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### استخدم نموذجًا أخف

إذا كنت تواجه OOMs، ففكّر في:

- استخدام نماذج معتمدة على API ‏(Claude، GPT) بدلًا من النماذج المحلية
- ضبط `agents.defaults.model.primary` على نموذج أصغر

### راقب الذاكرة

```bash
free -h
htop
```

---

## الاستمرارية

توجد كل الحالة في:

- `~/.openclaw/` — `openclaw.json`، و`auth-profiles.json` لكل وكيل، وحالة القناة/الموفّر، وبيانات الجلسات
- `~/.openclaw/workspace/` — مساحة العمل (`SOUL.md`، والذاكرة، وغير ذلك)

تستمر هذه البيانات بعد إعادة التشغيل. انسخها احتياطيًا بشكل دوري:

```bash
openclaw backup create
```

---

## البديل المجاني Oracle Cloud

يوفر Oracle Cloud مثيلات ARM من فئة **Always Free** أقوى بكثير من أي خيار مدفوع هنا — مقابل 0 دولار شهريًا.

| ما الذي تحصل عليه | المواصفات |
| ----------------- | ---------------------- |
| **4 OCPUs**       | ARM Ampere A1          |
| **24GB RAM**      | أكثر من كافٍ |
| **200GB storage** | Block volume           |
| **مجاني دائمًا**  | لا توجد رسوم على بطاقة الائتمان |

**المحاذير:**

- قد يكون التسجيل حساسًا (أعد المحاولة إذا فشل)
- بنية ARM — معظم الأشياء تعمل، لكن بعض الملفات التنفيذية تحتاج إلى نسخ ARM

للحصول على دليل الإعداد الكامل، راجع [Oracle Cloud](/platforms/oracle). وللحصول على نصائح التسجيل واستكشاف أخطاء عملية الانضمام وإصلاحها، راجع [دليل المجتمع](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd).

---

## استكشاف الأخطاء وإصلاحها

### البوابة لا تبدأ

```bash
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service --no-pager -n 50
```

### المنفذ مستخدم بالفعل

```bash
lsof -i :18789
kill <PID>
```

### نفاد الذاكرة

```bash
# تحقق من الذاكرة
free -h

# أضف المزيد من swap
# أو قم بالترقية إلى Droplet بقيمة 12 دولارًا/شهريًا (2GB RAM)
```

---

## راجع أيضًا

- [دليل Hetzner](/install/hetzner) — أرخص وأكثر قوة
- [تثبيت Docker](/install/docker) — إعداد داخل حاوية
- [Tailscale](/gateway/tailscale) — وصول بعيد آمن
- [الإعداد](/gateway/configuration) — مرجع الإعداد الكامل
