---
read_when:
    - Agregar o modificar el análisis de ubicación del canal
    - Usar campos de contexto de ubicación en prompts o herramientas del agente
summary: Análisis de ubicación entrante del canal (Telegram/WhatsApp/Matrix) y campos de contexto
title: Análisis de ubicación del canal
x-i18n:
    generated_at: "2026-04-05T12:35:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 10061f0c109240a9e0bcab649b17f03b674e8bdf410debf3669b7b6da8189d96
    source_path: channels/location.md
    workflow: 15
---

# Análisis de ubicación del canal

OpenClaw normaliza las ubicaciones compartidas desde los canales de chat en:

- texto legible para humanos añadido al cuerpo entrante, y
- campos estructurados en la carga útil del contexto de respuesta automática.

Actualmente compatible con:

- **Telegram** (pines de ubicación + lugares + ubicaciones en vivo)
- **WhatsApp** (`locationMessage` + `liveLocationMessage`)
- **Matrix** (`m.location` con `geo_uri`)

## Formato de texto

Las ubicaciones se representan como líneas claras sin corchetes:

- Pin:
  - `📍 48.858844, 2.294351 ±12m`
- Lugar con nombre:
  - `📍 Eiffel Tower — Champ de Mars, Paris (48.858844, 2.294351 ±12m)`
- Ubicación compartida en vivo:
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

Si el canal incluye una leyenda/comentario, se añade en la línea siguiente:

```
📍 48.858844, 2.294351 ±12m
Meet here
```

## Campos de contexto

Cuando hay una ubicación presente, estos campos se añaden a `ctx`:

- `LocationLat` (número)
- `LocationLon` (número)
- `LocationAccuracy` (número, metros; opcional)
- `LocationName` (cadena; opcional)
- `LocationAddress` (cadena; opcional)
- `LocationSource` (`pin | place | live`)
- `LocationIsLive` (booleano)

## Notas por canal

- **Telegram**: los lugares se asignan a `LocationName/LocationAddress`; las ubicaciones en vivo usan `live_period`.
- **WhatsApp**: `locationMessage.comment` y `liveLocationMessage.caption` se añaden como la línea de leyenda.
- **Matrix**: `geo_uri` se analiza como una ubicación de pin; la altitud se ignora y `LocationIsLive` siempre es false.
