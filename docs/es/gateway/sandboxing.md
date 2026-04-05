---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
status: active
summary: 'Cómo funciona el sandboxing en OpenClaw: modos, alcances, acceso al espacio de trabajo e imágenes'
title: Sandboxing
x-i18n:
    generated_at: "2026-04-05T12:43:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 756ebd5b9806c23ba720a311df7e3b4ffef6ce41ba4315ee4b36b5ea87b26e60
    source_path: gateway/sandboxing.md
    workflow: 15
---

# Sandboxing

OpenClaw puede ejecutar **herramientas dentro de backends de sandbox** para reducir el radio de impacto.
Esto es **opcional** y se controla mediante la configuración (`agents.defaults.sandbox` o
`agents.list[].sandbox`). Si el sandboxing está desactivado, las herramientas se ejecutan en el host.
El Gateway permanece en el host; la ejecución de herramientas se realiza en un sandbox aislado
cuando está habilitado.

Esto no es un límite de seguridad perfecto, pero limita materialmente el acceso al
sistema de archivos y a procesos cuando el modelo hace algo poco sensato.

## Qué entra en el sandbox

- Ejecución de herramientas (`exec`, `read`, `write`, `edit`, `apply_patch`, `process`, etc.).
- Navegador opcional en sandbox (`agents.defaults.sandbox.browser`).
  - De forma predeterminada, el navegador del sandbox se inicia automáticamente (garantiza que CDP sea accesible) cuando la herramienta del navegador lo necesita.
    Configúralo con `agents.defaults.sandbox.browser.autoStart` y `agents.defaults.sandbox.browser.autoStartTimeoutMs`.
  - De forma predeterminada, los contenedores del navegador del sandbox usan una red Docker dedicada (`openclaw-sandbox-browser`) en lugar de la red global `bridge`.
    Configúralo con `agents.defaults.sandbox.browser.network`.
  - `agents.defaults.sandbox.browser.cdpSourceRange` opcional restringe el ingreso de CDP en el borde del contenedor con una lista de permitidos CIDR (por ejemplo `172.21.0.1/32`).
  - El acceso del observador noVNC está protegido por contraseña de forma predeterminada; OpenClaw emite una URL con token de corta duración que sirve una página local de arranque y abre noVNC con la contraseña en el fragmento de URL (no en logs de consulta/cabecera).
  - `agents.defaults.sandbox.browser.allowHostControl` permite que las sesiones en sandbox apunten explícitamente al navegador del host.
  - Las listas de permitidos opcionales controlan `target: "custom"`: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

No entra en el sandbox:

- El propio proceso Gateway.
- Cualquier herramienta autorizada explícitamente para ejecutarse fuera del sandbox (por ejemplo `tools.elevated`).
  - **Elevated exec omite el sandboxing y usa la ruta de escape configurada (`gateway` de forma predeterminada, o `node` cuando el destino de exec es `node`).**
  - Si el sandboxing está desactivado, `tools.elevated` no cambia la ejecución (ya está en el host). Consulta [Elevated Mode](/tools/elevated).

## Modos

`agents.defaults.sandbox.mode` controla **cuándo** se usa el sandboxing:

- `"off"`: sin sandboxing.
- `"non-main"`: sandbox solo para sesiones **no principales** (predeterminado si quieres chats normales en el host).
- `"all"`: todas las sesiones se ejecutan en un sandbox.
  Nota: `"non-main"` se basa en `session.mainKey` (predeterminado `"main"`), no en el id del agente.
  Las sesiones de grupo/canal usan sus propias claves, por lo que cuentan como no principales y se ejecutarán en sandbox.

## Alcance

`agents.defaults.sandbox.scope` controla **cuántos contenedores** se crean:

- `"agent"` (predeterminado): un contenedor por agente.
- `"session"`: un contenedor por sesión.
- `"shared"`: un contenedor compartido por todas las sesiones en sandbox.

## Backend

`agents.defaults.sandbox.backend` controla **qué entorno de ejecución** proporciona el sandbox:

- `"docker"` (predeterminado): entorno de ejecución de sandbox local respaldado por Docker.
- `"ssh"`: entorno de ejecución remoto genérico respaldado por SSH.
- `"openshell"`: entorno de ejecución de sandbox respaldado por OpenShell.

La configuración específica de SSH se encuentra en `agents.defaults.sandbox.ssh`.
La configuración específica de OpenShell se encuentra en `plugins.entries.openshell.config`.

