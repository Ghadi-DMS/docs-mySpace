---
read_when:
    - Quieres entender la compactación automática y /compact
    - Estás depurando sesiones largas que alcanzan los límites de contexto
summary: Cómo OpenClaw resume conversaciones largas para mantenerse dentro de los límites del modelo
title: Compactación
x-i18n:
    generated_at: "2026-04-08T02:14:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: e6590b82a8c3a9c310998d653459ca4d8612495703ca0a8d8d306d7643142fd1
    source_path: concepts/compaction.md
    workflow: 15
---

# Compactación

Todo modelo tiene una ventana de contexto: la cantidad máxima de tokens que puede procesar.
Cuando una conversación se acerca a ese límite, OpenClaw **compacta** los mensajes
más antiguos en un resumen para que el chat pueda continuar.

## Cómo funciona

1. Los turnos más antiguos de la conversación se resumen en una entrada compacta.
2. El resumen se guarda en la transcripción de la sesión.
3. Los mensajes recientes se mantienen intactos.

Cuando OpenClaw divide el historial en fragmentos de compactación, mantiene las
llamadas de herramientas del asistente emparejadas con sus entradas `toolResult`
correspondientes. Si un punto de división cae dentro de un bloque de herramientas,
OpenClaw mueve el límite para que el par permanezca junto y se conserve la cola
actual no resumida.

El historial completo de la conversación permanece en el disco. La compactación solo cambia lo que el
modelo ve en el siguiente turno.

## Compactación automática

La compactación automática está activada de forma predeterminada. Se ejecuta cuando la sesión se acerca al
límite de contexto, o cuando el modelo devuelve un error de desbordamiento de contexto (en cuyo caso
OpenClaw compacta y reintenta). Las firmas típicas de desbordamiento incluyen
`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model` y `ollama error: context length
exceeded`.

<Info>
Antes de compactar, OpenClaw le recuerda automáticamente al agente que guarde notas importantes
en archivos de [memory](/es/concepts/memory). Esto evita la pérdida de contexto.
</Info>

Usa la configuración `agents.defaults.compaction` en tu `openclaw.json` para configurar el comportamiento de la compactación (modo, tokens objetivo, etc.).
La resumización de la compactación conserva los identificadores opacos de forma predeterminada (`identifierPolicy: "strict"`). Puedes cambiar esto con `identifierPolicy: "off"` o proporcionar texto personalizado con `identifierPolicy: "custom"` e `identifierInstructions`.

También puedes especificar opcionalmente un modelo diferente para la resumización de la compactación mediante `agents.defaults.compaction.model`. Esto resulta útil cuando tu modelo principal es un modelo local o pequeño y quieres que los resúmenes de compactación los produzca un modelo más capaz. La anulación acepta cualquier cadena `provider/model-id`:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Esto también funciona con modelos locales, por ejemplo, un segundo modelo de Ollama dedicado a la resumización o un especialista de compactación ajustado finamente:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Cuando no se establece, la compactación usa el modelo principal del agente.

## Proveedores de compactación conectables

Los plugins pueden registrar un proveedor de compactación personalizado mediante `registerCompactionProvider()` en la API del plugin. Cuando se registra y configura un proveedor, OpenClaw le delega la resumización en lugar de usar el flujo integrado de LLM.

Para usar un proveedor registrado, establece el id del proveedor en tu configuración:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Establecer un `provider` fuerza automáticamente `mode: "safeguard"`. Los proveedores reciben las mismas instrucciones de compactación y la misma política de conservación de identificadores que la ruta integrada, y OpenClaw sigue conservando el contexto de sufijo de turnos recientes y turnos divididos después de la salida del proveedor. Si el proveedor falla o devuelve un resultado vacío, OpenClaw vuelve a la resumización integrada por LLM.

## Compactación automática (activada de forma predeterminada)

Cuando una sesión se acerca o supera la ventana de contexto del modelo, OpenClaw activa la compactación automática y puede reintentar la solicitud original usando el contexto compactado.

Verás:

- `🧹 Auto-compaction complete` en modo detallado
- `/status` mostrando `🧹 Compactions: <count>`

Antes de la compactación, OpenClaw puede ejecutar un turno de **vaciado silencioso de memory** para almacenar
notas persistentes en el disco. Consulta [Memory](/es/concepts/memory) para conocer los detalles y la configuración.

## Compactación manual

Escribe `/compact` en cualquier chat para forzar una compactación. Agrega instrucciones para orientar
el resumen:

```
/compact Focus on the API design decisions
```

## Uso de un modelo diferente

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
comience, habilita `notifyUser`:

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

Cuando está habilitado, el usuario ve un mensaje corto (por ejemplo, "Compacting
context...") al inicio de cada ejecución de compactación.

## Compactación frente a poda

|                  | Compactación                  | Poda                             |
| ---------------- | ----------------------------- | -------------------------------- |
| **Qué hace**     | Resume la conversación antigua | Recorta resultados de herramientas antiguos |
| **¿Se guarda?**  | Sí (en la transcripción de la sesión) | No (solo en memoria, por solicitud) |
| **Alcance**      | Conversación completa         | Solo resultados de herramientas  |

La [poda de sesión](/es/concepts/session-pruning) es un complemento más ligero que
recorta la salida de las herramientas sin resumirla.

## Solución de problemas

**¿Compacta demasiado a menudo?** La ventana de contexto del modelo puede ser pequeña, o las salidas de las herramientas
pueden ser grandes. Prueba a habilitar la
[poda de sesión](/es/concepts/session-pruning).

**¿El contexto se siente obsoleto después de la compactación?** Usa `/compact Focus on <topic>` para
orientar el resumen, o habilita el [vaciado de memory](/es/concepts/memory) para que las notas
se conserven.

**¿Necesitas empezar desde cero?** `/new` inicia una sesión nueva sin compactar.

Para la configuración avanzada (tokens de reserva, conservación de identificadores, motores de
contexto personalizados, compactación del lado del servidor de OpenAI), consulta la
[Guía avanzada de gestión de sesiones](/es/reference/session-management-compaction).

## Relacionado

- [Session](/es/concepts/session) — gestión y ciclo de vida de la sesión
- [Session Pruning](/es/concepts/session-pruning) — recorte de resultados de herramientas
- [Context](/es/concepts/context) — cómo se construye el contexto para los turnos del agente
- [Hooks](/es/automation/hooks) — hooks del ciclo de vida de la compactación (before_compaction, after_compaction)
