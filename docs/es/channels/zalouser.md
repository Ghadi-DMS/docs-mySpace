---
read_when:
    - Configurar Zalo Personal para OpenClaw
    - Depurar el inicio de sesión o el flujo de mensajes de Zalo Personal
summary: Compatibilidad con cuentas personales de Zalo mediante `zca-js` nativo (inicio de sesión por QR), capacidades y configuración
title: Zalo Personal
x-i18n:
    generated_at: "2026-04-08T02:14:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 08f50edb2f4c6fe24972efe5e321f5fd0572c7d29af5c1db808151c7c943dc66
    source_path: channels/zalouser.md
    workflow: 15
---

# Zalo Personal (no oficial)

Estado: experimental. Esta integración automatiza una **cuenta personal de Zalo** mediante `zca-js` nativo dentro de OpenClaw.

> **Advertencia:** Esta es una integración no oficial y puede provocar la suspensión o el bloqueo de la cuenta. Úsela bajo su propia responsabilidad.

## Plugin incluido

Zalo Personal se incluye como un plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación separada.

Si usa una compilación anterior o una instalación personalizada que excluye Zalo Personal, instálelo manualmente:

- Instalar mediante CLI: `openclaw plugins install @openclaw/zalouser`
- O desde una extracción del código fuente: `openclaw plugins install ./path/to/local/zalouser-plugin`
- Detalles: [Plugins](/es/tools/plugin)

No se requiere ningún binario externo de CLI `zca`/`openzca`.

## Configuración rápida (principiante)

1. Asegúrese de que el plugin de Zalo Personal esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas o personalizadas pueden agregarlo manualmente con los comandos anteriores.
2. Inicie sesión (QR, en la máquina Gateway):
   - `openclaw channels login --channel zalouser`
   - Escanee el código QR con la aplicación móvil de Zalo.
3. Habilite el canal:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Reinicie el Gateway (o termine la configuración).
5. El acceso por DM usa `pairing` de forma predeterminada; apruebe el código de emparejamiento en el primer contacto.

## Qué es

- Se ejecuta completamente en proceso mediante `zca-js`.
- Usa detectores de eventos nativos para recibir mensajes entrantes.
- Envía respuestas directamente mediante la API de JS (texto/medios/enlace).
- Diseñado para casos de uso de “cuenta personal” donde la API de bots de Zalo no está disponible.

## Nombres

El id del canal es `zalouser` para dejar claro que esto automatiza una **cuenta personal de usuario de Zalo** (no oficial). Mantenemos `zalo` reservado para una posible integración futura con la API oficial de Zalo.

## Buscar IDs (directorio)

Use la CLI del directorio para descubrir contactos/grupos y sus IDs:

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Límites

- El texto saliente se divide en fragmentos de ~2000 caracteres (límites del cliente de Zalo).
- La transmisión en streaming está bloqueada de forma predeterminada.

## Control de acceso (DM)

`channels.zalouser.dmPolicy` admite: `pairing | allowlist | open | disabled` (predeterminado: `pairing`).

`channels.zalouser.allowFrom` acepta IDs o nombres de usuario. Durante la configuración, los nombres se resuelven a IDs usando la búsqueda de contactos en proceso del plugin.

Apruebe mediante:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Acceso a grupos (opcional)

- Predeterminado: `channels.zalouser.groupPolicy = "open"` (grupos permitidos). Use `channels.defaults.groupPolicy` para reemplazar el valor predeterminado cuando no esté establecido.
- Restrínjalo a una lista de permitidos con:
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups` (las claves deben ser IDs de grupo estables; los nombres se resuelven a IDs al iniciar cuando es posible)
  - `channels.zalouser.groupAllowFrom` (controla qué remitentes en los grupos permitidos pueden activar el bot)
- Bloquee todos los grupos: `channels.zalouser.groupPolicy = "disabled"`.
- El asistente de configuración puede solicitar listas de permitidos para grupos.
- Al iniciar, OpenClaw resuelve los nombres de grupo/usuario en las listas de permitidos a IDs y registra la asignación.
- La coincidencia de la lista de permitidos de grupos es solo por ID de forma predeterminada. Los nombres no resueltos se ignoran para la autenticación a menos que `channels.zalouser.dangerouslyAllowNameMatching: true` esté habilitado.
- `channels.zalouser.dangerouslyAllowNameMatching: true` es un modo de compatibilidad de emergencia que vuelve a habilitar la coincidencia por nombre de grupo mutable.
- Si `groupAllowFrom` no está establecido, el tiempo de ejecución usa `allowFrom` como alternativa para las comprobaciones de remitentes en grupos.
- Las comprobaciones de remitentes se aplican tanto a los mensajes normales de grupo como a los comandos de control (por ejemplo, `/new`, `/reset`).

Ejemplo:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { allow: true },
        "Work Chat": { allow: true },
      },
    },
  },
}
```

### Restricción de menciones en grupos

- `channels.zalouser.groups.<group>.requireMention` controla si las respuestas en grupos requieren una mención.
- Orden de resolución: id/nombre exacto del grupo -> slug normalizado del grupo -> `*` -> predeterminado (`true`).
- Esto se aplica tanto a los grupos en la lista de permitidos como al modo de grupo abierto.
- Citar un mensaje del bot cuenta como una mención implícita para la activación del grupo.
- Los comandos de control autorizados (por ejemplo, `/new`) pueden omitir la restricción de mención.
- Cuando se omite un mensaje de grupo porque se requiere mención, OpenClaw lo almacena como historial de grupo pendiente y lo incluye en el siguiente mensaje de grupo procesado.
- El límite del historial de grupo usa de forma predeterminada `messages.groupChat.historyLimit` (alternativa: `50`). Puede reemplazarlo por cuenta con `channels.zalouser.historyLimit`.

Ejemplo:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { allow: true, requireMention: true },
        "Work Chat": { allow: true, requireMention: false },
      },
    },
  },
}
```

## Varias cuentas

Las cuentas se asignan a perfiles `zalouser` en el estado de OpenClaw. Ejemplo:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## Escritura, reacciones y confirmaciones de entrega

- OpenClaw envía un evento de escritura antes de despachar una respuesta (mejor esfuerzo).
- La acción de reacción de mensaje `react` es compatible con `zalouser` en las acciones del canal.
  - Use `remove: true` para eliminar un emoji de reacción específico de un mensaje.
  - Semántica de reacciones: [Reacciones](/es/tools/reactions)
- Para los mensajes entrantes que incluyen metadatos de evento, OpenClaw envía confirmaciones de entregado + visto (mejor esfuerzo).

## Solución de problemas

**El inicio de sesión no se mantiene:**

- `openclaw channels status --probe`
- Vuelva a iniciar sesión: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**El nombre de la lista de permitidos/grupo no se resolvió:**

- Use IDs numéricos en `allowFrom`/`groupAllowFrom`/`groups`, o nombres exactos de amigos/grupos.

**Actualizó desde una configuración antigua basada en CLI:**

- Elimine cualquier suposición anterior sobre procesos externos de `zca`.
- El canal ahora se ejecuta completamente dentro de OpenClaw sin binarios externos de CLI.

## Relacionado

- [Descripción general de los canales](/es/channels) — todos los canales compatibles
- [Emparejamiento](/es/channels/pairing) — autenticación por DM y flujo de emparejamiento
- [Grupos](/es/channels/groups) — comportamiento del chat grupal y restricción de menciones
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) — modelo de acceso y refuerzo
