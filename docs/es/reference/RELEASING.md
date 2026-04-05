---
read_when:
    - Buscas definiciones públicas de los canales de lanzamiento
    - Buscas la nomenclatura de versiones y la cadencia
summary: Canales de lanzamiento públicos, nomenclatura de versiones y cadencia
title: Política de lanzamientos
x-i18n:
    generated_at: "2026-04-05T12:53:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: bb52a13264c802395aa55404c6baeec5c7b2a6820562e7a684057e70cc85668f
    source_path: reference/RELEASING.md
    workflow: 15
---

# Política de lanzamientos

OpenClaw tiene tres canales públicos de lanzamiento:

- stable: lanzamientos etiquetados que se publican en npm `beta` de forma predeterminada, o en npm `latest` cuando se solicita explícitamente
- beta: etiquetas de prelanzamiento que se publican en npm `beta`
- dev: la cabecera móvil de `main`

## Nomenclatura de versiones

- Versión de lanzamiento estable: `YYYY.M.D`
  - Etiqueta de Git: `vYYYY.M.D`
- Versión de lanzamiento de corrección estable: `YYYY.M.D-N`
  - Etiqueta de Git: `vYYYY.M.D-N`
- Versión de prelanzamiento beta: `YYYY.M.D-beta.N`
  - Etiqueta de Git: `vYYYY.M.D-beta.N`
- No añadas ceros a la izquierda al mes ni al día
- `latest` significa la versión estable actual promovida en npm
- `beta` significa el destino de instalación beta actual
- Los lanzamientos estables y las correcciones estables se publican en npm `beta` de forma predeterminada; los operadores de lanzamiento pueden apuntar explícitamente a `latest`, o promover más adelante una compilación beta validada
- Cada lanzamiento de OpenClaw distribuye juntos el paquete npm y la app de macOS

## Cadencia de lanzamientos

- Los lanzamientos pasan primero por beta
- Stable llega solo después de que se valide la beta más reciente
- El procedimiento detallado de lanzamiento, las aprobaciones, las credenciales y las notas de recuperación son
  solo para maintainers

## Verificación previa al lanzamiento

- Ejecuta `pnpm build && pnpm ui:build` antes de `pnpm release:check` para que existan los artefactos de lanzamiento `dist/*`
  esperados y el bundle de la Control UI para el paso de validación del pack
- Ejecuta `pnpm release:check` antes de cada lanzamiento etiquetado
- La verificación previa de npm de la rama main también ejecuta
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  antes de empaquetar el tarball, usando tanto los secrets de workflow `OPENAI_API_KEY` como
  `ANTHROPIC_API_KEY`
