---
read_when:
    - Configurar OpenClaw en una Raspberry Pi
    - Ejecutar OpenClaw en dispositivos ARM
    - Crear una IA personal barata y siempre activa
summary: Aloja OpenClaw en una Raspberry Pi para autoalojamiento siempre activo
title: Raspberry Pi
x-i18n:
    generated_at: "2026-04-05T12:46:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 222ccbfb18a8dcec483adac6f5647dcb455c84edbad057e0ba2589a6da570b4c
    source_path: install/raspberry-pi.md
    workflow: 15
---

# Raspberry Pi

Ejecuta un Gateway persistente y siempre activo de OpenClaw en una Raspberry Pi. Como la Pi solo actúa como gateway (los modelos se ejecutan en la nube mediante API), incluso una Pi modesta maneja bien la carga de trabajo.

## Requisitos previos

- Raspberry Pi 4 o 5 con 2 GB+ de RAM (se recomiendan 4 GB)
- Tarjeta MicroSD (16 GB+) o SSD USB (mejor rendimiento)
- Fuente de alimentación oficial de Pi
- Conexión de red (Ethernet o WiFi)
- Raspberry Pi OS de 64 bits (obligatorio; no uses 32 bits)
- Unos 30 minutos

## Configuración

<Steps>
  <Step title="Grabar el sistema operativo">
    Usa **Raspberry Pi OS Lite (64-bit)**; no hace falta escritorio para un servidor sin interfaz.

    1. Descarga [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
    2. Elige el sistema operativo: **Raspberry Pi OS Lite (64-bit)**.
    3. En el cuadro de configuración, preconfigura:
       - Nombre del host: `gateway-host`
       - Habilitar SSH
       - Establecer nombre de usuario y contraseña
       - Configurar WiFi (si no usas Ethernet)
    4. Grábalo en tu tarjeta SD o unidad USB, insértala y arranca la Pi.

  </Step>

  <Step title="Conectarse por SSH">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="Actualizar el sistema">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # Establecer zona horaria (importante para cron y recordatorios)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="Instalar Node.js 24">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="Añadir swap (importante para 2 GB o menos)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Reducir swappiness para dispositivos con poca RAM
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="Instalar OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="Ejecutar onboarding">
    ```bash
    openclaw onboard --install-daemon
    ```

    Sigue el asistente. Para dispositivos sin interfaz se recomiendan claves de API en lugar de OAuth. Telegram es el canal más fácil para empezar.

  </Step>

  <Step title="Verificar">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Acceder a la UI de Control">
    En tu ordenador, obtén una URL del panel desde la Pi:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    Luego crea un túnel SSH en otra terminal:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    Abre la URL impresa en tu navegador local. Para acceso remoto siempre activo, consulta [integración con Tailscale](/gateway/tailscale).

  </Step>
</Steps>

## Consejos de rendimiento

**Usa un SSD USB** -- Las tarjetas SD son lentas y se desgastan. Un SSD USB mejora drásticamente el rendimiento. Consulta la [guía de arranque USB de Pi](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot).

**Habilita la caché de compilación de módulos** -- Acelera las invocaciones repetidas de la CLI en hosts Pi de menor potencia:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

**Reduce el uso de memoria** -- Para configuraciones sin interfaz, libera memoria de GPU y deshabilita servicios no usados:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

## Solución de problemas

**Sin memoria** -- Verifica que swap esté activa con `free -h`. Deshabilita servicios no usados (`sudo systemctl disable cups bluetooth avahi-daemon`). Usa solo modelos basados en API.

**Rendimiento lento** -- Usa un SSD USB en lugar de una tarjeta SD. Comprueba si hay limitación térmica de CPU con `vcgencmd get_throttled` (debería devolver `0x0`).

**El servicio no se inicia** -- Revisa los registros con `journalctl --user -u openclaw-gateway.service --no-pager -n 100` y ejecuta `openclaw doctor --non-interactive`. Si se trata de una Pi sin interfaz, comprueba también que lingering esté habilitado: `sudo loginctl enable-linger "$(whoami)"`.

**Problemas con binarios ARM** -- Si una Skill falla con "exec format error", comprueba si el binario tiene una compilación ARM64. Verifica la arquitectura con `uname -m` (debería mostrar `aarch64`).

**WiFi se desconecta** -- Deshabilita la gestión de energía de WiFi: `sudo iwconfig wlan0 power off`.

## Siguientes pasos

- [Channels](/channels) -- conecta Telegram, WhatsApp, Discord y más
- [Configuración del gateway](/gateway/configuration) -- todas las opciones de configuración
- [Updating](/install/updating) -- mantén OpenClaw actualizado
