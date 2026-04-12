---
read_when:
    - Ви хочете запустити OpenClaw із хмарними або локальними моделями через Ollama
    - Вам потрібні вказівки щодо налаштування та конфігурації Ollama
summary: Запустіть OpenClaw з Ollama (хмарні та локальні моделі)
title: Ollama
x-i18n:
    generated_at: "2026-04-12T11:08:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec796241b884ca16ec7077df4f3f1910e2850487bb3ea94f8fdb37c77e02b219
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama — це локальне середовище виконання LLM, яке спрощує запуск моделей з відкритим кодом на вашій машині. OpenClaw інтегрується з нативним API Ollama (`/api/chat`), підтримує потокову передачу та виклик інструментів, а також може автоматично виявляти локальні моделі Ollama, якщо ви ввімкнете це через `OLLAMA_API_KEY` (або профіль автентифікації) і не визначите явний запис `models.providers.ollama`.

<Warning>
**Користувачі віддаленого Ollama**: Не використовуйте OpenAI-сумісну URL-адресу `/v1` (`http://host:11434/v1`) з OpenClaw. Це ламає виклик інструментів, і моделі можуть виводити сирий JSON інструментів як звичайний текст. Натомість використовуйте нативну URL-адресу API Ollama: `baseUrl: "http://host:11434"` (без `/v1`).
</Warning>

## Початок роботи

Виберіть бажаний спосіб налаштування та режим.

<Tabs>
  <Tab title="Онбординг (рекомендовано)">
    **Найкраще для:** найшвидшого шляху до робочого налаштування Ollama з автоматичним виявленням моделей.

    <Steps>
      <Step title="Запустіть онбординг">
        ```bash
        openclaw onboard
        ```

        Виберіть **Ollama** зі списку провайдерів.
      </Step>
      <Step title="Виберіть свій режим">
        - **Cloud + Local** — хмарні моделі та локальні моделі разом
        - **Local** — лише локальні моделі

        Якщо ви виберете **Cloud + Local** і не ввійшли на ollama.com, онбординг відкриє у браузері процес входу.
      </Step>
      <Step title="Виберіть модель">
        Онбординг виявляє доступні моделі та пропонує типові варіанти. Він автоматично завантажує вибрану модель, якщо вона недоступна локально.
      </Step>
      <Step title="Переконайтеся, що модель доступна">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    ### Неінтерактивний режим

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --accept-risk
    ```

    За потреби вкажіть власну базову URL-адресу або модель:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

  </Tab>

  <Tab title="Ручне налаштування">
    **Найкраще для:** повного контролю над інсталяцією, завантаженням моделей і конфігурацією.

    <Steps>
      <Step title="Встановіть Ollama">
        Завантажте з [ollama.com/download](https://ollama.com/download).
      </Step>
      <Step title="Завантажте локальну модель">
        ```bash
        ollama pull gemma4
        # або
        ollama pull gpt-oss:20b
        # або
        ollama pull llama3.3
        ```
      </Step>
      <Step title="Увійдіть для хмарних моделей (необов’язково)">
        Якщо ви також хочете хмарні моделі:

        ```bash
        ollama signin
        ```
      </Step>
      <Step title="Увімкніть Ollama для OpenClaw">
        Встановіть будь-яке значення для API key (Ollama не потребує справжнього ключа):

        ```bash
        # Встановити змінну середовища
        export OLLAMA_API_KEY="ollama-local"

        # Або налаштувати у вашому файлі конфігурації
        openclaw config set models.providers.ollama.apiKey "ollama-local"
        ```
      </Step>
      <Step title="Перегляньте та встановіть вашу модель">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Або встановіть типову модель у конфігурації:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Хмарні моделі

<Tabs>
  <Tab title="Cloud + Local">
    Хмарні моделі дають змогу запускати моделі, розміщені в хмарі, поряд із вашими локальними моделями. Приклади: `kimi-k2.5:cloud`, `minimax-m2.7:cloud` і `glm-5.1:cloud` -- для них **не** потрібен локальний `ollama pull`.

    Під час налаштування виберіть режим **Cloud + Local**. Майстер перевіряє, чи ви ввійшли в систему, і за потреби відкриває у браузері процес входу. Якщо автентифікацію неможливо перевірити, майстер повертається до типових локальних моделей.

    Ви також можете увійти безпосередньо на [ollama.com/signin](https://ollama.com/signin).

    OpenClaw наразі пропонує такі типові хмарні варіанти: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`.

  </Tab>

  <Tab title="Лише локально">
    У режимі лише локальних моделей OpenClaw виявляє моделі з локального екземпляра Ollama. Вхід у хмару не потрібен.

    OpenClaw наразі пропонує `gemma4` як типову локальну модель.

  </Tab>
