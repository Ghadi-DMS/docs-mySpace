---
read_when:
    - Ves la advertencia OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ves la advertencia OPENCLAW_EXTENSION_API_DEPRECATED
    - Estás actualizando un plugin a la arquitectura moderna de plugins
    - Mantienes un plugin externo de OpenClaw
sidebarTitle: Migrate to SDK
summary: Migra desde la capa heredada de compatibilidad retroactiva al SDK de plugins moderno
title: Migración del SDK de plugins
x-i18n:
    generated_at: "2026-04-09T01:30:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60cbb6c8be30d17770887d490c14e3a4538563339a5206fb419e51e0558bbc07
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migración del SDK de plugins

OpenClaw ha pasado de una capa amplia de compatibilidad retroactiva a una arquitectura moderna de plugins
con importaciones específicas y documentadas. Si tu plugin se creó antes de
la nueva arquitectura, esta guía te ayudará a migrarlo.

## Qué está cambiando

El sistema anterior de plugins proporcionaba dos superficies muy abiertas que permitían a los plugins importar
todo lo que necesitaran desde un único punto de entrada:

- **`openclaw/plugin-sdk/compat`** — una sola importación que reexportaba decenas de
  ayudantes. Se introdujo para mantener funcionando los plugins basados en hooks más antiguos mientras se
  construía la nueva arquitectura de plugins.
- **`openclaw/extension-api`** — un puente que daba a los plugins acceso directo a
  ayudantes del lado del host, como el ejecutor de agentes integrado.

Ambas superficies ahora están **obsoletas**. Siguen funcionando en tiempo de ejecución, pero los plugins
nuevos no deben usarlas, y los plugins existentes deberían migrar antes de que la próxima
versión principal las elimine.

<Warning>
  La capa de compatibilidad retroactiva se eliminará en una futura versión principal.
  Los plugins que sigan importando desde estas superficies dejarán de funcionar cuando eso ocurra.
</Warning>

## Por qué cambió esto

El enfoque anterior causaba problemas:

- **Inicio lento** — importar un ayudante cargaba decenas de módulos no relacionados
- **Dependencias circulares** — las reexportaciones amplias facilitaban la creación de ciclos de importación
- **Superficie de API poco clara** — no había forma de saber qué exportaciones eran estables y cuáles eran internas

El SDK de plugins moderno corrige esto: cada ruta de importación (`openclaw/plugin-sdk/\<subpath\>`)
es un módulo pequeño, autónomo, con un propósito claro y un contrato documentado.

