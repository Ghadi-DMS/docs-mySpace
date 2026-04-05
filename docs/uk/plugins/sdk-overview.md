---
read_when:
    - Вам потрібно знати, з якого підшляху SDK імпортувати
    - Вам потрібен довідник для всіх методів реєстрації в OpenClawPluginApi
    - Ви шукаєте конкретний експорт SDK
sidebarTitle: SDK Overview
summary: Карта імпортів, довідник API реєстрації та архітектура SDK
title: Огляд Plugin SDK
x-i18n:
    generated_at: "2026-04-05T21:37:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 16f570adb0f98c70862fe1adbe875d217a350f1945159e6f1de66081a0e5a09a
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Огляд Plugin SDK

Plugin SDK — це типізований контракт між plugins і core. Ця сторінка є
довідником для **того, що імпортувати** і **що можна реєструвати**.

<Tip>
  **Шукаєте практичний посібник?**
  - Перший plugin? Почніть із [Getting Started](/uk/plugins/building-plugins)
  - Channel plugin? Дивіться [Channel Plugins](/uk/plugins/sdk-channel-plugins)
  - Provider plugin? Дивіться [Provider Plugins](/uk/plugins/sdk-provider-plugins)
</Tip>

## Угода щодо імпорту

Завжди імпортуйте з конкретного підшляху:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Кожен підшлях — це невеликий самодостатній модуль. Це зберігає швидкий запуск і
запобігає проблемам із циклічними залежностями. Для специфічних для channel
допоміжних засобів entry/build віддавайте перевагу `openclaw/plugin-sdk/channel-core`; `openclaw/plugin-sdk/core` залишайте
для ширшої узагальненої поверхні та спільних допоміжних засобів, таких як
`buildChannelConfigSchema`.

Не додавайте й не використовуйте зручні seams з іменами provider, такі як
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, або
допоміжні seams із брендуванням channel. Bundled plugins мають компонувати узагальнені
підшляхи SDK у власних barrel-файлах `api.ts` або `runtime-api.ts`, а core
має або використовувати ці локальні для plugin barrel-файли, або додавати вузький узагальнений SDK
контракт, коли потреба справді є міжканальною.

Згенерована карта експортів усе ще містить невеликий набір допоміжних
seams для bundled plugins, таких як `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` і `plugin-sdk/matrix*`. Ці
підшляхи існують лише для супроводу bundled plugins і сумісності; їх
навмисно не включено до поширеної таблиці нижче, і вони не є рекомендованим
шляхом імпорту для нових сторонніх plugins.

## Довідник підшляхів

Найуживаніші підшляхи, згруповані за призначенням. Згенерований повний список із
понад 200 підшляхів міститься в `scripts/lib/plugin-sdk-entrypoints.json`.

Зарезервовані допоміжні підшляхи bundled plugins усе ще з’являються в цьому згенерованому списку.
Розглядайте їх як деталі реалізації / поверхні сумісності, якщо лише якась сторінка документації
явно не позначає один із них як публічний.

### Plugin entry

