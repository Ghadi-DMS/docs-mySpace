---
read_when:
    - عرض واجهة Control UI الخاصة بـ Gateway خارج localhost
    - أتمتة الوصول إلى لوحة التحكم عبر tailnet أو العامة
summary: تكامل Tailscale Serve/Funnel للوحة تحكم Gateway
title: Tailscale
x-i18n:
    generated_at: "2026-04-05T12:44:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4ca5316e804e089c31a78ae882b3082444e082fb2b36b73679ffede20590cb2e
    source_path: gateway/tailscale.md
    workflow: 15
---

# Tailscale (لوحة تحكم Gateway)

يمكن لـ OpenClaw تكوين Tailscale **Serve** (tailnet) أو **Funnel** (عام) تلقائيًا من أجل
لوحة تحكم Gateway ومنفذ WebSocket. وهذا يُبقي Gateway مرتبطة بـ loopback بينما
توفر Tailscale بروتوكول HTTPS، والتوجيه، و(في حالة Serve) رؤوس الهوية.

## الأوضاع

- `serve`: وضع Serve خاص بـ Tailnet فقط عبر `tailscale serve`. تظل gateway على `127.0.0.1`.
- `funnel`: HTTPS عام عبر `tailscale funnel`. ويتطلب OpenClaw كلمة مرور مشتركة.
- `off`: الافتراضي (من دون أتمتة Tailscale).

## المصادقة

اضبط `gateway.auth.mode` للتحكم في handshake:

- `none` (إدخال خاص فقط)
- `token` (الافتراضي عند ضبط `OPENCLAW_GATEWAY_TOKEN`)
- `password` (سر مشترك عبر `OPENCLAW_GATEWAY_PASSWORD` أو التكوين)
- `trusted-proxy` (reverse proxy مدرك للهوية؛ راجع [مصادقة Trusted Proxy](/gateway/trusted-proxy-auth))

عندما يكون `tailscale.mode = "serve"` وتكون `gateway.auth.allowTailscale` مضبوطة على `true`،
يمكن أن تستخدم مصادقة Control UI/WebSocket رؤوس هوية Tailscale
(`tailscale-user-login`) من دون تقديم token/كلمة مرور. ويتحقق OpenClaw
من الهوية عبر حل عنوان `x-forwarded-for` بواسطة daemon المحلي الخاص بـ Tailscale
(`tailscale whois`) ومطابقته مع الرأس قبل قبوله.
ولا يعامل OpenClaw الطلب على أنه Serve إلا عندما يصل من loopback مع
رؤوس Tailscale التالية: `x-forwarded-for` و`x-forwarded-proto` و`x-forwarded-host`.
ولا تستخدم نقاط نهاية HTTP API (مثل `/v1/*` و`/tools/invoke` و`/api/channels/*`)
مصادقة رؤوس هوية Tailscale. فهي ما زالت تتبع
وضع مصادقة HTTP العادي الخاص بالبوابة: مصادقة سر مشترك افتراضيًا، أو إعداد
trusted-proxy / `none` للإدخال الخاص عندما يتم تكوين ذلك عمدًا.
يفترض هذا التدفق بلا token أن مضيف البوابة موثوق. وإذا كان من الممكن تشغيل شيفرة محلية غير موثوقة
على المضيف نفسه، فعطّل `gateway.auth.allowTailscale` واطلب بدلًا من ذلك مصادقة token/password.
ولطلب بيانات اعتماد صريحة تعتمد على سر مشترك، اضبط `gateway.auth.allowTailscale: false`
واستخدم `gateway.auth.mode: "token"` أو `"password"`.

## أمثلة التكوين

### Tailnet فقط (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

الفتح: `https://<magicdns>/` (أو `gateway.controlUi.basePath` المكوّن لديك)

### Tailnet فقط (الربط بعنوان Tailnet IP)

استخدم هذا عندما تريد أن تستمع Gateway مباشرةً على عنوان Tailnet IP (من دون Serve/Funnel).

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

الاتصال من جهاز آخر على Tailnet:

- Control UI: ‏`http://<tailscale-ip>:18789/`
- WebSocket: ‏`ws://<tailscale-ip>:18789`

ملاحظة: لن يعمل loopback (`http://127.0.0.1:18789`) **في هذا الوضع**.

### الإنترنت العام (Funnel + كلمة مرور مشتركة)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

يفضَّل استخدام `OPENCLAW_GATEWAY_PASSWORD` بدلًا من حفظ كلمة مرور على القرص.

## أمثلة CLI

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## ملاحظات

- يتطلب Tailscale Serve/Funnel تثبيت CLI الخاص بـ `tailscale` وتسجيل الدخول فيه.
- يرفض `tailscale.mode: "funnel"` بدء التشغيل ما لم يكن وضع المصادقة هو `password` لتجنب التعرض العام.
- اضبط `gateway.tailscale.resetOnExit` إذا كنت تريد أن يقوم OpenClaw بالتراجع عن إعداد
  `tailscale serve` أو `tailscale funnel` عند الإيقاف.
- `gateway.bind: "tailnet"` هو ربط مباشر بـ Tailnet (من دون HTTPS، ومن دون Serve/Funnel).
- يفضّل `gateway.bind: "auto"` استخدام loopback؛ واستخدم `tailnet` إذا كنت تريد Tailnet فقط.
- يكشف Serve/Funnel فقط **واجهة التحكم + WS الخاصة بـ Gateway**. وتتصل العُقد عبر
  نقطة نهاية Gateway WS نفسها، لذا يمكن أن يعمل Serve أيضًا لوصول العُقد.

## التحكم في browser (Gateway بعيدة + browser محلي)

إذا كنت تشغّل Gateway على جهاز لكنك تريد التحكم في browser على جهاز آخر،
فشغّل **node host** على جهاز browser وأبقِ الاثنين على tailnet نفسها.
ستقوم Gateway بتمرير إجراءات browser إلى العقدة؛ ولا حاجة إلى خادم تحكم منفصل أو URL خاص بـ Serve.

تجنب Funnel للتحكم في browser؛ وتعامل مع pairing الخاص بالعُقدة على أنه وصول للمشغّل.

## المتطلبات المسبقة والقيود الخاصة بـ Tailscale

- يتطلب Serve تمكين HTTPS في tailnet لديك؛ وسيطالبك CLI إذا لم يكن موجودًا.
- يحقن Serve رؤوس هوية Tailscale؛ أما Funnel فلا يفعل ذلك.
- يتطلب Funnel إصدار Tailscale ‏v1.38.3+، وMagicDNS، وتمكين HTTPS، وسمة funnel node.
- لا يدعم Funnel إلا المنافذ `443` و`8443` و`10000` عبر TLS.
- يتطلب Funnel على macOS استخدام نسخة تطبيق Tailscale مفتوحة المصدر.

## تعرّف على المزيد

- نظرة عامة على Tailscale Serve: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- أمر `tailscale serve`: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- نظرة عامة على Tailscale Funnel: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- أمر `tailscale funnel`: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)
