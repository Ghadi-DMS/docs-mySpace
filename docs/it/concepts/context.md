---
read_when:
    - Vuoi capire cosa significa “context” in OpenClaw
    - Stai eseguendo il debug del motivo per cui il modello “sa” qualcosa (o lo ha dimenticato)
    - Vuoi ridurre l'overhead del contesto (`/context`, `/status`, `/compact`)
summary: 'Contesto: cosa vede il modello, come viene costruito e come ispezionarlo'
title: Contesto
x-i18n:
    generated_at: "2026-04-06T03:06:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: fe7dfe52cb1a64df229c8622feed1804df6c483a6243e0d2f309f6ff5c9fe521
    source_path: concepts/context.md
    workflow: 15
---

# Contesto

Il “contesto” è **tutto ciò che OpenClaw invia al modello per un'esecuzione**. È limitato dalla **finestra di contesto** del modello (limite di token).

Modello mentale per principianti:

- **Prompt di sistema** (costruito da OpenClaw): regole, strumenti, elenco delle Skills, informazioni su ora/runtime e file del workspace iniettati.
- **Cronologia della conversazione**: i tuoi messaggi + i messaggi dell'assistente per questa sessione.
- **Chiamate/risultati degli strumenti + allegati**: output dei comandi, letture di file, immagini/audio, ecc.

Il contesto _non è la stessa cosa_ della “memoria”: la memoria può essere archiviata su disco e ricaricata in seguito; il contesto è ciò che si trova nella finestra corrente del modello.

## Avvio rapido (ispezionare il contesto)

- `/status` → vista rapida di “quanto è piena la mia finestra?” + impostazioni della sessione.
- `/context list` → cosa viene iniettato + dimensioni approssimative (per file + totali).
- `/context detail` → ripartizione più approfondita: per file, dimensioni degli schemi degli strumenti, dimensioni delle voci delle skill e dimensione del prompt di sistema.
- `/usage tokens` → aggiunge il piè di pagina dell'utilizzo per risposta alle normali risposte.
- `/compact` → riassume la cronologia meno recente in una voce compatta per liberare spazio nella finestra.

Vedi anche: [Comandi slash](/it/tools/slash-commands), [Uso dei token e costi](/it/reference/token-use), [Compattazione](/it/concepts/compaction).

## Output di esempio

I valori variano in base al modello, al provider, alla policy degli strumenti e a ciò che si trova nel tuo workspace.

### `/context list`

```
🧠 Ripartizione del contesto
Workspace: <workspaceDir>
Massimo bootstrap/file: 20,000 caratteri
Sandbox: mode=non-main sandboxed=false
Prompt di sistema (esecuzione): 38,412 caratteri (~9,603 token) (Project Context 23,901 caratteri (~5,976 token))

File del workspace iniettati:
- AGENTS.md: OK | raw 1,742 caratteri (~436 token) | injected 1,742 caratteri (~436 token)
- SOUL.md: OK | raw 912 caratteri (~228 token) | injected 912 caratteri (~228 token)
- TOOLS.md: TRUNCATED | raw 54,210 caratteri (~13,553 token) | injected 20,962 caratteri (~5,241 token)
- IDENTITY.md: OK | raw 211 caratteri (~53 token) | injected 211 caratteri (~53 token)
- USER.md: OK | raw 388 caratteri (~97 token) | injected 388 caratteri (~97 token)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 caratteri (~0 token) | injected 0 caratteri (~0 token)

Elenco Skills (testo del prompt di sistema): 2,184 caratteri (~546 token) (12 skills)
Strumenti: read, edit, write, exec, process, browser, message, sessions_send, …
Elenco strumenti (testo del prompt di sistema): 1,032 caratteri (~258 token)
Schemi degli strumenti (JSON): 31,988 caratteri (~7,997 token) (contano nel contesto; non mostrati come testo)
Strumenti: (come sopra)

Token della sessione (in cache): 14,250 totale / ctx=32,000
```

### `/context detail`

```
🧠 Ripartizione del contesto (dettagliata)
…
Principali skill (dimensione della voce nel prompt):
- frontend-design: 412 caratteri (~103 token)
- oracle: 401 caratteri (~101 token)
… (+10 altre skill)

Principali strumenti (dimensione dello schema):
- browser: 9,812 caratteri (~2,453 token)
- exec: 6,240 caratteri (~1,560 token)
… (+N altri strumenti)
```

## Cosa conta ai fini della finestra di contesto

Tutto ciò che il modello riceve conta, inclusi:

- Prompt di sistema (tutte le sezioni).
- Cronologia della conversazione.
- Chiamate degli strumenti + risultati degli strumenti.
- Allegati/trascrizioni (immagini/audio/file).
- Riepiloghi di compattazione e artefatti di potatura.
- “Wrapper” del provider o intestazioni nascoste (non visibili, ma comunque conteggiate).

## Come OpenClaw costruisce il prompt di sistema

Il prompt di sistema è **di proprietà di OpenClaw** e viene ricostruito a ogni esecuzione. Include:

- Elenco degli strumenti + brevi descrizioni.
- Elenco delle Skills (solo metadati; vedi sotto).
- Posizione del workspace.
- Ora (UTC + ora utente convertita, se configurata).
- Metadati di runtime (host/OS/modello/ragionamento).
- File bootstrap del workspace iniettati sotto **Project Context**.

Ripartizione completa: [Prompt di sistema](/it/concepts/system-prompt).

## File del workspace iniettati (Project Context)

Per impostazione predefinita, OpenClaw inietta un insieme fisso di file del workspace (se presenti):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo alla prima esecuzione)

I file di grandi dimensioni vengono troncati per file usando `agents.defaults.bootstrapMaxChars` (predefinito `20000` caratteri). OpenClaw applica anche un limite totale di iniezione bootstrap tra i file con `agents.defaults.bootstrapTotalMaxChars` (predefinito `150000` caratteri). `/context` mostra le dimensioni **raw vs injected** e se si è verificato un troncamento.

Quando si verifica un troncamento, il runtime può iniettare un blocco di avviso nel prompt sotto Project Context. Configuralo con `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; valore predefinito `once`).

## Skills: iniettate vs caricate su richiesta

Il prompt di sistema include un **elenco delle Skills** compatto (nome + descrizione + posizione). Questo elenco ha un overhead reale.

Le istruzioni delle skill _non_ sono incluse per impostazione predefinita. Ci si aspetta che il modello usi `read` sul `SKILL.md` della skill **solo quando necessario**.

## Strumenti: ci sono due costi

Gli strumenti influenzano il contesto in due modi:

1. **Testo dell'elenco strumenti** nel prompt di sistema (quello che vedi come “Tooling”).
2. **Schemi degli strumenti** (JSON). Vengono inviati al modello affinché possa chiamare gli strumenti. Contano nel contesto anche se non li vedi come testo normale.

`/context detail` suddivide i principali schemi degli strumenti così puoi vedere cosa pesa di più.

## Comandi, direttive e "scorciatoie inline"

I comandi slash sono gestiti dal Gateway. Esistono alcuni comportamenti diversi:

- **Comandi standalone**: un messaggio composto solo da `/...` viene eseguito come comando.
- **Direttive**: `/think`, `/verbose`, `/reasoning`, `/elevated`, `/model`, `/queue` vengono rimosse prima che il modello veda il messaggio.
  - I messaggi contenenti solo direttive mantengono le impostazioni della sessione.
  - Le direttive inline in un messaggio normale agiscono come suggerimenti per quel singolo messaggio.
- **Scorciatoie inline** (solo mittenti consentiti): alcuni token `/...` all'interno di un messaggio normale possono essere eseguiti immediatamente (esempio: “hey /status”), e vengono rimossi prima che il modello veda il testo rimanente.

Dettagli: [Comandi slash](/it/tools/slash-commands).

## Sessioni, compattazione e potatura (cosa persiste)

Ciò che persiste tra i messaggi dipende dal meccanismo:

- **Cronologia normale** persiste nella trascrizione della sessione finché non viene compattata/potata dalla policy.
- **Compattazione** mantiene un riepilogo nella trascrizione e conserva intatti i messaggi recenti.
- **Potatura** rimuove i vecchi risultati degli strumenti dal prompt _in memoria_ per un'esecuzione, ma non riscrive la trascrizione.

Documentazione: [Sessione](/it/concepts/session), [Compattazione](/it/concepts/compaction), [Potatura della sessione](/it/concepts/session-pruning).

Per impostazione predefinita, OpenClaw usa il motore di contesto integrato `legacy` per l'assemblaggio e la
compattazione. Se installi un plugin che fornisce `kind: "context-engine"` e
lo selezioni con `plugins.slots.contextEngine`, OpenClaw delega invece
l'assemblaggio del contesto, `/compact` e i relativi hook del ciclo di vita del contesto dei sottoagenti a quel
motore. `ownsCompaction: false` non esegue automaticamente un fallback al motore
legacy; il motore attivo deve comunque implementare correttamente `compact()`. Vedi
[Context Engine](/it/concepts/context-engine) per l'intera
interfaccia estendibile, gli hook del ciclo di vita e la configurazione.

## Cosa riporta realmente `/context`

`/context` privilegia il report del prompt di sistema **costruito in esecuzione** più recente quando disponibile:

- `System prompt (run)` = acquisito dall'ultima esecuzione incorporata (capace di usare strumenti) e mantenuto nello store della sessione.
- `System prompt (estimate)` = calcolato al volo quando non esiste ancora alcun report di esecuzione.

In entrambi i casi, riporta dimensioni e principali contributori; **non** scarica il prompt di sistema completo né gli schemi degli strumenti.

## Correlati

- [Context Engine](/it/concepts/context-engine) — iniezione di contesto personalizzata tramite plugin
- [Compattazione](/it/concepts/compaction) — riepilogo delle conversazioni lunghe
- [Prompt di sistema](/it/concepts/system-prompt) — come viene costruito il prompt di sistema
- [Ciclo dell'agente](/it/concepts/agent-loop) — il ciclo completo di esecuzione dell'agente
