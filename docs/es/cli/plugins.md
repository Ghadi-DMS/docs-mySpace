---
read_when:
    - Quieres instalar o gestionar plugins del Gateway o paquetes compatibles
    - Quieres depurar errores de carga de plugins
summary: Referencia de CLI para `openclaw plugins` (listar, instalar, marketplace, desinstalar, habilitar/deshabilitar, doctor)
title: plugins
x-i18n:
    generated_at: "2026-04-05T12:39:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8c35ccf68cd7be1af5fee175bd1ce7de88b81c625a05a23887e5780e790df925
    source_path: cli/plugins.md
    workflow: 15
---

# `openclaw plugins`

Gestiona plugins/extensiones del Gateway, paquetes de hooks y paquetes compatibles.

Relacionado:

- Sistema de plugins: [Plugins](/tools/plugin)
- Compatibilidad de paquetes: [Paquetes de plugins](/plugins/bundles)
- Manifiesto + esquema del plugin: [Manifiesto del plugin](/plugins/manifest)
- Endurecimiento de seguridad: [Seguridad](/gateway/security)

## Comandos

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins install <path-or-spec>
openclaw plugins inspect <id>
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
openclaw plugins info <id>
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id>
openclaw plugins doctor
openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins marketplace list <marketplace>
openclaw plugins marketplace list <marketplace> --json
```

Los plugins empaquetados se incluyen con OpenClaw. Algunos están habilitados de forma predeterminada (por ejemplo,
proveedores de modelos empaquetados, proveedores de voz empaquetados y el plugin de navegador
empaquetado); otros requieren `plugins enable`.

Los plugins nativos de OpenClaw deben incluir `openclaw.plugin.json` con un esquema JSON
inline (`configSchema`, incluso si está vacío). Los paquetes compatibles usan sus propios
manifiestos de paquete.

`plugins list` muestra `Format: openclaw` o `Format: bundle`. La salida detallada de list/info
también muestra el subtipo de paquete (`codex`, `claude` o `cursor`) más las capacidades del paquete
detectadas.

### Instalar

```bash
openclaw plugins install <package>                      # ClawHub first, then npm
openclaw plugins install clawhub:<package>              # ClawHub only
openclaw plugins install <package> --force              # overwrite existing install
openclaw plugins install <package> --pin                # pin version
openclaw plugins install <package> --dangerously-force-unsafe-install
openclaw plugins install <path>                         # local path
openclaw plugins install <plugin>@<marketplace>         # marketplace
openclaw plugins install <plugin> --marketplace <name>  # marketplace (explicit)
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
```

Los nombres de paquete sin prefijo se comprueban primero en ClawHub y luego en npm. Nota de seguridad:
trata las instalaciones de plugins como ejecución de código. Prefiere versiones fijadas.

Si la configuración no es válida, `plugins install` normalmente falla en modo cerrado y te indica que
ejecutes primero `openclaw doctor --fix`. La única excepción documentada es una ruta estrecha de recuperación
de plugins empaquetados para plugins que optan explícitamente por
`openclaw.install.allowInvalidConfigRecovery`.

`--force` reutiliza el destino de instalación existente y sobrescribe en sitio un
plugin o paquete de hooks ya instalado. Úsalo cuando reinstales intencionadamente
el mismo id desde una nueva ruta local, archivo, paquete de ClawHub o artefacto de npm.

`--pin` se aplica solo a instalaciones de npm. No es compatible con `--marketplace`,
porque las instalaciones desde marketplace persisten metadatos del origen del marketplace en lugar de una
especificación de npm.

`--dangerously-force-unsafe-install` es una opción de emergencia para falsos positivos
en el analizador integrado de código peligroso. Permite que la instalación continúe incluso
cuando el analizador integrado informa hallazgos `critical`, pero **no** omite los bloqueos
de política del hook `before_install` del plugin ni omite fallos del análisis.

Este flag de CLI se aplica a los flujos de instalación/actualización de plugins. Las
instalaciones de dependencias de Skills respaldadas por Gateway usan la sobrescritura de solicitud
correspondiente `dangerouslyForceUnsafeInstall`, mientras que `openclaw skills install` sigue siendo un flujo
independiente de descarga/instalación de Skills desde ClawHub.

`plugins install` también es la superficie de instalación para paquetes de hooks que exponen
`openclaw.hooks` en `package.json`. Usa `openclaw hooks` para visibilidad filtrada de hooks
y habilitación por hook, no para instalación de paquetes.

Las especificaciones de npm son **solo de registro** (nombre de paquete + **versión exacta** opcional o
**dist-tag**). Se rechazan especificaciones git/URL/archivo y rangos semver. Las instalaciones de dependencias
se ejecutan con `--ignore-scripts` por seguridad.

Las especificaciones sin prefijo y `@latest` se mantienen en la vía estable. Si npm resuelve cualquiera
de ellas a una versión preliminar, OpenClaw se detiene y te pide que optes explícitamente por ella con una
etiqueta preliminar como `@beta`/`@rc` o una versión preliminar exacta como
`@1.2.3-beta.4`.

Si una especificación de instalación sin prefijo coincide con el id de un plugin empaquetado (por ejemplo `diffs`), OpenClaw
instala directamente el plugin empaquetado. Para instalar un paquete npm con el mismo
nombre, usa una especificación con scope explícito (por ejemplo `@scope/diffs`).

Archivos compatibles: `.zip`, `.tgz`, `.tar.gz`, `.tar`.

También se admiten instalaciones desde el marketplace de Claude.

Las instalaciones desde ClawHub usan un localizador explícito `clawhub:<package>`:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

OpenClaw ahora también prefiere ClawHub para especificaciones de plugins seguras para npm sin prefijo. Solo
recurre a npm si ClawHub no tiene ese paquete o versión:

```bash
openclaw plugins install openclaw-codex-app-server
```

OpenClaw descarga el archivo del paquete desde ClawHub, comprueba la compatibilidad anunciada
de la API del plugin / gateway mínimo y luego lo instala a través de la ruta normal
de archivos. Las instalaciones registradas conservan sus metadatos de origen de ClawHub para actualizaciones posteriores.

Usa la forma abreviada `plugin@marketplace` cuando el nombre del marketplace exista en la caché
del registro local de Claude en `~/.claude/plugins/known_marketplaces.json`:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Usa `--marketplace` cuando quieras pasar explícitamente el origen del marketplace:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

Los orígenes del marketplace pueden ser:

- un nombre de marketplace conocido por Claude de `~/.claude/plugins/known_marketplaces.json`
- una raíz local del marketplace o una ruta a `marketplace.json`
- una forma abreviada de repositorio de GitHub como `owner/repo`
- una URL de repositorio de GitHub como `https://github.com/owner/repo`
- una URL git

