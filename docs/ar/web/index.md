---
read_when:
    - تريد الوصول إلى Gateway عبر Tailscale
    - تريد واجهة Control UI في المتصفح وتحرير التهيئة
summary: 'واجهات الويب الخاصة بـ Gateway: واجهة Control UI، وأوضاع الربط، والأمان'
title: الويب
x-i18n:
    generated_at: "2026-04-05T13:00:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 15f5643283f7d37235d3d8104897f38db27ac5a9fdef6165156fb542d0e7048c
    source_path: web/index.md
    workflow: 15
---

# الويب (Gateway)

يقدّم Gateway **واجهة Control UI صغيرة في المتصفح** (Vite + Lit) من المنفذ نفسه الذي يستخدمه WebSocket الخاص بـ Gateway:

- الافتراضي: `http://<host>:18789/`
- بادئة اختيارية: عيّن `gateway.controlUi.basePath` (مثل `/openclaw`)

توجد الإمكانات في [Control UI](/web/control-ui).
تركّز هذه الصفحة على أوضاع الربط، والأمان، والواجهات المواجهة للويب.

## Webhooks

عندما تكون `hooks.enabled=true`، يكشف Gateway أيضًا عن نقطة نهاية webhook صغيرة على خادم HTTP نفسه.
راجع [تهيئة Gateway](/ar/gateway/configuration) ← `hooks` للحصول على تفاصيل المصادقة + الحمولات.

## التهيئة (مفعّلة افتراضيًا)

تكون **Control UI مفعّلة افتراضيًا** عند وجود الأصول (`dist/control-ui`).
يمكنك التحكم فيها عبر التهيئة:

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath optional
  },
}
```

## الوصول عبر Tailscale

### Serve المدمج (موصى به)

أبقِ Gateway على local loopback ودع Tailscale Serve يمرّره كوكيل:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

ثم ابدأ gateway:

```bash
openclaw gateway
```

افتح:

- `https://<magicdns>/` (أو قيمة `gateway.controlUi.basePath` التي هيأتها)

### الربط مع tailnet + token

```json5
{
  gateway: {
    bind: "tailnet",
    controlUi: { enabled: true },
    auth: { mode: "token", token: "your-token" },
  },
}
```

ثم ابدأ gateway (يستخدم هذا المثال غير المعتمد على local loopback
مصادقة token بالسر المشترك):

```bash
openclaw gateway
```

افتح:

- `http://<tailscale-ip>:18789/` (أو قيمة `gateway.controlUi.basePath` التي هيأتها)

### الإنترنت العام (Funnel)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password" }, // or OPENCLAW_GATEWAY_PASSWORD
  },
}
```

## ملاحظات الأمان

- تكون مصادقة Gateway مطلوبة افتراضيًا (token، أو password، أو trusted-proxy، أو رؤوس هوية Tailscale Serve عند تمكينها).
- لا تزال عمليات الربط غير المعتمدة على local loopback **تتطلب** مصادقة gateway. عمليًا، يعني ذلك مصادقة token/password أو reverse proxy مدركًا للهوية مع `gateway.auth.mode: "trusted-proxy"`.
- ينشئ المعالج مصادقة بالسر المشترك افتراضيًا، وعادةً ما يولّد
  token خاصًا بـ gateway (حتى على local loopback).
- في وضع السر المشترك، ترسل الواجهة `connect.params.auth.token` أو
  `connect.params.auth.password`.
- في الأوضاع الحاملة للهوية مثل Tailscale Serve أو `trusted-proxy`، يتم
  استيفاء فحص مصادقة WebSocket من رؤوس الطلب بدلًا من ذلك.
- بالنسبة إلى عمليات نشر Control UI غير المعتمدة على local loopback، عيّن `gateway.controlUi.allowedOrigins`
  صراحةً (كأصول كاملة). وبدون ذلك، يُرفض بدء تشغيل gateway افتراضيًا.
- يفعّل `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
  وضع الرجوع إلى أصل Host-header، لكنه يُعد خفضًا خطيرًا في مستوى الأمان.
- مع Serve، يمكن لرؤوس هوية Tailscale استيفاء مصادقة Control UI/WebSocket
  عندما تكون `gateway.auth.allowTailscale` مساوية لـ `true` (ولا حاجة إلى token/password).
  أما نقاط نهاية HTTP API فلا تستخدم رؤوس هوية Tailscale تلك؛ بل تتبع
  وضع مصادقة HTTP العادي الخاص بـ gateway بدلًا من ذلك. عيّن
  `gateway.auth.allowTailscale: false` لفرض بيانات اعتماد صريحة. راجع
  [Tailscale](/ar/gateway/tailscale) و[الأمان](/ar/gateway/security). يفترض
  هذا التدفق من دون token أن مضيف gateway موثوق.
- تتطلب `gateway.tailscale.mode: "funnel"` القيمة `gateway.auth.mode: "password"` (كلمة مرور مشتركة).

## بناء الواجهة

يقدّم Gateway الملفات الثابتة من `dist/control-ui`. ابنِها باستخدام:

```bash
pnpm ui:build # auto-installs UI deps on first run
```
