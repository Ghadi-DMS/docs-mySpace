---
read_when:
    - Usar o modificar la herramienta exec
    - Depurar el comportamiento de stdin o TTY
summary: Uso de la herramienta exec, modos de stdin y compatibilidad con TTY
title: Herramienta Exec
x-i18n:
    generated_at: "2026-04-05T12:55:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: b73e9900c109910fc4e178c888b7ad7f3a4eeaa34eb44bc816abba9af5d664d7
    source_path: tools/exec.md
    workflow: 15
---

# Herramienta exec

Ejecuta comandos de shell en el espacio de trabajo. Admite ejecución en primer plano y en segundo plano mediante `process`.
Si `process` no está permitido, `exec` se ejecuta de forma síncrona e ignora `yieldMs`/`background`.
Las sesiones en segundo plano tienen alcance por agente; `process` solo ve sesiones del mismo agente.

## Parámetros

- `command` (obligatorio)
- `workdir` (por defecto, cwd)
- `env` (sobrescrituras de clave/valor)
- `yieldMs` (predeterminado 10000): pasa automáticamente a segundo plano después del retraso
- `background` (bool): pasa a segundo plano inmediatamente
- `timeout` (segundos, predeterminado 1800): finaliza al expirar
- `pty` (bool): se ejecuta en un pseudo-terminal cuando está disponible (CLI solo TTY, agentes de codificación, interfaces de terminal)
- `host` (`auto | sandbox | gateway | node`): dónde ejecutar
- `security` (`deny | allowlist | full`): modo de aplicación para `gateway`/`node`
- `ask` (`off | on-miss | always`): solicitudes de aprobación para `gateway`/`node`
- `node` (string): id/nombre del nodo para `host=node`
- `elevated` (bool): solicita modo elevado (salir del sandbox hacia la ruta de host configurada); `security=full` solo se fuerza cuando elevated se resuelve a `full`

Notas:

- `host` usa `auto` de forma predeterminada: sandbox cuando el entorno de ejecución sandbox está activo para la sesión; en caso contrario, gateway.
- `auto` es la estrategia de enrutamiento predeterminada, no un comodín. Se permite `host=node` por llamada desde `auto`; `host=gateway` por llamada solo se permite cuando no hay un entorno de ejecución sandbox activo.
- Sin configuración adicional, `host=auto` sigue “simplemente funcionando”: sin sandbox se resuelve a `gateway`; con un sandbox activo permanece en el sandbox.
- `elevated` sale del sandbox hacia la ruta de host configurada: `gateway` de forma predeterminada, o `node` cuando `tools.exec.host=node` (o el valor predeterminado de la sesión es `host=node`). Solo está disponible cuando el acceso elevado está habilitado para la sesión/proveedor actual.
- Las aprobaciones de `gateway`/`node` están controladas por `~/.openclaw/exec-approvals.json`.
- `node` requiere un nodo emparejado (app complementaria o host de nodo sin interfaz).
- Si hay varios nodos disponibles, establece `exec.node` o `tools.exec.node` para seleccionar uno.
- `exec host=node` es la única ruta de ejecución de shell para nodos; el contenedor heredado `nodes.run` se ha eliminado.
- En hosts que no son Windows, exec usa `SHELL` cuando está definido; si `SHELL` es `fish`, prefiere `bash` (o `sh`)
  desde `PATH` para evitar scripts incompatibles con fish, y después vuelve a `SHELL` si ninguno existe.
- En hosts Windows, exec prefiere descubrir PowerShell 7 (`pwsh`) (Program Files, ProgramW6432 y luego PATH),
  y después vuelve a Windows PowerShell 5.1.
- La ejecución en host (`gateway`/`node`) rechaza `env.PATH` y las sobrescrituras del cargador (`LD_*`/`DYLD_*`) para
  evitar el secuestro de binarios o la inyección de código.
- OpenClaw establece `OPENCLAW_SHELL=exec` en el entorno del comando generado (incluida la ejecución en PTY y sandbox) para que las reglas del shell/perfil puedan detectar el contexto de la herramienta exec.
- Importante: el sandboxing está **desactivado de forma predeterminada**. Si el sandboxing está desactivado, `host=auto`
  implícito se resuelve a `gateway`. `host=sandbox` explícito sigue fallando de forma segura en lugar de
  ejecutarse silenciosamente en el host gateway. Habilita el sandboxing o usa `host=gateway` con aprobaciones.
- Las comprobaciones previas de scripts (para errores comunes de sintaxis de shell en Python/Node) solo inspeccionan archivos dentro del
  límite efectivo de `workdir`. Si una ruta de script se resuelve fuera de `workdir`, se omite la comprobación previa para
  ese archivo.
- Para trabajos de larga duración que comienzan ahora, inícialos una vez y confía en la activación automática
  al completarse cuando esté habilitada y el comando emita salida o falle.
  Usa `process` para registros, estado, entrada o intervención; no emules
  programación con bucles de espera, bucles de tiempo de espera o sondeos repetidos.
