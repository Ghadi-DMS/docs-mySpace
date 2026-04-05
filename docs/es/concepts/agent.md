---
read_when:
    - Cambiar el tiempo de ejecución del agente, la inicialización del espacio de trabajo o el comportamiento de la sesión
summary: Tiempo de ejecución del agente, contrato del espacio de trabajo e inicialización de sesión
title: Tiempo de ejecución del agente
x-i18n:
    generated_at: "2026-04-05T12:39:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: e2ff39f4114f009e5b1f86894ea4bb29b1c9512563b70d063f09ca7cde5e8948
    source_path: concepts/agent.md
    workflow: 15
---

# Tiempo de ejecución del agente

OpenClaw ejecuta un único tiempo de ejecución de agente integrado.

## Espacio de trabajo (obligatorio)

OpenClaw usa un único directorio de espacio de trabajo del agente (`agents.defaults.workspace`) como el **único** directorio de trabajo (`cwd`) del agente para herramientas y contexto.

Recomendado: usa `openclaw setup` para crear `~/.openclaw/openclaw.json` si no existe e inicializar los archivos del espacio de trabajo.

Diseño completo del espacio de trabajo + guía de copias de seguridad: [Espacio de trabajo del agente](/concepts/agent-workspace)

Si `agents.defaults.sandbox` está habilitado, las sesiones que no son la principal pueden anular esto con
espacios de trabajo por sesión en `agents.defaults.sandbox.workspaceRoot` (consulta
[Configuración del gateway](/gateway/configuration)).

## Archivos de inicialización (inyectados)

Dentro de `agents.defaults.workspace`, OpenClaw espera estos archivos editables por el usuario:

- `AGENTS.md` — instrucciones operativas + “memoria”
- `SOUL.md` — personalidad, límites, tono
- `TOOLS.md` — notas de herramientas mantenidas por el usuario (por ejemplo, `imsg`, `sag`, convenciones)
- `BOOTSTRAP.md` — ritual único de la primera ejecución (se elimina al completarse)
- `IDENTITY.md` — nombre/estilo/emoji del agente
- `USER.md` — perfil del usuario + forma preferida de tratamiento

En el primer turno de una nueva sesión, OpenClaw inyecta directamente en el contexto del agente el contenido de estos archivos.

Los archivos vacíos se omiten. Los archivos grandes se recortan y truncan con un marcador para que los prompts sigan siendo ligeros (lee el archivo para ver el contenido completo).

Si falta un archivo, OpenClaw inyecta una sola línea marcadora de “archivo faltante” (y `openclaw setup` creará una plantilla predeterminada segura).

`BOOTSTRAP.md` solo se crea para un **espacio de trabajo completamente nuevo** (sin otros archivos de inicialización presentes). Si lo eliminas después de completar el ritual, no debería recrearse en reinicios posteriores.

Para desactivar por completo la creación de archivos de inicialización (para espacios de trabajo ya preparados), establece:

```json5
{ agent: { skipBootstrap: true } }
```

## Herramientas integradas

Las herramientas principales (read/exec/edit/write y herramientas del sistema relacionadas) siempre están disponibles,
sujetas a la política de herramientas. `apply_patch` es opcional y está controlada por
`tools.exec.applyPatch`. `TOOLS.md` **no** controla qué herramientas existen; es
una guía sobre cómo _quieres_ que se usen.

## Skills

OpenClaw carga Skills desde estas ubicaciones (de mayor precedencia a menor):

- Espacio de trabajo: `<workspace>/skills`
- Skills de agente del proyecto: `<workspace>/.agents/skills`
- Skills de agente personales: `~/.agents/skills`
- Gestionadas/locales: `~/.openclaw/skills`
- Incluidas (entregadas con la instalación)
- Directorios adicionales de Skills: `skills.load.extraDirs`

Las Skills pueden estar controladas por config/env (consulta `skills` en [Configuración del gateway](/gateway/configuration)).

## Límites del tiempo de ejecución

El tiempo de ejecución del agente integrado está construido sobre el núcleo del agente Pi (modelos, herramientas y
canalización de prompts). La gestión de sesiones, el descubrimiento, la conexión de herramientas y la
entrega por canal son capas propiedad de OpenClaw sobre ese núcleo.

## Sesiones

Las transcripciones de sesiones se almacenan como JSONL en:

- `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

El ID de sesión es estable y lo elige OpenClaw.
No se leen carpetas de sesión heredadas de otras herramientas.

## Dirección durante el streaming

Cuando el modo de cola es `steer`, los mensajes entrantes se inyectan en la ejecución actual.
La dirección en cola se entrega **después de que el turno actual del asistente termine
de ejecutar sus llamadas de herramienta**, antes de la siguiente llamada al LLM. La dirección ya no omite
las llamadas de herramienta restantes del mensaje actual del asistente; en su lugar, inyecta el mensaje
en cola en el siguiente límite del modelo.

Cuando el modo de cola es `followup` o `collect`, los mensajes entrantes se retienen hasta que
termina el turno actual y luego comienza un nuevo turno del agente con las cargas útiles en cola. Consulta
[Cola](/concepts/queue) para ver el comportamiento de modo + debounce/límite.

El streaming por bloques envía los bloques completados del asistente en cuanto terminan; está
**desactivado de forma predeterminada** (`agents.defaults.blockStreamingDefault: "off"`).
Ajusta el límite mediante `agents.defaults.blockStreamingBreak` (`text_end` frente a `message_end`; el valor predeterminado es text_end).
Controla la fragmentación flexible de bloques con `agents.defaults.blockStreamingChunk` (valor predeterminado:
800–1200 caracteres; prefiere saltos de párrafo, luego nuevas líneas; las oraciones al final).
Combina fragmentos en streaming con `agents.defaults.blockStreamingCoalesce` para reducir
el spam de líneas individuales (fusión basada en inactividad antes del envío). Los canales que no son Telegram requieren
`*.blockStreaming: true` explícito para habilitar respuestas por bloques.
Los resúmenes detallados de herramientas se emiten al inicio de la herramienta (sin debounce); la IU de control
transmite la salida de herramientas mediante eventos del agente cuando están disponibles.
Más detalles: [Streaming + fragmentación](/concepts/streaming).

## Referencias de modelo

Las referencias de modelo en la configuración (por ejemplo, `agents.defaults.model` y `agents.defaults.models`) se analizan dividiendo en la **primera** `/`.

- Usa `provider/model` al configurar modelos.
- Si el propio ID del modelo contiene `/` (estilo OpenRouter), incluye el prefijo del proveedor (ejemplo: `openrouter/moonshotai/kimi-k2`).
- Si omites el proveedor, OpenClaw primero intenta un alias, luego una coincidencia única
  de proveedor configurado para ese ID exacto de modelo, y solo después recurre al proveedor predeterminado configurado. Si ese proveedor ya no expone el
  modelo predeterminado configurado, OpenClaw recurre al primer
  proveedor/modelo configurado en lugar de mostrar un valor predeterminado obsoleto de un proveedor eliminado.

## Configuración (mínima)

Como mínimo, establece:

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom` (muy recomendado)

---

_Siguiente: [Chats de grupo](/channels/group-messages)_ 🦞
