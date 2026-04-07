---
read_when:
    - Exécution de harnesses de code via ACP
    - Configuration de sessions ACP liées à une conversation sur des canaux de messagerie
    - Association d'une conversation de canal de messages à une session ACP persistante
    - Dépannage du backend ACP et du câblage du plugin
    - Utilisation des commandes `/acp` depuis le chat
summary: Utiliser des sessions d'exécution ACP pour Codex, Claude Code, Cursor, Gemini CLI, OpenClaw ACP et d'autres agents de harness
title: Agents ACP
x-i18n:
    generated_at: "2026-04-07T06:56:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: fb651ab39b05e537398623ee06cb952a5a07730fc75d3f7e0de20dd3128e72c6
    source_path: tools/acp-agents.md
    workflow: 15
---

# Agents ACP

Les sessions [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) permettent à OpenClaw d'exécuter des harnesses de code externes (par exemple Pi, Claude Code, Codex, Cursor, Copilot, OpenClaw ACP, OpenCode, Gemini CLI et d'autres harnesses ACPX pris en charge) via un plugin backend ACP.

Si vous demandez à OpenClaw en langage naturel de « lancer ceci dans Codex » ou de « démarrer Claude Code dans un fil », OpenClaw doit router cette demande vers l'environnement d'exécution ACP (et non vers l'environnement d'exécution natif des subagents). Chaque lancement de session ACP est suivi comme une [tâche d'arrière-plan](/fr/automation/tasks).

Si vous voulez que Codex ou Claude Code se connecte directement comme client MCP externe
à des conversations de canal OpenClaw existantes, utilisez
[`openclaw mcp serve`](/cli/mcp) au lieu d'ACP.

## Quelle page me faut-il ?

Il existe trois surfaces proches qu'il est facile de confondre :

| Vous voulez...                                                                     | Utilisez                              | Remarques                                                                                                       |
| ---------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Exécuter Codex, Claude Code, Gemini CLI ou un autre harness externe _via_ OpenClaw | Cette page : agents ACP               | Sessions liées au chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, tâches d'arrière-plan, contrôles d'exécution |
| Exposer une session Gateway OpenClaw _comme_ serveur ACP pour un éditeur ou un client      | [`openclaw acp`](/cli/acp)            | Mode pont. L'IDE/le client parle ACP à OpenClaw via stdio/WebSocket                                          |
| Réutiliser une CLI IA locale comme modèle de repli texte uniquement                                 | [CLI Backends](/fr/gateway/cli-backends) | Pas ACP. Pas d'outils OpenClaw, pas de contrôles ACP, pas d'environnement d'exécution de harness             |

## Cela fonctionne-t-il immédiatement ?

Généralement, oui.

- Les installations fraîches livrent maintenant le plugin d'exécution `acpx` intégré activé par défaut.
- Le plugin `acpx` intégré préfère son binaire `acpx` local au plugin et épinglé.
- Au démarrage, OpenClaw sonde ce binaire et le répare automatiquement si nécessaire.
- Commencez par `/acp doctor` si vous voulez une vérification rapide de l'état de préparation.

Ce qui peut encore arriver à la première utilisation :

- Un adaptateur de harness cible peut être récupéré à la demande avec `npx` la première fois que vous utilisez ce harness.
- L'authentification du fournisseur doit toujours exister sur l'hôte pour ce harness.
- Si l'hôte n'a pas d'accès npm/réseau, les récupérations d'adaptateur au premier lancement peuvent échouer jusqu'à ce que les caches soient préchauffés ou que l'adaptateur soit installé autrement.

Exemples :

- `/acp spawn codex` : OpenClaw doit être prêt à amorcer `acpx`, mais l'adaptateur ACP Codex peut encore nécessiter une récupération au premier lancement.
- `/acp spawn claude` : même situation pour l'adaptateur ACP Claude, plus l'authentification côté Claude sur cet hôte.

## Flux opérateur rapide

Utilisez ceci lorsque vous voulez un runbook `/acp` pratique :

1. Lancez une session :
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. Travaillez dans la conversation ou le fil lié (ou ciblez explicitement cette clé de session).
3. Vérifiez l'état d'exécution :
   - `/acp status`
4. Ajustez les options d'exécution selon les besoins :
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. Redirigez une session active sans remplacer le contexte :
   - `/acp steer resserre la journalisation et continue`
6. Arrêtez le travail :
   - `/acp cancel` (arrêter le tour en cours), ou
   - `/acp close` (fermer la session + supprimer les associations)

## Démarrage rapide pour les humains

Exemples de demandes naturelles :

- « Associe ce canal Discord à Codex. »
- « Démarre une session Codex persistante dans un fil ici et garde-la ciblée. »
- « Exécute ceci comme une session ACP Claude Code ponctuelle et résume le résultat. »
- « Associe cette conversation iMessage à Codex et garde les suivis dans le même espace de travail. »
- « Utilise Gemini CLI pour cette tâche dans un fil, puis garde les suivis dans ce même fil. »

Ce qu'OpenClaw doit faire :

1. Choisir `runtime: "acp"`.
2. Résoudre la cible de harness demandée (`agentId`, par exemple `codex`).
3. Si une association à la conversation courante est demandée et que le canal actif la prend en charge, associer la session ACP à cette conversation.
4. Sinon, si une association à un fil est demandée et que le canal actuel la prend en charge, associer la session ACP au fil.
5. Router les messages de suivi liés vers cette même session ACP jusqu'à la perte de focus/la fermeture/l'expiration.

## ACP versus subagents

Utilisez ACP lorsque vous voulez un environnement d'exécution de harness externe. Utilisez les subagents lorsque vous voulez des exécutions déléguées natives OpenClaw.

| Domaine       | Session ACP                           | Exécution de subagent                |
| ------------- | ------------------------------------- | ------------------------------------ |
| Exécution       | Plugin backend ACP (par exemple `acpx`) | Environnement d'exécution natif des subagents OpenClaw  |
| Clé de session   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Commandes principales | `/acp ...`                            | `/subagents ...`                   |
| Outil de lancement    | `sessions_spawn` avec `runtime:"acp"` | `sessions_spawn` (runtime par défaut) |

Voir aussi [Sub-agents](/fr/tools/subagents).

## Comment ACP exécute Claude Code

Pour Claude Code via ACP, la pile est la suivante :

1. Plan de contrôle des sessions ACP OpenClaw
2. plugin d'exécution `acpx` intégré
3. Adaptateur ACP Claude
4. Mécanisme d'exécution/session côté Claude

Distinction importante :

- ACP Claude est une session de harness avec contrôles ACP, reprise de session, suivi des tâches d'arrière-plan et association facultative à une conversation/un fil.
- Les backends CLI sont des environnements de repli locaux séparés en mode texte uniquement. Voir [CLI Backends](/fr/gateway/cli-backends).

Pour les opérateurs, la règle pratique est :

- vous voulez `/acp spawn`, des sessions associables, des contrôles d'exécution ou un travail de harness persistant : utilisez ACP
- vous voulez un simple repli texte local via la CLI brute : utilisez les backends CLI

## Sessions associées

### Associations à la conversation courante

Utilisez `/acp spawn <harness> --bind here` lorsque vous voulez que la conversation actuelle devienne un espace de travail ACP durable sans créer de fil enfant.

Comportement :

- OpenClaw continue de gérer le transport du canal, l'authentification, la sécurité et la livraison.
- La conversation actuelle est épinglée à la clé de session ACP lancée.
- Les messages de suivi dans cette conversation sont routés vers la même session ACP.
- `/new` et `/reset` réinitialisent la même session ACP associée sur place.
- `/acp close` ferme la session et supprime l'association à la conversation courante.

Ce que cela signifie en pratique :

- `--bind here` conserve la même surface de chat. Sur Discord, le canal actuel reste le canal actuel.
- `--bind here` peut toujours créer une nouvelle session ACP si vous lancez un nouveau travail. L'association attache cette session à la conversation courante.
- `--bind here` ne crée pas à lui seul un fil enfant Discord ou un sujet Telegram.
- L'environnement d'exécution ACP peut toujours avoir son propre répertoire de travail (`cwd`) ou un espace de travail géré par backend sur disque. Cet espace de travail d'exécution est distinct de la surface de chat et n'implique pas un nouveau fil de messagerie.
- Si vous lancez vers un autre agent ACP et ne passez pas `--cwd`, OpenClaw hérite par défaut de l'espace de travail de **l'agent cible**, pas de celui du demandeur.
- Si ce chemin d'espace de travail hérité est manquant (`ENOENT`/`ENOTDIR`), OpenClaw se replie sur le `cwd` par défaut du backend au lieu de réutiliser silencieusement le mauvais arbre.
- Si l'espace de travail hérité existe mais n'est pas accessible (par exemple `EACCES`), le lancement renvoie la véritable erreur d'accès au lieu d'ignorer `cwd`.

Modèle mental :

- surface de chat : endroit où les personnes continuent à parler (`canal Discord`, `sujet Telegram`, `discussion iMessage`)
- session ACP : état d'exécution durable Codex/Claude/Gemini vers lequel OpenClaw route
- fil/sujet enfant : surface de messagerie supplémentaire facultative créée uniquement par `--thread ...`
- espace de travail d'exécution : emplacement du système de fichiers où le harness s'exécute (`cwd`, extraction du dépôt, espace de travail backend)

Exemples :

- `/acp spawn codex --bind here` : conserver ce chat, lancer ou attacher une session ACP Codex, puis router les futurs messages ici vers elle
- `/acp spawn codex --thread auto` : OpenClaw peut créer un fil/sujet enfant et y associer la session ACP
- `/acp spawn codex --bind here --cwd /workspace/repo` : même association de chat que ci-dessus, mais Codex s'exécute dans `/workspace/repo`

Prise en charge de l'association à la conversation courante :

- Les canaux de chat/messages qui annoncent la prise en charge de l'association à la conversation courante peuvent utiliser `--bind here` via le chemin partagé d'association de conversation.
- Les canaux avec une sémantique personnalisée de fil/sujet peuvent toujours fournir une canonicalisation spécifique au canal derrière la même interface partagée.
- `--bind here` signifie toujours « associer la conversation courante sur place ».
- Les associations génériques à la conversation courante utilisent le stockage d'associations partagé d'OpenClaw et survivent aux redémarrages normaux de gateway.

Remarques :

- `--bind here` et `--thread ...` sont mutuellement exclusifs sur `/acp spawn`.
- Sur Discord, `--bind here` associe sur place le canal ou le fil actuel. `spawnAcpSessions` n'est requis que lorsque OpenClaw doit créer un fil enfant pour `--thread auto|here`.
- Si le canal actif n'expose pas d'associations ACP à la conversation courante, OpenClaw renvoie un message clair indiquant que ce n'est pas pris en charge.
- `resume` et les questions de « nouvelle session » sont des questions de session ACP, pas des questions de canal. Vous pouvez réutiliser ou remplacer l'état d'exécution sans changer la surface de chat actuelle.

### Sessions associées à un fil

Lorsque les associations de fil sont activées pour un adaptateur de canal, les sessions ACP peuvent être associées à des fils :

- OpenClaw associe un fil à une session ACP cible.
- Les messages de suivi dans ce fil sont routés vers la session ACP associée.
- La sortie ACP est renvoyée dans ce même fil.
- La perte de focus/la fermeture/l'archivage/l'expiration par délai d'inactivité ou âge maximal supprime l'association.

La prise en charge de l'association à un fil dépend de l'adaptateur. Si l'adaptateur de canal actif ne la prend pas en charge, OpenClaw renvoie un message clair indiquant qu'elle n'est pas prise en charge/indisponible.

Indicateurs de fonctionnalité requis pour l'ACP associé à un fil :

- `acp.enabled=true`
- `acp.dispatch.enabled` est activé par défaut (définissez `false` pour mettre en pause le routage ACP)
- Indicateur de lancement de fil ACP de l'adaptateur de canal activé (spécifique à l'adaptateur)
  - Discord : `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram : `channels.telegram.threadBindings.spawnAcpSessions=true`

### Canaux prenant en charge les fils

- Tout adaptateur de canal qui expose une capacité d'association de session/fil.
- Prise en charge intégrée actuelle :
  - Fils/canaux Discord
  - Sujets Telegram (sujets de forum dans les groupes/supergroupes et sujets de DM)
- Les canaux plugin peuvent ajouter cette prise en charge via la même interface d'association.

## Paramètres spécifiques aux canaux

Pour les workflows non éphémères, configurez des associations ACP persistantes dans les entrées `bindings[]` de premier niveau.

### Modèle d'association

- `bindings[].type="acp"` marque une association persistante de conversation ACP.
- `bindings[].match` identifie la conversation cible :
  - Canal ou fil Discord : `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - Sujet de forum Telegram : `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - Discussion DM/de groupe BlueBubbles : `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Préférez `chat_id:*` ou `chat_identifier:*` pour des associations de groupe stables.
  - Discussion DM/de groupe iMessage : `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Préférez `chat_id:*` pour des associations de groupe stables.
- `bindings[].agentId` est l'ID de l'agent OpenClaw propriétaire.
- Les remplacements ACP facultatifs se trouvent sous `bindings[].acp` :
  - `mode` (`persistent` ou `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### Valeurs par défaut d'exécution par agent

