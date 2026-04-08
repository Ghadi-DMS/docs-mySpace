---
read_when:
    - Implementar o actualizar clientes WS del gateway
    - Depurar incompatibilidades del protocolo o fallos de conexión
    - Regenerar el esquema/modelos del protocolo
summary: 'Protocolo WebSocket del Gateway: handshake, frames, control de versiones'
title: Protocolo del Gateway
x-i18n:
    generated_at: "2026-04-08T02:15:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8635e3ac1dd311dbd3a770b088868aa1495a8d53b3ebc1eae0dfda3b2bf4694a
    source_path: gateway/protocol.md
    workflow: 15
---

# Protocolo del Gateway (WebSocket)

El protocolo WS del Gateway es el **único plano de control + transporte de nodos** para
OpenClaw. Todos los clientes (CLI, UI web, app de macOS, nodos iOS/Android, nodos
sin interfaz) se conectan por WebSocket y declaran su **rol** + **alcance** en el
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
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
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

Durante el traspaso de bootstrap de confianza, `hello-ok.auth` también puede incluir entradas
adicionales de rol limitado en `deviceTokens`:

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

Para el flujo de bootstrap integrado de nodo/operador, el token principal del nodo permanece con
`scopes: []` y cualquier token de operador transferido permanece limitado a la lista de permitidos
del operador de bootstrap (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Las comprobaciones de alcance de bootstrap siguen
teniendo prefijo de rol: las entradas de operador solo satisfacen solicitudes de operador, y los
roles no operadores siguen necesitando alcances bajo su propio prefijo de rol.

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

## Estructura de tramas

- **Solicitud**: `{type:"req", id, method, params}`
- **Respuesta**: `{type:"res", id, ok, payload|error}`
- **Evento**: `{type:"event", event, payload, seq?, stateVersion?}`

Los métodos con efectos secundarios requieren **claves de idempotencia** (consulta el esquema).

## Roles + alcances

### Roles

- `operator` = cliente del plano de control (CLI/UI/automatización).
- `node` = host de capacidades (camera/screen/canvas/system.run).

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

Los métodos RPC del gateway registrados por plugins pueden solicitar su propio alcance de operador, pero
los prefijos de administración reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre se resuelven como `operator.admin`.

El alcance del método es solo la primera barrera. Algunos comandos slash alcanzados mediante
`chat.send` aplican comprobaciones de nivel de comando más estrictas encima. Por ejemplo, las
escrituras persistentes de `/config set` y `/config unset` requieren `operator.admin`.

`node.pair.approve` también tiene una comprobación adicional de alcance en el momento de la aprobación por encima del
alcance base del método:

- solicitudes sin comando: `operator.pairing`
- solicitudes con comandos de nodo que no son exec: `operator.pairing` + `operator.write`
- solicitudes que incluyen `system.run`, `system.run.prepare` o `system.which`:
  `operator.pairing` + `operator.admin`

### `caps`/`commands`/`permissions` (`node`)

Los nodos declaran las capacidades reclamadas en el momento de la conexión:

- `caps`: categorías de capacidades de alto nivel.
- `commands`: lista de permitidos de comandos para `invoke`.
- `permissions`: interruptores granulares (por ejemplo, `screen.record`, `camera.capture`).

El Gateway trata estas como **declaraciones** y aplica listas de permitidos del lado del servidor.

## Presencia

- `system-presence` devuelve entradas indexadas por identidad de dispositivo.
- Las entradas de presencia incluyen `deviceId`, `roles` y `scopes` para que las UI puedan mostrar una sola fila por dispositivo
  incluso cuando se conecta como **operator** y **node** a la vez.

## Familias comunes de métodos RPC

Esta página no es un volcado completo generado, pero la superficie WS pública es más amplia
que los ejemplos de handshake/auth anteriores. Estas son las principales familias de métodos que el
Gateway expone hoy.

`hello-ok.features.methods` es una lista de descubrimiento conservadora construida a partir de
`src/gateway/server-methods-list.ts` más las exportaciones de métodos cargadas de plugins/canales.
Trátala como descubrimiento de funciones, no como un volcado generado de cada helper invocable
implementado en `src/gateway/server-methods/*.ts`.

### Sistema e identidad

- `health` devuelve la instantánea de estado del gateway almacenada en caché o sondeada recientemente.
- `status` devuelve el resumen del gateway estilo `/status`; los campos sensibles se
  incluyen solo para clientes operador con alcance de administrador.
- `gateway.identity.get` devuelve la identidad del dispositivo del gateway usada por los flujos de relay y
  emparejamiento.
- `system-presence` devuelve la instantánea actual de presencia de los dispositivos
  operator/node conectados.
- `system-event` agrega un evento del sistema y puede actualizar/difundir el contexto
  de presencia.
- `last-heartbeat` devuelve el último evento heartbeat persistido.
- `set-heartbeats` activa o desactiva el procesamiento de heartbeats en el gateway.

### Modelos y uso

- `models.list` devuelve el catálogo de modelos permitidos en tiempo de ejecución.
- `usage.status` devuelve resúmenes de ventanas de uso/resto de cuota por proveedor.
- `usage.cost` devuelve resúmenes agregados de uso de costos para un intervalo de fechas.
- `doctor.memory.status` devuelve el estado de preparación de memoria vectorial / embeddings para el
  espacio de trabajo activo del agente predeterminado.
- `sessions.usage` devuelve resúmenes de uso por sesión.
- `sessions.usage.timeseries` devuelve series temporales de uso para una sesión.
- `sessions.usage.logs` devuelve entradas del registro de uso para una sesión.

### Canales y helpers de inicio de sesión

- `channels.status` devuelve resúmenes de estado de canales/plugins integrados + empaquetados.
- `channels.logout` cierra sesión en un canal/cuenta específico cuando el canal
  admite cierre de sesión.
- `web.login.start` inicia un flujo de inicio de sesión por QR/web para el proveedor de canal web
  con capacidad QR actual.
- `web.login.wait` espera a que ese flujo de inicio de sesión por QR/web finalice e inicia el
  canal si tiene éxito.
- `push.test` envía una notificación push de prueba de APNs a un nodo iOS registrado.
- `voicewake.get` devuelve los activadores de palabra de activación almacenados.
- `voicewake.set` actualiza los activadores de palabra de activación y difunde el cambio.

### Mensajería y registros

- `send` es el RPC de entrega saliente directa para envíos dirigidos a
  canal/cuenta/hilo fuera del ejecutor de chat.
- `logs.tail` devuelve la cola configurada del registro de archivos del gateway con cursor/límite y
  controles de bytes máximos.

### Talk y TTS

- `talk.config` devuelve la carga útil efectiva de configuración de Talk; `includeSecrets`
  requiere `operator.talk.secrets` (o `operator.admin`).
- `talk.mode` establece/difunde el estado actual del modo Talk para clientes
  WebChat/Control UI.
- `talk.speak` sintetiza voz mediante el proveedor de voz activo de Talk.
- `tts.status` devuelve el estado de TTS habilitado, proveedor activo, proveedores de reserva
  y estado de configuración del proveedor.
- `tts.providers` devuelve el inventario visible de proveedores de TTS.
- `tts.enable` y `tts.disable` activan o desactivan el estado de preferencias de TTS.
- `tts.setProvider` actualiza el proveedor de TTS preferido.
- `tts.convert` ejecuta una conversión puntual de texto a voz.

### Secretos, configuración, actualización y asistente

- `secrets.reload` vuelve a resolver los SecretRefs activos e intercambia el estado de secretos en tiempo de ejecución
  solo si todo se completa correctamente.
- `secrets.resolve` resuelve asignaciones de secretos dirigidas a comandos para un conjunto específico
  de comando/objetivo.
- `config.get` devuelve la instantánea de configuración actual y su hash.
- `config.set` escribe una carga útil de configuración validada.
- `config.patch` fusiona una actualización parcial de configuración.
- `config.apply` valida + reemplaza la carga útil completa de configuración.
- `config.schema` devuelve la carga útil del esquema de configuración activo usada por Control UI y
  herramientas CLI: esquema, `uiHints`, versión y metadatos de generación, incluidos
  metadatos de esquema de plugins + canales cuando el runtime puede cargarlos. El esquema
  incluye metadatos de campo `title` / `description` derivados de las mismas etiquetas
  y textos de ayuda usados por la UI, incluyendo ramas compuestas de objetos anidados, comodines, elementos
  de array y `anyOf` / `oneOf` / `allOf` cuando existe documentación de campo
  coincidente.
- `config.schema.lookup` devuelve una carga útil de búsqueda con alcance de ruta para una ruta de configuración:
  ruta normalizada, un nodo superficial del esquema, la sugerencia coincidente + `hintPath`, y
  resúmenes inmediatos de hijos para exploración en UI/CLI.
  - Los nodos de esquema de búsqueda conservan la documentación orientada al usuario y los campos comunes de validación:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    límites numéricos/de cadena/de array/de objeto y marcas booleanas como
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Los resúmenes de hijos exponen `key`, `path` normalizada, `type`, `required`,
    `hasChildren`, más la `hint` / `hintPath` coincidente.
- `update.run` ejecuta el flujo de actualización del gateway y programa un reinicio solo cuando
  la actualización en sí se completó correctamente.
- `wizard.start`, `wizard.next`, `wizard.status` y `wizard.cancel` exponen el
  asistente de onboarding mediante WS RPC.

### Familias principales existentes

#### Helpers de agente y espacio de trabajo

- `agents.list` devuelve entradas de agentes configurados.
- `agents.create`, `agents.update` y `agents.delete` administran registros de agentes y
  la conexión del espacio de trabajo.
- `agents.files.list`, `agents.files.get` y `agents.files.set` administran los
  archivos del espacio de trabajo de bootstrap expuestos para un agente.
- `agent.identity.get` devuelve la identidad efectiva del asistente para un agente o
  sesión.
- `agent.wait` espera a que termine una ejecución y devuelve la instantánea terminal cuando
  está disponible.

#### Control de sesiones

- `sessions.list` devuelve el índice actual de sesiones.
- `sessions.subscribe` y `sessions.unsubscribe` activan o desactivan las suscripciones a eventos de cambio de sesión
  para el cliente WS actual.
- `sessions.messages.subscribe` y `sessions.messages.unsubscribe` activan o desactivan
  las suscripciones a eventos de transcripción/mensajes para una sesión.
- `sessions.preview` devuelve vistas previas acotadas de transcripciones para claves
  de sesión específicas.
- `sessions.resolve` resuelve o canoniza un objetivo de sesión.
- `sessions.create` crea una nueva entrada de sesión.
- `sessions.send` envía un mensaje a una sesión existente.
- `sessions.steer` es la variante de interrupción y redirección para una sesión activa.
- `sessions.abort` aborta el trabajo activo de una sesión.
- `sessions.patch` actualiza metadatos/sobrescrituras de sesión.
- `sessions.reset`, `sessions.delete` y `sessions.compact` realizan mantenimiento
  de sesión.
- `sessions.get` devuelve la fila completa almacenada de la sesión.
- La ejecución de chat sigue usando `chat.history`, `chat.send`, `chat.abort` y
  `chat.inject`.
- `chat.history` está normalizado para visualización para clientes UI: las etiquetas de directiva en línea se
  eliminan del texto visible, las cargas útiles XML de llamadas de herramientas en texto plano (incluyendo
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y
  bloques truncados de llamadas de herramientas) y los tokens de control del modelo filtrados en ASCII/ancho completo
  se eliminan, las filas puras del asistente con tokens silenciosos como exactamente `NO_REPLY` /
  `no_reply` se omiten, y las filas sobredimensionadas pueden reemplazarse con marcadores de posición.

#### Emparejamiento de dispositivos y tokens de dispositivo

- `device.pair.list` devuelve dispositivos emparejados pendientes y aprobados.
- `device.pair.approve`, `device.pair.reject` y `device.pair.remove` administran
  los registros de emparejamiento de dispositivos.
- `device.token.rotate` rota un token de dispositivo emparejado dentro de sus límites aprobados de rol
  y alcance.
- `device.token.revoke` revoca un token de dispositivo emparejado.

#### Emparejamiento de nodos, `invoke` y trabajo pendiente

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` y `node.pair.verify` cubren el emparejamiento de nodos y la verificación
  de bootstrap.
- `node.list` y `node.describe` devuelven el estado de nodos conocidos/conectados.
- `node.rename` actualiza una etiqueta de nodo emparejado.
- `node.invoke` reenvía un comando a un nodo conectado.
- `node.invoke.result` devuelve el resultado de una solicitud `invoke`.
- `node.event` transporta eventos originados en nodos de vuelta al gateway.
- `node.canvas.capability.refresh` actualiza tokens de capacidad de canvas con alcance.
- `node.pending.pull` y `node.pending.ack` son las API de cola para nodos conectados.
- `node.pending.enqueue` y `node.pending.drain` administran trabajo pendiente persistente
  para nodos desconectados o sin conexión.

#### Familias de aprobación

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` y
  `exec.approval.resolve` cubren solicitudes puntuales de aprobación de exec más
  búsqueda/reproducción de aprobaciones pendientes.
- `exec.approval.waitDecision` espera una aprobación de exec pendiente y devuelve
  la decisión final (o `null` por timeout).
- `exec.approvals.get` y `exec.approvals.set` administran instantáneas de la política de aprobación
  de exec del gateway.
- `exec.approvals.node.get` y `exec.approvals.node.set` administran la política de aprobación local de exec
  del nodo mediante comandos de relay del nodo.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` y `plugin.approval.resolve` cubren
  flujos de aprobación definidos por plugins.

#### Otras familias principales

- automatización:
  - `wake` programa una inyección de texto de activación inmediata o en el siguiente heartbeat
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- Skills/herramientas: `skills.*`, `tools.catalog`, `tools.effective`

### Familias comunes de eventos

- `chat`: actualizaciones de chat de la UI como `chat.inject` y otros
  eventos de chat solo de transcripción.
- `session.message` y `session.tool`: actualizaciones de transcripción/flujo de eventos para una
  sesión suscrita.
- `sessions.changed`: cambió el índice o los metadatos de las sesiones.
- `presence`: actualizaciones de la instantánea de presencia del sistema.
- `tick`: evento periódico de keepalive / actividad.
- `health`: actualización de la instantánea de estado del gateway.
- `heartbeat`: actualización del flujo de eventos heartbeat.
- `cron`: evento de cambio de trabajo/ejecución cron.
- `shutdown`: notificación de apagado del gateway.
- `node.pair.requested` / `node.pair.resolved`: ciclo de vida del emparejamiento de nodos.
- `node.invoke.request`: difusión de solicitud de `invoke` de nodo.
- `device.pair.requested` / `device.pair.resolved`: ciclo de vida de dispositivos emparejados.
- `voicewake.changed`: cambió la configuración de los activadores de palabra de activación.
- `exec.approval.requested` / `exec.approval.resolved`: ciclo de vida de la
  aprobación de exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: ciclo de vida de la aprobación
  del plugin.

### Métodos helper de nodo

- Los nodos pueden llamar a `skills.bins` para obtener la lista actual de ejecutables
  de Skills para comprobaciones de permiso automático.

### Métodos helper de operador

- Los operadores pueden llamar a `tools.catalog` (`operator.read`) para obtener el catálogo de herramientas en tiempo de ejecución de un
  agente. La respuesta incluye herramientas agrupadas y metadatos de procedencia:
  - `source`: `core` o `plugin`
  - `pluginId`: propietario del plugin cuando `source="plugin"`
  - `optional`: si una herramienta de plugin es opcional
- Los operadores pueden llamar a `tools.effective` (`operator.read`) para obtener el inventario efectivo de herramientas en tiempo de ejecución
  para una sesión.
  - `sessionKey` es obligatorio.
  - El gateway deriva el contexto de runtime confiable del lado del servidor a partir de la sesión, en lugar de aceptar
    autenticación o contexto de entrega proporcionados por el llamante.
  - La respuesta tiene alcance de sesión y refleja lo que la conversación activa puede usar en este momento,
    incluidas herramientas del núcleo, de plugins y de canal.
- Los operadores pueden llamar a `skills.status` (`operator.read`) para obtener el inventario visible
  de Skills para un agente.
  - `agentId` es opcional; omítelo para leer el espacio de trabajo del agente predeterminado.
  - La respuesta incluye elegibilidad, requisitos faltantes, comprobaciones de configuración y
    opciones de instalación saneadas sin exponer valores secretos sin procesar.
- Los operadores pueden llamar a `skills.search` y `skills.detail` (`operator.read`) para
  metadatos de descubrimiento de ClawHub.
- Los operadores pueden llamar a `skills.install` (`operator.admin`) en dos modos:
  - Modo ClawHub: `{ source: "clawhub", slug, version?, force? }` instala una
    carpeta de Skills en el directorio `skills/` del espacio de trabajo del agente predeterminado.
  - Modo instalador del gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    ejecuta una acción declarada `metadata.openclaw.install` en el host del gateway.
- Los operadores pueden llamar a `skills.update` (`operator.admin`) en dos modos:
  - El modo ClawHub actualiza un slug rastreado o todas las instalaciones de ClawHub rastreadas en
    el espacio de trabajo del agente predeterminado.
  - El modo Config parchea valores de `skills.entries.<skillKey>` como `enabled`,
    `apiKey` y `env`.

## Aprobaciones de exec

- Cuando una solicitud de exec necesita aprobación, el gateway difunde `exec.approval.requested`.
- Los clientes operador resuelven llamando a `exec.approval.resolve` (requiere el alcance `operator.approvals`).
- Para `host=node`, `exec.approval.request` debe incluir `systemRunPlan` (`argv`/`cwd`/`rawCommand`/metadatos de sesión canónicos). Las solicitudes sin `systemRunPlan` se rechazan.
- Tras la aprobación, las llamadas reenviadas `node.invoke system.run` reutilizan ese
  `systemRunPlan` canónico como contexto autorizado de comando/cwd/sesión.
- Si un llamante modifica `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` entre la preparación y el reenvío final aprobado de `system.run`, el
  gateway rechaza la ejecución en lugar de confiar en la carga útil modificada.

## Respaldo de entrega del agente

- Las solicitudes `agent` pueden incluir `deliver=true` para solicitar entrega saliente.
- `bestEffortDeliver=false` mantiene el comportamiento estricto: los objetivos de entrega no resueltos o solo internos devuelven `INVALID_REQUEST`.
- `bestEffortDeliver=true` permite recurrir a ejecución solo de sesión cuando no se puede resolver ninguna ruta externa entregable (por ejemplo, sesiones internas/webchat o configuraciones multicanal ambiguas).

## Control de versiones

- `PROTOCOL_VERSION` está en `src/gateway/protocol/schema.ts`.
- Los clientes envían `minProtocol` + `maxProtocol`; el servidor rechaza incompatibilidades.
- Los esquemas + modelos se generan a partir de definiciones de TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Auth

- La autenticación del gateway con secreto compartido usa `connect.params.auth.token` o
  `connect.params.auth.password`, según el modo de autenticación configurado.
- Los modos con identidad como Tailscale Serve
  (`gateway.auth.allowTailscale: true`) o `gateway.auth.mode: "trusted-proxy"` no loopback
  satisfacen la comprobación de autenticación de `connect` a partir de las cabeceras de la solicitud en lugar de
  `connect.params.auth.*`.
- El ingreso privado `gateway.auth.mode: "none"` omite por completo la autenticación de `connect` con secreto compartido; no expongas ese modo en un ingreso público o no confiable.
- Después del emparejamiento, el Gateway emite un **token de dispositivo** con alcance según el rol + alcances de la conexión. Se devuelve en `hello-ok.auth.deviceToken` y el cliente debería persistirlo para futuras conexiones.
- Los clientes deberían persistir el `hello-ok.auth.deviceToken` principal después de cualquier
  conexión satisfactoria.
- Al reconectar con ese token de dispositivo **almacenado**, también se debería reutilizar el conjunto de alcances aprobados almacenado
  para ese token. Esto preserva el acceso de lectura/sondeo/estado
  que ya se había concedido y evita reducir silenciosamente las reconexiones a un
  alcance implícito más estrecho solo de administrador.
- La precedencia normal de autenticación en la conexión es primero token/contraseña compartidos explícitos, luego
  `deviceToken` explícito, luego token por dispositivo almacenado, y después token de bootstrap.
- Las entradas adicionales `hello-ok.auth.deviceTokens` son tokens de traspaso de bootstrap.
  Persístelos solo cuando la conexión haya usado autenticación de bootstrap en un transporte confiable
  como `wss://` o loopback/emparejamiento local.
- Si un cliente proporciona un `deviceToken` **explícito** o `scopes` explícitos, ese
  conjunto de alcances solicitado por el llamante sigue siendo el autorizado; los alcances en caché solo
  se reutilizan cuando el cliente está reutilizando el token por dispositivo almacenado.
- Los tokens de dispositivo pueden rotarse/revocarse mediante `device.token.rotate` y
  `device.token.revoke` (requiere el alcance `operator.pairing`).
- La emisión/rotación de tokens sigue limitada al conjunto de roles aprobado registrado en
  la entrada de emparejamiento de ese dispositivo; rotar un token no puede ampliar el dispositivo a un
  rol que la aprobación de emparejamiento nunca concedió.
- Para sesiones de token de dispositivo emparejado, la administración del dispositivo tiene alcance propio a menos que el
  llamante también tenga `operator.admin`: los llamantes sin admin solo pueden eliminar/revocar/rotar
  **su propia** entrada de dispositivo.
- `device.token.rotate` también comprueba el conjunto de alcances de operador solicitado frente a los
  alcances actuales de la sesión del llamante. Los llamantes sin admin no pueden rotar un token a
  un conjunto de alcances de operador más amplio del que ya tienen.
- Los fallos de autenticación incluyen `error.details.code` más sugerencias de recuperación:
  - `error.details.canRetryWithDeviceToken` (boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Comportamiento del cliente para `AUTH_TOKEN_MISMATCH`:
  - Los clientes de confianza pueden intentar un reintento limitado con un token por dispositivo en caché.
  - Si ese reintento falla, los clientes deberían detener los bucles automáticos de reconexión y mostrar orientación de acción al operador.

## Identidad del dispositivo + emparejamiento

- Los nodos deberían incluir una identidad de dispositivo estable (`device.id`) derivada de una
  huella digital de un par de claves.
- Los gateways emiten tokens por dispositivo + rol.
- Se requieren aprobaciones de emparejamiento para nuevos IDs de dispositivo, a menos que esté habilitada
  la aprobación automática local.
- La aprobación automática de emparejamiento se centra en conexiones directas local loopback.
- OpenClaw también tiene una ruta estrecha de autoconexión local de backend/contenedor para flujos helper de secreto compartido confiables.
- Las conexiones tailnet o LAN del mismo host siguen tratándose como remotas para el emparejamiento y
  requieren aprobación.
- Todos los clientes WS deben incluir identidad `device` durante `connect` (operator + node).
  Control UI solo puede omitirla en estos modos:
  - `gateway.controlUi.allowInsecureAuth=true` para compatibilidad con HTTP inseguro solo en localhost.
  - autenticación satisfactoria de operador de Control UI con `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (uso de emergencia, degradación grave de seguridad).
- Todas las conexiones deben firmar el nonce proporcionado por el servidor en `connect.challenge`.

### Diagnósticos de migración de autenticación del dispositivo

Para clientes heredados que siguen usando el comportamiento de firma anterior al desafío, `connect` ahora devuelve
códigos de detalle `DEVICE_AUTH_*` en `error.details.code` con un `error.details.reason` estable.

Fallos comunes de migración:

| Mensaje                     | details.code                     | details.reason           | Significado                                        |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | El cliente omitió `device.nonce` (o lo envió vacío). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | El cliente firmó con un nonce obsoleto o incorrecto. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | La carga útil de la firma no coincide con la carga útil v2. |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | La marca de tiempo firmada está fuera de la desviación permitida. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` no coincide con la huella digital de la clave pública. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Falló el formato/canonicalización de la clave pública. |

Objetivo de la migración:

- Espera siempre a `connect.challenge`.
- Firma la carga útil v2 que incluye el nonce del servidor.
- Envía el mismo nonce en `connect.params.device.nonce`.
- La carga útil de firma preferida es `v3`, que vincula `platform` y `deviceFamily`
  además de los campos de dispositivo/cliente/rol/alcances/token/nonce.
- Las firmas heredadas `v2` siguen aceptándose por compatibilidad, pero el anclaje de metadatos
  del dispositivo emparejado sigue controlando la política de comandos en la reconexión.

## TLS + pinning

- TLS es compatible con conexiones WS.
- Los clientes pueden fijar opcionalmente la huella digital del certificado del gateway (consulta la configuración `gateway.tls`
  más `gateway.remote.tlsFingerprint` o la CLI `--tls-fingerprint`).

## Alcance

Este protocolo expone la **API completa del gateway** (status, canales, modelos, chat,
agente, sesiones, nodos, aprobaciones, etc.). La superficie exacta está definida por los
esquemas TypeBox en `src/gateway/protocol/schema.ts`.