Para marketplaces remotos cargados desde GitHub o git, las entradas de plugins deben permanecer
dentro del repositorio clonado del marketplace. OpenClaw acepta orígenes de ruta relativos de
ese repositorio y rechaza orígenes de plugins HTTP(S), de ruta absoluta, git, GitHub y otros orígenes que no sean rutas en manifiestos remotos.

Para rutas locales y archivos, OpenClaw detecta automáticamente:

- plugins nativos de OpenClaw (`openclaw.plugin.json`)
- paquetes compatibles con Codex (`.codex-plugin/plugin.json`)
- paquetes compatibles con Claude (`.claude-plugin/plugin.json` o el diseño predeterminado
  de componentes de Claude)
- paquetes compatibles con Cursor (`.cursor-plugin/plugin.json`)

Los paquetes compatibles se instalan en la raíz normal de extensiones y participan en el
mismo flujo de listado/info/habilitación/deshabilitación. Hoy, los bundle Skills, las
command-skills de Claude, los valores predeterminados de Claude `settings.json`, los valores predeterminados de Claude `.lsp.json` /
`lspServers` declarados en el manifiesto, las command-skills de Cursor y los directorios
de hooks compatibles con Codex son compatibles; otras capacidades detectadas de paquetes se
muestran en diagnósticos/info, pero todavía no están conectadas a la ejecución en tiempo de ejecución.

### Listar

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

Usa `--enabled` para mostrar solo plugins cargados. Usa `--verbose` para cambiar de la
vista de tabla a líneas de detalle por plugin con metadatos de origen/procedencia/versión/activación.
Usa `--json` para inventario legible por máquina más diagnósticos
del registro.

Usa `--link` para evitar copiar un directorio local (lo añade a `plugins.load.paths`):

