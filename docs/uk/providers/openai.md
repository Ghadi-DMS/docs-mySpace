---
read_when:
    - Ви хочете використовувати моделі OpenAI в OpenClaw
    - Ви хочете автентифікацію через підписку Codex замість API-ключів
summary: Використовуйте OpenAI через API-ключі або підписку Codex в OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-06T00:04:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: d9a024b102f24a2f690efdbe633a01fbe00f500a74a8007af3245b8b948394ef
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI надає API для розробників для моделей GPT. Codex підтримує **вхід через ChatGPT** для доступу за підпискою
або **вхід через API key** для доступу з оплатою за використання. Для Codex cloud потрібен вхід через ChatGPT.
OpenAI прямо підтримує використання OAuth за підпискою у зовнішніх інструментах і робочих процесах, таких як OpenClaw.

## Типова взаємодія

OpenClaw може додавати невелике специфічне для OpenAI накладання промпту як для запусків `openai/*`, так і для
`openai-codex/*`. Типово це накладання робить асистента доброзичливим,
співпрацюючим, лаконічним, прямим і трохи більш емоційно виразним,
не замінюючи базовий системний промпт OpenClaw. Доброзичливе накладання також
дозволяє зрідка використовувати emoji, коли це природно, водночас зберігаючи загальну
лаконічність відповіді.

Ключ конфігурації:

`plugins.entries.openai.config.personality`

Дозволені значення:

- `"friendly"`: типове; увімкнути специфічне для OpenAI накладання.
- `"off"`: вимкнути накладання та використовувати лише базовий промпт OpenClaw.

Область дії:

- Застосовується до моделей `openai/*`.
- Застосовується до моделей `openai-codex/*`.
- Не впливає на інших провайдерів.

Цю поведінку увімкнено типово. Залиште `"friendly"` явно, якщо хочете, щоб
це збереглося після майбутніх локальних змін конфігурації:

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

### Вимкнути накладання промпту OpenAI

Якщо ви хочете незмінений базовий промпт OpenClaw, встановіть для накладання значення `"off"`:

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

## Варіант A: API key OpenAI (OpenAI Platform)

**Найкраще підходить для:** прямого доступу до API та оплати за використання.
Отримайте свій API key на панелі керування OpenAI.

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

У поточній документації OpenAI для моделей API вказано `gpt-5.4` і `gpt-5.4-pro` для прямого
використання API OpenAI. OpenClaw передає обидві через шлях Responses `openai/*`.
OpenClaw навмисно приховує застарілий рядок `openai/gpt-5.3-codex-spark`,
оскільки прямі виклики API OpenAI відхиляють його в реальному трафіку.

OpenClaw **не** показує `openai/gpt-5.3-codex-spark` на шляху прямого API OpenAI.
`pi-ai` усе ще постачається з вбудованим рядком для цієї моделі, але реальні запити до API OpenAI
зараз її відхиляють. В OpenClaw Spark розглядається лише як Codex.

## Генерація зображень

Вбудований плагін `openai` також реєструє генерацію зображень через спільний
інструмент `image_generate`.

- Типова модель зображень: `openai/gpt-image-1`
- Генерація: до 4 зображень за запит
- Режим редагування: увімкнено, до 5 еталонних зображень
- Підтримується `size`
- Поточне специфічне для OpenAI обмеження: OpenClaw наразі не передає перевизначення `aspectRatio` або
  `resolution` до OpenAI Images API

Щоб використовувати OpenAI як типового провайдера зображень:

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

Див. [Генерація зображень](/uk/tools/image-generation) щодо параметрів спільного інструмента,
вибору провайдера та поведінки під час відмови.

## Генерація відео

Вбудований плагін `openai` також реєструє генерацію відео через спільний
інструмент `video_generate`.

- Типова модель відео: `openai/sora-2`
- Режими: текст-у-відео, зображення-у-відео та сценарії з одним еталонним відео/редагуванням
- Поточні обмеження: 1 зображення або 1 еталонне відео на вхід
- Поточне специфічне для OpenAI обмеження: OpenClaw наразі передає лише перевизначення
  `size` для нативної генерації відео OpenAI. Непідтримувані необов’язкові перевизначення,
  такі як `aspectRatio`, `resolution`, `audio` і `watermark`, ігноруються
  та повертаються як попередження інструмента.

Щоб використовувати OpenAI як типового провайдера відео:

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

Див. [Генерація відео](/uk/tools/video-generation) щодо параметрів спільного інструмента,
вибору провайдера та поведінки під час відмови.

## Варіант B: підписка OpenAI Code (Codex)

