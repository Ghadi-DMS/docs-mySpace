---
read_when:
    - Necesitas entender cómo se normalizan las marcas de tiempo para el modelo
    - Configurar la zona horaria del usuario para los prompts del sistema
summary: Manejo de zonas horarias para agentes, sobres y prompts
title: Zonas horarias
x-i18n:
    generated_at: "2026-04-05T12:40:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 31a195fa43e3fc17b788d8e70d74ef55da998fc7997c4f0538d4331b1260baac
    source_path: concepts/timezone.md
    workflow: 15
---

# Zonas horarias

OpenClaw estandariza las marcas de tiempo para que el modelo vea una **única hora de referencia**.

## Sobres de mensajes (local por defecto)

Los mensajes entrantes se encapsulan en un sobre como este:

```
[Provider ... 2026-01-05 16:26 PST] message text
```

La marca de tiempo del sobre es **local del host por defecto**, con precisión de minutos.

Puedes anular esto con:

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | zona horaria IANA
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

- `envelopeTimezone: "utc"` usa UTC.
- `envelopeTimezone: "user"` usa `agents.defaults.userTimezone` (recurre a la zona horaria del host).
- Usa una zona horaria IANA explícita (por ejemplo, `"Europe/Vienna"`) para un desplazamiento fijo.
- `envelopeTimestamp: "off"` elimina las marcas de tiempo absolutas de los encabezados del sobre.
- `envelopeElapsed: "off"` elimina los sufijos de tiempo transcurrido (del estilo `+2m`).

### Ejemplos

**Local (predeterminado):**

```
[Signal Alice +1555 2026-01-18 00:19 PST] hello
```

**Zona horaria fija:**

```
[Signal Alice +1555 2026-01-18 06:19 GMT+1] hello
```

**Tiempo transcurrido:**

```
[Signal Alice +1555 +2m 2026-01-18T05:19Z] follow-up
```

## Cargas útiles de herramientas (datos sin procesar del proveedor + campos normalizados)

Las llamadas de herramientas (`channels.discord.readMessages`, `channels.slack.readMessages`, etc.) devuelven **marcas de tiempo sin procesar del proveedor**.
También adjuntamos campos normalizados para mantener la coherencia:

- `timestampMs` (milisegundos de época UTC)
- `timestampUtc` (cadena UTC ISO 8601)

Los campos sin procesar del proveedor se conservan.

## Zona horaria del usuario para el prompt del sistema

Establece `agents.defaults.userTimezone` para indicar al modelo la zona horaria local de la persona usuaria. Si no está
definida, OpenClaw resuelve la **zona horaria del host en tiempo de ejecución** (sin escribir en la configuración).

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

El prompt del sistema incluye:

- Sección `Current Date & Time` con hora local y zona horaria
- `Time format: 12-hour` o `24-hour`

Puedes controlar el formato del prompt con `agents.defaults.timeFormat` (`auto` | `12` | `24`).

Consulta [Fecha y hora](/date-time) para ver el comportamiento completo y ejemplos.

## Relacionado

- [Heartbeat](/gateway/heartbeat) — las horas activas usan la zona horaria para la programación
- [Trabajos cron](/automation/cron-jobs) — las expresiones cron usan la zona horaria para la programación
- [Fecha y hora](/date-time) — comportamiento completo de fecha/hora y ejemplos
