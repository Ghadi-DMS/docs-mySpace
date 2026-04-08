---
read_when:
    - Necesita saber desde qué subruta del SDK importar
    - Quiere una referencia de todos los métodos de registro en OpenClawPluginApi
    - Está buscando una exportación específica del SDK
sidebarTitle: SDK Overview
summary: Mapa de importaciones, referencia de la API de registro y arquitectura del SDK
title: Descripción general del Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:18:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: c5a41bd82d165dfbb7fbd6e4528cf322e9133a51efe55fa8518a7a0a626d9d30
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Descripción general del Plugin SDK

El Plugin SDK es el contrato tipado entre los plugins y el núcleo. Esta página es la
referencia de **qué importar** y **qué se puede registrar**.

<Tip>
  **¿Busca una guía práctica?**
  - ¿Su primer plugin? Empiece con [Primeros pasos](/es/plugins/building-plugins)
  - ¿Plugin de canal? Vea [Plugins de canal](/es/plugins/sdk-channel-plugins)
  - ¿Plugin de proveedor? Vea [Plugins de proveedor](/es/plugins/sdk-provider-plugins)
</Tip>

## Convención de importación

Importe siempre desde una subruta específica:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subruta es un módulo pequeño y autocontenido. Esto mantiene un inicio rápido y
evita problemas de dependencias circulares. Para los auxiliares específicos de entrada/compilación de canales,
prefiera `openclaw/plugin-sdk/channel-core`; reserve `openclaw/plugin-sdk/core` para
la superficie paraguas más amplia y auxiliares compartidos como
`buildChannelConfigSchema`.

No agregue ni dependa de capas de conveniencia con nombre de proveedor como
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
capas auxiliares con marca de canal. Los plugins integrados deben componer subrutas
genéricas del SDK dentro de sus propios archivos barril `api.ts` o `runtime-api.ts`, y el núcleo
debe usar esos barriles locales del plugin o agregar un contrato del SDK genérico y acotado
cuando la necesidad sea realmente transversal a varios canales.

