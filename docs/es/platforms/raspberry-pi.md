---
read_when:
    - Configurar OpenClaw en una Raspberry Pi
    - Ejecutar OpenClaw en dispositivos ARM
    - Construir una IA personal barata y siempre activa
summary: OpenClaw en Raspberry Pi (configuración autoalojada económica)
title: Raspberry Pi (plataforma)
x-i18n:
    generated_at: "2026-04-05T12:49:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 07f34e91899b7e0a31d9b944f3cb0cfdd4ecdeba58b619ae554379abdbf37eaf
    source_path: platforms/raspberry-pi.md
    workflow: 15
---

# OpenClaw en Raspberry Pi

## Objetivo

Ejecutar una Gateway persistente y siempre activa de OpenClaw en una Raspberry Pi por **~$35-80** de coste único (sin cuotas mensuales).

Perfecto para:

- Asistente personal de IA 24/7
- Centro de automatización del hogar
- Bot de Telegram/WhatsApp de bajo consumo y siempre disponible

## Requisitos de hardware

| Modelo de Pi     | RAM     | ¿Funciona? | Notas                              |
| ---------------- | ------- | ---------- | ---------------------------------- |
| **Pi 5**         | 4GB/8GB | ✅ Mejor   | Más rápida, recomendada            |
| **Pi 4**         | 4GB     | ✅ Bien    | Punto óptimo para la mayoría       |
| **Pi 4**         | 2GB     | ✅ OK      | Funciona, añade swap               |
| **Pi 4**         | 1GB     | ⚠️ Justo   | Posible con swap, config mínima    |
| **Pi 3B+**       | 1GB     | ⚠️ Lento   | Funciona pero va lento             |
| **Pi Zero 2 W**  | 512MB   | ❌         | No recomendado                     |

**Especificaciones mínimas:** 1GB RAM, 1 núcleo, 500MB de disco  
**Recomendado:** 2GB+ RAM, SO de 64 bits, tarjeta SD de 16GB+ (o SSD USB)

## Qué necesitas

- Raspberry Pi 4 o 5 (se recomiendan 2GB+)
- Tarjeta microSD (16GB+) o SSD USB (mejor rendimiento)
- Fuente de alimentación (se recomienda la PSU oficial de Pi)
- Conexión de red (Ethernet o WiFi)
- ~30 minutos

## 1) Grabar el SO

Usa **Raspberry Pi OS Lite (64-bit)**: no hace falta entorno de escritorio para un servidor headless.

