---
read_when:
    - Vous souhaitez exécuter OpenClaw sur un serveur local inferrs
    - Vous servez Gemma ou un autre modèle via inferrs
    - Vous avez besoin des indicateurs de compatibilité OpenClaw exacts pour inferrs
summary: Exécuter OpenClaw via inferrs (serveur local compatible OpenAI)
title: inferrs
x-i18n:
    generated_at: "2026-04-08T02:17:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: d84f660d49a682d0c0878707eebe1bc1e83dd115850687076ea3938b9f9c86c6
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs) peut servir des modèles locaux derrière une
API `/v1` compatible OpenAI. OpenClaw fonctionne avec `inferrs` via le chemin générique
`openai-completions`.

Pour l’instant, `inferrs` est mieux traité comme un backend OpenAI compatible auto-hébergé
personnalisé, et non comme un plugin fournisseur OpenClaw dédié.

## Démarrage rapide

1. Démarrez `inferrs` avec un modèle.

Exemple :

```bash
inferrs serve gg-hf-gg/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. Vérifiez que le serveur est accessible.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. Ajoutez une entrée explicite de fournisseur OpenClaw et pointez votre modèle par défaut vers celle-ci.

## Exemple complet de configuration

Cet exemple utilise Gemma 4 sur un serveur local `inferrs`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/gg-hf-gg/gemma-4-E2B-it" },
      models: {
        "inferrs/gg-hf-gg/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "gg-hf-gg/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## Pourquoi `requiresStringContent` est important

Certaines routes Chat Completions de `inferrs` n’acceptent que
`messages[].content` sous forme de chaîne, et non des tableaux structurés de parties de contenu.

Si les exécutions OpenClaw échouent avec une erreur comme :

```text
messages[1].content: invalid type: sequence, expected a string
```

définissez :

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw aplatira les parties de contenu composées uniquement de texte en chaînes simples avant d’envoyer
la requête.

## Réserve concernant Gemma et le schéma d’outil

Certaines combinaisons actuelles `inferrs` + Gemma acceptent de petites requêtes directes
`/v1/chat/completions` mais échouent encore sur des tours complets du runtime d’agent OpenClaw.

Si cela arrive, essayez d’abord ceci :

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

Cela désactive la surface de schéma d’outil d’OpenClaw pour le modèle et peut réduire la pression du prompt
sur les backends locaux plus stricts.

Si de petites requêtes directes fonctionnent toujours mais que les tours normaux d’agent OpenClaw continuent de
planter dans `inferrs`, le problème restant vient généralement du comportement amont du modèle/serveur plutôt que de la couche de transport d’OpenClaw.

## Test smoke manuel

Une fois configuré, testez les deux couches :

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"gg-hf-gg/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/gg-hf-gg/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

Si la première commande fonctionne mais que la seconde échoue, utilisez les notes de dépannage
ci-dessous.

## Dépannage

- `curl /v1/models` échoue : `inferrs` n’est pas en cours d’exécution, n’est pas accessible ou n’est
  pas lié à l’hôte/port attendu.
- `messages[].content ... expected a string` : définissez
  `compat.requiresStringContent: true`.
- Les petits appels directs `/v1/chat/completions` réussissent, mais `openclaw infer model run`
  échoue : essayez `compat.supportsTools: false`.
- OpenClaw n’obtient plus d’erreurs de schéma, mais `inferrs` plante encore sur de plus gros
  tours d’agent : traitez cela comme une limitation amont de `inferrs` ou du modèle et réduisez
  la pression du prompt ou changez de backend/modèle local.

## Comportement de type proxy

`inferrs` est traité comme un backend `/v1` compatible OpenAI de type proxy, et non comme
un endpoint OpenAI natif.

- la mise en forme de requête réservée à OpenAI natif ne s’applique pas ici
- pas de `service_tier`, pas de `store` Responses, pas d’indices de cache de prompt, et pas de mise en forme de charge utile de compatibilité reasoning OpenAI
- les en-têtes d’attribution OpenClaw cachés (`originator`, `version`, `User-Agent`)
  ne sont pas injectés sur des `baseUrl` `inferrs` personnalisées

## Voir aussi

- [Modèles locaux](/fr/gateway/local-models)
- [Dépannage de la passerelle](/fr/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Fournisseurs de modèles](/fr/concepts/model-providers)
