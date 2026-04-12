---
read_when:
    - Ви хочете використовувати моделі OpenAI в OpenClaw
    - Ви хочете автентифікацію через підписку Codex замість API-ключів
    - Вам потрібна суворіша поведінка виконання агента GPT-5
summary: Використовуйте OpenAI через API-ключі або підписку Codex в OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-12T09:46:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0a2eb5f742a68c4cdd1cff3bc9c0f52663b3d668b00a0db61ccf7dfb0d3f7b1d
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI надає API для розробників для моделей GPT. OpenClaw підтримує два способи автентифікації:

- **API key** — прямий доступ до OpenAI Platform з оплатою за використання (`openai/*` моделі)
- **Codex subscription** — вхід через ChatGPT/Codex із доступом за підпискою (`openai-codex/*` моделі)

OpenAI прямо підтримує використання OAuth за підпискою у зовнішніх інструментах і робочих процесах, таких як OpenClaw.

## Початок роботи

Виберіть бажаний спосіб автентифікації та виконайте кроки налаштування.

<Tabs>
  <Tab title="API key (OpenAI Platform)">
    **Найкраще для:** прямого доступу до API та оплати за використання.

    <Steps>
      <Step title="Отримайте свій API key">
        Створіть або скопіюйте API key з [панелі OpenAI Platform](https://platform.openai.com/api-keys).
      </Step>
      <Step title="Запустіть онбординг">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Або передайте ключ напряму:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="Перевірте, що модель доступна">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### Підсумок маршрутів

    | Model ref | Маршрут | Автентифікація |
    |-----------|-------|------|
    | `openai/gpt-5.4` | Прямий API OpenAI Platform | `OPENAI_API_KEY` |
    | `openai/gpt-5.4-pro` | Прямий API OpenAI Platform | `OPENAI_API_KEY` |

    <Note>
    Вхід через ChatGPT/Codex спрямовується через `openai-codex/*`, а не `openai/*`.
    </Note>

    ### Приклад конфігурації

    ```json5
    {
      env: { OPENAI_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
    }
    ```

    <Warning>
    OpenClaw **не** надає `openai/gpt-5.3-codex-spark` через прямий шлях API. Реальні запити до OpenAI API відхиляють цю модель. Spark доступний лише в Codex.
    </Warning>

  </Tab>

  <Tab title="Codex subscription">
    **Найкраще для:** використання вашої підписки ChatGPT/Codex замість окремого API key. Codex cloud потребує входу через ChatGPT.

    <Steps>
      <Step title="Запустіть Codex OAuth">
        ```bash
        openclaw onboard --auth-choice openai-codex
        ```

        Або запустіть OAuth напряму:

        ```bash
        openclaw models auth login --provider openai-codex
        ```
      </Step>
      <Step title="Установіть модель за замовчуванням">
        ```bash
        openclaw config set agents.defaults.model.primary openai-codex/gpt-5.4
        ```
      </Step>
      <Step title="Перевірте, що модель доступна">
        ```bash
        openclaw models list --provider openai-codex
        ```
      </Step>
    </Steps>

    ### Підсумок маршрутів

    | Model ref | Маршрут | Автентифікація |
    |-----------|-------|------|
    | `openai-codex/gpt-5.4` | ChatGPT/Codex OAuth | вхід Codex |
    | `openai-codex/gpt-5.3-codex-spark` | ChatGPT/Codex OAuth | вхід Codex (залежить від прав доступу) |

    <Note>
    Цей маршрут навмисно відокремлений від `openai/gpt-5.4`. Використовуйте `openai/*` з API key для прямого доступу до Platform, а `openai-codex/*` — для доступу за підпискою Codex.
    </Note>

    ### Приклад конфігурації

    ```json5
    {
      agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
    }
    ```

    <Tip>
    Якщо онбординг повторно використовує наявний вхід Codex CLI, цими обліковими даними й надалі керує Codex CLI. Після завершення строку дії OpenClaw спочатку повторно зчитує зовнішнє джерело Codex, а потім записує оновлені облікові дані назад у сховище Codex.
    </Tip>

    ### Обмеження вікна контексту

    OpenClaw розглядає метадані моделі та обмеження контексту під час виконання як окремі значення.

    Для `openai-codex/gpt-5.4`:

    - Нативний `contextWindow`: `1050000`
    - Обмеження `contextTokens` під час виконання за замовчуванням: `272000`

    Менше обмеження за замовчуванням на практиці дає кращі характеристики затримки та якості. Перевизначте його за допомогою `contextTokens`:

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

    <Note>
    Використовуйте `contextWindow`, щоб оголосити нативні метадані моделі. Використовуйте `contextTokens`, щоб обмежити бюджет контексту під час виконання.
    </Note>

  </Tab>
</Tabs>

## Генерація зображень

Вбудований Plugin `openai` реєструє генерацію зображень через інструмент `image_generate`.

| Можливість               | Значення                            |
| ------------------------ | ----------------------------------- |
| Модель за замовчуванням  | `openai/gpt-image-1`                |
| Максимум зображень на запит | 4                                |
| Режим редагування        | Увімкнено (до 5 еталонних зображень) |
| Перевизначення розміру   | Підтримується                       |
| Співвідношення сторін / роздільна здатність | Не передаються до OpenAI Images API |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-1" },
    },
  },
}
```

<Note>
Див. [Генерація зображень](/uk/tools/image-generation), щоб дізнатися про спільні параметри інструмента, вибір провайдера та поведінку перемикання при відмові.
</Note>

## Генерація відео

Вбудований Plugin `openai` реєструє генерацію відео через інструмент `video_generate`.

| Можливість      | Значення                                                                        |
| --------------- | ------------------------------------------------------------------------------- |
| Модель за замовчуванням | `openai/sora-2`                                                         |
| Режими          | Текст у відео, зображення у відео, редагування одного відео                     |
| Вхідні еталони  | 1 зображення або 1 відео                                                        |
| Перевизначення розміру | Підтримується                                                             |
| Інші перевизначення | `aspectRatio`, `resolution`, `audio`, `watermark` ігноруються з попередженням інструмента |

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
Див. [Генерація відео](/uk/tools/video-generation), щоб дізнатися про спільні параметри інструмента, вибір провайдера та поведінку перемикання при відмові.
</Note>

## Накладка особистості

OpenClaw додає невелику специфічну для OpenAI накладку запиту для запусків `openai/*` і `openai-codex/*`. Ця накладка робить асистента теплішим, більш налаштованим на співпрацю, лаконічнішим і трохи більш емоційно виразним, не замінюючи базовий системний запит.

| Значення               | Ефект                                |
| ---------------------- | ------------------------------------ |
| `"friendly"` (за замовчуванням) | Увімкнути специфічну для OpenAI накладку |
| `"on"`                 | Псевдонім для `"friendly"`           |
| `"off"`                | Використовувати лише базовий запит OpenClaw |

<Tabs>
  <Tab title="Config">
    ```json5
    {
      plugins: {
        entries: {
          openai: { config: { personality: "friendly" } },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set plugins.entries.openai.config.personality off
    ```
  </Tab>
</Tabs>

<Tip>
Під час виконання значення нечутливі до регістру, тому і `"Off"`, і `"off"` вимикають накладку.
</Tip>

## Розширена конфігурація

<AccordionGroup>
  <Accordion title="Транспорт (WebSocket чи SSE)">
    OpenClaw використовує WebSocket у першу чергу з резервним переходом на SSE (`"auto"`) для `openai/*` і `openai-codex/*`.

    У режимі `"auto"` OpenClaw:
    - Повторює одну ранню помилку WebSocket перед переходом на SSE
    - Після збою позначає WebSocket як деградований приблизно на 60 секунд і під час охолодження використовує SSE
    - Додає стабільні заголовки ідентичності сесії та ходу для повторних спроб і перепідключень
    - Нормалізує лічильники використання (`input_tokens` / `prompt_tokens`) між варіантами транспорту

    | Значення | Поведінка |
    |-------|----------|
    | `"auto"` (за замовчуванням) | Спочатку WebSocket, резервно SSE |
    | `"sse"` | Примусово лише SSE |
    | `"websocket"` | Примусово лише WebSocket |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai-codex/gpt-5.4": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    Пов’язана документація OpenAI:
    - [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
    - [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="Прогрівання WebSocket">
    OpenClaw за замовчуванням вмикає прогрівання WebSocket для `openai/*`, щоб зменшити затримку першого ходу.

    ```json5
    // Вимкнути прогрівання
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: { openaiWsWarmup: false },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Швидкий режим">
    OpenClaw надає спільний перемикач швидкого режиму для `openai/*` і `openai-codex/*`:

    - **Чат/UI:** `/fast status|on|off`
    - **Config:** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    Коли цей режим увімкнено, OpenClaw зіставляє швидкий режим із пріоритетною обробкою OpenAI (`service_tier = "priority"`). Наявні значення `service_tier` зберігаються, а швидкий режим не переписує `reasoning` або `text.verbosity`.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": { params: { fastMode: true } },
            "openai-codex/gpt-5.4": { params: { fastMode: true } },
          },
        },
      },
    }
    ```

    <Note>
    Перевизначення сесії мають пріоритет над конфігурацією. Очищення перевизначення сесії в UI сесій повертає сесію до налаштованого значення за замовчуванням.
    </Note>

  </Accordion>

  <Accordion title="Пріоритетна обробка (service_tier)">
    API OpenAI надає пріоритетну обробку через `service_tier`. Установіть її для кожної моделі в OpenClaw:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": { params: { serviceTier: "priority" } },
            "openai-codex/gpt-5.4": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    Підтримувані значення: `auto`, `default`, `flex`, `priority`.

    <Warning>
    `serviceTier` передається лише до нативних endpoint-ів OpenAI (`api.openai.com`) і нативних endpoint-ів Codex (`chatgpt.com/backend-api`). Якщо ви спрямовуєте будь-якого з цих провайдерів через проксі, OpenClaw залишає `service_tier` без змін.
    </Warning>

  </Accordion>

  <Accordion title="Серверна Compaction (Responses API)">
    Для прямих моделей OpenAI Responses (`openai/*` на `api.openai.com`) OpenClaw автоматично вмикає серверну Compaction:

    - Примусово встановлює `store: true` (якщо сумісність моделі не задає `supportsStore: false`)
    - Впроваджує `context_management: [{ type: "compaction", compact_threshold: ... }]`
    - `compact_threshold` за замовчуванням: 70% від `contextWindow` (або `80000`, якщо значення недоступне)

    <Tabs>
      <Tab title="Увімкнути явно">
        Корисно для сумісних endpoint-ів, таких як Azure OpenAI Responses:

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.4": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Власний поріг">
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
      </Tab>
      <Tab title="Вимкнути">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.4": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` керує лише впровадженням `context_management`. Прямі моделі OpenAI Responses усе одно примусово встановлюють `store: true`, якщо compat не задає `supportsStore: false`.
    </Note>

  </Accordion>

  <Accordion title="Суворий агентний режим GPT">
    Для запусків сімейства GPT-5 на `openai/*` і `openai-codex/*` OpenClaw може використовувати суворіший вбудований контракт виконання:

    ```json5
    {
      agents: {
        defaults: {
          embeddedPi: { executionContract: "strict-agentic" },
        },
      },
    }
    ```

    З `strict-agentic` OpenClaw:
    - Більше не вважає хід лише з планом успішним прогресом, коли доступна дія інструмента
    - Повторює хід із вказівкою діяти негайно
    - Автоматично вмикає `update_plan` для суттєвої роботи
    - Показує явний заблокований стан, якщо модель продовжує планувати без дії

    <Note>
    Поширюється лише на запуски сімейства GPT-5 від OpenAI і Codex. Для інших провайдерів і старіших сімейств моделей зберігається поведінка за замовчуванням.
    </Note>

  </Accordion>

  <Accordion title="Нативні маршрути та маршрути, сумісні з OpenAI">
    OpenClaw по-різному обробляє прямі endpoint-и OpenAI, Codex і Azure OpenAI порівняно із загальними проксі `/v1`, сумісними з OpenAI:

    **Нативні маршрути** (`openai/*`, `openai-codex/*`, Azure OpenAI):
    - Зберігають `reasoning: { effort: "none" }` без змін, коли міркування явно вимкнено
    - За замовчуванням використовують суворий режим для схем інструментів
    - Додають приховані заголовки атрибуції лише на перевірених нативних хостах
    - Зберігають формування запитів, специфічне для OpenAI (`service_tier`, `store`, compat міркувань, підказки кешу запитів)

    **Проксі/сумісні маршрути:**
    - Використовують м’якшу compat-поведінку
    - Не примушують суворі схеми інструментів або нативні заголовки

    Azure OpenAI використовує нативний транспорт і compat-поведінку, але не отримує приховані заголовки атрибуції.

  </Accordion>
</AccordionGroup>

## Пов’язане

<CardGroup cols={2}>
  <Card title="Вибір моделі" href="/uk/concepts/model-providers" icon="layers">
    Вибір провайдерів, посилань на моделі та поведінки перемикання при відмові.
  </Card>
  <Card title="Генерація зображень" href="/uk/tools/image-generation" icon="image">
    Спільні параметри інструмента зображень і вибір провайдера.
  </Card>
  <Card title="Генерація відео" href="/uk/tools/video-generation" icon="video">
    Спільні параметри інструмента відео та вибір провайдера.
  </Card>
  <Card title="OAuth і автентифікація" href="/uk/gateway/authentication" icon="key">
    Докладніше про автентифікацію та правила повторного використання облікових даних.
  </Card>
</CardGroup>