Las uniones heredadas de conveniencia para proveedores de canales incluidos también han desaparecido. Importaciones
como `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
las uniones de ayudantes con marca de canal y
`openclaw/plugin-sdk/telegram-core` eran atajos privados del monorepo, no
contratos estables de plugins. Usa en su lugar subrutas genéricas y estrechas del SDK. Dentro del
espacio de trabajo de plugins incluidos, conserva los ayudantes propios del proveedor en el
`api.ts` o `runtime-api.ts` de ese plugin.

Ejemplos actuales de proveedores incluidos:

- Anthropic mantiene los ayudantes de streaming específicos de Claude en su propia unión `api.ts` /
  `contract-api.ts`
- OpenAI mantiene los constructores de proveedores, los ayudantes de modelos predeterminados y los constructores
  de proveedores en tiempo real en su propio `api.ts`
- OpenRouter mantiene el constructor del proveedor y los ayudantes de incorporación/configuración en su propio
  `api.ts`

## Cómo migrar

<Steps>
  <Step title="Migra los controladores nativos de aprobación a hechos de capacidad">
    Los plugins de canal con capacidad de aprobación ahora exponen el comportamiento nativo de aprobación mediante
    `approvalCapability.nativeRuntime` más el registro compartido de contexto de ejecución.

    Cambios principales:

    - Reemplaza `approvalCapability.handler.loadRuntime(...)` por
      `approvalCapability.nativeRuntime`
    - Mueve la autenticación/entrega específica de aprobaciones fuera del cableado heredado `plugin.auth` /
      `plugin.approvals` y hacia `approvalCapability`
    - `ChannelPlugin.approvals` se eliminó del contrato público de
      plugin de canal; mueve los campos delivery/native/render a `approvalCapability`
    - `plugin.auth` permanece solo para los flujos de inicio/cierre de sesión del canal; los hooks
      de autenticación de aprobación ahí ya no son leídos por el núcleo
    - Registra objetos de entorno de ejecución propiedad del canal, como clientes, tokens o aplicaciones
      Bolt, mediante `openclaw/plugin-sdk/channel-runtime-context`
    - No envíes avisos de redirección propiedad del plugin desde controladores nativos de aprobación;
      el núcleo ahora es propietario de los avisos de “enrutado a otro lugar” a partir de los resultados reales de entrega
    - Al pasar `channelRuntime` a `createChannelManager(...)`, proporciona una
      superficie real de `createPluginRuntime().channel`. Los stubs parciales se rechazan.

    Consulta `/plugins/sdk-channel-plugins` para ver el diseño actual
    de la capacidad de aprobación.

  </Step>

  <Step title="Audita el comportamiento de fallback del wrapper de Windows">
    Si tu plugin usa `openclaw/plugin-sdk/windows-spawn`, los wrappers `.cmd`/`.bat` de Windows no resueltos ahora fallan de forma cerrada a menos que pases
    explícitamente `allowShellFallback: true`.

    ```typescript
    // Antes
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Después
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Establece esto solo para llamadores de compatibilidad de confianza que
      // aceptan intencionalmente el fallback mediado por shell.
      allowShellFallback: true,
    });
    ```

    Si tu llamador no depende intencionalmente del fallback de shell, no establezcas
    `allowShellFallback` y maneja en su lugar el error lanzado.

  </Step>

  <Step title="Busca importaciones obsoletas">
    Busca en tu plugin importaciones desde cualquiera de las dos superficies obsoletas:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Reemplázalas por importaciones específicas">
    Cada exportación de la superficie anterior se asigna a una ruta de importación moderna específica:

    ```typescript
    // Antes (capa obsoleta de compatibilidad retroactiva)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Después (importaciones modernas específicas)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Para los ayudantes del lado del host, usa el entorno de ejecución del plugin inyectado en lugar de importar
    directamente:

    ```typescript
    // Antes (puente extension-api obsoleto)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Después (entorno de ejecución inyectado)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    El mismo patrón se aplica a otros ayudantes heredados del puente:

    | Importación antigua | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | ayudantes de almacén de sesiones | `api.runtime.agent.session.*` |

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
  | `plugin-sdk/plugin-entry` | Ayudante canónico de entrada de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Reexportación paraguas heredada para definiciones/constructores de entrada de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Exportación del esquema de configuración raíz | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Ayudante de entrada de proveedor único | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definiciones y constructores específicos de entrada de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Ayudantes compartidos del asistente de configuración | Prompts de lista de permitidos, constructores de estado de configuración |
  | `plugin-sdk/setup-runtime` | Ayudantes de entorno de ejecución en tiempo de configuración | Adaptadores de parcheo de configuración seguros para importación, ayudantes de notas de búsqueda, `promptResolvedAllowFrom`, `splitSetupEntries`, proxies de configuración delegados |
  | `plugin-sdk/setup-adapter-runtime` | Ayudantes del adaptador de configuración | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Ayudantes de herramientas de configuración | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Ayudantes de múltiples cuentas | Ayudantes de lista/configuración/puerta de acciones de cuenta |
  | `plugin-sdk/account-id` | Ayudantes de id de cuenta | `DEFAULT_ACCOUNT_ID`, normalización de id de cuenta |
  | `plugin-sdk/account-resolution` | Ayudantes de búsqueda de cuentas | Ayudantes de búsqueda de cuentas + fallback predeterminado |
  | `plugin-sdk/account-helpers` | Ayudantes de cuenta específicos | Ayudantes de lista de cuentas/acciones de cuenta |
  | `plugin-sdk/channel-setup` | Adaptadores del asistente de configuración | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, además de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitivas de emparejamiento DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Cableado de prefijo de respuesta + indicador de escritura | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fábricas de adaptadores de configuración | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Constructores de esquemas de configuración | Tipos de esquema de configuración de canal |
  | `plugin-sdk/telegram-command-config` | Ayudantes de configuración de comandos de Telegram | Normalización de nombres de comandos, recorte de descripciones, validación de duplicados/conflictos |
  | `plugin-sdk/channel-policy` | Resolución de políticas de grupo/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Seguimiento de estado de cuenta | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Ayudantes de envelope de entrada | Ayudantes compartidos de enrutamiento + construcción de envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Ayudantes de respuesta entrante | Ayudantes compartidos de registro y despacho |
  | `plugin-sdk/messaging-targets` | Análisis de destinos de mensajería | Ayudantes de análisis/coincidencia de destinos |
  | `plugin-sdk/outbound-media` | Ayudantes de medios salientes | Carga compartida de medios salientes |
  | `plugin-sdk/outbound-runtime` | Ayudantes de entorno de ejecución saliente | Ayudantes de identidad saliente/delegados de envío |
  | `plugin-sdk/thread-bindings-runtime` | Ayudantes de asociación de hilos | Ciclo de vida y ayudantes de adaptadores de asociación de hilos |
  | `plugin-sdk/agent-media-payload` | Ayudantes heredados de payload de medios | Constructor de payload de medios del agente para diseños heredados de campos |
  | `plugin-sdk/channel-runtime` | Shim de compatibilidad obsoleto | Solo utilidades heredadas de entorno de ejecución de canal |
  | `plugin-sdk/channel-send-result` | Tipos de resultado de envío | Tipos de resultado de respuesta |
  | `plugin-sdk/runtime-store` | Almacenamiento persistente de plugins | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Ayudantes amplios de entorno de ejecución | Ayudantes de entorno de ejecución/logs/backup/instalación de plugins |
  | `plugin-sdk/runtime-env` | Ayudantes específicos de entorno de ejecución | Logger/entorno de ejecución, tiempo de espera, reintentos y ayudantes de backoff |
  | `plugin-sdk/plugin-runtime` | Ayudantes compartidos de entorno de ejecución de plugins | Ayudantes de comandos/hooks/http/interactividad de plugins |
  | `plugin-sdk/hook-runtime` | Ayudantes de canalización de hooks | Ayudantes compartidos de canalización de webhooks/hooks internos |
  | `plugin-sdk/lazy-runtime` | Ayudantes de entorno de ejecución perezoso | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Ayudantes de procesos | Ayudantes compartidos de exec |
  | `plugin-sdk/cli-runtime` | Ayudantes de entorno de ejecución de CLI | Formateo de comandos, esperas, ayudantes de versión |
  | `plugin-sdk/gateway-runtime` | Ayudantes de gateway | Cliente de gateway y ayudantes de parcheo de estado de canal |
  | `plugin-sdk/config-runtime` | Ayudantes de configuración | Ayudantes de carga/escritura de configuración |
  | `plugin-sdk/telegram-command-config` | Ayudantes de comandos de Telegram | Ayudantes de validación de comandos de Telegram con fallback estable cuando la superficie contractual incluida de Telegram no está disponible |
  | `plugin-sdk/approval-runtime` | Ayudantes de prompts de aprobación | Payload de aprobación de exec/plugin, ayudantes de capacidad/perfil de aprobación, ayudantes nativos de enrutamiento/entorno de ejecución de aprobaciones |
  | `plugin-sdk/approval-auth-runtime` | Ayudantes de autenticación de aprobación | Resolución de aprobadores, autenticación de acciones en el mismo chat |
  | `plugin-sdk/approval-client-runtime` | Ayudantes de cliente de aprobación | Ayudantes nativos de perfil/filtro de aprobación de exec |
  | `plugin-sdk/approval-delivery-runtime` | Ayudantes de entrega de aprobación | Adaptadores nativos de capacidad/entrega de aprobación |
  | `plugin-sdk/approval-gateway-runtime` | Ayudantes de gateway de aprobación | Ayudante compartido de resolución de gateway de aprobación |
  | `plugin-sdk/approval-handler-adapter-runtime` | Ayudantes de adaptador de aprobación | Ayudantes ligeros de carga de adaptadores nativos de aprobación para puntos de entrada de canal activos |
  | `plugin-sdk/approval-handler-runtime` | Ayudantes de controlador de aprobación | Ayudantes más amplios del entorno de ejecución del controlador de aprobación; prefiere las uniones más específicas de adaptador/gateway cuando sean suficientes |
  | `plugin-sdk/approval-native-runtime` | Ayudantes de destino de aprobación | Ayudantes nativos de asociación de destino/cuenta de aprobación |
  | `plugin-sdk/approval-reply-runtime` | Ayudantes de respuesta de aprobación | Ayudantes de payload de respuesta de aprobación de exec/plugin |
  | `plugin-sdk/channel-runtime-context` | Ayudantes de contexto de entorno de ejecución de canal | Ayudantes genéricos de registro/obtención/observación de contexto de entorno de ejecución de canal |
  | `plugin-sdk/security-runtime` | Ayudantes de seguridad | Ayudantes compartidos de confianza, control DM, contenido externo y recopilación de secretos |
  | `plugin-sdk/ssrf-policy` | Ayudantes de políticas SSRF | Ayudantes de lista de permitidos de hosts y políticas de red privada |
  | `plugin-sdk/ssrf-runtime` | Ayudantes de entorno de ejecución SSRF | Dispatcher fijado, fetch protegido, ayudantes de políticas SSRF |
  | `plugin-sdk/collection-runtime` | Ayudantes de caché acotada | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Ayudantes de control de diagnósticos | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Ayudantes de formato de errores | `formatUncaughtError`, `isApprovalNotFoundError`, ayudantes de grafo de errores |
  | `plugin-sdk/fetch-runtime` | Ayudantes de fetch/proxy envueltos | `resolveFetch`, ayudantes de proxy |
  | `plugin-sdk/host-runtime` | Ayudantes de normalización del host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Ayudantes de reintento | `RetryConfig`, `retryAsync`, ejecutores de políticas |
  | `plugin-sdk/allow-from` | Formateo de listas de permitidos | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapeo de entrada de listas de permitidos | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Control de comandos y ayudantes de superficie de comandos | `resolveControlCommandGate`, ayudantes de autorización de remitente, ayudantes de registro de comandos |
  | `plugin-sdk/command-status` | Renderizadores de estado/ayuda de comandos | `buildCommandsMessage`, `buildCommandsMessagePaginated`, `buildHelpMessage` |
  | `plugin-sdk/secret-input` | Análisis de entrada de secretos | Ayudantes de entrada de secretos |
  | `plugin-sdk/webhook-ingress` | Ayudantes de solicitudes webhook | Utilidades de destino webhook |
  | `plugin-sdk/webhook-request-guards` | Ayudantes de protección de solicitudes webhook | Ayudantes de lectura/límite de cuerpo de solicitud |
  | `plugin-sdk/reply-runtime` | Entorno de ejecución compartido de respuestas | Despacho entrante, heartbeat, planificador de respuestas, fragmentación |
  | `plugin-sdk/reply-dispatch-runtime` | Ayudantes específicos de despacho de respuestas | Ayudantes de finalización + despacho de proveedor |
  | `plugin-sdk/reply-history` | Ayudantes de historial de respuestas | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planificación de referencias de respuesta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Ayudantes de fragmentación de respuestas | Ayudantes de fragmentación de texto/Markdown |
  | `plugin-sdk/session-store-runtime` | Ayudantes de almacén de sesiones | Ayudantes de ruta de almacén + updated-at |
  | `plugin-sdk/state-paths` | Ayudantes de rutas de estado | Ayudantes de directorios de estado y OAuth |
  | `plugin-sdk/routing` | Ayudantes de enrutamiento/clave de sesión | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, ayudantes de normalización de claves de sesión |
  | `plugin-sdk/status-helpers` | Ayudantes de estado de canal | Constructores de resúmenes de estado de canal/cuenta, valores predeterminados de estado de entorno de ejecución, ayudantes de metadatos de incidencias |
  | `plugin-sdk/target-resolver-runtime` | Ayudantes de resolución de destino | Ayudantes compartidos de resolución de destino |
  | `plugin-sdk/string-normalization-runtime` | Ayudantes de normalización de cadenas | Ayudantes de normalización de slug/cadenas |
  | `plugin-sdk/request-url` | Ayudantes de URL de solicitud | Extrae URL de cadena de entradas tipo solicitud |
  | `plugin-sdk/run-command` | Ayudantes de comandos temporizados | Ejecutor de comandos temporizados con stdout/stderr normalizados |
  | `plugin-sdk/param-readers` | Lectores de parámetros | Lectores comunes de parámetros de herramientas/CLI |
  | `plugin-sdk/tool-payload` | Extracción de payload de herramientas | Extrae payloads normalizados de objetos de resultado de herramientas |
  | `plugin-sdk/tool-send` | Extracción de envío de herramientas | Extrae campos canónicos de destino de envío desde argumentos de herramientas |
  | `plugin-sdk/temp-path` | Ayudantes de rutas temporales | Ayudantes compartidos de rutas temporales de descarga |
  | `plugin-sdk/logging-core` | Ayudantes de logging | Logger de subsistema y ayudantes de redacción |
  | `plugin-sdk/markdown-table-runtime` | Ayudantes de tablas Markdown | Ayudantes de modo de tabla Markdown |
  | `plugin-sdk/reply-payload` | Tipos de respuesta de mensajes | Tipos de payload de respuesta |
  | `plugin-sdk/provider-setup` | Ayudantes curados de configuración de proveedores locales/autohospedados | Ayudantes de detección/configuración de proveedores autohospedados |
  | `plugin-sdk/self-hosted-provider-setup` | Ayudantes específicos de configuración de proveedores autohospedados compatibles con OpenAI | Los mismos ayudantes de detección/configuración de proveedores autohospedados |
  | `plugin-sdk/provider-auth-runtime` | Ayudantes de autenticación de proveedores en tiempo de ejecución | Ayudantes de resolución de claves API en tiempo de ejecución |
  | `plugin-sdk/provider-auth-api-key` | Ayudantes de configuración de claves API de proveedores | Ayudantes de incorporación/escritura de perfil de clave API |
  | `plugin-sdk/provider-auth-result` | Ayudantes de resultado de autenticación de proveedores | Constructor estándar de resultado de autenticación OAuth |
  | `plugin-sdk/provider-auth-login` | Ayudantes de inicio de sesión interactivo de proveedores | Ayudantes compartidos de inicio de sesión interactivo |
  | `plugin-sdk/provider-env-vars` | Ayudantes de variables de entorno de proveedores | Ayudantes de búsqueda de variables de entorno de autenticación del proveedor |
  | `plugin-sdk/provider-model-shared` | Ayudantes compartidos de modelos/replay de proveedores | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructores compartidos de políticas de replay, ayudantes de endpoints de proveedores y ayudantes de normalización de id de modelos |
  | `plugin-sdk/provider-catalog-shared` | Ayudantes compartidos de catálogo de proveedores | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Parches de incorporación de proveedores | Ayudantes de configuración de incorporación |
  | `plugin-sdk/provider-http` | Ayudantes HTTP de proveedores | Ayudantes genéricos de HTTP/capacidades de endpoints de proveedores |
  | `plugin-sdk/provider-web-fetch` | Ayudantes de web-fetch de proveedores | Ayudantes de registro/caché de proveedores web-fetch |
  | `plugin-sdk/provider-web-search-config-contract` | Ayudantes de configuración de búsqueda web de proveedores | Ayudantes específicos de configuración/credenciales de búsqueda web para proveedores que no necesitan cableado de habilitación del plugin |
  | `plugin-sdk/provider-web-search-contract` | Ayudantes contractuales de búsqueda web de proveedores | Ayudantes contractuales específicos de configuración/credenciales de búsqueda web como `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` y setters/getters de credenciales con alcance |
  | `plugin-sdk/provider-web-search` | Ayudantes de búsqueda web de proveedores | Ayudantes de registro/caché/entorno de ejecución de proveedores de búsqueda web |
  | `plugin-sdk/provider-tools` | Ayudantes de compatibilidad de herramientas/esquemas de proveedores | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpieza + diagnósticos de esquemas Gemini y ayudantes de compatibilidad xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Ayudantes de uso de proveedores | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` y otros ayudantes de uso de proveedores |
  | `plugin-sdk/provider-stream` | Ayudantes de envoltorio de streams de proveedores | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de envoltorios de streams y ayudantes compartidos de envoltorios Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Cola asíncrona ordenada | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Ayudantes compartidos de medios | Ayudantes de fetch/transformación/almacenamiento de medios más constructores de payload de medios |
  | `plugin-sdk/media-generation-runtime` | Ayudantes compartidos de generación de medios | Ayudantes compartidos de failover, selección de candidatos y mensajes de modelo faltante para generación de imágenes/video/música |
  | `plugin-sdk/media-understanding` | Ayudantes de comprensión de medios | Tipos de proveedores de comprensión de medios más exportaciones de ayudantes de imagen/audio orientados a proveedores |
  | `plugin-sdk/text-runtime` | Ayudantes compartidos de texto | Eliminación de texto visible para el asistente, renderizado/fragmentación/tablas Markdown, ayudantes de redacción, ayudantes de etiquetas de directivas, utilidades de texto seguro y ayudantes relacionados de texto/logging |
  | `plugin-sdk/text-chunking` | Ayudantes de fragmentación de texto | Ayudante de fragmentación de texto saliente |
  | `plugin-sdk/speech` | Ayudantes de voz | Tipos de proveedores de voz más exportaciones orientadas a proveedores de directivas, registro y validación |
  | `plugin-sdk/speech-core` | Núcleo compartido de voz | Tipos de proveedores de voz, registro, directivas, normalización |
  | `plugin-sdk/realtime-transcription` | Ayudantes de transcripción en tiempo real | Tipos de proveedores y ayudantes de registro |
  | `plugin-sdk/realtime-voice` | Ayudantes de voz en tiempo real | Tipos de proveedores y ayudantes de registro |
  | `plugin-sdk/image-generation-core` | Núcleo compartido de generación de imágenes | Tipos de generación de imágenes, failover, autenticación y ayudantes de registro |
  | `plugin-sdk/music-generation` | Ayudantes de generación musical | Tipos de proveedor/solicitud/resultado de generación musical |
  | `plugin-sdk/music-generation-core` | Núcleo compartido de generación musical | Tipos de generación musical, ayudantes de failover, búsqueda de proveedores y análisis de referencias de modelos |
  | `plugin-sdk/video-generation` | Ayudantes de generación de video | Tipos de proveedor/solicitud/resultado de generación de video |
  | `plugin-sdk/video-generation-core` | Núcleo compartido de generación de video | Tipos de generación de video, ayudantes de failover, búsqueda de proveedores y análisis de referencias de modelos |
  | `plugin-sdk/interactive-runtime` | Ayudantes de respuestas interactivas | Normalización/reducción de payload de respuestas interactivas |
  | `plugin-sdk/channel-config-primitives` | Primitivas de configuración de canal | Primitivas específicas de esquema de configuración de canal |
  | `plugin-sdk/channel-config-writes` | Ayudantes de escritura de configuración de canal | Ayudantes de autorización de escritura de configuración de canal |
  | `plugin-sdk/channel-plugin-common` | Preludio compartido de canal | Exportaciones compartidas de preludio de plugin de canal |
  | `plugin-sdk/channel-status` | Ayudantes de estado de canal | Ayudantes compartidos de instantáneas/resúmenes de estado de canal |
  | `plugin-sdk/allowlist-config-edit` | Ayudantes de configuración de lista de permitidos | Ayudantes de edición/lectura de lista de permitidos |
  | `plugin-sdk/group-access` | Ayudantes de acceso a grupos | Ayudantes compartidos de decisiones de acceso a grupos |
  | `plugin-sdk/direct-dm` | Ayudantes de DM directo | Ayudantes compartidos de autenticación/protección de DM directo |
  | `plugin-sdk/extension-shared` | Ayudantes compartidos de extensiones | Primitivas de canal/estado pasivo y ayudantes de proxy ambiental |
  | `plugin-sdk/webhook-targets` | Ayudantes de destino webhook | Registro de destinos webhook y ayudantes de instalación de rutas |
  | `plugin-sdk/webhook-path` | Ayudantes de ruta webhook | Ayudantes de normalización de rutas webhook |
  | `plugin-sdk/web-media` | Ayudantes compartidos de medios web | Ayudantes de carga de medios remotos/locales |
  | `plugin-sdk/zod` | Reexportación de Zod | Reexportación de `zod` para consumidores del SDK de plugins |
  | `plugin-sdk/memory-core` | Ayudantes incluidos de memory-core | Superficie de ayudantes de gestor/configuración/archivo/CLI de memoria |
  | `plugin-sdk/memory-core-engine-runtime` | Fachada de entorno de ejecución del motor de memoria | Fachada de entorno de ejecución de índice/búsqueda de memoria |
  | `plugin-sdk/memory-core-host-engine-foundation` | Motor base del host de memoria | Exportaciones del motor base del host de memoria |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Motor de embeddings del host de memoria | Exportaciones del motor de embeddings del host de memoria |
  | `plugin-sdk/memory-core-host-engine-qmd` | Motor QMD del host de memoria | Exportaciones del motor QMD del host de memoria |
  | `plugin-sdk/memory-core-host-engine-storage` | Motor de almacenamiento del host de memoria | Exportaciones del motor de almacenamiento del host de memoria |
  | `plugin-sdk/memory-core-host-multimodal` | Ayudantes multimodales del host de memoria | Ayudantes multimodales del host de memoria |
  | `plugin-sdk/memory-core-host-query` | Ayudantes de consultas del host de memoria | Ayudantes de consultas del host de memoria |
  | `plugin-sdk/memory-core-host-secret` | Ayudantes de secretos del host de memoria | Ayudantes de secretos del host de memoria |
  | `plugin-sdk/memory-core-host-events` | Ayudantes de registro de eventos del host de memoria | Ayudantes de registro de eventos del host de memoria |
  | `plugin-sdk/memory-core-host-status` | Ayudantes de estado del host de memoria | Ayudantes de estado del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-cli` | Entorno de ejecución CLI del host de memoria | Ayudantes de entorno de ejecución CLI del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-core` | Entorno de ejecución central del host de memoria | Ayudantes del entorno de ejecución central del host de memoria |
  | `plugin-sdk/memory-core-host-runtime-files` | Ayudantes de archivos/entorno de ejecución del host de memoria | Ayudantes de archivos/entorno de ejecución del host de memoria |
  | `plugin-sdk/memory-host-core` | Alias de entorno de ejecución central del host de memoria | Alias neutral respecto al proveedor para ayudantes del entorno de ejecución central del host de memoria |
  | `plugin-sdk/memory-host-events` | Alias de registro de eventos del host de memoria | Alias neutral respecto al proveedor para ayudantes del registro de eventos del host de memoria |
  | `plugin-sdk/memory-host-files` | Alias de archivos/entorno de ejecución del host de memoria | Alias neutral respecto al proveedor para ayudantes de archivos/entorno de ejecución del host de memoria |
  | `plugin-sdk/memory-host-markdown` | Ayudantes de Markdown gestionado | Ayudantes compartidos de Markdown gestionado para plugins cercanos a la memoria |
  | `plugin-sdk/memory-host-search` | Fachada de búsqueda activa de memoria | Fachada perezosa del entorno de ejecución del gestor de búsqueda de memoria activa |
  | `plugin-sdk/memory-host-status` | Alias de estado del host de memoria | Alias neutral respecto al proveedor para ayudantes de estado del host de memoria |
  | `plugin-sdk/memory-lancedb` | Ayudantes incluidos de memory-lancedb | Superficie de ayudantes de memory-lancedb |
  | `plugin-sdk/testing` | Utilidades de prueba | Ayudantes y mocks de prueba |
