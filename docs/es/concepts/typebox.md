---
read_when:
    - Actualización de esquemas del protocolo o generación de código
summary: Esquemas TypeBox como única fuente de verdad para el protocolo del gateway
title: TypeBox
x-i18n:
    generated_at: "2026-04-05T12:41:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6f508523998f94d12fbd6ce98d8a7d49fa641913196a4ab7b01f91f83c01c7eb
    source_path: concepts/typebox.md
    workflow: 15
---

# TypeBox como fuente de verdad del protocolo

Última actualización: 2026-01-10

TypeBox es una biblioteca de esquemas orientada primero a TypeScript. La usamos para definir el **protocolo WebSocket del Gateway** (handshake, solicitud/respuesta, eventos del servidor). Esos esquemas impulsan la **validación en tiempo de ejecución**, la **exportación de JSON Schema** y la **generación de código Swift** para la app de macOS. Una sola fuente de verdad; todo lo demás se genera.

Si quieres el contexto de protocolo de nivel superior, empieza con
[Arquitectura del Gateway](/concepts/architecture).

## Modelo mental (30 segundos)

Cada mensaje WS del Gateway es uno de estos tres marcos:

- **Solicitud**: `{ type: "req", id, method, params }`
- **Respuesta**: `{ type: "res", id, ok, payload | error }`
- **Evento**: `{ type: "event", event, payload, seq?, stateVersion? }`

El primer marco **debe** ser una solicitud `connect`. Después, los clientes pueden llamar
métodos (por ejemplo, `health`, `send`, `chat.send`) y suscribirse a eventos (por ejemplo,
`presence`, `tick`, `agent`).

Flujo de conexión (mínimo):

```
Client                    Gateway
  |---- req:connect -------->|
  |<---- res:hello-ok --------|
  |<---- event:tick ----------|
  |---- req:health ---------->|
  |<---- res:health ----------|
```

Métodos + eventos comunes:

| Categoría   | Ejemplos                                                   | Notas                              |
| ----------- | ---------------------------------------------------------- | ---------------------------------- |
| Núcleo      | `connect`, `health`, `status`                              | `connect` debe ir primero          |
| Mensajería  | `send`, `agent`, `agent.wait`, `system-event`, `logs.tail` | los efectos secundarios necesitan `idempotencyKey` |
| Chat        | `chat.history`, `chat.send`, `chat.abort`                  | WebChat usa estos                  |
| Sesiones    | `sessions.list`, `sessions.patch`, `sessions.delete`       | administración de sesiones         |
| Automatización | `wake`, `cron.list`, `cron.run`, `cron.runs`            | control de wake + cron             |
| Nodos       | `node.list`, `node.invoke`, `node.pair.*`                  | Gateway WS + acciones de nodo      |
| Eventos     | `tick`, `presence`, `agent`, `chat`, `health`, `shutdown`  | envío del servidor                 |

El inventario autoritativo anunciado de **discovery** vive en
`src/gateway/server-methods-list.ts` (`listGatewayMethods`, `GATEWAY_EVENTS`).

## Dónde viven los esquemas

- Origen: `src/gateway/protocol/schema.ts`
- Validadores en tiempo de ejecución (AJV): `src/gateway/protocol/index.ts`
- Registro anunciado de funciones/discovery: `src/gateway/server-methods-list.ts`
- Handshake del servidor + despacho de métodos: `src/gateway/server.impl.ts`
- Cliente de nodo: `src/gateway/client.ts`
- JSON Schema generado: `dist/protocol.schema.json`
- Modelos Swift generados: `apps/macos/Sources/OpenClawProtocol/GatewayModels.swift`

## Canalización actual

- `pnpm protocol:gen`
  - escribe JSON Schema (draft‑07) en `dist/protocol.schema.json`
- `pnpm protocol:gen:swift`
  - genera modelos Swift del gateway
- `pnpm protocol:check`
  - ejecuta ambos generadores y verifica que la salida esté confirmada

## Cómo se usan los esquemas en tiempo de ejecución

- **Lado del servidor**: cada marco entrante se valida con AJV. El handshake solo
  acepta una solicitud `connect` cuyos parámetros coincidan con `ConnectParams`.
- **Lado del cliente**: el cliente JS valida marcos de evento y respuesta antes
  de usarlos.
- **Discovery de funciones**: el Gateway envía una lista conservadora `features.methods`
  y `features.events` en `hello-ok` desde `listGatewayMethods()` y
  `GATEWAY_EVENTS`.
