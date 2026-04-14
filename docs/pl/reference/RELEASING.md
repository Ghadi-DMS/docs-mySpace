---
read_when:
    - Szukasz definicji publicznych kanałów wydań
    - Szukasz nazewnictwa wersji i harmonogramu wydań
summary: Publiczne kanały wydań, nazewnictwo wersji i harmonogram wydawniczy
title: Zasady wydań
x-i18n:
    generated_at: "2026-04-14T09:50:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3eaf9f1786b8c9fd4f5a9c657b623cb69d1a485958e1a9b8f108511839b63587
    source_path: reference/RELEASING.md
    workflow: 15
---

# Zasady wydań

OpenClaw ma trzy publiczne kanały wydań:

- stable: tagowane wydania, które domyślnie publikują do npm `beta`, albo do npm `latest`, gdy zostanie to wyraźnie wskazane
- beta: tagi wydań przedpremierowych, które publikują do npm `beta`
- dev: ruchoma głowa gałęzi `main`

## Nazewnictwo wersji

- Wersja wydania stable: `YYYY.M.D`
  - Tag Git: `vYYYY.M.D`
- Wersja poprawkowego wydania stable: `YYYY.M.D-N`
  - Tag Git: `vYYYY.M.D-N`
- Wersja wydania przedpremierowego beta: `YYYY.M.D-beta.N`
  - Tag Git: `vYYYY.M.D-beta.N`
- Nie dodawaj zer wiodących do miesiąca ani dnia
- `latest` oznacza aktualne promowane stabilne wydanie npm
- `beta` oznacza aktualny cel instalacji beta
- Wydania stable i poprawkowe wydania stable domyślnie publikują do npm `beta`; operatorzy wydań mogą jawnie wskazać `latest` albo później promować zweryfikowaną kompilację beta
- Każde wydanie OpenClaw obejmuje jednocześnie pakiet npm i aplikację macOS

## Harmonogram wydań

- Wydania przechodzą najpierw przez beta
- Stable pojawia się dopiero po zweryfikowaniu najnowszej bety
- Szczegółowa procedura wydania, zatwierdzenia, poświadczenia i uwagi dotyczące odzyskiwania są przeznaczone wyłącznie dla maintainerów

## Kontrola przed wydaniem

- Uruchom `pnpm build && pnpm ui:build` przed `pnpm release:check`, aby oczekiwane artefakty wydania `dist/*` oraz pakiet Control UI istniały na potrzeby kroku walidacji pakietu
- Uruchom `pnpm release:check` przed każdym tagowanym wydaniem
- Kontrole wydania są teraz uruchamiane w osobnym ręcznym workflow:
  `OpenClaw Release Checks`
