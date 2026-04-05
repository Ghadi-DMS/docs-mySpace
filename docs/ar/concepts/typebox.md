---
read_when:
    - تحديث مخططات البروتوكول أو توليد الشيفرة
summary: مخططات TypeBox بوصفها المصدر الوحيد للحقيقة لبروتوكول gateway
title: TypeBox
x-i18n:
    generated_at: "2026-04-05T12:42:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6f508523998f94d12fbd6ce98d8a7d49fa641913196a4ab7b01f91f83c01c7eb
    source_path: concepts/typebox.md
    workflow: 15
---

# TypeBox بوصفه مصدر الحقيقة للبروتوكول

آخر تحديث: 2026-01-10

TypeBox هي مكتبة مخططات تركز أولًا على TypeScript. نستخدمها لتعريف **بروتوكول Gateway
WebSocket** (المصافحة، والطلب/الاستجابة، وأحداث الخادم). وتدفع هذه المخططات
عمليات **التحقق في وقت التشغيل** و**تصدير JSON Schema** و**توليد شيفرة Swift** لتطبيق
macOS. مصدر واحد للحقيقة؛ وكل شيء آخر مولّد.

إذا كنت تريد سياق البروتوكول على المستوى الأعلى، فابدأ من
[معمارية Gateway](/concepts/architecture).

## النموذج الذهني (30 ثانية)

كل رسالة Gateway WS هي أحد ثلاثة إطارات:

- **طلب**: `{ type: "req", id, method, params }`
- **استجابة**: `{ type: "res", id, ok, payload | error }`
- **حدث**: `{ type: "event", event, payload, seq?, stateVersion? }`

**يجب** أن يكون الإطار الأول طلب `connect`. وبعد ذلك، يمكن للعملاء استدعاء
الأساليب (مثل `health` و`send` و`chat.send`) والاشتراك في
الأحداث (مثل `presence` و`tick` و`agent`).

تدفق الاتصال (الحد الأدنى):

```
العميل                    Gateway
  |---- req:connect -------->|
  |<---- res:hello-ok --------|
  |<---- event:tick ----------|
  |---- req:health ---------->|
  |<---- res:health ----------|
```

الأساليب + الأحداث الشائعة:

| الفئة | أمثلة | ملاحظات |
| ---------- | ---------------------------------------------------------- | ---------------------------------- |
| الأساسية | `connect` و`health` و`status` | يجب أن يكون `connect` أولًا |
| المراسلة | `send` و`agent` و`agent.wait` و`system-event` و`logs.tail` | تحتاج التأثيرات الجانبية إلى `idempotencyKey` |
| الدردشة | `chat.history` و`chat.send` و`chat.abort` | تستخدم WebChat هذه |
| الجلسات | `sessions.list` و`sessions.patch` و`sessions.delete` | إدارة الجلسات |
| الأتمتة | `wake` و`cron.list` و`cron.run` و`cron.runs` | التحكم في wake + cron |
| العقد | `node.list` و`node.invoke` و`node.pair.*` | Gateway WS + إجراءات العقد |
| الأحداث | `tick` و`presence` و`agent` و`chat` و`health` و`shutdown` | دفع من الخادم |

توجد قائمة **الاكتشاف** الإعلانية المعتمدة في
`src/gateway/server-methods-list.ts` ‏(`listGatewayMethods` و`GATEWAY_EVENTS`).

## مكان وجود المخططات

- المصدر: `src/gateway/protocol/schema.ts`
- أدوات التحقق في وقت التشغيل (AJV): `src/gateway/protocol/index.ts`
- سجل الميزات/الاكتشاف المُعلن: `src/gateway/server-methods-list.ts`
- المصافحة على الخادم + توزيع الأساليب: `src/gateway/server.impl.ts`
- عميل العقدة: `src/gateway/client.ts`
- JSON Schema المُولّد: `dist/protocol.schema.json`
- نماذج Swift المُولّدة: `apps/macos/Sources/OpenClawProtocol/GatewayModels.swift`

