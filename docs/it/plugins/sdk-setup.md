---
read_when:
    - Stai aggiungendo una procedura guidata di setup a un plugin
    - Devi capire setup-entry.ts rispetto a index.ts
    - Stai definendo schemi di configurazione del plugin o metadati openclaw in package.json
sidebarTitle: Setup and Config
summary: Procedure guidate di setup, setup-entry.ts, schemi di configurazione e metadati package.json
title: Setup e configurazione del plugin
x-i18n:
    generated_at: "2026-04-06T03:10:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: eac2586516d27bcd94cc4c259fe6274c792b3f9938c7ddd6dbf04a6dbb988dc9
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# Setup e configurazione del plugin

Riferimento per il packaging del plugin (metadati `package.json`), i manifest
(`openclaw.plugin.json`), le entry di setup e gli schemi di configurazione.

<Tip>
  **Cerchi una guida passo passo?** Le guide pratiche trattano il packaging nel contesto:
  [Plugin di canale](/it/plugins/sdk-channel-plugins#step-1-package-and-manifest) e
  [Plugin provider](/it/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Metadati del pacchetto

Il tuo `package.json` deve includere un campo `openclaw` che indichi al sistema plugin cosa
fornisce il tuo plugin:

**Plugin di canale:**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Breve descrizione del canale."
    }
  }
}
```

**Plugin provider / baseline di pubblicazione ClawHub:**

```json openclaw-clawhub-package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

Se pubblichi il plugin esternamente su ClawHub, i campi `compat` e `build`
sono obbligatori. Gli snippet canonici per la pubblicazione si trovano in
`docs/snippets/plugin-publish/`.

### Campi `openclaw`

| Campo        | Tipo       | Descrizione                                                                                           |
| ------------ | ---------- | ----------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | File entry point (relativi alla root del pacchetto)                                                   |
| `setupEntry` | `string`   | Entry solo setup leggera (opzionale)                                                                  |
| `channel`    | `object`   | Metadati del catalogo canali per setup, picker, quickstart e superfici di stato                      |
| `providers`  | `string[]` | ID provider registrati da questo plugin                                                               |
| `install`    | `object`   | Suggerimenti di installazione: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `allowInvalidConfigRecovery` |
| `startup`    | `object`   | Flag di comportamento all'avvio                                                                       |

### `openclaw.channel`

`openclaw.channel` è un metadato package leggero per la discovery del canale e le
superfici di setup prima del caricamento del runtime.

| Campo                                  | Tipo       | Significato                                                              |
| -------------------------------------- | ---------- | ------------------------------------------------------------------------ |
| `id`                                   | `string`   | ID canonico del canale.                                                  |
| `label`                                | `string`   | Etichetta principale del canale.                                         |
| `selectionLabel`                       | `string`   | Etichetta del picker/setup quando deve differire da `label`.             |
| `detailLabel`                          | `string`   | Etichetta secondaria di dettaglio per cataloghi canale più ricchi e superfici di stato. |
| `docsPath`                             | `string`   | Percorso della documentazione per i link di setup e selezione.           |
| `docsLabel`                            | `string`   | Etichetta di override usata per i link alla documentazione quando deve differire dall'ID canale. |
| `blurb`                                | `string`   | Breve descrizione di onboarding/catalogo.                                |
| `order`                                | `number`   | Ordine di ordinamento nei cataloghi canale.                              |
| `aliases`                              | `string[]` | Alias di ricerca aggiuntivi per la selezione del canale.                 |
| `preferOver`                           | `string[]` | ID plugin/canale a priorità inferiore che questo canale deve superare.   |
| `systemImage`                          | `string`   | Nome opzionale di icona/system-image per i cataloghi UI del canale.      |
| `selectionDocsPrefix`                  | `string`   | Testo prefisso prima dei link alla documentazione nelle superfici di selezione. |
| `selectionDocsOmitLabel`               | `boolean`  | Mostra direttamente il percorso della documentazione invece di un link documentazione etichettato nel testo di selezione. |
| `selectionExtras`                      | `string[]` | Stringhe brevi aggiuntive aggiunte nel testo di selezione.               |
| `markdownCapable`                      | `boolean`  | Contrassegna il canale come compatibile con Markdown per le decisioni di formattazione in uscita. |
| `exposure`                             | `object`   | Controlli di visibilità del canale per setup, elenchi configurati e superfici della documentazione. |
| `quickstartAllowFrom`                  | `boolean`  | Include questo canale nel flusso standard di setup quickstart `allowFrom`. |
| `forceAccountBinding`                  | `boolean`  | Richiede un binding esplicito dell'account anche quando esiste un solo account. |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Preferisce la ricerca della sessione quando risolve le destinazioni di annuncio per questo canale. |

