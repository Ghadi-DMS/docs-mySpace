---
read_when:
    - Llamar herramientas sin ejecutar un turno completo del agente
    - Crear automatizaciones que necesiten aplicar la política de herramientas
summary: Invoca una sola herramienta directamente mediante el endpoint HTTP del Gateway
title: API de invocación de herramientas
x-i18n:
    generated_at: "2026-04-05T12:43:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: e924f257ba50b25dea0ec4c3f9eed4c8cac8a53ddef18215f87ac7de330a37fd
    source_path: gateway/tools-invoke-http-api.md
    workflow: 15
---

# Invocación de herramientas (HTTP)

El Gateway de OpenClaw expone un endpoint HTTP sencillo para invocar directamente una sola herramienta. Siempre está habilitado y usa la autenticación del Gateway más la política de herramientas. Igual que la superficie compatible con OpenAI `/v1/*`, la autenticación bearer con secreto compartido se trata como acceso de operador de confianza para todo el gateway.

- `POST /tools/invoke`
- Mismo puerto que el Gateway (multiplexación WS + HTTP): `http://<gateway-host>:<port>/tools/invoke`

El tamaño máximo predeterminado de la carga útil es de 2 MB.

## Autenticación

Usa la configuración de autenticación del Gateway.

Rutas comunes de autenticación HTTP:

- autenticación con secreto compartido (`gateway.auth.mode="token"` o `"password"`):
  `Authorization: Bearer <token-or-password>`
- autenticación HTTP con identidad de confianza (`gateway.auth.mode="trusted-proxy"`):
  enruta a través del proxy con reconocimiento de identidad configurado y deja que inyecte los
  encabezados de identidad requeridos
- autenticación abierta para entrada privada (`gateway.auth.mode="none"`):
  no se requiere encabezado de autenticación

Notas:

- Cuando `gateway.auth.mode="token"`, usa `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
- Cuando `gateway.auth.mode="password"`, usa `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
- Cuando `gateway.auth.mode="trusted-proxy"`, la solicitud HTTP debe venir de una
  fuente de proxy confiable no loopback configurada; los proxies loopback en el mismo host
  no satisfacen este modo.
- Si `gateway.auth.rateLimit` está configurado y se producen demasiados errores de autenticación, el endpoint devuelve `429` con `Retry-After`.

## Límite de seguridad (importante)

Trata este endpoint como una superficie de **acceso completo de operador** para la instancia del gateway.

- La autenticación HTTP bearer aquí no es un modelo de alcances estrechos por usuario.
- Un token/contraseña válidos del Gateway para este endpoint deben tratarse como credenciales de propietario/operador.
- Para los modos de autenticación con secreto compartido (`token` y `password`), el endpoint restaura los valores predeterminados normales de operador completo incluso si quien llama envía un encabezado `x-openclaw-scopes` más restrictivo.
- La autenticación con secreto compartido también trata las invocaciones directas de herramientas en este endpoint como turnos de remitente propietario.
- Los modos HTTP con identidad de confianza (por ejemplo, autenticación de proxy de confianza o `gateway.auth.mode="none"` en una entrada privada) respetan `x-openclaw-scopes` cuando está presente y, en caso contrario, recurren al conjunto normal de alcances predeterminados de operador.
- Mantén este endpoint solo en loopback/tailnet/entrada privada; no lo expongas directamente a Internet pública.

Matriz de autenticación:

- `gateway.auth.mode="token"` o `"password"` + `Authorization: Bearer ...`
  - demuestra posesión del secreto compartido del operador del gateway
  - ignora `x-openclaw-scopes` más restrictivos
  - restaura el conjunto completo predeterminado de alcances de operador:
    `operator.admin`, `operator.approvals`, `operator.pairing`,
    `operator.read`, `operator.talk.secrets`, `operator.write`
  - trata las invocaciones directas de herramientas en este endpoint como turnos de remitente propietario
- modos HTTP con identidad de confianza (por ejemplo, autenticación de proxy de confianza o `gateway.auth.mode="none"` en entrada privada)
  - autentican alguna identidad externa de confianza o límite de despliegue
  - respetan `x-openclaw-scopes` cuando el encabezado está presente
  - recurren al conjunto normal predeterminado de alcances de operador cuando el encabezado está ausente
  - solo pierden la semántica de propietario cuando quien llama restringe explícitamente los alcances y omite `operator.admin`

