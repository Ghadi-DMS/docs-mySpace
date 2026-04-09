---
read_when:
    - Necesitas saber desde qué subruta del SDK importar
    - Quieres una referencia de todos los métodos de registro en OpenClawPluginApi
    - Estás buscando una exportación específica del SDK
sidebarTitle: SDK Overview
summary: Mapa de importación, referencia de la API de registro y arquitectura del SDK
title: Resumen del Plugin SDK
x-i18n:
    generated_at: "2026-04-09T01:30:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf205af060971931df97dca4af5110ce173d2b7c12f56ad7c62d664a402f2381
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Resumen del Plugin SDK

El plugin SDK es el contrato tipado entre los plugins y el núcleo. Esta página es la
referencia para **qué importar** y **qué puedes registrar**.

<Tip>
  **¿Buscas una guía práctica?**
  - ¿Tu primer plugin? Empieza con [Getting Started](/es/plugins/building-plugins)
  - ¿Plugin de canal? Consulta [Channel Plugins](/es/plugins/sdk-channel-plugins)
  - ¿Plugin de proveedor? Consulta [Provider Plugins](/es/plugins/sdk-provider-plugins)
</Tip>

## Convención de importación

Importa siempre desde una subruta específica:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subruta es un módulo pequeño y autocontenido. Esto mantiene un inicio rápido y
evita problemas de dependencias circulares. Para ayudantes de entrada/construcción específicos de canal,
prefiere `openclaw/plugin-sdk/channel-core`; conserva `openclaw/plugin-sdk/core` para
la superficie paraguas más amplia y ayudantes compartidos como
`buildChannelConfigSchema`.

No agregues ni dependas de puntos de acceso de conveniencia con nombre de proveedor como
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
puntos de acceso auxiliares con marca de canal. Los plugins integrados deben componer subrutas
genéricas del SDK dentro de sus propios barrels `api.ts` o `runtime-api.ts`, y el núcleo
debe usar esos barrels locales del plugin o agregar un contrato genérico y acotado del SDK
cuando la necesidad sea realmente entre canales.

El mapa de exportación generado todavía contiene un pequeño conjunto de puntos de acceso auxiliares
de plugins integrados como `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup`, y `plugin-sdk/matrix*`. Esas
subrutas existen solo para mantenimiento y compatibilidad de plugins integrados; se
omiten intencionalmente de la tabla común a continuación y no son la ruta de
importación recomendada para nuevos plugins de terceros.

## Referencia de subrutas

Las subrutas más utilizadas, agrupadas por propósito. La lista completa generada de
más de 200 subrutas está en `scripts/lib/plugin-sdk-entrypoints.json`.

Las subrutas auxiliares reservadas para plugins integrados siguen apareciendo en esa lista generada.
Trátalas como superficies de detalle de implementación/compatibilidad a menos que una página de documentación
promocione explícitamente una como pública.

### Entrada del plugin

