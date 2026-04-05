---
read_when:
    - Quieres leer o editar la configuración de forma no interactiva
summary: Referencia de la CLI para `openclaw config` (get/set/unset/file/schema/validate)
title: config
x-i18n:
    generated_at: "2026-04-05T12:38:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: e4de30f41e15297019151ad1a5b306cb331fd5c2beefd5ce5b98fcc51e95f0de
    source_path: cli/config.md
    workflow: 15
---

# `openclaw config`

Helpers de configuración para ediciones no interactivas en `openclaw.json`: valores get/set/unset/file/schema/validate
por ruta e impresión del archivo de configuración activo. Ejecútalo sin un subcomando para
abrir el asistente de configuración (igual que `openclaw configure`).

Opciones raíz:

- `--section <section>`: filtro repetible de sección de configuración guiada cuando ejecutas `openclaw config` sin un subcomando

Secciones guiadas compatibles:

- `workspace`
- `model`
- `web`
- `gateway`
- `daemon`
- `channels`
- `plugins`
- `skills`
- `health`

## Ejemplos

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### `config schema`

Imprime el esquema JSON generado para `openclaw.json` en stdout como JSON.

Qué incluye:

- El esquema de configuración raíz actual, más un campo de cadena `$schema` en la raíz para herramientas de editor
- Metadatos de documentación de campos `title` y `description` usados por la Control UI
- Los nodos de objeto anidado, comodín (`*`) y elemento de array (`[]`) heredan los mismos metadatos `title` / `description` cuando existe documentación de campo coincidente
- Las ramas `anyOf` / `oneOf` / `allOf` también heredan los mismos metadatos de documentación cuando existe documentación de campo coincidente
- Metadatos del esquema de plugins + canales en vivo en el mejor esfuerzo cuando se pueden cargar los manifiestos de tiempo de ejecución
- Un esquema de respaldo limpio incluso cuando la configuración actual no es válida

RPC de tiempo de ejecución relacionado:

- `config.schema.lookup` devuelve una ruta de configuración normalizada con un nodo de esquema superficial (`title`, `description`, `type`, `enum`, `const`, límites comunes), metadatos de sugerencias de UI coincidentes y resúmenes inmediatos de elementos hijo. Úsalo para exploración por ruta en la Control UI o clientes personalizados.

```bash
openclaw config schema
```

Redirígelo a un archivo cuando quieras inspeccionarlo o validarlo con otras herramientas:

```bash
openclaw config schema > openclaw.schema.json
```

### Rutas

Las rutas usan notación de punto o de corchetes:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.list[0].id
```

Usa el índice de la lista de agentes para apuntar a un agente específico:

```bash
openclaw config get agents.list
openclaw config set agents.list[1].tools.exec.node "node-id-or-name"
```

## Valores

Los valores se analizan como JSON5 cuando es posible; de lo contrario, se tratan como cadenas.
Usa `--strict-json` para requerir análisis JSON5. `--json` sigue siendo compatible como alias heredado.

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` imprime el valor sin procesar como JSON en lugar de texto con formato de terminal.

## Modos de `config set`

`openclaw config set` admite cuatro estilos de asignación:

1. Modo de valor: `openclaw config set <path> <value>`
2. Modo constructor de SecretRef:

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN
```

3. Modo constructor de proveedor (solo para la ruta `secrets.providers.<alias>`):

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-timeout-ms 5000
```

4. Modo por lotes (`--batch-json` o `--batch-file`):

```bash
openclaw config set --batch-json '[
  {
    "path": "secrets.providers.default",
    "provider": { "source": "env" }
  },
  {
    "path": "channels.discord.token",
    "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
  }
]'
```

```bash
openclaw config set --batch-file ./config-set.batch.json --dry-run
```

Nota de política:

- Las asignaciones de SecretRef se rechazan en superficies mutables en tiempo de ejecución no compatibles (por ejemplo `hooks.token`, `commands.ownerDisplaySecret`, tokens de webhook de enlace de hilos de Discord y JSON de credenciales de WhatsApp). Consulta [SecretRef Credential Surface](/reference/secretref-credential-surface).

El análisis por lotes siempre usa la carga útil del lote (`--batch-json`/`--batch-file`) como fuente de verdad.
`--strict-json` / `--json` no cambian el comportamiento del análisis por lotes.

El modo JSON path/value sigue siendo compatible tanto para SecretRefs como para proveedores:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

## Flags del constructor de proveedor

Los destinos del constructor de proveedor deben usar `secrets.providers.<alias>` como ruta.

Flags comunes:

- `--provider-source <env|file|exec>`
- `--provider-timeout-ms <ms>` (`file`, `exec`)

Proveedor env (`--provider-source env`):

- `--provider-allowlist <ENV_VAR>` (repetible)

