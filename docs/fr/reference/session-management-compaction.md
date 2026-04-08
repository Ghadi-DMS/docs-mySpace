---
read_when:
    - Vous devez déboguer des ids de session, des fichiers JSONL de transcription ou des champs de sessions.json
    - Vous modifiez le comportement de compaction automatique ou ajoutez des tâches de maintenance « avant compaction »
    - Vous voulez implémenter des vidages mémoire ou des tours système silencieux
summary: 'Analyse approfondie : stockage des sessions + transcriptions, cycle de vie et internals de compaction (auto)'
title: Analyse approfondie de la gestion des sessions
x-i18n:
    generated_at: "2026-04-08T02:18:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: cb1a4048646486693db8943a9e9c6c5bcb205f0ed532b34842de3d0346077454
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Gestion des sessions et compaction (analyse approfondie)

Ce document explique comment OpenClaw gère les sessions de bout en bout :

- **Routage des sessions** (comment les messages entrants sont mappés vers une `sessionKey`)
- **Stockage des sessions** (`sessions.json`) et ce qu’il suit
- **Persistance des transcriptions** (`*.jsonl`) et leur structure
- **Hygiène des transcriptions** (correctifs spécifiques aux fournisseurs avant les exécutions)
- **Limites de contexte** (fenêtre de contexte vs jetons suivis)
- **Compaction** (compaction manuelle + automatique) et où brancher le travail avant compaction
- **Maintenance silencieuse** (par ex. des écritures mémoire qui ne doivent pas produire de sortie visible par l’utilisateur)

Si vous voulez d’abord une vue d’ensemble de plus haut niveau, commencez par :

- [/concepts/session](/fr/concepts/session)
- [/concepts/compaction](/fr/concepts/compaction)
- [/concepts/memory](/fr/concepts/memory)
- [/concepts/memory-search](/fr/concepts/memory-search)
- [/concepts/session-pruning](/fr/concepts/session-pruning)
- [/reference/transcript-hygiene](/fr/reference/transcript-hygiene)

---

## Source de vérité : la Gateway

OpenClaw est conçu autour d’un seul **processus Gateway** qui possède l’état des sessions.

- Les UI (application macOS, web Control UI, TUI) doivent interroger la Gateway pour les listes de sessions et les comptes de jetons.
- En mode distant, les fichiers de session se trouvent sur l’hôte distant ; « vérifier vos fichiers locaux sur votre Mac » ne reflétera pas ce que la Gateway utilise.

---

## Deux couches de persistance

OpenClaw conserve les sessions dans deux couches :

1. **Stockage des sessions (`sessions.json`)**
   - Mappage clé/valeur : `sessionKey -> SessionEntry`
   - Petit, mutable, sûr à modifier (ou à supprimer des entrées)
   - Suit les métadonnées de session (id de session actuel, dernière activité, bascules, compteurs de jetons, etc.)

2. **Transcription (`<sessionId>.jsonl`)**
   - Transcription en ajout seul avec structure en arbre (les entrées ont `id` + `parentId`)
   - Stocke la conversation réelle + les appels d’outils + les résumés de compaction
   - Utilisée pour reconstruire le contexte du modèle lors des prochains tours

---

## Emplacements sur disque

Par agent, sur l’hôte Gateway :

- Stockage : `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcriptions : `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Sessions de sujet Telegram : `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw les résout via `src/config/sessions.ts`.

---

## Maintenance du stockage et contrôles disque

La persistance des sessions dispose de contrôles de maintenance automatiques (`session.maintenance`) pour `sessions.json` et les artefacts de transcription :

- `mode` : `warn` (par défaut) ou `enforce`
- `pruneAfter` : seuil d’âge des entrées obsolètes (par défaut `30d`)
- `maxEntries` : plafond des entrées dans `sessions.json` (par défaut `500`)
- `rotateBytes` : rotation de `sessions.json` lorsqu’il devient trop volumineux (par défaut `10mb`)
- `resetArchiveRetention` : rétention des archives de transcription `*.reset.<timestamp>` (par défaut : identique à `pruneAfter` ; `false` désactive le nettoyage)
- `maxDiskBytes` : budget facultatif pour le répertoire des sessions
- `highWaterBytes` : cible facultative après nettoyage (par défaut `80%` de `maxDiskBytes`)

Ordre d’application pour le nettoyage du budget disque (`mode: "enforce"`) :

1. Supprimer d’abord les plus anciens artefacts de transcription archivés ou orphelins.
2. Si l’utilisation reste au-dessus de la cible, évincer les plus anciennes entrées de session et leurs fichiers de transcription.
3. Continuer jusqu’à ce que l’utilisation soit inférieure ou égale à `highWaterBytes`.

En `mode: "warn"`, OpenClaw signale les évictions potentielles mais ne modifie pas le stockage/les fichiers.

Exécuter la maintenance à la demande :

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Sessions cron et journaux d’exécution

Les exécutions cron isolées créent aussi des entrées de session/transcriptions, et elles ont des contrôles de rétention dédiés :

- `cron.sessionRetention` (par défaut `24h`) supprime les anciennes sessions d’exécution cron isolées du stockage des sessions (`false` désactive).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` réduisent les fichiers `~/.openclaw/cron/runs/<jobId>.jsonl` (par défaut : `2_000_000` octets et `2000` lignes).

