---
read_when:
    - Depuración de la vista WebChat de Mac o del puerto loopback
summary: Cómo la app de Mac integra el WebChat del gateway y cómo depurarlo
title: WebChat (macOS)
x-i18n:
    generated_at: "2026-04-05T12:48:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f2c45fa5512cc9c5d3b3aa188d94e2e5a90e4bcce607d959d40bea8b17c90c5
    source_path: platforms/mac/webchat.md
    workflow: 15
---

# WebChat (app de macOS)

La app de barra de menús de macOS integra la interfaz de WebChat como una vista SwiftUI nativa. Se
conecta al Gateway y usa de forma predeterminada la **sesión principal** para el
agente seleccionado (con un selector de sesiones para otras sesiones).

- **Modo local**: se conecta directamente al WebSocket del Gateway local.
- **Modo remoto**: reenvía el puerto de control del Gateway por SSH y usa ese
  túnel como plano de datos.

## Inicio y depuración

- Manual: menú Lobster → “Open Chat”.
- Apertura automática para pruebas:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --webchat
  ```

- Registros: `./scripts/clawlog.sh` (subsystem `ai.openclaw`, category `WebChatSwiftUI`).

## Cómo está conectado

- Plano de datos: métodos WS del Gateway `chat.history`, `chat.send`, `chat.abort`,
  `chat.inject` y eventos `chat`, `agent`, `presence`, `tick`, `health`.
- `chat.history` devuelve filas de transcripción normalizadas para visualización: se eliminan
  las etiquetas de directivas inline del texto visible, se eliminan las cargas útiles XML de llamadas a herramientas en texto plano
  (incluyendo `<tool_call>...</tool_call>`,
  `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
  `<function_calls>...</function_calls>` y bloques truncados de llamadas a herramientas) y
  los tokens de control del modelo filtrados en ASCII/ancho completo, se omiten
  las filas puras del asistente con tokens silenciosos como `NO_REPLY` / `no_reply`
  exactos, y las filas sobredimensionadas pueden sustituirse por marcadores.
- Sesión: usa de forma predeterminada la sesión principal (`main`, o `global` cuando el alcance es
  global). La interfaz puede cambiar entre sesiones.
- El onboarding usa una sesión dedicada para mantener separada la configuración inicial.

## Superficie de seguridad

- El modo remoto reenvía solo el puerto de control WebSocket del Gateway por SSH.

## Limitaciones conocidas

- La interfaz está optimizada para sesiones de chat (no para un sandbox de navegador completo).
