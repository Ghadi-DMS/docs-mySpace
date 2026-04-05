---
read_when:
    - تصحيح مشكلات اكتشاف Bonjour على macOS/iOS
    - تغيير أنواع خدمات mDNS أو سجلات TXT أو تجربة استخدام الاكتشاف
summary: اكتشاف Bonjour/mDNS + تصحيح الأخطاء (إشارات Gateway والعملاء وأوضاع الفشل الشائعة)
title: اكتشاف Bonjour
x-i18n:
    generated_at: "2026-04-05T12:42:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7f5a7f3211c74d4d10fdc570fc102b3c949c0ded9409c54995ab8820e5787f02
    source_path: gateway/bonjour.md
    workflow: 15
---

# اكتشاف Bonjour / mDNS

يستخدم OpenClaw بروتوكول Bonjour ‏(mDNS / DNS‑SD) لاكتشاف Gateway نشط (نقطة نهاية WebSocket).
يُعد تصفح `local.` عبر البث المتعدد **وسيلة مريحة داخل LAN فقط**. أما للاكتشاف عبر الشبكات، فيمكن
أيضًا نشر الإشارة نفسها من خلال نطاق DNS-SD واسع النطاق مكوَّن. ويظل الاكتشاف
أفضل جهد ولا **يستبدل** الاتصال عبر SSH أو Tailnet.

## Bonjour واسع النطاق (Unicast DNS-SD) عبر Tailscale

إذا كانت العقدة وgateway على شبكات مختلفة، فلن يعبر mDNS متعدد الإرسال
هذا الحد. ويمكنك الحفاظ على تجربة الاكتشاف نفسها بالانتقال إلى **unicast DNS‑SD**
‏("Wide‑Area Bonjour") عبر Tailscale.

الخطوات عالية المستوى:

1. شغّل خادم DNS على مضيف gateway ‏(بحيث يمكن الوصول إليه عبر Tailnet).
2. انشر سجلات DNS‑SD للخدمة `_openclaw-gw._tcp` ضمن منطقة مخصصة
   (مثال: `openclaw.internal.`).
3. قم بتكوين **split DNS** في Tailscale بحيث يُحل نطاقك المختار عبر
   خادم DNS هذا للعملاء (بما في ذلك iOS).

يدعم OpenClaw أي نطاق اكتشاف؛ و`openclaw.internal.` مجرد مثال.
تتصفح عقد iOS/Android كلًا من `local.` ونطاقك واسع النطاق المكوَّن.

### تكوين Gateway (موصى به)

```json5
{
  gateway: { bind: "tailnet" }, // tailnet-only (موصى به)
  discovery: { wideArea: { enabled: true } }, // يفعّل نشر wide-area DNS-SD
}
```

### إعداد خادم DNS لمرة واحدة (مضيف gateway)

```bash
openclaw dns setup --apply
```

يعمل هذا على تثبيت CoreDNS وتكوينه بحيث:

- يستمع على المنفذ 53 فقط على واجهات Tailscale الخاصة بـ gateway
- يقدّم النطاق المختار (مثال: `openclaw.internal.`) من `~/.openclaw/dns/<domain>.db`

تحقق من جهاز متصل بـ tailnet:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### إعدادات DNS في Tailscale

في وحدة إدارة Tailscale:

- أضف nameserver يشير إلى عنوان tailnet IP الخاص بـ gateway ‏(UDP/TCP 53).
- أضف split DNS بحيث يستخدم نطاق الاكتشاف لديك ذلك الـ nameserver.

بمجرد قبول العملاء لـ tailnet DNS، يمكن لعقد iOS وCLI الخاصة بالاكتشاف
تصفح `_openclaw-gw._tcp` في نطاق الاكتشاف لديك من دون بث متعدد.

### أمان مستمع Gateway ‏(موصى به)

يرتبط منفذ Gateway WS ‏(الافتراضي `18789`) بـ loopback افتراضيًا. وللوصول عبر LAN/tailnet،
قم بالربط صراحةً وأبقِ المصادقة مفعلة.

بالنسبة إلى إعدادات tailnet-only:

- اضبط `gateway.bind: "tailnet"` في `~/.openclaw/openclaw.json`.
- أعد تشغيل Gateway ‏(أو أعد تشغيل تطبيق شريط القوائم على macOS).

## ما الذي يعلن

فقط Gateway هو الذي يعلن عن `_openclaw-gw._tcp`.

## أنواع الخدمات

- `_openclaw-gw._tcp` — إشارة نقل gateway ‏(تستخدمها عقد macOS/iOS/Android).

## مفاتيح TXT ‏(تلميحات غير سرية)

يعلن Gateway عن تلميحات صغيرة غير سرية لتسهيل تدفقات واجهة المستخدم:

