---
read_when:
    - Quieres usar modelos de Amazon Bedrock con OpenClaw
    - Necesitas configurar credenciales/región de AWS para llamadas a modelos
summary: Usar modelos de Amazon Bedrock (API Converse) con OpenClaw
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-04-05T12:51:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: a751824b679a9340db714ee5227e8d153f38f6c199ca900458a4ec092b4efe54
    source_path: providers/bedrock.md
    workflow: 15
---

# Amazon Bedrock

OpenClaw puede usar modelos de **Amazon Bedrock** mediante el proveedor de streaming **Bedrock Converse**
de pi-ai. La autenticación de Bedrock usa la **cadena predeterminada de credenciales del AWS SDK**,
no una clave API.

## Lo que admite pi-ai

- Proveedor: `amazon-bedrock`
- API: `bedrock-converse-stream`
- Autenticación: credenciales de AWS (variables de entorno, configuración compartida o rol de instancia)
- Región: `AWS_REGION` o `AWS_DEFAULT_REGION` (predeterminado: `us-east-1`)

## Descubrimiento automático de modelos

OpenClaw puede descubrir automáticamente modelos de Bedrock que admiten **streaming**
y **salida de texto**. El descubrimiento usa `bedrock:ListFoundationModels` y
`bedrock:ListInferenceProfiles`, y los resultados se almacenan en caché (predeterminado: 1 hora).

Cómo se habilita el proveedor implícito:

- Si `plugins.entries.amazon-bedrock.config.discovery.enabled` es `true`,
  OpenClaw intentará el descubrimiento incluso cuando no haya ningún marcador de entorno de AWS presente.
- Si `plugins.entries.amazon-bedrock.config.discovery.enabled` no está definido,
  OpenClaw solo agrega automáticamente el
  proveedor implícito de Bedrock cuando ve uno de estos marcadores de autenticación de AWS:
  `AWS_BEARER_TOKEN_BEDROCK`, `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`, o `AWS_PROFILE`.
- La ruta real de autenticación de tiempo de ejecución de Bedrock sigue usando la cadena predeterminada del AWS SDK, por lo que
  la configuración compartida, SSO y la autenticación por rol de instancia IMDS pueden funcionar incluso cuando el descubrimiento
  necesitó `enabled: true` para activarse.

Las opciones de configuración viven en `plugins.entries.amazon-bedrock.config.discovery`:

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          discovery: {
            enabled: true,
            region: "us-east-1",
            providerFilter: ["anthropic", "amazon"],
            refreshInterval: 3600,
            defaultContextWindow: 32000,
            defaultMaxTokens: 4096,
          },
        },
      },
    },
  },
}
```

Notas:

- `enabled` usa por defecto el modo automático. En modo automático, OpenClaw solo habilita el
  proveedor implícito de Bedrock cuando ve un marcador de entorno de AWS compatible.
- `region` usa por defecto `AWS_REGION` o `AWS_DEFAULT_REGION`, y luego `us-east-1`.
- `providerFilter` coincide con nombres de proveedores de Bedrock (por ejemplo `anthropic`).
- `refreshInterval` se expresa en segundos; configúralo en `0` para desactivar el almacenamiento en caché.
- `defaultContextWindow` (predeterminado: `32000`) y `defaultMaxTokens` (predeterminado: `4096`)
  se usan para modelos descubiertos (sobrescríbelos si conoces los límites de tu modelo).
- Para entradas explícitas `models.providers["amazon-bedrock"]`, OpenClaw aún puede
  resolver temprano la autenticación de Bedrock desde marcadores de entorno de AWS como
  `AWS_BEARER_TOKEN_BEDROCK` sin forzar la carga completa de la autenticación de tiempo de ejecución. La
  ruta real de autenticación para llamadas a modelos sigue usando la cadena predeterminada del AWS SDK.

## Onboarding

1. Asegúrate de que las credenciales de AWS estén disponibles en el **host del gateway**:

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
# Opcional:
export AWS_SESSION_TOKEN="..."
export AWS_PROFILE="your-profile"
# Opcional (clave API/token bearer de Bedrock):
export AWS_BEARER_TOKEN_BEDROCK="..."
```

2. Agrega un proveedor Bedrock y un modelo a tu configuración (no se requiere `apiKey`):

```json5
{
  models: {
    providers: {
      "amazon-bedrock": {
        baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
        api: "bedrock-converse-stream",
        auth: "aws-sdk",
        models: [
          {
            id: "us.anthropic.claude-opus-4-6-v1:0",
            name: "Claude Opus 4.6 (Bedrock)",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0" },
    },
  },
}
```

## Roles de instancia EC2

Al ejecutar OpenClaw en una instancia EC2 con un rol IAM adjunto, el AWS SDK
puede usar el servicio de metadatos de instancia (IMDS) para autenticación. Para el descubrimiento
de modelos de Bedrock, OpenClaw solo habilita automáticamente el proveedor implícito a partir de marcadores de entorno de AWS
salvo que configures explícitamente
`plugins.entries.amazon-bedrock.config.discovery.enabled: true`.

