---
read_when:
    - Implementar o cambiar el descubrimiento/publicidad de Bonjour
    - Ajustar modos de conexión remota (directa frente a SSH)
    - Diseñar descubrimiento de nodos + emparejamiento para nodos remotos
summary: Descubrimiento de nodos y transportes (Bonjour, Tailscale, SSH) para encontrar el gateway
title: Descubrimiento y transportes
x-i18n:
    generated_at: "2026-04-05T12:41:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: e76cca9279ca77b55e30d6e746f6325e5644134ef06b9c58f2cf3d793d092685
    source_path: gateway/discovery.md
    workflow: 15
---

# Descubrimiento y transportes

OpenClaw tiene dos problemas distintos que en la superficie parecen similares:

1. **Control remoto del operador**: la app de barra de menús de macOS controlando un gateway que se ejecuta en otro lugar.
2. **Emparejamiento de nodos**: iOS/Android (y futuros nodos) encontrando un gateway y emparejándose de forma segura.

El objetivo del diseño es mantener todo el descubrimiento/publicidad de red en el **Node Gateway** (`openclaw gateway`) y mantener a los clientes (app de Mac, iOS) como consumidores.

## Términos

- **Gateway**: un único proceso gateway de larga duración que posee el estado (sesiones, emparejamiento, registro de nodos) y ejecuta canales. La mayoría de las configuraciones usan uno por host; son posibles configuraciones aisladas con varios gateway.
- **Gateway WS (plano de control)**: el endpoint WebSocket en `127.0.0.1:18789` de forma predeterminada; puede enlazarse a LAN/tailnet mediante `gateway.bind`.
- **Transporte WS directo**: un endpoint Gateway WS orientado a LAN/tailnet (sin SSH).
- **Transporte SSH (respaldo)**: control remoto reenviando `127.0.0.1:18789` sobre SSH.
- **Puente TCP heredado (eliminado)**: transporte antiguo de nodos (consulta
  [Protocolo de puente](/gateway/bridge-protocol)); ya no se anuncia para
  descubrimiento y ya no forma parte de las compilaciones actuales.

Detalles del protocolo:

- [Protocolo Gateway](/gateway/protocol)
- [Protocolo de puente (heredado)](/gateway/bridge-protocol)

## Por qué mantenemos tanto "directo" como SSH

- **WS directo** es la mejor experiencia de usuario en la misma red y dentro de una tailnet:
  - descubrimiento automático en LAN mediante Bonjour
  - tokens de emparejamiento + ACL propiedad del gateway
  - no requiere acceso de shell; la superficie del protocolo puede mantenerse reducida y auditable
- **SSH** sigue siendo el respaldo universal:
  - funciona en cualquier lugar donde tengas acceso SSH (incluso entre redes no relacionadas)
  - sobrevive a problemas de multidifusión/mDNS
  - no requiere nuevos puertos de entrada aparte de SSH

## Entradas de descubrimiento (cómo saben los clientes dónde está el gateway)

### 1) Descubrimiento Bonjour / DNS-SD

El Bonjour por multidifusión es de mejor esfuerzo y no cruza redes. OpenClaw también puede explorar la
misma baliza del gateway mediante un dominio DNS-SD de área amplia configurado, por lo que el descubrimiento puede cubrir:

- `local.` en la misma LAN
- un dominio DNS-SD unicast configurado para descubrimiento entre redes

Dirección objetivo:

- El **gateway** anuncia su endpoint WS mediante Bonjour.
- Los clientes exploran y muestran una lista de “elegir un gateway”, y luego almacenan el endpoint elegido.

Detalles de resolución de problemas y balizas: [Bonjour](/gateway/bonjour).

#### Detalles de la baliza de servicio

- Tipos de servicio:
  - `_openclaw-gw._tcp` (baliza de transporte del gateway)
- Claves TXT (no secretas):
  - `role=gateway`
  - `transport=gateway`
  - `displayName=<friendly name>` (nombre para mostrar configurado por el operador)
  - `lanHost=<hostname>.local`
  - `gatewayPort=18789` (Gateway WS + HTTP)
  - `gatewayTls=1` (solo cuando TLS está habilitado)
  - `gatewayTlsSha256=<sha256>` (solo cuando TLS está habilitado y la huella digital está disponible)
  - `canvasPort=<port>` (puerto del host canvas; actualmente es el mismo que `gatewayPort` cuando el host canvas está habilitado)
  - `tailnetDns=<magicdns>` (pista opcional; se detecta automáticamente cuando Tailscale está disponible)
  - `sshPort=<port>` (solo en modo mDNS completo; DNS-SD de área amplia puede omitirlo, en cuyo caso se mantienen los valores predeterminados de SSH en `22`)
  - `cliPath=<path>` (solo en modo mDNS completo; DNS-SD de área amplia sigue escribiéndolo como pista de instalación remota)

