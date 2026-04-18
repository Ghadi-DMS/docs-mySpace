---
read_when:
    - Necesitas saber desde qué subruta del SDK importar
    - Quieres una referencia de todos los métodos de registro en OpenClawPluginApi
    - Estás buscando una exportación específica del SDK
sidebarTitle: SDK Overview
summary: Mapa de importación, referencia de la API de registro y arquitectura del SDK
title: Resumen del SDK de Plugin
x-i18n:
    generated_at: "2026-04-18T04:59:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 05d3d0022cca32d29c76f6cea01cdf4f88ac69ef0ef3d7fb8a60fbf9a6b9b331
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Resumen del SDK de Plugin

El SDK de Plugin es el contrato tipado entre los plugins y el núcleo. Esta página es la
referencia para **qué importar** y **qué puedes registrar**.

<Tip>
  **¿Buscas una guía práctica?**
  - ¿Tu primer plugin? Empieza con [Primeros pasos](/es/plugins/building-plugins)
  - ¿Plugin de canal? Consulta [Plugins de canal](/es/plugins/sdk-channel-plugins)
  - ¿Plugin de proveedor? Consulta [Plugins de proveedor](/es/plugins/sdk-provider-plugins)
</Tip>

## Convención de importación

Importa siempre desde una subruta específica:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subruta es un módulo pequeño y autocontenido. Esto mantiene un inicio rápido y
evita problemas de dependencia circular. Para los helpers de entrada/construcción específicos de canal,
prefiere `openclaw/plugin-sdk/channel-core`; reserva `openclaw/plugin-sdk/core` para
la superficie paraguas más amplia y los helpers compartidos, como
`buildChannelConfigSchema`.

No añadas ni dependas de superficies de conveniencia con nombre de proveedor como
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
superficies helper con marca de canal. Los plugins empaquetados deben componer
subrutas genéricas del SDK dentro de sus propios barrels `api.ts` o `runtime-api.ts`, y el núcleo
debe usar esos barrels locales del plugin o añadir un contrato SDK genérico y estrecho
cuando la necesidad sea realmente transversal a varios canales.

