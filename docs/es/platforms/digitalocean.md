---
read_when:
    - Configurar OpenClaw en DigitalOcean
    - Buscas alojamiento VPS barato para OpenClaw
summary: OpenClaw en DigitalOcean (opción sencilla de VPS de pago)
title: DigitalOcean (plataforma)
x-i18n:
    generated_at: "2026-04-05T12:47:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6ee4ad84c421f87064534a4fb433df1f70304502921841ec618318ed862d4092
    source_path: platforms/digitalocean.md
    workflow: 15
---

# OpenClaw en DigitalOcean

## Objetivo

Ejecutar un OpenClaw Gateway persistente en DigitalOcean por **$6/mes** (o $4/mes con precio reservado).

Si quieres una opción de $0/mes y no te importa ARM + configuración específica del proveedor, consulta la [guía de Oracle Cloud](/platforms/oracle).

## Comparación de costos (2026)

| Proveedor    | Plan            | Especificaciones        | Precio/mes   | Notas                                 |
| ------------ | --------------- | ---------------------- | ------------ | ------------------------------------- |
| Oracle Cloud | Always Free ARM | hasta 4 OCPU, 24GB RAM | $0           | ARM, capacidad limitada / peculiaridades de registro |
| Hetzner      | CX22            | 2 vCPU, 4GB RAM        | €3.79 (~$4)  | La opción de pago más barata          |
| DigitalOcean | Basic           | 1 vCPU, 1GB RAM        | $6           | IU sencilla, buena documentación      |
| Vultr        | Cloud Compute   | 1 vCPU, 1GB RAM        | $6           | Muchas ubicaciones                    |
| Linode       | Nanode          | 1 vCPU, 1GB RAM        | $5           | Ahora forma parte de Akamai           |

**Elegir un proveedor:**

- DigitalOcean: experiencia de usuario más sencilla + configuración predecible (esta guía)
- Hetzner: buena relación precio/rendimiento (consulta la [guía de Hetzner](/install/hetzner))
- Oracle Cloud: puede costar $0/mes, pero es más delicado y solo ARM (consulta la [guía de Oracle](/platforms/oracle))

---

## Requisitos previos

