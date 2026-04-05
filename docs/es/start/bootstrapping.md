---
read_when:
    - Entender qué ocurre en la primera ejecución del agente
    - Explicar dónde se encuentran los archivos de inicialización
    - Depurar la configuración de identidad durante la incorporación
sidebarTitle: Bootstrapping
summary: Ritual de inicialización del agente que prepara los archivos del espacio de trabajo y de identidad
title: Inicialización del agente
x-i18n:
    generated_at: "2026-04-05T12:53:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a08b5102f25c6c4bcdbbdd44384252a9e537b245a7b070c4961a72b4c6c6601
    source_path: start/bootstrapping.md
    workflow: 15
---

# Inicialización del agente

La inicialización es el ritual de **primera ejecución** que prepara un espacio de trabajo del agente y
recopila detalles de identidad. Ocurre después de la incorporación, cuando el agente se inicia
por primera vez.

## Qué hace la inicialización

En la primera ejecución del agente, OpenClaw inicializa el espacio de trabajo (valor predeterminado
`~/.openclaw/workspace`):

- Prepara `AGENTS.md`, `BOOTSTRAP.md`, `IDENTITY.md`, `USER.md`.
- Ejecuta un breve ritual de preguntas y respuestas (una pregunta a la vez).
- Escribe la identidad y las preferencias en `IDENTITY.md`, `USER.md`, `SOUL.md`.
- Elimina `BOOTSTRAP.md` al finalizar para que solo se ejecute una vez.

## Dónde se ejecuta

La inicialización siempre se ejecuta en el **host del gateway**. Si la app de macOS se conecta a
un Gateway remoto, el espacio de trabajo y los archivos de inicialización residen en esa máquina
remota.

<Note>
Cuando el Gateway se ejecuta en otra máquina, edita los archivos del espacio de trabajo en el host del gateway
(por ejemplo, `user@gateway-host:~/.openclaw/workspace`).
</Note>

## Documentos relacionados

- Incorporación de la app de macOS: [Onboarding](/start/onboarding)
- Diseño del espacio de trabajo: [Espacio de trabajo del agente](/es/concepts/agent-workspace)