**Найкраще підходить для:** використання доступу за підпискою ChatGPT/Codex замість API key.
Для Codex cloud потрібен вхід через ChatGPT, тоді як Codex CLI підтримує вхід через ChatGPT або API key.

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

У поточній документації Codex від OpenAI `gpt-5.4` вказано як актуальну модель Codex. OpenClaw
відображає її як `openai-codex/gpt-5.4` для використання через OAuth ChatGPT/Codex.

Якщо під час онбордингу повторно використовується наявний вхід Codex CLI, ці облікові дані
залишаються під керуванням Codex CLI. Після завершення строку дії OpenClaw спочатку знову читає зовнішнє джерело Codex
і, коли провайдер може їх оновити, записує оновлені облікові дані
назад у сховище Codex замість того, щоб брати їх під свій контроль в окремій копії
лише для OpenClaw.

Якщо ваш обліковий запис Codex має доступ до Codex Spark, OpenClaw також підтримує:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw розглядає Codex Spark як доступний лише для Codex. Він не надає прямий
шлях API-key `openai/gpt-5.3-codex-spark`.

OpenClaw також зберігає `openai-codex/gpt-5.3-codex-spark`, коли його виявляє `pi-ai`.
Сприймайте його як залежний від прав доступу та експериментальний: Codex Spark
відрізняється від GPT-5.4 `/fast`, а доступність залежить від облікового запису Codex /
ChatGPT, у який виконано вхід.

### Обмеження розміру контекстного вікна Codex

OpenClaw розглядає метадані моделі Codex і ліміт контексту під час виконання як окремі
значення.

Для `openai-codex/gpt-5.4`:

- нативне `contextWindow`: `1050000`
- типовий ліміт виконання `contextTokens`: `272000`

Це зберігає правдивість метаданих моделі, водночас залишаючи менше типове вікно
виконання, яке на практиці має кращі характеристики затримки та якості.

Якщо вам потрібен інший фактичний ліміт, встановіть `models.providers.<provider>.models[].contextTokens`:

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

### Типовий транспорт

OpenClaw використовує `pi-ai` для потокової передачі моделей. Для `openai/*`, і для
`openai-codex/*` типовим транспортом є `"auto"` (спочатку WebSocket, потім запасний
варіант SSE).

У режимі `"auto"` OpenClaw також повторює одну ранню WebSocket-помилку, яку можна повторити,
перш ніж перейти на SSE. Примусовий режим `"websocket"` і далі показує помилки транспорту
безпосередньо, а не приховує їх за запасним варіантом.

Після помилки WebSocket під час з’єднання або на ранньому етапі запиту в режимі `"auto"` OpenClaw позначає
WebSocket-шлях цієї сесії як деградований приблизно на 60 секунд і надсилає
наступні запити через SSE протягом періоду охолодження замість того, щоб постійно
перемикатися між транспортами.

Для нативних endpoint-ів сімейства OpenAI (`openai/*`, `openai-codex/*` і Azure
OpenAI Responses) OpenClaw також додає стабільний стан ідентифікації сесії та запиту
до запитів, щоб повторні спроби, перепідключення та запасний варіант SSE лишалися прив’язаними до тієї самої
ідентичності розмови. На нативних маршрутах сімейства OpenAI це включає стабільні
заголовки ідентифікації запитів сесії/запиту та відповідні метадані транспорту.

OpenClaw також нормалізує лічильники використання OpenAI для різних варіантів транспорту, перш ніж
вони потраплять на поверхні сесії/статусу. Нативний трафік OpenAI/Codex Responses може
повідомляти про використання як `input_tokens` / `output_tokens` або
`prompt_tokens` / `completion_tokens`; OpenClaw розглядає їх як ті самі лічильники вхідних
і вихідних токенів для `/status`, `/usage` і журналів сесій. Коли нативний
WebSocket-трафік не містить `total_tokens` (або вказує `0`), OpenClaw повертається до
нормалізованої суми вхідних і вихідних токенів, щоб відображення сесії/статусу лишалося заповненим.

