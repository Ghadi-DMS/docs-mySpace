---
read_when:
    - Uruchamianie testów lokalnie lub w CI
    - Dodawanie regresji dla błędów modeli/providerów
    - Debugowanie zachowania gateway + agenta
summary: 'Zestaw testowy: pakiety unit/e2e/live, uruchamianie w Dockerze i zakres poszczególnych testów'
title: Testowanie
x-i18n:
    generated_at: "2026-04-09T01:31:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 01117f41d8b171a4f1da11ed78486ee700e70ae70af54eb6060c57baf64ab21b
    source_path: help/testing.md
    workflow: 15
---

# Testowanie

OpenClaw ma trzy pakiety Vitest (unit/integration, e2e, live) oraz niewielki zestaw uruchomień w Dockerze.

Ten dokument jest przewodnikiem „jak testujemy”:

- Co obejmuje każdy pakiet testów (i czego celowo _nie_ obejmuje)
- Jakie polecenia uruchamiać w typowych przepływach pracy (lokalnie, przed push, debugowanie)
- Jak testy live wykrywają poświadczenia oraz wybierają modele/providery
- Jak dodawać regresje dla rzeczywistych problemów modeli/providerów

## Szybki start

Na co dzień:

- Pełna bramka (oczekiwana przed push): `pnpm build && pnpm check && pnpm test`
- Szybsze lokalne uruchomienie pełnego pakietu na wydajnej maszynie: `pnpm test:max`
- Bezpośrednia pętla watch Vitest: `pnpm test:watch`
- Bezpośrednie wskazywanie pliku obsługuje teraz również ścieżki rozszerzeń/kanałów: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Witryna QA oparta na Dockerze: `pnpm qa:lab:up`

Gdy dotykasz testów lub chcesz mieć większą pewność:

- Bramka pokrycia: `pnpm test:coverage`
- Pakiet E2E: `pnpm test:e2e`

Podczas debugowania rzeczywistych providerów/modeli (wymaga prawdziwych poświadczeń):

- Pakiet live (modele + sondy narzędzi/obrazów gateway): `pnpm test:live`
- Ciche uruchomienie jednego pliku live: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Wskazówka: gdy potrzebujesz tylko jednego nieudanego przypadku, zawężaj testy live za pomocą zmiennych środowiskowych allowlist opisanych poniżej.

## Pakiety testów (co uruchamia się gdzie)

Najlepiej myśleć o pakietach jako o „rosnącym realizmie” (i rosnącej niestabilności/koszcie):

### Unit / integration (domyślnie)

- Polecenie: `pnpm test`
- Konfiguracja: dziesięć sekwencyjnych uruchomień shardów (`vitest.full-*.config.ts`) na istniejących zakresowanych projektach Vitest
- Pliki: podstawowe inwentarze unit w `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` oraz dozwolone testy node w `ui` objęte przez `vitest.unit.config.ts`
- Zakres:
  - Czyste testy unit
  - Testy integracyjne in-process (auth gateway, routing, narzędzia, parsowanie, konfiguracja)
  - Deterministyczne regresje dla znanych błędów
- Oczekiwania:
  - Uruchamiane w CI
  - Nie wymagają prawdziwych kluczy
  - Powinny być szybkie i stabilne
- Uwaga o projektach:
  - Niezawężone `pnpm test` uruchamia teraz jedenaście mniejszych konfiguracji shardów (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) zamiast jednego ogromnego natywnego procesu root-project. To zmniejsza szczytowe RSS na obciążonych maszynach i zapobiega temu, by prace `auto-reply`/rozszerzeń zagłodziły niezwiązane pakiety.
  - `pnpm test --watch` nadal używa natywnego grafu projektów root `vitest.config.ts`, ponieważ pętla watch z wieloma shardami nie jest praktyczna.
  - `pnpm test`, `pnpm test:watch` i `pnpm test:perf:imports` najpierw kierują jawne cele plików/katalogów do zakresowanych ścieżek, więc `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` unika kosztu uruchamiania pełnego projektu root.
  - `pnpm test:changed` rozwija zmienione ścieżki git do tych samych zakresowanych ścieżek, gdy diff dotyka tylko routowalnych plików źródłowych/testowych; edycje konfiguracji/ustawień nadal wracają do szerokiego ponownego uruchomienia root-project.
  - Wybrane testy `plugin-sdk` i `commands` są również kierowane przez dedykowane lekkie ścieżki, które pomijają `test/setup-openclaw-runtime.ts`; pliki stanowe / ciężkie runtime pozostają na istniejących ścieżkach.
  - Wybrane pliki pomocnicze źródeł `plugin-sdk` i `commands` również mapują uruchomienia trybu changed na jawne testy sąsiednie w tych lekkich ścieżkach, tak aby edycje helperów nie powodowały ponownego uruchamiania całego ciężkiego pakietu dla tego katalogu.
  - `auto-reply` ma teraz trzy dedykowane koszyki: helpery główne top-level, testy integracyjne top-level `reply.*` oraz poddrzewo `src/auto-reply/reply/**`. Dzięki temu najcięższa praca harnessu reply nie trafia do tanich testów status/chunk/token.
- Uwaga o embedded runner:
  - Gdy zmieniasz wejścia wykrywania message-tool lub kontekst runtime compaction,
    zachowaj oba poziomy pokrycia.
  - Dodawaj ukierunkowane regresje helperów dla czystych granic routingu/normalizacji.
  - Utrzymuj też w dobrej kondycji pakiety integracyjne embedded runner:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` oraz
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Te pakiety sprawdzają, że zakresowane identyfikatory i zachowanie compaction nadal przepływają
    przez rzeczywiste ścieżki `run.ts` / `compact.ts`; testy samych helperów nie są
    wystarczającym zamiennikiem dla tych ścieżek integracyjnych.
- Uwaga o puli:
  - Bazowa konfiguracja Vitest domyślnie używa teraz `threads`.
  - Współdzielona konfiguracja Vitest ustawia też `isolate: false` i używa nieizolowanego runnera w projektach root, konfiguracjach e2e i live.
  - Główna ścieżka UI zachowuje ustawienia `jsdom` i optymalizator, ale teraz również działa na współdzielonym nieizolowanym runnerze.
  - Każdy shard `pnpm test` dziedziczy te same domyślne ustawienia `threads` + `isolate: false` ze współdzielonej konfiguracji Vitest.
  - Współdzielony launcher `scripts/run-vitest.mjs` domyślnie dodaje teraz także `--no-maglev` dla podrzędnych procesów Node Vitest, aby ograniczyć churn kompilacji V8 podczas dużych lokalnych uruchomień. Ustaw `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, jeśli chcesz porównać zachowanie ze standardowym V8.
