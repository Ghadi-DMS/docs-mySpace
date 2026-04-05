---
read_when:
    - Añadir o modificar el comportamiento de exec en segundo plano
    - Depurar tareas exec de larga duración
summary: Ejecución en segundo plano con exec y gestión de procesos
title: Exec en segundo plano y herramienta de procesos
x-i18n:
    generated_at: "2026-04-05T12:41:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4398e2850f6f050944f103ad637cd9f578e9cc7fb478bc5cd5d972c92289b831
    source_path: gateway/background-process.md
    workflow: 15
---

# Exec en segundo plano + herramienta de procesos

OpenClaw ejecuta comandos de shell mediante la herramienta `exec` y mantiene las tareas de larga duración en memoria. La herramienta `process` gestiona esas sesiones en segundo plano.

## Herramienta exec

Parámetros clave:

- `command` (obligatorio)
- `yieldMs` (predeterminado 10000): pasa automáticamente a segundo plano después de este retraso
- `background` (bool): pasar a segundo plano inmediatamente
- `timeout` (segundos, predeterminado 1800): matar el proceso después de este tiempo de espera
- `elevated` (bool): ejecutar fuera del sandbox si el modo elevado está habilitado/permitido (`gateway` de forma predeterminada, o `node` cuando el destino de exec es `node`)
- ¿Necesitas un TTY real? Establece `pty: true`.
- `workdir`, `env`

Comportamiento:

- Las ejecuciones en primer plano devuelven la salida directamente.
- Cuando se ejecuta en segundo plano (explícitamente o por tiempo de espera), la herramienta devuelve `status: "running"` + `sessionId` y una cola breve.
- La salida se mantiene en memoria hasta que se sondea o se borra la sesión.
- Si la herramienta `process` no está permitida, `exec` se ejecuta de forma síncrona e ignora `yieldMs`/`background`.
- Los comandos exec generados reciben `OPENCLAW_SHELL=exec` para reglas de shell/perfil con reconocimiento de contexto.
- Para trabajo de larga duración que empieza ahora, inícialo una vez y confía en la activación automática
  por finalización cuando esté habilitada y el comando emita salida o falle.
- Si la activación automática por finalización no está disponible, o necesitas confirmación
  de éxito silencioso para un comando que terminó correctamente sin salida, usa `process`
  para confirmar la finalización.
- No emules recordatorios ni seguimientos diferidos con bucles `sleep` ni sondeo
  repetido; usa cron para trabajo futuro.

## Puente de procesos hijo

Al generar procesos hijo de larga duración fuera de las herramientas exec/process (por ejemplo, reapariciones de la CLI o utilidades del gateway), adjunta la utilidad de puente de procesos hijo para que las señales de terminación se reenvíen y los listeners se desacoplen al salir o producirse un error. Esto evita procesos huérfanos en systemd y mantiene un comportamiento de apagado coherente entre plataformas.

Anulaciones de entorno:

- `PI_BASH_YIELD_MS`: yield predeterminado (ms)
- `PI_BASH_MAX_OUTPUT_CHARS`: límite de salida en memoria (caracteres)
- `OPENCLAW_BASH_PENDING_MAX_OUTPUT_CHARS`: límite pendiente de stdout/stderr por flujo (caracteres)
- `PI_BASH_JOB_TTL_MS`: TTL para sesiones terminadas (ms, limitado entre 1 min y 3 h)

Configuración (preferida):

- `tools.exec.backgroundMs` (predeterminado 10000)
- `tools.exec.timeoutSec` (predeterminado 1800)
- `tools.exec.cleanupMs` (predeterminado 1800000)
- `tools.exec.notifyOnExit` (predeterminado true): encola un evento del sistema + solicita heartbeat cuando finaliza un exec en segundo plano.
- `tools.exec.notifyOnExitEmptySuccess` (predeterminado false): cuando es true, también encola eventos de finalización para ejecuciones en segundo plano exitosas que no produjeron salida.

## Herramienta process

Acciones:

- `list`: sesiones en ejecución + terminadas
- `poll`: drenar salida nueva de una sesión (también informa del estado de salida)
- `log`: leer la salida agregada (admite `offset` + `limit`)
- `write`: enviar stdin (`data`, `eof` opcional)
- `send-keys`: enviar tokens de teclas explícitos o bytes a una sesión respaldada por PTY
- `submit`: enviar Enter / retorno de carro a una sesión respaldada por PTY
- `paste`: enviar texto literal, opcionalmente envuelto en modo de pegado entre corchetes
- `kill`: terminar una sesión en segundo plano
- `clear`: eliminar una sesión terminada de la memoria
- `remove`: matar si está en ejecución; de lo contrario, borrar si ha terminado

Notas:

- Solo las sesiones en segundo plano se listan/se conservan en memoria.
- Las sesiones se pierden al reiniciar el proceso (sin persistencia en disco).
- Los registros de sesión solo se guardan en el historial de chat si ejecutas `process poll/log` y se registra el resultado de la herramienta.
- `process` tiene alcance por agente; solo ve sesiones iniciadas por ese agente.
- Usa `poll` / `log` para estado, registros, confirmación de éxito silencioso o
  confirmación de finalización cuando la activación automática por finalización no esté disponible.
- Usa `write` / `send-keys` / `submit` / `paste` / `kill` cuando necesites entrada
  o intervención.
- `process list` incluye un `name` derivado (verbo del comando + objetivo) para búsquedas rápidas.
- `process log` usa `offset`/`limit` por líneas.
- Cuando se omiten `offset` y `limit`, devuelve las últimas 200 líneas e incluye una indicación de paginación.
- Cuando se proporciona `offset` y se omite `limit`, devuelve desde `offset` hasta el final (sin limitarse a 200).
- El sondeo es para estado bajo demanda, no para programación en bucle de espera. Si el trabajo debe
  ocurrir más tarde, usa cron.

## Ejemplos

Ejecutar una tarea larga y sondear más tarde:

```json
{ "tool": "exec", "command": "sleep 5 && echo done", "yieldMs": 1000 }
```

```json
{ "tool": "process", "action": "poll", "sessionId": "<id>" }
```

Iniciar inmediatamente en segundo plano:

```json
{ "tool": "exec", "command": "npm run build", "background": true }
```

Enviar stdin:

```json
{ "tool": "process", "action": "write", "sessionId": "<id>", "data": "y\n" }
```

Enviar teclas PTY:

```json
{ "tool": "process", "action": "send-keys", "sessionId": "<id>", "keys": ["C-c"] }
```

Enviar la línea actual:

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Pegar texto literal:

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```