- Para trabajos que deben ocurrir más tarde o según una programación, usa cron en lugar de
  patrones de suspensión/retraso con `exec`.

## Configuración

- `tools.exec.notifyOnExit` (predeterminado: true): cuando es true, las sesiones exec pasadas a segundo plano ponen en cola un evento del sistema y solicitan un heartbeat al salir.
- `tools.exec.approvalRunningNoticeMs` (predeterminado: 10000): emite un único aviso de “en ejecución” cuando un exec con aprobación tarda más que esto (0 lo desactiva).
- `tools.exec.host` (predeterminado: `auto`; se resuelve a `sandbox` cuando el entorno de ejecución sandbox está activo; en caso contrario, `gateway`)
- `tools.exec.security` (predeterminado: `deny` para sandbox, `full` para gateway + node cuando no está establecido)
- `tools.exec.ask` (predeterminado: `off`)
- La ejecución en host sin aprobación es el valor predeterminado para gateway + node. Si quieres comportamiento de aprobaciones/lista permitida, ajusta tanto `tools.exec.*` como el host `~/.openclaw/exec-approvals.json`; consulta [Aprobaciones de exec](/tools/exec-approvals#no-approval-yolo-mode).
- YOLO proviene de los valores predeterminados de la política del host (`security=full`, `ask=off`), no de `host=auto`. Si quieres forzar el enrutamiento a gateway o node, establece `tools.exec.host` o usa `/exec host=...`.
- `tools.exec.node` (predeterminado: sin establecer)
- `tools.exec.strictInlineEval` (predeterminado: false): cuando es true, las formas de evaluación inline del intérprete como `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e` y `osascript -e` siempre requieren aprobación explícita. `allow-always` aún puede conservar invocaciones benignas de intérprete/script, pero las formas inline-eval siguen solicitando aprobación cada vez.
- `tools.exec.pathPrepend`: lista de directorios que se anteponen a `PATH` para ejecuciones exec (solo gateway + sandbox).
- `tools.exec.safeBins`: binarios seguros solo de stdin que pueden ejecutarse sin entradas explícitas en la lista permitida. Para detalles del comportamiento, consulta [Safe bins](/tools/exec-approvals#safe-bins-stdin-only).
- `tools.exec.safeBinTrustedDirs`: directorios explícitos adicionales de confianza para comprobaciones de ruta de `safeBins`. Las entradas de `PATH` nunca reciben confianza automática. Los valores predeterminados integrados son `/bin` y `/usr/bin`.
- `tools.exec.safeBinProfiles`: política opcional de argv personalizada por safe bin (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).

Ejemplo:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Gestión de PATH

- `host=gateway`: combina tu `PATH` del shell de inicio de sesión con el entorno exec. Las sobrescrituras de `env.PATH` se
  rechazan para la ejecución en host. El propio daemon sigue ejecutándose con un `PATH` mínimo:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: ejecuta `sh -lc` (shell de inicio de sesión) dentro del contenedor, por lo que `/etc/profile` puede restablecer `PATH`.
  OpenClaw antepone `env.PATH` después de cargar el perfil mediante una variable de entorno interna (sin interpolación de shell);
  `tools.exec.pathPrepend` también se aplica aquí.
- `host=node`: solo se envían al nodo las sobrescrituras de entorno no bloqueadas que proporciones. Las sobrescrituras de `env.PATH` se
  rechazan para la ejecución en host y se ignoran en hosts de nodo. Si necesitas entradas adicionales en PATH en un nodo,
  configura el entorno del servicio del host del nodo (systemd/launchd) o instala herramientas en ubicaciones estándar.

Vinculación de nodo por agente (usa el índice de la lista de agentes en la configuración):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

UI de control: la pestaña Nodos incluye un pequeño panel “Vinculación de nodo exec” para la misma configuración.

## Sobrescrituras de sesión (`/exec`)

Usa `/exec` para establecer valores predeterminados **por sesión** para `host`, `security`, `ask` y `node`.
Envía `/exec` sin argumentos para mostrar los valores actuales.

Ejemplo:

```
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

## Modelo de autorización

`/exec` solo se respeta para **remitentes autorizados** (listas permitidas/emparejamiento del canal más `commands.useAccessGroups`).
Actualiza **solo el estado de la sesión** y no escribe en la configuración. Para desactivar exec de forma permanente, niégalo mediante la
política de herramientas (`tools.deny: ["exec"]` o por agente). Las aprobaciones del host siguen aplicándose a menos que establezcas explícitamente
`security=full` y `ask=off`.

## Aprobaciones de exec (app complementaria / host de nodo)

Los agentes en sandbox pueden requerir aprobación por solicitud antes de que `exec` se ejecute en el host gateway o node.
Consulta [Aprobaciones de exec](/tools/exec-approvals) para ver la política, la lista permitida y el flujo de la UI.

Cuando se requieren aprobaciones, la herramienta exec devuelve inmediatamente
`status: "approval-pending"` y un id de aprobación. Una vez aprobada (o denegada / expirada),
el Gateway emite eventos del sistema (`Exec finished` / `Exec denied`). Si el comando sigue
ejecutándose después de `tools.exec.approvalRunningNoticeMs`, se emite un único aviso `Exec running`.
En canales con tarjetas/botones de aprobación nativos, el agente debe apoyarse
primero en esa UI nativa e incluir un comando manual `/approve` solo cuando el
resultado de la herramienta indique explícitamente que las aprobaciones por chat no están disponibles o que la aprobación manual es la
única vía.

## Lista permitida + safe bins

La aplicación manual de la lista permitida coincide **solo con rutas de binarios resueltas** (sin coincidencias por nombre base). Cuando
`security=allowlist`, los comandos de shell se permiten automáticamente solo si cada segmento de la canalización está
en la lista permitida o es un safe bin. El encadenamiento (`;`, `&&`, `||`) y las redirecciones se rechazan en
modo allowlist a menos que cada segmento de nivel superior satisfaga la lista permitida (incluidos los safe bins).
Las redirecciones siguen sin ser compatibles.
La confianza duradera `allow-always` no evita esa regla: un comando encadenado sigue requiriendo que cada
segmento de nivel superior coincida.

`autoAllowSkills` es una ruta de conveniencia independiente dentro de las aprobaciones de exec. No es lo mismo que
las entradas manuales de lista permitida por ruta. Para una confianza estricta y explícita, mantén `autoAllowSkills` desactivado.

Usa los dos controles para tareas distintas:

- `tools.exec.safeBins`: pequeños filtros de flujo solo stdin.
- `tools.exec.safeBinTrustedDirs`: directorios adicionales explícitos de confianza para rutas ejecutables de safe bin.
- `tools.exec.safeBinProfiles`: política argv explícita para safe bins personalizados.
- allowlist: confianza explícita para rutas ejecutables.

No trates `safeBins` como una lista permitida genérica, y no añadas binarios de intérprete/entorno de ejecución (por ejemplo `python3`, `node`, `ruby`, `bash`). Si los necesitas, usa entradas explícitas de lista permitida y mantén habilitadas las solicitudes de aprobación.
`openclaw security audit` avisa cuando faltan entradas explícitas de perfil en `safeBins` para intérpretes/entornos de ejecución, y `openclaw doctor --fix` puede generar entradas personalizadas faltantes de `safeBinProfiles`.
`openclaw security audit` y `openclaw doctor` también avisan cuando vuelves a añadir explícitamente bins de comportamiento amplio como `jq` a `safeBins`.
Si añades intérpretes explícitamente a la lista permitida, habilita `tools.exec.strictInlineEval` para que las formas de evaluación inline de código sigan requiriendo una aprobación nueva.

Para ver todos los detalles y ejemplos de la política, consulta [Aprobaciones de exec](/tools/exec-approvals#safe-bins-stdin-only) y [Safe bins frente a allowlist](/tools/exec-approvals#safe-bins-versus-allowlist).

## Ejemplos

Primer plano:

```json
{ "tool": "exec", "command": "ls -la" }
```

Segundo plano + sondeo:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

El sondeo es para estado bajo demanda, no para bucles de espera. Si la activación automática
al completarse está habilitada, el comando puede reactivar la sesión cuando emita salida o falle.

Enviar teclas (estilo tmux):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Enviar (solo enviar CR):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Pegar (entre delimitadores por defecto):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` es una subherramienta de `exec` para ediciones estructuradas en varios archivos.
Está habilitada de forma predeterminada para modelos de OpenAI y OpenAI Codex. Usa configuración solo
cuando quieras desactivarla o restringirla a modelos específicos:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.4"] },
    },
  },
}
```

Notas:

- Solo está disponible para modelos de OpenAI/OpenAI Codex.
- La política de herramientas sigue aplicándose; `allow: ["write"]` permite implícitamente `apply_patch`.
- La configuración vive en `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` tiene como valor predeterminado `true`; establécelo en `false` para desactivar la herramienta en modelos OpenAI.
- `tools.exec.applyPatch.workspaceOnly` tiene como valor predeterminado `true` (contenido dentro del espacio de trabajo). Establécelo en `false` solo si intencionalmente quieres que `apply_patch` escriba/elimine fuera del directorio del espacio de trabajo.

## Relacionado

- [Aprobaciones de exec](/tools/exec-approvals) — compuertas de aprobación para comandos de shell
- [Sandboxing](/es/gateway/sandboxing) — ejecutar comandos en entornos con sandbox
- [Proceso en segundo plano](/es/gateway/background-process) — exec de larga duración y herramienta process
- [Seguridad](/es/gateway/security) — política de herramientas y acceso elevado