- Uwaga o szybkiej iteracji lokalnej:
  - `pnpm test:changed` kieruje przez zakresowane ścieżki, gdy zmienione ścieżki da się jednoznacznie przypisać do mniejszego pakietu.
  - `pnpm test:max` i `pnpm test:changed:max` zachowują to samo routowanie, tylko z wyższym limitem workerów.
  - Lokalna automatyczna skala workerów jest teraz celowo konserwatywna i również wycofuje się, gdy średnie obciążenie hosta jest już wysokie, więc wiele równoległych uruchomień Vitest domyślnie powoduje mniej szkód.
  - Bazowa konfiguracja Vitest oznacza projekty/pliki konfiguracyjne jako `forceRerunTriggers`, aby ponowne uruchomienia w trybie changed pozostawały poprawne po zmianach w okablowaniu testów.
  - Konfiguracja utrzymuje włączone `OPENCLAW_VITEST_FS_MODULE_CACHE` na obsługiwanych hostach; ustaw `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, jeśli chcesz mieć jedną jawną lokalizację cache do bezpośredniego profilowania.
- Uwaga o debugowaniu wydajności:
  - `pnpm test:perf:imports` włącza raportowanie czasu importu Vitest oraz wyjście z rozbiciem importów.
  - `pnpm test:perf:imports:changed` zawęża ten sam widok profilowania do plików zmienionych od `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` porównuje kierowane `test:changed` z natywną ścieżką root-project dla tego zatwierdzonego diffu i wypisuje wall time oraz macOS max RSS.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkuje bieżące brudne drzewo, kierując listę zmienionych plików przez `scripts/test-projects.mjs` i główną konfigurację Vitest.
  - `pnpm test:perf:profile:main` zapisuje profil CPU głównego wątku dla kosztów uruchamiania i transformacji Vitest/Vite.
  - `pnpm test:perf:profile:runner` zapisuje profile CPU+heap runnera dla pakietu unit z wyłączoną równoległością plików.

### E2E (gateway smoke)

- Polecenie: `pnpm test:e2e`
- Konfiguracja: `vitest.e2e.config.ts`
- Pliki: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Domyślne ustawienia runtime:
  - Używa Vitest `threads` z `isolate: false`, zgodnie z resztą repozytorium.
  - Używa workerów adaptacyjnych (CI: do 2, lokalnie: domyślnie 1).
  - Domyślnie działa w trybie cichym, aby zmniejszyć narzut I/O konsoli.
- Przydatne nadpisania:
  - `OPENCLAW_E2E_WORKERS=<n>` aby wymusić liczbę workerów (maksymalnie 16).
  - `OPENCLAW_E2E_VERBOSE=1` aby ponownie włączyć szczegółowe wyjście konsoli.
- Zakres:
  - Wieloinstancyjne zachowanie gateway end-to-end
  - Powierzchnie WebSocket/HTTP, parowanie węzłów i cięższy networking
- Oczekiwania:
  - Uruchamiane w CI (gdy są włączone w pipeline)
  - Nie wymagają prawdziwych kluczy
  - Mają więcej ruchomych części niż testy unit (mogą być wolniejsze)

### E2E: smoke backendu OpenShell

- Polecenie: `pnpm test:e2e:openshell`
- Plik: `test/openshell-sandbox.e2e.test.ts`
- Zakres:
  - Uruchamia izolowany gateway OpenShell na hoście za pomocą Dockera
  - Tworzy sandbox z tymczasowego lokalnego Dockerfile
  - Testuje backend OpenShell w OpenClaw przez rzeczywiste `sandbox ssh-config` + wykonanie SSH
  - Weryfikuje zdalne kanoniczne zachowanie systemu plików przez most fs sandboxa
- Oczekiwania:
  - Tylko opt-in; nie jest częścią domyślnego uruchomienia `pnpm test:e2e`
  - Wymaga lokalnego CLI `openshell` oraz działającego demona Dockera
  - Używa izolowanych `HOME` / `XDG_CONFIG_HOME`, a następnie niszczy testowy gateway i sandbox
- Przydatne nadpisania:
  - `OPENCLAW_E2E_OPENSHELL=1` aby włączyć test podczas ręcznego uruchamiania szerszego pakietu e2e
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` aby wskazać niestandardowy binarny plik CLI lub skrypt wrapper

### Live (prawdziwi providerzy + prawdziwe modele)

- Polecenie: `pnpm test:live`
- Konfiguracja: `vitest.live.config.ts`
- Pliki: `src/**/*.live.test.ts`
- Domyślnie: **włączone** przez `pnpm test:live` (ustawia `OPENCLAW_LIVE_TEST=1`)
- Zakres:
  - „Czy ten provider/model rzeczywiście działa _dzisiaj_ z prawdziwymi poświadczeniami?”
  - Wychwytywanie zmian formatów providerów, specyfiki wywoływania narzędzi, problemów z auth i zachowania limitów szybkości
- Oczekiwania:
  - Z założenia niestabilne w CI (prawdziwe sieci, rzeczywiste polityki providerów, limity, awarie)
  - Kosztują pieniądze / zużywają limity
  - Lepiej uruchamiać zawężone podzbiory niż „wszystko”
