---
read_when:
    - Szukasz definicji publicznych kanałów wydań
    - Szukasz nazewnictwa wersji i harmonogramu cyklu wydań
summary: Publiczne kanały wydań, nazewnictwo wersji i harmonogram cyklu wydań
title: Zasady wydań
x-i18n:
    generated_at: "2026-04-14T02:08:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: fdc32839447205d74ba7a20a45fbac8e13b199174b442a1e260e3fce056c63da
    source_path: reference/RELEASING.md
    workflow: 15
---

# Zasady wydań

OpenClaw ma trzy publiczne ścieżki wydań:

- stable: oznaczone tagami wydania, które domyślnie publikują do npm `beta`, albo do npm `latest`, gdy zostanie to wyraźnie wskazane
- beta: tagi wydań przedpremierowych publikowane do npm `beta`
- dev: bieżący stan gałęzi `main`

## Nazewnictwo wersji

- Wersja wydania stable: `YYYY.M.D`
  - Tag Git: `vYYYY.M.D`
- Wersja wydania poprawkowego stable: `YYYY.M.D-N`
  - Tag Git: `vYYYY.M.D-N`
- Wersja przedpremierowa beta: `YYYY.M.D-beta.N`
  - Tag Git: `vYYYY.M.D-beta.N`
- Nie dodawaj zer wiodących do miesiąca ani dnia
- `latest` oznacza aktualne promowane stabilne wydanie npm
- `beta` oznacza aktualny docelowy kanał instalacji beta
- Wydania stable i poprawkowe stable domyślnie publikują do npm `beta`; operatorzy wydań mogą jawnie wskazać `latest` albo później promować zweryfikowaną kompilację beta
- Każde wydanie OpenClaw obejmuje jednocześnie pakiet npm i aplikację macOS

## Częstotliwość wydań

- Wydania przechodzą najpierw przez beta
- Stable następuje dopiero po zweryfikowaniu najnowszej wersji beta
- Szczegółowa procedura wydań, akceptacje, poświadczenia i informacje o odzyskiwaniu są
  dostępne wyłącznie dla maintainerów

## Kontrola przed wydaniem

- Uruchom `pnpm build && pnpm ui:build` przed `pnpm release:check`, aby oczekiwane
  artefakty wydania `dist/*` i pakiet interfejsu Control UI istniały na potrzeby
  kroku walidacji pakietu
- Uruchamiaj `pnpm release:check` przed każdym oznaczonym tagiem wydaniu
- Kontrole wydań są teraz uruchamiane w osobnym ręcznym workflow:
  `OpenClaw Release Checks`
- Ten podział jest celowy: pozwala utrzymać właściwą ścieżkę wydania npm krótką,
  deterministyczną i skoncentrowaną na artefaktach, podczas gdy wolniejsze testy live działają
  we własnej ścieżce, aby nie opóźniać ani nie blokować publikacji
- Kontrole wydań muszą być uruchamiane z referencji workflow `main`, aby
  logika workflow i sekrety pozostawały kanoniczne
- Ten workflow akceptuje istniejący tag wydania albo aktualny pełny
  40-znakowy SHA commita `main`
- W trybie SHA commita akceptowany jest tylko aktualny HEAD `origin/main`; dla
  starszych commitów wydań użyj tagu wydania
- Walidacyjny preflight `OpenClaw NPM Release` również akceptuje aktualny pełny
  40-znakowy SHA commita `main` bez wymogu wypchniętego tagu
- Ta ścieżka SHA służy tylko do walidacji i nie może zostać promowana do rzeczywistej publikacji
- W trybie SHA workflow syntetyzuje `v<package.json version>` wyłącznie na potrzeby
  sprawdzenia metadanych pakietu; rzeczywista publikacja nadal wymaga prawdziwego tagu wydania
- Oba workflow pozostawiają właściwą ścieżkę publikacji i promocji na runnerach
  hostowanych przez GitHub, natomiast niemutująca ścieżka walidacji może używać większych
  runnerów Blacksmith Linux
