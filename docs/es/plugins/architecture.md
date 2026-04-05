---
read_when:
    - Crear o depurar plugins nativos de OpenClaw
    - Comprender el modelo de capacidades de plugins o los límites de propiedad
    - Trabajar en el pipeline de carga o el registro de plugins
    - Implementar hooks de tiempo de ejecución de proveedores o plugins de canal
sidebarTitle: Internals
summary: 'Aspectos internos de plugins: modelo de capacidades, propiedad, contratos, pipeline de carga y helpers de tiempo de ejecución'
title: Aspectos internos de plugins
x-i18n:
    generated_at: "2026-04-05T12:52:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1bc9d7261c3c7878d37140be77f210dd262d6c3edee2491ea534aa599e2800c0
    source_path: plugins/architecture.md
    workflow: 15
---

# Aspectos internos de plugins

<Info>
  Esta es la **referencia profunda de arquitectura**. Para guías prácticas, consulta:
  - [Install and use plugins](/tools/plugin) — guía de usuario
  - [Getting Started](/plugins/building-plugins) — primer tutorial de plugins
  - [Channel Plugins](/plugins/sdk-channel-plugins) — crea un canal de mensajería
  - [Provider Plugins](/plugins/sdk-provider-plugins) — crea un proveedor de modelos
  - [SDK Overview](/plugins/sdk-overview) — mapa de imports y API de registro
</Info>

Esta página cubre la arquitectura interna del sistema de plugins de OpenClaw.

## Modelo público de capacidades

Las capacidades son el modelo público de **plugin nativo** dentro de OpenClaw. Cada
plugin nativo de OpenClaw se registra en uno o más tipos de capacidad:

| Capacidad              | Método de registro                              | Plugins de ejemplo                  |
| ---------------------- | ------------------------------------------------ | ----------------------------------- |
| Inferencia de texto    | `api.registerProvider(...)`                      | `openai`, `anthropic`               |
| Backend de inferencia CLI | `api.registerCliBackend(...)`                 | `openai`, `anthropic`               |
| Voz                    | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`           |
| Transcripción en tiempo real | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                      |
| Voz en tiempo real     | `api.registerRealtimeVoiceProvider(...)`         | `openai`                            |
| Comprensión multimedia | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                  |
| Generación de imágenes | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Generación de video    | `api.registerVideoGenerationProvider(...)`       | `qwen`                              |
| Obtención web          | `api.registerWebFetchProvider(...)`              | `firecrawl`                         |
| Búsqueda web           | `api.registerWebSearchProvider(...)`             | `google`                            |
| Canal / mensajería     | `api.registerChannel(...)`                       | `msteams`, `matrix`                 |

Un plugin que registra cero capacidades pero proporciona hooks, herramientas o
servicios es un plugin **heredado solo de hooks**. Ese patrón sigue siendo totalmente compatible.

### Postura de compatibilidad externa

El modelo de capacidades ya está integrado en el núcleo y lo usan hoy los plugins
nativos/incluidos, pero la compatibilidad de plugins externos aún necesita un listón
más estricto que “si se exporta, entonces está congelado”.

Guía actual:

- **plugins externos existentes:** mantén funcionando las integraciones basadas en hooks; trátalo
  como la base de compatibilidad
- **nuevos plugins nativos/incluidos:** da preferencia al registro explícito de capacidades sobre
  accesos específicos de proveedor o nuevos diseños solo con hooks
- **plugins externos que adoptan el registro de capacidades:** permitido, pero trata las
  superficies helper específicas de capacidad como cambiantes, a menos que la documentación marque explícitamente un contrato como estable

Regla práctica:

- las API de registro de capacidades son la dirección prevista
- los hooks heredados siguen siendo la ruta más segura para evitar rupturas en plugins externos durante
  la transición
- no todas las subrutas helper exportadas son iguales; da preferencia al contrato
  documentado y estrecho, no a exports helper incidentales

### Formas de los plugins

OpenClaw clasifica cada plugin cargado en una forma según su comportamiento real
de registro (no solo según metadatos estáticos):

- **plain-capability** -- registra exactamente un tipo de capacidad (por ejemplo, un
  plugin solo de proveedor como `mistral`)
- **hybrid-capability** -- registra varios tipos de capacidad (por ejemplo
  `openai` posee inferencia de texto, voz, comprensión multimedia y generación
  de imágenes)
- **hook-only** -- registra solo hooks (tipados o personalizados), sin capacidades,
  herramientas, comandos ni servicios
- **non-capability** -- registra herramientas, comandos, servicios o rutas, pero no
  capacidades

Usa `openclaw plugins inspect <id>` para ver la forma de un plugin y el
desglose de capacidades. Consulta [CLI reference](/cli/plugins#inspect) para más detalles.

### Hooks heredados

El hook `before_agent_start` sigue siendo compatible como ruta de compatibilidad para
plugins solo con hooks. Plugins heredados reales siguen dependiendo de él.

Dirección:

- mantenerlo funcionando
- documentarlo como heredado
- preferir `before_model_resolve` para trabajo de reemplazo de modelo/proveedor
- preferir `before_prompt_build` para trabajo de mutación del prompt
- eliminarlo solo cuando el uso real caiga y la cobertura de fixtures demuestre la seguridad de la migración

### Señales de compatibilidad

Cuando ejecutes `openclaw doctor` o `openclaw plugins inspect <id>`, puedes ver
una de estas etiquetas:

| Señal                     | Significado                                                  |
| ------------------------- | ------------------------------------------------------------ |
| **config valid**          | La configuración se analiza correctamente y los plugins se resuelven |
| **compatibility advisory** | El plugin usa un patrón compatible pero más antiguo (p. ej. `hook-only`) |
| **legacy warning**        | El plugin usa `before_agent_start`, que está obsoleto        |
| **hard error**            | La configuración no es válida o el plugin no se pudo cargar  |

Ni `hook-only` ni `before_agent_start` romperán tu plugin hoy:
`hook-only` es informativo y `before_agent_start` solo activa una advertencia. Estas
señales también aparecen en `openclaw status --all` y `openclaw plugins doctor`.

## Descripción general de la arquitectura

El sistema de plugins de OpenClaw tiene cuatro capas:

1. **Manifest + descubrimiento**
   OpenClaw encuentra plugins candidatos a partir de rutas configuradas, raíces
   de espacio de trabajo, raíces globales de extensiones y extensiones incluidas. El descubrimiento lee primero
   los manifests nativos `openclaw.plugin.json` más los manifests de bundles compatibles.
2. **Habilitación + validación**
   El núcleo decide si un plugin descubierto está habilitado, deshabilitado, bloqueado o
   seleccionado para una ranura exclusiva como memory.
3. **Carga en tiempo de ejecución**
   Los plugins nativos de OpenClaw se cargan en proceso mediante jiti y registran
   capacidades en un registro central. Los bundles compatibles se normalizan en
   registros del registro sin importar código de tiempo de ejecución.
4. **Consumo de superficie**
   El resto de OpenClaw lee el registro para exponer herramientas, canales, configuración
   de proveedores, hooks, rutas HTTP, comandos CLI y servicios.

Para la CLI de plugins específicamente, el descubrimiento de comandos raíz se divide en dos fases:

- los metadatos en tiempo de análisis vienen de `registerCli(..., { descriptors: [...] })`
- el módulo CLI real del plugin puede seguir siendo lazy y registrarse en la primera invocación

Eso mantiene el código CLI propiedad del plugin dentro del plugin y aun así permite a OpenClaw
reservar nombres de comandos raíz antes del análisis.

El límite de diseño importante:

- el descubrimiento + la validación de configuración deben funcionar a partir de **metadatos de manifest/schema**
  sin ejecutar código del plugin
- el comportamiento nativo de tiempo de ejecución proviene de la ruta `register(api)` del módulo del plugin

Esta división permite a OpenClaw validar la configuración, explicar plugins faltantes/deshabilitados y
crear sugerencias de UI/schema antes de que el tiempo de ejecución completo esté activo.

### Plugins de canal y la herramienta compartida de mensajes

Los plugins de canal no necesitan registrar una herramienta independiente de send/edit/react para
las acciones normales de chat. OpenClaw mantiene una herramienta compartida `message` en el núcleo, y
los plugins de canal poseen el descubrimiento y la ejecución específicos del canal detrás de ella.

El límite actual es:

- el núcleo posee el host de la herramienta compartida `message`, el cableado del prompt, la contabilidad
  de sesiones/hilos y el despacho de ejecución
- los plugins de canal poseen el descubrimiento de acciones con alcance, el descubrimiento de capacidades y cualquier
  fragmento de schema específico del canal
- los plugins de canal poseen la gramática de conversación de sesión específica del proveedor, como
  cómo los ids de conversación codifican ids de hilo o heredan de conversaciones padre
- los plugins de canal ejecutan la acción final a través de su adaptador de acciones

Para plugins de canal, la superficie del SDK es
`ChannelMessageActionAdapter.describeMessageTool(...)`. Esa llamada unificada de descubrimiento
permite que un plugin devuelva sus acciones visibles, capacidades y contribuciones al schema
juntas, para que esas piezas no se desalineen.

El núcleo pasa el alcance de tiempo de ejecución a ese paso de descubrimiento. Los campos importantes incluyen:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` entrante de confianza

