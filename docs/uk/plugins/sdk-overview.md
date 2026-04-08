---
read_when:
    - Вам потрібно знати, з якого підшляху SDK імпортувати
    - Вам потрібен довідник для всіх методів реєстрації в OpenClawPluginApi
    - Ви шукаєте конкретний експорт SDK
sidebarTitle: SDK Overview
summary: Карта імпорту, довідник API реєстрації та архітектура SDK
title: Огляд Plugin SDK
x-i18n:
    generated_at: "2026-04-08T06:30:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: a9141dc0303c91974fe693ce48ad9f7dc4179418ae262a96011ad565aae87d21
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Огляд Plugin SDK

Plugin SDK — це типізований контракт між плагінами та ядром. Ця сторінка є
довідником про **що імпортувати** і **що можна реєструвати**.

<Tip>
  **Шукаєте практичний посібник?**
  - Перший плагін? Почніть із [Getting Started](/uk/plugins/building-plugins)
  - Плагін каналу? Див. [Channel Plugins](/uk/plugins/sdk-channel-plugins)
  - Плагін провайдера? Див. [Provider Plugins](/uk/plugins/sdk-provider-plugins)
</Tip>

## Угода про імпорт

Завжди імпортуйте з конкретного підшляху:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Кожен підшлях — це невеликий самодостатній модуль. Це забезпечує швидкий
запуск і запобігає проблемам із циклічними залежностями. Для специфічних для
каналів допоміжних функцій entry/build надавайте перевагу
`openclaw/plugin-sdk/channel-core`; `openclaw/plugin-sdk/core` залишайте для
ширшої узагальненої поверхні та спільних допоміжних функцій, таких як
`buildChannelConfigSchema`.

Не додавайте й не використовуйте зручні шари з іменами провайдерів, такі як
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, або допоміжні
шари з брендуванням каналів. Вбудовані плагіни мають поєднувати узагальнені
підшляхи SDK у власних barrel-файлах `api.ts` або `runtime-api.ts`, а ядро
має або використовувати ці локальні для плагіна barrel-файли, або додавати
вузький узагальнений контракт SDK, коли потреба справді є міжканальною.

Згенерована карта експортів усе ще містить невеликий набір допоміжних шарів
для вбудованих плагінів, таких як `plugin-sdk/feishu`,
`plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup` і
`plugin-sdk/matrix*`. Ці підшляхи існують лише для підтримки вбудованих
плагінів і сумісності; їх навмисно не включено до загальної таблиці нижче, і
вони не є рекомендованим шляхом імпорту для нових сторонніх плагінів.

## Довідник підшляхів

Найуживаніші підшляхи, згруповані за призначенням. Згенерований повний список із
понад 200 підшляхів міститься у `scripts/lib/plugin-sdk-entrypoints.json`.

Зарезервовані допоміжні підшляхи для вбудованих плагінів усе ще з’являються в
цьому згенерованому списку. Вважайте їх деталями реалізації/поверхнями
сумісності, якщо лише якась сторінка документації явно не проголошує одну з
них публічною.

### Точка входу плагіна

| Subpath                     | Ключові експорти                                                                                                                     |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                  |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                     |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                    |

