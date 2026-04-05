---
read_when:
    - Configurar SecretRef para credenciales de proveedores y referencias en `auth-profiles.json`
    - Operar de forma segura la recarga, auditoría, configuración y aplicación de secretos en producción
    - Comprender el fallo rápido en el arranque, el filtrado de superficies inactivas y el comportamiento de última instantánea válida conocida
summary: 'Gestión de secretos: contrato de SecretRef, comportamiento de instantáneas en tiempo de ejecución y limpieza segura en un solo sentido'
title: Gestión de secretos
x-i18n:
    generated_at: "2026-04-05T12:43:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: b91778cb7801fe24f050c15c0a9dd708dda91cb1ce86096e6bae57ebb6e0d41d
    source_path: gateway/secrets.md
    workflow: 15
---

# Gestión de secretos

OpenClaw admite SecretRef aditivos para que las credenciales compatibles no tengan que almacenarse como texto sin formato en la configuración.

El texto sin formato sigue funcionando. SecretRef es opcional para cada credencial.

## Objetivos y modelo de tiempo de ejecución

Los secretos se resuelven en una instantánea de tiempo de ejecución en memoria.

- La resolución es anticipada durante la activación, no diferida en las rutas de solicitud.
- El arranque falla rápidamente cuando un SecretRef efectivamente activo no puede resolverse.
- La recarga usa intercambio atómico: éxito completo o conservación de la última instantánea válida conocida.
- Las infracciones de política de SecretRef (por ejemplo, perfiles de autenticación en modo OAuth combinados con entrada SecretRef) hacen fallar la activación antes del intercambio de la instantánea de tiempo de ejecución.
- Las solicitudes de tiempo de ejecución leen solo desde la instantánea activa en memoria.
- Después de la primera activación/carga correcta de configuración, las rutas de código de tiempo de ejecución siguen leyendo esa instantánea activa en memoria hasta que una recarga correcta la reemplace.
- Las rutas de entrega saliente también leen de esa instantánea activa (por ejemplo, entrega de respuestas/hilos de Discord y envíos de acciones de Telegram); no vuelven a resolver SecretRef en cada envío.

Esto mantiene las interrupciones de proveedores de secretos fuera de las rutas de solicitud críticas.

## Filtrado de superficies activas

Los SecretRef solo se validan en superficies efectivamente activas.

- Superficies habilitadas: las referencias no resueltas bloquean el arranque/la recarga.
- Superficies inactivas: las referencias no resueltas no bloquean el arranque/la recarga.
- Las referencias inactivas emiten diagnósticos no fatales con el código `SECRETS_REF_IGNORED_INACTIVE_SURFACE`.

Ejemplos de superficies inactivas:

- Entradas de canal/cuenta deshabilitadas.
- Credenciales de canal de nivel superior que ninguna cuenta habilitada hereda.
- Superficies de herramientas/funciones deshabilitadas.
- Claves específicas de proveedor de búsqueda web que no están seleccionadas por `tools.web.search.provider`.
  En modo automático (sin proveedor definido), las claves se consultan por precedencia para la detección automática de proveedor hasta que una se resuelve.
  Después de la selección, las claves de proveedores no seleccionados se tratan como inactivas hasta ser seleccionadas.
- El material de autenticación SSH del sandbox (`agents.defaults.sandbox.ssh.identityData`,
  `certificateData`, `knownHostsData`, más los reemplazos por agente) está activo solo
  cuando el backend efectivo del sandbox es `ssh` para el agente predeterminado o un agente habilitado.
- `gateway.remote.token` / `gateway.remote.password` SecretRef están activos si se cumple una de estas condiciones:
  - `gateway.mode=remote`
  - `gateway.remote.url` está configurado
  - `gateway.tailscale.mode` es `serve` o `funnel`
  - En modo local sin esas superficies remotas:
    - `gateway.remote.token` está activo cuando la autenticación por token puede ganar y no hay token de entorno/autenticación configurado.
    - `gateway.remote.password` está activo solo cuando la autenticación por contraseña puede ganar y no hay contraseña de entorno/autenticación configurada.
