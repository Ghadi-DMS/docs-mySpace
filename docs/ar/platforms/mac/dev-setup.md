---
read_when:
    - إعداد بيئة التطوير الخاصة بـ macOS
summary: دليل إعداد للمطورين العاملين على تطبيق OpenClaw لـ macOS
title: إعداد تطوير macOS
x-i18n:
    generated_at: "2026-04-05T12:49:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: fd13f17391bdd87ef59e4c575e5da3312c4066de00905731263bff655a5db357
    source_path: platforms/mac/dev-setup.md
    workflow: 15
---

# إعداد المطور لـ macOS

يغطي هذا الدليل الخطوات اللازمة لبناء وتشغيل تطبيق OpenClaw لـ macOS من المصدر.

## المتطلبات الأساسية

قبل بناء التطبيق، تأكد من تثبيت ما يلي:

1. **Xcode 26.2+**: مطلوب لتطوير Swift.
2. **Node.js 24 وpnpm**: موصى بهما للبوابة وCLI ونصوص التغليف. ولا يزال Node 22 LTS، حاليًا `22.14+`، مدعومًا من أجل التوافق.

## 1. تثبيت التبعيات

ثبّت التبعيات على مستوى المشروع:

```bash
pnpm install
```

## 2. بناء التطبيق وتغليفه

لبناء تطبيق macOS وتغليفه داخل `dist/OpenClaw.app`، شغّل:

```bash
./scripts/package-mac-app.sh
```

إذا لم يكن لديك شهادة Apple Developer ID، فسيستخدم النص تلقائيًا **التوقيع المخصص** (`-`).

بالنسبة إلى أوضاع تشغيل التطوير، وأعلام التوقيع، واستكشاف أخطاء Team ID وإصلاحها، راجع README الخاصة بتطبيق macOS:
[https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md](https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md)

> **ملاحظة**: قد تؤدي التطبيقات الموقّعة بتوقيع مخصص إلى ظهور مطالبات أمان. وإذا تعطّل التطبيق فورًا مع "Abort trap 6"، فراجع قسم [استكشاف الأخطاء وإصلاحها](#troubleshooting).

## 3. تثبيت CLI

يتوقع تطبيق macOS وجود تثبيت عام لـ CLI الأمر `openclaw` لإدارة المهام الخلفية.

**لتثبيته (مستحسن):**

1. افتح تطبيق OpenClaw.
2. انتقل إلى تبويب إعدادات **General**.
3. انقر **"Install CLI"**.

بدلًا من ذلك، ثبّته يدويًا:

```bash
npm install -g openclaw@<version>
```

كما يعمل أيضًا `pnpm add -g openclaw@<version>` و`bun add -g openclaw@<version>`.
وبالنسبة إلى وقت تشغيل البوابة، يظل Node هو المسار الموصى به.

## استكشاف الأخطاء وإصلاحها

### فشل البناء: عدم تطابق toolchain أو SDK

يتوقع بناء تطبيق macOS أحدث macOS SDK وأداة Swift 6.2.

**تبعيات النظام (مطلوبة):**

- **أحدث إصدار macOS متاح في Software Update** (مطلوب بواسطة SDKs الخاصة بـ Xcode 26.2)
- **Xcode 26.2** ‏(أداة Swift 6.2)

**عمليات التحقق:**

```bash
xcodebuild -version
xcrun swift --version
```

إذا لم تتطابق الإصدارات، فحدّث macOS/Xcode وأعد تشغيل البناء.

### تعطل التطبيق عند منح الأذونات

إذا تعطل التطبيق عندما تحاول السماح بالوصول إلى **Speech Recognition** أو **Microphone**، فقد يكون السبب تلفًا في ذاكرة TCC المؤقتة أو عدم تطابق التوقيع.

**الإصلاح:**

1. أعد تعيين أذونات TCC:

   ```bash
   tccutil reset All ai.openclaw.mac.debug
   ```

2. إذا لم ينجح ذلك، فغيّر `BUNDLE_ID` مؤقتًا في [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) لفرض "بداية نظيفة" من macOS.

### البوابة تبقى في حالة "Starting..." إلى أجل غير مسمى

إذا ظلت حالة البوابة على "Starting..."، فتحقق مما إذا كانت عملية zombie تحتجز المنفذ:

```bash
openclaw gateway status
openclaw gateway stop

# إذا كنت لا تستخدم LaunchAgent (وضع التطوير / التشغيلات اليدوية)، فابحث عن المستمع:
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

إذا كان تشغيل يدوي يحتجز المنفذ، فأوقف تلك العملية (Ctrl+C). وكحل أخير، اقتل PID الذي وجدته أعلاه.