El mapa de exportaciones generado aún contiene un pequeño conjunto de
capas auxiliares para plugins integrados como `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Esas
subrutas existen solo para mantenimiento y compatibilidad de plugins integrados; se
omiten intencionalmente de la tabla común a continuación y no son la ruta de importación recomendada para plugins nuevos de terceros.

## Referencia de subrutas

Las subrutas de uso más común, agrupadas por propósito. La lista completa generada de
más de 200 subrutas está en `scripts/lib/plugin-sdk-entrypoints.json`.

Las subrutas auxiliares reservadas para plugins integrados siguen apareciendo en esa lista generada.
Trátelas como detalles de implementación o superficies de compatibilidad, a menos que una página de documentación
promocione explícitamente una como pública.

### Entrada de plugin

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
    | `plugin-sdk/setup` | Auxiliares compartidos para el asistente de configuración, prompts de lista de permitidos y constructores de estado de configuración |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Auxiliares de configuración/múltiples cuentas/bloqueo de acciones, y auxiliares de respaldo para cuenta predeterminada |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, auxiliares de normalización de id de cuenta |
    | `plugin-sdk/account-resolution` | Búsqueda de cuentas + auxiliares de respaldo predeterminado |
    | `plugin-sdk/account-helpers` | Auxiliares acotados para lista de cuentas/acciones de cuenta |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipos de esquema de configuración de canal |
    | `plugin-sdk/telegram-command-config` | Auxiliares de normalización/validación de comandos personalizados de Telegram con respaldo de contrato integrado |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Auxiliares compartidos para rutas entrantes y construcción de sobres |
    | `plugin-sdk/inbound-reply-dispatch` | Auxiliares compartidos para registrar y despachar entradas |
    | `plugin-sdk/messaging-targets` | Auxiliares de análisis/coincidencia de destinos |
    | `plugin-sdk/outbound-media` | Auxiliares compartidos para carga de medios salientes |
    | `plugin-sdk/outbound-runtime` | Auxiliares de identidad saliente y delegado de envío |
    | `plugin-sdk/thread-bindings-runtime` | Auxiliares de ciclo de vida y adaptadores para enlaces de hilos |
    | `plugin-sdk/agent-media-payload` | Constructor heredado de carga útil multimedia del agente |
    | `plugin-sdk/conversation-runtime` | Auxiliares de enlaces de conversación/hilo, pairing y enlaces configurados |
    | `plugin-sdk/runtime-config-snapshot` | Auxiliar de instantánea de configuración en tiempo de ejecución |
    | `plugin-sdk/runtime-group-policy` | Auxiliares de resolución de políticas de grupo en tiempo de ejecución |
    | `plugin-sdk/channel-status` | Auxiliares compartidos para instantáneas/resúmenes de estado del canal |
    | `plugin-sdk/channel-config-primitives` | Primitivas acotadas de esquema de configuración de canal |
    | `plugin-sdk/channel-config-writes` | Auxiliares de autorización para escritura de configuración de canal |
    | `plugin-sdk/channel-plugin-common` | Exportaciones de preludio compartidas para plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Auxiliares para leer/editar configuración de listas de permitidos |
    | `plugin-sdk/group-access` | Auxiliares compartidos para decisiones de acceso a grupos |
    | `plugin-sdk/direct-dm` | Auxiliares compartidos de autenticación/protección de DM directos |
    | `plugin-sdk/interactive-runtime` | Auxiliares de normalización/reducción de cargas útiles de respuestas interactivas |
    | `plugin-sdk/channel-inbound` | Auxiliares para debounce de entrada, coincidencia de menciones, políticas de mención y sobres |
    | `plugin-sdk/channel-send-result` | Tipos de resultados de respuesta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Auxiliares de análisis/coincidencia de destinos |
    | `plugin-sdk/channel-contract` | Tipos de contratos de canal |
    | `plugin-sdk/channel-feedback` | Cableado de feedback/reacciones |
    | `plugin-sdk/channel-secret-runtime` | Auxiliares acotados de contrato de secretos como `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` y tipos de destino de secretos |
  </Accordion>

  <Accordion title="Subrutas de proveedor">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Auxiliares seleccionados para configurar proveedores locales/autohospedados |
    | `plugin-sdk/self-hosted-provider-setup` | Auxiliares centrados en la configuración de proveedores autohospedados compatibles con OpenAI |
    | `plugin-sdk/cli-backend` | Valores predeterminados del backend de CLI + constantes de watchdog |
    | `plugin-sdk/provider-auth-runtime` | Auxiliares de resolución de claves de API en tiempo de ejecución para plugins de proveedor |
    | `plugin-sdk/provider-auth-api-key` | Auxiliares de incorporación/escritura de perfiles de claves de API como `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructor estándar de resultados de autenticación OAuth |
    | `plugin-sdk/provider-auth-login` | Auxiliares interactivos compartidos de inicio de sesión para plugins de proveedor |
    | `plugin-sdk/provider-env-vars` | Auxiliares de búsqueda de variables de entorno para autenticación de proveedor |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de políticas de repetición, auxiliares de endpoint de proveedor y normalización de id de modelo como `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Auxiliares genéricos de HTTP/capacidades de endpoint para proveedores |
    | `plugin-sdk/provider-web-fetch-contract` | Auxiliares acotados para contratos de configuración/selección de web fetch como `enablePluginInConfig` y `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Auxiliares de registro/caché de proveedores de web fetch |
    | `plugin-sdk/provider-web-search-contract` | Auxiliares acotados para contratos de configuración/credenciales de búsqueda web como `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` y setters/getters de credenciales con alcance |
    | `plugin-sdk/provider-web-search` | Auxiliares de registro/caché/tiempo de ejecución para proveedores de búsqueda web |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquemas Gemini y auxiliares de compatibilidad de xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` y similares |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de envoltorios de streaming y auxiliares compartidos de envoltorios para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Auxiliares de parches de configuración para onboarding |
    | `plugin-sdk/global-singleton` | Auxiliares de singleton/mapa/caché locales al proceso |
  </Accordion>

  <Accordion title="Subrutas de autenticación y seguridad">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, auxiliares de registro de comandos, auxiliares de autorización de remitentes |
    | `plugin-sdk/approval-auth-runtime` | Resolución de aprobadores y auxiliares de autenticación de acciones en el mismo chat |
    | `plugin-sdk/approval-client-runtime` | Auxiliares nativos de perfiles/filtros de aprobación de exec |
    | `plugin-sdk/approval-delivery-runtime` | Adaptadores nativos de capacidad/entrega de aprobaciones |
    | `plugin-sdk/approval-gateway-runtime` | Auxiliar compartido de resolución del gateway de aprobaciones |
    | `plugin-sdk/approval-handler-adapter-runtime` | Auxiliares ligeros de carga de adaptadores nativos de aprobación para puntos de entrada de canales de acceso frecuente |
    | `plugin-sdk/approval-handler-runtime` | Auxiliares más amplios de tiempo de ejecución para controladores de aprobación; prefiera las capas más acotadas de adaptador/gateway cuando sean suficientes |
    | `plugin-sdk/approval-native-runtime` | Auxiliares nativos de destino de aprobación y enlace de cuentas |
    | `plugin-sdk/approval-reply-runtime` | Auxiliares de cargas útiles de respuesta para aprobaciones de exec/plugin |
    | `plugin-sdk/command-auth-native` | Autenticación nativa de comandos + auxiliares nativos de destino de sesión |
    | `plugin-sdk/command-detection` | Auxiliares compartidos de detección de comandos |
    | `plugin-sdk/command-surface` | Auxiliares de normalización del cuerpo del comando y de superficie de comandos |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Auxiliares acotados de recopilación de contratos de secretos para superficies de secretos de canales/plugins |
    | `plugin-sdk/secret-ref-runtime` | Auxiliares acotados `coerceSecretRef` y de tipado de SecretRef para análisis de contratos/configuración de secretos |
    | `plugin-sdk/security-runtime` | Auxiliares compartidos de confianza, control de DM, contenido externo y recopilación de secretos |
    | `plugin-sdk/ssrf-policy` | Auxiliares de lista de permitidos de hosts y política SSRF de red privada |
    | `plugin-sdk/ssrf-runtime` | Dispatcher fijado, fetch protegido por SSRF y auxiliares de política SSRF |
    | `plugin-sdk/secret-input` | Auxiliares de análisis de entradas secretas |
    | `plugin-sdk/webhook-ingress` | Auxiliares para solicitudes/destinos de webhook |
    | `plugin-sdk/webhook-request-guards` | Auxiliares para tamaño de cuerpo/tiempo de espera de solicitudes |
  </Accordion>

  <Accordion title="Subrutas de tiempo de ejecución y almacenamiento">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/runtime` | Auxiliares amplios de tiempo de ejecución/registro/copias de seguridad/instalación de plugins |
    | `plugin-sdk/runtime-env` | Auxiliares acotados de entorno de ejecución, logger, timeout, retry y backoff |
    | `plugin-sdk/channel-runtime-context` | Auxiliares genéricos de registro y búsqueda de contexto de tiempo de ejecución del canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Auxiliares compartidos para comandos/hooks/http/elementos interactivos del plugin |
    | `plugin-sdk/hook-runtime` | Auxiliares compartidos del flujo de webhook/hooks internos |
    | `plugin-sdk/lazy-runtime` | Auxiliares de importación/vinculación diferida del tiempo de ejecución como `createLazyRuntimeModule`, `createLazyRuntimeMethod` y `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Auxiliares para ejecución de procesos |
    | `plugin-sdk/cli-runtime` | Auxiliares de formato, espera y versión para CLI |
    | `plugin-sdk/gateway-runtime` | Cliente del gateway y auxiliares de parches para estado de canales |
    | `plugin-sdk/config-runtime` | Auxiliares para cargar/escribir configuración |
    | `plugin-sdk/telegram-command-config` | Normalización de nombre/descripción de comandos de Telegram y comprobaciones de duplicados/conflictos, incluso cuando la superficie de contrato integrada de Telegram no está disponible |
    | `plugin-sdk/approval-runtime` | Auxiliares de aprobación de exec/plugin, constructores de capacidades de aprobación, auxiliares de autenticación/perfiles y auxiliares nativos de enrutamiento/tiempo de ejecución |
    | `plugin-sdk/reply-runtime` | Auxiliares compartidos para entradas/respuestas en tiempo de ejecución, fragmentación, despacho, heartbeat y planificador de respuestas |
    | `plugin-sdk/reply-dispatch-runtime` | Auxiliares acotados para despacho/finalización de respuestas |
    | `plugin-sdk/reply-history` | Auxiliares compartidos para historial de respuestas en ventanas cortas como `buildHistoryContext`, `recordPendingHistoryEntry` y `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Auxiliares acotados de fragmentación de texto/Markdown |
    | `plugin-sdk/session-store-runtime` | Auxiliares de rutas del almacén de sesiones + `updated-at` |
    | `plugin-sdk/state-paths` | Auxiliares de rutas de directorios de estado/OAuth |
    | `plugin-sdk/routing` | Auxiliares de ruta/clave de sesión/enlace de cuenta como `resolveAgentRoute`, `buildAgentSessionKey` y `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Auxiliares compartidos para resúmenes de estado de canal/cuenta, valores predeterminados del estado de ejecución y auxiliares de metadatos de incidencias |
    | `plugin-sdk/target-resolver-runtime` | Auxiliares compartidos de resolución de destinos |
    | `plugin-sdk/string-normalization-runtime` | Auxiliares de normalización de slug/cadenas |
    | `plugin-sdk/request-url` | Extrae URL en cadena de entradas tipo fetch/request |
    | `plugin-sdk/run-command` | Ejecutor de comandos temporizado con resultados normalizados de stdout/stderr |
    | `plugin-sdk/param-readers` | Lectores comunes de parámetros de herramientas/CLI |
    | `plugin-sdk/tool-send` | Extrae campos canónicos de destino de envío a partir de argumentos de herramientas |
    | `plugin-sdk/temp-path` | Auxiliares compartidos para rutas temporales de descarga |
    | `plugin-sdk/logging-core` | Logger de subsistema y auxiliares de redacción |
    | `plugin-sdk/markdown-table-runtime` | Auxiliares del modo de tablas Markdown |
    | `plugin-sdk/json-store` | Auxiliares pequeños para leer/escribir estado JSON |
    | `plugin-sdk/file-lock` | Auxiliares reentrantes de bloqueo de archivos |
    | `plugin-sdk/persistent-dedupe` | Auxiliares de caché de deduplicación respaldada por disco |
    | `plugin-sdk/acp-runtime` | Auxiliares de tiempo de ejecución/sesión ACP y despacho de respuestas |
    | `plugin-sdk/agent-config-primitives` | Primitivas acotadas de esquema de configuración de tiempo de ejecución del agente |
    | `plugin-sdk/boolean-param` | Lector flexible de parámetros booleanos |
    | `plugin-sdk/dangerous-name-runtime` | Auxiliares de resolución para coincidencias de nombres peligrosos |
    | `plugin-sdk/device-bootstrap` | Auxiliares de arranque de dispositivos y tokens de pairing |
    | `plugin-sdk/extension-shared` | Primitivas auxiliares compartidas para canales pasivos, estado y proxy ambiental |
    | `plugin-sdk/models-provider-runtime` | Auxiliares de comandos `/models` y respuestas de proveedor |
    | `plugin-sdk/skill-commands-runtime` | Auxiliares para listados de comandos de Skills |
    | `plugin-sdk/native-command-registry` | Auxiliares nativos de registro/construcción/serialización de comandos |
    | `plugin-sdk/provider-zai-endpoint` | Auxiliares de detección de endpoints de Z.AI |
    | `plugin-sdk/infra-runtime` | Auxiliares de eventos del sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Auxiliares pequeños de caché acotada |
    | `plugin-sdk/diagnostic-runtime` | Auxiliares de indicadores y eventos de diagnóstico |
    | `plugin-sdk/error-runtime` | Grafo de errores, formato, auxiliares compartidos de clasificación de errores, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Fetch envuelto, proxy y auxiliares de búsqueda fijada |
    | `plugin-sdk/host-runtime` | Auxiliares de normalización de hostname y host SCP |
    | `plugin-sdk/retry-runtime` | Auxiliares de configuración y ejecución de retry |
    | `plugin-sdk/agent-runtime` | Auxiliares de directorio/identidad/espacio de trabajo del agente |
    | `plugin-sdk/directory-runtime` | Consulta/deduplicación de directorios respaldadas por configuración |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subrutas de capacidades y pruebas">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Auxiliares compartidos para obtener/transformar/almacenar medios, además de constructores de cargas útiles multimedia |
    | `plugin-sdk/media-generation-runtime` | Auxiliares compartidos para failover de generación multimedia, selección de candidatos y mensajes de falta de modelo |
    | `plugin-sdk/media-understanding` | Tipos de proveedores de comprensión multimedia más exportaciones orientadas a proveedores para auxiliares de imagen/audio |
    | `plugin-sdk/text-runtime` | Auxiliares compartidos para texto/Markdown/registro como eliminación de texto visible para el asistente, renderizado/fragmentación/tablas Markdown, auxiliares de redacción, auxiliares de etiquetas de directivas y utilidades de texto seguro |
    | `plugin-sdk/text-chunking` | Auxiliar de fragmentación de texto saliente |
    | `plugin-sdk/speech` | Tipos de proveedores de voz más auxiliares orientados a proveedores para directivas, registro y validación |
    | `plugin-sdk/speech-core` | Tipos compartidos de proveedores de voz, registro, directivas y auxiliares de normalización |
    | `plugin-sdk/realtime-transcription` | Tipos de proveedores de transcripción en tiempo real y auxiliares de registro |
    | `plugin-sdk/realtime-voice` | Tipos de proveedores de voz en tiempo real y auxiliares de registro |
    | `plugin-sdk/image-generation` | Tipos de proveedores de generación de imágenes |
    | `plugin-sdk/image-generation-core` | Tipos compartidos de generación de imágenes, failover, autenticación y auxiliares de registro |
    | `plugin-sdk/music-generation` | Tipos de proveedores/solicitudes/resultados de generación musical |
    | `plugin-sdk/music-generation-core` | Tipos compartidos de generación musical, auxiliares de failover, búsqueda de proveedores y análisis de referencias de modelos |
    | `plugin-sdk/video-generation` | Tipos de proveedores/solicitudes/resultados de generación de video |
    | `plugin-sdk/video-generation-core` | Tipos compartidos de generación de video, auxiliares de failover, búsqueda de proveedores y análisis de referencias de modelos |
    | `plugin-sdk/webhook-targets` | Registro de destinos de webhook y auxiliares de instalación de rutas |
    | `plugin-sdk/webhook-path` | Auxiliares de normalización de rutas de webhook |
    | `plugin-sdk/web-media` | Auxiliares compartidos para carga de medios remotos/locales |
    | `plugin-sdk/zod` | Reexportación de `zod` para consumidores del Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Subrutas de memoria">
    | Subruta | Exportaciones clave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie auxiliar integrada de memory-core para auxiliares de administrador/configuración/archivos/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fachada de tiempo de ejecución para indexación/búsqueda en memoria |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportaciones del motor base del host de memoria |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exportaciones del motor de embeddings del host de memoria |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportaciones del motor QMD del host de memoria |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportaciones del motor de almacenamiento del host de memoria |
    | `plugin-sdk/memory-core-host-multimodal` | Auxiliares multimodales del host de memoria |
    | `plugin-sdk/memory-core-host-query` | Auxiliares de consulta del host de memoria |
    | `plugin-sdk/memory-core-host-secret` | Auxiliares de secretos del host de memoria |
    | `plugin-sdk/memory-core-host-events` | Auxiliares del diario de eventos del host de memoria |
    | `plugin-sdk/memory-core-host-status` | Auxiliares de estado del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-cli` | Auxiliares de tiempo de ejecución CLI del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-core` | Auxiliares principales de tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-core-host-runtime-files` | Auxiliares de archivos/tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-core` | Alias neutral respecto al proveedor para auxiliares principales de tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-events` | Alias neutral respecto al proveedor para auxiliares del diario de eventos del host de memoria |
    | `plugin-sdk/memory-host-files` | Alias neutral respecto al proveedor para auxiliares de archivos/tiempo de ejecución del host de memoria |
    | `plugin-sdk/memory-host-markdown` | Auxiliares compartidos de Markdown administrado para plugins adyacentes a memoria |
    | `plugin-sdk/memory-host-search` | Fachada de tiempo de ejecución activa de memoria para acceso al administrador de búsqueda |
    | `plugin-sdk/memory-host-status` | Alias neutral respecto al proveedor para auxiliares de estado del host de memoria |
    | `plugin-sdk/memory-lancedb` | Superficie auxiliar integrada de memory-lancedb |
  </Accordion>

  <Accordion title="Subrutas reservadas de auxiliares integrados">
    | Familia | Subrutas actuales | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Auxiliares de soporte del plugin Browser integrado (`browser-support` sigue siendo el barril de compatibilidad) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie auxiliar/de tiempo de ejecución integrada de Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie auxiliar/de tiempo de ejecución integrada de LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie auxiliar integrada de IRC |
    | Auxiliares específicos de canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Capas auxiliares/de compatibilidad para canales integrados |
    | Auxiliares específicos de autenticación/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Capas auxiliares para funciones/plugins integrados; `plugin-sdk/github-copilot-token` actualmente exporta `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API de registro

La devolución de llamada `register(api)` recibe un objeto `OpenClawPluginApi` con estos
métodos:

### Registro de capacidades

| Método                                           | Qué registra                    |
| ------------------------------------------------ | ------------------------------- |
| `api.registerProvider(...)`                      | Inferencia de texto (LLM)       |
| `api.registerCliBackend(...)`                    | Backend de inferencia CLI local |
| `api.registerChannel(...)`                       | Canal de mensajería             |
| `api.registerSpeechProvider(...)`                | Síntesis de texto a voz / STT   |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcripción en tiempo real por streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sesiones de voz en tiempo real dúplex |
| `api.registerMediaUnderstandingProvider(...)`    | Análisis de imagen/audio/video  |
| `api.registerImageGenerationProvider(...)`       | Generación de imágenes          |
| `api.registerMusicGenerationProvider(...)`       | Generación musical              |
| `api.registerVideoGenerationProvider(...)`       | Generación de video             |
| `api.registerWebFetchProvider(...)`              | Proveedor de obtención / scraping web |
| `api.registerWebSearchProvider(...)`             | Búsqueda web                    |

### Herramientas y comandos

| Método                          | Qué registra                                  |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Herramienta del agente (obligatoria o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizado (omite el LLM)          |

### Infraestructura

| Método                                         | Qué registra                           |
| ---------------------------------------------- | -------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook de eventos                        |
| `api.registerHttpRoute(params)`                | Endpoint HTTP del gateway              |
| `api.registerGatewayMethod(name, handler)`     | Método RPC del gateway                 |
| `api.registerCli(registrar, opts?)`            | Subcomando CLI                         |
| `api.registerService(service)`                 | Servicio en segundo plano              |
| `api.registerInteractiveHandler(registration)` | Controlador interactivo                |
| `api.registerMemoryPromptSupplement(builder)`  | Sección aditiva del prompt adyacente a memoria |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus aditivo de lectura/búsqueda de memoria |

Los espacios de nombres administrativos reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) siempre permanecen como `operator.admin`, incluso si un plugin intenta asignar
un alcance más restringido a un método del gateway. Prefiera prefijos específicos del plugin para
los métodos propiedad del plugin.

### Metadatos de registro de CLI

`api.registerCli(registrar, opts?)` acepta dos tipos de metadatos de nivel superior:

- `commands`: raíces de comandos explícitas propiedad del registrador
- `descriptors`: descriptores de comandos en tiempo de análisis usados para la ayuda del CLI raíz,
  el enrutamiento y el registro diferido del CLI del plugin

Si quiere que un comando del plugin siga cargándose de forma diferida en la ruta normal del CLI raíz,
proporcione `descriptors` que cubran cada raíz de comando de nivel superior expuesta por ese
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
        description: "Administrar cuentas, verificación, dispositivos y estado de perfil de Matrix",
        hasSubcommands: true,
      },
    ],
  },
);
```

