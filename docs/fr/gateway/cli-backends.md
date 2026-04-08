---
read_when:
    - Vous voulez une solution de repli fiable lorsque les fournisseurs d’API échouent
    - Vous utilisez Codex CLI ou d’autres AI CLI locaux et souhaitez les réutiliser
    - Vous voulez comprendre le pont MCP en local loopback pour l’accès aux outils du backend CLI
summary: 'Backends CLI : solution locale de repli pour AI CLI avec pont d’outils MCP facultatif'
title: Backends CLI
x-i18n:
    generated_at: "2026-04-08T02:14:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: b0e8c41f5f5a8e34466f6b765e5c08585ef1788fa9e9d953257324bcc6cbc414
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends CLI (runtime de repli)

OpenClaw peut exécuter des **AI CLI locaux** comme **solution de repli en texte seul** lorsque les fournisseurs d’API sont indisponibles,
limitent le débit ou se comportent temporairement mal. Cette approche est volontairement prudente :

- **Les outils OpenClaw ne sont pas injectés directement**, mais les backends avec `bundleMcp: true`
  peuvent recevoir les outils Gateway via un pont MCP en local loopback.
- **Streaming JSONL** pour les CLI qui le prennent en charge.
- **Les sessions sont prises en charge** (afin que les tours de suivi restent cohérents).
- **Les images peuvent être transmises** si le CLI accepte des chemins d’image.

Cela est conçu comme un **filet de sécurité** plutôt qu’un chemin principal. Utilisez-le lorsque vous
voulez des réponses textuelles « qui fonctionnent toujours » sans dépendre d’API externes.

Si vous voulez un runtime harness complet avec contrôles de session ACP, tâches en arrière-plan,
liaison de fil/conversation et sessions de code externes persistantes, utilisez
[ACP Agents](/fr/tools/acp-agents) à la place. Les backends CLI ne sont pas ACP.

## Démarrage rapide pour débutants

Vous pouvez utiliser Codex CLI **sans aucune configuration** (le plugin OpenAI intégré
enregistre un backend par défaut) :

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Si votre Gateway s’exécute sous launchd/systemd et que PATH est minimal, ajoutez simplement le
chemin de la commande :

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

C’est tout. Aucune clé, aucune configuration d’authentification supplémentaire n’est nécessaire au-delà du CLI lui-même.

Si vous utilisez un backend CLI intégré comme **fournisseur de messages principal** sur un
hôte Gateway, OpenClaw charge désormais automatiquement le plugin intégré propriétaire lorsque votre configuration
référence explicitement ce backend dans une référence de modèle ou sous
`agents.defaults.cliBackends`.

## L’utiliser comme solution de repli

Ajoutez un backend CLI à votre liste de repli afin qu’il ne s’exécute que lorsque les modèles principaux échouent :

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

Remarques :

- Si vous utilisez `agents.defaults.models` (liste d’autorisation), vous devez y inclure aussi les modèles de vos backends CLI.
- Si le fournisseur principal échoue (authentification, limites de débit, délais d’attente), OpenClaw
  essaiera ensuite le backend CLI.

## Aperçu de la configuration

Tous les backends CLI se trouvent sous :

```
agents.defaults.cliBackends
```

Chaque entrée est indexée par un **id de fournisseur** (par exemple `codex-cli`, `my-cli`).
L’id du fournisseur devient la partie gauche de votre référence de modèle :

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
2. **Construit un prompt système** en utilisant le même prompt OpenClaw + contexte d’espace de travail.
3. **Exécute le CLI** avec un id de session (si pris en charge) afin que l’historique reste cohérent.
4. **Analyse la sortie** (JSON ou texte brut) et renvoie le texte final.
5. **Persiste les id de session** par backend, afin que les suivis réutilisent la même session CLI.

<Note>
Le backend intégré Anthropic `claude-cli` est de nouveau pris en charge. Le personnel d’Anthropic
nous a indiqué que l’usage de Claude CLI dans le style OpenClaw est de nouveau autorisé, donc OpenClaw considère
l’utilisation de `claude -p` comme approuvée pour cette intégration, sauf si Anthropic publie
une nouvelle politique.
</Note>

## Sessions

- Si le CLI prend en charge les sessions, définissez `sessionArg` (par exemple `--session-id`) ou
  `sessionArgs` (espace réservé `{sessionId}`) lorsque l’id doit être inséré
  dans plusieurs drapeaux.
- Si le CLI utilise une **sous-commande de reprise** avec des drapeaux différents, définissez
  `resumeArgs` (remplace `args` lors de la reprise) et éventuellement `resumeOutput`
  (pour les reprises non JSON).
- `sessionMode` :
  - `always` : envoie toujours un id de session (nouvel UUID si aucun n’est stocké).
  - `existing` : envoie un id de session uniquement si un id a déjà été stocké.
  - `none` : n’envoie jamais d’id de session.

Remarques sur la sérialisation :

- `serialize: true` conserve l’ordre des exécutions sur la même voie.
- La plupart des CLI sérialisent sur une voie de fournisseur.
- OpenClaw abandonne la réutilisation de session CLI stockée lorsque l’état d’authentification du backend change, y compris lors d’une reconnexion, d’une rotation de jeton ou d’un changement d’identifiant de profil d’authentification.

