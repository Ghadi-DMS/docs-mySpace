---
read_when:
    - Agregar o modificar migraciones de Doctor
    - Introducir cambios de configuración incompatibles
summary: 'Comando Doctor: comprobaciones de estado, migraciones de configuración y pasos de reparación'
title: Doctor
x-i18n:
    generated_at: "2026-04-08T02:15:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3761a222d9db7088f78215575fa84e5896794ad701aa716e8bf9039a4424dca6
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` es la herramienta de reparación y migración de OpenClaw. Corrige
configuración/estado obsoletos, comprueba el estado y proporciona pasos de reparación accionables.

## Inicio rápido

```bash
openclaw doctor
```

### Sin interfaz / automatización

```bash
openclaw doctor --yes
```

Acepta los valores predeterminados sin solicitar confirmación (incluidos los pasos de reparación de reinicio/servicio/sandbox cuando corresponda).

```bash
openclaw doctor --repair
```

Aplica las reparaciones recomendadas sin solicitar confirmación (reparaciones + reinicios cuando sea seguro).

```bash
openclaw doctor --repair --force
```

Aplica también reparaciones agresivas (sobrescribe configuraciones personalizadas del supervisor).

```bash
openclaw doctor --non-interactive
```

Se ejecuta sin solicitudes y solo aplica migraciones seguras (normalización de configuración + movimientos de estado en disco). Omite acciones de reinicio/servicio/sandbox que requieren confirmación humana.
Las migraciones de estado heredado se ejecutan automáticamente cuando se detectan.

```bash
openclaw doctor --deep
```

Analiza los servicios del sistema para detectar instalaciones adicionales del gateway (launchd/systemd/schtasks).

Si quiere revisar los cambios antes de escribirlos, abra primero el archivo de configuración:

```bash
cat ~/.openclaw/openclaw.json
```

## Qué hace (resumen)

- Actualización previa opcional para instalaciones de git (solo interactivo).
- Comprobación de actualización del protocolo de la UI (reconstruye Control UI cuando el esquema del protocolo es más reciente).
- Comprobación de estado + solicitud de reinicio.
- Resumen del estado de Skills (aptas/faltantes/bloqueadas) y estado de plugins.
- Normalización de configuración para valores heredados.
- Migración de la configuración de Talk desde los campos planos heredados `talk.*` a `talk.provider` + `talk.providers.<provider>`.
- Comprobaciones de migración del navegador para configuraciones heredadas de la extensión de Chrome y preparación de Chrome MCP.
- Advertencias de sobrescritura del proveedor OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Advertencias de enmascaramiento de OAuth de Codex (`models.providers.openai-codex`).
- Comprobación de requisitos previos de TLS para perfiles OAuth de OpenAI Codex.
- Migración de estado heredado en disco (sesiones/directorio de agente/autenticación de WhatsApp).
- Migración de claves de contrato heredadas del manifiesto de plugins (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migración del almacén cron heredado (`jobId`, `schedule.cron`, campos de entrega/carga útil de nivel superior, `provider` en la carga útil, trabajos de respaldo webhook simples con `notify: true`).
- Inspección de archivos de bloqueo de sesión y limpieza de bloqueos obsoletos.
- Comprobaciones de integridad y permisos del estado (sesiones, transcripciones, directorio de estado).
- Comprobaciones de permisos del archivo de configuración (`chmod 600`) cuando se ejecuta localmente.
- Estado de autenticación de modelos: comprueba el vencimiento de OAuth, puede renovar tokens próximos a vencer y reporta estados de enfriamiento/deshabilitación de perfiles de autenticación.
- Detección de directorio de espacio de trabajo adicional (`~/openclaw`).
- Reparación de imagen de sandbox cuando el aislamiento está habilitado.
- Migración de servicios heredados y detección de gateways adicionales.
- Migración de estado heredado del canal Matrix (en modo `--fix` / `--repair`).
- Comprobaciones del tiempo de ejecución del gateway (servicio instalado pero no en ejecución; etiqueta launchd en caché).
- Advertencias de estado de canales (sondeadas desde el gateway en ejecución).
- Auditoría de configuración del supervisor (launchd/systemd/schtasks) con reparación opcional.
- Comprobaciones de buenas prácticas del tiempo de ejecución del gateway (Node frente a Bun, rutas de gestores de versiones).
- Diagnóstico de colisiones de puerto del gateway (predeterminado `18789`).
- Advertencias de seguridad para políticas de DM abiertas.
- Comprobaciones de autenticación del gateway para el modo de token local (ofrece generar un token cuando no existe una fuente de token; no sobrescribe configuraciones de token con SecretRef).
- Comprobación de linger de systemd en Linux.
- Comprobación del tamaño de archivos de arranque del espacio de trabajo (advertencias de truncamiento o cercanía al límite para archivos de contexto).
- Comprobación del estado del autocompletado del shell e instalación/actualización automática.
- Comprobación de preparación del proveedor de embeddings para búsqueda en memoria (modelo local, clave de API remota o binario QMD).
- Comprobaciones de instalación desde código fuente (desajuste del espacio de trabajo pnpm, recursos de UI faltantes, binario tsx faltante).
- Escribe la configuración actualizada + metadatos del asistente.

## Comportamiento detallado y fundamento

### 0) Actualización opcional (instalaciones de git)

Si esto es una extracción de git y doctor se está ejecutando de forma interactiva, ofrece
actualizar (fetch/rebase/build) antes de ejecutar doctor.

### 1) Normalización de configuración

Si la configuración contiene formas de valores heredados (por ejemplo `messages.ackReaction`
sin una sobrescritura específica del canal), doctor los normaliza al esquema actual.

Eso incluye los campos planos heredados de Talk. La configuración pública actual de Talk es
`talk.provider` + `talk.providers.<provider>`. Doctor reescribe las formas antiguas
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` en el mapa de proveedores.

