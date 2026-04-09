---
read_when:
    - Musisz wiedzieć, z której ścieżki podrzędnej SDK importować
    - Chcesz mieć dokumentację wszystkich metod rejestracji w OpenClawPluginApi
    - Szukasz konkretnego eksportu SDK
sidebarTitle: SDK Overview
summary: Mapa importów, dokumentacja API rejestracji i architektura SDK
title: Przegląd Plugin SDK
x-i18n:
    generated_at: "2026-04-09T01:30:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf205af060971931df97dca4af5110ce173d2b7c12f56ad7c62d664a402f2381
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Przegląd Plugin SDK

Plugin SDK to typowany kontrakt między pluginami a rdzeniem. Ta strona jest
dokumentacją referencyjną dla **tego, co importować** i **co można rejestrować**.

<Tip>
  **Szukasz przewodnika krok po kroku?**
  - Pierwszy plugin? Zacznij od [Pierwsze kroki](/pl/plugins/building-plugins)
  - Plugin kanału? Zobacz [Pluginy kanałów](/pl/plugins/sdk-channel-plugins)
  - Plugin dostawcy? Zobacz [Pluginy dostawców](/pl/plugins/sdk-provider-plugins)
</Tip>

## Konwencja importów

Zawsze importuj z konkretnej ścieżki podrzędnej:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Każda ścieżka podrzędna jest małym, samowystarczalnym modułem. Dzięki temu uruchamianie pozostaje szybkie i
zapobiega to problemom z zależnościami cyklicznymi. W przypadku helperów wejścia/budowania specyficznych dla kanałów
preferuj `openclaw/plugin-sdk/channel-core`; `openclaw/plugin-sdk/core` zachowaj dla
szerszej powierzchni zbiorczej i współdzielonych helperów, takich jak
`buildChannelConfigSchema`.

