---
read_when:
    - Vous avez besoin de la sémantique exacte ou des valeurs par défaut au niveau des champs de configuration
    - Vous validez des blocs de configuration de canal, de modèle, de passerelle ou d'outil
summary: Référence complète pour chaque clé de configuration OpenClaw, les valeurs par défaut et les paramètres des canaux
title: Référence de configuration
x-i18n:
    generated_at: "2026-04-07T06:56:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7768cb77e1d3fc483c66f655ea891d2c32f21b247e3c1a56a919b28a37f9b128
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Référence de configuration

Chaque champ disponible dans `~/.openclaw/openclaw.json`. Pour une vue d'ensemble orientée tâches, consultez [Configuration](/fr/gateway/configuration).

Le format de configuration est **JSON5** (commentaires + virgules finales autorisés). Tous les champs sont facultatifs — OpenClaw utilise des valeurs par défaut sûres lorsqu'ils sont omis.

---

## Canaux

Chaque canal démarre automatiquement lorsque sa section de configuration existe (sauf si `enabled: false`).

### Accès DM et groupe

Tous les canaux prennent en charge des stratégies DM et des stratégies de groupe :

| Stratégie DM        | Comportement                                                   |
| ------------------- | -------------------------------------------------------------- |
| `pairing` (par défaut) | Les expéditeurs inconnus reçoivent un code d'appairage à usage unique ; le propriétaire doit l'approuver |
| `allowlist`         | Seuls les expéditeurs dans `allowFrom` (ou le magasin d'autorisations appairé) |
| `open`              | Autoriser tous les DM entrants (nécessite `allowFrom: ["*"]`)  |
| `disabled`          | Ignorer tous les DM entrants                                   |

| Stratégie de groupe   | Comportement                                          |
| --------------------- | ----------------------------------------------------- |
| `allowlist` (par défaut) | Seuls les groupes correspondant à la liste d'autorisation configurée |
| `open`                | Contourne les listes d'autorisation de groupe (le filtrage par mention s'applique toujours) |
| `disabled`            | Bloque tous les messages de groupe/salon              |

<Note>
`channels.defaults.groupPolicy` définit la valeur par défaut lorsqu'un `groupPolicy` de fournisseur n'est pas défini.
Les codes d'appairage expirent après 1 heure. Les demandes d'appairage DM en attente sont limitées à **3 par canal**.
Si un bloc de fournisseur est totalement absent (`channels.<provider>` absent), la stratégie de groupe à l'exécution revient à `allowlist` (échec fermé) avec un avertissement au démarrage.
</Note>

### Remplacements de modèle par canal

Utilisez `channels.modelByChannel` pour épingler des identifiants de canal spécifiques à un modèle. Les valeurs acceptent `provider/model` ou des alias de modèle configurés. Le mappage de canal s'applique lorsqu'une session n'a pas déjà de remplacement de modèle (par exemple défini via `/model`).

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-4.1",
      },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### Valeurs par défaut des canaux et heartbeat

