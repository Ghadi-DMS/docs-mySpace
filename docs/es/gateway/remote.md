---
read_when:
    - Ejecutar o solucionar problemas de configuraciones remotas del gateway
summary: Acceso remoto mediante túneles SSH (Gateway WS) y tailnets
title: Acceso remoto
x-i18n:
    generated_at: "2026-04-05T12:43:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8596fa2a7fd44117dfe92b70c9d8f28c0e16d7987adf0d0769a9eff71d5bc081
    source_path: gateway/remote.md
    workflow: 15
---

# Acceso remoto (SSH, túneles y tailnets)

Este repositorio admite “remoto mediante SSH” manteniendo un único Gateway (el principal) en ejecución en un host dedicado (escritorio/servidor) y conectando los clientes a él.

- Para **operadores (tú / la app de macOS)**: el túnel SSH es el respaldo universal.
- Para **nodos (iOS/Android y futuros dispositivos)**: conéctate al **WebSocket** del Gateway (LAN/tailnet o túnel SSH según sea necesario).

## La idea central

- El WebSocket del Gateway se enlaza a **loopback** en tu puerto configurado (por defecto `18789`).
- Para uso remoto, reenvías ese puerto loopback mediante SSH (o usas una tailnet/VPN y dependes menos del túnel).

## Configuraciones comunes de VPN/tailnet (donde vive el agente)

Piensa en el **host del Gateway** como “donde vive el agente”. Es propietario de las sesiones, perfiles de autenticación, canales y estado.
Tu laptop/escritorio (y los nodos) se conectan a ese host.

### 1) Gateway siempre activo en tu tailnet (VPS o servidor doméstico)

Ejecuta el Gateway en un host persistente y accede a él mediante **Tailscale** o SSH.

- **La mejor experiencia de usuario:** mantén `gateway.bind: "loopback"` y usa **Tailscale Serve** para la UI de Control.
- **Respaldo:** mantén loopback + túnel SSH desde cualquier máquina que necesite acceso.
- **Ejemplos:** [exe.dev](/install/exe-dev) (VM sencilla) o [Hetzner](/install/hetzner) (VPS de producción).

Esto es ideal cuando tu laptop entra en suspensión con frecuencia, pero quieres que el agente esté siempre activo.

### 2) El Gateway se ejecuta en el escritorio de casa; la laptop solo controla de forma remota

La laptop **no** ejecuta el agente. Se conecta de forma remota:

- Usa el modo **Remote over SSH** de la app de macOS (Settings → General → “OpenClaw runs”).
- La app abre y administra el túnel, de modo que WebChat + comprobaciones de estado “simplemente funcionan”.

Runbook: [acceso remoto en macOS](/platforms/mac/remote).

### 3) El Gateway se ejecuta en la laptop; acceso remoto desde otras máquinas

Mantén el Gateway local, pero expónlo de forma segura:

- Túnel SSH hacia la laptop desde otras máquinas, o
- Expón la UI de Control con Tailscale Serve y mantén el Gateway solo en loopback.

Guía: [Tailscale](/gateway/tailscale) y [resumen web](/web).

## Flujo de comandos (qué se ejecuta dónde)

Un único servicio gateway es propietario del estado + canales. Los nodos son periféricos.

Ejemplo de flujo (Telegram → nodo):

- Llega un mensaje de Telegram al **Gateway**.
- El Gateway ejecuta el **agente** y decide si debe llamar a una herramienta del nodo.
- El Gateway llama al **nodo** a través del WebSocket del Gateway (`node.*` RPC).
- El nodo devuelve el resultado; el Gateway responde de vuelta a Telegram.

Notas:

- **Los nodos no ejecutan el servicio gateway.** Solo debe ejecutarse un gateway por host, salvo que intencionalmente uses perfiles aislados (consulta [Varios gateways](/gateway/multiple-gateways)).
- El “modo nodo” de la app de macOS es simplemente un cliente de nodo sobre el WebSocket del Gateway.

## Túnel SSH (CLI + herramientas)

Crea un túnel local hacia el Gateway WS remoto:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

Con el túnel activo:

- `openclaw health` y `openclaw status --deep` ahora llegan al gateway remoto mediante `ws://127.0.0.1:18789`.
- `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe` y `openclaw gateway call` también pueden apuntar a la URL reenviada mediante `--url` cuando sea necesario.

Nota: sustituye `18789` por tu `gateway.port` configurado (o `--port`/`OPENCLAW_GATEWAY_PORT`).
Nota: cuando pasas `--url`, la CLI no recurre a credenciales de configuración o entorno.
Incluye `--token` o `--password` explícitamente. La ausencia de credenciales explícitas es un error.

## Valores predeterminados remotos de la CLI

Puedes conservar un destino remoto para que los comandos de la CLI lo usen de forma predeterminada:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

Cuando el gateway es solo loopback, mantén la URL en `ws://127.0.0.1:18789` y abre primero el túnel SSH.

## Precedencia de credenciales

La resolución de credenciales del Gateway sigue un único contrato compartido entre rutas de call/probe/status y la monitorización de aprobación de exec en Discord. El host del nodo usa el mismo contrato base con una excepción en modo local (intencionalmente ignora `gateway.remote.*`):

