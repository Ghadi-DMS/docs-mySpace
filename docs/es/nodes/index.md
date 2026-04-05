---
read_when:
    - Emparejar nodos iOS/Android con una gateway
    - Usar canvas/camera del nodo para el contexto del agente
    - Añadir nuevos comandos de nodo o helpers de CLI
summary: 'Nodos: emparejamiento, capacidades, permisos y helpers de CLI para canvas/camera/screen/device/notifications/system'
title: Nodos
x-i18n:
    generated_at: "2026-04-05T12:47:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 201be0e13cb6d39608f0bbd40fd02333f68bd44f588538d1016fe864db7e038e
    source_path: nodes/index.md
    workflow: 15
---

# Nodos

Un **nodo** es un dispositivo complementario (macOS/iOS/Android/headless) que se conecta al **WebSocket** de Gateway (el mismo puerto que usan los operadores) con `role: "node"` y expone una superficie de comandos (por ejemplo `canvas.*`, `camera.*`, `device.*`, `notifications.*`, `system.*`) mediante `node.invoke`. Detalles del protocolo: [Protocolo de Gateway](/gateway/protocol).

Transporte heredado: [Protocolo de bridge](/gateway/bridge-protocol) (TCP JSONL;
solo histórico para los nodos actuales).

macOS también puede ejecutarse en **modo nodo**: la app de barra de menú se conecta al servidor WS de la Gateway y expone sus comandos locales de canvas/camera como un nodo (de modo que `openclaw nodes …` funciona contra este Mac).

Notas:

- Los nodos son **periféricos**, no gateways. No ejecutan el servicio de gateway.
- Los mensajes de Telegram/WhatsApp/etc. llegan a la **gateway**, no a los nodos.
- Runbook de solución de problemas: [/nodes/troubleshooting](/nodes/troubleshooting)

## Emparejamiento + estado

**Los nodos WS usan emparejamiento de dispositivos.** Los nodos presentan una identidad de dispositivo durante `connect`; la Gateway
crea una solicitud de emparejamiento de dispositivo para `role: node`. Apruébala mediante la CLI de devices (o la UI).

CLI rápida:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

Si un nodo vuelve a intentarlo con detalles de autenticación modificados (rol/scopes/clave pública), la solicitud
pendiente anterior se reemplaza y se crea un nuevo `requestId`. Vuelve a ejecutar
`openclaw devices list` antes de aprobar.

Notas:

- `nodes status` marca un nodo como **paired** cuando el rol de emparejamiento del dispositivo incluye `node`.
- El registro de emparejamiento del dispositivo es el contrato duradero de roles aprobados. La
  rotación del token permanece dentro de ese contrato; no puede ampliar un nodo emparejado a un
  rol distinto que la aprobación de emparejamiento nunca concedió.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/rename`) es un almacén de emparejamiento de nodos separado y gestionado por la gateway;
  **no** controla el handshake `connect` de WS.
- El ámbito de aprobación sigue los comandos declarados en la solicitud pendiente:
  - solicitud sin comando: `operator.pairing`
  - comandos de nodo sin exec: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## Host de nodo remoto (system.run)

Usa un **host de nodo** cuando tu Gateway se ejecuta en una máquina y quieres que los comandos
se ejecuten en otra. El modelo sigue hablando con la **gateway**; la gateway
reenvía las llamadas `exec` al **host de nodo** cuando se selecciona `host=node`.

### Qué se ejecuta en cada lugar

- **Host de Gateway**: recibe mensajes, ejecuta el modelo, enruta llamadas de herramientas.
- **Host de nodo**: ejecuta `system.run`/`system.which` en la máquina del nodo.
- **Aprobaciones**: se aplican en el host de nodo mediante `~/.openclaw/exec-approvals.json`.

Nota sobre aprobaciones:

- Las ejecuciones de nodo respaldadas por aprobación vinculan el contexto exacto de la solicitud.
- Para ejecuciones directas de shell/archivos de runtime, OpenClaw también intenta vincular, en la medida de lo posible, un único
  operando de archivo local concreto y deniega la ejecución si ese archivo cambia antes de ejecutarse.
- Si OpenClaw no puede identificar exactamente un archivo local concreto para un comando de intérprete/runtime,
  la ejecución respaldada por aprobación se deniega en lugar de fingir cobertura total del runtime. Usa sandboxing,
  hosts separados o una lista de permitidos/flujo de confianza explícito para una semántica más amplia de intérpretes.

### Iniciar un host de nodo (primer plano)

En la máquina del nodo:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

### Gateway remota mediante túnel SSH (enlace loopback)

Si la Gateway está enlazada a loopback (`gateway.bind=loopback`, valor predeterminado en modo local),
los hosts de nodo remotos no pueden conectarse directamente. Crea un túnel SSH y apunta el
host de nodo al extremo local del túnel.

Ejemplo (host de nodo -> host de gateway):

```bash
# Terminal A (mantener en ejecución): reenviar el puerto local 18790 -> gateway 127.0.0.1:18789
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# Terminal B: exportar el token de gateway y conectarse a través del túnel
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

