---
read_when:
    - تريد أن تعرض الوكلاء تعديلات الشيفرة أو Markdown على شكل فروق
    - تريد عنوان URL لعارض جاهز لـ canvas أو ملف فروق مُصيَّر
    - تحتاج إلى عناصر فروق مؤقتة ومضبوطة بإعدادات أمان افتراضية
summary: عارض فروق للقراءة فقط ومُصيِّر ملفات للوكلاء (أداة plugin اختيارية)
title: الفروق
x-i18n:
    generated_at: "2026-04-05T12:58:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 935539a6e584980eb7e57067c18112bb40a0be8522b9da649c7cf7f180fb45d4
    source_path: tools/diffs.md
    workflow: 15
---

# الفروق

`diffs` هي أداة plugin اختيارية تحتوي على إرشادات نظام مضمّنة قصيرة ومهارة مصاحبة تحوّل محتوى التغييرات إلى عنصر فروق للقراءة فقط للوكلاء.

تقبل أحد الخيارين التاليين:

- النص `before` و`after`
- `patch` موحّد

ويمكنها إرجاع:

- عنوان URL لعارض البوابة لعرضه في canvas
- مسار ملف مُصيَّر (PNG أو PDF) لتسليم الرسائل
- كلا المخرجين في استدعاء واحد

عند تفعيلها، تضيف plugin إرشادات استخدام موجزة إلى مساحة system-prompt، كما تكشف أيضًا عن مهارة مفصلة للحالات التي يحتاج فيها الوكيل إلى تعليمات أكثر اكتمالًا.

## بداية سريعة

1. فعّل plugin.
2. استدعِ `diffs` باستخدام `mode: "view"` للتدفقات التي يكون فيها canvas أولًا.
3. استدعِ `diffs` باستخدام `mode: "file"` لتدفقات تسليم الملفات في الدردشة.
4. استدعِ `diffs` باستخدام `mode: "both"` عندما تحتاج إلى كلا العنصرين.

