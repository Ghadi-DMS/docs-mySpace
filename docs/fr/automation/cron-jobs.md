---
read_when:
    - Planification de tâches en arrière-plan ou de réveils
    - Intégration de déclencheurs externes (webhooks, Gmail) dans OpenClaw
    - Choisir entre heartbeat et cron pour les tâches planifiées
summary: Tâches planifiées, webhooks et déclencheurs Gmail PubSub pour le planificateur du Gateway
title: Tâches planifiées
x-i18n:
    generated_at: "2026-04-11T02:44:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 04d94baa152de17d78515f7d545f099fe4810363ab67e06b465e489737f54665
    source_path: automation/cron-jobs.md
    workflow: 15
---

# Tâches planifiées (Cron)

Cron est le planificateur intégré du Gateway. Il conserve les tâches, réveille l’agent au bon moment et peut renvoyer la sortie vers un canal de chat ou un endpoint webhook.

## Démarrage rapide

```bash
# Ajouter un rappel ponctuel
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the cron docs draft" \
  --wake now \
  --delete-after-run

# Vérifier vos tâches
openclaw cron list

# Voir l’historique d’exécution
openclaw cron runs --id <job-id>
```

## Fonctionnement de cron

- Cron s’exécute **dans le processus Gateway** (pas dans le modèle).
- Les tâches sont conservées dans `~/.openclaw/cron/jobs.json`, donc les redémarrages ne font pas perdre les planifications.
- Toutes les exécutions cron créent des enregistrements de [tâche en arrière-plan](/fr/automation/tasks).
- Les tâches ponctuelles (`--at`) sont supprimées automatiquement après réussite par défaut.
- Les exécutions cron isolées ferment au mieux les onglets/processus de navigateur suivis pour leur session `cron:<jobId>` lorsque l’exécution se termine, afin que l’automatisation de navigateur détachée ne laisse pas de processus orphelins derrière elle.
- Les exécutions cron isolées protègent également contre les réponses d’accusé de réception obsolètes. Si le premier résultat n’est qu’une mise à jour de statut intermédiaire (`on it`, `pulling everything
together` et indications similaires) et qu’aucune exécution de sous-agent descendante n’est encore responsable de la réponse finale, OpenClaw relance une fois pour obtenir le résultat réel avant la livraison.

<a id="maintenance"></a>

La réconciliation des tâches pour cron est gérée par le runtime : une tâche cron active reste active tant que le runtime cron suit encore cette tâche comme étant en cours d’exécution, même si une ancienne ligne de session enfant existe encore.
Une fois que le runtime ne gère plus la tâche et que la fenêtre de grâce de 5 minutes expire, la maintenance peut marquer la tâche comme `lost`.

## Types de planification

| Type    | Option CLI | Description                                                  |
| ------- | ---------- | ------------------------------------------------------------ |
| `at`    | `--at`     | Horodatage ponctuel (ISO 8601 ou relatif, comme `20m`)       |
| `every` | `--every`  | Intervalle fixe                                              |
| `cron`  | `--cron`   | Expression cron à 5 ou 6 champs avec `--tz` facultatif      |

Les horodatages sans fuseau horaire sont traités comme UTC. Ajoutez `--tz America/New_York` pour une planification selon l’heure locale.

Les expressions récurrentes au début de chaque heure sont automatiquement décalées jusqu’à 5 minutes afin de réduire les pics de charge. Utilisez `--exact` pour imposer un timing précis ou `--stagger 30s` pour une fenêtre explicite.

## Styles d’exécution

| Style            | Valeur de `--session` | S’exécute dans         | Idéal pour                       |
| ---------------- | --------------------- | ---------------------- | -------------------------------- |
| Session principale | `main`              | Prochain tour heartbeat | Rappels, événements système      |
| Isolée           | `isolated`            | `cron:<jobId>` dédié   | Rapports, tâches d’arrière-plan  |
| Session actuelle | `current`             | Liée au moment de la création | Travail récurrent sensible au contexte |
| Session personnalisée | `session:custom-id` | Session nommée persistante | Workflows qui s’appuient sur l’historique |

