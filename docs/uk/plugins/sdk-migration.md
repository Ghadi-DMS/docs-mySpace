---
read_when:
    - Ви бачите попередження OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ви бачите попередження OPENCLAW_EXTENSION_API_DEPRECATED
    - Ви оновлюєте плагін до сучасної архітектури плагінів
    - Ви підтримуєте зовнішній плагін OpenClaw
sidebarTitle: Migrate to SDK
summary: Перейдіть із застарілого шару зворотної сумісності на сучасний plugin SDK
title: Міграція Plugin SDK
x-i18n:
    generated_at: "2026-04-05T21:36:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: f4a5aba3ff99f9cc75518a914e7a9faf92b9021f26031f7ace74d0b58a3416da
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Міграція Plugin SDK

OpenClaw перейшов від широкого шару зворотної сумісності до сучасної
архітектури плагінів із цільовими документованими імпортами. Якщо ваш плагін
було створено до появи нової архітектури, цей посібник допоможе вам виконати
міграцію.

## Що змінюється

Стара система плагінів надавала дві дуже широкі поверхні, які дозволяли
плагінам імпортувати будь-що потрібне з однієї точки входу:

- **`openclaw/plugin-sdk/compat`** — єдиний імпорт, який повторно експортував
  десятки допоміжних функцій. Його було запроваджено, щоб старіші плагіни на
  основі хуків продовжували працювати, поки створювалася нова архітектура
  плагінів.
- **`openclaw/extension-api`** — міст, який надавав плагінам прямий доступ до
  допоміжних функцій на стороні хоста, таких як вбудований запускник агента.

Обидві поверхні тепер **застарілі**. Вони досі працюють під час виконання, але
нові плагіни не повинні їх використовувати, а наявні плагіни слід перенести до
того, як у наступному мажорному випуску їх буде видалено.

<Warning>
  Шар зворотної сумісності буде видалено в одному з майбутніх мажорних
  випусків. Плагіни, які досі імпортують із цих поверхонь, перестануть
  працювати, коли це станеться.
</Warning>

## Чому це змінилося

Старий підхід спричиняв проблеми:

- **Повільний запуск** — імпорт однієї допоміжної функції завантажував десятки
  не пов’язаних модулів
- **Циклічні залежності** — широкі повторні експорти спрощували створення
  циклів імпорту
- **Неочевидна поверхня API** — не було способу визначити, які експорти є
  стабільними, а які внутрішніми

Сучасний plugin SDK виправляє це: кожен шлях імпорту (`openclaw/plugin-sdk/\<subpath\>`)
є невеликим автономним модулем із чітким призначенням і документованим
контрактом.

Застарілі зручні seam-інтерфейси провайдерів для вбудованих каналів також
прибрано. Імпорти на кшталт `openclaw/plugin-sdk/slack`,
`openclaw/plugin-sdk/discord`, `openclaw/plugin-sdk/signal`,
`openclaw/plugin-sdk/whatsapp`, брендовані допоміжні seam-інтерфейси каналів і
`openclaw/plugin-sdk/telegram-core` були приватними скороченнями
mono-repo, а не стабільними контрактами плагінів. Натомість використовуйте
вузькі універсальні підшляхи SDK. Усередині робочого простору вбудованих
плагінів зберігайте допоміжні функції, що належать провайдеру, у власних
`api.ts` або `runtime-api.ts` цього плагіна.

Поточні приклади вбудованих провайдерів:

- Anthropic зберігає допоміжні функції потоку, специфічні для Claude, у
  власному seam-інтерфейсі `api.ts` / `contract-api.ts`
- OpenAI зберігає побудовники провайдерів, допоміжні функції моделей за
  замовчуванням і побудовники realtime-провайдерів у власному `api.ts`
- OpenRouter зберігає побудовник провайдера та допоміжні функції онбордингу /
  конфігурації у власному `api.ts`

## Як виконати міграцію