- Uruchomienia live pobierają `~/.profile`, aby uzupełnić brakujące klucze API.
- Domyślnie uruchomienia live nadal izolują `HOME` i kopiują materiały konfiguracyjne/auth do tymczasowego katalogu domowego testów, aby fixtury unit nie mogły modyfikować twojego prawdziwego `~/.openclaw`.
- Ustaw `OPENCLAW_LIVE_USE_REAL_HOME=1` tylko wtedy, gdy celowo chcesz, aby testy live używały twojego prawdziwego katalogu domowego.
- `pnpm test:live` działa teraz domyślnie w cichszym trybie: zachowuje wyjście postępu `[live] ...`, ale ukrywa dodatkowy komunikat o `~/.profile` i wycisza logi bootstrap gateway / hałas Bonjour. Ustaw `OPENCLAW_LIVE_TEST_QUIET=0`, jeśli chcesz przywrócić pełne logi uruchomieniowe.
- Rotacja kluczy API (specyficzna dla providera): ustaw `*_API_KEYS` w formacie oddzielanym przecinkami/średnikami lub `*_API_KEY_1`, `*_API_KEY_2` (na przykład `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) albo nadpisanie per-live przez `OPENCLAW_LIVE_*_KEY`; testy ponawiają próby po odpowiedziach rate limit.
- Wyjście postępu/heartbeat:
  - Pakiety live emitują teraz linie postępu na stderr, więc długie wywołania providerów są widocznie aktywne nawet wtedy, gdy przechwytywanie konsoli Vitest jest ciche.
  - `vitest.live.config.ts` wyłącza przechwytywanie konsoli Vitest, aby linie postępu provider/gateway były natychmiast streamowane podczas uruchomień live.
  - Czas heartbeat dla bezpośrednich modeli regulujesz przez `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Czas heartbeat dla gateway/probe regulujesz przez `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Który pakiet powinienem uruchomić?

Skorzystaj z tej tabeli decyzyjnej:

- Edycja logiki/testów: uruchom `pnpm test` (oraz `pnpm test:coverage`, jeśli zmieniłeś dużo)
- Zmiany w sieci gateway / protokole WS / parowaniu: dodaj `pnpm test:e2e`
- Debugowanie „mój bot nie działa” / błędów specyficznych dla providera / wywoływania narzędzi: uruchom zawężone `pnpm test:live`

## Live: przegląd możliwości węzła Android

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skrypt: `pnpm android:test:integration`
- Cel: wywołać **każde polecenie aktualnie ogłaszane** przez podłączony węzeł Android i potwierdzić zachowanie kontraktu poleceń.
- Zakres:
  - Ustawienia wstępne/ręczne (pakiet nie instaluje, nie uruchamia ani nie paruje aplikacji).
  - Walidacja `node.invoke` gateway polecenie po poleceniu dla wybranego węzła Android.
- Wymagane przygotowanie:
  - Aplikacja Android już połączona i sparowana z gateway.
  - Aplikacja utrzymywana na pierwszym planie.
  - Uprawnienia/zgody na przechwytywanie przyznane dla możliwości, które mają przejść.
- Opcjonalne nadpisania celu:
  - `OPENCLAW_ANDROID_NODE_ID` lub `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Pełne szczegóły konfiguracji Android: [Android App](/pl/platforms/android)

## Live: smoke modeli (klucze profili)

Testy live są podzielone na dwie warstwy, aby łatwiej izolować awarie:

- „Direct model” mówi nam, czy provider/model w ogóle potrafi odpowiedzieć przy danym kluczu.
- „Gateway smoke” mówi nam, czy działa pełny pipeline gateway+agent dla tego modelu (sesje, historia, narzędzia, polityka sandboxa itp.).

### Warstwa 1: Bezpośrednie completion modelu (bez gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Cel:
  - Wyliczyć wykryte modele
  - Użyć `getApiKeyForModel` do wybrania modeli, do których masz poświadczenia
  - Wykonać małe completion dla każdego modelu (oraz ukierunkowane regresje tam, gdzie potrzeba)
- Jak włączyć:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli wywołujesz Vitest bezpośrednio)
- Ustaw `OPENCLAW_LIVE_MODELS=modern` (lub `all`, alias dla modern), aby faktycznie uruchomić ten pakiet; w przeciwnym razie zostanie pominięty, aby `pnpm test:live` pozostawało skupione na gateway smoke
- Jak wybierać modele:
  - `OPENCLAW_LIVE_MODELS=modern` aby uruchomić nowoczesną allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` jest aliasem dla nowoczesnej allowlisty
  - albo `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlista rozdzielana przecinkami)
  - Przeglądy modern/all domyślnie używają dobranego limitu o wysokim sygnale; ustaw `OPENCLAW_LIVE_MAX_MODELS=0` dla wyczerpującego przeglądu modern lub liczbę dodatnią dla mniejszego limitu.
