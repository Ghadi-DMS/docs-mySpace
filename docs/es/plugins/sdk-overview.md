---
read_when:
    - Necesitas saber desde qué subruta del SDK importar
    - Quieres una referencia de todos los métodos de registro en OpenClawPluginApi
    - Buscas una exportación específica del SDK
sidebarTitle: SDK Overview
summary: Mapa de importación, referencia de la API de registro y arquitectura del SDK
title: Resumen del SDK de plugins
x-i18n:
    generated_at: "2026-04-05T12:50:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0d7d8b6add0623766d36e81588ae783b525357b2f5245c38c8e2b07c5fc1d2b5
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Resumen del SDK de plugins

El SDK de plugins es el contrato tipado entre los plugins y el núcleo. Esta página es la
referencia para **qué importar** y **qué puedes registrar**.

<Tip>
  **¿Buscas una guía práctica?**
  - ¿Tu primer plugin? Empieza con [Getting Started](/plugins/building-plugins)
  - ¿Un plugin de canal? Consulta [Channel Plugins](/plugins/sdk-channel-plugins)
  - ¿Un plugin de proveedor? Consulta [Provider Plugins](/plugins/sdk-provider-plugins)
</Tip>

## Convención de importación

Importa siempre desde una subruta específica:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subruta es un módulo pequeño e independiente. Esto mantiene un arranque rápido y
evita problemas de dependencias circulares. Para helpers de entrada/compilación específicos de canal,
prefiere `openclaw/plugin-sdk/channel-core`; deja `openclaw/plugin-sdk/core` para
la superficie paraguas más amplia y helpers compartidos como
`buildChannelConfigSchema`.

No añadas ni dependas de superficies de conveniencia con nombre de proveedor como
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` o
superficies helper con marca de canal. Los plugins incluidos deben componer
subrutas genéricas del SDK dentro de sus propios barrels `api.ts` o `runtime-api.ts`, y el núcleo
debe usar esos barrels locales del plugin o añadir un contrato estrecho y genérico del SDK
cuando la necesidad sea realmente transversal entre canales.

El mapa de exportación generado todavía contiene un pequeño conjunto de superficies
helper de plugins incluidos como `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Esas
subrutas existen solo para mantenimiento y compatibilidad de plugins incluidos; se
omiten intencionadamente de la tabla común de abajo y no son la ruta de
importación recomendada para nuevos plugins de terceros.

## Referencia de subrutas

Las subrutas más usadas, agrupadas por propósito. La lista completa generada de
más de 200 subrutas vive en `scripts/lib/plugin-sdk-entrypoints.json`.

Las subrutas helper reservadas para plugins incluidos siguen apareciendo en esa lista generada.
Trátalas como superficies de detalle de implementación/compatibilidad salvo que una página de documentación
promueva explícitamente una como pública.

### Entrada del plugin

