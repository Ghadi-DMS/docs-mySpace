---
read_when:
    - Vuoi configurare QMD come backend della memoria
    - Vuoi funzionalità di memoria avanzate come il reranking o percorsi indicizzati aggiuntivi
summary: Sidecar di ricerca local-first con BM25, vettori, reranking ed espansione delle query
title: Motore di memoria QMD
x-i18n:
    generated_at: "2026-04-06T03:06:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 36642c7df94b88f562745dd2270334379f2aeeef4b363a8c13ef6be42dadbe5c
    source_path: concepts/memory-qmd.md
    workflow: 15
---

# Motore di memoria QMD

[QMD](https://github.com/tobi/qmd) è un sidecar di ricerca local-first che viene eseguito
insieme a OpenClaw. Combina BM25, ricerca vettoriale e reranking in un singolo
binario e può indicizzare contenuti oltre ai file di memoria del tuo workspace.

## Cosa aggiunge rispetto al motore integrato

- **Reranking ed espansione delle query** per un richiamo migliore.
- **Indicizzazione di directory aggiuntive** -- documentazione del progetto, note del team, qualsiasi cosa su disco.
- **Indicizzazione delle trascrizioni di sessione** -- per richiamare conversazioni precedenti.
- **Completamente locale** -- viene eseguito tramite Bun + node-llama-cpp, scarica automaticamente i modelli GGUF.
- **Fallback automatico** -- se QMD non è disponibile, OpenClaw torna al
  motore integrato senza interruzioni.

## Per iniziare

### Prerequisiti

- Installa QMD: `npm install -g @tobilu/qmd` oppure `bun install -g @tobilu/qmd`
- Build di SQLite che consenta le estensioni (`brew install sqlite` su macOS).
- QMD deve trovarsi nel `PATH` del gateway.
- macOS e Linux funzionano subito. Windows è supportato al meglio tramite WSL2.

### Abilitazione

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw crea una home QMD autonoma in
`~/.openclaw/agents/<agentId>/qmd/` e gestisce automaticamente il ciclo di vita
del sidecar -- raccolte, aggiornamenti ed esecuzioni di embedding sono gestiti per te.
Privilegia le forme attuali delle raccolte QMD e delle query MCP, ma continua a usare il fallback a
flag legacy `--mask` per le raccolte e a nomi di strumenti MCP più vecchi quando necessario.

## Come funziona il sidecar

- OpenClaw crea raccolte dai file di memoria del tuo workspace e da qualsiasi
  `memory.qmd.paths` configurato, quindi esegue `qmd update` + `qmd embed` all'avvio
  e periodicamente (impostazione predefinita: ogni 5 minuti).
- L'aggiornamento all'avvio viene eseguito in background, quindi l'avvio della chat non è bloccato.
- Le ricerche usano il `searchMode` configurato (predefinito: `search`; supporta anche
  `vsearch` e `query`). Se una modalità fallisce, OpenClaw riprova con `qmd query`.
- Se QMD fallisce completamente, OpenClaw torna al motore SQLite integrato.

<Info>
La prima ricerca potrebbe essere lenta -- QMD scarica automaticamente i modelli GGUF (~2 GB) per
il reranking e l'espansione delle query alla prima esecuzione di `qmd query`.
</Info>

## Override dei modelli

Le variabili d'ambiente dei modelli QMD vengono passate inalterate dal processo
gateway, quindi puoi ottimizzare QMD globalmente senza aggiungere nuova configurazione OpenClaw:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Dopo aver cambiato il modello di embedding, riesegui gli embedding in modo che l'indice corrisponda al
nuovo spazio vettoriale.

## Indicizzazione di percorsi aggiuntivi

Indirizza QMD verso directory aggiuntive per renderle ricercabili:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Gli snippet dai percorsi aggiuntivi appaiono come `qmd/<collection>/<relative-path>` nei
risultati di ricerca. `memory_get` riconosce questo prefisso e legge dalla root della raccolta corretta.

## Indicizzazione delle trascrizioni di sessione

Abilita l'indicizzazione delle sessioni per richiamare conversazioni precedenti:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Le trascrizioni vengono esportate come turni Utente/Assistente sanitizzati in una raccolta QMD
dedicata in `~/.openclaw/agents/<id>/qmd/sessions/`.

## Ambito della ricerca

Per impostazione predefinita, i risultati di ricerca QMD vengono mostrati solo nelle sessioni DM (non nei gruppi o
nei canali). Configura `memory.qmd.scope` per modificarlo:

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

Quando l'ambito nega una ricerca, OpenClaw registra un avviso con il canale derivato e
il tipo di chat, così i risultati vuoti sono più facili da diagnosticare.

## Citazioni

Quando `memory.citations` è `auto` o `on`, gli snippet di ricerca includono un
piè di pagina `Source: <path#line>`. Imposta `memory.citations = "off"` per omettere il piè di pagina
continuando comunque a passare internamente il percorso all'agente.

## Quando usarlo

Scegli QMD quando hai bisogno di:

- Reranking per risultati di qualità più alta.
- Cercare documentazione di progetto o note fuori dal workspace.
- Richiamare conversazioni di sessioni passate.
- Ricerca completamente locale senza chiavi API.

Per configurazioni più semplici, il [motore integrato](/it/concepts/memory-builtin) funziona bene
senza dipendenze aggiuntive.

## Risoluzione dei problemi

**QMD non trovato?** Assicurati che il binario sia nel `PATH` del gateway. Se OpenClaw
viene eseguito come servizio, crea un collegamento simbolico:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

**Prima ricerca molto lenta?** QMD scarica i modelli GGUF al primo utilizzo. Pre-riscalda
con `qmd query "test"` usando le stesse directory XDG usate da OpenClaw.

**La ricerca va in timeout?** Aumenta `memory.qmd.limits.timeoutMs` (predefinito: 4000ms).
Impostalo a `120000` per hardware più lento.

**Risultati vuoti nelle chat di gruppo?** Controlla `memory.qmd.scope` -- per impostazione predefinita consente solo
le sessioni DM.

**Repository temporanei visibili dal workspace che causano `ENAMETOOLONG` o indicizzazione non funzionante?**
L'attraversamento QMD attualmente segue il comportamento dello scanner QMD sottostante anziché
le regole dei symlink del motore integrato di OpenClaw. Mantieni i checkout temporanei di monorepo in
directory nascoste come `.tmp/` o fuori dalle root QMD indicizzate finché QMD non esporrà
un attraversamento sicuro rispetto ai cicli o controlli di esclusione espliciti.

## Configurazione

Per la superficie di configurazione completa (`memory.qmd.*`), modalità di ricerca, intervalli di
aggiornamento, regole di ambito e tutte le altre opzioni, vedi il
[Riferimento alla configurazione della memoria](/it/reference/memory-config).
