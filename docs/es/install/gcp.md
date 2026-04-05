---
read_when:
    - Quieres OpenClaw en ejecución 24/7 en GCP
    - Quieres un Gateway de nivel de producción, siempre activo, en tu propia VM
    - Quieres control total sobre persistencia, binarios y comportamiento de reinicio
summary: Ejecuta OpenClaw Gateway 24/7 en una VM de GCP Compute Engine (Docker) con estado persistente
title: GCP
x-i18n:
    generated_at: "2026-04-05T12:45:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 73daaee3de71dad5175f42abf3e11355f2603b2f9e2b2523eac4d4c7015e3ebc
    source_path: install/gcp.md
    workflow: 15
---

# OpenClaw en GCP Compute Engine (Docker, guía de VPS de producción)

## Objetivo

Ejecutar un OpenClaw Gateway persistente en una VM de GCP Compute Engine usando Docker, con estado duradero, binarios incorporados y un comportamiento de reinicio seguro.

Si quieres “OpenClaw 24/7 por ~$5-12/mes”, esta es una configuración fiable en Google Cloud.
El precio varía según el tipo de máquina y la región; elige la VM más pequeña que se ajuste a tu carga de trabajo y aumenta de tamaño si encuentras errores OOM.

## ¿Qué estamos haciendo? (en términos sencillos)

- Crear un proyecto de GCP y habilitar la facturación
- Crear una VM de Compute Engine
- Instalar Docker (entorno de ejecución aislado para la app)
- Iniciar OpenClaw Gateway en Docker
- Conservar `~/.openclaw` + `~/.openclaw/workspace` en el host (sobrevive a reinicios/reconstrucciones)
- Acceder a la UI de Control desde tu laptop mediante un túnel SSH

Ese estado montado de `~/.openclaw` incluye `openclaw.json`, por agente
`agents/<agentId>/agent/auth-profiles.json` y `.env`.

Se puede acceder al Gateway mediante:

- reenvío de puerto SSH desde tu laptop
- exposición directa del puerto si administras tú mismo el firewall y los tokens

Esta guía usa Debian en GCP Compute Engine.
Ubuntu también funciona; ajusta los paquetes según corresponda.
Para el flujo genérico de Docker, consulta [Docker](/install/docker).

---

## Ruta rápida (operadores con experiencia)

1. Crear un proyecto de GCP + habilitar la API de Compute Engine
2. Crear una VM de Compute Engine (e2-small, Debian 12, 20GB)
3. Conectarte por SSH a la VM
4. Instalar Docker
5. Clonar el repositorio de OpenClaw
6. Crear directorios persistentes en el host
7. Configurar `.env` y `docker-compose.yml`
8. Incorporar los binarios requeridos, compilar e iniciar

---

## Qué necesitas

- Cuenta de GCP (elegible para el nivel gratuito de e2-micro)
- CLI `gcloud` instalada (o usar Cloud Console)
- Acceso SSH desde tu laptop
- Comodidad básica con SSH + copiar/pegar
- ~20-30 minutos
- Docker y Docker Compose
- Credenciales de autenticación del modelo
- Credenciales opcionales de proveedor
  - QR de WhatsApp
  - token de bot de Telegram
  - OAuth de Gmail

---