| Subruta                    | Exportaciones clave                                                                                                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Subrutas de canal">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Exportación del esquema Zod raíz de `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, además de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers compartidos del asistente de configuración, prompts de listas de permitidos, constructores de estado de configuración |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de configuración/múltiples cuentas y compuertas de acciones, helpers de respaldo para cuenta predeterminada |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalización de ID de cuenta |
    | `plugin-sdk/account-resolution` | Helpers de búsqueda de cuentas + respaldo predeterminado |
    | `plugin-sdk/account-helpers` | Helpers limitados para lista de cuentas/acciones de cuenta |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipos de esquema de configuración de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalización/validación de comandos personalizados de Telegram con respaldo de contrato incluido |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers compartidos de rutas entrantes y construcción de sobres |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers compartidos para registrar y despachar entrada |
    | `plugin-sdk/messaging-targets` | Helpers de análisis/coincidencia de destinos |
    | `plugin-sdk/outbound-media` | Helpers compartidos para carga de medios salientes |
    | `plugin-sdk/outbound-runtime` | Helpers de identidad saliente y delegación de envío |
    | `plugin-sdk/thread-bindings-runtime` | Helpers de ciclo de vida de vinculaciones a hilos y adaptadores |
    | `plugin-sdk/agent-media-payload` | Constructor heredado de carga útil de medios del agente |
    | `plugin-sdk/conversation-runtime` | Helpers de conversación/vinculación a hilos, emparejamiento y vinculaciones configuradas |
    | `plugin-sdk/runtime-config-snapshot` | Helper de instantánea de configuración en tiempo de ejecución |
    | `plugin-sdk/runtime-group-policy` | Helpers de resolución de política de grupo en tiempo de ejecución |
    | `plugin-sdk/channel-status` | Helpers compartidos para instantáneas/resúmenes de estado de canal |
    | `plugin-sdk/channel-config-primitives` | Primitivas limitadas del esquema de configuración de canal |
    | `plugin-sdk/channel-config-writes` | Helpers de autorización de escrituras en configuración de canal |
    | `plugin-sdk/channel-plugin-common` | Exportaciones de preámbulo compartido de plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lectura/edición de configuración de listas de permitidos |
    | `plugin-sdk/group-access` | Helpers compartidos para decisiones de acceso a grupos |
    | `plugin-sdk/direct-dm` | Helpers compartidos de autenticación/protección de DM directos |
    | `plugin-sdk/interactive-runtime` | Helpers de normalización/reducción de cargas útiles de respuestas interactivas |
    | `plugin-sdk/channel-inbound` | Helpers de debounce, coincidencia de menciones y sobres |
    | `plugin-sdk/channel-send-result` | Tipos de resultados de respuesta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers de análisis/coincidencia de destinos |
    | `plugin-sdk/channel-contract` | Tipos de contrato de canal |
    | `plugin-sdk/channel-feedback` | Integración de feedback/reacciones |
  </Accordion>

  <Accordion title="Subrutas de proveedor">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers seleccionados de configuración para proveedores locales/autoalojados |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers enfocados para configuración de proveedores autoalojados compatibles con OpenAI |
    | `plugin-sdk/cli-backend` | Valores predeterminados del backend de CLI + constantes de watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers de resolución de claves de API en tiempo de ejecución para plugins de proveedor |
    | `plugin-sdk/provider-auth-api-key` | Helpers de onboarding/escritura de perfiles para claves de API |
    | `plugin-sdk/provider-auth-result` | Constructor estándar de resultados de autenticación OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers compartidos de inicio de sesión interactivo para plugins de proveedor |
    | `plugin-sdk/provider-env-vars` | Helpers de búsqueda de variables de entorno para autenticación de proveedor |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de políticas de repetición, helpers de endpoints de proveedor y de normalización de IDs de modelo como `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers genéricos de capacidades HTTP/endpoint de proveedor |
    | `plugin-sdk/provider-web-fetch` | Helpers de registro/caché para proveedores de web-fetch |
    | `plugin-sdk/provider-web-search` | Helpers de registro/caché/configuración para proveedores de web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquema Gemini y helpers de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` y similares |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de envolturas de stream y helpers compartidos de envolturas para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de parches de configuración para onboarding |
    | `plugin-sdk/global-singleton` | Helpers de singleton/mapa/caché locales al proceso |
  </Accordion>

  <Accordion title="Subrutas de autenticación y seguridad">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registro de comandos, helpers de autorización de remitentes |
    | `plugin-sdk/approval-auth-runtime` | Resolución de aprobadores y helpers de autenticación de acciones en el mismo chat |
    | `plugin-sdk/approval-client-runtime` | Helpers nativos de perfiles/filtros de aprobación de exec |
    | `plugin-sdk/approval-delivery-runtime` | Adaptadores nativos de capacidades/entrega de aprobaciones |
    | `plugin-sdk/approval-native-runtime` | Helpers nativos de destino de aprobación y vinculación de cuentas |
    | `plugin-sdk/approval-reply-runtime` | Helpers de carga útil de respuestas para aprobaciones de exec/plugin |
    | `plugin-sdk/command-auth-native` | Helpers nativos de autenticación de comandos + destino de sesión nativa |
    | `plugin-sdk/command-detection` | Helpers compartidos de detección de comandos |
    | `plugin-sdk/command-surface` | Helpers de normalización del cuerpo de comando y superficies de comandos |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | Helpers compartidos de confianza, control de DMs, contenido externo y recopilación de secretos |
    | `plugin-sdk/ssrf-policy` | Helpers de listas de permitidos de hosts y políticas SSRF de red privada |
    | `plugin-sdk/ssrf-runtime` | Helpers de dispatcher fijado, fetch protegido por SSRF y política SSRF |
    | `plugin-sdk/secret-input` | Helpers para analizar entrada de secretos |
    | `plugin-sdk/webhook-ingress` | Helpers de solicitud/destino para webhooks |
    | `plugin-sdk/webhook-request-guards` | Helpers de tamaño de cuerpo/timeout para solicitudes |
  </Accordion>

  <Accordion title="Subrutas de tiempo de ejecución y almacenamiento">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers amplios de tiempo de ejecución/logging/copia de seguridad/instalación de plugins |
    | `plugin-sdk/runtime-env` | Helpers limitados de entorno de tiempo de ejecución, logger, timeout, reintento y backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers compartidos de comandos/hooks/http/interactivo para plugins |
    | `plugin-sdk/hook-runtime` | Helpers compartidos para canalizaciones de hooks internos/webhook |
    | `plugin-sdk/lazy-runtime` | Helpers de importación/vinculación diferida en tiempo de ejecución como `createLazyRuntimeModule`, `createLazyRuntimeMethod` y `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers de ejecución de procesos |
    | `plugin-sdk/cli-runtime` | Helpers de formato CLI, espera y versión |
    | `plugin-sdk/gateway-runtime` | Helpers de cliente Gateway y parches de estado de canal |
    | `plugin-sdk/config-runtime` | Helpers de carga/escritura de configuración |
    | `plugin-sdk/telegram-command-config` | Normalización de nombres/descripciones de comandos de Telegram y comprobaciones de duplicados/conflictos, incluso cuando la superficie contractual incluida de Telegram no está disponible |
    | `plugin-sdk/approval-runtime` | Helpers de aprobaciones de exec/plugin, constructores de capacidades de aprobación, helpers de autenticación/perfiles, helpers nativos de enrutamiento/tiempo de ejecución |
    | `plugin-sdk/reply-runtime` | Helpers compartidos de tiempo de ejecución para entrada/respuesta, fragmentación, despacho, heartbeat, planificador de respuestas |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers limitados para despacho/finalización de respuestas |
    | `plugin-sdk/reply-history` | Helpers compartidos de historial de respuestas de ventana corta como `buildHistoryContext`, `recordPendingHistoryEntry` y `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers limitados de fragmentación de texto/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de ruta y marca temporal updated-at del almacén de sesiones |
    | `plugin-sdk/state-paths` | Helpers de rutas de directorios de estado/OAuth |
    | `plugin-sdk/routing` | Helpers de ruta/clave de sesión/vinculación de cuentas como `resolveAgentRoute`, `buildAgentSessionKey` y `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers compartidos de resumen de estado de canal/cuenta, valores predeterminados de estado de tiempo de ejecución y helpers de metadatos de incidencias |
    | `plugin-sdk/target-resolver-runtime` | Helpers compartidos de resolución de destinos |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalización de slugs/cadenas |
    | `plugin-sdk/request-url` | Extrae URLs de tipo string de entradas tipo fetch/request |
    | `plugin-sdk/run-command` | Ejecutor de comandos con temporización y resultados stdout/stderr normalizados |
    | `plugin-sdk/param-readers` | Lectores comunes de parámetros de herramienta/CLI |
    | `plugin-sdk/tool-send` | Extrae campos canónicos de destino de envío de argumentos de herramienta |
    | `plugin-sdk/temp-path` | Helpers compartidos de rutas temporales de descarga |
    | `plugin-sdk/logging-core` | Helpers de logger por subsistema y redacción |
    | `plugin-sdk/markdown-table-runtime` | Helpers para modo de tablas Markdown |
    | `plugin-sdk/json-store` | Helpers pequeños de lectura/escritura de estado JSON |
    | `plugin-sdk/file-lock` | Helpers de bloqueo de archivos reentrantes |
    | `plugin-sdk/persistent-dedupe` | Helpers de caché de deduplicación respaldada en disco |
    | `plugin-sdk/acp-runtime` | Helpers de ACP en tiempo de ejecución/sesión |
    | `plugin-sdk/agent-config-primitives` | Primitivas limitadas del esquema de configuración del agente en tiempo de ejecución |
    | `plugin-sdk/boolean-param` | Lector permisivo de parámetros booleanos |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de resolución de coincidencias de nombres peligrosos |
    | `plugin-sdk/device-bootstrap` | Helpers de bootstrap de dispositivo y tokens de emparejamiento |
    | `plugin-sdk/extension-shared` | Primitivas compartidas de ayuda de canales pasivos y estado |
    | `plugin-sdk/models-provider-runtime` | Helpers de respuestas para comando/proveedor `/models` |
    | `plugin-sdk/skill-commands-runtime` | Helpers de listado de comandos de Skills |
    | `plugin-sdk/native-command-registry` | Helpers de registro/construcción/serialización de comandos nativos |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de detección de endpoints Z.AI |
    | `plugin-sdk/infra-runtime` | Helpers de eventos de sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Helpers pequeños de caché acotada |
    | `plugin-sdk/diagnostic-runtime` | Helpers de indicadores y eventos de diagnóstico |
    | `plugin-sdk/error-runtime` | Helpers de grafo de errores, formato y clasificación compartida de errores, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch envuelto, proxy y búsqueda fijada |
    | `plugin-sdk/host-runtime` | Helpers de normalización de nombres de host y host SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuración de reintentos y ejecutor de reintentos |
    | `plugin-sdk/agent-runtime` | Helpers de directorio/identidad/espacio de trabajo del agente |
    | `plugin-sdk/directory-runtime` | Consulta/deduplicación de directorio respaldado por configuración |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subrutas de capacidades y pruebas">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers compartidos de fetch/transformación/almacenamiento de medios además de constructores de cargas útiles de medios |
    | `plugin-sdk/media-understanding` | Tipos de proveedores de comprensión de medios además de exportaciones helper para imágenes/audio orientadas al proveedor |
    | `plugin-sdk/text-runtime` | Helpers compartidos de texto/Markdown/logging como eliminación de texto visible para el asistente, renderizado/fragmentación/tablas Markdown, helpers de redacción, helpers de etiquetas directivas y utilidades de texto seguro |
    | `plugin-sdk/text-chunking` | Helper de fragmentación de texto saliente |
    | `plugin-sdk/speech` | Tipos de proveedores de voz además de exportaciones helper de directivas, registro y validación orientadas al proveedor |
    | `plugin-sdk/speech-core` | Tipos compartidos de proveedores de voz, helpers de registro, directivas y normalización |
    | `plugin-sdk/realtime-transcription` | Tipos de proveedores de transcripción en tiempo real y helpers de registro |
    | `plugin-sdk/realtime-voice` | Tipos de proveedores de voz en tiempo real y helpers de registro |
    | `plugin-sdk/image-generation` | Tipos de proveedores de generación de imágenes |
    | `plugin-sdk/image-generation-core` | Tipos compartidos de generación de imágenes, helpers de failover, autenticación y registro |
    | `plugin-sdk/video-generation` | Tipos de proveedor/solicitud/resultado de generación de video |
    | `plugin-sdk/video-generation-core` | Tipos compartidos de generación de video, helpers de failover, búsqueda de proveedores y análisis de model-ref |
    | `plugin-sdk/webhook-targets` | Registro de destinos webhook y helpers de instalación de rutas |
    | `plugin-sdk/webhook-path` | Helpers de normalización de rutas webhook |
    | `plugin-sdk/web-media` | Helpers compartidos de carga de medios remotos/locales |
    | `plugin-sdk/zod` | `zod` reexportado para consumidores del SDK de plugins |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Subrutas de memoria">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie helper incluida de memory-core para helpers de manager/configuración/archivos/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fachada de tiempo de ejecución de indexación/búsqueda de memoria |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportaciones del motor base de memoria del host |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exportaciones del motor de embeddings de memoria del host |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportaciones del motor QMD de memoria del host |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportaciones del motor de almacenamiento de memoria del host |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodales de memoria del host |
    | `plugin-sdk/memory-core-host-query` | Helpers de consultas de memoria del host |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secretos de memoria del host |
    | `plugin-sdk/memory-core-host-status` | Helpers de estado de memoria del host |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers de tiempo de ejecución CLI para memoria del host |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers centrales de tiempo de ejecución para memoria del host |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de archivos/tiempo de ejecución para memoria del host |
    | `plugin-sdk/memory-lancedb` | Superficie helper incluida de memory-lancedb |
  </Accordion>

  <Accordion title="Subrutas helper incluidas reservadas">
    | Familia | Subrutas actuales | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-config-support`, `plugin-sdk/browser-support` | Helpers de soporte del plugin de navegador incluido |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie helper/tiempo de ejecución de Matrix incluido |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie helper/tiempo de ejecución de LINE incluido |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie helper de IRC incluido |
    | Helpers específicos de canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Superficies de compatibilidad/helper de canales incluidos |
    | Helpers específicos de autenticación/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Superficies helper de funciones/plugins incluidos; `plugin-sdk/github-copilot-token` exporta actualmente `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API de registro

La devolución de llamada `register(api)` recibe un objeto `OpenClawPluginApi` con estos
métodos:

### Registro de capacidades

| Método                                           | Qué registra                     |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Inferencia de texto (LLM)        |
| `api.registerCliBackend(...)`                    | Backend local de inferencia CLI  |
| `api.registerChannel(...)`                       | Canal de mensajería              |
| `api.registerSpeechProvider(...)`                | Texto a voz / síntesis STT       |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcripción en tiempo real por streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sesiones dúplex de voz en tiempo real |
| `api.registerMediaUnderstandingProvider(...)`    | Análisis de imágenes/audio/video |
| `api.registerImageGenerationProvider(...)`       | Generación de imágenes           |
| `api.registerVideoGenerationProvider(...)`       | Generación de video              |
| `api.registerWebFetchProvider(...)`              | Proveedor de web fetch / scrape  |
| `api.registerWebSearchProvider(...)`             | Búsqueda web                     |

### Herramientas y comandos

| Método                          | Qué registra                                   |
| ------------------------------- | ---------------------------------------------- |
| `api.registerTool(tool, opts?)` | Herramienta del agente (requerida o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizado (omite el LLM)           |

### Infraestructura

| Método                                         | Qué registra             |
| ---------------------------------------------- | ------------------------ |
| `api.registerHook(events, handler, opts?)`     | Hook de evento           |
| `api.registerHttpRoute(params)`                | Endpoint HTTP del Gateway |
| `api.registerGatewayMethod(name, handler)`     | Método RPC del Gateway   |
| `api.registerCli(registrar, opts?)`            | Subcomando CLI           |
| `api.registerService(service)`                 | Servicio en segundo plano |
| `api.registerInteractiveHandler(registration)` | Controlador interactivo  |

Los espacios de nombres de administración del núcleo reservados (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre permanecen como `operator.admin`, aunque un plugin intente asignar un
alcance más estrecho a un método del gateway. Prefiere prefijos específicos de plugin para
métodos propiedad del plugin.

### Metadatos de registro de CLI

`api.registerCli(registrar, opts?)` acepta dos tipos de metadatos de nivel superior:

- `commands`: raíces explícitas de comandos que pertenecen al registrador
- `descriptors`: descriptores de comandos en tiempo de análisis usados para la ayuda del CLI raíz,
  el enrutamiento y el registro diferido del CLI del plugin

Si quieres que un comando de plugin permanezca cargado de forma diferida en la ruta normal del CLI raíz,
proporciona `descriptors` que cubran cada raíz de comando de nivel superior expuesta por ese
registrador.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Manage Matrix accounts, verification, devices, and profile state",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` por sí solo solo cuando no necesites registro diferido del CLI raíz.
Esa ruta de compatibilidad con carga ansiosa sigue estando permitida, pero no instala
marcadores de posición respaldados por descriptores para carga diferida en tiempo de análisis.

