---
read_when:
    - Configurar Matrix en OpenClaw
    - Configurar Matrix E2EE y verificación
summary: Estado de compatibilidad, configuración y ejemplos de configuración de Matrix
title: Matrix
x-i18n:
    generated_at: "2026-04-05T12:37:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: ba5c49ad2125d97adf66b5517f8409567eff8b86e20224a32fcb940a02cb0659
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix es el plugin de canal Matrix incluido para OpenClaw.
Usa el `matrix-js-sdk` oficial y admite mensajes directos, salas, hilos, multimedia, reacciones, encuestas, ubicación y E2EE.

## Plugin incluido

Matrix se entrega como plugin incluido en las versiones actuales de OpenClaw, por lo que las
compilaciones empaquetadas normales no necesitan una instalación aparte.

Si usas una compilación más antigua o una instalación personalizada que excluye Matrix, instálalo
manualmente:

Instalar desde npm:

```bash
openclaw plugins install @openclaw/matrix
```

Instalar desde una copia local:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Consulta [Plugins](/tools/plugin) para ver el comportamiento de los plugins y las reglas de instalación.

## Configuración

1. Asegúrate de que el plugin Matrix esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea una cuenta de Matrix en tu homeserver.
3. Configura `channels.matrix` con una de estas opciones:
   - `homeserver` + `accessToken`, o
   - `homeserver` + `userId` + `password`.
4. Reinicia el gateway.
5. Inicia un mensaje directo con el bot o invítalo a una sala.

Rutas de configuración interactiva:

```bash
openclaw channels add
openclaw configure --section channels
```

Lo que realmente pregunta el asistente de Matrix:

- URL del homeserver
- método de autenticación: token de acceso o contraseña
- ID de usuario solo cuando eliges autenticación con contraseña
- nombre del dispositivo opcional
- si quieres habilitar E2EE
- si quieres configurar ahora el acceso a salas de Matrix

Comportamiento importante del asistente:

- Si las variables de entorno de autenticación de Matrix ya existen para la cuenta seleccionada y esa cuenta todavía no tiene la autenticación guardada en la configuración, el asistente ofrece un atajo mediante variables de entorno y solo escribe `enabled: true` para esa cuenta.
- Cuando añades otra cuenta de Matrix de forma interactiva, el nombre de cuenta introducido se normaliza al ID de cuenta usado en la configuración y en las variables de entorno. Por ejemplo, `Ops Bot` se convierte en `ops-bot`.
- Los prompts de lista de permitidos de mensajes directos aceptan de inmediato valores completos `@user:server`. Los nombres para mostrar solo funcionan cuando la búsqueda en vivo del directorio encuentra una coincidencia exacta; en caso contrario, el asistente te pide reintentar con un ID completo de Matrix.
- Los prompts de lista de permitidos de salas aceptan directamente IDs y alias de sala. También pueden resolver en vivo nombres de salas unidas, pero los nombres no resueltos solo se conservan tal como se escribieron durante la configuración y luego se ignoran en la resolución de la lista de permitidos en tiempo de ejecución. Prefiere `!room:server` o `#alias:server`.
- La identidad de sala/sesión en tiempo de ejecución usa el ID estable de la sala de Matrix. Los alias declarados por la sala solo se usan como entradas de búsqueda, no como clave de sesión a largo plazo ni como identidad estable de grupo.
- Para resolver nombres de sala antes de guardarlos, usa `openclaw channels resolve --channel matrix "Project Room"`.

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

Configuración basada en contraseña (el token se almacena en caché tras iniciar sesión):

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

Equivalentes en variables de entorno (se usan cuando la clave de configuración no está definida):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Para cuentas no predeterminadas, usa variables de entorno con ámbito de cuenta:

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

Matrix escapa la puntuación en los ID de cuenta para mantener las variables de entorno con ámbito libres de colisiones.
Por ejemplo, `-` se convierte en `_X2D_`, por lo que `ops-prod` se asigna a `MATRIX_OPS_X2D_PROD_*`.

El asistente interactivo solo ofrece el atajo de variables de entorno cuando esas variables de autenticación ya están presentes y la cuenta seleccionada todavía no tiene la autenticación de Matrix guardada en la configuración.

## Ejemplo de configuración

Esta es una configuración base práctica con emparejamiento de mensajes directos, lista de permitidos de salas y E2EE habilitado:

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

## Vistas previas de streaming

El streaming de respuestas de Matrix es opcional.

