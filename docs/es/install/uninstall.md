---
read_when:
    - Quieres eliminar OpenClaw de una máquina
    - El servicio de gateway sigue ejecutándose después de la desinstalación
summary: Desinstalar OpenClaw por completo (CLI, servicio, estado, espacio de trabajo)
title: Desinstalar
x-i18n:
    generated_at: "2026-04-05T12:46:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 34c7d3e4ad17333439048dfda739fc27db47e7f9e4212fe17db0e4eb3d3ab258
    source_path: install/uninstall.md
    workflow: 15
---

# Desinstalar

Hay dos rutas:

- **Ruta fácil** si `openclaw` sigue instalado.
- **Eliminación manual del servicio** si la CLI ya no está pero el servicio sigue ejecutándose.

## Ruta fácil (la CLI sigue instalada)

Recomendado: usa el desinstalador integrado:

```bash
openclaw uninstall
```

Modo no interactivo (automatización / npx):

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

Pasos manuales (mismo resultado):

1. Detén el servicio de gateway:

```bash
openclaw gateway stop
```

2. Desinstala el servicio de gateway (launchd/systemd/schtasks):

```bash
openclaw gateway uninstall
```

3. Elimina estado + configuración:

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

Si estableciste `OPENCLAW_CONFIG_PATH` en una ubicación personalizada fuera del directorio de estado, elimina también ese archivo.

4. Elimina tu espacio de trabajo (opcional, elimina archivos del agente):

```bash
rm -rf ~/.openclaw/workspace
```

5. Elimina la instalación de la CLI (elige la que usaste):

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. Si instalaste la app de macOS:

```bash
rm -rf /Applications/OpenClaw.app
```

Notas:

- Si usaste perfiles (`--profile` / `OPENCLAW_PROFILE`), repite el paso 3 para cada directorio de estado (los predeterminados son `~/.openclaw-<profile>`).
- En modo remoto, el directorio de estado vive en el **host de la gateway**, así que ejecuta también allí los pasos 1-4.

## Eliminación manual del servicio (CLI no instalada)

Úsala si el servicio de gateway sigue ejecutándose pero `openclaw` no existe.

### macOS (launchd)

La etiqueta predeterminada es `ai.openclaw.gateway` (o `ai.openclaw.<profile>`; la heredada `com.openclaw.*` puede seguir existiendo):

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

Si usaste un perfil, reemplaza la etiqueta y el nombre del plist por `ai.openclaw.<profile>`. Elimina cualquier plist heredado `com.openclaw.*` si existe.

### Linux (unidad de usuario systemd)

El nombre predeterminado de la unidad es `openclaw-gateway.service` (o `openclaw-gateway-<profile>.service`):

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows (Scheduled Task)

El nombre predeterminado de la tarea es `OpenClaw Gateway` (o `OpenClaw Gateway (<profile>)`).
El script de la tarea vive en tu directorio de estado.

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd"
```

Si usaste un perfil, elimina el nombre de tarea correspondiente y `~\.openclaw-<profile>\gateway.cmd`.

## Instalación normal frente a checkout del código fuente

### Instalación normal (install.sh / npm / pnpm / bun)

Si usaste `https://openclaw.ai/install.sh` o `install.ps1`, la CLI se instaló con `npm install -g openclaw@latest`.
Elimínala con `npm rm -g openclaw` (o `pnpm remove -g` / `bun remove -g` si la instalaste de esa forma).

### Checkout del código fuente (git clone)

Si ejecutas desde un checkout del repositorio (`git clone` + `openclaw ...` / `bun run openclaw ...`):

1. Desinstala el servicio de gateway **antes** de eliminar el repositorio (usa la ruta fácil anterior o la eliminación manual del servicio).
2. Elimina el directorio del repositorio.
3. Elimina estado + espacio de trabajo como se muestra arriba.