### Registro de backend CLI

`api.registerCliBackend(...)` permite que un plugin controle la configuración predeterminada de un
backend CLI de IA local como `claude-cli` o `codex-cli`.

- El `id` del backend se convierte en el prefijo del proveedor en referencias de modelo como `claude-cli/opus`.
- La `config` del backend usa la misma forma que `agents.defaults.cliBackends.<id>`.
- La configuración del usuario sigue prevaleciendo. OpenClaw fusiona `agents.defaults.cliBackends.<id>` sobre el
  valor predeterminado del plugin antes de ejecutar la CLI.
- Usa `normalizeConfig` cuando un backend necesite reescrituras de compatibilidad después de la fusión
  (por ejemplo normalizar formas antiguas de indicadores).

### Slots exclusivos

| Método                                     | Qué registra                           |
| ------------------------------------------ | -------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motor de contexto (uno activo a la vez) |
| `api.registerMemoryPromptSection(builder)` | Constructor de sección de prompt de memoria |
| `api.registerMemoryFlushPlan(resolver)`    | Resolutor de plan de vaciado de memoria |
| `api.registerMemoryRuntime(runtime)`       | Adaptador de tiempo de ejecución de memoria |

### Adaptadores de embeddings de memoria

| Método                                         | Qué registra                                  |
| ---------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de embeddings de memoria para el plugin activo |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` y
  `registerMemoryRuntime` son exclusivos de los plugins de memoria.
- `registerMemoryEmbeddingProvider` permite que el plugin de memoria activo registre uno
  o más IDs de adaptadores de embeddings (por ejemplo `openai`, `gemini` o un ID definido por un plugin personalizado).
- La configuración del usuario como `agents.defaults.memorySearch.provider` y
  `agents.defaults.memorySearch.fallback` se resuelve contra esos IDs de
  adaptadores registrados.

### Eventos y ciclo de vida

| Método                                       | Qué hace                    |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook tipado de ciclo de vida |
| `api.onConversationBindingResolved(handler)` | Callback de vinculación de conversación |

### Semántica de decisión de hooks

- `before_tool_call`: devolver `{ block: true }` es terminal. En cuanto cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `before_tool_call`: devolver `{ block: false }` se trata como ausencia de decisión (igual que omitir `block`), no como una anulación.
- `before_install`: devolver `{ block: true }` es terminal. En cuanto cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `before_install`: devolver `{ block: false }` se trata como ausencia de decisión (igual que omitir `block`), no como una anulación.
- `message_sending`: devolver `{ cancel: true }` es terminal. En cuanto cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `message_sending`: devolver `{ cancel: false }` se trata como ausencia de decisión (igual que omitir `cancel`), no como una anulación.

### Campos del objeto API

| Campo                    | Tipo                      | Descripción                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID del plugin                                                                               |
| `api.name`               | `string`                  | Nombre para mostrar                                                                         |
| `api.version`            | `string?`                 | Versión del plugin (opcional)                                                               |
| `api.description`        | `string?`                 | Descripción del plugin (opcional)                                                           |
| `api.source`             | `string`                  | Ruta de origen del plugin                                                                   |
| `api.rootDir`            | `string?`                 | Directorio raíz del plugin (opcional)                                                       |
| `api.config`             | `OpenClawConfig`          | Instantánea actual de configuración (instantánea activa en memoria del tiempo de ejecución cuando está disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin desde `plugins.entries.<id>.config`                     |
| `api.runtime`            | `PluginRuntime`           | [Runtime helpers](/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Logger acotado (`debug`, `info`, `warn`, `error`)                                           |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera previa al arranque/configuración completa |
| `api.resolvePath(input)` | `(string) => string`      | Resolver ruta relativa a la raíz del plugin                                                 |

## Convención de módulos internos

Dentro de tu plugin, usa archivos barrel locales para las importaciones internas:

```
my-plugin/
  api.ts            # Exportaciones públicas para consumidores externos
  runtime-api.ts    # Exportaciones internas solo de tiempo de ejecución
  index.ts          # Punto de entrada del plugin
  setup-entry.ts    # Entrada ligera solo para configuración (opcional)