Notas de seguridad:

- Los registros TXT de Bonjour/mDNS **no están autenticados**. Los clientes deben tratar los valores TXT solo como pistas de experiencia de usuario.
- El enrutamiento (host/puerto) debe preferir el **endpoint de servicio resuelto** (SRV + A/AAAA) sobre `lanHost`, `tailnetDns` o `gatewayPort` proporcionados por TXT.
- La fijación de TLS nunca debe permitir que un `gatewayTlsSha256` anunciado anule una fijación almacenada previamente.
- Los nodos iOS/Android deben requerir una confirmación explícita de “confiar en esta huella digital” antes de almacenar una primera fijación (verificación fuera de banda) siempre que la ruta elegida esté protegida/basada en TLS.

Desactivar/anular:

- `OPENCLAW_DISABLE_BONJOUR=1` desactiva la publicidad.
- `gateway.bind` en `~/.openclaw/openclaw.json` controla el modo de enlace del Gateway.
- `OPENCLAW_SSH_PORT` anula el puerto SSH anunciado cuando se emite `sshPort`.
- `OPENCLAW_TAILNET_DNS` publica una pista `tailnetDns` (MagicDNS).
- `OPENCLAW_CLI_PATH` anula la ruta de CLI anunciada.

### 2) Tailnet (entre redes)

Para configuraciones de estilo Londres/Viena, Bonjour no ayudará. El objetivo “directo” recomendado es:

- nombre de Tailscale MagicDNS (preferido) o una IP tailnet estable.

Si el gateway puede detectar que se está ejecutando bajo Tailscale, publica `tailnetDns` como pista opcional para los clientes (incluidas las balizas de área amplia).

La app de macOS ahora prefiere nombres MagicDNS frente a direcciones IP Tailscale sin procesar para el descubrimiento de gateway. Esto mejora la fiabilidad cuando cambian las IP de tailnet (por ejemplo, tras reinicios de nodos o reasignación CGNAT), porque los nombres MagicDNS se resuelven automáticamente a la IP actual.

Para el emparejamiento de nodos móviles, las pistas de descubrimiento no relajan la seguridad de transporte en rutas tailnet/públicas:

- iOS/Android siguen requiriendo una ruta segura de primera conexión tailnet/pública (`wss://` o Tailscale Serve/Funnel).
- Una IP tailnet sin procesar descubierta es una pista de enrutamiento, no un permiso para usar `ws://` remoto en texto plano.
- La conexión directa privada de LAN con `ws://` sigue siendo compatible.
- Si quieres la ruta Tailscale más sencilla para nodos móviles, usa Tailscale Serve para que tanto el descubrimiento como el código de configuración se resuelvan al mismo endpoint MagicDNS seguro.

### 3) Destino manual / SSH

Cuando no hay una ruta directa (o la directa está desactivada), los clientes siempre pueden conectarse mediante SSH reenviando el puerto loopback del gateway.

Consulta [Acceso remoto](/gateway/remote).

## Selección de transporte (política del cliente)

Comportamiento recomendado del cliente:

1. Si hay un endpoint directo emparejado configurado y accesible, úsalo.
2. En caso contrario, si el descubrimiento encuentra un gateway en `local.` o en el dominio de área amplia configurado, ofrece una opción de un toque “Usar este gateway” y guárdalo como endpoint directo.
3. En caso contrario, si hay un DNS/IP de tailnet configurado, intenta directo.
   Para nodos móviles en rutas tailnet/públicas, directo significa un endpoint seguro, no `ws://` remoto en texto plano.
4. En caso contrario, recurre a SSH.

## Emparejamiento + autenticación (transporte directo)

El gateway es la fuente de verdad para la admisión de nodos/clientes.

- Las solicitudes de emparejamiento se crean/aprueban/rechazan en el gateway (consulta [Emparejamiento del Gateway](/gateway/pairing)).
- El gateway aplica:
  - autenticación (token / par de claves)
  - ámbitos/ACL (el gateway no es un proxy sin procesar para todos los métodos)
  - límites de tasa

## Responsabilidades por componente

- **Gateway**: anuncia balizas de descubrimiento, es propietario de las decisiones de emparejamiento y aloja el endpoint WS.
- **App de macOS**: te ayuda a elegir un gateway, muestra prompts de emparejamiento y usa SSH solo como respaldo.
- **Nodos iOS/Android**: exploran Bonjour por comodidad y se conectan al Gateway WS emparejado.
