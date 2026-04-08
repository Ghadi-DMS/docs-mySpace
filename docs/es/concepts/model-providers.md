---
read_when:
    - Necesitas una referencia de configuración de modelos proveedor por proveedor
    - Quieres configuraciones de ejemplo o comandos de incorporación en CLI para proveedores de modelos
summary: Resumen de proveedores de modelos con configuraciones de ejemplo + flujos de CLI
title: Proveedores de modelos
x-i18n:
    generated_at: "2026-04-08T02:15:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26b36a2bc19a28a7ef39aa8e81a0050fea1d452ac4969122e5cdf8755e690258
    source_path: concepts/model-providers.md
    workflow: 15
---

# Proveedores de modelos

Esta página cubre los **proveedores de LLM/modelos** (no canales de chat como WhatsApp/Telegram).
Para las reglas de selección de modelos, consulta [/concepts/models](/es/concepts/models).

## Reglas rápidas

- Las referencias de modelo usan `provider/model` (ejemplo: `opencode/claude-opus-4-6`).
- Si configuras `agents.defaults.models`, se convierte en la lista permitida.
- Ayudantes de CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Las reglas de ejecución de respaldo, las sondas de enfriamiento y la persistencia de anulación por sesión
  están documentadas en [/concepts/model-failover](/es/concepts/model-failover).
- `models.providers.*.models[].contextWindow` son metadatos nativos del modelo;
  `models.providers.*.models[].contextTokens` es el límite efectivo en tiempo de ejecución.
- Los plugins de proveedor pueden inyectar catálogos de modelos mediante `registerProvider({ catalog })`;
  OpenClaw fusiona esa salida en `models.providers` antes de escribir
  `models.json`.
- Los manifiestos de proveedor pueden declarar `providerAuthEnvVars` para que las
  sondas genéricas de autenticación basadas en variables de entorno no necesiten cargar el tiempo de ejecución del plugin. El mapa restante de variables de entorno del núcleo
  ahora es solo para proveedores no basados en plugins/del núcleo y algunos casos de precedencia genérica, como la incorporación de Anthropic con prioridad de clave API.
- Los plugins de proveedor también pueden encargarse del comportamiento en tiempo de ejecución del proveedor mediante
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, y
  `onModelSelected`.
