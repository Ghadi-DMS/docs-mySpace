---
read_when:
    - إعداد OpenClaw على Oracle Cloud
    - البحث عن استضافة VPS منخفضة التكلفة لـ OpenClaw
    - تريد تشغيل OpenClaw على مدار الساعة طوال أيام الأسبوع على خادم صغير
summary: OpenClaw على Oracle Cloud ‏(Always Free ARM)
title: Oracle Cloud (المنصة)
x-i18n:
    generated_at: "2026-04-05T12:51:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3a42cdf2d18e964123894d382d2d8052c6b8dbb0b3c7dac914477c4a2a0a244f
    source_path: platforms/oracle.md
    workflow: 15
---

# OpenClaw على Oracle Cloud ‏(OCI)

## الهدف

تشغيل Gateway دائم لـ OpenClaw على طبقة ARM **Always Free** من Oracle Cloud.

يمكن أن تكون الطبقة المجانية من Oracle مناسبة جدًا لـ OpenClaw (خصوصًا إذا كان لديك بالفعل حساب OCI)، لكنها تأتي مع بعض المقايضات:

- معمارية ARM ‏(تعمل معظم الأشياء، لكن بعض الملفات التنفيذية قد تكون x86 فقط)
- قد تكون السعة والتسجيل حساسين بعض الشيء

## مقارنة التكلفة (2026)

| الموفّر | الخطة | المواصفات | السعر/الشهر | ملاحظات |
| ------------ | --------------- | ---------------------- | -------- | --------------------- |
| Oracle Cloud | Always Free ARM | حتى 4 OCPU و24GB RAM | $0 | ARM، وسعة محدودة |
| Hetzner | CX22 | 2 vCPU و4GB RAM | ~ $4 | أرخص خيار مدفوع |
| DigitalOcean | Basic | 1 vCPU و1GB RAM | $6 | واجهة سهلة ومستندات جيدة |
| Vultr | Cloud Compute | 1 vCPU و1GB RAM | $6 | مواقع كثيرة |
| Linode | Nanode | 1 vCPU و1GB RAM | $5 | أصبحت الآن جزءًا من Akamai |

---

## المتطلبات المسبقة

