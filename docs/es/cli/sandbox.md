---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: Gestiona runtimes de sandbox e inspecciona la política efectiva de sandbox
title: CLI de Sandbox
x-i18n:
    generated_at: "2026-04-05T12:38:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: fa2783037da2901316108d35e04bb319d5d57963c2764b9146786b3c6474b48a
    source_path: cli/sandbox.md
    workflow: 15
---

# CLI de Sandbox

Gestiona runtimes de sandbox para la ejecución aislada de agentes.

## Descripción general

OpenClaw puede ejecutar agentes en runtimes de sandbox aislados por seguridad. Los comandos `sandbox` te ayudan a inspeccionar y recrear esos runtimes después de actualizaciones o cambios de configuración.

Actualmente eso suele significar:

- Contenedores sandbox de Docker
- Runtimes sandbox SSH cuando `agents.defaults.sandbox.backend = "ssh"`
- Runtimes sandbox de OpenShell cuando `agents.defaults.sandbox.backend = "openshell"`

Para `ssh` y `remote` de OpenShell, recrear es más importante que con Docker:

- el espacio de trabajo remoto es canónico después de la siembra inicial
- `openclaw sandbox recreate` elimina ese espacio de trabajo remoto canónico para el ámbito seleccionado
- el siguiente uso lo vuelve a sembrar desde el espacio de trabajo local actual

## Comandos

### `openclaw sandbox explain`

Inspecciona el modo/ámbito/acceso al espacio de trabajo efectivo de sandbox, la política de herramientas de sandbox y las puertas elevadas (con rutas de claves de configuración para corregirlas).

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

### `openclaw sandbox list`

Enumera todos los runtimes de sandbox con su estado y configuración.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # Enumera solo contenedores de navegador
openclaw sandbox list --json     # Salida JSON
```

**La salida incluye:**

- Nombre y estado del runtime
- Backend (`docker`, `openshell`, etc.)
- Etiqueta de configuración y si coincide con la configuración actual
- Antigüedad (tiempo desde la creación)
- Tiempo de inactividad (tiempo desde el último uso)
- Sesión/agente asociado

### `openclaw sandbox recreate`

Elimina runtimes de sandbox para forzar su recreación con la configuración actualizada.

```bash
openclaw sandbox recreate --all                # Recrear todos los contenedores
openclaw sandbox recreate --session main       # Sesión específica
openclaw sandbox recreate --agent mybot        # Agente específico
openclaw sandbox recreate --browser            # Solo contenedores de navegador
openclaw sandbox recreate --all --force        # Omitir confirmación
```

**Opciones:**

- `--all`: recrea todos los contenedores sandbox
- `--session <key>`: recrea el contenedor para una sesión específica
- `--agent <id>`: recrea los contenedores para un agente específico
- `--browser`: recrea solo los contenedores de navegador
- `--force`: omite la solicitud de confirmación

**Importante:** Los runtimes se recrean automáticamente la próxima vez que se use el agente.

## Casos de uso

### Después de actualizar una imagen de Docker

```bash
# Descargar nueva imagen
docker pull openclaw-sandbox:latest
docker tag openclaw-sandbox:latest openclaw-sandbox:bookworm-slim

# Actualizar la configuración para usar la nueva imagen
# Editar configuración: agents.defaults.sandbox.docker.image (o agents.list[].sandbox.docker.image)

# Recrear contenedores
openclaw sandbox recreate --all
```

### Después de cambiar la configuración de sandbox

```bash
# Editar configuración: agents.defaults.sandbox.* (o agents.list[].sandbox.*)

# Recrear para aplicar la nueva configuración
openclaw sandbox recreate --all
```

### Después de cambiar el destino SSH o el material de autenticación SSH

```bash
# Editar configuración:
# - agents.defaults.sandbox.backend
# - agents.defaults.sandbox.ssh.target
# - agents.defaults.sandbox.ssh.workspaceRoot
# - agents.defaults.sandbox.ssh.identityFile / certificateFile / knownHostsFile
# - agents.defaults.sandbox.ssh.identityData / certificateData / knownHostsData

openclaw sandbox recreate --all
```

Para el backend principal `ssh`, recreate elimina la raíz del espacio de trabajo remoto por ámbito
en el destino SSH. La siguiente ejecución lo vuelve a sembrar desde el espacio de trabajo local.

### Después de cambiar el origen, la política o el modo de OpenShell

```bash
# Editar configuración:
# - agents.defaults.sandbox.backend
# - plugins.entries.openshell.config.from
# - plugins.entries.openshell.config.mode
# - plugins.entries.openshell.config.policy

openclaw sandbox recreate --all
```

Para el modo `remote` de OpenShell, recreate elimina el espacio de trabajo remoto canónico
para ese ámbito. La siguiente ejecución lo vuelve a sembrar desde el espacio de trabajo local.

### Después de cambiar `setupCommand`

```bash
openclaw sandbox recreate --all
# o solo un agente:
openclaw sandbox recreate --agent family
```

### Solo para un agente específico

```bash
# Actualizar solo los contenedores de un agente
openclaw sandbox recreate --agent alfred
```

## ¿Por qué es necesario?

**Problema:** Cuando actualizas la configuración de sandbox:

- Los runtimes existentes siguen ejecutándose con la configuración antigua
- Los runtimes solo se podan después de 24 h de inactividad
- Los agentes que se usan con regularidad mantienen vivos indefinidamente los runtimes antiguos

**Solución:** Usa `openclaw sandbox recreate` para forzar la eliminación de runtimes antiguos. Se recrearán automáticamente con la configuración actual la próxima vez que se necesiten.

Consejo: prefiere `openclaw sandbox recreate` en lugar de una limpieza manual específica del backend.
Usa el registro de runtimes del Gateway y evita discrepancias cuando cambian las claves de ámbito/sesión.

## Configuración

Los ajustes de sandbox se encuentran en `~/.openclaw/openclaw.json` en `agents.defaults.sandbox` (las anulaciones por agente van en `agents.list[].sandbox`):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off, non-main, all
        "backend": "docker", // docker, ssh, openshell
        "scope": "agent", // session, agent, shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... más opciones de Docker
        },
        "prune": {
          "idleHours": 24, // Poda automática después de 24 h inactivo
          "maxAgeDays": 7, // Poda automática después de 7 días
        },
      },
    },
  },
}
```

## Ver también

- [Documentación de Sandbox](/gateway/sandboxing)
- [Configuración del agente](/concepts/agent-workspace)
- [Comando Doctor](/gateway/doctor) - Comprobar la configuración de sandbox
