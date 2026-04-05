---
read_when:
    - Trabajar en funcionalidades del canal de Microsoft Teams
summary: Estado del soporte del bot de Microsoft Teams, capacidades y configuración
title: Microsoft Teams
x-i18n:
    generated_at: "2026-04-05T12:37:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 99fc6e136893ec65dc85d3bc0c0d92134069a2f3b8cb4fcf66c14674399b3eaf
    source_path: channels/msteams.md
    workflow: 15
---

# Microsoft Teams

> "Abandonad toda esperanza quienes entráis aquí."

Actualizado: 2026-01-21

Estado: se admiten texto + archivos adjuntos en MD; el envío de archivos en canales/grupos requiere `sharePointSiteId` + permisos de Graph (consulta [Envío de archivos en chats grupales](#envío-de-archivos-en-chats-grupales)). Las encuestas se envían mediante Adaptive Cards. Las acciones de mensaje exponen `upload-file` explícito para envíos centrados primero en archivos.

## Plugin integrado

Microsoft Teams se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que no
se requiere una instalación aparte en la compilación empaquetada normal.

Si estás en una compilación anterior o en una instalación personalizada que excluye Teams integrado,
instálalo manualmente:

```bash
openclaw plugins install @openclaw/msteams
```

Copia local (al ejecutarse desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Detalles: [Plugins](/tools/plugin)

## Configuración rápida (principiante)

1. Asegúrate de que el plugin de Microsoft Teams esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea un **Azure Bot** (App ID + secreto de cliente + tenant ID).
3. Configura OpenClaw con esas credenciales.
4. Expón `/api/messages` (puerto 3978 de forma predeterminada) mediante una URL pública o túnel.
5. Instala el paquete de la aplicación de Teams e inicia el gateway.

Configuración mínima:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Nota: los chats grupales están bloqueados de forma predeterminada (`channels.msteams.groupPolicy: "allowlist"`). Para permitir respuestas en grupos, establece `channels.msteams.groupAllowFrom` (o usa `groupPolicy: "open"` para permitir a cualquier miembro, con control por mención).

## Objetivos

- Hablar con OpenClaw mediante MD de Teams, chats grupales o canales.
- Mantener el enrutamiento determinista: las respuestas siempre vuelven al canal por el que llegaron.
- Aplicar un comportamiento de canal seguro de forma predeterminada (menciones obligatorias salvo que se configure lo contrario).

## Escrituras de configuración

De forma predeterminada, Microsoft Teams puede escribir actualizaciones de configuración activadas por `/config set|unset` (requiere `commands.config: true`).

Desactívalo con:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Control de acceso (MD + grupos)

**Acceso por MD**

- Predeterminado: `channels.msteams.dmPolicy = "pairing"`. Los remitentes desconocidos se ignoran hasta que se aprueban.
- `channels.msteams.allowFrom` debe usar IDs de objeto AAD estables.
- Los UPN/nombres para mostrar son mutables; la coincidencia directa está deshabilitada de forma predeterminada y solo se habilita con `channels.msteams.dangerouslyAllowNameMatching: true`.
- El asistente puede resolver nombres a IDs mediante Microsoft Graph cuando las credenciales lo permiten.

**Acceso por grupo**

- Predeterminado: `channels.msteams.groupPolicy = "allowlist"` (bloqueado a menos que añadas `groupAllowFrom`). Usa `channels.defaults.groupPolicy` para anular el valor predeterminado cuando no esté establecido.
- `channels.msteams.groupAllowFrom` controla qué remitentes pueden activar en chats grupales/canales (recurre a `channels.msteams.allowFrom`).
- Establece `groupPolicy: "open"` para permitir a cualquier miembro (aun así, con control por mención de forma predeterminada).
- Para no permitir **ningún canal**, establece `channels.msteams.groupPolicy: "disabled"`.

Ejemplo:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["user@org.com"],
    },
  },
}
```

**Teams + lista de permitidos de canales**

- Limita las respuestas en grupos/canales enumerando equipos y canales en `channels.msteams.teams`.
- Las claves deben usar IDs estables de equipo e IDs de conversación de canal.
- Cuando `groupPolicy="allowlist"` y hay una lista de permitidos de equipos, solo se aceptan los equipos/canales enumerados (con control por mención).
- El asistente de configuración acepta entradas `Team/Channel` y las almacena por ti.
- Al iniciarse, OpenClaw resuelve los nombres de equipos/canales y de usuarios de la lista de permitidos a IDs (cuando los permisos de Graph lo permiten)
  y registra la asignación; los nombres de equipos/canales no resueltos se conservan tal como se escribieron, pero se ignoran para el enrutamiento de forma predeterminada a menos que `channels.msteams.dangerouslyAllowNameMatching: true` esté habilitado.

Ejemplo:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

## Cómo funciona

1. Asegúrate de que el plugin de Microsoft Teams esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea un **Azure Bot** (App ID + secreto + tenant ID).
3. Crea un **paquete de aplicación de Teams** que haga referencia al bot e incluya los permisos RSC indicados a continuación.
4. Carga/instala la aplicación de Teams en un equipo (o en ámbito personal para MD).
5. Configura `msteams` en `~/.openclaw/openclaw.json` (o variables de entorno) e inicia el gateway.
6. El gateway escucha el tráfico de webhook de Bot Framework en `/api/messages` de forma predeterminada.

## Configuración de Azure Bot (requisitos previos)

Antes de configurar OpenClaw, necesitas crear un recurso de Azure Bot.

### Paso 1: Crear Azure Bot

1. Ve a [Crear Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot)
2. Completa la pestaña **Básico**:

   | Campo              | Valor                                                    |
   | ------------------ | -------------------------------------------------------- |
   | **Identificador del bot** | El nombre de tu bot, p. ej., `openclaw-msteams` (debe ser único) |
   | **Suscripción**   | Selecciona tu suscripción de Azure                           |
   | **Grupo de recursos** | Crea uno nuevo o usa uno existente                               |
   | **Plan de precios**   | **Gratis** para desarrollo/pruebas                                 |
   | **Tipo de aplicación**    | **Single Tenant** (recomendado; consulta la nota abajo)         |
   | **Tipo de creación**  | **Create new Microsoft App ID**                          |

> **Aviso de retirada:** La creación de nuevos bots multiinquilino quedó obsoleta después del 2025-07-31. Usa **Single Tenant** para bots nuevos.

3. Haz clic en **Revisar + crear** → **Crear** (espera ~1-2 minutos)

### Paso 2: Obtener credenciales

1. Ve a tu recurso de Azure Bot → **Configuración**
2. Copia **Microsoft App ID** → este es tu `appId`
3. Haz clic en **Manage Password** → ve al registro de la aplicación
4. En **Certificates & secrets** → **New client secret** → copia el **Value** → este es tu `appPassword`
5. Ve a **Overview** → copia **Directory (tenant) ID** → este es tu `tenantId`

### Paso 3: Configurar el endpoint de mensajería

1. En Azure Bot → **Configuración**
2. Establece **Messaging endpoint** en la URL de tu webhook:
   - Producción: `https://your-domain.com/api/messages`
   - Desarrollo local: usa un túnel (consulta [Desarrollo local](#desarrollo-local-túneles) abajo)

### Paso 4: Habilitar el canal Teams

1. En Azure Bot → **Channels**
2. Haz clic en **Microsoft Teams** → Configurar → Guardar
3. Acepta los términos del servicio

## Desarrollo local (túneles)

Teams no puede acceder a `localhost`. Usa un túnel para el desarrollo local:

**Opción A: ngrok**

```bash
ngrok http 3978
# Copia la URL https, por ejemplo, https://abc123.ngrok.io
# Establece el endpoint de mensajería en: https://abc123.ngrok.io/api/messages
```

**Opción B: Tailscale Funnel**

```bash
tailscale funnel 3978
# Usa tu URL de Tailscale Funnel como endpoint de mensajería
```

## Teams Developer Portal (alternativa)

En lugar de crear manualmente un ZIP de manifiesto, puedes usar el [Teams Developer Portal](https://dev.teams.microsoft.com/apps):

1. Haz clic en **+ New app**
2. Completa la información básica (nombre, descripción, información del desarrollador)
3. Ve a **App features** → **Bot**
4. Selecciona **Enter a bot ID manually** y pega tu Azure Bot App ID
5. Marca los ámbitos: **Personal**, **Team**, **Group Chat**
6. Haz clic en **Distribute** → **Download app package**
7. En Teams: **Apps** → **Manage your apps** → **Upload a custom app** → selecciona el ZIP

Esto suele ser más sencillo que editar manualmente manifiestos JSON.

## Probar el bot

**Opción A: Azure Web Chat (verifica primero el webhook)**

1. En Azure Portal → tu recurso de Azure Bot → **Test in Web Chat**
2. Envía un mensaje: deberías ver una respuesta
3. Esto confirma que tu endpoint de webhook funciona antes de la configuración de Teams

**Opción B: Teams (después de instalar la aplicación)**

1. Instala la aplicación de Teams (carga lateral o catálogo de la organización)
2. Busca el bot en Teams y envía un MD
3. Revisa los registros del gateway para ver la actividad entrante

## Configuración (mínima, solo texto)

1. **Asegúrate de que el plugin de Microsoft Teams esté disponible**
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente:
     - Desde npm: `openclaw plugins install @openclaw/msteams`
     - Desde una copia local: `openclaw plugins install ./path/to/local/msteams-plugin`

2. **Registro del bot**
   - Crea un Azure Bot (consulta arriba) y anota:
     - App ID
     - Secreto de cliente (contraseña de la app)
     - Tenant ID (single-tenant)

3. **Manifiesto de la aplicación de Teams**
   - Incluye una entrada `bot` con `botId = <App ID>`.
   - Ámbitos: `personal`, `team`, `groupChat`.
   - `supportsFiles: true` (obligatorio para el manejo de archivos en ámbito personal).
   - Añade permisos RSC (abajo).
   - Crea iconos: `outline.png` (32x32) y `color.png` (192x192).
   - Comprime los tres archivos juntos: `manifest.json`, `outline.png`, `color.png`.

4. **Configura OpenClaw**

   ```json5
   {
     channels: {
       msteams: {
         enabled: true,
         appId: "<APP_ID>",
         appPassword: "<APP_PASSWORD>",
         tenantId: "<TENANT_ID>",
         webhook: { port: 3978, path: "/api/messages" },
       },
     },
   }
   ```

   También puedes usar variables de entorno en lugar de claves de configuración:
   - `MSTEAMS_APP_ID`
   - `MSTEAMS_APP_PASSWORD`
   - `MSTEAMS_TENANT_ID`

5. **Endpoint del bot**
   - Establece el Azure Bot Messaging Endpoint en:
     - `https://<host>:3978/api/messages` (o la ruta/puerto que hayas elegido).

6. **Ejecuta el gateway**
   - El canal Teams se inicia automáticamente cuando el plugin integrado o instalado manualmente está disponible y existe configuración `msteams` con credenciales.

## Acción de información de miembro

OpenClaw expone una acción `member-info` respaldada por Graph para Microsoft Teams, de modo que los agentes y las automatizaciones pueden resolver detalles de miembros del canal (nombre para mostrar, correo electrónico, rol) directamente desde Microsoft Graph.

Requisitos:

- Permiso RSC `Member.Read.Group` (ya incluido en el manifiesto recomendado)
- Para búsquedas entre equipos: permiso de aplicación Graph `User.Read.All` con consentimiento de administrador

La acción está controlada por `channels.msteams.actions.memberInfo` (habilitada de forma predeterminada cuando hay credenciales de Graph disponibles).

## Contexto del historial

- `channels.msteams.historyLimit` controla cuántos mensajes recientes de canal/grupo se encapsulan en el prompt.
- Recurre a `messages.groupChat.historyLimit`. Establece `0` para deshabilitarlo (predeterminado 50).
- El historial de hilos recuperado se filtra por las listas de permitidos de remitentes (`allowFrom` / `groupAllowFrom`), de modo que la siembra del contexto del hilo solo incluye mensajes de remitentes permitidos.
- El contexto de archivos adjuntos citados (`ReplyTo*` derivado del HTML de respuesta de Teams) actualmente se transmite tal como se recibe.
- En otras palabras, las listas de permitidos controlan quién puede activar al agente; solo determinadas rutas de contexto suplementario se filtran actualmente.
- El historial de MD puede limitarse con `channels.msteams.dmHistoryLimit` (turnos del usuario). Anulaciones por usuario: `channels.msteams.dms["<user_id>"].historyLimit`.

## Permisos RSC actuales de Teams (manifiesto)

Estos son los **permisos resourceSpecific existentes** en nuestro manifiesto de la aplicación de Teams. Solo se aplican dentro del equipo/chat donde la aplicación está instalada.

**Para canales (ámbito de equipo):**

- `ChannelMessage.Read.Group` (Application) - recibir todos los mensajes del canal sin @mention
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**Para chats grupales:**

- `ChatMessage.Read.Chat` (Application) - recibir todos los mensajes del chat grupal sin @mention

## Ejemplo de manifiesto de Teams (redactado)

Ejemplo mínimo y válido con los campos obligatorios. Sustituye los IDs y las URL.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Your Org",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### Advertencias del manifiesto (campos imprescindibles)

- `bots[].botId` **debe** coincidir con el Azure Bot App ID.
- `webApplicationInfo.id` **debe** coincidir con el Azure Bot App ID.
- `bots[].scopes` debe incluir las superficies que planeas usar (`personal`, `team`, `groupChat`).
- `bots[].supportsFiles: true` es obligatorio para el manejo de archivos en ámbito personal.
- `authorization.permissions.resourceSpecific` debe incluir lectura/envío de canal si quieres tráfico de canal.

### Actualizar una aplicación existente

Para actualizar una aplicación de Teams ya instalada (por ejemplo, para añadir permisos RSC):

1. Actualiza tu `manifest.json` con la nueva configuración
2. **Incrementa el campo `version`** (por ejemplo, `1.0.0` → `1.1.0`)
3. **Vuelve a comprimir** el manifiesto con los iconos (`manifest.json`, `outline.png`, `color.png`)
4. Carga el nuevo ZIP:
   - **Opción A (Teams Admin Center):** Teams Admin Center → Teams apps → Manage apps → busca tu aplicación → Upload new version
   - **Opción B (carga lateral):** En Teams → Apps → Manage your apps → Upload a custom app
5. **Para canales de equipo:** reinstala la aplicación en cada equipo para que los nuevos permisos surtan efecto
6. **Cierra Teams por completo y vuelve a iniciarlo** (no basta con cerrar la ventana) para borrar los metadatos de aplicación en caché

## Capacidades: solo RSC frente a Graph

### Con **solo Teams RSC** (aplicación instalada, sin permisos de Graph API)

Funciona:

- Leer contenido de **texto** de mensajes de canal.
- Enviar contenido de **texto** de mensajes de canal.
- Recibir archivos adjuntos de **archivos en chats personales (MD)**.

NO funciona:

- Contenido de **imágenes o archivos** de canal/grupo (la carga útil solo incluye un stub HTML).
- Descargar archivos adjuntos almacenados en SharePoint/OneDrive.
- Leer el historial de mensajes (más allá del evento del webhook en vivo).

### Con **Teams RSC + permisos de aplicación de Microsoft Graph**

Añade:

- Descarga de contenidos alojados (imágenes pegadas en mensajes).
- Descarga de archivos adjuntos almacenados en SharePoint/OneDrive.
- Lectura del historial de mensajes de canal/chat mediante Graph.

### RSC frente a Graph API

| Capacidad              | Permisos RSC         | Graph API                           |
| ---------------------- | -------------------- | ----------------------------------- |
| **Mensajes en tiempo real**  | Sí (mediante webhook)    | No (solo sondeo)                   |
| **Mensajes históricos** | No                   | Sí (puede consultar historial)             |
| **Complejidad de configuración**    | Solo manifiesto de la aplicación    | Requiere consentimiento de administrador + flujo de tokens |
| **Funciona sin conexión**       | No (debe estar en ejecución) | Sí (puede consultar en cualquier momento)                 |

**En resumen:** RSC es para escucha en tiempo real; Graph API es para acceso histórico. Para ponerse al día con mensajes perdidos mientras estabas desconectado, necesitas Graph API con `ChannelMessage.Read.All` (requiere consentimiento de administrador).

## Medios + historial con Graph habilitado (obligatorio para canales)

Si necesitas imágenes/archivos en **canales** o quieres obtener **historial de mensajes**, debes habilitar permisos de Microsoft Graph y conceder consentimiento de administrador.

1. En **App Registration** de Entra ID (Azure AD), añade permisos de **Application** de Microsoft Graph:
   - `ChannelMessage.Read.All` (archivos adjuntos e historial de canal)
   - `Chat.Read.All` o `ChatMessage.Read.All` (chats grupales)
2. **Concede consentimiento de administrador** para el tenant.
3. Incrementa la **versión del manifiesto** de la aplicación de Teams, vuelve a cargarlo y **reinstala la aplicación en Teams**.
4. **Cierra Teams por completo y vuelve a iniciarlo** para borrar los metadatos de aplicación en caché.

**Permiso adicional para menciones de usuario:** Las menciones @user funcionan de inmediato para usuarios que están en la conversación. Sin embargo, si quieres buscar y mencionar dinámicamente a usuarios que **no están en la conversación actual**, añade el permiso `User.Read.All` (Application) y concede consentimiento de administrador.

## Limitaciones conocidas

### Tiempos de espera del webhook

Teams entrega mensajes mediante webhook HTTP. Si el procesamiento tarda demasiado (por ejemplo, respuestas lentas del LLM), puedes ver:

- Tiempos de espera del gateway
- Teams reintentando el mensaje (provocando duplicados)
- Respuestas descartadas

OpenClaw gestiona esto devolviendo rápidamente y enviando respuestas de forma proactiva, pero las respuestas muy lentas aún pueden causar problemas.

### Formato

El markdown de Teams es más limitado que el de Slack o Discord:

- El formato básico funciona: **negrita**, _cursiva_, `code`, enlaces
- El markdown complejo (tablas, listas anidadas) puede no renderizarse correctamente
- Adaptive Cards son compatibles para encuestas y envíos arbitrarios de tarjetas (consulta abajo)

## Configuración

Ajustes clave (consulta `/gateway/configuration` para patrones de canal compartidos):

- `channels.msteams.enabled`: habilitar/deshabilitar el canal.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: credenciales del bot.
- `channels.msteams.webhook.port` (predeterminado `3978`)
- `channels.msteams.webhook.path` (predeterminado `/api/messages`)
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (predeterminado: pairing)
- `channels.msteams.allowFrom`: lista de permitidos de MD (se recomiendan IDs de objeto AAD). El asistente resuelve nombres a IDs durante la configuración cuando el acceso a Graph está disponible.
- `channels.msteams.dangerouslyAllowNameMatching`: interruptor de emergencia para volver a habilitar la coincidencia mutable de UPN/nombre para mostrar y el enrutamiento directo por nombre de equipo/canal.
- `channels.msteams.textChunkLimit`: tamaño del fragmento de texto saliente.
- `channels.msteams.chunkMode`: `length` (predeterminado) o `newline` para dividir en líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.msteams.mediaAllowHosts`: lista de permitidos de hosts para archivos adjuntos entrantes (predeterminada para dominios de Microsoft/Teams).
- `channels.msteams.mediaAuthAllowHosts`: lista de permitidos para adjuntar encabezados Authorization en reintentos de medios (predeterminada para hosts de Graph + Bot Framework).
- `channels.msteams.requireMention`: exigir @mention en canales/grupos (predeterminado true).
- `channels.msteams.replyStyle`: `thread | top-level` (consulta [Estilo de respuesta](#estilo-de-respuesta-hilos-frente-a-publicaciones)).
- `channels.msteams.teams.<teamId>.replyStyle`: anulación por equipo.
- `channels.msteams.teams.<teamId>.requireMention`: anulación por equipo.
- `channels.msteams.teams.<teamId>.tools`: anulaciones predeterminadas de política de herramientas por equipo (`allow`/`deny`/`alsoAllow`) usadas cuando falta una anulación de canal.
- `channels.msteams.teams.<teamId>.toolsBySender`: anulaciones predeterminadas de política de herramientas por equipo y remitente (`"*"` comodín admitido).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: anulación por canal.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: anulación por canal.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: anulaciones de política de herramientas por canal (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: anulaciones de política de herramientas por canal y remitente (`"*"` comodín admitido).
- Las claves `toolsBySender` deben usar prefijos explícitos:
  `id:`, `e164:`, `username:`, `name:` (las claves heredadas sin prefijo siguen asignándose solo a `id:`).
- `channels.msteams.actions.memberInfo`: habilitar o deshabilitar la acción de información de miembro respaldada por Graph (habilitada de forma predeterminada cuando hay credenciales de Graph disponibles).
- `channels.msteams.sharePointSiteId`: ID del sitio de SharePoint para cargas de archivos en chats grupales/canales (consulta [Envío de archivos en chats grupales](#envío-de-archivos-en-chats-grupales)).

## Enrutamiento y sesiones

- Las claves de sesión siguen el formato estándar de agente (consulta [/concepts/session](/concepts/session)):
  - Los mensajes directos comparten la sesión principal (`agent:<agentId>:<mainKey>`).
  - Los mensajes de canal/grupo usan el ID de conversación:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## Estilo de respuesta: hilos frente a publicaciones

Teams introdujo recientemente dos estilos de interfaz de canal sobre el mismo modelo de datos subyacente:

| Estilo                    | Descripción                                               | `replyStyle` recomendado |
| ------------------------ | --------------------------------------------------------- | ------------------------ |
| **Posts** (clásico)      | Los mensajes aparecen como tarjetas con respuestas en hilo debajo | `thread` (predeterminado)       |
| **Threads** (tipo Slack) | Los mensajes fluyen linealmente, más parecido a Slack                   | `top-level`              |

**El problema:** La API de Teams no expone qué estilo de interfaz usa un canal. Si usas el `replyStyle` incorrecto:

- `thread` en un canal con estilo Threads → las respuestas aparecen anidadas de forma extraña
- `top-level` en un canal con estilo Posts → las respuestas aparecen como publicaciones separadas de nivel superior en lugar de en el hilo

**Solución:** Configura `replyStyle` por canal según cómo esté configurado el canal:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

## Archivos adjuntos e imágenes

**Limitaciones actuales:**

- **MD:** Las imágenes y los archivos adjuntos funcionan mediante las API de archivos del bot de Teams.
- **Canales/grupos:** Los archivos adjuntos residen en el almacenamiento de M365 (SharePoint/OneDrive). La carga útil del webhook solo incluye un stub HTML, no los bytes reales del archivo. **Se requieren permisos de Graph API** para descargar archivos adjuntos de canal.
- Para envíos explícitos centrados primero en archivos, usa `action=upload-file` con `media` / `filePath` / `path`; el `message` opcional se convierte en el texto/comentario adjunto, y `filename` sustituye el nombre cargado.

Sin permisos de Graph, los mensajes de canal con imágenes se recibirán solo como texto (el contenido de la imagen no es accesible para el bot).
De forma predeterminada, OpenClaw solo descarga medios desde nombres de host de Microsoft/Teams. Anúlalo con `channels.msteams.mediaAllowHosts` (usa `["*"]` para permitir cualquier host).
Los encabezados Authorization solo se adjuntan para los hosts de `channels.msteams.mediaAuthAllowHosts` (predeterminados para hosts de Graph + Bot Framework). Mantén esta lista estricta (evita sufijos multiinquilino).

## Envío de archivos en chats grupales

Los bots pueden enviar archivos en MD usando el flujo FileConsentCard (integrado). Sin embargo, **enviar archivos en chats grupales/canales** requiere configuración adicional:

| Contexto                  | Cómo se envían los archivos                           | Configuración necesaria                                    |
| ------------------------ | -------------------------------------------- | ----------------------------------------------- |
| **MD**                  | FileConsentCard → el usuario acepta → el bot carga | Funciona sin configuración adicional                            |
| **Chats grupales/canales** | Cargar en SharePoint → compartir enlace            | Requiere `sharePointSiteId` + permisos de Graph |
| **Imágenes (cualquier contexto)** | Inline codificado en Base64                        | Funciona sin configuración adicional                            |

### Por qué los chats grupales necesitan SharePoint

Los bots no tienen una unidad personal de OneDrive (el endpoint `/me/drive` de Graph API no funciona para identidades de aplicación). Para enviar archivos en chats grupales/canales, el bot carga el archivo en un **sitio de SharePoint** y crea un enlace para compartir.

### Configuración

1. **Añade permisos de Graph API** en Entra ID (Azure AD) → App Registration:
   - `Sites.ReadWrite.All` (Application) - cargar archivos en SharePoint
   - `Chat.Read.All` (Application) - opcional, habilita enlaces para compartir por usuario

2. **Concede consentimiento de administrador** para el tenant.

3. **Obtén el ID de tu sitio de SharePoint:**

   ```bash
   # Mediante Graph Explorer o curl con un token válido:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # Ejemplo: para un sitio en "contoso.sharepoint.com/sites/BotFiles"
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # La respuesta incluye: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **Configura OpenClaw:**

   ```json5
   {
     channels: {
       msteams: {
         // ... other config ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### Comportamiento de uso compartido

| Permiso                              | Comportamiento de uso compartido                                          |
| --------------------------------------- | --------------------------------------------------------- |
| `Sites.ReadWrite.All` solo              | Enlace de uso compartido para toda la organización (cualquiera en la organización puede acceder) |
| `Sites.ReadWrite.All` + `Chat.Read.All` | Enlace de uso compartido por usuario (solo los miembros del chat pueden acceder)      |

El uso compartido por usuario es más seguro, ya que solo los participantes del chat pueden acceder al archivo. Si falta el permiso `Chat.Read.All`, el bot recurre al uso compartido para toda la organización.

### Comportamiento de respaldo

| Escenario                                          | Resultado                                             |
| ------------------------------------------------- | -------------------------------------------------- |
| Chat grupal + archivo + `sharePointSiteId` configurado | Cargar en SharePoint, enviar enlace para compartir            |
| Chat grupal + archivo + sin `sharePointSiteId`         | Intentar carga en OneDrive (puede fallar), enviar solo texto |
| Chat personal + archivo                              | Flujo FileConsentCard (funciona sin SharePoint)    |
| Cualquier contexto + imagen                               | Inline codificado en Base64 (funciona sin SharePoint)   |

### Ubicación de almacenamiento de archivos

Los archivos cargados se almacenan en una carpeta `/OpenClawShared/` dentro de la biblioteca de documentos predeterminada del sitio de SharePoint configurado.

## Encuestas (Adaptive Cards)

OpenClaw envía encuestas de Teams como Adaptive Cards (no existe una API nativa de encuestas de Teams).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> ...`
- Los votos se registran por el gateway en `~/.openclaw/msteams-polls.json`.
- El gateway debe permanecer en línea para registrar votos.
- Las encuestas aún no publican automáticamente resúmenes de resultados (inspecciona el archivo de almacenamiento si es necesario).

## Adaptive Cards (arbitrarias)

Envía cualquier JSON de Adaptive Card a usuarios o conversaciones de Teams usando la herramienta `message` o la CLI.

El parámetro `card` acepta un objeto JSON de Adaptive Card. Cuando se proporciona `card`, el texto del mensaje es opcional.

**Herramienta de agente:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  card: {
    type: "AdaptiveCard",
    version: "1.5",
    body: [{ type: "TextBlock", text: "Hello!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Hello!"}]}'
```

Consulta la [documentación de Adaptive Cards](https://adaptivecards.io/) para ver el esquema de tarjetas y ejemplos. Para detalles sobre el formato de destino, consulta [Formatos de destino](#formatos-de-destino) abajo.

## Formatos de destino

Los destinos de MSTeams usan prefijos para distinguir entre usuarios y conversaciones:

| Tipo de destino         | Formato                           | Ejemplo                                             |
| ------------------- | -------------------------------- | --------------------------------------------------- |
| Usuario (por ID)        | `user:<aad-object-id>`           | `user:40a1a0ed-4ff2-4164-a219-55518990c197`         |
| Usuario (por nombre)      | `user:<display-name>`            | `user:John Smith` (requiere Graph API)              |
| Grupo/canal       | `conversation:<conversation-id>` | `conversation:19:abc123...@thread.tacv2`            |
| Grupo/canal (sin procesar) | `<conversation-id>`              | `19:abc123...@thread.tacv2` (si contiene `@thread`) |

**Ejemplos de CLI:**

```bash
# Enviar a un usuario por ID
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "Hello"

# Enviar a un usuario por nombre para mostrar (activa búsqueda en Graph API)
openclaw message send --channel msteams --target "user:John Smith" --message "Hello"

# Enviar a un chat grupal o canal
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "Hello"

# Enviar una Adaptive Card a una conversación
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Hello"}]}'
```

**Ejemplos de herramienta de agente:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "Hello!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  card: {
    type: "AdaptiveCard",
    version: "1.5",
    body: [{ type: "TextBlock", text: "Hello" }],
  },
}
```

Nota: Sin el prefijo `user:`, los nombres se interpretan por defecto como resolución de grupo/equipo. Usa siempre `user:` cuando apuntes a personas por nombre para mostrar.

## Mensajería proactiva

- Los mensajes proactivos solo son posibles **después** de que un usuario haya interactuado, porque almacenamos las referencias de conversación en ese momento.
- Consulta `/gateway/configuration` para `dmPolicy` y el control por lista de permitidos.

## IDs de equipo y canal (error habitual)

El parámetro de consulta `groupId` en las URL de Teams **NO** es el ID de equipo usado para la configuración. Extrae los IDs de la ruta de la URL:

**URL de equipo:**

```
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    ID de equipo (decodifica la URL)
```

**URL de canal:**

```
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      ID de canal (decodifica la URL)
```

**Para la configuración:**

- ID de equipo = segmento de ruta después de `/team/` (decodificado de URL, p. ej., `19:Bk4j...@thread.tacv2`)
- ID de canal = segmento de ruta después de `/channel/` (decodificado de URL)
- **Ignora** el parámetro de consulta `groupId`

## Canales privados

Los bots tienen compatibilidad limitada en canales privados:

| Función                      | Canales estándar | Canales privados       |
| ---------------------------- | ----------------- | ---------------------- |
| Instalación del bot             | Sí               | Limitada                |
| Mensajes en tiempo real (webhook) | Sí               | Puede que no funcione           |
| Permisos RSC              | Sí               | Puede comportarse de forma diferente |
| @mentions                    | Sí               | Si el bot es accesible   |
| Historial de Graph API            | Sí               | Sí (con permisos) |

**Soluciones alternativas si los canales privados no funcionan:**

1. Usa canales estándar para las interacciones con el bot
2. Usa MD: los usuarios siempre pueden enviar mensajes directos al bot
3. Usa Graph API para acceso histórico (requiere `ChannelMessage.Read.All`)

## Solución de problemas

### Problemas comunes

- **Las imágenes no aparecen en canales:** faltan permisos de Graph o consentimiento de administrador. Reinstala la aplicación de Teams y cierra/reabre Teams por completo.
- **No hay respuestas en el canal:** las menciones son obligatorias de forma predeterminada; establece `channels.msteams.requireMention=false` o configúralo por equipo/canal.
- **Incompatibilidad de versión (Teams sigue mostrando el manifiesto antiguo):** elimina y vuelve a añadir la aplicación y cierra Teams por completo para actualizar.
- **401 Unauthorized del webhook:** Esperado al probar manualmente sin JWT de Azure; significa que el endpoint es accesible, pero la autenticación falló. Usa Azure Web Chat para probar correctamente.

### Errores al cargar el manifiesto

- **"Icon file cannot be empty":** El manifiesto hace referencia a archivos de icono de 0 bytes. Crea iconos PNG válidos (32x32 para `outline.png`, 192x192 para `color.png`).
- **"webApplicationInfo.Id already in use":** La aplicación sigue instalada en otro equipo/chat. Búscala y desinstálala primero, o espera 5-10 minutos para la propagación.
- **"Something went wrong" al cargar:** Cárgalo mediante [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com), abre las DevTools del navegador (F12) → pestaña Network, y revisa el cuerpo de la respuesta para ver el error real.
- **Fallo en la carga lateral:** Prueba "Upload an app to your org's app catalog" en lugar de "Upload a custom app"; esto suele evitar restricciones de carga lateral.

### Los permisos RSC no funcionan

1. Verifica que `webApplicationInfo.id` coincida exactamente con el App ID de tu bot
2. Vuelve a cargar la aplicación y reinstálala en el equipo/chat
3. Comprueba si el administrador de tu organización ha bloqueado los permisos RSC
4. Confirma que estás usando el ámbito correcto: `ChannelMessage.Read.Group` para equipos, `ChatMessage.Read.Chat` para chats grupales

## Referencias

- [Create Azure Bot](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - guía de configuración de Azure Bot
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - crear/gestionar aplicaciones de Teams
- [Esquema del manifiesto de la aplicación de Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [Recibir mensajes de canal con RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [Referencia de permisos RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Manejo de archivos del bot de Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) (canal/grupo requiere Graph)
- [Mensajería proactiva](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de MD y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y control por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y endurecimiento
