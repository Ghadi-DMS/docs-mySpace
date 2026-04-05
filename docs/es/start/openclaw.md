---
read_when:
    - Incorporar una nueva instancia de asistente
    - Revisar las implicaciones de seguridad y permisos
summary: Guía integral para ejecutar OpenClaw como asistente personal con advertencias de seguridad
title: Configuración de asistente personal
x-i18n:
    generated_at: "2026-04-05T12:54:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 02f10a9f7ec08f71143cbae996d91cbdaa19897a40f725d8ef524def41cf2759
    source_path: start/openclaw.md
    workflow: 15
---

# Crear un asistente personal con OpenClaw

OpenClaw es un gateway autoalojado que conecta Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo y más con agentes de IA. Esta guía cubre la configuración de "asistente personal": un número de WhatsApp dedicado que se comporta como tu asistente de IA siempre activo.

## ⚠️ Seguridad ante todo

Estás colocando a un agente en una posición en la que puede:

- ejecutar comandos en tu máquina (según tu política de herramientas)
- leer/escribir archivos en tu espacio de trabajo
- enviar mensajes de vuelta a través de WhatsApp/Telegram/Discord/Mattermost y otros canales integrados

Empieza de forma conservadora:

- Configura siempre `channels.whatsapp.allowFrom` (nunca ejecutes algo abierto al mundo en tu Mac personal).
- Usa un número de WhatsApp dedicado para el asistente.
- Los heartbeats ahora tienen como valor predeterminado cada 30 minutos. Desactívalos hasta que confíes en la configuración estableciendo `agents.defaults.heartbeat.every: "0m"`.

## Requisitos previos

- OpenClaw instalado e incorporado — consulta [Primeros pasos](/es/start/getting-started) si todavía no lo has hecho
- Un segundo número de teléfono (SIM/eSIM/prepago) para el asistente

## La configuración de dos teléfonos (recomendada)

Esto es lo que quieres:

```mermaid
flowchart TB
    A["<b>Tu teléfono (personal)<br></b><br>Tu WhatsApp<br>+1-555-YOU"] -- message --> B["<b>Segundo teléfono (asistente)<br></b><br>WhatsApp del asistente<br>+1-555-ASSIST"]
    B -- linked via QR --> C["<b>Tu Mac (openclaw)<br></b><br>Agente de IA"]
```

Si vinculas tu WhatsApp personal a OpenClaw, cada mensaje que recibas se convierte en “entrada del agente”. Rara vez es eso lo que quieres.

## Inicio rápido de 5 minutos

1. Empareja WhatsApp Web (muestra un QR; escanéalo con el teléfono del asistente):

```bash
openclaw channels login
```

2. Inicia el Gateway (déjalo en ejecución):

```bash
openclaw gateway --port 18789
```

3. Coloca una configuración mínima en `~/.openclaw/openclaw.json`:

```json5
{
  gateway: { mode: "local" },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

Ahora envía un mensaje al número del asistente desde tu teléfono en la lista permitida.

Cuando finaliza la incorporación, abrimos automáticamente el panel y mostramos un enlace limpio (sin token). Si solicita autenticación, pega el secreto compartido configurado en la configuración de la UI de control. La incorporación usa un token de forma predeterminada (`gateway.auth.token`), pero la autenticación por contraseña también funciona si cambiaste `gateway.auth.mode` a `password`. Para volver a abrirlo más tarde: `openclaw dashboard`.

## Dale al agente un espacio de trabajo (AGENTS)

OpenClaw lee instrucciones operativas y “memoria” desde su directorio de espacio de trabajo.

De forma predeterminada, OpenClaw usa `~/.openclaw/workspace` como espacio de trabajo del agente, y lo creará automáticamente (junto con `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` iniciales) durante la configuración o la primera ejecución del agente. `BOOTSTRAP.md` solo se crea cuando el espacio de trabajo es completamente nuevo (no debería volver a aparecer después de eliminarlo). `MEMORY.md` es opcional (no se crea automáticamente); cuando está presente, se carga para las sesiones normales. Las sesiones de subagentes solo inyectan `AGENTS.md` y `TOOLS.md`.

Consejo: trata esta carpeta como la “memoria” de OpenClaw y conviértela en un repositorio git (idealmente privado) para que tus archivos `AGENTS.md` + de memoria tengan copia de seguridad. Si git está instalado, los espacios de trabajo nuevos se inicializan automáticamente.

```bash
openclaw setup
```

Diseño completo del espacio de trabajo + guía de copia de seguridad: [Espacio de trabajo del agente](/es/concepts/agent-workspace)
Flujo de trabajo de memoria: [Memoria](/es/concepts/memory)

Opcional: elige un espacio de trabajo diferente con `agents.defaults.workspace` (admite `~`).

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

Si ya distribuyes tus propios archivos de espacio de trabajo desde un repositorio, puedes desactivar por completo la creación de archivos bootstrap:

```json5
{
  agent: {
    skipBootstrap: true,
  },
}
```

## La configuración que lo convierte en "un asistente"

OpenClaw usa de forma predeterminada una buena configuración de asistente, pero normalmente querrás ajustar lo siguiente:

- persona/instrucciones en [`SOUL.md`](/es/concepts/soul)
- valores predeterminados de thinking (si lo deseas)
- heartbeats (una vez que confíes en él)

Ejemplo:

```json5
{
  logging: { level: "info" },
  agent: {
    model: "anthropic/claude-opus-4-6",
    workspace: "~/.openclaw/workspace",
    thinkingDefault: "high",
    timeoutSeconds: 1800,
    // Comienza con 0; actívalo más tarde.
    heartbeat: { every: "0m" },
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  routing: {
    groupChat: {
      mentionPatterns: ["@openclaw", "openclaw"],
    },
  },
  session: {
    scope: "per-sender",
    resetTriggers: ["/new", "/reset"],
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080,
    },
  },
}
```

## Sesiones y memoria

- Archivos de sesión: `~/.openclaw/agents/<agentId>/sessions/{{SessionId}}.jsonl`
- Metadatos de sesión (uso de tokens, última ruta, etc.): `~/.openclaw/agents/<agentId>/sessions/sessions.json` (heredado: `~/.openclaw/sessions/sessions.json`)
- `/new` o `/reset` inicia una sesión nueva para ese chat (configurable mediante `resetTriggers`). Si se envía solo, el agente responde con un saludo breve para confirmar el restablecimiento.
- `/compact [instructions]` compacta el contexto de la sesión e informa el presupuesto de contexto restante.

## Heartbeats (modo proactivo)

De forma predeterminada, OpenClaw ejecuta un heartbeat cada 30 minutos con el prompt:
`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
Establece `agents.defaults.heartbeat.every: "0m"` para desactivarlo.

