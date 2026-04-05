---
read_when:
    - Depurar la pestaña Instances
    - Investigar filas de instancias duplicadas o obsoletas
    - Cambiar la conexión WS del gateway o las balizas de eventos del sistema
summary: Cómo se producen, combinan y muestran las entradas de presencia de OpenClaw
title: Presencia
x-i18n:
    generated_at: "2026-04-05T12:40:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: a004a1f87be08699c1b2cba97cad8678ce5e27baa425f59eaa18006fdcff26e7
    source_path: concepts/presence.md
    workflow: 15
---

# Presencia

La “presencia” de OpenClaw es una vista ligera y de mejor esfuerzo de:

- el propio **Gateway**, y
- los **clientes conectados al Gateway** (app de Mac, WebChat, CLI, etc.)

La presencia se usa principalmente para renderizar la pestaña **Instances** de la app de macOS y para
proporcionar visibilidad rápida al operador.

## Campos de presencia (lo que aparece)

Las entradas de presencia son objetos estructurados con campos como:

- `instanceId` (opcional, pero muy recomendable): identidad estable del cliente (normalmente `connect.client.instanceId`)
- `host`: nombre de host legible para humanos
- `ip`: dirección IP de mejor esfuerzo
- `version`: cadena de versión del cliente
- `deviceFamily` / `modelIdentifier`: pistas de hardware
- `mode`: `ui`, `webchat`, `cli`, `backend`, `probe`, `test`, `node`, ...
- `lastInputSeconds`: “segundos desde la última entrada del usuario” (si se conoce)
- `reason`: `self`, `connect`, `node-connected`, `periodic`, ...
- `ts`: marca de tiempo de la última actualización (ms desde epoch)

## Productores (de dónde viene la presencia)

Las entradas de presencia son producidas por múltiples fuentes y se **combinan**.

### 1) Entrada propia del Gateway

El Gateway siempre inicializa una entrada “propia” al arrancar para que las interfaces muestren el host del gateway
incluso antes de que se conecte cualquier cliente.

### 2) Conexión WebSocket

Cada cliente WS comienza con una solicitud `connect`. Tras un handshake exitoso, el
Gateway realiza un upsert de una entrada de presencia para esa conexión.

#### Por qué los comandos puntuales de la CLI no aparecen

La CLI a menudo se conecta para comandos breves y puntuales. Para evitar saturar la
lista de Instances, `client.mode === "cli"` **no** se convierte en una entrada de presencia.

### 3) Balizas `system-event`

Los clientes pueden enviar balizas periódicas más ricas mediante el método `system-event`. La app de Mac
usa esto para informar del nombre del host, la IP y `lastInputSeconds`.

### 4) Conexiones de nodos (role: node)

Cuando un nodo se conecta por el WebSocket del Gateway con `role: node`, el Gateway
realiza un upsert de una entrada de presencia para ese nodo (el mismo flujo que otros clientes WS).

## Reglas de combinación y desduplicación (por qué importa `instanceId`)

Las entradas de presencia se almacenan en un único mapa en memoria:

- Las entradas se indexan mediante una **clave de presencia**.
- La mejor clave es un `instanceId` estable (de `connect.client.instanceId`) que sobrevive a los reinicios.
- Las claves no distinguen entre mayúsculas y minúsculas.

Si un cliente se vuelve a conectar sin un `instanceId` estable, puede aparecer como una
fila **duplicada**.

## TTL y tamaño limitado

La presencia es intencionalmente efímera:

- **TTL:** las entradas de más de 5 minutos se eliminan
- **Máximo de entradas:** 200 (las más antiguas se descartan primero)

Esto mantiene la lista actualizada y evita un crecimiento de memoria sin límites.

## Advertencia para acceso remoto/túneles (IPs de loopback)

Cuando un cliente se conecta mediante un túnel SSH / reenvío local de puertos, el Gateway puede
ver la dirección remota como `127.0.0.1`. Para evitar sobrescribir una IP informada correctamente por el cliente,
las direcciones remotas de loopback se ignoran.

## Consumidores

### Pestaña Instances de macOS

La app de macOS renderiza la salida de `system-presence` y aplica un pequeño indicador
de estado (Active/Idle/Stale) según la antigüedad de la última actualización.

## Consejos de depuración

- Para ver la lista sin procesar, llama a `system-presence` contra el Gateway.
- Si ves duplicados:
  - confirma que los clientes envían un `client.instanceId` estable en el handshake
  - confirma que las balizas periódicas usan el mismo `instanceId`
  - comprueba si a la entrada derivada de la conexión le falta `instanceId` (en ese caso, los duplicados son esperables)
