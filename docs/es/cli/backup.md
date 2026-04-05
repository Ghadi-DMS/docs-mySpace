---
read_when:
    - Quieres un archivo de respaldo de primera clase para el estado local de OpenClaw
    - Quieres previsualizar qué rutas se incluirían antes de restablecer o desinstalar
summary: Referencia de la CLI para `openclaw backup` (crear archivos locales de respaldo)
title: backup
x-i18n:
    generated_at: "2026-04-05T12:37:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 700eda8f9eac1cc93a854fa579f128e5e97d4e6dfc0da75b437c0fb2a898a37d
    source_path: cli/backup.md
    workflow: 15
---

# `openclaw backup`

Crea un archivo local de respaldo para el estado de OpenClaw, la configuración, los perfiles de autenticación, las credenciales de canal/proveedor, las sesiones y, opcionalmente, los espacios de trabajo.

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T00-00-00.000Z-openclaw-backup.tar.gz
```

## Notas

- El archivo incluye un archivo `manifest.json` con las rutas de origen resueltas y la estructura del archivo.
- La salida predeterminada es un archivo `.tar.gz` con marca de tiempo en el directorio de trabajo actual.
- Si el directorio de trabajo actual está dentro de un árbol de origen respaldado, OpenClaw recurre a tu directorio personal como ubicación predeterminada del archivo.
- Los archivos de respaldo existentes nunca se sobrescriben.
- Las rutas de salida dentro de los árboles de estado/espacio de trabajo de origen se rechazan para evitar la auto-inclusión.
- `openclaw backup verify <archive>` valida que el archivo contenga exactamente un manifiesto raíz, rechaza rutas de archivo de estilo traversal y comprueba que cada carga útil declarada en el manifiesto exista en el tarball.
- `openclaw backup create --verify` ejecuta esa validación inmediatamente después de escribir el archivo.
- `openclaw backup create --only-config` respalda solo el archivo de configuración JSON activo.

## Qué se respalda

`openclaw backup create` planifica las fuentes de respaldo a partir de tu instalación local de OpenClaw:

- El directorio de estado devuelto por el resolvedor de estado local de OpenClaw, normalmente `~/.openclaw`
- La ruta del archivo de configuración activo
- El directorio `credentials/` resuelto cuando existe fuera del directorio de estado
- Los directorios de espacio de trabajo detectados a partir de la configuración actual, a menos que pases `--no-include-workspace`

Los perfiles de autenticación de modelos ya forman parte del directorio de estado en
`agents/<agentId>/agent/auth-profiles.json`, por lo que normalmente quedan cubiertos por la
entrada de respaldo del estado.

Si usas `--only-config`, OpenClaw omite el estado, la detección del directorio de credenciales y del espacio de trabajo, y archiva solo la ruta del archivo de configuración activo.

OpenClaw canoniza las rutas antes de crear el archivo. Si la configuración, el
directorio de credenciales o un espacio de trabajo ya están dentro del directorio de estado,
no se duplican como fuentes de respaldo de nivel superior independientes. Las rutas ausentes se
omiten.

La carga útil del archivo almacena el contenido de los archivos de esos árboles de origen, y el `manifest.json` incrustado registra las rutas de origen absolutas resueltas junto con la estructura del archivo usada para cada recurso.

## Comportamiento con configuración no válida

`openclaw backup` omite intencionalmente la comprobación previa normal de la configuración para poder seguir ayudando durante la recuperación. Como la detección del espacio de trabajo depende de una configuración válida, `openclaw backup create` ahora falla rápidamente cuando el archivo de configuración existe pero no es válido y el respaldo del espacio de trabajo sigue habilitado.

Si aun así quieres un respaldo parcial en esa situación, vuelve a ejecutar:

```bash
openclaw backup create --no-include-workspace
```

Esto mantiene dentro del alcance el estado, la configuración y el directorio de credenciales externo, al tiempo que omite por completo la detección del espacio de trabajo.

Si solo necesitas una copia del propio archivo de configuración, `--only-config` también funciona cuando la configuración está mal formada porque no depende de analizar la configuración para detectar el espacio de trabajo.

## Tamaño y rendimiento

OpenClaw no aplica un tamaño máximo integrado para el respaldo ni un límite de tamaño por archivo.

Los límites prácticos provienen de la máquina local y del sistema de archivos de destino:

- Espacio disponible para la escritura temporal del archivo más el archivo final
- Tiempo para recorrer árboles grandes de espacio de trabajo y comprimirlos en un `.tar.gz`
- Tiempo para volver a analizar el archivo si usas `openclaw backup create --verify` o ejecutas `openclaw backup verify`
- Comportamiento del sistema de archivos en la ruta de destino. OpenClaw prefiere un paso de publicación con hard link sin sobrescritura y recurre a la copia exclusiva cuando los hard links no son compatibles

Los espacios de trabajo grandes suelen ser el principal factor del tamaño del archivo. Si quieres un respaldo más pequeño o más rápido, usa `--no-include-workspace`.

Para el archivo más pequeño, usa `--only-config`.
