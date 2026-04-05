---
read_when:
    - Quieres una guía sencilla para principiantes sobre la TUI
    - Necesitas la lista completa de funciones, comandos y atajos de la TUI
summary: 'Interfaz de terminal (TUI): conéctate al Gateway desde cualquier máquina'
title: TUI
x-i18n:
    generated_at: "2026-04-05T12:57:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: a73f70d65ecc7bff663e8df28c07d70d2920d4732fbb8288c137d65b8653ac52
    source_path: web/tui.md
    workflow: 15
---

# TUI (interfaz de terminal)

## Inicio rápido

1. Inicia el Gateway.

```bash
openclaw gateway
```

2. Abre la TUI.

```bash
openclaw tui
```

3. Escribe un mensaje y presiona Enter.

Gateway remoto:

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

Usa `--password` si tu Gateway usa autenticación por contraseña.

## Lo que ves

- Encabezado: URL de conexión, agente actual, sesión actual.
- Registro del chat: mensajes del usuario, respuestas del asistente, avisos del sistema, tarjetas de herramientas.
- Línea de estado: estado de conexión/ejecución (connecting, running, streaming, idle, error).
- Pie de página: estado de conexión + agente + sesión + modelo + think/fast/verbose/reasoning + recuentos de tokens + deliver.
- Entrada: editor de texto con autocompletado.

## Modelo mental: agentes + sesiones

- Los agentes son slugs únicos (por ejemplo, `main`, `research`). El Gateway expone la lista.
- Las sesiones pertenecen al agente actual.
- Las claves de sesión se almacenan como `agent:<agentId>:<sessionKey>`.
  - Si escribes `/session main`, la TUI lo expande a `agent:<currentAgent>:main`.
  - Si escribes `/session agent:other:main`, cambias explícitamente a esa sesión de agente.
- Alcance de la sesión:
  - `per-sender` (predeterminado): cada agente tiene muchas sesiones.
  - `global`: la TUI siempre usa la sesión `global` (el selector puede estar vacío).
- El agente actual + la sesión actual siempre son visibles en el pie de página.

## Envío + entrega

- Los mensajes se envían al Gateway; la entrega a proveedores está desactivada de forma predeterminada.
- Activa la entrega:
  - `/deliver on`
  - o el panel de Configuración
  - o inicia con `openclaw tui --deliver`

## Selectores + superposiciones

- Selector de modelo: enumera los modelos disponibles y establece la anulación de la sesión.
- Selector de agente: elige un agente diferente.
- Selector de sesión: muestra solo las sesiones del agente actual.
- Configuración: alterna entrega, expansión de salida de herramientas y visibilidad de thinking.

## Atajos de teclado

- Enter: enviar mensaje
- Esc: abortar la ejecución activa
- Ctrl+C: borrar la entrada (presiona dos veces para salir)
- Ctrl+D: salir
- Ctrl+L: selector de modelo
- Ctrl+G: selector de agente
- Ctrl+P: selector de sesión
- Ctrl+O: alternar expansión de salida de herramientas
- Ctrl+T: alternar visibilidad de thinking (recarga el historial)

## Comandos slash

Principales:

- `/help`
- `/status`
- `/agent <id>` (o `/agents`)
- `/session <key>` (o `/sessions`)
- `/model <provider/model>` (o `/models`)

Controles de sesión:

- `/think <off|minimal|low|medium|high>`
- `/fast <status|on|off>`
- `/verbose <on|full|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full>`
- `/elevated <on|off|ask|full>` (alias: `/elev`)
- `/activation <mention|always>`
- `/deliver <on|off>`

Ciclo de vida de la sesión:

- `/new` o `/reset` (restablece la sesión)
- `/abort` (aborta la ejecución activa)
- `/settings`
- `/exit`

Otros comandos slash del Gateway (por ejemplo, `/context`) se reenvían al Gateway y se muestran como salida del sistema. Consulta [Comandos slash](/tools/slash-commands).

## Comandos de shell locales

- Antepon `!` a una línea para ejecutar un comando de shell local en el host de la TUI.
- La TUI solicita una vez por sesión permitir la ejecución local; si la rechazas, `!` sigue deshabilitado para la sesión.
- Los comandos se ejecutan en un shell nuevo y no interactivo en el directorio de trabajo de la TUI (sin `cd`/env persistente).
- Los comandos de shell locales reciben `OPENCLAW_SHELL=tui-local` en su entorno.
- Un `!` solo se envía como un mensaje normal; los espacios iniciales no activan la ejecución local.

## Salida de herramientas

- Las llamadas a herramientas se muestran como tarjetas con args + resultados.
- Ctrl+O alterna entre vistas contraídas/expandidas.
- Mientras las herramientas se ejecutan, las actualizaciones parciales se transmiten en la misma tarjeta.

## Colores del terminal

- La TUI mantiene el texto del cuerpo del asistente en el color de primer plano predeterminado de tu terminal para que tanto los terminales oscuros como los claros sigan siendo legibles.
- Si tu terminal usa un fondo claro y la detección automática es incorrecta, configura `OPENCLAW_THEME=light` antes de iniciar `openclaw tui`.
- Para forzar en su lugar la paleta oscura original, configura `OPENCLAW_THEME=dark`.

## Historial + streaming

- Al conectarse, la TUI carga el historial más reciente (200 mensajes de forma predeterminada).
- Las respuestas en streaming se actualizan en el lugar hasta finalizar.
- La TUI también escucha eventos de herramientas del agente para ofrecer tarjetas de herramientas más completas.

## Detalles de conexión

- La TUI se registra en el Gateway como `mode: "tui"`.
- Las reconexiones muestran un mensaje del sistema; las lagunas de eventos se reflejan en el registro.

## Opciones

- `--url <url>`: URL de WebSocket del Gateway (usa por defecto la configuración o `ws://127.0.0.1:<port>`)
- `--token <token>`: token del Gateway (si es obligatorio)
- `--password <password>`: contraseña del Gateway (si es obligatoria)
- `--session <key>`: clave de sesión (predeterminada: `main`, o `global` cuando el alcance es global)
- `--deliver`: entrega las respuestas del asistente al proveedor (desactivado de forma predeterminada)
- `--thinking <level>`: anula el nivel de thinking para los envíos
- `--message <text>`: envía un mensaje inicial después de conectarse
- `--timeout-ms <ms>`: tiempo de espera del agente en ms (usa por defecto `agents.defaults.timeoutSeconds`)
- `--history-limit <n>`: entradas de historial que se cargarán (predeterminado `200`)

Nota: cuando configuras `--url`, la TUI no recurre a las credenciales de configuración o entorno.
Pasa `--token` o `--password` explícitamente. La falta de credenciales explícitas es un error.

## Solución de problemas

No hay salida después de enviar un mensaje:

- Ejecuta `/status` en la TUI para confirmar que el Gateway está conectado y en reposo/ocupado.
- Revisa los registros del Gateway: `openclaw logs --follow`.
- Confirma que el agente puede ejecutarse: `openclaw status` y `openclaw models status`.
- Si esperas mensajes en un canal de chat, habilita la entrega (`/deliver on` o `--deliver`).

## Solución de problemas de conexión

- `disconnected`: asegúrate de que el Gateway esté en ejecución y de que `--url/--token/--password` sean correctos.
- No hay agentes en el selector: revisa `openclaw agents list` y tu configuración de enrutamiento.
- Selector de sesión vacío: puede que estés en alcance global o que aún no tengas sesiones.

## Relacionado

- [Control UI](/web/control-ui) — interfaz de control basada en web
- [Referencia de la CLI](/cli) — referencia completa de comandos de la CLI
