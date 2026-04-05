---
read_when:
    - Configurando la compatibilidad con Signal
    - Depurando el envío/la recepción de Signal
summary: Compatibilidad con Signal mediante signal-cli (JSON-RPC + SSE), rutas de configuración y modelo de números
title: Signal
x-i18n:
    generated_at: "2026-04-05T12:36:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: cdd855eb353aca6a1c2b04d14af0e3da079349297b54fa8243562c52b29118d9
    source_path: channels/signal.md
    workflow: 15
---

# Signal (signal-cli)

Estado: integración con CLI externa. El Gateway se comunica con `signal-cli` mediante HTTP JSON-RPC + SSE.

## Requisitos previos

- OpenClaw instalado en tu servidor (el flujo de Linux a continuación se probó en Ubuntu 24).
- `signal-cli` disponible en el host donde se ejecuta el gateway.
- Un número de teléfono que pueda recibir un SMS de verificación (para la ruta de registro por SMS).
- Acceso a un navegador para el captcha de Signal (`signalcaptchas.org`) durante el registro.

## Configuración rápida (principiante)

1. Usa un **número de Signal independiente** para el bot (recomendado).
2. Instala `signal-cli` (se requiere Java si usas la compilación JVM).
3. Elige una ruta de configuración:
   - **Ruta A (vinculación por QR):** `signal-cli link -n "OpenClaw"` y escanea con Signal.
   - **Ruta B (registro por SMS):** registra un número dedicado con captcha + verificación por SMS.
4. Configura OpenClaw y reinicia el gateway.
5. Envía un primer mensaje directo y aprueba el pairing (`openclaw pairing approve signal <CODE>`).

Configuración mínima:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

Referencia de campos:

| Field       | Descripción                                        |
| ----------- | -------------------------------------------------- |
| `account`   | Número de teléfono del bot en formato E.164 (`+15551234567`) |
| `cliPath`   | Ruta a `signal-cli` (`signal-cli` si está en `PATH`) |
| `dmPolicy`  | Política de acceso a mensajes directos (`pairing` recomendado) |
| `allowFrom` | Números de teléfono o valores `uuid:<id>` con permiso para mensajes directos |

## Qué es

- Canal de Signal mediante `signal-cli` (no `libsignal` integrado).
- Enrutamiento determinista: las respuestas siempre vuelven a Signal.
- Los mensajes directos comparten la sesión principal del agente; los grupos están aislados (`agent:<agentId>:signal:group:<groupId>`).

## Escrituras de configuración

De forma predeterminada, Signal puede escribir actualizaciones de configuración activadas por `/config set|unset` (requiere `commands.config: true`).

Desactívalo con:

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## El modelo de números (importante)

- El gateway se conecta a un **dispositivo de Signal** (la cuenta de `signal-cli`).
- Si ejecutas el bot en **tu cuenta personal de Signal**, ignorará tus propios mensajes (protección contra bucles).
- Para el caso "le escribo al bot y me responde", usa un **número de bot independiente**.

## Ruta de configuración A: vincular una cuenta de Signal existente (QR)

1. Instala `signal-cli` (compilación JVM o nativa).
2. Vincula una cuenta de bot:
   - `signal-cli link -n "OpenClaw"` y luego escanea el QR en Signal.
3. Configura Signal e inicia el gateway.

Ejemplo:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

