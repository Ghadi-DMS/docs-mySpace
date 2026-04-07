---
read_when:
    - Vous voulez déclencher ou piloter des TaskFlows depuis un système externe
    - Vous configurez le plugin Webhooks groupé
summary: 'Plugin Webhooks : entrée TaskFlow authentifiée pour une automatisation externe de confiance'
title: Plugin Webhooks
x-i18n:
    generated_at: "2026-04-07T06:52:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: a5da12a887752ec6ee853cfdb912db0ae28512a0ffed06fe3828ef2eee15bc9d
    source_path: plugins/webhooks.md
    workflow: 15
---

# Webhooks (plugin)

Le plugin Webhooks ajoute des routes HTTP authentifiées qui lient une
automatisation externe aux TaskFlows OpenClaw.

Utilisez-le lorsque vous voulez qu'un système de confiance tel que Zapier, n8n, un job CI ou un
service interne crée et pilote des TaskFlows gérés sans devoir d'abord écrire un plugin
personnalisé.

## Où il s'exécute

Le plugin Webhooks s'exécute dans le processus Gateway.

Si votre Gateway s'exécute sur une autre machine, installez et configurez le plugin sur
cet hôte Gateway, puis redémarrez la Gateway.

## Configurer les routes

Définissez la configuration sous `plugins.entries.webhooks.config` :

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "Pont TaskFlow Zapier",
            },
          },
        },
      },
    },
  },
}
```

Champs de route :

- `enabled` : facultatif, vaut `true` par défaut
- `path` : facultatif, vaut `/plugins/webhooks/<routeId>` par défaut
- `sessionKey` : session requise qui possède les TaskFlows liés
- `secret` : secret partagé requis ou SecretRef
- `controllerId` : ID de contrôleur facultatif pour les flux gérés créés
- `description` : note opérateur facultative

Entrées `secret` prises en charge :

- Chaîne en clair
- SecretRef avec `source: "env" | "file" | "exec"`

Si une route adossée à un secret ne peut pas résoudre son secret au démarrage, le plugin ignore
cette route et journalise un avertissement au lieu d'exposer un point de terminaison cassé.

## Modèle de sécurité

Chaque route est considérée comme digne de confiance pour agir avec l'autorité TaskFlow de son
`sessionKey` configuré.

Cela signifie que la route peut inspecter et modifier les TaskFlows possédés par cette session, vous
devez donc :

- Utiliser un secret fort et unique par route
- Préférer les références de secret aux secrets en clair intégrés
- Lier les routes à la session la plus étroite adaptée au flux de travail
- Exposer uniquement le chemin webhook spécifique dont vous avez besoin

Le plugin applique :

- Authentification par secret partagé
- Protections sur la taille du corps de requête et les délais d'expiration
- Limitation de débit à fenêtre fixe
- Limitation des requêtes en cours
- Accès aux TaskFlows lié au propriétaire via `api.runtime.taskFlow.bindSession(...)`

## Format de requête

Envoyez des requêtes `POST` avec :

- `Content-Type: application/json`
- `Authorization: Bearer <secret>` ou `x-openclaw-webhook-secret: <secret>`

Exemple :

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"Review inbound queue"}'
```

## Actions prises en charge

Le plugin accepte actuellement les valeurs JSON `action` suivantes :

- `create_flow`
- `get_flow`
- `list_flows`
- `find_latest_flow`
- `resolve_flow`
- `get_task_summary`
- `set_waiting`
- `resume_flow`
- `finish_flow`
- `fail_flow`
- `request_cancel`
- `cancel_flow`
- `run_task`

### `create_flow`

Crée un TaskFlow géré pour la session liée à la route.

Exemple :

```json
{
  "action": "create_flow",
  "goal": "Review inbound queue",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

Crée une tâche enfant gérée dans un TaskFlow géré existant.

Les runtimes autorisés sont :

- `subagent`
- `acp`

Exemple :

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "Inspect the next message batch"
}
```

## Forme de la réponse

Les réponses réussies renvoient :

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

Les requêtes rejetées renvoient :

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow not found.",
  "result": {}
}
```

Le plugin supprime intentionnellement les métadonnées de propriétaire/session des réponses webhook.

## Documentation connexe

- [SDK d'exécution des plugins](/fr/plugins/sdk-runtime)
- [Vue d'ensemble des hooks et webhooks](/fr/automation/hooks)
- [CLI webhooks](/cli/webhooks)
