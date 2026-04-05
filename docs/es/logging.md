---
read_when:
    - Necesitas un resumen del registro fácil de entender para principiantes
    - Quieres configurar niveles o formatos de registro
    - Estás solucionando problemas y necesitas encontrar registros rápidamente
summary: 'Resumen del registro: registros de archivos, salida de consola, seguimiento con la CLI y la UI de Control'
title: Resumen del registro
x-i18n:
    generated_at: "2026-04-05T12:47:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3a5e3800b7c5128602d05d5a35df4f88c373cfbe9397cca7e7154fff56a7f7ef
    source_path: logging.md
    workflow: 15
---

# Registro

OpenClaw tiene dos superficies principales de registro:

- **Registros de archivo** (líneas JSON) escritos por el Gateway.
- **Salida de consola** mostrada en terminales y en la UI de depuración del Gateway.

La pestaña **Logs** de la UI de Control sigue el registro de archivo del gateway. Esta página explica dónde
se encuentran los registros, cómo leerlos y cómo configurar niveles y formatos de registro.

## Dónde se encuentran los registros

De forma predeterminada, el Gateway escribe un archivo de registro rotativo en:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

La fecha usa la zona horaria local del host del gateway.

Puedes anular esto en `~/.openclaw/openclaw.json`:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## Cómo leer los registros

### CLI: seguimiento en vivo (recomendado)

Usa la CLI para seguir el archivo de registro del gateway mediante RPC:

```bash
openclaw logs --follow
```

Opciones útiles actuales:

- `--local-time`: mostrar las marcas de tiempo en tu zona horaria local
- `--url <url>` / `--token <token>` / `--timeout <ms>`: indicadores estándar de RPC del Gateway
- `--expect-final`: indicador de espera de respuesta final de RPC respaldada por agente (aceptado aquí mediante la capa de cliente compartida)

Modos de salida:

- **Sesiones TTY**: líneas de registro estructuradas, bonitas y con color.
- **Sesiones no TTY**: texto sin formato.
- `--json`: JSON delimitado por líneas (un evento de registro por línea).
- `--plain`: forzar texto sin formato en sesiones TTY.
- `--no-color`: deshabilitar colores ANSI.

Cuando pasas un `--url` explícito, la CLI no aplica automáticamente credenciales de configuración o
del entorno; incluye tú mismo `--token` si el Gateway de destino
requiere autenticación.

En modo JSON, la CLI emite objetos etiquetados por `type`:

- `meta`: metadatos del flujo (archivo, cursor, tamaño)
- `log`: entrada de registro analizada
- `notice`: indicaciones de truncamiento / rotación
- `raw`: línea de registro sin analizar

Si el Gateway de local loopback solicita emparejamiento, `openclaw logs` recurre automáticamente al
archivo de registro local configurado. Los destinos `--url` explícitos no
usan este respaldo.

Si el Gateway no es accesible, la CLI imprime una breve sugerencia para ejecutar:

```bash
openclaw doctor
```

### UI de Control (web)

La pestaña **Logs** de la UI de Control sigue el mismo archivo usando `logs.tail`.
Consulta [/web/control-ui](/web/control-ui) para saber cómo abrirla.

### Registros solo de canal

Para filtrar la actividad de canales (WhatsApp/Telegram/etc.), usa:

```bash
openclaw channels logs --channel whatsapp
```

## Formatos de registro

### Registros de archivo (JSONL)

Cada línea del archivo de registro es un objeto JSON. La CLI y la UI de Control analizan estas
entradas para renderizar una salida estructurada (hora, nivel, subsistema, mensaje).

### Salida de consola

Los registros de consola son **con reconocimiento de TTY** y están formateados para facilitar la lectura:

- Prefijos de subsistema (p. ej. `gateway/channels/whatsapp`)
- Coloreado por nivel (`info`/`warn`/`error`)
- Modo compacto o JSON opcional

El formato de la consola está controlado por `logging.consoleStyle`.

### Registros WebSocket del Gateway

`openclaw gateway` también tiene registro del protocolo WebSocket para tráfico RPC:

- modo normal: solo resultados interesantes (errores, errores de análisis, llamadas lentas)
- `--verbose`: todo el tráfico de solicitud/respuesta
- `--ws-log auto|compact|full`: elige el estilo de renderizado detallado
- `--compact`: alias de `--ws-log compact`

Ejemplos:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Configurar el registro

