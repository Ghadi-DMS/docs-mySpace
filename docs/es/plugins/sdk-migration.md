---
read_when:
    - Ves la advertencia OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ves la advertencia OPENCLAW_EXTENSION_API_DEPRECATED
    - Estás actualizando un plugin a la arquitectura moderna de plugins
    - Mantienes un plugin externo de OpenClaw
sidebarTitle: Migrate to SDK
summary: Migra desde la capa heredada de compatibilidad hacia atrás al moderno plugin SDK
title: Migración del Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:17:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 155a8b14bc345319c8516ebdb8a0ccdea2c5f7fa07dad343442996daee21ecad
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migración del Plugin SDK

OpenClaw ha pasado de una amplia capa de compatibilidad hacia atrás a una arquitectura
moderna de plugins con importaciones enfocadas y documentadas. Si tu plugin se creó antes
de la nueva arquitectura, esta guía te ayuda a migrarlo.

## Qué está cambiando

El sistema anterior de plugins proporcionaba dos superficies muy abiertas que permitían a los plugins importar
cualquier cosa que necesitaran desde un único punto de entrada:

- **`openclaw/plugin-sdk/compat`** — una sola importación que reexportaba decenas de
  helpers. Se introdujo para mantener en funcionamiento plugins más antiguos basados en hooks mientras se
  construía la nueva arquitectura de plugins.
- **`openclaw/extension-api`** — un puente que daba a los plugins acceso directo a
  helpers del lado del host, como el ejecutor de agente embebido.

Ambas superficies ahora están **obsoletas**. Siguen funcionando en tiempo de ejecución, pero los
plugins nuevos no deben usarlas, y los plugins existentes deberían migrar antes de que la próxima
versión principal las elimine.

<Warning>
  La capa de compatibilidad hacia atrás se eliminará en una futura versión principal.
  Los plugins que sigan importando desde estas superficies dejarán de funcionar cuando eso ocurra.
</Warning>

## Por qué cambió esto

El enfoque anterior causaba problemas:

- **Inicio lento** — importar un helper cargaba decenas de módulos no relacionados
- **Dependencias circulares** — las reexportaciones amplias facilitaban la creación de ciclos de importación
- **Superficie API poco clara** — no había forma de saber qué exportaciones eran estables y cuáles eran internas

El moderno plugin SDK soluciona esto: cada ruta de importación (`openclaw/plugin-sdk/\<subpath\>`)
es un módulo pequeño, autocontenido, con un propósito claro y un contrato documentado.

