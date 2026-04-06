---
read_when:
    - Vuoi usare i modelli Anthropic in OpenClaw
summary: Usa Anthropic Claude tramite chiavi API in OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-06T03:10:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: bbc6c4938674aedf20ff944bc04e742c9a7e77a5ff10ae4f95b5718504c57c2d
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

Anthropic sviluppa la famiglia di modelli **Claude** e fornisce l'accesso tramite API.
In OpenClaw, le nuove configurazioni Anthropic dovrebbero usare una chiave API. I profili legacy
con token Anthropic esistenti continuano comunque a essere rispettati a runtime se sono già
configurati.

<Warning>
Per Anthropic in OpenClaw, la suddivisione della fatturazione è:

- **Chiave API Anthropic**: normale fatturazione dell'API Anthropic.
- **Autenticazione con abbonamento Claude dentro OpenClaw**: Anthropic ha comunicato agli utenti OpenClaw il
  **4 aprile 2026 alle 12:00 PM PT / 8:00 PM BST** che questo conta come
  utilizzo di harness di terze parti e richiede **Extra Usage** (pay-as-you-go,
  fatturato separatamente dall'abbonamento).

Le nostre riproduzioni locali confermano questa distinzione:

- `claude -p` diretto può ancora funzionare
- `claude -p --append-system-prompt ...` può attivare la protezione Extra Usage quando
  il prompt identifica OpenClaw
- lo stesso prompt di sistema simile a OpenClaw **non** riproduce il blocco nel
  percorso Anthropic SDK + `ANTHROPIC_API_KEY`

Quindi la regola pratica è: **chiave API Anthropic, oppure abbonamento Claude con
Extra Usage**. Se vuoi il percorso di produzione più chiaro, usa una chiave API Anthropic.

Documentazione pubblica attuale di Anthropic:

- [Riferimento CLI Claude Code](https://code.claude.com/docs/en/cli-reference)
- [Panoramica dell'SDK Claude Agent](https://platform.claude.com/docs/en/agent-sdk/overview)

- [Uso di Claude Code con il tuo piano Pro o Max](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Uso di Claude Code con il tuo piano Team o Enterprise](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

Se vuoi il percorso di fatturazione più chiaro, usa invece una chiave API Anthropic.
OpenClaw supporta anche altre opzioni in stile abbonamento, inclusi [OpenAI
Codex](/it/providers/openai), [Qwen Cloud Coding Plan](/it/providers/qwen),
[MiniMax Coding Plan](/it/providers/minimax) e [Z.AI / GLM Coding
Plan](/it/providers/glm).
</Warning>

## Opzione A: chiave API Anthropic

**Ideale per:** accesso API standard e fatturazione basata sull'utilizzo.
Crea la tua chiave API nella Anthropic Console.

### Configurazione CLI

```bash
openclaw onboard
# scegli: Anthropic API key

# oppure in modo non interattivo
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### Snippet di configurazione Anthropic

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Valori predefiniti del thinking (Claude 4.6)

- I modelli Anthropic Claude 4.6 usano per impostazione predefinita il thinking `adaptive` in OpenClaw quando non è impostato alcun livello di thinking esplicito.
- Puoi fare override per messaggio (`/think:<level>`) oppure nei parametri del modello:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- Documentazione Anthropic correlata:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Modalità veloce (Anthropic API)

Il toggle condiviso `/fast` di OpenClaw supporta anche il traffico Anthropic pubblico diretto, incluse richieste autenticate con chiave API e OAuth inviate a `api.anthropic.com`.

- `/fast on` corrisponde a `service_tier: "auto"`
- `/fast off` corrisponde a `service_tier: "standard_only"`
- Valore predefinito di configurazione:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

Limiti importanti:

- OpenClaw inietta i service tier Anthropic solo per richieste dirette a `api.anthropic.com`. Se instradi `anthropic/*` tramite un proxy o gateway, `/fast` lascia `service_tier` invariato.
- I parametri espliciti del modello Anthropic `serviceTier` o `service_tier` hanno la precedenza sul valore predefinito di `/fast` quando entrambi sono impostati.
- Anthropic riporta il tier effettivo nella risposta sotto `usage.service_tier`. Negli account senza capacità Priority Tier, `service_tier: "auto"` può comunque risolversi in `standard`.

## Prompt caching (Anthropic API)

OpenClaw supporta la funzionalità di prompt caching di Anthropic. È **solo API**; l'autenticazione legacy con token Anthropic non rispetta le impostazioni della cache.

### Configurazione

Usa il parametro `cacheRetention` nella configurazione del modello:

| Valore  | Durata della cache | Descrizione                      |
| ------- | ------------------ | -------------------------------- |
| `none`  | Nessuna cache      | Disabilita il prompt caching     |
| `short` | 5 minuti           | Predefinito per auth con API Key |
| `long`  | 1 ora              | Cache estesa                     |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### Valori predefiniti

Quando usi l'autenticazione con chiave API Anthropic, OpenClaw applica automaticamente `cacheRetention: "short"` (cache di 5 minuti) per tutti i modelli Anthropic. Puoi fare override di questo comportamento impostando esplicitamente `cacheRetention` nella tua configurazione.

### Override `cacheRetention` per agente

Usa i parametri a livello di modello come base, quindi fai override per agenti specifici tramite `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // base per la maggior parte degli agenti
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // override solo per questo agente
    ],
  },
}
```

Ordine di merge della configurazione per i parametri relativi alla cache:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (corrispondenza su `id`, override per chiave)

Questo consente a un agente di mantenere una cache di lunga durata mentre un altro agente sullo stesso modello disabilita la cache per evitare costi di scrittura su traffico bursty o con basso riutilizzo.

### Note su Claude in Bedrock

- I modelli Anthropic Claude su Bedrock (`amazon-bedrock/*anthropic.claude*`) accettano il pass-through di `cacheRetention` quando configurato.
- I modelli Bedrock non Anthropic vengono forzati a `cacheRetention: "none"` a runtime.
- I valori predefiniti intelligenti della chiave API Anthropic inizializzano anche `cacheRetention: "short"` per i riferimenti di modello Claude-on-Bedrock quando non è impostato alcun valore esplicito.

## Finestra di contesto da 1M (beta Anthropic)

La finestra di contesto da 1M di Anthropic è limitata dalla beta. In OpenClaw, abilitala per modello
con `params.context1m: true` per i modelli Opus/Sonnet supportati.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw la mappa a `anthropic-beta: context-1m-2025-08-07` nelle richieste Anthropic.

Questo si attiva solo quando `params.context1m` è impostato esplicitamente a `true` per
quel modello.

Requisito: Anthropic deve consentire l'uso del contesto esteso con quella credenziale
(in genere fatturazione con chiave API, oppure il percorso di login Claude di OpenClaw / autenticazione con token legacy
con Extra Usage abilitato). In caso contrario Anthropic restituisce:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

Nota: Anthropic al momento rifiuta le richieste beta `context-1m-*` quando si usa
l'autenticazione legacy con token Anthropic (`sk-ant-oat-*`). Se configuri
`context1m: true` con quella modalità di autenticazione legacy, OpenClaw registra un avviso e
torna alla finestra di contesto standard saltando l'header beta context1m
pur mantenendo le beta OAuth richieste.

## Rimosso: backend Claude CLI

Il backend bundled Anthropic `claude-cli` è stato rimosso.

- L'avviso di Anthropic del 4 aprile 2026 dice che il traffico guidato da OpenClaw con login Claude è
  utilizzo di harness di terze parti e richiede **Extra Usage**.
- Le nostre riproduzioni locali mostrano inoltre che l'uso diretto di
  `claude -p --append-system-prompt ...` può incorrere nella stessa protezione quando il
  prompt aggiunto identifica OpenClaw.
- Lo stesso prompt di sistema simile a OpenClaw non attiva quella protezione nel
  percorso Anthropic SDK + `ANTHROPIC_API_KEY`.
- Usa chiavi API Anthropic per il traffico Anthropic in OpenClaw.

## Note

- La documentazione pubblica di Anthropic per Claude Code continua a documentare l'uso diretto della CLI come
  `claude -p`, ma il distinto avviso di Anthropic agli utenti OpenClaw dice che il
  percorso di login Claude di **OpenClaw** è utilizzo di harness di terze parti e richiede
  **Extra Usage** (pay-as-you-go fatturato separatamente dall'abbonamento).
  Le nostre riproduzioni locali mostrano inoltre che l'uso diretto di
  `claude -p --append-system-prompt ...` può incorrere nella stessa protezione quando il
  prompt aggiunto identifica OpenClaw, mentre la stessa forma di prompt non
  si riproduce nel percorso Anthropic SDK + `ANTHROPIC_API_KEY`. Per la produzione,
  consigliamo invece le chiavi API Anthropic.
- Il setup-token Anthropic è di nuovo disponibile in OpenClaw come percorso legacy/manuale. L'avviso di fatturazione di Anthropic specifico per OpenClaw continua comunque ad applicarsi, quindi usalo sapendo che Anthropic richiede **Extra Usage** per questo percorso.
- I dettagli di autenticazione + le regole di riutilizzo sono in [/concepts/oauth](/it/concepts/oauth).

## Risoluzione dei problemi

**Errori 401 / token improvvisamente non valido**

- L'autenticazione legacy con token Anthropic può scadere o essere revocata.
- Per le nuove configurazioni, passa a una chiave API Anthropic.

**Nessuna chiave API trovata per il provider "anthropic"**

- L'autenticazione è **per agente**. I nuovi agenti non ereditano le chiavi dell'agente principale.
- Esegui di nuovo l'onboarding per quell'agente, oppure configura una chiave API sull'host
  gateway, quindi verifica con `openclaw models status`.

**Nessuna credenziale trovata per il profilo `anthropic:default`**

- Esegui `openclaw models status` per vedere quale profilo di autenticazione è attivo.
- Esegui di nuovo l'onboarding, oppure configura una chiave API per quel percorso di profilo.

**Nessun profilo di autenticazione disponibile (tutti in cooldown/non disponibili)**

- Controlla `openclaw models status --json` per `auth.unusableProfiles`.
- I cooldown per rate limit Anthropic possono essere limitati al modello, quindi un modello Anthropic
  correlato potrebbe essere ancora utilizzabile anche quando quello corrente è in cooldown.
- Aggiungi un altro profilo Anthropic oppure attendi la fine del cooldown.

Altro: [/gateway/troubleshooting](/it/gateway/troubleshooting) e [/help/faq](/it/help/faq).
