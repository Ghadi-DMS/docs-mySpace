---
read_when:
    - Widzisz ostrzeżenie OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Widzisz ostrzeżenie OPENCLAW_EXTENSION_API_DEPRECATED
    - Aktualizujesz wtyczkę do nowoczesnej architektury wtyczek OpenClaw
    - Utrzymujesz zewnętrzną wtyczkę OpenClaw
sidebarTitle: Migrate to SDK
summary: Migracja ze starszej warstwy zgodności wstecznej do nowoczesnego Plugin SDK
title: Migracja Plugin SDK
x-i18n:
    generated_at: "2026-04-09T01:29:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60cbb6c8be30d17770887d490c14e3a4538563339a5206fb419e51e0558bbc07
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migracja Plugin SDK

OpenClaw przeszedł z szerokiej warstwy zgodności wstecznej na nowoczesną
architekturę wtyczek z ukierunkowanymi, udokumentowanymi importami. Jeśli Twoja wtyczka została zbudowana przed
wprowadzeniem nowej architektury, ten przewodnik pomoże Ci przeprowadzić migrację.

## Co się zmienia

Stary system wtyczek udostępniał dwie bardzo szerokie powierzchnie, które pozwalały wtyczkom importować
wszystko, czego potrzebowały, z jednego punktu wejścia:

- **`openclaw/plugin-sdk/compat`** — pojedynczy import, który reeksportował dziesiątki
  pomocniczych elementów. Został wprowadzony po to, aby starsze wtyczki oparte na hookach nadal działały, gdy
  budowana była nowa architektura wtyczek.
- **`openclaw/extension-api`** — pomost, który dawał wtyczkom bezpośredni dostęp do
  pomocników po stronie hosta, takich jak osadzony runner agenta.

Obie powierzchnie są teraz **przestarzałe**. Nadal działają w środowisku uruchomieniowym, ale nowe
wtyczki nie mogą z nich korzystać, a istniejące wtyczki powinny przeprowadzić migrację przed kolejnym
głównym wydaniem, które je usunie.

<Warning>
  Warstwa zgodności wstecznej zostanie usunięta w jednym z przyszłych głównych wydań.
  Wtyczki, które nadal importują z tych powierzchni, przestaną działać, gdy to nastąpi.
</Warning>

## Dlaczego to się zmieniło

Stare podejście powodowało problemy:

- **Wolne uruchamianie** — zaimportowanie jednego pomocnika ładowało dziesiątki niepowiązanych modułów
- **Zależności cykliczne** — szerokie reeksporty ułatwiały tworzenie cykli importu
- **Niejasna powierzchnia API** — nie było sposobu, aby rozróżnić, które eksporty były stabilne, a które wewnętrzne

Nowoczesne Plugin SDK to naprawia: każda ścieżka importu (`openclaw/plugin-sdk/\<subpath\>`)
jest małym, samodzielnym modułem o jasno określonym celu i udokumentowanym kontrakcie.

