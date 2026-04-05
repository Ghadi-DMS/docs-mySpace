---
read_when:
    - Compilando o firmando compilaciones de depuración para Mac
summary: Pasos de firma para compilaciones de depuración de macOS generadas por scripts de empaquetado
title: Firma en macOS
x-i18n:
    generated_at: "2026-04-05T12:48:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b16d726549cf6dc34dc9c60e14d8041426ebc0699ab59628aca1d094380334a
    source_path: platforms/mac/signing.md
    workflow: 15
---

# Firma en macOS (compilaciones de depuración)

Esta app normalmente se compila desde [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh), que ahora:

- establece un identificador de bundle de depuración estable: `ai.openclaw.mac.debug`
- escribe el Info.plist con ese id de bundle (sobrescríbelo mediante `BUNDLE_ID=...`)
- llama a [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) para firmar el binario principal y el bundle de la app, de modo que macOS trate cada recompilación como el mismo bundle firmado y mantenga los permisos TCC (notificaciones, accesibilidad, grabación de pantalla, micrófono, voz). Para permisos estables, usa una identidad de firma real; la firma ad-hoc es opt-in y frágil (consulta [permisos de macOS](/platforms/mac/permissions)).
- usa `CODESIGN_TIMESTAMP=auto` de forma predeterminada; habilita marcas de tiempo confiables para firmas Developer ID. Configura `CODESIGN_TIMESTAMP=off` para omitir el timestamping (compilaciones de depuración offline).
- inyecta metadatos de compilación en Info.plist: `OpenClawBuildTimestamp` (UTC) y `OpenClawGitCommit` (hash corto) para que el panel About pueda mostrar compilación, git y canal debug/release.
- **El empaquetado usa Node 24 por defecto**: el script ejecuta compilaciones de TS y la compilación de la Control UI. Node 22 LTS, actualmente `22.14+`, sigue siendo compatible por compatibilidad.
- lee `SIGN_IDENTITY` desde el entorno. Agrega `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"` (o tu certificado Developer ID Application) a tu shell rc para firmar siempre con tu certificado. La firma ad-hoc requiere activación explícita mediante `ALLOW_ADHOC_SIGNING=1` o `SIGN_IDENTITY="-"` (no se recomienda para pruebas de permisos).
- ejecuta una auditoría de Team ID después de firmar y falla si cualquier Mach-O dentro del bundle de la app está firmado con un Team ID distinto. Configura `SKIP_TEAM_ID_CHECK=1` para omitirla.

## Uso

```bash
# desde la raíz del repositorio
scripts/package-mac-app.sh               # selecciona identidad automáticamente; da error si no encuentra ninguna
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # certificado real
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh    # ad-hoc (los permisos no se conservarán)
SIGN_IDENTITY="-" scripts/package-mac-app.sh        # ad-hoc explícito (misma advertencia)
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh   # solución temporal solo para desarrollo ante desajuste de Team ID de Sparkle
```

### Nota sobre firma ad-hoc

Al firmar con `SIGN_IDENTITY="-"` (ad-hoc), el script desactiva automáticamente el **Hardened Runtime** (`--options runtime`). Esto es necesario para evitar fallos cuando la app intenta cargar frameworks integrados (como Sparkle) que no comparten el mismo Team ID. Las firmas ad-hoc también rompen la persistencia de permisos TCC; consulta [permisos de macOS](/platforms/mac/permissions) para ver los pasos de recuperación.

## Metadatos de compilación para About

`package-mac-app.sh` marca el bundle con:

- `OpenClawBuildTimestamp`: ISO8601 UTC en el momento del empaquetado
- `OpenClawGitCommit`: hash corto de git (o `unknown` si no está disponible)

La pestaña About lee estas claves para mostrar la versión, fecha de compilación, commit de git y si se trata de una compilación de depuración (mediante `#if DEBUG`). Ejecuta el empaquetador para actualizar estos valores después de cambios en el código.

## Por qué

Los permisos TCC están vinculados al identificador del bundle _y_ a la firma del código. Las compilaciones de depuración sin firmar con UUID cambiantes hacían que macOS olvidara las concesiones después de cada recompilación. Firmar los binarios (ad-hoc por defecto) y mantener un id/ruta de bundle fijo (`dist/OpenClaw.app`) conserva las concesiones entre compilaciones, igual que el enfoque de VibeTunnel.
