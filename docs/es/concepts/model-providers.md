---
read_when:
    - Necesitas una referencia de configuración de modelos proveedor por proveedor
    - Quieres configuraciones de ejemplo o comandos de onboarding de la CLI para proveedores de modelos
summary: Resumen de proveedores de modelos con configuraciones de ejemplo + flujos de la CLI
title: Proveedores de modelos
x-i18n:
    generated_at: "2026-04-05T12:41:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5d8f56a2a5319de03f7b86e7b19b9a89e7023f757930b5b5949568f680352a3a
    source_path: concepts/model-providers.md
    workflow: 15
---

# Proveedores de modelos

Esta página cubre los **proveedores de LLM/modelos** (no canales de chat como WhatsApp/Telegram).
Para las reglas de selección de modelos, consulta [/concepts/models](/concepts/models).

## Reglas rápidas

- Las referencias de modelos usan `provider/model` (ejemplo: `opencode/claude-opus-4-6`).
- Si configuras `agents.defaults.models`, se convierte en la allowlist.
- Helpers de la CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Las reglas de tiempo de ejecución para fallback, las sondas de cooldown y la persistencia de sobrescrituras de sesión están
  documentadas en [/concepts/model-failover](/concepts/model-failover).
- `models.providers.*.models[].contextWindow` es metadato nativo del modelo;
  `models.providers.*.models[].contextTokens` es el límite efectivo de tiempo de ejecución.
- Los plugins de proveedores pueden inyectar catálogos de modelos mediante `registerProvider({ catalog })`;
  OpenClaw fusiona esa salida en `models.providers` antes de escribir
  `models.json`.
- Los manifiestos de proveedores pueden declarar `providerAuthEnvVars` para que las
  sondas genéricas de autenticación basadas en variables de entorno no necesiten cargar el tiempo de ejecución del plugin. El mapa restante de variables de entorno del núcleo
  ahora es solo para proveedores no plugin/del núcleo y algunos casos de precedencia genérica
  como el onboarding con prioridad en clave API de Anthropic.
