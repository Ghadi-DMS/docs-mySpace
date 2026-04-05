---
read_when:
    - Quieres entender cómo se relaciona Task Flow con las tareas en segundo plano
    - Te encuentras con Task Flow o el flujo de tareas de openclaw en notas de la versión o documentación
    - Quieres inspeccionar o gestionar el estado duradero del flujo
summary: Capa de orquestación de flujos de Task Flow por encima de las tareas en segundo plano
title: Task Flow
x-i18n:
    generated_at: "2026-04-05T12:34:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 172871206b839845db807d9c627015890f7733b862e276853d5dbfbe29e03883
    source_path: automation/taskflow.md
    workflow: 15
---

# Task Flow

Task Flow es el sustrato de orquestación de flujos que se sitúa por encima de las [tareas en segundo plano](/automation/tasks). Gestiona flujos duraderos de varios pasos con su propio estado, seguimiento de revisiones y semántica de sincronización, mientras que las tareas individuales siguen siendo la unidad de trabajo desacoplado.

## Cuándo usar Task Flow

Usa Task Flow cuando el trabajo abarque varios pasos secuenciales o ramificados y necesites un seguimiento duradero del progreso entre reinicios del gateway. Para operaciones individuales en segundo plano, una [tarea](/automation/tasks) simple es suficiente.

| Escenario                             | Uso                    |
| ------------------------------------- | ---------------------- |
| Trabajo único en segundo plano        | Tarea simple           |
| Canalización de varios pasos (A, B y C) | Task Flow (gestionado) |
| Observar tareas creadas externamente  | Task Flow (reflejado)  |
| Recordatorio de una sola vez          | Trabajo cron           |

## Modos de sincronización

### Modo gestionado

Task Flow controla el ciclo de vida de extremo a extremo. Crea tareas como pasos del flujo, las lleva hasta su finalización y hace avanzar el estado del flujo automáticamente.

Ejemplo: un flujo de informe semanal que (1) recopila datos, (2) genera el informe y (3) lo entrega. Task Flow crea cada paso como una tarea en segundo plano, espera a que finalice y luego pasa al siguiente paso.

```
Flow: weekly-report
  Step 1: gather-data     → task created → succeeded
  Step 2: generate-report → task created → succeeded
  Step 3: deliver         → task created → running
```

### Modo reflejado

Task Flow observa tareas creadas externamente y mantiene el estado del flujo sincronizado sin asumir el control de la creación de tareas. Esto resulta útil cuando las tareas se originan en trabajos cron, comandos de CLI u otras fuentes, y quieres una vista unificada de su progreso como flujo.

Ejemplo: tres trabajos cron independientes que en conjunto forman una rutina de "operaciones matutinas". Un flujo reflejado sigue su progreso colectivo sin controlar cuándo ni cómo se ejecutan.

## Estado duradero y seguimiento de revisiones

Cada flujo conserva su propio estado y realiza un seguimiento de las revisiones para que el progreso sobreviva a los reinicios del gateway. El seguimiento de revisiones permite detectar conflictos cuando varias fuentes intentan hacer avanzar el mismo flujo de forma simultánea.

## Comportamiento de cancelación

`openclaw tasks flow cancel` establece una intención de cancelación persistente en el flujo. Las tareas activas dentro del flujo se cancelan y no se inician pasos nuevos. La intención de cancelación persiste entre reinicios, por lo que un flujo cancelado sigue cancelado aunque el gateway se reinicie antes de que hayan terminado todas las tareas secundarias.

## Comandos de CLI

```bash
# List active and recent flows
openclaw tasks flow list

# Show details for a specific flow
openclaw tasks flow show <lookup>

# Cancel a running flow and its active tasks
openclaw tasks flow cancel <lookup>
```

| Comando                           | Descripción                                         |
| --------------------------------- | --------------------------------------------------- |
| `openclaw tasks flow list`        | Muestra los flujos rastreados con estado y modo de sincronización |
| `openclaw tasks flow show <id>`   | Inspecciona un flujo por id de flujo o clave de búsqueda |
| `openclaw tasks flow cancel <id>` | Cancela un flujo en ejecución y sus tareas activas  |

## Cómo se relacionan los flujos con las tareas

Los flujos coordinan tareas, no las sustituyen. Un solo flujo puede controlar varias tareas en segundo plano durante su ciclo de vida. Usa `openclaw tasks` para inspeccionar registros de tareas individuales y `openclaw tasks flow` para inspeccionar el flujo de orquestación.

## Relacionado

- [Tareas en segundo plano](/automation/tasks) — el registro de trabajo desacoplado que coordinan los flujos
- [CLI: tasks](/cli/index#tasks) — referencia de comandos de CLI para `openclaw tasks flow`
- [Resumen de automatización](/automation) — todos los mecanismos de automatización de un vistazo
- [Trabajos cron](/automation/cron-jobs) — trabajos programados que pueden alimentar flujos