Use `commands` por sí solo únicamente cuando no necesite registro diferido en el CLI raíz.
Esa ruta de compatibilidad ansiosa sigue siendo compatible, pero no instala
marcadores respaldados por descriptores para la carga diferida en tiempo de análisis.

### Registro de backends de CLI

`api.registerCliBackend(...)` permite que un plugin posea la configuración predeterminada de un
backend de CLI de IA local como `codex-cli`.

- El `id` del backend se convierte en el prefijo del proveedor en referencias de modelo como `codex-cli/gpt-5`.
- La `config` del backend usa la misma forma que `agents.defaults.cliBackends.<id>`.
- La configuración del usuario sigue teniendo prioridad. OpenClaw fusiona `agents.defaults.cliBackends.<id>` sobre el
  valor predeterminado del plugin antes de ejecutar la CLI.
- Use `normalizeConfig` cuando un backend necesite reescrituras de compatibilidad después de la fusión
  (por ejemplo, normalizar formas antiguas de flags).

### Ranuras exclusivas

| Método                                     | Qué registra                           |
| ------------------------------------------ | -------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motor de contexto (uno activo a la vez) |
| `api.registerMemoryCapability(capability)` | Capacidad de memoria unificada         |
| `api.registerMemoryPromptSection(builder)` | Constructor de secciones de prompts de memoria |
| `api.registerMemoryFlushPlan(resolver)`    | Resolutor del plan de vaciado de memoria |
| `api.registerMemoryRuntime(runtime)`       | Adaptador de tiempo de ejecución de memoria |

