---
read_when:
    - Depurar la autenticación del modelo o la expiración de OAuth
    - Documentar la autenticación o el almacenamiento de credenciales
summary: 'Autenticación de modelos: OAuth, claves API y reutilización de Claude CLI'
title: Autenticación
x-i18n:
    generated_at: "2026-04-05T12:41:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1c0ceee7d10fe8d10345f32889b63425d81773f3a08d8ecd3fd88d965b207ddc
    source_path: gateway/authentication.md
    workflow: 15
---

# Autenticación (proveedores de modelos)

<Note>
Esta página cubre la autenticación de **proveedores de modelos** (claves API, OAuth, reutilización de Claude CLI). Para la autenticación de **conexión del gateway** (token, contraseña, trusted-proxy), consulta [Configuration](/gateway/configuration) y [Trusted Proxy Auth](/gateway/trusted-proxy-auth).
</Note>

OpenClaw admite OAuth y claves API para proveedores de modelos. Para hosts de gateway
siempre activos, las claves API suelen ser la opción más predecible. También se admiten
flujos de suscripción/OAuth cuando encajan con el modelo de cuenta de tu proveedor.

Consulta [/concepts/oauth](/concepts/oauth) para ver el flujo completo de OAuth y el
diseño del almacenamiento.
Para autenticación basada en SecretRef (proveedores `env`/`file`/`exec`), consulta [Administración de secretos](/gateway/secrets).
Para las reglas de elegibilidad/códigos de motivo de credenciales usadas por `models status --probe`, consulta
[Semántica de credenciales de autenticación](/auth-credential-semantics).

## Configuración recomendada (clave API, cualquier proveedor)

Si estás ejecutando un gateway de larga duración, empieza con una clave API para el proveedor
que elijas.
Para Anthropic en concreto, la autenticación con clave API es la vía segura. La reutilización de Claude CLI es
la otra vía compatible de configuración de estilo suscripción.