</Tabs>

## Виявлення моделей (неявний провайдер)

Коли ви встановлюєте `OLLAMA_API_KEY` (або профіль автентифікації) і **не** визначаєте `models.providers.ollama`, OpenClaw виявляє моделі з локального екземпляра Ollama за адресою `http://127.0.0.1:11434`.

| Поведінка             | Деталі                                                                                                                                                               |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Запит каталогу        | Виконує запити до `/api/tags`                                                                                                                                         |
| Виявлення можливостей | Використовує найкращі доступні запити `/api/show`, щоб зчитати `contextWindow` і виявити можливості (зокрема vision)                                               |
| Vision-моделі         | Моделі з можливістю `vision`, про яку повідомляє `/api/show`, позначаються як такі, що підтримують зображення (`input: ["text", "image"]`), тому OpenClaw автоматично додає зображення до запиту |
| Виявлення reasoning   | Позначає `reasoning` за евристикою назви моделі (`r1`, `reasoning`, `think`)                                                                                        |
| Обмеження токенів     | Встановлює `maxTokens` на типове максимальне обмеження токенів Ollama, яке використовує OpenClaw                                                                    |
| Вартість              | Встановлює всі вартості в `0`                                                                                                                                         |

Це дає змогу уникнути ручного додавання моделей і водночас зберігати каталог узгодженим із локальним екземпляром Ollama.

```bash
# Переглянути, які моделі доступні
ollama list
openclaw models list
```

Щоб додати нову модель, просто завантажте її через Ollama:

```bash
ollama pull mistral
```

Нова модель буде автоматично виявлена та стане доступною для використання.

<Note>
Якщо ви явно задаєте `models.providers.ollama`, автоматичне виявлення пропускається, і вам потрібно визначити моделі вручну. Дивіться розділ про явну конфігурацію нижче.
</Note>

## Конфігурація

<Tabs>
  <Tab title="Базова (неявне виявлення)">
    Найпростіший спосіб увімкнути Ollama — через змінну середовища:

    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Якщо встановлено `OLLAMA_API_KEY`, ви можете не вказувати `apiKey` у записі провайдера, і OpenClaw підставить його для перевірок доступності.
    </Tip>

  </Tab>

  <Tab title="Явна (ручні моделі)">
    Використовуйте явну конфігурацію, коли Ollama працює на іншому хості/порту, ви хочете примусово задати певні розміри контекстного вікна або списки моделей, або вам потрібні повністю ручні визначення моделей.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            apiKey: "ollama-local",
            api: "ollama",
            models: [
              {
                id: "gpt-oss:20b",
                name: "GPT-OSS 20B",
                reasoning: false,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 8192,
                maxTokens: 8192 * 10
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="Власна базова URL-адреса">
    Якщо Ollama працює на іншому хості або порту (явна конфігурація вимикає автоматичне виявлення, тому визначте моделі вручну):

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Без /v1 — використовуйте нативну URL-адресу API Ollama
            api: "ollama", // Вкажіть явно, щоб гарантувати нативну поведінку виклику інструментів
          },
        },
      },
    }
    ```

    <Warning>
    Не додавайте `/v1` до URL-адреси. Шлях `/v1` використовує OpenAI-сумісний режим, у якому виклик інструментів ненадійний. Використовуйте базову URL-адресу Ollama без суфікса шляху.
    </Warning>

  </Tab>
</Tabs>

### Вибір моделі

Після налаштування всі ваші моделі Ollama будуть доступні:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Ollama Web Search

OpenClaw підтримує **Ollama Web Search** як вбудований провайдер `web_search`.

| Властивість | Деталі                                                                                                               |
| ----------- | -------------------------------------------------------------------------------------------------------------------- |
| Хост        | Використовує налаштований хост Ollama (`models.providers.ollama.baseUrl`, якщо задано, інакше `http://127.0.0.1:11434`) |
| Автентифікація | Без ключа                                                                                                         |
| Вимога      | Ollama має бути запущений, і потрібно виконати вхід через `ollama signin`                                           |

Виберіть **Ollama Web Search** під час `openclaw onboard` або `openclaw configure --section web`, або задайте:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

<Note>
Повний опис налаштування та деталей поведінки дивіться в [Ollama Web Search](/uk/tools/ollama-search).
</Note>

## Розширена конфігурація