</Accordion>

Esta tabla es intencionalmente el subconjunto común de migración, no la superficie completa del SDK.
La lista completa de más de 200 puntos de entrada se encuentra en
`scripts/lib/plugin-sdk-entrypoints.json`.

Esa lista sigue incluyendo algunas uniones de ayudantes de plugins incluidos como
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` y `plugin-sdk/matrix*`. Siguen exportándose para
mantenimiento y compatibilidad de plugins incluidos, pero se omiten intencionalmente
de la tabla común de migración y no son el objetivo recomendado para
código nuevo de plugins.

La misma regla se aplica a otras familias de ayudantes incluidos como:

- ayudantes de soporte de navegador: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superficies de ayudantes/plugins incluidos como `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` y `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` actualmente expone la superficie específica de ayudantes de token
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` y `resolveCopilotApiToken`.

Usa la importación más específica que se ajuste al trabajo. Si no puedes encontrar una exportación,
revisa el código fuente en `src/plugin-sdk/` o pregunta en Discord.

## Cronograma de eliminación

| Cuándo | Qué ocurre |
| ---------------------- | ----------------------------------------------------------------------- |
| **Ahora** | Las superficies obsoletas emiten advertencias en tiempo de ejecución |
| **Próxima versión principal** | Las superficies obsoletas se eliminarán; los plugins que sigan usándolas fallarán |

Todos los plugins principales ya se han migrado. Los plugins externos deberían migrar
antes de la próxima versión principal.

## Cómo suprimir temporalmente las advertencias

Establece estas variables de entorno mientras trabajas en la migración:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Esta es una vía de escape temporal, no una solución permanente.

## Relacionado

- [Primeros pasos](/es/plugins/building-plugins) — crea tu primer plugin
- [Resumen del SDK](/es/plugins/sdk-overview) — referencia completa de importaciones por subruta
- [Plugins de canal](/es/plugins/sdk-channel-plugins) — creación de plugins de canal
- [Plugins de proveedor](/es/plugins/sdk-provider-plugins) — creación de plugins de proveedor
- [Aspectos internos de plugins](/es/plugins/architecture) — análisis detallado de la arquitectura
- [Manifiesto del plugin](/es/plugins/manifest) — referencia del esquema del manifiesto
