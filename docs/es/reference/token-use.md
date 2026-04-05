---
read_when:
    - Explicar el uso de tokens, los costos o las ventanas de contexto
    - Depurar el crecimiento del contexto o el comportamiento de compactación
summary: Cómo OpenClaw construye el contexto del prompt y muestra el uso de tokens + costos
title: Uso de tokens y costos
x-i18n:
    generated_at: "2026-04-05T12:54:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 14e7a0ac0311298cf1484d663799a3f5a9687dd5afc9702233e983aba1979f1d
    source_path: reference/token-use.md
    workflow: 15
---

# Uso de tokens y costos

OpenClaw rastrea **tokens**, no caracteres. Los tokens son específicos del modelo, pero la mayoría
de los modelos de estilo OpenAI promedian ~4 caracteres por token en texto en inglés.

## Cómo se construye el prompt del sistema

OpenClaw ensambla su propio prompt del sistema en cada ejecución. Incluye:

- Lista de herramientas + descripciones breves
- Lista de Skills (solo metadatos; las instrucciones se cargan bajo demanda con `read`)
- Instrucciones de autoactualización
- Archivos del espacio de trabajo + bootstrap (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` cuando es nuevo, además de `MEMORY.md` cuando está presente o `memory.md` como alternativa en minúsculas). Los archivos grandes se truncan según `agents.defaults.bootstrapMaxChars` (predeterminado: 20000), y la inyección total de bootstrap está limitada por `agents.defaults.bootstrapTotalMaxChars` (predeterminado: 150000). Los archivos `memory/*.md` se usan bajo demanda mediante herramientas de memoria y no se inyectan automáticamente.
- Hora (UTC + zona horaria del usuario)
- Etiquetas de respuesta + comportamiento de heartbeat
- Metadatos de runtime (host/OS/model/thinking)

Consulta el desglose completo en [Prompt del sistema](/es/concepts/system-prompt).

## Qué cuenta en la ventana de contexto

Todo lo que recibe el modelo cuenta para el límite de contexto:

- Prompt del sistema (todas las secciones indicadas arriba)
- Historial de conversación (mensajes del usuario + asistente)
- Llamadas a herramientas y resultados de herramientas
- Archivos adjuntos/transcripciones (imágenes, audio, archivos)
- Resúmenes de compactación y artefactos de poda
- Wrappers del proveedor o encabezados de seguridad (no visibles, pero igualmente contabilizados)

Para imágenes, OpenClaw reduce la escala de las cargas útiles de imágenes de transcripciones/herramientas antes de las llamadas al proveedor.
Usa `agents.defaults.imageMaxDimensionPx` (predeterminado: `1200`) para ajustar esto:

- Los valores más bajos suelen reducir el uso de vision tokens y el tamaño de la carga útil.
- Los valores más altos conservan más detalle visual para capturas de pantalla con mucho OCR/UI.

Para un desglose práctico (por archivo inyectado, herramientas, Skills y tamaño del prompt del sistema), usa `/context list` o `/context detail`. Consulta [Contexto](/es/concepts/context).

## Cómo ver el uso actual de tokens

Usa esto en el chat:

- `/status` → **tarjeta de estado con muchos emojis** con el modelo de la sesión, uso de contexto,
  tokens de entrada/salida de la última respuesta y **costo estimado** (solo con API key).
- `/usage off|tokens|full` → agrega un **pie de uso por respuesta** a cada respuesta.
  - Persiste por sesión (se almacena como `responseUsage`).
  - La autenticación OAuth **oculta el costo** (solo tokens).
- `/usage cost` → muestra un resumen local de costos a partir de los registros de sesión de OpenClaw.

Otras superficies:

- **TUI/Web TUI:** `/status` y `/usage` son compatibles.
- **CLI:** `openclaw status --usage` y `openclaw channels list` muestran
  ventanas de cuota del proveedor normalizadas (`X% left`, no costos por respuesta).
  Proveedores actuales con ventana de uso: Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi y z.ai.

Las superficies de uso normalizan alias comunes de campos nativos del proveedor antes de mostrarlos.
Para el tráfico de Responses de la familia OpenAI, eso incluye tanto `input_tokens` /
`output_tokens` como `prompt_tokens` / `completion_tokens`, de modo que los nombres de campo
específicos del transporte no cambian `/status`, `/usage` ni los resúmenes de sesión.
El uso JSON de Gemini CLI también se normaliza: el texto de respuesta proviene de `response`, y
`stats.cached` se asigna a `cacheRead`, con `stats.input_tokens - stats.cached`
usado cuando la CLI omite un campo `stats.input` explícito.
Para el tráfico nativo de Responses de la familia OpenAI, los alias de uso de WebSocket/SSE se
normalizan de la misma forma, y los totales recurren a la suma normalizada de entrada + salida cuando
`total_tokens` falta o es `0`.
Cuando la instantánea de la sesión actual es escasa, `/status` y `session_status` también pueden
recuperar contadores de tokens/cache y la etiqueta del modelo de runtime activo del registro de uso
de la transcripción más reciente. Los valores activos distintos de cero existentes siguen teniendo
prioridad sobre los valores de respaldo de la transcripción, y los totales de transcripción más grandes
orientados al prompt pueden prevalecer cuando los totales almacenados faltan o son menores.
La autenticación de uso para ventanas de cuota del proveedor proviene de hooks específicos del proveedor cuando
están disponibles; de lo contrario, OpenClaw recurre a credenciales OAuth/API-key coincidentes
de perfiles de autenticación, env o config.

## Estimación de costos (cuando se muestra)

Los costos se estiman a partir de tu configuración de precios del modelo:

```
models.providers.<provider>.models[].cost
```

Estos son **USD por 1M de tokens** para `input`, `output`, `cacheRead` y
`cacheWrite`. Si faltan los precios, OpenClaw muestra solo tokens. Los tokens OAuth
nunca muestran costo en dólares.

## Impacto del TTL de caché y la poda

El almacenamiento en caché del prompt del proveedor solo se aplica dentro de la ventana TTL de la caché. OpenClaw puede
ejecutar opcionalmente **poda de cache-ttl**: poda la sesión una vez que el TTL de la caché
ha expirado y luego restablece la ventana de caché para que las solicitudes posteriores puedan reutilizar el
contexto recién almacenado en caché en lugar de volver a almacenar en caché todo el historial. Esto mantiene más bajos
los costos de escritura en caché cuando una sesión queda inactiva más allá del TTL.

Configúralo en [Configuración del Gateway](/es/gateway/configuration) y consulta los
detalles del comportamiento en [Poda de sesión](/es/concepts/session-pruning).

Heartbeat puede mantener la caché **activa** durante periodos de inactividad. Si el TTL de la caché de tu modelo
es `1h`, establecer el intervalo de heartbeat justo por debajo de eso (por ejemplo, `55m`) puede evitar
volver a almacenar en caché todo el prompt, reduciendo los costos de escritura en caché.

En configuraciones con varios agentes, puedes mantener una configuración de modelo compartida y ajustar el comportamiento de caché
por agente con `agents.list[].params.cacheRetention`.

Para una guía completa, control por control, consulta [Prompt Caching](/reference/prompt-caching).

Para los precios de la API de Anthropic, las lecturas de caché son significativamente más baratas que los
tokens de entrada, mientras que las escrituras de caché se facturan con un multiplicador mayor. Consulta los precios de
prompt caching de Anthropic para ver las tarifas y multiplicadores TTL más recientes:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Ejemplo: mantener activa una caché de 1h con heartbeat

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Ejemplo: tráfico mixto con estrategia de caché por agente

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # línea base predeterminada para la mayoría de los agentes
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # mantener activa una caché larga para sesiones profundas
    - id: "alerts"
      params:
        cacheRetention: "none" # evitar escrituras en caché para notificaciones esporádicas
```

`agents.list[].params` se fusiona sobre `params` del modelo seleccionado, así que puedes
sobrescribir solo `cacheRetention` y heredar sin cambios otros valores predeterminados del modelo.

### Ejemplo: habilitar el encabezado beta de contexto 1M de Anthropic

La ventana de contexto 1M de Anthropic actualmente está protegida por beta. OpenClaw puede inyectar el
valor `anthropic-beta` necesario cuando habilitas `context1m` en modelos Opus
o Sonnet compatibles.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          context1m: true
```

Esto se asigna al encabezado beta `context-1m-2025-08-07` de Anthropic.

Esto solo se aplica cuando `context1m: true` está configurado en esa entrada del modelo.

Requisito: la credencial debe ser apta para uso de contexto largo (facturación con API key,
o la ruta de inicio de sesión de Claude de OpenClaw con Extra Usage habilitado). Si no,
Anthropic responde
con `HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

Si autenticas Anthropic con tokens OAuth/de suscripción (`sk-ant-oat-*`),
OpenClaw omite el encabezado beta `context-1m-*` porque Anthropic actualmente
rechaza esa combinación con HTTP 401.

## Consejos para reducir la presión de tokens

- Usa `/compact` para resumir sesiones largas.
- Recorta las salidas grandes de herramientas en tus flujos de trabajo.
- Reduce `agents.defaults.imageMaxDimensionPx` para sesiones con muchas capturas de pantalla.
- Mantén breves las descripciones de Skills (la lista de Skills se inyecta en el prompt).
- Prefiere modelos más pequeños para trabajo prolijo y exploratorio.

Consulta [Skills](/tools/skills) para ver la fórmula exacta de sobrecarga de la lista de Skills.