## المسار الحالي

- `pnpm protocol:gen`
  - يكتب JSON Schema ‏(draft‑07) إلى `dist/protocol.schema.json`
- `pnpm protocol:gen:swift`
  - يولّد نماذج Swift الخاصة بـ gateway
- `pnpm protocol:check`
  - يشغّل كلا المولّدين ويتحقق من أن المخرجات قد تم حفظها في commit

## كيفية استخدام المخططات في وقت التشغيل

- **من جهة الخادم**: يتم التحقق من كل إطار وارد باستخدام AJV. ولا تقبل المصافحة إلا
  طلب `connect` الذي تطابق معلماته `ConnectParams`.
- **من جهة العميل**: يتحقق عميل JS من أطر الأحداث والاستجابة قبل
  استخدامها.
- **اكتشاف الميزات**: يرسل Gateway قائمة محافظة من `features.methods`
  و`features.events` في `hello-ok` من `listGatewayMethods()` و
  `GATEWAY_EVENTS`.
- قائمة الاكتشاف هذه ليست تفريغًا مولدًا لكل مساعد قابل للاستدعاء في
  `coreGatewayHandlers`; فبعض RPCs المساعدة تُنفذ في
  `src/gateway/server-methods/*.ts` من دون تعدادها في قائمة
  الميزات المُعلنة.

## أمثلة على الإطارات

الاتصال (أول رسالة):

```json
{
  "type": "req",
  "id": "c1",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "openclaw-macos",
      "displayName": "macos",
      "version": "1.0.0",
      "platform": "macos 15.1",
      "mode": "ui",
      "instanceId": "A1B2"
    }
  }
}
```

استجابة Hello-ok:

```json
{
  "type": "res",
  "id": "c1",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 3,
    "server": { "version": "dev", "connId": "ws-1" },
    "features": { "methods": ["health"], "events": ["tick"] },
    "snapshot": {
      "presence": [],
      "health": {},
      "stateVersion": { "presence": 0, "health": 0 },
      "uptimeMs": 0
    },
    "policy": { "maxPayload": 1048576, "maxBufferedBytes": 1048576, "tickIntervalMs": 30000 }
  }
}
```

طلب + استجابة:

```json
{ "type": "req", "id": "r1", "method": "health" }
```

```json
{ "type": "res", "id": "r1", "ok": true, "payload": { "ok": true } }
```

حدث:

```json
{ "type": "event", "event": "tick", "payload": { "ts": 1730000000 }, "seq": 12 }
```

## عميل مصغّر (Node.js)

أصغر تدفق مفيد: connect + health.

```ts
import { WebSocket } from "ws";

const ws = new WebSocket("ws://127.0.0.1:18789");

ws.on("open", () => {
  ws.send(
    JSON.stringify({
      type: "req",
      id: "c1",
      method: "connect",
      params: {
        minProtocol: 3,
        maxProtocol: 3,
        client: {
          id: "cli",
          displayName: "example",
          version: "dev",
          platform: "node",
          mode: "cli",
        },
      },
    }),
  );
});

ws.on("message", (data) => {
  const msg = JSON.parse(String(data));
  if (msg.type === "res" && msg.id === "c1" && msg.ok) {
    ws.send(JSON.stringify({ type: "req", id: "h1", method: "health" }));
  }
  if (msg.type === "res" && msg.id === "h1") {
    console.log("health:", msg.payload);
    ws.close();
  }
});
```

## مثال عملي: إضافة أسلوب من البداية إلى النهاية

مثال: إضافة طلب جديد `system.echo` يعيد `{ ok: true, text }`.

1. **المخطط (مصدر الحقيقة)**

أضف إلى `src/gateway/protocol/schema.ts`:

```ts
export const SystemEchoParamsSchema = Type.Object(
  { text: NonEmptyString },
  { additionalProperties: false },
);

export const SystemEchoResultSchema = Type.Object(
  { ok: Type.Boolean(), text: NonEmptyString },
  { additionalProperties: false },
);
```

