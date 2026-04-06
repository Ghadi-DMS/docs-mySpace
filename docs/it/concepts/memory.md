---
read_when:
    - Vuoi capire come funziona la memoria
    - Vuoi sapere quali file di memoria scrivere
summary: Come OpenClaw ricorda le cose tra una sessione e l'altra
title: Panoramica della memoria
x-i18n:
    generated_at: "2026-04-06T03:06:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: d19d4fa9c4b3232b7a97f7a382311d2a375b562040de15e9fe4a0b1990b825e7
    source_path: concepts/memory.md
    workflow: 15
---

# Panoramica della memoria

OpenClaw ricorda le cose scrivendo **semplici file Markdown** nel workspace del
tuo agente. Il modello "ricorda" solo ciò che viene salvato su disco -- non c'è
alcuno stato nascosto.

## Come funziona

Il tuo agente ha tre file correlati alla memoria:

- **`MEMORY.md`** -- memoria a lungo termine. Fatti durevoli, preferenze e
  decisioni. Viene caricato all'inizio di ogni sessione DM.
- **`memory/YYYY-MM-DD.md`** -- note giornaliere. Contesto in corso e osservazioni.
  Le note di oggi e di ieri vengono caricate automaticamente.
- **`DREAMS.md`** (sperimentale, facoltativo) -- Dream Diary e riepiloghi delle
  scansioni di dreaming per la revisione umana.

Questi file si trovano nel workspace dell'agente (predefinito `~/.openclaw/workspace`).

<Tip>
Se vuoi che il tuo agente ricordi qualcosa, chiediglielo semplicemente: "Ricorda che
preferisco TypeScript." Lo scriverà nel file appropriato.
</Tip>

## Strumenti di memoria

L'agente ha due strumenti per lavorare con la memoria:

- **`memory_search`** -- trova note pertinenti usando la ricerca semantica, anche quando
  la formulazione è diversa dall'originale.
- **`memory_get`** -- legge uno specifico file di memoria o un intervallo di righe.

Entrambi gli strumenti sono forniti dal plugin di memoria attivo (predefinito: `memory-core`).

## Ricerca nella memoria

Quando è configurato un provider di embedding, `memory_search` usa la **ricerca
ibrida** -- combinando la similarità vettoriale (significato semantico) con la corrispondenza di parole chiave
(termini esatti come ID e simboli di codice). Funziona subito una volta che hai
una chiave API per qualsiasi provider supportato.

<Info>
OpenClaw rileva automaticamente il tuo provider di embedding dalle chiavi API disponibili. Se
hai configurato una chiave OpenAI, Gemini, Voyage o Mistral, la ricerca nella memoria è
abilitata automaticamente.
</Info>

Per i dettagli su come funziona la ricerca, le opzioni di ottimizzazione e la configurazione del provider, vedi
[Ricerca nella memoria](/it/concepts/memory-search).

## Backend di memoria

<CardGroup cols={3}>
<Card title="Integrato (predefinito)" icon="database" href="/it/concepts/memory-builtin">
Basato su SQLite. Funziona subito con ricerca per parole chiave, similarità vettoriale e
ricerca ibrida. Nessuna dipendenza aggiuntiva.
</Card>
<Card title="QMD" icon="search" href="/it/concepts/memory-qmd">
Sidecar local-first con reranking, espansione delle query e la possibilità di indicizzare
directory esterne al workspace.
</Card>
<Card title="Honcho" icon="brain" href="/it/concepts/memory-honcho">
Memoria cross-session nativa per l'AI con modellazione dell'utente, ricerca semantica e
consapevolezza multi-agente. Installazione del plugin.
</Card>
</CardGroup>

## Flush automatico della memoria

Prima che la [compattazione](/it/concepts/compaction) riassuma la tua conversazione, OpenClaw
esegue un turno silenzioso che ricorda all'agente di salvare il contesto importante nei file
di memoria. È attivo per impostazione predefinita -- non devi configurare nulla.

<Tip>
Il flush della memoria evita la perdita di contesto durante la compattazione. Se il tuo agente ha
fatti importanti nella conversazione che non sono ancora stati scritti in un file, verranno
salvati automaticamente prima che avvenga il riepilogo.
</Tip>

## Dreaming (sperimentale)

Il dreaming è un passaggio facoltativo di consolidamento in background per la memoria. Raccoglie
segnali a breve termine, assegna un punteggio ai candidati e promuove solo gli elementi qualificati nella
memoria a lungo termine (`MEMORY.md`).

È progettato per mantenere alto il rapporto segnale/rumore nella memoria a lungo termine:

- **Attivazione facoltativa**: disabilitato per impostazione predefinita.
- **Pianificato**: quando è abilitato, `memory-core` gestisce automaticamente un job cron ricorrente
  per una scansione completa di dreaming.
- **Con soglia**: le promozioni devono superare le soglie di punteggio, frequenza di richiamo e
  diversità delle query.
- **Rivedibile**: i riepiloghi delle fasi e le voci del diario vengono scritti in `DREAMS.md`
  per la revisione umana.

Per il comportamento delle fasi, i segnali di punteggio e i dettagli del Dream Diary, vedi
[Dreaming (sperimentale)](/concepts/dreaming).

## CLI

```bash
openclaw memory status          # Controlla lo stato dell'indice e del provider
openclaw memory search "query"  # Cerca dalla riga di comando
openclaw memory index --force   # Ricostruisce l'indice
```

## Ulteriori letture

- [Builtin Memory Engine](/it/concepts/memory-builtin) -- backend SQLite predefinito
- [QMD Memory Engine](/it/concepts/memory-qmd) -- sidecar local-first avanzato
- [Honcho Memory](/it/concepts/memory-honcho) -- memoria cross-session nativa per l'AI
- [Memory Search](/it/concepts/memory-search) -- pipeline di ricerca, provider e
  ottimizzazione
- [Dreaming (experimental)](/concepts/dreaming) -- promozione in background
  dal richiamo a breve termine alla memoria a lungo termine
- [Riferimento alla configurazione della memoria](/it/reference/memory-config) -- tutte le opzioni di configurazione
- [Compattazione](/it/concepts/compaction) -- come la compattazione interagisce con la memoria
