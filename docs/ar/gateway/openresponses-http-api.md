---
read_when:
    - دمج عملاء يتحدثون OpenResponses API
    - أنت تريد مدخلات قائمة على العناصر، أو استدعاءات أدوات من جهة العميل، أو أحداث SSE
summary: كشف نقطة نهاية HTTP متوافقة مع OpenResponses على المسار `/v1/responses` من Gateway
title: OpenResponses API
x-i18n:
    generated_at: "2026-04-05T12:44:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: b3f2905fe45accf2699de8a561d15311720f249f9229d26550c16577428ea8a9
    source_path: gateway/openresponses-http-api.md
    workflow: 15
---

# OpenResponses API (HTTP)

يمكن لـ Gateway في OpenClaw تقديم نقطة نهاية متوافقة مع OpenResponses على `POST /v1/responses`.

تكون نقطة النهاية هذه **معطلة افتراضيًا**. قم بتمكينها أولًا في الإعدادات.

- `POST /v1/responses`
- المنفذ نفسه الخاص بـ Gateway (تعدد WS + HTTP): ‏`http://<gateway-host>:<port>/v1/responses`

في الخلفية، يتم تنفيذ الطلبات كتشغيل وكيل عادي في Gateway (نفس مسار الشيفرة مثل
`openclaw agent`)، لذلك تتطابق آليات التوجيه/الأذونات/الإعدادات مع Gateway لديك.

## المصادقة والأمان والتوجيه

يتطابق السلوك التشغيلي مع [OpenAI Chat Completions](/gateway/openai-http-api):

- استخدم مسار مصادقة HTTP المطابق في Gateway:
  - مصادقة السر المشترك (`gateway.auth.mode="token"` أو `"password"`): ‏`Authorization: Bearer <token-or-password>`
  - مصادقة trusted-proxy (`gateway.auth.mode="trusted-proxy"`): ترويسات proxy مدركة للهوية من مصدر trusted proxy مهيأ وغير loopback
  - مصادقة مفتوحة على private-ingress (`gateway.auth.mode="none"`): من دون ترويسة مصادقة
- تعامل مع نقطة النهاية باعتبارها وصولًا تشغيليًا كاملاً إلى مثيل gateway
- بالنسبة إلى أوضاع مصادقة السر المشترك (`token` و`password`)، تجاهل قيم `x-openclaw-scopes` الأضيق المعلنة في bearer واستعد القيم الافتراضية الكاملة للمشغل
- بالنسبة إلى أوضاع HTTP الموثوقة الحاملة للهوية (مثل trusted proxy auth أو `gateway.auth.mode="none"`)، احترم `x-openclaw-scopes` عند وجودها، وإلا فارجع إلى مجموعة النطاقات الافتراضية العادية للمشغل
- اختر الوكلاء باستخدام `model: "openclaw"`، أو `model: "openclaw/default"`، أو `model: "openclaw/<agentId>"`، أو `x-openclaw-agent-id`
- استخدم `x-openclaw-model` عندما تريد تجاوز نموذج الواجهة الخلفية الخاص بالوكيل المحدد
- استخدم `x-openclaw-session-key` من أجل توجيه صريح للجلسة
- استخدم `x-openclaw-message-channel` عندما تريد سياق قناة دخول اصطناعية غير افتراضية

مصفوفة المصادقة:

- `gateway.auth.mode="token"` أو `"password"` + ‏`Authorization: Bearer ...`
  - يثبت امتلاك السر التشغيلي المشترك للـ gateway
  - يتجاهل `x-openclaw-scopes` الأضيق
  - يستعيد مجموعة النطاقات الافتراضية الكاملة للمشغل:
    `operator.admin` و`operator.approvals` و`operator.pairing`،
    و`operator.read` و`operator.talk.secrets` و`operator.write`
  - يتعامل مع أدوار الدردشة على نقطة النهاية هذه كأدوار مرسل مالك
- أوضاع HTTP الموثوقة الحاملة للهوية (مثل trusted proxy auth، أو `gateway.auth.mode="none"` على private ingress)
  - تحترم `x-openclaw-scopes` عندما تكون الترويسة موجودة
  - ترجع إلى مجموعة النطاقات الافتراضية العادية للمشغل عندما تكون الترويسة غائبة
  - لا تفقد دلالات المالك إلا عندما يضيّق المستدعي النطاقات صراحةً ويحذف `operator.admin`

يمكنك تمكين هذه النقطة أو تعطيلها عبر `gateway.http.endpoints.responses.enabled`.

