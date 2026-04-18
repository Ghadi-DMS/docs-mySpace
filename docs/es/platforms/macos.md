---
read_when:
    - Implementación de funciones de la aplicación para macOS
    - Cambio del ciclo de vida del Gateway o del puenteo de nodos en macOS
summary: Aplicación complementaria de OpenClaw para macOS (barra de menús + intermediario del Gateway)
title: Aplicación para macOS
x-i18n:
    generated_at: "2026-04-18T04:59:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: d637df2f73ced110223c48ea3c934045d782e150a46495f434cf924a6a00baf0
    source_path: platforms/macos.md
    workflow: 15
---

# Complemento de OpenClaw para macOS (barra de menús + intermediario del Gateway)

La aplicación de macOS es el **complemento de barra de menús** para OpenClaw. Se encarga de los permisos,
administra/se conecta al Gateway localmente (launchd o manual) y expone capacidades de macOS
al agente como un Node.

## Qué hace

- Muestra notificaciones nativas y estado en la barra de menús.
- Gestiona los avisos de TCC (Notificaciones, Accesibilidad, Grabación de pantalla, Micrófono,
  Reconocimiento de voz, Automatización/AppleScript).
- Ejecuta o se conecta al Gateway (local o remoto).
- Expone herramientas exclusivas de macOS (Canvas, Camera, Screen Recording, `system.run`).
- Inicia el servicio local del host de Node en modo **remoto** (launchd) y lo detiene en modo **local**.
- Puede alojar opcionalmente **PeekabooBridge** para automatización de UI.
- Instala la CLI global (`openclaw`) a pedido mediante npm, pnpm o bun (la aplicación prefiere npm, luego pnpm y luego bun; Node sigue siendo el entorno de ejecución recomendado para Gateway).

## Modo local vs. remoto

- **Local** (predeterminado): la aplicación se conecta a un Gateway local en ejecución si existe;
  de lo contrario, habilita el servicio launchd mediante `openclaw gateway install`.
- **Remoto**: la aplicación se conecta a un Gateway a través de SSH/Tailscale y nunca inicia
  un proceso local.
  La aplicación inicia el **servicio local del host de Node** para que el Gateway remoto pueda acceder a esta Mac.
  La aplicación no genera el Gateway como un proceso hijo.
  La detección del Gateway ahora prioriza los nombres MagicDNS de Tailscale sobre las IP sin procesar de la tailnet,
  por lo que la aplicación para Mac se recupera de forma más confiable cuando cambian las IP de la tailnet.

## Control de launchd

La aplicación administra un LaunchAgent por usuario con la etiqueta `ai.openclaw.gateway`
(o `ai.openclaw.<profile>` cuando se usa `--profile`/`OPENCLAW_PROFILE`; el legado `com.openclaw.*` todavía se descarga).

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Reemplaza la etiqueta por `ai.openclaw.<profile>` cuando ejecutes un perfil con nombre.

Si el LaunchAgent no está instalado, habilítalo desde la aplicación o ejecuta
`openclaw gateway install`.

## Capacidades del Node (mac)

La aplicación de macOS se presenta como un Node. Comandos comunes:

- Canvas: `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- Camera: `camera.snap`, `camera.clip`
- Pantalla: `screen.snapshot`, `screen.record`
- Sistema: `system.run`, `system.notify`

El Node informa un mapa `permissions` para que los agentes puedan decidir qué está permitido.

Servicio de Node + IPC de la aplicación:

- Cuando el servicio sin interfaz del host de Node está en ejecución (modo remoto), se conecta al Gateway WS como un Node.
- `system.run` se ejecuta en la aplicación de macOS (contexto UI/TCC) a través de un socket Unix local; los avisos y la salida permanecen dentro de la aplicación.

Diagrama (SCI):

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## Aprobaciones de ejecución (`system.run`)

`system.run` está controlado por las **aprobaciones de ejecución** en la aplicación de macOS (Ajustes → Aprobaciones de ejecución).
La seguridad + solicitud + lista de permitidos se almacenan localmente en la Mac en:

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
- El texto de comandos de shell sin procesar que contiene sintaxis de control o expansión del shell (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) se trata como una ausencia en la lista de permitidos y requiere aprobación explícita (o añadir el binario del shell a la lista de permitidos).
- Elegir “Permitir siempre” en el aviso añade ese comando a la lista de permitidos.
- Las anulaciones de entorno de `system.run` se filtran (elimina `PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`) y luego se fusionan con el entorno de la aplicación.
- Para envoltorios de shell (`bash|sh|zsh ... -c/-lc`), las anulaciones de entorno con alcance de solicitud se reducen a una pequeña lista de permitidos explícita (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Para decisiones de permitir siempre en modo de lista de permitidos, los envoltorios de despacho conocidos (`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservan las rutas del ejecutable interno en lugar de las rutas del envoltorio. Si desenvolverlo no es seguro, no se conserva automáticamente ninguna entrada en la lista de permitidos.