<AccordionGroup>
  <Accordion title="Підшляхи каналів">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Експорт кореневої схеми Zod для `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Спільні допоміжні функції майстра налаштування, запити списку дозволених, побудовники статусу налаштування |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Допоміжні функції багатoакаунтної конфігурації/контролю дій, допоміжні функції резервного вибору акаунта за замовчуванням |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, допоміжні функції нормалізації ID акаунта |
    | `plugin-sdk/account-resolution` | Пошук акаунта + допоміжні функції резервного вибору за замовчуванням |
    | `plugin-sdk/account-helpers` | Вузькі допоміжні функції для списку акаунтів/дій з акаунтами |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Типи схеми конфігурації каналу |
    | `plugin-sdk/telegram-command-config` | Допоміжні функції нормалізації/валідації кастомних команд Telegram з резервною підтримкою вбудованого контракту |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Спільні допоміжні функції побудови вхідних маршрутів та envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Спільні допоміжні функції запису та диспетчеризації вхідних даних |
    | `plugin-sdk/messaging-targets` | Допоміжні функції парсингу/зіставлення цілей |
    | `plugin-sdk/outbound-media` | Спільні допоміжні функції завантаження вихідних медіа |
    | `plugin-sdk/outbound-runtime` | Допоміжні функції для вихідної ідентичності/делегування надсилання |
    | `plugin-sdk/thread-bindings-runtime` | Життєвий цикл прив’язок потоків та допоміжні функції адаптерів |
    | `plugin-sdk/agent-media-payload` | Застарілий побудовник медіапейлоадів агента |
    | `plugin-sdk/conversation-runtime` | Допоміжні функції прив’язки розмов/потоків, pairing і налаштованих прив’язок |
    | `plugin-sdk/runtime-config-snapshot` | Допоміжна функція знімка конфігурації runtime |
    | `plugin-sdk/runtime-group-policy` | Допоміжні функції визначення політики груп у runtime |
    | `plugin-sdk/channel-status` | Спільні допоміжні функції знімка/зведення статусу каналу |
    | `plugin-sdk/channel-config-primitives` | Вузькі примітиви схеми конфігурації каналу |
    | `plugin-sdk/channel-config-writes` | Допоміжні функції авторизації запису конфігурації каналу |
    | `plugin-sdk/channel-plugin-common` | Спільні prelude-експорти плагінів каналів |
    | `plugin-sdk/allowlist-config-edit` | Допоміжні функції редагування/читання конфігурації списку дозволених |
    | `plugin-sdk/group-access` | Спільні допоміжні функції рішень щодо доступу до груп |
    | `plugin-sdk/direct-dm` | Спільні допоміжні функції авторизації/захисту прямих DM |
    | `plugin-sdk/interactive-runtime` | Допоміжні функції нормалізації/зведення інтерактивних пейлоадів відповідей |
    | `plugin-sdk/channel-inbound` | Допоміжні функції debounce для вхідних повідомлень, зіставлення згадок, політики згадок і допоміжні функції envelope |
    | `plugin-sdk/channel-send-result` | Типи результатів відповіді |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Допоміжні функції парсингу/зіставлення цілей |
    | `plugin-sdk/channel-contract` | Типи контракту каналу |
    | `plugin-sdk/channel-feedback` | Підключення зворотного зв’язку/реакцій |
    | `plugin-sdk/channel-secret-runtime` | Вузькі допоміжні функції secret-контракту, такі як `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, і типи secret-цілей |
  </Accordion>

  <Accordion title="Підшляхи провайдерів">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Кураторські допоміжні функції налаштування локальних/self-hosted провайдерів |
    | `plugin-sdk/self-hosted-provider-setup` | Сфокусовані допоміжні функції налаштування self-hosted провайдерів, сумісних з OpenAI |
    | `plugin-sdk/cli-backend` | Значення за замовчуванням для CLI backend + константи watchdog |
    | `plugin-sdk/provider-auth-runtime` | Допоміжні функції визначення API-ключів у runtime для плагінів провайдерів |
    | `plugin-sdk/provider-auth-api-key` | Допоміжні функції онбордингу/запису профілю API-ключів, такі як `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Стандартний побудовник результатів OAuth-автентифікації |
    | `plugin-sdk/provider-auth-login` | Спільні інтерактивні допоміжні функції входу для плагінів провайдерів |
    | `plugin-sdk/provider-env-vars` | Допоміжні функції пошуку env var для автентифікації провайдерів |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні побудовники політик replay, допоміжні функції endpoint провайдера та нормалізації ID моделей, такі як `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Узагальнені допоміжні функції для HTTP/можливостей endpoint провайдера |
    | `plugin-sdk/provider-web-fetch-contract` | Вузькі допоміжні функції контракту конфігурації/вибору web-fetch, такі як `enablePluginInConfig` і `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Допоміжні функції реєстрації/кешування провайдерів web-fetch |
    | `plugin-sdk/provider-web-search-contract` | Вузькі допоміжні функції контракту конфігурації/облікових даних web-search, такі як `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` і scoped setter/getter для облікових даних |
    | `plugin-sdk/provider-web-search` | Допоміжні функції реєстрації/кешування/runtime для провайдерів web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схеми Gemini + діагностика, а також допоміжні функції сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` та подібні |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи stream wrapper і спільні допоміжні wrapper-функції для Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Допоміжні функції внесення патчів у конфігурацію онбордингу |
    | `plugin-sdk/global-singleton` | Допоміжні функції process-local singleton/map/cache |
  </Accordion>

  <Accordion title="Підшляхи автентифікації та безпеки">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, допоміжні функції реєстру команд, допоміжні функції авторизації відправника |
    | `plugin-sdk/approval-auth-runtime` | Допоміжні функції визначення approver і авторизації дій у тому самому чаті |
    | `plugin-sdk/approval-client-runtime` | Допоміжні функції профілів/фільтрів підтвердження native exec |
    | `plugin-sdk/approval-delivery-runtime` | Адаптери можливостей/доставки native confirmation |
    | `plugin-sdk/approval-gateway-runtime` | Спільна допоміжна функція визначення approval gateway |
    | `plugin-sdk/approval-handler-adapter-runtime` | Полегшені допоміжні функції завантаження native approval adapter для гарячих точок входу каналів |
    | `plugin-sdk/approval-handler-runtime` | Ширші допоміжні функції runtime для approval handler; віддавайте перевагу вужчим adapter/gateway seams, якщо їх достатньо |
    | `plugin-sdk/approval-native-runtime` | Допоміжні функції цілей native approval + прив’язки акаунтів |
    | `plugin-sdk/approval-reply-runtime` | Допоміжні функції пейлоадів відповідей exec/plugin approval |
    | `plugin-sdk/command-auth-native` | Допоміжні функції native command auth + native session-target |
    | `plugin-sdk/command-detection` | Спільні допоміжні функції виявлення команд |
    | `plugin-sdk/command-surface` | Допоміжні функції нормалізації тіла команди та command-surface |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Вузькі допоміжні функції збирання secret-контрактів для secret-surface каналів/плагінів |
    | `plugin-sdk/secret-ref-runtime` | Вузькі допоміжні функції `coerceSecretRef` і типізації SecretRef для парсингу secret-контрактів/конфігурації |
    | `plugin-sdk/security-runtime` | Спільні допоміжні функції довіри, DM-gating, зовнішнього вмісту та збирання секретів |
    | `plugin-sdk/ssrf-policy` | Допоміжні функції політики SSRF для host allowlist і приватних мереж |
    | `plugin-sdk/ssrf-runtime` | Допоміжні функції pinned-dispatcher, fetch із захистом SSRF і політики SSRF |
    | `plugin-sdk/secret-input` | Допоміжні функції парсингу secret input |
    | `plugin-sdk/webhook-ingress` | Допоміжні функції запитів/цілей webhook |
    | `plugin-sdk/webhook-request-guards` | Допоміжні функції розміру тіла запиту/тайм-ауту |
  </Accordion>

  <Accordion title="Підшляхи runtime та сховища">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/runtime` | Широкі допоміжні функції runtime/логування/резервного копіювання/встановлення плагінів |
    | `plugin-sdk/runtime-env` | Вузькі допоміжні функції runtime env, logger, timeout, retry і backoff |
    | `plugin-sdk/channel-runtime-context` | Узагальнені допоміжні функції реєстрації та пошуку channel runtime-context |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Спільні допоміжні функції для команд/хуків/http/interactive плагінів |
    | `plugin-sdk/hook-runtime` | Спільні допоміжні функції pipeline для webhook/internal hook |
    | `plugin-sdk/lazy-runtime` | Допоміжні функції lazy-імпорту та прив’язки runtime, такі як `createLazyRuntimeModule`, `createLazyRuntimeMethod` і `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Допоміжні функції виконання процесів |
    | `plugin-sdk/cli-runtime` | Допоміжні функції форматування, очікування та версій для CLI |
    | `plugin-sdk/gateway-runtime` | Допоміжні функції клієнта gateway і patch статусу каналу |
    | `plugin-sdk/config-runtime` | Допоміжні функції завантаження/запису конфігурації |
    | `plugin-sdk/telegram-command-config` | Допоміжні функції нормалізації назв/описів команд Telegram і перевірки дублікатів/конфліктів, навіть коли поверхня вбудованого контракту Telegram недоступна |
    | `plugin-sdk/approval-runtime` | Допоміжні функції exec/plugin approval, побудовники approval-capability, допоміжні функції auth/profile, native routing/runtime |
    | `plugin-sdk/reply-runtime` | Спільні допоміжні функції runtime для вхідних повідомлень/відповідей, chunking, dispatch, heartbeat, planner відповідей |
    | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні функції dispatch/finalize відповіді |
    | `plugin-sdk/reply-history` | Спільні допоміжні функції коротковіконної історії відповідей, такі як `buildHistoryContext`, `recordPendingHistoryEntry` і `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Вузькі допоміжні функції chunking для тексту/markdown |
    | `plugin-sdk/session-store-runtime` | Допоміжні функції шляху сховища сесій + updated-at |
    | `plugin-sdk/state-paths` | Допоміжні функції шляхів до каталогів state/OAuth |
    | `plugin-sdk/routing` | Допоміжні функції маршруту/ключа сесії/прив’язки акаунта, такі як `resolveAgentRoute`, `buildAgentSessionKey` і `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Спільні допоміжні функції зведення статусу каналу/акаунта, значення runtime-state за замовчуванням і допоміжні функції метаданих проблем |
    | `plugin-sdk/target-resolver-runtime` | Спільні допоміжні функції визначення цілей |
    | `plugin-sdk/string-normalization-runtime` | Допоміжні функції нормалізації slug/рядків |
    | `plugin-sdk/request-url` | Витягування рядкових URL із fetch/request-подібних вхідних даних |
    | `plugin-sdk/run-command` | Runner команд із таймуванням і нормалізованими результатами stdout/stderr |
    | `plugin-sdk/param-readers` | Загальні читачі параметрів tool/CLI |
    | `plugin-sdk/tool-send` | Витягування канонічних полів цілей надсилання з аргументів tool |
    | `plugin-sdk/temp-path` | Спільні допоміжні функції шляхів до тимчасових завантажень |
    | `plugin-sdk/logging-core` | Допоміжні функції subsystem logger і redaction |
    | `plugin-sdk/markdown-table-runtime` | Допоміжні функції режиму таблиць Markdown |
    | `plugin-sdk/json-store` | Невеликі допоміжні функції читання/запису стану JSON |
    | `plugin-sdk/file-lock` | Допоміжні функції реентерабельного file-lock |
    | `plugin-sdk/persistent-dedupe` | Допоміжні функції дискового кешу дедуплікації |
    | `plugin-sdk/acp-runtime` | Допоміжні функції ACP runtime/session і reply-dispatch |
    | `plugin-sdk/agent-config-primitives` | Вузькі примітиви схеми конфігурації runtime агента |
    | `plugin-sdk/boolean-param` | Гнучкий читач булевих параметрів |
    | `plugin-sdk/dangerous-name-runtime` | Допоміжні функції визначення збігів небезпечних назв |
    | `plugin-sdk/device-bootstrap` | Допоміжні функції bootstrap пристрою та токенів pairing |
    | `plugin-sdk/extension-shared` | Спільні примітиви для passive-channel, status і ambient proxy helper |
    | `plugin-sdk/models-provider-runtime` | Допоміжні функції відповіді провайдера/команди `/models` |
    | `plugin-sdk/skill-commands-runtime` | Допоміжні функції перелічення команд Skills |
    | `plugin-sdk/native-command-registry` | Допоміжні функції реєстру/build/serialize native command |
    | `plugin-sdk/provider-zai-endpoint` | Допоміжні функції виявлення endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Допоміжні функції системних подій/heartbeat |
    | `plugin-sdk/collection-runtime` | Невеликі допоміжні функції обмеженого кешу |
    | `plugin-sdk/diagnostic-runtime` | Допоміжні функції діагностичних прапорців і подій |
    | `plugin-sdk/error-runtime` | Допоміжні функції графа помилок, форматування, спільної класифікації помилок, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Допоміжні функції обгорнутого fetch, proxy і pinned lookup |
    | `plugin-sdk/host-runtime` | Допоміжні функції нормалізації hostname і SCP host |
    | `plugin-sdk/retry-runtime` | Допоміжні функції конфігурації retry і виконання retry |
    | `plugin-sdk/agent-runtime` | Допоміжні функції каталогу/ідентичності/workspace агента |
    | `plugin-sdk/directory-runtime` | Запит/дедуплікація каталогів на основі конфігурації |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Підшляхи можливостей і тестування">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Спільні допоміжні функції fetch/transform/store для медіа, а також побудовники медіапейлоадів |
    | `plugin-sdk/media-generation-runtime` | Спільні допоміжні функції failover для генерації медіа, вибору кандидатів і повідомлень про відсутні моделі |
    | `plugin-sdk/media-understanding` | Типи провайдерів media understanding, а також provider-facing експорти допоміжних функцій для зображень/аудіо |
    | `plugin-sdk/text-runtime` | Спільні допоміжні функції тексту/markdown/логування, такі як видалення видимого для асистента тексту, допоміжні функції render/chunking/table для markdown, redaction helpers, directive-tag helpers і safe-text utilities |
    | `plugin-sdk/text-chunking` | Допоміжна функція chunking вихідного тексту |
    | `plugin-sdk/speech` | Типи провайдерів мовлення, а також provider-facing допоміжні функції directive, registry і validation |
    | `plugin-sdk/speech-core` | Спільні типи провайдерів мовлення, допоміжні функції registry, directive і normalization |
    | `plugin-sdk/realtime-transcription` | Типи провайдерів realtime transcription і допоміжні функції registry |
    | `plugin-sdk/realtime-voice` | Типи провайдерів realtime voice і допоміжні функції registry |
    | `plugin-sdk/image-generation` | Типи провайдерів генерації зображень |
    | `plugin-sdk/image-generation-core` | Спільні типи генерації зображень, допоміжні функції failover, auth і registry |
    | `plugin-sdk/music-generation` | Типи провайдерів/запитів/результатів генерації музики |
    | `plugin-sdk/music-generation-core` | Спільні типи генерації музики, допоміжні функції failover, пошуку провайдера і парсингу model-ref |
    | `plugin-sdk/video-generation` | Типи провайдерів/запитів/результатів генерації відео |
    | `plugin-sdk/video-generation-core` | Спільні типи генерації відео, допоміжні функції failover, пошуку провайдера і парсингу model-ref |
    | `plugin-sdk/webhook-targets` | Допоміжні функції реєстру webhook targets і встановлення маршрутів |
    | `plugin-sdk/webhook-path` | Допоміжні функції нормалізації шляху webhook |
    | `plugin-sdk/web-media` | Спільні допоміжні функції завантаження віддалених/локальних медіа |
    | `plugin-sdk/zod` | Повторний експорт `zod` для споживачів plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Підшляхи пам’яті">
    | Subpath | Ключові експорти |
    | --- | --- |
    | `plugin-sdk/memory-core` | Поверхня допоміжних функцій вбудованого memory-core для manager/config/file/CLI helpers |
    | `plugin-sdk/memory-core-engine-runtime` | Фасад runtime для memory index/search |
    | `plugin-sdk/memory-core-host-engine-foundation` | Експорти foundation engine для memory host |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Експорти embedding engine для memory host |
    | `plugin-sdk/memory-core-host-engine-qmd` | Експорти QMD engine для memory host |
    | `plugin-sdk/memory-core-host-engine-storage` | Експорти storage engine для memory host |
    | `plugin-sdk/memory-core-host-multimodal` | Допоміжні функції multimodal для memory host |
    | `plugin-sdk/memory-core-host-query` | Допоміжні функції query для memory host |
    | `plugin-sdk/memory-core-host-secret` | Допоміжні функції secret для memory host |
    | `plugin-sdk/memory-core-host-events` | Допоміжні функції журналу подій для memory host |
    | `plugin-sdk/memory-core-host-status` | Допоміжні функції статусу для memory host |
    | `plugin-sdk/memory-core-host-runtime-cli` | Допоміжні функції runtime CLI для memory host |
    | `plugin-sdk/memory-core-host-runtime-core` | Допоміжні функції core runtime для memory host |
    | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні функції файлів/runtime для memory host |
    | `plugin-sdk/memory-host-core` | Вендорно-нейтральний псевдонім для допоміжних функцій core runtime memory host |
    | `plugin-sdk/memory-host-events` | Вендорно-нейтральний псевдонім для допоміжних функцій журналу подій memory host |
    | `plugin-sdk/memory-host-files` | Вендорно-нейтральний псевдонім для допоміжних функцій файлів/runtime memory host |
    | `plugin-sdk/memory-host-markdown` | Спільні допоміжні функції managed-markdown для плагінів, суміжних із пам’яттю |
    | `plugin-sdk/memory-host-search` | Активний фасад runtime пам’яті для доступу до search-manager |
    | `plugin-sdk/memory-host-status` | Вендорно-нейтральний псевдонім для допоміжних функцій статусу memory host |
    | `plugin-sdk/memory-lancedb` | Поверхня допоміжних функцій вбудованого memory-lancedb |
  </Accordion>

  <Accordion title="Зарезервовані допоміжні підшляхи вбудованих компонентів">
    | Family | Поточні підшляхи | Призначення |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Допоміжні функції підтримки вбудованого browser plugin (`browser-support` залишається barrel-файлом сумісності) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Поверхня допоміжних функцій/runtime для вбудованого Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Поверхня допоміжних функцій/runtime для вбудованого LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Поверхня допоміжних функцій для вбудованого IRC |
    | Допоміжні функції, специфічні для каналів | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Поверхні сумісності/допоміжних функцій для вбудованих каналів |
    | Допоміжні функції, специфічні для auth/плагінів | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Допоміжні шари для вбудованих можливостей/плагінів; `plugin-sdk/github-copilot-token` наразі експортує `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API реєстрації

Зворотний виклик `register(api)` отримує об’єкт `OpenClawPluginApi` з такими
методами:

### Реєстрація можливостей

| Method                                           | Що реєструє                    |
| ------------------------------------------------ | ------------------------------ |
| `api.registerProvider(...)`                      | Текстовий inference (LLM)      |
| `api.registerCliBackend(...)`                    | Локальний backend inference CLI |
| `api.registerChannel(...)`                       | Канал обміну повідомленнями    |
| `api.registerSpeechProvider(...)`                | Синтез text-to-speech / STT    |
| `api.registerRealtimeTranscriptionProvider(...)` | Потокова realtime transcription |
| `api.registerRealtimeVoiceProvider(...)`         | Двосторонні realtime voice sessions |
| `api.registerMediaUnderstandingProvider(...)`    | Аналіз зображень/аудіо/відео   |
| `api.registerImageGenerationProvider(...)`       | Генерація зображень            |
| `api.registerMusicGenerationProvider(...)`       | Генерація музики               |
| `api.registerVideoGenerationProvider(...)`       | Генерація відео                |
| `api.registerWebFetchProvider(...)`              | Провайдер web fetch / scrape   |
| `api.registerWebSearchProvider(...)`             | Вебпошук                       |

### Інструменти й команди

| Method                          | Що реєструє                                   |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Інструмент агента (обов’язковий або `{ optional: true }`) |
| `api.registerCommand(def)`      | Користувацьку команду (оминає LLM)            |

### Інфраструктура

| Method                                         | Що реєструє                            |
| ---------------------------------------------- | -------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Хук подій                              |
| `api.registerHttpRoute(params)`                | HTTP endpoint gateway                  |
| `api.registerGatewayMethod(name, handler)`     | RPC-метод gateway                      |
| `api.registerCli(registrar, opts?)`            | Підкоманду CLI                         |
| `api.registerService(service)`                 | Фоновий сервіс                         |
| `api.registerInteractiveHandler(registration)` | Інтерактивний handler                  |
| `api.registerMemoryPromptSupplement(builder)`  | Додатковий розділ prompt, суміжний із пам’яттю |
| `api.registerMemoryCorpusSupplement(adapter)`  | Додатковий корпус пошуку/читання пам’яті |

Зарезервовані простори імен адміністратора ядра (`config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`) завжди залишаються `operator.admin`, навіть якщо
плагін намагається призначити вужчу область видимості для методу gateway.
Для методів, що належать плагіну, надавайте перевагу префіксам, специфічним
для цього плагіна.

### Метадані реєстрації CLI

`api.registerCli(registrar, opts?)` приймає два типи метаданих верхнього рівня:

- `commands`: явні корені команд, якими володіє registrar
- `descriptors`: дескриптори команд часу парсингу, що використовуються для
  довідки кореневого CLI, маршрутизації та лінивої реєстрації CLI плагінів

Якщо ви хочете, щоб команда плагіна залишалася ліниво завантажуваною у
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
        description: "Керування акаунтами Matrix, верифікацією, пристроями та станом профілю",
        hasSubcommands: true,
      },
    ],
  },
);
```

