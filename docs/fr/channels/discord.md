---
read_when:
    - Travail sur les fonctionnalités du canal Discord
summary: Statut de prise en charge, capacités et configuration du bot Discord
title: Discord
x-i18n:
    generated_at: "2026-04-08T06:52:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3cd2886fad941ae2129e681911309539e9a65a2352b777b538d7f4686a68f73f
    source_path: channels/discord.md
    workflow: 15
---

# Discord (API Bot)

Statut : prêt pour les MP et les canaux de serveur via la passerelle Discord officielle.

<CardGroup cols={3}>
  <Card title="Appairage" icon="link" href="/fr/channels/pairing">
    Les MP Discord sont en mode d'appairage par défaut.
  </Card>
  <Card title="Commandes slash" icon="terminal" href="/fr/tools/slash-commands">
    Comportement natif des commandes et catalogue de commandes.
  </Card>
  <Card title="Dépannage des canaux" icon="wrench" href="/fr/channels/troubleshooting">
    Diagnostics inter-canaux et flux de réparation.
  </Card>
</CardGroup>

## Configuration rapide

Vous devrez créer une nouvelle application avec un bot, ajouter le bot à votre serveur et l'appairer à OpenClaw. Nous vous recommandons d'ajouter votre bot à votre propre serveur privé. Si vous n'en avez pas encore, [créez-en un d'abord](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (choisissez **Create My Own > For me and my friends**).

<Steps>
  <Step title="Créer une application Discord et un bot">
    Accédez au [Portail développeur Discord](https://discord.com/developers/applications) et cliquez sur **New Application**. Donnez-lui un nom comme « OpenClaw ».

    Cliquez sur **Bot** dans la barre latérale. Définissez le **Username** sur le nom que vous donnez à votre agent OpenClaw.

  </Step>

  <Step title="Activer les intents privilégiés">
    Toujours sur la page **Bot**, faites défiler jusqu'à **Privileged Gateway Intents** et activez :

    - **Message Content Intent** (requis)
    - **Server Members Intent** (recommandé ; requis pour les listes d'autorisation de rôles et la correspondance nom-vers-ID)
    - **Presence Intent** (facultatif ; nécessaire uniquement pour les mises à jour de présence)

  </Step>

  <Step title="Copier le jeton de votre bot">
    Revenez en haut de la page **Bot** et cliquez sur **Reset Token**.

    <Note>
    Malgré son nom, cela génère votre premier jeton — rien n'est réellement « réinitialisé ».
    </Note>

    Copiez le jeton et enregistrez-le quelque part. C'est votre **Bot Token** et vous en aurez besoin dans un instant.

  </Step>

  <Step title="Générer une URL d'invitation et ajouter le bot à votre serveur">
    Cliquez sur **OAuth2** dans la barre latérale. Vous allez générer une URL d'invitation avec les bonnes autorisations pour ajouter le bot à votre serveur.

    Faites défiler jusqu'à **OAuth2 URL Generator** et activez :

    - `bot`
    - `applications.commands`

    Une section **Bot Permissions** apparaîtra en dessous. Activez :

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (facultatif)

    Copiez l'URL générée en bas, collez-la dans votre navigateur, sélectionnez votre serveur, puis cliquez sur **Continue** pour vous connecter. Vous devriez maintenant voir votre bot dans le serveur Discord.

  </Step>

  <Step title="Activer le mode développeur et récupérer vos ID">
    De retour dans l'application Discord, vous devez activer le mode développeur pour pouvoir copier les ID internes.

    1. Cliquez sur **User Settings** (icône d'engrenage à côté de votre avatar) → **Advanced** → activez **Developer Mode**
    2. Faites un clic droit sur l'**icône de votre serveur** dans la barre latérale → **Copy Server ID**
    3. Faites un clic droit sur **votre propre avatar** → **Copy User ID**

    Enregistrez votre **Server ID** et votre **User ID** avec votre Bot Token — vous enverrez les trois à OpenClaw à l'étape suivante.

  </Step>

  <Step title="Autoriser les MP des membres du serveur">
    Pour que l'appairage fonctionne, Discord doit autoriser votre bot à vous envoyer des MP. Faites un clic droit sur l'**icône de votre serveur** → **Privacy Settings** → activez **Direct Messages**.

    Cela permet aux membres du serveur (y compris les bots) de vous envoyer des MP. Gardez cette option activée si vous voulez utiliser les MP Discord avec OpenClaw. Si vous prévoyez uniquement d'utiliser des canaux de serveur, vous pouvez désactiver les MP après l'appairage.

  </Step>

  <Step title="Définir le jeton de votre bot de manière sécurisée (ne l'envoyez pas dans le chat)">
    Le jeton de votre bot Discord est un secret (comme un mot de passe). Définissez-le sur la machine qui exécute OpenClaw avant d'envoyer un message à votre agent.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    Si OpenClaw s'exécute déjà comme service en arrière-plan, redémarrez-le via l'app Mac OpenClaw ou en arrêtant puis redémarrant le processus `openclaw gateway run`.

  </Step>

  <Step title="Configurer OpenClaw et appairer">

    <Tabs>
      <Tab title="Demander à votre agent">
        Discutez avec votre agent OpenClaw sur n'importe quel canal existant (par ex. Telegram) et dites-lui. Si Discord est votre premier canal, utilisez plutôt l'onglet CLI / config.

        > « J'ai déjà défini le jeton de mon bot Discord dans la configuration. Merci de terminer la configuration de Discord avec l'ID utilisateur `<user_id>` et l'ID serveur `<server_id>`. »
      </Tab>
      <Tab title="CLI / config">
        Si vous préférez une configuration basée sur un fichier, définissez :

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Repli env pour le compte par défaut :

```bash
DISCORD_BOT_TOKEN=...
```

        Les valeurs `token` en texte brut sont prises en charge. Les valeurs SecretRef sont également prises en charge pour `channels.discord.token` avec les fournisseurs env/file/exec. Voir [Gestion des secrets](/fr/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="Approuver le premier appairage par MP">
    Attendez que la passerelle soit en cours d'exécution, puis envoyez un MP à votre bot dans Discord. Il répondra avec un code d'appairage.

    <Tabs>
      <Tab title="Demander à votre agent">
        Envoyez le code d'appairage à votre agent sur votre canal existant :

        > « Approuve ce code d'appairage Discord : `<CODE>` »
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Les codes d'appairage expirent après 1 heure.

    Vous devriez maintenant pouvoir discuter avec votre agent sur Discord via MP.

  </Step>
</Steps>

<Note>
La résolution des jetons tient compte du compte. Les valeurs de jeton de configuration priment sur le repli env. `DISCORD_BOT_TOKEN` n'est utilisé que pour le compte par défaut.
Pour les appels sortants avancés (outil de message/actions de canal), un `token` explicite par appel est utilisé pour cet appel. Cela s'applique aux actions de type envoi et lecture/sondage (par exemple read/search/fetch/thread/pins/permissions). Les paramètres de politique de compte et de nouvelle tentative proviennent toujours du compte sélectionné dans l'instantané d'exécution actif.
</Note>

## Recommandé : configurer un espace de travail de serveur

Une fois que les MP fonctionnent, vous pouvez configurer votre serveur Discord comme un espace de travail complet où chaque canal obtient sa propre session d'agent avec son propre contexte. C'est recommandé pour les serveurs privés où il n'y a que vous et votre bot.

<Steps>
  <Step title="Ajouter votre serveur à la liste d'autorisation des serveurs">
    Cela permet à votre agent de répondre dans n'importe quel canal de votre serveur, pas seulement dans les MP.

    <Tabs>
      <Tab title="Demander à votre agent">
        > « Ajoute mon ID de serveur Discord `<server_id>` à la liste d'autorisation des serveurs »
      </Tab>
      <Tab title="Config">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Autoriser les réponses sans @mention">
    Par défaut, votre agent ne répond dans les canaux de serveur que lorsqu'il est @mentionné. Pour un serveur privé, vous voudrez probablement qu'il réponde à chaque message.

    <Tabs>
      <Tab title="Demander à votre agent">
        > « Autorise mon agent à répondre sur ce serveur sans avoir besoin d'être @mentionné »
      </Tab>
      <Tab title="Config">
        Définissez `requireMention: false` dans votre configuration de serveur :

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Prévoir la mémoire dans les canaux de serveur">
    Par défaut, la mémoire à long terme (MEMORY.md) n'est chargée que dans les sessions MP. Les canaux de serveur ne chargent pas automatiquement MEMORY.md.

    <Tabs>
      <Tab title="Demander à votre agent">
        > « Quand je pose des questions dans des canaux Discord, utilise memory_search ou memory_get si tu as besoin de contexte à long terme depuis MEMORY.md. »
      </Tab>
      <Tab title="Manuel">
        Si vous avez besoin d'un contexte partagé dans chaque canal, placez les instructions stables dans `AGENTS.md` ou `USER.md` (elles sont injectées pour chaque session). Gardez les notes à long terme dans `MEMORY.md` et accédez-y à la demande avec les outils de mémoire.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Créez maintenant quelques canaux sur votre serveur Discord et commencez à discuter. Votre agent peut voir le nom du canal, et chaque canal obtient sa propre session isolée — vous pouvez donc configurer `#coding`, `#home`, `#research`, ou ce qui convient à votre flux de travail.

## Modèle d'exécution

- La passerelle gère la connexion Discord.
- Le routage des réponses est déterministe : les réponses entrantes Discord repartent vers Discord.
- Par défaut (`session.dmScope=main`), les discussions directes partagent la session principale de l'agent (`agent:main:main`).
- Les canaux de serveur utilisent des clés de session isolées (`agent:<agentId>:discord:channel:<channelId>`).
- Les MP de groupe sont ignorés par défaut (`channels.discord.dm.groupEnabled=false`).
- Les commandes slash natives s'exécutent dans des sessions de commande isolées (`agent:<agentId>:discord:slash:<userId>`), tout en transportant `CommandTargetSessionKey` vers la session de conversation routée.

## Canaux forum

Les canaux forum et média Discord n'acceptent que les publications dans des fils. OpenClaw prend en charge deux façons de les créer :

- Envoyer un message au parent du forum (`channel:<forumId>`) pour créer automatiquement un fil. Le titre du fil utilise la première ligne non vide de votre message.
- Utiliser `openclaw message thread create` pour créer un fil directement. Ne passez pas `--message-id` pour les canaux forum.

Exemple : envoyer au parent du forum pour créer un fil

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

Exemple : créer explicitement un fil de forum

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

Les parents de forum n'acceptent pas les composants Discord. Si vous avez besoin de composants, envoyez au fil lui-même (`channel:<threadId>`).

## Composants interactifs

OpenClaw prend en charge les conteneurs Discord components v2 pour les messages d'agent. Utilisez l'outil de message avec une charge utile `components`. Les résultats d'interaction sont renvoyés à l'agent comme des messages entrants normaux et suivent les paramètres Discord `replyToMode` existants.

Blocs pris en charge :

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Les rangées d'actions autorisent jusqu'à 5 boutons ou un seul menu de sélection
- Types de sélection : `string`, `user`, `role`, `mentionable`, `channel`

Par défaut, les composants sont à usage unique. Définissez `components.reusable=true` pour autoriser l'utilisation multiple des boutons, sélections et formulaires jusqu'à leur expiration.

Pour restreindre qui peut cliquer sur un bouton, définissez `allowedUsers` sur ce bouton (ID utilisateur Discord, tags, ou `*`). Lorsque cette option est configurée, les utilisateurs sans correspondance reçoivent un refus éphémère.

Les commandes slash `/model` et `/models` ouvrent un sélecteur de modèle interactif avec des listes déroulantes de fournisseur et de modèle ainsi qu'une étape Submit. La réponse du sélecteur est éphémère et seul l'utilisateur qui l'a invoqué peut l'utiliser.

Pièces jointes de fichier :

- les blocs `file` doivent pointer vers une référence de pièce jointe (`attachment://<filename>`)
- fournissez la pièce jointe via `media`/`path`/`filePath` (fichier unique) ; utilisez `media-gallery` pour plusieurs fichiers
- utilisez `filename` pour remplacer le nom de téléversement lorsqu'il doit correspondre à la référence de pièce jointe

Formulaires modaux :

- ajoutez `components.modal` avec jusqu'à 5 champs
- types de champ : `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw ajoute automatiquement un bouton de déclenchement

Exemple :

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Contrôle d'accès et routage

<Tabs>
  <Tab title="Politique de MP">
    `channels.discord.dmPolicy` contrôle l'accès aux MP (hérité : `channels.discord.dm.policy`) :

    - `pairing` (par défaut)
    - `allowlist`
    - `open` (nécessite que `channels.discord.allowFrom` inclue `"*"` ; hérité : `channels.discord.dm.allowFrom`)
    - `disabled`

    Si la politique de MP n'est pas ouverte, les utilisateurs inconnus sont bloqués (ou invités à s'appairer en mode `pairing`).

    Priorité multi-comptes :

    - `channels.discord.accounts.default.allowFrom` s'applique uniquement au compte `default`.
    - Les comptes nommés héritent de `channels.discord.allowFrom` lorsque leur propre `allowFrom` n'est pas défini.
    - Les comptes nommés n'héritent pas de `channels.discord.accounts.default.allowFrom`.

    Format de cible MP pour la remise :

    - `user:<id>`
    - mention `<@id>`

    Les ID numériques bruts sont ambigus et rejetés sauf si un type de cible utilisateur/canal explicite est fourni.

  </Tab>

  <Tab title="Politique de serveur">
    La gestion des serveurs est contrôlée par `channels.discord.groupPolicy` :

    - `open`
    - `allowlist`
    - `disabled`

    La base sécurisée quand `channels.discord` existe est `allowlist`.

    Comportement de `allowlist` :

    - le serveur doit correspondre à `channels.discord.guilds` (`id` préféré, slug accepté)
    - listes d'autorisation facultatives d'expéditeur : `users` (ID stables recommandés) et `roles` (ID de rôle uniquement) ; si l'une ou l'autre est configurée, les expéditeurs sont autorisés lorsqu'ils correspondent à `users` OU `roles`
    - la correspondance directe par nom/tag est désactivée par défaut ; activez `channels.discord.dangerouslyAllowNameMatching: true` uniquement comme mode de compatibilité de secours
    - les noms/tags sont pris en charge pour `users`, mais les ID sont plus sûrs ; `openclaw security audit` avertit lorsque des entrées de nom/tag sont utilisées
    - si un serveur a `channels` configuré, les canaux non listés sont refusés
    - si un serveur n'a pas de bloc `channels`, tous les canaux de ce serveur autorisé sont autorisés

    Exemple :

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    Si vous définissez uniquement `DISCORD_BOT_TOKEN` et ne créez pas de bloc `channels.discord`, le comportement de repli à l'exécution est `groupPolicy="allowlist"` (avec un avertissement dans les journaux), même si `channels.defaults.groupPolicy` est `open`.

  </Tab>

  <Tab title="Mentions et MP de groupe">
    Les messages de serveur sont soumis aux mentions par défaut.

    La détection de mention inclut :

    - mention explicite du bot
    - motifs de mention configurés (`agents.list[].groupChat.mentionPatterns`, repli `messages.groupChat.mentionPatterns`)
    - comportement implicite de réponse-au-bot dans les cas pris en charge

    `requireMention` est configuré par serveur/canal (`channels.discord.guilds...`).
    `ignoreOtherMentions` ignore éventuellement les messages qui mentionnent un autre utilisateur/rôle mais pas le bot (hors @everyone/@here).

    MP de groupe :

    - par défaut : ignorés (`dm.groupEnabled=false`)
    - liste d'autorisation facultative via `dm.groupChannels` (ID de canal ou slugs)

  </Tab>
</Tabs>

### Routage d'agent basé sur les rôles

Utilisez `bindings[].match.roles` pour router les membres d'un serveur Discord vers différents agents par ID de rôle. Les liaisons basées sur les rôles acceptent uniquement les ID de rôle et sont évaluées après les liaisons pair ou pair parent, et avant les liaisons serveur seules. Si une liaison définit aussi d'autres champs de correspondance (par exemple `peer` + `guildId` + `roles`), tous les champs configurés doivent correspondre.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Configuration du Portail développeur

<AccordionGroup>
  <Accordion title="Créer une application et un bot">

    1. Portail développeur Discord -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. Copiez le jeton du bot

  </Accordion>

  <Accordion title="Intents privilégiés">
    Dans **Bot -> Privileged Gateway Intents**, activez :

    - Message Content Intent
    - Server Members Intent (recommandé)

    L'intent de présence est facultatif et uniquement requis si vous voulez recevoir des mises à jour de présence. Définir la présence du bot (`setPresence`) ne nécessite pas d'activer les mises à jour de présence pour les membres.

  </Accordion>

  <Accordion title="Scopes OAuth et autorisations de base">
    Générateur d'URL OAuth :

    - scopes : `bot`, `applications.commands`

    Autorisations de base typiques :

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (facultatif)

    Évitez `Administrator` sauf en cas de besoin explicite.

  </Accordion>

  <Accordion title="Copier les ID">
    Activez le mode développeur Discord, puis copiez :

    - ID du serveur
    - ID du canal
    - ID de l'utilisateur

    Préférez les ID numériques dans la configuration OpenClaw pour des audits et sondages fiables.

  </Accordion>
</AccordionGroup>

## Commandes natives et authentification des commandes

- `commands.native` vaut par défaut `"auto"` et est activé pour Discord.
- Remplacement par canal : `channels.discord.commands.native`.
- `commands.native=false` efface explicitement les commandes natives Discord précédemment enregistrées.
- L'authentification des commandes natives utilise les mêmes listes d'autorisation/politiques Discord que la gestion normale des messages.
- Les commandes peuvent quand même être visibles dans l'interface Discord pour les utilisateurs non autorisés ; l'exécution applique toujours l'authentification OpenClaw et renvoie « non autorisé ».

Voir [Commandes slash](/fr/tools/slash-commands) pour le catalogue et le comportement des commandes.

Paramètres par défaut des commandes slash :

- `ephemeral: true`

## Détails des fonctionnalités

<AccordionGroup>
  <Accordion title="Balises de réponse et réponses natives">
    Discord prend en charge les balises de réponse dans la sortie de l'agent :

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Contrôlé par `channels.discord.replyToMode` :

    - `off` (par défaut)
    - `first`
    - `all`
    - `batched`

    Remarque : `off` désactive le threading implicite des réponses. Les balises explicites `[[reply_to_*]]` sont toujours respectées.
    `first` attache toujours la référence de réponse native implicite au premier message Discord sortant du tour.
    `batched` attache uniquement la référence de réponse native implicite de Discord lorsque le
    tour entrant était un lot temporisé de plusieurs messages. Cela est utile
    si vous voulez des réponses natives surtout pour les discussions ambiguës et chargées, pas pour chaque
    tour à message unique.

    Les ID de message sont exposés dans le contexte/l'historique afin que les agents puissent cibler des messages spécifiques.

  </Accordion>

  <Accordion title="Aperçu de flux en direct">
    OpenClaw peut diffuser des brouillons de réponse en envoyant un message temporaire puis en le modifiant à mesure que le texte arrive.

    - `channels.discord.streaming` contrôle la diffusion d'aperçu (`off` | `partial` | `block` | `progress`, par défaut : `off`).
    - La valeur par défaut reste `off` car les modifications d'aperçu Discord peuvent rapidement atteindre les limites de débit, surtout lorsque plusieurs bots ou passerelles partagent le même compte ou trafic de serveur.
    - `progress` est accepté pour la cohérence inter-canaux et est mappé vers `partial` sur Discord.
    - `channels.discord.streamMode` est un alias hérité et est migré automatiquement.
    - `partial` modifie un seul message d'aperçu à mesure que les jetons arrivent.
    - `block` émet des morceaux de taille brouillon (utilisez `draftChunk` pour ajuster la taille et les points de coupure).

    Exemple :

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    Valeurs par défaut de découpage en mode `block` (bornées par `channels.discord.textChunkLimit`) :

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    La diffusion d'aperçu est uniquement textuelle ; les réponses média reviennent à la remise normale.

    Remarque : la diffusion d'aperçu est distincte de la diffusion par blocs. Lorsque la diffusion par blocs est explicitement
    activée pour Discord, OpenClaw ignore le flux d'aperçu pour éviter une double diffusion.

  </Accordion>

  <Accordion title="Historique, contexte et comportement des fils">
    Contexte d'historique de serveur :

    - `channels.discord.historyLimit` par défaut `20`
    - repli : `messages.groupChat.historyLimit`
    - `0` désactive

    Contrôles d'historique des MP :

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Comportement des fils :

    - les fils Discord sont routés comme des sessions de canal
    - les métadonnées du fil parent peuvent être utilisées pour le lien vers la session parente
    - la configuration du fil hérite de la configuration du canal parent sauf si une entrée spécifique au fil existe

    Les sujets de canal sont injectés comme contexte **non fiable** (pas comme prompt système).
    Le contexte des réponses et messages cités reste actuellement tel que reçu.
    Les listes d'autorisation Discord servent principalement à contrôler qui peut déclencher l'agent, et non comme frontière complète de masquage de contexte supplémentaire.

  </Accordion>

  <Accordion title="Sessions liées aux fils pour les sous-agents">
    Discord peut lier un fil à une cible de session afin que les messages de suivi dans ce fil continuent d'être routés vers la même session (y compris les sessions de sous-agent).

    Commandes :

    - `/focus <target>` lie le fil actuel/nouveau à une cible de sous-agent/session
    - `/unfocus` supprime la liaison du fil actuel
    - `/agents` affiche les exécutions actives et l'état des liaisons
    - `/session idle <duration|off>` inspecte/met à jour la suppression automatique de liaison par inactivité pour les liaisons focalisées
    - `/session max-age <duration|off>` inspecte/met à jour l'âge maximal strict pour les liaisons focalisées

    Config :

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    Remarques :

    - `session.threadBindings.*` définit les valeurs par défaut globales.
    - `channels.discord.threadBindings.*` remplace le comportement de Discord.
    - `spawnSubagentSessions` doit être true pour créer/lier automatiquement des fils pour `sessions_spawn({ thread: true })`.
    - `spawnAcpSessions` doit être true pour créer/lier automatiquement des fils pour ACP (`/acp spawn ... --thread ...` ou `sessions_spawn({ runtime: "acp", thread: true })`).
    - Si les liaisons de fils sont désactivées pour un compte, `/focus` et les opérations liées aux liaisons de fils ne sont pas disponibles.

    Voir [Sous-agents](/fr/tools/subagents), [Agents ACP](/fr/tools/acp-agents) et [Référence de configuration](/fr/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Liaisons persistantes de canal ACP">
    Pour des espaces de travail ACP stables « toujours actifs », configurez des liaisons ACP typées de niveau supérieur ciblant des conversations Discord.

    Chemin de configuration :

    - `bindings[]` avec `type: "acp"` et `match.channel: "discord"`

    Exemple :

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Remarques :

    - `/acp spawn codex --bind here` lie le canal ou fil Discord actuel sur place et garde les messages futurs routés vers la même session ACP.
    - Cela peut quand même signifier « démarrer une nouvelle session ACP Codex », mais cela ne crée pas de nouveau fil Discord à lui seul. Le canal existant reste la surface de discussion.
    - Codex peut toujours s'exécuter dans son propre `cwd` ou espace de travail backend sur disque. Cet espace de travail est un état d'exécution, pas un fil Discord.
    - Les messages de fil peuvent hériter de la liaison ACP du canal parent.
    - Dans un canal ou fil lié, `/new` et `/reset` réinitialisent la même session ACP sur place.
    - Les liaisons temporaires de fils fonctionnent toujours et peuvent remplacer la résolution de la cible tant qu'elles sont actives.
    - `spawnAcpSessions` n'est requis que lorsque OpenClaw doit créer/lier un fil enfant via `--thread auto|here`. Ce n'est pas requis pour `/acp spawn ... --bind here` dans le canal actuel.

    Voir [Agents ACP](/fr/tools/acp-agents) pour les détails du comportement des liaisons.

  </Accordion>

  <Accordion title="Notifications de réaction">
    Mode de notification des réactions par serveur :

    - `off`
    - `own` (par défaut)
    - `all`
    - `allowlist` (utilise `guilds.<id>.users`)

    Les événements de réaction sont transformés en événements système et joints à la session Discord routée.

  </Accordion>

  <Accordion title="Réactions d'accusé de réception">
    `ackReaction` envoie un emoji d'accusé de réception pendant que OpenClaw traite un message entrant.

    Ordre de résolution :

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - repli sur l'emoji d'identité de l'agent (`agents.list[].identity.emoji`, sinon "👀")

    Remarques :

    - Discord accepte les emoji unicode ou les noms d'emoji personnalisés.
    - Utilisez `""` pour désactiver la réaction pour un canal ou un compte.

  </Accordion>

  <Accordion title="Écritures de configuration">
    Les écritures de configuration initiées par le canal sont activées par défaut.

    Cela affecte les flux `/config set|unset` (lorsque les fonctionnalités de commande sont activées).

    Désactiver :

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Proxy de passerelle">
    Routez le trafic WebSocket de la passerelle Discord et les recherches REST de démarrage (ID d'application + résolution de liste d'autorisation) via un proxy HTTP(S) avec `channels.discord.proxy`.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Remplacement par compte :

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Prise en charge de PluralKit">
    Activez la résolution PluralKit pour mapper les messages relayés à l'identité du membre du système :

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // optional; needed for private systems
      },
    },
  },
}
```

    Remarques :

    - les listes d'autorisation peuvent utiliser `pk:<memberId>`
    - les noms d'affichage des membres sont mis en correspondance par nom/slug uniquement lorsque `channels.discord.dangerouslyAllowNameMatching: true`
    - les recherches utilisent l'ID du message d'origine et sont limitées par fenêtre temporelle
    - si la recherche échoue, les messages relayés sont traités comme des messages de bot et ignorés sauf si `allowBots=true`

  </Accordion>

  <Accordion title="Configuration de présence">
    Les mises à jour de présence sont appliquées lorsque vous définissez un champ d'état ou d'activité, ou lorsque vous activez la présence automatique.

    Exemple état uniquement :

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Exemple d'activité (le statut personnalisé est le type d'activité par défaut) :

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    Exemple de streaming :

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Correspondance des types d'activité :

    - 0 : Playing
    - 1 : Streaming (nécessite `activityUrl`)
    - 2 : Listening
    - 3 : Watching
    - 4 : Custom (utilise le texte d'activité comme état du statut ; emoji facultatif)
    - 5 : Competing

    Exemple de présence automatique (signal d'état d'exécution) :

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    La présence automatique mappe la disponibilité d'exécution vers l'état Discord : sain => online, dégradé ou inconnu => idle, épuisé ou indisponible => dnd. Remplacements de texte facultatifs :

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (prend en charge le placeholder `{reason}`)

  </Accordion>

  <Accordion title="Approbations dans Discord">
    Discord prend en charge le traitement des approbations via boutons en MP et peut éventuellement publier les invites d'approbation dans le canal d'origine.

    Chemin de configuration :

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (facultatif ; utilise `commands.ownerAllowFrom` comme repli lorsque possible)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, par défaut : `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord active automatiquement les approbations d'exécution natives lorsque `enabled` n'est pas défini ou vaut `"auto"` et qu'au moins un approbateur peut être résolu, soit depuis `execApprovals.approvers`, soit depuis `commands.ownerAllowFrom`. Discord ne déduit pas les approbateurs d'exécution à partir de `allowFrom` du canal, de `dm.allowFrom` hérité, ni de `defaultTo` des messages directs. Définissez `enabled: false` pour désactiver explicitement Discord comme client d'approbation natif.

    Lorsque `target` vaut `channel` ou `both`, l'invite d'approbation est visible dans le canal. Seuls les approbateurs résolus peuvent utiliser les boutons ; les autres utilisateurs reçoivent un refus éphémère. Les invites d'approbation incluent le texte de la commande, donc n'activez la remise dans le canal que dans des canaux de confiance. Si l'ID du canal ne peut pas être dérivé de la clé de session, OpenClaw revient à la remise par MP.

    Discord affiche également les boutons d'approbation partagés utilisés par d'autres canaux de chat. L'adaptateur Discord natif ajoute principalement le routage des MP d'approbateur et la diffusion en éventail vers les canaux.
    Lorsque ces boutons sont présents, ils constituent l'expérience d'approbation principale ; OpenClaw
    ne doit inclure une commande manuelle `/approve` que lorsque le résultat de l'outil indique
    que les approbations par chat sont indisponibles ou que l'approbation manuelle est la seule voie possible.

    L'authentification de la passerelle pour ce gestionnaire utilise le même contrat partagé de résolution des identifiants que les autres clients de la passerelle :

    - authentification locale env-first (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` puis `gateway.auth.*`)
    - en mode local, `gateway.remote.*` peut être utilisé comme repli uniquement lorsque `gateway.auth.*` n'est pas défini ; les SecretRef locaux configurés mais non résolus échouent en mode fermé
    - prise en charge du mode distant via `gateway.remote.*` le cas échéant
    - les remplacements d'URL sont sûrs vis-à-vis des remplacements : les remplacements CLI ne réutilisent pas d'identifiants implicites, et les remplacements env utilisent uniquement des identifiants env

    Comportement de résolution des approbations :

    - Les ID préfixés par `plugin:` sont résolus via `plugin.approval.resolve`.
    - Les autres ID sont résolus via `exec.approval.resolve`.
    - Discord n'effectue pas ici de saut de repli supplémentaire exec-vers-plugin ; le préfixe
      de l'ID décide de la méthode de passerelle qu'il appelle.

    Les approbations d'exécution expirent après 30 minutes par défaut. Si les approbations échouent avec
    des ID d'approbation inconnus, vérifiez la résolution des approbateurs, l'activation de la fonctionnalité, et
    que le type d'ID d'approbation remis correspond bien à la demande en attente.

    Documentation associée : [Approbations d'exécution](/fr/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## Outils et contrôles d'actions

Les actions de message Discord incluent la messagerie, l'administration de canal, la modération, la présence et les actions de métadonnées.

Exemples principaux :

- messagerie : `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- réactions : `react`, `reactions`, `emojiList`
- modération : `timeout`, `kick`, `ban`
- présence : `setPresence`

L'action `event-create` accepte un paramètre `image` facultatif (URL ou chemin de fichier local) pour définir l'image de couverture de l'événement planifié.

Les contrôles d'actions se trouvent sous `channels.discord.actions.*`.

Comportement par défaut des contrôles :

| Groupe d'actions                                                                                                                                                         | Par défaut |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | activé     |
| roles                                                                                                                                                                    | désactivé  |
| moderation                                                                                                                                                               | désactivé  |
| presence                                                                                                                                                                 | désactivé  |

## UI Components v2

OpenClaw utilise Discord components v2 pour les approbations d'exécution et les marqueurs inter-contexte. Les actions de message Discord peuvent également accepter `components` pour une UI personnalisée (avancé ; nécessite la construction d'une charge utile de composant via l'outil discord), tandis que les `embeds` hérités restent disponibles mais ne sont pas recommandés.

- `channels.discord.ui.components.accentColor` définit la couleur d'accent utilisée par les conteneurs de composants Discord (hex).
- Définissez-la par compte avec `channels.discord.accounts.<id>.ui.components.accentColor`.
- `embeds` sont ignorés lorsque des components v2 sont présents.

Exemple :

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Canaux vocaux

OpenClaw peut rejoindre des canaux vocaux Discord pour des conversations continues en temps réel. Cela est distinct des pièces jointes de message vocal.

Exigences :

- Activez les commandes natives (`commands.native` ou `channels.discord.commands.native`).
- Configurez `channels.discord.voice`.
- Le bot a besoin des autorisations Connect + Speak dans le canal vocal cible.

Utilisez la commande native Discord uniquement `/vc join|leave|status` pour contrôler les sessions. La commande utilise l'agent par défaut du compte et suit les mêmes règles de liste d'autorisation et de politique de groupe que les autres commandes Discord.

Exemple de jonction automatique :

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
    },
  },
}
```

Remarques :

- `voice.tts` remplace `messages.tts` uniquement pour la lecture vocale.
- Les tours de transcription vocale dérivent le statut de propriétaire à partir de Discord `allowFrom` (ou `dm.allowFrom`) ; les intervenants non propriétaires ne peuvent pas accéder aux outils réservés au propriétaire (par exemple `gateway` et `cron`).
- La voix est activée par défaut ; définissez `channels.discord.voice.enabled=false` pour la désactiver.
- `voice.daveEncryption` et `voice.decryptionFailureTolerance` sont transmis aux options de jonction `@discordjs/voice`.
- Les valeurs par défaut de `@discordjs/voice` sont `daveEncryption=true` et `decryptionFailureTolerance=24` si elles ne sont pas définies.
- OpenClaw surveille également les échecs de déchiffrement à la réception et récupère automatiquement en quittant puis en rejoignant à nouveau le canal vocal après des échecs répétés dans une courte fenêtre.
- Si les journaux de réception affichent de façon répétée `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`, il peut s'agir du bug de réception amont `@discordjs/voice` suivi dans [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419).

## Messages vocaux

Les messages vocaux Discord affichent un aperçu de forme d'onde et nécessitent de l'audio OGG/Opus ainsi que des métadonnées. OpenClaw génère automatiquement la forme d'onde, mais il a besoin de `ffmpeg` et `ffprobe` disponibles sur l'hôte de la passerelle pour inspecter et convertir les fichiers audio.

Exigences et contraintes :

- Fournissez un **chemin de fichier local** (les URL sont refusées).
- Omettez le contenu texte (Discord n'autorise pas le texte + message vocal dans la même charge utile).
- Tout format audio est accepté ; OpenClaw convertit en OGG/Opus si nécessaire.

Exemple :

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Dépannage

<AccordionGroup>
  <Accordion title="Intents interdits utilisés ou le bot ne voit aucun message de serveur">

    - activez Message Content Intent
    - activez Server Members Intent lorsque vous dépendez de la résolution utilisateur/membre
    - redémarrez la passerelle après avoir modifié les intents

  </Accordion>

  <Accordion title="Messages de serveur bloqués de manière inattendue">

    - vérifiez `groupPolicy`
    - vérifiez la liste d'autorisation du serveur sous `channels.discord.guilds`
    - si la map `channels` du serveur existe, seuls les canaux listés sont autorisés
    - vérifiez le comportement `requireMention` et les motifs de mention

    Vérifications utiles :

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention false mais toujours bloqué">
    Causes fréquentes :

    - `groupPolicy="allowlist"` sans liste d'autorisation correspondante de serveur/canal
    - `requireMention` configuré au mauvais endroit (doit se trouver sous `channels.discord.guilds` ou dans l'entrée du canal)
    - expéditeur bloqué par la liste d'autorisation `users` du serveur/canal

  </Accordion>

  <Accordion title="Les gestionnaires longue durée expirent ou les réponses sont dupliquées">

    Journaux typiques :

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    Paramètre de budget du listener :

    - compte unique : `channels.discord.eventQueue.listenerTimeout`
    - multi-comptes : `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    Paramètre de délai d'expiration d'exécution du worker :

    - compte unique : `channels.discord.inboundWorker.runTimeoutMs`
    - multi-comptes : `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - par défaut : `1800000` (30 minutes) ; définissez `0` pour désactiver

    Base recommandée :

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    Utilisez `eventQueue.listenerTimeout` pour les initialisations de listener lentes et `inboundWorker.runTimeoutMs`
    uniquement si vous voulez une soupape de sécurité distincte pour les tours d'agent en file d'attente.

  </Accordion>

  <Accordion title="Incohérences d'audit des autorisations">
    Les vérifications d'autorisations `channels status --probe` ne fonctionnent que pour les ID de canal numériques.

    Si vous utilisez des clés slug, la correspondance à l'exécution peut toujours fonctionner, mais le sondage ne peut pas vérifier complètement les autorisations.

  </Accordion>

  <Accordion title="Problèmes de MP et d'appairage">

    - MP désactivés : `channels.discord.dm.enabled=false`
    - politique MP désactivée : `channels.discord.dmPolicy="disabled"` (hérité : `channels.discord.dm.policy`)
    - en attente d'approbation d'appairage en mode `pairing`

  </Accordion>

  <Accordion title="Boucles bot vers bot">
    Par défaut, les messages rédigés par des bots sont ignorés.

    Si vous définissez `channels.discord.allowBots=true`, utilisez des règles strictes de mention et de liste d'autorisation pour éviter les comportements de boucle.
    Préférez `channels.discord.allowBots="mentions"` pour n'accepter que les messages de bots qui mentionnent le bot.

  </Accordion>

  <Accordion title="La STT vocale abandonne avec DecryptionFailed(...)">

    - gardez OpenClaw à jour (`openclaw update`) afin que la logique de récupération de réception vocale Discord soit présente
    - confirmez `channels.discord.voice.daveEncryption=true` (par défaut)
    - partez de `channels.discord.voice.decryptionFailureTolerance=24` (valeur par défaut amont) et ajustez uniquement si nécessaire
    - surveillez les journaux pour :
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - si les échecs continuent après la reconnexion automatique, collectez les journaux et comparez-les à [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## Indications de référence de configuration

Référence principale :

- [Référence de configuration - Discord](/fr/gateway/configuration-reference#discord)

Champs Discord à fort signal :

- démarrage/auth : `enabled`, `token`, `accounts.*`, `allowBots`
- politique : `groupPolicy`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- commande : `commands.native`, `commands.useAccessGroups`, `configWrites`, `slashCommand.*`
- file d'événements : `eventQueue.listenerTimeout` (budget du listener), `eventQueue.maxQueueSize`, `eventQueue.maxConcurrency`
- worker entrant : `inboundWorker.runTimeoutMs`
- réponse/historique : `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- remise : `textChunkLimit`, `chunkMode`, `maxLinesPerMessage`
- streaming : `streaming` (alias hérité : `streamMode`), `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`
- média/nouvelle tentative : `mediaMaxMb`, `retry`
  - `mediaMaxMb` plafonne les téléversements Discord sortants (par défaut : `100MB`)
- actions : `actions.*`
- présence : `activity`, `status`, `activityType`, `activityUrl`
- UI : `ui.components.accentColor`
- fonctionnalités : `threadBindings`, `bindings[]` de niveau supérieur (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents`, `heartbeat`, `responsePrefix`

## Sécurité et opérations

- Traitez les jetons de bot comme des secrets (`DISCORD_BOT_TOKEN` de préférence dans les environnements supervisés).
- Accordez le minimum d'autorisations Discord nécessaire.
- Si le déploiement/l'état des commandes est obsolète, redémarrez la passerelle et revérifiez avec `openclaw channels status --probe`.

## Liens associés

- [Appairage](/fr/channels/pairing)
- [Groupes](/fr/channels/groups)
- [Routage des canaux](/fr/channels/channel-routing)
- [Sécurité](/fr/gateway/security)
- [Routage multi-agent](/fr/concepts/multi-agent)
- [Dépannage](/fr/channels/troubleshooting)
- [Commandes slash](/fr/tools/slash-commands)
