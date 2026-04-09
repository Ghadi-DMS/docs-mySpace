---
read_when:
    - Потрібно знати, з якого підшляху SDK імпортувати
    - Потрібен довідник для всіх методів реєстрації в OpenClawPluginApi
    - Ви шукаєте конкретний експорт SDK
sidebarTitle: SDK Overview
summary: Карта імпортів, довідник API реєстрації та архітектура SDK
title: Огляд Plugin SDK
x-i18n:
    generated_at: "2026-04-09T00:38:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf205af060971931df97dca4af5110ce173d2b7c12f56ad7c62d664a402f2381
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Огляд Plugin SDK

Plugin SDK — це типізований контракт між плагінами та ядром. Ця сторінка —
довідник про **що імпортувати** і **що можна реєструвати**.

<Tip>
  **Шукаєте покроковий посібник?**
  - Перший плагін? Почніть із [Getting Started](/uk/plugins/building-plugins)
  - Плагін каналу? Див. [Channel Plugins](/uk/plugins/sdk-channel-plugins)
  - Плагін провайдера? Див. [Provider Plugins](/uk/plugins/sdk-provider-plugins)
</Tip>

## Правило імпорту

Завжди імпортуйте з конкретного підшляху:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Кожен підшлях — це невеликий самодостатній модуль. Це забезпечує швидкий
запуск і запобігає проблемам із циклічними залежностями. Для специфічних для
каналів допоміжних засобів входу/збирання надавайте перевагу
`openclaw/plugin-sdk/channel-core`; залишайте `openclaw/plugin-sdk/core` для
ширшої узагальнювальної поверхні та спільних допоміжних засобів, таких як
`buildChannelConfigSchema`.

Не додавайте і не використовуйте зручні шви з назвами провайдерів, такі як
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, або допоміжні
шви з брендингом каналів. Вбудовані плагіни мають компонувати загальні
підшляхи SDK у власних barrel-файлах `api.ts` або `runtime-api.ts`, а ядро
має або використовувати ці локальні barrel-файли плагіна, або додавати вузький
загальний контракт SDK, якщо потреба справді є міжканальною.

Згенерована карта експортів усе ще містить невеликий набір допоміжних
швів для вбудованих плагінів, таких як `plugin-sdk/feishu`,
`plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup` і
`plugin-sdk/matrix*`. Ці підшляхи існують лише для підтримки вбудованих
плагінів і сумісності; їх навмисно не включено до загальної таблиці нижче,
і вони не є рекомендованим шляхом імпорту для нових сторонніх плагінів.

## Довідник підшляхів

Найуживаніші підшляхи, згруповані за призначенням. Згенерований повний список
із понад 200 підшляхів міститься в `scripts/lib/plugin-sdk-entrypoints.json`.

Зарезервовані допоміжні підшляхи вбудованих плагінів усе ще з’являються в
цьому згенерованому списку. Вважайте їх деталями реалізації / поверхнями
сумісності, якщо тільки якась сторінка документації явно не просуває один із
них як публічний.

### Вхід плагіна

| Підшлях                    | Ключові експорти                                                                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                  |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                     |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                    |

