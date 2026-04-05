---
read_when:
    - Quieres usar GitHub Copilot como proveedor de modelos
    - Necesitas el flujo `openclaw models auth login-github-copilot`
summary: Inicia sesión en GitHub Copilot desde OpenClaw usando el flujo de dispositivo
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-05T12:51:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92857c119c314e698f922dbdbbc15d21b64d33a25979a2ec0ac1e82e586db6d6
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

## ¿Qué es GitHub Copilot?

GitHub Copilot es el asistente de programación con IA de GitHub. Proporciona acceso a modelos de Copilot
para tu cuenta y plan de GitHub. OpenClaw puede usar Copilot como proveedor de modelos
de dos maneras diferentes.

## Dos formas de usar Copilot en OpenClaw

### 1) Proveedor integrado de GitHub Copilot (`github-copilot`)

Usa el flujo nativo de inicio de sesión por dispositivo para obtener un token de GitHub y luego cámbialo por
tokens de API de Copilot cuando OpenClaw se ejecute. Esta es la ruta **predeterminada** y más sencilla
porque no requiere VS Code.

### 2) Plugin Copilot Proxy (`copilot-proxy`)

Usa la extensión de VS Code **Copilot Proxy** como puente local. OpenClaw se comunica con
el endpoint `/v1` del proxy y usa la lista de modelos que configures allí. Elige esta opción
si ya ejecutas Copilot Proxy en VS Code o necesitas enrutar a través de él.
Debes habilitar el plugin y mantener en ejecución la extensión de VS Code.

Usa GitHub Copilot como proveedor de modelos (`github-copilot`). El comando de inicio de sesión ejecuta
el flujo de dispositivo de GitHub, guarda un perfil de autenticación y actualiza tu configuración para usar ese
perfil.

## Configuración por CLI

```bash
openclaw models auth login-github-copilot
```

Se te pedirá que visites una URL y que introduzcas un código de un solo uso. Mantén la terminal
abierta hasta que se complete.

### Flags opcionales

```bash
openclaw models auth login-github-copilot --yes
```

Para aplicar también en un solo paso el modelo predeterminado recomendado por el proveedor, usa en su lugar
el comando genérico de autenticación:

```bash
openclaw models auth login --provider github-copilot --method device --set-default
```

## Configurar un modelo predeterminado

```bash
openclaw models set github-copilot/gpt-4o
```

### Fragmento de configuración

```json5
{
  agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
}
```

## Notas

- Requiere un TTY interactivo; ejecútalo directamente en una terminal.
- La disponibilidad de modelos de Copilot depende de tu plan; si se rechaza un modelo, prueba
  otro ID (por ejemplo `github-copilot/gpt-4.1`).
- Los ID de modelo de Claude usan automáticamente el transporte Anthropic Messages; los modelos GPT, de la serie o
  y Gemini mantienen el transporte OpenAI Responses.
- El inicio de sesión almacena un token de GitHub en el almacén de perfiles de autenticación y lo intercambia por un
  token de API de Copilot cuando OpenClaw se ejecuta.
