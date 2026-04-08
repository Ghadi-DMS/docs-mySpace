---
read_when:
    - Ajouter ou modifier des migrations doctor
    - Introduire des changements de configuration incompatibles
summary: 'Commande Doctor : contrôles de santé, migrations de configuration et étapes de réparation'
title: Doctor
x-i18n:
    generated_at: "2026-04-08T02:15:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3761a222d9db7088f78215575fa84e5896794ad701aa716e8bf9039a4424dca6
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` est l'outil de réparation + migration pour OpenClaw. Il corrige les configurations et états obsolètes, vérifie l'état de santé et fournit des étapes de réparation exploitables.

## Démarrage rapide

```bash
openclaw doctor
```

### Headless / automatisation

```bash
openclaw doctor --yes
```

Accepte les valeurs par défaut sans demander de confirmation (y compris les étapes de réparation de redémarrage/service/sandbox lorsqu'elles s'appliquent).

```bash
openclaw doctor --repair
```

Applique les réparations recommandées sans demander de confirmation (réparations + redémarrages lorsque cela est sûr).

```bash
openclaw doctor --repair --force
```

Applique aussi les réparations agressives (écrase les configurations de superviseur personnalisées).

```bash
openclaw doctor --non-interactive
```

Exécute sans invites et applique uniquement les migrations sûres (normalisation de configuration + déplacements d'état sur disque). Ignore les actions de redémarrage/service/sandbox qui nécessitent une confirmation humaine.
Les migrations d'état héritées s'exécutent automatiquement lorsqu'elles sont détectées.

```bash
openclaw doctor --deep
```

Analyse les services système pour détecter des installations Gateway supplémentaires (launchd/systemd/schtasks).

Si vous souhaitez examiner les modifications avant l'écriture, ouvrez d'abord le fichier de configuration :

```bash
cat ~/.openclaw/openclaw.json
```

## Ce qu'il fait (résumé)

- Mise à jour préalable facultative pour les installations git (interactif uniquement).
- Vérification de fraîcheur du protocole UI (reconstruit le Control UI lorsque le schéma de protocole est plus récent).
- Contrôle de santé + invite de redémarrage.
- Résumé de l'état des Skills (éligibles/manquants/bloqués) et état des plugins.
- Normalisation de configuration pour les valeurs héritées.
- Migration de la configuration Talk depuis les champs plats hérités `talk.*` vers `talk.provider` + `talk.providers.<provider>`.
- Vérifications de migration du navigateur pour les anciennes configurations d'extension Chrome et préparation Chrome MCP.
- Avertissements de remplacement de fournisseur OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Avertissements de masquage OAuth Codex (`models.providers.openai-codex`).
- Vérification des prérequis TLS OAuth pour les profils OAuth OpenAI Codex.
- Migration d'état héritée sur disque (sessions/répertoire agent/authentification WhatsApp).
- Migration de clé de contrat de manifeste de plugin héritée (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migration du magasin cron hérité (`jobId`, `schedule.cron`, champs delivery/payload de premier niveau, `provider` de payload, jobs de secours webhook simples `notify: true`).
- Inspection des fichiers de verrou de session et nettoyage des verrous obsolètes.
- Vérifications d'intégrité et d'autorisations de l'état (sessions, transcriptions, répertoire d'état).
- Vérifications des autorisations du fichier de configuration (`chmod 600`) lors d'une exécution locale.
- Santé de l'authentification des modèles : vérifie l'expiration OAuth, peut actualiser les jetons proches de l'expiration et signale les états de cooldown/désactivation des profils d'authentification.
- Détection de répertoire de workspace supplémentaire (`~/openclaw`).
- Réparation d'image sandbox lorsque le sandboxing est activé.
- Migration de service héritée et détection de Gateway supplémentaire.
- Migration d'état héritée du canal Matrix (en mode `--fix` / `--repair`).
- Vérifications du runtime Gateway (service installé mais non en cours d'exécution ; label launchd mis en cache).
- Avertissements d'état des canaux (sondés depuis la Gateway en cours d'exécution).
- Audit de configuration du superviseur (launchd/systemd/schtasks) avec réparation facultative.
- Vérifications des bonnes pratiques du runtime Gateway (Node vs Bun, chemins de gestionnaire de versions).
- Diagnostics de collision de port Gateway (par défaut `18789`).
- Avertissements de sécurité pour les politiques de messages privés ouvertes.
- Vérifications d'authentification Gateway pour le mode jeton local (propose de générer un jeton lorsqu'aucune source de jeton n'existe ; n'écrase pas les configurations de jeton SecretRef).
- Vérification de `linger` systemd sur Linux.
- Vérification de la taille des fichiers de bootstrap du workspace (avertissements de troncature/proximité de limite pour les fichiers de contexte).
- Vérification de l'état de l'autocomplétion shell et installation/mise à niveau automatiques.
- Vérification de la préparation du fournisseur d'embeddings de recherche mémoire (modèle local, clé API distante ou binaire QMD).
- Vérifications d'installation depuis les sources (incohérence de workspace pnpm, ressources UI manquantes, binaire tsx manquant).
- Écrit la configuration mise à jour + les métadonnées de l'assistant.

## Comportement détaillé et justification

### 0) Mise à jour facultative (installations git)

S'il s'agit d'un checkout git et que doctor s'exécute en mode interactif, il propose de faire une mise à jour (fetch/rebase/build) avant d'exécuter doctor.

### 1) Normalisation de la configuration

Si la configuration contient des formes de valeurs héritées (par exemple `messages.ackReaction` sans remplacement spécifique à un canal), doctor les normalise vers le schéma actuel.

Cela inclut les anciens champs plats Talk. La configuration Talk publique actuelle est `talk.provider` + `talk.providers.<provider>`. Doctor réécrit les anciennes formes `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` dans la map du fournisseur.

### 2) Migrations de clés de configuration héritées

Lorsque la configuration contient des clés dépréciées, les autres commandes refusent de s'exécuter et vous demandent d'exécuter `openclaw doctor`.

Doctor va :

- Expliquer quelles clés héritées ont été trouvées.
- Afficher la migration qu'il a appliquée.
- Réécrire `~/.openclaw/openclaw.json` avec le schéma mis à jour.

La Gateway exécute aussi automatiquement les migrations doctor au démarrage lorsqu'elle détecte un format de configuration hérité, de sorte que les configurations obsolètes soient réparées sans intervention manuelle.
Les migrations du magasin de tâches cron sont gérées par `openclaw doctor --fix`.

Migrations actuelles :

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` de premier niveau
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- anciens `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Pour les canaux avec des `accounts` nommés mais des valeurs persistantes de canal de premier niveau à compte unique, déplace ces valeurs à portée de compte dans le compte promu choisi pour ce canal (`accounts.default` pour la plupart des canaux ; Matrix peut préserver une cible nommée/par défaut correspondante existante)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- suppression de `browser.relayBindHost` (paramètre relay d'extension hérité)

Les avertissements doctor incluent aussi des recommandations sur le compte par défaut pour les canaux multi-comptes :

- Si deux entrées ou plus dans `channels.<channel>.accounts` sont configurées sans `channels.<channel>.defaultAccount` ou `accounts.default`, doctor avertit que le routage de secours peut choisir un compte inattendu.
- Si `channels.<channel>.defaultAccount` est défini sur un identifiant de compte inconnu, doctor avertit et liste les identifiants de compte configurés.

### 2b) Remplacements de fournisseur OpenCode

Si vous avez ajouté `models.providers.opencode`, `opencode-zen` ou `opencode-go` manuellement, cela remplace le catalogue OpenCode intégré provenant de `@mariozechner/pi-ai`.
Cela peut forcer des modèles vers une mauvaise API ou remettre les coûts à zéro. Doctor affiche un avertissement afin que vous puissiez supprimer ce remplacement et restaurer le routage API + les coûts par modèle.

### 2c) Migration du navigateur et préparation Chrome MCP

Si la configuration de votre navigateur pointe encore vers le chemin de l'extension Chrome supprimée, doctor la normalise vers le modèle d'attachement Chrome MCP local à l'hôte actuel :

- `browser.profiles.*.driver: "extension"` devient `"existing-session"`
- `browser.relayBindHost` est supprimé

Doctor audite aussi le chemin Chrome MCP local à l'hôte lorsque vous utilisez `defaultProfile:
"user"` ou un profil `existing-session` configuré :

- vérifie si Google Chrome est installé sur le même hôte pour les profils d'auto-connexion par défaut
- vérifie la version de Chrome détectée et avertit lorsqu'elle est inférieure à Chrome 144
- rappelle d'activer le débogage à distance dans la page d'inspection du navigateur (par exemple `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  ou `edge://inspect/#remote-debugging`)

