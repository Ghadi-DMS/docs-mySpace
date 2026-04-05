---
read_when:
    - Quieres activar ejecuciones de agentes desde scripts o la línea de comandos
    - Necesitas entregar respuestas de agentes a un canal de chat de forma programática
summary: Ejecuta turnos de agente desde la CLI y, opcionalmente, entrega respuestas a canales
title: Agent Send
x-i18n:
    generated_at: "2026-04-05T12:54:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 42ea2977e89fb28d2afd07e5f6b1560ad627aea8b72fde36d8e324215c710afc
    source_path: tools/agent-send.md
    workflow: 15
---

# Agent Send

`openclaw agent` ejecuta un solo turno de agente desde la línea de comandos sin necesitar
un mensaje entrante de chat. Úsalo para flujos de trabajo con scripts, pruebas y
entrega programática.

## Inicio rápido

<Steps>
  <Step title="Ejecuta un turno de agente simple">
    ```bash
    openclaw agent --message "What is the weather today?"
    ```

    Esto envía el mensaje a través del Gateway e imprime la respuesta.

  </Step>

  <Step title="Dirígete a un agente o sesión específicos">
    ```bash
    # Dirigirse a un agente específico
    openclaw agent --agent ops --message "Summarize logs"

    # Dirigirse a un número de teléfono (deriva la clave de sesión)
    openclaw agent --to +15555550123 --message "Status update"

    # Reutilizar una sesión existente
    openclaw agent --session-id abc123 --message "Continue the task"
    ```

  </Step>

  <Step title="Entrega la respuesta a un canal">
    ```bash
    # Entregar a WhatsApp (canal predeterminado)
    openclaw agent --to +15555550123 --message "Report ready" --deliver

    # Entregar a Slack
    openclaw agent --agent ops --message "Generate report" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## Indicadores

| Flag                          | Descripción                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| `--message \<text\>`          | Mensaje que se enviará (obligatorio)                         |
| `--to \<dest\>`               | Deriva la clave de sesión desde un destino (teléfono, id de chat) |
| `--agent \<id\>`              | Se dirige a un agente configurado (usa su sesión `main`)     |
| `--session-id \<id\>`         | Reutiliza una sesión existente por id                        |
| `--local`                     | Fuerza el entorno de ejecución integrado local (omite Gateway) |
| `--deliver`                   | Envía la respuesta a un canal de chat                        |
| `--channel \<name\>`          | Canal de entrega (whatsapp, telegram, discord, slack, etc.)  |
| `--reply-to \<target\>`       | Anulación del destino de entrega                             |
| `--reply-channel \<name\>`    | Anulación del canal de entrega                               |
| `--reply-account \<id\>`      | Anulación del id de cuenta de entrega                        |
| `--thinking \<level\>`        | Establece el nivel de thinking (off, minimal, low, medium, high, xhigh) |
| `--verbose \<on\|full\|off\>` | Establece el nivel detallado                                 |
| `--timeout \<seconds\>`       | Anula el tiempo de espera del agente                         |
| `--json`                      | Genera JSON estructurado                                     |

## Comportamiento

- De forma predeterminada, la CLI pasa **por el Gateway**. Agrega `--local` para forzar el
  entorno de ejecución integrado en la máquina actual.
- Si el Gateway no es accesible, la CLI **recurre** a la ejecución integrada local.
- Selección de sesión: `--to` deriva la clave de sesión (los destinos de grupo/canal
  conservan el aislamiento; los chats directos colapsan a `main`).
- Los indicadores de thinking y verbose persisten en el almacenamiento de la sesión.
- Salida: texto sin formato de manera predeterminada, o `--json` para carga útil estructurada + metadatos.

## Ejemplos

```bash
# Turno simple con salida JSON
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json

# Turno con nivel de thinking
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium

# Entregar a un canal diferente al de la sesión
openclaw agent --agent ops --message "Alert" --deliver --reply-channel telegram --reply-to "@admin"
```

## Relacionado

- [Referencia de la CLI de agentes](/cli/agent)
- [Subagentes](/tools/subagents) — generación de subagentes en segundo plano
- [Sesiones](/es/concepts/session) — cómo funcionan las claves de sesión
