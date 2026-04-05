---
read_when:
    - Configurar aprobaciones de exec o listas de permitidos
    - Implementar la UX de aprobación de exec en la app de macOS
    - Revisar los prompts de escape del sandbox y sus implicaciones
summary: Aprobaciones de Exec, listas de permitidos y prompts de escape del sandbox
title: Aprobaciones de Exec
x-i18n:
    generated_at: "2026-04-05T12:56:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: a1efa3b78efe3ca6246acfb37830b103ede40cc5298dcc7da8e9fbc5f6cc88ef
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Aprobaciones de Exec

Las aprobaciones de Exec son la **barandilla de seguridad de la app complementaria / host de nodo** para permitir que un agente en sandbox ejecute
comandos en un host real (`gateway` o `node`). Piénsalo como un enclavamiento de seguridad:
los comandos solo se permiten cuando la política + la lista de permitidos + la aprobación opcional del usuario coinciden.
Las aprobaciones de Exec son **adicionales** a la política de herramientas y al control de modo elevado (a menos que elevated esté configurado en `full`, lo que omite las aprobaciones).
La política efectiva es la **más estricta** entre los valores predeterminados de `tools.exec.*` y los de aprobaciones; si se omite un campo de aprobaciones, se usa el valor de `tools.exec`.
El exec del host también usa el estado local de aprobaciones de esa máquina. Un valor local del host
`ask: "always"` en `~/.openclaw/exec-approvals.json` sigue mostrando prompts incluso si
la sesión o la configuración predeterminada solicitan `ask: "on-miss"`.
Usa `openclaw approvals get`, `openclaw approvals get --gateway` o
`openclaw approvals get --node <id|name|ip>` para inspeccionar la política solicitada,
las fuentes de política del host y el resultado efectivo.

Si la UI de la app complementaria **no está disponible**, cualquier solicitud que requiera un prompt se
resuelve mediante la **alternativa de ask** (predeterminado: denegar).

## Dónde se aplica

Las aprobaciones de Exec se aplican localmente en el host de ejecución:

- **host del gateway** → proceso `openclaw` en la máquina del gateway
- **host del nodo** → ejecutor del nodo (app complementaria de macOS o host de nodo sin interfaz)

Nota sobre el modelo de confianza:

- Las personas que llaman autenticadas en el gateway son operadores de confianza para ese Gateway.
- Los nodos emparejados amplían esa capacidad de operador de confianza al host del nodo.
- Las aprobaciones de Exec reducen el riesgo de ejecución accidental, pero no son un límite de autenticación por usuario.
- Las ejecuciones aprobadas en el host del nodo vinculan el contexto de ejecución canónico: `cwd` canónico, `argv` exacto, vinculación de `env`
  cuando está presente y ruta ejecutable fijada cuando corresponde.
- Para scripts de shell e invocaciones directas de archivos mediante intérprete/runtime, OpenClaw también intenta vincular
  un único operando de archivo local concreto. Si ese archivo vinculado cambia después de la aprobación pero antes de la ejecución,
  la ejecución se deniega en lugar de ejecutar contenido modificado.
- Esta vinculación de archivos es intencionalmente una aproximación de mejor esfuerzo, no un modelo semántico completo de todas las
  rutas de carga de intérpretes/runtime. Si el modo de aprobación no puede identificar exactamente un archivo local concreto
  para vincular, se niega a generar una ejecución respaldada por aprobación en lugar de fingir cobertura total.

División en macOS:

- El **servicio del host del nodo** reenvía `system.run` a la **app de macOS** mediante IPC local.
- La **app de macOS** aplica las aprobaciones y ejecuta el comando en el contexto de la UI.

## Configuración y almacenamiento

Las aprobaciones se almacenan en un archivo JSON local en el host de ejecución:

`~/.openclaw/exec-approvals.json`

Esquema de ejemplo:

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

Si quieres que el exec del host se ejecute sin prompts de aprobación, debes abrir **ambas** capas de política:

- política de exec solicitada en la configuración de OpenClaw (`tools.exec.*`)
- política local de aprobaciones del host en `~/.openclaw/exec-approvals.json`

Ahora este es el comportamiento predeterminado del host, a menos que lo endurezcas explícitamente:

