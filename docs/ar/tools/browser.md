---
read_when:
    - إضافة أتمتة متصفح يتحكم فيها الوكيل
    - تصحيح سبب تداخل openclaw مع Chrome الخاص بك
    - تنفيذ إعدادات المتصفح ودورة حياته في تطبيق macOS
summary: خدمة مدمجة للتحكم في المتصفح + أوامر الإجراءات
title: المتصفح (بإدارة OpenClaw)
x-i18n:
    generated_at: "2026-04-05T12:59:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: a41162efd397ea918469e16aa67e554bcbb517b3112df1d3e7927539b6a0926a
    source_path: tools/browser.md
    workflow: 15
---

# المتصفح (بإدارة openclaw)

يمكن لـ OpenClaw تشغيل **ملف تعريف مخصص لـ Chrome/Brave/Edge/Chromium** يتحكم فيه الوكيل.
وهو معزول عن متصفحك الشخصي ويُدار من خلال خدمة تحكم محلية صغيرة
داخل Gateway (على local loopback فقط).

عرض مبسط للمبتدئين:

- فكّر فيه على أنه **متصفح منفصل مخصص للوكيل فقط**.
- لا يلمس ملف تعريف `openclaw` **ملف تعريف متصفحك الشخصي**.
- يمكن للوكيل **فتح علامات التبويب وقراءة الصفحات والنقر والكتابة** ضمن مسار آمن.
- يتصل ملف التعريف المدمج `user` بجلسة Chrome الحقيقية المسجّل دخولك فيها عبر Chrome MCP.

## ما الذي ستحصل عليه

- ملف تعريف متصفح منفصل باسم **openclaw** (بتمييز برتقالي افتراضيًا).
- تحكم حتمي في علامات التبويب (عرض/فتح/تركيز/إغلاق).
- إجراءات الوكيل (نقر/كتابة/سحب/تحديد)، ولقطات snapshot، ولقطات شاشة، وPDF.
- دعم اختياري لعدة ملفات تعريف (`openclaw` و`work` و`remote` و...).

هذا المتصفح **ليس** متصفحك اليومي. بل هو سطح آمن ومعزول
لأتمتة الوكيل والتحقق.

## البدء السريع

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

إذا ظهرت لك رسالة “Browser disabled”، فقم بتمكينه في التهيئة (انظر أدناه) وأعد تشغيل
Gateway.