- Los plugins de proveedores también pueden gestionar el comportamiento de tiempo de ejecución del proveedor mediante
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
- Nota: `capabilities` del tiempo de ejecución del proveedor es metadato compartido del runner (familia del proveedor,
  peculiaridades de transcripción/herramientas, pistas de transporte/caché). No es lo
  mismo que el [modelo público de capacidades](/plugins/architecture#public-capability-model)
  que describe lo que registra un plugin (inferencia de texto, voz, etc.).

## Comportamiento del proveedor gestionado por el plugin

Los plugins de proveedores ahora pueden gestionar la mayor parte de la lógica específica del proveedor mientras que OpenClaw mantiene
el bucle genérico de inferencia.

División típica:

- `auth[].run` / `auth[].runNonInteractive`: el proveedor gestiona los flujos
  de onboarding/login para `openclaw onboard`, `openclaw models auth` y la configuración headless
- `wizard.setup` / `wizard.modelPicker`: el proveedor gestiona las etiquetas de elección de autenticación,
  alias heredados, pistas de allowlist de onboarding y entradas de configuración en los selectores de onboarding/modelos
- `catalog`: el proveedor aparece en `models.providers`
- `normalizeModelId`: el proveedor normaliza ids heredados/de vista previa del modelo antes de la
  búsqueda o canonicalización
- `normalizeTransport`: el proveedor normaliza `api` / `baseUrl` de la familia de transporte
  antes del ensamblado genérico del modelo; OpenClaw comprueba primero el proveedor coincidente,
  luego otros plugins de proveedores con capacidad de hook hasta que uno realmente cambie el
  transporte
- `normalizeConfig`: el proveedor normaliza la configuración `models.providers.<id>` antes de que
  el tiempo de ejecución la use; OpenClaw comprueba primero el proveedor coincidente, luego otros
  plugins de proveedores con capacidad de hook hasta que uno realmente cambie la configuración. Si ningún
  hook de proveedor reescribe la configuración, los helpers integrados de la familia Google aún
  normalizan las entradas compatibles de proveedores Google.
- `applyNativeStreamingUsageCompat`: el proveedor aplica reescrituras de compatibilidad de uso de streaming nativo impulsadas por el endpoint para proveedores de configuración
- `resolveConfigApiKey`: el proveedor resuelve autenticación con marcadores de entorno para proveedores de configuración
  sin forzar la carga completa de la autenticación de tiempo de ejecución. `amazon-bedrock` también tiene aquí
  un resolvedor integrado de marcadores de entorno AWS, aunque la autenticación de tiempo de ejecución de Bedrock usa
  la cadena predeterminada del SDK de AWS.
- `resolveSyntheticAuth`: el proveedor puede exponer disponibilidad de autenticación
  local/autohospedada u otra basada en configuración sin persistir secretos en texto plano
- `shouldDeferSyntheticProfileAuth`: el proveedor puede marcar placeholders sintéticos de perfiles almacenados
  como de menor precedencia que la autenticación basada en env/config
- `resolveDynamicModel`: el proveedor acepta ids de modelo aún no presentes en el
  catálogo estático local
- `prepareDynamicModel`: el proveedor necesita una actualización de metadatos antes de reintentar la
  resolución dinámica
- `normalizeResolvedModel`: el proveedor necesita reescrituras de transporte o base URL
- `contributeResolvedModelCompat`: el proveedor aporta flags de compatibilidad para sus
  modelos del proveedor incluso cuando llegan mediante otro transporte compatible
- `capabilities`: el proveedor publica peculiaridades de transcripción/herramientas/familia de proveedor
- `normalizeToolSchemas`: el proveedor limpia esquemas de herramientas antes de que el runner
  integrado los vea
- `inspectToolSchemas`: el proveedor muestra advertencias de esquema específicas del transporte
  después de la normalización
- `resolveReasoningOutputMode`: el proveedor elige contratos de salida de razonamiento
  nativos frente a etiquetados
- `prepareExtraParams`: el proveedor establece valores predeterminados o normaliza parámetros de solicitud por modelo
- `createStreamFn`: el proveedor reemplaza la ruta normal de streaming con un
  transporte totalmente personalizado
- `wrapStreamFn`: el proveedor aplica wrappers de compatibilidad de cabeceras/cuerpo/modelo a las solicitudes
- `resolveTransportTurnState`: el proveedor suministra cabeceras o metadatos
  nativos de transporte por turno
- `resolveWebSocketSessionPolicy`: el proveedor suministra cabeceras de sesión
  WebSocket nativas o política de cool-down de sesión
- `createEmbeddingProvider`: el proveedor gestiona el comportamiento de embeddings de memoria cuando
  corresponde al plugin del proveedor en lugar del selector central de embeddings del núcleo
- `formatApiKey`: el proveedor formatea perfiles de autenticación almacenados en la
  cadena `apiKey` de tiempo de ejecución esperada por el transporte
- `refreshOAuth`: el proveedor gestiona la actualización de OAuth cuando los
  actualizadores compartidos de `pi-ai` no son suficientes
- `buildAuthDoctorHint`: el proveedor añade orientación de reparación cuando falla la
  actualización de OAuth
- `matchesContextOverflowError`: el proveedor reconoce errores específicos del proveedor
  por desbordamiento de ventana de contexto que las heurísticas genéricas pasarían por alto
- `classifyFailoverReason`: el proveedor mapea errores en bruto específicos del proveedor del transporte/API
  a motivos de failover como límite de tasa o sobrecarga
- `isCacheTtlEligible`: el proveedor decide qué ids de modelo upstream admiten TTL de caché de prompts
- `buildMissingAuthMessage`: el proveedor reemplaza el error genérico del almacén de autenticación
  por una pista de recuperación específica del proveedor
- `suppressBuiltInModel`: el proveedor oculta filas upstream obsoletas y puede devolver un
  error gestionado por el proveedor para fallos de resolución directa
- `augmentModelCatalog`: el proveedor añade filas sintéticas/finales al catálogo después del
  descubrimiento y la fusión de configuración
- `isBinaryThinking`: el proveedor gestiona la UX binaria de thinking activado/desactivado
- `supportsXHighThinking`: el proveedor habilita `xhigh` para modelos seleccionados
- `resolveDefaultThinkingLevel`: el proveedor gestiona la política predeterminada de `/think` para una
  familia de modelos
- `applyConfigDefaults`: el proveedor aplica valores predeterminados globales específicos del proveedor
  durante la materialización de la configuración según el modo de autenticación, el entorno o la familia del modelo
- `isModernModelRef`: el proveedor gestiona la coincidencia de modelos preferidos live/smoke
- `prepareRuntimeAuth`: el proveedor convierte una credencial configurada en un token de tiempo
  de ejecución de corta duración
- `resolveUsageAuth`: el proveedor resuelve credenciales de uso/cuota para `/usage`
  y otras superficies relacionadas de estado/informes
- `fetchUsageSnapshot`: el proveedor gestiona la obtención/análisis del endpoint de uso mientras
  el núcleo sigue gestionando la estructura de resumen y el formato
- `onModelSelected`: el proveedor ejecuta efectos secundarios posteriores a la selección, como
  telemetría o registro de sesión gestionado por el proveedor

Ejemplos integrados actuales:

- `anthropic`: fallback de compatibilidad futura para Claude 4.6, pistas de reparación de autenticación, obtención del
  endpoint de uso, metadatos de TTL de caché/familia de proveedor y valores predeterminados globales de configuración
  sensibles a la autenticación
- `amazon-bedrock`: coincidencia de desbordamiento de contexto gestionada por el proveedor y clasificación
  de motivos de failover para errores específicos de Bedrock de throttling/no listo, además
  de la familia compartida de reproducción `anthropic-by-model` para protecciones de política de reproducción solo de Claude en tráfico Anthropic
- `anthropic-vertex`: protecciones de política de reproducción solo de Claude en tráfico de mensajes Anthropic
- `openrouter`: ids de modelo de paso directo, wrappers de solicitud, pistas de capacidad del proveedor,
  saneamiento de firmas de thinking de Gemini en tráfico Gemini por proxy, inyección de razonamiento por proxy mediante la familia de stream `openrouter-thinking`, reenvío de metadatos de enrutamiento y política de TTL de caché
- `github-copilot`: onboarding/login de dispositivo, fallback de modelo con compatibilidad futura,
  pistas de transcripción de thinking de Claude, intercambio de tokens de tiempo de ejecución y obtención del
  endpoint de uso
- `openai`: fallback de compatibilidad futura para GPT-5.4, normalización de transporte OpenAI directo,
  pistas de autenticación faltante con reconocimiento de Codex, supresión de Spark, filas sintéticas de catálogo de OpenAI/Codex, política de thinking/modelo live, normalización de alias de tokens de uso
  (`input` / `output` y familias `prompt` / `completion`), la familia compartida de stream `openai-responses-defaults` para wrappers nativos de OpenAI/Codex y metadatos de familia de proveedor
- `google` y `google-gemini-cli`: fallback de compatibilidad futura para Gemini 3.1,
  validación nativa de reproducción de Gemini, saneamiento de bootstrap replay, modo etiquetado
  de salida de razonamiento y coincidencia de modelos modernos; el OAuth de Gemini CLI también gestiona
  el formateo del token de perfil de autenticación, el análisis de tokens de uso y la obtención del endpoint de cuota para superficies de uso
- `moonshot`: transporte compartido, normalización de carga útil de thinking gestionada por plugin
- `kilocode`: transporte compartido, cabeceras de solicitud gestionadas por plugin, normalización de carga útil de razonamiento,
  saneamiento de firmas de thinking de Gemini por proxy y política de TTL de caché
- `zai`: fallback de compatibilidad futura para GLM-5, valores predeterminados de `tool_stream`, política de TTL de
  caché, política binaria de thinking/modelo live y autenticación de uso + obtención de cuota;
  los ids desconocidos `glm-5*` se sintetizan a partir de la plantilla integrada `glm-4.7`
- `xai`: normalización nativa de transporte Responses, reescrituras de alias `/fast` para
  variantes rápidas de Grok, `tool_stream` predeterminado y limpieza específica de xAI de esquemas de herramientas /
  cargas útiles de razonamiento
- `mistral`: metadatos de capacidades gestionados por plugin
- `opencode` y `opencode-go`: metadatos de capacidades gestionados por plugin más
  saneamiento de firmas de thinking de Gemini por proxy
- `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi`,
  `nvidia`, `qianfan`, `stepfun`, `synthetic`, `together`, `venice`,
  `vercel-ai-gateway` y `volcengine`: solo catálogos gestionados por plugin
- `qwen`: catálogos gestionados por plugin para modelos de texto más registros compartidos de proveedores
  de comprensión de medios y generación de video para sus superficies multimodales; la generación de video de Qwen usa los endpoints estándar de video DashScope con modelos Wan integrados como `wan2.6-t2v` y `wan2.7-r2v`
- `minimax`: catálogos gestionados por plugin, selección híbrida de política de reproducción Anthropic/OpenAI y lógica de autenticación/instantánea de uso
- `xiaomi`: catálogos gestionados por plugin más lógica de autenticación/instantánea de uso

El plugin integrado `openai` ahora gestiona ambos ids de proveedor: `openai` y
`openai-codex`.

Eso cubre a los proveedores que todavía encajan en los transportes normales de OpenClaw. Un proveedor
que necesite un ejecutor de solicitudes totalmente personalizado es una superficie de extensión
independiente y más profunda.

## Rotación de claves API

- Admite rotación genérica de proveedores para proveedores seleccionados.
- Configura varias claves mediante:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (sobrescritura live única, máxima prioridad)
  - `<PROVIDER>_API_KEYS` (lista separada por comas o punto y coma)
  - `<PROVIDER>_API_KEY` (clave principal)
  - `<PROVIDER>_API_KEY_*` (lista numerada, por ejemplo `<PROVIDER>_API_KEY_1`)
- Para proveedores de Google, `GOOGLE_API_KEY` también se incluye como respaldo.
- El orden de selección de claves preserva la prioridad y elimina duplicados.
- Las solicitudes se reintentan con la siguiente clave solo en respuestas de límite de tasa (por
  ejemplo `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` o mensajes periódicos de límite de uso).
- Los fallos que no son de límite de tasa fallan inmediatamente; no se intenta ninguna rotación de claves.
- Cuando fallan todas las claves candidatas, se devuelve el error final del último intento.

## Proveedores integrados (catálogo pi-ai)

OpenClaw se distribuye con el catálogo pi-ai. Estos proveedores no requieren
configuración `models.providers`; solo establece la autenticación y elige un modelo.

### OpenAI

- Proveedor: `openai`
- Autenticación: `OPENAI_API_KEY`
- Rotación opcional: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, más `OPENCLAW_LIVE_OPENAI_KEY` (sobrescritura única)
- Modelos de ejemplo: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- El transporte predeterminado es `auto` (primero WebSocket, respaldo SSE)
- Sobrescribe por modelo mediante `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- El calentamiento de WebSocket de OpenAI Responses está activado de forma predeterminada mediante `params.openaiWsWarmup` (`true`/`false`)
- El procesamiento prioritario de OpenAI puede activarse mediante `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` y `params.fastMode` asignan solicitudes Responses directas `openai/*` a `service_tier=priority` en `api.openai.com`
- Usa `params.serviceTier` cuando quieras un nivel explícito en lugar del interruptor compartido `/fast`
- Las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`,
  `User-Agent`) se aplican solo en tráfico nativo de OpenAI hacia `api.openai.com`, no en
  proxies genéricos compatibles con OpenAI
- Las rutas nativas de OpenAI también conservan `store` de Responses, pistas de caché de prompt y modelado de carga útil de compatibilidad de razonamiento de OpenAI; las rutas proxy no
- `openai/gpt-5.3-codex-spark` se suprime intencionadamente en OpenClaw porque la API live de OpenAI lo rechaza; Spark se trata solo como Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Proveedor: `anthropic`
- Autenticación: `ANTHROPIC_API_KEY`
- Rotación opcional: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, más `OPENCLAW_LIVE_ANTHROPIC_KEY` (sobrescritura única)
- Modelo de ejemplo: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey` o `openclaw onboard --auth-choice anthropic-cli`
- Las solicitudes Anthropic públicas directas admiten el interruptor compartido `/fast` y `params.fastMode`, incluido el tráfico autenticado por clave API y OAuth enviado a `api.anthropic.com`; OpenClaw lo asigna a Anthropic `service_tier` (`auto` frente a `standard_only`)
- Nota de facturación: la documentación pública de Claude Code de Anthropic todavía incluye el uso directo de Claude Code en terminal dentro de los límites del plan Claude. Por separado, Anthropic notificó a los usuarios de OpenClaw el **4 de abril de 2026 a las 12:00 PM PT / 8:00 PM BST** que la ruta de login de Claude de **OpenClaw** cuenta como uso de harness de terceros y requiere **Extra Usage** facturado por separado de la suscripción.
- El token de configuración de Anthropic vuelve a estar disponible como ruta heredada/manual de OpenClaw. Úsalo entendiendo que Anthropic indicó a los usuarios de OpenClaw que esta ruta requiere **Extra Usage**.

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
- El transporte predeterminado es `auto` (primero WebSocket, respaldo SSE)
- Sobrescribe por modelo mediante `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- `params.serviceTier` también se reenvía en solicitudes nativas de Codex Responses (`chatgpt.com/backend-api`)
- Las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`,
  `User-Agent`) solo se adjuntan en tráfico nativo de Codex hacia
  `chatgpt.com/backend-api`, no en proxies genéricos compatibles con OpenAI
- Comparte el mismo interruptor `/fast` y la misma configuración `params.fastMode` que `openai/*` directo; OpenClaw lo asigna a `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` sigue disponible cuando el catálogo OAuth de Codex lo expone; depende de los permisos
- `openai-codex/gpt-5.4` mantiene `contextWindow = 1050000` nativo y un `contextTokens = 272000` de tiempo de ejecución predeterminado; sobrescribe el límite de tiempo de ejecución con `models.providers.openai-codex.models[].contextTokens`
- Nota de política: el OAuth de OpenAI Codex es explícitamente compatible con herramientas/flujos externos como OpenClaw.

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

### Otras opciones hospedadas de estilo suscripción

- [Qwen Cloud](/providers/qwen): superficie del proveedor Qwen Cloud más mapeo de endpoints de Alibaba DashScope y Coding Plan
- [MiniMax](/providers/minimax): acceso mediante OAuth o clave API de MiniMax Coding Plan
- [GLM Models](/providers/glm): Z.AI Coding Plan o endpoints API generales

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
- Rotación opcional: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, respaldo `GOOGLE_API_KEY` y `OPENCLAW_LIVE_GEMINI_KEY` (sobrescritura única)
- Modelos de ejemplo: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Compatibilidad: la configuración heredada de OpenClaw que usa `google/gemini-3.1-flash-preview` se normaliza a `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Las ejecuciones directas de Gemini también aceptan `agents.defaults.models["google/<model>"].params.cachedContent`
  (o el heredado `cached_content`) para reenviar un handle nativo del proveedor
  `cachedContents/...`; los aciertos de caché de Gemini aparecen como `cacheRead` de OpenClaw

### Google Vertex y Gemini CLI

- Proveedores: `google-vertex`, `google-gemini-cli`
- Autenticación: Vertex usa gcloud ADC; Gemini CLI usa su flujo OAuth
- Precaución: el OAuth de Gemini CLI en OpenClaw es una integración no oficial. Algunos usuarios han informado de restricciones en cuentas de Google después de usar clientes de terceros. Revisa los términos de Google y usa una cuenta no crítica si decides continuar.
- El OAuth de Gemini CLI se distribuye como parte del plugin integrado `google`.
  - Instala Gemini CLI primero:
    - `brew install gemini-cli`
    - o `npm install -g @google/gemini-cli`
  - Habilita: `openclaw plugins enable google`
  - Inicia sesión: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Modelo predeterminado: `google-gemini-cli/gemini-3.1-pro-preview`
  - Nota: **no** pegas un client id ni un secret en `openclaw.json`. El flujo de login de la CLI almacena
    tokens en perfiles de autenticación en el host del gateway.
  - Si las solicitudes fallan después del login, configura `GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host del gateway.
  - Las respuestas JSON de Gemini CLI se analizan desde `response`; el uso usa `stats` como respaldo, con `stats.cached` normalizado a `cacheRead` de OpenClaw.

### Z.AI (GLM)

- Proveedor: `zai`
- Autenticación: `ZAI_API_KEY`
- Modelo de ejemplo: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Alias: `z.ai/*` y `z-ai/*` se normalizan a `zai/*`
  - `zai-api-key` detecta automáticamente el endpoint Z.AI correspondiente; `zai-coding-global`, `zai-coding-cn`, `zai-global` y `zai-cn` fuerzan una superficie específica

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
- El catálogo estático de respaldo incluye `kilocode/kilo/auto`; el descubrimiento live de
  `https://api.kilo.ai/api/gateway/models` puede ampliar aún más el catálogo de tiempo de ejecución.
- El enrutamiento upstream exacto detrás de `kilocode/kilo/auto` es gestionado por Kilo Gateway,
  no está codificado de forma fija en OpenClaw.

Consulta [/providers/kilocode](/providers/kilocode) para ver los detalles de configuración.

### Otros plugins de proveedores integrados

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Modelo de ejemplo: `openrouter/auto`
- OpenClaw aplica las cabeceras documentadas de atribución de app de OpenRouter solo cuando
  la solicitud realmente apunta a `openrouter.ai`
- Los marcadores específicos de OpenRouter Anthropic `cache_control` también se limitan a
  rutas verificadas de OpenRouter, no a URL proxy arbitrarias
- OpenRouter sigue en la ruta de estilo proxy compatible con OpenAI, por lo que el modelado de solicitudes exclusivo de OpenAI nativo (`serviceTier`, `store` de Responses,
  pistas de caché de prompt, cargas útiles de compatibilidad de razonamiento de OpenAI) no se reenvía
- Las referencias de OpenRouter respaldadas por Gemini conservan solo el saneamiento de firmas de thinking de Gemini por proxy; la validación nativa de reproducción de Gemini y las reescrituras de bootstrap permanecen desactivadas
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Modelo de ejemplo: `kilocode/kilo/auto`
- Las referencias de Kilo respaldadas por Gemini conservan la misma ruta de saneamiento de firmas de thinking de Gemini por proxy; `kilocode/kilo/auto` y otras pistas de proxy sin compatibilidad de razonamiento omiten la inyección de razonamiento por proxy
- MiniMax: `minimax` (clave API) y `minimax-portal` (OAuth)
- Autenticación: `MINIMAX_API_KEY` para `minimax`; `MINIMAX_OAUTH_TOKEN` o `MINIMAX_API_KEY` para `minimax-portal`
- Modelo de ejemplo: `minimax/MiniMax-M2.7` o `minimax-portal/MiniMax-M2.7`
- La configuración de onboarding/clave API de MiniMax escribe definiciones explícitas del modelo M2.7 con
  `input: ["text", "image"]`; el catálogo integrado del proveedor mantiene las referencias de chat
  solo texto hasta que se materializa esa configuración del proveedor
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
  - Las solicitudes xAI nativas integradas usan la ruta xAI Responses
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
  - Los modelos GLM en Cerebras usan los ids `zai-glm-4.7` y `zai-glm-4.6`.
  - URL base compatible con OpenAI: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Modelo de ejemplo de Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Consulta [Hugging Face (Inference)](/providers/huggingface).

## Proveedores mediante `models.providers` (personalizado/base URL)

Usa `models.providers` (o `models.json`) para agregar proveedores **personalizados**
o proxies compatibles con OpenAI/Anthropic.

Muchos de los plugins de proveedores integrados a continuación ya publican un catálogo predeterminado.
Usa entradas explícitas `models.providers.<id>` solo cuando quieras sobrescribir la
URL base, las cabeceras o la lista de modelos predeterminadas.

### Moonshot AI (Kimi)

Moonshot se distribuye como plugin integrado de proveedor. Usa el proveedor integrado de forma
predeterminada y agrega una entrada explícita `models.providers.moonshot` solo cuando
necesites sobrescribir la URL base o los metadatos del modelo:

- Proveedor: `moonshot`
- Autenticación: `MOONSHOT_API_KEY`
- Modelo de ejemplo: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` o `openclaw onboard --auth-choice moonshot-api-key-cn`

Ids de modelos Kimi K2:

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

El heredado `kimi/k2p5` sigue aceptándose como id de compatibilidad del modelo.

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

El onboarding usa por defecto la superficie de coding, pero el catálogo general `volcengine/*`
se registra al mismo tiempo.

En los selectores de onboarding/configuración de modelos, la elección de autenticación de Volcengine prioriza ambas
filas `volcengine/*` y `volcengine-plan/*`. Si esos modelos aún no se han cargado,
OpenClaw usa como respaldo el catálogo sin filtrar en lugar de mostrar un selector vacío
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

El onboarding usa por defecto la superficie de coding, pero el catálogo general `byteplus/*`
se registra al mismo tiempo.

En los selectores de onboarding/configuración de modelos, la elección de autenticación de BytePlus prioriza ambas
filas `byteplus/*` y `byteplus-plan/*`. Si esos modelos aún no se han cargado,
OpenClaw usa como respaldo el catálogo sin filtrar en lugar de mostrar un selector vacío
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

- MiniMax OAuth (global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax clave API (global): `--auth-choice minimax-global-api`
- MiniMax clave API (CN): `--auth-choice minimax-cn-api`
- Autenticación: `MINIMAX_API_KEY` para `minimax`; `MINIMAX_OAUTH_TOKEN` o
  `MINIMAX_API_KEY` para `minimax-portal`

Consulta [/providers/minimax](/providers/minimax) para ver los detalles de configuración, las opciones de modelo y los fragmentos de configuración.

En la ruta de streaming compatible con Anthropic de MiniMax, OpenClaw desactiva thinking de
forma predeterminada salvo que lo configures explícitamente, y `/fast on` reescribe
`MiniMax-M2.7` a `MiniMax-M2.7-highspeed`.

División de capacidades gestionadas por plugin:

- Los valores predeterminados de texto/chat siguen en `minimax/MiniMax-M2.7`
- La generación de imágenes es `minimax/image-01` o `minimax-portal/image-01`
- La comprensión de imágenes es `MiniMax-VL-01`, gestionada por plugin en ambas rutas de autenticación MiniMax
- La búsqueda web se mantiene en el id de proveedor `minimax`

### Ollama

Ollama se distribuye como plugin integrado de proveedor y usa la API nativa de Ollama:

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

Ollama se detecta localmente en `http://127.0.0.1:11434` cuando activas la opción con
`OLLAMA_API_KEY`, y el plugin integrado del proveedor agrega Ollama directamente a
`openclaw onboard` y al selector de modelos. Consulta [/providers/ollama](/providers/ollama)
para ver onboarding, modo cloud/local y configuración personalizada.

### vLLM

vLLM se distribuye como plugin integrado de proveedor para servidores locales/autohospedados
compatibles con OpenAI:

- Proveedor: `vllm`
- Autenticación: opcional (depende de tu servidor)
- URL base predeterminada: `http://127.0.0.1:8000/v1`

Para activar el autodescubrimiento localmente (cualquier valor funciona si tu servidor no exige autenticación):

```bash
export VLLM_API_KEY="vllm-local"
```

Luego configura un modelo (sustitúyelo por uno de los ids devueltos por `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Consulta [/providers/vllm](/providers/vllm) para obtener más información.

### SGLang

SGLang se distribuye como plugin integrado de proveedor para servidores autohospedados
compatibles con OpenAI y de alto rendimiento:

- Proveedor: `sglang`
- Autenticación: opcional (depende de tu servidor)
- URL base predeterminada: `http://127.0.0.1:30000/v1`

Para activar el autodescubrimiento localmente (cualquier valor funciona si tu servidor no
exige autenticación):

```bash
export SGLANG_API_KEY="sglang-local"
```

Luego configura un modelo (sustitúyelo por uno de los ids devueltos por `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Consulta [/providers/sglang](/providers/sglang) para obtener más información.

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
- Recomendado: establece valores explícitos que coincidan con los límites de tu proxy/modelo.
- Para `api: "openai-completions"` en endpoints no nativos (cualquier `baseUrl` no vacía cuyo host no sea `api.openai.com`), OpenClaw fuerza `compat.supportsDeveloperRole: false` para evitar errores 400 del proveedor por roles `developer` no compatibles.
- Las rutas proxy de estilo OpenAI-compatible también omiten el modelado de solicitudes exclusivo de OpenAI nativo: sin `service_tier`, sin `store` de Responses, sin pistas de caché de prompt, sin modelado de carga útil de compatibilidad de razonamiento de OpenAI y sin cabeceras ocultas de atribución de OpenClaw.
- Si `baseUrl` está vacía u omitida, OpenClaw mantiene el comportamiento predeterminado de OpenAI (que resuelve a `api.openai.com`).
- Por seguridad, un `compat.supportsDeveloperRole: true` explícito sigue siendo sobrescrito en endpoints no nativos `openai-completions`.

## Ejemplos de CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Consulta también: [/gateway/configuration](/gateway/configuration) para ver ejemplos completos de configuración.

## Relacionado

- [Models](/concepts/models) — configuración y alias de modelos
- [Model Failover](/concepts/model-failover) — cadenas de fallback y comportamiento de reintento
- [Configuration Reference](/gateway/configuration-reference#agent-defaults) — claves de configuración del modelo
- [Providers](/providers) — guías de configuración por proveedor
