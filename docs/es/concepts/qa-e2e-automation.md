---
read_when:
    - Ampliar qa-lab o qa-channel
    - Agregar escenarios de QA respaldados por el repositorio
    - Crear una automatización de QA con mayor realismo en torno al Dashboard del Gateway
summary: Forma de la automatización privada de QA para qa-lab, qa-channel, escenarios predefinidos e informes de protocolo
title: Automatización E2E de QA
x-i18n:
    generated_at: "2026-04-08T02:14:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b4aa5acc8e77303f4045d4f04372494cae21b89d2fdaba856dbb4855ced9d27
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatización E2E de QA

La pila privada de QA está pensada para ejercitar OpenClaw de una forma más
realista y con forma de canal que la que puede ofrecer una sola prueba unitaria.

Componentes actuales:

- `extensions/qa-channel`: canal de mensajes sintético con superficies de MD, canal, hilo,
  reacción, edición y eliminación.
- `extensions/qa-lab`: interfaz de depuración y bus de QA para observar la transcripción,
  inyectar mensajes entrantes y exportar un informe en Markdown.
- `qa/`: recursos semilla respaldados por el repositorio para la tarea inicial y los
  escenarios de QA de referencia.

El flujo actual del operador de QA es un sitio de QA de dos paneles:

- Izquierda: Dashboard del Gateway (Control UI) con el agente.
- Derecha: QA Lab, que muestra la transcripción estilo Slack y el plan de escenarios.

Ejecútalo con:

```bash
pnpm qa:lab:up
```

Eso compila el sitio de QA, inicia la vía del gateway respaldada por Docker y expone la
página de QA Lab donde un operador o un bucle de automatización puede darle al agente una
misión de QA, observar el comportamiento real del canal y registrar qué funcionó, qué falló o
qué siguió bloqueado.

Para una iteración más rápida de la interfaz de QA Lab sin reconstruir la imagen de Docker en cada ocasión,
inicia la pila con un bundle de QA Lab montado mediante bind mount:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` mantiene los servicios de Docker en una imagen precompilada y monta mediante bind mount
`extensions/qa-lab/web/dist` en el contenedor `qa-lab`. `qa:lab:watch`
recompila ese bundle cuando hay cambios, y el navegador se recarga automáticamente cuando cambia el hash
de recursos de QA Lab.

## Semillas respaldadas por el repositorio

Los recursos semilla viven en `qa/`:

- `qa/scenarios.md`

Estos están intencionalmente en git para que tanto las personas como el
agente puedan ver el plan de QA. La lista de referencia debe seguir siendo lo suficientemente amplia como para cubrir:

- chat por MD y por canal
- comportamiento de hilos
- ciclo de vida de las acciones de mensajes
- callbacks de cron
- recuperación de memoria
- cambio de modelo
- transferencia a subagente
- lectura del repositorio y de la documentación
- una pequeña tarea de compilación, como Lobster Invaders

## Informes

`qa-lab` exporta un informe de protocolo en Markdown a partir de la línea de tiempo observada del bus.
El informe debe responder:

- Qué funcionó
- Qué falló
- Qué siguió bloqueado
- Qué escenarios de seguimiento vale la pena agregar

## Documentación relacionada

- [Testing](/es/help/testing)
- [QA Channel](/es/channels/qa-channel)
- [Dashboard](/web/dashboard)