| Subruta                    | Exportaciones clave                                                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="Subrutas de canal">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Exportación del esquema Zod raíz de `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, además de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Ayudantes compartidos del asistente de configuración, prompts de lista de permitidos, constructores de estado de configuración |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Ayudantes de configuración de múltiples cuentas/puerta de acciones, ayudantes de fallback de cuenta predeterminada |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, ayudantes de normalización de id de cuenta |
    | `plugin-sdk/account-resolution` | Búsqueda de cuentas + ayudantes de fallback predeterminado |
    | `plugin-sdk/account-helpers` | Ayudantes acotados para lista de cuentas/acciones de cuenta |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipos de esquema de configuración de canal |
    | `plugin-sdk/telegram-command-config` | Ayudantes de normalización/validación de comandos personalizados de Telegram con fallback de contrato integrado |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Ayudantes compartidos de ruta entrante + construcción de sobre |
    | `plugin-sdk/inbound-reply-dispatch` | Ayudantes compartidos de registro y despacho entrante |
    | `plugin-sdk/messaging-targets` | Ayudantes de análisis/coincidencia de destinos |
    | `plugin-sdk/outbound-media` | Ayudantes compartidos de carga de medios salientes |
    | `plugin-sdk/outbound-runtime` | Ayudantes de identidad saliente/delegado de envío |
    | `plugin-sdk/thread-bindings-runtime` | Ayudantes de ciclo de vida y adaptador de vinculaciones de hilo |
    | `plugin-sdk/agent-media-payload` | Constructor heredado de payload de medios del agente |
    | `plugin-sdk/conversation-runtime` | Ayudantes de vinculación de conversación/hilo, emparejamiento y vinculación configurada |
    | `plugin-sdk/runtime-config-snapshot` | Ayudante de snapshot de configuración en tiempo de ejecución |
    | `plugin-sdk/runtime-group-policy` | Ayudantes de resolución de políticas de grupo en tiempo de ejecución |
    | `plugin-sdk/channel-status` | Ayudantes compartidos de snapshot/resumen del estado del canal |
    | `plugin-sdk/channel-config-primitives` | Primitivas acotadas de esquema de configuración de canal |
    | `plugin-sdk/channel-config-writes` | Ayudantes de autorización para escrituras de configuración de canal |
    | `plugin-sdk/channel-plugin-common` | Exportaciones de preludio compartidas de plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Ayudantes de edición/lectura de configuración de lista de permitidos |
    | `plugin-sdk/group-access` | Ayudantes compartidos de decisión de acceso a grupos |
    | `plugin-sdk/direct-dm` | Ayudantes compartidos de autenticación/protección de DM directa |
    | `plugin-sdk/interactive-runtime` | Ayudantes de normalización/reducción de payload de respuesta interactiva |
    | `plugin-sdk/channel-inbound` | Ayudantes de debounce entrante, coincidencia de menciones, política de menciones y sobres |
    | `plugin-sdk/channel-send-result` | Tipos de resultado de respuesta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Ayudantes de análisis/coincidencia de destinos |
    | `plugin-sdk/channel-contract` | Tipos de contrato de canal |
    | `plugin-sdk/channel-feedback` | Cableado de feedback/reacciones |
    | `plugin-sdk/channel-secret-runtime` | Ayudantes acotados de contrato de secretos como `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, y tipos de destino de secretos |
  </Accordion>

  <Accordion title="Subrutas de proveedor">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Ayudantes curados de configuración de proveedores locales/autohospedados |
    | `plugin-sdk/self-hosted-provider-setup` | Ayudantes centrados en la configuración de proveedores autohospedados compatibles con OpenAI |
    | `plugin-sdk/cli-backend` | Valores predeterminados del backend CLI + constantes watchdog |
    | `plugin-sdk/provider-auth-runtime` | Ayudantes de resolución de claves API en tiempo de ejecución para plugins de proveedores |
    | `plugin-sdk/provider-auth-api-key` | Ayudantes de incorporación/escritura de perfiles para claves API como `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructor estándar de resultados de autenticación OAuth |
    | `plugin-sdk/provider-auth-login` | Ayudantes compartidos de inicio de sesión interactivo para plugins de proveedores |
    | `plugin-sdk/provider-env-vars` | Ayudantes de búsqueda de variables de entorno de autenticación de proveedor |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de política de reproducción, ayudantes de endpoints de proveedor, y ayudantes de normalización de identificadores de modelo como `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Ayudantes genéricos de capacidades HTTP/endpoint para proveedores |
    | `plugin-sdk/provider-web-fetch-contract` | Ayudantes acotados de contrato para configuración/selección de web-fetch como `enablePluginInConfig` y `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Ayudantes de registro/caché de proveedores web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Ayudantes acotados de configuración/credenciales de búsqueda web para proveedores que no necesitan cableado de habilitación del plugin |
    | `plugin-sdk/provider-web-search-contract` | Ayudantes acotados de contrato para configuración/credenciales de búsqueda web como `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, y setters/getters de credenciales con ámbito |
    | `plugin-sdk/provider-web-search` | Ayudantes de registro/caché/tiempo de ejecución de proveedores de búsqueda web |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquemas Gemini, y ayudantes de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` y similares |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de envoltorios de stream, y ayudantes compartidos de envoltorios para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Ayudantes de parcheo de configuración para onboarding |
    | `plugin-sdk/global-singleton` | Ayudantes de singleton/mapa/caché locales al proceso |
  </Accordion>

  <Accordion title="Subrutas de autenticación y seguridad">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, ayudantes de registro de comandos, ayudantes de autorización del remitente |
    | `plugin-sdk/command-status` | Constructores de mensajes de comando/ayuda como `buildCommandsMessagePaginated` y `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Resolución de aprobadores y ayudantes de autenticación de acciones en el mismo chat |
    | `plugin-sdk/approval-client-runtime` | Ayudantes de perfiles/filtros de aprobación nativa de ejecución |
    | `plugin-sdk/approval-delivery-runtime` | Adaptadores compartidos de capacidad/entrega de aprobación nativa |
    | `plugin-sdk/approval-gateway-runtime` | Ayudante compartido de resolución de gateway de aprobación |
    | `plugin-sdk/approval-handler-adapter-runtime` | Ayudantes ligeros de carga de adaptadores de aprobación nativa para entrypoints rápidos de canal |
    | `plugin-sdk/approval-handler-runtime` | Ayudantes más amplios del tiempo de ejecución del manejador de aprobación; prefiere los puntos de acceso más acotados de adaptador/gateway cuando sean suficientes |
    | `plugin-sdk/approval-native-runtime` | Ayudantes de destino de aprobación nativa + vinculación de cuenta |
    | `plugin-sdk/approval-reply-runtime` | Ayudantes de payload de respuesta de aprobación de exec/plugin |
    | `plugin-sdk/command-auth-native` | Ayudantes de autenticación de comandos nativos + destinos de sesión nativa |
    | `plugin-sdk/command-detection` | Ayudantes compartidos de detección de comandos |
    | `plugin-sdk/command-surface` | Ayudantes de normalización del cuerpo de comandos y superficie de comandos |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Ayudantes acotados de recopilación de contratos de secretos para superficies secretas de canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Ayudantes acotados `coerceSecretRef` y de tipado SecretRef para análisis de contratos/configuración de secretos |
    | `plugin-sdk/security-runtime` | Ayudantes compartidos de confianza, protección de DM, contenido externo y recopilación de secretos |
    | `plugin-sdk/ssrf-policy` | Ayudantes de lista de permitidos de hosts y política SSRF de red privada |
    | `plugin-sdk/ssrf-runtime` | Ayudantes de dispatcher fijado, fetch protegido con SSRF y política SSRF |
    | `plugin-sdk/secret-input` | Ayudantes de análisis de entradas secretas |
    | `plugin-sdk/webhook-ingress` | Ayudantes de solicitudes/destinos de webhook |
    | `plugin-sdk/webhook-request-guards` | Ayudantes de tamaño del cuerpo de la solicitud/timeout |
  </Accordion>

  <Accordion title="Subrutas de tiempo de ejecución y almacenamiento">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/runtime` | Ayudantes amplios de tiempo de ejecución/logging/copias de seguridad/instalación de plugins |
    | `plugin-sdk/runtime-env` | Ayudantes acotados de entorno de ejecución, logger, timeout, reintento y backoff |
    | `plugin-sdk/channel-runtime-context` | Ayudantes genéricos de registro y búsqueda de contexto de tiempo de ejecución de canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Ayudantes compartidos de comandos/hooks/http/interacción de plugins |
    | `plugin-sdk/hook-runtime` | Ayudantes compartidos de pipeline de hooks webhook/internos |
    | `plugin-sdk/lazy-runtime` | Ayudantes de importación/vinculación lazy en tiempo de ejecución como `createLazyRuntimeModule`, `createLazyRuntimeMethod`, y `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Ayudantes de ejecución de procesos |
    | `plugin-sdk/cli-runtime` | Ayudantes de formato CLI, espera y versión |
    | `plugin-sdk/gateway-runtime` | Ayudantes de cliente de gateway y parcheo de estado de canal |
    | `plugin-sdk/config-runtime` | Ayudantes de carga/escritura de configuración |
    | `plugin-sdk/telegram-command-config` | Normalización de nombres/descripciones de comandos de Telegram y comprobaciones de duplicados/conflictos, incluso cuando la superficie de contrato integrada de Telegram no está disponible |
    | `plugin-sdk/approval-runtime` | Ayudantes de aprobación de exec/plugin, constructores de capacidad de aprobación, ayudantes de autenticación/perfiles, y ayudantes nativos de enrutamiento/tiempo de ejecución |
    | `plugin-sdk/reply-runtime` | Ayudantes compartidos de tiempo de ejecución para entrada/respuesta, fragmentación, despacho, heartbeat, planificador de respuestas |
    | `plugin-sdk/reply-dispatch-runtime` | Ayudantes acotados para despacho/finalización de respuestas |
    | `plugin-sdk/reply-history` | Ayudantes compartidos de historial de respuestas de ventana corta como `buildHistoryContext`, `recordPendingHistoryEntry`, y `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Ayudantes acotados de fragmentación de texto/markdown |
    | `plugin-sdk/session-store-runtime` | Ayudantes de ruta de almacén de sesiones + `updated-at` |
    | `plugin-sdk/state-paths` | Ayudantes de rutas de directorios de estado/OAuth |
    | `plugin-sdk/routing` | Ayudantes de vinculación de rutas/claves de sesión/cuentas como `resolveAgentRoute`, `buildAgentSessionKey`, y `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Ayudantes compartidos de resumen de estado de canal/cuenta, valores predeterminados de estado de tiempo de ejecución y ayudantes de metadatos de incidencias |
    | `plugin-sdk/target-resolver-runtime` | Ayudantes compartidos de resolución de destinos |
    | `plugin-sdk/string-normalization-runtime` | Ayudantes de normalización de slug/cadenas |
    | `plugin-sdk/request-url` | Extrae URL de cadena desde entradas tipo fetch/request |
    | `plugin-sdk/run-command` | Ejecutor de comandos temporizado con resultados stdout/stderr normalizados |
    | `plugin-sdk/param-readers` | Lectores comunes de parámetros para herramientas/CLI |
    | `plugin-sdk/tool-payload` | Extrae payloads normalizados desde objetos de resultado de herramientas |
    | `plugin-sdk/tool-send` | Extrae campos de destino de envío canónicos desde argumentos de herramientas |
    | `plugin-sdk/temp-path` | Ayudantes compartidos de rutas temporales de descarga |
    | `plugin-sdk/logging-core` | Logger de subsistema y ayudantes de redacción |
    | `plugin-sdk/markdown-table-runtime` | Ayudantes de modo de tabla Markdown |
    | `plugin-sdk/json-store` | Pequeños ayudantes de lectura/escritura de estado JSON |
    | `plugin-sdk/file-lock` | Ayudantes reentrantes de bloqueo de archivos |
    | `plugin-sdk/persistent-dedupe` | Ayudantes de caché de deduplicación persistente en disco |
    | `plugin-sdk/acp-runtime` | Ayudantes de tiempo de ejecución/sesión ACP y de despacho de respuestas |
    | `plugin-sdk/agent-config-primitives` | Primitivas acotadas de esquema de configuración de tiempo de ejecución del agente |
    | `plugin-sdk/boolean-param` | Lector flexible de parámetros booleanos |
    | `plugin-sdk/dangerous-name-runtime` | Ayudantes de resolución para coincidencia de nombres peligrosos |
    | `plugin-sdk/device-bootstrap` | Ayudantes de bootstrap del dispositivo y tokens de emparejamiento |
    | `plugin-sdk/extension-shared` | Primitivas auxiliares compartidas para canal pasivo, estado y proxy ambiental |
    | `plugin-sdk/models-provider-runtime` | Ayudantes del comando `/models` y de respuesta de proveedores |
    | `plugin-sdk/skill-commands-runtime` | Ayudantes de listado de comandos de Skills |
    | `plugin-sdk/native-command-registry` | Ayudantes de registro/construcción/serialización de comandos nativos |
    | `plugin-sdk/provider-zai-endpoint` | Ayudantes de detección de endpoints de Z.AI |
    | `plugin-sdk/infra-runtime` | Ayudantes de eventos del sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Pequeños ayudantes de caché acotada |
    | `plugin-sdk/diagnostic-runtime` | Ayudantes de indicadores y eventos de diagnóstico |
    | `plugin-sdk/error-runtime` | Grafo de errores, formato, ayudantes compartidos de clasificación de errores, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Ayudantes de fetch envuelto, proxy y búsquedas fijadas |
    | `plugin-sdk/host-runtime` | Ayudantes de normalización de hostname y host SCP |
    | `plugin-sdk/retry-runtime` | Ayudantes de configuración y ejecución de reintentos |
    | `plugin-sdk/agent-runtime` | Ayudantes de directorio/identidad/workspace del agente |
    | `plugin-sdk/directory-runtime` | Consulta/deduplicación de directorios respaldada por configuración |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subrutas de capacidades y pruebas">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Ayudantes compartidos de obtención/transformación/almacenamiento de medios más constructores de payload de medios |
    | `plugin-sdk/media-generation-runtime` | Ayudantes compartidos de failover en generación de medios, selección de candidatos y mensajes de modelo ausente |
    | `plugin-sdk/media-understanding` | Tipos de proveedor de comprensión de medios más exportaciones auxiliares de imagen/audio para proveedores |
    | `plugin-sdk/text-runtime` | Ayudantes compartidos de texto/markdown/logging como eliminación de texto visible para el asistente, renderizado/fragmentación/tablas de markdown, ayudantes de redacción, ayudantes de etiquetas de directivas y utilidades de texto seguro |
    | `plugin-sdk/text-chunking` | Ayudante de fragmentación de texto saliente |
    | `plugin-sdk/speech` | Tipos de proveedor de voz más exportaciones auxiliares de directivas, registro y validación para proveedores |
    | `plugin-sdk/speech-core` | Tipos compartidos de proveedor de voz, registro, directivas y ayudantes de normalización |
    | `plugin-sdk/realtime-transcription` | Tipos de proveedor de transcripción en tiempo real y ayudantes de registro |
    | `plugin-sdk/realtime-voice` | Tipos de proveedor de voz en tiempo real y ayudantes de registro |
    | `plugin-sdk/image-generation` | Tipos de proveedor de generación de imágenes |
    | `plugin-sdk/image-generation-core` | Tipos compartidos de generación de imágenes, failover, autenticación y ayudantes de registro |
    | `plugin-sdk/music-generation` | Tipos de proveedor/solicitud/resultado de generación musical |
    | `plugin-sdk/music-generation-core` | Tipos compartidos de generación musical, ayudantes de failover, búsqueda de proveedores y análisis de referencias de modelos |
    | `plugin-sdk/video-generation` | Tipos de proveedor/solicitud/resultado de generación de video |
    | `plugin-sdk/video-generation-core` | Tipos compartidos de generación de video, ayudantes de failover, búsqueda de proveedores y análisis de referencias de modelos |
    | `plugin-sdk/webhook-targets` | Registro de destinos webhook y ayudantes de instalación de rutas |
    | `plugin-sdk/webhook-path` | Ayudantes de normalización de rutas webhook |
    | `plugin-sdk/web-media` | Ayudantes compartidos de carga de medios remotos/locales |
    | `plugin-sdk/zod` | `zod` reexportado para consumidores del plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Subrutas de memoria">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie auxiliar integrada memory-core para ayudantes de manager/config/archivos/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fachada de tiempo de ejecución para índice/búsqueda de memoria |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportaciones del motor base del host de memoria |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exportaciones del motor de embeddings del host de memoria |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportaciones del motor QMD del host de memoria |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportaciones del motor de almacenamiento del host de memoria |
    | `plugin-sdk/memory-core-host-multimodal` | Ayudantes multimodales del host de memoria |
    | `plugin-sdk/memory-core-host-query` | Ayudantes de consulta del host de memoria |
    | `plugin-sdk/memory-core-host-secret` | Ayudantes de secretos del host de memoria |
    | `plugin-sdk/memory-core-host-events` | Ayudantes del diario de eventos del host de memoria |
    | `plugin-sdk/memory-core-host-status` | Ayudantes de estado del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-cli` | Ayudantes de tiempo de ejecución CLI del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-core` | Ayudantes centrales de tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-files` | Ayudantes de archivos/tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-core` | Alias neutral respecto al proveedor para los ayudantes centrales de tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-events` | Alias neutral respecto al proveedor para los ayudantes del diario de eventos del host de memoria |
    | `plugin-sdk/memory-host-files` | Alias neutral respecto al proveedor para los ayudantes de archivos/tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-markdown` | Ayudantes compartidos de markdown administrado para plugins adyacentes a memoria |
    | `plugin-sdk/memory-host-search` | Fachada activa de tiempo de ejecución de memoria para acceso al gestor de búsquedas |
    | `plugin-sdk/memory-host-status` | Alias neutral respecto al proveedor para los ayudantes de estado del host de memoria |
    | `plugin-sdk/memory-lancedb` | Superficie auxiliar integrada memory-lancedb |
  </Accordion>

  <Accordion title="Subrutas reservadas de ayudas integradas">
    | Familia | Subrutas actuales | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Ayudantes de soporte para el plugin integrado de navegador (`browser-support` sigue siendo el barrel de compatibilidad) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie auxiliar/de tiempo de ejecución integrada de Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie auxiliar/de tiempo de ejecución integrada de LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie auxiliar integrada de IRC |
    | Ayudantes específicos de canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Puntos de acceso de compatibilidad/ayuda para canales integrados |
    | Ayudantes específicos de auth/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Puntos de acceso auxiliares de funciones/plugins integrados; `plugin-sdk/github-copilot-token` exporta actualmente `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken`, y `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API de registro

El callback `register(api)` recibe un objeto `OpenClawPluginApi` con estos
métodos:

### Registro de capacidades

| Método                                           | Qué registra                    |
| ------------------------------------------------ | ------------------------------- |
| `api.registerProvider(...)`                      | Inferencia de texto (LLM)       |
| `api.registerCliBackend(...)`                    | Backend local de inferencia CLI |
| `api.registerChannel(...)`                       | Canal de mensajería             |
| `api.registerSpeechProvider(...)`                | Síntesis de texto a voz / STT   |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcripción en tiempo real por streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sesiones dúplex de voz en tiempo real |
| `api.registerMediaUnderstandingProvider(...)`    | Análisis de imagen/audio/video  |
| `api.registerImageGenerationProvider(...)`       | Generación de imágenes          |
| `api.registerMusicGenerationProvider(...)`       | Generación de música            |
| `api.registerVideoGenerationProvider(...)`       | Generación de video             |
| `api.registerWebFetchProvider(...)`              | Proveedor de obtención/scraping web |
| `api.registerWebSearchProvider(...)`             | Búsqueda web                    |

### Herramientas y comandos

| Método                          | Qué registra                                 |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Herramienta del agente (requerida o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizado (omite el LLM)         |

### Infraestructura

| Método                                         | Qué registra                           |
| ---------------------------------------------- | -------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook de eventos                        |
| `api.registerHttpRoute(params)`                | Endpoint HTTP de la gateway            |
| `api.registerGatewayMethod(name, handler)`     | Método RPC de la gateway               |
| `api.registerCli(registrar, opts?)`            | Subcomando de CLI                      |
| `api.registerService(service)`                 | Servicio en segundo plano              |
| `api.registerInteractiveHandler(registration)` | Manejador interactivo                  |
| `api.registerMemoryPromptSupplement(builder)`  | Sección de prompt aditiva adyacente a memoria |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus aditivo de búsqueda/lectura de memoria |

Los espacios de nombres administrativos reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre permanecen como `operator.admin`, incluso si un plugin intenta asignar un
ámbito más restringido a un método de gateway. Prefiere prefijos específicos del plugin para
métodos propiedad del plugin.

### Metadatos de registro de CLI

`api.registerCli(registrar, opts?)` acepta dos tipos de metadatos de nivel superior:

- `commands`: raíces de comandos explícitas propiedad del registrador
- `descriptors`: descriptores de comandos en tiempo de análisis usados para la ayuda del CLI raíz,
  enrutamiento y registro lazy del CLI del plugin

Si quieres que un comando de plugin permanezca con lazy-loading en la ruta normal del CLI raíz,
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
        description: "Administra cuentas, verificación, dispositivos y estado de perfil de Matrix",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` por sí solo solo cuando no necesites registro lazy del CLI raíz.
