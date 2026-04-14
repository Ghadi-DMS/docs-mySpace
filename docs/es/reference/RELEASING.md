---
read_when:
    - Buscando definiciones de canales de lanzamiento públicos
    - Buscando nomenclatura de versiones y cadencia
summary: Canales de lanzamiento públicos, nomenclatura de versiones y cadencia
title: Política de lanzamiento
x-i18n:
    generated_at: "2026-04-14T02:08:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: fdc32839447205d74ba7a20a45fbac8e13b199174b442a1e260e3fce056c63da
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
  - Etiqueta de Git: `vYYYY.M.D`
- Versión de lanzamiento de corrección estable: `YYYY.M.D-N`
  - Etiqueta de Git: `vYYYY.M.D-N`
- Versión de prelanzamiento beta: `YYYY.M.D-beta.N`
  - Etiqueta de Git: `vYYYY.M.D-beta.N`
- No rellenes con ceros el mes ni el día
- `latest` significa el lanzamiento estable de npm promovido actual
- `beta` significa el objetivo de instalación beta actual
- Los lanzamientos estables y las correcciones estables publican en npm `beta` de forma predeterminada; los operadores de lanzamiento pueden dirigirse explícitamente a `latest`, o promover más adelante una compilación beta validada
- Cada lanzamiento de OpenClaw publica juntos el paquete de npm y la app de macOS

## Cadencia de lanzamiento

- Los lanzamientos pasan primero por beta
- stable llega solo después de validar la beta más reciente
- El procedimiento detallado de lanzamiento, las aprobaciones, las credenciales y las notas de recuperación son solo para mantenedores

## Comprobaciones previas al lanzamiento

- Ejecuta `pnpm build && pnpm ui:build` antes de `pnpm release:check` para que existan los artefactos de lanzamiento esperados en `dist/*` y el paquete de Control UI para el paso de validación del paquete
- Ejecuta `pnpm release:check` antes de cada lanzamiento etiquetado
- Las comprobaciones de lanzamiento ahora se ejecutan en un flujo de trabajo manual separado:
  `OpenClaw Release Checks`
