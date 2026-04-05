---
read_when:
    - Quieres que OpenClaw reciba mensajes directos mediante Nostr
    - Estás configurando mensajería descentralizada
summary: Canal de mensajes directos de Nostr mediante mensajes cifrados NIP-04
title: Nostr
x-i18n:
    generated_at: "2026-04-05T12:35:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: f82829ee66fbeb3367007af343797140049ea49f2e842a695fa56acea0c80728
    source_path: channels/nostr.md
    workflow: 15
---

# Nostr

**Estado:** Plugin empaquetado opcional (desactivado de forma predeterminada hasta que se configure).

Nostr es un protocolo descentralizado para redes sociales. Este canal permite que OpenClaw reciba y responda a mensajes directos (DM) cifrados mediante NIP-04.

## Plugin empaquetado

Las versiones actuales de OpenClaw incluyen Nostr como plugin empaquetado, por lo que las compilaciones empaquetadas normales no necesitan una instalación independiente.

### Instalaciones antiguas/personalizadas

- El onboarding (`openclaw onboard`) y `openclaw channels add` siguen mostrando Nostr desde el catálogo compartido de canales.
- Si tu compilación excluye el plugin empaquetado de Nostr, instálalo manualmente.

```bash
openclaw plugins install @openclaw/nostr
```

Usa una copia local (flujos de desarrollo):

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

Reinicia el Gateway después de instalar o habilitar plugins.

### Configuración no interactiva

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

Usa `--use-env` para mantener `NOSTR_PRIVATE_KEY` en el entorno en lugar de almacenar la clave en la configuración.

## Configuración rápida

1. Genera un par de claves de Nostr (si es necesario):

```bash
# Using nak
nak key generate
```

2. Añádelo a la configuración:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. Exporta la clave:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. Reinicia el Gateway.

## Referencia de configuración

| Clave        | Tipo     | Predeterminado                              | Descripción                          |
| ------------ | -------- | ------------------------------------------- | ------------------------------------ |
| `privateKey` | string   | obligatorio                                 | Clave privada en formato `nsec` o hex |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | URL de relays (WebSocket)            |
| `dmPolicy`   | string   | `pairing`                                   | Política de acceso a mensajes directos |
| `allowFrom`  | string[] | `[]`                                        | Claves públicas de remitentes permitidos |
| `enabled`    | boolean  | `true`                                      | Habilita/deshabilita el canal        |
| `name`       | string   | -                                           | Nombre para mostrar                  |
| `profile`    | object   | -                                           | Metadatos de perfil NIP-01           |

## Metadatos del perfil

Los datos del perfil se publican como un evento NIP-01 `kind:0`. Puedes gestionarlos desde la interfaz de Control (Channels -> Nostr -> Profile) o configurarlos directamente en la configuración.

Ejemplo:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "Bot asistente personal por mensajes directos",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

Notas:

- Las URL del perfil deben usar `https://`.
- La importación desde relays fusiona campos y conserva las anulaciones locales.

## Control de acceso

### Políticas de mensajes directos

- **pairing** (predeterminada): los remitentes desconocidos reciben un código de emparejamiento.
- **allowlist**: solo las claves públicas de `allowFrom` pueden enviar mensajes directos.
- **open**: mensajes directos públicos entrantes (requiere `allowFrom: ["*"]`).
- **disabled**: ignora los mensajes directos entrantes.

Notas de aplicación:

- Las firmas de eventos entrantes se verifican antes de la política de remitente y del descifrado NIP-04, por lo que los eventos falsificados se rechazan de forma temprana.
- Las respuestas de emparejamiento se envían sin procesar el cuerpo original del mensaje directo.
- Los mensajes directos entrantes tienen limitación de tasa y las cargas útiles sobredimensionadas se descartan antes del descifrado.

### Ejemplo de lista de permitidos

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## Formatos de clave

Formatos aceptados:

- **Clave privada:** `nsec...` o hex de 64 caracteres
- **Claves públicas (`allowFrom`):** `npub...` o hex

## Relays

Predeterminados: `relay.damus.io` y `nos.lol`.

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

Consejos:

- Usa 2-3 relays para redundancia.
- Evita demasiados relays (latencia, duplicación).
- Los relays de pago pueden mejorar la fiabilidad.
- Los relays locales sirven para pruebas (`ws://localhost:7777`).

## Compatibilidad de protocolo

| NIP    | Estado    | Descripción                            |
| ------ | --------- | -------------------------------------- |
| NIP-01 | Compatible | Formato básico de eventos + metadatos de perfil |
| NIP-04 | Compatible | Mensajes directos cifrados (`kind:4`) |
| NIP-17 | Planificado | Mensajes directos envueltos como regalo |
| NIP-44 | Planificado | Cifrado versionado                    |

## Pruebas

### Relay local

```bash
# Start strfry
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### Prueba manual

1. Anota la clave pública del bot (npub) desde los registros.
2. Abre un cliente Nostr (Damus, Amethyst, etc.).
3. Envía un mensaje directo a la clave pública del bot.
4. Verifica la respuesta.

## Resolución de problemas

### No se reciben mensajes

- Verifica que la clave privada sea válida.
- Asegúrate de que las URL de los relays sean accesibles y usen `wss://` (o `ws://` para local).
- Confirma que `enabled` no sea `false`.
- Revisa los registros del Gateway para ver errores de conexión de relay.

### No se envían respuestas

- Comprueba que el relay acepte escrituras.
- Verifica la conectividad saliente.
- Observa posibles límites de tasa del relay.

### Respuestas duplicadas

- Es esperable cuando se usan varios relays.
- Los mensajes se deduplican por ID de evento; solo la primera entrega activa una respuesta.

## Seguridad

- Nunca confirmes claves privadas en el repositorio.
- Usa variables de entorno para las claves.
- Considera `allowlist` para bots de producción.
- Las firmas se verifican antes de la política de remitente, y la política de remitente se aplica antes del descifrado, por lo que los eventos falsificados se rechazan pronto y los remitentes desconocidos no pueden forzar trabajo criptográfico completo.

## Limitaciones (MVP)

- Solo mensajes directos (sin chats grupales).
- Sin archivos multimedia adjuntos.
- Solo NIP-04 (NIP-17 gift-wrap está planificado).

## Relacionado

- [Resumen de canales](/channels): todos los canales compatibles
- [Pairing](/channels/pairing): autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups): comportamiento de chats grupales y control por menciones
- [Enrutamiento de canales](/channels/channel-routing): enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security): modelo de acceso y endurecimiento
