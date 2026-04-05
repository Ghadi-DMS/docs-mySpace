---
read_when:
    - Implementar o actualizar clientes WS de gateway
    - Depurar desajustes de protocolo o fallos de conexión
    - Regenerar esquema/modelos del protocolo
summary: 'Protocolo WebSocket de Gateway: handshake, frames, control de versiones'
title: Protocolo de Gateway
x-i18n:
    generated_at: "2026-04-05T12:43:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: c37f5b686562dda3ba3516ac6982ad87b2f01d8148233284e9917099c6e96d87
    source_path: gateway/protocol.md
    workflow: 15
---

# Protocolo de Gateway (WebSocket)

El protocolo WS de Gateway es el **único plano de control + transporte de nodos** para
OpenClaw. Todos los clientes (CLI, interfaz web, app de macOS, nodos iOS/Android, nodos
headless) se conectan por WebSocket y declaran su **rol** + **ámbito** en el
momento del handshake.

## Transporte

- WebSocket, frames de texto con cargas JSON.
- El primer frame **debe** ser una solicitud `connect`.

## Handshake (connect)

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

Durante la transferencia de bootstrap de confianza, `hello-ok.auth` también puede incluir entradas
adicionales de rol acotado en `deviceTokens`:

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

Para el flujo integrado de bootstrap de nodo/operador, el token principal del nodo permanece con
`scopes: []` y cualquier token de operador transferido permanece acotado a la lista de permitidos
del operador de bootstrap (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Las comprobaciones de ámbito del bootstrap siguen
con prefijo de rol: las entradas de operador solo satisfacen solicitudes de operador, y los roles que no son de operador
siguen necesitando ámbitos bajo su propio prefijo de rol.

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

## Encapsulado

- **Solicitud**: `{type:"req", id, method, params}`
- **Respuesta**: `{type:"res", id, ok, payload|error}`
- **Evento**: `{type:"event", event, payload, seq?, stateVersion?}`

Los métodos con efectos secundarios requieren **claves de idempotencia** (consulta el esquema).

## Roles + ámbitos

### Roles

- `operator` = cliente del plano de control (CLI/UI/automatización).
- `node` = host de capacidades (camera/screen/canvas/system.run).

### Ámbitos (operator)

Ámbitos comunes:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` con `includeSecrets: true` requiere `operator.talk.secrets`
(o `operator.admin`).

Los métodos RPC de gateway registrados por plugins pueden solicitar su propio ámbito de operador, pero
los prefijos reservados del núcleo para administración (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre se resuelven a `operator.admin`.

El ámbito del método es solo la primera puerta. Algunos comandos slash alcanzados mediante
`chat.send` aplican comprobaciones de nivel de comando más estrictas además. Por ejemplo, las
escrituras persistentes con `/config set` y `/config unset` requieren `operator.admin`.

`node.pair.approve` también tiene una comprobación adicional de ámbito en el momento de aprobación además del
ámbito base del método:

- solicitudes sin comando: `operator.pairing`
- solicitudes con comandos de nodo que no son exec: `operator.pairing` + `operator.write`
- solicitudes que incluyen `system.run`, `system.run.prepare` o `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Los nodos declaran afirmaciones de capacidad al conectarse:

- `caps`: categorías de capacidad de alto nivel.
- `commands`: lista de permitidos de comandos para invoke.
- `permissions`: conmutadores granulares (por ejemplo `screen.record`, `camera.capture`).

La Gateway trata estas como **afirmaciones** y aplica listas de permitidos del lado del servidor.

## Presencia

- `system-presence` devuelve entradas indexadas por identidad del dispositivo.
- Las entradas de presencia incluyen `deviceId`, `roles` y `scopes` para que las interfaces puedan mostrar una única fila por dispositivo
  incluso cuando se conecta como **operator** y **node** a la vez.

## Familias comunes de métodos RPC

Esta página no es un volcado completo generado, pero la superficie WS pública es más amplia
que los ejemplos anteriores de handshake/autenticación. Estas son las principales familias de métodos que la
Gateway expone hoy.

`hello-ok.features.methods` es una lista conservadora de descubrimiento creada a partir de
`src/gateway/server-methods-list.ts` más los exports de métodos de plugins/canales cargados.
Trátala como descubrimiento de funciones, no como un volcado generado de todos los helpers invocables
implementados en `src/gateway/server-methods/*.ts`.

### Sistema e identidad

- `health` devuelve la instantánea de salud de la gateway en caché o sondeada recientemente.
- `status` devuelve el resumen de la gateway al estilo de `/status`; los campos sensibles se
  incluyen solo para clientes operator con ámbito de administración.
- `gateway.identity.get` devuelve la identidad del dispositivo gateway usada por relay y
  los flujos de emparejamiento.
- `system-presence` devuelve la instantánea actual de presencia para dispositivos
  operator/node conectados.
- `system-event` agrega un evento del sistema y puede actualizar/difundir el
  contexto de presencia.
- `last-heartbeat` devuelve el evento de heartbeat persistido más reciente.
- `set-heartbeats` activa o desactiva el procesamiento de heartbeat en la gateway.

### Modelos y uso

- `models.list` devuelve el catálogo de modelos permitidos en runtime.
- `usage.status` devuelve ventanas de uso del proveedor/resúmenes de cuota restante.
- `usage.cost` devuelve resúmenes agregados de uso de costes para un intervalo de fechas.
- `doctor.memory.status` devuelve la preparación de memoria vectorial / embeddings para el
  espacio de trabajo activo del agente predeterminado.
- `sessions.usage` devuelve resúmenes de uso por sesión.
- `sessions.usage.timeseries` devuelve una serie temporal de uso para una sesión.
- `sessions.usage.logs` devuelve entradas del registro de uso para una sesión.

### Canales y helpers de inicio de sesión

- `channels.status` devuelve resúmenes de estado de canales/plugins integrados y empaquetados.
- `channels.logout` cierra sesión en un canal/cuenta específico donde el canal
  admite logout.
- `web.login.start` inicia un flujo de inicio de sesión por QR/web para el
  proveedor actual de canal web compatible con QR.
- `web.login.wait` espera a que ese flujo de inicio de sesión por QR/web se complete y arranca el
  canal si tiene éxito.
- `push.test` envía una notificación push APNs de prueba a un nodo iOS registrado.
- `voicewake.get` devuelve los triggers almacenados de palabra de activación.
- `voicewake.set` actualiza los triggers de palabra de activación y difunde el cambio.

### Mensajería y registros

- `send` es la RPC de entrega saliente directa para envíos dirigidos a
  canal/cuenta/hilo fuera del ejecutor de chat.
- `logs.tail` devuelve la cola configurada del registro de archivos de la gateway con cursor/límite y
  controles de bytes máximos.

### Talk y TTS

- `talk.config` devuelve la carga de configuración efectiva de Talk; `includeSecrets`
  requiere `operator.talk.secrets` (o `operator.admin`).
- `talk.mode` establece/difunde el estado actual del modo Talk para clientes de WebChat/Control UI.
- `talk.speak` sintetiza voz mediante el proveedor de voz activo de Talk.
- `tts.status` devuelve el estado habilitado de TTS, proveedor activo, proveedores fallback,
  y estado de configuración del proveedor.
- `tts.providers` devuelve el inventario visible de proveedores TTS.
- `tts.enable` y `tts.disable` activan o desactivan el estado de preferencias TTS.
- `tts.setProvider` actualiza el proveedor TTS preferido.
- `tts.convert` ejecuta una conversión puntual de texto a voz.

### Secrets, configuración, actualización y asistente

- `secrets.reload` vuelve a resolver los SecretRefs activos e intercambia el estado de secretos en runtime
  solo si todo tiene éxito.
- `secrets.resolve` resuelve asignaciones de secretos dirigidas a comandos para un conjunto específico
  de comando/destino.
- `config.get` devuelve la instantánea y el hash de la configuración actual.
- `config.set` escribe una carga de configuración validada.
- `config.patch` fusiona una actualización parcial de configuración.
- `config.apply` valida + reemplaza la carga completa de configuración.
- `config.schema` devuelve la carga activa del esquema de configuración usada por Control UI y
  herramientas CLI: esquema, `uiHints`, versión y metadatos de generación, incluidos
  metadatos de esquemas de plugins + canales cuando el runtime puede cargarlos. El esquema
  incluye metadatos de campo `title` / `description` derivados de las mismas etiquetas
  y texto de ayuda usados por la interfaz, incluidas ramas de composición de objetos anidados,
  comodines, elementos de arreglo y `anyOf` / `oneOf` / `allOf` cuando existe
  documentación de campo coincidente.
- `config.schema.lookup` devuelve una carga de búsqueda delimitada a una ruta para una ruta de configuración:
  ruta normalizada, un nodo superficial del esquema, `hint` coincidente + `hintPath`, y
  resúmenes inmediatos de hijos para navegación detallada en UI/CLI.
  - Los nodos de esquema de búsqueda conservan la documentación orientada al usuario y campos comunes de validación:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    límites numéricos/de cadena/de arreglo/de objeto, y banderas booleanas como
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Los resúmenes de hijos exponen `key`, `path` normalizada, `type`, `required`,
    `hasChildren`, más `hint` / `hintPath` coincidentes.
- `update.run` ejecuta el flujo de actualización de la gateway y programa un reinicio solo cuando
  la actualización en sí tuvo éxito.
- `wizard.start`, `wizard.next`, `wizard.status` y `wizard.cancel` exponen el
  asistente de onboarding mediante WS RPC.

### Familias principales existentes

#### Helpers de agente y espacio de trabajo

- `agents.list` devuelve las entradas de agentes configuradas.
- `agents.create`, `agents.update` y `agents.delete` gestionan registros de agentes y
  la conexión con el espacio de trabajo.
- `agents.files.list`, `agents.files.get` y `agents.files.set` gestionan los
  archivos bootstrap del espacio de trabajo expuestos para un agente.
- `agent.identity.get` devuelve la identidad efectiva del asistente para un agente o
  sesión.
- `agent.wait` espera a que una ejecución termine y devuelve la instantánea terminal cuando
  está disponible.

#### Control de sesiones

- `sessions.list` devuelve el índice actual de sesiones.
- `sessions.subscribe` y `sessions.unsubscribe` activan o desactivan
  suscripciones a eventos de cambio de sesión para el cliente WS actual.
- `sessions.messages.subscribe` y `sessions.messages.unsubscribe` activan o desactivan
  suscripciones a eventos de transcripción/mensajes para una sesión.
- `sessions.preview` devuelve vistas previas acotadas de transcripciones para claves de sesión específicas.
- `sessions.resolve` resuelve o canoniza un destino de sesión.
- `sessions.create` crea una nueva entrada de sesión.
- `sessions.send` envía un mensaje a una sesión existente.
- `sessions.steer` es la variante de interrupción y redirección para una sesión activa.
- `sessions.abort` aborta trabajo activo para una sesión.
- `sessions.patch` actualiza metadatos/sobrescrituras de sesión.
- `sessions.reset`, `sessions.delete` y `sessions.compact` realizan mantenimiento de sesiones.
- `sessions.get` devuelve la fila completa almacenada de la sesión.
- la ejecución de chat sigue usando `chat.history`, `chat.send`, `chat.abort` y
  `chat.inject`.
- `chat.history` está normalizado para visualización en clientes de UI: las etiquetas de directiva inline se eliminan del texto visible, las cargas XML de llamadas de herramientas en texto plano (incluidas
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>`, y
  bloques truncados de llamadas de herramientas) y los tokens de control del modelo filtrados en ASCII/ancho completo
  se eliminan, las filas de asistente compuestas solo por tokens silenciosos como `NO_REPLY` / `no_reply`
  exactos se omiten, y las filas sobredimensionadas pueden reemplazarse con placeholders.

#### Emparejamiento de dispositivos y tokens de dispositivo

- `device.pair.list` devuelve dispositivos emparejados pendientes y aprobados.
- `device.pair.approve`, `device.pair.reject` y `device.pair.remove` gestionan
  registros de emparejamiento de dispositivos.
- `device.token.rotate` rota un token de dispositivo emparejado dentro de sus límites aprobados de rol
  y ámbito.
- `device.token.revoke` revoca un token de dispositivo emparejado.

#### Emparejamiento de nodos, invoke y trabajo pendiente

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` y `node.pair.verify` cubren el emparejamiento de nodos y la verificación
  de bootstrap.
- `node.list` y `node.describe` devuelven el estado de nodos conocidos/conectados.
- `node.rename` actualiza la etiqueta de un nodo emparejado.
- `node.invoke` reenvía un comando a un nodo conectado.
- `node.invoke.result` devuelve el resultado de una solicitud invoke.
- `node.event` transporta eventos originados en el nodo de vuelta a la gateway.
- `node.canvas.capability.refresh` actualiza tokens de capacidad de canvas delimitados.
- `node.pending.pull` y `node.pending.ack` son las APIs de cola para nodos conectados.
- `node.pending.enqueue` y `node.pending.drain` gestionan trabajo pendiente duradero
  para nodos offline/desconectados.

#### Familias de aprobaciones

- `exec.approval.request` y `exec.approval.resolve` cubren solicitudes puntuales
  de aprobación de exec.
- `exec.approval.waitDecision` espera una aprobación exec pendiente y devuelve
  la decisión final (o `null` por timeout).
- `exec.approvals.get` y `exec.approvals.set` gestionan instantáneas de política de aprobación exec de la gateway.
- `exec.approvals.node.get` y `exec.approvals.node.set` gestionan políticas locales de aprobación exec
  del nodo mediante comandos relay del nodo.
- `plugin.approval.request`, `plugin.approval.waitDecision` y
  `plugin.approval.resolve` cubren flujos de aprobación definidos por plugins.

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
- `sessions.changed`: cambió el índice de sesiones o sus metadatos.
- `presence`: actualizaciones de la instantánea de presencia del sistema.
- `tick`: evento periódico de keepalive / liveness.
- `health`: actualización de la instantánea de salud de la gateway.
- `heartbeat`: actualización del flujo de eventos de heartbeat.
- `cron`: evento de cambio de ejecución/trabajo cron.
- `shutdown`: notificación de apagado de la gateway.
- `node.pair.requested` / `node.pair.resolved`: ciclo de vida del emparejamiento de nodos.
- `node.invoke.request`: difusión de una solicitud invoke de nodo.
- `device.pair.requested` / `device.pair.resolved`: ciclo de vida de dispositivos emparejados.
- `voicewake.changed`: cambió la configuración de triggers de palabra de activación.
- `exec.approval.requested` / `exec.approval.resolved`: ciclo de vida de aprobación
  de exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: ciclo de vida de aprobación
  de plugins.

### Métodos helper para nodos

- Los nodos pueden llamar a `skills.bins` para obtener la lista actual de ejecutables de Skills
  para comprobaciones automáticas de permitidos.

### Métodos helper para operadores

- Los operadores pueden llamar a `tools.catalog` (`operator.read`) para obtener el catálogo de herramientas en runtime para un
  agente. La respuesta incluye herramientas agrupadas y metadatos de procedencia:
  - `source`: `core` o `plugin`
  - `pluginId`: plugin propietario cuando `source="plugin"`
  - `optional`: si una herramienta de plugin es opcional
- Los operadores pueden llamar a `tools.effective` (`operator.read`) para obtener el inventario efectivo de herramientas en runtime
  para una sesión.
  - `sessionKey` es obligatorio.
  - La gateway deriva el contexto confiable de runtime del lado del servidor a partir de la sesión en lugar de aceptar
    autenticación o contexto de entrega suministrados por quien llama.
  - La respuesta está delimitada a la sesión y refleja lo que la conversación activa puede usar ahora mismo,
    incluidas herramientas del núcleo, de plugins y de canal.
- Los operadores pueden llamar a `skills.status` (`operator.read`) para obtener el inventario visible
  de Skills para un agente.
  - `agentId` es opcional; omítelo para leer el espacio de trabajo del agente predeterminado.
  - La respuesta incluye aptitud, requisitos faltantes, comprobaciones de configuración y
    opciones de instalación saneadas sin exponer valores secretos sin procesar.
- Los operadores pueden llamar a `skills.search` y `skills.detail` (`operator.read`) para metadatos
  de descubrimiento de ClawHub.
- Los operadores pueden llamar a `skills.install` (`operator.admin`) en dos modos:
  - Modo ClawHub: `{ source: "clawhub", slug, version?, force? }` instala una
    carpeta de Skill en el directorio `skills/` del espacio de trabajo del agente predeterminado.
  - Modo instalador de gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    ejecuta una acción declarada `metadata.openclaw.install` en el host de la gateway.
- Los operadores pueden llamar a `skills.update` (`operator.admin`) en dos modos:
  - El modo ClawHub actualiza un slug rastreado o todas las instalaciones rastreadas de ClawHub en
    el espacio de trabajo del agente predeterminado.
  - El modo de configuración aplica parches a valores `skills.entries.<skillKey>` como `enabled`,
    `apiKey` y `env`.

## Aprobaciones de exec

- Cuando una solicitud exec necesita aprobación, la gateway difunde `exec.approval.requested`.
- Los clientes operator resuelven llamando a `exec.approval.resolve` (requiere el ámbito `operator.approvals`).
- Para `host=node`, `exec.approval.request` debe incluir `systemRunPlan` (metadatos canónicos de `argv`/`cwd`/`rawCommand`/sesión). Las solicitudes sin `systemRunPlan` se rechazan.
- Después de la aprobación, las llamadas reenviadas `node.invoke system.run` reutilizan ese
  `systemRunPlan` canónico como contexto autorizado de comando/cwd/sesión.
- Si quien llama modifica `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` entre prepare y el reenvío final aprobado de `system.run`, la
  gateway rechaza la ejecución en lugar de confiar en la carga modificada.

## Fallback de entrega del agente

- Las solicitudes `agent` pueden incluir `deliver=true` para solicitar entrega saliente.
- `bestEffortDeliver=false` mantiene el comportamiento estricto: los destinos de entrega no resueltos o solo internos devuelven `INVALID_REQUEST`.
- `bestEffortDeliver=true` permite fallback a ejecución solo en sesión cuando no puede resolverse ninguna ruta externa de entrega (por ejemplo, sesiones internas/webchat o configuraciones ambiguas de varios canales).

## Control de versiones

- `PROTOCOL_VERSION` vive en `src/gateway/protocol/schema.ts`.
- Los clientes envían `minProtocol` + `maxProtocol`; el servidor rechaza incompatibilidades.
- Los esquemas + modelos se generan a partir de definiciones TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Autenticación

- La autenticación de gateway con secreto compartido usa `connect.params.auth.token` o
  `connect.params.auth.password`, según el modo de autenticación configurado.
- Los modos con identidad como Tailscale Serve
  (`gateway.auth.allowTailscale: true`) o `gateway.auth.mode: "trusted-proxy"` sin loopback
  satisfacen la comprobación de autenticación de connect a partir de
  encabezados de solicitud en lugar de `connect.params.auth.*`.
- El ingreso privado `gateway.auth.mode: "none"` omite por completo la autenticación connect con secreto compartido;
  no expongas ese modo en ingresos públicos/no confiables.
- Después del emparejamiento, la Gateway emite un **token de dispositivo** delimitado al rol + ámbitos
  de la conexión. Se devuelve en `hello-ok.auth.deviceToken` y el cliente debe
  persistirlo para conexiones futuras.
- Los clientes deben persistir el `hello-ok.auth.deviceToken` principal después de cualquier
  conexión exitosa.
- Volver a conectar con ese token de dispositivo **almacenado** también debe reutilizar el conjunto de ámbitos aprobados almacenado para ese token. Esto conserva el acceso de lectura/sondeo/estado
  que ya se concedió y evita contraer silenciosamente las reconexiones a un
  ámbito implícito más estrecho solo de administración.
- La precedencia normal de autenticación connect es primero token/contraseña compartidos explícitos, luego
  `deviceToken` explícito, luego token por dispositivo almacenado, y después token bootstrap.
- Las entradas adicionales `hello-ok.auth.deviceTokens` son tokens de transferencia bootstrap.
  Persístelos solo cuando la conexión usó autenticación bootstrap en un transporte de confianza
  como `wss://` o loopback/emparejamiento local.
- Si un cliente proporciona un `deviceToken` **explícito** o `scopes` explícitos, ese
  conjunto de ámbitos solicitado por quien llama sigue siendo la autoridad; los ámbitos en caché solo
  se reutilizan cuando el cliente está reutilizando el token por dispositivo almacenado.
- Los tokens de dispositivo pueden rotarse/revocarse mediante `device.token.rotate` y
  `device.token.revoke` (requiere el ámbito `operator.pairing`).
- La emisión/rotación de tokens permanece acotada al conjunto de roles aprobados registrado en
  la entrada de emparejamiento de ese dispositivo; rotar un token no puede ampliar el dispositivo a
  un rol que la aprobación de emparejamiento nunca concedió.
- Para sesiones de token de dispositivo emparejado, la gestión de dispositivos está delimitada al propio dispositivo salvo que la persona que llama también tenga `operator.admin`: quienes no son administradores solo pueden eliminar/revocar/rotar
  su **propia** entrada de dispositivo.
- `device.token.rotate` también comprueba el conjunto de ámbitos operator solicitado frente a los
  ámbitos actuales de la sesión de quien llama. Quienes no son administradores no pueden rotar un token a
  un conjunto de ámbitos operator más amplio que el que ya tienen.
- Los fallos de autenticación incluyen `error.details.code` más sugerencias de recuperación:
  - `error.details.canRetryWithDeviceToken` (booleano)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Comportamiento del cliente para `AUTH_TOKEN_MISMATCH`:
  - Los clientes de confianza pueden intentar un único reintento acotado con un token por dispositivo en caché.
  - Si ese reintento falla, los clientes deben detener los bucles automáticos de reconexión y mostrar instrucciones de acción al operador.

## Identidad del dispositivo + emparejamiento

- Los nodos deben incluir una identidad estable de dispositivo (`device.id`) derivada de la
  huella digital de un par de claves.
- Las gateways emiten tokens por dispositivo + rol.
- Se requieren aprobaciones de emparejamiento para nuevos IDs de dispositivo salvo que la autoaprobación
  local esté habilitada.
- La autoaprobación de emparejamiento está centrada en conexiones directas por loopback local.
- OpenClaw también tiene una ruta limitada de autoconexión backend/contenedor-local para
  flujos helper de secreto compartido de confianza.
- Las conexiones tailnet o LAN desde el mismo host siguen tratándose como remotas para el emparejamiento y
  requieren aprobación.
- Todos los clientes WS deben incluir identidad `device` durante `connect` (operator + node).
  Control UI puede omitirla solo en estos modos:
  - `gateway.controlUi.allowInsecureAuth=true` para compatibilidad con HTTP inseguro solo en localhost.
  - autenticación exitosa de operator Control UI con `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (modo de emergencia, degradación grave de seguridad).
- Todas las conexiones deben firmar el nonce `connect.challenge` proporcionado por el servidor.

### Diagnósticos de migración de autenticación de dispositivos

Para clientes heredados que aún usan el comportamiento de firma previo al desafío, `connect` ahora devuelve
códigos de detalle `DEVICE_AUTH_*` en `error.details.code` con un `error.details.reason` estable.

Fallos comunes de migración:

| Mensaje                     | details.code                     | details.reason           | Significado                                        |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | El cliente omitió `device.nonce` (o lo envió vacío). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | El cliente firmó con un nonce obsoleto/incorrecto. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | La carga de la firma no coincide con la carga v2.  |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | La marca de tiempo firmada está fuera del desfase permitido. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` no coincide con la huella de la clave pública. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Falló el formato/canonicalización de la clave pública. |

Objetivo de la migración:

- Espera siempre a `connect.challenge`.
- Firma la carga v2 que incluye el nonce del servidor.
- Envía el mismo nonce en `connect.params.device.nonce`.
- La carga de firma preferida es `v3`, que vincula `platform` y `deviceFamily`
  además de campos de dispositivo/cliente/rol/ámbitos/token/nonce.
- Las firmas heredadas `v2` siguen aceptándose por compatibilidad, pero la fijación de metadatos
  del dispositivo emparejado sigue controlando la política de comandos en la reconexión.

## TLS + pinning

- TLS es compatible para conexiones WS.
- Los clientes pueden fijar opcionalmente la huella del certificado de la gateway (consulta la configuración `gateway.tls`
  más `gateway.remote.tlsFingerprint` o CLI `--tls-fingerprint`).

## Alcance

Este protocolo expone la **API completa de la gateway** (status, channels, models, chat,
agent, sessions, nodes, approvals, etc.). La superficie exacta está definida por los
esquemas TypeBox en `src/gateway/protocol/schema.ts`.
