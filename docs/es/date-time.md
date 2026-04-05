---
read_when:
    - Vas a cambiar cómo se muestran las marcas de tiempo al modelo o a los usuarios
    - Estás depurando el formato de hora en mensajes o en la salida del system prompt
summary: Manejo de fecha y hora en sobres, prompts, herramientas y conectores
title: Fecha y hora
x-i18n:
    generated_at: "2026-04-05T12:41:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 753af5946a006215d6af2467fa478f3abb42b1dff027cf85d5dc4c7ba4b58d39
    source_path: date-time.md
    workflow: 15
---

# Fecha y hora

OpenClaw usa por defecto **la hora local del host para las marcas de tiempo de transporte** y **la zona horaria del usuario solo en el system prompt**.
Las marcas de tiempo del proveedor se conservan para que las herramientas mantengan su semántica nativa (la hora actual está disponible mediante `session_status`).

## Sobres de mensajes (local por defecto)

Los mensajes entrantes se envuelven con una marca de tiempo (precisión de minuto):

```
[Provider ... 2026-01-05 16:26 PST] texto del mensaje
```

Esta marca de tiempo del sobre es **local del host por defecto**, independientemente de la zona horaria del proveedor.

Puedes reemplazar este comportamiento:

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
- `envelopeTimezone: "local"` usa la zona horaria del host.
- `envelopeTimezone: "user"` usa `agents.defaults.userTimezone` (recurre a la zona horaria del host).
- Usa una zona horaria IANA explícita (por ejemplo, `"America/Chicago"`) para una zona fija.
- `envelopeTimestamp: "off"` elimina las marcas de tiempo absolutas de los encabezados de sobre.
- `envelopeElapsed: "off"` elimina los sufijos de tiempo transcurrido (del estilo `+2m`).

### Ejemplos

**Local (predeterminado):**

```
[WhatsApp +1555 2026-01-18 00:19 PST] hola
```

**Zona horaria del usuario:**

```
[WhatsApp +1555 2026-01-18 00:19 CST] hola
```

**Tiempo transcurrido habilitado:**

```
[WhatsApp +1555 +30s 2026-01-18T05:19Z] seguimiento
```

## System prompt: Current Date & Time

Si se conoce la zona horaria del usuario, el system prompt incluye una sección dedicada
**Current Date & Time** solo con la **zona horaria** (sin formato de reloj/hora)
para mantener estable la caché del prompt:

```
Time zone: America/Chicago
```

Cuando el agente necesite la hora actual, usa la herramienta `session_status`; la tarjeta
de estado incluye una línea de marca de tiempo.

## Líneas de eventos del sistema (local por defecto)

Los eventos del sistema en cola insertados en el contexto del agente llevan como prefijo una marca de tiempo usando la
misma selección de zona horaria que los sobres de mensajes (predeterminado: hora local del host).

```
System: [2026-01-12 12:19:17 PST] Modelo cambiado.
```

### Configurar zona horaria + formato del usuario

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
      timeFormat: "auto", // auto | 12 | 24
    },
  },
}
```

- `userTimezone` establece la **zona horaria local del usuario** para el contexto del prompt.
- `timeFormat` controla la visualización en **12h/24h** en el prompt. `auto` sigue las preferencias del SO.

## Detección del formato de hora (auto)

Cuando `timeFormat: "auto"`, OpenClaw inspecciona la preferencia del SO (macOS/Windows)
y recurre al formato de la configuración regional. El valor detectado se **almacena en caché por proceso**
para evitar llamadas repetidas al sistema.

## Cargas de herramientas + conectores (hora sin procesar del proveedor + campos normalizados)

Las herramientas de canal devuelven **marcas de tiempo nativas del proveedor** y agregan campos normalizados para mantener la coherencia:

- `timestampMs`: milisegundos epoch (UTC)
- `timestampUtc`: cadena UTC ISO 8601

Los campos sin procesar del proveedor se conservan para no perder nada.

- Slack: cadenas tipo epoch de la API
- Discord: marcas de tiempo ISO UTC
- Telegram/WhatsApp: marcas de tiempo numéricas/ISO específicas del proveedor

Si necesitas hora local, conviértela aguas abajo usando la zona horaria conocida.

## Documentación relacionada

- [System Prompt](/concepts/system-prompt)
- [Timezones](/concepts/timezone)
- [Messages](/concepts/messages)