---

## Clés de session (`sessionKey`)

Une `sessionKey` identifie _dans quel compartiment de conversation_ vous vous trouvez (routage + isolation).

Modèles courants :

- Chat principal/direct (par agent) : `agent:<agentId>:<mainKey>` (par défaut `main`)
- Groupe : `agent:<agentId>:<channel>:group:<id>`
- Salon/canal (Discord/Slack) : `agent:<agentId>:<channel>:channel:<id>` ou `...:room:<id>`
- Cron : `cron:<job.id>`
- Webhook : `hook:<uuid>` (sauf si remplacé)

Les règles canoniques sont documentées dans [/concepts/session](/fr/concepts/session).

---

## Ids de session (`sessionId`)

Chaque `sessionKey` pointe vers un `sessionId` actuel (le fichier de transcription qui prolonge la conversation).

Règles empiriques :

- **Réinitialisation** (`/new`, `/reset`) crée un nouveau `sessionId` pour cette `sessionKey`.
- **Réinitialisation quotidienne** (par défaut à 4:00 du matin, heure locale sur l’hôte gateway) crée un nouveau `sessionId` au prochain message après la limite de réinitialisation.
- **Expiration par inactivité** (`session.reset.idleMinutes` ou l’ancien `session.idleMinutes`) crée un nouveau `sessionId` lorsqu’un message arrive après la fenêtre d’inactivité. Lorsque la réinitialisation quotidienne et l’inactivité sont toutes deux configurées, celle qui expire en premier l’emporte.
- **Garde-fou de bifurcation du parent de thread** (`session.parentForkMaxTokens`, par défaut `100000`) ignore la bifurcation de la transcription parent lorsque la session parente est déjà trop volumineuse ; le nouveau thread démarre à neuf. Définissez `0` pour désactiver.

Détail d’implémentation : la décision se prend dans `initSessionState()` dans `src/auto-reply/reply/session.ts`.

---

## Schéma du stockage des sessions (`sessions.json`)

Le type de valeur du stockage est `SessionEntry` dans `src/config/sessions.ts`.

Champs clés (liste non exhaustive) :