## تفعيل plugin

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
      },
    },
  },
}
```

## تعطيل إرشادات النظام المضمّنة

إذا كنت تريد إبقاء أداة `diffs` مفعلة مع تعطيل إرشاداتها المضمّنة في system-prompt، فاضبط `plugins.entries.diffs.hooks.allowPromptInjection` على `false`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

يمنع هذا hook الخاص بـ `before_prompt_build` في plugin الفروق مع الإبقاء على plugin والأداة والمهارة المصاحبة متاحة.

إذا كنت تريد تعطيل كل من الإرشادات والأداة، فعطّل plugin بدلًا من ذلك.

## سير عمل الوكيل المعتاد

1. يستدعي الوكيل `diffs`.
2. يقرأ الوكيل حقول `details`.
3. ثم يقوم الوكيل بأحد ما يلي:
   - يفتح `details.viewerUrl` باستخدام `canvas present`
   - يرسل `details.filePath` باستخدام `message` مع `path` أو `filePath`
   - أو يفعل الأمرين معًا

## أمثلة الإدخال

قبل وبعد:

```json
{
  "before": "# Hello\n\nOne",
  "after": "# Hello\n\nTwo",
  "path": "docs/example.md",
  "mode": "view"
}
```

Patch:

```json
{
  "patch": "diff --git a/src/example.ts b/src/example.ts\n--- a/src/example.ts\n+++ b/src/example.ts\n@@ -1 +1 @@\n-const x = 1;\n+const x = 2;\n",
  "mode": "both"
}
```

## مرجع مُدخلات الأداة

كل الحقول اختيارية ما لم يُذكر خلاف ذلك:

- `before` (`string`): النص الأصلي. مطلوب مع `after` عند حذف `patch`.
- `after` (`string`): النص المُحدَّث. مطلوب مع `before` عند حذف `patch`.
- `patch` (`string`): نص diff موحّد. وهو متنافر مع `before` و`after`.
- `path` (`string`): اسم ملف العرض لوضع before/after.
- `lang` (`string`): تلميح تجاوز اللغة لوضع before/after. القيم غير المعروفة تعود إلى نص عادي.
- `title` (`string`): تجاوز عنوان العارض.
- `mode` (`"view" | "file" | "both"`): وضع الإخراج. القيمة الافتراضية هي الافتراضي الخاص بـ plugin عند `defaults.mode`.
  الاسم المستعار المهجور: `"image"` يتصرف مثل `"file"` وما زال مقبولًا للتوافق مع الإصدارات السابقة.
- `theme` (`"light" | "dark"`): سمة العارض. القيمة الافتراضية هي الافتراضي الخاص بـ plugin عند `defaults.theme`.
- `layout` (`"unified" | "split"`): تخطيط diff. القيمة الافتراضية هي الافتراضي الخاص بـ plugin عند `defaults.layout`.
- `expandUnchanged` (`boolean`): يوسّع الأقسام غير المتغيرة عندما يكون السياق الكامل متاحًا. خيار لكل استدعاء فقط (وليس مفتاحًا افتراضيًا للـ plugin).
- `fileFormat` (`"png" | "pdf"`): تنسيق الملف المُصيَّر. القيمة الافتراضية هي الافتراضي الخاص بـ plugin عند `defaults.fileFormat`.
- `fileQuality` (`"standard" | "hq" | "print"`): إعداد جودة مسبق لعرض PNG أو PDF.
- `fileScale` (`number`): تجاوز مقياس الجهاز (`1`-`4`).
- `fileMaxWidth` (`number`): أقصى عرض للتصيير بوحدات CSS pixel (`640`-`2400`).
- `ttlSeconds` (`number`): مدة صلاحية العنصر بالثواني لمخرجات العارض والملف المستقل. الافتراضي 1800، والحد الأقصى 21600.
- `baseUrl` (`string`): تجاوز أصل عنوان URL للعارض. يتجاوز `viewerBaseUrl` الخاص بـ plugin. يجب أن يكون `http` أو `https`، ومن دون query/hash.

ما زالت الأسماء المستعارة القديمة للمدخلات مقبولة للتوافق مع الإصدارات السابقة:

- `format` -> `fileFormat`
- `imageFormat` -> `fileFormat`
- `imageQuality` -> `fileQuality`
- `imageScale` -> `fileScale`
- `imageMaxWidth` -> `fileMaxWidth`

التحقق والحدود:

- الحد الأقصى لكل من `before` و`after` هو 512 KiB.
- الحد الأقصى لـ `patch` هو 2 MiB.
- الحد الأقصى لـ `path` هو 2048 بايت.
- الحد الأقصى لـ `lang` هو 128 بايت.
- الحد الأقصى لـ `title` هو 1024 بايت.
- حد تعقيد patch: 128 ملفًا كحد أقصى و120000 سطر إجماليًا.
- يتم رفض استخدام `patch` مع `before` أو `after` معًا.
- حدود أمان الملفات المُصيَّرة (تنطبق على PNG وPDF):
  - `fileQuality: "standard"`: حد أقصى 8 MP (8,000,000 بكسل مُصيَّر).
  - `fileQuality: "hq"`: حد أقصى 14 MP (14,000,000 بكسل مُصيَّر).
  - `fileQuality: "print"`: حد أقصى 24 MP (24,000,000 بكسل مُصيَّر).
  - لدى PDF أيضًا حد أقصى يبلغ 50 صفحة.

## عقد تفاصيل الإخراج

تعيد الأداة بيانات وصفية منظَّمة ضمن `details`.

الحقول المشتركة للأوضاع التي تُنشئ عارضًا:

- `artifactId`
- `viewerUrl`
- `viewerPath`
- `title`
- `expiresAt`
- `inputKind`
- `fileCount`
- `mode`
- `context` (`agentId` و`sessionId` و`messageChannel` و`agentAccountId` عند التوفر)

حقول الملفات عند تصيير PNG أو PDF:

- `artifactId`
- `expiresAt`
- `filePath`
- `path` (القيمة نفسها لـ `filePath`، لتوافق أداة الرسائل)
- `fileBytes`
- `fileFormat`
- `fileQuality`
- `fileScale`
- `fileMaxWidth`

كما تُعاد أيضًا أسماء مستعارة للتوافق مع المستدعين الحاليين:

- `format` (القيمة نفسها لـ `fileFormat`)
- `imagePath` (القيمة نفسها لـ `filePath`)
- `imageBytes` (القيمة نفسها لـ `fileBytes`)
- `imageQuality` (القيمة نفسها لـ `fileQuality`)
- `imageScale` (القيمة نفسها لـ `fileScale`)
- `imageMaxWidth` (القيمة نفسها لـ `fileMaxWidth`)

ملخص سلوك الوضع:

- `mode: "view"`: حقول العارض فقط.
- `mode: "file"`: حقول الملف فقط، من دون عنصر عارض.
- `mode: "both"`: حقول العارض بالإضافة إلى حقول الملف. إذا فشل تصيير الملف، فسيُعاد العارض مع `fileError` والاسم المستعار التوافقي `imageError`.

## الأقسام غير المتغيرة المطوية

- يمكن للعارض أن يعرض أسطرًا مثل `N unmodified lines`.
- عناصر التحكم في التوسيع على هذه الأسطر مشروطة وليست مضمونة لكل نوع إدخال.
- تظهر عناصر التحكم في التوسيع عندما يحتوي diff المُصيَّر على بيانات سياق قابلة للتوسيع، وهو أمر شائع في مدخلات before/after.
- بالنسبة إلى كثير من مدخلات patch الموحّدة، لا تكون أجسام السياق المحذوفة متاحة في أجزاء patch المحللة، لذلك قد يظهر السطر من دون عناصر تحكم للتوسيع. هذا سلوك متوقَّع.
- ينطبق `expandUnchanged` فقط عندما يكون هناك سياق قابل للتوسيع.

## القيم الافتراضية للـ plugin

اضبط القيم الافتراضية على مستوى plugin في `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
          },
        },
      },
    },
  },
}
```

القيم الافتراضية المدعومة:

- `fontFamily`
- `fontSize`
- `lineSpacing`
- `layout`
- `showLineNumbers`
- `diffIndicators`
- `wordWrap`
- `background`
- `theme`
- `fileFormat`
- `fileQuality`
- `fileScale`
- `fileMaxWidth`
- `mode`

تتجاوز معلمات الأداة الصريحة هذه القيم الافتراضية.

إعداد عنوان URL الدائم للعارض:

- `viewerBaseUrl` (`string`، اختياري)
  - بديل مملوك للـ plugin للروابط المعادة الخاصة بالعارض عندما لا يمرر استدعاء الأداة `baseUrl`.
  - يجب أن يكون `http` أو `https`، ومن دون query/hash.

مثال:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## إعدادات الأمان

- `security.allowRemoteViewer` (`boolean`، الافتراضي `false`)
  - `false`: تُرفَض الطلبات غير القادمة من loopback إلى مسارات العارض.
  - `true`: يُسمح بالعارضات البعيدة إذا كان المسار المرمّز صالحًا.

مثال:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## دورة حياة العناصر والتخزين

- تُخزَّن العناصر ضمن المجلد الفرعي المؤقت: `$TMPDIR/openclaw-diffs`.
- تحتوي البيانات الوصفية لعنصر العارض على:
  - معرّف عنصر عشوائي (20 حرف hex)
  - رمزًا مميزًا عشوائيًا (48 حرف hex)
  - `createdAt` و`expiresAt`
  - مسار `viewer.html` المخزَّن
- مدة الصلاحية الافتراضية للعنصر هي 30 دقيقة عند عدم تحديدها.
- الحد الأقصى المقبول لمدة صلاحية العارض هو 6 ساعات.
- تعمل عملية التنظيف بشكل انتهازي بعد إنشاء العنصر.
- تُحذف العناصر منتهية الصلاحية.
- يزيل التنظيف الاحتياطي المجلدات القديمة الأثر الأقدم من 24 ساعة عندما تكون البيانات الوصفية مفقودة.

## عنوان URL للعارض وسلوك الشبكة

مسار العارض:

- `/plugins/diffs/view/{artifactId}/{token}`

أصول العارض:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`

