---
read_when:
    - Quieres entender `openclaw.ai/install.sh`
    - Quieres automatizar instalaciones (CI / sin interfaz)
    - Quieres instalar desde una copia de GitHub
summary: Cómo funcionan los scripts de instalación (`install.sh`, `install-cli.sh`, `install.ps1`), sus flags y la automatización
title: Detalles internos del instalador
x-i18n:
    generated_at: "2026-04-05T12:46:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: eced891572b8825b1f8a26ccc9d105ae8a38bd8ad89baef2f1927e27d4619e04
    source_path: install/installer.md
    workflow: 15
---

# Detalles internos del instalador

OpenClaw incluye tres scripts de instalación, servidos desde `openclaw.ai`.

| Script                             | Plataforma           | Qué hace                                                                                                       |
| ---------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| [`install.sh`](#installsh)         | macOS / Linux / WSL  | Instala Node si es necesario, instala OpenClaw mediante npm (predeterminado) o git, y puede ejecutar onboarding. |
| [`install-cli.sh`](#install-clish) | macOS / Linux / WSL  | Instala Node + OpenClaw en un prefijo local (`~/.openclaw`) con modos npm o copia git. No requiere root.     |
| [`install.ps1`](#installps1)       | Windows (PowerShell) | Instala Node si es necesario, instala OpenClaw mediante npm (predeterminado) o git, y puede ejecutar onboarding. |

## Comandos rápidos

<Tabs>
  <Tab title="install.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install-cli.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install.ps1">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag beta -NoOnboard -DryRun
    ```

  </Tab>
</Tabs>

<Note>
Si la instalación se completa correctamente pero `openclaw` no aparece en una terminal nueva, consulta [Resolución de problemas de Node.js](/install/node#troubleshooting).
</Note>

---

<a id="installsh"></a>

## install.sh

<Tip>
Recomendado para la mayoría de instalaciones interactivas en macOS/Linux/WSL.
</Tip>

### Flujo (`install.sh`)

<Steps>
  <Step title="Detectar el SO">
    Compatible con macOS y Linux (incluido WSL). Si detecta macOS, instala Homebrew si falta.
  </Step>
  <Step title="Garantizar Node.js 24 de forma predeterminada">
    Comprueba la versión de Node e instala Node 24 si es necesario (Homebrew en macOS, scripts de configuración de NodeSource en apt/dnf/yum de Linux). OpenClaw sigue siendo compatible con Node 22 LTS, actualmente `22.14+`, por compatibilidad.
  </Step>
  <Step title="Garantizar Git">
    Instala Git si falta.
  </Step>
  <Step title="Instalar OpenClaw">
    - Método `npm` (predeterminado): instalación global con npm
    - Método `git`: clona/actualiza el repositorio, instala dependencias con pnpm, compila y luego instala el wrapper en `~/.local/bin/openclaw`
  </Step>
  <Step title="Tareas posteriores a la instalación">
    - Actualiza, en el mejor esfuerzo, un servicio gateway cargado (`openclaw gateway install --force`, luego reinicio)
    - Ejecuta `openclaw doctor --non-interactive` en actualizaciones e instalaciones con git (mejor esfuerzo)
    - Intenta onboarding cuando corresponde (TTY disponible, onboarding no desactivado y comprobaciones de bootstrap/configuración superadas)
    - Usa por defecto `SHARP_IGNORE_GLOBAL_LIBVIPS=1`
  </Step>
</Steps>

### Detección de copia del código fuente

Si se ejecuta dentro de una copia de OpenClaw (`package.json` + `pnpm-workspace.yaml`), el script ofrece:

- usar la copia (`git`), o
- usar la instalación global (`npm`)

Si no hay TTY disponible y no se define un método de instalación, usa `npm` de forma predeterminada y muestra una advertencia.

El script sale con código `2` para selecciones de método no válidas o valores `--install-method` no válidos.

### Ejemplos (`install.sh`)

<Tabs>
  <Tab title="Predeterminado">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Omitir onboarding">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Instalación con git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```
  </Tab>
  <Tab title="GitHub main mediante npm">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --version main
    ```
  </Tab>
  <Tab title="Simulación">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --dry-run
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de flags">

| Flag                                  | Descripción                                                 |
| ------------------------------------- | ----------------------------------------------------------- |
| `--install-method npm\|git`           | Elige el método de instalación (predeterminado: `npm`). Alias: `--method` |
| `--npm`                               | Atajo para el método npm                                    |
| `--git`                               | Atajo para el método git. Alias: `--github`                 |
| `--version <version\|dist-tag\|spec>` | Versión npm, dist-tag o especificación de paquete (predeterminado: `latest`) |
| `--beta`                              | Usa la dist-tag beta si está disponible; en caso contrario usa `latest` |
| `--git-dir <path>`                    | Directorio de la copia (predeterminado: `~/openclaw`). Alias: `--dir` |
| `--no-git-update`                     | Omite `git pull` en copias existentes                       |
| `--no-prompt`                         | Desactiva los prompts                                       |
| `--no-onboard`                        | Omite onboarding                                            |
| `--onboard`                           | Habilita onboarding                                         |
| `--dry-run`                           | Muestra las acciones sin aplicar cambios                    |
| `--verbose`                           | Habilita salida de depuración (`set -x`, registros npm de nivel notice) |
| `--help`                              | Muestra el uso (`-h`)                                       |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                                                | Descripción                                   |
| ------------------------------------------------------- | --------------------------------------------- |
| `OPENCLAW_INSTALL_METHOD=git\|npm`                      | Método de instalación                         |
| `OPENCLAW_VERSION=latest\|next\|main\|<semver>\|<spec>` | Versión npm, dist-tag o especificación de paquete |
| `OPENCLAW_BETA=0\|1`                                    | Usar beta si está disponible                  |
| `OPENCLAW_GIT_DIR=<path>`                               | Directorio de la copia                        |
| `OPENCLAW_GIT_UPDATE=0\|1`                              | Activa o desactiva actualizaciones git        |
| `OPENCLAW_NO_PROMPT=1`                                  | Desactiva los prompts                         |
| `OPENCLAW_NO_ONBOARD=1`                                 | Omite onboarding                              |
| `OPENCLAW_DRY_RUN=1`                                    | Modo de simulación                            |
| `OPENCLAW_VERBOSE=1`                                    | Modo de depuración                            |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice`             | Nivel de registro de npm                      |
| `SHARP_IGNORE_GLOBAL_LIBVIPS=0\|1`                      | Controla el comportamiento de sharp/libvips (predeterminado: `1`) |

  </Accordion>
</AccordionGroup>

---

<a id="install-clish"></a>

## install-cli.sh

<Info>
Diseñado para entornos en los que quieres que todo viva bajo un prefijo local
(predeterminado `~/.openclaw`) y sin dependencia de Node del sistema. Admite instalaciones con npm
de forma predeterminada, además de instalaciones desde copia git dentro del mismo flujo por prefijo.
</Info>

### Flujo (`install-cli.sh`)

<Steps>
  <Step title="Instalar runtime local de Node">
    Descarga un tarball fijado de una versión LTS compatible de Node (la versión está integrada en el script y se actualiza independientemente) en `<prefix>/tools/node-v<version>` y verifica SHA-256.
  </Step>
  <Step title="Garantizar Git">
    Si Git no está disponible, intenta instalarlo mediante apt/dnf/yum en Linux o Homebrew en macOS.
  </Step>
  <Step title="Instalar OpenClaw bajo el prefijo">
    - Método `npm` (predeterminado): instala bajo el prefijo con npm y luego escribe el wrapper en `<prefix>/bin/openclaw`
    - Método `git`: clona/actualiza una copia (predeterminado `~/openclaw`) y aun así escribe el wrapper en `<prefix>/bin/openclaw`
  </Step>
  <Step title="Actualizar el servicio gateway cargado">
    Si ya hay un servicio gateway cargado desde ese mismo prefijo, el script ejecuta
    `openclaw gateway install --force`, luego `openclaw gateway restart` y
    sondea el estado del gateway en el mejor esfuerzo.
  </Step>
</Steps>

### Ejemplos (`install-cli.sh`)

<Tabs>
  <Tab title="Predeterminado">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```
  </Tab>
  <Tab title="Prefijo personalizado + versión">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --prefix /opt/openclaw --version latest
    ```
  </Tab>
  <Tab title="Instalación con git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --install-method git --git-dir ~/openclaw
    ```
  </Tab>
  <Tab title="Salida JSON para automatización">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="Ejecutar onboarding">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --onboard
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de flags">

| Flag                        | Descripción                                                                     |
| --------------------------- | ------------------------------------------------------------------------------- |
| `--prefix <path>`           | Prefijo de instalación (predeterminado: `~/.openclaw`)                          |
| `--install-method npm\|git` | Elige el método de instalación (predeterminado: `npm`). Alias: `--method`       |
| `--npm`                     | Atajo para el método npm                                                        |
| `--git`, `--github`         | Atajo para el método git                                                        |
| `--git-dir <path>`          | Directorio de la copia git (predeterminado: `~/openclaw`). Alias: `--dir`       |
| `--version <ver>`           | Versión de OpenClaw o dist-tag (predeterminado: `latest`)                       |
| `--node-version <ver>`      | Versión de Node (predeterminado: `22.22.0`)                                     |
| `--json`                    | Emite eventos NDJSON                                                            |
| `--onboard`                 | Ejecuta `openclaw onboard` después de la instalación                            |
| `--no-onboard`              | Omite onboarding (predeterminado)                                               |
| `--set-npm-prefix`          | En Linux, fuerza el prefijo npm a `~/.npm-global` si el prefijo actual no se puede escribir |
| `--help`                    | Muestra el uso (`-h`)                                                           |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                                    | Descripción                                        |
| ------------------------------------------- | -------------------------------------------------- |
| `OPENCLAW_PREFIX=<path>`                    | Prefijo de instalación                             |
| `OPENCLAW_INSTALL_METHOD=git\|npm`          | Método de instalación                              |
| `OPENCLAW_VERSION=<ver>`                    | Versión o dist-tag de OpenClaw                     |
| `OPENCLAW_NODE_VERSION=<ver>`               | Versión de Node                                    |
| `OPENCLAW_GIT_DIR=<path>`                   | Directorio de copia git para instalaciones con git |
| `OPENCLAW_GIT_UPDATE=0\|1`                  | Activa o desactiva actualizaciones git en copias existentes |
| `OPENCLAW_NO_ONBOARD=1`                     | Omite onboarding                                   |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice` | Nivel de registro de npm                           |
| `SHARP_IGNORE_GLOBAL_LIBVIPS=0\|1`          | Controla el comportamiento de sharp/libvips (predeterminado: `1`) |

  </Accordion>
</AccordionGroup>

---

<a id="installps1"></a>

## install.ps1

### Flujo (`install.ps1`)

<Steps>
  <Step title="Garantizar PowerShell + entorno Windows">
    Requiere PowerShell 5+.
  </Step>
  <Step title="Garantizar Node.js 24 de forma predeterminada">
    Si falta, intenta instalarlo mediante winget, luego Chocolatey y después Scoop. Node 22 LTS, actualmente `22.14+`, sigue siendo compatible por compatibilidad.
  </Step>
  <Step title="Instalar OpenClaw">
    - Método `npm` (predeterminado): instalación global con npm usando el `-Tag` seleccionado
    - Método `git`: clona/actualiza el repositorio, instala/compila con pnpm e instala el wrapper en `%USERPROFILE%\.local\bin\openclaw.cmd`
  </Step>
  <Step title="Tareas posteriores a la instalación">
    - Añade el directorio binario necesario al PATH del usuario cuando es posible
    - Actualiza, en el mejor esfuerzo, un servicio gateway cargado (`openclaw gateway install --force`, luego reinicio)
    - Ejecuta `openclaw doctor --non-interactive` en actualizaciones e instalaciones con git (mejor esfuerzo)
  </Step>
</Steps>

### Ejemplos (`install.ps1`)

<Tabs>
  <Tab title="Predeterminado">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
  <Tab title="Instalación con git">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git
    ```
  </Tab>
  <Tab title="GitHub main mediante npm">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag main
    ```
  </Tab>
  <Tab title="Directorio git personalizado">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -GitDir "C:\openclaw"
    ```
  </Tab>
  <Tab title="Simulación">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -DryRun
    ```
  </Tab>
  <Tab title="Traza de depuración">
    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de flags">

| Flag                        | Descripción                                                 |
| --------------------------- | ----------------------------------------------------------- |
| `-InstallMethod npm\|git`   | Método de instalación (predeterminado: `npm`)               |
| `-Tag <tag\|version\|spec>` | Dist-tag, versión o especificación de paquete npm (predeterminado: `latest`) |
| `-GitDir <path>`            | Directorio de la copia (predeterminado: `%USERPROFILE%\openclaw`) |
| `-NoOnboard`                | Omite onboarding                                            |
| `-NoGitUpdate`              | Omite `git pull`                                            |
| `-DryRun`                   | Solo muestra acciones                                       |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                           | Descripción              |
| ---------------------------------- | ------------------------ |
| `OPENCLAW_INSTALL_METHOD=git\|npm` | Método de instalación    |
| `OPENCLAW_GIT_DIR=<path>`          | Directorio de la copia   |
| `OPENCLAW_NO_ONBOARD=1`            | Omite onboarding         |
| `OPENCLAW_GIT_UPDATE=0`            | Desactiva `git pull`     |
| `OPENCLAW_DRY_RUN=1`               | Modo de simulación       |

  </Accordion>
</AccordionGroup>

<Note>
Si se usa `-InstallMethod git` y Git no está disponible, el script sale e imprime el enlace de Git for Windows.
</Note>

---

## CI y automatización

Usa flags/variables de entorno no interactivas para ejecuciones predecibles.

<Tabs>
  <Tab title="install.sh (npm no interactivo)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-prompt --no-onboard
    ```
  </Tab>
  <Tab title="install.sh (git no interactivo)">
    ```bash
    OPENCLAW_INSTALL_METHOD=git OPENCLAW_NO_PROMPT=1 \
      curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="install-cli.sh (JSON)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="install.ps1 (omitir onboarding)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

---

## Resolución de problemas

<AccordionGroup>
  <Accordion title="¿Por qué se requiere Git?">
    Git es necesario para el método de instalación `git`. En instalaciones con `npm`, Git también se comprueba/instala para evitar errores `spawn git ENOENT` cuando las dependencias usan URL de git.
  </Accordion>

  <Accordion title="¿Por qué npm da EACCES en Linux?">
    Algunas configuraciones de Linux apuntan el prefijo global de npm a rutas propiedad de root. `install.sh` puede cambiar el prefijo a `~/.npm-global` y añadir exportaciones PATH a los archivos rc del shell (cuando esos archivos existen).
  </Accordion>

  <Accordion title="Problemas con sharp/libvips">
    Los scripts usan por defecto `SHARP_IGNORE_GLOBAL_LIBVIPS=1` para evitar que sharp se compile contra libvips del sistema. Para sobrescribirlo:

    ```bash
    SHARP_IGNORE_GLOBAL_LIBVIPS=0 curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

  </Accordion>

  <Accordion title='Windows: "npm error spawn git / ENOENT"'>
    Instala Git for Windows, vuelve a abrir PowerShell y ejecuta de nuevo el instalador.
  </Accordion>

  <Accordion title='Windows: "openclaw is not recognized"'>
    Ejecuta `npm config get prefix` y añade ese directorio al PATH de tu usuario (no se necesita sufijo `\bin` en Windows), luego vuelve a abrir PowerShell.
  </Accordion>

  <Accordion title="Windows: cómo obtener salida detallada del instalador">
    `install.ps1` actualmente no expone un interruptor `-Verbose`.
    Usa el trazado de PowerShell para diagnósticos a nivel de script:

    ```powershell
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

  </Accordion>

  <Accordion title="openclaw no aparece después de instalar">
    Normalmente es un problema de PATH. Consulta [Resolución de problemas de Node.js](/install/node#troubleshooting).
  </Accordion>
</AccordionGroup>