## Images (transmission directe)

Si votre CLI accepte des chemins d’image, définissez `imageArg` :

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw écrira les images base64 dans des fichiers temporaires. Si `imageArg` est défini, ces
chemins sont transmis comme arguments CLI. Si `imageArg` est absent, OpenClaw ajoute
les chemins de fichiers au prompt (injection de chemin), ce qui suffit pour les CLI qui chargent automatiquement
les fichiers locaux à partir de chemins bruts.

## Entrées / sorties

- `output: "json"` (par défaut) essaie d’analyser le JSON et d’extraire le texte + l’id de session.
- Pour la sortie JSON de Gemini CLI, OpenClaw lit le texte de réponse depuis `response` et
  l’utilisation depuis `stats` lorsque `usage` est absent ou vide.
- `output: "jsonl"` analyse les flux JSONL (par exemple Codex CLI `--json`) et extrait le message final de l’agent ainsi que les identifiants de session
  lorsqu’ils sont présents.
- `output: "text"` traite stdout comme la réponse finale.

Modes d’entrée :

- `input: "arg"` (par défaut) transmet le prompt comme dernier argument CLI.
- `input: "stdin"` envoie le prompt via stdin.
- Si le prompt est très long et que `maxPromptArgChars` est défini, stdin est utilisé.

## Valeurs par défaut (propriété du plugin)

Le plugin OpenAI intégré enregistre aussi une valeur par défaut pour `codex-cli` :

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Le plugin Google intégré enregistre aussi une valeur par défaut pour `google-gemini-cli` :

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Prérequis : le Gemini CLI local doit être installé et disponible comme
`gemini` sur le `PATH` (`brew install gemini-cli` ou
`npm install -g @google/gemini-cli`).

Remarques sur le JSON de Gemini CLI :

- Le texte de réponse est lu depuis le champ JSON `response`.
- L’utilisation bascule sur `stats` lorsque `usage` est absent ou vide.
- `stats.cached` est normalisé en `cacheRead` OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de
  `stats.input_tokens - stats.cached`.

Ne remplacez ces valeurs que si nécessaire (cas courant : chemin `command` absolu).

## Valeurs par défaut propriété du plugin

Les valeurs par défaut des backends CLI font désormais partie de la surface du plugin :

- Les plugins les enregistrent avec `api.registerCliBackend(...)`.
- L’`id` du backend devient le préfixe du fournisseur dans les références de modèle.
- La configuration utilisateur dans `agents.defaults.cliBackends.<id>` continue de remplacer la valeur par défaut du plugin.
- Le nettoyage de configuration spécifique au backend reste propriété du plugin via le hook facultatif
  `normalizeConfig`.

## Superpositions bundle MCP

Les backends CLI **ne** reçoivent **pas** directement les appels d’outils OpenClaw, mais un backend peut
opter pour une superposition de configuration MCP générée avec `bundleMcp: true`.

Comportement intégré actuel :

- `claude-cli` : fichier de configuration MCP strict généré
- `codex-cli` : remplacements de configuration en ligne pour `mcp_servers`
- `google-gemini-cli` : fichier de paramètres système Gemini généré

Lorsque bundle MCP est activé, OpenClaw :

- lance un serveur MCP HTTP en loopback qui expose les outils Gateway au processus CLI
- authentifie le pont avec un jeton par session (`OPENCLAW_MCP_TOKEN`)
- limite l’accès aux outils à la session, au compte et au contexte de canal actuels
- charge les serveurs bundle-MCP activés pour l’espace de travail actuel
- les fusionne avec toute configuration ou forme de paramètres MCP backend existante
- réécrit la configuration de lancement en utilisant le mode d’intégration propriété du backend depuis l’extension propriétaire

Si aucun serveur MCP n’est activé, OpenClaw injecte tout de même une configuration stricte lorsqu’un
backend opte pour bundle MCP afin que les exécutions en arrière-plan restent isolées.

## Limitations

- **Aucun appel direct aux outils OpenClaw.** OpenClaw n’injecte pas d’appels d’outils dans
  le protocole du backend CLI. Les backends ne voient les outils Gateway que lorsqu’ils optent pour
  `bundleMcp: true`.
- **Le streaming dépend du backend.** Certains backends diffusent du JSONL ; d’autres mettent en mémoire tampon
  jusqu’à la fin.
- **Les sorties structurées** dépendent du format JSON du CLI.
- **Les sessions Codex CLI** reprennent via une sortie texte (pas JSONL), ce qui est moins
  structuré que l’exécution initiale `--json`. Les sessions OpenClaw continuent malgré tout de fonctionner
  normalement.

## Dépannage

- **CLI introuvable** : définissez `command` sur un chemin complet.
- **Nom de modèle incorrect** : utilisez `modelAliases` pour mapper `provider/model` → modèle CLI.
- **Pas de continuité de session** : assurez-vous que `sessionArg` est défini et que `sessionMode` n’est pas
  `none` (Codex CLI ne peut actuellement pas reprendre avec une sortie JSON).
- **Images ignorées** : définissez `imageArg` (et vérifiez que le CLI prend en charge les chemins de fichiers).
