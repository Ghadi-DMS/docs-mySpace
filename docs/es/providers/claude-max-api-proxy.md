---
read_when:
    - Quieres usar la suscripción Claude Max con herramientas compatibles con OpenAI
    - Quieres un servidor API local que envuelva Claude Code CLI
    - Quieres evaluar acceso a Anthropic basado en suscripción frente a clave API
summary: Proxy comunitario para exponer credenciales de suscripción de Claude como un endpoint compatible con OpenAI
title: Claude Max API Proxy
x-i18n:
    generated_at: "2026-04-05T12:51:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2e125a6a46e48371544adf1331137a1db51e93e905b8c44da482cf2fba180a09
    source_path: providers/claude-max-api-proxy.md
    workflow: 15
---

# Claude Max API Proxy

**claude-max-api-proxy** es una herramienta comunitaria que expone tu suscripción Claude Max/Pro como un endpoint API compatible con OpenAI. Esto te permite usar tu suscripción con cualquier herramienta que admita el formato de la API de OpenAI.

<Warning>
Esta ruta es solo de compatibilidad técnica. Anthropic ha bloqueado en el pasado parte del uso de suscripciones
fuera de Claude Code. Debes decidir por tu cuenta si quieres usarla y verificar los términos actuales de Anthropic antes de depender de ella.
</Warning>

## ¿Por qué usar esto?

| Enfoque                 | Costo                                                | Ideal para                                  |
| ----------------------- | ---------------------------------------------------- | ------------------------------------------- |
| API de Anthropic        | Pago por token (~$15/M entrada, $75/M salida para Opus) | Apps de producción, alto volumen         |
| Suscripción Claude Max  | $200/mes tarifa plana                                | Uso personal, desarrollo, uso ilimitado     |

Si tienes una suscripción Claude Max y quieres usarla con herramientas compatibles con OpenAI, este proxy puede reducir costos en algunos flujos de trabajo. Las claves API siguen siendo la vía de política más clara para uso en producción.

## Cómo funciona

```
Your App → claude-max-api-proxy → Claude Code CLI → Anthropic (via subscription)
     (OpenAI format)              (converts format)      (uses your login)
```

El proxy:

1. Acepta solicitudes en formato OpenAI en `http://localhost:3456/v1/chat/completions`
2. Las convierte en comandos de Claude Code CLI
3. Devuelve respuestas en formato OpenAI (con streaming compatible)

## Instalación

```bash
# Requires Node.js 20+ and Claude Code CLI
npm install -g claude-max-api-proxy

# Verify Claude CLI is authenticated
claude --version
```

## Uso

### Iniciar el servidor

```bash
claude-max-api
# Server runs at http://localhost:3456
```

### Probarlo

```bash
# Health check
curl http://localhost:3456/health

# List models
curl http://localhost:3456/v1/models

# Chat completion
curl http://localhost:3456/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### Con OpenClaw

Puedes apuntar OpenClaw al proxy como un endpoint personalizado compatible con OpenAI:

```json5
{
  env: {
    OPENAI_API_KEY: "not-needed",
    OPENAI_BASE_URL: "http://localhost:3456/v1",
  },
  agents: {
    defaults: {
      model: { primary: "openai/claude-opus-4" },
    },
  },
}
```

Esta ruta usa la misma vía de proxy compatible con OpenAI que otros backends
personalizados `/v1`:

- no se aplica la conformación de solicitudes nativa solo de OpenAI
- no hay `service_tier`, ni `store` de Responses, ni sugerencias de caché de prompts, ni
  conformación de carga útil de compatibilidad de razonamiento de OpenAI
- los encabezados de atribución ocultos de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en la URL del proxy

## Modelos disponibles

| ID del modelo      | Se asigna a       |
| ------------------ | ----------------- |
| `claude-opus-4`    | Claude Opus 4     |
| `claude-sonnet-4`  | Claude Sonnet 4   |
| `claude-haiku-4`   | Claude Haiku 4    |

## Inicio automático en macOS

Crea un LaunchAgent para ejecutar el proxy automáticamente:

```bash
cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.claude-max-api</string>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/node</string>
    <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key>
    <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
  </dict>
</dict>
</plist>
EOF

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
```

## Enlaces

- **npm:** [https://www.npmjs.com/package/claude-max-api-proxy](https://www.npmjs.com/package/claude-max-api-proxy)
- **GitHub:** [https://github.com/atalovesyou/claude-max-api-proxy](https://github.com/atalovesyou/claude-max-api-proxy)
- **Issues:** [https://github.com/atalovesyou/claude-max-api-proxy/issues](https://github.com/atalovesyou/claude-max-api-proxy/issues)

## Notas

- Esta es una **herramienta comunitaria**, no compatible oficialmente por Anthropic ni por OpenClaw
- Requiere una suscripción activa de Claude Max/Pro con Claude Code CLI autenticado
- El proxy se ejecuta localmente y no envía datos a servidores de terceros
- Las respuestas en streaming son totalmente compatibles

## Ver también

- [Proveedor Anthropic](/providers/anthropic) - Integración nativa de OpenClaw con Claude CLI o claves API
- [Proveedor OpenAI](/providers/openai) - Para suscripciones de OpenAI/Codex
