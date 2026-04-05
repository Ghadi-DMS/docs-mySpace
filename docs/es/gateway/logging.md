---
read_when:
    - Cambias la salida o los formatos de registro
    - Depuras la salida de la CLI o del gateway
summary: Superficies de registro, logs de archivo, estilos de logs WS y formato de consola
title: Registro del Gateway
x-i18n:
    generated_at: "2026-04-05T12:42:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 465fe66ae6a3bc844e75d3898aed15b3371481c4fe89ede40e5a9377e19bb74c
    source_path: gateway/logging.md
    workflow: 15
---

# Registro

Para obtener un resumen orientado al usuario (CLI + Control UI + configuración), consulta [/logging](/logging).

OpenClaw tiene dos “superficies” de registro:

- **Salida de consola** (lo que ves en la terminal / UI de depuración).
- **Logs de archivo** (líneas JSON) escritos por el registrador del gateway.

## Registrador basado en archivos

- El archivo de log rotativo predeterminado está en `/tmp/openclaw/` (un archivo por día): `openclaw-YYYY-MM-DD.log`
  - La fecha usa la zona horaria local del host del gateway.
- La ruta y el nivel del archivo de log se pueden configurar mediante `~/.openclaw/openclaw.json`:
  - `logging.file`
  - `logging.level`

El formato del archivo es un objeto JSON por línea.

La pestaña Logs de la Control UI sigue este archivo a través del gateway (`logs.tail`).
La CLI puede hacer lo mismo:

```bash
openclaw logs --follow
```

**Verbose frente a niveles de registro**

- **Los logs de archivo** se controlan exclusivamente mediante `logging.level`.
- `--verbose` solo afecta a la **verbosidad de la consola** (y al estilo de los logs WS); **no**
  aumenta el nivel de los logs de archivo.
- Para capturar detalles exclusivos de verbose en los logs de archivo, configura `logging.level` en `debug` o
  `trace`.

## Captura de consola

La CLI captura `console.log/info/warn/error/debug/trace` y los escribe en los logs de archivo,
mientras sigue imprimiéndolos en stdout/stderr.

Puedes ajustar la verbosidad de la consola de forma independiente mediante:

- `logging.consoleLevel` (predeterminado `info`)
- `logging.consoleStyle` (`pretty` | `compact` | `json`)

## Redacción de resúmenes de herramientas

Los resúmenes detallados de herramientas (por ejemplo `🛠️ Exec: ...`) pueden ocultar tokens sensibles antes de que lleguen al
stream de consola. Esto es **solo para herramientas** y no altera los logs de archivo.

- `logging.redactSensitive`: `off` | `tools` (predeterminado: `tools`)
- `logging.redactPatterns`: arreglo de cadenas regex (anula los valores predeterminados)
  - Usa cadenas regex sin procesar (auto `gi`), o `/pattern/flags` si necesitas indicadores personalizados.
  - Las coincidencias se enmascaran manteniendo los primeros 6 + los últimos 4 caracteres (longitud >= 18), en caso contrario `***`.
  - Los valores predeterminados cubren asignaciones de claves comunes, indicadores de CLI, campos JSON, cabeceras bearer, bloques PEM y prefijos de tokens populares.

## Logs WebSocket del gateway

El gateway imprime logs del protocolo WebSocket en dos modos:

- **Modo normal (sin `--verbose`)**: solo se imprimen resultados RPC “interesantes”:
  - errores (`ok=false`)
  - llamadas lentas (umbral predeterminado: `>= 50ms`)
  - errores de análisis
- **Modo verbose (`--verbose`)**: imprime todo el tráfico de solicitudes y respuestas WS.

### Estilo de logs WS

`openclaw gateway` admite un selector de estilo por gateway:

- `--ws-log auto` (predeterminado): el modo normal está optimizado; el modo verbose usa salida compacta
- `--ws-log compact`: salida compacta (solicitud/respuesta emparejada) cuando está en verbose
- `--ws-log full`: salida completa por trama cuando está en verbose
- `--compact`: alias de `--ws-log compact`

Ejemplos:

```bash
# optimized (only errors/slow)
openclaw gateway

# show all WS traffic (paired)
openclaw gateway --verbose --ws-log compact

# show all WS traffic (full meta)
openclaw gateway --verbose --ws-log full
```

## Formato de consola (registro por subsistema)

El formateador de consola **detecta TTY** e imprime líneas coherentes con prefijos.
Los registradores por subsistema mantienen la salida agrupada y fácil de examinar.

Comportamiento:

- **Prefijos de subsistema** en cada línea (por ejemplo `[gateway]`, `[canvas]`, `[tailscale]`)
- **Colores de subsistema** (estables por subsistema) más coloración por nivel
- **Color cuando la salida es un TTY o el entorno parece una terminal enriquecida** (`TERM`/`COLORTERM`/`TERM_PROGRAM`), respeta `NO_COLOR`
- **Prefijos de subsistema abreviados**: elimina los prefijos iniciales `gateway/` + `channels/`, conserva los últimos 2 segmentos (por ejemplo `whatsapp/outbound`)
- **Subregistradores por subsistema** (prefijo automático + campo estructurado `{ subsystem }`)
- **`logRaw()`** para salida de QR/UX (sin prefijo, sin formato)
- **Estilos de consola** (por ejemplo `pretty | compact | json`)
- **Nivel de log de consola** separado del nivel de log de archivo (el archivo mantiene el detalle completo cuando `logging.level` está configurado en `debug`/`trace`)
- **Los cuerpos de mensajes de WhatsApp** se registran en `debug` (usa `--verbose` para verlos)

Esto mantiene estables los logs de archivo existentes mientras hace que la salida interactiva sea fácil de examinar.
