---
read_when:
    - Exécuter des harnesses de code via ACP
    - Configurer des sessions ACP liées à une conversation sur des canaux de messagerie
    - Lier une conversation de canal de messagerie à une session ACP persistante
    - Résoudre les problèmes de backend ACP et de câblage des plugins
    - Utiliser les commandes /acp depuis le chat
summary: Utiliser des sessions d’exécution ACP pour Codex, Claude Code, Cursor, Gemini CLI, OpenClaw ACP et d’autres agents de harness
title: Agents ACP
x-i18n:
    generated_at: "2026-04-08T02:19:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71c7c0cdae5247aefef17a0029360950a1c2987ddcee21a1bb7d78c67da52950
    source_path: tools/acp-agents.md
    workflow: 15
---

# Agents ACP

Les sessions [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) permettent à OpenClaw d’exécuter des harnesses de code externes (par exemple Pi, Claude Code, Codex, Cursor, Copilot, OpenClaw ACP, OpenCode, Gemini CLI et d’autres harnesses ACPX pris en charge) via un plugin de backend ACP.

Si vous demandez à OpenClaw en langage naturel « exécute ceci dans Codex » ou « démarre Claude Code dans un fil », OpenClaw doit acheminer cette requête vers l’exécution ACP (et non vers l’exécution native de sous-agent). Chaque création de session ACP est suivie comme une [tâche d’arrière-plan](/fr/automation/tasks).

Si vous voulez que Codex ou Claude Code se connecte directement comme client MCP externe
à des conversations de canal OpenClaw existantes, utilisez plutôt
[`openclaw mcp serve`](/cli/mcp) qu’ACP.

## Quelle page me faut-il ?

Il existe trois surfaces proches qu’il est facile de confondre :

| Vous voulez...                                                                     | Utilisez ceci                         | Remarques                                                                                                        |
| ----------------------------------------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Exécuter Codex, Claude Code, Gemini CLI ou un autre harness externe _via_ OpenClaw | Cette page : agents ACP               | Sessions liées au chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, tâches d’arrière-plan, contrôles d’exécution |
| Exposer une session OpenClaw Gateway _comme_ serveur ACP pour un éditeur ou client | [`openclaw acp`](/cli/acp)            | Mode pont. L’IDE/client parle ACP à OpenClaw via stdio/WebSocket                                                |
| Réutiliser une IA CLI locale comme modèle de repli texte uniquement                | [Backends CLI](/fr/gateway/cli-backends) | Pas ACP. Pas d’outils OpenClaw, pas de contrôles ACP, pas d’exécution de harness                                |

## Est-ce que cela fonctionne immédiatement ?

En général, oui.

- Les nouvelles installations livrent désormais le plugin d’exécution `acpx` inclus activé par défaut.
- Le plugin `acpx` inclus privilégie son binaire `acpx` épinglé local au plugin.
- Au démarrage, OpenClaw sonde ce binaire et le répare automatiquement si nécessaire.
- Commencez par `/acp doctor` si vous voulez une vérification rapide de l’état de préparation.

Ce qui peut encore se produire à la première utilisation :

- Un adaptateur de harness cible peut être récupéré à la demande avec `npx` la première fois que vous utilisez ce harness.
- L’authentification du fournisseur doit toujours exister sur l’hôte pour ce harness.
- Si l’hôte n’a pas d’accès npm/réseau, les récupérations initiales d’adaptateur peuvent échouer tant que les caches ne sont pas préchauffés ou que l’adaptateur n’est pas installé autrement.

Exemples :

- `/acp spawn codex` : OpenClaw doit être prêt à initialiser `acpx`, mais l’adaptateur ACP Codex peut encore nécessiter une récupération au premier lancement.
- `/acp spawn claude` : même situation pour l’adaptateur ACP Claude, plus l’authentification côté Claude sur cet hôte.

## Flux opérateur rapide

Utilisez ceci si vous voulez un guide pratique pour `/acp` :

1. Créez une session :
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. Travaillez dans la conversation ou le fil lié (ou ciblez explicitement cette clé de session).
3. Vérifiez l’état d’exécution :
   - `/acp status`
4. Ajustez les options d’exécution si nécessaire :
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. Réorientez une session active sans remplacer le contexte :
   - `/acp steer tighten logging and continue`
6. Arrêtez le travail :
   - `/acp cancel` (arrêter le tour en cours), ou
   - `/acp close` (fermer la session + supprimer les liaisons)

## Démarrage rapide pour les humains

Exemples de demandes en langage naturel :