- Ejecuta `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (o la etiqueta beta/corrección correspondiente) antes de la aprobación
- Después de publicar en npm, ejecuta
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (o la versión beta/corrección correspondiente) para verificar la ruta de instalación
  publicada del registro en un prefijo temporal nuevo
- La automatización de lanzamiento de maintainers ahora usa verificación previa y luego promoción:
  - la publicación real en npm debe pasar con un `preflight_run_id` de npm exitoso
  - los lanzamientos estables de npm usan `beta` de forma predeterminada
  - la publicación estable en npm puede apuntar explícitamente a `latest` mediante una entrada del workflow
  - la promoción estable de npm de `beta` a `latest` sigue estando disponible como un modo manual explícito en el workflow de confianza `OpenClaw NPM Release`
  - ese modo de promoción sigue necesitando un `NPM_TOKEN` válido en el entorno `npm-release` porque la administración de `dist-tag` de npm es independiente de la publicación de confianza
  - el `macOS Release` público es solo de validación
  - la publicación real privada de mac debe pasar una validación previa privada de mac exitosa
    `preflight_run_id` y `validate_run_id`
  - las rutas de publicación reales promueven artefactos preparados en lugar de
    reconstruirlos otra vez
- Para lanzamientos de corrección estables como `YYYY.M.D-N`, el verificador posterior a la publicación
  también comprueba la misma ruta de actualización con prefijo temporal desde `YYYY.M.D` hasta `YYYY.M.D-N`
  para que las correcciones de lanzamiento no puedan dejar silenciosamente instalaciones globales más antiguas en la
  carga útil estable base
- La verificación previa del lanzamiento de npm falla de forma cerrada a menos que el tarball incluya tanto
  `dist/control-ui/index.html` como una carga útil no vacía en `dist/control-ui/assets/`
  para que no volvamos a distribuir un panel del navegador vacío
- Si el trabajo de lanzamiento tocó la planificación de CI, los manifiestos de tiempos de extensiones o las matrices
  rápidas de pruebas, regenera y revisa las salidas de matriz del workflow `checks-fast-extensions`
  propiedad del planificador desde `.github/workflows/ci.yml`
  antes de la aprobación para que las notas de lanzamiento no describan una disposición de CI obsoleta
- La preparación de lanzamiento estable de macOS también incluye las superficies del actualizador:
  - el lanzamiento de GitHub debe terminar con los archivos empaquetados `.zip`, `.dmg` y `.dSYM.zip`
  - `appcast.xml` en `main` debe apuntar al nuevo zip estable después de la publicación
  - la app empaquetada debe conservar un bundle id no de depuración, una URL de feed de Sparkle no vacía
    y un `CFBundleVersion` igual o superior al umbral canónico de compilación de Sparkle
    para esa versión de lanzamiento

## Entradas del workflow de NPM

`OpenClaw NPM Release` acepta estas entradas controladas por el operador:

- `tag`: etiqueta de lanzamiento obligatoria como `v2026.4.2`, `v2026.4.2-1`, o
  `v2026.4.2-beta.1`
- `preflight_only`: `true` para solo validación/compilación/empaquetado, `false` para la
  ruta de publicación real
- `preflight_run_id`: obligatorio en la ruta de publicación real para que el workflow reutilice
  el tarball preparado de la ejecución de verificación previa exitosa
- `npm_dist_tag`: etiqueta de destino de npm para la ruta de publicación; el valor predeterminado es `beta`
- `promote_beta_to_latest`: `true` para omitir la publicación y mover una compilación
  estable `beta` ya publicada a `latest`

Reglas:

- Las etiquetas estables y de corrección pueden publicarse en `beta` o en `latest`
- Las etiquetas de prelanzamiento beta pueden publicarse solo en `beta`
- La ruta de publicación real debe usar el mismo `npm_dist_tag` usado durante la verificación previa;
  el workflow verifica esos metadatos antes de que continúe la publicación
- El modo de promoción debe usar una etiqueta estable o de corrección, `preflight_only=false`,
  un `preflight_run_id` vacío y `npm_dist_tag=beta`
- El modo de promoción también requiere un `NPM_TOKEN` válido en el entorno `npm-release`
  porque `npm dist-tag add` sigue necesitando autenticación normal de npm

## Secuencia de lanzamiento estable de npm

Al crear un lanzamiento estable de npm:

1. Ejecuta `OpenClaw NPM Release` con `preflight_only=true`
2. Elige `npm_dist_tag=beta` para el flujo normal primero-beta, o `latest` solo
   cuando intencionalmente quieras una publicación estable directa
3. Guarda el `preflight_run_id` exitoso
4. Ejecuta `OpenClaw NPM Release` de nuevo con `preflight_only=false`, la misma
   `tag`, el mismo `npm_dist_tag` y el `preflight_run_id` guardado
5. Si el lanzamiento terminó en `beta`, ejecuta `OpenClaw NPM Release` más adelante con la
   misma `tag` estable, `promote_beta_to_latest=true`, `preflight_only=false`,
   `preflight_run_id` vacío y `npm_dist_tag=beta` cuando quieras mover esa
   compilación publicada a `latest`

El modo de promoción sigue requiriendo la aprobación del entorno `npm-release` y un
`NPM_TOKEN` válido en ese entorno.

Eso mantiene documentadas y visibles para el operador tanto la ruta de publicación directa
como la ruta de promoción primero-beta.

## Referencias públicas

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Los maintainers usan la documentación privada de lanzamientos en
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
para el runbook real.
