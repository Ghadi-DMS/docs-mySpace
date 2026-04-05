---
read_when:
    - تريد نشرًا آليًا للخادم مع تقوية أمنية
    - تحتاج إلى إعداد معزول بجدار ناري مع وصول عبر VPN
    - أنت تنشر على خوادم Debian/Ubuntu بعيدة
summary: تثبيت OpenClaw آلي ومحكم باستخدام Ansible وTailscale VPN وعزل الجدار الناري
title: Ansible
x-i18n:
    generated_at: "2026-04-05T12:45:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 27433c3b4afa09406052e428be7b1990476067e47ab8abf7145ff9547b37909a
    source_path: install/ansible.md
    workflow: 15
---

# تثبيت Ansible

انشر OpenClaw على خوادم الإنتاج باستخدام **[openclaw-ansible](https://github.com/openclaw/openclaw-ansible)** -- مُثبّت آلي بهندسة تضع الأمان أولًا.

<Info>
يُعد مستودع [openclaw-ansible](https://github.com/openclaw/openclaw-ansible) المصدر المعتمد لنشر Ansible. وهذه الصفحة مجرد نظرة عامة سريعة.
</Info>

## المتطلبات المسبقة

| المتطلب | التفاصيل                                                  |
| ------- | --------------------------------------------------------- |
| **نظام التشغيل** | Debian 11+ أو Ubuntu 20.04+                               |
| **الوصول**  | صلاحيات root أو sudo                                   |
| **الشبكة** | اتصال بالإنترنت لتثبيت الحزم              |
| **Ansible** | 2.14+ (يُثبت تلقائيًا بواسطة script البدء السريع) |

## ما الذي ستحصل عليه

- **أمان يعتمد على الجدار الناري أولًا** -- عزل UFW + Docker (إتاحة SSH + Tailscale فقط)
- **Tailscale VPN** -- وصول بعيد آمن من دون تعريض الخدمات للعامة
- **Docker** -- حاويات sandbox معزولة وعمليات bind على localhost فقط
- **دفاع متعدد الطبقات** -- بنية أمنية من 4 طبقات
- **تكامل Systemd** -- بدء تلقائي عند الإقلاع مع تقوية أمنية
- **إعداد بأمر واحد** -- نشر كامل خلال دقائق

## البدء السريع

تثبيت بأمر واحد:

```bash
curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw-ansible/main/install.sh | bash
```

## ما الذي سيتم تثبيته

يقوم playbook الخاص بـ Ansible بتثبيت وتكوين ما يلي:

1. **Tailscale** -- VPN شبكي للوصول البعيد الآمن
2. **UFW firewall** -- منافذ SSH + Tailscale فقط
3. **Docker CE + Compose V2** -- من أجل sandboxes الخاصة بالوكلاء
4. **Node.js 24 + pnpm** -- تبعيات وقت التشغيل (ما زال Node 22 LTS، حاليًا `22.14+`، مدعومًا)
5. **OpenClaw** -- على المضيف مباشرة، وليس داخل حاوية
6. **خدمة Systemd** -- بدء تلقائي مع تقوية أمنية

<Note>
تعمل البوابة مباشرة على المضيف (وليست داخل Docker)، لكن sandboxes الخاصة بالوكلاء تستخدم Docker للعزل. راجع [Sandboxing](/gateway/sandboxing) للتفاصيل.
</Note>

## الإعداد بعد التثبيت

<Steps>
  <Step title="انتقل إلى مستخدم openclaw">
    ```bash
    sudo -i -u openclaw
    ```
  </Step>
  <Step title="شغّل معالج الإعداد التفاعلي">
    يرشدك script ما بعد التثبيت خلال تكوين إعدادات OpenClaw.
  </Step>
  <Step title="وصّل موفري المراسلة">
    سجّل الدخول إلى WhatsApp أو Telegram أو Discord أو Signal:
    ```bash
    openclaw channels login
    ```
  </Step>
  <Step title="تحقق من التثبيت">
    ```bash
    sudo systemctl status openclaw
    sudo journalctl -u openclaw -f
    ```
  </Step>
  <Step title="اتصل بـ Tailscale">
    انضم إلى شبكة VPN لديك للوصول البعيد الآمن.
  </Step>
</Steps>

### أوامر سريعة

```bash
# تحقق من حالة الخدمة
sudo systemctl status openclaw

# عرض السجلات المباشرة
sudo journalctl -u openclaw -f

# إعادة تشغيل البوابة
sudo systemctl restart openclaw

# تسجيل دخول المزوّد (شغّله كمستخدم openclaw)
sudo -i -u openclaw
openclaw channels login
```

## البنية الأمنية

يستخدم النشر نموذج دفاع من 4 طبقات:

1. **الجدار الناري (UFW)** -- تعريض SSH (22) + Tailscale (41641/udp) فقط للعامة
2. **VPN (Tailscale)** -- لا يمكن الوصول إلى البوابة إلا عبر شبكة VPN
3. **عزل Docker** -- تمنع سلسلة DOCKER-USER في iptables تعريض المنافذ خارجيًا
4. **تقوية Systemd** -- ‏NoNewPrivileges، وPrivateTmp، ومستخدم غير مميز

للتحقق من سطح الهجوم الخارجي لديك:

```bash
nmap -p- YOUR_SERVER_IP
```

يجب أن يكون المنفذ 22 (SSH) فقط مفتوحًا. أما جميع الخدمات الأخرى (البوابة، وDocker) فهي مقفلة.

يتم تثبيت Docker من أجل sandboxes الخاصة بالوكلاء (تنفيذ الأدوات بشكل معزول)، وليس لتشغيل البوابة نفسها. راجع [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) لتكوين sandbox.

## التثبيت اليدوي

إذا كنت تفضل التحكم اليدوي بدلًا من الأتمتة:

<Steps>
  <Step title="ثبّت المتطلبات المسبقة">
    ```bash
    sudo apt update && sudo apt install -y ansible git
    ```
  </Step>
  <Step title="استنسخ المستودع">
    ```bash
    git clone https://github.com/openclaw/openclaw-ansible.git
    cd openclaw-ansible
    ```
  </Step>
  <Step title="ثبّت مجموعات Ansible">
    ```bash
    ansible-galaxy collection install -r requirements.yml
    ```
  </Step>
  <Step title="شغّل playbook">
    ```bash
    ./run-playbook.sh
    ```

    أو شغّله مباشرةً ثم نفّذ script الإعداد يدويًا بعد ذلك:
    ```bash
    ansible-playbook playbook.yml --ask-become-pass
    # ثم شغّل: /tmp/openclaw-setup.sh
    ```

  </Step>
</Steps>

## التحديث

يُعد مُثبّت Ansible OpenClaw للتحديثات اليدوية. راجع [التحديث](/install/updating) لمعرفة مسار التحديث القياسي.

لإعادة تشغيل playbook الخاص بـ Ansible (على سبيل المثال، لتغييرات التكوين):

```bash
cd openclaw-ansible
./run-playbook.sh
```

هذا الإجراء idempotent وآمن للتشغيل عدة مرات.

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="الجدار الناري يمنع اتصالي">
    - تأكد أولًا من إمكانية الوصول عبر Tailscale VPN
    - يُسمح دائمًا بالوصول عبر SSH (المنفذ 22)
    - لا يمكن الوصول إلى البوابة إلا عبر Tailscale حسب التصميم
  </Accordion>
  <Accordion title="الخدمة لا تبدأ">
    ```bash
    # تحقق من السجلات
    sudo journalctl -u openclaw -n 100

    # تحقق من الأذونات
    sudo ls -la /opt/openclaw

    # اختبر التشغيل اليدوي
    sudo -i -u openclaw
    cd ~/openclaw
    openclaw gateway run
    ```

  </Accordion>
  <Accordion title="مشكلات Docker sandbox">
    ```bash
    # تحقق من أن Docker يعمل
    sudo systemctl status docker

    # تحقق من صورة sandbox
    sudo docker images | grep openclaw-sandbox

    # ابنِ صورة sandbox إذا كانت مفقودة
    cd /opt/openclaw/openclaw
    sudo -u openclaw ./scripts/sandbox-setup.sh
    ```

  </Accordion>
  <Accordion title="فشل تسجيل دخول المزوّد">
    تأكد من أنك تعمل كمستخدم `openclaw`:
    ```bash
    sudo -i -u openclaw
    openclaw channels login
    ```
  </Accordion>
</AccordionGroup>

## التكوين المتقدم

للحصول على بنية أمنية مفصلة واستكشاف الأخطاء وإصلاحها، راجع مستودع openclaw-ansible:

- [البنية الأمنية](https://github.com/openclaw/openclaw-ansible/blob/main/docs/security.md)
- [التفاصيل التقنية](https://github.com/openclaw/openclaw-ansible/blob/main/docs/architecture.md)
- [دليل استكشاف الأخطاء وإصلاحها](https://github.com/openclaw/openclaw-ansible/blob/main/docs/troubleshooting.md)

## ذو صلة

- [openclaw-ansible](https://github.com/openclaw/openclaw-ansible) -- دليل النشر الكامل
- [Docker](/install/docker) -- إعداد البوابة داخل حاوية
- [Sandboxing](/gateway/sandboxing) -- تكوين sandbox الخاصة بالوكلاء
- [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) -- العزل لكل وكيل