- Jak wybierać providery:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (allowlista rozdzielana przecinkami)
- Skąd pochodzą klucze:
  - Domyślnie: ze store profili i fallbacków env
  - Ustaw `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby wymusić wyłącznie **store profili**
- Dlaczego to istnieje:
  - Oddziela „API providera jest zepsute / klucz jest nieprawidłowy” od „pipeline agenta gateway jest zepsuty”
  - Zawiera małe, izolowane regresje (przykład: OpenAI Responses/Codex Responses reasoning replay + przepływy tool-call)

### Warstwa 2: Gateway + smoke agenta deweloperskiego (to, co naprawdę robi "@openclaw")

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Cel:
  - Uruchomić in-process gateway
  - Utworzyć/załatać sesję `agent:dev:*` (nadpisanie modelu dla każdego uruchomienia)
  - Iterować po modelach-z-kluczami i potwierdzać:
    - „sensowną” odpowiedź (bez narzędzi)
    - że działa prawdziwe wywołanie narzędzia (`read` probe)
    - opcjonalne dodatkowe sondy narzędzi (`exec+read` probe)
    - że ścieżki regresji OpenAI (samo tool-call → follow-up) nadal działają
- Szczegóły sond (aby szybciej wyjaśniać awarie):
  - sonda `read`: test zapisuje plik nonce w workspace i prosi agenta, aby go `read` oraz odesłał nonce.
  - sonda `exec+read`: test prosi agenta, aby zapisał nonce do pliku tymczasowego przez `exec`, a następnie odczytał go przez `read`.
  - sonda obrazu: test dołącza wygenerowany PNG (kot + losowy kod) i oczekuje, że model zwróci `cat <CODE>`.
  - Referencja implementacji: `src/gateway/gateway-models.profiles.live.test.ts` oraz `src/gateway/live-image-probe.ts`.
- Jak włączyć:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli wywołujesz Vitest bezpośrednio)
- Jak wybierać modele:
  - Domyślnie: nowoczesna allowlista (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` jest aliasem dla nowoczesnej allowlisty
  - Albo ustaw `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (lub listę rozdzielaną przecinkami), aby zawęzić
  - Przeglądy gateway modern/all domyślnie używają dobranego limitu o wysokim sygnale; ustaw `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` dla wyczerpującego przeglądu modern lub liczbę dodatnią dla mniejszego limitu.
- Jak wybierać providery (aby uniknąć „wszystkiego z OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (allowlista rozdzielana przecinkami)
- Sondy narzędzi + obrazu są zawsze włączone w tym teście live:
  - sonda `read` + sonda `exec+read` (stress test narzędzi)
  - sonda obrazu działa, gdy model ogłasza obsługę wejścia obrazowego
  - Przepływ (wysoki poziom):
    - Test generuje mały PNG z „CAT” + losowym kodem (`src/gateway/live-image-probe.ts`)
    - Wysyła go przez `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway parsuje załączniki do `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Embedded agent przekazuje do modelu multimodalną wiadomość użytkownika
    - Asercja: odpowiedź zawiera `cat` + kod (tolerancja OCR: dopuszczalne drobne pomyłki)

Wskazówka: aby zobaczyć, co możesz testować na swojej maszynie (oraz dokładne identyfikatory `provider/model`), uruchom:

```bash
openclaw models list
openclaw models list --json
```

## Live: smoke backendu CLI (Claude, Codex, Gemini lub inne lokalne CLI)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Cel: zweryfikować pipeline Gateway + agent przy użyciu lokalnego backendu CLI, bez dotykania domyślnej konfiguracji.
- Domyślne ustawienia smoke specyficzne dla backendu znajdują się w definicji `cli-backend.ts` należącej do odpowiedniego rozszerzenia.
- Włączenie:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli wywołujesz Vitest bezpośrednio)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Domyślne ustawienia:
  - Domyślny provider/model: `claude-cli/claude-sonnet-4-6`
  - Zachowanie polecenia/argumentów/obrazów pochodzi z metadanych odpowiedniego pluginu backendu CLI.
- Nadpisania (opcjonalne):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` aby wysłać prawdziwy załącznik obrazu (ścieżki są wstrzykiwane do promptu).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` aby przekazywać ścieżki plików obrazów jako argumenty CLI zamiast przez wstrzyknięcie do promptu.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (lub `"list"`) aby kontrolować sposób przekazywania argumentów obrazów, gdy ustawione jest `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` aby wysłać drugą turę i zweryfikować przepływ wznowienia.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` aby wyłączyć domyślną sondę ciągłości tej samej sesji Claude Sonnet -> Opus (ustaw `1`, aby wymusić ją, gdy wybrany model obsługuje cel przełączenia).

Przykład:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Recepta Docker:

```bash
pnpm test:docker:live-cli-backend
```

Recepty Docker dla pojedynczego providera:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Uwagi:

- Runner Dockera znajduje się w `scripts/test-live-cli-backend-docker.sh`.
- Uruchamia smoke live backendu CLI wewnątrz obrazu Docker repo jako nieuprzywilejowany użytkownik `node`.
- Rozwiązuje metadane smoke CLI z odpowiedniego rozszerzenia, a następnie instaluje odpowiedni pakiet Linux CLI (`@anthropic-ai/claude-code`, `@openai/codex` lub `@google/gemini-cli`) do zapisywalnego prefiksu cache w `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (domyślnie: `~/.cache/openclaw/docker-cli-tools`).
- Smoke live backendu CLI testuje teraz ten sam przepływ end-to-end dla Claude, Codex i Gemini: tura tekstowa, tura klasyfikacji obrazu, a następnie wywołanie narzędzia MCP `cron` zweryfikowane przez CLI gateway.
- Domyślny smoke Claude dodatkowo łata sesję z Sonnet do Opus i sprawdza, że wznowiona sesja nadal pamięta wcześniejszą notatkę.

## Live: smoke ACP bind (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Cel: zweryfikować rzeczywisty przepływ conversation-bind ACP z aktywnym agentem ACP:
  - wysłać `/acp spawn <agent> --bind here`
  - powiązać syntetyczną konwersację kanału wiadomości w miejscu
  - wysłać zwykły follow-up w tej samej konwersacji
  - zweryfikować, że follow-up trafia do transkryptu powiązanej sesji ACP
- Włączenie:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Domyślne ustawienia:
  - Agenci ACP w Dockerze: `claude,codex,gemini`
  - Agent ACP dla bezpośredniego `pnpm test:live ...`: `claude`
  - Syntetyczny kanał: kontekst konwersacji w stylu DM Slacka
  - Backend ACP: `acpx`
- Nadpisania:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Uwagi:
  - Ta ścieżka używa powierzchni gateway `chat.send` z polami synthetic originating-route tylko dla administratora, aby testy mogły dołączać kontekst kanału wiadomości bez udawania dostarczenia na zewnątrz.
  - Gdy `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` nie jest ustawione, test używa wbudowanego rejestru agentów pluginu `acpx` dla wybranego agenta harness ACP.

Przykład:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Recepta Docker:

```bash
pnpm test:docker:live-acp-bind
```

Recepty Docker dla pojedynczego agenta:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Uwagi dotyczące Dockera:

- Runner Dockera znajduje się w `scripts/test-live-acp-bind-docker.sh`.
- Domyślnie uruchamia smoke ACP bind kolejno dla wszystkich obsługiwanych agentów live CLI: `claude`, `codex`, a następnie `gemini`.
- Użyj `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` lub `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, aby zawęzić macierz.
- Pobiera `~/.profile`, przygotowuje odpowiednie materiały auth CLI w kontenerze, instaluje `acpx` do zapisywalnego prefiksu npm, a następnie instaluje wymagane live CLI (`@anthropic-ai/claude-code`, `@openai/codex` lub `@google/gemini-cli`), jeśli go brakuje.
- Wewnątrz Dockera runner ustawia `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, aby `acpx` zachował zmienne env providera z pobranego profilu dostępne dla potomnego harnessu CLI.

### Zalecane recepty live

Wąskie, jawne allowlisty są najszybsze i najmniej podatne na błędy:

- Pojedynczy model, direct (bez gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Pojedynczy model, gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Wywoływanie narzędzi u kilku providerów:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Skupienie na Google (klucz API Gemini + Antigravity):
  - Gemini (klucz API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Uwagi:

- `google/...` używa API Gemini (klucz API).
- `google-antigravity/...` używa mostu OAuth Antigravity (endpoint agenta w stylu Cloud Code Assist).
- `google-gemini-cli/...` używa lokalnego Gemini CLI na twojej maszynie (osobne auth + specyfika narzędzi).
- Gemini API vs Gemini CLI:
  - API: OpenClaw wywołuje hostowane API Gemini Google przez HTTP (auth kluczem API / profilem); to właśnie większość użytkowników ma na myśli, mówiąc „Gemini”.
  - CLI: OpenClaw wykonuje lokalny binarny plik `gemini`; ma własne auth i może zachowywać się inaczej (streaming/obsługa narzędzi/rozjazd wersji).

## Live: macierz modeli (co obejmujemy)

Nie ma stałej „listy modeli CI” (live jest opt-in), ale to są **zalecane** modele do regularnego pokrywania na maszynie deweloperskiej z kluczami.

### Nowoczesny zestaw smoke (wywoływanie narzędzi + obraz)

To jest uruchomienie „typowych modeli”, które powinno stale działać:

- OpenAI (nie-Codex): `openai/gpt-5.4` (opcjonalnie: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (lub `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` oraz `google/gemini-3-flash-preview` (unikaj starszych modeli Gemini 2.x)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` oraz `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Uruchom gateway smoke z narzędziami + obrazem:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Bazowe: wywoływanie narzędzi (Read + opcjonalnie Exec)

Wybierz co najmniej jeden model z każdej rodziny providerów:

- OpenAI: `openai/gpt-5.4` (lub `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (lub `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (lub `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Opcjonalne dodatkowe pokrycie (warto mieć):

- xAI: `xai/grok-4` (lub najnowszy dostępny)
- Mistral: `mistral/`… (wybierz jeden model z obsługą narzędzi, który masz włączony)
- Cerebras: `cerebras/`… (jeśli masz dostęp)
- LM Studio: `lmstudio/`… (lokalnie; wywoływanie narzędzi zależy od trybu API)

### Vision: wysyłanie obrazu (załącznik → wiadomość multimodalna)

Uwzględnij przynajmniej jeden model obsługujący obrazy w `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/warianty OpenAI z obsługą vision itd.), aby przetestować sondę obrazu.

### Agregatory / alternatywne gateway

Jeśli masz włączone klucze, obsługujemy też testowanie przez:

- OpenRouter: `openrouter/...` (setki modeli; użyj `openclaw models scan`, aby znaleźć kandydatów z obsługą narzędzi i obrazów)
- OpenCode: `opencode/...` dla Zen oraz `opencode-go/...` dla Go (auth przez `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Więcej providerów, których możesz dodać do macierzy live (jeśli masz poświadczenia/konfigurację):

- Wbudowani: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Przez `models.providers` (własne endpointy): `minimax` (cloud/API), plus dowolny proxy zgodny z OpenAI/Anthropic (LM Studio, vLLM, LiteLLM itd.)

Wskazówka: nie próbuj na sztywno wpisywać „wszystkich modeli” w dokumentacji. Autorytatywną listą jest to, co zwraca `discoverModels(...)` na twojej maszynie + jakie klucze są dostępne.

## Poświadczenia (nigdy nie commituj)

Testy live wykrywają poświadczenia tak samo jak CLI. W praktyce oznacza to:

- Jeśli CLI działa, testy live powinny znaleźć te same klucze.
- Jeśli test live zgłasza „no creds”, debuguj to tak samo, jak debugowałbyś `openclaw models list` / wybór modelu.

- Profile auth per agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (to właśnie oznaczają „profile keys” w testach live)
- Konfiguracja: `~/.openclaw/openclaw.json` (lub `OPENCLAW_CONFIG_PATH`)
- Starszy katalog stanu: `~/.openclaw/credentials/` (kopiowany do przygotowanego katalogu live home, jeśli istnieje, ale nie jest głównym store kluczy profili)
- Lokalne uruchomienia live domyślnie kopiują aktywną konfigurację, pliki `auth-profiles.json` per agent, starsze `credentials/` oraz obsługiwane zewnętrzne katalogi auth CLI do tymczasowego katalogu domowego testów; przygotowane katalogi live home pomijają `workspace/` i `sandboxes/`, a nadpisania ścieżek `agents.*.workspace` / `agentDir` są usuwane, aby sondy nie działały na twoim rzeczywistym workspace hosta.

Jeśli chcesz polegać na kluczach env (np. wyeksportowanych w `~/.profile`), uruchamiaj testy lokalne po `source ~/.profile` albo użyj poniższych runnerów Docker (mogą zamontować `~/.profile` do kontenera).

## Deepgram live (transkrypcja audio)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Włączenie: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- Test: `src/agents/byteplus.live.test.ts`
- Włączenie: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Opcjonalne nadpisanie modelu: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- Test: `extensions/comfy/comfy.live.test.ts`
- Włączenie: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Zakres:
  - Testuje wbudowane ścieżki comfy dla obrazów, wideo i `music_generate`
  - Pomija każdą możliwość, jeśli `models.providers.comfy.<capability>` nie jest skonfigurowane
  - Przydatne po zmianach w przesyłaniu workflow comfy, odpytywaniu, pobieraniu lub rejestracji pluginu

## Live generowania obrazów

- Test: `src/image-generation/runtime.live.test.ts`
- Polecenie: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Zakres:
  - Wylicza każdy zarejestrowany plugin providera generowania obrazów
  - Ładuje brakujące zmienne env providera z twojego login shell (`~/.profile`) przed wykonaniem sond
  - Domyślnie używa aktywnych/env kluczy API przed zapisanymi profilami auth, aby stare klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń shell
  - Pomija providerów bez użytecznego auth/profilu/modelu
  - Uruchamia standardowe warianty generowania obrazów przez współdzieloną możliwość runtime:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Aktualnie objęci wbudowani providerzy:
  - `openai`
  - `google`
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Opcjonalne zachowanie auth:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` aby wymusić auth ze store profili i ignorować nadpisania tylko-env

## Live generowania muzyki

- Test: `extensions/music-generation-providers.live.test.ts`
- Włączenie: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Zakres:
  - Testuje współdzieloną wbudowaną ścieżkę providera generowania muzyki
  - Obecnie obejmuje Google i MiniMax
  - Ładuje zmienne env providera z twojego login shell (`~/.profile`) przed wykonaniem sond
  - Domyślnie używa aktywnych/env kluczy API przed zapisanymi profilami auth, aby stare klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń shell
  - Pomija providerów bez użytecznego auth/profilu/modelu
  - Uruchamia oba zadeklarowane tryby runtime, gdy są dostępne:
    - `generate` z wejściem opartym wyłącznie na prompt
    - `edit`, gdy provider deklaruje `capabilities.edit.enabled`
  - Obecne pokrycie współdzielonej ścieżki:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: osobny plik Comfy live, nie ten współdzielony przegląd
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Opcjonalne zachowanie auth:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` aby wymusić auth ze store profili i ignorować nadpisania tylko-env

## Live generowania wideo

- Test: `extensions/video-generation-providers.live.test.ts`
- Włączenie: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Zakres:
  - Testuje współdzieloną wbudowaną ścieżkę providera generowania wideo
  - Ładuje zmienne env providera z twojego login shell (`~/.profile`) przed wykonaniem sond
  - Domyślnie używa aktywnych/env kluczy API przed zapisanymi profilami auth, aby stare klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń shell
  - Pomija providerów bez użytecznego auth/profilu/modelu
  - Uruchamia oba zadeklarowane tryby runtime, gdy są dostępne:
    - `generate` z wejściem opartym wyłącznie na prompt
    - `imageToVideo`, gdy provider deklaruje `capabilities.imageToVideo.enabled` i wybrany provider/model akceptuje lokalne wejście obrazu oparte na buforze we współdzielonym przeglądzie
    - `videoToVideo`, gdy provider deklaruje `capabilities.videoToVideo.enabled` i wybrany provider/model akceptuje lokalne wejście wideo oparte na buforze we współdzielonym przeglądzie
  - Aktualnie zadeklarowani, ale pomijani providerzy `imageToVideo` we współdzielonym przeglądzie:
    - `vydra`, ponieważ wbudowane `veo3` jest tylko tekstowe, a wbudowany `kling` wymaga zdalnego URL obrazu
  - Pokrycie specyficzne dla providera Vydra:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - ten plik uruchamia `veo3` text-to-video oraz ścieżkę `kling`, która domyślnie używa fixtury ze zdalnym URL obrazu
  - Aktualne pokrycie live `videoToVideo`:
    - tylko `runway`, gdy wybranym modelem jest `runway/gen4_aleph`
  - Aktualnie zadeklarowani, ale pomijani providerzy `videoToVideo` we współdzielonym przeglądzie:
    - `alibaba`, `qwen`, `xai`, ponieważ te ścieżki wymagają obecnie zdalnych referencyjnych URL `http(s)` / MP4
    - `google`, ponieważ obecna współdzielona ścieżka Gemini/Veo używa lokalnego wejścia opartego na buforze i ta ścieżka nie jest akceptowana we współdzielonym przeglądzie
    - `openai`, ponieważ obecna współdzielona ścieżka nie gwarantuje dostępu do specyficznych dla organizacji funkcji video inpaint/remix
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Opcjonalne zachowanie auth:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` aby wymusić auth ze store profili i ignorować nadpisania tylko-env

## Harness live mediów

- Polecenie: `pnpm test:live:media`
- Cel:
  - Uruchamia współdzielone pakiety live dla obrazów, muzyki i wideo przez jedno natywne wejście repo
  - Automatycznie ładuje brakujące zmienne env providera z `~/.profile`
  - Domyślnie automatycznie zawęża każdy pakiet do providerów, którzy aktualnie mają użyteczne auth
  - Ponownie używa `scripts/test-live.mjs`, dzięki czemu zachowanie heartbeat i quiet mode pozostaje spójne
- Przykłady:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Runnery Docker (opcjonalne kontrole „działa w Linuxie”)

Te runnery Docker dzielą się na dwa koszyki:

- Runnery live-model: `test:docker:live-models` i `test:docker:live-gateway` uruchamiają tylko odpowiadający im plik live z kluczami profili wewnątrz obrazu Docker repo (`src/agents/models.profiles.live.test.ts` i `src/gateway/gateway-models.profiles.live.test.ts`), montując lokalny katalog konfiguracji i workspace (oraz pobierając `~/.profile`, jeśli jest zamontowany). Odpowiadające im lokalne entrypointy to `test:live:models-profiles` i `test:live:gateway-profiles`.
- Runnery Docker live domyślnie używają mniejszego limitu smoke, aby pełny przegląd Docker pozostawał praktyczny:
  `test:docker:live-models` domyślnie ustawia `OPENCLAW_LIVE_MAX_MODELS=12`, a
  `test:docker:live-gateway` domyślnie ustawia `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` oraz
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Nadpisz te zmienne env, gdy
  jawnie chcesz uruchomić większy, wyczerpujący przegląd.
- `test:docker:all` buduje obraz live Docker jeden raz przez `test:docker:live-build`, a następnie ponownie używa go dla dwóch ścieżek Docker live.
- Runnery smoke kontenerów: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` oraz `test:docker:plugins` uruchamiają jeden lub więcej rzeczywistych kontenerów i weryfikują ścieżki integracji wyższego poziomu.

Runnery Docker live-model bind-mountują też tylko potrzebne katalogi auth CLI (lub wszystkie obsługiwane, gdy uruchomienie nie jest zawężone), a następnie kopiują je do katalogu domowego kontenera przed startem, aby zewnętrzny OAuth CLI mógł odświeżać tokeny bez modyfikowania store auth hosta:

- Direct models: `pnpm test:docker:live-models` (skrypt: `scripts/test-live-models-docker.sh`)
- Smoke ACP bind: `pnpm test:docker:live-acp-bind` (skrypt: `scripts/test-live-acp-bind-docker.sh`)
- Smoke backendu CLI: `pnpm test:docker:live-cli-backend` (skrypt: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + agent deweloperski: `pnpm test:docker:live-gateway` (skrypt: `scripts/test-live-gateway-models-docker.sh`)
- Live smoke Open WebUI: `pnpm test:docker:openwebui` (skrypt: `scripts/e2e/openwebui-docker.sh`)
- Kreator onboardingu (TTY, pełne scaffoldowanie): `pnpm test:docker:onboard` (skrypt: `scripts/e2e/onboard-docker.sh`)
- Sieć gateway (dwa kontenery, auth WS + health): `pnpm test:docker:gateway-network` (skrypt: `scripts/e2e/gateway-network-docker.sh`)
- Most kanałów MCP (seedowany Gateway + most stdio + smoke surowych ramek powiadomień Claude): `pnpm test:docker:mcp-channels` (skrypt: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke instalacji + alias `/plugin` + semantyka restartu pakietu Claude): `pnpm test:docker:plugins` (skrypt: `scripts/e2e/plugins-docker.sh`)

Runnery Docker live-model bind-mountują też bieżący checkout tylko do odczytu i
przygotowują go do tymczasowego katalogu roboczego wewnątrz kontenera. Dzięki temu obraz runtime
pozostaje lekki, a Vitest nadal działa na dokładnie twoich lokalnych źródłach/konfiguracji.
Etap przygotowania pomija duże lokalne cache i wyniki buildów aplikacji, takie jak
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` oraz lokalne dla aplikacji katalogi `.build` lub
wyniki Gradle, aby uruchomienia Docker live nie traciły minut na kopiowanie
artefaktów specyficznych dla maszyny.
Ustawiają one również `OPENCLAW_SKIP_CHANNELS=1`, aby sondy gateway live nie uruchamiały
rzeczywistych workerów kanałów Telegram/Discord/itd. wewnątrz kontenera.
`test:docker:live-models` nadal uruchamia `pnpm test:live`, więc przekaż także
`OPENCLAW_LIVE_GATEWAY_*`, gdy musisz zawęzić lub wykluczyć pokrycie gateway
live w tej ścieżce Docker.
`test:docker:openwebui` jest smoke kompatybilności wyższego poziomu: uruchamia
kontener gateway OpenClaw z włączonymi endpointami HTTP zgodnymi z OpenAI,
uruchamia przypięty kontener Open WebUI względem tego gateway, loguje się przez
Open WebUI, weryfikuje, że `/api/models` udostępnia `openclaw/default`, a następnie wysyła
rzeczywiste żądanie czatu przez proxy `/api/chat/completions` Open WebUI.
Pierwsze uruchomienie może być zauważalnie wolniejsze, ponieważ Docker może potrzebować pobrać
obraz Open WebUI, a Open WebUI może potrzebować zakończyć własną konfigurację cold-start.
Ta ścieżka oczekuje użytecznego klucza modelu live, a `OPENCLAW_PROFILE_FILE`
(domyslnie `~/.profile`) jest podstawowym sposobem dostarczenia go w uruchomieniach Docker.
Udane uruchomienia wypisują mały ładunek JSON, taki jak `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` jest celowo deterministyczny i nie potrzebuje
prawdziwego konta Telegram, Discord ani iMessage. Uruchamia seedowany kontener Gateway,
startuje drugi kontener, który uruchamia `openclaw mcp serve`, a następnie
weryfikuje wykrywanie routowanych konwersacji, odczyty transkryptów, metadane załączników,
zachowanie kolejki zdarzeń live, routing wysyłki wychodzącej oraz powiadomienia w stylu Claude o kanałach +
uprawnieniach przez rzeczywisty most stdio MCP. Kontrola powiadomień
sprawdza bezpośrednio surowe ramki stdio MCP, więc smoke weryfikuje to, co most
rzeczywiście emituje, a nie tylko to, co akurat udostępnia konkretne SDK klienta.

Ręczny smoke wątku ACP w plain language (nie CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Zachowaj ten skrypt do przepływów regresji/debugowania. Może być ponownie potrzebny do walidacji routingu wątków ACP, więc nie usuwaj go.

Przydatne zmienne env:

- `OPENCLAW_CONFIG_DIR=...` (domyślnie: `~/.openclaw`) montowane do `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (domyślnie: `~/.openclaw/workspace`) montowane do `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (domyślnie: `~/.profile`) montowane do `/home/node/.profile` i pobierane przed uruchomieniem testów
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (domyślnie: `~/.cache/openclaw/docker-cli-tools`) montowane do `/home/node/.npm-global` dla cache instalacji CLI wewnątrz Dockera
- Zewnętrzne katalogi/pliki auth CLI pod `$HOME` są montowane tylko do odczytu pod `/host-auth...`, a następnie kopiowane do `/home/node/...` przed uruchomieniem testów
  - Domyślne katalogi: `.minimax`
  - Domyślne pliki: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Zawężone uruchomienia providerów montują tylko potrzebne katalogi/pliki wywnioskowane z `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Nadpisanie ręczne: `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` lub lista rozdzielana przecinkami, np. `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` aby zawęzić uruchomienie
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` aby filtrować providerów w kontenerze
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` aby upewnić się, że poświadczenia pochodzą ze store profili (a nie z env)
- `OPENCLAW_OPENWEBUI_MODEL=...` aby wybrać model udostępniany przez gateway dla smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` aby nadpisać prompt kontroli nonce używany przez smoke Open WebUI
- `OPENWEBUI_IMAGE=...` aby nadpisać przypięty tag obrazu Open WebUI

## Spójność dokumentacji

Po edycji dokumentacji uruchom kontrolę dokumentacji: `pnpm check:docs`.
Uruchom pełną walidację anchorów Mintlify, gdy potrzebujesz również kontroli nagłówków na stronie: `pnpm docs:check-links:anchors`.

## Regresje offline (bezpieczne dla CI)

Są to regresje „rzeczywistego pipeline” bez prawdziwych providerów:

- Wywoływanie narzędzi gateway (mock OpenAI, rzeczywista pętla gateway + agent): `src/gateway/gateway.test.ts` (przypadek: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Kreator gateway (WS `wizard.start`/`wizard.next`, zapisuje config + wymuszone auth): `src/gateway/gateway.test.ts` (przypadek: "runs wizard over ws and writes auth token config")

## Ewalucje niezawodności agenta (Skills)

Mamy już kilka bezpiecznych dla CI testów, które zachowują się jak „ewaluacje niezawodności agenta”:

- Mock wywoływania narzędzi przez rzeczywistą pętlę gateway + agent (`src/gateway/gateway.test.ts`).
- Przepływy kreatora end-to-end, które walidują okablowanie sesji i efekty konfiguracji (`src/gateway/gateway.test.ts`).

Czego nadal brakuje dla Skills (zobacz [Skills](/pl/tools/skills)):

- **Decisioning:** gdy Skills są wymienione w prompcie, czy agent wybiera właściwy skill (albo unika nieistotnych)?
- **Compliance:** czy agent czyta `SKILL.md` przed użyciem i wykonuje wymagane kroki/argumenty?
- **Kontrakty przepływu pracy:** scenariusze wieloturowe, które potwierdzają kolejność narzędzi, przenoszenie historii sesji i granice sandboxa.

Przyszłe ewaluacje powinny najpierw pozostać deterministyczne:

- Runner scenariuszy używający mock providerów do potwierdzania wywołań narzędzi + kolejności, odczytów plików skill i okablowania sesji.
- Mały pakiet scenariuszy skupionych na skillach (użyj vs unikaj, gating, prompt injection).
- Opcjonalne ewaluacje live (opt-in, sterowane env) dopiero po wdrożeniu pakietu bezpiecznego dla CI.

## Testy kontraktowe (kształt pluginów i kanałów)

Testy kontraktowe weryfikują, że każdy zarejestrowany plugin i kanał jest zgodny ze swoim
kontraktem interfejsu. Iterują po wszystkich wykrytych pluginach i uruchamiają pakiet
asercji kształtu i zachowania. Domyślna ścieżka unit `pnpm test` celowo
pomija te współdzielone pliki seam i smoke; uruchamiaj polecenia kontraktów jawnie,
gdy dotykasz współdzielonych powierzchni kanałów lub providerów.

### Polecenia

- Wszystkie kontrakty: `pnpm test:contracts`
- Tylko kontrakty kanałów: `pnpm test:contracts:channels`
- Tylko kontrakty providerów: `pnpm test:contracts:plugins`

### Kontrakty kanałów

Znajdują się w `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Podstawowy kształt pluginu (id, name, capabilities)
- **setup** - Kontrakt kreatora konfiguracji
- **session-binding** - Zachowanie wiązania sesji
- **outbound-payload** - Struktura ładunku wiadomości
- **inbound** - Obsługa wiadomości przychodzących
- **actions** - Handlery akcji kanału
- **threading** - Obsługa identyfikatorów wątków
- **directory** - API katalogu/listy kontaktów
- **group-policy** - Egzekwowanie polityki grup

### Kontrakty statusu providerów

Znajdują się w `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondy statusu kanału
- **registry** - Kształt rejestru pluginów

### Kontrakty providerów

Znajdują się w `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Kontrakt przepływu auth
- **auth-choice** - Wybór/selekcja auth
- **catalog** - API katalogu modeli
- **discovery** - Wykrywanie pluginów
- **loader** - Ładowanie pluginów
- **runtime** - Runtime providera
- **shape** - Kształt/interfejs pluginu
- **wizard** - Kreator konfiguracji

### Kiedy uruchamiać

- Po zmianie eksportów lub subpaths `plugin-sdk`
- Po dodaniu lub modyfikacji pluginu kanału lub providera
- Po refaktoryzacji rejestracji pluginów lub wykrywania

Testy kontraktowe uruchamiają się w CI i nie wymagają prawdziwych kluczy API.

## Dodawanie regresji (wskazówki)

Gdy naprawiasz problem providera/modelu wykryty w live:

- Jeśli to możliwe, dodaj regresję bezpieczną dla CI (mock/stub providera albo przechwycenie dokładnej transformacji kształtu żądania)
- Jeśli problem z natury występuje tylko live (rate limit, polityki auth), utrzymaj test live wąski i opt-in przez zmienne env
- Staraj się celować w najmniejszą warstwę, która wychwytuje błąd:
  - błąd konwersji/odtwarzania żądania providera → test direct models
  - błąd pipeline sesji/historii/narzędzi gateway → gateway live smoke albo bezpieczny dla CI test mock gateway
- Zabezpieczenie przechodzenia SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` wyprowadza jeden przykładowy cel dla każdej klasy SecretRef z metadanych rejestru (`listSecretTargetRegistryEntries()`), a następnie potwierdza, że identyfikatory exec segmentów przejścia są odrzucane.
  - Jeśli dodasz nową rodzinę celów SecretRef `includeInPlan` w `src/secrets/target-registry-data.ts`, zaktualizuj `classifyTargetClass` w tym teście. Test celowo kończy się niepowodzeniem dla niesklasyfikowanych identyfikatorów celów, aby nowe klasy nie mogły zostać po cichu pominięte.
