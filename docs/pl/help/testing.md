---
read_when:
    - Uruchamianie testów lokalnie lub w CI
    - Dodawanie testów regresji dla błędów modeli/dostawców
    - Debugowanie zachowania Gateway i agenta
summary: 'Zestaw testowy: pakiety testów unit/e2e/live, uruchamianie w Dockerze oraz zakres poszczególnych testów'
title: Testowanie
x-i18n:
    generated_at: "2026-04-16T21:51:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: af2bc0e9b5e08ca3119806d355b517290f6078fda430109e7a0b153586215e34
    source_path: help/testing.md
    workflow: 15
---

# Testowanie

OpenClaw ma trzy pakiety testów Vitest (unit/integration, e2e, live) oraz niewielki zestaw uruchomień w Dockerze.

Ten dokument jest przewodnikiem „jak testujemy”:

- Co obejmuje każdy pakiet testów (i czego celowo _nie_ obejmuje)
- Jakie polecenia uruchamiać w typowych przepływach pracy (lokalnie, przed pushem, debugowanie)
- Jak testy live wykrywają poświadczenia oraz wybierają modele/dostawców
- Jak dodawać testy regresji dla rzeczywistych problemów modeli/dostawców

## Szybki start

Na co dzień:

- Pełna bramka (oczekiwana przed pushem): `pnpm build && pnpm check && pnpm test`
- Szybsze lokalne uruchomienie pełnego pakietu na wydajnej maszynie: `pnpm test:max`
- Bezpośrednia pętla watch Vitest: `pnpm test:watch`
- Bezpośrednie wskazanie pliku obsługuje teraz także ścieżki rozszerzeń/kanałów: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Gdy iterujesz nad pojedynczą awarią, najpierw preferuj uruchomienia ukierunkowane.
- Witryna QA oparta na Dockerze: `pnpm qa:lab:up`
- Ścieżka QA oparta na maszynie wirtualnej Linux: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Gdy modyfikujesz testy lub chcesz mieć większą pewność:

- Bramka pokrycia: `pnpm test:coverage`
- Pakiet E2E: `pnpm test:e2e`

Podczas debugowania rzeczywistych dostawców/modeli (wymaga prawdziwych poświadczeń):

- Pakiet live (modele + sondy narzędzi/obrazów Gateway): `pnpm test:live`
- Ciche uruchomienie jednego pliku live: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Wskazówka: gdy potrzebujesz tylko jednego nieudanego przypadku, zawężaj testy live za pomocą zmiennych środowiskowych allowlist opisanych poniżej.

## Uruchomienia specyficzne dla QA

Te polecenia działają obok głównych pakietów testów, gdy potrzebujesz realizmu QA-lab:

- `pnpm openclaw qa suite`
  - Uruchamia scenariusze QA oparte na repozytorium bezpośrednio na hoście.
  - Domyślnie uruchamia wiele wybranych scenariuszy równolegle z izolowanymi workerami Gateway, maksymalnie do 64 workerów lub liczby wybranych scenariuszy. Użyj `--concurrency <count>`, aby dostroić liczbę workerów, albo `--concurrency 1`, aby użyć starszej ścieżki sekwencyjnej.
- `pnpm openclaw qa suite --runner multipass`
  - Uruchamia ten sam pakiet QA wewnątrz tymczasowej maszyny wirtualnej Multipass Linux.
  - Zachowuje to samo wybieranie scenariuszy co `qa suite` na hoście.
  - Używa tych samych flag wyboru dostawcy/modelu co `qa suite`.
  - Uruchomienia live przekazują do gościa obsługiwane wejścia autoryzacji QA, które są praktyczne: klucze dostawców oparte na env, ścieżkę konfiguracji dostawcy QA live oraz `CODEX_HOME`, jeśli jest obecne.
  - Katalogi wyjściowe muszą pozostać w katalogu głównym repozytorium, aby gość mógł zapisywać przez zamontowany workspace.
  - Zapisuje standardowy raport i podsumowanie QA oraz logi Multipass w `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Uruchamia witrynę QA opartą na Dockerze do pracy QA w stylu operatora.
- `pnpm openclaw qa matrix`
  - Uruchamia ścieżkę QA live dla Matrix względem tymczasowego homeservera Tuwunel opartego na Dockerze.
  - Ten host QA jest obecnie przeznaczony tylko do repozytorium/developmentu. Spakowane instalacje OpenClaw nie zawierają `qa-lab`, więc nie udostępniają `openclaw qa`.
  - Check-outy repozytorium ładują dołączony runner bezpośrednio; nie jest potrzebny osobny krok instalacji Plugin.
  - Tworzy trzech tymczasowych użytkowników Matrix (`driver`, `sut`, `observer`) oraz jeden prywatny pokój, a następnie uruchamia podrzędny proces QA Gateway z rzeczywistym Plugin Matrix jako transportem SUT.
  - Domyślnie używa przypiętego stabilnego obrazu Tuwunel `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Nadpisz przez `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`, jeśli chcesz przetestować inny obraz.
  - Matrix nie udostępnia współdzielonych flag źródła poświadczeń, ponieważ ścieżka tworzy tymczasowych użytkowników lokalnie.
  - Zapisuje raport QA Matrix, podsumowanie, artefakt observed-events oraz połączony log stdout/stderr w `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Uruchamia ścieżkę QA live dla Telegram względem rzeczywistej prywatnej grupy, używając tokenów bota driver i SUT z env.
  - Wymaga `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` oraz `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. Identyfikator grupy musi być numerycznym identyfikatorem czatu Telegram.
  - Obsługuje `--credential-source convex` dla współdzielonych poświadczeń z puli. Domyślnie używaj trybu env albo ustaw `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`, aby włączyć współdzielone dzierżawy.
  - Wymaga dwóch różnych botów w tej samej prywatnej grupie, przy czym bot SUT musi udostępniać nazwę użytkownika Telegram.
  - Aby uzyskać stabilną obserwację bot-do-bota, włącz Bot-to-Bot Communication Mode w `@BotFather` dla obu botów i upewnij się, że bot driver może obserwować ruch botów w grupie.
  - Zapisuje raport QA Telegram, podsumowanie oraz artefakt observed-messages w `.artifacts/qa-e2e/...`.

Ścieżki transportowe live współdzielą jeden standardowy kontrakt, aby nowe transporty nie odchodziły od ustalonego wzorca:

`qa-channel` pozostaje szerokim syntetycznym pakietem QA i nie jest częścią macierzy pokrycia transportów live.

| Ścieżka  | Canary | Bramka wzmianek | Blokada allowlist | Odpowiedź najwyższego poziomu | Wznowienie po restarcie | Dalszy ciąg wątku | Izolacja wątku | Obserwacja reakcji | Polecenie help |
| -------- | ------ | --------------- | ----------------- | ----------------------------- | ----------------------- | ----------------- | -------------- | ------------------ | -------------- |
| Matrix   | x      | x               | x                 | x                             | x                       | x                 | x              | x                  |                |
| Telegram | x      |                 |                   |                               |                         |                   |                |                    | x              |

### Współdzielone poświadczenia Telegram przez Convex (v1)

Gdy dla `openclaw qa telegram` włączone jest `--credential-source convex` (lub `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`), QA lab pobiera wyłączną dzierżawę z puli opartej na Convex, wysyła Heartbeat tej dzierżawy podczas działania ścieżki i zwalnia dzierżawę przy zamknięciu.

Referencyjny szablon projektu Convex:

- `qa/convex-credential-broker/`

Wymagane zmienne środowiskowe:

- `OPENCLAW_QA_CONVEX_SITE_URL` (na przykład `https://your-deployment.convex.site`)
- Jeden sekret dla wybranej roli:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` dla `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` dla `ci`
- Wybór roli poświadczeń:
  - CLI: `--credential-role maintainer|ci`
  - Domyślne z env: `OPENCLAW_QA_CREDENTIAL_ROLE` (domyślnie `maintainer`)

Opcjonalne zmienne środowiskowe:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (domyślnie `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (domyślnie `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (domyślnie `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (domyślnie `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (domyślnie `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (opcjonalny identyfikator śledzenia)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` pozwala na adresy Convex `http://` dla loopback wyłącznie do lokalnego developmentu.

`OPENCLAW_QA_CONVEX_SITE_URL` w normalnym użyciu powinno korzystać z `https://`.

Administracyjne polecenia maintainera (dodawanie/usuwanie/listowanie puli) wymagają konkretnie `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

Pomocnicze polecenia CLI dla maintainerów:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Użyj `--json`, aby uzyskać wynik czytelny maszynowo w skryptach i narzędziach CI.

Domyślny kontrakt endpointu (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - Żądanie: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Sukces: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Wyczerpane/możliwe do ponowienia: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - Żądanie: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Sukces: `{ status: "ok" }` (lub puste `2xx`)
- `POST /release`
  - Żądanie: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Sukces: `{ status: "ok" }` (lub puste `2xx`)
- `POST /admin/add` (tylko sekret maintainera)
  - Żądanie: `{ kind, actorId, payload, note?, status? }`
  - Sukces: `{ status: "ok", credential }`
- `POST /admin/remove` (tylko sekret maintainera)
  - Żądanie: `{ credentialId, actorId }`
  - Sukces: `{ status: "ok", changed, credential }`
  - Ochrona aktywnej dzierżawy: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (tylko sekret maintainera)
  - Żądanie: `{ kind?, status?, includePayload?, limit? }`
  - Sukces: `{ status: "ok", credentials, count }`

Kształt payloadu dla rodzaju Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` musi być ciągiem z numerycznym identyfikatorem czatu Telegram.
- `admin/add` waliduje ten kształt dla `kind: "telegram"` i odrzuca nieprawidłowy payload.

### Dodawanie kanału do QA

Dodanie kanału do systemu QA opartego na Markdown wymaga dokładnie dwóch rzeczy:

1. Adaptera transportu dla kanału.
2. Pakietu scenariuszy, który testuje kontrakt kanału.

Nie dodawaj nowego głównego korzenia poleceń QA, jeśli współdzielony host `qa-lab` może obsłużyć ten przepływ.

`qa-lab` odpowiada za współdzieloną mechanikę hosta:

- korzeń poleceń `openclaw qa`
- uruchamianie i zamykanie pakietu
- współbieżność workerów
- zapisywanie artefaktów
- generowanie raportów
- wykonywanie scenariuszy
- aliasy zgodności dla starszych scenariuszy `qa-channel`

Pluginy runnerów odpowiadają za kontrakt transportu:

- sposób montowania `openclaw qa <runner>` pod współdzielonym korzeniem `qa`
- sposób konfiguracji Gateway dla tego transportu
- sposób sprawdzania gotowości
- sposób wstrzykiwania zdarzeń przychodzących
- sposób obserwacji wiadomości wychodzących
- sposób udostępniania transkryptów i znormalizowanego stanu transportu
- sposób wykonywania akcji opartych na transporcie
- sposób obsługi resetu lub czyszczenia specyficznego dla transportu

Minimalny próg wdrożenia nowego kanału:

1. Zachowaj `qa-lab` jako właściciela współdzielonego korzenia `qa`.
2. Zaimplementuj runner transportu na współdzielonym styku hosta `qa-lab`.
3. Zachowaj mechanikę specyficzną dla transportu wewnątrz pluginu runnera lub harnessu Plugin.
4. Zamontuj runner jako `openclaw qa <runner>`, zamiast rejestrować konkurencyjny główny korzeń poleceń.  
   Pluginy runnerów powinny deklarować `qaRunners` w `openclaw.plugin.json` i eksportować pasującą tablicę `qaRunnerCliRegistrations` z `runtime-api.ts`.  
   Zachowaj lekkość `runtime-api.ts`; leniwe wykonanie CLI i runnera powinno pozostać za osobnymi entrypointami.
5. Napisz lub zaadaptuj scenariusze Markdown w `qa/scenarios/`.
6. W nowych scenariuszach używaj generycznych helperów scenariuszy.
7. Zachowaj działanie istniejących aliasów zgodności, chyba że repozytorium przechodzi celową migrację.

Zasada decyzyjna jest ścisła:

- Jeśli zachowanie można wyrazić jednokrotnie w `qa-lab`, umieść je w `qa-lab`.
- Jeśli zachowanie zależy od transportu jednego kanału, pozostaw je w pluginie tego runnera lub harnessie Plugin.
- Jeśli scenariusz potrzebuje nowej możliwości, z której może skorzystać więcej niż jeden kanał, dodaj generyczny helper zamiast gałęzi specyficznej dla kanału w `suite.ts`.
- Jeśli zachowanie ma sens tylko dla jednego transportu, pozostaw scenariusz jako specyficzny dla transportu i zaznacz to jawnie w kontrakcie scenariusza.

Preferowane nazwy generycznych helperów dla nowych scenariuszy:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Aliasy zgodności pozostają dostępne dla istniejących scenariuszy, w tym:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Nowe prace nad kanałami powinny używać generycznych nazw helperów.
Aliasy zgodności istnieją po to, by uniknąć migracji typu flag day, a nie jako model dla
tworzenia nowych scenariuszy.

## Pakiety testów (co uruchamia się gdzie)

Myśl o pakietach testów jako o „rosnącym realizmie” (i rosnącej niestabilności/koszcie):

### Unit / integration (domyślne)

- Polecenie: `pnpm test`
- Konfiguracja: dziesięć sekwencyjnych shardów (`vitest.full-*.config.ts`) uruchamianych na istniejących zakresowych projektach Vitest
- Pliki: inwentarze core/unit w `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` oraz dopuszczone testy Node w `ui` objęte przez `vitest.unit.config.ts`
- Zakres:
  - Czyste testy unit
  - Testy integracyjne w tym samym procesie (autoryzacja Gateway, routing, narzędzia, parsowanie, konfiguracja)
  - Deterministyczne testy regresji dla znanych błędów
- Oczekiwania:
  - Uruchamia się w CI
  - Nie wymaga prawdziwych kluczy
  - Powinno być szybkie i stabilne
- Uwaga o projektach:
  - Niezakresowe `pnpm test` uruchamia teraz jedenaście mniejszych konfiguracji shardów (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) zamiast jednego wielkiego natywnego procesu projektu głównego. Zmniejsza to szczytowe RSS na obciążonych maszynach i zapobiega temu, by prace `auto-reply`/rozszerzeń zagłodziły niezwiązane pakiety.
  - `pnpm test --watch` nadal używa natywnego grafu projektów z głównego `vitest.config.ts`, ponieważ pętla watch dla wielu shardów nie jest praktyczna.
  - `pnpm test`, `pnpm test:watch` i `pnpm test:perf:imports` kierują jawne cele plików/katalogów najpierw do zakresowych ścieżek, więc `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` unika kosztu uruchamiania pełnego projektu głównego.
  - `pnpm test:changed` rozwija zmienione ścieżki git do tych samych zakresowych ścieżek, gdy diff dotyczy wyłącznie routowalnych plików źródłowych/testowych; zmiany konfiguracji/ustawień nadal wracają do szerokiego ponownego uruchomienia projektu głównego.
  - Lekkie importowo testy unit z agentów, poleceń, pluginów, helperów `auto-reply`, `plugin-sdk` i podobnych czysto użytkowych obszarów są kierowane przez ścieżkę `unit-fast`, która pomija `test/setup-openclaw-runtime.ts`; pliki stanowe/ciężkie runtime pozostają na istniejących ścieżkach.
  - Wybrane pliki źródłowe helperów `plugin-sdk` i `commands` również mapują uruchomienia w trybie changed do jawnych testów sąsiednich w tych lekkich ścieżkach, dzięki czemu edycje helperów nie wymagają ponownego uruchamiania pełnego ciężkiego pakietu dla tego katalogu.
  - `auto-reply` ma teraz trzy dedykowane koszyki: helpery core najwyższego poziomu, testy integracyjne najwyższego poziomu `reply.*` oraz poddrzewo `src/auto-reply/reply/**`. Dzięki temu najcięższa praca harnessu odpowiedzi nie trafia do tanich testów status/chunk/token.
- Uwaga o osadzonym runnerze:
  - Gdy zmieniasz wejścia wykrywania message-tool lub kontekst runtime Compaction,
    zachowaj oba poziomy pokrycia.
  - Dodaj ukierunkowane testy regresji helperów dla czystych granic routingu/normalizacji.
  - Utrzymuj też w dobrym stanie osadzone pakiety testów runnera integracyjnego:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` oraz
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Te pakiety weryfikują, że zakresowe identyfikatory i zachowanie Compaction nadal przepływają
    przez rzeczywiste ścieżki `run.ts` / `compact.ts`; testy wyłącznie helperów nie są
    wystarczającym zamiennikiem dla tych ścieżek integracyjnych.
- Uwaga o puli:
  - Bazowa konfiguracja Vitest domyślnie używa `threads`.
  - Współdzielona konfiguracja Vitest ustawia też na stałe `isolate: false` i używa nieizolowanego runnera w projektach głównych, konfiguracjach e2e i live.
  - Główna ścieżka UI zachowuje ustawienie `jsdom` i optymalizator, ale teraz również działa na współdzielonym nieizolowanym runnerze.
  - Każdy shard `pnpm test` dziedziczy te same domyślne ustawienia `threads` + `isolate: false` ze współdzielonej konfiguracji Vitest.
  - Współdzielony launcher `scripts/run-vitest.mjs` domyślnie dodaje teraz także `--no-maglev` dla podrzędnych procesów Node Vitest, aby ograniczyć churn kompilacji V8 podczas dużych lokalnych uruchomień. Ustaw `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, jeśli chcesz porównać zachowanie ze standardowym V8.
- Uwaga o szybkiej iteracji lokalnej:
  - `pnpm test:changed` kieruje przez zakresowe ścieżki, gdy zmienione ścieżki można jednoznacznie przypisać do mniejszego pakietu.
  - `pnpm test:max` i `pnpm test:changed:max` zachowują to samo routowanie, tylko z wyższym limitem workerów.
  - Automatyczne skalowanie workerów lokalnych jest teraz celowo konserwatywne i dodatkowo ogranicza się, gdy średnie obciążenie hosta jest już wysokie, dzięki czemu wiele równoczesnych uruchomień Vitest domyślnie powoduje mniej szkód.
  - Bazowa konfiguracja Vitest oznacza projekty/pliki konfiguracji jako `forceRerunTriggers`, aby ponowne uruchomienia w trybie changed pozostawały poprawne po zmianach w okablowaniu testów.
  - Konfiguracja utrzymuje `OPENCLAW_VITEST_FS_MODULE_CACHE` włączone na obsługiwanych hostach; ustaw `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, jeśli chcesz mieć jedną jawną lokalizację cache do bezpośredniego profilowania.
- Uwaga o debugowaniu wydajności:
  - `pnpm test:perf:imports` włącza raportowanie czasu importu Vitest oraz wynik z rozbiciem importów.
  - `pnpm test:perf:imports:changed` ogranicza ten sam widok profilowania do plików zmienionych od `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` porównuje routowane `test:changed` z natywną ścieżką projektu głównego dla zatwierdzonego diffu i wypisuje czas ścienny oraz maksymalne RSS na macOS.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkuje bieżące brudne drzewo, kierując listę zmienionych plików przez `scripts/test-projects.mjs` oraz główną konfigurację Vitest.
  - `pnpm test:perf:profile:main` zapisuje profil CPU głównego wątku dla kosztu uruchamiania i transformacji Vitest/Vite.
  - `pnpm test:perf:profile:runner` zapisuje profile CPU+heap runnera dla pakietu unit przy wyłączonej równoległości plików.

### E2E (test dymny Gateway)

- Polecenie: `pnpm test:e2e`
- Konfiguracja: `vitest.e2e.config.ts`
- Pliki: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Domyślne ustawienia runtime:
  - Używa Vitest `threads` z `isolate: false`, zgodnie z resztą repozytorium.
  - Używa adaptacyjnej liczby workerów (CI: maksymalnie 2, lokalnie: domyślnie 1).
  - Domyślnie działa w trybie silent, aby ograniczyć narzut I/O konsoli.
- Przydatne nadpisania:
  - `OPENCLAW_E2E_WORKERS=<n>` aby wymusić liczbę workerów (limit 16).
  - `OPENCLAW_E2E_VERBOSE=1` aby ponownie włączyć szczegółowy output konsoli.
- Zakres:
  - Wieloinstancyjne zachowanie end-to-end Gateway
  - Powierzchnie WebSocket/HTTP, parowanie Node oraz cięższa komunikacja sieciowa
- Oczekiwania:
  - Uruchamia się w CI (gdy jest włączone w pipeline)
  - Nie wymaga prawdziwych kluczy
  - Ma więcej ruchomych części niż testy unit (może być wolniejsze)

### E2E: test dymny backendu OpenShell

- Polecenie: `pnpm test:e2e:openshell`
- Plik: `test/openshell-sandbox.e2e.test.ts`
- Zakres:
  - Uruchamia na hoście izolowany Gateway OpenShell przez Docker
  - Tworzy sandbox z tymczasowego lokalnego Dockerfile
  - Testuje backend OpenShell OpenClaw przez rzeczywiste `sandbox ssh-config` + wykonanie SSH
  - Weryfikuje zdalne kanoniczne zachowanie systemu plików przez most fs sandboxa
- Oczekiwania:
  - Tylko opt-in; nie jest częścią domyślnego uruchomienia `pnpm test:e2e`
  - Wymaga lokalnego CLI `openshell` oraz działającego demona Docker
  - Używa izolowanych `HOME` / `XDG_CONFIG_HOME`, a następnie niszczy testowy Gateway i sandbox
- Przydatne nadpisania:
  - `OPENCLAW_E2E_OPENSHELL=1` aby włączyć test przy ręcznym uruchamianiu szerszego pakietu e2e
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` aby wskazać niestandardowy binarny plik CLI lub skrypt wrapper

### Live (rzeczywiści dostawcy + rzeczywiste modele)

- Polecenie: `pnpm test:live`
- Konfiguracja: `vitest.live.config.ts`
- Pliki: `src/**/*.live.test.ts`
- Domyślnie: **włączone** przez `pnpm test:live` (ustawia `OPENCLAW_LIVE_TEST=1`)
- Zakres:
  - „Czy ten dostawca/model faktycznie działa _dzisiaj_ z prawdziwymi poświadczeniami?”
  - Wykrywa zmiany formatów dostawców, specyfikę wywołań narzędzi, problemy z autoryzacją i zachowanie limitów szybkości
- Oczekiwania:
  - Celowo nie jest stabilne w CI (rzeczywiste sieci, rzeczywiste polityki dostawców, limity, awarie)
  - Kosztuje pieniądze / zużywa limity szybkości
  - Lepiej uruchamiać zawężone podzbiory niż „wszystko”
- Uruchomienia live pobierają `~/.profile`, aby przechwycić brakujące klucze API.
- Domyślnie uruchomienia live nadal izolują `HOME` i kopiują materiały konfiguracyjne/autoryzacyjne do tymczasowego katalogu domowego testów, aby fixtury unit nie mogły modyfikować rzeczywistego `~/.openclaw`.
- Ustaw `OPENCLAW_LIVE_USE_REAL_HOME=1` tylko wtedy, gdy celowo chcesz, aby testy live używały rzeczywistego katalogu domowego.
- `pnpm test:live` domyślnie działa teraz w cichszym trybie: zachowuje output postępu `[live] ...`, ale ukrywa dodatkową informację o `~/.profile` i wycisza logi bootstrapu Gateway/hałas Bonjour. Ustaw `OPENCLAW_LIVE_TEST_QUIET=0`, jeśli chcesz z powrotem pełne logi uruchamiania.
- Rotacja kluczy API (specyficzna dla dostawcy): ustaw `*_API_KEYS` w formacie z przecinkami/średnikami lub `*_API_KEY_1`, `*_API_KEY_2` (na przykład `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) albo nadpisanie per live przez `OPENCLAW_LIVE_*_KEY`; testy ponawiają próbę przy odpowiedziach o limicie szybkości.
- Output postępu/Heartbeat:
  - Pakiety live emitują teraz linie postępu do stderr, dzięki czemu długie wywołania dostawców są wyraźnie aktywne nawet wtedy, gdy przechwytywanie konsoli Vitest jest ciche.
  - `vitest.live.config.ts` wyłącza przechwytywanie konsoli Vitest, dzięki czemu linie postępu dostawcy/Gateway są streamowane natychmiast podczas uruchomień live.
  - Heartbeat dla modeli bezpośrednich można dostroić przez `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Heartbeat Gateway/sond można dostroić przez `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Który pakiet testów uruchomić?

Użyj tej tabeli decyzyjnej:

- Edycja logiki/testów: uruchom `pnpm test` (oraz `pnpm test:coverage`, jeśli zmieniło się dużo)
- Zmiany w sieci Gateway / protokole WS / parowaniu: dodaj `pnpm test:e2e`
- Debugowanie „mój bot nie działa” / awarii specyficznych dla dostawcy / wywołań narzędzi: uruchom zawężone `pnpm test:live`

## Live: przegląd możliwości Node Android

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skrypt: `pnpm android:test:integration`
- Cel: wywołać **każde polecenie aktualnie ogłaszane** przez podłączony Node Android i potwierdzić zachowanie zgodne z kontraktem poleceń.
- Zakres:
  - Ręczna konfiguracja wstępna / wstępnie spełnione warunki (pakiet nie instaluje, nie uruchamia ani nie paruje aplikacji).
  - Walidacja `node.invoke` Gateway polecenie po poleceniu dla wybranego Node Android.
- Wymagana konfiguracja wstępna:
  - Aplikacja Android jest już połączona i sparowana z Gateway.
  - Aplikacja pozostaje na pierwszym planie.
  - Uprawnienia/zgody na przechwytywanie są przyznane dla możliwości, które mają przejść.
- Opcjonalne nadpisania celu:
  - `OPENCLAW_ANDROID_NODE_ID` lub `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Pełne szczegóły konfiguracji Android: [Aplikacja Android](/pl/platforms/android)

## Live: test dymny modeli (klucze profili)

Testy live są podzielone na dwie warstwy, aby można było izolować awarie:

- „Model bezpośredni” mówi nam, czy dostawca/model w ogóle odpowiada przy danym kluczu.
- „Test dymny Gateway” mówi nam, czy pełny pipeline Gateway+agenta działa dla danego modelu (sesje, historia, narzędzia, polityka sandboxa itd.).

### Warstwa 1: Bezpośrednie zakończenie modelu (bez Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Cel:
  - Wyliczyć wykryte modele
  - Użyć `getApiKeyForModel`, aby wybrać modele, dla których masz poświadczenia
  - Uruchomić małe zakończenie dla każdego modelu (oraz ukierunkowane regresje tam, gdzie to potrzebne)
- Jak włączyć:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli uruchamiasz Vitest bezpośrednio)
- Ustaw `OPENCLAW_LIVE_MODELS=modern` (lub `all`, alias dla modern), aby faktycznie uruchomić ten pakiet; w przeciwnym razie zostanie pominięty, żeby `pnpm test:live` pozostawało skupione na teście dymnym Gateway
- Jak wybierać modele:
  - `OPENCLAW_LIVE_MODELS=modern`, aby uruchomić nowoczesną allowlist (`Opus/Sonnet 4.6+`, `GPT-5.x + Codex`, `Gemini 3`, `GLM 4.7`, `MiniMax M2.7`, `Grok 4`)
  - `OPENCLAW_LIVE_MODELS=all` jest aliasem dla nowoczesnej allowlist
  - albo `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist rozdzielana przecinkami)
  - Przebiegi modern/all domyślnie używają dobranego limitu o wysokim sygnale; ustaw `OPENCLAW_LIVE_MAX_MODELS=0`, aby wykonać wyczerpujący przebieg modern, albo dodatnią liczbę dla mniejszego limitu.
- Jak wybierać dostawców:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (allowlist rozdzielana przecinkami)
- Skąd pochodzą klucze:
  - Domyślnie: magazyn profili i zapasowe wartości z env
  - Ustaw `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby wymusić wyłącznie **magazyn profili**
- Dlaczego to istnieje:
  - Oddziela „API dostawcy jest uszkodzone / klucz jest nieprawidłowy” od „pipeline agenta Gateway jest uszkodzony”
  - Zawiera małe, izolowane testy regresji (na przykład przepływy odtwarzania rozumowania OpenAI Responses/Codex Responses + wywołań narzędzi)

### Warstwa 2: test dymny Gateway + agenta deweloperskiego (to, co faktycznie robi „@openclaw”)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Cel:
  - Uruchomić Gateway w tym samym procesie
  - Utworzyć/zmodyfikować sesję `agent:dev:*` (nadpisanie modelu dla każdego uruchomienia)
  - Iterować po modelach-z-kluczami i potwierdzać:
    - „sensowną” odpowiedź (bez narzędzi)
    - że rzeczywiste wywołanie narzędzia działa (sonda `read`)
    - opcjonalne dodatkowe sondy narzędzi (sonda `exec+read`)
    - że ścieżki regresji OpenAI (tylko tool-call → dalszy ciąg) nadal działają
- Szczegóły sond (żeby można było szybko wyjaśniać awarie):
  - sonda `read`: test zapisuje plik nonce w workspace i prosi agenta, aby go `read` i odesłał nonce.
  - sonda `exec+read`: test prosi agenta, aby zapisał nonce do pliku tymczasowego przez `exec`, a następnie odczytał go przez `read`.
  - sonda obrazu: test dołącza wygenerowany plik PNG (kot + zrandomizowany kod) i oczekuje, że model zwróci `cat <CODE>`.
  - Odniesienie do implementacji: `src/gateway/gateway-models.profiles.live.test.ts` oraz `src/gateway/live-image-probe.ts`.
- Jak włączyć:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli uruchamiasz Vitest bezpośrednio)
- Jak wybierać modele:
  - Domyślnie: nowoczesna allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` jest aliasem dla nowoczesnej allowlist
  - Albo ustaw `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (lub listę rozdzielaną przecinkami), aby zawęzić
  - Przebiegi Gateway modern/all domyślnie używają dobranego limitu o wysokim sygnale; ustaw `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`, aby wykonać wyczerpujący przebieg modern, albo dodatnią liczbę dla mniejszego limitu.
- Jak wybierać dostawców (unikając „całego OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (allowlist rozdzielana przecinkami)
- Sondy narzędzi i obrazu są zawsze włączone w tym teście live:
  - sonda `read` + sonda `exec+read` (obciążenie narzędzi)
  - sonda obrazu uruchamia się, gdy model deklaruje obsługę wejścia obrazowego
  - Przepływ (wysoki poziom):
    - Test generuje mały PNG z napisem „CAT” + losowym kodem (`src/gateway/live-image-probe.ts`)
    - Wysyła go przez `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway parsuje załączniki do `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Osadzony agent przekazuje do modelu multimodalną wiadomość użytkownika
    - Potwierdzenie: odpowiedź zawiera `cat` + kod (tolerancja OCR: drobne błędy są dopuszczalne)

Wskazówka: aby zobaczyć, co możesz testować na swojej maszynie (oraz dokładne identyfikatory `provider/model`), uruchom:

```bash
openclaw models list
openclaw models list --json
```

## Live: test dymny backendu CLI (Claude, Codex, Gemini lub inne lokalne CLI)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Cel: zweryfikować pipeline Gateway + agenta z użyciem lokalnego backendu CLI, bez naruszania domyślnej konfiguracji.
- Domyślne ustawienia testu dymnego specyficzne dla backendu znajdują się w definicji `cli-backend.ts` należącej do odpowiedniego rozszerzenia.
- Włączanie:
  - `pnpm test:live` (lub `OPENCLAW_LIVE_TEST=1`, jeśli uruchamiasz Vitest bezpośrednio)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Domyślne:
  - Domyślny dostawca/model: `claude-cli/claude-sonnet-4-6`
  - Zachowanie command/args/image pochodzi z metadanych Plugin backendu CLI będącego właścicielem.
- Nadpisania (opcjonalne):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`, aby wysłać rzeczywisty załącznik obrazu (ścieżki są wstrzykiwane do promptu).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`, aby przekazywać ścieżki plików obrazów jako argumenty CLI zamiast przez wstrzyknięcie do promptu.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (lub `"list"`), aby sterować sposobem przekazywania argumentów obrazów, gdy ustawione jest `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`, aby wysłać drugą turę i zweryfikować przepływ wznowienia.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`, aby wyłączyć domyślną sondę ciągłości tej samej sesji Claude Sonnet -> Opus (ustaw `1`, aby wymusić jej włączenie, gdy wybrany model obsługuje cel przełączenia).

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

Recepty Docker dla pojedynczego dostawcy:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Uwagi:

- Runner Docker znajduje się w `scripts/test-live-cli-backend-docker.sh`.
- Uruchamia test dymny live backendu CLI wewnątrz repozytoryjnego obrazu Docker jako użytkownik bez uprawnień root `node`.
- Rozwiązuje metadane testu dymnego CLI z rozszerzenia będącego właścicielem, a następnie instaluje pasujący pakiet CLI dla Linux (`@anthropic-ai/claude-code`, `@openai/codex` lub `@google/gemini-cli`) do buforowanego zapisywalnego prefiksu pod `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (domyślnie: `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` wymaga przenośnego OAuth subskrypcji Claude Code przez `~/.claude/.credentials.json` z `claudeAiOauth.subscriptionType` albo `CLAUDE_CODE_OAUTH_TOKEN` z `claude setup-token`. Najpierw potwierdza bezpośrednie `claude -p` w Dockerze, a następnie uruchamia dwie tury Gateway backendu CLI bez zachowywania zmiennych env klucza API Anthropic. Ta ścieżka subskrypcji domyślnie wyłącza sondy Claude MCP/tool oraz obrazu, ponieważ Claude obecnie rozlicza użycie aplikacji zewnętrznych przez dodatkowe opłaty za użycie, a nie normalne limity planu subskrypcyjnego.
- Test dymny live backendu CLI wykonuje teraz ten sam pełny przepływ end-to-end dla Claude, Codex i Gemini: tura tekstowa, tura klasyfikacji obrazu, a następnie wywołanie narzędzia MCP `cron` zweryfikowane przez CLI Gateway.
- Domyślny test dymny Claude dodatkowo modyfikuje sesję z Sonnet na Opus i sprawdza, czy wznowiona sesja nadal pamięta wcześniejszą notatkę.

## Live: test dymny bind ACP (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Cel: zweryfikować rzeczywisty przepływ bindowania rozmowy ACP z aktywnym agentem ACP:
  - wysłać `/acp spawn <agent> --bind here`
  - zbindować syntetyczną rozmowę kanału wiadomości w miejscu
  - wysłać normalny dalszy ciąg w tej samej rozmowie
  - sprawdzić, czy dalszy ciąg trafia do transkryptu zbindowanej sesji ACP
- Włączanie:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Domyślne:
  - Agenci ACP w Dockerze: `claude,codex,gemini`
  - Agent ACP dla bezpośredniego `pnpm test:live ...`: `claude`
  - Kanał syntetyczny: kontekst rozmowy w stylu Slack DM
  - Backend ACP: `acpx`
- Nadpisania:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Uwagi:
  - Ta ścieżka używa powierzchni `chat.send` Gateway z administracyjnymi polami syntetycznej trasy źródłowej, aby testy mogły dołączać kontekst kanału wiadomości bez udawania dostarczenia na zewnątrz.
  - Gdy `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` nie jest ustawione, test używa wbudowanego rejestru agentów Plugin `acpx` dla wybranego agenta harnessu ACP.

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

- Runner Docker znajduje się w `scripts/test-live-acp-bind-docker.sh`.
- Domyślnie uruchamia test dymny bind ACP sekwencyjnie dla wszystkich obsługiwanych aktywnych agentów CLI: `claude`, `codex`, a następnie `gemini`.
- Użyj `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` lub `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, aby zawęzić macierz.
- Pobiera `~/.profile`, przygotowuje pasujące materiały autoryzacyjne CLI w kontenerze, instaluje `acpx` do zapisywalnego prefiksu npm, a następnie instaluje wymagane aktywne CLI (`@anthropic-ai/claude-code`, `@openai/codex` lub `@google/gemini-cli`), jeśli go brakuje.
- Wewnątrz Dockera runner ustawia `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, aby `acpx` zachował zmienne env dostawcy z pobranego profilu dostępne dla podrzędnego CLI harnessu.

## Live: test dymny harnessu app-server Codex

- Cel: zweryfikować należący do Plugin harness Codex przez normalną
  metodę Gateway `agent`:
  - załadować dołączony Plugin `codex`
  - wybrać `OPENCLAW_AGENT_RUNTIME=codex`
  - wysłać pierwszą turę agenta Gateway do `codex/gpt-5.4`
  - wysłać drugą turę do tej samej sesji OpenClaw i sprawdzić, czy wątek
    app-server może zostać wznowiony
  - uruchomić `/codex status` oraz `/codex models` przez tę samą ścieżkę
    poleceń Gateway
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Włączanie: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Domyślny model: `codex/gpt-5.4`
- Opcjonalna sonda obrazu: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Opcjonalna sonda MCP/narzędzia: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Test dymny ustawia `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, aby uszkodzony harness Codex
  nie mógł przejść przez ciche przełączenie awaryjne na PI.
- Autoryzacja: `OPENAI_API_KEY` z powłoki/profilu oraz opcjonalnie skopiowane
  `~/.codex/auth.json` i `~/.codex/config.toml`

Recepta lokalna:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Recepta Docker:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Uwagi dotyczące Dockera:

- Runner Docker znajduje się w `scripts/test-live-codex-harness-docker.sh`.
- Pobiera zamontowane `~/.profile`, przekazuje `OPENAI_API_KEY`, kopiuje pliki
  autoryzacyjne CLI Codex, jeśli są obecne, instaluje `@openai/codex` do zapisywalnego zamontowanego prefiksu npm,
  przygotowuje drzewo źródeł, a następnie uruchamia tylko aktywny test harnessu Codex.
- Docker domyślnie włącza sondy obrazu oraz MCP/narzędzia. Ustaw
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` lub
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`, gdy potrzebujesz bardziej zawężonego uruchomienia debugującego.
- Docker eksportuje również `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, zgodnie z konfiguracją testu live,
  tak aby przełączenie awaryjne `openai-codex/*` lub PI nie mogło ukryć regresji
  harnessu Codex.

### Zalecane recepty live

Wąskie, jawne allowlist są najszybsze i najmniej podatne na niestabilność:

- Pojedynczy model, bezpośrednio (bez Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Pojedynczy model, test dymny Gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Wywoływanie narzędzi dla kilku dostawców:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Skupienie na Google (klucz API Gemini + Antigravity):
  - Gemini (klucz API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Uwagi:

- `google/...` używa API Gemini (klucz API).
- `google-antigravity/...` używa mostu OAuth Antigravity (endpoint agenta w stylu Cloud Code Assist).
- `google-gemini-cli/...` używa lokalnego CLI Gemini na Twojej maszynie (osobna autoryzacja + specyficzne zachowania narzędzi).
- Gemini API vs Gemini CLI:
  - API: OpenClaw wywołuje hostowane API Gemini Google przez HTTP (autoryzacja przez klucz API / profil); to właśnie większość użytkowników ma na myśli, mówiąc „Gemini”.
  - CLI: OpenClaw uruchamia lokalny binarny plik `gemini`; ma własną autoryzację i może zachowywać się inaczej (streaming/obsługa narzędzi/rozjazd wersji).

## Live: macierz modeli (co obejmujemy)

Nie ma stałej „listy modeli CI” (live jest opt-in), ale to są **zalecane** modele do regularnego pokrywania na maszynie deweloperskiej z kluczami.

### Nowoczesny zestaw smoke (wywoływanie narzędzi + obraz)

To jest uruchomienie „typowych modeli”, które oczekujemy utrzymywać w działaniu:

- OpenAI (bez Codex): `openai/gpt-5.4` (opcjonalnie: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (lub `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` oraz `google/gemini-3-flash-preview` (unikaj starszych modeli Gemini 2.x)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` oraz `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Uruchom smoke Gateway z narzędziami + obrazem:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Poziom bazowy: wywoływanie narzędzi (Read + opcjonalnie Exec)

Wybierz co najmniej jeden model z każdej rodziny dostawców:

- OpenAI: `openai/gpt-5.4` (lub `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (lub `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (lub `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Opcjonalne dodatkowe pokrycie (mile widziane):

- xAI: `xai/grok-4` (lub najnowszy dostępny)
- Mistral: `mistral/`… (wybierz jeden model obsługujący `tools`, który masz włączony)
- Cerebras: `cerebras/`… (jeśli masz dostęp)
- LM Studio: `lmstudio/`… (lokalnie; wywoływanie narzędzi zależy od trybu API)

### Vision: wysyłanie obrazu (załącznik → wiadomość multimodalna)

Uwzględnij co najmniej jeden model obsługujący obrazy w `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/warianty OpenAI obsługujące vision itd.), aby przetestować sondę obrazu.

### Agregatory / alternatywne Gateway

Jeśli masz włączone klucze, obsługujemy też testowanie przez:

- OpenRouter: `openrouter/...` (setki modeli; użyj `openclaw models scan`, aby znaleźć kandydatów obsługujących narzędzia + obrazy)
- OpenCode: `opencode/...` dla Zen oraz `opencode-go/...` dla Go (autoryzacja przez `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Więcej dostawców, których możesz uwzględnić w macierzy live (jeśli masz poświadczenia/konfigurację):

- Wbudowani: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Przez `models.providers` (niestandardowe endpointy): `minimax` (cloud/API) oraz dowolny proxy zgodny z OpenAI/Anthropic (LM Studio, vLLM, LiteLLM itd.)

Wskazówka: nie próbuj na stałe wpisywać „wszystkich modeli” do dokumentacji. Autorytatywną listą jest to, co zwraca `discoverModels(...)` na Twojej maszynie + dostępne klucze.

## Poświadczenia (nigdy nie commituj)

Testy live wykrywają poświadczenia w ten sam sposób co CLI. Praktyczne konsekwencje:

- Jeśli CLI działa, testy live powinny znaleźć te same klucze.
- Jeśli test live mówi „brak poświadczeń”, debuguj to tak samo, jak debugowałbyś `openclaw models list` / wybór modelu.

- Profile autoryzacji per agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (to właśnie oznaczają „klucze profili” w testach live)
- Konfiguracja: `~/.openclaw/openclaw.json` (lub `OPENCLAW_CONFIG_PATH`)
- Katalog legacy state: `~/.openclaw/credentials/` (kopiowany do przygotowanego katalogu domowego live, jeśli istnieje, ale nie jest głównym magazynem kluczy profili)
- Lokalne uruchomienia live domyślnie kopiują aktywną konfigurację, pliki `auth-profiles.json` per agent, legacy `credentials/` oraz obsługiwane zewnętrzne katalogi autoryzacji CLI do tymczasowego katalogu domowego testów; przygotowane katalogi domowe live pomijają `workspace/` i `sandboxes/`, a nadpisania ścieżek `agents.*.workspace` / `agentDir` są usuwane, aby sondy nie działały na Twoim rzeczywistym workspace hosta.

Jeśli chcesz polegać na kluczach z env (np. eksportowanych w `~/.profile`), uruchamiaj testy lokalne po `source ~/.profile` albo użyj poniższych runnerów Docker (mogą montować `~/.profile` do kontenera).

## Live Deepgram (transkrypcja audio)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Włączanie: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live planu kodowania BytePlus

- Test: `src/agents/byteplus.live.test.ts`
- Włączanie: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Opcjonalne nadpisanie modelu: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live mediów workflow ComfyUI

- Test: `extensions/comfy/comfy.live.test.ts`
- Włączanie: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Zakres:
  - Testuje dołączone ścieżki obrazu, wideo i `music_generate` Comfy
  - Pomija każdą możliwość, jeśli `models.providers.comfy.<capability>` nie jest skonfigurowane
  - Przydatne po zmianach w przesyłaniu workflow Comfy, odpytywaniu, pobieraniu lub rejestracji Plugin

## Live generowania obrazów

- Test: `src/image-generation/runtime.live.test.ts`
- Polecenie: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Zakres:
  - Wylicza każdy zarejestrowany Plugin dostawcy generowania obrazów
  - Ładuje brakujące zmienne env dostawców z powłoki logowania (`~/.profile`) przed sondowaniem
  - Domyślnie używa aktywnych/środowiskowych kluczy API przed zapisanymi profilami autoryzacji, aby nieaktualne klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń z powłoki
  - Pomija dostawców bez używalnej autoryzacji/profilu/modelu
  - Uruchamia standardowe warianty generowania obrazów przez współdzieloną możliwość runtime:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Obecnie pokrywani dołączeni dostawcy:
  - `openai`
  - `google`
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Opcjonalne zachowanie autoryzacji:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby wymusić autoryzację z magazynu profili i ignorować nadpisania wyłącznie z env

## Live generowania muzyki

- Test: `extensions/music-generation-providers.live.test.ts`
- Włączanie: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Zakres:
  - Testuje współdzieloną dołączoną ścieżkę dostawców generowania muzyki
  - Obecnie obejmuje Google i MiniMax
  - Ładuje zmienne env dostawców z powłoki logowania (`~/.profile`) przed sondowaniem
  - Domyślnie używa aktywnych/środowiskowych kluczy API przed zapisanymi profilami autoryzacji, aby nieaktualne klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń z powłoki
  - Pomija dostawców bez używalnej autoryzacji/profilu/modelu
  - Uruchamia oba zadeklarowane tryby runtime, gdy są dostępne:
    - `generate` z wejściem wyłącznie prompt
    - `edit`, gdy dostawca deklaruje `capabilities.edit.enabled`
  - Obecne pokrycie współdzielonej ścieżki:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: osobny plik live Comfy, nie ten współdzielony przebieg
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Opcjonalne zachowanie autoryzacji:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby wymusić autoryzację z magazynu profili i ignorować nadpisania wyłącznie z env

## Live generowania wideo

- Test: `extensions/video-generation-providers.live.test.ts`
- Włączanie: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Zakres:
  - Testuje współdzieloną dołączoną ścieżkę dostawców generowania wideo
  - Domyślnie używa bezpiecznej dla wydań ścieżki smoke: dostawcy inni niż FAL, jedno żądanie text-to-video na dostawcę, jednosekundowy prompt lobster oraz limit operacji na dostawcę z `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (domyślnie `180000`)
  - Domyślnie pomija FAL, ponieważ opóźnienia kolejek po stronie dostawcy mogą dominować czas wydania; przekaż `--video-providers fal` lub `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`, aby uruchomić go jawnie
  - Ładuje zmienne env dostawców z powłoki logowania (`~/.profile`) przed sondowaniem
  - Domyślnie używa aktywnych/środowiskowych kluczy API przed zapisanymi profilami autoryzacji, aby nieaktualne klucze testowe w `auth-profiles.json` nie maskowały rzeczywistych poświadczeń z powłoki
  - Pomija dostawców bez używalnej autoryzacji/profilu/modelu
  - Domyślnie uruchamia tylko `generate`
  - Ustaw `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`, aby uruchamiać również zadeklarowane tryby transformacji, gdy są dostępne:
    - `imageToVideo`, gdy dostawca deklaruje `capabilities.imageToVideo.enabled` i wybrany dostawca/model akceptuje lokalne wejście obrazowe oparte na buforze we współdzielonym przebiegu
    - `videoToVideo`, gdy dostawca deklaruje `capabilities.videoToVideo.enabled` i wybrany dostawca/model akceptuje lokalne wejście wideo oparte na buforze we współdzielonym przebiegu
  - Obecni zadeklarowani, ale pomijani dostawcy `imageToVideo` we współdzielonym przebiegu:
    - `vydra`, ponieważ dołączony `veo3` jest tylko tekstowy, a dołączony `kling` wymaga zdalnego adresu URL obrazu
  - Pokrycie Vydra specyficzne dla dostawcy:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - ten plik uruchamia `veo3` text-to-video oraz ścieżkę `kling`, która domyślnie używa fixture ze zdalnym adresem URL obrazu
  - Obecne pokrycie live `videoToVideo`:
    - tylko `runway`, gdy wybrany model to `runway/gen4_aleph`
  - Obecni zadeklarowani, ale pomijani dostawcy `videoToVideo` we współdzielonym przebiegu:
    - `alibaba`, `qwen`, `xai`, ponieważ te ścieżki obecnie wymagają referencyjnych adresów URL `http(s)` / MP4
    - `google`, ponieważ obecna współdzielona ścieżka Gemini/Veo używa lokalnego wejścia opartego na buforze, a ta ścieżka nie jest akceptowana we współdzielonym przebiegu
    - `openai`, ponieważ obecna współdzielona ścieżka nie gwarantuje dostępu specyficznego dla organizacji do video inpaint/remix
- Opcjonalne zawężanie:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`, aby uwzględnić każdego dostawcę w domyślnym przebiegu, w tym FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`, aby obniżyć limit czasu operacji dla każdego dostawcy w agresywnym przebiegu smoke
- Opcjonalne zachowanie autoryzacji:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby wymusić autoryzację z magazynu profili i ignorować nadpisania wyłącznie z env

## Harness live mediów

- Polecenie: `pnpm test:live:media`
- Cel:
  - Uruchamia współdzielone pakiety live dla obrazów, muzyki i wideo przez jeden natywny dla repozytorium entrypoint
  - Automatycznie ładuje brakujące zmienne env dostawców z `~/.profile`
  - Domyślnie automatycznie zawęża każdy pakiet do dostawców, którzy aktualnie mają używalną autoryzację
  - Ponownie używa `scripts/test-live.mjs`, dzięki czemu zachowanie Heartbeat i trybu cichego pozostaje spójne
- Przykłady:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Runnery Docker (opcjonalne kontrole „działa w Linuxie”)

Te runnery Docker dzielą się na dwa koszyki:

- Runnery live modeli: `test:docker:live-models` oraz `test:docker:live-gateway` uruchamiają wyłącznie odpowiadający im plik live z kluczami profili wewnątrz repozytoryjnego obrazu Docker (`src/agents/models.profiles.live.test.ts` oraz `src/gateway/gateway-models.profiles.live.test.ts`), montując lokalny katalog konfiguracji i workspace (oraz pobierając `~/.profile`, jeśli jest zamontowany). Odpowiadające im lokalne entrypointy to `test:live:models-profiles` oraz `test:live:gateway-profiles`.
- Runnery Docker live domyślnie używają mniejszego limitu smoke, aby pełny przebieg Docker pozostawał praktyczny:
  `test:docker:live-models` domyślnie ustawia `OPENCLAW_LIVE_MAX_MODELS=12`, a
  `test:docker:live-gateway` domyślnie ustawia `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` oraz
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Nadpisz te zmienne env, gdy
  celowo chcesz większego, wyczerpującego skanowania.
- `test:docker:all` buduje obraz Docker live raz przez `test:docker:live-build`, a następnie używa go ponownie dla dwóch ścieżek Docker live.
- Runnery smoke kontenerów: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` oraz `test:docker:plugins` uruchamiają jeden lub więcej rzeczywistych kontenerów i weryfikują ścieżki integracji wyższego poziomu.

Runnery Docker live modeli montują również jako bind tylko potrzebne katalogi domowe autoryzacji CLI (lub wszystkie obsługiwane, gdy uruchomienie nie jest zawężone), a następnie kopiują je do katalogu domowego kontenera przed uruchomieniem, aby zewnętrzne OAuth CLI mogło odświeżać tokeny bez modyfikowania magazynu autoryzacji na hoście:

- Modele bezpośrednie: `pnpm test:docker:live-models` (skrypt: `scripts/test-live-models-docker.sh`)
- Smoke bind ACP: `pnpm test:docker:live-acp-bind` (skrypt: `scripts/test-live-acp-bind-docker.sh`)
- Smoke backendu CLI: `pnpm test:docker:live-cli-backend` (skrypt: `scripts/test-live-cli-backend-docker.sh`)
- Smoke harnessu app-server Codex: `pnpm test:docker:live-codex-harness` (skrypt: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + agent dev: `pnpm test:docker:live-gateway` (skrypt: `scripts/test-live-gateway-models-docker.sh`)
- Live smoke Open WebUI: `pnpm test:docker:openwebui` (skrypt: `scripts/e2e/openwebui-docker.sh`)
- Kreator onboardingu (TTY, pełne scaffoldowanie): `pnpm test:docker:onboard` (skrypt: `scripts/e2e/onboard-docker.sh`)
- Sieć Gateway (dwa kontenery, autoryzacja WS + health): `pnpm test:docker:gateway-network` (skrypt: `scripts/e2e/gateway-network-docker.sh`)
- Most kanałów MCP (zasiany Gateway + most stdio + smoke surowych ramek powiadomień Claude): `pnpm test:docker:mcp-channels` (skrypt: `scripts/e2e/mcp-channels-docker.sh`)
- Pluginy (smoke instalacji + alias `/plugin` + semantyka restartu pakietu Claude): `pnpm test:docker:plugins` (skrypt: `scripts/e2e/plugins-docker.sh`)

Runnery Docker live modeli montują także bieżący checkout tylko do odczytu i
przygotowują go w tymczasowym katalogu roboczym wewnątrz kontenera. Dzięki temu obraz runtime
pozostaje lekki, a Vitest nadal działa względem dokładnie lokalnego źródła/konfiguracji.
Krok przygotowania pomija duże lokalne cache oraz wyjścia buildów aplikacji, takie jak
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` oraz lokalne dla aplikacji katalogi `.build` lub
wyjściowe katalogi Gradle, dzięki czemu uruchomienia Docker live nie tracą minut na kopiowanie
artefaktów specyficznych dla maszyny.
Ustawiają też `OPENCLAW_SKIP_CHANNELS=1`, aby aktywne sondy Gateway nie uruchamiały
rzeczywistych workerów kanałów Telegram/Discord itd. wewnątrz kontenera.
`test:docker:live-models` nadal uruchamia `pnpm test:live`, więc przekazuj również
`OPENCLAW_LIVE_GATEWAY_*`, gdy chcesz zawęzić lub wykluczyć pokrycie Gateway
live z tej ścieżki Docker.
`test:docker:openwebui` jest smoke testem zgodności wyższego poziomu: uruchamia
kontener Gateway OpenClaw z włączonymi endpointami HTTP zgodnymi z OpenAI,
uruchamia przypięty kontener Open WebUI względem tego Gateway, loguje się przez
Open WebUI, sprawdza, że `/api/models` udostępnia `openclaw/default`, a następnie wysyła
rzeczywiste żądanie czatu przez proxy `/api/chat/completions` Open WebUI.
Pierwsze uruchomienie może być zauważalnie wolniejsze, ponieważ Docker może potrzebować pobrać
obraz Open WebUI, a Open WebUI może potrzebować ukończyć własną konfigurację zimnego startu.
Ta ścieżka oczekuje używalnego klucza aktywnego modelu, a `OPENCLAW_PROFILE_FILE`
(domyślnie `~/.profile`) jest podstawowym sposobem jego dostarczenia w uruchomieniach w Dockerze.
Udane uruchomienia wypisują mały payload JSON, taki jak `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` jest celowo deterministyczne i nie wymaga
rzeczywistego konta Telegram, Discord ani iMessage. Uruchamia zasiany kontener
Gateway, startuje drugi kontener, który uruchamia `openclaw mcp serve`, a następnie
weryfikuje wykrywanie rozmów przez routing, odczyty transkryptów, metadane załączników,
zachowanie kolejki zdarzeń live, routing wysyłania wychodzącego oraz powiadomienia
kanału + uprawnień w stylu Claude przez rzeczywisty most stdio MCP. Kontrola powiadomień
sprawdza bezpośrednio surowe ramki stdio MCP, więc smoke waliduje to, co most
faktycznie emituje, a nie tylko to, co akurat udostępnia konkretny SDK klienta.

Ręczny smoke prostego języka dla wątku ACP (nie CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Zachowaj ten skrypt do przepływów pracy regresji/debugowania. Może być znowu potrzebny do walidacji routingu wątków ACP, więc nie usuwaj go.

Przydatne zmienne env:

- `OPENCLAW_CONFIG_DIR=...` (domyślnie: `~/.openclaw`) montowane do `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (domyślnie: `~/.openclaw/workspace`) montowane do `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (domyślnie: `~/.profile`) montowane do `/home/node/.profile` i pobierane przed uruchomieniem testów
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1`, aby weryfikować wyłącznie zmienne env pobrane z `OPENCLAW_PROFILE_FILE`, używając tymczasowych katalogów config/workspace i bez zewnętrznych montowań autoryzacji CLI
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (domyślnie: `~/.cache/openclaw/docker-cli-tools`) montowane do `/home/node/.npm-global` dla buforowanych instalacji CLI wewnątrz Dockera
- Zewnętrzne katalogi/pliki autoryzacji CLI pod `$HOME` są montowane tylko do odczytu pod `/host-auth...`, a następnie kopiowane do `/home/node/...` przed rozpoczęciem testów
  - Domyślne katalogi: `.minimax`
  - Domyślne pliki: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Zawężone uruchomienia dostawców montują tylko potrzebne katalogi/pliki wywnioskowane z `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Nadpisz ręcznie przez `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` lub listę rozdzielaną przecinkami, na przykład `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, aby zawęzić uruchomienie
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, aby filtrować dostawców wewnątrz kontenera
- `OPENCLAW_SKIP_DOCKER_BUILD=1`, aby ponownie użyć istniejącego obrazu `openclaw:local-live` przy ponownych uruchomieniach, które nie wymagają przebudowy
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, aby upewnić się, że poświadczenia pochodzą z magazynu profili (a nie z env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, aby wybrać model udostępniany przez Gateway dla smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...`, aby nadpisać prompt sprawdzania nonce używany przez smoke Open WebUI
- `OPENWEBUI_IMAGE=...`, aby nadpisać przypięty tag obrazu Open WebUI

## Kontrola poprawności dokumentacji

Po edycjach dokumentacji uruchom kontrole docs: `pnpm check:docs`.
Uruchom pełną walidację anchorów Mintlify, gdy potrzebujesz także kontroli nagłówków w obrębie strony: `pnpm docs:check-links:anchors`.

## Regresja offline (bezpieczna dla CI)

To są regresje „rzeczywistego pipeline’u” bez prawdziwych dostawców:

- Wywoływanie narzędzi Gateway (mock OpenAI, rzeczywista pętla Gateway + agenta): `src/gateway/gateway.test.ts` (przypadek: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Kreator Gateway (WS `wizard.start`/`wizard.next`, zapis konfiguracji + wymuszona autoryzacja): `src/gateway/gateway.test.ts` (przypadek: "runs wizard over ws and writes auth token config")

## Ewalucje niezawodności agentów (Skills)

Mamy już kilka testów bezpiecznych dla CI, które działają jak „ewaluacje niezawodności agentów”:

- Mock wywoływania narzędzi przez rzeczywistą pętlę Gateway + agenta (`src/gateway/gateway.test.ts`).
- Pełne przepływy kreatora, które walidują okablowanie sesji i efekty konfiguracji (`src/gateway/gateway.test.ts`).

Czego nadal brakuje dla Skills (zobacz [Skills](/pl/tools/skills)):

- **Podejmowanie decyzji:** gdy Skills są wymienione w promptcie, czy agent wybiera właściwy Skill (albo unika nieistotnych)?
- **Zgodność:** czy agent odczytuje `SKILL.md` przed użyciem i stosuje wymagane kroki/argumenty?
- **Kontrakty przepływu pracy:** scenariusze wieloturowe, które potwierdzają kolejność narzędzi, przenoszenie historii sesji i granice sandboxa.

Przyszłe ewaluacje powinny najpierw pozostać deterministyczne:

- Runner scenariuszy używający mock dostawców do potwierdzania wywołań narzędzi + kolejności, odczytów plików Skill i okablowania sesji.
- Niewielki pakiet scenariuszy skupionych na Skills (użycie vs unikanie, bramkowanie, prompt injection).
- Opcjonalne ewaluacje live (opt-in, bramkowane env) dopiero po wdrożeniu pakietu bezpiecznego dla CI.

## Testy kontraktowe (kształt Plugin i kanałów)

Testy kontraktowe weryfikują, że każdy zarejestrowany Plugin i kanał jest zgodny ze swoim
kontraktem interfejsu. Iterują po wszystkich wykrytych pluginach i uruchamiają pakiet
potwierdzeń kształtu oraz zachowania. Domyślna ścieżka unit `pnpm test` celowo
pomija te współdzielone pliki seam i smoke; uruchamiaj polecenia kontraktowe jawnie,
gdy modyfikujesz współdzielone powierzchnie kanałów lub dostawców.

### Polecenia

- Wszystkie kontrakty: `pnpm test:contracts`
- Tylko kontrakty kanałów: `pnpm test:contracts:channels`
- Tylko kontrakty dostawców: `pnpm test:contracts:plugins`

### Kontrakty kanałów

Znajdują się w `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Podstawowy kształt Plugin (`id`, `name`, możliwości)
- **setup** - Kontrakt kreatora konfiguracji
- **session-binding** - Zachowanie bindowania sesji
- **outbound-payload** - Struktura payloadu wiadomości
- **inbound** - Obsługa wiadomości przychodzących
- **actions** - Handlery akcji kanału
- **threading** - Obsługa identyfikatorów wątków
- **directory** - API katalogu/listy
- **group-policy** - Egzekwowanie polityki grupowej

### Kontrakty statusu dostawców

Znajdują się w `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondy statusu kanału
- **registry** - Kształt rejestru Plugin

### Kontrakty dostawców

Znajdują się w `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Kontrakt przepływu autoryzacji
- **auth-choice** - Wybór/selekcja autoryzacji
- **catalog** - API katalogu modeli
- **discovery** - Wykrywanie Plugin
- **loader** - Ładowanie Plugin
- **runtime** - Runtime dostawcy
- **shape** - Kształt/interfejs Plugin
- **wizard** - Kreator konfiguracji

### Kiedy uruchamiać

- Po zmianie eksportów lub subścieżek plugin-sdk
- Po dodaniu lub modyfikacji kanału albo Plugin dostawcy
- Po refaktoryzacji rejestracji albo wykrywania Plugin

Testy kontraktowe uruchamiają się w CI i nie wymagają prawdziwych kluczy API.

## Dodawanie regresji (wskazówki)

Gdy naprawiasz problem dostawcy/modelu wykryty w live:

- Jeśli to możliwe, dodaj regresję bezpieczną dla CI (mock/stub dostawcy albo przechwyć dokładną transformację kształtu żądania)
- Jeśli z natury jest to problem wyłącznie live (limity szybkości, polityki autoryzacji), utrzymuj test live wąski i opt-in przez zmienne env
- Preferuj celowanie w najmniejszą warstwę, która wychwytuje błąd:
  - błąd konwersji/odtwarzania żądania dostawcy → test modeli bezpośrednich
  - błąd pipeline’u sesji/historii/narzędzi Gateway → smoke Gateway live albo bezpieczny dla CI mock test Gateway
- Guardrail przechodzenia SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` wyprowadza jeden przykładowy cel dla każdej klasy SecretRef z metadanych rejestru (`listSecretTargetRegistryEntries()`), a następnie potwierdza, że identyfikatory exec segmentów przechodzenia są odrzucane.
  - Jeśli dodasz nową rodzinę docelową SecretRef `includeInPlan` w `src/secrets/target-registry-data.ts`, zaktualizuj `classifyTargetClass` w tym teście. Test celowo kończy się niepowodzeniem dla niesklasyfikowanych identyfikatorów docelowych, aby nowych klas nie dało się pominąć po cichu.
