---
read_when:
    - Quieres añadir o eliminar cuentas de canal (WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Matrix)
    - Quieres comprobar el estado del canal o seguir los logs del canal
summary: Referencia de CLI para `openclaw channels` (cuentas, estado, inicio/cierre de sesión, logs)
title: channels
x-i18n:
    generated_at: "2026-04-05T12:37:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: d0f558fdb5f6ec54e7fdb7a88e5c24c9d2567174341bd3ea87848bce4cba5d29
    source_path: cli/channels.md
    workflow: 15
---

# `openclaw channels`

Gestiona las cuentas de canales de chat y su estado de ejecución en el Gateway.

Documentación relacionada:

- Guías de canales: [Channels](/channels/index)
- Configuración del Gateway: [Configuration](/gateway/configuration)

## Comandos comunes

```bash
openclaw channels list
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
```

## Estado / capacidades / resolución / logs

- `channels status`: `--probe`, `--timeout <ms>`, `--json`
- `channels capabilities`: `--channel <name>`, `--account <id>` (solo con `--channel`), `--target <dest>`, `--timeout <ms>`, `--json`
- `channels resolve`: `<entries...>`, `--channel <name>`, `--account <id>`, `--kind <auto|user|group>`, `--json`
- `channels logs`: `--channel <name|all>`, `--lines <n>`, `--json`

`channels status --probe` es la ruta en vivo: en un gateway accesible ejecuta comprobaciones
`probeAccount` por cuenta y, de forma opcional, `auditAccount`, por lo que la salida puede incluir el estado del transporte
más resultados de sondeo como `works`, `probe failed`, `audit ok` o `audit failed`.
Si no se puede acceder al gateway, `channels status` recurre a resúmenes basados solo en la configuración
en lugar de salida de sondeo en vivo.

## Añadir o eliminar cuentas

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

Consejo: `openclaw channels add --help` muestra indicadores específicos por canal (token, clave privada, token de app, rutas de signal-cli, etc.).

Algunas superficies comunes para añadir de forma no interactiva incluyen:

- canales con token de bot: `--token`, `--bot-token`, `--app-token`, `--token-file`
- campos de transporte de Signal/iMessage: `--signal-number`, `--cli-path`, `--http-url`, `--http-host`, `--http-port`, `--db-path`, `--service`, `--region`
- campos de Google Chat: `--webhook-path`, `--webhook-url`, `--audience-type`, `--audience`
- campos de Matrix: `--homeserver`, `--user-id`, `--access-token`, `--password`, `--device-name`, `--initial-sync-limit`
- campos de Nostr: `--private-key`, `--relay-urls`
- campos de Tlon: `--ship`, `--url`, `--code`, `--group-channels`, `--dm-allowlist`, `--auto-discover-channels`
- `--use-env` para autenticación respaldada por variables de entorno de la cuenta predeterminada cuando sea compatible

Cuando ejecutas `openclaw channels add` sin indicadores, el asistente interactivo puede solicitar:

- IDs de cuenta por cada canal seleccionado
- nombres para mostrar opcionales para esas cuentas
- `Bind configured channel accounts to agents now?`

Si confirmas vincular ahora, el asistente pregunta qué agente debe ser propietario de cada cuenta de canal configurada y escribe enlaces de enrutamiento con alcance de cuenta.

También puedes gestionar las mismas reglas de enrutamiento más adelante con `openclaw agents bindings`, `openclaw agents bind` y `openclaw agents unbind` (consulta [agents](/cli/agents)).

Cuando añades una cuenta no predeterminada a un canal que todavía usa ajustes de nivel superior para una sola cuenta, OpenClaw promociona los valores de nivel superior con alcance de cuenta al mapa de cuentas del canal antes de escribir la cuenta nueva. La mayoría de los canales colocan esos valores en `channels.<channel>.accounts.default`, pero los canales incluidos pueden conservar en su lugar una cuenta promocionada existente que coincida. Matrix es el ejemplo actual: si ya existe una cuenta con nombre, o si `defaultAccount` apunta a una cuenta con nombre existente, la promoción conserva esa cuenta en lugar de crear una nueva `accounts.default`.

El comportamiento de enrutamiento sigue siendo coherente:

- Los enlaces existentes solo por canal (sin `accountId`) siguen coincidiendo con la cuenta predeterminada.
- `channels add` no crea ni reescribe enlaces automáticamente en modo no interactivo.
- La configuración interactiva puede añadir opcionalmente enlaces con alcance de cuenta.

Si tu configuración ya estaba en un estado mixto (cuentas con nombre presentes y valores de nivel superior de una sola cuenta todavía configurados), ejecuta `openclaw doctor --fix` para mover los valores con alcance de cuenta a la cuenta promocionada elegida para ese canal. La mayoría de los canales promocionan en `accounts.default`; Matrix puede conservar un destino existente con nombre o predeterminado en su lugar.

## Iniciar o cerrar sesión (interactivo)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

Notas:

- `channels login` admite `--verbose`.
- `channels login` / `logout` pueden inferir el canal cuando solo hay un destino de inicio de sesión compatible configurado.

## Solución de problemas

- Ejecuta `openclaw status --deep` para un sondeo amplio.
- Usa `openclaw doctor` para correcciones guiadas.
- `openclaw channels list` imprime `Claude: HTTP 403 ... user:profile` → la instantánea de uso necesita el alcance `user:profile`. Usa `--no-usage`, o proporciona una clave de sesión de claude.ai (`CLAUDE_WEB_SESSION_KEY` / `CLAUDE_WEB_COOKIE`), o vuelve a autenticarte mediante Claude CLI.
- `openclaw channels status` recurre a resúmenes basados solo en la configuración cuando no se puede acceder al gateway. Si una credencial de canal compatible está configurada mediante SecretRef pero no está disponible en la ruta actual del comando, informa esa cuenta como configurada con notas degradadas en lugar de mostrarla como no configurada.

## Sondeo de capacidades

Obtén indicios de capacidades del proveedor (intenciones o alcances cuando estén disponibles) más compatibilidad estática de funciones:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

Notas:

- `--channel` es opcional; omítelo para listar todos los canales (incluidas las extensiones).
- `--account` solo es válido con `--channel`.
- `--target` acepta `channel:<id>` o un id numérico bruto de canal y solo se aplica a Discord.
- Los sondeos son específicos del proveedor: intenciones de Discord y permisos de canal opcionales; alcances de bot y usuario de Slack; indicadores del bot de Telegram y webhook; versión del daemon de Signal; token de app de Microsoft Teams y roles o alcances de Graph (anotados cuando se conocen). Los canales sin sondeos informan `Probe: unavailable`.

## Resolver nombres a IDs

Resuelve nombres de canal y usuario a IDs usando el directorio del proveedor:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

Notas:

- Usa `--kind user|group|auto` para forzar el tipo de destino.
- La resolución prioriza coincidencias activas cuando varias entradas comparten el mismo nombre.
- `channels resolve` es de solo lectura. Si una cuenta seleccionada está configurada mediante SecretRef pero esa credencial no está disponible en la ruta actual del comando, el comando devuelve resultados degradados sin resolver con notas en lugar de abortar toda la ejecución.