Esa ruta de compatibilidad eager sigue siendo compatible, pero no instala
marcadores de posición respaldados por descriptores para lazy loading en tiempo de análisis.

### Registro de backend CLI

`api.registerCliBackend(...)` permite que un plugin se encargue de la configuración predeterminada de un
backend CLI local de IA como `codex-cli`.

- El `id` del backend se convierte en el prefijo del proveedor en referencias de modelos como `codex-cli/gpt-5`.
- La `config` del backend usa la misma forma que `agents.defaults.cliBackends.<id>`.
- La configuración del usuario sigue teniendo prioridad. OpenClaw fusiona `agents.defaults.cliBackends.<id>` sobre la
  predeterminada del plugin antes de ejecutar el CLI.
- Usa `normalizeConfig` cuando un backend necesite reescrituras de compatibilidad después de la fusión
  (por ejemplo, normalizar formas antiguas de flags).

### Ranuras exclusivas

| Método                                     | Qué registra                                                                                                                                          |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motor de contexto (solo uno activo a la vez). El callback `assemble()` recibe `availableTools` y `citationsMode` para que el motor pueda adaptar las adiciones al prompt. |
| `api.registerMemoryCapability(capability)` | Capacidad unificada de memoria                                                                                                                        |
| `api.registerMemoryPromptSection(builder)` | Constructor de sección de prompt de memoria                                                                                                           |
| `api.registerMemoryFlushPlan(resolver)`    | Resolvedor de plan de vaciado de memoria                                                                                                              |
| `api.registerMemoryRuntime(runtime)`       | Adaptador de tiempo de ejecución de memoria                                                                                                           |