- Ten workflow uruchamia
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  z użyciem sekretów workflow `OPENAI_API_KEY` i `ANTHROPIC_API_KEY`
- Preflight wydania npm nie czeka już na oddzielną ścieżkę kontroli wydań
- Przed akceptacją uruchom `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (albo odpowiadający tag beta/poprawkowy)
- Po publikacji do npm uruchom
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (albo odpowiadającą wersję beta/poprawkową), aby zweryfikować opublikowaną ścieżkę
  instalacji z rejestru w nowym tymczasowym prefiksie
- Automatyzacja wydań maintainerów używa teraz modelu preflight-then-promote:
  - rzeczywista publikacja npm musi przejść pomyślny `preflight_run_id` npm
  - stabilne wydania npm domyślnie trafiają do `beta`
  - stabilna publikacja npm może jawnie wskazać `latest` przez input workflow
  - promocja stabilnego wydania npm z `beta` do `latest` jest nadal dostępna jako jawny tryb ręczny w zaufanym workflow `OpenClaw NPM Release`
  - bezpośrednie stabilne publikacje mogą również uruchamiać jawny tryb synchronizacji dist-tagów, który
    ustawia zarówno `latest`, jak i `beta` na już opublikowaną stabilną wersję
  - te tryby dist-tag nadal wymagają poprawnego `NPM_TOKEN` w środowisku `npm-release`, ponieważ zarządzanie npm `dist-tag` jest oddzielne od trusted publishing
  - publiczny `macOS Release` służy wyłącznie do walidacji
  - rzeczywista prywatna publikacja mac musi przejść pomyślny prywatny mac
    `preflight_run_id` i `validate_run_id`
  - rzeczywiste ścieżki publikacji promują przygotowane artefakty zamiast budować
    je ponownie
- W przypadku wydań poprawkowych stable, takich jak `YYYY.M.D-N`, weryfikator po publikacji
  sprawdza również tę samą ścieżkę aktualizacji w tymczasowym prefiksie z `YYYY.M.D` do `YYYY.M.D-N`,
  aby poprawki wydań nie pozostawiały po cichu starszych instalacji globalnych na
  bazowym stabilnym payloadzie
- Preflight wydania npm kończy się błędem domyślnie, jeśli tarball nie zawiera zarówno
  `dist/control-ui/index.html`, jak i niepustego payloadu `dist/control-ui/assets/`,
  abyśmy nie opublikowali ponownie pustego panelu przeglądarkowego
- Jeśli prace nad wydaniem dotyczyły planowania CI, manifestów czasowych rozszerzeń albo
  macierzy testów rozszerzeń, przed akceptacją zregeneruj i przejrzyj
  wyjścia macierzy workflow `checks-node-extensions` zarządzane przez planner z `.github/workflows/ci.yml`,
  aby informacje o wydaniu nie opisywały nieaktualnego układu CI
- Gotowość stabilnego wydania macOS obejmuje również powierzchnie aktualizatora:
  - wydanie GitHub musi ostatecznie zawierać spakowane `.zip`, `.dmg` i `.dSYM.zip`
  - `appcast.xml` na `main` musi po publikacji wskazywać nowy stabilny plik zip
  - spakowana aplikacja musi zachować niedebugowy bundle id, niepusty adres
    URL feedu Sparkle oraz `CFBundleVersion` równy lub wyższy od kanonicznego
    minimalnego progu kompilacji Sparkle dla tej wersji wydania

## Inputy workflow NPM

`OpenClaw NPM Release` akceptuje następujące inputy kontrolowane przez operatora:

- `tag`: wymagany tag wydania, taki jak `v2026.4.2`, `v2026.4.2-1` albo
  `v2026.4.2-beta.1`; gdy `preflight_only=true`, może to być również aktualny
  pełny 40-znakowy SHA commita `main` dla preflightu służącego wyłącznie do walidacji
- `preflight_only`: `true` dla samej walidacji/budowania/pakowania, `false` dla
  rzeczywistej ścieżki publikacji
- `preflight_run_id`: wymagane w rzeczywistej ścieżce publikacji, aby workflow ponownie użył
  przygotowanego tarballa z pomyślnie zakończonego przebiegu preflight
- `npm_dist_tag`: docelowy tag npm dla ścieżki publikacji; domyślnie `beta`
- `promote_beta_to_latest`: `true`, aby pominąć publikację i przenieść już opublikowaną
  stabilną kompilację `beta` do `latest`
- `sync_stable_dist_tags`: `true`, aby pominąć publikację i skierować zarówno `latest`, jak i
  `beta` na już opublikowaną stabilną wersję

`OpenClaw Release Checks` akceptuje następujące inputy kontrolowane przez operatora:

- `ref`: istniejący tag wydania albo aktualny pełny 40-znakowy SHA commita `main`
  do walidacji

Zasady:

- Tagi stable i poprawkowe mogą publikować zarówno do `beta`, jak i do `latest`
- Tagi przedpremierowe beta mogą publikować wyłącznie do `beta`
- Pełny input SHA commita jest dozwolony tylko wtedy, gdy `preflight_only=true`
- Tryb commit-SHA w kontrolach wydań również wymaga aktualnego HEAD `origin/main`
- Rzeczywista ścieżka publikacji musi używać tego samego `npm_dist_tag`, którego użyto podczas preflightu;
  workflow weryfikuje te metadane, zanim publikacja będzie kontynuowana
- Tryb promocji musi używać tagu stable lub poprawkowego, `preflight_only=false`,
  pustego `preflight_run_id` oraz `npm_dist_tag=beta`
- Tryb synchronizacji dist-tagów musi używać tagu stable lub poprawkowego,
  `preflight_only=false`, pustego `preflight_run_id`, `npm_dist_tag=latest`
  oraz `promote_beta_to_latest=false`
- Tryby promocji i synchronizacji dist-tagów również wymagają poprawnego `NPM_TOKEN`, ponieważ
  `npm dist-tag add` nadal wymaga zwykłego uwierzytelnienia npm; trusted publishing obejmuje
  wyłącznie ścieżkę publikacji pakietu

## Sekwencja stabilnego wydania npm

Podczas przygotowywania stabilnego wydania npm:

1. Uruchom `OpenClaw NPM Release` z `preflight_only=true`
   - Zanim tag zacznie istnieć, możesz użyć aktualnego pełnego SHA commita `main`
     do walidacyjnego dry run workflow preflight
2. Wybierz `npm_dist_tag=beta` dla normalnego przepływu beta-first albo `latest` tylko
   wtedy, gdy celowo chcesz bezpośredniej stabilnej publikacji
3. Uruchom osobno `OpenClaw Release Checks` z tym samym tagiem albo
   pełnym aktualnym SHA `main`, gdy chcesz objąć testami live prompt cache
   - To rozdzielenie jest celowe, aby testy live pozostawały dostępne bez
     ponownego sprzęgania długotrwałych lub niestabilnych kontroli z workflow publikacji
4. Zapisz pomyślny `preflight_run_id`
5. Uruchom ponownie `OpenClaw NPM Release` z `preflight_only=false`, tym samym
   `tag`, tym samym `npm_dist_tag` i zapisanym `preflight_run_id`
6. Jeśli wydanie trafiło do `beta`, uruchom później `OpenClaw NPM Release` z tym
   samym stabilnym `tag`, `promote_beta_to_latest=true`, `preflight_only=false`,
   pustym `preflight_run_id` i `npm_dist_tag=beta`, gdy chcesz przenieść tę
   opublikowaną kompilację do `latest`
7. Jeśli wydanie zostało celowo opublikowane bezpośrednio do `latest` i `beta`
   ma wskazywać tę samą stabilną kompilację, uruchom `OpenClaw NPM Release` z tym samym
   stabilnym `tag`, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`,
   `preflight_only=false`, pustym `preflight_run_id` i `npm_dist_tag=latest`

Tryby promocji i synchronizacji dist-tagów nadal wymagają akceptacji środowiska `npm-release`
oraz poprawnego `NPM_TOKEN` dostępnego dla tego przebiegu workflow.

Dzięki temu zarówno ścieżka publikacji bezpośredniej, jak i ścieżka promocji beta-first pozostają
udokumentowane i widoczne dla operatora.

## Publiczne odwołania

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainerzy korzystają z prywatnej dokumentacji wydań w
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
jako właściwego runbooka.