- حساب Oracle Cloud ‏([التسجيل](https://www.oracle.com/cloud/free/)) — راجع [دليل التسجيل المجتمعي](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) إذا واجهت مشكلات
- حساب Tailscale ‏(مجاني على [tailscale.com](https://tailscale.com))
- حوالي 30 دقيقة

## 1) إنشاء مثيل OCI

1. سجّل الدخول إلى [Oracle Cloud Console](https://cloud.oracle.com/)
2. انتقل إلى **Compute → Instances → Create Instance**
3. اضبط ما يلي:
   - **الاسم:** `openclaw`
   - **الصورة:** Ubuntu 24.04 ‏(aarch64)
   - **الشكل:** `VM.Standard.A1.Flex` ‏(Ampere ARM)
   - **OCPUs:** ‏2 ‏(أو حتى 4)
   - **الذاكرة:** ‏12 GB ‏(أو حتى 24 GB)
   - **قرص الإقلاع:** ‏50 GB ‏(حتى 200 GB مجانًا)
   - **مفتاح SSH:** أضف مفتاحك العام
4. انقر **Create**
5. دوّن عنوان IP العام

**نصيحة:** إذا فشل إنشاء المثيل مع الرسالة "Out of capacity"، فجرب نطاق توفر مختلفًا أو أعد المحاولة لاحقًا. سعة الطبقة المجانية محدودة.

## 2) الاتصال والتحديث

```bash
# الاتصال عبر عنوان IP العام
ssh ubuntu@YOUR_PUBLIC_IP

# تحديث النظام
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential
```

**ملاحظة:** الحزمة `build-essential` مطلوبة لتجميع بعض الاعتمادات على ARM.

## 3) تهيئة المستخدم واسم المضيف

```bash
# تعيين اسم المضيف
sudo hostnamectl set-hostname openclaw

# تعيين كلمة مرور لمستخدم ubuntu
sudo passwd ubuntu

# تمكين lingering ‏(يبقي خدمات المستخدم تعمل بعد تسجيل الخروج)
sudo loginctl enable-linger ubuntu
```

## 4) تثبيت Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw
```

يؤدي ذلك إلى تمكين Tailscale SSH، بحيث يمكنك الاتصال عبر `ssh openclaw` من أي جهاز على tailnet لديك — من دون الحاجة إلى عنوان IP عام.

تحقق:

```bash
tailscale status
```

**من الآن فصاعدًا، اتصل عبر Tailscale:** ‏`ssh ubuntu@openclaw` ‏(أو استخدم عنوان IP الخاص بـ Tailscale).

## 5) تثبيت OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
source ~/.bashrc
```

عندما تظهر المطالبة "How do you want to hatch your bot?"، اختر **"Do this later"**.

> ملاحظة: إذا واجهت مشكلات بناء أصلية على ARM، فابدأ بحزم النظام (مثل `sudo apt install -y build-essential`) قبل اللجوء إلى Homebrew.

## 6) تهيئة Gateway ‏(loopback + مصادقة token) وتمكين Tailscale Serve

استخدم مصادقة token كخيار افتراضي. فهي متوقعة وتجنب الحاجة إلى أي علامات “insecure auth” خاصة بـ Control UI.

```bash
# إبقاء Gateway خاصة داخل الجهاز الافتراضي
openclaw config set gateway.bind loopback

# فرض المصادقة على Gateway + Control UI
openclaw config set gateway.auth.mode token
openclaw doctor --generate-gateway-token

# الكشف عبر Tailscale Serve ‏(HTTPS + وصول عبر tailnet)
openclaw config set gateway.tailscale.mode serve
openclaw config set gateway.trustedProxies '["127.0.0.1"]'

systemctl --user restart openclaw-gateway.service
```

تُستخدم `gateway.trustedProxies=["127.0.0.1"]` هنا فقط لمعالجة forwarded-IP/local-client الخاصة بوكيل Tailscale Serve المحلي. وهي **ليست** `gateway.auth.mode: "trusted-proxy"`. وتحافظ مسارات عارض الفروق Diff viewer على سلوك fail-closed في هذا الإعداد: قد تعيد طلبات العارض الخام على `127.0.0.1` من دون ترويسات وكيل محوّلة الرسالة `Diff not found`. استخدم `mode=file` / `mode=both` للمرفقات، أو فعّل عمدًا العارضات البعيدة واضبط `plugins.entries.diffs.config.viewerBaseUrl` ‏(أو مرّر proxy ‏`baseUrl`) إذا كنت تحتاج إلى روابط عارض قابلة للمشاركة.

## 7) التحقق

```bash
# التحقق من الإصدار
openclaw --version

# التحقق من حالة الخدمة
systemctl --user status openclaw-gateway.service

# التحقق من Tailscale Serve
tailscale serve status

# اختبار الاستجابة المحلية
curl http://localhost:18789
```

## 8) تشديد أمان VCN

الآن وبعد أن أصبح كل شيء يعمل، شدد VCN لحظر كل حركة المرور باستثناء Tailscale. تعمل Oracle Virtual Cloud Network كجدار ناري عند حافة الشبكة — إذ تُحظر الحركة قبل أن تصل إلى المثيل.

1. انتقل إلى **Networking → Virtual Cloud Networks** في OCI Console
2. انقر VCN الخاصة بك → **Security Lists** → Default Security List
3. **أزل** كل قواعد الدخول باستثناء:
   - `0.0.0.0/0 UDP 41641` ‏(Tailscale)
4. احتفظ بقواعد الخروج الافتراضية ‏(السماح بكل الخروج)

يؤدي ذلك إلى حظر SSH على المنفذ 22 وHTTP وHTTPS وكل شيء آخر عند حافة الشبكة. ومن الآن فصاعدًا، لا يمكنك الاتصال إلا عبر Tailscale.

---

## الوصول إلى Control UI

من أي جهاز على شبكة Tailscale الخاصة بك:

```
https://openclaw.<tailnet-name>.ts.net/
```

استبدل `<tailnet-name>` باسم tailnet لديك (الظاهر في `tailscale status`).

لا حاجة إلى نفق SSH. إذ يوفر Tailscale:

- تشفير HTTPS ‏(شهادات تلقائية)
- المصادقة عبر هوية Tailscale
- الوصول من أي جهاز على tailnet لديك (حاسوب محمول، هاتف، وما إلى ذلك)

---

## الأمان: ‏VCN + Tailscale ‏(خط أساس موصى به)

مع تشديد VCN ‏(فتح UDP 41641 فقط) وربط Gateway بـ loopback، تحصل على دفاع قوي متعدد الطبقات: تُحظر الحركة العامة عند حافة الشبكة، ويحدث وصول الإدارة عبر tailnet الخاصة بك.

غالبًا ما يلغي هذا الإعداد _الحاجة_ إلى قواعد جدار ناري إضافية على المضيف فقط لمنع هجمات SSH العشوائية على مستوى الإنترنت — لكن ما زال ينبغي عليك تحديث نظام التشغيل، وتشغيل `openclaw security audit`، والتحقق من أنك لا تستمع بالخطأ على واجهات عامة.

### ما هو محمي بالفعل

| الخطوة التقليدية | هل هي مطلوبة؟ | لماذا |
| ------------------ | ----------- | ---------------------------------------------------------------------------- |
| جدار ناري UFW | لا | تحظر VCN الحركة قبل أن تصل إلى المثيل |
| fail2ban | لا | لا توجد هجمات brute force إذا كان المنفذ 22 محظورًا في VCN |
| تشديد sshd | لا | لا يستخدم Tailscale SSH الخدمة sshd |
| تعطيل تسجيل دخول root | لا | يستخدم Tailscale هوية Tailscale، وليس مستخدمي النظام |
| مصادقة SSH بالمفاتيح فقط | لا | يصادق Tailscale عبر tailnet لديك |
| تشديد IPv6 | عادة لا | يعتمد على إعدادات VCN/subnet لديك؛ تحقق مما تم تخصيصه/كشفه فعليًا |

### ما زال موصى به

- **أذونات بيانات الاعتماد:** ‏`chmod 700 ~/.openclaw`
- **تدقيق الأمان:** ‏`openclaw security audit`
- **تحديثات النظام:** ‏`sudo apt update && sudo apt upgrade` بانتظام
- **مراقبة Tailscale:** راجع الأجهزة في [Tailscale admin console](https://login.tailscale.com/admin)

### التحقق من الوضع الأمني

```bash
# التأكد من عدم وجود منافذ عامة تستمع
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# التحقق من أن Tailscale SSH نشط
tailscale status | grep -q 'offers: ssh' && echo "Tailscale SSH active"

# اختياري: تعطيل sshd بالكامل
sudo systemctl disable --now ssh
```

---

## حل احتياطي: نفق SSH

إذا لم يكن Tailscale Serve يعمل، فاستخدم نفق SSH:

```bash
# من جهازك المحلي (عبر Tailscale)
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

ثم افتح `http://localhost:18789`.

---

## استكشاف الأخطاء وإصلاحها

### فشل إنشاء المثيل ("Out of capacity")

مثيلات ARM في الطبقة المجانية شائعة الاستخدام. جرّب ما يلي:

- نطاق توفر مختلف
- إعادة المحاولة في أوقات انخفاض الضغط (الصباح الباكر)
- استخدم عامل التصفية "Always Free" عند اختيار الشكل

### تعذر اتصال Tailscale

```bash
# التحقق من الحالة
sudo tailscale status

# إعادة المصادقة
sudo tailscale up --ssh --hostname=openclaw --reset
```

### تعذر بدء Gateway

```bash
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service -n 50
```

### تعذر الوصول إلى Control UI

```bash
# التحقق من أن Tailscale Serve يعمل
tailscale serve status

# التحقق من أن gateway تستمع
curl http://localhost:18789

# إعادة التشغيل عند الحاجة
systemctl --user restart openclaw-gateway.service
```

### مشكلات الملفات التنفيذية على ARM

قد لا تملك بعض الأدوات بنيات ARM. تحقق من:

```bash
uname -m  # يجب أن يعرض aarch64
```

تعمل معظم حزم npm بشكل جيد. أما بالنسبة إلى الملفات التنفيذية، فابحث عن إصدارات `linux-arm64` أو `aarch64`.

---

## الاستمرارية

توجد كل الحالة في:

- `~/.openclaw/` — ‏`openclaw.json`، و`auth-profiles.json` لكل وكيل، وحالة القناة/الموفّر، وبيانات الجلسات
- `~/.openclaw/workspace/` — ‏workspace ‏(`SOUL.md`، والذاكرة، والقطع الأثرية)

خذ نسخًا احتياطية دوريًا:

```bash
openclaw backup create
```

---

## راجع أيضًا

- [الوصول البعيد إلى Gateway](/gateway/remote) — أنماط الوصول البعيد الأخرى
- [تكامل Tailscale](/gateway/tailscale) — مستندات Tailscale الكاملة
- [تهيئة Gateway](/gateway/configuration) — جميع خيارات الإعداد
- [دليل DigitalOcean](/platforms/digitalocean) — إذا كنت تريد خيارًا مدفوعًا + تسجيلًا أسهل
- [دليل Hetzner](/install/hetzner) — بديل قائم على Docker
