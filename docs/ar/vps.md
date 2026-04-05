---
read_when:
    - تريد تشغيل Gateway على خادم Linux أو VPS سحابي
    - تحتاج إلى خريطة سريعة لأدلة الاستضافة
    - تريد ضبطًا عامًا لخادم Linux من أجل OpenClaw
sidebarTitle: Linux Server
summary: شغّل OpenClaw على خادم Linux أو VPS سحابي — اختيار المزوّد، والبنية، والضبط
title: خادم Linux
x-i18n:
    generated_at: "2026-04-05T13:00:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7f2f26bbc116841a29055850ed5f491231554b90539bcbf91a6b519875d494fb
    source_path: vps.md
    workflow: 15
---

# خادم Linux

شغّل OpenClaw Gateway على أي خادم Linux أو VPS سحابي. تساعدك هذه الصفحة على
اختيار مزوّد، وتشرح كيفية عمل عمليات النشر السحابية، وتغطي ضبط Linux
العام الذي ينطبق في كل مكان.

## اختر مزوّدًا

<CardGroup cols={2}>
  <Card title="Railway" href="/ar/install/railway">إعداد بنقرة واحدة عبر المتصفح</Card>
  <Card title="Northflank" href="/ar/install/northflank">إعداد بنقرة واحدة عبر المتصفح</Card>
  <Card title="DigitalOcean" href="/ar/install/digitalocean">VPS مدفوع بسيط</Card>
  <Card title="Oracle Cloud" href="/ar/install/oracle">فئة ARM مجانية دائمًا</Card>
  <Card title="Fly.io" href="/ar/install/fly">Fly Machines</Card>
  <Card title="Hetzner" href="/ar/install/hetzner">Docker على Hetzner VPS</Card>
  <Card title="GCP" href="/ar/install/gcp">Compute Engine</Card>
  <Card title="Azure" href="/ar/install/azure">آلة Linux افتراضية</Card>
  <Card title="exe.dev" href="/ar/install/exe-dev">آلة افتراضية مع وكيل HTTPS</Card>
  <Card title="Raspberry Pi" href="/ar/install/raspberry-pi">استضافة ذاتية ARM</Card>
</CardGroup>

يعمل **AWS (EC2 / Lightsail / free tier)** أيضًا بشكل جيد.
يتوفر شرح فيديو من المجتمع على
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
(مورد مجتمعي -- قد يصبح غير متاح).

## كيف تعمل الإعدادات السحابية

- يعمل **Gateway على VPS** ويمتلك الحالة + مساحة العمل.
- تتصل من حاسوبك المحمول أو هاتفك عبر **Control UI** أو **Tailscale/SSH**.
- تعامل مع VPS بوصفه مصدر الحقيقة وقم **بنسخ احتياطي** للحالة + مساحة العمل بانتظام.
- الإعداد الآمن الافتراضي: أبقِ Gateway على loopback وادخل إليه عبر نفق SSH أو Tailscale Serve.
  وإذا ربطته بـ `lan` أو `tailnet`، فاشترط `gateway.auth.token` أو `gateway.auth.password`.

الصفحات ذات الصلة: [الوصول عن بُعد إلى Gateway](/ar/gateway/remote)، [مركز المنصات](/ar/platforms).

## وكيل شركة مشترك على VPS

يُعد تشغيل وكيل واحد لفريق إعدادًا صالحًا عندما يكون كل مستخدم داخل حد الثقة نفسه ويكون الوكيل مخصصًا للأعمال فقط.

- أبقه على وقت تشغيل مخصص (VPS/VM/حاوية + مستخدم/حسابات نظام تشغيل مخصصة).
- لا تسجّل دخول وقت التشغيل هذا إلى حسابات Apple/Google الشخصية أو إلى ملفات تعريف المتصفح/مدير كلمات المرور الشخصية.
- إذا كان المستخدمون خصومًا لبعضهم البعض، فقسّمهم حسب gateway/المضيف/مستخدم نظام التشغيل.

تفاصيل نموذج الأمان: [الأمان](/ar/gateway/security).

## استخدام nodes مع VPS

يمكنك إبقاء Gateway في السحابة وإقران **nodes** على أجهزتك المحلية
(Mac/iOS/Android/headless). توفر nodes قدرات الشاشة/الكاميرا/canvas المحلية و`system.run`
بينما يبقى Gateway في السحابة.

الوثائق: [Nodes](/ar/nodes)، [Nodes CLI](/cli/nodes).

## ضبط بدء التشغيل للآلات الافتراضية الصغيرة ومضيفي ARM

إذا كانت أوامر CLI تبدو بطيئة على الآلات الافتراضية منخفضة الطاقة (أو مضيفي ARM)، فعّل ذاكرة التخزين المؤقت لترجمة الوحدات في Node:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- يحسن `NODE_COMPILE_CACHE` أوقات بدء التشغيل للأوامر المتكررة.
- يتجنب `OPENCLAW_NO_RESPAWN=1` الحمل الزائد الإضافي عند بدء التشغيل الناتج عن مسار إعادة التشغيل الذاتي.
- يؤدي تشغيل الأمر لأول مرة إلى تدفئة الذاكرة المؤقتة؛ وتكون التشغيلات اللاحقة أسرع.
- للاطلاع على تفاصيل Raspberry Pi، راجع [Raspberry Pi](/ar/install/raspberry-pi).

### قائمة التحقق من ضبط systemd ‏(اختياري)

بالنسبة إلى مضيفي VM الذين يستخدمون `systemd`، ضع في اعتبارك ما يلي:

- أضف متغيرات بيئة للخدمة من أجل مسار بدء تشغيل مستقر:
  - `OPENCLAW_NO_RESPAWN=1`
  - `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- اجعل سلوك إعادة التشغيل صريحًا:
  - `Restart=always`
  - `RestartSec=2`
  - `TimeoutStartSec=90`
- فضّل الأقراص المدعومة بـ SSD لمسارات الحالة/الذاكرة المؤقتة لتقليل عقوبات البدء البارد الناتجة عن الإدخال/الإخراج العشوائي.

بالنسبة إلى المسار القياسي `openclaw onboard --install-daemon`، عدّل وحدة المستخدم:

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

إذا كنت قد ثبّتَّ عمدًا وحدة نظام بدلًا من ذلك، فعدّل
`openclaw-gateway.service` عبر `sudo systemctl edit openclaw-gateway.service`.

كيف تساعد سياسات `Restart=` في الاسترداد التلقائي:
[يمكن لـ systemd أتمتة استرداد الخدمة](https://www.redhat.com/en/blog/systemd-automate-recovery).
