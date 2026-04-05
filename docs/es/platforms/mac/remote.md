---
read_when:
    - Configurar o depurar el control remoto en Mac
summary: Flujo de la app de macOS para controlar una gateway remota de OpenClaw por SSH
title: Control remoto
x-i18n:
    generated_at: "2026-04-05T12:48:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 96e46e603c2275d04596b5d1ae0fb6858bd1a102a727dc13924ffcd9808fdf7e
    source_path: platforms/mac/remote.md
    workflow: 15
---

# OpenClaw remoto (macOS ⇄ host remoto)

Este flujo permite que la app de macOS actúe como un control remoto completo para una gateway de OpenClaw que se ejecuta en otro host (escritorio/servidor). Es la función **Remote over SSH** (ejecución remota) de la app. Todas las funciones — comprobaciones de salud, reenvío de Voice Wake y Web Chat — reutilizan la misma configuración SSH remota desde _Settings → General_.

## Modos

- **Local (this Mac)**: Todo se ejecuta en el portátil. No hay SSH.
- **Remote over SSH (default)**: Los comandos de OpenClaw se ejecutan en el host remoto. La app de Mac abre una conexión SSH con `-o BatchMode` más tu identidad/clave elegida y un reenvío de puerto local.
- **Remote direct (ws/wss)**: No hay túnel SSH. La app de Mac se conecta directamente a la URL de la gateway (por ejemplo, mediante Tailscale Serve o un proxy inverso HTTPS público).

## Transportes remotos

El modo remoto admite dos transportes:

- **Túnel SSH** (predeterminado): usa `ssh -N -L ...` para reenviar el puerto de la gateway a localhost. La gateway verá la IP del nodo como `127.0.0.1` porque el túnel es loopback.
- **Direct (ws/wss)**: se conecta directamente a la URL de la gateway. La gateway ve la IP real del cliente.

## Requisitos previos en el host remoto

1. Instala Node + pnpm y compila/instala la CLI de OpenClaw (`pnpm install && pnpm build && pnpm link --global`).
2. Asegúrate de que `openclaw` esté en PATH para shells no interactivos (symlink a `/usr/local/bin` o `/opt/homebrew/bin` si es necesario).
3. Abre SSH con autenticación por clave. Recomendamos IPs de **Tailscale** para una accesibilidad estable fuera de la LAN.

## Configuración de la app de macOS

1. Abre _Settings → General_.
2. En **OpenClaw runs**, elige **Remote over SSH** y configura:
   - **Transport**: **SSH tunnel** o **Direct (ws/wss)**.
   - **SSH target**: `user@host` (opcional `:port`).
     - Si la gateway está en la misma LAN y anuncia Bonjour, elígela de la lista detectada para rellenar automáticamente este campo.
   - **Gateway URL** (solo Direct): `wss://gateway.example.ts.net` (o `ws://...` para local/LAN).
   - **Identity file** (avanzado): ruta a tu clave.
   - **Project root** (avanzado): ruta remota del checkout usada para los comandos.
   - **CLI path** (avanzado): ruta opcional a un entrypoint/binario `openclaw` ejecutable (se rellena automáticamente cuando se anuncia).
3. Pulsa **Test remote**. El éxito indica que `openclaw status --json` se ejecuta correctamente en remoto. Los fallos suelen significar problemas de PATH/CLI; la salida 127 significa que la CLI no se encuentra en remoto.
4. Las comprobaciones de salud y Web Chat se ejecutarán ahora automáticamente a través de este túnel SSH.

## Web Chat

- **Túnel SSH**: Web Chat se conecta a la gateway mediante el puerto de control WebSocket reenviado (predeterminado 18789).
- **Direct (ws/wss)**: Web Chat se conecta directamente a la URL configurada de la gateway.
- Ya no existe un servidor HTTP independiente de WebChat.

## Permisos

- El host remoto necesita las mismas aprobaciones TCC que en local (Automation, Accessibility, Screen Recording, Microphone, Speech Recognition, Notifications). Ejecuta el onboarding en esa máquina para concederlos una vez.
- Los nodos anuncian su estado de permisos mediante `node.list` / `node.describe` para que los agentes sepan qué está disponible.

## Notas de seguridad

- Prefiere enlaces loopback en el host remoto y conéctate mediante SSH o Tailscale.
- El túnel SSH usa comprobación estricta de clave de host; primero confía en la clave del host para que exista en `~/.ssh/known_hosts`.
- Si enlazas la Gateway a una interfaz fuera de loopback, exige autenticación válida de Gateway: token, contraseña o un proxy inverso con identidad reconocida mediante `gateway.auth.mode: "trusted-proxy"`.
- Consulta [Security](/gateway/security) y [Tailscale](/gateway/tailscale).

## Flujo de inicio de sesión de WhatsApp (remoto)

- Ejecuta `openclaw channels login --verbose` **en el host remoto**. Escanea el QR con WhatsApp en tu teléfono.
- Vuelve a ejecutar el inicio de sesión en ese host si la autenticación caduca. La comprobación de salud mostrará los problemas de enlace.

## Solución de problemas

- **exit 127 / not found**: `openclaw` no está en PATH para shells sin login. Añádelo a `/etc/paths`, al rc de tu shell, o crea un symlink en `/usr/local/bin`/`/opt/homebrew/bin`.
- **Health probe failed**: comprueba la accesibilidad SSH, PATH y que Baileys haya iniciado sesión (`openclaw status --json`).
- **Web Chat stuck**: confirma que la gateway se esté ejecutando en el host remoto y que el puerto reenviado coincida con el puerto WS de la gateway; la UI requiere una conexión WS saludable.
- **Node IP shows 127.0.0.1**: es lo esperado con el túnel SSH. Cambia **Transport** a **Direct (ws/wss)** si quieres que la gateway vea la IP real del cliente.
- **Voice Wake**: las frases de activación se reenvían automáticamente en modo remoto; no hace falta un reenviador separado.

## Sonidos de notificación

Elige sonidos por notificación desde scripts con `openclaw` y `node.invoke`, por ejemplo:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Remote gateway ready" --sound Glass
```

Ya no existe un selector global de “sonido predeterminado” en la app; quien llama elige un sonido (o ninguno) por solicitud.