<AccordionGroup>
  <Accordion title="Підшляхи каналів">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Експорт кореневої Zod-схеми `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Спільні допоміжні засоби майстра налаштування, запити списків дозволених джерел, побудова статусу налаштування |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Допоміжні засоби для конфігурації/керування шлюзами дій з кількома обліковими записами, допоміжні засоби резервного використання типового облікового запису |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, допоміжні засоби нормалізації account-id |
    | `plugin-sdk/account-resolution` | Пошук облікового запису + допоміжні засоби резервного використання типового значення |
    | `plugin-sdk/account-helpers` | Вузькі допоміжні засоби для списків облікових записів / дій над обліковими записами |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Типи схеми конфігурації каналу |
    | `plugin-sdk/telegram-command-config` | Допоміжні засоби нормалізації/валідації користувацьких команд Telegram із резервним вбудованим контрактом |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Спільні допоміжні засоби маршрутизації вхідних повідомлень і побудови envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Спільні допоміжні засоби запису та диспетчеризації вхідних повідомлень |
    | `plugin-sdk/messaging-targets` | Допоміжні засоби розбору/зіставлення цілей |
    | `plugin-sdk/outbound-media` | Спільні допоміжні засоби завантаження вихідних медіа |
    | `plugin-sdk/outbound-runtime` | Допоміжні засоби ідентичності вихідних повідомлень / делегатів надсилання |
    | `plugin-sdk/thread-bindings-runtime` | Життєвий цикл прив’язок потоків і допоміжні засоби адаптерів |
    | `plugin-sdk/agent-media-payload` | Застарілий побудовник медіа payload агента |
    | `plugin-sdk/conversation-runtime` | Допоміжні засоби прив’язки розмов/потоків, pairинг і налаштованих прив’язок |
    | `plugin-sdk/runtime-config-snapshot` | Допоміжний засіб знімка конфігурації під час виконання |
    | `plugin-sdk/runtime-group-policy` | Допоміжні засоби визначення групових політик під час виконання |
    | `plugin-sdk/channel-status` | Спільні допоміжні засоби знімка/зведення статусу каналу |
    | `plugin-sdk/channel-config-primitives` | Вузькі примітиви схеми конфігурації каналу |
    | `plugin-sdk/channel-config-writes` | Допоміжні засоби авторизації запису конфігурації каналу |
    | `plugin-sdk/channel-plugin-common` | Спільні преамбулні експорти плагінів каналів |
    | `plugin-sdk/allowlist-config-edit` | Допоміжні засоби редагування/читання конфігурації списку дозволених джерел |
    | `plugin-sdk/group-access` | Спільні допоміжні засоби рішень щодо доступу груп |
    | `plugin-sdk/direct-dm` | Спільні допоміжні засоби автентифікації/захисту прямих DM |
    | `plugin-sdk/interactive-runtime` | Допоміжні засоби нормалізації/скорочення payload інтерактивних відповідей |
    | `plugin-sdk/channel-inbound` | Допоміжні засоби debounce для вхідних повідомлень, зіставлення згадок, політики згадок і envelope |
    | `plugin-sdk/channel-send-result` | Типи результатів відповіді |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Допоміжні засоби розбору/зіставлення цілей |
    | `plugin-sdk/channel-contract` | Типи контракту каналу |
    | `plugin-sdk/channel-feedback` | Підключення відгуків/реакцій |
    | `plugin-sdk/channel-secret-runtime` | Вузькі допоміжні засоби контракту секретів, такі як `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, а також типи цілей секретів |
  </Accordion>

  <Accordion title="Підшляхи провайдерів">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Кураторські допоміжні засоби налаштування локальних/self-hosted провайдерів |
    | `plugin-sdk/self-hosted-provider-setup` | Сфокусовані допоміжні засоби налаштування self-hosted провайдерів, сумісних з OpenAI |
    | `plugin-sdk/cli-backend` | Типові значення CLI-бекенду + константи watchdog |
    | `plugin-sdk/provider-auth-runtime` | Допоміжні засоби визначення API-ключів під час виконання для плагінів провайдерів |
    | `plugin-sdk/provider-auth-api-key` | Допоміжні засоби онбордингу/запису профілю API-ключа, такі як `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Стандартний побудовник результату OAuth-автентифікації |
    | `plugin-sdk/provider-auth-login` | Спільні інтерактивні допоміжні засоби входу для плагінів провайдерів |
    | `plugin-sdk/provider-env-vars` | Допоміжні засоби пошуку env var для автентифікації провайдерів |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні побудовники політик replay, допоміжні засоби endpoint провайдерів і нормалізації model-id, такі як `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Загальні допоміжні засоби HTTP/можливостей endpoint провайдерів |
    | `plugin-sdk/provider-web-fetch-contract` | Вузькі допоміжні засоби контракту конфігурації/вибору web-fetch, такі як `enablePluginInConfig` і `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Допоміжні засоби реєстрації/кешу провайдерів web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Вузькі допоміжні засоби конфігурації/облікових даних web-search для провайдерів, яким не потрібне підключення ввімкнення плагіна |
    | `plugin-sdk/provider-web-search-contract` | Вузькі допоміжні засоби контракту конфігурації/облікових даних web-search, такі як `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` і область дії setter/getter для облікових даних |
    | `plugin-sdk/provider-web-search` | Допоміжні засоби реєстрації/кешу/виконання провайдерів web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схеми Gemini + діагностика та допоміжні засоби сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` та подібні |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи обгорток потоків і спільні допоміжні засоби обгорток Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Допоміжні засоби патчів конфігурації онбордингу |
    | `plugin-sdk/global-singleton` | Допоміжні засоби локальних для процесу singleton/map/cache |
  </Accordion>

  <Accordion title="Підшляхи автентифікації та безпеки">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, допоміжні засоби реєстру команд, допоміжні засоби авторизації відправника |
    | `plugin-sdk/command-status` | Побудовники повідомлень команд/довідки, такі як `buildCommandsMessagePaginated` і `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Допоміжні засоби визначення approver і автентифікації дій у межах того самого чату |
    | `plugin-sdk/approval-client-runtime` | Допоміжні засоби профілів/фільтрів нативного схвалення виконання |
    | `plugin-sdk/approval-delivery-runtime` | Адаптери нативних можливостей/доставки схвалення |
    | `plugin-sdk/approval-gateway-runtime` | Спільний допоміжний засіб визначення шлюзу схвалення |
    | `plugin-sdk/approval-handler-adapter-runtime` | Полегшені допоміжні засоби завантаження нативного адаптера схвалення для гарячих entrypoint каналу |
    | `plugin-sdk/approval-handler-runtime` | Ширші допоміжні засоби runtime-обробника схвалення; надавайте перевагу вужчим швам adapter/gateway, коли їх достатньо |
    | `plugin-sdk/approval-native-runtime` | Допоміжні засоби нативних цілей схвалення + прив’язки облікових записів |
    | `plugin-sdk/approval-reply-runtime` | Допоміжні засоби payload відповідей схвалення exec/plugin |
    | `plugin-sdk/command-auth-native` | Допоміжні засоби нативної автентифікації команд + нативних цілей сеансу |
    | `plugin-sdk/command-detection` | Спільні допоміжні засоби виявлення команд |
    | `plugin-sdk/command-surface` | Допоміжні засоби нормалізації тіла команди та command-surface |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Вузькі допоміжні засоби збирання контракту секретів для поверхонь секретів каналу/плагіна |
    | `plugin-sdk/secret-ref-runtime` | Вузькі допоміжні засоби `coerceSecretRef` і типізації SecretRef для парсингу контракту секретів/конфігурації |
    | `plugin-sdk/security-runtime` | Спільні допоміжні засоби довіри, обмеження DM, зовнішнього вмісту та збирання секретів |
    | `plugin-sdk/ssrf-policy` | Допоміжні засоби політики SSRF для списків дозволених хостів і приватних мереж |
    | `plugin-sdk/ssrf-runtime` | Допоміжні засоби pinned-dispatcher, fetch із захистом SSRF і політики SSRF |
    | `plugin-sdk/secret-input` | Допоміжні засоби парсингу введення секретів |
    | `plugin-sdk/webhook-ingress` | Допоміжні засоби запиту/цілі webhook |
    | `plugin-sdk/webhook-request-guards` | Допоміжні засоби розміру тіла запиту / тайм-аутів |
  </Accordion>

  <Accordion title="Підшляхи runtime і сховища">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/runtime` | Широкі допоміжні засоби runtime/логування/резервних копій/встановлення плагінів |
    | `plugin-sdk/runtime-env` | Вужчі допоміжні засоби env runtime, логера, тайм-аутів, повторних спроб і backoff |
    | `plugin-sdk/channel-runtime-context` | Загальні допоміжні засоби реєстрації та пошуку channel runtime-context |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Спільні допоміжні засоби команд/хуків/http/інтерактивних функцій плагіна |
    | `plugin-sdk/hook-runtime` | Спільні допоміжні засоби pipeline webhook/внутрішніх хуків |
    | `plugin-sdk/lazy-runtime` | Допоміжні засоби лінивого імпорту/прив’язки runtime, такі як `createLazyRuntimeModule`, `createLazyRuntimeMethod` і `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Допоміжні засоби виконання процесів |
    | `plugin-sdk/cli-runtime` | Допоміжні засоби форматування CLI, очікування та версій |
    | `plugin-sdk/gateway-runtime` | Допоміжні засоби клієнта шлюзу та патчів статусу каналу |
    | `plugin-sdk/config-runtime` | Допоміжні засоби завантаження/запису конфігурації |
    | `plugin-sdk/telegram-command-config` | Нормалізація назв/описів команд Telegram і перевірки дублювань/конфліктів, навіть коли поверхня вбудованого контракту Telegram недоступна |
    | `plugin-sdk/approval-runtime` | Допоміжні засоби схвалення exec/plugin, побудовники можливостей схвалення, допоміжні засоби auth/profile, нативної маршрутизації/runtime |
    | `plugin-sdk/reply-runtime` | Спільні допоміжні засоби runtime для вхідних повідомлень/відповідей, chunking, dispatch, heartbeat, planner відповідей |
    | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні засоби dispatch/finalize для відповідей |
    | `plugin-sdk/reply-history` | Спільні допоміжні засоби історії відповідей для короткого вікна, такі як `buildHistoryContext`, `recordPendingHistoryEntry` і `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Вузькі допоміжні засоби chunking тексту/markdown |
    | `plugin-sdk/session-store-runtime` | Допоміжні засоби шляхів до сховища сеансів + updated-at |
    | `plugin-sdk/state-paths` | Допоміжні засоби шляхів до каталогів state/OAuth |
    | `plugin-sdk/routing` | Допоміжні засоби маршрутів / session-key / прив’язки облікових записів, такі як `resolveAgentRoute`, `buildAgentSessionKey` і `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Спільні допоміжні засоби зведення статусу каналу/облікового запису, типові значення стану runtime і допоміжні засоби метаданих проблем |
    | `plugin-sdk/target-resolver-runtime` | Спільні допоміжні засоби визначення цілей |
    | `plugin-sdk/string-normalization-runtime` | Допоміжні засоби нормалізації slug/рядків |
    | `plugin-sdk/request-url` | Витягування рядкових URL із fetch/request-подібних входів |
    | `plugin-sdk/run-command` | Виконавець команд із таймером і нормалізованими результатами stdout/stderr |
    | `plugin-sdk/param-readers` | Поширені читачі параметрів інструментів/CLI |
    | `plugin-sdk/tool-payload` | Витягування нормалізованих payload із об’єктів результатів інструментів |
    | `plugin-sdk/tool-send` | Витягування канонічних полів цілі надсилання з аргументів інструменту |
    | `plugin-sdk/temp-path` | Спільні допоміжні засоби шляхів для тимчасових завантажень |
    | `plugin-sdk/logging-core` | Допоміжні засоби логера підсистеми та редагування чутливих даних |
    | `plugin-sdk/markdown-table-runtime` | Допоміжні засоби режиму таблиць Markdown |
    | `plugin-sdk/json-store` | Невеликі допоміжні засоби читання/запису JSON-стану |
    | `plugin-sdk/file-lock` | Реентерабельні допоміжні засоби file-lock |
    | `plugin-sdk/persistent-dedupe` | Допоміжні засоби кешу дедуплікації на диску |
    | `plugin-sdk/acp-runtime` | Допоміжні засоби runtime/сеансів ACP і dispatch відповідей |
    | `plugin-sdk/agent-config-primitives` | Вузькі примітиви схеми конфігурації runtime агента |
    | `plugin-sdk/boolean-param` | Читач слабко типізованих boolean-параметрів |
    | `plugin-sdk/dangerous-name-runtime` | Допоміжні засоби визначення зіставлення небезпечних імен |
    | `plugin-sdk/device-bootstrap` | Допоміжні засоби bootstrap пристрою та токенів pairингу |
    | `plugin-sdk/extension-shared` | Спільні примітиви допоміжних засобів пасивних каналів, статусу та ambient proxy |
    | `plugin-sdk/models-provider-runtime` | Допоміжні засоби відповідей команди `/models` / провайдера |
    | `plugin-sdk/skill-commands-runtime` | Допоміжні засоби переліку команд Skills |
    | `plugin-sdk/native-command-registry` | Допоміжні засоби реєстру/build/serialize нативних команд |
    | `plugin-sdk/provider-zai-endpoint` | Допоміжні засоби виявлення endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Допоміжні засоби системних подій/heartbeat |
    | `plugin-sdk/collection-runtime` | Невеликі допоміжні засоби обмеженого кешу |
    | `plugin-sdk/diagnostic-runtime` | Допоміжні засоби прапорців і подій діагностики |
    | `plugin-sdk/error-runtime` | Допоміжні засоби графа помилок, форматування, спільної класифікації помилок, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Допоміжні засоби обгорнутого fetch, proxy і pinned lookup |
    | `plugin-sdk/host-runtime` | Допоміжні засоби нормалізації hostname і SCP host |
    | `plugin-sdk/retry-runtime` | Допоміжні засоби конфігурації повторних спроб і виконавця повторних спроб |
    | `plugin-sdk/agent-runtime` | Допоміжні засоби каталогу/ідентичності/робочого простору агента |
    | `plugin-sdk/directory-runtime` | Запит/дедуплікація каталогів на основі конфігурації |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Підшляхи можливостей і тестування">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Спільні допоміжні засоби fetch/transform/store для медіа плюс побудовники медіа payload |
    | `plugin-sdk/media-generation-runtime` | Спільні допоміжні засоби failover для генерації медіа, вибір кандидатів і повідомлення про відсутні моделі |
    | `plugin-sdk/media-understanding` | Типи провайдерів розуміння медіа плюс орієнтовані на провайдерів експорти допоміжних засобів для зображень/аудіо |
    | `plugin-sdk/text-runtime` | Спільні допоміжні засоби тексту/markdown/логування, такі як вилучення видимого асистенту тексту, render/chunking/table для markdown, редагування чутливих даних, допоміжні засоби тегів директив і утиліти безпечного тексту |
    | `plugin-sdk/text-chunking` | Допоміжний засіб chunking вихідного тексту |
    | `plugin-sdk/speech` | Типи провайдерів мовлення плюс орієнтовані на провайдерів допоміжні засоби директив, реєстру та валідації |
    | `plugin-sdk/speech-core` | Спільні типи провайдерів мовлення, допоміжні засоби реєстру, директив і нормалізації |
    | `plugin-sdk/realtime-transcription` | Типи провайдерів транскрипції в реальному часі та допоміжні засоби реєстру |
    | `plugin-sdk/realtime-voice` | Типи провайдерів голосу в реальному часі та допоміжні засоби реєстру |
    | `plugin-sdk/image-generation` | Типи провайдерів генерації зображень |
    | `plugin-sdk/image-generation-core` | Спільні типи генерації зображень, failover, auth і допоміжні засоби реєстру |
    | `plugin-sdk/music-generation` | Типи провайдерів/запитів/результатів генерації музики |
    | `plugin-sdk/music-generation-core` | Спільні типи генерації музики, допоміжні засоби failover, пошук провайдерів і парсинг model-ref |
    | `plugin-sdk/video-generation` | Типи провайдерів/запитів/результатів генерації відео |
    | `plugin-sdk/video-generation-core` | Спільні типи генерації відео, допоміжні засоби failover, пошук провайдерів і парсинг model-ref |
    | `plugin-sdk/webhook-targets` | Реєстр цілей webhook і допоміжні засоби встановлення маршрутів |
    | `plugin-sdk/webhook-path` | Допоміжні засоби нормалізації шляху webhook |
    | `plugin-sdk/web-media` | Спільні допоміжні засоби завантаження віддалених/локальних медіа |
    | `plugin-sdk/zod` | Повторно експортований `zod` для споживачів Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Підшляхи пам’яті">
    | Підшлях | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/memory-core` | Поверхня допоміжних засобів вбудованого memory-core для manager/config/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Runtime facade для індексації/пошуку в пам’яті |
    | `plugin-sdk/memory-core-host-engine-foundation` | Експорти foundation engine хоста пам’яті |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Експорти embedding engine хоста пам’яті |
    | `plugin-sdk/memory-core-host-engine-qmd` | Експорти QMD engine хоста пам’яті |
    | `plugin-sdk/memory-core-host-engine-storage` | Експорти storage engine хоста пам’яті |
    | `plugin-sdk/memory-core-host-multimodal` | Допоміжні засоби мультимодального хоста пам’яті |
    | `plugin-sdk/memory-core-host-query` | Допоміжні засоби запитів хоста пам’яті |
    | `plugin-sdk/memory-core-host-secret` | Допоміжні засоби секретів хоста пам’яті |
    | `plugin-sdk/memory-core-host-events` | Допоміжні засоби журналу подій хоста пам’яті |
    | `plugin-sdk/memory-core-host-status` | Допоміжні засоби статусу хоста пам’яті |
    | `plugin-sdk/memory-core-host-runtime-cli` | Допоміжні засоби CLI runtime хоста пам’яті |
    | `plugin-sdk/memory-core-host-runtime-core` | Допоміжні засоби core runtime хоста пам’яті |
    | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні засоби файлів/runtime хоста пам’яті |
    | `plugin-sdk/memory-host-core` | Нейтральний щодо постачальника псевдонім для допоміжних засобів core runtime хоста пам’яті |
    | `plugin-sdk/memory-host-events` | Нейтральний щодо постачальника псевдонім для допоміжних засобів журналу подій хоста пам’яті |
    | `plugin-sdk/memory-host-files` | Нейтральний щодо постачальника псевдонім для допоміжних засобів файлів/runtime хоста пам’яті |
    | `plugin-sdk/memory-host-markdown` | Спільні допоміжні засоби managed-markdown для плагінів, суміжних із пам’яттю |
    | `plugin-sdk/memory-host-search` | Active memory runtime facade для доступу до search-manager |
    | `plugin-sdk/memory-host-status` | Нейтральний щодо постачальника псевдонім для допоміжних засобів статусу хоста пам’яті |
    | `plugin-sdk/memory-lancedb` | Поверхня допоміжних засобів вбудованого memory-lancedb |
  </Accordion>

  <Accordion title="Зарезервовані підшляхи допоміжних засобів вбудованих плагінів">
    | Сімейство | Поточні підшляхи | Призначення |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Допоміжні засоби підтримки вбудованого browser plugin (`browser-support` залишається barrel-файлом сумісності) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Поверхня допоміжних засобів/runtime для вбудованого Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Поверхня допоміжних засобів/runtime для вбудованого LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Поверхня допоміжних засобів для вбудованого IRC |
    | Допоміжні засоби, специфічні для каналів | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Шви сумісності / допоміжні засоби для вбудованих каналів |
    | Допоміжні засоби, специфічні для auth/плагіна | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Шви допоміжних засобів для вбудованих функцій/плагінів; `plugin-sdk/github-copilot-token` наразі експортує `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API реєстрації

Колбек `register(api)` отримує об’єкт `OpenClawPluginApi` з такими
методами:

### Реєстрація можливостей

| Метод                                            | Що він реєструє                 |
| ------------------------------------------------ | ------------------------------- |
| `api.registerProvider(...)`                      | Текстовий inference (LLM)       |
| `api.registerCliBackend(...)`                    | Локальний CLI-бекенд inference  |
| `api.registerChannel(...)`                       | Канал обміну повідомленнями     |
| `api.registerSpeechProvider(...)`                | Синтез text-to-speech / STT     |
| `api.registerRealtimeTranscriptionProvider(...)` | Потокова транскрипція в реальному часі |
| `api.registerRealtimeVoiceProvider(...)`         | Дуплексні голосові сеанси в реальному часі |
| `api.registerMediaUnderstandingProvider(...)`    | Аналіз зображень/аудіо/відео    |
| `api.registerImageGenerationProvider(...)`       | Генерація зображень             |
| `api.registerMusicGenerationProvider(...)`       | Генерація музики                |
| `api.registerVideoGenerationProvider(...)`       | Генерація відео                 |
| `api.registerWebFetchProvider(...)`              | Провайдер web fetch / scrape    |
| `api.registerWebSearchProvider(...)`             | Вебпошук                        |

### Інструменти та команди

| Метод                          | Що він реєструє                              |
| ------------------------------ | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Інструмент агента (обов’язковий або `{ optional: true }`) |
| `api.registerCommand(def)`      | Користувацька команда (оминає LLM)           |

### Інфраструктура

| Метод                                         | Що він реєструє                    |
| --------------------------------------------- | ---------------------------------- |
| `api.registerHook(events, handler, opts?)`    | Хук події                          |
| `api.registerHttpRoute(params)`               | HTTP endpoint шлюзу                |
| `api.registerGatewayMethod(name, handler)`    | RPC-метод шлюзу                    |
| `api.registerCli(registrar, opts?)`           | Підкоманда CLI                     |
| `api.registerService(service)`                | Фоновий сервіс                     |
| `api.registerInteractiveHandler(registration)` | Інтерактивний обробник            |
| `api.registerMemoryPromptSupplement(builder)` | Адитивний розділ prompt, суміжний із пам’яттю |
| `api.registerMemoryCorpusSupplement(adapter)` | Адитивний корпус пошуку/читання пам’яті |

Зарезервовані простори імен адміністрування ядра (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) завжди залишаються `operator.admin`, навіть якщо плагін
намагається призначити вужчу область дії методу шлюзу. Для методів, що
належать плагіну, надавайте перевагу специфічним для плагіна префіксам.

### Метадані реєстрації CLI

`api.registerCli(registrar, opts?)` приймає два типи метаданих верхнього рівня:

- `commands`: явні корені команд, якими володіє registrar
- `descriptors`: дескриптори команд часу парсингу, що використовуються для довідки кореневого CLI,
  маршрутизації та лінивої реєстрації CLI плагіна

Якщо ви хочете, щоб команда плагіна залишалася ліниво завантажуваною в
звичайному шляху кореневого CLI, надайте `descriptors`, які охоплюють кожен
корінь команди верхнього рівня, що експонується цим registrar.

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
        description: "Керування обліковими записами Matrix, верифікацією, пристроями та станом профілю",
        hasSubcommands: true,
      },
    ],
  },
);
```

