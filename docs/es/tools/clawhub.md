---
read_when:
    - Presentar ClawHub a usuarios nuevos
    - Instalar, buscar o publicar skills o plugins
    - Explicar las banderas de la CLI de ClawHub y el comportamiento de sincronización
summary: 'Guía de ClawHub: registro público, flujos de instalación nativos de OpenClaw y flujos de trabajo de la CLI de ClawHub'
title: ClawHub
x-i18n:
    generated_at: "2026-04-05T12:55:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: e65b3fd770ca96a5dd828dce2dee4ef127268f4884180a912f43d7744bc5706f
    source_path: tools/clawhub.md
    workflow: 15
---

# ClawHub

ClawHub es el registro público de **skills y plugins de OpenClaw**.

- Usa comandos nativos de `openclaw` para buscar/instalar/actualizar skills e instalar plugins desde ClawHub.
- Usa la CLI independiente `clawhub` cuando necesites autenticación del registro, publicación, eliminación, restauración o flujos de trabajo de sincronización.

Sitio: [clawhub.ai](https://clawhub.ai)

## Flujos nativos de OpenClaw

Skills:

```bash
openclaw skills search "calendar"
openclaw skills install <skill-slug>
openclaw skills update --all
```

Plugins:

```bash
openclaw plugins install clawhub:<package>
openclaw plugins update --all
```

Las especificaciones de plugins simples compatibles con npm también se prueban contra ClawHub antes que contra npm:

```bash
openclaw plugins install openclaw-codex-app-server
```

Los comandos nativos de `openclaw` instalan en tu espacio de trabajo activo y conservan metadatos de origen para que las llamadas posteriores a `update` puedan seguir en ClawHub.

Las instalaciones de plugins validan la compatibilidad anunciada de `pluginApi` y `minGatewayVersion` antes de ejecutar la instalación del archivo, de modo que los hosts incompatibles fallen de forma cerrada al principio en lugar de instalar parcialmente el paquete.

`openclaw plugins install clawhub:...` solo acepta familias de plugins instalables. Si un paquete de ClawHub es en realidad una skill, OpenClaw se detiene y te indica que uses `openclaw skills install <slug>` en su lugar.

## Qué es ClawHub

- Un registro público para skills y plugins de OpenClaw.
- Un almacén versionado de paquetes de skills y metadatos.
- Una superficie de descubrimiento para búsqueda, etiquetas y señales de uso.

## Cómo funciona

1. Un usuario publica un paquete de skill (archivos + metadatos).
2. ClawHub almacena el paquete, analiza los metadatos y asigna una versión.
3. El registro indexa la skill para búsqueda y descubrimiento.
4. Los usuarios exploran, descargan e instalan skills en OpenClaw.

## Qué puedes hacer

- Publicar skills nuevas y nuevas versiones de skills existentes.
- Descubrir skills por nombre, etiquetas o búsqueda.
- Descargar paquetes de skills e inspeccionar sus archivos.
- Reportar skills abusivas o inseguras.
- Si eres moderador, ocultar, mostrar, eliminar o bloquear.

## Para quién es esto (apto para principiantes)

Si quieres añadir nuevas capacidades a tu agente de OpenClaw, ClawHub es la forma más fácil de encontrar e instalar skills. No necesitas saber cómo funciona el backend. Puedes:

- Buscar skills con lenguaje natural.
- Instalar una skill en tu espacio de trabajo.
- Actualizar skills más adelante con un solo comando.
- Hacer copia de seguridad de tus propias skills publicándolas.

## Inicio rápido (no técnico)

1. Busca algo que necesites:
   - `openclaw skills search "calendar"`
2. Instala una skill:
   - `openclaw skills install <skill-slug>`
3. Inicia una nueva sesión de OpenClaw para que detecte la nueva skill.
4. Si quieres publicar o gestionar la autenticación del registro, instala también la CLI independiente `clawhub`.

## Instalar la CLI de ClawHub

Solo necesitas esto para flujos de trabajo autenticados en el registro, como publicar/sincronizar:

```bash
npm i -g clawhub
```

```bash
pnpm add -g clawhub
```

## Cómo encaja en OpenClaw

El comando nativo `openclaw skills install` instala en el directorio `skills/` del espacio de trabajo activo. `openclaw plugins install clawhub:...` registra una instalación normal gestionada del plugin además de los metadatos de origen de ClawHub para actualizaciones.

Las instalaciones anónimas de plugins desde ClawHub también fallan de forma cerrada para paquetes privados. Los canales comunitarios u otros canales no oficiales aún pueden instalarse, pero OpenClaw muestra una advertencia para que los operadores puedan revisar el origen y la verificación antes de habilitarlos.

La CLI independiente `clawhub` también instala skills en `./skills` bajo tu directorio de trabajo actual. Si hay un espacio de trabajo de OpenClaw configurado, `clawhub` recurre a ese espacio de trabajo a menos que sobrescribas `--workdir` (o `CLAWHUB_WORKDIR`). OpenClaw carga las skills del espacio de trabajo desde `<workspace>/skills` y las detectará en la **siguiente** sesión. Si ya usas `~/.openclaw/skills` o skills incluidas, las skills del espacio de trabajo tienen prioridad.

Para más detalles sobre cómo se cargan, comparten y restringen las skills, consulta [Skills](/tools/skills).

## Descripción general del sistema de skills

Una skill es un paquete versionado de archivos que enseña a OpenClaw cómo realizar una tarea específica. Cada publicación crea una nueva versión, y el registro conserva un historial de versiones para que los usuarios puedan auditar los cambios.

Una skill típica incluye:

- Un archivo `SKILL.md` con la descripción principal y el uso.
- Configuraciones, scripts o archivos de apoyo opcionales usados por la skill.
- Metadatos como etiquetas, resumen y requisitos de instalación.

ClawHub usa metadatos para impulsar el descubrimiento y exponer de forma segura las capacidades de las skills. El registro también rastrea señales de uso (como estrellas y descargas) para mejorar la clasificación y la visibilidad.

## Qué proporciona el servicio (funciones)

- **Navegación pública** de skills y de su contenido `SKILL.md`.
- **Búsqueda** impulsada por embeddings (búsqueda vectorial), no solo por palabras clave.
- **Versionado** con semver, changelogs y etiquetas (incluida `latest`).
- **Descargas** como zip por versión.
- **Estrellas y comentarios** para comentarios de la comunidad.
- **Hooks de moderación** para aprobaciones y auditorías.
- **API compatible con CLI** para automatización y scripting.

## Seguridad y moderación

ClawHub es abierto por defecto. Cualquiera puede subir skills, pero una cuenta de GitHub debe tener al menos una semana de antigüedad para poder publicar. Esto ayuda a frenar los abusos sin bloquear a los colaboradores legítimos.

Reportes y moderación:

- Cualquier usuario con sesión iniciada puede reportar una skill.
- Los motivos del reporte son obligatorios y se registran.
- Cada usuario puede tener hasta 20 reportes activos a la vez.
- Las skills con más de 3 reportes únicos se ocultan automáticamente por defecto.
- Los moderadores pueden ver skills ocultas, volver a mostrarlas, eliminarlas o bloquear usuarios.
- El abuso de la función de reporte puede dar lugar al bloqueo de la cuenta.

¿Te interesa convertirte en moderador? Pregunta en el Discord de OpenClaw y contacta con un moderador o mantenedor.

## Comandos y parámetros de la CLI

Opciones globales (se aplican a todos los comandos):

- `--workdir <dir>`: directorio de trabajo (por defecto: directorio actual; recurre al espacio de trabajo de OpenClaw).
- `--dir <dir>`: directorio de skills, relativo a workdir (por defecto: `skills`).
- `--site <url>`: URL base del sitio (inicio de sesión en navegador).
- `--registry <url>`: URL base de la API del registro.
- `--no-input`: desactivar prompts (no interactivo).
- `-V, --cli-version`: imprimir la versión de la CLI.

Autenticación:

- `clawhub login` (flujo en navegador) o `clawhub login --token <token>`
- `clawhub logout`
- `clawhub whoami`

Opciones:

- `--token <token>`: pega un token de API.
- `--label <label>`: etiqueta almacenada para tokens de inicio de sesión en navegador (por defecto: `CLI token`).
- `--no-browser`: no abrir un navegador (requiere `--token`).

Búsqueda:

- `clawhub search "query"`
- `--limit <n>`: máximo de resultados.

Instalación:

- `clawhub install <slug>`
- `--version <version>`: instalar una versión específica.
- `--force`: sobrescribir si la carpeta ya existe.

Actualización:

- `clawhub update <slug>`
- `clawhub update --all`
- `--version <version>`: actualizar a una versión específica (solo un slug).
- `--force`: sobrescribir cuando los archivos locales no coincidan con ninguna versión publicada.

Lista:

- `clawhub list` (lee `.clawhub/lock.json`)

Publicar skills:

- `clawhub skill publish <path>`
- `--slug <slug>`: slug de la skill.
- `--name <name>`: nombre para mostrar.
- `--version <version>`: versión semver.
- `--changelog <text>`: texto del changelog (puede estar vacío).
- `--tags <tags>`: etiquetas separadas por comas (por defecto: `latest`).

Publicar plugins:

- `clawhub package publish <source>`
- `<source>` puede ser una carpeta local, `owner/repo`, `owner/repo@ref` o una URL de GitHub.
- `--dry-run`: construir el plan exacto de publicación sin subir nada.
- `--json`: emitir salida legible por máquina para CI.
- `--source-repo`, `--source-commit`, `--source-ref`: sobrescrituras opcionales cuando la detección automática no es suficiente.

Eliminar/restaurar (solo propietario/admin):

- `clawhub delete <slug> --yes`
- `clawhub undelete <slug> --yes`

Sincronizar (explorar skills locales + publicar nuevas/actualizadas):

- `clawhub sync`
- `--root <dir...>`: raíces de exploración adicionales.
- `--all`: subir todo sin prompts.
- `--dry-run`: mostrar qué se subiría.
- `--bump <type>`: `patch|minor|major` para actualizaciones (por defecto: `patch`).
- `--changelog <text>`: changelog para actualizaciones no interactivas.
- `--tags <tags>`: etiquetas separadas por comas (por defecto: `latest`).
- `--concurrency <n>`: comprobaciones del registro (por defecto: 4).

## Flujos de trabajo comunes para agentes

### Buscar skills

```bash
clawhub search "postgres backups"
```

### Descargar skills nuevas

```bash
clawhub install my-skill-pack
```

### Actualizar skills instaladas

```bash
clawhub update --all
```

### Hacer copia de seguridad de tus skills (publicar o sincronizar)

Para una sola carpeta de skill:

```bash
clawhub skill publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0 --tags latest
```

Para explorar y hacer copia de seguridad de muchas skills a la vez:

```bash
clawhub sync --all
```

### Publicar un plugin desde GitHub

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
clawhub package publish https://github.com/your-org/your-plugin
```

Los plugins de código deben incluir los metadatos obligatorios de OpenClaw en `package.json`:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

## Detalles avanzados (técnicos)

### Versionado y etiquetas

- Cada publicación crea una nueva `SkillVersion` de **semver**.
- Las etiquetas (como `latest`) apuntan a una versión; mover etiquetas te permite revertir.
- Los changelogs se adjuntan por versión y pueden estar vacíos al sincronizar o publicar actualizaciones.

### Cambios locales frente a versiones del registro

Las actualizaciones comparan el contenido local de la skill con las versiones del registro mediante un hash de contenido. Si los archivos locales no coinciden con ninguna versión publicada, la CLI pregunta antes de sobrescribir (o requiere `--force` en ejecuciones no interactivas).

### Exploración de sincronización y raíces de respaldo

`clawhub sync` explora primero tu workdir actual. Si no encuentra skills, recurre a ubicaciones heredadas conocidas (por ejemplo `~/openclaw/skills` y `~/.openclaw/skills`). Esto está diseñado para encontrar instalaciones antiguas de skills sin necesidad de banderas adicionales.

### Almacenamiento y lockfile

- Las skills instaladas se registran en `.clawhub/lock.json` bajo tu workdir.
- Los tokens de autenticación se almacenan en el archivo de configuración de la CLI de ClawHub (puedes sobrescribirlo mediante `CLAWHUB_CONFIG_PATH`).

### Telemetría (recuento de instalaciones)

Cuando ejecutas `clawhub sync` con la sesión iniciada, la CLI envía una instantánea mínima para calcular el recuento de instalaciones. Puedes desactivar esto por completo:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

## Variables de entorno

- `CLAWHUB_SITE`: sobrescribir la URL del sitio.
- `CLAWHUB_REGISTRY`: sobrescribir la URL de la API del registro.
- `CLAWHUB_CONFIG_PATH`: sobrescribir dónde almacena la CLI el token/la configuración.
- `CLAWHUB_WORKDIR`: sobrescribir el workdir predeterminado.
- `CLAWHUB_DISABLE_TELEMETRY=1`: desactivar la telemetría en `sync`.
