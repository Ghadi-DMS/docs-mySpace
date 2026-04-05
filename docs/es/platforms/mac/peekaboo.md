---
read_when:
    - Alojar PeekabooBridge en OpenClaw.app
    - Integrar Peekaboo mediante Swift Package Manager
    - Cambiar el protocolo o las rutas de PeekabooBridge
summary: Integración de PeekabooBridge para automatización de UI en macOS
title: Peekaboo Bridge
x-i18n:
    generated_at: "2026-04-05T12:48:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 30961eb502eecd23c017b58b834bd8cb00cab8b17302617d541afdace3ad8dba
    source_path: platforms/mac/peekaboo.md
    workflow: 15
---

# Peekaboo Bridge (automatización de UI en macOS)

OpenClaw puede alojar **PeekabooBridge** como un intermediario local de automatización de UI con reconocimiento de permisos. Esto permite que la CLI `peekaboo` controle la automatización de UI mientras reutiliza los permisos TCC de la app de macOS.

## Qué es esto (y qué no es)

- **Host**: OpenClaw.app puede actuar como host de PeekabooBridge.
- **Cliente**: usa la CLI `peekaboo` (sin una superficie separada `openclaw ui ...`).
- **UI**: las superposiciones visuales permanecen en Peekaboo.app; OpenClaw es un host intermediario fino.

## Habilitar el bridge

En la app de macOS:

- Settings → **Enable Peekaboo Bridge**

Cuando está habilitado, OpenClaw inicia un servidor local de socket UNIX. Si está deshabilitado, el host se detiene y `peekaboo` recurrirá a otros hosts disponibles.

## Orden de descubrimiento del cliente

Los clientes de Peekaboo normalmente prueban hosts en este orden:

1. Peekaboo.app (UX completa)
2. Claude.app (si está instalada)
3. OpenClaw.app (intermediario fino)

Usa `peekaboo bridge status --verbose` para ver qué host está activo y qué
ruta de socket está en uso. Puedes reemplazarlo con:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## Seguridad y permisos

- El bridge valida las **firmas de código del llamador**; se aplica una allowlist de TeamID (TeamID del host de Peekaboo + TeamID de la app OpenClaw).
- Las solicitudes agotan el tiempo tras ~10 segundos.
- Si faltan permisos obligatorios, el bridge devuelve un mensaje de error claro en lugar de abrir System Settings.

## Comportamiento de snapshots (automatización)

Los snapshots se almacenan en memoria y vencen automáticamente después de un breve período.
Si necesitas una retención más larga, vuelve a capturarlos desde el cliente.

## Solución de problemas

- Si `peekaboo` informa “bridge client is not authorized”, asegúrate de que el cliente esté correctamente firmado o ejecuta el host con `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` solo en modo **debug**.
- Si no se encuentra ningún host, abre una de las apps host (Peekaboo.app u OpenClaw.app) y confirma que los permisos estén concedidos.
