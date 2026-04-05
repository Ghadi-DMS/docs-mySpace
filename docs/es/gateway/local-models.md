---
read_when:
    - Quieres servir modelos desde tu propio equipo con GPU
    - Estás conectando LM Studio o un proxy compatible con OpenAI
    - Necesitas la guía más segura para modelos locales
summary: Ejecuta OpenClaw con LLM locales (LM Studio, vLLM, LiteLLM, endpoints personalizados de OpenAI)
title: Modelos locales
x-i18n:
    generated_at: "2026-04-05T12:42:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b99c8fb57f65c0b765fc75bd36933221b5aeb94c4a3f3428f92640ae064f8b6
    source_path: gateway/local-models.md
    workflow: 15
---

# Modelos locales

Lo local es posible, pero OpenClaw espera un contexto grande y defensas sólidas contra la inyección de prompts. Las tarjetas pequeñas truncarán el contexto y debilitarán la seguridad. Apunta alto: **≥2 Mac Studios al máximo o un equipo GPU equivalente (~$30k+)**. Una sola GPU de **24 GB** solo funciona para prompts más ligeros con mayor latencia. Usa la **variante de modelo más grande / de tamaño completo que puedas ejecutar**; los checkpoints muy cuantizados o “small” aumentan el riesgo de inyección de prompts (consulta [Seguridad](/gateway/security)).

Si quieres la configuración local con menos fricción, empieza con [Ollama](/providers/ollama) y `openclaw onboard`. Esta página es la guía con recomendaciones firmes para stacks locales de gama alta y servidores locales personalizados compatibles con OpenAI.

## Recomendado: LM Studio + modelo local grande (Responses API)

La mejor pila local actual. Carga un modelo grande en LM Studio (por ejemplo, una compilación de tamaño completo de Qwen, DeepSeek o Llama), habilita el servidor local (predeterminado `http://127.0.0.1:1234`) y usa Responses API para mantener el razonamiento separado del texto final.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Modelo local”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Lista de verificación de configuración**

- Instala LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- En LM Studio, descarga la **compilación de modelo más grande disponible** (evita variantes “small” o muy cuantizadas), inicia el servidor y confirma que `http://127.0.0.1:1234/v1/models` lo liste.
- Reemplaza `my-local-model` por el ID real del modelo que muestra LM Studio.
- Mantén el modelo cargado; la carga en frío añade latencia de inicio.
- Ajusta `contextWindow`/`maxTokens` si tu compilación de LM Studio es diferente.
- Para WhatsApp, usa Responses API para que solo se envíe el texto final.

Mantén configurados también los modelos alojados aunque estés ejecutando en local; usa `models.mode: "merge"` para que los respaldos sigan disponibles.

### Configuración híbrida: principal alojado, respaldo local

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Prioridad local con red de seguridad alojada

Intercambia el orden de principal y respaldo; mantén el mismo bloque de proveedores y `models.mode: "merge"` para poder recurrir a Sonnet o Opus cuando el equipo local no esté disponible.

### Alojamiento regional / enrutamiento de datos

- Las variantes alojadas de MiniMax/Kimi/GLM también existen en OpenRouter con endpoints fijados por región (por ejemplo, alojados en EE. UU.). Elige allí la variante regional para mantener el tráfico dentro de la jurisdicción que prefieras mientras sigues usando `models.mode: "merge"` para los respaldos de Anthropic/OpenAI.
- Solo local sigue siendo la opción más sólida en privacidad; el enrutamiento regional alojado es el punto intermedio cuando necesitas funciones del proveedor pero quieres controlar el flujo de datos.

## Otros proxies locales compatibles con OpenAI

vLLM, LiteLLM, OAI-proxy o gateways personalizados funcionan si exponen un endpoint `/v1` de estilo OpenAI. Reemplaza el bloque de proveedor anterior por tu endpoint y el ID de tu modelo:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Mantén `models.mode: "merge"` para que los modelos alojados sigan disponibles como respaldos.

Nota de comportamiento para backends `/v1` locales/proxificados:

- OpenClaw los trata como rutas compatibles con OpenAI de estilo proxy, no como
  endpoints nativos de OpenAI
- aquí no se aplica el modelado de solicitudes exclusivo de OpenAI nativo: sin
  `service_tier`, sin `store` de Responses, sin modelado de carga útil de compatibilidad
  de razonamiento de OpenAI y sin sugerencias de caché de prompt
- las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en estas URLs de proxy personalizadas

## Solución de problemas

- ¿El gateway puede llegar al proxy? `curl http://127.0.0.1:1234/v1/models`.
- ¿El modelo de LM Studio está descargado de memoria? Vuelve a cargarlo; el inicio en frío es una causa común de “bloqueo”.
- ¿Errores de contexto? Reduce `contextWindow` o aumenta el límite de tu servidor.
- Seguridad: los modelos locales omiten filtros del lado del proveedor; mantén los agentes enfocados y la compactación activada para limitar el radio de impacto de la inyección de prompts.
