---
read_when:
    - Quieres ejecutar OpenClaw con modelos en la nube o locales mediante Ollama
    - Necesitas orientación para la configuración y puesta en marcha de Ollama
summary: Ejecuta OpenClaw con Ollama (modelos en la nube y locales)
title: Ollama
x-i18n:
    generated_at: "2026-04-05T12:51:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 337b8ec3a7756e591e6d6f82e8ad13417f0f20c394ec540e8fc5756e0fc13c29
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama es un runtime LLM local que facilita ejecutar modelos open source en tu máquina. OpenClaw se integra con la API nativa de Ollama (`/api/chat`), admite streaming y llamadas a herramientas, y puede detectar automáticamente modelos locales de Ollama cuando optas por ello con `OLLAMA_API_KEY` (o un perfil de autenticación) y no defines una entrada explícita `models.providers.ollama`.

<Warning>
**Usuarios de Ollama remoto**: no uses la URL compatible con OpenAI `/v1` (`http://host:11434/v1`) con OpenClaw. Esto rompe las llamadas a herramientas y los modelos pueden emitir JSON de herramientas sin procesar como texto plano. Usa la URL nativa de la API de Ollama en su lugar: `baseUrl: "http://host:11434"` (sin `/v1`).
</Warning>

## Inicio rápido

### Onboarding (recomendado)

La forma más rápida de configurar Ollama es mediante onboarding:

```bash
openclaw onboard
```

Selecciona **Ollama** de la lista de proveedores. Onboarding hará lo siguiente:

1. Pedirá la URL base de Ollama desde la que se pueda alcanzar tu instancia (predeterminada `http://127.0.0.1:11434`).
2. Te permitirá elegir **Cloud + Local** (modelos en la nube y modelos locales) o **Local** (solo modelos locales).
3. Abrirá un flujo de inicio de sesión en el navegador si eliges **Cloud + Local** y no has iniciado sesión en ollama.com.
4. Detectará los modelos disponibles y sugerirá valores predeterminados.
5. Hará `pull` automáticamente del modelo seleccionado si no está disponible localmente.

También se admite modo no interactivo:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

Opcionalmente, especifica una URL base o modelo personalizados:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### Configuración manual

1. Instala Ollama: [https://ollama.com/download](https://ollama.com/download)

2. Haz `pull` de un modelo local si quieres inferencia local:

```bash
ollama pull glm-4.7-flash
# or
ollama pull gpt-oss:20b
# or
ollama pull llama3.3
```

3. Si también quieres modelos en la nube, inicia sesión:

```bash
ollama signin
```

4. Ejecuta onboarding y elige `Ollama`:

```bash
openclaw onboard
```

- `Local`: solo modelos locales
- `Cloud + Local`: modelos locales más modelos en la nube
- Los modelos en la nube como `kimi-k2.5:cloud`, `minimax-m2.5:cloud` y `glm-5:cloud` **no** requieren un `ollama pull` local

OpenClaw sugiere actualmente:

- valor predeterminado local: `glm-4.7-flash`
- valores predeterminados en la nube: `kimi-k2.5:cloud`, `minimax-m2.5:cloud`, `glm-5:cloud`

5. Si prefieres una configuración manual, habilita Ollama directamente para OpenClaw (cualquier valor sirve; Ollama no requiere una clave real):

```bash
# Set environment variable
export OLLAMA_API_KEY="ollama-local"

# Or configure in your config file
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. Inspecciona o cambia de modelos:

```bash
openclaw models list
openclaw models set ollama/glm-4.7-flash
```

7. O establece el valor predeterminado en la configuración:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/glm-4.7-flash" },
    },
  },
}
```

## Detección de modelos (proveedor implícito)

Cuando defines `OLLAMA_API_KEY` (o un perfil de autenticación) y **no** defines `models.providers.ollama`, OpenClaw detecta modelos desde la instancia local de Ollama en `http://127.0.0.1:11434`:

- Consulta `/api/tags`
- Usa búsquedas de mejor esfuerzo a `/api/show` para leer `contextWindow` cuando está disponible
- Marca `reasoning` con una heurística basada en el nombre del modelo (`r1`, `reasoning`, `think`)
- Establece `maxTokens` en el límite máximo de tokens predeterminado de Ollama usado por OpenClaw
- Establece todos los costos a `0`

Esto evita entradas manuales de modelos mientras mantiene el catálogo alineado con la instancia local de Ollama.

Para ver qué modelos están disponibles:

```bash
ollama list
openclaw models list
```

Para añadir un modelo nuevo, simplemente haz `pull` con Ollama:

```bash
ollama pull mistral
```

El nuevo modelo se detectará automáticamente y estará disponible para usar.

Si defines explícitamente `models.providers.ollama`, se omite la detección automática y tendrás que definir los modelos manualmente (consulta más abajo).

## Configuración

### Configuración básica (detección implícita)

La forma más sencilla de habilitar Ollama es mediante una variable de entorno:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Configuración explícita (modelos manuales)

Usa configuración explícita cuando:

- Ollama se ejecuta en otro host/puerto.
- Quieres forzar listas de modelos o ventanas de contexto específicas.
- Quieres definiciones de modelos totalmente manuales.

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

Si `OLLAMA_API_KEY` está definida, puedes omitir `apiKey` en la entrada del proveedor y OpenClaw la rellenará para las comprobaciones de disponibilidad.

### URL base personalizada (configuración explícita)

Si Ollama se ejecuta en otro host o puerto (la configuración explícita desactiva la detección automática, así que define los modelos manualmente):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // No /v1 - use native Ollama API URL
        api: "ollama", // Set explicitly to guarantee native tool-calling behavior
      },
    },
  },
}
```

<Warning>
No añadas `/v1` a la URL. La ruta `/v1` usa el modo compatible con OpenAI, donde las llamadas a herramientas no son fiables. Usa la URL base de Ollama sin sufijo de ruta.
</Warning>

### Selección de modelo

Una vez configurado, todos tus modelos de Ollama estarán disponibles:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Modelos en la nube

Los modelos en la nube te permiten ejecutar modelos alojados en la nube (por ejemplo `kimi-k2.5:cloud`, `minimax-m2.5:cloud`, `glm-5:cloud`) junto con tus modelos locales.

Para usar modelos en la nube, selecciona el modo **Cloud + Local** durante la configuración. El asistente comprueba si has iniciado sesión y abre un flujo de inicio de sesión en el navegador cuando es necesario. Si no se puede verificar la autenticación, el asistente vuelve a los valores predeterminados de modelos locales.

También puedes iniciar sesión directamente en [ollama.com/signin](https://ollama.com/signin).

## Ollama Web Search

OpenClaw también admite **Ollama Web Search** como proveedor empaquetado
de `web_search`.

- Usa tu host Ollama configurado (`models.providers.ollama.baseUrl` cuando
  está definido; en caso contrario `http://127.0.0.1:11434`).
