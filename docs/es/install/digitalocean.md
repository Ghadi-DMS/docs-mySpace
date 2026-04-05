---
read_when:
    - Configurar OpenClaw en DigitalOcean
    - Buscar un VPS de pago sencillo para OpenClaw
summary: Aloja OpenClaw en un Droplet de DigitalOcean
title: DigitalOcean
x-i18n:
    generated_at: "2026-04-05T12:44:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4b161db8ec643d8313938a2453ce6242fc1ee8ea1fd2069916276f1aadeb71f1
    source_path: install/digitalocean.md
    workflow: 15
---

# DigitalOcean

Ejecuta un Gateway persistente de OpenClaw en un Droplet de DigitalOcean.

## Requisitos previos

- Cuenta de DigitalOcean ([registro](https://cloud.digitalocean.com/registrations/new))
- Par de claves SSH (o disposición para usar autenticación por contraseña)
- Unos 20 minutos

## Configuración

<Steps>
  <Step title="Crear un Droplet">
    <Warning>
    Usa una imagen base limpia (Ubuntu 24.04 LTS). Evita imágenes de Marketplace de terceros con 1 clic a menos que hayas revisado sus scripts de inicio y la configuración predeterminada del firewall.
    </Warning>

    1. Inicia sesión en [DigitalOcean](https://cloud.digitalocean.com/).
    2. Haz clic en **Create > Droplets**.
    3. Elige:
       - **Region:** la más cercana a ti
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic, Regular, 1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** clave SSH (recomendado) o contraseña
    4. Haz clic en **Create Droplet** y anota la dirección IP.

  </Step>

  <Step title="Conectarse e instalar">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # Instalar Node.js 24
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # Instalar OpenClaw
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw --version
    ```

  </Step>

  <Step title="Ejecutar onboarding">
    ```bash
    openclaw onboard --install-daemon
    ```

    El asistente te guía por la autenticación del modelo, la configuración del canal, la generación del token del gateway y la instalación del daemon (systemd).

  </Step>

  <Step title="Añadir swap (recomendado para Droplets de 1 GB)">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="Verificar el gateway">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Acceder a la UI de Control">
    El gateway se enlaza a loopback de forma predeterminada. Elige una de estas opciones.

    **Opción A: túnel SSH (la más sencilla)**

    ```bash
    # Desde tu máquina local
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    Luego abre `http://localhost:18789`.

    **Opción B: Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    Luego abre `https://<magicdns>/` desde cualquier dispositivo de tu tailnet.

    **Opción C: enlace a tailnet (sin Serve)**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    Luego abre `http://<tailscale-ip>:18789` (se requiere token).

  </Step>
</Steps>

## Solución de problemas

**El gateway no se inicia** -- Ejecuta `openclaw doctor --non-interactive` y revisa los registros con `journalctl --user -u openclaw-gateway.service -n 50`.

**El puerto ya está en uso** -- Ejecuta `lsof -i :18789` para encontrar el proceso y luego deténlo.

**Sin memoria** -- Verifica que swap esté activa con `free -h`. Si sigues teniendo OOM, usa modelos basados en API (Claude, GPT) en lugar de modelos locales, o actualiza a un Droplet de 2 GB.

## Siguientes pasos

- [Channels](/channels) -- conecta Telegram, WhatsApp, Discord y más
- [Configuración del gateway](/gateway/configuration) -- todas las opciones de configuración
- [Updating](/install/updating) -- mantén OpenClaw actualizado