### Adaptadores de embeddings de memoria

| Método                                         | Qué registra                                   |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de embeddings de memoria para el plugin activo |

- `registerMemoryCapability` es la API exclusiva de plugin de memoria preferida.
- `registerMemoryCapability` también puede exponer `publicArtifacts.listArtifacts(...)`
  para que los plugins complementarios consuman artefactos de memoria exportados mediante
  `openclaw/plugin-sdk/memory-host-core` en lugar de acceder al diseño privado de un
  plugin de memoria específico.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan`, y
  `registerMemoryRuntime` son API exclusivas de plugin de memoria compatibles con el legado.
- `registerMemoryEmbeddingProvider` permite que el plugin de memoria activo registre uno
  o más ids de adaptadores de embeddings (por ejemplo `openai`, `gemini`, o un id personalizado definido por el plugin).
- La configuración del usuario, como `agents.defaults.memorySearch.provider` y
  `agents.defaults.memorySearch.fallback`, se resuelve contra esos ids de adaptadores registrados.

### Eventos y ciclo de vida

| Método                                       | Qué hace                    |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook de ciclo de vida tipado |
| `api.onConversationBindingResolved(handler)` | Callback de vinculación de conversación |

### Semántica de decisión de hooks

- `before_tool_call`: devolver `{ block: true }` es terminal. Una vez que cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `before_tool_call`: devolver `{ block: false }` se trata como sin decisión (igual que omitir `block`), no como una anulación.
- `before_install`: devolver `{ block: true }` es terminal. Una vez que cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `before_install`: devolver `{ block: false }` se trata como sin decisión (igual que omitir `block`), no como una anulación.
- `reply_dispatch`: devolver `{ handled: true, ... }` es terminal. Una vez que cualquier manejador reclama el despacho, se omiten los manejadores de menor prioridad y la ruta predeterminada de despacho del modelo.
- `message_sending`: devolver `{ cancel: true }` es terminal. Una vez que cualquier manejador lo establece, se omiten los manejadores de menor prioridad.
- `message_sending`: devolver `{ cancel: false }` se trata como sin decisión (igual que omitir `cancel`), no como una anulación.

### Campos del objeto API

| Campo                    | Tipo                      | Descripción                                                                                  |
| ------------------------ | ------------------------- | -------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id del plugin                                                                                 |
| `api.name`               | `string`                  | Nombre para mostrar                                                                           |
| `api.version`            | `string?`                 | Versión del plugin (opcional)                                                                 |
| `api.description`        | `string?`                 | Descripción del plugin (opcional)                                                             |
| `api.source`             | `string`                  | Ruta de origen del plugin                                                                     |
| `api.rootDir`            | `string?`                 | Directorio raíz del plugin (opcional)                                                         |
| `api.config`             | `OpenClawConfig`          | Snapshot actual de configuración (snapshot activo en memoria del tiempo de ejecución cuando está disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin desde `plugins.entries.<id>.config`                      |
| `api.runtime`            | `PluginRuntime`           | [Ayudantes de tiempo de ejecución](/es/plugins/sdk-runtime)                                      |
| `api.logger`             | `PluginLogger`            | Logger con ámbito (`debug`, `info`, `warn`, `error`)                                          |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de inicio/configuración previa a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resuelve la ruta relativa a la raíz del plugin                                                |

## Convención de módulos internos

Dentro de tu plugin, usa archivos barrel locales para las importaciones internas:

```
my-plugin/
  api.ts            # Exportaciones públicas para consumidores externos
  runtime-api.ts    # Exportaciones internas solo para tiempo de ejecución
  index.ts          # Punto de entrada del plugin
  setup-entry.ts    # Entrada ligera solo para configuración (opcional)
