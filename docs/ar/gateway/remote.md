---
read_when:
    - تشغيل إعدادات gateway البعيدة أو استكشاف أخطائها وإصلاحها
summary: الوصول البعيد باستخدام أنفاق SSH ‏(Gateway WS) وtailnets
title: الوصول البعيد
x-i18n:
    generated_at: "2026-04-05T12:44:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8596fa2a7fd44117dfe92b70c9d8f28c0e16d7987adf0d0769a9eff71d5bc081
    source_path: gateway/remote.md
    workflow: 15
---

# الوصول البعيد (SSH والأنفاق وtailnets)

يدعم هذا المستودع نمط “البعيد عبر SSH” من خلال إبقاء Gateway واحدة (الرئيسية) تعمل على مضيف مخصص (سطح مكتب/خادم) وتوصيل العملاء بها.

- بالنسبة إلى **المشغلين (أنت / تطبيق macOS)**: يكون نفق SSH هو الرجوع الاحتياطي العام.
- بالنسبة إلى **العقد (iOS/Android والأجهزة المستقبلية)**: اتصل بـ **WebSocket** الخاصة بـ Gateway ‏(LAN/tailnet أو نفق SSH حسب الحاجة).

## الفكرة الأساسية

- ترتبط Gateway WebSocket بعنوان **loopback** على المنفذ المكوّن لديك (الافتراضي 18789).
- للاستخدام البعيد، تقوم بتمرير منفذ loopback هذا عبر SSH (أو تستخدم tailnet/VPN وتقلل الحاجة إلى الأنفاق).

## إعدادات VPN/tailnet الشائعة (حيث يعيش الوكيل)

اعتبر **مضيف Gateway** هو “المكان الذي يعيش فيه الوكيل”. فهو يملك الجلسات وملفات تعريف المصادقة والقنوات والحالة.
ويتصل به الحاسوب المحمول/سطح المكتب لديك (والعقد).

### 1) Gateway دائمة التشغيل في tailnet الخاصة بك (VPS أو خادم منزلي)

شغّل Gateway على مضيف دائم الوصول والوصول إليها عبر **Tailscale** أو SSH.

- **أفضل تجربة استخدام:** أبقِ `gateway.bind: "loopback"` واستخدم **Tailscale Serve** لـ Control UI.
- **الرجوع الاحتياطي:** أبقِ loopback + نفق SSH من أي جهاز يحتاج إلى الوصول.
- **أمثلة:** [exe.dev](/install/exe-dev) (VM سهلة) أو [Hetzner](/install/hetzner) (VPS للإنتاج).

هذا مثالي عندما ينام حاسوبك المحمول كثيرًا ولكنك تريد أن يبقى الوكيل دائم التشغيل.

### 2) سطح المكتب المنزلي يشغّل Gateway، والحاسوب المحمول هو جهاز التحكم البعيد

لا يشغّل الحاسوب المحمول الوكيل. بل يتصل عن بُعد:

- استخدم وضع **Remote over SSH** في تطبيق macOS ‏(Settings → General → “OpenClaw runs”).
- يفتح التطبيق النفق ويديره، لذلك يعمل WebChat + فحوصات الصحة “فقط”.

دليل التشغيل: [الوصول البعيد على macOS](/platforms/mac/remote).

### 3) الحاسوب المحمول يشغّل Gateway، مع وصول بعيد من أجهزة أخرى

أبقِ Gateway محلية ولكن اكشفها بأمان:

- نفق SSH إلى الحاسوب المحمول من أجهزة أخرى، أو
- Tailscale Serve لـ Control UI مع إبقاء Gateway مقتصرة على loopback.

الدليل: [Tailscale](/gateway/tailscale) و[نظرة عامة على الويب](/web).

## تدفق الأوامر (ما الذي يعمل وأين)

خدمة gateway واحدة تملك الحالة + القنوات. أما العقد فهي أطراف ملحقة.

مثال على التدفق (Telegram → عقدة):

- تصل رسالة Telegram إلى **Gateway**.
- تشغّل Gateway **الوكيل** وتقرر ما إذا كانت ستستدعي أداة عقدة.
- تستدعي Gateway **العقدة** عبر Gateway WebSocket ‏(`node.*` RPC).
- تعيد العقدة النتيجة؛ ثم ترد Gateway مرة أخرى إلى Telegram.

ملاحظات:

