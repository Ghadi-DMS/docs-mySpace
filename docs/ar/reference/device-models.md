---
read_when:
    - تقوم بتحديث تعيينات معرّفات طرازات الأجهزة أو ملفات NOTICE/license
    - تغيّر كيفية عرض واجهة Instances لأسماء الأجهزة
summary: كيف يضمّن OpenClaw معرّفات طرازات أجهزة Apple لتوفير أسماء ودية في تطبيق macOS.
title: قاعدة بيانات طرازات الأجهزة
x-i18n:
    generated_at: "2026-04-05T12:54:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1d99c2538a0d8fdd80fa468fa402f63479ef2522e83745a0a46527a86238aeb2
    source_path: reference/device-models.md
    workflow: 15
---

# قاعدة بيانات طرازات الأجهزة (الأسماء الودية)

يعرض التطبيق المرافق على macOS أسماء ودية لطرازات أجهزة Apple في واجهة **Instances** عبر تعيين معرّفات طرازات Apple (مثل `iPad16,6` و`Mac16,6`) إلى أسماء قابلة للقراءة البشرية.

يتم تضمين التعيين كملفات JSON ضمن:

- `apps/macos/Sources/OpenClaw/Resources/DeviceModels/`

## مصدر البيانات

نقوم حاليًا بتضمين التعيين من المستودع المرخّص بترخيص MIT:

- `kyle-seongwoo-jun/apple-device-identifiers`

وللحفاظ على حتمية البناء، يتم تثبيت ملفات JSON عند commits محددة من المصدر (ومسجلة في `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md`).

## تحديث قاعدة البيانات

1. اختر commits من المصدر التي تريد التثبيت عليها (واحدة لـ iOS وواحدة لـ macOS).
2. حدّث تجزئات commit في `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md`.
3. أعد تنزيل ملفات JSON بعد تثبيتها على تلك commits:

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. تأكد من أن `apps/macos/Sources/OpenClaw/Resources/DeviceModels/LICENSE.apple-device-identifiers.txt` ما زال يطابق المصدر (واستبدله إذا تغيّر الترخيص في المصدر).
5. تحقق من أن تطبيق macOS يُبنى بشكل نظيف (من دون تحذيرات):

```bash
swift build --package-path apps/macos
```
