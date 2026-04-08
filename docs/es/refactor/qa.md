---
x-i18n:
    generated_at: "2026-04-08T02:18:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e156cc8e2fe946a0423862f937754a7caa1fe7e6863b50a80bff49a1c86e1e8
    source_path: refactor/qa.md
    workflow: 15
---

# Refactorización de QA

Estado: la migración fundacional ya se integró.

## Objetivo

Mover QA de OpenClaw de un modelo de definición dividida a una única fuente de verdad:

- metadatos de escenarios
- prompts enviados al modelo
- configuración y desmontaje
- lógica del arnés
- aserciones y criterios de éxito
- artefactos y sugerencias de informes

El estado final deseado es un arnés de QA genérico que cargue archivos potentes de definición de escenarios en lugar de codificar la mayor parte del comportamiento en TypeScript.

## Estado actual

La fuente principal de verdad ahora vive en `qa/scenarios.md`.

Implementado:

- `qa/scenarios.md`
  - paquete canónico de QA
  - identidad del operador
  - misión de inicio
  - metadatos de escenarios
  - vinculaciones de controladores
- `extensions/qa-lab/src/scenario-catalog.ts`
  - analizador del paquete Markdown + validación con zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - renderizado del plan a partir del paquete Markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - siembra archivos de compatibilidad generados más `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - selecciona escenarios ejecutables mediante vinculaciones de controladores definidas en Markdown
- Protocolo + UI del bus de QA
  - adjuntos inline genéricos para renderizado de imagen/video/audio/archivo

Superficies divididas restantes:

- `extensions/qa-lab/src/suite.ts`
  - todavía se encarga de la mayor parte de la lógica personalizada ejecutable de controladores
- `extensions/qa-lab/src/report.ts`
  - todavía deriva la estructura del informe a partir de salidas en tiempo de ejecución

Así que la división de la fuente de verdad está corregida, pero la ejecución sigue estando mayormente respaldada por controladores en lugar de ser totalmente declarativa.

## Cómo es realmente la superficie de escenarios

Leyendo la suite actual se ven unas cuantas clases distintas de escenarios.

### Interacción simple

- línea base del canal
- línea base de DM
- seguimiento en hilo
- cambio de modelo
- continuidad de aprobación
- reacción/edición/eliminación

### Mutación de configuración y tiempo de ejecución

- desactivación de skill mediante parche de configuración
- activación tras reinicio por aplicar configuración
- cambio de capacidad tras reinicio de configuración
- comprobación de desvío del inventario en tiempo de ejecución

### Aserciones de sistema de archivos y repositorio

- informe de descubrimiento de source/docs
- compilación de Lobster Invaders
- búsqueda de artefacto de imagen generado

### Orquestación de memoria

- recuperación de memoria
- herramientas de memoria en contexto de canal
- fallback por fallo de memoria
- clasificación de memoria de sesión
- aislamiento de memoria por hilo
- barrido de sueños de memoria

### Integración de herramientas y plugins

- llamada de herramientas MCP plugin-tools
- visibilidad de Skills
- instalación dinámica de skill
- generación nativa de imágenes
- roundtrip de imagen
- comprensión de imagen desde adjunto

### Multiturno y multi-actor

- transferencia a subagente
- síntesis fanout de subagente
- flujos de estilo recuperación tras reinicio

Estas categorías importan porque determinan los requisitos del DSL. Una lista plana de prompt + texto esperado no es suficiente.

## Dirección

### Fuente única de verdad

Usar `qa/scenarios.md` como la fuente de verdad escrita.

El paquete debe seguir siendo:

- legible para humanos en revisión
- analizable por máquina
- lo bastante rico para impulsar:
  - ejecución de la suite
  - arranque del espacio de trabajo de QA
  - metadatos de la UI de QA Lab
  - prompts de docs/descubrimiento
  - generación de informes

### Formato de autoría preferido

Usar Markdown como formato de nivel superior, con YAML estructurado dentro.

Forma recomendada:

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - referencias a docs
  - referencias a código
  - anulaciones de modelo/proveedor
  - prerrequisitos
- secciones en prosa
  - objetivo
  - notas
  - sugerencias de depuración
- bloques YAML delimitados
  - setup
  - steps
  - assertions
  - cleanup

Esto proporciona:

- mejor legibilidad en PR que un JSON gigante
- contexto más rico que YAML puro
- análisis estricto y validación con zod

El JSON sin procesar es aceptable solo como una forma intermedia generada.

## Forma propuesta del archivo de escenario

Ejemplo:

````md
---
id: image-generation-roundtrip
title: Roundtrip de generación de imagen
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objetivo

Verificar que los medios generados se vuelven a adjuntar en el turno de seguimiento.

# Configuración

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Pasos

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Comprobación de generación de imagen: genera una imagen QA de un faro y resúmela en una frase corta.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Comprobación de generación de imagen
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Comprobación de inspección de roundtrip de imagen: describe el adjunto del faro generado en una frase corta.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expectativa

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Comprobación de inspección de roundtrip de imagen
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Capacidades del ejecutor que el DSL debe cubrir

Según la suite actual, el ejecutor genérico necesita algo más que ejecutar prompts.

### Acciones de entorno y configuración

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Acciones de turno del agente

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Acciones de configuración y tiempo de ejecución

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Acciones de archivos y artefactos

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Acciones de memoria y cron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### Acciones de MCP

- `mcp.callTool`

### Aserciones

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## Variables y referencias a artefactos

El DSL debe admitir salidas guardadas y referencias posteriores.

Ejemplos de la suite actual:

- crear un hilo y luego reutilizar `threadId`
- crear una sesión y luego reutilizar `sessionKey`
- generar una imagen y luego adjuntar el archivo en el siguiente turno
- generar una cadena de marcador de activación y luego afirmar que aparece más tarde

Capacidades necesarias:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- referencias tipadas para rutas, claves de sesión, identificadores de hilo, marcadores, salidas de herramientas

Sin soporte de variables, el arnés seguirá filtrando lógica de escenarios de vuelta a TypeScript.

## Lo que debería permanecer como vías de escape

Un ejecutor declarativo completamente puro no es realista en la fase 1.

Algunos escenarios son inherentemente pesados en orquestación:

- barrido de sueños de memoria
- activación tras reinicio por aplicar configuración
- cambio de capacidad tras reinicio de configuración
- resolución de artefacto de imagen generado por marca temporal/ruta
- evaluación del informe de descubrimiento

Por ahora, estos deberían usar controladores personalizados explícitos.

Regla recomendada:

- 85-90% declarativo
- pasos `customHandler` explícitos para el resto difícil
- solo controladores personalizados con nombre y documentados
- sin código inline anónimo en el archivo de escenario

Eso mantiene limpio el motor genérico y al mismo tiempo permite avanzar.

## Cambio de arquitectura

### Actual

El Markdown de escenarios ya es la fuente de verdad para:

- ejecución de la suite
- archivos de arranque del espacio de trabajo
- catálogo de escenarios de la UI de QA Lab
- metadatos de informes
- prompts de descubrimiento

Compatibilidad generada:

- el espacio de trabajo sembrado sigue incluyendo `QA_KICKOFF_TASK.md`
- el espacio de trabajo sembrado sigue incluyendo `QA_SCENARIO_PLAN.md`
- el espacio de trabajo sembrado ahora también incluye `QA_SCENARIOS.md`

## Plan de refactorización

### Fase 1: cargador y esquema

Hecho.

- se añadió `qa/scenarios.md`
- se añadió el analizador para contenido de paquete Markdown YAML con nombre
- se validó con zod
- se cambiaron los consumidores al paquete analizado
- se eliminaron `qa/seed-scenarios.json` y `qa/QA_KICKOFF_TASK.md` a nivel de repositorio

### Fase 2: motor genérico

- dividir `extensions/qa-lab/src/suite.ts` en:
  - cargador
  - motor
  - registro de acciones
  - registro de aserciones
  - controladores personalizados
- mantener las funciones auxiliares existentes como operaciones del motor

Entregable:

- el motor ejecuta escenarios declarativos simples

Empezar con escenarios que son principalmente prompt + espera + aserción:

- seguimiento en hilo
- comprensión de imagen desde adjunto
- visibilidad e invocación de skill
- línea base del canal

Entregable:

- primeros escenarios reales definidos en Markdown enviados mediante el motor genérico

### Fase 4: migrar escenarios intermedios

- roundtrip de generación de imagen
- herramientas de memoria en contexto de canal
- clasificación de memoria de sesión
- transferencia a subagente
- síntesis fanout de subagente

Entregable:

- variables, artefactos, aserciones de herramientas y aserciones de registro de solicitudes validadas

### Fase 5: mantener los escenarios difíciles en controladores personalizados

- barrido de sueños de memoria
- activación tras reinicio por aplicar configuración
- cambio de capacidad tras reinicio de configuración
- desvío del inventario en tiempo de ejecución

Entregable:

- mismo formato de autoría, pero con bloques de pasos personalizados explícitos cuando sea necesario

### Fase 6: eliminar el mapa de escenarios codificado de forma rígida

Una vez que la cobertura del paquete sea lo bastante buena:

- eliminar la mayor parte de la ramificación TypeScript específica de escenarios de `extensions/qa-lab/src/suite.ts`

## Soporte de Fake Slack / medios enriquecidos

El bus actual de QA está orientado primero al texto.

Archivos relevantes:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Hoy el bus de QA admite:

- texto
- reacciones
- hilos

Todavía no modela adjuntos de medios inline.

### Contrato de transporte necesario

Añadir un modelo genérico de adjuntos para el bus de QA:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

Luego añadir `attachments?: QaBusAttachment[]` a:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Por qué primero genérico

No construir un modelo de medios exclusivo para Slack.

En su lugar:

- un modelo de transporte de QA genérico
- varios renderizadores encima de él
  - chat actual de QA Lab
  - futura web fake Slack
  - cualquier otra vista futura de transporte simulado

Esto evita lógica duplicada y permite que los escenarios de medios sigan siendo agnósticos del transporte.

### Trabajo de UI necesario

Actualizar la UI de QA para renderizar:

- vista previa de imagen inline
- reproductor de audio inline
- reproductor de video inline
- chip de adjunto de archivo

La UI actual ya puede renderizar hilos y reacciones, así que el renderizado de adjuntos debería añadirse sobre el mismo modelo de tarjeta de mensaje.

### Trabajo de escenarios habilitado por el transporte de medios

Una vez que los adjuntos fluyan por el bus de QA, podremos añadir escenarios de chat simulado más ricos:

- respuesta inline con imagen en fake Slack
- comprensión de adjunto de audio
- comprensión de adjunto de video
- orden mixto de adjuntos
- respuesta en hilo con medios retenidos

## Recomendación

El siguiente bloque de implementación debería ser:

1. añadir cargador de escenarios Markdown + esquema zod
2. generar el catálogo actual a partir de Markdown
3. migrar primero algunos escenarios simples
4. añadir soporte genérico de adjuntos al bus de QA
5. renderizar imagen inline en la UI de QA
6. luego ampliar a audio y video

Este es el camino más pequeño que demuestra ambos objetivos:

- QA genérico definido en Markdown
- superficies de mensajería simulada más ricas

## Preguntas abiertas

- si los archivos de escenario deberían permitir plantillas de prompt Markdown incrustadas con interpolación de variables
- si setup/cleanup deberían ser secciones con nombre o solo listas ordenadas de acciones
- si las referencias a artefactos deberían estar fuertemente tipadas en el esquema o basadas en cadenas
- si los controladores personalizados deberían vivir en un único registro o en registros por superficie
- si el archivo de compatibilidad JSON generado debería seguir versionado durante la migración
