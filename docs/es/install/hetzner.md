---
read_when:
    - Quieres OpenClaw ejecutándose 24/7 en un VPS en la nube (no en tu portátil)
    - Quieres un Gateway siempre activo y apto para producción en tu propio VPS
    - Quieres control total sobre persistencia, binarios y comportamiento de reinicio
    - Estás ejecutando OpenClaw en Docker sobre Hetzner o un proveedor similar
summary: Ejecuta OpenClaw Gateway 24/7 en un VPS económico de Hetzner (Docker) con estado duradero y binarios integrados
title: Hetzner
x-i18n:
    generated_at: "2026-04-05T12:45:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: d859e4c0943040b022835f320708f879a11eadef70f2816cf0f2824eaaf165ef
    source_path: install/hetzner.md
    workflow: 15
---

# OpenClaw en Hetzner (Docker, guía de producción para VPS)

## Objetivo

Ejecutar un OpenClaw Gateway persistente en un VPS de Hetzner usando Docker, con estado duradero, binarios integrados y comportamiento de reinicio seguro.

Si quieres “OpenClaw 24/7 por ~$5”, esta es la configuración fiable más sencilla.
Los precios de Hetzner cambian; elige el VPS Debian/Ubuntu más pequeño y amplía si llegas a OOM.

Recordatorio del modelo de seguridad:

- Los agentes compartidos dentro de una empresa están bien cuando todos pertenecen al mismo límite de confianza y el runtime es solo para uso empresarial.
- Mantén una separación estricta: VPS/runtime dedicado + cuentas dedicadas; nada de perfiles personales de Apple/Google/navegador/gestor de contraseñas en ese host.
- Si los usuarios son adversarios entre sí, sepáralos por gateway/host/usuario del SO.

Consulta [Seguridad](/gateway/security) y [Alojamiento VPS](/vps).

## ¿Qué estamos haciendo? (en términos sencillos)

- Alquilar un pequeño servidor Linux (VPS de Hetzner)
- Instalar Docker (runtime aislado de la aplicación)
- Iniciar OpenClaw Gateway en Docker
- Persistir `~/.openclaw` + `~/.openclaw/workspace` en el host (sobrevive a reinicios/reconstrucciones)
- Acceder a la interfaz de Control desde tu portátil mediante un túnel SSH

Ese estado montado de `~/.openclaw` incluye `openclaw.json`, `agents/<agentId>/agent/auth-profiles.json`
por agente y `.env`.

Se puede acceder al Gateway mediante:

- Reenvío de puertos SSH desde tu portátil
- Exposición directa del puerto si gestionas el firewall y los tokens por tu cuenta

Esta guía asume Ubuntu o Debian en Hetzner.  
Si usas otro VPS Linux, ajusta los paquetes en consecuencia.
Para el flujo genérico de Docker, consulta [Docker](/install/docker).

---

## Ruta rápida (operadores con experiencia)

1. Aprovisionar el VPS de Hetzner
2. Instalar Docker
3. Clonar el repositorio de OpenClaw
4. Crear directorios persistentes en el host
5. Configurar `.env` y `docker-compose.yml`
6. Integrar los binarios necesarios en la imagen
7. `docker compose up -d`
8. Verificar la persistencia y el acceso al Gateway

---

## Qué necesitas

- VPS de Hetzner con acceso root
- Acceso SSH desde tu portátil
- Comodidad básica con SSH + copiar/pegar
- ~20 minutos
- Docker y Docker Compose
- Credenciales de autenticación del modelo
- Credenciales opcionales de proveedores
  - QR de WhatsApp
  - token del bot de Telegram
  - OAuth de Gmail

---

<Steps>
  <Step title="Aprovisionar el VPS">
    Crea un VPS Ubuntu o Debian en Hetzner.

    Conéctate como root:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    Esta guía asume que el VPS tiene estado.
    No lo trates como infraestructura desechable.

  </Step>

  <Step title="Instalar Docker (en el VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    Verifica:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="Clonar el repositorio de OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    Esta guía asume que vas a construir una imagen personalizada para garantizar la persistencia de binarios.

  </Step>

  <Step title="Crear directorios persistentes en el host">
    Los contenedores Docker son efímeros.
    Todo el estado de larga duración debe vivir en el host.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # Set ownership to the container user (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="Configurar variables de entorno">
    Crea `.env` en la raíz del repositorio.

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=change-me-now
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=change-me-now
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Genera secretos robustos:

    ```bash
    openssl rand -hex 32
    ```

    **No confirmes este archivo en el repositorio.**

    Este archivo `.env` es para variables de entorno del contenedor/runtime, como `OPENCLAW_GATEWAY_TOKEN`.
    La autenticación almacenada de proveedores mediante OAuth/API key vive en el
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` montado.

  </Step>

  <Step title="Configuración de Docker Compose">
    Crea o actualiza `docker-compose.yml`.

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # Recommended: keep the Gateway loopback-only on the VPS; access via SSH tunnel.
          # To expose it publicly, remove the `127.0.0.1:` prefix and firewall accordingly.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` es solo para comodidad durante el arranque inicial; no sustituye una configuración adecuada del gateway. Aun así, configura la autenticación (`gateway.auth.token` o contraseña) y usa ajustes de bind seguros para tu despliegue.

  </Step>

  <Step title="Pasos compartidos del runtime Docker en VM">
    Usa la guía de runtime compartida para el flujo común de host Docker:

    - [Integrar los binarios necesarios en la imagen](/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Compilar y lanzar](/install/docker-vm-runtime#build-and-launch)
    - [Qué persiste y dónde](/install/docker-vm-runtime#what-persists-where)
    - [Actualizaciones](/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Acceso específico de Hetzner">
    Después de los pasos compartidos de compilación y lanzamiento, crea un túnel desde tu portátil:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    Abre:

    `http://127.0.0.1:18789/`

    Pega el secreto compartido configurado. Esta guía usa el token del gateway de
    forma predeterminada; si cambiaste a autenticación por contraseña, usa esa contraseña en su lugar.

  </Step>
</Steps>

El mapa compartido de persistencia está en [Docker VM Runtime](/install/docker-vm-runtime#what-persists-where).

## Infraestructura como código (Terraform)

Para equipos que prefieren flujos de trabajo de infraestructura como código, una configuración de Terraform mantenida por la comunidad proporciona:

- Configuración modular de Terraform con gestión de estado remoto
- Aprovisionamiento automatizado mediante cloud-init
- Scripts de despliegue (bootstrap, deploy, backup/restore)
- Endurecimiento de seguridad (firewall, UFW, acceso solo por SSH)
- Configuración de túnel SSH para acceso al gateway

**Repositorios:**

- Infraestructura: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- Configuración Docker: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

Este enfoque complementa la configuración de Docker anterior con despliegues reproducibles, infraestructura versionada y recuperación ante desastres automatizada.

> **Nota:** Mantenido por la comunidad. Para problemas o contribuciones, consulta los enlaces de los repositorios anteriores.

## Siguientes pasos

- Configurar canales de mensajería: [Canales](/channels)
- Configurar el Gateway: [Configuración del Gateway](/gateway/configuration)
- Mantener OpenClaw actualizado: [Actualización](/install/updating)
