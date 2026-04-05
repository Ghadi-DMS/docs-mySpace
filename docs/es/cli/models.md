---
read_when:
    - Quieres cambiar los modelos predeterminados o ver el estado de autenticación del proveedor
    - Quieres explorar los modelos/proveedores disponibles y depurar perfiles de autenticación
summary: Referencia de la CLI para `openclaw models` (status/list/set/scan, alias, respaldos, autenticación)
title: models
x-i18n:
    generated_at: "2026-04-05T12:38:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 04ba33181d49b6bbf3b5d5fa413aa6b388c9f29fb9d4952055d68c79f7bcfea0
    source_path: cli/models.md
    workflow: 15
---

# `openclaw models`

Descubrimiento, exploración y configuración de modelos (modelo predeterminado, respaldos, perfiles de autenticación).

Relacionado:

- Proveedores + modelos: [Models](/providers/models)
- Configuración de autenticación del proveedor: [Primeros pasos](/es/start/getting-started)

## Comandos comunes

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models scan
```

`openclaw models status` muestra los valores predeterminados/reservas resueltos junto con un resumen de autenticación.
Cuando hay instantáneas de uso del proveedor disponibles, la sección de estado de OAuth/clave de API incluye
ventanas de uso del proveedor e instantáneas de cuota.
Proveedores actuales con ventana de uso: Anthropic, GitHub Copilot, Gemini CLI, OpenAI
Codex, MiniMax, Xiaomi y z.ai. La autenticación de uso proviene de hooks específicos
del proveedor cuando están disponibles; de lo contrario, OpenClaw recurre a hacer coincidir
credenciales OAuth/clave de API de perfiles de autenticación, entorno o configuración.
Añade `--probe` para ejecutar sondeos de autenticación en vivo contra cada perfil de proveedor configurado.
Los sondeos son solicitudes reales (pueden consumir tokens y activar límites de tasa).
Usa `--agent <id>` para inspeccionar el estado de modelo/autenticación de un agente configurado. Si se omite,
el comando usa `OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR` si está configurado; en caso contrario, usa el
agente predeterminado configurado.
Las filas de sondeo pueden provenir de perfiles de autenticación, credenciales del entorno o `models.json`.

Notas:

- `models set <model-or-alias>` acepta `provider/model` o un alias.
- Las referencias de modelo se analizan dividiendo por la **primera** `/`. Si el ID del modelo incluye `/` (estilo OpenRouter), incluye el prefijo del proveedor (ejemplo: `openrouter/moonshotai/kimi-k2`).
- Si omites el proveedor, OpenClaw resuelve primero la entrada como un alias, luego
  como una coincidencia única de proveedor configurado para ese ID exacto de modelo, y solo entonces
  recurre al proveedor predeterminado configurado con una advertencia de obsolescencia.
  Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw
  recurre al primer proveedor/modelo configurado en lugar de mostrar un
  valor predeterminado obsoleto de un proveedor eliminado.
- `models status` puede mostrar `marker(<value>)` en la salida de autenticación para marcadores de posición no secretos (por ejemplo `OPENAI_API_KEY`, `secretref-managed`, `minimax-oauth`, `oauth:chutes`, `ollama-local`) en lugar de enmascararlos como secretos.

### `models status`

Opciones:

- `--json`
- `--plain`
- `--check` (salida 1=caducado/faltante, 2=próximo a caducar)
- `--probe` (sondeo en vivo de los perfiles de autenticación configurados)
- `--probe-provider <name>` (sondear un proveedor)
- `--probe-profile <id>` (repetible o IDs de perfil separados por comas)
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`
- `--agent <id>` (ID del agente configurado; anula `OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR`)

Categorías de estado del sondeo:

- `ok`
- `auth`
- `rate_limit`
- `billing`
- `timeout`
- `format`
- `unknown`
- `no_model`

Casos de detalle/código de motivo del sondeo que se pueden esperar:

- `excluded_by_auth_order`: existe un perfil almacenado, pero
  `auth.order.<provider>` explícito lo omitió, por lo que el sondeo informa la exclusión en lugar de
  intentarlo.
- `missing_credential`, `invalid_expires`, `expired`, `unresolved_ref`:
  el perfil está presente, pero no es apto/no se puede resolver.
- `no_model`: existe autenticación del proveedor, pero OpenClaw no pudo resolver un
  candidato de modelo apto para sondeo para ese proveedor.

## Alias + respaldos

```bash
openclaw models aliases list
openclaw models fallbacks list
```

## Perfiles de autenticación

```bash
openclaw models auth add
openclaw models auth login --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token
```

`models auth add` es el asistente interactivo de autenticación. Puede iniciar un flujo de autenticación
del proveedor (OAuth/clave de API) o guiarte hacia el pegado manual de tokens, según el
proveedor que elijas.

`models auth login` ejecuta el flujo de autenticación de un plugin de proveedor (OAuth/clave de API). Usa
`openclaw plugins list` para ver qué proveedores están instalados.

Ejemplos:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
openclaw models auth login --provider openai-codex --set-default
```

Notas:

- `login --provider anthropic --method cli --set-default` reutiliza un inicio de sesión local de Claude
  CLI y reescribe la ruta principal del modelo predeterminado de Anthropic a una referencia canónica
  `claude-cli/claude-*`.
- `setup-token` y `paste-token` siguen siendo comandos genéricos de token para proveedores
  que exponen métodos de autenticación por token.
- `setup-token` requiere un TTY interactivo y ejecuta el método de autenticación por token del proveedor
  (usando de forma predeterminada el método `setup-token` de ese proveedor cuando expone
  uno).
- `paste-token` acepta una cadena de token generada en otro lugar o desde automatización.
- `paste-token` requiere `--provider`, solicita el valor del token y lo escribe
  en el ID de perfil predeterminado `<provider>:manual` a menos que pases
  `--profile-id`.
- `paste-token --expires-in <duration>` almacena un vencimiento absoluto del token a partir de una
  duración relativa como `365d` o `12h`.
- Nota de facturación de Anthropic: Creemos que el respaldo de Claude Code CLI probablemente está permitido para automatización local gestionada por el usuario según la documentación pública de la CLI de Anthropic. Dicho esto, la política de Anthropic sobre harness de terceros genera suficiente ambigüedad respecto al uso respaldado por suscripción en productos externos como para que no lo recomendemos para producción. Anthropic también notificó a los usuarios de OpenClaw el **4 de abril de 2026 a las 12:00 PM PT / 8:00 PM BST** que la ruta de inicio de sesión de Claude en **OpenClaw** cuenta como uso de harness de terceros y requiere **Extra Usage** facturado por separado de la suscripción.
- `setup-token` / `paste-token` de Anthropic vuelven a estar disponibles como una ruta heredada/manual de OpenClaw. Úsalos con la expectativa de que Anthropic indicó a los usuarios de OpenClaw que esta ruta requiere **Extra Usage**.
