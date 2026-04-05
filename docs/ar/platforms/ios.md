---
read_when:
    - تنفيذ pairing أو إعادة توصيل عقدة iOS
    - تشغيل تطبيق iOS من المصدر
    - تصحيح اكتشاف gateway أو أوامر canvas
summary: 'تطبيق iOS للعقدة: الاتصال بـ Gateway وpairing وcanvas واستكشاف الأخطاء وإصلاحها'
title: تطبيق iOS
x-i18n:
    generated_at: "2026-04-05T12:50:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1e9d9cec58afd4003dff81d3e367bfbc6a634c1b229e433e08fd78fbb5f2e5a9
    source_path: platforms/ios.md
    workflow: 15
---

# تطبيق iOS (العقدة)

التوفر: معاينة داخلية. لم يتم توزيع تطبيق iOS للعامة بعد.

## ما الذي يفعله

- يتصل بـ Gateway عبر WebSocket ‏(LAN أو tailnet).
- يوفّر إمكانات العقدة: Canvas ولقطة الشاشة والتقاط الكاميرا والموقع ووضع التحدث وVoice wake.
- يستقبل أوامر `node.invoke` ويبلّغ عن أحداث حالة العقدة.

## المتطلبات

- Gateway تعمل على جهاز آخر (macOS أو Linux أو Windows عبر WSL2).
- مسار الشبكة:
  - LAN نفسها عبر Bonjour، **أو**
  - tailnet عبر unicast DNS-SD ‏(مثال نطاق: `openclaw.internal.`)، **أو**
  - مضيف/منفذ يدويان (رجوع احتياطي).

## بدء سريع (pair + connect)

1. ابدأ Gateway:

```bash
openclaw gateway --port 18789
```

2. في تطبيق iOS، افتح Settings واختر gateway مكتشفة (أو فعّل Manual Host وأدخل المضيف/المنفذ).

3. وافق على طلب pairing على مضيف gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

إذا أعاد التطبيق محاولة pairing مع تفاصيل مصادقة متغيرة (الدور/النطاقات/المفتاح العام)،
فسيتم استبدال الطلب المعلق السابق وإنشاء `requestId` جديد.
شغّل `openclaw devices list` مرة أخرى قبل الموافقة.

4. تحقّق من الاتصال:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## push مدعومة بالـ relay للبنى الرسمية

تستخدم البنى الرسمية الموزعة لتطبيق iOS push relay خارجية بدلًا من نشر
رمز APNs الخام إلى gateway.

المتطلب من جهة gateway:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

كيف يعمل التدفق:

- يسجّل تطبيق iOS نفسه لدى relay باستخدام App Attest وإيصال التطبيق.
- تعيد relay مقبض relay معتمًا بالإضافة إلى send grant بنطاق التسجيل.
- يجلب التطبيق هوية gateway المقترنة ويضمنها في تسجيل relay، بحيث يتم تفويض التسجيل المدعوم بالـ relay إلى gateway المحددة تلك.
- يمرر التطبيق هذا التسجيل المدعوم بالـ relay إلى gateway المقترنة باستخدام `push.apns.register`.
- تستخدم gateway مقبض relay المخزن هذا في `push.test` وعمليات الإيقاظ في الخلفية وwake nudges.
- يجب أن يطابق relay base URL في gateway عنوان relay URL المضمّن في البنية الرسمية/TestFlight من iOS.
- إذا اتصل التطبيق لاحقًا بـ gateway مختلفة أو ببنية ذات relay base URL مختلف، فإنه يحدّث تسجيل relay بدلًا من إعادة استخدام الربط القديم.

ما الذي **لا** تحتاجه gateway لهذا المسار:

- لا حاجة إلى relay token على مستوى النشر.
- لا حاجة إلى مفتاح APNs مباشر لعمليات الإرسال المدعومة بالـ relay في البنى الرسمية/TestFlight.

تدفق المشغّل المتوقع:

1. ثبّت البنية الرسمية/TestFlight من iOS.
2. اضبط `gateway.push.apns.relay.baseUrl` على gateway.
3. نفّذ pairing للتطبيق مع gateway ودعه يكمل الاتصال.
4. ينشر التطبيق `push.apns.register` تلقائيًا بعد حصوله على رمز APNs، واتصال جلسة المشغّل، ونجاح تسجيل relay.
5. بعد ذلك، يمكن لـ `push.test` وعمليات الإيقاظ عند إعادة الاتصال وwake nudges استخدام التسجيل المدعوم بالـ relay المخزن.

ملاحظة توافق:

- لا يزال `OPENCLAW_APNS_RELAY_BASE_URL` يعمل كتجاوز env مؤقت للـ gateway.

## المصادقة وتدفق الثقة

توجد relay لفرض قيدين لا يستطيع APNs المباشر على gateway توفيرهما
لبنى iOS الرسمية:

- يمكن فقط لبنى OpenClaw الأصلية من iOS الموزعة عبر Apple استخدام relay المستضافة.
- لا يمكن لـ gateway إرسال push مدعومة بالـ relay إلا لأجهزة iOS التي نفذت pairing مع تلك
  gateway تحديدًا.

التدفق خطوة بخطوة:

1. `iOS app -> gateway`
   - ينفذ التطبيق أولًا pairing مع gateway عبر تدفق مصادقة Gateway العادي.
   - وهذا يمنح التطبيق جلسة عقدة مصادقًا عليها بالإضافة إلى جلسة مشغّل مصادقًا عليها.
   - تُستخدم جلسة المشغّل لاستدعاء `gateway.identity.get`.