يتضمن سطح التوافق نفسه أيضًا ما يلي:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`

للحصول على الشرح المرجعي لكيفية توافق نماذج استهداف الوكيل، و`openclaw/default`، وتمرير embeddings، وتجاوزات نموذج الواجهة الخلفية معًا، راجع [OpenAI Chat Completions](/gateway/openai-http-api#agent-first-model-contract) و[قائمة النماذج وتوجيه الوكيل](/gateway/openai-http-api#model-list-and-agent-routing).

## سلوك الجلسة

افتراضيًا تكون نقطة النهاية **بلا حالة لكل طلب** (يتم إنشاء مفتاح جلسة جديد مع كل استدعاء).

إذا تضمّن الطلب سلسلة `user` من OpenResponses، فإن Gateway يشتق مفتاح جلسة ثابتًا
منها، بحيث يمكن للاستدعاءات المتكررة مشاركة جلسة وكيل.

## شكل الطلب (المدعوم)

يتبع الطلب OpenResponses API بمدخل قائم على العناصر. الدعم الحالي:

- `input`: سلسلة نصية أو مصفوفة من كائنات العناصر.
- `instructions`: تُدمج في system prompt.
- `tools`: تعريفات أدوات من جهة العميل (أدوات function).
- `tool_choice`: تصفية أدوات العميل أو اشتراطها.
- `stream`: تمكين بث SSE.
- `max_output_tokens`: حد تقريبي للمخرجات (يعتمد على الموفّر).
- `user`: توجيه ثابت للجلسة.

مقبول حاليًا لكن **يتم تجاهله**:

- `max_tool_calls`
- `reasoning`
- `metadata`
- `store`
- `truncation`

مدعوم:

- `previous_response_id`: يعيد OpenClaw استخدام جلسة الاستجابة السابقة عندما يبقى الطلب ضمن النطاق نفسه للوكيل/المستخدم/الجلسة المطلوبة.

## العناصر (input)

### `message`

الأدوار: `system` و`developer` و`user` و`assistant`.

- يتم إلحاق `system` و`developer` بـ system prompt.
- يصبح أحدث عنصر من `user` أو `function_call_output` هو "الرسالة الحالية".
- يتم تضمين رسائل user/assistant الأقدم كسجل من أجل السياق.

### `function_call_output` (أدوات قائمة على الأدوار)

أرسل نتائج الأدوات مرة أخرى إلى النموذج:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` و`item_reference`

يتم قبولهما من أجل توافق المخطط، لكن يتم تجاهلهما عند بناء prompt.

## الأدوات (أدوات function من جهة العميل)

قدّم الأدوات عبر `tools: [{ type: "function", function: { name, description?, parameters? } }]`.

إذا قرر الوكيل استدعاء أداة، فستعيد الاستجابة عنصر إخراج `function_call`.
بعد ذلك ترسل طلب متابعة مع `function_call_output` لمتابعة الدور.

## الصور (`input_image`)

يدعم مصادر base64 أو URL:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

أنواع MIME المسموح بها (حاليًا): ‏`image/jpeg` و`image/png` و`image/gif` و`image/webp` و`image/heic` و`image/heif`.
الحد الأقصى للحجم (حاليًا): ‏10MB.

## الملفات (`input_file`)

يدعم مصادر base64 أو URL:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

أنواع MIME المسموح بها (حاليًا): ‏`text/plain` و`text/markdown` و`text/html` و`text/csv`،
و`application/json` و`application/pdf`.

الحد الأقصى للحجم (حاليًا): ‏5MB.

السلوك الحالي:

- يتم فك ترميز محتوى الملف وإضافته إلى **system prompt**، وليس إلى رسالة المستخدم،
  بحيث يبقى مؤقتًا (ولا يُحفظ في سجل الجلسة).
- يتم تغليف النص المفكوك من الملف باعتباره **محتوى خارجيًا غير موثوق** قبل إضافته،
  بحيث تُعامل بايتات الملف كبيانات، لا كتعليمات موثوقة.
- تستخدم الكتلة المحقونة علامات حدود صريحة مثل
  `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` /
  `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` وتتضمن
  سطر بيانات وصفية باسم `Source: External`.
- يتعمد مسار إدخال الملفات هذا حذف الشعار الطويل `SECURITY NOTICE:`
  للحفاظ على ميزانية prompt؛ ومع ذلك تبقى علامات الحدود والبيانات الوصفية في مكانها.