- `tools.exec.security`: `full` en `gateway`/`node`
- `tools.exec.ask`: `off`
- host `askFallback`: `full`

Distinción importante:

- `tools.exec.host=auto` elige dónde se ejecuta exec: sandbox cuando está disponible, o gateway en caso contrario.
- YOLO elige cómo se aprueba el exec del host: `security=full` más `ask=off`.
- `auto` no convierte el enrutamiento al gateway en una anulación libre desde una sesión en sandbox. Se permite una solicitud por llamada con `host=node` desde `auto`, y `host=gateway` solo se permite desde `auto` cuando no hay un runtime de sandbox activo. Si quieres un valor predeterminado estable distinto de `auto`, configura `tools.exec.host` o usa `/exec host=...` explícitamente.

Si quieres una configuración más conservadora, vuelve a endurecer cualquiera de las dos capas a `allowlist` / `on-miss`
o `deny`.

Configuración persistente de "nunca mostrar prompt" en el host del gateway:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Luego haz que el archivo de aprobaciones del host coincida:

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

Para un host de nodo, aplica en su lugar el mismo archivo de aprobaciones en ese nodo:

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

Atajo solo para la sesión:

- `/exec security=full ask=off` cambia solo la sesión actual.
- `/elevated full` es un atajo de emergencia que también omite las aprobaciones de exec para esa sesión.

Si el archivo de aprobaciones del host sigue siendo más estricto que la configuración, la política más estricta del host sigue prevaleciendo.

## Controles de política

### Seguridad (`exec.security`)

- **deny**: bloquea todas las solicitudes de exec del host.
- **allowlist**: permite solo comandos incluidos en la lista de permitidos.
- **full**: permite todo (equivalente a elevated).

### Ask (`exec.ask`)

- **off**: nunca muestra prompts.
- **on-miss**: muestra prompt solo cuando no hay coincidencia en la lista de permitidos.
- **always**: muestra prompt en cada comando.
- La confianza duradera `allow-always` no suprime prompts cuando el modo ask efectivo es `always`

### Alternativa de ask (`askFallback`)

Si se requiere un prompt pero no hay ninguna UI accesible, la alternativa decide:

- **deny**: bloquear.
- **allowlist**: permitir solo si hay coincidencia en la lista de permitidos.
- **full**: permitir.

### Endurecimiento de evaluación inline del intérprete (`tools.exec.strictInlineEval`)

Cuando `tools.exec.strictInlineEval=true`, OpenClaw trata las formas de evaluación de código inline como de aprobación obligatoria, incluso si el binario del intérprete en sí está en la lista de permitidos.

Ejemplos:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Esto es defensa en profundidad para cargadores de intérprete que no se asignan limpiamente a un único operando de archivo estable. En modo estricto:

- estos comandos siguen necesitando aprobación explícita;
- `allow-always` no conserva automáticamente nuevas entradas de lista de permitidos para ellos.

## Lista de permitidos (por agente)

Las listas de permitidos son **por agente**. Si existen varios agentes, cambia el agente que estás
editando en la app de macOS. Los patrones son **coincidencias glob sin distinción entre mayúsculas y minúsculas**.
Los patrones deben resolverse a **rutas binarias** (las entradas solo con nombre base se ignoran).
Las entradas heredadas `agents.default` se migran a `agents.main` al cargar.
Las cadenas de shell como `echo ok && pwd` siguen necesitando que cada segmento de nivel superior cumpla las reglas de la lista de permitidos.

Ejemplos:

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Cada entrada de la lista de permitidos registra:

- **id** UUID estable usado para la identidad en la UI (opcional)
- **último uso** marca temporal
- **último comando usado**
- **última ruta resuelta**

## Permitir automáticamente CLIs de Skills

Cuando **Permitir automáticamente CLIs de Skills** está habilitado, los ejecutables referenciados por Skills conocidas
se tratan como incluidos en la lista de permitidos en nodos (nodo de macOS o host de nodo sin interfaz). Esto usa
`skills.bins` a través del RPC del Gateway para obtener la lista de binarios de Skills. Desactívalo si quieres listas de permitidos manuales estrictas.