Використовуйте лише `commands`, якщо вам не потрібна лінива реєстрація
кореневого CLI. Цей eager-шлях сумісності все ще підтримується, але він не
встановлює placeholder-елементи з підтримкою descriptor для лінивого
завантаження під час парсингу.

### Реєстрація CLI backend

`api.registerCliBackend(...)` дає змогу плагіну володіти конфігурацією за
замовчуванням для локального AI CLI backend, такого як `codex-cli`.

- `id` backend стає префіксом провайдера в model ref, таких як `codex-cli/gpt-5`.
- `config` backend використовує ту саму форму, що й `agents.defaults.cliBackends.<id>`.
- Конфігурація користувача все одно має пріоритет. OpenClaw об’єднує `agents.defaults.cliBackends.<id>` поверх
  значення плагіна за замовчуванням перед запуском CLI.
- Використовуйте `normalizeConfig`, коли backend потребує перезаписів сумісності після об’єднання
  (наприклад, для нормалізації старих форм прапорців).

### Ексклюзивні слоти

| Method                                     | Що реєструє                                                                                                                                                 |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Механізм контексту (одночасно активний лише один). Зворотний виклик `assemble()` отримує `availableTools` і `citationsMode`, щоб механізм міг адаптувати додавання до prompt. |
| `api.registerMemoryCapability(capability)` | Уніфіковану можливість пам’яті                                                                                                                               |
| `api.registerMemoryPromptSection(builder)` | Побудовник розділу prompt для пам’яті                                                                                                                       |
| `api.registerMemoryFlushPlan(resolver)`    | Визначник плану очищення пам’яті                                                                                                                            |
| `api.registerMemoryRuntime(runtime)`       | Адаптер runtime пам’яті                                                                                                                                     |