Las superficies heredadas de conveniencia para proveedores de canales empaquetados también han desaparecido. Las importaciones
como `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
las superficies helper con marca de canal y
`openclaw/plugin-sdk/telegram-core` eran atajos privados del monorepo, no
contratos estables para plugins. Usa en su lugar subrutas genéricas y específicas del SDK. Dentro del
espacio de trabajo de plugins empaquetados, mantén los helpers propiedad del proveedor en el propio
`api.ts` o `runtime-api.ts` de ese plugin.

Ejemplos actuales de proveedores empaquetados:

- Anthropic mantiene helpers de flujo específicos de Claude en su propia superficie `api.ts` /
  `contract-api.ts`
- OpenAI mantiene constructores de proveedores, helpers de modelos predeterminados y constructores de proveedores
  realtime en su propio `api.ts`
- OpenRouter mantiene el constructor del proveedor y los helpers de onboarding/configuración en su propio
  `api.ts`

## Cómo migrar

<Steps>
  <Step title="Migra los handlers nativos de aprobación a hechos de capacidad">
    Los plugins de canal con capacidad de aprobación ahora exponen el comportamiento nativo de aprobación mediante
    `approvalCapability.nativeRuntime` más el registro compartido de contexto de runtime.

    Cambios clave:

    - Reemplaza `approvalCapability.handler.loadRuntime(...)` por
      `approvalCapability.nativeRuntime`
    - Mueve la autenticación/entrega específica de aprobación fuera del cableado heredado `plugin.auth` /
      `plugin.approvals` y colócalo en `approvalCapability`
    - `ChannelPlugin.approvals` se ha eliminado del contrato público
      de plugins de canal; mueve los campos delivery/native/render a `approvalCapability`
    - `plugin.auth` se mantiene solo para flujos de login/logout del canal; los hooks de autenticación
      de aprobación allí ya no son leídos por el núcleo
    - Registra objetos de runtime propiedad del canal, como clientes, tokens o apps Bolt,
      mediante `openclaw/plugin-sdk/channel-runtime-context`
    - No envíes avisos de redirección propiedad del plugin desde handlers nativos de aprobación;
      el núcleo ahora se encarga de los avisos de enrutado a otro lugar a partir de resultados reales de entrega
    - Al pasar `channelRuntime` a `createChannelManager(...)`, proporciona una
      superficie real `createPluginRuntime().channel`. Los stubs parciales se rechazan.

    Consulta `/plugins/sdk-channel-plugins` para ver el diseño actual de
    capacidades de aprobación.

  </Step>

  <Step title="Audita el comportamiento de respaldo del wrapper de Windows">
    Si tu plugin usa `openclaw/plugin-sdk/windows-spawn`, los wrappers `.cmd`/`.bat`
    no resueltos de Windows ahora fallan en modo cerrado, a menos que pases explícitamente
    `allowShellFallback: true`.

    ```typescript
    // Antes
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Después
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Establece esto solo para llamantes de compatibilidad de confianza que
      // aceptan intencionalmente el respaldo mediado por shell.
      allowShellFallback: true,
    });
    ```

    Si tu llamante no depende intencionalmente del respaldo por shell, no establezcas
    `allowShellFallback` y maneja el error lanzado en su lugar.

  </Step>

  <Step title="Encuentra importaciones obsoletas">
    Busca en tu plugin importaciones desde cualquiera de estas dos superficies obsoletas:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Reemplázalas por importaciones enfocadas">
    Cada exportación de la superficie antigua se corresponde con una ruta de importación moderna específica:

    ```typescript
    // Antes (capa obsoleta de compatibilidad hacia atrás)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Después (importaciones modernas y enfocadas)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Para helpers del lado del host, usa el runtime de plugin inyectado en lugar de importar
    directamente:

    ```typescript
    // Antes (puente extension-api obsoleto)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Después (runtime inyectado)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    El mismo patrón se aplica a otros helpers heredados del puente:

    | Importación antigua | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helpers del almacén de sesiones | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Compila y prueba">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Referencia de rutas de importación

<Accordion title="Tabla común de rutas de importación">
  | Ruta de importación | Propósito | Exportaciones clave |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper canónico de entrada de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Reexportación paraguas heredada para definiciones/constructores de entrada de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Exportación del esquema raíz de configuración | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper de entrada de proveedor único | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definiciones y constructores enfocados de entrada de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helpers compartidos del asistente de configuración | Prompts de listas de permitidos, constructores de estado de configuración |
  | `plugin-sdk/setup-runtime` | Helpers de runtime en tiempo de configuración | Adaptadores de parches de configuración seguros para importación, helpers de notas de búsqueda, `promptResolvedAllowFrom`, `splitSetupEntries`, proxies delegados de configuración |
  | `plugin-sdk/setup-adapter-runtime` | Helpers del adaptador de configuración | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helpers de herramientas de configuración | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helpers de múltiples cuentas | Helpers de lista/configuración/compuertas de acciones de cuenta |
  | `plugin-sdk/account-id` | Helpers de id de cuenta | `DEFAULT_ACCOUNT_ID`, normalización de id de cuenta |
  | `plugin-sdk/account-resolution` | Helpers de búsqueda de cuenta | Helpers de búsqueda de cuenta + respaldo predeterminado |
  | `plugin-sdk/account-helpers` | Helpers de cuenta específicos | Helpers de lista de cuentas/acción de cuenta |
  | `plugin-sdk/channel-setup` | Adaptadores del asistente de configuración | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, además de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitivas de emparejamiento de DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Cableado de prefijo de respuesta + escritura | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fábricas de adaptadores de configuración | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Constructores de esquema de configuración | Tipos de esquema de configuración de canal |
  | `plugin-sdk/telegram-command-config` | Helpers de configuración de comandos de Telegram | Normalización de nombres de comando, recorte de descripciones, validación de duplicados/conflictos |
  | `plugin-sdk/channel-policy` | Resolución de políticas de grupo/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Seguimiento del estado de cuenta | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helpers de envoltorio entrante | Helpers compartidos de ruta + construcción de envoltorio |
  | `plugin-sdk/inbound-reply-dispatch` | Helpers de respuesta entrante | Helpers compartidos de registro y despacho |
  | `plugin-sdk/messaging-targets` | Análisis de objetivos de mensajería | Helpers de análisis/coincidencia de objetivos |
  | `plugin-sdk/outbound-media` | Helpers de medios salientes | Carga compartida de medios salientes |
  | `plugin-sdk/outbound-runtime` | Helpers de runtime saliente | Helpers delegados de identidad/envío saliente |
  | `plugin-sdk/thread-bindings-runtime` | Helpers de bindings de hilo | Ciclo de vida de bindings de hilo y helpers de adaptador |
  | `plugin-sdk/agent-media-payload` | Helpers heredados de carga útil de medios | Constructor de carga útil de medios de agente para diseños heredados de campos |
  | `plugin-sdk/channel-runtime` | Shim obsoleto de compatibilidad | Solo utilidades heredadas de runtime de canal |
  | `plugin-sdk/channel-send-result` | Tipos de resultado de envío | Tipos de resultado de respuesta |
  | `plugin-sdk/runtime-store` | Almacenamiento persistente del plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helpers amplios de runtime | Helpers de runtime/logging/backup/instalación de plugins |
  | `plugin-sdk/runtime-env` | Helpers específicos de entorno de runtime | Logger/entorno de runtime, timeout, retry y helpers de backoff |
  | `plugin-sdk/plugin-runtime` | Helpers compartidos de runtime de plugin | Helpers de comandos/hooks/http/interactivos de plugin |
  | `plugin-sdk/hook-runtime` | Helpers de pipeline de hooks | Helpers compartidos del pipeline de hooks webhook/internos |
  | `plugin-sdk/lazy-runtime` | Helpers de runtime diferido | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helpers de proceso | Helpers compartidos de exec |
  | `plugin-sdk/cli-runtime` | Helpers de runtime de CLI | Formato de comandos, esperas, helpers de versión |
  | `plugin-sdk/gateway-runtime` | Helpers del Gateway | Cliente Gateway y helpers de parcheo de estado de canal |
  | `plugin-sdk/config-runtime` | Helpers de configuración | Helpers de carga/escritura de configuración |
  | `plugin-sdk/telegram-command-config` | Helpers de comandos de Telegram | Helpers de validación de comandos de Telegram estables como respaldo cuando la superficie de contrato empaquetada de Telegram no está disponible |
  | `plugin-sdk/approval-runtime` | Helpers de prompts de aprobación | Carga útil de aprobación de exec/plugin, helpers de capacidad/perfil de aprobación, helpers nativos de routing/runtime de aprobación |
  | `plugin-sdk/approval-auth-runtime` | Helpers de auth de aprobación | Resolución del aprobador, auth de acción en el mismo chat |
  | `plugin-sdk/approval-client-runtime` | Helpers de cliente de aprobación | Helpers de perfil/filtro nativos de aprobación de exec |
  | `plugin-sdk/approval-delivery-runtime` | Helpers de entrega de aprobación | Adaptadores nativos de capacidad/entrega de aprobación |
  | `plugin-sdk/approval-gateway-runtime` | Helpers de Gateway de aprobación | Helper compartido de resolución de gateway de aprobación |
  | `plugin-sdk/approval-handler-adapter-runtime` | Helpers de adaptador de aprobación | Helpers ligeros de carga de adaptadores nativos de aprobación para puntos de entrada de canal activos |
  | `plugin-sdk/approval-handler-runtime` | Helpers de handler de aprobación | Helpers más amplios de runtime de handlers de aprobación; prefiere las superficies más específicas de adaptador/gateway cuando sean suficientes |
  | `plugin-sdk/approval-native-runtime` | Helpers de objetivos de aprobación | Helpers nativos de binding de objetivo/cuenta de aprobación |
  | `plugin-sdk/approval-reply-runtime` | Helpers de respuesta de aprobación | Helpers de carga útil de respuesta de aprobación de exec/plugin |
  | `plugin-sdk/channel-runtime-context` | Helpers de contexto de runtime de canal | Helpers genéricos de registro/obtención/observación del contexto de runtime de canal |
  | `plugin-sdk/security-runtime` | Helpers de seguridad | Helpers compartidos de confianza, restricción de DM, contenido externo y recopilación de secretos |
  | `plugin-sdk/ssrf-policy` | Helpers de política SSRF | Helpers de lista de permitidos de hosts y política de red privada |
  | `plugin-sdk/ssrf-runtime` | Helpers de runtime SSRF | Helpers de dispatcher fijado, fetch protegido y política SSRF |
  | `plugin-sdk/collection-runtime` | Helpers de caché limitada | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helpers de restricción diagnóstica | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helpers de formato de errores | `formatUncaughtError`, `isApprovalNotFoundError`, helpers de grafo de errores |
  | `plugin-sdk/fetch-runtime` | Helpers envueltos de fetch/proxy | `resolveFetch`, helpers de proxy |
  | `plugin-sdk/host-runtime` | Helpers de normalización del host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helpers de retry | `RetryConfig`, `retryAsync`, ejecutores de políticas |
  | `plugin-sdk/allow-from` | Formateo de lista de permitidos | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapeo de entradas de lista de permitidos | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Restricción de comandos y helpers de superficie de comandos | `resolveControlCommandGate`, helpers de autorización de remitente, helpers de registro de comandos |
  | `plugin-sdk/secret-input` | Análisis de entrada secreta | Helpers de entrada secreta |
  | `plugin-sdk/webhook-ingress` | Helpers de solicitud webhook | Utilidades de objetivo webhook |
  | `plugin-sdk/webhook-request-guards` | Helpers de guardas de cuerpo webhook | Helpers de lectura/límite del cuerpo de solicitud |
  | `plugin-sdk/reply-runtime` | Runtime compartido de respuesta | Despacho entrante, heartbeat, planificador de respuesta, fragmentación |
  | `plugin-sdk/reply-dispatch-runtime` | Helpers específicos de despacho de respuesta | Helpers de finalización + despacho de proveedor |
  | `plugin-sdk/reply-history` | Helpers de historial de respuesta | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planificación de referencias de respuesta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helpers de fragmentación de respuesta | Helpers de fragmentación de texto/markdown |
  | `plugin-sdk/session-store-runtime` | Helpers de almacén de sesiones | Ruta del almacén + helpers de updated-at |
  | `plugin-sdk/state-paths` | Helpers de rutas de estado | Helpers de directorios de estado y OAuth |
  | `plugin-sdk/routing` | Helpers de routing/clave de sesión | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helpers de normalización de clave de sesión |
  | `plugin-sdk/status-helpers` | Helpers de estado de canal | Constructores de resúmenes de estado de canal/cuenta, valores predeterminados de estado de runtime, helpers de metadatos de incidencias |
  | `plugin-sdk/target-resolver-runtime` | Helpers de resolución de objetivos | Helpers compartidos de resolución de objetivos |
  | `plugin-sdk/string-normalization-runtime` | Helpers de normalización de cadenas | Helpers de normalización de slug/cadenas |
  | `plugin-sdk/request-url` | Helpers de URL de solicitud | Extrae URLs de cadena de entradas similares a solicitudes |
  | `plugin-sdk/run-command` | Helpers de comandos temporizados | Ejecutor de comandos temporizados con stdout/stderr normalizados |
  | `plugin-sdk/param-readers` | Lectores de parámetros | Lectores comunes de parámetros de herramientas/CLI |
  | `plugin-sdk/tool-send` | Extracción de envío de herramientas | Extrae campos canónicos de objetivo de envío desde argumentos de herramientas |
  | `plugin-sdk/temp-path` | Helpers de ruta temporal | Helpers compartidos de ruta temporal de descarga |
  | `plugin-sdk/logging-core` | Helpers de logging | Logger de subsistema y helpers de redacción |
  | `plugin-sdk/markdown-table-runtime` | Helpers de tablas Markdown | Helpers de modo de tabla Markdown |
  | `plugin-sdk/reply-payload` | Tipos de respuesta de mensaje | Tipos de carga útil de respuesta |
  | `plugin-sdk/provider-setup` | Helpers seleccionados de configuración de proveedores locales/autohospedados | Helpers de descubrimiento/configuración de proveedores autohospedados |
  | `plugin-sdk/self-hosted-provider-setup` | Helpers enfocados de configuración de proveedores autohospedados compatibles con OpenAI | Los mismos helpers de descubrimiento/configuración de proveedores autohospedados |
  | `plugin-sdk/provider-auth-runtime` | Helpers de auth de runtime del proveedor | Helpers de resolución de API key en runtime |
  | `plugin-sdk/provider-auth-api-key` | Helpers de configuración de API key del proveedor | Helpers de onboarding/escritura de perfil de API key |
  | `plugin-sdk/provider-auth-result` | Helpers de resultado de auth del proveedor | Constructor estándar de resultado de auth OAuth |
  | `plugin-sdk/provider-auth-login` | Helpers de login interactivo del proveedor | Helpers compartidos de login interactivo |
  | `plugin-sdk/provider-env-vars` | Helpers de variables de entorno del proveedor | Helpers de búsqueda de variables de entorno de auth del proveedor |
  | `plugin-sdk/provider-model-shared` | Helpers compartidos de modelo/repetición del proveedor | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de políticas de repetición, helpers de endpoints del proveedor y helpers de normalización de id de modelo |
  | `plugin-sdk/provider-catalog-shared` | Helpers compartidos de catálogo del proveedor | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Parches de onboarding del proveedor | Helpers de configuración de onboarding |
  | `plugin-sdk/provider-http` | Helpers HTTP del proveedor | Helpers genéricos de HTTP/capacidades de endpoint del proveedor |
  | `plugin-sdk/provider-web-fetch` | Helpers de web-fetch del proveedor | Helpers de registro/caché del proveedor web-fetch |
  | `plugin-sdk/provider-web-search-contract` | Helpers de contrato de búsqueda web del proveedor | Helpers específicos de contrato de configuración/credenciales de búsqueda web como `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` y setters/getters de credenciales con alcance |
  | `plugin-sdk/provider-web-search` | Helpers de búsqueda web del proveedor | Helpers de registro/caché/runtime del proveedor de búsqueda web |
  | `plugin-sdk/provider-tools` | Helpers de compatibilidad de herramientas/esquema del proveedor | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquema Gemini y helpers de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helpers de uso del proveedor | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` y otros helpers de uso del proveedor |
  | `plugin-sdk/provider-stream` | Helpers de wrapper de flujo del proveedor | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de wrapper de flujo y helpers compartidos de wrappers Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Cola async ordenada | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helpers compartidos de medios | Helpers de obtención/transformación/almacenamiento de medios más constructores de carga útil de medios |
  | `plugin-sdk/media-generation-runtime` | Helpers compartidos de generación de medios | Helpers compartidos de failover, selección de candidatos y mensajes de modelo ausente para generación de imagen/video/música |
  | `plugin-sdk/media-understanding` | Helpers de comprensión de medios | Tipos de proveedor de comprensión de medios más exportaciones de helpers de imagen/audio orientados al proveedor |
  | `plugin-sdk/text-runtime` | Helpers compartidos de texto | Eliminación de texto visible para el asistente, helpers de renderizado/fragmentación/tablas Markdown, helpers de redacción, helpers de etiquetas de directiva, utilidades de texto seguro y helpers relacionados de texto/logging |
  | `plugin-sdk/text-chunking` | Helpers de fragmentación de texto | Helper de fragmentación de texto saliente |
  | `plugin-sdk/speech` | Helpers de voz | Tipos de proveedor de voz más helpers orientados al proveedor para directivas, registro y validación |
  | `plugin-sdk/speech-core` | Núcleo compartido de voz | Tipos de proveedor de voz, registro, directivas, normalización |
  | `plugin-sdk/realtime-transcription` | Helpers de transcripción en tiempo real | Tipos de proveedor y helpers de registro |
  | `plugin-sdk/realtime-voice` | Helpers de voz en tiempo real | Tipos de proveedor y helpers de registro |
  | `plugin-sdk/image-generation-core` | Núcleo compartido de generación de imágenes | Tipos de generación de imágenes, failover, auth y helpers de registro |
  | `plugin-sdk/music-generation` | Helpers de generación musical | Tipos de proveedor/solicitud/resultado de generación musical |
  | `plugin-sdk/music-generation-core` | Núcleo compartido de generación musical | Tipos de generación musical, helpers de failover, búsqueda de proveedor y análisis de model-ref |
  | `plugin-sdk/video-generation` | Helpers de generación de video | Tipos de proveedor/solicitud/resultado de generación de video |
  | `plugin-sdk/video-generation-core` | Núcleo compartido de generación de video | Tipos de generación de video, helpers de failover, búsqueda de proveedor y análisis de model-ref |
  | `plugin-sdk/interactive-runtime` | Helpers de respuesta interactiva | Normalización/reducción de carga útil de respuesta interactiva |
  | `plugin-sdk/channel-config-primitives` | Primitivas de configuración de canal | Primitivas específicas de esquema de configuración de canal |
  | `plugin-sdk/channel-config-writes` | Helpers de escritura de configuración de canal | Helpers de autorización de escritura de configuración de canal |
  | `plugin-sdk/channel-plugin-common` | Preludio compartido de canal | Exportaciones compartidas del preludio de plugin de canal |
  | `plugin-sdk/channel-status` | Helpers de estado de canal | Helpers compartidos de instantánea/resumen de estado de canal |
  | `plugin-sdk/allowlist-config-edit` | Helpers de configuración de lista de permitidos | Helpers de edición/lectura de configuración de lista de permitidos |
  | `plugin-sdk/group-access` | Helpers de acceso a grupos | Helpers compartidos de decisión de acceso a grupos |
  | `plugin-sdk/direct-dm` | Helpers de DM directo | Helpers compartidos de auth/protección para DM directo |
  | `plugin-sdk/extension-shared` | Helpers compartidos de extensiones | Primitivas de canal pasivo/estado y helper de proxy ambiental |
  | `plugin-sdk/webhook-targets` | Helpers de objetivos webhook | Registro de objetivos webhook y helpers de instalación de rutas |
  | `plugin-sdk/webhook-path` | Helpers de ruta webhook | Helpers de normalización de ruta webhook |
  | `plugin-sdk/web-media` | Helpers compartidos de medios web | Helpers de carga de medios remotos/locales |
  | `plugin-sdk/zod` | Reexportación de Zod | `zod` reexportado para consumidores del plugin SDK |
  | `plugin-sdk/memory-core` | Helpers empaquetados de memory-core | Superficie helper de administrador/configuración/archivo/CLI de memoria |
  | `plugin-sdk/memory-core-engine-runtime` | Fachada de runtime del motor de memoria | Fachada de runtime de índice/búsqueda de memoria |
  | `plugin-sdk/memory-core-host-engine-foundation` | Motor base del host de memoria | Exportaciones del motor base del host de memoria |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Motor de embeddings del host de memoria | Exportaciones del motor de embeddings del host de memoria |
  | `plugin-sdk/memory-core-host-engine-qmd` | Motor QMD del host de memoria | Exportaciones del motor QMD del host de memoria |
  | `plugin-sdk/memory-core-host-engine-storage` | Motor de almacenamiento del host de memoria | Exportaciones del motor de almacenamiento del host de memoria |
  | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodales del host de memoria | Helpers multimodales del host de memoria |
  | `plugin-sdk/memory-core-host-query` | Helpers de consultas del host de memoria | Helpers de consultas del host de memoria |
  | `plugin-sdk/memory-core-host-secret` | Helpers de secretos del host de memoria | Helpers de secretos del host de memoria |
  | `plugin-sdk/memory-core-host-events` | Helpers de diario de eventos del host de memoria | Helpers de diario de eventos del host de memoria |
  | `plugin-sdk/memory-core-host-status` | Helpers de estado del host de memoria | Helpers de estado del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI del host de memoria | Helpers de runtime CLI del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime principal del host de memoria | Helpers de runtime principal del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-files` | Helpers de archivos/runtime del host de memoria | Helpers de archivos/runtime del host de memoria |
  | `plugin-sdk/memory-host-core` | Alias de runtime principal del host de memoria | Alias neutral de proveedor para helpers del runtime principal del host de memoria |
  | `plugin-sdk/memory-host-events` | Alias del diario de eventos del host de memoria | Alias neutral de proveedor para helpers del diario de eventos del host de memoria |
  | `plugin-sdk/memory-host-files` | Alias de archivos/runtime del host de memoria | Alias neutral de proveedor para helpers de archivos/runtime del host de memoria |
  | `plugin-sdk/memory-host-markdown` | Helpers de markdown gestionado | Helpers compartidos de markdown gestionado para plugins adyacentes a memoria |
  | `plugin-sdk/memory-host-search` | Fachada activa de búsqueda en memoria | Fachada lazy del runtime del administrador de búsqueda de memoria activa |
  | `plugin-sdk/memory-host-status` | Alias de estado del host de memoria | Alias neutral de proveedor para helpers de estado del host de memoria |
  | `plugin-sdk/memory-lancedb` | Helpers empaquetados de memory-lancedb | Superficie helper de memory-lancedb |
  | `plugin-sdk/testing` | Utilidades de prueba | Helpers y mocks de prueba |
</Accordion>

Esta tabla es intencionalmente el subconjunto común de migración, no toda la
superficie del SDK. La lista completa de más de 200 puntos de entrada está en
`scripts/lib/plugin-sdk-entrypoints.json`.

Esa lista todavía incluye algunas superficies helper de plugins empaquetados como
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Siguen exportándose para
mantenimiento y compatibilidad de plugins empaquetados, pero se omiten intencionalmente
de la tabla común de migración y no son el objetivo recomendado para
código nuevo de plugins.

La misma regla se aplica a otras familias de helpers empaquetados como:

- helpers de compatibilidad con navegador: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superficies de helpers/plugins empaquetados como `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` y `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` expone actualmente la superficie específica de helpers de token
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken`.

