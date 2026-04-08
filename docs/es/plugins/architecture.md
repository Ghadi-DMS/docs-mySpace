---
read_when:
    - Crear o depurar plugins nativos de OpenClaw
    - Entender el modelo de capacidades de los plugins o los límites de propiedad
    - Trabajar en la canalización de carga o el registro de plugins
    - Implementar hooks de runtime de proveedores o plugins de canal
sidebarTitle: Internals
summary: 'Aspectos internos de los plugins: modelo de capacidades, propiedad, contratos, canalización de carga y helpers de runtime'
title: Aspectos internos de los plugins
x-i18n:
    generated_at: "2026-04-08T02:20:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: c40ecf14e2a0b2b8d332027aed939cd61fb4289a489f4cd4c076c96d707d1138
    source_path: plugins/architecture.md
    workflow: 15
---

# Aspectos internos de los plugins

<Info>
  Esta es la **referencia de arquitectura detallada**. Para guías prácticas, consulta:
  - [Install and use plugins](/es/tools/plugin) — guía del usuario
  - [Getting Started](/es/plugins/building-plugins) — primer tutorial de plugins
  - [Channel Plugins](/es/plugins/sdk-channel-plugins) — crea un canal de mensajería
  - [Provider Plugins](/es/plugins/sdk-provider-plugins) — crea un proveedor de modelos
  - [SDK Overview](/es/plugins/sdk-overview) — mapa de importaciones y API de registro
</Info>

Esta página cubre la arquitectura interna del sistema de plugins de OpenClaw.

## Modelo público de capacidades

Las capacidades son el modelo público de **plugin nativo** dentro de OpenClaw. Cada
plugin nativo de OpenClaw se registra en uno o más tipos de capacidad:

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Inferencia de texto         | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| Backend de inferencia CLI  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Voz                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Transcripción en tiempo real | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Voz en tiempo real         | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Comprensión multimedia    | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Generación de imágenes       | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Generación de música       | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Generación de video       | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Recuperación web              | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Búsqueda web             | `api.registerWebSearchProvider(...)`             | `google`                             |
| Canal / mensajería    | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

Un plugin que registra cero capacidades pero proporciona hooks, herramientas o
servicios es un plugin **heredado solo de hooks**. Ese patrón sigue siendo totalmente compatible.

### Postura de compatibilidad externa

El modelo de capacidades ya está implementado en el núcleo y hoy lo usan los plugins
incluidos/nativos, pero la compatibilidad de plugins externos aún necesita una barra más alta que “está
exportado, por lo tanto está congelado”.

Guía actual:

- **plugins externos existentes:** mantén funcionando las integraciones basadas en hooks; trátalo
  como la base de compatibilidad
- **nuevos plugins incluidos/nativos:** prefiere el registro explícito de capacidades en lugar de
  accesos específicos de proveedor o nuevos diseños solo con hooks
- **plugins externos que adopten el registro de capacidades:** permitido, pero trata las
  superficies helper específicas de capacidades como evolutivas salvo que la documentación marque explícitamente un contrato como estable

Regla práctica:

- las API de registro de capacidades son la dirección prevista
- los hooks heredados siguen siendo la ruta más segura para evitar rupturas en plugins externos durante
  la transición
- no todas las subrutas helper exportadas son iguales; prefiere el contrato documentado y limitado,
  no exportaciones helper incidentales

### Formas de plugins

OpenClaw clasifica cada plugin cargado en una forma según su comportamiento real
de registro (no solo según los metadatos estáticos):

- **plain-capability** -- registra exactamente un tipo de capacidad (por ejemplo, un
  plugin solo de proveedor como `mistral`)
- **hybrid-capability** -- registra varios tipos de capacidad (por ejemplo,
  `openai` posee inferencia de texto, voz, comprensión multimedia y generación
  de imágenes)
- **hook-only** -- registra solo hooks (tipados o personalizados), sin capacidades,
  herramientas, comandos ni servicios
- **non-capability** -- registra herramientas, comandos, servicios o rutas, pero no
  capacidades

