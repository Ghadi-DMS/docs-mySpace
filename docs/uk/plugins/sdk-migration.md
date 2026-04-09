---
read_when:
    - Ви бачите попередження OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Ви бачите попередження OPENCLAW_EXTENSION_API_DEPRECATED
    - Ви оновлюєте плагін до сучасної архітектури плагінів
    - Ви підтримуєте зовнішній плагін OpenClaw
sidebarTitle: Migrate to SDK
summary: Перейдіть зі застарілого шару зворотної сумісності на сучасний SDK плагінів
title: Міграція SDK плагінів
x-i18n:
    generated_at: "2026-04-09T00:38:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60cbb6c8be30d17770887d490c14e3a4538563339a5206fb419e51e0558bbc07
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Міграція SDK плагінів

OpenClaw перейшов від широкого шару зворотної сумісності до сучасної
архітектури плагінів із цільовими документованими імпортами. Якщо ваш плагін
було створено до появи нової архітектури, цей посібник допоможе вам
виконати міграцію.

## Що змінюється

Стара система плагінів надавала дві надто широкі поверхні, які дозволяли
плагінам імпортувати будь-що потрібне з однієї точки входу:

- **`openclaw/plugin-sdk/compat`** — один імпорт, який повторно експортував
  десятки допоміжних функцій. Його було запроваджено, щоб старіші плагіни на
  основі hooks продовжували працювати, поки будувалася нова архітектура
  плагінів.
- **`openclaw/extension-api`** — міст, який надавав плагінам прямий доступ до
  допоміжних засобів на боці хоста, наприклад до вбудованого запускальника
  агента.

Обидві поверхні тепер **застарілі**. Вони все ще працюють під час виконання,
але нові плагіни не повинні їх використовувати, а наявні плагіни мають
перейти до наступного мажорного релізу, перш ніж їх буде вилучено.

<Warning>
  Шар зворотної сумісності буде вилучено в одному з майбутніх мажорних
  релізів. Плагіни, які й надалі імпортують із цих поверхонь, перестануть
  працювати, коли це станеться.
</Warning>

## Чому це змінилося

Старий підхід спричиняв проблеми:

- **Повільний запуск** — імпорт однієї допоміжної функції завантажував десятки
  не пов’язаних між собою модулів
- **Циклічні залежності** — широкі повторні експорти спрощували створення
  циклів імпорту
- **Нечітка поверхня API** — не було способу визначити, які експорти є
  стабільними, а які внутрішніми

Сучасний SDK плагінів виправляє це: кожен шлях імпорту (`openclaw/plugin-sdk/\<subpath\>`)
є невеликим самодостатнім модулем із чіткою метою та документованим контрактом.