Establece `channels.matrix.streaming` en `"partial"` cuando quieras que OpenClaw envíe una sola respuesta en borrador,
edite ese borrador en el mismo lugar mientras el modelo genera texto y luego la finalice cuando la respuesta
termine:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` es el valor predeterminado. OpenClaw espera a la respuesta final y la envía una sola vez.
- `streaming: "partial"` crea un único mensaje de vista previa editable para el bloque actual del asistente en lugar de enviar varios mensajes parciales.
- `blockStreaming: true` habilita mensajes de progreso de Matrix por separado. Con `streaming: "partial"`, Matrix mantiene el borrador activo del bloque actual y conserva los bloques completados como mensajes separados.
- Cuando `streaming: "partial"` y `blockStreaming` está desactivado, Matrix solo edita el borrador activo y envía la respuesta completada una vez que ese bloque o turno termina.
- Si la vista previa ya no cabe en un único evento de Matrix, OpenClaw detiene el streaming de vista previa y vuelve a la entrega final normal.
- Las respuestas con multimedia siguen enviando archivos adjuntos normalmente. Si una vista previa obsoleta ya no puede reutilizarse de forma segura, OpenClaw la redacta antes de enviar la respuesta multimedia final.
- Las ediciones de vista previa generan llamadas adicionales a la API de Matrix. Deja el streaming desactivado si quieres el comportamiento más conservador respecto a límites de tasa.

`blockStreaming` no habilita por sí solo las vistas previas de borrador.
Usa `streaming: "partial"` para las ediciones de vista previa; luego añade `blockStreaming: true` solo si también quieres que los bloques completados del asistente permanezcan visibles como mensajes de progreso separados.

## Cifrado y verificación

En salas cifradas (E2EE), los eventos salientes de imagen usan `thumbnail_file` para que las vistas previas de imagen queden cifradas junto con el archivo adjunto completo. Las salas sin cifrar siguen usando `thumbnail_url` sin cifrar. No hace falta configuración: el plugin detecta automáticamente el estado de E2EE.

### Salas bot a bot

De forma predeterminada, se ignoran los mensajes de otras cuentas de Matrix de OpenClaw configuradas.

Usa `allowBots` cuando quieras permitir intencionalmente tráfico de Matrix entre agentes:

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

- `allowBots: true` acepta mensajes de otras cuentas bot de Matrix configuradas en salas permitidas y mensajes directos.
- `allowBots: "mentions"` acepta esos mensajes solo cuando mencionan visiblemente a este bot en las salas. Los mensajes directos siguen permitidos.
- `groups.<room>.allowBots` anula la configuración a nivel de cuenta para una sala.
- OpenClaw sigue ignorando mensajes del mismo ID de usuario de Matrix para evitar bucles de autorrespuesta.
- Matrix no expone aquí un indicador nativo de bot; OpenClaw trata "creado por bot" como "enviado por otra cuenta de Matrix configurada en este gateway de OpenClaw".

Usa listas estrictas de salas permitidas y requisitos de mención al habilitar tráfico bot a bot en salas compartidas.

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

Incluir la clave de recuperación almacenada en la salida legible por máquinas:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Inicializar el estado de cross-signing y verificación:

```bash
openclaw matrix verify bootstrap
```

Compatibilidad con varias cuentas: usa `channels.matrix.accounts` con credenciales por cuenta y `name` opcional. Consulta [Referencia de configuración](/gateway/configuration-reference#multi-account-all-channels) para ver el patrón compartido.

Diagnóstico detallado de bootstrap:

```bash
openclaw matrix verify bootstrap --verbose
```

Forzar un restablecimiento nuevo de la identidad de cross-signing antes de inicializar:

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

Diagnóstico detallado del estado de salud de la copia de seguridad:

```bash
openclaw matrix verify backup status --verbose
```

Restaurar claves de sala desde la copia de seguridad del servidor:

```bash
openclaw matrix verify backup restore
```

Diagnóstico detallado de la restauración:

```bash
openclaw matrix verify backup restore --verbose
```

Eliminar la copia de seguridad actual del servidor y crear una nueva base de referencia de copia de seguridad. Si la
clave de copia de seguridad almacenada no puede cargarse correctamente, este restablecimiento también puede recrear el almacenamiento secreto para que
futuros arranques en frío puedan cargar la nueva clave de copia de seguridad:

```bash
openclaw matrix verify backup reset --yes
```

Todos los comandos `verify` son concisos de forma predeterminada (incluido el registro interno silencioso del SDK) y solo muestran diagnósticos detallados con `--verbose`.
Usa `--json` para una salida completa legible por máquinas al automatizar.

En configuraciones con varias cuentas, los comandos de CLI de Matrix usan la cuenta predeterminada implícita de Matrix a menos que pases `--account <id>`.
Si configuras varias cuentas con nombre, primero establece `channels.matrix.defaultAccount` o esas operaciones implícitas de CLI se detendrán y te pedirán que elijas una cuenta explícitamente.
Usa `--account` siempre que quieras que las operaciones de verificación o dispositivo apunten explícitamente a una cuenta con nombre:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Cuando el cifrado está desactivado o no disponible para una cuenta con nombre, las advertencias de Matrix y los errores de verificación apuntan a la clave de configuración de esa cuenta, por ejemplo `channels.matrix.accounts.assistant.encryption`.

### Qué significa "verified"

OpenClaw trata este dispositivo de Matrix como verificado solo cuando está verificado por tu propia identidad de cross-signing.
En la práctica, `openclaw matrix verify status --verbose` expone tres señales de confianza:

- `Locally trusted`: este dispositivo solo es de confianza para el cliente actual
- `Cross-signing verified`: el SDK informa que el dispositivo está verificado mediante cross-signing
- `Signed by owner`: el dispositivo está firmado por tu propia clave de autofirma

`Verified by owner` pasa a ser `yes` solo cuando existe verificación por cross-signing o firma del propietario.
La confianza local por sí sola no es suficiente para que OpenClaw trate el dispositivo como totalmente verificado.

### Qué hace bootstrap

`openclaw matrix verify bootstrap` es el comando de reparación y configuración para cuentas de Matrix cifradas.
Hace todo lo siguiente en orden:

- inicializa el almacenamiento secreto, reutilizando una clave de recuperación existente cuando es posible
- inicializa el cross-signing y sube las claves públicas de cross-signing que falten
- intenta marcar y firmar mediante cross-signing el dispositivo actual
- crea una nueva copia de seguridad del lado del servidor para claves de sala si todavía no existe

Si el homeserver requiere autenticación interactiva para subir claves de cross-signing, OpenClaw intenta primero la subida sin autenticación, luego con `m.login.dummy` y después con `m.login.password` cuando `channels.matrix.password` está configurado.

Usa `--force-reset-cross-signing` solo cuando quieras descartar intencionalmente la identidad actual de cross-signing y crear una nueva.

Si intencionalmente quieres descartar la copia de seguridad actual de claves de sala y comenzar una nueva
base de referencia para futuros mensajes, usa `openclaw matrix verify backup reset --yes`.
Hazlo solo si aceptas que el historial cifrado antiguo irrecuperable seguirá
sin estar disponible y que OpenClaw puede recrear el almacenamiento secreto si el secreto
actual de la copia de seguridad no puede cargarse de forma segura.

### Nueva base de referencia de copia de seguridad

Si quieres mantener funcionando los futuros mensajes cifrados y aceptas perder el historial antiguo irrecuperable, ejecuta estos comandos en orden:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Añade `--account <id>` a cada comando cuando quieras apuntar explícitamente a una cuenta de Matrix con nombre.

### Comportamiento al iniciar

Cuando `encryption: true`, Matrix establece `startupVerification` en `"if-unverified"` de forma predeterminada.
Al iniciar, si este dispositivo sigue sin verificarse, Matrix solicitará la autoverificación en otro cliente de Matrix,
omitirá solicitudes duplicadas mientras una ya esté pendiente y aplicará un tiempo de espera local antes de reintentar tras reinicios.
Los intentos fallidos de solicitud se reintentan antes que la creación exitosa de solicitudes de forma predeterminada.
Establece `startupVerification: "off"` para desactivar las solicitudes automáticas al iniciar, o ajusta `startupVerificationCooldownHours`
si quieres una ventana de reintento más corta o más larga.

El inicio también realiza automáticamente una pasada conservadora de bootstrap criptográfico.
Esa pasada primero intenta reutilizar el almacenamiento secreto y la identidad actual de cross-signing, y evita restablecer el cross-signing a menos que ejecutes un flujo explícito de reparación bootstrap.

Si al iniciar se detecta un estado de bootstrap roto y `channels.matrix.password` está configurado, OpenClaw puede intentar una ruta de reparación más estricta.
Si el dispositivo actual ya está firmado por el propietario, OpenClaw conserva esa identidad en lugar de restablecerla automáticamente.

Actualización desde el plugin público anterior de Matrix:

- OpenClaw reutiliza automáticamente la misma cuenta de Matrix, token de acceso e identidad de dispositivo cuando es posible.
- Antes de ejecutar cualquier cambio de migración de Matrix que requiera acción, OpenClaw crea o reutiliza una instantánea de recuperación en `~/Backups/openclaw-migrations/`.
- Si usas varias cuentas de Matrix, establece `channels.matrix.defaultAccount` antes de actualizar desde el diseño anterior de almacenamiento plano para que OpenClaw sepa qué cuenta debe recibir ese estado heredado compartido.
- Si el plugin anterior almacenó localmente una clave de descifrado de copia de seguridad de claves de sala de Matrix, al iniciar o con `openclaw doctor --fix` se importará automáticamente al nuevo flujo de clave de recuperación.
- Si el token de acceso de Matrix cambió después de preparar la migración, el inicio ahora escanea raíces de almacenamiento hermanas con hash de token en busca de estado heredado pendiente de restauración antes de abandonar la restauración automática de la copia de seguridad.
- Si más tarde cambia el token de acceso de Matrix para la misma cuenta, homeserver y usuario, OpenClaw ahora prefiere reutilizar la raíz de almacenamiento existente más completa con hash de token en lugar de empezar desde un directorio de estado Matrix vacío.
- En el siguiente inicio del gateway, las claves de sala respaldadas se restauran automáticamente en el nuevo almacén criptográfico.
- Si el plugin anterior tenía claves de sala solo locales que nunca se respaldaron, OpenClaw mostrará una advertencia clara. Esas claves no pueden exportarse automáticamente desde el almacén criptográfico Rust anterior, por lo que parte del historial cifrado antiguo puede seguir sin estar disponible hasta recuperarse manualmente.
- Consulta [Migración de Matrix](/install/migrating-matrix) para ver el flujo completo de actualización, límites, comandos de recuperación y mensajes comunes de migración.

El estado cifrado en tiempo de ejecución se organiza bajo raíces por cuenta, por usuario y por hash de token en
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Ese directorio contiene el almacén de sincronización (`bot-storage.json`), el almacén criptográfico (`crypto/`),
el archivo de clave de recuperación (`recovery-key.json`), la instantánea de IndexedDB (`crypto-idb-snapshot.json`),
las asociaciones de hilos (`thread-bindings.json`) y el estado de verificación al iniciar (`startup-verification.json`)
cuando esas funciones están en uso.
Cuando el token cambia pero la identidad de la cuenta sigue siendo la misma, OpenClaw reutiliza la mejor
raíz existente para esa tupla cuenta/homeserver/usuario, de modo que el estado previo de sincronización, criptografía, asociaciones de hilos
y verificación al iniciar siga siendo visible.

### Modelo de almacén criptográfico de Node

Matrix E2EE en este plugin usa la ruta oficial de criptografía Rust de `matrix-js-sdk` en Node.
Esa ruta espera persistencia con respaldo de IndexedDB cuando quieres que el estado criptográfico sobreviva a los reinicios.

Actualmente OpenClaw lo proporciona en Node mediante:

- uso de `fake-indexeddb` como el shim de API de IndexedDB que espera el SDK
- restauración del contenido de IndexedDB de la criptografía Rust desde `crypto-idb-snapshot.json` antes de `initRustCrypto`
- persistencia del contenido actualizado de IndexedDB de vuelta en `crypto-idb-snapshot.json` tras la inicialización y durante el tiempo de ejecución
- serialización de la restauración y persistencia de instantáneas frente a `crypto-idb-snapshot.json` con un bloqueo de archivo consultivo para que la persistencia en tiempo de ejecución del gateway y el mantenimiento por CLI no compitan sobre el mismo archivo de instantánea

Esto es infraestructura de compatibilidad/almacenamiento, no una implementación criptográfica personalizada.
El archivo de instantánea es estado sensible en tiempo de ejecución y se almacena con permisos de archivo restrictivos.
Según el modelo de seguridad de OpenClaw, el host del gateway y el directorio local de estado de OpenClaw ya están dentro del límite del operador de confianza, por lo que esto es principalmente una cuestión de durabilidad operativa más que un límite de confianza remoto separado.

Mejora planificada:

- añadir compatibilidad con SecretRef para material de claves persistentes de Matrix, de modo que las claves de recuperación y secretos relacionados de cifrado del almacén puedan obtenerse desde proveedores de secretos de OpenClaw en lugar de solo desde archivos locales

## Administración de perfil

Actualiza el perfil propio de Matrix para la cuenta seleccionada con:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Añade `--account <id>` cuando quieras apuntar explícitamente a una cuenta con nombre.

Matrix acepta directamente URL de avatar `mxc://`. Cuando pasas una URL de avatar `http://` o `https://`, OpenClaw primero la sube a Matrix y luego almacena la URL `mxc://` resuelta en `channels.matrix.avatarUrl` (o en la anulación de la cuenta seleccionada).