Використовуйте лише `commands`, коли вам не потрібна лінива реєстрація
кореневого CLI. Цей eager-шлях сумісності залишається підтримуваним, але він
не встановлює placeholder-и на основі дескрипторів для лінивого завантаження
під час парсингу.

### Реєстрація CLI-бекенду

`api.registerCliBackend(...)` дає змогу плагіну володіти типовою конфігурацією
для локального AI CLI-бекенду, такого як `codex-cli`.

- `id` бекенду стає префіксом провайдера в model ref на кшталт `codex-cli/gpt-5`.
- `config` бекенду використовує ту саму форму, що й `agents.defaults.cliBackends.<id>`.
- Конфігурація користувача все одно має пріоритет. OpenClaw зливає `agents.defaults.cliBackends.<id>` поверх
  типового значення плагіна перед запуском CLI.
- Використовуйте `normalizeConfig`, коли бекенду потрібні переписування сумісності після злиття
  (наприклад, для нормалізації старих форм прапорців).

### Ексклюзивні слоти

| Метод                                     | Що він реєструє                                                                                                                                           |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Context engine (одночасно активний лише один). Колбек `assemble()` отримує `availableTools` і `citationsMode`, щоб engine міг адаптувати доповнення до prompt. |
| `api.registerMemoryCapability(capability)` | Уніфіковану можливість пам’яті                                                                                                                            |
| `api.registerMemoryPromptSection(builder)` | Побудовник розділу prompt пам’яті                                                                                                                         |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver плану flush пам’яті                                                                                                                              |
| `api.registerMemoryRuntime(runtime)`       | Адаптер runtime пам’яті                                                                                                                                   |