Застарілі зручні seams провайдерів для вбудованих каналів також зникли.
Імпорти на кшталт `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
канальні branded helper seams та
`openclaw/plugin-sdk/telegram-core` були приватними скороченнями монорепозиторію,
а не стабільними контрактами плагінів. Натомість використовуйте вузькі
загальні підшляхи SDK. Усередині робочого простору вбудованих плагінів
залишайте допоміжні засоби, що належать провайдеру, у власному
`api.ts` або `runtime-api.ts` цього плагіна.

Поточні приклади вбудованих провайдерів:

- Anthropic зберігає специфічні для Claude допоміжні засоби потоків у власному
  seam `api.ts` / `contract-api.ts`
- OpenAI зберігає builder-и провайдерів, допоміжні засоби моделей за
  замовчуванням і builder-и realtime-провайдерів у власному `api.ts`
- OpenRouter зберігає builder провайдера та допоміжні засоби
  онбордингу/конфігурації у власному `api.ts`

## Як виконати міграцію

<Steps>
  <Step title="Перенесіть обробники з нативним схваленням на факти можливостей">
    Плагіни каналів із підтримкою схвалення тепер надають нативну поведінку
    схвалення через `approvalCapability.nativeRuntime` разом зі спільним
    реєстром контексту runtime.

    Основні зміни:

    - Замініть `approvalCapability.handler.loadRuntime(...)` на
      `approvalCapability.nativeRuntime`
    - Перенесіть специфічні для схвалення auth/доставку зі застарілої
      прив’язки `plugin.auth` / `plugin.approvals` на `approvalCapability`
    - `ChannelPlugin.approvals` вилучено з публічного контракту channel-plugin;
      перемістіть поля delivery/native/render до `approvalCapability`
    - `plugin.auth` залишається лише для потоків входу/виходу каналу; hooks
      auth для схвалення там більше не зчитуються ядром
    - Реєструйте об’єкти runtime, що належать каналу, такі як клієнти, токени
      або застосунки Bolt, через `openclaw/plugin-sdk/channel-runtime-context`
    - Не надсилайте сповіщення про перенаправлення, що належать плагіну, з
      нативних обробників схвалення; ядро тепер саме володіє сповіщеннями
      про доставку в інше місце на основі фактичних результатів доставки
    - Під час передавання `channelRuntime` до `createChannelManager(...)`
      надавайте справжню поверхню `createPluginRuntime().channel`.
      Часткові заглушки відхиляються.

    Актуальну структуру можливостей схвалення дивіться в
    `/plugins/sdk-channel-plugins`.

  </Step>

  <Step title="Перевірте резервну поведінку Windows wrapper">
    Якщо ваш плагін використовує `openclaw/plugin-sdk/windows-spawn`, нерозв’язані
    wrapper-и Windows `.cmd`/`.bat` тепер завершуються із закритою відмовою,
    якщо ви явно не передасте `allowShellFallback: true`.

    ```typescript
    // До
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Після
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Установлюйте це лише для довірених сумісних викликів, які свідомо
      // допускають резервний перехід через shell.
      allowShellFallback: true,
    });
    ```

    Якщо ваш виклик не покладається навмисно на резервний перехід через shell,
    не встановлюйте `allowShellFallback` і натомість обробляйте викинуту
    помилку.

  </Step>

  <Step title="Знайдіть застарілі імпорти">
    Пошукайте у своєму плагіні імпорти з будь-якої із застарілих поверхонь:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Замініть їх на цільові імпорти">
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

    Для допоміжних засобів на боці хоста використовуйте впроваджений runtime
    плагіна замість прямого імпорту:

    ```typescript
    // До (застарілий міст extension-api)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Після (впроваджений runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Той самий шаблон застосовується і до інших застарілих допоміжних засобів
    моста:

    | Старий імпорт | Сучасний еквівалент |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | session store helpers | `api.runtime.agent.session.*` |

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
  | `plugin-sdk/plugin-entry` | Канонічний допоміжний засіб точки входу плагіна | `definePluginEntry` |
  | `plugin-sdk/core` | Застарілий umbrella re-export для визначень/builder-ів входу каналів | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Експорт кореневої схеми конфігурації | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Допоміжний засіб точки входу для одного провайдера | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Цільові визначення та builder-и входу каналів | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Спільні допоміжні засоби майстра налаштування | Запити allowlist, builder-и стану налаштування |
  | `plugin-sdk/setup-runtime` | Допоміжні засоби runtime під час налаштування | Безпечні для імпорту setup patch adapters, допоміжні засоби приміток пошуку, `promptResolvedAllowFrom`, `splitSetupEntries`, проксі делегованого налаштування |
  | `plugin-sdk/setup-adapter-runtime` | Допоміжні засоби adapter налаштування | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Допоміжні засоби інструментів налаштування | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Допоміжні засоби для кількох облікових записів | Допоміжні засоби списків/config/action-gate облікових записів |
  | `plugin-sdk/account-id` | Допоміжні засоби account-id | `DEFAULT_ACCOUNT_ID`, нормалізація account-id |
  | `plugin-sdk/account-resolution` | Допоміжні засоби пошуку облікових записів | Допоміжні засоби пошуку облікових записів + fallback за замовчуванням |
  | `plugin-sdk/account-helpers` | Вузькі допоміжні засоби для облікових записів | Допоміжні засоби списку облікових записів/дій з обліковими записами |
  | `plugin-sdk/channel-setup` | Адаптери майстра налаштування | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, а також `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Примітиви DM-поєднання | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Префікс відповіді + прив’язка typing | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Фабрики адаптерів конфігурації | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builder-и схем конфігурації | Типи схем конфігурації каналів |
  | `plugin-sdk/telegram-command-config` | Допоміжні засоби конфігурації команд Telegram | Нормалізація назв команд, обрізання описів, валідація дублікатів/конфліктів |
  | `plugin-sdk/channel-policy` | Визначення політики груп/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Відстеження стану облікових записів | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Допоміжні засоби вхідних envelope | Спільні допоміжні засоби route + builder-ів envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Допоміжні засоби вхідних відповідей | Спільні допоміжні засоби запису та dispatch |
  | `plugin-sdk/messaging-targets` | Розбір цілей повідомлень | Допоміжні засоби розбору/зіставлення цілей |
  | `plugin-sdk/outbound-media` | Допоміжні засоби вихідних медіа | Спільне завантаження вихідних медіа |
  | `plugin-sdk/outbound-runtime` | Допоміжні засоби вихідного runtime | Допоміжні засоби outbound identity/send delegate |
  | `plugin-sdk/thread-bindings-runtime` | Допоміжні засоби прив’язок потоків | Допоміжні засоби життєвого циклу прив’язок потоків та adapter |
  | `plugin-sdk/agent-media-payload` | Застарілі допоміжні засоби media payload | Builder agent media payload для застарілих макетів полів |
  | `plugin-sdk/channel-runtime` | Застарілий shim сумісності | Лише застарілі утиліти channel runtime |
  | `plugin-sdk/channel-send-result` | Типи результатів надсилання | Типи результатів відповідей |
  | `plugin-sdk/runtime-store` | Постійне сховище плагінів | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Широкі допоміжні засоби runtime | Допоміжні засоби runtime/logging/backup/встановлення плагінів |
  | `plugin-sdk/runtime-env` | Вузькі допоміжні засоби runtime env | Logger/runtime env, допоміжні засоби timeout, retry та backoff |
  | `plugin-sdk/plugin-runtime` | Спільні допоміжні засоби runtime плагінів | Допоміжні засоби команд/hooks/http/interactive плагіна |
  | `plugin-sdk/hook-runtime` | Допоміжні засоби pipeline hooks | Спільні допоміжні засоби pipeline webhook/internal hook |
  | `plugin-sdk/lazy-runtime` | Допоміжні засоби ледачого runtime | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Допоміжні засоби процесів | Спільні допоміжні засоби exec |
  | `plugin-sdk/cli-runtime` | Допоміжні засоби CLI runtime | Форматування команд, очікування, допоміжні засоби версій |
  | `plugin-sdk/gateway-runtime` | Допоміжні засоби gateway | Допоміжні засоби клієнта gateway і patch-ів стану каналів |
  | `plugin-sdk/config-runtime` | Допоміжні засоби конфігурації | Допоміжні засоби завантаження/запису конфігурації |
  | `plugin-sdk/telegram-command-config` | Допоміжні засоби команд Telegram | Допоміжні засоби валідації команд Telegram зі стабільним fallback, коли поверхня контракту вбудованого Telegram недоступна |
  | `plugin-sdk/approval-runtime` | Допоміжні засоби запитів схвалення | Payload схвалення exec/plugin, допоміжні засоби capability/profile схвалення, допоміжні засоби маршрутизації/runtime нативного схвалення |
  | `plugin-sdk/approval-auth-runtime` | Допоміжні засоби auth схвалення | Визначення approver, auth дій у тому самому чаті |
  | `plugin-sdk/approval-client-runtime` | Допоміжні засоби клієнта схвалення | Допоміжні засоби profile/filter нативного exec-схвалення |
  | `plugin-sdk/approval-delivery-runtime` | Допоміжні засоби доставки схвалення | Адаптери можливостей/доставки нативного схвалення |
  | `plugin-sdk/approval-gateway-runtime` | Допоміжні засоби gateway схвалення | Спільний допоміжний засіб визначення gateway схвалення |
  | `plugin-sdk/approval-handler-adapter-runtime` | Допоміжні засоби adapter схвалення | Полегшені допоміжні засоби завантаження adapter-ів нативного схвалення для гарячих точок входу каналів |
  | `plugin-sdk/approval-handler-runtime` | Допоміжні засоби обробників схвалення | Ширші допоміжні засоби runtime обробників схвалення; віддавайте перевагу вужчим seams adapter/gateway, коли їх достатньо |
  | `plugin-sdk/approval-native-runtime` | Допоміжні засоби цілей схвалення | Допоміжні засоби нативної прив’язки цілі/облікового запису схвалення |
  | `plugin-sdk/approval-reply-runtime` | Допоміжні засоби відповідей на схвалення | Допоміжні засоби payload відповіді для exec/plugin схвалення |
  | `plugin-sdk/channel-runtime-context` | Допоміжні засоби контексту runtime каналу | Узагальнені допоміжні засоби register/get/watch для контексту runtime каналу |
  | `plugin-sdk/security-runtime` | Допоміжні засоби безпеки | Спільні допоміжні засоби довіри, DM gating, зовнішнього вмісту та збирання секретів |
  | `plugin-sdk/ssrf-policy` | Допоміжні засоби політики SSRF | Допоміжні засоби allowlist хостів і політики приватної мережі |
  | `plugin-sdk/ssrf-runtime` | Допоміжні засоби SSRF runtime | Pinned-dispatcher, guarded fetch, допоміжні засоби політики SSRF |
  | `plugin-sdk/collection-runtime` | Допоміжні засоби обмеженого кешу | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Допоміжні засоби діагностичного gating | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Допоміжні засоби форматування помилок | `formatUncaughtError`, `isApprovalNotFoundError`, допоміжні засоби графа помилок |
  | `plugin-sdk/fetch-runtime` | Допоміжні засоби обгорнутого fetch/proxy | `resolveFetch`, допоміжні засоби proxy |
  | `plugin-sdk/host-runtime` | Допоміжні засоби нормалізації хоста | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Допоміжні засоби retry | `RetryConfig`, `retryAsync`, виконавці політик |
  | `plugin-sdk/allow-from` | Форматування allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Мапування вводу allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Допоміжні засоби gating команд і поверхні команд | `resolveControlCommandGate`, допоміжні засоби авторизації відправника, допоміжні засоби реєстру команд |
  | `plugin-sdk/command-status` | Рендерери стану/довідки команд | `buildCommandsMessage`, `buildCommandsMessagePaginated`, `buildHelpMessage` |
  | `plugin-sdk/secret-input` | Розбір вводу секретів | Допоміжні засоби вводу секретів |
  | `plugin-sdk/webhook-ingress` | Допоміжні засоби запитів webhook | Утиліти цілей webhook |
  | `plugin-sdk/webhook-request-guards` | Допоміжні засоби guards для тіла webhook | Допоміжні засоби читання/обмеження тіла запиту |
  | `plugin-sdk/reply-runtime` | Спільний runtime відповідей | Вхідний dispatch, heartbeat, планувальник відповідей, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Вузькі допоміжні засоби dispatch відповідей | Допоміжні засоби finalize + dispatch провайдера |
  | `plugin-sdk/reply-history` | Допоміжні засоби історії відповідей | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Планування посилань у відповідях | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Допоміжні засоби chunk відповідей | Допоміжні засоби chunking тексту/markdown |
  | `plugin-sdk/session-store-runtime` | Допоміжні засоби сховища сесій | Допоміжні засоби шляху сховища + updated-at |
  | `plugin-sdk/state-paths` | Допоміжні засоби шляхів стану | Допоміжні засоби каталогів стану та OAuth |
  | `plugin-sdk/routing` | Допоміжні засоби маршрутизації/ключів сесій | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, допоміжні засоби нормалізації ключів сесій |
  | `plugin-sdk/status-helpers` | Допоміжні засоби стану каналів | Builder-и зведення стану каналів/облікових записів, значення runtime-state за замовчуванням, допоміжні засоби метаданих проблем |
  | `plugin-sdk/target-resolver-runtime` | Допоміжні засоби визначення цілей | Спільні допоміжні засоби target resolver |
  | `plugin-sdk/string-normalization-runtime` | Допоміжні засоби нормалізації рядків | Допоміжні засоби нормалізації slug/рядків |
  | `plugin-sdk/request-url` | Допоміжні засоби URL запитів | Витяг рядкових URL із вхідних даних, схожих на запит |
  | `plugin-sdk/run-command` | Допоміжні засоби команд із таймером | Виконавець команд із нормалізованими stdout/stderr |
  | `plugin-sdk/param-readers` | Зчитувачі параметрів | Поширені зчитувачі параметрів інструментів/CLI |
  | `plugin-sdk/tool-payload` | Витяг payload інструментів | Витяг нормалізованих payload із об’єктів результатів інструментів |
  | `plugin-sdk/tool-send` | Витяг надсилання інструментів | Витяг канонічних полів цілі надсилання з аргументів інструментів |
  | `plugin-sdk/temp-path` | Допоміжні засоби тимчасових шляхів | Спільні допоміжні засоби шляхів тимчасових завантажень |
  | `plugin-sdk/logging-core` | Допоміжні засоби логування | Логер підсистеми та допоміжні засоби редагування чутливих даних |
  | `plugin-sdk/markdown-table-runtime` | Допоміжні засоби таблиць markdown | Допоміжні засоби режиму таблиць markdown |
  | `plugin-sdk/reply-payload` | Типи відповідей повідомлень | Типи payload відповідей |
  | `plugin-sdk/provider-setup` | Кураторські допоміжні засоби налаштування локальних/self-hosted провайдерів | Допоміжні засоби виявлення/конфігурації self-hosted провайдерів |
  | `plugin-sdk/self-hosted-provider-setup` | Цільові допоміжні засоби налаштування OpenAI-compatible self-hosted провайдерів | Ті самі допоміжні засоби виявлення/конфігурації self-hosted провайдерів |
  | `plugin-sdk/provider-auth-runtime` | Допоміжні засоби runtime auth провайдерів | Допоміжні засоби визначення API-ключів під час виконання |
  | `plugin-sdk/provider-auth-api-key` | Допоміжні засоби налаштування API-ключів провайдера | Допоміжні засоби онбордингу/запису профілю API-ключів |
  | `plugin-sdk/provider-auth-result` | Допоміжні засоби результатів auth провайдера | Стандартний builder auth-result для OAuth |
  | `plugin-sdk/provider-auth-login` | Допоміжні засоби інтерактивного входу провайдера | Спільні допоміжні засоби інтерактивного входу |
  | `plugin-sdk/provider-env-vars` | Допоміжні засоби env vars провайдерів | Допоміжні засоби пошуку env vars auth провайдера |
  | `plugin-sdk/provider-model-shared` | Спільні допоміжні засоби моделей/replay провайдерів | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, спільні builder-и політик replay, допоміжні засоби endpoint провайдера та нормалізації model-id |
  | `plugin-sdk/provider-catalog-shared` | Спільні допоміжні засоби каталогу провайдерів | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Патчі онбордингу провайдерів | Допоміжні засоби конфігурації онбордингу |
  | `plugin-sdk/provider-http` | Допоміжні засоби HTTP провайдерів | Загальні допоміжні засоби HTTP/можливостей endpoint провайдерів |
  | `plugin-sdk/provider-web-fetch` | Допоміжні засоби web-fetch провайдерів | Допоміжні засоби реєстрації/кешу провайдерів web-fetch |
  | `plugin-sdk/provider-web-search-config-contract` | Допоміжні засоби конфігурації веб-пошуку провайдерів | Вузькі допоміжні засоби конфігурації/облікових даних веб-пошуку для провайдерів, яким не потрібна прив’язка ввімкнення плагіна |
  | `plugin-sdk/provider-web-search-contract` | Допоміжні засоби контракту веб-пошуку провайдерів | Вузькі допоміжні засоби контракту конфігурації/облікових даних веб-пошуку, як-от `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` і scoped setter/getter-и облікових даних |
  | `plugin-sdk/provider-web-search` | Допоміжні засоби веб-пошуку провайдерів | Допоміжні засоби реєстрації/кешу/runtime провайдерів веб-пошуку |
  | `plugin-sdk/provider-tools` | Допоміжні засоби сумісності інструментів/схем провайдерів | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, очищення схем Gemini + діагностика, а також допоміжні засоби сумісності xAI, як-от `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Допоміжні засоби використання провайдерів | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` та інші допоміжні засоби використання провайдерів |
  | `plugin-sdk/provider-stream` | Допоміжні засоби обгорток потоків провайдерів | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, типи обгорток потоків і спільні допоміжні засоби обгорток Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Впорядкована async-черга | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Спільні допоміжні засоби медіа | Допоміжні засоби fetch/transform/store медіа, а також builder-и media payload |
  | `plugin-sdk/media-generation-runtime` | Спільні допоміжні засоби генерації медіа | Спільні допоміжні засоби failover, вибору кандидатів і повідомлень про відсутні моделі для генерації зображень/відео/музики |
  | `plugin-sdk/media-understanding` | Допоміжні засоби media-understanding | Типи провайдерів media-understanding, а також provider-facing експорти допоміжних засобів зображень/аудіо |
  | `plugin-sdk/text-runtime` | Спільні допоміжні засоби тексту | Видалення тексту, видимого асистенту, допоміжні засоби render/chunking/table markdown, редагування чутливих даних, допоміжні засоби тегів директив, безпечного тексту та пов’язані допоміжні засоби тексту/логування |
  | `plugin-sdk/text-chunking` | Допоміжні засоби chunking тексту | Допоміжний засіб chunking вихідного тексту |
  | `plugin-sdk/speech` | Допоміжні засоби мовлення | Типи speech-провайдерів, а також provider-facing допоміжні засоби директив, реєстру та валідації |
  | `plugin-sdk/speech-core` | Спільне ядро мовлення | Типи speech-провайдерів, реєстр, директиви, нормалізація |
  | `plugin-sdk/realtime-transcription` | Допоміжні засоби realtime transcription | Типи провайдерів і допоміжні засоби реєстру |
  | `plugin-sdk/realtime-voice` | Допоміжні засоби realtime voice | Типи провайдерів і допоміжні засоби реєстру |
  | `plugin-sdk/image-generation-core` | Спільне ядро генерації зображень | Типи генерації зображень, failover, auth і допоміжні засоби реєстру |
  | `plugin-sdk/music-generation` | Допоміжні засоби генерації музики | Типи провайдерів/запитів/результатів генерації музики |
  | `plugin-sdk/music-generation-core` | Спільне ядро генерації музики | Типи генерації музики, допоміжні засоби failover, пошуку провайдерів і розбору model-ref |
  | `plugin-sdk/video-generation` | Допоміжні засоби генерації відео | Типи провайдерів/запитів/результатів генерації відео |
  | `plugin-sdk/video-generation-core` | Спільне ядро генерації відео | Типи генерації відео, допоміжні засоби failover, пошуку провайдерів і розбору model-ref |
  | `plugin-sdk/interactive-runtime` | Допоміжні засоби інтерактивних відповідей | Нормалізація/зведення payload інтерактивних відповідей |
  | `plugin-sdk/channel-config-primitives` | Примітиви конфігурації каналів | Вузькі примітиви channel config-schema |
  | `plugin-sdk/channel-config-writes` | Допоміжні засоби запису конфігурації каналів | Допоміжні засоби авторизації запису конфігурації каналів |
  | `plugin-sdk/channel-plugin-common` | Спільна преамбула каналів | Експорти спільної преамбули плагінів каналів |
  | `plugin-sdk/channel-status` | Допоміжні засоби стану каналів | Спільні допоміжні засоби snapshot/summary стану каналів |
  | `plugin-sdk/allowlist-config-edit` | Допоміжні засоби конфігурації allowlist | Допоміжні засоби редагування/читання конфігурації allowlist |
  | `plugin-sdk/group-access` | Допоміжні засоби доступу до груп | Спільні допоміжні засоби рішень щодо group-access |
  | `plugin-sdk/direct-dm` | Допоміжні засоби direct-DM | Спільні допоміжні засоби auth/guard для direct-DM |
  | `plugin-sdk/extension-shared` | Спільні допоміжні засоби розширень | Примітиви пасивних каналів/стану та ambient proxy helper |
  | `plugin-sdk/webhook-targets` | Допоміжні засоби цілей webhook | Реєстр цілей webhook і допоміжні засоби встановлення маршрутів |
  | `plugin-sdk/webhook-path` | Допоміжні засоби шляхів webhook | Допоміжні засоби нормалізації шляхів webhook |
  | `plugin-sdk/web-media` | Спільні допоміжні засоби вебмедіа | Допоміжні засоби завантаження віддалених/локальних медіа |
  | `plugin-sdk/zod` | Повторний експорт Zod | Повторно експортований `zod` для споживачів SDK плагінів |
  | `plugin-sdk/memory-core` | Допоміжні засоби вбудованого memory-core | Поверхня допоміжних засобів memory manager/config/file/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Фасад runtime рушія пам’яті | Фасад runtime індексу/пошуку пам’яті |
  | `plugin-sdk/memory-core-host-engine-foundation` | Foundation engine хоста пам’яті | Експорти foundation engine хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Embedding engine хоста пам’яті | Експорти embedding engine хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-qmd` | QMD engine хоста пам’яті | Експорти QMD engine хоста пам’яті |
  | `plugin-sdk/memory-core-host-engine-storage` | Storage engine хоста пам’яті | Експорти storage engine хоста пам’яті |
  | `plugin-sdk/memory-core-host-multimodal` | Допоміжні засоби multimodal хоста пам’яті | Допоміжні засоби multimodal хоста пам’яті |
  | `plugin-sdk/memory-core-host-query` | Допоміжні засоби query хоста пам’яті | Допоміжні засоби query хоста пам’яті |
  | `plugin-sdk/memory-core-host-secret` | Допоміжні засоби secret хоста пам’яті | Допоміжні засоби secret хоста пам’яті |
  | `plugin-sdk/memory-core-host-events` | Допоміжні засоби журналу подій хоста пам’яті | Допоміжні засоби журналу подій хоста пам’яті |
  | `plugin-sdk/memory-core-host-status` | Допоміжні засоби стану хоста пам’яті | Допоміжні засоби стану хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-cli` | CLI runtime хоста пам’яті | Допоміжні засоби CLI runtime хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-core` | Core runtime хоста пам’яті | Допоміжні засоби core runtime хоста пам’яті |
  | `plugin-sdk/memory-core-host-runtime-files` | Допоміжні засоби файлів/runtime хоста пам’яті | Допоміжні засоби файлів/runtime хоста пам’яті |
  | `plugin-sdk/memory-host-core` | Псевдонім core runtime хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів core runtime хоста пам’яті |
  | `plugin-sdk/memory-host-events` | Псевдонім журналу подій хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів журналу подій хоста пам’яті |
  | `plugin-sdk/memory-host-files` | Псевдонім файлів/runtime хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів файлів/runtime хоста пам’яті |
  | `plugin-sdk/memory-host-markdown` | Допоміжні засоби керованого markdown | Спільні допоміжні засоби керованого markdown для суміжних із пам’яттю плагінів |
  | `plugin-sdk/memory-host-search` | Фасад пошуку активної пам’яті | Ледачий фасад runtime менеджера пошуку активної пам’яті |
  | `plugin-sdk/memory-host-status` | Псевдонім стану хоста пам’яті | Вендорно-нейтральний псевдонім для допоміжних засобів стану хоста пам’яті |
  | `plugin-sdk/memory-lancedb` | Допоміжні засоби вбудованого memory-lancedb | Поверхня допоміжних засобів memory-lancedb |
  | `plugin-sdk/testing` | Утиліти тестування | Допоміжні засоби тестування та mock-об’єкти |
