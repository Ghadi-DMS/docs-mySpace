---
read_when:
    - Configurar aprobaciones o listas de permitidos de exec
    - Implementar la UX de aprobación de exec en la app de macOS
    - Revisar los prompts de escape del sandbox y sus implicaciones
summary: Aprobaciones de exec, listas de permitidos y prompts de escape del sandbox
title: Aprobaciones de exec
x-i18n:
    generated_at: "2026-04-08T02:19:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6041929185bab051ad873cc4822288cb7d6f0470e19e7ae7a16b70f76dfc2cd9
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Aprobaciones de exec

Las aprobaciones de exec son la **medida de protección de la app complementaria / host del nodo** para permitir que un agente en sandbox ejecute
comandos en un host real (`gateway` o `node`). Piense en ello como un enclavamiento de seguridad:
los comandos solo se permiten cuando la política + la lista de permitidos + la aprobación del usuario (opcional) coinciden.
Las aprobaciones de exec son **adicionales** a la política de herramientas y al control elevated (a menos que elevated esté en `full`, lo que omite las aprobaciones).
La política efectiva es la **más estricta** entre `tools.exec.*` y los valores predeterminados de aprobaciones; si se omite un campo de aprobaciones, se usa el valor de `tools.exec`.
El exec del host también usa el estado local de aprobaciones en esa máquina. Un valor local del host
`ask: "always"` en `~/.openclaw/exec-approvals.json` sigue mostrando prompts incluso si
los valores predeterminados de la sesión o de la configuración solicitan `ask: "on-miss"`.
Use `openclaw approvals get`, `openclaw approvals get --gateway` o
`openclaw approvals get --node <id|name|ip>` para inspeccionar la política solicitada,
las fuentes de política del host y el resultado efectivo.

Si la UI de la app complementaria **no está disponible**, cualquier solicitud que requiera un prompt se
resuelve mediante el **respaldo de ask** (predeterminado: denegar).

Los clientes nativos de aprobación por chat también pueden exponer elementos específicos del canal en el
mensaje de aprobación pendiente. Por ejemplo, Matrix puede sembrar accesos directos de reacciones en el
prompt de aprobación (`✅` permitir una vez, `❌` denegar y `♾️` permitir siempre cuando esté disponible)
sin dejar de mantener los comandos `/approve ...` en el mensaje como alternativa.

## Dónde se aplica

Las aprobaciones de exec se aplican localmente en el host de ejecución:

- **host gateway** → proceso `openclaw` en la máquina gateway
- **host node** → ejecutor del nodo (app complementaria de macOS o host node sin interfaz)

Nota sobre el modelo de confianza:

- Las personas que llaman autenticadas por gateway son operadores de confianza para ese Gateway.
- Los nodos emparejados extienden esa capacidad de operador de confianza al host del nodo.
- Las aprobaciones de exec reducen el riesgo de ejecución accidental, pero no son un límite de autenticación por usuario.
- Las ejecuciones aprobadas en el host del nodo vinculan el contexto de ejecución canónico: `cwd` canónico, `argv` exacto, vinculación de variables de entorno
  cuando está presente y ruta de ejecutable fijada cuando corresponde.
- Para scripts de shell e invocaciones directas de archivos de intérprete/runtime, OpenClaw también intenta vincular
  un único operando de archivo local concreto. Si ese archivo vinculado cambia después de la aprobación y antes de la ejecución,
  la ejecución se deniega en lugar de ejecutar contenido desviado.
- Esta vinculación de archivos es intencionalmente de mejor esfuerzo, no un modelo semántico completo de todas las
  rutas de carga de intérpretes/runtimes. Si el modo de aprobación no puede identificar exactamente un archivo local concreto
  que vincular, se niega a emitir una ejecución respaldada por aprobación en lugar de fingir cobertura total.

División en macOS:

- El **servicio del host del nodo** reenvía `system.run` a la **app de macOS** mediante IPC local.
- La **app de macOS** aplica las aprobaciones + ejecuta el comando en el contexto de la UI.

## Configuración y almacenamiento

Las aprobaciones viven en un archivo JSON local en el host de ejecución:

`~/.openclaw/exec-approvals.json`

Ejemplo de esquema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Modo "YOLO" sin aprobación

