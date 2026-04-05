---
read_when:
    - Ejecución del Gateway desde la CLI (desarrollo o servidores)
    - Depuración de autenticación del Gateway, modos de enlace y conectividad
    - Detección de gateways mediante Bonjour (DNS-SD local y de área amplia)
summary: 'CLI del Gateway de OpenClaw (`openclaw gateway`): ejecutar, consultar y detectar gateways'
title: gateway
x-i18n:
    generated_at: "2026-04-05T12:38:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: e311ded0dbad84b8212f0968f3563998d49c5e0eb292a0dc4b3bd3c22d4fa7f2
    source_path: cli/gateway.md
    workflow: 15
---

# CLI del Gateway

El Gateway es el servidor WebSocket de OpenClaw (canales, nodos, sesiones, hooks).

Los subcomandos de esta página viven bajo `openclaw gateway …`.

Documentación relacionada:

- [/gateway/bonjour](/gateway/bonjour)
- [/gateway/discovery](/gateway/discovery)
- [/gateway/configuration](/gateway/configuration)

## Ejecutar el Gateway

Ejecuta un proceso local del Gateway:

```bash
openclaw gateway
```

Alias en primer plano:

```bash
openclaw gateway run
```

Notas:

- De forma predeterminada, el Gateway se niega a iniciarse a menos que `gateway.mode=local` esté configurado en `~/.openclaw/openclaw.json`. Usa `--allow-unconfigured` para ejecuciones ad hoc/de desarrollo.
- Se espera que `openclaw onboard --mode local` y `openclaw setup` escriban `gateway.mode=local`. Si el archivo existe pero falta `gateway.mode`, trátalo como una configuración dañada o sobrescrita y repárala en lugar de asumir implícitamente el modo local.
- Si el archivo existe y falta `gateway.mode`, el Gateway lo trata como un daño sospechoso en la configuración y se niega a “asumir local” por ti.
- Se bloquea enlazar más allá del loopback sin autenticación (medida de seguridad).
- `SIGUSR1` activa un reinicio en proceso cuando está autorizado (`commands.restart` está habilitado de forma predeterminada; establece `commands.restart: false` para bloquear el reinicio manual, mientras que las operaciones de herramienta/configuración de gateway apply/update siguen permitidas).
- Los controladores `SIGINT`/`SIGTERM` detienen el proceso del gateway, pero no restauran ningún estado personalizado del terminal. Si envuelves la CLI con una TUI o entrada en modo raw, restaura el terminal antes de salir.

### Opciones

- `--port <port>`: puerto WebSocket (el valor predeterminado viene de config/env; normalmente `18789`).
- `--bind <loopback|lan|tailnet|auto|custom>`: modo de enlace del listener.
- `--auth <token|password>`: sobrescritura del modo de autenticación.
- `--token <token>`: sobrescritura del token (también establece `OPENCLAW_GATEWAY_TOKEN` para el proceso).
- `--password <password>`: sobrescritura de contraseña. Advertencia: las contraseñas en línea pueden quedar expuestas en listados locales de procesos.
- `--password-file <path>`: lee la contraseña del gateway desde un archivo.
- `--tailscale <off|serve|funnel>`: expone el Gateway mediante Tailscale.
- `--tailscale-reset-on-exit`: restablece la configuración serve/funnel de Tailscale al apagarse.
- `--allow-unconfigured`: permite iniciar el gateway sin `gateway.mode=local` en la configuración. Esto omite la protección de inicio solo para arranque ad hoc/de desarrollo; no escribe ni repara el archivo de configuración.
- `--dev`: crea una configuración + espacio de trabajo de desarrollo si faltan (omite `BOOTSTRAP.md`).
- `--reset`: restablece la configuración de desarrollo + credenciales + sesiones + espacio de trabajo (requiere `--dev`).
- `--force`: mata cualquier listener existente en el puerto seleccionado antes de iniciar.
- `--verbose`: registros detallados.
- `--cli-backend-logs`: muestra solo registros del backend de la CLI en la consola (y habilita stdout/stderr).
- `--claude-cli-logs`: alias en desuso de `--cli-backend-logs`.
- `--ws-log <auto|full|compact>`: estilo de registro de websocket (predeterminado `auto`).
- `--compact`: alias de `--ws-log compact`.
- `--raw-stream`: registra eventos sin procesar del flujo del modelo en jsonl.
- `--raw-stream-path <path>`: ruta del jsonl del flujo sin procesar.