## Avisos automáticos de verificación

Matrix ahora publica avisos del ciclo de vida de verificación directamente en la sala estricta de verificación de mensajes directos como mensajes `m.notice`.
Eso incluye:

- avisos de solicitud de verificación
- avisos de verificación lista (con guía explícita "Verify by emoji")
- avisos de inicio y finalización de verificación
- detalles SAS (emoji y decimal) cuando están disponibles

Las solicitudes de verificación entrantes desde otro cliente Matrix se rastrean y OpenClaw las acepta automáticamente.
Para flujos de autoverificación, OpenClaw también inicia automáticamente el flujo SAS cuando la verificación por emoji está disponible y confirma su propio lado.
Para solicitudes de verificación desde otro usuario/dispositivo de Matrix, OpenClaw acepta automáticamente la solicitud y luego espera a que el flujo SAS continúe con normalidad.
Aun así, necesitas comparar el SAS de emoji o decimal en tu cliente Matrix y confirmar allí "They match" para completar la verificación.

OpenClaw no acepta automáticamente a ciegas flujos duplicados iniciados por sí mismo. Al iniciar se omite crear una nueva solicitud cuando ya hay pendiente una solicitud de autoverificación.

Los avisos de protocolo/sistema de verificación no se reenvían al canalización de chat del agente, por lo que no producen `NO_REPLY`.

