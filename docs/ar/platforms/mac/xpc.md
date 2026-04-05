---
read_when:
    - تعديل عقود IPC أو IPC الخاصة بتطبيق شريط القوائم
summary: بنية IPC في macOS لتطبيق OpenClaw، ونقل عقدة gateway، وPeekabooBridge
title: IPC في macOS
x-i18n:
    generated_at: "2026-04-05T12:50:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: d0211c334a4a59b71afb29dd7b024778172e529fa618985632d3d11d795ced92
    source_path: platforms/mac/xpc.md
    workflow: 15
---

# بنية OpenClaw IPC في macOS

**النموذج الحالي:** يربط مقبس Unix محلي **خدمة مضيف العقدة** بـ **تطبيق macOS** من أجل موافقات exec و`system.run`. ويوجد CLI للتصحيح باسم `openclaw-mac` لفحوصات الاكتشاف/الاتصال؛ ولا تزال إجراءات الوكيل تمر عبر Gateway WebSocket و`node.invoke`. وتستخدم أتمتة واجهة المستخدم PeekabooBridge.

## الأهداف

- مثيل واحد من تطبيق GUI يملك كل الأعمال المواجهة لـ TCC ‏(الإشعارات، وتسجيل الشاشة، والميكروفون، والنطق، وAppleScript).
- سطح صغير للأتمتة: Gateway + أوامر العقدة، بالإضافة إلى PeekabooBridge لأتمتة واجهة المستخدم.
- أذونات متوقعة: معرّف حزمة موقّع واحد دائمًا، ويتم تشغيله بواسطة launchd، بحيث تبقى منح TCC ثابتة.

## كيف يعمل

### Gateway + نقل العقدة

- يشغّل التطبيق Gateway ‏(الوضع المحلي) ويتصل به كعقدة.
- تُنفذ إجراءات الوكيل عبر `node.invoke` ‏(مثل `system.run` و`system.notify` و`canvas.*`).

### خدمة العقدة + IPC التطبيق

- تتصل خدمة مضيف عقدة بلا واجهة بـ Gateway WebSocket.
- تُمرَّر طلبات `system.run` إلى تطبيق macOS عبر مقبس Unix محلي.
- ينفذ التطبيق exec في سياق UI، ويطلب الإذن عند الحاجة، ثم يعيد المخرجات.

المخطط (SCI):

```
Agent -> Gateway -> Node Service (WS)
                      |  IPC (UDS + token + HMAC + TTL)
                      v
                  Mac App (UI + TCC + system.run)
```

### PeekabooBridge ‏(أتمتة واجهة المستخدم)

- تستخدم أتمتة واجهة المستخدم مقبس UNIX منفصلًا باسم `bridge.sock` وبروتوكول JSON الخاص بـ PeekabooBridge.
- ترتيب تفضيل المضيف (من جهة العميل): Peekaboo.app → Claude.app → OpenClaw.app → التنفيذ المحلي.
- الأمان: تتطلب مضيفات الجسر TeamID مسموحًا؛ ويُحمى منفذ الهروب DEBUG-only للمستخدم نفسه بواسطة `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` ‏(وفق اصطلاح Peekaboo).
- راجع: [استخدام PeekabooBridge](/platforms/mac/peekaboo) للتفاصيل.

## التدفقات التشغيلية

- إعادة التشغيل/إعادة البناء: `SIGN_IDENTITY="Apple Development: <Developer Name> (<TEAMID>)" scripts/restart-mac.sh`
  - يقتل المثيلات الموجودة
  - بناء Swift + الحزم
  - يكتب LaunchAgent ويقوم بالـ bootstrap/‏kickstart له
- مثيل واحد: يخرج التطبيق مبكرًا إذا كان هناك مثيل آخر يعمل بمعرّف الحزمة نفسه.

## ملاحظات التقوية

- فضّل طلب تطابق TeamID لجميع الأسطح ذات الامتيازات.
- PeekabooBridge: قد يسمح `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` ‏(DEBUG-only) للمتصلين من المستخدم نفسه أثناء التطوير المحلي.
- تبقى كل الاتصالات محلية فقط؛ ولا يتم كشف أي مقابس شبكية.
- تصدر مطالبات TCC فقط من حزمة تطبيق GUI؛ وحافظ على ثبات معرّف الحزمة الموقعة عبر عمليات إعادة البناء.
- تقوية IPC: وضع المقبس `0600`، ورمز، وفحوصات peer-UID، وتحدي/استجابة HMAC، وTTL قصير.