Esto importa para plugins sensibles al contexto. Un canal puede ocultar o exponer
acciones de mensajes según la cuenta activa, la sala/hilo/mensaje actual o
la identidad de confianza del solicitante, sin codificar ramas específicas del canal en la
herramienta principal `message`.

Por eso los cambios de enrutamiento del embedded-runner siguen siendo trabajo del plugin: el ejecutor es
responsable de reenviar la identidad actual del chat/sesión al límite de descubrimiento del plugin para que la herramienta compartida `message` exponga la superficie correcta,
propiedad del canal, para el turno actual.

Para helpers de ejecución propiedad del canal, los plugins incluidos deben mantener el tiempo de ejecución de ejecución
dentro de sus propios módulos de extensión. El núcleo ya no posee los tiempos de ejecución de acciones de mensaje de Discord,
Slack, Telegram o WhatsApp bajo `src/agents/tools`.
No publicamos subrutas separadas `plugin-sdk/*-action-runtime`, y los plugins incluidos
deben importar directamente su propio código de tiempo de ejecución local desde sus
módulos propiedad de la extensión.

El mismo límite se aplica a los seams del SDK con nombre de proveedor en general: el núcleo no
debe importar barrels de conveniencia específicos del canal para Slack, Discord, Signal,
WhatsApp o extensiones similares. Si el núcleo necesita un comportamiento, debe consumir el
barrel `api.ts` / `runtime-api.ts` del propio plugin incluido o promover la necesidad
a una capacidad genérica estrecha en el SDK compartido.

Para polls específicamente, hay dos rutas de ejecución:

- `outbound.sendPoll` es la base compartida para canales que encajan en el modelo común de
  poll
- `actions.handleAction("poll")` es la ruta preferida para semánticas de poll específicas del canal o parámetros extra de poll

Ahora el núcleo difiere el análisis compartido de poll hasta que el despacho de poll del plugin rechaza
la acción, para que los manejadores de poll propiedad del plugin puedan aceptar campos específicos
del canal sin quedar bloqueados primero por el analizador genérico de poll.

Consulta [Load pipeline](#load-pipeline) para la secuencia completa de inicio.

## Modelo de propiedad de capacidades

OpenClaw trata un plugin nativo como el límite de propiedad para una **empresa** o una
**función**, no como una colección desordenada de integraciones no relacionadas.

Eso significa:

- un plugin de empresa normalmente debe poseer todas las superficies de OpenClaw de esa empresa
- un plugin de función normalmente debe poseer toda la superficie de la función que introduce
- los canales deben consumir capacidades compartidas del núcleo en lugar de volver a implementar
  comportamiento de proveedor de forma ad hoc

Ejemplos:

- el plugin incluido `openai` posee el comportamiento del proveedor de modelos OpenAI y el comportamiento
  de voz + realtime-voice + media-understanding + image-generation de OpenAI
- el plugin incluido `elevenlabs` posee el comportamiento de voz de ElevenLabs
- el plugin incluido `microsoft` posee el comportamiento de voz de Microsoft
- el plugin incluido `google` posee el comportamiento del proveedor de modelos Google más el comportamiento
  de media-understanding + image-generation + web-search de Google
- el plugin incluido `firecrawl` posee el comportamiento de web-fetch de Firecrawl
- los plugins incluidos `minimax`, `mistral`, `moonshot` y `zai` poseen sus
  backends de media-understanding
- el plugin `voice-call` es un plugin de función: posee transporte de llamadas, herramientas,
  CLI, rutas y el puente de media-stream de Twilio, pero consume capacidades compartidas de voz
  más realtime-transcription y realtime-voice en lugar de importar plugins de proveedor directamente

El estado final previsto es:

- OpenAI vive en un solo plugin aunque abarque modelos de texto, voz, imágenes y
  futuro video
- otro proveedor puede hacer lo mismo para su propia superficie
- los canales no se preocupan por qué plugin de proveedor posee el proveedor; consumen el
  contrato de capacidad compartida expuesto por el núcleo

Esta es la distinción clave:

- **plugin** = límite de propiedad
- **capability** = contrato central que varios plugins pueden implementar o consumir

Por lo tanto, si OpenClaw agrega un nuevo dominio como video, la primera pregunta no es
“¿qué proveedor debería codificar video de forma fija?” La primera pregunta es “¿cuál es
el contrato central de capacidad de video?” Una vez que ese contrato existe, los plugins de proveedor
pueden registrarse contra él y los plugins de canal/función pueden consumirlo.

Si la capacidad aún no existe, el movimiento correcto suele ser:

1. definir la capacidad faltante en el núcleo
2. exponerla a través de la API/runtime del plugin de forma tipada
3. conectar canales/funciones contra esa capacidad
4. dejar que los plugins de proveedor registren implementaciones

Esto mantiene explícita la propiedad y evita al mismo tiempo un comportamiento del núcleo que dependa de un
único proveedor o de una ruta de código específica y aislada de un plugin.

### Capas de capacidad

Usa este modelo mental al decidir dónde debe ir el código:

- **capa central de capacidad**: orquestación compartida, política, respaldo, reglas de fusión
  de configuración, semántica de entrega y contratos tipados
- **capa de plugin de proveedor**: APIs específicas del proveedor, autenticación, catálogos de modelos, síntesis de voz,
  generación de imágenes, futuros backends de video, endpoints de uso
- **capa de plugin de canal/función**: integración de Slack/Discord/voice-call/etc.
  que consume capacidades centrales y las presenta en una superficie

Por ejemplo, TTS sigue esta forma:

- el núcleo posee la política TTS en tiempo de respuesta, el orden de respaldo, preferencias y entrega por canal
- `openai`, `elevenlabs` y `microsoft` poseen las implementaciones de síntesis
- `voice-call` consume el helper de tiempo de ejecución TTS de telefonía

Ese mismo patrón debe preferirse para futuras capacidades.

### Ejemplo de plugin de empresa con varias capacidades

Un plugin de empresa debe sentirse cohesivo desde fuera. Si OpenClaw tiene contratos compartidos
para modelos, voz, transcripción en tiempo real, voz en tiempo real, comprensión multimedia,
generación de imágenes, generación de video, web fetch y web search, un proveedor puede poseer
todas sus superficies en un solo lugar:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // hooks de auth/catálogo de modelos/tiempo de ejecución
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // configuración de voz del proveedor — implementa directamente la interfaz SpeechProviderPlugin
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // lógica de credenciales + fetch
      }),
    );
  },
};