| Subpath                     | Key exports                                                                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Підшляхи channel">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Експорт кореневої схеми Zod `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Спільні допоміжні засоби майстра налаштування, запити allowlist, конструктори стану налаштування |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Допоміжні засоби для конфігурації/контролю дій для кількох облікових записів, допоміжні засоби резервного використання облікового запису за замовчуванням |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, допоміжні засоби нормалізації id облікового запису |
    | `plugin-sdk/account-resolution` | Допоміжні засоби пошуку облікового запису + резервного використання значення за замовчуванням |
    | `plugin-sdk/account-helpers` | Вузькі допоміжні засоби для списків облікових записів / дій з обліковими записами |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Типи схеми конфігурації channel |
    | `plugin-sdk/telegram-command-config` | Допоміжні засоби нормалізації/валідації користувацьких команд Telegram із резервною підтримкою bundled-контракту |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Спільні допоміжні засоби маршрутизації вхідних повідомлень + побудови envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Спільні допоміжні засоби запису та диспетчеризації вхідних повідомлень |
    | `plugin-sdk/messaging-targets` | Допоміжні засоби розбору/зіставлення цілей |
    | `plugin-sdk/outbound-media` | Спільні допоміжні засоби завантаження вихідних медіа |
    | `plugin-sdk/outbound-runtime` | Допоміжні засоби для ідентичності вихідних повідомлень / делегування надсилання |
    | `plugin-sdk/thread-bindings-runtime` | Допоміжні засоби життєвого циклу thread-binding і адаптерів |
    | `plugin-sdk/agent-media-payload` | Застарілий конструктор медіа payload агента |
    | `plugin-sdk/conversation-runtime` | Допоміжні засоби прив’язки conversation/thread, pairing і configured-binding |
    | `plugin-sdk/runtime-config-snapshot` | Допоміжний засіб знімка runtime-конфігурації |
    | `plugin-sdk/runtime-group-policy` | Допоміжні засоби визначення runtime-політики груп |
    | `plugin-sdk/channel-status` | Спільні допоміжні засоби знімка/підсумку стану channel |
    | `plugin-sdk/channel-config-primitives` | Вузькі примітиви schema конфігурації channel |
    | `plugin-sdk/channel-config-writes` | Допоміжні засоби авторизації запису конфігурації channel |
    | `plugin-sdk/channel-plugin-common` | Спільні prelude-експорти plugin channel |
    | `plugin-sdk/allowlist-config-edit` | Допоміжні засоби редагування/читання конфігурації allowlist |
    | `plugin-sdk/group-access` | Спільні допоміжні засоби прийняття рішень щодо доступу груп |
    | `plugin-sdk/direct-dm` | Спільні допоміжні засоби автентифікації/захисту direct-DM |
    | `plugin-sdk/interactive-runtime` | Допоміжні засоби нормалізації/скорочення payload інтерактивних відповідей |
    | `plugin-sdk/channel-inbound` | Допоміжні засоби debounce, зіставлення згадок і envelope |
    | `plugin-sdk/channel-send-result` | Типи результатів відповіді |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Допоміжні засоби розбору/зіставлення цілей |
    | `plugin-sdk/channel-contract` | Типи контракту channel |
    | `plugin-sdk/channel-feedback` | Підключення feedback/reaction |
  </Accordion>

  <Accordion title="Підшляхи provider">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Відібрані допоміжні засоби налаштування локальних/self-hosted provider |
    | `plugin-sdk/self-hosted-provider-setup` | Сфокусовані допоміжні засоби налаштування self-hosted provider, сумісних з OpenAI |
    | `plugin-sdk/provider-auth-runtime` | Допоміжні засоби визначення API-ключа під час виконання для provider plugins |
    | `plugin-sdk/provider-auth-api-key` | Допоміжні засоби онбордингу API-ключа / запису профілю |
    | `plugin-sdk/provider-auth-result` | Стандартний конструктор результату OAuth-автентифікації |
    | `plugin-sdk/provider-auth-login` | Спільні допоміжні засоби інтерактивного входу для provider plugins |
    | `plugin-sdk/provider-env-vars` | Допоміжні засоби пошуку env vars автентифікації provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні конструктори політик replay, допоміжні засоби endpoint provider і допоміжні засоби нормалізації id моделей, такі як `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Узагальнені допоміжні засоби HTTP/можливостей endpoint provider |
    | `plugin-sdk/provider-web-fetch` | Допоміжні засоби реєстрації/кешування provider web-fetch |
    | `plugin-sdk/provider-web-search` | Допоміжні засоби реєстрації/кешування/конфігурації provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схеми Gemini + діагностика, а також допоміжні засоби сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` і подібні |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи stream wrapper і спільні допоміжні засоби wrapper для Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Допоміжні засоби виправлення конфігурації онбордингу |
    | `plugin-sdk/global-singleton` | Допоміжні засоби локального для процесу singleton/map/cache |
  </Accordion>

  <Accordion title="Підшляхи автентифікації та безпеки">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, допоміжні засоби реєстру команд, допоміжні засоби авторизації відправника |
    | `plugin-sdk/approval-auth-runtime` | Допоміжні засоби визначення погоджувача та автентифікації дій у тому самому чаті |
    | `plugin-sdk/approval-client-runtime` | Допоміжні засоби профілю/фільтра native exec approval |
    | `plugin-sdk/approval-delivery-runtime` | Адаптери native approval capability/delivery |
    | `plugin-sdk/approval-native-runtime` | Допоміжні засоби цілі native approval + прив’язки облікового запису |
    | `plugin-sdk/approval-reply-runtime` | Допоміжні засоби payload відповіді для exec/plugin approval |
    | `plugin-sdk/command-auth-native` | Допоміжні засоби native command auth + native session-target |
    | `plugin-sdk/command-detection` | Спільні допоміжні засоби виявлення команд |
    | `plugin-sdk/command-surface` | Допоміжні засоби нормалізації тіла команди та поверхні команди |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | Спільні допоміжні засоби довіри, DM gating, external-content і збирання секретів |
    | `plugin-sdk/ssrf-policy` | Допоміжні засоби allowlist хостів і політики SSRF для приватної мережі |
    | `plugin-sdk/ssrf-runtime` | Допоміжні засоби pinned-dispatcher, fetch із захистом SSRF і політики SSRF |
    | `plugin-sdk/secret-input` | Допоміжні засоби розбору секретного вводу |
    | `plugin-sdk/webhook-ingress` | Допоміжні засоби запитів/цілей webhook |
    | `plugin-sdk/webhook-request-guards` | Допоміжні засоби обмеження розміру тіла запиту / таймауту |
  </Accordion>

  <Accordion title="Підшляхи runtime та сховища">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/runtime` | Широка поверхня допоміжних засобів runtime/логування/резервного копіювання/встановлення plugin |
    | `plugin-sdk/runtime-env` | Вузькі допоміжні засоби runtime env, logger, timeout, retry і backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Спільні допоміжні засоби команд/hook/http/interactive для plugin |
    | `plugin-sdk/hook-runtime` | Спільні допоміжні засоби пайплайна webhook/internal hook |
    | `plugin-sdk/lazy-runtime` | Допоміжні засоби lazy runtime import/binding, такі як `createLazyRuntimeModule`, `createLazyRuntimeMethod` і `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Допоміжні засоби виконання процесів |
    | `plugin-sdk/cli-runtime` | Допоміжні засоби форматування CLI, очікування та визначення версії |
    | `plugin-sdk/gateway-runtime` | Допоміжні засоби клієнта gateway і patch стану channel |
    | `plugin-sdk/config-runtime` | Допоміжні засоби завантаження/запису конфігурації |
    | `plugin-sdk/telegram-command-config` | Допоміжні засоби нормалізації назв/описів команд Telegram і перевірки дублікатів/конфліктів, навіть коли поверхня bundled контракту Telegram недоступна |
    | `plugin-sdk/approval-runtime` | Допоміжні засоби exec/plugin approval, конструктори approval capability, допоміжні засоби auth/profile, native routing/runtime |
    | `plugin-sdk/reply-runtime` | Спільні допоміжні засоби inbound/reply runtime, chunking, dispatch, heartbeat, планувальник відповідей |
    | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні засоби dispatch/finalize відповіді |
    | `plugin-sdk/reply-history` | Спільні допоміжні засоби коротковіконної історії відповідей, такі як `buildHistoryContext`, `recordPendingHistoryEntry` і `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Вузькі допоміжні засоби chunking тексту/markdown |
    | `plugin-sdk/session-store-runtime` | Допоміжні засоби шляху сховища сесій + updated-at |
    | `plugin-sdk/state-paths` | Допоміжні засоби шляхів до каталогів state/OAuth |
    | `plugin-sdk/routing` | Допоміжні засоби маршрутизації/ключа сесії/прив’язки облікового запису, такі як `resolveAgentRoute`, `buildAgentSessionKey` і `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Спільні допоміжні засоби підсумку стану channel/облікового запису, значення за замовчуванням стану runtime і допоміжні засоби метаданих проблем |
    | `plugin-sdk/target-resolver-runtime` | Спільні допоміжні засоби визначення цілей |
    | `plugin-sdk/string-normalization-runtime` | Допоміжні засоби нормалізації slug/рядків |
    | `plugin-sdk/request-url` | Витягування рядкових URL із fetch/request-подібних вхідних даних |
    | `plugin-sdk/run-command` | Виконавець команд із таймером і нормалізованими результатами stdout/stderr |
    | `plugin-sdk/param-readers` | Поширені читачі параметрів tool/CLI |
    | `plugin-sdk/tool-send` | Витягування канонічних полів цілі надсилання з аргументів tool |
    | `plugin-sdk/temp-path` | Спільні допоміжні засоби шляхів для тимчасових завантажень |
    | `plugin-sdk/logging-core` | Допоміжні засоби subsystem logger і редагування конфіденційних даних |
    | `plugin-sdk/markdown-table-runtime` | Допоміжні засоби режиму таблиць Markdown |
    | `plugin-sdk/json-store` | Невеликі допоміжні засоби читання/запису стану JSON |
    | `plugin-sdk/file-lock` | Допоміжні засоби повторно вхідного file-lock |
    | `plugin-sdk/persistent-dedupe` | Допоміжні засоби дискового кешу dedupe |
    | `plugin-sdk/acp-runtime` | Допоміжні засоби ACP runtime/session і dispatch відповіді |
    | `plugin-sdk/agent-config-primitives` | Вузькі примітиви schema конфігурації runtime агента |
    | `plugin-sdk/boolean-param` | Читач нестрогих булевих параметрів |
    | `plugin-sdk/dangerous-name-runtime` | Допоміжні засоби визначення небезпечних імен |
    | `plugin-sdk/device-bootstrap` | Допоміжні засоби початкової ініціалізації пристрою та токенів pairing |
    | `plugin-sdk/extension-shared` | Спільні примітиви пасивного channel і допоміжних засобів стану |
    | `plugin-sdk/models-provider-runtime` | Допоміжні засоби відповіді `/models` command/provider |
    | `plugin-sdk/skill-commands-runtime` | Допоміжні засоби переліку команд Skills |
    | `plugin-sdk/native-command-registry` | Допоміжні засоби реєстру/build/serialize native command |
    | `plugin-sdk/provider-zai-endpoint` | Допоміжні засоби виявлення endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Допоміжні засоби системних подій/heartbeat |
    | `plugin-sdk/collection-runtime` | Невеликі допоміжні засоби обмеженого кешу |
    | `plugin-sdk/diagnostic-runtime` | Допоміжні засоби діагностичних прапорців і подій |
    | `plugin-sdk/error-runtime` | Граф помилок, форматування, спільні допоміжні засоби класифікації помилок, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Допоміжні засоби обгорнутого fetch, proxy і pinned lookup |
    | `plugin-sdk/host-runtime` | Допоміжні засоби нормалізації hostname і SCP host |
    | `plugin-sdk/retry-runtime` | Допоміжні засоби конфігурації retry і виконання retry |
    | `plugin-sdk/agent-runtime` | Допоміжні засоби каталогу/ідентичності/workspace агента |
    | `plugin-sdk/directory-runtime` | Запит каталогу на основі конфігурації / dedup |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Підшляхи можливостей і тестування">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Спільні допоміжні засоби fetch/transform/store медіа, а також конструктори media payload |
    | `plugin-sdk/media-understanding` | Типи provider для media understanding, а також допоміжні експорти для роботи з image/audio на стороні provider |
    | `plugin-sdk/text-runtime` | Спільні допоміжні засоби тексту/markdown/логування, такі як видалення видимого для асистента тексту, render/chunking/table helpers для markdown, допоміжні засоби редагування конфіденційних даних, directive-tag helpers і утиліти безпечного тексту |
    | `plugin-sdk/text-chunking` | Допоміжний засіб chunking вихідного тексту |
    | `plugin-sdk/speech` | Типи speech provider, а також допоміжні засоби директив, реєстру й валідації на стороні provider |
    | `plugin-sdk/speech-core` | Спільні типи speech provider, а також допоміжні засоби реєстру, директив і нормалізації |
    | `plugin-sdk/realtime-transcription` | Типи provider для realtime transcription і допоміжні засоби реєстру |
    | `plugin-sdk/realtime-voice` | Типи provider для realtime voice і допоміжні засоби реєстру |
    | `plugin-sdk/image-generation` | Типи provider для генерації зображень |
    | `plugin-sdk/image-generation-core` | Спільні типи image-generation, failover, auth і допоміжні засоби реєстру |
    | `plugin-sdk/video-generation` | Типи provider/request/result для генерації відео |
    | `plugin-sdk/video-generation-core` | Спільні типи video-generation, допоміжні засоби failover, lookup provider і розбору model-ref |
    | `plugin-sdk/webhook-targets` | Допоміжні засоби реєстру цілей webhook і встановлення маршрутів |
    | `plugin-sdk/webhook-path` | Допоміжні засоби нормалізації шляху webhook |
    | `plugin-sdk/web-media` | Спільні допоміжні засоби завантаження віддалених/локальних медіа |
    | `plugin-sdk/zod` | Повторно експортований `zod` для користувачів Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Підшляхи memory">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/memory-core` | Поверхня допоміжних засобів bundled memory-core для manager/config/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Runtime facade для індексації/пошуку memory |
    | `plugin-sdk/memory-core-host-engine-foundation` | Експорти foundation engine хоста memory |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Експорти embedding engine хоста memory |
    | `plugin-sdk/memory-core-host-engine-qmd` | Експорти QMD engine хоста memory |
    | `plugin-sdk/memory-core-host-engine-storage` | Експорти storage engine хоста memory |
    | `plugin-sdk/memory-core-host-multimodal` | Допоміжні засоби multimodal хоста memory |
    | `plugin-sdk/memory-core-host-query` | Допоміжні засоби query хоста memory |
    | `plugin-sdk/memory-core-host-secret` | Допоміжні засоби secret хоста memory |
    | `plugin-sdk/memory-core-host-events` | Допоміжні засоби журналу подій хоста memory |
    | `plugin-sdk/memory-core-host-status` | Допоміжні засоби стану хоста memory |
    | `plugin-sdk/memory-core-host-runtime-cli` | Допоміжні засоби runtime CLI хоста memory |
    | `plugin-sdk/memory-core-host-runtime-core` | Допоміжні засоби core runtime хоста memory |
    | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні засоби file/runtime хоста memory |
    | `plugin-sdk/memory-host-core` | Нейтральний до вендора псевдонім для допоміжних засобів core runtime хоста memory |
    | `plugin-sdk/memory-host-events` | Нейтральний до вендора псевдонім для допоміжних засобів журналу подій хоста memory |
    | `plugin-sdk/memory-host-files` | Нейтральний до вендора псевдонім для допоміжних засобів file/runtime хоста memory |
    | `plugin-sdk/memory-host-markdown` | Спільні допоміжні засоби керованого markdown для plugins, суміжних із memory |
    | `plugin-sdk/memory-host-search` | Активна runtime facade memory для доступу до search-manager |
    | `plugin-sdk/memory-host-status` | Нейтральний до вендора псевдонім для допоміжних засобів стану хоста memory |
    | `plugin-sdk/memory-lancedb` | Поверхня допоміжних засобів bundled memory-lancedb |
  </Accordion>

  <Accordion title="Зарезервовані підшляхи bundled-helper">
    | Family | Current subpaths | Intended use |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-support` | Допоміжні засоби підтримки bundled browser plugin |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Поверхня допоміжних засобів/runtime для bundled Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Поверхня допоміжних засобів/runtime для bundled LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Поверхня допоміжних засобів для bundled IRC |
    | Channel-specific helpers | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Bundled seams сумісності/допоміжних засобів для channel |
    | Auth/plugin-specific helpers | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Bundled seams допоміжних засобів feature/plugin; `plugin-sdk/github-copilot-token` наразі експортує `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API реєстрації