- « Lie ce canal Discord à Codex. »
- « Démarre une session Codex persistante dans un fil ici et garde-la focalisée. »
- « Exécute ceci comme session ACP Claude Code ponctuelle et résume le résultat. »
- « Lie ce chat iMessage à Codex et garde les suivis dans le même espace de travail. »
- « Utilise Gemini CLI pour cette tâche dans un fil, puis garde les suivis dans ce même fil. »

Ce qu’OpenClaw doit faire :

1. Choisir `runtime: "acp"`.
2. Résoudre la cible de harness demandée (`agentId`, par exemple `codex`).
3. Si une liaison à la conversation courante est demandée et que le canal actif la prend en charge, lier la session ACP à cette conversation.
4. Sinon, si une liaison de fil est demandée et que le canal actuel la prend en charge, lier la session ACP au fil.
5. Acheminer les messages de suivi liés vers cette même session ACP jusqu’à défocalisation/fermeture/expiration.

## ACP versus sous-agents

Utilisez ACP lorsque vous voulez une exécution de harness externe. Utilisez les sous-agents lorsque vous voulez des exécutions déléguées natives OpenClaw.

| Domaine        | Session ACP                           | Exécution de sous-agent            |
| -------------- | ------------------------------------- | ---------------------------------- |
| Exécution      | Plugin de backend ACP (par exemple acpx) | Exécution native de sous-agent OpenClaw |
| Clé de session | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Commandes principales | `/acp ...`                     | `/subagents ...`                   |
| Outil de création | `sessions_spawn` avec `runtime:"acp"` | `sessions_spawn` (exécution par défaut) |

Voir aussi [Sous-agents](/fr/tools/subagents).

## Comment ACP exécute Claude Code

Pour Claude Code via ACP, la pile est :

1. Plan de contrôle de session ACP OpenClaw
2. Plugin d’exécution `acpx` inclus
3. Adaptateur ACP Claude
4. Mécanisme d’exécution/session côté Claude

Distinction importante :

- ACP Claude est une session de harness avec contrôles ACP, reprise de session, suivi des tâches d’arrière-plan et liaison facultative à une conversation/un fil.
- Les backends CLI sont des exécutions locales de repli texte uniquement distinctes. Consultez [Backends CLI](/fr/gateway/cli-backends).

Pour les opérateurs, la règle pratique est :

- vous voulez `/acp spawn`, des sessions liables, des contrôles d’exécution ou du travail persistant de harness : utilisez ACP
- vous voulez un simple repli texte local via la CLI brute : utilisez les backends CLI

## Sessions liées

### Liaisons à la conversation courante

Utilisez `/acp spawn <harness> --bind here` lorsque vous voulez que la conversation actuelle devienne un espace de travail ACP durable sans créer de fil enfant.

Comportement :

- OpenClaw continue de gérer le transport du canal, l’authentification, la sécurité et la livraison.
- La conversation actuelle est épinglée à la clé de session ACP créée.
- Les messages de suivi dans cette conversation sont acheminés vers la même session ACP.
- `/new` et `/reset` réinitialisent sur place cette même session ACP liée.
- `/acp close` ferme la session et supprime la liaison de la conversation actuelle.

Ce que cela signifie en pratique :

- `--bind here` conserve la même surface de chat. Sur Discord, le canal actuel reste le canal actuel.
- `--bind here` peut tout de même créer une nouvelle session ACP si vous lancez un nouveau travail. La liaison attache cette session à la conversation actuelle.
- `--bind here` ne crée pas à lui seul un fil Discord enfant ni un sujet Telegram.
- L’exécution ACP peut toujours avoir son propre répertoire de travail (`cwd`) ou un espace de travail géré par le backend sur disque. Cet espace de travail d’exécution est distinct de la surface de chat et n’implique pas un nouveau fil de messagerie.
- Si vous créez une session vers un autre agent ACP et ne passez pas `--cwd`, OpenClaw hérite par défaut de l’espace de travail de **l’agent cible**, et non de celui du demandeur.
- Si ce chemin d’espace de travail hérité est manquant (`ENOENT`/`ENOTDIR`), OpenClaw revient au `cwd` par défaut du backend au lieu de réutiliser silencieusement le mauvais arbre.
- Si l’espace de travail hérité existe mais n’est pas accessible (par exemple `EACCES`), la création renvoie l’erreur réelle d’accès au lieu d’abandonner `cwd`.

Modèle mental :

