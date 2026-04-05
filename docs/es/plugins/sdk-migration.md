---
read_when:
    - Ves la advertencia OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ves la advertencia OPENCLAW_EXTENSION_API_DEPRECATED
    - Estás actualizando un plugin a la arquitectura moderna de plugins de OpenClaw
    - Mantienes un plugin externo de OpenClaw
sidebarTitle: Migrate to SDK
summary: Migra de la capa heredada de compatibilidad retroactiva al moderno plugin SDK
title: Migración del Plugin SDK
x-i18n:
    generated_at: "2026-04-05T12:50:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: c420b8d7de17aee16c5aa67e3a88da5750f0d84b07dd541f061081080e081196
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migración del Plugin SDK

OpenClaw ha pasado de una capa amplia de compatibilidad retroactiva a una arquitectura moderna de plugins
con importaciones enfocadas y documentadas. Si tu plugin se creó antes de
la nueva arquitectura, esta guía te ayuda a migrarlo.

## Qué está cambiando

El antiguo sistema de plugins proporcionaba dos superficies muy abiertas que permitían a los plugins importar
todo lo que necesitaran desde un único punto de entrada:

- **`openclaw/plugin-sdk/compat`** — una única importación que reexportaba decenas de
  utilidades. Se introdujo para mantener funcionando los plugins antiguos basados en hooks mientras se
  construía la nueva arquitectura de plugins.
- **`openclaw/extension-api`** — un puente que daba a los plugins acceso directo a
  utilidades del lado del host, como el ejecutor integrado del agente.

Ambas superficies ahora están **obsoletas**. Siguen funcionando en tiempo de ejecución, pero los
plugins nuevos no deben usarlas, y los plugins existentes deben migrar antes de que la siguiente
versión principal las elimine.

<Warning>
  La capa de compatibilidad retroactiva se eliminará en una futura versión principal.
  Los plugins que sigan importando desde estas superficies dejarán de funcionar cuando eso ocurra.
</Warning>

## Por qué cambió esto

El enfoque anterior causaba problemas:

- **Inicio lento** — importar una utilidad cargaba decenas de módulos no relacionados
- **Dependencias circulares** — las reexportaciones amplias facilitaban la creación de ciclos de importación
- **Superficie de API poco clara** — no había forma de saber qué exportaciones eran estables y cuáles eran internas

El moderno plugin SDK corrige esto: cada ruta de importación (`openclaw/plugin-sdk/\<subpath\>`)
es un módulo pequeño, autocontenido, con un propósito claro y un contrato documentado.