Колбек `register(api)` отримує об’єкт `OpenClawPluginApi` з такими
методами:

### Реєстрація можливостей

| Method                                           | What it registers                |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Текстовий inference (LLM)        |
| `api.registerChannel(...)`                       | Канал обміну повідомленнями      |
| `api.registerSpeechProvider(...)`                | Синтез text-to-speech / STT      |
| `api.registerRealtimeTranscriptionProvider(...)` | Потокова realtime transcription  |
| `api.registerRealtimeVoiceProvider(...)`         | Дуплексні сесії realtime voice   |
| `api.registerMediaUnderstandingProvider(...)`    | Аналіз зображень/аудіо/відео     |
| `api.registerImageGenerationProvider(...)`       | Генерація зображень              |
| `api.registerVideoGenerationProvider(...)`       | Генерація відео                  |
| `api.registerWebFetchProvider(...)`              | Provider для web fetch / scrape  |
| `api.registerWebSearchProvider(...)`             | Вебпошук                         |

### Tools і команди

| Method                          | What it registers                             |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Tool агента (обов’язковий або `{ optional: true }`) |
| `api.registerCommand(def)`      | Користувацька команда (обходить LLM)          |

### Інфраструктура

| Method                                         | What it registers                       |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook подій                              |
| `api.registerHttpRoute(params)`                | HTTP endpoint gateway                   |
| `api.registerGatewayMethod(name, handler)`     | RPC-метод gateway                       |
| `api.registerCli(registrar, opts?)`            | Підкоманда CLI                          |
| `api.registerService(service)`                 | Фонова служба                           |
| `api.registerInteractiveHandler(registration)` | Інтерактивний обробник                  |
| `api.registerMemoryPromptSupplement(builder)`  | Додатковий суміжний із memory розділ prompt |
| `api.registerMemoryCorpusSupplement(adapter)`  | Додатковий корпус для пошуку/читання memory |

