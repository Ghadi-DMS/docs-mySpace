---
read_when:
    - Aggiunta o modifica della CLI Models (models list/set/scan/aliases/fallbacks)
    - Modifica del comportamento di fallback del modello o dell'esperienza utente di selezione
    - Aggiornamento delle probe di scansione dei modelli (strumenti/immagini)
summary: 'CLI Models: elenco, impostazione, alias, fallback, scansione, stato'
title: CLI Models
x-i18n:
    generated_at: "2026-04-06T03:07:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299602ccbe0c3d6bbdb2deab22bc60e1300ef6843ed0b8b36be574cc0213c155
    source_path: concepts/models.md
    workflow: 15
---

# CLI Models

Vedi [/concepts/model-failover](/it/concepts/model-failover) per la rotazione dei
profili di autenticazione, i cooldown e come interagiscono con i fallback.
Panoramica rapida dei provider + esempi: [/concepts/model-providers](/it/concepts/model-providers).

## Come funziona la selezione del modello

OpenClaw seleziona i modelli in questo ordine:

1. Modello **primario** (`agents.defaults.model.primary` o `agents.defaults.model`).
2. **Fallback** in `agents.defaults.model.fallbacks` (in ordine).
3. Il **failover dell'autenticazione del provider** avviene all'interno di un provider prima di passare al
   modello successivo.

Correlati:

- `agents.defaults.models` è la allowlist/il catalogo dei modelli che OpenClaw può usare (più gli alias).
- `agents.defaults.imageModel` viene usato **solo quando** il modello primario non può accettare immagini.
- `agents.defaults.pdfModel` viene usato dallo strumento `pdf`. Se omesso, lo strumento
  usa come fallback `agents.defaults.imageModel`, quindi il modello di sessione/predefinito risolto.
- `agents.defaults.imageGenerationModel` viene usato dalla capacità condivisa di generazione immagini. Se omesso, `image_generate` può comunque dedurre un provider predefinito supportato da autenticazione. Prova prima il provider predefinito corrente, poi i restanti provider di generazione immagini registrati in ordine di ID provider. Se imposti un provider/modello specifico, configura anche l'autenticazione/la chiave API di quel provider.
- `agents.defaults.musicGenerationModel` viene usato dalla capacità condivisa di generazione musicale. Se omesso, `music_generate` può comunque dedurre un provider predefinito supportato da autenticazione. Prova prima il provider predefinito corrente, poi i restanti provider di generazione musicale registrati in ordine di ID provider. Se imposti un provider/modello specifico, configura anche l'autenticazione/la chiave API di quel provider.
- `agents.defaults.videoGenerationModel` viene usato dalla capacità condivisa di generazione video. Se omesso, `video_generate` può comunque dedurre un provider predefinito supportato da autenticazione. Prova prima il provider predefinito corrente, poi i restanti provider di generazione video registrati in ordine di ID provider. Se imposti un provider/modello specifico, configura anche l'autenticazione/la chiave API di quel provider.
- I valori predefiniti per agente possono sovrascrivere `agents.defaults.model` tramite `agents.list[].model` più i binding (vedi [/concepts/multi-agent](/it/concepts/multi-agent)).

## Policy rapida per i modelli

- Imposta come primario il modello di ultima generazione più potente a tua disposizione.
- Usa i fallback per attività sensibili a costo/latenza e chat a minor criticità.
- Per agenti con strumenti abilitati o input non attendibili, evita livelli di modello più vecchi/deboli.

## Onboarding (consigliato)

Se non vuoi modificare manualmente la configurazione, esegui l'onboarding:

```bash
openclaw onboard
```

Può configurare modello + autenticazione per provider comuni, inclusi **OpenAI Code (Codex)
subscription** (OAuth) e **Anthropic** (chiave API o Claude CLI).

## Chiavi di configurazione (panoramica)

- `agents.defaults.model.primary` e `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` e `agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` e `agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` e `agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` e `agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models` (allowlist + alias + parametri provider)
- `models.providers` (provider personalizzati scritti in `models.json`)

I riferimenti ai modelli sono normalizzati in minuscolo. Gli alias dei provider come `z.ai/*` vengono normalizzati
in `zai/*`.

Esempi di configurazione dei provider (incluso OpenCode) si trovano in
[/providers/opencode](/it/providers/opencode).

