---
read_when:
    - Vous cherchez les définitions des canaux de publication publics
    - Vous cherchez le nommage des versions et la cadence
summary: Canaux de publication publics, nommage des versions et cadence
title: Politique de publication
x-i18n:
    generated_at: "2026-04-12T23:33:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: dffc1ee5fdbb20bd1bf4b3f817d497fc0d87f70ed6c669d324fea66dc01d0b0b
    source_path: reference/RELEASING.md
    workflow: 15
---

# Politique de publication

OpenClaw a trois canaux de publication publics :

- stable : publications taguées qui publient sur npm `beta` par défaut, ou sur npm `latest` sur demande explicite
- beta : tags de préversion qui publient sur npm `beta`
- dev : la tête mouvante de `main`

## Nommage des versions

- Version de publication stable : `YYYY.M.D`
  - Tag Git : `vYYYY.M.D`
- Version de publication corrective stable : `YYYY.M.D-N`
  - Tag Git : `vYYYY.M.D-N`
- Version de préversion bêta : `YYYY.M.D-beta.N`
  - Tag Git : `vYYYY.M.D-beta.N`
- Ne pas ajouter de zéro initial au mois ni au jour
- `latest` signifie la version npm stable promue actuelle
- `beta` signifie la cible d’installation bêta actuelle
- Les publications stables et correctives stables publient sur npm `beta` par défaut ; les opérateurs de publication peuvent cibler explicitement `latest`, ou promouvoir plus tard une build bêta validée
- Chaque publication OpenClaw livre ensemble le paquet npm et l’app macOS

## Cadence de publication

- Les publications suivent d’abord le canal bêta
- Le stable ne suit qu’une fois la dernière bêta validée
- La procédure détaillée de publication, les approbations, les identifiants et les notes de récupération sont
  réservées aux mainteneurs

## Vérifications préalables à la publication

- Exécutez `pnpm build && pnpm ui:build` avant `pnpm release:check` afin que les
  artefacts de publication attendus `dist/*` et le bundle de la Control UI existent pour l’étape de
  validation du pack
- Exécutez `pnpm release:check` avant chaque publication taguée
- Les vérifications de publication s’exécutent désormais dans un workflow manuel distinct :
  `OpenClaw Release Checks`
- Cette séparation est intentionnelle : garder le vrai chemin de publication npm court,
  déterministe et centré sur les artefacts, tandis que les vérifications live plus lentes restent dans
  leur propre canal afin de ne pas ralentir ou bloquer la publication
- Les vérifications de publication doivent être déclenchées depuis la référence de workflow `main` afin que la
  logique du workflow et les secrets restent canoniques
- Ce workflow accepte soit un tag de publication existant, soit le SHA complet de 40 caractères du commit `main` actuel
- En mode SHA de commit, il n’accepte que le HEAD actuel de `origin/main` ; utilisez un
  tag de publication pour les anciens commits de publication
