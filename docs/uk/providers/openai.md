---
read_when:
    - Ви хочете використовувати моделі OpenAI в OpenClaw
    - Ви хочете використовувати автентифікацію через підписку Codex замість API-ключів
    - Вам потрібні суворіші правила виконання для агента GPT-5
summary: Використовуйте OpenAI через API-ключі або підписку Codex в OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-11T16:15:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7aa06fba9ac901e663685a6b26443a2f6aeb6ec3589d939522dc87cbb43497b4
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI надає API для розробників для моделей GPT. Codex підтримує **вхід через ChatGPT** для доступу за підпискою
або **вхід через API-ключ** для доступу з оплатою за використання. Codex cloud вимагає входу через ChatGPT.
OpenAI явно підтримує використання підписки через OAuth у зовнішніх інструментах і робочих процесах, таких як OpenClaw.

## Стиль взаємодії за замовчуванням

OpenClaw може додавати невелике специфічне для OpenAI накладання промпта як для запусків `openai/*`, так і для
`openai-codex/*`. За замовчуванням це накладання робить асистента теплим,
співпрацюючим, лаконічним, прямим і трохи більш емоційно виразним,
не замінюючи базовий системний промпт OpenClaw. Дружнє накладання також
дозволяє іноді використовувати емодзі, коли це природно доречно, водночас зберігаючи загальний
вивід лаконічним.

Ключ конфігурації:

`plugins.entries.openai.config.personality`

Дозволені значення:

- `"friendly"`: значення за замовчуванням; увімкнути специфічне для OpenAI накладання.
- `"on"`: псевдонім для `"friendly"`.
- `"off"`: вимкнути накладання і використовувати лише базовий промпт OpenClaw.

Область дії:

- Застосовується до моделей `openai/*`.
- Застосовується до моделей `openai-codex/*`.
- Не впливає на інших провайдерів.

Ця поведінка увімкнена за замовчуванням. Залиште `"friendly"` явно, якщо хочете, щоб
це збереглося попри майбутні локальні зміни конфігурації:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### Вимкнення накладання промпта OpenAI

Якщо ви хочете використовувати немодифікований базовий промпт OpenClaw, установіть для накладання значення `"off"`:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Ви також можете встановити це безпосередньо через CLI конфігурації:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw нормалізує це налаштування без урахування регістру під час виконання, тож значення на кшталт
`"Off"` також вимикають дружнє накладання.

## Варіант A: API-ключ OpenAI (OpenAI Platform)

**Найкраще для:** прямого доступу до API та білінгу за використанням.
Отримайте свій API-ключ на панелі керування OpenAI.

Підсумок маршрутів:

- `openai/gpt-5.4` = прямий маршрут через API OpenAI Platform
- Потрібен `OPENAI_API_KEY` (або еквівалентна конфігурація провайдера OpenAI)
- В OpenClaw вхід через ChatGPT/Codex спрямовується через `openai-codex/*`, а не `openai/*`

### Налаштування CLI