Notas:

- `openclaw node run` admite autenticación por token o contraseña.
- Se prefieren variables de entorno: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- El fallback en configuración es `gateway.auth.token` / `gateway.auth.password`.
- En modo local, el host de nodo ignora intencionadamente `gateway.remote.token` / `gateway.remote.password`.
- En modo remoto, `gateway.remote.token` / `gateway.remote.password` son válidos según las reglas de precedencia remota.
- Si hay SecretRefs activos `gateway.auth.*` configurados pero no resueltos, la autenticación del host de nodo falla de forma cerrada.
- La resolución de autenticación del host de nodo solo respeta variables de entorno `OPENCLAW_GATEWAY_*`.

### Iniciar un host de nodo (servicio)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node restart
```

### Emparejar + nombrar

En el host de la gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Si el nodo vuelve a intentarlo con detalles de autenticación modificados, vuelve a ejecutar `openclaw devices list`
y aprueba el `requestId` actual.

Opciones de nombre:

- `--display-name` en `openclaw node run` / `openclaw node install` (se conserva en `~/.openclaw/node.json` en el nodo).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` (sobrescritura en la gateway).

### Añadir los comandos a la lista de permitidos

Las aprobaciones de exec son **por host de nodo**. Añade entradas a la lista de permitidos desde la gateway:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

Las aprobaciones viven en el host de nodo en `~/.openclaw/exec-approvals.json`.

### Apuntar exec al nodo

Configura los valores predeterminados (configuración de gateway):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.security allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

O por sesión:

```
/exec host=node security=allowlist node=<id-or-name>
```

Una vez establecido, cualquier llamada `exec` con `host=node` se ejecuta en el host de nodo (sujeta a la
lista de permitidos/aprobaciones del nodo).

`host=auto` no elegirá implícitamente el nodo por sí solo, pero se permite una solicitud explícita por llamada `host=node` desde `auto`. Si quieres que exec en el nodo sea el valor predeterminado para la sesión, establece `tools.exec.host=node` o `/exec host=node ...` explícitamente.

Relacionado:

- [CLI de host de nodo](/cli/node)
- [Herramienta Exec](/tools/exec)
- [Exec approvals](/tools/exec-approvals)

## Invocar comandos

Nivel bajo (RPC sin procesar):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

Existen helpers de más alto nivel para los flujos de trabajo comunes de “dar al agente un adjunto MEDIA”.

## Capturas de pantalla (instantáneas de canvas)

Si el nodo está mostrando el Canvas (WebView), `canvas.snapshot` devuelve `{ format, base64 }`.

Helper de CLI (escribe en un archivo temporal e imprime `MEDIA:<path>`):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Controles de Canvas

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

Notas:

