---
read_when:
    - El centro de solución de problemas te indicó venir aquí para un diagnóstico más profundo
    - Necesitas secciones estables del runbook basadas en síntomas con comandos exactos
summary: Runbook profundo de solución de problemas para gateway, canales, automatización, nodos y navegador
title: Solución de problemas
x-i18n:
    generated_at: "2026-04-05T12:44:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 028226726e6adc45ca61d41510a953c4e21a3e85f3082af9e8085745c6ac3ec1
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# Solución de problemas del Gateway

Esta página es el runbook profundo.
Empieza en [/help/troubleshooting](/help/troubleshooting) si primero quieres el flujo rápido de triaje.

## Secuencia de comandos

Ejecuta estos primero, en este orden:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Señales esperadas de un estado saludable:

- `openclaw gateway status` muestra `Runtime: running` y `RPC probe: ok`.
- `openclaw doctor` no informa problemas bloqueantes de configuración/servicio.
- `openclaw channels status --probe` muestra el estado de transporte en vivo por cuenta y,
  donde sea compatible, resultados de sondeo/auditoría como `works` o `audit ok`.

## Anthropic 429 requiere uso extra para contexto largo

Usa esto cuando los registros/errores incluyan:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Busca lo siguiente:

- El modelo Anthropic Opus/Sonnet seleccionado tiene `params.context1m: true`.
- La credencial actual de Anthropic no es apta para uso de contexto largo.
- Las solicitudes fallan solo en sesiones largas/ejecuciones de modelo que necesitan la ruta beta de 1M.

Opciones de corrección:

1. Desactiva `context1m` para ese modelo y volver a la ventana de contexto normal.
2. Usa una clave API de Anthropic con facturación, o habilita Anthropic Extra Usage en la cuenta Anthropic OAuth/suscripción.
3. Configura modelos de respaldo para que las ejecuciones continúen cuando se rechacen las solicitudes de contexto largo de Anthropic.

Relacionado:

- [/providers/anthropic](/providers/anthropic)
- [/reference/token-use](/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Sin respuestas

Si los canales están activos pero nada responde, comprueba el enrutamiento y la política antes de volver a conectar nada.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Busca lo siguiente:

- Emparejamiento pendiente para remitentes de mensajes directos.
- Restricción por mención de grupo (`requireMention`, `mentionPatterns`).
- Desajustes en la lista de permitidos de canal/grupo.

Firmas comunes:

- `drop guild message (mention required` → mensaje de grupo ignorado hasta que haya mención.
- `pairing request` → el remitente necesita aprobación.
- `blocked` / `allowlist` → el remitente/canal fue filtrado por la política.

Relacionado:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/pairing](/channels/pairing)
- [/channels/groups](/channels/groups)

## Conectividad de la UI de control del panel

Cuando el panel/la UI de control no se conecta, valida la URL, el modo de autenticación y los supuestos de contexto seguro.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Busca lo siguiente:

- URL de sondeo y URL del panel correctas.
- Desajuste de modo de autenticación/token entre cliente y gateway.
- Uso de HTTP donde se requiere identidad del dispositivo.

Firmas comunes:

- `device identity required` → contexto no seguro o falta autenticación del dispositivo.
- `origin not allowed` → el `Origin` del navegador no está en `gateway.controlUi.allowedOrigins`
  (o te estás conectando desde un origen de navegador no loopback sin una
  lista de permitidos explícita).