- `sessionId` : id de transcription actuel (le nom du fichier en est dérivé sauf si `sessionFile` est défini)
- `updatedAt` : horodatage de la dernière activité
- `sessionFile` : remplacement facultatif explicite du chemin de transcription
- `chatType` : `direct | group | room` (utile pour les UI et la politique d’envoi)
- `provider`, `subject`, `room`, `space`, `displayName` : métadonnées pour l’étiquetage des groupes/canaux
- Basculements :
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (remplacement par session)
- Sélection du modèle :
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Compteurs de jetons (au mieux / dépendants du fournisseur) :
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount` : nombre de fois où la compaction automatique s’est terminée pour cette clé de session
- `memoryFlushAt` : horodatage du dernier vidage mémoire avant compaction
- `memoryFlushCompactionCount` : nombre de compactions lors de la dernière exécution du vidage

Le stockage peut être modifié sans danger, mais la Gateway fait autorité : elle peut réécrire ou réhydrater les entrées au fil des exécutions des sessions.

---

## Structure des transcriptions (`*.jsonl`)

Les transcriptions sont gérées par le `SessionManager` de `@mariozechner/pi-coding-agent`.

Le fichier est en JSONL :

- Première ligne : en-tête de session (`type: "session"`, inclut `id`, `cwd`, `timestamp`, `parentSession` facultatif)
- Puis : entrées de session avec `id` + `parentId` (arbre)

Types d’entrée notables :

- `message` : messages utilisateur/assistant/toolResult
- `custom_message` : messages injectés par extension qui _entrent bien_ dans le contexte du modèle (peuvent être masqués dans l’UI)
- `custom` : état d’extension qui _n’entre pas_ dans le contexte du modèle
- `compaction` : résumé de compaction persistant avec `firstKeptEntryId` et `tokensBefore`
- `branch_summary` : résumé persistant lors de la navigation dans une branche de l’arbre

OpenClaw ne « corrige » volontairement **pas** les transcriptions ; la Gateway utilise `SessionManager` pour les lire/écrire.

---

## Fenêtres de contexte vs jetons suivis

Deux concepts différents comptent :

1. **Fenêtre de contexte du modèle** : plafond strict par modèle (jetons visibles par le modèle)
2. **Compteurs du stockage de session** : statistiques glissantes écrites dans `sessions.json` (utilisées pour /status et les tableaux de bord)

Si vous ajustez les limites :

- La fenêtre de contexte provient du catalogue de modèles (et peut être remplacée via la configuration).
- `contextTokens` dans le stockage est une valeur d’estimation/de reporting à l’exécution ; ne la traitez pas comme une garantie stricte.

Pour en savoir plus, voir [/token-use](/fr/reference/token-use).

---

## Compaction : ce que c’est

La compaction résume les parties plus anciennes de la conversation en une entrée `compaction` persistante dans la transcription et conserve les messages récents intacts.

Après compaction, les futurs tours voient :

- Le résumé de compaction
- Les messages après `firstKeptEntryId`

La compaction est **persistante** (contrairement à l’élagage de session). Voir [/concepts/session-pruning](/fr/concepts/session-pruning).

## Limites de morceaux de compaction et appariement des outils

Lorsque OpenClaw découpe une longue transcription en morceaux de compaction, il conserve
les appels d’outil de l’assistant appariés avec leurs entrées `toolResult` correspondantes.

- Si le découpage basé sur la part de jetons tombe entre un appel d’outil et son résultat, OpenClaw
  déplace la limite vers le message d’appel d’outil de l’assistant au lieu de séparer
  la paire.
- Si un bloc final de résultat d’outil pousserait autrement le morceau au-delà de la cible,
  OpenClaw conserve ce bloc d’outil en attente et garde intacte la queue non résumée.
- Les blocs d’appel d’outil abandonnés/en erreur ne maintiennent pas une coupure en attente ouverte.

---

## Quand la compaction automatique se produit (runtime Pi)

Dans l’agent Pi embarqué, la compaction automatique se déclenche dans deux cas :

1. **Récupération après dépassement** : le modèle renvoie une erreur de dépassement de contexte
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded`, et variantes similaires propres aux fournisseurs) → compacter → réessayer.
2. **Maintenance par seuil** : après un tour réussi, lorsque :

`contextTokens > contextWindow - reserveTokens`

Où :

- `contextWindow` est la fenêtre de contexte du modèle
- `reserveTokens` est la marge réservée pour les prompts + la sortie du prochain modèle

Il s’agit de la sémantique du runtime Pi (OpenClaw consomme les événements, mais c’est Pi qui décide quand compacter).

---

## Paramètres de compaction (`reserveTokens`, `keepRecentTokens`)

Les paramètres de compaction de Pi vivent dans les paramètres Pi :

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw applique aussi un seuil de sécurité minimal pour les exécutions embarquées :

- Si `compaction.reserveTokens < reserveTokensFloor`, OpenClaw l’augmente.
- Le seuil minimal par défaut est de `20000` jetons.
- Définissez `agents.defaults.compaction.reserveTokensFloor: 0` pour désactiver ce seuil.
- S’il est déjà plus élevé, OpenClaw ne le modifie pas.

Pourquoi : laisser assez de marge pour la « maintenance » sur plusieurs tours (comme les écritures mémoire) avant que la compaction ne devienne inévitable.

Implémentation : `ensurePiCompactionReserveTokens()` dans `src/agents/pi-settings.ts`
(appelé depuis `src/agents/pi-embedded-runner.ts`).

---

## Fournisseurs de compaction enfichables

Les plugins peuvent enregistrer un fournisseur de compaction via `registerCompactionProvider()` sur l’API du plugin. Lorsque `agents.defaults.compaction.provider` est défini sur un id de fournisseur enregistré, l’extension safeguard délègue la synthèse à ce fournisseur au lieu du pipeline intégré `summarizeInStages`.

