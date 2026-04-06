---
read_when:
    - Vuoi comprendere OpenClaw OAuth end-to-end
    - Hai riscontrato problemi di invalidazione del token / logout
    - Vuoi i flussi di autenticazione Claude CLI o OAuth
    - Vuoi più account o instradamento per profilo
summary: 'OAuth in OpenClaw: scambio dei token, archiviazione e modelli multi-account'
title: OAuth
x-i18n:
    generated_at: "2026-04-06T03:06:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402e20dfeb6ae87a90cba5824a56a7ba3b964f3716508ea5cc48a47e5affdd73
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

OpenClaw supporta la “subscription auth” tramite OAuth per i provider che la offrono
(in particolare **OpenAI Codex (ChatGPT OAuth)**). Per Anthropic, la distinzione
pratica ora è:

- **Chiave API Anthropic**: normale fatturazione API Anthropic
- **Subscription auth Anthropic dentro OpenClaw**: Anthropic ha notificato agli utenti
  di OpenClaw il **4 aprile 2026 alle 12:00 PM PT / 8:00 PM BST** che ora questo
  richiede **Extra Usage**

OpenAI Codex OAuth è supportato esplicitamente per l'uso in strumenti esterni come
OpenClaw. Questa pagina spiega:

Per Anthropic in produzione, l'autenticazione con chiave API è il percorso consigliato più sicuro.

- come funziona lo **scambio** dei token OAuth (PKCE)
- dove vengono **archiviati** i token (e perché)
- come gestire **più account** (profili + override per sessione)

OpenClaw supporta anche **plugin provider** che includono i propri flussi OAuth o
con chiave API. Eseguili con:

```bash
openclaw models auth login --provider <id>
```

## Il token sink (perché esiste)

I provider OAuth spesso emettono un **nuovo refresh token** durante i flussi di login/refresh. Alcuni provider (o client OAuth) possono invalidare i refresh token più vecchi quando ne viene emesso uno nuovo per lo stesso utente/app.

Sintomo pratico:

- accedi tramite OpenClaw _e_ tramite Claude Code / Codex CLI → uno dei due viene casualmente “disconnesso” più tardi

Per ridurre questo problema, OpenClaw tratta `auth-profiles.json` come un **token sink**:

- il runtime legge le credenziali da **un solo posto**
- possiamo mantenere più profili e instradarli in modo deterministico
- quando le credenziali vengono riutilizzate da una CLI esterna come Codex CLI, OpenClaw
  le rispecchia con la provenienza e rilegge quella sorgente esterna invece di
  ruotare direttamente il refresh token

## Archiviazione (dove vivono i token)

I segreti vengono archiviati **per agente**:

- Profili auth (OAuth + chiavi API + riferimenti opzionali a livello di valore): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- File legacy di compatibilità: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (le voci statiche `api_key` vengono ripulite quando vengono rilevate)