- Nota: `capabilities` del tiempo de ejecución del proveedor son metadatos compartidos del ejecutor (familia de proveedor, particularidades de transcripción/herramientas, sugerencias de transporte/caché). No es lo
  mismo que el [modelo público de capacidades](/es/plugins/architecture#public-capability-model)
  que describe lo que registra un plugin (inferencia de texto, voz, etc.).

## Comportamiento de proveedor controlado por plugins

Los plugins de proveedor ahora pueden encargarse de la mayor parte de la lógica específica del proveedor mientras OpenClaw mantiene
el bucle genérico de inferencia.

División típica:

- `auth[].run` / `auth[].runNonInteractive`: el proveedor se encarga de los flujos de incorporación/inicio de sesión
  para `openclaw onboard`, `openclaw models auth` y la configuración sin interfaz
- `wizard.setup` / `wizard.modelPicker`: el proveedor se encarga de las etiquetas de elección de autenticación,
  alias heredados, sugerencias de lista permitida para la incorporación y entradas de configuración en los selectores de incorporación/modelos
- `catalog`: el proveedor aparece en `models.providers`
- `normalizeModelId`: el proveedor normaliza identificadores de modelo heredados/de vista previa antes de
  la búsqueda o canonicalización
- `normalizeTransport`: el proveedor normaliza `api` / `baseUrl` de la familia de transporte
  antes del ensamblaje genérico del modelo; OpenClaw comprueba primero el proveedor coincidente,
  luego otros plugins de proveedor con capacidad de hooks hasta que uno
  realmente cambie el transporte
- `normalizeConfig`: el proveedor normaliza la configuración `models.providers.<id>` antes de que
  el tiempo de ejecución la use; OpenClaw comprueba primero el proveedor coincidente, luego otros
  plugins de proveedor con capacidad de hooks hasta que uno realmente cambie la configuración. Si ningún
  hook de proveedor reescribe la configuración, los ayudantes integrados de la familia Google
  siguen normalizando las entradas de proveedor de Google compatibles.
- `applyNativeStreamingUsageCompat`: el proveedor aplica reescrituras de compatibilidad de uso de streaming nativo impulsadas por el endpoint para proveedores de configuración
- `resolveConfigApiKey`: el proveedor resuelve la autenticación con marcador de entorno para proveedores de configuración
  sin forzar la carga completa de la autenticación en tiempo de ejecución. `amazon-bedrock` también tiene aquí un
  resolvedor integrado de marcadores de entorno de AWS, aunque la autenticación en tiempo de ejecución de Bedrock usa
  la cadena predeterminada del SDK de AWS.
- `resolveSyntheticAuth`: el proveedor puede exponer disponibilidad de autenticación local/alojada por el usuario u otra
  autenticación respaldada por configuración sin persistir secretos en texto plano
- `shouldDeferSyntheticProfileAuth`: el proveedor puede marcar marcadores de posición de perfiles sintéticos almacenados
  como de menor precedencia que la autenticación respaldada por entorno/configuración
- `resolveDynamicModel`: el proveedor acepta identificadores de modelo que aún no están presentes en el
  catálogo estático local
- `prepareDynamicModel`: el proveedor necesita una actualización de metadatos antes de volver a intentar
  la resolución dinámica
- `normalizeResolvedModel`: el proveedor necesita reescrituras de transporte o URL base
- `contributeResolvedModelCompat`: el proveedor aporta indicadores de compatibilidad para sus
  modelos del proveedor incluso cuando llegan a través de otro transporte compatible
- `capabilities`: el proveedor publica particularidades de transcripción/herramientas/familia de proveedor
- `normalizeToolSchemas`: el proveedor limpia los esquemas de herramientas antes de que el
  ejecutor integrado los vea
- `inspectToolSchemas`: el proveedor muestra advertencias de esquemas específicas del transporte
  después de la normalización
- `resolveReasoningOutputMode`: el proveedor elige contratos de salida de razonamiento
  nativos o etiquetados
- `prepareExtraParams`: el proveedor define valores predeterminados o normaliza parámetros de solicitud por modelo
- `createStreamFn`: el proveedor reemplaza la ruta normal de streaming con un transporte
  completamente personalizado
- `wrapStreamFn`: el proveedor aplica envoltorios de compatibilidad de encabezados/cuerpo/modelo a las solicitudes
- `resolveTransportTurnState`: el proveedor suministra encabezados o metadatos nativos
  de transporte por turno
- `resolveWebSocketSessionPolicy`: el proveedor suministra encabezados nativos de sesión WebSocket
  o una política de enfriamiento de sesión
- `createEmbeddingProvider`: el proveedor se encarga del comportamiento de embeddings de memoria cuando
  corresponde al plugin del proveedor en lugar del conmutador central de embeddings del núcleo
- `formatApiKey`: el proveedor da formato a los perfiles de autenticación almacenados en la cadena
  `apiKey` esperada por el transporte en tiempo de ejecución
- `refreshOAuth`: el proveedor se encarga de la actualización OAuth cuando los
  actualizadores compartidos `pi-ai` no son suficientes
- `buildAuthDoctorHint`: el proveedor agrega orientación de reparación cuando falla
  la actualización OAuth
- `matchesContextOverflowError`: el proveedor reconoce errores de desbordamiento de ventana de contexto
  específicos del proveedor que la heurística genérica pasaría por alto
- `classifyFailoverReason`: el proveedor asigna errores sin procesar específicos del proveedor de transporte/API
  a motivos de conmutación por error, como límite de tasa o sobrecarga
- `isCacheTtlEligible`: el proveedor decide qué identificadores de modelo ascendentes admiten TTL de caché de prompt
- `buildMissingAuthMessage`: el proveedor reemplaza el error genérico del almacén de autenticación
  con una sugerencia de recuperación específica del proveedor
- `suppressBuiltInModel`: el proveedor oculta filas ascendentes obsoletas y puede devolver un
  error controlado por el proveedor ante fallos de resolución directa
- `augmentModelCatalog`: el proveedor agrega filas sintéticas/finales al catálogo después de
  la detección y la fusión de configuración
- `isBinaryThinking`: el proveedor se encarga de la UX de razonamiento binario activado/desactivado
- `supportsXHighThinking`: el proveedor habilita `xhigh` para modelos seleccionados
- `resolveDefaultThinkingLevel`: el proveedor se encarga de la política predeterminada de `/think` para una
  familia de modelos
- `applyConfigDefaults`: el proveedor aplica valores predeterminados globales específicos del proveedor
  durante la materialización de la configuración según el modo de autenticación, el entorno o la familia de modelos
- `isModernModelRef`: el proveedor se encarga de la coincidencia de modelos preferidos en vivo/smoke
- `prepareRuntimeAuth`: el proveedor convierte una credencial configurada en un token
  de tiempo de ejecución de corta duración
- `resolveUsageAuth`: el proveedor resuelve credenciales de uso/cuota para `/usage`
  y superficies relacionadas de estado/informes
- `fetchUsageSnapshot`: el proveedor se encarga de la obtención/análisis del endpoint de uso mientras
  el núcleo sigue encargándose del contenedor de resumen y el formato
- `onModelSelected`: el proveedor ejecuta efectos secundarios posteriores a la selección, como
  telemetría o contabilidad de sesión controlada por el proveedor

Ejemplos integrados actuales:

- `anthropic`: fallback de compatibilidad futura de Claude 4.6, sugerencias de reparación de autenticación, obtención de endpoints de uso, metadatos de caché TTL/familia de proveedor y valores predeterminados globales de configuración con reconocimiento de autenticación
- `amazon-bedrock`: coincidencia de desbordamiento de contexto y clasificación del motivo
  de conmutación por error controladas por el proveedor para errores específicos de Bedrock de limitación/no listo, además
  de la familia compartida de reproducción `anthropic-by-model` para protecciones de política de repetición solo para Claude en tráfico de Anthropic
- `anthropic-vertex`: protecciones de política de repetición solo para Claude en tráfico
  de mensajes de Anthropic
- `openrouter`: identificadores de modelo de paso directo, envoltorios de solicitud, sugerencias de capacidad del proveedor, saneamiento de firma de pensamiento de Gemini en tráfico Gemini proxy,
  inyección de razonamiento proxy mediante la familia de streams `openrouter-thinking`, reenvío de metadatos de enrutamiento y política de caché TTL
- `github-copilot`: incorporación/inicio de sesión del dispositivo, fallback de modelo con compatibilidad futura,
  sugerencias de transcripción de razonamiento de Claude, intercambio de tokens en tiempo de ejecución y obtención de endpoints de uso
- `openai`: fallback de compatibilidad futura de GPT-5.4, normalización directa del transporte de OpenAI,
  sugerencias de autenticación faltante con reconocimiento de Codex, supresión de Spark, filas sintéticas de catálogo OpenAI/Codex, política de razonamiento/modelo en vivo, normalización de alias de tokens de uso (`input` / `output` y familias `prompt` / `completion`), la familia compartida de streams `openai-responses-defaults` para envoltorios nativos OpenAI/Codex, metadatos de familia de proveedor, registro integrado de proveedor de generación de imágenes para `gpt-image-1` y registro integrado de proveedor de generación de video para `sora-2`
- `google` y `google-gemini-cli`: fallback de compatibilidad futura de Gemini 3.1,
  validación nativa de reproducción de Gemini, saneamiento de reproducción de bootstrap, modo de salida de razonamiento etiquetado, coincidencia de modelos modernos, registro integrado de proveedor de generación de imágenes para modelos de vista previa de imágenes Gemini y registro integrado de proveedor de generación de video para modelos Veo; OAuth de Gemini CLI también se encarga del formato de tokens de perfiles de autenticación, del análisis de tokens de uso y de la obtención de endpoints de cuota para superficies de uso
- `moonshot`: transporte compartido, normalización de carga útil de razonamiento controlada por plugin
- `kilocode`: transporte compartido, encabezados de solicitud controlados por plugin, normalización
  de carga útil de razonamiento, saneamiento de firma de pensamiento de Gemini proxy y política de caché TTL
- `zai`: fallback de compatibilidad futura de GLM-5, valores predeterminados de `tool_stream`, política de caché TTL, política de razonamiento binario/modelo en vivo y autenticación de uso + obtención de cuota;
  identificadores desconocidos `glm-5*` se sintetizan a partir de la plantilla integrada `glm-4.7`
- `xai`: normalización nativa del transporte Responses, reescrituras de alias `/fast` para
  variantes rápidas de Grok, `tool_stream` predeterminado, limpieza específica de xAI de esquema de herramientas /
  carga útil de razonamiento y registro integrado de proveedor de generación de video
  para `grok-imagine-video`
- `mistral`: metadatos de capacidad controlados por plugin
- `opencode` y `opencode-go`: metadatos de capacidad controlados por plugin más
  saneamiento de firma de pensamiento de Gemini proxy
- `alibaba`: catálogo de generación de video controlado por plugin para referencias directas a modelos Wan
  como `alibaba/wan2.6-t2v`
- `byteplus`: catálogos controlados por plugin más registro integrado de proveedor de generación de video
  para modelos Seedance de texto a video/imagen a video
- `fal`: registro integrado de proveedor de generación de video para proveedor de generación de imágenes de terceros alojado para modelos de imágenes FLUX más registro integrado de proveedor
  de generación de video para modelos de video de terceros alojados
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` y `volcengine`:
  solo catálogos controlados por plugin
- `qwen`: catálogos controlados por plugin para modelos de texto más registros compartidos
  de proveedores de comprensión de medios y generación de video para sus
  superficies multimodales; la generación de video de Qwen usa los endpoints de video Standard DashScope con modelos Wan integrados como `wan2.6-t2v` y `wan2.7-r2v`
- `runway`: registro de proveedor de generación de video controlado por plugin para modelos nativos
  de Runway basados en tareas como `gen4.5`
- `minimax`: catálogos controlados por plugin, registro integrado de proveedor de generación de video
  para modelos de video Hailuo, registro integrado de proveedor de generación de imágenes
  para `image-01`, selección híbrida de política de reproducción Anthropic/OpenAI y lógica de autenticación/instantánea de uso
- `together`: catálogos controlados por plugin más registro integrado de proveedor de generación de video
  para modelos de video Wan
- `xiaomi`: catálogos controlados por plugin más lógica de autenticación/instantánea de uso

El plugin integrado `openai` ahora se encarga de ambos identificadores de proveedor: `openai` y
`openai-codex`.

Eso cubre los proveedores que todavía encajan en los transportes normales de OpenClaw. Un proveedor
que necesite un ejecutor de solicitudes totalmente personalizado es una superficie de extensión
separada y más profunda.

## Rotación de claves API

- Admite rotación genérica de proveedores para proveedores seleccionados.
- Configura varias claves mediante:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (anulación única en vivo, máxima prioridad)
  - `<PROVIDER>_API_KEYS` (lista separada por comas o punto y coma)
  - `<PROVIDER>_API_KEY` (clave principal)
  - `<PROVIDER>_API_KEY_*` (lista numerada, por ejemplo `<PROVIDER>_API_KEY_1`)
- Para proveedores de Google, `GOOGLE_API_KEY` también se incluye como fallback.
- El orden de selección de claves conserva la prioridad y elimina duplicados.
- Las solicitudes se reintentan con la siguiente clave solo en respuestas por límite de tasa (por
  ejemplo `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded`, o mensajes periódicos de límite de uso).
- Los fallos que no son por límite de tasa fallan de inmediato; no se intenta rotación de claves.
- Cuando todas las claves candidatas fallan, se devuelve el error final del último intento.

## Proveedores integrados (catálogo pi-ai)

OpenClaw incluye el catálogo pi‑ai. Estos proveedores no requieren
configuración de `models.providers`; solo define la autenticación y elige un modelo.

### OpenAI

- Proveedor: `openai`
- Autenticación: `OPENAI_API_KEY`
- Rotación opcional: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, además de `OPENCLAW_LIVE_OPENAI_KEY` (anulación única)
- Modelos de ejemplo: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- El transporte predeterminado es `auto` (WebSocket primero, SSE como fallback)
- Anúlalo por modelo mediante `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- El calentamiento de WebSocket de OpenAI Responses está habilitado de forma predeterminada mediante `params.openaiWsWarmup` (`true`/`false`)
- El procesamiento prioritario de OpenAI puede habilitarse mediante `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` y `params.fastMode` asignan solicitudes directas de Responses `openai/*` a `service_tier=priority` en `api.openai.com`
- Usa `params.serviceTier` cuando quieras un nivel explícito en lugar del conmutador compartido `/fast`
- Los encabezados ocultos de atribución de OpenClaw (`originator`, `version`,
  `User-Agent`) se aplican solo en tráfico nativo de OpenAI a `api.openai.com`, no a
  proxies genéricos compatibles con OpenAI
- Las rutas nativas de OpenAI también conservan `store` de Responses, sugerencias de caché de prompt y
  ajuste de carga útil de compatibilidad de razonamiento de OpenAI; las rutas proxy no
- `openai/gpt-5.3-codex-spark` se suprime intencionalmente en OpenClaw porque la API en vivo de OpenAI lo rechaza; Spark se trata como exclusivo de Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Proveedor: `anthropic`
- Autenticación: `ANTHROPIC_API_KEY`
- Rotación opcional: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, además de `OPENCLAW_LIVE_ANTHROPIC_KEY` (anulación única)
- Modelo de ejemplo: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Las solicitudes directas a Anthropic público admiten el conmutador compartido `/fast` y `params.fastMode`, incluido el tráfico autenticado con clave API y OAuth enviado a `api.anthropic.com`; OpenClaw lo asigna a `service_tier` de Anthropic (`auto` frente a `standard_only`)
- Nota sobre Anthropic: el personal de Anthropic nos dijo que el uso de Claude CLI al estilo OpenClaw vuelve a estar permitido, así que OpenClaw trata la reutilización de Claude CLI y el uso de `claude -p` como autorizados para esta integración, salvo que Anthropic publique una nueva política.
- El token de configuración de Anthropic sigue disponible como una ruta de token compatible de OpenClaw, pero OpenClaw ahora prefiere la reutilización de Claude CLI y `claude -p` cuando están disponibles.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Proveedor: `openai-codex`
- Autenticación: OAuth (ChatGPT)
- Modelo de ejemplo: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` o `openclaw models auth login --provider openai-codex`
- El transporte predeterminado es `auto` (WebSocket primero, SSE como fallback)
- Anúlalo por modelo mediante `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- `params.serviceTier` también se reenvía en solicitudes nativas de Codex Responses (`chatgpt.com/backend-api`)
- Los encabezados ocultos de atribución de OpenClaw (`originator`, `version`,
  `User-Agent`) solo se adjuntan en tráfico nativo de Codex a
  `chatgpt.com/backend-api`, no a proxies genéricos compatibles con OpenAI
- Comparte el mismo conmutador `/fast` y la configuración `params.fastMode` que `openai/*` directo; OpenClaw lo asigna a `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` sigue disponible cuando el catálogo OAuth de Codex lo expone; depende de los permisos
- `openai-codex/gpt-5.4` mantiene `contextWindow = 1050000` nativo y un valor predeterminado en tiempo de ejecución de `contextTokens = 272000`; anula el límite en tiempo de ejecución con `models.providers.openai-codex.models[].contextTokens`
- Nota de política: OpenAI Codex OAuth es compatible explícitamente para herramientas/flujos de trabajo externos como OpenClaw.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Otras opciones alojadas de estilo suscripción

- [Qwen Cloud](/es/providers/qwen): superficie de proveedor de Qwen Cloud más asignación de endpoints de Alibaba DashScope y Coding Plan
- [MiniMax](/es/providers/minimax): acceso OAuth o con clave API a MiniMax Coding Plan
- [GLM Models](/es/providers/glm): Z.AI Coding Plan o endpoints generales de API

### OpenCode

- Autenticación: `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`)
- Proveedor de tiempo de ejecución Zen: `opencode`
- Proveedor de tiempo de ejecución Go: `opencode-go`
- Modelos de ejemplo: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` o `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (clave API)

