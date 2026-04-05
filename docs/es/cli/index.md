---
read_when:
    - Agregar o modificar comandos u opciones de la CLI
    - Documentar nuevas superficies de comandos
summary: Referencia de la CLI de OpenClaw para comandos, subcomandos y opciones de `openclaw`
title: Referencia de la CLI
x-i18n:
    generated_at: "2026-04-05T12:40:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7c25e5ebfe256412b44130dba39cf39b0a7d1d22e3abb417345e95c95ca139bf
    source_path: cli/index.md
    workflow: 15
---

# Referencia de la CLI

Esta página describe el comportamiento actual de la CLI. Si los comandos cambian, actualiza este documento.

## Páginas de comandos

- [`setup`](/cli/setup)
- [`onboard`](/cli/onboard)
- [`configure`](/cli/configure)
- [`config`](/cli/config)
- [`completion`](/cli/completion)
- [`doctor`](/cli/doctor)
- [`dashboard`](/cli/dashboard)
- [`backup`](/cli/backup)
- [`reset`](/cli/reset)
- [`uninstall`](/cli/uninstall)
- [`update`](/cli/update)
- [`message`](/cli/message)
- [`agent`](/cli/agent)
- [`agents`](/cli/agents)
- [`acp`](/cli/acp)
- [`mcp`](/cli/mcp)
- [`status`](/cli/status)
- [`health`](/cli/health)
- [`sessions`](/cli/sessions)
- [`gateway`](/cli/gateway)
- [`logs`](/cli/logs)
- [`system`](/cli/system)
- [`models`](/cli/models)
- [`memory`](/cli/memory)
- [`directory`](/cli/directory)
- [`nodes`](/cli/nodes)
- [`devices`](/cli/devices)
- [`node`](/cli/node)
- [`approvals`](/cli/approvals)
- [`sandbox`](/cli/sandbox)
- [`tui`](/cli/tui)
- [`browser`](/cli/browser)
- [`cron`](/cli/cron)
- [`tasks`](/cli/index#tasks)
- [`flows`](/cli/flows)
- [`dns`](/cli/dns)
- [`docs`](/cli/docs)
- [`hooks`](/cli/hooks)
- [`webhooks`](/cli/webhooks)
- [`pairing`](/cli/pairing)
- [`qr`](/cli/qr)
- [`plugins`](/cli/plugins) (comandos de plugins)
- [`channels`](/cli/channels)
- [`security`](/cli/security)
- [`secrets`](/cli/secrets)
- [`skills`](/cli/skills)
- [`daemon`](/cli/daemon) (alias heredado para comandos de servicio del gateway)
- [`clawbot`](/cli/clawbot) (espacio de nombres de alias heredado)
- [`voicecall`](/cli/voicecall) (plugin; si está instalado)

## Indicadores globales

- `--dev`: aísla el estado bajo `~/.openclaw-dev` y desplaza los puertos predeterminados.
- `--profile <name>`: aísla el estado bajo `~/.openclaw-<name>`.
- `--container <name>`: dirige la ejecución a un contenedor con nombre.
- `--no-color`: deshabilita los colores ANSI.
- `--update`: abreviatura de `openclaw update` (solo para instalaciones de origen).
- `-V`, `--version`, `-v`: imprime la versión y sale.

## Estilo de salida

- Los colores ANSI y los indicadores de progreso solo se renderizan en sesiones TTY.
- Los hipervínculos OSC-8 se renderizan como enlaces en los terminales compatibles; en caso contrario se usan URLs sin formato.
- `--json` (y `--plain` donde se admita) deshabilita el estilo para obtener una salida limpia.
- `--no-color` deshabilita el estilo ANSI; también se respeta `NO_COLOR=1`.
- Los comandos de larga duración muestran un indicador de progreso (OSC 9;4 cuando es compatible).

## Paleta de colores

OpenClaw usa una paleta lobster para la salida de la CLI.

- `accent` (#FF5A2D): encabezados, etiquetas, resaltados principales.
- `accentBright` (#FF7A3D): nombres de comandos, énfasis.
- `accentDim` (#D14A22): texto de resaltado secundario.
- `info` (#FF8A5B): valores informativos.
- `success` (#2FBF71): estados correctos.
- `warn` (#FFB020): advertencias, respaldos, atención.
- `error` (#E23D2D): errores, fallos.
- `muted` (#8B7F77): desénfasis, metadatos.

Fuente de verdad de la paleta: `src/terminal/palette.ts` (la “paleta lobster”).

## Árbol de comandos

```
openclaw [--dev] [--profile <name>] <command>
  setup
  onboard
  configure
  config
    get
    set
    unset
    file
    schema
    validate
  completion
  doctor
  dashboard
  backup
    create
    verify
  security
    audit
  secrets
    reload
    audit
    configure
    apply
  reset
  uninstall
  update
    wizard
    status
  channels
    list
    status
    capabilities
    resolve
    logs
    add
    remove
    login
    logout
  directory
    self
    peers list
    groups list|members
  skills
    search
    install
    update
    list
    info
    check
  plugins
    list
    inspect
    install
    uninstall
    update
    enable
    disable
    doctor
    marketplace list
  memory
    status
    index
    search
  message
    send
    broadcast
    poll
    react
    reactions
    read
    edit
    delete
    pin
    unpin
    pins
    permissions
    search
    thread create|list|reply
    emoji list|upload
    sticker send|upload
    role info|add|remove
    channel info|list
    member info
    voice status
    event list|create
    timeout
    kick
    ban
  agent
  agents
    list
    add
    delete
    bindings
    bind
    unbind
    set-identity
  acp
  mcp
    serve
    list
    show
    set
    unset
  status
  health
  sessions
    cleanup
  tasks
    list
    audit
    maintenance
    show
    notify
    cancel
    flow list|show|cancel
  gateway
    call
    usage-cost
    health
    status
    probe
    discover
    install
    uninstall
    start
    stop
    restart
    run
  daemon
    status
    install
    uninstall
    start
    stop
    restart
  logs
  system
    event
    heartbeat last|enable|disable
    presence
  models
    list
    status
    set
    set-image
    aliases list|add|remove
    fallbacks list|add|remove|clear
    image-fallbacks list|add|remove|clear
    scan
    auth add|login|login-github-copilot|setup-token|paste-token
    auth order get|set|clear
  sandbox
    list
    recreate
    explain
  cron
    status
    list
    add
    edit
    rm
    enable
    disable
    runs
    run
  nodes
    status
    describe
    list
    pending
    approve
    reject
    rename
    invoke
    notify
    push
    canvas snapshot|present|hide|navigate|eval
    canvas a2ui push|reset
    camera list|snap|clip
    screen record
    location get
  devices
    list
    remove
    clear
    approve
    reject
    rotate
    revoke
  node
    run
    status
    install
    uninstall
    stop
    restart
  approvals
    get
    set
    allowlist add|remove
  browser
    status
    start
    stop
    reset-profile
    tabs
    open
    focus
    close
    profiles
    create-profile
    delete-profile
    screenshot
    snapshot
    navigate
    resize
    click
    type
    press
    hover
    drag
    select
    upload
    fill
    dialog
    wait
    evaluate
    console
    pdf
  hooks
    list
    info
    check
    enable
    disable
    install
    update
  webhooks
    gmail setup|run
  pairing
    list
    approve
  qr
  clawbot
    qr
  docs
  dns
    setup
  tui
```

Nota: los plugins pueden agregar comandos adicionales de nivel superior (por ejemplo `openclaw voicecall`).

## Seguridad

- `openclaw security audit` — audita la configuración y el estado local en busca de errores de seguridad comunes.
- `openclaw security audit --deep` — sonda en vivo del Gateway en modo best-effort.
- `openclaw security audit --fix` — endurece los valores predeterminados seguros y los permisos de estado/configuración.

## Secrets

### `secrets`

Gestiona SecretRef y la higiene relacionada de configuración/tiempo de ejecución.

Subcomandos:

- `secrets reload`
- `secrets audit`
- `secrets configure`
- `secrets apply --from <path>`

Opciones de `secrets reload`:

- `--url`, `--token`, `--timeout`, `--expect-final`, `--json`

Opciones de `secrets audit`:

- `--check`
- `--allow-exec`
- `--json`

Opciones de `secrets configure`:

- `--apply`
- `--yes`
- `--providers-only`
- `--skip-provider-setup`
- `--agent <id>`
- `--allow-exec`
- `--plan-out <path>`
- `--json`

Opciones de `secrets apply --from <path>`:

- `--dry-run`
- `--allow-exec`
- `--json`

Notas:

- `reload` es un RPC del Gateway y conserva la última instantánea correcta conocida del tiempo de ejecución cuando falla la resolución.
- `audit --check` devuelve un valor distinto de cero cuando hay hallazgos; las referencias no resueltas usan un código de salida distinto de cero de mayor prioridad.
- Las comprobaciones exec en modo dry-run se omiten de forma predeterminada; usa `--allow-exec` para activarlas.

## Plugins

Gestiona extensiones y su configuración:

- `openclaw plugins list` — detecta plugins (usa `--json` para salida legible por máquina).
- `openclaw plugins inspect <id>` — muestra detalles de un plugin (`info` es un alias).
- `openclaw plugins install <path|.tgz|npm-spec|plugin@marketplace>` — instala un plugin (o agrega una ruta de plugin a `plugins.load.paths`; usa `--force` para sobrescribir un destino de instalación existente).
- `openclaw plugins marketplace list <marketplace>` — lista las entradas del marketplace antes de instalar.
- `openclaw plugins enable <id>` / `disable <id>` — activa o desactiva `plugins.entries.<id>.enabled`.
- `openclaw plugins doctor` — informa errores de carga de plugins.

La mayoría de los cambios en plugins requieren reiniciar el gateway. Consulta [/plugin](/tools/plugin).

## Memory

Búsqueda vectorial en `MEMORY.md` + `memory/*.md`:

- `openclaw memory status` — muestra estadísticas del índice; usa `--deep` para comprobaciones de preparación de vector + embeddings o `--fix` para reparar artefactos obsoletos de recuperación/promoción.
- `openclaw memory index` — reindexa archivos de memoria.
- `openclaw memory search "<query>"` (o `--query "<query>"`) — búsqueda semántica en la memoria.
- `openclaw memory promote` — clasifica recuperaciones a corto plazo y, opcionalmente, agrega las entradas principales a `MEMORY.md`.

## Sandbox

Gestiona runtimes de sandbox para ejecución aislada de agentes. Consulta [/cli/sandbox](/cli/sandbox).

Subcomandos:

- `sandbox list [--browser] [--json]`
- `sandbox recreate [--all] [--session <key>] [--agent <id>] [--browser] [--force]`
- `sandbox explain [--session <key>] [--agent <id>] [--json]`

Notas:

- `sandbox recreate` elimina los runtimes existentes para que el siguiente uso los vuelva a inicializar con la configuración actual.
- Para backends `ssh` y `remote` de OpenShell, recreate elimina el espacio de trabajo remoto canónico del alcance seleccionado.

## Comandos slash de chat

Los mensajes de chat admiten comandos `/...` (de texto y nativos). Consulta [/tools/slash-commands](/tools/slash-commands).

Puntos destacados:

- `/status` para diagnósticos rápidos.
- `/config` para cambios de configuración persistidos.
- `/debug` para reemplazos de configuración solo de tiempo de ejecución (memoria, no disco; requiere `commands.debug: true`).

## Configuración inicial e incorporación

### `completion`

Genera scripts de autocompletado del shell y opcionalmente los instala en tu perfil del shell.

Opciones:

- `-s, --shell <zsh|bash|powershell|fish>`
- `-i, --install`
- `--write-state`
- `-y, --yes`

Notas:

- Sin `--install` ni `--write-state`, `completion` imprime el script en stdout.
- `--install` escribe un bloque `OpenClaw Completion` en tu perfil del shell y lo apunta al script en caché bajo el directorio de estado de OpenClaw.

### `setup`

Inicializa la configuración y el espacio de trabajo.

Opciones:

- `--workspace <dir>`: ruta del espacio de trabajo del agente (predeterminada `~/.openclaw/workspace`).
- `--wizard`: ejecuta la incorporación.
- `--non-interactive`: ejecuta la incorporación sin preguntas.
- `--mode <local|remote>`: modo de incorporación.
- `--remote-url <url>`: URL remota del Gateway.
- `--remote-token <token>`: token remoto del Gateway.

La incorporación se ejecuta automáticamente cuando hay presentes indicadores de incorporación (`--non-interactive`, `--mode`, `--remote-url`, `--remote-token`).

### `onboard`

Incorporación interactiva para gateway, espacio de trabajo y Skills.

Opciones:

- `--workspace <dir>`
- `--reset` (restablece configuración + credenciales + sesiones antes de la incorporación)
- `--reset-scope <config|config+creds+sessions|full>` (predeterminado `config+creds+sessions`; usa `full` para eliminar también el espacio de trabajo)
- `--non-interactive`
- `--mode <local|remote>`
- `--flow <quickstart|advanced|manual>` (manual es un alias de advanced)
- `--auth-choice <choice>` donde `<choice>` es uno de:
  `chutes`, `deepseek-api-key`, `openai-codex`, `openai-api-key`,
  `openrouter-api-key`, `kilocode-api-key`, `litellm-api-key`, `ai-gateway-api-key`,
  `cloudflare-ai-gateway-api-key`, `moonshot-api-key`, `moonshot-api-key-cn`,
  `kimi-code-api-key`, `synthetic-api-key`, `venice-api-key`, `together-api-key`,
  `huggingface-api-key`, `apiKey`, `gemini-api-key`, `google-gemini-cli`, `zai-api-key`,
  `zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`, `xiaomi-api-key`,
  `minimax-global-oauth`, `minimax-global-api`, `minimax-cn-oauth`, `minimax-cn-api`,
  `opencode-zen`, `opencode-go`, `github-copilot`, `copilot-proxy`, `xai-api-key`,
  `mistral-api-key`, `volcengine-api-key`, `byteplus-api-key`, `qianfan-api-key`,
  `qwen-standard-api-key-cn`, `qwen-standard-api-key`, `qwen-api-key-cn`, `qwen-api-key`,
  `modelstudio-standard-api-key-cn`, `modelstudio-standard-api-key`,
  `modelstudio-api-key-cn`, `modelstudio-api-key`, `custom-api-key`, `skip`
- Nota sobre Qwen: `qwen-*` es la familia canónica de `auth-choice`. Los ids
  `modelstudio-*` siguen aceptándose solo como alias heredados de compatibilidad.
- `--secret-input-mode <plaintext|ref>` (predeterminado `plaintext`; usa `ref` para almacenar referencias env predeterminadas del proveedor en lugar de claves en texto sin formato)
- `--anthropic-api-key <key>`
- `--openai-api-key <key>`
- `--mistral-api-key <key>`
- `--openrouter-api-key <key>`
- `--ai-gateway-api-key <key>`
- `--moonshot-api-key <key>`
- `--kimi-code-api-key <key>`
- `--gemini-api-key <key>`
- `--zai-api-key <key>`
- `--minimax-api-key <key>`
- `--opencode-zen-api-key <key>`
- `--opencode-go-api-key <key>`
- `--custom-base-url <url>` (no interactivo; se usa con `--auth-choice custom-api-key`)
- `--custom-model-id <id>` (no interactivo; se usa con `--auth-choice custom-api-key`)
- `--custom-api-key <key>` (no interactivo; opcional; se usa con `--auth-choice custom-api-key`; recurre a `CUSTOM_API_KEY` cuando se omite)
- `--custom-provider-id <id>` (no interactivo; id opcional de proveedor personalizado)
- `--custom-compatibility <openai|anthropic>` (no interactivo; opcional; predeterminado `openai`)
- `--gateway-port <port>`
- `--gateway-bind <loopback|lan|tailnet|auto|custom>`
- `--gateway-auth <token|password>`
- `--gateway-token <token>`
- `--gateway-token-ref-env <name>` (no interactivo; almacena `gateway.auth.token` como un env SecretRef; requiere que esa variable de entorno esté definida; no puede combinarse con `--gateway-token`)
- `--gateway-password <password>`
- `--remote-url <url>`
- `--remote-token <token>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--install-daemon`
- `--no-install-daemon` (alias: `--skip-daemon`)
- `--daemon-runtime <node|bun>`
- `--skip-channels`
- `--skip-skills`
- `--skip-search`
- `--skip-health`
- `--skip-ui`
- `--cloudflare-ai-gateway-account-id <id>`
- `--cloudflare-ai-gateway-gateway-id <id>`
- `--node-manager <npm|pnpm|bun>` (administrador de nodos para configuración/incorporación de Skills; se recomienda pnpm, bun también es compatible)
- `--json`

### `configure`

Asistente interactivo de configuración (modelos, canales, Skills, gateway).

Opciones:

- `--section <section>` (repetible; limita el asistente a secciones específicas)

### `config`

Ayudas de configuración no interactivas (get/set/unset/file/schema/validate). Ejecutar `openclaw config` sin
subcomando inicia el asistente.

Subcomandos:

- `config get <path>`: imprime un valor de configuración (ruta dot/bracket).
- `config set`: admite cuatro modos de asignación:
  - modo valor: `config set <path> <value>` (análisis JSON5 o cadena)
  - modo constructor de SecretRef: `config set <path> --ref-provider <provider> --ref-source <source> --ref-id <id>`
  - modo constructor de proveedor: `config set secrets.providers.<alias> --provider-source <env|file|exec> ...`
  - modo por lotes: `config set --batch-json '<json>'` o `config set --batch-file <path>`
- `config set --dry-run`: valida asignaciones sin escribir `openclaw.json` (las comprobaciones exec de SecretRef se omiten de forma predeterminada).
- `config set --allow-exec --dry-run`: activa las comprobaciones dry-run de SecretRef exec (puede ejecutar comandos del proveedor).
- `config set --dry-run --json`: emite salida dry-run legible por máquina (comprobaciones + señal de completitud, operaciones, refs comprobadas/omitidas, errores).
- `config set --strict-json`: requiere análisis JSON5 para la entrada path/value. `--json` sigue siendo un alias heredado de análisis estricto fuera del modo de salida dry-run.
- `config unset <path>`: elimina un valor.
- `config file`: imprime la ruta del archivo de configuración activo.
- `config schema`: imprime el esquema JSON generado para `openclaw.json`, incluidos metadatos de documentación `title` / `description` propagados a través de ramas de objetos anidados, comodines, elementos de arrays y composición, además de metadatos en vivo best-effort del esquema de plugins/canales.
- `config validate`: valida la configuración actual contra el esquema sin iniciar el gateway.
- `config validate --json`: emite salida JSON legible por máquina.

### `doctor`

Comprobaciones de salud y correcciones rápidas (configuración + gateway + servicios heredados).

Opciones:

- `--no-workspace-suggestions`: deshabilita sugerencias de memoria del espacio de trabajo.
- `--yes`: acepta los valores predeterminados sin preguntar (sin interfaz).
- `--non-interactive`: omite preguntas; aplica solo migraciones seguras.
- `--deep`: analiza servicios del sistema en busca de instalaciones adicionales del gateway.
- `--repair` (alias: `--fix`): intenta reparaciones automáticas para los problemas detectados.
- `--force`: fuerza reparaciones incluso cuando no son estrictamente necesarias.
- `--generate-gateway-token`: genera un nuevo token de autenticación del gateway.

### `dashboard`

Abre la UI de control con tu token actual.

Opciones:

- `--no-open`: imprime la URL, pero no inicia un navegador

Notas:

- Para tokens de gateway gestionados por SecretRef, `dashboard` imprime o abre una URL sin tokenizar en lugar de exponer el secreto en la salida del terminal o en los argumentos de inicio del navegador.

### `update`

Actualiza la CLI instalada.

Opciones raíz:

- `--json`
- `--no-restart`
- `--dry-run`
- `--channel <stable|beta|dev>`
- `--tag <dist-tag|version|spec>`
- `--timeout <seconds>`
- `--yes`

Subcomandos:

- `update status`
- `update wizard`

Opciones de `update status`:

- `--json`
- `--timeout <seconds>`

Opciones de `update wizard`:

- `--timeout <seconds>`

Notas:

- `openclaw --update` se reescribe como `openclaw update`.

### `backup`

Crea y verifica archivos locales de respaldo para el estado de OpenClaw.

Subcomandos:

- `backup create`
- `backup verify <archive>`

Opciones de `backup create`:

- `--output <path>`
- `--json`
- `--dry-run`
- `--verify`
- `--only-config`
- `--no-include-workspace`

Opciones de `backup verify <archive>`:

- `--json`

## Ayudas de canales

### `channels`

Gestiona cuentas de canales de chat (WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Microsoft Teams).

Subcomandos:

- `channels list`: muestra canales configurados y perfiles de autenticación.
- `channels status`: comprueba la accesibilidad del gateway y la salud de los canales (`--probe` ejecuta comprobaciones de sonda/auditoría en vivo por cuenta cuando el gateway es accesible; si no lo es, recurre a resúmenes de canales basados solo en configuración. Usa `openclaw health` o `openclaw status --deep` para sondas más amplias de salud del gateway).
- Consejo: `channels status` imprime advertencias con correcciones sugeridas cuando puede detectar errores de configuración comunes (y luego te remite a `openclaw doctor`).
- `channels logs`: muestra registros recientes de canales del archivo de registro del gateway.
- `channels add`: configuración tipo asistente cuando no se pasan indicadores; los indicadores cambian al modo no interactivo.
  - Al agregar una cuenta no predeterminada a un canal que aún usa configuración de una sola cuenta de nivel superior, OpenClaw promociona los valores con alcance de cuenta al mapa de cuentas del canal antes de escribir la nueva cuenta. La mayoría de los canales usan `accounts.default`; Matrix puede conservar un destino existente coincidente con nombre/predeterminado.
  - `channels add` no interactivo no crea ni actualiza automáticamente bindings; los bindings solo de canal siguen coincidiendo con la cuenta predeterminada.
- `channels remove`: deshabilita de forma predeterminada; pasa `--delete` para eliminar entradas de configuración sin preguntas.
- `channels login`: inicio de sesión interactivo en canal (solo WhatsApp Web).
- `channels logout`: cierra la sesión de un canal (si es compatible).

Opciones comunes:

- `--channel <name>`: `whatsapp|telegram|discord|googlechat|slack|mattermost|signal|imessage|msteams`
- `--account <id>`: id de cuenta del canal (predeterminado `default`)
- `--name <label>`: nombre visible de la cuenta

Opciones de `channels login`:

- `--channel <channel>` (predeterminado `whatsapp`; admite `whatsapp`/`web`)
- `--account <id>`
- `--verbose`

Opciones de `channels logout`:

- `--channel <channel>` (predeterminado `whatsapp`)
- `--account <id>`

Opciones de `channels list`:

- `--no-usage`: omite instantáneas de uso/cuota del proveedor de modelos (solo con OAuth/API).
- `--json`: salida JSON (incluye uso salvo que se defina `--no-usage`).

Opciones de `channels status`:

- `--probe`
- `--timeout <ms>`
- `--json`

Opciones de `channels capabilities`:

- `--channel <name>`
- `--account <id>` (solo con `--channel`)
- `--target <dest>`
- `--timeout <ms>`
- `--json`

Opciones de `channels resolve`:

- `<entries...>`
- `--channel <name>`
- `--account <id>`
- `--kind <auto|user|group>`
- `--json`

Opciones de `channels logs`:

- `--channel <name|all>` (predeterminado `all`)
- `--lines <n>` (predeterminado `200`)
- `--json`

Notas:

- `channels login` admite `--verbose`.
- `channels capabilities --account` solo se aplica cuando se define `--channel`.
- `channels status --probe` puede mostrar el estado del transporte además de resultados de sonda/auditoría como `works`, `probe failed`, `audit ok` o `audit failed`, según la compatibilidad del canal.

Más detalles: [/concepts/oauth](/concepts/oauth)

Ejemplos:

```bash
openclaw channels add --channel telegram --account alerts --name "Alerts Bot" --token $TELEGRAM_BOT_TOKEN
openclaw channels add --channel discord --account work --name "Work Bot" --token $DISCORD_BOT_TOKEN
openclaw channels remove --channel discord --account work --delete
openclaw channels status --probe
openclaw status --deep
```

### `directory`

Busca IDs propios, de pares y de grupos para canales que exponen una superficie de directorio. Consulta [`openclaw directory`](/cli/directory).

Opciones comunes:

- `--channel <name>`
- `--account <id>`
- `--json`

Subcomandos:

- `directory self`
- `directory peers list [--query <text>] [--limit <n>]`
- `directory groups list [--query <text>] [--limit <n>]`
- `directory groups members --group-id <id> [--limit <n>]`

### `skills`

Lista e inspecciona Skills disponibles, además de información de disponibilidad.

Subcomandos:

- `skills search [query...]`: busca Skills en ClawHub.
- `skills search --limit <n> --json`: limita los resultados de búsqueda o emite salida legible por máquina.
- `skills install <slug>`: instala un Skill desde ClawHub en el espacio de trabajo activo.
- `skills install <slug> --version <version>`: instala una versión específica de ClawHub.
- `skills install <slug> --force`: sobrescribe una carpeta existente de Skill del espacio de trabajo.
- `skills update <slug|--all>`: actualiza Skills rastreados de ClawHub.
- `skills list`: lista Skills (predeterminado cuando no hay subcomando).
- `skills list --json`: emite en stdout el inventario de Skills legible por máquina.
- `skills list --verbose`: incluye requisitos faltantes en la tabla.
- `skills info <name>`: muestra detalles de un Skill.
- `skills info <name> --json`: emite en stdout detalles legibles por máquina.
- `skills check`: resumen de Skills listos frente a faltantes.
- `skills check --json`: emite en stdout la salida de disponibilidad legible por máquina.

Opciones:

- `--eligible`: muestra solo Skills listos.
- `--json`: salida JSON (sin estilo).
- `-v`, `--verbose`: incluye detalle de requisitos faltantes.

Consejo: usa `openclaw skills search`, `openclaw skills install` y `openclaw skills update` para Skills respaldados por ClawHub.

### `pairing`

Aprueba solicitudes de emparejamiento de mensajes directos entre canales.

Subcomandos:

- `pairing list [channel] [--channel <channel>] [--account <id>] [--json]`
- `pairing approve <channel> <code> [--account <id>] [--notify]`
- `pairing approve --channel <channel> [--account <id>] <code> [--notify]`

Notas:

- Si exactamente un canal compatible con emparejamiento está configurado, también se permite `pairing approve <code>`.
- Tanto `list` como `approve` admiten `--account <id>` para canales con varias cuentas.

### `devices`

Gestiona entradas de emparejamiento de dispositivos del gateway y tokens de dispositivo por rol.

Subcomandos:

- `devices list [--json]`
- `devices approve [requestId] [--latest]`
- `devices reject <requestId>`
- `devices remove <deviceId>`
- `devices clear --yes [--pending]`
- `devices rotate --device <id> --role <role> [--scope <scope...>]`
- `devices revoke --device <id> --role <role>`

Notas:

- `devices list` y `devices approve` pueden recurrir a archivos locales de emparejamiento en local loopback cuando el alcance directo de emparejamiento no está disponible.
- `devices approve` selecciona automáticamente la solicitud pendiente más reciente cuando no se pasa `requestId` o se define `--latest`.
- Las reconexiones con tokens almacenados reutilizan los alcances aprobados en caché del token; `devices rotate --scope ...` explícito actualiza ese conjunto de alcances almacenado para futuras reconexiones con token en caché.
- `devices rotate` y `devices revoke` devuelven cargas JSON.

### `qr`

Genera un QR de emparejamiento móvil y un código de configuración a partir de la configuración actual del Gateway. Consulta [`openclaw qr`](/cli/qr).

Opciones:

- `--remote`
- `--url <url>`
- `--public-url <url>`
- `--token <token>`
- `--password <password>`
- `--setup-code-only`
- `--no-ascii`
- `--json`

Notas:

- `--token` y `--password` son mutuamente excluyentes.
- El código de configuración lleva un token de bootstrap de corta duración, no el token/contraseña compartidos del gateway.
- La transferencia integrada de bootstrap mantiene el token del nodo principal en `scopes: []`.
- Cualquier token de bootstrap de operador transferido sigue limitado a `operator.approvals`, `operator.read`, `operator.talk.secrets` y `operator.write`.
- Las comprobaciones de alcance de bootstrap usan prefijos por rol, por lo que esa allowlist de operador solo satisface solicitudes de operador; los roles que no son operador siguen necesitando alcances bajo su propio prefijo de rol.
- `--remote` puede usar `gateway.remote.url` o la URL activa de Tailscale Serve/Funnel.
- Después de escanear, aprueba la solicitud con `openclaw devices list` / `openclaw devices approve <requestId>`.

### `clawbot`

Espacio de nombres de alias heredado. Actualmente admite `openclaw clawbot qr`, que corresponde a [`openclaw qr`](/cli/qr).

### `hooks`

Gestiona hooks internos del agente.

Subcomandos:

- `hooks list`
- `hooks info <name>`
- `hooks check`
- `hooks enable <name>`
- `hooks disable <name>`
- `hooks install <path-or-spec>` (alias obsoleto de `openclaw plugins install`)
- `hooks update [id]` (alias obsoleto de `openclaw plugins update`)

Opciones comunes:

- `--json`
- `--eligible`
- `-v`, `--verbose`

Notas:

- Los hooks gestionados por plugins no pueden habilitarse ni deshabilitarse mediante `openclaw hooks`; en su lugar, habilita o deshabilita el plugin propietario.
- `hooks install` y `hooks update` siguen funcionando como alias de compatibilidad, pero imprimen advertencias de obsolescencia y redirigen a los comandos de plugins.

### `webhooks`

Ayudas de webhooks. La superficie integrada actual es configuración + ejecutor de Gmail Pub/Sub:

- `webhooks gmail setup`
- `webhooks gmail run`

### `webhooks gmail`

Configuración + ejecutor del hook de Gmail Pub/Sub. Consulta [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration).

Subcomandos:

- `webhooks gmail setup` (requiere `--account <email>`; admite `--project`, `--topic`, `--subscription`, `--label`, `--hook-url`, `--hook-token`, `--push-token`, `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes`, `--tailscale`, `--tailscale-path`, `--tailscale-target`, `--push-endpoint`, `--json`)
- `webhooks gmail run` (sustituciones de tiempo de ejecución para los mismos indicadores)

Notas:

- `setup` configura la observación de Gmail más la ruta push orientada a OpenClaw.
- `run` inicia el observador local de Gmail/bucle de renovación con sustituciones opcionales de tiempo de ejecución.

### `dns`

Ayudas DNS de descubrimiento de área amplia (CoreDNS + Tailscale). La superficie integrada actual:

- `dns setup [--domain <domain>] [--apply]`

### `dns setup`

Ayuda DNS de descubrimiento de área amplia (CoreDNS + Tailscale). Consulta [/gateway/discovery](/gateway/discovery).

Opciones:

- `--domain <domain>`
- `--apply`: instala/actualiza la configuración de CoreDNS (requiere sudo; solo macOS).

Notas:

- Sin `--apply`, esta es una ayuda de planificación que imprime la configuración DNS recomendada de OpenClaw + Tailscale.
- `--apply` actualmente es compatible con macOS solo con CoreDNS de Homebrew.

## Mensajería + agente

### `message`

Mensajería saliente unificada + acciones de canal.

Consulta: [/cli/message](/cli/message)

Subcomandos:

- `message send|poll|react|reactions|read|edit|delete|pin|unpin|pins|permissions|search|timeout|kick|ban`
- `message thread <create|list|reply>`
- `message emoji <list|upload>`
- `message sticker <send|upload>`
- `message role <info|add|remove>`
- `message channel <info|list>`
- `message member info`
- `message voice status`
- `message event <list|create>`

Ejemplos:

- `openclaw message send --target +15555550123 --message "Hi"`
- `openclaw message poll --channel discord --target channel:123 --poll-question "Snack?" --poll-option Pizza --poll-option Sushi`

### `agent`

Ejecuta un turno de agente mediante el Gateway (o integrado con `--local`).

Pasa al menos un selector de sesión: `--to`, `--session-id` o `--agent`.

Obligatorio:

- `-m, --message <text>`

Opciones:

- `-t, --to <dest>` (para clave de sesión y entrega opcional)
- `--session-id <id>`
- `--agent <id>` (id del agente; reemplaza los bindings de enrutamiento)
- `--thinking <off|minimal|low|medium|high|xhigh>` (la compatibilidad del proveedor varía; no hay control a nivel CLI por modelo)
- `--verbose <on|off>`
- `--channel <channel>` (canal de entrega; omítelo para usar el canal de la sesión principal)
- `--reply-to <target>` (reemplazo del destino de entrega, separado del enrutamiento de sesión)
- `--reply-channel <channel>` (reemplazo del canal de entrega)
- `--reply-account <id>` (reemplazo del id de cuenta de entrega)
- `--local` (ejecución integrada; el registro de plugins sigue precargándose primero)
- `--deliver`
- `--json`
- `--timeout <seconds>`

Notas:

- El modo Gateway recurre al agente integrado cuando falla la solicitud al Gateway.
- `--local` sigue precargando el registro de plugins, por lo que los proveedores, herramientas y canales proporcionados por plugins siguen disponibles durante las ejecuciones integradas.
- `--channel`, `--reply-channel` y `--reply-account` afectan la entrega de la respuesta, no el enrutamiento.

### `agents`

Gestiona agentes aislados (espacios de trabajo + autenticación + enrutamiento).

Ejecutar `openclaw agents` sin subcomando equivale a `openclaw agents list`.

#### `agents list`

Lista agentes configurados.

Opciones:

- `--json`
- `--bindings`

#### `agents add [name]`

Agrega un nuevo agente aislado. Ejecuta el asistente guiado a menos que se pasen indicadores (o `--non-interactive`); `--workspace` es obligatorio en modo no interactivo.

Opciones:

- `--workspace <dir>`
- `--model <id>`
- `--agent-dir <dir>`
- `--bind <channel[:accountId]>` (repetible)
- `--non-interactive`
- `--json`

Las especificaciones de binding usan `channel[:accountId]`. Cuando se omite `accountId`, OpenClaw puede resolver el alcance de la cuenta mediante valores predeterminados del canal/hooks del plugin; de lo contrario es un binding de canal sin alcance explícito de cuenta.
Pasar cualquier indicador explícito de add cambia el comando a la ruta no interactiva. `main` está reservado y no puede usarse como nuevo id de agente.

#### `agents bindings`

Lista bindings de enrutamiento.

Opciones:

- `--agent <id>`
- `--json`

#### `agents bind`

Agrega bindings de enrutamiento para un agente.

Opciones:

- `--agent <id>` (predeterminado: el agente actual predeterminado)
- `--bind <channel[:accountId]>` (repetible)
- `--json`

#### `agents unbind`

Elimina bindings de enrutamiento de un agente.

Opciones:

- `--agent <id>` (predeterminado: el agente actual predeterminado)
- `--bind <channel[:accountId]>` (repetible)
- `--all`
- `--json`

Usa `--all` o `--bind`, no ambos.

#### `agents delete <id>`

Elimina un agente y poda su espacio de trabajo + estado.

Opciones:

- `--force`
- `--json`

Notas:

- `main` no puede eliminarse.
- Sin `--force`, se requiere confirmación interactiva.

#### `agents set-identity`

Actualiza la identidad de un agente (nombre/tema/emoji/avatar).

Opciones:

- `--agent <id>`
- `--workspace <dir>`
- `--identity-file <path>`
- `--from-identity`
- `--name <name>`
- `--theme <theme>`
- `--emoji <emoji>`
- `--avatar <value>`
- `--json`

Notas:

- `--agent` o `--workspace` pueden usarse para seleccionar el agente de destino.
- Cuando no se proporcionan campos explícitos de identidad, el comando lee `IDENTITY.md`.

### `acp`

Ejecuta el puente ACP que conecta IDEs con el Gateway.

Opciones raíz:

- `--url <url>`
- `--token <token>`
- `--token-file <path>`
- `--password <password>`
- `--password-file <path>`
- `--session <key>`
- `--session-label <label>`
- `--require-existing`
- `--reset-session`
- `--no-prefix-cwd`
- `--provenance <off|meta|meta+receipt>`
- `--verbose`

#### `acp client`

Cliente ACP interactivo para depuración del puente.

Opciones:

- `--cwd <dir>`
- `--server <command>`
- `--server-args <args...>`
- `--server-verbose`
- `--verbose`

Consulta [`acp`](/cli/acp) para conocer el comportamiento completo, notas de seguridad y ejemplos.

### `mcp`

Gestiona definiciones guardadas de servidores MCP y expone canales de OpenClaw sobre MCP stdio.

#### `mcp serve`

Expone conversaciones de canales de OpenClaw enrutadas sobre MCP stdio.

Opciones:

- `--url <url>`
- `--token <token>`
- `--token-file <path>`
- `--password <password>`
- `--password-file <path>`
- `--claude-channel-mode <auto|on|off>`
- `--verbose`

#### `mcp list`

Lista definiciones guardadas de servidores MCP.

Opciones:

- `--json`

#### `mcp show [name]`

Muestra una definición guardada de servidor MCP o el objeto completo guardado del servidor MCP.

Opciones:

- `--json`

#### `mcp set <name> <value>`

Guarda una definición de servidor MCP a partir de un objeto JSON.

#### `mcp unset <name>`

Elimina una definición guardada de servidor MCP.

### `approvals`

Gestiona aprobaciones de exec. Alias: `exec-approvals`.

#### `approvals get`

Obtiene la instantánea de aprobaciones de exec y la política efectiva.

Opciones:

- `--node <node>`
- `--gateway`
- `--json`
- opciones RPC de nodo de `openclaw nodes`

#### `approvals set`

Reemplaza las aprobaciones de exec con JSON de un archivo o stdin.

Opciones:

- `--node <node>`
- `--gateway`
- `--file <path>`
- `--stdin`
- `--json`
- opciones RPC de nodo de `openclaw nodes`

#### `approvals allowlist add|remove`

Edita la allowlist de exec por agente.

Opciones:

- `--node <node>`
- `--gateway`
- `--agent <id>` (predeterminado `*`)
- `--json`
- opciones RPC de nodo de `openclaw nodes`

### `status`

Muestra el estado de salud de la sesión vinculada y destinatarios recientes.

Opciones:

- `--json`
- `--all` (diagnóstico completo; solo lectura, apto para pegar)
- `--deep` (pide al gateway una sonda de salud en vivo, incluidas sondas de canal cuando se admiten)
- `--usage` (muestra uso/cuota del proveedor de modelos)
- `--timeout <ms>`
- `--verbose`
- `--debug` (alias de `--verbose`)

Notas:

- La vista general incluye el estado del servicio del Gateway + host del nodo cuando está disponible.
- `--usage` imprime ventanas de uso normalizadas del proveedor como `X% left`.

### Seguimiento de uso

OpenClaw puede mostrar uso/cuota del proveedor cuando hay credenciales OAuth/API disponibles.

Superficies:

- `/status` (agrega una línea corta de uso del proveedor cuando está disponible)
- `openclaw status --usage` (imprime el desglose completo por proveedor)
- barra de menú de macOS (sección Usage en Context)

Notas:

- Los datos provienen directamente de endpoints de uso del proveedor (sin estimaciones).
- La salida legible para humanos se normaliza a `X% left` entre proveedores.
- Proveedores con ventanas actuales de uso: Anthropic, GitHub Copilot, Gemini CLI, OpenAI Codex, MiniMax, Xiaomi y z.ai.
- Nota sobre MiniMax: `usage_percent` / `usagePercent` sin procesar significa cuota restante, por lo que OpenClaw lo invierte antes de mostrarlo; los campos basados en recuento siguen teniendo prioridad cuando están presentes. Las respuestas `model_remains` prefieren la entrada del modelo de chat, derivan la etiqueta de ventana a partir de marcas de tiempo cuando es necesario e incluyen el nombre del modelo en la etiqueta del plan.
- La autenticación para uso proviene de hooks específicos del proveedor cuando están disponibles; de lo contrario, OpenClaw recurre a credenciales OAuth/API-key coincidentes desde perfiles de autenticación, variables de entorno o configuración. Si no se resuelve ninguna, el uso se oculta.
- Detalles: consulta [Seguimiento de uso](/concepts/usage-tracking).

### `health`

Obtiene el estado de salud del Gateway en ejecución.

Opciones:

- `--json`
- `--timeout <ms>`
- `--verbose` (fuerza una sonda en vivo e imprime detalles de conexión del gateway)
- `--debug` (alias de `--verbose`)

Notas:

- `health` predeterminado puede devolver una instantánea reciente en caché del gateway.
- `health --verbose` fuerza una sonda en vivo y amplía la salida legible para humanos en todas las cuentas y agentes configurados.

### `sessions`

Lista sesiones de conversación almacenadas.

Opciones:

- `--json`
- `--verbose`
- `--store <path>`
- `--active <minutes>`
- `--agent <id>` (filtra sesiones por agente)
- `--all-agents` (muestra sesiones de todos los agentes)

Subcomandos:

- `sessions cleanup` — elimina sesiones expiradas u huérfanas

Notas:

- `sessions cleanup` también admite `--fix-missing` para podar entradas cuyos archivos de transcripción ya no existen.

## Restablecer / Desinstalar

### `reset`

Restablece la configuración/estado local (mantiene la CLI instalada).

Opciones:

- `--scope <config|config+creds+sessions|full>`
- `--yes`
- `--non-interactive`
- `--dry-run`

Notas:

- `--non-interactive` requiere `--scope` y `--yes`.

### `uninstall`

Desinstala el servicio gateway + datos locales (la CLI permanece).

Opciones:

- `--service`
- `--state`
- `--workspace`
- `--app`
- `--all`
- `--yes`
- `--non-interactive`
- `--dry-run`

Notas:

- `--non-interactive` requiere `--yes` y alcances explícitos (o `--all`).
- `--all` elimina juntos servicio, estado, espacio de trabajo y app.

### `tasks`

Lista y gestiona ejecuciones de [tareas en segundo plano](/automation/tasks) entre agentes.

- `tasks list` — muestra ejecuciones de tareas activas y recientes
- `tasks show <id>` — muestra detalles de una ejecución específica de tarea
- `tasks notify <id>` — cambia la política de notificación de una ejecución de tarea
- `tasks cancel <id>` — cancela una tarea en ejecución
- `tasks audit` — muestra problemas operativos (obsoletas, perdidas, fallos de entrega)
- `tasks maintenance [--apply] [--json]` — previsualiza o aplica limpieza/reconciliación de tareas y TaskFlow (sesiones hijas ACP/subagente, trabajos cron activos, ejecuciones en vivo de la CLI)
- `tasks flow list` — lista flujos activos y recientes de Task Flow
- `tasks flow show <lookup>` — inspecciona un flujo por id o clave de búsqueda
- `tasks flow cancel <lookup>` — cancela un flujo en ejecución y sus tareas activas

### `flows`

Atajo heredado de documentación. Los comandos de flujos están en `openclaw tasks flow`:

- `tasks flow list [--json]`
- `tasks flow show <lookup>`
- `tasks flow cancel <lookup>`

## Gateway

### `gateway`

Ejecuta el Gateway WebSocket.

Opciones:

- `--port <port>`
- `--bind <loopback|tailnet|lan|auto|custom>`
- `--token <token>`
- `--auth <token|password>`
- `--password <password>`
- `--password-file <path>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--allow-unconfigured`
- `--dev`
- `--reset` (restablece configuración dev + credenciales + sesiones + espacio de trabajo)
- `--force` (mata el listener existente en el puerto)
- `--verbose`
- `--cli-backend-logs`
- `--claude-cli-logs` (alias obsoleto)
- `--ws-log <auto|full|compact>`
- `--compact` (alias de `--ws-log compact`)
- `--raw-stream`
- `--raw-stream-path <path>`

### `gateway service`

Gestiona el servicio del Gateway (launchd/systemd/schtasks).

Subcomandos:

- `gateway status` (sondea el RPC del Gateway de forma predeterminada)
- `gateway install` (instalación del servicio)
- `gateway uninstall`
- `gateway start`
- `gateway stop`
- `gateway restart`

Notas:

- `gateway status` sondea el RPC del Gateway de forma predeterminada usando el puerto/configuración resueltos del servicio (reemplaza con `--url/--token/--password`).
- `gateway status` admite `--no-probe`, `--deep`, `--require-rpc` y `--json` para scripting.
- `gateway status` también muestra servicios heredados o adicionales del gateway cuando puede detectarlos (`--deep` agrega análisis a nivel del sistema). Los servicios de OpenClaw con nombre de perfil se tratan como de primera clase y no se marcan como “extra”.
- `gateway status` sigue disponible para diagnósticos incluso cuando falta la configuración local de la CLI o es no válida.
- `gateway status` imprime la ruta resuelta del archivo de registro, la instantánea de rutas/validez de configuración CLI frente a servicio y la URL resuelta del objetivo de la sonda.
- Si las SecretRef de autenticación del gateway no se resuelven en la ruta actual del comando, `gateway status --json` informa `rpc.authWarning` solo cuando falla la conectividad/autenticación de la sonda (las advertencias se suprimen cuando la sonda tiene éxito).
- En instalaciones Linux con systemd, las comprobaciones de deriva del token de estado incluyen tanto fuentes `Environment=` como `EnvironmentFile=` de la unidad.
- `gateway install|uninstall|start|stop|restart` admiten `--json` para scripting (la salida predeterminada sigue siendo amigable para humanos).
- `gateway install` usa Node runtime de forma predeterminada; bun **no se recomienda** (errores en WhatsApp/Telegram).
- Opciones de `gateway install`: `--port`, `--runtime`, `--token`, `--force`, `--json`.

### `daemon`

Alias heredado para los comandos de gestión del servicio Gateway. Consulta [/cli/daemon](/cli/daemon).

Subcomandos:

- `daemon status`
- `daemon install`
- `daemon uninstall`
- `daemon start`
- `daemon stop`
- `daemon restart`

Opciones comunes:

- `status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
- `install`: `--port`, `--runtime <node|bun>`, `--token`, `--force`, `--json`
- `uninstall|start|stop|restart`: `--json`

### `logs`

Sigue registros de archivos del Gateway mediante RPC.

Opciones:

- `--limit <n>`: número máximo de líneas de registro a devolver
- `--max-bytes <n>`: bytes máximos a leer del archivo de registro
- `--follow`: sigue el archivo de registro (estilo tail -f)
- `--interval <ms>`: intervalo de sondeo en ms al seguir
- `--local-time`: muestra marcas de tiempo en hora local
- `--json`: emite JSON delimitado por líneas
- `--plain`: deshabilita el formato estructurado
- `--no-color`: deshabilita colores ANSI
- `--url <url>`: URL WebSocket explícita del Gateway
- `--token <token>`: token del Gateway
- `--timeout <ms>`: tiempo de espera del RPC del Gateway
- `--expect-final`: espera una respuesta final cuando sea necesario

Ejemplos:

```bash
openclaw logs --follow
openclaw logs --limit 200
openclaw logs --plain
openclaw logs --json
openclaw logs --no-color
```

Notas:

- Si pasas `--url`, la CLI no aplica automáticamente credenciales de configuración ni de entorno.
- Los fallos de emparejamiento en local loopback recurren al archivo de registro local configurado; los objetivos explícitos con `--url` no.

### `gateway <subcommand>`

Ayudas de CLI del gateway (usa `--url`, `--token`, `--password`, `--timeout`, `--expect-final` para subcomandos RPC).
Cuando pasas `--url`, la CLI no aplica automáticamente credenciales de configuración ni de entorno.
Incluye `--token` o `--password` explícitamente. La ausencia de credenciales explícitas es un error.

Subcomandos:

- `gateway call <method> [--params <json>] [--url <url>] [--token <token>] [--password <password>] [--timeout <ms>] [--expect-final] [--json]`
- `gateway health`
- `gateway status`
- `gateway probe`
- `gateway discover`
- `gateway install|uninstall|start|stop|restart`
- `gateway run`

Notas:

- `gateway status --deep` agrega un análisis de servicio a nivel del sistema. Usa `gateway probe`,
  `health --verbose` o `status --deep` de nivel superior para obtener detalles más profundos de la sonda de tiempo de ejecución.

RPC comunes:

- `config.schema.lookup` (inspecciona un subárbol de configuración con un nodo de esquema superficial, metadatos de sugerencias coincidentes y resúmenes de hijos inmediatos)
- `config.get` (lee la instantánea actual de configuración + hash)
- `config.set` (valida + escribe la configuración completa; usa `baseHash` para concurrencia optimista)
- `config.apply` (valida + escribe configuración + reinicia + activa)
- `config.patch` (fusiona una actualización parcial + reinicia + activa)
- `update.run` (ejecuta actualización + reinicia + activa)

Consejo: al llamar directamente a `config.set`/`config.apply`/`config.patch`, pasa `baseHash` desde
`config.get` si ya existe una configuración.
Consejo: para ediciones parciales, inspecciona primero con `config.schema.lookup` y prefiere `config.patch`.
Consejo: estos RPC de escritura de configuración verifican previamente la resolución activa de SecretRef para refs del payload de configuración enviado y rechazan escrituras cuando una ref enviada efectivamente activa no está resuelta.
Consejo: la herramienta de tiempo de ejecución `gateway` solo para propietarios sigue negándose a reescribir `tools.exec.ask` o `tools.exec.security`; los alias heredados `tools.bash.*` se normalizan a las mismas rutas protegidas de exec.

## Modelos

Consulta [/concepts/models](/concepts/models) para conocer el comportamiento de respaldo y la estrategia de análisis.

Nota de facturación: Creemos que es probable que el respaldo de Claude Code CLI esté permitido para automatización local gestionada por el usuario según la documentación pública de CLI de Anthropic. Dicho esto, la política de harness de terceros de Anthropic crea suficiente ambigüedad en torno al uso respaldado por suscripción en productos externos como para que no lo recomendemos en producción. Anthropic también notificó a los usuarios de OpenClaw el **4 de abril de 2026 a las 12:00 PM PT / 8:00 PM BST** que la ruta de inicio de sesión de Claude de **OpenClaw** cuenta como uso de harness de terceros y requiere **Extra Usage** facturado por separado de la suscripción. Para producción, prefiere una API key de Anthropic u otro proveedor compatible de estilo suscripción como OpenAI Codex, Alibaba Cloud Model Studio Coding Plan, MiniMax Coding Plan o Z.AI / GLM Coding Plan.

Migración de Anthropic Claude CLI:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

Atajo de incorporación: `openclaw onboard --auth-choice anthropic-cli`

Anthropic setup-token también vuelve a estar disponible como ruta heredada/manual de autenticación.
Úsalo solo asumiendo que Anthropic informó a los usuarios de OpenClaw que la ruta de inicio de sesión de Claude de OpenClaw requiere **Extra Usage**.

Nota sobre alias heredado: `claude-cli` es el alias obsoleto de `auth-choice` para incorporación.
Usa `anthropic-cli` para la incorporación, o usa `models auth login` directamente.

### `models` (raíz)

`openclaw models` es un alias de `models status`.

Opciones raíz:

- `--status-json` (alias de `models status --json`)
- `--status-plain` (alias de `models status --plain`)

### `models list`

Opciones:

- `--all`
- `--local`
- `--provider <name>`
- `--json`
- `--plain`

### `models status`

Opciones:

- `--json`
- `--plain`
- `--check` (sale con 1=falta/expiró, 2=por expirar)
- `--probe` (sonda en vivo de perfiles de autenticación configurados)
- `--probe-provider <name>`
- `--probe-profile <id>` (repetible o separado por comas)
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`
- `--agent <id>`

Siempre incluye la vista general de autenticación y el estado de expiración OAuth para los perfiles del almacén de autenticación.
`--probe` ejecuta solicitudes en vivo (puede consumir tokens y activar límites de tasa).
Las filas de la sonda pueden provenir de perfiles de autenticación, credenciales de entorno o `models.json`.
Espera estados de sonda como `ok`, `auth`, `rate_limit`, `billing`, `timeout`,
`format`, `unknown` y `no_model`.
Cuando un `auth.order.<provider>` explícito omite un perfil almacenado, la sonda informa
`excluded_by_auth_order` en lugar de probar silenciosamente ese perfil.

### `models set <model>`

Establece `agents.defaults.model.primary`.

### `models set-image <model>`

Establece `agents.defaults.imageModel.primary`.

### `models aliases list|add|remove`

Opciones:

- `list`: `--json`, `--plain`
- `add <alias> <model>`
- `remove <alias>`

### `models fallbacks list|add|remove|clear`

Opciones:

- `list`: `--json`, `--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models image-fallbacks list|add|remove|clear`

Opciones:

- `list`: `--json`, `--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models scan`

Opciones:

- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>`
- `--concurrency <n>`
- `--no-probe`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

### `models auth add|login|login-github-copilot|setup-token|paste-token`

Opciones:

- `add`: ayuda interactiva de autenticación (flujo de autenticación del proveedor o pegado de token)
- `login`: `--provider <name>`, `--method <method>`, `--set-default`
- `login-github-copilot`: flujo de inicio de sesión OAuth de GitHub Copilot (`--yes`)
- `setup-token`: `--provider <name>`, `--yes`
- `paste-token`: `--provider <name>`, `--profile-id <id>`, `--expires-in <duration>`

Notas:

- `setup-token` y `paste-token` son comandos genéricos de token para proveedores que exponen métodos de autenticación por token.
- `setup-token` requiere un TTY interactivo y ejecuta el método de autenticación por token del proveedor.
- `paste-token` solicita el valor del token y usa de forma predeterminada el id de perfil de autenticación `<provider>:manual` cuando se omite `--profile-id`.
- Anthropic `setup-token` / `paste-token` vuelven a estar disponibles como ruta heredada/manual de OpenClaw. Anthropic informó a los usuarios de OpenClaw que esta ruta requiere **Extra Usage** en la cuenta Claude.

### `models auth order get|set|clear`

Opciones:

- `get`: `--provider <name>`, `--agent <id>`, `--json`
- `set`: `--provider <name>`, `--agent <id>`, `<profileIds...>`
- `clear`: `--provider <name>`, `--agent <id>`

## Sistema

### `system event`

Pone en cola un evento del sistema y opcionalmente activa un heartbeat (Gateway RPC).

Obligatorio:

- `--text <text>`

Opciones:

- `--mode <now|next-heartbeat>`
- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

### `system heartbeat last|enable|disable`

Controles de heartbeat (Gateway RPC).

Opciones:

- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

### `system presence`

Lista entradas de presencia del sistema (Gateway RPC).

Opciones:

- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

## Cron

Gestiona trabajos programados (Gateway RPC). Consulta [/automation/cron-jobs](/automation/cron-jobs).

Subcomandos:

- `cron status [--json]`
- `cron list [--all] [--json]` (salida en tabla de forma predeterminada; usa `--json` para salida sin procesar)
- `cron add` (alias: `create`; requiere `--name` y exactamente uno de `--at` | `--every` | `--cron`, y exactamente una carga de `--system-event` | `--message`)
- `cron edit <id>` (parchea campos)
- `cron rm <id>` (alias: `remove`, `delete`)
- `cron enable <id>`
- `cron disable <id>`
- `cron runs --id <id> [--limit <n>]`
- `cron run <id> [--due]`

Todos los comandos `cron` aceptan `--url`, `--token`, `--timeout`, `--expect-final`.

`cron add|edit --model ...` usa ese modelo permitido seleccionado para el trabajo. Si
el modelo no está permitido, cron avisa y recurre a la selección del modelo
del agente/predeterminado del trabajo. Las cadenas de respaldo configuradas siguen aplicándose, pero una simple
sustitución de modelo sin una lista explícita de respaldo por trabajo ya no agrega el modelo principal
del agente como destino adicional oculto de reintento.

## Host de nodo

### `node`

`node` ejecuta un **host de nodo sin interfaz** o lo gestiona como servicio en segundo plano. Consulta
[`openclaw node`](/cli/node).

Subcomandos:

- `node run --host <gateway-host> --port 18789`
- `node status`
- `node install [--host <gateway-host>] [--port <port>] [--tls] [--tls-fingerprint <sha256>] [--node-id <id>] [--display-name <name>] [--runtime <node|bun>] [--force]`
- `node uninstall`
- `node stop`
- `node restart`

Notas de autenticación:

- `node` resuelve la autenticación del gateway desde entorno/configuración (sin indicadores `--token`/`--password`): `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`, luego `gateway.auth.*`. En modo local, el host de nodo ignora intencionadamente `gateway.remote.*`; en `gateway.mode=remote`, `gateway.remote.*` participa según las reglas de prioridad remota.
- La resolución de autenticación del host de nodo solo respeta variables de entorno `OPENCLAW_GATEWAY_*`.

## Nodos

`nodes` se comunica con el Gateway y apunta a nodos emparejados. Consulta [/nodes](/nodes).

Opciones comunes:

- `--url`, `--token`, `--timeout`, `--json`

Subcomandos:

- `nodes status [--connected] [--last-connected <duration>]`
- `nodes describe --node <id|name|ip>`
- `nodes list [--connected] [--last-connected <duration>]`
- `nodes pending`
- `nodes approve <requestId>`
- `nodes reject <requestId>`
- `nodes rename --node <id|name|ip> --name <displayName>`
- `nodes invoke --node <id|name|ip> --command <command> [--params <json>] [--invoke-timeout <ms>] [--idempotency-key <key>]`
- `nodes notify --node <id|name|ip> [--title <text>] [--body <text>] [--sound <name>] [--priority <passive|active|timeSensitive>] [--delivery <system|overlay|auto>] [--invoke-timeout <ms>]` (solo mac)

Cámara:

- `nodes camera list --node <id|name|ip>`
- `nodes camera snap --node <id|name|ip> [--facing front|back|both] [--device-id <id>] [--max-width <px>] [--quality <0-1>] [--delay-ms <ms>] [--invoke-timeout <ms>]`
- `nodes camera clip --node <id|name|ip> [--facing front|back] [--device-id <id>] [--duration <ms|10s|1m>] [--no-audio] [--invoke-timeout <ms>]`

Canvas + pantalla:

- `nodes canvas snapshot --node <id|name|ip> [--format png|jpg|jpeg] [--max-width <px>] [--quality <0-1>] [--invoke-timeout <ms>]`
- `nodes canvas present --node <id|name|ip> [--target <urlOrPath>] [--x <px>] [--y <px>] [--width <px>] [--height <px>] [--invoke-timeout <ms>]`
- `nodes canvas hide --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas navigate <url> --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas eval [<js>] --node <id|name|ip> [--js <code>] [--invoke-timeout <ms>]`
- `nodes canvas a2ui push --node <id|name|ip> (--jsonl <path> | --text <text>) [--invoke-timeout <ms>]`
- `nodes canvas a2ui reset --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes screen record --node <id|name|ip> [--screen <index>] [--duration <ms|10s>] [--fps <n>] [--no-audio] [--out <path>] [--invoke-timeout <ms>]`

Ubicación:

- `nodes location get --node <id|name|ip> [--max-age <ms>] [--accuracy <coarse|balanced|precise>] [--location-timeout <ms>] [--invoke-timeout <ms>]`

## Browser

CLI de control del navegador (Chrome/Brave/Edge/Chromium dedicados). Consulta [`openclaw browser`](/cli/browser) y la [herramienta Browser](/tools/browser).

Opciones comunes:

- `--url`, `--token`, `--timeout`, `--expect-final`, `--json`
- `--browser-profile <name>`

Gestionar:

- `browser status`
- `browser start`
- `browser stop`
- `browser reset-profile`
- `browser tabs`
- `browser open <url>`
- `browser focus <targetId>`
- `browser close [targetId]`
- `browser profiles`
- `browser create-profile --name <name> [--color <hex>] [--cdp-url <url>] [--driver existing-session] [--user-data-dir <path>]`
- `browser delete-profile --name <name>`

Inspeccionar:

- `browser screenshot [targetId] [--full-page] [--ref <ref>] [--element <selector>] [--type png|jpeg]`
- `browser snapshot [--format aria|ai] [--target-id <id>] [--limit <n>] [--interactive] [--compact] [--depth <n>] [--selector <sel>] [--out <path>]`

Acciones:

- `browser navigate <url> [--target-id <id>]`
- `browser resize <width> <height> [--target-id <id>]`
- `browser click <ref> [--double] [--button <left|right|middle>] [--modifiers <csv>] [--target-id <id>]`
- `browser type <ref> <text> [--submit] [--slowly] [--target-id <id>]`
- `browser press <key> [--target-id <id>]`
- `browser hover <ref> [--target-id <id>]`
- `browser drag <startRef> <endRef> [--target-id <id>]`
- `browser select <ref> <values...> [--target-id <id>]`
- `browser upload <paths...> [--ref <ref>] [--input-ref <ref>] [--element <selector>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser fill [--fields <json>] [--fields-file <path>] [--target-id <id>]`
- `browser dialog --accept|--dismiss [--prompt <text>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser wait [--time <ms>] [--text <value>] [--text-gone <value>] [--target-id <id>]`
- `browser evaluate --fn <code> [--ref <ref>] [--target-id <id>]`
- `browser console [--level <error|warn|info>] [--target-id <id>]`
- `browser pdf [--target-id <id>]`

## Voice call

### `voicecall`

Utilidades de llamadas de voz proporcionadas por plugins. Solo aparece cuando el plugin de llamadas de voz está instalado y habilitado. Consulta [`openclaw voicecall`](/cli/voicecall).

Comandos comunes:

- `voicecall call --to <phone> --message <text> [--mode notify|conversation]`
- `voicecall start --to <phone> [--message <text>] [--mode notify|conversation]`
- `voicecall continue --call-id <id> --message <text>`
- `voicecall speak --call-id <id> --message <text>`
- `voicecall end --call-id <id>`
- `voicecall status --call-id <id>`
- `voicecall tail [--file <path>] [--since <n>] [--poll <ms>]`
- `voicecall latency [--file <path>] [--last <n>]`
- `voicecall expose [--mode off|serve|funnel] [--path <path>] [--port <port>] [--serve-path <path>]`

## Búsqueda de documentación

### `docs`

Busca en el índice en vivo de documentación de OpenClaw.

### `docs [query...]`

Busca en el índice en vivo de documentación.

## TUI

### `tui`

Abre la UI del terminal conectada al Gateway.

Opciones:

- `--url <url>`
- `--token <token>`
- `--password <password>`
- `--session <key>`
- `--deliver`
- `--thinking <level>`
- `--message <text>`
- `--timeout-ms <ms>` (predeterminado `agents.defaults.timeoutSeconds`)
- `--history-limit <n>`
