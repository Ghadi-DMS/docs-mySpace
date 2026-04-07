---
read_when:
    - Ajouter ou modifier des migrations doctor
    - Introduire des changements cassants de configuration
summary: 'Commande Doctor : vérifications d''état, migrations de configuration et étapes de réparation'
title: Doctor
x-i18n:
    generated_at: "2026-04-07T06:50:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: a834dc7aec79c20d17bc23d37fb5f5e99e628d964d55bd8cf24525a7ee57130c
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` est l'outil de réparation + migration pour OpenClaw. Il corrige les
configurations/états obsolètes, vérifie l'état général et fournit des étapes de réparation exploitables.

## Démarrage rapide

```bash
openclaw doctor
```

### Sans interface / automatisation

```bash
openclaw doctor --yes
```

Accepte les valeurs par défaut sans demander de confirmation (y compris les étapes de réparation de redémarrage/service/sandbox le cas échéant).

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

Analyse les services système pour détecter des installations supplémentaires de passerelle (launchd/systemd/schtasks).

Si vous voulez examiner les modifications avant l'écriture, ouvrez d'abord le fichier de configuration :

```bash
cat ~/.openclaw/openclaw.json
```

## Ce qu'il fait (résumé)

- Mise à jour préalable facultative pour les installations git (interactif uniquement).
- Vérification de fraîcheur du protocole UI (reconstruit le Control UI lorsque le schéma de protocole est plus récent).
- Vérification de l'état + invite au redémarrage.
- Résumé de l'état des Skills (éligibles/manquants/bloqués) et état des plugins.
- Normalisation de la configuration pour les valeurs héritées.
- Migration de la configuration Talk depuis les champs plats hérités `talk.*` vers `talk.provider` + `talk.providers.<provider>`.
- Vérifications de migration du navigateur pour les anciennes configurations d'extension Chrome et la préparation de Chrome MCP.
- Avertissements de remplacement du fournisseur OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Vérification des prérequis TLS OAuth pour les profils OAuth OpenAI Codex.
- Migration d'état hérité sur disque (sessions/répertoire d'agent/authentification WhatsApp).
- Migration des clés de contrat héritées des manifestes de plugin (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migration du magasin cron hérité (`jobId`, `schedule.cron`, champs delivery/payload de niveau supérieur, `provider` dans payload, tâches de secours webhook simples `notify: true`).
- Inspection des fichiers de verrouillage de session et nettoyage des verrous obsolètes.
- Vérifications d'intégrité et de permissions de l'état (sessions, transcriptions, répertoire d'état).
- Vérifications des permissions du fichier de configuration (`chmod 600`) lors d'une exécution en local.
- État de l'authentification des modèles : vérifie l'expiration OAuth, peut actualiser les jetons proches de l'expiration et signale les états de refroidissement/désactivation des profils d'authentification.
- Détection d'un répertoire de workspace supplémentaire (`~/openclaw`).
- Réparation de l'image sandbox lorsque le sandboxing est activé.
- Migration des anciens services et détection de passerelles supplémentaires.
- Migration d'état héritée du canal Matrix (en mode `--fix` / `--repair`).
- Vérifications d'exécution de la passerelle (service installé mais non exécuté ; libellé launchd en cache).
- Avertissements d'état des canaux (sondés depuis la passerelle en cours d'exécution).
- Audit de la configuration du superviseur (launchd/systemd/schtasks) avec réparation facultative.
- Vérifications des bonnes pratiques d'exécution de la passerelle (Node vs Bun, chemins de gestionnaire de versions).
- Diagnostic de conflit de port de la passerelle (par défaut `18789`).
- Avertissements de sécurité pour les politiques DM ouvertes.
- Vérifications d'authentification de la passerelle pour le mode à jeton local (propose de générer un jeton lorsqu'aucune source de jeton n'existe ; n'écrase pas les configurations token SecretRef).
- Vérification de linger systemd sous Linux.
- Vérification de la taille des fichiers bootstrap du workspace (avertissements de troncature/proximité de limite pour les fichiers de contexte).
- Vérification de l'état de l'autocomplétion du shell et installation/mise à niveau automatique.
- Vérification de l'état de préparation du fournisseur d'embedding pour la recherche mémoire (modèle local, clé API distante ou binaire QMD).
- Vérifications de l'installation depuis les sources (incohérence de workspace pnpm, ressources UI manquantes, binaire tsx manquant).
- Écrit la configuration mise à jour + les métadonnées de l'assistant.

## Comportement détaillé et justification

### 0) Mise à jour facultative (installations git)

S'il s'agit d'un dépôt git et que doctor est exécuté en mode interactif, il propose
de mettre à jour (fetch/rebase/build) avant d'exécuter doctor.

### 1) Normalisation de la configuration

Si la configuration contient des formes de valeurs héritées (par exemple `messages.ackReaction`
sans remplacement spécifique à un canal), doctor les normalise vers le schéma
actuel.

Cela inclut les champs plats hérités de Talk. La configuration publique actuelle de Talk est
`talk.provider` + `talk.providers.<provider>`. Doctor réécrit les anciennes formes
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` dans la table des fournisseurs.