### 2) Migraciones de claves de configuración heredadas

Cuando la configuración contiene claves obsoletas, otros comandos se niegan a ejecutarse y piden
que ejecute `openclaw doctor`.

Doctor hará lo siguiente:

- Explicar qué claves heredadas se encontraron.
- Mostrar la migración que aplicó.
- Reescribir `~/.openclaw/openclaw.json` con el esquema actualizado.

El Gateway también ejecuta automáticamente las migraciones de doctor al iniciarse cuando detecta un
formato de configuración heredado, por lo que las configuraciones obsoletas se reparan sin intervención manual.
Las migraciones del almacén de trabajos cron se gestionan con `openclaw doctor --fix`.

Migraciones actuales:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` de nivel superior
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` heredados → `talk.provider` + `talk.providers.<provider>`
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
- Para canales con `accounts` con nombre pero con valores de canal de cuenta única aún presentes en el nivel superior, mover esos valores con alcance de cuenta a la cuenta promovida elegida para ese canal (`accounts.default` para la mayoría de los canales; Matrix puede conservar un destino coincidente con nombre/predeterminado existente)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- eliminar `browser.relayBindHost` (configuración heredada del relay de la extensión)

Las advertencias de Doctor también incluyen orientación sobre cuentas predeterminadas para canales con varias cuentas:

- Si se configuran dos o más entradas `channels.<channel>.accounts` sin `channels.<channel>.defaultAccount` o `accounts.default`, doctor advierte que el enrutamiento de respaldo puede elegir una cuenta inesperada.
- Si `channels.<channel>.defaultAccount` está configurado con un ID de cuenta desconocido, doctor advierte y enumera los ID de cuenta configurados.

### 2b) Sobrescrituras del proveedor OpenCode

Si agregó manualmente `models.providers.opencode`, `opencode-zen` u `opencode-go`,
sobrescribe el catálogo integrado de OpenCode de `@mariozechner/pi-ai`.
Eso puede forzar modelos a la API incorrecta o dejar los costos en cero. Doctor advierte para que
pueda eliminar la sobrescritura y restaurar el enrutamiento por modelo + costos.

