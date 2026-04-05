---
read_when:
    - Configuración de OpenClaw en Oracle Cloud
    - Buscas alojamiento VPS de bajo costo para OpenClaw
    - Quieres OpenClaw 24/7 en un servidor pequeño
summary: OpenClaw en Oracle Cloud (ARM Always Free)
title: Oracle Cloud (plataforma)
x-i18n:
    generated_at: "2026-04-05T12:49:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3a42cdf2d18e964123894d382d2d8052c6b8dbb0b3c7dac914477c4a2a0a244f
    source_path: platforms/oracle.md
    workflow: 15
---

# OpenClaw en Oracle Cloud (OCI)

## Objetivo

Ejecutar un OpenClaw Gateway persistente en el nivel **Always Free** ARM de Oracle Cloud.

El nivel gratuito de Oracle puede encajar muy bien con OpenClaw (especialmente si ya tienes una cuenta de OCI), pero viene con compensaciones:

- Arquitectura ARM (la mayoría de cosas funcionan, pero algunos binarios pueden ser solo x86)
- La capacidad y el registro pueden ser algo delicados

## Comparación de costos (2026)

| Proveedor     | Plan             | Especificaciones        | Precio/mes | Notas                   |
| ------------- | ---------------- | ----------------------- | ---------- | ----------------------- |
| Oracle Cloud  | ARM Always Free  | hasta 4 OCPU, 24 GB RAM | $0         | ARM, capacidad limitada |
| Hetzner       | CX22             | 2 vCPU, 4 GB RAM        | ~ $4       | Opción de pago más barata |
| DigitalOcean  | Basic            | 1 vCPU, 1 GB RAM        | $6         | Interfaz sencilla, buena documentación |
| Vultr         | Cloud Compute    | 1 vCPU, 1 GB RAM        | $6         | Muchas ubicaciones      |
| Linode        | Nanode           | 1 vCPU, 1 GB RAM        | $5         | Ahora forma parte de Akamai |

---

## Requisitos previos

