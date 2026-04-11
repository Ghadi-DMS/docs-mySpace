---
read_when:
    - Vous voulez une solution de secours fiable lorsque les fournisseurs d’API échouent
    - Vous utilisez Codex CLI ou d’autres CLI d’IA locaux et souhaitez les réutiliser
    - Vous souhaitez comprendre le pont MCP en loopback pour l’accès aux outils des backends CLI
summary: 'Backends CLI : fallback CLI d’IA locale avec pont d’outil MCP optionnel'
title: Backends CLI
x-i18n:
    generated_at: "2026-04-11T02:44:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: d108dbea043c260a80d15497639298f71a6b4d800f68d7b39bc129f7667ca608
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends CLI (runtime de secours)

OpenClaw peut exécuter des **CLI d’IA locaux** comme **solution de secours en texte seul** lorsque les fournisseurs d’API sont indisponibles,
soumis à des limites de débit, ou se comportent temporairement mal. Cette approche est volontairement prudente :

- **Les outils OpenClaw ne sont pas injectés directement**, mais les backends avec `bundleMcp: true`
  peuvent recevoir les outils de la gateway via un pont MCP en loopback.
- **Streaming JSONL** pour les CLI qui le prennent en charge.
- **Les sessions sont prises en charge** (pour que les tours de suivi restent cohérents).
- **Les images peuvent être transmises** si le CLI accepte des chemins d’image.

Cela est conçu comme un **filet de sécurité** plutôt qu’un chemin principal. Utilisez-le lorsque vous
voulez des réponses textuelles « qui fonctionnent toujours » sans dépendre d’API externes.

Si vous voulez un runtime de harnais complet avec contrôles de session ACP, tâches en arrière-plan,
liaison thread/conversation, et sessions de codage externes persistantes, utilisez
[ACP Agents](/fr/tools/acp-agents) à la place. Les backends CLI ne sont pas ACP.

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

C’est tout. Aucune clé, aucune configuration d’authentification supplémentaire n’est nécessaire au-delà du CLI lui-même.

Si vous utilisez un backend CLI intégré comme **fournisseur de messages principal** sur un
hôte gateway, OpenClaw charge désormais automatiquement le plugin intégré propriétaire lorsque votre configuration
référence explicitement ce backend dans une référence de modèle ou sous
`agents.defaults.cliBackends`.

## L’utiliser comme solution de secours

Ajoutez un backend CLI à votre liste de solutions de secours pour qu’il ne s’exécute que lorsque les modèles principaux échouent :

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

- Si vous utilisez `agents.defaults.models` (liste d’autorisation), vous devez également y inclure les modèles de votre backend CLI.
- Si le fournisseur principal échoue (authentification, limites de débit, délais d’expiration), OpenClaw
  essaiera ensuite le backend CLI.

## Vue d’ensemble de la configuration

Tous les backends CLI se trouvent sous :

```
agents.defaults.cliBackends
```

Chaque entrée est indexée par un **id de fournisseur** (par exemple `codex-cli`, `my-cli`).
L’id de fournisseur devient le côté gauche de votre référence de modèle :

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
          // Les CLI de style Codex peuvent pointer vers un fichier de prompt à la place :
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
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

1. **Sélectionne un backend** en fonction du préfixe du fournisseur (`codex-cli/...`).
2. **Construit un prompt système** en utilisant le même prompt OpenClaw + contexte d’espace de travail.
3. **Exécute le CLI** avec un id de session (si pris en charge) afin que l’historique reste cohérent.
4. **Analyse la sortie** (JSON ou texte brut) et renvoie le texte final.
5. **Persiste les ids de session** par backend, afin que les suivis réutilisent la même session CLI.

<Note>
Le backend intégré Anthropic `claude-cli` est de nouveau pris en charge. Le personnel d’Anthropic
nous a indiqué que l’usage de Claude CLI dans le style OpenClaw est de nouveau autorisé, donc OpenClaw traite
l’usage de `claude -p` comme approuvé pour cette intégration, à moins qu’Anthropic ne publie
une nouvelle politique.
</Note>

