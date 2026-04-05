---
read_when:
    - Necesitas un método de instalación distinto del inicio rápido de Primeros pasos
    - Quieres desplegar en una plataforma en la nube
    - Necesitas actualizar, migrar o desinstalar
summary: 'Instalar OpenClaw: script de instalación, npm/pnpm/bun, desde el código fuente, Docker y más'
title: Instalación
x-i18n:
    generated_at: "2026-04-05T12:45:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: eca17c76a2a66166b3d8cda9dc3144ab920d30ad0ed2a220eb9389d7a383ba5d
    source_path: install/index.md
    workflow: 15
---

# Instalación

## Recomendado: script de instalación

La forma más rápida de instalar. Detecta tu sistema operativo, instala Node si es necesario, instala OpenClaw e inicia el onboarding.

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
</Tabs>

Para instalar sin ejecutar el onboarding:

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

Para ver todos los flags y opciones de CI/automatización, consulta [Elementos internos del instalador](/install/installer).

## Requisitos del sistema

- **Node 24** (recomendado) o Node 22.14+ — el script de instalación se encarga de esto automáticamente
- **macOS, Linux o Windows** — se admiten tanto Windows nativo como WSL2; WSL2 es más estable. Consulta [Windows](/platforms/windows).
- `pnpm` solo es necesario si compilas desde el código fuente

## Métodos de instalación alternativos

### Instalador con prefijo local (`install-cli.sh`)

Úsalo cuando quieras mantener OpenClaw y Node bajo un prefijo local como
`~/.openclaw`, sin depender de una instalación de Node en todo el sistema:

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

Admite instalaciones npm de forma predeterminada, además de instalaciones desde checkout git dentro del mismo
flujo con prefijo. Referencia completa: [Elementos internos del instalador](/install/installer#install-clish).

### npm, pnpm o bun

Si ya gestionas Node por tu cuenta:

<Tabs>
  <Tab title="npm">
    ```bash
    npm install -g openclaw@latest
    openclaw onboard --install-daemon
    ```
  </Tab>
  <Tab title="pnpm">
    ```bash
    pnpm add -g openclaw@latest
    pnpm approve-builds -g
    openclaw onboard --install-daemon
    ```

    <Note>
    pnpm requiere aprobación explícita para paquetes con scripts de compilación. Ejecuta `pnpm approve-builds -g` después de la primera instalación.
    </Note>

  </Tab>
  <Tab title="bun">
    ```bash
    bun add -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    Bun es compatible con la ruta de instalación global de la CLI. Para el runtime de Gateway, Node sigue siendo el runtime recomendado para el daemon.
    </Note>

  </Tab>
</Tabs>

<Accordion title="Solución de problemas: errores de compilación de sharp (npm)">
  Si `sharp` falla debido a una instalación global de libvips:

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

</Accordion>

### Desde el código fuente

Para colaboradores o cualquiera que quiera ejecutarlo desde un checkout local:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install && pnpm ui:build && pnpm build
pnpm link --global
openclaw onboard --install-daemon
```

O bien omite el enlace y usa `pnpm openclaw ...` desde dentro del repositorio. Consulta [Setup](/start/setup) para ver los flujos completos de desarrollo.

### Instalar desde GitHub main

```bash
npm install -g github:openclaw/openclaw#main
```

### Contenedores y gestores de paquetes

<CardGroup cols={2}>
  <Card title="Docker" href="/install/docker" icon="container">
    Despliegues en contenedor o headless.
  </Card>
  <Card title="Podman" href="/install/podman" icon="container">
    Alternativa en contenedor rootless a Docker.
  </Card>
  <Card title="Nix" href="/install/nix" icon="snowflake">
    Instalación declarativa mediante Nix flake.
  </Card>
  <Card title="Ansible" href="/install/ansible" icon="server">
    Aprovisionamiento automatizado de flotas.
  </Card>
  <Card title="Bun" href="/install/bun" icon="zap">
    Uso solo de CLI mediante el runtime de Bun.
  </Card>
</CardGroup>

## Verificar la instalación

```bash
openclaw --version      # confirmar que la CLI está disponible
openclaw doctor         # comprobar si hay problemas de configuración
openclaw gateway status # verificar que la Gateway está en ejecución
```

Si quieres inicio administrado después de la instalación:

- macOS: LaunchAgent mediante `openclaw onboard --install-daemon` o `openclaw gateway install`
- Linux/WSL2: servicio de usuario systemd mediante los mismos comandos
- Windows nativo: primero Scheduled Task, con fallback a un elemento de inicio de sesión por usuario en la carpeta Startup si se deniega la creación de la tarea

## Alojamiento y despliegue

Despliega OpenClaw en un servidor en la nube o VPS:

<CardGroup cols={3}>
  <Card title="VPS" href="/vps">Cualquier VPS Linux</Card>
  <Card title="Docker VM" href="/install/docker-vm-runtime">Pasos compartidos de Docker</Card>
  <Card title="Kubernetes" href="/install/kubernetes">K8s</Card>
  <Card title="Fly.io" href="/install/fly">Fly.io</Card>
  <Card title="Hetzner" href="/install/hetzner">Hetzner</Card>
  <Card title="GCP" href="/install/gcp">Google Cloud</Card>
  <Card title="Azure" href="/install/azure">Azure</Card>
  <Card title="Railway" href="/install/railway">Railway</Card>
  <Card title="Render" href="/install/render">Render</Card>
  <Card title="Northflank" href="/install/northflank">Northflank</Card>
</CardGroup>

## Actualizar, migrar o desinstalar

<CardGroup cols={3}>
  <Card title="Updating" href="/install/updating" icon="refresh-cw">
    Mantén OpenClaw actualizado.
  </Card>
  <Card title="Migrating" href="/install/migrating" icon="arrow-right">
    Muévete a una máquina nueva.
  </Card>
  <Card title="Uninstall" href="/install/uninstall" icon="trash-2">
    Elimina OpenClaw por completo.
  </Card>
</CardGroup>

## Solución de problemas: no se encuentra `openclaw`

Si la instalación se completó correctamente pero `openclaw` no se encuentra en tu terminal:

```bash
node -v           # ¿Node está instalado?
npm prefix -g     # ¿Dónde están los paquetes globales?
echo "$PATH"      # ¿Está el directorio global bin en PATH?
```

Si `$(npm prefix -g)/bin` no está en tu `$PATH`, añádelo al archivo de inicio de tu shell (`~/.zshrc` o `~/.bashrc`):

```bash
export PATH="$(npm prefix -g)/bin:$PATH"
```

Luego abre un terminal nuevo. Consulta [Node setup](/install/node) para más detalles.
