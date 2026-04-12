---
read_when:
    - Ви хочете запустити OpenClaw із хмарними або локальними моделями через Ollama
    - Вам потрібні вказівки щодо налаштування та конфігурації Ollama
summary: Запустіть OpenClaw з Ollama (хмарні та локальні моделі)
title: Ollama
x-i18n:
    generated_at: "2026-04-12T10:03:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 48c956d7b933d2ac637784eff1f0613b15a2b3eb5da7c030133427a73b064117
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama — це локальне середовище виконання LLM, яке спрощує запуск моделей із відкритим кодом на вашому комп’ютері. OpenClaw інтегрується з рідним API Ollama (`/api/chat`), підтримує потокову передачу та виклик інструментів, а також може автоматично виявляти локальні моделі Ollama, коли ви вмикаєте це за допомогою `OLLAMA_API_KEY` (або профілю автентифікації) і не визначаєте явний запис `models.providers.ollama`.

<Warning>
**Користувачі віддаленого Ollama**: Не використовуйте OpenAI-сумісну URL-адресу `/v1` (`http://host:11434/v1`) з OpenClaw. Це ламає виклик інструментів, і моделі можуть виводити необроблений JSON інструментів як звичайний текст. Натомість використовуйте рідну URL-адресу API Ollama: `baseUrl: "http://host:11434"` (без `/v1`).
</Warning>

## Початок роботи

Виберіть бажаний спосіб налаштування та режим.