Configuración recomendada para hosts respaldados por IMDS:

- Configura `plugins.entries.amazon-bedrock.config.discovery.enabled` en `true`.
- Configura `plugins.entries.amazon-bedrock.config.discovery.region` (o exporta `AWS_REGION`).
- **No** necesitas una clave API falsa.
- Solo necesitas `AWS_PROFILE=default` si específicamente quieres un marcador de entorno
  para el modo automático o para superficies de estado.

```bash
# Recomendado: habilitación explícita del descubrimiento + región
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# Opcional: agrega un marcador de entorno si quieres modo automático sin habilitación explícita
export AWS_PROFILE=default
export AWS_REGION=us-east-1
```

**Permisos IAM requeridos** para el rol de instancia EC2:

- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- `bedrock:ListFoundationModels` (para descubrimiento automático)
- `bedrock:ListInferenceProfiles` (para descubrimiento de perfiles de inferencia)

O adjunta la política gestionada `AmazonBedrockFullAccess`.

## Configuración rápida (ruta de AWS)

```bash
# 1. Crear rol IAM y perfil de instancia
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. Adjuntarlo a tu instancia EC2
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. En la instancia EC2, habilitar el descubrimiento explícitamente
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. Opcional: agregar un marcador de entorno si quieres modo automático sin habilitación explícita
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. Verificar que se descubren los modelos
openclaw models list
```

## Perfiles de inferencia

OpenClaw descubre **perfiles de inferencia regionales y globales** junto con
los modelos foundation. Cuando un perfil se asigna a un modelo foundation conocido, el
perfil hereda las capacidades de ese modelo (ventana de contexto, máximo de tokens,
reasoning, visión) y la región correcta de solicitud de Bedrock se inyecta
automáticamente. Esto significa que los perfiles Claude entre regiones funcionan sin sobrescrituras manuales del proveedor.

Los ID de perfiles de inferencia tienen este aspecto: `us.anthropic.claude-opus-4-6-v1:0` (regional)
o `anthropic.claude-opus-4-6-v1:0` (global). Si el modelo de respaldo ya está
en los resultados del descubrimiento, el perfil hereda todo su conjunto de capacidades;
de lo contrario, se aplican valores predeterminados seguros.

No se necesita ninguna configuración adicional. Siempre que el descubrimiento esté habilitado y el principal IAM
tenga `bedrock:ListInferenceProfiles`, los perfiles aparecen junto a
los modelos foundation en `openclaw models list`.

## Notas

- Bedrock requiere que el **acceso al modelo** esté habilitado en tu cuenta/región de AWS.
- El descubrimiento automático necesita los permisos `bedrock:ListFoundationModels` y
  `bedrock:ListInferenceProfiles`.
- Si dependes del modo automático, configura uno de los marcadores de entorno de autenticación de AWS compatibles en el
  host del gateway. Si prefieres autenticación IMDS/config compartida sin marcadores de entorno, configura
  `plugins.entries.amazon-bedrock.config.discovery.enabled: true`.
- OpenClaw muestra el origen de la credencial en este orden: `AWS_BEARER_TOKEN_BEDROCK`,
  luego `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, luego `AWS_PROFILE`, y por último la
  cadena predeterminada del AWS SDK.
- La compatibilidad con reasoning depende del modelo; consulta la tarjeta del modelo de Bedrock para
  conocer las capacidades actuales.
- Si prefieres un flujo de clave gestionado, también puedes colocar un proxy
  compatible con OpenAI delante de Bedrock y configurarlo como proveedor OpenAI.

## Guardrails

Puedes aplicar [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
a todas las invocaciones de modelos Bedrock agregando un objeto `guardrail` a la
configuración del plugin `amazon-bedrock`. Guardrails te permite aplicar filtrado de contenido,
denegación de temas, filtros de palabras, filtros de información sensible y comprobaciones
de grounding contextual.

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          guardrail: {
            guardrailIdentifier: "abc123", // ID de guardrail o ARN completo
            guardrailVersion: "1", // número de versión o "DRAFT"
            streamProcessingMode: "sync", // opcional: "sync" o "async"
            trace: "enabled", // opcional: "enabled", "disabled" o "enabled_full"
          },
        },
      },
    },
  },
}
```

- `guardrailIdentifier` (obligatorio) acepta un ID de guardrail (por ejemplo `abc123`) o un
  ARN completo (por ejemplo `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`).
- `guardrailVersion` (obligatorio) especifica qué versión publicada usar, o
  `"DRAFT"` para el borrador de trabajo.
- `streamProcessingMode` (opcional) controla si la evaluación de guardrail se ejecuta
  de forma síncrona (`"sync"`) o asíncrona (`"async"`) durante el streaming. Si
  se omite, Bedrock usa su comportamiento predeterminado.
- `trace` (opcional) habilita la salida de trazas de guardrail en la respuesta de la API. Configúralo en
  `"enabled"` o `"enabled_full"` para depuración; omítelo o configúralo en `"disabled"` para
  producción.

El principal IAM usado por el gateway debe tener el permiso `bedrock:ApplyGuardrail`
además de los permisos estándar de invocación.
