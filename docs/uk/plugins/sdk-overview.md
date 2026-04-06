---
read_when:
    - Вам потрібно знати, з якого підшляху SDK імпортувати
    - Вам потрібен довідник для всіх методів реєстрації в OpenClawPluginApi
    - Ви шукаєте конкретний експорт SDK
sidebarTitle: SDK Overview
summary: Карта імпортів, довідник API реєстрації та архітектура SDK
title: Огляд Plugin SDK
x-i18n:
    generated_at: "2026-04-06T04:11:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: acd2887ef52c66b2f234858d812bb04197ecd0bfb3e4f7bf3622f8fdc765acad
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Огляд Plugin SDK

Plugin SDK — це типізований контракт між plugins і core. Ця сторінка є
довідником про **що імпортувати** і **що можна реєструвати**.

<Tip>
  **Шукаєте практичний посібник?**
  - Перший plugin? Почніть із [Getting Started](/uk/plugins/building-plugins)
  - Channel plugin? Див. [Channel Plugins](/uk/plugins/sdk-channel-plugins)
  - Provider plugin? Див. [Provider Plugins](/uk/plugins/sdk-provider-plugins)
</Tip>

## Угода щодо імпорту

Завжди імпортуйте з конкретного підшляху:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Кожен підшлях — це невеликий, самодостатній модуль. Це пришвидшує запуск і
запобігає проблемам із циклічними залежностями. Для специфічних для channel
helpers входу/збирання надавайте перевагу `openclaw/plugin-sdk/channel-core`; `openclaw/plugin-sdk/core`
залишайте для ширшої umbrella-поверхні та спільних helpers, таких як
`buildChannelConfigSchema`.

Не додавайте та не покладайтеся на зручні seams з назвами provider, такі як
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, або
брендовані під channels helper seams. Bundled plugins мають компонувати загальні
підшляхи SDK у власних barrels `api.ts` або `runtime-api.ts`, а core
має або використовувати ці локальні для plugin barrels, або додавати вузький загальний SDK
contract, коли потреба справді є між channel.

Згенерована карта експортів усе ще містить невеликий набір helper seams для bundled-plugin,
таких як `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` і `plugin-sdk/matrix*`. Ці
підшляхи існують лише для супроводу bundled plugins і сумісності; вони
навмисно не включені до загальної таблиці нижче і не є рекомендованим
шляхом імпорту для нових сторонніх plugins.

## Довідник підшляхів

Найуживаніші підшляхи, згруповані за призначенням. Згенерований повний список із
понад 200 підшляхів міститься в `scripts/lib/plugin-sdk-entrypoints.json`.

Зарезервовані helper підшляхи bundled-plugin усе ще з’являються в цьому згенерованому списку.
Розглядайте їх як поверхні деталей реалізації/сумісності, якщо лише сторінка документації
не просуває якийсь із них явно як публічний.

### Вхід plugin

