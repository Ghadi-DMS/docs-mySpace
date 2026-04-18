---
read_when:
    - Ampliar qa-lab o qa-channel
    - Agregar escenarios de QA respaldados por el repositorio
    - Crear una automatización de QA de mayor realismo en torno al panel de Gateway
summary: Forma de la automatización privada de QA para qa-lab, qa-channel, escenarios con seed e informes de protocolo
title: Automatización E2E de QA
x-i18n:
    generated_at: "2026-04-18T04:59:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: adf8c5f74e8fabdc8e9fd7ecd41afce8b60354c7dd24d92ac926d3c527927cd4
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatización E2E de QA

La pila privada de QA está pensada para ejercitar OpenClaw de una manera más realista,
con forma de canal, que la que puede cubrir una sola prueba unitaria.

Piezas actuales:

- `extensions/qa-channel`: canal de mensajes sintético con superficies de MD, canal, hilo,
  reacción, edición y eliminación.
- `extensions/qa-lab`: interfaz de depuración y bus de QA para observar la transcripción,
  inyectar mensajes entrantes y exportar un informe en Markdown.
- `qa/`: recursos seed respaldados por el repositorio para la tarea inicial y los
  escenarios base de QA.

El flujo actual del operador de QA es un sitio de QA de dos paneles:

- Izquierda: panel de Gateway (Control UI) con el agente.
- Derecha: QA Lab, que muestra la transcripción tipo Slack y el plan del escenario.

Ejecútalo con:

```bash
pnpm qa:lab:up
```

Eso compila el sitio de QA, inicia el carril de Gateway respaldado por Docker y expone la
página de QA Lab, donde un operador o un bucle de automatización puede darle al agente una
misión de QA, observar el comportamiento real del canal y registrar qué funcionó, qué falló o
qué siguió bloqueado.

Para una iteración más rápida de la UI de QA Lab sin reconstruir la imagen de Docker cada vez,
inicia la pila con un bundle de QA Lab montado por enlace:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` mantiene los servicios de Docker sobre una imagen precompilada y monta por enlace
`extensions/qa-lab/web/dist` dentro del contenedor `qa-lab`. `qa:lab:watch`
recompila ese bundle cuando hay cambios, y el navegador se recarga automáticamente cuando cambia el hash
de recursos de QA Lab.

Para un carril smoke de Matrix con transporte real, ejecuta:

```bash
pnpm openclaw qa matrix
```

Ese carril aprovisiona un homeserver Tuwunel desechable en Docker, registra usuarios temporales
de controlador, SUT y observador, crea una sala privada y luego ejecuta el Plugin real de Matrix dentro
de un proceso hijo de Gateway de QA. El carril de transporte en vivo mantiene la configuración del proceso hijo
limitada al transporte bajo prueba, por lo que Matrix se ejecuta sin `qa-channel` en la configuración
del proceso hijo. Escribe los artefactos de informe estructurado y un registro combinado de stdout/stderr
en el directorio de salida de Matrix QA seleccionado. Para capturar también la salida externa de compilación/lanzamiento
de `scripts/run-node.mjs`, establece `OPENCLAW_RUN_NODE_OUTPUT_LOG=<path>` en un archivo de registro local del repositorio.

Para un carril smoke de Telegram con transporte real, ejecuta:

```bash
pnpm openclaw qa telegram
```

Ese carril apunta a un grupo privado real de Telegram en lugar de aprovisionar un servidor desechable.
Requiere `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` y
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`, además de dos bots distintos en el mismo
grupo privado. El bot SUT debe tener un nombre de usuario de Telegram, y la observación bot a bot
funciona mejor cuando ambos bots tienen habilitado el modo de comunicación Bot-to-Bot
en `@BotFather`.

Los carriles de transporte en vivo ahora comparten un contrato más pequeño en lugar de que cada uno invente
su propia forma de lista de escenarios.

`qa-channel` sigue siendo la amplia suite sintética de comportamiento del producto y no forma parte
de la matriz de cobertura de transporte en vivo.

