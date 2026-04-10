---
read_when:
    - إضافة أتمتة متصفح يتحكم فيها الوكيل
    - تصحيح سبب تداخل openclaw مع Chrome الخاص بك
    - تنفيذ إعدادات المتصفح + دورة الحياة في تطبيق macOS
summary: خدمة تحكم متكاملة بالمتصفح + أوامر الإجراءات
title: المتصفح (بإدارة OpenClaw)
x-i18n:
    generated_at: "2026-04-10T07:18:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: cd3424f62178bbf25923b8bc8e4d9f70e330f35428d01fe153574e5fa45d7604
    source_path: tools/browser.md
    workflow: 15
---

# المتصفح (بإدارة openclaw)

يمكن لـ OpenClaw تشغيل **ملف تعريف مخصص لـ Chrome/Brave/Edge/Chromium** يتحكم فيه الوكيل.
وهو معزول عن متصفحك الشخصي ويُدار عبر خدمة تحكم محلية صغيرة
داخل Gateway (على loopback فقط).

عرض للمبتدئين:

- اعتبره **متصفحًا منفصلًا مخصصًا للوكيل فقط**.
- ملف تعريف `openclaw` **لا** يلمس ملف تعريف متصفحك الشخصي.
- يمكن للوكيل **فتح علامات تبويب، وقراءة الصفحات، والنقر، والكتابة** ضمن مسار آمن.
- يرتبط ملف تعريف `user` المضمّن بجلسة Chrome الحقيقية التي سجّلت الدخول إليها عبر Chrome MCP.

## ما الذي ستحصل عليه

- ملف تعريف متصفح منفصل باسم **openclaw** (بتمييز برتقالي افتراضيًا).
- تحكم حتمي في علامات التبويب (عرض/فتح/تركيز/إغلاق).
- إجراءات الوكيل (نقر/كتابة/سحب/تحديد)، ولقطات حالة، ولقطات شاشة، وملفات PDF.
- دعم اختياري لعدة ملفات تعريف (`openclaw` و`work` و`remote` و...).

هذا المتصفح **ليس** متصفحك اليومي. إنه سطح آمن ومعزول
لأتمتة الوكيل والتحقق.

## البدء السريع

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

إذا ظهرت لك رسالة “المتصفح معطّل”، فقم بتمكينه في الإعدادات (انظر أدناه) ثم أعد تشغيل
Gateway.