| Subpath                     | Ключові експорти                                                                                                                      |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Підшляхи channel">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Експорт кореневої Zod-схеми `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Спільні helpers майстра налаштування, запити allowlist, збирачі status налаштування |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers мультиакаунтної конфігурації/action-gate, helpers fallback стандартного акаунта |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers нормалізації account-id |
    | `plugin-sdk/account-resolution` | Пошук акаунта + helpers fallback за замовчуванням |
    | `plugin-sdk/account-helpers` | Вузькі helpers списку акаунтів/дій з акаунтами |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Типи схеми конфігурації channel |
    | `plugin-sdk/telegram-command-config` | Helpers нормалізації/валідації Telegram custom-command з fallback на bundled-contract |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Спільні helpers побудови inbound route + envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Спільні helpers запису й диспетчеризації inbound |
    | `plugin-sdk/messaging-targets` | Helpers розбору/зіставлення targets |
    | `plugin-sdk/outbound-media` | Спільні helpers завантаження outbound media |
    | `plugin-sdk/outbound-runtime` | Helpers outbound identity/send delegate |
    | `plugin-sdk/thread-bindings-runtime` | Helpers життєвого циклу й адаптера thread-binding |
    | `plugin-sdk/agent-media-payload` | Legacy builder payload media агента |
    | `plugin-sdk/conversation-runtime` | Helpers conversation/thread binding, pairing і configured-binding |
    | `plugin-sdk/runtime-config-snapshot` | Helper snapshot конфігурації runtime |
    | `plugin-sdk/runtime-group-policy` | Helpers визначення group-policy під час runtime |
    | `plugin-sdk/channel-status` | Спільні helpers snapshot/summary status channel |
    | `plugin-sdk/channel-config-primitives` | Вузькі primitives схеми конфігурації channel |
    | `plugin-sdk/channel-config-writes` | Helpers авторизації запису конфігурації channel |
    | `plugin-sdk/channel-plugin-common` | Спільні prelude-експорти plugin channel |
    | `plugin-sdk/allowlist-config-edit` | Helpers редагування/читання конфігурації allowlist |
    | `plugin-sdk/group-access` | Спільні helpers ухвалення рішень щодо доступу груп |
    | `plugin-sdk/direct-dm` | Спільні helpers auth/guard для прямих DM |
    | `plugin-sdk/interactive-runtime` | Helpers нормалізації/скорочення payload інтерактивних відповідей |
    | `plugin-sdk/channel-inbound` | Helpers debounce, зіставлення згадок, envelope |
    | `plugin-sdk/channel-send-result` | Типи результатів відповіді |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers розбору/зіставлення targets |
    | `plugin-sdk/channel-contract` | Типи contract для channel |
    | `plugin-sdk/channel-feedback` | Підключення feedback/reaction |
  </Accordion>

  <Accordion title="Підшляхи provider">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Добірні helpers налаштування локального/self-hosted provider |
    | `plugin-sdk/self-hosted-provider-setup` | Фокусовані helpers налаштування self-hosted provider, сумісного з OpenAI |
    | `plugin-sdk/provider-auth-runtime` | Helpers визначення API-key під час runtime для provider plugins |
    | `plugin-sdk/provider-auth-api-key` | Helpers онбордингу API-key/запису профілю |
    | `plugin-sdk/provider-auth-result` | Стандартний builder результату OAuth auth |
    | `plugin-sdk/provider-auth-login` | Спільні helpers інтерактивного входу для provider plugins |
    | `plugin-sdk/provider-env-vars` | Helpers пошуку env vars auth provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні builders replay-policy, helpers endpoint provider і helpers нормалізації model-id, такі як `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Загальні helpers HTTP/можливостей endpoint для provider |
    | `plugin-sdk/provider-web-fetch` | Helpers реєстрації/кешування provider web-fetch |
    | `plugin-sdk/provider-web-search` | Helpers реєстрації/кешування/конфігурації provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схеми Gemini + діагностика, а також helpers сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` та подібні |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи stream wrapper та спільні helpers wrappers для Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers patch конфігурації онбордингу |
    | `plugin-sdk/global-singleton` | Helpers локального для процесу singleton/map/cache |
  </Accordion>

  <Accordion title="Підшляхи auth і безпеки">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers реєстру команд, helpers авторизації відправника |
    | `plugin-sdk/approval-auth-runtime` | Helpers визначення approver і auth дій у тому самому чаті |
    | `plugin-sdk/approval-client-runtime` | Helpers профілю/фільтра native exec approval |
    | `plugin-sdk/approval-delivery-runtime` | Native adapters можливостей/доставки approval |
    | `plugin-sdk/approval-native-runtime` | Helpers native approval target + account-binding |
    | `plugin-sdk/approval-reply-runtime` | Helpers payload відповіді для exec/plugin approval |
    | `plugin-sdk/command-auth-native` | Native auth команд + helpers native session-target |
    | `plugin-sdk/command-detection` | Спільні helpers виявлення команд |
    | `plugin-sdk/command-surface` | Нормалізація тіла команди й helpers command-surface |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | Спільні helpers довіри, DM gating, зовнішнього вмісту та збирання секретів |
    | `plugin-sdk/ssrf-policy` | Helpers allowlist хостів і політики SSRF для приватних мереж |
    | `plugin-sdk/ssrf-runtime` | Helpers pinned-dispatcher, fetch із захистом SSRF і політики SSRF |
    | `plugin-sdk/secret-input` | Helpers розбору secret input |
    | `plugin-sdk/webhook-ingress` | Helpers webhook request/target |
    | `plugin-sdk/webhook-request-guards` | Helpers розміру body request/таймауту |
  </Accordion>

  <Accordion title="Підшляхи runtime і сховища">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/runtime` | Широкі helpers runtime/logging/backup/install plugin |
    | `plugin-sdk/runtime-env` | Вузькі helpers env runtime, logger, timeout, retry і backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Спільні helpers команд/hook/http/interactive для plugin |
    | `plugin-sdk/hook-runtime` | Спільні helpers pipeline для webhook/internal hook |
    | `plugin-sdk/lazy-runtime` | Helpers lazy import/binding runtime, такі як `createLazyRuntimeModule`, `createLazyRuntimeMethod` і `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers exec процесів |
    | `plugin-sdk/cli-runtime` | Helpers форматування CLI, очікування та версії |
    | `plugin-sdk/gateway-runtime` | Helpers клієнта gateway і patch status channel |
    | `plugin-sdk/config-runtime` | Helpers завантаження/запису конфігурації |
    | `plugin-sdk/telegram-command-config` | Нормалізація назв/описів команд Telegram і перевірки дублікатів/конфліктів, навіть коли поверхня bundled Telegram contract недоступна |
    | `plugin-sdk/approval-runtime` | Helpers exec/plugin approval, builders можливостей approval, helpers auth/profile, helpers native routing/runtime |
    | `plugin-sdk/reply-runtime` | Спільні helpers inbound/reply runtime, chunking, dispatch, heartbeat, планувальник reply |
    | `plugin-sdk/reply-dispatch-runtime` | Вузькі helpers dispatch/finalize reply |
    | `plugin-sdk/reply-history` | Спільні helpers короткого вікна history відповідей, такі як `buildHistoryContext`, `recordPendingHistoryEntry` і `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Вузькі helpers chunking тексту/markdown |
    | `plugin-sdk/session-store-runtime` | Helpers шляху сховища session + updated-at |
    | `plugin-sdk/state-paths` | Helpers шляхів каталогів state/OAuth |
    | `plugin-sdk/routing` | Helpers route/session-key/account binding, такі як `resolveAgentRoute`, `buildAgentSessionKey` і `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Спільні helpers summary status channel/account, типові значення runtime-state та helpers metadata проблем |
    | `plugin-sdk/target-resolver-runtime` | Спільні helpers визначення targets |
    | `plugin-sdk/string-normalization-runtime` | Helpers нормалізації slug/рядків |
    | `plugin-sdk/request-url` | Витягування рядкових URL з input, подібних до fetch/request |
    | `plugin-sdk/run-command` | Запуск команд із таймером та нормалізованими результатами stdout/stderr |
    | `plugin-sdk/param-readers` | Загальні readers параметрів tool/CLI |
    | `plugin-sdk/tool-send` | Витяг канонічних полів цілі відправлення з аргументів tool |
    | `plugin-sdk/temp-path` | Спільні helpers шляхів тимчасового завантаження |
    | `plugin-sdk/logging-core` | Helpers logger підсистеми та редагування чутливих даних |
    | `plugin-sdk/markdown-table-runtime` | Helpers режиму таблиць Markdown |
    | `plugin-sdk/json-store` | Невеликі helpers читання/запису JSON state |
    | `plugin-sdk/file-lock` | Re-entrant helpers блокування файлів |
    | `plugin-sdk/persistent-dedupe` | Helpers кешу dedupe збереженого на диску |
    | `plugin-sdk/acp-runtime` | Helpers ACP runtime/session і reply-dispatch |
    | `plugin-sdk/agent-config-primitives` | Вузькі primitives схеми конфігурації runtime агента |
    | `plugin-sdk/boolean-param` | Гнучкий reader boolean-параметрів |
    | `plugin-sdk/dangerous-name-runtime` | Helpers визначення зіставлень небезпечних імен |
    | `plugin-sdk/device-bootstrap` | Helpers початкового налаштування пристрою й токенів pairing |
    | `plugin-sdk/extension-shared` | Спільні primitives пасивного channel і helpers status |
    | `plugin-sdk/models-provider-runtime` | Helpers відповіді команди `/models`/provider |
    | `plugin-sdk/skill-commands-runtime` | Helpers виведення команд Skills |
    | `plugin-sdk/native-command-registry` | Helpers реєстру/build/serialize native команд |
    | `plugin-sdk/provider-zai-endpoint` | Helpers виявлення endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Helpers системних подій/heartbeat |
    | `plugin-sdk/collection-runtime` | Невеликі helpers bounded cache |
    | `plugin-sdk/diagnostic-runtime` | Helpers діагностичних прапорців і подій |
    | `plugin-sdk/error-runtime` | Граф помилок, форматування, helpers класифікації спільних помилок, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Обгорнуті helpers fetch, proxy і pinned lookup |
    | `plugin-sdk/host-runtime` | Helpers нормалізації hostname і SCP host |
    | `plugin-sdk/retry-runtime` | Helpers конфігурації retry і запуску retry |
    | `plugin-sdk/agent-runtime` | Helpers каталогів/ідентичності/workspace агента |
    | `plugin-sdk/directory-runtime` | Запит/dedup каталогів на основі конфігурації |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Підшляхи можливостей і тестування">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Спільні helpers fetch/transform/store media, а також builders payload media |
    | `plugin-sdk/media-generation-runtime` | Спільні helpers failover генерації media, вибір candidate і повідомлення про відсутню model |
    | `plugin-sdk/media-understanding` | Типи provider media understanding плюс експорти helpers для image/audio, орієнтовані на provider |
    | `plugin-sdk/text-runtime` | Спільні helpers тексту/markdown/logging, такі як видалення видимого для асистента тексту, helpers render/chunking/table Markdown, helpers редагування, helpers тегів директив і utilities безпечного тексту |
    | `plugin-sdk/text-chunking` | Helper chunking outbound text |
    | `plugin-sdk/speech` | Типи speech provider плюс helpers для provider, орієнтовані на directives, registry і validation |
    | `plugin-sdk/speech-core` | Спільні типи speech provider, registry, directive і helpers нормалізації |
    | `plugin-sdk/realtime-transcription` | Типи provider realtime transcription і helpers registry |
    | `plugin-sdk/realtime-voice` | Типи provider realtime voice і helpers registry |
    | `plugin-sdk/image-generation` | Типи provider image generation |
    | `plugin-sdk/image-generation-core` | Спільні типи image-generation, helpers failover, auth і registry |
    | `plugin-sdk/music-generation` | Типи provider/request/result для генерації музики |
    | `plugin-sdk/music-generation-core` | Спільні типи music-generation, helpers failover, пошук provider і розбір model-ref |
    | `plugin-sdk/video-generation` | Типи provider/request/result для генерації відео |
    | `plugin-sdk/video-generation-core` | Спільні типи video-generation, helpers failover, пошук provider і розбір model-ref |
    | `plugin-sdk/webhook-targets` | Реєстр targets webhook і helpers встановлення route |
    | `plugin-sdk/webhook-path` | Helpers нормалізації шляхів webhook |
    | `plugin-sdk/web-media` | Спільні helpers завантаження віддалених/локальних media |
    | `plugin-sdk/zod` | Повторно експортований `zod` для користувачів plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Підшляхи memory">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/memory-core` | Поверхня bundled helper memory-core для helpers manager/config/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Runtime facade індексації/пошуку пам’яті |
    | `plugin-sdk/memory-core-host-engine-foundation` | Експорти foundation engine для memory host |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Експорти embedding engine для memory host |
    | `plugin-sdk/memory-core-host-engine-qmd` | Експорти QMD engine для memory host |
    | `plugin-sdk/memory-core-host-engine-storage` | Експорти storage engine для memory host |
    | `plugin-sdk/memory-core-host-multimodal` | Multimodal helpers для memory host |
    | `plugin-sdk/memory-core-host-query` | Helpers query для memory host |
    | `plugin-sdk/memory-core-host-secret` | Helpers secret для memory host |
    | `plugin-sdk/memory-core-host-events` | Helpers журналу подій для memory host |
    | `plugin-sdk/memory-core-host-status` | Helpers status для memory host |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers CLI runtime для memory host |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers core runtime для memory host |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers files/runtime для memory host |
    | `plugin-sdk/memory-host-core` | Vendor-neutral псевдонім для helpers core runtime memory host |
    | `plugin-sdk/memory-host-events` | Vendor-neutral псевдонім для helpers журналу подій memory host |
    | `plugin-sdk/memory-host-files` | Vendor-neutral псевдонім для helpers file/runtime memory host |
    | `plugin-sdk/memory-host-markdown` | Спільні helpers managed-markdown для plugins, суміжних із memory |
    | `plugin-sdk/memory-host-search` | Active memory runtime facade для доступу до search-manager |
    | `plugin-sdk/memory-host-status` | Vendor-neutral псевдонім для helpers status memory host |
    | `plugin-sdk/memory-lancedb` | Поверхня bundled helper memory-lancedb |
  </Accordion>

  <Accordion title="Зарезервовані підшляхи bundled-helper">
    | Family | Поточні підшляхи | Призначення |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers підтримки bundled browser plugin (`browser-support` залишається compatibility barrel) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Поверхня helper/runtime для bundled Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Поверхня helper/runtime для bundled LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Поверхня helper для bundled IRC |
    | Специфічні для channel helpers | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Bundled seams сумісності/helper для channel |
    | Специфічні для auth/plugin helpers | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Bundled helper seams для функцій/plugins; `plugin-sdk/github-copilot-token` наразі експортує `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API реєстрації

Callback `register(api)` отримує об’єкт `OpenClawPluginApi` з такими
методами:

### Реєстрація можливостей

| Method                                           | Що він реєструє                 |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Текстовий inference (LLM)        |
| `api.registerChannel(...)`                       | Канал обміну повідомленнями      |
| `api.registerSpeechProvider(...)`                | Text-to-speech / STT synthesis   |
| `api.registerRealtimeTranscriptionProvider(...)` | Потокова транскрипція realtime   |
| `api.registerRealtimeVoiceProvider(...)`         | Двоспрямовані voice-сесії realtime |
| `api.registerMediaUnderstandingProvider(...)`    | Аналіз зображень/аудіо/відео     |
| `api.registerImageGenerationProvider(...)`       | Генерація зображень              |
| `api.registerMusicGenerationProvider(...)`       | Генерація музики                 |
| `api.registerVideoGenerationProvider(...)`       | Генерація відео                  |
| `api.registerWebFetchProvider(...)`              | Provider для web fetch / scrape  |
| `api.registerWebSearchProvider(...)`             | Вебпошук                         |

### Tools і команди

| Method                          | Що він реєструє                                |
| ------------------------------- | ---------------------------------------------- |
| `api.registerTool(tool, opts?)` | Інструмент агента (обов’язковий або `{ optional: true }`) |
| `api.registerCommand(def)`      | Користувацьку команду (обходить LLM)           |

### Інфраструктура

| Method                                         | Що він реєструє                    |
| ---------------------------------------------- | ---------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Event hook                         |
| `api.registerHttpRoute(params)`                | HTTP endpoint gateway              |
| `api.registerGatewayMethod(name, handler)`     | RPC-метод gateway                  |
| `api.registerCli(registrar, opts?)`            | Підкоманду CLI                     |
| `api.registerService(service)`                 | Фоновий service                    |
| `api.registerInteractiveHandler(registration)` | Інтерактивний handler              |
| `api.registerMemoryPromptSupplement(builder)`  | Додаткову секцію prompt, суміжну з memory |
| `api.registerMemoryCorpusSupplement(adapter)`  | Додатковий corpus пошуку/читання memory |

Зарезервовані простори імен core admin (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) завжди залишаються `operator.admin`, навіть якщо plugin намагається призначити
вужчу область gateway method. Для
методів, що належать plugin, надавайте перевагу префіксам, специфічним для plugin.

### Метадані реєстрації CLI

`api.registerCli(registrar, opts?)` приймає два види метаданих верхнього рівня:

- `commands`: явні корені команд, якими володіє registrar
- `descriptors`: дескриптори команд на етапі парсингу, які використовуються для кореневої довідки CLI,
  маршрутизації та лінивої реєстрації CLI plugin

Якщо ви хочете, щоб команда plugin залишалася ліниво завантажуваною в звичайному кореневому шляху CLI,
надайте `descriptors`, які охоплюють кожен корінь команди верхнього рівня, що експонується цим
registrar.

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
        description: "Керуйте акаунтами Matrix, верифікацією, пристроями та станом профілю",
        hasSubcommands: true,
      },
    ],
  },
);
```

