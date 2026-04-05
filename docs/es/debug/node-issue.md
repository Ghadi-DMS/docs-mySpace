---
read_when:
    - Depurar fallos de scripts de desarrollo solo con Node o en modo watch
    - Investigar fallos del cargador tsx/esbuild en OpenClaw
summary: Notas y soluciones alternativas para el fallo "__name is not a function" con Node + tsx
title: Fallo de Node + tsx
x-i18n:
    generated_at: "2026-04-05T12:41:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: f5beab7cdfe7679680f65176234a617293ce495886cfffb151518adfa61dc8dc
    source_path: debug/node-issue.md
    workflow: 15
---

# Fallo "__name is not a function" con Node + tsx

## Resumen

Ejecutar OpenClaw con Node usando `tsx` falla al iniciarse con:

```
[openclaw] Failed to start CLI: TypeError: __name is not a function
    at createSubsystemLogger (.../src/logging/subsystem.ts:203:25)
    at .../src/agents/auth-profiles/constants.ts:25:20
```

Esto empezó después de cambiar los scripts de desarrollo de Bun a `tsx` (commit `2871657e`, 2026-01-06). La misma ruta de runtime funcionaba con Bun.

## Entorno

- Node: v25.x (observado en v25.3.0)
- tsx: 4.21.0
- SO: macOS (la reproducción también es probable en otras plataformas que ejecuten Node 25)

## Reproducción (solo Node)

```bash
# in repo root
node --version
pnpm install
node --import tsx src/entry.ts status
```

## Reproducción mínima en el repositorio

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

## Comprobación de la versión de Node

- Node 25.3.0: falla
- Node 22.22.0 (Homebrew `node@22`): falla
- Node 24: aún no está instalado aquí; necesita verificación

## Notas / hipótesis

- `tsx` usa esbuild para transformar TS/ESM. `keepNames` de esbuild emite un helper `__name` y envuelve las definiciones de funciones con `__name(...)`.
- El fallo indica que `__name` existe, pero no es una función en runtime, lo que implica que falta el helper o fue sobrescrito para este módulo en la ruta del cargador de Node 25.
- Se han informado problemas similares con el helper `__name` en otros consumidores de esbuild cuando falta el helper o se reescribe.

## Historial de regresión

- `2871657e` (2026-01-06): los scripts cambiaron de Bun a tsx para hacer que Bun fuera opcional.
- Antes de eso (ruta de Bun), `openclaw status` y `gateway:watch` funcionaban.

## Soluciones alternativas

- Usar Bun para scripts de desarrollo (reversión temporal actual).
- Usar Node + watch de tsc y luego ejecutar la salida compilada:

  ```bash
  pnpm exec tsc --watch --preserveWatchOutput
  node --watch openclaw.mjs status
  ```

- Confirmado localmente: `pnpm exec tsc -p tsconfig.json` + `node openclaw.mjs status` funciona en Node 25.
- Desactivar `keepNames` de esbuild en el cargador TS si es posible (evita la inserción del helper `__name`); tsx actualmente no expone esto.
- Probar Node LTS (22/24) con `tsx` para ver si el problema es específico de Node 25.

## Referencias

- [https://opennext.js.org/cloudflare/howtos/keep_names](https://opennext.js.org/cloudflare/howtos/keep_names)
- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## Próximos pasos

- Reproducir en Node 22/24 para confirmar la regresión en Node 25.
- Probar `tsx` nightly o fijar una versión anterior si existe una regresión conocida.
- Si también se reproduce en Node LTS, abrir un informe mínimo de reproducción upstream con el stack trace de `__name`.
