---
read_when:
    - Quieres entender cómo OpenClaw ensambla el contexto del modelo
    - Estás cambiando entre el motor heredado y un motor de plugin
    - Estás creando un plugin de motor de contexto
summary: 'Motor de contexto: ensamblaje de contexto conectable, compactación y ciclo de vida de subagentes'
title: Motor de contexto
x-i18n:
    generated_at: "2026-04-08T02:14:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: e8290ac73272eee275bce8e481ac7959b65386752caa68044d0c6f3e450acfb1
    source_path: concepts/context-engine.md
    workflow: 15
---

# Motor de contexto

Un **motor de contexto** controla cómo OpenClaw construye el contexto del modelo para cada ejecución.
Decide qué mensajes incluir, cómo resumir el historial anterior y cómo
gestionar el contexto a través de los límites de subagentes.

OpenClaw incluye un motor integrado `legacy`. Los plugins pueden registrar
motores alternativos que sustituyan el ciclo de vida activo del motor de contexto.

## Inicio rápido

Comprueba qué motor está activo:

```bash
openclaw doctor
# or inspect config directly:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### Instalar un plugin de motor de contexto

Los plugins de motor de contexto se instalan como cualquier otro plugin de OpenClaw. Instálalo
primero y luego selecciona el motor en la ranura:

```bash
# Install from npm
openclaw plugins install @martian-engineering/lossless-claw

# Or install from a local path (for development)
openclaw plugins install -l ./my-context-engine
```

Luego habilita el plugin y selecciónalo como motor activo en tu configuración:

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // must match the plugin's registered engine id
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // Plugin-specific config goes here (see the plugin's docs)
      },
    },
  },
}
```

Reinicia la puerta de enlace después de instalar y configurar.

Para volver al motor integrado, establece `contextEngine` en `"legacy"` (o
elimina la clave por completo: `"legacy"` es el valor predeterminado).

## Cómo funciona

Cada vez que OpenClaw ejecuta una indicación del modelo, el motor de contexto participa en
cuatro puntos del ciclo de vida:

1. **Ingesta** — se llama cuando se añade un mensaje nuevo a la sesión. El motor
   puede almacenar o indexar el mensaje en su propio almacén de datos.
2. **Ensamblaje** — se llama antes de cada ejecución del modelo. El motor devuelve un conjunto
   ordenado de mensajes (y un `systemPromptAddition` opcional) que caben dentro
   del presupuesto de tokens.
3. **Compactación** — se llama cuando la ventana de contexto está llena, o cuando el usuario ejecuta
   `/compact`. El motor resume el historial anterior para liberar espacio.
4. **Después del turno** — se llama después de que se complete una ejecución. El motor puede persistir el estado,
   activar la compactación en segundo plano o actualizar índices.

### Ciclo de vida del subagente (opcional)

Actualmente OpenClaw llama a un hook del ciclo de vida del subagente:

- **onSubagentEnded** — limpia cuando una sesión de subagente se completa o se depura.

El hook `prepareSubagentSpawn` forma parte de la interfaz para uso futuro, pero
el entorno de ejecución todavía no lo invoca.

### Adición a la indicación del sistema

El método `assemble` puede devolver una cadena `systemPromptAddition`. OpenClaw
la antepone a la indicación del sistema de la ejecución. Esto permite que los motores inyecten
guía de recuperación dinámica, instrucciones de recuperación o pistas
conscientes del contexto sin requerir archivos estáticos del espacio de trabajo.

## El motor heredado

El motor integrado `legacy` conserva el comportamiento original de OpenClaw:

- **Ingesta**: sin operación (el administrador de sesiones gestiona directamente la persistencia de mensajes).
- **Ensamblaje**: paso directo (la canalización existente sanitize → validate → limit
  del entorno de ejecución se encarga del ensamblaje del contexto).
- **Compactación**: delega en la compactación de resumen integrada, que crea
  un único resumen de los mensajes anteriores y mantiene intactos los mensajes recientes.
- **Después del turno**: sin operación.

El motor heredado no registra herramientas ni proporciona un `systemPromptAddition`.

Cuando no se establece `plugins.slots.contextEngine` (o se establece en `"legacy"`), este
motor se usa automáticamente.

## Motores de plugin

