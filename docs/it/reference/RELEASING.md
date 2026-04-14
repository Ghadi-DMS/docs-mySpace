---
read_when:
    - Cerco le definizioni dei canali di rilascio pubblici
    - Cerco la denominazione delle versioni e la cadenza
summary: Canali di rilascio pubblici, denominazione delle versioni e cadenza
title: Politica di rilascio
x-i18n:
    generated_at: "2026-04-14T02:08:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: fdc32839447205d74ba7a20a45fbac8e13b199174b442a1e260e3fce056c63da
    source_path: reference/RELEASING.md
    workflow: 15
---

# Politica di rilascio

OpenClaw ha tre canali di rilascio pubblici:

- stable: release taggate che pubblicano su npm `beta` per impostazione predefinita, oppure su npm `latest` quando richiesto esplicitamente
- beta: tag di prerelease che pubblicano su npm `beta`
- dev: la testa mobile di `main`

## Denominazione delle versioni

- Versione di rilascio stable: `YYYY.M.D`
  - Tag Git: `vYYYY.M.D`
- Versione di rilascio stable correttiva: `YYYY.M.D-N`
  - Tag Git: `vYYYY.M.D-N`
- Versione di prerelease beta: `YYYY.M.D-beta.N`
  - Tag Git: `vYYYY.M.D-beta.N`
- Non aggiungere zeri iniziali a mese o giorno
- `latest` indica l'attuale release npm stable promossa
- `beta` indica l'attuale destinazione di installazione beta
- Le release stable e stable correttive pubblicano su npm `beta` per impostazione predefinita; gli operatori di rilascio possono scegliere esplicitamente `latest`, oppure promuovere in seguito una build beta verificata
- Ogni release di OpenClaw distribuisce insieme il pacchetto npm e l'app macOS

## Cadenza di rilascio

- Le release passano prima da beta
- Stable segue solo dopo che l'ultima beta è stata convalidata
- La procedura di rilascio dettagliata, le approvazioni, le credenziali e le note di ripristino sono riservate ai maintainer

## Controlli preliminari del rilascio

- Esegui `pnpm build && pnpm ui:build` prima di `pnpm release:check` in modo che gli artifact di rilascio `dist/*` previsti e il bundle della Control UI esistano per il passaggio di convalida del pack
- Esegui `pnpm release:check` prima di ogni release taggata
- I controlli di rilascio ora vengono eseguiti in un workflow manuale separato:
  `OpenClaw Release Checks`
