---
read_when:
    - تحتاج إلى نظرة عامة على بنية الشبكة + الأمان
    - أنت تصحح أخطاء الوصول المحلي مقابل tailnet أو الاقتران
    - تريد القائمة المعتمدة لوثائق الشبكات
summary: 'مركز الشبكة: أسطح gateway، والاقتران، والاكتشاف، والأمان'
title: الشبكة
x-i18n:
    generated_at: "2026-04-05T12:48:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a5f39d4f40ad19646d372000c85b663770eae412af91e1c175eb27b22208118
    source_path: network.md
    workflow: 15
---

# مركز الشبكة

يربط هذا المركز الوثائق الأساسية لكيفية اتصال OpenClaw بالأجهزة واقترانها وتأمينها
عبر localhost وLAN وtailnet.

## النموذج الأساسي

تمر معظم العمليات عبر Gateway ‏(`openclaw gateway`)، وهي عملية واحدة طويلة التشغيل تملك اتصالات القنوات ومستوى التحكم WebSocket.

- **Loopback أولًا**: تستخدم Gateway WS افتراضيًا العنوان `ws://127.0.0.1:18789`.
  تتطلب عمليات الربط غير الخاصة بـ loopback مسار مصادقة صالحًا للـ gateway: مصادقة
  token/password بسر مشترك، أو نشر `trusted-proxy`
  غير loopback ومكوَّن بشكل صحيح.
- يُوصى باستخدام **Gateway واحدة لكل مضيف**. وللعزل، شغّل عدة بوابات مع ملفات تعريف ومنافذ معزولة ([Gateways متعددة](/gateway/multiple-gateways)).
- يتم تقديم **Canvas host** على المنفذ نفسه الخاص بـ Gateway ‏(`/__openclaw__/canvas/` و`/__openclaw__/a2ui/`)، ويكون محميًا بمصادقة Gateway عند الربط خارج loopback.
- يكون **الوصول عن بُعد** عادةً عبر نفق SSH أو VPN من نوع Tailscale ‏([الوصول عن بُعد](/gateway/remote)).

المراجع الأساسية:

- [بنية Gateway](/concepts/architecture)
- [بروتوكول Gateway](/gateway/protocol)
- [دليل تشغيل Gateway](/gateway)
- [أسطح الويب + أوضاع الربط](/web)

## الاقتران + الهوية

- [نظرة عامة على الاقتران (DM + nodes)](/channels/pairing)
- [اقتران العقد المملوك للـ Gateway](/gateway/pairing)
- [CLI الأجهزة (الاقتران + تدوير الرمز)](/cli/devices)
- [CLI الاقتران (الموافقات على الرسائل الخاصة)](/cli/pairing)

الثقة المحلية:

- يمكن الموافقة تلقائيًا على الاتصالات المباشرة عبر local loopback للحفاظ على
  سلاسة تجربة الاستخدام على المضيف نفسه.
- يملك OpenClaw أيضًا مسار اتصال ذاتي ضيقًا ومخصصًا للواجهة الخلفية/الحاوية المحلية
  لتدفقات المساعد الموثوق ذات السر المشترك.
- لا تزال عملاء tailnet وLAN، بما في ذلك روابط tailnet على المضيف نفسه،
  تتطلب موافقة صريحة على الاقتران.

## الاكتشاف + وسائل النقل

- [الاكتشاف ووسائل النقل](/gateway/discovery)
- [Bonjour / mDNS](/gateway/bonjour)
- [الوصول عن بُعد (SSH)](/gateway/remote)
- [Tailscale](/gateway/tailscale)

## Nodes + وسائل النقل

- [نظرة عامة على Nodes](/nodes)
- [بروتوكول Bridge ‏(العقد القديمة، لأغراض تاريخية)](/gateway/bridge-protocol)
- [دليل تشغيل العقدة: iOS](/platforms/ios)
- [دليل تشغيل العقدة: Android](/platforms/android)

## الأمان

- [نظرة عامة على الأمان](/gateway/security)
- [مرجع تكوين Gateway](/gateway/configuration)
- [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting)
- [Doctor](/gateway/doctor)
