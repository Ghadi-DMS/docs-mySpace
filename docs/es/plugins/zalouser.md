---
read_when:
    - Quieres compatibilidad con Zalo Personal (no oficial) en OpenClaw
    - Estás configurando o desarrollando el plugin zalouser
summary: 'Plugin de Zalo Personal: inicio de sesión por QR + mensajería mediante `zca-js` nativo (instalación del plugin + configuración del canal + herramienta)'
title: Plugin de Zalo Personal
x-i18n:
    generated_at: "2026-04-05T12:50:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3218c3ee34f36466d952aec1b479d451a6235c7c46918beb28698234a7fd0968
    source_path: plugins/zalouser.md
    workflow: 15
---

# Zalo Personal (plugin)

Compatibilidad con Zalo Personal para OpenClaw mediante un plugin, usando `zca-js` nativo para automatizar una cuenta normal de usuario de Zalo.

> **Warning:** La automatización no oficial puede provocar suspensión/bloqueo de la cuenta. Úsalo bajo tu propia responsabilidad.

## Nombre

El ID del canal es `zalouser` para dejar claro que esto automatiza una **cuenta personal de usuario de Zalo** (no oficial). Reservamos `zalo` para una posible integración futura con la API oficial de Zalo.

## Dónde se ejecuta

Este plugin se ejecuta **dentro del proceso del Gateway**.

Si usas un Gateway remoto, instálalo/configúralo en la **máquina que ejecuta el Gateway** y luego reinicia el Gateway.

No se requiere ningún binario CLI externo `zca`/`openzca`.

## Instalación

### Opción A: instalar desde npm

```bash
openclaw plugins install @openclaw/zalouser
```

Después reinicia el Gateway.

### Opción B: instalar desde una carpeta local (desarrollo)

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

Después reinicia el Gateway.

## Configuración

La configuración del canal se encuentra en `channels.zalouser` (no en `plugins.entries.*`):

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hello from OpenClaw"
openclaw directory peers list --channel zalouser --query "name"
```

## Herramienta del agente

Nombre de la herramienta: `zalouser`

Acciones: `send`, `image`, `link`, `friends`, `groups`, `me`, `status`

Las acciones de mensajes del canal también admiten `react` para reacciones a mensajes.