- **العقد لا تشغّل خدمة gateway.** يجب أن تعمل gateway واحدة فقط لكل مضيف ما لم تكن تشغّل ملفات تعريف معزولة عمدًا (راجع [Gateways متعددة](/gateway/multiple-gateways)).
- “وضع العقدة” في تطبيق macOS ليس إلا عميل عقدة عبر Gateway WebSocket.

## نفق SSH ‏(CLI + الأدوات)

أنشئ نفقًا محليًا إلى Gateway WS البعيدة:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

مع تشغيل النفق:

- سيصل `openclaw health` و`openclaw status --deep` الآن إلى gateway البعيدة عبر `ws://127.0.0.1:18789`.
- يمكن أيضًا لـ `openclaw gateway status` و`openclaw gateway health` و`openclaw gateway probe` و`openclaw gateway call` استهداف عنوان URL المُمرَّر عبر `--url` عند الحاجة.

ملاحظة: استبدل `18789` بالقيمة المكوّنة لديك في `gateway.port` (أو `--port`/`OPENCLAW_GATEWAY_PORT`).
ملاحظة: عند تمرير `--url`، لا يعود CLI إلى بيانات اعتماد التكوين أو البيئة.
ضمّن `--token` أو `--password` صراحةً. ويُعد غياب بيانات الاعتماد الصريحة خطأ.

## الإعدادات الافتراضية البعيدة لـ CLI

يمكنك حفظ هدف بعيد بحيث تستخدمه أوامر CLI افتراضيًا:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

عندما تكون gateway مقتصرة على loopback، أبقِ عنوان URL عند `ws://127.0.0.1:18789` وافتح نفق SSH أولًا.

## أولوية بيانات الاعتماد

يتبع حل بيانات اعتماد Gateway عقدًا مشتركًا واحدًا عبر مسارات call/probe/status ومراقبة Discord exec-approval. ويستخدم node-host العقد الأساسي نفسه مع استثناء واحد في الوضع المحلي (إذ يتجاهل عمدًا `gateway.remote.*`):

