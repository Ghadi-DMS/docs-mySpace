---
read_when:
    - Agregar o modificar migraciones de doctor
    - Introducir cambios incompatibles de configuración
summary: 'Comando Doctor: comprobaciones de estado, migraciones de configuración y pasos de reparación'
title: Doctor
x-i18n:
    generated_at: "2026-04-09T01:29:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 75d321bd1ad0e16c29f2382e249c51edfc3a8d33b55bdceea39e7dbcd4901fce
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` es la herramienta de reparación + migración de OpenClaw. Corrige
configuración/estado obsoletos, comprueba el estado y proporciona pasos de
reparación accionables.

## Inicio rápido

```bash
openclaw doctor
```

### Modo sin interfaz / automatización

```bash
openclaw doctor --yes
```

Acepta los valores predeterminados sin preguntar (incluidos los pasos de reparación de reinicio/servicio/sandbox cuando corresponda).

```bash
openclaw doctor --repair
```

Aplica las reparaciones recomendadas sin preguntar (reparaciones + reinicios cuando sea seguro).

```bash
openclaw doctor --repair --force
```

Aplica también reparaciones agresivas (sobrescribe configuraciones personalizadas del supervisor).

```bash
openclaw doctor --non-interactive
```

Se ejecuta sin preguntas y solo aplica migraciones seguras (normalización de configuración + movimientos de estado en disco). Omite acciones de reinicio/servicio/sandbox que requieren confirmación humana.
Las migraciones de estado heredado se ejecutan automáticamente cuando se detectan.

```bash
openclaw doctor --deep
```

Analiza los servicios del sistema para detectar instalaciones adicionales del gateway (launchd/systemd/schtasks).

Si quieres revisar los cambios antes de escribir, abre primero el archivo de configuración:

```bash
cat ~/.openclaw/openclaw.json
```

## Qué hace (resumen)

- Actualización previa opcional para instalaciones de git (solo interactivo).
- Comprobación de vigencia del protocolo de IU (reconstruye la Control UI cuando el esquema del protocolo es más reciente).
- Comprobación de estado + aviso para reiniciar.
- Resumen del estado de Skills (aptas/faltantes/bloqueadas) y estado de los plugins.
- Normalización de configuración para valores heredados.
- Migración de configuración de Talk desde campos planos heredados `talk.*` hacia `talk.provider` + `talk.providers.<provider>`.
- Comprobaciones de migración del navegador para configuraciones heredadas de la extensión de Chrome y preparación de Chrome MCP.
- Advertencias de anulaciones del proveedor OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Advertencias de sombreado de OAuth de Codex (`models.providers.openai-codex`).
- Comprobación de requisitos previos de TLS de OAuth para perfiles de OAuth de OpenAI Codex.
- Migración heredada de estado en disco (sesiones/directorio del agente/autenticación de WhatsApp).
- Migración heredada de claves de contrato en manifiestos de plugins (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migración heredada del almacén cron (`jobId`, `schedule.cron`, campos de entrega/carga útil de nivel superior, `provider` en la carga útil, trabajos simples de respaldo webhook con `notify: true`).
- Inspección de archivos de bloqueo de sesión y limpieza de bloqueos obsoletos.
- Comprobaciones de integridad y permisos del estado (sesiones, transcripciones, directorio de estado).
- Comprobaciones de permisos del archivo de configuración (`chmod 600`) cuando se ejecuta localmente.
- Estado de autenticación del modelo: comprueba vencimiento de OAuth, puede actualizar tokens próximos a vencer e informa de estados de enfriamiento/desactivación del perfil de autenticación.
- Detección de directorio de espacio de trabajo adicional (`~/openclaw`).
- Reparación de imagen de sandbox cuando el sandbox está habilitado.
- Migración de servicio heredado y detección de gateways adicionales.
- Migración heredada del estado del canal Matrix (en modo `--fix` / `--repair`).
- Comprobaciones del runtime del gateway (servicio instalado pero no en ejecución; etiqueta launchd en caché).
- Advertencias de estado de canales (sondeadas desde el gateway en ejecución).
- Auditoría de configuración del supervisor (launchd/systemd/schtasks) con reparación opcional.
- Comprobaciones de buenas prácticas del runtime del gateway (Node frente a Bun, rutas de gestores de versiones).
- Diagnósticos de colisión de puertos del gateway (predeterminado `18789`).
- Advertencias de seguridad para políticas de MD abiertas.
- Comprobaciones de autenticación del gateway para modo de token local (ofrece generar un token cuando no existe una fuente de token; no sobrescribe configuraciones de token SecretRef).
- Comprobación de systemd linger en Linux.
- Comprobación del tamaño de archivos bootstrap del espacio de trabajo (advertencias por truncamiento/cercanía al límite para archivos de contexto).
- Comprobación del estado de autocompletado del shell e instalación/actualización automática.
- Comprobación de preparación del proveedor de embeddings para búsqueda en memoria (modelo local, clave de API remota o binario QMD).
- Comprobaciones de instalación desde código fuente (desajuste de espacio de trabajo pnpm, recursos de IU faltantes, binario tsx faltante).
- Escribe la configuración actualizada + metadatos del asistente.

## Backfill y restablecimiento de la IU de Dreams

La escena Dreams de la Control UI incluye acciones de **Backfill**, **Reset** y **Clear Grounded**
para el flujo de trabajo de sueños fundamentados. Estas acciones usan métodos RPC
de estilo doctor del gateway, pero **no** forman parte de la reparación/migración
de la CLI `openclaw doctor`.

Qué hacen:

- **Backfill** analiza archivos históricos `memory/YYYY-MM-DD.md` en el espacio
  de trabajo activo, ejecuta el proceso fundamentado del diario REM y escribe
  entradas reversibles de backfill en `DREAMS.md`.
- **Reset** elimina solo esas entradas del diario de backfill marcadas de `DREAMS.md`.
- **Clear Grounded** elimina solo las entradas preparadas a corto plazo de tipo
  grounded-only que provinieron de una reproducción histórica y que aún no han
  acumulado recuperación en vivo ni soporte diario.

Qué **no** hacen por sí solas:

- no editan `MEMORY.md`
- no ejecutan migraciones completas de doctor
- no preparan automáticamente candidatos fundamentados en el almacén de promoción
  activa a corto plazo a menos que ejecutes explícitamente primero la ruta de la CLI preparada

Si quieres que la reproducción histórica fundamentada influya en la vía normal
de promoción profunda, usa en su lugar el flujo de CLI:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

Eso prepara candidatos duraderos fundamentados en el almacén de sueños a corto plazo,
manteniendo `DREAMS.md` como la superficie de revisión.

## Comportamiento detallado y justificación

### 0) Actualización opcional (instalaciones de git)

Si esto es un checkout de git y doctor se ejecuta de forma interactiva, ofrece
actualizar (fetch/rebase/build) antes de ejecutar doctor.

### 1) Normalización de configuración

Si la configuración contiene formas heredadas de valores (por ejemplo `messages.ackReaction`
sin una anulación específica por canal), doctor las normaliza al esquema actual.

Eso incluye campos planos heredados de Talk. La configuración pública actual de Talk es
`talk.provider` + `talk.providers.<provider>`. Doctor reescribe las formas antiguas
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` en el mapa del proveedor.