<Steps>
  <Step title="Instalar la CLI de gcloud (o usar Console)">
    **Opción A: CLI `gcloud`** (recomendada para automatización)

    Instala desde [https://cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)

    Inicializa y autentícate:

    ```bash
    gcloud init
    gcloud auth login
    ```

    **Opción B: Cloud Console**

    Todos los pasos se pueden hacer mediante la interfaz web en [https://console.cloud.google.com](https://console.cloud.google.com)

  </Step>

  <Step title="Crear un proyecto de GCP">
    **CLI:**

    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    ```

    Habilita la facturación en [https://console.cloud.google.com/billing](https://console.cloud.google.com/billing) (obligatorio para Compute Engine).

    Habilita la API de Compute Engine:

    ```bash
    gcloud services enable compute.googleapis.com
    ```

    **Console:**

    1. Ve a IAM & Admin > Create Project
    2. Ponle nombre y créalo
    3. Habilita la facturación para el proyecto
    4. Ve a APIs & Services > Enable APIs > busca "Compute Engine API" > Enable

  </Step>

  <Step title="Crear la VM">
    **Tipos de máquina:**

    | Tipo      | Especificaciones          | Costo              | Notas                                        |
    | --------- | ------------------------- | ------------------ | -------------------------------------------- |
    | e2-medium | 2 vCPU, 4GB RAM           | ~$25/mes           | La opción más fiable para compilaciones locales con Docker |
    | e2-small  | 2 vCPU, 2GB RAM           | ~$12/mes           | Mínimo recomendado para compilación con Docker |
    | e2-micro  | 2 vCPU (compartidas), 1GB RAM | Elegible para nivel gratuito | A menudo falla con OOM en compilaciones de Docker (exit 137) |

    **CLI:**

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

    **Console:**

    1. Ve a Compute Engine > VM instances > Create instance
    2. Nombre: `openclaw-gateway`
    3. Región: `us-central1`, zona: `us-central1-a`
    4. Tipo de máquina: `e2-small`
    5. Disco de arranque: Debian 12, 20GB
    6. Crear

  </Step>

  <Step title="Conectarte por SSH a la VM">
    **CLI:**

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    **Console:**

    Haz clic en el botón "SSH" junto a tu VM en el panel de Compute Engine.

    Nota: la propagación de claves SSH puede tardar entre 1 y 2 minutos después de crear la VM. Si la conexión es rechazada, espera y vuelve a intentarlo.

  </Step>

  <Step title="Instalar Docker (en la VM)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    Cierra la sesión y vuelve a entrar para que el cambio de grupo surta efecto:

    ```bash
    exit
    ```

    Luego vuelve a conectarte por SSH:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Verificar:

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
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="Configurar variables de entorno">
    Crea `.env` en la raíz del repositorio.

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=change-me-now
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=change-me-now
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Genera secretos robustos:

    ```bash
    openssl rand -hex 32
    ```

    **No confirmes este archivo al repositorio.**

    Este archivo `.env` es para variables de entorno del contenedor/entorno de ejecución como `OPENCLAW_GATEWAY_TOKEN`.
    La autenticación almacenada de proveedores OAuth/claves API vive en
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.

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
          # Recomendado: mantén el Gateway solo en loopback en la VM; accede mediante túnel SSH.
          # Para exponerlo públicamente, elimina el prefijo `127.0.0.1:` y configura el firewall en consecuencia.
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

    `--allow-unconfigured` es solo para facilitar el arranque inicial; no sustituye una configuración correcta del gateway. Aun así, establece la autenticación (`gateway.auth.token` o contraseña) y usa configuraciones de bind seguras para tu implementación.

  </Step>

  <Step title="Pasos compartidos del tiempo de ejecución de Docker en VM">
    Usa la guía compartida de tiempo de ejecución para el flujo común de host Docker:

    - [Incorporar los binarios requeridos en la imagen](/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Compilar e iniciar](/install/docker-vm-runtime#build-and-launch)
    - [Qué se conserva y dónde](/install/docker-vm-runtime#what-persists-where)
    - [Actualizaciones](/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Notas de inicio específicas de GCP">
    En GCP, si la compilación falla con `Killed` o `exit code 137` durante `pnpm install --frozen-lockfile`, la VM no tiene suficiente memoria. Usa como mínimo `e2-small`, o `e2-medium` para que las primeras compilaciones sean más fiables.

    Cuando hagas bind a la LAN (`OPENCLAW_GATEWAY_BIND=lan`), configura un origen de navegador de confianza antes de continuar:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    Si cambiaste el puerto del gateway, sustituye `18789` por el puerto configurado.

  </Step>

  <Step title="Acceder desde tu laptop">
    Crea un túnel SSH para reenviar el puerto del Gateway:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    Ábrelo en tu navegador:

    `http://127.0.0.1:18789/`

    Vuelve a imprimir un enlace limpio del panel:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    Si la UI solicita autenticación por secreto compartido, pega el token o la
    contraseña configurados en la configuración de la UI de Control. Este flujo de Docker escribe un token por
    defecto; si cambias la configuración del contenedor a autenticación por contraseña, usa esa
    contraseña en su lugar.

    Si la UI de Control muestra `unauthorized` o `disconnected (1008): pairing required`, aprueba el dispositivo del navegador:

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    ¿Necesitas otra vez la referencia de persistencia y actualización compartidas?
    Consulta [Tiempo de ejecución de Docker en VM](/install/docker-vm-runtime#what-persists-where) y [actualizaciones del tiempo de ejecución de Docker en VM](/install/docker-vm-runtime#updates).

  </Step>
</Steps>

---

## Solución de problemas

**Conexión SSH rechazada**

La propagación de claves SSH puede tardar entre 1 y 2 minutos después de crear la VM. Espera y vuelve a intentarlo.

**Problemas de OS Login**

Comprueba tu perfil de OS Login:

```bash
gcloud compute os-login describe-profile
```

Asegúrate de que tu cuenta tenga los permisos IAM necesarios (Compute OS Login o Compute OS Admin Login).

**Falta de memoria (OOM)**

Si la compilación de Docker falla con `Killed` y `exit code 137`, la VM fue finalizada por OOM. Sube a e2-small (mínimo) o e2-medium (recomendado para compilaciones locales fiables):

```bash
# Stop the VM first
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# Change machine type
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# Start the VM
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

---

## Cuentas de servicio (mejor práctica de seguridad)

Para uso personal, tu cuenta de usuario predeterminada funciona bien.

Para automatización o canalizaciones CI/CD, crea una cuenta de servicio dedicada con permisos mínimos:

1. Crea una cuenta de servicio:

   ```bash
   gcloud iam service-accounts create openclaw-deploy \
     --display-name="OpenClaw Deployment"
   ```

2. Concede el rol Compute Instance Admin (o un rol personalizado más limitado):

   ```bash
   gcloud projects add-iam-policy-binding my-openclaw-project \
     --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
     --role="roles/compute.instanceAdmin.v1"
   ```

Evita usar el rol Owner para automatización. Aplica el principio de privilegio mínimo.

Consulta [https://cloud.google.com/iam/docs/understanding-roles](https://cloud.google.com/iam/docs/understanding-roles) para detalles de roles IAM.

---

## Siguientes pasos

- Configurar canales de mensajería: [Canales](/channels)
- Emparejar dispositivos locales como nodos: [Nodos](/nodes)
- Configurar el Gateway: [Configuración del gateway](/gateway/configuration)