- Si `HEARTBEAT.md` existe pero está efectivamente vacío (solo líneas en blanco y encabezados markdown como `# Heading`), OpenClaw omite la ejecución del heartbeat para ahorrar llamadas a la API.
- Si falta el archivo, el heartbeat sigue ejecutándose y el modelo decide qué hacer.
- Si el agente responde con `HEARTBEAT_OK` (opcionalmente con un relleno breve; consulta `agents.defaults.heartbeat.ackMaxChars`), OpenClaw suprime la entrega saliente para ese heartbeat.
- De forma predeterminada, se permite la entrega de heartbeats a destinos de tipo DM `user:<id>`. Establece `agents.defaults.heartbeat.directPolicy: "block"` para suprimir la entrega a destinos directos mientras mantienes activas las ejecuciones del heartbeat.
- Los heartbeats ejecutan turnos completos del agente — los intervalos más cortos consumen más tokens.

```json5
{
  agent: {
    heartbeat: { every: "30m" },
  },
}
```

## Entrada y salida de medios

Los adjuntos entrantes (imágenes/audio/documentos) pueden mostrarse a tu comando mediante plantillas:

- `{{MediaPath}}` (ruta local del archivo temporal)
- `{{MediaUrl}}` (pseudo-URL)
- `{{Transcript}}` (si la transcripción de audio está habilitada)

Adjuntos salientes del agente: incluye `MEDIA:<path-or-url>` en su propia línea (sin espacios). Ejemplo:

```
Aquí está la captura de pantalla.
MEDIA:https://example.com/screenshot.png
```

OpenClaw los extrae y los envía como medios junto con el texto.

El comportamiento de rutas locales sigue el mismo modelo de confianza de lectura de archivos que el agente:

- Si `tools.fs.workspaceOnly` es `true`, las rutas locales salientes de `MEDIA:` siguen restringidas a la raíz temporal de OpenClaw, la caché de medios, las rutas del espacio de trabajo del agente y los archivos generados en sandbox.
- Si `tools.fs.workspaceOnly` es `false`, `MEDIA:` saliente puede usar archivos locales del host que el agente ya tiene permiso para leer.
- Los envíos locales del host siguen permitiendo solo medios y tipos de documentos seguros (imágenes, audio, video, PDF y documentos de Office). Los archivos de texto sin formato y los archivos que parezcan secretos no se tratan como medios enviables.

Eso significa que las imágenes/archivos generados fuera del espacio de trabajo ahora pueden enviarse cuando tu política de fs ya permite esas lecturas, sin volver a abrir la exfiltración arbitraria de adjuntos de texto del host.

## Lista de comprobación operativa

```bash
openclaw status          # estado local (credenciales, sesiones, eventos en cola)
openclaw status --all    # diagnóstico completo (solo lectura, apto para pegar)
openclaw status --deep   # pide al gateway una comprobación de estado en vivo con sondeos de canales cuando sea compatible
openclaw health --json   # instantánea de estado del gateway (WS; el valor predeterminado puede devolver una instantánea en caché reciente)
```

Los registros se encuentran en `/tmp/openclaw/` (predeterminado: `openclaw-YYYY-MM-DD.log`).

## Siguientes pasos

- WebChat: [WebChat](/web/webchat)
- Operaciones de Gateway: [Manual operativo de Gateway](/es/gateway)
- Cron + activaciones: [Trabajos cron](/es/automation/cron-jobs)
- Complemento de barra de menús de macOS: [App de OpenClaw para macOS](/es/platforms/macos)
- App de nodo iOS: [App de iOS](/es/platforms/ios)
- App de nodo Android: [App de Android](/es/platforms/android)
- Estado de Windows: [Windows (WSL2)](/es/platforms/windows)
- Estado de Linux: [App de Linux](/es/platforms/linux)
- Seguridad: [Seguridad](/es/gateway/security)
