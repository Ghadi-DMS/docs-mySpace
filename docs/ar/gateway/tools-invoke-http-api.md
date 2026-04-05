---
read_when:
    - استدعاء الأدوات من دون تشغيل دور وكيل كامل
    - إنشاء عمليات أتمتة تحتاج إلى فرض سياسة الأدوات
summary: استدعِ أداة واحدة مباشرة عبر نقطة نهاية HTTP الخاصة بـ Gateway
title: Tools Invoke API
x-i18n:
    generated_at: "2026-04-05T12:44:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: e924f257ba50b25dea0ec4c3f9eed4c8cac8a53ddef18215f87ac7de330a37fd
    source_path: gateway/tools-invoke-http-api.md
    workflow: 15
---

# Tools Invoke (HTTP)

يكشف Gateway في OpenClaw نقطة نهاية HTTP بسيطة لاستدعاء أداة واحدة مباشرة. وهي مفعلة دائمًا وتستخدم مصادقة Gateway بالإضافة إلى سياسة الأدوات. ومثل سطح `/v1/*` المتوافق مع OpenAI، تُعامل مصادقة bearer ذات السر المشترك على أنها وصول مشغّل موثوق لكامل البوابة.

- `POST /tools/invoke`
- المنفذ نفسه الخاص بـ Gateway (تعدد WS + HTTP): ‏`http://<gateway-host>:<port>/tools/invoke`

الحد الأقصى الافتراضي لحجم الحمولة هو 2 MB.

## المصادقة

تستخدم إعدادات مصادقة Gateway.

مسارات مصادقة HTTP الشائعة:

- مصادقة السر المشترك (`gateway.auth.mode="token"` أو `"password"`):
  ‏`Authorization: Bearer <token-or-password>`
- مصادقة HTTP الموثوقة الحاملة للهوية (`gateway.auth.mode="trusted-proxy"`):
  وجّه الطلب عبر proxy مدرك للهوية ومهيأ ودعه يحقن
  ترويسات الهوية المطلوبة
- مصادقة open auth على private-ingress (`gateway.auth.mode="none"`):
  لا حاجة إلى ترويسة مصادقة

ملاحظات:

- عندما تكون `gateway.auth.mode="token"`، استخدم `gateway.auth.token` (أو `OPENCLAW_GATEWAY_TOKEN`).
- عندما تكون `gateway.auth.mode="password"`، استخدم `gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`).
- عندما تكون `gateway.auth.mode="trusted-proxy"`، يجب أن يأتي طلب HTTP من
  مصدر trusted proxy مهيأ وغير loopback؛ ولا تستوفي proxies ذات loopback على المضيف نفسه
  هذا الوضع.
- إذا كانت `gateway.auth.rateLimit` مهيأة وحدث عدد كبير جدًا من حالات فشل المصادقة، فستعيد نقطة النهاية `429` مع `Retry-After`.

## الحد الأمني (مهم)

تعامل مع نقطة النهاية هذه على أنها سطح **وصول تشغيلي كامل** لمثيل البوابة.

- مصادقة HTTP bearer هنا ليست نموذج نطاقات ضيقًا لكل مستخدم.
- يجب التعامل مع رمز Gateway أو كلمة المرور الصالحة لهذه النقطة على أنها بيانات اعتماد مالك/مشغّل.
- بالنسبة إلى أوضاع مصادقة السر المشترك (`token` و`password`)، تستعيد نقطة النهاية القيم الافتراضية التشغيلية الكاملة المعتادة حتى إذا أرسل المستدعي ترويسة `x-openclaw-scopes` أضيق.
- كما تتعامل مصادقة السر المشترك مع استدعاءات الأدوات المباشرة على هذه النقطة على أنها أدوار مرسل مالك.
- تحترم أوضاع HTTP الموثوقة الحاملة للهوية (مثل trusted proxy auth أو `gateway.auth.mode="none"` على private ingress) الترويسة `x-openclaw-scopes` عند وجودها، وإلا فترجع إلى مجموعة النطاقات الافتراضية العادية للمشغل.
- أبقِ نقطة النهاية هذه على loopback/tailnet/private ingress فقط؛ ولا تكشفها مباشرة للإنترنت العام.

مصفوفة المصادقة:

- `gateway.auth.mode="token"` أو `"password"` + ‏`Authorization: Bearer ...`
  - يثبت امتلاك السر التشغيلي المشترك للـ gateway
  - يتجاهل `x-openclaw-scopes` الأضيق
  - يستعيد مجموعة النطاقات الافتراضية الكاملة للمشغل:
    `operator.admin` و`operator.approvals` و`operator.pairing`،
    و`operator.read` و`operator.talk.secrets` و`operator.write`
  - يتعامل مع استدعاءات الأدوات المباشرة على هذه النقطة على أنها أدوار مرسل مالك
- أوضاع HTTP الموثوقة الحاملة للهوية (مثل trusted proxy auth، أو `gateway.auth.mode="none"` على private ingress)
  - تصادق على هوية موثوقة خارجية أو حد نشر موثوق
  - تحترم `x-openclaw-scopes` عندما تكون الترويسة موجودة
  - ترجع إلى مجموعة النطاقات الافتراضية العادية للمشغل عندما تكون الترويسة غائبة
  - لا تفقد دلالات المالك إلا عندما يضيّق المستدعي النطاقات صراحةً ويحذف `operator.admin`