Utilisez `channels.defaults` pour partager le comportement de stratégie de groupe et de heartbeat entre les fournisseurs :

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy` : stratégie de groupe de repli lorsqu'un `groupPolicy` au niveau du fournisseur n'est pas défini.
- `channels.defaults.contextVisibility` : mode de visibilité du contexte supplémentaire par défaut pour tous les canaux. Valeurs : `all` (par défaut, inclut tout le contexte cité/thread/historique), `allowlist` (inclut uniquement le contexte des expéditeurs dans la liste d'autorisation), `allowlist_quote` (identique à allowlist mais conserve le contexte explicite de citation/réponse). Remplacement par canal : `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk` : inclure les statuts de canal sains dans la sortie heartbeat.
- `channels.defaults.heartbeat.showAlerts` : inclure les statuts dégradés/en erreur dans la sortie heartbeat.
- `channels.defaults.heartbeat.useIndicator` : afficher une sortie heartbeat compacte de type indicateur.

### WhatsApp

WhatsApp s'exécute via le canal web de la passerelle (Baileys Web). Il démarre automatiquement lorsqu'une session liée existe.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // coches bleues (false en mode auto-conversation)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

<Accordion title="WhatsApp multi-comptes">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Les commandes sortantes utilisent par défaut le compte `default` s'il est présent ; sinon le premier identifiant de compte configuré (trié).
- `channels.whatsapp.defaultAccount` facultatif remplace cette sélection par défaut lorsque sa valeur correspond à un identifiant de compte configuré.
- L'ancien répertoire d'authentification Baileys à compte unique est migré par `openclaw doctor` vers `whatsapp/default`.
- Remplacements par compte : `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: "partial", // off | partial | block | progress (default: off; opt in explicitly to avoid preview-edit rate limits)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Jeton du bot : `channels.telegram.botToken` ou `channels.telegram.tokenFile` (fichier ordinaire uniquement ; les liens symboliques sont rejetés), avec `TELEGRAM_BOT_TOKEN` comme repli pour le compte par défaut.
- `channels.telegram.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.
- Dans les configurations multi-comptes (2 identifiants de compte ou plus), définissez une valeur par défaut explicite (`channels.telegram.defaultAccount` ou `channels.telegram.accounts.default`) afin d'éviter le routage de repli ; `openclaw doctor` émet un avertissement lorsque cette valeur est absente ou invalide.
- `configWrites: false` bloque les écritures de configuration initiées par Telegram (migrations d'identifiants de supergroupe, `/config set|unset`).
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` configurent des liaisons ACP persistantes pour les sujets de forum (utilisez le format canonique `chatId:topic:topicId` dans `match.peer.id`). La sémantique des champs est partagée dans [ACP Agents](/fr/tools/acp-agents#channel-specific-settings).
- Les aperçus de flux Telegram utilisent `sendMessage` + `editMessageText` (fonctionne dans les discussions directes et de groupe).
- Stratégie de nouvelle tentative : voir [Stratégie de nouvelle tentative](/fr/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      chunkMode: "length", // length | newline
      streaming: "off", // off | partial | block | progress (progress maps to partial on Discord)
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in for sessions_spawn({ thread: true })
      },
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
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Jeton : `channels.discord.token`, avec `DISCORD_BOT_TOKEN` comme repli pour le compte par défaut.
- Les appels sortants directs qui fournissent un `token` Discord explicite utilisent ce jeton pour l'appel ; les paramètres de nouvelle tentative/de stratégie du compte proviennent toujours du compte sélectionné dans l'instantané d'exécution actif.
- `channels.discord.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.
- Utilisez `user:<id>` (DM) ou `channel:<id>` (canal de guilde) pour les cibles de livraison ; les identifiants numériques nus sont rejetés.
- Les slugs de guilde sont en minuscules avec les espaces remplacés par `-` ; les clés de canal utilisent le nom transformé en slug (sans `#`). Préférez les identifiants de guilde.
- Les messages rédigés par des bots sont ignorés par défaut. `allowBots: true` les active ; utilisez `allowBots: "mentions"` pour n'accepter que les messages de bot qui mentionnent le bot (les propres messages restent filtrés).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (et ses remplacements au niveau du canal) supprime les messages qui mentionnent un autre utilisateur ou rôle, mais pas le bot (hors @everyone/@here).
- `maxLinesPerMessage` (par défaut 17) divise les messages très hauts même s'ils font moins de 2000 caractères.
- `channels.discord.threadBindings` contrôle le routage lié aux threads Discord :
  - `enabled` : remplacement Discord pour les fonctionnalités de session liées aux threads (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` et la livraison/le routage liés)
  - `idleHours` : remplacement Discord pour la perte de focus automatique sur inactivité, en heures (`0` désactive)
  - `maxAgeHours` : remplacement Discord pour l'âge maximal strict, en heures (`0` désactive)
  - `spawnSubagentSessions` : option d'activation pour la création/liaison automatique de thread par `sessions_spawn({ thread: true })`
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` configurent des liaisons ACP persistantes pour les canaux et les threads (utilisez l'identifiant de canal/thread dans `match.peer.id`). La sémantique des champs est partagée dans [ACP Agents](/fr/tools/acp-agents#channel-specific-settings).
- `channels.discord.ui.components.accentColor` définit la couleur d'accent pour les conteneurs de composants Discord v2.
- `channels.discord.voice` active les conversations dans les canaux vocaux Discord ainsi que les remplacements facultatifs d'auto-jonction + TTS.
- `channels.discord.voice.daveEncryption` et `channels.discord.voice.decryptionFailureTolerance` sont transmis aux options DAVE de `@discordjs/voice` (`true` et `24` par défaut).
- OpenClaw tente également une récupération de réception vocale en quittant/rejoignant une session vocale après des échecs répétés de déchiffrement.
- `channels.discord.streaming` est la clé canonique du mode de flux. Les anciennes valeurs booléennes `streamMode` et `streaming` sont migrées automatiquement.
- `channels.discord.autoPresence` associe la disponibilité à l'exécution à la présence du bot (healthy => online, degraded => idle, exhausted => dnd) et autorise des remplacements facultatifs du texte d'état.
- `channels.discord.dangerouslyAllowNameMatching` réactive la correspondance sur nom/tag mutable (mode de compatibilité break-glass).
- `channels.discord.execApprovals` : livraison native des approbations exec et autorisation des approbateurs sur Discord.
  - `enabled` : `true`, `false` ou `"auto"` (par défaut). En mode auto, les approbations exec s'activent lorsque des approbateurs peuvent être résolus depuis `approvers` ou `commands.ownerAllowFrom`.
  - `approvers` : identifiants d'utilisateur Discord autorisés à approuver les demandes exec. Utilise `commands.ownerAllowFrom` comme repli lorsqu'il est omis.
  - `agentFilter` : liste d'autorisation facultative d'identifiants d'agent. Omettez-la pour transférer les approbations de tous les agents.
  - `sessionFilter` : modèles facultatifs de clé de session (sous-chaîne ou regex).
  - `target` : où envoyer les invites d'approbation. `"dm"` (par défaut) les envoie dans les DM des approbateurs, `"channel"` les envoie dans le canal d'origine, `"both"` les envoie aux deux. Lorsque la cible inclut `"channel"`, les boutons ne sont utilisables que par les approbateurs résolus.
  - `cleanupAfterResolve` : lorsqu'il vaut `true`, supprime les DM d'approbation après approbation, refus ou expiration.

**Modes de notification de réaction :** `off` (aucune), `own` (messages du bot, par défaut), `all` (tous les messages), `allowlist` (depuis `guilds.<id>.users` sur tous les messages).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON du compte de service : inline (`serviceAccount`) ou basé sur un fichier (`serviceAccountFile`).
- SecretRef pour le compte de service est également pris en charge (`serviceAccountRef`).
- Variables d'environnement de repli : `GOOGLE_CHAT_SERVICE_ACCOUNT` ou `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Utilisez `spaces/<spaceId>` ou `users/<userId>` pour les cibles de livraison.
- `channels.googlechat.dangerouslyAllowNameMatching` réactive la correspondance mutable sur le principal email (mode de compatibilité break-glass).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      textChunkLimit: 4000,
      chunkMode: "length",
      streaming: "partial", // off | partial | block | progress (mode aperçu)
      nativeStreaming: true, // utiliser l'API de streaming native Slack lorsque streaming=partial
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Mode socket** : nécessite `botToken` et `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` pour le repli env du compte par défaut).
- **Mode HTTP** : nécessite `botToken` plus `signingSecret` (à la racine ou par compte).
- `botToken`, `appToken`, `signingSecret` et `userToken` acceptent des chaînes
  en clair ou des objets SecretRef.
- Les instantanés de compte Slack exposent des champs de source/statut par identifiant comme
  `botTokenSource`, `botTokenStatus`, `appTokenStatus` et, en mode HTTP,
  `signingSecretStatus`. `configured_unavailable` signifie que le compte est
  configuré via SecretRef mais que le chemin de commande/d'exécution actuel n'a pas
  pu résoudre la valeur secrète.
- `configWrites: false` bloque les écritures de configuration initiées par Slack.
- `channels.slack.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.
- `channels.slack.streaming` est la clé canonique du mode de flux. Les anciennes valeurs booléennes `streamMode` et `streaming` sont migrées automatiquement.
- Utilisez `user:<id>` (DM) ou `channel:<id>` pour les cibles de livraison.

**Modes de notification de réaction :** `off`, `own` (par défaut), `all`, `allowlist` (depuis `reactionAllowlist`).

**Isolation de session par thread :** `thread.historyScope` est par thread (par défaut) ou partagé sur le canal. `thread.inheritParent` copie la transcription du canal parent vers les nouveaux threads.

- `typingReaction` ajoute une réaction temporaire au message Slack entrant pendant l'exécution d'une réponse, puis la retire à la fin. Utilisez un shortcode d'emoji Slack comme `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals` : livraison native des approbations exec et autorisation des approbateurs sur Slack. Même schéma que Discord : `enabled` (`true`/`false`/`"auto"`), `approvers` (identifiants d'utilisateur Slack), `agentFilter`, `sessionFilter` et `target` (`"dm"`, `"channel"` ou `"both"`).

| Groupe d'actions | Par défaut | Remarques               |
| ---------------- | ---------- | ----------------------- |
| reactions        | activé     | Réagir + lister les réactions |
| messages         | activé     | Lire/envoyer/modifier/supprimer |
| pins             | activé     | Épingler/désépingler/lister |
| memberInfo       | activé     | Informations sur les membres |
| emojiList        | activé     | Liste des emoji personnalisés |

### Mattermost

Mattermost est livré sous forme de plugin : `openclaw plugins install @openclaw/mattermost`.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // opt-in
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // URL explicite facultative pour les déploiements reverse-proxy/publics
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Modes de chat : `oncall` (répond sur @-mention, par défaut), `onmessage` (chaque message), `onchar` (messages commençant par un préfixe de déclenchement).

Lorsque les commandes natives Mattermost sont activées :

- `commands.callbackPath` doit être un chemin (par exemple `/api/channels/mattermost/command`), pas une URL complète.
- `commands.callbackUrl` doit résoudre vers le point de terminaison de la passerelle OpenClaw et être accessible depuis le serveur Mattermost.
- Les callbacks slash natifs sont authentifiés avec les jetons par commande renvoyés
  par Mattermost lors de l'enregistrement des commandes slash. Si l'enregistrement échoue ou si aucune
  commande n'est activée, OpenClaw rejette les callbacks avec
  `Unauthorized: invalid command token.`
- Pour les hôtes de callback privés/tailnet/internes, Mattermost peut nécessiter
  que `ServiceSettings.AllowedUntrustedInternalConnections` inclue l'hôte/le domaine de callback.
  Utilisez des valeurs hôte/domaine, pas des URL complètes.
- `channels.mattermost.configWrites` : autoriser ou refuser les écritures de configuration initiées par Mattermost.
- `channels.mattermost.requireMention` : exiger `@mention` avant de répondre dans les canaux.
- `channels.mattermost.groups.<channelId>.requireMention` : remplacement du filtrage par mention par canal (`"*"` pour la valeur par défaut).
- `channels.mattermost.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // liaison facultative au compte
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Modes de notification de réaction :** `off`, `own` (par défaut), `all`, `allowlist` (depuis `reactionAllowlist`).

- `channels.signal.account` : épingler le démarrage du canal à une identité de compte Signal spécifique.
- `channels.signal.configWrites` : autoriser ou refuser les écritures de configuration initiées par Signal.
- `channels.signal.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.

### BlueBubbles

BlueBubbles est le chemin iMessage recommandé (basé sur plugin, configuré sous `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, contrôles de groupe et actions avancées :
      // voir /channels/bluebubbles
    },
  },
}
```

- Chemins de clés du noyau couverts ici : `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- `channels.bluebubbles.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` peuvent lier des conversations BlueBubbles à des sessions ACP persistantes. Utilisez un handle BlueBubbles ou une chaîne cible (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) dans `match.peer.id`. Sémantique partagée des champs : [ACP Agents](/fr/tools/acp-agents#channel-specific-settings).
- La configuration complète du canal BlueBubbles est documentée dans [BlueBubbles](/fr/channels/bluebubbles).

### iMessage

OpenClaw lance `imsg rpc` (JSON-RPC sur stdio). Aucun démon ni port requis.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

- `channels.imessage.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.

- Nécessite un accès complet au disque pour la base de données Messages.
- Préférez les cibles `chat_id:<id>`. Utilisez `imsg chats --limit 20` pour lister les chats.
- `cliPath` peut pointer vers un wrapper SSH ; définissez `remoteHost` (`host` ou `user@host`) pour la récupération des pièces jointes via SCP.
- `attachmentRoots` et `remoteAttachmentRoots` restreignent les chemins des pièces jointes entrantes (par défaut : `/Users/*/Library/Messages/Attachments`).
- SCP utilise une vérification stricte de la clé d'hôte, alors assurez-vous que la clé de l'hôte relais existe déjà dans `~/.ssh/known_hosts`.
- `channels.imessage.configWrites` : autoriser ou refuser les écritures de configuration initiées par iMessage.
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` peuvent lier des conversations iMessage à des sessions ACP persistantes. Utilisez un handle normalisé ou une cible de chat explicite (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) dans `match.peer.id`. Sémantique partagée des champs : [ACP Agents](/fr/tools/acp-agents#channel-specific-settings).

<Accordion title="Exemple de wrapper SSH iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix est géré par extension et configuré sous `channels.matrix`.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- L'authentification par jeton utilise `accessToken` ; l'authentification par mot de passe utilise `userId` + `password`.
- `channels.matrix.proxy` route le trafic HTTP Matrix via un proxy HTTP(S) explicite. Les comptes nommés peuvent le remplacer avec `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` autorise les homeservers privés/internes. `proxy` et cette option réseau sont des contrôles indépendants.
- `channels.matrix.defaultAccount` sélectionne le compte préféré dans les configurations multi-comptes.
- `channels.matrix.autoJoin` vaut `off` par défaut, de sorte que les salles invitées et les nouvelles invitations de type DM sont ignorées tant que vous ne définissez pas `autoJoin: "allowlist"` avec `autoJoinAllowlist` ou `autoJoin: "always"`.
- `channels.matrix.execApprovals` : livraison native des approbations exec et autorisation des approbateurs sur Matrix.
  - `enabled` : `true`, `false` ou `"auto"` (par défaut). En mode auto, les approbations exec s'activent lorsque des approbateurs peuvent être résolus depuis `approvers` ou `commands.ownerAllowFrom`.
  - `approvers` : identifiants d'utilisateur Matrix (par ex. `@owner:example.org`) autorisés à approuver les demandes exec.
  - `agentFilter` : liste d'autorisation facultative d'identifiants d'agent. Omettez-la pour transférer les approbations de tous les agents.
  - `sessionFilter` : modèles facultatifs de clé de session (sous-chaîne ou regex).
  - `target` : où envoyer les invites d'approbation. `"dm"` (par défaut), `"channel"` (salle d'origine) ou `"both"`.
  - Remplacements par compte : `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` contrôle la manière dont les DM Matrix sont regroupés en sessions : `per-user` (par défaut) partage par pair routé, tandis que `per-room` isole chaque salle DM.
- Les sondes d'état Matrix et les recherches en direct dans l'annuaire utilisent la même stratégie proxy que le trafic à l'exécution.
- La configuration complète de Matrix, les règles de ciblage et les exemples de configuration sont documentés dans [Matrix](/fr/channels/matrix).

### Microsoft Teams

Microsoft Teams est géré par extension et configuré sous `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, stratégies d'équipe/de canal :
      // voir /channels/msteams
    },
  },
}
```

- Chemins de clés du noyau couverts ici : `channels.msteams`, `channels.msteams.configWrites`.
- La configuration complète de Teams (identifiants, webhook, stratégie DM/groupe, remplacements par équipe/par canal) est documentée dans [Microsoft Teams](/fr/channels/msteams).

### IRC

IRC est géré par extension et configuré sous `channels.irc`.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Chemins de clés du noyau couverts ici : `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- `channels.irc.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu'il correspond à un identifiant de compte configuré.
- La configuration complète du canal IRC (hôte/port/TLS/canaux/listes d'autorisation/filtrage par mention) est documentée dans [IRC](/fr/channels/irc).

### Multi-comptes (tous les canaux)

Exécutez plusieurs comptes par canal (chacun avec son propre `accountId`) :

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default` est utilisé lorsque `accountId` est omis (CLI + routage).
- Les jetons d'environnement ne s'appliquent qu'au compte **default**.
- Les paramètres de canal de base s'appliquent à tous les comptes sauf remplacement par compte.
- Utilisez `bindings[].match.accountId` pour router chaque compte vers un agent différent.
- Si vous ajoutez un compte non par défaut via `openclaw channels add` (ou l'onboarding du canal) alors que vous êtes encore sur une configuration de canal à compte unique de niveau supérieur, OpenClaw promeut d'abord les valeurs de niveau supérieur à portée de compte unique dans la map des comptes du canal afin que le compte d'origine continue de fonctionner. La plupart des canaux les déplacent dans `channels.<channel>.accounts.default` ; Matrix peut conserver à la place une cible nommée/par défaut existante correspondante.
- Les liaisons existantes uniquement au niveau du canal (sans `accountId`) continuent de correspondre au compte par défaut ; les liaisons à portée de compte restent facultatives.
- `openclaw doctor --fix` répare également les formes mixtes en déplaçant les valeurs de niveau supérieur à portée de compte unique vers le compte promu choisi pour ce canal. La plupart des canaux utilisent `accounts.default` ; Matrix peut conserver à la place une cible nommée/par défaut existante correspondante.

### Autres canaux d'extension

De nombreux canaux d'extension sont configurés comme `channels.<id>` et documentés dans leurs pages de canal dédiées (par exemple Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat et Twitch).
Voir l'index complet des canaux : [Canaux](/fr/channels).

### Filtrage par mention dans les chats de groupe

Les messages de groupe exigent par défaut une **mention requise** (mention de métadonnées ou motifs regex sûrs). Cela s'applique aux chats de groupe WhatsApp, Telegram, Discord, Google Chat et iMessage.

**Types de mention :**

- **Mentions de métadonnées** : @-mentions natives de la plateforme. Ignorées en mode auto-conversation WhatsApp.
- **Motifs de texte** : motifs regex sûrs dans `agents.list[].groupChat.mentionPatterns`. Les motifs invalides et les répétitions imbriquées non sûres sont ignorés.
- Le filtrage par mention n'est appliqué que lorsque la détection est possible (mentions natives ou au moins un motif).

```json5
{
  messages: {
    groupChat: { historyLimit: 50 },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` définit la valeur globale par défaut. Les canaux peuvent la remplacer avec `channels.<channel>.historyLimit` (ou par compte). Définissez `0` pour désactiver.

#### Limites d'historique DM

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Résolution : remplacement par DM → valeur par défaut du fournisseur → pas de limite (tout est conservé).

Pris en charge : `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Mode auto-conversation

Incluez votre propre numéro dans `allowFrom` pour activer le mode auto-conversation (ignore les @-mentions natives, répond uniquement aux motifs de texte) :

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Commandes (gestion des commandes de chat)

```json5
{
  commands: {
    native: "auto", // enregistrer les commandes natives lorsqu'elles sont prises en charge
    text: true, // analyser les /commands dans les messages de chat
    bash: false, // autoriser ! (alias : /bash)
    bashForegroundMs: 2000,
    config: false, // autoriser /config
    debug: false, // autoriser /debug
    restart: false, // autoriser /restart + outil de redémarrage de la passerelle
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Détails des commandes">

- Les commandes texte doivent être des messages **autonomes** commençant par `/`.
- `native: "auto"` active les commandes natives pour Discord/Telegram, et laisse Slack désactivé.
- Remplacement par canal : `channels.discord.commands.native` (booléen ou `"auto"`). `false` efface les commandes précédemment enregistrées.
- `channels.telegram.customCommands` ajoute des entrées supplémentaires au menu du bot Telegram.
- `bash: true` active `! <cmd>` pour le shell de l'hôte. Nécessite `tools.elevated.enabled` et que l'expéditeur soit dans `tools.elevated.allowFrom.<channel>`.
- `config: true` active `/config` (lecture/écriture de `openclaw.json`). Pour les clients `chat.send` de la passerelle, les écritures persistantes `/config set|unset` exigent aussi `operator.admin` ; la lecture seule `/config show` reste disponible pour les clients opérateur normaux avec portée d'écriture.
- `channels.<provider>.configWrites` contrôle les mutations de configuration par canal (par défaut : true).
- Pour les canaux multi-comptes, `channels.<provider>.accounts.<id>.configWrites` contrôle également les écritures ciblant ce compte (par exemple `/allowlist --config --account <id>` ou `/config set channels.<provider>.accounts.<id>...`).
- `allowFrom` est par fournisseur. Lorsqu'il est défini, c'est l'**unique** source d'autorisation (les listes d'autorisation/l'appairage du canal et `useAccessGroups` sont ignorés).
- `useAccessGroups: false` permet aux commandes de contourner les stratégies de groupe d'accès lorsque `allowFrom` n'est pas défini.

</Accordion>

---

## Valeurs par défaut des agents

### `agents.defaults.workspace`

Par défaut : `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Racine de dépôt facultative affichée dans la ligne Runtime du prompt système. Si elle n'est pas définie, OpenClaw la détecte automatiquement en remontant depuis l'espace de travail.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Liste d'autorisation des Skills par défaut facultative pour les agents qui ne définissent pas
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // hérite de github, weather
      { id: "docs", skills: ["docs-search"] }, // remplace les valeurs par défaut
      { id: "locked-down", skills: [] }, // aucun Skills
    ],
  },
}
```

- Omettez `agents.defaults.skills` pour des Skills non restreints par défaut.
- Omettez `agents.list[].skills` pour hériter des valeurs par défaut.
- Définissez `agents.list[].skills: []` pour n'avoir aucun Skills.
- Une liste non vide `agents.list[].skills` est l'ensemble final pour cet agent ; elle
  ne fusionne pas avec les valeurs par défaut.

### `agents.defaults.skipBootstrap`

Désactive la création automatique des fichiers bootstrap de l'espace de travail (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Contrôle quand les fichiers bootstrap de l'espace de travail sont injectés dans le prompt système. Par défaut : `"always"`.

- `"continuation-skip"` : les tours de continuation sûrs (après une réponse assistant terminée) ignorent la réinjection du bootstrap de l'espace de travail, ce qui réduit la taille du prompt. Les exécutions heartbeat et les nouvelles tentatives après compaction reconstruisent toujours le contexte.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Nombre maximal de caractères par fichier bootstrap d'espace de travail avant troncature. Par défaut : `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Nombre total maximal de caractères injectés sur l'ensemble des fichiers bootstrap de l'espace de travail. Par défaut : `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Contrôle le texte d'avertissement visible par l'agent lorsque le contexte bootstrap est tronqué.
Par défaut : `"once"`.

- `"off"` : ne jamais injecter de texte d'avertissement dans le prompt système.
- `"once"` : injecter l'avertissement une seule fois par signature de troncature unique (recommandé).
- `"always"` : injecter l'avertissement à chaque exécution lorsqu'une troncature existe.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

Taille maximale en pixels pour le côté le plus long des images dans les blocs image de transcription/outils avant les appels fournisseur.
Par défaut : `1200`.

Des valeurs plus basses réduisent généralement l'utilisation de vision-tokens et la taille de la charge utile des requêtes pour les exécutions riches en captures d'écran.
Des valeurs plus élevées préservent davantage de détails visuels.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Fuseau horaire pour le contexte du prompt système (pas les horodatages des messages). Utilise le fuseau horaire de l'hôte comme repli.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Format horaire dans le prompt système. Par défaut : `auto` (préférence de l'OS).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview"],
      },
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-i2v"],
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // paramètres fournisseur par défaut globaux
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 3,
    },
  },
}
```

- `model` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - La forme chaîne ne définit que le modèle principal.
  - La forme objet définit le principal plus les modèles de basculement ordonnés.
- `imageModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par le chemin d'outil `image` comme configuration de modèle de vision.
  - Également utilisé comme routage de repli lorsque le modèle sélectionné/par défaut ne peut pas accepter d'entrée image.
- `imageGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération d'images et toute future surface d'outil/plugin générant des images.
  - Valeurs typiques : `google/gemini-3.1-flash-image-preview` pour la génération d'images native Gemini, `fal/fal-ai/flux/dev` pour fal, ou `openai/gpt-image-1` pour OpenAI Images.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l'authentification/la clé API correspondante du fournisseur (par exemple `GEMINI_API_KEY` ou `GOOGLE_API_KEY` pour `google/*`, `OPENAI_API_KEY` pour `openai/*`, `FAL_KEY` pour `fal/*`).
  - S'il est omis, `image_generate` peut toujours inférer une valeur par défaut de fournisseur avec authentification. Il essaie d'abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération d'images enregistrés dans l'ordre des identifiants de fournisseur.
- `musicGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération musicale et l'outil intégré `music_generate`.
  - Valeurs typiques : `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` ou `minimax/music-2.5+`.
  - S'il est omis, `music_generate` peut toujours inférer une valeur par défaut de fournisseur avec authentification. Il essaie d'abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération musicale enregistrés dans l'ordre des identifiants de fournisseur.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l'authentification/la clé API correspondante du fournisseur.
- `videoGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération vidéo et l'outil intégré `video_generate`.
  - Valeurs typiques : `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` ou `qwen/wan2.7-r2v`.
  - S'il est omis, `video_generate` peut toujours inférer une valeur par défaut de fournisseur avec authentification. Il essaie d'abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération vidéo enregistrés dans l'ordre des identifiants de fournisseur.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l'authentification/la clé API correspondante du fournisseur.
  - Le fournisseur groupé de génération vidéo Qwen prend actuellement en charge jusqu'à 1 vidéo de sortie, 1 image d'entrée, 4 vidéos d'entrée, une durée de 10 secondes et des options au niveau du fournisseur `size`, `aspectRatio`, `resolution`, `audio` et `watermark`.
- `pdfModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par l'outil `pdf` pour le routage du modèle.
  - S'il est omis, l'outil PDF utilise `imageModel` comme repli, puis le modèle résolu de la session/par défaut.
- `pdfMaxBytesMb` : limite de taille PDF par défaut pour l'outil `pdf` lorsque `maxBytesMb` n'est pas passé à l'appel.
- `pdfMaxPages` : nombre maximal de pages pris en compte par le mode de repli d'extraction dans l'outil `pdf`.
- `verboseDefault` : niveau verbose par défaut pour les agents. Valeurs : `"off"`, `"on"`, `"full"`. Par défaut : `"off"`.
- `elevatedDefault` : niveau de sortie élevé par défaut pour les agents. Valeurs : `"off"`, `"on"`, `"ask"`, `"full"`. Par défaut : `"on"`.
- `model.primary` : format `provider/model` (ex. `openai/gpt-5.4`). Si vous omettez le fournisseur, OpenClaw essaie d'abord un alias, puis une correspondance unique de fournisseur configuré pour cet identifiant de modèle exact, et seulement ensuite revient au fournisseur par défaut configuré (comportement de compatibilité déprécié, donc préférez un `provider/model` explicite). Si ce fournisseur n'expose plus le modèle par défaut configuré, OpenClaw revient au premier fournisseur/modèle configuré au lieu d'exposer une valeur par défaut obsolète d'un fournisseur supprimé.
- `models` : catalogue de modèles configuré et liste d'autorisation pour `/model`. Chaque entrée peut inclure `alias` (raccourci) et `params` (spécifiques au fournisseur, par exemple `temperature`, `maxTokens`, `cacheRetention`, `context1m`).
- `params` : paramètres fournisseur globaux par défaut appliqués à tous les modèles. À définir dans `agents.defaults.params` (ex. `{ cacheRetention: "long" }`).
- Priorité de fusion de `params` (configuration) : `agents.defaults.params` (base globale) est remplacé par `agents.defaults.models["provider/model"].params` (par modèle), puis `agents.list[].params` (identifiant d'agent correspondant) remplace par clé. Voir [Prompt Caching](/fr/reference/prompt-caching) pour les détails.
- Les rédacteurs de configuration qui modifient ces champs (par exemple `/models set`, `/models set-image` et les commandes d'ajout/suppression de repli) enregistrent la forme objet canonique et préservent les listes de repli existantes quand c'est possible.
- `maxConcurrent` : nombre maximal d'exécutions d'agent parallèles entre les sessions (chaque session reste sérialisée). Par défaut : 4.

**Alias raccourcis intégrés** (ne s'appliquent que lorsque le modèle est dans `agents.defaults.models`) :

| Alias               | Modèle                                 |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Vos alias configurés ont toujours priorité sur les valeurs par défaut.

Les modèles Z.AI GLM-4.x activent automatiquement le mode thinking sauf si vous définissez `--thinking off` ou `agents.defaults.models["zai/<model>"].params.thinking` vous-même.
Les modèles Z.AI activent `tool_stream` par défaut pour le streaming des appels d'outils. Définissez `agents.defaults.models["zai/<model>"].params.tool_stream` à `false` pour le désactiver.
Les modèles Anthropic Claude 4.6 utilisent `adaptive` par défaut pour thinking lorsqu'aucun niveau de thinking explicite n'est défini.

### `agents.defaults.cliBackends`

Backends CLI facultatifs pour les exécutions de repli texte seul (sans appels d'outils). Utile comme secours lorsque les fournisseurs d'API échouent.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          modelArg: "--model",
          sessionArg: "--session",
          sessionMode: "existing",
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
        },
      },
    },
  },
}
```

- Les backends CLI sont d'abord textuels ; les outils sont toujours désactivés.
- Les sessions sont prises en charge lorsque `sessionArg` est défini.
- Le passage d'images est pris en charge lorsque `imageArg` accepte des chemins de fichiers.

### `agents.defaults.heartbeat`

Exécutions heartbeat périodiques.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m désactive
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        lightContext: false, // default: false; true keeps only HEARTBEAT.md from workspace bootstrap files
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every` : chaîne de durée (ms/s/m/h). Par défaut : `30m` (authentification par clé API) ou `1h` (authentification OAuth). Définissez `0m` pour désactiver.
- `suppressToolErrorWarnings` : lorsque `true`, supprime les charges utiles d'avertissement d'erreur d'outil pendant les exécutions heartbeat.
- `directPolicy` : stratégie de livraison directe/DM. `allow` (par défaut) autorise la livraison à cible directe. `block` supprime la livraison à cible directe et émet `reason=dm-blocked`.
- `lightContext` : lorsque `true`, les exécutions heartbeat utilisent un contexte bootstrap léger et ne conservent que `HEARTBEAT.md` parmi les fichiers bootstrap de l'espace de travail.
- `isolatedSession` : lorsque `true`, chaque exécution heartbeat s'exécute dans une session fraîche sans historique de conversation précédent. Même modèle d'isolation que cron `sessionTarget: "isolated"`. Réduit le coût en tokens par heartbeat d'environ 100K à environ 2–5K tokens.
- Par agent : définissez `agents.list[].heartbeat`. Lorsqu'un agent définit `heartbeat`, **seuls ces agents** exécutent des heartbeats.
- Les heartbeats exécutent des tours complets d'agent — des intervalles plus courts consomment plus de tokens.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // used when identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] disables reinjection
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        notifyUser: true, // send a brief notice when compaction starts (default: false)
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 6000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with the exact silent token NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

- `mode` : `default` ou `safeguard` (résumé par blocs pour les longs historiques). Voir [Compaction](/fr/concepts/compaction).
- `timeoutSeconds` : nombre maximal de secondes autorisées pour une opération unique de compaction avant qu'OpenClaw ne l'interrompe. Par défaut : `900`.
- `identifierPolicy` : `strict` (par défaut), `off` ou `custom`. `strict` préfixe des instructions intégrées de conservation des identifiants opaques lors du résumé de compaction.
- `identifierInstructions` : texte personnalisé facultatif de préservation des identifiants utilisé lorsque `identifierPolicy=custom`.
- `postCompactionSections` : noms facultatifs de sections H2/H3 d'AGENTS.md à réinjecter après compaction. Par défaut `["Session Startup", "Red Lines"]` ; définissez `[]` pour désactiver la réinjection. Lorsqu'il n'est pas défini ou explicitement défini sur cette paire par défaut, les anciens titres `Every Session`/`Safety` sont également acceptés comme repli hérité.
- `model` : remplacement facultatif `provider/model-id` uniquement pour le résumé de compaction. Utilisez-le lorsque la session principale doit conserver un modèle mais que les résumés de compaction doivent s'exécuter sur un autre ; lorsqu'il n'est pas défini, la compaction utilise le modèle principal de la session.
- `notifyUser` : lorsque `true`, envoie une brève notification à l'utilisateur au début de la compaction (par exemple, "Compacting context..."). Désactivé par défaut pour garder la compaction silencieuse.
- `memoryFlush` : tour agentique silencieux avant l'auto-compaction pour stocker des mémoires durables. Ignoré lorsque l'espace de travail est en lecture seule.

### `agents.defaults.contextPruning`

Émonde les **anciens résultats d'outils** du contexte en mémoire avant l'envoi au LLM. **Ne modifie pas** l'historique de session sur disque.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // durée (ms/s/m/h), unité par défaut : minutes
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] },
      },
    },
  },
}
```

<Accordion title="Comportement du mode cache-ttl">

- `mode: "cache-ttl"` active les passes d'élagage.
- `ttl` contrôle à quelle fréquence l'élagage peut s'exécuter de nouveau (après le dernier accès au cache).
- L'élagage tronque d'abord en douceur les résultats d'outil surdimensionnés, puis efface complètement les résultats d'outil plus anciens si nécessaire.

**Soft-trim** conserve le début + la fin et insère `...` au milieu.

**Hard-clear** remplace l'intégralité du résultat d'outil par l'espace réservé.

Remarques :

- Les blocs image ne sont jamais tronqués/effacés.
- Les ratios sont basés sur les caractères (approximation), pas sur des nombres exacts de tokens.
- S'il existe moins de `keepLastAssistants` messages assistant, l'élagage est ignoré.

</Accordion>

Voir [Élagage de session](/fr/concepts/session-pruning) pour les détails de comportement.

### Streaming par blocs

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (utiliser minMs/maxMs)
    },
  },
}
```

- Les canaux autres que Telegram nécessitent `*.blockStreaming: true` explicitement pour activer les réponses par blocs.
- Remplacements par canal : `channels.<channel>.blockStreamingCoalesce` (et variantes par compte). Signal/Slack/Discord/Google Chat utilisent par défaut `minChars: 1500`.
- `humanDelay` : pause aléatoire entre les réponses par blocs. `natural` = 800–2500 ms. Remplacement par agent : `agents.list[].humanDelay`.

Voir [Streaming](/fr/concepts/streaming) pour le comportement + les détails de segmentation.

### Indicateurs de saisie

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Valeurs par défaut : `instant` pour les chats directs/mentions, `message` pour les chats de groupe sans mention.
- Remplacements par session : `session.typingMode`, `session.typingIntervalSeconds`.

Voir [Indicateurs de saisie](/fr/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Sandboxing facultatif pour l'agent embarqué. Voir [Sandboxing](/fr/gateway/sandboxing) pour le guide complet.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        backend: "docker", // docker | ssh | openshell
        scope: "agent", // session | agent | shared
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / contenus inline également pris en charge :
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

<Accordion title="Détails du sandbox">

**Backend :**

- `docker` : runtime Docker local (par défaut)
- `ssh` : runtime distant générique via SSH
- `openshell` : runtime OpenShell

Lorsque `backend: "openshell"` est sélectionné, les paramètres spécifiques au runtime sont déplacés vers
`plugins.entries.openshell.config`.

**Configuration du backend SSH :**

- `target` : cible SSH au format `user@host[:port]`
- `command` : commande du client SSH (par défaut : `ssh`)
- `workspaceRoot` : racine distante absolue utilisée pour les espaces de travail par portée
- `identityFile` / `certificateFile` / `knownHostsFile` : fichiers locaux existants transmis à OpenSSH
- `identityData` / `certificateData` / `knownHostsData` : contenus inline ou SecretRefs que OpenClaw matérialise dans des fichiers temporaires au runtime
- `strictHostKeyChecking` / `updateHostKeys` : réglages de stratégie de clé d'hôte OpenSSH

**Priorité d'authentification SSH :**

- `identityData` a priorité sur `identityFile`
- `certificateData` a priorité sur `certificateFile`
- `knownHostsData` a priorité sur `knownHostsFile`
- Les valeurs `*Data` basées sur SecretRef sont résolues depuis l'instantané actif du runtime des secrets avant le démarrage de la session sandbox

**Comportement du backend SSH :**

- initialise l'espace de travail distant une seule fois après création ou recréation
- conserve ensuite l'espace de travail SSH distant comme référence canonique
- route `exec`, les outils de fichier et les chemins média via SSH
- ne synchronise pas automatiquement les modifications distantes vers l'hôte
- ne prend pas en charge les conteneurs de navigateur sandboxés

**Accès à l'espace de travail :**

- `none` : espace de travail sandbox par portée sous `~/.openclaw/sandboxes`
- `ro` : espace de travail sandbox à `/workspace`, espace de travail d'agent monté en lecture seule à `/agent`
- `rw` : espace de travail d'agent monté en lecture/écriture à `/workspace`

**Portée :**

- `session` : conteneur + espace de travail par session
- `agent` : un conteneur + un espace de travail par agent (par défaut)
- `shared` : conteneur et espace de travail partagés (pas d'isolation inter-sessions)

**Configuration du plugin OpenShell :**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // mirror | remote
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // facultatif
          gatewayEndpoint: "https://lab.example", // facultatif
          policy: "strict", // identifiant de stratégie OpenShell facultatif
          providers: ["openai"], // facultatif
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**Mode OpenShell :**

- `mirror` : initialise le distant depuis le local avant exec, resynchronise après exec ; l'espace de travail local reste canonique
- `remote` : initialise le distant une seule fois lors de la création du sandbox, puis conserve l'espace de travail distant comme canonique

En mode `remote`, les modifications locales sur l'hôte effectuées en dehors d'OpenClaw ne sont pas synchronisées automatiquement dans le sandbox après l'étape d'initialisation.
Le transport utilise SSH vers le sandbox OpenShell, mais le plugin gère le cycle de vie du sandbox et la synchronisation miroir facultative.

**`setupCommand`** s'exécute une fois après la création du conteneur (via `sh -lc`). Nécessite un accès réseau sortant, une racine accessible en écriture et un utilisateur root.

**Les conteneurs utilisent par défaut `network: "none"`** — définissez `"bridge"` (ou un réseau bridge personnalisé) si l'agent a besoin d'un accès sortant.
`"host"` est bloqué. `"container:<id>"` est bloqué par défaut sauf si vous définissez explicitement
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (break-glass).

**Les pièces jointes entrantes** sont préparées dans `media/inbound/*` dans l'espace de travail actif.

**`docker.binds`** monte des répertoires hôtes supplémentaires ; les montages globaux et par agent sont fusionnés.

**Navigateur sandboxé** (`sandbox.browser.enabled`) : Chromium + CDP dans un conteneur. L'URL noVNC est injectée dans le prompt système. Ne nécessite pas `browser.enabled` dans `openclaw.json`.
L'accès observateur noVNC utilise l'authentification VNC par défaut et OpenClaw émet une URL à jeton de courte durée (au lieu d'exposer le mot de passe dans l'URL partagée).

- `allowHostControl: false` (par défaut) bloque les sessions sandboxées qui ciblent le navigateur hôte.
- `network` vaut `openclaw-sandbox-browser` par défaut (réseau bridge dédié). Définissez `bridge` uniquement lorsque vous souhaitez explicitement une connectivité bridge globale.
- `cdpSourceRange` restreint éventuellement l'entrée CDP à la périphérie du conteneur à une plage CIDR (par exemple `172.21.0.1/32`).
- `sandbox.browser.binds` monte des répertoires hôtes supplémentaires uniquement dans le conteneur de navigateur sandboxé. Lorsqu'il est défini (y compris `[]`), il remplace `docker.binds` pour le conteneur du navigateur.
- Les options de lancement par défaut sont définies dans `scripts/sandbox-browser-entrypoint.sh` et réglées pour les hôtes de conteneur :
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-3d-apis`
  - `--disable-gpu`
  - `--disable-software-rasterizer`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-features=TranslateUI`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--renderer-process-limit=2`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--disable-extensions` (activé par défaut)
  - `--disable-3d-apis`, `--disable-software-rasterizer` et `--disable-gpu` sont
    activés par défaut et peuvent être désactivés avec
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` si l'utilisation de WebGL/3D l'exige.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` réactive les extensions si votre flux de travail
    en dépend.
  - `--renderer-process-limit=2` peut être modifié avec
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` ; définissez `0` pour utiliser la
    limite de processus par défaut de Chromium.
  - plus `--no-sandbox` et `--disable-setuid-sandbox` lorsque `noSandbox` est activé.
  - Les valeurs par défaut sont la base de l'image du conteneur ; utilisez une image navigateur personnalisée avec un point
    d'entrée personnalisé pour changer les valeurs par défaut du conteneur.

</Accordion>

Le sandboxing navigateur et `sandbox.docker.binds` sont actuellement limités à Docker.

Construisez les images :

```bash
scripts/sandbox-setup.sh           # image sandbox principale
scripts/sandbox-browser-setup.sh   # image navigateur facultative
```

### `agents.list` (remplacements par agent)

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // ou { primary, fallbacks }
        thinkingDefault: "high", // remplacement du niveau thinking par agent
        reasoningDefault: "on", // remplacement de la visibilité reasoning par agent
        fastModeDefault: false, // remplacement du mode rapide par agent
        params: { cacheRetention: "none" }, // remplace par clé les params de defaults.models correspondants
        skills: ["docs-search"], // remplace agents.defaults.skills lorsqu'il est défini
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id` : identifiant d'agent stable (obligatoire).
- `default` : lorsque plusieurs sont définis, le premier l'emporte (avertissement consigné). Si aucun n'est défini, la première entrée de la liste est la valeur par défaut.
- `model` : la forme chaîne remplace uniquement `primary` ; la forme objet `{ primary, fallbacks }` remplace les deux (`[]` désactive les fallbacks globaux). Les tâches cron qui ne remplacent que `primary` héritent toujours des fallbacks par défaut sauf si vous définissez `fallbacks: []`.
- `params` : paramètres de flux par agent fusionnés au-dessus de l'entrée de modèle sélectionnée dans `agents.defaults.models`. Utilisez-les pour des remplacements spécifiques à l'agent comme `cacheRetention`, `temperature` ou `maxTokens` sans dupliquer l'ensemble du catalogue de modèles.
- `skills` : liste d'autorisation des Skills facultative par agent. Si elle est omise, l'agent hérite de `agents.defaults.skills` lorsqu'il est défini ; une liste explicite remplace les valeurs par défaut au lieu de fusionner, et `[]` signifie aucun Skills.
- `thinkingDefault` : niveau thinking par défaut facultatif par agent (`off | minimal | low | medium | high | xhigh | adaptive`). Remplace `agents.defaults.thinkingDefault` pour cet agent lorsqu'aucun remplacement par message ou session n'est défini.
- `reasoningDefault` : visibilité reasoning par défaut facultative par agent (`on | off | stream`). S'applique lorsqu'aucun remplacement reasoning par message ou session n'est défini.
- `fastModeDefault` : valeur par défaut facultative par agent pour le mode rapide (`true | false`). S'applique lorsqu'aucun remplacement de mode rapide par message ou session n'est défini.
- `runtime` : descripteur de runtime facultatif par agent. Utilisez `type: "acp"` avec les valeurs par défaut `runtime.acp` (`agent`, `backend`, `mode`, `cwd`) lorsque l'agent doit utiliser par défaut des sessions de harnais ACP.
- `identity.avatar` : chemin relatif à l'espace de travail, URL `http(s)` ou URI `data:`.
- `identity` déduit des valeurs par défaut : `ackReaction` depuis `emoji`, `mentionPatterns` depuis `name`/`emoji`.
- `subagents.allowAgents` : liste d'autorisation d'identifiants d'agent pour `sessions_spawn` (`["*"]` = n'importe lequel ; par défaut : même agent uniquement).
- Garde-fou d'héritage du sandbox : si la session demandeuse est sandboxée, `sessions_spawn` rejette les cibles qui s'exécuteraient sans sandbox.
- `subagents.requireAgentId` : lorsqu'il vaut true, bloque les appels `sessions_spawn` qui omettent `agentId` (force une sélection explicite du profil ; par défaut : false).

---

## Routage multi-agent

Exécutez plusieurs agents isolés dans une seule passerelle. Voir [Multi-Agent](/fr/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Champs de correspondance de liaison

- `type` (facultatif) : `route` pour le routage normal (absence de type = route), `acp` pour les liaisons de conversation ACP persistantes.
- `match.channel` (obligatoire)
- `match.accountId` (facultatif ; `*` = n'importe quel compte ; omis = compte par défaut)
- `match.peer` (facultatif ; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (facultatif ; spécifique au canal)
- `acp` (facultatif ; uniquement pour `type: "acp"`) : `{ mode, label, cwd, backend }`

**Ordre de correspondance déterministe :**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exact, sans peer/guild/team)
5. `match.accountId: "*"` (à l'échelle du canal)
6. Agent par défaut

Dans chaque niveau, la première entrée `bindings` correspondante l'emporte.

Pour les entrées `type: "acp"`, OpenClaw résout par identité exacte de conversation (`match.channel` + compte + `match.peer.id`) et n'utilise pas l'ordre de niveaux de liaison route ci-dessus.

### Profils d'accès par agent

<Accordion title="Accès complet (sans sandbox)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Outils + espace de travail en lecture seule">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Aucun accès au système de fichiers (messagerie uniquement)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Voir [Sandbox et outils multi-agent](/fr/tools/multi-agent-sandbox-tools) pour les détails de priorité.

---

## Session

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    parentForkMaxTokens: 100000, // ignorer le fork du thread parent au-dessus de ce nombre de tokens (0 désactive)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // durée ou false
      maxDiskBytes: "500mb", // budget strict facultatif
      highWaterBytes: "400mb", // cible de nettoyage facultative
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    mainKey: "main", // hérité (le runtime utilise toujours "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Détails des champs de session">

- **`scope`** : stratégie de regroupement de base des sessions pour les contextes de chat de groupe.
  - `per-sender` (par défaut) : chaque expéditeur obtient une session isolée dans un contexte de canal.
  - `global` : tous les participants d'un contexte de canal partagent une seule session (à utiliser uniquement lorsqu'un contexte partagé est souhaité).
- **`dmScope`** : comment les DM sont regroupés.
  - `main` : tous les DM partagent la session principale.
  - `per-peer` : isole par identifiant d'expéditeur entre les canaux.
  - `per-channel-peer` : isole par canal + expéditeur (recommandé pour les boîtes de réception multi-utilisateurs).
  - `per-account-channel-peer` : isole par compte + canal + expéditeur (recommandé pour le multi-comptes).
- **`identityLinks`** : mappe des identifiants canoniques vers des pairs préfixés par fournisseur pour le partage de session inter-canaux.
- **`reset`** : stratégie principale de réinitialisation. `daily` réinitialise à `atHour` heure locale ; `idle` réinitialise après `idleMinutes`. Lorsque les deux sont configurés, la première échéance l'emporte.
- **`resetByType`** : remplacements par type (`direct`, `group`, `thread`). L'ancien `dm` est accepté comme alias de `direct`.
- **`parentForkMaxTokens`** : nombre maximal de `totalTokens` autorisé dans la session parente lors de la création d'une session de thread dérivée (par défaut `100000`).
  - Si le `totalTokens` parent dépasse cette valeur, OpenClaw démarre une nouvelle session de thread au lieu d'hériter de l'historique de transcription parent.
  - Définissez `0` pour désactiver ce garde-fou et toujours autoriser le fork parent.
- **`mainKey`** : champ hérité. Le runtime utilise désormais toujours `"main"` pour le compartiment principal des chats directs.
- **`agentToAgent.maxPingPongTurns`** : nombre maximal de tours de réponse en retour entre agents pendant les échanges agent-à-agent (entier, plage : `0`–`5`). `0` désactive l'enchaînement ping-pong.
- **`sendPolicy`** : correspondance par `channel`, `chatType` (`direct|group|channel`, avec alias hérité `dm`), `keyPrefix` ou `rawKeyPrefix`. Le premier refus l'emporte.
- **`maintenance`** : nettoyage + contrôles de rétention du magasin de sessions.
  - `mode` : `warn` émet uniquement des avertissements ; `enforce` applique le nettoyage.
  - `pruneAfter` : seuil d'âge pour les entrées obsolètes (par défaut `30d`).
  - `maxEntries` : nombre maximal d'entrées dans `sessions.json` (par défaut `500`).
  - `rotateBytes` : fait pivoter `sessions.json` lorsqu'il dépasse cette taille (par défaut `10mb`).
  - `resetArchiveRetention` : rétention des archives de transcription `*.reset.<timestamp>`. Par défaut, utilise `pruneAfter` ; définissez `false` pour désactiver.
  - `maxDiskBytes` : budget disque facultatif pour le répertoire de sessions. En mode `warn`, il enregistre des avertissements ; en mode `enforce`, il supprime d'abord les artefacts/sessions les plus anciens.
  - `highWaterBytes` : cible facultative après nettoyage du budget. Par défaut, `80%` de `maxDiskBytes`.
- **`threadBindings`** : valeurs par défaut globales pour les fonctionnalités de session liées aux threads.
  - `enabled` : interrupteur maître par défaut (les fournisseurs peuvent remplacer ; Discord utilise `channels.discord.threadBindings.enabled`)
  - `idleHours` : perte de focus automatique sur inactivité par défaut en heures (`0` désactive ; remplaçable par les fournisseurs)
  - `maxAgeHours` : âge maximal strict par défaut en heures (`0` désactive ; remplaçable par les fournisseurs)

</Accordion>

---

## Messages

```json5
{
  messages: {
    responsePrefix: "🦞", // ou "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all
    removeAckAfterReply: false,
    queue: {
      mode: "collect", // steer | followup | collect | steer-backlog | steer+backlog | queue | interrupt
      debounceMs: 1000,
      cap: 20,
      drop: "summarize", // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 désactive
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Préfixe de réponse

Remplacements par canal/compte : `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Résolution (le plus spécifique l'emporte) : compte → canal → global. `""` désactive et arrête la cascade. `"auto"` dérive `[{identity.name}]`.

**Variables de modèle :**

| Variable          | Description            | Exemple                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Nom court du modèle    | `claude-opus-4-6`           |
| `{modelFull}`     | Identifiant complet du modèle | `anthropic/claude-opus-4-6` |
| `{provider}`      | Nom du fournisseur     | `anthropic`                 |
| `{thinkingLevel}` | Niveau thinking actuel | `high`, `low`, `off`        |
| `{identity.name}` | Nom de l'identité de l'agent | (identique à `"auto"`)      |

Les variables ne sont pas sensibles à la casse. `{think}` est un alias de `{thinkingLevel}`.

### Réaction d'accusé de réception

- Par défaut : `identity.emoji` de l'agent actif, sinon `"👀"`. Définissez `""` pour désactiver.
- Remplacements par canal : `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Ordre de résolution : compte → canal → `messages.ackReaction` → repli identité.
- Portée : `group-mentions` (par défaut), `group-all`, `direct`, `all`.
- `removeAckAfterReply` : retire l'accusé de réception après la réponse sur Slack, Discord et Telegram.
- `messages.statusReactions.enabled` : active les réactions d'état du cycle de vie sur Slack, Discord et Telegram.
  Sur Slack et Discord, si non défini, les réactions d'état restent activées lorsque les réactions d'accusé de réception sont actives.
  Sur Telegram, définissez-le explicitement à `true` pour activer les réactions d'état du cycle de vie.

### Anti-rebond entrant

Regroupe les messages texte rapides du même expéditeur en un seul tour d'agent. Les médias/pièces jointes déclenchent immédiatement le vidage. Les commandes de contrôle contournent l'anti-rebond.

### TTS (text-to-speech)

```json5
{
  messages: {
    tts: {
      auto: "always", // off | always | inbound | tagged
      mode: "final", // final | all
      provider: "elevenlabs",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      openai: {
        apiKey: "openai_api_key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        voice: "alloy",
      },
    },
  },
}
```

- `auto` contrôle l'auto-TTS. `/tts off|always|inbound|tagged` remplace cette valeur par session.
- `summaryModel` remplace `agents.defaults.model.primary` pour le résumé automatique.
- `modelOverrides` est activé par défaut ; `modelOverrides.allowProvider` vaut `false` par défaut (opt-in).
- Les clés API utilisent `ELEVENLABS_API_KEY`/`XI_API_KEY` et `OPENAI_API_KEY` comme repli.
- `openai.baseUrl` remplace le point de terminaison OpenAI TTS. L'ordre de résolution est : config, puis `OPENAI_TTS_BASE_URL`, puis `https://api.openai.com/v1`.
- Lorsque `openai.baseUrl` pointe vers un point de terminaison non OpenAI, OpenClaw le traite comme un serveur TTS compatible OpenAI et assouplit la validation du modèle/de la voix.

---

## Talk

Valeurs par défaut du mode Talk (macOS/iOS/Android).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
    },
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- `talk.provider` doit correspondre à une clé dans `talk.providers` lorsque plusieurs fournisseurs Talk sont configurés.
- Les anciennes clés Talk plates (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) sont uniquement conservées pour compatibilité et migrées automatiquement dans `talk.providers.<provider>`.
- Les identifiants vocaux utilisent `ELEVENLABS_VOICE_ID` ou `SAG_VOICE_ID` comme repli.
- `providers.*.apiKey` accepte des chaînes en clair ou des objets SecretRef.
- Le repli `ELEVENLABS_API_KEY` ne s'applique que lorsqu'aucune clé API Talk n'est configurée.
- `providers.*.voiceAliases` permet aux directives Talk d'utiliser des noms conviviaux.
- `silenceTimeoutMs` contrôle le temps d'attente du mode Talk après le silence de l'utilisateur avant d'envoyer la transcription. S'il n'est pas défini, la fenêtre de pause par défaut de la plateforme est conservée (`700 ms sur macOS et Android, 900 ms sur iOS`).

---

## Outils

### Profils d'outils

`tools.profile` définit une liste d'autorisation de base avant `tools.allow`/`tools.deny` :

L'onboarding local définit par défaut `tools.profile: "coding"` dans les nouvelles configurations lorsqu'il n'est pas défini (les profils explicites existants sont conservés).

| Profil      | Inclut                                                                                                                        |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | `session_status` uniquement                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                    |
| `full`      | Aucune restriction (identique à non défini)                                                                                   |

### Groupes d'outils

| Groupe              | Outils                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` est accepté comme alias de `exec`)                                           |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                   |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                            |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                    |
| `group:ui`         | `browser`, `canvas`                                                                                                      |
| `group:automation` | `cron`, `gateway`                                                                                                        |
| `group:messaging`  | `message`                                                                                                                |
| `group:nodes`      | `nodes`                                                                                                                  |
| `group:agents`     | `agents_list`                                                                                                            |
| `group:media`      | `image`, `image_generate`, `video_generate`, `tts`                                                                       |
| `group:openclaw`   | Tous les outils intégrés (exclut les plugins fournisseur)                                                                |

### `tools.allow` / `tools.deny`

Stratégie globale d'autorisation/refus des outils (le refus l'emporte). Insensible à la casse, prend en charge les jokers `*`. S'applique même lorsque le sandbox Docker est désactivé.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Restreint davantage les outils pour des fournisseurs ou modèles spécifiques. Ordre : profil de base → profil fournisseur → allow/deny.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.elevated`

Contrôle l'accès exec élevé hors du sandbox :

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- Le remplacement par agent (`agents.list[].tools.elevated`) ne peut qu'ajouter des restrictions.
- `/elevated on|off|ask|full` stocke l'état par session ; les directives inline s'appliquent à un seul message.
- `exec` élevé contourne le sandboxing et utilise le chemin d'échappement configuré (`gateway` par défaut, ou `node` lorsque la cible exec est `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      applyPatch: {
        enabled: false,
        allowModels: ["gpt-5.4"],
      },
    },
  },
}
```

### `tools.loopDetection`

Les contrôles de sécurité de boucle d'outils sont **désactivés par défaut**. Définissez `enabled: true` pour activer la détection.
Les paramètres peuvent être définis globalement dans `tools.loopDetection` et remplacés par agent dans `agents.list[].tools.loopDetection`.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

- `historySize` : historique maximal des appels d'outils conservé pour l'analyse de boucle.
- `warningThreshold` : seuil de motif répétitif sans progression pour les avertissements.
- `criticalThreshold` : seuil répétitif plus élevé pour bloquer les boucles critiques.
- `globalCircuitBreakerThreshold` : seuil d'arrêt strict pour toute exécution sans progression.
- `detectors.genericRepeat` : avertit sur les appels répétés au même outil avec les mêmes arguments.
- `detectors.knownPollNoProgress` : avertit/bloque sur les outils de sondage connus (`process.poll`, `command_status`, etc.).
- `detectors.pingPong` : avertit/bloque sur les motifs de paire alternée sans progression.
- Si `warningThreshold >= criticalThreshold` ou `criticalThreshold >= globalCircuitBreakerThreshold`, la validation échoue.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // ou variable d'environnement BRAVE_API_KEY
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // facultatif ; omettre pour l'auto-détection
        maxChars: 50000,
        maxCharsCap: 50000,
        maxResponseBytes: 2000000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

### `tools.media`

Configure la compréhension des médias entrants (image/audio/vidéo) :

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // opt-in : envoyer directement au canal la musique/vidéo asynchrone terminée
      },
      audio: {
        enabled: true,
        maxBytes: 20971520,
        scope: {
          default: "deny",
          rules: [{ action: "allow", match: { chatType: "direct" } }],
        },
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] },
        ],
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },
  },
}
```

<Accordion title="Champs d'entrée de modèle média">

**Entrée fournisseur** (`type: "provider"` ou omis) :

- `provider` : identifiant du fournisseur d'API (`openai`, `anthropic`, `google`/`gemini`, `groq`, etc.)
- `model` : remplacement d'identifiant de modèle
- `profile` / `preferredProfile` : sélection du profil `auth-profiles.json`

**Entrée CLI** (`type: "cli"`) :

- `command` : exécutable à lancer
- `args` : arguments avec modèles (prend en charge `{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}`, etc.)

**Champs communs :**

- `capabilities` : liste facultative (`image`, `audio`, `video`). Valeurs par défaut : `openai`/`anthropic`/`minimax` → image, `google` → image+audio+video, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language` : remplacements par entrée.
- Les échecs passent à l'entrée suivante.

L'authentification fournisseur suit l'ordre standard : `auth-profiles.json` → variables d'environnement → `models.providers.*.apiKey`.

**Champs de fin asynchrone :**

- `asyncCompletion.directSend` : lorsque `true`, les tâches terminées `music_generate`
  et `video_generate` essaient d'abord une livraison directe au canal. Par défaut : `false`
  (ancien chemin de réveil de session demandeuse/livraison par modèle).

</Accordion>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Contrôle quelles sessions peuvent être ciblées par les outils de session (`sessions_list`, `sessions_history`, `sessions_send`).

Par défaut : `tree` (session actuelle + sessions qu'elle a engendrées, comme les sous-agents).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

Remarques :

- `self` : uniquement la clé de session actuelle.
- `tree` : session actuelle + sessions engendrées par la session actuelle (sous-agents).
- `agent` : toute session appartenant à l'identifiant d'agent actuel (peut inclure d'autres utilisateurs si vous exécutez des sessions par expéditeur sous le même identifiant d'agent).
- `all` : n'importe quelle session. Le ciblage inter-agents exige toujours `tools.agentToAgent`.
- Limitation sandbox : lorsque la session actuelle est sandboxée et que `agents.defaults.sandbox.sessionToolsVisibility="spawned"`, la visibilité est forcée à `tree` même si `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

Contrôle la prise en charge des pièces jointes inline pour `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in : définir true pour autoriser les pièces jointes de fichier inline
        maxTotalBytes: 5242880, // 5 Mo au total sur tous les fichiers
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 Mo par fichier
        retainOnSessionKeep: false, // conserver les pièces jointes lorsque cleanup="keep"
      },
    },
  },
}
```

Remarques :

- Les pièces jointes ne sont prises en charge que pour `runtime: "subagent"`. Le runtime ACP les rejette.
- Les fichiers sont matérialisés dans l'espace de travail enfant à `.openclaw/attachments/<uuid>/` avec un `.manifest.json`.
- Le contenu des pièces jointes est automatiquement expurgé de la persistance de transcription.
- Les entrées base64 sont validées avec des contrôles stricts d'alphabet/remplissage et une garde de taille avant décodage.
- Les permissions sont `0700` pour les répertoires et `0600` pour les fichiers.
- Le nettoyage suit la stratégie `cleanup` : `delete` supprime toujours les pièces jointes ; `keep` ne les conserve que lorsque `retainOnSessionKeep: true`.

### `tools.experimental`

Indicateurs d'outils intégrés expérimentaux. Désactivés par défaut sauf lorsqu'une règle d'activation automatique spécifique au runtime s'applique.

```json5
{
  tools: {
    experimental: {
      planTool: true, // activer update_plan expérimental
    },
  },
}
```

Remarques :

- `planTool` : active l'outil structuré `update_plan` pour le suivi de travaux multi-étapes non triviaux.
- Par défaut : `false` pour les fournisseurs non OpenAI. Les exécutions OpenAI et OpenAI Codex l'activent automatiquement.
- Lorsqu'il est activé, le prompt système ajoute également des consignes d'utilisation afin que le modèle ne l'utilise que pour des travaux substantiels et conserve au plus une étape `in_progress`.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model` : modèle par défaut pour les sous-agents lancés. S'il est omis, les sous-agents héritent du modèle de l'appelant.
- `allowAgents` : liste d'autorisation par défaut des identifiants d'agent cible pour `sessions_spawn` lorsque l'agent demandeur ne définit pas son propre `subagents.allowAgents` (`["*"]` = n'importe lequel ; par défaut : même agent uniquement).
- `runTimeoutSeconds` : délai d'expiration par défaut (secondes) pour `sessions_spawn` lorsque l'appel d'outil omet `runTimeoutSeconds`. `0` signifie aucun délai.
- Stratégie d'outil par sous-agent : `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Fournisseurs personnalisés et URL de base

OpenClaw utilise le catalogue de modèles intégré. Ajoutez des fournisseurs personnalisés via `models.providers` dans la configuration ou `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (par défaut) | replace
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

- Utilisez `authHeader: true` + `headers` pour des besoins d'authentification personnalisés.
- Remplacez la racine de configuration d'agent avec `OPENCLAW_AGENT_DIR` (ou `PI_CODING_AGENT_DIR`, un ancien alias de variable d'environnement).
- Priorité de fusion pour les identifiants de fournisseur correspondants :
  - Les valeurs `baseUrl` non vides de l'agent `models.json` l'emportent.
  - Les valeurs `apiKey` non vides de l'agent l'emportent uniquement lorsque ce fournisseur n'est pas géré par SecretRef dans le contexte actuel de configuration/profil d'authentification.
  - Les valeurs `apiKey` des fournisseurs gérés par SecretRef sont actualisées depuis les marqueurs de source (`ENV_VAR_NAME` pour les références env, `secretref-managed` pour les références fichier/exec) au lieu de persister les secrets résolus.
  - Les valeurs d'en-tête fournisseur gérées par SecretRef sont actualisées depuis les marqueurs de source (`secretref-env:ENV_VAR_NAME` pour les références env, `secretref-managed` pour les références fichier/exec).
  - Les valeurs `apiKey`/`baseUrl` vides ou absentes dans l'agent utilisent `models.providers` dans la configuration comme repli.
  - Pour les modèles correspondants, `contextWindow`/`maxTokens` utilisent la valeur la plus élevée entre la configuration explicite et les valeurs implicites du catalogue.
  - Pour les modèles correspondants, `contextTokens` préserve un plafond explicite du runtime lorsqu'il est présent ; utilisez-le pour limiter le contexte effectif sans modifier les métadonnées natives du modèle.
  - Utilisez `models.mode: "replace"` lorsque vous voulez que la configuration réécrive complètement `models.json`.
  - La persistance des marqueurs suit la source comme autorité : les marqueurs sont écrits à partir de l'instantané de configuration actif de la source (avant résolution), et non à partir des valeurs secrètes résolues du runtime.

### Détails des champs fournisseur

- `models.mode` : comportement du catalogue fournisseur (`merge` ou `replace`).
- `models.providers` : map de fournisseurs personnalisés indexée par identifiant fournisseur.
- `models.providers.*.api` : adaptateur de requête (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai`, etc.).
- `models.providers.*.apiKey` : identifiant du fournisseur (préférez SecretRef/substitution env).
- `models.providers.*.auth` : stratégie d'authentification (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat` : pour Ollama + `openai-completions`, injecter `options.num_ctx` dans les requêtes (par défaut : `true`).
- `models.providers.*.authHeader` : forcer le transport de l'identifiant dans l'en-tête `Authorization` lorsque nécessaire.
- `models.providers.*.baseUrl` : URL de base de l'API amont.
- `models.providers.*.headers` : en-têtes statiques supplémentaires pour le routage proxy/tenant.
- `models.providers.*.request` : remplacements de transport pour les requêtes HTTP model-provider.
  - `request.headers` : en-têtes supplémentaires (fusionnés avec les valeurs par défaut du fournisseur). Les valeurs acceptent SecretRef.
  - `request.auth` : remplacement de stratégie d'authentification. Modes : `"provider-default"` (utilise l'authentification intégrée du fournisseur), `"authorization-bearer"` (avec `token`), `"header"` (avec `headerName`, `value`, `prefix` facultatif).
  - `request.proxy` : remplacement de proxy HTTP. Modes : `"env-proxy"` (utilise les variables d'environnement `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (avec `url`). Les deux modes acceptent un sous-objet `tls` facultatif.
  - `request.tls` : remplacement TLS pour les connexions directes. Champs : `ca`, `cert`, `key`, `passphrase` (acceptent tous SecretRef), `serverName`, `insecureSkipVerify`.
- `models.providers.*.models` : entrées explicites du catalogue de modèles fournisseur.
- `models.providers.*.models.*.contextWindow` : métadonnées de fenêtre de contexte native du modèle.
- `models.providers.*.models.*.contextTokens` : plafond facultatif du contexte au runtime. Utilisez-le lorsque vous voulez un budget de contexte effectif plus faible que le `contextWindow` natif du modèle.
- `models.providers.*.models.*.compat.supportsDeveloperRole` : indication facultative de compatibilité. Pour `api: "openai-completions"` avec un `baseUrl` non natif non vide (hôte différent de `api.openai.com`), OpenClaw force cette valeur à `false` au runtime. Un `baseUrl` vide/omis conserve le comportement OpenAI par défaut.
- `plugins.entries.amazon-bedrock.config.discovery` : racine des paramètres d'auto-détection Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled` : activer/désactiver la détection implicite.
- `plugins.entries.amazon-bedrock.config.discovery.region` : région AWS pour la détection.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter` : filtre facultatif d'identifiant fournisseur pour une détection ciblée.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval` : intervalle d'interrogation pour l'actualisation de la détection.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow` : fenêtre de contexte de repli pour les modèles détectés.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens` : jetons de sortie maximaux de repli pour les modèles détectés.

### Exemples de fournisseurs

<Accordion title="Cerebras (GLM 4.6 / 4.7)">

```json5
{
  env: { CEREBRAS_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: {
        primary: "cerebras/zai-glm-4.7",
        fallbacks: ["cerebras/zai-glm-4.6"],
      },
      models: {
        "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
        "cerebras/zai-glm-4.6": { alias: "GLM 4.6 (Cerebras)" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
          { id: "zai-glm-4.6", name: "GLM 4.6 (Cerebras)" },
        ],
      },
    },
  },
}
```

Utilisez `cerebras/zai-glm-4.7` pour Cerebras ; `zai/glm-4.7` pour Z.AI direct.

</Accordion>

<Accordion title="OpenCode">

```json5
{
  agents: {
    defaults: {
      model: { primary: "opencode/claude-opus-4-6" },
      models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
    },
  },
}
```

Définissez `OPENCODE_API_KEY` (ou `OPENCODE_ZEN_API_KEY`). Utilisez des références `opencode/...` pour le catalogue Zen ou `opencode-go/...` pour le catalogue Go. Raccourci : `openclaw onboard --auth-choice opencode-zen` ou `openclaw onboard --auth-choice opencode-go`.

</Accordion>

<Accordion title="Z.AI (GLM-4.7)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-4.7" },
      models: { "zai/glm-4.7": {} },
    },
  },
}
```

Définissez `ZAI_API_KEY`. `z.ai/*` et `z-ai/*` sont acceptés comme alias. Raccourci : `openclaw onboard --auth-choice zai-api-key`.

- Point de terminaison général : `https://api.z.ai/api/paas/v4`
- Point de terminaison de code (par défaut) : `https://api.z.ai/api/coding/paas/v4`
- Pour le point de terminaison général, définissez un fournisseur personnalisé avec remplacement de l'URL de base.

</Accordion>

<Accordion title="Moonshot AI (Kimi)">

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: { "moonshot/kimi-k2.5": { alias: "Kimi K2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
        ],
      },
    },
  },
}
```

Pour le point de terminaison Chine : `baseUrl: "https://api.moonshot.cn/v1"` ou `openclaw onboard --auth-choice moonshot-api-key-cn`.

Les points de terminaison Moonshot natifs annoncent la compatibilité d'utilisation du streaming sur le transport partagé
`openai-completions`, et OpenClaw la détermine désormais à partir des capacités du point de terminaison
plutôt qu'à partir du seul identifiant intégré du fournisseur.

</Accordion>

<Accordion title="Kimi Coding">

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: { "kimi/kimi-code": { alias: "Kimi Code" } },
    },
  },
}
```

Compatible Anthropic, fournisseur intégré. Raccourci : `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic (compatible Anthropic)">

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

L'URL de base ne doit pas inclure `/v1` (le client Anthropic l'ajoute). Raccourci : `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (direct)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "minimax/MiniMax-M2.7" },
      models: {
        "minimax/MiniMax-M2.7": { alias: "Minimax" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Définissez `MINIMAX_API_KEY`. Raccourcis :
`openclaw onboard --auth-choice minimax-global-api` ou
`openclaw onboard --auth-choice minimax-cn-api`.
Le catalogue de modèles utilise désormais M2.7 uniquement par défaut.
Sur le chemin de streaming compatible Anthropic, OpenClaw désactive thinking de MiniMax
par défaut sauf si vous définissez explicitement `thinking` vous-même. `/fast on` ou
`params.fastMode: true` réécrit `MiniMax-M2.7` en
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="Modèles locaux (LM Studio)">

Voir [Modèles locaux](/fr/gateway/local-models). En bref : exécutez un grand modèle local via l'API LM Studio Responses sur un matériel sérieux ; conservez les modèles hébergés fusionnés comme repli.

</Accordion>

---

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // ou chaîne en clair
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled` : liste d'autorisation facultative pour les Skills groupés uniquement (les Skills gérés/de l'espace de travail ne sont pas affectés).
- `load.extraDirs` : racines Skills partagées supplémentaires (priorité la plus faible).
- `install.preferBrew` : lorsque `true`, préfère les installateurs Homebrew lorsque `brew` est
  disponible avant de revenir à d'autres types d'installateurs.
- `install.nodeManager` : préférence du gestionnaire Node pour les spécifications
  `metadata.openclaw.install` (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` désactive un Skills même s'il est groupé/installé.
- `entries.<skillKey>.apiKey` : champ pratique de clé API au niveau du Skills (lorsqu'il est pris en charge par le Skills).

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Chargés depuis `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions`, plus `plugins.load.paths`.
- La découverte accepte les plugins OpenClaw natifs ainsi que les bundles Codex et Claude compatibles, y compris les bundles Claude sans manifeste avec disposition par défaut.
- **Les modifications de configuration nécessitent un redémarrage de la passerelle.**
- `allow` : liste d'autorisation facultative (seuls les plugins listés sont chargés). `deny` l'emporte.
- `plugins.entries.<id>.apiKey` : champ pratique de clé API au niveau du plugin (lorsqu'il est pris en charge par le plugin).
- `plugins.entries.<id>.env` : map de variables d'environnement propre au plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection` : lorsque `false`, le noyau bloque `before_prompt_build` et ignore les champs de mutation du prompt provenant de l'ancien `before_agent_start`, tout en conservant les anciens `modelOverride` et `providerOverride`. S'applique aux hooks natifs du plugin et aux répertoires de hooks fournis par bundle pris en charge.
- `plugins.entries.<id>.subagent.allowModelOverride` : fait explicitement confiance à ce plugin pour demander des remplacements `provider` et `model` par exécution pour les exécutions de sous-agent en arrière-plan.
- `plugins.entries.<id>.subagent.allowedModels` : liste d'autorisation facultative des cibles canoniques `provider/model` pour les remplacements de sous-agent approuvés. Utilisez `"*"` uniquement lorsque vous voulez intentionnellement autoriser n'importe quel modèle.
- `plugins.entries.<id>.config` : objet de configuration défini par le plugin (validé par le schéma du plugin OpenClaw natif lorsque disponible).
- `plugins.entries.firecrawl.config.webFetch` : paramètres du fournisseur de récupération web Firecrawl.
  - `apiKey` : clé API Firecrawl (accepte SecretRef). Utilise `plugins.entries.firecrawl.config.webSearch.apiKey`, l'ancien `tools.web.fetch.firecrawl.apiKey` ou la variable d'environnement `FIRECRAWL_API_KEY` comme repli.
  - `baseUrl` : URL de base de l'API Firecrawl (par défaut : `https://api.firecrawl.dev`).
  - `onlyMainContent` : extraire uniquement le contenu principal des pages (par défaut : `true`).
  - `maxAgeMs` : âge maximal du cache en millisecondes (par défaut : `172800000` / 2 jours).
  - `timeoutSeconds` : délai d'expiration de la requête de scraping en secondes (par défaut : `60`).
- `plugins.entries.xai.config.xSearch` : paramètres xAI X Search (recherche web Grok).
  - `enabled` : activer le fournisseur X Search.
  - `model` : modèle Grok à utiliser pour la recherche (par ex. `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming` : paramètres de memory dreaming (expérimental). Voir [Dreaming](/fr/concepts/dreaming) pour les phases et seuils.
  - `enabled` : interrupteur maître de dreaming (par défaut `false`).
  - `frequency` : cadence cron pour chaque balayage complet de dreaming (`"0 3 * * *"` par défaut).
  - la stratégie de phase et les seuils sont des détails d'implémentation (pas des clés de configuration destinées aux utilisateurs).
- Les plugins bundle Claude activés peuvent aussi fournir des valeurs par défaut Pi intégrées depuis `settings.json` ; OpenClaw les applique comme paramètres d'agent assainis, pas comme correctifs de configuration OpenClaw bruts.
- `plugins.slots.memory` : choisir l'identifiant du plugin mémoire actif, ou `"none"` pour désactiver les plugins mémoire.
- `plugins.slots.contextEngine` : choisir l'identifiant du plugin moteur de contexte actif ; vaut `"legacy"` par défaut sauf si vous en installez et sélectionnez un autre.
- `plugins.installs` : métadonnées d'installation gérées par CLI utilisées par `openclaw plugins update`.
  - Inclut `source`, `spec`, `sourcePath`, `installPath`, `version`, `resolvedName`, `resolvedVersion`, `resolvedSpec`, `integrity`, `shasum`, `resolvedAt`, `installedAt`.
  - Considérez `plugins.installs.*` comme un état géré ; préférez les commandes CLI aux modifications manuelles.

Voir [Plugins](/fr/tools/plugin).

---

## Browser

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // mode trusted-network par défaut
      // allowPrivateNetwork: true, // ancien alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` désactive `act:evaluate` et `wait --fn`.
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` vaut `true` par défaut lorsqu'il n'est pas défini (modèle trusted-network).
- Définissez `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` pour une navigation Browser strictement limitée au public.
- En mode strict, les points de terminaison de profil CDP distants (`profiles.*.cdpUrl`) sont soumis au même blocage des réseaux privés pendant les vérifications de disponibilité/découverte.
- `ssrfPolicy.allowPrivateNetwork` reste pris en charge comme ancien alias.
- En mode strict, utilisez `ssrfPolicy.hostnameAllowlist` et `ssrfPolicy.allowedHostnames` pour des exceptions explicites.
- Les profils distants sont attach-only (start/stop/reset désactivés).
- `profiles.*.cdpUrl` accepte `http://`, `https://`, `ws://` et `wss://`.
  Utilisez HTTP(S) lorsque vous voulez que OpenClaw découvre `/json/version` ; utilisez WS(S)
  lorsque votre fournisseur vous donne une URL WebSocket DevTools directe.
- Les profils `existing-session` sont limités à l'hôte et utilisent Chrome MCP au lieu de CDP.
- Les profils `existing-session` peuvent définir `userDataDir` pour cibler un profil spécifique
  d'un navigateur basé sur Chromium tel que Brave ou Edge.
- Les profils `existing-session` conservent les limites actuelles de la route Chrome MCP :
  actions pilotées par instantané/référence au lieu d'un ciblage par sélecteur CSS, hooks d'envoi d'un seul fichier,
  pas de remplacements de délai de boîte de dialogue, pas de `wait --load networkidle`, et pas de
  `responsebody`, d'export PDF, d'interception de téléchargement ni d'actions par lot.
- Les profils locaux gérés `openclaw` attribuent automatiquement `cdpPort` et `cdpUrl` ; ne
  définissez `cdpUrl` explicitement que pour un CDP distant.
- Ordre d'auto-détection : navigateur par défaut s'il est basé sur Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- Service de contrôle : loopback uniquement (port dérivé de `gateway.port`, par défaut `18791`).
- `extraArgs` ajoute des indicateurs de lancement supplémentaires au démarrage local de Chromium (par exemple
  `--disable-gpu`, dimensionnement de fenêtre ou indicateurs de débogage).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, texte court, URL d'image ou URI data
    },
  },
}
```

- `seamColor` : couleur d'accent pour le chrome UI de l'application native (teinte de bulle du mode Talk, etc.).
- `assistant` : remplacement d'identité de l'interface de contrôle. Utilise l'identité de l'agent actif comme repli.

---

## Passerelle

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // ou OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // pour mode=trusted-proxy ; voir /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // allowedOrigins: ["https://control.example.com"], // requis pour une Control UI non loopback
      // dangerouslyAllowHostHeaderOriginFallback: false, // mode dangereux de repli d'origine à partir de l'en-tête Host
      // allowInsecureAuth: false,
      // dangerouslyDisableDeviceAuth: false,
    },
    remote: {
      url: "ws://gateway.tailnet:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Facultatif. Par défaut false.
    allowRealIpFallback: false,
    tools: {
      // Refus HTTP supplémentaires pour /tools/invoke
      deny: ["browser"],
      // Retirer des outils de la liste de refus HTTP par défaut
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Détails des champs de passerelle">

- `mode` : `local` (exécuter la passerelle) ou `remote` (se connecter à une passerelle distante). La passerelle refuse de démarrer sauf si la valeur est `local`.
- `port` : port multiplexé unique pour WS + HTTP. Priorité : `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind` : `auto`, `loopback` (par défaut), `lan` (`0.0.0.0`), `tailnet` (IP Tailscale uniquement) ou `custom`.
- **Alias bind hérités** : utilisez les valeurs de mode bind dans `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), pas les alias d'hôte (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Remarque Docker** : la liaison par défaut `loopback` écoute sur `127.0.0.1` à l'intérieur du conteneur. Avec le réseau bridge Docker (`-p 18789:18789`), le trafic arrive sur `eth0`, donc la passerelle est inaccessible. Utilisez `--network host`, ou définissez `bind: "lan"` (ou `bind: "custom"` avec `customBindHost: "0.0.0.0"`) pour écouter sur toutes les interfaces.
- **Auth** : requise par défaut. Les liaisons non loopback exigent l'authentification de la passerelle. En pratique, cela signifie un jeton/mot de passe partagé ou un reverse proxy sensible à l'identité avec `gateway.auth.mode: "trusted-proxy"`. L'assistant d'onboarding génère un jeton par défaut.
- Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés (y compris via SecretRefs), définissez explicitement `gateway.auth.mode` à `token` ou `password`. Le démarrage et les flux d'installation/réparation du service échouent lorsque les deux sont configurés et que le mode n'est pas déf