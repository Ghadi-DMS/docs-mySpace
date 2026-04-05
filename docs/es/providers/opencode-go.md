---
read_when:
    - Quieres el catálogo OpenCode Go
    - Necesitas las referencias de modelo en tiempo de ejecución para los modelos alojados en Go
summary: Usa el catálogo OpenCode Go con la configuración compartida de OpenCode
title: OpenCode Go
x-i18n:
    generated_at: "2026-04-05T12:51:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8650af7c64220c14bab8c22472fff8bebd7abde253e972b6a11784ad833d321c
    source_path: providers/opencode-go.md
    workflow: 15
---

# OpenCode Go

OpenCode Go es el catálogo Go dentro de [OpenCode](/providers/opencode).
Usa la misma `OPENCODE_API_KEY` que el catálogo Zen, pero mantiene el
id de proveedor en tiempo de ejecución `opencode-go` para que el enrutamiento upstream por modelo siga siendo correcto.

## Modelos compatibles

- `opencode-go/kimi-k2.5`
- `opencode-go/glm-5`
- `opencode-go/minimax-m2.5`

## Configuración de CLI

```bash
openclaw onboard --auth-choice opencode-go
# or non-interactive
openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
```

## Fragmento de configuración

```json5
{
  env: { OPENCODE_API_KEY: "YOUR_API_KEY_HERE" }, // pragma: allowlist secret
  agents: { defaults: { model: { primary: "opencode-go/kimi-k2.5" } } },
}
```

## Comportamiento de enrutamiento

OpenClaw gestiona automáticamente el enrutamiento por modelo cuando la referencia del modelo usa `opencode-go/...`.

## Notas

- Usa [OpenCode](/providers/opencode) para la configuración compartida y el resumen del catálogo.
- Las referencias de tiempo de ejecución siguen siendo explícitas: `opencode/...` para Zen, `opencode-go/...` para Go.
