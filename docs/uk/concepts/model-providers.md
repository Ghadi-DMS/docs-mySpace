---
read_when:
    - Вам потрібен довідник із налаштування моделей для кожного постачальника окремо
    - Ви хочете приклади конфігурацій або команд CLI для онбордингу постачальників моделей
summary: Огляд постачальників моделей із прикладами конфігурацій і потоками CLI
title: Постачальники моделей
x-i18n:
    generated_at: "2026-04-08T03:42:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 558ac9e34b67fcc3dd6791a01bebc17e1c34152fa6c5611593d681e8cfa532d9
    source_path: concepts/model-providers.md
    workflow: 15
---

# Постачальники моделей

На цій сторінці описано **постачальників LLM/моделей** (а не канали чату, як-от WhatsApp/Telegram).
Правила вибору моделей див. у [/concepts/models](/uk/concepts/models).

## Швидкі правила

- Посилання на моделі використовують формат `provider/model` (приклад: `opencode/claude-opus-4-6`).
- Якщо ви задаєте `agents.defaults.models`, це стає списком дозволених моделей.
- Допоміжні команди CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Резервні правила часу виконання, cooldown-перевірки та збереження перевизначень сеансу
  задокументовано в [/concepts/model-failover](/uk/concepts/model-failover).
- `models.providers.*.models[].contextWindow` — це власні метадані моделі;
  `models.providers.*.models[].contextTokens` — це фактичне обмеження під час виконання.
- Плагіни постачальників можуть впроваджувати каталоги моделей через `registerProvider({ catalog })`;
  OpenClaw об’єднує цей результат у `models.providers` перед записом
  `models.json`.
- Маніфести постачальників можуть оголошувати `providerAuthEnvVars`, щоб загальні перевірки
  автентифікації на основі змінних середовища не потребували завантаження часу виконання плагіна. Решта базової мапи змінних середовища
  тепер використовується лише для неплагінових/базових постачальників і кількох випадків
  загального пріоритету, як-от онбординг Anthropic із пріоритетом API-ключа.
- Плагіни постачальників також можуть володіти поведінкою часу виконання постачальника через
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, and
  `onModelSelected`.
