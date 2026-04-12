---
read_when:
    - Ви хочете використовувати GitHub Copilot як постачальника моделі
    - Вам потрібен потік `openclaw models auth login-github-copilot`
summary: Увійдіть у GitHub Copilot з OpenClaw за допомогою device flow
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-12T10:30:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 147a3c2d37919a36e54fb2739a54500be7ebd34eeed0efe3bde448da7376d541
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

GitHub Copilot — це AI-асистент GitHub для програмування. Він надає доступ до моделей
Copilot для вашого облікового запису GitHub і тарифного плану. OpenClaw може використовувати Copilot як
постачальника моделі двома різними способами.

## Два способи використання Copilot в OpenClaw

<Tabs>
  <Tab title="Вбудований постачальник (github-copilot)">
    Використовуйте нативний потік входу через device-login, щоб отримати токен GitHub, а потім обмінювати його на
    API-токени Copilot під час роботи OpenClaw. Це **типовий** і найпростіший шлях,
    оскільки він не потребує VS Code.

    <Steps>
      <Step title="Запустіть команду входу">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Вам буде запропоновано перейти за URL-адресою та ввести одноразовий код. Не закривайте
        термінал, доки процес не завершиться.
      </Step>
      <Step title="Установіть модель за замовчуванням">
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
    кінцевої точки `/v1` проксі й використовує список моделей, який ви там налаштуєте.

    <Note>
    Обирайте цей варіант, якщо ви вже використовуєте Copilot Proxy у VS Code або вам потрібно маршрутизувати
    через нього. Ви повинні ввімкнути Plugin і тримати розширення VS Code запущеним.
    </Note>

  </Tab>
</Tabs>

## Необов’язкові прапорці

| Flag            | Опис                                                |
| --------------- | --------------------------------------------------- |
| `--yes`         | Пропустити запит підтвердження                      |
| `--set-default` | Також застосувати рекомендовану модель постачальника за замовчуванням |

```bash
# Пропустити підтвердження
openclaw models auth login-github-copilot --yes

# Увійти й одразу встановити модель за замовчуванням
openclaw models auth login --provider github-copilot --method device --set-default
```

<AccordionGroup>
  <Accordion title="Потрібен інтерактивний TTY">
    Потік входу через device-login потребує інтерактивного TTY. Запускайте його безпосередньо в
    терміналі, а не в неінтерактивному скрипті чи CI-конвеєрі.
  </Accordion>

  <Accordion title="Доступність моделей залежить від вашого тарифного плану">
    Доступність моделей Copilot залежить від вашого тарифного плану GitHub. Якщо модель
    відхиляється, спробуйте інший ідентифікатор (наприклад, `github-copilot/gpt-4.1`).
  </Accordion>

  <Accordion title="Вибір транспорту">
    Ідентифікатори моделей Claude автоматично використовують транспорт Anthropic Messages. Моделі GPT,
    o-series і Gemini використовують транспорт OpenAI Responses. OpenClaw
    вибирає правильний транспорт на основі посилання на модель.
  </Accordion>

  <Accordion title="Зберігання токенів">
    Під час входу токен GitHub зберігається в сховищі профілів автентифікації та обмінюється
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
    Вибір постачальників, посилань на моделі та поведінки резервного перемикання.
  </Card>
  <Card title="OAuth і автентифікація" href="/uk/gateway/authentication" icon="key">
    Докладно про автентифікацію та правила повторного використання облікових даних.
  </Card>
</CardGroup>
