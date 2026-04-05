---
read_when:
    - Depuras problemas de descubrimiento Bonjour en macOS/iOS
    - Cambias tipos de servicio mDNS, registros TXT o la experiencia de descubrimiento
summary: Descubrimiento Bonjour/mDNS + depuración (beacons del Gateway, clientes y modos de fallo habituales)
title: Descubrimiento Bonjour
x-i18n:
    generated_at: "2026-04-05T12:41:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7f5a7f3211c74d4d10fdc570fc102b3c949c0ded9409c54995ab8820e5787f02
    source_path: gateway/bonjour.md
    workflow: 15
---

# Descubrimiento Bonjour / mDNS

OpenClaw usa Bonjour (mDNS / DNS‑SD) para descubrir un Gateway activo (endpoint WebSocket).
La exploración multicast de `local.` es una **comodidad solo para LAN**. Para el descubrimiento entre redes, el
mismo beacon también puede publicarse mediante un dominio DNS-SD de área amplia configurado. El descubrimiento
sigue siendo de mejor esfuerzo y **no** sustituye la conectividad basada en SSH o Tailnet.

## Bonjour de área amplia (DNS-SD unicast) sobre Tailscale

Si el nodo y el gateway están en redes distintas, el mDNS multicast no cruzará el
límite. Puedes mantener la misma experiencia de descubrimiento cambiando a **DNS‑SD unicast**
("Bonjour de área amplia") sobre Tailscale.

Pasos de alto nivel:

1. Ejecuta un servidor DNS en el host del gateway (accesible a través de la Tailnet).
2. Publica registros DNS‑SD para `_openclaw-gw._tcp` bajo una zona dedicada
   (ejemplo: `openclaw.internal.`).
3. Configura el **split DNS** de Tailscale para que tu dominio elegido se resuelva mediante ese
   servidor DNS para los clientes (incluido iOS).

OpenClaw admite cualquier dominio de descubrimiento; `openclaw.internal.` es solo un ejemplo.
Los nodos iOS/Android exploran tanto `local.` como el dominio de área amplia configurado.

### Configuración del Gateway (recomendada)

```json5
{
  gateway: { bind: "tailnet" }, // tailnet-only (recommended)
  discovery: { wideArea: { enabled: true } }, // enables wide-area DNS-SD publishing
}
```

### Configuración única del servidor DNS (host del gateway)

```bash
openclaw dns setup --apply
```

Esto instala CoreDNS y lo configura para:

- escuchar en el puerto 53 solo en las interfaces Tailscale del gateway
- servir tu dominio elegido (ejemplo: `openclaw.internal.`) desde `~/.openclaw/dns/<domain>.db`

Valídalo desde una máquina conectada a la tailnet:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### Configuración DNS de Tailscale

En la consola de administración de Tailscale:

- Añade un nameserver que apunte a la IP tailnet del gateway (UDP/TCP 53).
- Añade split DNS para que tu dominio de descubrimiento use ese nameserver.

Una vez que los clientes acepten el DNS de tailnet, los nodos iOS y el descubrimiento por CLI podrán explorar
`_openclaw-gw._tcp` en tu dominio de descubrimiento sin multicast.

### Seguridad del listener del Gateway (recomendada)

El puerto WS del Gateway (predeterminado `18789`) se vincula a loopback de forma predeterminada. Para acceso por LAN/tailnet,
vincúlalo explícitamente y mantén la autenticación habilitada.

Para configuraciones solo de tailnet:

- Configura `gateway.bind: "tailnet"` en `~/.openclaw/openclaw.json`.
- Reinicia el Gateway (o reinicia la app de barra de menús de macOS).

## Qué se anuncia

Solo el Gateway anuncia `_openclaw-gw._tcp`.

## Tipos de servicio

- `_openclaw-gw._tcp` — beacon de transporte del gateway (usado por nodos macOS/iOS/Android).

## Claves TXT (pistas no secretas)

El Gateway anuncia pequeñas pistas no secretas para hacer cómodos los flujos de UI:

- `role=gateway`
- `displayName=<friendly name>`
- `lanHost=<hostname>.local`
- `gatewayPort=<port>` (Gateway WS + HTTP)
- `gatewayTls=1` (solo cuando TLS está habilitado)
- `gatewayTlsSha256=<sha256>` (solo cuando TLS está habilitado y la huella digital está disponible)
- `canvasPort=<port>` (solo cuando el host de canvas está habilitado; actualmente es el mismo que `gatewayPort`)
- `transport=gateway`
- `tailnetDns=<magicdns>` (pista opcional cuando Tailnet está disponible)
- `sshPort=<port>` (solo en modo mDNS completo; DNS-SD de área amplia puede omitirlo)
- `cliPath=<path>` (solo en modo mDNS completo; DNS-SD de área amplia sigue escribiéndolo como pista para instalación remota)

Notas de seguridad:

- Los registros TXT de Bonjour/mDNS son **no autenticados**. Los clientes no deben tratar TXT como enrutamiento autoritativo.
- Los clientes deben enrutar usando el endpoint de servicio resuelto (SRV + A/AAAA). Trata `lanHost`, `tailnetDns`, `gatewayPort` y `gatewayTlsSha256` solo como pistas.
- El direccionamiento automático por SSH debe usar igualmente el host del servicio resuelto, no pistas basadas solo en TXT.
- La fijación de TLS nunca debe permitir que un `gatewayTlsSha256` anunciado anule una fijación almacenada previamente.
- Los nodos iOS/Android deben tratar las conexiones directas basadas en descubrimiento como **solo TLS** y requerir confirmación explícita del usuario antes de confiar en una huella digital por primera vez.

## Depuración en macOS

Herramientas integradas útiles:

- Explorar instancias:

  ```bash
  dns-sd -B _openclaw-gw._tcp local.
  ```

- Resolver una instancia (sustituye `<instance>`):

  ```bash
  dns-sd -L "<instance>" _openclaw-gw._tcp local.
  ```

Si la exploración funciona pero la resolución falla, normalmente te estás encontrando con una política de LAN o
un problema del resolvedor mDNS.

## Depuración en logs del Gateway

El Gateway escribe un archivo de log rotativo (impreso al iniciar como
`gateway log file: ...`). Busca líneas `bonjour:`, en especial:

- `bonjour: advertise failed ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`
- `bonjour: watchdog detected non-announced service ...`

## Depuración en nodos iOS

El nodo iOS usa `NWBrowser` para descubrir `_openclaw-gw._tcp`.

Para capturar logs:

- Settings → Gateway → Advanced → **Discovery Debug Logs**
- Settings → Gateway → Advanced → **Discovery Logs** → reproduce → **Copy**

El log incluye transiciones de estado del navegador y cambios en el conjunto de resultados.

## Modos de fallo habituales

- **Bonjour no cruza redes**: usa Tailnet o SSH.
- **Multicast bloqueado**: algunas redes Wi‑Fi deshabilitan mDNS.
- **Reposo / cambios de interfaz**: macOS puede eliminar temporalmente resultados mDNS; reintenta.
- **La exploración funciona pero la resolución falla**: mantén simples los nombres de máquina (evita emojis o
  puntuación), luego reinicia el Gateway. El nombre de instancia del servicio se deriva del
  nombre del host, así que los nombres demasiado complejos pueden confundir a algunos resolvedores.

## Nombres de instancia escapados (`\032`)

Bonjour/DNS‑SD suele escapar bytes en nombres de instancia de servicio como secuencias decimales `\DDD`
(por ejemplo, los espacios se convierten en `\032`).

- Esto es normal a nivel de protocolo.
- Las interfaces deben decodificar para mostrar (iOS usa `BonjourEscapes.decode`).

## Deshabilitación / configuración

- `OPENCLAW_DISABLE_BONJOUR=1` deshabilita la publicación (heredado: `OPENCLAW_DISABLE_BONJOUR`).
- `gateway.bind` en `~/.openclaw/openclaw.json` controla el modo de vínculo del Gateway.
- `OPENCLAW_SSH_PORT` anula el puerto SSH cuando se anuncia `sshPort` (heredado: `OPENCLAW_SSH_PORT`).
- `OPENCLAW_TAILNET_DNS` publica una pista MagicDNS en TXT (heredado: `OPENCLAW_TAILNET_DNS`).
- `OPENCLAW_CLI_PATH` anula la ruta de CLI anunciada (heredado: `OPENCLAW_CLI_PATH`).

## Documentación relacionada

- Política de descubrimiento y selección de transporte: [Discovery](/gateway/discovery)
- Emparejamiento y aprobaciones de nodos: [Emparejamiento del Gateway](/gateway/pairing)