Proveedor file (`--provider-source file`):

- `--provider-path <path>` (obligatorio)
- `--provider-mode <singleValue|json>`
- `--provider-max-bytes <bytes>`

Proveedor exec (`--provider-source exec`):

- `--provider-command <path>` (obligatorio)
- `--provider-arg <arg>` (repetible)
- `--provider-no-output-timeout-ms <ms>`
- `--provider-max-output-bytes <bytes>`
- `--provider-json-only`
- `--provider-env <KEY=VALUE>` (repetible)
- `--provider-pass-env <ENV_VAR>` (repetible)
- `--provider-trusted-dir <path>` (repetible)
- `--provider-allow-insecure-path`
- `--provider-allow-symlink-command`

Ejemplo de proveedor exec endurecido:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## Simulación

Usa `--dry-run` para validar cambios sin escribir `openclaw.json`.

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run

openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

Comportamiento de la simulación:

- Modo constructor: ejecuta comprobaciones de resolubilidad de SecretRef para refs/proveedores modificados.
- Modo JSON (`--strict-json`, `--json` o modo por lotes): ejecuta validación de esquema más comprobaciones de resolubilidad de SecretRef.
- La validación de políticas también se ejecuta para superficies de destino SecretRef no compatibles conocidas.
- Las comprobaciones de política evalúan toda la configuración posterior al cambio, por lo que las escrituras de objetos padre (por ejemplo, establecer `hooks` como objeto) no pueden omitir la validación de superficies no compatibles.
- Las comprobaciones de SecretRef exec se omiten de forma predeterminada durante la simulación para evitar efectos secundarios de comandos.
- Usa `--allow-exec` con `--dry-run` para activar las comprobaciones de SecretRef exec (esto puede ejecutar comandos del proveedor).
- `--allow-exec` es solo para simulación y da error si se usa sin `--dry-run`.

`--dry-run --json` imprime un informe legible por máquina:

- `ok`: si la simulación se aprobó
- `operations`: número de asignaciones evaluadas
- `checks`: si se ejecutaron comprobaciones de esquema/resolubilidad
- `checks.resolvabilityComplete`: si las comprobaciones de resolubilidad se completaron (false cuando se omiten refs exec)
- `refsChecked`: número de refs realmente resueltas durante la simulación
- `skippedExecRefs`: número de refs exec omitidas porque no se configuró `--allow-exec`
- `errors`: fallos estructurados de esquema/resolubilidad cuando `ok=false`

### Forma de salida JSON

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "schema" | "resolvability",
      message: string,
      ref?: string, // present for resolvability errors
    },
  ],
}
```

Ejemplo de éxito:

```json
{
  "ok": true,
  "operations": 1,
  "configPath": "~/.openclaw/openclaw.json",
  "inputModes": ["builder"],
  "checks": {
    "schema": false,
    "resolvability": true,
    "resolvabilityComplete": true
  },
  "refsChecked": 1,
  "skippedExecRefs": 0
}
```

Ejemplo de error:

```json
{
  "ok": false,
  "operations": 1,
  "configPath": "~/.openclaw/openclaw.json",
  "inputModes": ["builder"],
  "checks": {
    "schema": false,
    "resolvability": true,
    "resolvabilityComplete": true
  },
  "refsChecked": 1,
  "skippedExecRefs": 0,
  "errors": [
    {
      "kind": "resolvability",
      "message": "Error: Environment variable \"MISSING_TEST_SECRET\" is not set.",
      "ref": "env:default:MISSING_TEST_SECRET"
    }
  ]
}
```

Si la simulación falla:

- `config schema validation failed`: la forma de tu configuración posterior al cambio no es válida; corrige la ruta/valor o la forma del objeto proveedor/ref.
- `Config policy validation failed: unsupported SecretRef usage`: vuelve a mover esa credencial a una entrada de texto plano/cadena y mantén SecretRefs solo en superficies compatibles.
- `SecretRef assignment(s) could not be resolved`: el proveedor/ref referenciado no se puede resolver actualmente (falta una variable de entorno, puntero de archivo no válido, fallo del proveedor exec o incompatibilidad entre proveedor y fuente).
- `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: la simulación omitió refs exec; vuelve a ejecutar con `--allow-exec` si necesitas validación de resolubilidad exec.
- Para el modo por lotes, corrige las entradas con errores y vuelve a ejecutar `--dry-run` antes de escribir.

## Subcomandos

- `config file`: imprime la ruta del archivo de configuración activo (resuelta desde `OPENCLAW_CONFIG_PATH` o la ubicación predeterminada).

Reinicia el gateway después de las ediciones.

## Validate

Valida la configuración actual frente al esquema activo sin iniciar el
gateway.

```bash
openclaw config validate
openclaw config validate --json
```