Nie dodawaj ani nie polegaj na wygodnych warstwach nazwanych od dostawców, takich jak
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` ani
helperowych warstwach oznaczonych marką kanału. Bundlowane pluginy powinny składać ogólne
ścieżki podrzędne SDK we własnych barrelach `api.ts` lub `runtime-api.ts`, a rdzeń
powinien albo używać tych lokalnych barrelów pluginu, albo dodać wąski ogólny kontrakt SDK,
gdy potrzeba rzeczywiście dotyczy wielu kanałów.

Wygenerowana mapa eksportów nadal zawiera mały zestaw helperowych warstw bundlowanych pluginów,
takich jak `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` i `plugin-sdk/matrix*`. Te
ścieżki podrzędne istnieją wyłącznie dla utrzymania i zgodności bundlowanych pluginów; celowo pominięto je
w standardowej tabeli poniżej i nie są zalecaną ścieżką importu dla nowych pluginów zewnętrznych.

## Dokumentacja ścieżek podrzędnych

Najczęściej używane ścieżki podrzędne, pogrupowane według przeznaczenia. Wygenerowana pełna lista
ponad 200 ścieżek podrzędnych znajduje się w `scripts/lib/plugin-sdk-entrypoints.json`.

Zarezerwowane helperowe ścieżki podrzędne bundlowanych pluginów nadal pojawiają się na tej wygenerowanej liście.
Traktuj je jako szczegóły implementacyjne/powierzchnie zgodności, chyba że strona dokumentacji
wyraźnie promuje którąś z nich jako publiczną.

### Wejście pluginu

| Subpath                     | Key exports                                                                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Ścieżki podrzędne kanałów">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Eksport głównego schematu Zod `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, a także `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Współdzielone helpery kreatora konfiguracji, prompty allowlist i konstruktory statusu konfiguracji |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpery konfiguracji/wymuszania działań dla wielu kont oraz helpery zapasowego użycia konta domyślnego |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpery normalizacji identyfikatora konta |
    | `plugin-sdk/account-resolution` | Helpery wyszukiwania konta i zapasowego użycia wartości domyślnej |
    | `plugin-sdk/account-helpers` | Wąskie helpery list działań/kont |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Typy schematu konfiguracji kanału |
    | `plugin-sdk/telegram-command-config` | Helpery normalizacji/walidacji niestandardowych poleceń Telegram z zapasową obsługą kontraktu bundlowanego |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Współdzielone helpery tras przychodzących i konstruktora kopert |
    | `plugin-sdk/inbound-reply-dispatch` | Współdzielone helpery zapisu i wysyłania odpowiedzi przychodzących |
    | `plugin-sdk/messaging-targets` | Helpery parsowania/dopasowywania celów |
    | `plugin-sdk/outbound-media` | Współdzielone helpery ładowania mediów wychodzących |
    | `plugin-sdk/outbound-runtime` | Helpery tożsamości wychodzącej i delegowania wysyłki |
    | `plugin-sdk/thread-bindings-runtime` | Helpery cyklu życia i adapterów wiązań wątków |
    | `plugin-sdk/agent-media-payload` | Starszy konstruktor ładunku multimediów agenta |
    | `plugin-sdk/conversation-runtime` | Helpery wiązania rozmów/wątków, parowania i skonfigurowanych wiązań |
    | `plugin-sdk/runtime-config-snapshot` | Helper migawki konfiguracji runtime |
    | `plugin-sdk/runtime-group-policy` | Helpery rozstrzygania polityki grup runtime |
    | `plugin-sdk/channel-status` | Współdzielone helpery migawek/podsumowań statusu kanału |
    | `plugin-sdk/channel-config-primitives` | Wąskie prymitywy schematu konfiguracji kanału |
    | `plugin-sdk/channel-config-writes` | Helpery autoryzacji zapisu konfiguracji kanału |
    | `plugin-sdk/channel-plugin-common` | Współdzielone eksporty preludium pluginu kanału |
    | `plugin-sdk/allowlist-config-edit` | Helpery odczytu/edycji konfiguracji allowlist |
    | `plugin-sdk/group-access` | Współdzielone helpery decyzji dostępu do grup |
    | `plugin-sdk/direct-dm` | Współdzielone helpery auth i ochrony direct-DM |
    | `plugin-sdk/interactive-runtime` | Helpery normalizacji/redukcji ładunków odpowiedzi interaktywnych |
    | `plugin-sdk/channel-inbound` | Helpery debounce wiadomości przychodzących, dopasowania mention, polityki mention i kopert |
    | `plugin-sdk/channel-send-result` | Typy wyników odpowiedzi |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpery parsowania/dopasowywania celów |
    | `plugin-sdk/channel-contract` | Typy kontraktu kanału |
    | `plugin-sdk/channel-feedback` | Okablowanie feedbacku/reakcji |
    | `plugin-sdk/channel-secret-runtime` | Wąskie helpery kontraktu sekretów, takie jak `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, oraz typy celów sekretów |
  </Accordion>

  <Accordion title="Ścieżki podrzędne dostawców">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Wyselekcjonowane helpery konfiguracji lokalnych/samohostowanych dostawców |
    | `plugin-sdk/self-hosted-provider-setup` | Ukierunkowane helpery konfiguracji samohostowanego dostawcy zgodnego z OpenAI |
    | `plugin-sdk/cli-backend` | Domyślne wartości backendu CLI i stałe watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpery runtime do rozstrzygania kluczy API dla pluginów dostawców |
    | `plugin-sdk/provider-auth-api-key` | Helpery onboardingu/zapisu profilu klucza API, takie jak `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Standardowy konstruktor wyniku auth OAuth |
    | `plugin-sdk/provider-auth-login` | Współdzielone interaktywne helpery logowania dla pluginów dostawców |
    | `plugin-sdk/provider-env-vars` | Helpery wyszukiwania zmiennych env auth dostawcy |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, współdzielone konstruktory polityki replay, helpery endpointów dostawców oraz helpery normalizacji identyfikatorów modeli, takie jak `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Ogólne helpery możliwości HTTP/endpointów dostawców |
    | `plugin-sdk/provider-web-fetch-contract` | Wąskie helpery kontraktu konfiguracji/wyboru web-fetch, takie jak `enablePluginInConfig` i `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpery rejestracji/cache dostawców web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Wąskie helpery konfiguracji/poświadczeń web search dla dostawców, którzy nie potrzebują okablowania włączania pluginu |
    | `plugin-sdk/provider-web-search-contract` | Wąskie helpery kontraktu konfiguracji/poświadczeń web search, takie jak `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` oraz zakresowane settery/gettery poświadczeń |
    | `plugin-sdk/provider-web-search` | Helpery rejestracji/cache/runtime dostawców web search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, czyszczenie schematu Gemini + diagnostyka oraz helpery zgodności xAI, takie jak `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` i podobne |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, typy wrapperów strumieni oraz współdzielone helpery wrapperów Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpery łatania konfiguracji onboardingu |
    | `plugin-sdk/global-singleton` | Helpery singletonów/map/cache lokalnych dla procesu |
  </Accordion>

  <Accordion title="Ścieżki podrzędne auth i bezpieczeństwa">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpery rejestru poleceń, helpery autoryzacji nadawcy |
    | `plugin-sdk/command-status` | Konstruktory wiadomości poleceń/pomocy, takie jak `buildCommandsMessagePaginated` i `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Helpery rozstrzygania approvera i auth działań w tym samym czacie |
    | `plugin-sdk/approval-client-runtime` | Natywne helpery profili/filtrów zatwierdzania exec |
    | `plugin-sdk/approval-delivery-runtime` | Natywne adaptery możliwości/dostarczania zatwierdzeń |
    | `plugin-sdk/approval-gateway-runtime` | Współdzielony helper rozstrzygania gateway zatwierdzeń |
    | `plugin-sdk/approval-handler-adapter-runtime` | Lekkie helpery ładowania natywnych adapterów zatwierdzeń dla gorących punktów wejścia kanałów |
    | `plugin-sdk/approval-handler-runtime` | Szersze helpery runtime obsługi zatwierdzeń; gdy wystarczą, preferuj węższe warstwy adapter/gateway |
    | `plugin-sdk/approval-native-runtime` | Natywne helpery celów zatwierdzeń i wiązania kont |
    | `plugin-sdk/approval-reply-runtime` | Helpery ładunku odpowiedzi zatwierdzeń exec/pluginów |
    | `plugin-sdk/command-auth-native` | Natywne helpery auth poleceń i natywnych celów sesji |
    | `plugin-sdk/command-detection` | Współdzielone helpery wykrywania poleceń |
    | `plugin-sdk/command-surface` | Helpery normalizacji treści poleceń i powierzchni poleceń |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Wąskie helpery zbierania kontraktów sekretów dla powierzchni sekretów kanałów/pluginów |
    | `plugin-sdk/secret-ref-runtime` | Wąskie helpery `coerceSecretRef` i typowania SecretRef dla parsowania kontraktów sekretów/konfiguracji |
    | `plugin-sdk/security-runtime` | Współdzielone helpery zaufania, bramkowania DM, treści zewnętrznych i zbierania sekretów |
    | `plugin-sdk/ssrf-policy` | Helpery allowlist hostów i polityki SSRF dla sieci prywatnych |
    | `plugin-sdk/ssrf-runtime` | Helpery pinned-dispatcher, fetch chronionego przez SSRF i polityki SSRF |
    | `plugin-sdk/secret-input` | Helpery parsowania wejść sekretów |
    | `plugin-sdk/webhook-ingress` | Helpery żądań/celów webhooków |
    | `plugin-sdk/webhook-request-guards` | Helpery rozmiaru ciała żądania/timeoutu |
  </Accordion>

  <Accordion title="Ścieżki podrzędne runtime i przechowywania">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/runtime` | Szerokie helpery runtime/logowania/kopii zapasowych/instalacji pluginów |
    | `plugin-sdk/runtime-env` | Wąskie helpery env runtime, loggera, timeoutu, retry i backoff |
    | `plugin-sdk/channel-runtime-context` | Ogólne helpery rejestracji i wyszukiwania kontekstu runtime kanałów |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Współdzielone helpery poleceń/hooków/http/interaktywności pluginów |
    | `plugin-sdk/hook-runtime` | Współdzielone helpery potoku webhooków/wewnętrznych hooków |
    | `plugin-sdk/lazy-runtime` | Helpery leniwego importu/powiązań runtime, takie jak `createLazyRuntimeModule`, `createLazyRuntimeMethod` i `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpery wykonywania procesów |
    | `plugin-sdk/cli-runtime` | Helpery formatowania CLI, oczekiwania i wersji |
    | `plugin-sdk/gateway-runtime` | Helpery klienta Gateway i łatania statusu kanałów |
    | `plugin-sdk/config-runtime` | Helpery ładowania/zapisu konfiguracji |
    | `plugin-sdk/telegram-command-config` | Helpery normalizacji nazw/opisów poleceń Telegram oraz sprawdzania duplikatów/konfliktów, nawet gdy bundlowana powierzchnia kontraktu Telegram jest niedostępna |
    | `plugin-sdk/approval-runtime` | Helpery zatwierdzeń exec/pluginów, konstruktory możliwości zatwierdzeń, helpery auth/profili, natywne helpery routingu/runtime |
    | `plugin-sdk/reply-runtime` | Współdzielone helpery runtime dla wejścia/odpowiedzi, chunking, wysyłka, heartbeat, planer odpowiedzi |
    | `plugin-sdk/reply-dispatch-runtime` | Wąskie helpery wysyłki/finalizacji odpowiedzi |
    | `plugin-sdk/reply-history` | Współdzielone helpery krótkookresowej historii odpowiedzi, takie jak `buildHistoryContext`, `recordPendingHistoryEntry` i `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Wąskie helpery chunkingu tekstu/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpery ścieżek updated-at i magazynu sesji |
    | `plugin-sdk/state-paths` | Helpery ścieżek katalogów stanu/OAuth |
    | `plugin-sdk/routing` | Helpery tras/kluczy sesji/wiązania kont, takie jak `resolveAgentRoute`, `buildAgentSessionKey` i `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Współdzielone helpery podsumowania statusu kanału/konta, domyślne stany runtime i helpery metadanych problemów |
    | `plugin-sdk/target-resolver-runtime` | Współdzielone helpery rozstrzygania celów |
    | `plugin-sdk/string-normalization-runtime` | Helpery normalizacji slugów/ciągów |
    | `plugin-sdk/request-url` | Wyodrębniaj URL-e tekstowe z danych wejściowych podobnych do fetch/request |
    | `plugin-sdk/run-command` | Uruchamianie poleceń z limitem czasu i znormalizowanymi wynikami stdout/stderr |
    | `plugin-sdk/param-readers` | Typowe czytniki parametrów narzędzi/CLI |
    | `plugin-sdk/tool-payload` | Wyodrębnia znormalizowane ładunki z obiektów wyników narzędzi |
    | `plugin-sdk/tool-send` | Wyodrębnia kanoniczne pola celu wysyłki z argumentów narzędzi |
    | `plugin-sdk/temp-path` | Współdzielone helpery ścieżek tymczasowego pobierania |
    | `plugin-sdk/logging-core` | Helpery loggera subsystemu i redakcji |
    | `plugin-sdk/markdown-table-runtime` | Helpery trybu tabel Markdown |
    | `plugin-sdk/json-store` | Małe helpery odczytu/zapisu stanu JSON |
    | `plugin-sdk/file-lock` | Helpery reentrant file-lock |
    | `plugin-sdk/persistent-dedupe` | Helpery cache deduplikacji opartej na dysku |
    | `plugin-sdk/acp-runtime` | Helpery ACP runtime/sesji i wysyłki odpowiedzi |
    | `plugin-sdk/agent-config-primitives` | Wąskie prymitywy schematu konfiguracji runtime agenta |
    | `plugin-sdk/boolean-param` | Luźny czytnik parametrów boolean |
    | `plugin-sdk/dangerous-name-runtime` | Helpery rozstrzygania dopasowania niebezpiecznych nazw |
    | `plugin-sdk/device-bootstrap` | Helpery bootstrapu urządzenia i tokenów parowania |
    | `plugin-sdk/extension-shared` | Współdzielone prymitywy helperów kanałów pasywnych, statusu i ambient proxy |
    | `plugin-sdk/models-provider-runtime` | Helpery odpowiedzi dostawcy/polecenia `/models` |
    | `plugin-sdk/skill-commands-runtime` | Helpery listowania poleceń Skills |
    | `plugin-sdk/native-command-registry` | Natywne helpery rejestracji/budowy/serializacji poleceń |
    | `plugin-sdk/provider-zai-endpoint` | Helpery wykrywania endpointu Z.A.I |
    | `plugin-sdk/infra-runtime` | Helpery zdarzeń systemowych/heartbeat |
    | `plugin-sdk/collection-runtime` | Małe helpery cache z ograniczonym rozmiarem |
    | `plugin-sdk/diagnostic-runtime` | Helpery flag i zdarzeń diagnostycznych |
    | `plugin-sdk/error-runtime` | Helpery grafu błędów, formatowania, współdzielonej klasyfikacji błędów, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Opakowane helpery fetch, proxy i pinned lookup |
    | `plugin-sdk/host-runtime` | Helpery normalizacji nazw hostów i hostów SCP |
    | `plugin-sdk/retry-runtime` | Helpery konfiguracji retry i wykonawcy retry |
    | `plugin-sdk/agent-runtime` | Helpery katalogu/tożsamości/przestrzeni roboczej agenta |
    | `plugin-sdk/directory-runtime` | Zapytania/deduplikacja katalogów opartych na konfiguracji |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Ścieżki podrzędne możliwości i testowania">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Współdzielone helpery pobierania/przekształcania/przechowywania mediów oraz konstruktory ładunków mediów |
    | `plugin-sdk/media-generation-runtime` | Współdzielone helpery failover generowania mediów, wybór kandydatów i komunikaty o brakującym modelu |
    | `plugin-sdk/media-understanding` | Typy dostawców rozumienia mediów oraz eksporty helperów obrazów/audio po stronie dostawcy |
    | `plugin-sdk/text-runtime` | Współdzielone helpery tekstu/Markdown/logowania, takie jak usuwanie tekstu widocznego dla asystenta, helpery renderowania/chunkingu/tabel Markdown, helpery redakcji, helpery tagów dyrektyw i bezpieczne narzędzia tekstowe |
    | `plugin-sdk/text-chunking` | Helper chunkingu tekstu wychodzącego |
    | `plugin-sdk/speech` | Typy dostawców mowy oraz helpery dyrektyw, rejestru i walidacji po stronie dostawcy |
    | `plugin-sdk/speech-core` | Współdzielone typy dostawców mowy, rejestru, dyrektyw i helpery normalizacji |
    | `plugin-sdk/realtime-transcription` | Typy dostawców transkrypcji czasu rzeczywistego i helpery rejestru |
    | `plugin-sdk/realtime-voice` | Typy dostawców głosu czasu rzeczywistego i helpery rejestru |
    | `plugin-sdk/image-generation` | Typy dostawców generowania obrazów |
    | `plugin-sdk/image-generation-core` | Współdzielone typy generowania obrazów, failover, auth i helpery rejestru |
    | `plugin-sdk/music-generation` | Typy żądań/wyników/dostawców generowania muzyki |
    | `plugin-sdk/music-generation-core` | Współdzielone typy generowania muzyki, helpery failover, wyszukiwania dostawców i parsowania model-ref |
    | `plugin-sdk/video-generation` | Typy żądań/wyników/dostawców generowania wideo |
    | `plugin-sdk/video-generation-core` | Współdzielone typy generowania wideo, helpery failover, wyszukiwania dostawców i parsowania model-ref |
    | `plugin-sdk/webhook-targets` | Rejestr celów webhooków i helpery instalacji tras |
    | `plugin-sdk/webhook-path` | Helpery normalizacji ścieżek webhooków |
    | `plugin-sdk/web-media` | Współdzielone helpery ładowania zdalnych/lokalnych mediów |
    | `plugin-sdk/zod` | Ponownie eksportowany `zod` dla użytkowników Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Ścieżki podrzędne pamięci">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/memory-core` | Powierzchnia helperów bundlowanego memory-core dla helperów managera/konfiguracji/plików/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fasada runtime indeksowania/wyszukiwania pamięci |
    | `plugin-sdk/memory-core-host-engine-foundation` | Eksporty silnika foundation hosta pamięci |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Eksporty silnika embeddingów hosta pamięci |
    | `plugin-sdk/memory-core-host-engine-qmd` | Eksporty silnika QMD hosta pamięci |
    | `plugin-sdk/memory-core-host-engine-storage` | Eksporty silnika przechowywania hosta pamięci |
    | `plugin-sdk/memory-core-host-multimodal` | Wielomodalne helpery hosta pamięci |
    | `plugin-sdk/memory-core-host-query` | Helpery zapytań hosta pamięci |
    | `plugin-sdk/memory-core-host-secret` | Helpery sekretów hosta pamięci |
    | `plugin-sdk/memory-core-host-events` | Helpery dziennika zdarzeń hosta pamięci |
    | `plugin-sdk/memory-core-host-status` | Helpery statusu hosta pamięci |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpery runtime CLI hosta pamięci |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpery głównego runtime hosta pamięci |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpery plików/runtime hosta pamięci |
    | `plugin-sdk/memory-host-core` | Neutralny wobec dostawcy alias dla helperów głównego runtime hosta pamięci |
    | `plugin-sdk/memory-host-events` | Neutralny wobec dostawcy alias dla helperów dziennika zdarzeń hosta pamięci |
    | `plugin-sdk/memory-host-files` | Neutralny wobec dostawcy alias dla helperów plików/runtime hosta pamięci |
    | `plugin-sdk/memory-host-markdown` | Współdzielone helpery zarządzanego Markdown dla pluginów sąsiadujących z pamięcią |
    | `plugin-sdk/memory-host-search` | Aktywna fasada runtime pamięci do dostępu do menedżera wyszukiwania |
    | `plugin-sdk/memory-host-status` | Neutralny wobec dostawcy alias dla helperów statusu hosta pamięci |
    | `plugin-sdk/memory-lancedb` | Powierzchnia helperów bundlowanego memory-lancedb |
  </Accordion>

  <Accordion title="Zarezerwowane ścieżki podrzędne helperów bundlowanych">
    | Family | Current subpaths | Intended use |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpery wsparcia bundlowanego pluginu przeglądarki (`browser-support` pozostaje barrelem zgodności) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Powierzchnia helperów/runtime bundlowanego Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Powierzchnia helperów/runtime bundlowanego LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Powierzchnia helperów bundlowanego IRC |
    | Channel-specific helpers | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Warstwy zgodności/helperów bundlowanych kanałów |
    | Auth/plugin-specific helpers | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Warstwy helperów bundlowanych funkcji/pluginów; `plugin-sdk/github-copilot-token` obecnie eksportuje `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` i `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API rejestracji

Callback `register(api)` otrzymuje obiekt `OpenClawPluginApi` z tymi
metodami:

### Rejestracja możliwości

| Method                                           | What it registers                |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Wnioskowanie tekstowe (LLM)      |
| `api.registerCliBackend(...)`                    | Lokalny backend wnioskowania CLI |
| `api.registerChannel(...)`                       | Kanał wiadomości                 |
| `api.registerSpeechProvider(...)`                | Synteza text-to-speech / STT     |
| `api.registerRealtimeTranscriptionProvider(...)` | Strumieniowa transkrypcja czasu rzeczywistego |
| `api.registerRealtimeVoiceProvider(...)`         | Dwukierunkowe sesje głosu czasu rzeczywistego |
| `api.registerMediaUnderstandingProvider(...)`    | Analiza obrazów/audio/wideo      |
| `api.registerImageGenerationProvider(...)`       | Generowanie obrazów              |
| `api.registerMusicGenerationProvider(...)`       | Generowanie muzyki               |
| `api.registerVideoGenerationProvider(...)`       | Generowanie wideo                |
| `api.registerWebFetchProvider(...)`              | Dostawca web fetch / scrape      |
| `api.registerWebSearchProvider(...)`             | Wyszukiwanie w sieci             |

### Narzędzia i polecenia

| Method                          | What it registers                             |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Narzędzie agenta (wymagane lub `{ optional: true }`) |
| `api.registerCommand(def)`      | Niestandardowe polecenie (omija LLM)          |

### Infrastruktura

| Method                                         | What it registers                       |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook zdarzenia                          |
| `api.registerHttpRoute(params)`                | Punkt końcowy HTTP Gateway              |
| `api.registerGatewayMethod(name, handler)`     | Metoda RPC Gateway                      |
| `api.registerCli(registrar, opts?)`            | Podpolecenie CLI                        |
| `api.registerService(service)`                 | Usługa w tle                            |
| `api.registerInteractiveHandler(registration)` | Handler interaktywny                    |
| `api.registerMemoryPromptSupplement(builder)`  | Addytywna sekcja promptu sąsiadująca z pamięcią |
| `api.registerMemoryCorpusSupplement(adapter)`  | Addytywny korpus wyszukiwania/odczytu pamięci |

Zarezerwowane przestrzenie nazw administracyjnych rdzenia (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) zawsze pozostają `operator.admin`, nawet jeśli plugin próbuje przypisać
węższy zakres metody gateway. Dla metod należących do pluginu preferuj
prefiksy specyficzne dla pluginu.

### Metadane rejestracji CLI

`api.registerCli(registrar, opts?)` akceptuje dwa rodzaje metadanych najwyższego poziomu:

- `commands`: jawne korzenie poleceń należące do registrara
- `descriptors`: deskryptory poleceń używane na etapie parsowania dla głównej pomocy CLI,
  routingu i leniwej rejestracji CLI pluginów

Jeśli chcesz, aby polecenie pluginu pozostawało ładowane leniwie w normalnej ścieżce głównego CLI,
podaj `descriptors`, które obejmują każdy korzeń polecenia najwyższego poziomu udostępniany przez tego
registrara.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Zarządzaj kontami Matrix, weryfikacją, urządzeniami i stanem profilu",
        hasSubcommands: true,
      },
    ],
  },
);
```

