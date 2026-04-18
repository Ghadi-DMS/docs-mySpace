---
read_when:
    - Vous avez besoin de la sémantique exacte de la configuration au niveau des champs ou des valeurs par défaut.
    - Vous validez des blocs de configuration de canal, de modèle, de Gateway ou d’outil.
summary: Référence de configuration du Gateway pour les clés OpenClaw principales, les valeurs par défaut et les liens vers les références dédiées des sous-systèmes
title: Référence de configuration
x-i18n:
    generated_at: "2026-04-18T06:43:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4b504c9c6b47d7a327a0acf6934561c9b2606c01cc8ebe5526ccde73033d759f
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Référence de configuration

Référence de configuration principale pour `~/.openclaw/openclaw.json`. Pour une vue d’ensemble orientée tâches, voir [Configuration](/fr/gateway/configuration).

Cette page couvre les principales surfaces de configuration d’OpenClaw et renvoie ailleurs lorsqu’un sous-système dispose de sa propre référence plus détaillée. Elle **n’essaie pas** d’intégrer sur une seule page chaque catalogue de commandes propre à un canal/Plugin ni chaque réglage approfondi de mémoire/QMD.

Source de vérité du code :

- `openclaw config schema` affiche le schéma JSON actif utilisé pour la validation et l’interface utilisateur Control, avec les métadonnées bundle/plugin/channel fusionnées lorsqu’elles sont disponibles
- `config.schema.lookup` renvoie un nœud de schéma circonscrit à un chemin pour les outils d’exploration détaillée
- `pnpm config:docs:check` / `pnpm config:docs:gen` valident le hachage de base des documents de configuration par rapport à la surface de schéma actuelle

Références détaillées dédiées :

- [Référence de configuration de la mémoire](/fr/reference/memory-config) pour `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations` et la configuration de Dreaming sous `plugins.entries.memory-core.config.dreaming`
- [Commandes slash](/fr/tools/slash-commands) pour le catalogue actuel des commandes intégrées + bundle
- les pages du canal/Plugin propriétaire pour les surfaces de commande propres au canal

Le format de configuration est **JSON5** (commentaires + virgules finales autorisés). Tous les champs sont facultatifs — OpenClaw utilise des valeurs par défaut sûres lorsqu’ils sont omis.

---

## Canaux

Chaque canal démarre automatiquement lorsque sa section de configuration existe (sauf si `enabled: false`).

### Accès DM et groupe

Tous les canaux prennent en charge des politiques de DM et des politiques de groupe :

| Politique de DM     | Comportement                                                    |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (par défaut) | Les expéditeurs inconnus reçoivent un code d’appairage à usage unique ; le propriétaire doit approuver |
| `allowlist`         | Uniquement les expéditeurs dans `allowFrom` (ou le stockage d’autorisation appairé) |
| `open`              | Autoriser tous les DM entrants (nécessite `allowFrom: ["*"]`)   |
| `disabled`          | Ignorer tous les DM entrants                                    |

| Politique de groupe   | Comportement                                          |
| --------------------- | ----------------------------------------------------- |
| `allowlist` (par défaut) | Uniquement les groupes correspondant à la liste d’autorisation configurée |
| `open`                | Contourner les listes d’autorisation de groupe (le filtrage par mention s’applique toujours) |
| `disabled`            | Bloquer tous les messages de groupe/salon             |

<Note>
`channels.defaults.groupPolicy` définit la valeur par défaut lorsque `groupPolicy` d’un fournisseur n’est pas défini.
Les codes d’appairage expirent après 1 heure. Les demandes d’appairage DM en attente sont limitées à **3 par canal**.
Si un bloc fournisseur est entièrement absent (`channels.<provider>` absent), la politique de groupe d’exécution revient à `allowlist` (échec en mode fermé) avec un avertissement au démarrage.
</Note>

### Remplacements de modèle par canal

Utilisez `channels.modelByChannel` pour épingler des ID de canal spécifiques à un modèle. Les valeurs acceptent `provider/model` ou des alias de modèle configurés. Le mappage de canal s’applique lorsqu’une session n’a pas déjà un remplacement de modèle (par exemple, défini via `/model`).

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

### Valeurs par défaut des canaux et Heartbeat

Utilisez `channels.defaults` pour le comportement partagé de politique de groupe et de Heartbeat entre les fournisseurs :

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

- `channels.defaults.groupPolicy` : politique de groupe de repli lorsqu’un `groupPolicy` au niveau fournisseur n’est pas défini.
- `channels.defaults.contextVisibility` : mode de visibilité du contexte supplémentaire par défaut pour tous les canaux. Valeurs : `all` (par défaut, inclure tout le contexte de citation/fil/historique), `allowlist` (inclure uniquement le contexte des expéditeurs autorisés), `allowlist_quote` (identique à allowlist mais conserve le contexte explicite de citation/réponse). Remplacement par canal : `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk` : inclure les statuts de canaux sains dans la sortie Heartbeat.
- `channels.defaults.heartbeat.showAlerts` : inclure les statuts dégradés/en erreur dans la sortie Heartbeat.
- `channels.defaults.heartbeat.useIndicator` : afficher une sortie Heartbeat compacte de type indicateur.

### WhatsApp

WhatsApp fonctionne via le canal web du Gateway (Baileys Web). Il démarre automatiquement lorsqu’une session liée existe.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // coches bleues (false en mode auto-discussion)
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

<Accordion title="WhatsApp multi-compte">

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

