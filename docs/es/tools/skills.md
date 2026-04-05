---
read_when:
    - Añadiendo o modificando Skills
    - Cambiando la restricción de Skills o las reglas de carga
summary: 'Skills: gestionadas frente a workspace, reglas de restricción y conexión de config/env'
title: Skills
x-i18n:
    generated_at: "2026-04-05T12:57:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6bb0e2e7c2ff50cf19c759ea1da1fd1886dc11f94adc77cbfd816009f75d93ee
    source_path: tools/skills.md
    workflow: 15
---

# Skills (OpenClaw)

OpenClaw usa carpetas de Skills **compatibles con [AgentSkills](https://agentskills.io)** para enseñar al agente cómo usar herramientas y cuándo hacerlo. Cada Skill es un directorio que contiene un `SKILL.md` con frontmatter YAML e instrucciones. OpenClaw carga **Skills incluidas** además de sobrescrituras locales opcionales, y las filtra en el momento de la carga según el entorno, la configuración y la presencia de binarios.

## Ubicaciones y precedencia

OpenClaw carga Skills desde estas fuentes:

1. **Carpetas de Skills adicionales**: configuradas con `skills.load.extraDirs`
2. **Skills incluidas**: incluidas con la instalación (paquete npm o OpenClaw.app)
3. **Skills gestionadas/locales**: `~/.openclaw/skills`
4. **Skills personales del agente**: `~/.agents/skills`
5. **Skills del agente del proyecto**: `<workspace>/.agents/skills`
6. **Skills del workspace**: `<workspace>/skills`

Si hay un conflicto de nombres de Skills, la precedencia es:

`<workspace>/skills` (más alta) → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → Skills incluidas → `skills.load.extraDirs` (más baja)

## Skills por agente frente a Skills compartidas

En configuraciones **multiagente**, cada agente tiene su propio workspace. Eso significa:

- Las **Skills por agente** viven en `<workspace>/skills` solo para ese agente.
- Las **Skills del agente del proyecto** viven en `<workspace>/.agents/skills` y se aplican a
  ese workspace antes de la carpeta normal `skills/` del workspace.
- Las **Skills personales del agente** viven en `~/.agents/skills` y se aplican en todos los
  workspaces de esa máquina.
- Las **Skills compartidas** viven en `~/.openclaw/skills` (gestionadas/locales) y son visibles
  para **todos los agentes** de la misma máquina.
- También se pueden añadir **carpetas compartidas** mediante `skills.load.extraDirs` (precedencia
  más baja) si quieres un paquete común de Skills usado por varios agentes.

Si el mismo nombre de Skill existe en más de un lugar, se aplica la precedencia habitual:
gana el workspace, luego las Skills del agente del proyecto, después las Skills personales del agente,
luego las gestionadas/locales, luego las incluidas y, por último, los directorios extra.

## Allowlists de Skills por agente

La **ubicación** de una Skill y la **visibilidad** de una Skill son controles separados.

- La ubicación/precedencia decide qué copia gana cuando una Skill con el mismo nombre existe en varios sitios.
- Las allowlists del agente deciden qué Skills visibles puede usar realmente un agente.

Usa `agents.defaults.skills` como referencia compartida y luego sobrescribe por agente con
`agents.list[].skills`:

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // hereda github, weather
      { id: "docs", skills: ["docs-search"] }, // reemplaza los valores predeterminados
      { id: "locked-down", skills: [] }, // sin Skills
    ],
  },
}
```

Reglas:

- Omite `agents.defaults.skills` para permitir Skills sin restricciones de forma predeterminada.
- Omite `agents.list[].skills` para heredar `agents.defaults.skills`.
- Establece `agents.list[].skills: []` para no tener Skills.
- Una lista no vacía en `agents.list[].skills` es el conjunto final para ese agente; no
  se combina con los valores predeterminados.

OpenClaw aplica el conjunto efectivo de Skills del agente en la construcción del prompt, el
descubrimiento de slash commands de Skills, la sincronización del sandbox y las instantáneas de Skills.

## Plugins + Skills

Los plugins pueden incluir sus propias Skills listando directorios `skills` en
`openclaw.plugin.json` (rutas relativas a la raíz del plugin). Las Skills del plugin se cargan
cuando el plugin está habilitado. Actualmente, esos directorios se combinan en la misma ruta
de baja precedencia que `skills.load.extraDirs`, por lo que una Skill incluida, gestionada,
de agente o de workspace con el mismo nombre los sobrescribe.
Puedes restringirlas mediante `metadata.openclaw.requires.config` en la entrada de configuración
del plugin. Consulta [Plugins](/tools/plugin) para descubrimiento/configuración y [Tools](/tools) para la
superficie de herramientas que enseñan esas Skills.

## ClawHub (instalación + sincronización)

ClawHub es el registro público de Skills para OpenClaw. Explóralo en
[https://clawhub.ai](https://clawhub.ai). Usa los comandos nativos `openclaw skills`
para descubrir/instalar/actualizar Skills, o la CLI separada `clawhub` cuando
necesites flujos de publicación/sincronización.
Guía completa: [ClawHub](/tools/clawhub).

Flujos comunes:

- Instalar una Skill en tu workspace:
  - `openclaw skills install <skill-slug>`
- Actualizar todas las Skills instaladas:
  - `openclaw skills update --all`
- Sincronizar (examinar + publicar actualizaciones):
  - `clawhub sync --all`

El comando nativo `openclaw skills install` instala en el directorio `skills/` del workspace activo.
La CLI separada `clawhub` también instala en `./skills` dentro de tu directorio de trabajo
actual (o usa como respaldo el workspace de OpenClaw configurado).
OpenClaw lo detecta como `<workspace>/skills` en la siguiente sesión.

## Notas de seguridad

- Trata las Skills de terceros como **código no confiable**. Léelas antes de habilitarlas.
- Prefiere ejecuciones con sandbox para entradas no confiables y herramientas de riesgo. Consulta [Sandboxing](/es/gateway/sandboxing).
- El descubrimiento de Skills en el workspace y en directorios extra solo acepta raíces de Skills y archivos `SKILL.md` cuyo `realpath` resuelto permanezca dentro de la raíz configurada.
- Las instalaciones de dependencias de Skills respaldadas por Gateway (`skills.install`, incorporación y la UI de ajustes de Skills) ejecutan el escáner integrado de código peligroso antes de ejecutar los metadatos del instalador. Los hallazgos `critical` bloquean de forma predeterminada a menos que quien llama establezca explícitamente la sobrescritura de código peligroso; los hallazgos sospechosos siguen mostrando solo advertencias.
- `openclaw skills install <slug>` es diferente: descarga una carpeta de Skill de ClawHub en el workspace y no usa la ruta de metadatos del instalador anterior.
- `skills.entries.*.env` y `skills.entries.*.apiKey` inyectan secretos en el proceso **host**
  para ese turno del agente (no en el sandbox). Mantén los secretos fuera de los prompts y los logs.
- Para un modelo de amenazas y listas de comprobación más amplios, consulta [Security](/es/gateway/security).

## Formato (compatible con AgentSkills + Pi)

`SKILL.md` debe incluir al menos:

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
---
```

