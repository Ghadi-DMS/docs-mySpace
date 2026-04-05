---
read_when:
    - Configuración de Zalo Personal para OpenClaw
    - Depuración del inicio de sesión o del flujo de mensajes de Zalo Personal
summary: Compatibilidad con cuentas personales de Zalo mediante `zca-js` nativo (inicio de sesión con QR), capacidades y configuración
title: Zalo Personal
x-i18n:
    generated_at: "2026-04-05T12:37:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 331b95041463185472d242cb0a944972f0a8e99df8120bda6350eca86ad5963f
    source_path: channels/zalouser.md
    workflow: 15
---

# Zalo Personal (no oficial)

Estado: experimental. Esta integración automatiza una **cuenta personal de Zalo** mediante `zca-js` nativo dentro de OpenClaw.

> **Advertencia:** Esta es una integración no oficial y puede provocar la suspensión o el bloqueo de la cuenta. Úsala bajo tu propia responsabilidad.

## Plugin empaquetado

Zalo Personal se incluye como plugin empaquetado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación independiente.

Si usas una compilación antigua o una instalación personalizada que excluye Zalo Personal,
instálalo manualmente:

- Instalar mediante la CLI: `openclaw plugins install @openclaw/zalouser`
- O desde una copia local del código fuente: `openclaw plugins install ./path/to/local/zalouser-plugin`
- Detalles: [Plugins](/tools/plugin)

No se requiere ningún binario externo de CLI `zca`/`openzca`.

## Configuración rápida (principiante)

1. Asegúrate de que el plugin de Zalo Personal esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Inicia sesión (QR, en la máquina del Gateway):
   - `openclaw channels login --channel zalouser`
   - Escanea el código QR con la aplicación móvil de Zalo.
3. Habilita el canal:

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

4. Reinicia el Gateway (o completa la configuración).
5. El acceso por mensajes directos usa `pairing` de forma predeterminada; aprueba el código de emparejamiento en el primer contacto.

## Qué es

- Se ejecuta completamente en proceso mediante `zca-js`.
- Usa escuchas de eventos nativas para recibir mensajes entrantes.
- Envía respuestas directamente mediante la API de JS (texto/multimedia/enlace).
- Diseñado para casos de uso de “cuenta personal” en los que la API oficial de bots de Zalo no está disponible.

## Nomenclatura

El id del canal es `zalouser` para dejar explícito que esto automatiza una **cuenta de usuario personal de Zalo** (no oficial). Conservamos `zalo` para una posible integración futura con la API oficial de Zalo.

## Encontrar IDs (directorio)

Usa la CLI de directorio para descubrir contactos/grupos y sus IDs:

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Límites

- El texto saliente se divide en fragmentos de ~2000 caracteres (límites del cliente de Zalo).
- El streaming está bloqueado de forma predeterminada.

## Control de acceso (mensajes directos)

`channels.zalouser.dmPolicy` admite: `pairing | allowlist | open | disabled` (predeterminado: `pairing`).

`channels.zalouser.allowFrom` acepta IDs de usuario o nombres. Durante la configuración, los nombres se resuelven a IDs mediante la búsqueda de contactos en proceso del plugin.

Aprueba mediante:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Acceso a grupos (opcional)

- Predeterminado: `channels.zalouser.groupPolicy = "open"` (grupos permitidos). Usa `channels.defaults.groupPolicy` para sobrescribir el valor predeterminado cuando no esté definido.
- Restringe a una lista de permitidos con:
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups` (las claves deben ser IDs estables de grupo; los nombres se resuelven a IDs al iniciar cuando es posible)
  - `channels.zalouser.groupAllowFrom` (controla qué remitentes en grupos permitidos pueden activar el bot)
- Bloquea todos los grupos: `channels.zalouser.groupPolicy = "disabled"`.
- El asistente de configuración puede solicitar listas de permitidos de grupos.
- Al iniciar, OpenClaw resuelve nombres de grupos/usuarios en las listas de permitidos a IDs y registra la asignación.
- La coincidencia de listas de permitidos de grupos es solo por ID de forma predeterminada. Los nombres no resueltos se ignoran para autorización a menos que `channels.zalouser.dangerouslyAllowNameMatching: true` esté habilitado.
- `channels.zalouser.dangerouslyAllowNameMatching: true` es un modo de compatibilidad de emergencia que vuelve a habilitar la coincidencia mutable por nombre de grupo.
- Si `groupAllowFrom` no está definido, el entorno de ejecución recurre a `allowFrom` para las comprobaciones de remitentes de grupo.
- Las comprobaciones de remitente se aplican tanto a mensajes normales de grupo como a comandos de control (por ejemplo, `/new`, `/reset`).

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

### Control por menciones en grupos

- `channels.zalouser.groups.<group>.requireMention` controla si las respuestas en grupos requieren una mención.
- Orden de resolución: id/nombre exacto de grupo -> slug normalizado del grupo -> `*` -> valor predeterminado (`true`).
- Esto se aplica tanto a grupos en lista de permitidos como al modo de grupos abiertos.
- Los comandos de control autorizados (por ejemplo, `/new`) pueden omitir el control por menciones.
- Cuando se omite un mensaje de grupo porque se requiere mención, OpenClaw lo almacena como historial de grupo pendiente y lo incluye en el siguiente mensaje de grupo procesado.
- El límite del historial de grupo usa `messages.groupChat.historyLimit` de forma predeterminada (respaldo `50`). Puedes sobrescribirlo por cuenta con `channels.zalouser.historyLimit`.

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
- La acción de reacción de mensaje `react` es compatible con `zalouser` en las acciones de canal.
  - Usa `remove: true` para eliminar un emoji de reacción específico de un mensaje.
  - Semántica de reacciones: [Reactions](/tools/reactions)
- Para los mensajes entrantes que incluyen metadatos de eventos, OpenClaw envía confirmaciones de entregado + visto (mejor esfuerzo).

## Resolución de problemas

**El inicio de sesión no se mantiene:**

- `openclaw channels status --probe`
- Vuelve a iniciar sesión: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**El nombre de la lista de permitidos/grupo no se resolvió:**

- Usa IDs numéricos en `allowFrom`/`groupAllowFrom`/`groups`, o nombres exactos de amigo/grupo.

**Actualizaste desde una configuración antigua basada en CLI:**

- Elimina cualquier suposición antigua sobre procesos externos `zca`.
- El canal ahora se ejecuta completamente dentro de OpenClaw sin binarios externos de CLI.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento de chats grupales y control por menciones
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y endurecimiento
