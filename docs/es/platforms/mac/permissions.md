---
read_when:
    - Depurar prompts de permisos de macOS ausentes o bloqueados
    - Empaquetar o firmar la app de macOS
    - Cambiar identificadores de bundle o rutas de instalación de la app
summary: Persistencia de permisos de macOS (TCC) y requisitos de firma
title: Permisos de macOS
x-i18n:
    generated_at: "2026-04-05T12:48:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 250065b964c98c307a075ab9e23bf798f9d247f27befe2e5f271ffef1f497def
    source_path: platforms/mac/permissions.md
    workflow: 15
---

# Permisos de macOS (TCC)

Las concesiones de permisos en macOS son frágiles. TCC asocia una concesión de permiso con la
firma de código de la app, el identificador del bundle y la ruta en disco. Si cualquiera de ellos cambia,
macOS trata la app como nueva y puede descartar u ocultar los prompts.

## Requisitos para permisos estables

- Misma ruta: ejecuta la app desde una ubicación fija (para OpenClaw, `dist/OpenClaw.app`).
- Mismo identificador de bundle: cambiar el ID del bundle crea una nueva identidad de permiso.
- App firmada: las compilaciones sin firma o firmadas ad hoc no conservan permisos.
- Firma coherente: usa un certificado real de Apple Development o Developer ID
  para que la firma se mantenga estable entre reconstrucciones.

Las firmas ad hoc generan una nueva identidad en cada compilación. macOS olvidará
las concesiones anteriores, y los prompts pueden desaparecer por completo hasta que se borren las entradas obsoletas.

## Lista de recuperación cuando desaparecen los prompts

1. Cierra la app.
2. Elimina la entrada de la app en System Settings -> Privacy & Security.
3. Vuelve a iniciar la app desde la misma ruta y vuelve a conceder los permisos.
4. Si el prompt sigue sin aparecer, restablece las entradas de TCC con `tccutil` y vuelve a intentarlo.
5. Algunos permisos solo reaparecen después de un reinicio completo de macOS.

Ejemplos de restablecimiento (sustituye el ID del bundle según sea necesario):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Permisos de archivos y carpetas (Desktop/Documents/Downloads)

macOS también puede restringir Desktop, Documents y Downloads para procesos de terminal/en segundo plano. Si las lecturas de archivos o los listados de directorios se bloquean, concede acceso al mismo contexto de proceso que realiza las operaciones de archivo (por ejemplo Terminal/iTerm, app iniciada por LaunchAgent o proceso SSH).

Solución alternativa: mueve los archivos al espacio de trabajo de OpenClaw (`~/.openclaw/workspace`) si quieres evitar concesiones por carpeta.

Si estás probando permisos, firma siempre con un certificado real. Las compilaciones ad hoc
solo son aceptables para ejecuciones locales rápidas donde los permisos no importan.