Notas:

- Seguimos la especificación de AgentSkills para diseño/intención.
- El parser usado por el agente integrado solo admite claves de frontmatter de **una sola línea**.
- `metadata` debe ser un **objeto JSON de una sola línea**.
- Usa `{baseDir}` en las instrucciones para referirte a la ruta de la carpeta de la Skill.
- Claves opcionales de frontmatter:
  - `homepage` — URL mostrada como “Website” en la UI de Skills de macOS (también admitida mediante `metadata.openclaw.homepage`).
  - `user-invocable` — `true|false` (predeterminado: `true`). Cuando es `true`, la Skill se expone como un slash command de usuario.
  - `disable-model-invocation` — `true|false` (predeterminado: `false`). Cuando es `true`, la Skill se excluye del prompt del modelo (sigue disponible mediante invocación del usuario).
  - `command-dispatch` — `tool` (opcional). Cuando se establece en `tool`, el slash command omite el modelo y se envía directamente a una herramienta.
  - `command-tool` — nombre de la herramienta que se invocará cuando `command-dispatch: tool` esté establecido.
  - `command-arg-mode` — `raw` (predeterminado). Para el envío a herramientas, reenvía la cadena de argumentos sin procesar a la herramienta (sin análisis del núcleo).

    La herramienta se invoca con los parámetros:
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.

## Restricción (filtros en el momento de carga)

OpenClaw **filtra Skills en el momento de la carga** usando `metadata` (JSON de una sola línea):

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

Campos en `metadata.openclaw`:

- `always: true` — incluir siempre la Skill (omite las demás restricciones).
- `emoji` — emoji opcional usado por la UI de Skills de macOS.
- `homepage` — URL opcional mostrada como “Website” en la UI de Skills de macOS.
- `os` — lista opcional de plataformas (`darwin`, `linux`, `win32`). Si se establece, la Skill solo es apta en esos SO.
- `requires.bins` — lista; cada elemento debe existir en `PATH`.
- `requires.anyBins` — lista; al menos uno debe existir en `PATH`.
- `requires.env` — lista; la variable de entorno debe existir **o** proporcionarse en la configuración.
- `requires.config` — lista de rutas de `openclaw.json` que deben ser verdaderas.
- `primaryEnv` — nombre de la variable de entorno asociada a `skills.entries.<name>.apiKey`.
- `install` — array opcional de especificaciones de instalación usadas por la UI de Skills de macOS (brew/node/go/uv/download).

Nota sobre sandboxing:

- `requires.bins` se comprueba en el **host** en el momento de carga de la Skill.
- Si un agente está en sandbox, el binario también debe existir **dentro del contenedor**.
  Instálalo mediante `agents.defaults.sandbox.docker.setupCommand` (o una imagen personalizada).
  `setupCommand` se ejecuta una vez después de crear el contenedor.
  Las instalaciones de paquetes también requieren salida de red, un sistema de archivos raíz escribible y un usuario root en el sandbox.
  Ejemplo: la Skill `summarize` (`skills/summarize/SKILL.md`) necesita la CLI `summarize`
  en el contenedor del sandbox para ejecutarse allí.

Ejemplo de instalador:

```markdown
---
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Install Gemini CLI (brew)",
            },
          ],
      },
  }
---
```

Notas:

- Si se enumeran varios instaladores, el gateway elige una única opción **preferida** (brew cuando está disponible; de lo contrario, node).
- Si todos los instaladores son `download`, OpenClaw enumera cada entrada para que puedas ver los artefactos disponibles.
- Las especificaciones del instalador pueden incluir `os: ["darwin"|"linux"|"win32"]` para filtrar opciones por plataforma.
- Las instalaciones con Node respetan `skills.install.nodeManager` en `openclaw.json` (predeterminado: npm; opciones: npm/pnpm/yarn/bun).
  Esto solo afecta a las **instalaciones de Skills**; el runtime del Gateway debe seguir siendo Node
  (Bun no se recomienda para WhatsApp/Telegram).
- La selección de instaladores respaldada por Gateway se basa en preferencias, no solo en node:
  cuando las especificaciones de instalación mezclan tipos, OpenClaw prefiere Homebrew cuando
  `skills.install.preferBrew` está habilitado y `brew` existe; después `uv`, luego el
  administrador de node configurado y, después, otros respaldos como `go` o `download`.
- Si todas las especificaciones de instalación son `download`, OpenClaw muestra todas las opciones de descarga
  en lugar de contraerlas en un único instalador preferido.
- Instalaciones con Go: si falta `go` y `brew` está disponible, el gateway instala primero Go mediante Homebrew y establece `GOBIN` en el `bin` de Homebrew cuando es posible.
- Instalaciones con descarga: `url` (obligatorio), `archive` (`tar.gz` | `tar.bz2` | `zip`), `extract` (predeterminado: automático cuando se detecta un archivo), `stripComponents`, `targetDir` (predeterminado: `~/.openclaw/tools/<skillKey>`).

Si no existe `metadata.openclaw`, la Skill siempre es apta (a menos
que esté deshabilitada en la configuración o bloqueada por `skills.allowBundled` para Skills incluidas).

## Sobrescrituras de configuración (`~/.openclaw/openclaw.json`)

Las Skills incluidas/gestionadas pueden activarse o desactivarse y recibir valores de entorno:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // o cadena de texto plano
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

Nota: si el nombre de la Skill contiene guiones, encierra la clave entre comillas (JSON5 permite claves entre comillas).

Si quieres generación/edición de imágenes estándar dentro del propio OpenClaw, usa la
herramienta central `image_generate` con `agents.defaults.imageGenerationModel` en lugar de una
Skill incluida. Los ejemplos de Skills aquí son para flujos personalizados o de terceros.

Para análisis nativo de imágenes, usa la herramienta `image` con `agents.defaults.imageModel`.
Para generación/edición nativa de imágenes, usa `image_generate` con
`agents.defaults.imageGenerationModel`. Si eliges `openai/*`, `google/*`,
`fal/*` u otro modelo de imagen específico del proveedor, añade también la autenticación/clave API de ese proveedor.

Las claves de configuración coinciden con el **nombre de la Skill** de forma predeterminada. Si una Skill define
`metadata.openclaw.skillKey`, usa esa clave en `skills.entries`.

Reglas:

- `enabled: false` deshabilita la Skill aunque esté incluida/instalada.
- `env`: se inyecta **solo si** la variable todavía no está establecida en el proceso.
- `apiKey`: comodidad para Skills que declaran `metadata.openclaw.primaryEnv`.
  Admite una cadena de texto plano o un objeto SecretRef (`{ source, provider, id }`).
- `config`: contenedor opcional para campos personalizados por Skill; las claves personalizadas deben vivir aquí.
- `allowBundled`: allowlist opcional solo para Skills **incluidas**. Si se establece, solo
  las Skills incluidas de la lista son aptas (las Skills gestionadas/del workspace no se ven afectadas).

## Inyección de entorno (por ejecución del agente)

Cuando comienza una ejecución de agente, OpenClaw:

1. Lee los metadatos de la Skill.
2. Aplica cualquier `skills.entries.<key>.env` o `skills.entries.<key>.apiKey` a
   `process.env`.
3. Construye el prompt del sistema con las Skills **aptas**.
4. Restaura el entorno original cuando termina la ejecución.

Esto está **limitado a la ejecución del agente**, no a un entorno global de shell.

## Instantánea de sesión (rendimiento)

OpenClaw toma una instantánea de las Skills aptas **cuando empieza una sesión** y reutiliza esa lista en los turnos siguientes de la misma sesión. Los cambios en las Skills o en la configuración se aplican en la siguiente sesión nueva.

Las Skills también pueden actualizarse a mitad de sesión cuando el watcher de Skills está habilitado o cuando aparece un nuevo nodo remoto apto (consulta más abajo). Piensa en esto como una **recarga en caliente**: la lista actualizada se recoge en el siguiente turno del agente.

Si la allowlist efectiva de Skills del agente cambia para esa sesión, OpenClaw
actualiza la instantánea para que las Skills visibles permanezcan alineadas con el
agente actual.

## Nodos remotos de macOS (Gateway en Linux)

Si el Gateway se ejecuta en Linux pero hay un **nodo macOS** conectado **con `system.run` permitido** (la seguridad de Exec approvals no está establecida en `deny`), OpenClaw puede tratar las Skills exclusivas de macOS como aptas cuando los binarios requeridos están presentes en ese nodo. El agente debe ejecutar esas Skills mediante la herramienta `exec` con `host=node`.

Esto depende de que el nodo informe su compatibilidad con comandos y de una comprobación de binarios mediante `system.run`. Si el nodo macOS se desconecta más tarde, las Skills permanecen visibles; las invocaciones pueden fallar hasta que el nodo se vuelva a conectar.

## Watcher de Skills (actualización automática)

De forma predeterminada, OpenClaw observa las carpetas de Skills y aumenta la instantánea de Skills cuando cambian archivos `SKILL.md`. Configúralo en `skills.load`:

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250,
    },
  },
}
```

## Impacto en tokens (lista de Skills)

Cuando hay Skills aptas, OpenClaw inyecta una lista XML compacta de Skills disponibles en el prompt del sistema (mediante `formatSkillsForPrompt` en `pi-coding-agent`). El coste es determinista:

- **Sobrecarga base (solo cuando hay ≥1 Skill):** 195 caracteres.
- **Por Skill:** 97 caracteres + la longitud de los valores XML-escaped de `<name>`, `<description>` y `<location>`.

Fórmula (caracteres):

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

Notas:

- El escape XML expande `& < > " '` en entidades (`&amp;`, `&lt;`, etc.), lo que aumenta la longitud.
- El recuento de tokens varía según el tokenizador del modelo. Una estimación aproximada de estilo OpenAI es ~4 caracteres/token, así que **97 caracteres ≈ 24 tokens** por Skill, además de la longitud real de tus campos.

## Ciclo de vida de las Skills gestionadas

OpenClaw incluye un conjunto base de Skills como **Skills incluidas** como parte de la
instalación (paquete npm o OpenClaw.app). `~/.openclaw/skills` existe para sobrescrituras locales
(por ejemplo, fijar/parchar una Skill sin cambiar la copia incluida).
Las Skills del workspace son propiedad del usuario y sobrescriben ambas cuando hay conflictos de nombre.

## Referencia de configuración

Consulta [Skills config](/tools/skills-config) para ver el esquema completo de configuración.

## ¿Buscas más Skills?

Explora [https://clawhub.ai](https://clawhub.ai).

---

## Relacionado

- [Creating Skills](/tools/creating-skills) — crear Skills personalizadas
- [Skills Config](/tools/skills-config) — referencia de configuración de Skills
- [Slash Commands](/tools/slash-commands) — todos los slash commands disponibles
- [Plugins](/tools/plugin) — descripción general del sistema de plugins
