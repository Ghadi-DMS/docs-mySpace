---
read_when:
    - Travail sur le comportement du canal WhatsApp/web ou le routage de la boîte de réception
summary: Prise en charge du canal WhatsApp, contrôles d'accès, comportement de livraison et opérations
title: WhatsApp
x-i18n:
    generated_at: "2026-04-07T06:49:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e2ce84d869ace6c0bebd9ec17bdbbef997a5c31e5da410b02a19a0f103f7359
    source_path: channels/whatsapp.md
    workflow: 15
---

# WhatsApp (canal web)

Statut : prêt pour la production via WhatsApp Web (Baileys). La passerelle possède la ou les sessions liées.

## Installation (à la demande)

- L'onboarding (`openclaw onboard`) et `openclaw channels add --channel whatsapp`
  proposent d'installer le plugin WhatsApp la première fois que vous le sélectionnez.
- `openclaw channels login --channel whatsapp` propose également le flux d'installation lorsque
  le plugin n'est pas encore présent.
- Canal de développement + extraction git : utilise par défaut le chemin du plugin local.
- Stable/Beta : utilise par défaut le package npm `@openclaw/whatsapp`.

L'installation manuelle reste disponible :

```bash
openclaw plugins install @openclaw/whatsapp
```

<CardGroup cols={3}>
  <Card title="Appairage" icon="link" href="/fr/channels/pairing">
    La stratégie DM par défaut est l'appairage pour les expéditeurs inconnus.
  </Card>
  <Card title="Dépannage des canaux" icon="wrench" href="/fr/channels/troubleshooting">
    Diagnostics inter-canaux et procédures de réparation.
  </Card>
  <Card title="Configuration de la passerelle" icon="settings" href="/fr/gateway/configuration">
    Modèles et exemples complets de configuration de canal.
  </Card>
</CardGroup>

## Configuration rapide

<Steps>
  <Step title="Configurer la stratégie d'accès WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="Lier WhatsApp (QR)">

```bash
openclaw channels login --channel whatsapp
```

    Pour un compte spécifique :

```bash
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Démarrer la passerelle">

```bash
openclaw gateway
```

  </Step>

  <Step title="Approuver la première demande d'appairage (si vous utilisez le mode appairage)">

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    Les demandes d'appairage expirent après 1 heure. Les demandes en attente sont limitées à 3 par canal.

  </Step>
</Steps>

<Note>
OpenClaw recommande d'exécuter WhatsApp sur un numéro séparé lorsque c'est possible. (Les métadonnées du canal et le flux de configuration sont optimisés pour cette configuration, mais les configurations avec numéro personnel sont également prises en charge.)
</Note>

## Modèles de déploiement

<AccordionGroup>
  <Accordion title="Numéro dédié (recommandé)">
    Il s'agit du mode opérationnel le plus propre :

    - identité WhatsApp distincte pour OpenClaw
    - listes d'autorisation DM et limites de routage plus claires
    - risque réduit de confusion avec l'auto-conversation

    Modèle de stratégie minimal :

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Solution de repli avec numéro personnel">
    L'onboarding prend en charge le mode numéro personnel et écrit une base adaptée à l'auto-conversation :

    - `dmPolicy: "allowlist"`
    - `allowFrom` inclut votre numéro personnel
    - `selfChatMode: true`

    À l'exécution, les protections d'auto-conversation s'appuient sur le numéro personnel lié et `allowFrom`.

  </Accordion>

  <Accordion title="Périmètre du canal WhatsApp Web uniquement">
    Le canal de plateforme de messagerie est basé sur WhatsApp Web (`Baileys`) dans l'architecture actuelle des canaux OpenClaw.

    Il n'existe pas de canal de messagerie WhatsApp Twilio distinct dans le registre intégré des canaux de chat.

  </Accordion>
</AccordionGroup>

## Modèle d'exécution

- La passerelle possède le socket WhatsApp et la boucle de reconnexion.
- Les envois sortants nécessitent un écouteur WhatsApp actif pour le compte cible.
- Les discussions de statut et de diffusion sont ignorées (`@status`, `@broadcast`).
- Les discussions directes utilisent les règles de session DM (`session.dmScope` ; `main` par défaut regroupe les DM dans la session principale de l'agent).
- Les sessions de groupe sont isolées (`agent:<agentId>:whatsapp:group:<jid>`).
- Le transport WhatsApp Web respecte les variables d'environnement proxy standard sur l'hôte de la passerelle (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY` / variantes en minuscules). Préférez une configuration proxy au niveau de l'hôte plutôt que des paramètres proxy WhatsApp spécifiques au canal.