```

<Warning>
  Nunca importes tu propio plugin mediante `openclaw/plugin-sdk/<your-plugin>`
  desde código de producción. Enruta las importaciones internas mediante `./api.ts` o
  `./runtime-api.ts`. La ruta del SDK es solo el contrato externo.
</Warning>

Las superficies públicas de plugins incluidos cargadas mediante fachada (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` y archivos de entrada públicos similares) ahora prefieren la
instantánea activa de configuración en tiempo de ejecución cuando OpenClaw ya se está ejecutando. Si todavía no existe ninguna instantánea de tiempo de ejecución, recurren al archivo de configuración resuelto en disco.

Los plugins de proveedor también pueden exponer un barrel contractual local y limitado del plugin cuando un
helper es intencionalmente específico del proveedor y todavía no pertenece a una subruta genérica del SDK. Ejemplo incluido actual: el proveedor Anthropic mantiene sus helpers de stream de Claude en su propia superficie pública `api.ts` / `contract-api.ts` en lugar de promover la lógica de encabezados beta de Anthropic y `service_tier` a un contrato genérico `plugin-sdk/*`.

Otros ejemplos incluidos actuales:

- `@openclaw/openai-provider`: `api.ts` exporta constructores de proveedor,
  helpers de modelos predeterminados y constructores de proveedores en tiempo real
- `@openclaw/openrouter-provider`: `api.ts` exporta el constructor del proveedor además de
  helpers de onboarding/configuración

<Warning>
  El código de producción de extensiones también debe evitar las importaciones
  `openclaw/plugin-sdk/<other-plugin>`. Si un helper es realmente compartido, promuévelo a una subruta neutral del SDK
  como `openclaw/plugin-sdk/speech`, `.../provider-model-shared` u otra
  superficie orientada a capacidades en lugar de acoplar dos plugins entre sí.
</Warning>

## Relacionado

- [Entry Points](/plugins/sdk-entrypoints) — opciones de `definePluginEntry` y `defineChannelPluginEntry`
- [Runtime Helpers](/plugins/sdk-runtime) — referencia completa del espacio de nombres `api.runtime`
- [Setup and Config](/plugins/sdk-setup) — empaquetado, manifiestos, esquemas de configuración
- [Testing](/plugins/sdk-testing) — utilidades de pruebas y reglas de lint
- [SDK Migration](/plugins/sdk-migration) — migración desde superficies obsoletas
- [Plugin Internals](/plugins/architecture) — arquitectura profunda y modelo de capacidades
