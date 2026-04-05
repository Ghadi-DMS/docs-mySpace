---
read_when:
    - Quieres configurar proveedores de búsqueda de memoria o modelos de embeddings
    - Quieres configurar el backend de QMD
    - Quieres ajustar la búsqueda híbrida, MMR o el decaimiento temporal
    - Quieres habilitar la indexación de memoria multimodal
summary: Todos los parámetros de configuración para la búsqueda de memoria, los proveedores de embeddings, QMD, la búsqueda híbrida y la indexación multimodal
title: Referencia de configuración de memoria
x-i18n:
    generated_at: "2026-04-05T12:53:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 89e4c9740f71f5a47fc5e163742339362d6b95cb4757650c0c8a095cf3078caa
    source_path: reference/memory-config.md
    workflow: 15
---

# Referencia de configuración de memoria

Esta página enumera todos los parámetros de configuración para la búsqueda de memoria de OpenClaw. Para descripciones conceptuales generales, consulta:

- [Descripción general de la memoria](/es/concepts/memory) -- cómo funciona la memoria
- [Motor integrado](/es/concepts/memory-builtin) -- backend SQLite predeterminado
- [Motor QMD](/es/concepts/memory-qmd) -- sidecar local-first
- [Búsqueda de memoria](/es/concepts/memory-search) -- canalización de búsqueda y ajuste

Todos los ajustes de búsqueda de memoria se encuentran en `agents.defaults.memorySearch` en
`openclaw.json`, a menos que se indique lo contrario.

---

## Selección de proveedor

| Key        | Type      | Default          | Description                                                                      |
| ---------- | --------- | ---------------- | -------------------------------------------------------------------------------- |
| `provider` | `string`  | detectado automáticamente | ID del adaptador de embeddings: `openai`, `gemini`, `voyage`, `mistral`, `ollama`, `local` |
| `model`    | `string`  | predeterminado del proveedor | Nombre del modelo de embeddings                                                             |
| `fallback` | `string`  | `"none"`         | ID del adaptador de respaldo cuando falla el principal                                       |
| `enabled`  | `boolean` | `true`           | Habilita o deshabilita la búsqueda de memoria                                                  |

### Orden de detección automática

Cuando `provider` no está configurado, OpenClaw selecciona el primero disponible:

1. `local` -- si `memorySearch.local.modelPath` está configurado y el archivo existe.
2. `openai` -- si se puede resolver una clave de OpenAI.
3. `gemini` -- si se puede resolver una clave de Gemini.
4. `voyage` -- si se puede resolver una clave de Voyage.
5. `mistral` -- si se puede resolver una clave de Mistral.

`ollama` es compatible, pero no se detecta automáticamente (configúralo explícitamente).

### Resolución de clave de API

Los embeddings remotos requieren una clave de API. OpenClaw la resuelve a partir de:
perfiles de autenticación, `models.providers.*.apiKey` o variables de entorno.

| Provider | Env var                        | Config key                        |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Ollama   | `OLLAMA_API_KEY` (marcador de posición) | --                                |

Codex OAuth cubre solo chat/completions y no satisface las solicitudes
de embeddings.

---

## Configuración de endpoint remoto

Para endpoints personalizados compatibles con OpenAI o para sobrescribir los valores predeterminados del proveedor:

| Key              | Type     | Description                                        |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | URL base de API personalizada                                |
| `remote.apiKey`  | `string` | Sobrescribe la clave de API                                   |
| `remote.headers` | `object` | Encabezados HTTP adicionales (combinados con los valores predeterminados del proveedor) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Configuración específica de Gemini

