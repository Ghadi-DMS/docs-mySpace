---
read_when:
    - Ejecutar el host de nodo headless
    - Emparejar un nodo que no sea macOS para system.run
summary: Referencia de CLI para `openclaw node` (host de nodo headless)
title: node
x-i18n:
    generated_at: "2026-04-05T12:38:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6123b33ec46f2b85f2c815947435ac91bbe84456165ff0e504453356da55b46d
    source_path: cli/node.md
    workflow: 15
---

# `openclaw node`

Ejecuta un **host de nodo headless** que se conecta al WebSocket de Gateway y expone
`system.run` / `system.which` en esta máquina.

## ¿Por qué usar un host de nodo?

Usa un host de nodo cuando quieras que los agentes **ejecuten comandos en otras máquinas** de tu
red sin instalar allí una app complementaria completa de macOS.

Casos de uso comunes:

- Ejecutar comandos en equipos remotos Linux/Windows (servidores de compilación, máquinas de laboratorio, NAS).
- Mantener la ejecución **aislada** en la gateway, pero delegar ejecuciones aprobadas a otros hosts.
- Proporcionar un destino de ejecución ligero y headless para automatización o nodos de CI.

La ejecución sigue estando protegida por **aprobaciones de ejecución** y listas de permitidos por agente en el
host de nodo, para que puedas mantener el acceso a comandos limitado y explícito.

## Proxy de navegador (configuración cero)

Los hosts de nodo anuncian automáticamente un proxy de navegador si `browser.enabled` no está
deshabilitado en el nodo. Esto permite que el agente use automatización del navegador en ese nodo
sin configuración adicional.

De forma predeterminada, el proxy expone la superficie normal de perfiles del navegador del nodo. Si
estableces `nodeHost.browserProxy.allowProfiles`, el proxy se vuelve restrictivo:
se rechaza el direccionamiento a perfiles no incluidos en la lista de permitidos, y las rutas persistentes de
crear/eliminar perfiles se bloquean a través del proxy.

Desactívalo en el nodo si es necesario:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Ejecutar (primer plano)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Opciones:

- `--host <host>`: host de WebSocket de Gateway (predeterminado: `127.0.0.1`)
- `--port <port>`: puerto de WebSocket de Gateway (predeterminado: `18789`)
- `--tls`: usar TLS para la conexión con la gateway
- `--tls-fingerprint <sha256>`: huella esperada del certificado TLS (sha256)
- `--node-id <id>`: sobrescribir el id del nodo (borra el token de emparejamiento)
- `--display-name <name>`: sobrescribir el nombre para mostrar del nodo

## Autenticación de Gateway para host de nodo

`openclaw node run` y `openclaw node install` resuelven la autenticación de gateway desde configuración/entorno (sin flags `--token`/`--password` en comandos de nodo):

- `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` se comprueban primero.
- Luego fallback de configuración local: `gateway.auth.token` / `gateway.auth.password`.
- En modo local, el host de nodo intencionadamente no hereda `gateway.remote.token` / `gateway.remote.password`.
- Si `gateway.auth.token` / `gateway.auth.password` está configurado explícitamente mediante SecretRef y no se resuelve, la resolución de autenticación del nodo falla de forma cerrada (sin enmascaramiento por fallback remoto).
- En `gateway.mode=remote`, los campos del cliente remoto (`gateway.remote.token` / `gateway.remote.password`) también son válidos según las reglas de precedencia remota.
- La resolución de autenticación del host de nodo solo respeta variables de entorno `OPENCLAW_GATEWAY_*`.

## Servicio (segundo plano)

Instala un host de nodo headless como servicio de usuario.

```bash
openclaw node install --host <gateway-host> --port 18789
```

Opciones:

- `--host <host>`: host de WebSocket de Gateway (predeterminado: `127.0.0.1`)
- `--port <port>`: puerto de WebSocket de Gateway (predeterminado: `18789`)
- `--tls`: usar TLS para la conexión con la gateway
- `--tls-fingerprint <sha256>`: huella esperada del certificado TLS (sha256)
- `--node-id <id>`: sobrescribir el id del nodo (borra el token de emparejamiento)
- `--display-name <name>`: sobrescribir el nombre para mostrar del nodo
- `--runtime <runtime>`: runtime del servicio (`node` o `bun`)
- `--force`: reinstalar/sobrescribir si ya está instalado

Gestiona el servicio:

```bash
openclaw node status
openclaw node stop
openclaw node restart
openclaw node uninstall
```

Usa `openclaw node run` para un host de nodo en primer plano (sin servicio).

Los comandos de servicio aceptan `--json` para salida legible por máquina.

## Emparejamiento

La primera conexión crea una solicitud pendiente de emparejamiento de dispositivo (`role: node`) en la Gateway.
Apruébala mediante:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Si el nodo vuelve a intentar el emparejamiento con detalles de autenticación cambiados (rol/ámbitos/clave pública),
la solicitud pendiente anterior es reemplazada y se crea un nuevo `requestId`.
Ejecuta `openclaw devices list` de nuevo antes de aprobar.

El host de nodo almacena su id de nodo, token, nombre para mostrar y la información de conexión de gateway en
`~/.openclaw/node.json`.

## Aprobaciones de ejecución

`system.run` está protegido por aprobaciones locales de ejecución:

- `~/.openclaw/exec-approvals.json`
- [Exec approvals](/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (editar desde la Gateway)

Para ejecuciones asíncronas aprobadas del nodo, OpenClaw prepara un `systemRunPlan`
canónico antes de solicitar aprobación. El reenvío posterior de `system.run` aprobado reutiliza ese
plan almacenado, por lo que las ediciones en los campos command/cwd/session después de que se creó la solicitud de aprobación
se rechazan en lugar de cambiar lo que ejecuta el nodo.