### Адаптери вбудовування пам’яті

| Method                                         | Що реєструє                                      |
| ---------------------------------------------- | ------------------------------------------------ |
| `api.registerMemoryEmbeddingProvider(adapter)` | Адаптер вбудовування пам’яті для активного плагіна |

- `registerMemoryCapability` — це рекомендований API ексклюзивного плагіна пам’яті.
- `registerMemoryCapability` також може експонувати `publicArtifacts.listArtifacts(...)`,
  щоб допоміжні плагіни могли споживати експортовані артефакти пам’яті через
  `openclaw/plugin-sdk/memory-host-core` замість доступу до приватного
  макета конкретного плагіна пам’яті.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` і
  `registerMemoryRuntime` — це застарілі, але сумісні API ексклюзивних
  плагінів пам’яті.
- `registerMemoryEmbeddingProvider` дозволяє активному плагіну пам’яті
  реєструвати один або кілька ID адаптерів вбудовування (наприклад `openai`,
  `gemini` або власний ID, визначений плагіном).
- Конфігурація користувача, така як `agents.defaults.memorySearch.provider` і
  `agents.defaults.memorySearch.fallback`, визначається відносно цих
  зареєстрованих ID адаптерів.

### Події та життєвий цикл

| Method                                       | Що робить                    |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | Типізований хук життєвого циклу |
| `api.onConversationBindingResolved(handler)` | Зворотний виклик прив’язки розмови |

### Семантика рішень хуків

- `before_tool_call`: повернення `{ block: true }` є термінальним. Щойно будь-який handler встановлює його, handlers із нижчим пріоритетом пропускаються.
- `before_tool_call`: повернення `{ block: false }` вважається відсутністю рішення (так само, як пропуск `block`), а не перевизначенням.
- `before_install`: повернення `{ block: true }` є термінальним. Щойно будь-який handler встановлює його, handlers із нижчим пріоритетом пропускаються.
- `before_install`: повернення `{ block: false }` вважається відсутністю рішення (так само, як пропуск `block`), а не перевизначенням.
- `reply_dispatch`: повернення `{ handled: true, ... }` є термінальним. Щойно будь-який handler заявляє диспетчеризацію, handlers із нижчим пріоритетом і типовий шлях диспетчеризації моделі пропускаються.
- `message_sending`: повернення `{ cancel: true }` є термінальним. Щойно будь-який handler встановлює його, handlers із нижчим пріоритетом пропускаються.
- `message_sending`: повернення `{ cancel: false }` вважається відсутністю рішення (так само, як пропуск `cancel`), а не перевизначенням.

### Поля об’єкта API

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID плагіна                                                                                  |
| `api.name`               | `string`                  | Відображувана назва                                                                         |
| `api.version`            | `string?`                 | Версія плагіна (необов’язково)                                                              |
| `api.description`        | `string?`                 | Опис плагіна (необов’язково)                                                                |
| `api.source`             | `string`                  | Шлях до джерела плагіна                                                                     |
| `api.rootDir`            | `string?`                 | Кореневий каталог плагіна (необов’язково)                                                   |
| `api.config`             | `OpenClawConfig`          | Поточний знімок конфігурації (активний знімок runtime у пам’яті, якщо доступний)           |
| `api.pluginConfig`       | `Record<string, unknown>` | Специфічна для плагіна конфігурація з `plugins.entries.<id>.config`                         |
| `api.runtime`            | `PluginRuntime`           | [Допоміжні функції runtime](/uk/plugins/sdk-runtime)                                           |
| `api.logger`             | `PluginLogger`            | Локалізований logger (`debug`, `info`, `warn`, `error`)                                     |
| `api.registrationMode`   | `PluginRegistrationMode`  | Поточний режим завантаження; `"setup-runtime"` — це полегшене вікно запуску/налаштування до повного entry |
| `api.resolvePath(input)` | `(string) => string`      | Визначити шлях відносно кореня плагіна                                                      |

## Угода про внутрішні модулі

Усередині вашого плагіна використовуйте локальні barrel-файли для внутрішніх імпортів:

```
my-plugin/
  api.ts            # Публічні експорти для зовнішніх споживачів
  runtime-api.ts    # Лише внутрішні runtime-експорти
  index.ts          # Точка входу плагіна
  setup-entry.ts    # Полегшена точка входу лише для налаштування (необов’язково)