- `gateway.auth.token` SecretRef está inactivo para la resolución de autenticación de arranque cuando `OPENCLAW_GATEWAY_TOKEN` está definido, porque la entrada de token de entorno tiene prioridad para ese tiempo de ejecución.

## Diagnósticos de la superficie de autenticación del gateway

Cuando se configura un SecretRef en `gateway.auth.token`, `gateway.auth.password`,
`gateway.remote.token` o `gateway.remote.password`, el arranque/la recarga del gateway registra
explícitamente el estado de la superficie:

- `active`: el SecretRef forma parte de la superficie efectiva de autenticación y debe resolverse.
- `inactive`: el SecretRef se ignora para este tiempo de ejecución porque gana otra superficie de autenticación, o
  porque la autenticación remota está deshabilitada/no activa.

Estas entradas se registran con `SECRETS_GATEWAY_AUTH_SURFACE` e incluyen la razón usada por la
política de superficies activas, para que puedas ver por qué una credencial se trató como activa o inactiva.

## Comprobación previa de referencias en la incorporación

Cuando la incorporación se ejecuta en modo interactivo y eliges almacenamiento con SecretRef, OpenClaw ejecuta una validación previa antes de guardar:

- Referencias de entorno: valida el nombre de la variable de entorno y confirma que un valor no vacío sea visible durante la configuración.
- Referencias de proveedor (`file` o `exec`): valida la selección del proveedor, resuelve `id` y comprueba el tipo de valor resuelto.
- Ruta de reutilización Quickstart: cuando `gateway.auth.token` ya es un SecretRef, la incorporación lo resuelve antes del arranque de sonda/dashboard (para referencias `env`, `file` y `exec`) usando la misma barrera de fallo rápido.

Si la validación falla, la incorporación muestra el error y te permite reintentar.

## Contrato de SecretRef

Usa una única forma de objeto en todas partes:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

### `source: "env"`

```json5
{ source: "env", provider: "default", id: "OPENAI_API_KEY" }
```

Validación:

- `provider` debe coincidir con `^[a-z][a-z0-9_-]{0,63}$`
- `id` debe coincidir con `^[A-Z][A-Z0-9_]{0,127}$`

### `source: "file"`

```json5
{ source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
```

Validación:

- `provider` debe coincidir con `^[a-z][a-z0-9_-]{0,63}$`
- `id` debe ser un puntero JSON absoluto (`/...`)
- Escape RFC6901 en segmentos: `~` => `~0`, `/` => `~1`

### `source: "exec"`

```json5
{ source: "exec", provider: "vault", id: "providers/openai/apiKey" }
```

Validación:

- `provider` debe coincidir con `^[a-z][a-z0-9_-]{0,63}$`
- `id` debe coincidir con `^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`
- `id` no debe contener `.` ni `..` como segmentos de ruta delimitados por barra (por ejemplo, `a/../b` se rechaza)

## Configuración del proveedor