- La vérification préalable en mode validation uniquement de `OpenClaw NPM Release` accepte également le SHA complet de 40 caractères du commit `main` actuel sans nécessiter de tag poussé
- Ce chemin SHA est uniquement destiné à la validation et ne peut pas être promu en véritable publication
- En mode SHA, le workflow synthétise `v<package.json version>` uniquement pour la vérification des métadonnées du paquet ; la vraie publication nécessite toujours un vrai tag de publication
- Les deux workflows conservent la vraie publication et le chemin de promotion sur des runners hébergés par GitHub, tandis que le chemin de validation non mutatif peut utiliser les runners Linux Blacksmith plus puissants
- Ce workflow exécute
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  en utilisant à la fois les secrets de workflow `OPENAI_API_KEY` et `ANTHROPIC_API_KEY`
- La vérification préalable de publication npm n’attend plus le canal distinct des vérifications de publication
- Exécutez `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (ou le tag bêta/correctif correspondant) avant approbation
- Après la publication npm, exécutez
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (ou la version bêta/corrective correspondante) pour vérifier le chemin d’installation publié dans le registre
  dans un nouveau préfixe temporaire
- L’automatisation de publication des mainteneurs utilise désormais vérification préalable puis promotion :
  - la vraie publication npm doit réussir avec un `preflight_run_id` npm réussi
  - les publications npm stables ciblent `beta` par défaut
  - une publication npm stable peut cibler explicitement `latest` via une entrée de workflow
  - la promotion npm stable de `beta` vers `latest` reste disponible comme mode manuel explicite sur le workflow approuvé `OpenClaw NPM Release`
  - ce mode de promotion nécessite toujours un `NPM_TOKEN` valide dans l’environnement `npm-release` car la gestion de `dist-tag` npm est distincte de la publication approuvée
  - la `macOS Release` publique est uniquement destinée à la validation
  - la vraie publication mac privée doit réussir avec des `preflight_run_id` et `validate_run_id` privés mac réussis
  - les vrais chemins de publication promeuvent des artefacts préparés au lieu de les reconstruire à nouveau
- Pour les publications correctives stables comme `YYYY.M.D-N`, le vérificateur post-publication
  vérifie aussi le même chemin de mise à niveau en préfixe temporaire de `YYYY.M.D` vers `YYYY.M.D-N`
  afin que les corrections de publication ne puissent pas silencieusement laisser d’anciennes installations globales sur la charge utile stable de base
- La vérification préalable de publication npm échoue de manière sûre sauf si le tarball inclut à la fois
  `dist/control-ui/index.html` et une charge utile non vide `dist/control-ui/assets/`
  afin d’éviter d’expédier à nouveau un tableau de bord navigateur vide
- Si le travail de publication a touché la planification CI, les manifestes de synchronisation d’extension, ou
  les matrices de test d’extension, régénérez et examinez les sorties de matrice du workflow
  `checks-node-extensions` gérées par le planificateur depuis `.github/workflows/ci.yml`
  avant approbation afin que les notes de publication ne décrivent pas une disposition CI obsolète
- L’état de préparation de publication stable macOS inclut aussi les surfaces de mise à jour :
  - la publication GitHub doit finalement contenir les `.zip`, `.dmg` et `.dSYM.zip` empaquetés
  - `appcast.xml` sur `main` doit pointer vers le nouveau zip stable après la publication
  - l’app empaquetée doit conserver un bundle id non debug, une URL de flux Sparkle non vide, et un `CFBundleVersion` au moins égal au plancher canonique de build Sparkle pour cette version de publication

## Entrées du workflow NPM

`OpenClaw NPM Release` accepte ces entrées contrôlées par l’opérateur :

- `tag` : tag de publication requis tel que `v2026.4.2`, `v2026.4.2-1`, ou
  `v2026.4.2-beta.1` ; lorsque `preflight_only=true`, cela peut aussi être le SHA complet de 40 caractères du commit `main` actuel pour une vérification préalable de validation uniquement
- `preflight_only` : `true` pour validation/build/paquet uniquement, `false` pour le vrai chemin de publication
- `preflight_run_id` : requis sur le vrai chemin de publication afin que le workflow réutilise le tarball préparé depuis l’exécution de vérification préalable réussie
- `npm_dist_tag` : tag npm cible pour le chemin de publication ; vaut `beta` par défaut
- `promote_beta_to_latest` : `true` pour ignorer la publication et déplacer une build stable `beta` déjà publiée vers `latest`

`OpenClaw Release Checks` accepte ces entrées contrôlées par l’opérateur :

- `ref` : tag de publication existant ou SHA complet de 40 caractères du commit `main`
  actuel à valider

Règles :

- Les tags stables et correctifs peuvent publier soit vers `beta`, soit vers `latest`
- Les tags de préversion bêta ne peuvent publier que vers `beta`
- L’entrée SHA complet de commit n’est autorisée que lorsque `preflight_only=true`
- Le mode SHA de commit des vérifications de publication exige également le HEAD actuel de `origin/main`
- Le vrai chemin de publication doit utiliser le même `npm_dist_tag` que celui utilisé pendant la vérification préalable ;
  le workflow vérifie ces métadonnées avant de poursuivre la publication
- Le mode promotion doit utiliser un tag stable ou correctif, `preflight_only=false`,
  un `preflight_run_id` vide, et `npm_dist_tag=beta`
- Le mode promotion exige aussi un `NPM_TOKEN` valide dans l’environnement `npm-release`
  car `npm dist-tag add` nécessite toujours une authentification npm classique

## Séquence de publication npm stable

Lors de la publication d’une version npm stable :

1. Exécutez `OpenClaw NPM Release` avec `preflight_only=true`
   - Avant qu’un tag n’existe, vous pouvez utiliser le SHA complet du commit `main` actuel pour une simulation de validation uniquement du workflow de vérification préalable
2. Choisissez `npm_dist_tag=beta` pour le flux bêta-d’abord normal, ou `latest` uniquement
   lorsque vous souhaitez intentionnellement une publication stable directe
3. Exécutez `OpenClaw Release Checks` séparément avec le même tag ou le
   SHA complet actuel de `main` lorsque vous souhaitez une couverture live du cache de prompt
   - Cela est séparé volontairement afin que la couverture live reste disponible sans
     recoupler des vérifications longues ou instables au workflow de publication
4. Enregistrez le `preflight_run_id` réussi
5. Exécutez à nouveau `OpenClaw NPM Release` avec `preflight_only=false`, le même
   `tag`, le même `npm_dist_tag`, et le `preflight_run_id` enregistré
6. Si la publication a atterri sur `beta`, exécutez plus tard `OpenClaw NPM Release` avec le
   même `tag` stable, `promote_beta_to_latest=true`, `preflight_only=false`,
   `preflight_run_id` vide, et `npm_dist_tag=beta` lorsque vous souhaitez déplacer cette
   build publiée vers `latest`

Le mode promotion nécessite toujours l’approbation de l’environnement `npm-release` et un
`NPM_TOKEN` valide dans cet environnement.

Cela permet de garder à la fois le chemin de publication directe et le chemin de promotion bêta-d’abord
documentés et visibles pour l’opérateur.

## Références publiques

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Les mainteneurs utilisent la documentation privée de publication dans
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
comme véritable runbook.
