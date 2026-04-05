---
read_when:
    - Quieres entender cómo funciona la memoria
    - Quieres saber qué archivos de memoria escribir
summary: Cómo OpenClaw recuerda cosas entre sesiones
title: Resumen de memoria
x-i18n:
    generated_at: "2026-04-05T12:40:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 89fbd20cf2bcdf461a9e311ee0ff43b5f69d9953519656eecd419b4a419256f8
    source_path: concepts/memory.md
    workflow: 15
---

# Resumen de memoria

OpenClaw recuerda cosas escribiendo **archivos Markdown sin formato** en el
espacio de trabajo de tu agente. El modelo solo "recuerda" lo que se guarda en
disco; no existe estado oculto.

## Cómo funciona

Tu agente tiene dos lugares para almacenar recuerdos:

- **`MEMORY.md`** -- memoria a largo plazo. Hechos duraderos, preferencias y
  decisiones. Se carga al inicio de cada sesión de MD.
- **`memory/YYYY-MM-DD.md`** -- notas diarias. Contexto continuo y observaciones.
  Las notas de hoy y de ayer se cargan automáticamente.

Estos archivos se encuentran en el espacio de trabajo del agente (predeterminado `~/.openclaw/workspace`).

<Tip>
Si quieres que tu agente recuerde algo, simplemente pídeselo: "Recuerda que
prefiero TypeScript." Lo escribirá en el archivo adecuado.
</Tip>

## Herramientas de memoria

El agente tiene dos herramientas para trabajar con memoria:

- **`memory_search`** -- encuentra notas relevantes usando búsqueda semántica, incluso cuando
  la redacción difiere de la original.
- **`memory_get`** -- lee un archivo de memoria específico o un intervalo de líneas.

Ambas herramientas las proporciona el plugin de memoria activo (predeterminado: `memory-core`).

## Búsqueda de memoria

Cuando hay configurado un proveedor de embeddings, `memory_search` usa **búsqueda
híbrida**: combina similitud vectorial (significado semántico) con coincidencia por palabras clave
(términos exactos como IDs y símbolos de código). Esto funciona de inmediato una vez que tienes
una clave de API para cualquier proveedor compatible.

<Info>
OpenClaw detecta automáticamente tu proveedor de embeddings a partir de las claves de API disponibles. Si
tienes configurada una clave de OpenAI, Gemini, Voyage o Mistral, la búsqueda de memoria se
habilita automáticamente.
</Info>

Para obtener detalles sobre cómo funciona la búsqueda, opciones de ajuste y configuración del proveedor, consulta
[Búsqueda de memoria](/concepts/memory-search).

## Backends de memoria

<CardGroup cols={3}>
<Card title="Integrado (predeterminado)" icon="database" href="/concepts/memory-builtin">
Basado en SQLite. Funciona de inmediato con búsqueda por palabras clave, similitud vectorial y
búsqueda híbrida. Sin dependencias adicionales.
</Card>
<Card title="QMD" icon="search" href="/concepts/memory-qmd">
Sidecar local-first con reranking, expansión de consultas y capacidad para indexar
directorios fuera del espacio de trabajo.
</Card>
<Card title="Honcho" icon="brain" href="/concepts/memory-honcho">
Memoria nativa de IA entre sesiones con modelado de usuarios, búsqueda semántica y
conciencia multiagente. Requiere instalación del plugin.
</Card>
</CardGroup>

## Vaciado automático de memoria

Antes de que la [compactación](/concepts/compaction) resuma tu conversación, OpenClaw
ejecuta un turno silencioso que recuerda al agente guardar el contexto importante en archivos
de memoria. Esto está activado de forma predeterminada; no necesitas configurar nada.

<Tip>
El vaciado de memoria evita la pérdida de contexto durante la compactación. Si tu agente tiene
hechos importantes en la conversación que todavía no se han escrito en un archivo, se
guardarán automáticamente antes de que ocurra el resumen.
</Tip>

## Dreaming (experimental)

Dreaming es una pasada opcional de consolidación en segundo plano para la memoria. Revisa
los recuerdos a corto plazo de archivos diarios (`memory/YYYY-MM-DD.md`), los puntúa y
promueve solo los elementos que cumplen los requisitos a la memoria a largo plazo (`MEMORY.md`).

Está diseñado para mantener una memoria a largo plazo de alta señal:

- **Opt-in**: desactivado de forma predeterminada.
- **Programado**: cuando está habilitado, `memory-core` gestiona la tarea recurrente
  automáticamente.
- **Con umbrales**: las promociones deben superar umbrales de puntuación, frecuencia de recuerdo y diversidad de consultas.

Para el comportamiento por modo (`off`, `core`, `rem`, `deep`), señales de puntuación y parámetros
de ajuste, consulta [Dreaming (experimental)](/concepts/memory-dreaming).

## CLI

```bash
openclaw memory status          # Comprobar el estado del índice y el proveedor
openclaw memory search "query"  # Buscar desde la línea de comandos
openclaw memory index --force   # Reconstruir el índice
```

## Lecturas adicionales

- [Motor de memoria integrado](/concepts/memory-builtin) -- backend SQLite predeterminado
- [Motor de memoria QMD](/concepts/memory-qmd) -- sidecar local-first avanzado
- [Memoria Honcho](/concepts/memory-honcho) -- memoria nativa de IA entre sesiones
- [Búsqueda de memoria](/concepts/memory-search) -- canalización de búsqueda, proveedores y
  ajuste
- [Dreaming (experimental)](/concepts/memory-dreaming) -- promoción en segundo plano
  de recuerdo a corto plazo a memoria a largo plazo
- [Referencia de configuración de memoria](/reference/memory-config) -- todos los parámetros de configuración
- [Compactación](/concepts/compaction) -- cómo interactúa la compactación con la memoria