أضف كليهما إلى `ProtocolSchemas` وصدّر الأنواع:

```ts
  SystemEchoParams: SystemEchoParamsSchema,
  SystemEchoResult: SystemEchoResultSchema,
```

```ts
export type SystemEchoParams = Static<typeof SystemEchoParamsSchema>;
export type SystemEchoResult = Static<typeof SystemEchoResultSchema>;
```

2. **التحقق**

في `src/gateway/protocol/index.ts`، صدّر أداة تحقق AJV:

```ts
export const validateSystemEchoParams = ajv.compile<SystemEchoParams>(SystemEchoParamsSchema);
```

3. **سلوك الخادم**

أضف معالجًا في `src/gateway/server-methods/system.ts`:

```ts
export const systemHandlers: GatewayRequestHandlers = {
  "system.echo": ({ params, respond }) => {
    const text = String(params.text ?? "");
    respond(true, { ok: true, text });
  },
};
```

سجّله في `src/gateway/server-methods.ts` ‏(الذي يدمج بالفعل `systemHandlers`)،
ثم أضف `"system.echo"` إلى مدخل `listGatewayMethods` في
`src/gateway/server-methods-list.ts`.

إذا كان الأسلوب قابلًا للاستدعاء من عميل المشغّل أو عميل العقدة، فصنّفه أيضًا في
`src/gateway/method-scopes.ts` حتى تبقى فرضية النطاق وإعلانات ميزات `hello-ok`
متوافقة.

4. **إعادة التوليد**

```bash
pnpm protocol:check
```

5. **الاختبارات + المستندات**

أضف اختبار خادم في `src/gateway/server.*.test.ts` واذكر الأسلوب في المستندات.

## سلوك توليد شيفرة Swift

يُصدر مولد Swift ما يلي:

- تعداد `GatewayFrame` مع الحالات `req` و`res` و`event` و`unknown`
- هياكل/تعدادات payload مكتوبة بقوة
- قيم `ErrorCode` و`GATEWAY_PROTOCOL_VERSION`

تُحفَظ أنواع الإطارات غير المعروفة كحمولات خام من أجل التوافق المستقبلي.

## الإصدار والتوافق

- يوجد `PROTOCOL_VERSION` في `src/gateway/protocol/schema.ts`.
- يرسل العملاء `minProtocol` و`maxProtocol`؛ ويرفض الخادم عدم التطابق.
- تحتفظ نماذج Swift بأنواع الإطارات غير المعروفة لتجنب كسر العملاء الأقدم.

## أنماط المخططات والاصطلاحات

- تستخدم معظم الكائنات `additionalProperties: false` من أجل حمولات صارمة.
- تُعد `NonEmptyString` القيمة الافتراضية للمعرّفات وأسماء الأساليب/الأحداث.
- يستخدم `GatewayFrame` في المستوى الأعلى **مميّزًا** على `type`.
- تتطلب الأساليب ذات التأثيرات الجانبية عادةً قيمة `idempotencyKey` في المعلمات
  (مثال: `send` و`poll` و`agent` و`chat.send`).
- يقبل `agent` قيمة `internalEvents` الاختيارية من أجل سياق orchestration مُولّد وقت التشغيل
  (على سبيل المثال تسليم إكمال مهمة subagent/cron)؛ تعامل مع ذلك كسطح API داخلي.

## JSON Schema الحي

يوجد JSON Schema المُولّد في المستودع في `dist/protocol.schema.json`. وعادةً ما يكون
الملف الخام المنشور متاحًا على:

- [https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json](https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json)

## عندما تغيّر المخططات

1. حدّث مخططات TypeBox.
2. سجّل الأسلوب/الحدث في `src/gateway/server-methods-list.ts`.
3. حدّث `src/gateway/method-scopes.ts` عندما يحتاج RPC الجديد إلى تصنيف نطاق المشغّل أو
   العقدة.
4. شغّل `pnpm protocol:check`.
5. نفّذ commit للمخطط المُعاد توليده + نماذج Swift.