- Cuenta de DigitalOcean ([registro con $200 de crédito gratis](https://m.do.co/c/signup))
- Par de claves SSH (o disposición a usar autenticación por contraseña)
- ~20 minutos

## 1) Crear un Droplet

<Warning>
Usa una imagen base limpia (Ubuntu 24.04 LTS). Evita imágenes de terceros de Marketplace de un clic salvo que hayas revisado sus scripts de inicio y los valores predeterminados del firewall.
</Warning>

1. Inicia sesión en [DigitalOcean](https://cloud.digitalocean.com/)
2. Haz clic en **Create → Droplets**
3. Elige:
   - **Region:** la más cercana a ti (o a tus personas usuarias)
   - **Image:** Ubuntu 24.04 LTS
   - **Size:** Basic → Regular → **$6/mes** (1 vCPU, 1GB RAM, 25GB SSD)
   - **Authentication:** clave SSH (recomendado) o contraseña
4. Haz clic en **Create Droplet**
5. Anota la dirección IP

## 2) Conectarse mediante SSH

```bash
ssh root@YOUR_DROPLET_IP
```

## 3) Instalar OpenClaw

```bash
# Update system
apt update && apt upgrade -y

# Install Node.js 24
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt install -y nodejs

# Install OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# Verify
openclaw --version
```

## 4) Ejecutar onboarding

```bash
openclaw onboard --install-daemon
```

El asistente te guiará a través de:

- Autenticación del modelo (claves API u OAuth)
- Configuración de canales (Telegram, WhatsApp, Discord, etc.)
- Token del gateway (generado automáticamente)
- Instalación del daemon (systemd)

## 5) Verificar el Gateway

```bash
# Check status
openclaw status

# Check service
systemctl --user status openclaw-gateway.service

# View logs
journalctl --user -u openclaw-gateway.service -f
```

## 6) Acceder al panel

El gateway se enlaza a loopback de forma predeterminada. Para acceder a la UI de Control:

**Opción A: túnel SSH (recomendado)**

```bash
# From your local machine
ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP

# Then open: http://localhost:18789
```

**Opción B: Tailscale Serve (HTTPS, solo loopback)**

```bash
# On the droplet
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Configure Gateway to use Tailscale Serve
openclaw config set gateway.tailscale.mode serve
openclaw gateway restart
```

Abrir: `https://<magicdns>/`

Notas:

- Serve mantiene el Gateway solo en loopback y autentica el tráfico de la UI de Control/WebSocket mediante encabezados de identidad de Tailscale (la autenticación sin token asume que el host del gateway es de confianza; las API HTTP no usan esos encabezados de Tailscale y en su lugar siguen el modo normal de autenticación HTTP del gateway).
- Para exigir credenciales explícitas con secreto compartido, establece `gateway.auth.allowTailscale: false` y usa `gateway.auth.mode: "token"` o `"password"`.

**Opción C: bind a tailnet (sin Serve)**

```bash
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

Abrir: `http://<tailscale-ip>:18789` (requiere token).

## 7) Conectar tus canales

### Telegram

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### WhatsApp

```bash
openclaw channels login whatsapp
# Scan QR code
```

Consulta [Canales](/channels) para otros proveedores.

---

## Optimizaciones para 1GB de RAM

El droplet de $6 solo tiene 1GB de RAM. Para que todo funcione sin problemas:

### Añadir swap (recomendado)

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Usar un modelo más ligero

Si estás encontrando errores OOM, considera:

- usar modelos basados en API (Claude, GPT) en lugar de modelos locales
- establecer `agents.defaults.model.primary` en un modelo más pequeño

### Supervisar memoria

```bash
free -h
htop
```

---

## Persistencia

Todo el estado vive en:

- `~/.openclaw/`: `openclaw.json`, `auth-profiles.json` por agente, estado de canal/proveedor y datos de sesión
- `~/.openclaw/workspace/`: espacio de trabajo (`SOUL.md`, memoria, etc.)

Esto sobrevive a reinicios. Haz copias de seguridad periódicamente:

```bash
openclaw backup create
```

---

## Alternativa gratuita de Oracle Cloud

Oracle Cloud ofrece instancias ARM **Always Free** que son significativamente más potentes que cualquier opción de pago aquí, por $0/mes.

| Lo que obtienes    | Especificaciones        |
| ------------------ | ---------------------- |
| **4 OCPU**         | ARM Ampere A1          |
| **24GB RAM**       | Más que suficiente     |
| **200GB storage**  | Volumen en bloque      |
| **Forever free**   | Sin cargos de tarjeta de crédito |

**Advertencias:**

- El registro puede ser delicado (vuelve a intentarlo si falla)
- Arquitectura ARM: la mayoría de cosas funcionan, pero algunos binarios necesitan compilaciones ARM

Para la guía completa de configuración, consulta [Oracle Cloud](/platforms/oracle). Para consejos de registro y solución de problemas del proceso de alta, consulta esta [guía de la comunidad](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd).

---

## Solución de problemas

### El Gateway no se inicia

```bash
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service --no-pager -n 50
```

### Puerto ya en uso

```bash
lsof -i :18789
kill <PID>
```

### Falta de memoria

```bash
# Check memory
free -h

# Add more swap
# Or upgrade to $12/mo droplet (2GB RAM)
```

---

## Ver también

- [guía de Hetzner](/install/hetzner) — más barato, más potente
- [instalación con Docker](/install/docker) — configuración en contenedor
- [Tailscale](/gateway/tailscale) — acceso remoto seguro
- [Configuration](/gateway/configuration) — referencia completa de configuración
