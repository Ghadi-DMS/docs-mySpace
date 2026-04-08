---
read_when:
    - Editar el texto del prompt del sistema, la lista de herramientas o las secciones de hora/heartbeat
    - Cambiar el arranque del espacio de trabajo o el comportamiento de inyección de Skills
summary: Qué contiene el prompt del sistema de OpenClaw y cómo se ensambla
title: Prompt del sistema
x-i18n:
    generated_at: "2026-04-08T02:14:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: e55fc886bc8ec47584d07c9e60dfacd964dc69c7db976ea373877dc4fe09a79a
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt del sistema

OpenClaw crea un prompt del sistema personalizado para cada ejecución del agente. El prompt es **propiedad de OpenClaw** y no usa el prompt predeterminado de pi-coding-agent.

El prompt es ensamblado por OpenClaw e inyectado en cada ejecución del agente.

Los plugins de proveedor pueden aportar orientación del prompt compatible con caché sin reemplazar
el prompt completo propiedad de OpenClaw. El runtime del proveedor puede:

- reemplazar un pequeño conjunto de secciones centrales con nombre (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- inyectar un **prefijo estable** por encima del límite de caché del prompt
- inyectar un **sufijo dinámico** por debajo del límite de caché del prompt

Usa contribuciones propiedad del proveedor para el ajuste específico de familias de modelos. Conserva la mutación heredada del prompt
`before_prompt_build` por compatibilidad o para cambios de prompt verdaderamente globales,
no para el comportamiento normal del proveedor.

## Estructura

El prompt es intencionalmente compacto y usa secciones fijas:

- **Herramientas**: recordatorio de la fuente de verdad de herramientas estructuradas más orientación de runtime para el uso de herramientas.
- **Seguridad**: recordatorio breve de barreras de protección para evitar comportamientos de búsqueda de poder o eludir la supervisión.
- **Skills** (cuando estén disponibles): indica al modelo cómo cargar instrucciones de Skills bajo demanda.
- **Autoactualización de OpenClaw**: cómo inspeccionar la configuración de forma segura con
  `config.schema.lookup`, aplicar parches a la configuración con `config.patch`, reemplazar la
  configuración completa con `config.apply` y ejecutar `update.run` solo a petición explícita del usuario.
  La herramienta `gateway`, solo para el propietario, también se niega a reescribir
  `tools.exec.ask` / `tools.exec.security`, incluidas las alias heredadas `tools.bash.*`
  que se normalizan a esas rutas exec protegidas.
- **Espacio de trabajo**: directorio de trabajo (`agents.defaults.workspace`).
- **Documentación**: ruta local a la documentación de OpenClaw (repositorio o paquete npm) y cuándo leerla.
- **Archivos del espacio de trabajo (inyectados)**: indica que los archivos de arranque se incluyen a continuación.
- **Sandbox** (cuando está habilitado): indica el runtime en sandbox, las rutas del sandbox y si exec con privilegios elevados está disponible.
- **Fecha y hora actuales**: hora local del usuario, zona horaria y formato de hora.
- **Etiquetas de respuesta**: sintaxis opcional de etiquetas de respuesta para los proveedores compatibles.
- **Heartbeats**: prompt y comportamiento de confirmación del heartbeat, cuando los heartbeats están habilitados para el agente predeterminado.
- **Runtime**: host, sistema operativo, node, raíz del repositorio (cuando se detecta), nivel de razonamiento (una línea).
- **Razonamiento**: nivel actual de visibilidad + pista sobre el cambio con /reasoning.

La sección Herramientas también incluye orientación de runtime para trabajos de larga duración:

- usa cron para seguimientos futuros (`check back later`, recordatorios, trabajo recurrente)
  en lugar de bucles de espera con `exec`, trucos de retraso con `yieldMs` o sondeo repetido de `process`
- usa `exec` / `process` solo para comandos que empiezan ahora y continúan ejecutándose
  en segundo plano
- cuando la reactivación automática al completarse está habilitada, inicia el comando una sola vez y confía en
  la ruta de reactivación basada en push cuando emita salida o falle
- usa `process` para registros, estado, entrada o intervención cuando necesites
  inspeccionar un comando en ejecución
- si la tarea es más grande, prefiere `sessions_spawn`; la finalización del subagente es
  basada en push y se anuncia automáticamente de vuelta al solicitante
- no sondees `subagents list` / `sessions_list` en un bucle solo para esperar
  la finalización

Cuando la herramienta experimental `update_plan` está habilitada, Herramientas también le indica al
modelo que la use solo para trabajo no trivial de varios pasos, que mantenga exactamente un paso
`in_progress` y que evite repetir el plan completo después de cada actualización.

Las barreras de protección de seguridad en el prompt del sistema son orientativas. Guían el comportamiento del modelo, pero no aplican la política. Usa la política de herramientas, las aprobaciones de exec, el sandboxing y las listas permitidas de canales para la aplicación estricta; los operadores pueden desactivar esto por diseño.

En los canales con tarjetas o botones de aprobación nativos, el prompt de runtime ahora le indica al
agente que confíe primero en esa UI de aprobación nativa. Solo debe incluir un comando manual
`/approve` cuando el resultado de la herramienta diga que las aprobaciones en el chat no están disponibles o
que la aprobación manual es la única vía.

## Modos de prompt

OpenClaw puede renderizar prompts del sistema más pequeños para subagentes. El runtime establece un
`promptMode` para cada ejecución (no es una configuración visible para el usuario):

- `full` (predeterminado): incluye todas las secciones anteriores.
- `minimal`: se usa para subagentes; omite **Skills**, **Memory Recall**, **Autoactualización de OpenClaw**,
  **Alias de modelos**, **Identidad del usuario**, **Etiquetas de respuesta**,
  **Mensajería**, **Respuestas silenciosas** y **Heartbeats**. Herramientas, **Seguridad**,
  Espacio de trabajo, Sandbox, Fecha y hora actuales (cuando se conocen), Runtime y el
  contexto inyectado siguen estando disponibles.
- `none`: devuelve solo la línea base de identidad.

Cuando `promptMode=minimal`, los prompts inyectados adicionales se etiquetan como **Contexto del subagente**
en lugar de **Contexto del chat grupal**.

## Inyección del arranque del espacio de trabajo

Los archivos de arranque se recortan y se anexan bajo **Contexto del proyecto** para que el modelo vea el contexto de identidad y perfil sin necesitar lecturas explícitas:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en espacios de trabajo completamente nuevos)
- `MEMORY.md` cuando está presente; en caso contrario, `memory.md` como alternativa en minúsculas

