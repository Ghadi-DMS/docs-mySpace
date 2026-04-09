---
read_when:
    - Configurar Matrix en OpenClaw
    - Configurar E2EE y verificación de Matrix
summary: Estado de compatibilidad de Matrix, configuración y ejemplos de configuración
title: Matrix
x-i18n:
    generated_at: "2026-04-09T01:29:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28fc13c7620c1152200315ae69c94205da6de3180c53c814dd8ce03b5cb1758f
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix es un plugin de canal incluido con OpenClaw.
Usa el `matrix-js-sdk` oficial y admite mensajes directos, salas, hilos, medios, reacciones, encuestas, ubicación y E2EE.

## Plugin incluido

Matrix se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación aparte.

Si usas una compilación anterior o una instalación personalizada que excluye Matrix, instálalo manualmente:

Instalar desde npm:

```bash
openclaw plugins install @openclaw/matrix
```

Instalar desde una copia local:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Consulta [Plugins](/es/tools/plugin) para conocer el comportamiento de los plugins y las reglas de instalación.

## Configuración

1. Asegúrate de que el plugin de Matrix esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas o personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea una cuenta de Matrix en tu homeserver.
3. Configura `channels.matrix` con una de estas opciones:
   - `homeserver` + `accessToken`, o
   - `homeserver` + `userId` + `password`.
4. Reinicia la puerta de enlace.
5. Inicia un mensaje directo con el bot o invítalo a una sala.
   - Las invitaciones nuevas de Matrix solo funcionan cuando `channels.matrix.autoJoin` las permite.

Rutas de configuración interactiva:

```bash
openclaw channels add
openclaw configure --section channels
```

El asistente de Matrix solicita:

- URL del homeserver
- método de autenticación: token de acceso o contraseña
- ID de usuario (solo autenticación con contraseña)
- nombre opcional del dispositivo
- si se debe habilitar E2EE
- si se debe configurar el acceso a salas y la unión automática a invitaciones

Comportamientos clave del asistente:

- Si las variables de entorno de autenticación de Matrix ya existen y esa cuenta aún no tiene autenticación guardada en la configuración, el asistente ofrece un atajo de variables de entorno para mantener la autenticación en variables de entorno.
- Los nombres de cuenta se normalizan al ID de cuenta. Por ejemplo, `Ops Bot` se convierte en `ops-bot`.
- Las entradas de la lista de permitidos para mensajes directos aceptan `@user:server` directamente; los nombres visibles solo funcionan cuando la búsqueda en directorio en vivo encuentra una coincidencia exacta.
- Las entradas de la lista de permitidos para salas aceptan directamente ID y alias de sala. Prefiere `!room:server` o `#alias:server`; los nombres no resueltos se ignoran en tiempo de ejecución durante la resolución de la lista de permitidos.
- En el modo de lista de permitidos de unión automática a invitaciones, usa solo destinos de invitación estables: `!roomId:server`, `#alias:server` o `*`. Los nombres simples de sala se rechazan.
- Para resolver nombres de sala antes de guardar, usa `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` usa `off` de forma predeterminada.

Si lo dejas sin configurar, el bot no se unirá a salas invitadas ni a invitaciones nuevas de estilo DM, por lo que no aparecerá en grupos nuevos ni en DMs invitados a menos que primero te unas manualmente.

Establece `autoJoin: "allowlist"` junto con `autoJoinAllowlist` para restringir qué invitaciones acepta, o establece `autoJoin: "always"` si quieres que se una a todas las invitaciones.

En el modo `allowlist`, `autoJoinAllowlist` solo acepta `!roomId:server`, `#alias:server` o `*`.
</Warning>

Ejemplo de lista de permitidos:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Unirse a todas las invitaciones:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

Configuración mínima basada en token:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Configuración basada en contraseña (el token se almacena en caché después del inicio de sesión):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix almacena las credenciales en caché en `~/.openclaw/credentials/matrix/`.
La cuenta predeterminada usa `credentials.json`; las cuentas con nombre usan `credentials-<account>.json`.
Cuando existen credenciales en caché allí, OpenClaw considera Matrix como configurado para configuración inicial, doctor y detección del estado del canal, aunque la autenticación actual no esté establecida directamente en la configuración.

Equivalentes de variables de entorno (se usan cuando la clave de configuración no está establecida):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Para cuentas no predeterminadas, usa variables de entorno limitadas al ámbito de la cuenta:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Ejemplo para la cuenta `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Para el ID de cuenta normalizado `ops-bot`, usa:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix escapa la puntuación en los ID de cuenta para evitar colisiones en las variables de entorno acotadas al ámbito.
Por ejemplo, `-` se convierte en `_X2D_`, por lo que `ops-prod` se convierte en `MATRIX_OPS_X2D_PROD_*`.

El asistente interactivo solo ofrece el atajo de variables de entorno cuando esas variables de autenticación ya están presentes y la cuenta seleccionada aún no tiene autenticación de Matrix guardada en la configuración.

## Ejemplo de configuración

Esta es una configuración base práctica con emparejamiento de DM, lista de permitidos de sala y E2EE habilitado:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin` se aplica a todas las invitaciones de Matrix, incluidas las invitaciones de estilo DM. OpenClaw no puede clasificar de forma fiable una sala invitada como DM o grupo en el momento de la invitación, así que todas las invitaciones pasan primero por `autoJoin`. `dm.policy` se aplica después de que el bot se haya unido y la sala se haya clasificado como DM.

## Vistas previas en streaming

El streaming de respuestas en Matrix es opcional.

