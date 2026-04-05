---
read_when:
    - Trabajas con reacciones en cualquier canal
    - Entender cómo difieren las reacciones emoji entre plataformas
summary: Semántica de la herramienta de reacciones en todos los canales compatibles
title: Reacciones
x-i18n:
    generated_at: "2026-04-05T12:56:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9af2951eee32e73adb982dbdf39b32e4065993454e9cce2ad23b27565cab4f84
    source_path: tools/reactions.md
    workflow: 15
---

# Reacciones

El agente puede añadir y eliminar reacciones emoji en mensajes usando la herramienta `message`
con la acción `react`. El comportamiento de las reacciones varía según el canal.

## Cómo funciona

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- `emoji` es obligatorio al añadir una reacción.
- Establece `emoji` como una cadena vacía (`""`) para eliminar la(s) reacción(es) del bot.
- Establece `remove: true` para eliminar un emoji específico (requiere `emoji` no vacío).

## Comportamiento por canal

<AccordionGroup>
  <Accordion title="Discord and Slack">
    - Un `emoji` vacío elimina todas las reacciones del bot en el mensaje.
    - `remove: true` elimina solo el emoji especificado.
  </Accordion>

  <Accordion title="Google Chat">
    - Un `emoji` vacío elimina las reacciones de la app en el mensaje.
    - `remove: true` elimina solo el emoji especificado.
  </Accordion>

  <Accordion title="Telegram">
    - Un `emoji` vacío elimina las reacciones del bot.
    - `remove: true` también elimina reacciones, pero sigue requiriendo un `emoji` no vacío para la validación de la herramienta.
  </Accordion>

  <Accordion title="WhatsApp">
    - Un `emoji` vacío elimina la reacción del bot.
    - `remove: true` se asigna internamente a emoji vacío (sigue requiriendo `emoji` en la llamada a la herramienta).
  </Accordion>

  <Accordion title="Zalo Personal (zalouser)">
    - Requiere un `emoji` no vacío.
    - `remove: true` elimina esa reacción emoji específica.
  </Accordion>

  <Accordion title="Feishu/Lark">
    - Usa la herramienta `feishu_reaction` con las acciones `add`, `remove` y `list`.
    - Añadir/eliminar requiere `emoji_type`; eliminar también requiere `reaction_id`.
  </Accordion>

  <Accordion title="Signal">
    - Las notificaciones de reacciones entrantes se controlan con `channels.signal.reactionNotifications`: `"off"` las desactiva, `"own"` (predeterminado) emite eventos cuando los usuarios reaccionan a mensajes del bot, y `"all"` emite eventos para todas las reacciones.
  </Accordion>
</AccordionGroup>

## Nivel de reacción

La configuración `reactionLevel` por canal controla hasta qué punto el agente usa reacciones. Los valores suelen ser `off`, `ack`, `minimal` o `extensive`.

- [Telegram reactionLevel](/es/channels/telegram#reaction-notifications) — `channels.telegram.reactionLevel`
- [WhatsApp reactionLevel](/es/channels/whatsapp#reactions) — `channels.whatsapp.reactionLevel`

Establece `reactionLevel` en canales individuales para ajustar con qué grado de actividad reacciona el agente a los mensajes en cada plataforma.

## Relacionado

- [Agent Send](/tools/agent-send) — la herramienta `message` que incluye `react`
- [Channels](/es/channels) — configuración específica por canal