Esempio:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Integrazione di chat self-hosted basata su webhook.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guida:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` supporta:

- `configured`: include il canale nelle superfici di elenco in stile configurato/stato
- `setup`: include il canale nei picker interattivi di setup/configurazione
- `docs`: contrassegna il canale come pubblico nelle superfici di documentazione/navigazione

`showConfigured` e `showInSetup` restano supportati come alias legacy. Preferisci
`exposure`.

### `openclaw.install`

`openclaw.install` è un metadato package, non un metadato manifest.

| Campo                        | Tipo                 | Significato                                                                 |
| ---------------------------- | -------------------- | --------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | Specifica npm canonica per i flussi di installazione/aggiornamento.         |
| `localPath`                  | `string`             | Percorso di installazione locale di sviluppo o bundled.                     |
| `defaultChoice`              | `"npm"` \| `"local"` | Origine di installazione preferita quando entrambe sono disponibili.        |
| `minHostVersion`             | `string`             | Versione minima supportata di OpenClaw nel formato `>=x.y.z`.               |
| `allowInvalidConfigRecovery` | `boolean`            | Consente ai flussi di reinstallazione del plugin bundled di recuperare da specifici errori di configurazione obsoleta. |

Se `minHostVersion` è impostato, sia l'installazione sia il caricamento del registro dei manifest lo applicano.
Gli host più vecchi saltano il plugin; le stringhe di versione non valide vengono rifiutate.

`allowInvalidConfigRecovery` non è un bypass generale per configurazioni rotte. È
solo per il recupero limitato dei plugin bundled, così reinstallazione/setup possono riparare residui noti di upgrade come un percorso mancante del plugin bundled o una voce `channels.<id>`
obsoleta per quel medesimo plugin. Se la configurazione è rotta per motivi non correlati, l'installazione
continua a fallire in modo chiuso e indica all'operatore di eseguire `openclaw doctor --fix`.

### Caricamento completo differito

I plugin di canale possono attivare il caricamento differito con:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Quando è abilitato, OpenClaw carica solo `setupEntry` durante la fase di avvio
pre-listen, anche per i canali già configurati. L'entry completa viene caricata dopo che il
gateway inizia ad ascoltare.

<Warning>
  Abilita il caricamento differito solo quando `setupEntry` registra tutto ciò di cui il
  gateway ha bisogno prima di iniziare ad ascoltare (registrazione del canale, route HTTP,
  metodi del gateway). Se l'entry completa possiede capability di avvio necessarie, mantieni
  il comportamento predefinito.
</Warning>

Se la tua entry setup/completa registra metodi RPC del gateway, mantienili su un
prefisso specifico del plugin. Gli spazi dei nomi admin core riservati (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restano di proprietà del core e vengono sempre risolti
su `operator.admin`.

## Manifest del plugin

Ogni plugin nativo deve includere un `openclaw.plugin.json` nella root del pacchetto.
OpenClaw lo usa per convalidare la configurazione senza eseguire il codice del plugin.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Aggiunge capability My Plugin a OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Secret di verifica del webhook"
      }
    }
  }
}
```

Per i plugin di canale, aggiungi `kind` e `channels`:

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Anche i plugin senza configurazione devono includere uno schema. Uno schema vuoto è valido:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Consulta [Manifest del plugin](/it/plugins/manifest) per il riferimento completo dello schema.

## Pubblicazione su ClawHub

Per i pacchetti plugin, usa il comando ClawHub specifico per il pacchetto:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

L'alias legacy di pubblicazione solo-Skills è per le Skills. I pacchetti plugin dovrebbero
sempre usare `clawhub package publish`.