El mapa de exportación generado todavía contiene un pequeño conjunto de
superficies helper de plugins empaquetados como `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Esas
subrutas existen solo para mantenimiento y compatibilidad de plugins empaquetados; se
omiten intencionalmente de la tabla común de abajo y no son la ruta de importación
recomendada para nuevos plugins de terceros.

## Referencia de subrutas

Las subrutas más usadas habitualmente, agrupadas por propósito. La lista completa generada de
más de 200 subrutas vive en `scripts/lib/plugin-sdk-entrypoints.json`.

Las subrutas helper reservadas para plugins empaquetados siguen apareciendo en esa lista generada.
Trátalas como superficies de detalle de implementación/compatibilidad a menos que una página de documentación
promueva explícitamente una como pública.

### Entrada de Plugin

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
    | `plugin-sdk/setup` | Helpers compartidos para asistentes de configuración, prompts de allowlist y constructores de estado de configuración |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de compuertas de acción/configuración para múltiples cuentas, y helpers de fallback de cuenta predeterminada |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalización de id de cuenta |
    | `plugin-sdk/account-resolution` | Helpers de búsqueda de cuenta + fallback predeterminado |
    | `plugin-sdk/account-helpers` | Helpers específicos para lista de cuentas/acciones de cuenta |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipos de esquema de configuración de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalización/validación de comandos personalizados de Telegram con fallback de contrato empaquetado |
    | `plugin-sdk/command-gating` | Helpers específicos para compuertas de autorización de comandos |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers compartidos para rutas de entrada + construcción de sobres |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers compartidos para registrar y despachar entradas |
    | `plugin-sdk/messaging-targets` | Helpers de análisis/coincidencia de destinos |
    | `plugin-sdk/outbound-media` | Helpers compartidos para carga de medios salientes |
    | `plugin-sdk/outbound-runtime` | Helpers de delegación de envío/identidad saliente |
    | `plugin-sdk/poll-runtime` | Helpers específicos de normalización de encuestas |
    | `plugin-sdk/thread-bindings-runtime` | Helpers para el ciclo de vida de vinculaciones de hilos y adaptadores |
    | `plugin-sdk/agent-media-payload` | Constructor heredado de payload de medios del agente |
    | `plugin-sdk/conversation-runtime` | Helpers para vinculaciones de conversación/hilo, pairing y binding configurado |
    | `plugin-sdk/runtime-config-snapshot` | Helper de instantánea de configuración de runtime |
    | `plugin-sdk/runtime-group-policy` | Helpers de resolución de políticas de grupo en runtime |
    | `plugin-sdk/channel-status` | Helpers compartidos para instantáneas/resúmenes de estado del canal |
    | `plugin-sdk/channel-config-primitives` | Primitivas específicas de esquema de configuración de canal |
    | `plugin-sdk/channel-config-writes` | Helpers de autorización para escritura de configuración de canal |
    | `plugin-sdk/channel-plugin-common` | Exportaciones de preludio compartidas para plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lectura/edición de configuración de allowlist |
    | `plugin-sdk/group-access` | Helpers compartidos para decisiones de acceso a grupos |
    | `plugin-sdk/direct-dm` | Helpers compartidos de autenticación/protección para DM directos |
    | `plugin-sdk/interactive-runtime` | Helpers de normalización/reducción de payloads de respuesta interactiva |
    | `plugin-sdk/channel-inbound` | Barrel de compatibilidad para debounce de entrada, coincidencia de menciones, helpers de política de menciones y helpers de sobres |
    | `plugin-sdk/channel-mention-gating` | Helpers específicos de política de menciones sin la superficie más amplia de runtime de entrada |
    | `plugin-sdk/channel-location` | Helpers de contexto y formato de ubicación del canal |
    | `plugin-sdk/channel-logging` | Helpers de registro del canal para descartes de entrada y fallos de typing/ack |
    | `plugin-sdk/channel-send-result` | Tipos de resultado de respuesta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers de análisis/coincidencia de destinos |
    | `plugin-sdk/channel-contract` | Tipos de contrato de canal |
    | `plugin-sdk/channel-feedback` | Cableado de feedback/reacciones |
    | `plugin-sdk/channel-secret-runtime` | Helpers específicos de contrato de secretos, como `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, y tipos de destino de secretos |
  </Accordion>

  <Accordion title="Subrutas de proveedor">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers de configuración seleccionados para proveedores locales/autohospedados |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers específicos de configuración para proveedores autohospedados compatibles con OpenAI |
    | `plugin-sdk/cli-backend` | Valores predeterminados del backend de CLI + constantes watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers de runtime para resolución de claves API en plugins de proveedor |
    | `plugin-sdk/provider-auth-api-key` | Helpers de incorporación/escritura de perfiles de claves API como `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructor estándar de resultados de autenticación OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers compartidos de inicio de sesión interactivo para plugins de proveedor |
    | `plugin-sdk/provider-env-vars` | Helpers de búsqueda de variables de entorno de autenticación del proveedor |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de políticas de repetición, helpers de endpoint del proveedor y helpers de normalización de id de modelo como `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers genéricos de HTTP/capacidades de endpoint del proveedor |
    | `plugin-sdk/provider-web-fetch-contract` | Helpers específicos de contrato para configuración/selección de web-fetch, como `enablePluginInConfig` y `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpers de registro/caché para proveedores web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Helpers específicos de configuración/credenciales de búsqueda web para proveedores que no necesitan cableado de habilitación del plugin |
    | `plugin-sdk/provider-web-search-contract` | Helpers específicos de contrato para configuración/credenciales de búsqueda web como `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` y setters/getters de credenciales con alcance |
    | `plugin-sdk/provider-web-search` | Helpers de registro/caché/runtime para proveedores de búsqueda web |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquema Gemini, y helpers de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` y similares |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de wrapper de stream y helpers compartidos de wrapper para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de parcheo de configuración de incorporación |
    | `plugin-sdk/global-singleton` | Helpers de singleton/mapa/caché locales al proceso |
  </Accordion>

  <Accordion title="Subrutas de autenticación y seguridad">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers del registro de comandos, helpers de autorización de remitente |
    | `plugin-sdk/command-status` | Constructores de mensajes de comando/ayuda como `buildCommandsMessagePaginated` y `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Helpers de resolución de aprobadores y de autenticación de acciones en el mismo chat |
    | `plugin-sdk/approval-client-runtime` | Helpers de perfil/filtro de aprobación para exec nativo |
    | `plugin-sdk/approval-delivery-runtime` | Adaptadores de capacidad/entrega de aprobación nativa |
    | `plugin-sdk/approval-gateway-runtime` | Helper compartido de resolución de Gateway de aprobación |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helpers ligeros de carga de adaptadores de aprobación nativa para entrypoints de canal de alta frecuencia |
    | `plugin-sdk/approval-handler-runtime` | Helpers más amplios de runtime para manejadores de aprobación; prefiere las superficies más específicas de adaptador/Gateway cuando sean suficientes |
    | `plugin-sdk/approval-native-runtime` | Helpers nativos de destino de aprobación + binding de cuenta |
    | `plugin-sdk/approval-reply-runtime` | Helpers de payload de respuesta para aprobación de exec/plugin |
    | `plugin-sdk/command-auth-native` | Helpers de autenticación de comandos nativos + de destino de sesión nativa |
    | `plugin-sdk/command-detection` | Helpers compartidos de detección de comandos |
    | `plugin-sdk/command-surface` | Helpers de normalización del cuerpo del comando y de la superficie de comandos |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers específicos de recopilación de contratos de secretos para superficies de secretos de canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers específicos de tipado de `coerceSecretRef` y SecretRef para análisis de contrato/configuración de secretos |
    | `plugin-sdk/security-runtime` | Helpers compartidos de confianza, compuerta de DM, contenido externo y recopilación de secretos |
    | `plugin-sdk/ssrf-policy` | Helpers de allowlist de hosts y de política SSRF de red privada |
    | `plugin-sdk/ssrf-dispatcher` | Helpers específicos de dispatcher fijado sin la amplia superficie de runtime de infraestructura |
    | `plugin-sdk/ssrf-runtime` | Helpers de dispatcher fijado, fetch protegido contra SSRF y política SSRF |
    | `plugin-sdk/secret-input` | Helpers de análisis de entrada secreta |
    | `plugin-sdk/webhook-ingress` | Helpers de solicitud/destino de Webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de tamaño de cuerpo/timeout de solicitud |
  </Accordion>

  <Accordion title="Subrutas de runtime y almacenamiento">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers amplios de runtime/logging/copias de seguridad/instalación de plugins |
    | `plugin-sdk/runtime-env` | Helpers específicos de entorno de runtime, logger, timeout, retry y backoff |
    | `plugin-sdk/channel-runtime-context` | Helpers genéricos de registro y búsqueda de contexto de runtime de canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers compartidos de plugin para comandos/hooks/http/interactivos |
    | `plugin-sdk/hook-runtime` | Helpers compartidos del pipeline de hooks internos/Webhook |
    | `plugin-sdk/lazy-runtime` | Helpers de importación/binding diferido de runtime como `createLazyRuntimeModule`, `createLazyRuntimeMethod` y `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers de ejecución de procesos |
    | `plugin-sdk/cli-runtime` | Helpers de formato, espera y versión de CLI |
    | `plugin-sdk/gateway-runtime` | Helpers de cliente de Gateway y de parche de estado del canal |
    | `plugin-sdk/config-runtime` | Helpers de carga/escritura de configuración |
    | `plugin-sdk/telegram-command-config` | Helpers de normalización de nombre/descripción de comandos de Telegram y comprobaciones de duplicados/conflictos, incluso cuando la superficie de contrato empaquetada de Telegram no está disponible |
    | `plugin-sdk/text-autolink-runtime` | Detección de autolinks de referencias de archivo sin el amplio barrel `text-runtime` |
    | `plugin-sdk/approval-runtime` | Helpers de aprobación de exec/plugin, constructores de capacidad de aprobación, helpers de auth/perfil, helpers de routing/runtime nativo |
    | `plugin-sdk/reply-runtime` | Helpers compartidos de runtime de entrada/respuesta, chunking, despacho, Heartbeat, planificador de respuestas |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers específicos de despacho/finalización de respuestas |
    | `plugin-sdk/reply-history` | Helpers compartidos de historial de respuestas de ventana corta como `buildHistoryContext`, `recordPendingHistoryEntry` y `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers específicos de fragmentación de texto/markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de ruta de almacenamiento de sesión + updated-at |
    | `plugin-sdk/state-paths` | Helpers de rutas de directorio de estado/OAuth |
    | `plugin-sdk/routing` | Helpers de ruta/clave de sesión/binding de cuenta como `resolveAgentRoute`, `buildAgentSessionKey` y `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers compartidos de resumen de estado de canal/cuenta, valores predeterminados de estado de runtime y helpers de metadatos de incidencias |
    | `plugin-sdk/target-resolver-runtime` | Helpers compartidos de resolución de destino |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalización de slug/cadenas |
    | `plugin-sdk/request-url` | Extraer URL de cadena de entradas tipo fetch/request |
    | `plugin-sdk/run-command` | Ejecutor de comandos temporizado con resultados normalizados de stdout/stderr |
    | `plugin-sdk/param-readers` | Lectores comunes de parámetros de herramienta/CLI |
    | `plugin-sdk/tool-payload` | Extraer payloads normalizados de objetos de resultado de herramientas |
    | `plugin-sdk/tool-send` | Extraer campos de destino de envío canónicos de argumentos de herramienta |
    | `plugin-sdk/temp-path` | Helpers compartidos de rutas temporales de descarga |
    | `plugin-sdk/logging-core` | Helpers de logger de subsistema y redacción |
    | `plugin-sdk/markdown-table-runtime` | Helpers de modo de tabla Markdown |
    | `plugin-sdk/json-store` | Pequeños helpers de lectura/escritura de estado JSON |
    | `plugin-sdk/file-lock` | Helpers de bloqueo de archivos reentrante |
    | `plugin-sdk/persistent-dedupe` | Helpers de caché de deduplicación respaldada por disco |
    | `plugin-sdk/acp-runtime` | Helpers de runtime/sesión ACP y despacho de respuestas |
    | `plugin-sdk/acp-binding-resolve-runtime` | Resolución de binding ACP de solo lectura sin importaciones de inicio del ciclo de vida |
    | `plugin-sdk/agent-config-primitives` | Primitivas específicas de esquema de configuración de runtime del agente |
    | `plugin-sdk/boolean-param` | Lector flexible de parámetros booleanos |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de resolución de coincidencia de nombres peligrosos |
    | `plugin-sdk/device-bootstrap` | Helpers de bootstrap del dispositivo y token de pairing |
    | `plugin-sdk/extension-shared` | Primitivas helper compartidas para canal pasivo, estado y proxy ambiental |
    | `plugin-sdk/models-provider-runtime` | Helpers de respuesta de proveedor/comando `/models` |
    | `plugin-sdk/skill-commands-runtime` | Helpers de listado de comandos de Skills |
    | `plugin-sdk/native-command-registry` | Helpers de registro/construcción/serialización de comandos nativos |
    | `plugin-sdk/agent-harness` | Superficie experimental de plugin confiable para harnesses de agente de bajo nivel: tipos de harness, helpers de steering/abort de ejecución activa, helpers de puente de herramientas de OpenClaw y utilidades de resultados de intentos |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de detección de endpoint de Z.A.I |
    | `plugin-sdk/infra-runtime` | Helpers de eventos del sistema/Heartbeat |
    | `plugin-sdk/collection-runtime` | Pequeños helpers de caché acotada |
    | `plugin-sdk/diagnostic-runtime` | Helpers de flags y eventos de diagnóstico |
    | `plugin-sdk/error-runtime` | Grafo de errores, formato, helpers compartidos de clasificación de errores, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch envuelto, proxy y búsqueda fijada |
    | `plugin-sdk/runtime-fetch` | Fetch de runtime con reconocimiento de dispatcher sin importaciones de proxy/fetch protegido |
    | `plugin-sdk/response-limit-runtime` | Lector acotado del cuerpo de la respuesta sin la amplia superficie de runtime de medios |
    | `plugin-sdk/session-binding-runtime` | Estado actual del binding de conversación sin routing de binding configurado ni almacenes de pairing |
    | `plugin-sdk/session-store-runtime` | Helpers de lectura del almacenamiento de sesión sin importaciones amplias de escritura/mantenimiento de configuración |
    | `plugin-sdk/context-visibility-runtime` | Resolución de visibilidad de contexto y filtrado de contexto suplementario sin importaciones amplias de configuración/seguridad |
    | `plugin-sdk/string-coerce-runtime` | Helpers específicos de coerción y normalización de cadenas/registros primitivos sin importaciones de markdown/logging |
    | `plugin-sdk/host-runtime` | Helpers de hostname y normalización de host SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuración de retry y ejecutor de retry |
    | `plugin-sdk/agent-runtime` | Helpers de directorio/identidad/espacio de trabajo del agente |
    | `plugin-sdk/directory-runtime` | Consulta/deduplicación de directorio respaldada por configuración |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subrutas de capacidades y pruebas">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers compartidos de fetch/transformación/almacenamiento de medios, además de constructores de payload de medios |
    | `plugin-sdk/media-generation-runtime` | Helpers compartidos de failover de generación de medios, selección de candidatos y mensajería de modelo faltante |
    | `plugin-sdk/media-understanding` | Tipos de proveedor de comprensión de medios, además de exportaciones helper orientadas al proveedor para imagen/audio |
    | `plugin-sdk/text-runtime` | Helpers compartidos de texto/markdown/logging como eliminación de texto visible para el asistente, helpers de render/chunking/tabla Markdown, helpers de redacción, helpers de etiquetas de directivas y utilidades de texto seguro |
    | `plugin-sdk/text-chunking` | Helper de fragmentación de texto saliente |
    | `plugin-sdk/speech` | Tipos de proveedor de speech, además de helpers orientados al proveedor para directivas, registro y validación |
    | `plugin-sdk/speech-core` | Tipos compartidos de proveedor de speech, helpers de registro, directivas y normalización |
    | `plugin-sdk/realtime-transcription` | Tipos de proveedor de transcripción en tiempo real y helpers de registro |
    | `plugin-sdk/realtime-voice` | Tipos de proveedor de voz en tiempo real y helpers de registro |
    | `plugin-sdk/image-generation` | Tipos de proveedor de generación de imágenes |
    | `plugin-sdk/image-generation-core` | Tipos compartidos de generación de imágenes, helpers de failover, auth y registro |
    | `plugin-sdk/music-generation` | Tipos de proveedor/solicitud/resultado de generación de música |
    | `plugin-sdk/music-generation-core` | Tipos compartidos de generación de música, helpers de failover, búsqueda de proveedores y análisis de model-ref |
    | `plugin-sdk/video-generation` | Tipos de proveedor/solicitud/resultado de generación de video |
    | `plugin-sdk/video-generation-core` | Tipos compartidos de generación de video, helpers de failover, búsqueda de proveedores y análisis de model-ref |
    | `plugin-sdk/webhook-targets` | Helpers de registro de destinos de Webhook e instalación de rutas |
    | `plugin-sdk/webhook-path` | Helpers de normalización de ruta de Webhook |
    | `plugin-sdk/web-media` | Helpers compartidos de carga de medios remotos/locales |
    | `plugin-sdk/zod` | `zod` reexportado para consumidores del SDK de Plugin |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Subrutas de memoria">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie helper empaquetada de memory-core para helpers de administrador/configuración/archivo/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fachada de runtime de índice/búsqueda de memoria |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportaciones del motor base del host de memoria |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Contratos de embeddings del host de memoria, acceso al registro, proveedor local y helpers genéricos por lotes/remotos |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportaciones del motor QMD del host de memoria |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportaciones del motor de almacenamiento del host de memoria |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodales del host de memoria |
    | `plugin-sdk/memory-core-host-query` | Helpers de consulta del host de memoria |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secretos del host de memoria |
    | `plugin-sdk/memory-core-host-events` | Helpers del diario de eventos del host de memoria |
    | `plugin-sdk/memory-core-host-status` | Helpers de estado del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers de runtime de CLI del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers de runtime del núcleo del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de archivos/runtime del host de memoria |
    | `plugin-sdk/memory-host-core` | Alias neutral respecto al proveedor para los helpers de runtime del núcleo del host de memoria |
    | `plugin-sdk/memory-host-events` | Alias neutral respecto al proveedor para los helpers del diario de eventos del host de memoria |
    | `plugin-sdk/memory-host-files` | Alias neutral respecto al proveedor para los helpers de archivos/runtime del host de memoria |
    | `plugin-sdk/memory-host-markdown` | Helpers compartidos de markdown gestionado para plugins adyacentes a memoria |
    | `plugin-sdk/memory-host-search` | Fachada de runtime de Active Memory para acceso al administrador de búsqueda |
    | `plugin-sdk/memory-host-status` | Alias neutral respecto al proveedor para los helpers de estado del host de memoria |
    | `plugin-sdk/memory-lancedb` | Superficie helper empaquetada de memory-lancedb |
  </Accordion>

  <Accordion title="Subrutas helper empaquetadas reservadas">
    | Familia | Subrutas actuales | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de soporte del Plugin Browser empaquetado (`browser-support` sigue siendo el barrel de compatibilidad) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie helper/runtime empaquetada de Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie helper/runtime empaquetada de LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie helper empaquetada de IRC |
    | Helpers específicos de canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Superficies de compatibilidad/helper de canales empaquetados |
    | Helpers específicos de autenticación/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Superficies helper de funciones/plugins empaquetados; `plugin-sdk/github-copilot-token` actualmente exporta `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API de registro

La devolución de llamada `register(api)` recibe un objeto `OpenClawPluginApi` con estos
métodos:

### Registro de capacidades

| Método                                           | Qué registra                               |
| ------------------------------------------------ | ------------------------------------------ |
| `api.registerProvider(...)`                      | Inferencia de texto (LLM)                  |
| `api.registerAgentHarness(...)`                  | Ejecutor experimental de agente de bajo nivel |
| `api.registerCliBackend(...)`                    | Backend local de inferencia de CLI         |
| `api.registerChannel(...)`                       | Canal de mensajería                        |
| `api.registerSpeechProvider(...)`                | Síntesis de texto a voz / STT              |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcripción en tiempo real en streaming  |
| `api.registerRealtimeVoiceProvider(...)`         | Sesiones de voz en tiempo real dúplex      |
| `api.registerMediaUnderstandingProvider(...)`    | Análisis de imagen/audio/video             |
| `api.registerImageGenerationProvider(...)`       | Generación de imágenes                     |
| `api.registerMusicGenerationProvider(...)`       | Generación de música                       |
| `api.registerVideoGenerationProvider(...)`       | Generación de video                        |
| `api.registerWebFetchProvider(...)`              | Proveedor de fetch / scraping web          |
| `api.registerWebSearchProvider(...)`             | Búsqueda web                               |

### Herramientas y comandos

| Método                          | Qué registra                                  |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Herramienta del agente (obligatoria o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizado (omite el LLM)          |

### Infraestructura

| Método                                         | Qué registra                          |
| ---------------------------------------------- | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook de evento                        |
| `api.registerHttpRoute(params)`                | Endpoint HTTP de Gateway              |
| `api.registerGatewayMethod(name, handler)`     | Método RPC de Gateway                 |
| `api.registerCli(registrar, opts?)`            | Subcomando de CLI                     |
| `api.registerService(service)`                 | Servicio en segundo plano             |
| `api.registerInteractiveHandler(registration)` | Manejador interactivo                 |
| `api.registerMemoryPromptSupplement(builder)`  | Sección de prompt aditiva adyacente a memoria |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus aditivo de búsqueda/lectura de memoria |

Los espacios de nombres de administración reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre permanecen como `operator.admin`, incluso si un plugin intenta asignar un
alcance más estrecho a un método de Gateway. Prefiere prefijos específicos del plugin para
métodos propiedad del plugin.

### Metadatos de registro de CLI

`api.registerCli(registrar, opts?)` acepta dos tipos de metadatos de nivel superior:

- `commands`: raíces de comandos explícitas propiedad del registrador
- `descriptors`: descriptores de comandos en tiempo de análisis usados para la ayuda de la CLI raíz,
  el routing y el registro diferido de CLI del plugin

Si quieres que un comando de plugin permanezca con carga diferida en la ruta normal de la CLI raíz,
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
        description: "Administra cuentas de Matrix, verificación, dispositivos y estado del perfil",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` por sí solo solo cuando no necesites registro diferido en la CLI raíz.
