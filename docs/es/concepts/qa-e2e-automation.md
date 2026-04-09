---
read_when:
    - Ampliar qa-lab o qa-channel
    - Agregar escenarios de QA respaldados por el repositorio
    - Crear automatización de QA de mayor realismo alrededor del panel del Gateway
summary: Estructura de automatización de QA privada para qa-lab, qa-channel, escenarios sembrados e informes de protocolo
title: Automatización E2E de QA
x-i18n:
    generated_at: "2026-04-09T01:27:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: c922607d67e0f3a2489ac82bc9f510f7294ced039c1014c15b676d826441d833
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatización E2E de QA

La pila de QA privada está pensada para ejercitar OpenClaw de una manera más realista,
con forma de canal, de lo que puede hacerlo una sola prueba unitaria.

Partes actuales:

- `extensions/qa-channel`: canal de mensajes sintético con superficies de MD, canal, hilo,
  reacción, edición y eliminación.
- `extensions/qa-lab`: interfaz de depuración y bus de QA para observar la transcripción,
  inyectar mensajes entrantes y exportar un informe en Markdown.
- `qa/`: recursos semilla respaldados por el repositorio para la tarea inicial y los
  escenarios base de QA.

El flujo actual del operador de QA es un sitio de QA de dos paneles:

- Izquierda: panel del Gateway (Control UI) con el agente.
- Derecha: QA Lab, que muestra la transcripción tipo Slack y el plan del escenario.

Ejecuta esto con:

```bash
pnpm qa:lab:up
```

Eso compila el sitio de QA, inicia la ruta del gateway respaldada por Docker y expone la
página de QA Lab donde un operador o un bucle de automatización puede dar al agente una
misión de QA, observar el comportamiento real del canal y registrar qué funcionó, qué falló o
qué siguió bloqueado.

Para iterar más rápido en la UI de QA Lab sin reconstruir la imagen de Docker cada vez,
inicia la pila con un paquete de QA Lab montado mediante bind:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` mantiene los servicios de Docker sobre una imagen precompilada y monta mediante bind
`extensions/qa-lab/web/dist` dentro del contenedor `qa-lab`. `qa:lab:watch`
recompila ese paquete cuando hay cambios, y el navegador se recarga automáticamente cuando cambia el hash
de recursos de QA Lab.

## Semillas respaldadas por el repositorio

Los recursos semilla viven en `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Estos están intencionalmente en git para que el plan de QA sea visible tanto para las personas como para el
agente. La lista base debe seguir siendo lo bastante amplia como para cubrir:

- chat por MD y por canal
- comportamiento de hilos
- ciclo de vida de acciones de mensajes
- callbacks de cron
- recuperación de memoria
- cambio de modelo
- transferencia a subagente
- lectura del repositorio y de la documentación
- una pequeña tarea de compilación como Lobster Invaders

## Informes

`qa-lab` exporta un informe de protocolo en Markdown a partir de la línea de tiempo observada del bus.
El informe debe responder:

- Qué funcionó
- Qué falló
- Qué siguió bloqueado
- Qué escenarios de seguimiento vale la pena agregar

Para comprobaciones de carácter y estilo, ejecuta el mismo escenario en múltiples refs de modelos en vivo
y escribe un informe en Markdown evaluado:

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

El comando ejecuta procesos hijo locales del gateway de QA, no Docker. Los escenarios de evaluación de carácter
deben establecer la persona mediante `SOUL.md`, y luego ejecutar turnos de usuario normales
como chat, ayuda del espacio de trabajo y pequeñas tareas de archivos. No se le debe decir al modelo
candidato que está siendo evaluado. El comando conserva cada transcripción completa,
registra estadísticas básicas de ejecución y luego pide a los modelos juez en modo fast con
razonamiento `xhigh` que clasifiquen las ejecuciones por naturalidad, vibra y humor.
Usa `--blind-judge-models` al comparar proveedores: el prompt del juez sigue recibiendo
cada transcripción y estado de ejecución, pero las refs candidatas se sustituyen por
etiquetas neutras como `candidate-01`; el informe vuelve a asignar las clasificaciones a las refs reales después
del análisis.
Las ejecuciones candidatas usan de forma predeterminada pensamiento `high`, con `xhigh` para los modelos de OpenAI que
lo admiten. Sustituye un candidato específico en línea con
`--model provider/model,thinking=<level>`. `--thinking <level>` sigue estableciendo un
respaldo global, y la forma anterior `--model-thinking <provider/model=level>` se
mantiene por compatibilidad.
Las refs candidatas de OpenAI usan de forma predeterminada el modo fast para que se use el procesamiento prioritario allí
donde el proveedor lo admita. Agrega `,fast`, `,no-fast` o `,fast=false` en línea cuando un
solo candidato o juez necesite una sustitución. Pasa `--fast` solo cuando quieras
forzar el modo fast para todos los modelos candidatos. Las duraciones de candidatos y jueces se
registran en el informe para el análisis comparativo, pero los prompts de los jueces dicen explícitamente que
no clasifiquen por velocidad.
Tanto las ejecuciones de modelos candidatos como las de jueces usan por defecto una concurrencia de 16. Reduce
`--concurrency` o `--judge-concurrency` cuando los límites del proveedor o la presión del gateway local
hagan que una ejecución sea demasiado ruidosa.
Cuando no se pasa ningún `--model` candidato, la evaluación de carácter usa por defecto
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` y
`google/gemini-3.1-pro-preview` cuando no se pasa `--model`.
Cuando no se pasa `--judge-model`, los jueces usan por defecto
`openai/gpt-5.4,thinking=xhigh,fast` y
`anthropic/claude-opus-4-6,thinking=high`.

## Documentación relacionada

- [Testing](/es/help/testing)
- [QA Channel](/es/channels/qa-channel)
- [Dashboard](/web/dashboard)
