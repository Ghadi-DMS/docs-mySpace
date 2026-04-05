---
read_when:
    - Editar contratos IPC o el IPC de la app de barra de menús
summary: Arquitectura IPC de macOS para la app de OpenClaw, el transporte de nodo del gateway y PeekabooBridge
title: IPC de macOS
x-i18n:
    generated_at: "2026-04-05T12:48:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: d0211c334a4a59b71afb29dd7b024778172e529fa618985632d3d11d795ced92
    source_path: platforms/mac/xpc.md
    workflow: 15
---

# Arquitectura IPC de OpenClaw en macOS

**Modelo actual:** un socket Unix local conecta el **servicio de host del nodo** con la **app de macOS** para aprobaciones de exec + `system.run`. Existe una CLI de depuración `openclaw-mac` para comprobaciones de descubrimiento/conexión; las acciones del agente siguen fluyendo por el WebSocket del Gateway y `node.invoke`. La automatización de UI usa PeekabooBridge.

## Objetivos

- Una única instancia de la app GUI que sea propietaria de todo el trabajo orientado a TCC (notificaciones, grabación de pantalla, micrófono, voz, AppleScript).
- Una superficie pequeña para automatización: Gateway + comandos de nodo, más PeekabooBridge para automatización de UI.
- Permisos predecibles: siempre el mismo ID de bundle firmado, iniciado por launchd, para que las concesiones de TCC se mantengan.

## Cómo funciona

### Gateway + transporte de nodo

- La app ejecuta el Gateway (modo local) y se conecta a él como nodo.
- Las acciones del agente se realizan mediante `node.invoke` (por ejemplo `system.run`, `system.notify`, `canvas.*`).

### Servicio de nodo + IPC de la app

- Un servicio de host de nodo sin interfaz se conecta al WebSocket del Gateway.
- Las solicitudes `system.run` se reenvían a la app de macOS mediante un socket Unix local.
- La app ejecuta el comando en el contexto de la UI, muestra un prompt si es necesario y devuelve la salida.

Diagrama (SCI):

```
Agent -> Gateway -> Node Service (WS)
                      |  IPC (UDS + token + HMAC + TTL)
                      v
                  Mac App (UI + TCC + system.run)
```

### PeekabooBridge (automatización de UI)

- La automatización de UI usa un socket UNIX independiente llamado `bridge.sock` y el protocolo JSON de PeekabooBridge.
- Orden de preferencia del host (lado cliente): Peekaboo.app → Claude.app → OpenClaw.app → ejecución local.
- Seguridad: los hosts del puente requieren un TeamID permitido; la vía de escape para el mismo UID solo en DEBUG está protegida por `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (convención de Peekaboo).
- Consulta: [uso de PeekabooBridge](/platforms/mac/peekaboo) para más detalles.

## Flujos operativos

- Reiniciar/reconstruir: `SIGN_IDENTITY="Apple Development: <Developer Name> (<TEAMID>)" scripts/restart-mac.sh`
  - Mata las instancias existentes
  - Compila Swift + empaqueta
  - Escribe/inicializa/activa el LaunchAgent
- Instancia única: la app se cierra pronto si ya se está ejecutando otra instancia con el mismo ID de bundle.

## Notas de endurecimiento

- Es preferible exigir coincidencia de TeamID para todas las superficies privilegiadas.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (solo DEBUG) puede permitir llamadores con el mismo UID para desarrollo local.
- Toda la comunicación permanece solo local; no se exponen sockets de red.
- Los prompts de TCC se originan solo desde el bundle de la app GUI; mantén estable el ID de bundle firmado entre reconstrucciones.
- Endurecimiento de IPC: modo de socket `0600`, token, comprobaciones de UID par, desafío/respuesta HMAC, TTL corto.
