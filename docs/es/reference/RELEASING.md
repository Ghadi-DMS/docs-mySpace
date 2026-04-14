---
read_when:
    - Buscando definiciones de canales de lanzamiento públicos
    - Buscando nomenclatura de versiones y cadencia
summary: Canales de lanzamiento públicos, nomenclatura de versiones y cadencia
title: Política de lanzamiento
x-i18n:
    generated_at: "2026-04-14T05:10:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3eaf9f1786b8c9fd4f5a9c657b623cb69d1a485958e1a9b8f108511839b63587
    source_path: reference/RELEASING.md
    workflow: 15
---

# Política de lanzamiento

OpenClaw tiene tres canales de lanzamiento públicos:

- stable: lanzamientos etiquetados que publican en npm `beta` de forma predeterminada, o en npm `latest` cuando se solicita explícitamente
- beta: etiquetas de prelanzamiento que publican en npm `beta`
- dev: la cabecera móvil de `main`

## Nomenclatura de versiones

- Versión de lanzamiento estable: `YYYY.M.D`
  - Etiqueta Git: `vYYYY.M.D`
- Versión de lanzamiento estable de corrección: `YYYY.M.D-N`
  - Etiqueta Git: `vYYYY.M.D-N`
- Versión de prelanzamiento beta: `YYYY.M.D-beta.N`
  - Etiqueta Git: `vYYYY.M.D-beta.N`
- No agregues ceros a la izquierda al mes o al día
- `latest` significa la versión estable actual promovida en npm
- `beta` significa el destino de instalación beta actual
- Los lanzamientos estables y los lanzamientos estables de corrección publican en npm `beta` de forma predeterminada; los operadores de lanzamiento pueden apuntar a `latest` explícitamente, o promover más adelante una compilación beta validada
- Cada lanzamiento de OpenClaw distribuye el paquete npm y la app de macOS juntos

## Cadencia de lanzamientos

- Los lanzamientos avanzan primero por beta
- stable solo sigue después de que se valida la beta más reciente
- El procedimiento detallado de lanzamiento, las aprobaciones, las credenciales y las notas de recuperación son solo para maintainers

## Verificaciones previas al lanzamiento

- Ejecuta `pnpm build && pnpm ui:build` antes de `pnpm release:check` para que existan los artefactos de lanzamiento `dist/*` esperados y el paquete de la Control UI para el paso de validación del pack
- Ejecuta `pnpm release:check` antes de cada lanzamiento etiquetado
- Las comprobaciones de lanzamiento ahora se ejecutan en un workflow manual separado:
  `OpenClaw Release Checks`