### Adaptadores de embeddings de memoria

| Método                                         | Qué registra                                     |
| ---------------------------------------------- | ------------------------------------------------ |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de embeddings de memoria para el plugin activo |

- `registerMemoryCapability` es la API exclusiva preferida para plugins de memoria.
- `registerMemoryCapability` también puede exponer `publicArtifacts.listArtifacts(...)`
  para que plugins complementarios consuman artefactos de memoria exportados a través de
  `openclaw/plugin-sdk/memory-host-core` en lugar de acceder al diseño privado
  de un plugin de memoria específico.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` y
  `registerMemoryRuntime` son API exclusivas de plugins de memoria compatibles con versiones heredadas.
- `registerMemoryEmbeddingProvider` permite que el plugin de memoria activo registre uno
  o más id de adaptadores de embeddings (por ejemplo `openai`, `gemini` o un id
  personalizado definido por el plugin).
- La configuración del usuario, como `agents.defaults.memorySearch.provider` y
  `agents.defaults.memorySearch.fallback`, se resuelve contra esos id de adaptadores registrados.

### Eventos y ciclo de vida

| Método                                       | Qué hace                     |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook tipado del ciclo de vida |
| `api.onConversationBindingResolved(handler)` | Devolución de llamada de enlace de conversación |

### Semántica de decisión de hooks

- `before_tool_call`: devolver `{ block: true }` es terminal. Una vez que cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `before_tool_call`: devolver `{ block: false }` se trata como ausencia de decisión (igual que omitir `block`), no como una anulación.
- `before_install`: devolver `{ block: true }` es terminal. Una vez que cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `before_install`: devolver `{ block: false }` se trata como ausencia de decisión (igual que omitir `block`), no como una anulación.
- `reply_dispatch`: devolver `{ handled: true, ... }` es terminal. Una vez que cualquier controlador reclama el despacho, se omiten los controladores de menor prioridad y la ruta predeterminada de despacho del modelo.
- `message_sending`: devolver `{ cancel: true }` es terminal. Una vez que cualquier controlador lo establece, se omiten los controladores de menor prioridad.
- `message_sending`: devolver `{ cancel: false }` se trata como ausencia de decisión (igual que omitir `cancel`), no como una anulación.

### Campos del objeto API

| Campo                    | Tipo                      | Descripción                                                                                |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------ |
| `api.id`                 | `string`                  | Id del plugin                                                                              |
| `api.name`               | `string`                  | Nombre visible                                                                             |
| `api.version`            | `string?`                 | Versión del plugin (opcional)                                                              |
| `api.description`        | `string?`                 | Descripción del plugin (opcional)                                                          |
| `api.source`             | `string`                  | Ruta de origen del plugin                                                                  |
| `api.rootDir`            | `string?`                 | Directorio raíz del plugin (opcional)                                                      |
| `api.config`             | `OpenClawConfig`          | Instantánea actual de configuración (instantánea activa en memoria del tiempo de ejecución cuando está disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin desde `plugins.entries.<id>.config`                    |
| `api.runtime`            | `PluginRuntime`           | [Auxiliares de tiempo de ejecución](/es/plugins/sdk-runtime)                                  |
| `api.logger`             | `PluginLogger`            | Logger con alcance (`debug`, `info`, `warn`, `error`)                                      |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de inicio/configuración previa a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resuelve una ruta relativa a la raíz del plugin                                            |

## Convención de módulos internos

Dentro de su plugin, use archivos barril locales para las importaciones internas:

```
my-plugin/
  api.ts            # Exportaciones públicas para consumidores externos
  runtime-api.ts    # Exportaciones internas solo para tiempo de ejecución
  index.ts          # Punto de entrada del plugin
  setup-entry.ts    # Entrada ligera solo para configuración (opcional)
