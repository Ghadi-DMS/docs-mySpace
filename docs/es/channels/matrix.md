---
read_when:
    - Configuración de Matrix en OpenClaw
    - Configuración de E2EE y verificación de Matrix
summary: Estado de compatibilidad de Matrix, configuración y ejemplos de configuración
title: Matrix
x-i18n:
    generated_at: "2026-04-08T02:16:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec926df79a41fa296d63f0ec7219d0f32e075628d76df9ea490e93e4c5030f83
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix es el plugin de canal empaquetado de Matrix para OpenClaw.
Usa el `matrix-js-sdk` oficial y admite MD, salas, hilos, multimedia, reacciones, encuestas, ubicación y E2EE.

## Plugin empaquetado

Matrix se incluye como plugin empaquetado en las versiones actuales de OpenClaw, por lo que las
compilaciones empaquetadas normales no necesitan una instalación independiente.

Si estás en una compilación antigua o en una instalación personalizada que excluye Matrix, instálalo
manualmente:

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
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea una cuenta de Matrix en tu homeserver.
3. Configura `channels.matrix` con una de estas opciones:
   - `homeserver` + `accessToken`, o
   - `homeserver` + `userId` + `password`.
4. Reinicia el gateway.
5. Inicia una MD con el bot o invítalo a una sala.
   - Las invitaciones nuevas de Matrix solo funcionan cuando `channels.matrix.autoJoin` las permite.

Rutas de configuración interactiva:

```bash
openclaw channels add
openclaw configure --section channels
```

Lo que realmente pregunta el asistente de Matrix:

- URL del homeserver
- método de autenticación: token de acceso o contraseña
- ID de usuario solo cuando eliges autenticación por contraseña
- nombre del dispositivo opcional
- si se debe habilitar E2EE
- si se debe configurar ahora el acceso a salas de Matrix
- si se debe configurar ahora la unión automática a invitaciones de Matrix
- cuando la unión automática a invitaciones está habilitada, si debe ser `allowlist`, `always` u `off`

Comportamiento del asistente que importa:

- Si ya existen variables de entorno de autenticación de Matrix para la cuenta seleccionada, y esa cuenta todavía no tiene autenticación guardada en la configuración, el asistente ofrece un atajo de entorno para que la configuración pueda mantener la autenticación en variables de entorno en lugar de copiar secretos a la configuración.
- Cuando añades otra cuenta de Matrix de forma interactiva, el nombre de cuenta introducido se normaliza en el ID de cuenta usado en la configuración y en las variables de entorno. Por ejemplo, `Ops Bot` se convierte en `ops-bot`.
- Los prompts de allowlist para MD aceptan inmediatamente valores completos `@user:server`. Los nombres para mostrar solo funcionan cuando la búsqueda en vivo del directorio encuentra una coincidencia exacta; de lo contrario, el asistente te pide que lo intentes de nuevo con un ID completo de Matrix.
- Los prompts de allowlist para salas aceptan directamente IDs y alias de sala. También pueden resolver en vivo nombres de salas unidas, pero los nombres no resueltos solo se conservan tal como se escribieron durante la configuración y luego se ignoran en la resolución de allowlist en tiempo de ejecución. Prefiere `!room:server` o `#alias:server`.
- El asistente ahora muestra una advertencia explícita antes del paso de unión automática a invitaciones porque `channels.matrix.autoJoin` tiene como valor predeterminado `off`; los agentes no se unirán a salas invitadas ni a invitaciones nuevas de estilo MD a menos que lo configures.
- En el modo allowlist de unión automática a invitaciones, usa solo destinos de invitación estables: `!roomId:server`, `#alias:server` o `*`. Los nombres simples de salas se rechazan.
- La identidad de sala/sesión en tiempo de ejecución usa el ID estable de sala de Matrix. Los alias declarados por la sala solo se usan como entradas de búsqueda, no como clave de sesión a largo plazo ni como identidad estable de grupo.
- Para resolver nombres de salas antes de guardarlos, usa `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` tiene como valor predeterminado `off`.

Si lo dejas sin configurar, el bot no se unirá a salas invitadas ni a invitaciones nuevas de estilo MD, por lo que no aparecerá en grupos nuevos ni en MD invitadas a menos que primero te unas manualmente.

Establece `autoJoin: "allowlist"` junto con `autoJoinAllowlist` para restringir qué invitaciones acepta, o establece `autoJoin: "always"` si quieres que se una a todas las invitaciones.

En el modo `allowlist`, `autoJoinAllowlist` solo acepta `!roomId:server`, `#alias:server` o `*`.
</Warning>

Ejemplo de allowlist:

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
Cuando existen credenciales en caché ahí, OpenClaw considera Matrix como configurado para la configuración, doctor y el descubrimiento de estado del canal, incluso si la autenticación actual no está definida directamente en la configuración.

Equivalentes en variables de entorno (se usan cuando la clave de configuración no está definida):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Para cuentas no predeterminadas, usa variables de entorno con alcance de cuenta:

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

Matrix escapa la puntuación en los ID de cuenta para mantener las variables de entorno con alcance de cuenta libres de colisiones.
Por ejemplo, `-` se convierte en `_X2D_`, por lo que `ops-prod` se asigna a `MATRIX_OPS_X2D_PROD_*`.

El asistente interactivo solo ofrece el atajo de variables de entorno cuando esas variables de autenticación ya están presentes y la cuenta seleccionada aún no tiene autenticación de Matrix guardada en la configuración.

## Ejemplo de configuración

Esta es una configuración base práctica con emparejamiento en MD, allowlist de salas y E2EE habilitado:

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

`autoJoin` se aplica a las invitaciones de Matrix en general, no solo a las invitaciones de sala/grupo.
Eso incluye las invitaciones nuevas de estilo MD. En el momento de la invitación, OpenClaw no sabe de forma fiable si la
sala invitada acabará tratándose como una MD o como un grupo, por lo que todas las invitaciones pasan primero por la misma
decisión de `autoJoin`. `dm.policy` sigue aplicándose después de que el bot se haya unido y la sala se
clasifique como una MD, de modo que `autoJoin` controla el comportamiento de unión mientras que `dm.policy` controla el comportamiento de respuesta/acceso.