Doctor ne peut pas activer le paramètre côté Chrome pour vous. Chrome MCP local à l'hôte nécessite toujours :

- un navigateur basé sur Chromium 144+ sur l'hôte gateway/node
- le navigateur en cours d'exécution localement
- le débogage à distance activé dans ce navigateur
- l'approbation de la première invite de consentement d'attachement dans le navigateur

La préparation ici concerne uniquement les prérequis d'attachement local. Existing-session conserve les limites actuelles de routage Chrome MCP ; les routes avancées comme `responsebody`, l'export PDF, l'interception de téléchargement et les actions par lot nécessitent toujours un navigateur géré ou un profil CDP brut.

Cette vérification ne s'applique **pas** à Docker, sandbox, remote-browser ou aux autres flux headless. Ceux-ci continuent d'utiliser le CDP brut.

### 2d) Prérequis TLS OAuth

Lorsqu'un profil OAuth OpenAI Codex est configuré, doctor sonde le point de terminaison d'autorisation OpenAI pour vérifier que la pile TLS locale Node/OpenSSL peut valider la chaîne de certificats. Si la sonde échoue avec une erreur de certificat (par exemple `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, certificat expiré ou certificat auto-signé), doctor affiche des instructions de correction spécifiques à la plateforme. Sur macOS avec un Node Homebrew, la correction est généralement `brew postinstall ca-certificates`. Avec `--deep`, la sonde s'exécute même si la Gateway est saine.

### 2c) Remplacements de fournisseur OAuth Codex

Si vous avez précédemment ajouté d'anciens paramètres de transport OpenAI sous `models.providers.openai-codex`, ils peuvent masquer le chemin de fournisseur Codex OAuth intégré que les versions plus récentes utilisent automatiquement. Doctor avertit lorsqu'il voit ces anciens paramètres de transport en même temps que Codex OAuth afin que vous puissiez supprimer ou réécrire le remplacement de transport obsolète et récupérer le comportement intégré de routage/secours. Les proxies personnalisés et les remplacements d'en-têtes uniquement restent pris en charge et ne déclenchent pas cet avertissement.

### 3) Migrations d'état héritées (disposition sur disque)

Doctor peut migrer les anciennes dispositions sur disque vers la structure actuelle :

- Magasin de sessions + transcriptions :
  - de `~/.openclaw/sessions/` vers `~/.openclaw/agents/<agentId>/sessions/`
- Répertoire agent :
  - de `~/.openclaw/agent/` vers `~/.openclaw/agents/<agentId>/agent/`
- État d'authentification WhatsApp (Baileys) :
  - depuis l'ancien `~/.openclaw/credentials/*.json` (sauf `oauth.json`)
  - vers `~/.openclaw/credentials/whatsapp/<accountId>/...` (identifiant de compte par défaut : `default`)

Ces migrations sont exécutées au mieux et sont idempotentes ; doctor émet des avertissements lorsqu'il laisse des dossiers hérités comme sauvegardes. La Gateway/CLI migre aussi automatiquement les anciennes sessions + le répertoire agent au démarrage afin que l'historique/l'authentification/les modèles aboutissent dans le chemin par agent sans exécution manuelle de doctor. L'authentification WhatsApp est intentionnellement migrée uniquement via `openclaw doctor`. La normalisation du fournisseur Talk/de la map de fournisseurs compare désormais par égalité structurelle, de sorte que les différences dues au seul ordre des clés ne déclenchent plus de modifications répétées sans effet de `doctor --fix`.

### 3a) Migrations héritées du manifeste de plugin

Doctor analyse tous les manifestes de plugin installés à la recherche de clés de capacité de premier niveau dépréciées (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Lorsqu'elles sont trouvées, il propose de les déplacer dans l'objet `contracts` et de réécrire le fichier manifeste sur place. Cette migration est idempotente ;
si la clé `contracts` contient déjà les mêmes valeurs, la clé héritée est supprimée sans dupliquer les données.

### 3b) Migrations héritées du magasin cron

Doctor vérifie aussi le magasin de tâches cron (`~/.openclaw/cron/jobs.json` par défaut,
ou `cron.store` lorsqu'il est remplacé) à la recherche d'anciennes formes de tâches que l'ordonnanceur accepte encore pour compatibilité.

Nettoyages cron actuels inclus :

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- champs payload de premier niveau (`message`, `model`, `thinking`, ...) → `payload`
- champs delivery de premier niveau (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias de delivery `provider` du payload → `delivery.channel` explicite
- anciens jobs de secours webhook simples `notify: true` → `delivery.mode="webhook"` explicite avec `delivery.to=cron.webhook`

Doctor ne migre automatiquement les jobs `notify: true` que lorsqu'il peut le faire sans changer le comportement. Si un job combine un ancien secours notify avec un mode delivery non-webhook existant, doctor avertit et laisse ce job pour révision manuelle.

### 3c) Nettoyage des verrous de session

Doctor analyse chaque répertoire de sessions d'agent à la recherche de fichiers de verrou d'écriture obsolètes — fichiers laissés derrière lorsqu'une session s'est terminée anormalement. Pour chaque fichier de verrou trouvé, il signale :
le chemin, le PID, si le PID est toujours vivant, l'âge du verrou et s'il est considéré comme obsolète (PID mort ou plus ancien que 30 minutes). En mode `--fix` / `--repair`, il supprime automatiquement les fichiers de verrou obsolètes ; sinon, il affiche une note et vous demande de réexécuter avec `--fix`.

### 4) Vérifications d'intégrité de l'état (persistance des sessions, routage et sécurité)

Le répertoire d'état est le tronc cérébral opérationnel. S'il disparaît, vous perdez
les sessions, les identifiants, les journaux et la configuration (sauf si vous avez des sauvegardes ailleurs).

Doctor vérifie :

- **Répertoire d'état manquant** : avertit d'une perte d'état catastrophique, propose de recréer le répertoire et rappelle qu'il ne peut pas récupérer les données manquantes.
- **Autorisations du répertoire d'état** : vérifie l'écriture ; propose de réparer les autorisations (et émet une indication `chown` lorsqu'un décalage propriétaire/groupe est détecté).
- **Répertoire d'état macOS synchronisé dans le cloud** : avertit lorsque l'état se résout sous iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) ou `~/Library/CloudStorage/...` car les chemins sauvegardés par synchronisation peuvent provoquer des E/S plus lentes et des courses de verrou/synchronisation.
- **Répertoire d'état Linux sur SD ou eMMC** : avertit lorsque l'état se résout vers une source de montage `mmcblk*`, car les E/S aléatoires sur carte SD ou eMMC peuvent être plus lentes et s'user plus rapidement avec les écritures de session et d'identifiants.
- **Répertoires de sessions manquants** : `sessions/` et le répertoire du magasin de sessions sont requis pour conserver l'historique et éviter les plantages `ENOENT`.
- **Incohérence de transcription** : avertit lorsque des entrées de session récentes ont des fichiers de transcription manquants.
- **Session principale « JSONL sur 1 ligne »** : signale lorsque la transcription principale n'a qu'une seule ligne (l'historique ne s'accumule pas).
- **Répertoires d'état multiples** : avertit lorsqu'il existe plusieurs dossiers `~/.openclaw` dans différents répertoires personnels ou lorsque `OPENCLAW_STATE_DIR` pointe ailleurs (l'historique peut se répartir entre les installations).
- **Rappel de mode distant** : si `gateway.mode=remote`, doctor rappelle de l'exécuter sur l'hôte distant (l'état y est stocké).
- **Autorisations du fichier de configuration** : avertit si `~/.openclaw/openclaw.json` est lisible par le groupe/le monde et propose de le resserrer à `600`.

### 5) Santé de l'authentification des modèles (expiration OAuth)

Doctor inspecte les profils OAuth dans le magasin d'authentification, avertit lorsque les jetons sont proches de l'expiration ou expirés, et peut les actualiser lorsque cela est sûr. Si le profil OAuth/jeton Anthropic est obsolète, il suggère une clé API Anthropic ou la voie setup-token Anthropic.
Les invites d'actualisation n'apparaissent qu'en mode interactif (TTY) ; `--non-interactive`
ignore les tentatives d'actualisation.

Doctor signale aussi les profils d'authentification temporairement inutilisables en raison de :

- cooldowns courts (limites de débit/timeouts/échecs d'authentification)
- désactivations plus longues (échecs de facturation/crédit)

### 6) Validation du modèle de hooks

Si `hooks.gmail.model` est défini, doctor valide la référence du modèle par rapport au catalogue et à la liste d'autorisation et avertit lorsqu'elle ne sera pas résolue ou qu'elle n'est pas autorisée.

### 7) Réparation d'image sandbox

Lorsque le sandboxing est activé, doctor vérifie les images Docker et propose de construire ou de basculer vers les noms hérités si l'image actuelle est manquante.

### 7b) Dépendances runtime des plugins inclus

Doctor vérifie que les dépendances runtime des plugins inclus (par exemple les packages runtime du plugin Discord) sont présentes à la racine d'installation d'OpenClaw.
Si certaines sont manquantes, doctor signale les packages et les installe en mode `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migrations de service Gateway et indications de nettoyage

Doctor détecte les anciens services Gateway (launchd/systemd/schtasks) et propose de les supprimer puis d'installer le service OpenClaw avec le port Gateway actuel. Il peut aussi analyser les services supplémentaires ressemblant à une Gateway et afficher des indications de nettoyage.
Les services Gateway OpenClaw nommés par profil sont considérés comme de première classe et ne sont pas signalés comme « supplémentaires ».

### 8b) Migration Matrix au démarrage

Lorsqu'un compte de canal Matrix a une migration d'état héritée en attente ou exploitable,
doctor (en mode `--fix` / `--repair`) crée un instantané avant migration puis exécute les étapes de migration au mieux : migration de l'état Matrix hérité et préparation héritée de l'état chiffré. Ces deux étapes ne sont pas fatales ; les erreurs sont journalisées et le démarrage continue. En mode lecture seule (`openclaw doctor` sans `--fix`), cette vérification est entièrement ignorée.

### 9) Avertissements de sécurité

Doctor émet des avertissements lorsqu'un fournisseur est ouvert aux messages privés sans liste d'autorisation, ou lorsqu'une politique est configurée de manière dangereuse.

### 10) `linger` systemd (Linux)

S'il s'exécute comme service utilisateur systemd, doctor s'assure que le mode lingering est activé afin que la gateway reste active après la déconnexion.

### 11) État du workspace (Skills, plugins et répertoires hérités)

Doctor affiche un résumé de l'état du workspace pour l'agent par défaut :

- **État des Skills** : compte les Skills éligibles, aux exigences manquantes et bloqués par liste d'autorisation.
- **Répertoires de workspace hérités** : avertit lorsque `~/openclaw` ou d'autres répertoires de workspace hérités existent à côté du workspace actuel.
- **État des plugins** : compte les plugins chargés/désactivés/en erreur ; liste les identifiants de plugin pour toute erreur ; signale les capacités des plugins inclus.
- **Avertissements de compatibilité des plugins** : signale les plugins qui ont des problèmes de compatibilité avec le runtime actuel.
- **Diagnostics de plugins** : fait remonter tous les avertissements ou erreurs au chargement émis par le registre des plugins.

### 11b) Taille du fichier bootstrap

Doctor vérifie si les fichiers de bootstrap du workspace (par exemple `AGENTS.md`,
`CLAUDE.md` ou d'autres fichiers de contexte injectés) approchent ou dépassent le budget de caractères configuré. Il indique, pour chaque fichier, le nombre brut de caractères par rapport au nombre injecté, le pourcentage de troncature, la cause de la troncature (`max/file` ou `max/total`) et le total des caractères injectés comme fraction du budget total. Lorsque des fichiers sont tronqués ou proches de la limite, doctor affiche des conseils pour ajuster `agents.defaults.bootstrapMaxChars`
et `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Autocomplétion shell

Doctor vérifie si l'autocomplétion par tabulation est installée pour le shell actuel
(zsh, bash, fish ou PowerShell) :

- Si le profil shell utilise un modèle d'autocomplétion dynamique lent
  (`source <(openclaw completion ...)`), doctor le met à niveau vers la variante plus rapide avec fichier en cache.
- Si l'autocomplétion est configurée dans le profil mais que le fichier de cache est manquant,
  doctor régénère automatiquement le cache.
- Si aucune autocomplétion n'est configurée, doctor propose de l'installer
  (mode interactif uniquement ; ignoré avec `--non-interactive`).

Exécutez `openclaw completion --write-state` pour régénérer manuellement le cache.

### 12) Vérifications d'authentification Gateway (jeton local)

Doctor vérifie la préparation de l'authentification par jeton de la gateway locale.

- Si le mode jeton a besoin d'un jeton et qu'aucune source de jeton n'existe, doctor propose d'en générer un.
- Si `gateway.auth.token` est géré par SecretRef mais indisponible, doctor avertit et ne l'écrase pas avec du texte brut.
- `openclaw doctor --generate-gateway-token` force la génération uniquement lorsqu'aucun SecretRef de jeton n'est configuré.

### 12b) Réparations en lecture seule tenant compte de SecretRef

Certains flux de réparation doivent inspecter les identifiants configurés sans affaiblir le comportement d'échec rapide du runtime.

- `openclaw doctor --fix` utilise maintenant le même modèle récapitulatif SecretRef en lecture seule que les commandes de la famille status pour des réparations de configuration ciblées.
- Exemple : la réparation Telegram `allowFrom` / `groupAllowFrom` `@username` essaie d'utiliser les identifiants du bot configurés lorsqu'ils sont disponibles.
- Si le jeton du bot Telegram est configuré via SecretRef mais indisponible dans le chemin de commande actuel, doctor signale que l'identifiant est configuré mais indisponible et ignore l'auto-résolution au lieu de planter ou de signaler à tort que le jeton est manquant.

### 13) Contrôle de santé Gateway + redémarrage

Doctor exécute un contrôle de santé et propose de redémarrer la gateway lorsqu'elle semble défaillante.

### 13b) Préparation de la recherche mémoire

Doctor vérifie si le fournisseur d'embeddings configuré pour la recherche mémoire est prêt pour l'agent par défaut. Le comportement dépend du backend et du fournisseur configurés :

- **Backend QMD** : sonde si le binaire `qmd` est disponible et peut démarrer.
  Si ce n'est pas le cas, affiche des indications de correction, y compris le package npm et une option de chemin binaire manuel.
- **Fournisseur local explicite** : vérifie la présence d'un fichier de modèle local ou d'une URL de modèle distante/téléchargeable reconnue. S'il manque, suggère de passer à un fournisseur distant.
- **Fournisseur distant explicite** (`openai`, `voyage`, etc.) : vérifie qu'une clé API est présente dans l'environnement ou dans le magasin d'authentification. Affiche des conseils de correction exploitables si elle manque.
- **Fournisseur auto** : vérifie d'abord la disponibilité du modèle local, puis essaie chaque fournisseur distant selon l'ordre de sélection automatique.

Lorsqu'un résultat de sonde gateway est disponible (la gateway était saine au moment de la vérification), doctor le recoupe avec la configuration visible côté CLI et signale toute divergence.

Utilisez `openclaw memory status --deep` pour vérifier la préparation des embeddings au runtime.

### 14) Avertissements d'état des canaux

Si la gateway est saine, doctor exécute une sonde d'état des canaux et affiche les avertissements avec les corrections suggérées.

### 15) Audit + réparation de la configuration du superviseur

Doctor vérifie la configuration du superviseur installé (launchd/systemd/schtasks) pour détecter les valeurs par défaut manquantes ou obsolètes (par ex. dépendances systemd network-online et délai de redémarrage). Lorsqu'il détecte une incohérence, il recommande une mise à jour et peut réécrire le fichier de service/la tâche selon les valeurs par défaut actuelles.

Remarques :

- `openclaw doctor` demande confirmation avant de réécrire la configuration du superviseur.
- `openclaw doctor --yes` accepte les invites de réparation par défaut.
- `openclaw doctor --repair` applique les corrections recommandées sans invites.
- `openclaw doctor --repair --force` écrase les configurations de superviseur personnalisées.
- Si l'authentification par jeton nécessite un jeton et que `gateway.auth.token` est géré par SecretRef, l'installation/réparation du service doctor valide le SecretRef mais ne persiste pas les valeurs de jeton en texte brut résolues dans les métadonnées d'environnement du service superviseur.
- Si l'authentification par jeton nécessite un jeton et que le SecretRef de jeton configuré n'est pas résolu, doctor bloque le chemin d'installation/réparation avec des indications exploitables.
- Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés et que `gateway.auth.mode` n'est pas défini, doctor bloque l'installation/réparation jusqu'à ce que le mode soit défini explicitement.
- Pour les unités user-systemd Linux, les vérifications de dérive de jeton de doctor incluent maintenant les sources `Environment=` et `EnvironmentFile=` lors de la comparaison des métadonnées d'authentification du service.
- Vous pouvez toujours forcer une réécriture complète via `openclaw gateway install --force`.

### 16) Diagnostics du runtime Gateway + du port

Doctor inspecte le runtime du service (PID, dernier statut de sortie) et avertit lorsque le
service est installé mais n'est pas réellement en cours d'exécution. Il vérifie aussi les collisions de port sur le port Gateway (par défaut `18789`) et signale les causes probables (gateway déjà en cours d'exécution, tunnel SSH).

### 17) Bonnes pratiques du runtime Gateway

Doctor avertit lorsque le service gateway s'exécute sur Bun ou sur un chemin Node géré par un gestionnaire de versions (`nvm`, `fnm`, `volta`, `asdf`, etc.). Les canaux WhatsApp + Telegram nécessitent Node,
et les chemins de gestionnaire de versions peuvent se casser après des mises à niveau parce que le service ne charge pas l'initialisation de votre shell. Doctor propose de migrer vers une installation Node système lorsque disponible (Homebrew/apt/choco).

### 18) Écriture de la configuration + métadonnées de l'assistant

Doctor persiste toutes les modifications de configuration et inscrit des métadonnées d'assistant pour enregistrer l'exécution de doctor.

### 19) Conseils de workspace (sauvegarde + système mémoire)

Doctor suggère un système mémoire de workspace lorsqu'il est absent et affiche un conseil de sauvegarde si le workspace n'est pas déjà sous git.

Voir [/concepts/agent-workspace](/fr/concepts/agent-workspace) pour un guide complet de la structure du workspace et de la sauvegarde git (GitHub ou GitLab privé recommandé).
