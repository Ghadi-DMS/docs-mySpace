---
read_when:
    - Quieres entender qué funciones pueden llamar a API de pago
    - Necesitas auditar claves, costos y visibilidad del uso
    - Estás explicando los informes de costos de /status o /usage
summary: Audita qué puede gastar dinero, qué claves se usan y cómo ver el uso
title: Uso y costos de la API
x-i18n:
    generated_at: "2026-04-05T12:53:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71789950fe54dcdcd3e34c8ad6e3143f749cdfff5bbc2f14be4b85aaa467b14c
    source_path: reference/api-usage-costs.md
    workflow: 15
---

# Uso y costos de la API

Este documento enumera las **funciones que pueden invocar claves de API** y dónde aparecen sus costos. Se centra en las funciones de
OpenClaw que pueden generar uso del proveedor o llamadas a API de pago.

## Dónde aparecen los costos (chat + CLI)

**Instantánea del costo por sesión**

- `/status` muestra el modelo actual de la sesión, el uso del contexto y los tokens de la última respuesta.
- Si el modelo usa **autenticación con clave de API**, `/status` también muestra el **costo estimado** de la última respuesta.
- Si los metadatos de la sesión en vivo son escasos, `/status` puede recuperar los
  contadores de tokens/caché y la etiqueta activa del modelo de ejecución a partir de la entrada
  de uso más reciente de la transcripción. Los valores activos distintos de cero existentes siguen teniendo prioridad, y los totales de la transcripción dimensionados por el prompt
  pueden prevalecer cuando faltan los totales almacenados o son menores.

**Pie de costo por mensaje**

- `/usage full` agrega un pie de uso a cada respuesta, incluido el **costo estimado** (solo con clave de API).
- `/usage tokens` muestra solo los tokens; los flujos de OAuth/token de estilo suscripción y los flujos de CLI ocultan el costo en dólares.
- Nota sobre Gemini CLI: cuando la CLI devuelve salida JSON, OpenClaw lee el uso desde
  `stats`, normaliza `stats.cached` a `cacheRead` y deriva los tokens de entrada
  desde `stats.input_tokens - stats.cached` cuando es necesario.

Nota sobre Anthropic: la documentación pública de Claude Code de Anthropic todavía incluye el uso directo de Claude
Code en terminal dentro de los límites del plan Claude. Por separado, Anthropic informó a los usuarios de OpenClaw
que a partir del **4 de abril de 2026 a las 12:00 PM PT / 8:00 PM BST**, la
ruta de inicio de sesión de Claude en **OpenClaw** cuenta como uso de un entorno de terceros y
requiere **Extra Usage**, facturado por separado de la suscripción. Anthropic
no expone una estimación en dólares por mensaje que OpenClaw pueda mostrar en
`/usage full`.

**Ventanas de uso de la CLI (cuotas del proveedor)**

- `openclaw status --usage` y `openclaw channels list` muestran las **ventanas de uso**
  del proveedor (instantáneas de cuota, no costos por mensaje).
- La salida para humanos se normaliza a `X% left` entre proveedores.
- Proveedores actuales con ventana de uso: Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi y z.ai.
- Nota sobre MiniMax: sus campos sin procesar `usage_percent` / `usagePercent` significan cuota
  restante, por lo que OpenClaw los invierte antes de mostrarlos. Los campos basados en conteo siguen teniendo prioridad
  cuando están presentes. Si el proveedor devuelve `model_remains`, OpenClaw prefiere la entrada del modelo de chat, deriva la etiqueta de la ventana a partir de las marcas de tiempo cuando es necesario y
  incluye el nombre del modelo en la etiqueta del plan.
- La autenticación de uso para esas ventanas de cuota proviene de hooks específicos del proveedor cuando
  están disponibles; de lo contrario, OpenClaw recurre a credenciales OAuth/con clave de API coincidentes
  de perfiles de autenticación, variables de entorno o configuración.

Consulta [Uso y costos de tokens](/reference/token-use) para ver detalles y ejemplos.

## Cómo se detectan las claves

OpenClaw puede recoger credenciales de:

- **Perfiles de autenticación** (por agente, almacenados en `auth-profiles.json`).
- **Variables de entorno** (por ejemplo, `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`).
- **Configuración** (`models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`,
  `plugins.entries.firecrawl.config.webFetch.apiKey`, `memorySearch.*`,
  `talk.providers.*.apiKey`).
- **Skills** (`skills.entries.<name>.apiKey`) que pueden exportar claves al entorno del proceso de la skill.

## Funciones que pueden gastar claves

### 1) Respuestas del modelo central (chat + herramientas)

Cada respuesta o llamada a herramienta usa el **proveedor del modelo actual** (OpenAI, Anthropic, etc.). Esta es la
fuente principal de uso y costo.

Esto también incluye proveedores alojados de estilo suscripción que siguen facturando fuera de
la UI local de OpenClaw, como **OpenAI Codex**, **Alibaba Cloud Model Studio
Coding Plan**, **MiniMax Coding Plan**, **Z.AI / GLM Coding Plan** y
la ruta de inicio de sesión de Claude en OpenClaw de Anthropic con **Extra Usage** habilitado.

Consulta [Modelos](/es/providers/models) para la configuración de precios y [Uso y costos de tokens](/reference/token-use) para la visualización.

### 2) Comprensión de medios (audio/imagen/video)

Los medios entrantes pueden resumirse o transcribirse antes de que se ejecute la respuesta. Esto usa API de modelo/proveedor.

- Audio: OpenAI / Groq / Deepgram / Google / Mistral.
- Imagen: OpenAI / OpenRouter / Anthropic / Google / MiniMax / Moonshot / Qwen / Z.AI.
- Video: Google / Qwen / Moonshot.