## Vistas previas de streaming

El streaming de respuestas de Matrix es opcional.

Establece `channels.matrix.streaming` en `"partial"` cuando quieras que OpenClaw envíe una sola vista previa en vivo
de la respuesta, edite esa vista previa en su lugar mientras el modelo genera texto y luego la
finalice cuando la respuesta termine:

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
- `streaming: "partial"` crea un mensaje de vista previa editable para el bloque actual del asistente usando mensajes de texto normales de Matrix. Esto conserva el comportamiento heredado de notificación basado primero en la vista previa de Matrix, por lo que los clientes estándar pueden notificar con el primer texto de vista previa en streaming en lugar del bloque terminado.
- `streaming: "quiet"` crea un aviso de vista previa silencioso editable para el bloque actual del asistente. Usa esto solo cuando también configures reglas push del destinatario para las ediciones finalizadas de la vista previa.
- `blockStreaming: true` habilita mensajes de progreso de Matrix independientes. Con la vista previa en streaming habilitada, Matrix mantiene el borrador en vivo del bloque actual y conserva los bloques completados como mensajes independientes.
- Cuando la vista previa en streaming está activada y `blockStreaming` está desactivado, Matrix edita el borrador en vivo en su lugar y finaliza ese mismo evento cuando el bloque o el turno termina.
- Si la vista previa ya no cabe en un solo evento de Matrix, OpenClaw detiene la vista previa en streaming y vuelve a la entrega final normal.
- Las respuestas multimedia siguen enviando archivos adjuntos con normalidad. Si una vista previa obsoleta ya no puede reutilizarse de forma segura, OpenClaw la redacta antes de enviar la respuesta multimedia final.
- Las ediciones de vista previa tienen un costo adicional de llamadas a la API de Matrix. Deja el streaming desactivado si quieres el comportamiento más conservador respecto a los límites de tasa.

`blockStreaming` no habilita por sí solo las vistas previas de borrador.
Usa `streaming: "partial"` o `streaming: "quiet"` para las ediciones de vista previa; luego añade `blockStreaming: true` solo si también quieres que los bloques completados del asistente permanezcan visibles como mensajes de progreso independientes.

Si necesitas notificaciones estándar de Matrix sin reglas push personalizadas, usa `streaming: "partial"` para el comportamiento de vista previa primero o deja `streaming` desactivado para entrega solo final. Con `streaming: "off"`:

- `blockStreaming: true` envía cada bloque terminado como un mensaje normal de Matrix con notificación.
- `blockStreaming: false` envía solo la respuesta final completada como un mensaje normal de Matrix con notificación.

### Reglas push autoalojadas para vistas previas silenciosas finalizadas

Si ejecutas tu propia infraestructura de Matrix y quieres que las vistas previas silenciosas notifiquen solo cuando un bloque o la
respuesta final haya terminado, establece `streaming: "quiet"` y añade una regla push por usuario para las ediciones finalizadas de la vista previa.

Esto normalmente es una configuración del usuario destinatario, no un cambio de configuración global del homeserver:

Mapa rápido antes de empezar:

- usuario destinatario = la persona que debe recibir la notificación
- usuario bot = la cuenta de Matrix de OpenClaw que envía la respuesta
- usa el token de acceso del usuario destinatario para las llamadas a la API de abajo
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

2. Asegúrate de que la cuenta del destinatario ya recibe notificaciones push normales de Matrix. Las reglas
   para vistas previas silenciosas solo funcionan si ese usuario ya tiene pushers/dispositivos operativos.

3. Obtén el token de acceso del usuario destinatario.
   - Usa el token del usuario que recibe, no el token del bot.
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

Si esto no devuelve pushers/dispositivos activos, corrige primero las notificaciones normales de Matrix antes de añadir la
regla de OpenClaw de abajo.

OpenClaw marca las ediciones finalizadas de vista previa solo de texto con:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Crea una regla push de anulación para cada cuenta destinataria que deba recibir estas notificaciones:

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

- Las reglas push se indexan por `ruleId`. Volver a ejecutar `PUT` sobre el mismo ID de regla actualiza esa única regla.
- Si un usuario receptor debe recibir notificaciones de varias cuentas bot de Matrix de OpenClaw, crea una regla por bot con un ID de regla único para cada coincidencia de remitente.
- Un patrón simple es `openclaw-finalized-preview-<botname>`, como `openclaw-finalized-preview-ops` o `openclaw-finalized-preview-support`.

La regla se evalúa frente al remitente del evento:

- autentícate con el token del usuario receptor
- haz coincidir `sender` con el MXID del bot de OpenClaw

6. Verifica que la regla exista:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Prueba una respuesta en streaming. En modo silencioso, la sala debería mostrar una vista previa de borrador silenciosa y la edición final
   en su lugar debería notificar una vez que el bloque o el turno termine.

Si necesitas eliminar la regla más tarde, borra ese mismo ID de regla con el token del usuario receptor:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Notas:

- Crea la regla con el token de acceso del usuario receptor, no con el del bot.
- Las nuevas reglas `override` definidas por el usuario se insertan antes de las reglas de supresión predeterminadas, por lo que no se necesita ningún parámetro adicional de orden.
- Esto solo afecta a las ediciones de vista previa solo de texto que OpenClaw puede finalizar de forma segura en su lugar. Las alternativas para multimedia y las alternativas por vista previa obsoleta siguen usando la entrega normal de Matrix.
- Si `GET /_matrix/client/v3/pushers` no muestra pushers, el usuario todavía no tiene una entrega push de Matrix operativa para esta cuenta/dispositivo.

#### Synapse

Para Synapse, la configuración anterior normalmente es suficiente por sí sola:

- No se requiere ningún cambio especial en `homeserver.yaml` para las notificaciones de vista previa finalizada de OpenClaw.
- Si tu despliegue de Synapse ya envía notificaciones push normales de Matrix, el token de usuario + la llamada `pushrules` anterior es el paso principal de configuración.
- Si ejecutas Synapse detrás de un proxy inverso o workers, asegúrate de que `/_matrix/client/.../pushrules/` llegue correctamente a Synapse.
- Si ejecutas workers de Synapse, asegúrate de que los pushers estén en buen estado. La entrega push la gestiona el proceso principal o `synapse.app.pusher` / workers de pusher configurados.