1. Crea una clave API en la consola de tu proveedor.
2. Colócala en el **host del gateway** (la máquina que ejecuta `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Si el Gateway se ejecuta con systemd/launchd, es preferible poner la clave en
   `~/.openclaw/.env` para que el daemon pueda leerla:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

Luego reinicia el daemon (o reinicia tu proceso Gateway) y vuelve a comprobarlo:

```bash
openclaw models status
openclaw doctor
```

Si prefieres no administrar tú mismo las variables de entorno, el onboarding puede almacenar
claves API para el uso del daemon: `openclaw onboard`.

Consulta [Help](/help) para más detalles sobre la herencia de entorno (`env.shellEnv`,
`~/.openclaw/.env`, systemd/launchd).

## Anthropic: compatibilidad heredada de token

La autenticación con token de configuración de Anthropic sigue disponible en OpenClaw como una
ruta heredada/manual. La documentación pública de Claude Code de Anthropic todavía cubre el uso directo
de Claude Code en terminal bajo planes Claude, pero Anthropic informó por separado a usuarios de
OpenClaw que la ruta de **inicio de sesión de Claude en OpenClaw** cuenta como uso de arnés de terceros
y requiere **Extra Usage** facturado por separado de la
suscripción.

Para la ruta de configuración más clara, usa una clave API de Anthropic o migra a Claude CLI
en el host del gateway.

Entrada manual de token (cualquier proveedor; escribe `auth-profiles.json` + actualiza la configuración):

```bash
openclaw models auth paste-token --provider openrouter
```

También se admiten referencias de perfiles de autenticación para credenciales estáticas:

- las credenciales `api_key` pueden usar `keyRef: { source, provider, id }`
- las credenciales `token` pueden usar `tokenRef: { source, provider, id }`
- los perfiles en modo OAuth no admiten credenciales SecretRef; si `auth.profiles.<id>.mode` se establece en `"oauth"`, se rechaza la entrada `keyRef`/`tokenRef` respaldada por SecretRef para ese perfil.

Comprobación apta para automatización (salida `1` cuando falta/ha expirado, `2` cuando está por expirar):

```bash
openclaw models status --check
```

Sondeos de autenticación en vivo:

```bash
openclaw models status --probe
```

Notas:

- Las filas de sondeo pueden proceder de perfiles de autenticación, credenciales de entorno o `models.json`.
- Si `auth.order.<provider>` explícito omite un perfil almacenado, el sondeo informa
  `excluded_by_auth_order` para ese perfil en lugar de probarlo.
- Si existe autenticación pero OpenClaw no puede resolver un candidato de modelo sondeable para
  ese proveedor, el sondeo informa `status: no_model`.
- Los tiempos de espera por límite de tasa pueden estar limitados por modelo. Un perfil en espera para un
  modelo todavía puede ser utilizable para un modelo hermano en el mismo proveedor.

Los scripts opcionales de operaciones (systemd/Termux) están documentados aquí:
[Scripts de monitorización de autenticación](/help/scripts#auth-monitoring-scripts)

## Anthropic: migración a Claude CLI

Si Claude CLI ya está instalado y con sesión iniciada en el host del gateway, puedes
cambiar una configuración existente de Anthropic al backend de CLI. Esta es una
ruta de migración compatible en OpenClaw para reutilizar un inicio de sesión local de Claude CLI en ese
host.

Requisitos previos:

- `claude` instalado en el host del gateway
- Claude CLI ya ha iniciado sesión allí con `claude auth login`

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

Esto conserva tus perfiles de autenticación existentes de Anthropic para poder revertir, pero cambia la
selección de modelo predeterminada a `claude-cli/...` y añade entradas coincidentes de lista de permitidos de Claude CLI en `agents.defaults.models`.

Verificar:

```bash
openclaw models status
```

Atajo de onboarding:

```bash
openclaw onboard --auth-choice anthropic-cli
```

`openclaw onboard` y `openclaw configure` interactivos siguen prefiriendo Claude CLI
para Anthropic, pero el token de configuración de Anthropic vuelve a estar disponible como una
ruta heredada/manual y debe usarse con la expectativa de facturación de Extra Usage.

## Comprobar el estado de autenticación del modelo

```bash
openclaw models status
openclaw doctor
```

## Comportamiento de rotación de claves API (gateway)

Algunos proveedores admiten reintentar una solicitud con claves alternativas cuando una llamada API
alcanza un límite de tasa del proveedor.

- Orden de prioridad:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (una sola anulación)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Los proveedores de Google también incluyen `GOOGLE_API_KEY` como respaldo adicional.
- La misma lista de claves se desduplica antes de su uso.
- OpenClaw reintenta con la siguiente clave solo para errores de límite de tasa (por ejemplo,
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent
requests`, `ThrottlingException`, `concurrency limit reached`, o
  `workers_ai ... quota limit exceeded`).
- Los errores que no son de límite de tasa no se reintentan con claves alternativas.
- Si todas las claves fallan, se devuelve el error final del último intento.

## Controlar qué credencial se usa

### Por sesión (comando de chat)

Usa `/model <alias-or-id>@<profileId>` para fijar una credencial específica del proveedor para la sesión actual (ejemplo de ids de perfil: `anthropic:default`, `anthropic:work`).

Usa `/model` (o `/model list`) para un selector compacto; usa `/model status` para la vista completa (candidatos + siguiente perfil de autenticación, además de detalles del endpoint del proveedor cuando estén configurados).

### Por agente (anulación de CLI)

Establece una anulación explícita del orden de perfiles de autenticación para un agente (almacenada en el `auth-profiles.json` de ese agente):

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Usa `--agent <id>` para apuntar a un agente específico; omítelo para usar el agente predeterminado configurado.
Cuando depures problemas de orden, `openclaw models status --probe` muestra los perfiles almacenados
omitidos como `excluded_by_auth_order` en lugar de omitirlos silenciosamente.
Cuando depures problemas de cooldown, recuerda que los tiempos de espera por límite de tasa pueden estar vinculados
a un id de modelo en lugar de a todo el perfil del proveedor.

## Solución de problemas

### "No credentials found"

Si falta el perfil de Anthropic, migra esa configuración a Claude CLI o a una clave API
en el **host del gateway** y vuelve a comprobarlo:

```bash
openclaw models status
```

### Token a punto de expirar/expirado

Ejecuta `openclaw models status` para confirmar qué perfil está a punto de expirar. Si falta
o ha expirado un perfil heredado de token de Anthropic, migra esa configuración a Claude CLI
o a una clave API.

## Requisitos de Claude CLI

Solo son necesarios para la ruta de reutilización de Anthropic Claude CLI:

- CLI de Claude Code instalado (comando `claude` disponible)
