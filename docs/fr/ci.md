---
read_when:
    - Vous devez comprendre pourquoi un job CI s’est exécuté ou non
    - Vous déboguez des vérifications GitHub Actions en échec
summary: Graphe des jobs CI, portes de périmètre et équivalents des commandes locales
title: Pipeline CI
x-i18n:
    generated_at: "2026-04-09T06:51:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: d104f2510fadd674d7952aa08ad73e10f685afebea8d7f19adc1d428e2bdc908
    source_path: ci.md
    workflow: 15
---

# Pipeline CI

La CI s’exécute à chaque push vers `main` et sur chaque pull request. Elle utilise un ciblage intelligent pour ignorer les jobs coûteux lorsque seules des zones sans rapport ont changé.

## Vue d’ensemble des jobs

| Job                      | Objectif                                                                                  | Quand il s’exécute                  |
| ------------------------ | ----------------------------------------------------------------------------------------- | ----------------------------------- |
| `preflight`              | Détecter les changements uniquement liés à la documentation, les périmètres modifiés, les extensions modifiées, et construire le manifeste CI | Toujours sur les pushes et PR non brouillons |
| `security-fast`          | Détection de clés privées, audit des workflows via `zizmor`, audit des dépendances de production | Toujours sur les pushes et PR non brouillons |
| `build-artifacts`        | Construire `dist/` et la Control UI une seule fois, puis téléverser des artefacts réutilisables pour les jobs en aval | Changements pertinents pour Node    |
| `checks-fast-core`       | Voies rapides de vérification Linux, comme les contrôles bundled/plugin-contract/protocol | Changements pertinents pour Node    |
| `checks-fast-extensions` | Agréger les voies fragmentées d’extension une fois `checks-fast-extensions-shard` terminé | Changements pertinents pour Node    |
| `extension-fast`         | Tests ciblés uniquement pour les plugins groupés modifiés                                 | Lorsque des changements d’extension sont détectés |
| `check`                  | Porte locale principale dans la CI : `pnpm check` plus `pnpm build:strict-smoke`          | Changements pertinents pour Node    |
| `check-additional`       | Garde-fous d’architecture, de frontières et de cycles d’import, plus le harnais de régression gateway watch | Changements pertinents pour Node    |
| `build-smoke`            | Tests smoke de la CLI construite et test smoke de mémoire au démarrage                    | Changements pertinents pour Node    |
| `checks`                 | Voies Linux Node plus lourdes : tests complets, tests de canaux et compatibilité Node 22 réservée aux pushes | Changements pertinents pour Node    |
| `check-docs`             | Formatage de la documentation, lint et vérifications des liens cassés                     | Documentation modifiée              |
| `skills-python`          | Ruff + pytest pour les Skills basées sur Python                                           | Changements pertinents pour les Skills Python |
| `checks-windows`         | Voies de test spécifiques à Windows                                                       | Changements pertinents pour Windows |
| `macos-node`             | Voie de test TypeScript macOS utilisant les artefacts construits partagés                 | Changements pertinents pour macOS   |
| `macos-swift`            | Lint, build et tests Swift pour l’app macOS                                               | Changements pertinents pour macOS   |
| `android`                | Matrice de build et de tests Android                                                      | Changements pertinents pour Android |

## Ordre fail-fast

Les jobs sont ordonnés pour que les vérifications peu coûteuses échouent avant l’exécution des plus coûteuses :

1. `preflight` décide quelles voies existent réellement. La logique `docs-scope` et `changed-scope` correspond à des étapes à l’intérieur de ce job, pas à des jobs autonomes.
2. `security-fast`, `check`, `check-additional`, `check-docs` et `skills-python` échouent rapidement sans attendre les jobs plus lourds d’artefacts et de matrices de plateforme.
3. `build-artifacts` se chevauche avec les voies Linux rapides afin que les consommateurs en aval puissent démarrer dès que le build partagé est prêt.
4. Les voies plus lourdes de plateforme et d’exécution se répartissent ensuite : `checks-fast-core`, `checks-fast-extensions`, `extension-fast`, `checks`, `checks-windows`, `macos-node`, `macos-swift` et `android`.

La logique de périmètre se trouve dans `scripts/ci-changed-scope.mjs` et est couverte par des tests unitaires dans `src/scripts/ci-changed-scope.test.ts`.
Le workflow distinct `install-smoke` réutilise le même script de périmètre via son propre job `preflight`. Il calcule `run_install_smoke` à partir du signal changed-smoke plus restreint, afin que le smoke Docker/install ne s’exécute que pour les changements pertinents pour l’installation, le packaging et les conteneurs.

Sur les pushes, la matrice `checks` ajoute la voie `compat-node22`, réservée aux pushes. Sur les pull requests, cette voie est ignorée et la matrice reste centrée sur les voies normales de test/canal.

## Exécuteurs

| Exécuteur                        | Jobs                                                                                                |
| -------------------------------- | --------------------------------------------------------------------------------------------------- |
| `blacksmith-16vcpu-ubuntu-2404`  | `preflight`, `security-fast`, `build-artifacts`, vérifications Linux, vérifications de la documentation, Skills Python, `android` |
| `blacksmith-32vcpu-windows-2025` | `checks-windows`                                                                                    |
| `macos-latest`                   | `macos-node`, `macos-swift`                                                                         |

## Équivalents locaux

```bash
pnpm check          # types + lint + format
pnpm build:strict-smoke
pnpm check:import-cycles
pnpm test:gateway:watch-regression
pnpm test           # tests vitest
pnpm test:channels
pnpm check:docs     # format de la documentation + lint + liens cassés
pnpm build          # construit dist/ lorsque les voies CI artifact/build-smoke sont pertinentes
```
