---
read_when:
    - Configurer Matrix dans OpenClaw
    - Configurer E2EE et la vérification de Matrix
summary: État de la prise en charge de Matrix, configuration initiale et exemples de configuration
title: Matrix
x-i18n:
    generated_at: "2026-04-07T06:51:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: d53baa2ea5916cd00a99cae0ded3be41ffa13c9a69e8ea8461eb7baa6a99e13c
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix est le plugin de canal groupé Matrix pour OpenClaw.
Il utilise le `matrix-js-sdk` officiel et prend en charge les messages privés, les salons, les fils, les médias, les réactions, les sondages, la localisation et E2EE.

## Plugin groupé

Matrix est fourni comme plugin groupé dans les versions actuelles d'OpenClaw, donc les
builds packagés normaux n'ont pas besoin d'une installation séparée.

Si vous utilisez un ancien build ou une installation personnalisée qui exclut Matrix, installez-le
manuellement :

Installer depuis npm :

```bash
openclaw plugins install @openclaw/matrix
```

Installer depuis un dépôt local :

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Consultez [Plugins](/fr/tools/plugin) pour le comportement des plugins et les règles d'installation.

## Configuration initiale

1. Assurez-vous que le plugin Matrix est disponible.
   - Les versions packagées actuelles d'OpenClaw l'incluent déjà.
   - Les installations anciennes/personnalisées peuvent l'ajouter manuellement avec les commandes ci-dessus.
2. Créez un compte Matrix sur votre homeserver.
3. Configurez `channels.matrix` avec soit :
   - `homeserver` + `accessToken`, soit
   - `homeserver` + `userId` + `password`.
4. Redémarrez la passerelle.
5. Démarrez un message privé avec le bot ou invitez-le dans un salon.
   - Les nouvelles invitations Matrix ne fonctionnent que si `channels.matrix.autoJoin` les autorise.

Parcours de configuration interactifs :

```bash
openclaw channels add
openclaw configure --section channels
```

Ce que l'assistant Matrix demande réellement :

- URL du homeserver
- méthode d'authentification : jeton d'accès ou mot de passe
- ID utilisateur uniquement si vous choisissez l'authentification par mot de passe
- nom d'appareil facultatif
- activer ou non E2EE
- configurer ou non l'accès aux salons Matrix maintenant
- configurer ou non la jointure automatique aux invitations Matrix maintenant
- lorsque la jointure automatique aux invitations est activée, si elle doit être `allowlist`, `always` ou `off`

Comportement de l'assistant à connaître :

- Si des variables d'environnement d'authentification Matrix existent déjà pour le compte sélectionné, et que ce compte n'a pas déjà une authentification enregistrée dans la configuration, l'assistant propose un raccourci via les variables d'environnement afin que la configuration puisse conserver l'authentification dans les variables d'environnement au lieu de copier les secrets dans la configuration.
- Lorsque vous ajoutez un autre compte Matrix de façon interactive, le nom de compte saisi est normalisé en ID de compte utilisé dans la configuration et les variables d'environnement. Par exemple, `Ops Bot` devient `ops-bot`.
- Les invites d'allowlist pour message privé acceptent immédiatement des valeurs complètes `@user:server`. Les noms d'affichage ne fonctionnent que si la recherche en direct dans l'annuaire trouve une seule correspondance exacte ; sinon, l'assistant vous demande de réessayer avec un ID Matrix complet.
- Les invites d'allowlist de salon acceptent directement les ID et alias de salon. Elles peuvent aussi résoudre en direct les noms des salons rejoints, mais les noms non résolus ne sont conservés tels quels que pendant la configuration et sont ensuite ignorés par la résolution de l'allowlist à l'exécution. Préférez `!room:server` ou `#alias:server`.
- L'assistant affiche désormais un avertissement explicite avant l'étape de jointure automatique aux invitations, car `channels.matrix.autoJoin` vaut par défaut `off` ; les agents ne rejoindront pas les salons invités ni les nouvelles invitations de type message privé si vous ne le définissez pas.
- En mode allowlist pour la jointure automatique aux invitations, utilisez uniquement des cibles d'invitation stables : `!roomId:server`, `#alias:server` ou `*`. Les noms de salon simples sont rejetés.
- L'identité d'exécution de salon/session utilise l'ID de salon Matrix stable. Les alias déclarés par le salon ne sont utilisés que comme entrées de recherche, pas comme clé de session à long terme ni comme identité de groupe stable.
- Pour résoudre les noms de salon avant de les enregistrer, utilisez `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` vaut par défaut `off`.

Si vous le laissez non défini, le bot ne rejoindra pas les salons invités ni les nouvelles invitations de type message privé. Il n'apparaîtra donc pas dans les nouveaux groupes ou messages privés sur invitation, sauf si vous le faites rejoindre manuellement d'abord.

Définissez `autoJoin: "allowlist"` avec `autoJoinAllowlist` pour restreindre les invitations qu'il accepte, ou définissez `autoJoin: "always"` si vous voulez qu'il rejoigne chaque invitation.

En mode `allowlist`, `autoJoinAllowlist` accepte uniquement `!roomId:server`, `#alias:server` ou `*`.
</Warning>

Exemple d'allowlist :

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Rejoindre chaque invitation :

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

Configuration minimale basée sur un jeton :

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Configuration basée sur mot de passe (le jeton est mis en cache après connexion) :

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix stocke les identifiants mis en cache dans `~/.openclaw/credentials/matrix/`.
Le compte par défaut utilise `credentials.json` ; les comptes nommés utilisent `credentials-<account>.json`.
Lorsque des identifiants mis en cache existent à cet emplacement, OpenClaw considère Matrix comme configuré pour la configuration initiale, le diagnostic et la découverte de l'état des canaux, même si l'authentification actuelle n'est pas définie directement dans la configuration.

Équivalents en variables d'environnement (utilisés lorsque la clé de configuration n'est pas définie) :

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Pour les comptes non par défaut, utilisez des variables d'environnement ciblées par compte :

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Exemple pour le compte `ops` :

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Pour l'ID de compte normalisé `ops-bot`, utilisez :

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix échappe la ponctuation dans les ID de compte pour éviter les collisions dans les variables d'environnement ciblées.
Par exemple, `-` devient `_X2D_`, donc `ops-prod` est mappé vers `MATRIX_OPS_X2D_PROD_*`.

L'assistant interactif ne propose le raccourci via variables d'environnement que lorsque ces variables d'authentification sont déjà présentes et que le compte sélectionné n'a pas déjà une authentification Matrix enregistrée dans la configuration.

## Exemple de configuration