2. `iOS app -> relay`
   - يستدعي التطبيق نقاط نهاية تسجيل relay عبر HTTPS.
   - يتضمن التسجيل إثبات App Attest بالإضافة إلى إيصال التطبيق.
   - تتحقق relay من bundle ID، وإثبات App Attest، وإيصال Apple، وتطلب
     مسار التوزيع الرسمي/الإنتاجي.
   - وهذا ما يمنع بنى Xcode/dev المحلية من استخدام relay المستضافة. فقد تكون البنية المحلية
     موقعة، لكنها لا تستوفي إثبات توزيع Apple الرسمي الذي تتوقعه relay.

3. `gateway identity delegation`
   - قبل تسجيل relay، يجلب التطبيق هوية gateway المقترنة من
     `gateway.identity.get`.
   - يضمّن التطبيق هوية gateway هذه في حمولة تسجيل relay.
   - تعيد relay مقبض relay وsend grant بنطاق التسجيل يتم تفويضهما إلى
     هوية gateway تلك.

4. `gateway -> relay`
   - تخزن gateway مقبض relay وsend grant من `push.apns.register`.
   - عند `push.test` وعمليات الإيقاظ عند إعادة الاتصال وwake nudges، توقّع gateway طلب الإرسال باستخدام
     هوية الجهاز الخاصة بها.
   - تتحقق relay من send grant المخزن ومن توقيع gateway مقابل
     هوية gateway المفوضة من التسجيل.
   - لا تستطيع gateway أخرى إعادة استخدام هذا التسجيل المخزن، حتى لو حصلت بطريقة ما على المقبض.

5. `relay -> APNs`
   - تملك relay بيانات اعتماد APNs الإنتاجية ورمز APNs الخام للبنية الرسمية.
   - لا تخزّن gateway أبدًا رمز APNs الخام للبنى الرسمية المدعومة بالـ relay.
   - ترسل relay push النهائية إلى APNs نيابةً عن gateway المقترنة.

سبب إنشاء هذا التصميم:

- لإبقاء بيانات اعتماد APNs الإنتاجية خارج بوابات المستخدمين.
- لتجنب تخزين رموز APNs الخام الخاصة بالبنى الرسمية على gateway.
- للسماح باستخدام relay المستضافة فقط لبنى OpenClaw الرسمية/TestFlight.
- لمنع gateway واحدة من إرسال push للإيقاظ إلى أجهزة iOS تملكها gateway مختلفة.

تظل البنى المحلية/اليدوية على APNs المباشر. وإذا كنت تختبر هذه البنى من دون relay، فما تزال
gateway تحتاج إلى بيانات اعتماد APNs مباشرة:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

## مسارات الاكتشاف

### Bonjour ‏(LAN)

يتصفح تطبيق iOS الخدمة `_openclaw-gw._tcp` على `local.` وعند التكوين، نفس
نطاق اكتشاف DNS-SD الواسع. وتظهر Gateways الموجودة على LAN نفسها تلقائيًا من `local.`؛
أما الاكتشاف عبر الشبكات فيمكنه استخدام النطاق الواسع المكوَّن من دون تغيير نوع beacon.

### tailnet ‏(عبر الشبكات)

إذا تم حظر mDNS، فاستخدم منطقة unicast DNS-SD ‏(اختر نطاقًا؛ المثال:
`openclaw.internal.`) وTailscale split DNS.
راجع [Bonjour](/gateway/bonjour) للحصول على مثال CoreDNS.

### مضيف/منفذ يدوي

في Settings، فعّل **Manual Host** وأدخل مضيف gateway + المنفذ (الافتراضي `18789`).

## Canvas + A2UI

تعرض عقدة iOS لوحة WKWebView. استخدم `node.invoke` للتحكم بها:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

ملاحظات:

- يقدّم مضيف canvas في Gateway المسارين `/__openclaw__/canvas/` و`/__openclaw__/a2ui/`.
- يتم تقديمه من خادم HTTP الخاص بـ Gateway ‏(المنفذ نفسه الخاص بـ `gateway.port`، والافتراضي `18789`).
- تنتقل عقدة iOS تلقائيًا إلى A2UI عند الاتصال عندما يتم الإعلان عن عنوان URL لمضيف canvas.
- للعودة إلى scaffold المضمّن، استخدم `canvas.navigate` مع `{"url":""}`.

### Canvas eval / snapshot

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Voice wake + وضع التحدث

- تتوفر Voice wake ووضع التحدث في Settings.
- قد يعلق iOS الصوت في الخلفية؛ لذا تعامل مع ميزات الصوت على أنها بأفضل جهد عندما لا يكون التطبيق نشطًا.

## الأخطاء الشائعة

- `NODE_BACKGROUND_UNAVAILABLE`: اجلب تطبيق iOS إلى الواجهة الأمامية (تتطلب أوامر canvas/camera/screen ذلك).
- `A2UI_HOST_NOT_CONFIGURED`: لم تعلن Gateway عن عنوان URL لمضيف canvas؛ تحقق من `canvasHost` في [Gateway configuration](/gateway/configuration).
- لا يظهر طلب pairing مطلقًا: شغّل `openclaw devices list` ووافق يدويًا.
- يفشل إعادة الاتصال بعد إعادة التثبيت: تم مسح رمز pairing في Keychain؛ أعد pairing للعقدة.

## وثائق ذات صلة

- [Pairing](/channels/pairing)
- [Discovery](/gateway/discovery)
- [Bonjour](/gateway/bonjour)
