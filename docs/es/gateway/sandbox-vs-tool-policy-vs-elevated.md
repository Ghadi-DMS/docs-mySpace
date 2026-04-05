---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Por qué una herramienta está bloqueada: tiempo de ejecución del sandbox, política de allow/deny de herramientas y controles de exec elevado'
title: Sandbox vs política de herramientas vs Elevated
x-i18n:
    generated_at: "2026-04-05T12:42:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8d5ddc1dbf02b89f18d46e5473ff0a29b8a984426fe2db7270c170f2de0cdeac
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 15
---

# Sandbox vs política de herramientas vs Elevated

OpenClaw tiene tres controles relacionados (pero diferentes):

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) decide **dónde se ejecutan las herramientas** (Docker vs host).
2. **Política de herramientas** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) decide **qué herramientas están disponibles/permitidas**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) es una **vía de escape solo para exec** para ejecutar fuera del sandbox cuando estás en sandbox (`gateway` de forma predeterminada, o `node` cuando el destino de exec está configurado en `node`).

## Depuración rápida

Usa el inspector para ver lo que OpenClaw está haciendo _realmente_:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Imprime:

- modo/alcance/acceso al workspace efectivos del sandbox
- si la sesión está actualmente en sandbox (main vs no main)
- allow/deny efectivo de herramientas del sandbox (y si provino de agent/global/default)
- controles de elevated y rutas clave de corrección

## Sandbox: dónde se ejecutan las herramientas

El sandbox se controla mediante `agents.defaults.sandbox.mode`:

- `"off"`: todo se ejecuta en el host.
- `"non-main"`: solo las sesiones no main están en sandbox (sorpresa habitual para grupos/canales).
- `"all"`: todo está en sandbox.

Consulta [Sandboxing](/gateway/sandboxing) para ver la matriz completa (alcance, montajes de workspace, imágenes).

### Bind mounts (comprobación rápida de seguridad)

- `docker.binds` _atraviesa_ el sistema de archivos del sandbox: todo lo que montes será visible dentro del contenedor con el modo que configures (`:ro` o `:rw`).
- El valor predeterminado es lectura-escritura si omites el modo; prefiere `:ro` para código fuente/secretos.
- `scope: "shared"` ignora los binds por agente (solo se aplican los binds globales).
- OpenClaw valida las fuentes de bind dos veces: primero en la ruta de origen normalizada y luego de nuevo después de resolver a través del ancestro existente más profundo. Los escapes por padres con symlink no omiten las comprobaciones de rutas bloqueadas o raíces permitidas.
- Las rutas hoja inexistentes también se comprueban de forma segura. Si `/workspace/alias-out/new-file` se resuelve a través de un padre con symlink hacia una ruta bloqueada o fuera de las raíces permitidas configuradas, el bind se rechaza.
- Vincular `/var/run/docker.sock` entrega efectivamente el control del host al sandbox; hazlo solo intencionadamente.
- El acceso al workspace (`workspaceAccess: "ro"`/`"rw"`) es independiente de los modos de bind.

## Política de herramientas: qué herramientas existen/se pueden invocar

Importan dos capas:

- **Perfil de herramientas**: `tools.profile` y `agents.list[].tools.profile` (allowlist base)
- **Perfil de herramientas por proveedor**: `tools.byProvider[provider].profile` y `agents.list[].tools.byProvider[provider].profile`
- **Política global/por agente de herramientas**: `tools.allow`/`tools.deny` y `agents.list[].tools.allow`/`agents.list[].tools.deny`
- **Política de herramientas por proveedor**: `tools.byProvider[provider].allow/deny` y `agents.list[].tools.byProvider[provider].allow/deny`
- **Política de herramientas del sandbox** (solo se aplica cuando hay sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` y `agents.list[].tools.sandbox.tools.*`

Reglas generales:

- `deny` siempre gana.
- Si `allow` no está vacío, todo lo demás se trata como bloqueado.
- La política de herramientas es la parada estricta: `/exec` no puede sobrescribir una herramienta `exec` denegada.
- `/exec` solo cambia los valores predeterminados de sesión para remitentes autorizados; no concede acceso a herramientas.
  Las claves de herramientas por proveedor aceptan `provider` (por ejemplo `google-antigravity`) o `provider/model` (por ejemplo `openai/gpt-5.4`).

### Grupos de herramientas (atajos)

Las políticas de herramientas (global, agente, sandbox) admiten entradas `group:*` que se expanden a múltiples herramientas:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Grupos disponibles:

- `group:runtime`: `exec`, `process`, `code_execution` (`bash` se acepta como alias de `exec`)
- `group:fs`: `read`, `write`, `edit`, `apply_patch`
- `group:sessions`: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`
- `group:memory`: `memory_search`, `memory_get`
- `group:web`: `web_search`, `x_search`, `web_fetch`
- `group:ui`: `browser`, `canvas`
- `group:automation`: `cron`, `gateway`
- `group:messaging`: `message`
- `group:nodes`: `nodes`
- `group:agents`: `agents_list`
- `group:media`: `image`, `image_generate`, `tts`
- `group:openclaw`: todas las herramientas integradas de OpenClaw (excluye plugins de proveedores)

## Elevated: "ejecutar en host" solo para exec

Elevated **no** concede herramientas adicionales; solo afecta a `exec`.

- Si estás en sandbox, `/elevated on` (o `exec` con `elevated: true`) se ejecuta fuera del sandbox (las aprobaciones aún pueden aplicarse).
- Usa `/elevated full` para omitir las aprobaciones de exec para la sesión.
- Si ya estás ejecutando directamente, elevated es efectivamente una operación sin efecto (aunque sigue estando controlado).
- Elevated **no** está limitado a Skills y **no** sobrescribe allow/deny de herramientas.
- Elevated no concede sobrescrituras arbitrarias entre hosts desde `host=auto`; sigue las reglas normales del destino exec y solo conserva `node` cuando el destino configurado/de sesión ya es `node`.
- `/exec` es independiente de elevated. Solo ajusta los valores predeterminados de exec por sesión para remitentes autorizados.

Controles:

- Habilitación: `tools.elevated.enabled` (y opcionalmente `agents.list[].tools.elevated.enabled`)
- Allowlists de remitentes: `tools.elevated.allowFrom.<provider>` (y opcionalmente `agents.list[].tools.elevated.allowFrom.<provider>`)

Consulta [Elevated Mode](/tools/elevated).

## Correcciones comunes de la "cárcel del sandbox"

### "La herramienta X está bloqueada por la política de herramientas del sandbox"

Claves de corrección (elige una):

- Deshabilita el sandbox: `agents.defaults.sandbox.mode=off` (o por agente `agents.list[].sandbox.mode=off`)
- Permite la herramienta dentro del sandbox:
  - elimínala de `tools.sandbox.tools.deny` (o por agente `agents.list[].tools.sandbox.tools.deny`)
  - o agrégala a `tools.sandbox.tools.allow` (o a la allowlist por agente)

### "Pensé que esto era main, ¿por qué está en sandbox?"

En modo `"non-main"`, las claves de grupo/canal _no_ son main. Usa la clave de sesión main (mostrada por `sandbox explain`) o cambia el modo a `"off"`.

## Ver también

- [Sandboxing](/gateway/sandboxing) -- referencia completa del sandbox (modos, alcances, backends, imágenes)
- [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) -- sobrescrituras por agente y precedencia
- [Elevated Mode](/tools/elevated)
