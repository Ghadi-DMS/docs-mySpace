---
read_when:
    - Quieres comprobar rápidamente el estado del Gateway en ejecución
summary: Referencia de la CLI para `openclaw health` (instantánea de salud del gateway mediante RPC)
title: health
x-i18n:
    generated_at: "2026-04-05T12:37:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4ed2b9ceefee6159cabaae9172d2d88174626456e7503d5d2bcd142634188ff0
    source_path: cli/health.md
    workflow: 15
---

# `openclaw health`

Obtiene el estado del Gateway en ejecución.

Opciones:

- `--json`: salida legible por máquina
- `--timeout <ms>`: tiempo de espera de conexión en milisegundos (predeterminado `10000`)
- `--verbose`: registro detallado
- `--debug`: alias de `--verbose`

Ejemplos:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

Notas:

- `openclaw health` predeterminado solicita al gateway en ejecución su instantánea de estado. Cuando el
  gateway ya tiene una instantánea en caché reciente, puede devolver esa carga útil en caché y
  actualizarse en segundo plano.
- `--verbose` fuerza una comprobación en vivo, imprime detalles de conexión del gateway y amplía la
  salida legible por humanos en todas las cuentas y agentes configurados.
- La salida incluye almacenes de sesiones por agente cuando hay varios agentes configurados.
