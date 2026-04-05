---
read_when:
    - Quieres sandboxes gestionados en la nube en lugar de Docker local
    - Estás configurando el plugin de OpenShell
    - Necesitas elegir entre los modos de espacio de trabajo mirror y remote
summary: Usa OpenShell como backend de sandbox gestionado para agentes de OpenClaw
title: OpenShell
x-i18n:
    generated_at: "2026-04-05T12:42:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: aaf9027d0632a70fb86455f8bc46dc908ff766db0eb0cdf2f7df39c715241ead
    source_path: gateway/openshell.md
    workflow: 15
---

# OpenShell

OpenShell es un backend de sandbox gestionado para OpenClaw. En lugar de ejecutar
contenedores Docker localmente, OpenClaw delega el ciclo de vida del sandbox al CLI `openshell`,
que aprovisiona entornos remotos con ejecución de comandos basada en SSH.

El plugin de OpenShell reutiliza el mismo transporte SSH central y el puente de
sistema de archivos remoto que el [backend SSH](/gateway/sandboxing#ssh-backend) genérico. Añade
el ciclo de vida específico de OpenShell (`sandbox create/get/delete`, `sandbox ssh-config`)
y un modo opcional de espacio de trabajo `mirror`.

## Requisitos previos

- El CLI `openshell` instalado y en `PATH` (o configura una ruta personalizada mediante
  `plugins.entries.openshell.config.command`)
- Una cuenta de OpenShell con acceso a sandbox
- OpenClaw Gateway ejecutándose en el host

## Inicio rápido

1. Habilita el plugin y configura el backend de sandbox:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

2. Reinicia el Gateway. En el siguiente turno del agente, OpenClaw crea un
   sandbox de OpenShell y enruta la ejecución de herramientas a través de él.

3. Verifica:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Modos de espacio de trabajo

Esta es la decisión más importante al usar OpenShell.

### `mirror`

Usa `plugins.entries.openshell.config.mode: "mirror"` cuando quieras que el **espacio de trabajo local
siga siendo canónico**.

Comportamiento:

- Antes de `exec`, OpenClaw sincroniza el espacio de trabajo local con el sandbox de OpenShell.
- Después de `exec`, OpenClaw sincroniza el espacio de trabajo remoto de vuelta al espacio de trabajo local.
- Las herramientas de archivos siguen operando a través del puente de sandbox, pero el espacio de trabajo local
  sigue siendo la fuente de verdad entre turnos.

Ideal para:

- Editas archivos localmente fuera de OpenClaw y quieres que esos cambios sean visibles en el
  sandbox automáticamente.
- Quieres que el sandbox de OpenShell se comporte tanto como sea posible como el backend
  Docker.
- Quieres que el espacio de trabajo del host refleje las escrituras del sandbox después de cada turno de exec.

Compensación: coste adicional de sincronización antes y después de cada exec.

### `remote`

Usa `plugins.entries.openshell.config.mode: "remote"` cuando quieras que el
**espacio de trabajo de OpenShell pase a ser canónico**.

Comportamiento:

- Cuando el sandbox se crea por primera vez, OpenClaw inicializa el espacio de trabajo remoto a partir del
  espacio de trabajo local una sola vez.
- Después de eso, `exec`, `read`, `write`, `edit` y `apply_patch` operan
  directamente sobre el espacio de trabajo remoto de OpenShell.
- OpenClaw **no** sincroniza los cambios remotos de vuelta al espacio de trabajo local.
- Las lecturas de medios en tiempo de prompt siguen funcionando porque las herramientas de archivos y medios leen a través
  del puente de sandbox.

Ideal para:

- El sandbox debe vivir principalmente en el lado remoto.
- Quieres una menor sobrecarga de sincronización por turno.
- No quieres que las ediciones locales del host sobrescriban silenciosamente el estado remoto del sandbox.

Importante: si editas archivos en el host fuera de OpenClaw después de la inicialización inicial,
el sandbox remoto **no** ve esos cambios. Usa
`openclaw sandbox recreate` para volver a inicializarlo.

### Elegir un modo

|                          | `mirror`                    | `remote`                  |
| ------------------------ | --------------------------- | ------------------------- |
| **Espacio de trabajo canónico** | Host local                 | OpenShell remoto          |
| **Dirección de sincronización** | Bidireccional (cada exec)  | Inicialización única      |
| **Sobrecarga por turno** | Mayor (subida + descarga)   | Menor (operaciones remotas directas) |
| **¿Las ediciones locales son visibles?** | Sí, en el siguiente exec   | No, hasta recreate        |
| **Ideal para**           | Flujos de trabajo de desarrollo | Agentes de larga duración, CI   |

## Referencia de configuración

Toda la configuración de OpenShell se encuentra en `plugins.entries.openshell.config`:

| Clave                     | Tipo                     | Predeterminado | Descripción                                           |
| ------------------------- | ------------------------ | -------------- | ----------------------------------------------------- |
| `mode`                    | `"mirror"` o `"remote"`  | `"mirror"`     | Modo de sincronización del espacio de trabajo         |
| `command`                 | `string`                 | `"openshell"`  | Ruta o nombre del CLI `openshell`                     |
| `from`                    | `string`                 | `"openclaw"`   | Origen del sandbox para la primera creación           |
| `gateway`                 | `string`                 | —              | Nombre del gateway de OpenShell (`--gateway`)         |
| `gatewayEndpoint`         | `string`                 | —              | URL del endpoint del gateway de OpenShell (`--gateway-endpoint`) |
| `policy`                  | `string`                 | —              | ID de política de OpenShell para la creación del sandbox |
| `providers`               | `string[]`               | `[]`           | Nombres de proveedores que se adjuntan cuando se crea el sandbox |
| `gpu`                     | `boolean`                | `false`        | Solicitar recursos de GPU                             |
| `autoProviders`           | `boolean`                | `true`         | Pasar `--auto-providers` durante `sandbox create`     |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`   | Espacio de trabajo principal con escritura dentro del sandbox |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`     | Ruta de montaje del espacio de trabajo del agente (para acceso de solo lectura) |
| `timeoutSeconds`          | `number`                 | `120`          | Tiempo de espera para operaciones del CLI `openshell` |

La configuración a nivel de sandbox (`mode`, `scope`, `workspaceAccess`) se configura en
`agents.defaults.sandbox` igual que con cualquier backend. Consulta
[Sandboxing](/gateway/sandboxing) para ver la matriz completa.

## Ejemplos

### Configuración remota mínima

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### Modo mirror con GPU

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### OpenShell por agente con gateway personalizado

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## Gestión del ciclo de vida

Los sandboxes de OpenShell se gestionan a través del CLI normal de sandbox:

```bash
# List all sandbox runtimes (Docker + OpenShell)
openclaw sandbox list

# Inspect effective policy
openclaw sandbox explain

# Recreate (deletes remote workspace, re-seeds on next use)
openclaw sandbox recreate --all
```

Para el modo `remote`, **recreate es especialmente importante**: elimina el espacio de trabajo remoto
canónico para ese alcance. El siguiente uso inicializa un espacio de trabajo remoto nuevo a partir del
espacio de trabajo local.

Para el modo `mirror`, recreate principalmente restablece el entorno de ejecución remoto porque
el espacio de trabajo local sigue siendo canónico.

### Cuándo recrear

Recrea después de cambiar cualquiera de estos:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

```bash
openclaw sandbox recreate --all
```

## Limitaciones actuales

- El navegador del sandbox no es compatible con el backend OpenShell.
- `sandbox.docker.binds` no se aplica a OpenShell.
- Los controles de tiempo de ejecución específicos de Docker en `sandbox.docker.*` se aplican solo al
  backend Docker.

## Cómo funciona

1. OpenClaw llama a `openshell sandbox create` (con indicadores `--from`, `--gateway`,
   `--policy`, `--providers`, `--gpu` según la configuración).
2. OpenClaw llama a `openshell sandbox ssh-config <name>` para obtener detalles de conexión
   SSH del sandbox.
3. El núcleo escribe la configuración SSH en un archivo temporal y abre una sesión SSH usando el
   mismo puente de sistema de archivos remoto que el backend SSH genérico.
4. En modo `mirror`: sincroniza de local a remoto antes de exec, ejecuta, y sincroniza de vuelta después de exec.
5. En modo `remote`: inicializa una vez al crear y luego opera directamente sobre el
   espacio de trabajo remoto.

## Ver también

- [Sandboxing](/gateway/sandboxing) -- modos, alcances y comparación de backends
- [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) -- depurar herramientas bloqueadas
- [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) -- anulaciones por agente
- [Sandbox CLI](/cli/sandbox) -- comandos `openclaw sandbox`
