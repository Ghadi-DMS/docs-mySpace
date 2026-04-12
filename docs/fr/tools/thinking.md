---
read_when:
    - Ajuster l'analyse ou les valeurs par défaut des directives de réflexion, du mode rapide ou du mode verbeux
summary: Syntaxe des directives pour `/think`, `/fast`, `/verbose`, `/trace` et la visibilité du raisonnement
title: Niveaux de réflexion
x-i18n:
    generated_at: "2026-04-12T23:33:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f3b1341281f07ba4e9061e3355845dca234be04cc0d358594312beeb7676e68
    source_path: tools/thinking.md
    workflow: 15
---

# Niveaux de réflexion (directives `/think`)

## Ce que cela fait

- Directive inline dans tout corps entrant : `/t <level>`, `/think:<level>`, ou `/thinking <level>`.
- Niveaux (alias) : `off | minimal | low | medium | high | xhigh | adaptive`
  - minimal → « réfléchir »
  - low → « réfléchir sérieusement »
  - medium → « réfléchir davantage »
  - high → « ultrathink » (budget maximal)
  - xhigh → « ultrathink+ » (modèles GPT-5.2 + Codex uniquement)
  - adaptive → budget de raisonnement adaptatif géré par le fournisseur (pris en charge pour la famille de modèles Anthropic Claude 4.6)
  - `x-high`, `x_high`, `extra-high`, `extra high`, et `extra_high` sont mappés vers `xhigh`.
  - `highest`, `max` sont mappés vers `high`.
- Notes sur les fournisseurs :
  - Les modèles Anthropic Claude 4.6 utilisent `adaptive` par défaut lorsqu'aucun niveau de réflexion explicite n'est défini.
  - MiniMax (`minimax/*`) sur le chemin de streaming compatible Anthropic utilise par défaut `thinking: { type: "disabled" }` sauf si vous définissez explicitement thinking dans les paramètres du modèle ou de la requête. Cela évite les deltas `reasoning_content` divulgués par le format de flux Anthropic non natif de MiniMax.
  - Z.AI (`zai/*`) ne prend en charge qu'un thinking binaire (`on`/`off`). Tout niveau autre que `off` est traité comme `on` (mappé vers `low`).
  - Moonshot (`moonshot/*`) mappe `/think off` vers `thinking: { type: "disabled" }` et tout niveau autre que `off` vers `thinking: { type: "enabled" }`. Lorsque thinking est activé, Moonshot n'accepte que `tool_choice` `auto|none` ; OpenClaw normalise les valeurs incompatibles vers `auto`.

## Ordre de résolution