Зарезервовані простори назв адміністрування core (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) завжди залишаються `operator.admin`, навіть якщо plugin намагається призначити
вужчу область видимості методу gateway. Для
методів, що належать plugin, віддавайте перевагу префіксам, специфічним для plugin.

### Метадані реєстрації CLI

`api.registerCli(registrar, opts?)` приймає два види метаданих верхнього рівня:

- `commands`: явні корені команд, якими володіє реєстратор
- `descriptors`: дескриптори команд на етапі розбору, що використовуються для довідки кореневого CLI,
  маршрутизації та лінивої реєстрації CLI plugin

Якщо ви хочете, щоб команда plugin залишалася ліниво завантажуваною у звичайному шляху кореневого CLI,
надайте `descriptors`, які охоплюють кожен корінь команди верхнього рівня, що надається цим
реєстратором.

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

Використовуйте `commands` окремо лише тоді, коли вам не потрібна лінива реєстрація кореневого CLI.
Цей шлях eager сумісності залишається підтримуваним, але він не встановлює
плейсхолдери на основі descriptor для лінивого завантаження на етапі розбору.

### Ексклюзивні слоти

| Method                                     | What it registers                     |
| ------------------------------------------ | ------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Context engine (одночасно активний лише один) |
| `api.registerMemoryPromptSection(builder)` | Конструктор розділу prompt для memory |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver плану flush для memory       |
| `api.registerMemoryRuntime(runtime)`       | Адаптер runtime для memory            |

