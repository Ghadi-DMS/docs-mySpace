---
read_when:
    - Modifica del testo del prompt di sistema, dell’elenco degli strumenti o delle sezioni relative a orario/Heartbeat
    - Modifica del comportamento di bootstrap dello spazio di lavoro o dell’iniezione delle Skills
summary: Che cosa contiene il prompt di sistema di OpenClaw e come viene assemblato
title: Prompt di sistema
x-i18n:
    generated_at: "2026-04-18T08:05:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: e60705994cebdd9768926168cb1c6d17ab717d7ff02353a5d5e7478ba8191cab
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt di sistema

OpenClaw crea un prompt di sistema personalizzato per ogni esecuzione dell’agente. Il prompt è **di proprietà di OpenClaw** e non usa il prompt predefinito di pi-coding-agent.

Il prompt viene assemblato da OpenClaw e iniettato in ogni esecuzione dell’agente.

I Plugin provider possono contribuire con indicazioni per il prompt compatibili con la cache senza sostituire l’intero prompt di proprietà di OpenClaw. Il runtime del provider può:

- sostituire un piccolo insieme di sezioni core denominate (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- iniettare un **prefisso stabile** sopra il confine della cache del prompt
- iniettare un **suffisso dinamico** sotto il confine della cache del prompt

Usa i contributi di proprietà del provider per l’ottimizzazione specifica della famiglia di modelli. Mantieni la mutazione legacy del prompt `before_prompt_build` per compatibilità o per modifiche al prompt realmente globali, non per il normale comportamento del provider.

## Struttura

Il prompt è intenzionalmente compatto e usa sezioni fisse:

- **Tooling**: promemoria della fonte di verità degli strumenti strutturati più indicazioni a runtime sull’uso degli strumenti.
- **Safety**: breve promemoria di guardrail per evitare comportamenti orientati alla conquista di potere o all’elusione della supervisione.
- **Skills** (quando disponibili): indica al modello come caricare su richiesta le istruzioni delle skill.
- **Aggiornamento automatico di OpenClaw**: come ispezionare in sicurezza la configurazione con
  `config.schema.lookup`, modificare la configurazione con `config.patch`, sostituire l’intera
  configurazione con `config.apply` ed eseguire `update.run` solo su richiesta esplicita
  dell’utente. Anche lo strumento `gateway`, riservato al proprietario, rifiuta di riscrivere
  `tools.exec.ask` / `tools.exec.security`, comprese le alias legacy `tools.bash.*`
  che vengono normalizzate a quei percorsi exec protetti.
- **Workspace**: directory di lavoro (`agents.defaults.workspace`).
- **Documentation**: percorso locale alla documentazione di OpenClaw (repo o pacchetto npm) e quando leggerla.
- **Workspace Files (injected)**: indica che i file di bootstrap sono inclusi sotto.
- **Sandbox** (quando abilitato): indica il runtime in sandbox, i percorsi della sandbox e se è disponibile l’esecuzione con privilegi elevati.
- **Current Date & Time**: ora locale dell’utente, fuso orario e formato dell’ora.
- **Reply Tags**: sintassi facoltativa dei tag di risposta per i provider supportati.
- **Heartbeats**: prompt Heartbeat e comportamento di conferma, quando gli heartbeat sono abilitati per l’agente predefinito.
- **Runtime**: host, OS, node, modello, root del repo (quando rilevata), livello di ragionamento (una riga).
- **Reasoning**: livello di visibilità corrente + suggerimento per il toggle `/reasoning`.

La sezione Tooling include anche indicazioni a runtime per il lavoro di lunga durata:

- usa cron per attività future di follow-up (`check back later`, promemoria, lavoro ricorrente)
  invece di loop di sleep con `exec`, trucchi di ritardo con `yieldMs` o polling ripetuto di `process`
- usa `exec` / `process` solo per comandi che iniziano subito e continuano a essere eseguiti
  in background
- quando è abilitato il risveglio automatico al completamento, avvia il comando una sola volta e fai affidamento
  sul percorso di risveglio basato su push quando emette output o fallisce
- usa `process` per log, stato, input o intervento quando hai bisogno di
  ispezionare un comando in esecuzione
- se l’attività è più grande, preferisci `sessions_spawn`; il completamento del sotto-agente è
  basato su push e viene annunciato automaticamente al richiedente
- non fare polling di `subagents list` / `sessions_list` in un loop solo per attendere
  il completamento

Quando lo strumento sperimentale `update_plan` è abilitato, Tooling indica anche al
modello di usarlo solo per lavori non banali in più fasi, mantenere esattamente una fase
`in_progress` ed evitare di ripetere l’intero piano dopo ogni aggiornamento.

I guardrail di Safety nel prompt di sistema sono indicativi. Guidano il comportamento del modello ma non applicano policy. Usa policy degli strumenti, approvazioni exec, sandboxing e liste di autorizzazione dei canali per l’applicazione rigorosa; gli operatori possono disabilitare tutto questo intenzionalmente.

Sui canali con schede/pulsanti di approvazione nativi, il prompt runtime ora indica all’agente di affidarsi prima a tale UI di approvazione nativa. Dovrebbe includere un comando manuale `/approve` solo quando il risultato dello strumento indica che le approvazioni in chat non sono disponibili o che l’approvazione manuale è l’unico percorso.

## Modalità del prompt

OpenClaw può produrre prompt di sistema più piccoli per i sotto-agenti. Il runtime imposta un
`promptMode` per ogni esecuzione (non è una configurazione visibile all’utente):

- `full` (predefinito): include tutte le sezioni sopra.
- `minimal`: usato per i sotto-agenti; omette **Skills**, **Memory Recall**, **Aggiornamento automatico di OpenClaw**, **Alias del modello**, **Identità utente**, **Reply Tags**,
  **Messaggistica**, **Risposte silenziose** e **Heartbeats**. Tooling, **Safety**,
  Workspace, Sandbox, Current Date & Time (quando noto), Runtime e il contesto
  iniettato restano disponibili.
- `none`: restituisce solo la riga di identità di base.

Quando `promptMode=minimal`, i prompt extra iniettati sono etichettati **Subagent
Context** invece di **Group Chat Context**.

## Iniezione del bootstrap del workspace

I file di bootstrap vengono ridotti e aggiunti sotto **Project Context** in modo che il modello veda il contesto di identità e profilo senza richiedere letture esplicite:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo nei workspace completamente nuovi)
- `MEMORY.md` quando presente, altrimenti `memory.md` come fallback in minuscolo