#### Tuwunel

Para Tuwunel, usa el mismo flujo de configuración y la llamada a la API `pushrules` mostrada arriba:

- No se requiere ninguna configuración específica de Tuwunel para el marcador de vista previa finalizada en sí.
- Si las notificaciones normales de Matrix ya funcionan para ese usuario, el token de usuario + la llamada `pushrules` anterior es el paso principal de configuración.
- Si parece que las notificaciones desaparecen mientras el usuario está activo en otro dispositivo, comprueba si `suppress_push_when_active` está habilitado. Tuwunel añadió esta opción en Tuwunel 1.4.2 el 12 de septiembre de 2025, y puede suprimir intencionadamente los pushes a otros dispositivos mientras un dispositivo está activo.

## Cifrado y verificación

En salas cifradas (E2EE), los eventos salientes de imagen usan `thumbnail_file` para que las vistas previas de imagen se cifren junto con el archivo adjunto completo. Las salas sin cifrar siguen usando `thumbnail_url` sin formato. No se necesita ninguna configuración: el plugin detecta automáticamente el estado de E2EE.

### Salas de bot a bot

De forma predeterminada, los mensajes de otras cuentas configuradas de Matrix de OpenClaw se ignoran.

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

- `allowBots: true` acepta mensajes de otras cuentas bot configuradas de Matrix en salas y MD permitidas.
- `allowBots: "mentions"` acepta esos mensajes solo cuando mencionan visiblemente a este bot en salas. Las MD siguen estando permitidas.
- `groups.<room>.allowBots` reemplaza la configuración a nivel de cuenta para una sala.
- OpenClaw sigue ignorando los mensajes del mismo ID de usuario de Matrix para evitar bucles de autorrespuesta.
- Matrix no expone aquí un indicador nativo de bot; OpenClaw considera "creado por bot" como "enviado por otra cuenta de Matrix configurada en este gateway de OpenClaw".

Usa allowlists estrictas de salas y requisitos de mención al habilitar tráfico bot a bot en salas compartidas.

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

Inicializar el estado de cross-signing y verificación:

```bash
openclaw matrix verify bootstrap
```