- Примітка: `capabilities` часу виконання постачальника — це спільні метадані раннера (сімейство постачальника,
  особливості стенограми/інструментів, підказки щодо транспорту/кешу). Це не те
  саме, що [публічна модель можливостей](/uk/plugins/architecture#public-capability-model),
  яка описує, що реєструє плагін (текстове інференсування, мовлення тощо).

## Поведінка постачальника, що належить плагіну

Плагіни постачальників тепер можуть володіти більшістю специфічної для постачальника логіки, тоді як OpenClaw зберігає
загальний цикл інференсування.

Типовий розподіл:

- `auth[].run` / `auth[].runNonInteractive`: постачальник володіє потоками онбордингу/входу
  для `openclaw onboard`, `openclaw models auth` і безголового налаштування
- `wizard.setup` / `wizard.modelPicker`: постачальник володіє підписами вибору автентифікації,
  застарілими псевдонімами, підказками списку дозволених моделей під час онбордингу та записами налаштування в засобах вибору онбордингу/моделей
- `catalog`: постачальник з’являється в `models.providers`
- `normalizeModelId`: постачальник нормалізує застарілі/preview-ідентифікатори моделей перед
  пошуком або канонізацією
- `normalizeTransport`: постачальник нормалізує `api` / `baseUrl` сімейства транспорту
  перед загальним складанням моделі; OpenClaw спочатку перевіряє відповідного постачальника,
  потім інші плагіни постачальників із підтримкою хуків, доки один із них справді не змінить
  транспорт
- `normalizeConfig`: постачальник нормалізує конфігурацію `models.providers.<id>` перед
  її використанням під час виконання; OpenClaw спочатку перевіряє відповідного постачальника, потім інші
  плагіни постачальників із підтримкою хуків, доки один із них справді не змінить конфігурацію. Якщо жоден
  хук постачальника не переписує конфігурацію, вбудовані допоміжні засоби сімейства Google
  усе ще нормалізують підтримувані записи постачальників Google.
- `applyNativeStreamingUsageCompat`: постачальник застосовує переписування сумісності власного потокового використання на основі кінцевої точки для конфігурованих постачальників
- `resolveConfigApiKey`: постачальник розв’язує автентифікацію за маркером змінної середовища для конфігурованих постачальників
  без примусового повного завантаження автентифікації часу виконання. `amazon-bedrock` тут також має
  вбудований резолвер маркерів середовища AWS, хоча автентифікація часу виконання Bedrock використовує
  типовий ланцюжок AWS SDK.
- `resolveSyntheticAuth`: постачальник може надавати доступність локальної/self-hosted чи іншої
  автентифікації на основі конфігурації без збереження незашифрованих секретів
- `shouldDeferSyntheticProfileAuth`: постачальник може позначати збережені синтетичні профілі-заповнювачі
  як менш пріоритетні, ніж автентифікація на основі env/config
- `resolveDynamicModel`: постачальник приймає ідентифікатори моделей, яких ще немає в локальному
  статичному каталозі
- `prepareDynamicModel`: постачальнику потрібне оновлення метаданих перед повторною спробою
  динамічного розв’язання
- `normalizeResolvedModel`: постачальнику потрібні переписування транспорту або base URL
- `contributeResolvedModelCompat`: постачальник додає прапорці сумісності для своїх
  вендорних моделей навіть тоді, коли вони надходять через інший сумісний транспорт
- `capabilities`: постачальник публікує особливості стенограми/інструментів/сімейства постачальника
- `normalizeToolSchemas`: постачальник очищає схеми інструментів перед тим, як їх побачить
  вбудований раннер
- `inspectToolSchemas`: постачальник показує попередження про схему, специфічні для транспорту,
  після нормалізації
- `resolveReasoningOutputMode`: постачальник вибирає власні чи теговані
  контракти виводу міркувань
- `prepareExtraParams`: постачальник задає типові значення або нормалізує параметри запиту для кожної моделі
- `createStreamFn`: постачальник замінює звичайний шлях потоку повністю
  власним транспортом
- `wrapStreamFn`: постачальник застосовує обгортки сумісності заголовків/тіла/моделі запиту
- `resolveTransportTurnState`: постачальник надає власні заголовки або метадані транспорту
  для кожного ходу
- `resolveWebSocketSessionPolicy`: постачальник надає власні заголовки сеансу WebSocket
  або політику cooldown сеансу
- `createEmbeddingProvider`: постачальник володіє поведінкою embedding для пам’яті, коли вона
  належить плагіну постачальника, а не базовому комутатору embedding
- `formatApiKey`: постачальник форматує збережені профілі автентифікації у
  рядок `apiKey`, який очікує транспорт під час виконання
- `refreshOAuth`: постачальник володіє оновленням OAuth, коли спільних
  засобів оновлення `pi-ai` недостатньо
- `buildAuthDoctorHint`: постачальник додає вказівки з відновлення, коли оновлення OAuth
  не вдається
- `matchesContextOverflowError`: постачальник розпізнає специфічні для постачальника
  помилки переповнення контекстного вікна, які пропустили б загальні евристики
- `classifyFailoverReason`: постачальник мапить специфічні для постачальника сирі помилки транспорту/API
  на причини перемикання, як-от rate limit або перевантаження
- `isCacheTtlEligible`: постачальник визначає, які ідентифікатори моделей вищого рівня підтримують TTL кешу промптів
- `buildMissingAuthMessage`: постачальник замінює загальну помилку сховища автентифікації
  на специфічну для постачальника підказку з відновлення
- `suppressBuiltInModel`: постачальник приховує застарілі рядки вищого рівня і може повернути
  помилку, що належить вендору, для прямих помилок розв’язання
- `augmentModelCatalog`: постачальник додає синтетичні/фінальні рядки каталогу після
  виявлення та злиття конфігурації
- `isBinaryThinking`: постачальник володіє UX двійкового ввімкнення/вимкнення thinking
- `supportsXHighThinking`: постачальник вмикає `xhigh` для вибраних моделей
- `resolveDefaultThinkingLevel`: постачальник володіє типовою політикою `/think` для
  сімейства моделей
- `applyConfigDefaults`: постачальник застосовує специфічні глобальні типові значення постачальника
  під час матеріалізації конфігурації на основі режиму автентифікації, env або сімейства моделей
- `isModernModelRef`: постачальник володіє зіставленням бажаної live/smoke-моделі
- `prepareRuntimeAuth`: постачальник перетворює налаштовані облікові дані на короткоживучий
  токен часу виконання
- `resolveUsageAuth`: постачальник розв’язує облікові дані використання/квоти для `/usage`
  і пов’язаних поверхонь статусу/звітності
- `fetchUsageSnapshot`: постачальник володіє отриманням/розбором кінцевої точки використання, тоді як
  базова система все ще володіє оболонкою підсумку та форматуванням
- `onModelSelected`: постачальник виконує побічні ефекти після вибору, як-от
  телеметрія або ведення обліку сеансу, що належить постачальнику

Поточні вбудовані приклади:

- `anthropic`: резервна пряма сумісність для Claude 4.6, підказки з відновлення автентифікації, отримання
  кінцевої точки використання, метадані TTL кешу/сімейства постачальника та глобальні
  типові значення конфігурації з урахуванням автентифікації
- `amazon-bedrock`: зіставлення переповнення контексту та класифікація причин
  перемикання, що належать постачальнику, для специфічних помилок Bedrock щодо throttle/not-ready, а також
  спільне сімейство повторного відтворення `anthropic-by-model` для політики replay лише Claude
  на трафіку Anthropic
- `anthropic-vertex`: обмеження політики replay лише Claude для Anthropic-message
  трафіку
- `openrouter`: наскрізні ідентифікатори моделей, обгортки запитів, підказки
  щодо можливостей постачальника, санітизація thought-signature Gemini на проксійованому трафіку Gemini,
  проксійнe впровадження reasoning через сімейство потоків `openrouter-thinking`, переадресація
  метаданих маршрутизації та політика TTL кешу
- `github-copilot`: онбординг/вхід із пристрою, резервне суміщення моделей для майбутньої сумісності,
  підказки стенограми Claude-thinking, обмін токенів часу виконання та отримання
  кінцевої точки використання
- `openai`: резервна пряма сумісність для GPT-5.4, нормалізація прямого транспорту
  OpenAI, підказки про відсутню автентифікацію з урахуванням Codex, приглушення Spark, синтетичні
  рядки каталогу OpenAI/Codex, політика thinking/live-моделей, нормалізація псевдонімів токенів використання (`input` / `output` і сімейства `prompt` / `completion`), спільне
  сімейство потоків `openai-responses-defaults` для власних обгорток OpenAI/Codex,
  метадані сімейства постачальника, реєстрація вбудованого постачальника генерації зображень
  для `gpt-image-1` та реєстрація вбудованого постачальника генерації відео
  для `sora-2`
- `google` і `google-gemini-cli`: резервна пряма сумісність для Gemini 3.1,
  власна перевірка replay Gemini, санітизація bootstrap replay, тегований
  режим виводу reasoning, зіставлення сучасних моделей, реєстрація вбудованого постачальника генерації зображень для моделей Gemini image-preview і вбудована
  реєстрація постачальника генерації відео для моделей Veo; OAuth Gemini CLI також
  володіє форматуванням токенів профілю автентифікації, розбором токенів використання та отриманням
  кінцевої точки квот для поверхонь використання
- `moonshot`: спільний транспорт, нормалізація корисного навантаження thinking, що належить плагіну
- `kilocode`: спільний транспорт, заголовки запиту, що належать плагіну, нормалізація
  корисного навантаження reasoning, санітизація thought-signature проксійованого Gemini та політика TTL кешу
- `zai`: резервна пряма сумісність для GLM-5, типові значення `tool_stream`, TTL кешу,
  політика binary-thinking/live-моделей, а також автентифікація використання + отримання квот;
  невідомі ідентифікатори `glm-5*` синтезуються з вбудованого шаблону `glm-4.7`
- `xai`: нормалізація власного транспорту Responses, переписування псевдонімів `/fast` для
  швидких варіантів Grok, типовий `tool_stream`, очищення схем інструментів /
  корисного навантаження reasoning, специфічне для xAI, та вбудована реєстрація постачальника генерації відео
  для `grok-imagine-video`
- `mistral`: метадані можливостей, що належать плагіну
- `opencode` і `opencode-go`: метадані можливостей, що належать плагіну, плюс
  санітизація thought-signature проксійованого Gemini
- `alibaba`: каталог генерації відео, що належить плагіну, для прямих посилань на моделі Wan,
  таких як `alibaba/wan2.6-t2v`
- `byteplus`: каталоги, що належать плагіну, плюс вбудована реєстрація постачальника генерації відео
  для моделей Seedance text-to-video/image-to-video
- `fal`: вбудована реєстрація постачальника генерації відео для розміщеної сторонньої
  реєстрації постачальника генерації зображень для моделей зображень FLUX, а також вбудована
  реєстрація постачальника генерації відео для розміщених сторонніх відеомоделей
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` і `volcengine`:
  лише каталоги, що належать плагіну
- `qwen`: каталоги, що належать плагіну, для текстових моделей плюс спільні
  реєстрації постачальників media-understanding і генерації відео для його
  мультимодальних поверхонь; генерація відео Qwen використовує стандартні кінцеві точки DashScope video з вбудованими моделями Wan, такими як `wan2.6-t2v` і `wan2.7-r2v`
- `runway`: реєстрація постачальника генерації відео, що належить плагіну, для власних
  моделей на основі задач Runway, таких як `gen4.5`
- `minimax`: каталоги, що належать плагіну, вбудована реєстрація постачальника генерації відео
  для моделей відео Hailuo, вбудована реєстрація постачальника генерації зображень
  для `image-01`, гібридний вибір політики replay Anthropic/OpenAI
  та логіка автентифікації/знімка використання
- `together`: каталоги, що належать плагіну, плюс вбудована реєстрація постачальника генерації відео
  для відеомоделей Wan
- `xiaomi`: каталоги, що належать плагіну, плюс логіка автентифікації/знімка використання

Вбудований плагін `openai` тепер володіє обома ідентифікаторами постачальників: `openai` і
`openai-codex`.

Це охоплює постачальників, які все ще вписуються у звичайні транспорти OpenClaw. Постачальник,
якому потрібен повністю власний виконавець запитів, — це окрема, глибша поверхня розширення.

## Ротація API-ключів

- Підтримує загальну ротацію постачальників для вибраних постачальників.
- Налаштуйте кілька ключів через:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (єдиний live-override, найвищий пріоритет)
  - `<PROVIDER>_API_KEYS` (список через кому або крапку з комою)
  - `<PROVIDER>_API_KEY` (основний ключ)
  - `<PROVIDER>_API_KEY_*` (нумерований список, наприклад `<PROVIDER>_API_KEY_1`)
- Для постачальників Google як резерв також додається `GOOGLE_API_KEY`.
- Порядок вибору ключів зберігає пріоритет і видаляє дублікати значень.
- Запити повторюються з наступним ключем лише у відповідь на rate-limit (наприклад,
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` або періодичні повідомлення про ліміт використання).
- Помилки, не пов’язані з rate-limit, завершуються негайно; ротація ключів не виконується.
- Коли всі кандидатні ключі не спрацьовують, повертається фінальна помилка з останньої спроби.