Les tâches en **session principale** mettent en file un événement système et peuvent éventuellement réveiller le heartbeat (`--wake now` ou `--wake next-heartbeat`). Les tâches **isolées** exécutent un tour d’agent dédié avec une nouvelle session. Les **sessions personnalisées** (`session:xxx`) conservent le contexte entre les exécutions, ce qui permet des workflows comme des points quotidiens qui s’appuient sur des résumés précédents.

Pour les tâches isolées, l’arrêt du runtime inclut désormais un nettoyage du navigateur au mieux pour cette session cron. Les échecs de nettoyage sont ignorés afin que le résultat cron réel reste prioritaire.

Lorsque des exécutions cron isolées orchestrent des sous-agents, la livraison privilégie également la sortie finale descendante au texte intermédiaire obsolète du parent. Si des descendants sont encore en cours d’exécution, OpenClaw supprime cette mise à jour partielle du parent au lieu de l’annoncer.

### Options de charge utile pour les tâches isolées

- `--message` : texte du prompt (obligatoire pour les tâches isolées)
- `--model` / `--thinking` : remplacements du modèle et du niveau de réflexion
- `--light-context` : ignorer l’injection des fichiers bootstrap de l’espace de travail
- `--tools exec,read` : limiter les outils que la tâche peut utiliser

`--model` utilise le modèle autorisé sélectionné pour cette tâche. Si le modèle demandé n’est pas autorisé, cron enregistre un avertissement et revient à la sélection du modèle de l’agent/par défaut pour la tâche. Les chaînes de repli configurées s’appliquent toujours, mais un simple remplacement de modèle sans liste de repli explicite par tâche n’ajoute plus le modèle principal de l’agent comme cible de nouvelle tentative supplémentaire cachée.

L’ordre de priorité de sélection du modèle pour les tâches isolées est le suivant :

1. Remplacement de modèle du hook Gmail (lorsque l’exécution vient de Gmail et que ce remplacement est autorisé)
2. `model` de la charge utile par tâche
3. Remplacement du modèle de session cron stocké
4. Sélection du modèle de l’agent/par défaut

Le mode rapide suit également la sélection active résolue. Si la configuration du modèle sélectionné a `params.fastMode`, cron isolé l’utilise par défaut. Un remplacement `fastMode` de session stocké reste prioritaire sur la configuration dans les deux sens.

Si une exécution isolée rencontre un transfert en direct de changement de modèle, cron réessaie avec le provider/modèle remplacé et conserve cette sélection active avant de réessayer. Lorsque le changement transporte aussi un nouveau profil d’authentification, cron conserve également ce remplacement de profil d’authentification. Les nouvelles tentatives sont limitées : après la tentative initiale plus 2 nouvelles tentatives de changement, cron abandonne au lieu de boucler indéfiniment.

## Livraison et sortie

| Mode      | Ce qui se passe                                           |
| --------- | --------------------------------------------------------- |
| `announce` | Livre un résumé au canal cible (par défaut pour les tâches isolées) |
| `webhook` | Envoie par POST la charge utile de l’événement terminé vers une URL |
| `none`    | Interne uniquement, pas de livraison                      |

Utilisez `--announce --channel telegram --to "-1001234567890"` pour une livraison vers un canal. Pour les sujets de forum Telegram, utilisez `-1001234567890:topic:123`. Les cibles Slack/Discord/Mattermost doivent utiliser des préfixes explicites (`channel:<id>`, `user:<id>`).

Pour les tâches isolées gérées par cron, le runner gère le chemin final de livraison. L’agent reçoit pour instruction de renvoyer un résumé en texte brut, et ce résumé est ensuite envoyé via `announce`, `webhook`, ou conservé en interne pour `none`. `--no-deliver` ne redonne pas la livraison à l’agent ; il conserve l’exécution en interne.

Si la tâche d’origine indique explicitement d’envoyer un message à un destinataire externe, l’agent doit indiquer dans sa sortie à qui/où ce message doit aller au lieu d’essayer de l’envoyer directement.

Les notifications d’échec suivent un chemin de destination distinct :