```bash
openclaw plugins install -l ./my-plugin
```

`--force` no es compatible con `--link` porque las instalaciones enlazadas reutilizan la
ruta de origen en lugar de copiar sobre un destino de instalación gestionado.

Usa `--pin` en instalaciones de npm para guardar la especificación exacta resuelta (`name@version`) en
`plugins.installs` mientras mantienes el comportamiento predeterminado sin fijar.

### Desinstalar

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
```

`uninstall` elimina los registros del plugin de `plugins.entries`, `plugins.installs`,
la lista de permitidos de plugins y las entradas enlazadas de `plugins.load.paths` cuando corresponda.
Para los plugins de memoria activos, la ranura de memoria se restablece a `memory-core`.

De forma predeterminada, la desinstalación también elimina el directorio de instalación del plugin bajo la
raíz de plugins del directorio de estado activo. Usa
`--keep-files` para mantener los archivos en disco.

`--keep-config` se admite como alias obsoleto de `--keep-files`.

### Actualizar

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call@beta
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

Las actualizaciones se aplican a instalaciones rastreadas en `plugins.installs` y a instalaciones
rastreadas de paquetes de hooks en `hooks.internal.installs`.

Cuando pasas un id de plugin, OpenClaw reutiliza la especificación de instalación registrada para ese
plugin. Eso significa que las dist-tags almacenadas previamente, como `@beta`, y las versiones exactas
fijadas siguen usándose en ejecuciones posteriores de `update <id>`.

Para instalaciones de npm, también puedes pasar una especificación explícita de paquete npm con una dist-tag
o versión exacta. OpenClaw resuelve ese nombre de paquete de vuelta al registro rastreado del plugin,
actualiza ese plugin instalado y registra la nueva especificación npm para futuras
actualizaciones basadas en id.

Cuando existe un hash de integridad almacenado y cambia el hash del artefacto recuperado,
OpenClaw imprime una advertencia y pide confirmación antes de continuar. Usa el
`--yes` global para omitir prompts en ejecuciones de CI/no interactivas.

`--dangerously-force-unsafe-install` también está disponible en `plugins update` como una
sobrescritura de emergencia para falsos positivos del análisis integrado de código peligroso durante
actualizaciones de plugins. Aun así, no omite los bloqueos de política `before_install` del plugin
ni el bloqueo por fallo del análisis, y solo se aplica a actualizaciones de plugins, no a
actualizaciones de paquetes de hooks.

### Inspeccionar

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --json
```

Introspección profunda de un único plugin. Muestra identidad, estado de carga, origen,
capacidades registradas, hooks, herramientas, comandos, servicios, métodos de gateway,
rutas HTTP, flags de política, diagnósticos, metadatos de instalación, capacidades de paquetes
y cualquier soporte detectado de servidor MCP o LSP.

Cada plugin se clasifica según lo que realmente registra en tiempo de ejecución:

- **plain-capability** — un tipo de capacidad (por ejemplo, un plugin solo de proveedor)
- **hybrid-capability** — varios tipos de capacidad (por ejemplo, texto + voz + imágenes)
- **hook-only** — solo hooks, sin capacidades ni superficies
- **non-capability** — herramientas/comandos/servicios pero sin capacidades

Consulta [Formas de plugins](/plugins/architecture#plugin-shapes) para obtener más información sobre el modelo de capacidades.

El flag `--json` produce un informe legible por máquina apto para scripting y
auditoría.

`inspect --all` renderiza una tabla de toda la flota con columnas de forma, tipos de
capacidad, avisos de compatibilidad, capacidades de paquetes y resumen de hooks.

`info` es un alias de `inspect`.

### Doctor

```bash
openclaw plugins doctor
```

`doctor` informa errores de carga de plugins, diagnósticos de manifiesto/detección y
avisos de compatibilidad. Cuando todo está limpio imprime `No plugin issues
detected.`

### Marketplace

```bash
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
```

La lista de marketplace acepta una ruta local al marketplace, una ruta a `marketplace.json`, una
forma abreviada de GitHub como `owner/repo`, una URL de repositorio de GitHub o una URL git. `--json`
imprime la etiqueta del origen resuelto más el manifiesto del marketplace analizado y las
entradas de plugins.