## Consultar un Gateway en ejecución

Todos los comandos de consulta usan RPC por WebSocket.

Modos de salida:

- Predeterminado: legible para humanos (con color en TTY).
- `--json`: JSON legible por máquina (sin estilo/spinner).
- `--no-color` (o `NO_COLOR=1`): desactiva ANSI manteniendo el diseño legible para humanos.

Opciones compartidas (cuando son compatibles):

- `--url <url>`: URL WebSocket del Gateway.
- `--token <token>`: token del Gateway.
- `--password <password>`: contraseña del Gateway.
- `--timeout <ms>`: tiempo de espera/presupuesto (varía según el comando).
- `--expect-final`: espera una respuesta “final” (llamadas de agente).

Nota: cuando estableces `--url`, la CLI no recurre a credenciales de configuración ni de variables de entorno.
Pasa `--token` o `--password` explícitamente. La ausencia de credenciales explícitas es un error.

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

### `gateway usage-cost`

Obtiene resúmenes de costo de uso a partir de los registros de sesión.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --json
```

Opciones:

- `--days <days>`: número de días a incluir (predeterminado `30`).

### `gateway status`

`gateway status` muestra el servicio del Gateway (launchd/systemd/schtasks) junto con un sondeo RPC opcional.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

Opciones:

- `--url <url>`: añade un destino de sondeo explícito. Se siguen sondeando el remoto configurado + localhost.
- `--token <token>`: autenticación por token para el sondeo.
- `--password <password>`: autenticación por contraseña para el sondeo.
- `--timeout <ms>`: tiempo de espera del sondeo (predeterminado `10000`).
- `--no-probe`: omite el sondeo RPC (vista solo del servicio).
- `--deep`: analiza también servicios a nivel del sistema.
- `--require-rpc`: sale con código distinto de cero cuando falla el sondeo RPC. No puede combinarse con `--no-probe`.

Notas:

- `gateway status` sigue disponible para diagnóstico incluso cuando falta o es inválida la configuración local de la CLI.
- `gateway status` resuelve SecretRefs de autenticación configurados para la autenticación del sondeo cuando es posible.
- Si un SecretRef de autenticación requerido no se resuelve en esta ruta del comando, `gateway status --json` informa `rpc.authWarning` cuando falla la conectividad/autenticación del sondeo; pasa `--token`/`--password` explícitamente o resuelve primero el origen del secreto.
- Si el sondeo se realiza correctamente, se suprimen las advertencias de referencias de autenticación no resueltas para evitar falsos positivos.
- Usa `--require-rpc` en scripts y automatización cuando no basta con un servicio en escucha y necesitas que el propio RPC del Gateway esté en buen estado.
- `--deep` añade un análisis de mejor esfuerzo para instalaciones adicionales de launchd/systemd/schtasks. Cuando se detectan varios servicios tipo gateway, la salida legible para humanos imprime sugerencias de limpieza y advierte que la mayoría de instalaciones deberían ejecutar un solo gateway por máquina.
- La salida legible para humanos incluye la ruta resuelta del archivo de registro más una instantánea de las rutas/validez de la configuración de CLI frente al servicio para ayudar a diagnosticar desajustes de perfil o de directorio de estado.
- En instalaciones Linux con systemd, las comprobaciones de desajuste de autenticación del servicio leen los valores `Environment=` y `EnvironmentFile=` de la unidad (incluidos `%h`, rutas entre comillas, varios archivos y archivos opcionales con `-`).
- Las comprobaciones de desajuste resuelven SecretRefs de `gateway.auth.token` usando el entorno de ejecución combinado (primero el entorno del comando del servicio, luego el entorno del proceso como respaldo).
- Si la autenticación por token no está efectivamente activa (modo `gateway.auth.mode` explícito de `password`/`none`/`trusted-proxy`, o modo no definido donde puede ganar password y ningún candidato de token puede ganar), las comprobaciones de desajuste de token omiten la resolución del token configurado.

### `gateway probe`

`gateway probe` es el comando de “depurar todo”. Siempre sondea:

- tu gateway remoto configurado (si existe), y
- localhost (loopback) **aunque haya un remoto configurado**.

Si pasas `--url`, ese destino explícito se añade por delante de ambos. La salida legible para humanos etiqueta los
destinos como:

- `URL (explícita)`
- `Remoto (configurado)` o `Remoto (configurado, inactivo)`
- `Loopback local`

Si se puede alcanzar más de un gateway, los imprime todos. Se admiten varios gateways cuando usas perfiles/puertos aislados (por ejemplo, un bot de rescate), pero la mayoría de instalaciones siguen ejecutando un solo gateway.

```bash
openclaw gateway probe
openclaw gateway probe --json
```

Interpretación:

- `Reachable: yes` significa que al menos un destino aceptó una conexión WebSocket.
- `RPC: ok` significa que las llamadas RPC de detalle (`health`/`status`/`system-presence`/`config.get`) también se realizaron correctamente.
- `RPC: limited - missing scope: operator.read` significa que la conexión se realizó correctamente, pero el RPC de detalle está limitado por alcance. Esto se informa como accesibilidad **degradada**, no como error total.
- El código de salida es distinto de cero solo cuando no se puede alcanzar ningún destino sondeado.

Notas de JSON (`--json`):

- Nivel superior:
  - `ok`: se puede alcanzar al menos un destino.
  - `degraded`: al menos un destino tenía RPC de detalle limitado por alcance.
  - `primaryTargetId`: mejor destino para tratarlo como el ganador activo en este orden: URL explícita, túnel SSH, remoto configurado y luego loopback local.
  - `warnings[]`: registros de advertencia de mejor esfuerzo con `code`, `message` y `targetIds` opcionales.
  - `network`: sugerencias de URL de loopback local/tailnet derivadas de la configuración actual y de la red del host.
  - `discovery.timeoutMs` y `discovery.count`: el presupuesto/cantidad real de detección usado para esta pasada del sondeo.
- Por destino (`targets[].connect`):
  - `ok`: accesibilidad tras conectar + clasificación degradada.
  - `rpcOk`: éxito completo del RPC de detalle.
  - `scopeLimited`: el RPC de detalle falló por falta del alcance de operador.

Códigos de advertencia comunes:

- `ssh_tunnel_failed`: falló la configuración del túnel SSH; el comando volvió a sondeos directos.
- `multiple_gateways`: se podía alcanzar más de un destino; esto es poco habitual salvo que ejecutes intencionalmente perfiles aislados, como un bot de rescate.
- `auth_secretref_unresolved`: no se pudo resolver un SecretRef de autenticación configurado para un destino con error.
- `probe_scope_limited`: la conexión WebSocket se realizó correctamente, pero el RPC de detalle quedó limitado por falta de `operator.read`.

#### Remoto por SSH (paridad con la app de Mac)

El modo “Remote over SSH” de la app de macOS usa un reenvío de puerto local para que el gateway remoto (que puede estar enlazado solo a loopback) sea accesible en `ws://127.0.0.1:<port>`.