<Steps>
  <Step title="Перевірте резервну поведінку обгортки Windows">
    Якщо ваш плагін використовує `openclaw/plugin-sdk/windows-spawn`, нерозв’язані
    обгортки Windows `.cmd`/`.bat` тепер завершуються без резервного шляху, якщо
    ви явно не передасте `allowShellFallback: true`.

    ```typescript
    // До
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Після
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Установлюйте це лише для довірених сумісних викликів, які навмисно
      // допускають резервний шлях через оболонку.
      allowShellFallback: true,
    });
    ```

    Якщо ваш виклик не покладається навмисно на резервний шлях через оболонку,
    не встановлюйте `allowShellFallback` і натомість обробляйте викинуту
    помилку.

  </Step>

  <Step title="Знайдіть застарілі імпорти">
    Знайдіть у своєму плагіні імпорти з будь-якої із застарілих поверхонь:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Замініть їх цільовими імпортами">
    Кожен експорт зі старої поверхні відповідає конкретному сучасному шляху
    імпорту:

    ```typescript
    // До (застарілий шар зворотної сумісності)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Після (сучасні цільові імпорти)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Для допоміжних функцій на стороні хоста використовуйте впроваджене
    середовище виконання плагіна замість прямого імпорту:

    ```typescript
    // До (застарілий міст extension-api)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Після (впроваджене середовище виконання)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Такий самий шаблон застосовується й до інших застарілих допоміжних функцій
    моста:

    | Старий імпорт | Сучасний еквівалент |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | допоміжні функції сховища сесій | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Зберіть і протестуйте">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Довідник шляхів імпорту

