---
read_when:
    - Quieres habilitar o configurar `code_execution`
    - Quieres análisis remoto sin acceso al shell local
    - Quieres combinar `x_search` o `web_search` con análisis remoto de Python
summary: code_execution -- ejecuta análisis remoto de Python en entorno aislado con xAI
title: Code Execution
x-i18n:
    generated_at: "2026-04-05T12:55:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 48ca1ddd026cb14837df90ee74859eb98ba6d1a3fbc78da8a72390d0ecee5e40
    source_path: tools/code-execution.md
    workflow: 15
---

# Code Execution

`code_execution` ejecuta análisis remoto de Python en entorno aislado sobre la API de Responses de xAI.
Esto es diferente de [`exec`](/tools/exec) local:

- `exec` ejecuta comandos de shell en tu máquina o nodo
- `code_execution` ejecuta Python en el entorno aislado remoto de xAI

Usa `code_execution` para:

- cálculos
- tabulación
- estadísticas rápidas
- análisis de estilo gráfico
- analizar datos devueltos por `x_search` o `web_search`

**No** lo uses cuando necesites archivos locales, tu shell, tu repositorio o dispositivos
emparejados. Usa [`exec`](/tools/exec) para eso.

## Configuración

Necesitas una API key de xAI. Cualquiera de estas funciona:

- `XAI_API_KEY`
- `plugins.entries.xai.config.webSearch.apiKey`

Ejemplo:

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...",
          },
          codeExecution: {
            enabled: true,
            model: "grok-4-1-fast",
            maxTurns: 2,
            timeoutSeconds: 30,
          },
        },
      },
    },
  },
}
```

## Cómo usarlo

Pregunta de forma natural y deja clara la intención de análisis:

```text
Use code_execution to calculate the 7-day moving average for these numbers: ...
```

```text
Use x_search to find posts mentioning OpenClaw this week, then use code_execution to count them by day.
```

```text
Use web_search to gather the latest AI benchmark numbers, then use code_execution to compare percent changes.
```

La herramienta toma internamente un único parámetro `task`, por lo que el agente debe enviar
la solicitud completa de análisis y cualquier dato en línea en un solo prompt.

## Límites

- Esta es ejecución remota de xAI, no ejecución local de procesos.
- Debe tratarse como análisis efímero, no como un notebook persistente.
- No asumas acceso a archivos locales ni a tu espacio de trabajo.
- Para datos recientes de X, usa primero [`x_search`](/tools/web#x_search).

## Ver también

- [Herramientas web](/tools/web)
- [Exec](/tools/exec)
- [xAI](/es/providers/xai)
