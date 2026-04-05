---
read_when:
    - تريد إقران تطبيق عقدة محمول ببوابة بسرعة
    - تحتاج إلى إخراج رمز إعداد للمشاركة عن بُعد أو يدويًا
summary: مرجع CLI للأمر `openclaw qr` (إنشاء QR لإقران الهاتف المحمول + رمز الإعداد)
title: qr
x-i18n:
    generated_at: "2026-04-05T12:39:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: ee6469334ad09037318f938c7ac609b7d5e3385c0988562501bb02a1bfa411ff
    source_path: cli/qr.md
    workflow: 15
---

# `openclaw qr`

أنشئ QR لإقران الهاتف المحمول ورمز إعداد من تكوين البوابة الحالي لديك.

## الاستخدام

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --url wss://gateway.example/ws
```

## الخيارات

- `--remote`: يفضّل `gateway.remote.url`؛ وإذا لم يكن مضبوطًا، فلا يزال بإمكان `gateway.tailscale.mode=serve|funnel` توفير URL العام البعيد
- `--url <url>`: تجاوز URL البوابة المستخدم في الحمولة
- `--public-url <url>`: تجاوز URL العام المستخدم في الحمولة
- `--token <token>`: تجاوز token البوابة الذي يتولى تدفق bootstrap المصادقة مقابله
- `--password <password>`: تجاوز كلمة مرور البوابة التي يتولى تدفق bootstrap المصادقة مقابلها
- `--setup-code-only`: طباعة رمز الإعداد فقط
- `--no-ascii`: تخطي عرض QR بصيغة ASCII
- `--json`: إخراج JSON (`setupCode`, `gatewayUrl`, `auth`, `urlSource`)

## ملاحظات

- `--token` و`--password` متنافيان.
- يحمل رمز الإعداد نفسه الآن `bootstrapToken` قصير العمر ومعتمًا، وليس token/كلمة مرور البوابة المشتركة.
- في تدفق bootstrap المضمّن للعقدة/المشغّل، يظل token العقدة الأساسي يُمنح مع `scopes: []`.
- إذا أصدر bootstrap handoff أيضًا operator token، فإنه يبقى محصورًا في allowlist الخاصة بالـ bootstrap: ‏`operator.approvals` و`operator.read` و`operator.talk.secrets` و`operator.write`.
- تكون فحوصات نطاق bootstrap مسبوقة بالدور. وتلبّي allowlist الخاصة بالمشغّل طلبات المشغّل فقط؛ أما الأدوار غير المشغّلة فما زالت تحتاج إلى scopes تحت بادئة دورها الخاصة.
- يفشل إقران الهاتف المحمول بشكل fail-closed مع URLات بوابة Tailscale/العامة من نوع `ws://`. وما زال `ws://` على شبكات LAN الخاصة مدعومًا، لكن يجب أن تستخدم مسارات الهاتف المحمول عبر Tailscale/العامة Tailscale Serve/Funnel أو URL بوابة من نوع `wss://`.
- مع `--remote`، يتطلب OpenClaw إما `gateway.remote.url` أو
  `gateway.tailscale.mode=serve|funnel`.
- مع `--remote`، إذا كانت بيانات الاعتماد البعيدة النشطة فعليًا مكوّنة كـ SecretRefs ولم تمرر `--token` أو `--password`، فسيقوم الأمر بحلها من snapshot البوابة النشط. وإذا كانت البوابة غير متاحة، يفشل الأمر سريعًا.
- من دون `--remote`، يتم حل SecretRefs الخاصة بمصادقة البوابة المحلية عندما لا يتم تمرير تجاوز مصادقة عبر CLI:
  - يتم حل `gateway.auth.token` عندما يمكن لمصادقة token أن تفوز (وجود `gateway.auth.mode="token"` صريح أو وضع مستنتج حيث لا يفوز أي مصدر كلمة مرور).
  - يتم حل `gateway.auth.password` عندما يمكن لمصادقة كلمة المرور أن تفوز (وجود `gateway.auth.mode="password"` صريح أو وضع مستنتج من دون token فائز من auth/env).
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مكوّنين (بما في ذلك SecretRefs) ولم يتم ضبط `gateway.auth.mode`، فإن حل رمز الإعداد يفشل حتى يتم ضبط الوضع صراحةً.
- ملاحظة حول اختلاف إصدار البوابة: يتطلب مسار هذا الأمر بوابة تدعم `secrets.resolve`؛ وتُرجع البوابات الأقدم خطأ unknown-method.
- بعد المسح، وافق على إقران الجهاز باستخدام:
  - `openclaw devices list`
  - `openclaw devices approve <requestId>`