Voici une configuration de base pratique avec appairage en message privé, allowlist de salon et E2EE activé :

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin` s'applique aux invitations Matrix en général, et pas seulement aux invitations de salon/groupe.
Cela inclut les nouvelles invitations de type message privé. Au moment de l'invitation, OpenClaw ne sait pas de manière fiable si le
salon invité sera finalement traité comme un message privé ou un groupe, donc toutes les invitations passent d'abord par la même
décision `autoJoin`. `dm.policy` s'applique toujours une fois que le bot a rejoint le salon et qu'il est
classé comme message privé ; ainsi `autoJoin` contrôle le comportement de jointure tandis que `dm.policy` contrôle le comportement de réponse/d'accès.

## Aperçus en streaming

Le streaming des réponses Matrix est optionnel.

Définissez `channels.matrix.streaming` sur `"partial"` lorsque vous voulez qu'OpenClaw envoie un seul aperçu en direct
de réponse, modifie cet aperçu sur place pendant que le modèle génère le texte, puis le finalise une fois la
réponse terminée :

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` est la valeur par défaut. OpenClaw attend la réponse finale et l'envoie une seule fois.
- `streaming: "partial"` crée un message d'aperçu modifiable pour le bloc assistant actuel à l'aide de messages texte Matrix normaux. Cela préserve le comportement historique de notification « aperçu d'abord » de Matrix, donc les clients standard peuvent notifier sur le premier texte diffusé plutôt que sur le bloc terminé.
- `streaming: "quiet"` crée un aperçu discret modifiable pour le bloc assistant actuel. Utilisez-le uniquement si vous configurez aussi des règles push côté destinataire pour les modifications d'aperçu finalisées.
- `blockStreaming: true` active des messages de progression Matrix séparés. Avec le streaming d'aperçu activé, Matrix conserve le brouillon en direct pour le bloc actuel et préserve les blocs terminés comme messages séparés.
- Lorsque le streaming d'aperçu est activé et que `blockStreaming` est désactivé, Matrix modifie le brouillon en direct sur place et finalise ce même événement quand le bloc ou le tour se termine.
- Si l'aperçu ne peut plus tenir dans un seul événement Matrix, OpenClaw arrête le streaming d'aperçu et revient à une livraison finale normale.
- Les réponses multimédias envoient toujours les pièces jointes normalement. Si un aperçu obsolète ne peut plus être réutilisé en toute sécurité, OpenClaw le rédige avant d'envoyer la réponse multimédia finale.
- Les modifications d'aperçu coûtent des appels supplémentaires à l'API Matrix. Laissez le streaming désactivé si vous voulez le comportement le plus prudent vis-à-vis des limitations de débit.

`blockStreaming` n'active pas à lui seul les aperçus brouillon.
Utilisez `streaming: "partial"` ou `streaming: "quiet"` pour les modifications d'aperçu ; ajoutez ensuite `blockStreaming: true` uniquement si vous voulez également que les blocs assistant terminés restent visibles comme messages de progression séparés.

Si vous avez besoin de notifications Matrix standard sans règles push personnalisées, utilisez `streaming: "partial"` pour le comportement « aperçu d'abord » ou laissez `streaming` désactivé pour une livraison finale uniquement. Avec `streaming: "off"` :

- `blockStreaming: true` envoie chaque bloc terminé comme message Matrix normal avec notification.
- `blockStreaming: false` envoie uniquement la réponse finale terminée comme message Matrix normal avec notification.

### Règles push auto-hébergées pour des aperçus finalisés discrets

Si vous exploitez votre propre infrastructure Matrix et voulez que les aperçus discrets notifient uniquement lorsqu'un bloc ou la
réponse finale est terminé, définissez `streaming: "quiet"` et ajoutez une règle push par utilisateur pour les modifications d'aperçu finalisées.

Il s'agit généralement d'une configuration côté utilisateur destinataire, et non d'un changement global de configuration du homeserver :

Résumé rapide avant de commencer :

- utilisateur destinataire = la personne qui doit recevoir la notification
- utilisateur bot = le compte Matrix OpenClaw qui envoie la réponse
- utilisez le jeton d'accès de l'utilisateur destinataire pour les appels API ci-dessous
- faites correspondre `sender` dans la règle push à l'MXID complet de l'utilisateur bot

1. Configurez OpenClaw pour utiliser des aperçus discrets :

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Assurez-vous que le compte destinataire reçoit déjà les notifications push Matrix normales. Les règles
   d'aperçu discret ne fonctionnent que si cet utilisateur a déjà des pushers/appareils fonctionnels.

3. Récupérez le jeton d'accès de l'utilisateur destinataire.
   - Utilisez le jeton de l'utilisateur destinataire, pas celui du bot.
   - Réutiliser un jeton de session client existant est généralement le plus simple.
   - Si vous devez créer un nouveau jeton, vous pouvez vous connecter via l'API Client-Server Matrix standard :

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Vérifiez que le compte destinataire a déjà des pushers :

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Si cela ne renvoie aucun pusher/appareil actif, corrigez d'abord les notifications Matrix normales avant d'ajouter la
règle OpenClaw ci-dessous.

OpenClaw marque les modifications d'aperçu finalisées texte seul avec :

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Créez une règle push de surcharge pour chaque compte destinataire qui doit recevoir ces notifications :

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Remplacez ces valeurs avant d'exécuter la commande :

- `https://matrix.example.org` : l'URL de base de votre homeserver
- `$USER_ACCESS_TOKEN` : le jeton d'accès de l'utilisateur destinataire
- `openclaw-finalized-preview-botname` : un ID de règle unique à ce bot pour cet utilisateur destinataire
- `@bot:example.org` : l'MXID de votre bot Matrix OpenClaw, pas l'MXID de l'utilisateur destinataire

Important pour les configurations multi-bots :

- Les règles push sont indexées par `ruleId`. Relancer `PUT` sur le même ID de règle met à jour cette règle.
- Si un utilisateur destinataire doit recevoir des notifications de plusieurs comptes bot Matrix OpenClaw, créez une règle par bot avec un ID de règle unique pour chaque correspondance d'expéditeur.
- Un schéma simple est `openclaw-finalized-preview-<botname>`, par exemple `openclaw-finalized-preview-ops` ou `openclaw-finalized-preview-support`.

La règle est évaluée par rapport à l'expéditeur de l'événement :

- authentifiez-vous avec le jeton de l'utilisateur destinataire
- faites correspondre `sender` à l'MXID du bot OpenClaw

6. Vérifiez que la règle existe :

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Testez une réponse diffusée en streaming. En mode discret, le salon doit afficher un aperçu brouillon discret et la
   modification finale sur place doit notifier une fois le bloc ou le tour terminé.

Si vous devez supprimer la règle plus tard, supprimez ce même ID de règle avec le jeton de l'utilisateur destinataire :

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Remarques :

