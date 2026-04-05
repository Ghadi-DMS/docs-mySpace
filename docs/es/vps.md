---
read_when:
    - Quieres ejecutar el Gateway en un servidor Linux o VPS en la nube
    - Necesitas un mapa rápido de las guías de alojamiento
    - Quieres un ajuste genérico de servidores Linux para OpenClaw
sidebarTitle: Linux Server
summary: Ejecuta OpenClaw en un servidor Linux o VPS en la nube — selector de proveedor, arquitectura y ajuste
title: Servidor Linux
x-i18n:
    generated_at: "2026-04-05T12:57:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7f2f26bbc116841a29055850ed5f491231554b90539bcbf91a6b519875d494fb
    source_path: vps.md
    workflow: 15
---

# Servidor Linux

Ejecuta el Gateway de OpenClaw en cualquier servidor Linux o VPS en la nube. Esta página te ayuda a elegir un proveedor, explica cómo funcionan los despliegues en la nube y cubre el ajuste genérico de Linux que se aplica en todas partes.

## Elige un proveedor

<CardGroup cols={2}>
  <Card title="Railway" href="/es/install/railway">Configuración en navegador con un clic</Card>
  <Card title="Northflank" href="/es/install/northflank">Configuración en navegador con un clic</Card>
  <Card title="DigitalOcean" href="/es/install/digitalocean">VPS de pago sencilla</Card>
  <Card title="Oracle Cloud" href="/es/install/oracle">Nivel ARM Always Free</Card>
  <Card title="Fly.io" href="/es/install/fly">Fly Machines</Card>
  <Card title="Hetzner" href="/es/install/hetzner">Docker en VPS de Hetzner</Card>
  <Card title="GCP" href="/es/install/gcp">Compute Engine</Card>
  <Card title="Azure" href="/es/install/azure">VM Linux</Card>
  <Card title="exe.dev" href="/es/install/exe-dev">VM con proxy HTTPS</Card>
  <Card title="Raspberry Pi" href="/es/install/raspberry-pi">ARM autoalojado</Card>
</CardGroup>

**AWS (EC2 / Lightsail / nivel gratuito)** también funciona bien.
Hay disponible un video de la comunidad con un recorrido completo en
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
(recurso de la comunidad -- puede dejar de estar disponible).

## Cómo funcionan las configuraciones en la nube

- El **Gateway se ejecuta en el VPS** y posee el estado + el espacio de trabajo.
- Te conectas desde tu portátil o teléfono mediante la **Control UI** o **Tailscale/SSH**.
- Trata el VPS como la fuente de verdad y **haz copias de seguridad** del estado + el espacio de trabajo con regularidad.
- Valor predeterminado seguro: mantén el Gateway en loopback y accede a él mediante túnel SSH o Tailscale Serve.
  Si enlazas a `lan` o `tailnet`, exige `gateway.auth.token` o `gateway.auth.password`.

Páginas relacionadas: [Acceso remoto al Gateway](/es/gateway/remote), [Hub de plataformas](/es/platforms).

## Agente compartido de empresa en un VPS

Ejecutar un solo agente para un equipo es una configuración válida cuando todos los usuarios están dentro del mismo límite de confianza y el agente es solo para uso empresarial.

- Mantenlo en un runtime dedicado (VPS/VM/contenedor + usuario/cuentas de SO dedicados).
- No inicies sesión en ese runtime con cuentas personales de Apple/Google ni con perfiles personales de navegador/gestor de contraseñas.
- Si los usuarios son adversariales entre sí, sepáralos por gateway/host/usuario de SO.

Detalles del modelo de seguridad: [Seguridad](/es/gateway/security).

## Uso de nodos con un VPS

Puedes mantener el Gateway en la nube y emparejar **nodos** en tus dispositivos locales
(Mac/iOS/Android/sin interfaz). Los nodos proporcionan capacidades locales de pantalla/cámara/canvas y `system.run` mientras el Gateway permanece en la nube.

Documentación: [Nodes](/es/nodes), [CLI de Nodes](/cli/nodes).

## Ajuste de inicio para VM pequeñas y hosts ARM

Si los comandos de CLI se sienten lentos en VM de baja potencia (o hosts ARM), habilita la caché de compilación de módulos de Node:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` mejora los tiempos de inicio de comandos repetidos.
- `OPENCLAW_NO_RESPAWN=1` evita la sobrecarga adicional de inicio de una ruta de autoreinicio.
- La primera ejecución de un comando calienta la caché; las siguientes son más rápidas.
- Para detalles específicos de Raspberry Pi, consulta [Raspberry Pi](/es/install/raspberry-pi).

### Lista de comprobación de ajuste de systemd (opcional)

Para hosts VM que usen `systemd`, considera:

- Añadir variables de entorno del servicio para una ruta de inicio estable:
  - `OPENCLAW_NO_RESPAWN=1`
  - `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- Mantener explícito el comportamiento de reinicio:
  - `Restart=always`
  - `RestartSec=2`
  - `TimeoutStartSec=90`
- Preferir discos respaldados por SSD para las rutas de estado/caché para reducir las penalizaciones de arranque en frío por E/S aleatoria.

Para la ruta estándar `openclaw onboard --install-daemon`, edita la unidad de usuario:

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

Si instalaste deliberadamente una unidad del sistema, edita
`openclaw-gateway.service` mediante `sudo systemctl edit openclaw-gateway.service`.

Cómo ayudan las políticas `Restart=` a la recuperación automatizada:
[systemd puede automatizar la recuperación de servicios](https://www.redhat.com/en/blog/systemd-automate-recovery).
