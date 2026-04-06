---
read_when:
    - Vuoi che la promozione della memoria venga eseguita automaticamente
    - Vuoi capire cosa fa ogni fase del dreaming
    - Vuoi ottimizzare il consolidamento senza inquinare `MEMORY.md`
summary: Consolidamento della memoria in background con fasi light, deep e REM più un Dream Diary
title: Dreaming (sperimentale)
x-i18n:
    generated_at: "2026-04-06T03:06:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: f27da718176bebf59fe8a80fddd4fb5b6d814ac5647f6c1e8344bcfb328db9de
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (sperimentale)

Dreaming è il sistema di consolidamento della memoria in background in `memory-core`.
Aiuta OpenClaw a spostare segnali forti a breve termine nella memoria durevole, mantenendo
il processo spiegabile e verificabile.

Dreaming è **opt-in** ed è disabilitato per impostazione predefinita.

## Cosa scrive dreaming

Dreaming mantiene due tipi di output:

- **Stato macchina** in `memory/.dreams/` (recall store, segnali di fase, checkpoint di ingestione, lock).
- **Output leggibile dall'uomo** in `DREAMS.md` (o nell'esistente `dreams.md`) e file di report di fase facoltativi in `memory/dreaming/<phase>/YYYY-MM-DD.md`.

La promozione a lungo termine continua a scrivere solo in `MEMORY.md`.

## Modello a fasi

Dreaming usa tre fasi cooperative:

| Fase | Scopo                                     | Scrittura persistente |
| ----- | ----------------------------------------- | --------------------- |
| Light | Ordina e prepara il materiale recente a breve termine | No                    |
| Deep  | Valuta e promuove i candidati durevoli    | Sì (`MEMORY.md`)      |
| REM   | Riflette su temi e idee ricorrenti        | No                    |

Queste fasi sono dettagli interni di implementazione, non "modalità"
separate configurate dall'utente.

### Fase light

La fase light acquisisce segnali recenti di memoria giornaliera e tracce di recall, li deduplica
e prepara le righe candidate.

- Legge dallo stato di recall a breve termine e dai file recenti della memoria giornaliera.
- Scrive un blocco gestito `## Light Sleep` quando l'archiviazione include output inline.
- Registra segnali di rinforzo per il successivo ranking deep.
- Non scrive mai in `MEMORY.md`.

### Fase deep

La fase deep decide cosa diventa memoria a lungo termine.

- Classifica i candidati usando punteggio ponderato e soglie di accesso.
- Richiede il superamento di `minScore`, `minRecallCount` e `minUniqueQueries`.
- Reidrata gli snippet dai file giornalieri live prima della scrittura, quindi gli snippet obsoleti/eliminati vengono saltati.
- Aggiunge le voci promosse a `MEMORY.md`.
- Scrive un riepilogo `## Deep Sleep` in `DREAMS.md` e facoltativamente scrive `memory/dreaming/deep/YYYY-MM-DD.md`.

### Fase REM

La fase REM estrae pattern e segnali riflessivi.

- Costruisce riepiloghi di temi e riflessioni a partire da tracce recenti a breve termine.
- Scrive un blocco gestito `## REM Sleep` quando l'archiviazione include output inline.
- Registra segnali di rinforzo REM usati dal ranking deep.
- Non scrive mai in `MEMORY.md`.

## Dream Diary

Dreaming mantiene anche un **Dream Diary** narrativo in `DREAMS.md`.
Dopo che ogni fase ha accumulato materiale sufficiente, `memory-core` esegue un turno
di subagent in background best-effort (usando il modello runtime predefinito) e aggiunge una breve voce del diario.

Questo diario è destinato alla lettura umana nell'interfaccia Dreams, non è una fonte di promozione.

## Segnali di ranking deep

Il ranking deep usa sei segnali base ponderati più il rinforzo di fase:

| Segnale             | Peso | Descrizione                                       |
| ------------------- | ---- | ------------------------------------------------- |
| Frequenza           | 0.24 | Quanti segnali a breve termine ha accumulato la voce |
| Rilevanza           | 0.30 | Qualità media di recupero per la voce             |
| Diversità delle query | 0.15 | Contesti distinti query/giorno che l'hanno fatta emergere |
| Recenza             | 0.15 | Punteggio di freschezza con decadimento temporale |
| Consolidamento      | 0.10 | Forza di ricorrenza su più giorni                 |
| Ricchezza concettuale | 0.06 | Densità di tag concettuali da snippet/percorso    |

I riscontri delle fasi light e REM aggiungono un piccolo incremento con decadimento della recenza da
`memory/.dreams/phase-signals.json`.

## Pianificazione

Quando è abilitato, `memory-core` gestisce automaticamente un job cron per una
scansione completa del dreaming. Ogni scansione esegue le fasi in ordine: light -> REM -> deep.

Comportamento predefinito della cadenza:

| Impostazione         | Predefinito |
| -------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## Avvio rapido

Abilitare dreaming:

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

Abilitare dreaming con una cadenza di scansione personalizzata:

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

## Comando slash

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## Workflow CLI

Usa la promozione CLI per l'anteprima o l'applicazione manuale:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

`memory promote` manuale usa per impostazione predefinita le soglie della fase deep, salvo override
tramite flag CLI.

## Valori predefiniti principali

Tutte le impostazioni si trovano in `plugins.entries.memory-core.config.dreaming`.

| Chiave      | Predefinito |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

La policy delle fasi, le soglie e il comportamento di archiviazione sono dettagli interni di implementazione
(non configurazione esposta all'utente).

Vedi [Riferimento della configurazione della memoria](/it/reference/memory-config#dreaming-experimental)
per l'elenco completo delle chiavi.

## Interfaccia Dreams

Quando è abilitata, la scheda **Dreams** del Gateway mostra:

- stato corrente di abilitazione di dreaming
- stato a livello di fase e presenza della scansione gestita
- conteggi di breve termine, lungo termine e promossi oggi
- orario della prossima esecuzione pianificata
- un lettore Dream Diary espandibile basato su `doctor.memory.dreamDiary`

## Correlati

- [Memory](/it/concepts/memory)
- [Memory Search](/it/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Riferimento della configurazione della memoria](/it/reference/memory-config)