Esa ruta de compatibilidad ansiosa sigue siendo compatible, pero no instala
placeholders respaldados por descriptores para la carga diferida en tiempo de análisis.

### Registro de backend de CLI

`api.registerCliBackend(...)` permite que un plugin sea propietario de la configuración predeterminada de un
backend local de CLI de IA como `codex-cli`.

- El `id` del backend se convierte en el prefijo del proveedor en referencias de modelo como `codex-cli/gpt-5`.
- La `config` del backend usa la misma forma que `agents.defaults.cliBackends.<id>`.
- La configuración del usuario sigue teniendo prioridad. OpenClaw fusiona `agents.defaults.cliBackends.<id>` sobre la
  predeterminada del plugin antes de ejecutar la CLI.
- Usa `normalizeConfig` cuando un backend necesite reescrituras de compatibilidad después de la fusión
  (por ejemplo, para normalizar formas antiguas de flags).

### Slots exclusivos

| Método                                     | Qué registra                                                                                                                                         |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motor de contexto (solo uno activo a la vez). La devolución de llamada `assemble()` recibe `availableTools` y `citationsMode` para que el motor pueda adaptar adiciones al prompt. |
| `api.registerMemoryCapability(capability)` | Capacidad de memoria unificada                                                                                                                       |
| `api.registerMemoryPromptSection(builder)` | Constructor de sección de prompt de memoria                                                                                                          |
| `api.registerMemoryFlushPlan(resolver)`    | Resolutor de plan de vaciado de memoria                                                                                                              |
| `api.registerMemoryRuntime(runtime)`       | Adaptador de runtime de memoria                                                                                                                      |

