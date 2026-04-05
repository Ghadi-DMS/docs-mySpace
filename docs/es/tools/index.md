---
read_when:
    - Quieres entender qué herramientas proporciona OpenClaw
    - Necesitas configurar, permitir o denegar herramientas
    - Estás decidiendo entre herramientas integradas, Skills y plugins
summary: 'Resumen de herramientas y plugins de OpenClaw: qué puede hacer el agente y cómo ampliarlo'
title: Herramientas y plugins
x-i18n:
    generated_at: "2026-04-05T12:56:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 17768048b23f980de5e502cc30fbddbadc2e26ae62f0f03c5ab5bbcdeea67e50
    source_path: tools/index.md
    workflow: 15
---

# Herramientas y plugins

Todo lo que hace el agente más allá de generar texto ocurre mediante **herramientas**.
Las herramientas son la forma en que el agente lee archivos, ejecuta comandos, navega por la web, envía
mensajes e interactúa con dispositivos.

## Herramientas, Skills y plugins

OpenClaw tiene tres capas que funcionan juntas:

<Steps>
  <Step title="Las herramientas son lo que invoca el agente">
    Una herramienta es una función tipada que el agente puede invocar (por ejemplo, `exec`, `browser`,
    `web_search`, `message`). OpenClaw incluye un conjunto de **herramientas integradas** y
    los plugins pueden registrar otras adicionales.

    El agente ve las herramientas como definiciones estructuradas de funciones enviadas a la API del modelo.

  </Step>

  <Step title="Las Skills enseñan al agente cuándo y cómo">
    Una Skill es un archivo Markdown (`SKILL.md`) inyectado en el prompt del sistema.
    Las Skills proporcionan al agente contexto, restricciones y orientación paso a paso para
    usar las herramientas de forma eficaz. Las Skills viven en tu espacio de trabajo, en carpetas compartidas,
    o se incluyen dentro de plugins.

    [Referencia de Skills](/tools/skills) | [Crear Skills](/tools/creating-skills)

  </Step>

  <Step title="Los plugins lo empaquetan todo junto">
    Un plugin es un paquete que puede registrar cualquier combinación de capacidades:
    canales, proveedores de modelos, herramientas, Skills, voz, transcripción en tiempo real,
    voz en tiempo real, comprensión multimedia, generación de imágenes, generación de vídeo,
    web fetch, web search y más. Algunos plugins son **core** (incluidos con
    OpenClaw), otros son **externos** (publicados en npm por la comunidad).

    [Instalar y configurar plugins](/tools/plugin) | [Crear el tuyo](/es/plugins/building-plugins)

  </Step>
</Steps>

## Herramientas integradas

Estas herramientas se incluyen con OpenClaw y están disponibles sin instalar ningún plugin:

| Tool                                       | Qué hace                                                              | Página                                  |
| ------------------------------------------ | --------------------------------------------------------------------- | --------------------------------------- |
| `exec` / `process`                         | Ejecuta comandos de shell, gestiona procesos en segundo plano         | [Exec](/tools/exec)                     |
| `code_execution`                           | Ejecuta análisis remoto de Python en entorno aislado                  | [Code Execution](/tools/code-execution) |
| `browser`                                  | Controla un navegador Chromium (navegar, hacer clic, capturar pantalla) | [Browser](/tools/browser)               |
| `web_search` / `x_search` / `web_fetch`    | Busca en la web, busca publicaciones en X, obtiene contenido de páginas | [Web](/tools/web)                       |
| `read` / `write` / `edit`                  | E/S de archivos en el espacio de trabajo                              |                                         |
| `apply_patch`                              | Parches de archivos con varios hunks                                  | [Apply Patch](/tools/apply-patch)       |
| `message`                                  | Envía mensajes a través de todos los canales                          | [Agent Send](/tools/agent-send)         |
| `canvas`                                   | Controla Canvas del nodo (present, eval, snapshot)                    |                                         |
| `nodes`                                    | Descubre y selecciona dispositivos emparejados                        |                                         |
| `cron` / `gateway`                         | Gestiona trabajos programados; inspecciona, corrige, reinicia o actualiza el gateway |                                         |
| `image` / `image_generate`                 | Analiza o genera imágenes                                             |                                         |
| `tts`                                      | Conversión puntual de texto a voz                                     | [TTS](/tools/tts)                       |
| `sessions_*` / `subagents` / `agents_list` | Gestión de sesiones, estado y orquestación de subagentes              | [Sub-agents](/tools/subagents)          |
| `session_status`                           | Lectura ligera tipo `/status` y anulación del modelo por sesión       | [Session Tools](/es/concepts/session-tool) |

