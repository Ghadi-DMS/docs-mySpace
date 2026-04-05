---
read_when:
    - Buscando compatibilidad con SO o rutas de instalación
    - Decidiendo dónde ejecutar el Gateway
summary: Resumen de compatibilidad de plataformas (Gateway + apps complementarias)
title: Plataformas
x-i18n:
    generated_at: "2026-04-05T12:47:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: d5be4743fd39eca426d65db940f04f3a8fc3ff2c5e10b0e82bc55fc35a7d1399
    source_path: platforms/index.md
    workflow: 15
---

# Plataformas

El núcleo de OpenClaw está escrito en TypeScript. **Node es el tiempo de ejecución recomendado**.
Bun no se recomienda para el Gateway (errores de WhatsApp/Telegram).

Existen apps complementarias para macOS (app de barra de menús) y nodos móviles (iOS/Android). Las apps complementarias para Windows y
Linux están planificadas, pero el Gateway ya tiene compatibilidad completa hoy.
También están previstas apps complementarias nativas para Windows; se recomienda usar el Gateway mediante WSL2.

## Elige tu SO

- macOS: [macOS](/platforms/macos)
- iOS: [iOS](/platforms/ios)
- Android: [Android](/platforms/android)
- Windows: [Windows](/platforms/windows)
- Linux: [Linux](/platforms/linux)

## VPS y hosting

- Centro de VPS: [VPS hosting](/vps)
- Fly.io: [Fly.io](/install/fly)
- Hetzner (Docker): [Hetzner](/install/hetzner)
- GCP (Compute Engine): [GCP](/install/gcp)
- Azure (Linux VM): [Azure](/install/azure)
- exe.dev (VM + proxy HTTPS): [exe.dev](/install/exe-dev)

## Enlaces comunes

- Guía de instalación: [Getting Started](/es/start/getting-started)
- Runbook del Gateway: [Gateway](/gateway)
- Configuración del Gateway: [Configuration](/gateway/configuration)
- Estado del servicio: `openclaw gateway status`

## Instalación del servicio Gateway (CLI)

Usa una de estas opciones (todas compatibles):

- Asistente (recomendado): `openclaw onboard --install-daemon`
- Directo: `openclaw gateway install`
- Flujo de configuración: `openclaw configure` → selecciona **Gateway service**
- Reparar/migrar: `openclaw doctor` (ofrece instalar o corregir el servicio)

El destino del servicio depende del SO:

- macOS: LaunchAgent (`ai.openclaw.gateway` o `ai.openclaw.<profile>`; heredado `com.openclaw.*`)
- Linux/WSL2: servicio systemd de usuario (`openclaw-gateway[-<profile>].service`)
- Windows nativo: tarea programada (`OpenClaw Gateway` o `OpenClaw Gateway (<profile>)`), con un elemento de inicio de sesión por usuario en la carpeta Inicio como respaldo si se deniega la creación de la tarea
