---
read_when:
    - Necesitas seguir registros del Gateway de forma remota (sin SSH)
    - Quieres líneas de registro JSON para herramientas
summary: Referencia de la CLI para `openclaw logs` (seguir registros del gateway mediante RPC)
title: logs
x-i18n:
    generated_at: "2026-04-05T12:38:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 238a52e31a9a332cab513ced049e92d032b03c50376895ce57dffa2ee7d1e4b4
    source_path: cli/logs.md
    workflow: 15
---

# `openclaw logs`

Sigue los registros de archivos del Gateway mediante RPC (funciona en modo remoto).

Relacionado:

- Resumen del registro: [Logging](/logging)
- CLI del gateway: [gateway](/cli/gateway)

## Opciones

- `--limit <n>`: número máximo de líneas de registro que se devolverán (predeterminado `200`)
- `--max-bytes <n>`: bytes máximos que se leerán del archivo de registro (predeterminado `250000`)
- `--follow`: seguir el flujo de registros
- `--interval <ms>`: intervalo de sondeo durante el seguimiento (predeterminado `1000`)
- `--json`: emitir eventos JSON delimitados por líneas
- `--plain`: salida de texto sin formato sin formato visual
- `--no-color`: deshabilitar colores ANSI
- `--local-time`: mostrar las marcas de tiempo en tu zona horaria local

## Opciones compartidas de RPC del Gateway

`openclaw logs` también acepta los indicadores estándar del cliente Gateway:

- `--url <url>`: URL WebSocket del Gateway
- `--token <token>`: token del Gateway
- `--timeout <ms>`: tiempo de espera en ms (predeterminado `30000`)
- `--expect-final`: esperar una respuesta final cuando la llamada al Gateway esté respaldada por un agente

Cuando pasas `--url`, la CLI no aplica automáticamente credenciales de configuración o del entorno. Incluye `--token` explícitamente si el Gateway de destino requiere autenticación.

## Ejemplos

```bash
openclaw logs
openclaw logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --limit 500
openclaw logs --local-time
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

## Notas

- Usa `--local-time` para mostrar las marcas de tiempo en tu zona horaria local.
- Si el Gateway de local loopback solicita emparejamiento, `openclaw logs` recurre automáticamente al archivo de registro local configurado. Los destinos `--url` explícitos no usan este respaldo.
