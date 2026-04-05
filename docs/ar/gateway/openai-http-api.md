---
read_when:
    - دمج الأدوات التي تتوقع OpenAI Chat Completions
summary: كشف نقطة نهاية HTTP متوافقة مع OpenAI ‏`/v1/chat/completions` من Gateway
title: OpenAI Chat Completions
x-i18n:
    generated_at: "2026-04-05T12:43:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: c374b2f32ce693a8c752e2b0a2532c5f0299ed280f9a0e97b1a9d73bcec37b95
    source_path: gateway/openai-http-api.md
    workflow: 15
---

# OpenAI Chat Completions (HTTP)

يمكن لـ Gateway في OpenClaw تقديم نقطة نهاية Chat Completions صغيرة متوافقة مع OpenAI.

تكون نقطة النهاية هذه **معطلة افتراضيًا**. فعّلها أولًا في التكوين.

- `POST /v1/chat/completions`
- المنفذ نفسه الخاص بـ Gateway (تعدد إرسال WS + HTTP): ‏`http://<gateway-host>:<port>/v1/chat/completions`

عند تمكين سطح HTTP المتوافق مع OpenAI في Gateway، فإنه يقدّم أيضًا:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/responses`

تحت الغطاء، تُنفَّذ الطلبات كتشغيل وكيل عادي في Gateway (نفس مسار الشيفرة مثل `openclaw agent`)، لذا يطابق التوجيه/الأذونات/التكوين ما هو موجود في Gateway لديك.

## المصادقة

يستخدم تكوين المصادقة في Gateway.

مسارات مصادقة HTTP الشائعة:

- مصادقة السر المشترك (`gateway.auth.mode="token"` أو `"password"`):
  `Authorization: Bearer <token-or-password>`
- مصادقة HTTP موثوقة حاملة للهوية (`gateway.auth.mode="trusted-proxy"`):
  مرّر الطلب عبر proxy واعٍ بالهوية ومكوّن، ودعه يحقن
  ترويسات الهوية المطلوبة
- مصادقة مفتوحة على ingress خاص (`gateway.auth.mode="none"`):
  لا يلزم ترويسة مصادقة

ملاحظات:

- عندما تكون `gateway.auth.mode="token"`، استخدم `gateway.auth.token` (أو `OPENCLAW_GATEWAY_TOKEN`).
- عندما تكون `gateway.auth.mode="password"`، استخدم `gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`).
- عندما تكون `gateway.auth.mode="trusted-proxy"`، يجب أن يأتي طلب HTTP من
  مصدر trusted proxy غير loopback ومكوّن؛ ولا تفي proxies المحلية على المضيف نفسه
  بهذا الوضع.
- إذا كان `gateway.auth.rateLimit` مكوّنًا وحدث عدد كبير جدًا من إخفاقات المصادقة، فستعيد نقطة النهاية `429` مع `Retry-After`.

## الحد الأمني (مهم)

تعامل مع نقطة النهاية هذه باعتبارها سطح **وصول مشغل كامل** لمثيل gateway.

- مصادقة HTTP bearer هنا ليست نموذج نطاق ضيق لكل مستخدم.
- يجب التعامل مع رمز/كلمة مرور Gateway صالحين لهذه النقطة النهائية كما لو كانا بيانات اعتماد مالك/مشغل.
- تمر الطلبات عبر نفس مسار وكيل مستوى التحكم الذي تستخدمه إجراءات المشغل الموثوقة.
- لا يوجد حد أدوات منفصل لغير المالك/لكل مستخدم على نقطة النهاية هذه؛ فبمجرد أن يمرر المستدعي مصادقة Gateway هنا، يعامل OpenClaw هذا المستدعي باعتباره مشغلًا موثوقًا لهذا gateway.
- بالنسبة إلى أوضاع مصادقة السر المشترك (`token` و`password`)، تستعيد نقطة النهاية الإعدادات الافتراضية الكاملة للمشغل حتى إذا أرسل المستدعي ترويسة `x-openclaw-scopes` أضيق.
- أوضاع HTTP الموثوقة الحاملة للهوية (مثل مصادقة trusted proxy أو `gateway.auth.mode="none"`) تحترم `x-openclaw-scopes` عند وجودها، وإلا تعود إلى مجموعة نطاقات المشغل الافتراضية العادية.
- إذا كانت سياسة الوكيل المستهدف تسمح بأدوات حساسة، فيمكن لهذه النقطة النهائية استخدامها.
- أبقِ نقطة النهاية هذه على loopback أو tailnet أو ingress خاص فقط؛ ولا تكشفها مباشرة إلى الإنترنت العام.

مصفوفة المصادقة:

- `gateway.auth.mode="token"` أو `"password"` + `Authorization: Bearer ...`
  - يثبت امتلاك سر المشغل المشترك الخاص بـ gateway
  - يتجاهل `x-openclaw-scopes` الأضيق
  - يستعيد مجموعة نطاقات المشغل الافتراضية الكاملة:
    `operator.admin`, `operator.approvals`, `operator.pairing`,
    `operator.read`, `operator.talk.secrets`, `operator.write`
  - يعامل أدوار الدردشة على نقطة النهاية هذه باعتبارها أدوار مرسل مالك
- أوضاع HTTP الموثوقة الحاملة للهوية (مثل trusted proxy auth أو `gateway.auth.mode="none"` على ingress خاص)
  - تصادق هوية موثوقة خارجية أو حد نشر موثوق
  - تحترم `x-openclaw-scopes` عند وجود الترويسة
  - تعود إلى مجموعة نطاقات المشغل الافتراضية العادية عند غياب الترويسة
  - لا تفقد دلالات المالك إلا عندما يضيّق المستدعي النطاقات صراحةً ويحذف `operator.admin`

راجع [Security](/gateway/security) و[Remote access](/gateway/remote).

## عقد النموذج القائم على الوكيل

يعامل OpenClaw الحقل `model` في OpenAI على أنه **هدف وكيل**، وليس معرّف نموذج مزوّد خام.

- `model: "openclaw"` يوجّه إلى الوكيل الافتراضي المكوّن.
- `model: "openclaw/default"` يوجّه أيضًا إلى الوكيل الافتراضي المكوّن.
- `model: "openclaw/<agentId>"` يوجّه إلى وكيل محدد.

ترويسات طلب اختيارية:

- `x-openclaw-model: <provider/model-or-bare-id>` يتجاوز النموذج الخلفي للوكيل المحدد.
- لا يزال `x-openclaw-agent-id: <agentId>` مدعومًا كتجاوز للتوافق.
- `x-openclaw-session-key: <sessionKey>` يتحكم بالكامل في توجيه الجلسة.
- `x-openclaw-message-channel: <channel>` يضبط سياق قناة دخول اصطناعية للمطالبات والسياسات الواعية بالقنوات.

أسماء مستعارة للتوافق ما زالت مقبولة:

- `model: "openclaw:<agentId>"`
- `model: "agent:<agentId>"`

## تمكين نقطة النهاية

اضبط `gateway.http.endpoints.chatCompletions.enabled` على `true`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

## تعطيل نقطة النهاية

اضبط `gateway.http.endpoints.chatCompletions.enabled` على `false`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: false },
      },
    },
  },
}
```