### Higiene de dispositivos

Los dispositivos antiguos de Matrix administrados por OpenClaw pueden acumularse en la cuenta y hacer que sea más difícil razonar sobre la confianza en salas cifradas.
Enuméralos con:

```bash
openclaw matrix devices list
```

Elimina dispositivos obsoletos administrados por OpenClaw con:

```bash
openclaw matrix devices prune-stale
```

### Reparación de sala directa

Si el estado de mensajes directos se desincroniza, OpenClaw puede terminar con asignaciones `m.direct` obsoletas que apuntan a salas individuales antiguas en lugar del mensaje directo activo. Inspecciona la asignación actual para un par con:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Repárala con:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

La reparación mantiene la lógica específica de Matrix dentro del plugin:

- prefiere un mensaje directo estricto 1:1 que ya esté asignado en `m.direct`
- en caso contrario, recurre a cualquier mensaje directo estricto 1:1 actualmente unido con ese usuario
- si no existe un mensaje directo sano, crea una nueva sala directa y reescribe `m.direct` para que apunte a ella

El flujo de reparación no elimina automáticamente las salas antiguas. Solo elige el mensaje directo sano y actualiza la asignación para que los nuevos envíos de Matrix, avisos de verificación y otros flujos de mensajes directos vuelvan a apuntar a la sala correcta.

