---
read_when:
    - Debug dell'autenticazione dei modelli o della scadenza OAuth
    - Documentazione dell'autenticazione o dell'archiviazione delle credenziali
summary: 'Autenticazione dei modelli: OAuth, chiavi API e legacy setup-token di Anthropic'
title: Autenticazione
x-i18n:
    generated_at: "2026-04-06T03:07:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: f59ede3fcd7e692ad4132287782a850526acf35474b5bfcea29e0e23610636c2
    source_path: gateway/authentication.md
    workflow: 15
---

# Autenticazione (provider di modelli)

<Note>
Questa pagina copre l'autenticazione dei **provider di modelli** (chiavi API, OAuth e legacy setup-token di Anthropic). Per l'autenticazione della **connessione al gateway** (token, password, trusted-proxy), consulta [Configuration](/it/gateway/configuration) e [Trusted Proxy Auth](/it/gateway/trusted-proxy-auth).
</Note>

OpenClaw supporta OAuth e chiavi API per i provider di modelli. Per host gateway
sempre attivi, le chiavi API sono in genere l'opzione più prevedibile. Sono
supportati anche i flussi subscription/OAuth quando corrispondono al modello di account del provider.

Consulta [/concepts/oauth](/it/concepts/oauth) per il flusso OAuth completo e il layout
di archiviazione.
Per l'autenticazione basata su SecretRef (provider `env`/`file`/`exec`), consulta [Gestione dei secret](/it/gateway/secrets).
Per le regole di idoneità delle credenziali e dei codici motivo usate da `models status --probe`, consulta
[Semantica delle credenziali di autenticazione](/it/auth-credential-semantics).

## Configurazione consigliata (chiave API, qualsiasi provider)

Se stai eseguendo un gateway a lunga durata, inizia con una chiave API per il provider
scelto.
Per Anthropic in particolare, l'autenticazione con chiave API è il percorso sicuro. L'autenticazione in stile
subscription di Anthropic in OpenClaw è il percorso legacy setup-token e
deve essere trattata come un percorso di **Extra Usage**, non come un percorso di limiti del piano.

1. Crea una chiave API nella console del tuo provider.
2. Inseriscila sull'**host gateway** (la macchina che esegue `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Se il Gateway viene eseguito sotto systemd/launchd, è preferibile inserire la chiave in
   `~/.openclaw/.env` così il demone può leggerla:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

Quindi riavvia il demone (o riavvia il processo Gateway) e ricontrolla:

```bash
openclaw models status
openclaw doctor
```

Se preferisci non gestire personalmente le variabili env, l'onboarding può archiviare
le chiavi API per l'uso da parte del demone: `openclaw onboard`.

Consulta [Help](/it/help) per i dettagli sull'ereditarietà env (`env.shellEnv`,
`~/.openclaw/.env`, systemd/launchd).

## Anthropic: compatibilità legacy dei token

L'autenticazione con setup-token di Anthropic è ancora disponibile in OpenClaw come
percorso legacy/manuale. La documentazione pubblica di Claude Code di Anthropic continua a descrivere l'uso diretto
di Claude Code nel terminale con i piani Claude, ma Anthropic ha comunicato separatamente agli utenti di
OpenClaw che il percorso di login Claude di **OpenClaw** è considerato utilizzo di harness di terze parti e richiede **Extra Usage** fatturato separatamente
dall'abbonamento.

Per il percorso di configurazione più chiaro, usa una chiave API Anthropic. Se devi mantenere
un percorso Anthropic in stile subscription in OpenClaw, usa il percorso legacy setup-token
con l'aspettativa che Anthropic lo tratti come **Extra Usage**.

Inserimento manuale del token (qualsiasi provider; scrive `auth-profiles.json` + aggiorna la configurazione):

```bash
openclaw models auth paste-token --provider openrouter
```

Sono supportati anche i riferimenti ai profili di autenticazione per credenziali statiche:

- le credenziali `api_key` possono usare `keyRef: { source, provider, id }`
- le credenziali `token` possono usare `tokenRef: { source, provider, id }`
- i profili in modalità OAuth non supportano credenziali SecretRef; se `auth.profiles.<id>.mode` è impostato su `"oauth"`, l'input `keyRef`/`tokenRef` supportato da SecretRef per quel profilo viene rifiutato.

Controllo adatto all'automazione (uscita `1` se scaduto/mancante, `2` se in scadenza):

```bash
openclaw models status --check
```

Probe di autenticazione live:

```bash
openclaw models status --probe
```

Note:

- Le righe del probe possono provenire da profili di autenticazione, credenziali env o `models.json`.
- Se `auth.order.<provider>` esplicito omette un profilo archiviato, il probe riporta
  `excluded_by_auth_order` per quel profilo invece di provarlo.
- Se l'autenticazione esiste ma OpenClaw non riesce a risolvere un candidato modello sondabile per
  quel provider, il probe riporta `status: no_model`.
- I cooldown dei limiti di velocità possono essere limitati a un modello. Un profilo in cooldown per un
  modello può comunque essere utilizzabile per un modello sibling sullo stesso provider.

Gli script operativi opzionali (systemd/Termux) sono documentati qui:
[Script di monitoraggio dell'autenticazione](/it/help/scripts#auth-monitoring-scripts)

## Nota su Anthropic

Il backend Anthropic `claude-cli` è stato rimosso.

- Usa chiavi API Anthropic per il traffico Anthropic in OpenClaw.
- Il setup-token di Anthropic resta un percorso legacy/manuale e deve essere usato con
  l'aspettativa di fatturazione Extra Usage che Anthropic ha comunicato agli utenti OpenClaw.
- `openclaw doctor` ora rileva stato Anthropic Claude CLI obsoleto e rimosso. Se
  i byte delle credenziali archiviate esistono ancora, doctor li riconverte in
  profili token/OAuth Anthropic. In caso contrario, doctor rimuove la configurazione Claude CLI
  obsoleta e ti indirizza al recupero tramite chiave API o setup-token.

## Verifica dello stato dell'autenticazione dei modelli

```bash
openclaw models status
openclaw doctor
```

## Comportamento di rotazione delle chiavi API (gateway)

Alcuni provider supportano il nuovo tentativo di una richiesta con chiavi alternative quando una chiamata API
raggiunge un limite di velocità del provider.

- Ordine di priorità:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (singolo override)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- I provider Google includono anche `GOOGLE_API_KEY` come fallback aggiuntivo.
- Lo stesso elenco di chiavi viene deduplicato prima dell'uso.
- OpenClaw ritenta con la chiave successiva solo per errori di rate limit (per esempio
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent
requests`, `ThrottlingException`, `concurrency limit reached` o
  `workers_ai ... quota limit exceeded`).
- Gli errori diversi dal rate limit non vengono ritentati con chiavi alternative.
- Se tutte le chiavi falliscono, viene restituito l'errore finale dell'ultimo tentativo.

## Controllo della credenziale usata

### Per sessione (comando chat)

Usa `/model <alias-or-id>@<profileId>` per fissare una credenziale specifica del provider per la sessione corrente (ID profilo di esempio: `anthropic:default`, `anthropic:work`).

Usa `/model` (o `/model list`) per un selettore compatto; usa `/model status` per la vista completa (candidati + profilo di autenticazione successivo, oltre ai dettagli dell'endpoint provider quando configurati).

### Per agent (override CLI)

Imposta un override esplicito dell'ordine dei profili di autenticazione per un agent (archiviato nel suo `auth-profiles.json`):

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Usa `--agent <id>` per scegliere come target un agent specifico; omettilo per usare l'agent predefinito configurato.
Quando esegui il debug di problemi di ordine, `openclaw models status --probe` mostra i
profili archiviati omessi come `excluded_by_auth_order` invece di ignorarli silenziosamente.
Quando esegui il debug di problemi di cooldown, ricorda che i cooldown dei limiti di velocità possono essere associati
a un singolo ID modello anziché all'intero profilo provider.

## Risoluzione dei problemi

### "Nessuna credenziale trovata"

Se il profilo Anthropic manca, configura una chiave API Anthropic sull'**host gateway**
oppure imposta il percorso legacy setup-token di Anthropic, quindi ricontrolla:

```bash
openclaw models status
```

### Token in scadenza/scaduto

Esegui `openclaw models status` per confermare quale profilo è in scadenza. Se un profilo token
Anthropic legacy manca o è scaduto, aggiorna quella configurazione tramite
setup-token oppure migra a una chiave API Anthropic.

Se la macchina ha ancora uno stato Anthropic Claude CLI obsoleto e rimosso da build
precedenti, esegui:

```bash
openclaw doctor --yes
```

Doctor riconverte `anthropic:claude-cli` in token/OAuth Anthropic quando i
byte delle credenziali archiviate esistono ancora. In caso contrario rimuove i riferimenti a profilo/configurazione/modello Claude CLI obsoleti e lascia indicazioni per il passaggio successivo.
