---
read_when:
    - Ejecutar o corregir pruebas
summary: Cómo ejecutar pruebas localmente (vitest) y cuándo usar los modos force/cobertura
title: Pruebas
x-i18n:
    generated_at: "2026-04-08T02:18:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: f7c19390f7577b3a29796c67514c96fe4c86c9fa0c7686cd4e377c6e31dcd085
    source_path: reference/test.md
    workflow: 15
---

# Pruebas

- Kit completo de pruebas (suites, live, Docker): [Testing](/es/help/testing)

- `pnpm test:force`: Mata cualquier proceso de gateway persistente que esté ocupando el puerto de control predeterminado y luego ejecuta la suite completa de Vitest con un puerto de gateway aislado para que las pruebas del servidor no colisionen con una instancia en ejecución. Úsalo cuando una ejecución previa del gateway haya dejado ocupado el puerto 18789.
- `pnpm test:coverage`: Ejecuta la suite unitaria con cobertura V8 (mediante `vitest.unit.config.ts`). Los umbrales globales son 70% de líneas/ramas/funciones/sentencias. La cobertura excluye puntos de entrada con mucha integración (cableado de CLI, puentes gateway/telegram, servidor estático de webchat) para mantener el objetivo centrado en lógica comprobable con pruebas unitarias.
- `pnpm test:coverage:changed`: Ejecuta cobertura unitaria solo para archivos modificados desde `origin/main`.
- `pnpm test:changed`: expande las rutas git modificadas en carriles de Vitest acotados cuando el diff solo toca archivos fuente/prueba enroutables. Los cambios de configuración/setup siguen recurriendo a la ejecución nativa de proyectos raíz, para que las ediciones del cableado vuelvan a ejecutarse ampliamente cuando sea necesario.
- `pnpm test`: enruta objetivos explícitos de archivo/directorio mediante carriles de Vitest acotados. Las ejecuciones sin objetivo ahora ejecutan once configuraciones secuenciales de shards (`vitest.full-core-unit-src.config.ts`, `vitest.full-core-unit-security.config.ts`, `vitest.full-core-unit-ui.config.ts`, `vitest.full-core-unit-support.config.ts`, `vitest.full-core-support-boundary.config.ts`, `vitest.full-core-contracts.config.ts`, `vitest.full-core-bundled.config.ts`, `vitest.full-core-runtime.config.ts`, `vitest.full-agentic.config.ts`, `vitest.full-auto-reply.config.ts`, `vitest.full-extensions.config.ts`) en lugar de un único proceso gigantesco del proyecto raíz.
- Los archivos de prueba seleccionados de `plugin-sdk` y `commands` ahora se enrutan mediante carriles ligeros dedicados que conservan solo `test/setup.ts`, dejando los casos pesados en runtime en sus carriles existentes.
- Los archivos fuente auxiliares seleccionados de `plugin-sdk` y `commands` también asignan `pnpm test:changed` a pruebas hermanas explícitas en esos carriles ligeros, de modo que pequeñas ediciones de helpers evitan volver a ejecutar las suites pesadas respaldadas por runtime.
- `auto-reply` ahora también se divide en tres configuraciones dedicadas (`core`, `top-level`, `reply`) para que el arnés de respuestas no domine las pruebas más ligeras de estado/token/helper de nivel superior.
- La configuración base de Vitest ahora usa por defecto `pool: "threads"` e `isolate: false`, con el ejecutor compartido no aislado habilitado en las configuraciones del repositorio.
- `pnpm test:channels` ejecuta `vitest.channels.config.ts`.
- `pnpm test:extensions` ejecuta `vitest.extensions.config.ts`.
- `pnpm test:extensions`: ejecuta suites de extensiones/plugins.
- `pnpm test:perf:imports`: habilita los informes de duración de importación + desglose de importaciones de Vitest, y sigue usando el routing por carriles acotados para objetivos explícitos de archivo/directorio.
- `pnpm test:perf:imports:changed`: el mismo perfilado de importaciones, pero solo para archivos modificados desde `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` toma métricas de la ruta enroutada en modo changed frente a la ejecución nativa del proyecto raíz para el mismo diff git ya confirmado.
- `pnpm test:perf:changed:bench -- --worktree` toma métricas del conjunto actual de cambios del worktree sin confirmar primero.
- `pnpm test:perf:profile:main`: escribe un perfil de CPU para el hilo principal de Vitest (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: escribe perfiles de CPU + heap para el ejecutor unitario (`.artifacts/vitest-runner-profile`).
- Integración de gateway: opt-in mediante `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` o `pnpm test:gateway`.
- `pnpm test:e2e`: Ejecuta pruebas smoke end-to-end de gateway (WS/HTTP de múltiples instancias/emparejamiento de nodos). Usa por defecto `threads` + `isolate: false` con workers adaptativos en `vitest.e2e.config.ts`; ajústalo con `OPENCLAW_E2E_WORKERS=<n>` y establece `OPENCLAW_E2E_VERBOSE=1` para logs detallados.
- `pnpm test:live`: Ejecuta pruebas live de proveedores (minimax/zai). Requiere claves API y `LIVE=1` (o `*_LIVE_TEST=1` específico del proveedor) para dejar de omitirlas.
- `pnpm test:docker:openwebui`: Inicia OpenClaw + Open WebUI en Docker, inicia sesión a través de Open WebUI, comprueba `/api/models` y luego ejecuta un chat real proxificado mediante `/api/chat/completions`. Requiere una clave de modelo live utilizable (por ejemplo OpenAI en `~/.profile`), descarga una imagen externa de Open WebUI y no se espera que sea estable en CI como las suites normales unitarias/e2e.
- `pnpm test:docker:mcp-channels`: Inicia un contenedor Gateway precargado y un segundo contenedor cliente que ejecuta `openclaw mcp serve`, y luego verifica descubrimiento de conversaciones enroutadas, lecturas de transcripción, metadatos de adjuntos, comportamiento de cola de eventos live, routing de envío saliente y notificaciones estilo Claude de canal + permisos a través del puente stdio real. La afirmación de notificaciones de Claude lee directamente los marcos MCP stdio sin procesar para que el smoke refleje lo que el puente realmente emite.

## Puerta local para PR

Para comprobaciones locales de land/gate de PR, ejecuta:

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Si `pnpm test` falla de forma intermitente en un host cargado, vuelve a ejecutarlo una vez antes de tratarlo como una regresión; luego aísla con `pnpm test <path/to/test>`. Para hosts con memoria limitada, usa:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Bench de latencia de modelo (claves locales)

Script: [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

Uso:

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- Variables de entorno opcionales: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`
- Prompt predeterminado: “Reply with a single word: ok. No punctuation or extra text.”

Última ejecución (2025-12-31, 20 ejecuciones):

- mediana de minimax 1279ms (mín 1114, máx 2431)
- mediana de opus 2454ms (mín 1224, máx 3170)

## Bench de arranque de CLI

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

Presets:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: ambos presets

La salida incluye `sampleCount`, avg, p50, p95, min/max, distribución de código de salida/señal y resúmenes de RSS máximo para cada comando. `--cpu-prof-dir` / `--heap-prof-dir` opcionales escriben perfiles V8 por ejecución para que la medición de tiempos y la captura de perfiles usen el mismo arnés.

Convenciones de salida guardada:

- `pnpm test:startup:bench:smoke` escribe el artefacto smoke objetivo en `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` escribe el artefacto de la suite completa en `.artifacts/cli-startup-bench-all.json` usando `runs=5` y `warmup=1`
- `pnpm test:startup:bench:update` actualiza el fixture base versionado en `test/fixtures/cli-startup-bench.json` usando `runs=5` y `warmup=1`

Fixture versionado:

- `test/fixtures/cli-startup-bench.json`
- Actualízalo con `pnpm test:startup:bench:update`
- Compara los resultados actuales con el fixture mediante `pnpm test:startup:bench:check`

## Onboarding E2E (Docker)

Docker es opcional; esto solo hace falta para pruebas smoke de onboarding en contenedor.

Flujo completo de arranque en frío en un contenedor Linux limpio:

```bash
scripts/e2e/onboard-docker.sh
```

Este script controla el asistente interactivo mediante una pseudo-TTY, verifica archivos de config/workspace/sesión, luego inicia el gateway y ejecuta `openclaw health`.

## Smoke de importación de QR (Docker)

Garantiza que `qrcode-terminal` se cargue bajo los runtimes Node de Docker compatibles (Node 24 predeterminado, Node 22 compatible):

```bash
pnpm test:docker:qr
```