### Адаптери вбудовування memory

| Method                                         | What it registers                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Адаптер вбудовування memory для активного plugin |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` і
  `registerMemoryRuntime` є ексклюзивними для memory plugins.
- `registerMemoryEmbeddingProvider` дозволяє активному memory plugin реєструвати один
  або більше id адаптерів вбудовування (наприклад `openai`, `gemini` або користувацький
  id, визначений plugin).
- Конфігурація користувача, така як `agents.defaults.memorySearch.provider` і
  `agents.defaults.memorySearch.fallback`, визначається відносно цих зареєстрованих
  id адаптерів.

### Події та життєвий цикл

| Method                                       | What it does                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Типізований lifecycle hook    |
| `api.onConversationBindingResolved(handler)` | Колбек прив’язки conversation |

### Семантика рішень hook

- `before_tool_call`: повернення `{ block: true }` є термінальним. Щойно будь-який обробник встановлює це значення, обробники з нижчим пріоритетом пропускаються.
- `before_tool_call`: повернення `{ block: false }` вважається відсутністю рішення (так само, як і пропуск `block`), а не перевизначенням.
- `before_install`: повернення `{ block: true }` є термінальним. Щойно будь-який обробник встановлює це значення, обробники з нижчим пріоритетом пропускаються.
- `before_install`: повернення `{ block: false }` вважається відсутністю рішення (так само, як і пропуск `block`), а не перевизначенням.
- `reply_dispatch`: повернення `{ handled: true, ... }` є термінальним. Щойно будь-який обробник заявляє про dispatch, обробники з нижчим пріоритетом і стандартний шлях dispatch моделі пропускаються.
- `message_sending`: повернення `{ cancel: true }` є термінальним. Щойно будь-який обробник встановлює це значення, обробники з нижчим пріоритетом пропускаються.
- `message_sending`: повернення `{ cancel: false }` вважається відсутністю рішення (так само, як і пропуск `cancel`), а не перевизначенням.

### Поля об’єкта API

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id plugin                                                                                   |
| `api.name`               | `string`                  | Відображувана назва                                                                         |
| `api.version`            | `string?`                 | Версія plugin (необов’язково)                                                               |
| `api.description`        | `string?`                 | Опис plugin (необов’язково)                                                                 |
| `api.source`             | `string`                  | Шлях до джерела plugin                                                                      |
| `api.rootDir`            | `string?`                 | Кореневий каталог plugin (необов’язково)                                                    |
| `api.config`             | `OpenClawConfig`          | Поточний знімок конфігурації (активний знімок runtime в пам’яті, коли доступний)           |
| `api.pluginConfig`       | `Record<string, unknown>` | Конфігурація, специфічна для plugin, із `plugins.entries.<id>.config`                       |
| `api.runtime`            | `PluginRuntime`           | [Допоміжні засоби runtime](/uk/plugins/sdk-runtime)                                            |
| `api.logger`             | `PluginLogger`            | Logger з областю видимості (`debug`, `info`, `warn`, `error`)                               |
| `api.registrationMode`   | `PluginRegistrationMode`  | Поточний режим завантаження; `"setup-runtime"` — це полегшене вікно запуску/налаштування до повного entry |
| `api.resolvePath(input)` | `(string) => string`      | Визначити шлях відносно кореня plugin                                                       |

## Угода щодо внутрішніх модулів

У межах вашого plugin використовуйте локальні barrel-файли для внутрішніх імпортів:

```
my-plugin/
  api.ts            # Публічні експорти для зовнішніх споживачів
  runtime-api.ts    # Експорти runtime лише для внутрішнього використання
  index.ts          # Точка входу plugin
  setup-entry.ts    # Полегшена точка входу лише для налаштування (необов’язково)
