---
read_when:
    - Quieres usar Chutes con OpenClaw
    - Necesitas la ruta de configuración de OAuth o clave de API
    - Quieres conocer el modelo predeterminado, los alias o el comportamiento de descubrimiento
summary: Configuración de Chutes (OAuth o clave de API, descubrimiento de modelos, alias)
title: Chutes
x-i18n:
    generated_at: "2026-04-05T12:50:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: e275f32e7a19fa5b4c64ffabfb4bf116dd5c9ab95bfa25bd3b1a15d15e237674
    source_path: providers/chutes.md
    workflow: 15
---

# Chutes

[Chutes](https://chutes.ai) expone catálogos de modelos de código abierto mediante una
API compatible con OpenAI. OpenClaw admite tanto OAuth en navegador como autenticación directa por clave de API
para el proveedor integrado `chutes`.

- Proveedor: `chutes`
- API: compatible con OpenAI
- URL base: `https://llm.chutes.ai/v1`
- Autenticación:
  - OAuth mediante `openclaw onboard --auth-choice chutes`
  - Clave de API mediante `openclaw onboard --auth-choice chutes-api-key`
  - Variables de entorno en runtime: `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`

## Inicio rápido

### OAuth

```bash
openclaw onboard --auth-choice chutes
```

OpenClaw inicia el flujo del navegador localmente, o muestra una URL + flujo
de pegado de redirección en hosts remotos/sin interfaz. Los tokens OAuth se renuevan automáticamente mediante los perfiles de autenticación de OpenClaw.

Anulaciones opcionales de OAuth:

- `CHUTES_CLIENT_ID`
- `CHUTES_CLIENT_SECRET`
- `CHUTES_OAUTH_REDIRECT_URI`
- `CHUTES_OAUTH_SCOPES`

### Clave de API

```bash
openclaw onboard --auth-choice chutes-api-key
```

Obtén tu clave en
[chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys).

Ambas rutas de autenticación registran el catálogo integrado de Chutes y establecen el modelo predeterminado
en `chutes/zai-org/GLM-4.7-TEE`.

## Comportamiento de descubrimiento

Cuando la autenticación de Chutes está disponible, OpenClaw consulta el catálogo de Chutes con esa
credencial y usa los modelos descubiertos. Si el descubrimiento falla, OpenClaw recurre
a un catálogo estático integrado para que el onboarding y el inicio sigan funcionando.

## Alias predeterminados

OpenClaw también registra tres alias de conveniencia para el catálogo integrado de Chutes:

- `chutes-fast` -> `chutes/zai-org/GLM-4.7-FP8`
- `chutes-pro` -> `chutes/deepseek-ai/DeepSeek-V3.2-TEE`
- `chutes-vision` -> `chutes/chutesai/Mistral-Small-3.2-24B-Instruct-2506`

## Catálogo inicial integrado

El catálogo de respaldo integrado incluye referencias actuales de Chutes como:

- `chutes/zai-org/GLM-4.7-TEE`
- `chutes/zai-org/GLM-5-TEE`
- `chutes/deepseek-ai/DeepSeek-V3.2-TEE`
- `chutes/deepseek-ai/DeepSeek-R1-0528-TEE`
- `chutes/moonshotai/Kimi-K2.5-TEE`
- `chutes/chutesai/Mistral-Small-3.2-24B-Instruct-2506`
- `chutes/Qwen/Qwen3-Coder-Next-TEE`
- `chutes/openai/gpt-oss-120b-TEE`

## Ejemplo de configuración

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-4.7-TEE" },
      models: {
        "chutes/zai-org/GLM-4.7-TEE": { alias: "Chutes GLM 4.7" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

## Notas

- Ayuda sobre OAuth y requisitos de la aplicación de redirección: [Documentación de OAuth de Chutes](https://chutes.ai/docs/sign-in-with-chutes/overview)
- Tanto el descubrimiento con clave de API como con OAuth usan el mismo ID de proveedor `chutes`.
- Los modelos de Chutes se registran como `chutes/<model-id>`.