### Adaptadores de embeddings de memoria

| Método                                         | Qué registra                                   |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de embeddings de memoria para el plugin activo |

- `registerMemoryCapability` es la API exclusiva preferida para plugins de memoria.
- `registerMemoryCapability` también puede exponer `publicArtifacts.listArtifacts(...)`
  para que los plugins complementarios puedan consumir artefactos de memoria exportados mediante
  `openclaw/plugin-sdk/memory-host-core` en lugar de acceder al layout privado de un
  plugin de memoria específico.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` y
  `registerMemoryRuntime` son API exclusivas compatibles con versiones heredadas para plugins de memoria.
- `registerMemoryEmbeddingProvider` permite que el plugin de memoria activo registre uno
  o más ids de adaptador de embeddings (por ejemplo `openai`, `gemini` o un id personalizado
  definido por el plugin).
- La configuración del usuario, como `agents.defaults.memorySearch.provider` y
  `agents.defaults.memorySearch.fallback`, se resuelve respecto de esos ids de adaptador
  registrados.

### Eventos y ciclo de vida

| Método                                       | Qué hace                    |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook tipado de ciclo de vida |
| `api.onConversationBindingResolved(handler)` | Devolución de llamada de binding de conversación |

### Semántica de decisión de hooks

- `before_tool_call`: devolver `{ block: true }` es terminal. En cuanto cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `before_tool_call`: devolver `{ block: false }` se trata como si no hubiera decisión (igual que omitir `block`), no como una anulación.
- `before_install`: devolver `{ block: true }` es terminal. En cuanto cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `before_install`: devolver `{ block: false }` se trata como si no hubiera decisión (igual que omitir `block`), no como una anulación.
- `reply_dispatch`: devolver `{ handled: true, ... }` es terminal. En cuanto cualquier manejador reclama el despacho, se omiten los manejadores de menor prioridad y la ruta predeterminada de despacho del modelo.
- `message_sending`: devolver `{ cancel: true }` es terminal. En cuanto cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `message_sending`: devolver `{ cancel: false }` se trata como si no hubiera decisión (igual que omitir `cancel`), no como una anulación.

### Campos del objeto API

| Campo                    | Tipo                      | Descripción                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id del Plugin                                                                               |
| `api.name`               | `string`                  | Nombre para mostrar                                                                         |
| `api.version`            | `string?`                 | Versión del Plugin (opcional)                                                               |
| `api.description`        | `string?`                 | Descripción del Plugin (opcional)                                                           |
| `api.source`             | `string`                  | Ruta de origen del Plugin                                                                   |
| `api.rootDir`            | `string?`                 | Directorio raíz del Plugin (opcional)                                                       |
| `api.config`             | `OpenClawConfig`          | Instantánea actual de la configuración (instantánea activa del runtime en memoria cuando esté disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin desde `plugins.entries.<id>.config`                     |
| `api.runtime`            | `PluginRuntime`           | [Helpers de runtime](/es/plugins/sdk-runtime)                                                  |
| `api.logger`             | `PluginLogger`            | Logger con alcance (`debug`, `info`, `warn`, `error`)                                       |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de inicio/configuración previa a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resuelve una ruta relativa a la raíz del plugin                                             |

## Convención de módulos internos

Dentro de tu plugin, usa archivos barrel locales para las importaciones internas:

```
my-plugin/
  api.ts            # Exportaciones públicas para consumidores externos
  runtime-api.ts    # Exportaciones internas solo para runtime
  index.ts          # Punto de entrada del plugin
  setup-entry.ts    # Entrada ligera solo para configuración (opcional)