## Hilos

Matrix admite hilos nativos de Matrix tanto para respuestas automáticas como para envíos con la herramienta de mensajes.

- `threadReplies: "off"` mantiene las respuestas al nivel superior y mantiene los mensajes entrantes con hilo en la sesión padre.
- `threadReplies: "inbound"` responde dentro de un hilo solo cuando el mensaje entrante ya estaba en ese hilo.
- `threadReplies: "always"` mantiene las respuestas de la sala en un hilo con raíz en el mensaje activador y dirige esa conversación a través de la sesión correspondiente con ámbito de hilo desde el primer mensaje activador.
- `dm.threadReplies` anula la configuración de nivel superior solo para mensajes directos. Por ejemplo, puedes mantener aislados los hilos de sala y mantener planos los mensajes directos.
- Los mensajes entrantes con hilo incluyen el mensaje raíz del hilo como contexto adicional para el agente.
- Los envíos con la herramienta de mensajes ahora heredan automáticamente el hilo actual de Matrix cuando el destino es la misma sala, o el mismo usuario de mensaje directo, salvo que se proporcione un `threadId` explícito.
- Las asociaciones de hilos en tiempo de ejecución son compatibles con Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y `/acp spawn` vinculado a hilos ahora funcionan en salas y mensajes directos de Matrix.
- `/focus` en una sala o mensaje directo de Matrix de nivel superior crea un nuevo hilo de Matrix y lo vincula a la sesión de destino cuando `threadBindings.spawnSubagentSessions=true`.
- Ejecutar `/focus` o `/acp spawn --thread here` dentro de un hilo existente de Matrix vincula ese hilo actual en su lugar.

## Asociaciones de conversación ACP

Las salas, los mensajes directos y los hilos existentes de Matrix pueden convertirse en espacios de trabajo ACP duraderos sin cambiar la superficie del chat.

Flujo rápido para operadores:

- Ejecuta `/acp spawn codex --bind here` dentro del mensaje directo, sala o hilo existente de Matrix que quieras seguir usando.
- En un mensaje directo o sala de Matrix de nivel superior, el mensaje directo/sala actual sigue siendo la superficie de chat y los mensajes futuros se enrutan a la sesión ACP generada.
- Dentro de un hilo existente de Matrix, `--bind here` vincula ese hilo actual en su lugar.
- `/new` y `/reset` restablecen la misma sesión ACP vinculada en su lugar.
- `/acp close` cierra la sesión ACP y elimina la asociación.

Notas:

- `--bind here` no crea un hilo hijo de Matrix.
- `threadBindings.spawnAcpSessions` solo se requiere para `/acp spawn --thread auto|here`, donde OpenClaw necesita crear o vincular un hilo hijo de Matrix.

### Configuración de asociación de hilos

Matrix hereda los valores predeterminados globales de `session.threadBindings` y también admite anulaciones por canal:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Los indicadores de generación vinculada a hilos de Matrix son opcionales:

- Establece `threadBindings.spawnSubagentSessions: true` para permitir que `/focus` de nivel superior cree y vincule nuevos hilos de Matrix.
- Establece `threadBindings.spawnAcpSessions: true` para permitir que `/acp spawn --thread auto|here` vincule sesiones ACP a hilos de Matrix.

## Reacciones

Matrix admite acciones de reacción salientes, notificaciones de reacción entrantes y reacciones entrantes de confirmación.

- La herramienta de reacción saliente está controlada por `channels["matrix"].actions.reactions`.
- `react` añade una reacción a un evento específico de Matrix.
- `reactions` enumera el resumen actual de reacciones para un evento específico de Matrix.
- `emoji=""` elimina las reacciones propias de la cuenta bot en ese evento.
- `remove: true` elimina solo la reacción del emoji especificado de la cuenta bot.

El ámbito de las reacciones de confirmación se resuelve en este orden estándar de OpenClaw:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- respaldo al emoji de identidad del agente

El ámbito de `ackReaction` se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

El modo de notificación de reacciones se resuelve en este orden:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- predeterminado: `own`

Comportamiento actual:

- `reactionNotifications: "own"` reenvía eventos `m.reaction` añadidos cuando apuntan a mensajes de Matrix creados por el bot.
- `reactionNotifications: "off"` desactiva los eventos del sistema de reacciones.
- Las eliminaciones de reacciones todavía no se sintetizan en eventos del sistema porque Matrix las muestra como redacciones, no como eliminaciones independientes de `m.reaction`.

