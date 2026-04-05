---
read_when:
    - إعداد OpenClaw على Oracle Cloud
    - البحث عن استضافة VPS مجانية لـ OpenClaw
    - تريد تشغيل OpenClaw على مدار الساعة طوال أيام الأسبوع على خادم صغير
summary: استضافة OpenClaw على فئة ARM المجانية الدائمة من Oracle Cloud
title: Oracle Cloud
x-i18n:
    generated_at: "2026-04-05T12:48:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6915f8c428cfcbc215ba6547273df6e7b93212af6590827a3853f15617ba245e
    source_path: install/oracle.md
    workflow: 15
---

# Oracle Cloud

شغّل OpenClaw Gateway دائمة على فئة ARM **Always Free** في Oracle Cloud ‏(حتى 4 OCPU و24 GB RAM و200 GB تخزين) مجانًا.

## المتطلبات الأساسية

- حساب Oracle Cloud ‏([التسجيل](https://www.oracle.com/cloud/free/)) -- راجع [دليل التسجيل المجتمعي](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) إذا واجهت مشكلات
- حساب Tailscale (مجاني على [tailscale.com](https://tailscale.com))
- زوج مفاتيح SSH
- نحو 30 دقيقة

## الإعداد

<Steps>
  <Step title="إنشاء مثيل OCI">
    1. سجّل الدخول إلى [Oracle Cloud Console](https://cloud.oracle.com/).
    2. انتقل إلى **Compute > Instances > Create Instance**.
    3. اضبط:
       - **Name:** ‏`openclaw`
       - **Image:** ‏Ubuntu 24.04 ‏(aarch64)
       - **Shape:** ‏`VM.Standard.A1.Flex` ‏(Ampere ARM)
       - **OCPUs:** ‏2 (أو حتى 4)
       - **Memory:** ‏12 GB (أو حتى 24 GB)
       - **Boot volume:** ‏50 GB (حتى 200 GB مجانًا)
       - **SSH key:** أضف مفتاحك العام
    4. انقر **Create** ودوّن عنوان IP العام.

    <Tip>
    إذا فشل إنشاء المثيل برسالة "Out of capacity"، فجرّب نطاق توفر مختلفًا أو أعد المحاولة لاحقًا. سعة الفئة المجانية محدودة.
    </Tip>

  </Step>

  <Step title="الاتصال وتحديث النظام">
    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    يتطلب `build-essential` ترجمة ARM لبعض التبعيات.

  </Step>

  <Step title="ضبط المستخدم واسم المضيف">
    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    يؤدي تمكين linger إلى إبقاء خدمات المستخدم قيد التشغيل بعد تسجيل الخروج.

  </Step>

  <Step title="تثبيت Tailscale">
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    من الآن فصاعدًا، اتصل عبر Tailscale: ‏`ssh ubuntu@openclaw`.

  </Step>

  <Step title="تثبيت OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    عند ظهور المطالبة "How do you want to hatch your bot?"، اختر **Do this later**.

  </Step>

  <Step title="ضبط gateway">
    استخدم مصادقة الرمز مع Tailscale Serve للوصول البعيد الآمن.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw config set gateway.auth.mode token
    openclaw doctor --generate-gateway-token
    openclaw config set gateway.tailscale.mode serve
    openclaw config set gateway.trustedProxies '["127.0.0.1"]'

    systemctl --user restart openclaw-gateway.service
    ```

    تكون `gateway.trustedProxies=["127.0.0.1"]` هنا فقط لمعالجة IP المُمرَّر/العميل المحلي الخاصة بـ proxy المحلي في Tailscale Serve. وهي **ليست** `gateway.auth.mode: "trusted-proxy"`. تحافظ مسارات عارض الفروقات على سلوك الفشل المغلق في هذا الإعداد: يمكن أن تعيد طلبات العارض الخام إلى `127.0.0.1` من دون ترويسات proxy ممرَّرة القيمة `Diff not found`. استخدم `mode=file` / `mode=both` للمرفقات، أو فعّل العارضات البعيدة عمدًا واضبط `plugins.entries.diffs.config.viewerBaseUrl` (أو مرّر `baseUrl` خاصة بـ proxy) إذا كنت تحتاج إلى روابط عارض قابلة للمشاركة.

  </Step>

  <Step title="تشديد أمان VCN">
    احظر كل حركة المرور باستثناء Tailscale عند حافة الشبكة:

    1. انتقل إلى **Networking > Virtual Cloud Networks** في OCI Console.
    2. انقر VCN الخاصة بك، ثم **Security Lists > Default Security List**.
    3. **أزل** جميع قواعد ingress باستثناء `0.0.0.0/0 UDP 41641` ‏(Tailscale).
    4. أبقِ قواعد egress الافتراضية (السماح بكل الحركة الصادرة).

    يؤدي ذلك إلى حظر SSH على المنفذ 22 وHTTP وHTTPS وكل شيء آخر عند حافة الشبكة. ولن يمكنك الاتصال إلا عبر Tailscale من هذه النقطة فصاعدًا.

  </Step>

  <Step title="التحقق">
    ```bash
    openclaw --version
    systemctl --user status openclaw-gateway.service
    tailscale serve status
    curl http://localhost:18789
    ```

    ادخل إلى Control UI من أي جهاز على tailnet الخاصة بك:

    ```
    https://openclaw.<tailnet-name>.ts.net/
    ```

    استبدل `<tailnet-name>` باسم tailnet لديك (الظاهر في `tailscale status`).

  </Step>
</Steps>

## الرجوع الاحتياطي: نفق SSH

إذا لم يكن Tailscale Serve يعمل، فاستخدم نفق SSH من جهازك المحلي:

```bash
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

ثم افتح `http://localhost:18789`.

## استكشاف الأخطاء وإصلاحها

**فشل إنشاء المثيل ("Out of capacity")** -- تحظى مثيلات ARM المجانية بشعبية. جرّب نطاق توفر مختلفًا أو أعد المحاولة خلال ساعات انخفاض الضغط.

**Tailscale لا يتصل** -- شغّل `sudo tailscale up --ssh --hostname=openclaw --reset` لإعادة المصادقة.

**لا تبدأ Gateway** -- شغّل `openclaw doctor --non-interactive` وتحقق من السجلات باستخدام `journalctl --user -u openclaw-gateway.service -n 50`.

**مشكلات ثنائيات ARM** -- تعمل معظم حزم npm على ARM64. بالنسبة إلى الثنائيات الأصلية، ابحث عن إصدارات `linux-arm64` أو `aarch64`. تحقق من البنية باستخدام `uname -m`.

## الخطوات التالية

- [Channels](/channels) -- صِل Telegram وWhatsApp وDiscord والمزيد
- [Gateway configuration](/gateway/configuration) -- جميع خيارات التكوين
- [Updating](/install/updating) -- حافظ على OpenClaw محدّثًا