```

<Warning>
  Ніколи не імпортуйте власний плагін через `openclaw/plugin-sdk/<your-plugin>`
  у production-коді. Спрямовуйте внутрішні імпорти через `./api.ts` або
  `./runtime-api.ts`. Шлях SDK — це лише зовнішній контракт.
</Warning>

Публічні поверхні вбудованих плагінів, що завантажуються через фасад (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` та подібні публічні entry-файли), тепер віддають перевагу
активному знімку конфігурації runtime, якщо OpenClaw уже запущено. Якщо
знімок runtime ще не існує, вони повертаються до визначеного файлу конфігурації на диску.

Плагіни провайдерів також можуть експонувати вузький локальний barrel-контракт
плагіна, якщо допоміжна функція навмисно є специфічною для провайдера й поки що
не належить до узагальненого підшляху SDK. Поточний вбудований приклад:
провайдер Anthropic зберігає свої допоміжні функції потоків Claude у власному
публічному шарі `api.ts` / `contract-api.ts`, замість того щоб просувати логіку
Anthropic beta-header і `service_tier` в узагальнений контракт `plugin-sdk/*`.

Інші поточні вбудовані приклади:

- `@openclaw/openai-provider`: `api.ts` експортує побудовники провайдерів,
  допоміжні функції моделей за замовчуванням і побудовники realtime-провайдерів
- `@openclaw/openrouter-provider`: `api.ts` експортує побудовник провайдера та
  допоміжні функції онбордингу/конфігурації

<Warning>
  Production-код розширень також має уникати імпортів
  `openclaw/plugin-sdk/<other-plugin>`. Якщо допоміжна функція справді є
  спільною, перенесіть її до нейтрального підшляху SDK, такого як
  `openclaw/plugin-sdk/speech`, `.../provider-model-shared` або іншої
  поверхні, орієнтованої на можливості, замість зв’язування двох плагінів між собою.
</Warning>

## Пов’язане

- [Entry Points](/uk/plugins/sdk-entrypoints) — параметри `definePluginEntry` і `defineChannelPluginEntry`
- [Runtime Helpers](/uk/plugins/sdk-runtime) — повний довідник простору імен `api.runtime`
- [Setup and Config](/uk/plugins/sdk-setup) — пакування, маніфести, схеми конфігурації
- [Testing](/uk/plugins/sdk-testing) — тестові утиліти та правила lint
- [SDK Migration](/uk/plugins/sdk-migration) — міграція із застарілих поверхонь
- [Plugin Internals](/uk/plugins/architecture) — поглиблена архітектура та модель можливостей