Toda la configuración de registro se encuentra bajo `logging` en `~/.openclaw/openclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/tmp/openclaw/openclaw-YYYY-MM-DD.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Niveles de registro

- `logging.level`: nivel de los **registros de archivo** (JSONL).
- `logging.consoleLevel`: nivel de verbosidad de la **consola**.

Puedes anular ambos mediante la variable de entorno **`OPENCLAW_LOG_LEVEL`** (p. ej. `OPENCLAW_LOG_LEVEL=debug`). La variable de entorno tiene prioridad sobre el archivo de configuración, por lo que puedes aumentar la verbosidad para una sola ejecución sin editar `openclaw.json`. También puedes pasar la opción global de la CLI **`--log-level <level>`** (por ejemplo, `openclaw --log-level debug gateway run`), que anula la variable de entorno para ese comando.

`--verbose` solo afecta a la salida de consola y a la verbosidad del registro WS; no cambia
los niveles del registro de archivo.

### Estilos de consola

`logging.consoleStyle`:

- `pretty`: fácil de leer para humanos, con color y marcas de tiempo.
- `compact`: salida más ajustada (mejor para sesiones largas).
- `json`: JSON por línea (para procesadores de registros).

### Redacción

Los resúmenes de herramientas pueden redactar tokens sensibles antes de que lleguen a la consola:

- `logging.redactSensitive`: `off` | `tools` (predeterminado: `tools`)
- `logging.redactPatterns`: lista de cadenas regex para anular el conjunto predeterminado

La redacción afecta **solo a la salida de consola** y no altera los registros de archivo.

## Diagnósticos + OpenTelemetry

Los diagnósticos son eventos estructurados y legibles por máquina para ejecuciones de modelos **y**
telemetría del flujo de mensajes (webhooks, colas, estado de sesión). **No**
sustituyen a los registros; existen para alimentar métricas, trazas y otros exportadores.

Los eventos de diagnóstico se emiten dentro del proceso, pero los exportadores solo se adjuntan cuando
los diagnósticos + el plugin exportador están habilitados.

### OpenTelemetry frente a OTLP

- **OpenTelemetry (OTel)**: el modelo de datos + SDK para trazas, métricas y registros.
- **OTLP**: el protocolo de transporte utilizado para exportar datos de OTel a un recolector/backend.
- OpenClaw exporta mediante **OTLP/HTTP (protobuf)** hoy en día.

### Señales exportadas

- **Métricas**: contadores + histogramas (uso de tokens, flujo de mensajes, colas).
- **Trazas**: spans para uso de modelos + procesamiento de webhook/mensajes.
- **Registros**: exportados por OTLP cuando `diagnostics.otel.logs` está habilitado. El
  volumen de registros puede ser alto; ten en cuenta `logging.level` y los filtros del exportador.

### Catálogo de eventos de diagnóstico

Uso del modelo:

- `model.usage`: tokens, coste, duración, contexto, proveedor/modelo/canal, IDs de sesión.

Flujo de mensajes:

- `webhook.received`: ingreso de webhook por canal.
- `webhook.processed`: webhook gestionado + duración.
- `webhook.error`: errores del controlador de webhook.
- `message.queued`: mensaje encolado para su procesamiento.
- `message.processed`: resultado + duración + error opcional.

Cola + sesión:

- `queue.lane.enqueue`: entrada en carril de cola de comandos + profundidad.
- `queue.lane.dequeue`: salida de carril de cola de comandos + tiempo de espera.
- `session.state`: transición de estado de sesión + motivo.
- `session.stuck`: advertencia de sesión atascada + antigüedad.
- `run.attempt`: metadatos de reintento/intento de ejecución.
- `diagnostic.heartbeat`: contadores agregados (webhooks/cola/sesión).

### Habilitar diagnósticos (sin exportador)

Usa esto si quieres que los eventos de diagnóstico estén disponibles para plugins o destinos personalizados:

```json
{
  "diagnostics": {
    "enabled": true
  }
}
```

### Indicadores de diagnóstico (registros dirigidos)

Usa indicadores para activar registros adicionales y dirigidos de depuración sin aumentar `logging.level`.
Los indicadores no distinguen mayúsculas de minúsculas y admiten comodines (p. ej. `telegram.*` o `*`).

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Anulación por entorno (puntual):

```
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

Notas:

- Los registros de indicadores van al archivo de registro estándar (el mismo que `logging.file`).
- La salida sigue redactándose según `logging.redactSensitive`.
- Guía completa: [/diagnostics/flags](/diagnostics/flags).

### Exportar a OpenTelemetry

Los diagnósticos pueden exportarse mediante el plugin `diagnostics-otel` (OTLP/HTTP). Esto
funciona con cualquier recolector/backend de OpenTelemetry que acepte OTLP/HTTP.

```json
{
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": {
        "enabled": true
      }
    }
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw-gateway",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 0.2,
      "flushIntervalMs": 60000
    }
  }
}
```

Notas:

- También puedes habilitar el plugin con `openclaw plugins enable diagnostics-otel`.
- `protocol` actualmente solo admite `http/protobuf`. `grpc` se ignora.
- Las métricas incluyen uso de tokens, coste, tamaño de contexto, duración de ejecución y
  contadores/histogramas de flujo de mensajes (webhooks, colas, estado de sesión, profundidad/espera de cola).
- Las trazas/métricas pueden activarse o desactivarse con `traces` / `metrics` (predeterminado: activado). Las trazas
  incluyen spans de uso de modelos más spans de procesamiento de webhook/mensajes cuando están habilitadas.