### 2) Migraciones de claves heredadas de configuración

Cuando la configuración contiene claves obsoletas, otros comandos se niegan a ejecutarse y te piden
que ejecutes `openclaw doctor`.

Doctor hará lo siguiente:

- Explicar qué claves heredadas se encontraron.
- Mostrar la migración que aplicó.
- Reescribir `~/.openclaw/openclaw.json` con el esquema actualizado.

El Gateway también ejecuta automáticamente las migraciones de doctor al iniciarse cuando detecta
un formato heredado de configuración, de modo que las configuraciones obsoletas se reparan sin intervención manual.
Las migraciones del almacén de trabajos cron se gestionan con `openclaw doctor --fix`.

Migraciones actuales:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` de nivel superior
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- heredado `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Para canales con `accounts` con nombre pero con valores persistentes de canal de cuenta única en el nivel superior, mover esos valores con alcance de cuenta a la cuenta promovida elegida para ese canal (`accounts.default` para la mayoría de los canales; Matrix puede conservar un destino con nombre/predeterminado coincidente ya existente)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- eliminar `browser.relayBindHost` (configuración heredada del relay de la extensión)

Las advertencias de doctor también incluyen orientación sobre cuentas predeterminadas para canales con varias cuentas:

- Si se configuran dos o más entradas `channels.<channel>.accounts` sin `channels.<channel>.defaultAccount` ni `accounts.default`, doctor advierte que el enrutamiento de respaldo puede elegir una cuenta inesperada.
- Si `channels.<channel>.defaultAccount` está establecido en un ID de cuenta desconocido, doctor advierte y enumera los ID de cuenta configurados.

