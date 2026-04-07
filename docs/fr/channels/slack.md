---
read_when:
    - Configurer Slack ou déboguer le mode socket/HTTP de Slack
summary: Configuration de Slack et comportement d'exécution (Socket Mode + URL de requête HTTP)
title: Slack
x-i18n:
    generated_at: "2026-04-07T06:49:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2b8fd2cc6c638ee82069f0af2c2b6f6f49c87da709b941433a0343724a9907ea
    source_path: channels/slack.md
    workflow: 15
---

# Slack

Statut : prêt pour la production pour les DM et les canaux via les intégrations d'application Slack. Le mode par défaut est Socket Mode ; les URL de requête HTTP sont également prises en charge.

<CardGroup cols={3}>
  <Card title="Appairage" icon="link" href="/fr/channels/pairing">
    Les DM Slack utilisent le mode d'appairage par défaut.
  </Card>
  <Card title="Commandes slash" icon="terminal" href="/fr/tools/slash-commands">
    Comportement natif des commandes et catalogue des commandes.
  </Card>
  <Card title="Dépannage des canaux" icon="wrench" href="/fr/channels/troubleshooting">
    Diagnostics inter-canaux et guides de résolution.
  </Card>
</CardGroup>

## Configuration rapide

<Tabs>
  <Tab title="Socket Mode (par défaut)">
    <Steps>
      <Step title="Créer une nouvelle application Slack">
        Dans les paramètres de l'application Slack, appuyez sur le bouton **[Create New App](https://api.slack.com/apps/new)** :

        - choisissez **from a manifest** et sélectionnez un espace de travail pour votre application
        - collez le [manifeste d'exemple](#manifest-and-scope-checklist) ci-dessous et poursuivez la création
        - générez un **App-Level Token** (`xapp-...`) avec `connections:write`
        - installez l'application et copiez le **Bot Token** (`xoxb-...`) affiché
      </Step>

      <Step title="Configurer OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

        Variable d'environnement de secours (compte par défaut uniquement) :

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      </Step>

      <Step title="Démarrer la passerelle">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="URL de requête HTTP">
    <Steps>
      <Step title="Créer une nouvelle application Slack">
        Dans les paramètres de l'application Slack, appuyez sur le bouton **[Create New App](https://api.slack.com/apps/new)** :

        - choisissez **from a manifest** et sélectionnez un espace de travail pour votre application
        - collez le [manifeste d'exemple](#manifest-and-scope-checklist) et mettez à jour les URL avant de créer
        - enregistrez le **Signing Secret** pour la vérification des requêtes
        - installez l'application et copiez le **Bot Token** (`xoxb-...`) affiché

      </Step>

      <Step title="Configurer OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events",
    },
  },
}
```

        <Note>
        Utilisez des chemins webhook uniques pour le HTTP multi-compte

        Donnez à chaque compte un `webhookPath` distinct (par défaut `/slack/events`) afin que les enregistrements n'entrent pas en conflit.
        </Note>

      </Step>

      <Step title="Démarrer la passerelle">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Liste de vérification du manifeste et des scopes

<Tabs>
  <Tab title="Socket Mode (par défaut)">

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": true
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

  </Tab>

  <Tab title="URL de requête HTTP">

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": true
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Scopes d'autorisation facultatifs (opérations d'écriture)">
    Ajoutez le scope bot `chat:write.customize` si vous voulez que les messages sortants utilisent l'identité de l'agent actif (nom d'utilisateur et icône personnalisés) au lieu de l'identité par défaut de l'application Slack.

    Si vous utilisez une icône emoji, Slack attend une syntaxe `:nom_emoji:`.
  </Accordion>
  <Accordion title="Scopes facultatifs de jeton utilisateur (opérations de lecture)">
    Si vous configurez `channels.slack.userToken`, les scopes de lecture typiques sont :

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (si vous dépendez des lectures de recherche Slack)

  </Accordion>
</AccordionGroup>

## Modèle de jeton

- `botToken` + `appToken` sont requis pour Socket Mode.
- Le mode HTTP nécessite `botToken` + `signingSecret`.
- `botToken`, `appToken`, `signingSecret` et `userToken` acceptent des chaînes
  en clair ou des objets SecretRef.
- Les jetons de configuration remplacent la variable d'environnement de secours.
- La variable d'environnement de secours `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` s'applique uniquement au compte par défaut.
- `userToken` (`xoxp-...`) est uniquement dans la configuration (pas de variable d'environnement de secours) et utilise par défaut un comportement en lecture seule (`userTokenReadOnly: true`).

Comportement de l'instantané d'état :

- L'inspection des comptes Slack suit les champs `*Source` et `*Status`
  par identifiant (`botToken`, `appToken`, `signingSecret`, `userToken`).
- L'état est `available`, `configured_unavailable` ou `missing`.
- `configured_unavailable` signifie que le compte est configuré via SecretRef
  ou une autre source de secret non inline, mais que le chemin de commande/d'exécution actuel
  n'a pas pu résoudre la valeur réelle.
- En mode HTTP, `signingSecretStatus` est inclus ; en Socket Mode, la
  paire requise est `botTokenStatus` + `appTokenStatus`.

<Tip>
Pour les lectures d'actions/répertoire, le jeton utilisateur peut être privilégié lorsqu'il est configuré. Pour les écritures, le jeton bot reste prioritaire ; les écritures avec jeton utilisateur ne sont autorisées que lorsque `userTokenReadOnly: false` et que le jeton bot n'est pas disponible.
</Tip>

## Actions et contrôles

Les actions Slack sont contrôlées par `channels.slack.actions.*`.

Groupes d'actions disponibles dans l'outillage Slack actuel :

| Groupe     | Par défaut |
| ---------- | ---------- |
| messages   | activé     |
| reactions  | activé     |
| pins       | activé     |
| memberInfo | activé     |
| emojiList  | activé     |

Les actions actuelles des messages Slack incluent `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info` et `emoji-list`.

## Contrôle d'accès et routage

<Tabs>
  <Tab title="Politique des DM">
    `channels.slack.dmPolicy` contrôle l'accès DM (hérité : `channels.slack.dm.policy`) :

    - `pairing` (par défaut)
    - `allowlist`
    - `open` (nécessite que `channels.slack.allowFrom` inclue `"*"` ; hérité : `channels.slack.dm.allowFrom`)
    - `disabled`

    Indicateurs DM :

    - `dm.enabled` (true par défaut)
    - `channels.slack.allowFrom` (recommandé)
    - `dm.allowFrom` (hérité)
    - `dm.groupEnabled` (DM de groupe désactivés par défaut)
    - `dm.groupChannels` (liste d'autorisation MPIM facultative)

    Priorité multi-compte :

    - `channels.slack.accounts.default.allowFrom` s'applique uniquement au compte `default`.
    - Les comptes nommés héritent de `channels.slack.allowFrom` lorsque leur propre `allowFrom` n'est pas défini.
    - Les comptes nommés n'héritent pas de `channels.slack.accounts.default.allowFrom`.

    L'appairage dans les DM utilise `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="Politique des canaux">
    `channels.slack.groupPolicy` contrôle la gestion des canaux :

    - `open`
    - `allowlist`
    - `disabled`

    La liste d'autorisation des canaux se trouve sous `channels.slack.channels` et doit utiliser des ID de canal stables.

    Remarque d'exécution : si `channels.slack` est complètement absent (configuration par variable d'environnement uniquement), l'exécution revient à `groupPolicy="allowlist"` et enregistre un avertissement (même si `channels.defaults.groupPolicy` est défini).

    Résolution nom/ID :

    - les entrées de liste d'autorisation des canaux et les entrées de liste d'autorisation des DM sont résolues au démarrage lorsque l'accès au jeton le permet
    - les entrées de nom de canal non résolues sont conservées telles que configurées mais ignorées pour le routage par défaut
    - l'autorisation entrante et le routage des canaux utilisent les ID en priorité par défaut ; la correspondance directe de nom d'utilisateur/slug nécessite `channels.slack.dangerouslyAllowNameMatching: true`

  </Tab>

  <Tab title="Mentions et utilisateurs des canaux">
    Les messages de canal sont filtrés par mention par défaut.

    Sources de mention :

    - mention explicite de l'application (`<@botId>`)
    - modèles regex de mention (`agents.list[].groupChat.mentionPatterns`, secours `messages.groupChat.mentionPatterns`)
    - comportement implicite de réponse à un fil du bot (désactivé lorsque `thread.requireExplicitMention` vaut `true`)

    Contrôles par canal (`channels.slack.channels.<id>` ; noms uniquement via la résolution au démarrage ou `dangerouslyAllowNameMatching`) :

    - `requireMention`
    - `users` (liste d'autorisation)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - format de clé `toolsBySender` : `id:`, `e164:`, `username:`, `name:` ou joker `"*"`
      (les clés héritées sans préfixe sont toujours mappées vers `id:` uniquement)

  </Tab>
</Tabs>

## Fils, sessions et balises de réponse

- Les DM sont routés comme `direct` ; les canaux comme `channel` ; les MPIM comme `group`.
- Avec la valeur par défaut `session.dmScope=main`, les DM Slack sont regroupés dans la session principale de l'agent.
- Sessions de canal : `agent:<agentId>:slack:channel:<channelId>`.
- Les réponses dans un fil peuvent créer des suffixes de session de fil (`:thread:<threadTs>`) selon le cas.
- La valeur par défaut de `channels.slack.thread.historyScope` est `thread` ; celle de `thread.inheritParent` est `false`.
- `channels.slack.thread.initialHistoryLimit` contrôle combien de messages de fil existants sont récupérés lorsqu'une nouvelle session de fil démarre (par défaut `20` ; définissez `0` pour désactiver).
- `channels.slack.thread.requireExplicitMention` (par défaut `false`) : lorsque la valeur est `true`, supprime les mentions implicites dans les fils pour que le bot ne réponde qu'aux mentions explicites `@bot` à l'intérieur des fils, même lorsque le bot a déjà participé au fil. Sans cela, les réponses dans un fil où le bot a participé contournent le filtrage `requireMention`.

Contrôles de fil de réponse :

- `channels.slack.replyToMode`: `off|first|all|batched` (par défaut `off`)
- `channels.slack.replyToModeByChatType`: par `direct|group|channel`
- secours hérité pour les chats directs : `channels.slack.dm.replyToMode`

Les balises de réponse manuelles sont prises en charge :

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

Remarque : `replyToMode="off"` désactive **tout** le fil de réponse dans Slack, y compris les balises explicites `[[reply_to_*]]`. Cela diffère de Telegram, où les balises explicites sont toujours honorées en mode `"off"`. La différence reflète les modèles de fil des plateformes : les fils Slack masquent les messages du canal, tandis que les réponses Telegram restent visibles dans le flux principal du chat.

## Réactions d'accusé de réception

`ackReaction` envoie un emoji d'accusé de réception pendant qu'OpenClaw traite un message entrant.

Ordre de résolution :

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- secours emoji d'identité d'agent (`agents.list[].identity.emoji`, sinon "👀")

Remarques :

- Slack attend des shortcodes (par exemple `"eyes"`).
- Utilisez `""` pour désactiver la réaction pour le compte Slack ou globalement.

## Streaming de texte

`channels.slack.streaming` contrôle le comportement d'aperçu en direct :

- `off` : désactiver le streaming d'aperçu en direct.
- `partial` (par défaut) : remplace le texte d'aperçu par la dernière sortie partielle.
- `block` : ajoute des mises à jour d'aperçu par blocs.
- `progress` : affiche un texte d'état de progression pendant la génération, puis envoie le texte final.

`channels.slack.nativeStreaming` contrôle le streaming de texte natif Slack lorsque `streaming` vaut `partial` (par défaut : `true`).

- Un fil de réponse doit être disponible pour que le streaming de texte natif apparaisse. La sélection du fil suit toujours `replyToMode`. Sans cela, l'aperçu de brouillon normal est utilisé.
- Les médias et charges utiles non textuelles reviennent à la livraison normale.
- Si le streaming échoue au milieu de la réponse, OpenClaw revient à la livraison normale pour les charges utiles restantes.

Utiliser l'aperçu de brouillon au lieu du streaming de texte natif Slack :

```json5
{
  channels: {
    slack: {
      streaming: "partial",
      nativeStreaming: false,
    },
  },
}
```

Clés héritées :

- `channels.slack.streamMode` (`replace | status_final | append`) est migré automatiquement vers `channels.slack.streaming`.
- le booléen `channels.slack.streaming` est migré automatiquement vers `channels.slack.nativeStreaming`.

## Secours de réaction de saisie

`typingReaction` ajoute une réaction temporaire au message Slack entrant pendant qu'OpenClaw traite une réponse, puis la supprime lorsque l'exécution se termine. Cela est surtout utile en dehors des réponses dans les fils, qui utilisent un indicateur d'état par défaut "is typing...".

Ordre de résolution :

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

Remarques :

- Slack attend des shortcodes (par exemple `"hourglass_flowing_sand"`).
- La réaction est best-effort et le nettoyage est tenté automatiquement après la réponse ou après la fin du chemin d'échec.

## Médias, segmentation et livraison

<AccordionGroup>
  <Accordion title="Pièces jointes entrantes">
    Les pièces jointes de fichier Slack sont téléchargées depuis des URL privées hébergées par Slack (flux de requête authentifié par jeton) et écrites dans le stockage média lorsque la récupération réussit et que les limites de taille le permettent.

    Le plafond de taille entrant à l'exécution est de `20MB` par défaut, sauf remplacement par `channels.slack.mediaMaxMb`.

  </Accordion>

  <Accordion title="Texte et fichiers sortants">
    - les segments de texte utilisent `channels.slack.textChunkLimit` (par défaut 4000)
    - `channels.slack.chunkMode="newline"` active le découpage prioritaire par paragraphe
    - les envois de fichiers utilisent les API de téléchargement Slack et peuvent inclure des réponses de fil (`thread_ts`)
    - le plafond de média sortant suit `channels.slack.mediaMaxMb` lorsqu'il est configuré ; sinon, les envois de canal utilisent les valeurs par défaut de type MIME du pipeline média
  </Accordion>

  <Accordion title="Cibles de livraison">
    Cibles explicites privilégiées :

    - `user:<id>` pour les DM
    - `channel:<id>` pour les canaux

    Les DM Slack sont ouverts via les API de conversation Slack lors de l'envoi vers des cibles utilisateur.

  </Accordion>
</AccordionGroup>

## Commandes et comportement des slash commands

- Le mode automatique de commande native est **désactivé** pour Slack (`commands.native: "auto"` n'active pas les commandes natives Slack).
- Activez les gestionnaires de commande natifs Slack avec `channels.slack.commands.native: true` (ou globalement `commands.native: true`).
- Lorsque les commandes natives sont activées, enregistrez les slash commands correspondantes dans Slack (noms `/<command>`), avec une exception :
  - enregistrez `/agentstatus` pour la commande d'état (Slack réserve `/status`)
- Si les commandes natives ne sont pas activées, vous pouvez exécuter une seule slash command configurée via `channels.slack.slashCommand`.
- Les menus d'arguments natifs adaptent désormais leur stratégie de rendu :
  - jusqu'à 5 options : blocs de boutons
  - 6-100 options : menu de sélection statique
  - plus de 100 options : sélection externe avec filtrage asynchrone des options lorsque les gestionnaires d'options d'interactivité sont disponibles
  - si les valeurs d'option encodées dépassent les limites de Slack, le flux revient aux boutons
- Pour les longues charges utiles d'options, les menus d'arguments de slash command utilisent une boîte de dialogue de confirmation avant d'envoyer une valeur sélectionnée.

Paramètres par défaut des slash commands :

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

Les sessions de slash utilisent des clés isolées :

- `agent:<agentId>:slack:slash:<userId>`

et continuent de router l'exécution de la commande vers la session de conversation cible (`CommandTargetSessionKey`).

## Réponses interactives

Slack peut afficher des contrôles de réponse interactive rédigés par l'agent, mais cette fonctionnalité est désactivée par défaut.

Activez-la globalement :

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

Ou activez-la pour un seul compte Slack :

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

Lorsqu'elle est activée, les agents peuvent émettre des directives de réponse propres à Slack :

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

Ces directives sont compilées en Slack Block Kit et routent les clics ou sélections vers le chemin d'événement d'interaction Slack existant.

Remarques :

- Il s'agit d'une interface propre à Slack. Les autres canaux ne traduisent pas les directives Slack Block Kit vers leurs propres systèmes de boutons.
- Les valeurs de rappel interactives sont des jetons opaques générés par OpenClaw, pas des valeurs brutes rédigées par l'agent.
- Si les blocs interactifs générés dépassent les limites de Slack Block Kit, OpenClaw revient à la réponse textuelle d'origine au lieu d'envoyer une charge utile de blocs invalide.

## Approbations Exec dans Slack

Slack peut agir comme client d'approbation natif avec des boutons et interactions interactifs, au lieu de revenir à l'interface Web ou au terminal.

- Les approbations Exec utilisent `channels.slack.execApprovals.*` pour le routage natif DM/canal.
- Les approbations de plugin peuvent toujours se résoudre via la même surface de bouton native Slack lorsque la requête arrive déjà dans Slack et que le type d'identifiant d'approbation est `plugin:`.
- L'autorisation des approbateurs reste appliquée : seuls les utilisateurs identifiés comme approbateurs peuvent approuver ou refuser des demandes via Slack.

Cela utilise la même surface de bouton d'approbation partagée que les autres canaux. Lorsque `interactivity` est activé dans les paramètres de votre application Slack, les invites d'approbation s'affichent comme des boutons Block Kit directement dans la conversation.
Lorsque ces boutons sont présents, ils constituent l'expérience d'approbation principale ; OpenClaw
ne doit inclure une commande `/approve` manuelle que lorsque le résultat de l'outil indique que les
approbations par chat ne sont pas disponibles ou que l'approbation manuelle est le seul chemin.

Chemin de configuration :

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (facultatif ; revient à `commands.ownerAllowFrom` lorsque possible)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, par défaut : `dm`)
- `agentFilter`, `sessionFilter`

Slack active automatiquement les approbations Exec natives lorsque `enabled` n'est pas défini ou vaut `"auto"` et qu'au moins un
approbateur est résolu. Définissez `enabled: false` pour désactiver explicitement Slack comme client d'approbation natif.
Définissez `enabled: true` pour forcer les approbations natives lorsque des approbateurs sont résolus.

Comportement par défaut sans configuration explicite d'approbation Exec Slack :

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

Une configuration native Slack explicite n'est nécessaire que lorsque vous voulez remplacer les approbateurs, ajouter des filtres ou
opter pour une livraison au chat d'origine :

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

Le transfert partagé `approvals.exec` est distinct. Utilisez-le uniquement lorsque les invites d'approbation Exec doivent aussi
être routées vers d'autres chats ou des cibles explicites hors bande. Le transfert partagé `approvals.plugin` est également
distinct ; les boutons natifs Slack peuvent toujours résoudre les approbations de plugin lorsque ces requêtes arrivent déjà
dans Slack.

Le `/approve` dans le même chat fonctionne aussi dans les canaux et DM Slack qui prennent déjà en charge les commandes. Voir [Approbations Exec](/fr/tools/exec-approvals) pour le modèle complet de transfert des approbations.

## Événements et comportement opérationnel

- Les modifications/suppressions de message/diffusions de fil sont mappées vers des événements système.
- Les événements d'ajout/suppression de réaction sont mappés vers des événements système.
- Les événements d'entrée/sortie de membre, de création/renommage de canal et d'ajout/suppression d'épingle sont mappés vers des événements système.
- `channel_id_changed` peut migrer les clés de configuration de canal lorsque `configWrites` est activé.
- Les métadonnées de sujet/objectifs de canal sont traitées comme un contexte non fiable et peuvent être injectées dans le contexte de routage.
- Le lanceur de fil et l'initialisation du contexte d'historique initial de fil sont filtrés par les listes d'autorisation d'expéditeur configurées, selon le cas.
- Les actions de bloc et interactions de modal émettent des événements système structurés `Slack interaction: ...` avec des champs de charge utile riches :
  - actions de bloc : valeurs sélectionnées, libellés, valeurs de sélecteur et métadonnées `workflow_*`
  - événements de modal `view_submission` et `view_closed` avec métadonnées de canal routées et entrées de formulaire

## Pointeurs vers la référence de configuration

Référence principale :

- [Référence de configuration - Slack](/fr/gateway/configuration-reference#slack)

  Champs Slack à fort signal :
  - mode/auth : `mode`, `botToken`, `appToken`, `signingSecret`, `webhookPath`, `accounts.*`
  - accès DM : `dm.enabled`, `dmPolicy`, `allowFrom` (hérité : `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
  - bascule de compatibilité : `dangerouslyAllowNameMatching` (ultime recours ; laissez désactivé sauf besoin)
  - accès canal : `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`
  - fils/historique : `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
  - livraison : `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `streaming`, `nativeStreaming`
  - ops/fonctionnalités : `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

## Dépannage

<AccordionGroup>
  <Accordion title="Aucune réponse dans les canaux">
    Vérifiez, dans l'ordre :

    - `groupPolicy`
    - liste d'autorisation des canaux (`channels.slack.channels`)
    - `requireMention`
    - liste d'autorisation `users` par canal

    Commandes utiles :

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="Messages DM ignorés">
    Vérifiez :

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (ou hérité `channels.slack.dm.policy`)
    - approbations d'appairage / entrées de liste d'autorisation

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Le mode socket ne se connecte pas">
    Validez les jetons bot + app et l'activation de Socket Mode dans les paramètres de l'application Slack.

    Si `openclaw channels status --probe --json` affiche `botTokenStatus` ou
    `appTokenStatus: "configured_unavailable"`, le compte Slack est
    configuré mais l'exécution actuelle n'a pas pu résoudre la
    valeur adossée à SecretRef.

  </Accordion>

  <Accordion title="Le mode HTTP ne reçoit pas d'événements">
    Validez :

    - signing secret
    - chemin webhook
    - URL de requête Slack (Events + Interactivity + Slash Commands)
    - `webhookPath` unique par compte HTTP

    Si `signingSecretStatus: "configured_unavailable"` apparaît dans les
    instantanés de compte, le compte HTTP est configuré mais l'exécution actuelle n'a pas pu
    résoudre le signing secret adossé à SecretRef.

  </Accordion>

  <Accordion title="Les commandes natives/slash ne se déclenchent pas">
    Vérifiez si vous vouliez :

    - le mode de commande native (`channels.slack.commands.native: true`) avec les slash commands correspondantes enregistrées dans Slack
    - ou le mode de slash command unique (`channels.slack.slashCommand.enabled: true`)

    Vérifiez aussi `commands.useAccessGroups` et les listes d'autorisation de canal/utilisateur.

  </Accordion>
</AccordionGroup>

## Lié

- [Appairage](/fr/channels/pairing)
- [Groupes](/fr/channels/groups)
- [Sécurité](/fr/gateway/security)
- [Routage des canaux](/fr/channels/channel-routing)
- [Dépannage](/fr/channels/troubleshooting)
- [Configuration](/fr/gateway/configuration)
- [Commandes slash](/fr/tools/slash-commands)
