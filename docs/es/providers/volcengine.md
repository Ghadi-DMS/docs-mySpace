---
read_when:
    - Quieres usar modelos de Volcano Engine o Doubao con OpenClaw
    - Necesitas configurar la clave de API de Volcengine
summary: Configuración de Volcengine (modelos Doubao, endpoints generales + de código)
title: Volcengine (Doubao)
x-i18n:
    generated_at: "2026-04-05T12:52:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 85d9e737e906cd705fb31479d6b78d92b68c9218795ea9667516c1571dcaaf3a
    source_path: providers/volcengine.md
    workflow: 15
---

# Volcengine (Doubao)

El proveedor Volcengine ofrece acceso a modelos Doubao y a modelos de terceros
alojados en Volcano Engine, con endpoints separados para cargas de trabajo
generales y de código.

- Proveedores: `volcengine` (general) + `volcengine-plan` (código)
- Autenticación: `VOLCANO_ENGINE_API_KEY`
- API: compatible con OpenAI

## Inicio rápido

1. Configura la clave de API:

```bash
openclaw onboard --auth-choice volcengine-api-key
```

2. Establece un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "volcengine-plan/ark-code-latest" },
    },
  },
}
```

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

## Proveedores y endpoints

| Proveedor          | Endpoint                                  | Caso de uso       |
| ------------------ | ----------------------------------------- | ----------------- |
| `volcengine`      | `ark.cn-beijing.volces.com/api/v3`        | Modelos generales |
| `volcengine-plan` | `ark.cn-beijing.volces.com/api/coding/v3` | Modelos de código |

Ambos proveedores se configuran a partir de una única clave de API. La configuración registra ambos
automáticamente.

## Modelos disponibles

Proveedor general (`volcengine`):

| Referencia de modelo                          | Nombre                          | Entrada     | Contexto |
| --------------------------------------------- | ------------------------------- | ----------- | -------- |
| `volcengine/doubao-seed-1-8-251228`          | Doubao Seed 1.8                 | text, image | 256,000  |
| `volcengine/doubao-seed-code-preview-251028` | doubao-seed-code-preview-251028 | text, image | 256,000  |
| `volcengine/kimi-k2-5-260127`                | Kimi K2.5                       | text, image | 256,000  |
| `volcengine/glm-4-7-251222`                  | GLM 4.7                         | text, image | 200,000  |
| `volcengine/deepseek-v3-2-251201`            | DeepSeek V3.2                   | text, image | 128,000  |

Proveedor de código (`volcengine-plan`):

| Referencia de modelo                               | Nombre                   | Entrada | Contexto |
| -------------------------------------------------- | ------------------------ | ------- | -------- |
| `volcengine-plan/ark-code-latest`                 | Ark Coding Plan          | text    | 256,000  |
| `volcengine-plan/doubao-seed-code`                | Doubao Seed Code         | text    | 256,000  |
| `volcengine-plan/glm-4.7`                         | GLM 4.7 Coding           | text    | 200,000  |
| `volcengine-plan/kimi-k2-thinking`                | Kimi K2 Thinking         | text    | 256,000  |
| `volcengine-plan/kimi-k2.5`                       | Kimi K2.5 Coding         | text    | 256,000  |
| `volcengine-plan/doubao-seed-code-preview-251028` | Doubao Seed Code Preview | text    | 256,000  |

`openclaw onboard --auth-choice volcengine-api-key` actualmente establece
`volcengine-plan/ark-code-latest` como modelo predeterminado mientras también registra
el catálogo general `volcengine`.

Durante la selección de modelos en onboarding/configure, la opción de autenticación de Volcengine da preferencia
a las filas `volcengine/*` y `volcengine-plan/*`. Si esos modelos aún no
están cargados, OpenClaw recurre al catálogo sin filtrar en lugar de mostrar un
selector vacío limitado por proveedor.

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que
`VOLCANO_ENGINE_API_KEY` esté disponible para ese proceso (por ejemplo, en
`~/.openclaw/.env` o mediante `env.shellEnv`).
