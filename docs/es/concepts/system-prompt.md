---
read_when:
    - Edición del texto del prompt del sistema, la lista de herramientas o las secciones de hora/Heartbeat
    - Cambio del comportamiento de arranque del espacio de trabajo o de inyección de Skills
summary: Qué contiene el prompt del sistema de OpenClaw y cómo se ensambla
title: Prompt del sistema
x-i18n:
    generated_at: "2026-04-18T04:59:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: e60705994cebdd9768926168cb1c6d17ab717d7ff02353a5d5e7478ba8191cab
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt del sistema

OpenClaw crea un prompt del sistema personalizado para cada ejecución del agente. El prompt es **propiedad de OpenClaw** y no usa el prompt predeterminado de pi-coding-agent.

El prompt es ensamblado por OpenClaw e inyectado en cada ejecución del agente.

Los plugins de proveedores pueden aportar guía de prompt consciente de la caché sin reemplazar el prompt completo propiedad de OpenClaw. El runtime del proveedor puede:

- reemplazar un pequeño conjunto de secciones principales con nombre (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- inyectar un **prefijo estable** por encima del límite de caché del prompt
- inyectar un **sufijo dinámico** por debajo del límite de caché del prompt

Usa aportes propiedad del proveedor para ajustes específicos de familias de modelos. Mantén la mutación de prompt heredada de `before_prompt_build` para compatibilidad o para cambios de prompt realmente globales, no para el comportamiento normal del proveedor.

## Estructura

El prompt es intencionalmente compacto y usa secciones fijas:

- **Herramientas**: recordatorio de la fuente de verdad de herramientas estructuradas más guía de uso de herramientas en tiempo de ejecución.
- **Seguridad**: recordatorio breve de barreras de protección para evitar conductas de búsqueda de poder o evasión de supervisión.
- **Skills** (cuando están disponibles): le dice al modelo cómo cargar instrucciones de Skills bajo demanda.
- **Autoactualización de OpenClaw**: cómo inspeccionar la configuración de forma segura con
  `config.schema.lookup`, parchear la configuración con `config.patch`, reemplazar la configuración completa con `config.apply` y ejecutar `update.run` solo cuando el usuario lo solicite explícitamente. La herramienta `gateway`, solo para propietarios, también se niega a reescribir
  `tools.exec.ask` / `tools.exec.security`, incluidas las alias heredadas `tools.bash.*`
  que se normalizan a esas rutas de ejecución protegidas.
- **Espacio de trabajo**: directorio de trabajo (`agents.defaults.workspace`).
- **Documentación**: ruta local a la documentación de OpenClaw (repositorio o paquete npm) y cuándo leerla.
- **Archivos del espacio de trabajo (inyectados)**: indica que los archivos de arranque están incluidos más abajo.
- **Sandbox** (cuando está habilitado): indica el runtime aislado, las rutas del sandbox y si la ejecución con privilegios elevados está disponible.
- **Fecha y hora actuales**: hora local del usuario, zona horaria y formato de hora.
- **Etiquetas de respuesta**: sintaxis opcional de etiquetas de respuesta para proveedores compatibles.
- **Heartbeats**: prompt de Heartbeat y comportamiento de confirmación, cuando los Heartbeats están habilitados para el agente predeterminado.
- **Runtime**: host, SO, node, modelo, raíz del repositorio (cuando se detecta), nivel de razonamiento (una línea).
- **Razonamiento**: nivel de visibilidad actual + sugerencia para alternar con /reasoning.

La sección Herramientas también incluye guía de runtime para trabajo de larga duración:

- usar Cron para seguimientos futuros (`check back later`, recordatorios, trabajo recurrente)
  en lugar de bucles de suspensión con `exec`, trucos de retraso con `yieldMs` o sondeos repetidos de `process`
- usar `exec` / `process` solo para comandos que empiezan ahora y continúan ejecutándose
  en segundo plano
- cuando el despertar automático al completarse está habilitado, iniciar el comando una sola vez y confiar en
  la vía de despertar basada en push cuando emita salida o falle
- usar `process` para registros, estado, entrada o intervención cuando necesites
  inspeccionar un comando en ejecución
- si la tarea es más grande, preferir `sessions_spawn`; la finalización de subagentes está
  basada en push y se anuncia automáticamente de vuelta al solicitante
- no sondear `subagents list` / `sessions_list` en un bucle solo para esperar
  la finalización

Cuando la herramienta experimental `update_plan` está habilitada, Herramientas también le dice al
modelo que la use solo para trabajo no trivial de varios pasos, que mantenga exactamente un
paso `in_progress` y que evite repetir el plan completo después de cada actualización.

Las barreras de seguridad del prompt del sistema son orientativas. Guían el comportamiento del modelo, pero no aplican la política. Usa la política de herramientas, las aprobaciones de ejecución, el sandboxing y las listas de permitidos de canales para la aplicación estricta; los operadores pueden desactivarlas por diseño.

En canales con tarjetas o botones de aprobación nativos, el prompt de runtime ahora le dice al
agente que confíe primero en esa interfaz de aprobación nativa. Solo debe incluir un comando manual
`/approve` cuando el resultado de la herramienta diga que las aprobaciones en el chat no están disponibles o
que la aprobación manual es la única vía.

## Modos del prompt

OpenClaw puede renderizar prompts del sistema más pequeños para subagentes. El runtime establece un
`promptMode` para cada ejecución (no es una configuración orientada al usuario):

- `full` (predeterminado): incluye todas las secciones anteriores.
- `minimal`: se usa para subagentes; omite **Skills**, **Recuperación de memoria**, **Autoactualización de OpenClaw**, **Alias de modelos**, **Identidad del usuario**, **Etiquetas de respuesta**,
  **Mensajería**, **Respuestas silenciosas** y **Heartbeats**. Herramientas, **Seguridad**,
  Espacio de trabajo, Sandbox, Fecha y hora actuales (cuando se conocen), Runtime y el
  contexto inyectado siguen estando disponibles.
- `none`: devuelve solo la línea base de identidad.

Cuando `promptMode=minimal`, los prompts inyectados adicionales se etiquetan como **Contexto del subagente**
en lugar de **Contexto del chat grupal**.

## Inyección del arranque del espacio de trabajo

Los archivos de arranque se recortan y se agregan bajo **Contexto del proyecto** para que el modelo vea el contexto de identidad y perfil sin necesitar lecturas explícitas:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en espacios de trabajo completamente nuevos)
- `MEMORY.md` cuando está presente; en caso contrario, `memory.md` como alternativa en minúsculas

