---
read_when:
    - Desarrollar o depurar clientes de nodo (modo nodo de iOS/Android/macOS)
    - Investigar fallos de emparejamiento o autenticación del bridge
    - Auditar la superficie de nodo expuesta por la gateway
summary: 'Protocolo histórico de bridge (nodos heredados): TCP JSONL, emparejamiento, RPC con alcance'
title: Protocolo de bridge
x-i18n:
    generated_at: "2026-04-05T12:41:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2bc25c388f3d65944167d05ca78f987c84ca480f0213e3485b118ebf4858c50f
    source_path: gateway/bridge-protocol.md
    workflow: 15
---

# Protocolo de bridge (transporte de nodo heredado)

<Warning>
El bridge TCP ha sido **eliminado**. Las compilaciones actuales de OpenClaw no incluyen el listener del bridge y las claves de configuración `bridge.*` ya no están en el esquema. Esta página se conserva solo como referencia histórica. Usa el [Protocolo de Gateway](/gateway/protocol) para todos los clientes de nodo/operador.
</Warning>

## Por qué existía

- **Límite de seguridad**: el bridge expone una pequeña lista de permitidos en lugar de la
  superficie completa de API de la gateway.
- **Emparejamiento + identidad del nodo**: la admisión del nodo es propiedad de la gateway y está vinculada
  a un token por nodo.
- **UX de descubrimiento**: los nodos pueden descubrir gateways mediante Bonjour en LAN, o conectarse
  directamente a través de una tailnet.
- **WS loopback**: todo el plano de control WS permanece local salvo que se tunelice por SSH.

## Transporte

- TCP, un objeto JSON por línea (JSONL).
- TLS opcional (cuando `bridge.tls.enabled` es true).
- El puerto histórico predeterminado del listener era `18790` (las compilaciones actuales no inician un
  bridge TCP).

Cuando TLS está habilitado, los registros TXT de descubrimiento incluyen `bridgeTls=1` más
`bridgeTlsSha256` como pista no secreta. Ten en cuenta que los registros TXT de Bonjour/mDNS no están
autenticados; los clientes no deben tratar la huella anunciada como un pin autorizado sin intención explícita del usuario u otra verificación fuera de banda.

## Handshake + emparejamiento

1. El cliente envía `hello` con metadatos del nodo + token (si ya está emparejado).
2. Si no está emparejado, la gateway responde con `error` (`NOT_PAIRED`/`UNAUTHORIZED`).
3. El cliente envía `pair-request`.
4. La gateway espera la aprobación y luego envía `pair-ok` y `hello-ok`.

Históricamente, `hello-ok` devolvía `serverName` y podía incluir
`canvasHostUrl`.

## Frames

Cliente → Gateway:

- `req` / `res`: RPC de gateway con alcance (chat, sessions, config, health, voicewake, skills.bins)
- `event`: señales del nodo (transcripción de voz, solicitud de agente, suscripción de chat, ciclo de vida de exec)

Gateway → Cliente:

- `invoke` / `invoke-res`: comandos del nodo (`canvas.*`, `camera.*`, `screen.record`,
  `location.get`, `sms.send`)
- `event`: actualizaciones de chat para sesiones suscritas
- `ping` / `pong`: keepalive

La aplicación heredada de la lista de permitidos vivía en `src/gateway/server-bridge.ts` (eliminado).

## Eventos del ciclo de vida de exec

Los nodos pueden emitir eventos `exec.finished` o `exec.denied` para mostrar la actividad de system.run.
Estos se asignan a eventos del sistema en la gateway. (Los nodos heredados aún pueden emitir `exec.started`).

Campos de la carga (todos opcionales salvo que se indique lo contrario):

- `sessionKey` (obligatorio): sesión del agente que recibirá el evento del sistema.
- `runId`: id único de exec para agrupar.
- `command`: cadena de comando sin procesar o con formato.
- `exitCode`, `timedOut`, `success`, `output`: detalles de finalización (solo finished).
- `reason`: motivo de denegación (solo denied).

## Uso histórico de tailnet

- Vincular el bridge a una IP de tailnet: `bridge.bind: "tailnet"` en
  `~/.openclaw/openclaw.json` (solo histórico; `bridge.*` ya no es válido).
- Los clientes se conectan mediante nombre MagicDNS o IP de tailnet.
- Bonjour **no** atraviesa redes; usa host/puerto manual o DNS‑SD de área amplia
  cuando sea necesario.

## Control de versiones

El bridge era un **v1 implícito** (sin negociación min/max). Esta sección es
solo de referencia histórica; los clientes actuales de nodo/operador usan el WebSocket
[Protocolo de Gateway](/gateway/protocol).
