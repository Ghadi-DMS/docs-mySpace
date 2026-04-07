---
read_when:
    - Vérifier la couverture des identifiants SecretRef
    - Vérifier si un identifiant est éligible à `secrets configure` ou `secrets apply`
    - Vérifier pourquoi un identifiant est en dehors de la surface prise en charge
summary: Surface canonique des identifiants SecretRef pris en charge et non pris en charge
title: Surface des identifiants SecretRef
x-i18n:
    generated_at: "2026-04-07T06:53:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 211f4b504c5808f7790683066fc2c8b700c705c598f220a264daf971b81cc593
    source_path: reference/secretref-credential-surface.md
    workflow: 15
---

# Surface des identifiants SecretRef

Cette page définit la surface canonique des identifiants SecretRef.

Portée visée :

- Dans le périmètre : identifiants strictement fournis par l’utilisateur qu’OpenClaw ne génère ni ne fait tourner.
- Hors périmètre : identifiants générés ou renouvelés à l’exécution, matériel de rafraîchissement OAuth et artefacts de type session.

## Identifiants pris en charge

### Cibles `openclaw.json` (`secrets configure` + `secrets apply` + `secrets audit`)

[//]: # "secretref-supported-list-start"

- `models.providers.*.apiKey`
- `models.providers.*.headers.*`
- `models.providers.*.request.auth.token`
- `models.providers.*.request.auth.value`
- `models.providers.*.request.headers.*`
- `models.providers.*.request.proxy.tls.ca`
- `models.providers.*.request.proxy.tls.cert`
- `models.providers.*.request.proxy.tls.key`
- `models.providers.*.request.proxy.tls.passphrase`
- `models.providers.*.request.tls.ca`
- `models.providers.*.request.tls.cert`
- `models.providers.*.request.tls.key`
- `models.providers.*.request.tls.passphrase`
- `skills.entries.*.apiKey`
- `agents.defaults.memorySearch.remote.apiKey`
- `agents.list[].memorySearch.remote.apiKey`
- `talk.providers.*.apiKey`
- `messages.tts.providers.*.apiKey`
- `tools.web.fetch.firecrawl.apiKey`
- `plugins.entries.brave.config.webSearch.apiKey`
- `plugins.entries.google.config.webSearch.apiKey`
- `plugins.entries.xai.config.webSearch.apiKey`
- `plugins.entries.moonshot.config.webSearch.apiKey`
- `plugins.entries.perplexity.config.webSearch.apiKey`
- `plugins.entries.firecrawl.config.webSearch.apiKey`
- `plugins.entries.minimax.config.webSearch.apiKey`
- `plugins.entries.tavily.config.webSearch.apiKey`
- `tools.web.search.apiKey`
- `gateway.auth.password`
- `gateway.auth.token`
- `gateway.remote.token`
- `gateway.remote.password`
- `cron.webhookToken`
- `channels.telegram.botToken`
- `channels.telegram.webhookSecret`
- `channels.telegram.accounts.*.botToken`
- `channels.telegram.accounts.*.webhookSecret`
- `channels.slack.botToken`
- `channels.slack.appToken`
- `channels.slack.userToken`
- `channels.slack.signingSecret`
- `channels.slack.accounts.*.botToken`
- `channels.slack.accounts.*.appToken`
- `channels.slack.accounts.*.userToken`
- `channels.slack.accounts.*.signingSecret`
- `channels.discord.token`
- `channels.discord.pluralkit.token`
- `channels.discord.voice.tts.providers.*.apiKey`
- `channels.discord.accounts.*.token`
- `channels.discord.accounts.*.pluralkit.token`
- `channels.discord.accounts.*.voice.tts.providers.*.apiKey`
- `channels.irc.password`
- `channels.irc.nickserv.password`
- `channels.irc.accounts.*.password`
- `channels.irc.accounts.*.nickserv.password`
- `channels.bluebubbles.password`
- `channels.bluebubbles.accounts.*.password`
- `channels.feishu.appSecret`
- `channels.feishu.encryptKey`
- `channels.feishu.verificationToken`
- `channels.feishu.accounts.*.appSecret`
- `channels.feishu.accounts.*.encryptKey`
- `channels.feishu.accounts.*.verificationToken`
- `channels.msteams.appPassword`
- `channels.mattermost.botToken`
- `channels.mattermost.accounts.*.botToken`
- `channels.matrix.accessToken`
- `channels.matrix.password`
- `channels.matrix.accounts.*.accessToken`
- `channels.matrix.accounts.*.password`
- `channels.nextcloud-talk.botSecret`
- `channels.nextcloud-talk.apiPassword`
- `channels.nextcloud-talk.accounts.*.botSecret`
- `channels.nextcloud-talk.accounts.*.apiPassword`
- `channels.zalo.botToken`
- `channels.zalo.webhookSecret`
- `channels.zalo.accounts.*.botToken`
- `channels.zalo.accounts.*.webhookSecret`
- `channels.googlechat.serviceAccount` via le champ frère `serviceAccountRef` (exception de compatibilité)
- `channels.googlechat.accounts.*.serviceAccount` via le champ frère `serviceAccountRef` (exception de compatibilité)

### Cibles `auth-profiles.json` (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"` ; non pris en charge lorsque `auth.profiles.<id>.mode = "oauth"`)
- `profiles.*.tokenRef` (`type: "token"` ; non pris en charge lorsque `auth.profiles.<id>.mode = "oauth"`)

[//]: # "secretref-supported-list-end"

Remarques :

- Les cibles de plan de profil d’authentification nécessitent `agentId`.
- Les entrées de plan ciblent `profiles.*.key` / `profiles.*.token` et écrivent les références sœurs (`keyRef` / `tokenRef`).
- Les références de profil d’authentification sont incluses dans la résolution à l’exécution et dans la couverture d’audit.
- Garde-fou de politique OAuth : `auth.profiles.<id>.mode = "oauth"` ne peut pas être combiné avec des entrées SecretRef pour ce profil. Le démarrage/rechargement et la résolution du profil d’authentification échouent immédiatement lorsque cette politique est violée.
- Pour les fournisseurs de modèles gérés par SecretRef, les entrées générées `agents/*/agent/models.json` conservent des marqueurs non secrets (et non des valeurs secrètes résolues) pour les surfaces `apiKey`/en-têtes.
- La persistance des marqueurs fait autorité côté source : OpenClaw écrit les marqueurs depuis l’instantané actif de la configuration source (avant résolution), et non depuis les valeurs secrètes résolues à l’exécution.
- Pour la recherche web :
  - En mode fournisseur explicite (`tools.web.search.provider` défini), seule la clé du fournisseur sélectionné est active.
  - En mode auto (`tools.web.search.provider` non défini), seule la première clé de fournisseur qui se résout selon la priorité est active.
  - En mode auto, les références des fournisseurs non sélectionnés sont traitées comme inactives jusqu’à leur sélection.
  - Les anciens chemins de fournisseur `tools.web.search.*` se résolvent encore pendant la fenêtre de compatibilité, mais la surface canonique SecretRef est `plugins.entries.<plugin>.config.webSearch.*`.

## Identifiants non pris en charge

Les identifiants hors périmètre incluent :

[//]: # "secretref-unsupported-list-start"

- `commands.ownerDisplaySecret`
- `hooks.token`
- `hooks.gmail.pushToken`
- `hooks.mappings[].sessionKey`
- `auth-profiles.oauth.*`
- `channels.discord.threadBindings.webhookToken`
- `channels.discord.accounts.*.threadBindings.webhookToken`
- `channels.whatsapp.creds.json`
- `channels.whatsapp.accounts.*.creds.json`

[//]: # "secretref-unsupported-list-end"

Justification :

- Ces identifiants appartiennent à des classes générées, renouvelées, porteuses de session ou durables OAuth qui ne correspondent pas à une résolution SecretRef externe en lecture seule.