Establece `channels.matrix.streaming` en `"partial"` cuando quieras que OpenClaw envíe una única respuesta de vista previa en vivo, edite esa vista previa en el lugar mientras el modelo genera texto y luego la finalice cuando la respuesta esté lista:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` es el valor predeterminado. OpenClaw espera la respuesta final y la envía una sola vez.
- `streaming: "partial"` crea un único mensaje de vista previa editable para el bloque actual del asistente usando mensajes de texto normales de Matrix. Esto conserva el comportamiento heredado de Matrix de notificar primero la vista previa, por lo que los clientes estándar pueden notificar sobre el primer texto de vista previa en streaming en lugar del bloque finalizado.
- `streaming: "quiet"` crea un único aviso silencioso de vista previa editable para el bloque actual del asistente. Úsalo solo cuando también configures reglas push del destinatario para las ediciones de vista previa finalizadas.
- `blockStreaming: true` habilita mensajes de progreso de Matrix separados. Con la vista previa en streaming habilitada, Matrix mantiene el borrador en vivo del bloque actual y conserva los bloques completados como mensajes separados.
- Cuando la vista previa en streaming está activada y `blockStreaming` está desactivado, Matrix edita el borrador en vivo en el lugar y finaliza ese mismo evento cuando termina el bloque o el turno.
- Si la vista previa ya no cabe en un solo evento de Matrix, OpenClaw detiene la vista previa en streaming y vuelve a la entrega final normal.
- Las respuestas con medios siguen enviando adjuntos normalmente. Si una vista previa obsoleta ya no puede reutilizarse de forma segura, OpenClaw la redacta antes de enviar la respuesta final con medios.
- Las ediciones de vista previa implican llamadas adicionales a la API de Matrix. Deja el streaming desactivado si quieres el comportamiento más conservador respecto a límites de tasa.

`blockStreaming` no habilita por sí solo vistas previas de borrador.
Usa `streaming: "partial"` o `streaming: "quiet"` para las ediciones de vista previa; después añade `blockStreaming: true` solo si además quieres que los bloques completados del asistente sigan siendo visibles como mensajes de progreso separados.

Si necesitas notificaciones estándar de Matrix sin reglas push personalizadas, usa `streaming: "partial"` para el comportamiento de vista previa primero o deja `streaming` desactivado para entrega solo final. Con `streaming: "off"`:

- `blockStreaming: true` envía cada bloque terminado como un mensaje normal de Matrix con notificación.
- `blockStreaming: false` envía solo la respuesta final completa como un mensaje normal de Matrix con notificación.

### Reglas push autoalojadas para vistas previas silenciosas finalizadas

Si administras tu propia infraestructura de Matrix y quieres que las vistas previas silenciosas notifiquen solo cuando un bloque o la respuesta final haya terminado, establece `streaming: "quiet"` y añade una regla push por usuario para las ediciones de vista previa finalizadas.

Normalmente esta es una configuración del usuario destinatario, no un cambio global de configuración del homeserver:

Resumen rápido antes de empezar:

- usuario destinatario = la persona que debe recibir la notificación
- usuario bot = la cuenta de Matrix de OpenClaw que envía la respuesta
- usa el token de acceso del usuario destinatario para las llamadas a la API siguientes
- haz coincidir `sender` en la regla push con el MXID completo del usuario bot

1. Configura OpenClaw para usar vistas previas silenciosas:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Asegúrate de que la cuenta del destinatario ya reciba notificaciones push normales de Matrix. Las reglas de vista previa silenciosa solo funcionan si ese usuario ya tiene pushers o dispositivos funcionando.

3. Obtén el token de acceso del usuario destinatario.
   - Usa el token del usuario que recibe, no el del bot.
   - Reutilizar un token de sesión de cliente existente suele ser lo más sencillo.
   - Si necesitas emitir un token nuevo, puedes iniciar sesión mediante la API estándar Client-Server de Matrix:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Verifica que la cuenta del destinatario ya tenga pushers:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Si esto no devuelve pushers o dispositivos activos, primero corrige las notificaciones normales de Matrix antes de añadir la siguiente regla de OpenClaw.

OpenClaw marca las ediciones finalizadas de vista previa solo de texto con:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Crea una regla push de override para cada cuenta destinataria que deba recibir estas notificaciones:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Sustituye estos valores antes de ejecutar el comando:

- `https://matrix.example.org`: la URL base de tu homeserver
- `$USER_ACCESS_TOKEN`: el token de acceso del usuario receptor
- `openclaw-finalized-preview-botname`: un ID de regla único para este bot para este usuario receptor
- `@bot:example.org`: el MXID de tu bot de Matrix de OpenClaw, no el MXID del usuario receptor

Importante para configuraciones con varios bots:

- Las reglas push se indexan por `ruleId`. Volver a ejecutar `PUT` sobre el mismo ID de regla actualiza esa misma regla.
- Si un usuario receptor debe recibir notificaciones de varias cuentas de bot de Matrix de OpenClaw, crea una regla por bot con un ID de regla único para cada coincidencia de remitente.
- Un patrón sencillo es `openclaw-finalized-preview-<botname>`, como `openclaw-finalized-preview-ops` o `openclaw-finalized-preview-support`.

La regla se evalúa respecto al remitente del evento:

- autentícate con el token del usuario receptor
- haz coincidir `sender` con el MXID del bot de OpenClaw

6. Verifica que la regla exista:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Prueba una respuesta en streaming. En modo silencioso, la sala debería mostrar una vista previa de borrador silenciosa y la edición final en el lugar debería notificar una vez que termine el bloque o el turno.

Si necesitas eliminar la regla más adelante, borra ese mismo ID de regla con el token del usuario receptor:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Notas:

- Crea la regla con el token de acceso del usuario receptor, no con el del bot.
- Las nuevas reglas `override` definidas por el usuario se insertan antes de las reglas de supresión predeterminadas, así que no hace falta ningún parámetro adicional de orden.
- Esto solo afecta a las ediciones de vista previa solo de texto que OpenClaw puede finalizar en el lugar de forma segura. Los retrocesos de medios y los retrocesos por vista previa obsoleta siguen usando la entrega normal de Matrix.
- Si `GET /_matrix/client/v3/pushers` no muestra pushers, el usuario aún no tiene entrega push de Matrix funcionando para esa cuenta o dispositivo.