يحل مستند العارض هذه الأصول نسبةً إلى عنوان URL الخاص بالعارض، لذلك يُحفَظ أيضًا أي بادئة مسار اختيارية لـ `baseUrl` في طلبات الأصول.

سلوك بناء عنوان URL:

- إذا جرى توفير `baseUrl` في استدعاء الأداة، فسيُستخدَم بعد تحقق صارم.
- وإلا إذا كان `viewerBaseUrl` الخاص بـ plugin مضبوطًا، فسيُستخدَم.
- من دون أي تجاوز، تكون القيمة الافتراضية لعنوان URL للعارض هي loopback `127.0.0.1`.
- إذا كان وضع ربط Gateway هو `custom` وكان `gateway.customBindHost` مضبوطًا، فسيُستخدَم ذلك المضيف.

قواعد `baseUrl`:

- يجب أن يبدأ بـ `http://` أو `https://`.
- يتم رفض query وhash.
- يُسمح بالأصل مع مسار أساسي اختياري.

## نموذج الأمان

تقوية العارض:

- loopback فقط افتراضيًا.
- مسارات عارض مميّزة برمز مع تحقق صارم من المعرّف والرمز المميز.
- سياسة CSP لاستجابة العارض:
  - `default-src 'none'`
  - السكربتات والأصول من self فقط
  - لا يوجد `connect-src` صادر