- Créez la règle avec le jeton d'accès de l'utilisateur destinataire, pas celui du bot.
- Les nouvelles règles `override` définies par l'utilisateur sont insérées avant les règles de suppression par défaut, donc aucun paramètre d'ordre supplémentaire n'est nécessaire.
- Cela n'affecte que les modifications d'aperçu texte seul qu'OpenClaw peut finaliser en place en toute sécurité. Les replis pour médias et les replis d'aperçu obsolète utilisent toujours la livraison Matrix normale.
- Si `GET /_matrix/client/v3/pushers` n'affiche aucun pusher, l'utilisateur n'a pas encore de livraison push Matrix fonctionnelle pour ce compte/appareil.

#### Synapse

Pour Synapse, la configuration ci-dessus est généralement suffisante à elle seule :

- Aucun changement spécial dans `homeserver.yaml` n'est requis pour les notifications d'aperçu OpenClaw finalisées.
- Si votre déploiement Synapse envoie déjà des notifications push Matrix normales, le jeton utilisateur + l'appel `pushrules` ci-dessus constituent l'étape principale de configuration.
- Si vous exécutez Synapse derrière un proxy inverse ou des workers, assurez-vous que `/_matrix/client/.../pushrules/` atteint correctement Synapse.
- Si vous utilisez des workers Synapse, assurez-vous que les pushers sont sains. La livraison push est gérée par le processus principal ou par `synapse.app.pusher` / les workers pusher configurés.

#### Tuwunel

Pour Tuwunel, utilisez le même flux de configuration et le même appel API `pushrules` que ci-dessus :

- Aucune configuration spécifique à Tuwunel n'est requise pour le marqueur d'aperçu finalisé lui-même.
- Si les notifications Matrix normales fonctionnent déjà pour cet utilisateur, le jeton utilisateur + l'appel `pushrules` ci-dessus constituent l'étape principale de configuration.
- Si les notifications semblent disparaître pendant que l'utilisateur est actif sur un autre appareil, vérifiez si `suppress_push_when_active` est activé. Tuwunel a ajouté cette option dans Tuwunel 1.4.2 le 12 septembre 2025, et elle peut supprimer intentionnellement les notifications push vers d'autres appareils lorsqu'un appareil est actif.

## Chiffrement et vérification

Dans les salons chiffrés (E2EE), les événements d'image sortants utilisent `thumbnail_file` afin que les aperçus d'image soient chiffrés avec la pièce jointe complète. Les salons non chiffrés utilisent toujours `thumbnail_url` en clair. Aucune configuration n'est nécessaire — le plugin détecte automatiquement l'état E2EE.

### Salons bot à bot

Par défaut, les messages Matrix provenant d'autres comptes Matrix OpenClaw configurés sont ignorés.

