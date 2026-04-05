---
read_when:
    - تبحث عن دعم أنظمة التشغيل أو مسارات التثبيت
    - تقرر أين تشغّل Gateway
summary: نظرة عامة على دعم المنصات (Gateway + التطبيقات المرافقة)
title: المنصات
x-i18n:
    generated_at: "2026-04-05T12:49:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: d5be4743fd39eca426d65db940f04f3a8fc3ff2c5e10b0e82bc55fc35a7d1399
    source_path: platforms/index.md
    workflow: 15
---

# المنصات

تمت كتابة نواة OpenClaw بلغة TypeScript. ويُعد **Node هو وقت التشغيل الموصى به**.
ولا يُنصح باستخدام Bun مع Gateway (بسبب أخطاء WhatsApp/Telegram).

توجد تطبيقات مرافقة لكل من macOS (تطبيق شريط القوائم) والعُقد المحمولة (iOS/Android). أما تطبيقات Windows و
Linux المرافقة فهي مخطط لها، لكن Gateway مدعومة بالكامل اليوم.
كما أن التطبيقات المرافقة الأصلية لـ Windows مخطط لها أيضًا؛ ويوصى بتشغيل Gateway عبر WSL2.

## اختر نظام التشغيل لديك

- macOS: [macOS](/platforms/macos)
- iOS: [iOS](/platforms/ios)
- Android: [Android](/platforms/android)
- Windows: [Windows](/platforms/windows)
- Linux: [Linux](/platforms/linux)

## VPS والاستضافة

- مركز VPS: [استضافة VPS](/vps)
- Fly.io: [Fly.io](/install/fly)
- Hetzner (Docker): [Hetzner](/install/hetzner)
- GCP (Compute Engine): [GCP](/install/gcp)
- Azure (Linux VM): [Azure](/install/azure)
- exe.dev (VM + HTTPS proxy): [exe.dev](/install/exe-dev)

## روابط شائعة

- دليل التثبيت: [البدء](/ar/start/getting-started)
- دليل تشغيل Gateway: [Gateway](/gateway)
- تكوين Gateway: [التكوين](/gateway/configuration)
- حالة الخدمة: `openclaw gateway status`

## تثبيت خدمة Gateway (CLI)

استخدم إحدى الطرق التالية (كلها مدعومة):

- المعالج (موصى به): `openclaw onboard --install-daemon`
- مباشر: `openclaw gateway install`
- مسار configure: `openclaw configure` → اختر **Gateway service**
- الإصلاح/الترحيل: `openclaw doctor` (يعرض خيار تثبيت الخدمة أو إصلاحها)

يعتمد هدف الخدمة على نظام التشغيل:

- macOS: ‏LaunchAgent (`ai.openclaw.gateway` أو `ai.openclaw.<profile>`؛ والأسماء القديمة `com.openclaw.*`)
- Linux/WSL2: خدمة systemd للمستخدم (`openclaw-gateway[-<profile>].service`)
- Windows الأصلي: Scheduled Task (`OpenClaw Gateway` أو `OpenClaw Gateway (<profile>)`)، مع رجوع إلى عنصر تسجيل دخول داخل مجلد Startup لكل مستخدم إذا تم رفض إنشاء المهمة