Compatibilidad con varias cuentas: usa `channels.signal.accounts` con configuración por cuenta y `name` opcional. Consulta [`gateway/configuration`](/gateway/configuration-reference#multi-account-all-channels) para ver el patrón compartido.

## Ruta de configuración B: registrar un número de bot dedicado (SMS, Linux)

Usa esta opción si quieres un número de bot dedicado en lugar de vincular una cuenta existente de la app de Signal.

1. Obtén un número que pueda recibir SMS (o verificación por voz para líneas fijas).
   - Usa un número de bot dedicado para evitar conflictos de cuenta/sesión.
2. Instala `signal-cli` en el host del gateway:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

Si usas la compilación JVM (`signal-cli-${VERSION}.tar.gz`), instala primero JRE 25+.
Mantén `signal-cli` actualizado; upstream indica que las versiones antiguas pueden dejar de funcionar a medida que cambian las API del servidor de Signal.

3. Registra y verifica el número:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

Si se requiere captcha:

1. Abre `https://signalcaptchas.org/registration/generate.html`.
2. Completa el captcha y copia el destino del enlace `signalcaptcha://...` de "Open Signal".
3. Ejecuta desde la misma IP externa que la sesión del navegador cuando sea posible.
4. Ejecuta el registro de nuevo inmediatamente (los tokens de captcha caducan rápido):

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. Configura OpenClaw, reinicia el gateway y verifica el canal:

```bash
# Si ejecutas el gateway como un servicio systemd de usuario:
systemctl --user restart openclaw-gateway.service

# Luego verifica:
openclaw doctor
openclaw channels status --probe
```

5. Empareja el remitente de tus mensajes directos:
   - Envía cualquier mensaje al número del bot.
   - Aprueba el código en el servidor: `openclaw pairing approve signal <PAIRING_CODE>`.
   - Guarda el número del bot como contacto en tu teléfono para evitar "Unknown contact".

Importante: registrar una cuenta de número de teléfono con `signal-cli` puede desautenticar la sesión principal de la app de Signal para ese número. Prefiere un número de bot dedicado, o usa el modo de vinculación por QR si necesitas conservar tu configuración actual de la app del teléfono.

Referencias de upstream:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- Flujo de captcha: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- Flujo de vinculación: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## Modo de daemon externo (httpUrl)

Si quieres gestionar `signal-cli` tú mismo (arranques en frío lentos de JVM, inicialización de contenedor o CPU compartidas), ejecuta el daemon por separado y apunta OpenClaw hacia él:

```json5
{
  channels: {
    signal: {
      httpUrl: "http://127.0.0.1:8080",
      autoStart: false,
    },
  },
}
```

Esto omite el inicio automático y la espera de arranque dentro de OpenClaw. Para arranques lentos cuando se inicia automáticamente, configura `channels.signal.startupTimeoutMs`.

## Control de acceso (mensajes directos + grupos)

Mensajes directos:

- Predeterminado: `channels.signal.dmPolicy = "pairing"`.
- Los remitentes desconocidos reciben un código de pairing; los mensajes se ignoran hasta su aprobación (los códigos caducan después de 1 hora).
- Aprobar mediante:
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal <CODE>`
- El pairing es el intercambio de tokens predeterminado para los mensajes directos de Signal. Detalles: [Pairing](/channels/pairing)
- Los remitentes solo con UUID (de `sourceUuid`) se almacenan como `uuid:<id>` en `channels.signal.allowFrom`.

Grupos:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom` controla quién puede activar acciones en grupos cuando `allowlist` está configurado.
- `channels.signal.groups["<group-id>" | "*"]` puede sobrescribir el comportamiento del grupo con `requireMention`, `tools` y `toolsBySender`.
- Usa `channels.signal.accounts.<id>.groups` para sobrescrituras por cuenta en configuraciones con varias cuentas.
- Nota de tiempo de ejecución: si `channels.signal` falta por completo, el tiempo de ejecución usa `groupPolicy="allowlist"` como respaldo para las comprobaciones de grupos (incluso si `channels.defaults.groupPolicy` está configurado).

## Cómo funciona (comportamiento)

- `signal-cli` se ejecuta como daemon; el gateway lee eventos mediante SSE.
- Los mensajes entrantes se normalizan en el sobre compartido del canal.
- Las respuestas siempre se enrutan de vuelta al mismo número o grupo.

## Medios + límites

- El texto saliente se fragmenta según `channels.signal.textChunkLimit` (predeterminado 4000).
- Fragmentación opcional por líneas nuevas: configura `channels.signal.chunkMode="newline"` para dividir en líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- Se admiten adjuntos (base64 obtenido desde `signal-cli`).
- Límite de medios predeterminado: `channels.signal.mediaMaxMb` (predeterminado 8).
- Usa `channels.signal.ignoreAttachments` para omitir la descarga de medios.
- El contexto del historial de grupos usa `channels.signal.historyLimit` (o `channels.signal.accounts.*.historyLimit`), con respaldo en `messages.groupChat.historyLimit`. Establece `0` para desactivar (predeterminado 50).

## Indicadores de escritura + confirmaciones de lectura

- **Indicadores de escritura**: OpenClaw envía señales de escritura mediante `signal-cli sendTyping` y las actualiza mientras se está generando una respuesta.
- **Confirmaciones de lectura**: cuando `channels.signal.sendReadReceipts` es `true`, OpenClaw reenvía las confirmaciones de lectura para los mensajes directos permitidos.
- Signal-cli no expone confirmaciones de lectura para grupos.

## Reacciones (herramienta de mensajes)

- Usa `message action=react` con `channel=signal`.
- Destinos: E.164 del remitente o UUID (usa `uuid:<id>` desde la salida de pairing; el UUID sin prefijo también funciona).
- `messageId` es la marca de tiempo de Signal del mensaje al que estás reaccionando.
- Las reacciones en grupos requieren `targetAuthor` o `targetAuthorUuid`.

Ejemplos:

```
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

Configuración:

- `channels.signal.actions.reactions`: habilitar/deshabilitar acciones de reacción (predeterminado true).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive`.
  - `off`/`ack` deshabilita las reacciones del agente (la herramienta de mensajes `react` devolverá un error).
  - `minimal`/`extensive` habilita las reacciones del agente y establece el nivel de guía.
- Sobrescrituras por cuenta: `channels.signal.accounts.<id>.actions.reactions`, `channels.signal.accounts.<id>.reactionLevel`.

## Destinos de entrega (CLI/cron)

- Mensajes directos: `signal:+15551234567` (o E.164 sin formato adicional).
- Mensajes directos por UUID: `uuid:<id>` (o UUID sin prefijo).
- Grupos: `signal:group:<groupId>`.
- Nombres de usuario: `username:<name>` (si tu cuenta de Signal lo admite).

## Solución de problemas

Ejecuta primero esta secuencia:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Luego confirma el estado de pairing del mensaje directo si es necesario:

```bash
openclaw pairing list signal
```

Fallos comunes:

- El daemon es accesible pero no hay respuestas: verifica la configuración de la cuenta/daemon (`httpUrl`, `account`) y el modo de recepción.
- Se ignoran los mensajes directos: el remitente está pendiente de aprobación de pairing.
- Se ignoran los mensajes de grupo: el control por remitente/mención del grupo bloquea la entrega.
- Errores de validación de configuración después de editar: ejecuta `openclaw doctor --fix`.
- Signal no aparece en el diagnóstico: confirma `channels.signal.enabled: true`.

Comprobaciones adicionales:

```bash
openclaw pairing list signal
pgrep -af signal-cli
grep -i "signal" "/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log" | tail -20
```

Para el flujo de triaje: [/channels/troubleshooting](/channels/troubleshooting).

## Notas de seguridad

- `signal-cli` almacena las claves de la cuenta localmente (normalmente en `~/.local/share/signal-cli/data/`).
- Haz una copia de seguridad del estado de la cuenta de Signal antes de migrar o reconstruir el servidor.
- Mantén `channels.signal.dmPolicy: "pairing"` salvo que quieras explícitamente un acceso más amplio a los mensajes directos.
- La verificación por SMS solo se necesita para flujos de registro o recuperación, pero perder el control del número/cuenta puede complicar el nuevo registro.

## Referencia de configuración (Signal)

Configuración completa: [Configuration](/gateway/configuration)

Opciones del proveedor:

- `channels.signal.enabled`: habilitar/deshabilitar el inicio del canal.
- `channels.signal.account`: E.164 para la cuenta del bot.
- `channels.signal.cliPath`: ruta a `signal-cli`.
- `channels.signal.httpUrl`: URL completa del daemon (sobrescribe host/puerto).
- `channels.signal.httpHost`, `channels.signal.httpPort`: enlace del daemon (predeterminado 127.0.0.1:8080).
- `channels.signal.autoStart`: iniciar automáticamente el daemon (predeterminado true si `httpUrl` no está configurado).
- `channels.signal.startupTimeoutMs`: tiempo de espera de arranque en ms (límite 120000).
- `channels.signal.receiveMode`: `on-start | manual`.
- `channels.signal.ignoreAttachments`: omitir descargas de adjuntos.
- `channels.signal.ignoreStories`: ignorar historias del daemon.
- `channels.signal.sendReadReceipts`: reenviar confirmaciones de lectura.
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (predeterminado: pairing).
- `channels.signal.allowFrom`: allowlist de mensajes directos (E.164 o `uuid:<id>`). `open` requiere `"*"`. Signal no tiene nombres de usuario; usa identificadores de teléfono/UUID.
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (predeterminado: allowlist).
- `channels.signal.groupAllowFrom`: allowlist de remitentes de grupo.
- `channels.signal.groups`: sobrescrituras por grupo indexadas por ID de grupo de Signal (o `"*"`). Campos admitidos: `requireMention`, `tools`, `toolsBySender`.
- `channels.signal.accounts.<id>.groups`: versión por cuenta de `channels.signal.groups` para configuraciones con varias cuentas.
- `channels.signal.historyLimit`: máximo de mensajes de grupo que se incluyen como contexto (0 desactiva).
- `channels.signal.dmHistoryLimit`: límite del historial de mensajes directos en turnos de usuario. Sobrescrituras por usuario: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: tamaño de fragmento saliente (caracteres).
- `channels.signal.chunkMode`: `length` (predeterminado) o `newline` para dividir en líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.signal.mediaMaxMb`: límite de medios entrantes/salientes (MB).

Opciones globales relacionadas:

- `agents.list[].groupChat.mentionPatterns` (Signal no admite menciones nativas).
- `messages.groupChat.mentionPatterns` (respaldo global).
- `messages.responsePrefix`.

## Relacionado

- [Channels Overview](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de pairing
- [Groups](/channels/groups) — comportamiento de chats grupales y control por menciones
- [Channel Routing](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Security](/gateway/security) — modelo de acceso y endurecimiento
