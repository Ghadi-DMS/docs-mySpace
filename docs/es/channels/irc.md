---
read_when:
    - Quieres conectar OpenClaw a canales de IRC o mensajes directos
    - Estás configurando listas de permitidos de IRC, la política de grupos o la restricción por mención
summary: Configuración del plugin de IRC, controles de acceso y solución de problemas
title: IRC
x-i18n:
    generated_at: "2026-04-05T12:35:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: fceab2979db72116689c6c774d6736a8a2eee3559e3f3cf8969e673d317edd94
    source_path: channels/irc.md
    workflow: 15
---

# IRC

Usa IRC cuando quieras OpenClaw en canales clásicos (`#room`) y mensajes directos.
IRC se distribuye como un plugin de extensión, pero se configura en la configuración principal en `channels.irc`.

## Inicio rápido

1. Habilita la configuración de IRC en `~/.openclaw/openclaw.json`.
2. Configura al menos:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

Prefiere un servidor IRC privado para la coordinación del bot. Si usas intencionadamente una red IRC pública, algunas opciones habituales son Libera.Chat, OFTC y Snoonet. Evita canales públicos predecibles para el tráfico de canal secundario del bot o del enjambre.

3. Inicia o reinicia el gateway:

```bash
openclaw gateway run
```

## Valores predeterminados de seguridad

- `channels.irc.dmPolicy` tiene como valor predeterminado `"pairing"`.
- `channels.irc.groupPolicy` tiene como valor predeterminado `"allowlist"`.
- Con `groupPolicy="allowlist"`, configura `channels.irc.groups` para definir los canales permitidos.
- Usa TLS (`channels.irc.tls=true`) a menos que aceptes intencionadamente transporte en texto sin formato.

## Control de acceso

Hay dos “controles” separados para los canales de IRC:

1. **Acceso al canal** (`groupPolicy` + `groups`): si el bot acepta mensajes de un canal en absoluto.
2. **Acceso del remitente** (`groupAllowFrom` / `groups["#channel"].allowFrom` por canal): quién tiene permiso para activar el bot dentro de ese canal.

Claves de configuración:

- Lista de permitidos de mensajes directos (acceso del remitente en mensajes directos): `channels.irc.allowFrom`
- Lista de permitidos de remitentes de grupo (acceso del remitente en canales): `channels.irc.groupAllowFrom`
- Controles por canal (canal + remitente + reglas de mención): `channels.irc.groups["#channel"]`
- `channels.irc.groupPolicy="open"` permite canales sin configurar (**aun así, con restricción por mención de forma predeterminada**)

Las entradas de la lista de permitidos deben usar identidades de remitente estables (`nick!user@host`).
La coincidencia solo por nick es mutable y solo se habilita cuando `channels.irc.dangerouslyAllowNameMatching: true`.

### Error habitual: `allowFrom` es para mensajes directos, no para canales

Si ves registros como:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

…significa que el remitente no estaba permitido para mensajes de **grupo/canal**. Soluciónalo de una de estas formas:

- configurando `channels.irc.groupAllowFrom` (global para todos los canales), o
- configurando listas de permitidos de remitentes por canal: `channels.irc.groups["#channel"].allowFrom`

Ejemplo (permitir que cualquiera en `#tuirc-dev` hable con el bot):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": { allowFrom: ["*"] },
      },
    },
  },
}
```

## Activación de respuestas (menciones)

Aunque un canal esté permitido (mediante `groupPolicy` + `groups`) y el remitente esté permitido, OpenClaw usa de forma predeterminada **restricción por mención** en contextos de grupo.

Eso significa que puedes ver registros como `drop channel … (missing-mention)` a menos que el mensaje incluya un patrón de mención que coincida con el bot.

Para que el bot responda en un canal de IRC **sin necesitar una mención**, desactiva la restricción por mención para ese canal:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

O, para permitir **todos** los canales de IRC (sin lista de permitidos por canal) y seguir respondiendo sin menciones:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## Nota de seguridad (recomendada para canales públicos)

Si permites `allowFrom: ["*"]` en un canal público, cualquiera puede enviar indicaciones al bot.
Para reducir el riesgo, restringe las herramientas para ese canal.

### Las mismas herramientas para todos en el canal

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### Herramientas diferentes por remitente (el propietario obtiene más capacidades)

Usa `toolsBySender` para aplicar una política más estricta a `"*"` y una más flexible a tu nick:

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:eigen": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

Notas:

- Las claves de `toolsBySender` deben usar `id:` para los valores de identidad del remitente de IRC:
  `id:eigen` o `id:eigen!~eigen@174.127.248.171` para una coincidencia más fuerte.
- Las claves heredadas sin prefijo siguen aceptándose y se comparan solo como `id:`.
- La primera política de remitente coincidente es la que se aplica; `"*"` es el comodín de respaldo.

Para obtener más información sobre acceso de grupo frente a restricción por mención (y cómo interactúan), consulta: [/channels/groups](/channels/groups).

## NickServ

Para identificarte con NickServ después de conectar:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

Registro opcional único al conectar:

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

Desactiva `register` después de que el nick esté registrado para evitar intentos repetidos de REGISTER.

## Variables de entorno

La cuenta predeterminada admite:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (separados por comas)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

## Solución de problemas

- Si el bot se conecta pero nunca responde en canales, verifica `channels.irc.groups` **y** si la restricción por mención está descartando mensajes (`missing-mention`). Si quieres que responda sin menciones, configura `requireMention:false` para el canal.
- Si el inicio de sesión falla, verifica la disponibilidad del nick y la contraseña del servidor.
- Si TLS falla en una red personalizada, verifica el host/puerto y la configuración del certificado.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y restricción por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