export default plugin;
```

Lo importante no son los nombres exactos de los helpers. Importa la forma:

- un plugin posee la superficie del proveedor
- el núcleo sigue poseyendo los contratos de capacidad
- los canales y plugins de función consumen helpers `api.runtime.*`, no código del proveedor
- las pruebas de contrato pueden afirmar que el plugin registró las capacidades que
  dice poseer

### Ejemplo de capacidad: comprensión de video

OpenClaw ya trata la comprensión de imagen/audio/video como una capacidad compartida.
El mismo modelo de propiedad se aplica allí:

1. el núcleo define el contrato de media-understanding
2. los plugins de proveedor registran `describeImage`, `transcribeAudio` y
   `describeVideo` según corresponda
3. los plugins de canal y de función consumen el comportamiento compartido del núcleo en lugar de
   conectarse directamente al código del proveedor

Eso evita incorporar en el núcleo las suposiciones de video de un proveedor. El plugin posee
la superficie del proveedor; el núcleo posee el contrato de capacidad y el comportamiento de respaldo.

La generación de video ya sigue esa misma secuencia: el núcleo posee el contrato de capacidad
tipado y el helper de tiempo de ejecución, y los plugins de proveedor registran
implementaciones `api.registerVideoGenerationProvider(...)` contra él.

¿Necesitas una lista concreta de despliegue? Consulta
[Capability Cookbook](/tools/capability-cookbook).

## Contratos y aplicación

La superficie de la API de plugins es intencionalmente tipada y centralizada en
`OpenClawPluginApi`. Ese contrato define los puntos de registro compatibles y
los helpers de tiempo de ejecución de los que un plugin puede depender.

Por qué importa:

- los autores de plugins obtienen un estándar interno estable
- el núcleo puede rechazar propiedad duplicada como dos plugins que registren el mismo
  id de proveedor
- el inicio puede mostrar diagnósticos accionables para registros mal formados
- las pruebas de contrato pueden hacer cumplir la propiedad de plugins incluidos y evitar derivas silenciosas

Hay dos capas de aplicación:

1. **aplicación del registro en tiempo de ejecución**
   El registro de plugins valida los registros a medida que se cargan los plugins. Ejemplos:
   ids de proveedor duplicados, ids de proveedor de voz duplicados y registros
   mal formados producen diagnósticos de plugins en lugar de comportamiento indefinido.
2. **pruebas de contrato**
   Los plugins incluidos se capturan en registros de contrato durante las ejecuciones de prueba para que
   OpenClaw pueda afirmar explícitamente la propiedad. Hoy esto se usa para
   proveedores de modelos, proveedores de voz, proveedores de búsqueda web y
   propiedad de registro incluida.

El efecto práctico es que OpenClaw sabe, por adelantado, qué plugin posee qué
superficie. Eso permite que el núcleo y los canales compongan sin fricciones porque la
propiedad está declarada, tipada y es comprobable, en lugar de implícita.

### Qué pertenece a un contrato

Los buenos contratos de plugins son:

- tipados
- pequeños
- específicos por capacidad
- propiedad del núcleo
- reutilizables por varios plugins
- consumibles por canales/funciones sin conocimiento del proveedor

Los malos contratos de plugins son:

- política específica del proveedor oculta en el núcleo
- escapes puntuales de plugin que omiten el registro
- código de canal que accede directamente a una implementación de proveedor
- objetos ad hoc de tiempo de ejecución que no forman parte de `OpenClawPluginApi` ni de
  `api.runtime`

En caso de duda, eleva el nivel de abstracción: define primero la capacidad y luego
deja que los plugins se conecten a ella.

## Modelo de ejecución

Los plugins nativos de OpenClaw se ejecutan **en proceso** con el Gateway. No están
aislados. Un plugin nativo cargado tiene el mismo límite de confianza a nivel de proceso que
el código del núcleo.

Implicaciones:

- un plugin nativo puede registrar herramientas, manejadores de red, hooks y servicios
- un error en un plugin nativo puede hacer caer o desestabilizar el gateway
- un plugin nativo malicioso equivale a ejecución de código arbitrario dentro
  del proceso de OpenClaw

Los bundles compatibles son más seguros por defecto porque OpenClaw actualmente los trata
como paquetes de metadatos/contenido. En las versiones actuales, eso significa sobre todo
Skills incluidos.

Usa allowlists y rutas explícitas de instalación/carga para plugins no incluidos. Trata
los plugins de espacio de trabajo como código de desarrollo, no como valores predeterminados de producción.

Para nombres de paquetes de espacio de trabajo incluidos, mantén el id del plugin anclado al nombre npm:
`@openclaw/<id>` por defecto, o un sufijo tipado aprobado como
`-provider`, `-plugin`, `-speech`, `-sandbox` o `-media-understanding` cuando
el paquete expone intencionalmente un rol de plugin más estrecho.

Nota importante sobre confianza:

- `plugins.allow` confía en **ids de plugin**, no en la procedencia de la fuente.
- Un plugin de espacio de trabajo con el mismo id que un plugin incluido oculta intencionalmente
  la copia incluida cuando ese plugin de espacio de trabajo está habilitado/en allowlist.
- Esto es normal y útil para desarrollo local, pruebas de parches y hotfixes.

## Límite de exportación

OpenClaw exporta capacidades, no comodidad de implementación.

Mantén público el registro de capacidades. Reduce exports helper que no formen parte del contrato:

- subrutas helper específicas de plugins incluidos
- subrutas de plomería de tiempo de ejecución no pensadas como API pública
- helpers de conveniencia específicos del proveedor
- helpers de setup/onboarding que son detalles de implementación

Algunas subrutas helper de plugins incluidos aún permanecen en el mapa de exports generado del SDK
por compatibilidad y mantenimiento de plugins incluidos. Ejemplos actuales incluyen
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` y varios seams `plugin-sdk/matrix*`. Trátalos como
exports reservados de detalle de implementación, no como el patrón recomendado del SDK para
nuevos plugins de terceros.

## Pipeline de carga

Al iniciar, OpenClaw hace aproximadamente esto:

1. descubrir raíces candidatas de plugins
2. leer manifests nativos o compatibles de bundles y metadatos de paquetes
3. rechazar candidatos inseguros
4. normalizar la configuración de plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decidir la habilitación para cada candidato
6. cargar módulos nativos habilitados mediante jiti
7. llamar a los hooks nativos `register(api)` (o `activate(api)` — un alias heredado) y recopilar registros en el registro de plugins
8. exponer el registro a superficies de comandos/runtime

<Note>
`activate` es un alias heredado de `register`: el cargador resuelve el que esté presente (`def.register ?? def.activate`) y lo llama en el mismo punto. Todos los plugins incluidos usan `register`; prefiere `register` para plugins nuevos.
</Note>

Las barreras de seguridad ocurren **antes** de la ejecución en tiempo de ejecución. Los candidatos se bloquean
cuando la entrada escapa de la raíz del plugin, la ruta es escribible por todos, o la
propiedad de la ruta parece sospechosa para plugins no incluidos.

### Comportamiento manifest-first

El manifest es la fuente de verdad del plano de control. OpenClaw lo usa para:

- identificar el plugin
- descubrir canales/skills/schema de configuración declarados o capacidades del bundle
- validar `plugins.entries.<id>.config`
- ampliar etiquetas/placeholders de la UI de control
- mostrar metadatos de instalación/catálogo

Para plugins nativos, el módulo de tiempo de ejecución es la parte del plano de datos. Registra
el comportamiento real, como hooks, herramientas, comandos o flujos de proveedores.

### Qué almacena en caché el cargador

OpenClaw mantiene cachés cortas en proceso para:

- resultados de descubrimiento
- datos del registro de manifests
- registros de plugins cargados

Estas cachés reducen inicios con picos y la sobrecarga de comandos repetidos. Es seguro
pensar en ellas como cachés de rendimiento de corta vida, no como persistencia.

Nota de rendimiento:

- Define `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` o
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` para desactivar estas cachés.
- Ajusta las ventanas de caché con `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` y
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Modelo de registro

Los plugins cargados no mutan directamente globals aleatorios del núcleo. Se registran en un
registro central de plugins.

El registro rastrea:

- registros de plugins (identidad, fuente, origen, estado, diagnósticos)
- herramientas
- hooks heredados y hooks tipados
- canales
- proveedores
- manejadores RPC del gateway
- rutas HTTP
- registradores CLI
- servicios en segundo plano
- comandos propiedad del plugin

Las funciones centrales luego leen de ese registro en lugar de hablar directamente con los módulos de plugins.
Esto mantiene la carga unidireccional:

- módulo del plugin -> registro en el registro
- tiempo de ejecución del núcleo -> consumo del registro

Esa separación importa para el mantenimiento. Significa que la mayoría de las superficies centrales solo
necesitan un punto de integración: “leer el registro”, no “hacer casos especiales para cada
módulo de plugin”.

## Callbacks de binding de conversación

Los plugins que vinculan una conversación pueden reaccionar cuando se resuelve una aprobación.

Usa `api.onConversationBindingResolved(...)` para recibir un callback después de que una solicitud de binding sea aprobada o denegada:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Ahora existe un binding para este plugin + conversación.
        console.log(event.binding?.conversationId);
        return;
      }

      // La solicitud fue denegada; limpia cualquier estado local pendiente.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Campos de la carga del callback:

- `status`: `"approved"` o `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` o `"deny"`
- `binding`: el binding resuelto para solicitudes aprobadas
- `request`: el resumen original de la solicitud, sugerencia de desacople, id del remitente y
  metadatos de conversación

Este callback es solo de notificación. No cambia quién puede vincular una
conversación y se ejecuta después de que el manejo central de aprobación termina.

## Hooks de tiempo de ejecución del proveedor

Los plugins de proveedor ahora tienen dos capas:

- metadatos del manifest: `providerAuthEnvVars` para búsqueda barata de auth por entorno antes de
  cargar el runtime, más `providerAuthChoices` para etiquetas baratas de onboarding/auth-choice
  y metadatos de flags CLI antes de cargar el runtime
- hooks en tiempo de configuración: `catalog` / heredado `discovery` más `applyConfigDefaults`
- hooks de tiempo de ejecución: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw sigue poseyendo el bucle genérico del agente, failover, el manejo de transcripciones y la
política de herramientas. Estos hooks son la superficie de extensión para comportamiento específico del proveedor sin
necesitar un transporte de inferencia completamente personalizado.

Usa el manifest `providerAuthEnvVars` cuando el proveedor tenga credenciales basadas en entorno
que las rutas genéricas de auth/status/model-picker deban ver sin cargar el runtime del plugin.
Usa el manifest `providerAuthChoices` cuando las superficies CLI de onboarding/auth-choice
deban conocer el id de elección del proveedor, etiquetas de grupo y
cableado simple de auth con un solo flag sin cargar el runtime del proveedor. Mantén `envVars` del runtime del proveedor para sugerencias orientadas al operador como etiquetas de onboarding o variables de
configuración de client-id/client-secret de OAuth.

### Orden y uso de hooks

Para plugins de modelo/proveedor, OpenClaw llama a los hooks en este orden aproximado.
La columna “Cuándo usarlo” es la guía rápida de decisión.

| #   | Hook                              | Qué hace                                                                               | Cuándo usarlo                                                                                                                             |
| --- | --------------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publica configuración del proveedor en `models.providers` durante la generación de `models.json` | El proveedor posee un catálogo o valores predeterminados de base URL                                                                     |
| 2   | `applyConfigDefaults`             | Aplica valores predeterminados globales propiedad del proveedor durante la materialización de configuración | Los valores predeterminados dependen del modo de auth, del entorno o de la semántica de la familia de modelos del proveedor            |
| --  | _(búsqueda integrada de modelo)_  | OpenClaw prueba primero la ruta normal de registro/catálogo                             | _(no es un hook de plugin)_                                                                                                              |
| 3   | `normalizeModelId`                | Normaliza alias heredados o preview de ids de modelo antes de la búsqueda              | El proveedor posee la limpieza de alias antes de la resolución canónica del modelo                                                       |
| 4   | `normalizeTransport`              | Normaliza `api` / `baseUrl` de la familia del proveedor antes del ensamblaje genérico del modelo | El proveedor posee la limpieza del transporte para ids personalizados de proveedor en la misma familia de transporte                    |
| 5   | `normalizeConfig`                 | Normaliza `models.providers.<id>` antes de la resolución de runtime/proveedor          | El proveedor necesita limpieza de configuración que deba vivir con el plugin; los helpers incluidos de la familia Google también respaldan entradas compatibles de configuración Google |
| 6   | `applyNativeStreamingUsageCompat` | Aplica reescrituras de compatibilidad de uso de streaming nativo a proveedores configurados | El proveedor necesita correcciones de metadatos de uso de streaming nativo basadas en endpoint                                          |
| 7   | `resolveConfigApiKey`             | Resuelve auth de marcador de entorno para proveedores configurados antes de cargar el runtime de auth | El proveedor tiene resolución de API key mediante marcador de entorno propiedad del proveedor; `amazon-bedrock` también tiene aquí un resolvedor integrado de marcador de entorno AWS |
| 8   | `resolveSyntheticAuth`            | Expone auth local/self-hosted o basada en configuración sin conservar texto sin formato | El proveedor puede operar con un marcador de credencial sintético/local                                                                  |
| 9   | `shouldDeferSyntheticProfileAuth` | Baja la prioridad de placeholders sintéticos almacenados detrás de auth basada en entorno/configuración | El proveedor almacena perfiles sintéticos placeholder que no deben ganar precedencia                                                    |
| 10  | `resolveDynamicModel`             | Respaldo síncrono para ids de modelo propiedad del proveedor que aún no están en el registro local | El proveedor acepta ids de modelo upstream arbitrarios                                                                                   |
| 11  | `prepareDynamicModel`             | Calentamiento asíncrono, luego `resolveDynamicModel` vuelve a ejecutarse               | El proveedor necesita metadatos de red antes de resolver ids desconocidos                                                                |
| 12  | `normalizeResolvedModel`          | Reescritura final antes de que el embedded runner use el modelo resuelto               | El proveedor necesita reescrituras de transporte pero sigue usando un transporte central                                                 |
| 13  | `contributeResolvedModelCompat`   | Aporta indicadores de compatibilidad para modelos de proveedor detrás de otro transporte compatible | El proveedor reconoce sus propios modelos en transportes proxy sin asumir el control del proveedor                                       |
| 14  | `capabilities`                    | Metadatos de transcripción/herramientas propiedad del proveedor usados por lógica central compartida | El proveedor necesita particularidades de transcripción/familia de proveedor                                                             |
| 15  | `normalizeToolSchemas`            | Normaliza schemas de herramientas antes de que el embedded runner los vea              | El proveedor necesita limpieza de schema de la familia de transporte                                                                     |
| 16  | `inspectToolSchemas`              | Expone diagnósticos de schema propiedad del proveedor después de la normalización      | El proveedor quiere advertencias de palabras clave sin enseñar al núcleo reglas específicas del proveedor                               |
| 17  | `resolveReasoningOutputMode`      | Selecciona contrato de salida de razonamiento nativo frente a etiquetado               | El proveedor necesita razonamiento/salida final etiquetados en lugar de campos nativos                                                  |
| 18  | `prepareExtraParams`              | Normalización de parámetros de solicitud antes de wrappers genéricos de opciones de stream | El proveedor necesita parámetros predeterminados de solicitud o limpieza de parámetros por proveedor                                     |
| 19  | `createStreamFn`                  | Reemplaza completamente la ruta normal de stream por un transporte personalizado       | El proveedor necesita un protocolo wire personalizado, no solo un wrapper                                                                |
| 20  | `wrapStreamFn`                    | Wrapper de stream después de aplicar wrappers genéricos                                | El proveedor necesita wrappers de compatibilidad de headers/body/modelo sin un transporte personalizado                                  |
| 21  | `resolveTransportTurnState`       | Adjunta headers nativos por turno o metadatos de transporte                            | El proveedor quiere que transportes genéricos envíen identidad de turno nativa del proveedor                                             |
| 22  | `resolveWebSocketSessionPolicy`   | Adjunta headers nativos de WebSocket o política de enfriamiento de sesión              | El proveedor quiere que los transportes WS genéricos ajusten headers de sesión o política de respaldo                                   |
| 23  | `formatApiKey`                    | Formateador de perfil de auth: el perfil almacenado se convierte en la cadena `apiKey` de runtime | El proveedor almacena metadatos extra de auth y necesita una forma personalizada del token en runtime                                    |
| 24  | `refreshOAuth`                    | Reemplazo de refresh OAuth para endpoints personalizados de refresh o política de fallo de refresh | El proveedor no encaja en los refreshers compartidos `pi-ai`                                                                             |
| 25  | `buildAuthDoctorHint`             | Sugerencia de reparación añadida cuando falla el refresh OAuth                         | El proveedor necesita guía de reparación de auth propiedad del proveedor tras el fallo de refresh                                        |
| 26  | `matchesContextOverflowError`     | Detector de overflow de ventana de contexto propiedad del proveedor                    | El proveedor tiene errores crudos de overflow que las heurísticas genéricas no detectarían                                               |
| 27  | `classifyFailoverReason`          | Clasificación de motivos de failover propiedad del proveedor                           | El proveedor puede mapear errores crudos de API/transporte a rate-limit/sobrecarga/etc.                                                 |
| 28  | `isCacheTtlEligible`              | Política de caché de prompt para proveedores proxy/backhaul                            | El proveedor necesita control de TTL de caché específico de proxy                                                                        |
| 29  | `buildMissingAuthMessage`         | Sustitución del mensaje genérico de recuperación por auth faltante                     | El proveedor necesita una sugerencia específica del proveedor para auth faltante                                                          |
| 30  | `suppressBuiltInModel`            | Supresión de modelos upstream obsoletos más sugerencia opcional visible al usuario     | El proveedor necesita ocultar filas obsoletas upstream o reemplazarlas con una sugerencia del proveedor                                  |
| 31  | `augmentModelCatalog`             | Filas sintéticas/finales del catálogo agregadas después del descubrimiento             | El proveedor necesita filas sintéticas de compatibilidad futura en `models list` y selectores                                           |
| 32  | `isBinaryThinking`                | Conmutador on/off de razonamiento para proveedores de thinking binario                 | El proveedor expone solo thinking binario encendido/apagado                                                                              |
| 33  | `supportsXHighThinking`           | Compatibilidad con razonamiento `xhigh` para modelos seleccionados                     | El proveedor quiere `xhigh` solo en un subconjunto de modelos                                                                            |
| 34  | `resolveDefaultThinkingLevel`     | Nivel predeterminado de `/think` para una familia concreta de modelos                  | El proveedor posee la política predeterminada de `/think` para una familia de modelos                                                    |
| 35  | `isModernModelRef`                | Detector de modelos modernos para filtros live por perfil y selección smoke            | El proveedor posee el emparejamiento preferido de modelos para live/smoke                                                                |
| 36  | `prepareRuntimeAuth`              | Intercambia una credencial configurada por el token/clave real justo antes de la inferencia | El proveedor necesita un intercambio de token o una credencial efímera por solicitud                                                     |
| 37  | `resolveUsageAuth`                | Resuelve credenciales de uso/facturación para `/usage` y superficies de estado relacionadas | El proveedor necesita análisis personalizado del token de uso/cuota o una credencial de uso distinta                                     |
| 38  | `fetchUsageSnapshot`              | Obtiene y normaliza instantáneas de uso/cuota específicas del proveedor después de resolver auth | El proveedor necesita un endpoint de uso específico del proveedor o un parser de payload específico                                     |
| 39  | `createEmbeddingProvider`         | Construye un adaptador de embeddings propiedad del proveedor para memory/search        | El comportamiento de embeddings de memory debe vivir con el plugin del proveedor                                                          |
| 40  | `buildReplayPolicy`               | Devuelve una política de replay que controla el manejo de la transcripción del proveedor | El proveedor necesita una política personalizada de transcripción (por ejemplo, eliminar bloques de thinking)                             |
| 41  | `sanitizeReplayHistory`           | Reescribe el historial de replay después de la limpieza genérica de transcripción      | El proveedor necesita reescrituras específicas de replay más allá de los helpers compartidos de compactación                             |
| 42  | `validateReplayTurns`             | Validación o remodelado final de turnos de replay antes del embedded runner            | El transporte del proveedor necesita una validación más estricta de turnos tras el saneamiento genérico                                  |
| 43  | `onModelSelected`                 | Ejecuta efectos secundarios propiedad del proveedor después de seleccionar un modelo    | El proveedor necesita telemetría o estado propiedad del proveedor cuando un modelo se activa                                              |

`normalizeModelId`, `normalizeTransport` y `normalizeConfig` primero comprueban el
plugin de proveedor emparejado y luego recorren otros plugins de proveedor con capacidad de hook
hasta que uno realmente cambie el id del modelo o el transporte/configuración. Eso mantiene
funcionando los shims de alias/compatibilidad de proveedor sin obligar al llamador a saber qué
plugin incluido posee la reescritura. Si ningún hook de proveedor reescribe una entrada compatible
de configuración de la familia Google, el normalizador incluido de configuración Google sigue aplicando
esa limpieza de compatibilidad.

Si el proveedor necesita un protocolo wire completamente personalizado o un ejecutor de solicitud personalizado,
eso es otra clase de extensión. Estos hooks son para comportamiento de proveedor que
sigue ejecutándose sobre el bucle normal de inferencia de OpenClaw.

### Ejemplo de proveedor

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Ejemplos integrados

- Anthropic usa `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`
  y `wrapStreamFn` porque posee compatibilidad futura con Claude 4.6,
  sugerencias de familia de proveedor, guía de reparación de auth, integración de endpoint de uso,
  elegibilidad de caché de prompt, valores predeterminados de configuración conscientes de la auth,
  política predeterminada/adaptativa de thinking para Claude y modelado de stream específico de Anthropic para
  headers beta, `/fast` / `serviceTier` y `context1m`.
- Los helpers de stream específicos de Claude en Anthropic se mantienen por ahora en el
  seam público `api.ts` / `contract-api.ts` del propio plugin incluido. Esa superficie del paquete
  exporta `wrapAnthropicProviderStream`, helpers de beta-header,
  análisis de `service_tier` y builders de wrapper de Anthropic de bajo nivel, en lugar de ampliar el SDK genérico en torno a las reglas beta-header de un solo proveedor.
- OpenAI usa `resolveDynamicModel`, `normalizeResolvedModel` y
  `capabilities` más `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` e `isModernModelRef`
  porque posee compatibilidad futura con GPT-5.4, la normalización directa de OpenAI
  `openai-completions` -> `openai-responses`, pistas de auth conscientes de Codex,
  supresión de Spark, filas sintéticas de OpenAI y política de thinking /
  modelo live de GPT-5; la familia de stream `openai-responses-defaults` posee los
  wrappers nativos compartidos de OpenAI Responses para headers de atribución,
  `/fast`/`serviceTier`, verbosidad de texto, web search nativo de Codex,
  modelado de payload de compatibilidad de razonamiento y gestión de contexto de Responses.
- OpenRouter usa `catalog` más `resolveDynamicModel` y
  `prepareDynamicModel` porque el proveedor es pass-through y puede exponer nuevos
  ids de modelo antes de que se actualice el catálogo estático de OpenClaw; también usa
  `capabilities`, `wrapStreamFn` e `isCacheTtlEligible` para mantener
  fuera del núcleo los headers de solicitud específicos del proveedor, los metadatos de enrutamiento, los parches de razonamiento y
  la política de caché de prompt. Su política de replay viene de la familia
  `passthrough-gemini`, mientras que la familia de stream `openrouter-thinking`
  posee la inyección de razonamiento proxy y los saltos para modelos no compatibles / `auto`.
- GitHub Copilot usa `catalog`, `auth`, `resolveDynamicModel` y
  `capabilities` más `prepareRuntimeAuth` y `fetchUsageSnapshot` porque
  necesita login por dispositivo propiedad del proveedor, comportamiento de respaldo de modelos, particularidades
  de transcripción de Claude, un intercambio de token de GitHub -> token de Copilot y un endpoint de uso propiedad del proveedor.
- OpenAI Codex usa `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` y `augmentModelCatalog` más
  `prepareExtraParams`, `resolveUsageAuth` y `fetchUsageSnapshot` porque
  sigue ejecutándose sobre transportes centrales de OpenAI pero posee su normalización de
  transporte/base URL, política de respaldo de refresh OAuth, elección de transporte predeterminada,
  filas sintéticas del catálogo Codex e integración del endpoint de uso de ChatGPT; comparte la misma familia de stream `openai-responses-defaults` que OpenAI directo.
- Google AI Studio y Gemini CLI OAuth usan `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` e `isModernModelRef` porque la
  familia de replay `google-gemini` posee el respaldo de compatibilidad futura de Gemini 3.1,
  la validación nativa de replay de Gemini, el saneamiento de replay de bootstrap, el modo etiquetado
  de salida de razonamiento y el emparejamiento moderno de modelos, mientras que la
  familia de stream `google-thinking` posee la normalización del payload de thinking de Gemini;
  Gemini CLI OAuth también usa `formatApiKey`, `resolveUsageAuth` y
  `fetchUsageSnapshot` para formateo de tokens, análisis de tokens y
  conexión al endpoint de cuota.
- Anthropic Vertex usa `buildReplayPolicy` a través de la
  familia de replay `anthropic-by-model` para que la limpieza de replay específica de Claude quede
  limitada a ids de Claude en lugar de aplicarse a cada transporte `anthropic-messages`.
- Amazon Bedrock usa `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` y `resolveDefaultThinkingLevel` porque posee
  la clasificación específica de Bedrock de errores de throttle/not-ready/context-overflow
  para tráfico Anthropic-on-Bedrock; su política de replay sigue compartiendo la misma
  barrera solo-Claude `anthropic-by-model`.
- OpenRouter, Kilocode, Opencode y Opencode Go usan `buildReplayPolicy`
  a través de la familia de replay `passthrough-gemini` porque hacen proxy de modelos Gemini
  mediante transportes compatibles con OpenAI y necesitan saneamiento de
  firmas de thought de Gemini sin validación nativa de replay de Gemini ni reescrituras de bootstrap.
- MiniMax usa `buildReplayPolicy` a través de la
  familia de replay `hybrid-anthropic-openai` porque un proveedor posee tanto semántica
  de mensajes Anthropic como OpenAI-compatible; mantiene la eliminación de bloques de thinking solo de Claude
  en el lado Anthropic y, al mismo tiempo, vuelve a sobrescribir el modo de salida de razonamiento a nativo, y la familia de stream `minimax-fast-mode` posee las reescrituras de modelo en modo rápido en la ruta de stream compartida.
- Moonshot usa `catalog` más `wrapStreamFn` porque sigue usando el transporte
  compartido de OpenAI pero necesita normalización del payload de thinking propiedad del proveedor; la
  familia de stream `moonshot-thinking` mapea la configuración más el estado `/think` a su
  payload nativo binario de thinking.
- Kilocode usa `catalog`, `capabilities`, `wrapStreamFn` e
  `isCacheTtlEligible` porque necesita headers de solicitud propiedad del proveedor,
  normalización del payload de razonamiento, sugerencias de transcripción de Gemini y control de TTL
  de caché de Anthropic; la familia de stream `kilocode-thinking` mantiene la inyección de thinking de Kilo
  en la ruta de stream proxy compartida mientras omite `kilo/auto` y otros ids de modelo proxy que no admiten payloads explícitos de razonamiento.
- Z.AI usa `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` y `fetchUsageSnapshot` porque posee respaldo GLM-5,
  valores predeterminados `tool_stream`, UX de thinking binario, emparejamiento moderno de modelos y tanto
  auth de uso como obtención de cuota; la familia de stream `tool-stream-default-on` mantiene
  el wrapper predeterminado de `tool_stream` fuera del pegamento manuscrito por proveedor.
- xAI usa `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` e `isModernModelRef`
  porque posee normalización nativa del transporte xAI Responses, reescrituras de alias
  de Grok fast-mode, `tool_stream` predeterminado, limpieza estricta de herramientas / payload de razonamiento,
  reutilización de auth de respaldo para herramientas propiedad del plugin, resolución de modelo
  Grok con compatibilidad futura y parches de compatibilidad propiedad del proveedor como el perfil de schema de herramientas de xAI, palabras clave de schema no compatibles, `web_search` nativo y decodificación de argumentos de llamadas a herramientas con entidades HTML.
- Mistral, OpenCode Zen y OpenCode Go usan solo `capabilities` para mantener
  las particularidades de transcripción/herramientas fuera del núcleo.
- Proveedores incluidos solo de catálogo como `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` y `volcengine` usan
  solo `catalog`.
- Qwen usa `catalog` para su proveedor de texto más registros compartidos de media-understanding y
  video-generation para sus superficies multimodales.
- MiniMax y Xiaomi usan `catalog` más hooks de uso porque su comportamiento `/usage`
  es propiedad del plugin aunque la inferencia siga ejecutándose a través de los transportes compartidos.

## Helpers de tiempo de ejecución

Los plugins pueden acceder a helpers seleccionados del núcleo mediante `api.runtime`. Para TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Notas:

- `textToSpeech` devuelve la carga útil normal de salida TTS del núcleo para superficies de archivo/nota de voz.
- Usa la configuración central `messages.tts` y la selección de proveedor.
- Devuelve buffer de audio PCM + frecuencia de muestreo. Los plugins deben remuestrear/codificar para los proveedores.
- `listVoices` es opcional por proveedor. Úsalo para selectores de voz o flujos de setup propiedad del proveedor.
- Los listados de voces pueden incluir metadatos más ricos como configuración regional, género y etiquetas de personalidad para selectores conscientes del proveedor.
- OpenAI y ElevenLabs admiten telefonía hoy. Microsoft no.

Los plugins también pueden registrar proveedores de voz mediante `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Notas:

- Mantén la política TTS, el respaldo y la entrega de respuestas en el núcleo.
- Usa proveedores de voz para comportamiento de síntesis propiedad del proveedor.
- La entrada heredada `edge` de Microsoft se normaliza al id de proveedor `microsoft`.
- El modelo de propiedad preferido está orientado a empresa: un plugin de proveedor puede poseer
  texto, voz, imagen y futuros proveedores multimedia a medida que OpenClaw agregue esos
  contratos de capacidad.

Para comprensión de imagen/audio/video, los plugins registran un proveedor tipado de
media-understanding en lugar de una bolsa genérica clave/valor:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Notas:

- Mantén la orquestación, el respaldo, la configuración y el cableado del canal en el núcleo.
- Mantén el comportamiento del proveedor en el plugin del proveedor.
- La expansión aditiva debe seguir siendo tipada: nuevos métodos opcionales, nuevos campos opcionales
  de resultado, nuevas capacidades opcionales.
- La generación de video ya sigue el mismo patrón:
  - el núcleo posee el contrato de capacidad y el helper de tiempo de ejecución
  - los plugins de proveedor registran `api.registerVideoGenerationProvider(...)`
  - los plugins de función/canal consumen `api.runtime.videoGeneration.*`

Para helpers de tiempo de ejecución de media-understanding, los plugins pueden llamar a:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

