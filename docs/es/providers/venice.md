---
read_when:
    - Quieres una inferencia centrada en la privacidad en OpenClaw
    - Quieres orientación para configurar Venice AI
summary: Usa los modelos centrados en la privacidad de Venice AI en OpenClaw
title: Venice AI
x-i18n:
    generated_at: "2026-04-05T12:53:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 53313e45e197880feb7e90764ee8fd6bb7f5fd4fe03af46b594201c77fbc8eab
    source_path: providers/venice.md
    workflow: 15
---

# Venice AI (destacado de Venice)

**Venice** es nuestra configuración destacada de Venice para inferencia centrada en la privacidad con acceso anonimizado opcional a modelos propietarios.

Venice AI ofrece inferencia de IA centrada en la privacidad con soporte para modelos sin censura y acceso a los principales modelos propietarios a través de su proxy anonimizado. Toda la inferencia es privada de forma predeterminada: no se entrena con tus datos ni se registran.

## Por qué usar Venice en OpenClaw

- **Inferencia privada** para modelos de código abierto (sin registros).
- **Modelos sin censura** cuando los necesites.
- **Acceso anonimizado** a modelos propietarios (Opus/GPT/Gemini) cuando la calidad importe.
- Endpoints `/v1` compatibles con OpenAI.

## Modos de privacidad

Venice ofrece dos niveles de privacidad; entender esto es clave para elegir tu modelo:

| Modo           | Descripción                                                                                                                      | Modelos                                                       |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Privado**    | Totalmente privado. Los prompts/respuestas **nunca se almacenan ni se registran**. Efímero.                                     | Llama, Qwen, DeepSeek, Kimi, MiniMax, Venice Uncensored, etc. |
| **Anonimizado** | Se canaliza a través de Venice con los metadatos eliminados. El proveedor subyacente (OpenAI, Anthropic, Google, xAI) ve solicitudes anonimizadas. | Claude, GPT, Gemini, Grok                                     |

## Funciones

- **Centrado en la privacidad**: Elige entre los modos "private" (totalmente privado) y "anonymized" (con proxy)
- **Modelos sin censura**: Accede a modelos sin restricciones de contenido
- **Acceso a modelos principales**: Usa Claude, GPT, Gemini y Grok mediante el proxy anonimizado de Venice
- **API compatible con OpenAI**: Endpoints `/v1` estándar para una integración sencilla
- **Streaming**: ✅ Compatible con todos los modelos
- **Llamada a funciones**: ✅ Compatible en modelos seleccionados (consulta las capacidades del modelo)
- **Visión**: ✅ Compatible en modelos con capacidad de visión
- **Sin límites de tasa estrictos**: Puede aplicarse limitación por uso justo en casos de uso extremo

## Configuración

### 1. Obtener la clave de API