```bash
openclaw onboard --auth-choice openai-api-key
# або неінтерактивно
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Фрагмент конфігурації

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

У поточній документації моделей API OpenAI для прямого
використання API OpenAI перелічено `gpt-5.4` і `gpt-5.4-pro`. OpenClaw переспрямовує обидві через шлях `openai/*` Responses.
OpenClaw навмисно приховує застарілий рядок `openai/gpt-5.3-codex-spark`,
оскільки прямі виклики API OpenAI відхиляють його в реальному трафіку.

OpenClaw **не** показує `openai/gpt-5.3-codex-spark` на прямому маршруті
API OpenAI. `pi-ai` усе ще постачає вбудований рядок для цієї моделі, але реальні запити до API OpenAI
зараз її відхиляють. В OpenClaw Spark вважається доступним лише для Codex.

## Генерація зображень

Вбудований plugin `openai` також реєструє генерацію зображень через спільний
інструмент `image_generate`.

- Модель зображень за замовчуванням: `openai/gpt-image-1`
- Генерація: до 4 зображень за один запит
- Режим редагування: увімкнено, до 5 еталонних зображень
- Підтримує `size`
- Поточне специфічне для OpenAI обмеження: сьогодні OpenClaw не передає перевизначення `aspectRatio` або
  `resolution` до OpenAI Images API

Щоб використовувати OpenAI як провайдера зображень за замовчуванням:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Див. [Генерація зображень](/uk/tools/image-generation), щоб ознайомитися зі спільними
параметрами інструмента, вибором провайдера та поведінкою перемикання при збоях.

## Генерація відео

Вбудований plugin `openai` також реєструє генерацію відео через спільний
інструмент `video_generate`.

- Модель відео за замовчуванням: `openai/sora-2`
- Режими: text-to-video, image-to-video та сценарії single-video reference/edit
- Поточні обмеження: 1 еталонне зображення або 1 еталонне відео
- Поточне специфічне для OpenAI обмеження: OpenClaw наразі передає для нативної генерації відео OpenAI лише перевизначення `size`.
  Непідтримувані необов’язкові перевизначення, такі як `aspectRatio`, `resolution`, `audio` і `watermark`, ігноруються
  і повертаються як попередження інструмента.

Щоб використовувати OpenAI як провайдера відео за замовчуванням:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Див. [Генерація відео](/uk/tools/video-generation), щоб ознайомитися зі спільними
параметрами інструмента, вибором провайдера та поведінкою перемикання при збоях.

## Варіант B: підписка OpenAI Code (Codex)

**Найкраще для:** використання доступу за підпискою ChatGPT/Codex замість API-ключа.
Codex cloud вимагає входу через ChatGPT, тоді як CLI Codex підтримує вхід через ChatGPT або API-ключ.

Підсумок маршрутів:

- `openai-codex/gpt-5.4` = маршрут OAuth ChatGPT/Codex
- Використовує вхід через ChatGPT/Codex, а не прямий API-ключ OpenAI Platform
- Обмеження на боці провайдера для `openai-codex/*` можуть відрізнятися від досвіду використання вебверсії/застосунку ChatGPT

### Налаштування CLI (Codex OAuth)

```bash
# Запустити Codex OAuth у майстрі
openclaw onboard --auth-choice openai-codex

# Або запустити OAuth безпосередньо
openclaw models auth login --provider openai-codex
```

### Фрагмент конфігурації (підписка Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

У поточній документації Codex від OpenAI `gpt-5.4` вказано як поточну модель Codex. OpenClaw
відображає її як `openai-codex/gpt-5.4` для використання через OAuth ChatGPT/Codex.

Цей маршрут навмисно відокремлений від `openai/gpt-5.4`. Якщо вам потрібен
прямий шлях API OpenAI Platform, використовуйте `openai/*` з API-ключем. Якщо вам потрібен
вхід через ChatGPT/Codex, використовуйте `openai-codex/*`.

Якщо під час онбордингу повторно використовується наявний вхід CLI Codex, цими обліковими даними й надалі
керує CLI Codex. Після завершення строку дії OpenClaw спочатку повторно зчитує зовнішнє джерело Codex
і, коли провайдер може їх оновити, записує оновлені облікові дані назад у сховище Codex
замість того, щоб брати керування на себе в окремій копії лише для OpenClaw.

Якщо ваш обліковий запис Codex має право на Codex Spark, OpenClaw також підтримує:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw вважає Codex Spark доступним лише для Codex. Він не надає прямого
шляху `openai/gpt-5.3-codex-spark` з API-ключем.

OpenClaw також зберігає `openai-codex/gpt-5.3-codex-spark`, коли `pi-ai`
його виявляє. Вважайте його залежним від прав доступу та експериментальним: Codex Spark
відокремлений від GPT-5.4 `/fast`, а доступність залежить від облікового запису Codex /
ChatGPT, під яким виконано вхід.

### Обмеження контекстного вікна Codex

OpenClaw розглядає метадані моделі Codex і обмеження контексту під час виконання як окремі
значення.

Для `openai-codex/gpt-5.4`:

- нативний `contextWindow`: `1050000`
- обмеження `contextTokens` під час виконання за замовчуванням: `272000`

Це зберігає правдивість метаданих моделі, водночас залишаючи менше контекстне
вікно за замовчуванням, яке на практиці має кращі характеристики затримки й якості.

Якщо ви хочете інше фактичне обмеження, задайте `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Використовуйте `contextWindow` лише тоді, коли ви оголошуєте або перевизначаєте нативні
метадані моделі. Використовуйте `contextTokens`, коли хочете обмежити бюджет контексту під час виконання.

### Транспорт за замовчуванням

OpenClaw використовує `pi-ai` для потокової передачі моделей. І для `openai/*`, і для
`openai-codex/*` транспорт за замовчуванням — `"auto"` (спочатку WebSocket, потім резервний варіант SSE).

У режимі `"auto"` OpenClaw також повторює одну ранню WebSocket-помилку, яку можна повторити,
перш ніж перейти на SSE. Примусовий режим `"websocket"` і надалі безпосередньо показує помилки транспорту,
а не приховує їх за резервним варіантом.

Після помилки WebSocket під час підключення або на ранньому етапі запиту в режимі `"auto"` OpenClaw позначає
шлях WebSocket цього сеансу як деградований приблизно на 60 секунд і надсилає
наступні запити через SSE протягом періоду охолодження замість постійного
перемикання між транспортами.

Для нативних кінцевих точок сімейства OpenAI (`openai/*`, `openai-codex/*` і Azure
OpenAI Responses) OpenClaw також додає до запитів стабільний стан ідентичності сеансу й запиту,
щоб повторні спроби, повторні підключення та резервний перехід на SSE залишалися узгодженими з тією самою
ідентичністю розмови. На нативних маршрутах сімейства OpenAI це включає стабільні заголовки ідентичності запиту для сеансу/запиту та відповідні метадані транспорту.

OpenClaw також нормалізує лічильники використання OpenAI для різних варіантів транспорту, перш ніж
вони потрапляють до поверхонь session/status. Нативний трафік OpenAI/Codex Responses може
повідомляти використання як `input_tokens` / `output_tokens` або
`prompt_tokens` / `completion_tokens`; OpenClaw вважає їх однаковими лічильниками вхідних
і вихідних токенів для `/status`, `/usage` та журналів сеансів. Коли нативний
WebSocket-трафік не містить `total_tokens` (або повідомляє `0`), OpenClaw використовує нормалізовану суму вхідних і вихідних токенів як резервний варіант, щоб відображення session/status залишалося заповненим.

Ви можете задати `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: примусово використовувати SSE
- `"websocket"`: примусово використовувати WebSocket
- `"auto"`: спробувати WebSocket, а потім перейти на SSE

Для `openai/*` (Responses API) OpenClaw також за замовчуванням увімкнено прогрівання WebSocket
(`openaiWsWarmup: true`), коли використовується транспорт WebSocket.

Пов’язана документація OpenAI:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### Прогрівання OpenAI WebSocket

У документації OpenAI прогрівання описується як необов’язкове. OpenClaw вмикає його за замовчуванням для
`openai/*`, щоб зменшити затримку першого запиту під час використання транспорту WebSocket.

### Вимкнення прогрівання

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Явне ввімкнення прогрівання

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### Пріоритетна обробка для OpenAI і Codex

API OpenAI підтримує пріоритетну обробку через `service_tier=priority`. У
OpenClaw встановіть `agents.defaults.models["<provider>/<model>"].params.serviceTier`,
щоб передати це поле до нативних кінцевих точок OpenAI/Codex Responses.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Підтримувані значення: `auto`, `default`, `flex` і `priority`.

OpenClaw передає `params.serviceTier` як до прямих запитів `openai/*` Responses,
так і до запитів `openai-codex/*` Codex Responses, коли ці моделі вказують
на нативні кінцеві точки OpenAI/Codex.

Важлива поведінка:

- прямий `openai/*` має вказувати на `api.openai.com`
- `openai-codex/*` має вказувати на `chatgpt.com/backend-api`
- якщо ви спрямовуєте будь-якого з цих провайдерів через іншу базову URL-адресу або проксі, OpenClaw не змінює `service_tier`

### Швидкий режим OpenAI

OpenClaw надає спільний перемикач швидкого режиму як для сеансів `openai/*`, так і для
`openai-codex/*`:

- Chat/UI: `/fast status|on|off`
- Конфігурація: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Коли швидкий режим увімкнено, OpenClaw відображає його на пріоритетну обробку OpenAI:

- прямі виклики `openai/*` Responses до `api.openai.com` надсилають `service_tier = "priority"`
- виклики `openai-codex/*` Responses до `chatgpt.com/backend-api` також надсилають `service_tier = "priority"`
- наявні значення `service_tier` у payload зберігаються
- швидкий режим не переписує `reasoning` або `text.verbosity`

Для GPT 5.4 зокрема найпоширеніше налаштування таке:

- надішліть `/fast on` у сеансі з `openai/gpt-5.4` або `openai-codex/gpt-5.4`
- або встановіть `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- якщо ви також використовуєте Codex OAuth, установіть `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true` також

Приклад:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Перевизначення сеансу мають пріоритет над конфігурацією. Очищення перевизначення сеансу в інтерфейсі Sessions UI
повертає сеанс до налаштованого значення за замовчуванням.

### Нативні маршрути OpenAI проти OpenAI-сумісних маршрутів

OpenClaw по-різному обробляє прямі кінцеві точки OpenAI, Codex і Azure OpenAI
порівняно із загальними OpenAI-сумісними проксі `/v1`:

- нативні маршрути `openai/*`, `openai-codex/*` і Azure OpenAI зберігають
  `reasoning: { effort: "none" }` без змін, коли ви явно вимикаєте reasoning
- нативні маршрути сімейства OpenAI за замовчуванням використовують строгий режим для схем інструментів
- приховані заголовки атрибуції OpenClaw (`originator`, `version` і
  `User-Agent`) додаються лише до перевірених нативних хостів OpenAI
  (`api.openai.com`) і нативних хостів Codex (`chatgpt.com/backend-api`)
- нативні маршрути OpenAI/Codex зберігають специфічне для OpenAI формування запитів, таке як
  `service_tier`, Responses `store`, OpenAI reasoning-compat payloads і
  prompt-cache hints
- OpenAI-сумісні маршрути у стилі проксі зберігають вільнішу сумісну поведінку і
  не примушують до строгих схем інструментів, формування запитів лише для нативних маршрутів або прихованих
  заголовків атрибуції OpenAI/Codex

Azure OpenAI залишається в категорії нативної маршрутизації для транспортної та сумісної
поведінки, але не отримує приховані заголовки атрибуції OpenAI/Codex.

Це зберігає поточну нативну поведінку OpenAI Responses без примусового застосування старіших
OpenAI-сумісних shim-рішень до сторонніх бекендів `/v1`.

### Строгий агентний режим GPT

Для запусків сімейства GPT-5 через `openai/*` і `openai-codex/*` OpenClaw може використовувати
суворіший вбудований контракт виконання Pi:

```json5
{
  agents: {
    defaults: {
      embeddedPi: {
        executionContract: "strict-agentic",
      },
    },
  },
}
```

Із `strict-agentic` OpenClaw більше не вважає відповідь асистента, що містить лише план,
успішним прогресом, якщо доступна конкретна дія з інструментом. Він повторює
хід із вказівкою діяти негайно, автоматично вмикає структурований інструмент `update_plan` для
суттєвої роботи та показує явний стан блокування, якщо модель продовжує
планувати без дій.

Цей режим обмежений запусками сімейства GPT-5 для OpenAI і OpenAI Codex. Інші провайдери
та старіші сімейства моделей зберігають поведінку вбудованого Pi за замовчуванням, якщо ви явно
не ввімкнете для них інші налаштування часу виконання.

### Ущільнення на боці сервера OpenAI Responses

Для прямих моделей OpenAI Responses (`openai/*`, що використовують `api: "openai-responses"` з
`baseUrl` на `api.openai.com`) OpenClaw тепер автоматично вмикає підказки payload для
ущільнення на боці сервера OpenAI:

- Примусово встановлює `store: true` (якщо compat моделі не встановлює `supportsStore: false`)
- Вставляє `context_management: [{ type: "compaction", compact_threshold: ... }]`

За замовчуванням `compact_threshold` становить `70%` від `contextWindow` моделі (або `80000`,
якщо він недоступний).

### Явне ввімкнення ущільнення на боці сервера

Використовуйте це, якщо хочете примусово вставляти `context_management` для сумісних
моделей Responses (наприклад Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Увімкнення з власним порогом

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Вимкнення ущільнення на боці сервера

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` керує лише вставленням `context_management`.
Прямі моделі OpenAI Responses усе одно примусово встановлюють `store: true`, якщо compat не встановлює
`supportsStore: false`.

## Примітки

- Посилання на моделі завжди використовують формат `provider/model` (див. [/concepts/models](/uk/concepts/models)).
- Докладні відомості про автентифікацію та правила повторного використання наведено в [/concepts/oauth](/uk/concepts/oauth).