Todos estos archivos se **inyectan en la ventana de contexto** en cada turno, a menos que
se aplique una compuerta específica del archivo. `HEARTBEAT.md` se omite en ejecuciones normales cuando
los Heartbeats están deshabilitados para el agente predeterminado o
`agents.defaults.heartbeat.includeSystemPromptSection` es false. Mantén los archivos inyectados concisos,
especialmente `MEMORY.md`, que puede crecer con el tiempo y provocar un uso de contexto inesperadamente alto y una Compaction más frecuente.

> **Nota:** los archivos diarios `memory/*.md` **no** forman parte del Contexto del proyecto de arranque normal.
> En turnos normales, se accede a ellos bajo demanda mediante las herramientas
> `memory_search` y `memory_get`, por lo que no cuentan contra la ventana de
> contexto a menos que el modelo los lea explícitamente. Los turnos simples `/new` y
> `/reset` son la excepción: el runtime puede anteponer memoria diaria reciente
> como un bloque único de contexto de inicio para ese primer turno.

Los archivos grandes se truncan con un marcador. El tamaño máximo por archivo está controlado por
`agents.defaults.bootstrapMaxChars` (predeterminado: 12000). El contenido total de arranque inyectado
entre archivos está limitado por `agents.defaults.bootstrapTotalMaxChars`
(predeterminado: 60000). Los archivos ausentes inyectan un marcador corto de archivo faltante. Cuando ocurre truncamiento,
OpenClaw puede inyectar un bloque de advertencia en Contexto del proyecto; controla esto con
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
predeterminado: `once`).