- Las credenciales explícitas (`--token`, `--password` o la herramienta `gatewayToken`) siempre tienen prioridad en las rutas call que aceptan autenticación explícita.
- Seguridad de anulación de URL:
  - Las anulaciones de URL de la CLI (`--url`) nunca reutilizan credenciales implícitas de config/env.
  - Las anulaciones de URL por entorno (`OPENCLAW_GATEWAY_URL`) pueden usar solo credenciales del entorno (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`).
- Valores predeterminados en modo local:
  - token: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (el respaldo remoto solo se aplica cuando no está definido el valor local del token de autenticación)
  - password: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (el respaldo remoto solo se aplica cuando no está definido el valor local de la contraseña de autenticación)
- Valores predeterminados en modo remoto:
  - token: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - password: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Excepción del host del nodo en modo local: se ignoran `gateway.remote.token` / `gateway.remote.password`.
- Las comprobaciones de token de probe/status remotas son estrictas de forma predeterminada: usan solo `gateway.remote.token` (sin respaldo al token local) cuando apuntan al modo remoto.
- Las anulaciones de entorno del Gateway usan solo `OPENCLAW_GATEWAY_*`.

## IU de chat mediante SSH

WebChat ya no usa un puerto HTTP independiente. La IU de chat SwiftUI se conecta directamente al WebSocket del Gateway.

- Reenvía `18789` mediante SSH (ver arriba), luego conecta los clientes a `ws://127.0.0.1:18789`.
- En macOS, prefiere el modo “Remote over SSH” de la app, que administra el túnel automáticamente.

## App de macOS "Remote over SSH"

La app de barra de menús de macOS puede gestionar esta misma configuración de extremo a extremo (comprobaciones de estado remotas, WebChat y reenvío de Voice Wake).

Runbook: [acceso remoto en macOS](/platforms/mac/remote).

## Reglas de seguridad (remoto/VPN)

Versión corta: **mantén el Gateway solo en loopback** salvo que estés seguro de que necesitas un bind.

- **Loopback + SSH/Tailscale Serve** es la opción predeterminada más segura (sin exposición pública).
- `ws://` en texto plano es solo loopback de forma predeterminada. Para redes privadas de confianza,
  establece `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` en el proceso cliente como medida de emergencia.
- Los **binds fuera de loopback** (`lan`/`tailnet`/`custom`, o `auto` cuando loopback no está disponible) deben usar autenticación del gateway: token, contraseña o un proxy inverso con conocimiento de identidad con `gateway.auth.mode: "trusted-proxy"`.
- `gateway.remote.token` / `.password` son orígenes de credenciales del cliente. No configuran por sí mismos la autenticación del servidor.
- Las rutas call locales pueden usar `gateway.remote.*` como respaldo solo cuando `gateway.auth.*` no está definido.
- Si `gateway.auth.token` / `gateway.auth.password` están configurados explícitamente mediante SecretRef y no se pueden resolver, la resolución falla de forma cerrada (sin enmascaramiento por respaldo remoto).
- `gateway.remote.tlsFingerprint` fija el certificado TLS remoto cuando se usa `wss://`.
- **Tailscale Serve** puede autenticar el tráfico de la UI de Control/WebSocket mediante encabezados de identidad cuando `gateway.auth.allowTailscale: true`; los endpoints de la API HTTP no usan esa autenticación de encabezado de Tailscale y en su lugar siguen el modo normal de autenticación HTTP del gateway. Este flujo sin token asume que el host del gateway es de confianza. Establécelo en `false` si quieres autenticación con secreto compartido en todas partes.
- La autenticación **trusted-proxy** es solo para configuraciones de proxy con conocimiento de identidad fuera de loopback.
  Los proxies inversos en loopback en el mismo host no cumplen `gateway.auth.mode: "trusted-proxy"`.
- Trata el control del navegador como acceso de operador: solo tailnet + emparejamiento deliberado de nodos.

Análisis en profundidad: [Seguridad](/gateway/security).

### macOS: túnel SSH persistente mediante LaunchAgent

Para clientes macOS que se conectan a un gateway remoto, la configuración persistente más sencilla usa una entrada `LocalForward` en la configuración de SSH más un LaunchAgent para mantener el túnel activo entre reinicios y fallos.

#### Paso 1: añadir configuración SSH

Edita `~/.ssh/config`:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

Sustituye `<REMOTE_IP>` y `<REMOTE_USER>` por tus valores.

#### Paso 2: copiar la clave SSH (una sola vez)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### Paso 3: configurar el token del gateway

Guarda el token en la configuración para que persista entre reinicios:

```bash
openclaw config set gateway.remote.token "<your-token>"
```

#### Paso 4: crear el LaunchAgent

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

#### Paso 5: cargar el LaunchAgent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

El túnel se iniciará automáticamente al iniciar sesión, se reiniciará en caso de fallo y mantendrá activo el puerto reenviado.

Nota: si tienes un LaunchAgent `com.openclaw.ssh-tunnel` sobrante de una configuración anterior, descárgalo y elimínalo.

#### Solución de problemas

Comprueba si el túnel está en ejecución:

```bash
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789
```

Reiniciar el túnel:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel
```

Detener el túnel:

```bash
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| Entrada de configuración                | Qué hace                                                     |
| --------------------------------------- | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789`    | Reenvía el puerto local 18789 al puerto remoto 18789         |
| `ssh -N`                                | SSH sin ejecutar comandos remotos (solo reenvío de puertos) |
| `KeepAlive`                             | Reinicia automáticamente el túnel si falla                   |
| `RunAtLoad`                             | Inicia el túnel cuando el LaunchAgent se carga al iniciar sesión |