- `role=gateway`
- `displayName=<friendly name>`
- `lanHost=<hostname>.local`
- `gatewayPort=<port>` ‏(Gateway WS + HTTP)
- `gatewayTls=1` ‏(فقط عند تمكين TLS)
- `gatewayTlsSha256=<sha256>` ‏(فقط عند تمكين TLS وتوفر البصمة)
- `canvasPort=<port>` ‏(فقط عند تمكين مضيف canvas؛ وهو حاليًا نفس `gatewayPort`)
- `transport=gateway`
- `tailnetDns=<magicdns>` ‏(تلميح اختياري عند توفر Tailnet)
- `sshPort=<port>` ‏(في وضع mDNS الكامل فقط؛ قد يحذف wide-area DNS-SD هذه القيمة)
- `cliPath=<path>` ‏(في وضع mDNS الكامل فقط؛ يظل wide-area DNS-SD يكتبه كتلميح تثبيت عن بُعد)

ملاحظات أمنية:

- سجلات TXT في Bonjour/mDNS **غير موثقة**. يجب ألا يعتبر العملاء TXT توجيهًا موثوقًا.
- يجب على العملاء التوجيه باستخدام نقطة نهاية الخدمة المحلولة (SRV + A/AAAA). تعامل مع `lanHost` و`tailnetDns` و`gatewayPort` و`gatewayTlsSha256` على أنها تلميحات فقط.
- يجب أن يستخدم الاستهداف التلقائي لـ SSH مضيف الخدمة المحلول، وليس تلميحات TXT فقط.
- يجب ألا يسمح تثبيت TLS مطلقًا لقيمة `gatewayTlsSha256` المُعلنة بأن تتجاوز بصمة مخزنة مسبقًا.
- يجب على عقد iOS/Android التعامل مع الاتصالات المباشرة المعتمدة على الاكتشاف على أنها **TLS-only** وتتطلب تأكيدًا صريحًا من المستخدم قبل الوثوق ببصمة تُرى لأول مرة.

## تصحيح الأخطاء على macOS

أدوات مدمجة مفيدة:

- تصفح المثيلات:

  ```bash
  dns-sd -B _openclaw-gw._tcp local.
  ```

- حل مثيل واحد (استبدل `<instance>`):

  ```bash
  dns-sd -L "<instance>" _openclaw-gw._tcp local.
  ```

إذا كان التصفح يعمل لكن الحل يفشل، فأنت عادةً تصطدم بسياسة LAN أو
مشكلة في محلل mDNS.

## تصحيح الأخطاء في سجلات Gateway

يكتب Gateway ملف سجل متجددًا (يُطبع عند بدء التشغيل على شكل
`gateway log file: ...`). ابحث عن أسطر `bonjour:`، وخصوصًا:

- `bonjour: advertise failed ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`
- `bonjour: watchdog detected non-announced service ...`

## تصحيح الأخطاء على عقدة iOS

تستخدم عقدة iOS المكوّن `NWBrowser` لاكتشاف `_openclaw-gw._tcp`.

لالتقاط السجلات:

- الإعدادات → Gateway → متقدم → **Discovery Debug Logs**
- الإعدادات → Gateway → متقدم → **Discovery Logs** → أعد إنتاج المشكلة → **Copy**

يتضمن السجل انتقالات حالة المتصفح وتغييرات مجموعة النتائج.

## أوضاع الفشل الشائعة

- **Bonjour لا يعبر بين الشبكات**: استخدم Tailnet أو SSH.
- **البث المتعدد محظور**: بعض شبكات Wi‑Fi تعطل mDNS.
- **السكون / تبدل الواجهات**: قد يُسقط macOS نتائج mDNS مؤقتًا؛ أعد المحاولة.
- **التصفح يعمل لكن الحل يفشل**: أبقِ أسماء الأجهزة بسيطة (تجنب الرموز التعبيرية أو
  علامات الترقيم)، ثم أعد تشغيل Gateway. اسم مثيل الخدمة مشتق من
  اسم المضيف، لذا قد تربك الأسماء المعقدة جدًا بعض المحللات.

## أسماء المثيلات المُهربة (`\032`)

غالبًا ما يقوم Bonjour/DNS‑SD بتهريب البايتات في أسماء مثيلات الخدمة كسلاسل `\DDD`
عشرية (مثلًا تتحول المسافات إلى `\032`).

- هذا طبيعي على مستوى البروتوكول.
- يجب على واجهات المستخدم فك هذا الترميز للعرض (يستخدم iOS الدالة `BonjourEscapes.decode`).

## التعطيل / التكوين

- يقوم `OPENCLAW_DISABLE_BONJOUR=1` بتعطيل الإعلان (قديم: `OPENCLAW_DISABLE_BONJOUR`).
- يتحكم `gateway.bind` في `~/.openclaw/openclaw.json` في وضع ربط Gateway.
- يتجاوز `OPENCLAW_SSH_PORT` منفذ SSH عند الإعلان عن `sshPort` (قديم: `OPENCLAW_SSH_PORT`).
- ينشر `OPENCLAW_TAILNET_DNS` تلميح MagicDNS في TXT ‏(قديم: `OPENCLAW_TAILNET_DNS`).
- يتجاوز `OPENCLAW_CLI_PATH` مسار CLI المُعلن (قديم: `OPENCLAW_CLI_PATH`).

## وثائق ذات صلة

- سياسة الاكتشاف واختيار النقل: [الاكتشاف](/gateway/discovery)
- اقتران العقد + الموافقات: [اقتران Gateway](/gateway/pairing)