- No requiere clave.
- Requiere que Ollama esté en ejecución y que hayas iniciado sesión con `ollama signin`.

Elige **Ollama Web Search** durante `openclaw onboard` o
`openclaw configure --section web`, o establece:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Para ver todos los detalles de configuración y comportamiento, consulta [Ollama Web Search](/tools/ollama-search).

## Avanzado

### Modelos con reasoning

OpenClaw trata por defecto como compatibles con reasoning los modelos con nombres como `deepseek-r1`, `reasoning` o `think`:

```bash
ollama pull deepseek-r1:32b
```

### Costos de modelos

Ollama es gratuito y se ejecuta localmente, por lo que todos los costos de modelos se establecen en $0.

### Configuración de streaming

La integración de Ollama en OpenClaw usa la **API nativa de Ollama** (`/api/chat`) de forma predeterminada, que admite completamente streaming y llamadas a herramientas al mismo tiempo. No se necesita ninguna configuración especial.

#### Modo heredado compatible con OpenAI

<Warning>
**Las llamadas a herramientas no son fiables en el modo compatible con OpenAI.** Usa este modo solo si necesitas formato OpenAI para un proxy y no dependes del comportamiento nativo de llamadas a herramientas.
</Warning>

Si necesitas usar el endpoint compatible con OpenAI en su lugar (por ejemplo, detrás de un proxy que solo admite formato OpenAI), establece `api: "openai-completions"` explícitamente:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // default: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

Es posible que este modo no admita streaming + llamadas a herramientas simultáneamente. Puede que tengas que desactivar el streaming con `params: { streaming: false }` en la configuración del modelo.

Cuando se usa `api: "openai-completions"` con Ollama, OpenClaw inyecta `options.num_ctx` de forma predeterminada para que Ollama no vuelva silenciosamente a una ventana de contexto de 4096. Si tu proxy/servicio ascendente rechaza campos `options` desconocidos, desactiva este comportamiento:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### Ventanas de contexto

Para los modelos detectados automáticamente, OpenClaw usa la ventana de contexto informada por Ollama cuando está disponible; en caso contrario, recurre a la ventana de contexto predeterminada de Ollama usada por OpenClaw. Puedes sobrescribir `contextWindow` y `maxTokens` en la configuración explícita del proveedor.

## Resolución de problemas

### Ollama no se detecta

Asegúrate de que Ollama esté en ejecución, de que hayas definido `OLLAMA_API_KEY` (o un perfil de autenticación) y de que **no** hayas definido una entrada explícita `models.providers.ollama`:

```bash
ollama serve
```

Y de que la API sea accesible:

```bash
curl http://localhost:11434/api/tags
```

### No hay modelos disponibles

Si tu modelo no aparece en la lista, puedes:

- Hacer `pull` del modelo localmente, o
- Definir el modelo explícitamente en `models.providers.ollama`.

Para añadir modelos:

```bash
ollama list  # See what's installed
ollama pull glm-4.7-flash
ollama pull gpt-oss:20b
ollama pull llama3.3     # Or another model
```

### Conexión rechazada

Comprueba que Ollama se esté ejecutando en el puerto correcto:

```bash
# Check if Ollama is running
ps aux | grep ollama

# Or restart Ollama
ollama serve
```

## Ver también

- [Proveedores de modelos](/concepts/model-providers) - Resumen de todos los proveedores
- [Selección de modelos](/concepts/models) - Cómo elegir modelos
- [Configuración](/gateway/configuration) - Referencia completa de configuración
