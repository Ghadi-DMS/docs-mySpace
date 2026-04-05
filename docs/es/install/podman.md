---
read_when:
    - Quieres un gateway en contenedor con Podman en lugar de Docker
summary: Ejecuta OpenClaw en un contenedor Podman sin privilegios
title: Podman
x-i18n:
    generated_at: "2026-04-05T12:46:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6cb06e2d85b4b0c8a8c6e69c81f629c83b447cbcbb32e34b7876a1819c488020
    source_path: install/podman.md
    workflow: 15
---

# Podman

Ejecuta OpenClaw Gateway en un contenedor Podman sin privilegios, gestionado por tu usuario actual no root.

El modelo previsto es:

- Podman ejecuta el contenedor del gateway.
- La CLI `openclaw` de tu host es el plano de control.
- El estado persistente vive en el host bajo `~/.openclaw` de forma predeterminada.
- La gestión diaria usa `openclaw --container <name> ...` en lugar de `sudo -u openclaw`, `podman exec` o un usuario de servicio independiente.

## Requisitos previos

- **Podman** en modo rootless
- **CLI de OpenClaw** instalada en el host
- **Opcional:** `systemd --user` si quieres arranque automático gestionado por Quadlet
- **Opcional:** `sudo` solo si quieres `loginctl enable-linger "$(whoami)"` para persistencia tras reinicio en un host sin interfaz

## Inicio rápido

<Steps>
  <Step title="Configuración inicial">
    Desde la raíz del repositorio, ejecuta `./scripts/podman/setup.sh`.
  </Step>

  <Step title="Iniciar el contenedor del Gateway">
    Inicia el contenedor con `./scripts/run-openclaw-podman.sh launch`.
  </Step>

  <Step title="Ejecutar onboarding dentro del contenedor">
    Ejecuta `./scripts/run-openclaw-podman.sh launch setup`, luego abre `http://127.0.0.1:18789/`.
  </Step>

  <Step title="Gestionar el contenedor en ejecución desde la CLI del host">
    Establece `OPENCLAW_CONTAINER=openclaw` y luego usa comandos normales de `openclaw` desde el host.
  </Step>
</Steps>

Detalles de la configuración:

- `./scripts/podman/setup.sh` compila `openclaw:local` en tu almacén rootless de Podman de forma predeterminada, o usa `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` si defines alguno.
- Crea `~/.openclaw/openclaw.json` con `gateway.mode: "local"` si falta.
- Crea `~/.openclaw/.env` con `OPENCLAW_GATEWAY_TOKEN` si falta.
- Para inicios manuales, el helper lee solo una pequeña lista de permitidos de claves relacionadas con Podman desde `~/.openclaw/.env` y pasa variables de entorno de runtime explícitas al contenedor; no entrega el archivo de entorno completo a Podman.

Configuración gestionada por Quadlet:

```bash
./scripts/podman/setup.sh --quadlet
```

Quadlet es una opción solo para Linux porque depende de servicios systemd de usuario.

También puedes establecer `OPENCLAW_PODMAN_QUADLET=1`.

Variables de entorno opcionales para compilación/configuración:

- `OPENCLAW_IMAGE` o `OPENCLAW_PODMAN_IMAGE` -- usa una imagen existente/descargada en lugar de compilar `openclaw:local`
- `OPENCLAW_DOCKER_APT_PACKAGES` -- instala paquetes apt adicionales durante la compilación de la imagen
- `OPENCLAW_EXTENSIONS` -- preinstala dependencias de extensiones en tiempo de compilación

Inicio del contenedor:

```bash
./scripts/run-openclaw-podman.sh launch
```

El script inicia el contenedor con tu uid/gid actual usando `--userns=keep-id` y monta por bind tu estado de OpenClaw dentro del contenedor.

Onboarding:

```bash
./scripts/run-openclaw-podman.sh launch setup
```

Luego abre `http://127.0.0.1:18789/` y usa el token de `~/.openclaw/.env`.

Valor predeterminado de la CLI del host:

```bash
export OPENCLAW_CONTAINER=openclaw
```

Entonces comandos como estos se ejecutarán automáticamente dentro de ese contenedor:

```bash
openclaw dashboard --no-open
openclaw gateway status --deep   # includes extra service scan
openclaw doctor
openclaw channels login
```

