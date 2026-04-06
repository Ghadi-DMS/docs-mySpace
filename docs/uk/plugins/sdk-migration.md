---
read_when:
    - Ви бачите попередження OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ви бачите попередження OPENCLAW_EXTENSION_API_DEPRECATED
    - Ви оновлюєте plugin до сучасної архітектури plugin
    - Ви підтримуєте зовнішній plugin OpenClaw
sidebarTitle: Migrate to SDK
summary: Перейдіть із застарілого шару зворотної сумісності на сучасний plugin SDK
title: Міграція Plugin SDK
x-i18n:
    generated_at: "2026-04-06T04:11:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 94f12d1376edd8184714cc4dbea4a88fa8ed652f65e9365ede6176f3bf441b33
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Міграція Plugin SDK

OpenClaw перейшов від широкого шару зворотної сумісності до сучасної
архітектури plugin із цільовими, задокументованими імпортами. Якщо ваш plugin
було створено до появи нової архітектури, цей посібник допоможе вам виконати
міграцію.

## Що змінюється

Стара система plugin надавала дві дуже широкі поверхні, які дозволяли plugin
імпортувати все потрібне з однієї точки входу:

- **`openclaw/plugin-sdk/compat`** — один імпорт, який повторно експортував
  десятки допоміжних засобів. Його було запроваджено, щоб старіші plugin на
  основі hooks продовжували працювати, поки створювалася нова архітектура
  plugin.
- **`openclaw/extension-api`** — міст, який надавав plugins прямий доступ до
  допоміжних засобів на боці хоста, таких як вбудований запускальник агентів.

Обидві поверхні тепер **застарілі**. Вони все ще працюють під час виконання,
але нові plugins не повинні їх використовувати, а наявні plugins мають
перейти з них до того, як у наступному мажорному релізі їх буде видалено.

<Warning>
  Шар зворотної сумісності буде видалено в одному з майбутніх мажорних релізів.
  Plugins, які й далі імпортують із цих поверхонь, перестануть працювати,
  коли це станеться.
</Warning>

## Чому це змінилося

Старий підхід спричиняв проблеми:

- **Повільний запуск** — імпорт одного допоміжного засобу завантажував десятки
  не пов’язаних між собою модулів
- **Циклічні залежності** — широкі повторні експорти спрощували створення
  циклів імпорту
- **Нечітка поверхня API** — не було способу зрозуміти, які експорти є
  стабільними, а які — внутрішніми

Сучасний plugin SDK це виправляє: кожен шлях імпорту (`openclaw/plugin-sdk/\<subpath\>`)
є невеликим, самодостатнім модулем із чітким призначенням і задокументованим
контрактом.