Define los proveedores en `secrets.providers`:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // o "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
    resolution: {
      maxProviderConcurrency: 4,
      maxRefsPerProvider: 512,
      maxBatchBytes: 262144,
    },
  },
}
```

### Proveedor env

- Allowlist opcional mediante `allowlist`.
- Los valores de entorno faltantes o vacíos hacen fallar la resolución.

### Proveedor file

- Lee un archivo local desde `path`.
- `mode: "json"` espera una carga de objeto JSON y resuelve `id` como puntero.
- `mode: "singleValue"` espera el id de referencia `"value"` y devuelve el contenido del archivo.
- La ruta debe pasar comprobaciones de propiedad/permisos.
- Nota de cierre por fallo en Windows: si la verificación de ACL no está disponible para una ruta, la resolución falla. Solo para rutas de confianza, establece `allowInsecurePath: true` en ese proveedor para omitir las comprobaciones de seguridad de la ruta.

### Proveedor exec

- Ejecuta la ruta binaria absoluta configurada, sin shell.
- De forma predeterminada, `command` debe apuntar a un archivo normal (no un symlink).
- Define `allowSymlinkCommand: true` para permitir rutas de comando symlink (por ejemplo, shims de Homebrew). OpenClaw valida la ruta del destino resuelto.
- Combina `allowSymlinkCommand` con `trustedDirs` para rutas del gestor de paquetes (por ejemplo `["/opt/homebrew"]`).
- Admite tiempo de espera, tiempo de espera sin salida, límites de bytes de salida, allowlist de entorno y directorios de confianza.
- Nota de cierre por fallo en Windows: si la verificación de ACL no está disponible para la ruta del comando, la resolución falla. Solo para rutas de confianza, establece `allowInsecurePath: true` en ese proveedor para omitir las comprobaciones de seguridad de la ruta.

Carga útil de la solicitud (stdin):

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

Carga útil de la respuesta (stdout):

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

Errores opcionales por id:

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "message": "not found" } }
}
```

## Ejemplos de integración exec

### 1Password CLI

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // requerido para binarios con symlink de Homebrew
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

### HashiCorp Vault CLI

```json5
{
  secrets: {
    providers: {
      vault_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/vault",
        allowSymlinkCommand: true, // requerido para binarios con symlink de Homebrew
        trustedDirs: ["/opt/homebrew"],
        args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
        passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "vault_openai", id: "value" },
      },
    },
  },
}
```

### `sops`

```json5
{
  secrets: {
    providers: {
      sops_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/sops",
        allowSymlinkCommand: true, // requerido para binarios con symlink de Homebrew
        trustedDirs: ["/opt/homebrew"],
        args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
        passEnv: ["SOPS_AGE_KEY_FILE"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "sops_openai", id: "value" },
      },
    },
  },
}
```

## Variables de entorno del servidor MCP

Las variables de entorno del servidor MCP configuradas mediante `plugins.entries.acpx.config.mcpServers` admiten SecretInput. Esto mantiene las API keys y los tokens fuera de la configuración en texto sin formato:

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

Los valores de cadena en texto sin formato siguen funcionando. Las referencias de plantilla de entorno como `${MCP_SERVER_API_KEY}` y los objetos SecretRef se resuelven durante la activación del gateway antes de que se inicie el proceso del servidor MCP. Como con otras superficies de SecretRef, las referencias no resueltas solo bloquean la activación cuando el plugin `acpx` está efectivamente activo.

## Material de autenticación SSH del sandbox

El backend principal `ssh` del sandbox también admite SecretRef para material de autenticación SSH:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Comportamiento de tiempo de ejecución:

- OpenClaw resuelve estas referencias durante la activación del sandbox, no de forma diferida en cada llamada SSH.
- Los valores resueltos se escriben en archivos temporales con permisos restrictivos y se usan en la configuración SSH generada.
- Si el backend efectivo del sandbox no es `ssh`, estas referencias permanecen inactivas y no bloquean el arranque.

## Superficie de credenciales compatible

Las credenciales compatibles y no compatibles canónicas se enumeran en:

- [SecretRef Credential Surface](/reference/secretref-credential-surface)

Las credenciales generadas en tiempo de ejecución o rotativas y el material de actualización OAuth se excluyen intencionalmente de la resolución SecretRef de solo lectura.

## Comportamiento requerido y precedencia

- Campo sin referencia: sin cambios.
- Campo con referencia: obligatorio en superficies activas durante la activación.
- Si están presentes tanto texto sin formato como referencia, la referencia tiene prioridad en las rutas de precedencia compatibles.
- El centinela de redacción `__OPENCLAW_REDACTED__` está reservado para redacción/restauración interna de configuración y se rechaza como dato literal enviado de configuración.

Señales de advertencia y auditoría:

- `SECRETS_REF_OVERRIDES_PLAINTEXT` (advertencia en tiempo de ejecución)
- `REF_SHADOWED` (hallazgo de auditoría cuando las credenciales de `auth-profiles.json` tienen prioridad sobre las referencias de `openclaw.json`)

