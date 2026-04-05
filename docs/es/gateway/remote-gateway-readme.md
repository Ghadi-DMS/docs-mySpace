---
read_when: Connecting the macOS app to a remote gateway over SSH
summary: Configuración de túnel SSH para que OpenClaw.app se conecte a un gateway remoto
title: Configuración de gateway remoto
x-i18n:
    generated_at: "2026-04-05T12:42:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 55467956a3473fa36709715f017369471428f7566132f7feb47581caa98b4600
    source_path: gateway/remote-gateway-readme.md
    workflow: 15
---

> Este contenido se ha fusionado en [Acceso remoto](/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent). Consulta esa página para ver la guía actual.

# Ejecutar OpenClaw.app con un gateway remoto

OpenClaw.app usa túneles SSH para conectarse a un gateway remoto. Esta guía muestra cómo configurarlo.

## Resumen

```mermaid
flowchart TB
    subgraph Client["Máquina cliente"]
        direction TB
        A["OpenClaw.app"]
        B["ws://127.0.0.1:18789\n(puerto local)"]
        T["Túnel SSH"]

        A --> B
        B --> T
    end
    subgraph Remote["Máquina remota"]
        direction TB
        C["WebSocket del gateway"]
        D["ws://127.0.0.1:18789"]

        C --> D
    end
    T --> C
```

## Configuración rápida

### Paso 1: Agregar configuración SSH

Edita `~/.ssh/config` y agrega:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>          # p. ej., 172.27.187.184
    User <REMOTE_USER>            # p. ej., jefferson
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

Reemplaza `<REMOTE_IP>` y `<REMOTE_USER>` por tus valores.

### Paso 2: Copiar la clave SSH

Copia tu clave pública a la máquina remota (introduce la contraseña una vez):

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

### Paso 3: Configurar la autenticación del gateway remoto

```bash
openclaw config set gateway.remote.token "<your-token>"
```

Usa `gateway.remote.password` en su lugar si tu gateway remoto usa autenticación por contraseña.
`OPENCLAW_GATEWAY_TOKEN` sigue siendo válido como reemplazo a nivel de shell, pero la
configuración persistente de cliente remoto es `gateway.remote.token` / `gateway.remote.password`.

### Paso 4: Iniciar el túnel SSH

```bash
ssh -N remote-gateway &
```

### Paso 5: Reiniciar OpenClaw.app

```bash
# Sal de OpenClaw.app (⌘Q) y luego vuelve a abrirla:
open /path/to/OpenClaw.app
```

La app ahora se conectará al gateway remoto a través del túnel SSH.

---

## Iniciar automáticamente el túnel al iniciar sesión

Para que el túnel SSH se inicie automáticamente cuando inicies sesión, crea un Launch Agent.

### Crear el archivo PLIST

Guarda esto como `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

### Cargar el Launch Agent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

El túnel ahora:

- Se iniciará automáticamente cuando inicies sesión
- Se reiniciará si falla
- Seguirá ejecutándose en segundo plano

Nota heredada: elimina cualquier LaunchAgent residual `com.openclaw.ssh-tunnel` si existe.

---

## Solución de problemas

**Comprobar si el túnel está en ejecución:**

```bash
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789
```

**Reiniciar el túnel:**

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel
```

**Detener el túnel:**

```bash
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

---

## Cómo funciona

| Componente                           | Qué hace                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | Reenvía el puerto local 18789 al puerto remoto 18789         |
| `ssh -N`                             | SSH sin ejecutar comandos remotos (solo reenvío de puertos)  |
| `KeepAlive`                          | Reinicia automáticamente el túnel si falla                   |
| `RunAtLoad`                          | Inicia el túnel cuando se carga el agente                    |

OpenClaw.app se conecta a `ws://127.0.0.1:18789` en tu máquina cliente. El túnel SSH reenvía esa conexión al puerto 18789 en la máquina remota donde se está ejecutando el Gateway.
