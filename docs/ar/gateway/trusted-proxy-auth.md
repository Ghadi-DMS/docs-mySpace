---
read_when:
    - تشغّل OpenClaw خلف proxy مدرك للهوية
    - تقوم بإعداد Pomerium أو Caddy أو nginx مع OAuth أمام OpenClaw
    - تعالج أخطاء WebSocket 1008 unauthorized في إعدادات reverse proxy
    - تقرر أين تضبط HSTS وغيرها من رؤوس تقوية HTTP
summary: فوّض مصادقة البوابة إلى reverse proxy موثوق (Pomerium، وCaddy، وnginx + OAuth)
title: مصادقة Trusted Proxy
x-i18n:
    generated_at: "2026-04-05T12:45:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: ccd39736b43e8744de31566d5597b3fbf40ecb6ba9c8ba9d2343e1ab9bb8cd45
    source_path: gateway/trusted-proxy-auth.md
    workflow: 15
---

# مصادقة Trusted Proxy

> ⚠️ **ميزة حساسة أمنيًا.** يفوّض هذا الوضع المصادقة بالكامل إلى reverse proxy الخاص بك. قد يؤدي سوء التكوين إلى تعريض البوابة لوصول غير مصرّح به. اقرأ هذه الصفحة بعناية قبل التمكين.

## متى تستخدمها

استخدم وضع المصادقة `trusted-proxy` عندما:

- تشغّل OpenClaw خلف **proxy مدرك للهوية** (Pomerium، أو Caddy + OAuth، أو nginx + oauth2-proxy، أو Traefik + forward auth)
- يتولى proxy جميع عمليات المصادقة ويمرر هوية المستخدم عبر الرؤوس
- تكون في بيئة Kubernetes أو حاويات حيث يكون proxy هو المسار الوحيد إلى البوابة
- تواجه أخطاء WebSocket `1008 unauthorized` لأن المتصفحات لا تستطيع تمرير tokens في حمولة WS

## متى لا تستخدمها

- إذا كان proxy لديك لا يصادق المستخدمين (مجرد منهي TLS أو موازن حمل)
- إذا كان هناك أي مسار إلى البوابة يتجاوز proxy (ثغرات في الجدار الناري، أو وصول من الشبكة الداخلية)
- إذا لم تكن متأكدًا مما إذا كان proxy لديك يزيل/يستبدل الرؤوس المعاد توجيهها بشكل صحيح
- إذا كنت تحتاج فقط إلى وصول شخصي لمستخدم واحد (فكّر في Tailscale Serve + loopback لإعداد أبسط)

## كيف تعمل

1. يصادق reverse proxy المستخدمين (OAuth، أو OIDC، أو SAML، وما إلى ذلك)
2. يضيف proxy رأسًا يحتوي على هوية المستخدم المصادق عليه (مثل `x-forwarded-user: nick@example.com`)
3. يتحقق OpenClaw من أن الطلب جاء من **عنوان IP خاص بـ proxy موثوق** (مُكوَّن في `gateway.trustedProxies`)
4. يستخرج OpenClaw هوية المستخدم من الرأس المكوَّن
5. إذا كان كل شيء صحيحًا، يُصرَّح بالطلب

## سلوك الإقران في Control UI

عندما يكون `gateway.auth.mode = "trusted-proxy"` نشطًا ويجتاز الطلب
فحوصات trusted-proxy، يمكن لجلسات WebSocket الخاصة بـ Control UI الاتصال من دون
هوية إقران جهاز.

الآثار المترتبة:

- لم يعد الإقران هو البوابة الأساسية للوصول إلى Control UI في هذا الوضع.
- تصبح سياسة مصادقة reverse proxy و`allowUsers` هي التحكم الفعلي في الوصول.
- أبقِ دخول البوابة مقصورًا على عناوين IP الخاصة بالـ proxy الموثوق فقط (`gateway.trustedProxies` + الجدار الناري).

## التكوين

```json5
{
  gateway: {
    // تتوقع مصادقة trusted-proxy طلبات من مصدر trusted proxy غير loopback
    bind: "lan",

    // مهم جدًا: أضف هنا فقط عناوين IP الخاصة بالـ proxy
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // الرأس الذي يحتوي على هوية المستخدم المصادق عليه (مطلوب)
        userHeader: "x-forwarded-user",

        // اختياري: رؤوس يجب أن تكون موجودة (التحقق من proxy)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // اختياري: القصر على مستخدمين محددين (فارغ = السماح للجميع)
        allowUsers: ["nick@example.com", "admin@company.org"],
      },
    },
  },
}
```

قاعدة مهمة في وقت التشغيل:

- ترفض مصادقة trusted-proxy الطلبات القادمة من مصادر loopback (`127.0.0.1` و`::1` ونطاقات loopback CIDR).
- لا تستوفي reverse proxies المحلية على المضيف نفسه عبر loopback متطلبات مصادقة trusted-proxy.
- في إعدادات proxy المحلية على المضيف نفسه عبر loopback، استخدم مصادقة token/password بدلًا من ذلك، أو مرّر الحركة عبر عنوان trusted proxy غير loopback يمكن لـ OpenClaw التحقق منه.
- ما زالت عمليات نشر Control UI غير المعتمدة على loopback تحتاج إلى `gateway.controlUi.allowedOrigins` صريحة.

### مرجع التكوين

| الحقل                                       | مطلوب | الوصف                                                                         |
| ------------------------------------------- | ------ | ----------------------------------------------------------------------------- |
| `gateway.trustedProxies`                    | نعم    | مصفوفة بعناوين IP الخاصة بالـ proxy الموثوق بها. تُرفض الطلبات من العناوين الأخرى. |
| `gateway.auth.mode`                         | نعم    | يجب أن تكون `"trusted-proxy"`                                                |
| `gateway.auth.trustedProxy.userHeader`      | نعم    | اسم الرأس الذي يحتوي على هوية المستخدم المصادق عليه                          |
| `gateway.auth.trustedProxy.requiredHeaders` | لا     | رؤوس إضافية يجب أن تكون موجودة حتى يُعد الطلب موثوقًا                        |
| `gateway.auth.trustedProxy.allowUsers`      | لا     | allowlist لهويات المستخدمين. والفارغ يعني السماح لجميع المستخدمين المصادق عليهم. |

## إنهاء TLS وHSTS

استخدم نقطة إنهاء TLS واحدة وطبّق HSTS هناك.

### النمط الموصى به: إنهاء TLS في proxy

عندما يتولى reverse proxy التعامل مع HTTPS للنطاق `https://control.example.com`، اضبط
`Strict-Transport-Security` في proxy لذلك النطاق.

- مناسب لعمليات النشر المواجهة للإنترنت.
- يبقي الشهادة وسياسة تقوية HTTP في مكان واحد.
- يمكن أن يبقى OpenClaw على HTTP عبر loopback خلف proxy.

مثال على قيمة الرأس:

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### إنهاء TLS في Gateway

إذا كانت OpenClaw نفسها تقدم HTTPS مباشرة (من دون proxy ينهي TLS)، فاضبط:

```json5
{
  gateway: {
    tls: { enabled: true },
    http: {
      securityHeaders: {
        strictTransportSecurity: "max-age=31536000; includeSubDomains",
      },
    },
  },
}
```

يقبل `strictTransportSecurity` قيمة رأس كسلسلة نصية، أو `false` لتعطيله صراحةً.

### إرشادات النشر التدريجي

- ابدأ أولًا بقيمة max age قصيرة (مثل `max-age=300`) أثناء التحقق من الحركة.
- زدها إلى قيم طويلة العمر (مثل `max-age=31536000`) فقط بعد ارتفاع الثقة.
- أضف `includeSubDomains` فقط إذا كانت جميع النطاقات الفرعية جاهزة لـ HTTPS.
- استخدم preload فقط إذا كنت تستوفي عمدًا متطلبات preload لمجموعة نطاقاتك كاملة.
- لا تستفيد بيئات التطوير المحلية المعتمدة على loopback فقط من HSTS.

## أمثلة إعداد proxy

### Pomerium

يمرر Pomerium الهوية في `x-pomerium-claim-email` (أو رؤوس مطالبات أخرى) وJWT في `x-pomerium-jwt-assertion`.

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["10.0.0.1"], // عنوان IP الخاص بـ Pomerium
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-pomerium-claim-email",
        requiredHeaders: ["x-pomerium-jwt-assertion"],
      },
    },
  },
}
```

مقتطف من تكوين Pomerium:

```yaml
routes:
  - from: https://openclaw.example.com
    to: http://openclaw-gateway:18789
    policy:
      - allow:
          or:
            - email:
                is: nick@example.com
    pass_identity_headers: true
```

### Caddy مع OAuth

يمكن لـ Caddy مع plugin ‏`caddy-security` مصادقة المستخدمين وتمرير رؤوس الهوية.

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["10.0.0.1"], // عنوان IP الخاص بـ Caddy/sidecar proxy
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
      },
    },
  },
}
```

مقتطف من Caddyfile:

```
openclaw.example.com {
    authenticate with oauth2_provider
    authorize with policy1

    reverse_proxy openclaw:18789 {
        header_up X-Forwarded-User {http.auth.user.email}
    }
}
```

### nginx + oauth2-proxy

يقوم oauth2-proxy بمصادقة المستخدمين ويمرر الهوية في `x-auth-request-email`.

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["10.0.0.1"], // عنوان IP الخاص بـ nginx/oauth2-proxy
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-auth-request-email",
      },
    },
  },
}
```

مقتطف من تكوين nginx:

```nginx
location / {
    auth_request /oauth2/auth;
    auth_request_set $user $upstream_http_x_auth_request_email;

    proxy_pass http://openclaw:18789;
    proxy_set_header X-Auth-Request-Email $user;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

### Traefik مع Forward Auth

```json5
{
  gateway: {
    bind: "lan",
    trustedProxies: ["172.17.0.1"], // عنوان IP لحاوية Traefik
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
      },
    },
  },
}
```

## تكوين token مختلط

يرفض OpenClaw التكوينات الملتبسة التي يكون فيها كل من `gateway.auth.token` (أو `OPENCLAW_GATEWAY_TOKEN`) ووضع `trusted-proxy` نشطين في الوقت نفسه. قد تتسبب تكوينات token المختلطة في أن تصادق طلبات loopback بصمت عبر مسار المصادقة الخاطئ.

إذا رأيت خطأ `mixed_trusted_proxy_token` عند بدء التشغيل:

- أزل token المشترك عند استخدام وضع trusted-proxy، أو
- غيّر `gateway.auth.mode` إلى `"token"` إذا كنت تقصد مصادقة قائمة على token.

كما أن مصادقة trusted-proxy عبر loopback تفشل بشكل fail-closed: يجب على المستدعين على المضيف نفسه تقديم رؤوس الهوية المكوّنة عبر trusted proxy بدلًا من أن تتم مصادقتهم بصمت.

## رأس نطاقات المشغّل

مصادقة trusted-proxy هي وضع HTTP **يحمل هوية**، لذا يمكن للمستدعين
اختياريًا الإعلان عن نطاقات المشغّل باستخدام `x-openclaw-scopes`.

أمثلة:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

السلوك:

- عند وجود الرأس، يحترم OpenClaw مجموعة النطاقات المعلنة.
- عند وجود الرأس لكن بقيمة فارغة، يعلن الطلب عن **عدم** وجود نطاقات مشغّل.
- عند غياب الرأس، تعود واجهات HTTP التي تحمل الهوية إلى مجموعة النطاقات الافتراضية القياسية للمشغّل.
- تكون **مسارات HTTP الخاصة بـ plugin** والمصادق عليها عبر البوابة أضيق افتراضيًا: عند غياب `x-openclaw-scopes`، يعود نطاق وقت التشغيل فيها إلى `operator.write`.
- ما زال يجب على طلبات HTTP القادمة من المتصفح اجتياز `gateway.controlUi.allowedOrigins` (أو وضع الرجوع المقصود إلى رأس Host) حتى بعد نجاح مصادقة trusted-proxy.

قاعدة عملية:

- أرسل `x-openclaw-scopes` صراحةً عندما تريد أن يكون طلب trusted-proxy
  أضيق من القيم الافتراضية، أو عندما يحتاج مسار plugin مصادق عليه عبر البوابة
  إلى صلاحيات أقوى من نطاق الكتابة.

## قائمة التحقق الأمنية

قبل تمكين مصادقة trusted-proxy، تحقق من الآتي:

- [ ] **الـ proxy هو المسار الوحيد**: منفذ البوابة محمي بجدار ناري ضد كل شيء باستثناء الـ proxy
- [ ] **الحد الأدنى من trustedProxies**: فقط عناوين IP الفعلية للـ proxy، وليس شبكات فرعية كاملة
- [ ] **لا يوجد مصدر proxy من loopback**: تفشل مصادقة trusted-proxy بشكل fail-closed للطلبات القادمة من loopback
- [ ] **الـ proxy يزيل الرؤوس**: يقوم proxy لديك باستبدال رؤوس `x-forwarded-*` القادمة من العملاء (لا إضافتها)
- [ ] **إنهاء TLS**: يتولى proxy لديك TLS؛ ويتصل المستخدمون عبر HTTPS
- [ ] **allowedOrigins صريحة**: تستخدم عمليات نشر Control UI غير المعتمدة على loopback قيمة `gateway.controlUi.allowedOrigins` صريحة
- [ ] **تم ضبط allowUsers** (موصى به): قيّد الوصول إلى مستخدمين معروفين بدلًا من السماح لأي مستخدم مصادق عليه
- [ ] **لا يوجد تكوين token مختلط**: لا تضبط كلًا من `gateway.auth.token` و`gateway.auth.mode: "trusted-proxy"`

## التدقيق الأمني

سيضع `openclaw security audit` علامة على مصادقة trusted-proxy باعتبارها نتيجة ذات شدة **حرجة**. وهذا مقصود — فهو تذكير بأنك تفوض الأمان إلى إعداد proxy لديك.

يتحقق التدقيق من:

- تحذير/تذكير أساسي `gateway.trusted_proxy_auth` بدرجة warning/critical
- غياب تكوين `trustedProxies`
- غياب تكوين `userHeader`
- `allowUsers` فارغة (يسمح لأي مستخدم مصادق عليه)
- سياسة أصل متصفح شاملة أو مفقودة على أسطح Control UI المكشوفة

## استكشاف الأخطاء وإصلاحها

### `trusted_proxy_untrusted_source`

لم يأتِ الطلب من عنوان IP موجود في `gateway.trustedProxies`. تحقق من:

- هل عنوان IP الخاص بالـ proxy صحيح؟ (قد تتغير عناوين IP الخاصة بحاويات Docker)
- هل يوجد موازن حمل أمام الـ proxy؟
- استخدم `docker inspect` أو `kubectl get pods -o wide` لمعرفة عناوين IP الفعلية

### `trusted_proxy_loopback_source`

رفض OpenClaw طلب trusted-proxy قادمًا من loopback.

تحقق من:

- هل يتصل proxy من `127.0.0.1` / `::1`؟
- هل تحاول استخدام مصادقة trusted-proxy مع reverse proxy محلي على المضيف نفسه عبر loopback؟

الإصلاح:

- استخدم مصادقة token/password في إعدادات proxy المحلية على المضيف نفسه عبر loopback، أو
- مرّر الحركة عبر عنوان trusted proxy غير loopback وأبقِ عنوان IP هذا في `gateway.trustedProxies`.

### `trusted_proxy_user_missing`

كان رأس المستخدم فارغًا أو مفقودًا. تحقق من:

- هل تم تكوين proxy لتمرير رؤوس الهوية؟
- هل اسم الرأس صحيح؟ (غير حساس لحالة الأحرف، لكن التهجئة مهمة)
- هل المستخدم مصادق عليه فعليًا عند proxy؟

### `trusted*proxy_missing_header*\*`

أحد الرؤوس المطلوبة لم يكن موجودًا. تحقق من:

- تكوين proxy لديك لتلك الرؤوس المحددة
- وما إذا كانت الرؤوس تُزال في مكان ما ضمن السلسلة

### `trusted_proxy_user_not_allowed`

المستخدم مصادق عليه لكنه غير موجود في `allowUsers`. أضفه أو أزل allowlist.

### `trusted_proxy_origin_not_allowed`

نجحت مصادقة trusted-proxy، لكن رأس `Origin` الخاص بالمتصفح لم يجتز فحوصات أصل Control UI.

تحقق من:

- أن `gateway.controlUi.allowedOrigins` تتضمن أصل المتصفح الدقيق
- أنك لا تعتمد على أصول عامة شاملة إلا إذا كنت تريد عمدًا سلوك السماح للجميع
- إذا كنت تستخدم عمدًا وضع الرجوع إلى رأس Host، فتأكد من ضبط `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` عن قصد

### ما زال WebSocket يفشل

تأكد من أن proxy لديك:

- يدعم ترقيات WebSocket (`Upgrade: websocket` و`Connection: upgrade`)
- يمرر رؤوس الهوية في طلبات ترقية WebSocket (وليس HTTP فقط)
- لا يملك مسار مصادقة منفصلًا لاتصالات WebSocket

## الترحيل من مصادقة token

إذا كنت تنتقل من مصادقة token إلى trusted-proxy:

1. كوّن proxy لديك لمصادقة المستخدمين وتمرير الرؤوس
2. اختبر إعداد proxy بشكل مستقل (باستخدام curl مع الرؤوس)
3. حدّث تكوين OpenClaw بمصادقة trusted-proxy
4. أعد تشغيل البوابة
5. اختبر اتصالات WebSocket من Control UI
6. شغّل `openclaw security audit` وراجع النتائج

## ذو صلة

- [الأمان](/gateway/security) — دليل الأمان الكامل
- [التكوين](/gateway/configuration) — مرجع التكوين
- [الوصول عن بُعد](/gateway/remote) — أنماط أخرى للوصول عن بُعد
- [Tailscale](/gateway/tailscale) — بديل أبسط للوصول عبر tailnet فقط