<Tabs>
  <Tab title="Onboarding (recommended)">
    **Найкраще для:** найшвидшого способу отримати працездатне налаштування Ollama з автоматичним виявленням моделей.

    <Steps>
      <Step title="Run onboarding">
        ```bash
        openclaw onboard
        ```

        Виберіть **Ollama** зі списку постачальників.
      </Step>
      <Step title="Choose your mode">
        - **Cloud + Local** — хмарні та локальні моделі разом
        - **Local** — лише локальні моделі

        Якщо ви виберете **Cloud + Local** і не ввійшли в ollama.com, під час онбордингу відкриється потік входу через браузер.
      </Step>
      <Step title="Select a model">
        Під час онбордингу виявляються доступні моделі та пропонуються типові значення. Якщо вибрана модель недоступна локально, її буде автоматично завантажено.
      </Step>
      <Step title="Verify the model is available">
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

  <Tab title="Manual setup">
    **Найкраще для:** повного контролю над встановленням, завантаженням моделей і конфігурацією.

    <Steps>
      <Step title="Install Ollama">
        Завантажте з [ollama.com/download](https://ollama.com/download).
      </Step>
      <Step title="Pull a local model">
        ```bash
        ollama pull gemma4
        # або
        ollama pull gpt-oss:20b
        # або
        ollama pull llama3.3
        ```
      </Step>
      <Step title="Sign in for cloud models (optional)">
        Якщо вам також потрібні хмарні моделі:

        ```bash
        ollama signin
        ```
      </Step>
      <Step title="Enable Ollama for OpenClaw">
        Установіть будь-яке значення для ключа API (Ollama не вимагає справжнього ключа):

        ```bash
        # Установіть змінну середовища
        export OLLAMA_API_KEY="ollama-local"

        # Або налаштуйте у своєму файлі конфігурації
        openclaw config set models.providers.ollama.apiKey "ollama-local"
        ```
      </Step>
      <Step title="Inspect and set your model">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Або встановіть типове значення в конфігурації:

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
    Хмарні моделі дають змогу запускати хмарні моделі разом із локальними моделями. Приклади: `kimi-k2.5:cloud`, `minimax-m2.7:cloud` і `glm-5.1:cloud` -- для них **не** потрібен локальний `ollama pull`.

    Під час налаштування виберіть режим **Cloud + Local**. Майстер перевіряє, чи ви ввійшли в систему, і за потреби відкриває потік входу через браузер. Якщо не вдається перевірити автентифікацію, майстер повертається до типових значень локальних моделей.

    Ви також можете ввійти безпосередньо на [ollama.com/signin](https://ollama.com/signin).

    Зараз OpenClaw пропонує такі хмарні типові значення: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`.

  </Tab>

  <Tab title="Local only">
    У режимі лише локальних моделей OpenClaw виявляє моделі з локального екземпляра Ollama. Вхід у хмару не потрібен.

    Зараз OpenClaw пропонує `gemma4` як локальне типове значення.

  </Tab>
</Tabs>

## Виявлення моделей (неявний постачальник)

Коли ви встановлюєте `OLLAMA_API_KEY` (або профіль автентифікації) і **не** визначаєте `models.providers.ollama`, OpenClaw виявляє моделі з локального екземпляра Ollama за адресою `http://127.0.0.1:11434`.

| Поведінка             | Докладно                                                                                                                                                             |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Запит каталогу        | Виконує запити до `/api/tags`                                                                                                                                         |
| Визначення можливостей | Використовує найкращі зусилля з пошуком через `/api/show`, щоб зчитати `contextWindow` і визначити можливості (зокрема підтримку зображень)                       |
| Моделі з підтримкою зображень | Моделі з можливістю `vision`, про яку повідомляє `/api/show`, позначаються як такі, що підтримують зображення (`input: ["text", "image"]`), тому OpenClaw автоматично додає зображення в запит |
| Визначення міркування | Позначає `reasoning` за допомогою евристики на основі назви моделі (`r1`, `reasoning`, `think`)                                                                     |
| Ліміти токенів        | Установлює `maxTokens` на типове максимальне обмеження токенів Ollama, яке використовує OpenClaw                                                                    |
| Вартість              | Установлює всю вартість у `0`                                                                                                                                         |

Це дає змогу уникнути ручного додавання моделей і водночас зберігати каталог узгодженим із локальним екземпляром Ollama.

```bash
# Подивитися, які моделі доступні
ollama list
openclaw models list
```

Щоб додати нову модель, просто завантажте її через Ollama:

```bash
ollama pull mistral
```

Нова модель буде автоматично виявлена і стане доступною для використання.

<Note>
Якщо ви явно задаєте `models.providers.ollama`, автоматичне виявлення пропускається, і вам доведеться визначати моделі вручну. Дивіться розділ явної конфігурації нижче.
</Note>

## Конфігурація

<Tabs>
  <Tab title="Basic (implicit discovery)">
    Найпростіший спосіб увімкнути Ollama — через змінну середовища:

    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Якщо `OLLAMA_API_KEY` задано, ви можете не вказувати `apiKey` у записі постачальника, і OpenClaw підставить його для перевірок доступності.
    </Tip>

  </Tab>

  <Tab title="Explicit (manual models)">
    Використовуйте явну конфігурацію, якщо Ollama працює на іншому хості або порту, ви хочете примусово вказати певні контекстні вікна чи списки моделей, або вам потрібні повністю ручні визначення моделей.

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

  <Tab title="Custom base URL">
    Якщо Ollama працює на іншому хості або порту (явна конфігурація вимикає автоматичне виявлення, тому визначайте моделі вручну):

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Без /v1 - використовуйте рідну URL-адресу API Ollama
            api: "ollama", // Укажіть явно, щоб гарантувати рідну поведінку виклику інструментів
          },
        },
      },
    }
    ```

    <Warning>
    Не додавайте `/v1` до URL-адреси. Шлях `/v1` використовує OpenAI-сумісний режим, у якому виклик інструментів працює ненадійно. Використовуйте базову URL-адресу Ollama без суфікса шляху.
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

## Вебпошук Ollama

OpenClaw підтримує **Ollama Web Search** як вбудований постачальник `web_search`.

| Властивість | Докладно                                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------ |
| Хост        | Використовує налаштований хост Ollama (`models.providers.ollama.baseUrl`, якщо задано, інакше `http://127.0.0.1:11434`) |
| Автентифікація | Без ключа                                                                                                       |
| Вимога      | Ollama має працювати, і ви повинні бути авторизовані через `ollama signin`                                        |

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
Повні відомості про налаштування та поведінку дивіться в [Ollama Web Search](/uk/tools/ollama-search).
</Note>

## Розширена конфігурація

