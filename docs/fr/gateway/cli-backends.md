---
read_when:
    - Vous voulez une solution de repli fiable lorsque les fournisseurs d’API échouent
    - Vous utilisez Codex CLI ou d’autres CLI d’IA locales et voulez les réutiliser
    - Vous voulez comprendre le pont MCP loopback pour l’accès aux outils du backend CLI
summary: 'Backends CLI : solution de repli locale via des CLI d’IA avec pont d’outils MCP facultatif'
title: Backends CLI
x-i18n:
    generated_at: "2026-04-07T06:49:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: f061357f420455ad6ffaabe7fe28f1fb1b1769d73a4eb2e6f45c6eb3c2e36667
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends CLI (runtime de repli)

OpenClaw peut exécuter des **CLI d’IA locales** comme **solution de repli en texte seul** lorsque les fournisseurs d’API sont indisponibles,
limités en débit ou temporairement défaillants. Cette approche est volontairement prudente :

- **Les outils OpenClaw ne sont pas injectés directement**, mais les backends avec `bundleMcp: true`
  peuvent recevoir les outils de la gateway via un pont MCP loopback.
- **Streaming JSONL** pour les CLI qui le prennent en charge.
- **Les sessions sont prises en charge** (les tours suivants restent donc cohérents).
- **Les images peuvent être transmises** si la CLI accepte des chemins d’images.

Cela est conçu comme un **filet de sécurité** plutôt que comme un chemin principal. Utilisez-le lorsque vous
voulez des réponses textuelles « qui fonctionnent toujours » sans dépendre d’API externes.

Si vous voulez un runtime avec une architecture complète incluant les contrôles de session ACP, les tâches en arrière-plan,
la liaison fil/conversation et des sessions de codage externes persistantes, utilisez plutôt
[ACP Agents](/fr/tools/acp-agents). Les backends CLI ne sont pas ACP.

## Démarrage rapide pour débutants

Vous pouvez utiliser Codex CLI **sans aucune configuration** (le plugin OpenAI intégré
enregistre un backend par défaut) :

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Si votre gateway s’exécute sous launchd/systemd et que `PATH` est minimal, ajoutez simplement le
chemin de la commande :

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

C’est tout. Aucune clé ni configuration d’authentification supplémentaire n’est nécessaire au-delà de la CLI elle-même.

Si vous utilisez un backend CLI intégré comme **fournisseur principal de messages** sur un
hôte gateway, OpenClaw charge désormais automatiquement le plugin intégré propriétaire lorsque votre configuration
référence explicitement ce backend dans une référence de modèle ou sous
`agents.defaults.cliBackends`.

## L’utiliser comme solution de repli

Ajoutez un backend CLI à votre liste de repli afin qu’il ne s’exécute que lorsque les modèles principaux échouent :

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

Remarques :

- Si vous utilisez `agents.defaults.models` (liste d’autorisation), vous devez aussi y inclure les modèles de votre backend CLI.
- Si le fournisseur principal échoue (authentification, limites de débit, délais d’attente), OpenClaw
  essaiera ensuite le backend CLI.

## Vue d’ensemble de la configuration

Tous les backends CLI se trouvent sous :

```
agents.defaults.cliBackends
```

Chaque entrée est indexée par un **id de fournisseur** (par ex. `codex-cli`, `my-cli`).
L’id de fournisseur devient la partie gauche de votre référence de modèle :

```
<provider>/<model>
```

### Exemple de configuration

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
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## Fonctionnement

1. **Sélectionne un backend** selon le préfixe du fournisseur (`codex-cli/...`).
2. **Construit un prompt système** en utilisant le même prompt OpenClaw + le même contexte d’espace de travail.
3. **Exécute la CLI** avec un id de session (si pris en charge) afin que l’historique reste cohérent.
4. **Analyse la sortie** (JSON ou texte brut) et renvoie le texte final.
5. **Conserve les ids de session** par backend afin que les suivis réutilisent la même session CLI.

<Note>
Le backend `claude-cli` intégré d’Anthropic est de nouveau pris en charge. Le personnel Anthropic
nous a indiqué que l’utilisation de Claude CLI dans le style OpenClaw est de nouveau autorisée, donc OpenClaw traite
l’utilisation de `claude -p` comme approuvée pour cette intégration, sauf si Anthropic publie
une nouvelle politique.
</Note>

## Sessions

- Si la CLI prend en charge les sessions, définissez `sessionArg` (par ex. `--session-id`) ou
  `sessionArgs` (espace réservé `{sessionId}`) lorsque l’id doit être inséré
  dans plusieurs drapeaux.
- Si la CLI utilise une **sous-commande de reprise** avec des drapeaux différents, définissez
  `resumeArgs` (remplace `args` lors de la reprise) et éventuellement `resumeOutput`
  (pour les reprises non JSON).
- `sessionMode` :
  - `always` : envoie toujours un id de session (nouvel UUID si aucun n’est stocké).
  - `existing` : envoie un id de session uniquement si un id a déjà été stocké.
  - `none` : n’envoie jamais d’id de session.

Remarques sur la sérialisation :

- `serialize: true` maintient l’ordre des exécutions sur la même voie.
- La plupart des CLI sérialisent sur une voie fournisseur.
- OpenClaw abandonne la réutilisation des sessions CLI stockées lorsque l’état d’authentification du backend change, y compris après reconnexion, rotation de jeton ou changement d’identifiant de profil d’authentification.