Utilisez `agents.list[].runtime` pour définir une fois les valeurs par défaut ACP par agent :

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (ID de harness, par exemple `codex` ou `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

Priorité des remplacements pour les sessions ACP associées :

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. valeurs par défaut ACP globales (par exemple `acp.backend`)

Exemple :

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

Comportement :

- OpenClaw s'assure que la session ACP configurée existe avant utilisation.
- Les messages dans ce canal ou ce sujet sont routés vers la session ACP configurée.
- Dans les conversations associées, `/new` et `/reset` réinitialisent sur place la même clé de session ACP.
- Les associations d'exécution temporaires (par exemple créées par des flux de focus de fil) s'appliquent toujours lorsqu'elles sont présentes.
- Pour les lancements ACP inter-agents sans `cwd` explicite, OpenClaw hérite de l'espace de travail de l'agent cible depuis la configuration des agents.
- Les chemins d'espace de travail hérités manquants se replient sur le `cwd` par défaut du backend ; les véritables échecs d'accès remontent comme erreurs de lancement.

## Démarrer des sessions ACP (interfaces)

### Depuis `sessions_spawn`

Utilisez `runtime: "acp"` pour démarrer une session ACP depuis un tour d'agent ou un appel d'outil.

```json
{
  "task": "Open the repo and summarize failing tests",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

Remarques :

- `runtime` vaut par défaut `subagent`, donc définissez explicitement `runtime: "acp"` pour les sessions ACP.
- Si `agentId` est omis, OpenClaw utilise `acp.defaultAgent` lorsqu'il est configuré.
- `mode: "session"` nécessite `thread: true` pour conserver une conversation persistante associée.

Détails de l'interface :

- `task` (obligatoire) : prompt initial envoyé à la session ACP.
- `runtime` (obligatoire pour ACP) : doit être `"acp"`.
- `agentId` (facultatif) : ID de harness ACP cible. Se replie sur `acp.defaultAgent` s'il est défini.
- `thread` (facultatif, `false` par défaut) : demander un flux d'association à un fil lorsque pris en charge.
- `mode` (facultatif) : `run` (ponctuel) ou `session` (persistant).
  - la valeur par défaut est `run`
  - si `thread: true` et que `mode` est omis, OpenClaw peut adopter par défaut un comportement persistant selon le chemin d'exécution
  - `mode: "session"` nécessite `thread: true`
- `cwd` (facultatif) : répertoire de travail demandé pour l'exécution (validé par la politique du backend/de l'exécution). S'il est omis, le lancement ACP hérite de l'espace de travail de l'agent cible lorsqu'il est configuré ; les chemins hérités manquants se replient sur les valeurs par défaut du backend, tandis que les véritables erreurs d'accès sont renvoyées.
- `label` (facultatif) : libellé destiné à l'opérateur utilisé dans le texte de session/de bannière.
- `resumeSessionId` (facultatif) : reprendre une session ACP existante au lieu d'en créer une nouvelle. L'agent relit son historique de conversation via `session/load`. Nécessite `runtime: "acp"`.
- `streamTo` (facultatif) : `"parent"` diffuse les résumés de progression du lancement ACP initial vers la session demandeuse sous forme d'événements système.
  - Lorsqu'elles sont disponibles, les réponses acceptées incluent `streamLogPath` pointant vers un journal JSONL propre à la session (`<sessionId>.acp-stream.jsonl`) que vous pouvez suivre pour obtenir l'historique complet du relais.

### Reprendre une session existante

Utilisez `resumeSessionId` pour continuer une session ACP précédente au lieu de repartir de zéro. L'agent relit son historique de conversation via `session/load`, ce qui lui permet de reprendre avec tout le contexte de ce qui a précédé.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

Cas d'usage courants :

- Transférer une session Codex de votre ordinateur portable à votre téléphone — dites à votre agent de reprendre là où vous vous êtes arrêté
- Continuer une session de code commencée de manière interactive dans la CLI, maintenant sans interface via votre agent
- Reprendre un travail interrompu par un redémarrage de gateway ou un délai d'inactivité

Remarques :

- `resumeSessionId` nécessite `runtime: "acp"` — renvoie une erreur si utilisé avec l'environnement d'exécution des subagents.
- `resumeSessionId` restaure l'historique de conversation ACP amont ; `thread` et `mode` s'appliquent toujours normalement à la nouvelle session OpenClaw que vous créez, donc `mode: "session"` nécessite toujours `thread: true`.
- L'agent cible doit prendre en charge `session/load` (Codex et Claude Code le font).
- Si l'ID de session est introuvable, le lancement échoue avec une erreur claire — pas de repli silencieux vers une nouvelle session.

### Smoke test opérateur

Utilisez ceci après un déploiement gateway lorsque vous voulez une vérification rapide en conditions réelles que le lancement ACP fonctionne bien de bout en bout, et pas seulement aux tests unitaires.

Barre de validation recommandée :

1. Vérifiez la version/le commit de la gateway déployée sur l'hôte cible.
2. Confirmez que la source déployée inclut l'acceptation de filiation ACP dans
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. Ouvrez une session pont ACPX temporaire vers un agent réel (par exemple
   `razor(main)` sur `jpclawhq`).
4. Demandez à cet agent d'appeler `sessions_spawn` avec :
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - tâche : `Reply with exactly LIVE-ACP-SPAWN-OK`
5. Vérifiez que l'agent signale :
   - `accepted=yes`
   - une vraie `childSessionKey`
   - aucune erreur de validation
6. Nettoyez la session pont ACPX temporaire.

Exemple de prompt à l'agent réel :

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

Remarques :

- Gardez ce smoke test sur `mode: "run"` à moins de tester intentionnellement
  des sessions ACP persistantes associées à un fil.
- N'exigez pas `streamTo: "parent"` pour la validation de base. Ce chemin dépend des capacités
  de la session/du demandeur et constitue une vérification d'intégration distincte.
- Traitez les tests de `mode: "session"` lié à un fil comme une seconde passe d'intégration, plus riche,
  depuis un vrai fil Discord ou un vrai sujet Telegram.

## Compatibilité sandbox

Les sessions ACP s'exécutent actuellement sur l'environnement d'exécution de l'hôte, pas dans le sandbox OpenClaw.

Limites actuelles :

- Si la session demandeuse est sandboxée, les lancements ACP sont bloqués à la fois pour `sessions_spawn({ runtime: "acp" })` et `/acp spawn`.
  - Erreur : `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- `sessions_spawn` avec `runtime: "acp"` ne prend pas en charge `sandbox: "require"`.
  - Erreur : `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

Utilisez `runtime: "subagent"` lorsque vous avez besoin d'une exécution imposée par le sandbox.

### Depuis la commande `/acp`

Utilisez `/acp spawn` pour un contrôle opérateur explicite depuis le chat lorsque nécessaire.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

Indicateurs clés :

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

Voir [Slash Commands](/fr/tools/slash-commands).

## Résolution de la cible de session

La plupart des actions `/acp` acceptent une cible de session facultative (`session-key`, `session-id` ou `session-label`).

Ordre de résolution :

1. Argument de cible explicite (ou `--session` pour `/acp steer`)
   - essaie d'abord la clé
   - puis l'ID de session au format UUID
   - puis le libellé
2. Association du fil courant (si cette conversation/ce fil est associé à une session ACP)
3. Repli vers la session demandeuse actuelle

Les associations à la conversation courante et à un fil participent toutes deux à l'étape 2.

Si aucune cible n'est résolue, OpenClaw renvoie une erreur claire (`Unable to resolve session target: ...`).

## Modes d'association au lancement

`/acp spawn` prend en charge `--bind here|off`.

| Mode   | Comportement                                                               |
| ------ | -------------------------------------------------------------------------- |
| `here` | Associer sur place la conversation active actuelle ; échec s'il n'y en a pas. |
| `off`  | Ne pas créer d'association à la conversation courante.                     |

Remarques :

- `--bind here` est le chemin opérateur le plus simple pour « faire de ce canal ou chat un espace adossé à Codex ».
- `--bind here` ne crée pas de fil enfant.
- `--bind here` n'est disponible que sur les canaux qui exposent la prise en charge de l'association à la conversation courante.
- `--bind` et `--thread` ne peuvent pas être combinés dans le même appel `/acp spawn`.

## Modes de fil au lancement

`/acp spawn` prend en charge `--thread auto|here|off`.

| Mode   | Comportement                                                                                            |
| ------ | ------------------------------------------------------------------------------------------------------- |
| `auto` | Dans un fil actif : associer ce fil. Hors d'un fil : créer/associer un fil enfant lorsque pris en charge. |
| `here` | Exiger un fil actif en cours ; échec si ce n'est pas le cas.                                            |
| `off`  | Pas d'association. La session démarre sans association.                                                 |

Remarques :

- Sur les surfaces sans prise en charge d'association à un fil, le comportement par défaut est effectivement `off`.
- Le lancement avec association à un fil nécessite la prise en charge par la politique du canal :
  - Discord : `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram : `channels.telegram.threadBindings.spawnAcpSessions=true`
- Utilisez `--bind here` lorsque vous voulez épingler la conversation actuelle sans créer de fil enfant.

## Contrôles ACP

Famille de commandes disponibles :

- `/acp spawn`
- `/acp cancel`
- `/acp steer`
- `/acp close`
- `/acp status`
- `/acp set-mode`
- `/acp set`
- `/acp cwd`
- `/acp permissions`
- `/acp timeout`
- `/acp model`
- `/acp reset-options`
- `/acp sessions`
- `/acp doctor`
- `/acp install`

`/acp status` affiche les options d'exécution effectives et, lorsqu'ils sont disponibles, les identifiants de session au niveau de l'exécution et du backend.

Certains contrôles dépendent des capacités du backend. Si un backend ne prend pas en charge un contrôle, OpenClaw renvoie une erreur claire de contrôle non pris en charge.

## Recettes de commandes ACP

| Commande              | Ce qu'elle fait                                              | Exemple                                                       |
| -------------------- | ------------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Créer une session ACP ; association facultative à la conversation courante ou à un fil. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Annuler le tour en cours pour la session cible.               | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Envoyer une instruction de pilotage à la session en cours.    | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Fermer la session et dissocier les cibles de fil.             | `/acp close`                                                  |
| `/acp status`        | Afficher backend, mode, état, options d'exécution, capacités. | `/acp status`                                                 |
| `/acp set-mode`      | Définir le mode d'exécution pour la session cible.            | `/acp set-mode plan`                                          |
| `/acp set`           | Écriture générique d'option de configuration d'exécution.     | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Définir le remplacement du répertoire de travail d'exécution. | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Définir le profil de politique d'approbation.                 | `/acp permissions strict`                                     |
| `/acp timeout`       | Définir le délai d'exécution (secondes).                      | `/acp timeout 120`                                            |
| `/acp model`         | Définir le remplacement du modèle d'exécution.                | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Supprimer les remplacements d'options d'exécution de la session. | `/acp reset-options`                                          |
| `/acp sessions`      | Lister les sessions ACP récentes depuis le stockage.          | `/acp sessions`                                               |
| `/acp doctor`        | Santé du backend, capacités, correctifs exploitables.         | `/acp doctor`                                                 |
| `/acp install`       | Afficher les étapes d'installation et d'activation déterministes. | `/acp install`                                                |

`/acp sessions` lit le stockage pour la session actuellement associée ou la session demandeuse. Les commandes qui acceptent des jetons `session-key`, `session-id` ou `session-label` résolvent les cibles via la découverte de sessions gateway, y compris les racines personnalisées `session.store` par agent.

## Correspondance des options d'exécution

`/acp` a des commandes de commodité et un setter générique.

Opérations équivalentes :

- `/acp model <id>` correspond à la clé de configuration d'exécution `model`.
- `/acp permissions <profile>` correspond à la clé de configuration d'exécution `approval_policy`.
- `/acp timeout <seconds>` correspond à la clé de configuration d'exécution `timeout`.
- `/acp cwd <path>` met directement à jour le remplacement `cwd` d'exécution.
- `/acp set <key> <value>` est le chemin générique.
  - Cas particulier : `key=cwd` utilise le chemin de remplacement `cwd`.
- `/acp reset-options` efface tous les remplacements d'exécution pour la session cible.

## Prise en charge des harnesses acpx (actuelle)

Alias intégrés actuels des harnesses acpx :

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI : `cursor-agent acp`)
- `droid`
- `gemini`
- `iflow`
- `kilocode`
- `kimi`
- `kiro`
- `openclaw`
- `opencode`
- `pi`
- `qwen`

Lorsque OpenClaw utilise le backend acpx, préférez ces valeurs pour `agentId`, sauf si votre configuration acpx définit des alias d'agent personnalisés.
Si votre installation locale de Cursor expose encore ACP sous `agent acp`, remplacez la commande de l'agent `cursor` dans votre configuration acpx au lieu de modifier la valeur intégrée par défaut.

L'utilisation directe de la CLI acpx peut aussi cibler des adaptateurs arbitraires via `--agent <command>`, mais cette échappatoire brute est une fonctionnalité de la CLI acpx (pas du chemin normal `agentId` d'OpenClaw).

## Configuration requise

Référence ACP du cœur :

```json5
{
  acp: {
    enabled: true,
    // Facultatif. La valeur par défaut est true ; définissez false pour mettre en pause le routage ACP tout en conservant les contrôles /acp.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "pi",
      "qwen",
    ],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

La configuration d'association à un fil est spécifique à l'adaptateur de canal. Exemple pour Discord :

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

Si le lancement ACP lié à un fil ne fonctionne pas, vérifiez d'abord l'indicateur de fonctionnalité de l'adaptateur :

- Discord : `channels.discord.threadBindings.spawnAcpSessions=true`

Les associations à la conversation courante ne nécessitent pas de création de fil enfant. Elles nécessitent un contexte de conversation actif et un adaptateur de canal qui expose les associations de conversation ACP.

Voir [Référence de configuration](/fr/gateway/configuration-reference).

## Configuration du plugin pour le backend acpx

Les installations fraîches livrent le plugin d'exécution `acpx` intégré activé par défaut, donc ACP
fonctionne généralement sans étape d'installation manuelle du plugin.

Commencez par :

```text
/acp doctor
```

Si vous avez désactivé `acpx`, l'avez refusé via `plugins.allow` / `plugins.deny`, ou si vous voulez
basculer vers une extraction locale de développement, utilisez le chemin explicite du plugin :

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

Installation locale de l'espace de travail pendant le développement :

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Vérifiez ensuite la santé du backend :

```text
/acp doctor
```

### Configuration de la commande et de la version acpx

Par défaut, le plugin backend acpx intégré (`acpx`) utilise le binaire local au plugin et épinglé :

1. La commande vaut par défaut le `node_modules/.bin/acpx` local au plugin à l'intérieur du package du plugin ACPX.
2. La version attendue vaut par défaut le pin de l'extension.
3. Au démarrage, OpenClaw enregistre immédiatement le backend ACP comme non prêt.
4. Un travail de vérification en arrière-plan exécute `acpx --version`.
5. Si le binaire local au plugin est manquant ou ne correspond pas, il exécute :
   `npm install --omit=dev --no-save acpx@<pinned>` puis reverifie.

Vous pouvez remplacer la commande/la version dans la configuration du plugin :

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "command": "../acpx/dist/cli.js",
          "expectedVersion": "any"
        }
      }
    }
  }
}
```

Remarques :

- `command` accepte un chemin absolu, un chemin relatif ou un nom de commande (`acpx`).
- Les chemins relatifs sont résolus depuis le répertoire d'espace de travail OpenClaw.
- `expectedVersion: "any"` désactive la correspondance stricte de version.
- Lorsque `command` pointe vers un binaire/chemin personnalisé, l'auto-installation locale au plugin est désactivée.
- Le démarrage d'OpenClaw reste non bloquant pendant l'exécution de la vérification de santé du backend.

Voir [Plugins](/fr/tools/plugin).

### Installation automatique des dépendances

Lorsque vous installez OpenClaw globalement avec `npm install -g openclaw`, les dépendances d'exécution acpx
(binaires spécifiques à la plateforme) sont installées automatiquement
via un hook postinstall. Si l'installation automatique échoue, la gateway démarre quand même normalement et signale la dépendance manquante via `openclaw acp doctor`.

### Pont MCP des outils de plugin

Par défaut, les sessions ACPX **n'exposent pas** les outils enregistrés par plugin OpenClaw au
harness ACP.

Si vous voulez que des agents ACP tels que Codex ou Claude Code puissent appeler des
outils OpenClaw installés, comme le rappel/le stockage mémoire, activez le pont dédié :

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Ce que cela fait :

- Injecte un serveur MCP intégré nommé `openclaw-plugin-tools` dans l'amorçage de session ACPX.
- Expose les outils de plugin déjà enregistrés par les plugins OpenClaw installés et activés.
- Garde la fonctionnalité explicite et désactivée par défaut.

Remarques de sécurité et de confiance :

- Cela étend la surface d'outils du harness ACP.
- Les agents ACP n'obtiennent l'accès qu'aux outils de plugin déjà actifs dans la gateway.
- Traitez cela comme la même frontière de confiance que lorsque vous laissez ces plugins s'exécuter dans OpenClaw lui-même.
- Vérifiez les plugins installés avant d'activer cela.

Les `mcpServers` personnalisés continuent de fonctionner comme avant. Le pont intégré plugin-tools est une commodité supplémentaire activable, pas un remplacement de la configuration générique des serveurs MCP.

## Configuration des permissions

Les sessions ACP s'exécutent de manière non interactive — il n'y a pas de TTY pour approuver ou refuser les invites de permission d'écriture de fichier et d'exécution shell. Le plugin acpx fournit deux clés de configuration qui contrôlent la manière dont les permissions sont gérées :

Ces permissions de harness ACPX sont séparées des approbations d'exécution OpenClaw et séparées des indicateurs de contournement du fournisseur des backends CLI, comme Claude CLI `--permission-mode bypassPermissions`. ACPX `approve-all` est l'interrupteur de secours au niveau du harness pour les sessions ACP.

### `permissionMode`

Contrôle quelles opérations l'agent du harness peut effectuer sans invite.

| Valeur           | Comportement                                                  |
| --------------- | ------------------------------------------------------------- |
| `approve-all`   | Approuve automatiquement toutes les écritures de fichier et commandes shell.          |
| `approve-reads` | Approuve automatiquement les lectures uniquement ; les écritures et exécutions nécessitent des invites. |
| `deny-all`      | Refuse toutes les invites de permission.                      |

### `nonInteractivePermissions`

Contrôle ce qui se passe lorsqu'une invite de permission devrait être affichée mais qu'aucun TTY interactif n'est disponible (ce qui est toujours le cas pour les sessions ACP).

| Valeur  | Comportement                                                          |
| ------ | --------------------------------------------------------------------- |
| `fail` | Abandonne la session avec `AcpRuntimeError`. **(par défaut)**           |
| `deny` | Refuse silencieusement la permission et continue (dégradation progressive). |

### Configuration

Définir via la configuration du plugin :

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Redémarrez la gateway après modification de ces valeurs.

> **Important :** OpenClaw utilise actuellement par défaut `permissionMode=approve-reads` et `nonInteractivePermissions=fail`. Dans les sessions ACP non interactives, toute écriture ou exécution qui déclenche une invite de permission peut échouer avec `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> Si vous devez restreindre les permissions, définissez `nonInteractivePermissions` sur `deny` afin que les sessions se dégradent proprement au lieu de planter.

