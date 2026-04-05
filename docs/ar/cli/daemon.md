---
read_when:
    - ما زلت تستخدم `openclaw daemon ...` في scripts
    - تحتاج إلى أوامر دورة حياة الخدمة (install/start/stop/restart/status)
summary: مرجع CLI للأمر `openclaw daemon` ‏(اسم مستعار قديم لإدارة خدمة gateway)
title: daemon
x-i18n:
    generated_at: "2026-04-05T12:38:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 91fdaf3c4f3e7dd4dff86f9b74a653dcba2674573698cf51efc4890077994169
    source_path: cli/daemon.md
    workflow: 15
---

# `openclaw daemon`

اسم مستعار قديم لأوامر إدارة خدمة Gateway.

يُطابق `openclaw daemon ...` نفس سطح التحكم في الخدمة الموجود في أوامر خدمة `openclaw gateway ...`.

## الاستخدام

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## الأوامر الفرعية

- `status`: إظهار حالة تثبيت الخدمة وفحص سلامة Gateway
- `install`: تثبيت الخدمة (`launchd`/`systemd`/`schtasks`)
- `uninstall`: إزالة الخدمة
- `start`: بدء الخدمة
- `stop`: إيقاف الخدمة
- `restart`: إعادة تشغيل الخدمة

## الخيارات الشائعة

- `status`: ‏`--url` و`--token` و`--password` و`--timeout` و`--no-probe` و`--require-rpc` و`--deep` و`--json`
- `install`: ‏`--port` و`--runtime <node|bun>` و`--token` و`--force` و`--json`
- دورة الحياة (`uninstall|start|stop|restart`): ‏`--json`

ملاحظات:

- يقوم `status` بحل `SecretRefs` الخاصة بالمصادقة والمكوّنة لاستخدامها في مصادقة الفحص عندما يكون ذلك ممكنًا.
- إذا تعذر حل `SecretRef` مطلوب للمصادقة في مسار هذا الأمر، فإن `daemon status --json` يبلغ عن `rpc.authWarning` عندما يفشل اتصال/مصادقة الفحص؛ مرّر `--token`/`--password` صراحةً أو قم بحل مصدر السر أولًا.
- إذا نجح الفحص، فسيتم إخفاء تحذيرات auth-ref غير المحلولة لتجنب النتائج الإيجابية الكاذبة.
- يضيف `status --deep` فحصًا على مستوى النظام بأفضل جهد للخدمة. وعندما يعثر على خدمات أخرى شبيهة بـ gateway، يطبع الخرج المخصص للبشر تلميحات للتنظيف ويحذر من أن التوصية المعتادة ما تزال gateway واحدًا لكل جهاز.
- في تثبيتات Linux systemd، تتضمن فحوصات انحراف الرمز في `status` كلاً من مصدري الوحدة `Environment=` و`EnvironmentFile=`.
- تقوم فحوصات الانحراف بحل `SecretRefs` الخاصة بـ `gateway.auth.token` باستخدام بيئة التشغيل المدمجة (بيئة أوامر الخدمة أولًا، ثم بديل بيئة العملية).
- إذا لم تكن مصادقة الرمز فعالة عمليًا (تعيين صريح لـ `gateway.auth.mode` إلى `password`/`none`/`trusted-proxy`، أو عدم تعيين الوضع عندما يمكن أن تفوز كلمة المرور ولا يوجد مرشح رمز يمكن أن يفوز)، فإن فحوصات انحراف الرمز تتخطى حل رمز التكوين.
- عندما تتطلب مصادقة الرمز وجود رمز وتكون `gateway.auth.token` مُدارة عبر SecretRef، فإن `install` يتحقق من أن SecretRef قابلة للحل لكنه لا يحفظ الرمز المحلول في بيانات تعريف بيئة الخدمة.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكان SecretRef المكوَّن للرمز غير قابل للحل، يفشل التثبيت بشكل مغلق.
- إذا تم تكوين كل من `gateway.auth.token` و`gateway.auth.password` وكان `gateway.auth.mode` غير معيّن، فسيتم حظر التثبيت حتى يتم تعيين الوضع صراحةً.
- إذا كنت تشغّل عمدًا عدة بوابات على مضيف واحد، فاعزل المنافذ والتكوين/الحالة ومساحات العمل؛ راجع [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host).

## المفضّل

استخدم [`openclaw gateway`](/cli/gateway) للحصول على الوثائق والأمثلة الحالية.
