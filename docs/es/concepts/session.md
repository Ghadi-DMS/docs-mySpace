---
read_when:
    - Quieres entender el enrutamiento y el aislamiento de sesiones
    - Quieres configurar el alcance de mensajes directos para configuraciones con varios usuarios
summary: Cómo gestiona OpenClaw las sesiones de conversación
title: Gestión de sesiones
x-i18n:
    generated_at: "2026-04-05T12:40:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: ab985781e54b22a034489dafa4b52cc204b1a5da22ee9b62edc7f6697512cea1
    source_path: concepts/session.md
    workflow: 15
---

# Gestión de sesiones

OpenClaw organiza las conversaciones en **sesiones**. Cada mensaje se enruta a una
sesión según de dónde proviene: mensajes directos, chats de grupo, trabajos cron, etc.

## Cómo se enrutan los mensajes

| Origen          | Comportamiento              |
| --------------- | --------------------------- |
| Mensajes directos | Sesión compartida de forma predeterminada |
| Chats de grupo  | Aislado por grupo           |
| Salas/canales   | Aislado por sala            |
| Trabajos cron   | Sesión nueva por ejecución  |
| Webhooks        | Aislado por hook            |

## Aislamiento de mensajes directos

De forma predeterminada, todos los mensajes directos comparten una sola sesión para mantener la continuidad. Esto está bien para
configuraciones de un solo usuario.

<Warning>
Si varias personas pueden enviar mensajes a tu agente, habilita el aislamiento de mensajes directos. Sin él, todas
las personas usuarias comparten el mismo contexto de conversación: los mensajes privados de Alice serían
visibles para Bob.
</Warning>

**La solución:**

```json5
{
  session: {
    dmScope: "per-channel-peer", // aislar por canal + remitente
  },
}
```

Otras opciones:

- `main` (predeterminado): todos los mensajes directos comparten una sola sesión.
- `per-peer`: aislar por remitente (entre canales).
- `per-channel-peer`: aislar por canal + remitente (recomendado).
- `per-account-channel-peer`: aislar por cuenta + canal + remitente.

<Tip>
Si la misma persona se pone en contacto contigo desde varios canales, usa
`session.identityLinks` para vincular sus identidades y que compartan una sola sesión.
</Tip>

Verifica tu configuración con `openclaw security audit`.

## Ciclo de vida de la sesión

Las sesiones se reutilizan hasta que caducan:

- **Restablecimiento diario** (predeterminado): nueva sesión a las 4:00 a. m. hora local en el host del gateway.
- **Restablecimiento por inactividad** (opcional): nueva sesión tras un período de inactividad. Establece
  `session.reset.idleMinutes`.
- **Restablecimiento manual**: escribe `/new` o `/reset` en el chat. `/new <model>` también
  cambia el modelo.

Cuando están configurados tanto el restablecimiento diario como el de inactividad, prevalece el que venza primero.

## Dónde vive el estado

Todo el estado de la sesión es propiedad del **gateway**. Los clientes de IU consultan el gateway para obtener
los datos de la sesión.

- **Almacén:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **Transcripciones:** `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

## Mantenimiento de sesiones

OpenClaw limita automáticamente el almacenamiento de sesiones con el tiempo. De forma predeterminada, se ejecuta
en modo `warn` (informa de lo que se limpiaría). Establece `session.maintenance.mode`
en `"enforce"` para la limpieza automática:

```json5
{
  session: {
    maintenance: {
      mode: "enforce",
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

Vista previa con `openclaw sessions cleanup --dry-run`.

## Inspeccionar sesiones

- `openclaw status`: ruta del almacén de sesiones y actividad reciente.
- `openclaw sessions --json`: todas las sesiones (filtra con `--active <minutes>`).
- `/status` en el chat: uso de contexto, modelo y conmutadores.
- `/context list`: qué hay en el prompt del sistema.

## Más información

- [Poda de sesiones](/concepts/session-pruning) — recorte de resultados de herramientas
- [Compactación](/concepts/compaction) — resumen de conversaciones largas
- [Herramientas de sesión](/concepts/session-tool) — herramientas del agente para trabajo entre sesiones
- [Análisis profundo de la gestión de sesiones](/reference/session-management-compaction) —
  esquema del almacén, transcripciones, política de envío, metadatos de origen y configuración avanzada
- [Multi-Agent](/concepts/multi-agent) — enrutamiento y aislamiento de sesiones entre agentes
- [Tareas en segundo plano](/automation/tasks) — cómo el trabajo desacoplado crea registros de tareas con referencias de sesión
- [Enrutamiento de canales](/channels/channel-routing) — cómo se enrutan los mensajes entrantes a sesiones
