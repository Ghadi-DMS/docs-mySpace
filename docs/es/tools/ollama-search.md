---
read_when:
    - Quieres usar Ollama para `web_search`
    - Quieres un proveedor de `web_search` sin clave
    - Necesitas orientación para configurar Ollama Web Search
summary: Ollama Web Search mediante tu host de Ollama configurado
title: Ollama Web Search
x-i18n:
    generated_at: "2026-04-05T12:56:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3c1d0765594e0eb368c25cca21a712c054e71cf43e7bfb385d10feddd990f4fd
    source_path: tools/ollama-search.md
    workflow: 15
---

# Ollama Web Search

OpenClaw admite **Ollama Web Search** como proveedor incluido de `web_search`.
Usa la API experimental de búsqueda web de Ollama y devuelve resultados estructurados
con títulos, URL y fragmentos.

A diferencia del proveedor de modelos de Ollama, esta configuración no necesita una API key de
forma predeterminada. Sí requiere:

- un host de Ollama accesible desde OpenClaw
- `ollama signin`

## Configuración

<Steps>
  <Step title="Iniciar Ollama">
    Asegúrate de que Ollama esté instalado y en ejecución.
  </Step>
  <Step title="Iniciar sesión">
    Ejecuta:

    ```bash
    ollama signin
    ```

  </Step>
  <Step title="Elegir Ollama Web Search">
    Ejecuta:

    ```bash
    openclaw configure --section web
    ```

    Luego selecciona **Ollama Web Search** como proveedor.

  </Step>
</Steps>

Si ya usas Ollama para modelos, Ollama Web Search reutiliza el mismo
host configurado.

## Configuración

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

Anulación opcional del host de Ollama:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

Si no se establece una URL base explícita para Ollama, OpenClaw usa `http://127.0.0.1:11434`.

Si tu host de Ollama espera autenticación bearer, OpenClaw reutiliza
`models.providers.ollama.apiKey` (o la autenticación del proveedor respaldada por env correspondiente)
también para las solicitudes de búsqueda web.

## Notas

- Este proveedor no requiere un campo de API key específico para búsqueda web.
- Si el host de Ollama está protegido por autenticación, OpenClaw reutiliza la API key normal del
  proveedor de Ollama cuando está presente.
- OpenClaw muestra una advertencia durante la configuración si no puede acceder a Ollama o si no has iniciado sesión, pero
  no bloquea la selección.
- La detección automática en runtime puede recurrir a Ollama Web Search cuando no hay configurado ningún proveedor
  con credenciales de mayor prioridad.
- El proveedor usa el endpoint experimental `/api/experimental/web_search`
  de Ollama.

## Relacionado

- [Resumen de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Ollama](/es/providers/ollama) -- configuración del modelo Ollama y modos cloud/local
