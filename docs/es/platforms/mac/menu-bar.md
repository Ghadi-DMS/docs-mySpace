---
read_when:
    - Ajustar la UI del menú de Mac o la lógica de estado
summary: Lógica de estado de la barra de menús y lo que se muestra a los usuarios
title: Barra de menús
x-i18n:
    generated_at: "2026-04-05T12:48:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8eb73c0e671a76aae4ebb653c65147610bf3e6d3c9c0943d150e292e7761d16d
    source_path: platforms/mac/menu-bar.md
    workflow: 15
---

# Lógica de estado de la barra de menús

## Qué se muestra

- Mostramos el estado actual de trabajo del agente en el icono de la barra de menús y en la primera fila de estado del menú.
- El estado de salud se oculta mientras hay trabajo activo; vuelve cuando todas las sesiones están inactivas.
- El bloque “Nodes” del menú enumera solo **dispositivos** (nodes emparejados mediante `node.list`), no entradas de cliente/presencia.
- Aparece una sección “Usage” en Context cuando hay disponibles instantáneas de uso del proveedor.

## Modelo de estado

- Sesiones: los eventos llegan con `runId` (por ejecución) más `sessionKey` en la carga útil. La sesión “main” es la clave `main`; si falta, recurrimos a la sesión actualizada más recientemente.
- Prioridad: main siempre gana. Si main está activa, su estado se muestra inmediatamente. Si main está inactiva, se muestra la sesión no principal activa más reciente. No cambiamos constantemente a mitad de actividad; solo cambiamos cuando la sesión actual pasa a inactiva o main se activa.
- Tipos de actividad:
  - `job`: ejecución de comandos de alto nivel (`state: started|streaming|done|error`).
  - `tool`: `phase: start|result` con `toolName` y `meta/args`.

## Enumeración IconState (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (anulación de depuración)

### ActivityKind → glifo

- `exec` → 💻
- `read` → 📄
- `write` → ✍️
- `edit` → 📝
- `attach` → 📎
- predeterminado → 🛠️

### Asignación visual

- `idle`: criatura normal.
- `workingMain`: insignia con glifo, tinte completo, animación de pata “working”.
- `workingOther`: insignia con glifo, tinte atenuado, sin desplazamiento.
- `overridden`: usa el glifo/tinte elegido independientemente de la actividad.

## Texto de la fila de estado (menú)

- Mientras hay trabajo activo: `<Session role> · <activity label>`
  - Ejemplos: `Main · exec: pnpm test`, `Other · read: apps/macos/Sources/OpenClaw/AppState.swift`.
- Cuando está inactiva: vuelve al resumen de salud.

## Ingesta de eventos

- Origen: eventos `agent` del canal de control (`ControlChannel.handleAgentEvent`).
- Campos analizados:
  - `stream: "job"` con `data.state` para inicio/parada.
  - `stream: "tool"` con `data.phase`, `name`, `meta`/`args` opcionales.
- Etiquetas:
  - `exec`: primera línea de `args.command`.
  - `read`/`write`: ruta abreviada.
  - `edit`: ruta más tipo de cambio inferido a partir de `meta`/conteos de diff.
  - respaldo: nombre de la herramienta.

## Anulación de depuración

- Configuración ▸ Depuración ▸ selector “Icon override”:
  - `System (auto)` (predeterminado)
  - `Working: main` (por tipo de herramienta)
  - `Working: other` (por tipo de herramienta)
  - `Idle`
- Se almacena mediante `@AppStorage("iconOverride")`; se asigna a `IconState.overridden`.

## Lista de comprobación de pruebas

- Activa un job de sesión principal: verifica que el icono cambie inmediatamente y que la fila de estado muestre la etiqueta principal.
- Activa un job de sesión no principal mientras main está inactiva: icono/estado muestran la no principal; se mantienen estables hasta que termina.
- Inicia main mientras otra está activa: el icono cambia a main al instante.
- Ráfagas rápidas de herramientas: asegúrate de que la insignia no parpadee (gracia TTL en resultados de herramientas).
- La fila de salud reaparece una vez que todas las sesiones están inactivas.
