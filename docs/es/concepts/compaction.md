---
read_when:
    - Quieres entender la compactación automática y `/compact`
    - Estás depurando sesiones largas que alcanzan límites de contexto
summary: Cómo OpenClaw resume conversaciones largas para mantenerse dentro de los límites del modelo
title: Compactación
x-i18n:
    generated_at: "2026-04-05T12:39:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4c6dbd6ebdcd5f918805aafdc153925efef3e130faa3fab3c630832e938219fc
    source_path: concepts/compaction.md
    workflow: 15
---

# Compactación

Cada modelo tiene una ventana de contexto, el número máximo de tokens que puede procesar.
Cuando una conversación se acerca a ese límite, OpenClaw **compacta** los mensajes más antiguos
en un resumen para que el chat pueda continuar.

## Cómo funciona

1. Los turnos más antiguos de la conversación se resumen en una entrada compacta.
2. El resumen se guarda en la transcripción de la sesión.
3. Los mensajes recientes se mantienen intactos.

Cuando OpenClaw divide el historial en fragmentos de compactación, mantiene las
llamadas de herramientas del asistente emparejadas con sus entradas `toolResult`
correspondientes. Si un punto de división cae dentro de un bloque de herramientas,
OpenClaw mueve el límite para que la pareja permanezca unida y se preserve la cola
actual sin resumir.

El historial completo de la conversación permanece en disco. La compactación solo cambia lo que el
modelo ve en el siguiente turno.

## Compactación automática

La compactación automática está activada de forma predeterminada. Se ejecuta cuando la sesión se acerca al límite
de contexto, o cuando el modelo devuelve un error de desbordamiento de contexto (en cuyo caso
OpenClaw compacta y vuelve a intentarlo). Las firmas típicas de desbordamiento incluyen
`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model` y `ollama error: context length
exceeded`.

<Info>
Antes de compactar, OpenClaw recuerda automáticamente al agente que guarde notas importantes
en archivos de [memory](/concepts/memory). Esto evita la pérdida de contexto.
</Info>

## Compactación manual

Escribe `/compact` en cualquier chat para forzar una compactación. Añade instrucciones para orientar
el resumen:

```
/compact Focus on the API design decisions
```

## Usar un modelo diferente

De forma predeterminada, la compactación usa el modelo principal de tu agente. Puedes usar un modelo más
capaz para obtener mejores resúmenes:

```json5
{
  agents: {
    defaults: {
      compaction: {
        model: "openrouter/anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

## Aviso de inicio de compactación

De forma predeterminada, la compactación se ejecuta silenciosamente. Para mostrar un aviso breve cuando la compactación
comienza, habilita `notifyUser`:

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

Cuando está habilitado, el usuario ve un mensaje corto (por ejemplo, "Compactando
contexto...") al inicio de cada ejecución de compactación.

## Compactación frente a depuración

|                  | Compactación                  | Depuración                       |
| ---------------- | ----------------------------- | -------------------------------- |
| **Qué hace**     | Resume conversaciones antiguas | Recorta resultados de herramientas antiguos |
| **¿Se guarda?**  | Sí (en la transcripción de la sesión) | No (solo en memoria, por solicitud) |
| **Alcance**      | Conversación completa         | Solo resultados de herramientas  |

La [depuración de sesión](/concepts/session-pruning) es un complemento más ligero que
recorta la salida de herramientas sin resumir.

## Resolución de problemas

**¿Compacta demasiado a menudo?** La ventana de contexto del modelo puede ser pequeña, o las
salidas de herramientas pueden ser grandes. Intenta habilitar la
[depuración de sesión](/concepts/session-pruning).

**¿El contexto parece obsoleto después de compactar?** Usa `/compact Focus on <topic>` para
orientar el resumen, o habilita el [vaciado de memory](/concepts/memory) para que las notas
se conserven.

**¿Necesitas empezar de cero?** `/new` inicia una sesión nueva sin compactar.

Para la configuración avanzada (reserva de tokens, preservación de identificadores, motores de
contexto personalizados, compactación del lado del servidor de OpenAI), consulta la
[Guía detallada de gestión de sesiones](/reference/session-management-compaction).

## Relacionado

- [Sesión](/concepts/session) — gestión y ciclo de vida de la sesión
- [Depuración de sesión](/concepts/session-pruning) — recorte de resultados de herramientas
- [Contexto](/concepts/context) — cómo se construye el contexto para los turnos del agente
- [Hooks](/automation/hooks) — hooks del ciclo de vida de compactación (`before_compaction`, `after_compaction`)