```

<Warning>
  Nunca importes tu propio plugin mediante `openclaw/plugin-sdk/<your-plugin>`
  desde código de producción. Enruta las importaciones internas por `./api.ts` o
  `./runtime-api.ts`. La ruta del SDK es solo el contrato externo.
</Warning>

Las superficies públicas de plugins empaquetados cargadas mediante fachada (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` y archivos públicos de entrada similares) ahora prefieren la
instantánea activa de configuración del runtime cuando OpenClaw ya está en ejecución. Si aún no existe ninguna
instantánea de runtime, vuelven a la configuración resuelta en disco.

Los plugins de proveedor también pueden exponer un barrel de contrato local del plugin y más específico cuando un
helper es intencionalmente específico del proveedor y todavía no pertenece a una subruta genérica del SDK.
Ejemplo empaquetado actual: el proveedor Anthropic mantiene sus helpers de stream de Claude
en su propia superficie pública `api.ts` / `contract-api.ts` en lugar de
promover la lógica de encabezados beta de Anthropic y `service_tier` a un contrato genérico
`plugin-sdk/*`.

Otros ejemplos empaquetados actuales:

- `@openclaw/openai-provider`: `api.ts` exporta constructores de proveedor,
  helpers de modelo predeterminado y constructores de proveedor en tiempo real
- `@openclaw/openrouter-provider`: `api.ts` exporta el constructor del proveedor además de
  helpers de incorporación/configuración

<Warning>
  El código de producción de extensiones también debe evitar las importaciones `openclaw/plugin-sdk/<other-plugin>`.
  Si un helper es realmente compartido, promuévelo a una subruta neutral del SDK
  como `openclaw/plugin-sdk/speech`, `.../provider-model-shared` u otra
  superficie orientada a capacidades en lugar de acoplar dos plugins entre sí.
</Warning>

## Relacionado

- [Puntos de entrada](/es/plugins/sdk-entrypoints) — opciones de `definePluginEntry` y `defineChannelPluginEntry`
- [Helpers de runtime](/es/plugins/sdk-runtime) — referencia completa del espacio de nombres `api.runtime`
- [Configuración y setup](/es/plugins/sdk-setup) — empaquetado, manifiestos, esquemas de configuración
- [Pruebas](/es/plugins/sdk-testing) — utilidades de prueba y reglas de lint
- [Migración del SDK](/es/plugins/sdk-migration) — migración desde superficies obsoletas
- [Internals del Plugin](/es/plugins/architecture) — arquitectura profunda y modelo de capacidades
