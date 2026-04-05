---
read_when:
    - Quieres trabajos programados y activaciones
    - Estás depurando la ejecución y los registros de cron
summary: Referencia de la CLI para `openclaw cron` (programar y ejecutar trabajos en segundo plano)
title: cron
x-i18n:
    generated_at: "2026-04-05T12:37:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: f74ec8847835f24b3970f1b260feeb69c7ab6c6ec7e41615cbb73f37f14a8112
    source_path: cli/cron.md
    workflow: 15
---

# `openclaw cron`

Gestiona trabajos cron para el programador del Gateway.

Relacionado:

- Trabajos cron: [Trabajos cron](/automation/cron-jobs)

Consejo: ejecuta `openclaw cron --help` para ver toda la superficie de comandos.

Nota: los trabajos aislados de `cron add` usan `--announce` como entrega predeterminada. Usa `--no-deliver` para mantener
la salida como interna. `--deliver` sigue disponible como alias obsoleto de `--announce`.

Nota: las ejecuciones aisladas propiedad de cron esperan un resumen en texto plano y el ejecutor controla
la ruta final de envío. `--no-deliver` mantiene la ejecución interna; no devuelve la
entrega a la herramienta de mensajes del agente.

Nota: los trabajos de una sola vez (`--at`) se eliminan después del éxito de forma predeterminada. Usa `--keep-after-run` para conservarlos.

Nota: `--session` admite `main`, `isolated`, `current` y `session:<id>`.
Usa `current` para vincular a la sesión activa en el momento de la creación, o `session:<id>` para
una clave de sesión persistente explícita.

Nota: para trabajos CLI de una sola vez, las fechas y horas `--at` sin offset se tratan como UTC a menos que también pases
`--tz <iana>`, que interpreta esa hora de reloj local en la zona horaria indicada.

Nota: los trabajos recurrentes ahora usan retroceso exponencial de reintento tras errores consecutivos (30s → 1m → 5m → 15m → 60m), y luego vuelven al horario normal después de la siguiente ejecución correcta.

Nota: `openclaw cron run` ahora devuelve en cuanto la ejecución manual se pone en cola para su ejecución. Las respuestas correctas incluyen `{ ok: true, enqueued: true, runId }`; usa `openclaw cron runs --id <job-id>` para seguir el resultado final.

Nota: `openclaw cron run <job-id>` fuerza la ejecución de forma predeterminada. Usa `--due` para mantener el
comportamiento anterior de "ejecutar solo si corresponde".

Nota: los turnos cron aislados suprimen respuestas obsoletas solo de confirmación. Si el
primer resultado es solo una actualización provisional de estado y ninguna ejecución descendiente de subagente es
responsable de la respuesta final, cron vuelve a hacer una solicitud una vez para obtener el resultado real
antes de la entrega.

Nota: si una ejecución cron aislada devuelve solo el token silencioso (`NO_REPLY` /
`no_reply`), cron suprime tanto la entrega saliente directa como la ruta alternativa de
resumen en cola, por lo que no se publica nada de vuelta en el chat.

Nota: `cron add|edit --model ...` usa ese modelo permitido seleccionado para el trabajo.
Si el modelo no está permitido, cron avisa y recurre a la selección del modelo
del agente/predeterminado del trabajo. Las cadenas de respaldo configuradas siguen aplicándose, pero una simple
reemplazo de modelo sin una lista explícita de respaldo por trabajo ya no agrega el modelo principal
del agente como un destino adicional oculto de reintento.

Nota: la precedencia de modelo de cron aislado es primero la sustitución de Gmail-hook, luego
`--model` por trabajo, luego cualquier sustitución de modelo almacenada de sesión cron, y después la selección normal
de agente/predeterminada.

Nota: el modo rápido de cron aislado sigue la selección de modelo resuelta en vivo. La configuración del
modelo `params.fastMode` se aplica de forma predeterminada, pero una sustitución `fastMode` almacenada de sesión
sigue teniendo prioridad sobre la configuración.

Nota: si una ejecución aislada lanza `LiveSessionModelSwitchError`, cron conserva el
proveedor/modelo cambiado (y la sustitución de perfil de autenticación cambiada cuando está presente) antes de
reintentar. El bucle externo de reintento está limitado a 2 reintentos de cambio después del intento
inicial y luego aborta en lugar de entrar en un bucle infinito.

Nota: las notificaciones de fallo usan primero `delivery.failureDestination`, luego
`cron.failureDestination` global y, por último, recurren al destino principal de
anuncio del trabajo cuando no se configura un destino explícito de fallo.

Nota: la retención/poda se controla en la configuración:

- `cron.sessionRetention` (predeterminado `24h`) poda las sesiones completadas de ejecuciones aisladas.
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` podan `~/.openclaw/cron/runs/<jobId>.jsonl`.

Nota de actualización: si tienes trabajos cron antiguos anteriores al formato actual de entrega/almacenamiento, ejecuta
`openclaw doctor --fix`. Doctor ahora normaliza campos cron heredados (`jobId`, `schedule.cron`,
campos de entrega de nivel superior incluido el `threadId` heredado, alias de entrega `provider` de payload) y migra trabajos simples de
respaldo webhook `notify: true` a entrega webhook explícita cuando `cron.webhook` está
configurado.

## Ediciones comunes

Actualiza la configuración de entrega sin cambiar el mensaje:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

Deshabilita la entrega para un trabajo aislado:

```bash
openclaw cron edit <job-id> --no-deliver
```

Habilita un contexto de arranque ligero para un trabajo aislado:

```bash
openclaw cron edit <job-id> --light-context
```

Anuncia en un canal específico:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Crea un trabajo aislado con contexto de arranque ligero:

```bash
openclaw cron add \
  --name "Resumen matutino ligero" \
  --cron "0 7 * * *" \
  --session isolated \
  --message "Resume las actualizaciones de la noche." \
  --light-context \
  --no-deliver
```

`--light-context` se aplica solo a trabajos aislados de turnos de agente. Para ejecuciones cron, el modo ligero mantiene vacío el contexto de arranque en lugar de inyectar el conjunto completo de arranque del espacio de trabajo.

Nota sobre la responsabilidad de entrega:

- Los trabajos aislados propiedad de cron siempre enrutan la entrega final visible para el usuario a través del
  ejecutor cron (`announce`, `webhook` o `none` solo interno).
- Si la tarea menciona enviar mensajes a algún destinatario externo, el agente debe
  describir el destino previsto en su resultado en lugar de intentar enviarlo
  directamente.

## Comandos comunes de administración

Ejecución manual:

```bash
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron runs --id <job-id> --limit 50
```

Redirección de agente/sesión:

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

Ajustes de entrega:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

Nota sobre la entrega de fallos:

- `delivery.failureDestination` es compatible con trabajos aislados.
- Los trabajos de sesión principal solo pueden usar `delivery.failureDestination` cuando la
  entrega principal es `webhook`.
- Si no configuras ningún destino de fallo y el trabajo ya anuncia en un
  canal, las notificaciones de fallo reutilizan ese mismo destino de anuncio.