Ви можете встановити `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: примусово використовувати SSE
- `"websocket"`: примусово використовувати WebSocket
- `"auto"`: спробувати WebSocket, потім перейти на SSE

Для `openai/*` (Responses API) OpenClaw також типово вмикає попередній прогрів WebSocket
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

### Попередній прогрів WebSocket OpenAI

У документації OpenAI прогрів описано як необов’язковий. OpenClaw типово вмикає його для
`openai/*`, щоб зменшити затримку першого запиту під час використання транспорту WebSocket.

### Вимкнути прогрів

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

### Явно ввімкнути прогрів

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

### Пріоритетна обробка OpenAI і Codex

API OpenAI надає пріоритетну обробку через `service_tier=priority`. У
OpenClaw встановіть `agents.defaults.models["<provider>/<model>"].params.serviceTier`,
щоб передавати це поле до нативних endpoint-ів OpenAI/Codex Responses.

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

OpenClaw передає `params.serviceTier` і до прямих запитів `openai/*` Responses,
і до запитів `openai-codex/*` Codex Responses, коли ці моделі вказують
на нативні endpoint-и OpenAI/Codex.

Важлива поведінка:

- прямий `openai/*` має вказувати на `api.openai.com`
- `openai-codex/*` має вказувати на `chatgpt.com/backend-api`
- якщо ви спрямовуєте будь-якого з цих провайдерів через інший base URL або проксі, OpenClaw залишає `service_tier` без змін

### Швидкий режим OpenAI

OpenClaw надає спільний перемикач швидкого режиму як для сесій `openai/*`, так і для
`openai-codex/*`:

- Чат/UI: `/fast status|on|off`
- Конфігурація: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Коли швидкий режим увімкнено, OpenClaw відображає його на пріоритетну обробку OpenAI:

- прямі виклики `openai/*` Responses до `api.openai.com` надсилають `service_tier = "priority"`
- виклики `openai-codex/*` Responses до `chatgpt.com/backend-api` також надсилають `service_tier = "priority"`
- наявні значення `service_tier` у корисному навантаженні зберігаються
- швидкий режим не переписує `reasoning` або `text.verbosity`

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

Перевизначення на рівні сесії мають вищий пріоритет, ніж конфігурація. Очищення перевизначення сесії в UI сесій
повертає сесію до налаштованого типового значення.

### Нативні маршрути OpenAI проти OpenAI-compatible маршрутів

OpenClaw по-різному обробляє прямі endpoint-и OpenAI, Codex і Azure OpenAI
порівняно із загальними OpenAI-compatible проксі `/v1`:

- нативні маршрути `openai/*`, `openai-codex/*` і Azure OpenAI зберігають
  `reasoning: { effort: "none" }` без змін, коли ви явно вимикаєте reasoning
- нативні маршрути сімейства OpenAI типово встановлюють strict mode для схем інструментів
- приховані заголовки атрибуції OpenClaw (`originator`, `version` і
  `User-Agent`) додаються лише для перевірених нативних хостів OpenAI
  (`api.openai.com`) і нативних хостів Codex (`chatgpt.com/backend-api`)
- нативні маршрути OpenAI/Codex зберігають формування запитів, доступне лише для OpenAI, таке як
  `service_tier`, `store` у Responses, payload-и сумісності reasoning OpenAI і
  підказки кешу промптів
- OpenAI-compatible маршрути проксі-зразка зберігають м’якшу поведінку сумісності й не
  примушують strict mode для схем інструментів, нативне формування запитів або приховані
  заголовки атрибуції OpenAI/Codex

Azure OpenAI лишається в групі нативної маршрутизації щодо транспорту та поведінки
сумісності, але не отримує прихованих заголовків атрибуції OpenAI/Codex.

Це зберігає поточну нативну поведінку OpenAI Responses без нав’язування старих
OpenAI-compatible shim-ів стороннім бекендам `/v1`.

### Серверна компактація OpenAI Responses

Для прямих моделей OpenAI Responses (`openai/*`, що використовують `api: "openai-responses"` з
`baseUrl` на `api.openai.com`), OpenClaw тепер автоматично вмикає підказки корисного навантаження
для серверної компактації OpenAI:

- Примусово встановлює `store: true` (якщо сумісність моделі не задає `supportsStore: false`)
- Вставляє `context_management: [{ type: "compaction", compact_threshold: ... }]`

Типово `compact_threshold` становить `70%` від `contextWindow` моделі (або `80000`,
якщо значення недоступне).

### Явно ввімкнути серверну компактацію

Використовуйте це, якщо хочете примусово вставляти `context_management` для сумісних
моделей Responses (наприклад, Azure OpenAI Responses):

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

### Увімкнути з власним порогом

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

### Вимкнути серверну компактацію

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

`responsesServerCompaction` керує лише вставлянням `context_management`.
Прямі моделі OpenAI Responses і далі примусово встановлюють `store: true`, якщо сумісність
не задає `supportsStore: false`.

## Примітки

- Посилання на моделі завжди використовують формат `provider/model` (див. [/concepts/models](/uk/concepts/models)).
- Докладніше про автентифікацію та правила повторного використання — у [/concepts/oauth](/uk/concepts/oauth).
