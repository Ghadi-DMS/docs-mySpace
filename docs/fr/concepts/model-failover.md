---
read_when:
    - Diagnostiquer la rotation des profils d'authentification, les périodes de refroidissement ou le comportement de repli des modèles
    - Mettre à jour les règles de basculement pour les profils d'authentification ou les modèles
    - Comprendre comment les remplacements de modèle de session interagissent avec les nouvelles tentatives de repli
summary: Comment OpenClaw fait tourner les profils d'authentification et bascule entre les modèles
title: Basculement de modèle
x-i18n:
    generated_at: "2026-04-07T06:49:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: d88821e229610f236bdab3f798d5e8c173f61a77c01017cc87431126bf465e32
    source_path: concepts/model-failover.md
    workflow: 15
---

# Basculement de modèle

OpenClaw gère les échecs en deux étapes :

1. **Rotation des profils d'authentification** au sein du fournisseur actuel.
2. **Repli de modèle** vers le modèle suivant dans `agents.defaults.model.fallbacks`.

Ce document explique les règles d'exécution et les données qui les sous-tendent.

## Flux d'exécution

Pour une exécution de texte normale, OpenClaw évalue les candidats dans cet ordre :

1. Le modèle de session actuellement sélectionné.
2. Les `agents.defaults.model.fallbacks` configurés dans l'ordre.
3. Le modèle principal configuré à la fin lorsque l'exécution a démarré à partir d'un remplacement.

À l'intérieur de chaque candidat, OpenClaw essaie le basculement des profils d'authentification avant de passer
au candidat de modèle suivant.

Séquence générale :

1. Résoudre le modèle de session actif et la préférence de profil d'authentification.
2. Construire la chaîne de candidats de modèle.
3. Essayer le fournisseur actuel avec les règles de rotation/refroidissement des profils d'authentification.
4. Si ce fournisseur est épuisé avec une erreur justifiant un basculement, passer au
   candidat de modèle suivant.
5. Conserver le remplacement de repli sélectionné avant le début de la nouvelle tentative afin que les autres
   lecteurs de session voient le même fournisseur/modèle que celui que l'exécuteur est sur le point d'utiliser.
6. Si le candidat de repli échoue, restaurer uniquement les champs de remplacement de session
   appartenant au repli lorsqu'ils correspondent encore à ce candidat en échec.
7. Si tous les candidats échouent, lever une `FallbackSummaryError` avec les détails
   par tentative et l'expiration de refroidissement la plus proche lorsqu'elle est connue.

Cette approche est intentionnellement plus restreinte que « sauvegarder et restaurer toute la session ». Le
moteur de réponse ne conserve que les champs de sélection de modèle qu'il possède pour le repli :

- `providerOverride`
- `modelOverride`
- `authProfileOverride`
- `authProfileOverrideSource`
- `authProfileOverrideCompactionCount`

Cela empêche une nouvelle tentative de repli échouée d'écraser des mutations de session plus récentes et non liées
comme des changements manuels via `/model` ou des mises à jour de rotation de session intervenus pendant
l'exécution de la tentative.

## Stockage d'authentification (clés + OAuth)

OpenClaw utilise des **profils d'authentification** à la fois pour les clés API et les jetons OAuth.

- Les secrets sont stockés dans `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (hérité : `~/.openclaw/agent/auth-profiles.json`).
- L'état d'exécution du routage d'authentification est stocké dans `~/.openclaw/agents/<agentId>/agent/auth-state.json`.
- La configuration `auth.profiles` / `auth.order` correspond **uniquement aux métadonnées et au routage** (pas aux secrets).
- Fichier OAuth hérité utilisé uniquement pour l'importation : `~/.openclaw/credentials/oauth.json` (importé dans `auth-profiles.json` lors de la première utilisation).

Plus de détails : [/concepts/oauth](/fr/concepts/oauth)

Types d'identifiants :

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` pour certains fournisseurs)

## ID de profil

Les connexions OAuth créent des profils distincts afin que plusieurs comptes puissent coexister.

- Par défaut : `provider:default` lorsqu'aucun e-mail n'est disponible.
- OAuth avec e-mail : `provider:<email>` (par exemple `google-antigravity:user@gmail.com`).

Les profils se trouvent dans `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` sous `profiles`.

## Ordre de rotation

Lorsqu'un fournisseur a plusieurs profils, OpenClaw choisit un ordre comme suit :

1. **Configuration explicite** : `auth.order[provider]` (si défini).
2. **Profils configurés** : `auth.profiles` filtrés par fournisseur.
3. **Profils stockés** : entrées dans `auth-profiles.json` pour le fournisseur.

