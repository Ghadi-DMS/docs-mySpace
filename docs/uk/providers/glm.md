---
read_when:
    - Ви хочете використовувати моделі GLM в OpenClaw
    - Вам потрібні правила іменування моделей і налаштування
summary: Огляд сімейства моделей GLM + як використовувати його в OpenClaw
title: Моделі GLM
x-i18n:
    generated_at: "2026-04-08T03:40:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 79a55acfa139847b4b85dbc09f1068cbd2febb1e49f984a23ea9e3b43bc910eb
    source_path: providers/glm.md
    workflow: 15
---

# Моделі GLM

GLM — це **сімейство моделей** (а не компанія), доступне через платформу Z.AI. В OpenClaw моделі
GLM доступні через провайдера `zai` та ідентифікатори моделей на кшталт `zai/glm-5`.

## Налаштування CLI

```bash
# Загальне налаштування API-ключа з автоматичним визначенням кінцевої точки
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global, рекомендовано для користувачів Coding Plan
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (регіон Китай), рекомендовано для користувачів Coding Plan
openclaw onboard --auth-choice zai-coding-cn

# Загальний API
openclaw onboard --auth-choice zai-global

# Загальний API CN (регіон Китай)
openclaw onboard --auth-choice zai-cn
```

## Фрагмент конфігурації

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5.1" } } },
}
```

`zai-api-key` дає OpenClaw змогу визначити відповідну кінцеву точку Z.AI за ключем і
автоматично застосувати правильну базову URL-адресу. Використовуйте явні регіональні варіанти, якщо
хочете примусово вибрати конкретний Coding Plan або загальний API-інтерфейс.

## Поточні вбудовані моделі GLM

Наразі OpenClaw заповнює вбудованого провайдера `zai` такими посиланнями GLM:

- `glm-5.1`
- `glm-5`
- `glm-5-turbo`
- `glm-5v-turbo`
- `glm-4.7`
- `glm-4.7-flash`
- `glm-4.7-flashx`
- `glm-4.6`
- `glm-4.6v`
- `glm-4.5`
- `glm-4.5-air`
- `glm-4.5-flash`
- `glm-4.5v`

## Примітки

- Версії GLM і їх доступність можуть змінюватися; перевіряйте документацію Z.AI, щоб дізнатися про найновіші дані.
- Типове вбудоване посилання на модель — `zai/glm-5.1`.
- Докладніше про провайдера див. у [/providers/zai](/uk/providers/zai).
