---
read_when:
    - Quieres una configuración guiada para gateway, espacio de trabajo, autenticación, canales y Skills
summary: Referencia de CLI para `openclaw onboard` (incorporación interactiva)
title: onboard
x-i18n:
    generated_at: "2026-04-05T12:38:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6db61c8002c9e82e48ff44f72e176b58ad85fad5cb8434687455ed40add8cc2a
    source_path: cli/onboard.md
    workflow: 15
---

# `openclaw onboard`

Incorporación interactiva para la configuración de Gateway local o remoto.

## Guías relacionadas

- Centro de incorporación de CLI: [Onboarding (CLI)](/es/start/wizard)
- Resumen de incorporación: [Onboarding Overview](/start/onboarding-overview)
- Referencia de incorporación de CLI: [CLI Setup Reference](/start/wizard-cli-reference)
- Automatización de CLI: [CLI Automation](/start/wizard-cli-automation)
- Incorporación en macOS: [Onboarding (macOS App)](/start/onboarding)

## Ejemplos

```bash
openclaw onboard
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

Para destinos `ws://` en texto sin cifrar dentro de redes privadas (solo redes de confianza), configura
`OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` en el entorno del proceso de incorporación.

Proveedor personalizado no interactivo:

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai
```

`--custom-api-key` es opcional en modo no interactivo. Si se omite, la incorporación comprueba `CUSTOM_API_KEY`.

Ollama no interactivo:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` usa `http://127.0.0.1:11434` de forma predeterminada. `--custom-model-id` es opcional; si se omite, la incorporación usa los valores predeterminados sugeridos por Ollama. Los IDs de modelo en la nube, como `kimi-k2.5:cloud`, también funcionan aquí.

Almacenar claves de proveedor como referencias en lugar de texto sin formato:

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

Con `--secret-input-mode ref`, la incorporación escribe referencias respaldadas por variables de entorno en lugar de valores de clave en texto sin formato.
Para proveedores respaldados por perfiles de autenticación, esto escribe entradas `keyRef`; para proveedores personalizados, escribe `models.providers.<id>.apiKey` como una referencia de entorno (por ejemplo `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`).

Contrato del modo `ref` no interactivo:

- Configura la variable de entorno del proveedor en el entorno del proceso de incorporación (por ejemplo `OPENAI_API_KEY`).
- No pases indicadores de clave en línea (por ejemplo `--openai-api-key`) a menos que esa variable de entorno también esté configurada.
- Si se pasa un indicador de clave en línea sin la variable de entorno requerida, la incorporación falla de inmediato con instrucciones.

Opciones de token de Gateway en modo no interactivo:

- `--gateway-auth token --gateway-token <token>` almacena un token en texto sin formato.
- `--gateway-auth token --gateway-token-ref-env <name>` almacena `gateway.auth.token` como un SecretRef de entorno.
- `--gateway-token` y `--gateway-token-ref-env` son mutuamente excluyentes.
- `--gateway-token-ref-env` requiere una variable de entorno no vacía en el entorno del proceso de incorporación.
- Con `--install-daemon`, cuando la autenticación por token requiere un token, los tokens de Gateway gestionados por SecretRef se validan pero no se conservan como texto sin formato resuelto en los metadatos del entorno del servicio supervisor.
- Con `--install-daemon`, si el modo token requiere un token y el SecretRef de token configurado no está resuelto, la incorporación falla de forma segura con instrucciones de corrección.
- Con `--install-daemon`, si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está configurado, la incorporación bloquea la instalación hasta que el modo se configure explícitamente.
- La incorporación local escribe `gateway.mode="local"` en la configuración. Si posteriormente falta `gateway.mode` en un archivo de configuración, trátalo como daño en la configuración o una edición manual incompleta, no como un atajo válido de modo local.
- `--allow-unconfigured` es una vía de escape independiente del tiempo de ejecución del gateway. No significa que la incorporación pueda omitir `gateway.mode`.

Ejemplo:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

Estado del gateway local en incorporación no interactiva:

- A menos que pases `--skip-health`, la incorporación espera a que un gateway local accesible esté disponible antes de finalizar correctamente.
- `--install-daemon` inicia primero la ruta de instalación gestionada del gateway. Sin él, ya debes tener un gateway local en ejecución, por ejemplo `openclaw gateway run`.
- Si solo quieres escrituras de configuración, espacio de trabajo o arranque en automatización, usa `--skip-health`.
- En Windows nativo, `--install-daemon` intenta primero usar Tareas programadas y recurre a un elemento de inicio de sesión en la carpeta Startup por usuario si se deniega la creación de tareas.

Comportamiento de incorporación interactiva con modo de referencia:

- Elige **Use secret reference** cuando se te solicite.
- Luego elige una de estas opciones:
  - Variable de entorno
  - Proveedor de secretos configurado (`file` o `exec`)
- La incorporación realiza una validación previa rápida antes de guardar la referencia.
  - Si la validación falla, la incorporación muestra el error y te permite reintentar.

Opciones de endpoint Z.AI no interactivas:

Nota: `--auth-choice zai-api-key` ahora detecta automáticamente el mejor endpoint de Z.AI para tu clave (prefiere la API general con `zai/glm-5`).
Si específicamente quieres los endpoints del GLM Coding Plan, elige `zai-coding-global` o `zai-coding-cn`.

```bash
# Selección de endpoint sin solicitud
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# Otras opciones de endpoint de Z.AI:
# --auth-choice zai-coding-cn
# --auth-choice zai-global
# --auth-choice zai-cn
```

Ejemplo no interactivo de Mistral:

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

Notas sobre los flujos:

- `quickstart`: solicitudes mínimas, genera automáticamente un token de gateway.
- `manual`: solicitudes completas para puerto, bind y autenticación (alias de `advanced`).
- Cuando una opción de autenticación implica un proveedor preferido, la incorporación prefiltra los selectores de modelo predeterminado y lista de permitidos para ese proveedor. En el caso de Volcengine y BytePlus, esto también coincide con las variantes del plan de codificación (`volcengine-plan/*`, `byteplus-plan/*`).
- Si el filtro de proveedor preferido aún no produce modelos cargados, la incorporación recurre al catálogo sin filtrar en lugar de dejar el selector vacío.
- En el paso de búsqueda web, algunos proveedores pueden activar solicitudes de seguimiento específicas del proveedor:
  - **Grok** puede ofrecer configuración opcional de `x_search` con la misma `XAI_API_KEY` y una selección de modelo `x_search`.
  - **Kimi** puede solicitar la región de la API de Moonshot (`api.moonshot.ai` frente a `api.moonshot.cn`) y el modelo predeterminado de búsqueda web de Kimi.
- Comportamiento del alcance de mensajes directos en incorporación local: [CLI Setup Reference](/start/wizard-cli-reference#outputs-and-internals).
- Primer chat más rápido: `openclaw dashboard` (Control UI, sin configuración de canal).
- Proveedor personalizado: conecta cualquier endpoint compatible con OpenAI o Anthropic, incluidos proveedores alojados no listados. Usa Unknown para detección automática.

## Comandos de seguimiento comunes

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` no implica modo no interactivo. Usa `--non-interactive` para scripts.
</Note>