Compatibilidad con varias cuentas: usa `channels.matrix.accounts` con credenciales por cuenta y `name` opcional. Consulta [Referencia de configuración](/es/gateway/configuration-reference#multi-account-all-channels) para el patrón compartido.

Diagnóstico detallado de bootstrap:

```bash
openclaw matrix verify bootstrap --verbose
```

Forzar un reinicio nuevo de la identidad de cross-signing antes de hacer bootstrap:

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

Comprobar el estado del backup de claves de sala:

```bash
openclaw matrix verify backup status
```

Diagnóstico detallado del estado del backup:

```bash
openclaw matrix verify backup status --verbose
```

Restaurar claves de sala desde el backup del servidor:

```bash
openclaw matrix verify backup restore
```

Diagnóstico detallado de restauración:

```bash
openclaw matrix verify backup restore --verbose
```

Eliminar el backup actual del servidor y crear una nueva línea base de backup. Si la clave de
backup almacenada no puede cargarse limpiamente, este reinicio también puede recrear el almacenamiento de secretos para que
los futuros arranques en frío puedan cargar la nueva clave de backup:

```bash
openclaw matrix verify backup reset --yes
```

Todos los comandos `verify` son concisos de forma predeterminada (incluido el registro interno silencioso del SDK) y solo muestran diagnóstico detallado con `--verbose`.
Usa `--json` para obtener una salida completa legible por máquina al automatizar.

En configuraciones con varias cuentas, los comandos de Matrix CLI usan la cuenta predeterminada implícita de Matrix a menos que pases `--account <id>`.
Si configuras varias cuentas con nombre, establece primero `channels.matrix.defaultAccount` o esas operaciones implícitas de CLI se detendrán y te pedirán que elijas una cuenta explícitamente.
Usa `--account` siempre que quieras que las operaciones de verificación o de dispositivo apunten explícitamente a una cuenta con nombre:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Cuando el cifrado está deshabilitado o no disponible para una cuenta con nombre, las advertencias de Matrix y los errores de verificación apuntan a la clave de configuración de esa cuenta, por ejemplo `channels.matrix.accounts.assistant.encryption`.

### Qué significa "verified"

OpenClaw trata este dispositivo de Matrix como verificado solo cuando está verificado por tu propia identidad de cross-signing.
En la práctica, `openclaw matrix verify status --verbose` expone tres señales de confianza:

- `Locally trusted`: este dispositivo es de confianza solo para el cliente actual
- `Cross-signing verified`: el SDK informa que el dispositivo está verificado mediante cross-signing
- `Signed by owner`: el dispositivo está firmado por tu propia clave de self-signing

`Verified by owner` pasa a ser `yes` solo cuando hay verificación por cross-signing o firma del propietario.
La confianza local por sí sola no basta para que OpenClaw trate el dispositivo como totalmente verificado.

### Qué hace bootstrap

`openclaw matrix verify bootstrap` es el comando de reparación y configuración para cuentas cifradas de Matrix.
Hace todo lo siguiente en este orden:

- inicializa el almacenamiento de secretos, reutilizando una clave de recuperación existente cuando sea posible
- inicializa cross-signing y sube las claves públicas de cross-signing que falten
- intenta marcar y firmar con cross-signing el dispositivo actual
- crea un nuevo backup de claves de sala del lado del servidor si todavía no existe uno

Si el homeserver requiere autenticación interactiva para subir claves de cross-signing, OpenClaw intenta la carga primero sin autenticación, luego con `m.login.dummy` y después con `m.login.password` cuando `channels.matrix.password` está configurado.

Usa `--force-reset-cross-signing` solo cuando quieras intencionadamente descartar la identidad actual de cross-signing y crear una nueva.

Si intencionadamente quieres descartar el backup actual de claves de sala y comenzar una nueva
línea base de backup para futuros mensajes, usa `openclaw matrix verify backup reset --yes`.
Haz esto solo si aceptas que el historial cifrado antiguo irrecuperable seguirá
sin estar disponible y que OpenClaw puede recrear el almacenamiento de secretos si el secreto actual del backup
no puede cargarse de forma segura.

### Nueva línea base de backup

Si quieres mantener funcionando los futuros mensajes cifrados y aceptas perder el historial antiguo irrecuperable, ejecuta estos comandos en orden:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Añade `--account <id>` a cada comando cuando quieras apuntar explícitamente a una cuenta de Matrix con nombre.

### Comportamiento al iniciar

Cuando `encryption: true`, Matrix establece por defecto `startupVerification` en `"if-unverified"`.
Al iniciar, si este dispositivo sigue sin verificar, Matrix solicitará la autoverificación en otro cliente de Matrix,
omitirá solicitudes duplicadas mientras ya haya una pendiente y aplicará un enfriamiento local antes de reintentar tras reinicios.
Los intentos fallidos de solicitud se reintentan antes que la creación satisfactoria de solicitudes, de forma predeterminada.
Establece `startupVerification: "off"` para deshabilitar las solicitudes automáticas al iniciar, o ajusta `startupVerificationCooldownHours`
si quieres una ventana de reintento más corta o más larga.

El inicio también realiza automáticamente una pasada conservadora de bootstrap de crypto.
Esa pasada intenta reutilizar primero el almacenamiento de secretos y la identidad de cross-signing actuales, y evita restablecer cross-signing a menos que ejecutes un flujo explícito de reparación bootstrap.

Si el inicio encuentra un estado de bootstrap roto y `channels.matrix.password` está configurado, OpenClaw puede intentar una ruta de reparación más estricta.
Si el dispositivo actual ya está firmado por el propietario, OpenClaw preserva esa identidad en lugar de restablecerla automáticamente.

Actualización desde el plugin público anterior de Matrix:

- OpenClaw reutiliza automáticamente la misma cuenta de Matrix, el token de acceso y la identidad del dispositivo cuando es posible.
- Antes de ejecutar cualquier cambio de migración accionable de Matrix, OpenClaw crea o reutiliza una instantánea de recuperación en `~/Backups/openclaw-migrations/`.
- Si usas varias cuentas de Matrix, establece `channels.matrix.defaultAccount` antes de actualizar desde el diseño plano de almacenamiento antiguo para que OpenClaw sepa qué cuenta debe recibir ese estado heredado compartido.
- Si el plugin anterior almacenó localmente una clave de descifrado del backup de claves de sala de Matrix, el inicio o `openclaw doctor --fix` la importará automáticamente al nuevo flujo de clave de recuperación.
- Si el token de acceso de Matrix cambió después de que se preparó la migración, el inicio ahora examina raíces hermanas de almacenamiento por hash de token en busca de estado heredado pendiente de restauración antes de abandonar la restauración automática del backup.
- Si el token de acceso de Matrix cambia más tarde para la misma cuenta, homeserver y usuario, OpenClaw ahora prefiere reutilizar la raíz de almacenamiento por hash de token existente más completa en lugar de empezar desde un directorio de estado de Matrix vacío.
- En el siguiente inicio del gateway, las claves de sala respaldadas se restauran automáticamente en el nuevo almacén crypto.
- Si el plugin antiguo tenía claves de sala solo locales que nunca se respaldaron, OpenClaw lo advertirá claramente. Esas claves no pueden exportarse automáticamente desde el almacén crypto rust anterior, por lo que parte del historial cifrado antiguo puede seguir sin estar disponible hasta recuperarlo manualmente.
- Consulta [Migración de Matrix](/es/install/migrating-matrix) para ver el flujo completo de actualización, límites, comandos de recuperación y mensajes comunes de migración.

El estado de ejecución cifrado se organiza bajo raíces por cuenta, usuario y hash de token en
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Ese directorio contiene el almacén de sincronización (`bot-storage.json`), el almacén crypto (`crypto/`),
el archivo de clave de recuperación (`recovery-key.json`), la instantánea de IndexedDB (`crypto-idb-snapshot.json`),
los enlaces de hilos (`thread-bindings.json`) y el estado de verificación al inicio (`startup-verification.json`)
cuando esas funciones están en uso.
Cuando el token cambia pero la identidad de la cuenta sigue siendo la misma, OpenClaw reutiliza la mejor
raíz existente para esa tupla cuenta/homeserver/usuario, de modo que el estado de sincronización previo, el estado crypto, los enlaces de hilos
y el estado de verificación al inicio sigan siendo visibles.

### Modelo de almacén crypto de Node

La E2EE de Matrix en este plugin usa la ruta Rust crypto oficial de `matrix-js-sdk` en Node.
Esa ruta espera persistencia respaldada por IndexedDB cuando quieres que el estado crypto sobreviva a los reinicios.

Actualmente OpenClaw lo proporciona en Node mediante:

- uso de `fake-indexeddb` como shim de API IndexedDB que espera el SDK
- restauración del contenido de IndexedDB de Rust crypto desde `crypto-idb-snapshot.json` antes de `initRustCrypto`
- persistencia del contenido actualizado de IndexedDB de vuelta en `crypto-idb-snapshot.json` después de la inicialización y durante la ejecución
- serialización de la restauración y persistencia de la instantánea contra `crypto-idb-snapshot.json` con un bloqueo de archivo consultivo para que la persistencia del gateway en ejecución y el mantenimiento desde CLI no compitan por el mismo archivo de instantánea

Esto es infraestructura de compatibilidad/almacenamiento, no una implementación crypto personalizada.
El archivo de instantánea es un estado sensible en ejecución y se almacena con permisos restrictivos de archivo.
Bajo el modelo de seguridad de OpenClaw, el host del gateway y el directorio de estado local de OpenClaw ya están dentro del límite de confianza del operador, por lo que esto es principalmente una cuestión operativa de durabilidad y no un límite de confianza remoto independiente.

Mejora planificada:

- añadir compatibilidad con SecretRef para material persistente de claves de Matrix, de modo que las claves de recuperación y secretos relacionados para cifrado del almacenamiento puedan obtenerse de proveedores de secretos de OpenClaw en lugar de solo desde archivos locales

## Gestión de perfil

Actualiza el perfil propio de Matrix para la cuenta seleccionada con:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Añade `--account <id>` cuando quieras apuntar explícitamente a una cuenta de Matrix con nombre.

Matrix acepta directamente URL de avatar `mxc://`. Cuando pasas una URL de avatar `http://` o `https://`, OpenClaw primero la sube a Matrix y almacena la URL `mxc://` resuelta de vuelta en `channels.matrix.avatarUrl` (o en la anulación de la cuenta seleccionada).

## Avisos automáticos de verificación

Matrix ahora publica avisos del ciclo de vida de verificación directamente en la sala estricta de MD de verificación como mensajes `m.notice`.
Eso incluye:

- avisos de solicitud de verificación
- avisos de verificación lista (con orientación explícita de "Verificar por emoji")
- avisos de inicio y finalización de verificación
- detalles SAS (emoji y decimal) cuando están disponibles

Las solicitudes de verificación entrantes de otro cliente de Matrix se rastrean y OpenClaw las acepta automáticamente.
Para flujos de autoverificación, OpenClaw también inicia automáticamente el flujo SAS cuando la verificación por emoji está disponible y confirma su propio lado.
Para solicitudes de verificación de otro usuario/dispositivo de Matrix, OpenClaw acepta automáticamente la solicitud y luego espera a que el flujo SAS continúe normalmente.
Aun así, necesitas comparar el SAS de emoji o decimal en tu cliente de Matrix y confirmar allí "Coinciden" para completar la verificación.

OpenClaw no acepta automáticamente a ciegas flujos duplicados iniciados por sí mismo. Al iniciar se omite la creación de una nueva solicitud cuando ya hay una solicitud de autoverificación pendiente.

Los avisos de verificación de protocolo/sistema no se reenvían al canal de chat del agente, por lo que no producen `NO_REPLY`.

### Higiene de dispositivos

Los dispositivos antiguos de Matrix gestionados por OpenClaw pueden acumularse en la cuenta y hacer que la confianza en salas cifradas sea más difícil de razonar.
Haz una lista con:

```bash
openclaw matrix devices list
```

Elimina dispositivos obsoletos gestionados por OpenClaw con:

```bash
openclaw matrix devices prune-stale
```

### Reparación directa de sala

Si el estado de los mensajes directos se desincroniza, OpenClaw puede acabar con asignaciones `m.direct` obsoletas que apuntan a salas individuales antiguas en lugar de a la MD activa. Inspecciona la asignación actual para un par con:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Repárala con:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

La reparación mantiene la lógica específica de Matrix dentro del plugin:

- prefiere una MD estricta 1:1 que ya esté asignada en `m.direct`
- en caso contrario, recurre a cualquier MD estricta 1:1 actualmente unida con ese usuario
- si no existe una MD en buen estado, crea una nueva sala directa y reescribe `m.direct` para que apunte a ella

El flujo de reparación no elimina automáticamente las salas antiguas. Solo selecciona la MD en buen estado y actualiza la asignación para que los nuevos envíos de Matrix, avisos de verificación y otros flujos de mensajes directos vuelvan a apuntar a la sala correcta.

## Hilos

Matrix admite hilos nativos de Matrix tanto para respuestas automáticas como para envíos de herramientas de mensajes.

- `dm.sessionScope: "per-user"` (predeterminado) mantiene el enrutamiento de MD de Matrix con alcance por remitente, por lo que varias salas de MD pueden compartir una sesión cuando se resuelven al mismo par.
- `dm.sessionScope: "per-room"` aísla cada sala de MD de Matrix en su propia clave de sesión mientras sigue usando las comprobaciones normales de autenticación y allowlist de MD.
- Los enlaces explícitos de conversación de Matrix siguen teniendo prioridad sobre `dm.sessionScope`, por lo que las salas e hilos vinculados conservan su sesión de destino elegida.
- `threadReplies: "off"` mantiene las respuestas en el nivel superior y mantiene los mensajes entrantes en hilo en la sesión principal.
- `threadReplies: "inbound"` responde dentro de un hilo solo cuando el mensaje entrante ya estaba en ese hilo.
- `threadReplies: "always"` mantiene las respuestas de sala en un hilo enraizado en el mensaje que las activó y enruta esa conversación a través de la sesión con alcance de hilo coincidente desde el primer mensaje desencadenante.
- `dm.threadReplies` reemplaza la configuración de nivel superior solo para MD. Por ejemplo, puedes mantener aislados los hilos de sala mientras mantienes planas las MD.
- Los mensajes entrantes en hilos incluyen el mensaje raíz del hilo como contexto adicional para el agente.
- Los envíos de herramientas de mensajes ahora heredan automáticamente el hilo actual de Matrix cuando el destino es la misma sala, o el mismo destino de usuario de MD, a menos que se proporcione un `threadId` explícito.
- La reutilización del mismo objetivo de usuario de MD de la misma sesión solo se activa cuando los metadatos de la sesión actual prueban el mismo par de MD en la misma cuenta de Matrix; de lo contrario, OpenClaw vuelve al enrutamiento normal con alcance por usuario.
- Cuando OpenClaw detecta que una sala de MD de Matrix colisiona con otra sala de MD en la misma sesión compartida de MD de Matrix, publica un `m.notice` único en esa sala con la vía de escape `/focus` cuando los enlaces de hilos están habilitados y la sugerencia `dm.sessionScope`.
- Los enlaces de hilos en tiempo de ejecución son compatibles con Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y `/acp spawn` vinculado a hilo ahora funcionan en salas y MD de Matrix.
- `/focus` de sala/MD de Matrix en nivel superior crea un nuevo hilo de Matrix y lo vincula a la sesión de destino cuando `threadBindings.spawnSubagentSessions=true`.
- Ejecutar `/focus` o `/acp spawn --thread here` dentro de un hilo existente de Matrix vincula ese hilo actual en su lugar.

## Enlaces de conversación ACP

Las salas, MD e hilos existentes de Matrix pueden convertirse en espacios de trabajo ACP duraderos sin cambiar la superficie de chat.

Flujo rápido del operador:

- Ejecuta `/acp spawn codex --bind here` dentro de la MD, sala o hilo existente de Matrix que quieras seguir usando.
- En una MD o sala de Matrix de nivel superior, la MD/sala actual sigue siendo la superficie de chat y los mensajes futuros se enrutan a la sesión ACP generada.
- Dentro de un hilo existente de Matrix, `--bind here` vincula ese hilo actual en su lugar.
- `/new` y `/reset` restablecen la misma sesión ACP vinculada en su lugar.
- `/acp close` cierra la sesión ACP y elimina el vínculo.

Notas:

- `--bind here` no crea un hilo hijo de Matrix.
- `threadBindings.spawnAcpSessions` solo es necesario para `/acp spawn --thread auto|here`, donde OpenClaw necesita crear o vincular un hilo hijo de Matrix.

### Configuración de enlaces de hilos

Matrix hereda los valores predeterminados globales de `session.threadBindings` y también admite anulaciones por canal:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Las marcas de generación vinculadas a hilos de Matrix son opcionales:

- Establece `threadBindings.spawnSubagentSessions: true` para permitir que `/focus` de nivel superior cree y vincule nuevos hilos de Matrix.
- Establece `threadBindings.spawnAcpSessions: true` para permitir que `/acp spawn --thread auto|here` vincule sesiones ACP a hilos de Matrix.

## Reacciones

Matrix admite acciones de reacción salientes, notificaciones de reacción entrantes y reacciones de confirmación entrantes.

- Las herramientas de reacción saliente están controladas por `channels["matrix"].actions.reactions`.
- `react` añade una reacción a un evento específico de Matrix.
- `reactions` enumera el resumen actual de reacciones para un evento específico de Matrix.
- `emoji=""` elimina las propias reacciones de la cuenta del bot en ese evento.
- `remove: true` elimina solo la reacción del emoji especificado de la cuenta del bot.

El alcance de las reacciones de confirmación se resuelve en este orden estándar de OpenClaw:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- emoji de respaldo de la identidad del agente

El alcance de `ackReaction` se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

El modo de notificación de reacciones se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- predeterminado: `own`

Comportamiento actual:

- `reactionNotifications: "own"` reenvía eventos `m.reaction` añadidos cuando apuntan a mensajes de Matrix creados por el bot.
- `reactionNotifications: "off"` deshabilita los eventos de sistema de reacciones.
- Las eliminaciones de reacciones todavía no se sintetizan en eventos de sistema porque Matrix las expone como redacciones, no como eliminaciones independientes de `m.reaction`.

## Contexto del historial

- `channels.matrix.historyLimit` controla cuántos mensajes recientes de la sala se incluyen como `InboundHistory` cuando un mensaje de sala de Matrix activa al agente.
- Recurre a `messages.groupChat.historyLimit`. Si ambos no están configurados, el valor predeterminado efectivo es `0`, por lo que los mensajes de sala controlados por mención no se almacenan en búfer. Establece `0` para deshabilitarlo.
- El historial de salas de Matrix es solo de sala. Las MD siguen usando el historial normal de sesión.
- El historial de salas de Matrix es solo pendiente: OpenClaw almacena en búfer los mensajes de sala que todavía no activaron una respuesta y luego captura esa ventana cuando llega una mención u otro activador.
- El mensaje activador actual no se incluye en `InboundHistory`; permanece en el cuerpo principal entrante de ese turno.
- Los reintentos del mismo evento de Matrix reutilizan la instantánea original del historial en lugar de desviarse hacia mensajes más nuevos de la sala.

## Visibilidad del contexto

Matrix admite el control compartido `contextVisibility` para contexto suplementario de sala, como texto de respuesta obtenido, raíces de hilo e historial pendiente.

- `contextVisibility: "all"` es el valor predeterminado. El contexto suplementario se conserva tal como se recibió.
- `contextVisibility: "allowlist"` filtra el contexto suplementario a remitentes permitidos por las comprobaciones activas de allowlist de sala/usuario.
- `contextVisibility: "allowlist_quote"` se comporta como `allowlist`, pero sigue conservando una respuesta citada explícita.

Esta configuración afecta la visibilidad del contexto suplementario, no si el propio mensaje entrante puede activar una respuesta.
La autorización del activador sigue viniendo de `groupPolicy`, `groups`, `groupAllowFrom` y la configuración de políticas de MD.

## Ejemplo de política de MD y sala

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

Consulta [Grupos](/es/channels/groups) para el comportamiento de control por mención y allowlist.

Ejemplo de emparejamiento para MD de Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Si un usuario de Matrix no aprobado sigue enviándote mensajes antes de la aprobación, OpenClaw reutiliza el mismo código de emparejamiento pendiente y puede volver a enviar una respuesta de recordatorio tras un breve enfriamiento en lugar de emitir un código nuevo.

Consulta [Emparejamiento](/es/channels/pairing) para el flujo compartido de emparejamiento de MD y el diseño de almacenamiento.

## Aprobaciones de exec

Matrix puede actuar como cliente nativo de aprobación para una cuenta de Matrix. Los
controles nativos de enrutamiento de MD/canal siguen estando en la configuración de aprobación de exec:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (opcional; recurre a `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Los aprobadores deben ser ID de usuario de Matrix, como `@owner:example.org`. Matrix habilita automáticamente las aprobaciones nativas cuando `enabled` no está configurado o es `"auto"` y se puede resolver al menos un aprobador. Las aprobaciones de exec usan primero `execApprovals.approvers` y pueden recurrir a `channels.matrix.dm.allowFrom`. Las aprobaciones de plugins autorizan a través de `channels.matrix.dm.allowFrom`. Establece `enabled: false` para deshabilitar explícitamente Matrix como cliente nativo de aprobación. De lo contrario, las solicitudes de aprobación recurren a otras rutas de aprobación configuradas o a la política de respaldo de aprobación.

El enrutamiento nativo de Matrix ahora admite ambos tipos de aprobación:

- `channels.matrix.execApprovals.*` controla el modo nativo de distribución en MD/canal para los prompts de aprobación de Matrix.
- Las aprobaciones de exec usan el conjunto de aprobadores de exec de `execApprovals.approvers` o `channels.matrix.dm.allowFrom`.
- Las aprobaciones de plugins usan la allowlist de MD de Matrix de `channels.matrix.dm.allowFrom`.
- Los atajos por reacción y las actualizaciones de mensajes de Matrix se aplican tanto a las aprobaciones de exec como a las de plugins.

Reglas de entrega:

- `target: "dm"` envía los prompts de aprobación a las MD de los aprobadores
- `target: "channel"` envía el prompt de vuelta a la sala o MD de Matrix de origen
- `target: "both"` envía el prompt a las MD de los aprobadores y a la sala o MD de Matrix de origen

Los prompts de aprobación de Matrix siembran atajos de reacción en el mensaje principal de aprobación:

- `✅` = permitir una vez
- `❌` = denegar
- `♾️` = permitir siempre cuando esa decisión está permitida por la política efectiva de exec

Los aprobadores pueden reaccionar en ese mensaje o usar los comandos slash de respaldo: `/approve <id> allow-once`, `/approve <id> allow-always` o `/approve <id> deny`.

Solo los aprobadores resueltos pueden aprobar o denegar. Para las aprobaciones de exec, la entrega por canal incluye el texto del comando, así que solo habilita `channel` o `both` en salas de confianza.

Los prompts de aprobación de Matrix reutilizan el planificador compartido principal de aprobaciones. La superficie nativa específica de Matrix gestiona el enrutamiento de sala/MD, las reacciones y el comportamiento de envío/actualización/eliminación de mensajes tanto para las aprobaciones de exec como para las de plugins.

Anulación por cuenta:

- `channels.matrix.accounts.<account>.execApprovals`

Documentación relacionada: [Aprobaciones de exec](/es/tools/exec-approvals)

## Ejemplo de varias cuentas

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

Los valores de nivel superior `channels.matrix` actúan como predeterminados para las cuentas con nombre, a menos que una cuenta los reemplace.
Puedes limitar las entradas de sala heredadas a una cuenta de Matrix con `groups.<room>.account` (o el heredado `rooms.<room>.account`).
Las entradas sin `account` permanecen compartidas entre todas las cuentas de Matrix, y las entradas con `account: "default"` siguen funcionando cuando la cuenta predeterminada está configurada directamente en el nivel superior `channels.matrix.*`.
Los valores predeterminados parciales de autenticación compartida no crean por sí solos una cuenta predeterminada implícita independiente. OpenClaw solo sintetiza la cuenta `default` de nivel superior cuando esa cuenta predeterminada tiene autenticación nueva (`homeserver` más `accessToken`, o `homeserver` más `userId` y `password`); las cuentas con nombre pueden seguir siendo detectables a partir de `homeserver` más `userId` cuando las credenciales en caché satisfacen la autenticación más adelante.
Si Matrix ya tiene exactamente una cuenta con nombre, o `defaultAccount` apunta a una clave existente de cuenta con nombre, la promoción de reparación/configuración de una sola cuenta a varias cuentas conserva esa cuenta en lugar de crear una nueva entrada `accounts.default`. Solo las claves de autenticación/bootstrap de Matrix se mueven a esa cuenta promovida; las claves compartidas de política de entrega permanecen en el nivel superior.
Establece `defaultAccount` cuando quieras que OpenClaw prefiera una cuenta de Matrix con nombre para el enrutamiento implícito, sondeos y operaciones de CLI.
Si configuras varias cuentas con nombre, establece `defaultAccount` o pasa `--account <id>` para los comandos de CLI que dependan de la selección implícita de cuenta.
Pasa `--account <id>` a `openclaw matrix verify ...` y `openclaw matrix devices ...` cuando quieras anular esa selección implícita para un comando.

## Homeservers privados/LAN

De forma predeterminada, OpenClaw bloquea los homeservers de Matrix privados/internos por protección SSRF a menos que
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

Ejemplo de configuración por CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Esta aceptación explícita solo permite objetivos privados/internos de confianza. Los homeservers públicos en texto claro, como
`http://matrix.example.org:8008`, siguen bloqueados. Prefiere `https://` siempre que sea posible.

## Uso de proxy para tráfico de Matrix

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

Las cuentas con nombre pueden anular el valor predeterminado de nivel superior con `channels.matrix.accounts.<id>.proxy`.
OpenClaw usa la misma configuración de proxy para el tráfico de Matrix en ejecución y para los sondeos de estado de la cuenta.

## Resolución de objetivos

Matrix acepta estas formas de destino en cualquier lugar donde OpenClaw te pida un destino de sala o usuario:

- Usuarios: `@user:server`, `user:@user:server` o `matrix:user:@user:server`
- Salas: `!room:server`, `room:!room:server` o `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server` o `matrix:channel:#alias:server`

La búsqueda en vivo del directorio usa la cuenta de Matrix con sesión iniciada:

- Las búsquedas de usuarios consultan el directorio de usuarios de Matrix en ese homeserver.
- Las búsquedas de salas aceptan directamente IDs y alias de sala explícitos y luego recurren a la búsqueda de nombres de salas unidas para esa cuenta.
- La búsqueda por nombre de sala unida es de mejor esfuerzo. Si el nombre de una sala no puede resolverse a un ID o alias, se ignora en la resolución de allowlist en tiempo de ejecución.

## Referencia de configuración

- `enabled`: habilita o deshabilita el canal.
- `name`: etiqueta opcional para la cuenta.
- `defaultAccount`: ID de cuenta preferido cuando hay varias cuentas de Matrix configuradas.
- `homeserver`: URL del homeserver, por ejemplo `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: permite que esta cuenta de Matrix se conecte a homeservers privados/internos. Habilítalo cuando el homeserver se resuelva a `localhost`, una IP LAN/Tailscale o un host interno como `matrix-synapse`.
- `proxy`: URL opcional de proxy HTTP(S) para tráfico de Matrix. Las cuentas con nombre pueden reemplazar el valor predeterminado de nivel superior con su propio `proxy`.
- `userId`: ID completo de usuario de Matrix, por ejemplo `@bot:example.org`.
- `accessToken`: token de acceso para autenticación basada en token. Se admiten valores en texto plano y valores SecretRef para `channels.matrix.accessToken` y `channels.matrix.accounts.<id>.accessToken` en proveedores env/file/exec. Consulta [Gestión de secretos](/es/gateway/secrets).
- `password`: contraseña para inicio de sesión basado en contraseña. Se admiten valores en texto plano y valores SecretRef.
- `deviceId`: ID explícito del dispositivo de Matrix.
- `deviceName`: nombre para mostrar del dispositivo para inicio de sesión con contraseña.
- `avatarUrl`: URL almacenada del avatar propio para sincronización de perfil y actualizaciones `set-profile`.
- `initialSyncLimit`: límite de eventos de sincronización al iniciar.
- `encryption`: habilita E2EE.
- `allowlistOnly`: fuerza comportamiento solo con allowlist para MD y salas.
- `allowBots`: permite mensajes de otras cuentas configuradas de Matrix de OpenClaw (`true` o `"mentions"`).
- `groupPolicy`: `open`, `allowlist` o `disabled`.
- `contextVisibility`: modo de visibilidad de contexto suplementario de sala (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: allowlist de ID de usuario para tráfico de sala.
- Las entradas de `groupAllowFrom` deben ser ID completos de usuario de Matrix. Los nombres no resueltos se ignoran en tiempo de ejecución.
- `historyLimit`: cantidad máxima de mensajes de sala que se incluirán como contexto de historial de grupo. Recurre a `messages.groupChat.historyLimit`; si ambos no están configurados, el valor predeterminado efectivo es `0`. Establece `0` para deshabilitarlo.
- `replyToMode`: `off`, `first`, `all` o `batched`.
- `markdown`: configuración opcional de renderizado Markdown para texto saliente de Matrix.
- `streaming`: `off` (predeterminado), `partial`, `quiet`, `true` o `false`. `partial` y `true` habilitan actualizaciones de borrador basadas primero en vista previa con mensajes de texto normales de Matrix. `quiet` usa avisos de vista previa sin notificación para configuraciones autoalojadas de reglas push.
- `blockStreaming`: `true` habilita mensajes de progreso independientes para bloques completados del asistente mientras la vista previa en streaming del borrador está activa.
- `threadReplies`: `off`, `inbound` o `always`.
- `threadBindings`: anulaciones por canal para enrutamiento y ciclo de vida de sesiones vinculadas a hilos.
- `startupVerification`: modo automático de solicitud de autoverificación al iniciar (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: enfriamiento antes de reintentar solicitudes automáticas de verificación al iniciar.
- `textChunkLimit`: tamaño de fragmento de mensaje saliente.
- `chunkMode`: `length` o `newline`.
- `responsePrefix`: prefijo opcional de mensaje para respuestas salientes.
- `ackReaction`: anulación opcional de reacción de confirmación para este canal/cuenta.
- `ackReactionScope`: anulación opcional del alcance de reacción de confirmación (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: modo de notificación de reacciones entrantes (`own`, `off`).
- `mediaMaxMb`: límite de tamaño multimedia en MB para el manejo multimedia de Matrix. Se aplica a envíos salientes y al procesamiento multimedia entrante.
- `autoJoin`: política de unión automática a invitaciones (`always`, `allowlist`, `off`). Predeterminado: `off`. Esto se aplica a las invitaciones de Matrix en general, incluidas las invitaciones de estilo MD, no solo a las invitaciones de sala/grupo. OpenClaw toma esta decisión en el momento de la invitación, antes de poder clasificar con fiabilidad la sala unida como MD o grupo.
- `autoJoinAllowlist`: salas/alias permitidos cuando `autoJoin` es `allowlist`. Las entradas de alias se resuelven a IDs de sala durante el manejo de la invitación; OpenClaw no confía en el estado del alias afirmado por la sala invitada.
- `dm`: bloque de política de MD (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: controla el acceso a MD después de que OpenClaw se haya unido a la sala y la haya clasificado como MD. No cambia si una invitación se une automáticamente.
- Las entradas de `dm.allowFrom` deben ser ID completos de usuario de Matrix, a menos que ya las hayas resuelto mediante búsqueda en vivo del directorio.
- `dm.sessionScope`: `per-user` (predeterminado) o `per-room`. Usa `per-room` cuando quieras que cada sala de MD de Matrix mantenga contexto independiente incluso si el par es el mismo.
- `dm.threadReplies`: anulación de política de hilos solo para MD (`off`, `inbound`, `always`). Reemplaza la configuración de nivel superior `threadReplies` tanto para la ubicación de la respuesta como para el aislamiento de sesiones en MD.
- `execApprovals`: entrega nativa de aprobaciones de exec en Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID de usuario de Matrix autorizados para aprobar solicitudes de exec. Opcional cuando `dm.allowFrom` ya identifica a los aprobadores.
- `execApprovals.target`: `dm | channel | both` (predeterminado: `dm`).
- `accounts`: anulaciones con nombre por cuenta. Los valores de nivel superior `channels.matrix` actúan como predeterminados para estas entradas.
- `groups`: mapa de políticas por sala. Prefiere IDs o alias de sala; los nombres de sala no resueltos se ignoran en tiempo de ejecución. La identidad de sesión/grupo usa el ID estable de sala después de la resolución, mientras que las etiquetas legibles por humanos siguen viniendo de los nombres de sala.
- `groups.<room>.account`: restringe una entrada heredada de sala a una cuenta específica de Matrix en configuraciones con varias cuentas.
- `groups.<room>.allowBots`: anulación a nivel de sala para remitentes de bots configurados (`true` o `"mentions"`).
- `groups.<room>.users`: allowlist de remitentes por sala.
- `groups.<room>.tools`: anulaciones por sala para permitir/denegar herramientas.
- `groups.<room>.autoReply`: anulación a nivel de sala para control por mención. `true` deshabilita los requisitos de mención para esa sala; `false` los vuelve a forzar.
- `groups.<room>.skills`: filtro opcional de Skills a nivel de sala.
- `groups.<room>.systemPrompt`: fragmento opcional de system prompt a nivel de sala.
- `rooms`: alias heredado para `groups`.
- `actions`: control por acción de herramientas (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Relacionado

- [Resumen de canales](/es/channels) — todos los canales compatibles
- [Emparejamiento](/es/channels/pairing) — autenticación de MD y flujo de emparejamiento
- [Grupos](/es/channels/groups) — comportamiento del chat grupal y control por mención
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesión para mensajes
- [Seguridad](/es/gateway/security) — modelo de acceso y refuerzo
