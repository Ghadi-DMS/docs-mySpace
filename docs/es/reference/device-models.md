---
read_when:
    - Actualizas asignaciones de identificadores de modelo de dispositivos o archivos NOTICE/licencia
    - Cambias cómo la UI de Instances muestra los nombres de dispositivos
summary: Cómo OpenClaw incorpora identificadores de modelo de dispositivos Apple para mostrar nombres descriptivos en la app de macOS.
title: Base de datos de modelos de dispositivos
x-i18n:
    generated_at: "2026-04-05T12:52:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1d99c2538a0d8fdd80fa468fa402f63479ef2522e83745a0a46527a86238aeb2
    source_path: reference/device-models.md
    workflow: 15
---

# Base de datos de modelos de dispositivos (nombres descriptivos)

La app complementaria de macOS muestra nombres descriptivos de modelos de dispositivos Apple en la UI de **Instances** al asignar identificadores de modelo de Apple (por ejemplo `iPad16,6`, `Mac16,6`) a nombres legibles por personas.

La asignación se incorpora como JSON en:

- `apps/macos/Sources/OpenClaw/Resources/DeviceModels/`

## Fuente de datos

Actualmente incorporamos la asignación desde el repositorio con licencia MIT:

- `kyle-seongwoo-jun/apple-device-identifiers`

Para mantener compilaciones deterministas, los archivos JSON se fijan a commits específicos del upstream (registrados en `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md`).

## Actualizar la base de datos

1. Elige los commits del upstream que quieras fijar (uno para iOS, uno para macOS).
2. Actualiza los hashes de commit en `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md`.
3. Vuelve a descargar los archivos JSON, fijados a esos commits:

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. Asegúrate de que `apps/macos/Sources/OpenClaw/Resources/DeviceModels/LICENSE.apple-device-identifiers.txt` siga coincidiendo con el upstream (sustitúyelo si la licencia upstream cambia).
5. Verifica que la app de macOS se compile correctamente (sin advertencias):

```bash
swift build --package-path apps/macos
```
