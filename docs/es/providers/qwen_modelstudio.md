---
x-i18n:
    generated_at: "2026-04-05T12:52:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1066a1d0acebe4ae3500d18c21f7de07f43b9766daf3d13b098936734e9e7a2b
    source_path: providers/qwen_modelstudio.md
    workflow: 15
---

title: "Qwen / Model Studio"
summary: "Detalle del endpoint para el proveedor integrado qwen y su superficie heredada de compatibilidad con modelstudio"
read_when:

- Quieres detalles a nivel de endpoint para Qwen Cloud / Alibaba DashScope
- Necesitas entender la compatibilidad de variables de entorno para el proveedor qwen
- Quieres usar el endpoint Standard (pago por uso) o Coding Plan

---

# Qwen / Model Studio (Alibaba Cloud)

Esta página documenta la asignación de endpoints detrás del proveedor integrado `qwen`
de OpenClaw. El proveedor mantiene funcionando los IDs de proveedor `modelstudio`, los IDs de auth-choice y las
referencias de modelo como alias de compatibilidad, mientras `qwen` pasa a ser la superficie canónica.

<Info>

Si necesitas **`qwen3.6-plus`**, prefiere **Standard (pago por uso)**. La
disponibilidad de Coding Plan puede ir por detrás del catálogo público de Model Studio, y la
API de Coding Plan puede rechazar un modelo hasta que aparezca en la lista de modelos
compatibles de tu plan.

</Info>

- Proveedor: `qwen` (alias heredado: `modelstudio`)
- Autenticación: `QWEN_API_KEY`
- También aceptado: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- API: compatible con OpenAI

## Inicio rápido

### Standard (pago por uso)

```bash
# Endpoint de China
openclaw onboard --auth-choice qwen-standard-api-key-cn

# Endpoint global/internacional
openclaw onboard --auth-choice qwen-standard-api-key
```

### Coding Plan (suscripción)

```bash
# Endpoint de China
openclaw onboard --auth-choice qwen-api-key-cn

# Endpoint global/internacional
openclaw onboard --auth-choice qwen-api-key
```

Los IDs heredados de auth-choice `modelstudio-*` siguen funcionando como alias de compatibilidad, pero
los IDs canónicos de onboarding son las opciones `qwen-*` mostradas arriba.

Después del onboarding, establece un modelo predeterminado:

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
| Standard (pago por uso)   | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pago por uso)   | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (suscripción) | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (suscripción) | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

El proveedor selecciona automáticamente el endpoint según tu auth choice. Las opciones
canónicas usan la familia `qwen-*`; `modelstudio-*` sigue siendo solo de compatibilidad.
Puedes
anularlo con un `baseUrl` personalizado en la configuración.

Los endpoints nativos de Model Studio anuncian compatibilidad de uso de streaming en el
transporte compartido `openai-completions`. OpenClaw ahora basa eso en las
capacidades del endpoint, por lo que los IDs personalizados de proveedores compatibles con DashScope dirigidos a los
mismos hosts nativos heredan el mismo comportamiento de uso de streaming en lugar de
requerir específicamente el ID integrado del proveedor `qwen`.

## Obtén tu clave de API

- **Gestionar claves**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **Documentación**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Catálogo integrado

Actualmente, OpenClaw incluye este catálogo integrado de Qwen:

| Referencia de modelo       | Entrada     | Contexto  | Notas                                                |
| -------------------------- | ----------- | --------- | ---------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | Modelo predeterminado                                |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | Prefiere endpoints Standard cuando necesites este modelo |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Línea Qwen Max                                       |
| `qwen/qwen3-coder-next`     | text        | 262,144   | Coding                                               |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | Coding                                               |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | Razonamiento habilitado                              |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                  |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                  |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Moonshot AI mediante Alibaba                         |

La disponibilidad todavía puede variar según el endpoint y el plan de facturación, incluso cuando un modelo
está presente en el catálogo integrado.

La compatibilidad de uso de streaming nativo se aplica tanto a los hosts de Coding Plan como
a los hosts Standard compatibles con DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Disponibilidad de Qwen 3.6 Plus

`qwen3.6-plus` está disponible en los endpoints Standard (pago por uso) de Model Studio:

- China: `dashscope.aliyuncs.com/compatible-mode/v1`
- Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Si los endpoints de Coding Plan devuelven un error de "unsupported model" para
`qwen3.6-plus`, cambia a Standard (pago por uso) en lugar del par
endpoint/clave de Coding Plan.

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que
`QWEN_API_KEY` esté disponible para ese proceso (por ejemplo, en
`~/.openclaw/.env` o mediante `env.shellEnv`).