- Cuenta de Oracle Cloud ([registro](https://www.oracle.com/cloud/free/)) — consulta la [guía de registro de la comunidad](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) si tienes problemas
- Cuenta de Tailscale (gratis en [tailscale.com](https://tailscale.com))
- ~30 minutos

## 1) Crear una instancia OCI

1. Inicia sesión en [Oracle Cloud Console](https://cloud.oracle.com/)
2. Ve a **Compute → Instances → Create Instance**
3. Configura:
   - **Name:** `openclaw`
   - **Image:** Ubuntu 24.04 (aarch64)
   - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
   - **OCPUs:** 2 (o hasta 4)
   - **Memory:** 12 GB (o hasta 24 GB)
   - **Boot volume:** 50 GB (hasta 200 GB gratis)
   - **SSH key:** añade tu clave pública
4. Haz clic en **Create**
5. Anota la dirección IP pública

**Consejo:** si la creación de la instancia falla con "Out of capacity", prueba con otro dominio de disponibilidad o inténtalo más tarde. La capacidad del nivel gratuito es limitada.

## 2) Conectarse y actualizar

```bash
# Connect via public IP
ssh ubuntu@YOUR_PUBLIC_IP

# Update system
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential
```

**Nota:** `build-essential` es necesario para la compilación ARM de algunas dependencias.

## 3) Configurar usuario y hostname

```bash
# Set hostname
sudo hostnamectl set-hostname openclaw

# Set password for ubuntu user
sudo passwd ubuntu

# Enable lingering (keeps user services running after logout)
sudo loginctl enable-linger ubuntu
```

## 4) Instalar Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw
```

Esto habilita Tailscale SSH, de modo que puedes conectarte mediante `ssh openclaw` desde cualquier dispositivo de tu tailnet, sin necesidad de IP pública.

Verifica:

```bash
tailscale status
```

**A partir de ahora, conéctate mediante Tailscale:** `ssh ubuntu@openclaw` (o usa la IP de Tailscale).

## 5) Instalar OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
source ~/.bashrc
```

Cuando se te pregunte "How do you want to hatch your bot?", selecciona **"Do this later"**.

> Nota: si encuentras problemas de compilación nativa en ARM, empieza con paquetes del sistema (por ejemplo, `sudo apt install -y build-essential`) antes de recurrir a Homebrew.

## 6) Configurar el Gateway (loopback + autenticación por token) y habilitar Tailscale Serve

Usa autenticación por token como valor predeterminado. Es predecible y evita necesitar flags de interfaz de Control de “autenticación insegura”.

```bash
# Keep the Gateway private on the VM
openclaw config set gateway.bind loopback

# Require auth for the Gateway + Control UI
openclaw config set gateway.auth.mode token
openclaw doctor --generate-gateway-token

# Expose over Tailscale Serve (HTTPS + tailnet access)
openclaw config set gateway.tailscale.mode serve
openclaw config set gateway.trustedProxies '["127.0.0.1"]'

systemctl --user restart openclaw-gateway.service
```

`gateway.trustedProxies=["127.0.0.1"]` aquí es solo para el manejo de IP reenviada/cliente local del proxy local de Tailscale Serve. **No** es `gateway.auth.mode: "trusted-proxy"`. Las rutas del visor de diferencias mantienen comportamiento de fallo cerrado en esta configuración: las solicitudes raw al visor desde `127.0.0.1` sin encabezados de proxy reenviados pueden devolver `Diff not found`. Usa `mode=file` / `mode=both` para adjuntos, o habilita intencionalmente visores remotos y establece `plugins.entries.diffs.config.viewerBaseUrl` (o pasa un `baseUrl` de proxy) si necesitas enlaces compartibles del visor.

## 7) Verificar

```bash
# Check version
openclaw --version

# Check daemon status
systemctl --user status openclaw-gateway.service

# Check Tailscale Serve
tailscale serve status

# Test local response
curl http://localhost:18789
```

## 8) Endurecer la seguridad de la VCN

Ahora que todo funciona, endurece la VCN para bloquear todo el tráfico excepto Tailscale. La Virtual Cloud Network de OCI actúa como firewall en el borde de la red: el tráfico se bloquea antes de llegar a tu instancia.

1. Ve a **Networking → Virtual Cloud Networks** en la consola de OCI
2. Haz clic en tu VCN → **Security Lists** → Default Security List
3. **Elimina** todas las reglas de entrada excepto:
   - `0.0.0.0/0 UDP 41641` (Tailscale)
4. Mantén las reglas predeterminadas de salida (permitir todo el tráfico saliente)

Esto bloquea SSH en el puerto 22, HTTP, HTTPS y todo lo demás en el borde de la red. A partir de ahora, solo podrás conectarte mediante Tailscale.

---

## Acceder a la interfaz de Control

Desde cualquier dispositivo de tu red Tailscale:

```
https://openclaw.<tailnet-name>.ts.net/
```

Sustituye `<tailnet-name>` por el nombre de tu tailnet (visible en `tailscale status`).

No hace falta túnel SSH. Tailscale proporciona:

- Cifrado HTTPS (certificados automáticos)
- Autenticación mediante identidad de Tailscale
- Acceso desde cualquier dispositivo de tu tailnet (portátil, teléfono, etc.)

---

## Seguridad: VCN + Tailscale (línea base recomendada)

Con la VCN endurecida (solo UDP 41641 abierto) y el Gateway enlazado a loopback, obtienes una defensa en profundidad sólida: el tráfico público se bloquea en el borde de la red y el acceso administrativo ocurre a través de tu tailnet.

Esta configuración a menudo elimina la _necesidad_ de reglas adicionales de firewall en el host solo para detener ataques de fuerza bruta SSH desde Internet, pero aun así deberías mantener el SO actualizado, ejecutar `openclaw security audit` y verificar que no estás escuchando accidentalmente en interfaces públicas.

### Ya protegido

| Paso tradicional    | ¿Necesario? | Por qué                                                                       |
| ------------------- | ----------- | ----------------------------------------------------------------------------- |
| Firewall UFW        | No          | La VCN bloquea antes de que el tráfico llegue a la instancia                  |
| fail2ban            | No          | No hay fuerza bruta si el puerto 22 está bloqueado en la VCN                  |
| Endurecimiento de sshd | No       | Tailscale SSH no usa sshd                                                     |
| Deshabilitar inicio de sesión root | No | Tailscale usa identidad de Tailscale, no usuarios del sistema            |
| Autenticación SSH solo con clave | No | Tailscale autentica mediante tu tailnet                                    |
| Endurecimiento de IPv6 | Normalmente no | Depende de tu configuración de VCN/subred; verifica qué está realmente asignado/expuesto |

### Sigue siendo recomendable

- **Permisos de credenciales:** `chmod 700 ~/.openclaw`
- **Auditoría de seguridad:** `openclaw security audit`
- **Actualizaciones del sistema:** `sudo apt update && sudo apt upgrade` regularmente
- **Supervisar Tailscale:** revisa los dispositivos en la [consola de administración de Tailscale](https://login.tailscale.com/admin)

### Verificar postura de seguridad

```bash
# Confirm no public ports listening
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# Verify Tailscale SSH is active
tailscale status | grep -q 'offers: ssh' && echo "Tailscale SSH active"

# Optional: disable sshd entirely
sudo systemctl disable --now ssh
```

---

## Respaldo: túnel SSH

Si Tailscale Serve no funciona, usa un túnel SSH:

```bash
# From your local machine (via Tailscale)
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

Luego abre `http://localhost:18789`.

---

## Resolución de problemas

### La creación de la instancia falla ("Out of capacity")

Las instancias ARM del nivel gratuito son populares. Prueba:

- Otro dominio de disponibilidad
- Reintentar en horas valle (madrugada)
- Usar el filtro "Always Free" al seleccionar la forma

### Tailscale no se conecta

```bash
# Check status
sudo tailscale status

# Re-authenticate
sudo tailscale up --ssh --hostname=openclaw --reset
```

### El Gateway no inicia

```bash
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service -n 50
```

### No se puede alcanzar la interfaz de Control

```bash
# Verify Tailscale Serve is running
tailscale serve status

# Check gateway is listening
curl http://localhost:18789

# Restart if needed
systemctl --user restart openclaw-gateway.service
```

### Problemas con binarios ARM

Algunas herramientas pueden no tener compilaciones ARM. Comprueba:

```bash
uname -m  # Should show aarch64
```

La mayoría de paquetes npm funcionan bien. Para binarios, busca versiones `linux-arm64` o `aarch64`.

---

## Persistencia

Todo el estado vive en:

- `~/.openclaw/` — `openclaw.json`, `auth-profiles.json` por agente, estado de canales/proveedores y datos de sesión
- `~/.openclaw/workspace/` — espacio de trabajo (`SOUL.md`, memory, artefactos)

Haz copias de seguridad periódicamente:

```bash
openclaw backup create
```

---

## Ver también

- [Acceso remoto al Gateway](/gateway/remote) — otros patrones de acceso remoto
- [Integración con Tailscale](/gateway/tailscale) — documentación completa de Tailscale
- [Configuración del Gateway](/gateway/configuration) — todas las opciones de configuración
- [Guía de DigitalOcean](/platforms/digitalocean) — si quieres una opción de pago con registro más sencillo
- [Guía de Hetzner](/install/hetzner) — alternativa basada en Docker
