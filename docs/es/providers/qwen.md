---
x-i18n:
    generated_at: "2026-04-05T12:51:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 895b701d3a3950ea7482e5e870663ed93e0355e679199ed4622718d588ef18fa
    source_path: providers/qwen.md
    workflow: 15
---

summary: "Usa Qwen Cloud mediante el proveedor qwen integrado de OpenClaw"
read_when:

- Quieres usar Qwen con OpenClaw
- Antes usabas Qwen OAuth
  title: "Qwen"

---

# Qwen

<Warning>

**Qwen OAuth has been removed.** La integración OAuth de nivel gratuito
(`qwen-portal`) que usaba endpoints de `portal.qwen.ai` ya no está disponible.
Consulta [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) para
más contexto.

</Warning>

## Recomendado: Qwen Cloud

OpenClaw ahora trata Qwen como un proveedor integrado de primera clase con el id canónico
`qwen`. El proveedor integrado apunta a los endpoints de Qwen Cloud / Alibaba DashScope y
Coding Plan, y mantiene los id heredados `modelstudio` funcionando como alias de
compatibilidad.

- Proveedor: `qwen`
- Variable de entorno preferida: `QWEN_API_KEY`
- También aceptadas por compatibilidad: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- Estilo de API: compatible con OpenAI

Si quieres `qwen3.6-plus`, prefiere el endpoint **Standard (pay-as-you-go)**.
La compatibilidad de Coding Plan puede quedar rezagada respecto al catálogo público.

```bash
# Endpoint global de Coding Plan
openclaw onboard --auth-choice qwen-api-key

# Endpoint de Coding Plan para China
openclaw onboard --auth-choice qwen-api-key-cn

# Endpoint global Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key

# Endpoint Standard (pay-as-you-go) para China
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

Los id heredados `modelstudio-*` de `auth-choice` y las referencias de modelo `modelstudio/...` siguen
funcionando como alias de compatibilidad, pero los nuevos flujos de configuración deberían preferir los id canónicos
`qwen-*` de `auth-choice` y las referencias `qwen/...` de modelo.

Después del onboarding, configura un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## Plan de capacidades

La extensión `qwen` se está posicionando como el hogar del proveedor para toda la superficie de Qwen
Cloud, no solo para modelos de coding/texto.

- Modelos de texto/chat: integrados ahora
- Llamada a herramientas, salida estructurada, thinking: heredados del transporte compatible con OpenAI
- Generación de imágenes: planificada en la capa de plugin del proveedor
- Entendimiento de imágenes/vídeo: integrado ahora en el endpoint Standard
- Voz/audio: planificado en la capa de plugin del proveedor
- Embeddings/reranking de memoria: planificados mediante la superficie del adaptador de embeddings
- Generación de vídeo: integrada ahora mediante la capacidad compartida de generación de vídeo

## Complementos multimodales

La extensión `qwen` ahora también expone:

- Entendimiento de vídeo mediante `qwen-vl-max-latest`
- Generación de vídeo Wan mediante:
  - `wan2.6-t2v` (predeterminado)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

Estas superficies multimodales usan los endpoints **Standard** de DashScope, no los
endpoints de Coding Plan.

- URL base global/internacional Standard: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- URL base Standard para China: `https://dashscope.aliyuncs.com/compatible-mode/v1`

Para la generación de vídeo, OpenClaw asigna la región Qwen configurada al host
DashScope AIGC correspondiente antes de enviar el trabajo:

- Global/internacional: `https://dashscope-intl.aliyuncs.com`
- China: `https://dashscope.aliyuncs.com`

Eso significa que un `models.providers.qwen.baseUrl` normal que apunte a cualquiera de los hosts
Qwen de Coding Plan o Standard sigue manteniendo la generación de vídeo en el endpoint
regional correcto de vídeo de DashScope.

Para la generación de vídeo, configura explícitamente un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Límites actuales integrados de generación de vídeo de Qwen:

- Hasta **1** vídeo de salida por solicitud
- Hasta **1** imagen de entrada
- Hasta **4** vídeos de entrada
- Hasta **10 segundos** de duración
- Admite `size`, `aspectRatio`, `resolution`, `audio` y `watermark`

Consulta [Qwen / Model Studio](/providers/qwen_modelstudio) para detalles a nivel de endpoint
y notas de compatibilidad.