## Entry di setup

Il file `setup-entry.ts` è un'alternativa leggera a `index.ts` che
OpenClaw carica quando ha bisogno solo delle superfici di setup (onboarding, riparazione della configurazione,
ispezione di canali disabilitati).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Questo evita di caricare codice di runtime pesante (librerie crittografiche, registrazioni CLI,
servizi in background) durante i flussi di setup.

**Quando OpenClaw usa `setupEntry` invece dell'entry completa:**

- Il canale è disabilitato ma ha bisogno delle superfici di setup/onboarding
- Il canale è abilitato ma non configurato
- Il caricamento differito è abilitato (`deferConfiguredChannelFullLoadUntilAfterListen`)

**Cosa deve registrare `setupEntry`:**

- L'oggetto plugin del canale (tramite `defineSetupPluginEntry`)
- Eventuali route HTTP richieste prima del listen del gateway
- Eventuali metodi gateway necessari durante l'avvio

Questi metodi gateway di avvio dovrebbero comunque evitare gli spazi dei nomi
admin core riservati come `config.*` o `update.*`.

**Cosa NON dovrebbe includere `setupEntry`:**

- Registrazioni CLI
- Servizi in background
- Import runtime pesanti (crypto, SDK)
- Metodi gateway necessari solo dopo l'avvio

### Import helper di setup limitati

Per i percorsi solo setup più caldi, preferisci i seam helper di setup limitati invece del più ampio
ombrello `plugin-sdk/setup` quando ti serve solo una parte della superficie di setup:

| Percorso di importazione            | Usalo per                                                                              | Export chiave                                                                                                                                                                                                                                                                                 |
| ----------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`         | helper runtime in fase di setup che restano disponibili in `setupEntry` / avvio differito del canale | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime` | adapter di setup account consapevoli dell'ambiente                                     | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                         |
| `plugin-sdk/setup-tools`           | helper CLI/archivio/documentazione di setup/installazione                              | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                               |

Usa il seam più ampio `plugin-sdk/setup` quando vuoi l'intera cassetta degli attrezzi condivisa per il setup,
inclusi gli helper di patch della configurazione come
`moveSingleAccountChannelSectionToDefaultAccount(...)`.

Gli adapter di patch setup restano sicuri da importare sui percorsi caldi. La loro
ricerca della superficie di contratto bundled per la promozione single-account è lazy, quindi importare
`plugin-sdk/setup-runtime` non carica eager la discovery della superficie di contratto bundled prima che l'adapter venga effettivamente usato.

### Promozione single-account posseduta dal canale

Quando un canale passa da una configurazione top-level single-account a
`channels.<id>.accounts.*`, il comportamento condiviso predefinito è spostare i valori promossi con ambito account in `accounts.default`.

I canali bundled possono restringere o sostituire quella promozione tramite la loro
superficie di contratto setup:

- `singleAccountKeysToMove`: chiavi top-level aggiuntive che devono essere spostate nell'account
  promosso
- `namedAccountPromotionKeys`: quando esistono già account con nome, solo queste
  chiavi vengono spostate nell'account promosso; le chiavi condivise di policy/delivery restano nella root del
  canale
- `resolveSingleAccountPromotionTarget(...)`: sceglie quale account esistente
  riceve i valori promossi

Matrix è l'esempio bundled attuale. Se esiste già esattamente un account Matrix con nome,
oppure se `defaultAccount` punta a una chiave non canonica esistente come
`Ops`, la promozione preserva quell'account invece di creare una nuova voce
`accounts.default`.

## Schema di configurazione

La configurazione del plugin viene convalidata rispetto al JSON Schema nel tuo manifest. Gli utenti
configurano i plugin tramite:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Il tuo plugin riceve questa configurazione come `api.pluginConfig` durante la registrazione.

Per la configurazione specifica del canale, usa invece la sezione di configurazione del canale:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Creazione di schemi di configurazione del canale

Usa `buildChannelConfigSchema` da `openclaw/plugin-sdk/core` per convertire uno
schema Zod nel wrapper `ChannelConfigSchema` che OpenClaw convalida:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## Procedure guidate di setup

