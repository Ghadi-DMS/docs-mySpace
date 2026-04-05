---
read_when:
    - Quieres encolar un evento del sistema sin crear un trabajo cron
    - Necesitas habilitar o deshabilitar heartbeats
    - Quieres inspeccionar entradas de presencia del sistema
summary: Referencia de la CLI para `openclaw system` (eventos del sistema, heartbeat, presencia)
title: system
x-i18n:
    generated_at: "2026-04-05T12:39:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: a7d19afde9d9cde8a79b0bb8cec6e5673466f4cb9b575fb40111fc32f4eee5d7
    source_path: cli/system.md
    workflow: 15
---

# `openclaw system`

Utilidades de nivel de sistema para el Gateway: encolar eventos del sistema, controlar heartbeats
y ver la presencia.

Todos los subcomandos de `system` usan RPC del Gateway y aceptan los indicadores compartidos del cliente:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--expect-final`

## Comandos comunes

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
openclaw system event --text "Check for urgent follow-ups" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Encola un evento del sistema en la sesión **main**. El siguiente heartbeat lo insertará
como una línea `System:` en el prompt. Usa `--mode now` para activar el heartbeat
inmediatamente; `next-heartbeat` espera al siguiente ciclo programado.

Indicadores:

- `--text <text>`: texto obligatorio del evento del sistema.
- `--mode <mode>`: `now` o `next-heartbeat` (predeterminado).
- `--json`: salida legible por máquina.
- `--url`, `--token`, `--timeout`, `--expect-final`: indicadores compartidos de RPC del Gateway.

## `system heartbeat last|enable|disable`

Controles de heartbeat:

- `last`: muestra el último evento de heartbeat.
- `enable`: vuelve a activar los heartbeats (úsalo si estaban deshabilitados).
- `disable`: pausa los heartbeats.

Indicadores:

- `--json`: salida legible por máquina.
- `--url`, `--token`, `--timeout`, `--expect-final`: indicadores compartidos de RPC del Gateway.

## `system presence`

Enumera las entradas actuales de presencia del sistema que conoce el Gateway (nodos,
instancias y líneas de estado similares).

Indicadores:

- `--json`: salida legible por máquina.
- `--url`, `--token`, `--timeout`, `--expect-final`: indicadores compartidos de RPC del Gateway.

## Notas

- Requiere un Gateway en ejecución accesible mediante tu configuración actual (local o remota).
- Los eventos del sistema son efímeros y no se conservan entre reinicios.
