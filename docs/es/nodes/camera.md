---
read_when:
    - Agregar o modificar la captura de cámara en nodos iOS/Android o macOS
    - Extender flujos de trabajo de archivos temporales MEDIA accesibles para el agente
summary: 'Captura de cámara (nodos iOS/Android + app de macOS) para uso del agente: fotos (jpg) y clips de video cortos (mp4)'
title: Captura de cámara
x-i18n:
    generated_at: "2026-04-05T12:47:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 30b1beaac9602ff29733f72b953065f271928743c8fff03191a007e8b965c88d
    source_path: nodes/camera.md
    workflow: 15
---

# Captura de cámara (agente)

OpenClaw admite **captura de cámara** para flujos de trabajo del agente:

- **Nodo iOS** (emparejado mediante Gateway): captura una **foto** (`jpg`) o un **clip de video corto** (`mp4`, con audio opcional) mediante `node.invoke`.
- **Nodo Android** (emparejado mediante Gateway): captura una **foto** (`jpg`) o un **clip de video corto** (`mp4`, con audio opcional) mediante `node.invoke`.
- **App de macOS** (nodo mediante Gateway): captura una **foto** (`jpg`) o un **clip de video corto** (`mp4`, con audio opcional) mediante `node.invoke`.

Todo el acceso a la cámara está protegido mediante **configuraciones controladas por la persona usuaria**.

## Nodo iOS

### Configuración de usuario (activada de forma predeterminada)

- Pestaña de Settings de iOS → **Camera** → **Allow Camera** (`camera.enabled`)
  - Predeterminado: **activado** (una clave ausente se trata como habilitada).
  - Cuando está desactivado: los comandos `camera.*` devuelven `CAMERA_DISABLED`.

### Comandos (mediante Gateway `node.invoke`)

- `camera.list`
  - Carga útil de respuesta:
    - `devices`: matriz de `{ id, name, position, deviceType }`

- `camera.snap`
  - Parámetros:
    - `facing`: `front|back` (predeterminado: `front`)
    - `maxWidth`: número (opcional; predeterminado `1600` en el nodo iOS)
    - `quality`: `0..1` (opcional; predeterminado `0.9`)
    - `format`: actualmente `jpg`
    - `delayMs`: número (opcional; predeterminado `0`)
    - `deviceId`: cadena (opcional; de `camera.list`)
  - Carga útil de respuesta:
    - `format: "jpg"`
    - `base64: "<...>"`
    - `width`, `height`
  - Protección de carga útil: las fotos se recomprimen para mantener la carga útil base64 por debajo de 5 MB.

- `camera.clip`
  - Parámetros:
    - `facing`: `front|back` (predeterminado: `front`)
    - `durationMs`: número (predeterminado `3000`, limitado a un máximo de `60000`)
    - `includeAudio`: booleano (predeterminado `true`)
    - `format`: actualmente `mp4`
    - `deviceId`: cadena (opcional; de `camera.list`)
  - Carga útil de respuesta:
    - `format: "mp4"`
    - `base64: "<...>"`
    - `durationMs`
    - `hasAudio`

### Requisito de primer plano

Al igual que `canvas.*`, el nodo iOS solo permite comandos `camera.*` en **primer plano**. Las invocaciones en segundo plano devuelven `NODE_BACKGROUND_UNAVAILABLE`.

### Helper de CLI (archivos temporales + MEDIA)

La forma más sencilla de obtener archivos adjuntos es mediante el helper de CLI, que escribe el archivo multimedia decodificado en un archivo temporal e imprime `MEDIA:<path>`.

Ejemplos:

```bash
openclaw nodes camera snap --node <id>               # default: both front + back (2 MEDIA lines)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

Notas:

- `nodes camera snap` usa de forma predeterminada **ambas** orientaciones para dar al agente ambas vistas.
- Los archivos de salida son temporales (en el directorio temporal del sistema operativo) a menos que construyas tu propio wrapper.

## Nodo Android

### Configuración de usuario de Android (activada de forma predeterminada)

- Hoja de Settings de Android → **Camera** → **Allow Camera** (`camera.enabled`)
  - Predeterminado: **activado** (una clave ausente se trata como habilitada).
  - Cuando está desactivado: los comandos `camera.*` devuelven `CAMERA_DISABLED`.

### Permisos

- Android requiere permisos en tiempo de ejecución:
  - `CAMERA` para `camera.snap` y `camera.clip`.
  - `RECORD_AUDIO` para `camera.clip` cuando `includeAudio=true`.

Si faltan permisos, la app mostrará un prompt cuando sea posible; si se deniegan, las solicitudes `camera.*` fallan con un error
`*_PERMISSION_REQUIRED`.

### Requisito de primer plano en Android

Al igual que `canvas.*`, el nodo Android solo permite comandos `camera.*` en **primer plano**. Las invocaciones en segundo plano devuelven `NODE_BACKGROUND_UNAVAILABLE`.

### Comandos de Android (mediante Gateway `node.invoke`)

- `camera.list`
  - Carga útil de respuesta:
    - `devices`: matriz de `{ id, name, position, deviceType }`

### Protección de carga útil

Las fotos se recomprimen para mantener la carga útil base64 por debajo de 5 MB.

## App de macOS

### Configuración de usuario (desactivada de forma predeterminada)

La app complementaria de macOS expone una casilla de verificación:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`)
  - Predeterminado: **desactivado**
  - Cuando está desactivado: las solicitudes de cámara devuelven “Camera disabled by user”.

### Helper de CLI (node invoke)

Usa la CLI principal de `openclaw` para invocar comandos de cámara en el nodo macOS.

Ejemplos:

```bash
openclaw nodes camera list --node <id>            # list camera ids
openclaw nodes camera snap --node <id>            # prints MEDIA:<path>
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s          # prints MEDIA:<path>
openclaw nodes camera clip --node <id> --duration-ms 3000      # prints MEDIA:<path> (legacy flag)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

Notas:

- `openclaw nodes camera snap` usa de forma predeterminada `maxWidth=1600`, salvo que se anule.
- En macOS, `camera.snap` espera `delayMs` (predeterminado 2000ms) después del calentamiento/estabilización de la exposición antes de capturar.
- Las cargas útiles de fotos se recomprimen para mantener el base64 por debajo de 5 MB.

## Seguridad + límites prácticos

- El acceso a cámara y micrófono activa los prompts habituales de permisos del sistema operativo (y requiere cadenas de uso en `Info.plist`).
- Los clips de video están limitados (actualmente `<= 60s`) para evitar cargas útiles de nodo demasiado grandes (sobrecarga de base64 + límites de mensajes).

## Video de pantalla de macOS (a nivel del sistema operativo)

Para video de _pantalla_ (no de cámara), usa el complemento de macOS:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # prints MEDIA:<path>
```

Notas:

- Requiere el permiso de macOS **Screen Recording** (TCC).
