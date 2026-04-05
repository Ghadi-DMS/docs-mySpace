---
read_when:
    - Buscar ejemplos reales de uso de OpenClaw
    - Actualizar proyectos destacados de la comunidad
summary: Proyectos e integraciones creados por la comunidad con OpenClaw
title: Showcase
x-i18n:
    generated_at: "2026-04-05T12:54:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2917e9a476ef527ddb3e51c610bbafbd145e705c9cc29f191639fb63d238ef70
    source_path: start/showcase.md
    workflow: 15
---

# Showcase

Proyectos reales de la comunidad. Mira lo que la gente está creando con OpenClaw.

<Info>
**¿Quieres aparecer aquí?** Comparte tu proyecto en [#self-promotion en Discord](https://discord.gg/clawd) o [menciona a @openclaw en X](https://x.com/openclaw).
</Info>

## 🎥 OpenClaw en acción

Recorrido completo de configuración (28 min) por VelvetShark.

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/SaWSPZoPX34"
    title="OpenClaw: The self-hosted AI that Siri should have been (Full setup)"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[Ver en YouTube](https://www.youtube.com/watch?v=SaWSPZoPX34)

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/mMSKQvlmFuQ"
    title="Video de showcase de OpenClaw"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[Ver en YouTube](https://www.youtube.com/watch?v=mMSKQvlmFuQ)

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/5kkIJNUGFho"
    title="Showcase de la comunidad de OpenClaw"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[Ver en YouTube](https://www.youtube.com/watch?v=5kkIJNUGFho)

## 🆕 Recién salido de Discord

<CardGroup cols={2}>

<Card title="Revisión de PR → comentarios en Telegram" icon="code-pull-request" href="https://x.com/i/status/2010878524543131691">
  **@bangnokia** • `review` `github` `telegram`

OpenCode termina el cambio → abre un PR → OpenClaw revisa el diff y responde en Telegram con “sugerencias menores” además de un veredicto claro de fusión (incluidas las correcciones críticas que deben aplicarse primero).

  <img src="/assets/showcase/pr-review-telegram.jpg" alt="Comentarios de revisión de PR de OpenClaw entregados en Telegram" />
</Card>

<Card title="Skill de bodega de vinos en minutos" icon="wine-glass" href="https://x.com/i/status/2010916352454791216">
  **@prades_maxime** • `skills` `local` `csv`

Le pidió a “Robby” (@openclaw) una skill local para una bodega de vinos. Solicita un ejemplo de exportación CSV + dónde almacenarlo, y luego crea/prueba la skill rápidamente (962 botellas en el ejemplo).

  <img src="/assets/showcase/wine-cellar-skill.jpg" alt="OpenClaw creando una skill local de bodega de vinos a partir de CSV" />
</Card>

<Card title="Piloto automático para compras en Tesco" icon="cart-shopping" href="https://x.com/i/status/2009724862470689131">
  **@marchattonhere** • `automation` `browser` `shopping`

Plan semanal de comidas → productos habituales → reservar franja de entrega → confirmar pedido. Sin API, solo control del navegador.

  <img src="/assets/showcase/tesco-shop.jpg" alt="Automatización de compras en Tesco mediante chat" />
</Card>

<Card title="SNAG de captura de pantalla a Markdown" icon="scissors" href="https://github.com/am-will/snag">
  **@am-will** • `devtools` `screenshots` `markdown`

Tecla rápida para una región de la pantalla → visión de Gemini → Markdown instantáneo en el portapapeles.

  <img src="/assets/showcase/snag.png" alt="Herramienta SNAG de captura de pantalla a Markdown" />
</Card>

<Card title="IU de Agents" icon="window-maximize" href="https://releaseflow.net/kitze/agents-ui">
  **@kitze** • `ui` `skills` `sync`

App de escritorio para gestionar skills/comandos en Agents, Claude, Codex y OpenClaw.

  <img src="/assets/showcase/agents-ui.jpg" alt="App Agents UI" />
</Card>

<Card title="Notas de voz de Telegram (papla.media)" icon="microphone" href="https://papla.media/docs">
  **Comunidad** • `voice` `tts` `telegram`

Envuelve el TTS de papla.media y envía los resultados como notas de voz de Telegram (sin molesta reproducción automática).

  <img src="/assets/showcase/papla-tts.jpg" alt="Salida de nota de voz de Telegram desde TTS" />
</Card>

<Card title="CodexMonitor" icon="eye" href="https://clawhub.ai/odrobnik/codexmonitor">
  **@odrobnik** • `devtools` `codex` `brew`

Utilidad instalada con Homebrew para listar/inspeccionar/monitorizar sesiones locales de OpenAI Codex (CLI + VS Code).

  <img src="/assets/showcase/codexmonitor.png" alt="CodexMonitor en ClawHub" />
</Card>

<Card title="Control de impresora 3D Bambu" icon="print" href="https://clawhub.ai/tobiasbischoff/bambu-cli">
  **@tobiasbischoff** • `hardware` `3d-printing` `skill`

Controla y soluciona problemas de impresoras BambuLab: estado, trabajos, cámara, AMS, calibración y más.

  <img src="/assets/showcase/bambu-cli.png" alt="Skill Bambu CLI en ClawHub" />
</Card>

<Card title="Transporte de Viena (Wiener Linien)" icon="train" href="https://clawhub.ai/hjanuschka/wienerlinien">
  **@hjanuschka** • `travel` `transport` `skill`

Salidas en tiempo real, interrupciones, estado de ascensores y rutas para el transporte público de Viena.

  <img src="/assets/showcase/wienerlinien.png" alt="Skill Wiener Linien en ClawHub" />
</Card>

<Card title="Comidas escolares en ParentPay" icon="utensils" href="#">
  **@George5562** • `automation` `browser` `parenting`

Reserva automatizada de comidas escolares del Reino Unido mediante ParentPay. Usa coordenadas del ratón para pulsar de forma fiable las celdas de la tabla.
</Card>

<Card title="Subida a R2 (Send Me My Files)" icon="cloud-arrow-up" href="https://clawhub.ai/skills/r2-upload">
  **@julianengel** • `files` `r2` `presigned-urls`

Sube a Cloudflare R2/S3 y genera enlaces de descarga seguros prefirmados. Perfecto para instancias remotas de OpenClaw.
</Card>

<Card title="App de iOS mediante Telegram" icon="mobile" href="#">
  **@coard** • `ios` `xcode` `testflight`

Creó una app completa de iOS con mapas y grabación de voz, desplegada en TestFlight completamente a través del chat de Telegram.

  <img src="/assets/showcase/ios-testflight.jpg" alt="App de iOS en TestFlight" />
</Card>

<Card title="Asistente de salud para Oura Ring" icon="heart-pulse" href="#">
  **@AS** • `health` `oura` `calendar`

Asistente personal de salud con IA que integra datos de Oura ring con calendario, citas y horario del gimnasio.

  <img src="/assets/showcase/oura-health.png" alt="Asistente de salud para Oura ring" />
</Card>
<Card title="El equipo soñado de Kev (14+ Agents)" icon="robot" href="https://github.com/adam91holt/orchestrated-ai-articles">
  **@adam91holt** • `multi-agent` `orchestration` `architecture` `manifesto`

Más de 14 agentes bajo un único gateway con un orquestador Opus 4.5 que delega en workers de Codex. [Escrito técnico](https://github.com/adam91holt/orchestrated-ai-articles) completo que cubre la plantilla Dream Team, selección de modelos, sandboxing, webhooks, heartbeats y flujos de delegación. [Clawdspace](https://github.com/adam91holt/clawdspace) para el sandboxing de agentes. [Entrada del blog](https://adams-ai-journey.ghost.io/2026-the-year-of-the-orchestrator/).
</Card>

<Card title="CLI de Linear" icon="terminal" href="https://github.com/Finesssee/linear-cli">
  **@NessZerra** • `devtools` `linear` `cli` `issues`

CLI para Linear que se integra con flujos de trabajo agénticos (Claude Code, OpenClaw). Gestiona incidencias, proyectos y flujos de trabajo desde la terminal. ¡Primer PR externo fusionado!
</Card>

<Card title="CLI de Beeper" icon="message" href="https://github.com/blqke/beepcli">
  **@jules** • `messaging` `beeper` `cli` `automation`

Lee, envía y archiva mensajes mediante Beeper Desktop. Usa la API MCP local de Beeper para que los agentes puedan gestionar todos tus chats (iMessage, WhatsApp, etc.) en un solo lugar.
</Card>

</CardGroup>

## 🤖 Automatización y flujos de trabajo

<CardGroup cols={2}>

<Card title="Control de purificador de aire Winix" icon="wind" href="https://x.com/antonplex/status/2010518442471006253">
  **@antonplex** • `automation` `hardware` `air-quality`

Claude Code descubrió y confirmó los controles del purificador, y luego OpenClaw toma el relevo para gestionar la calidad del aire de la habitación.

  <img src="/assets/showcase/winix-air-purifier.jpg" alt="Control del purificador de aire Winix mediante OpenClaw" />
</Card>

<Card title="Bonitas fotos del cielo desde cámara" icon="camera" href="https://x.com/signalgaining/status/2010523120604746151">
  **@signalgaining** • `automation` `camera` `skill` `images`

Activado por una cámara en el tejado: pedir a OpenClaw que tome una foto del cielo cuando se vea bonito; diseñó una skill y tomó la foto.

  <img src="/assets/showcase/roof-camera-sky.jpg" alt="Instantánea del cielo desde la cámara del tejado capturada por OpenClaw" />
</Card>

<Card title="Escena visual de briefing matutino" icon="robot" href="https://x.com/buddyhadry/status/2010005331925954739">
  **@buddyhadry** • `automation` `briefing` `images` `telegram`

Un prompt programado genera cada mañana una única imagen de “escena” (clima, tareas, fecha, publicación/cita favorita) mediante una persona de OpenClaw.
</Card>

<Card title="Reserva de pista de pádel" icon="calendar-check" href="https://github.com/joshp123/padel-cli">
  **@joshp123** • `automation` `booking` `cli`
  
  CLI para comprobar disponibilidad en Playtomic y hacer reservas. No vuelvas a perder una pista libre.
  
  <img src="/assets/showcase/padel-screenshot.jpg" alt="Captura de pantalla de padel-cli" />
</Card>

<Card title="Captura de contabilidad" icon="file-invoice-dollar">
  **Comunidad** • `automation` `email` `pdf`
  
  Recoge PDFs del correo electrónico y prepara documentos para el asesor fiscal. Contabilidad mensual en piloto automático.
</Card>

<Card title="Modo desarrollador desde el sofá" icon="couch" href="https://davekiss.com">
  **@davekiss** • `telegram` `website` `migration` `astro`

Reconstruyó todo su sitio personal mediante Telegram mientras veía Netflix: Notion → Astro, 18 publicaciones migradas, DNS a Cloudflare. Nunca abrió un portátil.
</Card>

<Card title="Agente de búsqueda de empleo" icon="briefcase">
  **@attol8** • `automation` `api` `skill`

Busca ofertas de trabajo, las compara con palabras clave del CV y devuelve oportunidades relevantes con enlaces. Creado en 30 minutos con la API de JSearch.
</Card>

<Card title="Constructor de skill para Jira" icon="diagram-project" href="https://x.com/jdrhyne/status/2008336434827002232">
  **@jdrhyne** • `automation` `jira` `skill` `devtools`

OpenClaw se conectó a Jira y luego generó una nueva skill sobre la marcha (antes de que existiera en ClawHub).
</Card>

<Card title="Skill de Todoist mediante Telegram" icon="list-check" href="https://x.com/iamsubhrajyoti/status/2009949389884920153">
  **@iamsubhrajyoti** • `automation` `todoist` `skill` `telegram`

Automatizó tareas de Todoist e hizo que OpenClaw generara la skill directamente en el chat de Telegram.
</Card>

<Card title="Análisis de TradingView" icon="chart-line">
  **@bheem1798** • `finance` `browser` `automation`

Inicia sesión en TradingView mediante automatización del navegador, captura pantallas de gráficos y realiza análisis técnico bajo demanda. No necesita API, solo control del navegador.
</Card>

<Card title="Soporte automático en Slack" icon="slack">
  **@henrymascot** • `slack` `automation` `support`

Supervisa el canal de Slack de la empresa, responde de forma útil y reenvía notificaciones a Telegram. Corrigió de forma autónoma un error de producción en una app desplegada sin que nadie se lo pidiera.
</Card>

</CardGroup>

## 🧠 Conocimiento y memoria

<CardGroup cols={2}>

<Card title="Aprendizaje de chino xuezh" icon="language" href="https://github.com/joshp123/xuezh">
  **@joshp123** • `learning` `voice` `skill`
  
  Motor de aprendizaje de chino con comentarios de pronunciación y flujos de estudio mediante OpenClaw.
  
  <img src="/assets/showcase/xuezh-pronunciation.jpeg" alt="Comentarios de pronunciación de xuezh" />
</Card>

<Card title="Bóveda de memoria de WhatsApp" icon="vault">
  **Comunidad** • `memory` `transcription` `indexing`
  
  Ingiera exportaciones completas de WhatsApp, transcribe más de 1.000 notas de voz, las contrasta con logs de git y genera informes Markdown enlazados.
</Card>

<Card title="Búsqueda semántica de Karakeep" icon="magnifying-glass" href="https://github.com/jamesbrooksco/karakeep-semantic-search">
  **@jamesbrooksco** • `search` `vector` `bookmarks`
  
  Añade búsqueda vectorial a los marcadores de Karakeep usando embeddings de Qdrant + OpenAI/Ollama.
</Card>

<Card title="Memoria de Del Revés 2" icon="brain">
  **Comunidad** • `memory` `beliefs` `self-model`
  
  Gestor de memoria independiente que convierte archivos de sesión en recuerdos → creencias → modelo del yo en evolución.
</Card>

</CardGroup>

## 🎙️ Voz y teléfono

<CardGroup cols={2}>

<Card title="Puente telefónico Clawdia" icon="phone" href="https://github.com/alejandroOPI/clawdia-bridge">
  **@alejandroOPI** • `voice` `vapi` `bridge`
  
  Puente HTTP entre asistente de voz Vapi y OpenClaw. Llamadas telefónicas casi en tiempo real con tu agente.
</Card>

<Card title="Transcripción con OpenRouter" icon="microphone" href="https://clawhub.ai/obviyus/openrouter-transcribe">
  **@obviyus** • `transcription` `multilingual` `skill`

Transcripción de audio multilingüe mediante OpenRouter (Gemini, etc.). Disponible en ClawHub.
</Card>

</CardGroup>

## 🏗️ Infraestructura y despliegue

<CardGroup cols={2}>

<Card title="Complemento para Home Assistant" icon="home" href="https://github.com/ngutman/openclaw-ha-addon">
  **@ngutman** • `homeassistant` `docker` `raspberry-pi`
  
  Gateway de OpenClaw ejecutándose en Home Assistant OS con soporte para túnel SSH y estado persistente.
</Card>

<Card title="Skill de Home Assistant" icon="toggle-on" href="https://clawhub.ai/skills/homeassistant">
  **ClawHub** • `homeassistant` `skill` `automation`
  
  Controla y automatiza dispositivos de Home Assistant mediante lenguaje natural.
</Card>

<Card title="Empaquetado con Nix" icon="snowflake" href="https://github.com/openclaw/nix-openclaw">
  **@openclaw** • `nix` `packaging` `deployment`
  
  Configuración de OpenClaw con Nix y todo incluido para despliegues reproducibles.
</Card>

<Card title="Calendario CalDAV" icon="calendar" href="https://clawhub.ai/skills/caldav-calendar">
  **ClawHub** • `calendar` `caldav` `skill`
  
  Skill de calendario con khal/vdirsyncer. Integración de calendario autoalojado.
</Card>

</CardGroup>

## 🏠 Hogar y hardware

<CardGroup cols={2}>

<Card title="Automatización GoHome" icon="house-signal" href="https://github.com/joshp123/gohome">
  **@joshp123** • `home` `nix` `grafana`
  
  Automatización del hogar nativa de Nix con OpenClaw como interfaz, además de bonitos paneles de Grafana.
  
  <img src="/assets/showcase/gohome-grafana.png" alt="Panel de Grafana de GoHome" />
</Card>

<Card title="Aspiradora Roborock" icon="robot" href="https://github.com/joshp123/gohome/tree/main/plugins/roborock">
  **@joshp123** • `vacuum` `iot` `plugin`
  
  Controla tu aspiradora robot Roborock mediante conversación natural.
  
  <img src="/assets/showcase/roborock-screenshot.jpg" alt="Estado de Roborock" />
</Card>

</CardGroup>

## 🌟 Proyectos de la comunidad

<CardGroup cols={2}>

<Card title="Marketplace StarSwap" icon="star" href="https://star-swap.com/">
  **Comunidad** • `marketplace` `astronomy` `webapp`
  
  Marketplace completo de equipamiento astronómico. Creado con/alrededor del ecosistema OpenClaw.
</Card>

</CardGroup>

---

## Envía tu proyecto

¿Tienes algo que compartir? ¡Nos encantaría destacarlo!

<Steps>
  <Step title="Compártelo">
    Publica en [#self-promotion en Discord](https://discord.gg/clawd) o [menciona a @openclaw en X](https://x.com/openclaw)
  </Step>
  <Step title="Incluye detalles">
    Cuéntanos qué hace, enlaza el repositorio/demo y comparte una captura de pantalla si tienes una
  </Step>
  <Step title="Consigue aparecer">
    Añadiremos los proyectos más destacados a esta página
  </Step>
</Steps>