Tutti questi file vengono **iniettati nella finestra di contesto** a ogni turno, a meno che
non si applichi un gate specifico del file. `HEARTBEAT.md` viene omesso nelle esecuzioni normali quando
gli heartbeat sono disabilitati per l’agente predefinito oppure
`agents.defaults.heartbeat.includeSystemPromptSection` è false. Mantieni concisi i file
iniettati — in particolare `MEMORY.md`, che può crescere nel tempo e portare a
un uso del contesto inaspettatamente elevato e a Compaction più frequente.

> **Nota:** i file giornalieri `memory/*.md` **non** fanno parte del normale bootstrap
> Project Context. Nei turni ordinari vengono consultati su richiesta tramite gli
> strumenti `memory_search` e `memory_get`, quindi non contano rispetto alla
> finestra di contesto a meno che il modello non li legga esplicitamente. I turni `/new` e
> `/reset` senza altri contenuti sono l’eccezione: il runtime può anteporre la memoria
> giornaliera recente come blocco una tantum di contesto iniziale per quel primo turno.

I file grandi vengono troncati con un marcatore. La dimensione massima per file è controllata da
`agents.defaults.bootstrapMaxChars` (predefinito: 12000). Il contenuto bootstrap totale iniettato
tra i file è limitato da `agents.defaults.bootstrapTotalMaxChars`
(predefinito: 60000). I file mancanti iniettano un breve marcatore di file mancante. Quando si verifica il troncamento,
OpenClaw può iniettare un blocco di avviso in Project Context; controllalo con
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
predefinito: `once`).

Le sessioni dei sotto-agenti iniettano solo `AGENTS.md` e `TOOLS.md` (gli altri file bootstrap
vengono filtrati per mantenere piccolo il contesto del sotto-agente).

Gli hook interni possono intercettare questo passaggio tramite `agent:bootstrap` per modificare o sostituire
i file bootstrap iniettati (per esempio sostituendo `SOUL.md` con una persona alternativa).

Se vuoi rendere l’agente meno generico nel modo in cui suona, inizia con
[Guida alla personalità di SOUL.md](/it/concepts/soul).

Per ispezionare quanto contribuisce ogni file iniettato (grezzo vs iniettato, troncamento, più overhead dello schema degli strumenti), usa `/context list` o `/context detail`. Vedi [Contesto](/it/concepts/context).

## Gestione dell’ora

Il prompt di sistema include una sezione dedicata **Current Date & Time** quando il
fuso orario dell’utente è noto. Per mantenere stabile la cache del prompt, ora include solo
il **fuso orario** (nessun orologio dinamico o formato dell’ora).

Usa `session_status` quando l’agente ha bisogno dell’ora corrente; la scheda di stato
include una riga con il timestamp. Lo stesso strumento può facoltativamente impostare una
sostituzione del modello per sessione (`model=default` la cancella).

Configura con:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Vedi [Data e ora](/it/date-time) per i dettagli completi sul comportamento.

## Skills

Quando esistono Skills idonee, OpenClaw inietta un compatto **elenco delle skill disponibili**
(`formatSkillsForPrompt`) che include il **percorso del file** per ciascuna skill. Il
prompt istruisce il modello a usare `read` per caricare lo SKILL.md nella posizione
elencata (workspace, gestita o bundled). Se non ci sono skill idonee, la
sezione Skills viene omessa.

L’idoneità include gate dei metadati delle skill, controlli dell’ambiente/configurazione runtime
e la allowlist effettiva delle skill dell’agente quando `agents.defaults.skills` o
`agents.list[].skills` è configurato.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

Questo mantiene piccolo il prompt di base pur consentendo un uso mirato delle skill.

Il budget dell’elenco delle skill è gestito dal sottosistema delle skill:

- Predefinito globale: `skills.limits.maxSkillsPromptChars`
- Override per agente: `agents.list[].skillsLimits.maxSkillsPromptChars`

Gli estratti runtime generici delimitati usano una superficie diversa:

- `agents.defaults.contextLimits.*`
- `agents.list[].contextLimits.*`

Questa separazione mantiene il dimensionamento delle skill separato dal dimensionamento di lettura/iniezione runtime, come
`memory_get`, risultati di strumenti live e aggiornamenti di `AGENTS.md` dopo la Compaction.

## Documentation

Quando disponibile, il prompt di sistema include una sezione **Documentation** che punta alla
directory locale della documentazione di OpenClaw (o `docs/` nel workspace del repo oppure la documentazione del pacchetto npm bundled) e menziona anche il mirror pubblico, il repo sorgente, la community Discord e
ClawHub ([https://clawhub.ai](https://clawhub.ai)) per la scoperta delle skill. Il prompt istruisce il modello a consultare prima la documentazione locale
per comportamento, comandi, configurazione o architettura di OpenClaw e a eseguire
`openclaw status` direttamente quando possibile (chiedendo all’utente solo quando non ha accesso).
