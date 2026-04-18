---
x-i18n:
    generated_at: "2026-04-18T04:59:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: dbb2c70c82da7f6f12d90e25666635ff4147c52e8a94135e902d1de4f5cbccca
    source_path: refactor/qa.md
    workflow: 15
---

# Refactorización de QA

Estado: se consolidó la migración fundacional.

## Objetivo

Mover QA de OpenClaw de un modelo de definición dividida a una única fuente de verdad:

- metadatos de escenarios
- prompts enviados al modelo
- configuración y desmontaje
- lógica del arnés
- aserciones y criterios de éxito
- artefactos e indicaciones para informes

El estado final deseado es un arnés de QA genérico que cargue archivos potentes de definición de escenarios en lugar de codificar la mayor parte del comportamiento en TypeScript.

## Estado actual

La fuente principal de verdad ahora vive en `qa/scenarios/index.md` más un archivo por
escenario en `qa/scenarios/<theme>/*.md`.

Implementado:

- `qa/scenarios/index.md`
  - metadatos canónicos del paquete de QA
  - identidad del operador
  - misión inicial
- `qa/scenarios/<theme>/*.md`
  - un archivo markdown por escenario
  - metadatos del escenario
  - enlaces de handlers
  - configuración de ejecución específica del escenario
- `extensions/qa-lab/src/scenario-catalog.ts`
  - analizador del paquete markdown + validación con zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - renderizado del plan a partir del paquete markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - siembra archivos de compatibilidad generados más `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - selecciona escenarios ejecutables mediante enlaces de handlers definidos en markdown
- Protocolo de bus de QA + UI
  - adjuntos en línea genéricos para renderizado de imagen/video/audio/archivo

Superficies divididas restantes:

- `extensions/qa-lab/src/suite.ts`
  - todavía mantiene la mayor parte de la lógica ejecutable personalizada de handlers
- `extensions/qa-lab/src/report.ts`
  - todavía deriva la estructura del informe a partir de las salidas del runtime

Así que la división de la fuente de verdad está resuelta, pero la ejecución sigue dependiendo en gran medida de handlers en lugar de ser totalmente declarativa.

## Cómo es realmente la superficie de escenarios

Leer la suite actual muestra algunas clases distintas de escenarios.

### Interacción simple

- línea base del canal
- línea base de MD
- seguimiento en hilo
- cambio de modelo
- continuación de aprobación
- reacción/edición/eliminación

### Mutación de configuración y runtime

- deshabilitación de Skill mediante parche de configuración
- reactivación tras reinicio con config apply
- cambio de capacidad tras reinicio de configuración
- comprobación de deriva del inventario del runtime

### Aserciones de sistema de archivos y repositorio

- informe de descubrimiento de source/docs
- compilar Lobster Invaders
- búsqueda de artefacto de imagen generado

### Orquestación de memoria

- recuperación de memoria
- herramientas de memoria en contexto de canal
- fallback ante fallo de memoria
- clasificación de memoria de sesión
- aislamiento de memoria por hilo
- barrido de Dreaming de memoria

### Integración de herramientas y plugins

- llamada MCP plugin-tools
- visibilidad de Skills
- instalación dinámica de Skills
- generación nativa de imágenes
- roundtrip de imagen
- comprensión de imagen desde adjunto

### Multiturno y multiactor

- transferencia a subagente
- síntesis con fanout de subagentes
- flujos de estilo de recuperación tras reinicio

Estas categorías importan porque determinan los requisitos del DSL. Una lista plana de prompt + texto esperado no es suficiente.

## Dirección

### Fuente única de verdad

Usar `qa/scenarios/index.md` más `qa/scenarios/<theme>/*.md` como fuente de verdad redactada.

El paquete debe seguir siendo:

- legible para humanos en revisión
- interpretable por máquina
- lo bastante rico para impulsar:
  - ejecución de la suite
  - arranque del espacio de trabajo de QA
  - metadatos de la UI de QA Lab
  - prompts de documentación/descubrimiento
  - generación de informes

### Formato de autoría preferido

Usar markdown como formato de nivel superior, con YAML estructurado dentro.

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

Esto aporta:

- mejor legibilidad en PR que un JSON gigante
- contexto más rico que YAML puro
- análisis estricto y validación con zod

El JSON sin procesar es aceptable solo como forma generada intermedia.

## Forma propuesta del archivo de escenario

Ejemplo:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
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

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

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

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Capacidades del ejecutor que el DSL debe cubrir

Basándose en la suite actual, el ejecutor genérico necesita más que ejecución de prompts.

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

### Acciones de configuración y runtime

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

### Acciones de memoria y Cron

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
- generar una cadena de marcador de activación y luego afirmar que aparezca después

Capacidades necesarias:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- referencias tipadas para rutas, claves de sesión, ids de hilo, marcadores, salidas de herramientas

Sin soporte para variables, el arnés seguirá filtrando la lógica del escenario de vuelta a TypeScript.

## Qué debería mantenerse como vías de escape

Un ejecutor declarativo completamente puro no es realista en la fase 1.

Algunos escenarios son intrínsecamente intensivos en orquestación:

- barrido de Dreaming de memoria
- reactivación tras reinicio con config apply
- cambio de capacidad tras reinicio de configuración
- resolución de artefactos de imagen generados por marca de tiempo/ruta
- evaluación de informes de descubrimiento

Por ahora, estos deberían usar handlers personalizados explícitos.

Regla recomendada:

- 85-90 % declarativo
- pasos `customHandler` explícitos para la parte difícil restante
- solo handlers personalizados con nombre y documentados
- nada de código en línea anónimo en el archivo del escenario

Eso mantiene limpio el motor genérico y aun así permite avanzar.

## Cambio de arquitectura

### Actual

El markdown de escenarios ya es la fuente de verdad para:

- ejecución de la suite
- archivos de arranque del espacio de trabajo
- catálogo de escenarios de la UI de QA Lab
- metadatos de informes
- prompts de descubrimiento

Compatibilidad generada:

- el espacio de trabajo sembrado todavía incluye `QA_KICKOFF_TASK.md`
- el espacio de trabajo sembrado todavía incluye `QA_SCENARIO_PLAN.md`
- el espacio de trabajo sembrado ahora también incluye `QA_SCENARIOS.md`

## Plan de refactorización

### Fase 1: cargador y esquema

Hecho.

- se añadió `qa/scenarios/index.md`
- se dividieron los escenarios en `qa/scenarios/<theme>/*.md`
- se añadió un analizador para el contenido nombrado del paquete markdown YAML
- se validó con zod
- se cambiaron los consumidores para usar el paquete analizado
- se eliminaron `qa/seed-scenarios.json` y `qa/QA_KICKOFF_TASK.md` a nivel de repositorio

### Fase 2: motor genérico

- dividir `extensions/qa-lab/src/suite.ts` en:
  - cargador
  - motor
  - registro de acciones
  - registro de aserciones
  - handlers personalizados
- mantener las funciones auxiliares existentes como operaciones del motor

Entregable:

- el motor ejecuta escenarios declarativos simples

Empezar con escenarios que son mayormente prompt + espera + aserción:

- seguimiento en hilo
- comprensión de imagen desde adjunto
- visibilidad e invocación de Skills
- línea base del canal

Entregable:

- primeros escenarios reales definidos en markdown enviados a producción a través del motor genérico

### Fase 4: migrar escenarios intermedios

- roundtrip de generación de imagen
- herramientas de memoria en contexto de canal
- clasificación de memoria de sesión
- transferencia a subagente
- síntesis con fanout de subagentes

Entregable:

- variables, artefactos, aserciones de herramientas y aserciones de request-log demostradas

### Fase 5: mantener los escenarios difíciles en handlers personalizados

- barrido de Dreaming de memoria
- reactivación tras reinicio con config apply
- cambio de capacidad tras reinicio de configuración
- deriva del inventario del runtime

Entregable:

- mismo formato de autoría, pero con bloques de pasos personalizados explícitos donde sea necesario

### Fase 6: eliminar el mapa de escenarios codificado

Una vez que la cobertura del paquete sea lo bastante buena:

- eliminar la mayor parte del branching específico de escenarios en TypeScript de `extensions/qa-lab/src/suite.ts`

## Soporte de Fake Slack / medios enriquecidos

El bus de QA actual está orientado principalmente a texto.

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

Todavía no modela adjuntos multimedia en línea.

### Contrato de transporte necesario

Añadir un modelo genérico de adjuntos del bus de QA:

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

### Por qué genérico primero

No construyas un modelo de medios solo para Slack.

En su lugar:

- un modelo de transporte de QA genérico
- múltiples renderizadores encima de él
  - chat actual de QA Lab
  - futuro fake Slack web
  - cualquier otra vista de transporte simulado

Esto evita lógica duplicada y permite que los escenarios multimedia sigan siendo agnósticos al transporte.

### Trabajo de UI necesario

Actualizar la UI de QA para renderizar:

- vista previa de imagen en línea
- reproductor de audio en línea
- reproductor de video en línea
- chip de archivo adjunto

La UI actual ya puede renderizar hilos y reacciones, así que el renderizado de adjuntos debería añadirse sobre el mismo modelo de tarjeta de mensaje.

### Trabajo de escenarios habilitado por el transporte multimedia

Una vez que los adjuntos fluyan por el bus de QA, podremos añadir escenarios de chat simulado más ricos:

- respuesta con imagen en línea en fake Slack
- comprensión de adjuntos de audio
- comprensión de adjuntos de video
- orden mixto de adjuntos
- respuesta en hilo con medios conservados

## Recomendación

El siguiente bloque de implementación debería ser:

1. añadir cargador de escenarios markdown + esquema zod
2. generar el catálogo actual a partir de markdown
3. migrar primero algunos escenarios simples
4. añadir soporte genérico para adjuntos en el bus de QA
5. renderizar imagen en línea en la UI de QA
6. luego ampliar a audio y video

Este es el camino más pequeño que demuestra ambos objetivos:

- QA genérico definido en markdown
- superficies de mensajería simulada más ricas

## Preguntas abiertas

- si los archivos de escenarios deberían permitir plantillas de prompt markdown incrustadas con interpolación de variables
- si setup/cleanup deberían ser secciones con nombre o simplemente listas de acciones ordenadas
- si las referencias a artefactos deberían estar fuertemente tipadas en el esquema o basadas en cadenas
- si los handlers personalizados deberían vivir en un solo registro o en registros por superficie
- si el archivo de compatibilidad JSON generado debería seguir versionado durante la migración