Para transcripción de audio, los plugins pueden usar el runtime de media-understanding
o el alias STT más antiguo:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Opcional cuando el MIME no puede inferirse de forma fiable:
  mime: "audio/ogg",
});
```

Notas:

- `api.runtime.mediaUnderstanding.*` es la superficie compartida preferida para
  comprensión de imagen/audio/video.
- Usa la configuración central de audio de media-understanding (`tools.media.audio`) y el orden de respaldo del proveedor.
- Devuelve `{ text: undefined }` cuando no se produce salida de transcripción (por ejemplo, entrada omitida/no compatible).
- `api.runtime.stt.transcribeAudioFile(...)` permanece como alias de compatibilidad.

Los plugins también pueden lanzar ejecuciones de subagente en segundo plano mediante `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Notas:

- `provider` y `model` son reemplazos opcionales por ejecución, no cambios persistentes de sesión.
- OpenClaw solo respeta esos campos de reemplazo para llamadores de confianza.
- Para ejecuciones de respaldo propiedad del plugin, los operadores deben optar explícitamente con `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Usa `plugins.entries.<id>.subagent.allowedModels` para restringir plugins de confianza a objetivos canónicos concretos `provider/model`, o `"*"` para permitir explícitamente cualquier objetivo.
- Las ejecuciones de subagente de plugins no confiables siguen funcionando, pero las solicitudes de reemplazo se rechazan en lugar de recurrir silenciosamente a un valor predeterminado.

Para búsqueda web, los plugins pueden consumir el helper de tiempo de ejecución compartido en lugar de
acceder al cableado de la herramienta del agente:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Los plugins también pueden registrar proveedores de búsqueda web mediante
`api.registerWebSearchProvider(...)`.

Notas:

- Mantén la selección de proveedor, la resolución de credenciales y la semántica compartida de la solicitud en el núcleo.
- Usa proveedores de búsqueda web para transportes de búsqueda específicos del proveedor.
- `api.runtime.webSearch.*` es la superficie compartida preferida para plugins de función/canal que necesiten comportamiento de búsqueda sin depender del wrapper de herramienta del agente.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: genera una imagen usando la cadena configurada de proveedores de generación de imágenes.
- `listProviders(...)`: lista proveedores disponibles de generación de imágenes y sus capacidades.

## Rutas HTTP del Gateway

Los plugins pueden exponer endpoints HTTP con `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Campos de la ruta:

- `path`: ruta bajo el servidor HTTP del gateway.
- `auth`: obligatorio. Usa `"gateway"` para requerir la auth normal del gateway, o `"plugin"` para auth gestionada por el plugin / verificación de webhook.
- `match`: opcional. `"exact"` (predeterminado) o `"prefix"`.
- `replaceExisting`: opcional. Permite al mismo plugin reemplazar su propio registro de ruta existente.
- `handler`: devuelve `true` cuando la ruta ha manejado la solicitud.

Notas:

- `api.registerHttpHandler(...)` fue eliminado y provocará un error de carga del plugin. Usa `api.registerHttpRoute(...)` en su lugar.
- Las rutas de plugins deben declarar `auth` explícitamente.
- Los conflictos exactos `path + match` se rechazan salvo que `replaceExisting: true`, y un plugin no puede reemplazar la ruta de otro plugin.
- Las rutas solapadas con distintos niveles de `auth` se rechazan. Mantén las cadenas de caída `exact`/`prefix` solo en el mismo nivel de auth.
- Las rutas `auth: "plugin"` **no** reciben automáticamente alcances de tiempo de ejecución de operador. Son para webhooks/verificación de firma gestionados por el plugin, no para llamadas helper privilegiadas del Gateway.
- Las rutas `auth: "gateway"` se ejecutan dentro de un alcance de tiempo de ejecución de solicitud del Gateway, pero ese alcance es intencionalmente conservador:
  - la auth bearer de secreto compartido (`gateway.auth.mode = "token"` / `"password"`) mantiene los alcances de runtime de rutas de plugin fijados a `operator.write`, incluso si quien llama envía `x-openclaw-scopes`
  - los modos HTTP de confianza con identidad (por ejemplo `trusted-proxy` o `gateway.auth.mode = "none"` en un ingreso privado) respetan `x-openclaw-scopes` solo cuando el encabezado está explícitamente presente
  - si `x-openclaw-scopes` está ausente en esas solicitudes de rutas de plugin con identidad, el alcance de runtime recurre a `operator.write`
- Regla práctica: no asumas que una ruta de plugin con auth de gateway es una superficie implícita de administración. Si tu ruta necesita comportamiento exclusivo de admin, exige un modo de auth con identidad y documenta el contrato explícito del encabezado `x-openclaw-scopes`.

## Rutas de import del Plugin SDK

Usa subrutas del SDK en lugar del import monolítico `openclaw/plugin-sdk` al
crear plugins:

- `openclaw/plugin-sdk/plugin-entry` para primitivas de registro de plugins.
- `openclaw/plugin-sdk/core` para el contrato genérico compartido orientado a plugins.
- `openclaw/plugin-sdk/config-schema` para el export del schema Zod raíz
  de `openclaw.json` (`OpenClawSchema`).