#### Synapse

Para Synapse, normalmente la configuración anterior es suficiente por sí sola:

- No se requiere ningún cambio especial en `homeserver.yaml` para las notificaciones de vista previa finalizada de OpenClaw.
- Si tu despliegue de Synapse ya envía notificaciones push normales de Matrix, el token de usuario y la llamada `pushrules` de arriba son el paso principal de configuración.
- Si ejecutas Synapse detrás de un proxy inverso o workers, asegúrate de que `/_matrix/client/.../pushrules/` llegue correctamente a Synapse.
- Si usas workers de Synapse, asegúrate de que los pushers estén en buen estado. La entrega push la gestiona el proceso principal o `synapse.app.pusher` o los workers pusher configurados.

#### Tuwunel

Para Tuwunel, usa el mismo flujo de configuración y la misma llamada a la API `pushrules` mostrada arriba:

- No se requiere ninguna configuración específica de Tuwunel para el propio marcador de vista previa finalizada.
- Si las notificaciones normales de Matrix ya funcionan para ese usuario, el token de usuario y la llamada `pushrules` anterior son el paso principal de configuración.
- Si las notificaciones parecen desaparecer mientras el usuario está activo en otro dispositivo, comprueba si `suppress_push_when_active` está habilitado. Tuwunel añadió esta opción en Tuwunel 1.4.2 el 12 de septiembre de 2025, y puede suprimir intencionadamente los avisos push a otros dispositivos mientras uno está activo.

## Salas de bot a bot

De forma predeterminada, se ignoran los mensajes de otras cuentas de Matrix de OpenClaw configuradas.

