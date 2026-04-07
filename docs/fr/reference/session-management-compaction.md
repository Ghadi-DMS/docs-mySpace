---
read_when:
    - Vous devez déboguer les ID de session, les JSONL de transcription ou les champs de sessions.json
    - Vous modifiez le comportement de compaction automatique ou ajoutez une maintenance « pré-compaction »
    - Vous voulez implémenter des vidages mémoire ou des tours système silencieux
summary: 'Analyse approfondie : magasin de sessions + transcriptions, cycle de vie et internals de compaction (auto)'
title: Analyse approfondie de la gestion des sessions
x-i18n:
    generated_at: "2026-04-07T06:54:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: e379d624dd7808d3af25ed011079268ce6a9da64bb3f301598884ad4c46ab091
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Gestion des sessions et compaction (analyse approfondie)

Ce document explique comment OpenClaw gère les sessions de bout en bout :

- **Routage des sessions** (comment les messages entrants sont associés à une `sessionKey`)
- **Magasin de sessions** (`sessions.json`) et ce qu'il suit
- **Persistance des transcriptions** (`*.jsonl`) et leur structure
- **Hygiène des transcriptions** (corrections spécifiques au fournisseur avant les exécutions)
- **Limites de contexte** (fenêtre de contexte vs jetons suivis)
- **Compaction** (compaction manuelle + automatique) et où raccorder le travail de pré-compaction
- **Maintenance silencieuse** (par ex. écritures mémoire qui ne doivent pas produire de sortie visible par l'utilisateur)

Si vous voulez d'abord une vue d'ensemble de plus haut niveau, commencez par :

- [/concepts/session](/fr/concepts/session)
- [/concepts/compaction](/fr/concepts/compaction)
- [/concepts/memory](/fr/concepts/memory)
- [/concepts/memory-search](/fr/concepts/memory-search)
- [/concepts/session-pruning](/fr/concepts/session-pruning)
- [/reference/transcript-hygiene](/fr/reference/transcript-hygiene)

---

## Source de vérité : la Gateway

OpenClaw est conçu autour d'un seul **processus Gateway** qui possède l'état des sessions.

- Les interfaces utilisateur (application macOS, web Control UI, TUI) doivent interroger la Gateway pour les listes de sessions et les comptes de jetons.
- En mode distant, les fichiers de session se trouvent sur l'hôte distant ; « vérifier vos fichiers locaux sur le Mac » ne reflétera pas ce que la Gateway utilise.

---

## Deux couches de persistance

OpenClaw conserve les sessions dans deux couches :

1. **Magasin de sessions (`sessions.json`)**
   - Table clé/valeur : `sessionKey -> SessionEntry`
   - Petit, mutable, sûr à modifier (ou à supprimer des entrées)
   - Suit les métadonnées de session (ID de session actuel, dernière activité, bascules, compteurs de jetons, etc.)

2. **Transcription (`<sessionId>.jsonl`)**
   - Transcription en ajout seul avec structure en arbre (les entrées ont `id` + `parentId`)
   - Stocke la conversation réelle + les appels d'outils + les résumés de compaction
   - Utilisée pour reconstruire le contexte du modèle pour les prochains tours

---

## Emplacements sur disque

Par agent, sur l'hôte Gateway :

- Magasin : `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcriptions : `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Sessions de sujet Telegram : `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw résout ces chemins via `src/config/sessions.ts`.

---

## Maintenance du magasin et contrôles disque

La persistance des sessions dispose de contrôles automatiques de maintenance (`session.maintenance`) pour `sessions.json` et les artefacts de transcription :

- `mode` : `warn` (par défaut) ou `enforce`
- `pruneAfter` : seuil d'âge des entrées obsolètes (par défaut `30d`)
- `maxEntries` : limite d'entrées dans `sessions.json` (par défaut `500`)
- `rotateBytes` : rotation de `sessions.json` lorsqu'il devient trop volumineux (par défaut `10mb`)
- `resetArchiveRetention` : rétention des archives de transcription `*.reset.<timestamp>` (par défaut : identique à `pruneAfter` ; `false` désactive le nettoyage)
- `maxDiskBytes` : budget facultatif pour le répertoire des sessions
- `highWaterBytes` : cible facultative après nettoyage (par défaut `80%` de `maxDiskBytes`)

Ordre d'application pour le nettoyage du budget disque (`mode: "enforce"`) :

1. Supprimer d'abord les artefacts de transcription archivés ou orphelins les plus anciens.
2. Si l'utilisation reste au-dessus de la cible, évincer les entrées de session les plus anciennes et leurs fichiers de transcription.
3. Continuer jusqu'à ce que l'utilisation soit inférieure ou égale à `highWaterBytes`.

En `mode: "warn"`, OpenClaw signale les évictions potentielles mais ne modifie pas le magasin ni les fichiers.

Exécuter la maintenance à la demande :

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Sessions cron et journaux d'exécution

Les exécutions cron isolées créent également des entrées/transcriptions de session, et elles disposent de contrôles de rétention dédiés :

- `cron.sessionRetention` (par défaut `24h`) supprime les anciennes sessions d'exécution cron isolées du magasin de sessions (`false` désactive).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` nettoient les fichiers `~/.openclaw/cron/runs/<jobId>.jsonl` (valeurs par défaut : `2_000_000` octets et `2000` lignes).

---

## Clés de session (`sessionKey`)

Une `sessionKey` identifie _dans quel compartiment de conversation_ vous vous trouvez (routage + isolation).

Schémas courants :

- Chat principal/direct (par agent) : `agent:<agentId>:<mainKey>` (par défaut `main`)
- Groupe : `agent:<agentId>:<channel>:group:<id>`
- Salon/canal (Discord/Slack) : `agent:<agentId>:<channel>:channel:<id>` ou `...:room:<id>`
- Cron : `cron:<job.id>`
- Webhook : `hook:<uuid>` (sauf remplacement)

Les règles canoniques sont documentées dans [/concepts/session](/fr/concepts/session).

---

## ID de session (`sessionId`)

Chaque `sessionKey` pointe vers une `sessionId` actuelle (le fichier de transcription qui prolonge la conversation).

Règles pratiques :

- **Réinitialisation** (`/new`, `/reset`) crée une nouvelle `sessionId` pour cette `sessionKey`.
- **Réinitialisation quotidienne** (par défaut à 4:00 du matin, heure locale sur l'hôte gateway) crée une nouvelle `sessionId` au message suivant après la limite de réinitialisation.
- **Expiration d'inactivité** (`session.reset.idleMinutes` ou ancien `session.idleMinutes`) crée une nouvelle `sessionId` lorsqu'un message arrive après la fenêtre d'inactivité. Lorsque les réinitialisations quotidienne + d'inactivité sont toutes deux configurées, la première à expirer l'emporte.
- **Garde de fork du parent de thread** (`session.parentForkMaxTokens`, par défaut `100000`) saute le fork de transcription parent lorsque la session parente est déjà trop grande ; le nouveau thread commence à neuf. Définissez `0` pour désactiver.

Détail d'implémentation : la décision est prise dans `initSessionState()` dans `src/auto-reply/reply/session.ts`.

---

## Schéma du magasin de sessions (`sessions.json`)

Le type de valeur du magasin est `SessionEntry` dans `src/config/sessions.ts`.

Champs clés (non exhaustifs) :

- `sessionId` : ID de transcription actuel (le nom de fichier en est dérivé sauf si `sessionFile` est défini)
- `updatedAt` : horodatage de la dernière activité
- `sessionFile` : remplacement facultatif explicite du chemin de transcription
- `chatType` : `direct | group | room` (aide les UI et la politique d'envoi)
- `provider`, `subject`, `room`, `space`, `displayName` : métadonnées pour l'étiquetage des groupes/canaux
- Bascule :
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (remplacement par session)
- Sélection du modèle :
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Compteurs de jetons (au mieux / dépendants du fournisseur) :
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount` : nombre de compactages automatiques terminés pour cette clé de session
- `memoryFlushAt` : horodatage du dernier vidage mémoire avant compaction
- `memoryFlushCompactionCount` : nombre de compactages au moment du dernier vidage

Le magasin peut être modifié en toute sécurité, mais la Gateway fait autorité : elle peut réécrire ou réhydrater les entrées au fur et à mesure de l'exécution des sessions.

---

## Structure des transcriptions (`*.jsonl`)

Les transcriptions sont gérées par le `SessionManager` de `@mariozechner/pi-coding-agent`.

Le fichier est au format JSONL :

- Première ligne : en-tête de session (`type: "session"`, inclut `id`, `cwd`, `timestamp`, `parentSession` facultatif)
- Puis : entrées de session avec `id` + `parentId` (arbre)

Types d'entrée notables :

- `message` : messages utilisateur/assistant/toolResult
- `custom_message` : messages injectés par extension qui _entrent_ dans le contexte du modèle (peuvent être masqués dans l'UI)
- `custom` : état d'extension qui n'entre _pas_ dans le contexte du modèle
- `compaction` : résumé de compaction conservé avec `firstKeptEntryId` et `tokensBefore`
- `branch_summary` : résumé conservé lors de la navigation dans une branche de l'arbre

OpenClaw ne « corrige » intentionnellement **pas** les transcriptions ; la Gateway utilise `SessionManager` pour les lire/écrire.

---

## Fenêtres de contexte vs jetons suivis

Deux concepts différents sont importants :

1. **Fenêtre de contexte du modèle** : limite stricte par modèle (jetons visibles par le modèle)
2. **Compteurs du magasin de sessions** : statistiques glissantes écrites dans `sessions.json` (utilisées pour `/status` et les tableaux de bord)

Si vous ajustez les limites :

- La fenêtre de contexte provient du catalogue de modèles (et peut être remplacée via la configuration).
- `contextTokens` dans le magasin est une valeur d'estimation/de rapport à l'exécution ; ne la considérez pas comme une garantie stricte.

Pour plus de détails, voir [/token-use](/fr/reference/token-use).

---

## Compaction : ce que c'est

La compaction résume l'ancienne conversation dans une entrée `compaction` conservée dans la transcription et garde les messages récents intacts.

Après compaction, les prochains tours voient :

- Le résumé de compaction
- Les messages après `firstKeptEntryId`

La compaction est **persistante** (contrairement au pruning de session). Voir [/concepts/session-pruning](/fr/concepts/session-pruning).

## Limites de blocs de compaction et appariement des outils

Lorsque OpenClaw divise une longue transcription en blocs de compaction, il garde
les appels d'outils de l'assistant appariés avec leurs entrées `toolResult` correspondantes.

- Si le découpage par part de jetons tombe entre un appel d'outil et son résultat, OpenClaw
  déplace la limite vers le message d'appel d'outil de l'assistant au lieu de séparer
  la paire.
- Si un bloc final de résultat d'outil faisait autrement dépasser la cible au bloc,
  OpenClaw préserve ce bloc d'outil en attente et garde intacte la queue non résumée.
- Les blocs d'appel d'outil abandonnés/en erreur ne maintiennent pas une séparation en attente ouverte.

---

## Quand la compaction automatique se produit (runtime Pi)

Dans l'agent Pi intégré, la compaction automatique se déclenche dans deux cas :

1. **Récupération sur débordement** : le modèle renvoie une erreur de dépassement de contexte
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded`, et variantes similaires selon le fournisseur) → compactage → nouvelle tentative.
2. **Maintenance par seuil** : après un tour réussi, lorsque :

`contextTokens > contextWindow - reserveTokens`

Où :

- `contextWindow` est la fenêtre de contexte du modèle
- `reserveTokens` est la marge réservée pour les prompts + la sortie du modèle suivante

Il s'agit de la sémantique du runtime Pi (OpenClaw consomme les événements, mais c'est Pi qui décide quand compacter).

---

## Paramètres de compaction (`reserveTokens`, `keepRecentTokens`)

Les paramètres de compaction de Pi se trouvent dans les paramètres Pi :

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw applique aussi un seuil de sécurité pour les exécutions intégrées :

- Si `compaction.reserveTokens < reserveTokensFloor`, OpenClaw l'augmente.
- Le seuil par défaut est de `20000` jetons.
- Définissez `agents.defaults.compaction.reserveTokensFloor: 0` pour désactiver le seuil.
- S'il est déjà plus élevé, OpenClaw ne le modifie pas.

Pourquoi : laisser assez de marge pour la « maintenance » sur plusieurs tours (comme les écritures mémoire) avant que la compaction ne devienne inévitable.

Implémentation : `ensurePiCompactionReserveTokens()` dans `src/agents/pi-settings.ts`
(appelé depuis `src/agents/pi-embedded-runner.ts`).

---

## Surfaces visibles par l'utilisateur

Vous pouvez observer la compaction et l'état de la session via :

- `/status` (dans n'importe quelle session de chat)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Mode verbeux : `🧹 Auto-compaction complete` + nombre de compactages

---

## Maintenance silencieuse (`NO_REPLY`)

OpenClaw prend en charge les tours « silencieux » pour les tâches d'arrière-plan lorsque l'utilisateur ne doit pas voir de sortie intermédiaire.

Convention :

- L'assistant commence sa sortie avec le jeton silencieux exact `NO_REPLY` /
  `no_reply` pour indiquer « ne pas envoyer de réponse à l'utilisateur ».
- OpenClaw supprime/masque cela dans la couche de distribution.
- La suppression du jeton silencieux exact est insensible à la casse, donc `NO_REPLY` et
  `no_reply` comptent tous deux lorsque toute la charge utile se résume à ce jeton silencieux.
- Cela ne concerne que les vrais tours d'arrière-plan/sans distribution ; ce n'est pas un raccourci pour
  les requêtes utilisateur ordinaires nécessitant une action.

Depuis `2026.1.10`, OpenClaw supprime aussi le **streaming brouillon/saisie** lorsqu'un
bloc partiel commence par `NO_REPLY`, afin que les opérations silencieuses ne laissent pas fuiter de sortie partielle en milieu de tour.

---

## « Vidage mémoire » pré-compaction (implémenté)

Objectif : avant qu'une compaction automatique ne se produise, exécuter un tour agentique silencieux qui écrit un état durable
sur disque (par ex. `memory/YYYY-MM-DD.md` dans le workspace de l'agent) afin que la compaction ne puisse pas
effacer un contexte critique.

OpenClaw utilise l'approche de **vidage avant seuil** :

1. Surveiller l'utilisation du contexte de la session.
2. Lorsqu'elle franchit un « seuil souple » (en dessous du seuil de compaction de Pi), exécuter une directive silencieuse
   « écrire la mémoire maintenant » vers l'agent.
3. Utiliser le jeton silencieux exact `NO_REPLY` / `no_reply` pour que l'utilisateur ne voie
   rien.

Configuration (`agents.defaults.compaction.memoryFlush`) :

- `enabled` (par défaut : `true`)
- `softThresholdTokens` (par défaut : `4000`)
- `prompt` (message utilisateur pour le tour de vidage)
- `systemPrompt` (prompt système supplémentaire ajouté pour le tour de vidage)

Remarques :

- Le prompt system prompt par défaut incluent une indication `NO_REPLY` pour supprimer
  la distribution.
- Le vidage s'exécute une fois par cycle de compaction (suivi dans `sessions.json`).
- Le vidage ne s'exécute que pour les sessions Pi intégrées (les backends CLI l'ignorent).
- Le vidage est ignoré lorsque le workspace de session est en lecture seule (`workspaceAccess: "ro"` ou `"none"`).
- Voir [Memory](/fr/concepts/memory) pour la disposition des fichiers du workspace et les schémas d'écriture.

Pi expose aussi un hook `session_before_compact` dans l'API d'extension, mais la logique de
vidage d'OpenClaw vit aujourd'hui côté Gateway.

---

## Liste de vérification pour le dépannage

- Clé de session incorrecte ? Commencez par [/concepts/session](/fr/concepts/session) et confirmez la `sessionKey` dans `/status`.
- Incohérence entre magasin et transcription ? Confirmez l'hôte Gateway et le chemin du magasin à partir de `openclaw status`.
- Spam de compaction ? Vérifiez :
  - la fenêtre de contexte du modèle (trop petite)
  - les paramètres de compaction (`reserveTokens` trop élevé pour la fenêtre du modèle peut provoquer une compaction plus précoce)
  - le gonflement dû aux résultats d'outils : activez/ajustez le pruning de session
- Fuite de tours silencieux ? Confirmez que la réponse commence par `NO_REPLY` (jeton exact insensible à la casse) et que vous utilisez une build incluant le correctif de suppression du streaming.
