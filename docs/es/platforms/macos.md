---
read_when:
    - Implementando funciones de la app de macOS
    - Cambiando el ciclo de vida del gateway o el puente de nodos en macOS
summary: App complementaria de OpenClaw para macOS (barra de menús + broker del gateway)
title: App de macOS
x-i18n:
    generated_at: "2026-04-05T12:49:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: bfac937e352ede495f60af47edf3b8e5caa5b692ba0ea01d9fb0de9a44bbc135
    source_path: platforms/macos.md
    workflow: 15
---

# Complemento de OpenClaw para macOS (barra de menús + broker del gateway)

La app de macOS es el **complemento de barra de menús** de OpenClaw. Gestiona permisos,
administra/se conecta al Gateway localmente (launchd o manual) y expone capacidades
de macOS al agente como un nodo.

## Qué hace

- Muestra notificaciones nativas y estado en la barra de menús.
- Gestiona los prompts de TCC (Notificaciones, Accesibilidad, Grabación de Pantalla, Micrófono,
  Reconocimiento de Voz, Automatización/AppleScript).
- Ejecuta o se conecta al Gateway (local o remoto).
- Expone herramientas exclusivas de macOS (Canvas, Cámara, Grabación de Pantalla, `system.run`).
- Inicia el servicio local de node host en modo **remoto** (launchd), y lo detiene en modo **local**.
- Opcionalmente aloja **PeekabooBridge** para automatización de UI.
- Instala la CLI global (`openclaw`) bajo demanda mediante npm, pnpm o bun (la app prefiere npm, luego pnpm y después bun; Node sigue siendo el tiempo de ejecución recomendado para el Gateway).

## Modo local vs remoto

- **Local** (predeterminado): la app se conecta a un Gateway local ya en ejecución si existe;
  en caso contrario habilita el servicio launchd mediante `openclaw gateway install`.
- **Remoto**: la app se conecta a un Gateway por SSH/Tailscale y nunca inicia
  un proceso local.
  La app inicia el **servicio local de node host** para que el Gateway remoto pueda alcanzar este Mac.
  La app no inicia el Gateway como proceso hijo.
  El descubrimiento del Gateway ahora prefiere nombres MagicDNS de Tailscale frente a IP tailnet sin procesar,
  de modo que la app de Mac se recupera con más fiabilidad cuando cambian las IP tailnet.

## Control de launchd

La app gestiona un LaunchAgent por usuario con la etiqueta `ai.openclaw.gateway`
(o `ai.openclaw.<profile>` al usar `--profile`/`OPENCLAW_PROFILE`; el heredado `com.openclaw.*` sigue descargándose).

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Sustituye la etiqueta por `ai.openclaw.<profile>` cuando ejecutes un profile con nombre.

Si el LaunchAgent no está instalado, actívalo desde la app o ejecuta
`openclaw gateway install`.

## Capacidades del nodo (mac)

La app de macOS se presenta como un nodo. Comandos habituales:

- Canvas: `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- Cámara: `camera.snap`, `camera.clip`
- Pantalla: `screen.record`
- Sistema: `system.run`, `system.notify`

El nodo informa un mapa `permissions` para que los agentes puedan decidir qué está permitido.

Servicio de nodo + IPC de la app:

- Cuando el servicio headless de node host está en ejecución (modo remoto), se conecta al WS del Gateway como un nodo.
- `system.run` se ejecuta en la app de macOS (contexto UI/TCC) mediante un socket Unix local; los prompts y la salida permanecen en la app.

Diagrama (SCI):

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## Aprobaciones de exec (`system.run`)

`system.run` está controlado por **Exec approvals** en la app de macOS (Settings → Exec approvals).
Security + ask + allowlist se almacenan localmente en el Mac en:

```
~/.openclaw/exec-approvals.json
```

Ejemplo:

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

Notas:

- Las entradas de `allowlist` son patrones glob para rutas resueltas de binarios.
- El texto bruto del comando de shell que contiene sintaxis de control o expansión del shell (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) se trata como un fallo de allowlist y requiere aprobación explícita (o incluir el binario del shell en la allowlist).
- Elegir “Always Allow” en el prompt agrega ese comando a la allowlist.
- Las sobrescrituras del entorno en `system.run` se filtran (elimina `PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`) y luego se fusionan con el entorno de la app.
- Para wrappers de shell (`bash|sh|zsh ... -c/-lc`), las sobrescrituras del entorno con alcance de solicitud se reducen a una pequeña allowlist explícita (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Para decisiones de permitir siempre en modo allowlist, los wrappers de despacho conocidos (`env`, `nice`, `nohup`, `stdbuf`, `timeout`) persisten rutas internas del ejecutable en lugar de rutas del wrapper. Si el desempaquetado no es seguro, no se persiste automáticamente ninguna entrada de allowlist.

## Deep links

La app registra el esquema de URL `openclaw://` para acciones locales.