Equivalente en la CLI:

```bash
openclaw gateway probe --ssh user@gateway-host
```

Opciones:

- `--ssh <target>`: `user@host` o `user@host:port` (el puerto predeterminado es `22`).
- `--ssh-identity <path>`: archivo de identidad.
- `--ssh-auto`: elige el primer host de gateway detectado como destino SSH desde el
  endpoint de detección resuelto (`local.` más el dominio de área amplia configurado, si existe). Se ignoran las
  sugerencias solo-TXT.

Configuración (opcional, usada como valor predeterminado):

- `gateway.remote.sshTarget`
- `gateway.remote.sshIdentity`

### `gateway call <method>`

Asistente RPC de bajo nivel.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"sinceMs": 60000}'
```

Opciones:

- `--params <json>`: cadena de objeto JSON para parámetros (predeterminado `{}`)
- `--url <url>`
- `--token <token>`
- `--password <password>`
- `--timeout <ms>`
- `--expect-final`
- `--json`

Notas:

- `--params` debe ser JSON válido.
- `--expect-final` es principalmente para RPC de tipo agente que transmiten eventos intermedios antes de una carga útil final.

## Gestionar el servicio del Gateway

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

Opciones del comando:

- `gateway status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
- `gateway install`: `--port`, `--runtime <node|bun>`, `--token`, `--force`, `--json`
- `gateway uninstall|start|stop|restart`: `--json`