```

<Warning>
  Nunca importe su propio plugin mediante `openclaw/plugin-sdk/<your-plugin>`
  desde código de producción. Encamine las importaciones internas mediante `./api.ts` o
  `./runtime-api.ts`. La ruta del SDK es solo el contrato externo.
</Warning>

Las superficies públicas cargadas mediante fachada de plugins integrados (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` y archivos de entrada pública similares) ahora prefieren la
instantánea de configuración activa del tiempo de ejecución cuando OpenClaw ya está en ejecución. Si todavía no existe una instantánea de tiempo de ejecución, recurren al archivo de configuración resuelto en disco.

Los plugins de proveedor también pueden exponer un barril de contrato local del plugin y de alcance limitado cuando un auxiliar es intencionalmente específico del proveedor y aún no pertenece a una subruta genérica del SDK. Ejemplo integrado actual: el proveedor Anthropic mantiene sus auxiliares de streaming de Claude en su propia capa pública `api.ts` / `contract-api.ts` en lugar de promover la lógica del encabezado beta de Anthropic y `service_tier` a un contrato genérico `plugin-sdk/*`.

Otros ejemplos integrados actuales:

- `@openclaw/openai-provider`: `api.ts` exporta constructores de proveedores,
  auxiliares de modelos predeterminados y constructores de proveedores en tiempo real
- `@openclaw/openrouter-provider`: `api.ts` exporta el constructor del proveedor más
  auxiliares de onboarding/configuración

<Warning>
  El código de producción de extensiones también debe evitar importaciones desde `openclaw/plugin-sdk/<other-plugin>`.
  Si un auxiliar se comparte realmente, promuévalo a una subruta neutral del SDK
  como `openclaw/plugin-sdk/speech`, `.../provider-model-shared` u otra
  superficie orientada a capacidades, en lugar de acoplar dos plugins entre sí.
</Warning>

## Relacionado

- [Puntos de entrada](/es/plugins/sdk-entrypoints) — opciones de `definePluginEntry` y `defineChannelPluginEntry`
- [Auxiliares de tiempo de ejecución](/es/plugins/sdk-runtime) — referencia completa del espacio de nombres `api.runtime`
- [Configuración y config](/es/plugins/sdk-setup) — empaquetado, manifiestos y esquemas de configuración
- [Pruebas](/es/plugins/sdk-testing) — utilidades de prueba y reglas de lint
- [Migración del SDK](/es/plugins/sdk-migration) — migración desde superficies obsoletas
- [Internos de plugins](/es/plugins/architecture) — arquitectura profunda y modelo de capacidades