### `openclaw://agent`

Activa una solicitud `agent` del Gateway.
__OC_I18N_900004__
Parámetros de consulta:

- `message` (obligatorio)
- `sessionKey` (opcional)
- `thinking` (opcional)
- `deliver` / `to` / `channel` (opcional)
- `timeoutSeconds` (opcional)
- `key` (opcional, clave de modo unattended)

Seguridad:

- Sin `key`, la app solicita confirmación.
- Sin `key`, la app aplica un límite corto de mensaje para el prompt de confirmación e ignora `deliver` / `to` / `channel`.
- Con una `key` válida, la ejecución es unattended (pensado para automatizaciones personales).

## Flujo de onboarding (típico)

1. Instala e inicia **OpenClaw.app**.
2. Completa la lista de comprobación de permisos (prompts de TCC).
3. Asegúrate de que el modo **Local** esté activo y el Gateway esté en ejecución.
4. Instala la CLI si quieres acceso desde terminal.

## Ubicación del directorio de state (macOS)

Evita poner tu directorio de state de OpenClaw en iCloud u otras carpetas sincronizadas con la nube.
Las rutas respaldadas por sincronización pueden añadir latencia y ocasionalmente provocar carreras de bloqueo/sincronización de archivos para
sesiones y credenciales.

Prefiere una ruta local no sincronizada como:
__OC_I18N_900005__
Si `openclaw doctor` detecta state bajo:

- `~/Library/Mobile Documents/com~apple~CloudDocs/...`
- `~/Library/CloudStorage/...`

advertirá y recomendará volver a una ruta local.

## Flujo de compilación y desarrollo (nativo)

- `cd apps/macos && swift build`
- `swift run OpenClaw` (o Xcode)
- Empaquetar la app: `scripts/package-mac-app.sh`

## Depurar conectividad con el gateway (CLI de macOS)

Usa la CLI de depuración para ejercitar el mismo handshake WebSocket y la lógica de descubrimiento del Gateway
que usa la app de macOS, sin iniciar la app.
__OC_I18N_900006__
Opciones de conexión:

- `--url <ws://host:port>`: sobrescribe la configuración
- `--mode <local|remote>`: resuelve desde la configuración (predeterminado: configuración o local)
- `--probe`: fuerza una sonda nueva de estado
- `--timeout <ms>`: tiempo de espera de la solicitud (predeterminado: `15000`)
- `--json`: salida estructurada para comparar diferencias

Opciones de descubrimiento:

- `--include-local`: incluye gateways que normalmente se filtrarían como “local”
- `--timeout <ms>`: ventana total de descubrimiento (predeterminado: `2000`)
- `--json`: salida estructurada para comparar diferencias

Consejo: compáralo con `openclaw gateway discover --json` para ver si la
canalización de descubrimiento de la app de macOS (`local.` más el dominio amplio configurado, con
respaldo de wide-area y Tailscale Serve) difiere del descubrimiento basado en `dns-sd`
de la CLI de Node.

## Infraestructura de conexión remota (túneles SSH)

Cuando la app de macOS se ejecuta en modo **remoto**, abre un túnel SSH para que los componentes de UI locales
puedan comunicarse con un Gateway remoto como si estuviera en localhost.

### Túnel de control (puerto WebSocket del Gateway)

- **Propósito:** comprobaciones de estado, status, Web Chat, configuración y otras llamadas del plano de control.
- **Puerto local:** el puerto del Gateway (predeterminado `18789`), siempre estable.
- **Puerto remoto:** el mismo puerto del Gateway en el host remoto.
- **Comportamiento:** no hay puerto local aleatorio; la app reutiliza un túnel saludable existente
  o lo reinicia si es necesario.
- **Forma de SSH:** `ssh -N -L <local>:127.0.0.1:<remote>` con opciones BatchMode +
  ExitOnForwardFailure + keepalive.
- **Informe de IP:** el túnel SSH usa loopback, así que el gateway verá la
  IP del nodo como `127.0.0.1`. Usa transporte **Direct (ws/wss)** si quieres que aparezca la IP
  real del cliente (consulta [acceso remoto en macOS](/platforms/mac/remote)).

Para los pasos de configuración, consulta [acceso remoto en macOS](/platforms/mac/remote). Para
detalles del protocolo, consulta [Gateway protocol](/gateway/protocol).

## Documentación relacionada

- [Gateway runbook](/gateway)
- [Gateway (macOS)](/platforms/mac/bundled-gateway)
- [permisos de macOS](/platforms/mac/permissions)
- [Canvas](/platforms/mac/canvas)