| Key                    | Type     | Default                | Description                                |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | También admite `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | Para Embedding 2: 768, 1536 o 3072        |

<Warning>
Cambiar el modelo o `outputDimensionality` desencadena una reindexación completa automática.
</Warning>

---

## Configuración de embeddings locales

| Key                   | Type     | Default                | Description                     |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | descargado automáticamente        | Ruta al archivo de modelo GGUF         |
| `local.modelCacheDir` | `string` | predeterminado de node-llama-cpp | Directorio de caché para modelos descargados |

Modelo predeterminado: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB, se descarga automáticamente).
Requiere compilación nativa: `pnpm approve-builds` y luego `pnpm rebuild node-llama-cpp`.

---

## Configuración de búsqueda híbrida

Todo en `memorySearch.query.hybrid`:

| Key                   | Type      | Default | Description                        |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | Habilita la búsqueda híbrida BM25 + vectorial |
| `vectorWeight`        | `number`  | `0.7`   | Peso para las puntuaciones vectoriales (0-1)     |
| `textWeight`          | `number`  | `0.3`   | Peso para las puntuaciones BM25 (0-1)       |
| `candidateMultiplier` | `number`  | `4`     | Multiplicador del tamaño del conjunto de candidatos     |

### MMR (diversidad)

| Key           | Type      | Default | Description                          |
| ------------- | --------- | ------- | ------------------------------------ |
| `mmr.enabled` | `boolean` | `false` | Habilita la reclasificación con MMR                |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = máxima diversidad, 1 = máxima relevancia |

### Decaimiento temporal (recencia)

| Key                          | Type      | Default | Description               |
| ---------------------------- | --------- | ------- | ------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | Habilita el refuerzo por recencia      |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | La puntuación se reduce a la mitad cada N días |

Los archivos perennes (`MEMORY.md`, archivos sin fecha en `memory/`) nunca se degradan.

### Ejemplo completo

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## Rutas de memoria adicionales

| Key          | Type       | Description                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | Directorios o archivos adicionales para indexar |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

Las rutas pueden ser absolutas o relativas al espacio de trabajo. Los directorios se analizan
de forma recursiva en busca de archivos `.md`. El manejo de enlaces simbólicos depende del backend activo:
el motor integrado ignora los enlaces simbólicos, mientras que QMD sigue el comportamiento
del escáner QMD subyacente.

Para la búsqueda de transcripciones entre agentes con alcance por agente, usa
`agents.list[].memorySearch.qmd.extraCollections` en lugar de `memory.qmd.paths`.
Esas colecciones adicionales siguen la misma forma `{ path, name, pattern? }`, pero
se combinan por agente y pueden conservar nombres compartidos explícitos cuando la ruta
apunta fuera del espacio de trabajo actual.
Si la misma ruta resuelta aparece tanto en `memory.qmd.paths` como en
`memorySearch.qmd.extraCollections`, QMD conserva la primera entrada y omite el
duplicado.

---

## Memoria multimodal (Gemini)

Indexa imágenes y audio junto con Markdown usando Gemini Embedding 2:

| Key                       | Type       | Default    | Description                            |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | Habilita la indexación multimodal             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]` o `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | Tamaño máximo de archivo para indexación             |

Solo se aplica a archivos en `extraPaths`. Las raíces de memoria predeterminadas siguen siendo solo Markdown.
Requiere `gemini-embedding-2-preview`. `fallback` debe ser `"none"`.

Formatos compatibles: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(imágenes); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (audio).

---

## Caché de embeddings

| Key                | Type      | Default | Description                      |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | Almacena en caché los embeddings de fragmentos en SQLite |
| `cache.maxEntries` | `number`  | `50000` | Máximo de embeddings en caché            |

Evita volver a generar embeddings para texto sin cambios durante la reindexación o las actualizaciones de transcripciones.

---

## Indexación por lotes

| Key                           | Type      | Default | Description                |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | Habilita la API de embeddings por lotes |
| `remote.batch.concurrency`    | `number`  | `2`     | Trabajos por lotes en paralelo        |
| `remote.batch.wait`           | `boolean` | `true`  | Espera a que finalice el lote  |
| `remote.batch.pollIntervalMs` | `number`  | --      | Intervalo de sondeo              |
| `remote.batch.timeoutMinutes` | `number`  | --      | Tiempo de espera del lote              |

Disponible para `openai`, `gemini` y `voyage`. El procesamiento por lotes de OpenAI suele ser
más rápido y más económico para rellenos masivos grandes.

---

## Búsqueda de memoria de sesión (experimental)

Indexa transcripciones de sesiones y muéstralas mediante `memory_search`:

| Key                           | Type       | Default      | Description                             |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | Habilita la indexación de sesiones                 |
| `sources`                     | `string[]` | `["memory"]` | Agrega `"sessions"` para incluir transcripciones |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | Umbral de bytes para reindexación              |
| `sync.sessions.deltaMessages` | `number`   | `50`         | Umbral de mensajes para reindexación           |

La indexación de sesiones es opcional y se ejecuta de forma asíncrona. Los resultados pueden estar ligeramente
desactualizados. Los registros de sesión viven en disco, así que trata el acceso al sistema de archivos como el límite
de confianza.

---

## Aceleración vectorial SQLite (sqlite-vec)

| Key                          | Type      | Default | Description                       |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | Usa sqlite-vec para consultas vectoriales |
| `store.vector.extensionPath` | `string`  | incluido | Sobrescribe la ruta de sqlite-vec          |

Cuando sqlite-vec no está disponible, OpenClaw recurre automáticamente a la
similitud del coseno en proceso.

---

## Almacenamiento del índice

| Key                   | Type     | Default                               | Description                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | Ubicación del índice (admite el token `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                           | Tokenizador FTS5 (`unicode61` o `trigram`)   |

---

## Configuración del backend de QMD

