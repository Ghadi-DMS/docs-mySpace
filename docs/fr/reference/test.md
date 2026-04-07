---
read_when:
    - Exécuter ou corriger des tests
summary: Comment exécuter les tests localement (vitest) et quand utiliser les modes force/couverture
title: Tests
x-i18n:
    generated_at: "2026-04-07T06:54:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: a25236a707860307cc324f32752ad13a53e448bee9341d8df2e11655561e841c
    source_path: reference/test.md
    workflow: 15
---

# Tests

- Kit de test complet (suites, live, Docker) : [Tests](/fr/help/testing)

- `pnpm test:force` : tue tout processus gateway persistant qui occupe le port de contrôle par défaut, puis exécute la suite Vitest complète avec un port gateway isolé afin que les tests serveur n’entrent pas en collision avec une instance en cours d’exécution. Utilisez-le lorsqu’une exécution précédente de la gateway a laissé le port 18789 occupé.
- `pnpm test:coverage` : exécute la suite unitaire avec la couverture V8 (via `vitest.unit.config.ts`). Les seuils globaux sont de 70 % pour les lignes/branches/fonctions/instructions. La couverture exclut les points d’entrée à forte intégration (câblage CLI, ponts gateway/telegram, serveur statique webchat) afin de garder la cible centrée sur la logique testable unitairement.
- `pnpm test:coverage:changed` : exécute la couverture unitaire uniquement pour les fichiers modifiés depuis `origin/main`.
- `pnpm test:changed` : étend les chemins git modifiés en voies Vitest ciblées lorsque le diff ne touche que des fichiers source/test routables. Les modifications de config/setup reviennent toujours à l’exécution native des projets racine afin que les modifications de câblage soient relancées largement si nécessaire.
- `pnpm test` : achemine les cibles explicites de fichier/répertoire via des voies Vitest ciblées. Les exécutions sans cible exécutent désormais dix configurations de fragments séquentielles (`vitest.full-core-unit-src.config.ts`, `vitest.full-core-unit-security.config.ts`, `vitest.full-core-unit-ui.config.ts`, `vitest.full-core-unit-support.config.ts`, `vitest.full-core-contracts.config.ts`, `vitest.full-core-bundled.config.ts`, `vitest.full-core-runtime.config.ts`, `vitest.full-agentic.config.ts`, `vitest.full-auto-reply.config.ts`, `vitest.full-extensions.config.ts`) au lieu d’un unique énorme processus de projet racine.
- Certains fichiers de test `plugin-sdk` et `commands` passent désormais par des voies dédiées légères qui ne conservent que `test/setup.ts`, en laissant les cas à runtime lourd sur leurs voies existantes.
- Certains fichiers source d’assistance `plugin-sdk` et `commands` mappent aussi `pnpm test:changed` vers des tests frères explicites dans ces voies légères, afin que de petites modifications d’helpers évitent de relancer les suites lourdes adossées au runtime.
- `auto-reply` est désormais aussi réparti en trois configurations dédiées (`core`, `top-level`, `reply`) afin que le harness reply ne domine pas les tests plus légers de statut/token/helper au niveau supérieur.
- La configuration Vitest de base utilise désormais par défaut `pool: "threads"` et `isolate: false`, avec l’exécuteur partagé non isolé activé sur les configs du dépôt.
- `pnpm test:channels` exécute `vitest.channels.config.ts`.
- `pnpm test:extensions` exécute `vitest.extensions.config.ts`.
- `pnpm test:extensions` : exécute les suites extension/plugin.
- `pnpm test:perf:imports` : active le rapport de durée d’import + la ventilation des imports de Vitest, tout en continuant à utiliser le routage par voies ciblées pour les cibles explicites de fichier/répertoire.
- `pnpm test:perf:imports:changed` : même profilage des imports, mais uniquement pour les fichiers modifiés depuis `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` mesure le chemin changed-mode routé par rapport à l’exécution native du projet racine pour le même diff git validé.
- `pnpm test:perf:changed:bench -- --worktree` mesure l’ensemble de modifications du worktree courant sans effectuer de commit au préalable.
- `pnpm test:perf:profile:main` : écrit un profil CPU pour le thread principal de Vitest (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner` : écrit des profils CPU + heap pour l’exécuteur unitaire (`.artifacts/vitest-runner-profile`).
- Intégration gateway : opt-in via `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` ou `pnpm test:gateway`.
- `pnpm test:e2e` : exécute les tests smoke end-to-end de la gateway (WS/HTTP multi-instances/appairage de nœuds). Utilise par défaut `threads` + `isolate: false` avec des workers adaptatifs dans `vitest.e2e.config.ts` ; ajustez avec `OPENCLAW_E2E_WORKERS=<n>` et définissez `OPENCLAW_E2E_VERBOSE=1` pour des logs détaillés.
- `pnpm test:live` : exécute les tests live des fournisseurs (minimax/zai). Nécessite des clés API et `LIVE=1` (ou `*_LIVE_TEST=1` spécifique au fournisseur) pour ne plus les ignorer.
- `pnpm test:docker:openwebui` : démarre OpenClaw + Open WebUI sous Docker, se connecte via Open WebUI, vérifie `/api/models`, puis exécute un vrai chat proxyfié via `/api/chat/completions`. Nécessite une clé de modèle live utilisable (par exemple OpenAI dans `~/.profile`), télécharge une image Open WebUI externe et n’est pas censé être stable en CI comme les suites unitaires/e2e normales.
- `pnpm test:docker:mcp-channels` : démarre un conteneur Gateway initialisé et un second conteneur client qui lance `openclaw mcp serve`, puis vérifie la découverte de conversation routée, les lectures de transcription, les métadonnées de pièces jointes, le comportement de la file d’événements live, le routage d’envoi sortant et les notifications de canal + permissions de style Claude sur le véritable pont stdio. L’assertion de notification Claude lit directement les trames MCP stdio brutes afin que le smoke reflète ce que le pont émet réellement.

## Barrière locale de PR

Pour les vérifications locales de validation/atterrissage de PR, exécutez :

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Si `pnpm test` est instable sur un hôte chargé, relancez-le une fois avant de le traiter comme une régression, puis isolez avec `pnpm test <path/to/test>`. Pour les hôtes à mémoire contrainte, utilisez :

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Benchmark de latence des modèles (clés locales)

Script : [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

Utilisation :

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- Variables d’environnement facultatives : `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`
- Prompt par défaut : « Répondez avec un seul mot : ok. Pas de ponctuation ni de texte supplémentaire. »

Dernière exécution (2025-12-31, 20 exécutions) :

- minimax médiane 1279ms (min 1114, max 2431)
- opus médiane 2454ms (min 1224, max 3170)

## Benchmark de démarrage CLI

Script : [`scripts/bench-cli-startup.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-cli-startup.ts)