- surface de chat : là où les personnes continuent à parler (`canal Discord`, `sujet Telegram`, `chat iMessage`)
- session ACP : l’état d’exécution durable Codex/Claude/Gemini vers lequel OpenClaw achemine
- fil/sujet enfant : surface de messagerie supplémentaire facultative créée uniquement par `--thread ...`
- espace de travail d’exécution : l’emplacement du système de fichiers où le harness s’exécute (`cwd`, checkout du dépôt, espace de travail du backend)

Exemples :

- `/acp spawn codex --bind here` : garder ce chat, créer ou rattacher une session ACP Codex, et y acheminer les futurs messages
- `/acp spawn codex --thread auto` : OpenClaw peut créer un fil/sujet enfant et y lier la session ACP
- `/acp spawn codex --bind here --cwd /workspace/repo` : même liaison de chat que ci-dessus, mais Codex s’exécute dans `/workspace/repo`

Prise en charge de la liaison à la conversation courante :

- Les canaux de chat/message qui annoncent la prise en charge de la liaison à la conversation courante peuvent utiliser `--bind here` via le chemin partagé de liaison de conversation.
- Les canaux avec une sémantique personnalisée de fil/sujet peuvent toujours fournir une canonicalisation spécifique au canal derrière la même interface partagée.
- `--bind here` signifie toujours « lier la conversation courante sur place ».
- Les liaisons génériques à la conversation courante utilisent le stockage partagé de liaison OpenClaw et survivent aux redémarrages normaux de la passerelle.

Remarques :

- `--bind here` et `--thread ...` s’excluent mutuellement dans `/acp spawn`.
- Sur Discord, `--bind here` lie sur place le canal ou fil actuel. `spawnAcpSessions` n’est requis que lorsque OpenClaw doit créer un fil enfant pour `--thread auto|here`.
- Si le canal actif n’expose pas les liaisons ACP à la conversation courante, OpenClaw renvoie un message clair indiquant que ce n’est pas pris en charge.
- `resume` et les questions de « nouvelle session » sont des questions de session ACP, pas des questions de canal. Vous pouvez réutiliser ou remplacer l’état d’exécution sans changer la surface de chat actuelle.

### Sessions liées à un fil

Lorsque les liaisons de fils sont activées pour un adaptateur de canal, les sessions ACP peuvent être liées à des fils :

- OpenClaw lie un fil à une session ACP cible.
- Les messages de suivi dans ce fil sont acheminés vers la session ACP liée.
- La sortie ACP est renvoyée dans le même fil.
- La défocalisation/fermeture/archivage/expiration par inactivité ou par âge maximal supprime la liaison.

La prise en charge des liaisons de fils dépend de l’adaptateur. Si l’adaptateur du canal actif ne la prend pas en charge, OpenClaw renvoie un message clair indiquant que c’est non pris en charge/indisponible.

Drapeaux de fonctionnalité requis pour l’ACP lié à un fil :

- `acp.enabled=true`
- `acp.dispatch.enabled` est activé par défaut (définissez `false` pour suspendre la distribution ACP)
- drapeau d’adaptateur de canal pour création de fil ACP activé (spécifique à l’adaptateur)
  - Discord : `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram : `channels.telegram.threadBindings.spawnAcpSessions=true`

### Canaux prenant en charge les fils

- Tout adaptateur de canal exposant une capacité de liaison de session/fil.
- Prise en charge intégrée actuelle :
  - fils/canaux Discord
  - sujets Telegram (sujets de forum dans les groupes/supergroupes et sujets de MP)
- Les canaux de plugin peuvent ajouter la prise en charge via la même interface de liaison.

## Paramètres spécifiques au canal

Pour les flux non éphémères, configurez des liaisons ACP persistantes dans des entrées de niveau supérieur `bindings[]`.

### Modèle de liaison

- `bindings[].type="acp"` marque une liaison persistante de conversation ACP.
- `bindings[].match` identifie la conversation cible :
  - canal ou fil Discord : `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - sujet de forum Telegram : `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - chat de groupe/MP BlueBubbles : `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Préférez `chat_id:*` ou `chat_identifier:*` pour des liaisons de groupe stables.
  - chat de groupe/MP iMessage : `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Préférez `chat_id:*` pour des liaisons de groupe stables.
- `bindings[].agentId` est l’ID de l’agent OpenClaw propriétaire.
- Les remplacements ACP facultatifs se trouvent sous `bindings[].acp` :
  - `mode` (`persistent` ou `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### Valeurs par défaut d’exécution par agent