## Images (transmission directe)

Si votre CLI accepte des chemins d’images, définissez `imageArg` :

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw écrira les images base64 dans des fichiers temporaires. Si `imageArg` est défini, ces
chemins sont passés comme arguments de CLI. Si `imageArg` est absent, OpenClaw ajoute les
chemins de fichiers au prompt (injection de chemin), ce qui suffit pour les CLI qui chargent automatiquement
les fichiers locaux à partir de chemins simples.

## Entrées / sorties

- `output: "json"` (par défaut) essaie d’analyser le JSON et d’extraire le texte + l’id de session.
- Pour la sortie JSON de Gemini CLI, OpenClaw lit le texte de réponse depuis `response` et
  l’utilisation depuis `stats` lorsque `usage` est absent ou vide.
- `output: "jsonl"` analyse les flux JSONL (par exemple Codex CLI `--json`) et extrait le message final de l’agent ainsi que les identifiants de session
  lorsqu’ils sont présents.
- `output: "text"` traite stdout comme réponse finale.

Modes d’entrée :

- `input: "arg"` (par défaut) transmet le prompt comme dernier argument de la CLI.
- `input: "stdin"` envoie le prompt via stdin.
- Si le prompt est très long et que `maxPromptArgChars` est défini, stdin est utilisé.

## Valeurs par défaut (propriété du plugin)

Le plugin OpenAI intégré enregistre aussi une valeur par défaut pour `codex-cli` :

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Le plugin Google intégré enregistre aussi une valeur par défaut pour `google-gemini-cli` :

- `command: "gemini"`
- `args: ["--prompt", "--output-format", "json"]`
- `resumeArgs: ["--resume", "{sessionId}", "--prompt", "--output-format", "json"]`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Prérequis : la CLI Gemini locale doit être installée et disponible comme
`gemini` dans `PATH` (`brew install gemini-cli` ou
`npm install -g @google/gemini-cli`).

Remarques sur le JSON de Gemini CLI :

- Le texte de réponse est lu depuis le champ JSON `response`.
- L’utilisation revient à `stats` lorsque `usage` est absent ou vide.
- `stats.cached` est normalisé en `cacheRead` dans OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de
  `stats.input_tokens - stats.cached`.

Ne remplacez ces valeurs que si nécessaire (cas courant : chemin `command` absolu).

## Valeurs par défaut possédées par le plugin

Les valeurs par défaut des backends CLI font désormais partie de la surface du plugin :

- Les plugins les enregistrent avec `api.registerCliBackend(...)`.
- L’`id` du backend devient le préfixe fournisseur dans les références de modèle.
- La configuration utilisateur dans `agents.defaults.cliBackends.<id>` remplace toujours la valeur par défaut du plugin.
- Le nettoyage de configuration spécifique au backend reste géré par le plugin via le hook facultatif
  `normalizeConfig`.

## Overlays Bundle MCP

Les backends CLI **ne** reçoivent **pas** directement les appels d’outils OpenClaw, mais un backend peut
activer une overlay de configuration MCP générée avec `bundleMcp: true`.

Comportement intégré actuel :

- `codex-cli` : pas d’overlay Bundle MCP
- `google-gemini-cli` : pas d’overlay Bundle MCP

Lorsque Bundle MCP est activé, OpenClaw :

- lance un serveur HTTP MCP loopback qui expose les outils de la gateway au processus CLI
- authentifie le pont avec un jeton par session (`OPENCLAW_MCP_TOKEN`)
- limite l’accès aux outils au contexte actuel de session, de compte et de canal
- charge les serveurs bundle-MCP activés pour l’espace de travail actuel
- les fusionne avec tout `--mcp-config` existant du backend
- réécrit les arguments CLI pour transmettre `--strict-mcp-config --mcp-config <generated-file>`

Si aucun serveur MCP n’est activé, OpenClaw injecte tout de même une configuration stricte lorsqu’un
backend active Bundle MCP afin que les exécutions en arrière-plan restent isolées.

## Limitations

- **Pas d’appels directs aux outils OpenClaw.** OpenClaw n’injecte pas les appels d’outils dans
  le protocole du backend CLI. Les backends ne voient les outils de la gateway que lorsqu’ils activent
  `bundleMcp: true`.
- **Le streaming dépend du backend.** Certains backends diffusent en JSONL ; d’autres mettent en tampon
  jusqu’à la fin.
- **Les sorties structurées** dépendent du format JSON de la CLI.
- **Les sessions Codex CLI** reprennent via une sortie texte (pas de JSONL), ce qui est moins
  structuré que l’exécution initiale `--json`. Les sessions OpenClaw continuent de fonctionner
  normalement.

## Dépannage

- **CLI introuvable** : définissez `command` avec un chemin complet.
- **Nom de modèle incorrect** : utilisez `modelAliases` pour mapper `provider/model` → modèle CLI.
- **Pas de continuité de session** : assurez-vous que `sessionArg` est défini et que `sessionMode` n’est pas
  `none` (Codex CLI ne peut actuellement pas reprendre avec une sortie JSON).
- **Images ignorées** : définissez `imageArg` (et vérifiez que la CLI prend en charge les chemins de fichiers).
