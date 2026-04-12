---
read_when:
    - Szukasz publicznych definicji kanałów wydań
    - Szukasz nazewnictwa wersji i częstotliwości wydań
summary: Publiczne kanały wydań, nazewnictwo wersji i częstotliwość wydań
title: Zasady wydań
x-i18n:
    generated_at: "2026-04-12T23:33:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: dffc1ee5fdbb20bd1bf4b3f817d497fc0d87f70ed6c669d324fea66dc01d0b0b
    source_path: reference/RELEASING.md
    workflow: 15
---

# Zasady wydań

OpenClaw ma trzy publiczne ścieżki wydań:

- stable: tagowane wydania publikowane domyślnie do npm `beta`, albo do npm `latest`, jeśli zostanie to jawnie wskazane
- beta: tagi prerelease publikowane do npm `beta`
- dev: ruchoma głowa `main`

## Nazewnictwo wersji

- Wersja wydania stable: `YYYY.M.D`
  - Tag Git: `vYYYY.M.D`
- Wersja wydania poprawkowego stable: `YYYY.M.D-N`
  - Tag Git: `vYYYY.M.D-N`
- Wersja prerelease beta: `YYYY.M.D-beta.N`
  - Tag Git: `vYYYY.M.D-beta.N`
- Nie dopełniaj miesiąca ani dnia zerami
- `latest` oznacza bieżące promowane stabilne wydanie npm
- `beta` oznacza bieżący docelowy kanał instalacji beta
- Wydania stable i poprawkowe stable są domyślnie publikowane do npm `beta`; operatorzy wydań mogą jawnie wskazać `latest` albo później wypromować zweryfikowane wydanie beta
- Każde wydanie OpenClaw obejmuje jednocześnie pakiet npm i aplikację macOS

## Częstotliwość wydań

- Wydania przechodzą najpierw przez beta
- Stable następuje dopiero po zweryfikowaniu najnowszego beta
- Szczegółowa procedura wydania, zatwierdzenia, poświadczenia i notatki odzyskiwania są
  dostępne tylko dla maintainerów

## Preflight wydania

- Uruchom `pnpm build && pnpm ui:build` przed `pnpm release:check`, aby oczekiwane
  artefakty wydania `dist/*` oraz bundle Control UI istniały dla kroku
  walidacji pakietu
- Uruchom `pnpm release:check` przed każdym tagowanym wydaniem
- Kontrole wydania są teraz uruchamiane w osobnym ręcznym workflow:
  `OpenClaw Release Checks`
- Ten podział jest zamierzony: rzeczywista ścieżka wydania npm ma pozostać krótka,
  deterministyczna i skoncentrowana na artefaktach, podczas gdy wolniejsze testy live pozostają
  w osobnej ścieżce, aby nie opóźniały ani nie blokowały publikacji
- Kontrole wydania muszą być uruchamiane z odwołania workflow `main`, aby logika
  workflow i sekrety pozostały kanoniczne
- Ten workflow akceptuje istniejący tag wydania albo bieżący pełny 40-znakowy SHA commita `main`
- W trybie SHA commita akceptowany jest tylko bieżący HEAD `origin/main`; użyj
  tagu wydania dla starszych commitów wydania
- Walidacyjny preflight `OpenClaw NPM Release` także akceptuje bieżący
  pełny 40-znakowy SHA commita `main` bez wymagania wypchniętego tagu
- Ta ścieżka SHA służy wyłącznie do walidacji i nie może zostać wypromowana do rzeczywistej publikacji
- W trybie SHA workflow syntetyzuje `v<package.json version>` wyłącznie dla kontroli
  metadanych pakietu; rzeczywista publikacja nadal wymaga prawdziwego tagu wydania
- Oba workflow zachowują rzeczywistą ścieżkę publikacji i promocji na runnerach hostowanych przez GitHub, podczas gdy niemutująca ścieżka walidacji może używać większych
  runnerów Blacksmith Linux