إذا كان `openclaw browser` مفقودًا بالكامل، أو قال الوكيل إن أداة المتصفح
غير متاحة، فانتقل إلى [أمر المتصفح أو الأداة مفقود](/tools/browser#missing-browser-command-or-tool).

## التحكم عبر plugin

أصبحت أداة `browser` الافتراضية الآن plugin مجمّعًا يُشحن ممكّنًا
افتراضيًا. وهذا يعني أنه يمكنك تعطيله أو استبداله من دون إزالة بقية
نظام plugin في OpenClaw:

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

عطّل plugin المجمّع قبل تثبيت plugin آخر يوفّر
اسم أداة `browser` نفسه. تتطلب تجربة المتصفح الافتراضية كليهما:

- ألا يكون `plugins.entries.browser.enabled` معطّلًا
- وأن تكون `browser.enabled=true`

إذا عطّلت plugin فقط، فسيختفي معًا كل من CLI المتصفح المجمّع (`openclaw browser`)،
وطريقة gateway (`browser.request`)، وأداة الوكيل، وخدمة التحكم الافتراضية بالمتصفح.
وستبقى تهيئة `browser.*` كما هي ليُعاد استخدامها من قِبل plugin
بديل.

يمتلك plugin المتصفح المجمّع الآن أيضًا تنفيذ runtime الخاص بالمتصفح.
ويحتفظ core فقط بمساعدات Plugin SDK المشتركة بالإضافة إلى إعادة تصدير
للتوافق مع مسارات الاستيراد الداخلية الأقدم. عمليًا، تؤدي إزالة أو استبدال
حزمة plugin المتصفح إلى إزالة مجموعة ميزات المتصفح بدلًا من ترك runtime
ثانٍ مملوكًا لـ core في الخلف.

لا تزال تغييرات تهيئة المتصفح تتطلب إعادة تشغيل Gateway حتى يتمكن plugin المجمّع
من إعادة تسجيل خدمة المتصفح بالإعدادات الجديدة.

## أمر المتصفح أو الأداة مفقود

إذا أصبح `openclaw browser` فجأة أمرًا غير معروف بعد الترقية، أو
أبلغ الوكيل أن أداة المتصفح مفقودة، فالسبب الأكثر شيوعًا هو وجود قائمة
`plugins.allow` مقيّدة لا تتضمن `browser`.

مثال على تهيئة معطلة:

```json5
{
  plugins: {
    allow: ["telegram"],
  },
}
```

أصلح ذلك بإضافة `browser` إلى قائمة السماح الخاصة بـ plugin:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

ملاحظات مهمة:

- لا تكفي `browser.enabled=true` وحدها عند تعيين `plugins.allow`.
- لا تكفي `plugins.entries.browser.enabled=true` وحدها أيضًا عند تعيين `plugins.allow`.
- لا تقوم `tools.alsoAllow: ["browser"]` بتحميل plugin المتصفح المجمّع. بل تضبط سياسة الأدوات فقط بعد تحميل plugin بالفعل.
- إذا لم تكن بحاجة إلى قائمة سماح مقيّدة لـ plugin، فإن إزالة `plugins.allow` تعيد أيضًا سلوك المتصفح المجمّع الافتراضي.

الأعراض المعتادة:

- `openclaw browser` أمر غير معروف.
- `browser.request` مفقود.
- يبلّغ الوكيل أن أداة المتصفح غير متاحة أو مفقودة.

## ملفات التعريف: `openclaw` مقابل `user`

- `openclaw`: متصفح مُدار ومعزول (لا يحتاج إلى extension).
- `user`: ملف تعريف مدمج للاتصال بجلسة Chrome MCP الخاصة **بجلسة Chrome الحقيقية المسجّل دخولك فيها**.

بالنسبة إلى استدعاءات أداة متصفح الوكيل:

- الافتراضي: استخدام متصفح `openclaw` المعزول.
- يُفضَّل `profile="user"` عندما تكون جلسات تسجيل الدخول الحالية مهمة ويكون المستخدم
  أمام الكمبيوتر للنقر/الموافقة على أي مطالبة اتصال.
- `profile` هو التجاوز الصريح عندما تريد وضع متصفح محددًا.

عيّن `browser.defaultProfile: "openclaw"` إذا أردت الوضع المُدار افتراضيًا.

## التهيئة

توجد إعدادات المتصفح في `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // default: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy single-profile override
    remoteCdpTimeoutMs: 1500, // remote CDP HTTP timeout (ms)
    remoteCdpHandshakeTimeoutMs: 3000, // remote CDP WebSocket handshake timeout (ms)
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

- ترتبط خدمة التحكم بالمتصفح بعنوان local loopback على منفذ مشتق من `gateway.port`
  (الافتراضي: `18791`، أي gateway + 2).
- إذا تجاوزت منفذ Gateway (`gateway.port` أو `OPENCLAW_GATEWAY_PORT`)،
  فإن منافذ المتصفح المشتقة تتغير لتبقى ضمن "العائلة" نفسها.
- تكون القيمة الافتراضية لـ `cdpUrl` هي منفذ CDP المحلي المُدار عندما لا يتم تعيينها.
- ينطبق `remoteCdpTimeoutMs` على فحوصات الوصول إلى CDP البعيد (غير local loopback).
- ينطبق `remoteCdpHandshakeTimeoutMs` على فحوصات الوصول إلى WebSocket الخاص بـ CDP البعيد.
- تكون عملية التنقل/فتح علامة التبويب في المتصفح محمية من SSRF قبل التنقل ويُعاد التحقق منها قدر الإمكان على عنوان `http(s)` النهائي بعد التنقل.
- في وضع SSRF الصارم، يتم أيضًا التحقق من اكتشاف/فحص نقاط نهاية CDP البعيدة (`cdpUrl`، بما في ذلك عمليات البحث في `/json/version`).
- تكون القيمة الافتراضية لـ `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` هي `true` (نموذج الشبكة الموثوقة). عيّنها إلى `false` للتصفح العام الصارم فقط.
- لا يزال `browser.ssrfPolicy.allowPrivateNetwork` مدعومًا كاسم بديل قديم للتوافق.
- تعني `attachOnly: true` "عدم تشغيل متصفح محلي مطلقًا؛ والاتصال فقط إذا كان يعمل بالفعل."
- يلوّن `color` و`color` لكل ملف تعريف واجهة المتصفح لتتمكن من معرفة أي ملف تعريف نشط.
- ملف التعريف الافتراضي هو `openclaw` (متصفح مستقل مُدار من OpenClaw). استخدم `defaultProfile: "user"` للاشتراك في متصفح المستخدم المسجّل دخوله.
- ترتيب الاكتشاف التلقائي: متصفح النظام الافتراضي إذا كان قائمًا على Chromium؛ وإلا Chrome ← Brave ← Edge ← Chromium ← Chrome Canary.
- تقوم ملفات تعريف `openclaw` المحلية بتعيين `cdpPort`/`cdpUrl` تلقائيًا — عيّن هذه القيم فقط لـ CDP البعيد.
- يستخدم `driver: "existing-session"` Chrome DevTools MCP بدلًا من CDP الخام. لا
  تعيّن `cdpUrl` لهذا المشغّل.
- عيّن `browser.profiles.<name>.userDataDir` عندما يجب على ملف تعريف existing-session
  الاتصال بملف تعريف مستخدم Chromium غير افتراضي مثل Brave أو Edge.

## استخدام Brave (أو متصفح آخر قائم على Chromium)

إذا كان متصفحك **الافتراضي للنظام** قائمًا على Chromium (Chrome/Brave/Edge/إلخ)،
فإن OpenClaw يستخدمه تلقائيًا. عيّن `browser.executablePath` لتجاوز
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

## التحكم المحلي مقابل البعيد

- **التحكم المحلي (الافتراضي):** يبدأ Gateway خدمة التحكم على local loopback ويمكنه تشغيل متصفح محلي.
- **التحكم البعيد (مضيف node):** شغّل مضيف node على الجهاز الذي يحتوي على المتصفح؛ وسيقوم Gateway بتمرير إجراءات المتصفح إليه.
- **CDP البعيد:** عيّن `browser.profiles.<name>.cdpUrl` (أو `browser.cdpUrl`) من أجل
  الاتصال بمتصفح بعيد قائم على Chromium. في هذه الحالة، لن يشغّل OpenClaw متصفحًا محليًا.

يختلف سلوك الإيقاف حسب وضع ملف التعريف:

- ملفات التعريف المحلية المُدارة: يقوم `openclaw browser stop` بإيقاف عملية المتصفح التي
  شغّلها OpenClaw
- ملفات attach-only وCDP البعيدة: يقوم `openclaw browser stop` بإغلاق جلسة
  التحكم النشطة وتحرير تجاوزات محاكاة Playwright/CDP (منفذ العرض،
  ونظام الألوان، واللغة، والمنطقة الزمنية، ووضع عدم الاتصال، والحالة المشابهة)، حتى
  لو لم يُشغّل OpenClaw أي عملية متصفح

يمكن أن تتضمن عناوين URL الخاصة بـ CDP البعيد مصادقة:

- رموز استعلام (مثل `https://provider.example?token=<token>`)
- مصادقة HTTP Basic (مثل `https://user:pass@provider.example`)

يحافظ OpenClaw على المصادقة عند استدعاء نقاط نهاية `/json/*` وعند الاتصال
بـ WebSocket الخاص بـ CDP. فضّل استخدام متغيرات البيئة أو مديري الأسرار للرموز
بدلًا من تسجيلها في ملفات التهيئة.

## وكيل متصفح node (افتراضي بلا تهيئة)

إذا شغّلت **مضيف node** على الجهاز الذي يحتوي على متصفحك، فيمكن لـ OpenClaw
توجيه استدعاءات أداة المتصفح تلقائيًا إلى ذلك الـ node من دون أي تهيئة إضافية للمتصفح.
وهذا هو المسار الافتراضي لـ gatewayات البعيدة.

ملاحظات:

- يكشف مضيف node عن خادم التحكم المحلي في المتصفح عبر **أمر وكيل**.
- تأتي ملفات التعريف من تهيئة `browser.profiles` الخاصة بالـ node نفسه (كما في الوضع المحلي).
- `nodeHost.browserProxy.allowProfiles` اختياري. اتركه فارغًا للحصول على السلوك القديم/الافتراضي: تظل جميع ملفات التعريف المهيأة قابلة للوصول عبر الوكيل، بما في ذلك مسارات إنشاء/حذف ملفات التعريف.
- إذا عيّنت `nodeHost.browserProxy.allowProfiles`، فإن OpenClaw يتعامل معه كحد أقل الامتيازات: لا يمكن استهداف إلا ملفات التعريف الموجودة في قائمة السماح، وتُحظر مسارات إنشاء/حذف ملفات التعريف الدائمة على سطح الوكيل.
- عطّله إذا كنت لا تريده:
  - على الـ node: `nodeHost.browserProxy.enabled=false`
  - على الـ gateway: ‏`gateway.nodes.browser.mode="off"`

## Browserless (CDP بعيد مستضاف)

[Browserless](https://browserless.io) خدمة Chromium مستضافة تكشف
عناوين اتصال CDP عبر HTTPS وWebSocket. يمكن لـ OpenClaw استخدام أيٍّ من الشكلين، ولكن
بالنسبة إلى ملف تعريف متصفح بعيد، فإن أبسط خيار هو عنوان WebSocket المباشر
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
- اختر نقطة نهاية المنطقة التي تطابق حساب Browserless الخاص بك (راجع وثائقهم).
- إذا منحك Browserless عنوان HTTPS أساسيًا، فيمكنك إما تحويله إلى
  `wss://` لاتصال CDP مباشر أو الاحتفاظ بعنوان HTTPS وترك OpenClaw
  يكتشف `/json/version`.

## موفرو WebSocket CDP المباشر

تكشف بعض خدمات المتصفح المستضافة نقطة نهاية **WebSocket مباشرة** بدلًا من
آلية اكتشاف CDP القياسية المعتمدة على HTTP (`/json/version`). يدعم OpenClaw كلاهما:

- **نقاط النهاية HTTP(S)** — يستدعي OpenClaw ‏`/json/version` لاكتشاف
  عنوان WebSocket debugger، ثم يتصل به.
- **نقاط النهاية WebSocket** (`ws://` / `wss://`) — يتصل OpenClaw مباشرة،
  متجاوزًا `/json/version`. استخدم هذا لخدمات مثل
  [Browserless](https://browserless.io)،
  و[Browserbase](https://www.browserbase.com)، أو أي موفر يزوّدك بعنوان
  WebSocket.

### Browserbase

[Browserbase](https://www.browserbase.com) منصة سحابية لتشغيل
المتصفحات headless مع حل CAPTCHA مدمج، ووضع التخفي، ووكلاء سكنيين.

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

- [أنشئ حسابًا](https://www.browserbase.com/sign-up) وانسخ **API Key**
  من [لوحة معلومات Overview](https://www.browserbase.com/overview).
- استبدل `<BROWSERBASE_API_KEY>` بمفتاح API الحقيقي الخاص بـ Browserbase.
- ينشئ Browserbase جلسة متصفح تلقائيًا عند الاتصال عبر WebSocket، لذا لا
  حاجة إلى خطوة إنشاء جلسة يدويًا.
- تتيح الخطة المجانية جلسة متزامنة واحدة وساعة متصفح واحدة شهريًا.
  راجع [الأسعار](https://www.browserbase.com/pricing) لمعرفة حدود الخطط المدفوعة.
- راجع [وثائق Browserbase](https://docs.browserbase.com) للحصول على المرجع الكامل لـ API،
  وأدلة SDK، وأمثلة التكامل.

## الأمان

الأفكار الأساسية:

- التحكم بالمتصفح متاح على local loopback فقط؛ ويعبر الوصول من خلال مصادقة Gateway أو إقران node.
- يستخدم HTTP API المستقل للمتصفح على local loopback **مصادقة السر المشترك فقط**:
  مصادقة bearer لرمز gateway، أو `x-openclaw-password`، أو HTTP Basic auth مع
  كلمة مرور gateway المهيأة.
- **لا** تقوم رؤوس هوية Tailscale Serve و`gateway.auth.mode: "trusted-proxy"`
  بمصادقة HTTP API المستقل هذا للمتصفح على local loopback.
- إذا كان التحكم بالمتصفح ممكّنًا ولم تُهيأ أي مصادقة بالسر المشترك، فسيقوم OpenClaw
  بإنشاء `gateway.auth.token` تلقائيًا عند بدء التشغيل وحفظه في التهيئة.
- **لا** ينشئ OpenClaw هذا الرمز تلقائيًا عندما يكون `gateway.auth.mode`
  مضبوطًا بالفعل على `password` أو `none` أو `trusted-proxy`.
- أبقِ Gateway وأي مضيفات node على شبكة خاصة (Tailscale)؛ وتجنب تعريضها للعامة.
- تعامل مع عناوين/رموز CDP البعيدة باعتبارها أسرارًا؛ وفضّل متغيرات البيئة أو مدير أسرار.

نصائح CDP البعيد:

- فضّل نقاط النهاية المشفرة (HTTPS أو WSS) والرموز قصيرة العمر حيثما أمكن.
- تجنّب تضمين الرموز طويلة العمر مباشرة في ملفات التهيئة.

## ملفات التعريف (متصفحات متعددة)

يدعم OpenClaw عدة ملفات تعريف مسماة (تهيئات توجيه). يمكن أن تكون ملفات التعريف:

- **بإدارة OpenClaw**: مثيل متصفح مخصص قائم على Chromium مع دليل بيانات مستخدم ومنفذ CDP خاصين به
- **بعيد**: عنوان CDP صريح (متصفح قائم على Chromium يعمل في مكان آخر)
- **جلسة موجودة**: ملف تعريف Chrome الحالي لديك عبر الاتصال التلقائي بـ Chrome DevTools MCP

الإعدادات الافتراضية:

- يُنشأ ملف التعريف `openclaw` تلقائيًا إذا كان مفقودًا.
- ملف التعريف `user` مدمج للاتصال بـ Chrome MCP existing-session.
- تكون ملفات تعريف existing-session اختيارية فيما عدا `user`؛ أنشئها باستخدام `--driver existing-session`.
- تخصص منافذ CDP المحلية من **18800–18899** افتراضيًا.
- يؤدي حذف ملف تعريف إلى نقل دليل بياناته المحلي إلى سلة المهملات.

تقبل جميع نقاط نهاية التحكم `?profile=<name>`؛ ويستخدم CLI ‏`--browser-profile`.

## existing-session عبر Chrome DevTools MCP

يمكن لـ OpenClaw أيضًا الاتصال بملف تعريف متصفح قائم على Chromium يعمل بالفعل من خلال
خادم Chrome DevTools MCP الرسمي. وهذا يعيد استخدام علامات التبويب وحالة تسجيل الدخول
المفتوحة أصلًا في ملف تعريف المتصفح هذا.

مراجع الخلفية والإعداد الرسمية:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

ملف التعريف المدمج:

- `user`

اختياري: أنشئ ملف تعريف existing-session مخصصًا خاصًا بك إذا أردت
اسمًا أو لونًا أو دليل بيانات متصفح مختلفًا.

السلوك الافتراضي:

- يستخدم ملف التعريف المدمج `user` الاتصال التلقائي عبر Chrome MCP، والذي يستهدف
  ملف تعريف Google Chrome المحلي الافتراضي.

استخدم `userDataDir` مع Brave أو Edge أو Chromium أو ملف تعريف Chrome غير افتراضي:

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

1. افتح صفحة inspect الخاصة بذلك المتصفح للتصحيح عن بُعد.
2. فعّل التصحيح عن بُعد.
3. أبقِ المتصفح قيد التشغيل ووافق على مطالبة الاتصال عندما يتصل OpenClaw.

صفحات inspect الشائعة:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

اختبار دخان الاتصال المباشر:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

كيف يبدو النجاح:

- يعرض `status` القيمة `driver: existing-session`
- يعرض `status` القيمة `transport: chrome-mcp`
- يعرض `status` القيمة `running: true`
- يسرد `tabs` علامات التبويب المفتوحة بالفعل في متصفحك
- يعيد `snapshot` مراجع refs من علامة التبويب الحية المحددة

ما الذي يجب التحقق منه إذا لم ينجح الاتصال:

- أن يكون إصدار المتصفح المستهدف القائم على Chromium هو `144+`
- أن يكون التصحيح عن بُعد ممكّنًا في صفحة inspect لذلك المتصفح
- أن يكون المتصفح قد أظهر مطالبة موافقة الاتصال وأنك قبلتها
- ينقل `openclaw doctor` تهيئة المتصفح القديمة المعتمدة على extension ويتحقق من
  تثبيت Chrome محليًا لملفات تعريف الاتصال التلقائي الافتراضية، لكنه لا يستطيع
  تمكين التصحيح عن بُعد من جهة المتصفح نيابةً عنك

استخدام الوكيل:

- استخدم `profile="user"` عندما تحتاج إلى حالة المتصفح المسجّل دخول المستخدم فيها.
- إذا كنت تستخدم ملف تعريف existing-session مخصصًا، فمرر اسم ملف التعريف هذا صراحةً.
- اختر هذا الوضع فقط عندما يكون المستخدم أمام الكمبيوتر للموافقة على مطالبة
  الاتصال.
- يمكن لـ Gateway أو مضيف node تشغيل `npx chrome-devtools-mcp@latest --autoConnect`

ملاحظات:

- هذا المسار أعلى خطورة من ملف التعريف المعزول `openclaw` لأنه يمكنه
  العمل داخل جلسة المتصفح المسجّل دخولك فيها.
- لا يشغّل OpenClaw المتصفح لهذا المشغّل؛ بل يتصل
  بجلسة موجودة فقط.
- يستخدم OpenClaw هنا تدفق `--autoConnect` الرسمي لـ Chrome DevTools MCP. إذا
  تم تعيين `userDataDir`، فسيقوم OpenClaw بتمريره لاستهداف دليل
  بيانات مستخدم Chromium الصريح هذا.
- تدعم لقطات شاشة existing-session التقاط الصفحات والتقاط العناصر باستخدام `--ref`
  من مخرجات snapshot، ولكنها لا تدعم محددات CSS ‏`--element`.
- تعمل لقطات شاشة الصفحات في existing-session من دون Playwright عبر Chrome MCP.
  كما تعمل هناك أيضًا لقطات شاشة العناصر المستندة إلى ref (`--ref`)، لكن لا يمكن
  دمج `--full-page` مع `--ref` أو `--element`.
- لا تزال إجراءات existing-session أكثر محدودية من مسار المتصفح
  المُدار:
  - تتطلب `click` و`type` و`hover` و`scrollIntoView` و`drag` و`select`
    مراجع snapshot بدلًا من محددات CSS
  - تدعم `click` الزر الأيسر فقط (من دون تجاوزات للأزرار أو modifiers)
  - لا تدعم `type` الخيار `slowly=true`؛ استخدم `fill` أو `press`
  - لا تدعم `press` الخيار `delayMs`
  - لا تدعم `hover` و`scrollIntoView` و`drag` و`select` و`fill` و`evaluate`
    تجاوزات المهلة لكل استدعاء
  - تدعم `select` حاليًا قيمة واحدة فقط
- يدعم `wait --url` في existing-session الأنماط المطابقة التامة، والجزئية، وglob
  مثل بقية مشغّلات المتصفح. لا يزال `wait --load networkidle` غير مدعوم.
- تتطلب خطافات الرفع في existing-session استخدام `ref` أو `inputRef`، وتدعم ملفًا واحدًا في كل مرة، ولا تدعم الاستهداف عبر CSS ‏`element`.
- لا تدعم خطافات مربعات الحوار في existing-session تجاوزات المهلة.
- لا تزال بعض الميزات تتطلب مسار المتصفح المُدار، بما في ذلك إجراءات
  الدفعات، وتصدير PDF، واعتراض التنزيلات، و`responsebody`.
- يكون existing-session محليًا على المضيف. إذا كان Chrome موجودًا على جهاز مختلف أو
  ضمن مساحة اسم شبكة مختلفة، فاستخدم CDP البعيد أو مضيف node بدلًا من ذلك.

## ضمانات العزل

- **دليل بيانات مستخدم مخصص**: لا يلمس ملف تعريف متصفحك الشخصي مطلقًا.
- **منافذ مخصصة**: يتجنب `9222` لمنع التعارض مع مسارات عمل التطوير.
- **تحكم حتمي في علامات التبويب**: استهدف علامات التبويب عبر `targetId`، وليس "آخر علامة تبويب".

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
- Linux: يبحث عن `google-chrome` و`brave` و`microsoft-edge` و`chromium` وغيرها.
- Windows: يتحقق من مواقع التثبيت الشائعة.

## Control API ‏(اختياري)

بالنسبة إلى عمليات التكامل المحلية فقط، يكشف Gateway عن HTTP API صغير على local loopback:

- الحالة/البدء/الإيقاف: `GET /`، `POST /start`، `POST /stop`
- علامات التبويب: `GET /tabs`، `POST /tabs/open`، `POST /tabs/focus`، `DELETE /tabs/:targetId`
- snapshot/لقطة الشاشة: `GET /snapshot`، `POST /screenshot`
- الإجراءات: `POST /navigate`، `POST /act`
- الخطافات: `POST /hooks/file-chooser`، `POST /hooks/dialog`
- التنزيلات: `POST /download`، `POST /wait/download`
- التصحيح: `GET /console`، `POST /pdf`
- التصحيح: `GET /errors`، `GET /requests`، `POST /trace/start`، `POST /trace/stop`، `POST /highlight`
- الشبكة: `POST /response/body`
- الحالة: `GET /cookies`، `POST /cookies/set`، `POST /cookies/clear`
- الحالة: `GET /storage/:kind`، `POST /storage/:kind/set`، `POST /storage/:kind/clear`
- الإعدادات: `POST /set/offline`، `POST /set/headers`، `POST /set/credentials`، `POST /set/geolocation`، `POST /set/media`، `POST /set/timezone`، `POST /set/locale`، `POST /set/device`

تقبل جميع نقاط النهاية `?profile=<name>`.

إذا كانت مصادقة gateway بالسر المشترك مهيأة، فإن مسارات HTTP الخاصة بالمتصفح تتطلب مصادقة أيضًا:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` أو HTTP Basic auth باستخدام كلمة المرور نفسها

ملاحظات:

- **لا** يستهلك HTTP API المستقل للمتصفح على local loopback أوضاع trusted-proxy أو
  رؤوس هوية Tailscale Serve.
- إذا كانت قيمة `gateway.auth.mode` هي `none` أو `trusted-proxy`، فإن مسارات المتصفح
  هذه على local loopback لا ترث أوضاع الهوية الحاملة هذه؛ لذا أبقها على local loopback فقط.

### متطلب Playwright

تتطلب بعض الميزات (navigate/act/AI snapshot/role snapshot،
ولقطات شاشة العناصر، وPDF) وجود Playwright. وإذا لم يكن Playwright مثبتًا، فستعيد نقاط النهاية هذه
خطأ 501 واضحًا.

ما الذي لا يزال يعمل من دون Playwright:

- لقطات ARIA
- لقطات شاشة الصفحات لمتصفح `openclaw` المُدار عندما يكون WebSocket
  CDP لكل علامة تبويب متاحًا
- لقطات شاشة الصفحات لملفات تعريف `existing-session` / Chrome MCP
- لقطات شاشة existing-session المعتمدة على ref (`--ref`) من مخرجات snapshot

ما الذي لا يزال يحتاج إلى Playwright:

- `navigate`
- `act`
- AI snapshots / role snapshots
- لقطات شاشة العناصر بمحددات CSS ‏(`--element`)
- تصدير PDF الكامل للمتصفح

ترفض لقطات شاشة العناصر أيضًا الخيار `--full-page`؛ حيث تعيد المسارات الرسالة `fullPage is
not supported for element screenshots`.

إذا ظهرت لك الرسالة `Playwright is not available in this gateway build`، فثبّت
حزمة Playwright الكاملة (وليس `playwright-core`) وأعد تشغيل gateway، أو أعد تثبيت
OpenClaw مع دعم المتصفح.

#### تثبيت Playwright في Docker

إذا كان Gateway يعمل داخل Docker، فتجنب `npx playwright` (تعارضات npm override).
استخدم CLI المجمّع بدلًا من ذلك:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

للاحتفاظ بتنزيلات المتصفح، عيّن `PLAYWRIGHT_BROWSERS_PATH` (على سبيل المثال،
`/home/node/.cache/ms-playwright`) وتأكد من الاحتفاظ بـ `/home/node` عبر
`OPENCLAW_HOME_VOLUME` أو bind mount. راجع [Docker](/ar/install/docker).

## كيف يعمل (داخليًا)

التدفق عالي المستوى:

- يقبل **خادم تحكم** صغير طلبات HTTP.
- ويتصل بالمتصفحات القائمة على Chromium (Chrome/Brave/Edge/Chromium) عبر **CDP**.
- وللإجراءات المتقدمة (النقر/الكتابة/snapshot/PDF)، فإنه يستخدم **Playwright** فوق
  CDP.
- وعند غياب Playwright، لا تتوفر إلا العمليات التي لا تعتمد على Playwright.

يحافظ هذا التصميم على واجهة مستقرة وحتمية للوكيل مع السماح
لك بتبديل المتصفحات وملفات التعريف المحلية/البعيدة.

## مرجع CLI سريع

تقبل جميع الأوامر `--browser-profile <name>` لاستهداف ملف تعريف محدد.
كما تقبل جميع الأوامر `--json` للحصول على مخرجات قابلة للقراءة آليًا (حمولات مستقرة).

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

ملاحظة عن دورة الحياة:

- بالنسبة إلى ملفات التعريف attach-only وCDP البعيدة، يظل `openclaw browser stop`
  هو أمر التنظيف الصحيح بعد الاختبارات. فهو يغلق جلسة التحكم النشطة ويمسح
  تجاوزات المحاكاة المؤقتة بدلًا من قتل
  المتصفح الأساسي.
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

- إن `upload` و`dialog` هما استدعاءا **تجهيز مسبق**؛ شغّلهما قبل النقر/الضغط
  الذي يطلق أداة اختيار الملف/مربع الحوار.
- تُقيّد مسارات إخراج التنزيل والتتبع بجذور مجلدات OpenClaw المؤقتة:
  - التتبعات: `/tmp/openclaw` (البديل: `${os.tmpdir()}/openclaw`)
  - التنزيلات: `/tmp/openclaw/downloads` (البديل: `${os.tmpdir()}/openclaw/downloads`)
- تُقيّد مسارات الرفع بجذر OpenClaw المؤقت الخاص بالرفع:
  - عمليات الرفع: `/tmp/openclaw/uploads` (البديل: `${os.tmpdir()}/openclaw/uploads`)
- يمكن لـ `upload` أيضًا تعيين مدخلات الملفات مباشرة عبر `--input-ref` أو `--element`.
- `snapshot`:
  - `--format ai` (الافتراضي عند تثبيت Playwright): يعيد AI snapshot مع مراجع رقمية (`aria-ref="<n>"`).
  - `--format aria`: يعيد شجرة إمكانية الوصول (من دون مراجع؛ للفحص فقط).
  - `--efficient` (أو `--mode efficient`): إعداد مسبق مضغوط لـ role snapshot ‏(interactive + compact + depth + maxChars أقل).
  - الإعداد الافتراضي في التهيئة (للأداة/CLI فقط): عيّن `browser.snapshotDefaults.mode: "efficient"` لاستخدام snapshots الفعّالة عندما لا يمرر المستدعي وضعًا (راجع [تهيئة Gateway](/ar/gateway/configuration-reference#browser)).
  - تفرض خيارات role snapshot ‏(`--interactive` و`--compact` و`--depth` و`--selector`) لقطة قائمة على الأدوار مع مراجع مثل `ref=e12`.
  - يقيّد `--frame "<iframe selector>"` role snapshots إلى iframe محدد (ويقترن بمراجع أدوار مثل `e12`).
  - ينتج `--interactive` قائمة مسطحة وسهلة الاختيار للعناصر التفاعلية (الأفضل لتوجيه الإجراءات).
  - يضيف `--labels` لقطة شاشة لمنفذ العرض فقط مع تراكب تسميات المراجع (ويطبع `MEDIA:<path>`).
- تتطلب `click`/`type`/إلخ `ref` من `snapshot` (إما رقميًا `12` أو مرجع دور `e12`).
  لا تُدعم محددات CSS عمدًا للإجراءات.

## snapshots والمراجع

يدعم OpenClaw نمطين من "snapshot":

- **AI snapshot (مراجع رقمية)**: ‏`openclaw browser snapshot` (الافتراضي؛ `--format ai`)
  - الإخراج: لقطة نصية تتضمن مراجع رقمية.
  - الإجراءات: `openclaw browser click 12`، ‏`openclaw browser type 23 "hello"`.
  - داخليًا، يتم حل المرجع عبر `aria-ref` الخاص بـ Playwright.

- **Role snapshot (مراجع أدوار مثل `e12`)**: ‏`openclaw browser snapshot --interactive` (أو `--compact` أو `--depth` أو `--selector` أو `--frame`)
  - الإخراج: قائمة/شجرة قائمة على الأدوار تتضمن `[ref=e12]` (ومع `[nth=1]` اختياريًا).
  - الإجراءات: `openclaw browser click e12`، ‏`openclaw browser highlight e12`.
  - داخليًا، يُحل المرجع عبر `getByRole(...)` ‏(مع `nth()` للتكرارات).
  - أضف `--labels` لتضمين لقطة شاشة لمنفذ العرض مع تراكب تسميات `e12`.

سلوك المراجع:

- **لا** تكون المراجع مستقرة عبر عمليات التنقل؛ إذا فشل شيء ما، فأعد تشغيل `snapshot` واستخدم مرجعًا جديدًا.
- إذا أُخذ role snapshot باستخدام `--frame`، فستُقيّد مراجع الأدوار إلى ذلك iframe حتى role snapshot التالي.

## تحسينات الانتظار

يمكنك الانتظار لأكثر من مجرد الوقت/النص:

- الانتظار لعنوان URL ‏(مع دعم glob من Playwright):
  - `openclaw browser wait --url "**/dash"`
- الانتظار لحالة التحميل:
  - `openclaw browser wait --load networkidle`
- الانتظار لمُسند JS:
  - `openclaw browser wait --fn "window.ready===true"`
- الانتظار حتى يصبح محدد مرئيًا:
  - `openclaw browser wait "#main"`

يمكن الجمع بينها:

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
2. استخدم `click <ref>` / `type <ref>` (فضّل مراجع الأدوار في الوضع interactive)
3. إذا استمر الفشل: استخدم `openclaw browser highlight <ref>` لمعرفة ما الذي يستهدفه Playwright
4. إذا كان سلوك الصفحة غريبًا:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. للتصحيح العميق: سجّل تتبعًا:
   - `openclaw browser trace start`
   - أعد إنتاج المشكلة
   - `openclaw browser trace stop` (يطبع `TRACE:<path>`)

## إخراج JSON

يُستخدم `--json` في السكربتات والأدوات المنظمة.

أمثلة:

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

تتضمن role snapshots في JSON حقول `refs` بالإضافة إلى كتلة `stats` صغيرة (lines/chars/refs/interactive) لكي تتمكن الأدوات من الاستدلال على حجم الحمولة وكثافتها.

## مفاتيح الحالة والبيئة

هذه مفيدة لمسارات العمل من نوع "اجعل الموقع يتصرف كما لو كان X":

- ملفات تعريف الارتباط: `cookies` و`cookies set` و`cookies clear`
- التخزين: `storage local|session get|set|clear`
- عدم الاتصال: `set offline on|off`
- الرؤوس: `set headers --headers-json '{"X-Debug":"1"}'` (لا يزال `set headers --json '{"X-Debug":"1"}'` القديم مدعومًا)
- مصادقة HTTP basic: ‏`set credentials user pass` (أو `--clear`)
- الموقع الجغرافي: `set geo <lat> <lon> --origin "https://example.com"` (أو `--clear`)
- الوسائط: `set media dark|light|no-preference|none`
- المنطقة الزمنية / اللغة المحلية: `set timezone ...`، ‏`set locale ...`
- الجهاز / منفذ العرض:
  - `set device "iPhone 14"` (إعدادات أجهزة Playwright المسبقة)
  - `set viewport 1280 720`

## الأمان والخصوصية

- قد يحتوي ملف تعريف متصفح openclaw على جلسات مسجّل دخولها؛ لذا تعامل معه على أنه حساس.
- ينفذ `browser act kind=evaluate` / `openclaw browser evaluate` و`wait --fn`
  JavaScript عشوائيًا في سياق الصفحة. ويمكن لحقن prompt توجيه
  هذا السلوك. عطّله باستخدام `browser.evaluateEnabled=false` إذا لم تكن بحاجة إليه.
- لملاحظات تسجيل الدخول ومكافحة الروبوتات (X/Twitter، إلخ)، راجع [تسجيل الدخول إلى المتصفح + النشر على X/Twitter](/tools/browser-login).
- أبقِ Gateway/مضيف node خاصًا (على local loopback أو داخل tailnet فقط).
- نقاط نهاية CDP البعيدة قوية التأثير؛ قم بتمريرها عبر نفق واحمها.

مثال للوضع الصارم (حظر الوجهات الخاصة/الداخلية افتراضيًا):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optional exact allow
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

للمشكلات الخاصة بـ Linux (خصوصًا snap Chromium)، راجع
[استكشاف أخطاء المتصفح وإصلاحها](/tools/browser-linux-troubleshooting).

لإعدادات WSL2 Gateway + Windows Chrome ذات المضيف المنفصل، راجع
[استكشاف أخطاء WSL2 + Windows + remote Chrome CDP وإصلاحها](/tools/browser-wsl2-windows-remote-cdp-troubleshooting).

## أدوات الوكيل + كيفية عمل التحكم

يحصل الوكيل على **أداة واحدة** لأتمتة المتصفح:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

كيفية الربط:

- يعيد `browser snapshot` شجرة UI مستقرة (AI أو ARIA).
- يستخدم `browser act` معرّفات `ref` من snapshot من أجل النقر/الكتابة/السحب/التحديد.
- يلتقط `browser screenshot` البكسلات (صفحة كاملة أو عنصر).
- يقبل `browser`:
  - `profile` لاختيار ملف تعريف متصفح مسمى (openclaw أو chrome أو remote CDP).
  - `target` ‏(`sandbox` | `host` | `node`) لتحديد مكان وجود المتصفح.
  - في الجلسات المعزولة، يتطلب `target: "host"` أن تكون `agents.defaults.sandbox.browser.allowHostControl=true`.
  - إذا لم يتم تحديد `target`: تستخدم الجلسات المعزولة القيمة الافتراضية `sandbox`، وتستخدم الجلسات غير المعزولة القيمة الافتراضية `host`.
  - إذا كان node قادر على تشغيل المتصفح متصلًا، فقد تُوجَّه الأداة إليه تلقائيًا ما لم تثبّت `target="host"` أو `target="node"`.

وهذا يحافظ على سلوك الوكيل حتميًا ويتجنب المحددات الهشة.

## ذو صلة

- [نظرة عامة على الأدوات](/tools) — جميع أدوات الوكيل المتاحة
- [العزل](/ar/gateway/sandboxing) — التحكم بالمتصفح في البيئات المعزولة
- [الأمان](/ar/gateway/security) — مخاطر التحكم بالمتصفح ووسائل التقوية