Utilisation :

- `pnpm test:startup:bench`
- `pnpm test:startup:bench:smoke`
- `pnpm test:startup:bench:save`
- `pnpm test:startup:bench:update`
- `pnpm test:startup:bench:check`
- `pnpm tsx scripts/bench-cli-startup.ts`
- `pnpm tsx scripts/bench-cli-startup.ts --runs 12`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3`
- `pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all`
- `pnpm tsx scripts/bench-cli-startup.ts --preset all --output .artifacts/cli-startup-bench-all.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case gatewayStatusJson --output .artifacts/cli-startup-bench-smoke.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu`
- `pnpm tsx scripts/bench-cli-startup.ts --json`

Presets :

- `startup` : `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real` : `health`, `status`, `status --json`, `sessions`, `sessions --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all` : les deux presets

La sortie inclut `sampleCount`, avg, p50, p95, min/max, la distribution des codes de sortie/signaux et des résumés de RSS maximal pour chaque commande. Les options facultatives `--cpu-prof-dir` / `--heap-prof-dir` écrivent des profils V8 par exécution afin que la mesure temporelle et la capture de profils utilisent le même harness.

Conventions de sortie sauvegardée :

- `pnpm test:startup:bench:smoke` écrit l’artefact smoke ciblé dans `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` écrit l’artefact de la suite complète dans `.artifacts/cli-startup-bench-all.json` avec `runs=5` et `warmup=1`
- `pnpm test:startup:bench:update` rafraîchit la fixture de référence versionnée dans `test/fixtures/cli-startup-bench.json` avec `runs=5` et `warmup=1`

Fixture versionnée :

- `test/fixtures/cli-startup-bench.json`
- Rafraîchissez-la avec `pnpm test:startup:bench:update`
- Comparez les résultats courants avec la fixture via `pnpm test:startup:bench:check`

## Onboarding E2E (Docker)

Docker est facultatif ; cela n’est nécessaire que pour les tests smoke d’onboarding conteneurisés.

Flux complet de démarrage à froid dans un conteneur Linux propre :

```bash
scripts/e2e/onboard-docker.sh
```

Ce script pilote l’assistant interactif via un pseudo-TTY, vérifie les fichiers de config/espace de travail/session, puis démarre la gateway et exécute `openclaw health`.

## Smoke d’import QR (Docker)

Garantit que `qrcode-terminal` se charge sous les runtimes Node Docker pris en charge (Node 24 par défaut, Node 22 compatible) :

```bash
pnpm test:docker:qr
```