Starsze wygodne punkty integracji dostawców dla dołączonych kanałów również zniknęły. Importy
takie jak `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
pomocnicze punkty integracji oznaczone marką kanału oraz
`openclaw/plugin-sdk/telegram-core` były prywatnymi skrótami mono-repo, a nie
stabilnymi kontraktami wtyczek. Zamiast tego używaj wąskich, ogólnych podścieżek SDK. W obrębie
dołączonego obszaru roboczego wtyczek trzymaj pomocniki należące do dostawcy we własnym
`api.ts` lub `runtime-api.ts` tej wtyczki.

Bieżące przykłady dołączonych dostawców:

- Anthropic trzyma pomocniki strumieni specyficzne dla Claude we własnym punkcie integracji `api.ts` /
  `contract-api.ts`
- OpenAI trzyma konstruktory dostawców, pomocniki modeli domyślnych i konstruktory dostawców realtime
  we własnym `api.ts`
- OpenRouter trzyma konstruktor dostawcy oraz pomocniki onboardingu/konfiguracji we własnym
  `api.ts`

## Jak przeprowadzić migrację

<Steps>
  <Step title="Przenieś handlery approval-native do faktów o możliwościach">
    Wtyczki kanałów obsługujące zatwierdzanie udostępniają teraz natywne zachowanie zatwierdzania przez
    `approvalCapability.nativeRuntime` oraz współdzielony rejestr kontekstu runtime.

    Najważniejsze zmiany:

    - Zastąp `approvalCapability.handler.loadRuntime(...)` przez
      `approvalCapability.nativeRuntime`
    - Przenieś uwierzytelnianie/dostarczanie specyficzne dla zatwierdzeń z przestarzałego połączenia `plugin.auth` /
      `plugin.approvals` do `approvalCapability`
    - `ChannelPlugin.approvals` zostało usunięte z publicznego kontraktu
      wtyczek kanałów; przenieś pola delivery/native/render do `approvalCapability`
    - `plugin.auth` pozostaje tylko dla przepływów logowania/wylogowania kanału; hooki
      uwierzytelniania zatwierdzeń w tym miejscu nie są już odczytywane przez core
    - Rejestruj obiekty runtime należące do kanału, takie jak klienci, tokeny czy aplikacje
      Bolt, przez `openclaw/plugin-sdk/channel-runtime-context`
    - Nie wysyłaj komunikatów o przekierowaniu należących do wtyczki z natywnych handlerów zatwierdzeń;
      komunikaty o dostarczeniu gdzie indziej na podstawie rzeczywistych wyników dostarczenia są teraz obsługiwane przez core
    - Przy przekazywaniu `channelRuntime` do `createChannelManager(...)` podaj
      rzeczywistą powierzchnię `createPluginRuntime().channel`. Częściowe stuby są odrzucane.

    Zobacz `/plugins/sdk-channel-plugins`, aby poznać bieżący układ możliwości zatwierdzania.

  </Step>

  <Step title="Sprawdź zachowanie awaryjne wrappera Windows">
    Jeśli Twoja wtyczka używa `openclaw/plugin-sdk/windows-spawn`, nierozwiązane wrappery Windows
    `.cmd`/`.bat` kończą się teraz zamknięciem awaryjnym, chyba że jawnie przekażesz
    `allowShellFallback: true`.

    ```typescript
    // Przed
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Po
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Ustawiaj to tylko dla zaufanych wywołań zgodności, które celowo
      // akceptują awaryjne przejście przez powłokę.
      allowShellFallback: true,
    });
    ```

    Jeśli Twój kod wywołujący nie polega celowo na awaryjnym przejściu przez powłokę, nie ustawiaj
    `allowShellFallback` i zamiast tego obsłuż wyrzucany błąd.

  </Step>

  <Step title="Znajdź przestarzałe importy">
    Wyszukaj w swojej wtyczce importy z którejkolwiek z przestarzałych powierzchni:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Zastąp ukierunkowanymi importami">
    Każdy eksport ze starej powierzchni ma odpowiednik w konkretnej nowoczesnej ścieżce importu:

    ```typescript
    // Przed (przestarzała warstwa zgodności wstecznej)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Po (nowoczesne, ukierunkowane importy)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    W przypadku pomocników po stronie hosta użyj wstrzykniętego runtime wtyczki zamiast importować
    je bezpośrednio:

    ```typescript
    // Przed (przestarzały pomost extension-api)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Po (wstrzyknięty runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Ten sam wzorzec dotyczy innych starszych pomocników pomostu:

    | Stary import | Nowoczesny odpowiednik |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | pomocniki magazynu sesji | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Zbuduj i przetestuj">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Odniesienie do ścieżek importu

<Accordion title="Tabela typowych ścieżek importu">
  | Ścieżka importu | Przeznaczenie | Kluczowe eksporty |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Kanoniczny pomocnik punktu wejścia wtyczki | `definePluginEntry` |
  | `plugin-sdk/core` | Starszy zbiorczy reeksport definicji/budowniczych wejść kanałów | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Eksport głównego schematu konfiguracji | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Pomocnik punktu wejścia pojedynczego dostawcy | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Ukierunkowane definicje i konstruktory wejść kanałów | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Współdzielone pomocniki kreatora konfiguracji | Prompty listy dozwolonych, konstruktory statusu konfiguracji |
  | `plugin-sdk/setup-runtime` | Pomocniki runtime dla czasu konfiguracji | Bezpieczne importowo adaptery łatek konfiguracji, pomocniki notatek wyszukiwania, `promptResolvedAllowFrom`, `splitSetupEntries`, delegowane proxy konfiguracji |
  | `plugin-sdk/setup-adapter-runtime` | Pomocniki adaptera konfiguracji | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Pomocniki narzędzi konfiguracji | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Pomocniki wielu kont | Pomocniki listy kont/konfiguracji/bramek akcji |
  | `plugin-sdk/account-id` | Pomocniki identyfikatora konta | `DEFAULT_ACCOUNT_ID`, normalizacja identyfikatora konta |
  | `plugin-sdk/account-resolution` | Pomocniki wyszukiwania kont | Wyszukiwanie kont + pomocniki awaryjnego użycia wartości domyślnej |
  | `plugin-sdk/account-helpers` | Wąskie pomocniki kont | Pomocniki listy kont/akcji na koncie |
  | `plugin-sdk/channel-setup` | Adaptery kreatora konfiguracji | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Podstawy parowania DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Prefiks odpowiedzi + logika wpisywania | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fabryki adapterów konfiguracji | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Konstruktory schematów konfiguracji | Typy schematów konfiguracji kanału |
  | `plugin-sdk/telegram-command-config` | Pomocniki konfiguracji poleceń Telegram | Normalizacja nazw poleceń, przycinanie opisów, walidacja duplikatów/konfliktów |
  | `plugin-sdk/channel-policy` | Rozstrzyganie polityki grup/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Śledzenie statusu konta | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Pomocniki kopert wejściowych | Współdzielone pomocniki trasy + konstruktora koperty |
  | `plugin-sdk/inbound-reply-dispatch` | Pomocniki odpowiedzi wejściowych | Współdzielone pomocniki zapisu i wysyłki |
  | `plugin-sdk/messaging-targets` | Parsowanie celów wiadomości | Pomocniki parsowania/dopasowywania celów |
  | `plugin-sdk/outbound-media` | Pomocniki mediów wychodzących | Współdzielone ładowanie mediów wychodzących |
  | `plugin-sdk/outbound-runtime` | Pomocniki runtime dla ruchu wychodzącego | Pomocniki tożsamości wychodzącej/delegata wysyłki |
  | `plugin-sdk/thread-bindings-runtime` | Pomocniki powiązań wątków | Cykl życia powiązań wątków i pomocniki adapterów |
  | `plugin-sdk/agent-media-payload` | Starsze pomocniki ładunku mediów | Konstruktor ładunku mediów agenta dla starszych układów pól |
  | `plugin-sdk/channel-runtime` | Przestarzały shim zgodności | Tylko starsze narzędzia runtime kanału |
  | `plugin-sdk/channel-send-result` | Typy wyników wysyłki | Typy wyników odpowiedzi |
  | `plugin-sdk/runtime-store` | Trwałe przechowywanie wtyczek | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Szerokie pomocniki runtime | Pomocniki runtime/logowania/kopii zapasowej/instalacji wtyczek |
  | `plugin-sdk/runtime-env` | Wąskie pomocniki środowiska runtime | Logger/pomocniki środowiska runtime, timeout, retry i backoff |
  | `plugin-sdk/plugin-runtime` | Współdzielone pomocniki runtime wtyczek | Pomocniki poleceń/hooków/http/interaktywnych dla wtyczek |
  | `plugin-sdk/hook-runtime` | Pomocniki potoku hooków | Współdzielone pomocniki potoku webhooków/wewnętrznych hooków |
  | `plugin-sdk/lazy-runtime` | Pomocniki leniwego runtime | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Pomocniki procesów | Współdzielone pomocniki `exec` |
  | `plugin-sdk/cli-runtime` | Pomocniki runtime CLI | Formatowanie poleceń, oczekiwania, pomocniki wersji |
  | `plugin-sdk/gateway-runtime` | Pomocniki bramy | Klient bramy i pomocniki łatek statusu kanału |
  | `plugin-sdk/config-runtime` | Pomocniki konfiguracji | Pomocniki ładowania/zapisu konfiguracji |
  | `plugin-sdk/telegram-command-config` | Pomocniki poleceń Telegram | Pomocniki walidacji poleceń Telegram ze stabilnym fallbackiem, gdy powierzchnia kontraktu dołączonego Telegrama jest niedostępna |
  | `plugin-sdk/approval-runtime` | Pomocniki promptów zatwierdzania | Ładunek zatwierdzania exec/wtyczki, pomocniki możliwości/profili zatwierdzania, natywne pomocniki routingu/runtime zatwierdzania |
  | `plugin-sdk/approval-auth-runtime` | Pomocniki uwierzytelniania zatwierdzeń | Rozstrzyganie zatwierdzających, uwierzytelnianie akcji w tym samym czacie |
  | `plugin-sdk/approval-client-runtime` | Pomocniki klienta zatwierdzeń | Pomocniki profilu/filtrów natywnego zatwierdzania exec |
  | `plugin-sdk/approval-delivery-runtime` | Pomocniki dostarczania zatwierdzeń | Adaptery możliwości/dostarczania natywnych zatwierdzeń |
  | `plugin-sdk/approval-gateway-runtime` | Pomocniki bramy zatwierdzeń | Współdzielony pomocnik rozstrzygania bramy zatwierdzeń |
  | `plugin-sdk/approval-handler-adapter-runtime` | Pomocniki adaptera zatwierdzeń | Lekkie pomocniki ładowania natywnych adapterów zatwierdzania dla gorących punktów wejścia kanału |
  | `plugin-sdk/approval-handler-runtime` | Pomocniki handlera zatwierdzeń | Szersze pomocniki runtime handlera zatwierdzeń; preferuj węższe punkty integracji adaptera/bramy, gdy są wystarczające |
  | `plugin-sdk/approval-native-runtime` | Pomocniki celu zatwierdzeń | Pomocniki natywnego wiązania celu/konta dla zatwierdzeń |
  | `plugin-sdk/approval-reply-runtime` | Pomocniki odpowiedzi zatwierdzeń | Pomocniki ładunku odpowiedzi zatwierdzeń exec/wtyczki |
  | `plugin-sdk/channel-runtime-context` | Pomocniki kontekstu runtime kanału | Ogólne pomocniki register/get/watch dla kontekstu runtime kanału |
  | `plugin-sdk/security-runtime` | Pomocniki bezpieczeństwa | Współdzielone pomocniki zaufania, bramek DM, treści zewnętrznych i zbierania sekretów |
  | `plugin-sdk/ssrf-policy` | Pomocniki polityki SSRF | Pomocniki listy dozwolonych hostów i polityki sieci prywatnej |
  | `plugin-sdk/ssrf-runtime` | Pomocniki runtime SSRF | Przypięty dispatcher, strzeżony fetch, pomocniki polityki SSRF |
  | `plugin-sdk/collection-runtime` | Pomocniki ograniczonego cache | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Pomocniki bramek diagnostycznych | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Pomocniki formatowania błędów | `formatUncaughtError`, `isApprovalNotFoundError`, pomocniki grafu błędów |
  | `plugin-sdk/fetch-runtime` | Pomocniki opakowanego fetch/proxy | `resolveFetch`, pomocniki proxy |
  | `plugin-sdk/host-runtime` | Pomocniki normalizacji hosta | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Pomocniki ponawiania | `RetryConfig`, `retryAsync`, uruchamiacze polityk |
  | `plugin-sdk/allow-from` | Formatowanie listy dozwolonych | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapowanie wejść listy dozwolonych | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Bramkowanie poleceń i pomocniki powierzchni poleceń | `resolveControlCommandGate`, pomocniki autoryzacji nadawcy, pomocniki rejestru poleceń |
  | `plugin-sdk/command-status` | Renderery statusu/pomocy poleceń | `buildCommandsMessage`, `buildCommandsMessagePaginated`, `buildHelpMessage` |
  | `plugin-sdk/secret-input` | Parsowanie wejść sekretów | Pomocniki wejść sekretów |
  | `plugin-sdk/webhook-ingress` | Pomocniki żądań webhooków | Narzędzia celu webhooka |
  | `plugin-sdk/webhook-request-guards` | Pomocniki guardów żądań webhooków | Pomocniki odczytu/limitów treści żądania |
  | `plugin-sdk/reply-runtime` | Współdzielony runtime odpowiedzi | Wysyłka wejściowa, heartbeat, planista odpowiedzi, dzielenie na części |
  | `plugin-sdk/reply-dispatch-runtime` | Wąskie pomocniki wysyłki odpowiedzi | Pomocniki finalizacji + wysyłki dostawcy |
  | `plugin-sdk/reply-history` | Pomocniki historii odpowiedzi | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planowanie odwołań odpowiedzi | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Pomocniki dzielenia odpowiedzi | Pomocniki dzielenia tekstu/markdownu |
  | `plugin-sdk/session-store-runtime` | Pomocniki magazynu sesji | Ścieżka magazynu + pomocniki `updated-at` |
  | `plugin-sdk/state-paths` | Pomocniki ścieżek stanu | Pomocniki katalogów stanu i OAuth |
  | `plugin-sdk/routing` | Pomocniki routingu/klucza sesji | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, pomocniki normalizacji klucza sesji |
  | `plugin-sdk/status-helpers` | Pomocniki statusu kanału | Konstruktory podsumowań statusu kanału/konta, domyślne wartości stanu runtime, pomocniki metadanych problemów |
  | `plugin-sdk/target-resolver-runtime` | Pomocniki rozstrzygania celu | Współdzielone pomocniki rozstrzygania celu |
  | `plugin-sdk/string-normalization-runtime` | Pomocniki normalizacji ciągów | Pomocniki normalizacji slugów/ciągów |
  | `plugin-sdk/request-url` | Pomocniki URL żądania | Wyodrębnianie tekstowych URL z danych wejściowych podobnych do żądania |
  | `plugin-sdk/run-command` | Pomocniki poleceń z limitem czasu | Uruchamiacz poleceń z znormalizowanym stdout/stderr |
  | `plugin-sdk/param-readers` | Odczyt parametrów | Typowe odczyty parametrów narzędzi/CLI |
  | `plugin-sdk/tool-payload` | Wyodrębnianie ładunku narzędzia | Wyodrębnianie znormalizowanych ładunków z obiektów wyników narzędzi |
  | `plugin-sdk/tool-send` | Wyodrębnianie wysyłki narzędzia | Wyodrębnianie kanonicznych pól celu wysyłki z argumentów narzędzia |
  | `plugin-sdk/temp-path` | Pomocniki ścieżek tymczasowych | Współdzielone pomocniki ścieżek tymczasowego pobierania |
  | `plugin-sdk/logging-core` | Pomocniki logowania | Logger podsystemu i pomocniki redakcji danych |
  | `plugin-sdk/markdown-table-runtime` | Pomocniki tabel markdown | Pomocniki trybów tabel markdown |
  | `plugin-sdk/reply-payload` | Typy odpowiedzi wiadomości | Typy ładunku odpowiedzi |
  | `plugin-sdk/provider-setup` | Kuratorowane pomocniki konfiguracji lokalnych/samohostowanych dostawców | Pomocniki wykrywania/konfiguracji samohostowanych dostawców |
  | `plugin-sdk/self-hosted-provider-setup` | Ukierunkowane pomocniki konfiguracji samohostowanych dostawców zgodnych z OpenAI | Te same pomocniki wykrywania/konfiguracji samohostowanych dostawców |
  | `plugin-sdk/provider-auth-runtime` | Pomocniki uwierzytelniania runtime dostawców | Pomocniki runtime rozstrzygania klucza API |
  | `plugin-sdk/provider-auth-api-key` | Pomocniki konfiguracji klucza API dostawcy | Pomocniki onboardingu/zapisu profilu dla klucza API |
  | `plugin-sdk/provider-auth-result` | Pomocniki wyniku uwierzytelnienia dostawcy | Standardowy konstruktor wyniku uwierzytelnienia OAuth |
  | `plugin-sdk/provider-auth-login` | Pomocniki interaktywnego logowania dostawcy | Współdzielone pomocniki logowania interaktywnego |
  | `plugin-sdk/provider-env-vars` | Pomocniki zmiennych środowiskowych dostawcy | Pomocniki wyszukiwania zmiennych środowiskowych uwierzytelniania dostawcy |
  | `plugin-sdk/provider-model-shared` | Współdzielone pomocniki modeli/odtwarzania dostawców | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, współdzielone konstruktory polityk odtwarzania, pomocniki endpointów dostawców oraz normalizacja identyfikatorów modeli |
  | `plugin-sdk/provider-catalog-shared` | Współdzielone pomocniki katalogu dostawców | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Łatki onboardingu dostawców | Pomocniki konfiguracji onboardingu |
  | `plugin-sdk/provider-http` | Pomocniki HTTP dostawców | Ogólne pomocniki HTTP/zdolności endpointów dostawców |
  | `plugin-sdk/provider-web-fetch` | Pomocniki web-fetch dostawców | Pomocniki rejestracji/cache dostawcy web-fetch |
  | `plugin-sdk/provider-web-search-config-contract` | Pomocniki konfiguracji wyszukiwania w sieci dla dostawców | Wąskie pomocniki konfiguracji/poświadczeń wyszukiwania w sieci dla dostawców, którzy nie potrzebują logiki włączania wtyczki |
  | `plugin-sdk/provider-web-search-contract` | Pomocniki kontraktu wyszukiwania w sieci dla dostawców | Wąskie pomocniki kontraktu konfiguracji/poświadczeń wyszukiwania w sieci, takie jak `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` oraz funkcje ustawiania/pobierania poświadczeń o ograniczonym zakresie |
  | `plugin-sdk/provider-web-search` | Pomocniki wyszukiwania w sieci dla dostawców | Pomocniki rejestracji/cache/runtime dostawcy wyszukiwania w sieci |
  | `plugin-sdk/provider-tools` | Pomocniki zgodności narzędzi/schematów dostawców | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, czyszczenie schematu Gemini + diagnostyka oraz pomocniki zgodności xAI, takie jak `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Pomocniki użycia dostawców | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` i inne pomocniki użycia dostawców |
  | `plugin-sdk/provider-stream` | Pomocniki opakowań strumieni dostawców | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, typy opakowań strumieni oraz współdzielone opakowania Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Uporządkowana kolejka async | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Współdzielone pomocniki mediów | Pomocniki pobierania/przekształcania/przechowywania mediów oraz konstruktory ładunków mediów |
  | `plugin-sdk/media-generation-runtime` | Współdzielone pomocniki generowania mediów | Współdzielone pomocniki failover, wyboru kandydatów i komunikatów o brakujących modelach dla generowania obrazów/wideo/muzyki |
  | `plugin-sdk/media-understanding` | Pomocniki media-understanding | Typy dostawców media-understanding oraz eksporty pomocników obrazów/audio dla dostawców |
  | `plugin-sdk/text-runtime` | Współdzielone pomocniki tekstowe | Usuwanie tekstu widocznego dla asystenta, renderowanie/dzielenie/tabele markdown, pomocniki redakcji, pomocniki tagów dyrektyw, bezpieczny tekst i powiązane pomocniki tekstu/logowania |
  | `plugin-sdk/text-chunking` | Pomocniki dzielenia tekstu | Pomocnik dzielenia tekstu wychodzącego |
  | `plugin-sdk/speech` | Pomocniki mowy | Typy dostawców mowy oraz eksporty pomocników dyrektyw, rejestru i walidacji dla dostawców |
  | `plugin-sdk/speech-core` | Współdzielony rdzeń mowy | Typy dostawców mowy, rejestr, dyrektywy, normalizacja |
  | `plugin-sdk/realtime-transcription` | Pomocniki transkrypcji realtime | Typy dostawców i pomocniki rejestru |
  | `plugin-sdk/realtime-voice` | Pomocniki głosu realtime | Typy dostawców i pomocniki rejestru |
  | `plugin-sdk/image-generation-core` | Współdzielony rdzeń generowania obrazów | Typy generowania obrazów, failover, uwierzytelnianie i pomocniki rejestru |
  | `plugin-sdk/music-generation` | Pomocniki generowania muzyki | Typy dostawcy/żądania/wyniku generowania muzyki |
  | `plugin-sdk/music-generation-core` | Współdzielony rdzeń generowania muzyki | Typy generowania muzyki, pomocniki failover, wyszukiwanie dostawcy i parsowanie model-ref |
  | `plugin-sdk/video-generation` | Pomocniki generowania wideo | Typy dostawcy/żądania/wyniku generowania wideo |
  | `plugin-sdk/video-generation-core` | Współdzielony rdzeń generowania wideo | Typy generowania wideo, pomocniki failover, wyszukiwanie dostawcy i parsowanie model-ref |
  | `plugin-sdk/interactive-runtime` | Pomocniki odpowiedzi interaktywnych | Normalizacja/redukcja ładunku odpowiedzi interaktywnych |
  | `plugin-sdk/channel-config-primitives` | Prymitywy konfiguracji kanału | Wąskie prymitywy schematu konfiguracji kanału |
  | `plugin-sdk/channel-config-writes` | Pomocniki zapisów konfiguracji kanału | Pomocniki autoryzacji zapisu konfiguracji kanału |
  | `plugin-sdk/channel-plugin-common` | Współdzielone preludium kanału | Eksporty współdzielonego preludium wtyczki kanału |
  | `plugin-sdk/channel-status` | Pomocniki statusu kanału | Współdzielone pomocniki migawek/podsumowań statusu kanału |
  | `plugin-sdk/allowlist-config-edit` | Pomocniki konfiguracji listy dozwolonych | Pomocniki edycji/odczytu konfiguracji listy dozwolonych |
  | `plugin-sdk/group-access` | Pomocniki dostępu grupowego | Współdzielone pomocniki decyzji o dostępie grupowym |
  | `plugin-sdk/direct-dm` | Pomocniki bezpośrednich DM | Współdzielone pomocniki uwierzytelniania/guardów bezpośrednich DM |
  | `plugin-sdk/extension-shared` | Współdzielone pomocniki rozszerzeń | Prymitywy pomocników pasywnego kanału/statusu i proxy otoczenia |
  | `plugin-sdk/webhook-targets` | Pomocniki celów webhooków | Rejestr celów webhooków i pomocniki instalacji tras |
  | `plugin-sdk/webhook-path` | Pomocniki ścieżek webhooków | Pomocniki normalizacji ścieżek webhooków |
  | `plugin-sdk/web-media` | Współdzielone pomocniki mediów webowych | Pomocniki ładowania mediów zdalnych/lokalnych |
  | `plugin-sdk/zod` | Reeksport zod | Reeksport `zod` dla odbiorców Plugin SDK |
  | `plugin-sdk/memory-core` | Pomocniki dołączonego memory-core | Powierzchnia pomocników menedżera pamięci/konfiguracji/plików/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Fasada runtime silnika pamięci | Fasada runtime indeksowania/wyszukiwania pamięci |
  | `plugin-sdk/memory-core-host-engine-foundation` | Hostowy silnik podstawowy pamięci | Eksporty hostowego silnika podstawowego pamięci |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Hostowy silnik embeddingów pamięci | Eksporty hostowego silnika embeddingów pamięci |
  | `plugin-sdk/memory-core-host-engine-qmd` | Hostowy silnik QMD pamięci | Eksporty hostowego silnika QMD pamięci |
  | `plugin-sdk/memory-core-host-engine-storage` | Hostowy silnik przechowywania pamięci | Eksporty hostowego silnika przechowywania pamięci |
  | `plugin-sdk/memory-core-host-multimodal` | Pomocniki multimodalne hosta pamięci | Pomocniki multimodalne hosta pamięci |
  | `plugin-sdk/memory-core-host-query` | Pomocniki zapytań hosta pamięci | Pomocniki zapytań hosta pamięci |
  | `plugin-sdk/memory-core-host-secret` | Pomocniki sekretów hosta pamięci | Pomocniki sekretów hosta pamięci |
  | `plugin-sdk/memory-core-host-events` | Pomocniki dziennika zdarzeń hosta pamięci | Pomocniki dziennika zdarzeń hosta pamięci |
  | `plugin-sdk/memory-core-host-status` | Pomocniki statusu hosta pamięci | Pomocniki statusu hosta pamięci |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI hosta pamięci | Pomocniki runtime CLI hosta pamięci |
  | `plugin-sdk/memory-core-host-runtime-core` | Główny runtime hosta pamięci | Pomocniki głównego runtime hosta pamięci |
  | `plugin-sdk/memory-core-host-runtime-files` | Pomocniki plików/runtime hosta pamięci | Pomocniki plików/runtime hosta pamięci |
  | `plugin-sdk/memory-host-core` | Alias głównego runtime hosta pamięci | Neutralny względem dostawcy alias pomocników głównego runtime hosta pamięci |
  | `plugin-sdk/memory-host-events` | Alias dziennika zdarzeń hosta pamięci | Neutralny względem dostawcy alias pomocników dziennika zdarzeń hosta pamięci |
  | `plugin-sdk/memory-host-files` | Alias plików/runtime hosta pamięci | Neutralny względem dostawcy alias pomocników plików/runtime hosta pamięci |
  | `plugin-sdk/memory-host-markdown` | Pomocniki zarządzanego markdownu | Współdzielone pomocniki zarządzanego markdownu dla wtyczek powiązanych z pamięcią |
  | `plugin-sdk/memory-host-search` | Fasada wyszukiwania aktywnej pamięci | Leniwa fasada runtime menedżera wyszukiwania aktywnej pamięci |
  | `plugin-sdk/memory-host-status` | Alias statusu hosta pamięci | Neutralny względem dostawcy alias pomocników statusu hosta pamięci |
  | `plugin-sdk/memory-lancedb` | Pomocniki dołączonego memory-lancedb | Powierzchnia pomocników memory-lancedb |
  | `plugin-sdk/testing` | Narzędzia testowe | Pomocniki testowe i mocki |
</Accordion>

Ta tabela celowo obejmuje typowy podzbiór do migracji, a nie pełną powierzchnię
SDK. Pełna lista ponad 200 punktów wejścia znajduje się w
`scripts/lib/plugin-sdk-entrypoints.json`.

Ta lista nadal zawiera niektóre pomocnicze punkty integracji dołączonych wtyczek, takie jak
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` oraz `plugin-sdk/matrix*`. Nadal są one eksportowane na potrzeby
utrzymania dołączonych wtyczek i zgodności, ale celowo
pominięto je w tabeli typowej migracji i nie są zalecanym celem dla
nowego kodu wtyczek.