- Proveedor: `google`
- Autenticación: `GEMINI_API_KEY`
- Rotación opcional: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, fallback de `GOOGLE_API_KEY` y `OPENCLAW_LIVE_GEMINI_KEY` (anulación única)
- Modelos de ejemplo: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Compatibilidad: la configuración heredada de OpenClaw que usa `google/gemini-3.1-flash-preview` se normaliza a `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Las ejecuciones directas de Gemini también aceptan `agents.defaults.models["google/<model>"].params.cachedContent`
  (o el heredado `cached_content`) para reenviar un identificador nativo del proveedor
  `cachedContents/...`; los aciertos de caché de Gemini aparecen como `cacheRead` en OpenClaw

### Google Vertex y Gemini CLI

- Proveedores: `google-vertex`, `google-gemini-cli`
- Autenticación: Vertex usa gcloud ADC; Gemini CLI usa su flujo OAuth
- Precaución: OAuth de Gemini CLI en OpenClaw es una integración no oficial. Algunos usuarios han informado restricciones en su cuenta de Google después de usar clientes de terceros. Revisa los términos de Google y usa una cuenta no crítica si decides continuar.
- Gemini CLI OAuth se distribuye como parte del plugin integrado `google`.
  - Instala Gemini CLI primero:
    - `brew install gemini-cli`
    - o `npm install -g @google/gemini-cli`
  - Habilita: `openclaw plugins enable google`
  - Inicia sesión: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Modelo predeterminado: `google-gemini-cli/gemini-3-flash-preview`
  - Nota: **no** pegas un id de cliente ni un secreto en `openclaw.json`. El flujo de inicio de sesión de CLI almacena
    tokens en perfiles de autenticación en el host de la gateway.
  - Si las solicitudes fallan después de iniciar sesión, define `GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host de la gateway.
  - Las respuestas JSON de Gemini CLI se analizan desde `response`; el uso vuelve a
    `stats`, con `stats.cached` normalizado a `cacheRead` de OpenClaw.

