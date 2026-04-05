---
read_when:
    - Trabajas en funciones o webhooks de Zalo
summary: Estado de compatibilidad, capacidades y configuración del bot de Zalo
title: Zalo
x-i18n:
    generated_at: "2026-04-05T12:37:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: ab94642ba28e79605b67586af8f71c18bc10e0af60343a7df508e6823b6f4119
    source_path: channels/zalo.md
    workflow: 15
---

# Zalo (Bot API)

Estado: experimental. Los mensajes directos son compatibles. La sección [Capacidades](#capacidades) a continuación refleja el comportamiento actual de los bots de Marketplace.

## Plugin incluido

Zalo se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación independiente.

Si usas una compilación antigua o una instalación personalizada que excluye Zalo, instálalo manualmente:

- Instalar mediante CLI: `openclaw plugins install @openclaw/zalo`
- O desde un checkout del código fuente: `openclaw plugins install ./path/to/local/zalo-plugin`
- Detalles: [Plugins](/tools/plugin)

## Configuración rápida (principiante)

1. Asegúrate de que el plugin de Zalo esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas o personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Configura el token:
   - Entorno: `ZALO_BOT_TOKEN=...`
   - O configuración: `channels.zalo.accounts.default.botToken: "..."`.
3. Reinicia el gateway (o termina la configuración).
4. El acceso por mensaje directo usa emparejamiento de forma predeterminada; aprueba el código de emparejamiento en el primer contacto.

Configuración mínima:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

## Qué es

Zalo es una aplicación de mensajería centrada en Vietnam; su Bot API permite que el Gateway ejecute un bot para conversaciones 1:1.
Es una buena opción para soporte o notificaciones cuando quieres un enrutamiento determinista de vuelta a Zalo.

Esta página refleja el comportamiento actual de OpenClaw para **bots de Zalo Bot Creator / Marketplace**.
**Los bots de Zalo Official Account (OA)** pertenecen a otra superficie de producto de Zalo y pueden comportarse de forma diferente.

- Un canal de Zalo Bot API controlado por el Gateway.
- Enrutamiento determinista: las respuestas vuelven a Zalo; el modelo nunca elige canales.
- Los mensajes directos comparten la sesión principal del agente.
- La sección [Capacidades](#capacidades) a continuación muestra la compatibilidad actual de los bots de Marketplace.

## Configuración (ruta rápida)

### 1) Crear un token de bot (Zalo Bot Platform)

1. Ve a [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com) e inicia sesión.
2. Crea un bot nuevo y configura sus ajustes.
3. Copia el token completo del bot (normalmente `numeric_id:secret`). Para bots de Marketplace, el token de ejecución utilizable puede aparecer en el mensaje de bienvenida del bot después de su creación.

### 2) Configurar el token (entorno o configuración)

Ejemplo:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

Si más adelante pasas a una superficie de bot de Zalo donde los grupos estén disponibles, puedes añadir configuración específica de grupos como `groupPolicy` y `groupAllowFrom` explícitamente. Para el comportamiento actual de los bots de Marketplace, consulta [Capacidades](#capacidades).

Opción de entorno: `ZALO_BOT_TOKEN=...` (funciona solo para la cuenta predeterminada).

Compatibilidad con varias cuentas: usa `channels.zalo.accounts` con tokens por cuenta y `name` opcional.

3. Reinicia el gateway. Zalo se inicia cuando se resuelve un token (entorno o configuración).
4. El acceso por mensaje directo usa emparejamiento de forma predeterminada. Aprueba el código cuando se contacte al bot por primera vez.

## Cómo funciona (comportamiento)

- Los mensajes entrantes se normalizan en el sobre compartido del canal con marcadores de posición de medios.
- Las respuestas siempre se enrutan de vuelta al mismo chat de Zalo.
- Long-polling de forma predeterminada; el modo webhook está disponible con `channels.zalo.webhookUrl`.

## Límites

- El texto saliente se divide en fragmentos de 2000 caracteres (límite de la API de Zalo).
- Las descargas y cargas de medios están limitadas por `channels.zalo.mediaMaxMb` (predeterminado: 5).
- La transmisión está bloqueada de forma predeterminada porque el límite de 2000 caracteres hace que sea menos útil.

## Control de acceso (mensajes directos)

### Acceso por mensaje directo

- Predeterminado: `channels.zalo.dmPolicy = "pairing"`. Los remitentes desconocidos reciben un código de emparejamiento; los mensajes se ignoran hasta que se aprueban (los códigos caducan tras 1 hora).
- Aprobar mediante:
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
- El emparejamiento es el intercambio de tokens predeterminado. Detalles: [Emparejamiento](/channels/pairing)
- `channels.zalo.allowFrom` acepta IDs numéricos de usuario (no hay búsqueda por nombre de usuario).

## Control de acceso (grupos)

Para **bots de Zalo Bot Creator / Marketplace**, la compatibilidad con grupos no estaba disponible en la práctica porque no se podía añadir el bot a un grupo en absoluto.

Eso significa que las claves de configuración relacionadas con grupos a continuación existen en el esquema, pero no eran utilizables para los bots de Marketplace:

- `channels.zalo.groupPolicy` controla el manejo de entrada de grupos: `open | allowlist | disabled`.
- `channels.zalo.groupAllowFrom` restringe qué IDs de remitente pueden activar el bot en grupos.
- Si `groupAllowFrom` no está configurado, Zalo recurre a `allowFrom` para las comprobaciones del remitente.
- Nota de tiempo de ejecución: si falta por completo `channels.zalo`, el tiempo de ejecución sigue recurriendo a `groupPolicy="allowlist"` por seguridad.

Los valores de política de grupo (cuando el acceso a grupos está disponible en la superficie de tu bot) son:

- `groupPolicy: "disabled"` — bloquea todos los mensajes de grupo.
- `groupPolicy: "open"` — permite a cualquier miembro del grupo (con restricción por mención).
- `groupPolicy: "allowlist"` — valor predeterminado de cierre por fallo; solo se aceptan remitentes permitidos.

Si usas una superficie de producto distinta de bots de Zalo y has verificado un comportamiento de grupo funcional, documéntalo por separado en lugar de asumir que coincide con el flujo de bots de Marketplace.

## Long-polling frente a webhook

- Predeterminado: long-polling (no se requiere URL pública).
- Modo webhook: configura `channels.zalo.webhookUrl` y `channels.zalo.webhookSecret`.
  - El secreto del webhook debe tener entre 8 y 256 caracteres.
  - La URL del webhook debe usar HTTPS.
  - Zalo envía eventos con la cabecera `X-Bot-Api-Secret-Token` para verificación.
  - El HTTP del gateway gestiona las solicitudes de webhook en `channels.zalo.webhookPath` (de forma predeterminada, la ruta de la URL del webhook).
  - Las solicitudes deben usar `Content-Type: application/json` (o tipos de medios `+json`).
  - Los eventos duplicados (`event_name + message_id`) se ignoran durante una breve ventana de repetición.
  - El tráfico en ráfaga se limita por tasa por ruta/fuente y puede devolver HTTP 429.

**Nota:** `getUpdates` (polling) y webhook son mutuamente excluyentes según la documentación de la API de Zalo.

## Tipos de mensajes compatibles

Para una vista rápida de la compatibilidad, consulta [Capacidades](#capacidades). Las notas a continuación añaden detalle donde el comportamiento necesita contexto adicional.

- **Mensajes de texto**: compatibilidad completa con fragmentación a 2000 caracteres.
- **URL simples en texto**: se comportan como entrada de texto normal.
- **Vistas previas de enlaces / tarjetas de enlaces enriquecidos**: consulta el estado de los bots de Marketplace en [Capacidades](#capacidades); no activaban una respuesta de forma fiable.
- **Mensajes con imágenes**: consulta el estado de los bots de Marketplace en [Capacidades](#capacidades); el manejo de imágenes entrantes no era fiable (indicador de escritura sin respuesta final).
- **Stickers**: consulta el estado de los bots de Marketplace en [Capacidades](#capacidades).
- **Notas de voz / archivos de audio / video / archivos adjuntos genéricos**: consulta el estado de los bots de Marketplace en [Capacidades](#capacidades).
- **Tipos no compatibles**: se registran en logs (por ejemplo, mensajes de usuarios protegidos).

## Capacidades

Esta tabla resume el comportamiento actual de los **bots de Zalo Bot Creator / Marketplace** en OpenClaw.

| Función                     | Estado                                  |
| --------------------------- | --------------------------------------- |
| Mensajes directos           | ✅ Compatible                            |
| Grupos                      | ❌ No disponible para bots de Marketplace |
| Medios (imágenes entrantes) | ⚠️ Limitado / verifica en tu entorno     |
| Medios (imágenes salientes) | ⚠️ No se volvió a probar para bots de Marketplace |
| URL simples en texto        | ✅ Compatible                            |
| Vistas previas de enlaces   | ⚠️ No fiable para bots de Marketplace    |
| Reacciones                  | ❌ No compatible                         |
| Stickers                    | ⚠️ Sin respuesta del agente para bots de Marketplace |
| Notas de voz / audio / video | ⚠️ Sin respuesta del agente para bots de Marketplace |
| Archivos adjuntos           | ⚠️ Sin respuesta del agente para bots de Marketplace |
| Hilos                       | ❌ No compatible                         |
| Encuestas                   | ❌ No compatible                         |
| Comandos nativos            | ❌ No compatible                         |
| Transmisión                 | ⚠️ Bloqueada (límite de 2000 caracteres) |

## Destinos de entrega (CLI/cron)

- Usa un id de chat como destino.
- Ejemplo: `openclaw message send --channel zalo --target 123456789 --message "hi"`.

## Solución de problemas

**El bot no responde:**

- Comprueba que el token sea válido: `openclaw channels status --probe`
- Verifica que el remitente esté aprobado (emparejamiento o allowFrom)
- Consulta los logs del gateway: `openclaw logs --follow`

**El webhook no recibe eventos:**

- Asegúrate de que la URL del webhook use HTTPS
- Verifica que el token secreto tenga entre 8 y 256 caracteres
- Confirma que el endpoint HTTP del gateway sea accesible en la ruta configurada
- Comprueba que el polling de `getUpdates` no esté en ejecución (son mutuamente excluyentes)

## Referencia de configuración (Zalo)

Configuración completa: [Configuration](/gateway/configuration)

Las claves planas de nivel superior (`channels.zalo.botToken`, `channels.zalo.dmPolicy` y similares) son una abreviatura heredada para una sola cuenta. Para configuraciones nuevas, prefiere `channels.zalo.accounts.<id>.*`. Ambas formas siguen documentándose aquí porque existen en el esquema.

Opciones del proveedor:

- `channels.zalo.enabled`: habilitar o deshabilitar el inicio del canal.
- `channels.zalo.botToken`: token del bot de Zalo Bot Platform.
- `channels.zalo.tokenFile`: leer el token desde una ruta de archivo normal. Se rechazan los enlaces simbólicos.
- `channels.zalo.dmPolicy`: `pairing | allowlist | open | disabled` (predeterminado: pairing).
- `channels.zalo.allowFrom`: lista de permitidos para mensajes directos (IDs de usuario). `open` requiere `"*"`. El asistente pedirá IDs numéricos.
- `channels.zalo.groupPolicy`: `open | allowlist | disabled` (predeterminado: allowlist). Está presente en la configuración; consulta [Capacidades](#capacidades) y [Control de acceso (grupos)](#control-de-acceso-grupos) para el comportamiento actual de los bots de Marketplace.
- `channels.zalo.groupAllowFrom`: lista de permitidos de remitentes de grupo (IDs de usuario). Recurre a `allowFrom` cuando no está configurado.
- `channels.zalo.mediaMaxMb`: límite de medios entrantes/salientes (MB, predeterminado: 5).
- `channels.zalo.webhookUrl`: habilitar modo webhook (requiere HTTPS).
- `channels.zalo.webhookSecret`: secreto del webhook (8-256 caracteres).
- `channels.zalo.webhookPath`: ruta del webhook en el servidor HTTP del gateway.
- `channels.zalo.proxy`: URL de proxy para solicitudes de API.

Opciones de varias cuentas:

- `channels.zalo.accounts.<id>.botToken`: token por cuenta.
- `channels.zalo.accounts.<id>.tokenFile`: archivo de token normal por cuenta. Se rechazan los enlaces simbólicos.
- `channels.zalo.accounts.<id>.name`: nombre para mostrar.
- `channels.zalo.accounts.<id>.enabled`: habilitar o deshabilitar cuenta.
- `channels.zalo.accounts.<id>.dmPolicy`: política de mensajes directos por cuenta.
- `channels.zalo.accounts.<id>.allowFrom`: lista de permitidos por cuenta.
- `channels.zalo.accounts.<id>.groupPolicy`: política de grupos por cuenta. Está presente en la configuración; consulta [Capacidades](#capacidades) y [Control de acceso (grupos)](#control-de-acceso-grupos) para el comportamiento actual de los bots de Marketplace.
- `channels.zalo.accounts.<id>.groupAllowFrom`: lista de permitidos de remitentes de grupo por cuenta.
- `channels.zalo.accounts.<id>.webhookUrl`: URL de webhook por cuenta.
- `channels.zalo.accounts.<id>.webhookSecret`: secreto de webhook por cuenta.
- `channels.zalo.accounts.<id>.webhookPath`: ruta de webhook por cuenta.
- `channels.zalo.accounts.<id>.proxy`: URL de proxy por cuenta.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y restricción por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