I plugin di canale possono fornire procedure guidate di setup interattive per `openclaw onboard`.
La procedura guidata è un oggetto `ChannelSetupWizard` nel `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connesso",
    unconfiguredLabel: "Non configurato",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Token bot",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Usare MY_CHANNEL_BOT_TOKEN dall'ambiente?",
      keepPrompt: "Mantenere il token corrente?",
      inputPrompt: "Inserisci il token del tuo bot:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

Il tipo `ChannelSetupWizard` supporta `credentials`, `textInputs`,
`dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` e altro.
Consulta i pacchetti plugin bundled (ad esempio il plugin Discord `src/channel.setup.ts`) per
esempi completi.

Per i prompt della allowlist DM che richiedono solo il flusso standard
`note -> prompt -> parse -> merge -> patch`, preferisci gli helper di setup condivisi da `openclaw/plugin-sdk/setup`: `createPromptParsedAllowFromForAccount(...)`,
`createTopLevelChannelParsedAllowFromPrompt(...)` e
`createNestedChannelParsedAllowFromPrompt(...)`.

Per i blocchi di stato del setup del canale che variano solo per etichette, punteggi e righe aggiuntive opzionali, preferisci `createStandardChannelSetupStatus(...)` da
`openclaw/plugin-sdk/setup` invece di costruire manualmente lo stesso oggetto `status` in
ogni plugin.

Per le superfici di setup opzionali che dovrebbero comparire solo in certi contesti, usa
`createOptionalChannelSetupSurface` da `openclaw/plugin-sdk/channel-setup`:

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Restituisce { setupAdapter, setupWizard }
```

`plugin-sdk/channel-setup` espone anche i builder di livello inferiore
`createOptionalChannelSetupAdapter(...)` e
`createOptionalChannelSetupWizard(...)` quando ti serve solo una metà di
quella superficie di installazione opzionale.

L'adapter/procedura guidata opzionale generata fallisce in modo chiuso sulle vere scritture di configurazione. Riusa un unico messaggio che richiede installazione in `validateInput`,
`applyAccountConfig` e `finalize`, e aggiunge un link alla documentazione quando `docsPath` è
impostato.

Per UI di setup supportate da binari, preferisci gli helper delegati condivisi invece di
copiare la stessa colla binario/stato in ogni canale:

- `createDetectedBinaryStatus(...)` per blocchi di stato che variano solo per etichette,
  suggerimenti, punteggi e rilevamento binario
- `createCliPathTextInput(...)` per input di testo basati su percorso
- `createDelegatedSetupWizardStatusResolvers(...)`,
  `createDelegatedPrepare(...)`, `createDelegatedFinalize(...)` e
  `createDelegatedResolveConfigured(...)` quando `setupEntry` deve inoltrare lazy a una procedura guidata completa più pesante
- `createDelegatedTextInputShouldPrompt(...)` quando `setupEntry` deve solo
  delegare una decisione `textInputs[*].shouldPrompt`

## Pubblicazione e installazione

**Plugin esterni:** pubblica su [ClawHub](/it/tools/clawhub) o npm, poi installa:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw prova prima ClawHub e poi ripiega automaticamente su npm. Puoi anche
forzare esplicitamente ClawHub:

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # solo ClawHub
```

Non esiste un override `npm:` corrispondente. Usa la normale package spec npm quando
vuoi il percorso npm dopo il fallback ClawHub:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**Plugin nel repository:** posizionali sotto l'albero workspace dei plugin bundled e verranno automaticamente
rilevati durante la build.

**Gli utenti possono installare:**

```bash
openclaw plugins install <package-name>
```

<Info>
  Per le installazioni provenienti da npm, `openclaw plugins install` esegue
  `npm install --ignore-scripts` (nessuno script di lifecycle). Mantieni gli alberi di dipendenze del plugin in puro JS/TS ed evita pacchetti che richiedono build `postinstall`.
</Info>

## Correlati

- [SDK Entry Points](/it/plugins/sdk-entrypoints) -- `definePluginEntry` e `defineChannelPluginEntry`
- [Manifest del plugin](/it/plugins/manifest) -- riferimento completo dello schema del manifest
- [Creare plugin](/it/plugins/building-plugins) -- guida introduttiva passo passo