## "Model is not allowed" (e perché le risposte si interrompono)

Se `agents.defaults.models` è impostato, diventa la **allowlist** per `/model` e per
gli override di sessione. Quando un utente seleziona un modello che non è in quella allowlist,
OpenClaw restituisce:

```
Model "provider/model" is not allowed. Use /model to list available models.
```

Questo accade **prima** che venga generata una normale risposta, quindi il messaggio può dare la sensazione
che “non abbia risposto”. La soluzione è una delle seguenti:

- Aggiungere il modello a `agents.defaults.models`, oppure
- Cancellare la allowlist (rimuovere `agents.defaults.models`), oppure
- Scegliere un modello da `/model list`.

Esempio di configurazione della allowlist:

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-6" },
    models: {
      "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## Cambiare modello in chat (`/model`)

Puoi cambiare i modelli per la sessione corrente senza riavviare:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

Note:

- `/model` (e `/model list`) è un selettore compatto e numerato (famiglia di modelli + provider disponibili).
- Su Discord, `/model` e `/models` aprono un selettore interattivo con menu a discesa per provider e modello più un passaggio di invio.
- `/model <#>` seleziona dal selettore.
- `/model` rende immediatamente persistente la nuova selezione della sessione.
- Se l'agente è inattivo, l'esecuzione successiva usa subito il nuovo modello.
- Se un'esecuzione è già attiva, OpenClaw contrassegna uno switch live come in sospeso e riavvia nel nuovo modello solo in un punto di retry pulito.
- Se l'attività degli strumenti o l'output della risposta sono già iniziati, lo switch in sospeso può restare in coda fino a una successiva opportunità di retry o al turno utente seguente.
- `/model status` è la vista dettagliata (candidati di autenticazione e, quando configurati, `baseUrl` dell'endpoint provider + modalità `api`).
- I riferimenti ai modelli vengono analizzati separando sul **primo** `/`. Usa `provider/model` quando digiti `/model <ref>`.
- Se l'ID del modello contiene a sua volta `/` (stile OpenRouter), devi includere il prefisso del provider (esempio: `/model openrouter/moonshotai/kimi-k2`).
- Se ometti il provider, OpenClaw risolve l'input in questo ordine:
  1. corrispondenza alias
  2. corrispondenza univoca del provider configurato per quell'esatto ID modello senza prefisso
  3. fallback deprecato al provider predefinito configurato
     Se quel provider non espone più il modello predefinito configurato, OpenClaw
     usa invece come fallback il primo provider/modello configurato per evitare
     di mostrare un valore predefinito obsoleto di un provider rimosso.

Comportamento/configurazione completa del comando: [Comandi slash](/it/tools/slash-commands).

## Comandi CLI

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models` (senza sottocomando) è una scorciatoia per `models status`.

### `models list`

Mostra per impostazione predefinita i modelli configurati. Flag utili:

- `--all`: catalogo completo
- `--local`: solo provider locali
- `--provider <name>`: filtra per provider
- `--plain`: un modello per riga
- `--json`: output leggibile dalla macchina

### `models status`

Mostra il modello primario risolto, i fallback, il modello immagine e una panoramica dell'autenticazione
dei provider configurati. Mostra anche lo stato di scadenza OAuth per i profili trovati
nell'archivio di autenticazione (avviso entro 24h per impostazione predefinita). `--plain` stampa solo il
modello primario risolto.
Lo stato OAuth viene sempre mostrato (ed è incluso nell'output `--json`). Se un provider configurato
non ha credenziali, `models status` stampa una sezione **Autenticazione mancante**.
Il JSON include `auth.oauth` (finestra di avviso + profili) e `auth.providers`
(autenticazione effettiva per provider, incluse le credenziali supportate da env). `auth.oauth`
riguarda solo lo stato dei profili nell'archivio di autenticazione; i provider solo-env non compaiono lì.
Usa `--check` per l'automazione (uscita `1` se mancante/scaduto, `2` se in scadenza).
Usa `--probe` per controlli live dell'autenticazione; le righe della probe possono provenire da profili di autenticazione, credenziali env
o `models.json`.
Se `auth.order.<provider>` esplicito omette un profilo archiviato, la probe riporta
`excluded_by_auth_order` invece di provarlo. Se l'autenticazione esiste ma non può essere risolto alcun
modello sondabile per quel provider, la probe riporta `status: no_model`.

La scelta dell'autenticazione dipende da provider/account. Per host gateway sempre attivi, le chiavi API
sono di solito la soluzione più prevedibile; sono supportati anche il riuso di Claude CLI e i profili OAuth/token Anthropic esistenti.

Esempio (Claude CLI):

```bash
claude auth login
openclaw models status
```

## Scansione (modelli gratuiti OpenRouter)

`openclaw models scan` ispeziona il **catalogo di modelli gratuiti** di OpenRouter e può
facoltativamente sondare i modelli per il supporto a strumenti e immagini.

Flag principali:

- `--no-probe`: salta le probe live (solo metadati)
- `--min-params <b>`: dimensione minima dei parametri (miliardi)
- `--max-age-days <days>`: salta i modelli più vecchi
- `--provider <name>`: filtro per prefisso provider
- `--max-candidates <n>`: dimensione dell'elenco fallback
- `--set-default`: imposta `agents.defaults.model.primary` sulla prima selezione
- `--set-image`: imposta `agents.defaults.imageModel.primary` sulla prima selezione immagine

Le probe richiedono una chiave API OpenRouter (da profili di autenticazione o
`OPENROUTER_API_KEY`). Senza una chiave, usa `--no-probe` per elencare solo i candidati.

I risultati della scansione sono classificati in base a:

1. Supporto immagini
2. Latenza degli strumenti
3. Dimensione del contesto
4. Conteggio dei parametri

Input

- Elenco OpenRouter `/models` (filtro `:free`)
- Richiede una chiave API OpenRouter dai profili di autenticazione o `OPENROUTER_API_KEY` (vedi [/environment](/it/help/environment))
- Filtri facoltativi: `--max-age-days`, `--min-params`, `--provider`, `--max-candidates`
- Controlli probe: `--timeout`, `--concurrency`

Quando viene eseguito in un TTY, puoi selezionare i fallback in modo interattivo. In modalità non interattiva,
passa `--yes` per accettare i valori predefiniti.

## Registro dei modelli (`models.json`)

I provider personalizzati in `models.providers` vengono scritti in `models.json` sotto la
directory dell'agente (predefinita `~/.openclaw/agents/<agentId>/agent/models.json`). Questo file
viene unito per impostazione predefinita a meno che `models.mode` non sia impostato su `replace`.

Precedenza della modalità merge per ID provider corrispondenti:

- `baseUrl` non vuoto già presente nel `models.json` dell'agente ha la precedenza.
- `apiKey` non vuoto nel `models.json` dell'agente ha la precedenza solo quando quel provider non è gestito da SecretRef nel contesto corrente di configurazione/profilo di autenticazione.
- I valori `apiKey` dei provider gestiti da SecretRef vengono aggiornati dai marker di origine (`ENV_VAR_NAME` per riferimenti env, `secretref-managed` per riferimenti file/exec) invece di rendere persistenti i secret risolti.
- I valori header dei provider gestiti da SecretRef vengono aggiornati dai marker di origine (`secretref-env:ENV_VAR_NAME` per riferimenti env, `secretref-managed` per riferimenti file/exec).
- `apiKey`/`baseUrl` dell'agente vuoti o mancanti usano come fallback `models.providers` della configurazione.
- Gli altri campi del provider vengono aggiornati dalla configurazione e dai dati di catalogo normalizzati.

La persistenza dei marker è autorevole rispetto alla fonte: OpenClaw scrive i marker dallo snapshot di configurazione della fonte attiva (pre-risoluzione), non dai valori secret runtime risolti.
Questo vale ogni volta che OpenClaw rigenera `models.json`, inclusi i percorsi guidati da comando come `openclaw agent`.

## Correlati

- [Model Providers](/it/concepts/model-providers) — instradamento provider e autenticazione
- [Model Failover](/it/concepts/model-failover) — catene di fallback
- [Image Generation](/it/tools/image-generation) — configurazione del modello immagine
- [Music Generation](/tools/music-generation) — configurazione del modello musicale
- [Video Generation](/tools/video-generation) — configurazione del modello video
- [Configuration Reference](/it/gateway/configuration-reference#agent-defaults) — chiavi di configurazione del modello