Configura `memory.backend = "qmd"` para habilitarlo. Todos los ajustes de QMD se encuentran en
`memory.qmd`:

| Key                      | Type      | Default  | Description                                  |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | Ruta del ejecutable de QMD                          |
| `searchMode`             | `string`  | `search` | Comando de búsqueda: `search`, `vsearch`, `query` |
| `includeDefaultMemory`   | `boolean` | `true`   | Indexa automáticamente `MEMORY.md` + `memory/**/*.md`    |
| `paths[]`                | `array`   | --       | Rutas adicionales: `{ name, path, pattern? }`      |
| `sessions.enabled`       | `boolean` | `false`  | Indexa transcripciones de sesión                    |
| `sessions.retentionDays` | `number`  | --       | Retención de transcripciones                         |
| `sessions.exportDir`     | `string`  | --       | Directorio de exportación                             |

### Programación de actualizaciones

| Key                       | Type      | Default | Description                           |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | Intervalo de actualización                      |
| `update.debounceMs`       | `number`  | `15000` | Antirrebote para cambios de archivos                 |
| `update.onBoot`           | `boolean` | `true`  | Actualiza al iniciar                    |
| `update.waitForBootSync`  | `boolean` | `false` | Bloquea el inicio hasta que termine la actualización |
| `update.embedInterval`    | `string`  | --      | Cadencia independiente para embeddings                |
| `update.commandTimeoutMs` | `number`  | --      | Tiempo de espera para comandos de QMD              |
| `update.updateTimeoutMs`  | `number`  | --      | Tiempo de espera para operaciones de actualización de QMD     |
| `update.embedTimeoutMs`   | `number`  | --      | Tiempo de espera para operaciones de embeddings de QMD      |

### Límites

| Key                       | Type     | Default | Description                |
| ------------------------- | -------- | ------- | -------------------------- |
| `limits.maxResults`       | `number` | `6`     | Máximo de resultados de búsqueda         |
| `limits.maxSnippetChars`  | `number` | --      | Limita la longitud del fragmento       |
| `limits.maxInjectedChars` | `number` | --      | Limita el total de caracteres inyectados |
| `limits.timeoutMs`        | `number` | `4000`  | Tiempo de espera de búsqueda             |

### Alcance

Controla qué sesiones pueden recibir resultados de búsqueda de QMD. Mismo esquema que
[`session.sendPolicy`](/es/gateway/configuration-reference#session):

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

El valor predeterminado es solo DM. `match.keyPrefix` coincide con la clave de sesión normalizada;
`match.rawKeyPrefix` coincide con la clave sin procesar, incluida `agent:<id>:`.

### Citas

`memory.citations` se aplica a todos los backends:

| Value            | Behavior                                            |
| ---------------- | --------------------------------------------------- |
| `auto` (predeterminado) | Incluye el pie `Source: <path#line>` en los fragmentos    |
| `on`             | Siempre incluye el pie                               |
| `off`            | Omite el pie (la ruta igualmente se pasa internamente al agente) |

### Ejemplo completo de QMD

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming (experimental)

Dreaming se configura en `plugins.entries.memory-core.config.dreaming`,
no en `agents.defaults.memorySearch`. Para detalles conceptuales y comandos
de chat, consulta [Dreaming](/es/concepts/memory-dreaming).

| Key                | Type     | Default        | Description                               |
| ------------------ | -------- | -------------- | ----------------------------------------- |
| `mode`             | `string` | `"off"`        | Preajuste: `off`, `core`, `rem` o `deep`   |
| `cron`             | `string` | predeterminado del preajuste | Sobrescribe la expresión cron para la programación |
| `timezone`         | `string` | zona horaria del usuario  | Zona horaria para la evaluación de la programación          |
| `limit`            | `number` | predeterminado del preajuste | Máximo de candidatos para promover por ciclo       |
| `minScore`         | `number` | predeterminado del preajuste | Puntuación ponderada mínima para promoción      |
| `minRecallCount`   | `number` | predeterminado del preajuste | Umbral mínimo del conteo de recuperación            |
| `minUniqueQueries` | `number` | predeterminado del preajuste | Umbral mínimo del conteo de consultas distintas    |

### Valores predeterminados del preajuste

| Mode   | Cadence        | minScore | minRecallCount | minUniqueQueries |
| ------ | -------------- | -------- | -------------- | ---------------- |
| `off`  | Deshabilitado       | --       | --             | --               |
| `core` | Diario a las 3 AM     | 0.75     | 3              | 2                |
| `rem`  | Cada 6 horas  | 0.85     | 4              | 3                |
| `deep` | Cada 12 horas | 0.80     | 3              | 3                |

### Ejemplo

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            mode: "core",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```
