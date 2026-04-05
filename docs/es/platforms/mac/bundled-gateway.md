---
read_when:
    - Empaquetar OpenClaw.app
    - Depurar el servicio launchd del gateway en macOS
    - Instalar la CLI del gateway para macOS
summary: Runtime del Gateway en macOS (servicio launchd externo)
title: Gateway en macOS
x-i18n:
    generated_at: "2026-04-05T12:48:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 69e41528b35d69c13608cb9a34b39a7f02e1134204d1b496cbdd191798f39607
    source_path: platforms/mac/bundled-gateway.md
    workflow: 15
---

# Gateway en macOS (launchd externo)

OpenClaw.app ya no incluye Node/Bun ni el runtime del Gateway. La app de macOS
espera una instalación **externa** de la CLI `openclaw`, no inicia el Gateway como
proceso hijo y administra un servicio launchd por usuario para mantener el Gateway
en ejecución (o se conecta a un Gateway local existente si ya hay uno en ejecución).

## Instalar la CLI (obligatorio para el modo local)

Node 24 es el runtime predeterminado en Mac. Node 22 LTS, actualmente `22.14+`, sigue funcionando por compatibilidad. Luego instala `openclaw` globalmente:

```bash
npm install -g openclaw@<version>
```

El botón **Install CLI** de la app de macOS ejecuta el mismo flujo de instalación global que la app
usa internamente: da preferencia a npm primero, luego pnpm y después bun si ese es el único
gestor de paquetes detectado. Node sigue siendo el runtime recomendado para el Gateway.

## Launchd (Gateway como LaunchAgent)

Etiqueta:

- `ai.openclaw.gateway` (o `ai.openclaw.<profile>`; los `com.openclaw.*` heredados pueden permanecer)

Ubicación del plist (por usuario):

- `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
  (o `~/Library/LaunchAgents/ai.openclaw.<profile>.plist`)

Administrador:

- La app de macOS administra la instalación/actualización de LaunchAgent en modo local.
- La CLI también puede instalarlo: `openclaw gateway install`.

Comportamiento:

- “OpenClaw Active” habilita/deshabilita el LaunchAgent.
- Cerrar la app **no** detiene el gateway (launchd lo mantiene activo).
- Si ya hay un Gateway en ejecución en el puerto configurado, la app se conecta a
  él en lugar de iniciar uno nuevo.

Registro:

- stdout/err de launchd: `/tmp/openclaw/openclaw-gateway.log`

## Compatibilidad de versiones

La app de macOS comprueba la versión del gateway con respecto a su propia versión. Si son
incompatibles, actualiza la CLI global para que coincida con la versión de la app.

## Comprobación rápida

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

Luego:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```