| Carril   | Canary | Puerta por mención | Bloqueo por allowlist | Respuesta de nivel superior | Reanudación tras reinicio | Seguimiento en hilo | Aislamiento de hilo | Observación de reacciones | Comando help |
| -------- | ------ | ------------------ | --------------------- | --------------------------- | ------------------------- | ------------------- | ------------------- | ------------------------- | ------------ |
| Matrix   | x      | x                  | x                     | x                           | x                         | x                   | x                   | x                         |              |
| Telegram | x      |                    |                       |                             |                           |                     |                     |                           | x            |

Esto mantiene `qa-channel` como la amplia suite de comportamiento del producto, mientras que Matrix,
Telegram y futuros transportes en vivo comparten una lista explícita de comprobación del contrato
de transporte.

Para un carril de VM Linux desechable sin incorporar Docker en la ruta de QA, ejecuta:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Esto inicia un huésped nuevo de Multipass, instala dependencias, compila OpenClaw
dentro del huésped, ejecuta `qa suite` y luego copia el informe y resumen normales de QA de vuelta a
`.artifacts/qa-e2e/...` en el host.
Reutiliza el mismo comportamiento de selección de escenarios que `qa suite` en el host.
Las ejecuciones del host y de Multipass ejecutan varios escenarios seleccionados en paralelo
con workers de Gateway aislados de forma predeterminada, hasta 64 workers o la cantidad de escenarios
seleccionados. Usa `--concurrency <count>` para ajustar la cantidad de workers, o
`--concurrency 1` para ejecución en serie.
Las ejecuciones en vivo reenvían las entradas de autenticación de QA compatibles que son prácticas para el
huésped: claves de proveedor basadas en variables de entorno, la ruta de configuración del proveedor en vivo de QA y
`CODEX_HOME` cuando esté presente. Mantén `--output-dir` bajo la raíz del repositorio para que el huésped
pueda escribir de vuelta a través del espacio de trabajo montado.

## Seeds respaldados por el repositorio

Los recursos seed viven en `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/<theme>/*.md`

Estos están intencionalmente en git para que el plan de QA sea visible tanto para las personas como para el
agente.

`qa-lab` debe seguir siendo un ejecutor genérico de Markdown. Cada archivo Markdown de escenario es
la fuente de verdad para una ejecución de prueba y debe definir:

- metadatos del escenario
- metadatos opcionales de categoría, capacidad, carril y riesgo
- referencias de documentación y código
- requisitos opcionales de Plugin
- parche opcional de configuración de Gateway
- el `qa-flow` ejecutable

La superficie de ejecución reutilizable que respalda `qa-flow` puede seguir siendo genérica
y transversal. Por ejemplo, los escenarios Markdown pueden combinar ayudantes del lado del transporte
con ayudantes del lado del navegador que controlan la Control UI integrada a través de la
interfaz `browser.request` de Gateway sin agregar un ejecutor de caso especial.

Los archivos de escenario deben agruparse por capacidad del producto en lugar de por carpeta del árbol
de código fuente. Mantén estables los ID de escenario cuando los archivos se muevan; usa `docsRefs` y `codeRefs`
para la trazabilidad de implementación.

La lista base debe seguir siendo lo suficientemente amplia como para cubrir:

- chat por MD y canal
- comportamiento de hilos
- ciclo de vida de acciones de mensajes
- callbacks de Cron
- recuperación de memoria
- cambio de modelo
- transferencia a subagente
- lectura del repositorio y de la documentación
- una pequeña tarea de compilación como Lobster Invaders

## Carriles de proveedor simulado

`qa suite` tiene dos carriles locales de proveedor simulado:

- `mock-openai` es el simulador de OpenClaw consciente del escenario. Sigue siendo el
  carril simulado determinista predeterminado para QA respaldado por el repositorio y compuertas de paridad.
- `aimock` inicia un servidor de proveedor respaldado por AIMock para cobertura experimental de protocolo,
  fixtures, grabación/reproducción y caos. Es complementario y no reemplaza al despachador de escenarios
  `mock-openai`.

La implementación del carril de proveedor vive en `extensions/qa-lab/src/providers/`.
Cada proveedor es dueño de sus valores predeterminados, el arranque del servidor local, la configuración
del modelo de Gateway, las necesidades de preparación del perfil de autenticación y los indicadores de capacidad
live/mock. El código compartido de suite y Gateway debe enrutar a través del registro de proveedores
en lugar de ramificarse según los nombres de proveedor.