1. Descarga [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Elige SO: **Raspberry Pi OS Lite (64-bit)**
3. Haz clic en el icono de engranaje (⚙️) para preconfigurar:
   - Establece el hostname: `gateway-host`
   - Habilita SSH
   - Establece usuario/contraseña
   - Configura WiFi (si no usas Ethernet)
4. Graba la imagen en tu tarjeta SD / unidad USB
5. Inserta y arranca la Pi

## 2) Conectarte por SSH

```bash
ssh user@gateway-host
# o usa la dirección IP
ssh user@192.168.x.x
```

## 3) Configuración del sistema

```bash
# Actualizar el sistema
sudo apt update && sudo apt upgrade -y

# Instalar paquetes esenciales
sudo apt install -y git curl build-essential

# Establecer zona horaria (importante para cron/recordatorios)
sudo timedatectl set-timezone America/Chicago  # Cámbialo por tu zona horaria
```

## 4) Instalar Node.js 24 (ARM64)

```bash
# Instalar Node.js mediante NodeSource
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

# Verificar
node --version  # Debe mostrar v24.x.x
npm --version
```

## 5) Añadir swap (importante para 2GB o menos)

El swap evita cierres por falta de memoria:

```bash
# Crear un archivo swap de 2GB
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Hacerlo permanente
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Optimizar para poca RAM (reducir swappiness)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 6) Instalar OpenClaw

### Opción A: instalación estándar (recomendada)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

### Opción B: instalación modificable (para experimentar)

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
npm install
npm run build
npm link
```

La instalación modificable te da acceso directo a los registros y al código, lo que resulta útil para depurar problemas específicos de ARM.

## 7) Ejecutar onboarding

```bash
openclaw onboard --install-daemon
```

Sigue el asistente:

1. **Modo Gateway:** Local
2. **Auth:** se recomiendan API keys (OAuth puede ser delicado en una Pi headless)
3. **Canales:** Telegram es lo más fácil para empezar
4. **Daemon:** Sí (systemd)

## 8) Verificar la instalación

```bash
# Comprobar el estado
openclaw status

# Comprobar el servicio (instalación estándar = unidad de usuario systemd)
systemctl --user status openclaw-gateway.service

# Ver registros
journalctl --user -u openclaw-gateway.service -f
```

## 9) Acceder al dashboard de OpenClaw

Sustituye `user@gateway-host` por tu nombre de usuario y hostname o IP de la Pi.

En tu ordenador, pídele a la Pi que imprima una URL nueva del dashboard:

```bash
ssh user@gateway-host 'openclaw dashboard --no-open'
```

El comando imprime `Dashboard URL:`. Según cómo esté configurado
`gateway.auth.token`, la URL puede ser un enlace simple `http://127.0.0.1:18789/` o uno
que incluya `#token=...`.

En otro terminal de tu ordenador, crea el túnel SSH:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Luego abre la URL del dashboard impresa en tu navegador local.

Si la interfaz solicita autenticación por secreto compartido, pega el token o la contraseña
configurados en la configuración de Control UI. Para autenticación por token, usa `gateway.auth.token` (o
`OPENCLAW_GATEWAY_TOKEN`).

Para acceso remoto siempre activo, consulta [Tailscale](/gateway/tailscale).

---

## Optimizaciones de rendimiento

### Usar un SSD USB (gran mejora)

Las tarjetas SD son lentas y se desgastan. Un SSD USB mejora enormemente el rendimiento:

```bash
# Comprobar si está arrancando desde USB
lsblk
```

Consulta la [guía de arranque USB de Pi](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot) para la configuración.

### Acelerar el inicio de la CLI (caché de compilación de módulos)

En hosts Pi de menor potencia, habilita la caché de compilación de módulos de Node para que las ejecuciones repetidas de la CLI sean más rápidas:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

Notas:

- `NODE_COMPILE_CACHE` acelera las ejecuciones posteriores (`status`, `health`, `--help`).
- `/var/tmp` sobrevive a los reinicios mejor que `/tmp`.
- `OPENCLAW_NO_RESPAWN=1` evita el coste adicional de arranque por el autorrespawn de la CLI.
- La primera ejecución calienta la caché; las posteriores son las que más se benefician.

### Ajuste de arranque de systemd (opcional)

Si esta Pi ejecuta principalmente OpenClaw, añade un drop-in del servicio para reducir
las oscilaciones en los reinicios y mantener estable el entorno de arranque:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

Luego aplícalo:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway.service
```

Si es posible, mantén el estado/la caché de OpenClaw en almacenamiento respaldado por SSD para evitar
cuellos de botella de E/S aleatoria de tarjetas SD durante arranques en frío.

Si esta es una Pi headless, habilita lingering una vez para que el servicio de usuario sobreviva
al cierre de sesión:

```bash
sudo loginctl enable-linger "$(whoami)"
```

Cómo las políticas `Restart=` ayudan con la recuperación automatizada:
[systemd can automate service recovery](https://www.redhat.com/en/blog/systemd-automate-recovery).

### Reducir el uso de memoria

```bash
# Desactivar la asignación de memoria GPU (headless)
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt

# Desactivar Bluetooth si no se necesita
sudo systemctl disable bluetooth
```

### Supervisar recursos

```bash
# Comprobar memoria
free -h

# Comprobar la temperatura de la CPU
vcgencmd measure_temp

# Supervisión en vivo
htop
```

---

## Notas específicas de ARM

### Compatibilidad de binarios

La mayoría de las funciones de OpenClaw funcionan en ARM64, pero algunos binarios externos pueden necesitar compilaciones ARM:

| Herramienta        | Estado ARM64 | Notas                               |
| ------------------ | ------------ | ----------------------------------- |
| Node.js            | ✅           | Funciona muy bien                   |
| WhatsApp (Baileys) | ✅           | JS puro, sin problemas              |
| Telegram           | ✅           | JS puro, sin problemas              |
| gog (Gmail CLI)    | ⚠️           | Comprueba si existe una versión ARM |
| Chromium (browser) | ✅           | `sudo apt install chromium-browser` |

Si falla una Skill, comprueba si su binario tiene compilación ARM. Muchas herramientas Go/Rust la tienen; algunas no.

### 32 bits frente a 64 bits

**Usa siempre un SO de 64 bits.** Node.js y muchas herramientas modernas lo requieren. Compruébalo con:

```bash
uname -m
# Debe mostrar: aarch64 (64-bit) y no armv7l (32-bit)
```

---

## Configuración de modelo recomendada

Como la Pi es solo la Gateway (los modelos se ejecutan en la nube), usa modelos basados en API:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openai/gpt-5.4-mini"]
      }
    }
  }
}
```

**No intentes ejecutar LLM locales en una Pi**: incluso los modelos pequeños son demasiado lentos. Deja que Claude/GPT hagan el trabajo pesado.

---

## Inicio automático al arrancar

Onboarding lo configura, pero para verificarlo:

```bash
# Comprobar que el servicio está habilitado
systemctl --user is-enabled openclaw-gateway.service

# Habilitarlo si no lo está
systemctl --user enable openclaw-gateway.service

# Iniciar al arrancar
systemctl --user start openclaw-gateway.service
```

---

## Solución de problemas

### Falta de memoria (OOM)

```bash
# Comprobar memoria
free -h

# Añadir más swap (consulta el paso 5)
# O reducir los servicios que se ejecutan en la Pi
```

### Rendimiento lento

- Usa un SSD USB en lugar de una tarjeta SD
- Desactiva servicios no utilizados: `sudo systemctl disable cups bluetooth avahi-daemon`
- Comprueba el throttling de la CPU: `vcgencmd get_throttled` (debe devolver `0x0`)

### El servicio no arranca

```bash
# Comprobar registros
journalctl --user -u openclaw-gateway.service --no-pager -n 100

# Solución común: reconstruir
cd ~/openclaw  # si usas instalación modificable
npm run build
systemctl --user restart openclaw-gateway.service
```

### Problemas con binarios ARM

Si una Skill falla con "exec format error":

1. Comprueba si el binario tiene una compilación ARM64
2. Intenta compilar desde el código fuente
3. O usa un contenedor Docker con compatibilidad ARM

### Cortes de WiFi

Para Pi headless con WiFi:

```bash
# Desactivar la gestión de energía WiFi
sudo iwconfig wlan0 power off

# Hacerlo permanente
echo 'wireless-power off' | sudo tee -a /etc/network/interfaces
```

---

## Comparación de costes

| Configuración     | Coste único | Coste mensual | Notas                     |
| ----------------- | ----------- | ------------- | ------------------------- |
| **Pi 4 (2GB)**    | ~$45        | $0            | + electricidad (~$5/año)  |
| **Pi 4 (4GB)**    | ~$55        | $0            | Recomendado               |
| **Pi 5 (4GB)**    | ~$60        | $0            | Mejor rendimiento         |
| **Pi 5 (8GB)**    | ~$80        | $0            | Excesivo pero preparado para el futuro |
| DigitalOcean      | $0          | $6/mes        | $72/año                   |
| Hetzner           | $0          | €3.79/mes     | ~$50/año                  |

**Punto de equilibrio:** una Pi se amortiza en ~6-12 meses frente a un VPS en la nube.

---

## Ver también

- [Guía de Linux](/platforms/linux) — configuración general en Linux
- [Guía de DigitalOcean](/platforms/digitalocean) — alternativa en la nube
- [Guía de Hetzner](/install/hetzner) — configuración Docker
- [Tailscale](/gateway/tailscale) — acceso remoto
- [Nodes](/nodes) — empareja tu portátil/teléfono con la gateway de la Pi