Las sesiones de subagentes solo inyectan `AGENTS.md` y `TOOLS.md` (los demás archivos de arranque
se filtran para mantener pequeño el contexto del subagente).

Los hooks internos pueden interceptar este paso mediante `agent:bootstrap` para mutar o reemplazar
los archivos de arranque inyectados (por ejemplo, reemplazando `SOUL.md` por una persona alternativa).

Si quieres que el agente suene menos genérico, empieza con
[Guía de personalidad de SOUL.md](/es/concepts/soul).

Para inspeccionar cuánto aporta cada archivo inyectado (sin procesar frente a inyectado, truncamiento, además de la sobrecarga del esquema de herramientas), usa `/context list` o `/context detail`. Consulta [Contexto](/es/concepts/context).

## Manejo del tiempo

El prompt del sistema incluye una sección dedicada de **Fecha y hora actuales** cuando se conoce la
zona horaria del usuario. Para mantener estable la caché del prompt, ahora solo incluye la
**zona horaria** (sin reloj dinámico ni formato de hora).

Usa `session_status` cuando el agente necesite la hora actual; la tarjeta de estado
incluye una línea de marca de tiempo. La misma herramienta también puede establecer opcionalmente una
anulación de modelo por sesión (`model=default` la borra).

Configura con:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Consulta [Fecha y hora](/es/date-time) para ver todos los detalles del comportamiento.

## Skills

Cuando existen Skills elegibles, OpenClaw inyecta una **lista compacta de Skills disponibles**
(`formatSkillsForPrompt`) que incluye la **ruta del archivo** para cada Skill. El
prompt le indica al modelo que use `read` para cargar el SKILL.md en la ubicación
indicada (espacio de trabajo, administrado o empaquetado). Si no hay Skills elegibles, se omite la sección
Skills.

La elegibilidad incluye compuertas de metadatos de Skills, comprobaciones del entorno/configuración de runtime
y la lista de permitidos efectiva de Skills del agente cuando `agents.defaults.skills` o
`agents.list[].skills` está configurado.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

Esto mantiene pequeño el prompt base mientras sigue permitiendo el uso dirigido de Skills.

El presupuesto de la lista de Skills pertenece al subsistema de Skills:

- Predeterminado global: `skills.limits.maxSkillsPromptChars`
- Anulación por agente: `agents.list[].skillsLimits.maxSkillsPromptChars`

Los extractos de runtime genéricos limitados usan una superficie diferente:

- `agents.defaults.contextLimits.*`
- `agents.list[].contextLimits.*`

Esa separación mantiene el dimensionamiento de Skills separado del dimensionamiento de lectura/inyección del runtime, como
`memory_get`, resultados de herramientas en vivo y actualizaciones de AGENTS.md posteriores a la Compaction.

## Documentación

Cuando está disponible, el prompt del sistema incluye una sección de **Documentación** que apunta al
directorio local de documentación de OpenClaw (ya sea `docs/` en el espacio de trabajo del repositorio o la documentación del paquete npm empaquetado) y también menciona la réplica pública, el repositorio fuente, el Discord de la comunidad y
ClawHub ([https://clawhub.ai](https://clawhub.ai)) para el descubrimiento de Skills. El prompt le indica al modelo que consulte primero la documentación local
para el comportamiento, los comandos, la configuración o la arquitectura de OpenClaw, y que ejecute
`openclaw status` por sí mismo cuando sea posible (preguntando al usuario solo cuando no tenga acceso).