Le backend intégré OpenAI `codex-cli` transmet le prompt système d’OpenClaw via
la surcharge de configuration `model_instructions_file` de Codex (`-c
model_instructions_file="..."`). Codex n’expose pas d’option de type Claude
`--append-system-prompt`, donc OpenClaw écrit le prompt assemblé dans un
fichier temporaire pour chaque nouvelle session Codex CLI.

Le backend intégré Anthropic `claude-cli` reçoit l’instantané des Skills OpenClaw
de deux façons : le catalogue compact de Skills OpenClaw dans le prompt système ajouté, et
un plugin Claude Code temporaire transmis avec `--plugin-dir`. Le plugin contient
uniquement les Skills éligibles pour cet agent/cette session, afin que le résolveur natif de Skills de Claude Code
voie le même ensemble filtré que celui qu’OpenClaw annoncerait autrement dans
le prompt. Les surcharges d’environnement/de clé API des Skills sont toujours appliquées par OpenClaw à l’environnement
du processus enfant pour l’exécution.

## Sessions

- Si le CLI prend en charge les sessions, définissez `sessionArg` (par exemple `--session-id`) ou
  `sessionArgs` (espace réservé `{sessionId}`) lorsque l’id doit être inséré
  dans plusieurs options.
- Si le CLI utilise une **sous-commande de reprise** avec des options différentes, définissez
  `resumeArgs` (remplace `args` lors de la reprise) et éventuellement `resumeOutput`
  (pour les reprises non JSON).
- `sessionMode` :
  - `always` : envoie toujours un id de session (nouvel UUID si aucun n’est stocké).
  - `existing` : envoie un id de session uniquement si un id a déjà été stocké.
  - `none` : n’envoie jamais d’id de session.

Remarques sur la sérialisation :

- `serialize: true` maintient l’ordre des exécutions sur la même voie.
- La plupart des CLI sérialisent sur une voie de fournisseur.
- OpenClaw abandonne la réutilisation d’une session CLI stockée lorsque l’état d’authentification du backend change, y compris en cas de reconnexion, rotation de jeton, ou modification d’un identifiant de profil d’authentification.

## Images (transmission directe)

Si votre CLI accepte des chemins d’image, définissez `imageArg` :

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw écrira les images base64 dans des fichiers temporaires. Si `imageArg` est défini, ces
chemins sont transmis comme arguments CLI. Si `imageArg` est absent, OpenClaw ajoute
les chemins de fichiers au prompt (injection de chemin), ce qui suffit pour les CLI qui chargent automatiquement
les fichiers locaux à partir de simples chemins.

## Entrées / sorties

- `output: "json"` (par défaut) essaie d’analyser le JSON et d’extraire le texte + l’id de session.
- Pour la sortie JSON de Gemini CLI, OpenClaw lit le texte de réponse depuis `response` et
  l’usage depuis `stats` lorsque `usage` est absent ou vide.
- `output: "jsonl"` analyse les flux JSONL (par exemple Codex CLI `--json`) et extrait le message final de l’agent ainsi que les
  identifiants de session lorsqu’ils sont présents.
- `output: "text"` traite stdout comme réponse finale.

Modes d’entrée :

- `input: "arg"` (par défaut) transmet le prompt comme dernier argument CLI.
- `input: "stdin"` envoie le prompt via stdin.
- Si le prompt est très long et que `maxPromptArgChars` est défini, stdin est utilisé.

## Valeurs par défaut (propriété du plugin)

Le plugin OpenAI intégré enregistre également une valeur par défaut pour `codex-cli` :

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Le plugin Google intégré enregistre également une valeur par défaut pour `google-gemini-cli` :

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Prérequis : le Gemini CLI local doit être installé et disponible sous le nom
`gemini` dans `PATH` (`brew install gemini-cli` ou
`npm install -g @google/gemini-cli`).

Remarques sur le JSON de Gemini CLI :

- Le texte de réponse est lu depuis le champ JSON `response`.
- L’usage retombe sur `stats` lorsque `usage` est absent ou vide.
- `stats.cached` est normalisé en `cacheRead` OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de
  `stats.input_tokens - stats.cached`.

