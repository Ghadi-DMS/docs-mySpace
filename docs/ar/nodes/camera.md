---
read_when:
    - إضافة أو تعديل التقاط الكاميرا على عُقد iOS/Android أو macOS
    - توسيع مسارات عمل الملفات المؤقتة MEDIA التي يمكن للوكلاء الوصول إليها
summary: 'التقاط الكاميرا (عُقد iOS/Android + تطبيق macOS) لاستخدام الوكيل: صور (`jpg`) ومقاطع فيديو قصيرة (`mp4`)'
title: التقاط الكاميرا
x-i18n:
    generated_at: "2026-04-05T12:48:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 30b1beaac9602ff29733f72b953065f271928743c8fff03191a007e8b965c88d
    source_path: nodes/camera.md
    workflow: 15
---

# التقاط الكاميرا (للوكيل)

يدعم OpenClaw **التقاط الكاميرا** في مسارات عمل الوكيل:

- **عقدة iOS** (مقترنة عبر Gateway): التقاط **صورة** (`jpg`) أو **مقطع فيديو قصير** (`mp4`، مع صوت اختياري) عبر `node.invoke`.
- **عقدة Android** (مقترنة عبر Gateway): التقاط **صورة** (`jpg`) أو **مقطع فيديو قصير** (`mp4`، مع صوت اختياري) عبر `node.invoke`.
- **تطبيق macOS** (عقدة عبر Gateway): التقاط **صورة** (`jpg`) أو **مقطع فيديو قصير** (`mp4`، مع صوت اختياري) عبر `node.invoke`.

تخضع كل عمليات الوصول إلى الكاميرا إلى **إعدادات يتحكم بها المستخدم**.

## عقدة iOS

### إعداد المستخدم (مفعّل افتراضيًا)

- علامة تبويب الإعدادات في iOS → **Camera** → **Allow Camera** (`camera.enabled`)
  - الافتراضي: **مفعّل** (تُعامل القيمة المفقودة على أنها مفعّلة).
  - عند التعطيل: تعيد أوامر `camera.*` القيمة `CAMERA_DISABLED`.

### الأوامر (عبر Gateway `node.invoke`)

- `camera.list`
  - حمولة الاستجابة:
    - `devices`: مصفوفة من `{ id, name, position, deviceType }`

- `camera.snap`
  - المعلمات:
    - `facing`: ‏`front|back` (الافتراضي: `front`)
    - `maxWidth`: رقم (اختياري؛ الافتراضي `1600` على عقدة iOS)
    - `quality`: ‏`0..1` (اختياري؛ الافتراضي `0.9`)
    - `format`: حاليًا `jpg`
    - `delayMs`: رقم (اختياري؛ الافتراضي `0`)
    - `deviceId`: سلسلة نصية (اختياري؛ من `camera.list`)
  - حمولة الاستجابة:
    - `format: "jpg"`
    - `base64: "<...>"`
    - `width`, `height`
  - حماية الحمولة: يُعاد ضغط الصور للحفاظ على حمولة `base64` تحت 5 MB.

- `camera.clip`
  - المعلمات:
    - `facing`: ‏`front|back` (الافتراضي: `front`)
    - `durationMs`: رقم (الافتراضي `3000`، ومقيّد بحد أقصى `60000`)
    - `includeAudio`: قيمة منطقية (الافتراضي `true`)
    - `format`: حاليًا `mp4`
    - `deviceId`: سلسلة نصية (اختياري؛ من `camera.list`)
  - حمولة الاستجابة:
    - `format: "mp4"`
    - `base64: "<...>"`
    - `durationMs`
    - `hasAudio`

### متطلب المقدمة

مثل `canvas.*`، تسمح عقدة iOS بأوامر `camera.*` فقط في **المقدمة**. وتعطي الاستدعاءات في الخلفية `NODE_BACKGROUND_UNAVAILABLE`.

### مساعد CLI (ملفات مؤقتة + MEDIA)