- `canvas present` acepta URL o rutas de archivo locales (`--target`), además de `--x/--y/--width/--height` opcionales para el posicionamiento.
- `canvas eval` acepta JS inline (`--js`) o un argumento posicional.

### A2UI (Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

Notas:

- Solo se admite A2UI v0.8 JSONL (v0.9/createSurface se rechaza).

## Fotos + videos (camera del nodo)

Fotos (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # predeterminado: ambas orientaciones (2 líneas MEDIA)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
```

Clips de video (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

Notas:

- El nodo debe estar en **primer plano** para `canvas.*` y `camera.*` (las llamadas en segundo plano devuelven `NODE_BACKGROUND_UNAVAILABLE`).
- La duración del clip está limitada (actualmente `<= 60s`) para evitar cargas base64 demasiado grandes.
- Android solicitará permisos `CAMERA`/`RECORD_AUDIO` cuando sea posible; si se deniegan, fallará con `*_PERMISSION_REQUIRED`.

## Grabaciones de pantalla (nodos)

Los nodos compatibles exponen `screen.record` (`mp4`). Ejemplo:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

Notas:

- La disponibilidad de `screen.record` depende de la plataforma del nodo.
- Las grabaciones de pantalla están limitadas a `<= 60s`.
- `--no-audio` desactiva la captura del micrófono en las plataformas compatibles.
- Usa `--screen <index>` para seleccionar una pantalla cuando haya varias disponibles.

## Ubicación (nodos)

Los nodos exponen `location.get` cuando la ubicación está habilitada en la configuración.

Helper de CLI:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Notas:

- La ubicación está **desactivada por defecto**.
- “Always” requiere permiso del sistema; la obtención en segundo plano es best-effort.
- La respuesta incluye lat/lon, precisión (metros) y marca de tiempo.

## SMS (nodos Android)

Los nodos Android pueden exponer `sms.send` cuando el usuario concede el permiso **SMS** y el dispositivo admite telefonía.

Invocación de bajo nivel:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

Notas:

- La solicitud de permiso debe aceptarse en el dispositivo Android antes de que se anuncie la capacidad.
- Los dispositivos solo Wi‑Fi sin telefonía no anunciarán `sms.send`.

## Comandos de dispositivo Android + datos personales

Los nodos Android pueden anunciar familias de comandos adicionales cuando las capacidades correspondientes están habilitadas.

Familias disponibles:

- `device.status`, `device.info`, `device.permissions`, `device.health`
- `notifications.list`, `notifications.actions`
- `photos.latest`
- `contacts.search`, `contacts.add`
- `calendar.events`, `calendar.add`
- `callLog.search`
- `sms.search`
- `motion.activity`, `motion.pedometer`

Ejemplos de invocación:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

Notas:

- Los comandos de movimiento están controlados por capacidades según los sensores disponibles.

## Comandos del sistema (host de nodo / nodo Mac)

El nodo de macOS expone `system.run`, `system.notify` y `system.execApprovals.get/set`.
El host de nodo headless expone `system.run`, `system.which` y `system.execApprovals.get/set`.

Ejemplos:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway ready"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"name":"git"}'
```

Notas:

- `system.run` devuelve stdout/stderr/código de salida en la carga.
- La ejecución de shell ahora pasa por la herramienta `exec` con `host=node`; `nodes` sigue siendo la superficie de RPC directa para comandos explícitos del nodo.
- `nodes invoke` no expone `system.run` ni `system.run.prepare`; estos permanecen solo en la ruta exec.
- La ruta exec prepara un `systemRunPlan` canónico antes de la aprobación. Una vez
  concedida una aprobación, la gateway reenvía ese plan almacenado, no ningún
  campo command/cwd/session editado posteriormente por quien llama.
- `system.notify` respeta el estado del permiso de notificaciones en la app de macOS.
- Los metadatos de nodo `platform` / `deviceFamily` no reconocidos usan una lista de permitidos conservadora que excluye `system.run` y `system.which`. Si intencionadamente necesitas esos comandos para una plataforma desconocida, añádelos explícitamente con `gateway.nodes.allowCommands`.
- `system.run` admite `--cwd`, `--env KEY=VAL`, `--command-timeout` y `--needs-screen-recording`.
- Para wrappers de shell (`bash|sh|zsh ... -c/-lc`), los valores `--env` delimitados a la solicitud se reducen a una lista de permitidos explícita (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Para decisiones de permitir siempre en modo allowlist, los wrappers de despacho conocidos (`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservan las rutas internas del ejecutable en lugar de las rutas del wrapper. Si el desempaquetado no es seguro, no se conserva automáticamente ninguna entrada en la lista de permitidos.
- En hosts de nodo Windows en modo allowlist, las ejecuciones mediante wrappers de shell con `cmd.exe /c` requieren aprobación (la entrada en la lista de permitidos por sí sola no autoriza automáticamente el formato wrapper).
- `system.notify` admite `--priority <passive|active|timeSensitive>` y `--delivery <system|overlay|auto>`.
- Los hosts de nodo ignoran las sobrescrituras de `PATH` y eliminan claves peligrosas de inicio/shell (`DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`). Si necesitas entradas adicionales de PATH, configura el entorno del servicio del host de nodo (o instala herramientas en ubicaciones estándar) en lugar de pasar `PATH` mediante `--env`.
- En modo nodo de macOS, `system.run` está controlado por exec approvals en la app de macOS (Settings → Exec approvals).
  Ask/allowlist/full se comportan igual que en el host de nodo headless; las solicitudes denegadas devuelven `SYSTEM_RUN_DENIED`.
- En el host de nodo headless, `system.run` está controlado por exec approvals (`~/.openclaw/exec-approvals.json`).

## Vinculación de nodo para exec

Cuando hay varios nodos disponibles, puedes vincular exec a un nodo específico.
Esto establece el nodo predeterminado para `exec host=node` (y puede sobrescribirse por agente).

Valor predeterminado global:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

Sobrescritura por agente:

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Desactívalo para permitir cualquier nodo:

```bash
openclaw config unset tools.exec.node
openclaw config unset agents.list[0].tools.exec.node
```

## Mapa de permisos

Los nodos pueden incluir un mapa `permissions` en `node.list` / `node.describe`, indexado por nombre de permiso (por ejemplo `screenRecording`, `accessibility`) con valores booleanos (`true` = concedido).

## Host de nodo headless (multiplataforma)

OpenClaw puede ejecutar un **host de nodo headless** (sin UI) que se conecta al
WebSocket de Gateway y expone `system.run` / `system.which`. Esto es útil en Linux/Windows
o para ejecutar un nodo mínimo junto a un servidor.

Inícialo:

```bash
openclaw node run --host <gateway-host> --port 18789
```

Notas:

- El emparejamiento sigue siendo obligatorio (la Gateway mostrará una solicitud de emparejamiento de dispositivo).
- El host de nodo almacena su id de nodo, token, nombre para mostrar e información de conexión de gateway en `~/.openclaw/node.json`.
- Las exec approvals se aplican localmente mediante `~/.openclaw/exec-approvals.json`
  (consulta [Exec approvals](/tools/exec-approvals)).
- En macOS, el host de nodo headless ejecuta `system.run` localmente por defecto. Establece
  `OPENCLAW_NODE_EXEC_HOST=app` para enrutar `system.run` a través del host exec de la app complementaria; añade
  `OPENCLAW_NODE_EXEC_FALLBACK=0` para exigir el host de la app y fallar de forma cerrada si no está disponible.
- Añade `--tls` / `--tls-fingerprint` cuando el WS de Gateway use TLS.

## Modo nodo en Mac

- La app de barra de menú de macOS se conecta al servidor WS de Gateway como nodo (de modo que `openclaw nodes …` funciona contra este Mac).
- En modo remoto, la app abre un túnel SSH para el puerto de Gateway y se conecta a `localhost`.