### Elegir un backend

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **Dónde se ejecuta** | Contenedor local                 | Cualquier host accesible por SSH | Sandbox gestionado por OpenShell                    |
| **Configuración**   | `scripts/sandbox-setup.sh`       | Clave SSH + host de destino    | Plugin de OpenShell habilitado                      |
| **Modelo de espacio de trabajo** | Montaje bind o copia             | Remoto-canónico (inicialización única) | `mirror` o `remote`                                 |
| **Control de red**  | `docker.network` (predeterminado: none) | Depende del host remoto        | Depende de OpenShell                                |
| **Navegador en sandbox** | Compatible                      | No compatible                  | Aún no compatible                                   |
| **Montajes bind**   | `docker.binds`                   | N/A                            | N/A                                                 |
| **Ideal para**      | Desarrollo local, aislamiento completo | Delegar a una máquina remota  | Sandboxes remotos gestionados con sincronización bidireccional opcional |

### Backend SSH

Usa `backend: "ssh"` cuando quieras que OpenClaw ejecute en sandbox `exec`, herramientas de archivos y lecturas de medios en
una máquina arbitraria accesible por SSH.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // Or use SecretRefs / inline contents instead of local files:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Cómo funciona:

- OpenClaw crea una raíz remota por alcance bajo `sandbox.ssh.workspaceRoot`.
- En el primer uso tras crear o recrear, OpenClaw inicializa ese espacio de trabajo remoto a partir del espacio de trabajo local una sola vez.
- Después de eso, `exec`, `read`, `write`, `edit`, `apply_patch`, lecturas de medios en tiempo de prompt y preparación de medios entrantes se ejecutan directamente sobre el espacio de trabajo remoto por SSH.
- OpenClaw no sincroniza automáticamente los cambios remotos de vuelta al espacio de trabajo local.

Material de autenticación:

- `identityFile`, `certificateFile`, `knownHostsFile`: usan archivos locales existentes y los pasan mediante configuración OpenSSH.
- `identityData`, `certificateData`, `knownHostsData`: usan cadenas en línea o SecretRefs. OpenClaw los resuelve mediante la instantánea normal de secretos del entorno de ejecución, los escribe en archivos temporales con `0600` y los elimina cuando termina la sesión SSH.
- Si se configuran tanto `*File` como `*Data` para el mismo elemento, `*Data` tiene prioridad para esa sesión SSH.

Este es un modelo **remoto-canónico**. El espacio de trabajo SSH remoto se convierte en el estado real del sandbox después de la inicialización inicial.

Consecuencias importantes:

- Las ediciones locales del host realizadas fuera de OpenClaw después del paso de inicialización no son visibles en remoto hasta que recrees el sandbox.
- `openclaw sandbox recreate` elimina la raíz remota por alcance y vuelve a inicializar desde local en el siguiente uso.
- El navegador en sandbox no es compatible con el backend SSH.
- La configuración `sandbox.docker.*` no se aplica al backend SSH.

### Backend OpenShell

Usa `backend: "openshell"` cuando quieras que OpenClaw ejecute herramientas en sandbox dentro de un
entorno remoto gestionado por OpenShell. Para ver la guía completa de configuración, la
referencia de configuración y la comparación de modos de espacio de trabajo, consulta la
[página de OpenShell](/gateway/openshell).