Ta sama zasada dotyczy innych rodzin dołączonych pomocników, takich jak:

- pomocniki obsługi przeglądarki: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- powierzchnie dołączonych pomocników/wtyczek, takie jak `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` oraz `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` obecnie udostępnia wąską
powierzchnię pomocników tokenów `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` oraz `resolveCopilotApiToken`.

Używaj najwęższego importu, który pasuje do zadania. Jeśli nie możesz znaleźć eksportu,
sprawdź źródło w `src/plugin-sdk/` albo zapytaj na Discord.

## Harmonogram usunięcia

| Kiedy | Co się stanie |
| ---------------------- | ----------------------------------------------------------------------- |
| **Teraz** | Przestarzałe powierzchnie emitują ostrzeżenia runtime |
| **Następne główne wydanie** | Przestarzałe powierzchnie zostaną usunięte; wtyczki, które nadal z nich korzystają, przestaną działać |

Wszystkie główne wtyczki zostały już zmigrowane. Zewnętrzne wtyczki powinny przeprowadzić
migrację przed następnym głównym wydaniem.

## Tymczasowe wyciszanie ostrzeżeń

Ustaw te zmienne środowiskowe podczas pracy nad migracją:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

To tymczasowe wyjście awaryjne, a nie trwałe rozwiązanie.

## Powiązane

- [Pierwsze kroki](/pl/plugins/building-plugins) — zbuduj swoją pierwszą wtyczkę
- [Przegląd SDK](/pl/plugins/sdk-overview) — pełne odniesienie do importów podścieżek
- [Wtyczki kanałów](/pl/plugins/sdk-channel-plugins) — tworzenie wtyczek kanałów
- [Wtyczki dostawców](/pl/plugins/sdk-provider-plugins) — tworzenie wtyczek dostawców
- [Wewnętrzne elementy wtyczek](/pl/plugins/architecture) — szczegółowe omówienie architektury
- [Manifest wtyczki](/pl/plugins/manifest) — odniesienie do schematu manifestu