### Адаптери embedding для пам’яті

| Метод                                         | Що він реєструє                              |
| --------------------------------------------- | -------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Адаптер embedding пам’яті для активного плагіна |

- `registerMemoryCapability` — рекомендований API ексклюзивного memory-plugin.
- `registerMemoryCapability` також може експонувати `publicArtifacts.listArtifacts(...)`,
  щоб супутні плагіни могли споживати експортовані артефакти пам’яті через
  `openclaw/plugin-sdk/memory-host-core` замість звернення до приватного
  макета конкретного плагіна пам’яті.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` і
  `registerMemoryRuntime` — це застаріло-сумісні API ексклюзивного memory-plugin.
- `registerMemoryEmbeddingProvider` дає змогу активному memory-plugin зареєструвати
  один або кілька id адаптерів embedding (наприклад, `openai`, `gemini` або
  користувацький id, визначений плагіном).
- Конфігурація користувача, така як `agents.defaults.memorySearch.provider` і
  `agents.defaults.memorySearch.fallback`, визначається відносно цих зареєстрованих
  id адаптерів.

### Події та життєвий цикл

| Метод                                       | Що він робить                |
| ------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`          | Типізований хук життєвого циклу |
| `api.onConversationBindingResolved(handler)` | Колбек визначення прив’язки розмови |

