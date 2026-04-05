---
read_when:
    - Estás gestionando nodos emparejados (cámaras, pantalla, canvas)
    - Necesitas aprobar solicitudes o invocar comandos de nodo
summary: Referencia de CLI para `openclaw nodes` (estado, emparejamiento, invocación, cámara/canvas/pantalla)
title: nodes
x-i18n:
    generated_at: "2026-04-05T12:38:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1ce3095591c4623ad18e3eca8d8083e5c10266fbf94afea2d025f0ba8093a175
    source_path: cli/nodes.md
    workflow: 15
---

# `openclaw nodes`

Gestiona nodos emparejados (dispositivos) e invoca capacidades de nodo.

Relacionado:

- Resumen de nodos: [Nodes](/nodes)
- Cámara: [Nodos de cámara](/nodes/camera)
- Imágenes: [Nodos de imagen](/nodes/images)

Opciones comunes:

- `--url`, `--token`, `--timeout`, `--json`

## Comandos comunes

```bash
openclaw nodes list
openclaw nodes list --connected
openclaw nodes list --last-connected 24h
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes rename --node <id|name|ip> --name <displayName>
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
```

`nodes list` imprime tablas de nodos pendientes y emparejados. Las filas emparejadas incluyen la antigüedad de la conexión más reciente (Last Connect).
Usa `--connected` para mostrar solo los nodos conectados actualmente. Usa `--last-connected <duration>` para
filtrar a los nodos que se conectaron dentro de una duración determinada (por ejemplo, `24h`, `7d`).

Nota sobre la aprobación:

- `openclaw nodes pending` solo necesita alcance de emparejamiento.
- `openclaw nodes approve <requestId>` hereda requisitos de alcance adicionales de la
  solicitud pendiente:
  - solicitud sin comandos: solo emparejamiento
  - comandos de nodo sin ejecución: emparejamiento + escritura
  - `system.run` / `system.run.prepare` / `system.which`: emparejamiento + administración

## Invocar

```bash
openclaw nodes invoke --node <id|name|ip> --command <command> --params <json>
```

Indicadores de invocación:

- `--params <json>`: cadena de objeto JSON (predeterminado `{}`).
- `--invoke-timeout <ms>`: tiempo de espera de invocación de nodo (predeterminado `15000`).
- `--idempotency-key <key>`: clave de idempotencia opcional.
- `system.run` y `system.run.prepare` están bloqueados aquí; usa la herramienta `exec` con `host=node` para ejecución de shell.

Para la ejecución de shell en un nodo, usa la herramienta `exec` con `host=node` en lugar de `openclaw nodes run`.
La CLI de `nodes` ahora está centrada en capacidades: RPC directa mediante `nodes invoke`, además de emparejamiento, cámara,
pantalla, ubicación, canvas y notificaciones.