Usa `allowBots` cuando quieras intencionadamente tráfico de Matrix entre agentes:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` acepta mensajes de otras cuentas de bot de Matrix configuradas en salas y DMs permitidos.
- `allowBots: "mentions"` acepta esos mensajes solo cuando mencionan visiblemente a este bot en salas. Los DMs siguen permitidos.
- `groups.<room>.allowBots` sobrescribe la configuración a nivel de cuenta para una sala.
- OpenClaw sigue ignorando los mensajes del mismo ID de usuario de Matrix para evitar bucles de autorrespuesta.
- Matrix no expone aquí una marca nativa de bot; OpenClaw trata "escrito por bot" como "enviado por otra cuenta de Matrix configurada en esta puerta de enlace de OpenClaw".

Usa listas de permitidos estrictas para salas y requisitos de mención cuando habilites tráfico de bot a bot en salas compartidas.

## Cifrado y verificación

En las salas cifradas (E2EE), los eventos salientes de imagen usan `thumbnail_file` para que las vistas previas de imagen se cifren junto con el adjunto completo. Las salas no cifradas siguen usando `thumbnail_url` sin cifrar. No se necesita configuración: el plugin detecta automáticamente el estado E2EE.

Habilitar cifrado:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Comprobar el estado de verificación:

```bash
openclaw matrix verify status
```

Estado detallado (diagnóstico completo):

```bash
openclaw matrix verify status --verbose
```

Incluir la clave de recuperación almacenada en la salida legible por máquina:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Inicializar el estado de firma cruzada y verificación:

```bash
openclaw matrix verify bootstrap
```

Diagnóstico detallado del bootstrap:

```bash
openclaw matrix verify bootstrap --verbose
```

Forzar un reinicio completo de una identidad nueva de firma cruzada antes del bootstrap:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Verificar este dispositivo con una clave de recuperación:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Detalles detallados de verificación del dispositivo:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Comprobar el estado de salud de la copia de seguridad de claves de sala:

```bash
openclaw matrix verify backup status
```

Diagnóstico detallado del estado de la copia de seguridad:

```bash
openclaw matrix verify backup status --verbose
```

Restaurar claves de sala desde la copia de seguridad del servidor:

```bash
openclaw matrix verify backup restore
```

Diagnóstico detallado de restauración:

```bash
openclaw matrix verify backup restore --verbose
```

Eliminar la copia de seguridad actual del servidor y crear una nueva base de copia de seguridad. Si la clave de copia de seguridad almacenada no se puede cargar limpiamente, este restablecimiento también puede recrear el almacenamiento secreto para que futuros inicios en frío puedan cargar la nueva clave de copia de seguridad:

```bash
openclaw matrix verify backup reset --yes
```

Todos los comandos `verify` son concisos de forma predeterminada (incluido el registro silencioso interno del SDK) y muestran diagnósticos detallados solo con `--verbose`.
Usa `--json` para obtener salida completa legible por máquina en scripts.

En configuraciones de varias cuentas, los comandos CLI de Matrix usan la cuenta predeterminada implícita de Matrix a menos que pases `--account <id>`.
Si configuras varias cuentas con nombre, primero establece `channels.matrix.defaultAccount` o esas operaciones CLI implícitas se detendrán y te pedirán que elijas una cuenta explícitamente.
Usa `--account` siempre que quieras que las operaciones de verificación o de dispositivo apunten explícitamente a una cuenta con nombre:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Cuando el cifrado está deshabilitado o no disponible para una cuenta con nombre, las advertencias de Matrix y los errores de verificación apuntan a la clave de configuración de esa cuenta, por ejemplo `channels.matrix.accounts.assistant.encryption`.

### Qué significa "verificado"

OpenClaw trata este dispositivo de Matrix como verificado solo cuando está verificado por tu propia identidad de firma cruzada.
En la práctica, `openclaw matrix verify status --verbose` expone tres señales de confianza:

- `Locally trusted`: este dispositivo solo es de confianza para el cliente actual
- `Cross-signing verified`: el SDK informa que el dispositivo está verificado mediante firma cruzada
- `Signed by owner`: el dispositivo está firmado por tu propia clave de autofirma

`Verified by owner` pasa a ser `yes` solo cuando hay verificación mediante firma cruzada o firma del propietario.
La confianza local por sí sola no basta para que OpenClaw trate el dispositivo como totalmente verificado.

### Qué hace bootstrap

`openclaw matrix verify bootstrap` es el comando de reparación y configuración para cuentas cifradas de Matrix.
Hace todo lo siguiente en este orden:

- inicializa el almacenamiento secreto, reutilizando una clave de recuperación existente cuando sea posible
- inicializa la firma cruzada y sube las claves públicas de firma cruzada que falten
- intenta marcar y firmar mediante firma cruzada el dispositivo actual
- crea una nueva copia de seguridad de claves de sala en el servidor si todavía no existe una

Si el homeserver requiere autenticación interactiva para subir claves de firma cruzada, OpenClaw intenta primero la subida sin autenticación, luego con `m.login.dummy` y después con `m.login.password` cuando `channels.matrix.password` está configurado.

Usa `--force-reset-cross-signing` solo cuando quieras intencionadamente descartar la identidad actual de firma cruzada y crear una nueva.

Si quieres descartar intencionadamente la copia de seguridad actual de claves de sala y empezar una nueva base de copia de seguridad para mensajes futuros, usa `openclaw matrix verify backup reset --yes`.
Hazlo solo si aceptas que el historial cifrado antiguo irrecuperable seguirá sin estar disponible y que OpenClaw podría recrear el almacenamiento secreto si el secreto actual de copia de seguridad no puede cargarse de forma segura.

### Nueva base de copia de seguridad

Si quieres mantener funcionando los futuros mensajes cifrados y aceptas perder el historial antiguo irrecuperable, ejecuta estos comandos en orden:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Añade `--account <id>` a cada comando cuando quieras apuntar explícitamente a una cuenta de Matrix con nombre.

### Comportamiento al inicio

Cuando `encryption: true`, Matrix usa `"if-unverified"` como valor predeterminado para `startupVerification`.
Al inicio, si este dispositivo sigue sin verificar, Matrix solicitará una autoverificación en otro cliente de Matrix, omitirá solicitudes duplicadas mientras ya haya una pendiente y aplicará un tiempo de espera local antes de reintentar después de reinicios.
Los intentos de solicitud fallidos se reintentan antes que la creación correcta de solicitudes de forma predeterminada.
Establece `startupVerification: "off"` para desactivar las solicitudes automáticas al inicio, o ajusta `startupVerificationCooldownHours` si quieres una ventana de reintento más corta o más larga.

El inicio también realiza automáticamente un paso conservador de bootstrap criptográfico.
Ese paso intenta primero reutilizar el almacenamiento secreto actual y la identidad actual de firma cruzada, y evita restablecer la firma cruzada a menos que ejecutes un flujo explícito de reparación de bootstrap.

Si durante el inicio se detecta un estado de bootstrap dañado y `channels.matrix.password` está configurado, OpenClaw puede intentar una ruta de reparación más estricta.
Si el dispositivo actual ya está firmado por el propietario, OpenClaw preserva esa identidad en lugar de restablecerla automáticamente.

Consulta [Migración de Matrix](/es/install/migrating-matrix) para ver el flujo completo de actualización, los límites, los comandos de recuperación y los mensajes de migración comunes.

### Avisos de verificación

Matrix publica avisos del ciclo de vida de verificación directamente en la sala estricta de DM de verificación como mensajes `m.notice`.
Esto incluye:

- avisos de solicitud de verificación
- avisos de verificación lista (con orientación explícita "Verificar por emoji")
- inicio y finalización de verificación
- detalles SAS (emoji y decimal) cuando estén disponibles

Las solicitudes de verificación entrantes desde otro cliente de Matrix se rastrean y OpenClaw las acepta automáticamente.
Para flujos de autoverificación, OpenClaw también inicia automáticamente el flujo SAS cuando la verificación por emoji pasa a estar disponible y confirma su propio lado.
Para solicitudes de verificación de otro usuario o dispositivo de Matrix, OpenClaw acepta automáticamente la solicitud y luego espera a que el flujo SAS continúe de forma normal.
Aun así, debes comparar el SAS de emoji o decimal en tu cliente de Matrix y confirmar allí "Coinciden" para completar la verificación.

OpenClaw no acepta automáticamente a ciegas flujos duplicados iniciados por sí mismo. El inicio omite crear una nueva solicitud cuando ya hay una solicitud de autoverificación pendiente.

Los avisos del protocolo o sistema de verificación no se reenvían a la canalización de chat del agente, así que no producen `NO_REPLY`.

### Higiene de dispositivos

Los dispositivos antiguos de Matrix gestionados por OpenClaw pueden acumularse en la cuenta y dificultar entender la confianza en salas cifradas.
Haz una lista con:

```bash
openclaw matrix devices list
```

Elimina dispositivos obsoletos gestionados por OpenClaw con:

```bash
openclaw matrix devices prune-stale
```

### Almacén criptográfico

E2EE de Matrix usa la ruta criptográfica Rust oficial de `matrix-js-sdk` en Node, con `fake-indexeddb` como shim de IndexedDB. El estado criptográfico se conserva en un archivo de instantánea (`crypto-idb-snapshot.json`) y se restaura al inicio. El archivo de instantánea es un estado de ejecución sensible almacenado con permisos de archivo restrictivos.

El estado cifrado de ejecución vive bajo raíces por cuenta, por usuario y por hash de token en
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Ese directorio contiene el almacén de sincronización (`bot-storage.json`), el almacén criptográfico (`crypto/`),
el archivo de clave de recuperación (`recovery-key.json`), la instantánea de IndexedDB (`crypto-idb-snapshot.json`),
los enlaces de hilos (`thread-bindings.json`) y el estado de verificación al inicio (`startup-verification.json`).
Cuando el token cambia pero la identidad de la cuenta sigue siendo la misma, OpenClaw reutiliza la mejor
raíz existente para esa tupla de cuenta, homeserver y usuario para que el estado de sincronización previo, el estado criptográfico, los enlaces de hilos
y el estado de verificación al inicio sigan estando visibles.

## Gestión de perfil

Actualiza el perfil propio de Matrix para la cuenta seleccionada con:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Añade `--account <id>` cuando quieras apuntar explícitamente a una cuenta de Matrix con nombre.

Matrix acepta directamente URL de avatar `mxc://`. Cuando pasas una URL de avatar `http://` o `https://`, OpenClaw la sube primero a Matrix y almacena la URL `mxc://` resuelta de vuelta en `channels.matrix.avatarUrl` (o en la sobrescritura de la cuenta seleccionada).

