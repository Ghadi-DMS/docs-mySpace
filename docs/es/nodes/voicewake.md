---
read_when:
    - Cambiando el comportamiento o los valores predeterminados de las palabras de activación por voz
    - Agregando nuevas plataformas de nodo que necesiten sincronización de palabras de activación
summary: Palabras de activación de voz globales (gestionadas por el Gateway) y cómo se sincronizan entre nodos
title: Activación por voz
x-i18n:
    generated_at: "2026-04-05T12:47:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: a80e0cf7f68a3d48ff79af0ffb3058a7a0ecebd2cdbaad20b9ff53bc2b39dc84
    source_path: nodes/voicewake.md
    workflow: 15
---

# Activación por voz (palabras de activación globales)

OpenClaw trata las **palabras de activación** como una única lista global gestionada por el **Gateway**.

- **No hay palabras de activación personalizadas por nodo**.
- **Cualquier UI de nodo/app puede editar** la lista; los cambios se persisten en el Gateway y se difunden a todo el mundo.
- macOS e iOS mantienen interruptores locales de **Voice Wake activado/desactivado** (la UX local y los permisos son diferentes).
- Android actualmente mantiene Voice Wake desactivado y usa un flujo manual de micrófono en la pestaña Voice.

## Almacenamiento (host del Gateway)

Las palabras de activación se almacenan en la máquina del gateway en:

- `~/.openclaw/settings/voicewake.json`

Forma:

```json
{ "triggers": ["openclaw", "claude", "computer"], "updatedAtMs": 1730000000000 }
```

## Protocolo

### Métodos

- `voicewake.get` → `{ triggers: string[] }`
- `voicewake.set` con parámetros `{ triggers: string[] }` → `{ triggers: string[] }`

Notas:

- Los triggers se normalizan (se recortan espacios y se eliminan vacíos). Las listas vacías vuelven a los valores predeterminados.
- Se aplican límites por seguridad (topes de cantidad/longitud).

### Eventos

- Carga útil de `voicewake.changed`: `{ triggers: string[] }`

Quién lo recibe:

- Todos los clientes WebSocket (app de macOS, WebChat, etc.)
- Todos los nodos conectados (iOS/Android), y también al conectar un nodo como envío inicial del “estado actual”.

## Comportamiento del cliente

### App de macOS

- Usa la lista global para controlar los triggers de `VoiceWakeRuntime`.
- Editar “Trigger words” en la configuración de Voice Wake llama a `voicewake.set` y luego se apoya en la difusión para mantener sincronizados los demás clientes.

### Nodo iOS

- Usa la lista global para la detección de triggers de `VoiceWakeManager`.
- Editar Wake Words en Settings llama a `voicewake.set` (a través del WS del Gateway) y también mantiene reactiva la detección local de palabras de activación.

### Nodo Android

- Voice Wake está actualmente desactivado en el tiempo de ejecución/configuración de Android.
- La voz en Android usa captura manual de micrófono en la pestaña Voice en lugar de triggers por palabras de activación.