Utilisez `agents.list[].runtime` pour définir une seule fois les valeurs par défaut ACP par agent :

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (ID de harness, par exemple `codex` ou `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

Ordre de priorité des remplacements pour les sessions ACP liées :

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

- OpenClaw s’assure que la session ACP configurée existe avant utilisation.
- Les messages dans ce canal ou sujet sont acheminés vers la session ACP configurée.
- Dans les conversations liées, `/new` et `/reset` réinitialisent sur place cette même clé de session ACP.
- Les liaisons d’exécution temporaires (par exemple créées par des flux de focalisation de fil) continuent de s’appliquer lorsqu’elles sont présentes.
- Pour les créations ACP inter-agents sans `cwd` explicite, OpenClaw hérite de l’espace de travail de l’agent cible depuis la configuration de l’agent.
- Les chemins d’espace de travail hérités manquants reviennent au `cwd` par défaut du backend ; les véritables échecs d’accès sont remontés comme erreurs de création.

## Démarrer des sessions ACP (interfaces)

### Depuis `sessions_spawn`

Utilisez `runtime: "acp"` pour démarrer une session ACP depuis un tour d’agent ou un appel d’outil.

```json
{
  "task": "Ouvre le dépôt et résume les tests en échec",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

Remarques :

- `runtime` vaut par défaut `subagent`, donc définissez explicitement `runtime: "acp"` pour les sessions ACP.
- Si `agentId` est omis, OpenClaw utilise `acp.defaultAgent` lorsqu’il est configuré.
- `mode: "session"` requiert `thread: true` pour conserver une conversation liée persistante.

Détails de l’interface :

- `task` (obligatoire) : prompt initial envoyé à la session ACP.
- `runtime` (obligatoire pour ACP) : doit être `"acp"`.
- `agentId` (facultatif) : ID de harness ACP cible. Se rabat sur `acp.defaultAgent` s’il est défini.
- `thread` (facultatif, par défaut `false`) : demande un flux de liaison de fil lorsque pris en charge.
- `mode` (facultatif) : `run` (ponctuel) ou `session` (persistant).
  - la valeur par défaut est `run`
  - si `thread: true` et que le mode est omis, OpenClaw peut par défaut adopter un comportement persistant selon le chemin d’exécution
  - `mode: "session"` requiert `thread: true`
- `cwd` (facultatif) : répertoire de travail demandé pour l’exécution (validé par la politique backend/exécution). S’il est omis, la création ACP hérite de l’espace de travail de l’agent cible lorsqu’il est configuré ; les chemins hérités manquants reviennent aux valeurs par défaut du backend, tandis que les véritables erreurs d’accès sont renvoyées.
- `label` (facultatif) : libellé orienté opérateur utilisé dans le texte de session/bannière.
- `resumeSessionId` (facultatif) : reprendre une session ACP existante au lieu d’en créer une nouvelle. L’agent rejoue son historique de conversation via `session/load`. Requiert `runtime: "acp"`.
- `streamTo` (facultatif) : `"parent"` diffuse les résumés de progression de l’exécution ACP initiale vers la session demandeuse sous forme d’événements système.
  - Lorsque disponible, les réponses acceptées incluent `streamLogPath` pointant vers un journal JSONL limité à la session (`<sessionId>.acp-stream.jsonl`) que vous pouvez suivre pour obtenir l’historique complet du relais.

### Reprendre une session existante

Utilisez `resumeSessionId` pour continuer une session ACP précédente au lieu de repartir de zéro. L’agent rejoue son historique de conversation via `session/load`, ce qui lui permet de reprendre avec tout le contexte précédent.

```json
{
  "task": "Continue là où nous nous étions arrêtés — corrige les échecs de test restants",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

Cas d’usage courants :

- Transférer une session Codex de votre ordinateur portable à votre téléphone — demandez à votre agent de reprendre là où vous vous êtes arrêté
- Continuer une session de code commencée de façon interactive dans la CLI, maintenant de façon headless via votre agent
- Reprendre un travail interrompu par un redémarrage de passerelle ou un délai d’inactivité

Remarques :

- `resumeSessionId` requiert `runtime: "acp"` — renvoie une erreur s’il est utilisé avec l’exécution de sous-agent.
- `resumeSessionId` restaure l’historique de conversation ACP amont ; `thread` et `mode` s’appliquent toujours normalement à la nouvelle session OpenClaw que vous créez, donc `mode: "session"` requiert toujours `thread: true`.
- L’agent cible doit prendre en charge `session/load` (Codex et Claude Code le font).
- Si l’ID de session est introuvable, la création échoue avec une erreur claire — aucun repli silencieux vers une nouvelle session.

### Test de fumée opérateur

Utilisez ceci après un déploiement de passerelle si vous voulez une vérification rapide en conditions réelles que la création ACP
fonctionne réellement de bout en bout, et ne se limite pas à réussir les tests unitaires.

Validation recommandée :

1. Vérifiez la version/le commit de la passerelle déployée sur l’hôte cible.
2. Confirmez que la source déployée inclut l’acceptation de lignée ACP dans
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. Ouvrez une session pont ACPX temporaire vers un agent actif (par exemple
   `razor(main)` sur `jpclawhq`).
4. Demandez à cet agent d’appeler `sessions_spawn` avec :
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - tâche : `Reply with exactly LIVE-ACP-SPAWN-OK`
5. Vérifiez que l’agent signale :
   - `accepted=yes`
   - une vraie `childSessionKey`
   - aucune erreur de validateur
6. Nettoyez la session pont ACPX temporaire.

Exemple de prompt à l’agent actif :

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

Remarques :

- Gardez ce test de fumée sur `mode: "run"` sauf si vous testez intentionnellement
  des sessions ACP persistantes liées à un fil.
- N’exigez pas `streamTo: "parent"` pour la validation de base. Ce chemin dépend
  des capacités du demandeur/de la session et constitue une vérification d’intégration distincte.
- Considérez les tests `mode: "session"` liés à un fil comme un second passage d’intégration plus riche
  depuis un vrai fil Discord ou sujet Telegram.

## Compatibilité sandbox

Les sessions ACP s’exécutent actuellement sur l’exécution de l’hôte, et non dans la sandbox OpenClaw.

Limites actuelles :

- Si la session demandeuse est sandboxée, les créations ACP sont bloquées à la fois pour `sessions_spawn({ runtime: "acp" })` et `/acp spawn`.
  - Erreur : `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- `sessions_spawn` avec `runtime: "acp"` ne prend pas en charge `sandbox: "require"`.
  - Erreur : `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

Utilisez `runtime: "subagent"` lorsque vous avez besoin d’une exécution imposée par sandbox.

### Depuis la commande `/acp`

Utilisez `/acp spawn` pour un contrôle opérateur explicite depuis le chat lorsque nécessaire.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

Drapeaux principaux :

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

Consultez [Commandes slash](/fr/tools/slash-commands).

## Résolution de cible de session

La plupart des actions `/acp` acceptent une cible de session facultative (`session-key`, `session-id` ou `session-label`).

Ordre de résolution :

1. Argument de cible explicite (ou `--session` pour `/acp steer`)
   - essaie d’abord la clé
   - puis l’ID de session de forme UUID
   - puis le libellé
2. Liaison du fil courant (si cette conversation/ce fil est lié à une session ACP)
3. Repli sur la session demandeuse actuelle

Les liaisons à la conversation courante et les liaisons de fils participent toutes deux à l’étape 2.

Si aucune cible n’est résolue, OpenClaw renvoie une erreur claire (`Unable to resolve session target: ...`).

## Modes de liaison lors de la création

`/acp spawn` prend en charge `--bind here|off`.

| Mode   | Comportement                                                            |
| ------ | ----------------------------------------------------------------------- |
| `here` | Lie sur place la conversation active actuelle ; échoue si aucune n’est active. |
| `off`  | Ne crée pas de liaison à la conversation courante.                      |

Remarques :

- `--bind here` est le chemin opérateur le plus simple pour « faire de ce canal ou chat un espace propulsé par Codex ».
- `--bind here` ne crée pas de fil enfant.
- `--bind here` n’est disponible que sur les canaux exposant la prise en charge de la liaison à la conversation courante.
- `--bind` et `--thread` ne peuvent pas être combinés dans le même appel `/acp spawn`.

## Modes de fil lors de la création

`/acp spawn` prend en charge `--thread auto|here|off`.

| Mode   | Comportement                                                                                         |
| ------ | ---------------------------------------------------------------------------------------------------- |
| `auto` | Dans un fil actif : lie ce fil. Hors d’un fil : crée/lie un fil enfant lorsque pris en charge.      |
| `here` | Requiert le fil actif actuel ; échoue si vous n’êtes pas dans un fil.                               |
| `off`  | Pas de liaison. La session démarre sans liaison.                                                    |

Remarques :

- Sur les surfaces sans liaison de fil, le comportement par défaut est en pratique `off`.
- La création liée à un fil nécessite la prise en charge par la politique du canal :
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

`/acp status` affiche les options d’exécution effectives et, lorsqu’ils sont disponibles, les identifiants de session au niveau de l’exécution et du backend.

Certains contrôles dépendent des capacités du backend. Si un backend ne prend pas en charge un contrôle, OpenClaw renvoie une erreur claire indiquant que ce contrôle n’est pas pris en charge.

## Guide pratique des commandes ACP

| Commande             | Ce qu’elle fait                                                | Exemple                                                       |
| -------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Crée une session ACP ; liaison courante ou de fil facultative. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Annule le tour en cours pour la session cible.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Envoie une instruction de pilotage à une session en cours.     | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Ferme la session et dissocie les cibles de fil.                | `/acp close`                                                  |
| `/acp status`        | Affiche backend, mode, état, options d’exécution, capacités.   | `/acp status`                                                 |
| `/acp set-mode`      | Définit le mode d’exécution pour la session cible.             | `/acp set-mode plan`                                          |
| `/acp set`           | Écriture générique d’option de configuration d’exécution.      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Définit un remplacement du répertoire de travail d’exécution.  | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Définit le profil de politique d’approbation.                  | `/acp permissions strict`                                     |
| `/acp timeout`       | Définit le délai d’exécution (secondes).                       | `/acp timeout 120`                                            |
| `/acp model`         | Définit un remplacement du modèle d’exécution.                 | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Supprime les remplacements d’options d’exécution de session.   | `/acp reset-options`                                          |
| `/acp sessions`      | Liste les sessions ACP récentes depuis le stockage.            | `/acp sessions`                                               |
| `/acp doctor`        | Santé du backend, capacités, correctifs exploitables.          | `/acp doctor`                                                 |
| `/acp install`       | Affiche des étapes déterministes d’installation et d’activation. | `/acp install`                                              |

`/acp sessions` lit le stockage de la session courante liée ou demandeuse. Les commandes qui acceptent les jetons `session-key`, `session-id` ou `session-label` résolvent les cibles via la découverte de sessions de la passerelle, y compris les racines personnalisées `session.store` par agent.

## Correspondance des options d’exécution

`/acp` propose des commandes pratiques et un setter générique.

Opérations équivalentes :

- `/acp model <id>` correspond à la clé de configuration d’exécution `model`.
- `/acp permissions <profile>` correspond à la clé de configuration d’exécution `approval_policy`.
- `/acp timeout <seconds>` correspond à la clé de configuration d’exécution `timeout`.
- `/acp cwd <path>` met directement à jour le remplacement `cwd` d’exécution.
- `/acp set <key> <value>` est le chemin générique.
  - Cas particulier : `key=cwd` utilise le chemin de remplacement `cwd`.
- `/acp reset-options` efface tous les remplacements d’exécution pour la session cible.

## Prise en charge des harnesses acpx (actuelle)

Alias de harness intégrés actuels d’acpx :

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

Quand OpenClaw utilise le backend acpx, préférez ces valeurs pour `agentId` sauf si votre configuration acpx définit des alias d’agent personnalisés.
Si votre installation locale de Cursor expose encore ACP comme `agent acp`, remplacez la commande de l’agent `cursor` dans votre configuration acpx au lieu de modifier la valeur intégrée par défaut.

L’utilisation directe de la CLI acpx peut aussi cibler des adaptateurs arbitraires via `--agent <command>`, mais cette échappatoire brute est une fonctionnalité de la CLI acpx (et non du chemin normal `agentId` d’OpenClaw).

## Configuration requise

Base ACP du cœur :

```json5
{
  acp: {
    enabled: true,
    // Facultatif. La valeur par défaut est true ; définissez false pour suspendre la distribution ACP tout en conservant les contrôles /acp.
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

La configuration de liaison de fil est spécifique à l’adaptateur de canal. Exemple pour Discord :

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

Si la création ACP liée à un fil ne fonctionne pas, vérifiez d’abord le drapeau de fonctionnalité de l’adaptateur :

- Discord : `channels.discord.threadBindings.spawnAcpSessions=true`

Les liaisons à la conversation courante ne requièrent pas la création d’un fil enfant. Elles nécessitent un contexte de conversation actif et un adaptateur de canal qui expose les liaisons de conversation ACP.

Consultez [Référence de configuration](/fr/gateway/configuration-reference).

## Configuration du plugin pour le backend acpx

Les nouvelles installations livrent le plugin d’exécution `acpx` inclus activé par défaut, donc ACP
fonctionne généralement sans étape d’installation manuelle du plugin.

Commencez par :

```text
/acp doctor
```

Si vous avez désactivé `acpx`, l’avez refusé via `plugins.allow` / `plugins.deny`, ou si vous voulez
basculer vers un checkout local de développement, utilisez le chemin explicite du plugin :

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

Installation locale de l’espace de travail pendant le développement :

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Puis vérifiez l’état du backend :

```text
/acp doctor
```

### Configuration de la commande et de la version d’acpx

Par défaut, le plugin de backend acpx inclus (`acpx`) utilise le binaire épinglé local au plugin :

1. La commande pointe par défaut vers `node_modules/.bin/acpx` local au plugin à l’intérieur du paquet du plugin ACPX.
2. La version attendue correspond par défaut à l’épinglage de l’extension.
3. Au démarrage, le backend ACP est immédiatement enregistré comme non prêt.
4. Une tâche d’assurance en arrière-plan vérifie `acpx --version`.
5. Si le binaire local au plugin est manquant ou ne correspond pas, il exécute :
   `npm install --omit=dev --no-save acpx@<pinned>` puis revérifie.

Vous pouvez remplacer commande/version dans la configuration du plugin :

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
- Les chemins relatifs sont résolus à partir du répertoire de travail OpenClaw.
- `expectedVersion: "any"` désactive la vérification stricte de version.
- Lorsque `command` pointe vers un binaire/chemin personnalisé, l’auto-installation locale au plugin est désactivée.
- Le démarrage d’OpenClaw reste non bloquant pendant l’exécution de la vérification de santé du backend.

Consultez [Plugins](/fr/tools/plugin).

### Installation automatique des dépendances

Lorsque vous installez OpenClaw globalement avec `npm install -g openclaw`, les dépendances d’exécution acpx
(binaires spécifiques à la plateforme) sont installées automatiquement
via un hook postinstall. Si l’installation automatique échoue, la passerelle démarre quand même
normalement et signale la dépendance manquante via `openclaw acp doctor`.

### Pont MCP des outils de plugin

Par défaut, les sessions ACPX **n’exposent pas** les outils enregistrés par plugin OpenClaw
au harness ACP.

Si vous voulez que des agents ACP tels que Codex ou Claude Code puissent appeler des
outils de plugin OpenClaw installés tels que rappel/stockage mémoire, activez le pont dédié :

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Ce que cela fait :

- Injecte un serveur MCP intégré nommé `openclaw-plugin-tools` dans le bootstrap de session ACPX.
- Expose les outils de plugin déjà enregistrés par les plugins OpenClaw installés et activés.
- Garde la fonctionnalité explicite et désactivée par défaut.

Remarques de sécurité et de confiance :

- Cela élargit la surface d’outils du harness ACP.
- Les agents ACP n’obtiennent l’accès qu’aux outils de plugin déjà actifs dans la passerelle.
- Considérez cela comme la même frontière de confiance que celle consistant à laisser ces plugins s’exécuter dans OpenClaw lui-même.
- Examinez les plugins installés avant de l’activer.

Les `mcpServers` personnalisés continuent de fonctionner comme avant. Le pont intégré plugin-tools est
un confort supplémentaire optionnel, pas un remplacement de la configuration générique des serveurs MCP.

### Configuration du délai d’exécution

Le plugin `acpx` inclus définit par défaut les tours d’exécution embarqués à un délai de 120 secondes.
Cela laisse suffisamment de temps aux harnesses plus lents comme Gemini CLI pour terminer
le démarrage et l’initialisation ACP. Remplacez cette valeur si votre hôte a besoin d’une autre
limite d’exécution :

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

Redémarrez la passerelle après avoir modifié cette valeur.

## Configuration des permissions

Les sessions ACP s’exécutent de manière non interactive — il n’y a pas de TTY pour approuver ou refuser les invites d’autorisation d’écriture de fichiers et d’exécution shell. Le plugin acpx fournit deux clés de configuration qui contrôlent la façon dont les permissions sont gérées :

Ces permissions de harness ACPX sont distinctes des approbations exec OpenClaw et distinctes des drapeaux de contournement propres aux fournisseurs des backends CLI, comme Claude CLI `--permission-mode bypassPermissions`. ACPX `approve-all` est le commutateur de secours au niveau du harness pour les sessions ACP.

### `permissionMode`

Contrôle quelles opérations l’agent du harness peut effectuer sans invite.

| Valeur          | Comportement                                              |
| --------------- | --------------------------------------------------------- |
| `approve-all`   | Approuve automatiquement toutes les écritures de fichiers et commandes shell. |
| `approve-reads` | Approuve automatiquement les lectures uniquement ; les écritures et exécutions nécessitent des invites. |
| `deny-all`      | Refuse toutes les invites de permission.                  |

### `nonInteractivePermissions`

Contrôle ce qui se passe lorsqu’une invite de permission devrait être affichée mais qu’aucun TTY interactif n’est disponible (ce qui est toujours le cas pour les sessions ACP).

| Valeur | Comportement                                                      |
| ------ | ----------------------------------------------------------------- |
| `fail` | Interrompt la session avec `AcpRuntimeError`. **(par défaut)**    |
| `deny` | Refuse silencieusement la permission et continue (dégradation progressive). |

### Configuration

Définissez via la configuration du plugin :

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Redémarrez la passerelle après avoir modifié ces valeurs.

> **Important :** OpenClaw utilise actuellement par défaut `permissionMode=approve-reads` et `nonInteractivePermissions=fail`. Dans les sessions ACP non interactives, toute écriture ou exécution qui déclenche une invite de permission peut échouer avec `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> Si vous devez restreindre les permissions, définissez `nonInteractivePermissions` sur `deny` afin que les sessions se dégradent proprement au lieu de planter.

## Dépannage

| Symptôme                                                                    | Cause probable                                                                  | Correctif                                                                                                                                                         |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured`                                     | Plugin de backend manquant ou désactivé.                                        | Installez et activez le plugin de backend, puis exécutez `/acp doctor`.                                                                                          |
| `ACP is disabled by policy (acp.enabled=false)`                             | ACP désactivé globalement.                                                      | Définissez `acp.enabled=true`.                                                                                                                                     |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | Distribution désactivée depuis les messages normaux du fil.                     | Définissez `acp.dispatch.enabled=true`.                                                                                                                            |
| `ACP agent "<id>" is not allowed by policy`                                 | Agent absent de l’allowlist.                                                    | Utilisez un `agentId` autorisé ou mettez à jour `acp.allowedAgents`.                                                                                              |
| `Unable to resolve session target: ...`                                     | Mauvais jeton clé/id/libellé.                                                   | Exécutez `/acp sessions`, copiez la clé/le libellé exact, puis réessayez.                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation` | `--bind here` utilisé sans conversation active pouvant être liée.               | Déplacez-vous vers le chat/canal cible et réessayez, ou utilisez une création non liée.                                                                         |
| `Conversation bindings are unavailable for <channel>.`                      | L’adaptateur ne prend pas en charge les liaisons ACP à la conversation courante. | Utilisez `/acp spawn ... --thread ...` lorsque c’est pris en charge, configurez des `bindings[]` de niveau supérieur, ou utilisez un canal pris en charge.    |
| `--thread here requires running /acp spawn inside an active ... thread`     | `--thread here` utilisé hors d’un contexte de fil.                              | Déplacez-vous vers le fil cible ou utilisez `--thread auto`/`off`.                                                                                               |
| `Only <user-id> can rebind this channel/conversation/thread.`               | Un autre utilisateur possède la cible de liaison active.                        | Reliez à nouveau en tant que propriétaire ou utilisez une autre conversation ou un autre fil.                                                                    |
| `Thread bindings are unavailable for <channel>.`                            | L’adaptateur ne prend pas en charge les liaisons de fils.                       | Utilisez `--thread off` ou passez à un adaptateur/canal pris en charge.                                                                                          |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | L’exécution ACP s’exécute côté hôte ; la session demandeuse est sandboxée.      | Utilisez `runtime="subagent"` depuis les sessions sandboxées, ou lancez la création ACP depuis une session non sandboxée.                                       |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | `sandbox="require"` demandé pour l’exécution ACP.                               | Utilisez `runtime="subagent"` pour un sandboxing obligatoire, ou utilisez ACP avec `sandbox="inherit"` depuis une session non sandboxée.                       |
| Métadonnées ACP manquantes pour la session liée                             | Métadonnées de session ACP obsolètes/supprimées.                                | Recréez avec `/acp spawn`, puis reliez de nouveau/focalisez le fil.                                                                                               |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | `permissionMode` bloque les écritures/exécutions dans la session ACP non interactive. | Définissez `plugins.entries.acpx.config.permissionMode` sur `approve-all` et redémarrez la passerelle. Consultez [Configuration des permissions](#configuration-des-permissions). |
| La session ACP échoue tôt avec peu de sortie                                | Les invites de permission sont bloquées par `permissionMode`/`nonInteractivePermissions`. | Vérifiez les logs de la passerelle pour `AcpRuntimeError`. Pour toutes les permissions, définissez `permissionMode=approve-all` ; pour une dégradation progressive, définissez `nonInteractivePermissions=deny`. |
| La session ACP reste bloquée indéfiniment après la fin du travail           | Le processus du harness est terminé, mais la session ACP n’a pas signalé sa fin. | Surveillez avec `ps aux \| grep acpx` ; tuez manuellement les processus obsolètes.                                                                               |
