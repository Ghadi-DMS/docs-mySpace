---
read_when:
    - Quieres ejecutar o escribir flujos de trabajo `.prose`
    - Quieres habilitar el plugin OpenProse
    - Necesitas entender el almacenamiento del estado
summary: 'OpenProse: flujos de trabajo `.prose`, comandos slash y estado en OpenClaw'
title: OpenProse
x-i18n:
    generated_at: "2026-04-05T12:50:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 95f86ed3029c5599b6a6bed1f75b2e10c8808cf7ffa5e33dbfb1801a7f65f405
    source_path: prose.md
    workflow: 15
---

# OpenProse

OpenProse es un formato portátil de flujos de trabajo basado primero en Markdown para orquestar sesiones de IA. En OpenClaw se incluye como un plugin que instala un paquete de Skills de OpenProse junto con un comando slash `/prose`. Los programas residen en archivos `.prose` y pueden generar varios subagentes con control explícito del flujo.

Sitio oficial: [https://www.prose.md](https://www.prose.md)

## Qué puede hacer

- Investigación + síntesis multiagente con paralelismo explícito.
- Flujos de trabajo repetibles y seguros para aprobación (revisión de código, triaje de incidentes, canalizaciones de contenido).
- Programas `.prose` reutilizables que puedes ejecutar en runtimes de agentes compatibles.

## Instalar + habilitar

Los plugins integrados están deshabilitados de forma predeterminada. Habilita OpenProse:

```bash
openclaw plugins enable open-prose
```

Reinicia el Gateway después de habilitar el plugin.

Desarrollo/copia local: `openclaw plugins install ./path/to/local/open-prose-plugin`

Documentación relacionada: [Plugins](/tools/plugin), [Plugin manifest](/plugins/manifest), [Skills](/tools/skills).

## Comando slash

OpenProse registra `/prose` como un comando de Skills invocable por el usuario. Se enruta a las instrucciones de la VM de OpenProse y usa herramientas de OpenClaw internamente.

Comandos comunes:

```
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

## Ejemplo: un archivo `.prose` sencillo

```prose
# Research + synthesis with two agents running in parallel.

input topic: "What should we research?"

agent researcher:
  model: sonnet
  prompt: "You research thoroughly and cite sources."

agent writer:
  model: opus
  prompt: "You write a concise summary."

parallel:
  findings = session: researcher
    prompt: "Research {topic}."
  draft = session: writer
    prompt: "Summarize {topic}."

session "Merge the findings + draft into a final answer."
context: { findings, draft }
```

## Ubicaciones de archivos

OpenProse mantiene el estado en `.prose/` dentro de tu espacio de trabajo:

```
.prose/
├── .env
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose
│       ├── state.md
│       ├── bindings/
│       └── agents/
└── agents/
```

Los agentes persistentes a nivel de usuario se encuentran en:

```
~/.prose/agents/
```

## Modos de estado

OpenProse admite varios backends de estado:

- **filesystem** (predeterminado): `.prose/runs/...`
- **in-context**: transitorio, para programas pequeños
- **sqlite** (experimental): requiere el binario `sqlite3`
- **postgres** (experimental): requiere `psql` y una cadena de conexión

Notas:

- sqlite/postgres son opcionales y experimentales.
- Las credenciales de postgres fluyen a los registros de subagentes; usa una base de datos dedicada con privilegios mínimos.

## Programas remotos

`/prose run <handle/slug>` se resuelve a `https://p.prose.md/<handle>/<slug>`.
Las URL directas se obtienen tal cual. Esto usa la herramienta `web_fetch` (o `exec` para POST).

## Asignación al runtime de OpenClaw

Los programas de OpenProse se asignan a primitivas de OpenClaw:

| Concepto de OpenProse      | Herramienta de OpenClaw |
| -------------------------- | ----------------------- |
| Generar sesión / herramienta Task | `sessions_spawn` |
| Leer/escribir archivos     | `read` / `write` |
| Obtención web              | `web_fetch`      |

Si tu lista de permitidos de herramientas bloquea estas herramientas, los programas de OpenProse fallarán. Consulta [Configuración de Skills](/tools/skills-config).

## Seguridad + aprobaciones

Trata los archivos `.prose` como código. Revísalos antes de ejecutarlos. Usa las listas de permitidos de herramientas y las puertas de aprobación de OpenClaw para controlar efectos secundarios.

Para flujos de trabajo deterministas y controlados por aprobación, compáralo con [Lobster](/tools/lobster).