Використовуйте лише `commands`, тільки якщо вам не потрібна лінива коренева реєстрація CLI.
Цей eager-шлях сумісності й надалі підтримується, але він не встановлює
placeholders на основі descriptors для лінивого завантаження під час парсингу.

### Ексклюзивні слоти

| Method                                     | Що він реєструє                    |
| ------------------------------------------ | ---------------------------------- |
| `api.registerContextEngine(id, factory)`   | Context engine (одночасно активний лише один) |
| `api.registerMemoryPromptSection(builder)` | Builder секції prompt для memory   |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver плану скидання memory     |
| `api.registerMemoryRuntime(runtime)`       | Адаптер runtime для memory         |

### Адаптери embedding для memory

| Method                                         | Що він реєструє                                     |
| ---------------------------------------------- | --------------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Адаптер embedding для memory для активного plugin   |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` і
  `registerMemoryRuntime` є ексклюзивними для memory plugins.
- `registerMemoryEmbeddingProvider` дає активному memory plugin змогу реєструвати
  один або кілька id адаптерів embedding (наприклад `openai`, `gemini` або custom
  id, визначений plugin).
- Конфігурація користувача, як-от `agents.defaults.memorySearch.provider` і
  `agents.defaults.memorySearch.fallback`, визначається відносно цих зареєстрованих
  id адаптерів.

### Події та життєвий цикл

| Method                                       | Що він робить              |
| -------------------------------------------- | -------------------------- |
| `api.on(hookName, handler, opts?)`           | Типізований lifecycle hook |
| `api.onConversationBindingResolved(handler)` | Callback binding conversation |

### Семантика рішень hook

- `before_tool_call`: повернення `{ block: true }` є термінальним. Щойно будь-який handler встановлює це значення, handlers з нижчим пріоритетом пропускаються.
- `before_tool_call`: повернення `{ block: false }` трактується як відсутність рішення (так само, як пропуск `block`), а не як перевизначення.
- `before_install`: повернення `{ block: true }` є термінальним. Щойно будь-який handler встановлює це значення, handlers з нижчим пріоритетом пропускаються.
- `before_install`: повернення `{ block: false }` трактується як відсутність рішення (так само, як пропуск `block`), а не як перевизначення.
- `reply_dispatch`: повернення `{ handled: true, ... }` є термінальним. Щойно будь-який handler бере dispatch на себе, handlers з нижчим пріоритетом і стандартний шлях dispatch моделі пропускаються.
- `message_sending`: повернення `{ cancel: true }` є термінальним. Щойно будь-який handler встановлює це значення, handlers з нижчим пріоритетом пропускаються.
- `message_sending`: повернення `{ cancel: false }` трактується як відсутність рішення (так само, як пропуск `cancel`), а не як перевизначення.

### Поля об’єкта API

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id plugin                                                                                   |
| `api.name`               | `string`                  | Відображувана назва                                                                         |
| `api.version`            | `string?`                 | Версія plugin (необов’язково)                                                               |
| `api.description`        | `string?`                 | Опис plugin (необов’язково)                                                                 |
| `api.source`             | `string`                  | Шлях до джерела plugin                                                                      |
| `api.rootDir`            | `string?`                 | Кореневий каталог plugin (необов’язково)                                                    |
| `api.config`             | `OpenClawConfig`          | Поточний snapshot конфігурації (активний snapshot runtime у пам’яті, коли доступний)       |
| `api.pluginConfig`       | `Record<string, unknown>` | Специфічна для plugin конфігурація з `plugins.entries.<id>.config`                         |
| `api.runtime`            | `PluginRuntime`           | [Helpers runtime](/uk/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Logger з областю видимості (`debug`, `info`, `warn`, `error`)                               |
| `api.registrationMode`   | `PluginRegistrationMode`  | Поточний режим завантаження; `"setup-runtime"` — це полегшене вікно запуску/налаштування до повного entry |
| `api.resolvePath(input)` | `(string) => string`      | Визначення шляху відносно кореня plugin                                                     |

## Угода щодо внутрішніх модулів

Усередині вашого plugin використовуйте локальні barrel-файли для внутрішніх імпортів:

```
my-plugin/
  api.ts            # Публічні експорти для зовнішніх споживачів
  runtime-api.ts    # Експорти runtime лише для внутрішнього використання
  index.ts          # Точка входу plugin
  setup-entry.ts    # Полегшений entry лише для налаштування (необов’язково)