1. Regístrate en [venice.ai](https://venice.ai)
2. Ve a **Settings → API Keys → Create new key**
3. Copia tu clave de API (formato: `vapi_xxxxxxxxxxxx`)

### 2. Configurar OpenClaw

**Opción A: Variable de entorno**

```bash
export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
```

**Opción B: Configuración interactiva (recomendada)**

```bash
openclaw onboard --auth-choice venice-api-key
```

Esto hará lo siguiente:

1. Solicitar tu clave de API (o usar `VENICE_API_KEY` existente)
2. Mostrar todos los modelos de Venice disponibles
3. Permitirte elegir tu modelo predeterminado
4. Configurar el proveedor automáticamente

**Opción C: No interactiva**

```bash
openclaw onboard --non-interactive \
  --auth-choice venice-api-key \
  --venice-api-key "vapi_xxxxxxxxxxxx"
```

### 3. Verificar la configuración

```bash
openclaw agent --model venice/kimi-k2-5 --message "Hola, ¿estás funcionando?"
```

## Selección de modelo

Después de la configuración, OpenClaw muestra todos los modelos de Venice disponibles. Elige según tus necesidades:

- **Modelo predeterminado**: `venice/kimi-k2-5` para un razonamiento privado sólido más visión.
- **Opción de alta capacidad**: `venice/claude-opus-4-6` para la ruta anonimizada de Venice más potente.
- **Privacidad**: Elige modelos "private" para una inferencia totalmente privada.
- **Capacidad**: Elige modelos "anonymized" para acceder a Claude, GPT y Gemini mediante el proxy de Venice.

Cambia tu modelo predeterminado en cualquier momento:

```bash
openclaw models set venice/kimi-k2-5
openclaw models set venice/claude-opus-4-6
```

Muestra todos los modelos disponibles:

```bash
openclaw models list | grep venice
```

## Configurar mediante `openclaw configure`

1. Ejecuta `openclaw configure`
2. Selecciona **Model/auth**
3. Elige **Venice AI**

## ¿Qué modelo debería usar?

| Caso de uso                | Modelo recomendado               | Motivo                                       |
| -------------------------- | -------------------------------- | -------------------------------------------- |
| **Chat general (predeterminado)** | `kimi-k2-5`                      | Razonamiento privado sólido más visión       |
| **Mejor calidad general**  | `claude-opus-4-6`                | Opción anonimizada de Venice más potente     |
| **Privacidad + código**    | `qwen3-coder-480b-a35b-instruct` | Modelo privado para código con gran contexto |
| **Visión privada**         | `kimi-k2-5`                      | Soporte de visión sin salir del modo privado |
| **Rápido + barato**        | `qwen3-4b`                       | Modelo de razonamiento ligero                |
| **Tareas privadas complejas** | `deepseek-v3.2`                  | Razonamiento sólido, pero sin soporte de herramientas de Venice |
| **Sin censura**            | `venice-uncensored`              | Sin restricciones de contenido               |

## Modelos disponibles (41 en total)

### Modelos privados (26) - Totalmente privados, sin registros

| ID del modelo                           | Nombre                              | Contexto | Funciones                  |
| -------------------------------------- | ----------------------------------- | ------- | -------------------------- |
| `kimi-k2-5`                            | Kimi K2.5                           | 256k    | Predeterminado, razonamiento, visión |
| `kimi-k2-thinking`                     | Kimi K2 Thinking                    | 256k    | Razonamiento               |
| `llama-3.3-70b`                        | Llama 3.3 70B                       | 128k    | General                    |
| `llama-3.2-3b`                         | Llama 3.2 3B                        | 128k    | General                    |
| `hermes-3-llama-3.1-405b`              | Hermes 3 Llama 3.1 405B             | 128k    | General, herramientas deshabilitadas |
| `qwen3-235b-a22b-thinking-2507`        | Qwen3 235B Thinking                 | 128k    | Razonamiento               |
| `qwen3-235b-a22b-instruct-2507`        | Qwen3 235B Instruct                 | 128k    | General                    |
| `qwen3-coder-480b-a35b-instruct`       | Qwen3 Coder 480B                    | 256k    | Código                     |
| `qwen3-coder-480b-a35b-instruct-turbo` | Qwen3 Coder 480B Turbo              | 256k    | Código                     |
| `qwen3-5-35b-a3b`                      | Qwen3.5 35B A3B                     | 256k    | Razonamiento, visión       |
| `qwen3-next-80b`                       | Qwen3 Next 80B                      | 256k    | General                    |
| `qwen3-vl-235b-a22b`                   | Qwen3 VL 235B (Vision)              | 256k    | Visión                     |
| `qwen3-4b`                             | Venice Small (Qwen3 4B)             | 32k     | Rápido, razonamiento       |
| `deepseek-v3.2`                        | DeepSeek V3.2                       | 160k    | Razonamiento, herramientas deshabilitadas |
| `venice-uncensored`                    | Venice Uncensored (Dolphin-Mistral) | 32k     | Sin censura, herramientas deshabilitadas |
| `mistral-31-24b`                       | Venice Medium (Mistral)             | 128k    | Visión                     |
| `google-gemma-3-27b-it`                | Google Gemma 3 27B Instruct         | 198k    | Visión                     |
| `openai-gpt-oss-120b`                  | OpenAI GPT OSS 120B                 | 128k    | General                    |
| `nvidia-nemotron-3-nano-30b-a3b`       | NVIDIA Nemotron 3 Nano 30B          | 128k    | General                    |
| `olafangensan-glm-4.7-flash-heretic`   | GLM 4.7 Flash Heretic               | 128k    | Razonamiento               |
| `zai-org-glm-4.6`                      | GLM 4.6                             | 198k    | General                    |
| `zai-org-glm-4.7`                      | GLM 4.7                             | 198k    | Razonamiento               |
| `zai-org-glm-4.7-flash`                | GLM 4.7 Flash                       | 128k    | Razonamiento               |
| `zai-org-glm-5`                        | GLM 5                               | 198k    | Razonamiento               |
| `minimax-m21`                          | MiniMax M2.1                        | 198k    | Razonamiento               |
| `minimax-m25`                          | MiniMax M2.5                        | 198k    | Razonamiento               |

### Modelos anonimizados (15) - A través del proxy de Venice

| ID del modelo                    | Nombre                         | Contexto | Funciones                 |
| ------------------------------- | ------------------------------ | ------- | ------------------------- |
| `claude-opus-4-6`               | Claude Opus 4.6 (via Venice)   | 1M      | Razonamiento, visión      |
| `claude-opus-4-5`               | Claude Opus 4.5 (via Venice)   | 198k    | Razonamiento, visión      |
| `claude-sonnet-4-6`             | Claude Sonnet 4.6 (via Venice) | 1M      | Razonamiento, visión      |
| `claude-sonnet-4-5`             | Claude Sonnet 4.5 (via Venice) | 198k    | Razonamiento, visión      |
| `openai-gpt-54`                 | GPT-5.4 (via Venice)           | 1M      | Razonamiento, visión      |
| `openai-gpt-53-codex`           | GPT-5.3 Codex (via Venice)     | 400k    | Razonamiento, visión, código |
| `openai-gpt-52`                 | GPT-5.2 (via Venice)           | 256k    | Razonamiento              |
| `openai-gpt-52-codex`           | GPT-5.2 Codex (via Venice)     | 256k    | Razonamiento, visión, código |
| `openai-gpt-4o-2024-11-20`      | GPT-4o (via Venice)            | 128k    | Visión                    |
| `openai-gpt-4o-mini-2024-07-18` | GPT-4o Mini (via Venice)       | 128k    | Visión                    |
| `gemini-3-1-pro-preview`        | Gemini 3.1 Pro (via Venice)    | 1M      | Razonamiento, visión      |
| `gemini-3-pro-preview`          | Gemini 3 Pro (via Venice)      | 198k    | Razonamiento, visión      |
| `gemini-3-flash-preview`        | Gemini 3 Flash (via Venice)    | 256k    | Razonamiento, visión      |
| `grok-41-fast`                  | Grok 4.1 Fast (via Venice)     | 1M      | Razonamiento, visión      |
| `grok-code-fast-1`              | Grok Code Fast 1 (via Venice)  | 256k    | Razonamiento, código      |

## Descubrimiento de modelos

OpenClaw detecta automáticamente los modelos desde la API de Venice cuando `VENICE_API_KEY` está configurada. Si la API no está accesible, recurre a un catálogo estático.

El endpoint `/models` es público (no se necesita autenticación para listar), pero la inferencia requiere una clave de API válida.

## Streaming y soporte de herramientas

| Función              | Soporte                                                 |
| -------------------- | ------------------------------------------------------- |
| **Streaming**        | ✅ Todos los modelos                                    |
| **Llamada a funciones** | ✅ La mayoría de los modelos (consulta `supportsFunctionCalling` en la API) |
| **Visión/Imágenes**  | ✅ Modelos marcados con la función "Vision"             |
| **Modo JSON**        | ✅ Compatible mediante `response_format`                |

## Precios

Venice usa un sistema basado en créditos. Consulta [venice.ai/pricing](https://venice.ai/pricing) para ver las tarifas actuales:

- **Modelos privados**: Generalmente de menor costo
- **Modelos anonimizados**: Similares al precio directo de la API + una pequeña tarifa de Venice

## Comparación: Venice frente a API directa

| Aspecto      | Venice (anonimizado)          | API directa          |
| ------------ | ----------------------------- | -------------------- |
| **Privacidad** | Metadatos eliminados, anonimizado | Tu cuenta vinculada  |
| **Latencia** | +10-50ms (proxy)              | Directa              |
| **Funciones** | La mayoría de las funciones compatibles | Funciones completas |
| **Facturación** | Créditos de Venice            | Facturación del proveedor |

## Ejemplos de uso

```bash
# Usa el modelo privado predeterminado
openclaw agent --model venice/kimi-k2-5 --message "Quick health check"

# Usa Claude Opus mediante Venice (anonimizado)
openclaw agent --model venice/claude-opus-4-6 --message "Summarize this task"

# Usa un modelo sin censura
openclaw agent --model venice/venice-uncensored --message "Draft options"

# Usa un modelo de visión con imagen
openclaw agent --model venice/qwen3-vl-235b-a22b --message "Review attached image"

# Usa un modelo para código
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct --message "Refactor this function"
```

## Solución de problemas

### Clave de API no reconocida

```bash
echo $VENICE_API_KEY
openclaw models list | grep venice
```

Asegúrate de que la clave comience con `vapi_`.

### Modelo no disponible

El catálogo de modelos de Venice se actualiza dinámicamente. Ejecuta `openclaw models list` para ver los modelos actualmente disponibles. Algunos modelos pueden estar temporalmente fuera de línea.

### Problemas de conexión

La API de Venice está en `https://api.venice.ai/api/v1`. Asegúrate de que tu red permita conexiones HTTPS.

## Ejemplo de archivo de configuración

```json5
{
  env: { VENICE_API_KEY: "vapi_..." },
  agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
  models: {
    mode: "merge",
    providers: {
      venice: {
        baseUrl: "https://api.venice.ai/api/v1",
        apiKey: "${VENICE_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2-5",
            name: "Kimi K2.5",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 256000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## Enlaces

- [Venice AI](https://venice.ai)
- [Documentación de la API](https://docs.venice.ai)
- [Precios](https://venice.ai/pricing)
- [Estado](https://status.venice.ai)
