---
read_when:
    - Quieres acceso a modelos alojados por OpenCode
    - Quieres elegir entre los catálogos Zen y Go
summary: Usar catálogos OpenCode Zen y Go con OpenClaw
title: OpenCode
x-i18n:
    generated_at: "2026-04-05T12:51:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: c23bc99208d9275afcb1731c28eee250c9f4b7d0578681ace31416135c330865
    source_path: providers/opencode.md
    workflow: 15
---

# OpenCode

OpenCode expone dos catálogos alojados en OpenClaw:

- `opencode/...` para el catálogo **Zen**
- `opencode-go/...` para el catálogo **Go**

Ambos catálogos usan la misma clave API de OpenCode. OpenClaw mantiene separados
los id de proveedor en tiempo de ejecución para que el enrutamiento upstream por modelo siga siendo correcto, pero el onboarding y la documentación los tratan
como una única configuración de OpenCode.

## Configuración por CLI

### Catálogo Zen

```bash
openclaw onboard --auth-choice opencode-zen
openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
```

### Catálogo Go

```bash
openclaw onboard --auth-choice opencode-go
openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
```

## Fragmento de configuración

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## Catálogos

### Zen

- Proveedor en tiempo de ejecución: `opencode`
- Modelos de ejemplo: `opencode/claude-opus-4-6`, `opencode/gpt-5.4`, `opencode/gemini-3-pro`
- Ideal cuando quieres el proxy multmodelo curado de OpenCode

### Go

- Proveedor en tiempo de ejecución: `opencode-go`
- Modelos de ejemplo: `opencode-go/kimi-k2.5`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5`
- Ideal cuando quieres la línea Kimi/GLM/MiniMax alojada por OpenCode

## Notas

- También se admite `OPENCODE_ZEN_API_KEY`.
- Al introducir una clave OpenCode durante la configuración, se almacenan credenciales para ambos proveedores en tiempo de ejecución.
- Inicias sesión en OpenCode, añades los datos de facturación y copias tu clave API.
- La facturación y la disponibilidad del catálogo se gestionan desde el panel de OpenCode.
- Las referencias OpenCode respaldadas por Gemini permanecen en la ruta Gemini por proxy, por lo que OpenClaw mantiene
  allí el saneamiento de firmas de pensamiento de Gemini sin habilitar la
  validación nativa de reproducción Gemini ni las reescrituras de bootstrap.
- Las referencias OpenCode que no son Gemini mantienen la política mínima de reproducción compatible con OpenAI.
