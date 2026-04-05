---
read_when:
    - Ejecutar más de un Gateway en la misma máquina
    - Necesitas configuración/estado/puertos aislados por Gateway
summary: Ejecuta varios Gateways de OpenClaw en un mismo host (aislamiento, puertos y perfiles)
title: Varios Gateways
x-i18n:
    generated_at: "2026-04-05T12:42:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 061f204bf56b28c6bd0e2c9aee6c561a8a162ca219060117fea4d3a007f01899
    source_path: gateway/multiple-gateways.md
    workflow: 15
---

# Varios Gateways (mismo host)

La mayoría de las configuraciones deberían usar un solo Gateway porque un único Gateway puede gestionar múltiples conexiones de mensajería y agentes. Si necesitas un aislamiento o redundancia más fuertes (por ejemplo, un bot de rescate), ejecuta Gateways separados con perfiles/puertos aislados.

## Lista de comprobación de aislamiento (obligatoria)

- `OPENCLAW_CONFIG_PATH` — archivo de configuración por instancia
- `OPENCLAW_STATE_DIR` — sesiones, credenciales y cachés por instancia
- `agents.defaults.workspace` — raíz del espacio de trabajo por instancia
- `gateway.port` (o `--port`) — único por instancia
- Los puertos derivados (browser/canvas) no deben solaparse

Si estos se comparten, tendrás conflictos de configuración y de puertos.

## Recomendado: perfiles (`--profile`)

Los perfiles delimitan automáticamente `OPENCLAW_STATE_DIR` + `OPENCLAW_CONFIG_PATH` y añaden un sufijo a los nombres de los servicios.

```bash
# main
openclaw --profile main setup
openclaw --profile main gateway --port 18789

# rescue
openclaw --profile rescue setup
openclaw --profile rescue gateway --port 19001
```

Servicios por perfil:

```bash
openclaw --profile main gateway install
openclaw --profile rescue gateway install
```

## Guía del bot de rescate

Ejecuta un segundo Gateway en el mismo host con su propio:

- perfil/configuración
- directorio de estado
- espacio de trabajo
- puerto base (más puertos derivados)

Esto mantiene el bot de rescate aislado del bot principal para que pueda depurar o aplicar cambios de configuración si el bot principal está caído.

Espaciado de puertos: deja al menos 20 puertos entre puertos base para que los puertos derivados de browser/canvas/CDP nunca colisionen.

### Cómo instalarlo (bot de rescate)

```bash
# Main bot (existing or fresh, without --profile param)
# Runs on port 18789 + Chrome CDC/Canvas/... Ports
openclaw onboard
openclaw gateway install

# Rescue bot (isolated profile + ports)
openclaw --profile rescue onboard
# Notes:
# - workspace name will be postfixed with -rescue per default
# - Port should be at least 18789 + 20 Ports,
#   better choose completely different base port, like 19789,
# - rest of the onboarding is the same as normal

# To install the service (if not happened automatically during setup)
openclaw --profile rescue gateway install
```

## Asignación de puertos (derivados)

Puerto base = `gateway.port` (o `OPENCLAW_GATEWAY_PORT` / `--port`).

- puerto del servicio de control del navegador = base + 2 (solo loopback)
- el host de canvas se sirve en el servidor HTTP del Gateway (mismo puerto que `gateway.port`)
- Los puertos CDP del perfil del navegador se asignan automáticamente desde `browser.controlPort + 9 .. + 108`

Si sobrescribes cualquiera de estos en la configuración o en variables de entorno, debes mantenerlos únicos por instancia.

## Notas sobre browser/CDP (error habitual)

- **No** fijes `browser.cdpUrl` a los mismos valores en varias instancias.
- Cada instancia necesita su propio puerto de control del navegador y su propio rango de CDP (derivado de su puerto de gateway).
- Si necesitas puertos CDP explícitos, establece `browser.profiles.<name>.cdpPort` por instancia.
- Chrome remoto: usa `browser.profiles.<name>.cdpUrl` (por perfil y por instancia).

## Ejemplo manual con variables de entorno

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw-main \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19001
```

## Comprobaciones rápidas

```bash
openclaw --profile main gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw --profile main status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

Interpretación:

- `gateway status --deep` ayuda a detectar servicios obsoletos de launchd/systemd/schtasks de instalaciones anteriores.
- El texto de advertencia de `gateway probe`, como `multiple reachable gateways detected`, solo es esperable cuando ejecutas intencionadamente más de un gateway aislado.