<AccordionGroup>
  <Accordion title="Legacy OpenAI-compatible mode">
    <Warning>
    **Виклик інструментів в OpenAI-сумісному режимі працює ненадійно.** Використовуйте цей режим лише тоді, коли вам потрібен формат OpenAI для проксі й ви не залежите від рідної поведінки виклику інструментів.
    </Warning>

    Якщо вам потрібно використовувати натомість OpenAI-сумісну кінцеву точку (наприклад, за проксі, який підтримує лише формат OpenAI), явно встановіть `api: "openai-completions"`:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // типово: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    Цей режим може не підтримувати одночасно потокову передачу й виклик інструментів. Можливо, вам доведеться вимкнути потокову передачу за допомогою `params: { streaming: false }` у конфігурації моделі.

    Коли `api: "openai-completions"` використовується з Ollama, OpenClaw типово додає `options.num_ctx`, щоб Ollama непомітно не повертався до контекстного вікна 4096. Якщо ваш проксі або upstream відхиляє невідомі поля `options`, вимкніть цю поведінку:

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

  <Accordion title="Context windows">
    Для автоматично виявлених моделей OpenClaw використовує контекстне вікно, про яке повідомляє Ollama, якщо воно доступне; інакше використовується типове контекстне вікно Ollama, яке застосовує OpenClaw.

    Ви можете перевизначити `contextWindow` і `maxTokens` в явній конфігурації постачальника:

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

  <Accordion title="Моделі з міркуванням">
    OpenClaw типово вважає моделі з назвами на кшталт `deepseek-r1`, `reasoning` або `think` такими, що підтримують міркування.

    ```bash
    ollama pull deepseek-r1:32b
    ```

    Додаткова конфігурація не потрібна -- OpenClaw позначає їх автоматично.

  </Accordion>

  <Accordion title="Вартість моделей">
    Ollama є безкоштовним і працює локально, тому для всіх моделей установлено вартість $0. Це стосується як автоматично виявлених, так і вручну визначених моделей.
  </Accordion>

  <Accordion title="Конфігурація потокової передачі">
    Інтеграція Ollama в OpenClaw типово використовує **рідний API Ollama** (`/api/chat`), який повністю підтримує одночасно потокову передачу та виклик інструментів. Жодної спеціальної конфігурації не потрібно.

    <Tip>
    Якщо вам потрібно використовувати OpenAI-сумісну кінцеву точку, дивіться розділ "Legacy OpenAI-compatible mode" вище. У цьому режимі потокова передача та виклик інструментів можуть не працювати одночасно.
    </Tip>

  </Accordion>
</AccordionGroup>

## Усунення несправностей

<AccordionGroup>
  <Accordion title="Ollama не виявлено">
    Переконайтеся, що Ollama запущено, що ви встановили `OLLAMA_API_KEY` (або профіль автентифікації) і що ви **не** визначили явний запис `models.providers.ollama`:

    ```bash
    ollama serve
    ```

    Перевірте, що API доступний:

    ```bash
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Немає доступних моделей">
    Якщо вашу модель не вказано у списку, або завантажте її локально, або визначте її явно в `models.providers.ollama`.

    ```bash
    ollama list  # Подивитися, що встановлено
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Або іншу модель
    ```

  </Accordion>

  <Accordion title="У з’єднанні відмовлено">
    Переконайтеся, що Ollama працює на правильному порту:

    ```bash
    # Перевірити, чи запущено Ollama
    ps aux | grep ollama

    # Або перезапустити Ollama
    ollama serve
    ```

  </Accordion>
</AccordionGroup>

<Note>
Більше довідки: [Усунення несправностей](/uk/help/troubleshooting) і [FAQ](/uk/help/faq).
</Note>

## Пов’язане

<CardGroup cols={2}>
  <Card title="Постачальники моделей" href="/uk/concepts/model-providers" icon="layers">
    Огляд усіх постачальників, посилань на моделі та поведінки перемикання при відмові.
  </Card>
  <Card title="Вибір моделі" href="/uk/concepts/models" icon="brain">
    Як вибирати та налаштовувати моделі.
  </Card>
  <Card title="Ollama Web Search" href="/uk/tools/ollama-search" icon="magnifying-glass">
    Повні відомості про налаштування та поведінку вебпошуку на основі Ollama.
  </Card>
  <Card title="Конфігурація" href="/uk/gateway/configuration" icon="gear">
    Повний довідник із конфігурації.
  </Card>
</CardGroup>