Las costuras heredadas de conveniencia para proveedores en canales integrados también han desaparecido. Las importaciones
como `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
las costuras de utilidades con marca de canal y
`openclaw/plugin-sdk/telegram-core` eran atajos privados del monorepo, no
contratos estables de plugin. Usa en su lugar subrutas genéricas y estrechas del SDK. Dentro del
espacio de trabajo de plugins integrados, mantén las utilidades propias del proveedor en el
propio `api.ts` o `runtime-api.ts` de ese plugin.

Ejemplos actuales de proveedores integrados:

- Anthropic mantiene las utilidades específicas de streams de Claude en su propia costura `api.ts` /
  `contract-api.ts`
- OpenAI mantiene los builders del proveedor, utilidades del modelo predeterminado y builders
  del proveedor realtime en su propio `api.ts`
- OpenRouter mantiene el builder del proveedor y las utilidades de onboarding/configuración en su
  propio `api.ts`

## Cómo migrar

<Steps>
  <Step title="Auditar el comportamiento de respaldo del wrapper de Windows">
    Si tu plugin usa `openclaw/plugin-sdk/windows-spawn`, los wrappers de Windows
    `.cmd`/`.bat` no resueltos ahora fallan en modo cerrado a menos que pases explícitamente
    `allowShellFallback: true`.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    Si tu llamador no depende intencionadamente del respaldo mediante shell, no configures
    `allowShellFallback` y maneja en su lugar el error lanzado.

  </Step>

  <Step title="Encontrar importaciones obsoletas">
    Busca en tu plugin importaciones desde cualquiera de las dos superficies obsoletas:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Reemplazar por importaciones enfocadas">
    Cada exportación de la superficie antigua se asigna a una ruta de importación moderna específica:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Para utilidades del lado del host, usa el runtime inyectado del plugin en lugar de importar
    directamente:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    El mismo patrón se aplica a otras utilidades heredadas del puente:

    | Old import | Modern equivalent |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | session store helpers | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Compilar y probar">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Referencia de rutas de importación

<Accordion title="Tabla común de rutas de importación">
  | Import path | Purpose | Key exports |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Utilidad canónica de entrada de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Reexportación paraguas heredada para definiciones/builders de entradas de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Exportación del esquema de configuración raíz | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Utilidad de entrada de proveedor único | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definiciones y builders enfocados de entradas de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Utilidades compartidas del asistente de configuración | Prompts de allowlist, builders de estado de configuración |
  | `plugin-sdk/setup-runtime` | Utilidades del runtime en tiempo de configuración | Adaptadores de patch de configuración seguros para importación, utilidades de notas de lookup, `promptResolvedAllowFrom`, `splitSetupEntries`, proxies de configuración delegados |
  | `plugin-sdk/setup-adapter-runtime` | Utilidades del adaptador de configuración | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Utilidades de herramientas de configuración | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Utilidades para múltiples cuentas | Utilidades de lista de cuentas/configuración/puertas de acción |
  | `plugin-sdk/account-id` | Utilidades de ID de cuenta | `DEFAULT_ACCOUNT_ID`, normalización de ID de cuenta |
  | `plugin-sdk/account-resolution` | Utilidades de búsqueda de cuentas | Utilidades de búsqueda de cuentas + respaldo predeterminado |
  | `plugin-sdk/account-helpers` | Utilidades estrechas de cuenta | Utilidades de lista de cuentas/acciones de cuenta |
  | `plugin-sdk/channel-setup` | Adaptadores del asistente de configuración | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, además de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitivas de pairing de MD | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Cableado de prefijo de respuesta + escritura | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fábricas de adaptadores de configuración | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builders de esquema de configuración | Tipos de esquema de configuración de canal |
  | `plugin-sdk/telegram-command-config` | Utilidades de configuración de comandos de Telegram | Normalización de nombres de comandos, recorte de descripciones, validación de duplicados/conflictos |
  | `plugin-sdk/channel-policy` | Resolución de políticas de grupo/MD | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Seguimiento de estado de cuentas | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Utilidades de envoltura entrante | Utilidades compartidas de rutas + builders de envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Utilidades de respuestas entrantes | Utilidades compartidas de registro y despacho |
  | `plugin-sdk/messaging-targets` | Análisis de destinos de mensajería | Utilidades de análisis/coincidencia de destinos |
  | `plugin-sdk/outbound-media` | Utilidades de medios salientes | Carga compartida de medios salientes |
  | `plugin-sdk/outbound-runtime` | Utilidades del runtime saliente | Utilidades de identidad saliente/delegado de envío |
  | `plugin-sdk/thread-bindings-runtime` | Utilidades de enlaces de hilos | Ciclo de vida de enlaces de hilos y utilidades del adaptador |
  | `plugin-sdk/agent-media-payload` | Utilidades heredadas de carga de medios | Builder de carga de medios del agente para diseños heredados de campos |
  | `plugin-sdk/channel-runtime` | Shim de compatibilidad obsoleto | Solo utilidades heredadas del runtime de canal |
  | `plugin-sdk/channel-send-result` | Tipos de resultado de envío | Tipos de resultado de respuesta |
  | `plugin-sdk/runtime-store` | Almacenamiento persistente de plugins | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Utilidades amplias del runtime | Utilidades de runtime/logging/backup/instalación de plugins |
  | `plugin-sdk/runtime-env` | Utilidades estrechas de entorno del runtime | Logger/entorno del runtime, utilidades de timeout, retry y backoff |
  | `plugin-sdk/plugin-runtime` | Utilidades compartidas del runtime de plugins | Utilidades compartidas de comandos/hooks/http/interactivo de plugins |
  | `plugin-sdk/hook-runtime` | Utilidades del pipeline de hooks | Utilidades compartidas de webhook/hooks internos |
  | `plugin-sdk/lazy-runtime` | Utilidades de runtime diferido | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Utilidades de procesos | Utilidades compartidas de exec |
  | `plugin-sdk/cli-runtime` | Utilidades del runtime de CLI | Formateo de comandos, esperas, utilidades de versión |
  | `plugin-sdk/gateway-runtime` | Utilidades de gateway | Cliente de gateway y utilidades de patch de estado de canal |
  | `plugin-sdk/config-runtime` | Utilidades de configuración | Utilidades de carga/escritura de configuración |
  | `plugin-sdk/telegram-command-config` | Utilidades de comandos de Telegram | Utilidades estables de respaldo para validación de comandos de Telegram cuando la superficie contractual integrada de Telegram no está disponible |
  | `plugin-sdk/approval-runtime` | Utilidades de prompts de aprobación | Payload de aprobación exec/plugin, utilidades de capacidad/perfil de aprobación, utilidades nativas de enrutamiento/runtime de aprobación |
  | `plugin-sdk/approval-auth-runtime` | Utilidades de autenticación de aprobación | Resolución de aprobadores, autenticación de acciones en el mismo chat |
  | `plugin-sdk/approval-client-runtime` | Utilidades de cliente de aprobación | Utilidades nativas de perfil/filtro de aprobación de exec |
  | `plugin-sdk/approval-delivery-runtime` | Utilidades de entrega de aprobación | Adaptadores nativos de capacidad/entrega de aprobación |
  | `plugin-sdk/approval-native-runtime` | Utilidades de destino de aprobación | Utilidades nativas de destino de aprobación/enlace de cuenta |
  | `plugin-sdk/approval-reply-runtime` | Utilidades de respuesta de aprobación | Utilidades de payload de respuesta de aprobación exec/plugin |
  | `plugin-sdk/security-runtime` | Utilidades de seguridad | Utilidades compartidas de confianza, DM gating, contenido externo y recopilación de secretos |
  | `plugin-sdk/ssrf-policy` | Utilidades de política SSRF | Utilidades de allowlist de hosts y política de red privada |
  | `plugin-sdk/ssrf-runtime` | Utilidades SSRF del runtime | Dispatcher fijado, fetch protegido, utilidades de política SSRF |
  | `plugin-sdk/collection-runtime` | Utilidades de caché acotada | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Utilidades de compuertas de diagnóstico | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Utilidades de formateo de errores | `formatUncaughtError`, `isApprovalNotFoundError`, utilidades de grafo de errores |
  | `plugin-sdk/fetch-runtime` | Utilidades de fetch/proxy encapsuladas | `resolveFetch`, utilidades de proxy |
  | `plugin-sdk/host-runtime` | Utilidades de normalización de host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Utilidades de retry | `RetryConfig`, `retryAsync`, ejecutores de políticas |
  | `plugin-sdk/allow-from` | Formateo de allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapeo de entradas de allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Compuertas de comandos y utilidades de superficie de comandos | `resolveControlCommandGate`, utilidades de autorización de remitentes, utilidades de registro de comandos |
  | `plugin-sdk/secret-input` | Análisis de entradas secretas | Utilidades de entrada secreta |
  | `plugin-sdk/webhook-ingress` | Utilidades de solicitudes webhook | Utilidades de destino webhook |
  | `plugin-sdk/webhook-request-guards` | Utilidades de guardas de cuerpo webhook | Utilidades de lectura/límite del cuerpo de la solicitud |
  | `plugin-sdk/reply-runtime` | Runtime compartido de respuesta | Despacho entrante, heartbeat, planificador de respuestas, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Utilidades estrechas de despacho de respuesta | Utilidades de finalización + despacho del proveedor |
  | `plugin-sdk/reply-history` | Utilidades del historial de respuestas | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planificación de referencias de respuesta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Utilidades de fragmentación de respuestas | Utilidades de fragmentación de texto/markdown |
  | `plugin-sdk/session-store-runtime` | Utilidades del almacén de sesiones | Utilidades de ruta del almacén + updated-at |
  | `plugin-sdk/state-paths` | Utilidades de rutas de estado | Utilidades de directorio de estado y OAuth |
  | `plugin-sdk/routing` | Utilidades de enrutamiento/clave de sesión | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, utilidades de normalización de claves de sesión |
  | `plugin-sdk/status-helpers` | Utilidades de estado de canal | Builders de resumen/instantánea de estado de canal/cuenta, valores predeterminados del estado del runtime, utilidades de metadatos de incidencias |
  | `plugin-sdk/target-resolver-runtime` | Utilidades de resolución de destino | Utilidades compartidas de resolución de destino |
  | `plugin-sdk/string-normalization-runtime` | Utilidades de normalización de cadenas | Utilidades de normalización de slug/cadenas |
  | `plugin-sdk/request-url` | Utilidades de URL de solicitud | Extraer URL de cadena de entradas tipo request |
  | `plugin-sdk/run-command` | Utilidades de comando temporizado | Ejecutor de comandos temporizado con stdout/stderr normalizados |
  | `plugin-sdk/param-readers` | Lectores de parámetros | Lectores comunes de parámetros de herramientas/CLI |
  | `plugin-sdk/tool-send` | Extracción de envíos de herramientas | Extraer campos canónicos de destino de envío de args de herramientas |
  | `plugin-sdk/temp-path` | Utilidades de rutas temporales | Utilidades compartidas de rutas temporales de descarga |
  | `plugin-sdk/logging-core` | Utilidades de logging | Logger de subsistema y utilidades de redacción |
  | `plugin-sdk/markdown-table-runtime` | Utilidades de tablas Markdown | Utilidades de modo de tablas Markdown |
  | `plugin-sdk/reply-payload` | Tipos de respuesta de mensajes | Tipos de payload de respuesta |
  | `plugin-sdk/provider-setup` | Utilidades de configuración curadas para proveedores locales/autohospedados | Utilidades de descubrimiento/configuración de proveedores autohospedados |
  | `plugin-sdk/self-hosted-provider-setup` | Utilidades enfocadas de configuración de proveedores autohospedados compatibles con OpenAI | Las mismas utilidades de descubrimiento/configuración de proveedores autohospedados |
  | `plugin-sdk/provider-auth-runtime` | Utilidades de autenticación de proveedores en runtime | Utilidades de resolución de clave API en runtime |
  | `plugin-sdk/provider-auth-api-key` | Utilidades de configuración de clave API de proveedores | Utilidades de onboarding/escritura de perfiles de clave API |
  | `plugin-sdk/provider-auth-result` | Utilidades de resultados de autenticación de proveedor | Builder estándar de resultados de autenticación OAuth |
  | `plugin-sdk/provider-auth-login` | Utilidades de inicio de sesión interactivo de proveedores | Utilidades compartidas de inicio de sesión interactivo |
  | `plugin-sdk/provider-env-vars` | Utilidades de variables de entorno de proveedor | Utilidades de lookup de variables de entorno de autenticación de proveedor |
  | `plugin-sdk/provider-model-shared` | Utilidades compartidas de modelo/replay de proveedor | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builders compartidos de políticas de replay, utilidades de endpoint de proveedor y utilidades de normalización de ID de modelo |
  | `plugin-sdk/provider-catalog-shared` | Utilidades compartidas del catálogo de proveedores | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Parches de onboarding de proveedor | Utilidades de configuración de onboarding |
  | `plugin-sdk/provider-http` | Utilidades HTTP de proveedor | Utilidades genéricas de HTTP/capacidades de endpoint del proveedor |
  | `plugin-sdk/provider-web-fetch` | Utilidades web-fetch de proveedor | Utilidades de registro/caché de proveedor web-fetch |
  | `plugin-sdk/provider-web-search` | Utilidades web-search de proveedor | Utilidades de registro/caché/configuración de proveedor web-search |
  | `plugin-sdk/provider-tools` | Utilidades de compatibilidad de herramientas/esquemas de proveedor | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza de esquemas Gemini + diagnósticos, y utilidades de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Utilidades de uso de proveedores | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` y otras utilidades de uso de proveedores |
  | `plugin-sdk/provider-stream` | Utilidades de envoltura de streams de proveedor | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de envolturas de stream y utilidades compartidas de envolturas para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Cola asíncrona ordenada | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Utilidades compartidas de medios | Utilidades de fetch/transform/store de medios más builders de payload de medios |
  | `plugin-sdk/media-understanding` | Utilidades de entendimiento de medios | Tipos de proveedores de entendimiento de medios más exportaciones de utilidades de imagen/audio de cara al proveedor |
  | `plugin-sdk/text-runtime` | Utilidades compartidas de texto | Eliminación de texto visible para el asistente, renderizado/chunking/tablas Markdown, utilidades de redacción, utilidades de etiquetas de directivas, utilidades de texto seguro y otras utilidades relacionadas de texto/logging |
  | `plugin-sdk/text-chunking` | Utilidades de fragmentación de texto | Utilidad de fragmentación de texto saliente |
  | `plugin-sdk/speech` | Utilidades de voz | Tipos de proveedores de voz más exportaciones de utilidades de directivas, registro y validación de cara al proveedor |
  | `plugin-sdk/speech-core` | Núcleo compartido de voz | Tipos de proveedores de voz, registro, directivas, normalización |
  | `plugin-sdk/realtime-transcription` | Utilidades de transcripción en tiempo real | Tipos de proveedores y utilidades de registro |
  | `plugin-sdk/realtime-voice` | Utilidades de voz en tiempo real | Tipos de proveedores y utilidades de registro |
  | `plugin-sdk/image-generation-core` | Núcleo compartido de generación de imágenes | Tipos, failover, autenticación y utilidades de registro para generación de imágenes |
  | `plugin-sdk/video-generation` | Utilidades de generación de vídeo | Tipos de proveedor/solicitud/resultado de generación de vídeo |
  | `plugin-sdk/video-generation-core` | Núcleo compartido de generación de vídeo | Tipos de generación de vídeo, utilidades de failover, lookup de proveedor y análisis de refs de modelo |
  | `plugin-sdk/interactive-runtime` | Utilidades de respuesta interactiva | Normalización/reducción de payload de respuesta interactiva |
  | `plugin-sdk/channel-config-primitives` | Primitivas de configuración de canal | Primitivas estrechas de esquema de configuración de canal |
  | `plugin-sdk/channel-config-writes` | Utilidades de escritura de configuración de canal | Utilidades de autorización de escritura de configuración de canal |
  | `plugin-sdk/channel-plugin-common` | Preludio compartido de canal | Exportaciones compartidas de preludio de plugin de canal |
  | `plugin-sdk/channel-status` | Utilidades de estado de canal | Utilidades compartidas de instantánea/resumen de estado de canal |
  | `plugin-sdk/allowlist-config-edit` | Utilidades de configuración de allowlist | Utilidades de edición/lectura de configuración de allowlist |
  | `plugin-sdk/group-access` | Utilidades de acceso a grupos | Utilidades compartidas de decisiones de acceso a grupos |
  | `plugin-sdk/direct-dm` | Utilidades de MD directos | Utilidades compartidas de autenticación/protección de MD directos |
  | `plugin-sdk/extension-shared` | Utilidades compartidas de extensión | Primitivas de utilidades pasivas de canal/estado |
  | `plugin-sdk/webhook-targets` | Utilidades de destinos webhook | Registro de destinos webhook y utilidades de instalación de rutas |
  | `plugin-sdk/webhook-path` | Utilidades de rutas webhook | Utilidades de normalización de rutas webhook |
  | `plugin-sdk/web-media` | Utilidades compartidas de medios web | Utilidades de carga de medios remotos/locales |
  | `plugin-sdk/zod` | Reexportación de zod | `zod` reexportado para consumidores del plugin SDK |
  | `plugin-sdk/memory-core` | Utilidades integradas de memory-core | Superficie de utilidades de gestor/configuración/archivo/CLI de memoria |
  | `plugin-sdk/memory-core-engine-runtime` | Fachada del runtime del motor de memoria | Fachada del runtime de índice/búsqueda de memoria |
  | `plugin-sdk/memory-core-host-engine-foundation` | Motor foundation del host de memoria | Exportaciones del motor foundation del host de memoria |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Motor de embeddings del host de memoria | Exportaciones del motor de embeddings del host de memoria |
  | `plugin-sdk/memory-core-host-engine-qmd` | Motor QMD del host de memoria | Exportaciones del motor QMD del host de memoria |
  | `plugin-sdk/memory-core-host-engine-storage` | Motor de almacenamiento del host de memoria | Exportaciones del motor de almacenamiento del host de memoria |
  | `plugin-sdk/memory-core-host-multimodal` | Utilidades multimodales del host de memoria | Utilidades multimodales del host de memoria |
  | `plugin-sdk/memory-core-host-query` | Utilidades de consulta del host de memoria | Utilidades de consulta del host de memoria |
  | `plugin-sdk/memory-core-host-secret` | Utilidades de secretos del host de memoria | Utilidades de secretos del host de memoria |
  | `plugin-sdk/memory-core-host-status` | Utilidades de estado del host de memoria | Utilidades de estado del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI del host de memoria | Utilidades del runtime CLI del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime central del host de memoria | Utilidades del runtime central del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-files` | Utilidades de archivos/runtime del host de memoria | Utilidades de archivos/runtime del host de memoria |
  | `plugin-sdk/memory-lancedb` | Utilidades integradas de memory-lancedb | Superficie de utilidades de memory-lancedb |
  | `plugin-sdk/testing` | Utilidades de prueba | Utilidades de prueba y mocks |
</Accordion>

Esta tabla es intencionalmente el subconjunto común de migración, no la superficie completa del SDK.
La lista completa de más de 200 entrypoints está en
`scripts/lib/plugin-sdk-entrypoints.json`.

Esa lista aún incluye algunas costuras de utilidades de plugins integrados como
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Siguen exportadas para
mantenimiento de plugins integrados y compatibilidad, pero se omiten intencionalmente
de la tabla común de migración y no son el destino recomendado para
código nuevo de plugins.

La misma regla se aplica a otras familias de utilidades integradas como:

- utilidades de soporte del browser: `plugin-sdk/browser-config-support`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superficies de utilidades/plugins integrados como `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` y `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` actualmente expone la estrecha
superficie de utilidades de token `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken`.

Usa la importación más estrecha que coincida con la tarea. Si no encuentras una exportación,
consulta el código fuente en `src/plugin-sdk/` o pregunta en Discord.

## Cronograma de eliminación

| When                   | What happens                                                            |
| ---------------------- | ----------------------------------------------------------------------- |
| **Ahora**                | Las superficies obsoletas emiten advertencias en tiempo de ejecución                               |
| **Próxima versión principal** | Las superficies obsoletas se eliminarán; los plugins que sigan usándolas dejarán de funcionar |

Todos los plugins del núcleo ya se han migrado. Los plugins externos deben migrar
antes de la próxima versión principal.

## Suprimir temporalmente las advertencias

Configura estas variables de entorno mientras trabajas en la migración:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Esto es una vía de escape temporal, no una solución permanente.

## Relacionado

- [Primeros pasos](/plugins/building-plugins) — crea tu primer plugin
- [Resumen del SDK](/plugins/sdk-overview) — referencia completa de importaciones por subruta
- [Plugins de canal](/plugins/sdk-channel-plugins) — crear plugins de canal
- [Plugins de proveedor](/plugins/sdk-provider-plugins) — crear plugins de proveedor
- [Aspectos internos de plugins](/plugins/architecture) — análisis profundo de la arquitectura
- [Manifiesto de plugin](/plugins/manifest) — referencia del esquema del manifiesto