Usa la importación más específica que se ajuste a la tarea. Si no encuentras una exportación,
consulta el código fuente en `src/plugin-sdk/` o pregunta en Discord.

## Cronograma de eliminación

| Cuándo               | Qué ocurre                                                            |
| -------------------- | --------------------------------------------------------------------- |
| **Ahora**            | Las superficies obsoletas emiten advertencias en tiempo de ejecución  |
| **Próxima versión principal** | Las superficies obsoletas se eliminarán; los plugins que sigan usándolas fallarán |

Todos los plugins principales ya se han migrado. Los plugins externos deberían migrar
antes de la próxima versión principal.

## Suprimir temporalmente las advertencias

Configura estas variables de entorno mientras trabajas en la migración:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Esto es una vía de escape temporal, no una solución permanente.

## Relacionado

- [Primeros pasos](/es/plugins/building-plugins) — crea tu primer plugin
- [Resumen del SDK](/es/plugins/sdk-overview) — referencia completa de importación por subrutas
- [Plugins de canal](/es/plugins/sdk-channel-plugins) — crear plugins de canal
- [Plugins de proveedor](/es/plugins/sdk-provider-plugins) — crear plugins de proveedor
- [Internals de plugins](/es/plugins/architecture) — análisis profundo de la arquitectura
- [Manifiesto del plugin](/es/plugins/manifest) — referencia del esquema del manifiesto
