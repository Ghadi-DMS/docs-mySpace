---
permalink: /security/formal-verification/
read_when:
    - Revisar las garantías o limitaciones de los modelos formales de seguridad
    - Reproducir o actualizar las comprobaciones del modelo de seguridad con TLA+/TLC
summary: Modelos de seguridad verificados por máquina para las rutas de mayor riesgo de OpenClaw.
title: Verificación formal (modelos de seguridad)
x-i18n:
    generated_at: "2026-04-05T12:53:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0f7cd2461dcc00d320a5210e50279d76a7fa84e0830c440398323d75e262a38a
    source_path: security/formal-verification.md
    workflow: 15
---

# Verificación formal (modelos de seguridad)

Esta página hace seguimiento de los **modelos formales de seguridad** de OpenClaw (TLA+/TLC hoy; más según sea necesario).

> Nota: algunos enlaces antiguos pueden referirse al nombre anterior del proyecto.

**Objetivo (norte):** proporcionar un argumento verificado por máquina de que OpenClaw aplica su
política de seguridad prevista (autorización, aislamiento de sesiones, restricción de herramientas y
seguridad ante configuraciones incorrectas), bajo supuestos explícitos.

**Qué es esto (hoy):** una **suite de regresión de seguridad** ejecutable, impulsada por atacantes:

- Cada afirmación tiene una comprobación de modelo ejecutable sobre un espacio de estados finito.
- Muchas afirmaciones tienen un **modelo negativo** emparejado que produce una traza de contraejemplo para una clase de error realista.

**Qué no es esto (todavía):** una prueba de que “OpenClaw es seguro en todos los aspectos” ni de que la implementación completa en TypeScript sea correcta.

## Dónde viven los modelos

Los modelos se mantienen en un repositorio separado: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models).

## Advertencias importantes

- Estos son **modelos**, no la implementación completa en TypeScript. Puede haber divergencias entre el modelo y el código.
- Los resultados están acotados por el espacio de estados explorado por TLC; que salga “verde” no implica seguridad más allá de los supuestos y límites modelados.
- Algunas afirmaciones dependen de supuestos ambientales explícitos (por ejemplo, implementación correcta, entradas de configuración correctas).

## Reproducción de resultados

Hoy, los resultados se reproducen clonando el repositorio de modelos localmente y ejecutando TLC (ver abajo). Una futura iteración podría ofrecer:

- Modelos ejecutados en CI con artefactos públicos (trazas de contraejemplo, registros de ejecución)
- un flujo de trabajo alojado de “ejecutar este modelo” para comprobaciones pequeñas y acotadas

Primeros pasos:

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Java 11+ required (TLC runs on the JVM).
# The repo vendors a pinned `tla2tools.jar` (TLA+ tools) and provides `bin/tlc` + Make targets.

make <target>
```

### Exposición del gateway y mala configuración de gateway abierto

**Afirmación:** vincular más allá del loopback sin autenticación puede hacer posible un compromiso remoto / aumentar la exposición; el token/la contraseña bloquea a atacantes no autenticados (según los supuestos del modelo).

- Ejecuciones verdes:
  - `make gateway-exposure-v2`
  - `make gateway-exposure-v2-protected`
- Rojo (esperado):
  - `make gateway-exposure-v2-negative`

Ver también: `docs/gateway-exposure-matrix.md` en el repositorio de modelos.

### Pipeline de ejecución de nodos (capacidad de mayor riesgo)

**Afirmación:** `exec host=node` requiere (a) una allowlist de comandos de nodo más los comandos declarados y (b) aprobación en vivo cuando esté configurada; las aprobaciones están tokenizadas para evitar la repetición (en el modelo).

- Ejecuciones verdes:
  - `make nodes-pipeline`
  - `make approvals-token`
- Rojo (esperado):
  - `make nodes-pipeline-negative`
  - `make approvals-token-negative`

### Almacén de emparejamiento (restricción de MD)

**Afirmación:** las solicitudes de emparejamiento respetan el TTL y los límites de solicitudes pendientes.

- Ejecuciones verdes:
  - `make pairing`
  - `make pairing-cap`
- Rojo (esperado):
  - `make pairing-negative`
  - `make pairing-cap-negative`

### Restricción de entrada (menciones + bypass de comandos de control)

**Afirmación:** en contextos de grupo que requieren mención, un “comando de control” no autorizado no puede omitir la restricción por mención.

- Verde:
  - `make ingress-gating`
- Rojo (esperado):
  - `make ingress-gating-negative`

### Aislamiento de routing/clave de sesión

**Afirmación:** los MD de pares distintos no colapsan en la misma sesión a menos que estén explícitamente vinculados/configurados.

- Verde:
  - `make routing-isolation`
- Rojo (esperado):
  - `make routing-isolation-negative`

## v1++: modelos acotados adicionales (concurrencia, reintentos, corrección de trazas)

Estos son modelos complementarios que aumentan la fidelidad en torno a modos de fallo del mundo real (actualizaciones no atómicas, reintentos y fan-out de mensajes).

### Concurrencia / idempotencia del almacén de emparejamiento

**Afirmación:** un almacén de emparejamiento debe aplicar `MaxPending` e idempotencia incluso bajo intercalaciones (es decir, “comprobar y luego escribir” debe ser atómico / bloqueado; la actualización no debería crear duplicados).

Qué significa:

- Bajo solicitudes concurrentes, no puedes superar `MaxPending` para un canal.
- Las solicitudes/reactualizaciones repetidas para el mismo `(channel, sender)` no deben crear filas pendientes activas duplicadas.

- Ejecuciones verdes:
  - `make pairing-race` (comprobación de límite atómica/bloqueada)
  - `make pairing-idempotency`
  - `make pairing-refresh`
  - `make pairing-refresh-race`
- Rojo (esperado):
  - `make pairing-race-negative` (condición de carrera de límite begin/commit no atómica)
  - `make pairing-idempotency-negative`
  - `make pairing-refresh-negative`
  - `make pairing-refresh-race-negative`

### Correlación de trazas / idempotencia de entrada

**Afirmación:** la ingesta debe conservar la correlación de trazas a través del fan-out y ser idempotente ante reintentos del proveedor.

Qué significa:

- Cuando un evento externo se convierte en varios mensajes internos, cada parte conserva la misma identidad de traza/evento.
- Los reintentos no provocan procesamiento doble.
- Si faltan los ID de eventos del proveedor, la deduplicación recurre a una clave segura (por ejemplo, ID de traza) para evitar descartar eventos distintos.

- Verde:
  - `make ingress-trace`
  - `make ingress-trace2`
  - `make ingress-idempotency`
  - `make ingress-dedupe-fallback`
- Rojo (esperado):
  - `make ingress-trace-negative`
  - `make ingress-trace2-negative`
  - `make ingress-idempotency-negative`
  - `make ingress-dedupe-fallback-negative`

### Precedencia de dmScope en routing + identityLinks

**Afirmación:** el routing debe mantener las sesiones de MD aisladas de forma predeterminada y solo colapsar sesiones cuando se configure explícitamente (precedencia por canal + enlaces de identidad).

Qué significa:

- Las anulaciones de dmScope específicas por canal deben prevalecer sobre los valores predeterminados globales.
- `identityLinks` solo debe colapsar dentro de grupos vinculados explícitamente, no entre pares no relacionados.

- Verde:
  - `make routing-precedence`
  - `make routing-identitylinks`
- Rojo (esperado):
  - `make routing-precedence-negative`
  - `make routing-identitylinks-negative`