### 2) Migrations de clés de configuration héritées

Lorsque la configuration contient des clés obsolètes, les autres commandes refusent de s'exécuter et demandent
de lancer `openclaw doctor`.

Doctor va :

- Expliquer quelles clés héritées ont été trouvées.
- Montrer la migration qu'il a appliquée.
- Réécrire `~/.openclaw/openclaw.json` avec le schéma mis à jour.

La passerelle exécute également automatiquement les migrations doctor au démarrage lorsqu'elle détecte un
format de configuration hérité, de sorte que les configurations obsolètes sont réparées sans intervention manuelle.
Les migrations du magasin de tâches cron sont gérées par `openclaw doctor --fix`.

Migrations actuelles :

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` de niveau supérieur
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- anciennes formes `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
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
- Pour les canaux avec des `accounts` nommés mais des valeurs de canal de niveau supérieur à compte unique encore présentes, déplacer ces valeurs limitées au compte vers le compte promu choisi pour ce canal (`accounts.default` pour la plupart des canaux ; Matrix peut conserver une cible nommée/par défaut existante correspondante)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- suppression de `browser.relayBindHost` (ancien paramètre de relais d'extension)

Les avertissements doctor incluent aussi des recommandations sur les comptes par défaut pour les canaux multicomptes :

- Si deux entrées ou plus `channels.<channel>.accounts` sont configurées sans `channels.<channel>.defaultAccount` ou `accounts.default`, doctor avertit que le routage de repli peut choisir un compte inattendu.
- Si `channels.<channel>.defaultAccount` est défini sur un ID de compte inconnu, doctor avertit et liste les ID de compte configurés.

### 2b) Remplacements du fournisseur OpenCode

Si vous avez ajouté `models.providers.opencode`, `opencode-zen` ou `opencode-go`
manuellement, cela remplace le catalogue OpenCode intégré de `@mariozechner/pi-ai`.
Cela peut forcer des modèles sur la mauvaise API ou remettre les coûts à zéro. Doctor avertit afin que vous
puissiez supprimer le remplacement et restaurer le routage API + les coûts par modèle.

### 2c) Migration du navigateur et préparation de Chrome MCP

Si votre configuration de navigateur pointe encore vers l'ancien chemin d'extension Chrome, doctor
la normalise vers le modèle actuel d'attachement Chrome MCP local à l'hôte :

- `browser.profiles.*.driver: "extension"` devient `"existing-session"`
- `browser.relayBindHost` est supprimé

Doctor audite aussi le chemin Chrome MCP local à l'hôte lorsque vous utilisez `defaultProfile:
"user"` ou un profil `existing-session` configuré :

- vérifie si Google Chrome est installé sur le même hôte pour les profils
  d'autoconnexion par défaut
- vérifie la version détectée de Chrome et avertit lorsqu'elle est inférieure à Chrome 144
- rappelle d'activer le débogage à distance dans la page d'inspection du navigateur (par
  exemple `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  ou `edge://inspect/#remote-debugging`)

Doctor ne peut pas activer pour vous le paramètre côté Chrome. Le Chrome MCP local à l'hôte
nécessite toujours :

- un navigateur basé sur Chromium 144+ sur l'hôte gateway/node
- le navigateur exécuté localement
- le débogage à distance activé dans ce navigateur
- l'approbation de la première invite de consentement d'attachement dans le navigateur

Ici, l'état de préparation concerne uniquement les prérequis d'attachement local. Existing-session conserve
les limites actuelles des routes Chrome MCP ; les routes avancées comme `responsebody`, l'export PDF,
l'interception des téléchargements et les actions par lot exigent toujours un
navigateur géré ou un profil CDP brut.

Cette vérification **ne s'applique pas** à Docker, au sandbox, à `remote-browser` ni aux autres
flux sans interface. Ceux-ci continuent d'utiliser du CDP brut.