<Accordion title="Таблиця поширених шляхів імпорту">
  | Шлях імпорту | Призначення | Ключові експорти |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Канонічна допоміжна функція точки входу плагіна | `definePluginEntry` |
  | `plugin-sdk/core` | Застарілий узагальнений повторний експорт для визначень / побудовників точок входу каналів | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Експорт кореневої схеми конфігурації | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Допоміжна функція точки входу для одного провайдера | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Цільові визначення та побудовники точок входу каналів | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Спільні допоміжні функції майстра налаштування | Запити allowlist, побудовники статусу налаштування |
  | `plugin-sdk/setup-runtime` | Допоміжні функції середовища виконання під час налаштування | Безпечні для імпорту адаптери патчів налаштування, допоміжні функції для приміток пошуку, `promptResolvedAllowFrom`, `splitSetupEntries`, делеговані проксі налаштування |
  | `plugin-sdk/setup-adapter-runtime` | Допоміжні функції адаптера налаштування | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Допоміжні інструменти налаштування | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Допоміжні функції для кількох облікових записів | Допоміжні функції списку облікових записів / конфігурації / шлюзу дій |
  | `plugin-sdk/account-id` | Допоміжні функції ID облікового запису | `DEFAULT_ACCOUNT_ID`, нормалізація ID облікового запису |
  | `plugin-sdk/account-resolution` | Допоміжні функції пошуку облікових записів | Допоміжні функції пошуку облікового запису + резервного використання значення за замовчуванням |
  | `plugin-sdk/account-helpers` | Вузькі допоміжні функції облікових записів | Допоміжні функції списку облікових записів / дій з обліковими записами |
  | `plugin-sdk/channel-setup` | Адаптери майстра налаштування | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Примітиви парування DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Префікс відповіді + підключення індикатора друку | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Фабрики адаптерів конфігурації | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Побудовники схем конфігурації | Типи схем конфігурації каналу |
  | `plugin-sdk/telegram-command-config` | Допоміжні функції конфігурації команд Telegram | Нормалізація назв команд, обрізання описів, валідація дублікатів / конфліктів |
  | `plugin-sdk/channel-policy` | Визначення політики груп / DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Відстеження статусу облікового запису | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Допоміжні функції вхідного конверта | Спільні допоміжні функції маршруту + побудовника конверта |
  | `plugin-sdk/inbound-reply-dispatch` | Допоміжні функції вхідних відповідей | Спільні допоміжні функції запису та диспетчеризації |
  | `plugin-sdk/messaging-targets` | Розбір цілей повідомлень | Допоміжні функції розбору / зіставлення цілей |
  | `plugin-sdk/outbound-media` | Допоміжні функції вихідних медіа | Спільне завантаження вихідних медіа |
  | `plugin-sdk/outbound-runtime` | Допоміжні функції середовища виконання вихідних дій | Допоміжні функції ідентичності / делегування надсилання вихідних дій |
  | `plugin-sdk/thread-bindings-runtime` | Допоміжні функції прив’язок потоків | Життєвий цикл прив’язок потоків і допоміжні функції адаптера |
  | `plugin-sdk/agent-media-payload` | Застарілі допоміжні функції медіапейлоаду | Побудовник медіапейлоаду агента для застарілих схем полів |
  | `plugin-sdk/channel-runtime` | Застарілий shim сумісності | Лише застарілі утиліти середовища виконання каналу |
  | `plugin-sdk/channel-send-result` | Типи результатів надсилання | Типи результатів відповіді |
  | `plugin-sdk/runtime-store` | Постійне сховище плагіна | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Широкі допоміжні функції середовища виконання | Допоміжні функції runtime / logging / backup / встановлення плагінів |
  | `plugin-sdk/runtime-env` | Вузькі допоміжні функції середовища runtime | Logger / runtime env, timeout, retry і backoff helpers |
  | `plugin-sdk/plugin-runtime` | Спільні допоміжні функції runtime плагінів | Допоміжні функції команд / hooks / http / interactive плагінів |
  | `plugin-sdk/hook-runtime` | Допоміжні функції конвеєра hooks | Спільні допоміжні функції конвеєра webhook / internal hook |
  | `plugin-sdk/lazy-runtime` | Допоміжні функції лінивого runtime | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Допоміжні функції процесів | Спільні допоміжні функції exec |
  | `plugin-sdk/cli-runtime` | Допоміжні функції CLI runtime | Форматування команд, очікування, допоміжні функції версій |
  | `plugin-sdk/gateway-runtime` | Допоміжні функції gateway | Клієнт gateway і допоміжні функції патчів статусу каналу |
  | `plugin-sdk/config-runtime` | Допоміжні функції конфігурації | Допоміжні функції завантаження / запису конфігурації |
  | `plugin-sdk/telegram-command-config` | Допоміжні функції команд Telegram | Допоміжні функції валідації команд Telegram зі стабільним резервним шляхом, коли поверхня контракту вбудованого Telegram недоступна |
  | `plugin-sdk/approval-runtime` | Допоміжні функції запитів на схвалення | Exec / plugin approval payload, approval capability / profile helpers, native approval routing / runtime helpers |
  | `plugin-sdk/approval-auth-runtime` | Допоміжні функції авторизації схвалення | Визначення затверджувача, авторизація дій у тому самому чаті |
  | `plugin-sdk/approval-client-runtime` | Допоміжні функції клієнта схвалення | Допоміжні функції профілю / фільтра native exec approval |
  | `plugin-sdk/approval-delivery-runtime` | Допоміжні функції доставки схвалення | Адаптери native approval capability / delivery |
  | `plugin-sdk/approval-native-runtime` | Допоміжні функції цілей схвалення | Допоміжні функції native approval target / account binding |
  | `plugin-sdk/approval-reply-runtime` | Допоміжні функції відповіді на схвалення | Допоміжні функції exec / plugin approval reply payload |
  | `plugin-sdk/security-runtime` | Допоміжні функції безпеки | Спільні допоміжні функції trust, DM gating, external-content і збирання секретів |
  | `plugin-sdk/ssrf-policy` | Допоміжні функції політики SSRF | Допоміжні функції allowlist хостів і політики приватної мережі |
  | `plugin-sdk/ssrf-runtime` | Допоміжні функції SSRF runtime | Допоміжні функції pinned-dispatcher, guarded fetch, SSRF policy |
  | `plugin-sdk/collection-runtime` | Допоміжні функції обмеженого кешу | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Допоміжні функції діагностичного шлюзу | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Допоміжні функції форматування помилок | `formatUncaughtError`, `isApprovalNotFoundError`, допоміжні функції графа помилок |
  | `plugin-sdk/fetch-runtime` | Допоміжні функції обгорнутого fetch / proxy | `resolveFetch`, допоміжні функції proxy |
  | `plugin-sdk/host-runtime` | Допоміжні функції нормалізації хоста | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Допоміжні функції retry | `RetryConfig`, `retryAsync`, виконавці політик |
  | `plugin-sdk/allow-from` | Форматування allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Мапінг вхідних даних allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Шлюз команд і допоміжні функції поверхні команд | `resolveControlCommandGate`, допоміжні функції авторизації відправника, допоміжні функції реєстру команд |
  | `plugin-sdk/secret-input` | Розбір введення секретів | Допоміжні функції введення секретів |
  | `plugin-sdk/webhook-ingress` | Допоміжні функції запитів webhook | Утиліти цілей webhook |
  | `plugin-sdk/webhook-request-guards` | Допоміжні функції захисту тіла запиту webhook | Допоміжні функції читання / обмеження тіла запиту |
  | `plugin-sdk/reply-runtime` | Спільний runtime відповідей | Вхідна диспетчеризація, heartbeat, планувальник відповідей, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні функції диспетчеризації відповідей | Допоміжні функції finalize + provider dispatch |
  | `plugin-sdk/reply-history` | Допоміжні функції історії відповідей | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Планування посилань відповіді | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Допоміжні функції chunk відповідей | Допоміжні функції chunking тексту / markdown |
  | `plugin-sdk/session-store-runtime` | Допоміжні функції сховища сесій | Допоміжні функції шляху сховища та `updated-at` |
  | `plugin-sdk/state-paths` | Допоміжні функції шляхів стану | Допоміжні функції каталогів state та OAuth |
  | `plugin-sdk/routing` | Допоміжні функції маршрутизації / ключів сесії | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, допоміжні функції нормалізації ключів сесії |
  | `plugin-sdk/status-helpers` | Допоміжні функції статусу каналу | Побудовники підсумку статусу каналу / облікового запису, значення runtime-state за замовчуванням, допоміжні функції метаданих проблем |
  | `plugin-sdk/target-resolver-runtime` | Допоміжні функції визначення цілі | Спільні допоміжні функції визначення цілі |
  | `plugin-sdk/string-normalization-runtime` | Допоміжні функції нормалізації рядків | Допоміжні функції нормалізації slug / string |
  | `plugin-sdk/request-url` | Допоміжні функції URL запиту | Витягування URL-рядків із request-подібних входів |
  | `plugin-sdk/run-command` | Допоміжні функції команд із таймером | Виконавець команд із нормалізованими stdout / stderr |
  | `plugin-sdk/param-readers` | Зчитувачі параметрів | Загальні зчитувачі параметрів інструментів / CLI |
  | `plugin-sdk/tool-send` | Витягування tool send | Витягування канонічних полів цілі надсилання з аргументів інструмента |
  | `plugin-sdk/temp-path` | Допоміжні функції тимчасових шляхів | Спільні допоміжні функції тимчасових шляхів для завантажень |
  | `plugin-sdk/logging-core` | Допоміжні функції логування | Логер підсистеми та допоміжні функції редагування чутливих даних |
  | `plugin-sdk/markdown-table-runtime` | Допоміжні функції таблиць Markdown | Допоміжні режими таблиць Markdown |
  | `plugin-sdk/reply-payload` | Типи payload відповідей повідомлень | Типи payload відповідей |
  | `plugin-sdk/provider-setup` | Добірні допоміжні функції налаштування локальних / self-hosted провайдерів | Допоміжні функції виявлення / конфігурації self-hosted провайдерів |
  | `plugin-sdk/self-hosted-provider-setup` | Цільові допоміжні функції налаштування OpenAI-compatible self-hosted провайдерів | Ті самі допоміжні функції виявлення / конфігурації self-hosted провайдерів |
  | `plugin-sdk/provider-auth-runtime` | Допоміжні функції runtime авторизації провайдера | Допоміжні функції визначення API-ключів у runtime |
  | `plugin-sdk/provider-auth-api-key` | Допоміжні функції налаштування API-ключів провайдера | Допоміжні функції онбордингу / запису профілю для API-ключів |
  | `plugin-sdk/provider-auth-result` | Допоміжні функції результату авторизації провайдера | Стандартний побудовник OAuth auth-result |
  | `plugin-sdk/provider-auth-login` | Допоміжні функції інтерактивного входу провайдера | Спільні допоміжні функції інтерактивного входу |
  | `plugin-sdk/provider-env-vars` | Допоміжні функції env vars провайдера | Допоміжні функції пошуку env vars для авторизації провайдера |
  | `plugin-sdk/provider-model-shared` | Спільні допоміжні функції моделей / replay провайдера | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні побудовники політик replay, допоміжні функції endpoint провайдера та нормалізації ID моделей |
  | `plugin-sdk/provider-catalog-shared` | Спільні допоміжні функції каталогу провайдера | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Патчі онбордингу провайдера | Допоміжні функції конфігурації онбордингу |
  | `plugin-sdk/provider-http` | Допоміжні функції HTTP провайдера | Загальні допоміжні функції HTTP / можливостей endpoint провайдера |
  | `plugin-sdk/provider-web-fetch` | Допоміжні функції web-fetch провайдера | Допоміжні функції реєстрації / кешування провайдера web-fetch |
  | `plugin-sdk/provider-web-search` | Допоміжні функції web-search провайдера | Допоміжні функції реєстрації / кешування / конфігурації провайдера web-search |
  | `plugin-sdk/provider-tools` | Допоміжні функції сумісності інструментів / схем провайдера | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схем Gemini + діагностика, а також допоміжні функції сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Допоміжні функції використання провайдера | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` та інші допоміжні функції використання провайдера |
  | `plugin-sdk/provider-stream` | Допоміжні функції обгортки потоку провайдера | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи обгорток потоку та спільні допоміжні функції обгорток Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Впорядкована асинхронна черга | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Спільні допоміжні функції медіа | Допоміжні функції отримання / перетворення / збереження медіа, а також побудовники медіапейлоадів |
  | `plugin-sdk/media-understanding` | Допоміжні функції media-understanding | Типи провайдерів media understanding, а також експорти допоміжних функцій зображень / аудіо для провайдерів |
  | `plugin-sdk/text-runtime` | Спільні допоміжні функції тексту | Прибирання видимого для асистента тексту, допоміжні функції render / chunking / table для markdown, редагування чутливих даних, допоміжні функції тегів директив, безпечного тексту та інші пов’язані текстові / logging helpers |
  | `plugin-sdk/text-chunking` | Допоміжні функції chunking тексту | Допоміжна функція chunking вихідного тексту |
  | `plugin-sdk/speech` | Допоміжні функції speech | Типи провайдерів speech, а також експорти директив, реєстру та валідації для провайдерів |
  | `plugin-sdk/speech-core` | Спільне ядро speech | Типи провайдерів speech, реєстр, директиви, нормалізація |
  | `plugin-sdk/realtime-transcription` | Допоміжні функції realtime transcription | Типи провайдерів і допоміжні функції реєстру |
  | `plugin-sdk/realtime-voice` | Допоміжні функції realtime voice | Типи провайдерів і допоміжні функції реєстру |
  | `plugin-sdk/image-generation-core` | Спільне ядро image generation | Допоміжні функції типів, failover, auth і реєстру image generation |
  | `plugin-sdk/video-generation` | Допоміжні функції video generation | Типи провайдерів / запитів / результатів video generation |
  | `plugin-sdk/video-generation-core` | Спільне ядро video generation | Типи video generation, допоміжні функції failover, пошуку провайдера та розбору model-ref |
  | `plugin-sdk/interactive-runtime` | Допоміжні функції інтерактивних відповідей | Нормалізація / зменшення payload інтерактивних відповідей |
  | `plugin-sdk/channel-config-primitives` | Примітиви конфігурації каналу | Вузькі примітиви channel config-schema |
  | `plugin-sdk/channel-config-writes` | Допоміжні функції запису конфігурації каналу | Допоміжні функції авторизації запису конфігурації каналу |
  | `plugin-sdk/channel-plugin-common` | Спільна прелюдія каналу | Експорти спільної прелюдії channel plugin |
  | `plugin-sdk/channel-status` | Допоміжні функції статусу каналу | Спільні допоміжні функції знімка / підсумку статусу каналу |
  | `plugin-sdk/allowlist-config-edit` | Допоміжні функції конфігурації allowlist | Допоміжні функції редагування / читання конфігурації allowlist |
  | `plugin-sdk/group-access` | Допоміжні функції доступу до груп | Спільні допоміжні функції визначення доступу до груп |
  | `plugin-sdk/direct-dm` | Допоміжні функції direct-DM | Спільні допоміжні функції auth / guard для direct-DM |
  | `plugin-sdk/extension-shared` | Спільні допоміжні функції розширень | Примітиви допоміжних функцій passive-channel / status |
  | `plugin-sdk/webhook-targets` | Допоміжні функції цілей webhook | Реєстр цілей webhook і допоміжні функції встановлення маршрутів |
  | `plugin-sdk/webhook-path` | Допоміжні функції шляхів webhook | Допоміжні функції нормалізації шляхів webhook |
  | `plugin-sdk/web-media` | Спільні допоміжні функції web media | Допоміжні функції завантаження віддалених / локальних медіа |
  | `plugin-sdk/zod` | Повторний експорт Zod | Повторно експортований `zod` для споживачів plugin SDK |
  | `plugin-sdk/memory-core` | Вбудовані допоміжні функції memory-core | Поверхня допоміжних функцій менеджера пам’яті / конфігурації / файлів / CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Фасад runtime рушія пам’яті | Фасад runtime індексації / пошуку пам’яті |
  | `plugin-sdk/memory-core-host-engine-foundation` | Базовий рушій memory host | Експорти базового рушія memory host |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Рушій embedding memory host | Експорти рушія embedding memory host |
  | `plugin-sdk/memory-core-host-engine-qmd` | Рушій QMD memory host | Експорти рушія QMD memory host |
  | `plugin-sdk/memory-core-host-engine-storage` | Рушій сховища memory host | Експорти рушія сховища memory host |
  | `plugin-sdk/memory-core-host-multimodal` | Допоміжні функції multimodal memory host | Допоміжні функції multimodal memory host |
  | `plugin-sdk/memory-core-host-query` | Допоміжні функції query memory host | Допоміжні функції query memory host |
  | `plugin-sdk/memory-core-host-secret` | Допоміжні функції secret memory host | Допоміжні функції secret memory host |
  | `plugin-sdk/memory-core-host-events` | Допоміжні функції журналу подій memory host | Допоміжні функції журналу подій memory host |
  | `plugin-sdk/memory-core-host-status` | Допоміжні функції статусу memory host | Допоміжні функції статусу memory host |
  | `plugin-sdk/memory-core-host-runtime-cli` | CLI runtime memory host | Допоміжні функції CLI runtime memory host |
  | `plugin-sdk/memory-core-host-runtime-core` | Core runtime memory host | Допоміжні функції core runtime memory host |
  | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні функції файлів / runtime memory host | Допоміжні функції файлів / runtime memory host |
  | `plugin-sdk/memory-host-core` | Псевдонім core runtime memory host | Нейтральний щодо постачальника псевдонім допоміжних функцій core runtime memory host |
  | `plugin-sdk/memory-host-events` | Псевдонім журналу подій memory host | Нейтральний щодо постачальника псевдонім допоміжних функцій журналу подій memory host |
  | `plugin-sdk/memory-host-files` | Псевдонім файлів / runtime memory host | Нейтральний щодо постачальника псевдонім допоміжних функцій файлів / runtime memory host |
  | `plugin-sdk/memory-host-markdown` | Допоміжні функції керованого markdown | Спільні допоміжні функції керованого markdown для плагінів, суміжних із пам’яттю |
  | `plugin-sdk/memory-host-search` | Фасад пошуку активної пам’яті | Лінивий фасад runtime менеджера пошуку активної пам’яті |
  | `plugin-sdk/memory-host-status` | Псевдонім статусу memory host | Нейтральний щодо постачальника псевдонім допоміжних функцій статусу memory host |
  | `plugin-sdk/memory-lancedb` | Вбудовані допоміжні функції memory-lancedb | Поверхня допоміжних функцій memory-lancedb |
  | `plugin-sdk/testing` | Утиліти тестування | Допоміжні функції тестування та mocks |
</Accordion>

Ця таблиця навмисно містить поширену підмножину для міграції, а не повну
поверхню SDK. Повний список із понад 200 точок входу міститься у
`scripts/lib/plugin-sdk-entrypoints.json`.

Цей список досі містить деякі seam-інтерфейси допоміжних функцій вбудованих
плагінів, наприклад `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` і `plugin-sdk/matrix*`. Вони
й надалі експортуються для підтримки та сумісності вбудованих плагінів, але їх
навмисно пропущено в таблиці поширених шляхів міграції, і вони не є
рекомендованою ціллю для нового коду плагінів.

Те саме правило застосовується до інших сімейств вбудованих допоміжних
функцій, таких як:

- допоміжні функції підтримки браузера: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- поверхні вбудованих допоміжних функцій / плагінів, такі як `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` і `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` наразі надає вузьку поверхню допоміжних
функцій токена: `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken`.

Використовуйте найвужчий імпорт, який відповідає завданню. Якщо ви не можете
знайти експорт, перевірте вихідний код у `src/plugin-sdk/` або запитайте в Discord.

## Часова шкала видалення

| Коли                   | Що відбувається                                                         |
| ---------------------- | ----------------------------------------------------------------------- |
| **Зараз**              | Застарілі поверхні виводять попередження під час виконання              |
| **Наступний мажорний випуск** | Застарілі поверхні буде видалено; плагіни, які їх досі використовують, перестануть працювати |

Усі основні плагіни вже перенесено. Зовнішні плагіни слід перенести до
наступного мажорного випуску.

## Тимчасове приглушення попереджень

Установіть ці змінні середовища, поки працюєте над міграцією:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Це тимчасовий обхідний механізм, а не постійне рішення.

## Пов’язане

- [Початок роботи](/uk/plugins/building-plugins) — створіть свій перший плагін
- [Огляд SDK](/uk/plugins/sdk-overview) — повний довідник підшляхів імпорту
- [Плагіни каналів](/uk/plugins/sdk-channel-plugins) — створення плагінів каналів
- [Плагіни провайдерів](/uk/plugins/sdk-provider-plugins) — створення плагінів провайдерів
- [Внутрішня будова плагінів](/uk/plugins/architecture) — глибоке занурення в архітектуру
- [Маніфест плагіна](/uk/plugins/manifest) — довідник схеми маніфесту