Todos estos archivos se **inyectan en la ventana de contexto** en cada turno, salvo que se aplique una restricción específica del archivo. `HEARTBEAT.md` se omite en ejecuciones normales cuando
los heartbeats están deshabilitados para el agente predeterminado o
`agents.defaults.heartbeat.includeSystemPromptSection` es false. Mantén los archivos inyectados concisos,
especialmente `MEMORY.md`, que puede crecer con el tiempo y provocar
un uso de contexto inesperadamente alto y una compactación más frecuente.

> **Nota:** los archivos diarios `memory/*.md` **no** se inyectan automáticamente. Se
> accede a ellos bajo demanda mediante las herramientas `memory_search` y `memory_get`, por lo que
> no cuentan contra la ventana de contexto a menos que el modelo los lea explícitamente.

Los archivos grandes se truncan con un marcador. El tamaño máximo por archivo está controlado por
`agents.defaults.bootstrapMaxChars` (predeterminado: 20000). El contenido total de arranque inyectado
entre archivos está limitado por `agents.defaults.bootstrapTotalMaxChars`
(predeterminado: 150000). Los archivos faltantes inyectan un breve marcador de archivo faltante. Cuando ocurre truncamiento,
OpenClaw puede inyectar un bloque de advertencia en Contexto del proyecto; esto se controla con
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
predeterminado: `once`).

Las sesiones de subagentes solo inyectan `AGENTS.md` y `TOOLS.md` (los demás archivos de arranque
se filtran para mantener pequeño el contexto del subagente).

Los hooks internos pueden interceptar este paso mediante `agent:bootstrap` para mutar o reemplazar
los archivos de arranque inyectados (por ejemplo, cambiando `SOUL.md` por una persona alternativa).

Si quieres que el agente suene menos genérico, empieza con
[Guía de personalidad de SOUL.md](/es/concepts/soul).

Para inspeccionar cuánto aporta cada archivo inyectado (sin procesar frente a inyectado, truncamiento y además la sobrecarga del esquema de herramientas), usa `/context list` o `/context detail`. Consulta [Contexto](/es/concepts/context).

## Manejo del tiempo

El prompt del sistema incluye una sección dedicada de **Fecha y hora actuales** cuando se conoce la
zona horaria del usuario. Para mantener la estabilidad de la caché del prompt, ahora solo incluye
la **zona horaria** (sin reloj dinámico ni formato de hora).

Usa `session_status` cuando el agente necesite la hora actual; la tarjeta de estado
incluye una línea de marca temporal. La misma herramienta también puede configurar opcionalmente una anulación del
modelo por sesión (`model=default` la borra).

Configura con:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Consulta [Fecha y hora](/es/date-time) para ver todos los detalles del comportamiento.

## Skills

Cuando existen Skills elegibles, OpenClaw inyecta una **lista compacta de skills disponibles**
(`formatSkillsForPrompt`) que incluye la **ruta del archivo** de cada Skill. El
prompt le indica al modelo que use `read` para cargar el SKILL.md en la ubicación
indicada (espacio de trabajo, gestionado o incluido). Si no hay Skills elegibles, la
sección Skills se omite.

La elegibilidad incluye restricciones de metadatos de Skills, comprobaciones del entorno/configuración de runtime
y la lista efectiva de Skills permitidas del agente cuando `agents.defaults.skills` o
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

Esto mantiene pequeño el prompt base y al mismo tiempo permite un uso específico y dirigido de las Skills.

## Documentación

Cuando está disponible, el prompt del sistema incluye una sección de **Documentación** que apunta al
directorio local de documentación de OpenClaw (ya sea `docs/` en el espacio de trabajo del repositorio o la
documentación incluida en el paquete npm) y también menciona el espejo público, el repositorio fuente, la comunidad de Discord y
ClawHub ([https://clawhub.ai](https://clawhub.ai)) para descubrir skills. El prompt le indica al modelo que consulte primero la documentación local
para el comportamiento, los comandos, la configuración o la arquitectura de OpenClaw, y que ejecute
`openclaw status` por sí mismo cuando sea posible (preguntando al usuario solo cuando no tenga acceso).