## Вбудовані постачальники (каталог pi-ai)

OpenClaw постачається з каталогом pi‑ai. Ці постачальники не потребують
конфігурації `models.providers`; достатньо задати автентифікацію й вибрати модель.

### OpenAI

- Постачальник: `openai`
- Автентифікація: `OPENAI_API_KEY`
- Необов’язкова ротація: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, а також `OPENCLAW_LIVE_OPENAI_KEY` (єдине перевизначення)
- Приклади моделей: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Типовий транспорт — `auto` (спочатку WebSocket, резервно SSE)
- Перевизначення для окремої моделі через `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` або `"auto"`)
- Підігрів WebSocket OpenAI Responses типово ввімкнено через `params.openaiWsWarmup` (`true`/`false`)
- Пріоритетну обробку OpenAI можна ввімкнути через `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` і `params.fastMode` маплять прямі запити OpenAI `openai/*` Responses на `service_tier=priority` у `api.openai.com`
- Використовуйте `params.serviceTier`, якщо вам потрібен явний рівень замість спільного перемикача `/fast`
- Приховані заголовки атрибуції OpenClaw (`originator`, `version`,
  `User-Agent`) застосовуються лише до власного трафіку OpenAI на `api.openai.com`, а не до
  загальних OpenAI-сумісних проксі
