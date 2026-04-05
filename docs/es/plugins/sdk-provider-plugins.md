---
read_when:
    - Estás creando un nuevo plugin de proveedor de modelos
    - Quieres añadir un proxy compatible con OpenAI o un LLM personalizado a OpenClaw
    - Necesitas entender la autenticación del proveedor, los catálogos y los hooks de tiempo de ejecución
sidebarTitle: Provider Plugins
summary: Guía paso a paso para crear un plugin de proveedor de modelos para OpenClaw
title: Creación de plugins de proveedor
x-i18n:
    generated_at: "2026-04-05T12:50:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: e781e5fc436b2189b9f8cc63e7611f49df1fd2526604a0596a0631f49729b085
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Creación de plugins de proveedor

Esta guía te lleva paso a paso por la creación de un plugin de proveedor que añade un proveedor de modelos
(LLM) a OpenClaw. Al final tendrás un proveedor con un catálogo de modelos,
autenticación por clave API y resolución dinámica de modelos.

<Info>
  Si todavía no has creado ningún plugin de OpenClaw, lee primero
  [Primeros pasos](/plugins/building-plugins) para conocer la estructura básica del
  paquete y la configuración del manifiesto.
</Info>

## Recorrido

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
    credenciales sin cargar el tiempo de ejecución de tu plugin. `modelSupport` es opcional
    y permite que OpenClaw cargue automáticamente tu plugin de proveedor a partir de identificadores abreviados de modelo
    como `acme-large` antes de que existan hooks de tiempo de ejecución. Si publicas el
    proveedor en ClawHub, esos campos `openclaw.compat` y `openclaw.build`
    son obligatorios en `package.json`.

  </Step>

  <Step title="Registrar el proveedor">
    Un proveedor mínimo necesita `id`, `label`, `auth` y `catalog`:

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

    Ese es un proveedor funcional. Las personas usuarias ya pueden usar
    `openclaw onboard --acme-ai-api-key <key>` y seleccionar
    `acme-ai/acme-large` como su modelo.

    Para proveedores incluidos que solo registran un proveedor de texto con autenticación
    por clave API más un único tiempo de ejecución respaldado por catálogo, prefiere el helper más específico
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
    el modelo predeterminado del agente durante el onboarding, usa los helpers predefinidos de
    `openclaw/plugin-sdk/provider-onboard`. Los helpers más específicos son
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` y
    `createModelCatalogPresetAppliers(...)`.

    Cuando un endpoint nativo del proveedor admite bloques de uso en streaming en la ruta
    normal `openai-completions`, prefiere los helpers de catálogo compartidos en
    `openclaw/plugin-sdk/provider-catalog-shared` en lugar de codificar comprobaciones por id de proveedor. `supportsNativeStreamingUsageCompat(...)` y
    `applyProviderNativeStreamingUsageCompat(...)` detectan la compatibilidad a partir del mapa de capacidades del
    endpoint, de modo que los endpoints nativos estilo Moonshot/DashScope sigan
    participando incluso cuando un plugin use un id de proveedor personalizado.

  </Step>

  <Step title="Añadir resolución dinámica de modelos">
    Si tu proveedor acepta identificadores de modelo arbitrarios (como un proxy o router),
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
    calentamiento asíncrono; `resolveDynamicModel` se ejecuta de nuevo cuando termina.

  </Step>

  <Step title="Añadir hooks de tiempo de ejecución (según sea necesario)">
    La mayoría de proveedores solo necesitan `catalog` + `resolveDynamicModel`. Añade hooks
    de forma incremental según lo requiera tu proveedor.

    Los builders de helpers compartidos ahora cubren las familias más comunes de reproducción/compatibilidad de herramientas,
    por lo que los plugins normalmente no necesitan cablear cada hook manualmente uno a uno:

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

    Familias de reproducción disponibles hoy:

    | Familia | Qué incorpora |
    | --- | --- |
    | `openai-compatible` | Política compartida de reproducción estilo OpenAI para transportes compatibles con OpenAI, incluido saneamiento de id de llamadas de herramientas, correcciones de orden asistente-primero y validación genérica de turnos Gemini donde el transporte lo necesita |
    | `anthropic-by-model` | Política de reproducción compatible con Claude elegida por `modelId`, de modo que los transportes Anthropic-message solo reciben limpieza de bloques de razonamiento específica de Claude cuando el modelo resuelto es realmente un id de Claude |
    | `google-gemini` | Política nativa de reproducción Gemini más saneamiento de reproducción de bootstrap y modo de salida de razonamiento etiquetado |
    | `passthrough-gemini` | Saneamiento de firmas de pensamiento Gemini para modelos Gemini que se ejecutan a través de transportes proxy compatibles con OpenAI; no habilita validación de reproducción nativa Gemini ni reescrituras de bootstrap |
    | `hybrid-anthropic-openai` | Política híbrida para proveedores que mezclan superficies de modelos Anthropic-message y compatibles con OpenAI en un solo plugin; la eliminación opcional de bloques de razonamiento solo para Claude queda limitada al lado Anthropic |

    Ejemplos reales incluidos:

    - `google` y `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` y `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` y `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` y `zai`: `openai-compatible`

    Familias de stream disponibles hoy:

    | Familia | Qué incorpora |
    | --- | --- |
    | `google-thinking` | Normalización de carga útil de pensamiento Gemini en la ruta de stream compartida |
    | `kilocode-thinking` | Envoltorio de razonamiento Kilo en la ruta compartida de stream proxy, con `kilo/auto` e ids de razonamiento proxy no compatibles omitiendo el pensamiento inyectado |
    | `moonshot-thinking` | Mapeo binario de carga útil de pensamiento nativo Moonshot a partir de la configuración + nivel `/think` |
    | `minimax-fast-mode` | Reescritura de modelo fast-mode de MiniMax en la ruta compartida de stream |
    | `openai-responses-defaults` | Envoltorios compartidos nativos de OpenAI/Codex Responses: encabezados de atribución, `/fast`/`serviceTier`, verbosidad de texto, búsqueda web nativa de Codex, conformación de carga útil de compatibilidad de razonamiento y gestión de contexto de Responses |
    | `openrouter-thinking` | Envoltorio de razonamiento de OpenRouter para rutas proxy, con omisiones de modelo no compatible/`auto` gestionadas de forma centralizada |
    | `tool-stream-default-on` | Envoltorio `tool_stream` activado por defecto para proveedores como z.ai que quieren streaming de herramientas salvo que se desactive explícitamente |

    Ejemplos reales incluidos:

    - `google` y `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` y `minimax-portal`: `minimax-fast-mode`
    - `openai` y `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared` también exporta la enumeración de familias de reproducción
    además de los helpers compartidos sobre los que se construyen esas familias. Las
    exportaciones públicas comunes incluyen:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - builders de reproducción compartidos como `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` y
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - helpers de reproducción Gemini como `sanitizeGoogleGeminiReplayHistory(...)`
      y `resolveTaggedReasoningOutputMode()`
    - helpers de endpoint/modelo como `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` y
      `normalizeNativeXaiModelId(...)`

    `openclaw/plugin-sdk/provider-stream` expone tanto el builder de familias como
    los helpers públicos de envoltorio que esas familias reutilizan. Las exportaciones públicas
    comunes incluyen:

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

    Algunos helpers de stream se mantienen intencionalmente locales al proveedor. Ejemplo
    actual incluido: `@openclaw/anthropic-provider` exporta
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` y los
    builders de envoltorio Anthropic de nivel inferior desde su punto de unión público `api.ts` /
    `contract-api.ts`. Esos helpers siguen siendo específicos de Anthropic porque
    también codifican el manejo beta de OAuth de Claude y la restricción `context1m`.

    Otros proveedores incluidos también mantienen envoltorios específicos del transporte como locales cuando
    el comportamiento no se comparte limpiamente entre familias. Ejemplo actual: el
    plugin xAI incluido mantiene la conformación nativa de xAI Responses en su propio
    `wrapStreamFn`, incluida reescritura de alias `/fast`, `tool_stream`
    predeterminado, limpieza de herramientas estrictas no compatibles y eliminación de
    carga útil de razonamiento específica de xAI.

    `openclaw/plugin-sdk/provider-tools` actualmente expone una familia compartida
    de esquema de herramientas más helpers compartidos de esquema/compatibilidad:

    - `ProviderToolCompatFamily` documenta hoy el inventario de familias compartidas.
    - `buildProviderToolCompatFamilyHooks("gemini")` conecta limpieza
      de esquema Gemini + diagnósticos para proveedores que necesitan esquemas de herramientas seguros para Gemini.
    - `normalizeGeminiToolSchemas(...)` e `inspectGeminiToolSchemas(...)`
      son los helpers públicos subyacentes de esquema Gemini.
    - `resolveXaiModelCompatPatch()` devuelve el parche de compatibilidad incluido de xAI:
      `toolSchemaProfile: "xai"`, palabras clave de esquema no compatibles, compatibilidad
      nativa con `web_search` y decodificación de argumentos de llamadas de herramientas con entidades HTML.
    - `applyXaiModelCompat(model)` aplica ese mismo parche de compatibilidad de xAI a un
      modelo resuelto antes de que llegue al ejecutor.

    Ejemplo real incluido: el plugin xAI usa `normalizeResolvedModel` más
    `contributeResolvedModelCompat` para mantener esos metadatos de compatibilidad como propiedad del
    proveedor en lugar de codificar reglas de xAI en el núcleo.

    El mismo patrón de raíz de paquete también respalda a otros proveedores incluidos:

    - `@openclaw/openai-provider`: `api.ts` exporta builders de proveedor,
      helpers de modelo predeterminado y builders de proveedor de tiempo real
    - `@openclaw/openrouter-provider`: `api.ts` exporta el builder del proveedor
      más helpers de onboarding/configuración

    <Tabs>
      <Tab title="Intercambio de token">
        Para proveedores que necesitan un intercambio de token antes de cada llamada de inferencia:

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
      <Tab title="Identidad nativa del transporte">
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
      OpenClaw llama a los hooks en este orden. La mayoría de proveedores solo usan 2-3:

      | # | Hook | Cuándo usarlo |
      | --- | --- | --- |
      | 1 | `catalog` | Catálogo de modelos o valores predeterminados de `baseUrl` |
      | 2 | `applyConfigDefaults` | Valores predeterminados globales del proveedor durante la materialización de configuración |
      | 3 | `normalizeModelId` | Limpieza de alias de id de modelo heredados/preview antes de la búsqueda |
      | 4 | `normalizeTransport` | Limpieza de `api` / `baseUrl` de la familia del proveedor antes del ensamblado genérico del modelo |
      | 5 | `normalizeConfig` | Normalizar configuración `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | Reescrituras de compatibilidad de uso en streaming nativo para proveedores de configuración |
      | 7 | `resolveConfigApiKey` | Resolución de autenticación de marcador de entorno propiedad del proveedor |
      | 8 | `resolveSyntheticAuth` | Autenticación sintética local/autohospedada o respaldada por configuración |
      | 9 | `shouldDeferSyntheticProfileAuth` | Rebajar los marcadores de perfil almacenado sintéticos detrás de autenticación env/config |
      | 10 | `resolveDynamicModel` | Aceptar ids arbitrarios de modelo upstream |
      | 11 | `prepareDynamicModel` | Obtención asíncrona de metadatos antes de resolver |
      | 12 | `normalizeResolvedModel` | Reescrituras de transporte antes del ejecutor |

    Notas sobre el respaldo en tiempo de ejecución:

    - `normalizeConfig` comprueba primero el proveedor coincidente y luego otros
      plugins de proveedor capaces de usar hooks hasta que uno realmente cambia la configuración.
      Si ningún hook de proveedor reescribe una entrada compatible de la familia Google, la
      normalización de configuración de Google incluida sigue aplicándose.
    - `resolveConfigApiKey` usa el hook del proveedor cuando se expone. La ruta
      incluida `amazon-bedrock` también tiene aquí un resolvedor integrado de marcadores de entorno de AWS,
      aunque la autenticación de tiempo de ejecución de Bedrock en sí sigue usando la cadena
      predeterminada del SDK de AWS.
      | 13 | `contributeResolvedModelCompat` | Indicadores de compatibilidad para modelos de proveedor detrás de otro transporte compatible |
      | 14 | `capabilities` | Bolsa estática heredada de capacidades; solo compatibilidad |
      | 15 | `normalizeToolSchemas` | Limpieza de esquema de herramientas propiedad del proveedor antes del registro |
      | 16 | `inspectToolSchemas` | Diagnósticos de esquema de herramientas propiedad del proveedor |
      | 17 | `resolveReasoningOutputMode` | Contrato de salida de razonamiento etiquetado frente a nativo |
      | 18 | `prepareExtraParams` | Parámetros de solicitud predeterminados |
      | 19 | `createStreamFn` | Transporte `StreamFn` totalmente personalizado |
      | 20 | `wrapStreamFn` | Envoltorios personalizados de encabezados/cuerpo en la ruta de stream normal |
      | 21 | `resolveTransportTurnState` | Encabezados/metadatos nativos por turno |
      | 22 | `resolveWebSocketSessionPolicy` | Encabezados nativos de sesión WS/enfriamiento |
      | 23 | `formatApiKey` | Forma personalizada del token en tiempo de ejecución |
      | 24 | `refreshOAuth` | Actualización personalizada de OAuth |
      | 25 | `buildAuthDoctorHint` | Guía de reparación de autenticación |
      | 26 | `matchesContextOverflowError` | Detección de desbordamiento propiedad del proveedor |
      | 27 | `classifyFailoverReason` | Clasificación de límite de tasa/sobrecarga propiedad del proveedor |
      | 28 | `isCacheTtlEligible` | Restricción de TTL de caché de prompts |
      | 29 | `buildMissingAuthMessage` | Pista personalizada de autenticación faltante |
      | 30 | `suppressBuiltInModel` | Ocultar filas upstream obsoletas |
      | 31 | `augmentModelCatalog` | Filas sintéticas de compatibilidad futura |
      | 32 | `isBinaryThinking` | Pensamiento binario activado/desactivado |
      | 33 | `supportsXHighThinking` | Compatibilidad con razonamiento `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | Política predeterminada de `/think` |
      | 35 | `isModernModelRef` | Coincidencia de modelo live/smoke |
      | 36 | `prepareRuntimeAuth` | Intercambio de token antes de la inferencia |
      | 37 | `resolveUsageAuth` | Análisis personalizado de credenciales de uso |
      | 38 | `fetchUsageSnapshot` | Endpoint personalizado de uso |
      | 39 | `createEmbeddingProvider` | Adaptador de embeddings propiedad del proveedor para memoria/búsqueda |
      | 40 | `buildReplayPolicy` | Política personalizada de reproducción/compactación de transcripciones |
      | 41 | `sanitizeReplayHistory` | Reescrituras de reproducción específicas del proveedor después de la limpieza genérica |
      | 42 | `validateReplayTurns` | Validación estricta de turnos de reproducción antes del ejecutor integrado |
      | 43 | `onModelSelected` | Callback posterior a la selección (por ejemplo, telemetría) |

      Para descripciones detalladas y ejemplos reales, consulta
      [Internals: Provider Runtime Hooks](/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="Añadir capacidades extra (opcional)">
    <a id="step-5-add-extra-capabilities"></a>
    Un plugin de proveedor puede registrar voz, transcripción en tiempo real, voz en tiempo real,
    comprensión multimedia, generación de imágenes, generación de video, recuperación web
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
          maxVideos: 1,
          maxDurationSeconds: 10,
          supportsResolution: true,
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
    [Internals: Capability Ownership](/plugins/architecture#capability-ownership-model).

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
├── package.json              # metadatos openclaw.providers
├── openclaw.plugin.json      # Manifiesto con providerAuthEnvVars
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Pruebas
    └── usage.ts              # Endpoint de uso (opcional)
```

## Referencia del orden del catálogo

`catalog.order` controla cuándo se fusiona tu catálogo en relación con los
proveedores integrados:

| Orden     | Cuándo       | Caso de uso                                     |
| --------- | ------------ | ----------------------------------------------- |
| `simple`  | Primera pasada | Proveedores simples con clave API            |
| `profile` | Después de simple | Proveedores restringidos por perfiles de autenticación |
| `paired`  | Después de profile | Sintetizar múltiples entradas relacionadas |
| `late`    | Última pasada | Anular proveedores existentes (gana en colisión) |

## Siguientes pasos

- [Plugins de canal](/plugins/sdk-channel-plugins) — si tu plugin también proporciona un canal
- [Tiempo de ejecución del SDK](/plugins/sdk-runtime) — helpers `api.runtime` (TTS, búsqueda, subagente)
- [Resumen del SDK](/plugins/sdk-overview) — referencia completa de importaciones de subrutas
- [Internals de plugins](/plugins/architecture#provider-runtime-hooks) — detalles de hooks y ejemplos incluidos
