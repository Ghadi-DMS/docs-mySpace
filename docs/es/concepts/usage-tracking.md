---
read_when:
    - Estás conectando superficies de uso/cuota de proveedores
    - Necesitas explicar el comportamiento del seguimiento de uso o los requisitos de autenticación
summary: Superficies de seguimiento de uso y requisitos de credenciales
title: Seguimiento de uso
x-i18n:
    generated_at: "2026-04-05T12:40:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 62164492c61a8d602e3b73879c13ce3e14ce35964b7f2ffd389a4e6a7ec7e9c0
    source_path: concepts/usage-tracking.md
    workflow: 15
---

# Seguimiento de uso

## Qué es

- Obtiene el uso/la cuota del proveedor directamente de sus endpoints de uso.
- No hay costos estimados; solo las ventanas informadas por el proveedor.
- La salida de estado legible para humanos se normaliza a `X% left`, incluso cuando una
  API upstream informa cuota consumida, cuota restante o solo conteos sin procesar.
- `/status` a nivel de sesión y `session_status` pueden recurrir a la entrada de uso más reciente de la
  transcripción cuando la instantánea de la sesión en vivo es escasa. Ese
  respaldo completa contadores faltantes de tokens/caché, puede recuperar la etiqueta del
  modelo activo en tiempo de ejecución y prefiere el total mayor orientado a prompts cuando faltan
  metadatos de sesión o son menores. Los valores existentes en vivo distintos de cero siguen teniendo prioridad.

## Dónde aparece

- `/status` en chats: tarjeta de estado con muchos emojis con tokens de sesión + costo estimado (solo clave API). El uso del proveedor se muestra para el **proveedor del modelo actual** cuando está disponible como una ventana normalizada `X% left`.
- `/usage off|tokens|full` en chats: pie de uso por respuesta (OAuth muestra solo tokens).
- `/usage cost` en chats: resumen local de costos agregado desde los registros de sesión de OpenClaw.
- CLI: `openclaw status --usage` imprime un desglose completo por proveedor.
- CLI: `openclaw channels list` imprime la misma instantánea de uso junto con la configuración del proveedor (usa `--no-usage` para omitirla).
- Barra de menús de macOS: sección “Usage” en Context (solo si está disponible).

## Proveedores + credenciales

- **Anthropic (Claude)**: tokens OAuth en perfiles de autenticación.
- **GitHub Copilot**: tokens OAuth en perfiles de autenticación.
- **Gemini CLI**: tokens OAuth en perfiles de autenticación.
  - El uso JSON recurre a `stats`; `stats.cached` se normaliza a
    `cacheRead`.
- **OpenAI Codex**: tokens OAuth en perfiles de autenticación (`accountId` se usa cuando está presente).
- **MiniMax**: clave API o perfil de autenticación OAuth de MiniMax. OpenClaw trata
  `minimax`, `minimax-cn` y `minimax-portal` como la misma superficie de cuota de MiniMax,
  prefiere el OAuth de MiniMax almacenado cuando está presente y, de lo contrario, recurre a
  `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` o `MINIMAX_API_KEY`.
  Los campos sin procesar `usage_percent` / `usagePercent` de MiniMax significan cuota
  **restante**, por lo que OpenClaw los invierte antes de mostrarlos; los campos basados en conteos tienen prioridad cuando
  están presentes.
  - Las etiquetas de ventana del plan de programación provienen de los campos de horas/minutos del proveedor cuando
    están presentes y luego recurren al intervalo `start_time` / `end_time`.
  - Si el endpoint del plan de programación devuelve `model_remains`, OpenClaw prefiere la
    entrada del modelo de chat, deriva la etiqueta de ventana a partir de marcas de tiempo cuando faltan
    los campos explícitos `window_hours` / `window_minutes`, e incluye el nombre del modelo
    en la etiqueta del plan.
- **Xiaomi MiMo**: clave API mediante env/config/auth store (`XIAOMI_API_KEY`).
- **z.ai**: clave API mediante env/config/auth store.

El uso se oculta cuando no puede resolverse una autenticación utilizable de uso del proveedor. Los proveedores
pueden suministrar lógica de autenticación de uso específica del plugin; de lo contrario, OpenClaw recurre a
emparejar credenciales OAuth/claves API de perfiles de autenticación, variables de entorno
o configuración.
