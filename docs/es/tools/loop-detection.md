---
read_when:
    - Un usuario informa que los agentes se quedan atascados repitiendo llamadas a herramientas
    - Necesitas ajustar la protección contra llamadas repetitivas
    - Estás editando políticas de runtime/herramientas del agente
summary: Cómo habilitar y ajustar las barreras de protección que detectan bucles repetitivos de llamadas a herramientas
title: Detección de bucles de herramientas
x-i18n:
    generated_at: "2026-04-05T12:56:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: dc3c92579b24cfbedd02a286b735d99a259b720f6d9719a9b93902c9fc66137d
    source_path: tools/loop-detection.md
    workflow: 15
---

# Detección de bucles de herramientas

OpenClaw puede evitar que los agentes se queden atascados en patrones repetidos de llamadas a herramientas.
La protección está **desactivada de forma predeterminada**.

Actívala solo cuando sea necesario, porque con ajustes estrictos puede bloquear llamadas repetidas legítimas.

## Por qué existe esto

- Detectar secuencias repetitivas que no avanzan.
- Detectar bucles de alta frecuencia sin resultados (misma herramienta, mismas entradas, errores repetidos).
- Detectar patrones específicos de llamadas repetidas para herramientas de sondeo conocidas.

## Bloque de configuración

Valores globales predeterminados:

```json5
{
  tools: {
    loopDetection: {
      enabled: false,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

Sobrescritura por agente (opcional):

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
            warningThreshold: 8,
            criticalThreshold: 16,
          },
        },
      },
    ],
  },
}
```

### Comportamiento de los campos

- `enabled`: interruptor principal. `false` significa que no se realiza detección de bucles.
- `historySize`: número de llamadas recientes a herramientas que se conservan para el análisis.
- `warningThreshold`: umbral antes de clasificar un patrón solo como advertencia.
- `criticalThreshold`: umbral para bloquear patrones repetitivos de bucle.
- `globalCircuitBreakerThreshold`: umbral global de corte para ausencia de progreso.
- `detectors.genericRepeat`: detecta patrones repetidos de misma herramienta + mismos parámetros.
- `detectors.knownPollNoProgress`: detecta patrones conocidos de tipo sondeo sin cambio de estado.
- `detectors.pingPong`: detecta patrones alternantes de ping-pong.

## Configuración recomendada

- Empieza con `enabled: true`, dejando los valores predeterminados sin cambios.
- Mantén los umbrales ordenados como `warningThreshold < criticalThreshold < globalCircuitBreakerThreshold`.
- Si se producen falsos positivos:
  - aumenta `warningThreshold` y/o `criticalThreshold`
  - (opcionalmente) aumenta `globalCircuitBreakerThreshold`
  - desactiva solo el detector que esté causando problemas
  - reduce `historySize` para usar un contexto histórico menos estricto

## Registros y comportamiento esperado

Cuando se detecta un bucle, OpenClaw informa de un evento de bucle y bloquea o atenúa el siguiente ciclo de herramientas según la gravedad.
Esto protege a los usuarios frente a un gasto descontrolado de tokens y bloqueos, al tiempo que preserva el acceso normal a las herramientas.

- Prefiere primero la advertencia y la supresión temporal.
- Escala solo cuando se acumule evidencia repetida.

## Notas

- `tools.loopDetection` se combina con las sobrescrituras a nivel de agente.
- La configuración por agente sobrescribe o amplía por completo los valores globales.
- Si no existe configuración, las barreras de protección permanecen desactivadas.