OpenShell reutiliza el mismo transporte SSH central y el puente de sistema de archivos remoto que el
backend SSH genérico, y añade el ciclo de vida específico de OpenShell
(`sandbox create/get/delete`, `sandbox ssh-config`) además del modo opcional de espacio de trabajo `mirror`.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote", // mirror | remote
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
        },
      },
    },
  },
}
```

Modos de OpenShell:

- `mirror` (predeterminado): el espacio de trabajo local sigue siendo canónico. OpenClaw sincroniza los archivos locales con OpenShell antes de exec y sincroniza el espacio de trabajo remoto de vuelta después de exec.
- `remote`: el espacio de trabajo de OpenShell es canónico una vez creado el sandbox. OpenClaw inicializa el espacio de trabajo remoto una vez a partir del espacio de trabajo local, y luego las herramientas de archivos y exec se ejecutan directamente sobre el sandbox remoto sin volver a sincronizar cambios.

Detalles del transporte remoto:

- OpenClaw solicita a OpenShell la configuración SSH específica del sandbox mediante `openshell sandbox ssh-config <name>`.
- El núcleo escribe esa configuración SSH en un archivo temporal, abre la sesión SSH y reutiliza el mismo puente de sistema de archivos remoto usado por `backend: "ssh"`.
- Solo en modo `mirror` cambia el ciclo de vida: sincroniza de local a remoto antes de exec y vuelve a sincronizar después de exec.

Limitaciones actuales de OpenShell:

- el navegador del sandbox aún no es compatible
- `sandbox.docker.binds` no es compatible con el backend OpenShell
- los controles de tiempo de ejecución específicos de Docker en `sandbox.docker.*` siguen aplicándose solo al backend Docker

#### Modos de espacio de trabajo

OpenShell tiene dos modelos de espacio de trabajo. Esta es la parte que más importa en la práctica.

##### `mirror`

Usa `plugins.entries.openshell.config.mode: "mirror"` cuando quieras que el **espacio de trabajo local siga siendo canónico**.

Comportamiento:

- Antes de `exec`, OpenClaw sincroniza el espacio de trabajo local con el sandbox de OpenShell.
- Después de `exec`, OpenClaw sincroniza el espacio de trabajo remoto de vuelta al espacio de trabajo local.
- Las herramientas de archivos siguen operando a través del puente del sandbox, pero el espacio de trabajo local sigue siendo la fuente de verdad entre turnos.

Úsalo cuando:

- editas archivos localmente fuera de OpenClaw y quieres que esos cambios aparezcan automáticamente en el sandbox
- quieres que el sandbox de OpenShell se comporte lo más parecido posible al backend Docker
- quieres que el espacio de trabajo del host refleje las escrituras del sandbox después de cada turno de exec

Compensación:

- coste adicional de sincronización antes y después de exec

##### `remote`

Usa `plugins.entries.openshell.config.mode: "remote"` cuando quieras que el **espacio de trabajo de OpenShell pase a ser canónico**.

Comportamiento:

- Cuando se crea el sandbox por primera vez, OpenClaw inicializa el espacio de trabajo remoto a partir del espacio de trabajo local una sola vez.
- Después de eso, `exec`, `read`, `write`, `edit` y `apply_patch` operan directamente sobre el espacio de trabajo remoto de OpenShell.
- OpenClaw **no** sincroniza los cambios remotos de vuelta al espacio de trabajo local después de exec.
- Las lecturas de medios en tiempo de prompt siguen funcionando porque las herramientas de archivos y medios leen a través del puente del sandbox en lugar de asumir una ruta local del host.
- El transporte es SSH hacia el sandbox de OpenShell devuelto por `openshell sandbox ssh-config`.

Consecuencias importantes:

- Si editas archivos en el host fuera de OpenClaw después del paso de inicialización, el sandbox remoto **no** verá esos cambios automáticamente.
- Si el sandbox se recrea, el espacio de trabajo remoto se vuelve a inicializar a partir del espacio de trabajo local.
- Con `scope: "agent"` o `scope: "shared"`, ese espacio de trabajo remoto se comparte en ese mismo alcance.

Úsalo cuando:

- el sandbox deba vivir principalmente en el lado remoto de OpenShell
- quieras una menor sobrecarga de sincronización por turno
- no quieras que las ediciones locales del host sobrescriban silenciosamente el estado remoto del sandbox

Elige `mirror` si piensas en el sandbox como un entorno de ejecución temporal.
Elige `remote` si piensas en el sandbox como el espacio de trabajo real.

#### Ciclo de vida de OpenShell

Los sandboxes de OpenShell siguen gestionándose mediante el ciclo de vida normal del sandbox:

- `openclaw sandbox list` muestra entornos de ejecución de OpenShell además de los de Docker
- `openclaw sandbox recreate` elimina el entorno de ejecución actual y permite que OpenClaw lo recree en el siguiente uso
- la lógica de limpieza también tiene en cuenta el backend

Para el modo `remote`, recreate es especialmente importante:

- recreate elimina el espacio de trabajo remoto canónico para ese alcance
- el siguiente uso inicializa un espacio de trabajo remoto nuevo a partir del espacio de trabajo local

Para el modo `mirror`, recreate principalmente restablece el entorno de ejecución remoto
porque el espacio de trabajo local sigue siendo canónico de todos modos.

## Acceso al espacio de trabajo

`agents.defaults.sandbox.workspaceAccess` controla **qué puede ver** el sandbox:

- `"none"` (predeterminado): las herramientas ven un espacio de trabajo de sandbox bajo `~/.openclaw/sandboxes`.
- `"ro"`: monta el espacio de trabajo del agente en solo lectura en `/agent` (deshabilita `write`/`edit`/`apply_patch`).
- `"rw"`: monta el espacio de trabajo del agente en lectura/escritura en `/workspace`.

Con el backend OpenShell:

- el modo `mirror` sigue usando el espacio de trabajo local como fuente canónica entre turnos de exec
- el modo `remote` usa el espacio de trabajo remoto de OpenShell como fuente canónica después de la inicialización inicial
- `workspaceAccess: "ro"` y `"none"` siguen restringiendo el comportamiento de escritura de la misma manera

Los medios entrantes se copian al espacio de trabajo activo del sandbox (`media/inbound/*`).
Nota sobre Skills: la herramienta `read` tiene como raíz el sandbox. Con `workspaceAccess: "none"`,
OpenClaw refleja los Skills aptos en el espacio de trabajo del sandbox (`.../skills`) para
que puedan leerse. Con `"rw"`, los Skills del espacio de trabajo se pueden leer desde
`/workspace/skills`.

## Montajes bind personalizados

`agents.defaults.sandbox.docker.binds` monta directorios adicionales del host en el contenedor.
Formato: `host:container:mode` (por ejemplo, `"/home/user/source:/source:rw"`).

Los montajes globales y por agente se **fusionan** (no se reemplazan). En `scope: "shared"`, los montajes por agente se ignoran.

`agents.defaults.sandbox.browser.binds` monta directorios adicionales del host solo en el contenedor del **navegador del sandbox**.

- Cuando se configura (incluido `[]`), reemplaza `agents.defaults.sandbox.docker.binds` para el contenedor del navegador.
- Cuando se omite, el contenedor del navegador recurre a `agents.defaults.sandbox.docker.binds` (compatibilidad hacia atrás).

Ejemplo (código fuente de solo lectura + un directorio de datos adicional):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

Notas de seguridad:

- Los montajes bind eluden el sistema de archivos del sandbox: exponen rutas del host con el modo que configures (`:ro` o `:rw`).
- OpenClaw bloquea orígenes de montaje peligrosos (por ejemplo: `docker.sock`, `/etc`, `/proc`, `/sys`, `/dev` y montajes padre que los expondrían).
- OpenClaw también bloquea raíces comunes de credenciales en directorios personales como `~/.aws`, `~/.cargo`, `~/.config`, `~/.docker`, `~/.gnupg`, `~/.netrc`, `~/.npm` y `~/.ssh`.
- La validación de montajes bind no se basa solo en coincidencias de cadenas. OpenClaw normaliza la ruta de origen y luego la resuelve de nuevo a través del ancestro existente más profundo antes de volver a comprobar rutas bloqueadas y raíces permitidas.
- Eso significa que los escapes mediante padres simbólicos siguen fallando de forma segura aunque la hoja final aún no exista. Ejemplo: `/workspace/run-link/new-file` sigue resolviéndose como `/var/run/...` si `run-link` apunta allí.
- Las raíces de origen permitidas se canonizan del mismo modo, por lo que una ruta que solo parece estar dentro de la lista de permitidos antes de la resolución de enlaces simbólicos sigue siendo rechazada como `outside allowed roots`.
- Los montajes sensibles (secretos, claves SSH, credenciales de servicio) deben ser `:ro` salvo que sea absolutamente necesario.
- Combínalo con `workspaceAccess: "ro"` si solo necesitas acceso de lectura al espacio de trabajo; los modos de bind siguen siendo independientes.
- Consulta [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) para ver cómo interactúan los montajes bind con la política de herramientas y elevated exec.

## Imágenes y configuración

Imagen Docker predeterminada: `openclaw-sandbox:bookworm-slim`

Constrúyela una vez:

```bash
scripts/sandbox-setup.sh
```

Nota: la imagen predeterminada **no** incluye Node. Si un Skills necesita Node (u
otros entornos de ejecución), o bien crea una imagen personalizada o instala mediante
`sandbox.docker.setupCommand` (requiere salida de red + raíz con escritura +
usuario root).

Si quieres una imagen de sandbox más funcional con herramientas comunes (por ejemplo
`curl`, `jq`, `nodejs`, `python3`, `git`), crea:

```bash
scripts/sandbox-common-setup.sh
```

Luego configura `agents.defaults.sandbox.docker.image` en
`openclaw-sandbox-common:bookworm-slim`.

Imagen del navegador en sandbox:

```bash
scripts/sandbox-browser-setup.sh
```

De forma predeterminada, los contenedores Docker del sandbox se ejecutan **sin red**.
Anúlalo con `agents.defaults.sandbox.docker.network`.

La imagen incluida del navegador en sandbox también aplica valores predeterminados conservadores para el inicio de Chromium
en cargas de trabajo en contenedores. Los valores predeterminados actuales del contenedor incluyen:

- `--remote-debugging-address=127.0.0.1`
- `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
- `--user-data-dir=${HOME}/.chrome`
- `--no-first-run`
- `--no-default-browser-check`
- `--disable-3d-apis`
- `--disable-gpu`
- `--disable-dev-shm-usage`
- `--disable-background-networking`
- `--disable-extensions`
- `--disable-features=TranslateUI`
- `--disable-breakpad`
- `--disable-crash-reporter`
- `--disable-software-rasterizer`
- `--no-zygote`
- `--metrics-recording-only`
- `--renderer-process-limit=2`
- `--no-sandbox` y `--disable-setuid-sandbox` cuando `noSandbox` está habilitado.
- Los tres indicadores de refuerzo gráfico (`--disable-3d-apis`,
  `--disable-software-rasterizer`, `--disable-gpu`) son opcionales y resultan útiles
  cuando los contenedores no tienen soporte GPU. Configura `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0`
  si tu carga de trabajo requiere WebGL u otras funciones 3D/del navegador.
- `--disable-extensions` está habilitado de forma predeterminada y puede deshabilitarse con
  `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` para flujos que dependen de extensiones.
- `--renderer-process-limit=2` se controla mediante
  `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`, donde `0` mantiene el valor predeterminado de Chromium.

Si necesitas un perfil de ejecución distinto, usa una imagen personalizada del navegador y proporciona
tu propio entrypoint. Para perfiles locales de Chromium (sin contenedor), usa
`browser.extraArgs` para añadir indicadores de inicio adicionales.

Valores predeterminados de seguridad:

- `network: "host"` está bloqueado.
- `network: "container:<id>"` está bloqueado de forma predeterminada (riesgo de omisión por unión de namespace).
- Anulación de emergencia: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

Las instalaciones Docker y el gateway en contenedor se documentan aquí:
[Docker](/install/docker)

Para despliegues del gateway en Docker, `scripts/docker/setup.sh` puede inicializar la configuración del sandbox.
Configura `OPENCLAW_SANDBOX=1` (o `true`/`yes`/`on`) para habilitar esa ruta. Puedes
anular la ubicación del socket con `OPENCLAW_DOCKER_SOCKET`. Referencia completa de
configuración y variables de entorno: [Docker](/install/docker#agent-sandbox).

## setupCommand (configuración única del contenedor)

`setupCommand` se ejecuta **una vez** después de crear el contenedor del sandbox (no en cada ejecución).
Se ejecuta dentro del contenedor mediante `sh -lc`.

Rutas:

- Global: `agents.defaults.sandbox.docker.setupCommand`
- Por agente: `agents.list[].sandbox.docker.setupCommand`

Problemas comunes:

- `docker.network` predeterminado es `"none"` (sin salida), por lo que las instalaciones de paquetes fallarán.
- `docker.network: "container:<id>"` requiere `dangerouslyAllowContainerNamespaceJoin: true` y es solo de emergencia.
- `readOnlyRoot: true` impide escrituras; configura `readOnlyRoot: false` o crea una imagen personalizada.
- `user` debe ser root para instalaciones de paquetes (omite `user` o configura `user: "0:0"`).
- El sandbox exec **no** hereda `process.env` del host. Usa
  `agents.defaults.sandbox.docker.env` (o una imagen personalizada) para las claves de API de Skills.

## Política de herramientas y rutas de escape

Las políticas de permitir/denegar herramientas siguen aplicándose antes de las reglas del sandbox. Si una herramienta está denegada
globalmente o por agente, el sandboxing no la restablece.

`tools.elevated` es una ruta de escape explícita que ejecuta `exec` fuera del sandbox (`gateway` de forma predeterminada, o `node` cuando el destino de exec es `node`).
Las directivas `/exec` solo se aplican a remitentes autorizados y persisten por sesión; para deshabilitar de forma rígida
`exec`, usa la política de herramientas deny (consulta [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)).

Depuración:

- Usa `openclaw sandbox explain` para inspeccionar el modo de sandbox efectivo, la política de herramientas y claves de configuración sugeridas para corregir.
- Consulta [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) para el modelo mental de “¿por qué está bloqueado esto?”.
  Mantenlo restringido.

## Anulaciones para varios agentes

Cada agente puede anular sandbox y herramientas:
`agents.list[].sandbox` y `agents.list[].tools` (además de `agents.list[].tools.sandbox.tools` para política de herramientas del sandbox).
Consulta [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) para ver la precedencia.

## Ejemplo mínimo de habilitación

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## Documentación relacionada

- [OpenShell](/gateway/openshell) -- configuración del backend de sandbox gestionado, modos de espacio de trabajo y referencia de configuración
- [Sandbox Configuration](/gateway/configuration-reference#agentsdefaultssandbox)
- [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) -- depurar “¿por qué está bloqueado esto?”
- [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) -- anulaciones por agente y precedencia
- [Security](/gateway/security)