- `provider` : id d’un plugin fournisseur de compaction enregistré. Laissez vide pour la synthèse LLM par défaut.
- Définir un `provider` force `mode: "safeguard"`.
- Les fournisseurs reçoivent les mêmes instructions de compaction et la même politique de préservation des identifiants que le chemin intégré.
- Le safeguard préserve toujours le contexte de suffixe des tours récents et des tours fractionnés après la sortie du fournisseur.
- Si le fournisseur échoue ou renvoie un résultat vide, OpenClaw revient automatiquement à la synthèse LLM intégrée.
- Les signaux d’abandon/timeout sont relancés (et non absorbés) pour respecter l’annulation par l’appelant.

Source : `src/plugins/compaction-provider.ts`, `src/agents/pi-hooks/compaction-safeguard.ts`.

---

## Surfaces visibles par l’utilisateur

Vous pouvez observer la compaction et l’état de session via :

- `/status` (dans n’importe quelle session de chat)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Mode verbeux : `🧹 Auto-compaction complete` + nombre de compactions

---

## Maintenance silencieuse (`NO_REPLY`)

OpenClaw prend en charge des tours « silencieux » pour les tâches d’arrière-plan où l’utilisateur ne doit pas voir de sortie intermédiaire.

Convention :

- L’assistant commence sa sortie avec le jeton silencieux exact `NO_REPLY` /
  `no_reply` pour indiquer « ne pas livrer de réponse à l’utilisateur ».
- OpenClaw le supprime/le masque dans la couche de livraison.
- La suppression exacte du jeton silencieux ne tient pas compte de la casse, donc `NO_REPLY` et
  `no_reply` comptent tous deux lorsque la charge utile complète n’est que le jeton silencieux.
- C’est réservé aux véritables tours d’arrière-plan/sans livraison ; ce n’est pas un raccourci pour
  des demandes utilisateur ordinaires et exploitables.

Depuis `2026.1.10`, OpenClaw supprime aussi le **streaming brouillon/saisie** lorsqu’un
morceau partiel commence par `NO_REPLY`, afin que les opérations silencieuses ne divulguent pas de sortie partielle en milieu de tour.

---

## « Vidage mémoire » avant compaction (implémenté)

Objectif : avant qu’une compaction automatique ne se produise, exécuter un tour agentique silencieux qui écrit un
état durable sur disque (par ex. `memory/YYYY-MM-DD.md` dans l’espace de travail de l’agent) afin que la compaction ne puisse pas
effacer de contexte critique.

OpenClaw utilise l’approche de **vidage avant seuil** :

1. Surveiller l’utilisation du contexte de session.
2. Lorsqu’elle franchit un « seuil souple » (inférieur au seuil de compaction de Pi), exécuter une directive silencieuse
   « écrire la mémoire maintenant » vers l’agent.
3. Utiliser le jeton silencieux exact `NO_REPLY` / `no_reply` pour que l’utilisateur ne
   voie rien.

Configuration (`agents.defaults.compaction.memoryFlush`) :

- `enabled` (par défaut : `true`)
- `softThresholdTokens` (par défaut : `4000`)
- `prompt` (message utilisateur pour le tour de vidage)
- `systemPrompt` (prompt système supplémentaire ajouté au tour de vidage)

Notes :

- Le prompt système/prompt par défaut inclut un indice `NO_REPLY` pour supprimer
  la livraison.
- Le vidage s’exécute une fois par cycle de compaction (suivi dans `sessions.json`).
- Le vidage ne s’exécute que pour les sessions Pi embarquées (les backends CLI l’ignorent).
- Le vidage est ignoré lorsque l’espace de travail de la session est en lecture seule (`workspaceAccess: "ro"` ou `"none"`).
- Voir [Mémoire](/fr/concepts/memory) pour la disposition des fichiers d’espace de travail et les modèles d’écriture.

Pi expose aussi un hook `session_before_compact` dans l’API d’extension, mais la logique de
vidage d’OpenClaw vit aujourd’hui côté Gateway.

---

## Liste de vérification de dépannage

- Mauvaise clé de session ? Commencez par [/concepts/session](/fr/concepts/session) et confirmez la `sessionKey` dans `/status`.
- Incohérence entre stockage et transcription ? Confirmez l’hôte Gateway et le chemin du stockage depuis `openclaw status`.
- Trop de compactions ? Vérifiez :
  - la fenêtre de contexte du modèle (trop petite)
  - les paramètres de compaction (`reserveTokens` trop élevé pour la fenêtre du modèle peut provoquer une compaction plus tôt)
  - le gonflement des résultats d’outil : activez/ajustez l’élagage de session
- Fuite de tours silencieux ? Confirmez que la réponse commence par `NO_REPLY` (jeton exact insensible à la casse) et que vous utilisez une build qui inclut le correctif de suppression du streaming.