### Семантика рішень хуків

- `before_tool_call`: повернення `{ block: true }` є термінальним. Щойно якийсь обробник встановить його, обробники з нижчим пріоритетом пропускаються.
- `before_tool_call`: повернення `{ block: false }` трактується як відсутність рішення (так само, як і пропуск `block`), а не як перевизначення.
- `before_install`: повернення `{ block: true }` є термінальним. Щойно якийсь обробник встановить його, обробники з нижчим пріоритетом пропускаються.
- `before_install`: повернення `{ block: false }` трактується як відсутність рішення (так само, як і пропуск `block`), а не як перевизначення.
- `reply_dispatch`: повернення `{ handled: true, ... }` є термінальним. Щойно якийсь обробник візьме dispatch на себе, обробники з нижчим пріоритетом і типовий шлях dispatch моделі пропускаються.
- `message_sending`: повернення `{ cancel: true }` є термінальним. Щойно якийсь обробник встановить його, обробники з нижчим пріоритетом пропускаються.
- `message_sending`: повернення `{ cancel: false }` трактується як відсутність рішення (так само, як і пропуск `cancel`), а не як перевизначення.

### Поля об’єкта API

| Поле                     | Тип                       | Опис                                                                                      |
| ------------------------ | ------------------------- | ----------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID плагіна                                                                                |
| `api.name`               | `string`                  | Відображувана назва                                                                       |
| `api.version`            | `string?`                 | Версія плагіна (необов’язково)                                                            |
| `api.description`        | `string?`                 | Опис плагіна (необов’язково)                                                              |
| `api.source`             | `string`                  | Шлях до джерела плагіна                                                                   |
| `api.rootDir`            | `string?`                 | Кореневий каталог плагіна (необов’язково)                                                 |
| `api.config`             | `OpenClawConfig`          | Поточний знімок конфігурації (активний знімок runtime у пам’яті, якщо доступний)         |
| `api.pluginConfig`       | `Record<string, unknown>` | Конфігурація, специфічна для плагіна, з `plugins.entries.<id>.config`                     |
| `api.runtime`            | `PluginRuntime`           | [Допоміжні засоби runtime](/uk/plugins/sdk-runtime)                                          |
| `api.logger`             | `PluginLogger`            | Логер з областю дії (`debug`, `info`, `warn`, `error`)                                    |
| `api.registrationMode`   | `PluginRegistrationMode`  | Поточний режим завантаження; `"setup-runtime"` — це полегшене вікно запуску/налаштування перед повним entry |
| `api.resolvePath(input)` | `(string) => string`      | Визначення шляху відносно кореня плагіна                                                  |