## Adaptadores de transporte

`qa-lab` es dueño de una interfaz genérica de transporte para escenarios Markdown de QA.
`qa-channel` es el primer adaptador sobre esa interfaz, pero el objetivo de diseño es más amplio:
canales futuros, reales o sintéticos, deberían integrarse en el mismo ejecutor de suite en lugar
de agregar un ejecutor de QA específico para cada transporte.

A nivel de arquitectura, la división es:

- `qa-lab` es dueño de la ejecución genérica de escenarios, la concurrencia de workers, la escritura de artefactos y los informes.
- el adaptador de transporte es dueño de la configuración de Gateway, la preparación, la observación de entrada y salida, las acciones de transporte y el estado de transporte normalizado.
- los archivos Markdown de escenarios bajo `qa/scenarios/` definen la ejecución de prueba; `qa-lab` proporciona la superficie de ejecución reutilizable que los ejecuta.

La guía de adopción dirigida a mantenedores para nuevos adaptadores de canal se encuentra en
[Testing](/es/help/testing#adding-a-channel-to-qa).

## Informes

`qa-lab` exporta un informe de protocolo en Markdown a partir de la línea de tiempo observada del bus.
El informe debe responder:

- Qué funcionó
- Qué falló
- Qué siguió bloqueado
- Qué escenarios de seguimiento vale la pena agregar

Para comprobaciones de carácter y estilo, ejecuta el mismo escenario con múltiples referencias de modelos en vivo
y escribe un informe Markdown evaluado:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

El comando ejecuta procesos hijo locales de Gateway de QA, no Docker. Los escenarios de evaluación de carácter
deben establecer la personalidad mediante `SOUL.md`, y luego ejecutar turnos de usuario normales
como chat, ayuda del espacio de trabajo y pequeñas tareas con archivos. No se le debe decir al
modelo candidato que está siendo evaluado. El comando conserva cada transcripción completa,
registra estadísticas básicas de ejecución y luego pide a los modelos jueces en modo fast con
razonamiento `xhigh` que clasifiquen las ejecuciones por naturalidad, vibra y humor.
Usa `--blind-judge-models` al comparar proveedores: el prompt del juez sigue recibiendo
cada transcripción y estado de ejecución, pero las referencias candidatas se reemplazan por
etiquetas neutras como `candidate-01`; el informe vuelve a mapear las clasificaciones a las referencias reales
después del análisis.
Las ejecuciones candidatas usan `high` thinking de forma predeterminada, con `xhigh` para los modelos de OpenAI
que lo admiten. Sustituye un candidato específico en línea con
`--model provider/model,thinking=<level>`. `--thinking <level>` sigue estableciendo una
alternativa global, y la forma antigua `--model-thinking <provider/model=level>` se
mantiene por compatibilidad.
Las referencias candidatas de OpenAI usan el modo fast de forma predeterminada para que se use procesamiento prioritario
cuando el proveedor lo admita. Agrega `,fast`, `,no-fast` o `,fast=false` en línea cuando
un solo candidato o juez necesite una sustitución. Pasa `--fast` solo cuando quieras
forzar el modo fast para todos los modelos candidatos. Las duraciones de candidatos y jueces se
registran en el informe para análisis comparativo, pero los prompts de los jueces indican explícitamente
que no se clasifique por velocidad.
Tanto las ejecuciones de modelos candidatos como las de los jueces usan concurrencia 16 de forma predeterminada. Reduce
`--concurrency` o `--judge-concurrency` cuando los límites del proveedor o la presión del Gateway local
hagan que una ejecución sea demasiado ruidosa.
Cuando no se pasa ningún `--model` candidato, la evaluación de carácter usa por defecto
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` y
`google/gemini-3.1-pro-preview` cuando no se pasa `--model`.
Cuando no se pasa ningún `--judge-model`, los jueces usan por defecto
`openai/gpt-5.4,thinking=xhigh,fast` y
`anthropic/claude-opus-4-6,thinking=high`.

## Documentación relacionada

- [Testing](/es/help/testing)
- [QA Channel](/es/channels/qa-channel)
- [Dashboard](/web/dashboard)
