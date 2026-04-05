---
read_when:
    - Todavía usas `openclaw daemon ...` en scripts
    - Necesitas comandos de ciclo de vida del servicio (install/start/stop/restart/status)
summary: Referencia de CLI para `openclaw daemon` (alias heredado para la administración del servicio Gateway)
title: daemon
x-i18n:
    generated_at: "2026-04-05T12:37:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 91fdaf3c4f3e7dd4dff86f9b74a653dcba2674573698cf51efc4890077994169
    source_path: cli/daemon.md
    workflow: 15
---

# `openclaw daemon`

Alias heredado para comandos de administración del servicio Gateway.

`openclaw daemon ...` se asigna a la misma superficie de control de servicio que los comandos de servicio `openclaw gateway ...`.

## Uso

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Subcomandos

- `status`: mostrar el estado de instalación del servicio y sondear el estado del Gateway
- `install`: instalar el servicio (`launchd`/`systemd`/`schtasks`)
- `uninstall`: eliminar el servicio
- `start`: iniciar el servicio
- `stop`: detener el servicio
- `restart`: reiniciar el servicio

## Opciones comunes

- `status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
- `install`: `--port`, `--runtime <node|bun>`, `--token`, `--force`, `--json`
- ciclo de vida (`uninstall|start|stop|restart`): `--json`

Notas:

- `status` resuelve SecretRef de autenticación configurados para la autenticación del sondeo cuando es posible.
- Si un SecretRef de autenticación requerido no está resuelto en esta ruta de comando, `daemon status --json` informa `rpc.authWarning` cuando falla la conectividad/autenticación del sondeo; pasa `--token`/`--password` explícitamente o resuelve primero el origen del secreto.
- Si el sondeo se realiza correctamente, las advertencias de auth-ref no resueltas se omiten para evitar falsos positivos.
- `status --deep` añade un escaneo del servicio a nivel de sistema en la medida de lo posible. Cuando encuentra otros servicios similares a gateway, la salida legible para humanos muestra sugerencias de limpieza y advierte que la recomendación normal sigue siendo un gateway por máquina.
- En instalaciones Linux con systemd, las comprobaciones de deriva de token de `status` incluyen tanto las fuentes `Environment=` como `EnvironmentFile=` de la unidad.
- Las comprobaciones de deriva resuelven SecretRef de `gateway.auth.token` usando el entorno de ejecución combinado (primero el entorno de comando del servicio, luego el entorno del proceso como respaldo).
- Si la autenticación por token no está efectivamente activa (modo `gateway.auth.mode` explícito de `password`/`none`/`trusted-proxy`, o modo no definido donde puede imponerse password y ningún candidato de token puede imponerse), las comprobaciones de deriva de token omiten la resolución del token de configuración.
- Cuando la autenticación por token requiere un token y `gateway.auth.token` está administrado por SecretRef, `install` valida que el SecretRef pueda resolverse pero no conserva el token resuelto en los metadatos del entorno del servicio.
- Si la autenticación por token requiere un token y el SecretRef de token configurado no está resuelto, la instalación falla de forma cerrada.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está definido, la instalación se bloquea hasta que el modo se establezca explícitamente.
- Si intencionalmente ejecutas varios gateway en un mismo host, aísla puertos, configuración/estado y espacios de trabajo; consulta [/gateway#multiple-gateways-same-host](/gateway#multiple-gateways-same-host).

## Preferencia

Usa [`openclaw gateway`](/cli/gateway) para la documentación y los ejemplos actuales.