- يتم تحليل PDFs بحثًا عن النص أولًا. وإذا تم العثور على قدر قليل من النص،
  تُحوَّل الصفحات الأولى إلى صور نقطية وتمرر إلى النموذج، وتستخدم كتلة الملف المحقونة
  العنصر النائب `[PDF content rendered to images]`.

يستخدم تحليل PDF الإصدار القديم الملائم لـ Node من `pdfjs-dist` (من دون worker). أما
الإصدار الحديث من PDF.js فيتوقع browser workers/DOM globals، لذلك لا يُستخدم في Gateway.

إعدادات الجلب عبر URL الافتراضية:

- `files.allowUrl`: ‏`true`
- `images.allowUrl`: ‏`true`
- `maxUrlParts`: ‏`8` (إجمالي أجزاء `input_file` + `input_image` القائمة على URL لكل طلب)
- تكون الطلبات محمية (حل DNS، وحظر عناوين IP الخاصة، وحدود إعادة التوجيه، والمهلات).
- تدعم قوائم السماح الاختيارية لأسماء المضيفين لكل نوع إدخال (`files.urlAllowlist` و`images.urlAllowlist`).
  - مضيف مطابق تمامًا: `"cdn.example.com"`
  - نطاقات فرعية wildcard: ‏`"*.assets.example.com"` (لا يطابق النطاق الجذر)
  - تعني قوائم السماح الفارغة أو المحذوفة عدم وجود تقييد على أسماء المضيفين.
- لتعطيل الجلب القائم على URL بالكامل، اضبط `files.allowUrl: false` و/أو `images.allowUrl: false`.

## حدود الملفات + الصور (config)

يمكن ضبط القيم الافتراضية ضمن `gateway.http.endpoints.responses`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxBodyBytes: 20000000,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 200000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

القيم الافتراضية عند الحذف:

- `maxBodyBytes`: ‏20MB
- `maxUrlParts`: ‏8
- `files.maxBytes`: ‏5MB
- `files.maxChars`: ‏200k
- `files.maxRedirects`: ‏3
- `files.timeoutMs`: ‏10s
- `files.pdf.maxPages`: ‏4
- `files.pdf.maxPixels`: ‏4,000,000
- `files.pdf.minTextChars`: ‏200
- `images.maxBytes`: ‏10MB
- `images.maxRedirects`: ‏3
- `images.timeoutMs`: ‏10s
- تُقبل مصادر `input_image` من نوع HEIC/HEIF وتُطبَّع إلى JPEG قبل تسليمها إلى الموفّر.

ملاحظة أمنية:

- تُفرض قوائم السماح لعناوين URL قبل الجلب وعلى قفزات إعادة التوجيه.
- لا يؤدي إدراج اسم مضيف في قائمة السماح إلى تجاوز حظر عناوين IP الخاصة/الداخلية.
- بالنسبة إلى البوابات المكشوفة على الإنترنت، طبّق ضوابط خروج الشبكة بالإضافة إلى وسائل الحماية على مستوى التطبيق.
  راجع [الأمان](/gateway/security).

## البث (SSE)

اضبط `stream: true` لتلقي Server-Sent Events ‏(SSE):

- `Content-Type: text/event-stream`
- كل سطر حدث يكون بالشكل `event: <type>` و`data: <json>`
- ينتهي التدفق بـ `data: [DONE]`

أنواع الأحداث المُصدرة حاليًا:

- `response.created`
- `response.in_progress`
- `response.output_item.added`
- `response.content_part.added`
- `response.output_text.delta`
- `response.output_text.done`
- `response.content_part.done`
- `response.output_item.done`
- `response.completed`
- `response.failed` (عند الخطأ)

## الاستخدام

يتم ملء `usage` عندما يبلّغ الموفّر الأساسي عن أعداد الرموز.
يقوم OpenClaw بتطبيع الأسماء المستعارة الشائعة بأسلوب OpenAI قبل أن تصل هذه العدادات
إلى أسطح الحالة/الجلسة اللاحقة، بما في ذلك `input_tokens` / `output_tokens`
و`prompt_tokens` / `completion_tokens`.

## الأخطاء

تستخدم الأخطاء كائن JSON بالشكل التالي:

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

الحالات الشائعة:

- `401` مصادقة مفقودة/غير صالحة
- `400` جسم طلب غير صالح
- `405` طريقة غير صحيحة

## أمثلة

من دون بث:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

مع البث:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```
