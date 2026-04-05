---
read_when:
    - تريد مضيف Linux رخيصًا ويعمل دائمًا للبوابة
    - تريد وصولًا عن بُعد إلى Control UI من دون تشغيل VPS خاص بك
summary: شغّل OpenClaw Gateway على exe.dev (آلة افتراضية + proxy عبر HTTPS) للوصول عن بُعد
title: exe.dev
x-i18n:
    generated_at: "2026-04-05T12:46:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: ff95b6f35b95df35c1b0cae3215647eefe88d2b7f19923868385036cc0dbdbf1
    source_path: install/exe-dev.md
    workflow: 15
---

# exe.dev

الهدف: تشغيل OpenClaw Gateway على آلة افتراضية من exe.dev، بحيث يمكن الوصول إليها من جهازك المحمول عبر: `https://<vm-name>.exe.xyz`

تفترض هذه الصفحة استخدام صورة **exeuntu** الافتراضية من exe.dev. إذا اخترت توزيعة مختلفة، فطابق الحزم وفقًا لذلك.

## المسار السريع للمبتدئين

1. [https://exe.new/openclaw](https://exe.new/openclaw)
2. املأ مفتاح/رمز المصادقة حسب الحاجة
3. انقر على "Agent" بجانب الآلة الافتراضية وانتظر حتى تنتهي Shelley من التهيئة
4. افتح `https://<vm-name>.exe.xyz/` وصادق باستخدام السر المشترك المكوَّن (يستخدم هذا الدليل مصادقة token افتراضيًا، لكن مصادقة password تعمل أيضًا إذا غيّرت `gateway.auth.mode`)
5. وافق على أي طلبات إقران أجهزة معلقة باستخدام `openclaw devices approve <requestId>`

## ما الذي تحتاجه

- حساب exe.dev
- وصول `ssh exe.dev` إلى الآلات الافتراضية في [exe.dev](https://exe.dev) (اختياري)

## التثبيت الآلي باستخدام Shelley

يمكن لـ Shelley، وهي الوكيل الخاص بـ [exe.dev](https://exe.dev)، تثبيت OpenClaw فورًا باستخدام
الـ prompt الخاص بنا. والـ prompt المستخدم هو كما يلي:

```
Set up OpenClaw (https://docs.openclaw.ai/install) on this VM. Use the non-interactive and accept-risk flags for openclaw onboarding. Add the supplied auth or token as needed. Configure nginx to forward from the default port 18789 to the root location on the default enabled site config, making sure to enable Websocket support. Pairing is done by "openclaw devices list" and "openclaw devices approve <request id>". Make sure the dashboard shows that OpenClaw's health is OK. exe.dev handles forwarding from port 8000 to port 80/443 and HTTPS for us, so the final "reachable" should be <vm-name>.exe.xyz, without port specification.
```

## التثبيت اليدوي

## 1) أنشئ الآلة الافتراضية

من جهازك:

```bash
ssh exe.dev new
```

ثم اتصل:

```bash
ssh <vm-name>.exe.xyz
```

نصيحة: أبقِ هذه الآلة الافتراضية **ذات حالة**. إذ يخزّن OpenClaw الملفات `openclaw.json`، و
`auth-profiles.json` لكل وكيل، والجلسات، وحالة القنوات/المزوّدين ضمن
`~/.openclaw/`، بالإضافة إلى مساحة العمل ضمن `~/.openclaw/workspace/`.

## 2) ثبّت المتطلبات المسبقة (على الآلة الافتراضية)

```bash
sudo apt-get update
sudo apt-get install -y git curl jq ca-certificates openssl
```

## 3) ثبّت OpenClaw

شغّل script تثبيت OpenClaw:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

## 4) اضبط nginx ليعمل كـ proxy لـ OpenClaw على المنفذ 8000

حرّر `/etc/nginx/sites-enabled/default` باستخدام

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 8000;
    listen [::]:8000;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings for long-lived connections
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

استبدل رؤوس إعادة التوجيه بدلًا من الحفاظ على السلاسل التي يزوّدها العميل.
يثق OpenClaw في بيانات IP الوصفية المعاد توجيهها فقط من proxies المكوّنة صراحةً،
وتُعامل سلاسل `X-Forwarded-For` ذات نمط الإلحاق على أنها مخاطرة تتعلق بالتقوية الأمنية.

## 5) ادخل إلى OpenClaw وامنح الصلاحيات

ادخل إلى `https://<vm-name>.exe.xyz/` (راجع مخرجات Control UI من الإعداد التفاعلي). وإذا طلب المصادقة، فألصق
السر المشترك المكوَّن من الآلة الافتراضية. يستخدم هذا الدليل مصادقة token، لذا استرجع `gateway.auth.token`
باستخدام `openclaw config get gateway.auth.token` (أو أنشئ واحدًا باستخدام `openclaw doctor --generate-gateway-token`).
إذا غيّرت البوابة إلى مصادقة password، فاستخدم `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` بدلًا من ذلك.
وافق على الأجهزة باستخدام `openclaw devices list` و`openclaw devices approve <requestId>`. وعند الشك، استخدم Shelley من متصفحك!

## الوصول عن بُعد

يتم التعامل مع الوصول عن بُعد بواسطة مصادقة [exe.dev](https://exe.dev). افتراضيًا،
تتم إعادة توجيه حركة HTTP من المنفذ 8000 إلى `https://<vm-name>.exe.xyz`
مع مصادقة عبر البريد الإلكتروني.

## التحديث

```bash
npm i -g openclaw@latest
openclaw doctor
openclaw gateway restart
openclaw health
```

الدليل: [التحديث](/install/updating)
