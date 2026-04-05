---
read_when:
    - Quieres un paso de LLM solo JSON dentro de flujos de trabajo
    - Necesitas salida de LLM validada por esquema para automatización
summary: Tareas de LLM solo JSON para flujos de trabajo (herramienta opcional de plugin)
title: Tarea de LLM
x-i18n:
    generated_at: "2026-04-05T12:55:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: cbe9b286a8e958494de06a59b6e7b750a82d492158df344c7afe30fce24f0584
    source_path: tools/llm-task.md
    workflow: 15
---

# Tarea de LLM

`llm-task` es una **herramienta opcional de plugin** que ejecuta una tarea de LLM solo JSON y
devuelve una salida estructurada (opcionalmente validada contra JSON Schema).

Esto es ideal para motores de flujos de trabajo como Lobster: puedes agregar un único paso de LLM
sin escribir código personalizado de OpenClaw para cada flujo de trabajo.

## Habilitar el plugin

1. Habilita el plugin:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. Añade la herramienta a la allowlist (se registra con `optional: true`):

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

## Configuración (opcional)

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai-codex",
          "defaultModel": "gpt-5.4",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai-codex/gpt-5.4"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

`allowedModels` es una allowlist de cadenas `provider/model`. Si está configurada, cualquier solicitud
fuera de la lista se rechaza.

## Parámetros de la herramienta

- `prompt` (string, obligatorio)
- `input` (any, opcional)
- `schema` (object, JSON Schema opcional)
- `provider` (string, opcional)
- `model` (string, opcional)
- `thinking` (string, opcional)
- `authProfileId` (string, opcional)
- `temperature` (number, opcional)
- `maxTokens` (number, opcional)
- `timeoutMs` (number, opcional)

`thinking` acepta los ajustes predeterminados estándar de razonamiento de OpenClaw, como `low` o `medium`.

## Salida

Devuelve `details.json`, que contiene el JSON analizado (y lo valida contra
`schema` cuando se proporciona).

## Ejemplo: paso de flujo de trabajo de Lobster

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "thinking": "low",
  "input": {
    "subject": "Hello",
    "body": "Can you help?"
  },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

## Notas de seguridad

- La herramienta es **solo JSON** e instruye al modelo para que genere únicamente JSON (sin
  bloques de código ni comentarios).
- No se exponen herramientas al modelo para esta ejecución.
- Trata la salida como no confiable a menos que la valides con `schema`.
- Coloca aprobaciones antes de cualquier paso con efectos secundarios (enviar, publicar, ejecutar).
