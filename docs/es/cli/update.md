---
read_when:
    - Quieres actualizar de forma segura un checkout del código fuente
    - Necesitas entender el comportamiento abreviado de `--update`
summary: Referencia de CLI para `openclaw update` (actualización del código fuente relativamente segura + reinicio automático del gateway)
title: update
x-i18n:
    generated_at: "2026-04-05T12:39:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 12c8098654b644c3666981d379f6c018e84fde56a5420f295d78052f9001bdad
    source_path: cli/update.md
    workflow: 15
---

# `openclaw update`

Actualiza OpenClaw de forma segura y cambia entre los canales estable/beta/desarrollo.

Si instalaste mediante **npm/pnpm/bun** (instalación global, sin metadatos de git),
las actualizaciones se realizan mediante el flujo del gestor de paquetes de [Actualización](/install/updating).

## Uso

```bash
openclaw update
openclaw update status
openclaw update wizard
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --json
openclaw --update
```

## Opciones

- `--no-restart`: omite reiniciar el servicio Gateway después de una actualización correcta.
- `--channel <stable|beta|dev>`: establece el canal de actualización (git + npm; se conserva en la configuración).
- `--tag <dist-tag|version|spec>`: sobrescribe el destino del paquete solo para esta actualización. Para instalaciones de paquetes, `main` se asigna a `github:openclaw/openclaw#main`.
- `--dry-run`: muestra una vista previa de las acciones de actualización planificadas (canal/etiqueta/destino/flujo de reinicio) sin escribir configuración, instalar, sincronizar plugins ni reiniciar.
- `--json`: imprime JSON `UpdateRunResult` legible por máquina.
- `--timeout <seconds>`: tiempo de espera por paso (el valor predeterminado es 1200 s).
- `--yes`: omite las solicitudes de confirmación (por ejemplo, confirmación de downgrade)

Nota: los downgrades requieren confirmación porque las versiones anteriores pueden romper la configuración.

## `update status`

Muestra el canal de actualización activo + la etiqueta/rama/SHA de git (para checkouts del código fuente), además de la disponibilidad de actualizaciones.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

Opciones:

- `--json`: imprime JSON de estado legible por máquina.
- `--timeout <seconds>`: tiempo de espera para las comprobaciones (el valor predeterminado es 3 s).

## `update wizard`

Flujo interactivo para elegir un canal de actualización y confirmar si se debe reiniciar el Gateway
después de actualizar (el valor predeterminado es reiniciar). Si seleccionas `dev` sin un checkout de git,
ofrece crear uno.

Opciones:

- `--timeout <seconds>`: tiempo de espera para cada paso de actualización (predeterminado `1200`)

## Qué hace

Cuando cambias de canal explícitamente (`--channel ...`), OpenClaw también mantiene alineado
el método de instalación:

- `dev` → asegura un checkout de git (predeterminado: `~/openclaw`, se puede sobrescribir con `OPENCLAW_GIT_DIR`),
  lo actualiza e instala la CLI global desde ese checkout.
- `stable` → instala desde npm usando `latest`.
- `beta` → prefiere la etiqueta de distribución `beta` de npm, pero recurre a `latest` cuando beta
  falta o es más antigua que la versión estable actual.

El actualizador automático del núcleo del Gateway (cuando está habilitado mediante configuración) reutiliza esta misma ruta de actualización.

## Flujo de checkout de git

Canales:

- `stable`: hace checkout de la etiqueta no beta más reciente, luego compila + ejecuta doctor.
- `beta`: prefiere la etiqueta `-beta` más reciente, pero recurre a la etiqueta estable más reciente
  cuando falta beta o es más antigua.
- `dev`: hace checkout de `main`, luego hace fetch + rebase.

Resumen de alto nivel:

1. Requiere un worktree limpio (sin cambios no confirmados).
2. Cambia al canal seleccionado (etiqueta o rama).
3. Hace fetch del upstream (solo dev).
4. Solo dev: ejecuta un preflight de lint + compilación de TypeScript en un worktree temporal; si el tip falla, retrocede hasta 10 commits para encontrar la compilación limpia más reciente.
5. Hace rebase sobre el commit seleccionado (solo dev).
6. Instala dependencias (se prefiere pnpm; respaldo con npm; bun sigue disponible como respaldo secundario de compatibilidad).
7. Compila + compila la Control UI.
8. Ejecuta `openclaw doctor` como comprobación final de “actualización segura”.
9. Sincroniza los plugins con el canal activo (dev usa extensiones integradas; stable/beta usa npm) y actualiza los plugins instalados con npm.

## Abreviatura `--update`

`openclaw --update` se reescribe como `openclaw update` (útil para shells y scripts de lanzador).

## Consulta también

- `openclaw doctor` (ofrece ejecutar primero la actualización en checkouts de git)
- [Canales de desarrollo](/install/development-channels)
- [Actualización](/install/updating)
- [Referencia de CLI](/cli)