## Hilos

Matrix admite hilos nativos de Matrix tanto para respuestas automáticas como para envíos mediante herramientas de mensajes.

- `dm.sessionScope: "per-user"` (predeterminado) mantiene el enrutamiento de DM de Matrix acotado al remitente, de modo que varias salas de DM puedan compartir una sesión cuando se resuelven al mismo par.
- `dm.sessionScope: "per-room"` aísla cada sala de DM de Matrix en su propia clave de sesión mientras sigue usando las comprobaciones normales de autenticación y lista de permitidos para DM.
- Los enlaces explícitos de conversación de Matrix siguen teniendo prioridad sobre `dm.sessionScope`, de modo que las salas y los hilos enlazados mantienen su sesión de destino elegida.
- `threadReplies: "off"` mantiene las respuestas en el nivel superior y conserva los mensajes entrantes con hilo en la sesión principal.
- `threadReplies: "inbound"` responde dentro de un hilo solo cuando el mensaje entrante ya estaba en ese hilo.
- `threadReplies: "always"` mantiene las respuestas de la sala en un hilo con raíz en el mensaje que las activa y enruta esa conversación por la sesión acotada al hilo correspondiente desde el primer mensaje que la activa.
- `dm.threadReplies` sobrescribe la configuración de nivel superior solo para DMs. Por ejemplo, puedes mantener aislados los hilos de sala mientras mantienes planos los DMs.
- Los mensajes entrantes con hilo incluyen el mensaje raíz del hilo como contexto adicional para el agente.
- Los envíos mediante herramientas de mensajes heredan automáticamente el hilo actual de Matrix cuando el destino es la misma sala, o el mismo destino de usuario de DM, salvo que se proporcione un `threadId` explícito.
- La reutilización del mismo destino de usuario de DM y la misma sesión solo se activa cuando los metadatos de la sesión actual demuestran el mismo par de DM en la misma cuenta de Matrix; en caso contrario, OpenClaw vuelve al enrutamiento normal acotado al usuario.
- Cuando OpenClaw detecta que una sala de DM de Matrix entra en conflicto con otra sala de DM en la misma sesión compartida de DM de Matrix, publica un `m.notice` único en esa sala con la vía de escape `/focus` cuando los enlaces de hilos están habilitados y la pista `dm.sessionScope`.
- Se admiten enlaces de hilos en tiempo de ejecución para Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y `/acp spawn` enlazado a hilo funcionan en salas y DMs de Matrix.
- `/focus` en una sala o DM de Matrix de nivel superior crea un nuevo hilo de Matrix y lo enlaza a la sesión de destino cuando `threadBindings.spawnSubagentSessions=true`.
- Ejecutar `/focus` o `/acp spawn --thread here` dentro de un hilo de Matrix existente enlaza ese hilo actual en su lugar.

## Enlaces de conversación ACP

Las salas, DMs y hilos existentes de Matrix pueden convertirse en espacios de trabajo ACP duraderos sin cambiar la superficie de chat.

Flujo rápido del operador:

- Ejecuta `/acp spawn codex --bind here` dentro del DM, la sala o el hilo existente de Matrix que quieras seguir usando.
- En un DM o sala de Matrix de nivel superior, el DM o la sala actual siguen siendo la superficie de chat y los mensajes futuros se enrutan a la sesión ACP creada.
- Dentro de un hilo existente de Matrix, `--bind here` enlaza ese hilo actual en su lugar.
- `/new` y `/reset` restablecen la misma sesión ACP enlazada en su lugar.
- `/acp close` cierra la sesión ACP y elimina el enlace.

Notas:

- `--bind here` no crea un hilo hijo de Matrix.
- `threadBindings.spawnAcpSessions` solo es necesario para `/acp spawn --thread auto|here`, donde OpenClaw necesita crear o enlazar un hilo hijo de Matrix.

### Configuración de enlace de hilos

Matrix hereda los valores predeterminados globales de `session.threadBindings`, y también admite sobrescrituras por canal:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Las marcas de creación enlazada a hilo de Matrix son opcionales:

- Establece `threadBindings.spawnSubagentSessions: true` para permitir que `/focus` de nivel superior cree y enlace nuevos hilos de Matrix.
- Establece `threadBindings.spawnAcpSessions: true` para permitir que `/acp spawn --thread auto|here` enlace sesiones ACP a hilos de Matrix.

## Reacciones

Matrix admite acciones salientes de reacciones, notificaciones entrantes de reacciones y reacciones entrantes de acuse.

- Las herramientas salientes de reacción están controladas por `channels["matrix"].actions.reactions`.
- `react` añade una reacción a un evento específico de Matrix.
- `reactions` muestra el resumen actual de reacciones para un evento específico de Matrix.
- `emoji=""` elimina las reacciones propias de la cuenta del bot en ese evento.
- `remove: true` elimina solo la reacción del emoji especificado de la cuenta del bot.

El ámbito de las reacciones de acuse se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- alternativa del emoji de identidad del agente

El alcance de la reacción de acuse se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

El modo de notificación de reacciones se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- predeterminado: `own`

Comportamiento:

- `reactionNotifications: "own"` reenvía eventos añadidos de `m.reaction` cuando apuntan a mensajes de Matrix escritos por el bot.
- `reactionNotifications: "off"` desactiva los eventos del sistema de reacciones.
- La eliminación de reacciones no se sintetiza en eventos del sistema porque Matrix los muestra como redacciones, no como eliminaciones independientes de `m.reaction`.

## Contexto del historial

- `channels.matrix.historyLimit` controla cuántos mensajes recientes de la sala se incluyen como `InboundHistory` cuando un mensaje de una sala de Matrix activa al agente. Recurre a `messages.groupChat.historyLimit`; si ambos no están configurados, el valor efectivo predeterminado es `0`. Establece `0` para desactivar.
- El historial de salas de Matrix es solo de sala. Los DMs siguen usando el historial normal de sesión.
- El historial de salas de Matrix es solo pendiente: OpenClaw almacena en búfer los mensajes de sala que aún no han activado una respuesta, y luego toma una instantánea de esa ventana cuando llega una mención u otro activador.
- El mensaje activador actual no se incluye en `InboundHistory`; permanece en el cuerpo principal entrante de ese turno.
- Los reintentos del mismo evento de Matrix reutilizan la instantánea original del historial en lugar de avanzar con mensajes más nuevos de la sala.

## Visibilidad del contexto

Matrix admite el control compartido `contextVisibility` para contexto suplementario de la sala, como texto de respuesta recuperado, raíces de hilo e historial pendiente.

- `contextVisibility: "all"` es el valor predeterminado. El contexto suplementario se mantiene tal como se recibió.
- `contextVisibility: "allowlist"` filtra el contexto suplementario a remitentes permitidos por las comprobaciones activas de lista de permitidos de sala o usuario.
- `contextVisibility: "allowlist_quote"` se comporta como `allowlist`, pero sigue conservando una respuesta citada explícita.

Esta configuración afecta a la visibilidad del contexto suplementario, no a si el mensaje entrante en sí puede activar una respuesta.
La autorización del activador sigue viniendo de `groupPolicy`, `groups`, `groupAllowFrom` y la configuración de la política de DM.

## Política de DM y sala

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Consulta [Groups](/es/channels/groups) para conocer el comportamiento de las menciones obligatorias y la lista de permitidos.

Ejemplo de emparejamiento para DMs de Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Si un usuario de Matrix no aprobado sigue enviándote mensajes antes de la aprobación, OpenClaw reutiliza el mismo código pendiente de emparejamiento y puede enviar de nuevo una respuesta recordatoria tras un breve tiempo de espera en lugar de generar un código nuevo.

Consulta [Pairing](/es/channels/pairing) para ver el flujo compartido de emparejamiento de DM y la distribución del almacenamiento.

## Reparación directa de sala

Si el estado de los mensajes directos se desincroniza, OpenClaw puede acabar con asignaciones `m.direct` obsoletas que apuntan a salas antiguas individuales en lugar del DM activo. Inspecciona la asignación actual para un par con:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Repárala con:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

El flujo de reparación:

- prefiere un DM estricto 1:1 que ya esté asignado en `m.direct`
- recurre a cualquier DM estricto 1:1 actualmente unido con ese usuario
- crea una nueva sala directa y reescribe `m.direct` si no existe ningún DM saludable

El flujo de reparación no elimina automáticamente las salas antiguas. Solo elige el DM saludable y actualiza la asignación para que los nuevos envíos de Matrix, los avisos de verificación y otros flujos de mensajes directos vuelvan a apuntar a la sala correcta.

## Aprobaciones de exec

Matrix puede actuar como cliente nativo de aprobación para una cuenta de Matrix. Los controles nativos
de enrutamiento de DM/canal siguen estando bajo la configuración de aprobación de exec:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (opcional; usa `channels.matrix.dm.allowFrom` como alternativa)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Los aprobadores deben ser ID de usuario de Matrix como `@owner:example.org`. Matrix habilita automáticamente las aprobaciones nativas cuando `enabled` no está configurado o es `"auto"` y se puede resolver al menos un aprobador. Las aprobaciones de exec usan primero `execApprovals.approvers` y pueden recurrir a `channels.matrix.dm.allowFrom`. Las aprobaciones de plugins autorizan mediante `channels.matrix.dm.allowFrom`. Establece `enabled: false` para desactivar explícitamente Matrix como cliente nativo de aprobación. En caso contrario, las solicitudes de aprobación recurren a otras rutas de aprobación configuradas o a la política alternativa de aprobación.

El enrutamiento nativo de Matrix admite ambos tipos de aprobación:

- `channels.matrix.execApprovals.*` controla el modo nativo de distribución en DM/canal para los avisos de aprobación de Matrix.
- Las aprobaciones de exec usan el conjunto de aprobadores de exec de `execApprovals.approvers` o `channels.matrix.dm.allowFrom`.
- Las aprobaciones de plugins usan la lista de permitidos de DM de Matrix de `channels.matrix.dm.allowFrom`.
- Los atajos por reacción de Matrix y las actualizaciones de mensajes se aplican tanto a las aprobaciones de exec como a las de plugins.

Reglas de entrega:

- `target: "dm"` envía avisos de aprobación a los DMs de los aprobadores
- `target: "channel"` envía el aviso de vuelta a la sala o DM original de Matrix
- `target: "both"` envía a los DMs de los aprobadores y a la sala o DM original de Matrix

Los avisos de aprobación de Matrix siembran atajos de reacción en el mensaje principal de aprobación:

- `✅` = permitir una vez
- `❌` = denegar
- `♾️` = permitir siempre cuando esa decisión esté permitida por la política efectiva de exec

Los aprobadores pueden reaccionar sobre ese mensaje o usar los comandos slash alternativos: `/approve <id> allow-once`, `/approve <id> allow-always`, o `/approve <id> deny`.