Usa `openclaw plugins inspect <id>` para ver la forma de un plugin y el desglose
de capacidades. Consulta [CLI reference](/cli/plugins#inspect) para más detalles.

### Hooks heredados

El hook `before_agent_start` sigue siendo compatible como vía de compatibilidad para
plugins solo de hooks. Los plugins heredados reales siguen dependiendo de él.

Dirección:

- mantenerlo funcionando
- documentarlo como heredado
- preferir `before_model_resolve` para el trabajo de sobrescritura de modelo/proveedor
- preferir `before_prompt_build` para la mutación de prompts
- eliminarlo solo cuando baje el uso real y la cobertura de fixtures demuestre la seguridad de la migración

### Señales de compatibilidad

Cuando ejecutes `openclaw doctor` o `openclaw plugins inspect <id>`, puedes ver
una de estas etiquetas:

| Signal                     | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | La configuración se analiza correctamente y los plugins se resuelven                       |
| **compatibility advisory** | El plugin usa un patrón compatible pero antiguo (por ejemplo, `hook-only`) |
| **legacy warning**         | El plugin usa `before_agent_start`, que está obsoleto        |
| **hard error**             | La configuración no es válida o el plugin no se pudo cargar                   |

Ni `hook-only` ni `before_agent_start` romperán tu plugin hoy --
`hook-only` es informativo, y `before_agent_start` solo activa una advertencia. Estas
señales también aparecen en `openclaw status --all` y `openclaw plugins doctor`.

## Resumen de arquitectura

El sistema de plugins de OpenClaw tiene cuatro capas:

1. **Manifest + discovery**
   OpenClaw encuentra plugins candidatos a partir de rutas configuradas, raíces del espacio de trabajo,
   raíces globales de extensiones y extensiones incluidas. El descubrimiento lee primero
   los manifests nativos `openclaw.plugin.json` y los manifests de paquetes compatibles.
2. **Enablement + validation**
   El núcleo decide si un plugin descubierto está habilitado, deshabilitado, bloqueado o
   seleccionado para una ranura exclusiva como memoria.
3. **Runtime loading**
   Los plugins nativos de OpenClaw se cargan dentro del proceso mediante jiti y registran
   capacidades en un registro central. Los paquetes compatibles se normalizan en
   registros del registro sin importar código de runtime.
4. **Surface consumption**
   El resto de OpenClaw lee el registro para exponer herramientas, canales, configuración
   de proveedores, hooks, rutas HTTP, comandos CLI y servicios.

En el caso específico del CLI de plugins, el descubrimiento de comandos raíz se divide en dos fases:

- los metadatos en tiempo de análisis provienen de `registerCli(..., { descriptors: [...] })`
- el módulo CLI real del plugin puede seguir siendo perezoso y registrarse en la primera invocación

Eso mantiene el código del CLI propiedad del plugin dentro del propio plugin y aun así permite que OpenClaw
reserve nombres de comandos raíz antes del análisis.

El límite de diseño importante:

- el descubrimiento y la validación de configuración deben funcionar a partir de **metadatos de manifest/esquema**
  sin ejecutar código del plugin
- el comportamiento nativo de runtime proviene de la ruta `register(api)` del módulo del plugin

Esa separación permite a OpenClaw validar configuración, explicar plugins faltantes/deshabilitados y
crear pistas para la UI/esquema antes de que el runtime completo esté activo.

### Plugins de canal y la herramienta compartida de mensajes

Los plugins de canal no necesitan registrar una herramienta independiente para enviar/editar/reaccionar en
acciones normales de chat. OpenClaw mantiene una única herramienta compartida `message` en el núcleo, y
los plugins de canal poseen el descubrimiento y la ejecución específicos del canal detrás de ella.

El límite actual es:

- el núcleo posee el host de la herramienta compartida `message`, la integración con prompts, la
  contabilidad de sesión/hilo y el despacho de ejecución
- los plugins de canal poseen el descubrimiento de acciones con alcance, el descubrimiento de capacidades
  y cualquier fragmento de esquema específico del canal
- los plugins de canal poseen la gramática de conversación de sesión específica del proveedor, como
  cómo los ids de conversación codifican ids de hilo o heredan de conversaciones padre
- los plugins de canal ejecutan la acción final a través de su adaptador de acciones

Para los plugins de canal, la superficie del SDK es
`ChannelMessageActionAdapter.describeMessageTool(...)`. Esa llamada unificada de descubrimiento
permite que un plugin devuelva sus acciones visibles, capacidades y contribuciones de esquema
juntas para que esas piezas no se desincronicen.

El núcleo pasa el alcance de runtime a ese paso de descubrimiento. Los campos importantes incluyen:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` entrante de confianza

Eso importa para los plugins sensibles al contexto. Un canal puede ocultar o exponer
acciones de mensajes según la cuenta activa, la sala/hilo/mensaje actual o la identidad
confiable del solicitante, sin codificar ramas específicas del canal en la herramienta
`message` del núcleo.

Por eso los cambios de enrutamiento del ejecutor incrustado siguen siendo trabajo del plugin: el ejecutor
es responsable de reenviar la identidad actual del chat/sesión al límite de descubrimiento del plugin
para que la herramienta compartida `message` exponga la superficie propiedad del canal correcta para el turno actual.

Para los helpers de ejecución propiedad del canal, los plugins incluidos deben mantener el runtime de ejecución
dentro de sus propios módulos de extensión. El núcleo ya no posee los runtimes de acciones de mensajes
de Discord, Slack, Telegram o WhatsApp en `src/agents/tools`.
No publicamos subrutas `plugin-sdk/*-action-runtime` separadas, y los plugins incluidos
deben importar directamente su propio código de runtime local desde sus
módulos propiedad de la extensión.

El mismo límite se aplica a las uniones del SDK con nombre de proveedor en general: el núcleo
no debe importar barriles de conveniencia específicos de canal para Slack, Discord, Signal,
WhatsApp o extensiones similares. Si el núcleo necesita un comportamiento, debe
consumir el barril propio `api.ts` / `runtime-api.ts` del plugin incluido o promover la necesidad
a una capacidad genérica y estrecha en el SDK compartido.

En el caso específico de las encuestas, hay dos rutas de ejecución:

- `outbound.sendPoll` es la base compartida para canales que encajan en el modelo
  común de encuestas
- `actions.handleAction("poll")` es la ruta preferida para semánticas de encuestas específicas del canal
  o parámetros adicionales de encuestas

El núcleo ahora difiere el análisis compartido de encuestas hasta después de que el despacho de encuestas del plugin
rechace la acción, para que los controladores de encuestas propiedad del plugin puedan aceptar
campos específicos del canal sin quedar bloqueados primero por el analizador genérico de encuestas.

Consulta [Load pipeline](#load-pipeline) para la secuencia completa de inicio.

## Modelo de propiedad de capacidades

OpenClaw trata un plugin nativo como el límite de propiedad de una **empresa** o una
**función**, no como un conjunto desordenado de integraciones no relacionadas.

Eso significa:

- un plugin de empresa normalmente debe poseer todas las superficies de OpenClaw orientadas a esa empresa
- un plugin de función normalmente debe poseer toda la superficie de la función que introduce
- los canales deben consumir capacidades compartidas del núcleo en lugar de reimplementar
  comportamiento de proveedores de forma ad hoc

Ejemplos:

- el plugin incluido `openai` posee el comportamiento de proveedor de modelos de OpenAI y el comportamiento de OpenAI
  en voz + voz en tiempo real + comprensión multimedia + generación de imágenes
- el plugin incluido `elevenlabs` posee el comportamiento de voz de ElevenLabs
- el plugin incluido `microsoft` posee el comportamiento de voz de Microsoft
- el plugin incluido `google` posee el comportamiento de proveedor de modelos de Google además de Google
  en comprensión multimedia + generación de imágenes + búsqueda web
- el plugin incluido `firecrawl` posee el comportamiento de recuperación web de Firecrawl
- los plugins incluidos `minimax`, `mistral`, `moonshot` y `zai` poseen sus
  backends de comprensión multimedia
- el plugin incluido `qwen` posee el comportamiento de proveedor de texto de Qwen más
  comprensión multimedia y generación de video
- el plugin `voice-call` es un plugin de función: posee transporte de llamadas, herramientas,
  CLI, rutas y puente de flujo multimedia de Twilio, pero consume voz compartida
  más capacidades de transcripción en tiempo real y voz en tiempo real en lugar de
  importar directamente plugins de proveedores

El estado final deseado es:

- OpenAI vive en un único plugin aunque abarque modelos de texto, voz, imágenes y
  video futuro
- otro proveedor puede hacer lo mismo para su propia superficie
- a los canales no les importa qué plugin de proveedor posee el proveedor; consumen el
  contrato de capacidad compartida expuesto por el núcleo

Esta es la distinción clave:

- **plugin** = límite de propiedad
- **capacidad** = contrato del núcleo que varios plugins pueden implementar o consumir

Así, si OpenClaw agrega un nuevo dominio como video, la primera pregunta no es
“¿qué proveedor debe codificar el manejo de video?”. La primera pregunta es “¿cuál es
el contrato central de capacidad de video?”. Una vez que ese contrato exista, los plugins de proveedores
pueden registrarse contra él y los plugins de canal/función pueden consumirlo.

Si la capacidad todavía no existe, el movimiento correcto normalmente es:

1. definir la capacidad faltante en el núcleo
2. exponerla a través de la API/runtime del plugin de forma tipada
3. conectar canales/funciones a esa capacidad
4. dejar que los plugins de proveedores registren implementaciones

Esto mantiene la propiedad explícita y evita al mismo tiempo un comportamiento del núcleo que dependa
de un único proveedor o de una ruta de código específica para un plugin puntual.

### Estratificación de capacidades

Usa este modelo mental al decidir dónde debe ir el código:

- **capa de capacidad del núcleo**: orquestación compartida, política, fallback, configuración
  reglas de fusión, semántica de entrega y contratos tipados
- **capa de plugin de proveedor**: API específicas del proveedor, autenticación, catálogos de modelos,
  síntesis de voz, generación de imágenes, futuros backends de video, endpoints de uso
- **capa de plugin de canal/función**: integración de Slack/Discord/voice-call/etc.
  que consume capacidades del núcleo y las presenta en una superficie

Por ejemplo, TTS sigue esta forma:

- el núcleo posee la política de TTS en tiempo de respuesta, el orden de fallback, las preferencias y la entrega por canal
- `openai`, `elevenlabs` y `microsoft` poseen las implementaciones de síntesis
- `voice-call` consume el helper de runtime de TTS para telefonía

Ese mismo patrón debe preferirse para capacidades futuras.

### Ejemplo de plugin de empresa con varias capacidades

Un plugin de empresa debe sentirse cohesivo desde fuera. Si OpenClaw tiene contratos compartidos
para modelos, voz, transcripción en tiempo real, voz en tiempo real, comprensión multimedia,
generación de imágenes, generación de video, recuperación web y búsqueda web,
un proveedor puede poseer todas sus superficies en un solo lugar:

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
      // hooks de autenticación/catálogo de modelos/runtime
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

Lo importante no son los nombres exactos de los helpers. Lo importante es la forma:

- un plugin posee la superficie del proveedor
- el núcleo sigue poseyendo los contratos de capacidad
- los canales y plugins de funciones consumen helpers `api.runtime.*`, no código del proveedor
- las pruebas de contrato pueden afirmar que el plugin registró las capacidades que
  dice poseer

### Ejemplo de capacidad: comprensión de video

OpenClaw ya trata la comprensión de imagen/audio/video como una única
capacidad compartida. Aquí se aplica el mismo modelo de propiedad:

1. el núcleo define el contrato de comprensión multimedia
2. los plugins de proveedores registran `describeImage`, `transcribeAudio` y
   `describeVideo` según corresponda
3. los plugins de canal y funciones consumen el comportamiento compartido del núcleo en lugar de
   conectarse directamente a código del proveedor

Eso evita incorporar al núcleo las suposiciones de video de un único proveedor. El plugin posee
la superficie del proveedor; el núcleo posee el contrato de capacidad y el comportamiento de fallback.

La generación de video ya usa esa misma secuencia: el núcleo posee el
contrato tipado de capacidad y el helper de runtime, y los plugins de proveedores registran
implementaciones `api.registerVideoGenerationProvider(...)` contra él.

¿Necesitas una lista concreta de despliegue? Consulta
[Capability Cookbook](/es/plugins/architecture).

## Contratos y aplicación

La superficie de la API de plugins es intencionalmente tipada y está centralizada en
`OpenClawPluginApi`. Ese contrato define los puntos de registro compatibles y
los helpers de runtime de los que un plugin puede depender.

Por qué esto importa:

- los autores de plugins obtienen un único estándar interno estable
- el núcleo puede rechazar propiedad duplicada, como dos plugins que registren el mismo
  id de proveedor
- el inicio puede mostrar diagnósticos accionables para registros mal formados
- las pruebas de contrato pueden aplicar la propiedad de plugins incluidos y evitar una deriva silenciosa

Hay dos capas de aplicación:

1. **aplicación del registro en runtime**
   El registro de plugins valida los registros a medida que se cargan los plugins. Ejemplos:
   ids de proveedor duplicados, ids de proveedor de voz duplicados y registros
   mal formados producen diagnósticos del plugin en lugar de un comportamiento indefinido.
2. **pruebas de contrato**
   Los plugins incluidos se capturan en registros de contrato durante las ejecuciones de prueba para que
   OpenClaw pueda afirmar explícitamente la propiedad. Hoy esto se usa para proveedores de modelos,
   proveedores de voz, proveedores de búsqueda web y propiedad de registros incluidos.

El efecto práctico es que OpenClaw sabe, por adelantado, qué plugin posee qué
superficie. Eso permite que el núcleo y los canales compongan sin fricciones porque la propiedad está
declarada, tipada y es comprobable en lugar de implícita.

### Qué pertenece a un contrato

Los buenos contratos de plugins son:

- tipados
- pequeños
- específicos de capacidad
- propiedad del núcleo
- reutilizables por varios plugins
- consumibles por canales/funciones sin conocimiento del proveedor

Los malos contratos de plugins son:

- política específica del proveedor oculta en el núcleo
- vías de escape puntuales para plugins que eluden el registro
- código de canal que accede directamente a una implementación de proveedor
- objetos de runtime ad hoc que no forman parte de `OpenClawPluginApi` o
  `api.runtime`

En caso de duda, eleva el nivel de abstracción: define primero la capacidad y luego
deja que los plugins se conecten a ella.

## Modelo de ejecución

Los plugins nativos de OpenClaw se ejecutan **dentro del proceso** con el Gateway. No están
aislados. Un plugin nativo cargado tiene el mismo límite de confianza a nivel de proceso que
el código del núcleo.

Implicaciones:

- un plugin nativo puede registrar herramientas, manejadores de red, hooks y servicios
- un error en un plugin nativo puede hacer caer o desestabilizar el gateway
- un plugin nativo malicioso equivale a ejecución arbitraria de código dentro
  del proceso de OpenClaw

Los paquetes compatibles son más seguros por defecto porque OpenClaw actualmente los trata
como paquetes de metadatos/contenido. En las versiones actuales, eso significa sobre todo
Skills incluidos.

Usa listas permitidas y rutas explícitas de instalación/carga para plugins no incluidos.
Trata los plugins del espacio de trabajo como código de desarrollo, no como valores predeterminados de producción.

Para los nombres de paquetes del espacio de trabajo incluidos, mantén el id del plugin anclado en el
nombre npm: `@openclaw/<id>` de forma predeterminada, o un sufijo tipado aprobado como
`-provider`, `-plugin`, `-speech`, `-sandbox` o `-media-understanding` cuando
el paquete exponga intencionalmente un rol de plugin más limitado.

Nota importante de confianza:

- `plugins.allow` confía en **ids de plugin**, no en la procedencia de la fuente.
- Un plugin del espacio de trabajo con el mismo id que un plugin incluido
  sombrea intencionalmente la copia incluida cuando ese plugin del espacio de trabajo está habilitado/en lista permitida.
- Esto es normal y útil para desarrollo local, pruebas de parches y hotfixes.

## Límite de exportación

OpenClaw exporta capacidades, no comodidades de implementación.

Mantén público el registro de capacidades. Recorta las exportaciones helper no contractuales:

- subrutas específicas de plugins incluidos
- subrutas de infraestructura de runtime no destinadas a ser API pública
- helpers de conveniencia específicos del proveedor
- helpers de configuración/onboarding que son detalles de implementación

Algunas subrutas helper de plugins incluidos todavía permanecen en el mapa de exportación del SDK
generado por compatibilidad y mantenimiento de plugins incluidos. Algunos ejemplos actuales son
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` y varias uniones `plugin-sdk/matrix*`. Trátalas como
exportaciones reservadas de detalle de implementación, no como el patrón recomendado del SDK para
nuevos plugins de terceros.

## Canalización de carga

En el inicio, OpenClaw hace aproximadamente esto:

1. descubrir raíces candidatas de plugins
2. leer manifests nativos o compatibles y metadatos de paquetes
3. rechazar candidatos inseguros
4. normalizar la configuración de plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decidir la habilitación para cada candidato
6. cargar módulos nativos habilitados mediante jiti
7. llamar a los hooks nativos `register(api)` (o `activate(api)` — un alias heredado) y recopilar registros en el registro de plugins
8. exponer el registro a comandos/superficies de runtime

<Note>
`activate` es un alias heredado de `register`: el cargador resuelve el que esté presente (`def.register ?? def.activate`) y lo llama en el mismo punto. Todos los plugins incluidos usan `register`; prefiere `register` para los plugins nuevos.
</Note>

Las compuertas de seguridad ocurren **antes** de la ejecución del runtime. Los candidatos se bloquean
cuando la entrada escapa de la raíz del plugin, la ruta tiene permisos de escritura globales o la
propiedad de la ruta parece sospechosa en plugins no incluidos.

### Comportamiento manifest-first

El manifest es la fuente de verdad del plano de control. OpenClaw lo usa para:

- identificar el plugin
- descubrir canales/Skills/esquema de configuración declarados o capacidades del paquete
- validar `plugins.entries.<id>.config`
- ampliar etiquetas/placeholders de Control UI
- mostrar metadatos de instalación/catálogo

Para los plugins nativos, el módulo de runtime es la parte del plano de datos. Registra
el comportamiento real, como hooks, herramientas, comandos o flujos de proveedores.

### Qué almacena en caché el cargador

OpenClaw mantiene cachés cortas dentro del proceso para:

- resultados de descubrimiento
- datos del registro de manifests
- registros de plugins cargados

Estas cachés reducen el inicio brusco y la sobrecarga de comandos repetidos. Es seguro
pensar en ellas como cachés de rendimiento de corta duración, no como persistencia.

Nota de rendimiento:

- Configura `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` o
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` para desactivar estas cachés.
- Ajusta las ventanas de caché con `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` y
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Modelo de registro

Los plugins cargados no mutan directamente variables globales aleatorias del núcleo. Se registran en un
registro central de plugins.

El registro rastrea:

- registros de plugins (identidad, origen, procedencia, estado, diagnósticos)
- herramientas
- hooks heredados y hooks tipados
- canales
- proveedores
- manejadores RPC del gateway
- rutas HTTP
- registradores CLI
- servicios en segundo plano
- comandos propiedad del plugin

Las funciones del núcleo luego leen de ese registro en lugar de hablar directamente con los módulos
del plugin. Esto mantiene la carga en una sola dirección:

- módulo de plugin -> registro en el registro
- runtime del núcleo -> consumo del registro

Esa separación importa para la mantenibilidad. Significa que la mayoría de las superficies del núcleo solo
necesitan un punto de integración: “leer el registro”, no “hacer casos especiales para cada módulo
de plugin”.

## Callbacks de vinculación de conversaciones

Los plugins que vinculan una conversación pueden reaccionar cuando se resuelve una aprobación.

Usa `api.onConversationBindingResolved(...)` para recibir un callback después de que una solicitud de vinculación sea aprobada o denegada:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Ahora existe una vinculación para este plugin + conversación.
        console.log(event.binding?.conversationId);
        return;
      }

      // La solicitud fue denegada; limpia cualquier estado local pendiente.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Campos de la carga útil del callback:

- `status`: `"approved"` o `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` o `"deny"`
- `binding`: la vinculación resuelta para solicitudes aprobadas
- `request`: el resumen de la solicitud original, indicación de desvinculación, id del remitente y
  metadatos de conversación

Este callback es solo de notificación. No cambia quién puede vincular una
conversación y se ejecuta después de que finalice el manejo de aprobación del núcleo.

## Hooks de runtime del proveedor

Los plugins de proveedores ahora tienen dos capas:

- metadatos del manifest: `providerAuthEnvVars` para una búsqueda económica de autenticación de proveedor basada en variables de entorno
  antes de cargar el runtime, `channelEnvVars` para una búsqueda económica de canal/env/configuración
  antes de cargar el runtime, además de `providerAuthChoices` para etiquetas económicas
  de onboarding/elección de autenticación y metadatos de banderas CLI antes de cargar el runtime
- hooks en tiempo de configuración: `catalog` / heredado `discovery` más `applyConfigDefaults`
- hooks de runtime: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
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

OpenClaw sigue poseyendo el bucle genérico del agente, el failover, el manejo de transcripciones y la
política de herramientas. Estos hooks son la superficie de extensión para el comportamiento específico del proveedor sin
necesitar un transporte de inferencia totalmente personalizado.

Usa el manifest `providerAuthEnvVars` cuando el proveedor tenga credenciales basadas en variables de entorno
que las rutas genéricas de autenticación/estado/selector de modelos deban ver sin cargar el runtime del plugin.
Usa el manifest `providerAuthChoices` cuando las superficies CLI de onboarding/elección de autenticación
deban conocer el id de elección del proveedor, las etiquetas de grupo y la integración simple
de autenticación con una sola bandera sin cargar el runtime del proveedor. Mantén `envVars` en el runtime del proveedor
para pistas orientadas al operador, como etiquetas de onboarding o variables de
configuración de client-id/client-secret de OAuth.

Usa el manifest `channelEnvVars` cuando un canal tenga autenticación o configuración impulsadas por entorno que
las rutas genéricas de fallback de shell-env, comprobaciones de config/estado o prompts de configuración deban ver
sin cargar el runtime del canal.

### Orden de hooks y uso

Para plugins de modelos/proveedores, OpenClaw llama a los hooks en este orden aproximado.
La columna “When to use” es la guía rápida de decisión.

| #   | Hook                              | What it does                                                                                                   | When to use                                                                                                                                 |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publica la configuración del proveedor en `models.providers` durante la generación de `models.json`                                | El proveedor posee un catálogo o valores predeterminados de URL base                                                                                                |
| 2   | `applyConfigDefaults`             | Aplica valores predeterminados globales del proveedor durante la materialización de configuración                                      | Los valores predeterminados dependen del modo de autenticación, del entorno o de la semántica de la familia de modelos del proveedor                                                                       |
| --  | _(built-in model lookup)_         | OpenClaw prueba primero la ruta normal de registro/catálogo                                                          | _(no es un hook de plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Normaliza alias heredados o de vista previa de ids de modelo antes de la búsqueda                                                     | El proveedor posee la limpieza de alias antes de la resolución canónica del modelo                                                                               |
| 4   | `normalizeTransport`              | Normaliza `api` / `baseUrl` de una familia de proveedores antes del ensamblado genérico del modelo                                      | El proveedor posee la limpieza de transporte para ids de proveedor personalizados en la misma familia de transporte                                                        |
| 5   | `normalizeConfig`                 | Normaliza `models.providers.<id>` antes de la resolución de runtime/proveedor                                           | El proveedor necesita limpieza de configuración que debe vivir con el plugin; los helpers incluidos de la familia Google también respaldan entradas de configuración compatibles |
| 6   | `applyNativeStreamingUsageCompat` | Aplica reescrituras de compatibilidad de uso de streaming nativo a proveedores de configuración                                               | El proveedor necesita correcciones de metadatos de uso de streaming nativo impulsadas por el endpoint                                                                        |
| 7   | `resolveConfigApiKey`             | Resuelve autenticación de marcador de entorno para proveedores de configuración antes de cargar la autenticación del runtime                                       | El proveedor tiene resolución de API key por marcador de entorno propiedad del proveedor; `amazon-bedrock` también tiene aquí un resolvedor incorporado de marcadores de entorno de AWS                |
| 8   | `resolveSyntheticAuth`            | Expone autenticación local/alojada por uno mismo o respaldada por configuración sin persistir texto plano                                   | El proveedor puede operar con un marcador de credencial sintética/local                                                                               |
| 9   | `resolveExternalAuthProfiles`     | Superpone perfiles de autenticación externa propiedad del proveedor; `persistence` predeterminado es `runtime-only` para credenciales propiedad de CLI/app | El proveedor reutiliza credenciales de autenticación externa sin persistir tokens de actualización copiados                                                          |
| 10  | `shouldDeferSyntheticProfileAuth` | Rebaja placeholders de perfiles sintéticos almacenados detrás de autenticación respaldada por env/config                                      | El proveedor almacena perfiles placeholder sintéticos que no deberían ganar precedencia                                                               |
| 11  | `resolveDynamicModel`             | Fallback síncrono para ids de modelo propiedad del proveedor que aún no están en el registro local                                       | El proveedor acepta ids de modelo arbitrarios del upstream                                                                                               |
| 12  | `prepareDynamicModel`             | Calentamiento asíncrono, luego `resolveDynamicModel` se ejecuta otra vez                                                           | El proveedor necesita metadatos de red antes de resolver ids desconocidos                                                                                |
| 13  | `normalizeResolvedModel`          | Reescritura final antes de que el ejecutor incrustado use el modelo resuelto                                               | El proveedor necesita reescrituras de transporte pero sigue usando un transporte del núcleo                                                                           |
| 14  | `contributeResolvedModelCompat`   | Aporta banderas de compatibilidad para modelos de proveedor detrás de otro transporte compatible                                  | El proveedor reconoce sus propios modelos en transportes proxy sin asumir el control del proveedor                                                     |
| 15  | `capabilities`                    | Metadatos de transcripción/herramientas propiedad del proveedor usados por la lógica compartida del núcleo                                           | El proveedor necesita particularidades de transcripción/familia de proveedor                                                                                            |
| 16  | `normalizeToolSchemas`            | Normaliza esquemas de herramientas antes de que el ejecutor incrustado los vea                                                    | El proveedor necesita limpieza de esquemas de la familia de transporte                                                                                              |
| 17  | `inspectToolSchemas`              | Expone diagnósticos de esquemas propiedad del proveedor después de la normalización                                                  | El proveedor quiere advertencias de keywords sin enseñar al núcleo reglas específicas del proveedor                                                               |
| 18  | `resolveReasoningOutputMode`      | Selecciona contrato nativo frente a etiquetado de salida de razonamiento                                                              | El proveedor necesita salida de razonamiento/final etiquetada en lugar de campos nativos                                                                       |
| 19  | `prepareExtraParams`              | Normalización de parámetros de solicitud antes de los wrappers genéricos de opciones de stream                                              | El proveedor necesita parámetros de solicitud predeterminados o limpieza de parámetros por proveedor                                                                         |
| 20  | `createStreamFn`                  | Sustituye por completo la ruta normal de stream con un transporte personalizado                                                   | El proveedor necesita un protocolo de red personalizado, no solo un wrapper                                                                                   |
| 21  | `wrapStreamFn`                    | Wrapper de stream después de aplicar los wrappers genéricos                                                              | El proveedor necesita wrappers de compatibilidad de encabezados/cuerpo/modelo de solicitud sin un transporte personalizado                                                        |
| 22  | `resolveTransportTurnState`       | Adjunta encabezados nativos por turno o metadatos de transporte                                                           | El proveedor quiere que transportes genéricos envíen identidad de turno nativa del proveedor                                                                     |
| 23  | `resolveWebSocketSessionPolicy`   | Adjunta encabezados nativos de WebSocket o política de enfriamiento de sesión                                                    | El proveedor quiere que transportes genéricos WS ajusten encabezados de sesión o política de fallback                                                             |
| 24  | `formatApiKey`                    | Formateador de perfil de autenticación: el perfil almacenado se convierte en la cadena `apiKey` del runtime                                     | El proveedor almacena metadatos de autenticación adicionales y necesita una forma de token de runtime personalizada                                                                  |
| 25  | `refreshOAuth`                    | Sobrescritura de actualización de OAuth para endpoints de actualización personalizados o política de fallos de actualización                                  | El proveedor no encaja en los actualizadores compartidos de `pi-ai`                                                                                         |
| 26  | `buildAuthDoctorHint`             | Pista de reparación añadida cuando falla la actualización de OAuth                                                                  | El proveedor necesita una guía de reparación de autenticación propiedad del proveedor tras un fallo de actualización                                                                    |
| 27  | `matchesContextOverflowError`     | Comparador de desbordamiento de ventana de contexto propiedad del proveedor                                                                 | El proveedor tiene errores de desbordamiento en bruto que las heurísticas genéricas pasarían por alto                                                                              |
| 28  | `classifyFailoverReason`          | Clasificación de motivo de failover propiedad del proveedor                                                                  | El proveedor puede mapear errores brutos de API/transporte a rate-limit/sobrecarga/etc.                                                                        |
| 29  | `isCacheTtlEligible`              | Política de caché de prompts para proveedores proxy/backhaul                                                               | El proveedor necesita control específico de TTL de caché para proxies                                                                                              |
| 30  | `buildMissingAuthMessage`         | Sustitución del mensaje genérico de recuperación por falta de autenticación                                                      | El proveedor necesita una pista de recuperación específica del proveedor por falta de autenticación                                                                               |
| 31  | `suppressBuiltInModel`            | Supresión de modelos upstream obsoletos más una pista opcional de error orientada al usuario                                          | El proveedor necesita ocultar filas upstream obsoletas o sustituirlas por una pista del proveedor                                                               |
| 32  | `augmentModelCatalog`             | Filas sintéticas/finales de catálogo añadidas después del descubrimiento                                                          | El proveedor necesita filas sintéticas de compatibilidad futura en `models list` y selectores                                                                   |
| 33  | `isBinaryThinking`                | Conmutador de razonamiento on/off para proveedores de razonamiento binario                                                          | El proveedor expone solo razonamiento binario activado/desactivado                                                                                                |
| 34  | `supportsXHighThinking`           | Compatibilidad con razonamiento `xhigh` para modelos seleccionados                                                                  | El proveedor quiere `xhigh` solo en un subconjunto de modelos                                                                                           |
| 35  | `resolveDefaultThinkingLevel`     | Nivel `/think` predeterminado para una familia de modelos específica                                                             | El proveedor posee la política predeterminada de `/think` para una familia de modelos                                                                                    |
| 36  | `isModernModelRef`                | Comparador de modelos modernos para filtros de perfiles en vivo y selección de smoke                                              | El proveedor posee la coincidencia de modelos preferidos en vivo/smoke                                                                                           |
| 37  | `prepareRuntimeAuth`              | Intercambia una credencial configurada por el token/clave real de runtime justo antes de la inferencia                       | El proveedor necesita un intercambio de token o una credencial de solicitud de corta duración                                                                           |
| 38  | `resolveUsageAuth`                | Resuelve credenciales de uso/facturación para `/usage` y superficies de estado relacionadas                                     | El proveedor necesita análisis personalizado de token de uso/cuota o una credencial de uso diferente                                                             |
| 39  | `fetchUsageSnapshot`              | Obtiene y normaliza snapshots de uso/cuota específicos del proveedor después de resolver la autenticación                             | El proveedor necesita un endpoint de uso específico del proveedor o un analizador de carga                                                                         |
| 40  | `createEmbeddingProvider`         | Crea un adaptador de embeddings propiedad del proveedor para memoria/búsqueda                                                     | El comportamiento de embeddings de memoria debe vivir con el plugin del proveedor                                                                                  |
| 41  | `buildReplayPolicy`               | Devuelve una política de replay que controla el manejo de transcripciones para el proveedor                                        | El proveedor necesita una política personalizada de transcripción (por ejemplo, eliminación de bloques de pensamiento)                                                             |
| 42  | `sanitizeReplayHistory`           | Reescribe el historial de replay después de la limpieza genérica de transcripciones                                                        | El proveedor necesita reescrituras de replay específicas del proveedor más allá de los helpers compartidos de compactación                                                           |
| 43  | `validateReplayTurns`             | Validación o remodelado final de turnos de replay antes del ejecutor incrustado                                           | El transporte del proveedor necesita validación más estricta de turnos después de la limpieza genérica                                                                  |
| 44  | `onModelSelected`                 | Ejecuta efectos secundarios propiedad del proveedor después de la selección                                                                 | El proveedor necesita telemetría o estado propiedad del proveedor cuando un modelo se vuelve activo                                                                |

`normalizeModelId`, `normalizeTransport` y `normalizeConfig` primero comprueban el
plugin de proveedor coincidente, y luego recorren otros plugins de proveedor con capacidad de hook
hasta que uno realmente cambia el id del modelo o el transporte/configuración. Eso mantiene
funcionando los shims de alias/compatibilidad de proveedores sin exigir que quien llama sepa qué
plugin incluido posee la reescritura. Si ningún hook de proveedor reescribe una entrada compatible
de configuración de la familia Google, el normalizador de configuración incluido de Google sigue aplicando
esa limpieza de compatibilidad.

Si el proveedor necesita un protocolo de red totalmente personalizado o un ejecutor de solicitudes personalizado,
esa es otra clase de extensión. Estos hooks son para comportamiento del proveedor
que sigue ejecutándose en el bucle normal de inferencia de OpenClaw.

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
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  y `wrapStreamFn` porque posee la compatibilidad futura de Claude 4.6,
  pistas de familia de proveedor, guía de reparación de autenticación, integración
  con endpoint de uso, elegibilidad de caché de prompts, valores predeterminados de configuración con reconocimiento de autenticación, política
  predeterminada/adaptativa de pensamiento de Claude y modelado de stream específico de Anthropic para
  encabezados beta, `/fast` / `serviceTier` y `context1m`.
- Los helpers de stream específicos de Claude de Anthropic permanecen por ahora en la
  propia unión pública `api.ts` / `contract-api.ts` del plugin incluido. Esa superficie de paquete
  exporta `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` y los generadores
  de wrappers de Anthropic de nivel inferior en lugar de ampliar el SDK genérico alrededor de las reglas
  de encabezados beta de un solo proveedor.
- OpenAI usa `resolveDynamicModel`, `normalizeResolvedModel` y
  `capabilities` más `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` e `isModernModelRef`
  porque posee la compatibilidad futura de GPT-5.4, la normalización directa
  de OpenAI `openai-completions` -> `openai-responses`, pistas de autenticación
  con reconocimiento de Codex, supresión de Spark, filas sintéticas de lista de OpenAI y la política
  de pensamiento / modelos en vivo de GPT-5; la familia de streams `openai-responses-defaults` posee los
  wrappers compartidos nativos de OpenAI Responses para encabezados de atribución,
  `/fast`/`serviceTier`, verbosidad de texto, búsqueda web nativa de Codex,
  modelado de carga de compatibilidad de razonamiento y gestión de contexto de Responses.
- OpenRouter usa `catalog` más `resolveDynamicModel` y
  `prepareDynamicModel` porque el proveedor es pass-through y puede exponer nuevos
  ids de modelo antes de que se actualice el catálogo estático de OpenClaw; también usa
  `capabilities`, `wrapStreamFn` e `isCacheTtlEligible` para mantener
  encabezados de solicitud específicos del proveedor, metadatos de enrutamiento, parches de razonamiento y
  política de caché de prompts fuera del núcleo. Su política de replay proviene de la
  familia `passthrough-gemini`, mientras que la familia de streams `openrouter-thinking`
  posee la inyección de razonamiento proxy y los saltos de modelo no compatible / `auto`.
- GitHub Copilot usa `catalog`, `auth`, `resolveDynamicModel` y
  `capabilities` más `prepareRuntimeAuth` y `fetchUsageSnapshot` porque
  necesita inicio de sesión de dispositivo propiedad del proveedor, comportamiento de fallback de modelos, particularidades
  de transcripción de Claude, un intercambio de token de GitHub -> token de Copilot y un endpoint
  de uso propiedad del proveedor.
- OpenAI Codex usa `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` y `augmentModelCatalog` más
  `prepareExtraParams`, `resolveUsageAuth` y `fetchUsageSnapshot` porque
  aún se ejecuta sobre los transportes centrales de OpenAI pero posee su normalización
  de transporte/URL base, política de fallback para actualización de OAuth, elección predeterminada de transporte,
  filas sintéticas de catálogo de Codex e integración con el endpoint de uso de ChatGPT; comparte la misma familia de streams `openai-responses-defaults` que OpenAI directo.
- Google AI Studio y Gemini CLI OAuth usan `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` e `isModernModelRef` porque la
  familia de replay `google-gemini` posee el fallback de compatibilidad futura de Gemini 3.1,
  la validación nativa de replay de Gemini, la sanitización de replay de bootstrap, el modo
  etiquetado de salida de razonamiento y la coincidencia de modelos modernos, mientras que la
  familia de streams `google-thinking` posee la normalización de cargas de pensamiento de Gemini;
  Gemini CLI OAuth también usa `formatApiKey`, `resolveUsageAuth` y
  `fetchUsageSnapshot` para formateo de tokens, análisis de tokens e integración
  con el endpoint de cuota.
- Anthropic Vertex usa `buildReplayPolicy` a través de la
  familia de replay `anthropic-by-model` para que la limpieza de replay específica de Claude quede
  limitada a ids de Claude en lugar de a todo transporte `anthropic-messages`.
- Amazon Bedrock usa `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` y `resolveDefaultThinkingLevel` porque posee la clasificación específica de Bedrock para errores de limitación, no-listo y desbordamiento de contexto
  en tráfico de Anthropic sobre Bedrock; su política de replay sigue compartiendo la misma
  protección `anthropic-by-model` solo para Claude.
- OpenRouter, Kilocode, Opencode y Opencode Go usan `buildReplayPolicy`
  a través de la familia de replay `passthrough-gemini` porque hacen proxy de modelos Gemini
  a través de transportes compatibles con OpenAI y necesitan la sanitización de firmas de pensamiento de Gemini sin validación nativa de replay de Gemini ni reescrituras
  de bootstrap.
- MiniMax usa `buildReplayPolicy` a través de la
  familia `hybrid-anthropic-openai` porque un proveedor posee tanto semántica de mensajes
  de Anthropic como compatible con OpenAI; mantiene la eliminación de bloques de pensamiento solo de Claude en el lado
  Anthropic mientras sobrescribe el modo de salida de razonamiento de vuelta a nativo, y la familia de streams `minimax-fast-mode` posee las reescrituras de modelos de modo rápido en la ruta compartida de stream.
- Moonshot usa `catalog` más `wrapStreamFn` porque todavía usa el transporte
  compartido de OpenAI pero necesita normalización de cargas de pensamiento propiedad del proveedor; la
  familia de streams `moonshot-thinking` asigna la configuración más el estado `/think` a su
  carga nativa binaria de pensamiento.
- Kilocode usa `catalog`, `capabilities`, `wrapStreamFn` e
  `isCacheTtlEligible` porque necesita encabezados de solicitud propiedad del proveedor,
  normalización de cargas de razonamiento, pistas de transcripción de Gemini y control
  de TTL de caché de Anthropic; la familia de streams `kilocode-thinking` mantiene la inyección de pensamiento de Kilo en la ruta compartida de stream proxy mientras omite `kilo/auto` y
  otros ids de modelo proxy que no admiten cargas explícitas de razonamiento.
- Z.AI usa `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` y `fetchUsageSnapshot` porque posee el fallback de GLM-5,
  valores predeterminados de `tool_stream`, UX de pensamiento binario, coincidencia de modelos modernos y tanto autenticación de uso como obtención de cuota; la familia de streams `tool-stream-default-on` mantiene el wrapper predeterminado activado de `tool_stream` fuera del pegamento escrito a mano por proveedor.
- xAI usa `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` e `isModernModelRef`
  porque posee la normalización nativa de transporte xAI Responses, reescrituras de alias
  de modo rápido de Grok, `tool_stream` predeterminado, limpieza estricta de herramientas/cargas de razonamiento,
  reutilización de autenticación de fallback para herramientas propiedad del plugin, resolución de modelos Grok
  con compatibilidad futura y parches de compatibilidad propiedad del proveedor como el perfil
  de esquema de herramientas de xAI, keywords de esquema no compatibles, `web_search` nativo y decodificación
  de entidades HTML en argumentos de llamadas de herramientas.
- Mistral, OpenCode Zen y OpenCode Go usan solo `capabilities` para mantener
  fuera del núcleo las particularidades de transcripción/herramientas.
- Los proveedores incluidos solo de catálogo como `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` y `volcengine` usan
  solo `catalog`.
- Qwen usa `catalog` para su proveedor de texto más registros compartidos de comprensión multimedia y
  generación de video para sus superficies multimodales.
- MiniMax y Xiaomi usan `catalog` más hooks de uso porque su comportamiento de `/usage`
  es propiedad del plugin aunque la inferencia siga ejecutándose a través de los transportes compartidos.

## Helpers de runtime

Los plugins pueden acceder a helpers seleccionados del núcleo a través de `api.runtime`. Para TTS:

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

- `textToSpeech` devuelve la carga normal de salida de TTS del núcleo para superficies de archivo/nota de voz.
- Usa la configuración central `messages.tts` y la selección de proveedor.
- Devuelve un buffer de audio PCM + frecuencia de muestreo. Los plugins deben remuestrear/codificar para los proveedores.
- `listVoices` es opcional por proveedor. Úsalo para selectores de voz o flujos de configuración propiedad del proveedor.
- Los listados de voces pueden incluir metadatos más ricos como locale, género y etiquetas de personalidad para selectores con conocimiento del proveedor.
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

- Mantén la política de TTS, el fallback y la entrega de respuestas en el núcleo.
- Usa proveedores de voz para comportamiento de síntesis propiedad del proveedor.
- La entrada heredada `edge` de Microsoft se normaliza al id de proveedor `microsoft`.
- El modelo de propiedad preferido está orientado a la empresa: un plugin de proveedor puede poseer
  texto, voz, imagen y futuros proveedores multimedia a medida que OpenClaw agregue esos
  contratos de capacidad.

Para comprensión de imagen/audio/video, los plugins registran un proveedor tipado
de comprensión multimedia en lugar de una bolsa genérica de clave/valor:

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

- Mantén la orquestación, el fallback, la configuración y la integración con canales en el núcleo.
- Mantén el comportamiento del proveedor en el plugin del proveedor.
- La expansión aditiva debe seguir siendo tipada: nuevos métodos opcionales, nuevos campos de resultado opcionales, nuevas capacidades opcionales.
- La generación de video ya sigue el mismo patrón:
  - el núcleo posee el contrato de capacidad y el helper de runtime
  - los plugins de proveedores registran `api.registerVideoGenerationProvider(...)`
  - los plugins de función/canal consumen `api.runtime.videoGeneration.*`

Para los helpers de runtime de comprensión multimedia, los plugins pueden llamar a:

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

Para transcripción de audio, los plugins pueden usar el runtime de comprensión multimedia
o el alias antiguo STT:

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
- Usa la configuración central de audio de comprensión multimedia (`tools.media.audio`) y el orden de fallback de proveedores.
- Devuelve `{ text: undefined }` cuando no se produce salida de transcripción (por ejemplo, entrada omitida/no compatible).
- `api.runtime.stt.transcribeAudioFile(...)` permanece como alias de compatibilidad.

Los plugins también pueden lanzar ejecuciones de subagentes en segundo plano mediante `api.runtime.subagent`:

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

- `provider` y `model` son sobrescrituras opcionales por ejecución, no cambios persistentes de sesión.
- OpenClaw solo respeta esos campos de sobrescritura para llamadores de confianza.
- Para ejecuciones de fallback propiedad del plugin, los operadores deben habilitarlo expresamente con `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Usa `plugins.entries.<id>.subagent.allowedModels` para restringir plugins de confianza a objetivos canónicos específicos `provider/model`, o `"*"` para permitir explícitamente cualquier objetivo.
- Las ejecuciones de subagentes de plugins no confiables siguen funcionando, pero las solicitudes de sobrescritura se rechazan en lugar de usar silencio como fallback.

Para búsqueda web, los plugins pueden consumir el helper compartido de runtime en lugar de
acceder directamente a la integración de herramientas del agente:

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

- Mantén la selección de proveedor, la resolución de credenciales y la semántica compartida de solicitudes en el núcleo.
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
- `listProviders(...)`: lista los proveedores disponibles de generación de imágenes y sus capacidades.

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
- `auth`: obligatorio. Usa `"gateway"` para requerir autenticación normal del gateway, o `"plugin"` para autenticación/validación de webhook gestionadas por el plugin.
- `match`: opcional. `"exact"` (predeterminado) o `"prefix"`.
- `replaceExisting`: opcional. Permite que el mismo plugin reemplace su propio registro de ruta existente.
- `handler`: devuelve `true` cuando la ruta manejó la solicitud.

Notas:

- `api.registerHttpHandler(...)` fue eliminado y causará un error de carga del plugin. Usa `api.registerHttpRoute(...)` en su lugar.
- Las rutas de plugins deben declarar `auth` explícitamente.
- Los conflictos exactos de `path + match` se rechazan salvo que `replaceExisting: true`, y un plugin no puede reemplazar la ruta de otro plugin.
- Las rutas superpuestas con distintos niveles de `auth` se rechazan. Mantén cadenas de paso `exact`/`prefix` solo en el mismo nivel de autenticación.
- Las rutas `auth: "plugin"` **no** reciben automáticamente scopes de runtime del operador. Son para webhooks/validación de firmas gestionados por el plugin, no para llamadas privilegiadas a helpers del Gateway.
- Las rutas `auth: "gateway"` se ejecutan dentro de un scope de runtime de solicitud del Gateway, pero ese scope es intencionalmente conservador:
  - la autenticación bearer por secreto compartido (`gateway.auth.mode = "token"` / `"password"`) mantiene los scopes de runtime de rutas de plugin fijados en `operator.write`, incluso si quien llama envía `x-openclaw-scopes`
  - los modos HTTP de confianza con identidad (por ejemplo `trusted-proxy` o `gateway.auth.mode = "none"` en un ingreso privado) respetan `x-openclaw-scopes` solo cuando la cabecera está presente de forma explícita
  - si `x-openclaw-scopes` está ausente en esas solicitudes de rutas de plugin con identidad, el scope de runtime vuelve a `operator.write`
- Regla práctica: no asumas que una ruta de plugin autenticada por gateway es implícitamente una superficie de administración. Si tu ruta necesita comportamiento exclusivo de admin, exige un modo de autenticación con identidad y documenta el contrato explícito de la cabecera `x-openclaw-scopes`.

## Rutas de importación del Plugin SDK

Usa subrutas del SDK en lugar de la importación monolítica `openclaw/plugin-sdk` al
crear plugins:

- `openclaw/plugin-sdk/plugin-entry` para primitivas de registro de plugins.
- `openclaw/plugin-sdk/core` para el contrato genérico compartido orientado a plugins.
- `openclaw/plugin-sdk/config-schema` para la exportación del esquema Zod raíz de `openclaw.json`
  (`OpenClawSchema`).
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
  `openclaw/plugin-sdk/webhook-ingress` para la integración compartida de
  configuración/autenticación/respuesta/webhook. `channel-inbound` es la superficie compartida para debounce, coincidencia de menciones,
  helpers de política de menciones entrantes, formato de envelopes y helpers de
  contexto de envelopes entrantes.
  `channel-setup` es la unión estrecha de configuración opcional durante la instalación.
  `setup-runtime` es la superficie de configuración segura en runtime usada por `setupEntry` /
  inicio diferido, incluidos los adaptadores de parches de configuración seguros para importación.
  `setup-adapter-runtime` es la unión del adaptador de configuración de cuentas consciente del entorno.
  `setup-tools` es la pequeña unión helper para CLI/archivo/docs (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Subrutas de dominio como `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
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
  `telegram-command-config` es la unión pública estrecha para normalización/validación de comandos personalizados de Telegram y sigue disponible incluso si la superficie de contrato incluida de Telegram no está disponible temporalmente.
  `text-runtime` es la unión compartida de texto/markdown/logging, incluida
  la eliminación de texto visible para el asistente, helpers de renderizado/segmentación de markdown, helpers de redacción,
  helpers de etiquetas directivas y utilidades de texto seguro.
- Las uniones de canal específicas de aprobación deben preferir un único contrato `approvalCapability` en el plugin. El núcleo
  luego lee autenticación de aprobación, entrega, renderizado, enrutamiento nativo y comportamiento perezoso del manejador nativo a través de esa única capacidad
  en lugar de mezclar el comportamiento de aprobación en campos no relacionados del plugin.
- `openclaw/plugin-sdk/channel-runtime` está obsoleto y permanece solo como shim de compatibilidad para plugins antiguos. El código nuevo debe importar
  las primitivas genéricas más estrechas, y el código del repositorio no debe agregar nuevas importaciones del shim.
- Los aspectos internos de extensiones incluidas siguen siendo privados. Los plugins externos deben usar solo subrutas `openclaw/plugin-sdk/*`. El código/pruebas del núcleo de OpenClaw puede usar los puntos de entrada públicos del repositorio bajo la raíz de un paquete de plugin como `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` y archivos de alcance estrecho como
  `login-qr-api.js`. Nunca importes `src/*` de un paquete de plugin desde el núcleo o desde otra extensión.
- División del punto de entrada del repositorio:
  `<plugin-package-root>/api.js` es el barril de helpers/tipos,
  `<plugin-package-root>/runtime-api.js` es el barril solo de runtime,
  `<plugin-package-root>/index.js` es la entrada del plugin incluido
  y `<plugin-package-root>/setup-entry.js` es la entrada del plugin de configuración.
- Ejemplos actuales de proveedores incluidos:
  - Anthropic usa `api.js` / `contract-api.js` para helpers de stream de Claude como
    `wrapAnthropicProviderStream`, helpers de encabezados beta y análisis de `service_tier`.
  - OpenAI usa `api.js` para constructores de proveedores, helpers de modelos predeterminados y constructores de proveedores en tiempo real.
  - OpenRouter usa `api.js` para su constructor de proveedor más helpers de onboarding/configuración,
    mientras que `register.runtime.js` todavía puede reexportar helpers genéricos
    `plugin-sdk/provider-stream` para uso local del repositorio.
- Los puntos de entrada públicos cargados mediante fachadas prefieren la instantánea activa de configuración de runtime
  cuando existe una, y luego recurren al archivo de configuración resuelto en disco cuando
  OpenClaw aún no está sirviendo una instantánea de runtime.
- Las primitivas genéricas compartidas siguen siendo el contrato público preferido del SDK. Aún existe
  un pequeño conjunto reservado de compatibilidad de uniones helper con marca de canal incluidas. Trátalas
  como uniones de compatibilidad/mantenimiento de los incluidos, no como nuevos objetivos de importación para terceros; los nuevos contratos entre canales deben seguir llegando a subrutas genéricas `plugin-sdk/*` o a los barriles locales `api.js` /
  `runtime-api.js` del plugin.

Nota de compatibilidad:

- Evita el barril raíz `openclaw/plugin-sdk` en código nuevo.
- Prefiere primero las primitivas estables y estrechas. Las nuevas subrutas de configuración/emparejamiento/respuesta/
  feedback/contrato/inbound/threading/comando/secret-input/webhook/infra/
  allowlist/status/message-tool son el contrato previsto para trabajo nuevo
  de plugins incluidos y externos.
  El análisis/coincidencia de objetivos pertenece a `openclaw/plugin-sdk/channel-targets`.
  Las compuertas de acciones de mensajes y los helpers de id de mensaje de reacciones pertenecen a
  `openclaw/plugin-sdk/channel-actions`.
- Los barriles helper específicos de extensiones incluidas no son estables por defecto. Si un
  helper solo lo necesita una extensión incluida, mantenlo detrás de la unión local
  `api.js` o `runtime-api.js` de la extensión en lugar de promocionarlo a
  `openclaw/plugin-sdk/<extension>`.
- Las nuevas uniones helper compartidas deben ser genéricas, no con marca de canal. El análisis compartido
  de objetivos pertenece a `openclaw/plugin-sdk/channel-targets`; los aspectos internos específicos del canal
  permanecen detrás de la unión local `api.js` o `runtime-api.js` del plugin propietario.
- Existen subrutas específicas de capacidad como `image-generation`,
  `media-understanding` y `speech` porque los plugins incluidos/nativos las usan hoy.
  Su presencia no significa por sí sola que cada helper exportado sea un
  contrato externo congelado a largo plazo.

## Esquemas de la herramienta de mensajes

Los plugins deben poseer las contribuciones de esquema específicas del canal de `describeMessageTool(...)`.
Mantén los campos específicos del proveedor en el plugin, no en el núcleo compartido.

Para fragmentos compartidos de esquema portable, reutiliza los helpers genéricos exportados a través de
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` para cargas útiles tipo cuadrícula de botones
- `createMessageToolCardSchema()` para cargas útiles estructuradas de tarjetas

Si una forma de esquema solo tiene sentido para un proveedor, defínela en el
código fuente de ese plugin en lugar de promoverla al SDK compartido.

## Resolución de objetivos de canal

Los plugins de canal deben poseer la semántica de objetivos específica del canal. Mantén el host
genérico compartido de salida y usa la superficie del adaptador de mensajería para las reglas del proveedor:

- `messaging.inferTargetChatType({ to })` decide si un objetivo normalizado
  debe tratarse como `direct`, `group` o `channel` antes de buscar en el directorio.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indica al núcleo si una
  entrada debe pasar directamente a resolución tipo id en lugar de búsqueda en directorio.
- `messaging.targetResolver.resolveTarget(...)` es el fallback del plugin cuando
  el núcleo necesita una resolución final propiedad del proveedor tras la normalización o tras un fallo de directorio.
- `messaging.resolveOutboundSessionRoute(...)` posee la construcción de rutas de sesión específicas del proveedor una vez que se resuelve un objetivo.

División recomendada:

- Usa `inferTargetChatType` para decisiones de categoría que deban ocurrir antes
  de buscar en peers/grupos.
- Usa `looksLikeId` para comprobaciones de “tratar esto como un id de objetivo explícito/nativo”.
- Usa `resolveTarget` para fallback de normalización específico del proveedor, no para
  búsqueda amplia en directorio.
- Mantén ids nativos del proveedor como ids de chat, ids de hilo, JIDs, handles e ids de sala dentro de valores `target` o parámetros específicos del proveedor, no en campos genéricos del SDK.

## Directorios respaldados por configuración

Los plugins que derivan entradas de directorio a partir de la configuración deben mantener esa lógica en el
plugin y reutilizar los helpers compartidos de
`openclaw/plugin-sdk/directory-runtime`.

Usa esto cuando un canal necesite peers/grupos respaldados por configuración como:

- peers de MD impulsados por allowlist
- mapas configurados de canales/grupos
- fallbacks de directorio estáticos con alcance por cuenta

Los helpers compartidos en `directory-runtime` solo manejan operaciones genéricas:

- filtrado de consultas
- aplicación de límites
- helpers de deduplicación/normalización
- creación de `ChannelDirectoryEntry[]`

La inspección de cuentas y la normalización de ids específicas del canal deben seguir en la implementación del plugin.

## Catálogos de proveedores

Los plugins de proveedores pueden definir catálogos de modelos para inferencia con
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` devuelve la misma forma que OpenClaw escribe en
`models.providers`:

- `{ provider }` para una entrada de proveedor
- `{ providers }` para varias entradas de proveedor

Usa `catalog` cuando el plugin posea ids de modelos específicos del proveedor, valores
predeterminados de URL base o metadatos de modelos limitados por autenticación.

`catalog.order` controla cuándo se fusiona el catálogo de un plugin respecto de los
proveedores implícitos integrados de OpenClaw:

- `simple`: proveedores sencillos impulsados por API key o entorno
- `profile`: proveedores que aparecen cuando existen perfiles de autenticación
- `paired`: proveedores que sintetizan varias entradas de proveedor relacionadas
- `late`: última pasada, después de otros proveedores implícitos

Los proveedores posteriores ganan en colisión de claves, por lo que los plugins pueden
sobrescribir intencionalmente una entrada de proveedor integrada con el mismo id de proveedor.

Compatibilidad:

- `discovery` sigue funcionando como alias heredado
- si se registran tanto `catalog` como `discovery`, OpenClaw usa `catalog`

## Inspección de canal en solo lectura

Si tu plugin registra un canal, prefiere implementar
`plugin.config.inspectAccount(cfg, accountId)` junto con `resolveAccount(...)`.

Por qué:

- `resolveAccount(...)` es la ruta de runtime. Se le permite asumir que las credenciales
  están totalmente materializadas y puede fallar rápido cuando faltan secretos obligatorios.
- Las rutas de comandos de solo lectura como `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` y los flujos de doctor/configuración
  de reparación no deberían necesitar materializar credenciales de runtime solo para
  describir la configuración.

Comportamiento recomendado para `inspectAccount(...)`:

- Devuelve solo el estado descriptivo de la cuenta.
- Conserva `enabled` y `configured`.
- Incluye campos de origen/estado de credenciales cuando corresponda, como:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- No necesitas devolver valores brutos de tokens solo para informar disponibilidad en solo lectura. Devolver `tokenStatus: "available"` (y el campo de origen correspondiente) es suficiente para comandos tipo status.
- Usa `configured_unavailable` cuando una credencial esté configurada mediante SecretRef pero no esté disponible en la ruta de comando actual.

Esto permite que los comandos de solo lectura informen “configurado pero no disponible en esta ruta de comando”
en lugar de fallar o informar erróneamente que la cuenta no está configurada.

## Paquetes pack

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

Cada entrada se convierte en un plugin. Si el pack lista varias extensiones, el id del plugin
pasa a ser `name/<fileBase>`.

Si tu plugin importa dependencias npm, instálalas en ese directorio para que
`node_modules` esté disponible (`npm install` / `pnpm install`).

Barrera de seguridad: cada entrada `openclaw.extensions` debe permanecer dentro del directorio del plugin
después de resolver symlinks. Las entradas que escapen del directorio del paquete son
rechazadas.

Nota de seguridad: `openclaw plugins install` instala dependencias de plugins con
`npm install --omit=dev --ignore-scripts` (sin scripts de ciclo de vida, sin dependencias de desarrollo en runtime). Mantén los árboles de dependencias de plugins como "JS/TS puro" y evita paquetes que requieran compilaciones en `postinstall`.

Opcional: `openclaw.setupEntry` puede apuntar a un módulo ligero solo de configuración.
Cuando OpenClaw necesita superficies de configuración para un plugin de canal deshabilitado, o
cuando un plugin de canal está habilitado pero aún no configurado, carga `setupEntry`
en lugar de la entrada completa del plugin. Esto mantiene el inicio y la configuración más ligeros
cuando la entrada principal del plugin también conecta herramientas, hooks u otro código solo de runtime.

Opcional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
puede hacer que un plugin de canal use la misma ruta `setupEntry` durante la
fase de inicio previa a listen del gateway, incluso cuando el canal ya está configurado.

Usa esto solo cuando `setupEntry` cubra completamente la superficie de inicio que debe existir
antes de que el gateway empiece a escuchar. En la práctica, eso significa que la entrada de configuración
debe registrar toda capacidad propiedad del canal de la que dependa el inicio, como:

- el propio registro del canal
- cualquier ruta HTTP que deba estar disponible antes de que el gateway empiece a escuchar
- cualquier método del gateway, herramienta o servicio que deba existir durante esa misma ventana

Si tu entrada completa sigue poseyendo alguna capacidad de inicio requerida, no habilites
esta bandera. Mantén el comportamiento predeterminado del plugin y deja que OpenClaw cargue la
entrada completa durante el inicio.

Los canales incluidos también pueden publicar helpers de superficie contractual solo de configuración que el núcleo
puede consultar antes de que se cargue el runtime completo del canal. La superficie actual de promoción de configuración es:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

El núcleo usa esa superficie cuando necesita promover una configuración heredada de canal de una sola cuenta a
`channels.<id>.accounts.*` sin cargar la entrada completa del plugin.
Matrix es el ejemplo incluido actual: mueve solo claves de autenticación/bootstrap a una cuenta promovida con nombre cuando ya existen cuentas con nombre, y puede preservar una clave configurada no canónica de cuenta predeterminada en lugar de crear siempre
`accounts.default`.

Esos adaptadores de parches de configuración mantienen perezoso el descubrimiento de superficies contractuales incluidas. El tiempo de importación sigue siendo ligero; la superficie de promoción se carga
solo en el primer uso en lugar de reingresar al inicio del canal incluido al importar el módulo.

Cuando esas superficies de inicio incluyan métodos RPC del gateway, mantenlas en un
prefijo específico del plugin. Los espacios de nombres de administración del núcleo (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) siguen reservados y siempre se resuelven
a `operator.admin`, incluso si un plugin solicita un scope más estrecho.

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

### Metadatos de catálogo de canal

Los plugins de canal pueden anunciar metadatos de configuración/descubrimiento mediante `openclaw.channel` y
pistas de instalación mediante `openclaw.install`. Esto mantiene los datos de catálogo libres de núcleo.

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

Campos útiles de `openclaw.channel` más allá del ejemplo mínimo:

- `detailLabel`: etiqueta secundaria para superficies más ricas de catálogo/status
- `docsLabel`: sobrescribe el texto del enlace a la documentación
- `preferOver`: ids de plugin/canal de menor prioridad a los que esta entrada de catálogo debe superar
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: controles de texto para superficies de selección
- `markdownCapable`: marca el canal como compatible con markdown para decisiones de formato saliente
- `exposure.configured`: oculta el canal de las superficies de listado de canales configurados cuando se establece en `false`
- `exposure.setup`: oculta el canal de los selectores interactivos de configuración cuando se establece en `false`
- `exposure.docs`: marca el canal como interno/privado para superficies de navegación de documentación
- `showConfigured` / `showInSetup`: alias heredados aún aceptados por compatibilidad; prefiere `exposure`
- `quickstartAllowFrom`: hace que el canal participe en el flujo estándar de inicio rápido `allowFrom`
- `forceAccountBinding`: requiere vinculación explícita de cuenta incluso cuando solo existe una cuenta
- `preferSessionLookupForAnnounceTarget`: prefiere la búsqueda de sesión al resolver objetivos de anuncios

OpenClaw también puede fusionar **catálogos externos de canales** (por ejemplo, una
exportación de registro MPM). Coloca un archivo JSON en una de estas rutas:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

O apunta `OPENCLAW_PLUGIN_CATALOG_PATHS` (o `OPENCLAW_MPM_CATALOG_PATHS`) a
uno o más archivos JSON (delimitados por coma/punto y coma/`PATH`). Cada archivo debe
contener `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. El analizador también acepta `"packages"` o `"plugins"` como alias heredados de la clave `"entries"`.

## Plugins de motor de contexto

Los plugins de motor de contexto poseen la orquestación del contexto de sesión para ingesta,
ensamblaje y compactación. Regístralos desde tu plugin con
`api.registerContextEngine(id, factory)`, y luego selecciona el motor activo con
`plugins.slots.contextEngine`.

Usa esto cuando tu plugin necesite reemplazar o ampliar la canalización predeterminada de contexto
en lugar de simplemente agregar búsqueda de memoria o hooks.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Si tu motor **no** posee el algoritmo de compactación, mantén `compact()`
implementado y delega explícitamente:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

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
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Agregar una nueva capacidad

Cuando un plugin necesite un comportamiento que no encaje en la API actual, no eludas
el sistema de plugins con un acceso privado. Agrega la capacidad faltante.

Secuencia recomendada:

1. definir el contrato del núcleo
   Decide qué comportamiento compartido debe poseer el núcleo: política, fallback, configuración
   de fusión, ciclo de vida, semántica orientada al canal y forma del helper de runtime.
2. agregar superficies tipadas de registro/runtime de plugins
   Extiende `OpenClawPluginApi` y/o `api.runtime` con la superficie de capacidad tipada más pequeña y útil.
3. conectar consumidores del núcleo + canal/función
   Los canales y plugins de funciones deben consumir la nueva capacidad a través del núcleo,
   no importando directamente una implementación de proveedor.
4. registrar implementaciones de proveedores
   Los plugins de proveedores registran entonces sus backends contra la capacidad.
5. agregar cobertura contractual
   Agrega pruebas para que la propiedad y la forma del registro sigan siendo explícitas con el tiempo.

Así es como OpenClaw se mantiene con criterio sin quedar codificado de forma rígida a la visión del mundo
de un solo proveedor. Consulta [Capability Cookbook](/es/plugins/architecture)
para una lista concreta de archivos y un ejemplo trabajado.

### Lista de verificación de capacidad

Cuando agregues una nueva capacidad, la implementación normalmente debe tocar estas
superficies a la vez:

- tipos de contrato del núcleo en `src/<capability>/types.ts`
- ejecutor/helper de runtime del núcleo en `src/<capability>/runtime.ts`
- superficie de registro de la API de plugins en `src/plugins/types.ts`
- integración del registro de plugins en `src/plugins/registry.ts`
- exposición de runtime de plugins en `src/plugins/runtime/*` cuando plugins de función/canal
  necesiten consumirla
- helpers de captura/prueba en `src/test-utils/plugin-registration.ts`
- afirmaciones de propiedad/contrato en `src/plugins/contracts/registry.ts`
- documentación para operadores/plugins en `docs/`

Si falta una de esas superficies, normalmente es una señal de que la capacidad todavía
no está totalmente integrada.

### Plantilla de capacidad

Patrón mínimo:

```ts
// contrato del núcleo
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// API del plugin
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// helper compartido de runtime para plugins de función/canal
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
- los plugins de proveedores poseen las implementaciones del proveedor
- los plugins de función/canal consumen helpers de runtime
- las pruebas de contrato mantienen explícita la propiedad