## Dépannage

| Symptôme                                                                     | Cause probable                                                                    | Correctif                                                                                                                                                               |
| --------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured`                                     | Plugin backend manquant ou désactivé.                                             | Installez et activez le plugin backend, puis exécutez `/acp doctor`.                                                                                                  |
| `ACP is disabled by policy (acp.enabled=false)`                             | ACP désactivé globalement.                                                          | Définissez `acp.enabled=true`.                                                                                                                                         |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | Le routage depuis les messages normaux du fil est désactivé.                      | Définissez `acp.dispatch.enabled=true`.                                                                                                                                |
| `ACP agent "<id>" is not allowed by policy`                                 | Agent absent de l'allowlist.                                                         | Utilisez un `agentId` autorisé ou mettez à jour `acp.allowedAgents`.                                                                                                  |
| `Unable to resolve session target: ...`                                     | Mauvais jeton clé/id/libellé.                                                         | Exécutez `/acp sessions`, copiez la clé/le libellé exact, puis réessayez.                                                                                             |
| `--bind here requires running /acp spawn inside an active ... conversation` | `--bind here` utilisé sans conversation active pouvant être associée.                     | Déplacez-vous vers le chat/canal cible et réessayez, ou utilisez un lancement sans association.                                                                       |
| `Conversation bindings are unavailable for <channel>.`                      | L'adaptateur ne dispose pas de la capacité d'association ACP à la conversation courante.                      | Utilisez `/acp spawn ... --thread ...` lorsque c'est pris en charge, configurez `bindings[]` de premier niveau, ou passez à un canal pris en charge.                 |
| `--thread here requires running /acp spawn inside an active ... thread`     | `--thread here` utilisé hors d'un contexte de fil.                                  | Déplacez-vous vers le fil cible ou utilisez `--thread auto`/`off`.                                                                                                    |
| `Only <user-id> can rebind this channel/conversation/thread.`               | Un autre utilisateur possède la cible d'association active.                                    | Réassociez en tant que propriétaire ou utilisez une autre conversation ou un autre fil.                                                                               |
| `Thread bindings are unavailable for <channel>.`                            | L'adaptateur ne dispose pas de la capacité d'association à un fil.                                        | Utilisez `--thread off` ou passez à un adaptateur/canal pris en charge.                                                                                               |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | L'environnement d'exécution ACP est côté hôte ; la session demandeuse est sandboxée.                       | Utilisez `runtime="subagent"` depuis des sessions sandboxées, ou lancez ACP depuis une session non sandboxée.                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | `sandbox="require"` demandé pour l'environnement d'exécution ACP.                                  | Utilisez `runtime="subagent"` pour un sandbox obligatoire, ou ACP avec `sandbox="inherit"` depuis une session non sandboxée.                                         |
| Métadonnées ACP manquantes pour la session associée                                      | Métadonnées de session ACP obsolètes/supprimées.                                             | Recréez-la avec `/acp spawn`, puis réassociez/refocalisez le fil.                                                                                                     |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | `permissionMode` bloque les écritures/exécutions dans une session ACP non interactive.             | Définissez `plugins.entries.acpx.config.permissionMode` sur `approve-all` et redémarrez la gateway. Voir [Configuration des permissions](#configuration-des-permissions).                 |
| La session ACP échoue très tôt avec peu de sortie                                  | Les invites de permission sont bloquées par `permissionMode`/`nonInteractivePermissions`. | Vérifiez les journaux gateway pour `AcpRuntimeError`. Pour les permissions complètes, définissez `permissionMode=approve-all` ; pour une dégradation progressive, définissez `nonInteractivePermissions=deny`. |
| La session ACP reste bloquée indéfiniment après la fin du travail                       | Le processus du harness est terminé mais la session ACP n'a pas signalé son achèvement.             | Surveillez avec `ps aux \| grep acpx` ; tuez manuellement les processus obsolètes.                                                                                                |