إذا كان `openclaw browser` مفقودًا بالكامل، أو إذا قال الوكيل إن أداة المتصفح
غير متاحة، فانتقل إلى [أمر المتصفح أو الأداة مفقود](/ar/tools/browser#missing-browser-command-or-tool).

## التحكم عبر الإضافات

أصبحت أداة `browser` الافتراضية الآن إضافة مضمّنة تُشحن وهي مفعّلة
افتراضيًا. وهذا يعني أنه يمكنك تعطيلها أو استبدالها دون إزالة بقية
نظام إضافات OpenClaw:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

عطّل الإضافة المضمّنة قبل تثبيت إضافة أخرى توفّر اسم أداة `browser`
نفسه. تتطلب تجربة المتصفح الافتراضية الأمرين التاليين معًا:

- ألا يكون `plugins.entries.browser.enabled` معطّلًا
- وأن يكون `browser.enabled=true`

إذا عطّلت الإضافة فقط، فسيختفي معًا CLI المتصفح المضمّن (`openclaw browser`)،
وطريقة gateway (`browser.request`)، وأداة الوكيل، وخدمة التحكم الافتراضية بالمتصفح.
بينما تظل إعدادات `browser.*` لديك كما هي ليستفيد منها
إضافة بديلة.

أصبحت الإضافة المضمّنة للمتصفح تملك أيضًا تنفيذ وقت تشغيل المتصفح الآن.
ويحتفظ النواة فقط بمساعدات Plugin SDK المشتركة إضافةً إلى
إعادة تصدير التوافق لمسارات الاستيراد الداخلية الأقدم. عمليًا، فإن إزالة
حزمة إضافة المتصفح أو استبدالها يزيل مجموعة ميزات المتصفح بدلًا من
ترك وقت تشغيل ثانٍ تملكه النواة في الخلفية.

ما تزال تغييرات إعدادات المتصفح تتطلب إعادة تشغيل Gateway حتى تتمكن
الإضافة المضمّنة من إعادة تسجيل خدمة المتصفح بإعدادات جديدة.

## أمر المتصفح أو الأداة مفقود

إذا أصبح `openclaw browser` فجأة أمرًا غير معروف بعد الترقية، أو
أبلغ الوكيل عن غياب أداة المتصفح، فالسبب الأكثر شيوعًا هو وجود قائمة
`plugins.allow` مقيّدة لا تتضمن `browser`.

مثال على إعداد معطّل:

```json5
{
  plugins: {
    allow: ["telegram"],
  },
}
```

أصلحه بإضافة `browser` إلى قائمة السماح الخاصة بالإضافات:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

ملاحظات مهمة:

- لا يكفي `browser.enabled=true` وحده عندما يكون `plugins.allow` مضبوطًا.
- ولا يكفي أيضًا `plugins.entries.browser.enabled=true` وحده عندما يكون `plugins.allow` مضبوطًا.
- إن `tools.alsoAllow: ["browser"]` **لا** يحمّل إضافة المتصفح المضمّنة. فهو يضبط فقط سياسة الأدوات بعد أن تكون الإضافة قد حُمّلت بالفعل.
- إذا لم تكن بحاجة إلى قائمة سماح مقيّدة للإضافات، فإن إزالة `plugins.allow` تعيد أيضًا سلوك المتصفح المضمّن الافتراضي.

الأعراض المعتادة:

- يكون `openclaw browser` أمرًا غير معروف.
- يكون `browser.request` مفقودًا.
- يبلغ الوكيل أن أداة المتصفح غير متاحة أو مفقودة.

## ملفات التعريف: `openclaw` مقابل `user`

- `openclaw`: متصفح مُدار ومعزول (لا يتطلب إضافة).
- `user`: ملف تعريف ارتباط Chrome MCP مضمّن لجلسة **Chrome الحقيقية المسجّل الدخول إليها** لديك.

بالنسبة إلى استدعاءات أداة المتصفح من الوكيل:

- الافتراضي: استخدم متصفح `openclaw` المعزول.
- فضّل `profile="user"` عندما تكون الجلسات الحالية المسجّل الدخول إليها مهمة ويكون المستخدم
  أمام الكمبيوتر للنقر/الموافقة على أي مطالبة بالارتباط.
- `profile` هو التجاوز الصريح عندما تريد وضع متصفح محددًا.

اضبط `browser.defaultProfile: "openclaw"` إذا كنت تريد الوضع المُدار افتراضيًا.

## الإعدادات

توجد إعدادات المتصفح في `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // الافتراضي: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // وضع الشبكة الموثوقة الافتراضي
      // allowPrivateNetwork: true, // اسم مستعار قديم
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // تجاوز قديم لملف تعريف واحد
    remoteCdpTimeoutMs: 1500, // مهلة HTTP البعيدة لـ CDP (مللي ثانية)
    remoteCdpHandshakeTimeoutMs: 3000, // مهلة مصافحة WebSocket البعيدة لـ CDP (مللي ثانية)
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

ملاحظات:

- ترتبط خدمة التحكم بالمتصفح على loopback بمنفذ مشتق من `gateway.port`
  (الافتراضي: `18791`، أي gateway + 2).
- إذا تجاوزت منفذ Gateway (`gateway.port` أو `OPENCLAW_GATEWAY_PORT`)،
  فإن منافذ المتصفح المشتقة تتحرك للحفاظ على بقائها ضمن “العائلة” نفسها.
- يكون `cdpUrl` افتراضيًا هو منفذ CDP المحلي المُدار عندما لا يكون مضبوطًا.
- ينطبق `remoteCdpTimeoutMs` على فحوصات إمكانية الوصول البعيدة لـ CDP (غير loopback).
- ينطبق `remoteCdpHandshakeTimeoutMs` على فحوصات إمكانية الوصول لمصافحة WebSocket البعيدة لـ CDP.
- يخضع التنقل/فتح علامات التبويب في المتصفح لحماية SSRF قبل التنقل، ويُعاد فحصه قدر الإمكان على عنوان `http(s)` النهائي بعد التنقل.
- في وضع SSRF الصارم، يتم أيضًا فحص اكتشاف/استطلاع نقاط نهاية CDP البعيدة (`cdpUrl`، بما في ذلك عمليات البحث `/json/version`).
- يكون `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` افتراضيًا مضبوطًا على `true` (نموذج الشبكة الموثوقة). اضبطه على `false` للتصفح الصارم العام فقط.
- يظل `browser.ssrfPolicy.allowPrivateNetwork` مدعومًا كاسم مستعار قديم لأغراض التوافق.
- يعني `attachOnly: true` أنه “لا تُشغّل متصفحًا محليًا أبدًا؛ بل ارتبط به فقط إذا كان يعمل بالفعل.”
- يضيف `color` إضافةً إلى `color` لكل ملف تعريف تلوينًا لواجهة المتصفح حتى تتمكن من رؤية الملف النشط.
- ملف التعريف الافتراضي هو `openclaw` (متصفح مستقل بإدارة OpenClaw). استخدم `defaultProfile: "user"` للاشتراك في متصفح المستخدم المسجّل الدخول إليه.
- ترتيب الاكتشاف التلقائي: متصفح النظام الافتراضي إذا كان معتمدًا على Chromium؛ وإلا Chrome ← Brave ← Edge ← Chromium ← Chrome Canary.
- تقوم ملفات تعريف `openclaw` المحلية بتعيين `cdpPort`/`cdpUrl` تلقائيًا — اضبطهما فقط من أجل CDP البعيد.
- يستخدم `driver: "existing-session"` Chrome DevTools MCP بدلًا من CDP الخام. لا
  تضبط `cdpUrl` لهذا المشغّل.
- اضبط `browser.profiles.<name>.userDataDir` عندما يجب أن يرتبط ملف تعريف existing-session
  بملف تعريف مستخدم Chromium غير افتراضي مثل Brave أو Edge.

## استخدام Brave (أو متصفح آخر قائم على Chromium)

إذا كان متصفحك **الافتراضي على النظام** قائمًا على Chromium (Chrome/Brave/Edge/إلخ)،
فإن OpenClaw يستخدمه تلقائيًا. اضبط `browser.executablePath` لتجاوز
الاكتشاف التلقائي:

مثال CLI:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## التحكم المحلي مقابل التحكم البعيد

- **التحكم المحلي (الافتراضي):** يبدأ Gateway خدمة التحكم على loopback ويمكنه تشغيل متصفح محلي.
- **التحكم البعيد (مضيف العقدة):** شغّل مضيف عقدة على الجهاز الذي يحتوي على المتصفح؛ وسيقوم Gateway بتمرير إجراءات المتصفح إليه.
- **CDP البعيد:** اضبط `browser.profiles.<name>.cdpUrl` (أو `browser.cdpUrl`) من أجل
  الارتباط بمتصفح بعيد قائم على Chromium. في هذه الحالة، لن يقوم OpenClaw بتشغيل متصفح محلي.

يختلف سلوك الإيقاف حسب وضع ملف التعريف:

- ملفات التعريف المحلية المُدارة: يوقف `openclaw browser stop` عملية المتصفح التي
  شغّلها OpenClaw
- ملفات تعريف attach-only وCDP البعيدة: يغلق `openclaw browser stop` جلسة
  التحكم النشطة ويحرر تجاوزات محاكاة Playwright/CDP (منفذ العرض،
  ونظام الألوان، واللغة المحلية، والمنطقة الزمنية، ووضع عدم الاتصال، والحالة المشابهة)، حتى
  لو لم يشغّل OpenClaw أي عملية متصفح

يمكن أن تتضمن عناوين URL الخاصة بـ CDP البعيد مصادقة:

- رموز استعلام (مثل `https://provider.example?token=<token>`)
- مصادقة HTTP Basic (مثل `https://user:pass@provider.example`)

يحافظ OpenClaw على بيانات المصادقة عند استدعاء نقاط النهاية `/json/*` وعند الاتصال
بـ WebSocket الخاص بـ CDP. فضّل متغيرات البيئة أو مديري الأسرار للرموز
بدلًا من تضمينها في ملفات الإعدادات.

## وكيل متصفح العقدة (الافتراضي من دون إعدادات)

إذا كنت تشغّل **مضيف عقدة** على الجهاز الذي يحتوي على متصفحك، فيمكن لـ OpenClaw
توجيه استدعاءات أداة المتصفح تلقائيًا إلى تلك العقدة من دون أي إعدادات متصفح إضافية.
وهذا هو المسار الافتراضي للـ Gateways البعيدة.

ملاحظات:

- يكشف مضيف العقدة عن خادم التحكم المحلي في المتصفح عبر **أمر وكيل**.
- تأتي ملفات التعريف من إعداد `browser.profiles` الخاص بالعقدة نفسها (مثل المحلي).
- إن `nodeHost.browserProxy.allowProfiles` اختياري. اتركه فارغًا للحصول على السلوك القديم/الافتراضي: تبقى جميع ملفات التعريف المضبوطة قابلة للوصول عبر الوكيل، بما في ذلك مسارات إنشاء/حذف ملفات التعريف.
- إذا ضبطت `nodeHost.browserProxy.allowProfiles`، فسيتعامل OpenClaw معه على أنه حد أقل الامتيازات: لا يمكن استهداف إلا ملفات التعريف الموجودة في قائمة السماح، وتُحظر مسارات إنشاء/حذف ملفات التعريف الدائمة على سطح الوكيل.
- عطّله إذا كنت لا تريده:
  - على العقدة: `nodeHost.browserProxy.enabled=false`
  - على gateway: `gateway.nodes.browser.mode="off"`

## Browserless (CDP بعيد مستضاف)

[Browserless](https://browserless.io) خدمة Chromium مستضافة تكشف
عناوين اتصال CDP عبر HTTPS وWebSocket. يمكن لـ OpenClaw استخدام أي من الشكلين، لكن
بالنسبة إلى ملف تعريف متصفح بعيد فإن أبسط خيار هو عنوان WebSocket المباشر
من وثائق الاتصال الخاصة بـ Browserless.

مثال:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

ملاحظات:

- استبدل `<BROWSERLESS_API_KEY>` برمز Browserless الحقيقي الخاص بك.
- اختر نقطة نهاية المنطقة التي تطابق حساب Browserless لديك (راجع وثائقهم).
- إذا زوّدك Browserless بعنوان أساسي HTTPS، فيمكنك إما تحويله إلى
  `wss://` لاتصال CDP مباشر أو الإبقاء على عنوان HTTPS وترك OpenClaw
  يكتشف `/json/version`.

## موفّرو WebSocket CDP المباشر

تكشف بعض خدمات المتصفح المستضافة نقطة نهاية **WebSocket مباشرة** بدلًا من
الاكتشاف القياسي لـ CDP المعتمد على HTTP (`/json/version`). يدعم OpenClaw كلاهما:

- **نقاط نهاية HTTP(S)** — يستدعي OpenClaw ‎`/json/version` لاكتشاف
  عنوان WebSocket الخاص بالمصحح، ثم يتصل به.
- **نقاط نهاية WebSocket** (`ws://` / `wss://`) — يتصل OpenClaw مباشرة،
  متجاوزًا ‎`/json/version`. استخدم هذا للخدمات مثل
  [Browserless](https://browserless.io)،
  [Browserbase](https://www.browserbase.com)، أو أي موفّر يسلّمك
  عنوان WebSocket.

### Browserbase

[Browserbase](https://www.browserbase.com) منصة سحابية لتشغيل
متصفحات headless مع حل CAPTCHA مضمّن، ووضع التخفي، ووكلاء سكنيين.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

ملاحظات:

- [سجّل](https://www.browserbase.com/sign-up) وانسخ **مفتاح API**
  من [لوحة Overview](https://www.browserbase.com/overview).
- استبدل `<BROWSERBASE_API_KEY>` بمفتاح API الحقيقي الخاص بـ Browserbase.
- ينشئ Browserbase جلسة متصفح تلقائيًا عند الاتصال عبر WebSocket، لذا لا
  حاجة إلى خطوة إنشاء جلسة يدويًا.
- تتيح الخطة المجانية جلسة متزامنة واحدة وساعة متصفح واحدة شهريًا.
  راجع [الأسعار](https://www.browserbase.com/pricing) لمعرفة حدود الخطط المدفوعة.
- راجع [وثائق Browserbase](https://docs.browserbase.com) للحصول على مرجع API
  الكامل، وأدلة SDK، وأمثلة التكامل.

## الأمان

الأفكار الأساسية:

- يكون التحكم في المتصفح على loopback فقط؛ ويمر الوصول عبر مصادقة Gateway أو اقتران العقدة.
- تستخدم واجهة HTTP المستقلة للمتصفح على loopback **مصادقة السرّ المشترك فقط**:
  رمز gateway بطريقة bearer، أو `x-openclaw-password`، أو HTTP Basic auth باستخدام
  كلمة مرور gateway المضبوطة.
- لا توفّر رؤوس هوية Tailscale Serve و`gateway.auth.mode: "trusted-proxy"`
  **مصادقة** لواجهة API المستقلة هذه الخاصة بالمتصفح على loopback.
- إذا كان التحكم في المتصفح مفعّلًا ولم تُضبط أي مصادقة سرّ مشترك، فإن OpenClaw
  ينشئ `gateway.auth.token` تلقائيًا عند بدء التشغيل ويحفظه في الإعدادات.
- لا ينشئ OpenClaw هذا الرمز تلقائيًا عندما يكون `gateway.auth.mode`
  مضبوطًا أصلًا على `password` أو `none` أو `trusted-proxy`.
- أبقِ Gateway وأي مضيفات عقدة على شبكة خاصة (Tailscale)؛ وتجنب تعريضها علنًا.
- تعامل مع عناوين URL/الرموز الخاصة بـ CDP البعيد على أنها أسرار؛ وفضّل متغيرات البيئة أو مدير أسرار.

نصائح CDP البعيد:

- فضّل نقاط النهاية المشفّرة (HTTPS أو WSS) والرموز قصيرة العمر متى أمكن.
- تجنّب تضمين الرموز طويلة العمر مباشرةً في ملفات الإعدادات.

## ملفات التعريف (متعدد المتصفحات)

يدعم OpenClaw عدة ملفات تعريف مسماة (إعدادات توجيه). يمكن أن تكون ملفات التعريف:

- **بإدارة openclaw**: نسخة متصفح مستقلة قائمة على Chromium مع دليل بيانات مستخدم خاص بها + منفذ CDP
- **بعيدة**: عنوان CDP URL صريح (متصفح قائم على Chromium يعمل في مكان آخر)
- **جلسة موجودة**: ملف تعريف Chrome الحالي لديك عبر الاتصال التلقائي لـ Chrome DevTools MCP

الإعدادات الافتراضية:

- يُنشأ ملف تعريف `openclaw` تلقائيًا إذا كان مفقودًا.
- ملف تعريف `user` مضمّن من أجل ارتباط Chrome MCP بجلسة موجودة.
- ملفات تعريف الجلسة الموجودة هي خيار اشتراك إضافي بعد `user`؛ أنشئها باستخدام `--driver existing-session`.
- تُخصَّص منافذ CDP المحلية من **18800–18899** افتراضيًا.
- يؤدي حذف ملف تعريف إلى نقل دليل بياناته المحلي إلى سلة المهملات.

تقبل جميع نقاط نهاية التحكم ‎`?profile=<name>`؛ ويستخدم CLI الخيار `--browser-profile`.

## الجلسة الموجودة عبر Chrome DevTools MCP

يمكن لـ OpenClaw أيضًا الارتباط بملف تعريف متصفح قائم على Chromium يعمل حاليًا عبر
خادم Chrome DevTools MCP الرسمي. وهذا يعيد استخدام علامات التبويب وحالة تسجيل الدخول
المفتوحة بالفعل في ملف تعريف المتصفح ذلك.

مراجع الخلفية والإعداد الرسمية:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

ملف التعريف المضمّن:

- `user`

اختياري: أنشئ ملف تعريف existing-session مخصصًا إذا كنت تريد
اسمًا مختلفًا، أو لونًا مختلفًا، أو دليل بيانات متصفح مختلفًا.

السلوك الافتراضي:

- يستخدم ملف تعريف `user` المضمّن الاتصال التلقائي لـ Chrome MCP، والذي يستهدف
  ملف تعريف Google Chrome المحلي الافتراضي.

استخدم `userDataDir` لـ Brave أو Edge أو Chromium أو ملف تعريف Chrome غير افتراضي:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

ثم في المتصفح المطابق:

1. افتح صفحة inspect الخاصة بذلك المتصفح من أجل التصحيح البعيد.
2. فعّل التصحيح البعيد.
3. أبقِ المتصفح قيد التشغيل ووافق على مطالبة الاتصال عندما يرتبط OpenClaw.

صفحات inspect الشائعة:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

اختبار smoke للارتباط المباشر:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

شكل النجاح:

- يعرض `status` القيمة `driver: existing-session`
- يعرض `status` القيمة `transport: chrome-mcp`
- يعرض `status` القيمة `running: true`
- يعرض `tabs` علامات التبويب المفتوحة لديك بالفعل في المتصفح
- يعيد `snapshot` مراجع من علامة التبويب الحية المحددة

ما الذي يجب التحقق منه إذا لم يعمل الارتباط:

- أن يكون إصدار المتصفح المستهدف القائم على Chromium هو `144+`
- أن يكون التصحيح البعيد مفعّلًا في صفحة inspect لذلك المتصفح
- أن المتصفح عرض مطالبة موافقة الارتباط وأنك قبلتها
- يقوم `openclaw doctor` بترحيل إعدادات المتصفح القديمة المعتمدة على الإضافات ويتحقق من
  أن Chrome مثبّت محليًا لملفات تعريف الاتصال التلقائي الافتراضية، لكنه لا يستطيع
  تفعيل التصحيح البعيد في جهة المتصفح نيابةً عنك

استخدام الوكيل:

- استخدم `profile="user"` عندما تحتاج إلى حالة المتصفح التي سجّل المستخدم الدخول بها.
- إذا كنت تستخدم ملف تعريف existing-session مخصصًا، فمرّر اسم ملف التعريف الصريح هذا.
- اختر هذا الوضع فقط عندما يكون المستخدم أمام الكمبيوتر للموافقة على مطالبة
  الارتباط.
- يمكن لـ Gateway أو مضيف العقدة تشغيل `npx chrome-devtools-mcp@latest --autoConnect`

ملاحظات:

- هذا المسار أعلى خطورة من ملف تعريف `openclaw` المعزول لأنه يمكنه
  العمل داخل جلسة المتصفح المسجّل الدخول إليها لديك.
- لا يشغّل OpenClaw المتصفح لهذا المشغّل؛ بل يرتبط فقط بجلسة
  موجودة.
- يستخدم OpenClaw هنا التدفق الرسمي `--autoConnect` الخاص بـ Chrome DevTools MCP. إذا
  كان `userDataDir` مضبوطًا، فإن OpenClaw يمرّره لاستهداف
  دليل بيانات مستخدم Chromium الصريح ذلك.
- تدعم لقطات الشاشة في existing-session لقطات الصفحة ولقطات العناصر عبر `--ref`
  من خرج snapshot، لكن ليس محددات CSS `--element`.
- تعمل لقطات شاشة الصفحة في existing-session من دون Playwright عبر Chrome MCP.
  كما تعمل أيضًا لقطات العناصر المعتمدة على المرجع (`--ref`) هناك، لكن لا يمكن
  دمج `--full-page` مع `--ref` أو `--element`.
- لا تزال إجراءات existing-session أكثر محدودية من
  مسار المتصفح المُدار:
  - تتطلب `click` و`type` و`hover` و`scrollIntoView` و`drag` و`select`
    مراجع snapshot بدلًا من محددات CSS
  - `click` لزر الفأرة الأيسر فقط (من دون تجاوزات للأزرار أو المعدّلات)
  - لا يدعم `type` الخيار `slowly=true`؛ استخدم `fill` أو `press`
  - لا يدعم `press` الخيار `delayMs`
  - لا تدعم `hover` و`scrollIntoView` و`drag` و`select` و`fill` و`evaluate`
    تجاوزات المهلة لكل استدعاء
  - يدعم `select` حاليًا قيمة واحدة فقط
- يدعم `wait --url` في existing-session الأنماط الدقيقة وأنماط المطابقة الجزئية وglob
  مثل باقي مشغّلات المتصفح. أما `wait --load networkidle` فغير مدعوم بعد.
- تتطلب hooks الرفع في existing-session قيمة `ref` أو `inputRef`، وتدعم ملفًا واحدًا
  في كل مرة، ولا تدعم استهداف CSS عبر `element`.
- لا تدعم hooks مربعات الحوار في existing-session تجاوزات المهلة.
- لا تزال بعض الميزات تتطلب مسار المتصفح المُدار، بما في ذلك الإجراءات
  الدفعية، وتصدير PDF، واعتراض التنزيلات، و`responsebody`.
- existing-session محلي على المضيف. إذا كان Chrome موجودًا على جهاز مختلف أو ضمن
  namespace شبكة مختلفة، فاستخدم CDP البعيد أو مضيف عقدة بدلًا من ذلك.

## ضمانات العزل

- **دليل بيانات مستخدم مخصص**: لا يلمس أبدًا ملف تعريف متصفحك الشخصي.
- **منافذ مخصصة**: يتجنب `9222` لمنع التعارضات مع سير عمل التطوير.
- **تحكم حتمي في علامات التبويب**: يستهدف علامات التبويب عبر `targetId`، وليس “آخر علامة تبويب”.

## اختيار المتصفح

عند التشغيل محليًا، يختار OpenClaw أول متصفح متاح:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

يمكنك التجاوز باستخدام `browser.executablePath`.

المنصات:

- macOS: يتحقق من `/Applications` و`~/Applications`.
- Linux: يبحث عن `google-chrome` و`brave` و`microsoft-edge` و`chromium` وما إلى ذلك.
- Windows: يتحقق من مواقع التثبيت الشائعة.

## واجهة API للتحكم (اختيارية)

بالنسبة إلى عمليات التكامل المحلية فقط، يوفّر Gateway واجهة HTTP صغيرة على loopback:

- الحالة/البدء/الإيقاف: `GET /` و`POST /start` و`POST /stop`
- علامات التبويب: `GET /tabs` و`POST /tabs/open` و`POST /tabs/focus` و`DELETE /tabs/:targetId`
- snapshot/screenshot: `GET /snapshot` و`POST /screenshot`
- الإجراءات: `POST /navigate` و`POST /act`
- Hooks: `POST /hooks/file-chooser` و`POST /hooks/dialog`
- التنزيلات: `POST /download` و`POST /wait/download`
- التصحيح: `GET /console` و`POST /pdf`
- التصحيح: `GET /errors` و`GET /requests` و`POST /trace/start` و`POST /trace/stop` و`POST /highlight`
- الشبكة: `POST /response/body`
- الحالة: `GET /cookies` و`POST /cookies/set` و`POST /cookies/clear`
- الحالة: `GET /storage/:kind` و`POST /storage/:kind/set` و`POST /storage/:kind/clear`
- الإعدادات: `POST /set/offline` و`POST /set/headers` و`POST /set/credentials` و`POST /set/geolocation` و`POST /set/media` و`POST /set/timezone` و`POST /set/locale` و`POST /set/device`

تقبل جميع نقاط النهاية ‎`?profile=<name>`.

إذا كانت مصادقة gateway بالسرّ المشترك مضبوطة، فإن مسارات HTTP الخاصة بالمتصفح تتطلب المصادقة أيضًا:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` أو HTTP Basic auth باستخدام كلمة المرور هذه

ملاحظات:

- لا تستهلك واجهة API المستقلة هذه الخاصة بالمتصفح على loopback
  وضع trusted-proxy أو رؤوس هوية Tailscale Serve.
- إذا كان `gateway.auth.mode` هو `none` أو `trusted-proxy`، فإن مسارات المتصفح هذه على loopback
  لا ترث أوضاع الهوية الحاملة هذه؛ لذا أبقِها على loopback فقط.

### عقد أخطاء `/act`

يستخدم `POST /act` استجابة خطأ منظَّمة لفشل التحقق على مستوى المسار
وفشل السياسات:

```json
{ "error": "<message>", "code": "ACT_*" }
```

قيم `code` الحالية:

- `ACT_KIND_REQUIRED` (HTTP 400): القيمة `kind` مفقودة أو غير معروفة.
- `ACT_INVALID_REQUEST` (HTTP 400): فشلت حمولة الإجراء في التطبيع أو التحقق.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): استُخدم `selector` مع نوع إجراء غير مدعوم.
- `ACT_EVALUATE_DISABLED` (HTTP 403): تم تعطيل `evaluate` (أو `wait --fn`) عبر الإعدادات.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): يتعارض `targetId` على المستوى الأعلى أو ضمن دفعة مع هدف الطلب.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): الإجراء غير مدعوم لملفات تعريف existing-session.

قد تستمر أعطال وقت التشغيل الأخرى في إرجاع `{ "error": "<message>" }` من دون
حقل `code`.

### متطلب Playwright

تتطلب بعض الميزات Playwright. وإذا لم يكن Playwright مثبّتًا، فإن نقاط النهاية
هذه تُرجع خطأ 501 واضحًا.

ما الذي يظل يعمل من دون Playwright:

- لقطات ARIA
- لقطات شاشة الصفحة لمتصفح `openclaw` المُدار عندما يكون WebSocket
  الخاص بـ CDP لكل علامة تبويب متاحًا
- لقطات شاشة الصفحة لملفات تعريف `existing-session` / Chrome MCP
- لقطات شاشة `existing-session` المعتمدة على `--ref` من خرج snapshot

ما الذي لا يزال يحتاج إلى Playwright:

- `navigate`
- `act`
- لقطات AI / لقطات الأدوار
- لقطات عناصر محدد CSS (`--element`)
- تصدير PDF الكامل للمتصفح

كما ترفض لقطات العناصر أيضًا `--full-page`؛ إذ يعيد المسار الرسالة `fullPage is
not supported for element screenshots`.

إذا رأيت `Playwright is not available in this gateway build`، فثبّت الحزمة الكاملة
من Playwright (وليس `playwright-core`) وأعد تشغيل gateway، أو أعد تثبيت
OpenClaw مع دعم المتصفح.

#### تثبيت Playwright في Docker

إذا كان Gateway لديك يعمل داخل Docker، فتجنب `npx playwright` (تعارضات npm override).
استخدم CLI المضمّن بدلًا من ذلك:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

للاحتفاظ بتنزيلات المتصفح، اضبط `PLAYWRIGHT_BROWSERS_PATH` (على سبيل المثال،
`/home/node/.cache/ms-playwright`) وتأكد من الاحتفاظ بالدليل `/home/node` عبر
`OPENCLAW_HOME_VOLUME` أو bind mount. راجع [Docker](/ar/install/docker).

## كيف يعمل (داخليًا)

التدفق عالي المستوى:

- يقبل **خادم تحكم** صغير طلبات HTTP.
- ويتصل بمتصفحات قائمة على Chromium (Chrome/Brave/Edge/Chromium) عبر **CDP**.
- وبالنسبة إلى الإجراءات المتقدمة (النقر/الكتابة/snapshot/PDF)، فإنه يستخدم **Playwright** فوق
  CDP.
- وعند غياب Playwright، لا تتوفر إلا العمليات التي لا تعتمد على Playwright.

يحافظ هذا التصميم على بقاء الوكيل على واجهة مستقرة وحتمية، مع السماح
لك بتبديل المتصفحات وملفات التعريف المحلية/البعيدة.

## مرجع CLI السريع

تقبل جميع الأوامر `--browser-profile <name>` لاستهداف ملف تعريف محدد.
كما تقبل جميع الأوامر أيضًا `--json` للحصول على خرج قابل للقراءة آليًا (حمولات مستقرة).

الأساسيات:

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

الفحص:

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`

ملاحظة حول دورة الحياة:

- بالنسبة إلى ملفات التعريف attach-only وCDP البعيدة، يظل `openclaw browser stop`
  أمر التنظيف الصحيح بعد الاختبارات. فهو يغلق جلسة التحكم النشطة ويمسح
  تجاوزات المحاكاة المؤقتة بدلًا من إنهاء المتصفح
  الأساسي.
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

الإجراءات:

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

الحالة:

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

ملاحظات:

- الاستدعاءات `upload` و`dialog` هي استدعاءات **تجهيز مسبق**؛ شغّلها قبل النقر/الضغط
  الذي يؤدي إلى فتح محدد الملفات/مربع الحوار.
- تكون مسارات خرج التنزيلات وtrace مقيّدة بجذور الملفات المؤقتة لـ OpenClaw:
  - traces: `/tmp/openclaw` (بديل: `${os.tmpdir()}/openclaw`)
  - downloads: `/tmp/openclaw/downloads` (بديل: `${os.tmpdir()}/openclaw/downloads`)
- تكون مسارات الرفع مقيّدة بجذر رفع مؤقت لـ OpenClaw:
  - uploads: `/tmp/openclaw/uploads` (بديل: `${os.tmpdir()}/openclaw/uploads`)
- يمكن لـ `upload` أيضًا ضبط مدخلات الملفات مباشرة عبر `--input-ref` أو `--element`.
- `snapshot`:
  - `--format ai` (الافتراضي عند تثبيت Playwright): يعيد AI snapshot مع مراجع رقمية (`aria-ref="<n>"`).
  - `--format aria`: يعيد شجرة إمكانية الوصول (من دون مراجع؛ للفحص فقط).
  - `--efficient` (أو `--mode efficient`): إعداد مسبق مضغوط لـ role snapshot (تفاعلي + مضغوط + عمق + maxChars أقل).
  - الإعداد الافتراضي في config (للأداة/CLI فقط): اضبط `browser.snapshotDefaults.mode: "efficient"` لاستخدام snapshots فعّالة عندما لا يمرر المستدعي وضعًا (راجع [إعدادات Gateway](/ar/gateway/configuration-reference#browser)).
  - تفرض خيارات role snapshot (`--interactive` و`--compact` و`--depth` و`--selector`) لقطة قائمة على الأدوار مع مراجع مثل `ref=e12`.
  - يقيّد `--frame "<iframe selector>"` role snapshots داخل iframe (ويقترن مع مراجع أدوار مثل `e12`).
  - ينتج `--interactive` قائمة مسطحة وسهلة الاختيار للعناصر التفاعلية (الأفضل لتنفيذ الإجراءات).
  - يضيف `--labels` لقطة شاشة ضمن حدود viewport مع تسميات مراجع متراكبة (يطبع `MEDIA:<path>`).
- تتطلب `click` و`type` وما إلى ذلك قيمة `ref` من `snapshot` (إما رقمية `12` أو مرجع دور `e12`).
  ولا تُدعم محددات CSS عمدًا للإجراءات.

## snapshots والمراجع

يدعم OpenClaw نمطين من “snapshot”:

- **AI snapshot (مراجع رقمية)**: `openclaw browser snapshot` (الافتراضي؛ `--format ai`)
  - الخرج: snapshot نصي يتضمن مراجع رقمية.
  - الإجراءات: `openclaw browser click 12` و`openclaw browser type 23 "hello"`.
  - داخليًا، يُحل المرجع عبر `aria-ref` الخاص بـ Playwright.

- **Role snapshot (مراجع أدوار مثل `e12`)**: `openclaw browser snapshot --interactive` (أو `--compact` أو `--depth` أو `--selector` أو `--frame`)
  - الخرج: قائمة/شجرة قائمة على الأدوار مع `[ref=e12]` (واختياريًا `[nth=1]`).
  - الإجراءات: `openclaw browser click e12` و`openclaw browser highlight e12`.
  - داخليًا، يُحل المرجع عبر `getByRole(...)` (مع `nth()` أيضًا عند وجود تكرارات).
  - أضف `--labels` لتضمين لقطة شاشة ضمن viewport مع تسميات `e12` متراكبة.

سلوك المراجع:

- المراجع **ليست مستقرة عبر التنقلات**؛ وإذا فشل شيء ما، فأعد تشغيل `snapshot` واستخدم مرجعًا جديدًا.
- إذا أُخذ role snapshot باستخدام `--frame`، فستكون مراجع الأدوار مقيّدة بذلك iframe حتى role snapshot التالي.

## تحسينات `wait`

يمكنك الانتظار لأكثر من مجرد الوقت/النص:

- الانتظار لعنوان URL (مع دعم glob في Playwright):
  - `openclaw browser wait --url "**/dash"`
- الانتظار لحالة التحميل:
  - `openclaw browser wait --load networkidle`
- الانتظار لشرط JavaScript:
  - `openclaw browser wait --fn "window.ready===true"`
- الانتظار حتى يصبح محدد مرئيًا:
  - `openclaw browser wait "#main"`

يمكن دمج هذه الشروط:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## مسارات عمل التصحيح

عندما يفشل إجراء ما (مثل “not visible” أو “strict mode violation” أو “covered”):

1. `openclaw browser snapshot --interactive`
2. استخدم `click <ref>` / `type <ref>` (وفضّل مراجع الأدوار في الوضع التفاعلي)
3. إذا استمر الفشل: استخدم `openclaw browser highlight <ref>` لمعرفة ما الذي يستهدفه Playwright
4. إذا تصرفت الصفحة بشكل غريب:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. للتصحيح المتعمق: سجّل trace:
   - `openclaw browser trace start`
   - أعد إنتاج المشكلة
   - `openclaw browser trace stop` (يطبع `TRACE:<path>`)

## خرج JSON

إن `--json` مخصص للبرمجة النصية والأدوات المنظَّمة.

أمثلة:

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

تتضمن role snapshots في JSON حقول `refs` إضافةً إلى كتلة `stats` صغيرة (الأسطر/الأحرف/المراجع/التفاعلية) حتى تتمكن الأدوات من فهم حجم الحمولة وكثافتها.

## عناصر التحكم في الحالة والبيئة

تفيد هذه في مسارات عمل من نوع “اجعل الموقع يتصرف كأنه X”:

- ملفات تعريف الارتباط: `cookies` و`cookies set` و`cookies clear`
- التخزين: `storage local|session get|set|clear`
- عدم الاتصال: `set offline on|off`
- الرؤوس: `set headers --headers-json '{"X-Debug":"1"}'` (لا يزال الشكل القديم `set headers --json '{"X-Debug":"1"}'` مدعومًا)
- مصادقة HTTP الأساسية: `set credentials user pass` (أو `--clear`)
- الموقع الجغرافي: `set geo <lat> <lon> --origin "https://example.com"` (أو `--clear`)
- الوسائط: `set media dark|light|no-preference|none`
- المنطقة الزمنية / اللغة المحلية: `set timezone ...` و`set locale ...`
- الجهاز / viewport:
  - `set device "iPhone 14"` (إعدادات أجهزة Playwright المسبقة)
  - `set viewport 1280 720`

## الأمان والخصوصية

- قد يحتوي ملف تعريف متصفح openclaw على جلسات مسجّل الدخول إليها؛ لذا تعامل معه على أنه حساس.
- يقوم `browser act kind=evaluate` / `openclaw browser evaluate` و`wait --fn`
  بتنفيذ JavaScript عشوائي داخل سياق الصفحة. ويمكن لحقن المطالبات
  توجيه هذا السلوك. عطّله عبر `browser.evaluateEnabled=false` إذا لم تكن تحتاجه.
- فيما يتعلق بتسجيلات الدخول وملاحظات مكافحة الروبوتات (X/Twitter وما إلى ذلك)، راجع [تسجيل الدخول في المتصفح + النشر على X/Twitter](/ar/tools/browser-login).
- أبقِ Gateway/مضيف العقدة خاصًا (loopback أو tailnet فقط).
- نقاط نهاية CDP البعيدة قوية جدًا؛ مرّرها عبر نفق واحمها.

مثال الوضع الصارم (حظر الوجهات الخاصة/الداخلية افتراضيًا):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // سماح دقيق اختياري
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

للمشكلات الخاصة بـ Linux (وخاصة snap Chromium)، راجع
[استكشاف أخطاء المتصفح وإصلاحها](/ar/tools/browser-linux-troubleshooting).

وبالنسبة إلى إعدادات WSL2 Gateway + Windows Chrome ذات المضيف المنفصل، راجع
[استكشاف أخطاء WSL2 + Windows + remote Chrome CDP وإصلاحها](/ar/tools/browser-wsl2-windows-remote-cdp-troubleshooting).

## أدوات الوكيل + كيف يعمل التحكم

يحصل الوكيل على **أداة واحدة** لأتمتة المتصفح:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

كيفية الربط:

- يعيد `browser snapshot` شجرة واجهة مستقرة (AI أو ARIA).
- يستخدم `browser act` معرّفات `ref` من `snapshot` من أجل النقر/الكتابة/السحب/التحديد.
- يلتقط `browser screenshot` البكسلات (الصفحة كاملة أو عنصرًا).
- يقبل `browser`:
  - `profile` لاختيار ملف تعريف متصفح مسمّى (openclaw أو chrome أو remote CDP).
  - `target` (`sandbox` | `host` | `node`) لتحديد مكان وجود المتصفح.
  - في الجلسات المعزولة، يتطلب `target: "host"` القيمة `agents.defaults.sandbox.browser.allowHostControl=true`.
  - إذا لم يُمرَّر `target`: تكون الجلسات المعزولة افتراضيًا على `sandbox`، وتكون الجلسات غير المعزولة افتراضيًا على `host`.
  - إذا كانت هناك عقدة متصلة تدعم المتصفح، فقد تُوجَّه الأداة إليها تلقائيًا ما لم تثبّت `target="host"` أو `target="node"`.

يحافظ ذلك على حتمية الوكيل ويتجنب المحددات الهشّة.

## ذو صلة

- [نظرة عامة على الأدوات](/ar/tools) -- جميع أدوات الوكيل المتاحة
- [العزل](/ar/gateway/sandboxing) -- التحكم في المتصفح داخل البيئات المعزولة
- [الأمان](/ar/gateway/security) -- مخاطر التحكم في المتصفح ووسائل تقويته