## Contrôle d'accès et activation

<Tabs>
  <Tab title="Stratégie DM">
    `channels.whatsapp.dmPolicy` contrôle l'accès aux discussions directes :

    - `pairing` (par défaut)
    - `allowlist`
    - `open` (nécessite que `allowFrom` inclue `"*"`)
    - `disabled`

    `allowFrom` accepte les numéros au format E.164 (normalisés en interne).

    Remplacement multi-comptes : `channels.whatsapp.accounts.<id>.dmPolicy` (et `allowFrom`) est prioritaire sur les valeurs par défaut au niveau du canal pour ce compte.

    Détails du comportement à l'exécution :

    - les appairages sont conservés dans le magasin d'autorisations du canal et fusionnés avec `allowFrom` configuré
    - si aucune liste d'autorisation n'est configurée, le numéro personnel lié est autorisé par défaut
    - les DM sortants `fromMe` ne sont jamais appairés automatiquement

  </Tab>

  <Tab title="Stratégie de groupe + listes d'autorisation">
    L'accès aux groupes comporte deux niveaux :

    1. **Liste d'autorisation des appartenances au groupe** (`channels.whatsapp.groups`)
       - si `groups` est omis, tous les groupes sont éligibles
       - si `groups` est présent, il agit comme une liste d'autorisation de groupes (`"*"` autorisé)

    2. **Stratégie des expéditeurs de groupe** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`)
       - `open` : la liste d'autorisation des expéditeurs est contournée
       - `allowlist` : l'expéditeur doit correspondre à `groupAllowFrom` (ou `*`)
       - `disabled` : bloque toutes les entrées de groupe

    Repli de la liste d'autorisation des expéditeurs :

    - si `groupAllowFrom` n'est pas défini, l'exécution se replie sur `allowFrom` quand il est disponible
    - les listes d'autorisation des expéditeurs sont évaluées avant l'activation par mention/réponse

    Remarque : si aucun bloc `channels.whatsapp` n'existe du tout, le repli de stratégie de groupe à l'exécution est `allowlist` (avec un avertissement dans les journaux), même si `channels.defaults.groupPolicy` est défini.

  </Tab>

  <Tab title="Mentions + /activation">
    Les réponses dans les groupes nécessitent une mention par défaut.

    La détection de mention comprend :

    - des mentions WhatsApp explicites de l'identité du bot
    - des motifs regex de mention configurés (`agents.list[].groupChat.mentionPatterns`, repli `messages.groupChat.mentionPatterns`)
    - la détection implicite de réponse au bot (l'expéditeur de la réponse correspond à l'identité du bot)

    Note de sécurité :

    - la citation/réponse satisfait uniquement le filtrage par mention ; elle **n'accorde pas** d'autorisation à l'expéditeur
    - avec `groupPolicy: "allowlist"`, les expéditeurs qui ne sont pas dans la liste d'autorisation restent bloqués même s'ils répondent au message d'un utilisateur autorisé

    Commande d'activation au niveau de la session :

    - `/activation mention`
    - `/activation always`

    `activation` met à jour l'état de la session (pas la configuration globale). Elle est limitée au propriétaire.

  </Tab>
</Tabs>

## Comportement avec numéro personnel et auto-conversation

Lorsque le numéro personnel lié est également présent dans `allowFrom`, les protections d'auto-conversation WhatsApp s'activent :

- ignorer les accusés de lecture pour les tours d'auto-conversation
- ignorer le comportement de déclenchement automatique par mention-JID qui vous interpellerait autrement vous-même
- si `messages.responsePrefix` n'est pas défini, les réponses d'auto-conversation utilisent par défaut `[{identity.name}]` ou `[openclaw]`

## Normalisation des messages et contexte

<AccordionGroup>
  <Accordion title="Enveloppe entrante + contexte de réponse">
    Les messages WhatsApp entrants sont enveloppés dans l'enveloppe entrante partagée.

    Si une réponse citée existe, le contexte est ajouté sous cette forme :

    ```text
    [En réponse à <sender> id:<stanzaId>]
    <corps cité ou espace réservé de média>
    [/En réponse]
    ```

    Les champs de métadonnées de réponse sont également renseignés quand ils sont disponibles (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, JID/E.164 de l'expéditeur).

  </Accordion>

  <Accordion title="Espaces réservés de médias et extraction de position/contact">
    Les messages entrants contenant uniquement des médias sont normalisés avec des espaces réservés tels que :

    - `<media:image>`
    - `<media:video>`
    - `<media:audio>`
    - `<media:document>`
    - `<media:sticker>`

    Les charges utiles de position et de contact sont normalisées en contexte textuel avant le routage.

  </Accordion>

  <Accordion title="Injection de l'historique de groupe en attente">
    Pour les groupes, les messages non traités peuvent être mis en mémoire tampon et injectés comme contexte lorsque le bot est finalement déclenché.

    - limite par défaut : `50`
    - configuration : `channels.whatsapp.historyLimit`
    - repli : `messages.groupChat.historyLimit`
    - `0` désactive

    Marqueurs d'injection :

    - `[Messages du chat depuis votre dernière réponse - pour le contexte]`
    - `[Message actuel - répondez à celui-ci]`

  </Accordion>

  <Accordion title="Accusés de lecture">
    Les accusés de lecture sont activés par défaut pour les messages WhatsApp entrants acceptés.

    Désactiver globalement :

    ```json5
    {
      channels: {
        whatsapp: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Remplacement par compte :

    ```json5
    {
      channels: {
        whatsapp: {
          accounts: {
            work: {
              sendReadReceipts: false,
            },
          },
        },
      },
    }
    ```

    Les tours d'auto-conversation ignorent les accusés de lecture même lorsqu'ils sont activés globalement.

  </Accordion>
</AccordionGroup>

## Livraison, segmentation et médias

<AccordionGroup>
  <Accordion title="Segmentation du texte">
    - limite de segment par défaut : `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.chunkMode = "length" | "newline"`
    - le mode `newline` privilégie les limites de paragraphe (lignes vides), puis se replie sur une segmentation sûre par longueur
  </Accordion>

  <Accordion title="Comportement des médias sortants">
    - prend en charge les charges utiles image, vidéo, audio (note vocale PTT) et document
    - `audio/ogg` est réécrit en `audio/ogg; codecs=opus` pour la compatibilité avec les notes vocales
    - la lecture des GIF animés est prise en charge via `gifPlayback: true` sur les envois vidéo
    - les légendes sont appliquées au premier élément média lors de l'envoi de charges utiles de réponse multi-médias
    - la source du média peut être HTTP(S), `file://` ou des chemins locaux
  </Accordion>

  <Accordion title="Limites de taille des médias et comportement de repli">
    - plafond d'enregistrement des médias entrants : `channels.whatsapp.mediaMaxMb` (par défaut `50`)
    - plafond d'envoi des médias sortants : `channels.whatsapp.mediaMaxMb` (par défaut `50`)
    - les remplacements par compte utilisent `channels.whatsapp.accounts.<accountId>.mediaMaxMb`
    - les images sont automatiquement optimisées (redimensionnement/balayage de qualité) pour respecter les limites
    - en cas d'échec d'envoi de média, le repli du premier élément envoie un avertissement textuel au lieu d'abandonner silencieusement la réponse
  </Accordion>
</AccordionGroup>

## Niveau de réaction

`channels.whatsapp.reactionLevel` contrôle dans quelle mesure l'agent utilise les réactions emoji sur WhatsApp :

| Niveau        | Réactions d'accusé de réception | Réactions initiées par l'agent | Description                                      |
| ------------- | -------------------------------- | ------------------------------ | ------------------------------------------------ |
| `"off"`       | Non                              | Non                            | Aucune réaction                                  |
| `"ack"`       | Oui                              | Non                            | Réactions d'accusé de réception uniquement (pré-réponse) |
| `"minimal"`   | Oui                              | Oui (conservatrices)           | Accusé de réception + réactions agent avec directives conservatrices |
| `"extensive"` | Oui                              | Oui (encouragées)              | Accusé de réception + réactions agent avec directives encourageantes |

Par défaut : `"minimal"`.

Les remplacements par compte utilisent `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{
  channels: {
    whatsapp: {
      reactionLevel: "ack",
    },
  },
}
```

## Réactions d'accusé de réception

WhatsApp prend en charge les réactions d'accusé de réception immédiates à la réception d'un message entrant via `channels.whatsapp.ackReaction`.
Les réactions d'accusé de réception sont conditionnées par `reactionLevel` — elles sont supprimées lorsque `reactionLevel` vaut `"off"`.

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

Notes sur le comportement :

- envoyées immédiatement après l'acceptation d'un message entrant (pré-réponse)
- les échecs sont consignés dans les journaux mais ne bloquent pas la livraison normale des réponses
- le mode groupe `mentions` réagit sur les tours déclenchés par mention ; l'activation de groupe `always` contourne cette vérification
- WhatsApp utilise `channels.whatsapp.ackReaction` (l'ancien `messages.ackReaction` n'est pas utilisé ici)

## Multi-comptes et identifiants

<AccordionGroup>
  <Accordion title="Sélection du compte et valeurs par défaut">
    - les identifiants de compte proviennent de `channels.whatsapp.accounts`
    - sélection du compte par défaut : `default` s'il est présent, sinon le premier identifiant de compte configuré (trié)
    - les identifiants de compte sont normalisés en interne pour la recherche
  </Accordion>

  <Accordion title="Chemins des identifiants et compatibilité héritée">
    - chemin d'authentification actuel : `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
    - fichier de sauvegarde : `creds.json.bak`
    - l'ancienne authentification par défaut dans `~/.openclaw/credentials/` est toujours reconnue/migrée pour les flux de compte par défaut
  </Accordion>

  <Accordion title="Comportement de déconnexion">
    `openclaw channels logout --channel whatsapp [--account <id>]` efface l'état d'authentification WhatsApp pour ce compte.

    Dans les anciens répertoires d'authentification, `oauth.json` est conservé tandis que les fichiers d'authentification Baileys sont supprimés.

  </Accordion>
</AccordionGroup>

## Outils, actions et écritures de configuration

- La prise en charge des outils de l'agent inclut l'action de réaction WhatsApp (`react`).
- Contrôles des actions :
  - `channels.whatsapp.actions.reactions`
  - `channels.whatsapp.actions.polls`
- Les écritures de configuration initiées par le canal sont activées par défaut (désactivez via `channels.whatsapp.configWrites=false`).

## Dépannage

<AccordionGroup>
  <Accordion title="Non lié (QR requis)">
    Symptôme : l'état du canal indique qu'il n'est pas lié.

    Correctif :

    ```bash
    openclaw channels login --channel whatsapp
    openclaw channels status
    ```

  </Accordion>

  <Accordion title="Lié mais déconnecté / boucle de reconnexion">
    Symptôme : compte lié avec déconnexions répétées ou tentatives de reconnexion.

    Correctif :

    ```bash
    openclaw doctor
    openclaw logs --follow
    ```

    Si nécessaire, reliez avec `channels login`.

  </Accordion>

  <Accordion title="Aucun écouteur actif lors de l'envoi">
    Les envois sortants échouent immédiatement lorsqu'aucun écouteur de passerelle actif n'existe pour le compte cible.

    Assurez-vous que la passerelle est en cours d'exécution et que le compte est lié.

  </Accordion>

  <Accordion title="Messages de groupe ignorés de manière inattendue">
    Vérifiez dans cet ordre :

    - `groupPolicy`
    - `groupAllowFrom` / `allowFrom`
    - les entrées de liste d'autorisation `groups`
    - le filtrage par mention (`requireMention` + motifs de mention)
    - les clés dupliquées dans `openclaw.json` (JSON5) : les entrées ultérieures remplacent les précédentes, donc conservez un seul `groupPolicy` par portée

  </Accordion>

  <Accordion title="Avertissement d'exécution Bun">
    L'exécution de la passerelle WhatsApp doit utiliser Node. Bun est signalé comme incompatible pour un fonctionnement stable de la passerelle WhatsApp/Telegram.
  </Accordion>
</AccordionGroup>

## Pointeurs de référence de configuration

Référence principale :

- [Référence de configuration - WhatsApp](/fr/gateway/configuration-reference#whatsapp)

Champs WhatsApp à fort impact :

- accès : `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`
- livraison : `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`
- multi-comptes : `accounts.<id>.enabled`, `accounts.<id>.authDir`, remplacements au niveau du compte
- opérations : `configWrites`, `debounceMs`, `web.enabled`, `web.heartbeatSeconds`, `web.reconnect.*`
- comportement de session : `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`

## Associé

- [Appairage](/fr/channels/pairing)
- [Groupes](/fr/channels/groups)
- [Sécurité](/fr/gateway/security)
- [Routage des canaux](/fr/channels/channel-routing)
- [Routage multi-agent](/fr/concepts/multi-agent)
- [Dépannage](/fr/channels/troubleshooting)
