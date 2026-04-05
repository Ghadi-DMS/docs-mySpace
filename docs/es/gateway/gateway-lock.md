---
read_when:
    - Ejecutas o depuras el proceso del gateway
    - Investigas la aplicación de instancia única
summary: Bloqueo singleton del Gateway usando el bind del listener WebSocket
title: Bloqueo del Gateway
x-i18n:
    generated_at: "2026-04-05T12:41:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 726c687ab53f2dd1e46afed8fc791b55310a5c1e62f79a0e38a7dc4ca7576093
    source_path: gateway/gateway-lock.md
    workflow: 15
---

# Bloqueo del Gateway

## Por qué

- Garantizar que solo se ejecute una instancia del gateway por puerto base en el mismo host; los gateways adicionales deben usar perfiles aislados y puertos únicos.
- Sobrevivir a fallos/SIGKILL sin dejar archivos de bloqueo obsoletos.
- Fallar rápidamente con un error claro cuando el puerto de control ya está ocupado.

## Mecanismo

- El gateway vincula el listener WebSocket (predeterminado `ws://127.0.0.1:18789`) inmediatamente al iniciar usando un listener TCP exclusivo.
- Si el bind falla con `EADDRINUSE`, el inicio lanza `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- El sistema operativo libera el listener automáticamente al salir cualquier proceso, incluidos fallos y SIGKILL; no se necesita un archivo de bloqueo independiente ni un paso de limpieza.
- Al apagarse, el gateway cierra el servidor WebSocket y el servidor HTTP subyacente para liberar el puerto con rapidez.

## Superficie de error

- Si otro proceso mantiene el puerto, el inicio lanza `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- Otros fallos de bind aparecen como `GatewayLockError("failed to bind gateway socket on ws://127.0.0.1:<port>: …")`.

## Notas operativas

- Si el puerto está ocupado por _otro_ proceso, el error es el mismo; libera el puerto o elige otro con `openclaw gateway --port <port>`.
- La app de macOS sigue manteniendo su propia protección ligera por PID antes de iniciar el gateway; el bloqueo de tiempo de ejecución se aplica mediante el bind de WebSocket.

## Relacionado

- [Multiple Gateways](/gateway/multiple-gateways) — ejecutar varias instancias con puertos únicos
- [Troubleshooting](/gateway/troubleshooting) — diagnosticar `EADDRINUSE` y conflictos de puertos