- Primitivas estables de canal como `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input` y
  `openclaw/plugin-sdk/webhook-ingress` para cableado compartido de
  setup/auth/reply/webhook. `channel-inbound` es el hogar compartido para
  debounce, coincidencia de menciones, formato de sobres y helpers de contexto
  de sobre entrante.
  `channel-setup` es el seam estrecho de setup para instalación opcional.
  `setup-runtime` es la superficie de setup segura para runtime usada por `setupEntry` /
  inicio diferido, incluidos los adaptadores de parche de setup seguros para import.
  `setup-adapter-runtime` es el seam del adaptador de setup de cuenta consciente del entorno.
  `setup-tools` es el seam pequeño de helpers CLI/archivo/docs (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Subrutas de dominio como `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store` y
  `openclaw/plugin-sdk/directory-runtime` para helpers compartidos de runtime/configuración.
  `telegram-command-config` es el seam público estrecho para normalización/validación de comandos personalizados de Telegram y sigue disponible incluso si la superficie de contrato incluida de Telegram no está disponible temporalmente.
  `text-runtime` es el seam compartido de texto/markdown/registro, incluida la eliminación de texto visible para el asistente, helpers de renderizado/fragmentación de markdown, helpers de redacción, helpers de etiquetas de directivas y utilidades de texto seguro.
- Los seams de canal específicos de aprobación deben preferir un único contrato `approvalCapability` en el plugin. Luego, el núcleo lee auth, entrega, renderizado y comportamiento de enrutamiento nativo de aprobaciones a través de esa única capacidad en lugar de mezclar el comportamiento de aprobaciones en campos no relacionados del plugin.
- `openclaw/plugin-sdk/channel-runtime` está obsoleto y permanece solo como shim de compatibilidad para plugins más antiguos. El código nuevo debe importar las primitivas genéricas más estrechas, y el código del repositorio no debe agregar nuevos imports del shim.
- Los internos de extensiones incluidas siguen siendo privados. Los plugins externos deben usar solo subrutas `openclaw/plugin-sdk/*`. El código/pruebas del núcleo de OpenClaw puede usar los puntos de entrada públicos del repositorio bajo la raíz de un paquete de plugin como `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` y archivos de alcance estrecho como
  `login-qr-api.js`. Nunca importes `src/*` de un paquete de plugin desde el núcleo ni desde otra extensión.
- División del punto de entrada del repositorio:
  `<plugin-package-root>/api.js` es el barrel de helpers/tipos,
  `<plugin-package-root>/runtime-api.js` es el barrel solo de tiempo de ejecución,
  `<plugin-package-root>/index.js` es la entrada del plugin incluido
  y `<plugin-package-root>/setup-entry.js` es la entrada del plugin de setup.
- Ejemplos actuales de proveedores incluidos:
  - Anthropic usa `api.js` / `contract-api.js` para helpers de stream de Claude como
    `wrapAnthropicProviderStream`, helpers de beta-header y parsing de `service_tier`.
  - OpenAI usa `api.js` para builders de proveedor, helpers de modelos predeterminados y builders de proveedores realtime.
  - OpenRouter usa `api.js` para su builder de proveedor más helpers de onboarding/configuración, mientras que `register.runtime.js` puede seguir reexportando helpers genéricos `plugin-sdk/provider-stream` para uso local del repositorio.
- Los puntos de entrada públicos cargados mediante facade prefieren la instantánea activa de configuración de runtime
  cuando existe una, y luego recurren al archivo de configuración resuelto en disco cuando
  OpenClaw aún no está sirviendo una instantánea de runtime.
- Las primitivas genéricas compartidas siguen siendo el contrato público preferido del SDK. Aún existe un pequeño conjunto reservado de compatibilidad de seams helper con marca de canal incluida. Trátalos como seams de mantenimiento/compatibilidad incluidos, no como nuevos objetivos de import de terceros; los nuevos contratos entre canales deben seguir llegando a subrutas genéricas `plugin-sdk/*` o a los barrels locales `api.js` /
  `runtime-api.js` del plugin.

Nota de compatibilidad:

- Evita el barrel raíz `openclaw/plugin-sdk` en código nuevo.
- Da preferencia primero a las primitivas estables y estrechas. Las nuevas subrutas de
  setup/pairing/reply/feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool son el contrato previsto para nuevo trabajo de plugins incluidos y externos.
  El análisis/emparejamiento de targets pertenece a `openclaw/plugin-sdk/channel-targets`.
  Las barreras de acciones de mensaje y los helpers de id de mensaje de reacciones pertenecen a
  `openclaw/plugin-sdk/channel-actions`.
- Los barrels helper específicos de extensiones incluidas no son estables por defecto. Si un
  helper solo lo necesita una extensión incluida, mantenlo detrás del seam local `api.js` o `runtime-api.js` de la extensión en lugar de promoverlo a
  `openclaw/plugin-sdk/<extension>`.
- Los nuevos seams helper compartidos deben ser genéricos, no de marca de canal. El análisis compartido de target
  pertenece a `openclaw/plugin-sdk/channel-targets`; los internos específicos del canal siguen detrás del seam local `api.js` o `runtime-api.js` del plugin propietario.
- Existen subrutas específicas por capacidad como `image-generation`,
  `media-understanding` y `speech` porque los plugins nativos/incluidos las usan
  hoy. Su presencia no significa por sí sola que cada helper exportado sea un
  contrato externo congelado a largo plazo.

## Schemas de herramientas de mensajes

Los plugins deben poseer las contribuciones específicas del canal al schema de `describeMessageTool(...)`.
Mantén los campos específicos del proveedor en el plugin, no en el núcleo compartido.

Para fragmentos de schema portables compartidos, reutiliza los helpers genéricos exportados por
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` para cargas tipo cuadrícula de botones
- `createMessageToolCardSchema()` para cargas de tarjetas estructuradas

Si una forma de schema solo tiene sentido para un proveedor, defínela en el código
del propio plugin en lugar de promoverla al SDK compartido.

## Resolución de target de canal

Los plugins de canal deben poseer la semántica específica del canal para targets. Mantén
genérico el host saliente compartido y usa la superficie del adaptador de mensajería para las reglas del proveedor:

- `messaging.inferTargetChatType({ to })` decide si un target normalizado
  debe tratarse como `direct`, `group` o `channel` antes de la búsqueda en el directorio.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indica al núcleo si una
  entrada debe pasar directamente a una resolución tipo id en lugar de una búsqueda en directorio.
- `messaging.targetResolver.resolveTarget(...)` es el respaldo del plugin cuando
  el núcleo necesita una resolución final propiedad del proveedor después de la normalización o
  después de un fallo en el directorio.
- `messaging.resolveOutboundSessionRoute(...)` posee la construcción específica del proveedor
  de la ruta de sesión saliente una vez resuelto un target.

División recomendada:

- Usa `inferTargetChatType` para decisiones de categoría que deban ocurrir antes
  de buscar peers/groups.
- Usa `looksLikeId` para comprobaciones del tipo “tratar esto como un id de target explícito/nativo”.
- Usa `resolveTarget` como respaldo específico del proveedor para normalización, no para
  búsqueda amplia en directorio.
- Mantén ids nativos del proveedor como chat ids, thread ids, JIDs, handles e room ids dentro de valores `target` o parámetros específicos del proveedor, no en campos genéricos del SDK.

## Directorios respaldados por configuración

Los plugins que derivan entradas de directorio desde la configuración deben mantener esa lógica en el
plugin y reutilizar los helpers compartidos de
`openclaw/plugin-sdk/directory-runtime`.

Usa esto cuando un canal necesite peers/groups respaldados por configuración, como:

- peers DM controlados por allowlist
- mapas configurados de canal/grupo
- respaldos estáticos de directorio con alcance por cuenta

Los helpers compartidos en `directory-runtime` solo gestionan operaciones genéricas:

- filtrado de consulta
- aplicación de límites
- helpers de deduplicación/normalización
- construcción de `ChannelDirectoryEntry[]`

La inspección de cuentas específica del canal y la normalización de ids deben permanecer en la
implementación del plugin.

## Catálogos de proveedores

Los plugins de proveedor pueden definir catálogos de modelos para inferencia con
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` devuelve la misma forma que OpenClaw escribe en
`models.providers`:

- `{ provider }` para una entrada de proveedor
- `{ providers }` para varias entradas de proveedor

Usa `catalog` cuando el plugin posea ids de modelo específicos del proveedor, valores predeterminados de base URL
o metadatos de modelos condicionados por auth.

`catalog.order` controla cuándo se fusiona el catálogo de un plugin con respecto a
los proveedores implícitos integrados de OpenClaw:

- `simple`: proveedores simples controlados por API key o entorno
- `profile`: proveedores que aparecen cuando existen perfiles de auth
- `paired`: proveedores que sintetizan varias entradas de proveedor relacionadas
- `late`: último paso, después de otros proveedores implícitos

Los proveedores posteriores ganan en colisión de clave, así que los plugins pueden
sobrescribir intencionalmente una entrada integrada de proveedor con el mismo id de proveedor.

Compatibilidad:

- `discovery` sigue funcionando como alias heredado
- si se registran ambos `catalog` y `discovery`, OpenClaw usa `catalog`

## Inspección de canal de solo lectura

Si tu plugin registra un canal, prefiere implementar
`plugin.config.inspectAccount(cfg, accountId)` junto con `resolveAccount(...)`.

Por qué:

- `resolveAccount(...)` es la ruta de runtime. Puede asumir que las credenciales
  están completamente materializadas y puede fallar rápidamente cuando faltan secretos requeridos.
- Las rutas de comandos de solo lectura como `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` y los flujos de doctor/config repair
  no deberían necesitar materializar credenciales de runtime solo para describir la configuración.

Comportamiento recomendado de `inspectAccount(...)`:

- Devuelve solo estado descriptivo de la cuenta.
- Conserva `enabled` y `configured`.
- Incluye campos de origen/estado de credenciales cuando sea relevante, como:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- No necesitas devolver valores brutos de tokens solo para informar disponibilidad de solo lectura. Devolver `tokenStatus: "available"` (y el campo de origen correspondiente) es suficiente para comandos tipo status.
- Usa `configured_unavailable` cuando una credencial esté configurada mediante SecretRef pero
  no disponible en la ruta de comandos actual.

Esto permite que los comandos de solo lectura informen “configured but unavailable in this command
path” en lugar de fallar o informar incorrectamente que la cuenta no está configurada.

## Package packs

Un directorio de plugin puede incluir un `package.json` con `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Cada entrada se convierte en un plugin. Si el paquete lista varias extensiones, el id del plugin
pasa a ser `name/<fileBase>`.

Si tu plugin importa dependencias npm, instálalas en ese directorio para que
`node_modules` esté disponible (`npm install` / `pnpm install`).

Barrera de seguridad: cada entrada `openclaw.extensions` debe permanecer dentro del directorio del plugin
después de resolver symlinks. Las entradas que escapen del directorio del paquete se
rechazan.

Nota de seguridad: `openclaw plugins install` instala dependencias de plugins con
`npm install --omit=dev --ignore-scripts` (sin lifecycle scripts y sin dev dependencies en runtime). Mantén los árboles de dependencias de plugins como "pure JS/TS" y evita paquetes que requieran builds en `postinstall`.

Opcional: `openclaw.setupEntry` puede apuntar a un módulo ligero solo de setup.
Cuando OpenClaw necesita superficies de setup para un plugin de canal deshabilitado, o
cuando un plugin de canal está habilitado pero aún no configurado, carga `setupEntry`
en lugar de la entrada completa del plugin. Esto mantiene el inicio y el setup más ligeros
cuando la entrada principal del plugin también conecta herramientas, hooks u otro código solo de runtime.

Opcional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
puede hacer que un plugin de canal opte por la misma ruta `setupEntry` durante la
fase de inicio previa a listen del gateway, incluso cuando el canal ya está configurado.

Usa esto solo cuando `setupEntry` cubra completamente la superficie de inicio que debe existir
antes de que el gateway empiece a escuchar. En la práctica, eso significa que la entrada de setup
debe registrar todas las capacidades propiedad del canal de las que depende el inicio, como:

- el propio registro del canal
- cualquier ruta HTTP que deba estar disponible antes de que el gateway empiece a escuchar
- cualquier método del gateway, herramienta o servicio que deba existir durante esa misma ventana

Si tu entrada completa todavía posee alguna capacidad de inicio requerida, no habilites
esta bandera. Mantén el plugin con el comportamiento predeterminado y deja que OpenClaw cargue la
entrada completa durante el inicio.

Los canales incluidos también pueden publicar helpers de superficie de contrato solo de setup que el núcleo
puede consultar antes de que se cargue el runtime completo del canal. La superficie actual de promoción de setup es:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

El núcleo usa esa superficie cuando necesita promover una configuración heredada de canal de una sola cuenta a
`channels.<id>.accounts.*` sin cargar la entrada completa del plugin.
Matrix es el ejemplo incluido actual: mueve solo claves de auth/bootstrap a una
cuenta promocionada con nombre cuando ya existen cuentas con nombre, y puede conservar una
clave de cuenta predeterminada configurada no canónica en lugar de crear siempre
`accounts.default`.

Esos adaptadores de parche de setup mantienen lazy el descubrimiento de la superficie de contrato incluida.
El tiempo de importación sigue siendo ligero; la superficie de promoción se carga solo en el primer uso en lugar de volver a entrar en el inicio del canal incluido en la importación del módulo.

Cuando esas superficies de inicio incluyen métodos RPC del gateway, mantenlos en un
prefijo específico del plugin. Los espacios de nombres centrales de administración (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) siguen reservados y siempre se resuelven
a `operator.admin`, incluso si un plugin solicita un alcance más estrecho.

Ejemplo:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadatos de catálogo de canales

Los plugins de canal pueden anunciar metadatos de setup/descubrimiento mediante `openclaw.channel` e
instrucciones de instalación mediante `openclaw.install`. Esto mantiene el catálogo central sin datos.

Ejemplo:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Campos útiles de `openclaw.channel` además del ejemplo mínimo:

- `detailLabel`: etiqueta secundaria para superficies más ricas de catálogo/status
- `docsLabel`: sobrescribe el texto del enlace de documentación
- `preferOver`: ids de plugin/canal de menor prioridad a los que esta entrada del catálogo debe adelantar
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: controles de copia para superficies de selección
- `markdownCapable`: marca el canal como compatible con markdown para decisiones de formato saliente
- `showConfigured`: oculta el canal en superficies de listado de canales configurados cuando se define en `false`
- `quickstartAllowFrom`: hace que el canal opte por el flujo estándar quickstart de `allowFrom`
- `forceAccountBinding`: requiere binding explícito de cuenta incluso cuando solo existe una cuenta
- `preferSessionLookupForAnnounceTarget`: prefiere la búsqueda de sesión al resolver objetivos de announce

OpenClaw también puede fusionar **catálogos externos de canales** (por ejemplo, una exportación de un
registro MPM). Deja un archivo JSON en una de estas rutas:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

O apunta `OPENCLAW_PLUGIN_CATALOG_PATHS` (o `OPENCLAW_MPM_CATALOG_PATHS`) a
uno o más archivos JSON (delimitados por coma/punto y coma/`PATH`). Cada archivo debe
contener `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. El analizador también acepta `"packages"` o `"plugins"` como alias heredados de la clave `"entries"`.

## Plugins de motor de contexto

Los plugins de motor de contexto poseen la orquestación del contexto de sesión para ingestión,
ensamblaje y compactación. Regístralos desde tu plugin con
`api.registerContextEngine(id, factory)` y luego selecciona el motor activo con
`plugins.slots.contextEngine`.

Usa esto cuando tu plugin necesite reemplazar o ampliar el pipeline de contexto predeterminado
en lugar de solo agregar búsqueda en memory o hooks.

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Si tu motor **no** posee el algoritmo de compactación, mantén `compact()`
implementado y delégalo explícitamente:

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Agregar una nueva capacidad

Cuando un plugin necesite un comportamiento que no encaje en la API actual, no omitas
el sistema de plugins con un acceso privado. Agrega la capacidad que falta.

Secuencia recomendada:

1. define el contrato central
   Decide qué comportamiento compartido debe poseer el núcleo: política, respaldo, fusión de configuración,
   ciclo de vida, semántica orientada a canales y forma del helper de tiempo de ejecución.
2. agrega superficies tipadas de registro/runtime para plugins
   Amplía `OpenClawPluginApi` y/o `api.runtime` con la superficie tipada de capacidad
   más pequeña que sea útil.
3. conecta consumidores del núcleo + canales/funciones
   Los canales y plugins de función deben consumir la nueva capacidad a través del núcleo,
   no importando directamente una implementación del proveedor.
4. registra implementaciones de proveedores
   Luego los plugins de proveedor registran sus backends contra la capacidad.
5. agrega cobertura de contrato
   Agrega pruebas para que la propiedad y la forma del registro sigan siendo explícitas con el tiempo.

Así es como OpenClaw se mantiene con criterio sin quedar codificado rígidamente a la
visión del mundo de un único proveedor. Consulta [Capability Cookbook](/tools/capability-cookbook)
para una lista concreta de archivos y un ejemplo trabajado.

### Lista de verificación de capacidades

Cuando agregues una nueva capacidad, la implementación normalmente debe tocar estas
superficies en conjunto:

- tipos del contrato central en `src/<capability>/types.ts`
- ejecutor/helper de runtime central en `src/<capability>/runtime.ts`
- superficie de registro de la API de plugins en `src/plugins/types.ts`
- cableado del registro de plugins en `src/plugins/registry.ts`
- exposición de runtime de plugins en `src/plugins/runtime/*` cuando los plugins de función/canal
  necesiten consumirla
- helpers de captura/prueba en `src/test-utils/plugin-registration.ts`
- afirmaciones de propiedad/contrato en `src/plugins/contracts/registry.ts`
- documentación para operadores/plugins en `docs/`

Si falta una de esas superficies, normalmente es señal de que la capacidad
aún no está completamente integrada.

### Plantilla de capacidad

Patrón mínimo:

```ts
// contrato central
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// API de plugin
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// helper de runtime compartido para plugins de función/canal
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Patrón de prueba de contrato:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Eso mantiene la regla simple:

- el núcleo posee el contrato de capacidad + la orquestación
- los plugins de proveedor poseen las implementaciones del proveedor
- los plugins de función/canal consumen helpers de runtime
- las pruebas de contrato mantienen explícita la propiedad