### Z.AI (GLM)

- Proveedor: `zai`
- Autenticación: `ZAI_API_KEY`
- Modelo de ejemplo: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Alias: `z.ai/*` y `z-ai/*` se normalizan a `zai/*`
  - `zai-api-key` detecta automáticamente el endpoint coincidente de Z.AI; `zai-coding-global`, `zai-coding-cn`, `zai-global` y `zai-cn` fuerzan una superficie específica

### Vercel AI Gateway

- Proveedor: `vercel-ai-gateway`
- Autenticación: `AI_GATEWAY_API_KEY`
- Modelo de ejemplo: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Proveedor: `kilocode`
- Autenticación: `KILOCODE_API_KEY`
- Modelo de ejemplo: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- URL base: `https://api.kilo.ai/api/gateway/`
- El catálogo estático de fallback incluye `kilocode/kilo/auto`; el descubrimiento en vivo de
  `https://api.kilo.ai/api/gateway/models` puede ampliar aún más el catálogo
  en tiempo de ejecución.
- El enrutamiento ascendente exacto detrás de `kilocode/kilo/auto` es propiedad de Kilo Gateway,
  no está codificado de forma rígida en OpenClaw.

Consulta [/providers/kilocode](/es/providers/kilocode) para conocer los detalles de configuración.