En macOS, Podman machine puede hacer que el navegador parezca no local para el gateway.
Si la interfaz de Control informa errores de autenticación de dispositivo después del inicio, usa la guía de Tailscale en
[Podman + Tailscale](#podman--tailscale).

<a id="podman--tailscale"></a>

## Podman + Tailscale

Para HTTPS o acceso remoto desde navegador, sigue la documentación principal de Tailscale.

Nota específica de Podman:

- Mantén el host publicado de Podman en `127.0.0.1`.
- Prefiere `tailscale serve` gestionado por el host en lugar de `openclaw gateway --tailscale serve`.
- En macOS, si el contexto de autenticación de dispositivo del navegador local no es fiable, usa acceso mediante Tailscale en lugar de soluciones improvisadas con túneles locales.

Consulta:

- [Tailscale](/gateway/tailscale)
- [Interfaz de Control](/web/control-ui)

## Systemd (Quadlet, opcional)

Si ejecutaste `./scripts/podman/setup.sh --quadlet`, la configuración instala un archivo Quadlet en:

```bash
~/.config/containers/systemd/openclaw.container
```

Comandos útiles:

- **Iniciar:** `systemctl --user start openclaw.service`
- **Detener:** `systemctl --user stop openclaw.service`
- **Estado:** `systemctl --user status openclaw.service`
- **Registros:** `journalctl --user -u openclaw.service -f`

Después de editar el archivo Quadlet:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw.service
```

Para persistencia tras arranque en hosts SSH/sin interfaz, habilita lingering para tu usuario actual:

```bash
sudo loginctl enable-linger "$(whoami)"
```

## Configuración, entorno y almacenamiento

- **Directorio de configuración:** `~/.openclaw`
- **Directorio del espacio de trabajo:** `~/.openclaw/workspace`
- **Archivo de token:** `~/.openclaw/.env`
- **Helper de lanzamiento:** `./scripts/run-openclaw-podman.sh`

El script de lanzamiento y Quadlet montan por bind el estado del host dentro del contenedor:

- `OPENCLAW_CONFIG_DIR` -> `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR` -> `/home/node/.openclaw/workspace`

De forma predeterminada, estos son directorios del host, no estado anónimo del contenedor, así que
`openclaw.json`, `auth-profiles.json` por agente, el estado de canales/proveedores,
las sesiones y el espacio de trabajo sobreviven al reemplazo del contenedor.
La configuración de Podman también inicializa `gateway.controlUi.allowedOrigins` para `127.0.0.1` y `localhost` en el puerto publicado del gateway para que el dashboard local funcione con el bind no loopback del contenedor.

Variables de entorno útiles para el lanzador manual:

- `OPENCLAW_PODMAN_CONTAINER` -- nombre del contenedor (`openclaw` de forma predeterminada)
- `OPENCLAW_PODMAN_IMAGE` / `OPENCLAW_IMAGE` -- imagen a ejecutar
- `OPENCLAW_PODMAN_GATEWAY_HOST_PORT` -- puerto del host asignado al `18789` del contenedor
- `OPENCLAW_PODMAN_BRIDGE_HOST_PORT` -- puerto del host asignado al `18790` del contenedor
- `OPENCLAW_PODMAN_PUBLISH_HOST` -- interfaz del host para puertos publicados; el valor predeterminado es `127.0.0.1`
- `OPENCLAW_GATEWAY_BIND` -- modo de bind del gateway dentro del contenedor; el valor predeterminado es `lan`
- `OPENCLAW_PODMAN_USERNS` -- `keep-id` (predeterminado), `auto` o `host`

El lanzador manual lee `~/.openclaw/.env` antes de finalizar los valores predeterminados de contenedor/imagen, así que puedes persistirlos ahí.

Si usas un `OPENCLAW_CONFIG_DIR` o `OPENCLAW_WORKSPACE_DIR` no predeterminados, establece las mismas variables tanto para `./scripts/podman/setup.sh` como para los comandos posteriores `./scripts/run-openclaw-podman.sh launch`. El lanzador local del repositorio no persiste sobrescrituras de rutas personalizadas entre shells.

Nota sobre Quadlet:

- El servicio Quadlet generado mantiene intencionadamente una forma predeterminada fija y endurecida: puertos publicados en `127.0.0.1`, `--bind lan` dentro del contenedor y espacio de nombres de usuario `keep-id`.
- Fija `OPENCLAW_NO_RESPAWN=1`, `Restart=on-failure` y `TimeoutStartSec=300`.
- Publica tanto `127.0.0.1:18789:18789` (gateway) como `127.0.0.1:18790:18790` (bridge).
- Lee `~/.openclaw/.env` como `EnvironmentFile` de runtime para valores como `OPENCLAW_GATEWAY_TOKEN`, pero no consume la lista de permitidos de sobrescrituras específicas de Podman del lanzador manual.
- Si necesitas puertos de publicación personalizados, host de publicación u otros flags de ejecución del contenedor, usa el lanzador manual o edita directamente `~/.config/containers/systemd/openclaw.container`, luego vuelve a cargar y reinicia el servicio.

## Comandos útiles

- **Registros del contenedor:** `podman logs -f openclaw`
- **Detener contenedor:** `podman stop openclaw`
- **Eliminar contenedor:** `podman rm -f openclaw`
- **Abrir URL del dashboard desde la CLI del host:** `openclaw dashboard --no-open`
- **Estado/salud desde la CLI del host:** `openclaw gateway status --deep` (sondeo RPC + exploración adicional
  de servicios)

## Resolución de problemas

- **Permiso denegado (EACCES) en configuración o espacio de trabajo:** El contenedor se ejecuta con `--userns=keep-id` y `--user <your uid>:<your gid>` de forma predeterminada. Asegúrate de que las rutas de configuración/espacio de trabajo del host sean propiedad de tu usuario actual.
- **Inicio del Gateway bloqueado (falta `gateway.mode=local`):** Asegúrate de que `~/.openclaw/openclaw.json` exista y establezca `gateway.mode="local"`. `scripts/podman/setup.sh` lo crea si falta.
- **Los comandos CLI del contenedor llegan al destino equivocado:** Usa `openclaw --container <name> ...` explícitamente, o exporta `OPENCLAW_CONTAINER=<name>` en tu shell.
- **`openclaw update` falla con `--container`:** Es lo esperado. Vuelve a compilar/descargar la imagen y luego reinicia el contenedor o el servicio Quadlet.
- **El servicio Quadlet no inicia:** Ejecuta `systemctl --user daemon-reload`, luego `systemctl --user start openclaw.service`. En sistemas sin interfaz también puede que necesites `sudo loginctl enable-linger "$(whoami)"`.
- **SELinux bloquea montajes bind:** Deja el comportamiento de montaje predeterminado; el lanzador añade automáticamente `:Z` en Linux cuando SELinux está en modo enforcing o permissive.

## Relacionado

- [Docker](/install/docker)
- [Proceso en segundo plano del Gateway](/gateway/background-process)
- [Resolución de problemas del Gateway](/gateway/troubleshooting)