Solo los aprobadores resueltos pueden aprobar o denegar. Para aprobaciones de exec, la entrega por canal incluye el texto del comando, así que activa `channel` o `both` solo en salas de confianza.

Sobrescritura por cuenta:

- `channels.matrix.accounts.<account>.execApprovals`

Documentación relacionada: [Aprobaciones de exec](/es/tools/exec-approvals)

## Varias cuentas

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Los valores de nivel superior en `channels.matrix` actúan como valores predeterminados para las cuentas con nombre, a menos que una cuenta los sobrescriba.
Puedes acotar entradas de sala heredadas a una cuenta de Matrix con `groups.<room>.account`.
Las entradas sin `account` siguen compartidas entre todas las cuentas de Matrix, y las entradas con `account: "default"` siguen funcionando cuando la cuenta predeterminada se configura directamente en `channels.matrix.*` de nivel superior.
Los valores predeterminados parciales compartidos de autenticación no crean por sí mismos una cuenta predeterminada implícita aparte. OpenClaw solo sintetiza la cuenta `default` de nivel superior cuando ese valor predeterminado tiene autenticación reciente (`homeserver` más `accessToken`, o `homeserver` más `userId` y `password`); las cuentas con nombre pueden seguir siendo detectables a partir de `homeserver` más `userId` cuando las credenciales en caché satisfacen la autenticación más adelante.
Si Matrix ya tiene exactamente una cuenta con nombre, o `defaultAccount` apunta a una clave existente de cuenta con nombre, la reparación o promoción de configuración de una sola cuenta a varias cuentas conserva esa cuenta en lugar de crear una entrada nueva `accounts.default`. Solo las claves de autenticación o bootstrap de Matrix se mueven a esa cuenta promovida; las claves compartidas de política de entrega permanecen en el nivel superior.
Establece `defaultAccount` cuando quieras que OpenClaw prefiera una cuenta de Matrix con nombre para enrutamiento implícito, sondeos y operaciones de CLI.
Si configuras varias cuentas con nombre, establece `defaultAccount` o pasa `--account <id>` para los comandos CLI que dependen de la selección implícita de cuenta.
Pasa `--account <id>` a `openclaw matrix verify ...` y `openclaw matrix devices ...` cuando quieras sobrescribir esa selección implícita en un comando.