### Otros plugins de proveedor integrados

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Modelo de ejemplo: `openrouter/auto`
- OpenClaw aplica los encabezados de atribución de aplicación documentados por OpenRouter solo cuando
  la solicitud realmente se dirige a `openrouter.ai`
- Los marcadores `cache_control` específicos de Anthropic para OpenRouter también están limitados a
  rutas OpenRouter verificadas, no a URLs proxy arbitrarias
- OpenRouter permanece en la ruta estilo proxy compatible con OpenAI, por lo que el ajuste de solicitudes exclusivo de OpenAI nativo (`serviceTier`, `store` de Responses,
  sugerencias de caché de prompt, cargas útiles de compatibilidad de razonamiento de OpenAI) no se reenvía
- Las referencias de OpenRouter respaldadas por Gemini conservan solo el saneamiento de firma de pensamiento de Gemini proxy;
  la validación de reproducción nativa de Gemini y las reescrituras de bootstrap permanecen desactivadas
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Modelo de ejemplo: `kilocode/kilo/auto`
- Las referencias de Kilo respaldadas por Gemini conservan la misma ruta de saneamiento de firma de pensamiento de Gemini proxy; `kilocode/kilo/auto` y otras sugerencias sin soporte de razonamiento proxy
  omiten la inyección de razonamiento proxy