Utilisez `allowBots` lorsque vous voulez intentionnellement du trafic inter-agents Matrix :

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` accepte les messages d'autres comptes bot Matrix configurés dans les salons et messages privés autorisés.
- `allowBots: "mentions"` n'accepte ces messages que lorsqu'ils mentionnent visiblement ce bot dans les salons. Les messages privés restent autorisés.
- `groups.<room>.allowBots` remplace le paramètre au niveau du compte pour un salon.
- OpenClaw ignore toujours les messages provenant du même ID utilisateur Matrix afin d'éviter les boucles d'autoréponse.
- Matrix n'expose pas ici d'indicateur natif de bot ; OpenClaw considère « rédigé par un bot » comme « envoyé par un autre compte Matrix configuré sur cette passerelle OpenClaw ».

Utilisez des allowlists de salon strictes et des exigences de mention lorsque vous activez le trafic bot à bot dans des salons partagés.

Activer le chiffrement :

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Vérifier l'état de vérification :

```bash
openclaw matrix verify status
```

État détaillé (diagnostics complets) :

```bash
openclaw matrix verify status --verbose
```

Inclure la clé de récupération stockée dans la sortie lisible par machine :

```bash
openclaw matrix verify status --include-recovery-key --json
```

Initialiser l'état de cross-signing et de vérification :

```bash
openclaw matrix verify bootstrap
```

Prise en charge multi-comptes : utilisez `channels.matrix.accounts` avec des identifiants par compte et un `name` facultatif. Consultez [Référence de configuration](/fr/gateway/configuration-reference#multi-account-all-channels) pour le schéma partagé.

Diagnostics détaillés d'initialisation :

```bash
openclaw matrix verify bootstrap --verbose
```

Forcer une réinitialisation d'identité de cross-signing avant l'initialisation :

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Vérifier cet appareil avec une clé de récupération :

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Détails détaillés de vérification d'appareil :

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Vérifier l'état de santé de la sauvegarde des clés de salon :

```bash
openclaw matrix verify backup status
```

Diagnostics détaillés de santé de la sauvegarde :

```bash
openclaw matrix verify backup status --verbose
```

Restaurer les clés de salon depuis la sauvegarde serveur :

```bash
openclaw matrix verify backup restore
```

Diagnostics détaillés de restauration :

```bash
openclaw matrix verify backup restore --verbose
```

Supprimez la sauvegarde serveur actuelle et créez une nouvelle base de sauvegarde. Si la
clé de sauvegarde stockée ne peut pas être chargée proprement, cette réinitialisation peut aussi recréer le stockage secret afin que les
futurs démarrages à froid puissent charger la nouvelle clé de sauvegarde :

```bash
openclaw matrix verify backup reset --yes
```

Toutes les commandes `verify` sont concises par défaut (y compris la journalisation SDK interne discrète) et n'affichent des diagnostics détaillés qu'avec `--verbose`.
Utilisez `--json` pour une sortie complète lisible par machine dans les scripts.

Dans les configurations multi-comptes, les commandes CLI Matrix utilisent le compte Matrix par défaut implicite, sauf si vous passez `--account <id>`.
Si vous configurez plusieurs comptes nommés, définissez d'abord `channels.matrix.defaultAccount`, sinon ces opérations CLI implicites s'arrêteront et vous demanderont de choisir explicitement un compte.
Utilisez `--account` dès que vous voulez que les opérations de vérification ou d'appareil ciblent explicitement un compte nommé :

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Lorsque le chiffrement est désactivé ou indisponible pour un compte nommé, les avertissements Matrix et les erreurs de vérification pointent vers la clé de configuration de ce compte, par exemple `channels.matrix.accounts.assistant.encryption`.

### Ce que signifie « vérifié »

OpenClaw considère cet appareil Matrix comme vérifié uniquement lorsqu'il est vérifié par votre propre identité de cross-signing.
En pratique, `openclaw matrix verify status --verbose` expose trois signaux de confiance :

- `Locally trusted` : cet appareil n'est approuvé que par le client actuel
- `Cross-signing verified` : le SDK signale l'appareil comme vérifié via le cross-signing
- `Signed by owner` : l'appareil est signé par votre propre clé de self-signing

`Verified by owner` ne devient `yes` que lorsque la vérification par cross-signing ou la signature par le propriétaire est présente.
La confiance locale seule ne suffit pas pour qu'OpenClaw considère l'appareil comme entièrement vérifié.

### Ce que fait l'initialisation

`openclaw matrix verify bootstrap` est la commande de réparation et de configuration des comptes Matrix chiffrés.
Elle effectue toutes les opérations suivantes dans cet ordre :

- initialise le stockage secret, en réutilisant une clé de récupération existante lorsque possible
- initialise le cross-signing et téléverse les clés publiques de cross-signing manquantes
- tente de marquer et de signer par cross-signing l'appareil actuel
- crée une nouvelle sauvegarde côté serveur des clés de salon si elle n'existe pas déjà

Si le homeserver exige une authentification interactive pour téléverser des clés de cross-signing, OpenClaw tente d'abord le téléversement sans authentification, puis avec `m.login.dummy`, puis avec `m.login.password` lorsque `channels.matrix.password` est configuré.

Utilisez `--force-reset-cross-signing` uniquement si vous voulez explicitement supprimer l'identité de cross-signing actuelle et en créer une nouvelle.

Si vous voulez explicitement supprimer la sauvegarde actuelle des clés de salon et démarrer une nouvelle
base de sauvegarde pour les futurs messages, utilisez `openclaw matrix verify backup reset --yes`.
Faites-le uniquement si vous acceptez que l'ancien historique chiffré irrécupérable reste
indisponible et qu'OpenClaw puisse recréer le stockage secret si le secret de sauvegarde actuel ne peut pas être chargé de manière sûre.

### Nouvelle base de sauvegarde

Si vous voulez conserver le fonctionnement des futurs messages chiffrés et acceptez de perdre l'ancien historique irrécupérable, exécutez ces commandes dans l'ordre :

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Ajoutez `--account <id>` à chaque commande lorsque vous voulez cibler explicitement un compte Matrix nommé.

### Comportement au démarrage

Lorsque `encryption: true`, Matrix définit par défaut `startupVerification` sur `"if-unverified"`.
Au démarrage, si cet appareil n'est toujours pas vérifié, Matrix demandera une auto-vérification dans un autre client Matrix,
évitera les requêtes en double lorsqu'une requête est déjà en attente et appliquera un délai local avant de réessayer après les redémarrages.
Les tentatives de requête échouées réessaient plus tôt que la création réussie d'une requête par défaut.
Définissez `startupVerification: "off"` pour désactiver les requêtes automatiques au démarrage, ou ajustez `startupVerificationCooldownHours`
si vous voulez une fenêtre de nouvelle tentative plus courte ou plus longue.

Le démarrage effectue également automatiquement un passage conservateur d'initialisation crypto.
Ce passage essaie d'abord de réutiliser le stockage secret actuel et l'identité de cross-signing existante, et évite de réinitialiser le cross-signing sauf si vous exécutez un flux explicite de réparation par initialisation.

Si le démarrage détecte un état d'initialisation défectueux et que `channels.matrix.password` est configuré, OpenClaw peut tenter un chemin de réparation plus strict.
Si l'appareil actuel est déjà signé par le propriétaire, OpenClaw préserve cette identité au lieu de la réinitialiser automatiquement.

Mise à niveau depuis l'ancien plugin public Matrix :

- OpenClaw réutilise automatiquement le même compte Matrix, jeton d'accès et identité d'appareil lorsque c'est possible.
- Avant l'exécution de toute modification de migration Matrix exploitable, OpenClaw crée ou réutilise un instantané de récupération sous `~/Backups/openclaw-migrations/`.
- Si vous utilisez plusieurs comptes Matrix, définissez `channels.matrix.defaultAccount` avant la mise à niveau depuis l'ancien stockage à plat afin qu'OpenClaw sache quel compte doit recevoir cet état hérité partagé.
- Si l'ancien plugin stockait localement une clé de déchiffrement de sauvegarde des clés de salon Matrix, le démarrage ou `openclaw doctor --fix` l'importera automatiquement dans le nouveau flux de clé de récupération.
- Si le jeton d'accès Matrix a changé après la préparation de la migration, le démarrage analyse désormais les racines de stockage sœurs basées sur le hachage du jeton à la recherche d'un état de restauration hérité en attente avant d'abandonner la restauration automatique de la sauvegarde.
- Si le jeton d'accès Matrix change plus tard pour le même compte, homeserver et utilisateur, OpenClaw préfère désormais réutiliser la racine de stockage basée sur le hachage du jeton la plus complète existante plutôt que de repartir d'un répertoire d'état Matrix vide.
- Au prochain démarrage de la passerelle, les clés de salon sauvegardées sont restaurées automatiquement dans le nouveau magasin crypto.
- Si l'ancien plugin possédait des clés de salon uniquement locales qui n'ont jamais été sauvegardées, OpenClaw émettra un avertissement clair. Ces clés ne peuvent pas être exportées automatiquement depuis l'ancien magasin crypto Rust, donc une partie de l'ancien historique chiffré peut rester indisponible jusqu'à une récupération manuelle.
- Consultez [Migration Matrix](/fr/install/migrating-matrix) pour le flux complet de mise à niveau, les limites, les commandes de récupération et les messages de migration courants.

L'état chiffré à l'exécution est organisé sous des racines par compte, par utilisateur et par hachage de jeton dans
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Ce répertoire contient le magasin de synchronisation (`bot-storage.json`), le magasin crypto (`crypto/`),
le fichier de clé de récupération (`recovery-key.json`), l'instantané IndexedDB (`crypto-idb-snapshot.json`),
les liaisons de fils (`thread-bindings.json`) et l'état de vérification au démarrage (`startup-verification.json`)
lorsque ces fonctionnalités sont utilisées.
Lorsque le jeton change mais que l'identité du compte reste la même, OpenClaw réutilise la meilleure
racine existante pour ce tuple compte/homeserver/utilisateur afin que l'état de synchronisation précédent, l'état crypto, les liaisons de fils
et l'état de vérification au démarrage restent disponibles.

### Modèle de magasin crypto Node

Le chiffrement E2EE Matrix dans ce plugin utilise le chemin crypto Rust officiel du `matrix-js-sdk` dans Node.
Ce chemin attend une persistance basée sur IndexedDB lorsque vous voulez que l'état crypto survive aux redémarrages.

OpenClaw fournit actuellement cela dans Node en :

- utilisant `fake-indexeddb` comme shim API IndexedDB attendu par le SDK
- restaurant le contenu IndexedDB crypto Rust depuis `crypto-idb-snapshot.json` avant `initRustCrypto`
- persistant le contenu IndexedDB mis à jour dans `crypto-idb-snapshot.json` après l'initialisation et pendant l'exécution
- sérialisant la restauration et la persistance de l'instantané sur `crypto-idb-snapshot.json` avec un verrou de fichier consultatif afin d'éviter que la persistance à l'exécution de la passerelle et la maintenance CLI ne se concurrencent sur le même fichier d'instantané

Il s'agit d'une plomberie de compatibilité/stockage, pas d'une implémentation crypto personnalisée.
Le fichier d'instantané est un état d'exécution sensible et est stocké avec des permissions de fichier restrictives.
Dans le modèle de sécurité d'OpenClaw, l'hôte de la passerelle et le répertoire d'état local OpenClaw se trouvent déjà à l'intérieur de la frontière de confiance de l'opérateur ; il s'agit donc principalement d'un enjeu de durabilité opérationnelle plutôt que d'une frontière de confiance distante distincte.

Amélioration prévue :

- ajouter la prise en charge de SecretRef pour le matériel de clé Matrix persistant afin que les clés de récupération et les secrets de chiffrement de magasin associés puissent provenir des fournisseurs de secrets OpenClaw au lieu de fichiers locaux uniquement

## Gestion du profil

Mettez à jour le profil Matrix du compte sélectionné avec :

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Ajoutez `--account <id>` si vous voulez cibler explicitement un compte Matrix nommé.

Matrix accepte directement les URL d'avatar `mxc://`. Lorsque vous passez une URL d'avatar `http://` ou `https://`, OpenClaw la téléverse d'abord dans Matrix puis enregistre l'URL `mxc://` résolue dans `channels.matrix.avatarUrl` (ou dans la surcharge du compte sélectionné).