File legacy solo per importazione (ancora supportato, ma non è l'archivio principale):

- `~/.openclaw/credentials/oauth.json` (importato in `auth-profiles.json` al primo utilizzo)

Tutto quanto sopra rispetta anche `$OPENCLAW_STATE_DIR` (override della directory di stato). Riferimento completo: [/gateway/configuration](/it/gateway/configuration-reference#auth-storage)

Per i riferimenti statici ai segreti e il comportamento di attivazione degli snapshot a runtime, vedi [Gestione dei segreti](/it/gateway/secrets).

## Compatibilità legacy dei token Anthropic

<Warning>
La documentazione pubblica di Claude Code di Anthropic dice che l'uso diretto di Claude Code resta entro
i limiti dell'abbonamento Claude. Separatamente, Anthropic ha comunicato agli utenti di OpenClaw il
**4 aprile 2026 alle 12:00 PM PT / 8:00 PM BST** che **OpenClaw conta come
un harness di terze parti**. I profili token Anthropic esistenti restano tecnicamente
utilizzabili in OpenClaw, ma Anthropic afferma che il percorso OpenClaw ora richiede **Extra
Usage** (pay-as-you-go fatturato separatamente dall'abbonamento) per quel
traffico.

Per la documentazione attuale di Anthropic sui piani direct-Claude-Code, vedi [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
e [Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

Se vuoi altre opzioni in stile abbonamento in OpenClaw, vedi [OpenAI
Codex](/it/providers/openai), [Qwen Cloud Coding
Plan](/it/providers/qwen), [MiniMax Coding Plan](/it/providers/minimax),
e [Z.AI / GLM Coding Plan](/it/providers/glm).
</Warning>

OpenClaw ora espone di nuovo setup-token Anthropic come percorso legacy/manuale.
L'avviso di fatturazione specifico per OpenClaw di Anthropic continua ad applicarsi a quel percorso, quindi
usalo con l'aspettativa che Anthropic richieda **Extra Usage** per
il traffico Claude-login guidato da OpenClaw.

## Migrazione Claude CLI Anthropic

Anthropic non ha più un percorso di migrazione locale Claude CLI supportato in
OpenClaw. Usa chiavi API Anthropic per il traffico Anthropic, oppure mantieni l'autenticazione legacy
basata su token solo dove è già configurata e con l'aspettativa
che Anthropic tratti quel percorso OpenClaw come **Extra Usage**.

## Scambio OAuth (come funziona il login)

I flussi di login interattivi di OpenClaw sono implementati in `@mariozechner/pi-ai` e collegati ai wizard/comandi.

### Setup-token Anthropic

Forma del flusso:

1. avvia Anthropic setup-token o paste-token da OpenClaw
2. OpenClaw archivia la credenziale Anthropic risultante in un profilo auth
3. la selezione del modello resta su `anthropic/...`
4. i profili auth Anthropic esistenti restano disponibili per rollback/controllo dell'ordine

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth è supportato esplicitamente per l'uso al di fuori di Codex CLI, inclusi i flussi di lavoro OpenClaw.

Forma del flusso (PKCE):

1. genera verifier/challenge PKCE + `state` casuale
2. apri `https://auth.openai.com/oauth/authorize?...`
3. prova a catturare il callback su `http://127.0.0.1:1455/auth/callback`
4. se non è possibile associare il callback (o se sei remoto/headless), incolla l'URL di reindirizzamento/il codice
5. esegui lo scambio su `https://auth.openai.com/oauth/token`
6. estrai `accountId` dal token di accesso e archivia `{ access, refresh, expires, accountId }`

Il percorso del wizard è `openclaw onboard` → scelta auth `openai-codex`.

## Refresh + scadenza

I profili archiviano un timestamp `expires`.

A runtime:

- se `expires` è nel futuro → usa il token di accesso archiviato
- se è scaduto → esegui il refresh (con un lock sul file) e sovrascrivi le credenziali archiviate
- eccezione: le credenziali riutilizzate da una CLI esterna restano gestite esternamente; OpenClaw
  rilegge l'archivio auth della CLI e non usa mai direttamente il refresh token copiato

Il flusso di refresh è automatico; in genere non è necessario gestire i token manualmente.

## Più account (profili) + instradamento

Due modelli:

### 1) Preferito: agenti separati

Se vuoi che “personale” e “lavoro” non interagiscano mai, usa agenti isolati (sessioni + credenziali + workspace separati):

```bash
openclaw agents add work
openclaw agents add personal
```

Poi configura l'autenticazione per agente (wizard) e instrada le chat verso l'agente corretto.

### 2) Avanzato: più profili in un solo agente

`auth-profiles.json` supporta più ID profilo per lo stesso provider.

Scegli quale profilo usare:

- globalmente tramite l'ordinamento della configurazione (`auth.order`)
- per sessione tramite `/model ...@<profileId>`

Esempio (override di sessione):

- `/model Opus@anthropic:work`

Come vedere quali ID profilo esistono:

- `openclaw channels list --json` (mostra `auth[]`)

Documenti correlati:

- [/concepts/model-failover](/it/concepts/model-failover) (regole di rotazione + cooldown)
- [/tools/slash-commands](/it/tools/slash-commands) (superficie dei comandi)

## Correlati

- [Autenticazione](/it/gateway/authentication) — panoramica dell'autenticazione dei provider di modelli
- [Segreti](/it/gateway/secrets) — archiviazione delle credenziali e SecretRef
- [Riferimento di configurazione](/it/gateway/configuration-reference#auth-storage) — chiavi di configurazione auth
