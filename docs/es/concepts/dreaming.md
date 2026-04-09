---
read_when:
    - Quieres que la promoción de memoria se ejecute automáticamente
    - Quieres entender qué hace cada fase de dreaming
    - Quieres ajustar la consolidación sin contaminar `MEMORY.md`
summary: Consolidación de memoria en segundo plano con fases ligera, profunda y REM, además de un Diario de Sueños
title: Dreaming (experimental)
x-i18n:
    generated_at: "2026-04-09T01:27:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26476eddb8260e1554098a6adbb069cf7f5e284cf2e09479c6d9d8f8b93280ef
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (experimental)

Dreaming es el sistema de consolidación de memoria en segundo plano en `memory-core`.
Ayuda a OpenClaw a mover señales sólidas de corto plazo a memoria duradera mientras
mantiene el proceso explicable y revisable.

Dreaming es **optativo** y está deshabilitado de forma predeterminada.

## Qué escribe dreaming

Dreaming mantiene dos tipos de salida:

- **Estado de máquina** en `memory/.dreams/` (almacén de recuperación, señales de fase, puntos de control de ingesta, bloqueos).
- **Salida legible por humanos** en `DREAMS.md` (o `dreams.md` existente) y archivos opcionales de informe de fase en `memory/dreaming/<phase>/YYYY-MM-DD.md`.

La promoción a largo plazo sigue escribiendo solo en `MEMORY.md`.

## Modelo de fases

Dreaming usa tres fases cooperativas:

| Fase | Propósito                                 | Escritura duradera |
| ----- | ----------------------------------------- | ------------------ |
| Light | Ordenar y preparar material reciente de corto plazo | No                 |
| Deep  | Puntuar y promover candidatos duraderos   | Sí (`MEMORY.md`)   |
| REM   | Reflexionar sobre temas e ideas recurrentes | No               |

Estas fases son detalles internos de implementación, no "modos"
configurables por el usuario por separado.

### Fase Light

La fase Light ingiere señales recientes de memoria diaria y rastros de recuperación, los deduplica
y prepara líneas candidatas.

- Lee del estado de recuperación de corto plazo, archivos recientes de memoria diaria y transcripciones de sesión redactadas cuando están disponibles.
- Escribe un bloque administrado `## Light Sleep` cuando el almacenamiento incluye salida en línea.
- Registra señales de refuerzo para una clasificación profunda posterior.
- Nunca escribe en `MEMORY.md`.

### Fase Deep

La fase Deep decide qué pasa a convertirse en memoria a largo plazo.

- Clasifica los candidatos usando puntuación ponderada y umbrales de validación.
- Requiere que `minScore`, `minRecallCount` y `minUniqueQueries` se cumplan.
- Rehidrata fragmentos desde archivos diarios activos antes de escribir, por lo que se omiten los fragmentos obsoletos o eliminados.
- Agrega las entradas promovidas a `MEMORY.md`.
- Escribe un resumen `## Deep Sleep` en `DREAMS.md` y, opcionalmente, escribe `memory/dreaming/deep/YYYY-MM-DD.md`.

### Fase REM

La fase REM extrae patrones y señales reflexivas.

- Construye resúmenes de temas y reflexiones a partir de rastros recientes de corto plazo.
- Escribe un bloque administrado `## REM Sleep` cuando el almacenamiento incluye salida en línea.
- Registra señales de refuerzo REM usadas por la clasificación profunda.
- Nunca escribe en `MEMORY.md`.

## Ingesta de transcripciones de sesión

Dreaming puede ingerir transcripciones de sesión redactadas en el corpus de dreaming. Cuando
las transcripciones están disponibles, se incorporan en la fase Light junto con señales de
memoria diaria y rastros de recuperación. El contenido personal y sensible se redacta
antes de la ingesta.

## Diario de Sueños

Dreaming también mantiene un **Diario de Sueños** narrativo en `DREAMS.md`.
Después de que cada fase tiene suficiente material, `memory-core` ejecuta un turno
de subagente en segundo plano de mejor esfuerzo (usando el modelo de runtime predeterminado)
y agrega una entrada breve del diario.

Este diario es para lectura humana en la IU de Dreams, no una fuente de promoción.

También existe una vía de relleno histórico fundamentado para trabajo de revisión y recuperación:

- `memory rem-harness --path ... --grounded` previsualiza la salida del diario fundamentado a partir de notas históricas `YYYY-MM-DD.md`.
- `memory rem-backfill --path ...` escribe entradas fundamentadas reversibles del diario en `DREAMS.md`.
- `memory rem-backfill --path ... --stage-short-term` prepara candidatos duraderos fundamentados en el mismo almacén de evidencias de corto plazo que la fase Deep normal ya usa.
- `memory rem-backfill --rollback` y `--rollback-short-term` eliminan esos artefactos de relleno preparados sin tocar las entradas normales del diario ni la recuperación activa de corto plazo.

La IU de Control expone el mismo flujo de relleno/restablecimiento del diario para que puedas inspeccionar
los resultados en la escena de Dreams antes de decidir si los candidatos fundamentados
merecen promoción. La escena también muestra una vía fundamentada distinta para que puedas ver
qué entradas preparadas de corto plazo provinieron de la reproducción histórica, qué elementos promovidos
fueron impulsados por la vía fundamentada, y borrar solo las entradas preparadas exclusivamente fundamentadas sin
tocar el estado normal activo de corto plazo.

## Señales de clasificación profunda

La clasificación profunda usa seis señales base ponderadas más refuerzo de fase:

| Señal               | Peso | Descripción                                       |
| ------------------- | ---- | ------------------------------------------------- |
| Frecuencia          | 0.24 | Cuántas señales de corto plazo acumuló la entrada |
| Relevancia          | 0.30 | Calidad media de recuperación de la entrada       |
| Diversidad de consultas | 0.15 | Contextos distintos de consulta/día en que apareció |
| Recencia            | 0.15 | Puntuación de frescura con decaimiento temporal   |
| Consolidación       | 0.10 | Fuerza de recurrencia en varios días              |
| Riqueza conceptual  | 0.06 | Densidad de etiquetas conceptuales del fragmento/ruta |

Los aciertos de las fases Light y REM agregan un pequeño impulso con decaimiento por recencia desde
`memory/.dreams/phase-signals.json`.

## Programación

Cuando está habilitado, `memory-core` administra automáticamente un trabajo cron para un barrido
completo de dreaming. Cada barrido ejecuta las fases en orden: light -> REM -> deep.

Comportamiento predeterminado de la cadencia:

| Ajuste               | Predeterminado |
| -------------------- | -------------- |
| `dreaming.frequency` | `0 3 * * *`    |

## Inicio rápido

Habilitar dreaming:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Habilitar dreaming con una cadencia de barrido personalizada:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Comando de barra inclinada

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## Flujo de trabajo de la CLI

Usa la promoción de la CLI para previsualizar o aplicar manualmente:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

`memory promote` manual usa umbrales de la fase Deep de forma predeterminada, a menos que se reemplacen
con banderas de la CLI.

Explica por qué un candidato específico se promovería o no:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Previsualiza reflexiones REM, verdades candidatas y salida de promoción profunda sin
escribir nada:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Valores predeterminados clave

Todos los ajustes están en `plugins.entries.memory-core.config.dreaming`.

| Clave       | Predeterminado |
| ----------- | -------------- |
| `enabled`   | `false`        |
| `frequency` | `0 3 * * *`    |

La política de fases, los umbrales y el comportamiento de almacenamiento son detalles internos de implementación
(no configuración orientada al usuario).

Consulta la [referencia de configuración de Memory](/es/reference/memory-config#dreaming-experimental)
para ver la lista completa de claves.

## IU de Dreams

Cuando está habilitada, la pestaña **Dreams** de Gateway muestra:

- estado actual de dreaming habilitado
- estado a nivel de fase y presencia de barrido administrado
- recuentos de corto plazo, fundamentados, de señales y promovidos hoy
- hora de la próxima ejecución programada
- una vía de escena fundamentada distinta para entradas preparadas de reproducción histórica
- un lector expandible de Diario de Sueños respaldado por `doctor.memory.dreamDiary`

## Relacionado

- [Memory](/es/concepts/memory)
- [Memory Search](/es/concepts/memory-search)
- [CLI de memory](/cli/memory)
- [referencia de configuración de Memory](/es/reference/memory-config)