- Les commandes sortantes utilisent par défaut le compte `default` s’il est présent ; sinon, le premier ID de compte configuré (trié).
- `channels.whatsapp.defaultAccount` facultatif remplace cette sélection de compte par défaut de repli lorsqu’il correspond à un ID de compte configuré.
- L’ancien répertoire d’authentification Baileys mono-compte est migré par `openclaw doctor` vers `whatsapp/default`.
- Remplacements par compte : `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

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

- Jeton du bot : `channels.telegram.botToken` ou `channels.telegram.tokenFile` (fichier ordinaire uniquement ; les liens symboliques sont rejetés), avec `TELEGRAM_BOT_TOKEN` comme repli pour le compte par défaut.
- `channels.telegram.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.
- Dans les configurations multi-comptes (2 ID de compte ou plus), définissez une valeur par défaut explicite (`channels.telegram.defaultAccount` ou `channels.telegram.accounts.default`) pour éviter le routage de repli ; `openclaw doctor` avertit lorsque cela est absent ou invalide.
- `configWrites: false` bloque les écritures de configuration initiées par Telegram (migrations d’ID de supergroupe, `/config set|unset`).
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` configurent des liaisons ACP persistantes pour les sujets de forum (utilisez le format canonique `chatId:topic:topicId` dans `match.peer.id`). La sémantique des champs est partagée dans [Agents ACP](/fr/tools/acp-agents#channel-specific-settings).
- Les aperçus de flux Telegram utilisent `sendMessage` + `editMessageText` (fonctionne dans les discussions directes et de groupe).
- Politique de nouvelle tentative : voir [Politique de nouvelle tentative](/fr/concepts/retry).

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

- Jeton : `channels.discord.token`, avec `DISCORD_BOT_TOKEN` comme repli pour le compte par défaut.
- Les appels sortants directs qui fournissent un `token` Discord explicite utilisent ce jeton pour l’appel ; les paramètres de nouvelle tentative/politique du compte proviennent toujours du compte sélectionné dans l’instantané d’exécution actif.
- `channels.discord.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.
- Utilisez `user:<id>` (DM) ou `channel:<id>` (canal de guilde) pour les cibles de livraison ; les ID numériques seuls sont rejetés.
- Les slugs de guilde sont en minuscules avec les espaces remplacés par `-` ; les clés de canal utilisent le nom transformé en slug (sans `#`). Préférez les ID de guilde.
- Les messages rédigés par des bots sont ignorés par défaut. `allowBots: true` les active ; utilisez `allowBots: "mentions"` pour n’accepter que les messages de bots qui mentionnent le bot (les propres messages restent filtrés).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (et les remplacements au niveau du canal) ignore les messages qui mentionnent un autre utilisateur ou rôle mais pas le bot (à l’exclusion de @everyone/@here).
- `maxLinesPerMessage` (17 par défaut) scinde les messages hauts même lorsqu’ils restent sous 2000 caractères.
- `channels.discord.threadBindings` contrôle le routage lié aux fils Discord :
  - `enabled` : remplacement Discord pour les fonctionnalités de session liées aux fils (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` et la livraison/le routage liés)
  - `idleHours` : remplacement Discord pour la perte de focus automatique après inactivité en heures (`0` désactive)
  - `maxAgeHours` : remplacement Discord pour l’âge maximal strict en heures (`0` désactive)
  - `spawnSubagentSessions` : option d’activation pour la création/liaison automatique de fils avec `sessions_spawn({ thread: true })`
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` configurent des liaisons ACP persistantes pour les canaux et les fils (utilisez l’ID de canal/fil dans `match.peer.id`). La sémantique des champs est partagée dans [Agents ACP](/fr/tools/acp-agents#channel-specific-settings).
- `channels.discord.ui.components.accentColor` définit la couleur d’accent pour les conteneurs Discord components v2.
- `channels.discord.voice` active les conversations dans les canaux vocaux Discord ainsi que les remplacements facultatifs d’auto-jonction + TTS.
- `channels.discord.voice.daveEncryption` et `channels.discord.voice.decryptionFailureTolerance` sont transmis aux options DAVE de `@discordjs/voice` (`true` et `24` par défaut).
- OpenClaw tente également une récupération de réception vocale en quittant/rejoignant une session vocale après des échecs répétés de déchiffrement.
- `channels.discord.streaming` est la clé canonique du mode de flux. Les anciennes valeurs `streamMode` et booléennes `streaming` sont migrées automatiquement.
- `channels.discord.autoPresence` associe la disponibilité d’exécution à la présence du bot (sain => online, dégradé => idle, épuisé => dnd) et autorise des remplacements facultatifs du texte d’état.
- `channels.discord.dangerouslyAllowNameMatching` réactive la correspondance par nom/tag mutable (mode de compatibilité « break-glass »).
- `channels.discord.execApprovals` : livraison des approbations d’exécution native à Discord et autorisation des approbateurs.
  - `enabled` : `true`, `false` ou `"auto"` (par défaut). En mode auto, les approbations d’exécution s’activent lorsque des approbateurs peuvent être résolus à partir de `approvers` ou `commands.ownerAllowFrom`.
  - `approvers` : ID d’utilisateurs Discord autorisés à approuver les demandes d’exécution. Revient à `commands.ownerAllowFrom` lorsqu’il est omis.
  - `agentFilter` : liste d’autorisation facultative des ID d’agents. Omettez-la pour transmettre les approbations pour tous les agents.
  - `sessionFilter` : motifs facultatifs de clé de session (sous-chaîne ou regex).
  - `target` : où envoyer les invites d’approbation. `"dm"` (par défaut) envoie vers les DM des approbateurs, `"channel"` envoie vers le canal d’origine, `"both"` envoie vers les deux. Lorsque la cible inclut `"channel"`, les boutons ne sont utilisables que par les approbateurs résolus.
  - `cleanupAfterResolve` : lorsqu’il vaut `true`, supprime les DM d’approbation après approbation, refus ou expiration.

**Modes de notification de réaction :** `off` (aucune), `own` (messages du bot, par défaut), `all` (tous les messages), `allowlist` (depuis `guilds.<id>.users` sur tous les messages).

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

- JSON du compte de service : intégré (`serviceAccount`) ou basé sur un fichier (`serviceAccountFile`).
- SecretRef du compte de service également pris en charge (`serviceAccountRef`).
- Variables d’environnement de repli : `GOOGLE_CHAT_SERVICE_ACCOUNT` ou `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Utilisez `spaces/<spaceId>` ou `users/<userId>` pour les cibles de livraison.
- `channels.googlechat.dangerouslyAllowNameMatching` réactive la correspondance mutable du principal d’e-mail (mode de compatibilité « break-glass »).

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
      streaming: {
        mode: "partial", // off | partial | block | progress
        nativeTransport: true, // use Slack native streaming API when mode=partial
      },
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

- Le **mode socket** exige à la fois `botToken` et `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` pour le repli d’environnement du compte par défaut).
- Le **mode HTTP** exige `botToken` plus `signingSecret` (à la racine ou par compte).
- `botToken`, `appToken`, `signingSecret` et `userToken` acceptent des chaînes en clair
  ou des objets SecretRef.
- Les instantanés de compte Slack exposent des champs de source/statut par identifiant tels que
  `botTokenSource`, `botTokenStatus`, `appTokenStatus` et, en mode HTTP,
  `signingSecretStatus`. `configured_unavailable` signifie que le compte est
  configuré via SecretRef mais que le chemin actuel de commande/d’exécution n’a pas pu
  résoudre la valeur du secret.
- `configWrites: false` bloque les écritures de configuration initiées par Slack.
- `channels.slack.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.
- `channels.slack.streaming.mode` est la clé canonique du mode de flux Slack. `channels.slack.streaming.nativeTransport` contrôle le transport de flux natif de Slack. Les anciennes valeurs `streamMode`, booléennes `streaming` et `nativeStreaming` sont migrées automatiquement.
- Utilisez `user:<id>` (DM) ou `channel:<id>` pour les cibles de livraison.

**Modes de notification de réaction :** `off`, `own` (par défaut), `all`, `allowlist` (depuis `reactionAllowlist`).

**Isolation des sessions de fil :** `thread.historyScope` est par fil (par défaut) ou partagé sur le canal. `thread.inheritParent` copie la transcription du canal parent vers les nouveaux fils.

- Le flux natif Slack ainsi que le statut de fil « is typing... » de style assistant Slack exigent une cible de réponse dans un fil. Les DM de niveau supérieur restent hors fil par défaut, ils utilisent donc `typingReaction` ou la livraison normale au lieu de l’aperçu de style fil.
- `typingReaction` ajoute une réaction temporaire au message Slack entrant pendant le traitement d’une réponse, puis la retire à la fin. Utilisez un shortcode d’emoji Slack tel que `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals` : livraison des approbations d’exécution native à Slack et autorisation des approbateurs. Même schéma que Discord : `enabled` (`true`/`false`/`"auto"`), `approvers` (ID d’utilisateurs Slack), `agentFilter`, `sessionFilter` et `target` (`"dm"`, `"channel"` ou `"both"`).

| Groupe d’actions | Par défaut | Notes                  |
| ---------------- | ---------- | ---------------------- |
| reactions        | activé     | Réagir + lister les réactions |
| messages         | activé     | Lire/envoyer/modifier/supprimer |
| pins             | activé     | Épingler/désépingler/lister |
| memberInfo       | activé     | Informations sur les membres |
| emojiList        | activé     | Liste des emojis personnalisés |

### Mattermost

Mattermost est fourni comme Plugin : `openclaw plugins install @openclaw/mattermost`.

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
        // Optional explicit URL for reverse-proxy/public deployments
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Modes de chat : `oncall` (répond sur mention @, par défaut), `onmessage` (chaque message), `onchar` (messages commençant par un préfixe de déclenchement).

Lorsque les commandes natives Mattermost sont activées :

- `commands.callbackPath` doit être un chemin (par exemple `/api/channels/mattermost/command`), pas une URL complète.
- `commands.callbackUrl` doit pointer vers le point de terminaison Gateway OpenClaw et être accessible depuis le serveur Mattermost.
- Les rappels slash natifs sont authentifiés à l’aide des jetons par commande renvoyés
  par Mattermost lors de l’enregistrement des commandes slash. Si l’enregistrement échoue ou si aucune
  commande n’est activée, OpenClaw rejette les rappels avec
  `Unauthorized: invalid command token.`
- Pour les hôtes de rappel privés/tailnet/internes, Mattermost peut exiger que
  `ServiceSettings.AllowedUntrustedInternalConnections` inclue l’hôte/domaine de rappel.
  Utilisez des valeurs d’hôte/domaine, pas des URL complètes.
- `channels.mattermost.configWrites` : autoriser ou refuser les écritures de configuration initiées par Mattermost.
- `channels.mattermost.requireMention` : exiger une `@mention` avant de répondre dans les canaux.
- `channels.mattermost.groups.<channelId>.requireMention` : remplacement du filtrage par mention au niveau du canal (`"*"` pour la valeur par défaut).
- `channels.mattermost.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // liaison de compte facultative
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

**Modes de notification de réaction :** `off`, `own` (par défaut), `all`, `allowlist` (depuis `reactionAllowlist`).

- `channels.signal.account` : épingle le démarrage du canal à une identité de compte Signal spécifique.
- `channels.signal.configWrites` : autorise ou refuse les écritures de configuration initiées par Signal.
- `channels.signal.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.

### BlueBubbles

BlueBubbles est le chemin recommandé pour iMessage (pris en charge par un Plugin, configuré sous `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, group controls, and advanced actions:
      // see /channels/bluebubbles
    },
  },
}
```

- Chemins de clés principaux couverts ici : `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- `channels.bluebubbles.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` peuvent lier des conversations BlueBubbles à des sessions ACP persistantes. Utilisez un handle BlueBubbles ou une chaîne cible (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) dans `match.peer.id`. Sémantique partagée des champs : [Agents ACP](/fr/tools/acp-agents#channel-specific-settings).
- La configuration complète du canal BlueBubbles est documentée dans [BlueBubbles](/fr/channels/bluebubbles).

### iMessage

OpenClaw lance `imsg rpc` (JSON-RPC sur stdio). Aucun démon ni port n’est requis.

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

- `channels.imessage.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.

- Nécessite un accès complet au disque pour la base de données Messages.
- Préférez les cibles `chat_id:<id>`. Utilisez `imsg chats --limit 20` pour lister les discussions.
- `cliPath` peut pointer vers un wrapper SSH ; définissez `remoteHost` (`host` ou `user@host`) pour la récupération des pièces jointes via SCP.
- `attachmentRoots` et `remoteAttachmentRoots` restreignent les chemins des pièces jointes entrantes (par défaut : `/Users/*/Library/Messages/Attachments`).
- SCP utilise une vérification stricte de la clé d’hôte ; assurez-vous donc que la clé d’hôte du relais existe déjà dans `~/.ssh/known_hosts`.
- `channels.imessage.configWrites` : autorise ou refuse les écritures de configuration initiées par iMessage.
- Les entrées `bindings[]` de niveau supérieur avec `type: "acp"` peuvent lier des conversations iMessage à des sessions ACP persistantes. Utilisez un handle normalisé ou une cible de discussion explicite (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) dans `match.peer.id`. Sémantique partagée des champs : [Agents ACP](/fr/tools/acp-agents#channel-specific-settings).

<Accordion title="Exemple de wrapper SSH iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix est pris en charge par une extension et configuré sous `channels.matrix`.

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

- L’authentification par jeton utilise `accessToken` ; l’authentification par mot de passe utilise `userId` + `password`.
- `channels.matrix.proxy` achemine le trafic HTTP Matrix via un proxy HTTP(S) explicite. Les comptes nommés peuvent le remplacer avec `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` autorise les homeservers privés/internes. `proxy` et cette option réseau sont des contrôles indépendants.
- `channels.matrix.defaultAccount` sélectionne le compte préféré dans les configurations multi-comptes.
- `channels.matrix.autoJoin` vaut par défaut `off`, les salons invités et les nouvelles invitations de type DM sont donc ignorés tant que vous ne définissez pas `autoJoin: "allowlist"` avec `autoJoinAllowlist` ou `autoJoin: "always"`.
- `channels.matrix.execApprovals` : livraison des approbations d’exécution native à Matrix et autorisation des approbateurs.
  - `enabled` : `true`, `false` ou `"auto"` (par défaut). En mode auto, les approbations d’exécution s’activent lorsque des approbateurs peuvent être résolus à partir de `approvers` ou `commands.ownerAllowFrom`.
  - `approvers` : ID d’utilisateurs Matrix (par ex. `@owner:example.org`) autorisés à approuver les demandes d’exécution.
  - `agentFilter` : liste d’autorisation facultative des ID d’agents. Omettez-la pour transmettre les approbations pour tous les agents.
  - `sessionFilter` : motifs facultatifs de clé de session (sous-chaîne ou regex).
  - `target` : où envoyer les invites d’approbation. `"dm"` (par défaut), `"channel"` (salon d’origine) ou `"both"`.
  - Remplacements par compte : `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` contrôle la façon dont les DM Matrix sont groupés en sessions : `per-user` (par défaut) partage par pair acheminé, tandis que `per-room` isole chaque salon DM.
- Les sondes d’état Matrix et les recherches dans l’annuaire en direct utilisent la même politique de proxy que le trafic d’exécution.
- La configuration complète de Matrix, les règles de ciblage et les exemples de configuration sont documentés dans [Matrix](/fr/channels/matrix).

### Microsoft Teams

Microsoft Teams est pris en charge par une extension et configuré sous `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // see /channels/msteams
    },
  },
}
```

- Chemins de clés principaux couverts ici : `channels.msteams`, `channels.msteams.configWrites`.
- La configuration complète de Teams (identifiants, webhook, politique DM/groupe, remplacements par équipe/par canal) est documentée dans [Microsoft Teams](/fr/channels/msteams).

### IRC

IRC est pris en charge par une extension et configuré sous `channels.irc`.

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

- Chemins de clés principaux couverts ici : `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- `channels.irc.defaultAccount` facultatif remplace la sélection du compte par défaut lorsqu’il correspond à un ID de compte configuré.
- La configuration complète du canal IRC (hôte/port/TLS/canaux/listes d’autorisation/filtrage par mention) est documentée dans [IRC](/fr/channels/irc).

### Multi-compte (tous les canaux)

Exécutez plusieurs comptes par canal (chacun avec son propre `accountId`) :

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
- Les jetons d’environnement ne s’appliquent qu’au compte **default**.
- Les paramètres de base du canal s’appliquent à tous les comptes sauf remplacement par compte.
- Utilisez `bindings[].match.accountId` pour acheminer chaque compte vers un agent différent.
- Si vous ajoutez un compte non par défaut via `openclaw channels add` (ou l’intégration du canal) alors que vous utilisez encore une configuration de canal de niveau supérieur mono-compte, OpenClaw promeut d’abord les valeurs mono-compte de niveau supérieur circonscrites au compte vers la map de comptes du canal afin que le compte d’origine continue de fonctionner. La plupart des canaux les déplacent vers `channels.<channel>.accounts.default` ; Matrix peut à la place conserver une cible nommée/par défaut existante correspondante.
- Les liaisons existantes uniquement au niveau du canal (sans `accountId`) continuent à correspondre au compte par défaut ; les liaisons circonscrites au compte restent facultatives.
- `openclaw doctor --fix` répare également les formes mixtes en déplaçant les valeurs mono-compte de niveau supérieur circonscrites au compte vers le compte promu choisi pour ce canal. La plupart des canaux utilisent `accounts.default` ; Matrix peut à la place conserver une cible nommée/par défaut existante correspondante.

### Autres canaux d’extension

De nombreux canaux d’extension sont configurés en tant que `channels.<id>` et documentés dans leurs pages de canal dédiées (par exemple Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat et Twitch).
Voir l’index complet des canaux : [Canaux](/fr/channels).

### Filtrage par mention dans les discussions de groupe

Les messages de groupe exigent par défaut **une mention obligatoire** (mention dans les métadonnées ou motifs regex sûrs). S’applique aux discussions de groupe WhatsApp, Telegram, Discord, Google Chat et iMessage.

**Types de mention :**

- **Mentions dans les métadonnées** : mentions @ natives de la plateforme. Ignorées en mode auto-discussion WhatsApp.
- **Motifs de texte** : motifs regex sûrs dans `agents.list[].groupChat.mentionPatterns`. Les motifs invalides et les répétitions imbriquées non sûres sont ignorés.
- Le filtrage par mention n’est appliqué que lorsque la détection est possible (mentions natives ou au moins un motif).

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

#### Limites d’historique DM

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

Résolution : remplacement par DM → valeur par défaut du fournisseur → aucune limite (tout est conservé).

Pris en charge : `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Mode auto-discussion

Incluez votre propre numéro dans `allowFrom` pour activer le mode auto-discussion (ignore les mentions @ natives, répond uniquement aux motifs textuels) :

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
    native: "auto", // register native commands when supported
    nativeSkills: "auto", // register native skill commands when supported
    text: true, // parse /commands in chat messages
    bash: false, // allow ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // allow /config
    mcp: false, // allow /mcp
    plugins: false, // allow /plugins
    debug: false, // allow /debug
    restart: true, // allow /restart + gateway restart tool
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Détails des commandes">

- Ce bloc configure les surfaces de commande. Pour le catalogue actuel des commandes intégrées + bundle, voir [Commandes slash](/fr/tools/slash-commands).
- Cette page est une **référence des clés de configuration**, pas le catalogue complet des commandes. Les commandes propres à un canal/Plugin, comme QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, device-pair `/pair`, memory `/dreaming`, phone-control `/phone` et Talk `/voice`, sont documentées dans leurs pages de canal/Plugin ainsi que dans [Commandes slash](/fr/tools/slash-commands).
- Les commandes textuelles doivent être des messages **autonomes** commençant par `/`.
- `native: "auto"` active les commandes natives pour Discord/Telegram, et laisse Slack désactivé.
- `nativeSkills: "auto"` active les commandes natives de Skills pour Discord/Telegram, et laisse Slack désactivé.
- Remplacement par canal : `channels.discord.commands.native` (booléen ou `"auto"`). `false` efface les commandes précédemment enregistrées.
- Remplacez l’enregistrement natif des commandes de Skills par canal avec `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` ajoute des entrées supplémentaires au menu du bot Telegram.
- `bash: true` active `! <cmd>` pour le shell hôte. Nécessite `tools.elevated.enabled` et que l’expéditeur soit dans `tools.elevated.allowFrom.<channel>`.
- `config: true` active `/config` (lit/écrit `openclaw.json`). Pour les clients `chat.send` du Gateway, les écritures persistantes `/config set|unset` exigent également `operator.admin` ; `/config show` en lecture seule reste disponible pour les clients opérateur normaux avec portée d’écriture.
- `mcp: true` active `/mcp` pour la configuration du serveur MCP géré par OpenClaw sous `mcp.servers`.
- `plugins: true` active `/plugins` pour la découverte, l’installation et les contrôles d’activation/désactivation des plugins.
- `channels.<provider>.configWrites` contrôle les mutations de configuration par canal (par défaut : true).
- Pour les canaux multi-comptes, `channels.<provider>.accounts.<id>.configWrites` contrôle également les écritures qui ciblent ce compte (par exemple `/allowlist --config --account <id>` ou `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` désactive `/restart` et les actions de l’outil de redémarrage du Gateway. Par défaut : `true`.
- `ownerAllowFrom` est la liste d’autorisation explicite du propriétaire pour les commandes/outils réservés au propriétaire. Elle est distincte de `allowFrom`.
- `ownerDisplay: "hash"` hache les ID du propriétaire dans le prompt système. Définissez `ownerDisplaySecret` pour contrôler le hachage.
- `allowFrom` est défini par fournisseur. Lorsqu’il est défini, c’est l’**unique** source d’autorisation (les listes d’autorisation/appairage du canal et `useAccessGroups` sont ignorés).
- `useAccessGroups: false` permet aux commandes de contourner les politiques de groupes d’accès lorsque `allowFrom` n’est pas défini.
- Cartographie de la documentation des commandes :
  - catalogue intégré + bundle : [Commandes slash](/fr/tools/slash-commands)
  - surfaces de commande propres aux canaux : [Canaux](/fr/channels)
  - commandes QQ Bot : [QQ Bot](/fr/channels/qqbot)
  - commandes d’appairage : [Appairage](/fr/channels/pairing)
  - commande de carte LINE : [LINE](/fr/channels/line)
  - Dreaming de la mémoire : [Dreaming](/fr/concepts/dreaming)

</Accordion>

---

## Valeurs par défaut des agents

### `agents.defaults.workspace`

Par défaut : `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Racine de dépôt facultative affichée dans la ligne Runtime du prompt système. Si elle n’est pas définie, OpenClaw la détecte automatiquement en remontant depuis le workspace.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Liste d’autorisation de Skills par défaut facultative pour les agents qui ne définissent pas
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

- Omettez `agents.defaults.skills` pour autoriser tous les Skills par défaut.
- Omettez `agents.list[].skills` pour hériter des valeurs par défaut.
- Définissez `agents.list[].skills: []` pour n’avoir aucun Skills.
- Une liste non vide `agents.list[].skills` est l’ensemble final pour cet agent ; elle
  ne fusionne pas avec les valeurs par défaut.

### `agents.defaults.skipBootstrap`

Désactive la création automatique des fichiers bootstrap du workspace (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Contrôle quand les fichiers bootstrap du workspace sont injectés dans le prompt système. Par défaut : `"always"`.

- `"continuation-skip"` : les tours de continuation sûrs (après une réponse complète de l’assistant) évitent la réinjection du bootstrap du workspace, réduisant la taille du prompt. Les exécutions Heartbeat et les nouvelles tentatives post-Compaction reconstruisent toujours le contexte.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Nombre maximal de caractères par fichier bootstrap du workspace avant troncature. Par défaut : `12000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 12000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Nombre total maximal de caractères injectés dans l’ensemble des fichiers bootstrap du workspace. Par défaut : `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Contrôle le texte d’avertissement visible par l’agent lorsque le contexte bootstrap est tronqué.
Par défaut : `"once"`.

- `"off"` : n’injecte jamais de texte d’avertissement dans le prompt système.
- `"once"` : injecte l’avertissement une fois par signature unique de troncature (recommandé).
- `"always"` : injecte l’avertissement à chaque exécution lorsqu’une troncature existe.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### Cartographie de propriété du budget de contexte

OpenClaw possède plusieurs budgets de prompt/contexte à fort volume, et ils sont
délibérément répartis par sous-système au lieu de tous passer par un unique
réglage générique.

- `agents.defaults.bootstrapMaxChars` /
  `agents.defaults.bootstrapTotalMaxChars` :
  injection bootstrap normale du workspace.
- `agents.defaults.startupContext.*` :
  préambule de démarrage à usage unique pour `/new` et `/reset`, y compris les fichiers
  récents `memory/*.md` quotidiens.
- `skills.limits.*` :
  la liste compacte des Skills injectée dans le prompt système.
- `agents.defaults.contextLimits.*` :
  extraits d’exécution bornés et blocs injectés appartenant à l’exécution.
- `memory.qmd.limits.*` :
  extraits de recherche mémoire indexés et dimensionnement d’injection.

Utilisez le remplacement correspondant par agent uniquement lorsqu’un agent a besoin d’un
budget différent :

- `agents.list[].skillsLimits.maxSkillsPromptChars`
- `agents.list[].contextLimits.*`

#### `agents.defaults.startupContext`

Contrôle le préambule de démarrage du premier tour injecté lors des exécutions
`/new` et `/reset` sans autre contenu.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

Valeurs par défaut partagées pour les surfaces de contexte d’exécution bornées.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        memoryGetDefaultLines: 120,
        toolResultMaxChars: 16000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars` : plafond d’extrait `memory_get` par défaut avant l’ajout des
  métadonnées de troncature et de l’avis de continuation.
- `memoryGetDefaultLines` : fenêtre de lignes `memory_get` par défaut lorsque `lines` est
  omis.
- `toolResultMaxChars` : plafond des résultats d’outil en direct utilisé pour les résultats persistés et la
  récupération de débordement.
- `postCompactionMaxChars` : plafond d’extrait de `AGENTS.md` utilisé lors de l’injection de
  rafraîchissement post-Compaction.

#### `agents.list[].contextLimits`

Remplacement par agent pour les réglages partagés `contextLimits`. Les champs omis héritent
de `agents.defaults.contextLimits`.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        toolResultMaxChars: 16000,
      },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
          toolResultMaxChars: 8000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

Plafond global pour la liste compacte des Skills injectée dans le prompt système. Cela
n’affecte pas la lecture à la demande des fichiers `SKILL.md`.

```json5
{
  skills: {
    limits: {
      maxSkillsPromptChars: 18000,
    },
  },
}
```

#### `agents.list[].skillsLimits.maxSkillsPromptChars`

Remplacement par agent pour le budget du prompt de Skills.

```json5
{
  agents: {
    list: [
      {
        id: "tiny-local",
        skillsLimits: {
          maxSkillsPromptChars: 6000,
        },
      },
    ],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

Taille maximale en pixels du plus long côté d’une image dans les blocs image de transcription/outil avant les appels fournisseur.
Par défaut : `1200`.

Des valeurs plus faibles réduisent généralement l’usage des jetons de vision et la taille des charges utiles pour les exécutions riches en captures d’écran.
Des valeurs plus élevées préservent davantage de détails visuels.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Fuseau horaire pour le contexte du prompt système (pas les horodatages des messages). Revient au fuseau horaire de l’hôte.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Format de l’heure dans le prompt système. Par défaut : `auto` (préférence de l’OS).

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
      params: { cacheRetention: "long" }, // paramètres fournisseur globaux par défaut
      embeddedHarness: {
        runtime: "auto", // auto | pi | registered harness id, e.g. codex
        fallback: "pi", // pi | none
      },
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

- `model` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - La forme chaîne définit uniquement le modèle principal.
  - La forme objet définit le principal plus des modèles de bascule ordonnés.
- `imageModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par le chemin de l’outil `image` comme configuration de modèle de vision.
  - Également utilisé comme routage de repli lorsque le modèle sélectionné/par défaut ne peut pas accepter d’entrée image.
- `imageGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération d’images et toute future surface d’outil/Plugin qui génère des images.
  - Valeurs typiques : `google/gemini-3.1-flash-image-preview` pour la génération d’images Gemini native, `fal/fal-ai/flux/dev` pour fal, ou `openai/gpt-image-1` pour OpenAI Images.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l’authentification/la clé API du fournisseur correspondant (par exemple `GEMINI_API_KEY` ou `GOOGLE_API_KEY` pour `google/*`, `OPENAI_API_KEY` pour `openai/*`, `FAL_KEY` pour `fal/*`).
  - Si omis, `image_generate` peut quand même déduire une valeur par défaut de fournisseur avec authentification. Il essaie d’abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération d’images enregistrés dans l’ordre des ID de fournisseur.
- `musicGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération musicale et l’outil intégré `music_generate`.
  - Valeurs typiques : `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` ou `minimax/music-2.5+`.
  - Si omis, `music_generate` peut quand même déduire une valeur par défaut de fournisseur avec authentification. Il essaie d’abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération musicale enregistrés dans l’ordre des ID de fournisseur.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l’authentification/la clé API du fournisseur correspondant.
- `videoGenerationModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par la capacité partagée de génération vidéo et l’outil intégré `video_generate`.
  - Valeurs typiques : `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` ou `qwen/wan2.7-r2v`.
  - Si omis, `video_generate` peut quand même déduire une valeur par défaut de fournisseur avec authentification. Il essaie d’abord le fournisseur par défaut actuel, puis les autres fournisseurs de génération vidéo enregistrés dans l’ordre des ID de fournisseur.
  - Si vous sélectionnez directement un fournisseur/modèle, configurez aussi l’authentification/la clé API du fournisseur correspondant.
  - Le fournisseur groupé de génération vidéo Qwen prend en charge jusqu’à 1 vidéo de sortie, 1 image d’entrée, 4 vidéos d’entrée, une durée de 10 secondes et les options de niveau fournisseur `size`, `aspectRatio`, `resolution`, `audio` et `watermark`.
- `pdfModel` : accepte soit une chaîne (`"provider/model"`), soit un objet (`{ primary, fallbacks }`).
  - Utilisé par l’outil `pdf` pour le routage du modèle.
  - Si omis, l’outil PDF revient à `imageModel`, puis au modèle de session/par défaut résolu.
- `pdfMaxBytesMb` : limite de taille PDF par défaut pour l’outil `pdf` lorsque `maxBytesMb` n’est pas transmis à l’appel.
- `pdfMaxPages` : nombre maximal de pages pris en compte par défaut par le mode de repli d’extraction dans l’outil `pdf`.
- `verboseDefault` : niveau détaillé par défaut pour les agents. Valeurs : `"off"`, `"on"`, `"full"`. Par défaut : `"off"`.
- `elevatedDefault` : niveau de sortie élevée par défaut pour les agents. Valeurs : `"off"`, `"on"`, `"ask"`, `"full"`. Par défaut : `"on"`.
- `model.primary` : format `provider/model` (par ex. `openai/gpt-5.4`). Si vous omettez le fournisseur, OpenClaw essaie d’abord un alias, puis une correspondance unique de fournisseur configuré pour cet ID de modèle exact, et seulement ensuite revient au fournisseur par défaut configuré (comportement de compatibilité déprécié, donc préférez l’explicite `provider/model`). Si ce fournisseur n’expose plus le modèle par défaut configuré, OpenClaw revient au premier fournisseur/modèle configuré au lieu d’exposer un ancien défaut d’un fournisseur supprimé.
- `models` : le catalogue de modèles configuré et la liste d’autorisation pour `/model`. Chaque entrée peut inclure `alias` (raccourci) et `params` (spécifiques au fournisseur, par exemple `temperature`, `maxTokens`, `cacheRetention`, `context1m`).
- `params` : paramètres globaux de fournisseur par défaut appliqués à tous les modèles. À définir dans `agents.defaults.params` (par ex. `{ cacheRetention: "long" }`).
- Priorité de fusion de `params` (configuration) : `agents.defaults.params` (base globale) est remplacé par `agents.defaults.models["provider/model"].params` (par modèle), puis `agents.list[].params` (ID d’agent correspondant) remplace par clé. Voir [Prompt Caching](/fr/reference/prompt-caching) pour les détails.
- `embeddedHarness` : politique d’exécution embarquée bas niveau par défaut pour les agents. Utilisez `runtime: "auto"` pour laisser les harness de Plugin enregistrés revendiquer les modèles pris en charge, `runtime: "pi"` pour forcer le harness PI intégré, ou un ID de harness enregistré tel que `runtime: "codex"`. Définissez `fallback: "none"` pour désactiver le repli automatique vers PI.
- Les outils d’écriture de configuration qui modifient ces champs (par exemple `/models set`, `/models set-image` et les commandes d’ajout/suppression de repli) enregistrent la forme objet canonique et préservent les listes de repli existantes lorsque possible.
- `maxConcurrent` : nombre maximal d’exécutions parallèles d’agents entre les sessions (chaque session reste sérialisée). Par défaut : 4.

### `agents.defaults.embeddedHarness`

`embeddedHarness` contrôle quel exécuteur bas niveau traite les tours d’agent embarqués.
La plupart des déploiements devraient conserver la valeur par défaut `{ runtime: "auto", fallback: "pi" }`.
Utilisez-le lorsqu’un Plugin de confiance fournit un harness natif, comme le harness
groupé du serveur d’application Codex.

```json5
{
  agents: {
    defaults: {
      model: "codex/gpt-5.4",
      embeddedHarness: {
        runtime: "codex",
        fallback: "none",
      },
    },
  },
}
```

- `runtime` : `"auto"`, `"pi"` ou un ID de harness de Plugin enregistré. Le Plugin Codex groupé enregistre `codex`.
- `fallback` : `"pi"` ou `"none"`. `"pi"` conserve le harness PI intégré comme repli de compatibilité. `"none"` fait échouer une sélection de harness de Plugin absente ou non prise en charge au lieu d’utiliser silencieusement PI.
- Remplacements d’environnement : `OPENCLAW_AGENT_RUNTIME=<id|auto|pi>` remplace `runtime` ; `OPENCLAW_AGENT_HARNESS_FALLBACK=none` désactive le repli PI pour ce processus.
- Pour les déploiements Codex uniquement, définissez `model: "codex/gpt-5.4"`, `embeddedHarness.runtime: "codex"` et `embeddedHarness.fallback: "none"`.
- Cela ne contrôle que le harness de chat embarqué. La génération de média, la vision, les PDF, la musique, la vidéo et le TTS utilisent toujours leurs paramètres fournisseur/modèle.

**Raccourcis d’alias intégrés** (s’appliquent uniquement lorsque le modèle figure dans `agents.defaults.models`) :

| Alias               | Modèle                                |
| ------------------- | ------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`           |
| `sonnet`            | `anthropic/claude-sonnet-4-6`         |
| `gpt`               | `openai/gpt-5.4`                      |
| `gpt-mini`          | `openai/gpt-5.4-mini`                 |
| `gpt-nano`          | `openai/gpt-5.4-nano`                 |
| `gemini`            | `google/gemini-3.1-pro-preview`       |
| `gemini-flash`      | `google/gemini-3-flash-preview`       |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Vos alias configurés ont toujours priorité sur les valeurs par défaut.

Les modèles Z.AI GLM-4.x activent automatiquement le mode thinking sauf si vous définissez `--thinking off` ou si vous définissez vous-même `agents.defaults.models["zai/<model>"].params.thinking`.
Les modèles Z.AI activent `tool_stream` par défaut pour le streaming des appels d’outils. Définissez `agents.defaults.models["zai/<model>"].params.tool_stream` sur `false` pour le désactiver.
Les modèles Anthropic Claude 4.6 utilisent par défaut le mode thinking `adaptive` lorsqu’aucun niveau de thinking explicite n’est défini.

### `agents.defaults.cliBackends`

Backends CLI facultatifs pour les exécutions de repli en texte seul (sans appels d’outil). Utile comme solution de secours lorsque les fournisseurs d’API échouent.

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

- Les backends CLI sont orientés texte ; les outils sont toujours désactivés.
- Les sessions sont prises en charge lorsque `sessionArg` est défini.
- Le passage d’images est pris en charge lorsque `imageArg` accepte des chemins de fichiers.

### `agents.defaults.systemPromptOverride`

Remplace l’intégralité du prompt système assemblé par OpenClaw par une chaîne fixe. À définir au niveau par défaut (`agents.defaults.systemPromptOverride`) ou par agent (`agents.list[].systemPromptOverride`). Les valeurs par agent ont priorité ; une valeur vide ou composée uniquement d’espaces est ignorée. Utile pour des expériences de prompt contrôlées.

```json5
{
  agents: {
    defaults: {
      systemPromptOverride: "You are a helpful assistant.",
    },
  },
}
```

### `agents.defaults.heartbeat`

Exécutions Heartbeat périodiques.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
        lightContext: false, // default: false; true keeps only HEARTBEAT.md from workspace bootstrap files
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every` : chaîne de durée (ms/s/m/h). Par défaut : `30m` (authentification par clé API) ou `1h` (authentification OAuth). Définissez `0m` pour désactiver.
- `includeSystemPromptSection` : lorsqu’il vaut false, omet la section Heartbeat du prompt système et ignore l’injection de `HEARTBEAT.md` dans le contexte bootstrap. Par défaut : `true`.
- `suppressToolErrorWarnings` : lorsqu’il vaut true, supprime les charges utiles d’avertissement d’erreur d’outil pendant les exécutions Heartbeat.
- `timeoutSeconds` : durée maximale en secondes autorisée pour un tour d’agent Heartbeat avant interruption. Laissez non défini pour utiliser `agents.defaults.timeoutSeconds`.
- `directPolicy` : politique de livraison directe/DM. `allow` (par défaut) autorise la livraison à une cible directe. `block` supprime la livraison à une cible directe et émet `reason=dm-blocked`.
- `lightContext` : lorsqu’il vaut true, les exécutions Heartbeat utilisent un contexte bootstrap allégé et ne conservent que `HEARTBEAT.md` parmi les fichiers bootstrap du workspace.
- `isolatedSession` : lorsqu’il vaut true, chaque Heartbeat s’exécute dans une nouvelle session sans historique de conversation préalable. Même schéma d’isolation que le Cron `sessionTarget: "isolated"`. Réduit le coût en jetons par Heartbeat d’environ ~100K à ~2-5K jetons.
- Par agent : définissez `agents.list[].heartbeat`. Lorsqu’un agent quelconque définit `heartbeat`, **seuls ces agents** exécutent des Heartbeat.
- Les Heartbeat exécutent des tours d’agent complets — des intervalles plus courts consomment plus de jetons.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
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

- `mode` : `default` ou `safeguard` (résumé par blocs pour les longs historiques). Voir [Compaction](/fr/concepts/compaction).
- `provider` : ID d’un Plugin fournisseur de Compaction enregistré. Lorsqu’il est défini, le `summarize()` du fournisseur est appelé à la place du résumé LLM intégré. Revient à l’implémentation intégrée en cas d’échec. Définir un fournisseur force `mode: "safeguard"`. Voir [Compaction](/fr/concepts/compaction).
- `timeoutSeconds` : nombre maximal de secondes autorisé pour une opération unique de Compaction avant qu’OpenClaw ne l’interrompe. Par défaut : `900`.
- `identifierPolicy` : `strict` (par défaut), `off` ou `custom`. `strict` préfixe des consignes intégrées de conservation des identifiants opaques pendant le résumé de Compaction.
- `identifierInstructions` : texte personnalisé facultatif de conservation des identifiants utilisé lorsque `identifierPolicy=custom`.
- `postCompactionSections` : noms facultatifs de sections H2/H3 de `AGENTS.md` à réinjecter après Compaction. Par défaut : `["Session Startup", "Red Lines"]` ; définissez `[]` pour désactiver la réinjection. Lorsqu’il n’est pas défini ou qu’il est explicitement défini sur cette paire par défaut, les anciens en-têtes `Every Session`/`Safety` sont aussi acceptés comme repli hérité.
- `model` : remplacement facultatif `provider/model-id` pour le résumé de Compaction uniquement. Utilisez-le lorsque la session principale doit conserver un modèle mais que les résumés de Compaction doivent être exécutés sur un autre ; lorsqu’il n’est pas défini, la Compaction utilise le modèle principal de la session.
- `notifyUser` : lorsqu’il vaut `true`, envoie un bref avis à l’utilisateur au démarrage de la Compaction (par exemple, « Compactage du contexte... »). Désactivé par défaut afin que la Compaction reste silencieuse.
- `memoryFlush` : tour agentique silencieux avant la Compaction automatique pour stocker les mémoires durables. Ignoré lorsque le workspace est en lecture seule.

### `agents.defaults.contextPruning`

Émonde les **anciens résultats d’outils** du contexte en mémoire avant l’envoi au LLM. **Ne modifie pas** l’historique de session sur le disque.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // duration (ms/s/m/h), default unit: minutes
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

- `mode: "cache-ttl"` active les passes d’émondage.
- `ttl` contrôle à quelle fréquence l’émondage peut être réexécuté (après le dernier accès au cache).
- L’émondage tronque d’abord partiellement les résultats d’outils surdimensionnés, puis efface complètement les anciens résultats d’outils si nécessaire.

**Le tronquage partiel** conserve le début + la fin et insère `...` au milieu.

**L’effacement complet** remplace l’intégralité du résultat d’outil par le texte de remplacement.

Remarques :

- Les blocs d’image ne sont jamais tronqués/effacés.
- Les ratios sont basés sur les caractères (approximatifs), pas sur des comptes exacts de jetons.
- S’il existe moins de `keepLastAssistants` messages d’assistant, l’émondage est ignoré.

</Accordion>

Voir [Émondage de session](/fr/concepts/session-pruning) pour les détails de comportement.

### Streaming par blocs

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (use minMs/maxMs)
    },
  },
}
```

- Les canaux autres que Telegram exigent `*.blockStreaming: true` explicite pour activer les réponses par blocs.
- Remplacements par canal : `channels.<channel>.blockStreamingCoalesce` (et variantes par compte). Signal/Slack/Discord/Google Chat utilisent par défaut `minChars: 1500`.
- `humanDelay` : pause aléatoire entre les réponses par blocs. `natural` = 800–2500ms. Remplacement par agent : `agents.list[].humanDelay`.

Voir [Streaming](/fr/concepts/streaming) pour le comportement et les détails de segmentation.

### Indicateurs de frappe

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

- Valeurs par défaut : `instant` pour les discussions directes/mentions, `message` pour les discussions de groupe sans mention.
- Remplacements par session : `session.typingMode`, `session.typingIntervalSeconds`.

Voir [Indicateurs de frappe](/fr/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Sandboxing facultatif pour l’agent embarqué. Voir [Sandboxing](/fr/gateway/sandboxing) pour le guide complet.

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
          // SecretRefs / inline contents also supported:
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

<Accordion title="Détails du bac à sable">

**Backend :**

- `docker` : environnement d’exécution Docker local (par défaut)
- `ssh` : environnement d’exécution distant générique adossé à SSH
- `openshell` : environnement d’exécution OpenShell

Lorsque `backend: "openshell"` est sélectionné, les paramètres spécifiques à l’environnement d’exécution sont déplacés vers
`plugins.entries.openshell.config`.

**Configuration du backend SSH :**

- `target` : cible SSH au format `user@host[:port]`
- `command` : commande client SSH (par défaut : `ssh`)
- `workspaceRoot` : racine distante absolue utilisée pour les workspaces par portée
- `identityFile` / `certificateFile` / `knownHostsFile` : fichiers locaux existants transmis à OpenSSH
- `identityData` / `certificateData` / `knownHostsData` : contenus intégrés ou SecretRefs qu’OpenClaw matérialise dans des fichiers temporaires à l’exécution
- `strictHostKeyChecking` / `updateHostKeys` : réglages de politique de clé d’hôte OpenSSH

**Priorité de l’authentification SSH :**

- `identityData` a priorité sur `identityFile`
- `certificateData` a priorité sur `certificateFile`
- `knownHostsData` a priorité sur `knownHostsFile`
- Les valeurs `*Data` adossées à SecretRef sont résolues à partir de l’instantané actif du runtime des secrets avant le démarrage de la session sandbox

**Comportement du backend SSH :**

- initialise le workspace distant une fois après création ou recréation
- puis conserve le workspace SSH distant comme canonique
- achemine `exec`, les outils de fichiers et les chemins média via SSH
- ne resynchronise pas automatiquement les modifications distantes vers l’hôte
- ne prend pas en charge les conteneurs de navigateur sandbox

**Accès au workspace :**

- `none` : workspace sandbox par portée sous `~/.openclaw/sandboxes`
- `ro` : workspace sandbox à `/workspace`, workspace agent monté en lecture seule sur `/agent`
- `rw` : workspace agent monté en lecture/écriture sur `/workspace`

**Portée :**

- `session` : conteneur + workspace par session
- `agent` : un conteneur + workspace par agent (par défaut)
- `shared` : conteneur et workspace partagés (aucune isolation inter-sessions)

**Configuration du Plugin OpenShell :**

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
          gateway: "lab", // optional
          gatewayEndpoint: "https://lab.example", // optional
          policy: "strict", // optional OpenShell policy id
          providers: ["openai"], // optional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**Mode OpenShell :**

- `mirror` : initialise le distant à partir du local avant `exec`, resynchronise vers le local après `exec` ; le workspace local reste canonique
- `remote` : initialise le distant une fois lors de la création du sandbox, puis conserve le workspace distant comme canonique

En mode `remote`, les modifications locales sur l’hôte effectuées en dehors d’OpenClaw ne sont pas automatiquement synchronisées dans le sandbox après l’étape d’initialisation.
Le transport s’effectue en SSH dans le sandbox OpenShell, mais le Plugin gère le cycle de vie du sandbox et la synchronisation miroir facultative.

**`setupCommand`** s’exécute une fois après la création du conteneur (via `sh -lc`). Nécessite une sortie réseau, une racine accessible en écriture et l’utilisateur root.

**Les conteneurs utilisent par défaut `network: "none"`** — définissez `"bridge"` (ou un réseau bridge personnalisé) si l’agent a besoin d’un accès sortant.
`"host"` est bloqué. `"container:<id>"` est bloqué par défaut sauf si vous définissez explicitement
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (mode « break-glass »).

**Les pièces jointes entrantes** sont placées dans `media/inbound/*` dans le workspace actif.

**`docker.binds`** monte des répertoires hôte supplémentaires ; les montages globaux et par agent sont fusionnés.

**Navigateur sandboxé** (`sandbox.browser.enabled`) : Chromium + CDP dans un conteneur. L’URL noVNC est injectée dans le prompt système. Ne nécessite pas `browser.enabled` dans `openclaw.json`.
L’accès observateur noVNC utilise l’authentification VNC par défaut et OpenClaw émet une URL à jeton de courte durée (au lieu d’exposer le mot de passe dans l’URL partagée).

- `allowHostControl: false` (par défaut) empêche les sessions sandboxées de cibler le navigateur de l’hôte.
- `network` vaut par défaut `openclaw-sandbox-browser` (réseau bridge dédié). Définissez-le sur `bridge` uniquement si vous voulez explicitement une connectivité bridge globale.
- `cdpSourceRange` restreint éventuellement l’entrée CDP à la périphérie du conteneur à une plage CIDR (par exemple `172.21.0.1/32`).
- `sandbox.browser.binds` monte des répertoires hôte supplémentaires uniquement dans le conteneur du navigateur sandboxé. Lorsqu’il est défini (y compris `[]`), il remplace `docker.binds` pour le conteneur du navigateur.
- Les valeurs par défaut de lancement sont définies dans `scripts/sandbox-browser-entrypoint.sh` et réglées pour les hôtes de conteneurs :
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
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` si un usage WebGL/3D l’exige.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` réactive les extensions si votre flux de travail
    en dépend.
  - `--renderer-process-limit=2` peut être modifié avec
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` ; définissez `0` pour utiliser la
    limite de processus par défaut de Chromium.
  - plus `--no-sandbox` et `--disable-setuid-sandbox` lorsque `noSandbox` est activé.
  - Les valeurs par défaut correspondent à la base de l’image du conteneur ; utilisez une image de navigateur personnalisée avec un point d’entrée personnalisé pour modifier les valeurs par défaut du conteneur.

</Accordion>

Le sandboxing du navigateur et `sandbox.docker.binds` sont disponibles uniquement avec Docker.

Construire les images :

```bash
scripts/sandbox-setup.sh           # image sandbox principale
scripts/sandbox-browser-setup.sh   # image de navigateur facultative
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
        model: "anthropic/claude-opus-4-6", // or { primary, fallbacks }
        thinkingDefault: "high", // per-agent thinking level override
        reasoningDefault: "on", // per-agent reasoning visibility override
        fastModeDefault: false, // per-agent fast mode override
        embeddedHarness: { runtime: "auto", fallback: "pi" },
        params: { cacheRetention: "none" }, // overrides matching defaults.models params by key
        skills: ["docs-search"], // replaces agents.defaults.skills when set
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

- `id` : ID d’agent stable (obligatoire).
- `default` : lorsque plusieurs sont définis, le premier l’emporte (avertissement journalisé). Si aucun n’est défini, la première entrée de la liste est la valeur par défaut.
- `model` : la forme chaîne remplace uniquement `primary` ; la forme objet `{ primary, fallbacks }` remplace les deux (`[]` désactive les replis globaux). Les tâches Cron qui ne remplacent que `primary` héritent toujours des replis par défaut sauf si vous définissez `fallbacks: []`.
- `params` : paramètres de flux par agent fusionnés sur l’entrée de modèle sélectionnée dans `agents.defaults.models`. Utilisez cela pour des remplacements spécifiques à l’agent comme `cacheRetention`, `temperature` ou `maxTokens` sans dupliquer tout le catalogue de modèles.
- `skills` : liste d’autorisation de Skills facultative par agent. Si elle est omise, l’agent hérite de `agents.defaults.skills` lorsqu’elle est définie ; une liste explicite remplace les valeurs par défaut au lieu de les fusionner, et `[]` signifie aucun Skills.
- `thinkingDefault` : niveau de thinking par défaut facultatif par agent (`off | minimal | low | medium | high | xhigh | adaptive`). Remplace `agents.defaults.thinkingDefault` pour cet agent lorsqu’aucun remplacement par message ou session n’est défini.
- `reasoningDefault` : visibilité de reasoning par défaut facultative par agent (`on | off | stream`). S’applique lorsqu’aucun remplacement de reasoning par message ou session n’est défini.
- `fastModeDefault` : valeur par défaut facultative par agent pour le mode rapide (`true | false`). S’applique lorsqu’aucun remplacement de mode rapide par message ou session n’est défini.
- `embeddedHarness` : remplacement facultatif par agent de la politique de harness bas niveau. Utilisez `{ runtime: "codex", fallback: "none" }` pour rendre un agent réservé à Codex tandis que les autres agents conservent le repli PI par défaut.
- `runtime` : descripteur d’exécution facultatif par agent. Utilisez `type: "acp"` avec les valeurs par défaut `runtime.acp` (`agent`, `backend`, `mode`, `cwd`) lorsque l’agent doit utiliser par défaut des sessions harness ACP.
- `identity.avatar` : chemin relatif au workspace, URL `http(s)` ou URI `data:`.
- `identity` dérive des valeurs par défaut : `ackReaction` depuis `emoji`, `mentionPatterns` depuis `name`/`emoji`.
- `subagents.allowAgents` : liste d’autorisation des ID d’agents pour `sessions_spawn` (`["*"]` = n’importe lequel ; par défaut : même agent uniquement).
- Garde d’héritage du sandbox : si la session demandeuse est sandboxée, `sessions_spawn` rejette les cibles qui s’exécuteraient hors sandbox.
- `subagents.requireAgentId` : lorsqu’il vaut true, bloque les appels `sessions_spawn` qui omettent `agentId` (force une sélection explicite de profil ; par défaut : false).

---

## Routage multi-agent

Exécute plusieurs agents isolés dans un seul Gateway. Voir [Multi-Agent](/fr/concepts/multi-agent).

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

### Champs `match` des liaisons

- `type` (facultatif) : `route` pour le routage normal (type manquant = route par défaut), `acp` pour les liaisons ACP persistantes de conversation.
- `match.channel` (obligatoire)
- `match.accountId` (facultatif ; `*` = n’importe quel compte ; omis = compte par défaut)
- `match.peer` (facultatif ; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (facultatif ; spécifique au canal)
- `acp` (facultatif ; uniquement pour `type: "acp"`) : `{ mode, label, cwd, backend }`

**Ordre de correspondance déterministe :**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exact, sans pair/guilde/équipe)
5. `match.accountId: "*"` (à l’échelle du canal)
6. Agent par défaut

À l’intérieur de chaque niveau, la première entrée `bindings` correspondante l’emporte.

Pour les entrées `type: "acp"`, OpenClaw résout par identité exacte de conversation (`match.channel` + compte + `match.peer.id`) et n’utilise pas l’ordre par niveaux des liaisons de routage ci-dessus.

### Profils d’accès par agent

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

<Accordion title="Outils + workspace en lecture seule">

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

Voir [Sandbox & outils multi-agent](/fr/tools/multi-agent-sandbox-tools) pour les détails de priorité.

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
    parentForkMaxTokens: 100000, // skip parent-thread fork above this token count (0 disables)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    mainKey: "main", // legacy (runtime always uses "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Détails des champs de session">

- **`scope`** : stratégie de base de regroupement des sessions pour les contextes de discussion de groupe.
  - `per-sender` (par défaut) : chaque expéditeur obtient une session isolée dans un contexte de canal.
  - `global` : tous les participants d’un contexte de canal partagent une seule session (à utiliser uniquement lorsqu’un contexte partagé est voulu).
- **`dmScope`** : façon dont les DM sont regroupés.
  - `main` : tous les DM partagent la session principale.
  - `per-peer` : isole par ID d’expéditeur entre les canaux.
  - `per-channel-peer` : isole par canal + expéditeur (recommandé pour les boîtes de réception multi-utilisateurs).
  - `per-account-channel-peer` : isole par compte + canal + expéditeur (recommandé pour le multi-compte).
- **`identityLinks`** : mappe les ID canoniques vers des pairs préfixés par fournisseur pour le partage de session inter-canaux.
- **`reset`** : politique de réinitialisation principale. `daily` réinitialise à `atHour` selon l’heure locale ; `idle` réinitialise après `idleMinutes`. Lorsque les deux sont configurés, la première échéance l’emporte.
- **`resetByType`** : remplacements par type (`direct`, `group`, `thread`). L’ancien `dm` est accepté comme alias de `direct`.
- **`parentForkMaxTokens`** : `totalTokens` maximal autorisé pour la session parente lors de la création d’une session de fil dérivée (par défaut `100000`).
  - Si `totalTokens` de la parente dépasse cette valeur, OpenClaw démarre une nouvelle session de fil au lieu d’hériter de l’historique de transcription de la parente.
  - Définissez `0` pour désactiver cette protection et toujours autoriser la dérivation depuis la parente.
- **`mainKey`** : champ hérité. L’exécution utilise toujours `"main"` pour le compartiment principal des discussions directes.
- **`agentToAgent.maxPingPongTurns`** : nombre maximal de tours de réponse aller-retour entre agents pendant les échanges agent-à-agent (entier, plage : `0`–`5`). `0` désactive l’enchaînement ping-pong.
- **`sendPolicy`** : correspondance par `channel`, `chatType` (`direct|group|channel`, avec l’alias hérité `dm`), `keyPrefix` ou `rawKeyPrefix`. Le premier refus l’emporte.
- **`maintenance`** : contrôles de nettoyage + rétention du stockage de session.
  - `mode` : `warn` émet uniquement des avertissements ; `enforce` applique le nettoyage.
  - `pruneAfter` : seuil d’ancienneté pour les entrées obsolètes (par défaut `30d`).
  - `maxEntries` : nombre maximal d’entrées dans `sessions.json` (par défaut `500`).
  - `rotateBytes` : fait pivoter `sessions.json` lorsqu’il dépasse cette taille (par défaut `10mb`).
  - `resetArchiveRetention` : rétention pour les archives de transcription `*.reset.<timestamp>`. Par défaut, hérite de `pruneAfter` ; définissez `false` pour désactiver.
  - `maxDiskBytes` : budget disque facultatif pour le répertoire des sessions. En mode `warn`, journalise des avertissements ; en mode `enforce`, supprime d’abord les artefacts/sessions les plus anciens.
  - `highWaterBytes` : cible facultative après nettoyage du budget. Par défaut : `80%` de `maxDiskBytes`.
- **`threadBindings`** : valeurs globales par défaut pour les fonctionnalités de session liées aux fils.
  - `enabled` : interrupteur maître par défaut (les fournisseurs peuvent le remplacer ; Discord utilise `channels.discord.threadBindings.enabled`)
  - `idleHours` : perte de focus automatique par défaut après inactivité en heures (`0` désactive ; les fournisseurs peuvent remplacer)
  - `maxAgeHours` : âge maximal strict par défaut en heures (`0` désactive ; les fournisseurs peuvent remplacer)

</Accordion>

---

## Messages

```json5
{
  messages: {
    responsePrefix: "🦞", // or "auto"
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
      debounceMs: 2000, // 0 disables
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Préfixe de réponse

Remplacements par canal/compte : `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Résolution (le plus spécifique l’emporte) : compte → canal → global. `""` désactive et arrête la cascade. `"auto"` dérive `[{identity.name}]`.

**Variables de modèle :**

| Variable          | Description               | Exemple                     |
| ----------------- | ------------------------- | --------------------------- |
| `{model}`         | Nom court du modèle       | `claude-opus-4-6`           |
| `{modelFull}`     | Identifiant complet du modèle | `anthropic/claude-opus-4-6` |
| `{provider}`      | Nom du fournisseur        | `anthropic`                 |
| `{thinkingLevel}` | Niveau de thinking actuel | `high`, `low`, `off`        |
| `{identity.name}` | Nom d’identité de l’agent | (identique à `"auto"`)      |

Les variables sont insensibles à la casse. `{think}` est un alias de `{thinkingLevel}`.

### Réaction d’accusé

- Par défaut, utilise `identity.emoji` de l’agent actif, sinon `"👀"`. Définissez `""` pour désactiver.
- Remplacements par canal : `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Ordre de résolution : compte → canal → `messages.ackReaction` → repli d’identité.
- Portée : `group-mentions` (par défaut), `group-all`, `direct`, `all`.
- `removeAckAfterReply` supprime l’accusé après la réponse sur Slack, Discord et Telegram.
- `messages.statusReactions.enabled` active les réactions d’état du cycle de vie sur Slack, Discord et Telegram.
  Sur Slack et Discord, non défini conserve les réactions d’état actives lorsque les réactions d’accusé sont actives.
  Sur Telegram, définissez-le explicitement sur `true` pour activer les réactions d’état du cycle de vie.

### Antirebond entrant

Regroupe les messages texte rapides d’un même expéditeur en un seul tour d’agent. Les médias/pièces jointes déclenchent un vidage immédiat. Les commandes de contrôle contournent l’antirebond.

### TTS (synthèse vocale)

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

- `auto` contrôle le mode d’auto-TTS par défaut : `off`, `always`, `inbound` ou `tagged`. `/tts on|off` peut remplacer les préférences locales, et `/tts status` affiche l’état effectif.
- `summaryModel` remplace `agents.defaults.model.primary` pour l’auto-résumé.
- `modelOverrides` est activé par défaut ; `modelOverrides.allowProvider` vaut par défaut `false` (activation explicite requise).
- Les clés API reviennent à `ELEVENLABS_API_KEY`/`XI_API_KEY` et `OPENAI_API_KEY`.
- `openai.baseUrl` remplace le point de terminaison OpenAI TTS. L’ordre de résolution est : configuration, puis `OPENAI_TTS_BASE_URL`, puis `https://api.openai.com/v1`.
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

- `talk.provider` doit correspondre à une clé de `talk.providers` lorsque plusieurs fournisseurs Talk sont configurés.
- Les anciennes clés Talk plates (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) sont compatibles uniquement et sont migrées automatiquement vers `talk.providers.<provider>`.
- Les ID de voix reviennent à `ELEVENLABS_VOICE_ID` ou `SAG_VOICE_ID`.
- `providers.*.apiKey` accepte des chaînes en clair ou des objets SecretRef.
- Le repli `ELEVENLABS_API_KEY` ne s’applique que lorsqu’aucune clé API Talk n’est configurée.
- `providers.*.voiceAliases` permet aux directives Talk d’utiliser des noms conviviaux.
- `silenceTimeoutMs` contrôle combien de temps le mode Talk attend après le silence de l’utilisateur avant d’envoyer la transcription. Non défini conserve la fenêtre de pause par défaut de la plateforme (`700 ms sur macOS et Android, 900 ms sur iOS`).

---

## Outils

### Profils d’outils

`tools.profile` définit une liste d’autorisation de base avant `tools.allow`/`tools.deny` :

L’intégration locale définit par défaut les nouvelles configurations locales sur `tools.profile: "coding"` lorsqu’il n’est pas défini (les profils explicites existants sont conservés).

| Profil      | Inclut                                                                                                                         |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `minimal`   | `session_status` uniquement                                                                                                    |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                      |
| `full`      | Aucune restriction (identique à non défini)                                                                                    |

### Groupes d’outils

| Groupe             | Outils                                                                                                                  |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` est accepté comme alias de `exec`)                                         |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                 |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                  |
| `group:ui`         | `browser`, `canvas`                                                                                                    |
| `group:automation` | `cron`, `gateway`                                                                                                      |
| `group:messaging`  | `message`                                                                                                              |
| `group:nodes`      | `nodes`                                                                                                                |
| `group:agents`     | `agents_list`                                                                                                          |
| `group:media`      | `image`, `image_generate`, `video_generate`, `tts`                                                                     |
| `group:openclaw`   | Tous les outils intégrés (hors plugins fournisseurs)                                                                   |

### `tools.allow` / `tools.deny`

Politique globale d’autorisation/refus des outils (le refus l’emporte). Insensible à la casse, prend en charge les jokers `*`. S’applique même lorsque le sandbox Docker est désactivé.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Restreint davantage les outils pour des fournisseurs ou modèles spécifiques. Ordre : profil de base → profil fournisseur → autorisation/refus.

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

Contrôle l’accès `exec` élevé hors du sandbox :

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

- Le remplacement par agent (`agents.list[].tools.elevated`) ne peut que restreindre davantage.
- `/elevated on|off|ask|full` stocke l’état par session ; les directives en ligne s’appliquent à un seul message.
- L’`exec` élevé contourne le sandboxing et utilise le chemin d’échappement configuré (`gateway` par défaut, ou `node` lorsque la cible d’exécution est `node`).

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

Les vérifications de sécurité contre les boucles d’outils sont **désactivées par défaut**. Définissez `enabled: true` pour activer la détection.
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

- `historySize` : historique maximal des appels d’outils conservé pour l’analyse des boucles.
- `warningThreshold` : seuil de motif répétitif sans progression pour les avertissements.
- `criticalThreshold` : seuil répétitif plus élevé pour bloquer les boucles critiques.
- `globalCircuitBreakerThreshold` : seuil d’arrêt strict pour toute exécution sans progression.
- `detectors.genericRepeat` : avertit en cas d’appels répétés au même outil avec les mêmes arguments.
- `detectors.knownPollNoProgress` : avertit/bloque pour les outils d’interrogation connus (`process.poll`, `command_status`, etc.).
- `detectors.pingPong` : avertit/bloque pour les motifs alternés de paires sans progression.
- Si `warningThreshold >= criticalThreshold` ou `criticalThreshold >= globalCircuitBreakerThreshold`, la validation échoue.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
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

Configure la compréhension des médias entrants (image/audio/vidéo) :

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // opt-in: send finished async music/video directly to the channel
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

<Accordion title="Champs des entrées de modèle média">

**Entrée fournisseur** (`type: "provider"` ou omis) :

- `provider` : ID du fournisseur d’API (`openai`, `anthropic`, `google`/`gemini`, `groq`, etc.)
- `model` : remplacement de l’ID du modèle
- `profile` / `preferredProfile` : sélection de profil `auth-profiles.json`

**Entrée CLI** (`type: "cli"`) :

- `command` : exécutable à lancer
- `args` : arguments avec modèles (prend en charge `{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}`, etc.)

**Champs communs :**

- `capabilities` : liste facultative (`image`, `audio`, `video`). Valeurs par défaut : `openai`/`anthropic`/`minimax` → image, `google` → image+audio+video, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language` : remplacements par entrée.
- En cas d’échec, bascule vers l’entrée suivante.

L’authentification fournisseur suit l’ordre standard : `auth-profiles.json` → variables d’environnement → `models.providers.*.apiKey`.

**Champs de fin asynchrone :**

- `asyncCompletion.directSend` : lorsqu’il vaut `true`, les tâches asynchrones terminées `music_generate`
  et `video_generate` tentent d’abord une livraison directe au canal. Par défaut : `false`
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

Par défaut : `tree` (session actuelle + sessions engendrées par elle, comme les sous-agents).

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

Remarques :

- `self` : uniquement la clé de session actuelle.
- `tree` : session actuelle + sessions engendrées par la session actuelle (sous-agents).
- `agent` : toute session appartenant à l’ID d’agent actuel (peut inclure d’autres utilisateurs si vous utilisez des sessions par expéditeur sous le même ID d’agent).
- `all` : toute session. Le ciblage inter-agents exige toujours `tools.agentToAgent`.
- Restriction sandbox : lorsque la session actuelle est sandboxée et que `agents.defaults.sandbox.sessionToolsVisibility="spawned"`, la visibilité est forcée à `tree` même si `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

Contrôle la prise en charge des pièces jointes intégrées pour `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

Remarques :

- Les pièces jointes ne sont prises en charge que pour `runtime: "subagent"`. Le runtime ACP les rejette.
- Les fichiers sont matérialisés dans le workspace enfant sous `.openclaw/attachments/<uuid>/` avec un `.manifest.json`.
- Le contenu des pièces jointes est automatiquement caviardé dans la persistance de transcription.
- Les entrées base64 sont validées avec des vérifications strictes d’alphabet/remplissage et une protection de taille avant décodage.
- Les permissions sont `0700` pour les répertoires et `0600` pour les fichiers.
- Le nettoyage suit la politique `cleanup` : `delete` supprime toujours les pièces jointes ; `keep` ne les conserve que lorsque `retainOnSessionKeep: true`.

### `tools.experimental`

Indicateurs expérimentaux d’outils intégrés. Désactivés par défaut sauf lorsqu’une règle d’activation automatique stricte-agentique GPT-5 s’applique.

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

Remarques :

- `planTool` : active l’outil structuré expérimental `update_plan` pour le suivi de travaux non triviaux à plusieurs étapes.
- Par défaut : `false` sauf si `agents.defaults.embeddedPi.executionContract` (ou un remplacement par agent) est défini sur `"strict-agentic"` pour une exécution OpenAI ou OpenAI Codex de la famille GPT-5. Définissez `true` pour forcer l’activation hors de ce cadre, ou `false` pour le laisser désactivé même pour les exécutions GPT-5 strict-agentiques.
- Lorsqu’il est activé, le prompt système ajoute également des consignes d’utilisation afin que le modèle ne l’utilise que pour un travail substantiel et conserve au plus une étape `in_progress`.

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

- `model` : modèle par défaut pour les sous-agents engendrés. S’il est omis, les sous-agents héritent du modèle de l’appelant.
- `allowAgents` : liste d’autorisation par défaut des ID d’agents cibles pour `sessions_spawn` lorsque l’agent demandeur ne définit pas son propre `subagents.allowAgents` (`["*"]` = n’importe lequel ; par défaut : même agent uniquement).
- `runTimeoutSeconds` : délai d’expiration par défaut (en secondes) pour `sessions_spawn` lorsque l’appel d’outil omet `runTimeoutSeconds`. `0` signifie aucun délai.
- Politique d’outils par sous-agent : `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Fournisseurs personnalisés et URL de base

OpenClaw utilise le catalogue intégré de modèles. Ajoutez des fournisseurs personnalisés via `models.providers` dans la configuration ou `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (default) | replace
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

- Utilisez `authHeader: true` + `headers` pour des besoins d’authentification personnalisés.
- Remplacez la racine de configuration agent avec `OPENCLAW_AGENT_DIR` (ou `PI_CODING_AGENT_DIR`, alias hérité de variable d’environnement).
- Priorité de fusion pour les ID de fournisseur correspondants :
  - Les valeurs `baseUrl` d’agent `models.json` non vides l’emportent.
  - Les valeurs `apiKey` d’agent non vides l’emportent uniquement lorsque ce fournisseur n’est pas géré par SecretRef dans le contexte actuel de configuration/profil d’authentification.
  - Les valeurs `apiKey` de fournisseur gérées par SecretRef sont rafraîchies depuis les marqueurs de source (`ENV_VAR_NAME` pour les références d’environnement, `secretref-managed` pour les références fichier/exec) au lieu de persister les secrets résolus.
  - Les valeurs d’en-tête de fournisseur gérées par SecretRef sont rafraîchies depuis les marqueurs de source (`secretref-env:ENV_VAR_NAME` pour les références d’environnement, `secretref-managed` pour les références fichier/exec).
  - Les `apiKey`/`baseUrl` d’agent vides ou absents reviennent à `models.providers` dans la configuration.
  - Les `contextWindow`/`maxTokens` des modèles correspondants utilisent la valeur la plus élevée entre la configuration explicite et les valeurs implicites du catalogue.
  - Les `contextTokens` des modèles correspondants conservent un plafond d’exécution explicite lorsqu’il est présent ; utilisez-le pour limiter le contexte effectif sans modifier les métadonnées natives du modèle.
  - Utilisez `models.mode: "replace"` lorsque vous voulez que la configuration réécrive entièrement `models.json`.
  - La persistance des marqueurs est pilotée par la source : les marqueurs sont écrits à partir de l’instantané actif de la configuration source (avant résolution), et non à partir des valeurs secrètes d’exécution résolues.

### Détails des champs fournisseur

- `models.mode` : comportement du catalogue de fournisseurs (`merge` ou `replace`).
- `models.providers` : map de fournisseurs personnalisés indexée par ID de fournisseur.
- `models.providers.*.api` : adaptateur de requête (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai`, etc.).
- `models.providers.*.apiKey` : identifiant de fournisseur (préférez SecretRef / la substitution d’environnement).
- `models.providers.*.auth` : stratégie d’authentification (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat` : pour Ollama + `openai-completions`, injecte `options.num_ctx` dans les requêtes (par défaut : `true`).
- `models.providers.*.authHeader` : force le transport de l’identifiant dans l’en-tête `Authorization` lorsque requis.
- `models.providers.*.baseUrl` : URL de base de l’API en amont.
- `models.providers.*.headers` : en-têtes statiques supplémentaires pour le routage proxy/locataire.
- `models.providers.*.request` : remplacements de transport pour les requêtes HTTP du fournisseur de modèle.
  - `request.headers` : en-têtes supplémentaires (fusionnés avec les valeurs par défaut du fournisseur). Les valeurs acceptent SecretRef.
  - `request.auth` : remplacement de stratégie d’authentification. Modes : `"provider-default"` (utilise l’authentification intégrée du fournisseur), `"authorization-bearer"` (avec `token`), `"header"` (avec `headerName`, `value`, `prefix` facultatif).
  - `request.proxy` : remplacement de proxy HTTP. Modes : `"env-proxy"` (utilise les variables d’environnement `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (avec `url`). Les deux modes acceptent un sous-objet `tls` facultatif.
  - `request.tls` : remplacement TLS pour les connexions directes. Champs : `ca`, `cert`, `key`, `passphrase` (acceptent tous SecretRef), `serverName`, `insecureSkipVerify`.
  - `request.allowPrivateNetwork` : lorsqu’il vaut `true`, autorise HTTPS vers `baseUrl` lorsque DNS se résout vers des plages privées, CGNAT ou similaires, via la garde SSRF de récupération HTTP du fournisseur (activation explicite opérateur pour les points de terminaison OpenAI compatibles auto-hébergés de confiance). WebSocket utilise le même `request` pour les en-têtes/TLS, mais pas cette garde SSRF de récupération. Par défaut `false`.
- `models.providers.*.models` : entrées explicites du catalogue de modèles du fournisseur.
- `models.providers.*.models.*.contextWindow` : métadonnées de fenêtre de contexte native du modèle.
- `models.providers.*.models.*.contextTokens` : plafond de contexte d’exécution facultatif. Utilisez-le lorsque vous voulez un budget de contexte effectif plus petit que `contextWindow` natif du modèle.
- `models.providers.*.models.*.compat.supportsDeveloperRole` : indice de compatibilité facultatif. Pour `api: "openai-completions"` avec un `baseUrl` non vide et non natif (hôte différent de `api.openai.com`), OpenClaw le force à `false` à l’exécution. Un `baseUrl` vide/omis conserve le comportement OpenAI par défaut.
- `models.providers.*.models.*.compat.requiresStringContent` : indice de compatibilité facultatif pour les points de terminaison de chat OpenAI compatibles qui n’acceptent que des chaînes. Lorsqu’il vaut `true`, OpenClaw aplatie les tableaux `messages[].content` purement textuels en chaînes simples avant d’envoyer la requête.
- `plugins.entries.amazon-bedrock.config.discovery` : racine des paramètres d’auto-détection Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled` : active/désactive la détection implicite.
- `plugins.entries.amazon-bedrock.config.discovery.region` : région AWS pour la détection.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter` : filtre facultatif d’ID de fournisseur pour une détection ciblée.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval` : intervalle d’interrogation pour le rafraîchissement de la détection.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow` : fenêtre de contexte de repli pour les modèles détectés.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens` : nombre maximal de jetons de sortie de repli pour les modèles détectés.

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

Utilisez `cerebras/zai-glm-4.7` pour Cerebras ; `zai/glm-4.7` pour Z.AI en direct.

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

Définissez `OPENCODE_API_KEY` (ou `OPENCODE_ZEN_API_KEY`). Utilisez les références `opencode/...` pour le catalogue Zen ou les références `opencode-go/...` pour le catalogue Go. Raccourci : `openclaw onboard --auth-choice opencode-zen` ou `openclaw onboard --auth-choice opencode-go`.

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

Définissez `ZAI_API_KEY`. `z.ai/*` et `z-ai/*` sont des alias acceptés. Raccourci : `openclaw onboard --auth-choice zai-api-key`.

- Point de terminaison général : `https://api.z.ai/api/paas/v4`
- Point de terminaison de code (par défaut) : `https://api.z.ai/api/coding/paas/v4`
- Pour le point de terminaison général, définissez un fournisseur personnalisé avec le remplacement de l’URL de base.

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

Pour le point de terminaison Chine : `baseUrl: "https://api.moonshot.cn/v1"` ou `openclaw onboard --auth-choice moonshot-api-key-cn`.

Les points de terminaison Moonshot natifs annoncent la compatibilité d’usage du streaming sur le transport partagé
`openai-completions`, et OpenClaw s’appuie sur les capacités du point de terminaison
plutôt que sur le seul ID de fournisseur intégré.

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

Compatible Anthropic, fournisseur intégré. Raccourci : `openclaw onboard --auth-choice kimi-code-api-key`.

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

L’URL de base doit omettre `/v1` (le client Anthropic l’ajoute). Raccourci : `openclaw onboard --auth-choice synthetic-api-key`.

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

Définissez `MINIMAX_API_KEY`. Raccourcis :
`openclaw onboard --auth-choice minimax-global-api` ou
`openclaw onboard --auth-choice minimax-cn-api`.
Le catalogue de modèles utilise par défaut uniquement M2.7.
Sur le chemin de streaming compatible Anthropic, OpenClaw désactive thinking de MiniMax
par défaut sauf si vous définissez explicitement `thinking`. `/fast on` ou
`params.fastMode: true` réécrit `MiniMax-M2.7` en
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="Modèles locaux (LM Studio)">

Voir [Modèles locaux](/fr/gateway/local-models). En bref : exécutez un grand modèle local via l’API Responses de LM Studio sur du matériel sérieux ; conservez les modèles hébergés fusionnés pour le repli.

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
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled` : liste d’autorisation facultative pour les Skills groupés uniquement (les Skills gérés/workspace ne sont pas affectés).
- `load.extraDirs` : racines supplémentaires de Skills partagés (priorité la plus faible).
- `install.preferBrew` : lorsqu’il vaut true, préfère les installateurs Homebrew lorsque `brew` est
  disponible avant de revenir à d’autres types d’installateurs.
- `install.nodeManager` : préférence d’installateur Node pour les spécifications `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` désactive un Skills même s’il est groupé/installé.
- `entries.<skillKey>.apiKey` : champ pratique pour les Skills déclarant une variable d’environnement principale (chaîne en clair ou objet SecretRef).

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

- Chargé depuis `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions`, ainsi que `plugins.load.paths`.
- La découverte accepte les plugins OpenClaw natifs ainsi que les bundles Codex et Claude compatibles, y compris les bundles Claude sans manifeste avec disposition par défaut.
- **Les modifications de configuration exigent un redémarrage du Gateway.**
- `allow` : liste d’autorisation facultative (seuls les plugins listés sont chargés). `deny` l’emporte.
- `plugins.entries.<id>.apiKey` : champ pratique de clé API au niveau du plugin (lorsqu’il est pris en charge par le plugin).
- `plugins.entries.<id>.env` : map de variables d’environnement circonscrite au plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection` : lorsqu’il vaut `false`, le cœur bloque `before_prompt_build` et ignore les champs de mutation du prompt provenant de l’ancien `before_agent_start`, tout en conservant les anciens `modelOverride` et `providerOverride`. S’applique aux hooks de plugins natifs et aux répertoires de hooks fournis par bundle pris en charge.
- `plugins.entries.<id>.subagent.allowModelOverride` : fait explicitement confiance à ce plugin pour demander des remplacements `provider` et `model` par exécution pour les exécutions de sous-agent en arrière-plan.
- `plugins.entries.<id>.subagent.allowedModels` : liste d’autorisation facultative des cibles canoniques `provider/model` pour les remplacements de sous-agent de confiance. Utilisez `"*"` uniquement lorsque vous voulez délibérément autoriser n’importe quel modèle.
- `plugins.entries.<id>.config` : objet de configuration défini par le plugin (validé par le schéma du plugin OpenClaw natif lorsqu’il est disponible).
- `plugins.entries.firecrawl.config.webFetch` : paramètres du fournisseur de récupération web Firecrawl.
  - `apiKey` : clé API Firecrawl (accepte SecretRef). Revient à `plugins.entries.firecrawl.config.webSearch.apiKey`, à l’ancien `tools.web.fetch.firecrawl.apiKey` ou à la variable d’environnement `FIRECRAWL_API_KEY`.
  - `baseUrl` : URL de base de l’API Firecrawl (par défaut : `https://api.firecrawl.dev`).
  - `onlyMainContent` : extrait uniquement le contenu principal des pages (par défaut : `true`).
  - `maxAgeMs` : âge maximal du cache en millisecondes (par défaut : `172800000` / 2 jours).
  - `timeoutSeconds` : délai d’expiration de la requête de scraping en secondes (par défaut : `60`).
- `plugins.entries.xai.config.xSearch` : paramètres xAI X Search (recherche web Grok).
  - `enabled` : active le fournisseur X Search.
  - `model` : modèle Grok à utiliser pour la recherche (par ex. `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming` : paramètres de Dreaming de la mémoire. Voir [Dreaming](/fr/concepts/dreaming) pour les phases et les seuils.
  - `enabled` : interrupteur maître de Dreaming (par défaut `false`).
  - `frequency` : cadence Cron pour chaque balayage complet de Dreaming (`"0 3 * * *"` par défaut).
  - la politique de phase et les seuils sont des détails d’implémentation (pas des clés de configuration destinées à l’utilisateur).
- La configuration complète de la mémoire se trouve dans [Référence de configuration de la mémoire](/fr/reference/memory-config) :
  - `agents.defaults.memorySearch.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Les plugins de bundle Claude activés peuvent également contribuer des valeurs par défaut Pi embarquées depuis `settings.json` ; OpenClaw les applique comme paramètres d’agent nettoyés, pas comme correctifs bruts de configuration OpenClaw.
- `plugins.slots.memory` : choisit l’ID du plugin mémoire actif, ou `"none"` pour désactiver les plugins mémoire.
- `plugins.slots.contextEngine` : choisit l’ID du plugin moteur de contexte actif ; par défaut `"legacy"` sauf si vous installez et sélectionnez un autre moteur.
- `plugins.installs` : métadonnées d’installation gérées par la CLI utilisées par `openclaw plugins update`.
  - Inclut `source`, `spec`, `sourcePath`, `installPath`, `version`, `resolvedName`, `resolvedVersion`, `resolvedSpec`, `integrity`, `shasum`, `resolvedAt`, `installedAt`.
  - Traitez `plugins.installs.*` comme un état géré ; préférez les commandes CLI aux modifications manuelles.

Voir [Plugins](/fr/tools/plugin).

---

## Navigateur

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // allowPrivateNetwork: true, // legacy alias
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
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` est désactivé lorsqu’il n’est pas défini, donc la navigation du navigateur reste stricte par défaut.
- Définissez `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` uniquement lorsque vous faites délibérément confiance à la navigation du navigateur vers un réseau privé.
- En mode strict, les points de terminaison des profils CDP distants (`profiles.*.cdpUrl`) sont soumis au même blocage des réseaux privés lors des vérifications d’accessibilité/découverte.
- `ssrfPolicy.allowPrivateNetwork` reste pris en charge comme alias hérité.
- En mode strict, utilisez `ssrfPolicy.hostnameAllowlist` et `ssrfPolicy.allowedHostnames` pour des exceptions explicites.
- Les profils distants sont en attachement seul (démarrage/arrêt/réinitialisation désactivés).
- `profiles.*.cdpUrl` accepte `http://`, `https://`, `ws://` et `wss://`.
  Utilisez HTTP(S) lorsque vous voulez qu’OpenClaw découvre `/json/version` ; utilisez WS(S)
  lorsque votre fournisseur vous donne une URL WebSocket DevTools directe.
- Les profils `existing-session` sont réservés à l’hôte et utilisent Chrome MCP au lieu de CDP.
- Les profils `existing-session` peuvent définir `userDataDir` pour cibler un profil spécifique
  d’un navigateur basé sur Chromium comme Brave ou Edge.
- Les profils `existing-session` conservent les limites actuelles de la route Chrome MCP :
  actions pilotées par instantané/référence au lieu d’un ciblage par sélecteur CSS, hooks d’envoi
  d’un seul fichier, pas de remplacement du délai d’expiration des boîtes de dialogue, pas de `wait --load networkidle`, et pas de
  `responsebody`, export PDF, interception de téléchargement ou actions par lots.
- Les profils locaux gérés `openclaw` assignent automatiquement `cdpPort` et `cdpUrl` ; ne définissez
  `cdpUrl` explicitement que pour un CDP distant.
- Ordre d’auto-détection : navigateur par défaut s’il est basé sur Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- Service de contrôle : loopback uniquement (port dérivé de `gateway.port`, par défaut `18791`).
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
      avatar: "CB", // emoji, short text, image URL, or data URI
    },
  },
}
```

- `seamColor` : couleur d’accent pour l’habillage de l’interface native de l’application (teinte de bulle du mode Talk, etc.).
- `assistant` : remplacement d’identité pour l’interface utilisateur Control. Revient à l’identité de l’agent actif.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // or OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // for mode=trusted-proxy; see /gateway/trusted-proxy-auth
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
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // dangerous: allow absolute external http(s) embed URLs
      // allowedOrigins: ["https://control.example.com"], // required for non-loopback Control UI
      // dangerouslyAllowHostHeaderOriginFallback: false, // dangerous Host-header origin fallback mode
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
    // Optional. Default false.
    allowRealIpFallback: false,
    tools: {
      // Additional /tools/invoke HTTP denies
      deny: ["browser"],
      // Remove tools from the default HTTP deny list
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

<Accordion title="Détails des champs du Gateway">

- `mode` : `local` (exécuter le Gateway) ou `remote` (se connecter à un Gateway distant). Le Gateway refuse de démarrer sauf en mode `local`.
- `port` : port multiplexé unique pour WS + HTTP. Priorité : `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind` : `auto`, `loopback` (par défaut), `lan` (`0.0.0.0`), `tailnet` (IP Tailscale uniquement) ou `custom`.
- **Alias `bind` hérités** : utilisez les valeurs de mode bind dans `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), pas les alias d’hôte (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Remarque Docker** : la liaison `loopback` par défaut écoute sur `127.0.0.1` à l’intérieur du conteneur. Avec le réseau bridge Docker (`-p 18789:18789`), le trafic arrive sur `eth0`, donc le Gateway est inaccessible. Utilisez `--network host`, ou définissez `bind: "lan"` (ou `bind: "custom"` avec `customBindHost: "0.0.0.0"`) pour écouter sur toutes les interfaces.
- **Auth** : requise par défaut. Les liaisons non loopback exigent l’authentification Gateway. En pratique, cela signifie un jeton/mot de passe partagé ou un proxy inverse à gestion d’identité avec `gateway.auth.mode: "trusted-proxy"`. L’assistant d’intégration génère un jeton par défaut.
- Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés (y compris via SecretRef), définissez explicitement `gateway.auth.mode` sur `token` ou `password`. Le démarrage et les flux d’installation/réparation du service échouent lorsque les deux sont configurés et que le mode n’est pas défini.
- `gateway.auth.mode: "none"` : mode explicite sans authentification. À utiliser uniquement pour des configurations loopback locales de confiance ; ce mode n’est volontairement pas proposé dans les invites d’intégration.
- `gateway.auth.mode: "trusted-proxy"` : délègue l’authentification à un proxy inverse à gestion d’identité et fait confiance aux en-têtes d’identité provenant de `gateway.trustedProxies` (voir [Authentification par proxy de confiance](/fr/gateway/trusted-proxy-auth)). Ce mode attend une source de proxy **non loopback** ; les proxys inverses loopback sur la même machine ne satisfont pas l’authentification trusted-proxy.
- `gateway.auth.allowTailscale` : lorsqu’il vaut `true`, les en-têtes d’identité Tailscale Serve peuvent satisfaire l’authentification de l’interface utilisateur Control/WebSocket (vérifiés via `tailscale whois`). Les points de terminaison API HTTP n’utilisent **pas** cette authentification par en-tête Tailscale ; ils suivent à la place le mode d’authentification HTTP normal du Gateway. Ce flux sans jeton suppose que l’hôte Gateway est de confiance. Par défaut `true` lorsque `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit` : limiteur facultatif des échecs d’authentification. S’applique par IP cliente et par portée d’authentification (secret partagé et jeton d’appareil sont suivis séparément). Les tentatives bloquées renvoient `429` + `Retry-After`.
  - Sur le chemin asynchrone Tailscale Serve de l’interface utilisateur Control, les tentatives échouées pour un même `{scope, clientIp}` sont sérialisées avant l’écriture de l’échec. Des tentatives mauvaises concurrentes du même client peuvent donc déclencher le limiteur à la deuxième requête au lieu de laisser passer les deux comme de simples non-correspondances.
  - `gateway.auth.rateLimit.exemptLoopback` vaut par défaut `true` ; définissez `false` lorsque vous voulez délibérément limiter aussi le trafic localhost (pour des configurations de test ou des déploiements de proxy stricts).
- Les tentatives d’authentification WS d’origine navigateur sont toujours limitées avec exemption loopback désactivée (défense en profondeur contre la force brute localhost depuis le navigateur).
- En loopback, ces verrouillages provenant d’origines navigateur sont isolés par valeur `Origin`
  normalisée, de sorte que des échecs répétés depuis une origine localhost n’entraînent pas automatiquement
  le verrouillage d’une autre origine.
- `tailscale.mode` : `serve` (tailnet uniquement, liaison loopback) ou `funnel` (public, nécessite une authentification).
- `controlUi.allowedOrigins` : liste d’autorisation explicite des origines navigateur pour les connexions WebSocket au Gateway. Requise lorsque des clients navigateur sont attendus depuis des origines non loopback.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback` : mode dangereux qui active le repli d’origine basé sur l’en-tête Host pour les déploiements qui reposent volontairement sur une politique d’origine fondée sur Host.
- `remote.transport` : `ssh` (par défaut) ou `direct` (ws/wss). Pour `direct`, `remote.url` doit être `ws://` ou `wss://`.
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` : remplacement client « break-glass » qui autorise `ws://` en clair vers des IP réseau privé de confiance ; par défaut, le texte en clair reste limité au loopback.
- `gateway.remote.token` / `.password` sont des champs d’identifiants du client distant. Ils ne configurent pas à eux seuls l’authentification du Gateway.
- `gateway.push.apns.relay.baseUrl` : URL HTTPS de base du relais APNs externe utilisé par les builds iOS officiels/TestFlight après qu’ils ont publié des enregistrements adossés au relais vers le Gateway. Cette URL doit correspondre à l’URL du relais compilée dans le build iOS.
- `gateway.push.apns.relay.timeoutMs` : délai d’envoi Gateway-vers-relais en millisecondes. Par défaut `10000`.
- Les enregistrements adossés au relais sont délégués à une identité spécifique du Gateway. L’app iOS appairée récupère `gateway.identity.get`, inclut cette identité dans l’enregistrement du relais et transmet au Gateway une autorisation d’envoi circonscrite à l’enregistrement. Un autre Gateway ne peut pas réutiliser cet enregistrement stocké.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS` : remplacements d’environnement temporaires pour la configuration de relais ci-dessus.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` : échappatoire réservée au développement pour les URL de relais HTTP loopback. Les URL de relais de production doivent rester en HTTPS.
- `gateway.channelHealthCheckMinutes` : intervalle du moniteur d’état des canaux en minutes. Définissez `0` pour désactiver globalement les redémarrages du moniteur d’état. Par défaut : `5`.
- `gateway.channelStaleEventThresholdMinutes` : seuil de socket obsolète en minutes. Conservez une valeur supérieure ou égale à `gateway.channelHealthCheckMinutes`. Par défaut : `30`.
- `gateway.channelMaxRestartsPerHour` : nombre maximal de redémarrages du moniteur d’état par canal/compte sur une heure glissante. Par défaut : `10`.
- `channels.<provider>.healthMonitor.enabled` : désactivation par canal des redémarrages du moniteur d’état tout en conservant le moniteur global activé.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled` : remplacement par compte pour les canaux multi-comptes. Lorsqu’il est défini, il a priorité sur le remplacement au niveau du canal.
- Les chemins d’appel du Gateway local peuvent utiliser `gateway.remote.*` comme repli uniquement lorsque `gateway.auth.*` n’est pas défini.
- Si `gateway.auth.token` / `gateway.auth.password` est explicitement configuré via SecretRef et non résolu, la résolution échoue en mode fermé (aucun repli distant ne masque cela).
- `trustedProxies` : IP de proxy inverse qui terminent TLS ou injectent des en-têtes client transférés. Listez uniquement les proxys que vous contrôlez. Les entrées loopback restent valides pour les configurations de proxy sur le même hôte / détection locale (par exemple Tailscale Serve ou un proxy inverse local), mais elles ne rendent **pas** les requêtes loopback admissibles à `gateway.auth.mode: "trusted-proxy"`.
- `allowRealIpFallback` : lorsqu’il vaut `true`, le Gateway accepte `X-Real-IP` si `X-Forwarded-For` est absent. Par défaut `false` pour un comportement en mode fermé.
- `gateway.tools.deny` : noms d’outils supplémentaires bloqués pour `POST /tools/invoke` HTTP (étend la liste de refus par défaut).
- `gateway.tools.allow` : supprime des noms d’outils de la liste de refus HTTP par défaut.

</Accordion>

### Points de terminaison compatibles OpenAI

- Chat Completions : désactivé par défaut. Activez avec `gateway.http.endpoints.chatCompletions.enabled: true`.
- API Responses : `gateway.http.endpoints.responses.enabled`.
- Renforcement des entrées URL de Responses :
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    Les listes d’autorisation vides sont traitées comme non définies ; utilisez `gateway.http.endpoints.responses.files.allowUrl=false`
    et/ou `gateway.http.endpoints.responses.images.allowUrl=false` pour désactiver la récupération d’URL.
- En-tête facultatif de durcissement des réponses :
  - `gateway.http.securityHeaders.strictTransportSecurity` (à définir uniquement pour des origines HTTPS que vous contrôlez ; voir [Authentification par proxy de confiance](/fr/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### Isolation multi-instance

Exécutez plusieurs Gateway sur un même hôte avec des ports et répertoires d’état uniques :

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

Indicateurs pratiques : `--dev` (utilise `~/.openclaw-dev` + port `19001`), `--profile <name>` (utilise `~/.openclaw-<name>`).

Voir [Plusieurs Gateway](/fr/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled` : active la terminaison TLS au niveau de l’écouteur Gateway (HTTPS/WSS) (par défaut : `false`).
- `autoGenerate` : génère automatiquement une paire locale cert/key autosignée lorsqu’aucun fichier explicite n’est configuré ; pour usage local/dev uniquement.
- `certPath` : chemin du système de fichiers vers le fichier de certificat TLS.
- `keyPath` : chemin du système de fichiers vers la clé privée TLS ; gardez des permissions restrictives.
- `caPath` : chemin facultatif vers un bundle CA pour la vérification client ou des chaînes de confiance personnalisées.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode` : contrôle la manière dont les modifications de configuration sont appliquées à l’exécution.
  - `"off"` : ignore les modifications en direct ; les changements nécessitent un redémarrage explicite.
  - `"restart"` : redémarre toujours le processus Gateway lors d’un changement de configuration.
  - `"hot"` : applique les changements dans le processus sans redémarrage.
  - `"hybrid"` (par défaut) : tente d’abord un rechargement à chaud ; revient à un redémarrage si nécessaire.
- `debounceMs` : fenêtre d’antirebond en ms avant l’application des changements de configuration (entier non négatif).
- `deferralTimeoutMs` : durée maximale en ms d’attente des opérations en cours avant de forcer un redémarrage (par défaut : `300000` = 5 minutes).

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    maxBodyBytes: 262144,
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.4-mini",
      },
    ],
  },
}
```

Auth : `Authorization: Bearer <token>` ou `x-openclaw-token: <token>`.
Les jetons de hook dans la chaîne de requête sont rejetés.

Remarques de validation et de sécurité :

- `hooks.enabled=true` exige un `hooks.token` non vide.
- `hooks.token` doit être **distinct** de `gateway.auth.token` ; réutiliser le jeton Gateway est rejeté.
- `hooks.path` ne peut pas être `/` ; utilisez un sous-chemin dédié comme `/hooks`.
- Si `hooks.allowRequestSessionKey=true`, contraignez `hooks.allowedSessionKeyPrefixes` (par exemple `["hook:"]`).

**Points de terminaison :**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - `sessionKey` provenant de la charge utile de requête n’est accepté que lorsque `hooks.allowRequestSessionKey=true` (par défaut : `false`).
- `POST /hooks/<name>` → résolu via `hooks.mappings`

<Accordion title="Détails des mappings">

- `match.path` correspond au sous-chemin après `/hooks` (par ex. `/hooks/gmail` → `gmail`).
- `match.source` correspond à un champ de la charge utile pour les chemins génériques.
- Les modèles comme `{{messages[0].subject}}` lisent depuis la charge utile.
- `transform` peut pointer vers un module JS/TS renvoyant une action de hook.
  - `transform.module` doit être un chemin relatif et rester dans `hooks.transformsDir` (les chemins absolus et la traversée sont rejetés).
- `agentId` achemine vers un agent spécifique ; les ID inconnus reviennent à l’agent par défaut.
- `allowedAgentIds` : restreint le routage explicite (`*` ou omis = tout autoriser, `[]` = tout refuser).
- `defaultSessionKey` : clé de session fixe facultative pour les exécutions d’agent hook sans `sessionKey` explicite.
- `allowRequestSessionKey` : autorise les appelants de `/hooks/agent` à définir `sessionKey` (par défaut : `false`).
- `allowedSessionKeyPrefixes` : liste d’autorisation facultative de préfixes pour les valeurs explicites de `sessionKey` (requête + mapping), par ex. `["hook:"]`.
- `deliver: true` envoie la réponse finale vers un canal ; `channel` vaut `last` par défaut.
- `model` remplace le LLM pour cette exécution de hook (doit être autorisé si le catalogue de modèles est défini).

</Accordion>

### Intégration Gmail

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

- Le Gateway démarre automatiquement `gog gmail watch serve` au démarrage lorsqu’il est configuré. Définissez `OPENCLAW_SKIP_GMAIL_WATCHER=1` pour le désactiver.
- N’exécutez pas un `gog gmail watch serve` séparé en parallèle du Gateway.

---

## Hôte Canvas

```json5
{
  canvasHost: {
    root: "~/.openclaw/workspace/canvas",
    liveReload: true,
    // enabled: false, // or OPENCLAW_SKIP_CANVAS_HOST=1
  },
}
```

- Sert du HTML/CSS/JS modifiable par l’agent et A2UI sur HTTP sous le port du Gateway :
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- Local uniquement : conservez `gateway.bind: "loopback"` (par défaut).
- Liaisons non loopback : les routes canvas exigent l’authentification Gateway (token/password/trusted-proxy), comme les autres surfaces HTTP du Gateway.
- Les WebViews Node n’envoient généralement pas d’en-têtes d’authentification ; après appairage et connexion d’un Node, le Gateway annonce des URL de capacité circonscrites au Node pour l’accès à canvas/A2UI.
- Les URL de capacité sont liées à la session WS active du Node et expirent rapidement. Le repli basé sur l’IP n’est pas utilisé.
- Injecte un client de rechargement en direct dans le HTML servi.
- Crée automatiquement un `index.html` de départ lorsqu’il est vide.
- Sert aussi A2UI à `/__openclaw__/a2ui/`.
- Les changements exigent un redémarrage du Gateway.
- Désactivez le rechargement en direct pour les grands répertoires ou en cas d’erreurs `EMFILE`.

---

## Découverte

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (par défaut) : omet `cliPath` + `sshPort` des enregistrements TXT.
- `full` : inclut `cliPath` + `sshPort`.
- Le nom d’hôte vaut par défaut `openclaw`. Remplacez-le avec `OPENCLAW_MDNS_HOSTNAME`.

### Zone étendue (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

Écrit une zone DNS-SD unicast sous `~/.openclaw/dns/`. Pour une découverte inter-réseaux, associez-la à un serveur DNS (CoreDNS recommandé) + DNS scindé Tailscale.

Configuration : `openclaw dns setup --apply`.

---

## Environnement

### `env` (variables d’environnement intégrées)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- Les variables d’environnement intégrées ne sont appliquées que si l’environnement du processus ne contient pas déjà la clé.
- Fichiers `.env` : `.env` du répertoire courant + `~/.openclaw/.env` (aucun des deux ne remplace les variables existantes).
- `shellEnv` : importe les clés attendues manquantes depuis votre profil de shell de connexion.
- Voir [Environnement](/fr/help/environment) pour l’ordre de priorité complet.

### Substitution de variables d’environnement

Référencez des variables d’environnement dans n’importe quelle chaîne de configuration avec `${VAR_NAME}` :

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- Seuls les noms en majuscules sont reconnus : `[A-Z_][A-Z0-9_]*`.
- Les variables manquantes/vides provoquent une erreur au chargement de la configuration.
- Échappez avec `$${VAR}` pour obtenir un littéral `${VAR}`.
- Fonctionne avec `$include`.

---

## Secrets

Les références de secret sont additives : les valeurs en clair continuent de fonctionner.

### `SecretRef`

Utilisez une seule forme d’objet :

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

Validation :

- motif `provider` : `^[a-z][a-z0-9_-]{0,63}$`
- motif `source: "env"` pour `id` : `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` pour `id` : pointeur JSON absolu (par exemple `"/providers/openai/apiKey"`)
- motif `source: "exec"` pour `id` : `^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`
- les `id` de `source: "exec"` ne doivent pas contenir de segments de chemin `/./` ou `/../` séparés par des slashs (par exemple `a/../b` est rejeté)

### Surface d’identifiants prise en charge

- Matrice canonique : [Surface d’identifiants SecretRef](/fr/reference/secretref-credential-surface)
- `secrets apply` cible les chemins d’identifiants pris en charge dans `openclaw.json`.
- Les références de `auth-profiles.json` sont incluses dans la résolution à l’exécution et dans la couverture d’audit.

### Configuration des fournisseurs de secrets

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optional explicit env provider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

Remarques :

- Le fournisseur `file` prend en charge `mode: "json"` et `mode: "singleValue"` (`id` doit être `"value"` en mode singleValue).
- Le fournisseur `exec` exige un chemin `command` absolu et utilise des charges utiles de protocole sur stdin/stdout.
- Par défaut, les chemins de commande symboliques sont rejetés. Définissez `allowSymlinkCommand: true` pour autoriser les chemins symboliques tout en validant le chemin cible résolu.
- Si `trustedDirs` est configuré, la vérification des répertoires de confiance s’applique au chemin cible résolu.
- L’environnement enfant de `exec` est minimal par défaut ; transmettez explicitement les variables requises avec `passEnv`.
- Les références de secret sont résolues au moment de l’activation dans un instantané en mémoire, puis les chemins de requête ne lisent que cet instantané.
- Le filtrage des surfaces actives s’applique pendant l’activation : les références non résolues sur des surfaces activées font échouer le démarrage/rechargement, tandis que les surfaces inactives sont ignorées avec diagnostics.

---

## Stockage d’authentification

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai-codex:personal": { provider: "openai-codex", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      "openai-codex": ["openai-codex:personal"],
    },
  },
}
```

- Les profils par agent sont stockés dans `<agentDir>/auth-profiles.json`.
- `auth-profiles.json` prend en charge les références au niveau valeur (`keyRef` pour `api_key`, `tokenRef` pour `token`) pour les modes d’identifiants statiques.
- Les profils en mode OAuth (`auth.profiles.<id>.mode = "oauth"`) ne prennent pas en charge les identifiants de profil d’authentification adossés à SecretRef.
- Les identifiants statiques d’exécution proviennent d’instantanés résolus en mémoire ; les anciennes entrées statiques `auth.json` sont nettoyées lorsqu’elles sont découvertes.
- Les anciennes importations OAuth proviennent de `~/.openclaw/credentials/oauth.json`.
- Voir [OAuth](/fr/concepts/oauth).
- Comportement d’exécution des secrets et outils `audit/configure/apply` : [Gestion des secrets](/fr/gateway/secrets).

### `auth.cooldowns`

```json5
{
  auth: {
    cooldowns: {
      billingBackoffHours: 5,
      billingBackoffHoursByProvider: { anthropic: 3, openai: 8 },
      billingMaxHours: 24,
      authPermanentBackoffMinutes: 10,
      authPermanentMaxMinutes: 60,
      failureWindowHours: 24,
      overloadedProfileRotations: 1,
      overloadedBackoffMs: 0,
      rateLimitedProfileRotations: 1,
    },
  },
}
```

- `billingBackoffHours` : repli de base en heures lorsqu’un profil échoue à cause de vraies
  erreurs de facturation / crédit insuffisant (par défaut : `5`). Un texte de facturation explicite peut
  toujours aboutir ici même sur des réponses `401`/`403`, mais les correspondances de texte
  spécifiques au fournisseur restent circonscrites au fournisseur qui les possède (par exemple OpenRouter
  `Key limit exceeded`). Les messages `402` de fenêtre d’usage
  réessayables ou de limite de dépenses d’organisation/espace de travail restent dans le chemin `rate_limit`
  à la place.
- `billingBackoffHoursByProvider` : remplacements facultatifs par fournisseur pour les heures de repli de facturation.
- `billingMaxHours` : plafond en heures pour la croissance exponentielle du repli de facturation (par défaut : `24`).
- `authPermanentBackoffMinutes` : repli de base en minutes pour les échecs `auth_permanent` à forte confiance (par défaut : `10`).
- `authPermanentMaxMinutes` : plafond en minutes pour la croissance du repli `auth_permanent` (par défaut : `60`).
- `failureWindowHours` : fenêtre glissante en heures utilisée pour les compteurs de repli (par défaut : `24`).
- `overloadedProfileRotations` : nombre maximal de rotations de profil d’authentification pour un même fournisseur en cas de surcharge avant de basculer vers le modèle de repli (par défaut : `1`). Les formes de fournisseur occupé comme `ModelNotReadyException` aboutissent ici.
- `overloadedBackoffMs` : délai fixe avant de réessayer une rotation de fournisseur/profil surchargé (par défaut : `0`).
- `rateLimitedProfileRotations` : nombre maximal de rotations de profil d’authentification pour un même fournisseur en cas de limitation de débit avant de basculer vers le modèle de repli (par défaut : `1`). Ce compartiment de limitation de débit inclut du texte typé fournisseur tel que `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded` et `resource exhausted`.

---

## Journalisation

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Fichier journal par défaut : `/tmp/openclaw/openclaw-YYYY-MM-DD.log`.
- Définissez `logging.file` pour un chemin stable.
- `consoleLevel` passe à `debug` avec `--verbose`.
- `maxFileBytes` : taille maximale du fichier journal en octets avant suppression des écritures (entier positif ; par défaut : `524288000` = 500 MB). Utilisez une rotation de journaux externe pour les déploiements de production.

---

## Diagnostics

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],
    stuckSessionWarnMs: 30000,

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      sampleRate: 1.0,
      flushIntervalMs: 5000,
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled` : interrupteur maître de sortie d’instrumentation (par défaut : `true`).
- `flags` : tableau de chaînes d’indicateurs activant une sortie de journal ciblée (prend en charge les jokers comme `"telegram.*"` ou `"*"`).
- `stuckSessionWarnMs` : seuil d’ancienneté en ms pour émettre des avertissements de session bloquée pendant qu’une session reste en état de traitement.
- `otel.enabled` : active le pipeline d’export OpenTelemetry (par défaut : `false`).
- `otel.endpoint` : URL du collecteur pour l’export OTel.
- `otel.protocol` : `"http/protobuf"` (par défaut) ou `"grpc"`.
- `otel.headers` : en-têtes de métadonnées HTTP/gRPC supplémentaires envoyés avec les requêtes d’export OTel.
- `otel.serviceName` : nom du service pour les attributs de ressource.
- `otel.traces` / `otel.metrics` / `otel.logs` : activent l’export des traces, métriques ou journaux.
- `otel.sampleRate` : taux d’échantillonnage des traces `0`–`1`.
- `otel.flushIntervalMs` : intervalle périodique de vidage de télémétrie en ms.
- `cacheTrace.enabled` : journalise des instantanés de trace de cache pour les exécutions embarquées (par défaut : `false`).
- `cacheTrace.filePath` : chemin de sortie pour le JSONL de trace de cache (par défaut : `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem` : contrôlent ce qui est inclus dans la sortie de trace de cache (tous par défaut : `true`).

---

## Mise à jour

```json5
{
  update: {
    channel: "stable", // stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
      stableDelayHours: 6,
      stableJitterHours: 12,
      betaCheckIntervalHours: 1,
    },
  },
}
```

- `channel` : canal de publication pour les installations npm/git — `"stable"`, `"beta"` ou `"dev"`.
- `checkOnStart` : vérifie les mises à jour npm au démarrage du Gateway (par défaut : `true`).
- `auto.enabled` : active la mise à jour automatique en arrière-plan pour les installations de paquets (par défaut : `false`).
- `auto.stableDelayHours` : délai minimal en heures avant application automatique sur le canal stable (par défaut : `6` ; max : `168`).
- `auto.stableJitterHours` : fenêtre supplémentaire d’étalement du déploiement du canal stable en heures (par défaut : `12` ; max : `168`).
- `auto.betaCheckIntervalHours` : fréquence des vérifications du canal bêta en heures (par défaut : `1` ; max : `24`).

---

## ACP

```json5
{
  acp: {
    enabled: false,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    maxConcurrentSessions: 10,

    stream: {
      coalesceIdleMs: 50,
      maxChunkChars: 1000,
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
      hiddenBoundarySeparator: "paragraph", // none | space | newline | paragraph
      maxOutputChars: 50000,
      maxSessionUpdateChars: 500,
    },

    runtime: {
      ttlMinutes: 30,
    },
  },
}
```

- `enabled` : porte de fonctionnalité ACP globale (par défaut : `false`).
- `dispatch.enabled` : porte indépendante pour la répartition des tours de session ACP (par défaut : `true`). Définissez `false` pour conserver les commandes ACP disponibles tout en bloquant l’exécution.
- `backend` : ID par défaut du backend d’exécution ACP (doit correspondre à un Plugin d’exécution ACP enregistré).
- `defaultAgent` : ID d’agent ACP cible de repli lorsque les créations ne spécifient pas de cible explicite.
- `allowedAgents` : liste d’autorisation des ID d’agents autorisés pour les sessions d’exécution ACP ; vide signifie aucune restriction supplémentaire.
- `maxConcurrentSessions` : nombre maximal de sessions ACP actives simultanément.
- `stream.coalesceIdleMs` : fenêtre de vidage inactif en ms pour le texte diffusé.
- `stream.maxChunkChars` : taille maximale de bloc avant découpage de la projection par blocs diffusés.
- `stream.repeatSuppression` : supprime les lignes d’état/d’outil répétées par tour (par défaut : `true`).
- `stream.deliveryMode` : `"live"` diffuse progressivement ; `"final_only"` met en tampon jusqu’aux événements terminaux du tour.
- `stream.hiddenBoundarySeparator` : séparateur avant le texte visible après des événements d’outil cachés (par défaut : `"paragraph"`).
- `stream.maxOutputChars` : nombre maximal de caractères de sortie assistant projetés par tour ACP.
- `stream.maxSessionUpdateChars` : nombre maximal de caractères pour les lignes projetées d’état/mise à jour ACP.
- `stream.tagVisibility` : enregistrement des noms de balises vers des remplacements booléens de visibilité pour les événements diffusés.
- `runtime.ttlMinutes` : TTL d’inactivité en minutes pour les workers de session ACP avant nettoyage admissible.
- `runtime.installCommand` : commande d’installation facultative à exécuter lors de l’amorçage d’un environnement d’exécution ACP.

---

## CLI

```json5
{
  cli: {
    banner: {
      taglineMode: "off", // random | default | off
    },
  },
}
```

- `cli.banner.taglineMode` contrôle le style du slogan de la bannière :
  - `"random"` (par défaut) : slogans tournants humoristiques/de saison.
  - `"default"` : slogan neutre fixe (`All your chats, one OpenClaw.`).
  - `"off"` : aucun texte de slogan (le titre/version de la bannière reste affiché).
- Pour masquer toute la bannière (pas seulement les slogans), définissez la variable d’environnement `OPENCLAW_HIDE_BANNER=1`.

---

## Assistant

Métadonnées écrites par les flux CLI de configuration guidée (`onboard`, `configure`, `doctor`) :

```json5
{
  wizard: {
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
  },
}
```

---

## Identité

Voir les champs d’identité de `agents.list` sous [Valeurs par défaut des agents](#agent-defaults).

---

## Bridge (hérité, supprimé)

Les builds actuels n’incluent plus le bridge TCP. Les Node se connectent via le WebSocket Gateway. Les clés `bridge.*` ne font plus partie du schéma de configuration (la validation échoue tant qu’elles ne sont pas supprimées ; `openclaw doctor --fix` peut retirer les clés inconnues).

<Accordion title="Configuration bridge héritée (référence historique)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 2,
    webhook: "https://example.invalid/legacy", // deprecated fallback for stored notify:true jobs
    webhookToken: "replace-with-dedicated-token", // optional bearer token for outbound webhook auth
    sessionRetention: "24h", // duration string or false
    runLog: {
      maxBytes: "2mb", // default 2_000_000 bytes
      keepLines: 2000, // default 2000
    },
  },
}
```

- `sessionRetention` : durée de conservation des sessions d’exécution Cron isolées terminées avant émondage depuis `sessions.json`. Contrôle également le nettoyage des transcriptions Cron archivées supprimées. Par défaut : `24h` ; définissez `false` pour désactiver.
- `runLog.maxBytes` : taille maximale par fichier journal d’exécution (`cron/runs/<jobId>.jsonl`) avant émondage. Par défaut : `2_000_000` octets.
- `runLog.keepLines` : lignes les plus récentes conservées lorsque l’émondage du journal d’exécution est déclenché. Par défaut : `2000`.
- `webhookToken` : jeton bearer utilisé pour la livraison POST Webhook Cron (`delivery.mode = "webhook"`), si omis aucun en-tête d’authentification n’est envoyé.
- `webhook` : URL Webhook de repli héritée obsolète (http/https) utilisée uniquement pour les tâches stockées qui ont encore `notify: true`.

### `cron.retry`

```json5
{
  cron: {
    retry: {
      maxAttempts: 3,
      backoffMs: [30000, 60000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "timeout", "server_error"],
    },
  },
}
```

- `maxAttempts` : nombre maximal de nouvelles tentatives pour les tâches ponctuelles en cas d’erreurs transitoires (par défaut : `3` ; plage : `0`–`10`).
- `backoffMs` : tableau des délais de repli en ms pour chaque tentative de nouvelle tentative (par défaut : `[30000, 60000, 300000]` ; 1 à 10 entrées).
- `retryOn` : types d’erreurs qui déclenchent de nouvelles tentatives — `"rate_limit"`, `"overloaded"`, `"network"`, `"timeout"`, `"server_error"`. Omettez pour réessayer tous les types transitoires.

S’applique uniquement aux tâches Cron ponctuelles. Les tâches récurrentes utilisent une gestion distincte des échecs.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled` : active les alertes d’échec pour les tâches Cron (par défaut : `false`).
- `after` : nombre d’échecs consécutifs avant le déclenchement d’une alerte (entier positif, min : `1`).
- `cooldownMs` : nombre minimal de millisecondes entre des alertes répétées pour une même tâche (entier non négatif).
- `mode` : mode de livraison — `"announce"` envoie via un message de canal ; `"webhook"` publie sur le Webhook configuré.
- `accountId` : ID facultatif de compte ou de canal pour circonscrire la livraison de l’alerte.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- Destination par défaut pour les notifications d’échec Cron sur l’ensemble des tâches.
- `mode` : `"announce"` ou `"webhook"` ; par défaut `"announce"` lorsque suffisamment de données cibles existent.
- `channel` : remplacement de canal pour la livraison par annonce. `"last"` réutilise le dernier canal de livraison connu.
- `to` : cible explicite d’annonce ou URL Webhook. Obligatoire en mode Webhook.
- `accountId` : remplacement facultatif du compte pour la livraison.
- `delivery.failureDestination` par tâche remplace cette valeur par défaut globale.
- Lorsqu’aucune destination d’échec globale ou par tâche n’est définie, les tâches qui livrent déjà via `announce` reviennent à cette cible principale d’annonce en cas d’échec.
- `delivery.failureDestination` n’est pris en charge que pour les tâches `sessionTarget="isolated"` sauf si le `delivery.mode` principal de la tâche est `"webhook"`.

Voir [Tâches Cron](/fr/automation/cron-jobs). Les exécutions Cron isolées sont suivies comme [tâches d’arrière-plan](/fr/automation/tasks).

---

## Variables de modèle média

Espaces réservés de modèle développés dans `tools.media.models[].args` :

| Variable           | Description                                       |
| ------------------ | ------------------------------------------------- |
| `{{Body}}`         | Corps complet du message entrant                  |
| `{{RawBody}}`      | Corps brut (sans wrappers d’historique/expéditeur) |
| `{{BodyStripped}}` | Corps avec les mentions de groupe supprimées      |
| `{{From}}`         | Identifiant de l’expéditeur                       |
| `{{To}}`           | Identifiant de destination                        |
| `{{MessageSid}}`   | ID du message du canal                            |
| `{{SessionId}}`    | UUID de la session actuelle                       |
| `{{IsNewSession}}` | `"true"` lorsqu’une nouvelle session est créée    |
| `{{MediaUrl}}`     | pseudo-URL du média entrant                       |
| `{{MediaPath}}`    | chemin local du média                             |
| `{{MediaType}}`    | type de média (image/audio/document/…)            |
| `{{Transcript}}`   | transcription audio                               |
| `{{Prompt}}`       | prompt média résolu pour les entrées CLI          |
| `{{MaxChars}}`     | nombre maximal de caractères résolu pour les entrées CLI |
| `{{ChatType}}`     | `"direct"` ou `"group"`                           |
| `{{GroupSubject}}` | sujet du groupe (au mieux)                        |
| `{{GroupMembers}}` | aperçu des membres du groupe (au mieux)           |
| `{{SenderName}}`   | nom d’affichage de l’expéditeur (au mieux)        |
| `{{SenderE164}}`   | numéro de téléphone de l’expéditeur (au mieux)    |
| `{{Provider}}`     | indication du fournisseur (whatsapp, telegram, discord, etc.) |

---

## Inclusions de configuration (`$include`)

Divisez la configuration en plusieurs fichiers :

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Comportement de fusion :**

- Fichier unique : remplace l’objet conteneur.
- Tableau de fichiers : fusion profonde dans l’ordre (les suivants remplacent les précédents).
- Clés sœurs : fusionnées après les inclusions (remplacent les valeurs incluses).
- Inclusions imbriquées : jusqu’à 10 niveaux de profondeur.
- Chemins : résolus relativement au fichier incluant, mais doivent rester à l’intérieur du répertoire de configuration de niveau supérieur (`dirname` de `openclaw.json`). Les formes absolues/`../` sont autorisées uniquement si elles se résolvent toujours à l’intérieur de cette limite.
- Erreurs : messages clairs pour les fichiers manquants, les erreurs d’analyse et les inclusions circulaires.

---

_Related: [Configuration](/fr/gateway/configuration) · [Exemples de configuration](/fr/gateway/configuration-examples) · [Doctor](/fr/gateway/doctor)_