- `cron.failureDestination` définit une valeur par défaut globale pour les notifications d’échec.
- `job.delivery.failureDestination` remplace cela par tâche.
- Si aucun des deux n’est défini et que la tâche livre déjà via `announce`, les notifications d’échec reviennent désormais à cette cible principale `announce`.
- `delivery.failureDestination` est pris en charge uniquement sur les tâches `sessionTarget="isolated"`, sauf si le mode principal de livraison est `webhook`.

## Exemples CLI

Rappel ponctuel (session principale) :

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

Tâche isolée récurrente avec livraison :

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Tâche isolée avec remplacement de modèle et de réflexion :

```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly deep analysis of project progress." \
  --model "opus" \
  --thinking high \
  --announce
```

## Webhooks

Gateway peut exposer des endpoints webhook HTTP pour des déclencheurs externes. Activez-les dans la configuration :

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### Authentification

Chaque requête doit inclure le token du hook via un en-tête :

- `Authorization: Bearer <token>` (recommandé)
- `x-openclaw-token: <token>`

Les tokens dans la chaîne de requête sont rejetés.

### POST /hooks/wake

Mettre en file un événement système pour la session principale :

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"New email received","mode":"now"}'
```

- `text` (obligatoire) : description de l’événement
- `mode` (facultatif) : `now` (par défaut) ou `next-heartbeat`

### POST /hooks/agent

Exécuter un tour d’agent isolé :

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.4-mini"}'
```

Champs : `message` (obligatoire), `name`, `agentId`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`.

### Hooks mappés (POST /hooks/\<name\>)

Les noms de hook personnalisés sont résolus via `hooks.mappings` dans la configuration. Les mappings peuvent transformer des charges utiles arbitraires en actions `wake` ou `agent` avec des modèles ou des transformations par code.

### Sécurité

- Gardez les endpoints de hook derrière loopback, tailnet ou un proxy inverse de confiance.
- Utilisez un token de hook dédié ; ne réutilisez pas les tokens d’authentification du gateway.
- Gardez `hooks.path` sur un sous-chemin dédié ; `/` est rejeté.
- Définissez `hooks.allowedAgentIds` pour limiter le routage explicite via `agentId`.
- Gardez `hooks.allowRequestSessionKey=false` sauf si vous avez besoin de sessions sélectionnées par l’appelant.
- Si vous activez `hooks.allowRequestSessionKey`, définissez également `hooks.allowedSessionKeyPrefixes` pour contraindre les formes de clés de session autorisées.
- Les charges utiles des hooks sont encapsulées avec des limites de sécurité par défaut.

## Intégration Gmail PubSub

Reliez les déclencheurs de boîte de réception Gmail à OpenClaw via Google PubSub.

**Prérequis** : CLI `gcloud`, `gog` (gogcli), hooks OpenClaw activés, Tailscale pour l’endpoint HTTPS public.

### Configuration via assistant (recommandée)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

Cela écrit la configuration `hooks.gmail`, active le preset Gmail et utilise Tailscale Funnel pour l’endpoint push.

### Démarrage automatique du Gateway

Lorsque `hooks.enabled=true` et que `hooks.gmail.account` est défini, le Gateway démarre `gog gmail watch serve` au démarrage et renouvelle automatiquement la surveillance. Définissez `OPENCLAW_SKIP_GMAIL_WATCHER=1` pour désactiver cela.

### Configuration manuelle ponctuelle

1. Sélectionnez le projet GCP qui possède le client OAuth utilisé par `gog` :

```bash
gcloud auth login
gcloud config set project <project-id>
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

2. Créez le topic et accordez l’accès push à Gmail :

```bash
gcloud pubsub topics create gog-gmail-watch
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

3. Démarrez la surveillance :

```bash
gog gmail watch start \
  --account openclaw@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

### Remplacement du modèle Gmail

```json5
{
  hooks: {
    gmail: {
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

## Gestion des tâches

```bash
# Lister toutes les tâches
openclaw cron list

# Modifier une tâche
openclaw cron edit <jobId> --message "Updated prompt" --model "opus"

# Forcer l’exécution immédiate d’une tâche
openclaw cron run <jobId>

# Exécuter seulement si l’échéance est atteinte
openclaw cron run <jobId> --due

