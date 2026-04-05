---
read_when:
    - Estás depurando rechazos de solicitudes del proveedor vinculados a la forma de la transcripción
    - Estás cambiando la lógica de saneamiento de transcripciones o de reparación de llamadas a herramientas
    - Estás investigando discrepancias de ID de llamadas a herramientas entre proveedores
summary: 'Referencia: reglas de saneamiento y reparación de transcripciones específicas del proveedor'
title: Higiene de transcripciones
x-i18n:
    generated_at: "2026-04-05T12:53:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 217afafb693cf89651e8fa361252f7b5c197feb98d20be4697a83e6dedc0ec3f
    source_path: reference/transcript-hygiene.md
    workflow: 15
---

# Higiene de transcripciones (ajustes por proveedor)

Este documento describe las **correcciones específicas del proveedor** aplicadas a las transcripciones antes de una ejecución
(construcción del contexto del modelo). Estos son ajustes **en memoria** usados para satisfacer
requisitos estrictos del proveedor. Estos pasos de higiene **no** reescriben la transcripción JSONL almacenada
en disco; sin embargo, una pasada aparte de reparación del archivo de sesión puede reescribir archivos JSONL con formato incorrecto
eliminando líneas no válidas antes de que se cargue la sesión. Cuando se produce una reparación, el archivo original
se respalda junto al archivo de sesión.

El alcance incluye:

- Saneamiento de ID de llamadas a herramientas
- Validación de entradas de llamadas a herramientas
- Reparación del emparejamiento de resultados de herramientas
- Validación / ordenación de turnos
- Limpieza de firmas de pensamiento
- Saneamiento de cargas útiles de imágenes
- Etiquetado de procedencia de entrada del usuario (para prompts enrutados entre sesiones)

Si necesitas detalles sobre el almacenamiento de transcripciones, consulta:

- [/reference/session-management-compaction](/reference/session-management-compaction)

---

## Dónde se ejecuta esto

Toda la higiene de transcripciones está centralizada en el runner integrado:

- Selección de políticas: `src/agents/transcript-policy.ts`
- Aplicación de saneamiento/reparación: `sanitizeSessionHistory` en `src/agents/pi-embedded-runner/google.ts`

La política usa `provider`, `modelApi` y `modelId` para decidir qué aplicar.

Por separado de la higiene de transcripciones, los archivos de sesión se reparan (si es necesario) antes de cargarse:

- `repairSessionFileIfNeeded` en `src/agents/session-file-repair.ts`
- Llamado desde `run/attempt.ts` y `compact.ts` (runner integrado)

---

## Regla global: saneamiento de imágenes

Las cargas útiles de imágenes siempre se sanean para evitar el rechazo del lado del proveedor debido a límites
de tamaño (reducir escala/recomprimir imágenes base64 sobredimensionadas).

Esto también ayuda a controlar la presión de tokens impulsada por imágenes para modelos con capacidad de visión.
Dimensiones máximas más bajas generalmente reducen el uso de tokens; dimensiones más altas preservan más detalle.

Implementación:

- `sanitizeSessionMessagesImages` en `src/agents/pi-embedded-helpers/images.ts`
- `sanitizeContentBlocksImages` en `src/agents/tool-images.ts`
- El lado máximo de la imagen se puede configurar con `agents.defaults.imageMaxDimensionPx` (predeterminado: `1200`).

---

## Regla global: llamadas a herramientas malformadas

Los bloques de llamadas a herramientas del asistente a los que les faltan tanto `input` como `arguments` se eliminan
antes de que se construya el contexto del modelo. Esto evita rechazos del proveedor por llamadas a herramientas
persistidas parcialmente (por ejemplo, después de un fallo por límite de tasa).

Implementación:

- `sanitizeToolCallInputs` en `src/agents/session-transcript-repair.ts`
- Aplicado en `sanitizeSessionHistory` en `src/agents/pi-embedded-runner/google.ts`

---

## Regla global: procedencia de entradas entre sesiones

Cuando un agente envía un prompt a otra sesión mediante `sessions_send` (incluidos
los pasos de responder/anunciar de agente a agente), OpenClaw persiste el turno de usuario creado con:

- `message.provenance.kind = "inter_session"`

Estos metadatos se escriben en el momento de agregar la transcripción y no cambian el rol
(`role: "user"` se mantiene por compatibilidad con el proveedor). Los lectores de transcripciones pueden usar
esto para evitar tratar los prompts internos enrutados como instrucciones redactadas por el usuario final.

Durante la reconstrucción del contexto, OpenClaw también antepone un breve marcador `[Inter-session message]`
a esos turnos de usuario en memoria para que el modelo pueda distinguirlos de
las instrucciones externas del usuario final.

---

## Matriz de proveedores (comportamiento actual)

**OpenAI / OpenAI Codex**

- Solo saneamiento de imágenes.
- Eliminar firmas de razonamiento huérfanas (elementos de razonamiento independientes sin un bloque de contenido posterior) para transcripciones de OpenAI Responses/Codex.
- Sin saneamiento de ID de llamadas a herramientas.
- Sin reparación del emparejamiento de resultados de herramientas.
- Sin validación ni reordenación de turnos.
- Sin resultados sintéticos de herramientas.
- Sin eliminación de firmas de pensamiento.

**Google (Generative AI / Gemini CLI / Antigravity)**

- Saneamiento de ID de llamadas a herramientas: alfanumérico estricto.
- Reparación del emparejamiento de resultados de herramientas y resultados sintéticos de herramientas.
- Validación de turnos (alternancia de turnos estilo Gemini).
- Ajuste de ordenación de turnos de Google (anteponer un pequeño bootstrap de usuario si el historial comienza con el asistente).
- Claude de Antigravity: normalizar firmas de pensamiento; eliminar bloques de pensamiento sin firma.

**Anthropic / Minimax (compatibles con Anthropic)**

- Reparación del emparejamiento de resultados de herramientas y resultados sintéticos de herramientas.
- Validación de turnos (fusionar turnos consecutivos del usuario para satisfacer la alternancia estricta).

**Mistral (incluida la detección basada en ID de modelo)**

- Saneamiento de ID de llamadas a herramientas: strict9 (alfanumérico de longitud 9).

**OpenRouter Gemini**

- Limpieza de firmas de pensamiento: eliminar valores `thought_signature` que no sean base64 (conservar base64).

**Todo lo demás**

- Solo saneamiento de imágenes.

---

## Comportamiento histórico (anterior a 2026.1.22)

Antes de la versión 2026.1.22, OpenClaw aplicaba múltiples capas de higiene de transcripciones:

- Una **extensión transcript-sanitize** se ejecutaba en cada construcción de contexto y podía:
  - Reparar el emparejamiento de uso/resultado de herramientas.
  - Sanear ID de llamadas a herramientas (incluido un modo no estricto que preservaba `_`/`-`).
- El runner también realizaba saneamiento específico del proveedor, lo que duplicaba trabajo.
- Se producían mutaciones adicionales fuera de la política del proveedor, entre ellas:
  - Eliminar etiquetas `<final>` del texto del asistente antes de persistirlo.
  - Eliminar turnos vacíos de error del asistente.
  - Recortar contenido del asistente después de llamadas a herramientas.

Esta complejidad causó regresiones entre proveedores (especialmente en el emparejamiento
`call_id|fc_id` de `openai-responses`). La limpieza de 2026.1.22 eliminó la extensión, centralizó
la lógica en el runner e hizo que OpenAI fuera de **no intervención** más allá del saneamiento de imágenes.