Un plugin puede registrar un motor de contexto usando la API de plugins:

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", () => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // Store the message in your data store
      return { ingested: true };
    },

    async assemble({ sessionId, messages, tokenBudget, availableTools, citationsMode }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

Luego habilítalo en la configuración:

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### La interfaz ContextEngine

Miembros obligatorios:

| Member             | Kind     | Purpose                                                  |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | Property | Id, nombre, versión del motor y si posee la compactación |
| `ingest(params)`   | Method   | Almacenar un solo mensaje                                |
| `assemble(params)` | Method   | Construir contexto para una ejecución del modelo (devuelve `AssembleResult`) |
| `compact(params)`  | Method   | Resumir/reducir contexto                                 |

`assemble` devuelve un `AssembleResult` con:

- `messages` — los mensajes ordenados que se enviarán al modelo.
- `estimatedTokens` (obligatorio, `number`) — la estimación del motor del total de
  tokens en el contexto ensamblado. OpenClaw usa esto para las decisiones sobre el umbral de compactación
  y para los informes de diagnóstico.
- `systemPromptAddition` (opcional, `string`) — se antepone a la indicación del sistema.

Miembros opcionales:

| Member                         | Kind   | Purpose                                                                                                         |
| ------------------------------ | ------ | --------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | Method | Inicializar el estado del motor para una sesión. Se llama una vez cuando el motor ve una sesión por primera vez (p. ej., importar historial). |
| `ingestBatch(params)`          | Method | Ingerir un turno completado como lote. Se llama después de que se completa una ejecución, con todos los mensajes de ese turno a la vez.     |
| `afterTurn(params)`            | Method | Trabajo posterior a la ejecución en el ciclo de vida (persistir estado, activar compactación en segundo plano).                                         |
| `prepareSubagentSpawn(params)` | Method | Configurar estado compartido para una sesión hija.                                                                        |
| `onSubagentEnded(params)`      | Method | Limpiar después de que termine un subagente.                                                                                 |
| `dispose()`                    | Method | Liberar recursos. Se llama durante el apagado de la puerta de enlace o la recarga del plugin, no por sesión.                           |

### ownsCompaction

`ownsCompaction` controla si la autocompactación integrada en intento de Pi sigue
habilitada para la ejecución:

- `true` — el motor controla el comportamiento de compactación. OpenClaw desactiva la
  autocompactación integrada de Pi para esa ejecución, y la implementación `compact()` del motor es
  responsable de `/compact`, de la compactación de recuperación por desbordamiento y de cualquier
  compactación proactiva que quiera hacer en `afterTurn()`.
- `false` o sin establecer — la autocompactación integrada de Pi todavía puede ejecutarse durante la
  ejecución de la indicación, pero el método `compact()` del motor activo sigue llamándose para
  `/compact` y la recuperación por desbordamiento.

`ownsCompaction: false` **no** significa que OpenClaw vuelva automáticamente a
la ruta de compactación del motor heredado.

Eso significa que hay dos patrones de plugin válidos:

- **Modo propietario** — implementa tu propio algoritmo de compactación y establece
  `ownsCompaction: true`.
- **Modo delegado** — establece `ownsCompaction: false` y haz que `compact()` llame a
  `delegateCompactionToRuntime(...)` desde `openclaw/plugin-sdk/core` para usar
  el comportamiento de compactación integrado de OpenClaw.

Un `compact()` sin operación no es seguro para un motor activo no propietario porque
desactiva la ruta normal de compactación de `/compact` y de recuperación por desbordamiento para esa
ranura de motor.

## Referencia de configuración

```json5
{
  plugins: {
    slots: {
      // Select the active context engine. Default: "legacy".
      // Set to a plugin id to use a plugin engine.
      contextEngine: "legacy",
    },
  },
}
```

La ranura es exclusiva en tiempo de ejecución: solo se resuelve un motor de contexto registrado
para una ejecución u operación de compactación determinada. Otros plugins habilitados
`kind: "context-engine"` todavía pueden cargarse y ejecutar su código de
registro; `plugins.slots.contextEngine` solo selecciona qué id de motor registrado
resuelve OpenClaw cuando necesita un motor de contexto.

## Relación con la compactación y la memoria

- **Compactación** es una de las responsabilidades del motor de contexto. El motor heredado
  delega en el resumen integrado de OpenClaw. Los motores de plugin pueden implementar
  cualquier estrategia de compactación (resúmenes DAG, recuperación vectorial, etc.).
- **Plugins de memoria** (`plugins.slots.memory`) están separados de los motores de contexto.
  Los plugins de memoria proporcionan búsqueda/recuperación; los motores de contexto controlan lo que el
  modelo ve. Pueden trabajar juntos: un motor de contexto podría usar datos del plugin de memoria
  durante el ensamblaje. Los motores de plugin que quieran la ruta activa de indicación de memoria
  deben preferir `buildMemorySystemPromptAddition(...)` de
  `openclaw/plugin-sdk/core`, que convierte las secciones activas de la indicación de memoria
  en un `systemPromptAddition` listo para anteponer. Si un motor necesita un control
  de nivel inferior, aún puede extraer líneas sin procesar de
  `openclaw/plugin-sdk/memory-host-core` mediante
  `buildActiveMemoryPromptSection(...)`.
- **Depuración de sesiones** (recorte de resultados antiguos de herramientas en memoria) sigue ejecutándose
  independientemente de qué motor de contexto esté activo.

## Consejos

- Usa `openclaw doctor` para verificar que tu motor se está cargando correctamente.
- Si cambias de motor, las sesiones existentes continúan con su historial actual.
  El nuevo motor toma el control en las ejecuciones futuras.
- Los errores del motor se registran y se muestran en los diagnósticos. Si un motor de plugin
  no logra registrarse o no se puede resolver el id de motor seleccionado, OpenClaw
  no recurre automáticamente a otro; las ejecuciones fallan hasta que corrijas el plugin o
  cambies `plugins.slots.contextEngine` de nuevo a `"legacy"`.
- Para desarrollo, usa `openclaw plugins install -l ./my-engine` para enlazar un
  directorio de plugin local sin copiarlo.

Ver también: [Compactación](/es/concepts/compaction), [Contexto](/es/concepts/context),
[Plugins](/es/tools/plugin), [Manifiesto de plugin](/es/plugins/manifest).

## Relacionado

- [Contexto](/es/concepts/context) — cómo se construye el contexto para los turnos del agente
- [Arquitectura de plugins](/es/plugins/architecture) — registrar plugins de motor de contexto
- [Compactación](/es/concepts/compaction) — resumir conversaciones largas