- `device nonce required` / `device nonce mismatch` → el cliente no está completando el
  flujo de autenticación del dispositivo basado en desafío (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → el cliente firmó la carga útil incorrecta
  (o una marca de tiempo obsoleta) para el handshake actual.
- `AUTH_TOKEN_MISMATCH` con `canRetryWithDeviceToken=true` → el cliente puede hacer un reintento de confianza con el token de dispositivo en caché.
- Ese reintento con token en caché reutiliza el conjunto de ámbitos en caché almacenado con el
  token de dispositivo emparejado. Los llamadores con `deviceToken` explícito / `scopes` explícitos conservan
  su conjunto de ámbitos solicitado.
- Fuera de esa ruta de reintento, la precedencia de autenticación de conexión es
  primero token/contraseña compartidos explícitos, luego `deviceToken` explícito, luego token de dispositivo almacenado,
  y después token de bootstrap.
- En la ruta asíncrona de la UI de control de Tailscale Serve, los intentos fallidos para el mismo
  `{scope, ip}` se serializan antes de que el limitador registre el fallo. Por tanto, dos reintentos
  concurrentes incorrectos del mismo cliente pueden mostrar `retry later`
  en el segundo intento en lugar de dos simples desajustes.
- `too many failed authentication attempts (retry later)` desde un cliente loopback
  de origen de navegador → los fallos repetidos desde ese mismo `Origin` normalizado quedan
  bloqueados temporalmente; otro origen localhost usa un bucket independiente.
- `unauthorized` repetido después de ese reintento → deriva del token compartido/token de dispositivo; actualiza la configuración del token y vuelve a aprobar/rotar el token de dispositivo si es necesario.
- `gateway connect failed:` → objetivo de host/puerto/url incorrecto.

### Mapa rápido de códigos de detalle de autenticación

Usa `error.details.code` de la respuesta fallida de `connect` para elegir la siguiente acción:

| Código de detalle            | Significado                                             | Acción recomendada                                                                                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | El cliente no envió un token compartido requerido.      | Pega/establece el token en el cliente y vuelve a intentarlo. Para rutas del panel: `openclaw config get gateway.auth.token` y luego pégalo en la configuración de la UI de control.                                                                                                  |
| `AUTH_TOKEN_MISMATCH`        | El token compartido no coincide con el token de autenticación del gateway. | Si `canRetryWithDeviceToken=true`, permite un reintento de confianza. Los reintentos con token en caché reutilizan ámbitos aprobados almacenados; los llamadores con `deviceToken` / `scopes` explícitos conservan los ámbitos solicitados. Si sigue fallando, ejecuta la [lista de recuperación de deriva de token](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | El token por dispositivo en caché está obsoleto o revocado. | Rota/vuelve a aprobar el token de dispositivo usando la [CLI de devices](/cli/devices), luego vuelve a conectar.                                                                                                                                                                      |
| `PAIRING_REQUIRED`           | La identidad del dispositivo es conocida pero no está aprobada para este rol. | Aprueba la solicitud pendiente: `openclaw devices list` y luego `openclaw devices approve <requestId>`.                                                                                                                                                                               |

Comprobación de migración de autenticación de dispositivo v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Si los registros muestran errores de nonce/firma, actualiza el cliente que se conecta y verifica que:

1. espera `connect.challenge`
2. firma la carga útil vinculada al desafío
3. envía `connect.params.device.nonce` con el mismo nonce del desafío

Si `openclaw devices rotate` / `revoke` / `remove` se deniega inesperadamente:

- las sesiones con token de dispositivo emparejado solo pueden gestionar **su propio**
  dispositivo salvo que el llamador también tenga `operator.admin`
- `openclaw devices rotate --scope ...` solo puede solicitar ámbitos de operador que
  la sesión llamadora ya posea

Relacionado:

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/gateway/configuration) (modos de autenticación del gateway)
- [/gateway/trusted-proxy-auth](/gateway/trusted-proxy-auth)
- [/gateway/remote](/gateway/remote)
- [/cli/devices](/cli/devices)

## El servicio Gateway no se está ejecutando

Usa esto cuando el servicio está instalado pero el proceso no se mantiene activo.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # also scan system-level services
```

Busca lo siguiente:

- `Runtime: stopped` con pistas de salida.
- Desajuste de configuración del servicio (`Config (cli)` frente a `Config (service)`).
- Conflictos de puerto/listener.
- Instalaciones adicionales de launchd/systemd/schtasks cuando se usa `--deep`.
- Pistas de limpieza de `Other gateway-like services detected (best effort)`.

Firmas comunes:

- `Gateway start blocked: set gateway.mode=local` o `existing config is missing gateway.mode` → el modo gateway local no está habilitado, o el archivo de configuración fue sobrescrito y perdió `gateway.mode`. Corrección: establece `gateway.mode="local"` en tu configuración, o vuelve a ejecutar `openclaw onboard --mode local` / `openclaw setup` para volver a estampar la configuración esperada en modo local. Si ejecutas OpenClaw mediante Podman, la ruta predeterminada de configuración es `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → bind no loopback sin una ruta válida de autenticación del gateway (token/contraseña, o trusted-proxy donde esté configurado).
- `another gateway instance is already listening` / `EADDRINUSE` → conflicto de puerto.
- `Other gateway-like services detected (best effort)` → existen unidades launchd/systemd/schtasks obsoletas o paralelas. La mayoría de configuraciones deben mantener un gateway por máquina; si necesitas más de uno, aísla puertos + config/estado/espacio de trabajo. Consulta [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host).

Relacionado:

- [/gateway/background-process](/gateway/background-process)
- [/gateway/configuration](/gateway/configuration)
- [/gateway/doctor](/gateway/doctor)

## Advertencias del sondeo del Gateway

Usa esto cuando `openclaw gateway probe` alcanza algo, pero aun así muestra un bloque de advertencia.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Busca lo siguiente:

- `warnings[].code` y `primaryTargetId` en la salida JSON.
- Si la advertencia trata sobre respaldo SSH, varios gateways, ámbitos faltantes o referencias de autenticación no resueltas.

Firmas comunes:

- `SSH tunnel failed to start; falling back to direct probes.` → la configuración SSH falló, pero el comando aun así probó objetivos directos configurados/loopback.
- `multiple reachable gateways detected` → respondió más de un objetivo. Normalmente esto significa una configuración intencional con varios gateway o listeners obsoletos/duplicados.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → la conexión funcionó, pero el RPC de detalle está limitado por ámbito; empareja identidad del dispositivo o usa credenciales con `operator.read`.
- texto de advertencia por SecretRef no resuelto de `gateway.auth.*` / `gateway.remote.*` → el material de autenticación no estaba disponible en esta ruta de comando para el objetivo fallido.

Relacionado:

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host)
- [/gateway/remote](/gateway/remote)

