---
read_when:
    - Quieres acceder al Gateway mediante Tailscale
    - Quieres la UI de control en el navegador y editar la configuración
summary: 'Superficies web del Gateway: UI de control, modos de bind y seguridad'
title: Web
x-i18n:
    generated_at: "2026-04-05T12:57:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 15f5643283f7d37235d3d8104897f38db27ac5a9fdef6165156fb542d0e7048c
    source_path: web/index.md
    workflow: 15
---

# Web (Gateway)

El Gateway sirve una pequeña **UI de control del navegador** (Vite + Lit) desde el mismo puerto que el WebSocket del Gateway:

- predeterminado: `http://<host>:18789/`
- prefijo opcional: establece `gateway.controlUi.basePath` (por ejemplo, `/openclaw`)

Las capacidades están en [Control UI](/web/control-ui).
Esta página se centra en los modos de bind, la seguridad y las superficies orientadas a la web.

## Webhooks

Cuando `hooks.enabled=true`, el Gateway también expone un pequeño endpoint de webhook en el mismo servidor HTTP.
Consulta [Configuración del Gateway](/es/gateway/configuration) → `hooks` para autenticación + cargas útiles.

## Configuración (activada por defecto)

La Control UI está **activada por defecto** cuando los assets están presentes (`dist/control-ui`).
Puedes controlarla mediante la configuración:

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // `basePath` opcional
  },
}
```

## Acceso con Tailscale

### Serve integrado (recomendado)

Mantén el Gateway en loopback y deja que Tailscale Serve actúe como proxy:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

Luego inicia el gateway:

```bash
openclaw gateway
```

Abre:

- `https://<magicdns>/` (o el `gateway.controlUi.basePath` que hayas configurado)

### Bind en tailnet + token

```json5
{
  gateway: {
    bind: "tailnet",
    controlUi: { enabled: true },
    auth: { mode: "token", token: "your-token" },
  },
}
```

Luego inicia el gateway (este ejemplo sin loopback usa autenticación
con token de secreto compartido):

```bash
openclaw gateway
```

Abre:

- `http://<tailscale-ip>:18789/` (o el `gateway.controlUi.basePath` que hayas configurado)

### Internet pública (Funnel)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password" }, // o OPENCLAW_GATEWAY_PASSWORD
  },
}
```

## Notas de seguridad

- La autenticación del Gateway es obligatoria por defecto (token, contraseña, proxy de confianza o encabezados de identidad de Tailscale Serve cuando está habilitado).
- Los binds sin loopback siguen **requiriendo** autenticación del gateway. En la práctica eso significa autenticación con token/contraseña o un proxy inverso con conocimiento de identidad con `gateway.auth.mode: "trusted-proxy"`.
- El asistente crea autenticación de secreto compartido por defecto y normalmente genera un
  token de gateway (incluso en loopback).
- En modo de secreto compartido, la UI envía `connect.params.auth.token` o
  `connect.params.auth.password`.
- En modos con identidad, como Tailscale Serve o `trusted-proxy`, la
  comprobación de autenticación de WebSocket se satisface desde los encabezados de la solicitud.
- Para despliegues de Control UI sin loopback, establece `gateway.controlUi.allowedOrigins`
  explícitamente (orígenes completos). Sin ello, el inicio del gateway se rechaza por defecto.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` habilita
  el modo de respaldo de origen mediante encabezado Host, pero es una degradación de seguridad peligrosa.
- Con Serve, los encabezados de identidad de Tailscale pueden satisfacer la autenticación de la Control UI/WebSocket
  cuando `gateway.auth.allowTailscale` es `true` (no se requiere token/contraseña).
  Los endpoints de la API HTTP no usan esos encabezados de identidad de Tailscale; siguen
  el modo normal de autenticación HTTP del gateway. Establece
  `gateway.auth.allowTailscale: false` para requerir credenciales explícitas. Consulta
  [Tailscale](/es/gateway/tailscale) y [Security](/es/gateway/security). Este
  flujo sin token asume que el host del gateway es de confianza.
- `gateway.tailscale.mode: "funnel"` requiere `gateway.auth.mode: "password"` (contraseña compartida).

## Compilar la UI

El Gateway sirve archivos estáticos desde `dist/control-ui`. Compílalos con:

```bash
pnpm ui:build # instala automáticamente las dependencias de la UI en la primera ejecución
```