Używaj samego `commands` tylko wtedy, gdy nie potrzebujesz leniwej rejestracji głównego CLI.
Ta zgodna ścieżka eager nadal jest obsługiwana, ale nie instaluje
placeholderów opartych na deskryptorach dla leniwego ładowania na etapie parsowania.

### Rejestracja backendu CLI

`api.registerCliBackend(...)` pozwala pluginowi zarządzać domyślną konfiguracją lokalnego
backendu AI CLI, takiego jak `codex-cli`.

- `id` backendu staje się prefiksem dostawcy w odwołaniach do modeli takich jak `codex-cli/gpt-5`.
- `config` backendu używa tego samego kształtu co `agents.defaults.cliBackends.<id>`.
- Konfiguracja użytkownika nadal ma pierwszeństwo. OpenClaw scala `agents.defaults.cliBackends.<id>` z
  domyślną wartością pluginu przed uruchomieniem CLI.
- Używaj `normalizeConfig`, gdy backend potrzebuje przekształceń zgodności po scaleniu
  (na przykład normalizacji starych kształtów flag).

### Wyłączne sloty

| Method                                     | What it registers                                                                                                                                         |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Silnik kontekstu (aktywny może być tylko jeden naraz). Callback `assemble()` otrzymuje `availableTools` i `citationsMode`, aby silnik mógł dostosować dodatki do promptu. |
| `api.registerMemoryCapability(capability)` | Ujednolicona możliwość pamięci                                                                                                                            |
| `api.registerMemoryPromptSection(builder)` | Konstruktor sekcji promptu pamięci                                                                                                                        |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver planu opróżniania pamięci                                                                                                                        |
| `api.registerMemoryRuntime(runtime)`       | Adapter runtime pamięci                                                                                                                                   |

