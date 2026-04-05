---
read_when:
    - Agregando soporte de ubicación en nodos o UI de permisos
    - Diseñando permisos de ubicación de Android o comportamiento en primer plano
summary: Comando de ubicación para nodos (`location.get`), modos de permisos y comportamiento en primer plano de Android
title: Comando de ubicación
x-i18n:
    generated_at: "2026-04-05T12:47:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5c691cfe147b0b9b16b3a4984d544c168a46b37f91d55b82b2507407d2011529
    source_path: nodes/location-command.md
    workflow: 15
---

# Comando de ubicación (nodos)

## TL;DR

- `location.get` es un comando de nodo (mediante `node.invoke`).
- Está desactivado por defecto.
- La configuración de la app de Android usa un selector: Off / While Using.
- Interruptor independiente: Precise Location.

## Por qué un selector (y no solo un interruptor)

Los permisos del SO tienen varios niveles. Podemos exponer un selector en la app, pero el SO sigue decidiendo la concesión real.

- iOS/macOS puede exponer **While Using** o **Always** en prompts/configuración del sistema.
- La app de Android actualmente admite solo ubicación en primer plano.
- La ubicación precisa es una concesión aparte (iOS 14+ “Precise”, Android “fine” frente a “coarse”).

El selector en la UI controla el modo que solicitamos; la concesión real vive en la configuración del SO.

## Modelo de configuración

Por dispositivo de nodo:

- `location.enabledMode`: `off | whileUsing`
- `location.preciseEnabled`: bool

Comportamiento de la UI:

- Al seleccionar `whileUsing` se solicita permiso en primer plano.
- Si el SO deniega el nivel solicitado, vuelve al nivel concedido más alto y muestra el estado.

## Mapeo de permisos (`node.permissions`)

Opcional. El nodo de macOS informa `location` mediante el mapa de permisos; iOS/Android pueden omitirlo.

## Comando: `location.get`

Se llama mediante `node.invoke`.

Parámetros (sugeridos):

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

Carga útil de respuesta:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

Errores (códigos estables):

- `LOCATION_DISABLED`: el selector está desactivado.
- `LOCATION_PERMISSION_REQUIRED`: falta el permiso para el modo solicitado.
- `LOCATION_BACKGROUND_UNAVAILABLE`: la app está en segundo plano, pero solo se permite While Using.
- `LOCATION_TIMEOUT`: no se obtuvo una ubicación a tiempo.
- `LOCATION_UNAVAILABLE`: fallo del sistema / no hay proveedores.

## Comportamiento en segundo plano

- La app de Android deniega `location.get` cuando está en segundo plano.
- Mantén OpenClaw abierto al solicitar la ubicación en Android.
- Otras plataformas de nodo pueden comportarse de forma diferente.

## Integración con modelo/herramientas

- Superficie de herramientas: la herramienta `nodes` agrega la acción `location_get` (requiere nodo).
- CLI: `openclaw nodes location get --node <id>`.
- Guía para agentes: llamarlo solo cuando la persona usuaria haya habilitado la ubicación y entienda el alcance.

## Texto UX (sugerido)

- Off: “El uso compartido de ubicación está deshabilitado.”
- While Using: “Solo cuando OpenClaw está abierto.”
- Precise: “Usa la ubicación GPS precisa. Desactívalo para compartir una ubicación aproximada.”
