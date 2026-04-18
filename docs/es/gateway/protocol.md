---
read_when:
    - Implementación o actualización de clientes WS de Gateway
    - Depuración de discrepancias de protocolo o fallos de conexión
    - Regeneración del esquema/modelos del protocolo
summary: 'Protocolo WebSocket de Gateway: handshake, tramas, control de versiones'
title: Protocolo de Gateway
x-i18n:
    generated_at: "2026-04-18T04:59:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f0eebcfdd8c926c90b4753a6d96c59e3134ddb91740f65478f11eb75be85e41
    source_path: gateway/protocol.md
    workflow: 15
---

# Protocolo de Gateway (WebSocket)

El protocolo WS de Gateway es el **único plano de control + transporte de nodos** para
OpenClaw. Todos los clientes (CLI, interfaz web, app de macOS, nodos iOS/Android,
nodos sin interfaz) se conectan por WebSocket y declaran su **rol** + **alcance** en el
momento del handshake.

## Transporte

- WebSocket, tramas de texto con cargas útiles JSON.
- La primera trama **debe** ser una solicitud `connect`.

## Handshake (`connect`)

Gateway → Cliente (desafío previo a la conexión):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Cliente → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → Cliente:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 3,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`, `features`, `snapshot` y `policy` son obligatorios según el esquema
(`src/gateway/protocol/schema/frames.ts`). `canvasHostUrl` es opcional. `auth`
informa el rol/alcances negociados cuando están disponibles, e incluye `deviceToken`
cuando el gateway emite uno.

Cuando no se emite ningún token de dispositivo, `hello-ok.auth` todavía puede informar los
permisos negociados:

```json
{
  "auth": {
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Cuando se emite un token de dispositivo, `hello-ok` también incluye:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Durante la transferencia de arranque confiable, `hello-ok.auth` también puede incluir entradas de rol
adicionales acotadas en `deviceTokens`:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Para el flujo de arranque integrado de nodo/operador, el token principal del nodo sigue siendo
`scopes: []` y cualquier token de operador transferido sigue acotado a la lista de permisos del operador
de arranque (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Las comprobaciones de alcance de arranque siguen con prefijo de rol:
las entradas de operador solo satisfacen solicitudes de operador, y los roles que no son de operador
siguen necesitando alcances bajo el prefijo de su propio rol.

### Ejemplo de nodo

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## Enmarcado

- **Solicitud**: `{type:"req", id, method, params}`
- **Respuesta**: `{type:"res", id, ok, payload|error}`
- **Evento**: `{type:"event", event, payload, seq?, stateVersion?}`

Los métodos con efectos secundarios requieren **claves de idempotencia** (ver esquema).

## Roles + alcances

### Roles

- `operator` = cliente del plano de control (CLI/UI/automatización).
- `node` = host de capacidades (`camera`/`screen`/`canvas`/`system.run`).

### Alcances (`operator`)

Alcances comunes:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` con `includeSecrets: true` requiere `operator.talk.secrets`
(o `operator.admin`).

Los métodos RPC de Gateway registrados por Plugin pueden solicitar su propio alcance de operador, pero
los prefijos administrativos centrales reservados (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre se resuelven como `operator.admin`.

El alcance del método es solo el primer filtro. Algunos comandos slash alcanzados mediante
`chat.send` aplican comprobaciones más estrictas a nivel de comando además de eso. Por ejemplo, las escrituras
persistentes de `/config set` y `/config unset` requieren `operator.admin`.

`node.pair.approve` también tiene una comprobación adicional de alcance en el momento de la aprobación además del
alcance base del método:

- solicitudes sin comando: `operator.pairing`
- solicitudes con comandos de nodo que no son `exec`: `operator.pairing` + `operator.write`
- solicitudes que incluyen `system.run`, `system.run.prepare` o `system.which`:
  `operator.pairing` + `operator.admin`

### `caps`/`commands`/`permissions` (`node`)

Los nodos declaran reclamaciones de capacidad en el momento de conectarse:

- `caps`: categorías de capacidad de alto nivel.
- `commands`: lista permitida de comandos para invocación.
- `permissions`: controles granulares (por ejemplo, `screen.record`, `camera.capture`).

Gateway los trata como **reclamaciones** y aplica listas permitidas del lado del servidor.

## Presencia

- `system-presence` devuelve entradas indexadas por identidad de dispositivo.
- Las entradas de presencia incluyen `deviceId`, `roles` y `scopes` para que las UI puedan mostrar una sola fila por dispositivo
  incluso cuando se conecta tanto como **operator** como **node**.

## Familias comunes de métodos RPC

Esta página no es un volcado completo generado, pero la superficie WS pública es más amplia
que los ejemplos de handshake/autenticación anteriores. Estas son las principales familias de métodos que
Gateway expone hoy.

`hello-ok.features.methods` es una lista conservadora de descubrimiento construida a partir de
`src/gateway/server-methods-list.ts` más las exportaciones de métodos de plugins/canales cargados.
Trátela como descubrimiento de funciones, no como un volcado generado de cada helper invocable
implementado en `src/gateway/server-methods/*.ts`.

### Sistema e identidad

- `health` devuelve la instantánea de estado del gateway almacenada en caché o recién sondeada.
- `status` devuelve el resumen del gateway al estilo `/status`; los campos sensibles se
  incluyen solo para clientes `operator` con alcance de administrador.
- `gateway.identity.get` devuelve la identidad del dispositivo del gateway usada por los flujos de relay y
  emparejamiento.
- `system-presence` devuelve la instantánea de presencia actual para los dispositivos
  `operator`/`node` conectados.
- `system-event` agrega un evento del sistema y puede actualizar/transmitir el contexto
  de presencia.
- `last-heartbeat` devuelve el evento Heartbeat persistido más reciente.
- `set-heartbeats` activa o desactiva el procesamiento de Heartbeat en el gateway.

### Modelos y uso

- `models.list` devuelve el catálogo de modelos permitido en tiempo de ejecución.
- `usage.status` devuelve resúmenes de ventanas de uso/restante de cuota del proveedor.
- `usage.cost` devuelve resúmenes agregados de uso de costo para un intervalo de fechas.
- `doctor.memory.status` devuelve la preparación de memoria vectorial/embeddings para el
  espacio de trabajo del agente predeterminado activo.
- `sessions.usage` devuelve resúmenes de uso por sesión.
- `sessions.usage.timeseries` devuelve series temporales de uso para una sesión.
- `sessions.usage.logs` devuelve entradas del registro de uso para una sesión.

### Canales y helpers de inicio de sesión

- `channels.status` devuelve resúmenes de estado de canales/plugins integrados + empaquetados.
- `channels.logout` cierra la sesión de un canal/cuenta específicos cuando el canal
  admite cierre de sesión.
- `web.login.start` inicia un flujo de inicio de sesión por QR/web para el proveedor del canal web
  actual compatible con QR.
- `web.login.wait` espera a que ese flujo de inicio de sesión por QR/web finalice y arranca el
  canal si se completa con éxito.
- `push.test` envía una notificación push APNs de prueba a un nodo iOS registrado.
- `voicewake.get` devuelve los disparadores de palabras de activación almacenados.
- `voicewake.set` actualiza los disparadores de palabras de activación y transmite el cambio.

### Mensajería y registros

- `send` es el RPC de entrega saliente directa para envíos dirigidos a canal/cuenta/hilo
  fuera del ejecutor de chat.
- `logs.tail` devuelve la cola del archivo de registros configurado del gateway con cursor/límite y
  controles de bytes máximos.

### Talk y TTS

- `talk.config` devuelve la carga útil de configuración efectiva de Talk; `includeSecrets`
  requiere `operator.talk.secrets` (o `operator.admin`).
- `talk.mode` establece/transmite el estado actual del modo Talk para los clientes de WebChat/Control UI.
- `talk.speak` sintetiza voz a través del proveedor de voz activo de Talk.
- `tts.status` devuelve el estado de activación de TTS, el proveedor activo, los proveedores de respaldo
  y el estado de configuración del proveedor.
- `tts.providers` devuelve el inventario visible de proveedores TTS.
- `tts.enable` y `tts.disable` activan y desactivan el estado de preferencias de TTS.
- `tts.setProvider` actualiza el proveedor TTS preferido.
- `tts.convert` ejecuta una conversión puntual de texto a voz.

### Secretos, configuración, actualización y asistente

- `secrets.reload` vuelve a resolver los SecretRef activos e intercambia el estado de secretos en tiempo de ejecución
  solo si todo se completa correctamente.
- `secrets.resolve` resuelve asignaciones de secretos dirigidas a comandos para un conjunto específico
  de comandos/objetivos.
- `config.get` devuelve la instantánea y el hash de la configuración actual.
- `config.set` escribe una carga útil de configuración validada.
- `config.patch` fusiona una actualización parcial de la configuración.
- `config.apply` valida y reemplaza la carga útil de configuración completa.
- `config.schema` devuelve la carga útil del esquema de configuración en vivo usada por Control UI y
  las herramientas CLI: esquema, `uiHints`, versión y metadatos de generación, incluidos
  metadatos de esquema de plugins + canales cuando el entorno de ejecución puede cargarlos. El esquema
  incluye metadatos de campo `title` / `description` derivados de las mismas etiquetas
  y el mismo texto de ayuda usados por la UI, incluidas las ramas anidadas de composición de objetos, comodines,
  elementos de array y `anyOf` / `oneOf` / `allOf` cuando existe documentación de campo
  coincidente.
- `config.schema.lookup` devuelve una carga útil de búsqueda acotada a ruta para una ruta de configuración:
  ruta normalizada, un nodo superficial del esquema, `hint` + `hintPath` coincidentes y
  resúmenes inmediatos de hijos para la exploración detallada en UI/CLI.
  - Los nodos de esquema de búsqueda conservan la documentación orientada al usuario y los campos comunes de validación:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    límites numéricos/de cadenas/de arrays/de objetos y banderas booleanas como
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Los resúmenes de hijos exponen `key`, `path` normalizada, `type`, `required`,
    `hasChildren`, además de `hint` / `hintPath` coincidentes.
- `update.run` ejecuta el flujo de actualización del gateway y programa un reinicio solo cuando
  la propia actualización se completó correctamente.
- `wizard.start`, `wizard.next`, `wizard.status` y `wizard.cancel` exponen el
  asistente de incorporación a través de WS RPC.

### Familias principales existentes

#### Agente y helpers de espacio de trabajo

- `agents.list` devuelve las entradas de agente configuradas.
- `agents.create`, `agents.update` y `agents.delete` administran los registros de agentes y
  la conexión del espacio de trabajo.
- `agents.files.list`, `agents.files.get` y `agents.files.set` administran los
  archivos del espacio de trabajo de arranque expuestos para un agente.
- `agent.identity.get` devuelve la identidad efectiva del asistente para un agente o
  sesión.
- `agent.wait` espera a que una ejecución termine y devuelve la instantánea terminal cuando
  está disponible.

#### Control de sesión

- `sessions.list` devuelve el índice de sesión actual.
- `sessions.subscribe` y `sessions.unsubscribe` activan o desactivan las suscripciones a eventos de cambio de sesión
  para el cliente WS actual.
- `sessions.messages.subscribe` y `sessions.messages.unsubscribe` activan o desactivan
  las suscripciones a eventos de transcripción/mensaje para una sesión.
- `sessions.preview` devuelve vistas previas acotadas de la transcripción para claves de sesión
  específicas.
- `sessions.resolve` resuelve o canonicaliza un destino de sesión.
- `sessions.create` crea una nueva entrada de sesión.
- `sessions.send` envía un mensaje a una sesión existente.
- `sessions.steer` es la variante de interrumpir y redirigir para una sesión activa.
- `sessions.abort` aborta el trabajo activo de una sesión.
- `sessions.patch` actualiza metadatos/anulaciones de la sesión.
- `sessions.reset`, `sessions.delete` y `sessions.compact` realizan tareas de
  mantenimiento de la sesión.
- `sessions.get` devuelve la fila completa de la sesión almacenada.
- la ejecución del chat sigue usando `chat.history`, `chat.send`, `chat.abort` y
  `chat.inject`.
- `chat.history` está normalizado para visualización para clientes UI: las etiquetas de directiva en línea se
  eliminan del texto visible, las cargas útiles XML de llamada de herramienta en texto plano (incluidas
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y
  bloques de llamada de herramienta truncados) y los tokens de control del modelo filtrados en ASCII/ancho completo
  se eliminan, las filas del asistente compuestas únicamente por tokens silenciosos como `NO_REPLY` /
  `no_reply` exactos se omiten, y las filas sobredimensionadas pueden reemplazarse por marcadores de posición.

#### Emparejamiento de dispositivos y tokens de dispositivo

- `device.pair.list` devuelve los dispositivos emparejados pendientes y aprobados.
- `device.pair.approve`, `device.pair.reject` y `device.pair.remove` administran
  los registros de emparejamiento de dispositivos.
- `device.token.rotate` rota un token de dispositivo emparejado dentro de sus límites aprobados de rol
  y alcance.
- `device.token.revoke` revoca un token de dispositivo emparejado.

#### Emparejamiento de nodos, invocación y trabajo pendiente

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` y `node.pair.verify` cubren el emparejamiento de nodos y la
  verificación de arranque.
- `node.list` y `node.describe` devuelven el estado de nodos conocidos/conectados.
- `node.rename` actualiza una etiqueta de nodo emparejado.
- `node.invoke` reenvía un comando a un nodo conectado.
- `node.invoke.result` devuelve el resultado de una solicitud de invocación.
- `node.event` transporta eventos originados en el nodo de vuelta al gateway.
- `node.canvas.capability.refresh` actualiza tokens acotados de capacidad de canvas.
- `node.pending.pull` y `node.pending.ack` son las API de cola para nodos conectados.
- `node.pending.enqueue` y `node.pending.drain` administran trabajo pendiente duradero
  para nodos desconectados o fuera de línea.

#### Familias de aprobación

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` y
  `exec.approval.resolve` cubren solicitudes puntuales de aprobación de `exec` más
  la búsqueda/repetición de aprobaciones pendientes.
- `exec.approval.waitDecision` espera una aprobación `exec` pendiente y devuelve
  la decisión final (o `null` en caso de tiempo de espera agotado).
- `exec.approvals.get` y `exec.approvals.set` administran instantáneas de la política
  de aprobación `exec` del gateway.
- `exec.approvals.node.get` y `exec.approvals.node.set` administran la política local de aprobación `exec`
  del nodo mediante comandos de relay del nodo.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` y `plugin.approval.resolve` cubren
  flujos de aprobación definidos por Plugin.

#### Otras familias principales

- automatización:
  - `wake` programa una inyección de texto de activación inmediata o en el siguiente Heartbeat
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- Skills/herramientas: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`

### Familias comunes de eventos

- `chat`: actualizaciones de chat de la UI, como `chat.inject` y otros
  eventos de chat solo de transcripción.
- `session.message` y `session.tool`: actualizaciones de transcripción/flujo de eventos para una
  sesión suscrita.
- `sessions.changed`: cambió el índice de sesión o sus metadatos.
- `presence`: actualizaciones de la instantánea de presencia del sistema.
- `tick`: evento periódico de keepalive / actividad.
- `health`: actualización de la instantánea de estado del gateway.
- `heartbeat`: actualización del flujo de eventos Heartbeat.
- `cron`: evento de cambio de ejecución/trabajo de Cron.
- `shutdown`: notificación de apagado del gateway.
- `node.pair.requested` / `node.pair.resolved`: ciclo de vida del emparejamiento de nodos.
- `node.invoke.request`: difusión de solicitud de invocación de nodo.
- `device.pair.requested` / `device.pair.resolved`: ciclo de vida del dispositivo emparejado.
- `voicewake.changed`: cambió la configuración de disparadores de palabra de activación.
- `exec.approval.requested` / `exec.approval.resolved`: ciclo de vida de aprobación de `exec`.
- `plugin.approval.requested` / `plugin.approval.resolved`: ciclo de vida de aprobación de Plugin.

### Métodos helper de nodo

- Los nodos pueden llamar a `skills.bins` para obtener la lista actual de ejecutables de Skills
  para comprobaciones de permiso automático.

### Métodos helper de operador

- Los operadores pueden llamar a `commands.list` (`operator.read`) para obtener el inventario de comandos en tiempo de ejecución para un
  agente.
  - `agentId` es opcional; omítalo para leer el espacio de trabajo del agente predeterminado.
  - `scope` controla a qué superficie apunta el `name` principal:
    - `text` devuelve el token principal de comando de texto sin la `/` inicial
    - `native` y la ruta predeterminada `both` devuelven nombres nativos compatibles con proveedor
      cuando están disponibles
  - `textAliases` contiene alias slash exactos como `/model` y `/m`.
  - `nativeName` contiene el nombre de comando nativo compatible con el proveedor cuando existe.
  - `provider` es opcional y solo afecta el nombre nativo más la disponibilidad de comandos nativos
    del Plugin.
  - `includeArgs=false` omite del resultado los metadatos serializados de argumentos.
- Los operadores pueden llamar a `tools.catalog` (`operator.read`) para obtener el catálogo de herramientas en tiempo de ejecución para un
  agente. La respuesta incluye herramientas agrupadas y metadatos de procedencia:
  - `source`: `core` o `plugin`
  - `pluginId`: propietario del Plugin cuando `source="plugin"`
  - `optional`: si una herramienta de Plugin es opcional
- Los operadores pueden llamar a `tools.effective` (`operator.read`) para obtener el inventario efectivo de herramientas en tiempo de ejecución
  para una sesión.
  - `sessionKey` es obligatorio.
  - El gateway deriva el contexto confiable de tiempo de ejecución del lado del servidor a partir de la sesión en lugar de aceptar
    autenticación o contexto de entrega proporcionados por quien llama.
  - La respuesta está acotada a la sesión y refleja lo que la conversación activa puede usar en este momento,
    incluidas herramientas de núcleo, Plugin y canal.
- Los operadores pueden llamar a `skills.status` (`operator.read`) para obtener el inventario visible
  de Skills para un agente.
  - `agentId` es opcional; omítalo para leer el espacio de trabajo del agente predeterminado.
  - La respuesta incluye elegibilidad, requisitos faltantes, comprobaciones de configuración y
    opciones de instalación saneadas sin exponer valores secretos sin procesar.
- Los operadores pueden llamar a `skills.search` y `skills.detail` (`operator.read`) para
  metadatos de descubrimiento de ClawHub.
- Los operadores pueden llamar a `skills.install` (`operator.admin`) en dos modos:
  - Modo ClawHub: `{ source: "clawhub", slug, version?, force? }` instala una
    carpeta de skill en el directorio `skills/` del espacio de trabajo del agente predeterminado.
  - Modo instalador de gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    ejecuta una acción declarada `metadata.openclaw.install` en el host del gateway.
- Los operadores pueden llamar a `skills.update` (`operator.admin`) en dos modos:
  - El modo ClawHub actualiza un `slug` rastreado o todas las instalaciones rastreadas de ClawHub en
    el espacio de trabajo del agente predeterminado.
  - El modo configuración aplica un parche a valores `skills.entries.<skillKey>` como `enabled`,
    `apiKey` y `env`.

## Aprobaciones de `exec`

- Cuando una solicitud `exec` necesita aprobación, el gateway difunde `exec.approval.requested`.
- Los clientes operador resuelven llamando a `exec.approval.resolve` (requiere alcance `operator.approvals`).
- Para `host=node`, `exec.approval.request` debe incluir `systemRunPlan` (`argv`/`cwd`/`rawCommand`/metadatos de sesión canónicos). Las solicitudes sin `systemRunPlan` se rechazan.
- Después de la aprobación, las llamadas reenviadas `node.invoke system.run` reutilizan ese
  `systemRunPlan` canónico como contexto autoritativo de comando/cwd/sesión.
- Si quien llama modifica `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` entre `prepare` y el reenvío final aprobado de `system.run`, el
  gateway rechaza la ejecución en lugar de confiar en la carga útil modificada.

## Respaldo de entrega del agente

- Las solicitudes `agent` pueden incluir `deliver=true` para solicitar entrega saliente.
- `bestEffortDeliver=false` mantiene un comportamiento estricto: los destinos de entrega no resueltos o solo internos devuelven `INVALID_REQUEST`.
- `bestEffortDeliver=true` permite recurrir a ejecución solo de sesión cuando no puede resolverse ninguna ruta externa entregable (por ejemplo, sesiones internas/webchat o configuraciones ambiguas de múltiples canales).

## Control de versiones

- `PROTOCOL_VERSION` se encuentra en `src/gateway/protocol/schema/protocol-schemas.ts`.
- Los clientes envían `minProtocol` + `maxProtocol`; el servidor rechaza discrepancias.
- Los esquemas + modelos se generan a partir de definiciones TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### Constantes del cliente

El cliente de referencia en `src/gateway/client.ts` usa estos valores predeterminados. Los valores son
estables en todo el protocolo v3 y son la línea base esperada para clientes de terceros.

| Constant                                  | Default                                               | Source                                                     |
| ----------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `3`                                                   | `src/gateway/protocol/schema/protocol-schemas.ts`          |
| Tiempo de espera de solicitud (por RPC)   | `30_000` ms                                           | `src/gateway/client.ts` (`requestTimeoutMs`)               |
| Tiempo de espera de preautorización / desafío de conexión | `10_000` ms                              | `src/gateway/handshake-timeouts.ts` (clamp `250`–`10_000`) |
| Retroceso inicial de reconexión           | `1_000` ms                                            | `src/gateway/client.ts` (`backoffMs`)                      |
| Retroceso máximo de reconexión            | `30_000` ms                                           | `src/gateway/client.ts` (`scheduleReconnect`)              |
| Límite de reintento rápido tras cierre por token de dispositivo | `250` ms                         | `src/gateway/client.ts`                                    |
| Período de gracia de parada forzada antes de `terminate()` | `250` ms                          | `FORCE_STOP_TERMINATE_GRACE_MS`                            |
| Tiempo de espera predeterminado de `stopAndWait()` | `1_000` ms                                 | `STOP_AND_WAIT_TIMEOUT_MS`                                 |
| Intervalo `tick` predeterminado (antes de `hello-ok`) | `30_000` ms                           | `src/gateway/client.ts`                                    |
| Cierre por tiempo de espera de `tick`     | código `4000` cuando el silencio supera `tickIntervalMs * 2` | `src/gateway/client.ts`                          |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024` (25 MB)                            | `src/gateway/server-constants.ts`                          |

El servidor anuncia los valores efectivos `policy.tickIntervalMs`, `policy.maxPayload`
y `policy.maxBufferedBytes` en `hello-ok`; los clientes deben respetar esos valores
en lugar de los valores predeterminados previos al handshake.

## Autenticación

- La autenticación del gateway con secreto compartido usa `connect.params.auth.token` o
  `connect.params.auth.password`, según el modo de autenticación configurado.
- Los modos con identidad, como Tailscale Serve
  (`gateway.auth.allowTailscale: true`) o `gateway.auth.mode: "trusted-proxy"`
  en interfaces que no son de loopback, satisfacen la comprobación de autenticación de `connect` a partir de
  encabezados de la solicitud en lugar de `connect.params.auth.*`.
- `gateway.auth.mode: "none"` para ingreso privado omite por completo la autenticación `connect` con secreto compartido;
  no exponga ese modo en ingresos públicos o no confiables.
- Después del emparejamiento, Gateway emite un **token de dispositivo** limitado al rol + alcances
  de la conexión. Se devuelve en `hello-ok.auth.deviceToken` y el cliente debe
  persistirlo para conexiones futuras.
- Los clientes deben persistir el `hello-ok.auth.deviceToken` principal después de cualquier
  conexión exitosa.
- Al reconectarse con ese token de dispositivo **almacenado**, también se debe reutilizar el conjunto de alcances aprobados
  almacenado para ese token. Esto preserva el acceso de lectura/sondeo/estado
  que ya se concedió y evita que las reconexiones colapsen silenciosamente a un
  alcance implícito más estrecho de solo administrador.
- Ensamblado de autenticación `connect` del lado del cliente (`selectConnectAuth` en
  `src/gateway/client.ts`):
  - `auth.password` es ortogonal y siempre se reenvía cuando está establecido.
  - `auth.token` se completa en este orden de prioridad: primero el token compartido explícito,
    luego un `deviceToken` explícito y, después, un token almacenado por dispositivo (indexado por
    `deviceId` + `role`).
  - `auth.bootstrapToken` se envía solo cuando ninguna de las opciones anteriores resolvió un
    `auth.token`. Un token compartido o cualquier token de dispositivo resuelto lo suprime.
  - La autopromoción de un token de dispositivo almacenado en el reintento puntual
    `AUTH_TOKEN_MISMATCH` está restringida **solo a endpoints confiables**:
    loopback, o `wss://` con `tlsFingerprint` fijado. `wss://` público
    sin fijación no califica.
- Las entradas adicionales `hello-ok.auth.deviceTokens` son tokens de transferencia de arranque.
  Persístalos solo cuando la conexión haya usado autenticación de arranque en un transporte confiable
  como `wss://` o loopback/emparejamiento local.
- Si un cliente proporciona un `deviceToken` **explícito** o `scopes` explícitos, ese
  conjunto de alcances solicitado por el llamador sigue siendo autoritativo; los alcances en caché solo
  se reutilizan cuando el cliente está reutilizando el token almacenado por dispositivo.
- Los tokens de dispositivo pueden rotarse/revocarse mediante `device.token.rotate` y
  `device.token.revoke` (requiere alcance `operator.pairing`).
- La emisión/rotación de tokens se mantiene limitada al conjunto de roles aprobado registrado en
  la entrada de emparejamiento de ese dispositivo; rotar un token no puede ampliar el dispositivo a un
  rol que la aprobación de emparejamiento nunca concedió.
- Para sesiones de token de dispositivo emparejado, la administración del dispositivo está autocontenida salvo que el
  llamador también tenga `operator.admin`: quienes no son administradores solo pueden quitar/revocar/rotar
  su **propia** entrada de dispositivo.
- `device.token.rotate` también verifica el conjunto solicitado de alcances de operador frente a los
  alcances de la sesión actual del llamador. Quienes no son administradores no pueden rotar un token a
  un conjunto de alcances de operador más amplio del que ya tienen.
- Los fallos de autenticación incluyen `error.details.code` más sugerencias de recuperación:
  - `error.details.canRetryWithDeviceToken` (booleano)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Comportamiento del cliente para `AUTH_TOKEN_MISMATCH`:
  - Los clientes confiables pueden intentar un reintento acotado con un token almacenado por dispositivo.
  - Si ese reintento falla, los clientes deben detener los bucles automáticos de reconexión y mostrar orientación de acción al operador.

## Identidad del dispositivo + emparejamiento

- Los nodos deben incluir una identidad de dispositivo estable (`device.id`) derivada de la huella digital
  de un par de claves.
- Los gateways emiten tokens por dispositivo + rol.
- Las aprobaciones de emparejamiento son necesarias para nuevos ID de dispositivo, a menos que la aprobación automática local
  esté habilitada.
- La aprobación automática de emparejamiento se centra en conexiones directas de loopback local.
- OpenClaw también tiene una ruta estrecha de autoconexión local al backend/contenedor para
  flujos auxiliares confiables con secreto compartido.
- Las conexiones tailnet o LAN del mismo host se siguen tratando como remotas para el emparejamiento y
  requieren aprobación.
- Todos los clientes WS deben incluir identidad `device` durante `connect` (`operator` + `node`).
  Control UI puede omitirla solo en estos modos:
  - `gateway.controlUi.allowInsecureAuth=true` para compatibilidad con HTTP inseguro solo en localhost.
  - autenticación exitosa de Control UI de operador con `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (medida extrema, degradación grave de seguridad).
- Todas las conexiones deben firmar el nonce `connect.challenge` proporcionado por el servidor.

### Diagnósticos de migración de autenticación de dispositivo

Para clientes heredados que todavía usan el comportamiento de firma previo al desafío, `connect` ahora devuelve
códigos de detalle `DEVICE_AUTH_*` en `error.details.code` con un `error.details.reason` estable.

Fallos comunes de migración:

| Message                     | details.code                     | details.reason           | Meaning                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | El cliente omitió `device.nonce` (o lo envió vacío).     |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | El cliente firmó con un nonce obsoleto o incorrecto.            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | La carga útil de la firma no coincide con la carga útil v2.       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | La marca de tiempo firmada está fuera del desfase permitido.          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` no coincide con la huella digital de la clave pública. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | El formato/canonicalización de la clave pública falló.         |

Objetivo de la migración:

- Espere siempre a `connect.challenge`.
- Firme la carga útil v2 que incluye el nonce del servidor.
- Envíe el mismo nonce en `connect.params.device.nonce`.
- La carga útil de firma preferida es `v3`, que vincula `platform` y `deviceFamily`
  además de los campos de dispositivo/cliente/rol/alcances/token/nonce.
- Las firmas heredadas `v2` siguen aceptándose por compatibilidad, pero la fijación de metadatos
  de dispositivos emparejados sigue controlando la política de comandos al reconectarse.

## TLS + fijación

- TLS es compatible con conexiones WS.
- Los clientes pueden fijar opcionalmente la huella digital del certificado del gateway (consulte la configuración `gateway.tls`
  más `gateway.remote.tlsFingerprint` o la CLI `--tls-fingerprint`).

## Alcance

Este protocolo expone la **API completa del gateway** (estado, canales, modelos, chat,
agente, sesiones, nodos, aprobaciones, etc.). La superficie exacta está definida por los
esquemas TypeBox en `src/gateway/protocol/schema.ts`.