Si quiere que el exec del host se ejecute sin prompts de aprobación, debe abrir **ambas** capas de política:

- la política solicitada de exec en la configuración de OpenClaw (`tools.exec.*`)
- la política local de aprobaciones del host en `~/.openclaw/exec-approvals.json`

Este es ahora el comportamiento predeterminado del host, a menos que lo endurezca explícitamente:

- `tools.exec.security`: `full` en `gateway`/`node`
- `tools.exec.ask`: `off`
- `askFallback` del host: `full`

Distinción importante:

- `tools.exec.host=auto` elige dónde se ejecuta exec: sandbox cuando está disponible, en caso contrario gateway.
- YOLO elige cómo se aprueba el exec del host: `security=full` más `ask=off`.
- En modo YOLO, OpenClaw no añade un control adicional de aprobación por ofuscación heurística de comandos encima de la política configurada de exec del host.
- `auto` no convierte el enrutamiento al gateway en una omisión gratuita desde una sesión en sandbox. Se permite una solicitud por llamada `host=node` desde `auto`, y `host=gateway` solo se permite desde `auto` cuando no hay un runtime de sandbox activo. Si quiere un valor predeterminado estable no automático, configure `tools.exec.host` o use `/exec host=...` explícitamente.

Si quiere una configuración más conservadora, vuelva a endurecer cualquiera de las capas a `allowlist` / `on-miss`
o `deny`.

Configuración persistente de "nunca pedir" en el host gateway:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Luego configure el archivo de aprobaciones del host para que coincida:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Para un host node, aplique en su lugar el mismo archivo de aprobaciones en ese nodo:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Acceso directo solo para la sesión:

- `/exec security=full ask=off` cambia solo la sesión actual.
- `/elevated full` es un acceso directo de emergencia que también omite las aprobaciones de exec para esa sesión.

Si el archivo de aprobaciones del host sigue siendo más estricto que la configuración, la política más estricta del host sigue prevaleciendo.

## Controles de política

### Seguridad (`exec.security`)

- **deny**: bloquea todas las solicitudes de exec del host.
- **allowlist**: permite solo comandos de la lista de permitidos.
- **full**: permite todo (equivalente a elevated).

### Ask (`exec.ask`)

- **off**: nunca mostrar prompt.
- **on-miss**: mostrar prompt solo cuando la lista de permitidos no coincide.
- **always**: mostrar prompt en cada comando.
- la confianza persistente `allow-always` no suprime los prompts cuando el modo ask efectivo es `always`

### Respaldo de ask (`askFallback`)

Si se requiere un prompt pero no hay ninguna UI accesible, el respaldo decide:

- **deny**: bloquear.
- **allowlist**: permitir solo si la lista de permitidos coincide.
- **full**: permitir.

### Endurecimiento de eval en línea del intérprete (`tools.exec.strictInlineEval`)

Cuando `tools.exec.strictInlineEval=true`, OpenClaw trata las formas de evaluación de código en línea como exclusivas de aprobación, incluso si el binario del intérprete en sí está en la lista de permitidos.

Ejemplos:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Esto es una defensa en profundidad para cargadores de intérpretes que no se asignan claramente a un único operando de archivo estable. En modo estricto:

- estos comandos siguen necesitando aprobación explícita;
- `allow-always` no conserva automáticamente nuevas entradas de lista de permitidos para ellos.

## Lista de permitidos (por agente)

Las listas de permitidos son **por agente**. Si existen varios agentes, cambie cuál está
editando en la app de macOS. Los patrones son **coincidencias de glob sin distinción entre mayúsculas y minúsculas**.
Los patrones deben resolverse a **rutas de binarios** (las entradas solo con basename se ignoran).
Las entradas heredadas `agents.default` se migran a `agents.main` al cargar.
Las cadenas de shell como `echo ok && pwd` siguen requiriendo que cada segmento de nivel superior satisfaga las reglas de la lista de permitidos.

Ejemplos:

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Cada entrada de la lista de permitidos registra:

- **id** UUID estable usado para la identidad en la UI (opcional)
- **último uso** marca de tiempo
- **último comando usado**
- **última ruta resuelta**

## Permitir automáticamente CLIs de Skills