- Ten workflow uruchamia
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  z użyciem sekretów workflow `OPENAI_API_KEY` i `ANTHROPIC_API_KEY`
- Preflight wydania npm nie czeka już na osobną ścieżkę kontroli wydań
- Przed zatwierdzeniem uruchom `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (albo odpowiadający tag beta/poprawkowy)
- Po publikacji npm uruchom
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (albo odpowiadającą wersję beta/poprawkową), aby zweryfikować opublikowaną ścieżkę
  instalacji z rejestru w świeżym tymczasowym prefiksie
- Automatyzacja wydań maintainerów używa teraz modelu preflight-then-promote:
  - rzeczywista publikacja npm musi przejść pomyślny npm `preflight_run_id`
  - wydania stable npm domyślnie trafiają do `beta`
  - publikacja stable npm może jawnie wskazać `latest` przez wejście workflow
  - promocja stable npm z `beta` do `latest` nadal jest dostępna jako jawny tryb ręczny w zaufanym workflow `OpenClaw NPM Release`
  - ten tryb promocji nadal wymaga prawidłowego `NPM_TOKEN` w środowisku `npm-release`, ponieważ zarządzanie npm `dist-tag` jest oddzielone od zaufanej publikacji
  - publiczne `macOS Release` służy wyłącznie do walidacji
  - rzeczywista prywatna publikacja na mac musi przejść pomyślne prywatne identyfikatory
    `preflight_run_id` i `validate_run_id`
  - rzeczywiste ścieżki publikacji promują przygotowane artefakty zamiast budować
    je ponownie
- Dla wydań poprawkowych stable takich jak `YYYY.M.D-N` weryfikator po publikacji
  sprawdza także tę samą ścieżkę aktualizacji z tymczasowym prefiksem z `YYYY.M.D` do `YYYY.M.D-N`,
  aby poprawki wydania nie mogły po cichu pozostawić starszych globalnych instalacji na
  bazowym payloadzie stable
- Preflight wydania npm kończy się blokująco, jeśli tarball nie zawiera jednocześnie
  `dist/control-ui/index.html` i niepustego payloadu `dist/control-ui/assets/`,
  abyśmy nie wysłali ponownie pustego dashboardu przeglądarkowego
- Jeśli prace nad wydaniem dotyczyły planowania CI, manifestów czasu rozszerzeń albo
  macierzy testów rozszerzeń, przed zatwierdzeniem zregeneruj i przejrzyj
  wyjścia macierzy workflow `checks-node-extensions` należące do planera z `.github/workflows/ci.yml`,
  aby informacje o wydaniu nie opisywały nieaktualnego układu CI
- Gotowość wydania stable na macOS obejmuje także powierzchnie aktualizatora:
  - wydanie GitHub musi ostatecznie zawierać spakowane `.zip`, `.dmg` i `.dSYM.zip`
  - `appcast.xml` na `main` musi wskazywać nowy stabilny plik zip po publikacji
  - spakowana aplikacja musi zachować niedebugowy bundle id, niepusty URL feedu Sparkle
    oraz `CFBundleVersion` równy lub wyższy od kanonicznego minimalnego poziomu builda Sparkle
    dla tej wersji wydania

## Wejścia workflow NPM

`OpenClaw NPM Release` akceptuje następujące wejścia sterowane przez operatora:

- `tag`: wymagany tag wydania, taki jak `v2026.4.2`, `v2026.4.2-1` albo
  `v2026.4.2-beta.1`; gdy `preflight_only=true`, może to być również bieżący
  pełny 40-znakowy SHA commita `main` dla preflightu wyłącznie walidacyjnego
- `preflight_only`: `true` dla samej walidacji/budowania/pakowania, `false` dla
  rzeczywistej ścieżki publikacji
- `preflight_run_id`: wymagane na rzeczywistej ścieżce publikacji, aby workflow ponownie użył
  przygotowanego tarballa z pomyślnego przebiegu preflight
- `npm_dist_tag`: docelowy tag npm dla ścieżki publikacji; domyślnie `beta`
- `promote_beta_to_latest`: `true`, aby pominąć publikację i przenieść już opublikowane
  stabilne wydanie `beta` na `latest`

`OpenClaw Release Checks` akceptuje następujące wejścia sterowane przez operatora:

- `ref`: istniejący tag wydania albo bieżący pełny 40-znakowy SHA commita `main`
  do walidacji

Zasady:

- Tagi stable i poprawek mogą być publikowane do `beta` albo `latest`
- Tagi prerelease beta mogą być publikowane wyłącznie do `beta`
- Pełny SHA commita jest dozwolony tylko wtedy, gdy `preflight_only=true`
- Tryb SHA commita dla kontroli wydań także wymaga bieżącego HEAD `origin/main`
- Rzeczywista ścieżka publikacji musi używać tego samego `npm_dist_tag`, które było użyte podczas preflightu;
  workflow weryfikuje te metadane, zanim publikacja będzie mogła być kontynuowana
- Tryb promocji musi używać tagu stable albo poprawkowego, `preflight_only=false`,
  pustego `preflight_run_id` oraz `npm_dist_tag=beta`
- Tryb promocji wymaga także prawidłowego `NPM_TOKEN` w środowisku `npm-release`,
  ponieważ `npm dist-tag add` nadal wymaga zwykłego auth npm

## Sekwencja stabilnego wydania npm

Podczas przygotowywania stabilnego wydania npm:

1. Uruchom `OpenClaw NPM Release` z `preflight_only=true`
   - Zanim tag będzie istnieć, możesz użyć bieżącego pełnego SHA commita `main`
     do walidacyjnego dry run workflow preflight
2. Wybierz `npm_dist_tag=beta` dla normalnego przepływu beta-first albo `latest` tylko
   wtedy, gdy celowo chcesz bezpośredniej publikacji stable
3. Uruchom osobno `OpenClaw Release Checks` z tym samym tagiem albo
   pełnym bieżącym SHA `main`, jeśli chcesz pokrycia live prompt cache
   - To jest rozdzielone celowo, aby pokrycie live pozostało dostępne bez
     ponownego sprzęgania długich albo niestabilnych kontroli z workflow publikacji
4. Zapisz pomyślny `preflight_run_id`
5. Uruchom ponownie `OpenClaw NPM Release` z `preflight_only=false`, tym samym
   `tag`, tym samym `npm_dist_tag` i zapisanym `preflight_run_id`
6. Jeśli wydanie trafiło do `beta`, uruchom później `OpenClaw NPM Release` z
   tym samym stabilnym `tag`, `promote_beta_to_latest=true`, `preflight_only=false`,
   pustym `preflight_run_id` oraz `npm_dist_tag=beta`, gdy chcesz przenieść ten
   opublikowany build do `latest`

Tryb promocji nadal wymaga zatwierdzenia środowiska `npm-release` i prawidłowego
`NPM_TOKEN` w tym środowisku.

To pozwala zachować zarówno ścieżkę bezpośredniej publikacji, jak i ścieżkę promocji beta-first
udokumentowane i widoczne dla operatora.

## Publiczne odwołania

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainerzy używają prywatnej dokumentacji wydań w
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
jako właściwego runbooka.
