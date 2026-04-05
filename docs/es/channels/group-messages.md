---
read_when:
    - Cambiar reglas o menciones de mensajes de grupo
summary: Comportamiento y configuración para el manejo de mensajes de grupo de WhatsApp (mentionPatterns se comparten entre superficies)
title: Mensajes de grupo
x-i18n:
    generated_at: "2026-04-05T12:35:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2543be5bc4c6f188f955df580a6fef585ecbfc1be36ade5d34b1a9157e021bc5
    source_path: channels/group-messages.md
    workflow: 15
---

# Mensajes de grupo (canal web de WhatsApp)

Objetivo: permitir que Clawd esté en grupos de WhatsApp, se active solo cuando lo mencionen y mantenga ese hilo separado de la sesión de mensajes directos personales.

Nota: `agents.list[].groupChat.mentionPatterns` ahora también lo usan Telegram/Discord/Slack/iMessage; este documento se centra en el comportamiento específico de WhatsApp. Para configuraciones con varios agentes, establece `agents.list[].groupChat.mentionPatterns` por agente (o usa `messages.groupChat.mentionPatterns` como respaldo global).

## Implementación actual (2025-12-03)

- Modos de activación: `mention` (predeterminado) o `always`. `mention` requiere una mención (menciones reales de WhatsApp @ mediante `mentionedJids`, patrones regex seguros o el E.164 del bot en cualquier parte del texto). `always` activa el agente en cada mensaje, pero debe responder solo cuando pueda aportar un valor significativo; de lo contrario, devuelve el token silencioso exacto `NO_REPLY` / `no_reply`. Los valores predeterminados pueden establecerse en la configuración (`channels.whatsapp.groups`) y anularse por grupo mediante `/activation`. Cuando se establece `channels.whatsapp.groups`, también actúa como una lista de permitidos de grupos (incluye `"*"` para permitir todos).
- Política de grupo: `channels.whatsapp.groupPolicy` controla si se aceptan mensajes de grupo (`open|disabled|allowlist`). `allowlist` usa `channels.whatsapp.groupAllowFrom` (respaldo: `channels.whatsapp.allowFrom` explícito). El valor predeterminado es `allowlist` (bloqueado hasta que añadas remitentes).
- Sesiones por grupo: las claves de sesión tienen el formato `agent:<agentId>:whatsapp:group:<jid>`, por lo que comandos como `/verbose on` o `/think high` (enviados como mensajes independientes) se aplican solo a ese grupo; el estado del mensaje directo personal no se modifica. Los heartbeat se omiten para los hilos de grupo.
- Inyección de contexto: los mensajes de grupo **solo pendientes** (50 de forma predeterminada) que _no_ activaron una ejecución se anteponen bajo `[Chat messages since your last reply - for context]`, con la línea activadora bajo `[Current message - respond to this]`. Los mensajes ya presentes en la sesión no se vuelven a inyectar.
- Visibilidad del remitente: cada lote de grupo ahora termina con `[from: Sender Name (+E164)]` para que Pi sepa quién está hablando.
- Efímeros/view-once: los desempaquetamos antes de extraer texto/menciones, para que las menciones dentro de ellos sigan activando.
- Prompt del sistema para grupos: en el primer turno de una sesión de grupo (y siempre que `/activation` cambie el modo) inyectamos un breve texto en el prompt del sistema como `You are replying inside the WhatsApp group "<subject>". Group members: Alice (+44...), Bob (+43...), … Activation: trigger-only … Address the specific sender noted in the message context.` Si los metadatos no están disponibles, igualmente indicamos al agente que es un chat de grupo.

## Ejemplo de configuración (WhatsApp)

Añade un bloque `groupChat` a `~/.openclaw/openclaw.json` para que las menciones por nombre para mostrar funcionen incluso cuando WhatsApp elimina la `@` visual en el cuerpo del texto:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          historyLimit: 50,
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    ],
  },
}
```

Notas:

- Las regex no distinguen entre mayúsculas y minúsculas y usan las mismas protecciones de regex seguras que otras superficies de regex de configuración; los patrones no válidos y las repeticiones anidadas inseguras se ignoran.
- WhatsApp sigue enviando menciones canónicas mediante `mentionedJids` cuando alguien toca el contacto, por lo que el respaldo con el número rara vez es necesario, pero es una red de seguridad útil.

### Comando de activación (solo propietario)

Usa el comando de chat de grupo:

- `/activation mention`
- `/activation always`

Solo el número del propietario (de `channels.whatsapp.allowFrom`, o el E.164 del propio bot si no está definido) puede cambiar esto. Envía `/status` como mensaje independiente en el grupo para ver el modo de activación actual.

## Cómo usarlo

1. Añade tu cuenta de WhatsApp (la que ejecuta OpenClaw) al grupo.
2. Di `@openclaw …` (o incluye el número). Solo los remitentes permitidos pueden activarlo a menos que establezcas `groupPolicy: "open"`.
3. El prompt del agente incluirá el contexto reciente del grupo más el marcador final `[from: …]` para que pueda dirigirse a la persona correcta.
4. Las directivas a nivel de sesión (`/verbose on`, `/think high`, `/new` o `/reset`, `/compact`) se aplican solo a la sesión de ese grupo; envíalas como mensajes independientes para que se registren. Tu sesión personal de mensajes directos permanece independiente.

## Pruebas / verificación

- Prueba manual:
  - Envía una mención `@openclaw` en el grupo y confirma una respuesta que haga referencia al nombre del remitente.
  - Envía una segunda mención y verifica que el bloque de historial se incluya y luego se borre en el siguiente turno.
- Revisa los registros del gateway (ejecútalo con `--verbose`) para ver entradas `inbound web message` que muestran `from: <groupJid>` y el sufijo `[from: …]`.

## Consideraciones conocidas

- Los heartbeat se omiten intencionalmente para los grupos a fin de evitar difusiones ruidosas.
- La supresión de eco usa la cadena de lote combinada; si envías el mismo texto dos veces sin menciones, solo la primera recibirá una respuesta.
- Las entradas del almacén de sesiones aparecerán como `agent:<agentId>:whatsapp:group:<jid>` en el almacén de sesiones (`~/.openclaw/agents/<agentId>/sessions/sessions.json` de forma predeterminada); si falta una entrada, solo significa que el grupo aún no ha activado una ejecución.
- Los indicadores de escritura en grupos siguen `agents.defaults.typingMode` (predeterminado: `message` cuando no hay mención).