### 2c) Migración del navegador y preparación de Chrome MCP

Si su configuración del navegador aún apunta a la ruta eliminada de la extensión de Chrome, doctor
la normaliza al modelo actual de conexión de Chrome MCP local al host:

- `browser.profiles.*.driver: "extension"` pasa a ser `"existing-session"`
- se elimina `browser.relayBindHost`

Doctor también audita la ruta de Chrome MCP local al host cuando usa `defaultProfile:
"user"` o un perfil `existing-session` configurado:

- comprueba si Google Chrome está instalado en el mismo host para perfiles predeterminados
  de conexión automática
- comprueba la versión detectada de Chrome y advierte cuando es inferior a Chrome 144
- recuerda habilitar la depuración remota en la página de inspección del navegador (por
  ejemplo `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  o `edge://inspect/#remote-debugging`)

Doctor no puede habilitar la configuración del lado de Chrome por usted. El Chrome MCP
local al host aún requiere:

- un navegador basado en Chromium 144+ en el host gateway/nodo
- el navegador ejecutándose localmente
- depuración remota habilitada en ese navegador
- aprobar la primera solicitud de consentimiento de conexión en el navegador

La preparación aquí solo se refiere a los requisitos previos de conexión local. Existing-session mantiene
los límites de ruta actuales de Chrome MCP; las rutas avanzadas como `responsebody`, exportación PDF,
intercepción de descargas y acciones por lotes siguen requiriendo un navegador gestionado
o un perfil CDP sin procesar.

Esta comprobación **no** se aplica a Docker, sandbox, remote-browser ni a otros
flujos sin interfaz. Esos siguen usando CDP sin procesar.

### 2d) Requisitos previos de TLS para OAuth

Cuando se configura un perfil OAuth de OpenAI Codex, doctor sondea el endpoint de autorización de OpenAI
para verificar que la pila TLS local de Node/OpenSSL pueda validar la cadena de certificados. Si el sondeo falla con un error de certificado (por
ejemplo `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, certificado vencido o certificado autofirmado),
doctor muestra orientación de corrección específica para cada plataforma. En macOS con Node de Homebrew, la
corrección suele ser `brew postinstall ca-certificates`. Con `--deep`, el sondeo se ejecuta
incluso si el gateway está en buen estado.

### 2c) Sobrescrituras del proveedor OAuth de Codex

Si anteriormente agregó configuraciones heredadas de transporte OpenAI en
`models.providers.openai-codex`, pueden enmascarar la ruta integrada del
proveedor OAuth de Codex que las versiones más recientes usan automáticamente. Doctor advierte cuando ve
esas configuraciones antiguas de transporte junto con OAuth de Codex para que pueda eliminar o reescribir
la sobrescritura de transporte obsoleta y recuperar el comportamiento integrado de enrutamiento/respaldo.
Los proxies personalizados y las sobrescrituras solo de encabezados siguen siendo compatibles y no
activan esta advertencia.

### 3) Migraciones de estado heredado (diseño en disco)

Doctor puede migrar diseños antiguos en disco a la estructura actual:

- Almacén de sesiones + transcripciones:
  - de `~/.openclaw/sessions/` a `~/.openclaw/agents/<agentId>/sessions/`
- Directorio del agente:
  - de `~/.openclaw/agent/` a `~/.openclaw/agents/<agentId>/agent/`
- Estado de autenticación de WhatsApp (Baileys):
  - desde `~/.openclaw/credentials/*.json` heredado (excepto `oauth.json`)
  - hacia `~/.openclaw/credentials/whatsapp/<accountId>/...` (id de cuenta predeterminada: `default`)

Estas migraciones son de mejor esfuerzo e idempotentes; doctor emitirá advertencias cuando
deje carpetas heredadas como copias de seguridad. El Gateway/CLI también migra automáticamente
las sesiones heredadas + el directorio del agente al iniciarse, de modo que el historial/autenticación/modelos
terminen en la ruta por agente sin ejecutar doctor manualmente. La autenticación de WhatsApp se migra intencionalmente solo
mediante `openclaw doctor`. La normalización del proveedor/mapa de proveedores de Talk ahora
compara por igualdad estructural, por lo que las diferencias solo en el orden de las claves ya no activan
cambios repetidos de `doctor --fix` sin efecto.

### 3a) Migraciones heredadas del manifiesto de plugins

Doctor analiza todos los manifiestos de plugins instalados para detectar claves obsoletas de capacidades
de nivel superior (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Cuando las encuentra, ofrece moverlas al objeto `contracts`
y reescribir el archivo de manifiesto in situ. Esta migración es idempotente;
si la clave `contracts` ya tiene los mismos valores, la clave heredada se elimina
sin duplicar los datos.

### 3b) Migraciones heredadas del almacén cron

Doctor también comprueba el almacén de trabajos cron (`~/.openclaw/cron/jobs.json` de forma predeterminada,
o `cron.store` cuando se sobrescribe) para detectar formas antiguas de trabajos que el planificador aún
acepta por compatibilidad.

Las limpiezas cron actuales incluyen:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- campos de carga útil de nivel superior (`message`, `model`, `thinking`, ...) → `payload`
- campos de entrega de nivel superior (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias de entrega `provider` en la carga útil → `delivery.channel` explícito
- trabajos heredados simples de respaldo webhook con `notify: true` → `delivery.mode="webhook"` explícito con `delivery.to=cron.webhook`

Doctor solo migra automáticamente trabajos `notify: true` cuando puede hacerlo sin
cambiar el comportamiento. Si un trabajo combina el respaldo heredado de notify con un modo de entrega
existente no webhook, doctor advierte y deja ese trabajo para revisión manual.

### 3c) Limpieza de bloqueos de sesión

Doctor analiza cada directorio de sesión de agente para detectar archivos de bloqueo de escritura obsoletos:
archivos que quedaron atrás cuando una sesión salió de forma anómala. Para cada archivo de bloqueo encontrado informa:
la ruta, el PID, si el PID sigue activo, la antigüedad del bloqueo y si se
considera obsoleto (PID muerto o más de 30 minutos). En modo `--fix` / `--repair`
elimina automáticamente los archivos de bloqueo obsoletos; en caso contrario, muestra una nota y
le indica volver a ejecutar con `--fix`.

### 4) Comprobaciones de integridad del estado (persistencia de sesión, enrutamiento y seguridad)

El directorio de estado es el tronco encefálico operativo. Si desaparece, pierde
sesiones, credenciales, registros y configuración (a menos que tenga copias de seguridad en otro lugar).

Doctor comprueba:

- **Falta el directorio de estado**: advierte sobre una pérdida catastrófica del estado, solicita volver a crear
  el directorio y recuerda que no puede recuperar los datos faltantes.
- **Permisos del directorio de estado**: verifica que se pueda escribir; ofrece reparar permisos
  (y emite una sugerencia de `chown` cuando detecta discrepancia de propietario/grupo).
- **Directorio de estado sincronizado en la nube de macOS**: advierte cuando el estado se resuelve bajo iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) o
  `~/Library/CloudStorage/...` porque las rutas respaldadas por sincronización pueden causar E/S más lentas
  y condiciones de carrera de bloqueo/sincronización.
- **Directorio de estado en SD o eMMC de Linux**: advierte cuando el estado se resuelve a una fuente de montaje `mmcblk*`,
  porque la E/S aleatoria respaldada por SD o eMMC puede ser más lenta y desgastarse
  más rápido con las escrituras de sesión y credenciales.
- **Faltan directorios de sesión**: `sessions/` y el directorio del almacén de sesiones son
  necesarios para persistir el historial y evitar fallos `ENOENT`.
- **Desajuste de transcripción**: advierte cuando las entradas recientes de sesión tienen
  archivos de transcripción faltantes.
- **Sesión principal con “JSONL de 1 línea”**: marca cuando la transcripción principal solo tiene una
  línea (el historial no se está acumulando).
- **Varios directorios de estado**: advierte cuando existen varias carpetas `~/.openclaw` entre
  directorios de inicio o cuando `OPENCLAW_STATE_DIR` apunta a otro lugar (el historial puede
  dividirse entre instalaciones).
- **Recordatorio de modo remoto**: si `gateway.mode=remote`, doctor recuerda ejecutarlo en
  el host remoto (el estado vive allí).
- **Permisos del archivo de configuración**: advierte si `~/.openclaw/openclaw.json` es
  legible por grupo/mundo y ofrece restringirlo a `600`.

### 5) Estado de autenticación de modelos (vencimiento de OAuth)

Doctor inspecciona los perfiles OAuth en el almacén de autenticación, advierte cuando los tokens están
próximos a vencer o vencidos, y puede renovarlos cuando es seguro. Si el perfil OAuth/token
de Anthropic está obsoleto, sugiere una clave de API de Anthropic o la
ruta de token de configuración de Anthropic.
Las solicitudes de renovación solo aparecen cuando se ejecuta de forma interactiva (TTY); `--non-interactive`
omite los intentos de renovación.

Doctor también informa perfiles de autenticación que están temporalmente inutilizables debido a:

- enfriamientos breves (límites de velocidad/tiempos de espera/fallos de autenticación)
- deshabilitaciones más largas (fallos de facturación/crédito)

### 6) Validación del modelo de hooks

Si `hooks.gmail.model` está configurado, doctor valida la referencia del modelo con el
catálogo y la lista de permitidos y advierte cuando no se resolverá o no está permitido.

### 7) Reparación de imagen de sandbox

Cuando el aislamiento está habilitado, doctor comprueba las imágenes de Docker y ofrece compilar o
cambiar a nombres heredados si falta la imagen actual.

### 7b) Dependencias de tiempo de ejecución de plugins integrados

Doctor verifica que las dependencias de tiempo de ejecución de plugins integrados (por ejemplo, los
paquetes de tiempo de ejecución del plugin de Discord) estén presentes en la raíz de instalación de OpenClaw.
Si falta alguna, doctor informa los paquetes y los instala en
modo `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migraciones de servicios del gateway y sugerencias de limpieza

Doctor detecta servicios heredados del gateway (launchd/systemd/schtasks) y
ofrece eliminarlos e instalar el servicio de OpenClaw usando el puerto actual del gateway.
También puede buscar servicios adicionales similares al gateway y mostrar sugerencias de limpieza.
Los servicios de gateway de OpenClaw con nombre de perfil se consideran de primera clase y no
se marcan como “adicionales”.

### 8b) Migración de Matrix al inicio

Cuando una cuenta del canal Matrix tiene una migración de estado heredado pendiente o procesable,
doctor (en modo `--fix` / `--repair`) crea una instantánea previa a la migración y luego
ejecuta los pasos de migración de mejor esfuerzo: migración del estado heredado de Matrix y preparación
del estado cifrado heredado. Ambos pasos no son fatales; los errores se registran y el
inicio continúa. En modo de solo lectura (`openclaw doctor` sin `--fix`) esta comprobación
se omite por completo.

### 9) Advertencias de seguridad

Doctor emite advertencias cuando un proveedor está abierto a DM sin una lista de permitidos, o
cuando una política está configurada de forma peligrosa.

### 10) Linger de systemd (Linux)

Si se ejecuta como servicio de usuario de systemd, doctor asegura que linger esté habilitado para que el
gateway siga activo después de cerrar sesión.

### 11) Estado del espacio de trabajo (Skills, plugins y directorios heredados)

Doctor muestra un resumen del estado del espacio de trabajo para el agente predeterminado:

- **Estado de Skills**: cuenta las Skills aptas, con requisitos faltantes y bloqueadas por lista de permitidos.
- **Directorios heredados del espacio de trabajo**: advierte cuando `~/openclaw` u otros directorios heredados del espacio de trabajo
  existen junto al espacio de trabajo actual.
- **Estado de plugins**: cuenta plugins cargados/deshabilitados/con error; enumera los ID de plugins con
  errores; informa las capacidades de plugins integrados.
- **Advertencias de compatibilidad de plugins**: marca plugins que tienen problemas de compatibilidad con
  el tiempo de ejecución actual.
- **Diagnóstico de plugins**: muestra cualquier advertencia o error en tiempo de carga emitido por el
  registro de plugins.

### 11b) Tamaño del archivo de arranque

Doctor comprueba si los archivos de arranque del espacio de trabajo (por ejemplo `AGENTS.md`,
`CLAUDE.md` u otros archivos de contexto inyectados) están cerca o por encima del presupuesto de
caracteres configurado. Informa por archivo los recuentos de caracteres sin procesar frente a inyectados, el
porcentaje de truncamiento, la causa del truncamiento (`max/file` o `max/total`) y el total de caracteres inyectados
como fracción del presupuesto total. Cuando los archivos están truncados o cerca
del límite, doctor muestra consejos para ajustar `agents.defaults.bootstrapMaxChars`
y `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Autocompletado del shell

Doctor comprueba si el autocompletado con tabulador está instalado para el shell actual
(zsh, bash, fish o PowerShell):

- Si el perfil del shell usa un patrón lento de autocompletado dinámico
  (`source <(openclaw completion ...)`), doctor lo actualiza a la variante más rápida
  de archivo en caché.
- Si el autocompletado está configurado en el perfil pero falta el archivo de caché,
  doctor regenera automáticamente la caché.
- Si no hay ningún autocompletado configurado, doctor solicita instalarlo
  (solo en modo interactivo; se omite con `--non-interactive`).

Ejecute `openclaw completion --write-state` para regenerar manualmente la caché.

### 12) Comprobaciones de autenticación del gateway (token local)

Doctor comprueba la preparación de la autenticación por token del gateway local.

- Si el modo de token necesita un token y no existe una fuente de token, doctor ofrece generar uno.
- Si `gateway.auth.token` está gestionado por SecretRef pero no está disponible, doctor advierte y no lo sobrescribe con texto sin formato.
- `openclaw doctor --generate-gateway-token` fuerza la generación solo cuando no hay ningún token SecretRef configurado.

### 12b) Reparaciones de solo lectura con reconocimiento de SecretRef

Algunos flujos de reparación necesitan inspeccionar las credenciales configuradas sin debilitar el comportamiento de fallo rápido del tiempo de ejecución.

- `openclaw doctor --fix` ahora usa el mismo modelo resumido de SecretRef de solo lectura que la familia de comandos de estado para reparaciones de configuración específicas.
- Ejemplo: la reparación de `@username` en Telegram para `allowFrom` / `groupAllowFrom` intenta usar credenciales del bot configuradas cuando están disponibles.
- Si el token del bot de Telegram está configurado mediante SecretRef pero no está disponible en la ruta actual del comando, doctor informa que la credencial está configurada-pero-no-disponible y omite la resolución automática en lugar de fallar o informar erróneamente que falta el token.

### 13) Comprobación de estado del gateway + reinicio

Doctor ejecuta una comprobación de estado y ofrece reiniciar el gateway cuando parece
no estar en buen estado.

### 13b) Preparación de búsqueda en memoria

Doctor comprueba si el proveedor de embeddings configurado para búsqueda en memoria está preparado
para el agente predeterminado. El comportamiento depende del backend y del proveedor configurados:

- **Backend QMD**: sondea si el binario `qmd` está disponible y puede iniciarse.
  Si no, muestra orientación de corrección, incluido el paquete npm y una opción manual de ruta binaria.
- **Proveedor local explícito**: comprueba si existe un archivo de modelo local o una URL
  reconocida de modelo remoto/descargable. Si falta, sugiere cambiar a un proveedor remoto.
- **Proveedor remoto explícito** (`openai`, `voyage`, etc.): verifica que haya una clave de API
  presente en el entorno o en el almacén de autenticación. Muestra sugerencias de corrección accionables si falta.
- **Proveedor automático**: comprueba primero la disponibilidad de modelos locales y luego prueba cada proveedor remoto
  en el orden de selección automática.

Cuando hay disponible un resultado de sondeo del gateway (el gateway estaba en buen estado en el momento de la
comprobación), doctor cruza ese resultado con la configuración visible desde la CLI y señala
cualquier discrepancia.

Use `openclaw memory status --deep` para verificar la preparación de embeddings en tiempo de ejecución.

### 14) Advertencias de estado de canales

Si el gateway está en buen estado, doctor ejecuta un sondeo de estado de los canales e informa
advertencias con correcciones sugeridas.

### 15) Auditoría + reparación de configuración del supervisor

Doctor comprueba si la configuración del supervisor instalado (launchd/systemd/schtasks) tiene
valores predeterminados faltantes u obsoletos (por ejemplo, dependencias de systemd network-online y
retraso de reinicio). Cuando encuentra una discrepancia, recomienda una actualización y puede
reescribir el archivo de servicio/tarea a los valores predeterminados actuales.

Notas:

- `openclaw doctor` solicita confirmación antes de reescribir la configuración del supervisor.
- `openclaw doctor --yes` acepta las solicitudes de reparación predeterminadas.
- `openclaw doctor --repair` aplica las correcciones recomendadas sin solicitudes.
- `openclaw doctor --repair --force` sobrescribe configuraciones personalizadas del supervisor.
- Si la autenticación por token requiere un token y `gateway.auth.token` está gestionado por SecretRef, la instalación/reparación del servicio de doctor valida el SecretRef, pero no persiste valores resueltos de token en texto sin formato en los metadatos de entorno del servicio supervisor.
- Si la autenticación por token requiere un token y el token SecretRef configurado no está resuelto, doctor bloquea la ruta de instalación/reparación con orientación accionable.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está definido, doctor bloquea la instalación/reparación hasta que el modo se establezca explícitamente.
- Para unidades user-systemd de Linux, las comprobaciones de deriva de token de doctor ahora incluyen tanto fuentes `Environment=` como `EnvironmentFile=` al comparar metadatos de autenticación del servicio.
- Siempre puede forzar una reescritura completa mediante `openclaw gateway install --force`.

### 16) Diagnóstico del tiempo de ejecución del gateway + puerto

Doctor inspecciona el tiempo de ejecución del servicio (PID, último estado de salida) y advierte cuando el
servicio está instalado pero en realidad no se está ejecutando. También comprueba colisiones
de puerto en el puerto del gateway (predeterminado `18789`) e informa causas probables (gateway ya
en ejecución, túnel SSH).

### 17) Buenas prácticas del tiempo de ejecución del gateway

Doctor advierte cuando el servicio del gateway se ejecuta con Bun o con una ruta de Node gestionada por versiones
(`nvm`, `fnm`, `volta`, `asdf`, etc.). Los canales de WhatsApp + Telegram requieren Node,
y las rutas de gestores de versiones pueden fallar después de actualizaciones porque el servicio no
carga la inicialización de su shell. Doctor ofrece migrar a una instalación de Node del sistema cuando
está disponible (Homebrew/apt/choco).

### 18) Escritura de configuración + metadatos del asistente

Doctor conserva cualquier cambio de configuración y marca metadatos del asistente para registrar la
ejecución de doctor.

### 19) Consejos del espacio de trabajo (copia de seguridad + sistema de memoria)

Doctor sugiere un sistema de memoria del espacio de trabajo cuando falta y muestra un consejo de copia de seguridad
si el espacio de trabajo todavía no está bajo git.

Consulte [/concepts/agent-workspace](/es/concepts/agent-workspace) para ver una guía completa sobre
la estructura del espacio de trabajo y la copia de seguridad con git (se recomienda GitHub o GitLab privados).