- MiniMax: `minimax` (clave API) y `minimax-portal` (OAuth)
- Autenticación: `MINIMAX_API_KEY` para `minimax`; `MINIMAX_OAUTH_TOKEN` o `MINIMAX_API_KEY` para `minimax-portal`
- Modelo de ejemplo: `minimax/MiniMax-M2.7` o `minimax-portal/MiniMax-M2.7`
- La incorporación/configuración con clave API de MiniMax escribe definiciones explícitas del modelo M2.7 con
  `input: ["text", "image"]`; el catálogo integrado del proveedor mantiene las referencias de chat
  solo como texto hasta que se materializa esa configuración del proveedor
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Modelo de ejemplo: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` o `KIMICODE_API_KEY`)
- Modelo de ejemplo: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Modelo de ejemplo: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` o `DASHSCOPE_API_KEY`)
- Modelo de ejemplo: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Modelo de ejemplo: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Modelos de ejemplo: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Modelo de ejemplo: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Modelo de ejemplo: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` o `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Modelo de ejemplo: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Modelo de ejemplo: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Las solicitudes nativas integradas a xAI usan la ruta xAI Responses
  - `/fast` o `params.fastMode: true` reescriben `grok-3`, `grok-3-mini`,
    `grok-4` y `grok-4-0709` a sus variantes `*-fast`
  - `tool_stream` está activado de forma predeterminada; configura
    `agents.defaults.models["xai/<model>"].params.tool_stream` en `false` para
    desactivarlo
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Modelo de ejemplo: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - Los modelos GLM en Cerebras usan los identificadores `zai-glm-4.7` y `zai-glm-4.6`.
  - URL base compatible con OpenAI: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Modelo de ejemplo de Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Consulta [Hugging Face (Inference)](/es/providers/huggingface).