### Adaptery embeddingów pamięci

| Method                                         | What it registers                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adapter embeddingów pamięci dla aktywnego pluginu |

- `registerMemoryCapability` to preferowane API wyłącznego pluginu pamięci.
- `registerMemoryCapability` może również udostępniać `publicArtifacts.listArtifacts(...)`,
  aby pluginy towarzyszące mogły używać eksportowanych artefaktów pamięci przez
  `openclaw/plugin-sdk/memory-host-core` zamiast sięgać do prywatnego układu konkretnego
  pluginu pamięci.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` i
  `registerMemoryRuntime` to starsze, zgodne wstecz API wyłącznych pluginów pamięci.
- `registerMemoryEmbeddingProvider` pozwala aktywnemu pluginowi pamięci zarejestrować jeden
  lub więcej identyfikatorów adapterów embeddingów (na przykład `openai`, `gemini` lub niestandardowy identyfikator zdefiniowany przez plugin).
- Konfiguracja użytkownika, taka jak `agents.defaults.memorySearch.provider` i
  `agents.defaults.memorySearch.fallback`, jest rozstrzygana względem tych zarejestrowanych
  identyfikatorów adapterów.

### Zdarzenia i cykl życia

| Method                                       | What it does                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Typowany hook cyklu życia     |
| `api.onConversationBindingResolved(handler)` | Callback rozstrzygnięcia wiązania rozmowy |

### Semantyka decyzji hooków

- `before_tool_call`: zwrócenie `{ block: true }` jest końcowe. Gdy dowolny handler to ustawi, handlery o niższym priorytecie są pomijane.
- `before_tool_call`: zwrócenie `{ block: false }` jest traktowane jako brak decyzji (tak samo jak pominięcie `block`), a nie jako nadpisanie.
- `before_install`: zwrócenie `{ block: true }` jest końcowe. Gdy dowolny handler to ustawi, handlery o niższym priorytecie są pomijane.
- `before_install`: zwrócenie `{ block: false }` jest traktowane jako brak decyzji (tak samo jak pominięcie `block`), a nie jako nadpisanie.
- `reply_dispatch`: zwrócenie `{ handled: true, ... }` jest końcowe. Gdy dowolny handler przejmie wysyłkę, handlery o niższym priorytecie i domyślna ścieżka wysyłki modelu są pomijane.
- `message_sending`: zwrócenie `{ cancel: true }` jest końcowe. Gdy dowolny handler to ustawi, handlery o niższym priorytecie są pomijane.
- `message_sending`: zwrócenie `{ cancel: false }` jest traktowane jako brak decyzji (tak samo jak pominięcie `cancel`), a nie jako nadpisanie.

### Pola obiektu API

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Identyfikator pluginu                                                                       |
| `api.name`               | `string`                  | Nazwa wyświetlana                                                                           |
| `api.version`            | `string?`                 | Wersja pluginu (opcjonalnie)                                                                |
| `api.description`        | `string?`                 | Opis pluginu (opcjonalnie)                                                                  |
| `api.source`             | `string`                  | Ścieżka źródłowa pluginu                                                                    |
| `api.rootDir`            | `string?`                 | Katalog główny pluginu (opcjonalnie)                                                        |
| `api.config`             | `OpenClawConfig`          | Bieżąca migawka konfiguracji (aktywny migawkowy stan runtime w pamięci, gdy jest dostępny) |
| `api.pluginConfig`       | `Record<string, unknown>` | Konfiguracja specyficzna dla pluginu z `plugins.entries.<id>.config`                        |
| `api.runtime`            | `PluginRuntime`           | [Helpery runtime](/pl/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Logger z zakresem (`debug`, `info`, `warn`, `error`)                                        |
| `api.registrationMode`   | `PluginRegistrationMode`  | Bieżący tryb ładowania; `"setup-runtime"` to lekki etap uruchamiania/konfiguracji przed pełnym wejściem |
| `api.resolvePath(input)` | `(string) => string`      | Rozstrzyga ścieżkę względem katalogu głównego pluginu                                       |

## Konwencja modułów wewnętrznych

Wewnątrz swojego pluginu używaj lokalnych plików barrel dla importów wewnętrznych:

```
my-plugin/
  api.ts            # Eksporty publiczne dla zewnętrznych użytkowników
  runtime-api.ts    # Eksporty runtime tylko do użytku wewnętrznego
  index.ts          # Punkt wejścia pluginu
  setup-entry.ts    # Lekki punkt wejścia tylko do konfiguracji (opcjonalnie)