## سلوك الجلسة

افتراضيًا، تكون نقطة النهاية **عديمة الحالة لكل طلب** (يُولَّد مفتاح جلسة جديد مع كل استدعاء).

إذا تضمّن الطلب سلسلة OpenAI `user`، فسيشتق Gateway مفتاح جلسة ثابتًا منها، بحيث يمكن للمكالمات المتكررة مشاركة جلسة وكيل.

## لماذا يهم هذا السطح

هذه هي مجموعة التوافق الأعلى قيمة للواجهات الأمامية والأدوات المستضافة ذاتيًا:

- تتوقع معظم إعدادات Open WebUI وLobeChat وLibreChat وجود `/v1/models`.
- تتوقع العديد من أنظمة RAG وجود `/v1/embeddings`.
- يمكن عادةً لعملاء دردشة OpenAI الحاليين البدء باستخدام `/v1/chat/completions`.
- يفضّل العملاء الأكثر اعتمادًا على الوكلاء بشكل متزايد `/v1/responses`.

## قائمة النماذج وتوجيه الوكيل

<AccordionGroup>
  <Accordion title="ماذا يعيد `/v1/models`؟">
    قائمة أهداف وكلاء OpenClaw.

    تكون المعرّفات المعادة هي `openclaw` و`openclaw/default` وإدخالات `openclaw/<agentId>`.
    استخدمها مباشرةً كقيم OpenAI `model`.

  </Accordion>
  <Accordion title="هل يسرد `/v1/models` الوكلاء أم الوكلاء الفرعيين؟">
    إنه يسرد أهداف الوكلاء من المستوى الأعلى، وليس نماذج مزوّدات الخلفية ولا الوكلاء الفرعيين.

    يظل الوكلاء الفرعيون طوبولوجيا تنفيذ داخلية. ولا يظهرون كنماذج زائفة.

  </Accordion>
  <Accordion title="لماذا تم تضمين `openclaw/default`؟">
    `openclaw/default` هو الاسم المستعار الثابت للوكيل الافتراضي المكوّن.

    وهذا يعني أن العملاء يمكنهم الاستمرار في استخدام معرّف واحد متوقع حتى لو تغيّر معرّف الوكيل الافتراضي الحقيقي بين البيئات.

  </Accordion>
  <Accordion title="كيف أتجاوز النموذج الخلفي؟">
    استخدم `x-openclaw-model`.

    أمثلة:
    `x-openclaw-model: openai/gpt-5.4`
    `x-openclaw-model: gpt-5.4`

    إذا حذفته، فسيعمل الوكيل المحدد باختياره المعتاد للنموذج المكوّن.

  </Accordion>
  <Accordion title="كيف تنسجم embeddings مع هذا العقد؟">
    يستخدم `/v1/embeddings` معرّفات `model` نفسها القائمة على أهداف الوكلاء.

    استخدم `model: "openclaw/default"` أو `model: "openclaw/<agentId>"`.
    وعندما تحتاج إلى نموذج embedding محدد، أرسله في `x-openclaw-model`.
    ومن دون تلك الترويسة، يمر الطلب إلى إعداد embedding المعتاد للوكيل المحدد.

  </Accordion>
