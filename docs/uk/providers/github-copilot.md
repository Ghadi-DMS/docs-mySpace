---
read_when:
    - Ви хочете використовувати GitHub Copilot як постачальника моделей
    - Вам потрібен процес `openclaw models auth login-github-copilot`
summary: Увійдіть у GitHub Copilot з OpenClaw за допомогою device flow
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-12T11:08:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 51fee006e7d4e78e37b0c29356b0090b132de727d99b603441767d3fb642140b
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

GitHub Copilot — це помічник для програмування на базі ШІ від GitHub. Він надає доступ до моделей Copilot для вашого облікового запису та плану GitHub. OpenClaw може використовувати Copilot як постачальника моделей двома різними способами.

## Два способи використання Copilot в OpenClaw

<Tabs>
  <Tab title="Вбудований постачальник (github-copilot)">
    Використовуйте нативний процес входу через device flow, щоб отримати токен GitHub, а потім обмінювати його на API-токени Copilot під час роботи OpenClaw. Це **типовий** і найпростіший шлях, оскільки він не потребує VS Code.

    <Steps>
      <Step title="Запустіть команду входу">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Вам буде запропоновано перейти за URL-адресою та ввести одноразовий код. Тримайте
        термінал відкритим, доки процес не завершиться.
      </Step>
      <Step title="Установіть типову модель">
        ```bash
        openclaw models set github-copilot/gpt-4o
        ```

        Або в конфігурації:

        ```json5
        {
          agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Плагін Copilot Proxy (copilot-proxy)">
    Використовуйте розширення VS Code **Copilot Proxy** як локальний міст. OpenClaw звертається до
    кінцевої точки `/v1` проксі та використовує список моделей, який ви там налаштуєте.

    <Note>
    Виберіть це, якщо ви вже запускаєте Copilot Proxy у VS Code або вам потрібно маршрутизувати
    через нього. Ви повинні ввімкнути Plugin і підтримувати роботу розширення VS Code.
    </Note>

  </Tab>
</Tabs>

## Необов’язкові прапорці

| Flag            | Опис                                                   |
| --------------- | ------------------------------------------------------ |
| `--yes`         | Пропустити запит на підтвердження                      |
| `--set-default` | Також застосувати рекомендовану типову модель постачальника |

```bash
# Пропустити підтвердження
openclaw models auth login-github-copilot --yes

# Увійти та встановити типову модель за один крок
openclaw models auth login --provider github-copilot --method device --set-default
```

<AccordionGroup>
  <Accordion title="Потрібен інтерактивний TTY">
    Для процесу входу через device flow потрібен інтерактивний TTY. Запускайте його безпосередньо в
    терміналі, а не в неінтерактивному скрипті чи конвеєрі CI.
  </Accordion>

  <Accordion title="Доступність моделей залежить від вашого плану">
    Доступність моделей Copilot залежить від вашого плану GitHub. Якщо модель
    відхиляється, спробуйте інший ID (наприклад, `github-copilot/gpt-4.1`).
  </Accordion>

  <Accordion title="Вибір транспорту">
    Ідентифікатори моделей Claude автоматично використовують транспорт Anthropic Messages. GPT,
    моделі серії o та Gemini зберігають транспорт OpenAI Responses. OpenClaw
    вибирає правильний транспорт на основі посилання на модель.
  </Accordion>

  <Accordion title="Порядок пріоритету змінних середовища">
    OpenClaw визначає автентифікацію Copilot зі змінних середовища в такому
    порядку пріоритету:

    | Priority | Variable              | Notes                              |
    | -------- | --------------------- | ---------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN` | Найвищий пріоритет, специфічний для Copilot |
    | 2        | `GH_TOKEN`            | Токен GitHub CLI (резервний варіант) |
    | 3        | `GITHUB_TOKEN`        | Стандартний токен GitHub (найнижчий пріоритет) |

    Коли задано кілька змінних, OpenClaw використовує змінну з найвищим пріоритетом.
    Процес входу через device flow (`openclaw models auth login-github-copilot`) зберігає
    свій токен у сховищі профілів автентифікації та має пріоритет над усіма змінними
    середовища.

  </Accordion>

  <Accordion title="Зберігання токенів">
    Під час входу токен GitHub зберігається у сховищі профілів автентифікації та обмінюється
    на API-токен Copilot під час роботи OpenClaw. Вам не потрібно керувати
    токеном вручну.
  </Accordion>
</AccordionGroup>

<Warning>
Потрібен інтерактивний TTY. Запускайте команду входу безпосередньо в терміналі, а не
всередині headless-скрипту чи завдання CI.
</Warning>

## Пов’язане

<CardGroup cols={2}>
  <Card title="Вибір моделі" href="/uk/concepts/model-providers" icon="layers">
    Вибір постачальників, посилань на моделі та поведінки перемикання на резервний варіант.
  </Card>
  <Card title="OAuth і автентифікація" href="/uk/gateway/authentication" icon="key">
    Докладно про автентифікацію та правила повторного використання облікових даних.
  </Card>
</CardGroup>