```

<Warning>
  Nunca importes tu propio plugin mediante `openclaw/plugin-sdk/<your-plugin>`
  desde código de producción. Dirige las importaciones internas a través de `./api.ts` o
  `./runtime-api.ts`. La ruta del SDK es solo el contrato externo.
</Warning>

Las superficies públicas de plugins integrados cargadas por fachada (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts`, y archivos de entrada pública similares) ahora prefieren el
snapshot activo de configuración en tiempo de ejecución cuando OpenClaw ya se está ejecutando. Si todavía
no existe un snapshot en tiempo de ejecución, recurren al archivo de configuración resuelto en disco.

Los plugins de proveedores también pueden exponer un barrel local del plugin con un contrato acotado cuando un
ayudante es intencionalmente específico del proveedor y aún no pertenece a una subruta genérica
del SDK. Ejemplo integrado actual: el proveedor Anthropic conserva sus
ayudantes de stream Claude en su propio punto de acceso público `api.ts` / `contract-api.ts` en lugar de
promover la lógica de encabezado beta de Anthropic y `service_tier` a un contrato genérico
`plugin-sdk/*`.

Otros ejemplos integrados actuales:

- `@openclaw/openai-provider`: `api.ts` exporta constructores de proveedores,
  ayudantes de modelo predeterminado y constructores de proveedores realtime
- `@openclaw/openrouter-provider`: `api.ts` exporta el constructor del proveedor más
  ayudantes de onboarding/configuración

<Warning>
  El código de producción de extensiones también debe evitar las importaciones
  `openclaw/plugin-sdk/<other-plugin>`. Si un ayudante es realmente compartido, promuévelo a una subruta neutra del SDK
  como `openclaw/plugin-sdk/speech`, `.../provider-model-shared`, u otra
  superficie orientada a capacidades en lugar de acoplar dos plugins entre sí.
</Warning>

## Relacionado

- [Entry Points](/es/plugins/sdk-entrypoints) — opciones de `definePluginEntry` y `defineChannelPluginEntry`
- [Runtime Helpers](/es/plugins/sdk-runtime) — referencia completa del espacio de nombres `api.runtime`
- [Setup and Config](/es/plugins/sdk-setup) — empaquetado, manifiestos, esquemas de configuración
- [Testing](/es/plugins/sdk-testing) — utilidades de prueba y reglas de lint
- [SDK Migration](/es/plugins/sdk-migration) — migración desde superficies obsoletas
- [Plugin Internals](/es/plugins/architecture) — arquitectura profunda y modelo de capacidades
