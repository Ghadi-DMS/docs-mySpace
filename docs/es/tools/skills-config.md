---
read_when:
    - Agregar o modificar la configuración de Skills
    - Ajustar la allowlist empaquetada o el comportamiento de instalación
summary: Esquema de configuración de Skills y ejemplos
title: Configuración de Skills
x-i18n:
    generated_at: "2026-04-05T12:56:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7839f39f68c1442dcf4740b09886e0ef55762ce0d4b9f7b4f493a8c130c84579
    source_path: tools/skills-config.md
    workflow: 15
---

# Configuración de Skills

La mayor parte de la configuración del cargador/instalación de Skills vive en `skills` dentro de
`~/.openclaw/openclaw.json`. La visibilidad de Skills específica de cada agente vive en
`agents.defaults.skills` y `agents.list[].skills`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills", "~/Projects/oss/some-skill-pack/skills"],
      watch: true,
      watchDebounceMs: 250,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun (el runtime del Gateway sigue siendo Node; bun no se recomienda)
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // o string de texto sin formato
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

Para la generación/edición de imágenes integrada, prefiere `agents.defaults.imageGenerationModel`
más la herramienta central `image_generate`. `skills.entries.*` es solo para flujos de trabajo de Skills personalizadas o
de terceros.

Si seleccionas un proveedor/modelo de imagen específico, configura también la
autenticación/clave de API de ese proveedor. Ejemplos típicos: `GEMINI_API_KEY` o `GOOGLE_API_KEY` para
`google/*`, `OPENAI_API_KEY` para `openai/*` y `FAL_KEY` para `fal/*`.

Ejemplos:

- Configuración nativa de estilo Nano Banana: `agents.defaults.imageGenerationModel.primary: "google/gemini-3.1-flash-image-preview"`
- Configuración nativa de fal: `agents.defaults.imageGenerationModel.primary: "fal/fal-ai/flux/dev"`

## Allowlists de Skills por agente

Usa la configuración del agente cuando quieras las mismas raíces de Skills de máquina/espacio de trabajo, pero un
conjunto visible de Skills diferente por agente.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // hereda los valores predeterminados -> github, weather
      { id: "docs", skills: ["docs-search"] }, // reemplaza los valores predeterminados
      { id: "locked-down", skills: [] }, // sin Skills
    ],
  },
}
```

Reglas:

- `agents.defaults.skills`: allowlist base compartida para agentes que omiten
  `agents.list[].skills`.
- Omite `agents.defaults.skills` para dejar las Skills sin restricciones de forma predeterminada.
- `agents.list[].skills`: conjunto final explícito de Skills para ese agente; no se
  fusiona con los valores predeterminados.
- `agents.list[].skills: []`: no expone ninguna Skill para ese agente.

## Campos

- Las raíces integradas de Skills siempre incluyen `~/.openclaw/skills`, `~/.agents/skills`,
  `<workspace>/.agents/skills` y `<workspace>/skills`.
- `allowBundled`: allowlist opcional solo para Skills **empaquetadas**. Cuando se configura, solo
  las Skills empaquetadas de la lista son elegibles (las Skills gestionadas, de agente y de espacio de trabajo no se ven afectadas).
- `load.extraDirs`: directorios adicionales de Skills que se deben escanear (precedencia más baja).
- `load.watch`: observa las carpetas de Skills y actualiza la instantánea de Skills (predeterminado: true).
- `load.watchDebounceMs`: debounce para eventos del observador de Skills en milisegundos (predeterminado: 250).
- `install.preferBrew`: prefiere instaladores de brew cuando estén disponibles (predeterminado: true).
- `install.nodeManager`: preferencia de instalador de Node (`npm` | `pnpm` | `yarn` | `bun`, predeterminado: npm).
  Esto solo afecta a las **instalaciones de Skills**; el runtime del Gateway debe seguir siendo Node
  (Bun no se recomienda para WhatsApp/Telegram).
  - `openclaw setup --node-manager` es más limitado y actualmente acepta `npm`,
    `pnpm` o `bun`. Configura manualmente `skills.install.nodeManager: "yarn"` si
    quieres instalaciones de Skills respaldadas por Yarn.
- `entries.<skillKey>`: anulaciones por Skill.
- `agents.defaults.skills`: allowlist predeterminada opcional de Skills heredada por los agentes
  que omiten `agents.list[].skills`.
- `agents.list[].skills`: allowlist final opcional de Skills por agente; las listas explícitas
  reemplazan los valores predeterminados heredados en lugar de fusionarse.

Campos por Skill:

- `enabled`: establécelo en `false` para deshabilitar una Skill aunque esté empaquetada/instalada.
- `env`: variables de entorno inyectadas para la ejecución del agente (solo si aún no están configuradas).
- `apiKey`: comodidad opcional para Skills que declaran una variable de entorno principal.
  Admite string de texto sin formato u objeto SecretRef (`{ source, provider, id }`).

## Notas

- Las claves bajo `entries` se asignan al nombre de la Skill de forma predeterminada. Si una Skill define
  `metadata.openclaw.skillKey`, usa esa clave en su lugar.
- La precedencia de carga es `<workspace>/skills` → `<workspace>/.agents/skills` →
  `~/.agents/skills` → `~/.openclaw/skills` → Skills empaquetadas →
  `skills.load.extraDirs`.
- Los cambios en las Skills se recogen en el siguiente turno del agente cuando el observador está habilitado.

### Skills en sandbox + variables de entorno

Cuando una sesión está **en sandbox**, los procesos de Skills se ejecutan dentro de Docker. El sandbox
**no** hereda el `process.env` del host.

Usa una de estas opciones:

- `agents.defaults.sandbox.docker.env` (o `agents.list[].sandbox.docker.env` por agente)
- incorpora el env en tu imagen personalizada de sandbox

`env` global y `skills.entries.<skill>.env/apiKey` se aplican **solo** a ejecuciones en el host.