- Власні маршрути OpenAI також зберігають `store` Responses, підказки кешу промптів і
  формування корисного навантаження для сумісності reasoning OpenAI; проксі-маршрути — ні
- `openai/gpt-5.3-codex-spark` навмисно приглушено в OpenClaw, оскільки жива API OpenAI його відхиляє; Spark розглядається лише як Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Постачальник: `anthropic`
- Автентифікація: `ANTHROPIC_API_KEY`
- Необов’язкова ротація: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, а також `OPENCLAW_LIVE_ANTHROPIC_KEY` (єдине перевизначення)
- Приклад моделі: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Прямі публічні запити Anthropic підтримують спільний перемикач `/fast` і `params.fastMode`, включно з трафіком, автентифікованим API-ключем та OAuth, надісланим до `api.anthropic.com`; OpenClaw мапить це на Anthropic `service_tier` (`auto` проти `standard_only`)
- Примітка щодо Anthropic: співробітники Anthropic повідомили нам, що використання Claude CLI у стилі OpenClaw знову дозволене, тож OpenClaw вважає повторне використання Claude CLI і `claude -p` санкціонованими для цієї інтеграції, якщо Anthropic не опублікує нову політику.
- Токен налаштування Anthropic і далі доступний як підтримуваний шлях токена OpenClaw, але тепер OpenClaw надає перевагу повторному використанню Claude CLI і `claude -p`, коли це доступно.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Постачальник: `openai-codex`
- Автентифікація: OAuth (ChatGPT)
- Приклад моделі: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` або `openclaw models auth login --provider openai-codex`
- Типовий транспорт — `auto` (спочатку WebSocket, резервно SSE)
- Перевизначення для окремої моделі через `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` або `"auto"`)
- `params.serviceTier` також передається у власних запитах Codex Responses (`chatgpt.com/backend-api`)
- Приховані заголовки атрибуції OpenClaw (`originator`, `version`,
  `User-Agent`) додаються лише до власного трафіку Codex на
  `chatgpt.com/backend-api`, а не до загальних OpenAI-сумісних проксі
- Використовує той самий перемикач `/fast` і конфігурацію `params.fastMode`, що й прямий `openai/*`; OpenClaw мапить це на `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` і далі доступний, коли каталог OAuth Codex його показує; залежить від прав доступу
- `openai-codex/gpt-5.4` зберігає власний `contextWindow = 1050000` і типове обмеження часу виконання `contextTokens = 272000`; перевизначте ліміт часу виконання через `models.providers.openai-codex.models[].contextTokens`
- Примітка щодо політики: OpenAI Codex OAuth офіційно підтримується для зовнішніх інструментів/робочих процесів, таких як OpenClaw.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Інші розміщені варіанти у стилі підписки

- [Qwen Cloud](/uk/providers/qwen): поверхня постачальника Qwen Cloud, а також мапінг кінцевих точок Alibaba DashScope і Coding Plan
- [MiniMax](/uk/providers/minimax): доступ MiniMax Coding Plan через OAuth або API-ключ
- [GLM Models](/uk/providers/glm): кінцеві точки Z.AI Coding Plan або загального API

### OpenCode

- Автентифікація: `OPENCODE_API_KEY` (або `OPENCODE_ZEN_API_KEY`)
- Постачальник часу виконання Zen: `opencode`
- Постачальник часу виконання Go: `opencode-go`
- Приклади моделей: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` або `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API-ключ)

- Постачальник: `google`
- Автентифікація: `GEMINI_API_KEY`
- Необов’язкова ротація: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, резервний `GOOGLE_API_KEY` і `OPENCLAW_LIVE_GEMINI_KEY` (єдине перевизначення)
- Приклади моделей: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Сумісність: застаріла конфігурація OpenClaw з `google/gemini-3.1-flash-preview` нормалізується до `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Прямі запуски Gemini також приймають `agents.defaults.models["google/<model>"].params.cachedContent`
  (або застарілий `cached_content`) для передавання власного
  дескриптора `cachedContents/...`; попадання в кеш Gemini відображаються як OpenClaw `cacheRead`

### Google Vertex і Gemini CLI

- Постачальники: `google-vertex`, `google-gemini-cli`
- Автентифікація: Vertex використовує gcloud ADC; Gemini CLI використовує власний потік OAuth
- Увага: OAuth Gemini CLI в OpenClaw — неофіційна інтеграція. Деякі користувачі повідомляли про обмеження облікового запису Google після використання сторонніх клієнтів. Ознайомтеся з умовами Google і використовуйте некритичний обліковий запис, якщо вирішите продовжити.
- OAuth Gemini CLI постачається як частина вбудованого плагіна `google`.
  - Спочатку встановіть Gemini CLI:
    - `brew install gemini-cli`
    - або `npm install -g @google/gemini-cli`
  - Увімкнення: `openclaw plugins enable google`
  - Вхід: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Типова модель: `google-gemini-cli/gemini-3-flash-preview`
  - Примітка: **не потрібно** вставляти client id або secret у `openclaw.json`. Потік входу CLI зберігає
    токени в профілях автентифікації на хості шлюзу.
  - Якщо запити не працюють після входу, задайте `GOOGLE_CLOUD_PROJECT` або `GOOGLE_CLOUD_PROJECT_ID` на хості шлюзу.
  - JSON-відповіді Gemini CLI розбираються з `response`; використання резервно береться з
    `stats`, а `stats.cached` нормалізується в OpenClaw `cacheRead`.

### Z.AI (GLM)

- Постачальник: `zai`
- Автентифікація: `ZAI_API_KEY`
- Приклад моделі: `zai/glm-5.1`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Псевдоніми: `z.ai/*` і `z-ai/*` нормалізуються до `zai/*`
  - `zai-api-key` автоматично визначає відповідну кінцеву точку Z.AI; `zai-coding-global`, `zai-coding-cn`, `zai-global` і `zai-cn` примусово задають конкретну поверхню

### Vercel AI Gateway

- Постачальник: `vercel-ai-gateway`
- Автентифікація: `AI_GATEWAY_API_KEY`
- Приклад моделі: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Постачальник: `kilocode`
- Автентифікація: `KILOCODE_API_KEY`
- Приклад моделі: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Base URL: `https://api.kilo.ai/api/gateway/`
- Статичний резервний каталог постачається з `kilocode/kilo/auto`; живе
  виявлення `https://api.kilo.ai/api/gateway/models` може додатково розширити
  каталог часу виконання.
- Точна маршрутизація на верхньому рівні за `kilocode/kilo/auto` належить Kilo Gateway,
  а не жорстко закодована в OpenClaw.

Докладніше про налаштування див. у [/providers/kilocode](/uk/providers/kilocode).

### Інші вбудовані плагіни постачальників

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Приклад моделі: `openrouter/auto`
- OpenClaw застосовує документовані заголовки атрибуції застосунку OpenRouter лише тоді, коли
  запит справді націлений на `openrouter.ai`
- Маркери Anthropic `cache_control`, специфічні для OpenRouter, так само обмежені
  перевіреними маршрутами OpenRouter, а не довільними URL проксі
- OpenRouter залишається на проксійному OpenAI-сумісному шляху, тому власне
  лише-OpenAI формування запитів (`serviceTier`, `store` Responses,
  підказки кешу промптів, payload для сумісності reasoning OpenAI) не передається
- Посилання OpenRouter на базі Gemini зберігають лише санітизацію thought-signature проксійованого Gemini;
  власна перевірка replay Gemini та переписування bootstrap залишаються вимкненими
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Приклад моделі: `kilocode/kilo/auto`
- Посилання Kilo на базі Gemini зберігають той самий шлях санітизації thought-signature
  проксійованого Gemini; `kilocode/kilo/auto` та інші підказки, що не підтримують proxy-reasoning,
  пропускають упровадження proxy reasoning
- MiniMax: `minimax` (API-ключ) і `minimax-portal` (OAuth)
- Автентифікація: `MINIMAX_API_KEY` для `minimax`; `MINIMAX_OAUTH_TOKEN` або `MINIMAX_API_KEY` для `minimax-portal`
- Приклад моделі: `minimax/MiniMax-M2.7` або `minimax-portal/MiniMax-M2.7`
- Онбординг MiniMax/налаштування API-ключа записує явні визначення моделей M2.7 з
  `input: ["text", "image"]`; вбудований каталог постачальника зберігає посилання чату
  лише текстовими, доки конфігурація цього постачальника не буде матеріалізована
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Приклад моделі: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` або `KIMICODE_API_KEY`)
- Приклад моделі: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Приклад моделі: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` або `DASHSCOPE_API_KEY`)
- Приклад моделі: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Приклад моделі: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Приклади моделей: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Приклад моделі: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Приклад моделі: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` або `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Приклад моделі: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Приклад моделі: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Власні вбудовані запити xAI використовують шлях xAI Responses
  - `/fast` або `params.fastMode: true` переписує `grok-3`, `grok-3-mini`,
    `grok-4` і `grok-4-0709` на їхні варіанти `*-fast`
  - `tool_stream` типово ввімкнено; задайте
    `agents.defaults.models["xai/<model>"].params.tool_stream` у `false`, щоб
    вимкнути його
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Приклад моделі: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - Моделі GLM на Cerebras використовують ідентифікатори `zai-glm-4.7` і `zai-glm-4.6`.
  - OpenAI-сумісний base URL: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Приклад моделі Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Див. [Hugging Face (Inference)](/uk/providers/huggingface).

## Постачальники через `models.providers` (custom/base URL)

Використовуйте `models.providers` (або `models.json`), щоб додати **власних** постачальників або
OpenAI/Anthropic‑сумісні проксі.

Багато з наведених нижче вбудованих плагінів постачальників уже публікують типовий каталог.
Використовуйте явні записи `models.providers.<id>` лише тоді, коли хочете перевизначити
типовий base URL, заголовки або список моделей.

### Moonshot AI (Kimi)

Moonshot постачається як вбудований плагін постачальника. Використовуйте вбудованого постачальника
типово й додавайте явний запис `models.providers.moonshot` лише тоді, коли
потрібно перевизначити base URL або метадані моделі:

- Постачальник: `moonshot`
- Автентифікація: `MOONSHOT_API_KEY`
- Приклад моделі: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` або `openclaw onboard --auth-choice moonshot-api-key-cn`

Ідентифікатори моделей Kimi K2:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding використовує Anthropic-сумісну кінцеву точку Moonshot AI:

- Постачальник: `kimi`
- Автентифікація: `KIMI_API_KEY`
- Приклад моделі: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

Застарілий `kimi/k2p5` і далі приймається як сумісний ідентифікатор моделі.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) надає доступ до Doubao та інших моделей у Китаї.

- Постачальник: `volcengine` (coding: `volcengine-plan`)
- Автентифікація: `VOLCANO_ENGINE_API_KEY`
- Приклад моделі: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

Під час онбордингу типово вибирається coding-поверхня, але загальний каталог `volcengine/*`
реєструється одночасно.

У засобах вибору моделей під час онбордингу/налаштування вибір автентифікації Volcengine надає перевагу і рядкам
`volcengine/*`, і `volcengine-plan/*`. Якщо ці моделі ще не завантажено,
OpenClaw повертається до нефільтрованого каталогу замість показу порожнього
засобу вибору в межах постачальника.

Доступні моделі:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Coding-моделі (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (International)

BytePlus ARK надає міжнародним користувачам доступ до тих самих моделей, що й Volcano Engine.

- Постачальник: `byteplus` (coding: `byteplus-plan`)
- Автентифікація: `BYTEPLUS_API_KEY`
- Приклад моделі: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

Під час онбордингу типово вибирається coding-поверхня, але загальний каталог `byteplus/*`
реєструється одночасно.

У засобах вибору моделей під час онбордингу/налаштування вибір автентифікації BytePlus надає перевагу і рядкам
`byteplus/*`, і `byteplus-plan/*`. Якщо ці моделі ще не завантажено,
OpenClaw повертається до нефільтрованого каталогу замість показу порожнього
засобу вибору в межах постачальника.

Доступні моделі:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Coding-моделі (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic надає Anthropic-сумісні моделі за постачальником `synthetic`:

- Постачальник: `synthetic`
- Автентифікація: `SYNTHETIC_API_KEY`
- Приклад моделі: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax налаштовується через `models.providers`, оскільки використовує власні кінцеві точки:

- MiniMax OAuth (Global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax API key (Global): `--auth-choice minimax-global-api`
- MiniMax API key (CN): `--auth-choice minimax-cn-api`
- Автентифікація: `MINIMAX_API_KEY` для `minimax`; `MINIMAX_OAUTH_TOKEN` або
  `MINIMAX_API_KEY` для `minimax-portal`

Докладніше про налаштування, варіанти моделей і фрагменти конфігурації див. у [/providers/minimax](/uk/providers/minimax).

На Anthropic-сумісному потоковому шляху MiniMax OpenClaw типово вимикає thinking,
якщо ви явно його не задасте, а `/fast on` переписує
`MiniMax-M2.7` на `MiniMax-M2.7-highspeed`.

Розподіл можливостей, що належать плагіну:

- Типові значення тексту/чату залишаються на `minimax/MiniMax-M2.7`
- Генерація зображень — це `minimax/image-01` або `minimax-portal/image-01`
- Розуміння зображень — це належний плагіну `MiniMax-VL-01` на обох шляхах автентифікації MiniMax
- Вебпошук залишається на ідентифікаторі постачальника `minimax`

### Ollama

Ollama постачається як вбудований плагін постачальника й використовує власне API Ollama:

- Постачальник: `ollama`
- Автентифікація: не потрібна (локальний сервер)
- Приклад моделі: `ollama/llama3.3`
- Встановлення: [https://ollama.com/download](https://ollama.com/download)

```bash
# Install Ollama, then pull a model:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama виявляється локально за адресою `http://127.0.0.1:11434`, коли ви вмикаєте це через
`OLLAMA_API_KEY`, а вбудований плагін постачальника додає Ollama безпосередньо до
`openclaw onboard` і засобу вибору моделей. Докладніше про онбординг, режими cloud/local і користувацьку конфігурацію див. у [/providers/ollama](/uk/providers/ollama).

### vLLM

vLLM постачається як вбудований плагін постачальника для локальних/self-hosted OpenAI-сумісних
серверів:

- Постачальник: `vllm`
- Автентифікація: необов’язкова (залежить від вашого сервера)
- Типовий base URL: `http://127.0.0.1:8000/v1`

Щоб локально ввімкнути автовиявлення (підійде будь-яке значення, якщо ваш сервер не вимагає автентифікації):

```bash
export VLLM_API_KEY="vllm-local"
```

Потім задайте модель (замініть на один з ідентифікаторів, що повертає `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Докладніше див. у [/providers/vllm](/uk/providers/vllm).

### SGLang

SGLang постачається як вбудований плагін постачальника для швидких self-hosted
OpenAI-сумісних серверів:

- Постачальник: `sglang`
- Автентифікація: необов’язкова (залежить від вашого сервера)
- Типовий base URL: `http://127.0.0.1:30000/v1`

Щоб локально ввімкнути автовиявлення (підійде будь-яке значення, якщо ваш сервер не
вимагає автентифікації):

```bash
export SGLANG_API_KEY="sglang-local"
```

Потім задайте модель (замініть на один з ідентифікаторів, що повертає `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Докладніше див. у [/providers/sglang](/uk/providers/sglang).

### Локальні проксі (LM Studio, vLLM, LiteLLM тощо)

Приклад (OpenAI‑сумісний):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Примітки:

- Для власних постачальників `reasoning`, `input`, `cost`, `contextWindow` і `maxTokens` є необов’язковими.
  Якщо їх не задано, OpenClaw використовує такі типові значення:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Рекомендація: задавайте явні значення, що відповідають лімітам вашого проксі/моделі.
- Для `api: "openai-completions"` на невласних кінцевих точках (будь-який непорожній `baseUrl`, чий хост не є `api.openai.com`) OpenClaw примусово задає `compat.supportsDeveloperRole: false`, щоб уникнути помилок постачальника 400 через непідтримувані ролі `developer`.
- Проксійні OpenAI-сумісні маршрути також пропускають власне лише-OpenAI формування запитів:
  без `service_tier`, без `store` Responses, без підказок кешу промптів, без
  формування payload для сумісності reasoning OpenAI і без прихованих заголовків атрибуції OpenClaw.
- Якщо `baseUrl` порожній/не вказаний, OpenClaw зберігає типову поведінку OpenAI (яка веде до `api.openai.com`).
- Для безпеки явне `compat.supportsDeveloperRole: true` усе одно перевизначається на невласних кінцевих точках `openai-completions`.

## Приклади CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Див. також: [/gateway/configuration](/uk/gateway/configuration) для повних прикладів конфігурації.

## Пов’язане

- [Models](/uk/concepts/models) — конфігурація моделей і псевдоніми
- [Model Failover](/uk/concepts/model-failover) — ланцюжки резервного перемикання та поведінка повторних спроб
- [Configuration Reference](/uk/gateway/configuration-reference#agent-defaults) — ключі конфігурації моделей
- [Providers](/uk/providers) — інструкції з налаштування для кожного постачальника
