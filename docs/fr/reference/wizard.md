---
read_when:
    - Rechercher une étape ou un drapeau d’onboarding spécifique
    - Automatiser l’onboarding avec le mode non interactif
    - Déboguer le comportement de l’onboarding
sidebarTitle: Onboarding Reference
summary: 'Référence complète pour l’onboarding CLI : chaque étape, drapeau et champ de configuration'
title: Référence de l’onboarding
x-i18n:
    generated_at: "2026-04-07T06:55:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: a142b9ec4323fabb9982d05b64375d2b4a4007dffc910acbee3a38ff871a7236
    source_path: reference/wizard.md
    workflow: 15
---

# Référence de l’onboarding

Ceci est la référence complète pour `openclaw onboard`.
Pour une vue d’ensemble de haut niveau, voir [Onboarding (CLI)](/fr/start/wizard).

## Détails du flux (mode local)

<Steps>
  <Step title="Détection de configuration existante">
    - Si `~/.openclaw/openclaw.json` existe, choisissez **Conserver / Modifier / Réinitialiser**.
    - Relancer l’onboarding **n’efface rien** sauf si vous choisissez explicitement **Réinitialiser**
      (ou passez `--reset`).
    - La CLI `--reset` utilise par défaut `config+creds+sessions` ; utilisez `--reset-scope full`
      pour supprimer aussi l’espace de travail.
    - Si la configuration est invalide ou contient des clés héritées, l’assistant s’arrête et vous demande
      d’exécuter `openclaw doctor` avant de continuer.
    - La réinitialisation utilise `trash` (jamais `rm`) et propose les portées suivantes :
      - Configuration uniquement
      - Configuration + identifiants + sessions
      - Réinitialisation complète (supprime aussi l’espace de travail)
  </Step>
  <Step title="Modèle/Auth">
    - **Clé API Anthropic** : utilise `ANTHROPIC_API_KEY` si présent ou demande une clé, puis l’enregistre pour l’utilisation du daemon.
    - **Clé API Anthropic** : choix d’assistant Anthropic préféré dans onboarding/configure.
    - **Jeton de configuration Anthropic** : toujours disponible dans onboarding/configure, même si OpenClaw préfère maintenant réutiliser Claude CLI quand c’est possible.
    - **Abonnement OpenAI Code (Codex) (Codex CLI)** : si `~/.codex/auth.json` existe, l’onboarding peut le réutiliser. Les identifiants Codex CLI réutilisés restent gérés par Codex CLI ; à expiration, OpenClaw relit d’abord cette source et, lorsque le provider peut les rafraîchir, réécrit l’identifiant rafraîchi dans le stockage Codex au lieu d’en prendre lui-même la gestion.
    - **Abonnement OpenAI Code (Codex) (OAuth)** : flux navigateur ; collez le `code#state`.
      - Définit `agents.defaults.model` sur `openai-codex/gpt-5.4` lorsque le modèle n’est pas défini ou correspond à `openai/*`.
    - **Clé API OpenAI** : utilise `OPENAI_API_KEY` si présent ou demande une clé, puis la stocke dans les profils d’authentification.
      - Définit `agents.defaults.model` sur `openai/gpt-5.4` lorsque le modèle n’est pas défini, correspond à `openai/*` ou à `openai-codex/*`.
    - **Clé API xAI (Grok)** : demande `XAI_API_KEY` et configure xAI comme provider de modèle.
    - **OpenCode** : demande `OPENCODE_API_KEY` (ou `OPENCODE_ZEN_API_KEY`, à obtenir sur https://opencode.ai/auth) et vous laisse choisir le catalogue Zen ou Go.
    - **Ollama** : demande l’URL de base Ollama, propose le mode **Cloud + Local** ou **Local**, découvre les modèles disponibles et récupère automatiquement le modèle local sélectionné si nécessaire.
    - Plus de détails : [Ollama](/fr/providers/ollama)
    - **Clé API** : stocke la clé pour vous.
    - **Vercel AI Gateway (proxy multi-modèles)** : demande `AI_GATEWAY_API_KEY`.
    - Plus de détails : [Vercel AI Gateway](/fr/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway** : demande l’ID de compte, l’ID Gateway et `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - Plus de détails : [Cloudflare AI Gateway](/fr/providers/cloudflare-ai-gateway)
    - **MiniMax** : la configuration est écrite automatiquement ; la valeur hébergée par défaut est `MiniMax-M2.7`.
      La configuration par clé API utilise `minimax/...`, et la configuration OAuth utilise
      `minimax-portal/...`.
    - Plus de détails : [MiniMax](/fr/providers/minimax)
    - **StepFun** : la configuration est écrite automatiquement pour StepFun standard ou Step Plan sur les endpoints Chine ou globaux.
    - Standard inclut actuellement `step-3.5-flash`, et Step Plan inclut aussi `step-3.5-flash-2603`.
    - Plus de détails : [StepFun](/fr/providers/stepfun)
    - **Synthetic (compatible Anthropic)** : demande `SYNTHETIC_API_KEY`.
    - Plus de détails : [Synthetic](/fr/providers/synthetic)
    - **Moonshot (Kimi K2)** : la configuration est écrite automatiquement.
    - **Kimi Coding** : la configuration est écrite automatiquement.
    - Plus de détails : [Moonshot AI (Kimi + Kimi Coding)](/fr/providers/moonshot)
    - **Ignorer** : aucune authentification n’est encore configurée.
    - Choisissez un modèle par défaut parmi les options détectées (ou saisissez manuellement provider/model). Pour une meilleure qualité et un risque plus faible d’injection de prompt, choisissez le modèle de dernière génération le plus robuste disponible dans votre pile de providers.
    - L’onboarding exécute une vérification du modèle et avertit si le modèle configuré est inconnu ou si l’authentification est manquante.
    - Le mode de stockage des clés API utilise par défaut des valeurs de profil d’authentification en clair. Utilisez `--secret-input-mode ref` pour stocker à la place des références adossées à des variables d’environnement (par exemple `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - Les profils d’authentification se trouvent dans `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (clés API + OAuth). `~/.openclaw/credentials/oauth.json` est une source héritée réservée à l’importation.
    - Plus de détails : [/concepts/oauth](/fr/concepts/oauth)
    <Note>
    Astuce pour environnement headless/serveur : terminez OAuth sur une machine avec navigateur, puis copiez
    le `auth-profiles.json` de cet agent (par exemple
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, ou le chemin correspondant
    `$OPENCLAW_STATE_DIR/...`) vers l’hôte Gateway. `credentials/oauth.json`
    n’est qu’une source d’importation héritée.
    </Note>
  </Step>
  <Step title="Espace de travail">
    - Valeur par défaut `~/.openclaw/workspace` (configurable).
    - Initialise les fichiers d’espace de travail nécessaires au rituel d’amorçage de l’agent.
    - Disposition complète de l’espace de travail + guide de sauvegarde : [Espace de travail de l’agent](/fr/concepts/agent-workspace)
  </Step>
  <Step title="Gateway">
    - Port, mode de liaison, mode d’authentification, exposition Tailscale.
    - Recommandation d’authentification : conservez **Token** même pour loopback afin que les clients WS locaux doivent s’authentifier.
    - En mode token, la configuration interactive propose :
      - **Générer/stocker un token en clair** (par défaut)
      - **Utiliser SecretRef** (opt-in)
      - Le démarrage rapide réutilise les SecretRef `gateway.auth.token` existants via les providers `env`, `file` et `exec` pour la sonde d’onboarding/l’amorçage du tableau de bord.
      - Si ce SecretRef est configuré mais ne peut pas être résolu, l’onboarding échoue tôt avec un message de correction clair au lieu de dégrader silencieusement l’authentification d’exécution.
    - En mode mot de passe, la configuration interactive prend également en charge le stockage en clair ou SecretRef.
    - Chemin SecretRef pour token en mode non interactif : `--gateway-token-ref-env <ENV_VAR>`.
      - Nécessite une variable d’environnement non vide dans l’environnement du processus d’onboarding.
      - Ne peut pas être combiné avec `--gateway-token`.
    - Désactivez l’authentification uniquement si vous faites totalement confiance à chaque processus local.
    - Les liaisons hors loopback nécessitent toujours une authentification.
  </Step>
  <Step title="Canaux">
    - [WhatsApp](/fr/channels/whatsapp) : connexion QR facultative.
    - [Telegram](/fr/channels/telegram) : token de bot.
    - [Discord](/fr/channels/discord) : token de bot.
    - [Google Chat](/fr/channels/googlechat) : JSON de compte de service + audience webhook.
    - [Mattermost](/fr/channels/mattermost) (plugin) : token de bot + URL de base.
    - [Signal](/fr/channels/signal) : installation facultative de `signal-cli` + configuration du compte.
    - [BlueBubbles](/fr/channels/bluebubbles) : **recommandé pour iMessage** ; URL du serveur + mot de passe + webhook.
    - [iMessage](/fr/channels/imessage) : chemin CLI `imsg` hérité + accès à la base de données.
    - Sécurité des messages privés : le mode par défaut est l’appairage. Le premier message privé envoie un code ; approuvez-le via `openclaw pairing approve <channel> <code>` ou utilisez des listes d’autorisation.
  </Step>
  <Step title="Recherche web">
    - Choisissez un provider pris en charge tel que Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG ou Tavily (ou ignorez cette étape).
    - Les providers adossés à une API peuvent utiliser des variables d’environnement ou une configuration existante pour une configuration rapide ; les providers sans clé utilisent à la place leurs prérequis spécifiques.
    - Ignorez avec `--skip-search`.
    - Configurez plus tard : `openclaw configure --section web`.
  </Step>
  <Step title="Installation du daemon">
    - macOS : LaunchAgent
      - Nécessite une session utilisateur connectée ; pour un environnement headless, utilisez un LaunchDaemon personnalisé (non fourni).
    - Linux (et Windows via WSL2) : unité systemd utilisateur
      - L’onboarding tente d’activer le lingering via `loginctl enable-linger <user>` afin que la Gateway reste active après déconnexion.
      - Peut demander sudo (écrit dans `/var/lib/systemd/linger`) ; il essaie d’abord sans sudo.
    - **Sélection du runtime :** Node (recommandé ; requis pour WhatsApp/Telegram). Bun est **non recommandé**.
    - Si l’authentification par token nécessite un token et que `gateway.auth.token` est géré par SecretRef, l’installation du daemon le valide mais ne conserve pas les valeurs de token en clair résolues dans les métadonnées d’environnement du service supervisé.
    - Si l’authentification par token nécessite un token et que le SecretRef de token configuré n’est pas résolu, l’installation du daemon est bloquée avec des indications exploitables.
    - Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés et que `gateway.auth.mode` n’est pas défini, l’installation du daemon est bloquée jusqu’à ce que le mode soit défini explicitement.
  </Step>
  <Step title="Vérification de l’état de santé">
    - Démarre la Gateway (si nécessaire) et exécute `openclaw health`.
    - Astuce : `openclaw status --deep` ajoute la sonde de santé Gateway en direct à la sortie d’état, y compris les sondes de canal lorsqu’elles sont prises en charge (nécessite une Gateway joignable).
  </Step>
  <Step title="Skills (recommandé)">
    - Lit les Skills disponibles et vérifie les exigences.
    - Vous permet de choisir un gestionnaire de nœuds : **npm / pnpm** (bun non recommandé).
    - Installe les dépendances facultatives (certaines utilisent Homebrew sur macOS).
  </Step>
  <Step title="Fin">
    - Résumé + étapes suivantes, y compris les applications iOS/Android/macOS pour des fonctionnalités supplémentaires.
  </Step>
</Steps>

<Note>
Si aucune interface graphique n’est détectée, l’onboarding affiche les instructions de redirection de port SSH pour l’interface de contrôle au lieu d’ouvrir un navigateur.
Si les assets de l’interface de contrôle sont absents, l’onboarding tente de les construire ; la solution de repli est `pnpm ui:build` (installe automatiquement les dépendances UI).
</Note>

## Mode non interactif

Utilisez `--non-interactive` pour automatiser ou script l’onboarding :

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

Ajoutez `--json` pour un résumé lisible par machine.

Token Gateway SecretRef en mode non interactif :

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` et `--gateway-token-ref-env` sont mutuellement exclusifs.

<Note>
`--json` **n’implique pas** le mode non interactif. Utilisez `--non-interactive` (et `--workspace`) pour les scripts.
</Note>

Des exemples de commandes spécifiques aux providers sont disponibles dans [Automatisation CLI](/fr/start/wizard-cli-automation#provider-specific-examples).
Utilisez cette page de référence pour la sémantique des drapeaux et l’ordre des étapes.

### Ajouter un agent (mode non interactif)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.4 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## RPC de l’assistant Gateway

La Gateway expose le flux d’onboarding via RPC (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
Les clients (application macOS, interface de contrôle) peuvent afficher les étapes sans réimplémenter la logique d’onboarding.

## Configuration Signal (`signal-cli`)

L’onboarding peut installer `signal-cli` depuis les releases GitHub :

- Télécharge l’asset de release approprié.
- Le stocke sous `~/.openclaw/tools/signal-cli/<version>/`.
- Écrit `channels.signal.cliPath` dans votre configuration.

Remarques :

- Les builds JVM nécessitent **Java 21**.
- Les builds natifs sont utilisés lorsqu’ils sont disponibles.
- Windows utilise WSL2 ; l’installation de signal-cli suit le flux Linux à l’intérieur de WSL.

## Ce que l’assistant écrit

Champs typiques dans `~/.openclaw/openclaw.json` :

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (si Minimax est choisi)
- `tools.profile` (l’onboarding local utilise par défaut `"coding"` lorsqu’il n’est pas défini ; les valeurs explicites existantes sont préservées)
- `gateway.*` (mode, liaison, authentification, Tailscale)
- `session.dmScope` (détails du comportement : [Référence de configuration CLI](/fr/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Listes d’autorisation de canal (Slack/Discord/Matrix/Microsoft Teams) lorsque vous activez cette option pendant les invites (les noms sont résolus vers des identifiants lorsque c’est possible).
- `skills.install.nodeManager`
  - `setup --node-manager` accepte `npm`, `pnpm` ou `bun`.
  - La configuration manuelle peut toujours utiliser `yarn` en définissant directement `skills.install.nodeManager`.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` écrit `agents.list[]` et éventuellement `bindings`.

Les identifiants WhatsApp sont stockés sous `~/.openclaw/credentials/whatsapp/<accountId>/`.
Les sessions sont stockées sous `~/.openclaw/agents/<agentId>/sessions/`.

Certains canaux sont fournis sous forme de plugins. Lorsque vous en choisissez un pendant la configuration, l’onboarding
vous demandera de l’installer (npm ou chemin local) avant de pouvoir le configurer.

## Documentation associée

- Vue d’ensemble de l’onboarding : [Onboarding (CLI)](/fr/start/wizard)
- Onboarding de l’application macOS : [Onboarding](/fr/start/onboarding)
- Référence de configuration : [Configuration Gateway](/fr/gateway/configuration)
- Providers : [WhatsApp](/fr/channels/whatsapp), [Telegram](/fr/channels/telegram), [Discord](/fr/channels/discord), [Google Chat](/fr/channels/googlechat), [Signal](/fr/channels/signal), [BlueBubbles](/fr/channels/bluebubbles) (iMessage), [iMessage](/fr/channels/imessage) (hérité)
- Skills : [Skills](/fr/tools/skills), [Configuration Skills](/fr/tools/skills-config)