- Esta separación es intencional: mantiene la ruta real de lanzamiento a npm corta, determinista y centrada en artefactos, mientras que las comprobaciones en vivo más lentas permanecen en su propio canal para que no ralenticen ni bloqueen la publicación
- Las comprobaciones de lanzamiento deben lanzarse desde la referencia de flujo de trabajo de `main` para que la lógica del flujo de trabajo y los secretos sigan siendo canónicos
- Ese flujo de trabajo acepta una etiqueta de lanzamiento existente o el SHA completo actual de 40 caracteres del commit de `main`
- En el modo de SHA de commit, solo acepta el HEAD actual de `origin/main`; usa una etiqueta de lanzamiento para commits de lanzamiento más antiguos
- La comprobación previa de solo validación de `OpenClaw NPM Release` también acepta el SHA completo actual de 40 caracteres del commit de `main` sin requerir una etiqueta enviada
- Esa ruta de SHA es solo de validación y no puede promoverse a una publicación real
- En modo SHA, el flujo de trabajo sintetiza `v<package.json version>` solo para la comprobación de metadatos del paquete; la publicación real sigue requiriendo una etiqueta de lanzamiento real
- Ambos flujos de trabajo mantienen la ruta real de publicación y promoción en runners alojados por GitHub, mientras que la ruta de validación no mutante puede usar los runners Linux Blacksmith más grandes
- Ese flujo de trabajo ejecuta
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  usando los secretos del flujo de trabajo `OPENAI_API_KEY` y `ANTHROPIC_API_KEY`
- La comprobación previa de lanzamiento a npm ya no espera al canal separado de comprobaciones de lanzamiento
- Ejecuta `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (o la etiqueta beta/corrección correspondiente) antes de la aprobación
- Después de publicar en npm, ejecuta
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (o la versión beta/corrección correspondiente) para verificar la ruta de instalación publicada del registro en un prefijo temporal nuevo
- La automatización de lanzamiento de mantenedores ahora usa comprobación previa y luego promoción:
  - la publicación real en npm debe pasar con un `preflight_run_id` exitoso de npm
  - los lanzamientos estables en npm usan `beta` de forma predeterminada
  - la publicación estable en npm puede dirigirse explícitamente a `latest` mediante una entrada del flujo de trabajo
  - la promoción estable en npm de `beta` a `latest` sigue disponible como un modo manual explícito en el flujo de trabajo confiable `OpenClaw NPM Release`
  - las publicaciones estables directas también pueden ejecutar un modo explícito de sincronización de dist-tag que apunta tanto `latest` como `beta` a la versión estable ya publicada
  - esos modos de dist-tag siguen necesitando un `NPM_TOKEN` válido en el entorno `npm-release` porque la gestión de `npm dist-tag` es independiente de la publicación confiable
  - la `macOS Release` pública es solo de validación
  - la publicación real privada de mac debe pasar con `preflight_run_id` y `validate_run_id` exitosos del mac privado
  - las rutas de publicación reales promocionan artefactos preparados en lugar de volver a compilarlos
- Para lanzamientos de corrección estable como `YYYY.M.D-N`, el verificador posterior a la publicación también comprueba la misma ruta de actualización con prefijo temporal desde `YYYY.M.D` hasta `YYYY.M.D-N` para que las correcciones de lanzamiento no dejen silenciosamente instalaciones globales antiguas en la carga útil estable base
- La comprobación previa de lanzamiento a npm falla de forma cerrada a menos que el tarball incluya tanto `dist/control-ui/index.html` como una carga útil no vacía en `dist/control-ui/assets/`, para que no volvamos a publicar un panel de navegador vacío
- Si el trabajo de lanzamiento tocó la planificación de CI, los manifiestos de temporización de extensiones o las matrices de prueba de extensiones, regenera y revisa las salidas de la matriz del flujo de trabajo `checks-node-extensions` propiedad del planificador desde `.github/workflows/ci.yml` antes de la aprobación para que las notas de lanzamiento no describan una disposición de CI obsoleta
- La preparación de lanzamiento estable de macOS también incluye las superficies del actualizador:
  - el lanzamiento en GitHub debe terminar con los archivos empaquetados `.zip`, `.dmg` y `.dSYM.zip`
  - `appcast.xml` en `main` debe apuntar al nuevo zip estable después de la publicación
  - la app empaquetada debe mantener un bundle id no de depuración, una URL de feed de Sparkle no vacía y un `CFBundleVersion` igual o superior al límite mínimo canónico de compilación de Sparkle para esa versión de lanzamiento

## Entradas del flujo de trabajo de NPM

`OpenClaw NPM Release` acepta estas entradas controladas por el operador:

- `tag`: etiqueta de lanzamiento requerida como `v2026.4.2`, `v2026.4.2-1`, o `v2026.4.2-beta.1`; cuando `preflight_only=true`, también puede ser el SHA completo actual de 40 caracteres del commit de `main` para una comprobación previa solo de validación
- `preflight_only`: `true` para solo validación/compilación/empaquetado, `false` para la ruta de publicación real
- `preflight_run_id`: obligatorio en la ruta de publicación real para que el flujo de trabajo reutilice el tarball preparado de la ejecución exitosa de la comprobación previa
- `npm_dist_tag`: etiqueta de destino de npm para la ruta de publicación; el valor predeterminado es `beta`
- `promote_beta_to_latest`: `true` para omitir la publicación y mover una compilación estable `beta` ya publicada a `latest`
- `sync_stable_dist_tags`: `true` para omitir la publicación y apuntar tanto `latest` como `beta` a una versión estable ya publicada

`OpenClaw Release Checks` acepta estas entradas controladas por el operador:

- `ref`: etiqueta de lanzamiento existente o el SHA completo actual de 40 caracteres del commit de `main` para validar

Reglas:

- Las etiquetas estables y de corrección pueden publicar en `beta` o en `latest`
- Las etiquetas beta de prelanzamiento solo pueden publicar en `beta`
- La entrada de SHA completo de commit solo se permite cuando `preflight_only=true`
- El modo SHA de commit de comprobaciones de lanzamiento también requiere el HEAD actual de `origin/main`
- La ruta de publicación real debe usar el mismo `npm_dist_tag` usado durante la comprobación previa; el flujo de trabajo verifica esos metadatos antes de continuar con la publicación
- El modo de promoción debe usar una etiqueta estable o de corrección, `preflight_only=false`, un `preflight_run_id` vacío y `npm_dist_tag=beta`
- El modo de sincronización de dist-tag debe usar una etiqueta estable o de corrección, `preflight_only=false`, un `preflight_run_id` vacío, `npm_dist_tag=latest`, y `promote_beta_to_latest=false`
- Los modos de promoción y sincronización de dist-tag también requieren un `NPM_TOKEN` válido porque `npm dist-tag add` sigue necesitando autenticación normal de npm; la publicación confiable cubre solo la ruta de publicación del paquete

## Secuencia de lanzamiento estable en npm

Al crear un lanzamiento estable en npm:

1. Ejecuta `OpenClaw NPM Release` con `preflight_only=true`
   - Antes de que exista una etiqueta, puedes usar el SHA completo actual de `main` para una ejecución en seco solo de validación del flujo de trabajo de comprobación previa
2. Elige `npm_dist_tag=beta` para el flujo normal que pasa primero por beta, o `latest` solo cuando intencionalmente quieras una publicación estable directa
3. Ejecuta `OpenClaw Release Checks` por separado con la misma etiqueta o el SHA completo actual de `main` cuando quieras cobertura en vivo de caché de prompts
   - Esto está separado a propósito para que la cobertura en vivo siga disponible sin volver a acoplar comprobaciones largas o inestables al flujo de trabajo de publicación
4. Guarda el `preflight_run_id` exitoso
5. Ejecuta `OpenClaw NPM Release` otra vez con `preflight_only=false`, la misma `tag`, el mismo `npm_dist_tag` y el `preflight_run_id` guardado
6. Si el lanzamiento llegó a `beta`, ejecuta `OpenClaw NPM Release` más adelante con la misma `tag` estable, `promote_beta_to_latest=true`, `preflight_only=false`, `preflight_run_id` vacío y `npm_dist_tag=beta` cuando quieras mover esa compilación publicada a `latest`
7. Si el lanzamiento se publicó intencionalmente directamente en `latest` y `beta` debe seguir la misma compilación estable, ejecuta `OpenClaw NPM Release` con la misma `tag` estable, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`, `preflight_only=false`, `preflight_run_id` vacío y `npm_dist_tag=latest`

Los modos de promoción y sincronización de dist-tag siguen requiriendo la aprobación del entorno `npm-release` y un `NPM_TOKEN` válido accesible para esa ejecución del flujo de trabajo.

Eso mantiene tanto la ruta de publicación directa como la ruta de promoción beta primero documentadas y visibles para el operador.

## Referencias públicas

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Los mantenedores usan la documentación privada de lanzamiento en
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
para el procedimiento real.
