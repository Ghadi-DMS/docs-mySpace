---
read_when:
    - Quieres usar Fireworks con OpenClaw
    - Necesitas la variable de entorno de la clave de API de Fireworks o el ID del modelo predeterminado
summary: Configuración de Fireworks (autenticación + selección de modelo)
x-i18n:
    generated_at: "2026-04-05T12:51:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 20083d5c248abd9a7223e6d188f0265ae27381940ee0067dff6d1d46d908c552
    source_path: providers/fireworks.md
    workflow: 15
---

# Fireworks

[Fireworks](https://fireworks.ai) expone modelos abiertos y enrutados mediante una API compatible con OpenAI. OpenClaw ahora incluye un plugin integrado de proveedor Fireworks.

- Proveedor: `fireworks`
- Autenticación: `FIREWORKS_API_KEY`
- API: chat/completions compatible con OpenAI
- URL base: `https://api.fireworks.ai/inference/v1`
- Modelo predeterminado: `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo`

## Inicio rápido

Configura la autenticación de Fireworks mediante onboarding:

```bash
openclaw onboard --auth-choice fireworks-api-key
```

Esto almacena tu clave de Fireworks en la configuración de OpenClaw y establece el modelo inicial de Fire Pass como predeterminado.

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## Nota sobre el entorno

Si el Gateway se ejecuta fuera de tu shell interactivo, asegúrate de que `FIREWORKS_API_KEY`
también esté disponible para ese proceso. Una clave que exista solo en `~/.profile` no
servirá a un daemon launchd/systemd a menos que ese entorno también se importe allí.

## Catálogo integrado

| Referencia de modelo                                    | Nombre                      | Entrada    | Contexto | Salida máxima | Notas                                         |
| ------------------------------------------------------- | --------------------------- | ---------- | -------- | ------------- | --------------------------------------------- |
| `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo` | Kimi K2.5 Turbo (Fire Pass) | text,image | 256,000  | 256,000       | Modelo inicial integrado predeterminado en Fireworks |

## IDs personalizados de modelos Fireworks

OpenClaw también acepta IDs dinámicos de modelos Fireworks. Usa el ID exacto del modelo o router que muestra Fireworks y antepón `fireworks/`.

Ejemplo:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/routers/kimi-k2p5-turbo",
      },
    },
  },
}
```

Si Fireworks publica un modelo más reciente, como una nueva versión de Qwen o Gemma, puedes cambiar directamente a él usando su ID de modelo de Fireworks sin esperar a una actualización del catálogo integrado.
