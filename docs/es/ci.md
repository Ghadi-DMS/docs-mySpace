---
read_when:
    - Necesitas entender por qué un trabajo de CI se ejecutó o no
    - Estás depurando comprobaciones fallidas de GitHub Actions
summary: Grafo de trabajos de CI, controles de alcance y equivalentes de comandos locales
title: Pipeline de CI
x-i18n:
    generated_at: "2026-04-05T12:37:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5a95b6e584b4309bc249866ea436b4dfe30e0298ab8916eadbc344edae3d1194
    source_path: ci.md
    workflow: 15
---

# Pipeline de CI

La CI se ejecuta en cada push a `main` y en cada pull request. Usa un alcance inteligente para omitir trabajos costosos cuando solo cambiaron áreas no relacionadas.

## Resumen de trabajos

| Job                      | Propósito                                                                                | Cuándo se ejecuta                   |
| ------------------------ | ---------------------------------------------------------------------------------------- | ----------------------------------- |
| `preflight`              | Detectar cambios solo de documentación, alcances modificados, extensiones modificadas y compilar el manifiesto de CI | Siempre en pushes y PR no borrador  |
| `security-fast`          | Detección de claves privadas, auditoría de workflows mediante `zizmor`, auditoría de dependencias de producción | Siempre en pushes y PR no borrador  |
| `build-artifacts`        | Compilar `dist/` y la Control UI una vez, y subir artefactos reutilizables para trabajos posteriores | Cambios relevantes para Node        |
| `checks-fast-core`       | Rutas rápidas de corrección en Linux, como comprobaciones de contratos bundled/plugin/protocol | Cambios relevantes para Node        |
| `checks-fast-extensions` | Agregar las rutas fragmentadas de extensiones después de que se complete `checks-fast-extensions-shard` | Cambios relevantes para Node        |
| `extension-fast`         | Pruebas enfocadas solo para los plugins integrados modificados                           | Cuando se detectan cambios en extensiones |
| `check`                  | Puerta local principal en CI: `pnpm check` más `pnpm build:strict-smoke`                | Cambios relevantes para Node        |
| `check-additional`       | Protecciones de arquitectura y límites, más el arnés de regresión de observación del gateway | Cambios relevantes para Node        |
| `build-smoke`            | Pruebas smoke de la CLI compilada y smoke de memoria al inicio                          | Cambios relevantes para Node        |
| `checks`                 | Rutas Linux Node más pesadas: pruebas completas, pruebas de canales y compatibilidad con Node 22 solo en push | Cambios relevantes para Node        |
| `check-docs`             | Comprobaciones de formato, lint y enlaces rotos de la documentación                     | Cuando cambia la documentación      |
| `skills-python`          | Ruff + pytest para Skills respaldadas por Python                                        | Cambios relevantes para Skills de Python |
| `checks-windows`         | Rutas de prueba específicas de Windows                                                  | Cambios relevantes para Windows     |
| `macos-node`             | Ruta de pruebas TypeScript en macOS usando los artefactos compilados compartidos        | Cambios relevantes para macOS       |
| `macos-swift`            | Lint, compilación y pruebas de Swift para la app de macOS                               | Cambios relevantes para macOS       |
| `android`                | Matriz de compilación y pruebas de Android                                              | Cambios relevantes para Android     |

## Orden de fallo rápido

Los trabajos están ordenados para que las comprobaciones baratas fallen antes de que se ejecuten las costosas:

1. `preflight` decide qué rutas existen realmente. La lógica `docs-scope` y `changed-scope` son pasos dentro de este trabajo, no trabajos independientes.
2. `security-fast`, `check`, `check-additional`, `check-docs` y `skills-python` fallan rápidamente sin esperar a los trabajos más pesados de artefactos y matrices de plataforma.
3. `build-artifacts` se superpone con las rutas rápidas de Linux para que los consumidores posteriores puedan comenzar en cuanto la compilación compartida esté lista.
4. Después se abren en abanico las rutas más pesadas de plataforma y tiempo de ejecución: `checks-fast-core`, `checks-fast-extensions`, `extension-fast`, `checks`, `checks-windows`, `macos-node`, `macos-swift` y `android`.

La lógica de alcance vive en `scripts/ci-changed-scope.mjs` y está cubierta por pruebas unitarias en `src/scripts/ci-changed-scope.test.ts`.
El workflow independiente `install-smoke` reutiliza el mismo script de alcance mediante su propio trabajo `preflight`. Calcula `run_install_smoke` a partir de la señal más limitada changed-smoke, por lo que el smoke de Docker/install solo se ejecuta para cambios relevantes de instalación, empaquetado y contenedores.

En los pushes, la matriz `checks` agrega la ruta `compat-node22`, solo para push. En pull requests, esa ruta se omite y la matriz sigue centrada en las rutas normales de pruebas/canales.

## Runners

| Runner                           | Jobs                                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `blacksmith-16vcpu-ubuntu-2404`  | `preflight`, `security-fast`, `build-artifacts`, comprobaciones de Linux, comprobaciones de documentación, Skills de Python, `android` |
| `blacksmith-32vcpu-windows-2025` | `checks-windows`                                                                                     |
| `macos-latest`                   | `macos-node`, `macos-swift`                                                                          |

## Equivalentes locales

```bash
pnpm check          # tipos + lint + formato
pnpm build:strict-smoke
pnpm test:gateway:watch-regression
pnpm test           # pruebas de vitest
pnpm test:channels
pnpm check:docs     # formato de documentación + lint + enlaces rotos
pnpm build          # compilar dist cuando importan las rutas de artefactos/compilación smoke de CI
```