Ne surchargez que si nécessaire (cas courant : chemin `command` absolu).

## Valeurs par défaut propriété du plugin

Les valeurs par défaut des backends CLI font désormais partie de la surface du plugin :

- Les plugins les enregistrent avec `api.registerCliBackend(...)`.
- L’`id` du backend devient le préfixe du fournisseur dans les références de modèle.
- La configuration utilisateur dans `agents.defaults.cliBackends.<id>` surcharge toujours la valeur par défaut du plugin.
- Le nettoyage de configuration spécifique au backend reste propriété du plugin via le hook optionnel
  `normalizeConfig`.

Les plugins qui ont besoin de petits shims de compatibilité de prompt/message peuvent déclarer
des transformations de texte bidirectionnelles sans remplacer un fournisseur ou un backend CLI :

```typescript
api.registerTextTransforms({
  input: [
    { from: /red basket/g, to: "blue basket" },
    { from: /paper ticket/g, to: "digital ticket" },
    { from: /left shelf/g, to: "right shelf" },
  ],
  output: [
    { from: /blue basket/g, to: "red basket" },
    { from: /digital ticket/g, to: "paper ticket" },
    { from: /right shelf/g, to: "left shelf" },
  ],
});
```

`input` réécrit le prompt système et le prompt utilisateur transmis au CLI. `output`
réécrit les deltas diffusés de l’assistant et le texte final analysé avant qu’OpenClaw ne gère
ses propres marqueurs de contrôle et la livraison au canal.

Pour les CLI qui émettent du JSONL compatible avec le stream-json de Claude Code, définissez
`jsonlDialect: "claude-stream-json"` dans la configuration de ce backend.

## Overlays MCP intégrés

Les backends CLI **ne** reçoivent **pas** directement les appels d’outils OpenClaw, mais un backend peut
opter pour un overlay de configuration MCP généré avec `bundleMcp: true`.

Comportement intégré actuel :

- `claude-cli` : fichier de configuration MCP strict généré
- `codex-cli` : surcharges de configuration en ligne pour `mcp_servers`
- `google-gemini-cli` : fichier de paramètres système Gemini généré

Lorsque bundle MCP est activé, OpenClaw :

- lance un serveur MCP HTTP en loopback qui expose les outils de la gateway au processus CLI
- authentifie le pont avec un jeton par session (`OPENCLAW_MCP_TOKEN`)
- limite l’accès aux outils à la session, au compte, et au contexte de canal actuels
- charge les serveurs bundle-MCP activés pour l’espace de travail actuel
- les fusionne avec toute forme existante de configuration/paramètres MCP du backend
- réécrit la configuration de lancement à l’aide du mode d’intégration propriété du backend depuis l’extension propriétaire

Si aucun serveur MCP n’est activé, OpenClaw injecte quand même une configuration stricte lorsqu’un
backend opte pour bundle MCP afin que les exécutions en arrière-plan restent isolées.

## Limitations

- **Pas d’appels directs aux outils OpenClaw.** OpenClaw n’injecte pas d’appels d’outils dans
  le protocole du backend CLI. Les backends ne voient les outils de la gateway que lorsqu’ils optent pour
  `bundleMcp: true`.
- **Le streaming est spécifique au backend.** Certains backends diffusent en JSONL ; d’autres mettent en tampon
  jusqu’à la fin.
- **Les sorties structurées** dépendent du format JSON du CLI.
- **Les sessions Codex CLI** reprennent via une sortie texte (pas de JSONL), ce qui est moins
  structuré que l’exécution initiale avec `--json`. Les sessions OpenClaw continuent de fonctionner
  normalement.

## Dépannage

- **CLI introuvable** : définissez `command` sur un chemin complet.
- **Nom de modèle incorrect** : utilisez `modelAliases` pour mapper `provider/model` → modèle CLI.
- **Aucune continuité de session** : assurez-vous que `sessionArg` est défini et que `sessionMode` n’est pas
  `none` (Codex CLI ne peut actuellement pas reprendre avec une sortie JSON).
- **Images ignorées** : définissez `imageArg` (et vérifiez que le CLI prend en charge les chemins de fichiers).