Consulta [Referencia de configuración](/es/gateway/configuration-reference#multi-account-all-channels) para ver el patrón compartido de varias cuentas.

## Homeservers privados/LAN

De forma predeterminada, OpenClaw bloquea homeservers privados o internos de Matrix para protección SSRF, a menos que
optes explícitamente por permitirlos por cuenta.

Si tu homeserver se ejecuta en localhost, una IP LAN/Tailscale o un nombre de host interno, habilita
`network.dangerouslyAllowPrivateNetwork` para esa cuenta de Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Ejemplo de configuración desde CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Esta aceptación explícita solo permite destinos privados o internos de confianza. Los homeservers públicos en texto claro como
`http://matrix.example.org:8008` siguen bloqueados. Prefiere `https://` siempre que sea posible.

## Uso de proxy para el tráfico de Matrix

Si tu despliegue de Matrix necesita un proxy HTTP(S) saliente explícito, establece `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Las cuentas con nombre pueden sobrescribir el valor predeterminado de nivel superior con `channels.matrix.accounts.<id>.proxy`.
OpenClaw usa la misma configuración de proxy para el tráfico de Matrix en tiempo de ejecución y para los sondeos de estado de la cuenta.

## Resolución de destinos

Matrix acepta estos formatos de destino en cualquier lugar donde OpenClaw te pida un objetivo de sala o usuario:

- Usuarios: `@user:server`, `user:@user:server`, o `matrix:user:@user:server`
- Salas: `!room:server`, `room:!room:server`, o `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server`, o `matrix:channel:#alias:server`

La búsqueda en directorio en vivo usa la cuenta de Matrix que ha iniciado sesión:

- Las búsquedas de usuario consultan el directorio de usuarios de Matrix en ese homeserver.
- Las búsquedas de sala aceptan directamente ID y alias de sala explícitos, y después recurren a buscar nombres de salas unidas para esa cuenta.
- La búsqueda de nombres de salas unidas es de mejor esfuerzo. Si un nombre de sala no puede resolverse a un ID o alias, se ignora en la resolución de la lista de permitidos en tiempo de ejecución.

## Referencia de configuración

- `enabled`: habilita o deshabilita el canal.
- `name`: etiqueta opcional para la cuenta.
- `defaultAccount`: ID de cuenta preferido cuando se configuran varias cuentas de Matrix.
- `homeserver`: URL del homeserver, por ejemplo `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: permite que esta cuenta de Matrix se conecte a homeservers privados o internos. Habilítalo cuando el homeserver se resuelva a `localhost`, una IP LAN/Tailscale o un host interno como `matrix-synapse`.
- `proxy`: URL opcional de proxy HTTP(S) para el tráfico de Matrix. Las cuentas con nombre pueden sobrescribir el valor predeterminado de nivel superior con su propio `proxy`.
- `userId`: ID completo de usuario de Matrix, por ejemplo `@bot:example.org`.
- `accessToken`: token de acceso para autenticación basada en token. Se admiten valores en texto plano y valores SecretRef para `channels.matrix.accessToken` y `channels.matrix.accounts.<id>.accessToken` en proveedores env/file/exec. Consulta [Gestión de secretos](/es/gateway/secrets).
- `password`: contraseña para inicio de sesión basado en contraseña. Se admiten valores en texto plano y valores SecretRef.
- `deviceId`: ID explícito del dispositivo de Matrix.
- `deviceName`: nombre para mostrar del dispositivo en inicio de sesión con contraseña.
- `avatarUrl`: URL del avatar propio almacenada para sincronización de perfil y actualizaciones de `profile set`.
- `initialSyncLimit`: número máximo de eventos obtenidos durante la sincronización de inicio.
- `encryption`: habilita E2EE.
- `allowlistOnly`: cuando es `true`, actualiza la política de sala `open` a `allowlist` y obliga a que todas las políticas DM activas excepto `disabled` (incluidas `pairing` y `open`) pasen a `allowlist`. No afecta a las políticas `disabled`.
- `allowBots`: permite mensajes de otras cuentas de Matrix de OpenClaw configuradas (`true` o `"mentions"`).
- `groupPolicy`: `open`, `allowlist` o `disabled`.
- `contextVisibility`: modo de visibilidad de contexto suplementario de sala (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: lista de permitidos de ID de usuario para tráfico de sala. Las entradas deben ser ID completos de usuario de Matrix; los nombres no resueltos se ignoran en tiempo de ejecución.
- `historyLimit`: número máximo de mensajes de sala que se incluirán como contexto del historial de grupo. Recurre a `messages.groupChat.historyLimit`; si ambos no están configurados, el valor efectivo predeterminado es `0`. Establece `0` para desactivar.
- `replyToMode`: `off`, `first`, `all`, o `batched`.
- `markdown`: configuración opcional de renderizado Markdown para texto saliente de Matrix.
- `streaming`: `off` (predeterminado), `"partial"`, `"quiet"`, `true`, o `false`. `"partial"` y `true` habilitan actualizaciones de borrador con vista previa primero usando mensajes de texto normales de Matrix. `"quiet"` usa avisos de vista previa sin notificación para configuraciones autoalojadas con reglas push. `false` equivale a `"off"`.
- `blockStreaming`: `true` habilita mensajes de progreso separados para bloques completados del asistente mientras la vista previa de borrador en streaming está activa.
- `threadReplies`: `off`, `inbound`, o `always`.
- `threadBindings`: sobrescrituras por canal para enrutamiento y ciclo de vida de sesiones enlazadas a hilos.
- `startupVerification`: modo automático de solicitud de autoverificación al inicio (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: tiempo de espera antes de reintentar solicitudes automáticas de verificación al inicio.
- `textChunkLimit`: tamaño en caracteres de los fragmentos de mensajes salientes (se aplica cuando `chunkMode` es `length`).
- `chunkMode`: `length` divide mensajes por cantidad de caracteres; `newline` divide en límites de línea.
- `responsePrefix`: cadena opcional añadida al principio de todas las respuestas salientes de este canal.
- `ackReaction`: sobrescritura opcional de reacción de acuse para este canal o cuenta.
- `ackReactionScope`: sobrescritura opcional del alcance de la reacción de acuse (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: modo de notificación entrante de reacciones (`own`, `off`).
- `mediaMaxMb`: límite de tamaño de medios en MB para envíos salientes y procesamiento de medios entrantes.
- `autoJoin`: política de unión automática a invitaciones (`always`, `allowlist`, `off`). Predeterminado: `off`. Se aplica a todas las invitaciones de Matrix, incluidas las invitaciones de estilo DM.
- `autoJoinAllowlist`: salas o alias permitidos cuando `autoJoin` es `allowlist`. Las entradas de alias se resuelven a ID de sala durante el manejo de la invitación; OpenClaw no confía en el estado del alias afirmado por la sala invitada.
- `dm`: bloque de política de DM (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: controla el acceso a DM después de que OpenClaw se haya unido a la sala y la haya clasificado como DM. No cambia si una invitación se une automáticamente o no.
- `dm.allowFrom`: las entradas deben ser ID completos de usuario de Matrix a menos que ya las hayas resuelto mediante búsqueda de directorio en vivo.
- `dm.sessionScope`: `per-user` (predeterminado) o `per-room`. Usa `per-room` cuando quieras que cada sala de DM de Matrix mantenga contexto separado aunque el par sea el mismo.
- `dm.threadReplies`: sobrescritura de política de hilos solo para DMs (`off`, `inbound`, `always`). Sobrescribe la configuración de nivel superior `threadReplies` tanto para la colocación de respuestas como para el aislamiento de sesión en DMs.
- `execApprovals`: entrega nativa de aprobaciones de exec de Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID de usuario de Matrix autorizados para aprobar solicitudes de exec. Opcional cuando `dm.allowFrom` ya identifica a los aprobadores.
- `execApprovals.target`: `dm | channel | both` (predeterminado: `dm`).
- `accounts`: sobrescrituras con nombre por cuenta. Los valores de nivel superior en `channels.matrix` actúan como predeterminados para estas entradas.
- `groups`: mapa de políticas por sala. Prefiere ID o alias de sala; los nombres de sala no resueltos se ignoran en tiempo de ejecución. La identidad de sesión o grupo usa el ID de sala estable después de la resolución.
- `groups.<room>.account`: restringe una entrada de sala heredada a una cuenta de Matrix específica en configuraciones de varias cuentas.
- `groups.<room>.allowBots`: sobrescritura a nivel de sala para remitentes bot configurados (`true` o `"mentions"`).
- `groups.<room>.users`: lista de permitidos por remitente a nivel de sala.
- `groups.<room>.tools`: sobrescrituras por sala para permitir o denegar herramientas.
- `groups.<room>.autoReply`: sobrescritura a nivel de sala para el requisito de menciones. `true` desactiva los requisitos de mención para esa sala; `false` los vuelve a forzar.
- `groups.<room>.skills`: filtro opcional de Skills a nivel de sala.
- `groups.<room>.systemPrompt`: fragmento opcional de system prompt a nivel de sala.
- `rooms`: alias heredado para `groups`.
- `actions`: control por acción de herramientas (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Relacionado

- [Resumen de canales](/es/channels) — todos los canales compatibles
- [Pairing](/es/channels/pairing) — autenticación DM y flujo de emparejamiento
- [Groups](/es/channels/groups) — comportamiento de chat grupal y requisito de menciones
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesión para mensajes
- [Security](/es/gateway/security) — modelo de acceso y refuerzo