## Внутрішнє правило модулів

Усередині вашого плагіна використовуйте локальні barrel-файли для внутрішніх імпортів:

```
my-plugin/
  api.ts            # Публічні експорти для зовнішніх споживачів
  runtime-api.ts    # Експорти runtime лише для внутрішнього використання
  index.ts          # Точка входу плагіна
  setup-entry.ts    # Полегшений вхід лише для налаштування (необов’язково)
```

<Warning>
  Ніколи не імпортуйте власний плагін через `openclaw/plugin-sdk/<your-plugin>`
  у production-коді. Спрямовуйте внутрішні імпорти через `./api.ts` або
  `./runtime-api.ts`. Шлях SDK — це лише зовнішній контракт.
</Warning>

Публічні поверхні вбудованих плагінів, що завантажуються через facade
(`api.ts`, `runtime-api.ts`, `index.ts`, `setup-entry.ts` та подібні публічні
entry-файли), тепер віддають перевагу активному знімку конфігурації runtime,
коли OpenClaw уже запущено. Якщо знімка runtime ще немає, вони переходять до
резервного використання визначеного файла конфігурації на диску.

Плагіни провайдерів також можуть експонувати вузький локальний barrel-файл
контракту плагіна, коли допоміжний засіб навмисно є специфічним для провайдера
і ще не належить до загального підшляху SDK. Поточний вбудований приклад:
провайдер Anthropic тримає свої допоміжні засоби потоку Claude у власному
публічному шві `api.ts` / `contract-api.ts` замість просування логіки
beta-header Anthropic і `service_tier` до загального контракту
`plugin-sdk/*`.