- Establece `headers` cuando tu recolector requiera autenticación.
- Variables de entorno compatibles: `OTEL_EXPORTER_OTLP_ENDPOINT`,
  `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_PROTOCOL`.

### Métricas exportadas (nombres + tipos)

Uso del modelo:

- `openclaw.tokens` (contador, atributos: `openclaw.token`, `openclaw.channel`,
  `openclaw.provider`, `openclaw.model`)
- `openclaw.cost.usd` (contador, atributos: `openclaw.channel`, `openclaw.provider`,
  `openclaw.model`)
- `openclaw.run.duration_ms` (histograma, atributos: `openclaw.channel`,
  `openclaw.provider`, `openclaw.model`)
- `openclaw.context.tokens` (histograma, atributos: `openclaw.context`,
  `openclaw.channel`, `openclaw.provider`, `openclaw.model`)

Flujo de mensajes:

- `openclaw.webhook.received` (contador, atributos: `openclaw.channel`,
  `openclaw.webhook`)
- `openclaw.webhook.error` (contador, atributos: `openclaw.channel`,
  `openclaw.webhook`)
- `openclaw.webhook.duration_ms` (histograma, atributos: `openclaw.channel`,
  `openclaw.webhook`)
- `openclaw.message.queued` (contador, atributos: `openclaw.channel`,
  `openclaw.source`)
- `openclaw.message.processed` (contador, atributos: `openclaw.channel`,
  `openclaw.outcome`)
- `openclaw.message.duration_ms` (histograma, atributos: `openclaw.channel`,
  `openclaw.outcome`)

Colas + sesiones:

- `openclaw.queue.lane.enqueue` (contador, atributos: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` (contador, atributos: `openclaw.lane`)
- `openclaw.queue.depth` (histograma, atributos: `openclaw.lane` o
  `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` (histograma, atributos: `openclaw.lane`)
- `openclaw.session.state` (contador, atributos: `openclaw.state`, `openclaw.reason`)
- `openclaw.session.stuck` (contador, atributos: `openclaw.state`)
- `openclaw.session.stuck_age_ms` (histograma, atributos: `openclaw.state`)
- `openclaw.run.attempt` (contador, atributos: `openclaw.attempt`)

### Spans exportados (nombres + atributos clave)

- `openclaw.model.usage`
  - `openclaw.channel`, `openclaw.provider`, `openclaw.model`
  - `openclaw.sessionKey`, `openclaw.sessionId`
  - `openclaw.tokens.*` (input/output/cache_read/cache_write/total)
- `openclaw.webhook.processed`
  - `openclaw.channel`, `openclaw.webhook`, `openclaw.chatId`
- `openclaw.webhook.error`
  - `openclaw.channel`, `openclaw.webhook`, `openclaw.chatId`,
    `openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`, `openclaw.outcome`, `openclaw.chatId`,
    `openclaw.messageId`, `openclaw.sessionKey`, `openclaw.sessionId`,
    `openclaw.reason`
- `openclaw.session.stuck`
  - `openclaw.state`, `openclaw.ageMs`, `openclaw.queueDepth`,
    `openclaw.sessionKey`, `openclaw.sessionId`

### Muestreo + vaciado

- Muestreo de trazas: `diagnostics.otel.sampleRate` (0.0–1.0, solo spans raíz).
- Intervalo de exportación de métricas: `diagnostics.otel.flushIntervalMs` (mín. 1000 ms).

### Notas del protocolo

- Los endpoints OTLP/HTTP pueden establecerse mediante `diagnostics.otel.endpoint` o
  `OTEL_EXPORTER_OTLP_ENDPOINT`.
- Si el endpoint ya contiene `/v1/traces` o `/v1/metrics`, se usa tal cual.
- Si el endpoint ya contiene `/v1/logs`, se usa tal cual para los registros.
- `diagnostics.otel.logs` habilita la exportación de registros OTLP para la salida del registrador principal.

### Comportamiento de exportación de registros

- Los registros OTLP usan los mismos registros estructurados que se escriben en `logging.file`.
- Respetan `logging.level` (nivel de registro de archivo). La redacción de consola **no** se aplica
  a los registros OTLP.
- Las instalaciones con alto volumen deberían preferir muestreo/filtrado en el recolector OTLP.

## Consejos para la solución de problemas

- **¿No se puede alcanzar el Gateway?** Ejecuta primero `openclaw doctor`.
- **¿Registros vacíos?** Comprueba que el Gateway esté en ejecución y escribiendo en la ruta de archivo
  de `logging.file`.
- **¿Necesitas más detalle?** Establece `logging.level` en `debug` o `trace` y vuelve a intentarlo.

## Relacionado

- [Internos del registro del Gateway](/gateway/logging) — estilos de registro WS, prefijos de subsistema y captura de consola
- [Diagnósticos](/gateway/configuration-reference#diagnostics) — exportación de OpenTelemetry y configuración de trazas de caché
