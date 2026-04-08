---
read_when:
    - Ви хочете використовувати Z.AI / моделі GLM в OpenClaw
    - Вам потрібне просте налаштування ZAI_API_KEY
summary: Використовуйте Z.AI (моделі GLM) з OpenClaw
title: Z.AI
x-i18n:
    generated_at: "2026-04-08T03:40:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66cbd9813ee28d202dcae34debab1b0cf9927793acb00743c1c62b48d9e381f9
    source_path: providers/zai.md
    workflow: 15
---

# Z.AI

Z.AI — це API-платформа для моделей **GLM**. Вона надає REST API для GLM і використовує API-ключі
для автентифікації. Створіть свій API-ключ у консолі Z.AI. OpenClaw використовує провайдер `zai`
із API-ключем Z.AI.

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
автоматично застосувати правильний базовий URL. Використовуйте явні регіональні варіанти, якщо
ви хочете примусово вибрати певний Coding Plan або загальний API-інтерфейс.

## Вбудований каталог GLM

Наразі OpenClaw попередньо заповнює вбудований провайдер `zai` такими моделями:

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

- Моделі GLM доступні як `zai/<model>` (приклад: `zai/glm-5`).
- Базове посилання на вбудовану модель: `zai/glm-5.1`
- Невідомі ідентифікатори `glm-5*` все одно розв’язуються через шлях вбудованого провайдера
  шляхом синтезу метаданих провайдера на основі шаблону `glm-4.7`, коли ідентифікатор
  відповідає поточній формі сімейства GLM-5.
- `tool_stream` увімкнено за замовчуванням для потокової передачі викликів інструментів у Z.AI. Установіть
  `agents.defaults.models["zai/<model>"].params.tool_stream` в `false`, щоб вимкнути це.
- Див. [/providers/glm](/uk/providers/glm), щоб ознайомитися з оглядом сімейства моделей.
- Z.AI використовує Bearer-автентифікацію з вашим API-ключем.
