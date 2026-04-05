---
read_when:
    - أنت تستخدم `openclaw browser` وتريد أمثلة على المهام الشائعة
    - تريد التحكم في متصفح يعمل على جهاز آخر عبر مضيف node
    - تريد الاتصال بـ Chrome المحلي المسجل الدخول إليه عبر Chrome MCP
summary: مرجع CLI للأمر `openclaw browser` ‏(دورة الحياة، والملفات الشخصية، وعلامات التبويب، والإجراءات، والحالة، وتصحيح الأخطاء)
title: browser
x-i18n:
    generated_at: "2026-04-05T12:37:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: c89a7483dd733863dd8ebd47a14fbb411808ad07daaed515c1270978de9157e7
    source_path: cli/browser.md
    workflow: 15
---

# `openclaw browser`

أدر سطح التحكم في المتصفح الخاص بـ OpenClaw ونفّذ إجراءات المتصفح (دورة الحياة، والملفات الشخصية، وعلامات التبويب، واللقطات، ولقطات الشاشة، والتنقل، والإدخال، ومحاكاة الحالة، وتصحيح الأخطاء).

ذو صلة:

- أداة المتصفح + API: ‏[أداة المتصفح](/tools/browser)

## العلامات الشائعة

- `--url <gatewayWsUrl>`: عنوان URL لـ Gateway WebSocket ‏(القيمة الافتراضية من التكوين).
- `--token <token>`: رمز Gateway ‏(إذا كان مطلوبًا).
- `--timeout <ms>`: مهلة الطلب (بالمللي ثانية).
- `--expect-final`: انتظار استجابة Gateway نهائية.
- `--browser-profile <name>`: اختيار ملف تعريف متصفح (القيمة الافتراضية من التكوين).
- `--json`: خرج قابل للقراءة آليًا (حيثما كان مدعومًا).

## البدء السريع (محلي)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

## دورة الحياة

```bash
openclaw browser status
openclaw browser start
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

ملاحظات:

- بالنسبة إلى الملفات الشخصية `attachOnly` وCDP البعيدة، فإن `openclaw browser stop` يغلق
  جلسة التحكم النشطة ويمسح تجاوزات المحاكاة المؤقتة حتى عندما
  لا يكون OpenClaw قد شغّل عملية المتصفح بنفسه.
- بالنسبة إلى الملفات الشخصية المحلية المُدارة، فإن `openclaw browser stop` يوقف
  عملية المتصفح التي تم تشغيلها.

## إذا كان الأمر مفقودًا

إذا كان `openclaw browser` أمرًا غير معروف، فتحقق من `plugins.allow` في
`~/.openclaw/openclaw.json`.

عند وجود `plugins.allow`، يجب إدراج plugin المتصفح المضمّن
بشكل صريح:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

لا يؤدي `browser.enabled=true` إلى استعادة الأمر الفرعي في CLI عندما
تستبعد قائمة السماح الخاصة بـ plugin العنصر `browser`.

ذو صلة: [أداة المتصفح](/tools/browser#missing-browser-command-or-tool)

## الملفات الشخصية

الملفات الشخصية هي إعدادات توجيه متصفح مسماة. عمليًا:

- `openclaw`: يشغّل أو يتصل بمثيل Chrome مخصص ومدار بواسطة OpenClaw ‏(دليل بيانات مستخدم معزول).
- `user`: يتحكم في جلسة Chrome الحالية المسجل الدخول إليها عبر Chrome DevTools MCP.
- ملفات CDP الشخصية المخصصة: تشير إلى نقطة نهاية CDP محلية أو بعيدة.

```bash
openclaw browser profiles
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

استخدم ملفًا شخصيًا محددًا:

```bash
openclaw browser --browser-profile work tabs
```

## علامات التبويب

```bash
openclaw browser tabs
openclaw browser tab new
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai
openclaw browser focus <targetId>
openclaw browser close <targetId>
```

## اللقطة / لقطة الشاشة / الإجراءات

اللقطة:

```bash
openclaw browser snapshot
```

لقطة الشاشة:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
```

ملاحظات:

- `--full-page` مخصص لالتقاطات الصفحة فقط؛ ولا يمكن دمجه مع `--ref`
  أو `--element`.
- تدعم الملفات الشخصية `existing-session` / `user` لقطات شاشة للصفحة ولقطات `--ref`
  من خرج اللقطة، لكنها لا تدعم لقطات CSS ‏`--element`.

التنقل/النقر/الكتابة (أتمتة واجهة مستخدم قائمة على `ref`):

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
```

مساعدات الملفات + مربعات الحوار:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
```

## الحالة والتخزين

منفذ العرض + المحاكاة:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

ملفات تعريف الارتباط + التخزين:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## تصحيح الأخطاء

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## Chrome الحالي عبر MCP

استخدم ملف التعريف المضمّن `user`، أو أنشئ ملف تعريف `existing-session` خاصًا بك:

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser --browser-profile chrome-live tabs
```

هذا المسار خاص بالمضيف فقط. بالنسبة إلى Docker، أو الخوادم عديمة الواجهة، أو Browserless، أو الإعدادات البعيدة الأخرى، استخدم ملف تعريف CDP بدلًا من ذلك.

القيود الحالية لـ existing-session:

- تستخدم الإجراءات المعتمدة على اللقطات مراجع refs، وليس محددات CSS
- `click` يدعم النقر الأيسر فقط
- `type` لا يدعم `slowly=true`
- `press` لا يدعم `delayMs`
- ترفض `hover` و`scrollintoview` و`drag` و`select` و`fill` و`evaluate`
  تجاوزات المهلة لكل استدعاء
- `select` يدعم قيمة واحدة فقط
- `wait --load networkidle` غير مدعوم
- تتطلب عمليات رفع الملفات `--ref` / `--input-ref`، ولا تدعم
  CSS ‏`--element`، وتدعم حاليًا ملفًا واحدًا في كل مرة
- لا تدعم ربطات مربعات الحوار `--timeout`
- تدعم لقطات الشاشة التقاطات الصفحة و`--ref`، لكنها لا تدعم CSS ‏`--element`
- ما زالت `responsebody` واعتراض التنزيلات وتصدير PDF والإجراءات المجمعة
  تتطلب متصفحًا مُدارًا أو ملف تعريف CDP خامًا

## التحكم في المتصفح عن بُعد (proxy مضيف node)

إذا كان Gateway يعمل على جهاز مختلف عن المتصفح، فشغّل **مضيف node** على الجهاز الذي يحتوي على Chrome/Brave/Edge/Chromium. سيقوم Gateway بتمرير إجراءات المتصفح إلى تلك العقدة (ولا يلزم خادم منفصل للتحكم في المتصفح).

استخدم `gateway.nodes.browser.mode` للتحكم في التوجيه التلقائي و`gateway.nodes.browser.node` لتثبيت عقدة محددة إذا كانت هناك عدة عقد متصلة.

الأمان + الإعداد عن بُعد: [أداة المتصفح](/tools/browser)، [الوصول عن بُعد](/gateway/remote)، [Tailscale](/gateway/tailscale)، [الأمان](/gateway/security)