```

<Warning>
  Nigdy nie importuj własnego pluginu przez `openclaw/plugin-sdk/<your-plugin>`
  w kodzie produkcyjnym. Kieruj importy wewnętrzne przez `./api.ts` lub
  `./runtime-api.ts`. Ścieżka SDK jest wyłącznie kontraktem zewnętrznym.
</Warning>

Ładowane przez fasadę publiczne powierzchnie bundlowanych pluginów (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` i podobne publiczne pliki wejściowe) teraz preferują
aktywną migawkę konfiguracji runtime, gdy OpenClaw już działa. Jeśli migawka runtime jeszcze nie istnieje,
używają w zamian rozstrzygniętego pliku konfiguracji z dysku.

Pluginy dostawców mogą także udostępniać wąski, lokalny barrel kontraktu pluginu, gdy helper jest
celowo specyficzny dla dostawcy i jeszcze nie należy do ogólnej ścieżki podrzędnej SDK.
Obecny bundlowany przykład: dostawca Anthropic przechowuje helpery strumieni Claude we
własnej publicznej warstwie `api.ts` / `contract-api.ts` zamiast promować logikę nagłówków beta Anthropic i `service_tier`
do ogólnego kontraktu `plugin-sdk/*`.

Inne obecne bundlowane przykłady:

- `@openclaw/openai-provider`: `api.ts` eksportuje konstruktory dostawców,
  helpery domyślnych modeli i konstruktory dostawców realtime
- `@openclaw/openrouter-provider`: `api.ts` eksportuje konstruktor dostawcy oraz
  helpery onboardingu/konfiguracji

<Warning>
  Kod produkcyjny rozszerzeń powinien także unikać importów `openclaw/plugin-sdk/<other-plugin>`.
  Jeśli helper jest rzeczywiście współdzielony, przenieś go do neutralnej ścieżki podrzędnej SDK,
  takiej jak `openclaw/plugin-sdk/speech`, `.../provider-model-shared` lub innej
  powierzchni zorientowanej na możliwości, zamiast łączyć ze sobą dwa pluginy.
</Warning>

## Powiązane

- [Punkty wejścia](/pl/plugins/sdk-entrypoints) — opcje `definePluginEntry` i `defineChannelPluginEntry`
- [Helpery runtime](/pl/plugins/sdk-runtime) — pełna dokumentacja przestrzeni nazw `api.runtime`
- [Konfiguracja i config](/pl/plugins/sdk-setup) — pakowanie, manifesty, schematy konfiguracji
- [Testowanie](/pl/plugins/sdk-testing) — narzędzia testowe i reguły lint
- [Migracja SDK](/pl/plugins/sdk-migration) — migracja ze starszych powierzchni
- [Wnętrze pluginów](/pl/plugins/architecture) — szczegółowa architektura i model możliwości