## Contexto del historial

- `channels.matrix.historyLimit` controla cuántos mensajes recientes de la sala se incluyen como `InboundHistory` cuando un mensaje de sala de Matrix activa al agente.
- Recurre a `messages.groupChat.historyLimit`. Establece `0` para desactivarlo.
- El historial de salas de Matrix es solo de la sala. Los mensajes directos siguen usando el historial normal de la sesión.
- El historial de salas de Matrix es solo pendiente: OpenClaw almacena en búfer los mensajes de sala que todavía no activaron una respuesta y luego toma una instantánea de esa ventana cuando llega una mención u otro activador.
- El mensaje activador actual no se incluye en `InboundHistory`; permanece en el cuerpo principal entrante de ese turno.
- Los reintentos del mismo evento de Matrix reutilizan la instantánea original del historial en lugar de desplazarse hacia mensajes más recientes de la sala.

## Visibilidad del contexto

Matrix admite el control compartido `contextVisibility` para contexto suplementario de sala, como texto de respuesta obtenido, raíces de hilos e historial pendiente.

- `contextVisibility: "all"` es el valor predeterminado. El contexto suplementario se conserva tal como se recibió.
- `contextVisibility: "allowlist"` filtra el contexto suplementario a remitentes permitidos por las comprobaciones activas de listas de permitidos de sala/usuario.
- `contextVisibility: "allowlist_quote"` se comporta como `allowlist`, pero aun así conserva una respuesta citada explícita.

Esta configuración afecta la visibilidad del contexto suplementario, no si el propio mensaje entrante puede activar una respuesta.
La autorización del activador sigue viniendo de `groupPolicy`, `groups`, `groupAllowFrom` y la configuración de política de mensajes directos.

## Ejemplo de política de mensajes directos y salas

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

Consulta [Groups](/channels/groups) para el comportamiento de restricción por mención y listas de permitidos.

Ejemplo de emparejamiento para mensajes directos de Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Si un usuario de Matrix no aprobado sigue enviándote mensajes antes de la aprobación, OpenClaw reutiliza el mismo código de emparejamiento pendiente y puede volver a enviar una respuesta recordatoria tras un breve tiempo de espera en lugar de generar un código nuevo.

Consulta [Pairing](/channels/pairing) para el flujo compartido de emparejamiento de mensajes directos y el diseño del almacenamiento.

## Aprobaciones de exec

