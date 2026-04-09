---
read_when:
    - Quieres usar Qwen con OpenClaw
    - Antes usabas Qwen OAuth
summary: Usa Qwen Cloud mediante el proveedor qwen empaquetado de OpenClaw
title: Qwen
x-i18n:
    generated_at: "2026-04-09T01:30:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4786df2cb6ec1ab29d191d012c61dcb0e5468bf0f8561fbbb50eed741efad325
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth ha sido eliminado.** La integración OAuth de nivel gratuito
(`qwen-portal`) que usaba endpoints de `portal.qwen.ai` ya no está disponible.
Consulta [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) para
obtener contexto.

</Warning>

## Recomendado: Qwen Cloud

OpenClaw ahora trata Qwen como un proveedor empaquetado de primera clase con el id canónico
`qwen`. El proveedor empaquetado apunta a los endpoints de Qwen Cloud / Alibaba DashScope y
Coding Plan, y mantiene los ids heredados `modelstudio` funcionando como alias de
compatibilidad.

- Proveedor: `qwen`
- Variable de entorno preferida: `QWEN_API_KEY`
- También se aceptan por compatibilidad: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- Estilo de API: compatible con OpenAI

Si quieres `qwen3.6-plus`, prefiere el endpoint **Standard (pay-as-you-go)**.
La compatibilidad con Coding Plan puede ir por detrás del catálogo público.

```bash
# Endpoint global de Coding Plan
openclaw onboard --auth-choice qwen-api-key

# Endpoint de Coding Plan en China
openclaw onboard --auth-choice qwen-api-key-cn

# Endpoint global Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key

# Endpoint Standard (pay-as-you-go) en China
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

Los ids heredados `modelstudio-*` para `auth-choice` y las referencias de modelo `modelstudio/...` siguen
funcionando como alias de compatibilidad, pero los nuevos flujos de configuración deberían preferir los ids canónicos
`qwen-*` para `auth-choice` y las referencias de modelo `qwen/...`.

Después de la configuración inicial, establece un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## Tipos de plan y endpoints

| Plan                       | Región | Auth choice                | Endpoint                                         |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| Standard (pay-as-you-go)   | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)   | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (subscription) | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (subscription) | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

El proveedor selecciona automáticamente el endpoint según tu `auth choice`. Las opciones canónicas
usan la familia `qwen-*`; `modelstudio-*` queda solo para compatibilidad.
Puedes reemplazarlo con un `baseUrl` personalizado en la configuración.

Los endpoints nativos de Model Studio anuncian compatibilidad de uso de streaming en el
transporte compartido `openai-completions`. OpenClaw ahora basa eso en las capacidades del endpoint,
por lo que los ids de proveedor personalizados compatibles con DashScope que apunten a los
mismos hosts nativos heredan el mismo comportamiento de uso de streaming en lugar de
requerir específicamente el id del proveedor integrado `qwen`.

## Obtén tu clave de API

- **Gestionar claves**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **Documentación**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Catálogo integrado

Actualmente, OpenClaw incluye este catálogo Qwen empaquetado. El catálogo configurado
tiene en cuenta el endpoint: las configuraciones de Coding Plan omiten modelos que solo se sabe que funcionan en
el endpoint Standard.

| Referencia de modelo       | Entrada     | Contexto  | Notas                                              |
| -------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`        | text, image | 1,000,000 | Modelo predeterminado                              |
| `qwen/qwen3.6-plus`        | text, image | 1,000,000 | Prefiere endpoints Standard cuando necesites este modelo |
| `qwen/qwen3-max-2026-01-23`| text        | 262,144   | Línea Qwen Max                                     |
| `qwen/qwen3-coder-next`    | text        | 262,144   | Coding                                             |
| `qwen/qwen3-coder-plus`    | text        | 1,000,000 | Coding                                             |
| `qwen/MiniMax-M2.5`        | text        | 1,000,000 | Reasoning habilitado                               |
| `qwen/glm-5`               | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`             | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`           | text, image | 262,144   | Moonshot AI mediante Alibaba                       |

La disponibilidad puede seguir variando según el endpoint y el plan de facturación, incluso cuando un modelo
esté presente en el catálogo empaquetado.

La compatibilidad de uso de streaming nativo se aplica tanto a los hosts de Coding Plan como a los
hosts Standard compatibles con DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Disponibilidad de Qwen 3.6 Plus

`qwen3.6-plus` está disponible en los endpoints Model Studio Standard (pay-as-you-go):

- China: `dashscope.aliyuncs.com/compatible-mode/v1`
- Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Si los endpoints de Coding Plan devuelven un error de "unsupported model" para
`qwen3.6-plus`, cambia a Standard (pay-as-you-go) en lugar del par
endpoint/clave de Coding Plan.

## Plan de capacidades

La extensión `qwen` se está posicionando como la base del proveedor para toda la superficie de Qwen
Cloud, no solo para modelos de coding/texto.

- Modelos de texto/chat: empaquetados ahora
- Llamadas a herramientas, salida estructurada, thinking: heredados del transporte compatible con OpenAI
- Generación de imágenes: planificada en la capa del plugin de proveedor
- Comprensión de imagen/video: empaquetada ahora en el endpoint Standard
- Voz/audio: planificada en la capa del plugin de proveedor
- Embeddings/reranking de memoria: planificados mediante la superficie del adaptador de embeddings
- Generación de video: empaquetada ahora mediante la capacidad compartida de generación de video

## Complementos multimodales

La extensión `qwen` ahora también expone:

- Comprensión de video mediante `qwen-vl-max-latest`
- Generación de video Wan mediante:
  - `wan2.6-t2v` (predeterminado)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

Estas superficies multimodales usan los endpoints DashScope **Standard**, no los
endpoints de Coding Plan.

- URL base Standard global/internacional: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- URL base Standard de China: `https://dashscope.aliyuncs.com/compatible-mode/v1`

Para la generación de video, OpenClaw asigna la región de Qwen configurada al host AIGC de
DashScope correspondiente antes de enviar el trabajo:

- Global/internacional: `https://dashscope-intl.aliyuncs.com`
- China: `https://dashscope.aliyuncs.com`

Eso significa que un `models.providers.qwen.baseUrl` normal que apunte a cualquiera de los
hosts Qwen de Coding Plan o Standard sigue manteniendo la generación de video en el endpoint
regional correcto de video de DashScope.

Para la generación de video, establece explícitamente un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Límites actuales empaquetados de generación de video de Qwen:

- Hasta **1** video de salida por solicitud
- Hasta **1** imagen de entrada
- Hasta **4** videos de entrada
- Hasta **10 segundos** de duración
- Admite `size`, `aspectRatio`, `resolution`, `audio` y `watermark`
- El modo de imagen/video de referencia actualmente requiere **URLs http(s) remotas**. Las
  rutas de archivo locales se rechazan de entrada porque el endpoint de video de DashScope no
  acepta buffers locales cargados para esas referencias.

Consulta [Video Generation](/es/tools/video-generation) para ver los parámetros
compartidos de la herramienta, la selección de proveedor y el comportamiento de conmutación por error.

## Nota sobre el entorno

Si la Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `QWEN_API_KEY` esté
disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).