## Notifications automatiques de vérification

Matrix publie désormais directement des notifications du cycle de vie de la vérification dans le salon de message privé strict de vérification sous forme de messages `m.notice`.
Cela inclut :

- les notifications de demande de vérification
- les notifications indiquant que la vérification est prête (avec des instructions explicites « Vérifier par emoji »)
- les notifications de début et de fin de vérification
- les détails SAS (emoji et décimal) lorsqu'ils sont disponibles

Les demandes de vérification entrantes provenant d'un autre client Matrix sont suivies et automatiquement acceptées par OpenClaw.
Pour les flux d'auto-vérification, OpenClaw démarre également automatiquement le flux SAS lorsque la vérification par emoji devient disponible et confirme son propre côté.
Pour les demandes de vérification provenant d'un autre utilisateur/appareil Matrix, OpenClaw accepte automatiquement la demande puis attend que le flux SAS se poursuive normalement.
Vous devez toujours comparer les emoji ou le SAS décimal dans votre client Matrix et confirmer « Ils correspondent » pour terminer la vérification.

OpenClaw n'accepte pas aveuglément automatiquement les flux en double auto-initiés. Le démarrage ignore la création d'une nouvelle demande lorsqu'une demande d'auto-vérification est déjà en attente.

Les notifications protocole/système de vérification ne sont pas transmises au pipeline de discussion de l'agent, donc elles ne produisent pas `NO_REPLY`.

### Hygiène des appareils

Les anciens appareils Matrix gérés par OpenClaw peuvent s'accumuler sur le compte et compliquer la compréhension de la confiance dans les salons chiffrés.
Listez-les avec :

```bash
openclaw matrix devices list
```

Supprimez les appareils OpenClaw obsolètes avec :

```bash
openclaw matrix devices prune-stale
```

### Réparation de salon direct

Si l'état des messages directs se désynchronise, OpenClaw peut se retrouver avec des mappages `m.direct` obsolètes pointant vers d'anciens salons solo au lieu du message privé actif. Inspectez le mappage actuel pour un pair avec :

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Réparez-le avec :

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

La réparation conserve la logique spécifique à Matrix dans le plugin :

- elle préfère un message privé strict 1:1 déjà mappé dans `m.direct`
- sinon elle se rabat sur n'importe quel message privé strict 1:1 actuellement rejoint avec cet utilisateur
- si aucun message privé sain n'existe, elle crée un nouveau salon direct et réécrit `m.direct` pour pointer vers celui-ci

Le flux de réparation ne supprime pas automatiquement les anciens salons. Il choisit uniquement le message privé sain et met à jour le mappage afin que les nouveaux envois Matrix, les notifications de vérification et les autres flux de messages directs ciblent de nouveau le bon salon.

## Fils

Matrix prend en charge les fils Matrix natifs à la fois pour les réponses automatiques et pour les envois avec l'outil de message.

- `dm.sessionScope: "per-user"` (par défaut) conserve le routage des messages privés Matrix au niveau de l'expéditeur, afin que plusieurs salons de message privé puissent partager une session lorsqu'ils se résolvent vers le même pair.
- `dm.sessionScope: "per-room"` isole chaque salon de message privé Matrix dans sa propre clé de session tout en utilisant les contrôles normaux d'authentification et d'allowlist des messages privés.
- Les liaisons explicites de conversation Matrix priment toujours sur `dm.sessionScope`, de sorte que les salons et fils liés conservent leur session cible choisie.
- `threadReplies: "off"` conserve les réponses au niveau supérieur et maintient les messages entrants en fil sur la session parente.
- `threadReplies: "inbound"` répond dans un fil uniquement lorsque le message entrant était déjà dans ce fil.
- `threadReplies: "always"` conserve les réponses de salon dans un fil enraciné sur le message déclencheur et achemine cette conversation via la session ciblée par fil correspondante à partir du premier message déclencheur.
- `dm.threadReplies` remplace le paramètre de niveau supérieur pour les messages privés uniquement. Par exemple, vous pouvez isoler les fils de salon tout en gardant les messages privés à plat.
- Les messages entrants en fil incluent le message racine du fil comme contexte supplémentaire pour l'agent.
- Les envois via l'outil de message héritent désormais automatiquement du fil Matrix actuel lorsque la cible est le même salon, ou la même cible utilisateur en message privé, sauf si un `threadId` explicite est fourni.
- La réutilisation de la même cible utilisateur de message privé pour la même session n'intervient que lorsque les métadonnées de la session actuelle prouvent le même pair de message privé sur le même compte Matrix ; sinon OpenClaw revient au routage normal à portée utilisateur.
- Lorsque OpenClaw voit un salon de message privé Matrix entrer en collision avec un autre salon de message privé sur la même session de message privé Matrix partagée, il publie une seule fois un `m.notice` dans ce salon avec l'échappatoire `/focus` lorsque les liaisons de fils sont activées et l'indication `dm.sessionScope`.
- Les liaisons de fils à l'exécution sont prises en charge pour Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` et `/acp spawn` lié à un fil fonctionnent désormais dans les salons et messages privés Matrix.
- Le `/focus` de salon/message privé Matrix au niveau supérieur crée un nouveau fil Matrix et le lie à la session cible lorsque `threadBindings.spawnSubagentSessions=true`.
- L'exécution de `/focus` ou `/acp spawn --thread here` à l'intérieur d'un fil Matrix existant lie plutôt ce fil actuel.