```

<Warning>
  Ніколи не імпортуйте власний plugin через `openclaw/plugin-sdk/<your-plugin>`
  з production-коду. Спрямовуйте внутрішні імпорти через `./api.ts` або
  `./runtime-api.ts`. Шлях SDK — це лише зовнішній contract.
</Warning>

Публічні поверхні bundled plugin, завантажувані через facade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` та подібні публічні entry-файли), тепер надають перевагу
активному snapshot конфігурації runtime, коли OpenClaw уже запущений. Якщо snapshot runtime
ще не існує, вони повертаються до визначеного файла конфігурації на диску.

Provider plugins також можуть експортувати вузький локальний для plugin barrel контракту, коли
helper є навмисно специфічним для provider і поки що не належить до загального
підшляху SDK. Поточний bundled приклад: provider Anthropic зберігає свої Claude
stream helpers у власному публічному seam `api.ts` / `contract-api.ts` замість
просування логіки Anthropic beta-header і `service_tier` у загальний
contract `plugin-sdk/*`.

Інші поточні bundled приклади:

- `@openclaw/openai-provider`: `api.ts` експортує builders provider,
  helpers моделей за замовчуванням і builders realtime provider
- `@openclaw/openrouter-provider`: `api.ts` експортує builder provider плюс
  helpers онбордингу/конфігурації

<Warning>
  Production-код extension також має уникати імпортів `openclaw/plugin-sdk/<other-plugin>`.
  Якщо helper справді є спільним, перенесіть його до нейтрального підшляху SDK,
  наприклад `openclaw/plugin-sdk/speech`, `.../provider-model-shared` або іншої
  поверхні, орієнтованої на можливості, замість жорсткого зв’язування двох plugins.
</Warning>

## Пов’язані сторінки

- [Entry Points](/uk/plugins/sdk-entrypoints) — параметри `definePluginEntry` і `defineChannelPluginEntry`
- [Runtime Helpers](/uk/plugins/sdk-runtime) — повний довідник простору імен `api.runtime`
- [Setup and Config](/uk/plugins/sdk-setup) — пакування, маніфести, схеми конфігурації
- [Testing](/uk/plugins/sdk-testing) — утиліти тестування та правила lint
- [SDK Migration](/uk/plugins/sdk-migration) — міграція із застарілих поверхонь
- [Plugin Internals](/uk/plugins/architecture) — глибока архітектура та модель можливостей