Cuando **Permitir automáticamente CLIs de Skills** está habilitado, los ejecutables referenciados por Skills conocidas
se tratan como incluidos en la lista de permitidos en nodos (nodo de macOS o host node sin interfaz). Esto usa
`skills.bins` a través del RPC del Gateway para obtener la lista de binarios de Skills. Desactívelo si quiere listas de permitidos manuales estrictas.

Notas importantes sobre confianza:

- Esta es una **lista de permitidos implícita de conveniencia**, separada de las entradas manuales de rutas en la lista de permitidos.
- Está pensada para entornos de operadores de confianza donde Gateway y node comparten el mismo límite de confianza.
- Si necesita una confianza estricta y explícita, mantenga `autoAllowSkills: false` y use solo entradas manuales de rutas en la lista de permitidos.

## Bins seguros (solo stdin)

`tools.exec.safeBins` define una pequeña lista de binarios **solo stdin** (por ejemplo `cut`)
que pueden ejecutarse en modo de lista de permitidos **sin** entradas explícitas en la lista de permitidos. Los bins seguros rechazan
argumentos posicionales de archivo y tokens similares a rutas, por lo que solo pueden operar sobre el flujo entrante.
Trate esto como una ruta rápida y limitada para filtros de flujo, no como una lista general de confianza.
**No** añada binarios de intérprete o runtime (por ejemplo `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) a `safeBins`.
Si un comando puede evaluar código, ejecutar subcomandos o leer archivos por diseño, prefiera entradas explícitas en la lista de permitidos y mantenga habilitados los prompts de aprobación.
Los bins seguros personalizados deben definir un perfil explícito en `tools.exec.safeBinProfiles.<bin>`.
La validación es determinista solo a partir de la forma de `argv` (sin comprobaciones de existencia en el sistema de archivos del host), lo que
evita el comportamiento de oráculo de existencia de archivos a partir de diferencias entre permitir/denegar.
Las opciones orientadas a archivos se deniegan para los bins seguros predeterminados (por ejemplo `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
Los bins seguros también aplican una política explícita por binario para flags que rompen el comportamiento
solo stdin (por ejemplo `sort -o/--output/--compress-program` y flags recursivos de grep).
Las opciones largas se validan en modo fail-closed en modo de bins seguros: las flags desconocidas y las
abreviaturas ambiguas se rechazan.
Flags denegadas por perfil de safe-bin:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Los bins seguros también fuerzan que los tokens de `argv` se traten como **texto literal** en el momento de la ejecución (sin globbing
ni expansión de `$VARS`) para segmentos solo stdin, de modo que patrones como `*` o `$HOME/...` no puedan
usarse para introducir lecturas de archivos.
Los bins seguros también deben resolverse desde directorios de binarios de confianza (valores predeterminados del sistema más
`tools.exec.safeBinTrustedDirs` opcional). Las entradas de `PATH` nunca se consideran automáticamente de confianza.
Los directorios predeterminados de confianza para safe-bin son intencionalmente mínimos: `/bin`, `/usr/bin`.
Si su ejecutable safe-bin vive en rutas de gestor de paquetes/usuario (por ejemplo
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), agréguelas explícitamente
a `tools.exec.safeBinTrustedDirs`.
Las cadenas de shell y las redirecciones no se permiten automáticamente en modo de lista de permitidos.

Las cadenas de shell (`&&`, `||`, `;`) se permiten cuando cada segmento de nivel superior satisface la lista de permitidos
(incluidos safe bins o la auto-permisión de Skills). Las redirecciones siguen sin ser compatibles en modo de lista de permitidos.
La sustitución de comandos (`$()` / comillas invertidas) se rechaza durante el análisis de lista de permitidos, incluso dentro de
comillas dobles; use comillas simples si necesita texto literal `$()`.
En las aprobaciones de la app complementaria de macOS, el texto shell sin procesar que contiene sintaxis de control o expansión del shell
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) se trata como un fallo de coincidencia en la lista de permitidos, a menos
que el propio binario del shell esté incluido en la lista de permitidos.
Para wrappers de shell (`bash|sh|zsh ... -c/-lc`), las sobrescrituras de entorno con alcance de solicitud se reducen a una
pequeña lista explícita de permitidos (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Para decisiones `allow-always` en modo de lista de permitidos, los wrappers de despacho conocidos
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservan las rutas del ejecutable interno en lugar de las
rutas del wrapper. Los multiplexores de shell (`busybox`, `toybox`) también se desempaquetan para applets de shell (`sh`, `ash`,
etc.), de modo que se conserven ejecutables internos en lugar de binarios multiplexores. Si un wrapper o
multiplexor no puede desempaquetarse de forma segura, no se conserva automáticamente ninguna entrada en la lista de permitidos.
Si incluye en la lista de permitidos intérpretes como `python3` o `node`, prefiera `tools.exec.strictInlineEval=true` para que la evaluación en línea siga requiriendo aprobación explícita. En modo estricto, `allow-always` puede seguir conservando invocaciones benignas de intérprete/script, pero los portadores de evaluación en línea no se conservan automáticamente.

Bins seguros predeterminados:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` y `sort` no están en la lista predeterminada. Si decide habilitarlos, mantenga entradas explícitas en la lista de permitidos para
sus flujos de trabajo que no sean solo stdin.
Para `grep` en modo safe-bin, proporcione el patrón con `-e`/`--regexp`; la forma de patrón posicional se
rechaza para que no puedan introducirse operandos de archivo como posicionales ambiguos.

### Bins seguros frente a lista de permitidos

| Tema            | `tools.exec.safeBins`                                  | Lista de permitidos (`exec-approvals.json`)                  |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Objetivo         | Permitir automáticamente filtros limitados de stdin    | Confiar explícitamente en ejecutables específicos            |
| Tipo de coincidencia | Nombre del ejecutable + política `argv` de safe-bin | Patrón glob de ruta del ejecutable resuelto                  |
| Alcance de argumentos | Restringido por el perfil safe-bin y reglas de tokens literales | Solo coincidencia de ruta; los argumentos son, por lo demás, su responsabilidad |
| Ejemplos típicos | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, CLIs personalizados       |
| Mejor uso        | Transformaciones de texto de bajo riesgo en pipelines  | Cualquier herramienta con comportamiento o efectos secundarios más amplios |

Ubicación de la configuración:

- `safeBins` proviene de la configuración (`tools.exec.safeBins` o por agente `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` proviene de la configuración (`tools.exec.safeBinTrustedDirs` o por agente `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` proviene de la configuración (`tools.exec.safeBinProfiles` o por agente `agents.list[].tools.exec.safeBinProfiles`). Las claves de perfiles por agente sobrescriben las claves globales.
- las entradas de la lista de permitidos viven en el archivo local del host `~/.openclaw/exec-approvals.json` bajo `agents.<id>.allowlist` (o mediante Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` advierte con `tools.exec.safe_bins_interpreter_unprofiled` cuando aparecen bins de intérprete/runtime en `safeBins` sin perfiles explícitos.
- `openclaw doctor --fix` puede generar entradas faltantes `safeBinProfiles.<bin>` como `{}` (revise y endurezca después). Los bins de intérprete/runtime no se generan automáticamente.

Ejemplo de perfil personalizado:
__OC_I18N_900004__
Si decide incluir explícitamente `jq` en `safeBins`, OpenClaw sigue rechazando el builtin `env` en modo safe-bin
para que `jq -n env` no pueda volcar el entorno del proceso del host sin una ruta explícita en la lista de permitidos
o un prompt de aprobación.

## Edición en Control UI

Use la tarjeta **Control UI → Nodes → Exec approvals** para editar valores predeterminados, sobrescrituras
por agente y listas de permitidos. Elija un alcance (Valores predeterminados o un agente), ajuste la política,
agregue/elimine patrones de la lista de permitidos y luego haga clic en **Save**. La UI muestra metadatos de **último uso**
por patrón para que pueda mantener limpia la lista.

El selector de destino elige **Gateway** (aprobaciones locales) o un **Node**. Los nodos
deben anunciar `system.execApprovals.get/set` (app de macOS o host node sin interfaz).
Si un nodo aún no anuncia aprobaciones de exec, edite directamente su
`~/.openclaw/exec-approvals.json` local.

CLI: `openclaw approvals` admite edición de gateway o node (consulte [CLI de aprobaciones](/cli/approvals)).

## Flujo de aprobación

Cuando se requiere un prompt, el gateway difunde `exec.approval.requested` a los clientes operadores.
Control UI y la app de macOS lo resuelven mediante `exec.approval.resolve`, luego el gateway reenvía la
solicitud aprobada al host del nodo.

Para `host=node`, las solicitudes de aprobación incluyen una carga útil canónica `systemRunPlan`. El gateway usa
ese plan como contexto autoritativo de comando/cwd/sesión al reenviar solicitudes aprobadas de `system.run`.

Esto importa para la latencia de aprobación asíncrona:

- la ruta de exec del nodo prepara un plan canónico por adelantado
- el registro de aprobación almacena ese plan y sus metadatos de vinculación
- una vez aprobado, la llamada final reenviada de `system.run` reutiliza el plan almacenado
  en lugar de confiar en ediciones posteriores de la persona que llama
- si la persona que llama cambia `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` después de que se creó la solicitud de aprobación, el gateway rechaza la
  ejecución reenviada por discrepancia de aprobación

## Comandos de intérprete/runtime

Las ejecuciones de intérprete/runtime respaldadas por aprobación son intencionalmente conservadoras:

- El contexto exacto de `argv`/`cwd`/`env` siempre queda vinculado.
- Las formas de script de shell directo y de archivo directo de runtime se vinculan, en la medida de lo posible, a una instantánea concreta de un archivo local.
- Las formas comunes de wrappers de gestores de paquetes que aun así se resuelven a un único archivo local directo (por ejemplo
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) se desempaquetan antes de la vinculación.
- Si OpenClaw no puede identificar exactamente un archivo local concreto para un comando de intérprete/runtime
  (por ejemplo scripts de paquetes, formas eval, cadenas de cargadores específicas del runtime o formas ambiguas de varios archivos),
  la ejecución respaldada por aprobación se deniega en lugar de afirmar una cobertura semántica que no
  tiene.
- Para esos flujos de trabajo, prefiera sandboxing, un límite de host separado o un flujo explícito de
  lista de permitidos/full donde el operador acepte la semántica más amplia del runtime.

Cuando se requieren aprobaciones, la herramienta exec devuelve inmediatamente un id de aprobación. Use ese id para
correlacionar eventos posteriores del sistema (`Exec finished` / `Exec denied`). Si no llega ninguna decisión antes del
timeout, la solicitud se trata como un timeout de aprobación y se muestra como motivo de denegación.

### Comportamiento de entrega de seguimiento

Después de que termina una ejecución asíncrona aprobada, OpenClaw envía un turno de seguimiento de `agent` a la misma sesión.

- Si existe un destino de entrega externo válido (canal entregable más objetivo `to`), la entrega de seguimiento usa ese canal.
- En flujos solo de webchat o de sesión interna sin destino externo, la entrega de seguimiento permanece solo en sesión (`deliver: false`).
- Si una persona que llama solicita explícitamente una entrega externa estricta sin un canal externo resoluble, la solicitud falla con `INVALID_REQUEST`.
- Si `bestEffortDeliver` está habilitado y no se puede resolver ningún canal externo, la entrega se degrada a solo sesión en lugar de fallar.

El diálogo de confirmación incluye:

- comando + argumentos
- cwd
- id del agente
- ruta resuelta del ejecutable
- host + metadatos de política

Acciones:

- **Permitir una vez** → ejecutar ahora
- **Permitir siempre** → agregar a la lista de permitidos + ejecutar
- **Denegar** → bloquear

## Reenvío de aprobaciones a canales de chat

Puede reenviar prompts de aprobación de exec a cualquier canal de chat (incluidos canales de plugin) y aprobarlos
con `/approve`. Esto usa el flujo normal de entrega saliente.

Configuración:
__OC_I18N_900005__
Responder en el chat:
__OC_I18N_900006__
El comando `/approve` gestiona tanto aprobaciones de exec como aprobaciones de plugin. Si el id no coincide con una aprobación de exec pendiente, comprueba automáticamente las aprobaciones de plugin en su lugar.

### Reenvío de aprobaciones de plugins

El reenvío de aprobaciones de plugins usa el mismo flujo de entrega que las aprobaciones de exec, pero tiene su
propia configuración independiente en `approvals.plugin`. Habilitar o deshabilitar una no afecta a la otra.
__OC_I18N_900007__
La forma de configuración es idéntica a `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` y `targets` funcionan del mismo modo.

Los canales que admiten respuestas interactivas compartidas muestran los mismos botones de aprobación tanto para exec como para
plugins. Los canales sin UI interactiva compartida recurren a texto sin formato con instrucciones `/approve`.

### Aprobaciones en el mismo chat en cualquier canal

Cuando una solicitud de aprobación de exec o plugin se origina desde una superficie de chat entregable, ese mismo chat
puede aprobarla con `/approve` de forma predeterminada. Esto se aplica a canales como Slack, Matrix y
Microsoft Teams, además de los flujos ya existentes de Web UI y terminal UI.

Esta ruta compartida de comandos de texto usa el modelo normal de autenticación del canal para esa conversación. Si el
chat de origen ya puede enviar comandos y recibir respuestas, las solicitudes de aprobación ya no necesitan un
adaptador de entrega nativo independiente solo para seguir pendientes.

Discord y Telegram también admiten `/approve` en el mismo chat, pero esos canales siguen usando su
lista de aprobadores resuelta para la autorización, incluso cuando la entrega nativa de aprobaciones está deshabilitada.

Para Telegram y otros clientes nativos de aprobación que llaman directamente al Gateway,
esta alternativa está intencionalmente limitada a fallos de tipo "approval not found". Una denegación o error real
de aprobación de exec no vuelve a intentarlo silenciosamente como aprobación de plugin.

### Entrega nativa de aprobaciones

Algunos canales también pueden actuar como clientes nativos de aprobación. Los clientes nativos añaden DM a aprobadores, fanout al chat de origen
y una UX interactiva de aprobación específica del canal, además del flujo compartido de `/approve`
en el mismo chat.

Cuando hay disponibles tarjetas/botones nativos de aprobación, esa UI nativa es la
ruta principal orientada al agente. El agente tampoco debería mostrar un comando de chat plano duplicado
`/approve` a menos que el resultado de la herramienta diga que las aprobaciones por chat no están disponibles o
que la aprobación manual sea la única ruta restante.

Modelo genérico:

- la política de exec del host sigue decidiendo si se requiere aprobación de exec
- `approvals.exec` controla el reenvío de prompts de aprobación a otros destinos de chat
- `channels.<channel>.execApprovals` controla si ese canal actúa como cliente nativo de aprobación

Los clientes nativos de aprobación habilitan automáticamente la entrega prioritaria por DM cuando se cumplen todas estas condiciones:

- el canal admite entrega nativa de aprobaciones
- los aprobadores pueden resolverse desde `execApprovals.approvers` explícito o desde las
  fuentes alternativas documentadas para ese canal
- `channels.<channel>.execApprovals.enabled` no está establecido o es `"auto"`

Configure `enabled: false` para deshabilitar explícitamente un cliente nativo de aprobación. Configure `enabled: true` para forzarlo
cuando los aprobadores se resuelvan. La entrega pública al chat de origen sigue siendo explícita mediante
`channels.<channel>.execApprovals.target`.

Preguntas frecuentes: [¿Por qué hay dos configuraciones de aprobación de exec para aprobaciones por chat?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`

Estos clientes nativos de aprobación añaden enrutamiento por DM y fanout opcional al canal encima del flujo compartido
de `/approve` en el mismo chat y de los botones compartidos de aprobación.

Comportamiento compartido:

- Slack, Matrix, Microsoft Teams y chats entregables similares usan el modelo normal de autenticación del canal
  para `/approve` en el mismo chat
- cuando un cliente nativo de aprobación se habilita automáticamente, el destino predeterminado de entrega nativa son los DM a aprobadores
- para Discord y Telegram, solo los aprobadores resueltos pueden aprobar o denegar
- los aprobadores de Discord pueden ser explícitos (`execApprovals.approvers`) o inferidos de `commands.ownerAllowFrom`
- los aprobadores de Telegram pueden ser explícitos (`execApprovals.approvers`) o inferidos de la configuración existente del propietario (`allowFrom`, más `defaultTo` de mensajes directos cuando se admite)
- los aprobadores de Slack pueden ser explícitos (`execApprovals.approvers`) o inferidos de `commands.ownerAllowFrom`
- los botones nativos de Slack conservan el tipo de id de aprobación, de modo que los id `plugin:` pueden resolver aprobaciones de plugins
  sin una segunda capa alternativa local de Slack
- el enrutamiento nativo de DM/canal de Matrix y los accesos directos de reacciones gestionan tanto aprobaciones de exec como de plugins;
  la autorización de plugins sigue proviniendo de `channels.matrix.dm.allowFrom`
- quien realiza la solicitud no necesita ser aprobador
- el chat de origen puede aprobar directamente con `/approve` cuando ese chat ya admite comandos y respuestas
- los botones nativos de aprobación de Discord enrutan según el tipo de id de aprobación: los id `plugin:` van
  directamente a aprobaciones de plugins, todo lo demás va a aprobaciones de exec
- los botones nativos de aprobación de Telegram siguen la misma alternativa limitada de exec a plugin que `/approve`
- cuando `target` nativo habilita la entrega al chat de origen, los prompts de aprobación incluyen el texto del comando
- las aprobaciones de exec pendientes vencen después de 30 minutos de forma predeterminada
- si ninguna UI de operador o cliente de aprobación configurado puede aceptar la solicitud, el prompt recurre a `askFallback`

Telegram usa de forma predeterminada los DM a aprobadores (`target: "dm"`). Puede cambiar a `channel` o `both` cuando
quiera que los prompts de aprobación aparezcan también en el chat/tema de Telegram de origen. Para temas de foros de Telegram,
OpenClaw conserva el tema para el prompt de aprobación y el seguimiento posterior a la aprobación.

Consulte:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### Flujo IPC de macOS
__OC_I18N_900008__
Notas de seguridad:

- Modo de socket Unix `0600`, token almacenado en `exec-approvals.json`.
- Comprobación de pares con el mismo UID.
- Desafío/respuesta (nonce + token HMAC + hash de solicitud) + TTL corto.

## Eventos del sistema

El ciclo de vida de exec aparece como mensajes del sistema:

- `Exec running` (solo si el comando supera el umbral de aviso de ejecución)
- `Exec finished`
- `Exec denied`

Estos se publican en la sesión del agente después de que el nodo informa del evento.
Las aprobaciones de exec en el host gateway emiten los mismos eventos de ciclo de vida cuando termina el comando (y, opcionalmente, cuando se ejecuta más tiempo que el umbral).
Los exec controlados por aprobación reutilizan el id de aprobación como `runId` en estos mensajes para facilitar la correlación.

## Comportamiento ante aprobación denegada

Cuando se deniega una aprobación asíncrona de exec, OpenClaw evita que el agente reutilice
la salida de cualquier ejecución anterior del mismo comando en la sesión. El motivo de la denegación
se transmite con orientación explícita de que no hay ninguna salida del comando disponible, lo que impide
que el agente afirme que existe una salida nueva o repita el comando denegado con
resultados obsoletos de una ejecución anterior satisfactoria.

## Implicaciones

- **full** es potente; prefiera listas de permitidos cuando sea posible.
- **ask** le mantiene informado y sigue permitiendo aprobaciones rápidas.
- Las listas de permitidos por agente evitan que las aprobaciones de un agente se filtren a otros.
- Las aprobaciones solo se aplican a solicitudes de exec del host de **remitentes autorizados**. Los remitentes no autorizados no pueden emitir `/exec`.
- `/exec security=full` es una comodidad a nivel de sesión para operadores autorizados y, por diseño, omite aprobaciones.
  Para bloquear por completo el exec del host, configure la seguridad de aprobaciones en `deny` o deniegue la herramienta `exec` mediante política de herramientas.

Relacionado:

- [Herramienta Exec](/es/tools/exec)
- [Modo elevated](/es/tools/elevated)
- [Skills](/es/tools/skills)

## Relacionado

- [Exec](/es/tools/exec) — herramienta de ejecución de comandos de shell
- [Sandboxing](/es/gateway/sandboxing) — modos de sandbox y acceso al espacio de trabajo
- [Security](/es/gateway/security) — modelo de seguridad y endurecimiento
- [Sandbox vs Tool Policy vs Elevated](/es/gateway/sandbox-vs-tool-policy-vs-elevated) — cuándo usar cada uno
