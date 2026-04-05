---
read_when:
    - Quieres cambiar entre stable/beta/dev
    - Quieres fijar una versión, etiqueta o SHA específica
    - Estás etiquetando o publicando versiones preliminares
sidebarTitle: Release Channels
summary: 'Canales stable, beta y dev: semántica, cambio, fijación y etiquetado'
title: Canales de lanzamiento
x-i18n:
    generated_at: "2026-04-05T12:44:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3f33a77bf356f989cd4de5f8bb57f330c276e7571b955bea6994a4527e40258d
    source_path: install/development-channels.md
    workflow: 15
---

# Canales de desarrollo

OpenClaw se distribuye en tres canales de actualización:

- **stable**: dist-tag de npm `latest`. Recomendado para la mayoría de los usuarios.
- **beta**: dist-tag de npm `beta` cuando está vigente; si falta beta o es más antigua que
  la versión stable más reciente, el flujo de actualización vuelve a `latest`.
- **dev**: cabeza móvil de `main` (git). dist-tag de npm: `dev` (cuando se publica).
  La rama `main` es para experimentación y desarrollo activo. Puede contener
  funciones incompletas o cambios incompatibles. No la uses en gateways de producción.

Normalmente distribuimos primero las compilaciones stable a **beta**, las probamos ahí y luego ejecutamos un
paso explícito de promoción que mueve la compilación validada a `latest` sin
cambiar el número de versión. Los mantenedores también pueden publicar una versión stable
directamente en `latest` cuando sea necesario. Los dist-tags son la fuente de verdad para instalaciones npm.

## Cambiar de canal

```bash
openclaw update --channel stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` conserva tu elección en la configuración (`update.channel`) y alinea el
método de instalación:

- **`stable`** (instalaciones de paquete): actualiza mediante dist-tag `latest` de npm.
- **`beta`** (instalaciones de paquete): prefiere el dist-tag `beta` de npm, pero vuelve a
  `latest` cuando falta `beta` o es más antiguo que la etiqueta stable actual.
- **`stable`** (instalaciones git): hace checkout de la etiqueta git stable más reciente.
- **`beta`** (instalaciones git): prefiere la etiqueta git beta más reciente, pero vuelve a
  la etiqueta git stable más reciente cuando falta beta o es más antigua.
- **`dev`**: garantiza un checkout git (predeterminado `~/openclaw`, sobrescribible con
  `OPENCLAW_GIT_DIR`), cambia a `main`, hace rebase sobre upstream, compila e
  instala la CLI global desde ese checkout.

Consejo: si quieres stable + dev en paralelo, mantén dos clones y apunta tu
gateway a la versión stable.

## Direccionamiento puntual por versión o etiqueta

Usa `--tag` para apuntar a un dist-tag, versión o especificación de paquete concretos para una sola
actualización **sin** cambiar tu canal persistido:

```bash
# Instalar una versión específica
openclaw update --tag 2026.4.1-beta.1

# Instalar desde el dist-tag beta (puntual, no se conserva)
openclaw update --tag beta

# Instalar desde la rama main de GitHub (tarball npm)
openclaw update --tag main

# Instalar desde una especificación de paquete npm concreta
openclaw update --tag openclaw@2026.4.1-beta.1
```

Notas:

- `--tag` se aplica **solo a instalaciones de paquete (npm)**. Las instalaciones git lo ignoran.
- La etiqueta no se conserva. Tu siguiente `openclaw update` usará tu canal configurado
  como siempre.
- Protección contra downgrade: si la versión objetivo es más antigua que tu versión actual,
  OpenClaw solicita confirmación (omite con `--yes`).
- `--channel beta` es distinto de `--tag beta`: el flujo de canal puede volver
  a stable/latest cuando falta beta o es más antigua, mientras que `--tag beta` apunta al
  dist-tag `beta` sin procesar en esa única ejecución.

## Simulación

Previsualiza lo que haría `openclaw update` sin realizar cambios:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

La simulación muestra el canal efectivo, la versión objetivo, las acciones previstas y
si se requeriría confirmación por downgrade.

## Plugins y canales

Cuando cambias de canal con `openclaw update`, OpenClaw también sincroniza las
fuentes de los plugins:

- `dev` prefiere plugins integrados desde el checkout git.
- `stable` y `beta` restauran paquetes de plugins instalados desde npm.
- Los plugins instalados desde npm se actualizan después de que se completa la actualización del núcleo.

## Comprobar el estado actual

```bash
openclaw update status
```

Muestra el canal activo, el tipo de instalación (git o paquete), la versión actual y
la fuente (configuración, etiqueta git, rama git o valor predeterminado).

## Buenas prácticas de etiquetado

- Etiqueta las versiones a las que quieres que lleguen los checkouts git (`vYYYY.M.D` para stable,
  `vYYYY.M.D-beta.N` para beta).
- `vYYYY.M.D.beta.N` también se reconoce por compatibilidad, pero prefiere `-beta.N`.
- Las etiquetas heredadas `vYYYY.M.D-<patch>` siguen reconociéndose como stable (no beta).
- Mantén las etiquetas inmutables: nunca muevas ni reutilices una etiqueta.
- Los dist-tags de npm siguen siendo la fuente de verdad para instalaciones npm:
  - `latest` -> stable
  - `beta` -> compilación candidata o compilación stable primero en beta
  - `dev` -> instantánea de main (opcional)

## Disponibilidad de la app de macOS

Las compilaciones beta y dev pueden **no** incluir una versión de la app de macOS. Está bien:

- La etiqueta git y el dist-tag de npm aún pueden publicarse.
- Indica “sin compilación de macOS para esta beta” en las notas de la versión o en el changelog.
