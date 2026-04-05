---
read_when:
    - Elegir una ruta de onboarding
    - Configurar un entorno nuevo
sidebarTitle: Onboarding Overview
summary: Descripción general de las opciones y flujos de onboarding de OpenClaw
title: Descripción general del onboarding
x-i18n:
    generated_at: "2026-04-05T12:54:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 374697c1dbe0c3871c43164076fbed7119ef032f4a40d0f6e421051f914806e5
    source_path: start/onboarding-overview.md
    workflow: 15
---

# Descripción general del onboarding

OpenClaw tiene dos rutas de onboarding. Ambas configuran la autenticación, el Gateway y los canales de chat opcionales; solo difieren en cómo interactúas con la configuración.

## ¿Qué ruta debo usar?

|                | Onboarding con CLI                     | Onboarding en la app de macOS |
| -------------- | -------------------------------------- | ----------------------------- |
| **Plataformas**  | macOS, Linux, Windows (nativo o WSL2) | Solo macOS                    |
| **Interfaz**  | Asistente en terminal                  | IU guiada en la app           |
| **Ideal para**   | Servidores, sin interfaz, control total | Mac de escritorio, configuración visual |
| **Automatización** | `--non-interactive` para scripts        | Solo manual                   |
| **Comando**    | `openclaw onboard`                     | Inicia la app                 |

La mayoría de los usuarios deberían empezar con el **onboarding con CLI**: funciona en todas partes y te da el mayor control.

## Qué configura el onboarding

Independientemente de la ruta que elijas, el onboarding configura:

1. **Proveedor de modelos y autenticación** — clave API, OAuth o token de configuración para el proveedor elegido
2. **Espacio de trabajo** — directorio para archivos del agente, plantillas de bootstrap y memoria
3. **Gateway** — puerto, dirección de enlace, modo de autenticación
4. **Canales** (opcional) — canales de chat integrados y plugins incluidos, como BlueBubbles, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, Telegram, WhatsApp y más
5. **Daemon** (opcional) — servicio en segundo plano para que el Gateway se inicie automáticamente

## Onboarding con CLI

Ejecuta en cualquier terminal:

```bash
openclaw onboard
```

Añade `--install-daemon` para instalar también el servicio en segundo plano en un solo paso.

Referencia completa: [Onboarding (CLI)](/es/start/wizard)
Documentación del comando CLI: [`openclaw onboard`](/cli/onboard)

## Onboarding en la app de macOS

Abre la app OpenClaw. El asistente de primera ejecución te guía por los mismos pasos con una interfaz visual.

Referencia completa: [Onboarding (app de macOS)](/start/onboarding)

## Proveedores personalizados o no listados

Si tu proveedor no aparece en el onboarding, elige **Custom Provider** e introduce:

- modo de compatibilidad de API (compatible con OpenAI, compatible con Anthropic o detección automática)
- URL base y clave API
- ID del modelo y alias opcional

Pueden coexistir varios endpoints personalizados; cada uno obtiene su propio ID de endpoint.
