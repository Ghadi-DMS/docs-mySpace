---
read_when:
    - Quieres usar DeepSeek con OpenClaw
    - Necesitas la variable de entorno de API key o la elección de autenticación en CLI
summary: Configuración de DeepSeek (autenticación + selección de modelo)
x-i18n:
    generated_at: "2026-04-05T12:50:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 35f339ca206399496ce094eb8350e0870029ce9605121bcf86c4e9b94f3366c6
    source_path: providers/deepseek.md
    workflow: 15
---

# DeepSeek

[DeepSeek](https://www.deepseek.com) proporciona potentes modelos de IA con una API compatible con OpenAI.

- Proveedor: `deepseek`
- Autenticación: `DEEPSEEK_API_KEY`
- API: compatible con OpenAI
- URL base: `https://api.deepseek.com`

## Inicio rápido

Establece la API key (recomendado: guardarla para la Gateway):

```bash
openclaw onboard --auth-choice deepseek-api-key
```

Esto solicitará tu API key y establecerá `deepseek/deepseek-chat` como modelo predeterminado.

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice deepseek-api-key \
  --deepseek-api-key "$DEEPSEEK_API_KEY" \
  --skip-health \
  --accept-risk
```

## Nota sobre el entorno

Si la Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `DEEPSEEK_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).

## Catálogo integrado

| Ref de modelo                | Nombre            | Entrada | Contexto | Salida máxima | Notas                                               |
| ---------------------------- | ----------------- | ------- | -------- | ------------- | --------------------------------------------------- |
| `deepseek/deepseek-chat`     | DeepSeek Chat     | text    | 131,072  | 8,192         | Modelo predeterminado; superficie sin razonamiento DeepSeek V3.2 |
| `deepseek/deepseek-reasoner` | DeepSeek Reasoner | text    | 131,072  | 65,536        | Superficie V3.2 con razonamiento habilitado         |

Ambos modelos integrados anuncian actualmente compatibilidad con uso de streaming en el código fuente.

Obtén tu API key en [platform.deepseek.com](https://platform.deepseek.com/api_keys).