Consulta [Comprensión de medios](/es/nodes/media-understanding).

### 3) Generación de imágenes y video

Las capacidades compartidas de generación también pueden gastar claves de proveedor:

- Generación de imágenes: OpenAI / Google / fal / MiniMax
- Generación de video: Qwen

La generación de imágenes puede inferir un valor predeterminado de proveedor respaldado por autenticación cuando
`agents.defaults.imageGenerationModel` no está configurado. La generación de video actualmente
requiere un `agents.defaults.videoGenerationModel` explícito, como
`qwen/wan2.6-t2v`.

Consulta [Generación de imágenes](/tools/image-generation), [Qwen Cloud](/es/providers/qwen)
y [Modelos](/es/concepts/models).

### 4) Embeddings de memoria + búsqueda semántica

La búsqueda semántica de memoria usa **API de embeddings** cuando está configurada para proveedores remotos:

- `memorySearch.provider = "openai"` → embeddings de OpenAI
- `memorySearch.provider = "gemini"` → embeddings de Gemini
- `memorySearch.provider = "voyage"` → embeddings de Voyage
- `memorySearch.provider = "mistral"` → embeddings de Mistral
- `memorySearch.provider = "ollama"` → embeddings de Ollama (local/alojado por cuenta propia; normalmente sin facturación de API alojada)
- Respaldo opcional a un proveedor remoto si fallan los embeddings locales

Puedes mantenerlo local con `memorySearch.provider = "local"` (sin uso de API).

Consulta [Memoria](/es/concepts/memory).

### 5) Herramienta de búsqueda web

`web_search` puede generar cargos de uso según tu proveedor:

- **Brave Search API**: `BRAVE_API_KEY` o `plugins.entries.brave.config.webSearch.apiKey`
- **Exa**: `EXA_API_KEY` o `plugins.entries.exa.config.webSearch.apiKey`
- **Firecrawl**: `FIRECRAWL_API_KEY` o `plugins.entries.firecrawl.config.webSearch.apiKey`
- **Gemini (Google Search)**: `GEMINI_API_KEY` o `plugins.entries.google.config.webSearch.apiKey`
- **Grok (xAI)**: `XAI_API_KEY` o `plugins.entries.xai.config.webSearch.apiKey`
- **Kimi (Moonshot)**: `KIMI_API_KEY`, `MOONSHOT_API_KEY` o `plugins.entries.moonshot.config.webSearch.apiKey`
- **MiniMax Search**: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_API_KEY` o `plugins.entries.minimax.config.webSearch.apiKey`
- **Ollama Web Search**: sin clave de forma predeterminada, pero requiere un host de Ollama accesible más `ollama signin`; también puede reutilizar la autenticación bearer normal del proveedor Ollama cuando el host la requiere
- **Perplexity Search API**: `PERPLEXITY_API_KEY`, `OPENROUTER_API_KEY` o `plugins.entries.perplexity.config.webSearch.apiKey`
- **Tavily**: `TAVILY_API_KEY` o `plugins.entries.tavily.config.webSearch.apiKey`
- **DuckDuckGo**: respaldo sin clave (sin facturación de API, pero no oficial y basado en HTML)
- **SearXNG**: `SEARXNG_BASE_URL` o `plugins.entries.searxng.config.webSearch.baseUrl` (sin clave/alojado por cuenta propia; sin facturación de API alojada)

Las rutas heredadas de proveedor `tools.web.search.*` todavía se cargan mediante el shim temporal de compatibilidad, pero ya no son la superficie de configuración recomendada.

**Crédito gratuito de Brave Search:** Cada plan de Brave incluye \$5/mes en crédito
gratuito renovable. El plan Search cuesta \$5 por cada 1,000 solicitudes, por lo que el crédito cubre
1,000 solicitudes/mes sin cargo. Configura tu límite de uso en el panel de Brave
para evitar cargos inesperados.

Consulta [Herramientas web](/tools/web).

### 5) Herramienta de recuperación web (Firecrawl)

`web_fetch` puede llamar a **Firecrawl** cuando hay una clave de API presente:

- `FIRECRAWL_API_KEY` o `plugins.entries.firecrawl.config.webFetch.apiKey`

Si Firecrawl no está configurado, la herramienta recurre a recuperación directa + readability (sin API de pago).

Consulta [Herramientas web](/tools/web).

### 6) Instantáneas de uso del proveedor (status/health)

Algunos comandos de estado llaman a **endpoints de uso del proveedor** para mostrar ventanas de cuota o el estado de la autenticación.
Normalmente son llamadas de bajo volumen, pero igualmente alcanzan las API del proveedor:

- `openclaw status --usage`
- `openclaw models status --json`

Consulta [Models CLI](/cli/models).

### 7) Resumen de protección de compactación

La protección de compactación puede resumir el historial de la sesión usando el **modelo actual**, lo que
invoca API del proveedor cuando se ejecuta.

Consulta [Gestión de sesiones + compactación](/reference/session-management-compaction).

### 8) Escaneo / sondeo de modelos

`openclaw models scan` puede sondear modelos de OpenRouter y usa `OPENROUTER_API_KEY` cuando
el sondeo está habilitado.

Consulta [Models CLI](/cli/models).

### 9) Talk (voz)

El modo Talk puede invocar **ElevenLabs** cuando está configurado:

- `ELEVENLABS_API_KEY` o `talk.providers.elevenlabs.apiKey`

Consulta [Modo Talk](/es/nodes/talk).

### 10) Skills (API de terceros)

Las Skills pueden almacenar `apiKey` en `skills.entries.<name>.apiKey`. Si una skill usa esa clave para API externas,
puede generar costos según el proveedor de la skill.

Consulta [Skills](/tools/skills).