Si aucun ordre explicite n'est configuré, OpenClaw utilise un ordre round-robin :

- **Clé primaire :** type de profil (**OAuth avant les clés API**).
- **Clé secondaire :** `usageStats.lastUsed` (le plus ancien d'abord, dans chaque type).
- Les **profils en refroidissement/désactivés** sont déplacés à la fin, triés par expiration la plus proche.

### Affinité de session (favorable au cache)

OpenClaw **épingle le profil d'authentification choisi par session** afin de garder les caches du fournisseur actifs.
Il **n'effectue pas** de rotation à chaque requête. Le profil épinglé est réutilisé jusqu'à ce que :

- la session soit réinitialisée (`/new` / `/reset`)
- une compaction se termine (le compteur de compaction augmente)
- le profil soit en refroidissement/désactivé

La sélection manuelle via `/model …@<profileId>` définit un **remplacement utilisateur** pour cette session
et n'est pas automatiquement rotée jusqu'au démarrage d'une nouvelle session.

Les profils épinglés automatiquement (sélectionnés par le routeur de session) sont traités comme une **préférence** :
ils sont essayés en premier, mais OpenClaw peut passer à un autre profil en cas de limites de débit/délais d'attente.
Les profils épinglés par l'utilisateur restent verrouillés sur ce profil ; s'il échoue et que des replis de modèle
sont configurés, OpenClaw passe au modèle suivant au lieu de changer de profil.

### Pourquoi OAuth peut « sembler perdu »

Si vous avez à la fois un profil OAuth et un profil de clé API pour le même fournisseur, le round-robin peut basculer entre eux d'un message à l'autre à moins qu'ils ne soient épinglés. Pour forcer un seul profil :

- Épinglez avec `auth.order[provider] = ["provider:profileId"]`, ou
- Utilisez un remplacement par session via `/model …` avec un remplacement de profil (lorsqu'il est pris en charge par votre surface UI/chat).

## Périodes de refroidissement

Lorsqu'un profil échoue en raison d'erreurs d'authentification/de limite de débit (ou d'un délai d'attente qui
ressemble à une limite de débit), OpenClaw le marque comme étant en refroidissement et passe au profil suivant.
Cette catégorie de limite de débit est plus large qu'un simple `429` : elle inclut aussi des messages de fournisseur
tels que `Too many concurrent requests`, `ThrottlingException`,
`concurrency limit reached`, `workers_ai ... quota limit exceeded`,
`throttled`, `resource exhausted`, et des limites périodiques de fenêtre d'utilisation telles que
`weekly/monthly limit reached`.
Les erreurs de format/de requête invalide (par exemple les erreurs de validation d'ID d'appel d'outil Cloud Code Assist)
sont traitées comme justifiant un basculement et utilisent les mêmes périodes de refroidissement.
Les erreurs OpenAI-compatibles de raison d'arrêt telles que `Unhandled stop reason: error`,
`stop reason: error`, et `reason: error` sont classées comme des signaux
de délai d'attente/de basculement.
Le texte générique côté fournisseur à portée fournisseur peut également tomber dans cette catégorie de délai d'attente lorsque
la source correspond à un motif transitoire connu. Par exemple, pour Anthropic, le simple
`An unknown error occurred` et les charges JSON `api_error` avec un texte serveur transitoire
tel que `internal server error`, `unknown error, 520`, `upstream error`,
ou `backend error` sont traités comme des délais d'attente justifiant un basculement. Le texte générique spécifique à OpenRouter
tel que le simple `Provider returned error` est également traité comme
un délai d'attente uniquement lorsque le contexte fournisseur est réellement OpenRouter. Le texte générique de repli interne
tel que `LLM request failed with an unknown error.` reste
conservateur et ne déclenche pas de basculement à lui seul.

Les périodes de refroidissement pour limite de débit peuvent aussi être limitées au modèle :

- OpenClaw enregistre `cooldownModel` pour les échecs de limite de débit lorsque l'ID
  du modèle en échec est connu.
- Un modèle frère sur le même fournisseur peut toujours être essayé lorsque le refroidissement est
  limité à un autre modèle.
- Les fenêtres de facturation/désactivation bloquent toujours l'ensemble du profil sur tous les modèles.

Les périodes de refroidissement utilisent un backoff exponentiel :

- 1 minute
- 5 minutes
- 25 minutes
- 1 heure (plafond)

L'état est stocké dans `auth-state.json` sous `usageStats` :

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## Désactivations de facturation

Les échecs de facturation/de crédit (par exemple « crédits insuffisants » / « solde de crédit trop faible ») sont traités comme justifiant un basculement, mais ils ne sont généralement pas transitoires. Au lieu d'une courte période de refroidissement, OpenClaw marque le profil comme **désactivé** (avec un backoff plus long) et passe au profil/fournisseur suivant.

Toutes les réponses ayant une apparence de facturation ne sont pas `402`, et tous les `402` HTTP n'entrent pas
dans cette catégorie. OpenClaw conserve le texte explicite de facturation dans la voie de facturation même lorsqu'un
fournisseur renvoie plutôt `401` ou `403`, mais les correspondances spécifiques au fournisseur restent
limitées au fournisseur qui les possède (par exemple OpenRouter `403 Key limit
exceeded`). Pendant ce temps, les erreurs temporaires `402` de fenêtre d'utilisation et
de limite de dépense d'organisation/espace de travail sont classées comme `rate_limit` lorsque
le message semble réessayable (par exemple `weekly usage limit exhausted`, `daily
limit reached, resets tomorrow`, ou `organization spending limit exceeded`).
Elles restent sur le chemin de courte période de refroidissement/de basculement au lieu du long
chemin de désactivation pour facturation.

L'état est stocké dans `auth-state.json` :

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

Valeurs par défaut :

- Le backoff de facturation commence à **5 heures**, double à chaque échec de facturation et est plafonné à **24 heures**.
- Les compteurs de backoff sont réinitialisés si le profil n'a pas échoué pendant **24 heures** (configurable).
- Les nouvelles tentatives pour surcharge autorisent **1 rotation de profil du même fournisseur** avant le repli de modèle.
- Les nouvelles tentatives pour surcharge utilisent un backoff de **0 ms** par défaut.

## Repli de modèle

Si tous les profils d'un fournisseur échouent, OpenClaw passe au modèle suivant dans
`agents.defaults.model.fallbacks`. Cela s'applique aux échecs d'authentification, aux limites de débit et aux
délais d'attente qui ont épuisé la rotation des profils (les autres erreurs ne font pas avancer le repli).

Les erreurs de surcharge et de limite de débit sont gérées plus agressivement que les périodes de refroidissement de facturation. Par défaut, OpenClaw autorise une nouvelle tentative de profil d'authentification sur le même fournisseur,
puis passe immédiatement au modèle de repli configuré suivant sans attendre.
Les signaux de fournisseur occupé tels que `ModelNotReadyException` tombent dans cette catégorie
de surcharge. Réglez ce comportement avec `auth.cooldowns.overloadedProfileRotations`,
`auth.cooldowns.overloadedBackoffMs`, et
`auth.cooldowns.rateLimitedProfileRotations`.

Lorsqu'une exécution commence avec un remplacement de modèle (hooks ou CLI), les replis se terminent quand même sur
`agents.defaults.model.primary` après avoir essayé les éventuels replis configurés.

### Règles de chaîne de candidats

OpenClaw construit la liste des candidats à partir du `provider/model` actuellement demandé
plus les replis configurés.

Règles :

- Le modèle demandé est toujours en premier.
- Les replis configurés explicites sont dédupliqués mais ne sont pas filtrés par la liste d'autorisation
  de modèles. Ils sont traités comme une intention explicite de l'opérateur.
- Si l'exécution actuelle se trouve déjà sur un repli configuré dans la même famille de fournisseurs,
  OpenClaw continue d'utiliser la chaîne configurée complète.
- Si l'exécution actuelle est sur un fournisseur différent de la configuration et que ce modèle actuel
  ne fait pas déjà partie de la chaîne de repli configurée, OpenClaw n'ajoute pas
  de replis configurés non liés provenant d'un autre fournisseur.
- Lorsque l'exécution a démarré à partir d'un remplacement, le modèle principal configuré est ajouté à
  la fin afin que la chaîne puisse revenir à la valeur par défaut normale une fois les
  candidats précédents épuisés.

### Quelles erreurs font avancer le repli

Le repli de modèle continue sur :

- les échecs d'authentification
- les limites de débit et l'épuisement du refroidissement
- les erreurs de surcharge/fournisseur occupé
- les erreurs de basculement de type délai d'attente
- les désactivations de facturation
- `LiveSessionModelSwitchError`, qui est normalisée en chemin de basculement afin qu'un
  modèle conservé obsolète ne crée pas de boucle de nouvelle tentative externe
- les autres erreurs non reconnues lorsqu'il reste encore des candidats

Le repli de modèle ne continue pas sur :

- les abandons explicites qui ne sont pas de type délai d'attente/basculement
- les erreurs de dépassement de contexte qui doivent rester dans la logique de compaction/nouvelle tentative
  (par exemple `request_too_large`, `INVALID_ARGUMENT: input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `The input is too long for the model`, ou `ollama error: context
length exceeded`)
- une erreur inconnue finale lorsqu'il ne reste plus de candidats

### Comportement de saut de refroidissement vs sondage

Lorsque tous les profils d'authentification d'un fournisseur sont déjà en refroidissement, OpenClaw
ne saute pas automatiquement ce fournisseur pour toujours. Il prend une décision par candidat :

- Les échecs d'authentification persistants sautent immédiatement l'ensemble du fournisseur.
- Les désactivations de facturation sont généralement sautées, mais le candidat principal peut quand même être sondé
  avec limitation afin que la récupération reste possible sans redémarrage.
- Le candidat principal peut être sondé près de l'expiration du refroidissement, avec une limitation par fournisseur.
- Les modèles frères de repli sur le même fournisseur peuvent être tentés malgré le refroidissement lorsque
  l'échec semble transitoire (`rate_limit`, `overloaded`, ou inconnu). Cela est
  particulièrement pertinent lorsqu'une limite de débit est limitée au modèle et qu'un modèle frère peut
  encore se rétablir immédiatement.
- Les sondages de refroidissement transitoire sont limités à un par fournisseur et par exécution de repli afin
  qu'un seul fournisseur ne bloque pas le repli inter-fournisseurs.

## Remplacements de session et basculement de modèle en direct

Les changements de modèle de session sont un état partagé. L'exécuteur actif, la commande `/model`,
la compaction/les mises à jour de session, et la réconciliation de session en direct lisent ou écrivent tous
des parties de la même entrée de session.

Cela signifie que les nouvelles tentatives de repli doivent se coordonner avec le basculement de modèle en direct :

- Seuls les changements de modèle explicites initiés par l'utilisateur marquent un changement en direct en attente. Cela
  inclut `/model`, `session_status(model=...)`, et `sessions.patch`.
- Les changements de modèle initiés par le système tels que la rotation de repli, les remplacements de heartbeat,
  ou la compaction ne marquent jamais à eux seuls un changement en direct en attente.
- Avant le début d'une nouvelle tentative de repli, le moteur de réponse conserve les champs de remplacement
  de repli sélectionnés dans l'entrée de session.
- La réconciliation de session en direct préfère les remplacements de session conservés aux champs de modèle
  d'exécution obsolètes.
- Si la tentative de repli échoue, l'exécuteur restaure uniquement les champs de remplacement
  qu'il a écrits, et seulement s'ils correspondent encore à ce candidat en échec.

Cela évite la course classique :

1. Le modèle principal échoue.
2. Le candidat de repli est choisi en mémoire.
3. Le stockage de session indique encore l'ancien modèle principal.
4. La réconciliation de session en direct lit l'état de session obsolète.
5. La nouvelle tentative revient à l'ancien modèle avant le début de la tentative de repli.

Le remplacement de repli conservé ferme cette fenêtre, et la restauration ciblée
préserve les changements de session manuels ou d'exécution plus récents.

## Observabilité et résumés d'échec

`runWithModelFallback(...)` enregistre les détails par tentative qui alimentent les journaux et
les messages de refroidissement visibles par l'utilisateur :

- fournisseur/modèle tenté
- raison (`rate_limit`, `overloaded`, `billing`, `auth`, `model_not_found`, et
  raisons de basculement similaires)
- statut/code facultatif
- résumé d'erreur lisible par un humain

Lorsque tous les candidats échouent, OpenClaw lève `FallbackSummaryError`. Le moteur de réponse externe
peut l'utiliser pour construire un message plus spécifique tel que « tous les modèles
sont temporairement limités en débit » et inclure l'expiration de refroidissement la plus proche lorsqu'elle est connue.

Ce résumé de refroidissement tient compte du modèle :

- les limites de débit non liées et limitées à d'autres modèles sont ignorées pour la
  chaîne fournisseur/modèle tentée
- si le blocage restant est une limite de débit limitée au modèle correspondante, OpenClaw
  signale la dernière expiration correspondante qui bloque encore ce modèle

## Configuration associée

Voir [Configuration de la passerelle](/fr/gateway/configuration) pour :

- `auth.profiles` / `auth.order`
- `auth.cooldowns.billingBackoffHours` / `auth.cooldowns.billingBackoffHoursByProvider`
- `auth.cooldowns.billingMaxHours` / `auth.cooldowns.failureWindowHours`
- `auth.cooldowns.overloadedProfileRotations` / `auth.cooldowns.overloadedBackoffMs`
- `auth.cooldowns.rateLimitedProfileRotations`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- le routage de `agents.defaults.imageModel`

Voir [Modèles](/fr/concepts/models) pour un aperçu plus large de la sélection de modèle et du repli.
