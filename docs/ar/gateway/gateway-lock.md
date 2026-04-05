---
read_when:
    - تشغيل عملية gateway أو تصحيح أخطائها
    - التحقيق في فرض المثيل الواحد
summary: حارس التفرد لـ Gateway باستخدام ربط مستمع WebSocket
title: قفل Gateway
x-i18n:
    generated_at: "2026-04-05T12:42:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 726c687ab53f2dd1e46afed8fc791b55310a5c1e62f79a0e38a7dc4ca7576093
    source_path: gateway/gateway-lock.md
    workflow: 15
---

# قفل Gateway

## لماذا

- التأكد من تشغيل مثيل gateway واحد فقط لكل منفذ أساسي على المضيف نفسه؛ ويجب على أي gateway إضافية استخدام ملفات تعريف معزولة ومنافذ فريدة.
- الصمود أمام الأعطال/SIGKILL من دون ترك ملفات قفل قديمة.
- الفشل سريعًا برسالة واضحة عندما يكون منفذ التحكم مشغولًا بالفعل.

## الآلية

- يقوم gateway بربط مستمع WebSocket ‏(الافتراضي `ws://127.0.0.1:18789`) فورًا عند بدء التشغيل باستخدام مستمع TCP حصري.
- إذا فشل الربط مع `EADDRINUSE`، فإن بدء التشغيل يطرح `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- يحرر نظام التشغيل المستمع تلقائيًا عند خروج أي عملية، بما في ذلك الأعطال وSIGKILL — ولا حاجة إلى ملف قفل منفصل أو خطوة تنظيف.
- عند الإيقاف، يغلق gateway خادم WebSocket وخادم HTTP الأساسي لتحرير المنفذ بسرعة.

## سطح الخطأ

- إذا كانت هناك عملية أخرى تحتفظ بالمنفذ، فإن بدء التشغيل يطرح `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- تظهر إخفاقات الربط الأخرى على شكل `GatewayLockError("failed to bind gateway socket on ws://127.0.0.1:<port>: …")`.

## ملاحظات تشغيلية

- إذا كان المنفذ مشغولًا بواسطة عملية _أخرى_، فستكون الرسالة نفسها؛ حرر المنفذ أو اختر منفذًا آخر باستخدام `openclaw gateway --port <port>`.
- ما زال تطبيق macOS يحتفظ بحارس PID خفيف خاص به قبل تشغيل gateway؛ لكن قفل وقت التشغيل يُفرض بواسطة ربط WebSocket.

## ذو صلة

- [Gateways متعددة](/gateway/multiple-gateways) — تشغيل عدة مثيلات بمنافذ فريدة
- [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting) — تشخيص `EADDRINUSE` وتعارضات المنافذ
