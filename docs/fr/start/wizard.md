---
read_when:
    - Exécuter ou configurer l'onboarding CLI
    - Configurer une nouvelle machine
sidebarTitle: 'Onboarding: CLI'
summary: 'Onboarding CLI : configuration guidée de la gateway, du workspace, des canaux et des Skills'
title: Onboarding (CLI)
x-i18n:
    generated_at: "2026-04-07T06:54:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6773b07afa8babf1b5ac94d857063d08094a962ee21ec96ca966e99ad57d107d
    source_path: start/wizard.md
    workflow: 15
---

# Onboarding (CLI)

L'onboarding CLI est la méthode **recommandée** pour configurer OpenClaw sur macOS,
Linux ou Windows (via WSL2, fortement recommandé).
Il configure une Gateway locale ou une connexion à une Gateway distante, ainsi que les canaux, les Skills
et les valeurs par défaut du workspace dans un flux guidé unique.

```bash
openclaw onboard
```

<Info>
Le moyen le plus rapide d'obtenir une première conversation : ouvrez le Control UI (aucune configuration de canal nécessaire). Exécutez
`openclaw dashboard` et discutez dans le navigateur. Documentation : [Dashboard](/web/dashboard).
</Info>

Pour reconfigurer plus tard :

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` n'implique pas le mode non interactif. Pour les scripts, utilisez `--non-interactive`.
</Note>

<Tip>
L'onboarding CLI inclut une étape de recherche web dans laquelle vous pouvez choisir un fournisseur
tel que Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search,
Ollama Web Search, Perplexity, SearXNG ou Tavily. Certains fournisseurs nécessitent une
clé API, tandis que d'autres n'en nécessitent pas. Vous pouvez aussi configurer cela plus tard avec
`openclaw configure --section web`. Documentation : [Outils web](/fr/tools/web).
</Tip>

## QuickStart vs Avancé

L'onboarding commence avec **QuickStart** (valeurs par défaut) ou **Avancé** (contrôle complet).

<Tabs>
  <Tab title="QuickStart (valeurs par défaut)">
    - Gateway locale (loopback)
    - Workspace par défaut (ou workspace existant)
    - Port de Gateway **18789**
    - Authentification Gateway **Token** (généré automatiquement, même en loopback)
    - Politique d'outil par défaut pour les nouvelles configurations locales : `tools.profile: "coding"` (un profil explicite existant est conservé)
    - Portée DM par défaut : l'onboarding local écrit `session.dmScope: "per-channel-peer"` lorsqu'elle n'est pas définie. Détails : [Référence de configuration CLI](/fr/start/wizard-cli-reference#outputs-and-internals)
    - Exposition Tailscale **désactivée**
    - Les DM Telegram + WhatsApp utilisent par défaut une **liste d'autorisation** (votre numéro de téléphone vous sera demandé)
  </Tab>
  <Tab title="Avancé (contrôle complet)">
    - Expose chaque étape (mode, workspace, gateway, canaux, daemon, Skills).
  </Tab>
</Tabs>

## Ce que l'onboarding configure

Le **mode local (par défaut)** vous guide à travers ces étapes :

1. **Modèle/Auth** — choisissez n'importe quel flux fournisseur/authentification pris en charge (clé API, OAuth ou authentification manuelle spécifique au fournisseur), y compris Custom Provider
   (compatible OpenAI, compatible Anthropic ou détection automatique Unknown). Choisissez un modèle par défaut.
   Remarque de sécurité : si cet agent doit exécuter des outils ou traiter du contenu de webhook/hooks, privilégiez le modèle de dernière génération le plus robuste disponible et gardez une politique d'outils stricte. Les niveaux plus faibles ou plus anciens sont plus faciles à injecter par prompt.
   Pour les exécutions non interactives, `--secret-input-mode ref` stocke des références adossées à l'environnement dans les profils d'authentification au lieu de valeurs de clé API en clair.
   En mode `ref` non interactif, la variable d'environnement du fournisseur doit être définie ; passer des drapeaux de clé inline sans cette variable d'environnement échoue immédiatement.
   En mode interactif, choisir le mode de référence secrète vous permet de pointer soit vers une variable d'environnement soit vers une référence fournisseur configurée (`file` ou `exec`), avec une validation préalable rapide avant l'enregistrement.
   Pour Anthropic, l'onboarding/configure interactif propose **Anthropic Claude CLI** comme chemin local privilégié et **Anthropic API key** comme chemin de production recommandé. Anthropic setup-token reste également disponible comme chemin pris en charge pour l'authentification par jeton.
2. **Workspace** — emplacement des fichiers d'agent (par défaut `~/.openclaw/workspace`). Initialise les fichiers bootstrap.
3. **Gateway** — port, adresse de liaison, mode d'authentification, exposition Tailscale.
   En mode jeton interactif, choisissez le stockage par défaut du jeton en clair ou optez pour SecretRef.
   Chemin SecretRef du jeton en mode non interactif : `--gateway-token-ref-env <ENV_VAR>`.
4. **Canaux** — canaux de chat intégrés et en bundle tels que BlueBubbles, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp, etc.
5. **Daemon** — installe un LaunchAgent (macOS), une unité utilisateur systemd (Linux/WSL2) ou une tâche planifiée native Windows avec repli par dossier Startup propre à l'utilisateur.
   Si l'authentification par jeton nécessite un jeton et que `gateway.auth.token` est géré par SecretRef, l'installation du daemon le valide mais ne conserve pas le jeton résolu dans les métadonnées d'environnement du service superviseur.
   Si l'authentification par jeton nécessite un jeton et que le SecretRef de jeton configuré n'est pas résolu, l'installation du daemon est bloquée avec des indications exploitables.
   Si `gateway.auth.token` et `gateway.auth.password` sont tous deux configurés et que `gateway.auth.mode` n'est pas défini, l'installation du daemon est bloquée jusqu'à ce que le mode soit défini explicitement.
6. **Vérification d'état** — démarre la Gateway et vérifie qu'elle fonctionne.
7. **Skills** — installe les Skills recommandés et les dépendances facultatives.

<Note>
Relancer l'onboarding **n'efface rien** sauf si vous choisissez explicitement **Reset** (ou passez `--reset`).
CLI `--reset` cible par défaut la configuration, les identifiants et les sessions ; utilisez `--reset-scope full` pour inclure le workspace.
Si la configuration est invalide ou contient des clés héritées, l'onboarding vous demande d'abord d'exécuter `openclaw doctor`.
</Note>

Le **mode distant** configure uniquement le client local pour qu'il se connecte à une Gateway ailleurs.
Il **n'installe pas** et ne modifie rien sur l'hôte distant.

## Ajouter un autre agent

Utilisez `openclaw agents add <name>` pour créer un agent séparé avec son propre workspace,
ses propres sessions et profils d'authentification. L'exécution sans `--workspace` lance l'onboarding.

Ce que cela définit :

- `agents.list[].name`
- `agents.list[].workspace`
- `agents.list[].agentDir`

Remarques :

- Les workspaces par défaut suivent `~/.openclaw/workspace-<agentId>`.
- Ajoutez `bindings` pour router les messages entrants (l'onboarding peut le faire).
- Drapeaux non interactifs : `--model`, `--agent-dir`, `--bind`, `--non-interactive`.

## Référence complète

Pour des descriptions détaillées étape par étape et les sorties de configuration, voir
[Référence de configuration CLI](/fr/start/wizard-cli-reference).
Pour des exemples non interactifs, voir [Automatisation CLI](/fr/start/wizard-cli-automation).
Pour la référence technique approfondie, y compris les détails RPC, voir
[Référence de l'onboarding](/fr/reference/wizard).

## Documentation associée

- Référence des commandes CLI : [`openclaw onboard`](/cli/onboard)
- Vue d'ensemble de l'onboarding : [Vue d'ensemble de l'onboarding](/fr/start/onboarding-overview)
- Onboarding de l'application macOS : [Onboarding](/fr/start/onboarding)
- Rituel de premier lancement d'agent : [Initialisation de l'agent](/fr/start/bootstrapping)
