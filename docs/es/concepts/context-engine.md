---
read_when:
    - Quieres entender cómo OpenClaw ensambla el contexto del modelo
    - Estás cambiando entre el motor heredado y un motor de plugin
    - Estás creando un plugin de motor de contexto
summary: 'Motor de contexto: ensamblado de contexto conectable, compactación y ciclo de vida de subagentes'
title: Motor de contexto
x-i18n:
    generated_at: "2026-04-05T12:39:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 19fd8cbb0e953f58fd84637fc4ceefc65984312cf2896d338318bc8cf860e6d9
    source_path: concepts/context-engine.md
    workflow: 15
---

# Motor de contexto

Un **motor de contexto** controla cómo OpenClaw construye el contexto del modelo para cada ejecución.
Decide qué mensajes incluir, cómo resumir el historial anterior y cómo
gestionar el contexto a través de los límites de subagentes.

OpenClaw incluye un motor integrado `legacy`. Los plugins pueden registrar
motores alternativos que sustituyen el ciclo de vida activo del motor de contexto.

## Inicio rápido

Comprueba qué motor está activo:

```bash
openclaw doctor
# o inspecciona la configuración directamente:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### Instalar un plugin de motor de contexto

Los plugins de motor de contexto se instalan como cualquier otro plugin de OpenClaw. Instálalo
primero y luego selecciona el motor en el slot:

```bash
# Instalar desde npm
openclaw plugins install @martian-engineering/lossless-claw

# O instalar desde una ruta local (para desarrollo)
openclaw plugins install -l ./my-context-engine
```

Luego habilita el plugin y selecciónalo como motor activo en tu configuración:

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // debe coincidir con el id de motor registrado del plugin
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // La configuración específica del plugin va aquí (consulta la documentación del plugin)
      },
    },
  },
}
```

Reinicia el gateway después de instalarlo y configurarlo.

Para volver al motor integrado, establece `contextEngine` en `"legacy"` (o
elimina la clave por completo: `"legacy"` es el valor predeterminado).

## Cómo funciona

Cada vez que OpenClaw ejecuta un prompt de modelo, el motor de contexto participa en
cuatro puntos del ciclo de vida:

1. **Ingest** — se llama cuando se añade un mensaje nuevo a la sesión. El motor
   puede almacenar o indexar el mensaje en su propio almacén de datos.
2. **Assemble** — se llama antes de cada ejecución del modelo. El motor devuelve un conjunto
   ordenado de mensajes (y un `systemPromptAddition` opcional) que cabe dentro
   del presupuesto de tokens.
3. **Compact** — se llama cuando la ventana de contexto está llena, o cuando el usuario ejecuta
   `/compact`. El motor resume el historial anterior para liberar espacio.
4. **After turn** — se llama después de que se complete una ejecución. El motor puede conservar el estado,
   activar la compactación en segundo plano o actualizar índices.

### Ciclo de vida de subagentes (opcional)

Actualmente OpenClaw llama a un hook del ciclo de vida de subagentes:

- **onSubagentEnded** — limpia cuando una sesión de subagente finaliza o se barre.

El hook `prepareSubagentSpawn` forma parte de la interfaz para uso futuro, pero
el runtime todavía no lo invoca.

### Adición al prompt del sistema

El método `assemble` puede devolver una cadena `systemPromptAddition`. OpenClaw
la antepone al prompt del sistema para la ejecución. Esto permite que los motores inyecten
guías dinámicas de recuerdo, instrucciones de recuperación o sugerencias según el contexto
sin requerir archivos estáticos del espacio de trabajo.

## El motor heredado

El motor integrado `legacy` preserva el comportamiento original de OpenClaw:

- **Ingest**: no-op (el gestor de sesiones se encarga directamente de la persistencia de mensajes).
- **Assemble**: paso directo (la canalización existente sanitize → validate → limit
  del runtime se encarga del ensamblado del contexto).
- **Compact**: delega en la compactación por resumen integrada, que crea
  un único resumen de los mensajes anteriores y mantiene intactos los mensajes recientes.
- **After turn**: no-op.

El motor heredado no registra herramientas ni proporciona `systemPromptAddition`.

Cuando no se establece `plugins.slots.contextEngine` (o se establece en `"legacy"`), este
motor se usa automáticamente.

## Motores de plugin

Un plugin puede registrar un motor de contexto usando la API de plugins:

```ts
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

    async assemble({ sessionId, messages, tokenBudget }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: "Use lcm_grep to search history...",
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

| Miembro            | Tipo     | Propósito                                                  |
| ------------------ | -------- | ---------------------------------------------------------- |
| `info`             | Propiedad | ID, nombre, versión del motor y si es propietario de la compactación |
| `ingest(params)`   | Método   | Almacenar un solo mensaje                                   |
| `assemble(params)` | Método   | Construir contexto para una ejecución del modelo (devuelve `AssembleResult`) |
| `compact(params)`  | Método   | Resumir/reducir el contexto                                 |

`assemble` devuelve un `AssembleResult` con:

- `messages` — los mensajes ordenados que se enviarán al modelo.
- `estimatedTokens` (obligatorio, `number`) — la estimación del motor del total de
  tokens en el contexto ensamblado. OpenClaw usa esto para decisiones de umbral
  de compactación e informes de diagnóstico.
- `systemPromptAddition` (opcional, `string`) — se antepone al prompt del sistema.

Miembros opcionales:

| Miembro                        | Tipo   | Propósito                                                                                                         |
| ----------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | Método | Inicializar el estado del motor para una sesión. Se llama una vez cuando el motor ve por primera vez una sesión (p. ej., importar historial). |
| `ingestBatch(params)`          | Método | Ingerir un turno completado como lote. Se llama después de completar una ejecución, con todos los mensajes de ese turno de una vez.     |
| `afterTurn(params)`            | Método | Trabajo posterior a la ejecución (conservar estado, activar compactación en segundo plano).                                         |
| `prepareSubagentSpawn(params)` | Método | Configurar estado compartido para una sesión hija.                                                                        |
| `onSubagentEnded(params)`      | Método | Limpiar después de que termine un subagente.                                                                                 |
| `dispose()`                    | Método | Liberar recursos. Se llama durante el apagado del gateway o la recarga del plugin; no por sesión.                           |

### ownsCompaction

`ownsCompaction` controla si la autocompactación integrada en el intento de Pi permanece
habilitada para la ejecución:

- `true` — el motor es propietario del comportamiento de compactación. OpenClaw deshabilita la
  autocompactación integrada de Pi para esa ejecución, y la implementación `compact()` del motor
  es responsable de `/compact`, la compactación de recuperación por desbordamiento y cualquier compactación
  proactiva que quiera hacer en `afterTurn()`.
- `false` o sin establecer — la autocompactación integrada de Pi todavía puede ejecutarse durante la
  ejecución del prompt, pero el método `compact()` del motor activo sigue siendo llamado para
  `/compact` y la recuperación por desbordamiento.

`ownsCompaction: false` **no** significa que OpenClaw recurra automáticamente
a la ruta de compactación del motor heredado.

Eso significa que hay dos patrones de plugin válidos:

- **Modo propietario** — implementa tu propio algoritmo de compactación y establece
  `ownsCompaction: true`.
- **Modo delegado** — establece `ownsCompaction: false` y haz que `compact()` llame a
  `delegateCompactionToRuntime(...)` desde `openclaw/plugin-sdk/core` para usar
  el comportamiento de compactación integrado de OpenClaw.

Un `compact()` no-op no es seguro para un motor activo no propietario porque
deshabilita la ruta normal de compactación de `/compact` y recuperación por desbordamiento para ese
slot de motor.

## Referencia de configuración

```json5
{
  plugins: {
    slots: {
      // Selecciona el motor de contexto activo. Predeterminado: "legacy".
      // Establécelo en un id de plugin para usar un motor de plugin.
      contextEngine: "legacy",
    },
  },
}
```

El slot es exclusivo en tiempo de ejecución: solo se resuelve un motor de contexto registrado
para una determinada ejecución u operación de compactación. Otros plugins habilitados
`kind: "context-engine"` aún pueden cargarse y ejecutar su código de
registro; `plugins.slots.contextEngine` solo selecciona qué id de motor registrado
resuelve OpenClaw cuando necesita un motor de contexto.

## Relación con la compactación y la memoria

- **Compactación** es una responsabilidad del motor de contexto. El motor heredado
  delega en el resumen integrado de OpenClaw. Los motores de plugin pueden implementar
  cualquier estrategia de compactación (resúmenes DAG, recuperación vectorial, etc.).
- **Plugins de memoria** (`plugins.slots.memory`) son independientes de los motores de contexto.
  Los plugins de memoria proporcionan búsqueda/recuperación; los motores de contexto controlan lo
  que ve el modelo. Pueden trabajar juntos: un motor de contexto podría usar datos del
  plugin de memoria durante el ensamblado.
- **Poda de sesiones** (recorte en memoria de resultados antiguos de herramientas) sigue ejecutándose
  independientemente del motor de contexto activo.

## Consejos

- Usa `openclaw doctor` para verificar que tu motor se esté cargando correctamente.
- Si cambias de motor, las sesiones existentes continúan con su historial actual.
  El nuevo motor toma el control para las ejecuciones futuras.
- Los errores del motor se registran y se muestran en los diagnósticos. Si un motor de plugin
  no se registra o no se puede resolver el id del motor seleccionado, OpenClaw
  no recurre automáticamente a otro; las ejecuciones fallan hasta que corrijas el plugin o
  vuelvas a cambiar `plugins.slots.contextEngine` a `"legacy"`.
- Para desarrollo, usa `openclaw plugins install -l ./my-engine` para vincular un
  directorio de plugin local sin copiarlo.

Consulta también: [Compactación](/concepts/compaction), [Context](/concepts/context),
[Plugins](/tools/plugin), [Manifiesto del plugin](/plugins/manifest).

## Relacionado

- [Context](/concepts/context) — cómo se construye el contexto para los turnos del agente
- [Arquitectura de plugins](/plugins/architecture) — registrar plugins de motor de contexto
- [Compactación](/concepts/compaction) — resumir conversaciones largas
