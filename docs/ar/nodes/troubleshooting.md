---
read_when:
    - العقدة متصلة لكن أدوات camera/canvas/screen/exec تفشل
    - تحتاج إلى النموذج الذهني للتمييز بين pairing الخاصة بالعقدة والموافقات
summary: استكشاف أخطاء إقران العقدة، ومتطلبات المقدمة، والأذونات، وإخفاقات الأدوات وإصلاحها
title: استكشاف أخطاء العقدة وإصلاحها
x-i18n:
    generated_at: "2026-04-05T12:49:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: c2e431e6a35c482a655e01460bef9fab5d5a5ae7dc46f8f992ee51100f5c937e
    source_path: nodes/troubleshooting.md
    workflow: 15
---

# استكشاف أخطاء العقدة وإصلاحها

استخدم هذه الصفحة عندما تكون العقدة مرئية في الحالة لكن أدوات العقدة تفشل.

## تسلسل الأوامر

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

ثم شغّل الفحوصات الخاصة بالعقدة:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

الإشارات السليمة:

- العقدة متصلة ومقترنة بالدور `node`.
- يتضمن `nodes describe` الإمكانية التي تستدعيها.
- تُظهر موافقات exec الوضع/قائمة السماح المتوقعة.

## متطلبات المقدمة

تكون `canvas.*` و`camera.*` و`screen.*` خاصة بالمقدمة فقط على عُقد iOS/Android.

فحص سريع وإصلاح:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

إذا رأيت `NODE_BACKGROUND_UNAVAILABLE`، فأعد تطبيق العقدة إلى المقدمة ثم أعد المحاولة.

## مصفوفة الأذونات

| الإمكانية                   | iOS                                     | Android                                      | تطبيق عقدة macOS                | رمز الفشل المعتاد           |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | ----------------------------- | ------------------------------ |
| `camera.snap`, `camera.clip` | Camera (+ mic لصوت clip)           | Camera (+ mic لصوت clip)                | Camera (+ mic لصوت clip) | `*_PERMISSION_REQUIRED`        |
| `screen.record`              | Screen Recording (+ mic اختياري)       | مطالبة التقاط الشاشة (+ mic اختياري)       | Screen Recording              | `*_PERMISSION_REQUIRED`        |
| `location.get`               | While Using أو Always (بحسب الوضع) | موقع في المقدمة/الخلفية بحسب الوضع | إذن الموقع           | `LOCATION_PERMISSION_REQUIRED` |
| `system.run`                 | غير منطبق (مسار مضيف العقدة)                    | غير منطبق (مسار مضيف العقدة)                         | موافقات Exec مطلوبة       | `SYSTEM_RUN_DENIED`            |

## pairing مقابل الموافقات

هاتان بوابتان مختلفتان:

1. **إقران الجهاز**: هل يمكن لهذه العقدة الاتصال بالبوابة؟
2. **سياسة أوامر عقدة البوابة**: هل يُسمح بمعرّف أمر RPC عبر `gateway.nodes.allowCommands` / `denyCommands` والإعدادات الافتراضية الخاصة بالمنصة؟
3. **موافقات exec**: هل يمكن لهذه العقدة تشغيل أمر shell محدد محليًا؟

فحوصات سريعة:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

إذا كان pairing مفقودًا، فوافق أولًا على جهاز العقدة.
إذا كان `nodes describe` يفتقد أمرًا ما، فتحقق من سياسة أوامر عقدة البوابة وما إذا كانت العقدة قد أعلنت ذلك الأمر فعليًا عند الاتصال.
إذا كان pairing جيدًا لكن `system.run` يفشل، فأصلح موافقات exec/قائمة السماح على تلك العقدة.

يُعد pairing الخاص بالعقدة بوابة هوية/ثقة، وليس سطح موافقة لكل أمر. أما بالنسبة إلى `system.run`، فتوجد السياسة الخاصة بكل عقدة في ملف موافقات exec الخاص بتلك العقدة (`openclaw approvals get --node ...`)، وليس في سجل pairing الخاص بالبوابة.

بالنسبة إلى عمليات `host=node` المعتمدة على الموافقات، تربط البوابة التنفيذ أيضًا بـ
`systemRunPlan` القياسي المُحضّر. فإذا قام مستدعٍ لاحق بتعديل
command/cwd أو بيانات الجلسة الوصفية قبل تمرير التشغيل الموافق عليه، فسترفض البوابة
التشغيل باعتباره عدم تطابق في الموافقة بدلًا من الثقة في الحمولة المعدّلة.

## رموز أخطاء العقدة الشائعة

- `NODE_BACKGROUND_UNAVAILABLE` → التطبيق في الخلفية؛ انقله إلى المقدمة.
- `CAMERA_DISABLED` → مفتاح الكاميرا معطل في إعدادات العقدة.
- `*_PERMISSION_REQUIRED` → إذن نظام التشغيل مفقود/مرفوض.
- `LOCATION_DISABLED` → وضع الموقع معطل.
- `LOCATION_PERMISSION_REQUIRED` → وضع الموقع المطلوب غير ممنوح.
- `LOCATION_BACKGROUND_UNAVAILABLE` → التطبيق في الخلفية لكن إذن While Using فقط موجود.
- `SYSTEM_RUN_DENIED: approval required` → يتطلب طلب exec موافقة صريحة.
- `SYSTEM_RUN_DENIED: allowlist miss` → الأمر محظور بواسطة وضع allowlist.
  على مضيفات عقد Windows، تُعامل صيغ shell-wrapper مثل `cmd.exe /c ...` على أنها حالات allowlist miss في
  وضع allowlist ما لم تتم الموافقة عليها عبر تدفق ask.

## حلقة الاستعادة السريعة

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

إذا بقيت المشكلة:

- أعد الموافقة على pairing الخاصة بالجهاز.
- أعد فتح تطبيق العقدة (في المقدمة).
- امنح أذونات نظام التشغيل مرة أخرى.
- أعد إنشاء/تعديل سياسة موافقة exec.

ذو صلة:

- [/nodes/index](/nodes/index)
- [/nodes/camera](/nodes/camera)
- [/nodes/location-command](/nodes/location-command)
- [/tools/exec-approvals](/tools/exec-approvals)
- [/gateway/pairing](/gateway/pairing)
