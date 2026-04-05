---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: Soluciona problemas de inicio de CDP en Chrome/Brave/Edge/Chromium para el control del navegador de OpenClaw en Linux
title: Solución de problemas del navegador
x-i18n:
    generated_at: "2026-04-05T12:54:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ff8e6741558c1b5db86826c5e1cbafe35e35afe5cb2a53296c16653da59e516
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 15
---

# Solución de problemas del navegador (Linux)

## Problema: "Failed to start Chrome CDP on port 18800"

El servidor de control del navegador de OpenClaw no logra iniciar Chrome/Brave/Edge/Chromium con el error:

```
{"error":"Error: Failed to start Chrome CDP on port 18800 for profile \"openclaw\"."}
```

### Causa raíz

En Ubuntu (y en muchas distribuciones de Linux), la instalación predeterminada de Chromium es un **paquete snap**. El confinamiento AppArmor de snap interfiere con la forma en que OpenClaw inicia y supervisa el proceso del navegador.

El comando `apt install chromium` instala un paquete de sustitución que redirige a snap:

```
Note, selecting 'chromium-browser' instead of 'chromium'
chromium-browser is already the newest version (2:1snap1-0ubuntu2).
```

Esto NO es un navegador real; es solo un contenedor.

### Solución 1: Instalar Google Chrome (recomendado)

Instala el paquete oficial `.deb` de Google Chrome, que no está aislado por snap:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # si hay errores de dependencias
```

Luego actualiza tu configuración de OpenClaw (`~/.openclaw/openclaw.json`):

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Solución 2: Usar Snap Chromium con el modo solo conexión

Si debes usar Chromium de snap, configura OpenClaw para conectarse a un navegador iniciado manualmente:

1. Actualiza la configuración:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

2. Inicia Chromium manualmente:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

3. Opcionalmente crea un servicio de usuario de systemd para iniciar Chrome automáticamente:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw Browser (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Actívalo con: `systemctl --user enable --now openclaw-browser.service`

### Verificar que el navegador funciona

Comprueba el estado:

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
```

Prueba la navegación:

```bash
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### Referencia de configuración

| Opción                   | Descripción                                                          | Predeterminado                                              |
| ------------------------ | -------------------------------------------------------------------- | ----------------------------------------------------------- |
| `browser.enabled`        | Habilita el control del navegador                                    | `true`                                                      |
| `browser.executablePath` | Ruta a un binario de navegador basado en Chromium (Chrome/Brave/Edge/Chromium) | detectado automáticamente (prefiere el navegador predeterminado cuando está basado en Chromium) |
| `browser.headless`       | Ejecuta sin GUI                                                      | `false`                                                     |
| `browser.noSandbox`      | Añade la marca `--no-sandbox` (necesaria en algunas configuraciones de Linux) | `false`                                                     |
| `browser.attachOnly`     | No inicia el navegador, solo se conecta a uno existente              | `false`                                                     |
| `browser.cdpPort`        | Puerto de Chrome DevTools Protocol                                   | `18800`                                                     |

### Problema: "No Chrome tabs found for profile=\"user\""

Estás usando un perfil `existing-session` / Chrome MCP. OpenClaw puede ver Chrome local,
pero no hay pestañas abiertas disponibles a las que conectarse.

Opciones para solucionarlo:

1. **Usa el navegador administrado:** `openclaw browser start --browser-profile openclaw`
   (o establece `browser.defaultProfile: "openclaw"`).
2. **Usa Chrome MCP:** asegúrate de que Chrome local esté en ejecución con al menos una pestaña abierta, luego vuelve a intentarlo con `--browser-profile user`.

Notas:

- `user` es solo para el host. Para servidores Linux, contenedores o hosts remotos, prefiere perfiles CDP.
- `user` y otros perfiles `existing-session` mantienen los límites actuales de Chrome MCP:
  acciones basadas en referencias, hooks de carga de un solo archivo, sin anulaciones de tiempo de espera de diálogos, sin
  `wait --load networkidle`, y sin `responsebody`, exportación a PDF, interceptación de descargas
  ni acciones por lotes.
- Los perfiles locales `openclaw` asignan automáticamente `cdpPort`/`cdpUrl`; configúralos solo para CDP remoto.
- Los perfiles CDP remotos aceptan `http://`, `https://`, `ws://` y `wss://`.
  Usa HTTP(S) para el descubrimiento de `/json/version`, o WS(S) cuando tu servicio
  de navegador te proporcione una URL directa de socket DevTools.
