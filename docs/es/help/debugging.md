---
read_when:
    - Necesitas inspeccionar la salida sin procesar del modelo para detectar filtración de razonamiento
    - Quieres ejecutar el Gateway en modo watch mientras iteras
    - Necesitas un flujo de trabajo de depuración repetible
summary: 'Herramientas de depuración: modo watch, flujos sin procesar del modelo y rastreo de filtración de razonamiento'
title: Depuración
x-i18n:
    generated_at: "2026-04-05T12:43:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: f90d944ecc2e846ca0b26a162126ceefb3a3c6cf065c99b731359ec79d4289e3
    source_path: help/debugging.md
    workflow: 15
---

# Depuración

Esta página cubre ayudantes de depuración para salida en streaming, especialmente cuando un
proveedor mezcla razonamiento en texto normal.

## Sobrescrituras de depuración en runtime

Usa `/debug` en el chat para establecer sobrescrituras de configuración **solo en runtime** (memoria, no disco).
`/debug` está desactivado de forma predeterminada; habilítalo con `commands.debug: true`.
Esto resulta útil cuando necesitas alternar configuraciones poco habituales sin editar `openclaw.json`.

Ejemplos:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug unset messages.responsePrefix
/debug reset
```

`/debug reset` borra todas las sobrescrituras y vuelve a la configuración en disco.

## Modo watch del Gateway

Para iterar rápidamente, ejecuta el gateway bajo el observador de archivos:

```bash
pnpm gateway:watch
```

Esto se asigna a:

```bash
node scripts/watch-node.mjs gateway --force
```

El observador reinicia ante archivos relevantes para la compilación en `src/`, archivos fuente de extensiones,
`package.json` de extensiones y metadatos `openclaw.plugin.json`, `tsconfig.json`,
`package.json` y `tsdown.config.ts`. Los cambios en metadatos de extensiones reinician el
gateway sin forzar una recompilación de `tsdown`; los cambios en código fuente y configuración
siguen recompilando `dist` primero.

Añade cualquier flag de CLI del gateway después de `gateway:watch` y se pasarán en cada
reinicio.

## Perfil de desarrollo + gateway de desarrollo (`--dev`)

Usa el perfil de desarrollo para aislar el estado y levantar una configuración
segura y desechable para depuración. Hay **dos** flags `--dev`:

- **`--dev` global (perfil):** aísla el estado en `~/.openclaw-dev` y
  usa por defecto el puerto `19001` para el gateway (los puertos derivados se desplazan con él).
- **`gateway --dev`:** indica al Gateway que cree automáticamente una configuración +
  espacio de trabajo predeterminados cuando falten (y omita `BOOTSTRAP.md`).

Flujo recomendado (perfil de desarrollo + arranque de desarrollo):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

Si todavía no tienes una instalación global, ejecuta la CLI mediante `pnpm openclaw ...`.

Qué hace esto:

1. **Aislamiento del perfil** (`--dev` global)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (browser/canvas se desplazan en consecuencia)

2. **Arranque de desarrollo** (`gateway --dev`)
   - Escribe una configuración mínima si falta (`gateway.mode=local`, bind loopback).
   - Establece `agent.workspace` al espacio de trabajo de desarrollo.
   - Establece `agent.skipBootstrap=true` (sin `BOOTSTRAP.md`).
   - Inicializa los archivos del espacio de trabajo si faltan:
     `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`.
   - Identidad predeterminada: **C3‑PO** (droide de protocolo).
   - Omite proveedores de canal en modo de desarrollo (`OPENCLAW_SKIP_CHANNELS=1`).

Flujo de restablecimiento (inicio limpio):

```bash
pnpm gateway:dev:reset
```

Nota: `--dev` es un flag **global** de perfil y algunos ejecutores lo consumen.
Si necesitas escribirlo explícitamente, usa la forma con variable de entorno:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

`--reset` borra configuración, credenciales, sesiones y el espacio de trabajo de desarrollo (usando
`trash`, no `rm`), y luego recrea la configuración de desarrollo predeterminada.

Consejo: si ya se está ejecutando un gateway que no es de desarrollo (launchd/systemd), detenlo primero:

```bash
openclaw gateway stop
```

## Registro de flujo sin procesar (OpenClaw)

OpenClaw puede registrar el **flujo sin procesar del asistente** antes de cualquier filtrado/formateo.
Esta es la mejor forma de ver si el razonamiento llega como deltas de texto plano
(o como bloques de pensamiento separados).

Habilítalo mediante la CLI:

```bash
pnpm gateway:watch --raw-stream
```

Sobrescritura opcional de ruta:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

Variables de entorno equivalentes:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

Archivo predeterminado:

`~/.openclaw/logs/raw-stream.jsonl`

## Registro de fragmentos sin procesar (pi-mono)

Para capturar **fragmentos sin procesar compatibles con OpenAI** antes de que se analicen en bloques,
pi-mono expone un registrador separado:

```bash
PI_RAW_STREAM=1
```

Ruta opcional:

```bash
PI_RAW_STREAM_PATH=~/.pi-mono/logs/raw-openai-completions.jsonl
```

Archivo predeterminado:

`~/.pi-mono/logs/raw-openai-completions.jsonl`

> Nota: esto solo lo emiten los procesos que usan el proveedor
> `openai-completions` de pi-mono.

## Notas de seguridad

- Los registros de flujo sin procesar pueden incluir prompts completos, salida de herramientas y datos del usuario.
- Mantén los registros en local y elimínalos después de depurar.
- Si compartes registros, elimina primero secretos y PII.