## Liaisons de conversation ACP

Les salons, messages privés et fils Matrix existants peuvent être transformés en espaces de travail ACP durables sans changer la surface de discussion.

Flux opérateur rapide :

- Exécutez `/acp spawn codex --bind here` dans le message privé, salon ou fil existant Matrix que vous voulez continuer à utiliser.
- Dans un message privé ou salon Matrix de niveau supérieur, le message privé/salon actuel reste la surface de discussion et les futurs messages sont acheminés vers la session ACP créée.
- À l'intérieur d'un fil Matrix existant, `--bind here` lie ce fil actuel sur place.
- `/new` et `/reset` réinitialisent sur place la même session ACP liée.
- `/acp close` ferme la session ACP et supprime la liaison.

Remarques :

- `--bind here` ne crée pas de fil Matrix enfant.
- `threadBindings.spawnAcpSessions` n'est requis que pour `/acp spawn --thread auto|here`, quand OpenClaw doit créer ou lier un fil Matrix enfant.

### Configuration des liaisons de fils

Matrix hérite des valeurs par défaut globales depuis `session.threadBindings` et prend aussi en charge des surcharges par canal :

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Les indicateurs de création liés à des fils Matrix sont optionnels :

- Définissez `threadBindings.spawnSubagentSessions: true` pour permettre à `/focus` de niveau supérieur de créer et lier de nouveaux fils Matrix.
- Définissez `threadBindings.spawnAcpSessions: true` pour permettre à `/acp spawn --thread auto|here` de lier des sessions ACP à des fils Matrix.

## Réactions

Matrix prend en charge les actions de réaction sortantes, les notifications de réaction entrantes et les réactions d'accusé de réception entrantes.

- L'outillage de réaction sortante est contrôlé par `channels["matrix"].actions.reactions`.
- `react` ajoute une réaction à un événement Matrix spécifique.
- `reactions` liste le résumé actuel des réactions pour un événement Matrix spécifique.
- `emoji=""` supprime les propres réactions du compte bot sur cet événement.
- `remove: true` supprime uniquement la réaction emoji spécifiée du compte bot.

La portée des réactions d'accusé de réception est résolue dans cet ordre :

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- repli vers l'emoji d'identité de l'agent

La portée de la réaction d'accusé de réception se résout dans cet ordre :

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Le mode de notification de réaction se résout dans cet ordre :

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- défaut : `own`

Comportement actuel :

- `reactionNotifications: "own"` transmet les événements `m.reaction` ajoutés lorsqu'ils ciblent des messages Matrix rédigés par le bot.
- `reactionNotifications: "off"` désactive les événements système de réaction.
- Les suppressions de réaction ne sont toujours pas synthétisées en événements système, car Matrix les expose comme des rédactions et non comme des suppressions autonomes de `m.reaction`.

## Contexte d'historique

- `channels.matrix.historyLimit` contrôle combien de messages récents du salon sont inclus comme `InboundHistory` lorsqu'un message de salon Matrix déclenche l'agent.
- Cette valeur se rabat sur `messages.groupChat.historyLimit`. Si les deux ne sont pas définies, la valeur par défaut effective est `0`, donc les messages de salon avec exigence de mention ne sont pas mis en tampon. Définissez `0` pour désactiver.
- L'historique de salon Matrix est limité au salon. Les messages privés continuent d'utiliser l'historique normal de session.
- L'historique de salon Matrix est limité aux éléments en attente : OpenClaw met en tampon les messages de salon qui n'ont pas encore déclenché de réponse, puis prend un instantané de cette fenêtre lorsqu'une mention ou un autre déclencheur arrive.
- Le message déclencheur actuel n'est pas inclus dans `InboundHistory` ; il reste dans le corps entrant principal pour ce tour.
- Les nouvelles tentatives sur le même événement Matrix réutilisent l'instantané d'historique d'origine au lieu de dériver vers les nouveaux messages du salon.

## Visibilité du contexte

Matrix prend en charge le contrôle partagé `contextVisibility` pour le contexte de salon supplémentaire, tel que le texte de réponse récupéré, les racines de fil et l'historique en attente.

- `contextVisibility: "all"` est la valeur par défaut. Le contexte supplémentaire est conservé tel que reçu.
- `contextVisibility: "allowlist"` filtre le contexte supplémentaire aux expéditeurs autorisés par les vérifications d'allowlist actives du salon/de l'utilisateur.
- `contextVisibility: "allowlist_quote"` se comporte comme `allowlist`, mais conserve quand même une réponse citée explicite.

Ce paramètre affecte la visibilité du contexte supplémentaire, pas la possibilité pour le message entrant lui-même de déclencher une réponse.
L'autorisation de déclenchement provient toujours de `groupPolicy`, `groups`, `groupAllowFrom` et des paramètres de politique des messages privés.

## Exemple de politique des messages privés et salons

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Consultez [Groups](/fr/channels/groups) pour le comportement de contrainte par mention et d'allowlist.

Exemple d'appairage pour les messages privés Matrix :

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Si un utilisateur Matrix non approuvé continue à vous envoyer des messages avant approbation, OpenClaw réutilise le même code d'appairage en attente et peut renvoyer une réponse de rappel après un court délai au lieu de générer un nouveau code.

Consultez [Pairing](/fr/channels/pairing) pour le flux partagé d'appairage des messages privés et la structure de stockage.

## Approbations exec

Matrix peut agir comme client d'approbation exec pour un compte Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (facultatif ; se rabat sur `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, par défaut : `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Les approbateurs doivent être des ID utilisateur Matrix comme `@owner:example.org`. Matrix active automatiquement les approbations exec natives quand `enabled` n'est pas défini ou vaut `"auto"` et qu'au moins un approbateur peut être résolu, soit depuis `execApprovals.approvers`, soit depuis `channels.matrix.dm.allowFrom`. Définissez `enabled: false` pour désactiver explicitement Matrix comme client d'approbation natif. Sinon, les demandes d'approbation se rabattent sur d'autres routes d'approbation configurées ou sur la politique de repli d'approbation exec.

Le routage natif Matrix est aujourd'hui réservé à exec :