Comportamiento de compatibilidad de Google Chat:

- `serviceAccountRef` tiene prioridad sobre `serviceAccount` en texto sin formato.
- El valor en texto sin formato se ignora cuando la referencia hermana está configurada.

## Disparadores de activación

La activación de secretos se ejecuta en:

- Arranque (comprobación previa más activación final)
- Ruta de aplicación en caliente de recarga de configuración
- Ruta de verificación de reinicio de recarga de configuración
- Recarga manual mediante `secrets.reload`
- Comprobación previa de RPC de escritura de configuración del Gateway (`config.set` / `config.apply` / `config.patch`) para la resolubilidad de SecretRef en superficies activas dentro de la carga enviada de configuración antes de conservar las ediciones

Contrato de activación:

- El éxito intercambia la instantánea de forma atómica.
- El fallo en el arranque aborta el arranque del gateway.
- El fallo en la recarga de tiempo de ejecución conserva la última instantánea válida conocida.
- El fallo en la comprobación previa de RPC de escritura rechaza la configuración enviada y mantiene sin cambios tanto la configuración en disco como la instantánea activa de tiempo de ejecución.
- Proporcionar un token de canal explícito por llamada a un helper/herramienta saliente no activa la activación de SecretRef; los puntos de activación siguen siendo arranque, recarga y `secrets.reload` explícito.

## Señales de degradación y recuperación

Cuando la activación en tiempo de recarga falla después de un estado saludable, OpenClaw entra en un estado degradado de secretos.

Evento del sistema de una sola emisión y códigos de registro:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

Comportamiento:

- Degradado: el tiempo de ejecución conserva la última instantánea válida conocida.
- Recuperado: se emite una vez después de la siguiente activación correcta.
- Los fallos repetidos mientras ya está degradado registran advertencias, pero no saturan con eventos.
- El fallo rápido en el arranque no emite eventos de degradación porque el tiempo de ejecución nunca llegó a activarse.

## Resolución en la ruta de comandos

Las rutas de comandos pueden optar por la resolución compatible de SecretRef mediante RPC de instantánea del gateway.

Hay dos comportamientos generales:

- Las rutas de comandos estrictas (por ejemplo las rutas de memoria remota de `openclaw memory` y `openclaw qr --remote` cuando necesita referencias remotas de secreto compartido) leen desde la instantánea activa y fallan rápidamente cuando un SecretRef requerido no está disponible.
- Las rutas de comandos de solo lectura (por ejemplo `openclaw status`, `openclaw status --all`, `openclaw channels status`, `openclaw channels resolve`, `openclaw security audit` y los flujos de reparación de doctor/configuración de solo lectura) también prefieren la instantánea activa, pero se degradan en lugar de abortar cuando un SecretRef dirigido no está disponible en esa ruta de comando.

Comportamiento de solo lectura:

- Cuando el gateway está en ejecución, estos comandos leen primero desde la instantánea activa.
- Si la resolución del gateway es incompleta o el gateway no está disponible, intentan un respaldo local dirigido para la superficie específica del comando.
- Si un SecretRef dirigido sigue sin estar disponible, el comando continúa con salida degradada de solo lectura y diagnósticos explícitos como “configured but unavailable in this command path”.
- Este comportamiento degradado es solo local al comando. No debilita las rutas de arranque, recarga ni envío/autenticación del tiempo de ejecución.

Otras notas:

- La actualización de instantánea después de la rotación de secretos del backend se gestiona con `openclaw secrets reload`.
- Método RPC del Gateway usado por estas rutas de comando: `secrets.resolve`.

## Flujo de auditoría y configuración

Flujo predeterminado del operador:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

### `secrets audit`

Los hallazgos incluyen:

- valores en texto sin formato en reposo (`openclaw.json`, `auth-profiles.json`, `.env` y `agents/*/agent/models.json` generado)
- residuos de encabezados sensibles de proveedores en texto sin formato en entradas generadas de `models.json`
- referencias no resueltas
- sombras de precedencia (`auth-profiles.json` tiene prioridad sobre referencias de `openclaw.json`)
- residuos heredados (`auth.json`, recordatorios OAuth)

Nota sobre exec:

- De forma predeterminada, audit omite las comprobaciones de resolubilidad de SecretRef exec para evitar efectos secundarios del comando.
- Usa `openclaw secrets audit --allow-exec` para ejecutar proveedores exec durante la auditoría.

Nota sobre residuos de encabezados:

- La detección de encabezados sensibles de proveedores se basa en heurísticas de nombres (nombres comunes de encabezados de autenticación/credenciales y fragmentos como `authorization`, `x-api-key`, `token`, `secret`, `password` y `credential`).

### `secrets configure`

Helper interactivo que:

- configura primero `secrets.providers` (`env`/`file`/`exec`, agregar/editar/eliminar)
- te permite seleccionar campos compatibles que contienen secretos en `openclaw.json` más `auth-profiles.json` para el alcance de un agente
- puede crear directamente un nuevo mapeo `auth-profiles.json` en el selector de destino
- captura detalles de SecretRef (`source`, `provider`, `id`)
- ejecuta resolución previa
- puede aplicar inmediatamente

Nota sobre exec:

- La comprobación previa omite las comprobaciones exec de SecretRef a menos que se defina `--allow-exec`.
- Si aplicas directamente desde `configure --apply` y el plan incluye referencias/proveedores exec, mantén `--allow-exec` también para el paso de aplicación.

Modos útiles:

- `openclaw secrets configure --providers-only`
- `openclaw secrets configure --skip-provider-setup`
- `openclaw secrets configure --agent <id>`

Valores predeterminados de aplicación en `configure`:

- limpia credenciales estáticas coincidentes de `auth-profiles.json` para los proveedores objetivo
- limpia entradas heredadas estáticas `api_key` de `auth.json`
- limpia líneas secretas conocidas coincidentes de `<config-dir>/.env`

### `secrets apply`

Aplica un plan guardado:

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
```

Nota sobre exec:

- dry-run omite comprobaciones exec a menos que se defina `--allow-exec`.
- El modo de escritura rechaza planes que contengan referencias/proveedores exec a menos que se defina `--allow-exec`.

Para los detalles estrictos del contrato de destino/ruta y las reglas exactas de rechazo, consulta:

- [Secrets Apply Plan Contract](/gateway/secrets-plan-contract)

## Política de seguridad en un solo sentido

OpenClaw intencionalmente no escribe copias de respaldo de reversión que contengan valores históricos de secretos en texto sin formato.

Modelo de seguridad:

- la comprobación previa debe tener éxito antes del modo de escritura
- la activación en tiempo de ejecución se valida antes de confirmar
- apply actualiza archivos usando reemplazo atómico de archivos y restauración best-effort en caso de fallo

## Notas de compatibilidad de autenticación heredada

Para credenciales estáticas, el tiempo de ejecución ya no depende del almacenamiento heredado en texto sin formato.

- La fuente de credenciales en tiempo de ejecución es la instantánea resuelta en memoria.
- Las entradas heredadas estáticas `api_key` se limpian cuando se detectan.
- El comportamiento de compatibilidad relacionado con OAuth sigue siendo independiente.

## Nota sobre la UI web

Algunas uniones de SecretInput son más fáciles de configurar en modo de editor sin formato que en modo de formulario.

## Documentación relacionada

- Comandos CLI: [secrets](/cli/secrets)
- Detalles del contrato del plan: [Secrets Apply Plan Contract](/gateway/secrets-plan-contract)
- Superficie de credenciales: [SecretRef Credential Surface](/reference/secretref-credential-surface)
- Configuración de autenticación: [Authentication](/gateway/authentication)
- Postura de seguridad: [Security](/gateway/security)
- Precedencia de entorno: [Environment Variables](/help/environment)
