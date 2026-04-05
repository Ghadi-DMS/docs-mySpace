---
read_when:
    - Cambiar el comportamiento o los valores predeterminados de los indicadores de escritura
summary: Cuándo OpenClaw muestra indicadores de escritura y cómo ajustarlos
title: Indicadores de escritura
x-i18n:
    generated_at: "2026-04-05T12:40:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28c8c395a135fc0745181aab66a93582177e6acd0b3496debcbb98159a4f11dc
    source_path: concepts/typing-indicators.md
    workflow: 15
---

# Indicadores de escritura

Los indicadores de escritura se envían al canal de chat mientras una ejecución está activa. Usa
`agents.defaults.typingMode` para controlar **cuándo** comienza la escritura y `typingIntervalSeconds`
para controlar **con qué frecuencia** se actualiza.

## Valores predeterminados

Cuando `agents.defaults.typingMode` **no está configurado**, OpenClaw mantiene el comportamiento heredado:

- **Chats directos**: la escritura comienza inmediatamente una vez que empieza el bucle del modelo.
- **Chats de grupo con una mención**: la escritura comienza inmediatamente.
- **Chats de grupo sin una mención**: la escritura comienza solo cuando el texto del mensaje empieza a transmitirse.
- **Ejecuciones de heartbeat**: la escritura está deshabilitada.

## Modos

Establece `agents.defaults.typingMode` en uno de estos valores:

- `never` — nunca se muestra el indicador de escritura.
- `instant` — inicia la escritura **en cuanto comienza el bucle del modelo**, aunque la ejecución
  más tarde devuelva solo el token de respuesta silenciosa.
- `thinking` — inicia la escritura en el **primer delta de razonamiento** (requiere
  `reasoningLevel: "stream"` para la ejecución).
- `message` — inicia la escritura en el **primer delta de texto no silencioso** (ignora
  el token silencioso `NO_REPLY`).

Orden de “qué tan pronto se activa”:
`never` → `message` → `thinking` → `instant`

## Configuración

```json5
{
  agent: {
    typingMode: "thinking",
    typingIntervalSeconds: 6,
  },
}
```

Puedes reemplazar el modo o la cadencia por sesión:

```json5
{
  session: {
    typingMode: "message",
    typingIntervalSeconds: 4,
  },
}
```

## Notas

- El modo `message` no mostrará escritura para respuestas solo silenciosas cuando toda la
  carga sea exactamente el token silencioso (por ejemplo `NO_REPLY` / `no_reply`,
  con coincidencia sin distinguir mayúsculas de minúsculas).
- `thinking` solo se activa si la ejecución transmite razonamiento (`reasoningLevel: "stream"`).
  Si el modelo no emite deltas de razonamiento, la escritura no comenzará.
- Los heartbeats nunca muestran escritura, independientemente del modo.
- `typingIntervalSeconds` controla la **cadencia de actualización**, no el momento de inicio.
  El valor predeterminado es 6 segundos.