- `channels.matrix.execApprovals.*` contrôle le routage natif message privé/canal pour les approbations exec uniquement.
- Les approbations de plugin utilisent toujours le `/approve` partagé dans la même discussion ainsi que tout transfert `approvals.plugin` configuré.
- Matrix peut toujours réutiliser `channels.matrix.dm.allowFrom` pour l'autorisation d'approbation de plugin lorsqu'il peut déduire les approbateurs en toute sécurité, mais il n'expose pas de chemin natif séparé de diffusion en éventail message privé/canal pour les approbations de plugin.

Règles de livraison :

- `target: "dm"` envoie les invites d'approbation dans les messages privés des approbateurs
- `target: "channel"` renvoie l'invite dans le salon ou message privé Matrix d'origine
- `target: "both"` envoie aux messages privés des approbateurs et au salon ou message privé Matrix d'origine

Les invites d'approbation Matrix initialisent des raccourcis de réaction sur le message principal d'approbation :

- `✅` = autoriser une fois
- `❌` = refuser
- `♾️` = toujours autoriser lorsque cette décision est permise par la politique exec effective

Les approbateurs peuvent réagir sur ce message ou utiliser les commandes slash de repli : `/approve <id> allow-once`, `/approve <id> allow-always` ou `/approve <id> deny`.

Seuls les approbateurs résolus peuvent approuver ou refuser. La livraison dans le canal inclut le texte de commande ; n'activez donc `channel` ou `both` que dans des salons de confiance.

Les invites d'approbation Matrix réutilisent le planificateur partagé d'approbation du cœur. La surface native spécifique à Matrix n'est qu'un transport pour les approbations exec : routage salon/message privé et comportement d'envoi/mise à jour/suppression des messages.

Surcharge par compte :

- `channels.matrix.accounts.<account>.execApprovals`

Documentation connexe : [Approbations exec](/fr/tools/exec-approvals)

