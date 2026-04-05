---
read_when:
    - تشغيل OpenClaw Gateway داخل WSL2 بينما يوجد Chrome على Windows
    - رؤية أخطاء متداخلة بين المتصفح وواجهة التحكم عبر WSL2 وWindows
    - الاختيار بين Chrome MCP المحلي على المضيف وCDP البعيد الخام في إعدادات المضيف المنقسم
summary: استكشاف أخطاء WSL2 + Gateway + Chrome البعيد عبر CDP على Windows وإصلاحها على طبقات
title: استكشاف أخطاء WSL2 + Windows + Chrome البعيد عبر CDP وإصلاحها
x-i18n:
    generated_at: "2026-04-05T12:57:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 99df2988d3c6cf36a8c2124d5b724228d095a60b2d2b552f3810709b5086127d
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 15
---

# استكشاف أخطاء WSL2 + Windows + Chrome البعيد عبر CDP وإصلاحها

يغطي هذا الدليل إعداد المضيف المنقسم الشائع حيث:

- يعمل OpenClaw Gateway داخل WSL2
- يعمل Chrome على Windows
- يجب أن يعبر التحكم بالمتصفح الحد الفاصل بين WSL2 وWindows

كما يغطي نمط الفشل متعدد الطبقات من [issue #39369](https://github.com/openclaw/openclaw/issues/39369): إذ يمكن أن تظهر عدة مشكلات مستقلة في الوقت نفسه، مما يجعل الطبقة الخاطئة تبدو معطلة أولًا.

## اختر وضع المتصفح الصحيح أولًا

لديك نمطان صالحان:

### الخيار 1: CDP بعيد خام من WSL2 إلى Windows

استخدم ملف تعريف متصفح بعيد يشير من WSL2 إلى نقطة نهاية CDP خاصة بـ Chrome على Windows.

اختر هذا عندما:

- يبقى Gateway داخل WSL2
- يعمل Chrome على Windows
- تحتاج إلى أن يعبر التحكم بالمتصفح الحد الفاصل بين WSL2 وWindows

### الخيار 2: Chrome MCP محلي على المضيف

استخدم `existing-session` / `user` فقط عندما يعمل Gateway نفسه على المضيف نفسه الذي يعمل عليه Chrome.

اختر هذا عندما:

- يعمل OpenClaw وChrome على الجهاز نفسه
- تريد حالة المتصفح المحلية المسجل دخولها
- لا تحتاج إلى نقل المتصفح عبر مضيفات مختلفة
- لا تحتاج إلى مسارات متقدمة مُدارة/خاصة بـ raw-CDP فقط مثل `responsebody` أو
  تصدير PDF أو اعتراض التنزيلات أو الإجراءات الدفعية

بالنسبة إلى Gateway داخل WSL2 مع Chrome على Windows، فضّل raw remote CDP. إن Chrome MCP محلي على المضيف، وليس جسرًا من WSL2 إلى Windows.

## البنية العاملة

الشكل المرجعي:

- يشغّل WSL2 البوابة على `127.0.0.1:18789`
- يفتح Windows واجهة التحكم في متصفح عادي على `http://127.0.0.1:18789/`
- يكشف Chrome على Windows نقطة نهاية CDP على المنفذ `9222`
- يستطيع WSL2 الوصول إلى نقطة نهاية CDP هذه على Windows
- يوجّه OpenClaw ملف تعريف المتصفح إلى العنوان القابل للوصول من WSL2

## لماذا يبدو هذا الإعداد مربكًا

قد تتداخل عدة حالات فشل:

- لا يستطيع WSL2 الوصول إلى نقطة نهاية CDP على Windows
- تُفتح واجهة التحكم من مصدر غير آمن
- لا تتطابق `gateway.controlUi.allowedOrigins` مع مصدر الصفحة
- الرمز المميز أو الإقران مفقود
- يشير ملف تعريف المتصفح إلى العنوان الخطأ

وبسبب ذلك، فإن إصلاح طبقة واحدة قد يُبقي خطأ مختلفًا ظاهرًا.

## القاعدة الحرجة لواجهة التحكم

عندما تُفتح الواجهة من Windows، استخدم localhost الخاص بـ Windows إلا إذا كان لديك إعداد HTTPS مقصود.

استخدم:

`http://127.0.0.1:18789/`

لا تستخدم عنوان LAN افتراضيًا لواجهة التحكم. فقد يؤدي HTTP العادي على عنوان LAN أو عنوان tailnet إلى سلوك متعلق بالمصدر غير الآمن/مصادقة الجهاز، وهو أمر غير مرتبط بـ CDP نفسه. راجع [واجهة التحكم](/web/control-ui).

## تحقّق على طبقات

اعمل من الأعلى إلى الأسفل. لا تتجاوز المراحل.

### الطبقة 1: التحقق من أن Chrome يقدّم CDP على Windows

شغّل Chrome على Windows مع تفعيل التصحيح البعيد:

```powershell
chrome.exe --remote-debugging-port=9222
```

من Windows، تحقّق أولًا من Chrome نفسه:

```powershell
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:9222/json/list
```

إذا فشل ذلك على Windows، فالمشكلة ليست في OpenClaw بعد.

### الطبقة 2: التحقق من أن WSL2 يستطيع الوصول إلى نقطة النهاية على Windows

من WSL2، اختبر العنوان الدقيق الذي تخطط لاستخدامه في `cdpUrl`:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

النتيجة الجيدة:

- يعيد `/json/version` بيانات JSON تحتوي على معلومات Browser / Protocol-Version
- يعيد `/json/list` بيانات JSON (والمصفوفة الفارغة مقبولة إذا لم تكن هناك صفحات مفتوحة)

إذا فشل ذلك:

- لا يعرّض Windows المنفذ إلى WSL2 بعد
- العنوان غير صحيح من جهة WSL2
- لا يزال جدار الحماية / إعادة توجيه المنفذ / الوكيل المحلي مفقودًا

أصلح ذلك قبل تعديل إعدادات OpenClaw.

### الطبقة 3: إعداد ملف تعريف المتصفح الصحيح

بالنسبة إلى raw remote CDP، وجّه OpenClaw إلى العنوان القابل للوصول من WSL2:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

ملاحظات:

- استخدم العنوان القابل للوصول من WSL2، وليس العنوان الذي يعمل فقط على Windows
- أبقِ `attachOnly: true` للمتصفحات المُدارة خارجيًا
- يمكن أن تكون `cdpUrl` من النوع `http://` أو `https://` أو `ws://` أو `wss://`
- استخدم HTTP(S) عندما تريد أن يكتشف OpenClaw المسار `/json/version`
- استخدم WS(S) فقط عندما يمنحك مزوّد المتصفح عنوان socket مباشرًا لـ DevTools
- اختبر عنوان URL نفسه باستخدام `curl` قبل أن تتوقع نجاح OpenClaw

### الطبقة 4: التحقق من طبقة واجهة التحكم بشكل منفصل

افتح الواجهة من Windows:

`http://127.0.0.1:18789/`

ثم تحقّق من:

- أن مصدر الصفحة يطابق ما تتوقعه `gateway.controlUi.allowedOrigins`
- أن مصادقة الرمز المميز أو الإقران مضبوطة بشكل صحيح
- أنك لا تعالج مشكلة مصادقة في واجهة التحكم كما لو كانت مشكلة في المتصفح

صفحة مفيدة:

- [واجهة التحكم](/web/control-ui)

### الطبقة 5: التحقق من التحكم الكامل بالمتصفح من البداية إلى النهاية

من WSL2:

```bash
openclaw browser open https://example.com --browser-profile remote
openclaw browser tabs --browser-profile remote
```

النتيجة الجيدة:

- تُفتح علامة التبويب في Chrome على Windows
- يعيد `openclaw browser tabs` الهدف
- تعمل الإجراءات اللاحقة (`snapshot` و`screenshot` و`navigate`) من ملف التعريف نفسه

## أخطاء شائعة مضللة

تعامل مع كل رسالة على أنها مؤشر خاص بطبقة معينة:

- `control-ui-insecure-auth`
  - مشكلة في مصدر الواجهة / السياق الآمن، وليست مشكلة في نقل CDP
- `token_missing`
  - مشكلة في إعدادات المصادقة
- `pairing required`
  - مشكلة في اعتماد الجهاز
- `Remote CDP for profile "remote" is not reachable`
  - لا يستطيع WSL2 الوصول إلى `cdpUrl` المضبوط
- `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable`
  - استجابت نقطة نهاية HTTP، لكن تعذّر مع ذلك فتح WebSocket الخاص بـ DevTools
- استمرار تجاوزات viewport / الوضع الداكن / اللغة المحلية / عدم الاتصال بعد جلسة بعيدة
  - شغّل `openclaw browser stop --browser-profile remote`
  - يؤدي هذا إلى إغلاق جلسة التحكم النشطة وتحرير حالة محاكاة Playwright/CDP من دون إعادة تشغيل البوابة أو المتصفح الخارجي
- `gateway timeout after 1500ms`
  - غالبًا ما تكون ما تزال مشكلة وصول CDP أو نقطة نهاية بعيدة بطيئة/غير قابلة للوصول
- `No Chrome tabs found for profile="user"`
  - تم اختيار ملف تعريف Chrome MCP محلي حيث لا توجد علامات تبويب محلية متاحة على المضيف

## قائمة تحقق سريعة للفرز

1. Windows: هل يعمل `curl http://127.0.0.1:9222/json/version`؟
2. WSL2: هل يعمل `curl http://WINDOWS_HOST_OR_IP:9222/json/version`؟
3. إعدادات OpenClaw: هل تستخدم `browser.profiles.<name>.cdpUrl` هذا العنوان الدقيق القابل للوصول من WSL2؟
4. واجهة التحكم: هل تفتح `http://127.0.0.1:18789/` بدلًا من عنوان LAN؟
5. هل تحاول استخدام `existing-session` عبر WSL2 وWindows بدلًا من raw remote CDP؟

## الخلاصة العملية

يكون هذا الإعداد عادةً قابلًا للتنفيذ. الجزء الصعب هو أن نقل المتصفح، وأمان مصدر واجهة التحكم، والرمز المميز/الإقران، يمكن أن يفشل كل منها بشكل مستقل مع أنها تبدو متشابهة من جهة المستخدم.

عند الشك:

- تحقّق أولًا من نقطة نهاية Chrome على Windows محليًا
- ثم تحقّق من نقطة النهاية نفسها من WSL2
- وبعدها فقط ابدأ في تصحيح إعدادات OpenClaw أو مصادقة واجهة التحكم
