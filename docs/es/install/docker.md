---
read_when:
    - Quieres una gateway en contenedor en lugar de instalaciones locales
    - Estás validando el flujo de Docker
summary: Configuración opcional basada en Docker y onboarding para OpenClaw
title: Docker
x-i18n:
    generated_at: "2026-04-05T12:45:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4628362d52597f85e72c214efe96b2923c7a59a8592b3044dc8c230318c515b8
    source_path: install/docker.md
    workflow: 15
---

# Docker (opcional)

Docker es **opcional**. Úsalo solo si quieres una gateway en contenedor o validar el flujo de Docker.

## ¿Docker es adecuado para mí?

- **Sí**: quieres un entorno de gateway aislado y desechable, o ejecutar OpenClaw en un host sin instalaciones locales.
- **No**: estás ejecutándolo en tu propia máquina y solo quieres el bucle de desarrollo más rápido. Usa en su lugar el flujo normal de instalación.
- **Nota sobre sandboxing**: el sandboxing del agente también usa Docker, pero **no** requiere que toda la gateway se ejecute en Docker. Consulta [Sandboxing](/gateway/sandboxing).

## Requisitos previos

- Docker Desktop (o Docker Engine) + Docker Compose v2
- Al menos 2 GB de RAM para la compilación de la imagen (`pnpm install` puede terminar por OOM en hosts de 1 GB con salida 137)
- Suficiente espacio en disco para imágenes y registros
- Si se ejecuta en un VPS/host público, revisa
  [Endurecimiento de seguridad para exposición de red](/gateway/security),
  especialmente la política de firewall `DOCKER-USER` de Docker.

## Gateway en contenedor

<Steps>
  <Step title="Construir la imagen">
    Desde la raíz del repositorio, ejecuta el script de configuración:

    ```bash
    ./scripts/docker/setup.sh
    ```

    Esto construye la imagen de la gateway localmente. Para usar una imagen preconstruida:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    Las imágenes preconstruidas se publican en el
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw).
    Etiquetas comunes: `main`, `latest`, `<version>` (por ejemplo `2026.2.26`).

  </Step>

  <Step title="Completar el onboarding">
    El script de configuración ejecuta el onboarding automáticamente. Hará lo siguiente:

    - solicitará API keys de proveedores
    - generará un token de gateway y lo escribirá en `.env`
    - iniciará la gateway mediante Docker Compose

    Durante la configuración, el onboarding previo al arranque y las escrituras de configuración se ejecutan mediante
    `openclaw-gateway` directamente. `openclaw-cli` es para comandos que ejecutas después de
    que el contenedor de la gateway ya exista.

  </Step>

  <Step title="Abrir la interfaz de usuario de Control">
    Abre `http://127.0.0.1:18789/` en tu navegador y pega el secreto compartido
    configurado en Settings. El script de configuración escribe un token en `.env` de
    forma predeterminada; si cambias la configuración del contenedor a autenticación por contraseña, usa esa
    contraseña en su lugar.

    ¿Necesitas la URL otra vez?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="Configurar canales (opcional)">
    Usa el contenedor de CLI para añadir canales de mensajería:

    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    Documentación: [WhatsApp](/channels/whatsapp), [Telegram](/channels/telegram), [Discord](/channels/discord)

  </Step>
</Steps>

### Flujo manual

Si prefieres ejecutar cada paso tú mismo en lugar de usar el script de configuración:

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.mode local
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.bind lan
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.controlUi.allowedOrigins \
  '["http://localhost:18789","http://127.0.0.1:18789"]' --strict-json
docker compose up -d openclaw-gateway
```

<Note>
Ejecuta `docker compose` desde la raíz del repositorio. Si habilitaste `OPENCLAW_EXTRA_MOUNTS`
o `OPENCLAW_HOME_VOLUME`, el script de configuración escribe `docker-compose.extra.yml`;
inclúyelo con `-f docker-compose.yml -f docker-compose.extra.yml`.
</Note>

<Note>
Como `openclaw-cli` comparte el espacio de nombres de red de `openclaw-gateway`, es una
herramienta posterior al arranque. Antes de `docker compose up -d openclaw-gateway`, ejecuta el onboarding
y las escrituras de configuración de tiempo de configuración mediante `openclaw-gateway` con
`--no-deps --entrypoint node`.
</Note>

### Variables de entorno

El script de configuración acepta estas variables de entorno opcionales:

| Variable                       | Propósito                                                        |
| ------------------------------ | ---------------------------------------------------------------- |
| `OPENCLAW_IMAGE`               | Usar una imagen remota en lugar de compilar localmente           |
| `OPENCLAW_DOCKER_APT_PACKAGES` | Instalar paquetes apt adicionales durante la compilación (separados por espacios) |
| `OPENCLAW_EXTENSIONS`          | Preinstalar dependencias de extensiones en tiempo de compilación (nombres separados por espacios) |
| `OPENCLAW_EXTRA_MOUNTS`        | Bind mounts extra del host (separados por comas `source:target[:opts]`) |
| `OPENCLAW_HOME_VOLUME`         | Persistir `/home/node` en un volumen Docker con nombre           |
| `OPENCLAW_SANDBOX`             | Activar el bootstrap de sandbox (`1`, `true`, `yes`, `on`)       |
| `OPENCLAW_DOCKER_SOCKET`       | Sobrescribir la ruta del socket de Docker                        |

### Comprobaciones de salud

Endpoints de sondeo del contenedor (no requieren autenticación):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz     # readiness
```