### 2b) Anulaciones del proveedor OpenCode

Si agregaste manualmente `models.providers.opencode`, `opencode-zen` o `opencode-go`,
eso sobrescribe el catálogo integrado de OpenCode de `@mariozechner/pi-ai`.
Eso puede forzar modelos a la API equivocada o poner los costos a cero. Doctor advierte
para que puedas eliminar la anulación y restaurar el enrutamiento por modelo + costos.

### 2c) Migración del navegador y preparación de Chrome MCP

Si tu configuración del navegador aún apunta a la ruta eliminada de la extensión de Chrome, doctor
la normaliza al modelo actual de conexión host-local de Chrome MCP:

- `browser.profiles.*.driver: "extension"` pasa a ser `"existing-session"`
- se elimina `browser.relayBindHost`

Doctor también audita la ruta host-local de Chrome MCP cuando usas `defaultProfile:
"user"` o un perfil `existing-session` configurado:

- comprueba si Google Chrome está instalado en el mismo host para perfiles de
  conexión automática predeterminados
- comprueba la versión detectada de Chrome y advierte cuando es inferior a Chrome 144
- recuerda habilitar la depuración remota en la página de inspección del navegador (por
  ejemplo `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  o `edge://inspect/#remote-debugging`)

Doctor no puede habilitar la configuración del lado de Chrome por ti. Chrome MCP host-local
todavía requiere:

- un navegador basado en Chromium 144+ en el host del gateway/nodo
- el navegador ejecutándose localmente
- depuración remota habilitada en ese navegador
- aprobar el primer aviso de consentimiento de conexión en el navegador

La preparación aquí se refiere solo a los requisitos previos de conexión local. Existing-session mantiene
los límites actuales de rutas de Chrome MCP; rutas avanzadas como `responsebody`, exportación
a PDF, interceptación de descargas y acciones por lotes siguen requiriendo un
navegador gestionado o un perfil CDP sin procesar.

Esta comprobación **no** se aplica a Docker, sandbox, remote-browser ni otros
flujos sin interfaz. Esos siguen usando CDP sin procesar.

### 2d) Requisitos previos de TLS para OAuth

Cuando se configura un perfil de OAuth de OpenAI Codex, doctor sondea el endpoint de autorización de OpenAI
para verificar que la pila TLS local de Node/OpenSSL pueda validar la cadena de certificados.
Si el sondeo falla con un error de certificado (por
ejemplo `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, certificado vencido o certificado autofirmado),
doctor imprime orientación de corrección específica de la plataforma. En macOS con un Node de Homebrew, la
corrección suele ser `brew postinstall ca-certificates`. Con `--deep`, el sondeo se ejecuta
incluso si el gateway está en buen estado.

### 2c) Anulaciones del proveedor OAuth de Codex

Si anteriormente agregaste configuraciones heredadas de transporte de OpenAI en
`models.providers.openai-codex`, estas pueden ocultar la ruta integrada del
proveedor OAuth de Codex que las versiones más recientes usan automáticamente. Doctor
advierte cuando ve esas configuraciones antiguas de transporte junto con OAuth de Codex
para que puedas eliminar o reescribir la anulación obsoleta de transporte y recuperar
el comportamiento integrado de enrutamiento/respaldo. Los proxies personalizados y las anulaciones
solo de encabezados siguen siendo compatibles y no activan esta advertencia.

### 3) Migraciones heredadas de estado (disposición en disco)

Doctor puede migrar disposiciones antiguas en disco a la estructura actual:

- Almacén de sesiones + transcripciones:
  - de `~/.openclaw/sessions/` a `~/.openclaw/agents/<agentId>/sessions/`
- Directorio del agente:
  - de `~/.openclaw/agent/` a `~/.openclaw/agents/<agentId>/agent/`
- Estado de autenticación de WhatsApp (Baileys):
  - desde `~/.openclaw/credentials/*.json` heredado (excepto `oauth.json`)
  - a `~/.openclaw/credentials/whatsapp/<accountId>/...` (ID de cuenta predeterminado: `default`)

Estas migraciones son por mejor esfuerzo e idempotentes; doctor emitirá advertencias cuando
deje carpetas heredadas como copias de seguridad. El Gateway/CLI también migra automáticamente
las sesiones heredadas + el directorio del agente al iniciarse, para que el historial/autenticación/modelos
terminen en la ruta por agente sin una ejecución manual de doctor. La autenticación de WhatsApp se migra
intencionalmente solo mediante `openclaw doctor`. La normalización de proveedor/mapa de proveedores de Talk ahora
compara por igualdad estructural, por lo que las diferencias solo en el orden de claves ya no provocan
cambios repetidos sin efecto de `doctor --fix`.

### 3a) Migraciones heredadas de manifiestos de plugins

Doctor analiza todos los manifiestos de plugins instalados en busca de claves
obsoletas de capacidades de nivel superior (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Cuando las encuentra, ofrece moverlas al objeto `contracts`
y reescribir el archivo del manifiesto in situ. Esta migración es idempotente;
si la clave `contracts` ya tiene los mismos valores, la clave heredada se elimina
sin duplicar los datos.

### 3b) Migraciones heredadas del almacén cron

Doctor también comprueba el almacén de trabajos cron (`~/.openclaw/cron/jobs.json` de forma predeterminada,
o `cron.store` si se ha sobrescrito) en busca de formas antiguas de trabajos que el planificador aún
acepta por compatibilidad.

Las limpiezas actuales de cron incluyen:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- campos de carga útil de nivel superior (`message`, `model`, `thinking`, ...) → `payload`
- campos de entrega de nivel superior (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias de entrega `provider` en la carga útil → `delivery.channel` explícito
- trabajos heredados simples de respaldo webhook con `notify: true` → `delivery.mode="webhook"` explícito con `delivery.to=cron.webhook`

Doctor solo migra automáticamente trabajos `notify: true` cuando puede hacerlo sin
cambiar el comportamiento. Si un trabajo combina un respaldo notify heredado con un modo de
entrega existente no webhook, doctor advierte y deja ese trabajo para revisión manual.

### 3c) Limpieza de bloqueos de sesión

Doctor analiza cada directorio de sesiones de agente en busca de archivos obsoletos de bloqueo de escritura:
archivos que quedan cuando una sesión salió de forma anómala. Para cada archivo de bloqueo encontrado informa:
la ruta, el PID, si el PID sigue activo, la antigüedad del bloqueo y si se
considera obsoleto (PID muerto o con más de 30 minutos). En modo `--fix` / `--repair`
elimina automáticamente los archivos de bloqueo obsoletos; de lo contrario imprime una nota e
indica que lo vuelvas a ejecutar con `--fix`.

### 4) Comprobaciones de integridad del estado (persistencia de sesión, enrutamiento y seguridad)

El directorio de estado es el tronco operativo principal. Si desaparece, pierdes
sesiones, credenciales, registros y configuración (a menos que tengas copias de seguridad en otro lugar).

Doctor comprueba:

- **Falta el directorio de estado**: advierte sobre pérdida catastrófica de estado, ofrece recrear
  el directorio y recuerda que no puede recuperar datos faltantes.
- **Permisos del directorio de estado**: verifica que sea escribible; ofrece reparar permisos
  (y emite una sugerencia de `chown` cuando detecta desajuste de propietario/grupo).
- **Directorio de estado sincronizado en la nube en macOS**: advierte cuando el estado se resuelve bajo iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) o
  `~/Library/CloudStorage/...` porque las rutas respaldadas por sincronización pueden causar E/S más lentas
  y condiciones de carrera de bloqueo/sincronización.
- **Directorio de estado en SD o eMMC en Linux**: advierte cuando el estado se resuelve a un origen de montaje `mmcblk*`,
  porque la E/S aleatoria respaldada por SD o eMMC puede ser más lenta y desgastarse más rápido con escrituras de sesiones y credenciales.
- **Faltan directorios de sesión**: `sessions/` y el directorio del almacén de sesiones son
  necesarios para conservar el historial y evitar fallos `ENOENT`.
- **Desajuste de transcripciones**: advierte cuando entradas recientes de sesión tienen archivos
  de transcripción faltantes.
- **Sesión principal “JSONL de 1 línea”**: marca cuando la transcripción principal tiene solo una
  línea (el historial no se está acumulando).
- **Varios directorios de estado**: advierte cuando existen varias carpetas `~/.openclaw` entre
  directorios personales o cuando `OPENCLAW_STATE_DIR` apunta a otro sitio (el historial puede dividirse entre instalaciones).
- **Recordatorio de modo remoto**: si `gateway.mode=remote`, doctor recuerda ejecutarlo en
  el host remoto (el estado vive allí).
- **Permisos del archivo de configuración**: advierte si `~/.openclaw/openclaw.json` es
  legible por grupo/mundo y ofrece restringirlo a `600`.

### 5) Estado de autenticación del modelo (vencimiento de OAuth)

Doctor inspecciona perfiles OAuth en el almacén de autenticación, advierte cuando los tokens
están por vencer o vencidos, y puede actualizarlos cuando sea seguro. Si el perfil
de OAuth/token de Anthropic está obsoleto, sugiere una clave de API de Anthropic o la
ruta de setup-token de Anthropic.
Los avisos de actualización solo aparecen cuando se ejecuta de forma interactiva (TTY); `--non-interactive`
omite los intentos de actualización.

Cuando una actualización de OAuth falla de forma permanente (por ejemplo `refresh_token_reused`,
`invalid_grant` o un proveedor te indica que vuelvas a iniciar sesión), doctor informa
que se requiere reautenticación e imprime el comando exacto `openclaw models auth login --provider ...`
que debes ejecutar.

Doctor también informa de perfiles de autenticación temporalmente inutilizables debido a:

- enfriamientos cortos (límites de tasa/tiempos de espera/fallos de autenticación)
- desactivaciones más largas (fallos de facturación/crédito)

### 6) Validación del modelo de Hooks

Si `hooks.gmail.model` está configurado, doctor valida la referencia del modelo frente al
catálogo y la lista de permitidos y advierte cuando no se resolverá o no está permitido.

### 7) Reparación de imagen de sandbox

Cuando el sandbox está habilitado, doctor comprueba las imágenes de Docker y ofrece compilar o
cambiar a nombres heredados si falta la imagen actual.

### 7b) Dependencias de runtime de plugins incluidos

Doctor verifica que las dependencias de runtime de los plugins incluidos (por ejemplo los
paquetes de runtime del plugin de Discord) estén presentes en la raíz de instalación de OpenClaw.
Si falta alguna, doctor informa de los paquetes y los instala en modo
`openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migraciones de servicios del gateway y sugerencias de limpieza

Doctor detecta servicios heredados del gateway (launchd/systemd/schtasks) y
ofrece eliminarlos e instalar el servicio de OpenClaw usando el puerto actual del gateway.
También puede buscar servicios adicionales similares al gateway e imprimir sugerencias de limpieza.
Los servicios de gateway de OpenClaw con nombre de perfil se consideran de primera clase y no se
marcan como "adicionales".

### 8b) Migración de Matrix al iniciar

Cuando una cuenta del canal Matrix tiene una migración heredada de estado pendiente o aplicable,
doctor (en modo `--fix` / `--repair`) crea una instantánea previa a la migración y luego
ejecuta los pasos de migración por mejor esfuerzo: migración heredada del estado de Matrix
y preparación heredada del estado cifrado. Ambos pasos no son fatales; los errores se registran y
el inicio continúa. En modo de solo lectura (`openclaw doctor` sin `--fix`) esta comprobación
se omite por completo.

### 9) Advertencias de seguridad

Doctor emite advertencias cuando un proveedor está abierto a MD sin lista de permitidos, o
cuando una política está configurada de forma peligrosa.

### 10) systemd linger (Linux)

Si se ejecuta como servicio de usuario systemd, doctor se asegura de que lingering esté habilitado para que el
gateway siga activo después del cierre de sesión.

### 11) Estado del espacio de trabajo (Skills, plugins y directorios heredados)

Doctor imprime un resumen del estado del espacio de trabajo para el agente predeterminado:

- **Estado de Skills**: cuenta Skills aptas, con requisitos faltantes y bloqueadas por la lista de permitidos.
- **Directorios heredados del espacio de trabajo**: advierte cuando existen `~/openclaw` u otros directorios heredados del espacio de trabajo
  junto al espacio de trabajo actual.
- **Estado de plugins**: cuenta plugins cargados/deshabilitados/con errores; enumera los ID de plugin para cualquier
  error; informa de las capacidades de los plugins del bundle.
- **Advertencias de compatibilidad de plugins**: marca plugins que tienen problemas de compatibilidad con
  el runtime actual.
- **Diagnósticos de plugins**: muestra cualquier advertencia o error de carga emitido por el
  registro de plugins.

### 11b) Tamaño del archivo bootstrap

Doctor comprueba si los archivos bootstrap del espacio de trabajo (por ejemplo `AGENTS.md`,
`CLAUDE.md` u otros archivos de contexto inyectados) están cerca o por encima del presupuesto
configurado de caracteres. Informa por archivo del recuento de caracteres sin procesar frente al inyectado, el porcentaje
de truncamiento, la causa del truncamiento (`max/file` o `max/total`) y el total de caracteres
inyectados como fracción del presupuesto total. Cuando los archivos están truncados o cerca
del límite, doctor imprime sugerencias para ajustar `agents.defaults.bootstrapMaxChars`
y `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Autocompletado del shell

Doctor comprueba si el autocompletado con tabulador está instalado para el shell actual
(zsh, bash, fish o PowerShell):

- Si el perfil del shell usa un patrón lento de autocompletado dinámico
  (`source <(openclaw completion ...)`), doctor lo actualiza a la variante más rápida
  con archivo en caché.
- Si el autocompletado está configurado en el perfil pero falta el archivo de caché,
  doctor regenera la caché automáticamente.
- Si no hay ningún autocompletado configurado, doctor ofrece instalarlo
  (solo en modo interactivo; se omite con `--non-interactive`).

Ejecuta `openclaw completion --write-state` para regenerar la caché manualmente.

### 12) Comprobaciones de autenticación del gateway (token local)

Doctor comprueba la preparación de autenticación por token del gateway local.

- Si el modo token necesita un token y no existe una fuente de token, doctor ofrece generar uno.
- Si `gateway.auth.token` está gestionado por SecretRef pero no disponible, doctor advierte y no lo sobrescribe con texto sin formato.
- `openclaw doctor --generate-gateway-token` fuerza la generación solo cuando no hay un token SecretRef configurado.

### 12b) Reparaciones de solo lectura compatibles con SecretRef

Algunos flujos de reparación necesitan inspeccionar credenciales configuradas sin debilitar el comportamiento fail-fast del runtime.

- `openclaw doctor --fix` ahora usa el mismo modelo resumido de SecretRef de solo lectura que los comandos de la familia status para reparaciones específicas de configuración.
- Ejemplo: la reparación de Telegram `allowFrom` / `groupAllowFrom` con `@username` intenta usar credenciales configuradas del bot cuando están disponibles.
- Si el token del bot de Telegram está configurado mediante SecretRef pero no está disponible en la ruta actual del comando, doctor informa que la credencial está configurada pero no disponible y omite la resolución automática en lugar de fallar o informar incorrectamente que falta el token.

### 13) Comprobación de estado del gateway + reinicio

Doctor ejecuta una comprobación de estado y ofrece reiniciar el gateway cuando parece
no estar en buen estado.

### 13b) Preparación de la búsqueda en memoria

Doctor comprueba si el proveedor de embeddings configurado para búsqueda en memoria está listo
para el agente predeterminado. El comportamiento depende del backend y del proveedor configurados:

- **Backend QMD**: sondea si el binario `qmd` está disponible y puede iniciarse.
  Si no, imprime orientación para corregirlo, incluido el paquete npm y una opción manual de ruta al binario.
- **Proveedor local explícito**: comprueba si hay un archivo de modelo local o una URL de modelo remota/descargable reconocida. Si falta, sugiere cambiar a un proveedor remoto.
- **Proveedor remoto explícito** (`openai`, `voyage`, etc.): verifica que haya una clave de API
  presente en el entorno o en el almacén de autenticación. Imprime sugerencias de corrección accionables si falta.
- **Proveedor automático**: comprueba primero la disponibilidad del modelo local y luego prueba cada
  proveedor remoto en orden de selección automática.

Cuando hay disponible un resultado del sondeo del gateway (el gateway estaba en buen estado en el momento de la
comprobación), doctor lo cruza con la configuración visible desde la CLI y señala
cualquier discrepancia.

Usa `openclaw memory status --deep` para verificar en runtime la preparación de embeddings.

### 14) Advertencias de estado de los canales

Si el gateway está en buen estado, doctor ejecuta un sondeo de estado de canales e informa
advertencias con correcciones sugeridas.

### 15) Auditoría + reparación de configuración del supervisor

Doctor comprueba la configuración instalada del supervisor (launchd/systemd/schtasks) en busca de
valores predeterminados ausentes o desactualizados (p. ej., dependencias de systemd network-online y
retraso de reinicio). Cuando encuentra una discrepancia, recomienda una actualización y puede
reescribir el archivo de servicio/tarea a los valores predeterminados actuales.

Notas:

- `openclaw doctor` pregunta antes de reescribir la configuración del supervisor.
- `openclaw doctor --yes` acepta los avisos de reparación predeterminados.
- `openclaw doctor --repair` aplica las correcciones recomendadas sin preguntas.
- `openclaw doctor --repair --force` sobrescribe configuraciones personalizadas del supervisor.
- Si la autenticación por token requiere un token y `gateway.auth.token` está gestionado por SecretRef, la instalación/reparación del servicio valida el SecretRef pero no conserva valores de token resueltos en texto sin formato en los metadatos del entorno del servicio del supervisor.
- Si la autenticación por token requiere un token y el token SecretRef configurado no está resuelto, doctor bloquea la ruta de instalación/reparación con orientación accionable.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está establecido, doctor bloquea la instalación/reparación hasta que el modo se configure explícitamente.
- Para unidades user-systemd en Linux, las comprobaciones de divergencia del token de doctor ahora incluyen tanto las fuentes `Environment=` como `EnvironmentFile=` al comparar metadatos de autenticación del servicio.
- Siempre puedes forzar una reescritura completa mediante `openclaw gateway install --force`.

### 16) Diagnósticos del runtime del gateway + puerto

Doctor inspecciona el runtime del servicio (PID, último estado de salida) y advierte cuando el
servicio está instalado pero en realidad no se está ejecutando. También comprueba colisiones
de puertos en el puerto del gateway (predeterminado `18789`) e informa causas probables (gateway ya
en ejecución, túnel SSH).

### 17) Buenas prácticas del runtime del gateway

Doctor advierte cuando el servicio del gateway se ejecuta con Bun o una ruta de Node gestionada por versiones
(`nvm`, `fnm`, `volta`, `asdf`, etc.). Los canales de WhatsApp + Telegram requieren Node,
y las rutas de gestores de versiones pueden fallar después de actualizaciones porque el servicio no
carga la inicialización de tu shell. Doctor ofrece migrar a una instalación de Node del sistema cuando
está disponible (Homebrew/apt/choco).

### 18) Escritura de configuración + metadatos del asistente

Doctor guarda cualquier cambio de configuración y registra metadatos del asistente para anotar la ejecución
de doctor.

### 19) Sugerencias del espacio de trabajo (copia de seguridad + sistema de memoria)

Doctor sugiere un sistema de memoria para el espacio de trabajo cuando falta y muestra una sugerencia de copia de seguridad
si el espacio de trabajo aún no está bajo git.

Consulta [/concepts/agent-workspace](/es/concepts/agent-workspace) para obtener una guía completa sobre la
estructura del espacio de trabajo y la copia de seguridad con git (se recomienda GitHub o GitLab privados).