## Canal conectado pero los mensajes no fluyen

Si el estado del canal es conectado pero el flujo de mensajes está muerto, céntrate en la política, los permisos y las reglas de entrega específicas del canal.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Busca lo siguiente:

- Política de mensajes directos (`pairing`, `allowlist`, `open`, `disabled`).
- Lista de permitidos de grupo y requisitos de mención.
- Permisos/ámbitos API del canal faltantes.

Firmas comunes:

- `mention required` → mensaje ignorado por la política de mención en grupo.
- trazas `pairing` / aprobación pendiente → el remitente no está aprobado.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → problema de autenticación/permisos del canal.

Relacionado:

- [/channels/troubleshooting](/channels/troubleshooting)
- [/channels/whatsapp](/channels/whatsapp)
- [/channels/telegram](/channels/telegram)
- [/channels/discord](/channels/discord)

## Entrega de cron y heartbeat

Si cron o heartbeat no se ejecutó o no se entregó, verifica primero el estado del programador y luego el destino de entrega.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Busca lo siguiente:

- Cron habilitado y siguiente activación presente.
- Estado del historial de ejecución del trabajo (`ok`, `skipped`, `error`).
- Motivos de omisión de heartbeat (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

Firmas comunes:

- `cron: scheduler disabled; jobs will not run automatically` → cron deshabilitado.
- `cron: timer tick failed` → falló el tick del programador; comprueba errores de archivo/registro/runtime.
- `heartbeat skipped` con `reason=quiet-hours` → fuera de la ventana de horas activas.
- `heartbeat skipped` con `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe pero solo contiene líneas en blanco / encabezados Markdown, así que OpenClaw omite la llamada al modelo.
- `heartbeat skipped` con `reason=no-tasks-due` → `HEARTBEAT.md` contiene un bloque `tasks:`, pero ninguna de las tareas vence en este tick.
- `heartbeat: unknown accountId` → id de cuenta no válido para el destino de entrega de heartbeat.
- `heartbeat skipped` con `reason=dm-blocked` → el destino de heartbeat se resolvió a un destino tipo DM mientras `agents.defaults.heartbeat.directPolicy` (o la anulación por agente) está establecido en `block`.

Relacionado:

- [/automation/cron-jobs#troubleshooting](/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/automation/cron-jobs)
- [/gateway/heartbeat](/gateway/heartbeat)

## Falla la herramienta de nodo emparejado

Si un nodo está emparejado pero las herramientas fallan, aísla el estado de primer plano, permisos y aprobaciones.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Busca lo siguiente:

- Nodo en línea con las capacidades esperadas.
- Permisos del sistema operativo para cámara/micrófono/ubicación/pantalla.
- Estado de aprobaciones de exec y lista de permitidos.

Firmas comunes:

- `NODE_BACKGROUND_UNAVAILABLE` → la app del nodo debe estar en primer plano.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → falta permiso del sistema operativo.
- `SYSTEM_RUN_DENIED: approval required` → aprobación de exec pendiente.
- `SYSTEM_RUN_DENIED: allowlist miss` → comando bloqueado por la lista de permitidos.

Relacionado:

- [/nodes/troubleshooting](/nodes/troubleshooting)
- [/nodes/index](/nodes/index)
- [/tools/exec-approvals](/tools/exec-approvals)

## Falla la herramienta de navegador

Usa esto cuando las acciones de la herramienta de navegador fallan aunque el propio gateway esté saludable.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Busca lo siguiente:

- Si `plugins.allow` está establecido e incluye `browser`.
- Ruta válida al ejecutable del navegador.
- Alcanzabilidad del perfil CDP.
- Disponibilidad local de Chrome para perfiles `existing-session` / `user`.

Firmas comunes:

- `unknown command "browser"` o `unknown command 'browser'` → el plugin incluido del navegador está excluido por `plugins.allow`.
- herramienta de navegador faltante / no disponible mientras `browser.enabled=true` → `plugins.allow` excluye `browser`, así que el plugin nunca se cargó.
- `Failed to start Chrome CDP on port` → el proceso del navegador no pudo iniciarse.
- `browser.executablePath not found` → la ruta configurada no es válida.
- `browser.cdpUrl must be http(s) or ws(s)` → la URL CDP configurada usa un esquema no compatible como `file:` o `ftp:`.
- `browser.cdpUrl has invalid port` → la URL CDP configurada tiene un puerto incorrecto o fuera de rango.
- `No Chrome tabs found for profile="user"` → el perfil de adjuntar Chrome MCP no tiene pestañas locales abiertas de Chrome.
- `Remote CDP for profile "<name>" is not reachable` → el endpoint CDP remoto configurado no es accesible desde el host del gateway.
- `Browser attachOnly is enabled ... not reachable` o `Browser attachOnly is enabled and CDP websocket ... is not reachable` → el perfil attach-only no tiene un objetivo accesible, o el endpoint HTTP respondió pero aun así no pudo abrirse el WebSocket CDP.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → la instalación actual del gateway no incluye el paquete completo de Playwright; las instantáneas ARIA y capturas básicas de página aún pueden funcionar, pero la navegación, instantáneas de IA, capturas de elementos por selector CSS y exportación PDF seguirán sin estar disponibles.
- `fullPage is not supported for element screenshots` → la solicitud de captura mezcló `--full-page` con `--ref` o `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → las llamadas de captura en Chrome MCP / `existing-session` deben usar captura de página o un `--ref` de instantánea, no `--element` CSS.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → los hooks de subida de archivos de Chrome MCP requieren referencias de instantánea, no selectores CSS.
- `existing-session file uploads currently support one file at a time.` → envía una sola subida por llamada en perfiles Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` → los hooks de diálogo en perfiles Chrome MCP no admiten anulaciones de tiempo de espera.
- `response body is not supported for existing-session profiles yet.` → `responsebody` todavía requiere un navegador gestionado o un perfil CDP sin procesar.
- anulaciones obsoletas de viewport / modo oscuro / locale / offline en perfiles attach-only o CDP remoto → ejecuta `openclaw browser stop --browser-profile <name>` para cerrar la sesión de control activa y liberar el estado de emulación Playwright/CDP sin reiniciar todo el gateway.

Relacionado:

- [/tools/browser-linux-troubleshooting](/tools/browser-linux-troubleshooting)
- [/tools/browser](/tools/browser)

## Si actualizaste y algo se rompió de repente

La mayoría de problemas posteriores a una actualización son deriva de configuración o valores predeterminados más estrictos que ahora se están aplicando.

### 1) Cambió el comportamiento de autenticación y anulación de URL

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

Qué comprobar:

- Si `gateway.mode=remote`, las llamadas de la CLI pueden estar apuntando a remoto mientras tu servicio local está bien.
- Las llamadas explícitas con `--url` no recurren a credenciales almacenadas.

Firmas comunes:

- `gateway connect failed:` → objetivo URL incorrecto.
- `unauthorized` → endpoint accesible pero autenticación incorrecta.

### 2) Los guardarraíles de bind y autenticación son más estrictos

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

Qué comprobar:

- Los binds no loopback (`lan`, `tailnet`, `custom`) necesitan una ruta válida de autenticación del gateway: autenticación compartida por token/contraseña, o una implementación `trusted-proxy` no loopback correctamente configurada.
- Las claves antiguas como `gateway.token` no sustituyen `gateway.auth.token`.

Firmas comunes:

- `refusing to bind gateway ... without auth` → bind no loopback sin una ruta válida de autenticación del gateway.
- `RPC probe: failed` mientras el runtime se está ejecutando → el gateway está activo pero es inaccesible con la autenticación/url actual.

### 3) Cambió el estado de emparejamiento e identidad del dispositivo

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

Qué comprobar:

- Aprobaciones pendientes de dispositivos para panel/nodos.
- Aprobaciones pendientes de emparejamiento de mensajes directos tras cambios de política o identidad.

Firmas comunes:

- `device identity required` → no se cumple la autenticación del dispositivo.
- `pairing required` → debe aprobarse el remitente/dispositivo.

Si la configuración del servicio y el runtime siguen sin coincidir después de las comprobaciones, reinstala los metadatos del servicio desde el mismo directorio de perfil/estado:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Relacionado:

- [/gateway/pairing](/gateway/pairing)
- [/gateway/authentication](/gateway/authentication)
- [/gateway/background-process](/gateway/background-process)
