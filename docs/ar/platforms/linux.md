---
read_when:
    - تبحث عن حالة التطبيق المرافق لـ Linux
    - تخطط لتغطية المنصات أو للمساهمات
summary: دعم Linux + حالة التطبيق المرافق
title: تطبيق Linux
x-i18n:
    generated_at: "2026-04-05T12:49:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5dbfc89eb65e04347479fc6c9a025edec902fb0c544fb8d5bd09c24558ea03b1
    source_path: platforms/linux.md
    workflow: 15
---

# تطبيق Linux

تُدعَم Gateway بالكامل على Linux. **Node هو وقت التشغيل الموصى به**.
ولا يُنصح باستخدام Bun مع Gateway (بسبب أخطاء WhatsApp/Telegram).

التطبيقات المرافقة الأصلية لـ Linux مخطط لها. والمساهمات مرحب بها إذا كنت تريد المساعدة في بناء واحد منها.

## المسار السريع للمبتدئين (VPS)

1. ثبّت Node 24 (موصى به؛ وما زال Node 22 LTS، حاليًا `22.14+`، يعمل من أجل التوافق)
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. من حاسوبك المحمول: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. افتح `http://127.0.0.1:18789/` وصادق باستخدام السر المشترك المكوَّن (token افتراضيًا؛ أو password إذا ضبطت `gateway.auth.mode: "password"`)

الدليل الكامل لخادم Linux: [خادم Linux](/vps). ومثال VPS خطوة بخطوة: [exe.dev](/install/exe-dev)

## التثبيت

- [البدء](/ar/start/getting-started)
- [التثبيت والتحديثات](/install/updating)
- مسارات اختيارية: [Bun (تجريبي)](/install/bun)، و[Nix](/install/nix)، و[Docker](/install/docker)

## Gateway

- [دليل تشغيل Gateway](/gateway)
- [التكوين](/gateway/configuration)

## تثبيت خدمة Gateway (CLI)

استخدم إحدى الطرق التالية:

```
openclaw onboard --install-daemon
```

أو:

```
openclaw gateway install
```

أو:

```
openclaw configure
```

اختر **Gateway service** عندما يُطلب منك ذلك.

الإصلاح/الترحيل:

```
openclaw doctor
```

## التحكم في النظام (وحدة systemd للمستخدم)

يثبّت OpenClaw خدمة systemd **للمستخدم** افتراضيًا. استخدم خدمة **نظام**
للخوادم المشتركة أو التي تعمل دائمًا. يقوم كل من `openclaw gateway install` و
`openclaw onboard --install-daemon` بالفعل بتوليد الوحدة القياسية الحالية
لك؛ ولا تكتب واحدة يدويًا إلا عندما تحتاج إلى إعداد مخصص للنظام/مدير الخدمة.
وتوجد إرشادات الخدمة الكاملة في [دليل تشغيل Gateway](/gateway).

إعداد أدنى:

أنشئ `~/.config/systemd/user/openclaw-gateway[-<profile>].service`:

```
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

فعّلها:

```
systemctl --user enable --now openclaw-gateway[-<profile>].service
```