Para el trabajo con imágenes, usa `image` para análisis e `image_generate` para generación o edición. Si apuntas a `openai/*`, `google/*`, `fal/*` u otro proveedor de imágenes no predeterminado, configura primero la autenticación/API key de ese proveedor.

`session_status` es la herramienta ligera de estado/lectura del grupo de sesiones.
Responde preguntas de estilo `/status` sobre la sesión actual y puede
establecer opcionalmente una anulación del modelo por sesión; `model=default` borra esa
anulación. Al igual que `/status`, puede rellenar contadores escasos de tokens/caché y la
etiqueta del modelo activo de runtime a partir de la entrada de uso de transcripción más reciente.

`gateway` es la herramienta de runtime solo para propietarios para operaciones del gateway:

- `config.schema.lookup` para un subárbol de configuración acotado a una ruta antes de editar
- `config.get` para la instantánea de configuración actual + hash
- `config.patch` para actualizaciones parciales de configuración con reinicio
- `config.apply` solo para reemplazo completo de la configuración
- `update.run` para autoactualización explícita + reinicio

Para cambios parciales, prefiere `config.schema.lookup` y luego `config.patch`. Usa
`config.apply` solo cuando pretendas reemplazar toda la configuración.
La herramienta también se niega a cambiar `tools.exec.ask` o `tools.exec.security`;
los alias heredados `tools.bash.*` se normalizan a las mismas rutas exec protegidas.

### Herramientas proporcionadas por plugins

Los plugins pueden registrar herramientas adicionales. Algunos ejemplos:

- [Lobster](/tools/lobster) — runtime de flujo de trabajo tipado con aprobaciones reanudables
- [LLM Task](/tools/llm-task) — paso de LLM solo JSON para salida estructurada
- [Diffs](/tools/diffs) — visor y renderizador de diffs
- [OpenProse](/es/prose) — orquestación de flujo de trabajo centrada en Markdown

## Configuración de herramientas

### Listas de permitidos y denegados

Controla qué herramientas puede invocar el agente mediante `tools.allow` / `tools.deny` en
la configuración. Denegar siempre prevalece sobre permitir.

```json5
{
  tools: {
    allow: ["group:fs", "browser", "web_search"],
    deny: ["exec"],
  },
}
```

### Perfiles de herramientas

`tools.profile` establece una lista base de permitidos antes de aplicar `allow`/`deny`.
Anulación por agente: `agents.list[].tools.profile`.

| Perfil      | Qué incluye                                                                                                 |
| ----------- | ----------------------------------------------------------------------------------------------------------- |
| `full`      | Sin restricciones (igual que no establecerlo)                                                               |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                  |
| `minimal`   | Solo `session_status`                                                                                       |

### Grupos de herramientas

Usa atajos `group:*` en listas de permitidos/denegados:

| Grupo              | Herramientas                                                                                               |
| ------------------ | ---------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | exec, process, code_execution (`bash` se acepta como alias de `exec`)                                      |
| `group:fs`         | read, write, edit, apply_patch                                                                             |
| `group:sessions`   | sessions_list, sessions_history, sessions_send, sessions_spawn, sessions_yield, subagents, session_status |
| `group:memory`     | memory_search, memory_get                                                                                  |
| `group:web`        | web_search, x_search, web_fetch                                                                            |
| `group:ui`         | browser, canvas                                                                                            |
| `group:automation` | cron, gateway                                                                                              |
| `group:messaging`  | message                                                                                                    |
| `group:nodes`      | nodes                                                                                                      |
| `group:agents`     | agents_list                                                                                                |
| `group:media`      | image, image_generate, tts                                                                                 |
| `group:openclaw`   | Todas las herramientas integradas de OpenClaw (excluye herramientas de plugins)                           |

`sessions_history` devuelve una vista de recuperación acotada y filtrada por seguridad. Elimina
etiquetas de thinking, andamiaje de `<relevant-memories>`, cargas útiles XML de llamadas a herramientas en texto plano
(incluyendo `<tool_call>...</tool_call>`,
`<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
`<function_calls>...</function_calls>`, y bloques de llamadas a herramientas truncados),
andamiaje degradado de llamadas a herramientas, tokens de control del modelo filtrados en ASCII/ancho completo,
y XML malformado de llamadas a herramientas de MiniMax del texto del asistente, y luego aplica
redacción/truncamiento y posibles marcadores de fila sobredimensionada en lugar de actuar
como un volcado de transcripción sin procesar.

### Restricciones específicas del proveedor

Usa `tools.byProvider` para restringir herramientas para proveedores específicos sin
cambiar los valores predeterminados globales:

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
    },
  },
}
```