## Cuerpo de la solicitud

```json
{
  "tool": "sessions_list",
  "action": "json",
  "args": {},
  "sessionKey": "main",
  "dryRun": false
}
```

Campos:

- `tool` (string, obligatorio): nombre de la herramienta a invocar.
- `action` (string, opcional): se asigna a args si el esquema de la herramienta admite `action` y la carga útil args lo omitió.
- `args` (object, opcional): argumentos específicos de la herramienta.
- `sessionKey` (string, opcional): clave de sesión de destino. Si se omite o es `"main"`, el Gateway usa la clave de sesión principal configurada (respeta `session.mainKey` y el agente predeterminado, o `global` en alcance global).
- `dryRun` (boolean, opcional): reservado para uso futuro; actualmente se ignora.

## Comportamiento de política + enrutamiento

La disponibilidad de herramientas se filtra mediante la misma cadena de políticas usada por los agentes del Gateway:

- `tools.profile` / `tools.byProvider.profile`
- `tools.allow` / `tools.byProvider.allow`
- `agents.<id>.tools.allow` / `agents.<id>.tools.byProvider.allow`
- políticas de grupo (si la clave de sesión corresponde a un grupo o canal)
- política de subagente (cuando se invoca con una clave de sesión de subagente)

Si una herramienta no está permitida por la política, el endpoint devuelve **404**.

Notas importantes sobre límites:

- Las aprobaciones de exec son protecciones de operador, no un límite de autorización independiente para este endpoint HTTP. Si una herramienta es accesible aquí mediante la autenticación del Gateway + política de herramientas, `/tools/invoke` no añade un prompt adicional de aprobación por llamada.
- No compartas credenciales bearer del Gateway con llamantes que no sean de confianza. Si necesitas separación entre límites de confianza, ejecuta gateways separados (e idealmente usuarios/hosts de SO separados).

El HTTP del Gateway también aplica de forma predeterminada una lista de denegación estricta (incluso si la política de sesión permite la herramienta):

- `exec` — ejecución directa de comandos (superficie RCE)
- `spawn` — creación arbitraria de procesos hijo (superficie RCE)
- `shell` — ejecución de comandos de shell (superficie RCE)
- `fs_write` — mutación arbitraria de archivos en el host
- `fs_delete` — eliminación arbitraria de archivos en el host
- `fs_move` — mover/renombrar archivos arbitrariamente en el host
- `apply_patch` — la aplicación de parches puede reescribir archivos arbitrarios
- `sessions_spawn` — orquestación de sesiones; generar agentes remotamente es RCE
- `sessions_send` — inyección de mensajes entre sesiones
- `cron` — plano de control de automatización persistente
- `gateway` — plano de control del gateway; impide la reconfiguración mediante HTTP
- `nodes` — el relevo de comandos de nodos puede alcanzar `system.run` en hosts emparejados
- `whatsapp_login` — configuración interactiva que requiere escaneo QR en terminal; se queda colgado en HTTP

Puedes personalizar esta lista de denegación mediante `gateway.tools`:

```json5
{
  gateway: {
    tools: {
      // Additional tools to block over HTTP /tools/invoke
      deny: ["browser"],
      // Remove tools from the default deny list
      allow: ["gateway"],
    },
  },
}
```

Para ayudar a que las políticas de grupo resuelvan el contexto, opcionalmente puedes establecer:

- `x-openclaw-message-channel: <channel>` (ejemplo: `slack`, `telegram`)
- `x-openclaw-account-id: <accountId>` (cuando existen varias cuentas)

## Respuestas

- `200` → `{ ok: true, result }`
- `400` → `{ ok: false, error: { type, message } }` (solicitud no válida o error de entrada de herramienta)
- `401` → no autorizado
- `429` → autenticación limitada por tasa (`Retry-After` establecido)
- `404` → herramienta no disponible (no encontrada o no incluida en la lista de permitidos)
- `405` → método no permitido
- `500` → `{ ok: false, error: { type, message } }` (error inesperado de ejecución de herramienta; mensaje saneado)

## Ejemplo

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer secret' \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "sessions_list",
    "action": "json",
    "args": {}
  }'
```
