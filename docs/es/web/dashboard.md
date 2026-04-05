---
read_when:
    - Cambiar los modos de autenticación o exposición del panel
summary: Acceso y autenticación del panel del Gateway (Control UI)
title: Panel
x-i18n:
    generated_at: "2026-04-05T12:57:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 316e082ae4759f710b457487351e30c53b34c7c2b4bf84ad7b091a50538af5cc
    source_path: web/dashboard.md
    workflow: 15
---

# Panel (Control UI)

El panel del Gateway es la Control UI en el navegador servida en `/` por defecto
(se sobrescribe con `gateway.controlUi.basePath`).

Apertura rápida (Gateway local):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (o [http://localhost:18789/](http://localhost:18789/))

Referencias clave:

- [Control UI](/web/control-ui) para el uso y las capacidades de la IU.
- [Tailscale](/es/gateway/tailscale) para la automatización de Serve/Funnel.
- [Superficies web](/web) para los modos de enlace y las notas de seguridad.

La autenticación se aplica en el handshake de WebSocket mediante la ruta de
autenticación configurada del gateway:

- `connect.params.auth.token`
- `connect.params.auth.password`
- encabezados de identidad de Tailscale Serve cuando `gateway.auth.allowTailscale: true`
- encabezados de identidad de proxy de confianza cuando `gateway.auth.mode: "trusted-proxy"`

Consulta `gateway.auth` en [Configuración del Gateway](/es/gateway/configuration).

Nota de seguridad: la Control UI es una **superficie de administración** (chat, configuración, aprobaciones de exec).
No la expongas públicamente. La IU mantiene los tokens de URL del panel en `sessionStorage`
para la sesión actual de la pestaña del navegador y la URL del gateway seleccionada, y los elimina de la URL después de la carga.
Prefiere localhost, Tailscale Serve o un túnel SSH.

## Ruta rápida (recomendada)

- Después del onboarding, la CLI abre automáticamente el panel e imprime un enlace limpio (sin token).
- Vuelve a abrirlo en cualquier momento: `openclaw dashboard` (copia el enlace, abre el navegador si es posible y muestra una pista de SSH si no hay interfaz).
- Si la IU solicita autenticación por secreto compartido, pega el token o la
  contraseña configurados en la configuración de Control UI.

## Aspectos básicos de autenticación (local frente a remoto)

- **Localhost**: abre `http://127.0.0.1:18789/`.
- **Origen del token de secreto compartido**: `gateway.auth.token` (o
  `OPENCLAW_GATEWAY_TOKEN`); `openclaw dashboard` puede pasarlo mediante un fragmento de URL
  para un bootstrap de una sola vez, y la Control UI lo mantiene en `sessionStorage` para la
  sesión actual de la pestaña del navegador y la URL del gateway seleccionada en lugar de `localStorage`.
- Si `gateway.auth.token` está gestionado por SecretRef, `openclaw dashboard`
  imprime/copia/abre por diseño una URL sin token. Esto evita exponer
  tokens gestionados externamente en logs del shell, historial del portapapeles o argumentos de lanzamiento del navegador.
- Si `gateway.auth.token` está configurado como un SecretRef y no está resuelto en tu
  shell actual, `openclaw dashboard` sigue imprimiendo una URL sin token junto con
  instrucciones útiles para configurar la autenticación.
- **Contraseña de secreto compartido**: usa la `gateway.auth.password` configurada (o
  `OPENCLAW_GATEWAY_PASSWORD`). El panel no conserva contraseñas entre recargas.
- **Modos con identidad**: Tailscale Serve puede satisfacer la autenticación de Control UI/WebSocket
  mediante encabezados de identidad cuando `gateway.auth.allowTailscale: true`, y un
  proxy inverso con reconocimiento de identidad no loopback puede satisfacer
  `gateway.auth.mode: "trusted-proxy"`. En esos modos, el panel no
  necesita que pegues un secreto compartido para el WebSocket.
- **No localhost**: usa Tailscale Serve, un enlace no loopback con secreto compartido, un
  proxy inverso no loopback con reconocimiento de identidad con
  `gateway.auth.mode: "trusted-proxy"` o un túnel SSH. Las API HTTP siguen usando
  autenticación por secreto compartido a menos que ejecutes intencionadamente
  `gateway.auth.mode: "none"` para ingreso privado o autenticación HTTP de proxy de confianza. Consulta
  [Superficies web](/web).

<a id="if-you-see-unauthorized-1008"></a>

## Si ves "unauthorized" / 1008

- Asegúrate de que el gateway sea accesible (local: `openclaw status`; remoto: túnel SSH `ssh -N -L 18789:127.0.0.1:18789 user@host` y luego abre `http://127.0.0.1:18789/`).
- Para `AUTH_TOKEN_MISMATCH`, los clientes pueden hacer un único reintento de confianza con un token de dispositivo en caché cuando el gateway devuelve pistas de reintento. Ese reintento con token en caché reutiliza los ámbitos aprobados almacenados en caché del token; los llamadores con `deviceToken` explícito / `scopes` explícitos conservan el conjunto de ámbitos solicitado. Si la autenticación sigue fallando después de ese reintento, resuelve manualmente la desviación del token.
- Fuera de esa ruta de reintento, la precedencia de autenticación de conexión es: primero token/contraseña compartidos explícitos, luego `deviceToken` explícito, luego token de dispositivo almacenado y, por último, token de bootstrap.
- En la ruta asíncrona de Control UI de Tailscale Serve, los intentos fallidos para el mismo
  `{scope, ip}` se serializan antes de que el limitador de autenticación fallida los registre, por lo que
  el segundo reintento incorrecto concurrente ya puede mostrar `retry later`.
- Para los pasos de reparación de la desviación del token, sigue la [Lista de comprobación para la recuperación de desviación del token](/cli/devices#token-drift-recovery-checklist).
- Recupera o proporciona el secreto compartido desde el host del gateway:
  - Token: `openclaw config get gateway.auth.token`
  - Contraseña: resuelve la `gateway.auth.password` configurada o
    `OPENCLAW_GATEWAY_PASSWORD`
  - Token gestionado por SecretRef: resuelve el proveedor de secretos externo o exporta
    `OPENCLAW_GATEWAY_TOKEN` en este shell y luego vuelve a ejecutar `openclaw dashboard`
  - No hay secreto compartido configurado: `openclaw doctor --generate-gateway-token`
- En la configuración del panel, pega el token o la contraseña en el campo de autenticación
  y luego conecta.