La imagen de Docker incluye un `HEALTHCHECK` integrado que consulta `/healthz`.
Si las comprobaciones siguen fallando, Docker marca el contenedor como `unhealthy` y
los sistemas de orquestación pueden reiniciarlo o reemplazarlo.

Instantánea profunda autenticada de salud:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN frente a loopback

`scripts/docker/setup.sh` usa de forma predeterminada `OPENCLAW_GATEWAY_BIND=lan` para que el acceso del host a
`http://127.0.0.1:18789` funcione con la publicación de puertos de Docker.

- `lan` (predeterminado): el navegador del host y la CLI del host pueden alcanzar el puerto publicado de la gateway.
- `loopback`: solo los procesos dentro del espacio de nombres de red del contenedor pueden alcanzar
  directamente la gateway.

<Note>
Usa valores de modo de enlace en `gateway.bind` (`lan` / `loopback` / `custom` /
`tailnet` / `auto`), no alias de host como `0.0.0.0` o `127.0.0.1`.
</Note>

### Almacenamiento y persistencia

Docker Compose hace bind-mount de `OPENCLAW_CONFIG_DIR` a `/home/node/.openclaw` y
`OPENCLAW_WORKSPACE_DIR` a `/home/node/.openclaw/workspace`, de modo que esas rutas
sobreviven al reemplazo del contenedor.

Ese directorio de configuración montado es donde OpenClaw guarda:

- `openclaw.json` para la configuración de comportamiento
- `agents/<agentId>/agent/auth-profiles.json` para la autenticación almacenada de OAuth/API-key de proveedores
- `.env` para secretos de runtime respaldados por entorno, como `OPENCLAW_GATEWAY_TOKEN`

