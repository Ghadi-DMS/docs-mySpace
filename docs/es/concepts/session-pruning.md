---
read_when:
    - Quieres reducir el crecimiento del contexto por salidas de herramientas
    - Quieres entender la optimización de la caché de prompts de Anthropic
summary: Recorte de resultados antiguos de herramientas para mantener un contexto ligero y un almacenamiento en caché eficiente
title: Depuración de sesión
x-i18n:
    generated_at: "2026-04-05T12:40:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1569a50e0018cca3e3ceefbdddaf093843df50cdf2f7bf62fe925299875cb487
    source_path: concepts/session-pruning.md
    workflow: 15
---

# Depuración de sesión

La depuración de sesión recorta **resultados antiguos de herramientas** del contexto antes de cada
llamada al LLM. Reduce el crecimiento del contexto por salidas acumuladas de herramientas (resultados de exec, lecturas de archivos, resultados de búsqueda) sin reescribir el texto normal de la conversación.

<Info>
La depuración es solo en memoria: no modifica la transcripción de la sesión en disco.
Tu historial completo siempre se conserva.
</Info>

## Por qué importa

Las sesiones largas acumulan salida de herramientas que infla la ventana de contexto. Esto
aumenta el costo y puede forzar la [compactación](/concepts/compaction) antes de
lo necesario.

La depuración es especialmente valiosa para el **almacenamiento en caché de prompts de Anthropic**. Después de que expire el
TTL de la caché, la siguiente solicitud vuelve a almacenar en caché el prompt completo. La depuración reduce el
tamaño de escritura en caché, lo que disminuye directamente el costo.

## Cómo funciona

1. Espera a que expire el TTL de la caché (predeterminado 5 minutos).
2. Encuentra resultados antiguos de herramientas para la depuración normal (el texto de conversación se deja intacto).
3. **Recorte suave** de resultados sobredimensionados: conserva el principio y el final, e inserta `...`.
4. **Borrado duro** del resto: lo reemplaza con un marcador.
5. Restablece el TTL para que las solicitudes de seguimiento reutilicen la caché nueva.

## Limpieza heredada de imágenes

OpenClaw también ejecuta una limpieza idempotente separada para sesiones heredadas antiguas que
persistían bloques de imagen sin procesar en el historial.

- Conserva los **3 turnos completados más recientes** byte por byte para que los
  prefijos de caché de prompts para seguimientos recientes sigan siendo estables.
- Los bloques de imagen antiguos ya procesados en el historial `user` o `toolResult` pueden
  reemplazarse por `[image data removed - already processed by model]`.
- Esto es independiente de la depuración normal por TTL de caché. Existe para evitar que cargas útiles
  de imagen repetidas invaliden las cachés de prompts en turnos posteriores.

## Valores predeterminados inteligentes

OpenClaw habilita automáticamente la depuración para perfiles de Anthropic:

| Tipo de perfil                                           | Depuración habilitada | Heartbeat |
| -------------------------------------------------------- | --------------------- | --------- |
| Autenticación OAuth/token de Anthropic (incluida la reutilización de Claude CLI) | Sí                    | 1 hora    |
| API key                                                  | Sí                    | 30 min    |

Si estableces valores explícitos, OpenClaw no los sobrescribe.

## Habilitar o deshabilitar

La depuración está desactivada de forma predeterminada para proveedores que no son Anthropic. Para habilitarla:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

Para deshabilitarla: establece `mode: "off"`.

## Depuración frente a compactación

|            | Depuración          | Compactación            |
| ---------- | ------------------- | ----------------------- |
| **Qué**    | Recorta resultados de herramientas | Resume la conversación |
| **¿Se guarda?** | No (por solicitud) | Sí (en la transcripción) |
| **Alcance** | Solo resultados de herramientas | Conversación completa   |

Se complementan entre sí: la depuración mantiene la salida de herramientas ligera entre
ciclos de compactación.

## Más información

- [Compactación](/concepts/compaction): reducción de contexto basada en resúmenes
- [Configuración del Gateway](/gateway/configuration): todas las opciones de configuración de depuración
  (`contextPruning.*`)
