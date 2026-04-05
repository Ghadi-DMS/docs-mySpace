---
read_when:
    - Quieres emparejar rápidamente una app móvil de nodo con un gateway
    - Necesitas la salida del código de configuración para compartirla de forma remota/manual
summary: Referencia de la CLI para `openclaw qr` (generar QR de pairing móvil + código de configuración)
title: qr
x-i18n:
    generated_at: "2026-04-05T12:38:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: ee6469334ad09037318f938c7ac609b7d5e3385c0988562501bb02a1bfa411ff
    source_path: cli/qr.md
    workflow: 15
---

# `openclaw qr`

Genera un QR de pairing móvil y un código de configuración a partir de tu configuración actual del Gateway.

## Uso

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --url wss://gateway.example/ws
```

## Opciones

- `--remote`: prioriza `gateway.remote.url`; si no está configurada, `gateway.tailscale.mode=serve|funnel` todavía puede proporcionar la URL pública remota
- `--url <url>`: sobrescribe la URL del gateway usada en la carga útil
- `--public-url <url>`: sobrescribe la URL pública usada en la carga útil
- `--token <token>`: sobrescribe qué token del gateway usa el flujo de bootstrap para autenticarse
- `--password <password>`: sobrescribe qué contraseña del gateway usa el flujo de bootstrap para autenticarse
- `--setup-code-only`: imprime solo el código de configuración
- `--no-ascii`: omite la representación ASCII del QR
- `--json`: emite JSON (`setupCode`, `gatewayUrl`, `auth`, `urlSource`)

## Notas

- `--token` y `--password` son mutuamente excluyentes.
- El propio código de configuración ahora contiene un `bootstrapToken` opaco de corta duración, no el token/contraseña compartido del gateway.
- En el flujo integrado de bootstrap de nodo/operador, el token principal del nodo sigue llegando con `scopes: []`.
- Si la transferencia de bootstrap también emite un token de operador, este permanece limitado a la allowlist de bootstrap: `operator.approvals`, `operator.read`, `operator.talk.secrets`, `operator.write`.
- Las comprobaciones de alcance de bootstrap usan prefijos de rol. Esa allowlist de operador solo satisface solicitudes de operador; los roles que no son de operador siguen necesitando alcances bajo su propio prefijo de rol.
- El pairing móvil falla de forma cerrada para URL de gateway `ws://` públicas o de Tailscale. `ws://` en LAN privada sigue siendo compatible, pero las rutas móviles públicas o de Tailscale deben usar Tailscale Serve/Funnel o una URL de gateway `wss://`.
- Con `--remote`, OpenClaw requiere `gateway.remote.url` o
  `gateway.tailscale.mode=serve|funnel`.
- Con `--remote`, si las credenciales remotas activas efectivas están configuradas como SecretRefs y no pasas `--token` ni `--password`, el comando las resuelve desde la instantánea activa del gateway. Si el gateway no está disponible, el comando falla rápidamente.
- Sin `--remote`, los SecretRefs de autenticación del gateway local se resuelven cuando no se pasa ninguna sobrescritura de autenticación por CLI:
  - `gateway.auth.token` se resuelve cuando la autenticación por token puede prevalecer (modo explícito `gateway.auth.mode="token"` o modo inferido en el que ninguna fuente de contraseña prevalece).
  - `gateway.auth.password` se resuelve cuando la autenticación por contraseña puede prevalecer (modo explícito `gateway.auth.mode="password"` o modo inferido sin un token ganador desde auth/env).
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados (incluidos SecretRefs) y `gateway.auth.mode` no está definido, la resolución del código de configuración falla hasta que el modo se establezca explícitamente.
- Nota sobre desfase de versión del gateway: esta ruta de comando requiere un gateway compatible con `secrets.resolve`; los gateways antiguos devuelven un error de método desconocido.
- Después de escanear, aprueba el pairing del dispositivo con:
  - `openclaw devices list`
  - `openclaw devices approve <requestId>`