</AccordionGroup>

## البث (SSE)

اضبط `stream: true` لتلقي Server-Sent Events ‏(SSE):

- `Content-Type: text/event-stream`
- يكون كل سطر حدث بصيغة `data: <json>`
- ينتهي البث بـ `data: [DONE]`

## إعداد سريع لـ Open WebUI

لإعداد اتصال Open WebUI أساسي:

- عنوان URL الأساسي: `http://127.0.0.1:18789/v1`
- عنوان URL الأساسي لـ Docker على macOS: ‏`http://host.docker.internal:18789/v1`
- مفتاح API: رمز bearer الخاص بـ Gateway
- النموذج: `openclaw/default`

السلوك المتوقع:

- يجب أن يسرد `GET /v1/models` القيمة `openclaw/default`
- يجب أن يستخدم Open WebUI القيمة `openclaw/default` كمعرّف نموذج الدردشة
- إذا كنت تريد مزوّد/نموذج خلفي محددًا لذلك الوكيل، فاضبط النموذج الافتراضي المعتاد للوكيل أو أرسل `x-openclaw-model`

اختبار سريع:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

إذا أعاد ذلك `openclaw/default`، فيمكن لمعظم إعدادات Open WebUI الاتصال باستخدام عنوان URL الأساسي نفسه والرمز نفسه.

## أمثلة

بدون بث:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"hi"}]
  }'
```

بث:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"hi"}]
  }'
```

سرد النماذج:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

جلب نموذج واحد:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

إنشاء embeddings:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

ملاحظات:

- يعيد `/v1/models` أهداف وكلاء OpenClaw، وليس فهارس مزوّدات خام.
- تكون `openclaw/default` موجودة دائمًا بحيث يعمل معرّف ثابت واحد عبر البيئات.
- تنتمي تجاوزات المزوّد/النموذج الخلفي إلى `x-openclaw-model`، وليس إلى الحقل `model` في OpenAI.
- يدعم `/v1/embeddings` القيمة `input` كسلسلة أو كمصفوفة سلاسل.