Інші поточні вбудовані приклади:

- `@openclaw/openai-provider`: `api.ts` експортує побудовники провайдерів,
  допоміжні засоби типових моделей і побудовники провайдерів realtime
- `@openclaw/openrouter-provider`: `api.ts` експортує побудовник провайдера та
  допоміжні засоби онбордингу/конфігурації

<Warning>
  Production-код розширень також має уникати імпортів
  `openclaw/plugin-sdk/<other-plugin>`. Якщо допоміжний засіб справді є спільним,
  перенесіть його до нейтрального підшляху SDK, такого як
  `openclaw/plugin-sdk/speech`, `.../provider-model-shared` або іншої
  поверхні, орієнтованої на можливості, замість зв’язування двох плагінів між собою.
</Warning>

## Пов’язане

- [Entry Points](/uk/plugins/sdk-entrypoints) — параметри `definePluginEntry` і `defineChannelPluginEntry`
- [Runtime Helpers](/uk/plugins/sdk-runtime) — повний довідник простору імен `api.runtime`
- [Setup and Config](/uk/plugins/sdk-setup) — пакування, маніфести, схеми конфігурації
- [Testing](/uk/plugins/sdk-testing) — утиліти тестування та правила lint
- [SDK Migration](/uk/plugins/sdk-migration) — міграція з застарілих поверхонь
- [Plugin Internals](/uk/plugins/architecture) — поглиблена архітектура та модель можливостей