## Exemple multi-comptes

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Les valeurs de niveau supérieur `channels.matrix` servent de valeurs par défaut pour les comptes nommés, sauf si un compte les remplace.
Vous pouvez limiter une entrée de salon héritée à un compte Matrix avec `groups.<room>.account` (ou l'ancien `rooms.<room>.account`).
Les entrées sans `account` restent partagées entre tous les comptes Matrix, et les entrées avec `account: "default"` fonctionnent toujours lorsque le compte par défaut est configuré directement au niveau supérieur dans `channels.matrix.*`.
Les valeurs d'authentification partagées partielles par défaut ne créent pas à elles seules un compte implicite par défaut distinct. OpenClaw ne synthétise le compte `default` de niveau supérieur que lorsque ce défaut dispose d'une authentification récente (`homeserver` plus `accessToken`, ou `homeserver` plus `userId` et `password`) ; les comptes nommés peuvent toujours rester détectables à partir de `homeserver` plus `userId` lorsque des identifiants mis en cache satisfont ultérieurement l'authentification.
Si Matrix possède déjà exactement un compte nommé, ou si `defaultAccount` pointe vers une clé de compte nommé existante, la promotion de réparation/configuration d'un seul compte vers plusieurs comptes préserve ce compte au lieu de créer une nouvelle entrée `accounts.default`. Seules les clés d'authentification/d'initialisation Matrix sont déplacées dans ce compte promu ; les clés partagées de politique de livraison restent au niveau supérieur.
Définissez `defaultAccount` lorsque vous voulez qu'OpenClaw privilégie un compte Matrix nommé pour le routage implicite, le sondage et les opérations CLI.
Si vous configurez plusieurs comptes nommés, définissez `defaultAccount` ou passez `--account <id>` pour les commandes CLI qui dépendent d'une sélection implicite de compte.
Passez `--account <id>` à `openclaw matrix verify ...` et `openclaw matrix devices ...` lorsque vous voulez remplacer cette sélection implicite pour une commande.

## Homeservers privés/LAN

Par défaut, OpenClaw bloque les homeservers Matrix privés/internes pour la protection SSRF, sauf si vous
activez explicitement cette possibilité compte par compte.

Si votre homeserver s'exécute sur localhost, une IP LAN/Tailscale ou un nom d'hôte interne, activez
`network.dangerouslyAllowPrivateNetwork` pour ce compte Matrix :

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Exemple de configuration CLI :

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Cette option n'autorise que les cibles privées/internes de confiance. Les homeservers publics en clair tels que
`http://matrix.example.org:8008` restent bloqués. Préférez `https://` lorsque c'est possible.

## Proxy du trafic Matrix

Si votre déploiement Matrix nécessite un proxy HTTP(S) sortant explicite, définissez `channels.matrix.proxy` :

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Les comptes nommés peuvent remplacer la valeur par défaut de niveau supérieur avec `channels.matrix.accounts.<id>.proxy`.
OpenClaw utilise le même paramètre de proxy pour le trafic Matrix à l'exécution et les sondes d'état du compte.

## Résolution des cibles

Matrix accepte ces formes de cible partout où OpenClaw vous demande une cible de salon ou d'utilisateur :

- Utilisateurs : `@user:server`, `user:@user:server` ou `matrix:user:@user:server`
- Salons : `!room:server`, `room:!room:server` ou `matrix:room:!room:server`
- Alias : `#alias:server`, `channel:#alias:server` ou `matrix:channel:#alias:server`

La recherche en direct dans l'annuaire utilise le compte Matrix connecté :

- Les recherches d'utilisateurs interrogent l'annuaire utilisateur Matrix sur ce homeserver.
- Les recherches de salons acceptent directement les ID et alias explicites de salon, puis se rabattent sur la recherche parmi les noms des salons rejoints pour ce compte.
- La recherche par nom de salon rejoint est au mieux. Si un nom de salon ne peut pas être résolu en ID ou alias, il est ignoré par la résolution de l'allowlist à l'exécution.

## Référence de configuration

- `enabled` : activer ou désactiver le canal.
- `name` : étiquette facultative pour le compte.
- `defaultAccount` : ID de compte préféré lorsque plusieurs comptes Matrix sont configurés.
- `homeserver` : URL du homeserver, par exemple `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork` : autoriser ce compte Matrix à se connecter à des homeservers privés/internes. Activez-le lorsque le homeserver se résout vers `localhost`, une IP LAN/Tailscale ou un hôte interne tel que `matrix-synapse`.
- `proxy` : URL facultative de proxy HTTP(S) pour le trafic Matrix. Les comptes nommés peuvent remplacer la valeur par défaut de niveau supérieur avec leur propre `proxy`.
- `userId` : ID utilisateur Matrix complet, par exemple `@bot:example.org`.
- `accessToken` : jeton d'accès pour l'authentification par jeton. Les valeurs en clair et les valeurs SecretRef sont prises en charge pour `channels.matrix.accessToken` et `channels.matrix.accounts.<id>.accessToken` via les fournisseurs env/file/exec. Consultez [Gestion des secrets](/fr/gateway/secrets).
- `password` : mot de passe pour la connexion par mot de passe. Les valeurs en clair et les valeurs SecretRef sont prises en charge.
- `deviceId` : ID explicite d'appareil Matrix.
- `deviceName` : nom d'affichage de l'appareil pour la connexion par mot de passe.
- `avatarUrl` : URL d'avatar personnel stockée pour la synchronisation du profil et les mises à jour `set-profile`.
- `initialSyncLimit` : limite d'événements de synchronisation au démarrage.
- `encryption` : activer E2EE.
- `allowlistOnly` : forcer un comportement allowlist uniquement pour les messages privés et salons.
- `allowBots` : autoriser les messages d'autres comptes Matrix OpenClaw configurés (`true` ou `"mentions"`).
- `groupPolicy` : `open`, `allowlist` ou `disabled`.
- `contextVisibility` : mode de visibilité du contexte supplémentaire de salon (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom` : allowlist des ID utilisateur pour le trafic de salon.
- Les entrées `groupAllowFrom` doivent être des ID utilisateur Matrix complets. Les noms non résolus sont ignorés à l'exécution.
- `historyLimit` : nombre maximal de messages de salon à inclure comme contexte d'historique de groupe. Se rabat sur `messages.groupChat.historyLimit` ; si les deux ne sont pas définis, la valeur par défaut effective est `0`. Définissez `0` pour désactiver.
- `replyToMode` : `off`, `first`, `all` ou `batched`.
- `markdown` : configuration facultative de rendu Markdown pour le texte Matrix sortant.
- `streaming` : `off` (par défaut), `partial`, `quiet`, `true` ou `false`. `partial` et `true` activent des mises à jour de brouillon avec aperçu d'abord à l'aide de messages texte Matrix normaux. `quiet` utilise des aperçus discrets sans notification pour les configurations auto-hébergées avec règles push.
- `blockStreaming` : `true` active des messages de progression séparés pour les blocs assistant terminés pendant que le streaming d'aperçu brouillon est actif.
- `threadReplies` : `off`, `inbound` ou `always`.
- `threadBindings` : surcharges par canal pour le routage et le cycle de vie des sessions liées à un fil.
- `startupVerification` : mode automatique de demande d'auto-vérification au démarrage (`if-unverified`, `off`).
- `startupVerificationCooldownHours` : délai avant nouvelle tentative des demandes automatiques de vérification au démarrage.
- `textChunkLimit` : taille des blocs de message sortant.
- `chunkMode` : `length` ou `newline`.
- `responsePrefix` : préfixe de message facultatif pour les réponses sortantes.
- `ackReaction` : surcharge facultative de réaction d'accusé de réception pour ce canal/compte.
- `ackReactionScope` : surcharge facultative de portée de réaction d'accusé de réception (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications` : mode de notification de réaction entrante (`own`, `off`).
- `mediaMaxMb` : limite de taille des médias en Mo pour la gestion des médias Matrix. Elle s'applique aux envois sortants et au traitement des médias entrants.
- `autoJoin` : politique de jointure automatique aux invitations (`always`, `allowlist`, `off`). Par défaut : `off`. Cela s'applique aux invitations Matrix en général, y compris aux invitations de type message privé, et pas seulement aux invitations de salon/groupe. OpenClaw prend cette décision au moment de l'invitation, avant de pouvoir classifier de manière fiable le salon rejoint comme message privé ou groupe.
- `autoJoinAllowlist` : salons/alias autorisés lorsque `autoJoin` est `allowlist`. Les entrées d'alias sont résolues en ID de salon pendant la gestion des invitations ; OpenClaw ne fait pas confiance à l'état d'alias revendiqué par le salon invité.
- `dm` : bloc de politique des messages privés (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy` : contrôle l'accès aux messages privés après qu'OpenClaw a rejoint le salon et l'a classé comme message privé. Cela ne change pas le fait qu'une invitation soit rejointe automatiquement.
- Les entrées `dm.allowFrom` doivent être des ID utilisateur Matrix complets, sauf si vous les avez déjà résolues via une recherche en direct dans l'annuaire.
- `dm.sessionScope` : `per-user` (par défaut) ou `per-room`. Utilisez `per-room` si vous voulez que chaque salon de message privé Matrix conserve un contexte séparé même si le pair est le même.
- `dm.threadReplies` : surcharge de politique de fil pour les messages privés uniquement (`off`, `inbound`, `always`). Elle remplace le paramètre `threadReplies` de niveau supérieur à la fois pour le placement des réponses et pour l'isolation de session dans les messages privés.
- `execApprovals` : livraison native Matrix des approbations exec (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers` : ID utilisateur Matrix autorisés à approuver les demandes exec. Facultatif lorsque `dm.allowFrom` identifie déjà les approbateurs.
- `execApprovals.target` : `dm | channel | both` (par défaut : `dm`).
- `accounts` : surcharges nommées par compte. Les valeurs de niveau supérieur `channels.matrix` servent de valeurs par défaut à ces entrées.
- `groups` : mappage de politique par salon. Préférez les ID ou alias de salon ; les noms de salon non résolus sont ignorés à l'exécution. L'identité de session/groupe utilise l'ID de salon stable après résolution, tandis que les libellés lisibles par un humain proviennent toujours des noms de salon.
- `groups.<room>.account` : restreindre une entrée de salon héritée à un compte Matrix spécifique dans les configurations multi-comptes.
- `groups.<room>.allowBots` : surcharge au niveau du salon pour les expéditeurs bot configurés (`true` ou `"mentions"`).
- `groups.<room>.users` : allowlist des expéditeurs par salon.
- `groups.<room>.tools` : surcharges d'autorisation/refus d'outils par salon.
- `groups.<room>.autoReply` : surcharge de contrainte par mention au niveau du salon. `true` désactive l'exigence de mention pour ce salon ; `false` la réactive de force.
- `groups.<room>.skills` : filtre facultatif de Skills au niveau du salon.
- `groups.<room>.systemPrompt` : extrait facultatif de prompt système au niveau du salon.
- `rooms` : alias hérité de `groups`.
- `actions` : contrôle par action des outils (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Voir aussi

- [Vue d'ensemble des canaux](/fr/channels) — tous les canaux pris en charge
- [Pairing](/fr/channels/pairing) — authentification des messages privés et flux d'appairage
- [Groups](/fr/channels/groups) — comportement de discussion de groupe et contrainte par mention
- [Routage des canaux](/fr/channels/channel-routing) — routage de session pour les messages
- [Sécurité](/fr/gateway/security) — modèle d'accès et durcissement