Notas importantes sobre la confianza:

- Esta es una **lista de permitidos implícita de conveniencia**, separada de las entradas manuales de lista de permitidos por ruta.
- Está pensada para entornos de operadores de confianza donde Gateway y nodo están dentro del mismo límite de confianza.
- Si necesitas confianza estrictamente explícita, mantén `autoAllowSkills: false` y usa solo entradas manuales de lista de permitidos por ruta.

## Bins seguros (solo stdin)

`tools.exec.safeBins` define una pequeña lista de binarios **solo stdin** (por ejemplo `cut`)
que pueden ejecutarse en modo de lista de permitidos **sin** entradas explícitas en la lista de permitidos. Los bins seguros rechazan
argumentos posicionales de archivo y tokens parecidos a rutas, por lo que solo pueden operar sobre el flujo entrante.
Considera esto una vía rápida limitada para filtros de flujo, no una lista general de confianza.
**No** añadas bins de intérprete o runtime (por ejemplo `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) a `safeBins`.
Si un comando puede evaluar código, ejecutar subcomandos o leer archivos por diseño, prefiere entradas explícitas de lista de permitidos y mantén habilitados los prompts de aprobación.
Los bins seguros personalizados deben definir un perfil explícito en `tools.exec.safeBinProfiles.<bin>`.
La validación es determinista solo a partir de la forma de `argv` (sin comprobaciones de existencia en el sistema de archivos del host), lo cual
evita el comportamiento de oráculo de existencia de archivos derivado de las diferencias entre permitir/denegar.
Las opciones orientadas a archivos se deniegan para los bins seguros predeterminados (por ejemplo `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
Los bins seguros también aplican una política explícita de flags por binario para opciones que rompen el
comportamiento de solo stdin (por ejemplo `sort -o/--output/--compress-program` y flags recursivos de grep).
Las opciones largas se validan en modo fail-closed en el modo safe-bin: se rechazan los flags desconocidos y las abreviaturas ambiguas.
Flags denegados por perfil de safe-bin:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Los bins seguros también fuerzan que los tokens de `argv` se traten como **texto literal** en tiempo de ejecución (sin expansión de glob
ni de `$VARS`) para segmentos solo stdin, por lo que patrones como `*` o `$HOME/...` no pueden
usarse para introducir lecturas de archivos.
Los bins seguros también deben resolverse desde directorios binarios de confianza (valores del sistema por defecto más
`tools.exec.safeBinTrustedDirs` opcionales). Las entradas de `PATH` nunca se consideran de confianza automáticamente.
Los directorios predeterminados de confianza para safe-bin son intencionalmente mínimos: `/bin`, `/usr/bin`.
Si tu ejecutable safe-bin está en rutas de gestor de paquetes/usuario (por ejemplo
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), añádelas explícitamente
a `tools.exec.safeBinTrustedDirs`.
Las cadenas de shell y redirecciones no se permiten automáticamente en modo de lista de permitidos.

Las cadenas de shell (`&&`, `||`, `;`) se permiten cuando cada segmento de nivel superior cumple la lista de permitidos
(incluidos bins seguros o la auto-permisión de Skills). Las redirecciones siguen sin estar admitidas en modo de lista de permitidos.
La sustitución de comandos (`$()` / comillas invertidas) se rechaza durante el análisis de la lista de permitidos, incluso dentro de
comillas dobles; usa comillas simples si necesitas texto literal `$()`.
En las aprobaciones de la app complementaria de macOS, el texto shell en bruto que contenga sintaxis de control o expansión del shell
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) se trata como un fallo de coincidencia de la lista de permitidos a menos que
el propio binario del shell esté en la lista de permitidos.
Para wrappers de shell (`bash|sh|zsh ... -c/-lc`), las anulaciones de `env` con alcance de solicitud se reducen a una
lista pequeña y explícita (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Para decisiones `allow-always` en modo de lista de permitidos, los wrappers de despacho conocidos
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservan las rutas de ejecutables internas en lugar de las
rutas de los wrappers. Los multiplexores de shell (`busybox`, `toybox`) también se desempaquetan para applets de shell (`sh`, `ash`,
etc.) de modo que se conservan los ejecutables internos en lugar de los binarios multiplexores. Si un wrapper o
multiplexor no puede desempaquetarse con seguridad, no se conserva ninguna entrada en la lista de permitidos automáticamente.
Si incluyes intérpretes como `python3` o `node` en la lista de permitidos, prefiere `tools.exec.strictInlineEval=true` para que la evaluación inline siga requiriendo aprobación explícita. En modo estricto, `allow-always` aún puede conservar invocaciones benignas de intérprete/script, pero los vehículos de evaluación inline no se conservan automáticamente.

Bins seguros predeterminados:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` y `sort` no están en la lista predeterminada. Si los activas explícitamente, mantén entradas explícitas en la lista de permitidos para
sus flujos de trabajo que no son solo stdin.
Para `grep` en modo safe-bin, proporciona el patrón con `-e`/`--regexp`; la forma de patrón posicional se
rechaza para que no se puedan introducir operandos de archivo como posicionales ambiguos.

### Bins seguros frente a lista de permitidos

| Topic            | `tools.exec.safeBins`                                  | Allowlist (`exec-approvals.json`)                            |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Goal             | Permitir automáticamente filtros limitados de stdin    | Confiar explícitamente en ejecutables específicos            |
| Match type       | Nombre de ejecutable + política `argv` de safe-bin     | Patrón glob de ruta ejecutable resuelta                      |
| Argument scope   | Restringido por el perfil de safe-bin y reglas de tokens literales | Solo coincidencia de ruta; los argumentos son, por lo demás, tu responsabilidad |
| Typical examples | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, CLIs personalizadas       |
| Best use         | Transformaciones de texto de bajo riesgo en pipelines  | Cualquier herramienta con comportamiento más amplio o efectos secundarios |

Ubicación de la configuración:

- `safeBins` proviene de la configuración (`tools.exec.safeBins` o por agente en `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` proviene de la configuración (`tools.exec.safeBinTrustedDirs` o por agente en `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` proviene de la configuración (`tools.exec.safeBinProfiles` o por agente en `agents.list[].tools.exec.safeBinProfiles`). Las claves de perfil por agente prevalecen sobre las globales.
- Las entradas de la lista de permitidos viven en el archivo local del host `~/.openclaw/exec-approvals.json` en `agents.<id>.allowlist` (o mediante la UI de control / `openclaw approvals allowlist ...`).
- `openclaw security audit` avisa con `tools.exec.safe_bins_interpreter_unprofiled` cuando aparecen bins de intérprete/runtime en `safeBins` sin perfiles explícitos.
- `openclaw doctor --fix` puede crear de forma asistida las entradas personalizadas faltantes `safeBinProfiles.<bin>` como `{}` (revisa y endurece después). Los bins de intérprete/runtime no se crean automáticamente.

Ejemplo de perfil personalizado:
__OC_I18N_900004__
Si activas explícitamente `jq` en `safeBins`, OpenClaw sigue rechazando el builtin `env` en modo safe-bin
para que `jq -n env` no pueda volcar el entorno del proceso del host sin una ruta explícita en la lista de permitidos
o un prompt de aprobación.

## Edición en la UI de control

Usa la tarjeta **UI de control → Nodos → Aprobaciones de Exec** para editar valores predeterminados, anulaciones
por agente y listas de permitidos. Elige un alcance (Predeterminados o un agente), ajusta la política,
añade/elimina patrones de la lista de permitidos y luego **Guardar**. La UI muestra metadatos de **último uso**
por patrón para ayudarte a mantener la lista ordenada.

El selector de destino elige **Gateway** (aprobaciones locales) o un **Nodo**. Los nodos
deben anunciar `system.execApprovals.get/set` (app de macOS o host de nodo sin interfaz).
Si un nodo todavía no anuncia aprobaciones de exec, edita directamente su
`~/.openclaw/exec-approvals.json` local.

CLI: `openclaw approvals` admite edición de gateway o nodo (consulta [CLI de Approvals](/cli/approvals)).

## Flujo de aprobación

Cuando se requiere un prompt, el gateway difunde `exec.approval.requested` a los clientes operadores.
La UI de control y la app de macOS lo resuelven mediante `exec.approval.resolve`, y luego el gateway reenvía la
solicitud aprobada al host del nodo.

Para `host=node`, las solicitudes de aprobación incluyen una carga útil canónica `systemRunPlan`. El gateway usa
ese plan como comando/contexto de `cwd`/sesión autoritativo al reenviar las solicitudes aprobadas de `system.run`.

Esto importa para la latencia de aprobación asíncrona:

- la ruta de exec del nodo prepara un único plan canónico por adelantado
- el registro de aprobación almacena ese plan y sus metadatos de vinculación
- una vez aprobado, la llamada final reenviada a `system.run` reutiliza el plan almacenado
  en lugar de confiar en ediciones posteriores de la persona que llama
- si la persona que llama cambia `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` después de crear la solicitud de aprobación, el gateway rechaza la
  ejecución reenviada por discrepancia de aprobación

## Comandos de intérprete/runtime

Las ejecuciones de intérprete/runtime respaldadas por aprobación son intencionalmente conservadoras:

- El contexto exacto de `argv`/`cwd`/`env` siempre se vincula.
- Las formas directas de script de shell y archivo de runtime se vinculan, en la medida de lo posible, a una instantánea concreta de un archivo local.
- Las formas comunes de wrappers de gestores de paquetes que aún se resuelven a un archivo local directo (por ejemplo
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) se desempaquetan antes de la vinculación.
- Si OpenClaw no puede identificar exactamente un archivo local concreto para un comando de intérprete/runtime
  (por ejemplo scripts de paquetes, formas eval, cadenas de carga específicas del runtime o formas ambiguas de varios archivos),
  la ejecución respaldada por aprobación se deniega en lugar de afirmar una cobertura semántica que no tiene.
- Para estos flujos de trabajo, prefiere el sandboxing, un límite de host independiente o un flujo explícito de
  lista de permitidos/full en el que el operador acepte la semántica más amplia del runtime.

Cuando se requieren aprobaciones, la herramienta exec devuelve de inmediato un id de aprobación. Usa ese id para
correlacionar eventos posteriores del sistema (`Exec finished` / `Exec denied`). Si no llega ninguna decisión antes del
tiempo de espera, la solicitud se trata como timeout de aprobación y se muestra como motivo de denegación.

### Comportamiento de entrega de seguimiento

Después de que finalice un exec asíncrono aprobado, OpenClaw envía un turno `agent` de seguimiento a la misma sesión.

- Si existe un destino externo de entrega válido (canal entregable más `to` de destino), la entrega de seguimiento usa ese canal.
- En flujos solo de webchat o de sesión interna sin destino externo, la entrega de seguimiento permanece solo en la sesión (`deliver: false`).
- Si una persona que llama solicita explícitamente entrega externa estricta sin un canal externo resoluble, la solicitud falla con `INVALID_REQUEST`.
- Si `bestEffortDeliver` está habilitado y no puede resolverse ningún canal externo, la entrega se degrada a solo sesión en lugar de fallar.

El cuadro de diálogo de confirmación incluye:

- comando + argumentos
- `cwd`
- id del agente
- ruta resuelta del ejecutable
- host + metadatos de política

Acciones:

- **Allow once** → ejecutar ahora
- **Always allow** → añadir a la lista de permitidos + ejecutar
- **Deny** → bloquear

## Reenvío de aprobaciones a canales de chat

Puedes reenviar prompts de aprobación de exec a cualquier canal de chat (incluidos canales de plugins) y aprobarlos
con `/approve`. Esto usa el pipeline normal de entrega saliente.

Configuración:
__OC_I18N_900005__
Responder en el chat:
__OC_I18N_900006__
El comando `/approve` gestiona tanto aprobaciones de exec como de plugins. Si el ID no coincide con una aprobación de exec pendiente, comprueba automáticamente las aprobaciones de plugins.

### Reenvío de aprobaciones de plugins

El reenvío de aprobaciones de plugins usa el mismo pipeline de entrega que las aprobaciones de exec, pero tiene su propia
configuración independiente en `approvals.plugin`. Activar o desactivar una no afecta a la otra.
__OC_I18N_900007__
La forma de la configuración es idéntica a `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` y `targets` funcionan igual.

Los canales que admiten respuestas interactivas compartidas muestran los mismos botones de aprobación para aprobaciones de exec y de plugins. Los canales sin UI interactiva compartida recurren a texto sin formato con instrucciones de `/approve`.

### Aprobaciones en el mismo chat en cualquier canal

Cuando una solicitud de aprobación de exec o de plugin se origina en una superficie de chat entregable, ese mismo chat
ahora puede aprobarla con `/approve` de forma predeterminada. Esto se aplica a canales como Slack, Matrix y
Microsoft Teams, además de los flujos existentes de UI web y UI de terminal.

Esta ruta compartida basada en comandos de texto usa el modelo normal de autenticación del canal para esa conversación. Si el
chat de origen ya puede enviar comandos y recibir respuestas, las solicitudes de aprobación ya no necesitan un adaptador
nativo de entrega independiente solo para permanecer pendientes.

Discord y Telegram también admiten `/approve` en el mismo chat, pero esos canales siguen usando su
lista resuelta de aprobadores para la autorización incluso cuando la entrega nativa de aprobaciones está deshabilitada.

Para Telegram y otros clientes nativos de aprobación que llaman directamente al Gateway,
esta alternativa está intencionalmente limitada a errores de tipo "approval not found". Una denegación/error real
de aprobación de exec no vuelve a intentarse silenciosamente como aprobación de plugin.

### Entrega nativa de aprobaciones

Algunos canales también pueden actuar como clientes nativos de aprobación. Los clientes nativos añaden MD a aprobadores, difusión al chat de origen
y UX de aprobación interactiva específica del canal además del flujo compartido de `/approve` en el mismo chat.

Cuando las tarjetas/botones nativos de aprobación están disponibles, esa UI nativa es la ruta principal
orientada al agente. El agente no debería repetir además un comando simple duplicado de chat
`/approve` a menos que el resultado de la herramienta diga que las aprobaciones por chat no están disponibles o que la aprobación manual sea la única ruta restante.

Modelo genérico:

- la política de exec del host sigue decidiendo si se requiere aprobación de exec
- `approvals.exec` controla el reenvío de prompts de aprobación a otros destinos de chat
- `channels.<channel>.execApprovals` controla si ese canal actúa como cliente nativo de aprobación

Los clientes nativos de aprobación activan automáticamente la entrega prioritaria por MD cuando se cumplen todas estas condiciones:

- el canal admite entrega nativa de aprobaciones
- los aprobadores pueden resolverse a partir de `execApprovals.approvers` explícito o de las fuentes de alternativa documentadas de ese canal
- `channels.<channel>.execApprovals.enabled` no está configurado o es `"auto"`

Configura `enabled: false` para desactivar explícitamente un cliente nativo de aprobación. Configura `enabled: true` para forzarlo
cuando los aprobadores se resuelvan. La entrega pública al chat de origen sigue siendo explícita mediante
`channels.<channel>.execApprovals.target`.

FAQ: [¿Por qué hay dos configuraciones de aprobación de exec para aprobaciones por chat?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`

Estos clientes nativos de aprobación añaden enrutamiento por MD y difusión opcional al canal además del flujo compartido
de `/approve` en el mismo chat y de los botones de aprobación compartidos.

Comportamiento compartido:

- Slack, Matrix, Microsoft Teams y chats entregables similares usan el modelo normal de autenticación del canal
  para `/approve` en el mismo chat
- cuando un cliente nativo de aprobación se activa automáticamente, el destino nativo predeterminado de entrega son las MD de los aprobadores
- para Discord y Telegram, solo los aprobadores resueltos pueden aprobar o denegar
- los aprobadores de Discord pueden ser explícitos (`execApprovals.approvers`) o inferirse de `commands.ownerAllowFrom`
- los aprobadores de Telegram pueden ser explícitos (`execApprovals.approvers`) o inferirse de la configuración existente del propietario (`allowFrom`, más `defaultTo` de mensajes directos cuando esté admitido)
- los aprobadores de Slack pueden ser explícitos (`execApprovals.approvers`) o inferirse de `commands.ownerAllowFrom`
- los botones nativos de Slack preservan el tipo de id de aprobación, de modo que los ids `plugin:` pueden resolver aprobaciones de plugins
  sin una segunda capa alternativa local de Slack
- el enrutamiento nativo de MD/canal de Matrix es solo para exec; las aprobaciones de plugins de Matrix permanecen en el
  flujo compartido de `/approve` en el mismo chat y en las rutas opcionales de reenvío `approvals.plugin`
- la persona que solicita no necesita ser aprobadora
- el chat de origen puede aprobar directamente con `/approve` cuando ese chat ya admite comandos y respuestas
- los botones nativos de aprobación de Discord enrutan según el tipo de id de aprobación: los ids `plugin:` van
  directamente a aprobaciones de plugins; todo lo demás va a aprobaciones de exec
- los botones nativos de aprobación de Telegram siguen la misma alternativa limitada de exec a plugin que `/approve`
- cuando `target` nativo habilita la entrega al chat de origen, los prompts de aprobación incluyen el texto del comando
- las aprobaciones de exec pendientes caducan tras 30 minutos de forma predeterminada
- si ninguna UI de operador o cliente de aprobación configurado puede aceptar la solicitud, el prompt recurre a `askFallback`

Telegram usa por defecto las MD de los aprobadores (`target: "dm"`). Puedes cambiar a `channel` o `both` cuando
quieras que los prompts de aprobación aparezcan también en el chat/tema de Telegram de origen. En temas de foros de Telegram, OpenClaw conserva el tema tanto para el prompt de aprobación como para el seguimiento posterior a la aprobación.

Consulta:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### Flujo IPC de macOS
__OC_I18N_900008__
Notas de seguridad:

- Modo de socket Unix `0600`, token almacenado en `exec-approvals.json`.
- Comprobación de par con mismo UID.
- Desafío/respuesta (nonce + token HMAC + hash de solicitud) + TTL corto.

## Eventos del sistema

El ciclo de vida de Exec se expone como mensajes del sistema:

- `Exec running` (solo si el comando supera el umbral de aviso de ejecución)
- `Exec finished`
- `Exec denied`

Estos se publican en la sesión del agente después de que el nodo informe del evento.
Las aprobaciones de exec en el host del gateway emiten los mismos eventos del ciclo de vida cuando el comando termina (y opcionalmente cuando se ejecuta más tiempo que el umbral).
Los execs controlados por aprobación reutilizan el id de aprobación como `runId` en estos mensajes para facilitar la correlación.

## Comportamiento ante aprobación denegada

Cuando se deniega una aprobación de exec asíncrono, OpenClaw impide que el agente reutilice
la salida de cualquier ejecución anterior del mismo comando en la sesión. El motivo de la denegación
se transmite con orientación explícita de que no hay salida del comando disponible, lo que evita
que el agente afirme que hay una nueva salida o repita el comando denegado con
resultados obsoletos de una ejecución exitosa anterior.

## Implicaciones

- **full** es potente; prefiere listas de permitidos cuando sea posible.
- **ask** te mantiene al tanto y aun así permite aprobaciones rápidas.
- Las listas de permitidos por agente evitan que las aprobaciones de un agente se filtren a otros.
- Las aprobaciones solo se aplican a solicitudes de exec del host de **remitentes autorizados**. Los remitentes no autorizados no pueden emitir `/exec`.
- `/exec security=full` es una comodidad a nivel de sesión para operadores autorizados y omite las aprobaciones por diseño.
  Para bloquear por completo el exec del host, configura la seguridad de aprobaciones en `deny` o deniega la herramienta `exec` mediante la política de herramientas.

Relacionado:

- [Herramienta Exec](/tools/exec)
- [Modo elevado](/tools/elevated)
- [Skills](/tools/skills)

## Relacionado

- [Exec](/tools/exec) — herramienta de ejecución de comandos de shell
- [Sandboxing](/es/gateway/sandboxing) — modos de sandbox y acceso al espacio de trabajo
- [Security](/es/gateway/security) — modelo de seguridad y endurecimiento
- [Sandbox vs Tool Policy vs Elevated](/es/gateway/sandbox-vs-tool-policy-vs-elevated) — cuándo usar cada uno