## جسم الطلب

```json
{
  "tool": "sessions_list",
  "action": "json",
  "args": {},
  "sessionKey": "main",
  "dryRun": false
}
```

الحقول:

- `tool` (سلسلة نصية، مطلوب): اسم الأداة المراد استدعاؤها.
- `action` (سلسلة نصية، اختياري): يُربط داخل args إذا كان مخطط الأداة يدعم `action` وكان حمل args قد حذفه.
- `args` (كائن، اختياري): وسائط خاصة بالأداة.
- `sessionKey` (سلسلة نصية، اختياري): مفتاح الجلسة الهدف. إذا تم حذفه أو كان `"main"`، تستخدم Gateway مفتاح الجلسة الرئيسية المهيأ (مع احترام `session.mainKey` والوكيل الافتراضي، أو `global` في النطاق العام).
- `dryRun` (منطقي، اختياري): محجوز لاستخدام مستقبلي؛ ويتم تجاهله حاليًا.

## سلوك السياسة + التوجيه

تتم تصفية توفر الأداة عبر سلسلة السياسة نفسها المستخدمة من قبل وكلاء Gateway:

- `tools.profile` / `tools.byProvider.profile`
- `tools.allow` / `tools.byProvider.allow`
- `agents.<id>.tools.allow` / `agents.<id>.tools.byProvider.allow`
- سياسات المجموعات (إذا كان مفتاح الجلسة يرتبط بمجموعة أو قناة)
- سياسة الوكيل الفرعي (عند الاستدعاء باستخدام مفتاح جلسة وكيل فرعي)

إذا لم تكن الأداة مسموحًا بها وفق السياسة، تعيد نقطة النهاية **404**.

ملاحظات مهمة حول الحدود:

- موافقات Exec هي وسائل حماية تشغيلية، وليست حدًا منفصلًا للتفويض بالنسبة إلى نقطة نهاية HTTP هذه. إذا كانت الأداة قابلة للوصول هنا عبر مصادقة Gateway + سياسة الأدوات، فإن `/tools/invoke` لا يضيف مطالبة موافقة إضافية لكل استدعاء.
- لا تشارك بيانات اعتماد Gateway bearer مع جهات غير موثوقة. إذا كنت بحاجة إلى فصل عبر حدود الثقة، فشغّل بوابات منفصلة (ويفضّل أيضًا مستخدمو/مضيفو نظام تشغيل منفصلون).

كما تطبق Gateway HTTP قائمة رفض صارمة افتراضيًا (حتى إذا سمحت سياسة الجلسة بالأداة):

- `exec` — تنفيذ أوامر مباشرة (سطح RCE)
- `spawn` — إنشاء عمليات فرعية عشوائية (سطح RCE)
- `shell` — تنفيذ أوامر shell (سطح RCE)
- `fs_write` — تعديل ملفات المضيف بشكل عشوائي
- `fs_delete` — حذف ملفات المضيف بشكل عشوائي
- `fs_move` — نقل/إعادة تسمية ملفات المضيف بشكل عشوائي
- `apply_patch` — يمكن لتطبيق الرقع إعادة كتابة ملفات عشوائية
- `sessions_spawn` — طبقة تنسيق الجلسات؛ تشغيل الوكلاء عن بُعد هو RCE
- `sessions_send` — حقن رسائل عبر الجلسات
- `cron` — طبقة التحكم في الأتمتة الدائمة
- `gateway` — طبقة التحكم في البوابة؛ تمنع إعادة الإعداد عبر HTTP
- `nodes` — يمكن لمرحل أوامر العقدة الوصول إلى system.run على المضيفين المقترنين
- `whatsapp_login` — إعداد تفاعلي يتطلب مسح QR من الطرفية؛ ويتعطل على HTTP

يمكنك تخصيص قائمة الرفض هذه عبر `gateway.tools`:

```json5
{
  gateway: {
    tools: {
      // أدوات إضافية لحظرها عبر HTTP /tools/invoke
      deny: ["browser"],
      // إزالة أدوات من قائمة الرفض الافتراضية
      allow: ["gateway"],
    },
  },
}
```

وللمساعدة في حل سياق سياسات المجموعات، يمكنك اختياريًا تعيين:

- `x-openclaw-message-channel: <channel>` (مثال: `slack`، `telegram`)
- `x-openclaw-account-id: <accountId>` (عند وجود حسابات متعددة)

## الاستجابات

- `200` ← `{ ok: true, result }`
- `400` ← `{ ok: false, error: { type, message } }` (طلب غير صالح أو خطأ في إدخال الأداة)
- `401` ← غير مصرح
- `429` ← تم تطبيق حد المعدل على المصادقة (`Retry-After` مضبوط)
- `404` ← الأداة غير متاحة (غير موجودة أو غير موجودة في قائمة السماح)
- `405` ← الطريقة غير مسموح بها
- `500` ← `{ ok: false, error: { type, message } }` (خطأ غير متوقع أثناء تنفيذ الأداة؛ رسالة منقحة)

## مثال

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer secret' \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "sessions_list",
    "action": "json",
    "args": {}
  }'
```