- Questa separazione è intenzionale: mantiene il percorso reale di rilascio npm breve, deterministico e focalizzato sugli artifact, mentre i controlli live più lenti restano nel proprio canale così da non rallentare o bloccare la pubblicazione
- I controlli di rilascio devono essere avviati dal workflow ref di `main` in modo che la logica del workflow e i segreti restino canonici
- Quel workflow accetta un tag di rilascio esistente oppure l'attuale commit SHA completo di 40 caratteri di `main`
- In modalità commit-SHA accetta solo l'attuale HEAD di `origin/main`; usa un tag di rilascio per commit di rilascio precedenti
- Anche il controllo preliminare di sola convalida `OpenClaw NPM Release` accetta l'attuale commit SHA completo di 40 caratteri di `main` senza richiedere un tag già pubblicato
- Quel percorso SHA è solo di convalida e non può essere promosso a una pubblicazione reale
- In modalità SHA il workflow sintetizza `v<package.json version>` solo per il controllo dei metadati del pacchetto; la pubblicazione reale richiede comunque un vero tag di rilascio
- Entrambi i workflow mantengono il percorso reale di pubblicazione e promozione su runner ospitati da GitHub, mentre il percorso di convalida non mutante può usare i runner Linux Blacksmith più grandi
- Quel workflow esegue
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  usando entrambi i secret di workflow `OPENAI_API_KEY` e `ANTHROPIC_API_KEY`
- Il controllo preliminare della release npm non aspetta più il canale separato dei controlli di rilascio
- Esegui `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (o il tag beta/correttivo corrispondente) prima dell'approvazione
- Dopo la pubblicazione su npm, esegui
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (o la versione beta/correttiva corrispondente) per verificare il percorso di installazione del registro pubblicato in un prefisso temporaneo pulito
- L'automazione di rilascio dei maintainer ora usa il modello preflight-then-promote:
  - la reale pubblicazione npm deve superare con esito positivo un `preflight_run_id` npm
  - le release npm stable usano `beta` per impostazione predefinita
  - la pubblicazione npm stable può scegliere esplicitamente `latest` tramite input del workflow
  - la promozione npm stable da `beta` a `latest` è ancora disponibile come modalità manuale esplicita nel workflow attendibile `OpenClaw NPM Release`
  - le pubblicazioni stable dirette possono anche eseguire una modalità esplicita di sincronizzazione dei dist-tag che punta sia `latest` sia `beta` alla versione stable già pubblicata
  - queste modalità dei dist-tag richiedono comunque un `NPM_TOKEN` valido nell'ambiente `npm-release` perché la gestione dei `dist-tag` di npm è separata dalla pubblicazione attendibile
  - `macOS Release` pubblico è solo di convalida
  - la vera pubblicazione privata mac deve superare con esito positivo i valori `preflight_run_id` e `validate_run_id` del preflight mac privato
  - i percorsi di pubblicazione reali promuovono artifact preparati invece di ricostruirli di nuovo
- Per release stable correttive come `YYYY.M.D-N`, il verificatore post-pubblicazione controlla anche lo stesso percorso di aggiornamento con prefisso temporaneo da `YYYY.M.D` a `YYYY.M.D-N`, così le correzioni di rilascio non possono lasciare silenziosamente installazioni globali più vecchie sul payload stable di base
- Il controllo preliminare della release npm fallisce in modalità closed se il tarball non include sia `dist/control-ui/index.html` sia un payload `dist/control-ui/assets/` non vuoto, così evitiamo di distribuire di nuovo una dashboard browser vuota
- Se il lavoro di rilascio ha toccato la pianificazione CI, i manifest di temporizzazione delle estensioni o le matrici di test delle estensioni, rigenera e rivedi gli output della matrice del workflow `checks-node-extensions` gestiti dal planner da `.github/workflows/ci.yml` prima dell'approvazione, così le note di rilascio non descriveranno un layout CI obsoleto
- La prontezza della release stable macOS include anche le superfici dell'updater:
  - la release GitHub deve finire con i file pacchettizzati `.zip`, `.dmg` e `.dSYM.zip`
  - `appcast.xml` su `main` deve puntare al nuovo zip stable dopo la pubblicazione
  - l'app pacchettizzata deve mantenere un bundle id non di debug, un URL del feed Sparkle non vuoto e un `CFBundleVersion` pari o superiore alla soglia canonica di build Sparkle per quella versione di rilascio

## Input del workflow NPM

`OpenClaw NPM Release` accetta questi input controllati dall'operatore:

- `tag`: tag di rilascio obbligatorio come `v2026.4.2`, `v2026.4.2-1`, oppure `v2026.4.2-beta.1`; quando `preflight_only=true`, può anche essere l'attuale commit SHA completo di 40 caratteri di `main` per un preflight solo di convalida
- `preflight_only`: `true` per sola convalida/build/package, `false` per il percorso di pubblicazione reale
- `preflight_run_id`: obbligatorio nel percorso di pubblicazione reale così il workflow riutilizza il tarball preparato dall'esecuzione di preflight riuscita
- `npm_dist_tag`: tag npm di destinazione per il percorso di pubblicazione; il valore predefinito è `beta`
- `promote_beta_to_latest`: `true` per saltare la pubblicazione e spostare su `latest` una build stable `beta` già pubblicata
- `sync_stable_dist_tags`: `true` per saltare la pubblicazione e far puntare sia `latest` sia `beta` a una versione stable già pubblicata

`OpenClaw Release Checks` accetta questi input controllati dall'operatore:

- `ref`: tag di rilascio esistente oppure l'attuale commit SHA completo di 40 caratteri di `main` da convalidare

Regole:

- I tag stable e correttivi possono pubblicare su `beta` o `latest`
- I tag di prerelease beta possono pubblicare solo su `beta`
- L'input SHA del commit completo è consentito solo quando `preflight_only=true`
- La modalità commit-SHA dei controlli di rilascio richiede anche l'attuale HEAD di `origin/main`
- Il percorso di pubblicazione reale deve usare lo stesso `npm_dist_tag` usato durante il preflight; il workflow verifica quei metadati prima di continuare la pubblicazione
- La modalità promozione deve usare un tag stable o correttivo, `preflight_only=false`, un `preflight_run_id` vuoto e `npm_dist_tag=beta`
- La modalità di sincronizzazione dei dist-tag deve usare un tag stable o correttivo, `preflight_only=false`, un `preflight_run_id` vuoto, `npm_dist_tag=latest` e `promote_beta_to_latest=false`
- Le modalità promozione e sincronizzazione dei dist-tag richiedono anche un `NPM_TOKEN` valido perché `npm dist-tag add` richiede ancora l'autenticazione npm normale; la pubblicazione attendibile copre solo il percorso di pubblicazione del pacchetto

## Sequenza di rilascio npm stable

Quando prepari una release npm stable:

1. Esegui `OpenClaw NPM Release` con `preflight_only=true`
   - Prima che esista un tag, puoi usare l'attuale commit SHA completo di `main` per una prova a secco solo di convalida del workflow di preflight
2. Scegli `npm_dist_tag=beta` per il normale flusso beta-first, oppure `latest` solo quando vuoi intenzionalmente una pubblicazione stable diretta
3. Esegui separatamente `OpenClaw Release Checks` con lo stesso tag o con l'attuale commit SHA completo di `main` quando vuoi una copertura live della cache dei prompt
   - Questo è separato di proposito così la copertura live resta disponibile senza riaccoppiare controlli lunghi o instabili al workflow di pubblicazione
4. Salva il `preflight_run_id` riuscito
5. Esegui di nuovo `OpenClaw NPM Release` con `preflight_only=false`, lo stesso `tag`, lo stesso `npm_dist_tag` e il `preflight_run_id` salvato
6. Se la release è arrivata su `beta`, esegui successivamente `OpenClaw NPM Release` con lo stesso `tag` stable, `promote_beta_to_latest=true`, `preflight_only=false`, `preflight_run_id` vuoto e `npm_dist_tag=beta` quando vuoi spostare quella build pubblicata su `latest`
7. Se la release è stata intenzionalmente pubblicata direttamente su `latest` e `beta` deve seguire la stessa build stable, esegui `OpenClaw NPM Release` con lo stesso `tag` stable, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`, `preflight_only=false`, `preflight_run_id` vuoto e `npm_dist_tag=latest`

Le modalità promozione e sincronizzazione dei dist-tag richiedono comunque l'approvazione dell'ambiente `npm-release` e un `NPM_TOKEN` valido accessibile a quell'esecuzione del workflow.

Questo mantiene sia il percorso di pubblicazione diretta sia il percorso di promozione beta-first documentati e visibili agli operatori.

## Riferimenti pubblici

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

I maintainer usano la documentazione privata di rilascio in
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
per la runbook effettiva.
