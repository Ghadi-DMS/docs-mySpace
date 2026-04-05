---
read_when:
    - Ejecutar o corregir pruebas
summary: Cómo ejecutar pruebas localmente (vitest) y cuándo usar los modos force/coverage
title: Pruebas
x-i18n:
    generated_at: "2026-04-05T12:53:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 78390107a9ac2bdc4294d4d0204467c5efdd98faebaf308f3a4597ab966a6d26
    source_path: reference/test.md
    workflow: 15
---

# Pruebas

- Kit completo de pruebas (suites, en vivo, Docker): [Testing](/es/help/testing)

- `pnpm test:force`: Mata cualquier proceso persistente del gateway que esté reteniendo el puerto de control predeterminado y luego ejecuta la suite completa de Vitest con un puerto de gateway aislado para que las pruebas del servidor no colisionen con una instancia en ejecución. Úsalo cuando una ejecución previa del gateway haya dejado ocupado el puerto 18789.
- `pnpm test:coverage`: Ejecuta la suite unitaria con cobertura V8 (mediante `vitest.unit.config.ts`). Los umbrales globales son 70% de líneas/ramas/funciones/instrucciones. La cobertura excluye puntos de entrada con mucha integración (cableado de CLI, puentes de gateway/telegram y servidor estático de webchat) para mantener el objetivo centrado en la lógica comprobable con pruebas unitarias.
- `pnpm test:coverage:changed`: Ejecuta cobertura unitaria solo para los archivos modificados desde `origin/main`.
- `pnpm test:changed`: ejecuta la configuración nativa de proyectos de Vitest con `--changed origin/main`. La configuración base trata los archivos de proyectos/configuración como `forceRerunTriggers` para que los cambios de cableado sigan relanzando ampliamente cuando sea necesario.
- `pnpm test`: ejecuta directamente la configuración nativa de proyectos raíz de Vitest. Los filtros de archivos funcionan de forma nativa en todos los proyectos configurados.
- La configuración base de Vitest ahora usa por defecto `pool: "threads"` e `isolate: false`, con el runner compartido no aislado habilitado en las configuraciones del repositorio.
- `pnpm test:channels` ejecuta `vitest.channels.config.ts`.
- `pnpm test:extensions` ejecuta `vitest.extensions.config.ts`.
- `pnpm test:extensions`: ejecuta las suites de extensiones/plugins.
- `pnpm test:perf:imports`: habilita los informes de duración de importación + desglose de importaciones de Vitest para la ejecución nativa de proyectos raíz.
- `pnpm test:perf:imports:changed`: el mismo perfilado de importaciones, pero solo para los archivos modificados desde `origin/main`.
- `pnpm test:perf:profile:main`: escribe un perfil de CPU para el hilo principal de Vitest (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: escribe perfiles de CPU + heap para el runner unitario (`.artifacts/vitest-runner-profile`).
- Integración de gateway: activación opcional mediante `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` o `pnpm test:gateway`.
- `pnpm test:e2e`: Ejecuta las pruebas de humo end-to-end del gateway (emparejamiento multiinstancia WS/HTTP/nodo). Usa por defecto `threads` + `isolate: false` con workers adaptativos en `vitest.e2e.config.ts`; ajústalo con `OPENCLAW_E2E_WORKERS=<n>` y establece `OPENCLAW_E2E_VERBOSE=1` para registros detallados.
- `pnpm test:live`: Ejecuta pruebas en vivo de proveedores (minimax/zai). Requiere claves API y `LIVE=1` (o `*_LIVE_TEST=1` específico del proveedor) para dejar de omitirlas.
- `pnpm test:docker:openwebui`: Inicia OpenClaw + Open WebUI en Docker, inicia sesión a través de Open WebUI, comprueba `/api/models` y luego ejecuta un chat real proxyado a través de `/api/chat/completions`. Requiere una clave de modelo en vivo utilizable (por ejemplo, OpenAI en `~/.profile`), descarga una imagen externa de Open WebUI y no se espera que sea estable en CI como las suites normales unitarias/e2e.
- `pnpm test:docker:mcp-channels`: Inicia un contenedor Gateway preconfigurado y un segundo contenedor cliente que lanza `openclaw mcp serve`, luego verifica el descubrimiento enrutado de conversaciones, lecturas de transcripciones, metadatos de adjuntos, comportamiento de la cola de eventos en vivo, enrutamiento de envíos salientes y notificaciones de canal + permisos al estilo Claude sobre el puente stdio real. La aserción de notificación de Claude lee directamente los frames MCP raw de stdio para que la prueba de humo refleje lo que el puente realmente emite.

## Puerta local de PR

Para comprobaciones locales de aterrizaje/puerta de PR, ejecuta:

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Si `pnpm test` falla de forma intermitente en un host cargado, vuelve a ejecutarlo una vez antes de tratarlo como una regresión y luego aíslalo con `pnpm test <path/to/test>`. Para hosts con memoria limitada, usa:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Benchmark de latencia de modelos (claves locales)

Script: [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

Uso:

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- Variables de entorno opcionales: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`
- Prompt predeterminado: “Reply with a single word: ok. No punctuation or extra text.”

Última ejecución (2025-12-31, 20 ejecuciones):

- mediana de minimax: 1279ms (mín. 1114, máx. 2431)
- mediana de opus: 2454ms (mín. 1224, máx. 3170)

## Benchmark de inicio de CLI

Script: [`scripts/bench-cli-startup.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-cli-startup.ts)

Uso:

- `pnpm test:startup:bench`
- `pnpm test:startup:bench:smoke`
- `pnpm test:startup:bench:save`
- `pnpm test:startup:bench:update`
- `pnpm test:startup:bench:check`
- `pnpm tsx scripts/bench-cli-startup.ts`
- `pnpm tsx scripts/bench-cli-startup.ts --runs 12`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3`
- `pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all`
- `pnpm tsx scripts/bench-cli-startup.ts --preset all --output .artifacts/cli-startup-bench-all.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case gatewayStatusJson --output .artifacts/cli-startup-bench-smoke.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu`
- `pnpm tsx scripts/bench-cli-startup.ts --json`

Preajustes:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: ambos preajustes

La salida incluye `sampleCount`, avg, p50, p95, min/max, distribución de códigos de salida/señales y resúmenes de RSS máximo para cada comando. La opción `--cpu-prof-dir` / `--heap-prof-dir` escribe perfiles V8 por ejecución para que la medición de tiempo y la captura de perfiles usen el mismo arnés.

Convenciones de salida guardada:

- `pnpm test:startup:bench:smoke` escribe el artefacto de humo dirigido en `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` escribe el artefacto de la suite completa en `.artifacts/cli-startup-bench-all.json` usando `runs=5` y `warmup=1`
- `pnpm test:startup:bench:update` actualiza el fixture base versionado en `test/fixtures/cli-startup-bench.json` usando `runs=5` y `warmup=1`

Fixture versionado:

- `test/fixtures/cli-startup-bench.json`
- Actualízalo con `pnpm test:startup:bench:update`
- Compara los resultados actuales con el fixture mediante `pnpm test:startup:bench:check`

## E2E de onboarding (Docker)

Docker es opcional; esto solo se necesita para pruebas de humo de onboarding en contenedores.

Flujo completo de inicio en frío en un contenedor Linux limpio:

```bash
scripts/e2e/onboard-docker.sh
```

Este script maneja el asistente interactivo mediante un pseudo-TTY, verifica los archivos de configuración/espacio de trabajo/sesión y luego inicia el gateway y ejecuta `openclaw health`.

## Prueba de humo de importación de QR (Docker)

Asegura que `qrcode-terminal` se cargue bajo los runtimes de Node para Docker compatibles (Node 24 predeterminado, Node 22 compatible):

```bash
pnpm test:docker:qr
```
