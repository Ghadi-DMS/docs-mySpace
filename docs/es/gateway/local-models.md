---
read_when:
    - Quieres servir modelos desde tu propio equipo con GPU
    - Estás configurando LM Studio o un proxy compatible con OpenAI
    - Necesitas la guía más segura para modelos locales
summary: Ejecuta OpenClaw en LLM locales (LM Studio, vLLM, LiteLLM, endpoints OpenAI personalizados)
title: Modelos locales
x-i18n:
    generated_at: "2026-04-08T02:14:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: d619d72b0e06914ebacb7e9f38b746caf1b9ce8908c9c6638c3acdddbaa025e8
    source_path: gateway/local-models.md
    workflow: 15
---

# Modelos locales

Lo local es viable, pero OpenClaw espera un contexto amplio y defensas sólidas contra la inyección de prompts. Las tarjetas pequeñas truncarán el contexto y debilitarán la seguridad. Apunta alto: **≥2 Mac Studio al máximo o un equipo GPU equivalente (~$30k+)**. Una sola GPU de **24 GB** funciona solo para prompts más ligeros y con mayor latencia. Usa la **variante de modelo más grande o completa que puedas ejecutar**; los checkpoints muy cuantizados o “small” aumentan el riesgo de inyección de prompts (consulta [Security](/es/gateway/security)).

Si quieres la configuración local con menos fricción, empieza con [Ollama](/es/providers/ollama) y `openclaw onboard`. Esta página es la guía con recomendaciones concretas para stacks locales de gama alta y servidores locales personalizados compatibles con OpenAI.

## Recomendado: LM Studio + modelo local grande (Responses API)

La mejor pila local actual. Carga un modelo grande en LM Studio (por ejemplo, una compilación completa de Qwen, DeepSeek o Llama), habilita el servidor local (valor predeterminado `http://127.0.0.1:1234`) y usa Responses API para mantener el razonamiento separado del texto final.

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
            name: “Local Model”,
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
- En LM Studio, descarga la **compilación de modelo más grande disponible** (evita variantes “small” o muy cuantizadas), inicia el servidor y confirma que `http://127.0.0.1:1234/v1/models` la muestra.
- Sustituye `my-local-model` por el ID de modelo real que aparece en LM Studio.
- Mantén el modelo cargado; una carga en frío añade latencia de inicio.
- Ajusta `contextWindow`/`maxTokens` si tu compilación de LM Studio es distinta.
- Para WhatsApp, usa Responses API para que solo se envíe el texto final.

Mantén los modelos alojados configurados incluso cuando ejecutes en local; usa `models.mode: "merge"` para que las alternativas sigan disponibles.

### Configuración híbrida: principal alojado, alternativa local

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

Intercambia el orden de principal y alternativa; mantén el mismo bloque `providers` y `models.mode: "merge"` para poder recurrir a Sonnet o Opus cuando el equipo local no esté disponible.

### Alojamiento regional / enrutamiento de datos

- Las variantes alojadas de MiniMax/Kimi/GLM también existen en OpenRouter con endpoints fijados por región (por ejemplo, alojados en EE. UU.). Elige allí la variante regional para mantener el tráfico dentro de la jurisdicción que prefieras mientras sigues usando `models.mode: "merge"` para las alternativas de Anthropic/OpenAI.
- Solo local sigue siendo la opción más sólida para la privacidad; el enrutamiento regional alojado es el punto intermedio cuando necesitas funciones del proveedor pero quieres controlar el flujo de datos.

## Otros proxies locales compatibles con OpenAI

vLLM, LiteLLM, OAI-proxy o puertas de enlace personalizadas funcionan si exponen un endpoint `/v1` de estilo OpenAI. Sustituye el bloque del proveedor anterior por tu endpoint y tu ID de modelo:

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

Mantén `models.mode: "merge"` para que los modelos alojados sigan disponibles como alternativas.

Nota de comportamiento para backends `/v1` locales o con proxy:

- OpenClaw los trata como rutas compatibles con OpenAI de estilo proxy, no como
  endpoints nativos de OpenAI
- aquí no se aplica el moldeado de solicitudes exclusivo de OpenAI nativo: sin
  `service_tier`, sin Responses `store`, sin moldeado de carga útil de compatibilidad de razonamiento de OpenAI
  y sin sugerencias de caché de prompts
- los encabezados de atribución ocultos de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en estas URL de proxy personalizadas

Notas de compatibilidad para backends compatibles con OpenAI más estrictos:

- Algunos servidores aceptan solo `messages[].content` como cadena en Chat Completions, no
  matrices estructuradas de partes de contenido. Configura
  `models.providers.<provider>.models[].compat.requiresStringContent: true` para
  esos endpoints.
- Algunos backends locales más pequeños o más estrictos son inestables con la forma completa del prompt
  del entorno de ejecución de agentes de OpenClaw, especialmente cuando se incluyen esquemas de herramientas. Si el
  backend funciona para llamadas directas pequeñas a `/v1/chat/completions` pero falla en turnos normales
  del agente de OpenClaw, prueba primero con
  `models.providers.<provider>.models[].compat.supportsTools: false`.
- Si el backend sigue fallando solo en ejecuciones más grandes de OpenClaw, el problema restante
  suele ser la capacidad del modelo/servidor aguas arriba o un error del backend, no la
  capa de transporte de OpenClaw.

## Solución de problemas

- ¿La puerta de enlace puede alcanzar el proxy? `curl http://127.0.0.1:1234/v1/models`.
- ¿El modelo de LM Studio se descargó de memoria? Vuélvelo a cargar; el arranque en frío es una causa común de “bloqueo”.
- ¿Errores de contexto? Reduce `contextWindow` o aumenta el límite de tu servidor.
- ¿El servidor compatible con OpenAI devuelve `messages[].content ... expected a string`?
  Añade `compat.requiresStringContent: true` en esa entrada de modelo.
- ¿Las llamadas directas pequeñas a `/v1/chat/completions` funcionan, pero `openclaw infer model run`
  falla en Gemma u otro modelo local? Primero desactiva los esquemas de herramientas con
  `compat.supportsTools: false` y vuelve a probar. Si el servidor sigue fallando solo
  con prompts más grandes de OpenClaw, trátalo como una limitación del servidor/modelo aguas arriba.
- Seguridad: los modelos locales omiten los filtros del lado del proveedor; mantén los agentes limitados y la compactación activada para reducir el radio de impacto de la inyección de prompts.