- Ten podział jest celowy: rzeczywista ścieżka wydania npm ma pozostać krótka, deterministyczna i skoncentrowana na artefaktach, podczas gdy wolniejsze testy live mają pozostać w osobnym kanale, aby nie opóźniały ani nie blokowały publikacji
- Kontrole wydania muszą być uruchamiane z referencji workflow `main`, aby logika workflow i sekrety pozostały kanoniczne
- Ten workflow akceptuje istniejący tag wydania albo bieżący pełny 40-znakowy SHA commita `main`
- W trybie commit-SHA akceptowany jest tylko bieżący HEAD `origin/main`; dla starszych commitów wydania użyj tagu wydania
- Walidacyjny preflight `OpenClaw NPM Release` również akceptuje bieżący pełny 40-znakowy SHA commita `main` bez wymagania wypchniętego tagu
- Ścieżka SHA jest tylko walidacyjna i nie może zostać promowana do rzeczywistej publikacji
- W trybie SHA workflow syntetyzuje `v<package.json version>` wyłącznie na potrzeby sprawdzenia metadanych pakietu; rzeczywista publikacja nadal wymaga prawdziwego tagu wydania
- Oba workflow utrzymują rzeczywistą ścieżkę publikacji i promocji na runnerach hostowanych przez GitHub, podczas gdy niemutująca ścieżka walidacji może używać większych runnerów Blacksmith Linux
- Ten workflow uruchamia
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  z użyciem sekretów workflow `OPENAI_API_KEY` i `ANTHROPIC_API_KEY`
- Preflight wydania npm nie czeka już na osobny kanał kontroli wydania
- Przed zatwierdzeniem uruchom `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (albo odpowiadający tag beta/poprawkowy)
- Po publikacji do npm uruchom
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (albo odpowiadającą wersję beta/poprawkową), aby zweryfikować opublikowaną ścieżkę instalacji z rejestru w świeżym tymczasowym prefiksie
- Automatyzacja wydań maintainerów używa teraz modelu preflight-then-promote:
  - rzeczywista publikacja npm musi przejść pomyślny npm `preflight_run_id`
  - wydania stable do npm domyślnie trafiają do `beta`
  - publikacja stable do npm może jawnie wskazać `latest` przez input workflow
  - promocja stable w npm z `beta` do `latest` nadal jest dostępna jako jawny tryb ręczny w zaufanym workflow `OpenClaw NPM Release`
  - bezpośrednie publikacje stable mogą także uruchamiać jawny tryb synchronizacji dist-tag, który kieruje zarówno `latest`, jak i `beta` na już opublikowaną wersję stable
  - te tryby dist-tag nadal wymagają poprawnego `NPM_TOKEN` w środowisku `npm-release`, ponieważ zarządzanie `npm dist-tag` jest oddzielne od zaufanej publikacji
  - publiczne `macOS Release` służy wyłącznie do walidacji
  - rzeczywista prywatna publikacja mac musi przejść pomyślne prywatne mac
    `preflight_run_id` i `validate_run_id`
  - rzeczywiste ścieżki publikacji promują przygotowane artefakty zamiast ponownie je budować
- W przypadku poprawkowych wydań stable, takich jak `YYYY.M.D-N`, weryfikator po publikacji sprawdza również tę samą ścieżkę aktualizacji w tymczasowym prefiksie z `YYYY.M.D` do `YYYY.M.D-N`, aby poprawki wydań nie mogły po cichu pozostawić starszych globalnych instalacji przy bazowym ładunku stable
- Preflight wydania npm kończy się błędem w trybie fail-closed, chyba że tarball zawiera zarówno `dist/control-ui/index.html`, jak i niepustą zawartość `dist/control-ui/assets/`, abyśmy nie wysłali ponownie pustego panelu przeglądarkowego
- `pnpm test:install:smoke` wymusza również limit `unpackedSize` dla `npm pack` na kandydacie tarballa aktualizacji, dzięki czemu e2e instalatora wychwytuje przypadkowe zwiększenie rozmiaru pakietu przed ścieżką publikacji wydania
- Jeśli prace nad wydaniem dotyczyły planowania CI, manifestów czasu rozszerzeń albo macierzy testów rozszerzeń, przed zatwierdzeniem zregeneruj i przejrzyj należące do planera wyjścia macierzy workflow `checks-node-extensions` z `.github/workflows/ci.yml`, aby informacje o wydaniu nie opisywały nieaktualnego układu CI
- Gotowość stabilnego wydania macOS obejmuje także powierzchnie aktualizatora:
  - wydanie GitHub musi ostatecznie zawierać spakowane `.zip`, `.dmg` i `.dSYM.zip`
  - `appcast.xml` na `main` musi po publikacji wskazywać nowy stabilny plik zip
  - spakowana aplikacja musi zachować niedebugowe bundle id, niepusty adres URL kanału Sparkle oraz `CFBundleVersion` równy lub wyższy od kanonicznego minimalnego poziomu kompilacji Sparkle dla tej wersji wydania

## Inputy workflow NPM

`OpenClaw NPM Release` akceptuje następujące inputy sterowane przez operatora:

- `tag`: wymagany tag wydania, taki jak `v2026.4.2`, `v2026.4.2-1` albo
  `v2026.4.2-beta.1`; gdy `preflight_only=true`, może to być również bieżący
  pełny 40-znakowy SHA commita `main` dla walidacyjnego preflightu
- `preflight_only`: `true` dla samej walidacji/budowania/pakowania, `false` dla rzeczywistej ścieżki publikacji
- `preflight_run_id`: wymagany w rzeczywistej ścieżce publikacji, aby workflow ponownie wykorzystał przygotowany tarball z pomyślnego uruchomienia preflightu
- `npm_dist_tag`: docelowy tag npm dla ścieżki publikacji; domyślnie `beta`
- `promote_beta_to_latest`: `true`, aby pominąć publikację i przenieść już opublikowaną stabilną kompilację `beta` na `latest`
- `sync_stable_dist_tags`: `true`, aby pominąć publikację i skierować zarówno `latest`, jak i `beta` na już opublikowaną stabilną wersję

`OpenClaw Release Checks` akceptuje następujące inputy sterowane przez operatora:

- `ref`: istniejący tag wydania albo bieżący pełny 40-znakowy SHA commita `main` do zweryfikowania

Reguły:

- Tagi stable i poprawkowe mogą publikować zarówno do `beta`, jak i do `latest`
- Tagi wydań przedpremierowych beta mogą publikować wyłącznie do `beta`
- Pełny input SHA commita jest dozwolony tylko wtedy, gdy `preflight_only=true`
- Tryb commit-SHA kontroli wydania wymaga również bieżącego HEAD `origin/main`
- Rzeczywista ścieżka publikacji musi używać tego samego `npm_dist_tag`, który został użyty podczas preflightu; workflow weryfikuje te metadane przed kontynuacją publikacji
- Tryb promocji musi używać tagu stable albo poprawkowego, `preflight_only=false`,
  pustego `preflight_run_id` oraz `npm_dist_tag=beta`
- Tryb synchronizacji dist-tag musi używać tagu stable albo poprawkowego,
  `preflight_only=false`, pustego `preflight_run_id`, `npm_dist_tag=latest`
  oraz `promote_beta_to_latest=false`
- Tryby promocji i synchronizacji dist-tag również wymagają poprawnego `NPM_TOKEN`, ponieważ `npm dist-tag add` nadal wymaga zwykłego uwierzytelnienia npm; zaufana publikacja obejmuje tylko ścieżkę publikacji pakietu

## Sekwencja wydania stable npm

Podczas przygotowywania stabilnego wydania npm:

1. Uruchom `OpenClaw NPM Release` z `preflight_only=true`
   - Zanim tag będzie istniał, możesz użyć bieżącego pełnego SHA commita `main`
     do walidacyjnego próbnego uruchomienia workflow preflight
2. Wybierz `npm_dist_tag=beta` dla zwykłego przepływu beta-first albo `latest` tylko wtedy, gdy celowo chcesz wykonać bezpośrednią publikację stable
3. Uruchom osobno `OpenClaw Release Checks` z tym samym tagiem albo pełnym bieżącym SHA `main`, gdy chcesz uzyskać pokrycie live dla cache promptów
   - To rozdzielenie jest celowe, aby pokrycie live pozostawało dostępne bez ponownego łączenia długotrwałych albo niestabilnych kontroli z workflow publikacji
4. Zapisz pomyślny `preflight_run_id`
5. Uruchom ponownie `OpenClaw NPM Release` z `preflight_only=false`, tym samym
   `tag`, tym samym `npm_dist_tag` i zapisanym `preflight_run_id`
6. Jeśli wydanie trafiło do `beta`, uruchom później `OpenClaw NPM Release` z tym samym stabilnym `tag`, `promote_beta_to_latest=true`, `preflight_only=false`,
   pustym `preflight_run_id` i `npm_dist_tag=beta`, gdy chcesz przenieść tę
   opublikowaną kompilację do `latest`
7. Jeśli wydanie zostało celowo opublikowane bezpośrednio do `latest` i `beta`
   powinno wskazywać tę samą stabilną kompilację, uruchom `OpenClaw NPM Release` z tym samym stabilnym `tag`, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`, `preflight_only=false`, pustym `preflight_run_id` i `npm_dist_tag=latest`

Tryby promocji i synchronizacji dist-tag nadal wymagają zatwierdzenia środowiska `npm-release` oraz poprawnego `NPM_TOKEN` dostępnego dla tego uruchomienia workflow.

Dzięki temu zarówno ścieżka bezpośredniej publikacji, jak i ścieżka promocji beta-first pozostają udokumentowane i widoczne dla operatora.

## Publiczne odniesienia

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainerzy używają prywatnej dokumentacji wydań w
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
jako właściwego runbooka.