</Accordion>

Ця таблиця навмисно охоплює лише поширену підмножину для міграції, а не всю
поверхню SDK. Повний список із понад 200 точок входу міститься у
`scripts/lib/plugin-sdk-entrypoints.json`.

Цей список усе ще містить деякі seams допоміжних засобів вбудованих плагінів,
такі як `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` і `plugin-sdk/matrix*`. Вони й надалі експортуються
для підтримки вбудованих плагінів і сумісності, але навмисно не включені до
таблиці поширеної міграції та не є рекомендованою ціллю для нового коду
плагінів.

Те саме правило стосується й інших сімейств вбудованих допоміжних засобів,
таких як:

- допоміжні засоби підтримки браузера: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- поверхні вбудованих helper/plugin, такі як `plugin-sdk/googlechat`,
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

Використовуйте найвужчий імпорт, який відповідає вашому завданню. Якщо ви не
можете знайти експорт, перегляньте джерело в `src/plugin-sdk/` або запитайте в Discord.

## Хронологія вилучення

| Коли | Що відбувається |
| ---------------------- | ----------------------------------------------------------------------- |
| **Зараз** | Застарілі поверхні надсилають попередження під час виконання |
| **Наступний мажорний реліз** | Застарілі поверхні буде вилучено; плагіни, які все ще їх використовують, перестануть працювати |

Усі основні плагіни вже перенесено. Зовнішні плагіни мають виконати міграцію
до наступного мажорного релізу.

## Тимчасове приглушення попереджень

Установіть ці змінні середовища, поки працюєте над міграцією:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Це тимчасовий обхідний механізм, а не постійне рішення.

## Пов’язане

- [Початок роботи](/uk/plugins/building-plugins) — створіть свій перший плагін
- [Огляд SDK](/uk/plugins/sdk-overview) — повний довідник імпорту підшляхів
- [Плагіни каналів](/uk/plugins/sdk-channel-plugins) — створення плагінів каналів
- [Плагіни провайдерів](/uk/plugins/sdk-provider-plugins) — створення плагінів провайдерів
- [Внутрішня будова плагінів](/uk/plugins/architecture) — глибокий огляд архітектури
- [Маніфест плагіна](/uk/plugins/manifest) — довідник схеми маніфесту
