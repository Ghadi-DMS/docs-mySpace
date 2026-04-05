---
read_when:
    - Quieres hacer una pregunta lateral rápida sobre la sesión actual
    - Estás implementando o depurando el comportamiento de BTW en varios clientes
summary: Preguntas laterales efímeras con /btw
title: Preguntas laterales con BTW
x-i18n:
    generated_at: "2026-04-05T12:54:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: aeef33ba19eb0561693fecea9dd39d6922df93be0b9a89446ed17277bcee58aa
    source_path: tools/btw.md
    workflow: 15
---

# Preguntas laterales con BTW

`/btw` te permite hacer una pregunta lateral rápida sobre la **sesión actual** sin
convertir esa pregunta en parte del historial normal de la conversación.

Está inspirado en el comportamiento de `/btw` de Claude Code, pero adaptado a la
arquitectura de Gateway y multicanal de OpenClaw.

## Qué hace

Cuando envías:

```text
/btw what changed?
```

OpenClaw:

1. toma una instantánea del contexto de la sesión actual,
2. ejecuta una llamada de modelo independiente **sin herramientas**,
3. responde solo a la pregunta lateral,
4. deja intacta la ejecución principal,
5. **no** escribe la pregunta ni la respuesta de BTW en el historial de la sesión,
6. emite la respuesta como un **resultado lateral en vivo** en lugar de como un mensaje normal del asistente.

El modelo mental importante es:

- mismo contexto de sesión
- consulta lateral independiente de una sola vez
- sin llamadas a herramientas
- sin contaminar el contexto futuro
- sin persistencia en la transcripción

## Qué no hace

`/btw` **no**:

- crea una nueva sesión duradera,
- continúa la tarea principal inacabada,
- ejecuta herramientas ni bucles de herramientas del agente,
- escribe los datos de pregunta/respuesta de BTW en el historial de la transcripción,
- aparece en `chat.history`,
- sobrevive a una recarga.

Es intencionalmente **efímero**.

## Cómo funciona el contexto

BTW usa la sesión actual solo como **contexto de fondo**.

Si la ejecución principal está activa en ese momento, OpenClaw toma una instantánea del
estado actual del mensaje e incluye el prompt principal en curso como contexto de fondo, mientras
indica explícitamente al modelo que:

- responda solo a la pregunta lateral,
- no retome ni complete la tarea principal inacabada,
- no emita llamadas a herramientas ni pseudollamadas a herramientas.

Eso mantiene BTW aislado de la ejecución principal y, al mismo tiempo, le permite saber de
qué trata la sesión.

## Modelo de entrega

BTW **no** se entrega como un mensaje normal del asistente en la transcripción.

En el nivel de protocolo de Gateway:

- el chat normal del asistente usa el evento `chat`
- BTW usa el evento `chat.side_result`

Esta separación es intencional. Si BTW reutilizara la ruta normal del evento `chat`,
los clientes lo tratarían como si fuera historial normal de conversación.

Como BTW usa un evento en vivo separado y no se vuelve a reproducir desde
`chat.history`, desaparece después de recargar.

## Comportamiento por superficie

### TUI

En la TUI, BTW se muestra en línea en la vista de la sesión actual, pero sigue siendo
efímero:

- visiblemente distinto de una respuesta normal del asistente
- descartable con `Enter` o `Esc`
- no se vuelve a reproducir al recargar

### Canales externos

En canales como Telegram, WhatsApp y Discord, BTW se entrega como una
respuesta puntual claramente etiquetada porque esas superficies no tienen un concepto local de
superposición efímera.

La respuesta sigue tratándose como un resultado lateral, no como historial normal de la sesión.

### Control UI / web

Gateway emite BTW correctamente como `chat.side_result`, y BTW no se incluye
en `chat.history`, por lo que el contrato de persistencia ya es correcto para web.

La Control UI actual todavía necesita un consumidor dedicado de `chat.side_result` para
mostrar BTW en vivo en el navegador. Hasta que llegue ese soporte del lado del cliente, BTW es una
función a nivel de Gateway con comportamiento completo en TUI y canales externos, pero aún no
una experiencia completa en el navegador.

## Cuándo usar BTW

Usa `/btw` cuando quieras:

- una aclaración rápida sobre el trabajo actual,
- una respuesta lateral factual mientras una ejecución larga sigue en curso,
- una respuesta temporal que no deba pasar a formar parte del contexto futuro de la sesión.

Ejemplos:

```text
/btw what file are we editing?
/btw what does this error mean?
/btw summarize the current task in one sentence
/btw what is 17 * 19?
```

## Cuándo no usar BTW

No uses `/btw` cuando quieras que la respuesta pase a formar parte del
contexto de trabajo futuro de la sesión.

En ese caso, pregunta normalmente en la sesión principal en lugar de usar BTW.

## Relacionado

- [Comandos con barra](/tools/slash-commands)
- [Niveles de thinking](/tools/thinking)
- [Sesión](/es/concepts/session)
