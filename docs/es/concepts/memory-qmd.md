---
read_when:
    - Quieres configurar QMD como tu backend de memoria
    - Quieres funciones avanzadas de memoria como reranking o rutas indexadas adicionales
summary: Sidecar de búsqueda local-first con BM25, vectores, reranking y expansión de consultas
title: Motor de memoria QMD
x-i18n:
    generated_at: "2026-04-05T12:40:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: fa8a31ec1a6cc83b6ab413b7dbed6a88055629251664119bfd84308ed166c58e
    source_path: concepts/memory-qmd.md
    workflow: 15
---

# Motor de memoria QMD

[QMD](https://github.com/tobi/qmd) es un sidecar de búsqueda local-first que se ejecuta
junto a OpenClaw. Combina BM25, búsqueda vectorial y reranking en un único
binario, y puede indexar contenido más allá de los archivos de memoria de tu espacio de trabajo.

## Qué añade frente al motor integrado

- **Reranking y expansión de consultas** para mejorar la recuperación.
- **Indexar directorios adicionales** -- documentación del proyecto, notas del equipo, cualquier cosa en disco.
- **Indexar transcripciones de sesión** -- recuperar conversaciones anteriores.
- **Completamente local** -- se ejecuta mediante Bun + node-llama-cpp, descarga automáticamente modelos GGUF.
- **Fallback automático** -- si QMD no está disponible, OpenClaw vuelve al
  motor integrado sin problemas.

## Primeros pasos

### Requisitos previos

- Instala QMD: `bun install -g @tobilu/qmd`
- Compilación de SQLite que permita extensiones (`brew install sqlite` en macOS).
- QMD debe estar en el `PATH` de la gateway.
- macOS y Linux funcionan de inmediato. Windows tiene mejor compatibilidad mediante WSL2.

### Habilitar

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw crea un directorio base autocontenido de QMD en
`~/.openclaw/agents/<agentId>/qmd/` y gestiona el ciclo de vida del sidecar
automáticamente -- las colecciones, actualizaciones y ejecuciones de embeddings se gestionan por ti.

## Cómo funciona el sidecar

- OpenClaw crea colecciones a partir de los archivos de memoria de tu espacio de trabajo y cualquier
  `memory.qmd.paths` configurado, luego ejecuta `qmd update` + `qmd embed` al arrancar
  y periódicamente (cada 5 minutos de forma predeterminada).
- La actualización de arranque se ejecuta en segundo plano para que el inicio del chat no se bloquee.
- Las búsquedas usan el `searchMode` configurado (predeterminado: `search`; también admite
  `vsearch` y `query`). Si un modo falla, OpenClaw vuelve a intentar con `qmd query`.
- Si QMD falla por completo, OpenClaw vuelve al motor SQLite integrado.

<Info>
La primera búsqueda puede ser lenta -- QMD descarga automáticamente modelos GGUF (~2 GB) para
reranking y expansión de consultas en la primera ejecución de `qmd query`.
</Info>

## Indexar rutas adicionales

Apunta QMD a directorios adicionales para hacerlos buscables:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Los fragmentos de rutas adicionales aparecen como `qmd/<collection>/<relative-path>` en
los resultados de búsqueda. `memory_get` entiende este prefijo y lee desde la raíz
correcta de la colección.

## Indexar transcripciones de sesión

Habilita la indexación de sesiones para recuperar conversaciones anteriores:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Las transcripciones se exportan como turnos saneados de Usuario/Asistente a una colección QMD
dedicada en `~/.openclaw/agents/<id>/qmd/sessions/`.

## Alcance de búsqueda

De forma predeterminada, los resultados de búsqueda de QMD solo aparecen en sesiones de mensajes directos (no en grupos ni
canales). Configura `memory.qmd.scope` para cambiar esto:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Cuando el alcance deniega una búsqueda, OpenClaw registra una advertencia con el canal derivado y
el tipo de chat para que los resultados vacíos sean más fáciles de depurar.

## Citas

Cuando `memory.citations` es `auto` o `on`, los fragmentos de búsqueda incluyen un
pie `Source: <path#line>`. Establece `memory.citations = "off"` para omitir el pie
mientras la ruta sigue pasándose internamente al agente.

## Cuándo usarlo

Elige QMD cuando necesites:

- Reranking para obtener resultados de mayor calidad.
- Buscar documentación del proyecto o notas fuera del espacio de trabajo.
- Recuperar conversaciones de sesiones anteriores.
- Búsqueda completamente local sin API keys.

Para configuraciones más simples, el [motor integrado](/concepts/memory-builtin) funciona bien
sin dependencias adicionales.

## Solución de problemas

**¿QMD no se encuentra?** Asegúrate de que el binario esté en el `PATH` de la gateway. Si OpenClaw
se ejecuta como servicio, crea un symlink:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

**¿La primera búsqueda es muy lenta?** QMD descarga modelos GGUF en el primer uso. Precaliéntalo
con `qmd query "test"` usando los mismos directorios XDG que usa OpenClaw.

**¿La búsqueda agota el tiempo de espera?** Aumenta `memory.qmd.limits.timeoutMs` (predeterminado: 4000ms).
Establécelo en `120000` para hardware más lento.

**¿Resultados vacíos en chats de grupo?** Comprueba `memory.qmd.scope` -- el valor predeterminado solo
permite sesiones de mensajes directos.

**¿Repositorios temporales visibles desde el espacio de trabajo provocan `ENAMETOOLONG` o indexación rota?**
El recorrido de QMD actualmente sigue el comportamiento del escáner QMD subyacente en lugar de
las reglas de symlink integradas de OpenClaw. Mantén los checkouts temporales de monorepo bajo
directorios ocultos como `.tmp/` o fuera de las raíces QMD indexadas hasta que QMD exponga
un recorrido seguro ante ciclos o controles explícitos de exclusión.

## Configuración

Para ver la superficie completa de configuración (`memory.qmd.*`), modos de búsqueda, intervalos
de actualización, reglas de alcance y todos los demás controles, consulta la
[referencia de configuración de memoria](/reference/memory-config).