- تقييد محاولات الفشل البعيدة عند تفعيل الوصول البعيد:
  - 40 فشلًا لكل 60 ثانية
  - حظر لمدة 60 ثانية (`429 Too Many Requests`)

تقوية تصيير الملفات:

- توجيه طلبات متصفح لقطات الشاشة مُعطَّل افتراضيًا ما لم يُسمح به صراحةً.
- يُسمح فقط بأصول العارض المحلية من `http://127.0.0.1/plugins/diffs/assets/*`.
- تُحظر طلبات الشبكة الخارجية.

## متطلبات المتصفح لوضع الملف

يحتاج `mode: "file"` و`mode: "both"` إلى متصفح متوافق مع Chromium.

ترتيب التحليل:

1. `browser.executablePath` في تكوين OpenClaw.
2. متغيرات البيئة:
   - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
   - `BROWSER_EXECUTABLE_PATH`
   - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`
3. الرجوع إلى آلية اكتشاف أوامر/مسارات المنصة.

نص الإخفاق الشائع:

- `Diff PNG/PDF rendering requires a Chromium-compatible browser...`

يمكن إصلاح ذلك بتثبيت Chrome أو Chromium أو Edge أو Brave، أو بضبط أحد خيارات مسار الملف التنفيذي أعلاه.

## استكشاف الأخطاء وإصلاحها

أخطاء التحقق من الإدخال:

- `Provide patch or both before and after text.`
  - أدرج كلًا من `before` و`after`، أو وفّر `patch`.
- `Provide either patch or before/after input, not both.`
  - لا تخلط بين أوضاع الإدخال.
- `Invalid baseUrl: ...`
  - استخدم أصل `http(s)` مع مسار اختياري، ومن دون query/hash.
- `{field} exceeds maximum size (...)`
  - قلّل حجم الحمولة.
- رفض patch كبير
  - قلّل عدد ملفات patch أو إجمالي الأسطر.

مشكلات إمكانية الوصول إلى العارض:

- يُحل عنوان URL للعارض إلى `127.0.0.1` افتراضيًا.
- في سيناريوهات الوصول البعيد، إمّا:
  - اضبط `viewerBaseUrl` الخاص بـ plugin، أو
  - مرّر `baseUrl` لكل استدعاء أداة، أو
  - استخدم `gateway.bind=custom` و`gateway.customBindHost`
- إذا كان `gateway.trustedProxies` يتضمن loopback لبروكسي على المضيف نفسه (مثل Tailscale Serve)، فإن طلبات العارض الخام عبر loopback من دون رؤوس forwarded client-IP تفشل بشكل مغلق حسب التصميم.
- بالنسبة إلى هذه البنية للبروكسي:
  - يُفضّل `mode: "file"` أو `mode: "both"` عندما تحتاج فقط إلى مرفق، أو
  - فعّل `security.allowRemoteViewer` عمدًا واضبط `viewerBaseUrl` الخاص بـ plugin أو مرّر `baseUrl` للبروكسي/العامة عندما تحتاج إلى عنوان URL قابل للمشاركة للعارض
- فعّل `security.allowRemoteViewer` فقط عندما تقصد تمكين وصول خارجي إلى العارض.

صف الأسطر غير المعدلة لا يحتوي على زر توسيع:

- قد يحدث هذا مع مدخلات patch عندما لا يحمل patch سياقًا قابلًا للتوسيع.
- هذا متوقع ولا يدل على فشل في العارض.

العنصر غير موجود:

- انتهت صلاحية العنصر بسبب TTL.
- تغيّر الرمز المميز أو المسار.
- أزال التنظيف البيانات القديمة.

## إرشادات التشغيل

- يُفضّل `mode: "view"` للمراجعات التفاعلية المحلية داخل canvas.
- يُفضّل `mode: "file"` لقنوات الدردشة الصادرة التي تحتاج إلى مرفق.
- أبقِ `allowRemoteViewer` معطّلًا ما لم يتطلب نشرُك عناوين URL بعيدة للعارض.
- اضبط `ttlSeconds` قصيرة وصريحة للفروق الحساسة.
- تجنب إرسال الأسرار ضمن إدخال diff عندما لا يكون ذلك مطلوبًا.
- إذا كانت قناتك تضغط الصور بقوة (مثل Telegram أو WhatsApp)، ففضّل إخراج PDF (`fileFormat: "pdf"`).

محرك تصيير الفروق:

- يعمل بواسطة [Diffs](https://diffs.com).

## الوثائق ذات الصلة

- [نظرة عامة على الأدوات](/tools)
- [Plugins](/tools/plugin)
- [Browser](/tools/browser)
