---
read_when:
    - Editando el texto del prompt del sistema, la lista de herramientas o las secciones de hora/heartbeat
    - Cambiando el comportamiento de bootstrap del espacio de trabajo o de inyección de Skills
summary: Qué contiene el prompt del sistema de OpenClaw y cómo se ensambla
title: Prompt del sistema
x-i18n:
    generated_at: "2026-04-05T12:41:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: f86b2fa496b183b64e86e6ddc493e4653ff8c9727d813fe33c8f8320184d022f
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt del sistema

OpenClaw construye un prompt del sistema personalizado para cada ejecución de agente. El prompt **pertenece a OpenClaw** y no usa el prompt predeterminado de pi-coding-agent.

El prompt lo ensambla OpenClaw y se inyecta en cada ejecución del agente.

## Estructura

El prompt es intencionalmente compacto y usa secciones fijas:

- **Tooling**: lista actual de herramientas + descripciones breves.
- **Safety**: recordatorio breve de guardarraíles para evitar comportamiento de búsqueda de poder o eludir la supervisión.
- **Skills** (cuando están disponibles): indica al modelo cómo cargar instrucciones de Skills bajo demanda.
- **OpenClaw Self-Update**: cómo inspeccionar la configuración de forma segura con
  `config.schema.lookup`, aplicar parches a la configuración con `config.patch`, reemplazar toda la
  configuración con `config.apply` y ejecutar `update.run` solo cuando el usuario lo solicite
  explícitamente. La herramienta `gateway`, solo para el propietario, también se niega a reescribir
  `tools.exec.ask` / `tools.exec.security`, incluidas las alias heredadas `tools.bash.*`
  que se normalizan a esas rutas protegidas de exec.
- **Workspace**: directorio de trabajo (`agents.defaults.workspace`).
- **Documentation**: ruta local a la documentación de OpenClaw (repositorio o paquete npm) y cuándo leerla.
- **Workspace Files (injected)**: indica que los archivos bootstrap se incluyen a continuación.
- **Sandbox** (cuando está habilitado): indica el entorno de ejecución en sandbox, las rutas del sandbox y si exec elevado está disponible.
- **Current Date & Time**: hora local del usuario, zona horaria y formato de hora.
- **Reply Tags**: sintaxis opcional de etiquetas de respuesta para proveedores compatibles.
- **Heartbeats**: prompt de heartbeat y comportamiento de confirmación.
- **Runtime**: host, SO, node, modelo, raíz del repositorio (cuando se detecta), nivel de thinking (una línea).
- **Reasoning**: nivel actual de visibilidad + pista del alternador `/reasoning`.

La sección Tooling también incluye orientación de ejecución para trabajo de larga duración:

- usa cron para seguimientos futuros (`check back later`, recordatorios, trabajo recurrente)
  en lugar de bucles `sleep` de `exec`, trucos de retardo `yieldMs` o sondeo repetido de `process`
- usa `exec` / `process` solo para comandos que comienzan ahora y continúan ejecutándose
  en segundo plano
- cuando la activación automática al completarse está habilitada, inicia el comando una vez y confía en
  la ruta de activación basada en push cuando emita salida o falle
- usa `process` para registros, estado, entrada o intervención cuando necesites
  inspeccionar un comando en ejecución
- si la tarea es más grande, prefiere `sessions_spawn`; la finalización de subagentes se basa en
  push y se anuncia automáticamente de vuelta al solicitante
- no hagas sondeo en bucle de `subagents list` / `sessions_list` solo para esperar
  a que se complete

Los guardarraíles de seguridad del prompt del sistema son orientativos. Guían el comportamiento del modelo, pero no aplican políticas. Usa la política de herramientas, aprobaciones de exec, sandboxing y listas de permitidos del canal para una aplicación estricta; los operadores pueden deshabilitarlos por diseño.

En canales con tarjetas/botones nativos de aprobación, el prompt de ejecución ahora le indica al
agente que confíe primero en esa interfaz nativa de aprobación. Solo debería incluir un comando manual
`/approve` cuando el resultado de la herramienta diga que las aprobaciones por chat no están disponibles o
que la aprobación manual es la única vía.

## Modos del prompt

OpenClaw puede renderizar prompts del sistema más pequeños para subagentes. El entorno de ejecución establece un
`promptMode` para cada ejecución (no es una configuración orientada al usuario):

- `full` (predeterminado): incluye todas las secciones anteriores.
- `minimal`: se usa para subagentes; omite **Skills**, **Memory Recall**, **OpenClaw
  Self-Update**, **Model Aliases**, **User Identity**, **Reply Tags**,
  **Messaging**, **Silent Replies** y **Heartbeats**. Tooling, **Safety**,
  Workspace, Sandbox, Current Date & Time (cuando se conoce), Runtime y el
  contexto inyectado siguen disponibles.
