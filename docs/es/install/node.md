---
read_when:
    - Necesitas instalar Node.js antes de instalar OpenClaw
    - Instalaste OpenClaw pero `openclaw` no se encuentra como comando
    - '`npm install -g` falla por problemas de permisos o de PATH'
summary: 'Instala y configura Node.js para OpenClaw: requisitos de versión, opciones de instalación y solución de problemas de PATH'
title: Node.js
x-i18n:
    generated_at: "2026-04-05T12:46:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5e880f6132359dba8720638669df2d71cf857d516cbf5df2589ffeed269b5120
    source_path: install/node.md
    workflow: 15
---

# Node.js

OpenClaw requiere **Node 22.14 o posterior**. **Node 24 es el runtime predeterminado y recomendado** para instalaciones, CI y flujos de trabajo de versiones. Node 22 sigue siendo compatible mediante la línea LTS activa. El [script de instalación](/install#alternative-install-methods) detectará e instalará Node automáticamente; esta página es para cuando quieras configurar Node por tu cuenta y asegurarte de que todo esté correctamente conectado (versiones, PATH, instalaciones globales).

## Comprueba tu versión

```bash
node -v
```

Si esto muestra `v24.x.x` o superior, estás usando el valor predeterminado recomendado. Si muestra `v22.14.x` o superior, estás en la ruta compatible de Node 22 LTS, pero aun así recomendamos actualizar a Node 24 cuando te resulte conveniente. Si Node no está instalado o la versión es demasiado antigua, elige uno de los métodos de instalación de abajo.

## Instalar Node

<Tabs>
  <Tab title="macOS">
    **Homebrew** (recomendado):

    ```bash
    brew install node
    ```

    O descarga el instalador de macOS desde [nodejs.org](https://nodejs.org/).

  </Tab>
  <Tab title="Linux">
    **Ubuntu / Debian:**

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

    **Fedora / RHEL:**

    ```bash
    sudo dnf install nodejs
    ```

    O usa un gestor de versiones (consulta abajo).

  </Tab>
  <Tab title="Windows">
    **winget** (recomendado):

    ```powershell
    winget install OpenJS.NodeJS.LTS
    ```

    **Chocolatey:**

    ```powershell
    choco install nodejs-lts
    ```

    O descarga el instalador de Windows desde [nodejs.org](https://nodejs.org/).

  </Tab>
</Tabs>

<Accordion title="Usar un gestor de versiones (nvm, fnm, mise, asdf)">
  Los gestores de versiones te permiten cambiar fácilmente entre versiones de Node. Opciones populares:

- [**fnm**](https://github.com/Schniz/fnm) — rápido, multiplataforma
- [**nvm**](https://github.com/nvm-sh/nvm) — muy usado en macOS/Linux
- [**mise**](https://mise.jdx.dev/) — políglota (Node, Python, Ruby, etc.)

Ejemplo con fnm:

```bash
fnm install 24
fnm use 24
```

  <Warning>
  Asegúrate de que tu gestor de versiones esté inicializado en el archivo de inicio de tu shell (`~/.zshrc` o `~/.bashrc`). Si no lo está, puede que `openclaw` no se encuentre en nuevas sesiones de terminal porque el PATH no incluirá el directorio bin de Node.
  </Warning>
</Accordion>

## Solución de problemas

### `openclaw: command not found`

Casi siempre significa que el directorio bin global de npm no está en tu PATH.

<Steps>
  <Step title="Encuentra tu prefijo global de npm">
    ```bash
    npm prefix -g
    ```
  </Step>
  <Step title="Comprueba si está en tu PATH">
    ```bash
    echo "$PATH"
    ```

    Busca `<npm-prefix>/bin` (macOS/Linux) o `<npm-prefix>` (Windows) en la salida.

  </Step>
  <Step title="Añádelo al archivo de inicio de tu shell">
    <Tabs>
      <Tab title="macOS / Linux">
        Añádelo a `~/.zshrc` o `~/.bashrc`:

        ```bash
        export PATH="$(npm prefix -g)/bin:$PATH"
        ```

        Luego abre una nueva terminal (o ejecuta `rehash` en zsh / `hash -r` en bash).
      </Tab>
      <Tab title="Windows">
        Añade la salida de `npm prefix -g` a tu PATH del sistema mediante Configuración → Sistema → Variables de entorno.
      </Tab>
    </Tabs>

  </Step>
</Steps>

### Errores de permisos en `npm install -g` (Linux)

Si ves errores `EACCES`, cambia el prefijo global de npm a un directorio escribible por el usuario:

```bash
mkdir -p "$HOME/.npm-global"
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
```

Añade la línea `export PATH=...` a tu `~/.bashrc` o `~/.zshrc` para que sea permanente.

## Relacionado

- [Resumen de instalación](/install) — todos los métodos de instalación
- [Updating](/install/updating) — mantener OpenClaw actualizado
- [Primeros pasos](/es/start/getting-started) — primeros pasos después de la instalación