# Voir l’historique d’exécution
openclaw cron runs --id <jobId> --limit 50

# Supprimer une tâche
openclaw cron remove <jobId>

# Sélection d’agent (configurations multi-agents)
openclaw cron add --name "Ops sweep" --cron "0 6 * * *" --session isolated --message "Check ops queue" --agent ops
openclaw cron edit <jobId> --clear-agent
```

Remarque sur le remplacement de modèle :

- `openclaw cron add|edit --model ...` modifie le modèle sélectionné de la tâche.
- Si le modèle est autorisé, ce provider/modèle exact est transmis à l’exécution de l’agent isolé.
- S’il n’est pas autorisé, cron émet un avertissement et revient à la sélection du modèle de l’agent/par défaut pour la tâche.
- Les chaînes de repli configurées s’appliquent toujours, mais un simple remplacement `--model` sans liste de repli explicite par tâche ne retombe plus sur le modèle principal de l’agent comme cible de nouvelle tentative supplémentaire silencieuse.

## Configuration

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1,
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
    runLog: { maxBytes: "2mb", keepLines: 2000 },
  },
}
```

Désactiver cron : `cron.enabled: false` ou `OPENCLAW_SKIP_CRON=1`.

**Nouvelle tentative pour une tâche ponctuelle** : les erreurs temporaires (limite de débit, surcharge, réseau, erreur serveur) sont réessayées jusqu’à 3 fois avec un backoff exponentiel. Les erreurs permanentes désactivent immédiatement la tâche.

**Nouvelle tentative pour une tâche récurrente** : backoff exponentiel (de 30 s à 60 min) entre les nouvelles tentatives. Le backoff est réinitialisé après la prochaine exécution réussie.

**Maintenance** : `cron.sessionRetention` (par défaut `24h`) supprime les entrées de session d’exécution isolée. `cron.runLog.maxBytes` / `cron.runLog.keepLines` élaguent automatiquement les fichiers journaux d’exécution.

## Dépannage

### Échelle de commandes

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

### Cron ne se déclenche pas

- Vérifiez `cron.enabled` et la variable d’environnement `OPENCLAW_SKIP_CRON`.
- Confirmez que le Gateway fonctionne en continu.
- Pour les planifications `cron`, vérifiez le fuseau horaire (`--tz`) par rapport au fuseau horaire de l’hôte.
- `reason: not-due` dans la sortie d’exécution signifie que l’exécution manuelle a été vérifiée avec `openclaw cron run <jobId> --due` et que l’échéance de la tâche n’était pas encore atteinte.

### Cron s’est déclenché mais aucune livraison

- Le mode de livraison `none` signifie qu’aucun message externe n’est attendu.
- Une cible de livraison manquante/invalide (`channel`/`to`) signifie que l’envoi sortant a été ignoré.
- Les erreurs d’authentification du canal (`unauthorized`, `Forbidden`) signifient que la livraison a été bloquée par les identifiants.
- Si l’exécution isolée renvoie uniquement le jeton silencieux (`NO_REPLY` / `no_reply`), OpenClaw supprime la livraison sortante directe ainsi que le chemin de résumé mis en file de secours, donc rien n’est renvoyé dans le chat.
- Pour les tâches isolées gérées par cron, ne vous attendez pas à ce que l’agent utilise l’outil de messagerie comme solution de secours. Le runner gère la livraison finale ; `--no-deliver` la conserve en interne au lieu d’autoriser un envoi direct.

### Pièges liés au fuseau horaire

- Cron sans `--tz` utilise le fuseau horaire de l’hôte du gateway.
- Les planifications `at` sans fuseau horaire sont traitées comme UTC.
- `activeHours` de heartbeat utilise la résolution de fuseau horaire configurée.

## Liens associés

- [Automatisation et tâches](/fr/automation) — aperçu de tous les mécanismes d’automatisation
- [Tâches en arrière-plan](/fr/automation/tasks) — journal des tâches pour les exécutions cron
- [Heartbeat](/fr/gateway/heartbeat) — tours périodiques de la session principale
- [Fuseau horaire](/fr/concepts/timezone) — configuration du fuseau horaire