Notas:

- `gateway install` admite `--port`, `--runtime`, `--token`, `--force`, `--json`.
- Cuando la autenticación por token requiere un token y `gateway.auth.token` está gestionado por SecretRef, `gateway install` valida que el SecretRef pueda resolverse, pero no persiste el token resuelto en los metadatos del entorno del servicio.
- Si la autenticación por token requiere un token y el SecretRef del token configurado no se resuelve, la instalación falla en modo cerrado en lugar de persistir un respaldo en texto plano.
- Para autenticación por contraseña en `gateway run`, prefiere `OPENCLAW_GATEWAY_PASSWORD`, `--password-file` o un `gateway.auth.password` respaldado por SecretRef en lugar de `--password` en línea.
- En modo de autenticación inferida, `OPENCLAW_GATEWAY_PASSWORD` solo en shell no relaja los requisitos de token para la instalación; usa configuración duradera (`gateway.auth.password` o `env` de configuración) al instalar un servicio gestionado.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está definido, se bloquea la instalación hasta que el modo se establezca explícitamente.
- Los comandos de ciclo de vida aceptan `--json` para scripts.

## Detectar gateways (Bonjour)

`gateway discover` busca beacons del Gateway (`_openclaw-gw._tcp`).

- Multicast DNS-SD: `local.`
- Unicast DNS-SD (Bonjour de área amplia): elige un dominio (ejemplo: `openclaw.internal.`) y configura split DNS + un servidor DNS; consulta [/gateway/bonjour](/gateway/bonjour)

Solo los gateways con la detección Bonjour habilitada (predeterminada) anuncian el beacon.

Los registros de detección de área amplia incluyen (TXT):

- `role` (sugerencia de rol del gateway)
- `transport` (sugerencia de transporte, por ejemplo `gateway`)
- `gatewayPort` (puerto WebSocket, normalmente `18789`)
- `sshPort` (opcional; los clientes usan por defecto `22` como destino SSH cuando falta)
- `tailnetDns` (hostname de MagicDNS, cuando está disponible)
- `gatewayTls` / `gatewayTlsSha256` (TLS habilitado + huella digital del certificado)
- `cliPath` (sugerencia de instalación remota escrita en la zona de área amplia)

### `gateway discover`

```bash
openclaw gateway discover
```

Opciones:

- `--timeout <ms>`: tiempo de espera por comando (browse/resolve); predeterminado `2000`.
- `--json`: salida legible por máquina (también desactiva estilo/spinner).

Ejemplos:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

Notas:

- La CLI busca en `local.` más el dominio de área amplia configurado cuando uno está habilitado.
- `wsUrl` en la salida JSON se deriva del endpoint resuelto del servicio, no de
  sugerencias solo-TXT como `lanHost` o `tailnetDns`.
- En mDNS `local.`, `sshPort` y `cliPath` solo se difunden cuando
  `discovery.mdns.mode` es `full`. DNS-SD de área amplia sigue escribiendo `cliPath`; `sshPort`
  sigue siendo opcional allí también.