## Proveedores mediante `models.providers` (personalizado/URL base)

Usa `models.providers` (o `models.json`) para agregar proveedores **personalizados** o
proxies compatibles con OpenAI/Anthropic.

Muchos de los plugins de proveedor integrados a continuación ya publican un catálogo predeterminado.
Usa entradas explícitas `models.providers.<id>` solo cuando quieras anular la
URL base, los encabezados o la lista de modelos predeterminados.

### Moonshot AI (Kimi)

Moonshot se distribuye como un plugin de proveedor integrado. Usa el proveedor integrado de
forma predeterminada y agrega una entrada explícita `models.providers.moonshot` solo cuando
necesites anular la URL base o los metadatos del modelo:

- Proveedor: `moonshot`
- Autenticación: `MOONSHOT_API_KEY`
- Modelo de ejemplo: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` o `openclaw onboard --auth-choice moonshot-api-key-cn`

Identificadores de modelo Kimi K2:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding usa el endpoint compatible con Anthropic de Moonshot AI:

- Proveedor: `kimi`
- Autenticación: `KIMI_API_KEY`
- Modelo de ejemplo: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

El identificador de modelo heredado `kimi/k2p5` sigue aceptándose como identificador de compatibilidad.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) proporciona acceso a Doubao y otros modelos en China.

- Proveedor: `volcengine` (coding: `volcengine-plan`)
- Autenticación: `VOLCANO_ENGINE_API_KEY`
- Modelo de ejemplo: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

La incorporación usa de forma predeterminada la superficie de coding, pero el catálogo general `volcengine/*`
se registra al mismo tiempo.

En los selectores de incorporación/configuración de modelos, la opción de autenticación de Volcengine prioriza tanto las filas `volcengine/*` como `volcengine-plan/*`. Si esos modelos aún no se han cargado,
OpenClaw vuelve al catálogo sin filtrar en lugar de mostrar un selector vacío
limitado al proveedor.

Modelos disponibles:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Modelos de coding (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (internacional)

BytePlus ARK proporciona acceso a los mismos modelos que Volcano Engine para usuarios internacionales.

- Proveedor: `byteplus` (coding: `byteplus-plan`)
- Autenticación: `BYTEPLUS_API_KEY`
- Modelo de ejemplo: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

La incorporación usa de forma predeterminada la superficie de coding, pero el catálogo general `byteplus/*`
se registra al mismo tiempo.

En los selectores de incorporación/configuración de modelos, la opción de autenticación de BytePlus prioriza tanto
las filas `byteplus/*` como `byteplus-plan/*`. Si esos modelos aún no se han cargado,
OpenClaw vuelve al catálogo sin filtrar en lugar de mostrar un selector vacío
limitado al proveedor.

Modelos disponibles:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Modelos de coding (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic proporciona modelos compatibles con Anthropic detrás del proveedor `synthetic`:

- Proveedor: `synthetic`
- Autenticación: `SYNTHETIC_API_KEY`
- Modelo de ejemplo: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax se configura mediante `models.providers` porque usa endpoints personalizados:

- MiniMax OAuth (Global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax clave API (Global): `--auth-choice minimax-global-api`
- MiniMax clave API (CN): `--auth-choice minimax-cn-api`
- Autenticación: `MINIMAX_API_KEY` para `minimax`; `MINIMAX_OAUTH_TOKEN` o
  `MINIMAX_API_KEY` para `minimax-portal`

Consulta [/providers/minimax](/es/providers/minimax) para conocer los detalles de configuración, las opciones de modelo y los fragmentos de configuración.

En la ruta de streaming compatible con Anthropic de MiniMax, OpenClaw desactiva el razonamiento de forma predeterminada
a menos que lo configures explícitamente, y `/fast on` reescribe
`MiniMax-M2.7` a `MiniMax-M2.7-highspeed`.

División de capacidades controladas por plugin:

- Los valores predeterminados de texto/chat permanecen en `minimax/MiniMax-M2.7`
- La generación de imágenes es `minimax/image-01` o `minimax-portal/image-01`
- La comprensión de imágenes es `MiniMax-VL-01` controlado por plugin en ambas rutas de autenticación de MiniMax
- La búsqueda web permanece en el identificador de proveedor `minimax`

### Ollama

Ollama se distribuye como un plugin de proveedor integrado y usa la API nativa de Ollama:

- Proveedor: `ollama`
- Autenticación: no requerida (servidor local)
- Modelo de ejemplo: `ollama/llama3.3`
- Instalación: [https://ollama.com/download](https://ollama.com/download)

```bash
# Instala Ollama y luego descarga un modelo:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama se detecta localmente en `http://127.0.0.1:11434` cuando optas por usarlo con
`OLLAMA_API_KEY`, y el plugin de proveedor integrado agrega Ollama directamente a
`openclaw onboard` y al selector de modelos. Consulta [/providers/ollama](/es/providers/ollama)
para ver incorporación, modo cloud/local y configuración personalizada.

### vLLM

vLLM se distribuye como un plugin de proveedor integrado para servidores locales/alojados por el usuario
compatibles con OpenAI:

- Proveedor: `vllm`
- Autenticación: opcional (depende de tu servidor)
- URL base predeterminada: `http://127.0.0.1:8000/v1`

Para habilitar el descubrimiento automático localmente (cualquier valor funciona si tu servidor no exige autenticación):

```bash
export VLLM_API_KEY="vllm-local"
```

Luego define un modelo (sustitúyelo por uno de los identificadores devueltos por `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Consulta [/providers/vllm](/es/providers/vllm) para obtener más detalles.

### SGLang

SGLang se distribuye como un plugin de proveedor integrado para servidores rápidos alojados por el usuario
compatibles con OpenAI:

- Proveedor: `sglang`
- Autenticación: opcional (depende de tu servidor)
- URL base predeterminada: `http://127.0.0.1:30000/v1`

Para habilitar el descubrimiento automático localmente (cualquier valor funciona si tu servidor no
exige autenticación):

```bash
export SGLANG_API_KEY="sglang-local"
```

Luego define un modelo (sustitúyelo por uno de los identificadores devueltos por `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Consulta [/providers/sglang](/es/providers/sglang) para obtener más detalles.

### Proxies locales (LM Studio, vLLM, LiteLLM, etc.)

Ejemplo (compatible con OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Notas:

- Para proveedores personalizados, `reasoning`, `input`, `cost`, `contextWindow` y `maxTokens` son opcionales.
  Cuando se omiten, OpenClaw usa estos valores predeterminados:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Recomendado: configura valores explícitos que coincidan con los límites de tu proxy/modelo.
- Para `api: "openai-completions"` en endpoints no nativos (cualquier `baseUrl` no vacía cuyo host no sea `api.openai.com`), OpenClaw fuerza `compat.supportsDeveloperRole: false` para evitar errores 400 del proveedor por roles `developer` no compatibles.
- Las rutas estilo proxy compatibles con OpenAI también omiten el ajuste de solicitudes exclusivo de OpenAI nativo: sin `service_tier`, sin `store` de Responses, sin sugerencias de caché de prompt, sin
  ajuste de carga útil de compatibilidad de razonamiento de OpenAI y sin encabezados
  ocultos de atribución de OpenClaw.
- Si `baseUrl` está vacía/se omite, OpenClaw mantiene el comportamiento predeterminado de OpenAI (que se resuelve en `api.openai.com`).
- Por seguridad, una configuración explícita `compat.supportsDeveloperRole: true` sigue siendo anulada en endpoints no nativos `openai-completions`.

## Ejemplos de CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Consulta también: [/gateway/configuration](/es/gateway/configuration) para ver ejemplos completos de configuración.

## Relacionado

- [Models](/es/concepts/models) — configuración de modelos y alias
- [Model Failover](/es/concepts/model-failover) — cadenas de fallback y comportamiento de reintento
- [Configuration Reference](/es/gateway/configuration-reference#agent-defaults) — claves de configuración de modelos
- [Providers](/es/providers) — guías de configuración por proveedor