Para ver detalles completos de persistencia en despliegues de VM, consulta
[Docker VM Runtime - What persists where](/install/docker-vm-runtime#what-persists-where).

**Puntos críticos de crecimiento en disco:** vigila `media/`, archivos JSONL de sesión, `cron/runs/*.jsonl`,
y registros rotativos de archivos en `/tmp/openclaw/`.

### Helpers de shell (opcional)

Para una gestión diaria más sencilla de Docker, instala `ClawDock`:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

Si instalaste ClawDock desde la antigua ruta raw `scripts/shell-helpers/clawdock-helpers.sh`, vuelve a ejecutar el comando de instalación anterior para que tu archivo helper local siga la nueva ubicación.

Luego usa `clawdock-start`, `clawdock-stop`, `clawdock-dashboard`, etc. Ejecuta
`clawdock-help` para ver todos los comandos.
Consulta [ClawDock](/install/clawdock) para la guía completa de helpers.

<AccordionGroup>
  <Accordion title="Habilitar sandbox del agente para la gateway en Docker">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    Ruta de socket personalizada (por ejemplo, Docker rootless):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    El script monta `docker.sock` solo después de que pasen los requisitos previos del sandbox. Si
    la configuración del sandbox no puede completarse, el script restablece `agents.defaults.sandbox.mode`
    a `off`.

  </Accordion>

  <Accordion title="Automatización / CI (no interactivo)">
    Desactiva la asignación de pseudo-TTY de Compose con `-T`:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="Nota de seguridad de red compartida">
    `openclaw-cli` usa `network_mode: "service:openclaw-gateway"` para que los comandos
    de CLI puedan alcanzar la gateway por `127.0.0.1`. Trata esto como un
    límite de confianza compartido. La configuración de compose elimina `NET_RAW`/`NET_ADMIN` y habilita
    `no-new-privileges` en `openclaw-cli`.
  </Accordion>

  <Accordion title="Permisos y EACCES">
    La imagen se ejecuta como `node` (uid 1000). Si ves errores de permisos en
    `/home/node/.openclaw`, asegúrate de que los bind mounts de tu host pertenezcan al uid 1000:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

  </Accordion>

  <Accordion title="Reconstrucciones más rápidas">
    Ordena tu Dockerfile para que las capas de dependencias queden en caché. Esto evita volver a ejecutar
    `pnpm install` salvo que cambien los lockfiles:

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="Opciones de contenedor para usuarios avanzados">
    La imagen predeterminada prioriza la seguridad y se ejecuta como usuario no root `node`. Para un
    contenedor con más funciones:

    1. **Persistir `/home/node`**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **Incorporar dependencias del sistema**: `export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"`
    3. **Instalar navegadores de Playwright**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    4. **Persistir descargas del navegador**: establece
       `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright` y usa
       `OPENCLAW_HOME_VOLUME` o `OPENCLAW_EXTRA_MOUNTS`.

  </Accordion>

  <Accordion title="OAuth de OpenAI Codex (Docker headless)">
    Si eliges OAuth de OpenAI Codex en el asistente, se abre una URL en el navegador. En
    configuraciones Docker o headless, copia la URL completa de redirección a la que llegues y pégala
    de vuelta en el asistente para completar la autenticación.
  </Accordion>

  <Accordion title="Metadatos de imagen base">
    La imagen principal de Docker usa `node:24-bookworm` y publica anotaciones OCI de imagen base
    incluyendo `org.opencontainers.image.base.name`,
    `org.opencontainers.image.source` y otras. Consulta
    [OCI image annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md).
  </Accordion>
</AccordionGroup>

### ¿Ejecutándolo en un VPS?

Consulta [Hetzner (Docker VPS)](/install/hetzner) y
[Docker VM Runtime](/install/docker-vm-runtime) para ver los pasos de despliegue en VM compartida,
incluidos binary baking, persistencia y actualizaciones.

## Sandbox del agente

Cuando `agents.defaults.sandbox` está habilitado, la gateway ejecuta la ejecución de herramientas del agente
(shell, lectura/escritura de archivos, etc.) dentro de contenedores Docker aislados, mientras que la
propia gateway permanece en el host. Esto te da una barrera fuerte alrededor de sesiones de agentes no confiables o
multi-tenant sin containerizar toda la gateway.

El ámbito del sandbox puede ser por agente (predeterminado), por sesión o compartido. Cada ámbito
obtiene su propio espacio de trabajo montado en `/workspace`. También puedes configurar
políticas de herramientas de permitir/denegar, aislamiento de red, límites de recursos y contenedores de navegador.

Para ver la configuración completa, imágenes, notas de seguridad y perfiles multiagente, consulta:

- [Sandboxing](/gateway/sandboxing) -- referencia completa de sandbox
- [OpenShell](/gateway/openshell) -- acceso de shell interactivo a contenedores sandbox
- [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) -- sobrescrituras por agente

### Activación rápida

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared
      },
    },
  },
}
```

Construye la imagen sandbox predeterminada:

```bash
scripts/sandbox-setup.sh
```

## Solución de problemas

<AccordionGroup>
  <Accordion title="Falta la imagen o el contenedor sandbox no arranca">
    Construye la imagen sandbox con
    [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)
    o establece `agents.defaults.sandbox.docker.image` en tu imagen personalizada.
    Los contenedores se crean automáticamente por sesión bajo demanda.
  </Accordion>

  <Accordion title="Errores de permisos en el sandbox">
    Establece `docker.user` en un UID:GID que coincida con la propiedad de tu espacio de trabajo montado,
    o haz chown de la carpeta del espacio de trabajo.
  </Accordion>

  <Accordion title="No se encuentran herramientas personalizadas en el sandbox">
    OpenClaw ejecuta comandos con `sh -lc` (shell de inicio de sesión), que carga
    `/etc/profile` y puede restablecer PATH. Establece `docker.env.PATH` para anteponer tus
    rutas personalizadas de herramientas, o añade un script en `/etc/profile.d/` en tu Dockerfile.
  </Accordion>

  <Accordion title="Finalización por OOM durante la compilación de imagen (salida 137)">
    La VM necesita al menos 2 GB de RAM. Usa una clase de máquina mayor y vuelve a intentarlo.
  </Accordion>

  <Accordion title="Unauthorized o se requiere emparejamiento en la interfaz de usuario de Control">
    Obtén un enlace fresco de dashboard y aprueba el dispositivo del navegador:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    Más detalle: [Dashboard](/web/dashboard), [Devices](/cli/devices).

  </Accordion>

  <Accordion title="El destino de gateway muestra ws://172.x.x.x o errores de emparejamiento desde Docker CLI">
    Restablece el modo y enlace de la gateway:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.mode local
    docker compose run --rm openclaw-cli config set gateway.bind lan
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## Relacionado

- [Resumen de instalación](/install) — todos los métodos de instalación
- [Podman](/install/podman) — alternativa de Podman a Docker
- [ClawDock](/install/clawdock) — configuración comunitaria de Docker Compose
- [Updating](/install/updating) — mantener OpenClaw actualizado
- [Configuración](/gateway/configuration) — configuración de gateway después de la instalación