Matrix puede actuar como cliente de aprobación de exec para una cuenta de Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (opcional; recurre a `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Las personas aprobadoras deben ser ID de usuario de Matrix como `@owner:example.org`. Matrix habilita automáticamente las aprobaciones nativas de exec cuando `enabled` no está definido o es `"auto"` y puede resolverse al menos una persona aprobadora, ya sea desde `execApprovals.approvers` o desde `channels.matrix.dm.allowFrom`. Establece `enabled: false` para desactivar explícitamente Matrix como cliente nativo de aprobación. En caso contrario, las solicitudes de aprobación recurren a otras rutas de aprobación configuradas o a la política de respaldo de aprobación de exec.

El enrutamiento nativo de Matrix hoy es solo para exec:

- `channels.matrix.execApprovals.*` controla el enrutamiento nativo de mensajes directos/canales solo para aprobaciones de exec.
- Las aprobaciones de plugins siguen usando `/approve` en el mismo chat compartido más cualquier reenvío configurado en `approvals.plugin`.
- Matrix aún puede reutilizar `channels.matrix.dm.allowFrom` para la autorización de aprobaciones de plugins cuando puede inferir con seguridad quién aprueba, pero no expone una ruta nativa separada de distribución por mensajes directos/canales para aprobaciones de plugins.

Reglas de entrega:

- `target: "dm"` envía los prompts de aprobación a los mensajes directos de quienes aprueban
- `target: "channel"` devuelve el prompt a la sala o mensaje directo de Matrix de origen
- `target: "both"` envía a los mensajes directos de quienes aprueban y a la sala o mensaje directo de Matrix de origen

Matrix usa hoy prompts de aprobación de texto. Quienes aprueban los resuelven con `/approve <id> allow-once`, `/approve <id> allow-always` o `/approve <id> deny`.

Solo las personas aprobadoras resueltas pueden aprobar o denegar. La entrega al canal incluye el texto del comando, por lo que solo debes habilitar `channel` o `both` en salas de confianza.

Los prompts de aprobación de Matrix reutilizan el planificador de aprobaciones compartido del núcleo. La superficie nativa específica de Matrix es solo transporte para aprobaciones de exec: enrutamiento de sala/mensaje directo y comportamiento de envío/actualización/eliminación de mensajes.

Anulación por cuenta:

- `channels.matrix.accounts.<account>.execApprovals`

Documentación relacionada: [Aprobaciones de exec](/tools/exec-approvals)

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

Los valores de nivel superior de `channels.matrix` actúan como predeterminados para las cuentas con nombre, a menos que una cuenta los anule.
Puedes limitar una entrada de sala heredada a una cuenta concreta de Matrix con `groups.<room>.account` (o el legado `rooms.<room>.account`).
Las entradas sin `account` permanecen compartidas entre todas las cuentas de Matrix, y las entradas con `account: "default"` siguen funcionando cuando la cuenta predeterminada se configura directamente en `channels.matrix.*` de nivel superior.
Los valores predeterminados parciales compartidos de autenticación no crean por sí solos una cuenta predeterminada implícita separada. OpenClaw solo sintetiza la cuenta `default` de nivel superior cuando esa cuenta predeterminada tiene autenticación nueva (`homeserver` más `accessToken`, o `homeserver` más `userId` y `password`); las cuentas con nombre pueden seguir siendo detectables desde `homeserver` más `userId` cuando las credenciales almacenadas en caché satisfacen la autenticación más adelante.
Si Matrix ya tiene exactamente una cuenta con nombre, o `defaultAccount` apunta a una clave existente de cuenta con nombre, la promoción de reparación/configuración de una sola cuenta a varias cuentas conserva esa cuenta en lugar de crear una nueva entrada `accounts.default`. Solo las claves de autenticación/bootstrap de Matrix se mueven a esa cuenta promovida; las claves compartidas de política de entrega permanecen en el nivel superior.
Establece `defaultAccount` cuando quieras que OpenClaw prefiera una cuenta de Matrix con nombre para enrutamiento implícito, sondeos y operaciones de CLI.
Si configuras varias cuentas con nombre, establece `defaultAccount` o pasa `--account <id>` para los comandos de CLI que dependen de la selección implícita de cuenta.
Pasa `--account <id>` a `openclaw matrix verify ...` y `openclaw matrix devices ...` cuando quieras anular esa selección implícita para un comando.

## Homeservers privados/LAN

De forma predeterminada, OpenClaw bloquea los homeservers de Matrix privados/internos como protección SSRF, a menos que
optes explícitamente por permitirlos por cuenta.

Si tu homeserver se ejecuta en localhost, una IP de LAN/Tailscale o un nombre de host interno, habilita
`allowPrivateNetwork` para esa cuenta de Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      allowPrivateNetwork: true,
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

Esta opción solo permite destinos privados/internos de confianza. Los homeservers públicos en texto claro como
`http://matrix.example.org:8008` siguen bloqueados. Prefiere `https://` siempre que sea posible.

## Uso de proxy para el tráfico de Matrix

Si tu implementación de Matrix necesita un proxy HTTP(S) saliente explícito, establece `channels.matrix.proxy`:

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
OpenClaw usa la misma configuración de proxy para el tráfico de Matrix en tiempo de ejecución y para los sondeos de estado de la cuenta.

## Resolución de destino

Matrix acepta estos formatos de destino en cualquier lugar donde OpenClaw te pida un objetivo de sala o usuario:

- Usuarios: `@user:server`, `user:@user:server` o `matrix:user:@user:server`
- Salas: `!room:server`, `room:!room:server` o `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server` o `matrix:channel:#alias:server`

La búsqueda en vivo del directorio usa la cuenta de Matrix con la sesión iniciada:

- Las búsquedas de usuario consultan el directorio de usuarios de Matrix en ese homeserver.
- Las búsquedas de sala aceptan directamente IDs y alias explícitos de sala, y luego recurren a buscar nombres de salas unidas para esa cuenta.
- La búsqueda por nombre de sala unida es de mejor esfuerzo. Si un nombre de sala no puede resolverse a un ID o alias, se ignora en la resolución de listas de permitidos en tiempo de ejecución.

## Referencia de configuración

- `enabled`: habilitar o deshabilitar el canal.
- `name`: etiqueta opcional para la cuenta.
- `defaultAccount`: ID de cuenta preferido cuando hay varias cuentas de Matrix configuradas.
- `homeserver`: URL del homeserver, por ejemplo `https://matrix.example.org`.
- `allowPrivateNetwork`: permite que esta cuenta de Matrix se conecte a homeservers privados/internos. Habilítalo cuando el homeserver resuelva a `localhost`, una IP de LAN/Tailscale o un host interno como `matrix-synapse`.
- `proxy`: URL opcional de proxy HTTP(S) para el tráfico de Matrix. Las cuentas con nombre pueden anular el valor predeterminado de nivel superior con su propio `proxy`.
- `userId`: ID completo de usuario de Matrix, por ejemplo `@bot:example.org`.
- `accessToken`: token de acceso para autenticación basada en token. Se admiten valores en texto plano y valores SecretRef para `channels.matrix.accessToken` y `channels.matrix.accounts.<id>.accessToken` en proveedores env/file/exec. Consulta [Administración de secretos](/gateway/secrets).
- `password`: contraseña para inicio de sesión basado en contraseña. Se admiten valores en texto plano y valores SecretRef.
- `deviceId`: ID explícito del dispositivo de Matrix.
- `deviceName`: nombre para mostrar del dispositivo para inicio de sesión con contraseña.
- `avatarUrl`: URL de avatar propio almacenada para sincronización de perfil y actualizaciones `set-profile`.
- `initialSyncLimit`: límite de eventos de sincronización al iniciar.
- `encryption`: habilitar E2EE.
- `allowlistOnly`: forzar comportamiento solo de lista de permitidos para mensajes directos y salas.
- `allowBots`: permitir mensajes de otras cuentas de Matrix de OpenClaw configuradas (`true` o `"mentions"`).
- `groupPolicy`: `open`, `allowlist` o `disabled`.
- `contextVisibility`: modo de visibilidad del contexto suplementario de sala (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: lista de permitidos de ID de usuario para tráfico de sala.
- Las entradas de `groupAllowFrom` deben ser IDs completos de usuario de Matrix. Los nombres no resueltos se ignoran en tiempo de ejecución.
- `historyLimit`: máximo de mensajes de sala que se incluirán como contexto de historial de grupo. Recurre a `messages.groupChat.historyLimit`. Establece `0` para desactivarlo.
- `replyToMode`: `off`, `first` o `all`.
- `markdown`: configuración opcional de renderizado Markdown para texto saliente de Matrix.
- `streaming`: `off` (predeterminado), `partial`, `true` o `false`. `partial` y `true` habilitan vistas previas de borrador de un solo mensaje con actualizaciones de edición en el mismo lugar.
- `blockStreaming`: `true` habilita mensajes de progreso separados para bloques completados del asistente mientras el streaming de vista previa de borrador está activo.
- `threadReplies`: `off`, `inbound` o `always`.
- `threadBindings`: anulaciones por canal para enrutamiento y ciclo de vida de sesiones vinculadas a hilos.
- `startupVerification`: modo automático de solicitud de autoverificación al iniciar (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: tiempo de espera antes de reintentar solicitudes automáticas de verificación al iniciar.
- `textChunkLimit`: tamaño de fragmento de mensajes salientes.
- `chunkMode`: `length` o `newline`.
- `responsePrefix`: prefijo opcional de mensaje para respuestas salientes.
- `ackReaction`: anulación opcional de reacción de confirmación para este canal/cuenta.
- `ackReactionScope`: anulación opcional del ámbito de reacción de confirmación (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: modo de notificación de reacciones entrantes (`own`, `off`).
- `mediaMaxMb`: límite de tamaño multimedia en MB para el manejo de multimedia de Matrix. Se aplica a envíos salientes y al procesamiento de multimedia entrante.
- `autoJoin`: política de unión automática a invitaciones (`always`, `allowlist`, `off`). Predeterminado: `off`.
- `autoJoinAllowlist`: salas/alias permitidos cuando `autoJoin` es `allowlist`. Las entradas de alias se resuelven a IDs de sala durante el manejo de invitaciones; OpenClaw no confía en el estado de alias declarado por la sala invitada.
- `dm`: bloque de política de mensajes directos (`enabled`, `policy`, `allowFrom`, `threadReplies`).
- Las entradas de `dm.allowFrom` deben ser IDs completos de usuario de Matrix, a menos que ya las hayas resuelto mediante una búsqueda en vivo del directorio.
- `dm.threadReplies`: anulación de política de hilos solo para mensajes directos (`off`, `inbound`, `always`). Anula la configuración de nivel superior `threadReplies` tanto para la ubicación de respuestas como para el aislamiento de sesiones en mensajes directos.
- `execApprovals`: entrega nativa de aprobaciones de exec de Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID de usuario de Matrix con permiso para aprobar solicitudes de exec. Es opcional cuando `dm.allowFrom` ya identifica quién aprueba.
- `execApprovals.target`: `dm | channel | both` (predeterminado: `dm`).
- `accounts`: anulaciones con nombre por cuenta. Los valores de nivel superior de `channels.matrix` actúan como predeterminados para estas entradas.
- `groups`: mapa de políticas por sala. Prefiere IDs o alias de sala; los nombres de sala no resueltos se ignoran en tiempo de ejecución. La identidad de sesión/grupo usa el ID estable de la sala tras la resolución, mientras que las etiquetas legibles para humanos siguen viniendo de los nombres de sala.
- `groups.<room>.account`: restringe una entrada de sala heredada a una cuenta específica de Matrix en configuraciones con varias cuentas.
- `groups.<room>.allowBots`: anulación a nivel de sala para remitentes bot configurados (`true` o `"mentions"`).
- `groups.<room>.users`: lista de permitidos de remitentes por sala.
- `groups.<room>.tools`: anulaciones de permiso/denegación de herramientas por sala.
- `groups.<room>.autoReply`: anulación a nivel de sala para restricción por mención. `true` desactiva los requisitos de mención para esa sala; `false` los vuelve a forzar.
- `groups.<room>.skills`: filtro opcional de Skills a nivel de sala.
- `groups.<room>.systemPrompt`: fragmento opcional de prompt del sistema a nivel de sala.
- `rooms`: alias heredado de `groups`.
- `actions`: control por acción para herramientas (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Groups](/channels/groups) — comportamiento del chat de grupo y restricción por menciones
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y endurecimiento
