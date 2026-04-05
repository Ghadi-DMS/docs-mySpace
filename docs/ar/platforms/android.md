---
read_when:
    - اقتران عقدة Android أو إعادة توصيلها
    - تصحيح اكتشاف gateway أو المصادقة في Android
    - التحقق من تطابق سجل الدردشة عبر العملاء
summary: 'تطبيق Android ‏(العقدة): دليل التشغيل للاتصال + سطح أوامر Connect/Chat/Voice/Canvas'
title: تطبيق Android
x-i18n:
    generated_at: "2026-04-05T12:49:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2223891afc3aa34af4aaf5410b4f1c6aebcf24bab68a6c47dd9832882d5260db
    source_path: platforms/android.md
    workflow: 15
---

# تطبيق Android ‏(العقدة)

> **ملاحظة:** لم يتم إصدار تطبيق Android للعامة بعد. الشيفرة المصدرية متاحة في [مستودع OpenClaw](https://github.com/openclaw/openclaw) ضمن `apps/android`. يمكنك بناؤه بنفسك باستخدام Java 17 وAndroid SDK ‏(`./gradlew :app:assemblePlayDebug`). راجع [apps/android/README.md](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md) للحصول على تعليمات البناء.

## لمحة سريعة عن الدعم

- الدور: تطبيق عقدة مرافق (لا يستضيف Android الـ Gateway).
- هل يتطلب Gateway؟ نعم (شغّله على macOS أو Linux أو Windows عبر WSL2).
- التثبيت: [البدء](/ar/start/getting-started) + [الاقتران](/channels/pairing).
- Gateway: [دليل التشغيل](/gateway) + [الإعدادات](/gateway/configuration).
  - البروتوكولات: [بروتوكول Gateway](/gateway/protocol) ‏(العقد + مستوى التحكم).

## التحكم في النظام

يوجد التحكم في النظام (`launchd/systemd`) على مضيف Gateway. راجع [Gateway](/gateway).

## دليل تشغيل الاتصال

تطبيق عقدة Android ⇄ ‏(mDNS/NSD + WebSocket) ⇄ **Gateway**

يتصل Android مباشرة بـ Gateway WebSocket ويستخدم اقتران الجهاز (`role: node`).

بالنسبة إلى Tailscale أو المضيفين العموميين، يتطلب Android نقطة نهاية آمنة:

- المفضل: Tailscale Serve / Funnel مع `https://<magicdns>` / `wss://<magicdns>`
- مدعوم أيضًا: أي عنوان URL آخر لـ Gateway من نوع `wss://` مع نقطة نهاية TLS حقيقية
- لا يزال `ws://` غير المشفر مدعومًا على عناوين LAN الخاصة / مضيفات `.local`، بالإضافة إلى `localhost` و`127.0.0.1` وجسر محاكي Android ‏(`10.0.2.2`)

### المتطلبات المسبقة

- يمكنك تشغيل Gateway على الجهاز "الرئيسي".
- يستطيع جهاز/محاكي Android الوصول إلى Gateway WebSocket:
  - على الشبكة المحلية نفسها باستخدام mDNS/NSD، **أو**
  - على tailnet نفسها في Tailscale باستخدام Wide-Area Bonjour / unicast DNS-SD ‏(انظر أدناه)، **أو**
  - مضيف/منفذ gateway يدويًا (تراجع)
- لا يستخدم اقتران Android المحمول عبر tailnet/public نقاط نهاية raw tailnet IP من نوع `ws://`. استخدم Tailscale Serve أو عنوان URL آخر من نوع `wss://` بدلًا من ذلك.
- يمكنك تشغيل CLI ‏(`openclaw`) على جهاز gateway ‏(أو عبر SSH).

### 1) ابدأ Gateway

```bash
openclaw gateway --port 18789 --verbose
```

أكّد في السجلات أنك ترى شيئًا مثل:

- `listening on ws://0.0.0.0:18789`

للوصول البعيد من Android عبر Tailscale، فضّل Serve/Funnel بدلًا من الربط الخام على tailnet:

```bash
openclaw gateway --tailscale serve
```

يوفر هذا لـ Android نقطة نهاية آمنة من نوع `wss://` / `https://`. ولا يكفي إعداد `gateway.bind: "tailnet"` العادي لأول عملية اقتران عن بُعد مع Android ما لم تقم أيضًا بإنهاء TLS بشكل منفصل.

### 2) تحقّق من الاكتشاف (اختياري)

من جهاز gateway:

```bash
dns-sd -B _openclaw-gw._tcp local.
```

مزيد من ملاحظات التصحيح: [Bonjour](/gateway/bonjour).

إذا كنت قد أعددت أيضًا نطاق اكتشاف واسع المجال، فقارن مع:

```bash
openclaw gateway discover --json
```

يعرض هذا `local.` بالإضافة إلى النطاق واسع المجال المهيأ في تمريرة واحدة ويستخدم
نقطة نهاية الخدمة المحلولة بدلًا من تلميحات TXT فقط.

#### اكتشاف tailnet ‏(Vienna ⇄ London) عبر unicast DNS-SD

لن يعبر اكتشاف Android باستخدام NSD/mDNS الشبكات. إذا كانت عقدة Android وgateway على شبكتين مختلفتين لكنهما متصلتان عبر Tailscale، فاستخدم Wide-Area Bonjour / unicast DNS-SD بدلًا من ذلك.

لا يكفي الاكتشاف وحده لاقتران Android عبر tailnet/public. فما يزال المسار المكتشف يحتاج إلى نقطة نهاية آمنة (`wss://` أو Tailscale Serve):

1. أعد منطقة DNS-SD ‏(مثل `openclaw.internal.`) على مضيف gateway وانشر سجلات `_openclaw-gw._tcp`.
2. اضبط Tailscale split DNS للنطاق الذي اخترته مع توجيهه إلى خادم DNS ذلك.

التفاصيل ومثال إعداد CoreDNS: ‏[Bonjour](/gateway/bonjour).

### 3) اتصل من Android

في تطبيق Android:

- يحافظ التطبيق على اتصال gateway حيًا عبر **خدمة foreground** ‏(إشعار دائم).
- افتح علامة التبويب **Connect**.
- استخدم وضع **Setup Code** أو **Manual**.
- إذا كان الاكتشاف محظورًا، فاستخدم المضيف/المنفذ اليدويين في **Advanced controls**. بالنسبة إلى مضيفات LAN الخاصة، لا يزال `ws://` يعمل. أما بالنسبة إلى مضيفات Tailscale/public، ففعّل TLS واستخدم نقطة نهاية `wss://` / Tailscale Serve.

بعد أول اقتران ناجح، يعيد Android الاتصال تلقائيًا عند التشغيل:

- نقطة النهاية اليدوية (إن كانت مفعلة)، وإلا
- آخر Gateway مكتشف (بأفضل جهد).

### 4) وافق على الاقتران (CLI)

على جهاز gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

تفاصيل الاقتران: [الاقتران](/channels/pairing).

### 5) تحقّق من أن العقدة متصلة

- عبر حالة العقد:

  ```bash
  openclaw nodes status
  ```

- عبر Gateway:

  ```bash
  openclaw gateway call node.list --params "{}"
  ```

### 6) الدردشة + السجل

تدعم علامة التبويب Chat في Android اختيار الجلسة (الافتراضي `main`، بالإضافة إلى الجلسات الموجودة الأخرى):

- السجل: `chat.history` ‏(مطبّع للعرض؛ تتم إزالة وسوم التوجيه المضمنة
  من النص المرئي، كما تتم إزالة حمولات XML النصية العادية لاستدعاء الأدوات (بما في ذلك
  `<tool_call>...</tool_call>` و`<function_call>...</function_call>`،
  و`<tool_calls>...</tool_calls>` و`<function_calls>...</function_calls>`، و
  كتل استدعاء الأدوات المقتطعة) ورموز التحكم المتسربة للنموذج بنسختي ASCII/العرض الكامل،
  وتُحذف صفوف المساعد الصامتة البحتة مثل `NO_REPLY` /
  `no_reply` المطابقة تمامًا، ويمكن استبدال الصفوف كبيرة الحجم بعناصر نائبة)
- الإرسال: `chat.send`
- تحديثات push ‏(بأفضل جهد): `chat.subscribe` → `event:"chat"`

### 7) Canvas + الكاميرا

#### Gateway Canvas Host ‏(موصى به لمحتوى الويب)

إذا كنت تريد أن تعرض العقدة HTML/CSS/JS حقيقيًا يمكن للوكيل تعديله على القرص، فوجه العقدة إلى مضيف canvas الخاص بـ Gateway.

ملاحظة: تقوم العقد بتحميل canvas من خادم HTTP الخاص بـ Gateway ‏(المنفذ نفسه الخاص بـ `gateway.port`، والافتراضي `18789`).

1. أنشئ `~/.openclaw/workspace/canvas/index.html` على مضيف gateway.

2. انتقل بالعقدة إليه (LAN):

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet ‏(اختياري): إذا كان كلا الجهازين على Tailscale، فاستخدم اسم MagicDNS أو tailnet IP بدل `.local`، مثل `http://<gateway-magicdns>:18789/__openclaw__/canvas/`.

يقوم هذا الخادم بحقن عميل live-reload في HTML ويعيد التحميل عند تغيّر الملفات.
ويقع مضيف A2UI عند `http://<gateway-host>:18789/__openclaw__/a2ui/`.

أوامر Canvas ‏(foreground فقط):

- `canvas.eval` و`canvas.snapshot` و`canvas.navigate` ‏(استخدم `{"url":""}` أو `{"url":"/"}` للعودة إلى البنية الافتراضية). يعيد `canvas.snapshot` القيمة `{ format, base64 }` ‏(الافتراضي `format="jpeg"`).
- A2UI: ‏`canvas.a2ui.push` و`canvas.a2ui.reset` ‏(`canvas.a2ui.pushJSONL` اسم بديل قديم)

أوامر الكاميرا (foreground فقط؛ محكومة بالأذونات):

- `camera.snap` ‏(jpg)
- `camera.clip` ‏(mp4)

راجع [عقدة الكاميرا](/nodes/camera) للمعلمات ومساعدات CLI.

### 8) الصوت + سطح أوامر Android الموسع

- الصوت: يستخدم Android تدفق تشغيل/إيقاف واحدًا للميكروفون في علامة التبويب Voice مع التقاط النص المفرغ وتشغيل `talk.speak`. ويتم استخدام TTS المحلي للنظام فقط عندما لا يكون `talk.speak` متاحًا. ويتوقف الصوت عندما يغادر التطبيق foreground.
- تمت إزالة مفاتيح Voice wake/talk-mode حاليًا من تجربة Android ووقت تشغيله.
- عائلات أوامر Android الإضافية (يعتمد توفرها على الجهاز + الأذونات):
  - `device.status` و`device.info` و`device.permissions` و`device.health`
  - `notifications.list` و`notifications.actions` ‏(راجع [إعادة توجيه الإشعارات](#notification-forwarding) أدناه)
  - `photos.latest`
  - `contacts.search` و`contacts.add`
  - `calendar.events` و`calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity` و`motion.pedometer`

## نقاط دخول المساعد

يدعم Android تشغيل OpenClaw من مشغّل المساعد في النظام (Google
Assistant). وعند إعداد ذلك، يؤدي الضغط المطول على زر الصفحة الرئيسية أو قول "Hey Google, ask
OpenClaw..." إلى فتح التطبيق وتمرير prompt إلى محرر الدردشة.

يستخدم هذا بيانات **App Actions** الوصفية في Android المعلنة في manifest الخاص بالتطبيق. ولا
يلزم أي إعداد إضافي على جانب gateway — إذ يتم التعامل مع نية المساعد بالكامل بواسطة
تطبيق Android وتمريرها كرسالة دردشة عادية.

<Note>
يعتمد توفر App Actions على الجهاز، وإصدار Google Play Services،
وعلى ما إذا كان المستخدم قد عيّن OpenClaw كتطبيق المساعد الافتراضي.
</Note>

## إعادة توجيه الإشعارات

يمكن لـ Android إعادة توجيه إشعارات الجهاز إلى gateway كأحداث. وتتيح لك عدة عناصر تحكم تحديد نطاق الإشعارات المعاد توجيهها ومتى يتم ذلك.

| المفتاح                          | النوع          | الوصف                                                                                          |
| -------------------------------- | -------------- | ---------------------------------------------------------------------------------------------- |
| `notifications.allowPackages`    | string[]       | إعادة توجيه الإشعارات فقط من أسماء الحزم هذه. وإذا تم ضبطه، يتم تجاهل جميع الحزم الأخرى.      |
| `notifications.denyPackages`     | string[]       | عدم إعادة توجيه الإشعارات مطلقًا من أسماء الحزم هذه. ويُطبّق بعد `allowPackages`.             |
| `notifications.quietHours.start` | string (HH:mm) | بداية نافذة ساعات الهدوء (بالتوقيت المحلي للجهاز). يتم كتم الإشعارات أثناء هذه النافذة.        |
| `notifications.quietHours.end`   | string (HH:mm) | نهاية نافذة ساعات الهدوء.                                                                       |
| `notifications.rateLimit`        | number         | الحد الأقصى للإشعارات المعاد توجيهها لكل حزمة في الدقيقة. ويتم إسقاط الإشعارات الزائدة.        |

يستخدم منتقي الإشعارات أيضًا سلوكًا أكثر أمانًا لأحداث الإشعارات المعاد توجيهها، مما يمنع إعادة التوجيه العرضية لإشعارات النظام الحساسة.

مثال على الإعدادات:

```json5
{
  notifications: {
    allowPackages: ["com.slack", "com.whatsapp"],
    denyPackages: ["com.android.systemui"],
    quietHours: {
      start: "22:00",
      end: "07:00",
    },
    rateLimit: 5,
  },
}
```

<Note>
تتطلب إعادة توجيه الإشعارات إذن Android Notification Listener. ويطلب التطبيق هذا أثناء الإعداد.
</Note>
