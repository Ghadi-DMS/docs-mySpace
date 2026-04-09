---
read_when:
    - Quieres ejecutar OpenClaw con modelos en la nube o locales mediante Ollama
    - Necesitas orientación para la configuración y puesta en marcha de Ollama
summary: Ejecuta OpenClaw con Ollama (modelos en la nube y locales)
title: Ollama
x-i18n:
    generated_at: "2026-04-09T01:30:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: d3295a7c879d3636a2ffdec05aea6e670e54a990ef52bd9b0cae253bc24aa3f7
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama es un entorno de ejecución local de LLM que facilita ejecutar modelos de código abierto en tu equipo. OpenClaw se integra con la API nativa de Ollama (`/api/chat`), admite streaming y tool calling, y puede detectar automáticamente modelos locales de Ollama cuando lo habilitas con `OLLAMA_API_KEY` (o un perfil de autenticación) y no defines una entrada explícita `models.providers.ollama`.

<Warning>
**Usuarios de Ollama remoto**: No uses la URL compatible con OpenAI `/v1` (`http://host:11434/v1`) con OpenClaw. Esto rompe el tool calling y los modelos pueden producir JSON de herramientas sin procesar como texto sin formato. Usa en su lugar la URL de la API nativa de Ollama: `baseUrl: "http://host:11434"` (sin `/v1`).
</Warning>

## Inicio rápido

### Incorporación guiada (recomendado)

La forma más rápida de configurar Ollama es mediante la incorporación guiada:

```bash
openclaw onboard
```

Selecciona **Ollama** en la lista de proveedores. La incorporación guiada hará lo siguiente:

1. Te pedirá la URL base de Ollama en la que se puede acceder a tu instancia (predeterminada: `http://127.0.0.1:11434`).
2. Te permitirá elegir **Cloud + Local** (modelos en la nube y modelos locales) o **Local** (solo modelos locales).
3. Abrirá un flujo de inicio de sesión en el navegador si eliges **Cloud + Local** y no has iniciado sesión en ollama.com.
4. Detectará los modelos disponibles y sugerirá valores predeterminados.
5. Hará `pull` automáticamente del modelo seleccionado si no está disponible localmente.

También se admite el modo no interactivo:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

Opcionalmente, especifica una URL base o un modelo personalizados:

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
ollama pull gemma4
# o
ollama pull gpt-oss:20b
# o
ollama pull llama3.3
```

3. Si también quieres modelos en la nube, inicia sesión:

```bash
ollama signin
```

4. Ejecuta la incorporación guiada y elige `Ollama`:

```bash
openclaw onboard
```

- `Local`: solo modelos locales
- `Cloud + Local`: modelos locales más modelos en la nube
- Los modelos en la nube como `kimi-k2.5:cloud`, `minimax-m2.7:cloud` y `glm-5.1:cloud` **no** requieren un `ollama pull` local

OpenClaw sugiere actualmente:

- valor predeterminado local: `gemma4`
- valores predeterminados en la nube: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. Si prefieres la configuración manual, habilita Ollama directamente para OpenClaw (cualquier valor funciona; Ollama no requiere una clave real):

```bash
# Establecer variable de entorno
export OLLAMA_API_KEY="ollama-local"

# O configurar en tu archivo de configuración
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. Inspecciona o cambia de modelo:

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. O establece el valor predeterminado en la configuración:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## Detección de modelos (proveedor implícito)

Cuando estableces `OLLAMA_API_KEY` (o un perfil de autenticación) y **no** defines `models.providers.ollama`, OpenClaw detecta modelos desde la instancia local de Ollama en `http://127.0.0.1:11434`:

- Consulta `/api/tags`
- Usa búsquedas de `/api/show` con mejor esfuerzo para leer `contextWindow` y detectar capacidades (incluida visión) cuando estén disponibles
- Los modelos con una capacidad `vision` informada por `/api/show` se marcan como compatibles con imágenes (`input: ["text", "image"]`), por lo que OpenClaw inyecta automáticamente imágenes en el prompt para esos modelos
- Marca `reasoning` mediante una heurística por nombre de modelo (`r1`, `reasoning`, `think`)
- Establece `maxTokens` en el límite máximo predeterminado de tokens de Ollama usado por OpenClaw
- Establece todos los costos en `0`

Esto evita entradas manuales de modelos y mantiene el catálogo alineado con la instancia local de Ollama.

Para ver qué modelos están disponibles:

```bash
ollama list
openclaw models list
```

Para añadir un modelo nuevo, simplemente haz `pull` con Ollama:

```bash
ollama pull mistral
```

El modelo nuevo se detectará automáticamente y estará disponible para usarse.

Si defines `models.providers.ollama` de forma explícita, se omite la detección automática y debes definir los modelos manualmente (ver más abajo).

## Configuración

### Configuración básica (detección implícita)

La forma más sencilla de habilitar Ollama es mediante una variable de entorno:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Configuración explícita (modelos manuales)

Usa configuración explícita cuando:

- Ollama se ejecuta en otro host o puerto.
- Quieres forzar ventanas de contexto o listas de modelos específicas.
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