```

<Warning>
  Ніколи не імпортуйте власний plugin через `openclaw/plugin-sdk/<your-plugin>`
  у production-коді. Для внутрішніх імпортів використовуйте `./api.ts` або
  `./runtime-api.ts`. Шлях SDK — це лише зовнішній контракт.
</Warning>

Публічні поверхні bundled plugins, що завантажуються через facade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` та подібні публічні entry-файли), тепер віддають перевагу
активному знімку конфігурації runtime, коли OpenClaw уже запущено. Якщо знімок runtime
ще не існує, вони повертаються до визначеного конфігураційного файлу на диску.

Provider plugins також можуть надавати вузький локальний для plugin barrel контракту, коли
допоміжний засіб навмисно є специфічним для provider і ще не належить до узагальненого SDK
підшляху. Поточний bundled-приклад: provider Anthropic зберігає свої допоміжні засоби
Claude stream у власному публічному seam `api.ts` / `contract-api.ts` замість
перенесення логіки Anthropic beta-header і `service_tier` до узагальненого
контракту `plugin-sdk/*`.

Інші поточні bundled-приклади:

- `@openclaw/openai-provider`: `api.ts` експортує конструктори provider,
  допоміжні засоби моделей за замовчуванням і конструктори realtime provider
- `@openclaw/openrouter-provider`: `api.ts` експортує конструктор provider, а також
  допоміжні засоби онбордингу/конфігурації

<Warning>
  Production-код extension також має уникати імпортів `openclaw/plugin-sdk/<other-plugin>`.
  Якщо допоміжний засіб справді є спільним, перенесіть його до нейтрального підшляху SDK,
  такого як `openclaw/plugin-sdk/speech`, `.../provider-model-shared` або іншої
  поверхні, орієнтованої на можливості, замість того щоб зв’язувати два plugins між собою.
</Warning>

## Пов’язане

- [Entry Points](/uk/plugins/sdk-entrypoints) — параметри `definePluginEntry` і `defineChannelPluginEntry`
- [Runtime Helpers](/uk/plugins/sdk-runtime) — повний довідник простору назв `api.runtime`
- [Setup and Config](/uk/plugins/sdk-setup) — пакування, маніфести, схеми конфігурації
- [Testing](/uk/plugins/sdk-testing) — утиліти тестування та правила lint
- [SDK Migration](/uk/plugins/sdk-migration) — міграція із застарілих поверхонь
- [Plugin Internals](/uk/plugins/architecture) — поглиблена архітектура та модель можливостей
