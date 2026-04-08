---
read_when:
    - El centro de solución de problemas te envió aquí para un diagnóstico más profundo
    - Necesitas secciones estables de runbook basadas en síntomas con comandos exactos
summary: Guía detallada de solución de problemas para gateway, canales, automatización, nodos y navegador
title: Solución de problemas
x-i18n:
    generated_at: "2026-04-08T02:15:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 02c9537845248db0c9d315bf581338a93215fe6fe3688ed96c7105cbb19fe6ba
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# Solución de problemas del Gateway

Esta página es el runbook detallado.
Empieza en [/help/troubleshooting](/es/help/troubleshooting) si primero quieres el flujo rápido de triaje.

## Escalera de comandos

Ejecuta estos primero, en este orden:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Señales esperadas de buen estado:

- `openclaw gateway status` muestra `Runtime: running` y `RPC probe: ok`.
- `openclaw doctor` no informa problemas bloqueantes de configuración/servicio.
- `openclaw channels status --probe` muestra el estado de transporte en vivo por cuenta y,
  donde se admite, resultados de sondeo/auditoría como `works` o `audit ok`.

## Anthropic 429: se requiere uso adicional para contexto largo

Usa esto cuando los registros/errores incluyan:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Busca lo siguiente:

- El modelo Anthropic Opus/Sonnet seleccionado tiene `params.context1m: true`.
- La credencial actual de Anthropic no es apta para el uso de contexto largo.
- Las solicitudes fallan solo en sesiones largas/ejecuciones de modelo que necesitan la ruta beta de 1M.

Opciones de corrección:

1. Desactiva `context1m` para ese modelo y volver a la ventana de contexto normal.
2. Usa una credencial de Anthropic apta para solicitudes de contexto largo, o cambia a una clave de API de Anthropic.
3. Configura modelos alternativos para que las ejecuciones continúen cuando las solicitudes de contexto largo de Anthropic sean rechazadas.

Relacionado:

- [/providers/anthropic](/es/providers/anthropic)
- [/reference/token-use](/es/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/es/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Un backend local compatible con OpenAI pasa sondeos directos, pero fallan las ejecuciones del agente

Usa esto cuando:

- `curl ... /v1/models` funciona
- las llamadas directas pequeñas a `/v1/chat/completions` funcionan
- las ejecuciones de modelos de OpenClaw fallan solo en turnos normales del agente

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Busca lo siguiente:

- las llamadas directas pequeñas tienen éxito, pero las ejecuciones de OpenClaw fallan solo con prompts más grandes
- errores del backend sobre `messages[].content` que espera una cadena
- fallos del backend que aparecen solo con recuentos mayores de tokens de prompt o con prompts completos del runtime del agente

Firmas comunes:

- `messages[...].content: invalid type: sequence, expected a string` → el backend
  rechaza partes de contenido estructurado de Chat Completions. Corrección: configura
  `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- las solicitudes directas pequeñas funcionan, pero las ejecuciones del agente de OpenClaw fallan con
  fallos del backend/modelo (por ejemplo Gemma en algunas compilaciones de `inferrs`) → el transporte de OpenClaw
  probablemente ya sea correcto; el backend está fallando con la forma de prompt más grande del runtime del agente.
- los fallos disminuyen tras desactivar las herramientas, pero no desaparecen → los esquemas de herramientas
  eran parte de la presión, pero el problema restante sigue estando en la capacidad del modelo/servidor aguas arriba
  o en un error del backend.

Opciones de corrección:

1. Configura `compat.requiresStringContent: true` para backends de Chat Completions que solo admiten cadenas.
2. Configura `compat.supportsTools: false` para modelos/backends que no pueden manejar
   de forma fiable la superficie de esquemas de herramientas de OpenClaw.
3. Reduce la presión del prompt cuando sea posible: bootstrap del espacio de trabajo más pequeño, historial
   de sesión más corto, modelo local más ligero o un backend con mejor compatibilidad con contexto largo.
4. Si las solicitudes directas pequeñas siguen funcionando mientras los turnos del agente OpenClaw todavía fallan
   dentro del backend, trátalo como una limitación del servidor/modelo aguas arriba y presenta
   allí una reproducción con la forma de carga aceptada.

Relacionado:

- [/gateway/local-models](/es/gateway/local-models)
- [/gateway/configuration#models](/es/gateway/configuration#models)
- [/gateway/configuration-reference#openai-compatible-endpoints](/es/gateway/configuration-reference#openai-compatible-endpoints)

## Sin respuestas

Si los canales están activos pero nada responde, comprueba el enrutamiento y la política antes de reconectar nada.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Busca lo siguiente:

- Emparejamiento pendiente para remitentes de MD.
- Restricción de menciones en grupos (`requireMention`, `mentionPatterns`).
- Desajustes en listas permitidas de canal/grupo.

Firmas comunes:

- `drop guild message (mention required` → el mensaje de grupo se ignora hasta que haya mención.
- `pairing request` → el remitente necesita aprobación.
- `blocked` / `allowlist` → el remitente/canal fue filtrado por política.

Relacionado:

- [/channels/troubleshooting](/es/channels/troubleshooting)
- [/channels/pairing](/es/channels/pairing)
- [/channels/groups](/es/channels/groups)

## Conectividad del Dashboard de Control UI

Cuando el dashboard/Control UI no se conecta, valida la URL, el modo de autenticación y las suposiciones de contexto seguro.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Busca lo siguiente:

- URL correcta del sondeo y del dashboard.
- Desajuste de modo/token de autenticación entre cliente y gateway.
- Uso de HTTP donde se requiere identidad del dispositivo.

Firmas comunes:

- `device identity required` → contexto no seguro o falta autenticación de dispositivo.
- `origin not allowed` → el `Origin` del navegador no está en `gateway.controlUi.allowedOrigins`
  (o te estás conectando desde un origen de navegador no loopback sin una
  lista permitida explícita).
- `device nonce required` / `device nonce mismatch` → el cliente no está completando el
  flujo de autenticación de dispositivo basado en desafío (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → el cliente firmó la carga útil incorrecta
  (o una marca de tiempo obsoleta) para el handshake actual.
- `AUTH_TOKEN_MISMATCH` con `canRetryWithDeviceToken=true` → el cliente puede hacer un reintento confiable con token de dispositivo en caché.
- Ese reintento con token en caché reutiliza el conjunto de scopes en caché almacenado con el token
  de dispositivo emparejado. Las llamadas con `deviceToken` explícito / `scopes` explícitos mantienen su
  conjunto de scopes solicitado en su lugar.
- Fuera de esa ruta de reintento, la precedencia de autenticación de conexión es primero
  token/contraseña compartidos explícitos, luego `deviceToken` explícito, luego token de dispositivo almacenado
  y luego token bootstrap.
- En la ruta asíncrona de Control UI con Tailscale Serve, los intentos fallidos para el mismo
  `{scope, ip}` se serializan antes de que el limitador registre el fallo. Por lo tanto, dos reintentos concurrentes incorrectos del mismo cliente pueden mostrar `retry later`
  en el segundo intento en lugar de dos desajustes simples.
- `too many failed authentication attempts (retry later)` desde un cliente loopback de origen de navegador
  → los fallos repetidos desde ese mismo `Origin` normalizado se bloquean temporalmente; otro origen localhost usa un bucket separado.
- `unauthorized` repetido después de ese reintento → desfase de token compartido/token de dispositivo; actualiza la configuración del token y vuelve a aprobar/rotar el token de dispositivo si es necesario.
- `gateway connect failed:` → host/puerto/URL de destino incorrectos.

### Mapa rápido de códigos de detalle de autenticación

Usa `error.details.code` de la respuesta fallida de `connect` para elegir la siguiente acción:

| Código de detalle             | Significado                                              | Acción recomendada                                                                                                                                                                                                                                                                          |
| ----------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | El cliente no envió un token compartido requerido.       | Pega/configura el token en el cliente y vuelve a intentarlo. Para rutas del dashboard: `openclaw config get gateway.auth.token` y luego pégalo en la configuración de Control UI.                                                                                                         |
| `AUTH_TOKEN_MISMATCH`        | El token compartido no coincidía con el token auth del gateway. | Si `canRetryWithDeviceToken=true`, permite un reintento confiable. Los reintentos con token en caché reutilizan los scopes aprobados almacenados; las llamadas con `deviceToken` / `scopes` explícitos mantienen los scopes solicitados. Si sigue fallando, ejecuta la [lista de recuperación por desfase de token](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | El token por dispositivo en caché está obsoleto o revocado. | Rota/vuelve a aprobar el token de dispositivo usando [devices CLI](/cli/devices), luego vuelve a conectar.                                                                                                                                                                                 |
| `PAIRING_REQUIRED`           | La identidad del dispositivo es conocida pero no está aprobada para este rol. | Aprueba la solicitud pendiente: `openclaw devices list` y luego `openclaw devices approve <requestId>`.                                                                                                                                                                                     |

Comprobación de migración a autenticación de dispositivo v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Si los registros muestran errores de nonce/firma, actualiza el cliente que se conecta y verifica que:

1. espere a `connect.challenge`
2. firme la carga útil vinculada al desafío
3. envíe `connect.params.device.nonce` con el mismo nonce del desafío

Si `openclaw devices rotate` / `revoke` / `remove` es denegado de forma inesperada:

- las sesiones con token de dispositivo emparejado solo pueden administrar **su propio** dispositivo, a menos que la
  llamada también tenga `operator.admin`
- `openclaw devices rotate --scope ...` solo puede solicitar scopes de operador que
  la sesión llamante ya tenga

Relacionado:

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/es/gateway/configuration) (modos de autenticación del gateway)
- [/gateway/trusted-proxy-auth](/es/gateway/trusted-proxy-auth)
- [/gateway/remote](/es/gateway/remote)
- [/cli/devices](/cli/devices)

## El servicio Gateway no se está ejecutando

Usa esto cuando el servicio está instalado pero el proceso no se mantiene activo.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # también escanea servicios a nivel del sistema
```

Busca lo siguiente:

- `Runtime: stopped` con pistas de salida.
- Desajuste de configuración del servicio (`Config (cli)` frente a `Config (service)`).
- Conflictos de puerto/listener.
- Instalaciones adicionales de launchd/systemd/schtasks cuando se usa `--deep`.
- Pistas de limpieza de `Other gateway-like services detected (best effort)`.

Firmas comunes:

- `Gateway start blocked: set gateway.mode=local` o `existing config is missing gateway.mode` → el modo de gateway local no está habilitado, o el archivo de configuración fue sobrescrito y perdió `gateway.mode`. Corrección: configura `gateway.mode="local"` en tu configuración, o vuelve a ejecutar `openclaw onboard --mode local` / `openclaw setup` para volver a sellar la configuración esperada de modo local. Si estás ejecutando OpenClaw mediante Podman, la ruta de configuración predeterminada es `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → bind no loopback sin una ruta válida de autenticación del gateway (token/contraseña, o trusted-proxy donde esté configurado).
- `another gateway instance is already listening` / `EADDRINUSE` → conflicto de puertos.
- `Other gateway-like services detected (best effort)` → existen unidades launchd/systemd/schtasks obsoletas o paralelas. La mayoría de las configuraciones deberían mantener un gateway por máquina; si realmente necesitas más de uno, aísla puertos + config/estado/espacio de trabajo. Consulta [/gateway#multiple-gateways-same-host](/es/gateway#multiple-gateways-same-host).

Relacionado:

- [/gateway/background-process](/es/gateway/background-process)
- [/gateway/configuration](/es/gateway/configuration)
- [/gateway/doctor](/es/gateway/doctor)

## Advertencias del sondeo del Gateway

Usa esto cuando `openclaw gateway probe` llega a algo, pero aun así imprime un bloque de advertencia.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Busca lo siguiente:

- `warnings[].code` y `primaryTargetId` en la salida JSON.
- Si la advertencia trata sobre fallback SSH, múltiples gateways, scopes faltantes o refs de autenticación no resueltos.

Firmas comunes:

- `SSH tunnel failed to start; falling back to direct probes.` → la configuración SSH falló, pero el comando aun así probó los destinos configurados/loopback directos.
- `multiple reachable gateways detected` → respondió más de un destino. Normalmente esto significa una configuración intencional con varios gateways o listeners obsoletos/duplicados.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → la conexión funcionó, pero el RPC de detalle está limitado por scopes; empareja la identidad del dispositivo o usa credenciales con `operator.read`.
- texto de advertencia de SecretRef no resuelto `gateway.auth.*` / `gateway.remote.*` → el material de autenticación no estaba disponible en esta ruta de comando para el destino fallido.

Relacionado:

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/es/gateway#multiple-gateways-same-host)
- [/gateway/remote](/es/gateway/remote)

## El canal está conectado, pero los mensajes no fluyen

Si el estado del canal es conectado pero el flujo de mensajes está muerto, céntrate en la política, los permisos y las reglas de entrega específicas del canal.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Busca lo siguiente:

- Política de MD (`pairing`, `allowlist`, `open`, `disabled`).
- Lista permitida de grupos y requisitos de mención.
- Permisos/scopes faltantes de la API del canal.

Firmas comunes:

- `mention required` → el mensaje fue ignorado por la política de mención en grupos.
- rastros de `pairing` / aprobación pendiente → el remitente no está aprobado.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → problema de autenticación/permisos del canal.

Relacionado:

- [/channels/troubleshooting](/es/channels/troubleshooting)
- [/channels/whatsapp](/es/channels/whatsapp)
- [/channels/telegram](/es/channels/telegram)
- [/channels/discord](/es/channels/discord)

## Entrega de cron y heartbeat

Si cron o heartbeat no se ejecutaron o no entregaron, primero verifica el estado del planificador y luego el destino de entrega.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Busca lo siguiente:

- Cron habilitado y próxima activación presente.
- Estado del historial de ejecuciones del trabajo (`ok`, `skipped`, `error`).
- Motivos de omisión de heartbeat (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

Firmas comunes:

- `cron: scheduler disabled; jobs will not run automatically` → cron deshabilitado.
- `cron: timer tick failed` → falló el tick del planificador; revisa errores de archivo/registro/runtime.
- `heartbeat skipped` con `reason=quiet-hours` → fuera del intervalo de horas activas.
- `heartbeat skipped` con `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe pero solo contiene líneas en blanco / encabezados markdown, por lo que OpenClaw omite la llamada al modelo.
- `heartbeat skipped` con `reason=no-tasks-due` → `HEARTBEAT.md` contiene un bloque `tasks:`, pero ninguna de las tareas vence en este tick.
- `heartbeat: unknown accountId` → id de cuenta no válido para el destino de entrega de heartbeat.
- `heartbeat skipped` con `reason=dm-blocked` → el destino de heartbeat se resolvió como un destino de estilo MD mientras `agents.defaults.heartbeat.directPolicy` (o la sobrescritura por agente) está configurado en `block`.

Relacionado:

- [/automation/cron-jobs#troubleshooting](/es/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/es/automation/cron-jobs)
- [/gateway/heartbeat](/es/gateway/heartbeat)

## La herramienta de nodo emparejado falla

Si un nodo está emparejado pero fallan las herramientas, aísla el estado de primer plano, permisos y aprobaciones.

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
- Aprobaciones de ejecución y estado de lista permitida.

Firmas comunes:

- `NODE_BACKGROUND_UNAVAILABLE` → la app del nodo debe estar en primer plano.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → falta un permiso del sistema operativo.
- `SYSTEM_RUN_DENIED: approval required` → aprobación de ejecución pendiente.
- `SYSTEM_RUN_DENIED: allowlist miss` → el comando fue bloqueado por la lista permitida.

Relacionado:

- [/nodes/troubleshooting](/es/nodes/troubleshooting)
- [/nodes/index](/es/nodes/index)
- [/tools/exec-approvals](/es/tools/exec-approvals)

## La herramienta del navegador falla

Usa esto cuando las acciones de la herramienta del navegador fallan aunque el propio gateway esté en buen estado.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Busca lo siguiente:

- Si `plugins.allow` está configurado e incluye `browser`.
- Ruta válida al ejecutable del navegador.
- Alcance del perfil CDP.
- Disponibilidad de Chrome local para perfiles `existing-session` / `user`.

Firmas comunes:

- `unknown command "browser"` o `unknown command 'browser'` → el plugin de navegador incluido está excluido por `plugins.allow`.
- falta la herramienta del navegador / no disponible mientras `browser.enabled=true` → `plugins.allow` excluye `browser`, así que el plugin nunca se cargó.
- `Failed to start Chrome CDP on port` → el proceso del navegador no pudo iniciarse.
- `browser.executablePath not found` → la ruta configurada no es válida.
- `browser.cdpUrl must be http(s) or ws(s)` → la URL de CDP configurada usa un esquema no compatible como `file:` o `ftp:`.
- `browser.cdpUrl has invalid port` → la URL de CDP configurada tiene un puerto incorrecto o fuera de rango.
- `No Chrome tabs found for profile="user"` → el perfil de adjuntar Chrome MCP no tiene pestañas locales de Chrome abiertas.
- `Remote CDP for profile "<name>" is not reachable` → el endpoint remoto de CDP configurado no es accesible desde el host del gateway.
- `Browser attachOnly is enabled ... not reachable` o `Browser attachOnly is enabled and CDP websocket ... is not reachable` → el perfil solo de adjuntar no tiene un destino accesible, o el endpoint HTTP respondió pero aun así no se pudo abrir el WebSocket de CDP.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → la instalación actual del gateway no incluye el paquete completo de Playwright; las instantáneas ARIA y las capturas básicas de página pueden seguir funcionando, pero la navegación, las instantáneas con IA, las capturas de elementos por selector CSS y la exportación a PDF seguirán sin estar disponibles.
- `fullPage is not supported for element screenshots` → la solicitud de captura mezcló `--full-page` con `--ref` o `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → las llamadas de captura de Chrome MCP / `existing-session` deben usar captura de página o un `--ref` de instantánea, no `--element` CSS.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → los hooks de subida de archivos en Chrome MCP necesitan refs de instantánea, no selectores CSS.
- `existing-session file uploads currently support one file at a time.` → envía una subida por llamada en perfiles Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` → los hooks de diálogos en perfiles Chrome MCP no admiten sobrescrituras de tiempo de espera.
- `response body is not supported for existing-session profiles yet.` → `responsebody` todavía requiere un navegador administrado o un perfil CDP sin procesar.
- sobrescrituras obsoletas de viewport / dark-mode / locale / offline en perfiles solo de adjuntar o CDP remoto → ejecuta `openclaw browser stop --browser-profile <name>` para cerrar la sesión de control activa y liberar el estado de emulación de Playwright/CDP sin reiniciar todo el gateway.

Relacionado:

- [/tools/browser-linux-troubleshooting](/es/tools/browser-linux-troubleshooting)
- [/tools/browser](/es/tools/browser)

## Si actualizaste y de repente algo dejó de funcionar

La mayoría de las fallas posteriores a una actualización se deben a desfase de configuración o a valores predeterminados más estrictos que ahora se están aplicando.

### 1) Cambió el comportamiento de autenticación y sobrescritura de URL

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

Qué comprobar:

- Si `gateway.mode=remote`, las llamadas de CLI pueden estar apuntando a remoto aunque tu servicio local esté bien.
- Las llamadas explícitas con `--url` no recurren a credenciales almacenadas.

Firmas comunes:

- `gateway connect failed:` → URL de destino incorrecta.
- `unauthorized` → el endpoint es accesible pero la autenticación es incorrecta.

### 2) Las barreras de protección de bind y auth son más estrictas

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

Qué comprobar:

- Los bind no loopback (`lan`, `tailnet`, `custom`) necesitan una ruta válida de autenticación del gateway: autenticación con token/contraseña compartidos, o una implementación `trusted-proxy` no loopback configurada correctamente.
- Las claves antiguas como `gateway.token` no sustituyen a `gateway.auth.token`.

Firmas comunes:

- `refusing to bind gateway ... without auth` → bind no loopback sin una ruta válida de autenticación del gateway.
- `RPC probe: failed` mientras el runtime está en ejecución → el gateway está activo pero no es accesible con la autenticación/URL actuales.

### 3) Cambió el estado de emparejamiento e identidad de dispositivo

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

Qué comprobar:

- Aprobaciones de dispositivos pendientes para dashboard/nodos.
- Aprobaciones pendientes de emparejamiento de MD tras cambios de política o identidad.

Firmas comunes:

- `device identity required` → no se cumple la autenticación de dispositivo.
- `pairing required` → el remitente/dispositivo debe ser aprobado.

Si la configuración del servicio y el runtime siguen sin coincidir después de las comprobaciones, reinstala los metadatos del servicio desde el mismo directorio de perfil/estado:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Relacionado:

- [/gateway/pairing](/es/gateway/pairing)
- [/gateway/authentication](/es/gateway/authentication)
- [/gateway/background-process](/es/gateway/background-process)
