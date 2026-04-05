---
read_when:
    - Diagnosticas la rotación de perfiles de autenticación, tiempos de enfriamiento o el comportamiento de respaldo de modelos
    - Actualizas reglas de conmutación por error para perfiles de autenticación o modelos
    - Quieres entender cómo las anulaciones de modelo de sesión interactúan con los reintentos por respaldo
summary: Cómo OpenClaw rota perfiles de autenticación y recurre a modelos alternativos
title: Conmutación por error de modelos
x-i18n:
    generated_at: "2026-04-05T12:40:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 899041aa0854e4f347343797649fd11140a01e069e88b1fbc0a76e6b375f6c96
    source_path: concepts/model-failover.md
    workflow: 15
---

# Conmutación por error de modelos

OpenClaw maneja los fallos en dos etapas:

1. **Rotación de perfiles de autenticación** dentro del proveedor actual.
2. **Respaldo de modelo** al siguiente modelo en `agents.defaults.model.fallbacks`.

Este documento explica las reglas de tiempo de ejecución y los datos que las respaldan.

## Flujo de tiempo de ejecución

Para una ejecución de texto normal, OpenClaw evalúa los candidatos en este orden:

1. El modelo de sesión seleccionado actualmente.
2. Los `agents.defaults.model.fallbacks` configurados en orden.
3. El modelo principal configurado al final cuando la ejecución empezó desde una anulación.

Dentro de cada candidato, OpenClaw intenta la conmutación por error del perfil de autenticación antes de avanzar
al siguiente candidato de modelo.

Secuencia de alto nivel:

1. Resolver el modelo de sesión activo y la preferencia de perfil de autenticación.
2. Construir la cadena de candidatos de modelo.
3. Probar el proveedor actual con reglas de rotación y tiempo de enfriamiento del perfil de autenticación.
4. Si ese proveedor se agota con un error apto para conmutación por error, pasar al siguiente
   candidato de modelo.
5. Conservar la anulación de respaldo seleccionada antes de que comience el reintento para que otros
   lectores de sesión vean el mismo proveedor o modelo que el ejecutor está a punto de usar.
6. Si el candidato de respaldo falla, revertir solo los campos de anulación de sesión que pertenecen al respaldo cuando todavía coincidan con ese candidato fallido.
7. Si todos los candidatos fallan, lanzar un `FallbackSummaryError` con detalles por intento
   y la expiración de tiempo de enfriamiento más próxima cuando se conozca.

Esto es intencionadamente más limitado que "guardar y restaurar toda la sesión". El
ejecutor de respuestas solo conserva los campos de selección de modelo que le pertenecen para el respaldo:

- `providerOverride`
- `modelOverride`
- `authProfileOverride`
- `authProfileOverrideSource`
- `authProfileOverrideCompactionCount`

Eso evita que un reintento de respaldo fallido sobrescriba mutaciones más recientes y no relacionadas de la sesión,
como cambios manuales con `/model` o actualizaciones de rotación de sesión que
ocurrieron mientras el intento se estaba ejecutando.

## Almacenamiento de autenticación (claves + OAuth)

OpenClaw usa **perfiles de autenticación** tanto para claves de API como para tokens OAuth.

- Los secretos viven en `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (heredado: `~/.openclaw/agent/auth-profiles.json`).
- La configuración `auth.profiles` / `auth.order` es **solo metadatos + enrutamiento** (sin secretos).
- Archivo OAuth heredado solo para importación: `~/.openclaw/credentials/oauth.json` (importado a `auth-profiles.json` en el primer uso).

Más detalle: [/concepts/oauth](/concepts/oauth)

Tipos de credenciales:

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` para algunos proveedores)

## IDs de perfil

Los inicios de sesión OAuth crean perfiles distintos para que puedan coexistir varias cuentas.

- Predeterminado: `provider:default` cuando no hay correo electrónico disponible.
- OAuth con correo electrónico: `provider:<email>` (por ejemplo `google-antigravity:user@gmail.com`).

Los perfiles viven en `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` bajo `profiles`.

## Orden de rotación

Cuando un proveedor tiene varios perfiles, OpenClaw elige un orden así:

1. **Configuración explícita**: `auth.order[provider]` (si está configurado).
2. **Perfiles configurados**: `auth.profiles` filtrados por proveedor.
3. **Perfiles almacenados**: entradas en `auth-profiles.json` para el proveedor.

Si no se configura un orden explícito, OpenClaw usa un orden round-robin:

- **Clave principal:** tipo de perfil (**OAuth antes que claves de API**).
- **Clave secundaria:** `usageStats.lastUsed` (el más antiguo primero, dentro de cada tipo).
- Los perfiles en enfriamiento o deshabilitados se mueven al final, ordenados por la expiración más próxima.

### Persistencia por sesión (favorable para caché)

OpenClaw **fija el perfil de autenticación elegido por sesión** para mantener calientes las cachés del proveedor.
**No** rota en cada solicitud. El perfil fijado se reutiliza hasta que:

- se reinicia la sesión (`/new` / `/reset`)
- se completa una compactación (se incrementa el recuento de compactación)
- el perfil está en enfriamiento o deshabilitado

La selección manual mediante `/model …@<profileId>` establece una **anulación del usuario** para esa sesión
y no se rota automáticamente hasta que comienza una nueva sesión.

Los perfiles fijados automáticamente (seleccionados por el enrutador de sesiones) se tratan como una **preferencia**:
se prueban primero, pero OpenClaw puede rotar a otro perfil ante límites de tasa o tiempos de espera.
Los perfiles fijados por el usuario permanecen bloqueados a ese perfil; si falla y hay respaldos de modelo
configurados, OpenClaw pasa al siguiente modelo en lugar de cambiar de perfil.

### Por qué OAuth puede "parecer perdido"

Si tienes tanto un perfil OAuth como un perfil de clave de API para el mismo proveedor, el round-robin puede alternar entre ellos entre mensajes a menos que estén fijados. Para forzar un solo perfil:

- Fíjalo con `auth.order[provider] = ["provider:profileId"]`, o
- Usa una anulación por sesión mediante `/model …` con una anulación de perfil (cuando tu UI/superficie de chat la admita).

## Tiempos de enfriamiento

Cuando un perfil falla debido a errores de autenticación o límite de tasa (o un tiempo de espera que parece
un límite de tasa), OpenClaw lo marca en enfriamiento y pasa al siguiente perfil.
Ese grupo de límites de tasa es más amplio que un simple `429`: también incluye mensajes del proveedor
como `Too many concurrent requests`, `ThrottlingException`,
`concurrency limit reached`, `workers_ai ... quota limit exceeded`,
`throttled`, `resource exhausted` y límites periódicos de ventana de uso como
`weekly/monthly limit reached`.
Los errores de formato o solicitud no válida (por ejemplo, fallos de validación de ID de llamada de herramienta de Cloud Code Assist) se tratan como aptos para conmutación por error y usan los mismos tiempos de enfriamiento.
Los errores de razón de parada compatibles con OpenAI, como `Unhandled stop reason: error`,
`stop reason: error` y `reason: error`, se clasifican como
señales de tiempo de espera/conmutación por error.
El texto genérico de servidor con alcance de proveedor también puede acabar en ese grupo de tiempo de espera cuando
la fuente coincide con un patrón transitorio conocido. Por ejemplo, el
mensaje simple de Anthropic `An unknown error occurred` y las cargas útiles JSON `api_error` con texto transitorio del servidor
como `internal server error`, `unknown error, 520`, `upstream error`
o `backend error` se tratan como aptos para conmutación por error por tiempo de espera. El texto genérico de upstream específico de OpenRouter, como `Provider returned error`, también se trata como
tiempo de espera solo cuando el contexto del proveedor es realmente OpenRouter. El texto genérico de respaldo interno como `LLM request failed with an unknown error.` se mantiene
conservador y no activa la conmutación por error por sí solo.

Los tiempos de enfriamiento por límite de tasa también pueden tener alcance de modelo:

- OpenClaw registra `cooldownModel` para fallos por límite de tasa cuando se conoce
  el ID del modelo que falló.
- Aun así se puede probar un modelo hermano del mismo proveedor cuando el enfriamiento está
  limitado a un modelo diferente.
- Las ventanas de facturación o deshabilitación siguen bloqueando todo el perfil entre modelos.

Los tiempos de enfriamiento usan retroceso exponencial:

- 1 minuto
- 5 minutos
- 25 minutos
- 1 hora (límite)

El estado se almacena en `auth-profiles.json` bajo `usageStats`:

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## Deshabilitaciones por facturación

Los fallos por facturación o crédito (por ejemplo “insufficient credits” / “credit balance too low”) se tratan como aptos para conmutación por error, pero normalmente no son transitorios. En lugar de un tiempo de enfriamiento corto, OpenClaw marca el perfil como **deshabilitado** (con un retroceso más largo) y rota al siguiente perfil o proveedor.

