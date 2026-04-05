---
read_when:
    - Quieres editar aprobaciones de exec desde la CLI
    - Necesitas gestionar allowlists en hosts de gateway o nodo
summary: Referencia de la CLI para `openclaw approvals` (aprobaciones de exec para hosts de gateway o nodo)
title: approvals
x-i18n:
    generated_at: "2026-04-05T12:37:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b2532bfd3e6e6ce43c96a2807df2dd00cb7b4320b77a7dfd09bee0531da610e
    source_path: cli/approvals.md
    workflow: 15
---

# `openclaw approvals`

Gestiona aprobaciones de exec para el **host local**, el **host del gateway** o un **host de nodo**.
De forma predeterminada, los comandos apuntan al archivo local de aprobaciones en disco. Usa `--gateway` para apuntar al gateway, o `--node` para apuntar a un nodo específico.

Alias: `openclaw exec-approvals`

Relacionado:

- Aprobaciones de exec: [Exec approvals](/tools/exec-approvals)
- Nodos: [Nodes](/nodes)

## Comandos comunes

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
```

`openclaw approvals get` ahora muestra la política de exec efectiva para destinos locales, de gateway y de nodo:

- política `tools.exec` solicitada
- política del archivo de aprobaciones del host
- resultado efectivo después de aplicar las reglas de precedencia

La precedencia es intencional:

- el archivo de aprobaciones del host es la fuente de verdad aplicable
- la política `tools.exec` solicitada puede restringir o ampliar la intención, pero el resultado efectivo sigue derivándose de las reglas del host
- `--node` combina el archivo de aprobaciones del host del nodo con la política `tools.exec` del gateway, porque ambas siguen aplicándose en tiempo de ejecución
- si la configuración del gateway no está disponible, la CLI usa como respaldo la instantánea de aprobaciones del nodo y señala que no se pudo calcular la política final de tiempo de ejecución

## Reemplazar aprobaciones desde un archivo

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` acepta JSON5, no solo JSON estricto. Usa `--file` o `--stdin`, no ambos.

## Ejemplo de "nunca preguntar" / YOLO

Para un host que nunca debería detenerse por aprobaciones de exec, establece los valores predeterminados de aprobaciones del host en `full` + `off`:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Variante para nodo:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Esto cambia solo el **archivo de aprobaciones del host**. Para mantener alineada la política solicitada de OpenClaw, configura también:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
```

Por qué `tools.exec.host=gateway` en este ejemplo:

- `host=auto` sigue significando "sandbox cuando esté disponible; en caso contrario, gateway".
- YOLO trata sobre aprobaciones, no sobre enrutamiento.
- Si quieres exec en host incluso cuando hay un sandbox configurado, haz explícita la elección del host con `gateway` o `/exec host=gateway`.

Esto coincide con el comportamiento actual de YOLO predeterminado en host. Endurézcalo si quieres aprobaciones.

## Helpers de allowlist

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## Opciones comunes

`get`, `set` y `allowlist add|remove` admiten:

- `--node <id|name|ip>`
- `--gateway`
- opciones compartidas de RPC de nodo: `--url`, `--token`, `--timeout`, `--json`

Notas sobre destinos:

- sin flags de destino, se usa el archivo local de aprobaciones en disco
- `--gateway` apunta al archivo de aprobaciones del host del gateway
- `--node` apunta a un host de nodo después de resolver id, nombre, IP o prefijo de id

`allowlist add|remove` también admite:

- `--agent <id>` (el valor predeterminado es `*`)

## Notas

- `--node` usa el mismo resolvedor que `openclaw nodes` (id, nombre, ip o prefijo de id).
- `--agent` tiene como valor predeterminado `"*"`, que se aplica a todos los agentes.
- El host del nodo debe anunciar `system.execApprovals.get/set` (app de macOS o host de nodo headless).
- Los archivos de aprobaciones se almacenan por host en `~/.openclaw/exec-approvals.json`.
