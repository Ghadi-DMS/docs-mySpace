---
read_when:
    - Vuoi che la promozione della memoria venga eseguita automaticamente
    - Vuoi capire cosa fa ciascuna fase del dreaming
    - Vuoi regolare il consolidamento senza inquinare `MEMORY.md`
summary: Consolidamento della memoria in background con fasi light, deep e REM, più un Dream Diary
title: Dreaming (sperimentale)
x-i18n:
    generated_at: "2026-04-08T08:12:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0254f3b0949158264e583c12f36f2b1a83d1b44dc4da01a1b272422d38e8655d
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (sperimentale)

Dreaming è il sistema di consolidamento della memoria in background in `memory-core`.
Aiuta OpenClaw a spostare segnali forti a breve termine nella memoria duratura mantenendo
il processo spiegabile e verificabile.

Dreaming è **opt-in** ed è disattivato per impostazione predefinita.

## Cosa scrive il dreaming

Dreaming mantiene due tipi di output:

- **Stato macchina** in `memory/.dreams/` (archivio di richiamo, segnali di fase, checkpoint di ingestione, lock).
- **Output leggibile da umani** in `DREAMS.md` (o `dreams.md` esistente) e file di report di fase opzionali in `memory/dreaming/<phase>/YYYY-MM-DD.md`.

La promozione a lungo termine continua a scrivere solo in `MEMORY.md`.

## Modello delle fasi

Dreaming usa tre fasi cooperative:

| Fase | Scopo                                    | Scrittura duratura |
| ----- | ---------------------------------------- | ------------------ |
| Light | Ordina e prepara il materiale recente a breve termine | No                 |
| Deep  | Valuta e promuove i candidati durevoli   | Sì (`MEMORY.md`)   |
| REM   | Riflette sui temi e sulle idee ricorrenti | No                |

Queste fasi sono dettagli interni di implementazione, non "modalità" separate configurabili dall'utente.

### Fase light

La fase light acquisisce i segnali recenti della memoria giornaliera e le tracce di richiamo, li deduplica
e prepara righe candidate.

- Legge dallo stato di richiamo a breve termine, dai file recenti della memoria giornaliera e dalle trascrizioni di sessione redatte quando disponibili.
- Scrive un blocco gestito `## Light Sleep` quando l'archiviazione include output inline.
- Registra segnali di rinforzo per il ranking deep successivo.
- Non scrive mai in `MEMORY.md`.

### Fase deep

La fase deep decide cosa diventa memoria a lungo termine.

- Classifica i candidati usando punteggi pesati e soglie di accesso.
- Richiede il superamento di `minScore`, `minRecallCount` e `minUniqueQueries`.
- Reidrata gli snippet dai file giornalieri live prima di scrivere, quindi gli snippet obsoleti o eliminati vengono ignorati.
- Aggiunge le voci promosse a `MEMORY.md`.
- Scrive un riepilogo `## Deep Sleep` in `DREAMS.md` e facoltativamente scrive `memory/dreaming/deep/YYYY-MM-DD.md`.

### Fase REM

La fase REM estrae pattern e segnali riflessivi.

- Costruisce riepiloghi di temi e riflessioni a partire dalle recenti tracce a breve termine.
- Scrive un blocco gestito `## REM Sleep` quando l'archiviazione include output inline.
- Registra segnali di rinforzo REM usati dal ranking deep.
- Non scrive mai in `MEMORY.md`.

## Ingestione delle trascrizioni di sessione

Dreaming può acquisire trascrizioni di sessione redatte nel corpus del dreaming. Quando
le trascrizioni sono disponibili, vengono inserite nella fase light insieme ai segnali
della memoria giornaliera e alle tracce di richiamo. I contenuti personali e sensibili vengono redatti
prima dell'ingestione.

## Dream Diary

Dreaming mantiene anche un **Dream Diary** narrativo in `DREAMS.md`.
Dopo che ogni fase ha accumulato materiale sufficiente, `memory-core` esegue un turno
di subagent in background best-effort (usando il modello runtime predefinito) e aggiunge una breve voce di diario.

Questo diario è destinato alla lettura umana nell'interfaccia Dreams, non è una fonte di promozione.

## Segnali di ranking deep

Il ranking deep usa sei segnali di base pesati più il rinforzo di fase:

| Segnale            | Peso  | Descrizione                                        |
| ------------------ | ----- | -------------------------------------------------- |
| Frequenza          | 0.24  | Quanti segnali a breve termine ha accumulato la voce |
| Rilevanza          | 0.30  | Qualità media di recupero per la voce              |
| Diversità delle query | 0.15 | Contesti distinti di query/giorno che l'hanno fatta emergere |
| Recency            | 0.15  | Punteggio di freschezza con decadimento temporale  |
| Consolidamento     | 0.10  | Intensità della ricorrenza su più giorni           |
| Ricchezza concettuale | 0.06 | Densità dei tag concettuali da snippet/percorso   |

I riscontri delle fasi light e REM aggiungono un piccolo incremento con decadimento temporale da
`memory/.dreams/phase-signals.json`.

## Pianificazione

Quando è abilitato, `memory-core` gestisce automaticamente un job cron per un ciclo
completo di dreaming. Ogni ciclo esegue le fasi in ordine: light -> REM -> deep.

Comportamento predefinito della cadenza:

| Impostazione        | Predefinito |
| ------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## Avvio rapido

Abilita il dreaming:

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

Abilita il dreaming con una cadenza di ciclo personalizzata:

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

## Flusso di lavoro CLI

Usa la promozione CLI per l'anteprima o l'applicazione manuale:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

`memory promote` manuale usa per impostazione predefinita le soglie della fase deep, a meno che non vengano sovrascritte
con flag CLI.

Spiega perché un candidato specifico verrebbe o non verrebbe promosso:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Visualizza in anteprima le riflessioni REM, le verità candidate e l'output di promozione deep senza
scrivere nulla:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Valori predefiniti principali

Tutte le impostazioni si trovano sotto `plugins.entries.memory-core.config.dreaming`.

| Chiave      | Predefinito |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

La policy delle fasi, le soglie e il comportamento di archiviazione sono dettagli interni di implementazione
(non configurazione rivolta all'utente).

Consulta [Riferimento alla configurazione della memoria](/it/reference/memory-config#dreaming-experimental)
per l'elenco completo delle chiavi.

## Interfaccia Dreams

Quando è abilitata, la scheda **Dreams** del Gateway mostra:

- stato corrente di abilitazione del dreaming
- stato a livello di fase e presenza del ciclo gestito
- conteggi di breve termine, lungo termine e promossi oggi
- tempistica della prossima esecuzione pianificata
- un lettore Dream Diary espandibile basato su `doctor.memory.dreamDiary`

## Correlati

- [Memoria](/it/concepts/memory)
- [Ricerca nella memoria](/it/concepts/memory-search)
- [CLI memory](/cli/memory)
- [Riferimento alla configurazione della memoria](/it/reference/memory-config)