- تنتصر دائمًا بيانات الاعتماد الصريحة (`--token` أو `--password` أو أداة `gatewayToken`) في مسارات الاستدعاء التي تقبل مصادقة صريحة.
- أمان تجاوز عنوان URL:
  - لا تعيد تجاوزات CLI لعنوان URL ‏(`--url`) استخدام بيانات اعتماد ضمنية من التكوين/البيئة أبدًا.
  - قد تستخدم تجاوزات عنوان URL من البيئة (`OPENCLAW_GATEWAY_URL`) بيانات اعتماد البيئة فقط (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`).
- الإعدادات الافتراضية للوضع المحلي:
  - الرمز المميز: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (لا ينطبق الرجوع إلى البعيد إلا عندما يكون إدخال رمز المصادقة المحلي غير معيّن)
  - كلمة المرور: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (لا ينطبق الرجوع إلى البعيد إلا عندما يكون إدخال كلمة مرور المصادقة المحلية غير معيّن)
- الإعدادات الافتراضية للوضع البعيد:
  - الرمز المميز: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - كلمة المرور: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- استثناء node-host في الوضع المحلي: يتم تجاهل `gateway.remote.token` / `gateway.remote.password`.
- تكون فحوصات رموز probe/status البعيدة صارمة افتراضيًا: إذ تستخدم `gateway.remote.token` فقط (من دون رجوع إلى الرمز المحلي) عند استهداف الوضع البعيد.
- تستخدم تجاوزات بيئة Gateway المتغيرات `OPENCLAW_GATEWAY_*` فقط.

## واجهة الدردشة عبر SSH

لم يعد WebChat يستخدم منفذ HTTP منفصلًا. بل تتصل واجهة SwiftUI للدردشة مباشرةً بـ Gateway WebSocket.

- مرّر المنفذ `18789` عبر SSH ‏(انظر أعلاه)، ثم صِل العملاء إلى `ws://127.0.0.1:18789`.
- على macOS، فضّل وضع “Remote over SSH” في التطبيق، لأنه يدير النفق تلقائيًا.

## وضع "Remote over SSH" في تطبيق macOS

يمكن لتطبيق شريط القوائم في macOS إدارة الإعداد نفسه بالكامل (فحوصات الحالة البعيدة، وWebChat، وتمرير Voice Wake).

دليل التشغيل: [الوصول البعيد على macOS](/platforms/mac/remote).

## قواعد الأمان (البعيد/VPN)

باختصار: **أبقِ Gateway مقتصرة على loopback** ما لم تكن متأكدًا أنك بحاجة إلى bind.

- **Loopback + SSH/Tailscale Serve** هو الإعداد الافتراضي الأكثر أمانًا (من دون كشف عام).
- تكون `ws://` النصية الصريحة مقتصرة على loopback افتراضيًا. وللشبكات الخاصة الموثوقة،
  اضبط `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` على عملية العميل كخيار كسر زجاج.
- **الربط غير الخاص بـ loopback** ‏(`lan`/`tailnet`/`custom`، أو `auto` عندما لا تكون loopback متاحة) يجب أن يستخدم مصادقة gateway: رمزًا مميزًا أو كلمة مرور أو reverse proxy واعيًا بالهوية مع `gateway.auth.mode: "trusted-proxy"`.
- إن `gateway.remote.token` / `.password` هما مصدران لبيانات اعتماد العميل. وهما **لا** يكوّنان مصادقة الخادم بحد ذاتهما.
- يمكن لمسارات الاستدعاء المحلية استخدام `gateway.remote.*` كرجوع احتياطي فقط عندما يكون `gateway.auth.*` غير معيّن.
- إذا كان `gateway.auth.token` / `gateway.auth.password` مكوّنين صراحةً عبر SecretRef وغير محلولين، فإن الحل يفشل بشكل مغلق (من دون أن يخفي الرجوع الاحتياطي البعيد ذلك).
- يقوم `gateway.remote.tlsFingerprint` بتثبيت شهادة TLS البعيدة عند استخدام `wss://`.
- يمكن لـ **Tailscale Serve** مصادقة حركة Control UI/WebSocket عبر ترويسات الهوية
  عندما تكون `gateway.auth.allowTailscale: true`؛ أما نقاط نهاية HTTP API فلا
  تستخدم مصادقة ترويسات Tailscale هذه، بل تتبع وضع مصادقة HTTP العادي
  الخاص بـ gateway. ويفترض هذا التدفق من دون رمز مميز أن مضيف gateway موثوق. اضبطها على
  `false` إذا كنت تريد مصادقة بالسر المشترك في كل مكان.
- إن مصادقة **Trusted-proxy** مخصصة فقط لإعدادات proxy غير loopback والواعية بالهوية.
  أما reverse proxies المحلية على المضيف نفسه عبر loopback فلا تفي بـ `gateway.auth.mode: "trusted-proxy"`.
- تعامل مع التحكم في المتصفح كأنه وصول مشغل: tailnet فقط + pairing مقصود للعقدة.

للتفاصيل: [Security](/gateway/security).

### macOS: نفق SSH دائم عبر LaunchAgent

بالنسبة إلى عملاء macOS الذين يتصلون بـ gateway بعيدة، فإن أسهل إعداد دائم يستخدم إدخال `LocalForward` في إعداد SSH مع LaunchAgent للإبقاء على النفق حيًا عبر إعادة التشغيل والأعطال.

#### الخطوة 1: أضف إعداد SSH

حرّر `~/.ssh/config`:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

استبدل `<REMOTE_IP>` و`<REMOTE_USER>` بالقيم الخاصة بك.

#### الخطوة 2: انسخ مفتاح SSH ‏(مرة واحدة)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### الخطوة 3: اضبط رمز gateway

خزّن الرمز في التكوين حتى يستمر عبر عمليات إعادة التشغيل:

```bash
openclaw config set gateway.remote.token "<your-token>"
```

#### الخطوة 4: أنشئ LaunchAgent

احفظ هذا الملف باسم `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### الخطوة 5: حمّل LaunchAgent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

سيبدأ النفق تلقائيًا عند تسجيل الدخول، ويُعاد تشغيله عند التعطل، ويحافظ على المنفذ المُمرَّر فعالًا.

ملاحظة: إذا كان لديك LaunchAgent متبقٍ باسم `com.openclaw.ssh-tunnel` من إعداد أقدم، فقم بإلغاء تحميله وحذفه.

#### استكشاف الأخطاء وإصلاحها

تحقق مما إذا كان النفق يعمل:

```bash
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789
```

أعد تشغيل النفق:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel
```

أوقف النفق:

```bash
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| إدخال التكوين                         | ما الذي يفعله                                               |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | يمرّر المنفذ المحلي 18789 إلى المنفذ البعيد 18789            |
| `ssh -N`                             | SSH من دون تنفيذ أوامر بعيدة (تمرير منافذ فقط)              |
| `KeepAlive`                          | يعيد تشغيل النفق تلقائيًا إذا تعطّل                         |
| `RunAtLoad`                          | يبدأ النفق عند تحميل LaunchAgent وقت تسجيل الدخول           |