Si `OLLAMA_API_KEY` está configurada, puedes omitir `apiKey` en la entrada del proveedor y OpenClaw la completará para las comprobaciones de disponibilidad.

### URL base personalizada (configuración explícita)

Si Ollama se está ejecutando en un host o puerto distintos (la configuración explícita desactiva la detección automática, por lo que debes definir los modelos manualmente):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // Sin /v1: usa la URL de la API nativa de Ollama
        api: "ollama", // Establécelo explícitamente para garantizar el comportamiento nativo de tool calling
      },
    },
  },
}
```

<Warning>
No añadas `/v1` a la URL. La ruta `/v1` usa el modo compatible con OpenAI, donde el tool calling no es fiable. Usa la URL base de Ollama sin sufijo de ruta.
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

Los modelos en la nube te permiten ejecutar modelos alojados en la nube (por ejemplo, `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`) junto con tus modelos locales.

Para usar modelos en la nube, selecciona el modo **Cloud + Local** durante la configuración. El asistente comprueba si has iniciado sesión y abre un flujo de inicio de sesión en el navegador cuando es necesario. Si no se puede verificar la autenticación, el asistente recurre a los valores predeterminados de modelos locales.

También puedes iniciar sesión directamente en [ollama.com/signin](https://ollama.com/signin).

## Ollama Web Search

OpenClaw también admite **Ollama Web Search** como proveedor integrado de `web_search`.

- Usa tu host de Ollama configurado (`models.providers.ollama.baseUrl` cuando está definido; de lo contrario, `http://127.0.0.1:11434`).
- No requiere clave.
- Requiere que Ollama esté en ejecución y que hayas iniciado sesión con `ollama signin`.

Elige **Ollama Web Search** durante `openclaw onboard` o `openclaw configure --section web`, o establece:

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

Para conocer todos los detalles de configuración y comportamiento, consulta [Ollama Web Search](/es/tools/ollama-search).

## Avanzado

### Modelos de razonamiento

OpenClaw trata de forma predeterminada como compatibles con razonamiento a los modelos con nombres como `deepseek-r1`, `reasoning` o `think`:

```bash
ollama pull deepseek-r1:32b
```

### Costos del modelo

Ollama es gratuito y se ejecuta localmente, por lo que todos los costos de modelo se establecen en $0.

### Configuración de streaming

La integración de Ollama en OpenClaw usa de forma predeterminada la **API nativa de Ollama** (`/api/chat`), que admite completamente streaming y tool calling al mismo tiempo. No se necesita ninguna configuración especial.

#### Modo heredado compatible con OpenAI

<Warning>
**El tool calling no es fiable en el modo compatible con OpenAI.** Usa este modo solo si necesitas el formato OpenAI para un proxy y no dependes del comportamiento nativo de tool calling.
</Warning>

Si necesitas usar en su lugar el endpoint compatible con OpenAI (por ejemplo, detrás de un proxy que solo admite formato OpenAI), establece `api: "openai-completions"` explícitamente:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // predeterminado: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

Es posible que este modo no admita streaming + tool calling simultáneamente. Puede que necesites desactivar el streaming con `params: { streaming: false }` en la configuración del modelo.

Cuando se usa `api: "openai-completions"` con Ollama, OpenClaw inyecta `options.num_ctx` de forma predeterminada para que Ollama no vuelva silenciosamente a una ventana de contexto de 4096. Si tu proxy o servicio ascendente rechaza campos `options` desconocidos, desactiva este comportamiento:

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

Para los modelos detectados automáticamente, OpenClaw usa la ventana de contexto informada por Ollama cuando está disponible; de lo contrario, recurre a la ventana de contexto predeterminada de Ollama usada por OpenClaw. Puedes sobrescribir `contextWindow` y `maxTokens` en la configuración explícita del proveedor.

## Solución de problemas

### Ollama no se detecta

Asegúrate de que Ollama esté en ejecución, de que hayas configurado `OLLAMA_API_KEY` (o un perfil de autenticación) y de que **no** hayas definido una entrada explícita `models.providers.ollama`:

```bash
ollama serve
```

Y de que se pueda acceder a la API:

```bash
curl http://localhost:11434/api/tags
```

### No hay modelos disponibles

Si tu modelo no aparece en la lista, haz una de estas dos cosas:

- Haz `pull` del modelo localmente, o
- Define el modelo explícitamente en `models.providers.ollama`.

Para añadir modelos:

```bash
ollama list  # Ver qué está instalado
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # O cualquier otro modelo
```

### Conexión rechazada

Comprueba que Ollama se esté ejecutando en el puerto correcto:

```bash
# Comprobar si Ollama se está ejecutando
ps aux | grep ollama

# O reiniciar Ollama
ollama serve
```

## Ver también

- [Proveedores de modelos](/es/concepts/model-providers) - Resumen de todos los proveedores
- [Selección de modelos](/es/concepts/models) - Cómo elegir modelos
- [Configuración](/es/gateway/configuration) - Referencia completa de configuración
