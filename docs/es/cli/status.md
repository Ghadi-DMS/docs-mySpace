---
read_when:
    - Quieres un diagnóstico rápido del estado de los canales y de los destinatarios recientes de la sesión
    - Quieres un estado completo “all” que se pueda pegar para depuración
summary: Referencia de CLI para `openclaw status` (diagnósticos, sondeos, instantáneas de uso)
title: status
x-i18n:
    generated_at: "2026-04-05T12:39:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: fbe9d94fbe9938cd946ee6f293b5bd3b464b75e1ade2eacdd851788c3bffe94e
    source_path: cli/status.md
    workflow: 15
---

# `openclaw status`

Diagnósticos para canales y sesiones.

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

Notas:

- `--deep` ejecuta sondeos en vivo (WhatsApp Web + Telegram + Discord + Slack + Signal).
- `--usage` imprime ventanas de uso normalizadas como `X% left`.
- Los campos sin procesar `usage_percent` / `usagePercent` de MiniMax representan la cuota restante, por lo que OpenClaw los invierte antes de mostrarlos; los campos basados en recuento tienen prioridad cuando están presentes. Las respuestas de `model_remains` prefieren la entrada del modelo de chat, derivan la etiqueta de ventana a partir de las marcas de tiempo cuando es necesario e incluyen el nombre del modelo en la etiqueta del plan.
- Cuando la instantánea de la sesión actual es escasa, `/status` puede completar contadores de tokens y caché a partir del registro de uso de la transcripción más reciente. Los valores activos no nulos existentes siguen teniendo prioridad sobre los valores de respaldo de la transcripción.
- El respaldo de la transcripción también puede recuperar la etiqueta activa del modelo de tiempo de ejecución cuando falta en la entrada de sesión en vivo. Si ese modelo de transcripción difiere del modelo seleccionado, status resuelve la ventana de contexto con el modelo de tiempo de ejecución recuperado en lugar del seleccionado.
- Para el cómputo del tamaño del prompt, el respaldo de la transcripción prefiere el total orientado a prompt más grande cuando faltan metadatos de sesión o son menores, para que las sesiones de proveedores personalizados no se reduzcan a pantallas de `0` tokens.
- La salida incluye almacenes de sesión por agente cuando hay varios agentes configurados.
- El resumen incluye el estado de instalación y ejecución del servicio del host del Gateway y del nodo cuando está disponible.
- El resumen incluye el canal de actualización y el SHA de git (para checkouts desde código fuente).
- La información de actualización aparece en el resumen; si hay una actualización disponible, status imprime una sugerencia para ejecutar `openclaw update` (consulta [Updating](/install/updating)).
- Las superficies de estado de solo lectura (`status`, `status --json`, `status --all`) resuelven SecretRefs compatibles para sus rutas de configuración de destino cuando es posible.
- Si se configura un SecretRef de canal compatible pero no está disponible en la ruta actual del comando, status sigue siendo de solo lectura e informa una salida degradada en lugar de fallar. La salida para personas muestra advertencias como “configured token unavailable in this command path”, y la salida JSON incluye `secretDiagnostics`.
- Cuando la resolución local de SecretRef del comando tiene éxito, status prefiere la instantánea resuelta y elimina de la salida final los marcadores transitorios de canal “secret unavailable”.
- `status --all` incluye una fila de resumen de secretos y una sección de diagnóstico que resume los diagnósticos de secretos (truncados para legibilidad) sin detener la generación del informe.
