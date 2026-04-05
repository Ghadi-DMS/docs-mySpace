---
read_when:
    - Quieres el bucle de desarrollo local más rápido (bun + watch)
    - Te encuentras con problemas de instalación/parches/scripts de ciclo de vida de Bun
summary: 'Flujo de trabajo con Bun (experimental): instalaciones y particularidades frente a pnpm'
title: Bun (Experimental)
x-i18n:
    generated_at: "2026-04-05T12:44:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: b0845567834124bb9206db64df013dc29f3b61a04da4f7e7f0c2823a9ecd67a6
    source_path: install/bun.md
    workflow: 15
---

# Bun (Experimental)

<Warning>
Bun **no se recomienda para el runtime del gateway** (problemas conocidos con WhatsApp y Telegram). Usa Node para producción.
</Warning>

Bun es un runtime local opcional para ejecutar TypeScript directamente (`bun run ...`, `bun --watch ...`). El gestor de paquetes predeterminado sigue siendo `pnpm`, que es totalmente compatible y lo usa la herramienta de documentación. Bun no puede usar `pnpm-lock.yaml` y lo ignorará.

## Instalación

<Steps>
  <Step title="Instalar dependencias">
    ```sh
    bun install
    ```

    `bun.lock` / `bun.lockb` están ignorados por git, así que no hay cambios innecesarios en el repositorio. Para omitir por completo la escritura del lockfile:

    ```sh
    bun install --no-save
    ```

  </Step>
  <Step title="Compilar y probar">
    ```sh
    bun run build
    bun run vitest run
    ```
  </Step>
</Steps>

## Scripts de ciclo de vida

Bun bloquea los scripts de ciclo de vida de dependencias a menos que se confíe explícitamente en ellos. Para este repositorio, los scripts que suelen bloquearse no son necesarios:

- `@whiskeysockets/baileys` `preinstall` -- comprueba que la versión principal de Node sea >= 20 (OpenClaw usa Node 24 de forma predeterminada y sigue siendo compatible con Node 22 LTS, actualmente `22.14+`)
- `protobufjs` `postinstall` -- emite advertencias sobre esquemas de versión incompatibles (sin artefactos de compilación)

Si te encuentras con un problema de runtime que requiera estos scripts, confía en ellos explícitamente:

```sh
bun pm trust @whiskeysockets/baileys protobufjs
```

## Advertencias

Algunos scripts siguen codificando pnpm de forma fija (por ejemplo `docs:build`, `ui:*`, `protocol:check`). Por ahora, ejecútalos con pnpm.