أسهل طريقة للحصول على المرفقات هي عبر مساعد CLI، الذي يكتب الوسائط المفكوكة إلى ملف مؤقت ويطبع `MEDIA:<path>`.

أمثلة:

```bash
openclaw nodes camera snap --node <id>               # default: both front + back (2 MEDIA lines)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

ملاحظات:

- يستخدم `nodes camera snap` الوضع الافتراضي **لكلا** الوجهتين لمنح الوكيل كلا المنظورين.
- تكون ملفات الإخراج مؤقتة (في دليل temp الخاص بنظام التشغيل) ما لم تنشئ wrapper خاصًا بك.

## عقدة Android

### إعداد المستخدم في Android (مفعّل افتراضيًا)

- ورقة إعدادات Android → **Camera** → **Allow Camera** (`camera.enabled`)
  - الافتراضي: **مفعّل** (تُعامل القيمة المفقودة على أنها مفعّلة).
  - عند التعطيل: تعيد أوامر `camera.*` القيمة `CAMERA_DISABLED`.

### الأذونات

- يتطلب Android أذونات وقت التشغيل:
  - `CAMERA` لكل من `camera.snap` و`camera.clip`.
  - `RECORD_AUDIO` للأمر `camera.clip` عندما تكون `includeAudio=true`.

إذا كانت الأذونات مفقودة، فسيعرض التطبيق مطالبة عندما يكون ذلك ممكنًا؛ وإذا تم رفضها، تفشل طلبات `camera.*` بخطأ
`*_PERMISSION_REQUIRED`.

### متطلب المقدمة في Android

مثل `canvas.*`، تسمح عقدة Android بأوامر `camera.*` فقط في **المقدمة**. وتعطي الاستدعاءات في الخلفية `NODE_BACKGROUND_UNAVAILABLE`.

### أوامر Android (عبر Gateway `node.invoke`)

- `camera.list`
  - حمولة الاستجابة:
    - `devices`: مصفوفة من `{ id, name, position, deviceType }`

### حماية الحمولة

يُعاد ضغط الصور للحفاظ على حمولة `base64` تحت 5 MB.

## تطبيق macOS

### إعداد المستخدم (معطّل افتراضيًا)

يعرض التطبيق المرافق على macOS مربع اختيار:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`)
  - الافتراضي: **معطّل**
  - عند التعطيل: تعيد طلبات الكاميرا “Camera disabled by user”.

### مساعد CLI (node invoke)

استخدم CLI الرئيسي لـ `openclaw` لاستدعاء أوامر الكاميرا على عقدة macOS.

أمثلة:

```bash
openclaw nodes camera list --node <id>            # list camera ids
openclaw nodes camera snap --node <id>            # prints MEDIA:<path>
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s          # prints MEDIA:<path>
openclaw nodes camera clip --node <id> --duration-ms 3000      # prints MEDIA:<path> (legacy flag)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

ملاحظات:

- يستخدم `openclaw nodes camera snap` القيمة الافتراضية `maxWidth=1600` ما لم يتم تجاوزها.
- على macOS، ينتظر `camera.snap` مدة `delayMs` (الافتراضي 2000ms) بعد الاستعداد/استقرار التعريض قبل الالتقاط.
- يُعاد ضغط حمولات الصور للحفاظ على `base64` تحت 5 MB.

## السلامة والحدود العملية

- يؤدي الوصول إلى الكاميرا والميكروفون إلى إظهار مطالبات أذونات نظام التشغيل المعتادة (ويتطلب سلاسل الاستخدام في `Info.plist`).
- تكون مقاطع الفيديو محدودة بحد أقصى (حاليًا `<= 60s`) لتجنب حمولات العقدة الكبيرة جدًا (حمل `base64` الزائد + حدود الرسائل).

## فيديو شاشة macOS (على مستوى نظام التشغيل)

بالنسبة إلى فيديو **الشاشة** (وليس الكاميرا)، استخدم التطبيق المرافق على macOS:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # prints MEDIA:<path>
```

ملاحظات:

- يتطلب إذن macOS **Screen Recording** ‏(TCC).