<AccordionGroup>
  <Accordion title="Застарілий OpenAI-сумісний режим">
    <Warning>
    **Виклик інструментів ненадійний в OpenAI-сумісному режимі.** Використовуйте цей режим лише тоді, коли вам потрібен формат OpenAI для проксі й ви не залежите від нативної поведінки виклику інструментів.
    </Warning>

    Якщо вам все ж потрібно використовувати OpenAI-сумісну кінцеву точку (наприклад, за проксі, який підтримує лише формат OpenAI), явно задайте `api: "openai-completions"`:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // default: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    У цьому режимі потокова передача та виклик інструментів можуть не підтримуватися одночасно. Можливо, вам доведеться вимкнути потокову передачу через `params: { streaming: false }` у конфігурації моделі.

    Коли `api: "openai-completions"` використовується з Ollama, OpenClaw типово додає `options.num_ctx`, щоб Ollama не переходив мовчки до контекстного вікна 4096. Якщо ваш проксі/апстрим відхиляє невідомі поля `options`, вимкніть цю поведінку:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Контекстні вікна">
    Для автоматично виявлених моделей OpenClaw використовує розмір контекстного вікна, який повідомляє Ollama, а якщо він недоступний — повертається до типового контекстного вікна Ollama, яке використовує OpenClaw.

    Ви можете перевизначити `contextWindow` і `maxTokens` у явній конфігурації провайдера:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
              }
            ]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Моделі reasoning">
    OpenClaw типово вважає моделі з назвами на кшталт `deepseek-r1`, `reasoning` або `think` такими, що підтримують reasoning.

    ```bash
    ollama pull deepseek-r1:32b
    ```

    Додаткова конфігурація не потрібна -- OpenClaw позначає їх автоматично.

  </Accordion>

  <Accordion title="Вартість моделей">
    Ollama є безкоштовним і працює локально, тому вартість усіх моделей встановлена в $0. Це стосується як автоматично виявлених, так і вручну визначених моделей.
  </Accordion>

  <Accordion title="Ембедінги пам’яті">
    Вбудований Plugin Ollama реєструє провайдер ембедінгів пам’яті для
    [пошуку в пам’яті](/uk/concepts/memory). Він використовує налаштовану базову URL-адресу Ollama
    та API key.

    | Властивість    | Значення            |
    | --------------- | ------------------- |
    | Типова модель   | `nomic-embed-text`  |
    | Автозавантаження | Так — модель ембедінгів завантажується автоматично, якщо її немає локально |

    Щоб вибрати Ollama як провайдера ембедінгів для пошуку в пам’яті:

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: { provider: "ollama" },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Конфігурація потокової передачі">
    Інтеграція Ollama в OpenClaw типово використовує **нативний API Ollama** (`/api/chat`), який повністю підтримує одночасно потокову передачу та виклик інструментів. Жодна спеціальна конфігурація не потрібна.

    <Tip>
    Якщо вам потрібно використовувати OpenAI-сумісну кінцеву точку, дивіться розділ "Застарілий OpenAI-сумісний режим" вище. У цьому режимі потокова передача та виклик інструментів можуть не працювати одночасно.
    </Tip>

  </Accordion>
</AccordionGroup>

## Усунення несправностей

<AccordionGroup>
  <Accordion title="Ollama не виявлено">
    Переконайтеся, що Ollama запущено, що ви встановили `OLLAMA_API_KEY` (або профіль автентифікації), і що ви **не** визначили явний запис `models.providers.ollama`:

    ```bash
    ollama serve
    ```

    Перевірте, що API доступний:

    ```bash
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Немає доступних моделей">
    Якщо ваша модель не відображається у списку, або завантажте її локально, або визначте її явно в `models.providers.ollama`.

    ```bash
    ollama list  # Переглянути, що встановлено
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Або іншу модель
    ```

  </Accordion>

  <Accordion title="У з’єднанні відмовлено">
    Перевірте, що Ollama працює на правильному порту:

    ```bash
    # Перевірити, чи запущено Ollama
    ps aux | grep ollama

    # Або перезапустити Ollama
    ollama serve
    ```

  </Accordion>
</AccordionGroup>

<Note>
Більше допомоги: [Усунення несправностей](/uk/help/troubleshooting) і [FAQ](/uk/help/faq).
</Note>

## Пов’язане

<CardGroup cols={2}>
  <Card title="Провайдери моделей" href="/uk/concepts/model-providers" icon="layers">
    Огляд усіх провайдерів, посилань на моделі та поведінки failover.
  </Card>
  <Card title="Вибір моделі" href="/uk/concepts/models" icon="brain">
    Як вибирати та налаштовувати моделі.
  </Card>
  <Card title="Ollama Web Search" href="/uk/tools/ollama-search" icon="magnifying-glass">
    Повні відомості про налаштування та поведінку вебпошуку на базі Ollama.
  </Card>
  <Card title="Конфігурація" href="/uk/gateway/configuration" icon="gear">
    Повний довідник із конфігурації.
  </Card>
</CardGroup>