- Esta separación es intencional: mantiene corta, determinista y centrada en artefactos la ruta real de lanzamiento a npm, mientras que las comprobaciones live más lentas permanecen en su propio canal para que no ralenticen ni bloqueen la publicación
- Las comprobaciones de lanzamiento deben ejecutarse desde la referencia del workflow `main` para que la lógica del workflow y los secretos permanezcan canónicos
- Ese workflow acepta una etiqueta de lanzamiento existente o el SHA actual completo de 40 caracteres del commit de `main`
- En el modo de SHA de commit, solo acepta el HEAD actual de `origin/main`; usa una etiqueta de lanzamiento para commits de lanzamiento anteriores
- La verificación previa solo de validación de `OpenClaw NPM Release` también acepta el SHA actual completo de 40 caracteres del commit de `main` sin requerir una etiqueta enviada
- Esa ruta de SHA es solo de validación y no puede promoverse a una publicación real
- En modo SHA, el workflow sintetiza `v<package.json version>` solo para la comprobación de metadatos del paquete; la publicación real sigue requiriendo una etiqueta de lanzamiento real
- Ambos workflows mantienen la ruta real de publicación y promoción en runners alojados por GitHub, mientras que la ruta de validación no mutante puede usar los runners Linux Blacksmith más grandes
- Ese workflow ejecuta
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  usando los secretos del workflow `OPENAI_API_KEY` y `ANTHROPIC_API_KEY`
- La verificación previa del lanzamiento a npm ya no espera al canal separado de comprobaciones de lanzamiento
- Ejecuta `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (o la etiqueta beta/corrección correspondiente) antes de la aprobación
- Después de publicar en npm, ejecuta
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (o la versión beta/corrección correspondiente) para verificar la ruta de instalación del registro publicado en un prefijo temporal nuevo
- La automatización de lanzamiento de maintainer ahora usa verificación previa y luego promoción:
  - la publicación real en npm debe pasar un `preflight_run_id` de npm exitoso
  - los lanzamientos estables de npm usan `beta` de forma predeterminada
  - la publicación estable en npm puede apuntar a `latest` explícitamente mediante una entrada del workflow
  - la promoción estable de npm de `beta` a `latest` sigue disponible como un modo manual explícito en el workflow confiable `OpenClaw NPM Release`
  - las publicaciones estables directas también pueden ejecutar un modo explícito de sincronización de dist-tag que hace que tanto `latest` como `beta` apunten a la versión estable ya publicada
  - esos modos de dist-tag todavía necesitan un `NPM_TOKEN` válido en el entorno `npm-release` porque la gestión de `dist-tag` de npm es independiente de la publicación confiable
  - `macOS Release` público es solo de validación
  - la publicación real privada de mac debe pasar `preflight_run_id` y `validate_run_id` privados de mac exitosos
  - las rutas reales de publicación promueven artefactos preparados en lugar de reconstruirlos otra vez
- Para lanzamientos estables de corrección como `YYYY.M.D-N`, el verificador posterior a la publicación también comprueba la misma ruta de actualización con prefijo temporal desde `YYYY.M.D` hasta `YYYY.M.D-N`, de modo que las correcciones de lanzamiento no puedan dejar silenciosamente instalaciones globales anteriores en la carga útil estable base
- La verificación previa del lanzamiento a npm falla de forma cerrada a menos que el tarball incluya tanto `dist/control-ui/index.html` como una carga útil no vacía en `dist/control-ui/assets/`, para que no volvamos a distribuir un panel del navegador vacío
- `pnpm test:install:smoke` también aplica el presupuesto de `unpackedSize` del npm pack sobre el tarball candidato de actualización, de modo que el instalador e2e detecte aumentos accidentales del pack antes de la ruta de publicación del lanzamiento
- Si el trabajo de lanzamiento tocó la planificación de CI, los manifiestos de temporización de extensiones o las matrices de pruebas de extensiones, regenera y revisa las salidas de la matriz del workflow `checks-node-extensions`, propiedad del planificador, desde `.github/workflows/ci.yml` antes de la aprobación para que las notas de lanzamiento no describan una disposición de CI obsoleta
- La preparación del lanzamiento estable de macOS también incluye las superficies del actualizador:
  - el lanzamiento de GitHub debe terminar con los archivos empaquetados `.zip`, `.dmg` y `.dSYM.zip`
  - `appcast.xml` en `main` debe apuntar al nuevo zip estable después de la publicación
  - la app empaquetada debe mantener un bundle id no de depuración, una URL de feed de Sparkle no vacía y un `CFBundleVersion` igual o superior al piso canónico de compilación de Sparkle para esa versión de lanzamiento

## Entradas del workflow de NPM

`OpenClaw NPM Release` acepta estas entradas controladas por el operador:

- `tag`: etiqueta de lanzamiento requerida, como `v2026.4.2`, `v2026.4.2-1` o `v2026.4.2-beta.1`; cuando `preflight_only=true`, también puede ser el SHA actual completo de 40 caracteres del commit de `main` para una verificación previa solo de validación
- `preflight_only`: `true` para solo validación/compilación/paquete, `false` para la ruta de publicación real
- `preflight_run_id`: requerido en la ruta de publicación real para que el workflow reutilice el tarball preparado desde la ejecución exitosa de la verificación previa
- `npm_dist_tag`: etiqueta de destino de npm para la ruta de publicación; el valor predeterminado es `beta`
- `promote_beta_to_latest`: `true` para omitir la publicación y mover una compilación estable `beta` ya publicada a `latest`
- `sync_stable_dist_tags`: `true` para omitir la publicación y hacer que tanto `latest` como `beta` apunten a una versión estable ya publicada

`OpenClaw Release Checks` acepta estas entradas controladas por el operador:

- `ref`: etiqueta de lanzamiento existente o el SHA actual completo de 40 caracteres del commit de `main` para validar

Reglas:

- Las etiquetas estables y de corrección pueden publicar en `beta` o en `latest`
- Las etiquetas beta de prelanzamiento solo pueden publicar en `beta`
- La entrada de SHA de commit completo solo se permite cuando `preflight_only=true`
- El modo SHA de commit para comprobaciones de lanzamiento también requiere el HEAD actual de `origin/main`
- La ruta de publicación real debe usar el mismo `npm_dist_tag` utilizado durante la verificación previa; el workflow verifica esos metadatos antes de continuar con la publicación
- El modo de promoción debe usar una etiqueta estable o de corrección, `preflight_only=false`, un `preflight_run_id` vacío y `npm_dist_tag=beta`
- El modo de sincronización de dist-tag debe usar una etiqueta estable o de corrección, `preflight_only=false`, un `preflight_run_id` vacío, `npm_dist_tag=latest` y `promote_beta_to_latest=false`
- Los modos de promoción y sincronización de dist-tag también requieren un `NPM_TOKEN` válido porque `npm dist-tag add` sigue necesitando autenticación normal de npm; la publicación confiable solo cubre la ruta de publicación del paquete

## Secuencia de lanzamiento estable de npm

Al crear un lanzamiento estable de npm:

1. Ejecuta `OpenClaw NPM Release` con `preflight_only=true`
   - Antes de que exista una etiqueta, puedes usar el SHA actual completo de `main` para una ejecución en seco solo de validación del workflow de verificación previa
2. Elige `npm_dist_tag=beta` para el flujo normal primero-beta, o `latest` solo cuando intencionalmente quieras una publicación estable directa
3. Ejecuta `OpenClaw Release Checks` por separado con la misma etiqueta o el SHA actual completo de `main` cuando quieras cobertura live de prompt cache
   - Esto es intencionalmente separado para que la cobertura live siga disponible sin volver a acoplar comprobaciones largas o inestables al workflow de publicación
4. Guarda el `preflight_run_id` exitoso
5. Ejecuta `OpenClaw NPM Release` otra vez con `preflight_only=false`, la misma `tag`, el mismo `npm_dist_tag` y el `preflight_run_id` guardado
6. Si el lanzamiento llegó a `beta`, ejecuta más tarde `OpenClaw NPM Release` con la misma `tag` estable, `promote_beta_to_latest=true`, `preflight_only=false`, `preflight_run_id` vacío y `npm_dist_tag=beta` cuando quieras mover esa compilación publicada a `latest`
7. Si el lanzamiento se publicó intencionalmente de forma directa en `latest` y `beta` debe seguir la misma compilación estable, ejecuta `OpenClaw NPM Release` con la misma `tag` estable, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`, `preflight_only=false`, `preflight_run_id` vacío y `npm_dist_tag=latest`

Los modos de promoción y sincronización de dist-tag aún requieren la aprobación del entorno `npm-release` y un `NPM_TOKEN` válido accesible para esa ejecución del workflow.

Eso mantiene documentadas y visibles para el operador tanto la ruta de publicación directa como la ruta de promoción primero-beta.

## Referencias públicas

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Los maintainers usan la documentación privada de lanzamiento en
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
para el runbook real.
