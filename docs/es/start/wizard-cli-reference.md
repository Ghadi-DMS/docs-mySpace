---
read_when:
    - Necesitas conocer en detalle el comportamiento de `openclaw onboard`
    - Estás depurando resultados de onboarding o integrando clientes de onboarding
sidebarTitle: CLI reference
summary: Referencia completa del flujo de configuración de la CLI, la configuración de autenticación/modelos, salidas e internals
title: Referencia de configuración de la CLI
x-i18n:
    generated_at: "2026-04-05T12:55:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ec4e685e3237e450d11c45826c2bb34b82c0bba1162335f8fbb07f51ba00a70
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# Referencia de configuración de la CLI

Esta página es la referencia completa de `openclaw onboard`.
Para la guía breve, consulta [Onboarding (CLI)](/es/start/wizard).

## Qué hace el asistente

El modo local (predeterminado) te guía a través de:

- Configuración de modelo y autenticación (OAuth de suscripción a OpenAI Code, Anthropic Claude CLI o clave de API, además de opciones de MiniMax, GLM, Ollama, Moonshot, StepFun y AI Gateway)
- Ubicación del espacio de trabajo y archivos de bootstrap
- Configuración del gateway (puerto, bind, autenticación, tailscale)
- Canales y proveedores (Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, BlueBubbles y otros plugins de canales incluidos)
- Instalación del daemon (LaunchAgent, unidad de usuario de systemd o Scheduled Task nativa de Windows con alternativa de carpeta Inicio)
- Comprobación de estado
- Configuración de Skills

El modo remoto configura esta máquina para conectarse a un gateway en otro lugar.
No instala ni modifica nada en el host remoto.

## Detalles del flujo local

