---
read_when:
    - Ви хочете використовувати генерацію зображень fal в OpenClaw
    - Вам потрібен потік автентифікації `FAL_KEY`
    - Вам потрібні типові налаштування fal для `image_generate` або `video_generate`
summary: налаштування генерації зображень і відео fal в OpenClaw
title: fal
x-i18n:
    generated_at: "2026-04-11T01:39:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6865c3110cbb9d63f4ecec4467f89a801478759cd7a2f3efa83e8f7ce1dc3e67
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw постачається з вбудованим провайдером `fal` для хостингової генерації зображень і відео.

- Провайдер: `fal`
- Автентифікація: `FAL_KEY` (канонічний варіант; `FAL_API_KEY` також працює як запасний)
- API: кінцеві точки моделей fal

## Швидкий старт

1. Установіть API-ключ:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. Установіть типову модель зображень:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Генерація зображень

Вбудований провайдер генерації зображень `fal` типово використовує
`fal/fal-ai/flux/dev`.

- Генерація: до 4 зображень за запит
- Режим редагування: увімкнено, 1 еталонне зображення
- Підтримує `size`, `aspectRatio` і `resolution`
- Поточне обмеження редагування: кінцева точка редагування зображень fal **не** підтримує
  перевизначення `aspectRatio`

Щоб використовувати fal як типовий провайдер зображень:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Генерація відео

Вбудований провайдер генерації відео `fal` типово використовує
`fal/fal-ai/minimax/video-01-live`.

- Режими: перетворення тексту на відео та потоки з одним еталонним зображенням
- Середовище виконання: потік submit/status/result на основі черги для довготривалих завдань
- Посилання на моделі Seedance 2.0:
  - `fal/bytedance/seedance-2.0/fast/text-to-video`
  - `fal/bytedance/seedance-2.0/fast/image-to-video`
  - `fal/bytedance/seedance-2.0/text-to-video`
  - `fal/bytedance/seedance-2.0/image-to-video`

Щоб використовувати Seedance 2.0 як типову модель відео:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
      },
    },
  },
}
```

## Пов’язане

- [Генерація зображень](/uk/tools/image-generation)
- [Генерація відео](/uk/tools/video-generation)
- [Довідник із конфігурації](/uk/gateway/configuration-reference#agent-defaults)