1. Directive inline sur le message (s'applique uniquement à ce message).
2. Remplacement de session (défini en envoyant un message contenant uniquement une directive).
3. Valeur par défaut par agent (`agents.list[].thinkingDefault` dans la configuration).
4. Valeur par défaut globale (`agents.defaults.thinkingDefault` dans la configuration).
5. Repli : `adaptive` pour les modèles Anthropic Claude 4.6, `low` pour les autres modèles compatibles avec le raisonnement, `off` sinon.

## Définir une valeur par défaut de session

- Envoyez un message qui contient **uniquement** la directive (espaces autorisés), par exemple `/think:medium` ou `/t high`.
- Cela reste appliqué à la session actuelle (par expéditeur par défaut) ; effacé par `/think:off` ou une réinitialisation après inactivité de session.
- Une réponse de confirmation est envoyée (`Thinking level set to high.` / `Thinking disabled.`). Si le niveau est invalide (par exemple `/thinking big`), la commande est rejetée avec une indication et l'état de la session reste inchangé.
- Envoyez `/think` (ou `/think:`) sans argument pour voir le niveau de réflexion actuel.

## Application par agent

- **Pi embarqué** : le niveau résolu est transmis au runtime d'agent Pi en processus.

## Mode rapide (/fast)

- Niveaux : `on|off`.
- Un message contenant uniquement la directive active/désactive un remplacement de session du mode rapide et répond `Fast mode enabled.` / `Fast mode disabled.`.
- Envoyez `/fast` (ou `/fast status`) sans mode pour voir l'état effectif actuel du mode rapide.
- OpenClaw résout le mode rapide dans cet ordre :
  1. `/fast on|off` inline/contenant uniquement la directive
  2. Remplacement de session
  3. Valeur par défaut par agent (`agents.list[].fastModeDefault`)
  4. Configuration par modèle : `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Repli : `off`
- Pour `openai/*`, le mode rapide correspond au traitement prioritaire OpenAI en envoyant `service_tier=priority` sur les requêtes Responses prises en charge.
- Pour `openai-codex/*`, le mode rapide envoie le même indicateur `service_tier=priority` sur Codex Responses. OpenClaw conserve un seul basculement `/fast` partagé entre les deux chemins d'authentification.
- Pour les requêtes directes publiques `anthropic/*`, y compris le trafic authentifié par OAuth envoyé à `api.anthropic.com`, le mode rapide correspond aux niveaux de service Anthropic : `/fast on` définit `service_tier=auto`, `/fast off` définit `service_tier=standard_only`.
- Pour `minimax/*` sur le chemin compatible Anthropic, `/fast on` (ou `params.fastMode: true`) réécrit `MiniMax-M2.7` en `MiniMax-M2.7-highspeed`.
- Les paramètres explicites de modèle Anthropic `serviceTier` / `service_tier` remplacent la valeur par défaut du mode rapide lorsque les deux sont définis. OpenClaw continue à ignorer l'injection du niveau de service Anthropic pour les URL de base proxy non Anthropic.

## Directives verbeuses (/verbose ou /v)

- Niveaux : `on` (minimal) | `full` | `off` (par défaut).
- Un message contenant uniquement la directive active le mode verbeux de session et répond `Verbose logging enabled.` / `Verbose logging disabled.` ; les niveaux invalides renvoient une indication sans modifier l'état.
- `/verbose off` stocke un remplacement explicite de session ; effacez-le via l'interface Sessions en choisissant `inherit`.
- La directive inline n'affecte que ce message ; les valeurs par défaut de session/globales s'appliquent sinon.
- Envoyez `/verbose` (ou `/verbose:`) sans argument pour voir le niveau verbeux actuel.
- Lorsque le mode verbeux est activé, les agents qui émettent des résultats d'outils structurés (Pi, autres agents JSON) renvoient chaque appel d'outil dans son propre message contenant uniquement des métadonnées, préfixé par `<emoji> <tool-name>: <arg>` lorsque disponible (chemin/commande). Ces résumés d'outils sont envoyés dès le démarrage de chaque outil (bulles séparées), et non comme deltas de streaming.
- Les résumés d'échec d'outil restent visibles en mode normal, mais les suffixes détaillés d'erreur brute sont masqués sauf si verbose vaut `on` ou `full`.
- Lorsque verbose vaut `full`, les sorties d'outils sont également transmises après leur achèvement (bulle séparée, tronquée à une longueur sûre). Si vous basculez `/verbose on|full|off` pendant qu'une exécution est en cours, les bulles d'outils suivantes respectent le nouveau réglage.

## Directives de trace du Plugin (/trace)

- Niveaux : `on` | `off` (par défaut).
- Un message contenant uniquement la directive active la sortie de trace du Plugin pour la session et répond `Plugin trace enabled.` / `Plugin trace disabled.`.
- La directive inline n'affecte que ce message ; les valeurs par défaut de session/globales s'appliquent sinon.
- Envoyez `/trace` (ou `/trace:`) sans argument pour voir le niveau de trace actuel.
- `/trace` est plus étroit que `/verbose` : il n'expose que les lignes de trace/de débogage appartenant au Plugin, comme les résumés de débogage Active Memory.
- Les lignes de trace peuvent apparaître dans `/status` et comme message de diagnostic de suivi après la réponse normale de l'assistant.

## Visibilité du raisonnement (/reasoning)

- Niveaux : `on|off|stream`.
- Un message contenant uniquement la directive active/désactive l'affichage des blocs de réflexion dans les réponses.
- Lorsqu'il est activé, le raisonnement est envoyé comme **message séparé** préfixé par `Reasoning:`.
- `stream` (Telegram uniquement) : diffuse le raisonnement dans la bulle de brouillon Telegram pendant la génération de la réponse, puis envoie la réponse finale sans le raisonnement.
- Alias : `/reason`.
- Envoyez `/reasoning` (ou `/reasoning:`) sans argument pour voir le niveau de raisonnement actuel.
- Ordre de résolution : directive inline, puis remplacement de session, puis valeur par défaut par agent (`agents.list[].reasoningDefault`), puis repli (`off`).

## Voir aussi

- La documentation du mode élevé se trouve dans [Elevated mode](/fr/tools/elevated).

## Heartbeats

- Le corps de la sonde Heartbeat est l'invite Heartbeat configurée (par défaut : `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Les directives inline dans un message Heartbeat s'appliquent comme d'habitude (mais évitez de modifier les valeurs par défaut de session à partir des Heartbeats).
- L'envoi Heartbeat transmet par défaut uniquement la charge utile finale. Pour envoyer aussi le message séparé `Reasoning:` (lorsqu'il est disponible), définissez `agents.defaults.heartbeat.includeReasoning: true` ou, par agent, `agents.list[].heartbeat.includeReasoning: true`.

## Interface de chat web

- Le sélecteur de réflexion du chat web reflète le niveau stocké de la session depuis le magasin de sessions/configuration entrante au chargement de la page.
- Choisir un autre niveau écrit immédiatement le remplacement de session via `sessions.patch` ; il n'attend pas l'envoi suivant et ce n'est pas un remplacement ponctuel `thinkingOnce`.
- La première option est toujours `Default (<resolved level>)`, où la valeur par défaut résolue provient du modèle actif de la session : `adaptive` pour Claude 4.6 sur Anthropic/Bedrock, `low` pour les autres modèles compatibles avec le raisonnement, `off` sinon.
- Le sélecteur reste conscient du fournisseur :
  - la plupart des fournisseurs affichent `off | minimal | low | medium | high | adaptive`
  - Z.AI affiche le binaire `off | on`
- `/think:<level>` fonctionne toujours et met à jour le même niveau de session stocké, de sorte que les directives de chat et le sélecteur restent synchronisés.
