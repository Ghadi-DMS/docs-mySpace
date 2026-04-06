---
read_when:
    - Stai collegando il trasporto QA sintetico a un'esecuzione di test locale o CI
    - Hai bisogno della superficie di configurazione del `qa-channel` incluso
    - Stai iterando sull'automazione QA end-to-end
summary: Plugin di canale sintetico di classe Slack per scenari QA deterministici di OpenClaw
title: Canale QA
x-i18n:
    generated_at: "2026-04-06T03:05:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b88cd73df2f61b34ad1eb83c3450f8fe15a51ac69fbb5a9eca0097564d67a06
    source_path: channels/qa-channel.md
    workflow: 15
---

# Canale QA

`qa-channel` è un trasporto di messaggi sintetico incluso per la QA automatizzata di OpenClaw.

Non è un canale di produzione. Esiste per esercitare lo stesso confine del plugin di canale
usato dai trasporti reali, mantenendo allo stesso tempo lo stato deterministico e completamente
ispezionabile.

## Cosa fa oggi

- Grammatica di destinazione di classe Slack:
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- Bus sintetico basato su HTTP per:
  - iniezione di messaggi in ingresso
  - acquisizione della trascrizione in uscita
  - creazione di thread
  - reazioni
  - modifiche
  - eliminazioni
  - azioni di ricerca e lettura
- Esecutore di autocontrollo lato host incluso che scrive un report Markdown

## Configurazione

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

Chiavi account supportate:

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## Esecutore

Sezione verticale attuale:

```bash
pnpm qa:e2e
```

Ora questo passa attraverso l'estensione `qa-lab` inclusa. Avvia il
bus QA nel repository, avvia la sezione di runtime `qa-channel` inclusa, esegue un
autocontrollo deterministico e scrive un report Markdown in `.artifacts/qa-e2e/`.

Interfaccia utente debugger privata:

```bash
pnpm qa:lab:build
pnpm openclaw qa ui
```

Suite QA completa supportata dal repository:

```bash
pnpm openclaw qa suite
```

Questo avvia il debugger QA privato a un URL locale, separato dal
bundle della Control UI distribuito.

## Ambito

L'ambito attuale è intenzionalmente limitato:

- bus + trasporto plugin
- grammatica di instradamento con thread
- azioni sui messaggi gestite dal canale
- reportistica Markdown

Il lavoro successivo aggiungerà:

- orchestrazione OpenClaw con Docker
- esecuzione della matrice provider/modello
- individuazione degli scenari più ricca
- orchestrazione nativa OpenClaw in seguito
