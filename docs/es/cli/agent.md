---
read_when:
    - Quieres ejecutar un turno de agente desde scripts (opcionalmente entregar la respuesta)
summary: Referencia de la CLI para `openclaw agent` (envía un turno de agente mediante el Gateway)
title: agent
x-i18n:
    generated_at: "2026-04-05T12:37:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0627f943bc7f3556318008f76dc6150788cf06927dccdc7d2681acb98f257d56
    source_path: cli/agent.md
    workflow: 15
---

# `openclaw agent`

Ejecuta un turno de agente mediante el Gateway (usa `--local` para modo integrado).
Usa `--agent <id>` para dirigirte directamente a un agente configurado.

Pasa al menos un selector de sesión:

- `--to <dest>`
- `--session-id <id>`
- `--agent <id>`

Relacionado:

- Herramienta de envío del agente: [Agent send](/tools/agent-send)

## Opciones

- `-m, --message <text>`: cuerpo del mensaje obligatorio
- `-t, --to <dest>`: destinatario usado para derivar la clave de sesión
- `--session-id <id>`: id de sesión explícito
- `--agent <id>`: id del agente; reemplaza los bindings de enrutamiento
- `--thinking <off|minimal|low|medium|high|xhigh>`: nivel de razonamiento del agente
- `--verbose <on|off>`: conserva el nivel detallado para la sesión
- `--channel <channel>`: canal de entrega; omítelo para usar el canal de la sesión principal
- `--reply-to <target>`: reemplazo del destino de entrega
- `--reply-channel <channel>`: reemplazo del canal de entrega
- `--reply-account <id>`: reemplazo de la cuenta de entrega
- `--local`: ejecuta directamente el agente integrado (después de la precarga del registro de plugins)
- `--deliver`: envía la respuesta de vuelta al canal/destino seleccionado
- `--timeout <seconds>`: reemplaza el tiempo de espera del agente (predeterminado 600 o el valor de configuración)
- `--json`: salida JSON

## Ejemplos

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Run locally" --local
```

## Notas

- El modo Gateway recurre al agente integrado cuando falla la solicitud al Gateway. Usa `--local` para forzar la ejecución integrada desde el principio.
- `--local` sigue precargando primero el registro de plugins, por lo que los proveedores, herramientas y canales proporcionados por plugins siguen estando disponibles durante las ejecuciones integradas.
- `--channel`, `--reply-channel` y `--reply-account` afectan a la entrega de la respuesta, no al enrutamiento de la sesión.
- Cuando este comando activa la regeneración de `models.json`, las credenciales de proveedores gestionadas por SecretRef se conservan como marcadores no secretos (por ejemplo nombres de variables de entorno, `secretref-env:ENV_VAR_NAME` o `secretref-managed`), no como texto sin formato secreto resuelto.
- Las escrituras de marcadores son autoritativas según el origen: OpenClaw conserva los marcadores desde la instantánea de configuración de origen activa, no desde los valores secretos resueltos en tiempo de ejecución.
