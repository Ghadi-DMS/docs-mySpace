---
read_when:
    - Actualizar el comportamiento o los valores predeterminados de reintento del proveedor
    - Depurar errores de envío del proveedor o límites de tasa
summary: Política de reintentos para llamadas salientes al proveedor
title: Política de reintentos
x-i18n:
    generated_at: "2026-04-05T12:40:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 55bb261ff567f46ce447be9c0ee0c5b5e6d2776287d7662762656c14108dd607
    source_path: concepts/retry.md
    workflow: 15
---

# Política de reintentos

## Objetivos

- Reintentar por solicitud HTTP, no por flujo de varios pasos.
- Preservar el orden reintentando solo el paso actual.
- Evitar duplicar operaciones no idempotentes.

## Valores predeterminados

- Intentos: 3
- Límite máximo de retraso: 30000 ms
- Jitter: 0.1 (10 por ciento)
- Valores predeterminados por proveedor:
  - Retraso mínimo de Telegram: 400 ms
  - Retraso mínimo de Discord: 500 ms

## Comportamiento

### Discord

- Reintenta solo en errores de límite de tasa (HTTP 429).
- Usa `retry_after` de Discord cuando está disponible; en caso contrario, usa backoff exponencial.

### Telegram

- Reintenta en errores transitorios (429, timeout, connect/reset/closed, temporalmente no disponible).
- Usa `retry_after` cuando está disponible; en caso contrario, usa backoff exponencial.
- Los errores de análisis de Markdown no se reintentan; recurren a texto sin formato.

## Configuración

Establece la política de reintentos por proveedor en `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    telegram: {
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
    discord: {
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

## Notas

- Los reintentos se aplican por solicitud (envío de mensajes, carga de medios, reacción, encuesta, sticker).
- Los flujos compuestos no reintentan los pasos ya completados.
