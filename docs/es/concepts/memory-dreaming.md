---
read_when:
    - Quieres que la promoción de memory se ejecute automáticamente
    - Quieres entender los modos y umbrales de dreaming
    - Quieres ajustar la consolidación sin contaminar MEMORY.md
summary: Promoción en segundo plano desde el recuerdo a corto plazo hacia la memory a largo plazo
title: Dreaming (experimental)
x-i18n:
    generated_at: "2026-04-05T12:39:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: e9dbb29e9b49e940128c4e08c3fd058bb6ebb0148ca214b78008e3d5763ef1ab
    source_path: concepts/memory-dreaming.md
    workflow: 15
---

# Dreaming (experimental)

Dreaming es el proceso en segundo plano de consolidación de memory en `memory-core`.

Se llama "dreaming" porque el sistema revisita lo que surgió durante el día
y decide qué vale la pena conservar como contexto duradero.

Dreaming es **experimental**, **de adhesión voluntaria** y está **desactivado de forma predeterminada**.

## Qué hace dreaming

1. Rastrea eventos de recuerdo a corto plazo a partir de aciertos de `memory_search` en
   `memory/YYYY-MM-DD.md`.
2. Puntúa esos candidatos de recuerdo con señales ponderadas.
3. Promueve solo los candidatos válidos a `MEMORY.md`.

Esto mantiene la memory a largo plazo centrada en contexto duradero y repetido en lugar de
detalles puntuales.

## Señales de promoción

Dreaming combina cuatro señales:

- **Frecuencia**: con qué frecuencia se recordó el mismo candidato.
- **Relevancia**: qué tan fuertes fueron las puntuaciones de recuerdo cuando se recuperó.
- **Diversidad de consultas**: cuántas intenciones de consulta distintas lo hicieron aparecer.
- **Recencia**: ponderación temporal sobre recuerdos recientes.

La promoción requiere que se superen todos los umbrales configurados, no solo una señal.

### Pesos de las señales

| Señal     | Peso | Descripción                                           |
| --------- | ---- | ----------------------------------------------------- |
| Frecuencia | 0.35 | Con qué frecuencia se recordó la misma entrada        |
| Relevancia | 0.35 | Promedio de puntuaciones de recuerdo al recuperarse   |
| Diversidad | 0.15 | Número de intenciones de consulta distintas que la hicieron aparecer |
| Recencia   | 0.15 | Decaimiento temporal (semivida de 14 días)            |

## Cómo funciona

1. **Seguimiento de recuerdos** -- Cada acierto de `memory_search` se registra en
   `memory/.dreams/short-term-recall.json` con el recuento de recuerdos, las puntuaciones y el
   hash de la consulta.
2. **Puntuación programada** -- En la cadencia configurada, los candidatos se clasifican
   mediante señales ponderadas. Todos los umbrales deben superarse simultáneamente.
3. **Promoción** -- Las entradas válidas se añaden a `MEMORY.md` con una
   marca de tiempo de promoción.
4. **Limpieza** -- Las entradas ya promovidas se filtran de ciclos futuros. Un
   bloqueo de archivo evita ejecuciones concurrentes.

## Modos

`dreaming.mode` controla la cadencia y los umbrales predeterminados:

| Modo   | Cadencia        | minScore | minRecallCount | minUniqueQueries |
| ------ | --------------- | -------- | -------------- | ---------------- |
| `off`  | Desactivado     | --       | --             | --               |
| `core` | Diario a las 3 AM | 0.75   | 3              | 2                |
| `rem`  | Cada 6 horas    | 0.85     | 4              | 3                |
| `deep` | Cada 12 horas   | 0.80     | 3              | 3                |

## Modelo de programación

Cuando dreaming está habilitado, `memory-core` gestiona automáticamente la
programación recurrente. No necesitas crear manualmente un trabajo cron para esta función.

Aun así, puedes ajustar el comportamiento con sobrescrituras explícitas como:

- `dreaming.frequency` (expresión cron)
- `dreaming.timezone`
- `dreaming.limit`
- `dreaming.minScore`
- `dreaming.minRecallCount`
- `dreaming.minUniqueQueries`

## Configurar

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "mode": "core"
          }
        }
      }
    }
  }
}
```

## Comandos de chat

Cambia de modo y comprueba el estado desde el chat:

```
/dreaming core          # Cambia al modo core (nocturno)
/dreaming rem           # Cambia al modo rem (cada 6 h)
/dreaming deep          # Cambia al modo deep (cada 12 h)
/dreaming off           # Desactiva dreaming
/dreaming status        # Muestra la configuración y cadencia actuales
/dreaming help          # Muestra la guía de modos
```

## Comandos de CLI

Previsualiza y aplica promociones desde la línea de comandos:

```bash
# Preview promotion candidates
openclaw memory promote

# Apply promotions to MEMORY.md
openclaw memory promote --apply

# Limit preview count
openclaw memory promote --limit 5

# Include already-promoted entries
openclaw memory promote --include-promoted

# Check dreaming status
openclaw memory status --deep
```

Consulta [CLI de memory](/cli/memory) para la referencia completa de flags.

## Interfaz de Dreams

Cuando dreaming está habilitado, la barra lateral del Gateway muestra una pestaña **Dreams** con
estadísticas de memory (recuento a corto plazo, recuento a largo plazo, recuento promovido) y la hora del siguiente ciclo programado.

## Más información

- [Memory](/concepts/memory)
- [Búsqueda en memory](/concepts/memory-search)
- [CLI de memory](/cli/memory)
- [Referencia de configuración de memory](/reference/memory-config)
