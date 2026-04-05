---
read_when:
    - Cambiar el comportamiento del icono de la barra de menús
summary: Estados y animaciones del icono de la barra de menús para OpenClaw en macOS
title: Icono de la barra de menús
x-i18n:
    generated_at: "2026-04-05T12:48:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: a67a6e6bbdc2b611ba365d3be3dd83f9e24025d02366bc35ffcce9f0b121872b
    source_path: platforms/mac/icon.md
    workflow: 15
---

# Estados del icono de la barra de menús

Autor: steipete · Actualizado: 2025-12-06 · Alcance: app de macOS (`apps/macos`)

- **Inactivo:** animación normal del icono (parpadeo, algún meneo ocasional).
- **En pausa:** el elemento de estado usa `appearsDisabled`; sin movimiento.
- **Disparador de voz (orejas grandes):** el detector de Voice wake llama a `AppState.triggerVoiceEars(ttl: nil)` cuando se oye la palabra de activación, manteniendo `earBoostActive=true` mientras se captura la locución. Las orejas se agrandan (1.9x), obtienen agujeros circulares para mejorar la legibilidad y luego descienden mediante `stopVoiceEars()` tras 1 s de silencio. Solo se activa desde el pipeline de voz dentro de la app.
- **Trabajando (agente en ejecución):** `AppState.isWorking=true` activa un micromovimiento de “revoloteo de cola/patas”: meneo más rápido de las patas y un ligero desplazamiento mientras el trabajo está en curso. Actualmente se activa alrededor de las ejecuciones del agente en WebChat; añade el mismo cambio alrededor de otras tareas largas cuando las conectes.

Puntos de conexión

- Voice wake: el runtime/tester llama a `AppState.triggerVoiceEars(ttl: nil)` al activarse y a `stopVoiceEars()` tras 1 s de silencio para que coincida con la ventana de captura.
- Actividad del agente: establece `AppStateStore.shared.setWorking(true/false)` alrededor de los intervalos de trabajo (ya hecho en la llamada del agente de WebChat). Mantén los intervalos cortos y restablécelos en bloques `defer` para evitar animaciones atascadas.

Formas y tamaños

- El icono base se dibuja en `CritterIconRenderer.makeIcon(blink:legWiggle:earWiggle:earScale:earHoles:)`.
- La escala de las orejas usa `1.0` de forma predeterminada; el refuerzo de voz establece `earScale=1.9` y activa `earHoles=true` sin cambiar el marco general (imagen de plantilla de 18×18 pt renderizada en un backing store Retina de 36×36 px).
- El movimiento rápido usa un meneo de patas de hasta ~1.0 con un pequeño balanceo horizontal; se suma a cualquier meneo inactivo existente.

Notas de comportamiento

- No hay activación externa por CLI/broker para orejas/trabajo; mantenlo interno a las propias señales de la app para evitar oscilaciones accidentales.
- Mantén TTLs cortos (&lt;10s) para que el icono vuelva rápidamente a la línea base si una tarea se cuelga.
