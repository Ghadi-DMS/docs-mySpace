---
read_when:
    - Implementando aprobaciones de pairing de nodos sin UI de macOS
    - Agregando flujos de CLI para aprobar nodos remotos
    - Extendiendo el protocolo del gateway con gestión de nodos
summary: Pairing de nodos gestionado por el Gateway (Opción B) para iOS y otros nodos remotos
title: Pairing gestionado por el Gateway
x-i18n:
    generated_at: "2026-04-05T12:42:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8f90818c84daeb190f27df7413e23362372806f2c4250e4954295fbf6df70233
    source_path: gateway/pairing.md
    workflow: 15
---

# Pairing gestionado por el Gateway (Opción B)

En el pairing gestionado por el Gateway, el **Gateway** es la fuente de verdad sobre qué nodos
pueden unirse. Las UI (app de macOS, futuros clientes) son solo frontends que
aprueban o rechazan solicitudes pendientes.

**Importante:** los nodos WS usan **pairing de dispositivo** (rol `node`) durante `connect`.
`node.pair.*` es un almacén de pairing independiente y **no** controla el handshake de WS.
Solo los clientes que llaman explícitamente a `node.pair.*` usan este flujo.

## Conceptos

- **Solicitud pendiente**: un nodo pidió unirse; requiere aprobación.
- **Nodo emparejado**: nodo aprobado con un token de autenticación emitido.
- **Transporte**: el endpoint WS del Gateway reenvía solicitudes, pero no decide
  la pertenencia. (Se eliminó la compatibilidad heredada con puente TCP).

## Cómo funciona el pairing

1. Un nodo se conecta al WS del Gateway y solicita pairing.
2. El Gateway almacena una **solicitud pendiente** y emite `node.pair.requested`.
3. Apruebas o rechazas la solicitud (CLI o UI).
4. Al aprobarla, el Gateway emite un **nuevo token** (los tokens se rotan al volver a emparejar).
5. El nodo se reconecta usando el token y ahora está “emparejado”.

Las solicitudes pendientes caducan automáticamente después de **5 minutos**.

## Flujo de trabajo de la CLI (compatible con headless)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` muestra los nodos emparejados/conectados y sus capacidades.

## Superficie de API (protocolo del gateway)

Eventos:

- `node.pair.requested` — se emite cuando se crea una nueva solicitud pendiente.
- `node.pair.resolved` — se emite cuando una solicitud se aprueba/rechaza/caduca.

Métodos:

- `node.pair.request` — crea o reutiliza una solicitud pendiente.
- `node.pair.list` — lista nodos pendientes + emparejados (`operator.pairing`).
- `node.pair.approve` — aprueba una solicitud pendiente (emite un token).
- `node.pair.reject` — rechaza una solicitud pendiente.
- `node.pair.verify` — verifica `{ nodeId, token }`.

Notas:

- `node.pair.request` es idempotente por nodo: las llamadas repetidas devuelven la misma
  solicitud pendiente.
- Las solicitudes repetidas para el mismo nodo pendiente también actualizan los metadatos almacenados del nodo
  y la instantánea más reciente de comandos declarados en allowlist para visibilidad del operador.
- La aprobación **siempre** genera un token nuevo; nunca se devuelve un token desde
  `node.pair.request`.
- Las solicitudes pueden incluir `silent: true` como indicación para flujos de autoaprobación.
- `node.pair.approve` usa los comandos declarados de la solicitud pendiente para aplicar
  alcances de aprobación adicionales:
  - solicitud sin comandos: `operator.pairing`
  - solicitud de comandos sin exec: `operator.pairing` + `operator.write`
  - solicitud `system.run` / `system.run.prepare` / `system.which`:
    `operator.pairing` + `operator.admin`

Importante:

- El pairing de nodos es un flujo de confianza/identidad más emisión de tokens.
- **No** fija la superficie de comandos live del nodo por nodo.
- Los comandos live del nodo provienen de lo que el nodo declara en la conexión después de aplicar la
  política global de comandos de nodo del gateway (`gateway.nodes.allowCommands` /
  `denyCommands`).
- La política `system.run` de permitir/preguntar por nodo reside en el nodo, en
  `exec.approvals.node.*`, no en el registro de pairing.

## Control de comandos del nodo (2026.3.31+)

<Warning>
**Cambio incompatible:** a partir de `2026.3.31`, los comandos de nodo están deshabilitados hasta que se apruebe el pairing del nodo. El pairing de dispositivo por sí solo ya no basta para exponer los comandos declarados del nodo.
</Warning>

Cuando un nodo se conecta por primera vez, el pairing se solicita automáticamente. Hasta que se apruebe la solicitud de pairing, todos los comandos pendientes de ese nodo se filtran y no se ejecutarán. Una vez establecida la confianza mediante la aprobación del pairing, los comandos declarados del nodo pasan a estar disponibles, sujetos a la política normal de comandos.

Esto significa:

- Los nodos que antes dependían solo del pairing de dispositivo para exponer comandos ahora deben completar el pairing del nodo.
- Los comandos puestos en cola antes de la aprobación del pairing se descartan, no se aplazan.

## Límites de confianza de eventos del nodo (2026.3.31+)

<Warning>
**Cambio incompatible:** las ejecuciones originadas en nodos ahora permanecen en una superficie de confianza reducida.
</Warning>

Los resúmenes originados en nodos y los eventos de sesión relacionados se restringen a la superficie de confianza prevista. Es posible que los flujos impulsados por notificaciones o activados por nodos que antes dependían de un acceso más amplio a herramientas del host o la sesión deban ajustarse. Este refuerzo garantiza que los eventos del nodo no puedan escalar a acceso a herramientas a nivel de host más allá de lo que permite el límite de confianza del nodo.

## Autoaprobación (app de macOS)

La app de macOS puede intentar opcionalmente una **aprobación silenciosa** cuando:

- la solicitud está marcada como `silent`, y
- la app puede verificar una conexión SSH al host del gateway usando el mismo usuario.

Si la aprobación silenciosa falla, vuelve al prompt normal de “Approve/Reject”.

## Almacenamiento (local, privado)

El estado de pairing se almacena bajo el directorio de estado del Gateway (predeterminado `~/.openclaw`):

- `~/.openclaw/nodes/paired.json`
- `~/.openclaw/nodes/pending.json`

Si sobrescribes `OPENCLAW_STATE_DIR`, la carpeta `nodes/` se mueve con él.

Notas de seguridad:

- Los tokens son secretos; trata `paired.json` como un archivo sensible.
- Rotar un token requiere una nueva aprobación (o eliminar la entrada del nodo).

## Comportamiento del transporte

- El transporte es **sin estado**; no almacena pertenencia.
- Si el Gateway está desconectado o el pairing está deshabilitado, los nodos no pueden emparejarse.
- Si el Gateway está en modo remoto, el pairing sigue realizándose contra el almacén del Gateway remoto.
