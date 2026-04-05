---
read_when:
    - Estás aprobando solicitudes de emparejamiento de dispositivos
    - Necesitas rotar o revocar tokens de dispositivos
summary: Referencia de CLI para `openclaw devices` (emparejamiento de dispositivos + rotación/revocación de tokens)
title: devices
x-i18n:
    generated_at: "2026-04-05T12:37:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: e2f9fcb8e3508a703590f87caaafd953a5d3557e11c958cbb2be1d67bb8720f4
    source_path: cli/devices.md
    workflow: 15
---

# `openclaw devices`

Gestiona solicitudes de emparejamiento de dispositivos y tokens con alcance de dispositivo.

## Comandos

### `openclaw devices list`

Lista las solicitudes de emparejamiento pendientes y los dispositivos emparejados.

```
openclaw devices list
openclaw devices list --json
```

La salida de solicitudes pendientes incluye el rol y los alcances solicitados para que las aprobaciones
puedan revisarse antes de aprobarlas.

### `openclaw devices remove <deviceId>`

Elimina una entrada de dispositivo emparejado.

Cuando estás autenticado con un token de dispositivo emparejado, los llamadores no administradores pueden
eliminar solo **su propia** entrada de dispositivo. Eliminar otro dispositivo requiere
`operator.admin`.

```
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

### `openclaw devices clear --yes [--pending]`

Borra dispositivos emparejados en bloque.

```
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

### `openclaw devices approve [requestId] [--latest]`

Aprueba una solicitud pendiente de emparejamiento de dispositivo. Si se omite `requestId`, OpenClaw
aprueba automáticamente la solicitud pendiente más reciente.

Nota: si un dispositivo reintenta el emparejamiento con detalles de autenticación modificados (rol, alcances o clave
pública), OpenClaw reemplaza la entrada pendiente anterior y emite un nuevo
`requestId`. Ejecuta `openclaw devices list` justo antes de aprobar para usar el
ID actual.

```
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

### `openclaw devices reject <requestId>`

Rechaza una solicitud pendiente de emparejamiento de dispositivo.

```
openclaw devices reject <requestId>
```

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

Rota un token de dispositivo para un rol específico (opcionalmente actualizando alcances).
El rol de destino ya debe existir en el contrato aprobado de emparejamiento de ese dispositivo;
la rotación no puede emitir un rol nuevo no aprobado.
Si omites `--scope`, las conexiones posteriores con el token rotado almacenado reutilizan los
alcances aprobados en caché de ese token. Si pasas valores explícitos de `--scope`, esos
se convierten en el conjunto de alcances almacenado para futuras reconexiones con token en caché.
Los llamadores no administradores con dispositivo emparejado pueden rotar solo **su propio** token de dispositivo.
Además, cualquier valor explícito de `--scope` debe mantenerse dentro de los propios
alcances de operador de la sesión del llamador; la rotación no puede emitir un token de operador más amplio que el que el llamador
ya tiene.

```
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

Devuelve la nueva carga útil del token como JSON.

### `openclaw devices revoke --device <id> --role <role>`

Revoca un token de dispositivo para un rol específico.

Los llamadores no administradores con dispositivo emparejado pueden revocar solo **su propio** token de dispositivo.
Revocar el token de otro dispositivo requiere `operator.admin`.

```
openclaw devices revoke --device <deviceId> --role node
```

Devuelve el resultado de la revocación como JSON.

## Opciones comunes

- `--url <url>`: URL de WebSocket del Gateway (usa de forma predeterminada `gateway.remote.url` cuando está configurado).
- `--token <token>`: token del Gateway (si es necesario).
- `--password <password>`: contraseña del Gateway (autenticación por contraseña).
- `--timeout <ms>`: tiempo de espera de RPC.
- `--json`: salida JSON (recomendado para scripts).

Nota: cuando configuras `--url`, la CLI no recurre a credenciales de configuración o entorno.
Pasa `--token` o `--password` explícitamente. La ausencia de credenciales explícitas es un error.

## Notas

- La rotación de token devuelve un token nuevo (sensible). Trátalo como un secreto.
- Estos comandos requieren el alcance `operator.pairing` (o `operator.admin`).
- La rotación de token se mantiene dentro del conjunto de roles de emparejamiento aprobados y de la línea base de alcances
  aprobados para ese dispositivo. Una entrada de token en caché extraviada no concede un nuevo
  destino de rotación.
- Para sesiones con token de dispositivo emparejado, la gestión entre dispositivos es solo para administradores:
  `remove`, `rotate` y `revoke` son solo para el propio dispositivo salvo que el llamador tenga
  `operator.admin`.
- `devices clear` está intencionadamente protegido con `--yes`.
- Si el alcance de emparejamiento no está disponible en local loopback (y no se pasa un `--url` explícito), `list` y `approve` pueden usar una reserva local de emparejamiento.
- `devices approve` selecciona automáticamente la solicitud pendiente más reciente cuando omites `requestId` o pasas `--latest`.

## Lista de verificación para recuperación de deriva de token

Úsala cuando Control UI u otros clientes sigan fallando con `AUTH_TOKEN_MISMATCH` o `AUTH_DEVICE_TOKEN_MISMATCH`.

1. Confirma la fuente actual del token del gateway:

```bash
openclaw config get gateway.auth.token
```

2. Lista los dispositivos emparejados e identifica el id del dispositivo afectado:

```bash
openclaw devices list
```

3. Rota el token de operador del dispositivo afectado:

```bash
openclaw devices rotate --device <deviceId> --role operator
```

4. Si la rotación no es suficiente, elimina el emparejamiento obsoleto y vuelve a aprobar:

```bash
openclaw devices remove <deviceId>
openclaw devices list
openclaw devices approve <requestId>
```

5. Reintenta la conexión del cliente con el token o la contraseña compartidos actuales.

Notas:

- La precedencia normal de autenticación de reconexión es primero token o contraseña compartidos explícitos, luego `deviceToken` explícito, después token de dispositivo almacenado y finalmente token de arranque.
- La recuperación confiable de `AUTH_TOKEN_MISMATCH` puede enviar temporalmente tanto el token compartido como el token de dispositivo almacenado juntos para ese único reintento limitado.

Relacionado:

- [Solución de problemas de autenticación del panel](/web/dashboard#if-you-see-unauthorized-1008)
- [Solución de problemas del gateway](/gateway/troubleshooting#dashboard-control-ui-connectivity)
