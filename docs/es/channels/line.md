---
read_when:
    - Quieres conectar OpenClaw a LINE
    - Necesitas la configuración del webhook y las credenciales de LINE
    - Quieres opciones de mensajes específicas de LINE
summary: Configuración, ajuste y uso del plugin de LINE Messaging API
title: LINE
x-i18n:
    generated_at: "2026-04-05T12:35:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: b4782b2aa3e8654505d7f1fd6fc112adf125b5010fc84d655d033688ded37414
    source_path: channels/line.md
    workflow: 15
---

# LINE

LINE se conecta a OpenClaw mediante la LINE Messaging API. El plugin se ejecuta como receptor de webhooks en el gateway y usa tu token de acceso del canal y el secreto del canal para la autenticación.

Estado: plugin integrado. Se admiten mensajes directos, chats de grupo, medios, ubicaciones, mensajes Flex, mensajes de plantilla y respuestas rápidas. No se admiten reacciones ni hilos.

## Plugin integrado

LINE se distribuye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación independiente.

Si usas una compilación antigua o una instalación personalizada que excluye LINE, instálalo manualmente:

```bash
openclaw plugins install @openclaw/line
```

Checkout local (cuando se ejecuta desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## Configuración inicial

1. Crea una cuenta de LINE Developers y abre la consola:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. Crea (o elige) un proveedor y añade un canal de **Messaging API**.
3. Copia el **Channel access token** y el **Channel secret** desde la configuración del canal.
4. Activa **Use webhook** en la configuración de Messaging API.
5. Establece la URL del webhook en tu endpoint del gateway (se requiere HTTPS):

```
https://gateway-host/line/webhook
```

El gateway responde a la verificación de webhooks de LINE (GET) y a los eventos entrantes (POST).
Si necesitas una ruta personalizada, establece `channels.line.webhookPath` o
`channels.line.accounts.<id>.webhookPath` y actualiza la URL en consecuencia.

Nota de seguridad:

- La verificación de firma de LINE depende del cuerpo (HMAC sobre el cuerpo sin procesar), por lo que OpenClaw aplica límites estrictos de tamaño del cuerpo y tiempo de espera antes de la verificación.
- OpenClaw procesa los eventos de webhook a partir de los bytes sin procesar de la solicitud verificada. Los valores `req.body` transformados por middleware ascendente se ignoran para mantener la integridad de la firma.

## Configurar

Configuración mínima:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

Variables de entorno (solo cuenta predeterminada):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

Archivos de token/secreto:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` y `secretFile` deben apuntar a archivos normales. Los enlaces simbólicos se rechazan.

Múltiples cuentas:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## Control de acceso

Los mensajes directos usan pairing de forma predeterminada. Los remitentes desconocidos reciben un código de pairing y sus mensajes se ignoran hasta que se aprueban.

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

Listas de permitidos y políticas:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled`
- `channels.line.allowFrom`: ID de usuario de LINE permitidos para MD
- `channels.line.groupPolicy`: `allowlist | open | disabled`
- `channels.line.groupAllowFrom`: ID de usuario de LINE permitidos para grupos
- Anulaciones por grupo: `channels.line.groups.<groupId>.allowFrom`
- Nota de ejecución: si falta por completo `channels.line`, la ejecución usa como alternativa `groupPolicy="allowlist"` para las comprobaciones de grupos (incluso si `channels.defaults.groupPolicy` está establecido).

Los ID de LINE distinguen entre mayúsculas y minúsculas. Los ID válidos tienen este aspecto:

- Usuario: `U` + 32 caracteres hexadecimales
- Grupo: `C` + 32 caracteres hexadecimales
- Sala: `R` + 32 caracteres hexadecimales

## Comportamiento de los mensajes

- El texto se divide en fragmentos de 5000 caracteres.
- Se elimina el formato Markdown; los bloques de código y las tablas se convierten en tarjetas Flex cuando es posible.
- Las respuestas en streaming se almacenan en búfer; LINE recibe fragmentos completos con una animación de carga mientras el agente trabaja.
- Las descargas de medios están limitadas por `channels.line.mediaMaxMb` (10 de forma predeterminada).

## Datos del canal (mensajes enriquecidos)

Usa `channelData.line` para enviar respuestas rápidas, ubicaciones, tarjetas Flex o mensajes de plantilla.

```json5
{
  text: "Here you go",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Office",
        address: "123 Main St",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Status card",
        contents: {
          /* Flex payload */
        },
      },
      templateMessage: {
        type: "confirm",
        text: "Proceed?",
        confirmLabel: "Yes",
        confirmData: "yes",
        cancelLabel: "No",
        cancelData: "no",
      },
    },
  },
}
```

El plugin de LINE también incluye un comando `/card` para preajustes de mensajes Flex:

```
/card info "Welcome" "Thanks for joining!"
```

## Soporte de ACP

LINE admite enlaces de conversación ACP (Agent Communication Protocol):

- `/acp spawn <agent> --bind here` enlaza el chat actual de LINE a una sesión ACP sin crear un hilo secundario.
- Los enlaces ACP configurados y las sesiones ACP activas enlazadas a conversaciones funcionan en LINE igual que en otros canales de conversación.

Consulta [Agentes ACP](/tools/acp-agents) para más detalles.

## Medios salientes

El plugin de LINE admite el envío de imágenes, vídeos y archivos de audio mediante la herramienta de mensajes del agente. Los medios se envían a través de la ruta de entrega específica de LINE con el manejo adecuado de vista previa y seguimiento:

- **Imágenes**: se envían como mensajes de imagen de LINE con generación automática de vista previa.
- **Vídeos**: se envían con manejo explícito de vista previa y tipo de contenido.
- **Audio**: se envía como mensajes de audio de LINE.

Los envíos de medios genéricos recurren a la ruta existente solo para imágenes cuando no hay una ruta específica de LINE disponible.

## Solución de problemas

- **La verificación del webhook falla:** asegúrate de que la URL del webhook use HTTPS y de que `channelSecret` coincida con el de la consola de LINE.
- **No hay eventos entrantes:** confirma que la ruta del webhook coincida con `channels.line.webhookPath`
  y que el gateway sea accesible desde LINE.
- **Errores de descarga de medios:** aumenta `channels.line.mediaMaxMb` si los medios superan el límite predeterminado.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de MD y flujo de pairing
- [Grupos](/channels/groups) — comportamiento de chats de grupo y control por menciones
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y endurecimiento