- Esa lista de discovery no es un volcado generado de cada helper invocable en
  `coreGatewayHandlers`; algunos RPC auxiliares están implementados en
  `src/gateway/server-methods/*.ts` sin estar enumerados en la lista anunciada
  de funciones.

## Marcos de ejemplo

Connect (primer mensaje):

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

Respuesta hello-ok:

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

Solicitud + respuesta:

```json
{ "type": "req", "id": "r1", "method": "health" }
```

```json
{ "type": "res", "id": "r1", "ok": true, "payload": { "ok": true } }
```

Evento:

```json
{ "type": "event", "event": "tick", "payload": { "ts": 1730000000 }, "seq": 12 }
```

## Cliente mínimo (Node.js)

El flujo útil más pequeño: connect + health.

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

## Ejemplo trabajado: añadir un método de extremo a extremo

Ejemplo: añade una nueva solicitud `system.echo` que devuelva `{ ok: true, text }`.

1. **Esquema (fuente de verdad)**

Añade a `src/gateway/protocol/schema.ts`:

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

Añade ambos a `ProtocolSchemas` y exporta tipos:

```ts
  SystemEchoParams: SystemEchoParamsSchema,
  SystemEchoResult: SystemEchoResultSchema,
```

```ts
export type SystemEchoParams = Static<typeof SystemEchoParamsSchema>;
export type SystemEchoResult = Static<typeof SystemEchoResultSchema>;
```

2. **Validación**

En `src/gateway/protocol/index.ts`, exporta un validador AJV:

```ts
export const validateSystemEchoParams = ajv.compile<SystemEchoParams>(SystemEchoParamsSchema);
```

3. **Comportamiento del servidor**

Añade un manejador en `src/gateway/server-methods/system.ts`:

```ts
export const systemHandlers: GatewayRequestHandlers = {
  "system.echo": ({ params, respond }) => {
    const text = String(params.text ?? "");
    respond(true, { ok: true, text });
  },
};
```

Regístralo en `src/gateway/server-methods.ts` (ya fusiona `systemHandlers`),
luego añade `"system.echo"` a la entrada de `listGatewayMethods` en
`src/gateway/server-methods-list.ts`.

Si el método puede ser llamado por clientes operador o nodo, clasifícalo también en
`src/gateway/method-scopes.ts` para que la aplicación de alcance y la
publicidad de funciones de `hello-ok` sigan alineadas.

4. **Regenerar**

```bash
pnpm protocol:check
```

5. **Pruebas + documentación**

Añade una prueba de servidor en `src/gateway/server.*.test.ts` y menciona el método en la documentación.

## Comportamiento de la generación de código Swift

El generador de Swift emite:

- `GatewayFrame` enum con casos `req`, `res`, `event` y `unknown`
- Structs/enums de carga útil con tipado fuerte
- Valores `ErrorCode` y `GATEWAY_PROTOCOL_VERSION`

Los tipos de marco desconocidos se conservan como cargas útiles sin procesar para compatibilidad hacia adelante.

## Versionado + compatibilidad

- `PROTOCOL_VERSION` vive en `src/gateway/protocol/schema.ts`.
- Los clientes envían `minProtocol` + `maxProtocol`; el servidor rechaza incompatibilidades.
- Los modelos Swift conservan tipos de marco desconocidos para evitar romper clientes más antiguos.

## Patrones y convenciones de esquemas

- La mayoría de objetos usan `additionalProperties: false` para cargas útiles estrictas.
- `NonEmptyString` es el valor predeterminado para IDs y nombres de método/evento.
- El `GatewayFrame` de nivel superior usa un **discriminador** en `type`.
- Los métodos con efectos secundarios normalmente requieren un `idempotencyKey` en los parámetros
  (ejemplo: `send`, `poll`, `agent`, `chat.send`).
- `agent` acepta `internalEvents` opcionales para contexto de orquestación generado por el entorno de ejecución
  (por ejemplo, transferencia de finalización de tareas de subagente/cron); trátalo como una superficie de API interna.

## JSON Schema activo

El JSON Schema generado está en el repositorio en `dist/protocol.schema.json`. El
archivo raw publicado suele estar disponible en:

- [https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json](https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json)

## Cuando cambies esquemas

1. Actualiza los esquemas TypeBox.
2. Registra el método/evento en `src/gateway/server-methods-list.ts`.
3. Actualiza `src/gateway/method-scopes.ts` cuando el nuevo RPC necesite clasificación de alcance de operador o
   nodo.
4. Ejecuta `pnpm protocol:check`.
5. Confirma el esquema regenerado + los modelos Swift.