Застарілі зручні seams провайдерів для вбудованих каналів також зникли.
Імпорти на кшталт `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
допоміжні seams із брендуванням каналу та
`openclaw/plugin-sdk/telegram-core` були приватними скороченнями для mono-repo,
а не стабільними контрактами plugin. Натомість використовуйте вузькі
загальні subpaths SDK. Усередині робочого простору вбудованих plugin
залишайте допоміжні засоби, що належать провайдеру, у власному
`api.ts` або `runtime-api.ts` цього plugin.

Поточні приклади вбудованих провайдерів:

- Anthropic зберігає допоміжні засоби потоку, специфічні для Claude, у
  власному seam `api.ts` / `contract-api.ts`
- OpenAI зберігає конструктори провайдерів, допоміжні засоби моделей за
  замовчуванням і конструктори realtime-провайдерів у власному `api.ts`
- OpenRouter зберігає конструктор провайдера та допоміжні засоби onboarding/config
  у власному `api.ts`

## Як виконати міграцію

<Steps>
  <Step title="Перевірте резервну поведінку Windows wrapper">
    Якщо ваш plugin використовує `openclaw/plugin-sdk/windows-spawn`, нерозв’язані
    wrapper-и Windows `.cmd`/`.bat` тепер безпечно завершуються помилкою, якщо
    ви явно не передасте `allowShellFallback: true`.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    Якщо ваш виклик не покладається навмисно на резервний перехід через shell,
    не встановлюйте `allowShellFallback` і натомість обробляйте викинуту
    помилку.

  </Step>

  <Step title="Знайдіть застарілі імпорти">
    Знайдіть у своєму plugin імпорти з будь-якої із застарілих поверхонь:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Замініть їх на цільові імпорти">
    Кожен експорт зі старої поверхні зіставляється з певним сучасним шляхом
    імпорту:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Для допоміжних засобів на боці хоста використовуйте впроваджений runtime
    plugin замість прямого імпорту:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Той самий шаблон застосовується й до інших застарілих допоміжних засобів
    bridge:

    | Старий імпорт | Сучасний еквівалент |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | допоміжні засоби session store | `api.runtime.agent.session.*` |

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
  | `plugin-sdk/plugin-entry` | Канонічний допоміжний засіб точки входу plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Застарілий парасольковий повторний експорт для визначень/конструкторів входу каналу | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Експорт кореневої схеми config | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Допоміжний засіб точки входу для одного провайдера | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Цільові визначення та конструктори входу каналу | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Спільні допоміжні засоби майстра налаштування | Запити allowlist, конструктори статусу налаштування |
  | `plugin-sdk/setup-runtime` | Допоміжні засоби runtime для етапу налаштування | Адаптери patch налаштування, безпечні для імпорту, допоміжні засоби для приміток пошуку, `promptResolvedAllowFrom`, `splitSetupEntries`, делеговані проксі налаштування |
  | `plugin-sdk/setup-adapter-runtime` | Допоміжні засоби адаптера налаштування | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Допоміжні засоби інструментарію налаштування | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Допоміжні засоби для кількох облікових записів | Допоміжні засоби списку облікових записів/config/action-gate |
  | `plugin-sdk/account-id` | Допоміжні засоби account-id | `DEFAULT_ACCOUNT_ID`, нормалізація account-id |
  | `plugin-sdk/account-resolution` | Допоміжні засоби пошуку облікового запису | Допоміжні засоби пошуку облікового запису + fallback за замовчуванням |
  | `plugin-sdk/account-helpers` | Вузькі допоміжні засоби облікового запису | Допоміжні засоби списку облікових записів/дій з обліковим записом |
  | `plugin-sdk/channel-setup` | Адаптери майстра налаштування | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Примітиви сполучення DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Підключення префікса відповіді + індикатора набору | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Фабрики адаптерів config | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Конструктори схем config | Типи схем config каналу |
  | `plugin-sdk/telegram-command-config` | Допоміжні засоби config команд Telegram | Нормалізація назв команд, обрізання описів, перевірка дублікатів/конфліктів |
  | `plugin-sdk/channel-policy` | Визначення політики груп/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Відстеження статусу облікового запису | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Допоміжні засоби вхідного envelope | Спільні допоміжні засоби маршрутизації + конструктора envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Допоміжні засоби вхідної відповіді | Спільні допоміжні засоби запису та dispatch |
  | `plugin-sdk/messaging-targets` | Розбір цілей повідомлень | Допоміжні засоби розбору/зіставлення цілей |
  | `plugin-sdk/outbound-media` | Допоміжні засоби вихідних медіа | Спільне завантаження вихідних медіа |
  | `plugin-sdk/outbound-runtime` | Допоміжні засоби вихідного runtime | Допоміжні засоби ідентичності вихідних даних/send delegate |
  | `plugin-sdk/thread-bindings-runtime` | Допоміжні засоби прив’язок потоків | Життєвий цикл прив’язки потоків і допоміжні засоби адаптера |
  | `plugin-sdk/agent-media-payload` | Застарілі допоміжні засоби media payload | Конструктор media payload агента для застарілих макетів полів |
  | `plugin-sdk/channel-runtime` | Застарілий shim сумісності | Лише застарілі утиліти runtime каналу |
  | `plugin-sdk/channel-send-result` | Типи результату send | Типи результату відповіді |
  | `plugin-sdk/runtime-store` | Постійне сховище plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Широкі допоміжні засоби runtime | Допоміжні засоби runtime/logging/backup/встановлення plugin |
  | `plugin-sdk/runtime-env` | Вузькі допоміжні засоби env runtime | Допоміжні засоби logger/env runtime, timeout, retry і backoff |
  | `plugin-sdk/plugin-runtime` | Спільні допоміжні засоби runtime plugin | Допоміжні засоби plugin commands/hooks/http/interactive |
  | `plugin-sdk/hook-runtime` | Допоміжні засоби pipeline hooks | Спільні допоміжні засоби pipeline webhook/internal hook |
  | `plugin-sdk/lazy-runtime` | Допоміжні засоби lazy runtime | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Допоміжні засоби процесів | Спільні допоміжні засоби exec |
  | `plugin-sdk/cli-runtime` | Допоміжні засоби runtime CLI | Форматування команд, очікування, допоміжні засоби версії |
  | `plugin-sdk/gateway-runtime` | Допоміжні засоби gateway | Клієнт gateway і допоміжні засоби patch статусу каналу |
  | `plugin-sdk/config-runtime` | Допоміжні засоби config | Допоміжні засоби завантаження/запису config |
  | `plugin-sdk/telegram-command-config` | Допоміжні засоби команд Telegram | Допоміжні засоби перевірки команд Telegram зі стабільним fallback, коли поверхня контракту вбудованого Telegram недоступна |
  | `plugin-sdk/approval-runtime` | Допоміжні засоби запиту approval | Payload approval для exec/plugin, допоміжні засоби capability/profile approval, нативні допоміжні засоби маршрутизації/runtime approval |
  | `plugin-sdk/approval-auth-runtime` | Допоміжні засоби auth approval | Визначення approver, auth дій у тому самому чаті |
  | `plugin-sdk/approval-client-runtime` | Допоміжні засоби клієнта approval | Нативні допоміжні засоби profile/filter approval для exec |
  | `plugin-sdk/approval-delivery-runtime` | Допоміжні засоби доставки approval | Нативні адаптери capability/delivery approval |
  | `plugin-sdk/approval-native-runtime` | Допоміжні засоби цілей approval | Нативні допоміжні засоби прив’язки цілей/облікових записів approval |
  | `plugin-sdk/approval-reply-runtime` | Допоміжні засоби відповіді approval | Допоміжні засоби payload відповіді approval для exec/plugin |
  | `plugin-sdk/security-runtime` | Допоміжні засоби безпеки | Спільні допоміжні засоби trust, DM gating, external-content і збирання секретів |
  | `plugin-sdk/ssrf-policy` | Допоміжні засоби політики SSRF | Допоміжні засоби allowlist хостів і політики приватної мережі |
  | `plugin-sdk/ssrf-runtime` | Допоміжні засоби runtime SSRF | Допоміжні засоби pinned-dispatcher, guarded fetch, політики SSRF |
  | `plugin-sdk/collection-runtime` | Допоміжні засоби обмеженого кешу | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Допоміжні засоби діагностичного gating | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Допоміжні засоби форматування помилок | `formatUncaughtError`, `isApprovalNotFoundError`, допоміжні засоби графа помилок |
  | `plugin-sdk/fetch-runtime` | Допоміжні засоби обгорнутого fetch/proxy | `resolveFetch`, допоміжні засоби proxy |
  | `plugin-sdk/host-runtime` | Допоміжні засоби нормалізації хоста | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Допоміжні засоби retry | `RetryConfig`, `retryAsync`, засоби запуску політик |
  | `plugin-sdk/allow-from` | Форматування allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Мапування вхідних даних allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Допоміжні засоби gating команд і поверхні команд | `resolveControlCommandGate`, допоміжні засоби авторизації відправника, допоміжні засоби реєстру команд |
  | `plugin-sdk/secret-input` | Розбір секретного вводу | Допоміжні засоби секретного вводу |
  | `plugin-sdk/webhook-ingress` | Допоміжні засоби запитів webhook | Утиліти цілей webhook |
  | `plugin-sdk/webhook-request-guards` | Допоміжні засоби guard для тіла webhook-запиту | Допоміжні засоби читання/обмеження тіла запиту |
  | `plugin-sdk/reply-runtime` | Спільний runtime відповіді | Вхідний dispatch, heartbeat, планувальник відповідей, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні засоби dispatch відповіді | Допоміжні засоби finalize + dispatch провайдера |
  | `plugin-sdk/reply-history` | Допоміжні засоби історії відповідей | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Планування reference відповіді | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Допоміжні засоби chunk відповіді | Допоміжні засоби chunking тексту/markdown |
  | `plugin-sdk/session-store-runtime` | Допоміжні засоби session store | Допоміжні засоби шляху сховища + updated-at |
  | `plugin-sdk/state-paths` | Допоміжні засоби шляхів стану | Допоміжні засоби каталогів state і OAuth |
  | `plugin-sdk/routing` | Допоміжні засоби маршрутизації/ключа session | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, допоміжні засоби нормалізації ключа session |
  | `plugin-sdk/status-helpers` | Допоміжні засоби статусу каналу | Конструктори зведень статусу каналу/облікового запису, типові значення runtime-state, допоміжні засоби метаданих проблем |
  | `plugin-sdk/target-resolver-runtime` | Допоміжні засоби визначення цілі | Спільні допоміжні засоби target resolver |
  | `plugin-sdk/string-normalization-runtime` | Допоміжні засоби нормалізації рядків | Допоміжні засоби нормалізації slug/рядків |
  | `plugin-sdk/request-url` | Допоміжні засоби URL запиту | Витягування рядкових URL з request-подібних вхідних даних |
  | `plugin-sdk/run-command` | Допоміжні засоби команд із таймером | Виконавець команд із нормалізованими stdout/stderr |
  | `plugin-sdk/param-readers` | Зчитувачі параметрів | Загальні зчитувачі параметрів tool/CLI |
  | `plugin-sdk/tool-send` | Витягування send для tool | Витягування канонічних полів цілі send з аргументів tool |
  | `plugin-sdk/temp-path` | Допоміжні засоби тимчасових шляхів | Спільні допоміжні засоби тимчасового шляху для завантаження |
  | `plugin-sdk/logging-core` | Допоміжні засоби logging | Допоміжні засоби logger підсистеми та редагування |
  | `plugin-sdk/markdown-table-runtime` | Допоміжні засоби markdown-table | Допоміжні засоби режиму таблиць markdown |
  | `plugin-sdk/reply-payload` | Типи payload відповіді повідомлення | Типи payload відповіді |
  | `plugin-sdk/provider-setup` | Кураторські допоміжні засоби налаштування локальних/self-hosted провайдерів | Допоміжні засоби виявлення/config self-hosted провайдера |
  | `plugin-sdk/self-hosted-provider-setup` | Цільові допоміжні засоби налаштування self-hosted провайдерів, сумісних з OpenAI | Ті самі допоміжні засоби виявлення/config self-hosted провайдера |
  | `plugin-sdk/provider-auth-runtime` | Допоміжні засоби auth runtime провайдера | Допоміжні засоби визначення API key під час виконання |
  | `plugin-sdk/provider-auth-api-key` | Допоміжні засоби налаштування API key провайдера | Допоміжні засоби onboarding/profile-write для API key |
  | `plugin-sdk/provider-auth-result` | Допоміжні засоби результату auth провайдера | Стандартний конструктор результату auth OAuth |
  | `plugin-sdk/provider-auth-login` | Допоміжні засоби інтерактивного входу провайдера | Спільні допоміжні засоби інтерактивного входу |
  | `plugin-sdk/provider-env-vars` | Допоміжні засоби env vars провайдера | Допоміжні засоби пошуку env vars auth провайдера |
  | `plugin-sdk/provider-model-shared` | Спільні допоміжні засоби моделей/повторного відтворення провайдера | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні конструктори політики повторного відтворення, допоміжні засоби endpoint провайдера та нормалізації model-id |
  | `plugin-sdk/provider-catalog-shared` | Спільні допоміжні засоби каталогу провайдера | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patches onboarding провайдера | Допоміжні засоби config onboarding |
  | `plugin-sdk/provider-http` | Допоміжні засоби HTTP провайдера | Загальні допоміжні засоби HTTP/можливостей endpoint провайдера |
  | `plugin-sdk/provider-web-fetch` | Допоміжні засоби web-fetch провайдера | Допоміжні засоби реєстрації/кешу провайдера web-fetch |
  | `plugin-sdk/provider-web-search` | Допоміжні засоби web-search провайдера | Допоміжні засоби реєстрації/кешу/config провайдера web-search |
  | `plugin-sdk/provider-tools` | Допоміжні засоби сумісності tool/schema провайдера | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схеми Gemini + діагностика, а також допоміжні засоби сумісності xAI, такі як `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Допоміжні засоби використання провайдера | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` та інші допоміжні засоби використання провайдера |
  | `plugin-sdk/provider-stream` | Допоміжні засоби обгортки потоку провайдера | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи обгорток потоку та спільні допоміжні засоби обгорток Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Упорядкована асинхронна черга | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Спільні допоміжні засоби медіа | Допоміжні засоби fetch/transform/store медіа та конструктори media payload |
  | `plugin-sdk/media-generation-runtime` | Спільні допоміжні засоби генерації медіа | Спільні допоміжні засоби failover, вибору кандидатів і повідомлень про відсутню модель для генерації зображень/відео/музики |
  | `plugin-sdk/media-understanding` | Допоміжні засоби media-understanding | Типи провайдерів media understanding та експорти допоміжних засобів зображення/аудіо для провайдера |
  | `plugin-sdk/text-runtime` | Спільні допоміжні засоби тексту | Видалення тексту, видимого асистенту, допоміжні засоби render/chunking/table для markdown, допоміжні засоби редагування, допоміжні засоби тегів директив, утиліти безпечного тексту та пов’язані допоміжні засоби text/logging |
  | `plugin-sdk/text-chunking` | Допоміжні засоби chunking тексту | Допоміжний засіб chunking вихідного тексту |
  | `plugin-sdk/speech` | Допоміжні засоби speech | Типи провайдерів speech та експорти допоміжних засобів directives, registry і validation для провайдера |
  | `plugin-sdk/speech-core` | Спільне ядро speech | Типи провайдерів speech, registry, directives, normalization |
  | `plugin-sdk/realtime-transcription` | Допоміжні засоби realtime transcription | Типи провайдерів і допоміжні засоби registry |
  | `plugin-sdk/realtime-voice` | Допоміжні засоби realtime voice | Типи провайдерів і допоміжні засоби registry |
  | `plugin-sdk/image-generation-core` | Спільне ядро генерації зображень | Допоміжні засоби типів, failover, auth і registry для генерації зображень |
  | `plugin-sdk/music-generation` | Допоміжні засоби генерації музики | Типи провайдера/запиту/результату генерації музики |
  | `plugin-sdk/music-generation-core` | Спільне ядро генерації музики | Допоміжні засоби типів, failover, пошуку провайдера та розбору model-ref для генерації музики |
  | `plugin-sdk/video-generation` | Допоміжні засоби генерації відео | Типи провайдера/запиту/результату генерації відео |
  | `plugin-sdk/video-generation-core` | Спільне ядро генерації відео | Допоміжні засоби типів, failover, пошуку провайдера та розбору model-ref для генерації відео |
  | `plugin-sdk/interactive-runtime` | Допоміжні засоби інтерактивної відповіді | Нормалізація/зменшення payload інтерактивної відповіді |
  | `plugin-sdk/channel-config-primitives` | Примітиви config каналу | Вузькі примітиви channel config-schema |
  | `plugin-sdk/channel-config-writes` | Допоміжні засоби запису config каналу | Допоміжні засоби авторизації запису config каналу |
  | `plugin-sdk/channel-plugin-common` | Спільний прелюд каналу | Експорти спільного прелюду channel plugin |
  | `plugin-sdk/channel-status` | Допоміжні засоби статусу каналу | Спільні допоміжні засоби snapshot/summary статусу каналу |
  | `plugin-sdk/allowlist-config-edit` | Допоміжні засоби config allowlist | Допоміжні засоби редагування/читання config allowlist |
  | `plugin-sdk/group-access` | Допоміжні засоби доступу до групи | Спільні допоміжні засоби рішень щодо group-access |
  | `plugin-sdk/direct-dm` | Допоміжні засоби direct-DM | Спільні допоміжні засоби auth/guard для direct-DM |
  | `plugin-sdk/extension-shared` | Спільні допоміжні засоби extension | Примітиви допоміжних засобів passive-channel/status |
  | `plugin-sdk/webhook-targets` | Допоміжні засоби цілей webhook | Реєстр цілей webhook і допоміжні засоби встановлення маршрутів |
  | `plugin-sdk/webhook-path` | Допоміжні засоби шляху webhook | Допоміжні засоби нормалізації шляху webhook |
  | `plugin-sdk/web-media` | Спільні допоміжні засоби web media | Допоміжні засоби завантаження віддалених/локальних медіа |
  | `plugin-sdk/zod` | Повторний експорт Zod | Повторно експортований `zod` для споживачів plugin SDK |
  | `plugin-sdk/memory-core` | Вбудовані допоміжні засоби memory-core | Поверхня допоміжних засобів memory manager/config/file/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Фасад runtime рушія пам’яті | Фасад runtime індексації/пошуку пам’яті |
  | `plugin-sdk/memory-core-host-engine-foundation` | Базовий рушій хоста пам’яті | Експорти базового рушія хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Рушій embedding хоста пам’яті | Експорти рушія embedding хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-qmd` | Рушій QMD хоста пам’яті | Експорти рушія QMD хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-storage` | Рушій сховища хоста пам’яті | Експорти рушія сховища хоста пам’яті |
  | `plugin-sdk/memory-core-host-multimodal` | Мультимодальні допоміжні засоби хоста пам’яті | Мультимодальні допоміжні засоби хоста пам’яті |
  | `plugin-sdk/memory-core-host-query` | Допоміжні засоби запитів хоста пам’яті | Допоміжні засоби запитів хоста пам’яті |
  | `plugin-sdk/memory-core-host-secret` | Допоміжні засоби секретів хоста пам’яті | Допоміжні засоби секретів хоста пам’яті |
  | `plugin-sdk/memory-core-host-events` | Допоміжні засоби журналу подій хоста пам’яті | Допоміжні засоби журналу подій хоста пам’яті |
  | `plugin-sdk/memory-core-host-status` | Допоміжні засоби статусу хоста пам’яті | Допоміжні засоби статусу хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-cli` | CLI runtime хоста пам’яті | Допоміжні засоби CLI runtime хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-core` | Базовий runtime хоста пам’яті | Допоміжні засоби базового runtime хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні засоби файлів/runtime хоста пам’яті | Допоміжні засоби файлів/runtime хоста пам’яті |
  | `plugin-sdk/memory-host-core` | Псевдонім базового runtime хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів базового runtime хоста пам’яті |
  | `plugin-sdk/memory-host-events` | Псевдонім журналу подій хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів журналу подій хоста пам’яті |
  | `plugin-sdk/memory-host-files` | Псевдонім файлів/runtime хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів файлів/runtime хоста пам’яті |
  | `plugin-sdk/memory-host-markdown` | Допоміжні засоби керованого markdown | Спільні допоміжні засоби керованого markdown для plugin, суміжних із пам’яттю |
  | `plugin-sdk/memory-host-search` | Фасад пошуку активної пам’яті | Лінивий фасад runtime search-manager активної пам’яті |
  | `plugin-sdk/memory-host-status` | Псевдонім статусу хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів статусу хоста пам’яті |
  | `plugin-sdk/memory-lancedb` | Вбудовані допоміжні засоби memory-lancedb | Поверхня допоміжних засобів memory-lancedb |
  | `plugin-sdk/testing` | Утиліти тестування | Допоміжні засоби тестування та mocks |
</Accordion>

Ця таблиця навмисно містить поширену підмножину для міграції, а не повну
поверхню SDK. Повний список із понад 200 точок входу міститься у
`scripts/lib/plugin-sdk-entrypoints.json`.

У цьому списку все ще є деякі seams допоміжних засобів вбудованих plugin,
такі як `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` і `plugin-sdk/matrix*`. Вони й далі експортуються для
підтримки та сумісності вбудованих plugin, але навмисно не включені до
таблиці поширеної міграції й не є рекомендованою ціллю для нового коду plugin.

Те саме правило застосовується до інших сімейств вбудованих допоміжних засобів,
таких як:

- допоміжні засоби підтримки браузера: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- поверхні вбудованих допоміжних засобів/plugin, такі як `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` і `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` наразі надає вузьку поверхню допоміжних
засобів токена `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` і `resolveCopilotApiToken`.

Використовуйте найвужчий імпорт, який відповідає завданню. Якщо ви не можете
знайти потрібний експорт, перегляньте вихідний код у `src/plugin-sdk/` або
запитайте в Discord.

## Часова шкала видалення

| Коли | Що відбувається |
| ---------------------- | ----------------------------------------------------------------------- |
| **Зараз** | Застарілі поверхні виводять попередження під час виконання |
| **Наступний мажорний реліз** | Застарілі поверхні буде видалено; plugins, які все ще їх використовують, перестануть працювати |

Усі core plugins уже мігровано. Зовнішні plugins мають виконати міграцію
до наступного мажорного релізу.

## Тимчасове приглушення попереджень

Поки ви працюєте над міграцією, установіть ці змінні середовища:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Це тимчасовий обхідний механізм, а не постійне рішення.

## Пов’язане

- [Початок роботи](/uk/plugins/building-plugins) — створіть свій перший plugin
- [Огляд SDK](/uk/plugins/sdk-overview) — повний довідник імпортів subpath
- [Channel Plugins](/uk/plugins/sdk-channel-plugins) — створення channel plugins
- [Provider Plugins](/uk/plugins/sdk-provider-plugins) — створення provider plugins
- [Внутрішня будова Plugins](/uk/plugins/architecture) — глибоке занурення в архітектуру
- [Маніфест Plugin](/uk/plugins/manifest) — довідник зі схеми маніфесту