- `none`: devuelve solo la línea base de identidad.

Cuando `promptMode=minimal`, los prompts inyectados adicionales se etiquetan como **Subagent
Context** en lugar de **Group Chat Context**.

## Inyección de bootstrap del espacio de trabajo

Los archivos bootstrap se recortan y se añaden bajo **Project Context** para que el modelo vea el contexto de identidad y perfil sin necesitar lecturas explícitas:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en espacios de trabajo completamente nuevos)
- `MEMORY.md` cuando existe, o bien `memory.md` como respaldo en minúsculas

Todos estos archivos **se inyectan en la ventana de contexto** en cada turno, lo que
significa que consumen tokens. Mantenlos concisos, especialmente `MEMORY.md`, que puede
crecer con el tiempo y llevar a un uso de contexto inesperadamente alto y a una compactación
más frecuente.

> **Nota:** los archivos diarios `memory/*.md` **no** se inyectan automáticamente. Se
> accede a ellos bajo demanda mediante las herramientas `memory_search` y `memory_get`, por lo que
> no cuentan contra la ventana de contexto a menos que el modelo los lea explícitamente.

Los archivos grandes se truncan con un marcador. El tamaño máximo por archivo se controla mediante
`agents.defaults.bootstrapMaxChars` (predeterminado: 20000). El contenido total de bootstrap
inyectado entre archivos está limitado por `agents.defaults.bootstrapTotalMaxChars`
(predeterminado: 150000). Los archivos faltantes inyectan un marcador breve de archivo faltante. Cuando ocurre truncamiento,
OpenClaw puede inyectar un bloque de advertencia en Project Context; contrólalo con
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
predeterminado: `once`).

Las sesiones de subagentes solo inyectan `AGENTS.md` y `TOOLS.md` (los demás archivos bootstrap
se filtran para mantener pequeño el contexto del subagente).

Los hooks internos pueden interceptar este paso mediante `agent:bootstrap` para mutar o reemplazar
los archivos bootstrap inyectados (por ejemplo, intercambiar `SOUL.md` por una persona alternativa).

Si quieres que el agente suene menos genérico, empieza con
[Guía de personalidad de SOUL.md](/concepts/soul).

Para inspeccionar cuánto aporta cada archivo inyectado (sin procesar frente a inyectado, truncamiento, además de la sobrecarga del esquema de herramientas), usa `/context list` o `/context detail`. Consulta [Contexto](/concepts/context).

## Manejo del tiempo

El prompt del sistema incluye una sección dedicada **Current Date & Time** cuando se conoce la
zona horaria del usuario. Para mantener estable la caché del prompt, ahora solo incluye
la **zona horaria** (sin reloj dinámico ni formato de hora).

Usa `session_status` cuando el agente necesite la hora actual; la tarjeta de estado
incluye una línea de marca de tiempo. La misma herramienta también puede establecer opcionalmente una
anulación de modelo por sesión (`model=default` la borra).

Configúralo con:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Consulta [Fecha y hora](/date-time) para todos los detalles del comportamiento.

## Skills

Cuando existen Skills elegibles, OpenClaw inyecta una **lista compacta de Skills disponibles**
(`formatSkillsForPrompt`) que incluye la **ruta del archivo** para cada Skill. El
prompt instruye al modelo a usar `read` para cargar el `SKILL.md` en la ubicación
indicada (espacio de trabajo, gestionada o integrada). Si no hay Skills elegibles, la
sección Skills se omite.

La elegibilidad incluye compuertas de metadatos de Skills, comprobaciones del entorno/configuración de ejecución
y la lista de permitidos efectiva de Skills del agente cuando se configura `agents.defaults.skills` o
`agents.list[].skills`.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

Esto mantiene pequeño el prompt base y al mismo tiempo permite un uso dirigido de Skills.

## Documentación

Cuando está disponible, el prompt del sistema incluye una sección **Documentation** que apunta al
directorio local de documentación de OpenClaw (ya sea `docs/` en el espacio de trabajo del repositorio o la documentación del
paquete npm integrado) y también menciona el espejo público, el repositorio fuente, la comunidad de Discord y
ClawHub ([https://clawhub.ai](https://clawhub.ai)) para descubrir Skills. El prompt instruye al modelo a consultar primero la documentación local
para el comportamiento, comandos, configuración o arquitectura de OpenClaw, y a ejecutar
`openclaw status` por sí mismo cuando sea posible (preguntando al usuario solo cuando no tenga acceso).