<Steps>
  <Step title="Detección de configuración existente">
    - Si existe `~/.openclaw/openclaw.json`, elige entre Conservar, Modificar o Restablecer.
    - Volver a ejecutar el asistente no borra nada a menos que elijas explícitamente Restablecer (o pases `--reset`).
    - La CLI `--reset` usa de forma predeterminada `config+creds+sessions`; usa `--reset-scope full` para eliminar también el espacio de trabajo.
    - Si la configuración no es válida o contiene claves heredadas, el asistente se detiene y te pide que ejecutes `openclaw doctor` antes de continuar.
    - El restablecimiento usa `trash` y ofrece estos alcances:
      - Solo configuración
      - Configuración + credenciales + sesiones
      - Restablecimiento completo (también elimina el espacio de trabajo)
  </Step>
  <Step title="Modelo y autenticación">
    - La matriz completa de opciones está en [Opciones de autenticación y modelo](#auth-and-model-options).
  </Step>
  <Step title="Espacio de trabajo">
    - Valor predeterminado `~/.openclaw/workspace` (configurable).
    - Genera los archivos del espacio de trabajo necesarios para el ritual de bootstrap de primera ejecución.
    - Estructura del espacio de trabajo: [Espacio de trabajo del agente](/es/concepts/agent-workspace).
  </Step>
  <Step title="Gateway">
    - Solicita puerto, bind, modo de autenticación y exposición de tailscale.
    - Recomendación: mantén habilitada la autenticación por token incluso para loopback, para que los clientes WS locales deban autenticarse.
    - En modo token, la configuración interactiva ofrece:
      - **Generar/almacenar token en texto plano** (predeterminado)
      - **Usar SecretRef** (opcional)
    - En modo contraseña, la configuración interactiva también admite almacenamiento en texto plano o SecretRef.
    - Ruta no interactiva de SecretRef para token: `--gateway-token-ref-env <ENV_VAR>`.
      - Requiere una variable de entorno no vacía en el entorno del proceso de onboarding.
      - No puede combinarse con `--gateway-token`.
    - Desactiva la autenticación solo si confías plenamente en todos los procesos locales.
    - Los binds que no son loopback siguen requiriendo autenticación.
  </Step>
  <Step title="Canales">
    - [WhatsApp](/es/channels/whatsapp): inicio de sesión por QR opcional
    - [Telegram](/es/channels/telegram): token del bot
    - [Discord](/es/channels/discord): token del bot
    - [Google Chat](/es/channels/googlechat): JSON de cuenta de servicio + audiencia del webhook
    - [Mattermost](/es/channels/mattermost): token del bot + URL base
    - [Signal](/es/channels/signal): instalación opcional de `signal-cli` + configuración de cuenta
    - [BlueBubbles](/es/channels/bluebubbles): recomendado para iMessage; URL del servidor + contraseña + webhook
    - [iMessage](/es/channels/imessage): ruta heredada de la CLI `imsg` + acceso a la BD
    - Seguridad en mensajes directos: el valor predeterminado es el emparejamiento. El primer mensaje directo envía un código; apruébalo mediante
      `openclaw pairing approve <channel> <code>` o usa listas de permitidos.
  </Step>
  <Step title="Instalación del daemon">
    - macOS: LaunchAgent
      - Requiere una sesión de usuario iniciada; para modo sin interfaz, usa un LaunchDaemon personalizado (no incluido).
    - Linux y Windows mediante WSL2: unidad de usuario de systemd
      - El asistente intenta `loginctl enable-linger <user>` para que el gateway siga activo después del cierre de sesión.
      - Puede solicitar sudo (escribe en `/var/lib/systemd/linger`); primero lo intenta sin sudo.
    - Windows nativo: primero Scheduled Task
      - Si se deniega la creación de la tarea, OpenClaw recurre a un elemento de inicio de sesión por usuario en la carpeta Inicio e inicia el gateway de inmediato.
      - Las Scheduled Tasks siguen siendo la opción preferida porque proporcionan mejor estado del supervisor.
    - Selección de entorno de ejecución: Node (recomendado; obligatorio para WhatsApp y Telegram). Bun no se recomienda.
  </Step>
  <Step title="Comprobación de estado">
    - Inicia el gateway (si es necesario) y ejecuta `openclaw health`.
    - `openclaw status --deep` añade la sonda de estado del gateway en vivo a la salida de estado, incluidas las sondas de canal cuando están disponibles.
  </Step>
  <Step title="Skills">
    - Lee las Skills disponibles y comprueba los requisitos.
    - Te permite elegir el gestor de Node: npm, pnpm o bun.
    - Instala dependencias opcionales (algunas usan Homebrew en macOS).
  </Step>
  <Step title="Finalizar">
    - Resumen y pasos siguientes, incluidas las opciones para la app de iOS, Android y macOS.
  </Step>
</Steps>

<Note>
Si no se detecta ninguna GUI, el asistente imprime instrucciones de reenvío de puertos por SSH para la interfaz de control en lugar de abrir un navegador.
Si faltan los recursos de la interfaz de control, el asistente intenta compilarlos; la alternativa es `pnpm ui:build` (instala automáticamente las dependencias de la UI).
</Note>

## Detalles del modo remoto

El modo remoto configura esta máquina para conectarse a un gateway en otro lugar.

<Info>
El modo remoto no instala ni modifica nada en el host remoto.
</Info>

Lo que configuras:

- URL del gateway remoto (`ws://...`)
- Token si el gateway remoto requiere autenticación (recomendado)

<Note>
- Si el gateway es solo loopback, usa túneles SSH o una tailnet.
- Pistas de detección:
  - macOS: Bonjour (`dns-sd`)
  - Linux: Avahi (`avahi-browse`)
</Note>

## Opciones de autenticación y modelo

<AccordionGroup>
  <Accordion title="Clave de API de Anthropic">
    Usa `ANTHROPIC_API_KEY` si está presente o solicita una clave, y luego la guarda para el uso del daemon.
  </Accordion>
  <Accordion title="Anthropic Claude CLI">
    Reutiliza un inicio de sesión local de Claude CLI en el host del gateway y cambia la
    selección del modelo a una referencia canónica `claude-cli/claude-*`.

    Esta es una ruta de reserva local disponible en `openclaw onboard` y
    `openclaw configure`. Para producción, se prefiere una clave de API de Anthropic.

    - macOS: comprueba el elemento del llavero "Claude Code-credentials"
    - Linux y Windows: reutiliza `~/.claude/.credentials.json` si está presente

    En macOS, elige "Always Allow" para que los inicios con launchd no se bloqueen.

  </Accordion>
  <Accordion title="Suscripción a OpenAI Code (reutilización de Codex CLI)">
    Si existe `~/.codex/auth.json`, el asistente puede reutilizarlo.
    Las credenciales reutilizadas de Codex CLI siguen siendo administradas por Codex CLI; cuando caducan, OpenClaw
    vuelve a leer primero esa fuente y, cuando el proveedor puede renovarlas, escribe
    la credencial renovada de nuevo en el almacenamiento de Codex en lugar de asumir su control
    directamente.
  </Accordion>
  <Accordion title="Suscripción a OpenAI Code (OAuth)">
    Flujo en navegador; pega `code#state`.

    Establece `agents.defaults.model` en `openai-codex/gpt-5.4` cuando el modelo no está configurado o es `openai/*`.

  </Accordion>
  <Accordion title="Clave de API de OpenAI">
    Usa `OPENAI_API_KEY` si está presente o solicita una clave, y luego almacena la credencial en perfiles de autenticación.

    Establece `agents.defaults.model` en `openai/gpt-5.4` cuando el modelo no está configurado, es `openai/*` o `openai-codex/*`.

  </Accordion>
  <Accordion title="Clave de API de xAI (Grok)">
    Solicita `XAI_API_KEY` y configura xAI como proveedor de modelos.
  </Accordion>
  <Accordion title="OpenCode">
    Solicita `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`) y te permite elegir el catálogo Zen o Go.
    URL de configuración: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="Clave de API (genérica)">
    Almacena la clave por ti.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    Solicita `AI_GATEWAY_API_KEY`.
    Más detalles: [Vercel AI Gateway](/es/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    Solicita el ID de cuenta, el ID del gateway y `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    Más detalles: [Cloudflare AI Gateway](/es/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    La configuración se escribe automáticamente. El valor predeterminado alojado es `MiniMax-M2.7`; la configuración con clave de API usa
    `minimax/...`, y la configuración con OAuth usa `minimax-portal/...`.
    Más detalles: [MiniMax](/es/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    La configuración se escribe automáticamente para StepFun estándar o Step Plan en endpoints de China o globales.
    Actualmente, Standard incluye `step-3.5-flash`, y Step Plan también incluye `step-3.5-flash-2603`.
    Más detalles: [StepFun](/es/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (compatible con Anthropic)">
    Solicita `SYNTHETIC_API_KEY`.
    Más detalles: [Synthetic](/es/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (Cloud y modelos abiertos locales)">
    Solicita la URL base (predeterminada `http://127.0.0.1:11434`), y luego ofrece modo Cloud + Local o Local.
    Descubre los modelos disponibles y sugiere valores predeterminados.
    Más detalles: [Ollama](/es/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot y Kimi Coding">
    Las configuraciones de Moonshot (Kimi K2) y Kimi Coding se escriben automáticamente.
    Más detalles: [Moonshot AI (Kimi + Kimi Coding)](/es/providers/moonshot).
  </Accordion>
  <Accordion title="Proveedor personalizado">
    Funciona con endpoints compatibles con OpenAI y con Anthropic.

    El onboarding interactivo admite las mismas opciones de almacenamiento de claves de API que otros flujos de claves de API de proveedores:
    - **Pegar clave de API ahora** (texto plano)
    - **Usar referencia a secreto** (referencia de entorno o referencia de proveedor configurada, con validación previa)

    Flags no interactivos:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (opcional; usa `CUSTOM_API_KEY` como alternativa)
    - `--custom-provider-id` (opcional)
    - `--custom-compatibility <openai|anthropic>` (opcional; el valor predeterminado es `openai`)

  </Accordion>
  <Accordion title="Omitir">
    Deja la autenticación sin configurar.
  </Accordion>
</AccordionGroup>

Comportamiento del modelo:

- Elige el modelo predeterminado entre las opciones detectadas o introduce el proveedor y el modelo manualmente.
- Cuando el onboarding comienza desde una opción de autenticación de proveedor, el selector de modelos prioriza
  ese proveedor automáticamente. Para Volcengine y BytePlus, esta misma preferencia
  también coincide con sus variantes de plan de codificación (`volcengine-plan/*`,
  `byteplus-plan/*`).
- Si ese filtro de proveedor preferido quedara vacío, el selector vuelve al
  catálogo completo en lugar de no mostrar modelos.
- El asistente ejecuta una comprobación del modelo y avisa si el modelo configurado es desconocido o le falta autenticación.

Rutas de credenciales y perfiles:

- Perfiles de autenticación (claves de API + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Importación heredada de OAuth: `~/.openclaw/credentials/oauth.json`

Modo de almacenamiento de credenciales:

- El comportamiento predeterminado del onboarding persiste las claves de API como valores en texto plano en los perfiles de autenticación.
- `--secret-input-mode ref` habilita el modo de referencia en lugar del almacenamiento en texto plano de claves.
  En la configuración interactiva, puedes elegir entre:
  - referencia de variable de entorno (por ejemplo `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - referencia de proveedor configurado (`file` o `exec`) con alias de proveedor + id
- El modo de referencia interactivo ejecuta una validación previa rápida antes de guardar.
  - Referencias de entorno: valida el nombre de la variable + un valor no vacío en el entorno actual del onboarding.
  - Referencias de proveedor: valida la configuración del proveedor y resuelve el id solicitado.
  - Si la validación previa falla, el onboarding muestra el error y te permite reintentar.
- En modo no interactivo, `--secret-input-mode ref` solo usa respaldo en variables de entorno.
  - Configura la variable de entorno del proveedor en el entorno del proceso de onboarding.
  - Los flags de clave en línea (por ejemplo `--openai-api-key`) requieren que esa variable de entorno esté configurada; de lo contrario, el onboarding falla rápidamente.
  - Para proveedores personalizados, el modo `ref` no interactivo almacena `models.providers.<id>.apiKey` como `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - En ese caso de proveedor personalizado, `--custom-api-key` requiere que `CUSTOM_API_KEY` esté configurada; de lo contrario, el onboarding falla rápidamente.
- Las credenciales de autenticación del gateway admiten opciones de texto plano y SecretRef en la configuración interactiva:
  - Modo token: **Generar/almacenar token en texto plano** (predeterminado) o **Usar SecretRef**.
  - Modo contraseña: texto plano o SecretRef.
- Ruta no interactiva de SecretRef para token: `--gateway-token-ref-env <ENV_VAR>`.
- Las configuraciones existentes en texto plano siguen funcionando sin cambios.

<Note>
Consejo para modo sin interfaz y servidores: completa OAuth en una máquina con navegador y luego copia
el `auth-profiles.json` de ese agente (por ejemplo
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, o la ruta coincidente de
`$OPENCLAW_STATE_DIR/...`) al host del gateway. `credentials/oauth.json`
es solo una fuente heredada de importación.
</Note>

## Salidas e internals

Campos habituales en `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (si se eligió Minimax)
- `tools.profile` (el onboarding local usa de forma predeterminada `"coding"` cuando no está configurado; los valores explícitos existentes se conservan)
- `gateway.*` (mode, bind, auth, tailscale)
- `session.dmScope` (el onboarding local lo establece de forma predeterminada en `per-channel-peer` cuando no está configurado; los valores explícitos existentes se conservan)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Listas de permitidos de canales (Slack, Discord, Matrix, Microsoft Teams) cuando las activas durante las indicaciones (los nombres se resuelven a IDs cuando es posible)
- `skills.install.nodeManager`
  - El flag `setup --node-manager` acepta `npm`, `pnpm` o `bun`.
  - La configuración manual todavía puede establecer `skills.install.nodeManager: "yarn"` más adelante.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` escribe en `agents.list[]` y `bindings` opcionales.

Las credenciales de WhatsApp se almacenan en `~/.openclaw/credentials/whatsapp/<accountId>/`.
Las sesiones se almacenan en `~/.openclaw/agents/<agentId>/sessions/`.

<Note>
Algunos canales se entregan como plugins. Cuando se seleccionan durante la configuración, el asistente
solicita instalar el plugin (npm o ruta local) antes de la configuración del canal.
</Note>

RPC del asistente de gateway:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

Los clientes (app de macOS e interfaz de control) pueden renderizar los pasos sin volver a implementar la lógica de onboarding.

Comportamiento de la configuración de Signal:

- Descarga el recurso de la versión correspondiente
- Lo almacena en `~/.openclaw/tools/signal-cli/<version>/`
- Escribe `channels.signal.cliPath` en la configuración
- Las compilaciones JVM requieren Java 21
- Se usan compilaciones nativas cuando están disponibles
- Windows usa WSL2 y sigue el flujo de Linux para signal-cli dentro de WSL

## Documentación relacionada

- Hub de onboarding: [Onboarding (CLI)](/es/start/wizard)
- Automatización y scripts: [Automatización de CLI](/start/wizard-cli-automation)
- Referencia de comandos: [`openclaw onboard`](/cli/onboard)