## Enlaces profundos

La aplicación registra el esquema de URL `openclaw://` para acciones locales.

### `openclaw://agent`

Activa una solicitud `agent` del Gateway.
__OC_I18N_900004__
Parámetros de consulta:

- `message` (obligatorio)
- `sessionKey` (opcional)
- `thinking` (opcional)
- `deliver` / `to` / `channel` (opcional)
- `timeoutSeconds` (opcional)
- `key` (opcional, clave para modo no supervisado)

Seguridad:

- Sin `key`, la aplicación solicita confirmación.
- Sin `key`, la aplicación aplica un límite corto al mensaje para el aviso de confirmación e ignora `deliver` / `to` / `channel`.
- Con una `key` válida, la ejecución no es supervisada (pensado para automatizaciones personales).

## Flujo de incorporación (típico)

1. Instala y abre **OpenClaw.app**.
2. Completa la lista de comprobación de permisos (avisos de TCC).
3. Asegúrate de que el modo **Local** esté activo y de que el Gateway esté en ejecución.
4. Instala la CLI si quieres acceso desde la terminal.

## Ubicación del directorio de estado (macOS)

Evita colocar el directorio de estado de OpenClaw en iCloud u otras carpetas sincronizadas con la nube.
Las rutas respaldadas por sincronización pueden añadir latencia y, ocasionalmente, provocar condiciones de carrera de bloqueo/sincronización de archivos para
sesiones y credenciales.

Prefiere una ruta de estado local no sincronizada, como:
__OC_I18N_900005__
Si `openclaw doctor` detecta el estado en:

- `~/Library/Mobile Documents/com~apple~CloudDocs/...`
- `~/Library/CloudStorage/...`

mostrará una advertencia y recomendará volver a una ruta local.

## Flujo de compilación y desarrollo (nativo)

- `cd apps/macos && swift build`
- `swift run OpenClaw` (o Xcode)
- Empaquetar la aplicación: `scripts/package-mac-app.sh`

## Depurar la conectividad del Gateway (CLI de macOS)

Usa la CLI de depuración para probar el mismo protocolo de enlace WebSocket del Gateway y la misma lógica de detección
que usa la aplicación de macOS, sin abrir la aplicación.
__OC_I18N_900006__
Opciones de conexión:

- `--url <ws://host:port>`: anula la configuración
- `--mode <local|remote>`: resuelve desde la configuración (predeterminado: config o local)
- `--probe`: fuerza una nueva comprobación de estado
- `--timeout <ms>`: tiempo de espera de la solicitud (predeterminado: `15000`)
- `--json`: salida estructurada para comparar diferencias

Opciones de detección:

- `--include-local`: incluye Gateways que se filtrarían como “locales”
- `--timeout <ms>`: ventana total de detección (predeterminada: `2000`)
- `--json`: salida estructurada para comparar diferencias

Consejo: compáralo con `openclaw gateway discover --json` para ver si la
canalización de detección de la aplicación de macOS (`local.` más el dominio de área amplia configurado, con
área amplia y Tailscale Serve como alternativas)
difiere de la detección basada en `dns-sd` de la CLI de Node.

## Infraestructura de conexión remota (túneles SSH)

Cuando la aplicación de macOS se ejecuta en modo **Remoto**, abre un túnel SSH para que los componentes
de UI locales puedan comunicarse con un Gateway remoto como si estuviera en localhost.

### Túnel de control (puerto WebSocket del Gateway)

- **Propósito:** comprobaciones de estado, estado, Web Chat, configuración y otras llamadas del plano de control.
- **Puerto local:** el puerto del Gateway (predeterminado `18789`), siempre estable.
- **Puerto remoto:** el mismo puerto del Gateway en el host remoto.
- **Comportamiento:** no se usa un puerto local aleatorio; la aplicación reutiliza un túnel saludable existente
  o lo reinicia si es necesario.
- **Forma de SSH:** `ssh -N -L <local>:127.0.0.1:<remote>` con opciones BatchMode +
  ExitOnForwardFailure + keepalive.
- **Informe de IP:** el túnel SSH usa loopback, por lo que el gateway verá la IP del node
  como `127.0.0.1`. Usa el transporte **Direct (ws/wss)** si quieres que aparezca la IP real
  del cliente (consulta [acceso remoto en macOS](/es/platforms/mac/remote)).

Para ver los pasos de configuración, consulta [acceso remoto en macOS](/es/platforms/mac/remote). Para detalles
del protocolo, consulta [protocolo del Gateway](/es/gateway/protocol).

## Documentación relacionada

- [Guía operativa del Gateway](/es/gateway)
- [Gateway (macOS)](/es/platforms/mac/bundled-gateway)
- [Permisos de macOS](/es/platforms/mac/permissions)
- [Canvas](/es/platforms/mac/canvas)