### 2d) Prérequis TLS OAuth

Lorsqu'un profil OAuth OpenAI Codex est configuré, doctor sonde le point d'autorisation OpenAI
pour vérifier que la pile TLS locale Node/OpenSSL peut valider la chaîne de certificats. Si le sondage échoue avec une erreur de certificat (par
exemple `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, certificat expiré ou certificat autosigné),
doctor affiche des conseils de correction spécifiques à la plateforme. Sur macOS avec un Node Homebrew, la
correction est généralement `brew postinstall ca-certificates`. Avec `--deep`, le sondage s'exécute
même si la passerelle est saine.

### 3) Migrations d'état héritées (organisation sur disque)

Doctor peut migrer d'anciennes organisations sur disque vers la structure actuelle :

- Magasin de sessions + transcriptions :
  - depuis `~/.openclaw/sessions/` vers `~/.openclaw/agents/<agentId>/sessions/`
- Répertoire d'agent :
  - depuis `~/.openclaw/agent/` vers `~/.openclaw/agents/<agentId>/agent/`
- État d'authentification WhatsApp (Baileys) :
  - depuis l'ancien `~/.openclaw/credentials/*.json` (sauf `oauth.json`)
  - vers `~/.openclaw/credentials/whatsapp/<accountId>/...` (ID de compte par défaut : `default`)

Ces migrations sont réalisées au mieux et sont idempotentes ; doctor émet des avertissements lorsqu'il
laisse des dossiers hérités derrière lui comme sauvegardes. La passerelle/CLI migre aussi automatiquement
les anciennes sessions + le répertoire d'agent au démarrage afin que l'historique/l'authentification/les modèles arrivent dans le
chemin par agent sans exécution manuelle de doctor. L'authentification WhatsApp n'est volontairement migrée que
via `openclaw doctor`. La normalisation Talk provider/provider-map compare désormais par égalité structurelle,
de sorte que les différences d'ordre des clés à elles seules ne déclenchent plus de
modifications répétées sans effet de `doctor --fix`.

### 3a) Migrations héritées des manifestes de plugin

Doctor analyse tous les manifestes de plugin installés à la recherche de clés de capacité
obsolètes de niveau supérieur (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Lorsqu'il en trouve, il propose de les déplacer dans l'objet `contracts`
et de réécrire le fichier manifeste sur place. Cette migration est idempotente ;
si la clé `contracts` contient déjà les mêmes valeurs, la clé héritée est supprimée
sans dupliquer les données.

### 3b) Migrations héritées du magasin cron

Doctor vérifie aussi le magasin de tâches cron (`~/.openclaw/cron/jobs.json` par défaut,
ou `cron.store` lorsqu'il est remplacé) pour détecter les anciennes formes de tâches que l'ordonnanceur accepte encore
pour des raisons de compatibilité.

Nettoyages cron actuels :

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- champs payload de niveau supérieur (`message`, `model`, `thinking`, ...) → `payload`
- champs delivery de niveau supérieur (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias delivery `provider` dans payload → `delivery.channel` explicite
- tâches de secours webhook héritées simples `notify: true` → `delivery.mode="webhook"` explicite avec `delivery.to=cron.webhook`

Doctor ne migre automatiquement les tâches `notify: true` que lorsqu'il peut le faire sans
changer le comportement. Si une tâche combine le mécanisme de secours notify hérité avec un mode delivery
non webhook existant, doctor avertit et laisse cette tâche pour une révision manuelle.

### 3c) Nettoyage des verrous de session

Doctor analyse chaque répertoire de session d'agent à la recherche de fichiers de verrouillage d'écriture obsolètes — des fichiers laissés
derrière lorsqu'une session s'est arrêtée anormalement. Pour chaque fichier de verrou trouvé, il indique :
le chemin, le PID, si le PID est encore actif, l'âge du verrou et s'il est
considéré comme obsolète (PID mort ou plus de 30 minutes). En mode `--fix` / `--repair`,
il supprime automatiquement les fichiers de verrouillage obsolètes ; sinon il affiche une note et
vous demande de relancer avec `--fix`.

### 4) Vérifications d'intégrité de l'état (persistance de session, routage et sécurité)

Le répertoire d'état est le tronc cérébral opérationnel. S'il disparaît, vous perdez
sessions, identifiants, journaux et configuration (sauf si vous avez des sauvegardes ailleurs).

Doctor vérifie :

- **Répertoire d'état manquant** : avertit d'une perte d'état catastrophique, propose de recréer
  le répertoire et rappelle qu'il ne peut pas récupérer les données manquantes.
- **Permissions du répertoire d'état** : vérifie la possibilité d'écriture ; propose de réparer les permissions
  (et affiche une indication `chown` lorsqu'une incohérence propriétaire/groupe est détectée).
- **Répertoire d'état macOS synchronisé sur le cloud** : avertit lorsque l'état se résout sous iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) ou
  `~/Library/CloudStorage/...` car les chemins synchronisés peuvent entraîner des E/S plus lentes
  et des courses de verrouillage/synchronisation.
- **Répertoire d'état Linux sur SD ou eMMC** : avertit lorsque l'état se résout vers une source de montage `mmcblk*`,
  car les E/S aléatoires sur carte SD ou eMMC peuvent être plus lentes et s'user
  plus vite lors des écritures de session et d'identifiants.
- **Répertoires de session manquants** : `sessions/` et le répertoire du magasin de sessions sont
  nécessaires pour conserver l'historique et éviter les plantages `ENOENT`.
- **Incohérence de transcription** : avertit lorsque des entrées de session récentes ont des
  fichiers de transcription manquants.
- **Session principale « JSONL sur 1 ligne »** : signale lorsque la transcription principale n'a qu'une
  seule ligne (l'historique ne s'accumule pas).
- **Plusieurs répertoires d'état** : avertit lorsque plusieurs dossiers `~/.openclaw` existent entre
  des répertoires personnels ou lorsque `OPENCLAW_STATE_DIR` pointe ailleurs (l'historique peut
  se répartir entre plusieurs installations).
- **Rappel mode distant** : si `gateway.mode=remote`, doctor rappelle de l'exécuter
  sur l'hôte distant (l'état s'y trouve).
- **Permissions du fichier de configuration** : avertit si `~/.openclaw/openclaw.json` est
  lisible par le groupe/le monde et propose de le durcir à `600`.

### 5) État de l'authentification des modèles (expiration OAuth)

Doctor inspecte les profils OAuth dans le magasin d'authentification, avertit lorsque les jetons
expirent ou ont expiré, et peut les actualiser lorsque cela est sûr. Si le profil
OAuth/jeton Anthropic est obsolète, il suggère une clé API Anthropic ou la
méthode de jeton d'installation Anthropic.
Les invites d'actualisation n'apparaissent qu'en mode interactif (TTY) ; `--non-interactive`
ignore les tentatives d'actualisation.

Doctor signale aussi les profils d'authentification temporairement inutilisables à cause de :

- courtes périodes de refroidissement (limites de débit/délais d'attente/échecs d'authentification)
- désactivations plus longues (échecs de facturation/crédit)

### 6) Validation du modèle hooks

Si `hooks.gmail.model` est défini, doctor valide la référence du modèle par rapport au
catalogue et à la liste d'autorisation et avertit lorsqu'elle ne pourra pas être résolue ou n'est pas autorisée.

### 7) Réparation de l'image sandbox

Lorsque le sandboxing est activé, doctor vérifie les images Docker et propose de construire ou
de basculer vers les anciens noms si l'image actuelle est manquante.

### 7b) Dépendances d'exécution des plugins intégrés

Doctor vérifie que les dépendances d'exécution des plugins intégrés (par exemple les
packages d'exécution du plugin Discord) sont présentes à la racine de l'installation OpenClaw.
S'il en manque, doctor signale les packages et les installe en mode
`openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migrations des services de passerelle et indications de nettoyage

Doctor détecte les anciens services de passerelle (launchd/systemd/schtasks) et
propose de les supprimer puis d'installer le service OpenClaw en utilisant le port actuel de la passerelle.
Il peut aussi analyser les services supplémentaires de type passerelle et afficher des indications de nettoyage.
Les services de passerelle OpenClaw nommés par profil sont considérés comme de première classe et ne sont pas
signalés comme « supplémentaires ».

### 8b) Migration Matrix au démarrage

Lorsqu'un compte de canal Matrix a une migration d'état héritée en attente ou exploitable,
doctor (en mode `--fix` / `--repair`) crée un instantané avant migration puis
exécute les étapes de migration au mieux : migration de l'état Matrix hérité et préparation
de l'ancien état chiffré. Les deux étapes ne sont pas fatales ; les erreurs sont consignées et le
démarrage continue. En mode lecture seule (`openclaw doctor` sans `--fix`), cette vérification
est complètement ignorée.

### 9) Avertissements de sécurité

Doctor émet des avertissements lorsqu'un fournisseur est ouvert aux DM sans liste d'autorisation, ou
lorsqu'une politique est configurée de manière dangereuse.

### 10) Linger systemd (Linux)

S'il s'exécute comme service utilisateur systemd, doctor s'assure que le linger est activé afin que la
passerelle reste active après la déconnexion.

### 11) État du workspace (Skills, plugins et anciens répertoires)

Doctor affiche un résumé de l'état du workspace pour l'agent par défaut :

- **État des Skills** : compte les Skills éligibles, à prérequis manquants et bloqués par liste d'autorisation.
- **Anciens répertoires de workspace** : avertit lorsque `~/openclaw` ou d'autres anciens répertoires de workspace
  existent à côté du workspace actuel.
- **État des plugins** : compte les plugins chargés/désactivés/en erreur ; liste les ID de plugin pour les
  erreurs ; signale les capacités des plugins du bundle.
- **Avertissements de compatibilité des plugins** : signale les plugins qui ont des problèmes de compatibilité avec
  l'environnement d'exécution actuel.
- **Diagnostics de plugin** : expose les avertissements ou erreurs au chargement émis par le
  registre de plugins.

### 11b) Taille des fichiers bootstrap

Doctor vérifie si les fichiers bootstrap du workspace (par exemple `AGENTS.md`,
`CLAUDE.md` ou d'autres fichiers de contexte injectés) sont proches ou au-delà du
budget de caractères configuré. Il signale, par fichier, le nombre brut de caractères vs injectés, le
pourcentage de troncature, la cause de la troncature (`max/file` ou `max/total`) et le total des caractères injectés en tant que fraction du budget total. Lorsque des fichiers sont tronqués ou proches
de la limite, doctor affiche des conseils pour régler `agents.defaults.bootstrapMaxChars`
et `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Autocomplétion du shell

Doctor vérifie si l'autocomplétion par tabulation est installée pour le shell actuel
(zsh, bash, fish ou PowerShell) :

- Si le profil du shell utilise un modèle lent d'autocomplétion dynamique
  (`source <(openclaw completion ...)`), doctor le met à niveau vers la variante plus rapide
  basée sur un fichier en cache.
- Si l'autocomplétion est configurée dans le profil mais que le fichier de cache est manquant,
  doctor régénère automatiquement le cache.
- Si aucune autocomplétion n'est configurée, doctor propose de l'installer
  (mode interactif uniquement ; ignoré avec `--non-interactive`).

Exécutez `openclaw completion --write-state` pour régénérer manuellement le cache.

### 12) Vérifications d'authentification de la passerelle (jeton local)

Doctor vérifie l'état de préparation de l'authentification par jeton de la passerelle locale.

- Si le mode jeton a besoin d'un jeton et qu'aucune source de jeton n'existe, doctor propose d'en générer un.
- Si `gateway.auth.token` est géré par SecretRef mais indisponible, doctor avertit et ne l'écrase pas avec du texte brut.
- `openclaw doctor --generate-gateway-token` force la génération uniquement lorsqu'aucun SecretRef de jeton n'est configuré.

### 12b) Réparations en lecture seule tenant compte de SecretRef

Certains flux de réparation doivent inspecter les identifiants configurés sans affaiblir le comportement d'échec rapide à l'exécution.

- `openclaw doctor --fix` utilise maintenant le même modèle de résumé SecretRef en lecture seule que les commandes de la famille status pour les réparations de configuration ciblées.
- Exemple : la réparation Telegram `allowFrom` / `groupAllowFrom` avec `@username` essaie d'utiliser les identifiants du bot configurés lorsqu'ils sont disponibles.
- Si le jeton du bot Telegram est configuré via SecretRef mais indisponible dans le chemin de commande actuel, doctor signale que l'identifiant est configuré mais indisponible et ignore la résolution automatique au lieu de planter ou d'indiquer à tort que le jeton est manquant.

### 13) Vérification de l'état de la passerelle + redémarrage

Doctor exécute une vérification d'état et propose de redémarrer la passerelle lorsqu'elle semble
en mauvais état.

### 13b) Préparation de la recherche mémoire

Doctor vérifie si le fournisseur d'embedding configuré pour la recherche mémoire est prêt
pour l'agent par défaut. Le comportement dépend du backend et du fournisseur configurés :

- **Backend QMD** : sonde si le binaire `qmd` est disponible et peut démarrer.
  Sinon, affiche des conseils de correction, y compris le package npm et une option manuelle de chemin binaire.
- **Fournisseur local explicite** : vérifie la présence d'un fichier de modèle local ou d'une URL de modèle distante/téléchargeable reconnue. S'il manque, suggère de passer à un fournisseur distant.
- **Fournisseur distant explicite** (`openai`, `voyage`, etc.) : vérifie qu'une clé API est
  présente dans l'environnement ou le magasin d'authentification. Affiche des conseils de correction exploitables si elle est absente.
- **Fournisseur automatique** : vérifie d'abord la disponibilité du modèle local, puis essaie chaque
  fournisseur distant dans l'ordre de sélection automatique.

Lorsqu'un résultat de sonde de passerelle est disponible (la passerelle était saine au moment de la
vérification), doctor le recoupe avec la configuration visible par la CLI et signale
toute divergence.

Utilisez `openclaw memory status --deep` pour vérifier l'état de préparation des embeddings à l'exécution.

### 14) Avertissements sur l'état des canaux

Si la passerelle est saine, doctor exécute une sonde d'état des canaux et signale
les avertissements avec des corrections suggérées.

### 15) Audit de la configuration du superviseur + réparation

Doctor vérifie la configuration du superviseur installé (launchd/systemd/schtasks) pour
détecter des valeurs par défaut manquantes ou obsolètes (par exemple dépendances systemd network-online et
délai de redémarrage). Lorsqu'il trouve une incohérence, il recommande une mise à jour et peut
réécrire le fichier de service/la tâche avec les valeurs par défaut actuelles.

Remarques :

- `openclaw doctor` demande confirmation avant de réécrire la configuration du superviseur.
- `openclaw doctor --yes` accepte les invites de réparation par défaut.
- `openclaw doctor --repair` applique les corrections recommandées sans invites.
- `openclaw doctor --repair --force` écrase les configurations de superviseur personnalisées.
- Si l'authentification par jeton nécessite un jeton et que `gateway.auth.token` est géré par SecretRef, l'installation/réparation du service doctor valide le SecretRef mais ne conserve pas les valeurs de jeton en clair résolues dans les métadonnées d'environnement du service superviseur.
- Si l'authentification par jeton nécessite un jeton et que le SecretRef de jeton configuré n'est pas résolu, doctor bloque le chemin d'installation/réparation avec des conseils exploitables.
- Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés et que `gateway.auth.mode` n'est pas défini, doctor bloque l'installation/réparation jusqu'à ce que le mode soit défini explicitement.
- Pour les unités user-systemd Linux, les vérifications de dérive de jeton de doctor incluent maintenant à la fois les sources `Environment=` et `EnvironmentFile=` lors de la comparaison des métadonnées d'authentification du service.
- Vous pouvez toujours forcer une réécriture complète via `openclaw gateway install --force`.

### 16) Diagnostic d'exécution de la passerelle + du port

Doctor inspecte l'exécution du service (PID, dernier état de sortie) et avertit lorsque le
service est installé mais n'est pas réellement en cours d'exécution. Il vérifie aussi les conflits
sur le port de la passerelle (par défaut `18789`) et signale les causes probables (passerelle déjà
en cours d'exécution, tunnel SSH).

### 17) Bonnes pratiques d'exécution de la passerelle

Doctor avertit lorsque le service de passerelle s'exécute sur Bun ou via un chemin Node géré par un gestionnaire de versions
(`nvm`, `fnm`, `volta`, `asdf`, etc.). Les canaux WhatsApp + Telegram exigent Node,
et les chemins de gestionnaire de versions peuvent se casser après des mises à niveau parce que le service ne
charge pas l'initialisation de votre shell. Doctor propose de migrer vers une installation Node système lorsqu'elle est
disponible (Homebrew/apt/choco).

### 18) Écriture de la configuration + métadonnées de l'assistant

Doctor conserve toute modification de configuration et appose des métadonnées de l'assistant pour enregistrer l'exécution
de doctor.

### 19) Conseils de workspace (sauvegarde + système mémoire)

Doctor suggère un système de mémoire de workspace lorsqu'il est absent et affiche un conseil de sauvegarde
si le workspace n'est pas déjà sous git.

Voir [/concepts/agent-workspace](/fr/concepts/agent-workspace) pour un guide complet sur la
structure du workspace et la sauvegarde git (GitHub ou GitLab privé recommandé).
