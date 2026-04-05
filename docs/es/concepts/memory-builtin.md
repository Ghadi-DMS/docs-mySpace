---
read_when:
    - Quieres entender el backend de memoria predeterminado
    - Quieres configurar proveedores de embeddings o búsqueda híbrida
summary: El backend de memoria predeterminado basado en SQLite con búsqueda por palabras clave, vectorial e híbrida
title: Motor de memoria integrado
x-i18n:
    generated_at: "2026-04-05T12:39:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 181c40a43332315bf915ff6f395d9d5fd766c889e1a8d1aa525f9ba0198d3367
    source_path: concepts/memory-builtin.md
    workflow: 15
---

# Motor de memoria integrado

El motor integrado es el backend de memoria predeterminado. Almacena tu índice de memoria en
una base de datos SQLite por agente y no necesita dependencias adicionales para empezar.

## Qué ofrece

- **Búsqueda por palabras clave** mediante indexación de texto completo FTS5 (puntuación BM25).
- **Búsqueda vectorial** mediante embeddings de cualquier proveedor compatible.
- **Búsqueda híbrida** que combina ambas para obtener mejores resultados.
- **Compatibilidad con CJK** mediante tokenización trigram para chino, japonés y coreano.
- **Aceleración con sqlite-vec** para consultas vectoriales dentro de la base de datos (opcional).

## Primeros pasos

Si tienes una API key para OpenAI, Gemini, Voyage o Mistral, el motor integrado
la detecta automáticamente y habilita la búsqueda vectorial. No se necesita configuración.

Para establecer un proveedor explícitamente:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
      },
    },
  },
}
```

Sin un proveedor de embeddings, solo está disponible la búsqueda por palabras clave.

## Proveedores de embeddings compatibles

| Proveedor | ID        | Detección automática | Notas                               |
| --------- | --------- | -------------------- | ----------------------------------- |
| OpenAI    | `openai`  | Sí                   | Predeterminado: `text-embedding-3-small` |
| Gemini    | `gemini`  | Sí                   | Admite multimodal (imagen + audio)  |
| Voyage    | `voyage`  | Sí                   |                                     |
| Mistral   | `mistral` | Sí                   |                                     |
| Ollama    | `ollama`  | No                   | Local, establecer explícitamente    |
| Local     | `local`   | Sí (primero)         | Modelo GGUF, descarga de ~0.6 GB    |

La detección automática elige el primer proveedor cuya API key puede resolverse, en el
orden mostrado. Establece `memorySearch.provider` para sobrescribirlo.

## Cómo funciona la indexación

OpenClaw indexa `MEMORY.md` y `memory/*.md` en fragmentos (~400 tokens con
superposición de 80 tokens) y los almacena en una base de datos SQLite por agente.

- **Ubicación del índice:** `~/.openclaw/memory/<agentId>.sqlite`
- **Supervisión de archivos:** los cambios en archivos de memoria activan una reindexación con debounce (1.5 s).
- **Reindexación automática:** cuando cambia el proveedor de embeddings, el modelo o la configuración de fragmentación,
  todo el índice se reconstruye automáticamente.
- **Reindexación a demanda:** `openclaw memory index --force`

<Info>
También puedes indexar archivos Markdown fuera del espacio de trabajo con
`memorySearch.extraPaths`. Consulta la
[referencia de configuración](/reference/memory-config#additional-memory-paths).
</Info>

## Cuándo usarlo

El motor integrado es la opción correcta para la mayoría de los usuarios:

- Funciona de inmediato sin dependencias adicionales.
- Maneja bien la búsqueda por palabras clave y vectorial.
- Admite todos los proveedores de embeddings.
- La búsqueda híbrida combina lo mejor de ambos enfoques de recuperación.

Considera cambiar a [QMD](/concepts/memory-qmd) si necesitas reranking, expansión
de consultas o quieres indexar directorios fuera del espacio de trabajo.

Considera [Honcho](/concepts/memory-honcho) si quieres memoria entre sesiones con
modelado automático del usuario.

## Solución de problemas

**¿Búsqueda de memoria deshabilitada?** Revisa `openclaw memory status`. Si no se
detecta ningún proveedor, establece uno explícitamente o añade una API key.

**¿Resultados obsoletos?** Ejecuta `openclaw memory index --force` para reconstruir. El observador
puede no detectar cambios en casos límite poco frecuentes.

**¿sqlite-vec no carga?** OpenClaw vuelve automáticamente a similitud coseno en proceso.
Revisa los registros para ver el error de carga específico.

## Configuración

Para la configuración de proveedores de embeddings, ajuste de búsqueda híbrida (pesos, MMR, decaimiento
temporal), indexación por lotes, memoria multimodal, sqlite-vec, rutas adicionales y
todos los demás controles de configuración, consulta la
[referencia de configuración de memoria](/reference/memory-config).
