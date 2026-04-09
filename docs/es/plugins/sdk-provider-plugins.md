---
read_when:
    - Estás creando un nuevo plugin de proveedor de modelos
    - Quieres añadir a OpenClaw un proxy compatible con OpenAI o un LLM personalizado
    - Necesitas entender la autenticación del proveedor, los catálogos y los hooks de runtime
sidebarTitle: Provider Plugins
summary: Guía paso a paso para crear un plugin de proveedor de modelos para OpenClaw
title: Crear plugins de proveedor
x-i18n:
    generated_at: "2026-04-09T01:30:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 38d9af522dc19e49c81203a83a4096f01c2398b1df771c848a30ad98f251e9e1
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Crear plugins de proveedor

Esta guía explica cómo crear un plugin de proveedor que añade un proveedor de modelos
(LLM) a OpenClaw. Al final tendrás un proveedor con un catálogo de modelos,
autenticación mediante clave de API y resolución dinámica de modelos.

<Info>
  Si aún no has creado ningún plugin de OpenClaw, lee primero
  [Getting Started](/es/plugins/building-plugins) para conocer la estructura básica
  del paquete y la configuración del manifiesto.
</Info>

## Tutorial

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paquete y manifiesto">
    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model provider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "providerAuthEnvVars": {
        "acme-ai": ["ACME_AI_API_KEY"]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API key",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API key"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    El manifiesto declara `providerAuthEnvVars` para que OpenClaw pueda detectar
    credenciales sin cargar el runtime de tu plugin. Añade `providerAuthAliases`
    cuando una variante del proveedor deba reutilizar la autenticación de otro id de proveedor. `modelSupport`
    es opcional y permite que OpenClaw cargue automáticamente tu plugin de proveedor a partir de ids
    abreviados de modelo como `acme-large` antes de que existan hooks de runtime. Si publicas el
    proveedor en ClawHub, esos campos `openclaw.compat` y `openclaw.build`
    son obligatorios en `package.json`.

  </Step>

  <Step title="Registrar el proveedor">
    Un proveedor mínimo necesita un `id`, `label`, `auth` y `catalog`:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });
      },
    });
    ```

    Ese es un proveedor funcional. Ahora los usuarios pueden
    ejecutar `openclaw onboard --acme-ai-api-key <key>` y seleccionar
    `acme-ai/acme-large` como su modelo.

    Para proveedores incluidos que solo registran un proveedor de texto con
    autenticación por clave de API más un único runtime respaldado por catálogo, es preferible
    usar el helper más específico
    `defineSingleProviderPluginEntry(...)`:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    Si tu flujo de autenticación también necesita aplicar parches a `models.providers.*`, alias y
    al modelo predeterminado del agente durante la incorporación, usa los helpers predefinidos de
    `openclaw/plugin-sdk/provider-onboard`. Los helpers más específicos son
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` y
    `createModelCatalogPresetAppliers(...)`.

    Cuando el endpoint nativo de un proveedor admite bloques de uso en streaming en el
    transporte normal `openai-completions`, es preferible usar los helpers compartidos de catálogo en
    `openclaw/plugin-sdk/provider-catalog-shared` en lugar de codificar comprobaciones
    del id del proveedor. `supportsNativeStreamingUsageCompat(...)` y
    `applyProviderNativeStreamingUsageCompat(...)` detectan la compatibilidad a partir del mapa de capacidades
    del endpoint, de modo que los endpoints nativos tipo Moonshot/DashScope sigan
    activándose incluso cuando un plugin esté usando un id de proveedor personalizado.

  </Step>

  <Step title="Añadir resolución dinámica de modelos">
    Si tu proveedor acepta ids de modelo arbitrarios (como un proxy o router),
    añade `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    Si la resolución requiere una llamada de red, usa `prepareDynamicModel` para el
    calentamiento asíncrono; `resolveDynamicModel` se ejecuta de nuevo cuando se completa.

  </Step>

  <Step title="Añadir hooks de runtime (según sea necesario)">
    La mayoría de los proveedores solo necesitan `catalog` + `resolveDynamicModel`. Añade hooks
    de forma incremental según lo requiera tu proveedor.

    Los generadores de helpers compartidos ahora cubren las familias más habituales de
    reproducción/compatibilidad con herramientas, así que normalmente los plugins no necesitan
    conectar manualmente cada hook uno por uno:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Familias de reproducción disponibles actualmente:

    | Family | What it wires in |
    | --- | --- |
    | `openai-compatible` | Política compartida de reproducción de estilo OpenAI para transportes compatibles con OpenAI, incluida la limpieza de `tool-call-id`, correcciones de orden con el asistente primero y validación genérica de turnos de Gemini cuando el transporte lo necesita |
    | `anthropic-by-model` | Política de reproducción compatible con Claude elegida por `modelId`, de modo que los transportes `anthropic-message` solo reciban limpieza de bloques de razonamiento específica de Claude cuando el modelo resuelto sea realmente un id de Claude |
    | `google-gemini` | Política nativa de reproducción de Gemini más saneamiento de reproducción bootstrap y modo etiquetado de salida de razonamiento |
    | `passthrough-gemini` | Saneamiento de firmas de pensamiento de Gemini para modelos Gemini que se ejecutan mediante transportes proxy compatibles con OpenAI; no habilita validación nativa de reproducción de Gemini ni reescrituras de bootstrap |
    | `hybrid-anthropic-openai` | Política híbrida para proveedores que mezclan superficies de modelos `anthropic-message` y compatibles con OpenAI en un solo plugin; la eliminación opcional de bloques de razonamiento solo para Claude permanece limitada al lado Anthropic |

    Ejemplos reales incluidos:

    - `google` y `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` y `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` y `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` y `zai`: `openai-compatible`

    Familias de stream disponibles actualmente:

    | Family | What it wires in |
    | --- | --- |
    | `google-thinking` | Normalización de carga útil de pensamiento de Gemini en la ruta de stream compartida |
    | `kilocode-thinking` | Envoltorio de razonamiento de Kilo en la ruta de stream proxy compartida, con `kilo/auto` e ids de razonamiento proxy no compatibles omitiendo el pensamiento inyectado |
    | `moonshot-thinking` | Asignación de carga útil binaria nativa de pensamiento de Moonshot a partir de la configuración y del nivel `/think` |
    | `minimax-fast-mode` | Reescritura de modelos de modo rápido de MiniMax en la ruta de stream compartida |
    | `openai-responses-defaults` | Envoltorios compartidos de Responses nativas de OpenAI/Codex: encabezados de atribución, `/fast`/`serviceTier`, verbosidad del texto, búsqueda web nativa de Codex, ajuste de carga útil compatible con razonamiento y gestión de contexto de Responses |
    | `openrouter-thinking` | Envoltorio de razonamiento de OpenRouter para rutas proxy, con omisiones para modelos no compatibles/`auto` gestionadas de forma centralizada |
    | `tool-stream-default-on` | Envoltorio `tool_stream` activado por defecto para proveedores como Z.AI que quieren streaming de herramientas salvo que se desactive explícitamente |

    Ejemplos reales incluidos:

    - `google` y `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` y `minimax-portal`: `minimax-fast-mode`
    - `openai` y `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared` también exporta el enum de familias de reproducción
    junto con los helpers compartidos sobre los que se construyen esas familias. Las
    exportaciones públicas habituales incluyen:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - generadores compartidos de reproducción como `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` y
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - helpers de reproducción de Gemini como `sanitizeGoogleGeminiReplayHistory(...)`
      y `resolveTaggedReasoningOutputMode()`
    - helpers de endpoint/modelo como `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` y
      `normalizeNativeXaiModelId(...)`

    `openclaw/plugin-sdk/provider-stream` expone tanto el generador de familias como
    los helpers públicos de envoltorio que reutilizan esas familias. Las
    exportaciones públicas habituales incluyen:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - envoltorios compartidos de OpenAI/Codex como
      `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)` y
      `createCodexNativeWebSearchWrapper(...)`
    - envoltorios compartidos de proxy/proveedor como `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)` y `createMinimaxFastModeWrapper(...)`

    Algunos helpers de stream se mantienen locales al proveedor de forma intencionada. Ejemplo
    actual incluido: `@openclaw/anthropic-provider` exporta
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` y los
    generadores de envoltorio Anthropic de nivel inferior desde su punto de unión público `api.ts` /
    `contract-api.ts`. Esos helpers siguen siendo específicos de Anthropic porque
    también codifican el manejo beta de OAuth de Claude y el control de `context1m`.

    Otros proveedores incluidos también mantienen envoltorios específicos de transporte como locales cuando
    el comportamiento no se comparte limpiamente entre familias. Ejemplo actual: el
    plugin xAI incluido mantiene el ajuste nativo de Responses de xAI en su propio
    `wrapStreamFn`, incluyendo reescrituras de alias `/fast`, `tool_stream` predeterminado,
    limpieza estricta de herramientas no compatibles y eliminación de carga útil
    de razonamiento específica de xAI.

    `openclaw/plugin-sdk/provider-tools` actualmente expone una familia compartida
    de esquemas de herramientas más helpers compartidos de esquema/compatibilidad:

    - `ProviderToolCompatFamily` documenta hoy el inventario de familias compartidas.
    - `buildProviderToolCompatFamilyHooks("gemini")` conecta limpieza de esquemas Gemini
      + diagnósticos para proveedores que necesitan esquemas de herramientas seguros para Gemini.
    - `normalizeGeminiToolSchemas(...)` y `inspectGeminiToolSchemas(...)`
      son los helpers públicos subyacentes para esquemas Gemini.
    - `resolveXaiModelCompatPatch()` devuelve el parche de compatibilidad xAI incluido:
      `toolSchemaProfile: "xai"`, palabras clave de esquema no compatibles, compatibilidad nativa con
      `web_search` y decodificación de argumentos de llamadas a herramientas con entidades HTML.
    - `applyXaiModelCompat(model)` aplica ese mismo parche de compatibilidad xAI a un
      modelo resuelto antes de que llegue al ejecutor.

    Ejemplo real incluido: el plugin xAI usa `normalizeResolvedModel` más
    `contributeResolvedModelCompat` para mantener esos metadatos de compatibilidad bajo
    propiedad del proveedor en lugar de codificar reglas xAI en el núcleo.

    El mismo patrón de raíz de paquete también respalda otros proveedores incluidos:

    - `@openclaw/openai-provider`: `api.ts` exporta generadores de proveedor,
      helpers de modelo predeterminado y generadores de proveedores realtime
    - `@openclaw/openrouter-provider`: `api.ts` exporta el generador de proveedor
      más helpers de incorporación/configuración

    <Tabs>
      <Tab title="Intercambio de tokens">
        Para proveedores que necesitan un intercambio de tokens antes de cada llamada de inferencia:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="Encabezados personalizados">
        Para proveedores que necesitan encabezados de solicitud personalizados o modificaciones del cuerpo:

        ```typescript
        // wrapStreamFn returns a StreamFn derived from ctx.streamFn
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="Identidad de transporte nativa">
        Para proveedores que necesitan encabezados o metadatos nativos de solicitud/sesión en
        transportes HTTP o WebSocket genéricos:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="Uso y facturación">
        Para proveedores que exponen datos de uso/facturación:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```
      </Tab>
    </Tabs>

    <Accordion title="Todos los hooks de proveedor disponibles">
      OpenClaw llama a los hooks en este orden. La mayoría de los proveedores solo usan 2 o 3:

      | # | Hook | Cuándo usarlo |
      | --- | --- | --- |
      | 1 | `catalog` | Catálogo de modelos o valores predeterminados de `baseUrl` |
      | 2 | `applyConfigDefaults` | Valores predeterminados globales propiedad del proveedor durante la materialización de la configuración |
      | 3 | `normalizeModelId` | Limpieza de alias heredados/de vista previa de ids de modelo antes de la búsqueda |
      | 4 | `normalizeTransport` | Limpieza de familia del proveedor `api` / `baseUrl` antes del ensamblado genérico del modelo |
      | 5 | `normalizeConfig` | Normalizar configuración `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | Reescrituras de compatibilidad de uso nativo en streaming para proveedores de configuración |
      | 7 | `resolveConfigApiKey` | Resolución de autenticación de marcadores de entorno propiedad del proveedor |
      | 8 | `resolveSyntheticAuth` | Autenticación sintética local/autohospedada o basada en configuración |
      | 9 | `shouldDeferSyntheticProfileAuth` | Rebajar placeholders sintéticos de perfiles almacenados por detrás de autenticación env/config |
      | 10 | `resolveDynamicModel` | Aceptar ids de modelo arbitrarios del upstream |
      | 11 | `prepareDynamicModel` | Obtención asíncrona de metadatos antes de resolver |
      | 12 | `normalizeResolvedModel` | Reescrituras de transporte antes del ejecutor |

      Notas sobre el fallback en runtime:

      - `normalizeConfig` primero comprueba el proveedor coincidente y después otros
        plugins de proveedor con capacidad de hooks hasta que uno realmente cambie la configuración.
        Si ningún hook de proveedor reescribe una entrada de configuración compatible de la familia Google,
        se sigue aplicando el normalizador de configuración de Google incluido.
      - `resolveConfigApiKey` usa el hook del proveedor cuando se expone. La ruta incluida
        de `amazon-bedrock` también tiene aquí un resolvedor integrado de marcadores de entorno AWS,
        aunque la autenticación de runtime de Bedrock siga usando la cadena predeterminada del SDK de AWS.
      | 13 | `contributeResolvedModelCompat` | Indicadores de compatibilidad para modelos de proveedor detrás de otro transporte compatible |
      | 14 | `capabilities` | Bolsa estática heredada de capacidades; solo compatibilidad |
      | 15 | `normalizeToolSchemas` | Limpieza de esquemas de herramientas propiedad del proveedor antes del registro |
      | 16 | `inspectToolSchemas` | Diagnósticos de esquemas de herramientas propiedad del proveedor |
      | 17 | `resolveReasoningOutputMode` | Contrato de salida de razonamiento etiquetado frente a nativo |
      | 18 | `prepareExtraParams` | Parámetros predeterminados de la solicitud |
      | 19 | `createStreamFn` | Transporte `StreamFn` completamente personalizado |
      | 20 | `wrapStreamFn` | Envoltorios personalizados de encabezados/cuerpo en la ruta de stream normal |
      | 21 | `resolveTransportTurnState` | Encabezados/metadatos nativos por turno |
      | 22 | `resolveWebSocketSessionPolicy` | Encabezados/enfriamiento de sesión WS nativos |
      | 23 | `formatApiKey` | Forma personalizada del token de runtime |
      | 24 | `refreshOAuth` | Actualización OAuth personalizada |
      | 25 | `buildAuthDoctorHint` | Orientación de reparación de autenticación |
      | 26 | `matchesContextOverflowError` | Detección de desbordamiento propiedad del proveedor |
      | 27 | `classifyFailoverReason` | Clasificación propiedad del proveedor de límite de tasa/sobrecarga |
      | 28 | `isCacheTtlEligible` | Control TTL de la caché de prompts |
      | 29 | `buildMissingAuthMessage` | Sugerencia personalizada de autenticación faltante |
      | 30 | `suppressBuiltInModel` | Ocultar filas upstream obsoletas |
      | 31 | `augmentModelCatalog` | Filas sintéticas de compatibilidad futura |
      | 32 | `isBinaryThinking` | Pensamiento binario activado/desactivado |
      | 33 | `supportsXHighThinking` | Compatibilidad con razonamiento `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | Política predeterminada de `/think` |
      | 35 | `isModernModelRef` | Coincidencia de modelos live/smoke |
      | 36 | `prepareRuntimeAuth` | Intercambio de tokens antes de la inferencia |
      | 37 | `resolveUsageAuth` | Análisis personalizado de credenciales de uso |
      | 38 | `fetchUsageSnapshot` | Endpoint personalizado de uso |
      | 39 | `createEmbeddingProvider` | Adaptador de embeddings propiedad del proveedor para memoria/búsqueda |
      | 40 | `buildReplayPolicy` | Política personalizada de reproducción/compactación de transcripciones |
      | 41 | `sanitizeReplayHistory` | Reescrituras de reproducción específicas del proveedor tras la limpieza genérica |
      | 42 | `validateReplayTurns` | Validación estricta de turnos de reproducción antes del ejecutor embebido |
      | 43 | `onModelSelected` | Callback posterior a la selección (p. ej., telemetría) |

      Nota sobre ajuste de prompts:

      - `resolveSystemPromptContribution` permite que un proveedor inyecte
        orientación de system prompt consciente de la caché para una familia de modelos. Es preferible usarlo en lugar de
        `before_prompt_build` cuando el comportamiento pertenece a una familia de proveedor/modelo
        y debe conservar la división estable/dinámica de la caché.

      Para descripciones detalladas y ejemplos del mundo real, consulta
      [Internals: Provider Runtime Hooks](/es/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="Añadir capacidades adicionales (opcional)">
    <a id="step-5-add-extra-capabilities"></a>
    Un plugin de proveedor puede registrar speech, transcripción realtime, voz
    realtime, comprensión multimedia, generación de imágenes, generación de video, web fetch
    y búsqueda web junto con la inferencia de texto:

    ```typescript
    register(api) {
      api.registerProvider({ id: "acme-ai", /* ... */ });

      api.registerSpeechProvider({
        id: "acme-ai",
        label: "Acme Speech",
        isConfigured: ({ config }) => Boolean(config.messages?.tts),
        synthesize: async (req) => ({
          audioBuffer: Buffer.from(/* PCM data */),
          outputFormat: "mp3",
          fileExtension: ".mp3",
          voiceCompatible: false,
        }),
      });

      api.registerRealtimeTranscriptionProvider({
        id: "acme-ai",
        label: "Acme Realtime Transcription",
        isConfigured: () => true,
        createSession: (req) => ({
          connect: async () => {},
          sendAudio: () => {},
          close: () => {},
          isConnected: () => true,
        }),
      });

      api.registerRealtimeVoiceProvider({
        id: "acme-ai",
        label: "Acme Realtime Voice",
        isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
        createBridge: (req) => ({
          connect: async () => {},
          sendAudio: () => {},
          setMediaTimestamp: () => {},
          submitToolResult: () => {},
          acknowledgeMark: () => {},
          close: () => {},
          isConnected: () => true,
        }),
      });

      api.registerMediaUnderstandingProvider({
        id: "acme-ai",
        capabilities: ["image", "audio"],
        describeImage: async (req) => ({ text: "A photo of..." }),
        transcribeAudio: async (req) => ({ text: "Transcript..." }),
      });

      api.registerImageGenerationProvider({
        id: "acme-ai",
        label: "Acme Images",
        generate: async (req) => ({ /* image result */ }),
      });

      api.registerVideoGenerationProvider({
        id: "acme-ai",
        label: "Acme Video",
        capabilities: {
          generate: {
            maxVideos: 1,
            maxDurationSeconds: 10,
            supportsResolution: true,
          },
          imageToVideo: {
            enabled: true,
            maxVideos: 1,
            maxInputImages: 1,
            maxDurationSeconds: 5,
          },
          videoToVideo: {
            enabled: false,
          },
        },
        generateVideo: async (req) => ({ videos: [] }),
      });

      api.registerWebFetchProvider({
        id: "acme-ai-fetch",
        label: "Acme Fetch",
        hint: "Fetch pages through Acme's rendering backend.",
        envVars: ["ACME_FETCH_API_KEY"],
        placeholder: "acme-...",
        signupUrl: "https://acme.example.com/fetch",
        credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
        getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
        setCredentialValue: (fetchConfigTarget, value) => {
          const acme = (fetchConfigTarget.acme ??= {});
          acme.apiKey = value;
        },
        createTool: () => ({
          description: "Fetch a page through Acme Fetch.",
          parameters: {},
          execute: async (args) => ({ content: [] }),
        }),
      });

      api.registerWebSearchProvider({
        id: "acme-ai-search",
        label: "Acme Search",
        search: async (req) => ({ content: [] }),
      });
    }
    ```

    OpenClaw clasifica esto como un plugin de **capacidad híbrida**. Este es el
    patrón recomendado para plugins de empresa (un plugin por proveedor). Consulta
    [Internals: Capability Ownership](/es/plugins/architecture#capability-ownership-model).

    Para la generación de video, es preferible usar la forma de capacidades con reconocimiento de modo que se muestra arriba:
    `generate`, `imageToVideo` y `videoToVideo`. Los campos agregados planos como
    `maxInputImages`, `maxInputVideos` y `maxDurationSeconds` no son
    suficientes para anunciar limpiamente compatibilidad con modo de transformación o modos deshabilitados.

    Los proveedores de generación de música deben seguir el mismo patrón:
    `generate` para generación solo a partir de prompt y `edit` para generación
    basada en imagen de referencia. Los campos agregados planos como `maxInputImages`,
    `supportsLyrics` y `supportsFormat` no son suficientes para anunciar compatibilidad
    con edición; el contrato esperado son bloques explícitos `generate` / `edit`.

  </Step>

  <Step title="Probar">
    <a id="step-6-test"></a>
    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Export your provider config object from index.ts or a dedicated file
    import { acmeProvider } from "./provider.js";

    describe("acme-ai provider", () => {
      it("resolves dynamic models", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("returns catalog when key is available", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("returns null catalog when no key", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## Publicar en ClawHub

Los plugins de proveedor se publican igual que cualquier otro plugin de código externo:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

No uses aquí el alias heredado de publicación solo para Skills; los paquetes de plugins deben usar
`clawhub package publish`.

## Estructura de archivos

```
<bundled-plugin-root>/acme-ai/
├── package.json              # metadatos de openclaw.providers
├── openclaw.plugin.json      # manifiesto con metadatos de autenticación del proveedor
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # pruebas
    └── usage.ts              # endpoint de uso (opcional)
```

## Referencia del orden del catálogo

`catalog.order` controla cuándo se fusiona tu catálogo en relación con los
proveedores integrados:

| Order     | When          | Use case                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | Primer paso   | Proveedores sencillos con clave de API          |
| `profile` | Después de simple  | Proveedores condicionados a perfiles de autenticación |
| `paired`  | Después de profile | Sintetizar varias entradas relacionadas         |
| `late`    | Último paso   | Sobrescribir proveedores existentes (gana en colisión) |

## Próximos pasos

- [Channel Plugins](/es/plugins/sdk-channel-plugins) — si tu plugin también proporciona un canal
- [SDK Runtime](/es/plugins/sdk-runtime) — helpers `api.runtime` (TTS, búsqueda, subagent)
- [SDK Overview](/es/plugins/sdk-overview) — referencia completa de importaciones por subruta
- [Plugin Internals](/es/plugins/architecture#provider-runtime-hooks) — detalles de hooks y ejemplos incluidos
