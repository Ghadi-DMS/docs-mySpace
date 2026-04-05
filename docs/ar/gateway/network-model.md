---
read_when:
    - تريد عرضًا موجزًا لنموذج شبكات Gateway
summary: كيفية اتصال Gateway والعُقد وcanvas host.
title: نموذج الشبكة
x-i18n:
    generated_at: "2026-04-05T12:43:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7d02d87f38ee5a9fae228f5028892b192c50b473ab4441bbe0b40ee85a1dd402
    source_path: gateway/network-model.md
    workflow: 15
---

# نموذج الشبكة

> تم دمج هذا المحتوى في [الشبكة](/network#core-model). راجع تلك الصفحة للحصول على الدليل الحالي.

تمر معظم العمليات عبر Gateway (`openclaw gateway`)، وهي عملية واحدة طويلة التشغيل
تملك اتصالات القنوات ومستوى التحكم WebSocket.

## القواعد الأساسية

- يُوصى باستخدام Gateway واحدة لكل مضيف. وهي العملية الوحيدة المسموح لها بامتلاك جلسة WhatsApp Web. وبالنسبة إلى bots الإنقاذ أو العزل الصارم، شغّل عدة Gateways مع ملفات شخصية ومنافذ معزولة. راجع [Gateways متعددة](/gateway/multiple-gateways).
- loopback أولًا: تكون القيمة الافتراضية لـ Gateway WS هي `ws://127.0.0.1:18789`. ينشئ المعالج مصادقة سر مشترك افتراضيًا وعادةً ما يولّد token، حتى مع loopback. وللوصول غير المعتمد على loopback، استخدم مسار مصادقة Gateway صالحًا: مصادقة token/password ذات السر المشترك، أو نشر `trusted-proxy` غير loopback مضبوطًا بشكل صحيح. وتعمل بيئات tailnet/الهاتف المحمول عادةً بشكل أفضل عبر Tailscale Serve أو نقطة نهاية `wss://` أخرى بدلًا من `ws://` خام عبر tailnet.
- تتصل العُقد بـ Gateway WS عبر LAN أو tailnet أو SSH حسب الحاجة. وقد
  تمت إزالة الجسر القديم المستند إلى TCP.
- يتم تقديم canvas host بواسطة خادم HTTP الخاص بـ Gateway على **المنفذ نفسه** الخاص بـ Gateway (الافتراضي `18789`):
  - `/__openclaw__/canvas/`
  - `/__openclaw__/a2ui/`
    عند تكوين `gateway.auth` وربط Gateway خارج loopback، تتم حماية هذه المسارات بواسطة مصادقة Gateway. وتستخدم عملاء العُقد URLات قدرات بنطاق العُقدة مرتبطة بجلسة WS النشطة الخاصة بها. راجع [تكوين Gateway](/gateway/configuration) (`canvasHost`، `gateway`).
- يكون الاستخدام البعيد عادةً عبر نفق SSH أو VPN خاص بـ tailnet. راجع [الوصول عن بُعد](/gateway/remote) و[الاكتشاف](/gateway/discovery).
