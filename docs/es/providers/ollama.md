---
read_when:
    - Quieres ejecutar OpenClaw con modelos en la nube o locales mediante Ollama
    - Necesitas orientación de configuración y puesta en marcha de Ollama
summary: Ejecuta OpenClaw con Ollama (modelos en la nube y locales)
title: Ollama
x-i18n:
    generated_at: "2026-04-08T02:18:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 222ec68f7d4bb29cc7796559ddef1d5059f5159e7a51e2baa3a271ddb3abb716
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama es un runtime de LLM local que facilita ejecutar modelos de código abierto en tu equipo. OpenClaw se integra con la API nativa de Ollama (`/api/chat`), admite streaming y llamadas a herramientas, y puede detectar automáticamente modelos locales de Ollama cuando optas por usar `OLLAMA_API_KEY` (o un perfil de autenticación) y no defines una entrada explícita `models.providers.ollama`.

<Warning>
**Usuarios de Ollama remoto**: No uses la URL compatible con OpenAI `/v1` (`http://host:11434/v1`) con OpenClaw. Esto rompe las llamadas a herramientas y los modelos pueden generar JSON de herramientas sin procesar como texto plano. Usa en su lugar la URL de la API nativa de Ollama: `baseUrl: "http://host:11434"` (sin `/v1`).
</Warning>

## Inicio rápido

### Onboarding (recomendado)

La forma más rápida de configurar Ollama es mediante el onboarding:

```bash
openclaw onboard
```

Selecciona **Ollama** en la lista de proveedores. El onboarding hará lo siguiente:

1. Pedirá la URL base de Ollama donde se puede acceder a tu instancia (valor predeterminado `http://127.0.0.1:11434`).
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

4. Ejecuta el onboarding y elige `Ollama`:

```bash
openclaw onboard
```

- `Local`: solo modelos locales
- `Cloud + Local`: modelos locales más modelos en la nube
- Los modelos en la nube como `kimi-k2.5:cloud`, `minimax-m2.7:cloud` y `glm-5.1:cloud` **no** requieren un `ollama pull` local

OpenClaw sugiere actualmente:

- valor predeterminado local: `gemma4`
- valores predeterminados en la nube: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. Si prefieres una configuración manual, habilita Ollama para OpenClaw directamente (cualquier valor funciona; Ollama no requiere una clave real):

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

Cuando estableces `OLLAMA_API_KEY` (o un perfil de autenticación) y **no** defines `models.providers.ollama`, OpenClaw detecta modelos de la instancia local de Ollama en `http://127.0.0.1:11434`:

- Consulta `/api/tags`
- Usa búsquedas `/api/show` en modo best-effort para leer `contextWindow` cuando está disponible
- Marca `reasoning` con una heurística basada en el nombre del modelo (`r1`, `reasoning`, `think`)
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

El nuevo modelo se detectará automáticamente y estará disponible para usar.

Si configuras `models.providers.ollama` explícitamente, la detección automática se omite y debes definir los modelos manualmente (ver más abajo).

## Configuración

### Configuración básica (detección implícita)

La forma más sencilla de habilitar Ollama es mediante una variable de entorno:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Configuración explícita (modelos manuales)

Usa configuración explícita cuando:

- Ollama se ejecuta en otro host/puerto.
- Quieres forzar ventanas de contexto o listas de modelos específicas.
- Quieres definiciones de modelos completamente manuales.

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

Si `OLLAMA_API_KEY` está establecido, puedes omitir `apiKey` en la entrada del proveedor y OpenClaw lo completará para las comprobaciones de disponibilidad.

### URL base personalizada (configuración explícita)

Si Ollama se está ejecutando en un host o puerto diferente (la configuración explícita desactiva la detección automática, así que define los modelos manualmente):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // Sin /v1: usa la URL nativa de la API de Ollama
        api: "ollama", // Establécelo explícitamente para garantizar el comportamiento nativo de llamadas a herramientas
      },
    },
  },
}
```

<Warning>
No añadas `/v1` a la URL. La ruta `/v1` usa el modo compatible con OpenAI, donde las llamadas a herramientas no son fiables. Usa la URL base de Ollama sin un sufijo de ruta.
</Warning>

### Selección de modelos

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

Para usar modelos en la nube, selecciona el modo **Cloud + Local** durante la configuración. El asistente comprueba si has iniciado sesión y abre un flujo de inicio de sesión en el navegador cuando es necesario. Si no se puede verificar la autenticación, el asistente vuelve a los valores predeterminados de modelos locales.

También puedes iniciar sesión directamente en [ollama.com/signin](https://ollama.com/signin).

## Búsqueda web de Ollama

OpenClaw también es compatible con **Ollama Web Search** como proveedor
empaquetado de `web_search`.

- Usa tu host de Ollama configurado (`models.providers.ollama.baseUrl` cuando
  está configurado; de lo contrario `http://127.0.0.1:11434`).
- No requiere clave.
- Requiere que Ollama esté en ejecución y que hayas iniciado sesión con `ollama signin`.

Elige **Ollama Web Search** durante `openclaw onboard` o
`openclaw configure --section web`, o configura:

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

Para ver la configuración completa y los detalles del comportamiento, consulta [Ollama Web Search](/es/tools/ollama-search).

## Avanzado

### Modelos de razonamiento

OpenClaw trata de forma predeterminada los modelos con nombres como `deepseek-r1`, `reasoning` o `think` como modelos con capacidad de razonamiento:

```bash
ollama pull deepseek-r1:32b
```

### Costos de modelos

Ollama es gratuito y se ejecuta localmente, por lo que todos los costos de los modelos se establecen en $0.

### Configuración de streaming

La integración de Ollama en OpenClaw usa la **API nativa de Ollama** (`/api/chat`) de forma predeterminada, que admite completamente streaming y llamadas a herramientas al mismo tiempo. No se necesita ninguna configuración especial.

#### Modo heredado compatible con OpenAI

<Warning>
**Las llamadas a herramientas no son fiables en el modo compatible con OpenAI.** Usa este modo solo si necesitas el formato de OpenAI para un proxy y no dependes del comportamiento nativo de llamadas a herramientas.
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

Es posible que este modo no admita streaming + llamadas a herramientas simultáneamente. Puede que necesites desactivar el streaming con `params: { streaming: false }` en la configuración del modelo.

Cuando se usa `api: "openai-completions"` con Ollama, OpenClaw inyecta `options.num_ctx` de forma predeterminada para que Ollama no vuelva silenciosamente a una ventana de contexto de 4096. Si tu proxy/upstream rechaza campos `options` desconocidos, desactiva este comportamiento:

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

### No se detecta Ollama

Asegúrate de que Ollama se está ejecutando, de que estableciste `OLLAMA_API_KEY` (o un perfil de autenticación) y de que **no** definiste una entrada explícita `models.providers.ollama`:

```bash
ollama serve
```

Y de que la API sea accesible:

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