No todas las respuestas con forma de facturación son `402`, y no todos los `402` HTTP terminan
aquí. OpenClaw mantiene el texto explícito de facturación en la ruta de facturación incluso cuando un
proveedor devuelve `401` o `403`, pero los comparadores específicos del proveedor siguen teniendo
alcance solo para el proveedor al que pertenecen (por ejemplo, OpenRouter `403 Key limit
exceeded`). Mientras tanto, los límites temporales de uso y
límites de gasto de organización o espacio de trabajo con `402` se clasifican como `rate_limit` cuando
el mensaje parece reintentable (por ejemplo `weekly usage limit exhausted`, `daily
limit reached, resets tomorrow` o `organization spending limit exceeded`).
Esos permanecen en la ruta de enfriamiento corto y conmutación por error en lugar de la ruta larga
de deshabilitación por facturación.

El estado se almacena en `auth-profiles.json`:

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

Valores predeterminados:

- El retroceso por facturación empieza en **5 horas**, se duplica por cada fallo de facturación y tiene un máximo de **24 horas**.
- Los contadores de retroceso se reinician si el perfil no ha fallado durante **24 horas** (configurable).
- Los reintentos por sobrecarga permiten **1 rotación de perfil del mismo proveedor** antes del respaldo de modelo.
- Los reintentos por sobrecarga usan **0 ms de retroceso** de forma predeterminada.

## Respaldo de modelo

Si fallan todos los perfiles de un proveedor, OpenClaw pasa al siguiente modelo en
`agents.defaults.model.fallbacks`. Esto se aplica a fallos de autenticación, límites de tasa y
tiempos de espera que agotaron la rotación de perfiles (otros errores no hacen avanzar el respaldo).

Los errores de sobrecarga y límite de tasa se manejan de forma más agresiva que los tiempos de enfriamiento
por facturación. De forma predeterminada, OpenClaw permite un reintento de perfil de autenticación del mismo proveedor
y luego cambia al siguiente respaldo de modelo configurado sin esperar.
Las señales de proveedor ocupado como `ModelNotReadyException` entran en ese grupo de sobrecarga.
Ajusta esto con `auth.cooldowns.overloadedProfileRotations`,
`auth.cooldowns.overloadedBackoffMs` y
`auth.cooldowns.rateLimitedProfileRotations`.

Cuando una ejecución empieza con una anulación de modelo (hooks o CLI), los respaldos siguen terminando en
`agents.defaults.model.primary` después de probar cualquier respaldo configurado.

### Reglas de la cadena de candidatos

OpenClaw construye la lista de candidatos a partir del `provider/model` solicitado actualmente
más los respaldos configurados.

Reglas:

- El modelo solicitado siempre va primero.
- Los respaldos configurados explícitamente se deduplican, pero no se filtran por la lista de permitidos de modelos. Se tratan como intención explícita del operador.
- Si la ejecución actual ya está en un respaldo configurado de la misma familia de proveedor,
  OpenClaw sigue usando la cadena configurada completa.
- Si la ejecución actual está en un proveedor distinto del de la configuración y ese modelo actual no
  forma ya parte de la cadena de respaldos configurada, OpenClaw no
  añade respaldos configurados no relacionados de otro proveedor.
- Cuando la ejecución empezó desde una anulación, el modelo principal configurado se añade al
  final para que la cadena pueda volver al valor predeterminado normal una vez se agoten
  los candidatos anteriores.

### Qué errores hacen avanzar el respaldo

El respaldo de modelo continúa con:

- fallos de autenticación
- límites de tasa y agotamiento del tiempo de enfriamiento
- errores de sobrecarga/proveedor ocupado
- errores de conmutación por error con forma de tiempo de espera
- deshabilitaciones por facturación
- `LiveSessionModelSwitchError`, que se normaliza en una ruta de conmutación por error para que un
  modelo persistido obsoleto no cree un bucle de reintento externo
- otros errores no reconocidos cuando aún quedan candidatos

El respaldo de modelo no continúa con:

- abortos explícitos que no tienen forma de tiempo de espera o conmutación por error
- errores de desbordamiento de contexto que deben permanecer dentro de la lógica de compactación/reintento
  (por ejemplo `request_too_large`, `INVALID_ARGUMENT: input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `The input is too long for the model` o `ollama error: context
length exceeded`)
- un error desconocido final cuando ya no quedan candidatos

### Omitir por enfriamiento frente a comportamiento de sondeo

Cuando todos los perfiles de autenticación de un proveedor ya están en enfriamiento, OpenClaw no
omite automáticamente ese proveedor para siempre. Toma una decisión por candidato:

- Los fallos persistentes de autenticación omiten todo el proveedor inmediatamente.
- Las deshabilitaciones por facturación normalmente se omiten, pero el candidato principal aún puede sondearse
  con limitación para que la recuperación sea posible sin reiniciar.
- El candidato principal puede sondearse cerca de la expiración del enfriamiento, con una limitación por proveedor.
- Los modelos hermanos del mismo proveedor pueden intentarse a pesar del enfriamiento cuando el
  fallo parece transitorio (`rate_limit`, `overloaded` o desconocido). Esto es
  especialmente relevante cuando un límite de tasa tiene alcance de modelo y un modelo hermano puede
  recuperarse de inmediato.
- Los sondeos transitorios en enfriamiento se limitan a uno por proveedor en cada ejecución de respaldo para que
  un solo proveedor no bloquee el respaldo entre proveedores.

## Anulaciones de sesión y cambio de modelo en vivo

Los cambios de modelo de sesión son estado compartido. El ejecutor activo, el comando `/model`,
la compactación/actualizaciones de sesión y la reconciliación de sesión en vivo leen o escriben
partes de la misma entrada de sesión.

Eso significa que los reintentos de respaldo deben coordinarse con el cambio de modelo en vivo:

- Solo los cambios de modelo explícitos iniciados por el usuario marcan un cambio pendiente en vivo. Eso
  incluye `/model`, `session_status(model=...)` y `sessions.patch`.
- Los cambios de modelo impulsados por el sistema, como rotación de respaldo,
  anulaciones de heartbeat o compactación, nunca marcan por sí mismos un cambio pendiente en vivo.
- Antes de que empiece un reintento de respaldo, el ejecutor de respuestas conserva los campos de anulación
  de respaldo seleccionados en la entrada de sesión.
- La reconciliación de sesión en vivo prefiere las anulaciones persistidas de la sesión frente a
  campos obsoletos del modelo de tiempo de ejecución.
- Si el intento de respaldo falla, el ejecutor revierte solo los campos de anulación
  que escribió, y solo si todavía coinciden con ese candidato fallido.

Esto evita la carrera clásica:

1. El primario falla.
2. Se elige en memoria un candidato de respaldo.
3. El almacén de sesión aún indica el primario antiguo.
4. La reconciliación de sesión en vivo lee el estado obsoleto de la sesión.
5. El reintento vuelve al modelo antiguo antes de que comience el intento de respaldo.

La anulación persistida de respaldo cierra esa ventana, y la reversión estrecha
mantiene intactos los cambios más recientes de sesión manuales o de tiempo de ejecución.

## Observabilidad y resúmenes de fallo

`runWithModelFallback(...)` registra detalles por intento que alimentan los logs y
los mensajes de tiempo de enfriamiento orientados al usuario:

- proveedor/modelo intentado
- motivo (`rate_limit`, `overloaded`, `billing`, `auth`, `model_not_found` y
  motivos similares de conmutación por error)
- estado/código opcional
- resumen de error legible para humanos

Cuando todos los candidatos fallan, OpenClaw lanza `FallbackSummaryError`. El
ejecutor externo de respuestas puede usarlo para construir un mensaje más específico como "todos los modelos
están temporalmente limitados por tasa" e incluir la expiración de enfriamiento más próxima cuando se conozca.

Ese resumen de enfriamiento tiene en cuenta el modelo:

- se ignoran límites de tasa de otros modelos no relacionados para la cadena
  proveedor/modelo intentada
- si el bloqueo restante es un límite de tasa con alcance de modelo coincidente, OpenClaw
  informa la última expiración coincidente que todavía bloquea ese modelo

## Configuración relacionada

Consulta [Gateway configuration](/gateway/configuration) para:

- `auth.profiles` / `auth.order`
- `auth.cooldowns.billingBackoffHours` / `auth.cooldowns.billingBackoffHoursByProvider`
- `auth.cooldowns.billingMaxHours` / `auth.cooldowns.failureWindowHours`
- `auth.cooldowns.overloadedProfileRotations` / `auth.cooldowns.overloadedBackoffMs`
- `auth.cooldowns.rateLimitedProfileRotations`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` routing

Consulta [Models](/concepts/models) para ver el resumen general de selección de modelo y respaldo.
