---
read_when:
    - تغيير المصادقة على لوحة التحكم أو أوضاع تعريضها
summary: الوصول إلى لوحة gateway ‏(Control UI) والمصادقة
title: لوحة التحكم
x-i18n:
    generated_at: "2026-04-05T13:00:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 316e082ae4759f710b457487351e30c53b34c7c2b4bf84ad7b091a50538af5cc
    source_path: web/dashboard.md
    workflow: 15
---

# لوحة التحكم (Control UI)

لوحة تحكم Gateway هي Control UI في المتصفح والمقدَّمة افتراضيًا عند `/`
(يمكن تجاوزها بواسطة `gateway.controlUi.basePath`).

فتح سريع (Gateway محلي):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) ‏(أو [http://localhost:18789/](http://localhost:18789/))

مراجع أساسية:

- [Control UI](/web/control-ui) للاستخدام وقدرات الواجهة.
- [Tailscale](/ar/gateway/tailscale) لأتمتة Serve/Funnel.
- [أسطح الويب](/web) لأوضاع الربط وملاحظات الأمان.

تُفرض المصادقة عند مصافحة WebSocket عبر مسار مصادقة gateway المضبوط:

- `connect.params.auth.token`
- `connect.params.auth.password`
- ترويسات هوية Tailscale Serve عندما يكون `gateway.auth.allowTailscale: true`
- ترويسات هوية الوكيل الموثوق عندما يكون `gateway.auth.mode: "trusted-proxy"`

راجع `gateway.auth` في [إعدادات Gateway](/ar/gateway/configuration).

ملاحظة أمنية: تُعد Control UI **سطح إدارة** ‏(دردشة، وإعدادات، وموافقات exec).
لا تعرّضها علنًا. تحتفظ الواجهة برموز URL الخاصة باللوحة في `sessionStorage`
لجلسة علامة تبويب المتصفح الحالية وعنوان gateway المحدد، وتزيلها من عنوان URL بعد التحميل.
فضّل localhost، أو Tailscale Serve، أو نفق SSH.

## المسار السريع (موصى به)

- بعد الإعداد الأولي، يقوم CLI بفتح لوحة التحكم تلقائيًا ويطبع رابطًا نظيفًا (من دون رمز).
- أعد فتحها في أي وقت: `openclaw dashboard` ‏(ينسخ الرابط، ويفتح المتصفح إن أمكن، ويعرض تلميح SSH إذا كان النظام headless).
- إذا طلبت الواجهة مصادقة سرّ مشترك، فألصق الرمز أو
  كلمة المرور المضبوطة في إعدادات Control UI.

## أساسيات المصادقة (محلي مقابل بعيد)

- **localhost**: افتح `http://127.0.0.1:18789/`.
- **مصدر الرمز ذي السرّ المشترك**: ‏`gateway.auth.token` ‏(أو
  `OPENCLAW_GATEWAY_TOKEN`)؛ ويمكن لـ `openclaw dashboard` تمريره عبر مقطع URL
  لتمهيد لمرة واحدة، وتحتفظ به Control UI في `sessionStorage` من أجل
  جلسة علامة التبويب الحالية وعنوان gateway المحدد بدلًا من `localStorage`.
- إذا كان `gateway.auth.token` مُدارًا عبر SecretRef، فإن `openclaw dashboard`
  يطبع/ينسخ/يفتح عنوان URL غير مضمَّن فيه رمز عمدًا. وهذا يتجنب كشف
  الرموز المُدارة خارجيًا في سجلات الصدفة أو محفوظات الحافظة أو
  وسائط تشغيل المتصفح.
- إذا كان `gateway.auth.token` مُعدًا كـ SecretRef ولم يُحل في
  الصدفة الحالية، فإن `openclaw dashboard` لا يزال يطبع عنوان URL غير مضمَّن فيه رمز
  بالإضافة إلى إرشادات عملية لإعداد المصادقة.
- **كلمة مرور ذات سرّ مشترك**: استخدم `gateway.auth.password` المضبوطة (أو
  `OPENCLAW_GATEWAY_PASSWORD`). لا تحتفظ اللوحة بكلمات المرور عبر
  عمليات إعادة التحميل.
- **أوضاع حاملة للهوية**: يمكن لـ Tailscale Serve تلبية مصادقة Control UI/WebSocket
  عبر ترويسات الهوية عندما يكون `gateway.auth.allowTailscale: true`، ويمكن
  لوكيل عكسي غير loopback ومدرك للهوية تلبية المصادقة عند
  `gateway.auth.mode: "trusted-proxy"`. في هذه الأوضاع، لا تحتاج اللوحة إلى
  سرّ مشترك ملصق من أجل WebSocket.
- **ليس localhost**: استخدم Tailscale Serve، أو ربطًا غير loopback بسرّ مشترك، أو
  وكيلًا عكسيًا غير loopback ومدركًا للهوية مع
  `gateway.auth.mode: "trusted-proxy"`، أو نفق SSH. ولا تزال HTTP APIs تستخدم
  مصادقة السرّ المشترك ما لم تشغّل عمدًا وضع
  `gateway.auth.mode: "none"` الخاص بإدخال خاص، أو مصادقة HTTP للوكيل الموثوق. راجع
  [أسطح الويب](/web).

<a id="if-you-see-unauthorized-1008"></a>

## إذا رأيت "unauthorized" / 1008

- تأكد من إمكانية الوصول إلى gateway ‏(محليًا: `openclaw status`؛ وعن بُعد: نفق SSH ‏`ssh -N -L 18789:127.0.0.1:18789 user@host` ثم افتح `http://127.0.0.1:18789/`).
- بالنسبة إلى `AUTH_TOKEN_MISMATCH`، قد ينفذ العملاء إعادة محاولة موثوقة واحدة باستخدام رمز جهاز مخزَّن مؤقتًا عندما يعيد gateway تلميحات إعادة المحاولة. وتعيد إعادة المحاولة بهذا الرمز المخزَّن استخدام النطاقات المعتمدة المخزَّنة للرمز؛ أما المستدعون الصريحون لـ `deviceToken` / `scopes` الصريحة فيحتفظون بمجموعة النطاقات التي طلبوها. وإذا استمرت المصادقة في الفشل بعد تلك الإعادة، فقم بحل انحراف الرمز يدويًا.
- خارج مسار إعادة المحاولة هذا، تكون أولوية مصادقة الاتصال صريحة: الرمز/كلمة المرور المشتركة أولًا، ثم `deviceToken` الصريح، ثم رمز الجهاز المخزَّن، ثم رمز التمهيد.
- على مسار Control UI غير المتزامن في Tailscale Serve، تتم
  سلسلة المحاولات الفاشلة لنفس
  `{scope, ip}` قبل أن يسجّل محدِّد المحاولات الفاشلة فشل المصادقة، لذلك قد تُظهر إعادة المحاولة السيئة الثانية المتزامنة بالفعل عبارة `retry later`.
- لخطوات إصلاح انحراف الرمز، اتبع [قائمة التحقق من استرداد انحراف الرمز](/cli/devices#token-drift-recovery-checklist).
- استخرج أو وفّر السرّ المشترك من مضيف gateway:
  - الرمز: ‏`openclaw config get gateway.auth.token`
  - كلمة المرور: حل `gateway.auth.password` المضبوطة أو
    `OPENCLAW_GATEWAY_PASSWORD`
  - الرمز المُدار عبر SecretRef: حل موفّر الأسرار الخارجي أو صدّر
    `OPENCLAW_GATEWAY_TOKEN` في هذه الصدفة، ثم أعد تشغيل `openclaw dashboard`
  - لا يوجد سرّ مشترك مضبوط: ‏`openclaw doctor --generate-gateway-token`
- في إعدادات اللوحة، ألصق الرمز أو كلمة المرور في حقل المصادقة،
  ثم اتصل.
