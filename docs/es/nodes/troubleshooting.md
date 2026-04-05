---
read_when:
    - El nodo está conectado, pero fallan las herramientas de cámara/canvas/pantalla/exec
    - Necesitas el modelo mental de emparejamiento de nodos frente a aprobaciones
summary: Resolución de problemas de emparejamiento de nodos, requisitos de primer plano, permisos y fallos de herramientas
title: Resolución de problemas de nodos
x-i18n:
    generated_at: "2026-04-05T12:47:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: c2e431e6a35c482a655e01460bef9fab5d5a5ae7dc46f8f992ee51100f5c937e
    source_path: nodes/troubleshooting.md
    workflow: 15
---

# Resolución de problemas de nodos

Usa esta página cuando un nodo aparezca en status, pero fallen las herramientas del nodo.

## Secuencia de comandos

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Luego ejecuta comprobaciones específicas del nodo:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Señales de buen estado:

- El nodo está conectado y emparejado para el rol `node`.
- `nodes describe` incluye la capacidad que estás llamando.
- Las aprobaciones de exec muestran el modo/lista de permitidos esperados.

## Requisitos de primer plano

`canvas.*`, `camera.*` y `screen.*` solo funcionan en primer plano en nodos de iOS/Android.

Comprobación y corrección rápidas:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

Si ves `NODE_BACKGROUND_UNAVAILABLE`, lleva la app del nodo al primer plano y vuelve a intentarlo.

## Matriz de permisos

| Capacidad                    | iOS                                     | Android                                      | app de nodo de macOS          | Código de fallo típico         |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | ----------------------------- | ------------------------------ |
| `camera.snap`, `camera.clip` | Cámara (+ micrófono para audio del clip) | Cámara (+ micrófono para audio del clip)     | Cámara (+ micrófono para audio del clip) | `*_PERMISSION_REQUIRED`        |
| `screen.record`              | Grabación de pantalla (+ micrófono opcional) | Prompt de captura de pantalla (+ micrófono opcional) | Grabación de pantalla         | `*_PERMISSION_REQUIRED`        |
| `location.get`               | Mientras se usa o siempre (según el modo) | Ubicación en primer plano/segundo plano según el modo | Permiso de ubicación          | `LOCATION_PERMISSION_REQUIRED` |
| `system.run`                 | n/a (ruta del host del nodo)            | n/a (ruta del host del nodo)                 | Requiere aprobaciones de exec | `SYSTEM_RUN_DENIED`            |

## Emparejamiento frente a aprobaciones

Estos son controles diferentes:

1. **Emparejamiento del dispositivo**: ¿puede este nodo conectarse al gateway?
2. **Política del gateway para comandos de nodo**: ¿está permitido el id del comando RPC por `gateway.nodes.allowCommands` / `denyCommands` y los valores predeterminados de la plataforma?
3. **Aprobaciones de exec**: ¿puede este nodo ejecutar localmente un comando de shell específico?

Comprobaciones rápidas:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

Si falta el emparejamiento, aprueba primero el dispositivo del nodo.
Si `nodes describe` no muestra un comando, comprueba la política del gateway para comandos de nodo y si el nodo realmente declaró ese comando al conectarse.
Si el emparejamiento está bien, pero `system.run` falla, corrige las aprobaciones de exec/la lista de permitidos en ese nodo.

El emparejamiento de nodos es un control de identidad/confianza, no una superficie de aprobación por comando. Para `system.run`, la política por nodo vive en el archivo de aprobaciones de exec de ese nodo (`openclaw approvals get --node ...`), no en el registro de emparejamiento del gateway.

Para ejecuciones `host=node` respaldadas por aprobación, el gateway también vincula la ejecución al `systemRunPlan` canónico preparado. Si una llamada posterior muta el comando/cwd o los metadatos de sesión antes de que la ejecución aprobada se reenvíe, el gateway rechaza la ejecución como discrepancia de aprobación en lugar de confiar en la carga útil editada.

## Códigos de error habituales del nodo

- `NODE_BACKGROUND_UNAVAILABLE` → la app está en segundo plano; llévala al primer plano.
- `CAMERA_DISABLED` → el interruptor de cámara está desactivado en la configuración del nodo.
- `*_PERMISSION_REQUIRED` → falta un permiso del sistema operativo o fue denegado.
- `LOCATION_DISABLED` → el modo de ubicación está desactivado.
- `LOCATION_PERMISSION_REQUIRED` → no se concedió el modo de ubicación solicitado.
- `LOCATION_BACKGROUND_UNAVAILABLE` → la app está en segundo plano, pero solo existe permiso Mientras se usa.
- `SYSTEM_RUN_DENIED: approval required` → la solicitud de exec necesita aprobación explícita.
- `SYSTEM_RUN_DENIED: allowlist miss` → el comando está bloqueado por el modo de lista de permitidos.
  En hosts de nodos Windows, formas con envoltorio de shell como `cmd.exe /c ...` se tratan como fallos de lista de permitidos en
  modo allowlist, salvo que se aprueben mediante el flujo ask.

## Bucle rápido de recuperación

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

Si sigues atascado:

- Vuelve a aprobar el emparejamiento del dispositivo.
- Vuelve a abrir la app del nodo (primer plano).
- Vuelve a conceder permisos del SO.
- Recrea/ajusta la política de aprobación de exec.

Relacionado:

- [/nodes/index](/nodes/index)
- [/nodes/camera](/nodes/camera)
- [/nodes/location-command](/nodes/location-command)
- [/tools/exec-approvals](/tools/exec-approvals)
- [/gateway/pairing](/gateway/pairing)
