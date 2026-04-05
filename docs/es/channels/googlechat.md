---
read_when:
    - Trabajas en funcionalidades del canal de Google Chat
summary: Estado de compatibilidad, capacidades y configuración de la app de Google Chat
title: Google Chat
x-i18n:
    generated_at: "2026-04-05T12:35:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 570894ed798dd0b9ba42806b050927216379a1228fcd2f96de565bc8a4ac7c2c
    source_path: channels/googlechat.md
    workflow: 15
---

# Google Chat (Chat API)

Estado: listo para mensajes directos y espacios mediante webhooks de Google Chat API (solo HTTP).

## Configuración rápida (principiante)

1. Crea un proyecto de Google Cloud y habilita **Google Chat API**.
   - Ve a: [Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - Habilita la API si aún no está habilitada.
2. Crea una **Service Account**:
   - Pulsa **Create Credentials** > **Service Account**.
   - Asígnale el nombre que quieras (por ejemplo, `openclaw-chat`).
   - Deja los permisos en blanco (pulsa **Continue**).
   - Deja en blanco los principales con acceso (pulsa **Done**).
3. Crea y descarga la **JSON Key**:
   - En la lista de cuentas de servicio, haz clic en la que acabas de crear.
   - Ve a la pestaña **Keys**.
   - Haz clic en **Add Key** > **Create new key**.
   - Selecciona **JSON** y pulsa **Create**.
4. Guarda el archivo JSON descargado en tu host del gateway (por ejemplo, `~/.openclaw/googlechat-service-account.json`).
5. Crea una app de Google Chat en [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat):
   - Completa la **Application info**:
     - **App name**: (por ejemplo, `OpenClaw`)
     - **Avatar URL**: (por ejemplo, `https://openclaw.ai/logo.png`)
     - **Description**: (por ejemplo, `Personal AI Assistant`)
   - Habilita **Interactive features**.
   - En **Functionality**, marca **Join spaces and group conversations**.
   - En **Connection settings**, selecciona **HTTP endpoint URL**.
   - En **Triggers**, selecciona **Use a common HTTP endpoint URL for all triggers** y configúralo con la URL pública de tu gateway seguida de `/googlechat`.
     - _Consejo: ejecuta `openclaw status` para encontrar la URL pública de tu gateway._
   - En **Visibility**, marca **Make this Chat app available to specific people and groups in &lt;Your Domain&gt;**.
   - Introduce tu dirección de correo electrónico (por ejemplo, `user@example.com`) en el cuadro de texto.
   - Haz clic en **Save** al final.
6. **Habilita el estado de la app**:
   - Después de guardar, **actualiza la página**.
   - Busca la sección **App status** (normalmente cerca de la parte superior o inferior después de guardar).
   - Cambia el estado a **Live - available to users**.
   - Haz clic en **Save** de nuevo.
7. Configura OpenClaw con la ruta de la cuenta de servicio y la audiencia del webhook:
   - Entorno: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json`
   - O configuración: `channels.googlechat.serviceAccountFile: "/path/to/service-account.json"`.
8. Configura el tipo y el valor de la audiencia del webhook (debe coincidir con la configuración de tu app de Chat).
9. Inicia el gateway. Google Chat hará POST a la ruta de tu webhook.

## Añadir a Google Chat

Una vez que el gateway esté en ejecución y tu correo esté añadido a la lista de visibilidad:

1. Ve a [Google Chat](https://chat.google.com/).
2. Haz clic en el icono **+** (más) junto a **Direct Messages**.
3. En la barra de búsqueda (donde normalmente añades personas), escribe el **App name** que configuraste en Google Cloud Console.
   - **Nota**: el bot _no_ aparecerá en la lista de exploración de "Marketplace" porque es una app privada. Debes buscarlo por nombre.
4. Selecciona tu bot en los resultados.
5. Haz clic en **Add** o **Chat** para iniciar una conversación 1:1.
6. Envía "Hello" para activar el asistente.

## URL pública (solo webhook)

Los webhooks de Google Chat requieren un endpoint HTTPS público. Por seguridad, **expón solo la ruta `/googlechat`** a internet. Mantén el panel de OpenClaw y otros endpoints sensibles en tu red privada.

### Opción A: Tailscale Funnel (recomendada)

Usa Tailscale Serve para el panel privado y Funnel para la ruta pública del webhook. Esto mantiene `/` como privado mientras expone solo `/googlechat`.

1. **Comprueba a qué dirección está vinculado tu gateway:**

   ```bash
   ss -tlnp | grep 18789
   ```

   Toma nota de la dirección IP (por ejemplo, `127.0.0.1`, `0.0.0.0` o tu IP de Tailscale como `100.x.x.x`).

2. **Expón el panel solo a la tailnet (puerto 8443):**

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # If bound to Tailscale IP only (e.g., 100.106.161.80):
   tailscale serve --bg --https 8443 http://100.106.161.80:18789
   ```

3. **Expón públicamente solo la ruta del webhook:**

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # If bound to Tailscale IP only (e.g., 100.106.161.80):
   tailscale funnel --bg --set-path /googlechat http://100.106.161.80:18789/googlechat
   ```

4. **Autoriza el nodo para acceso Funnel:**
   Si se te solicita, visita la URL de autorización que aparece en la salida para habilitar Funnel para este nodo en la política de tu tailnet.

5. **Verifica la configuración:**

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

La URL pública de tu webhook será:
`https://<node-name>.<tailnet>.ts.net/googlechat`

Tu panel privado seguirá siendo solo para la tailnet:
`https://<node-name>.<tailnet>.ts.net:8443/`

Usa la URL pública (sin `:8443`) en la configuración de la app de Google Chat.

> Nota: esta configuración persiste tras reinicios. Para eliminarla más adelante, ejecuta `tailscale funnel reset` y `tailscale serve reset`.

### Opción B: Proxy inverso (Caddy)

Si usas un proxy inverso como Caddy, redirige solo la ruta específica:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

Con esta configuración, cualquier solicitud a `your-domain.com/` se ignorará o devolverá 404, mientras que `your-domain.com/googlechat` se enruta de forma segura a OpenClaw.

### Opción C: Cloudflare Tunnel

Configura las reglas de entrada de tu túnel para enrutar solo la ruta del webhook:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default Rule**: HTTP 404 (Not Found)

## Cómo funciona

1. Google Chat envía POST de webhook al gateway. Cada solicitud incluye una cabecera `Authorization: Bearer <token>`.
   - OpenClaw verifica la autenticación bearer antes de leer o analizar cuerpos completos de webhook cuando la cabecera está presente.
   - Las solicitudes de Google Workspace Add-on que incluyen `authorizationEventObject.systemIdToken` en el cuerpo son compatibles mediante un presupuesto de cuerpo previo a la autenticación más estricto.
2. OpenClaw verifica el token con el `audienceType` y la `audience` configurados:
   - `audienceType: "app-url"` → la audiencia es la URL HTTPS de tu webhook.
   - `audienceType: "project-number"` → la audiencia es el número del proyecto de Cloud.
3. Los mensajes se enrutan por espacio:
   - Los mensajes directos usan la clave de sesión `agent:<agentId>:googlechat:direct:<spaceId>`.
   - Los espacios usan la clave de sesión `agent:<agentId>:googlechat:group:<spaceId>`.
4. El acceso a mensajes directos usa emparejamiento de forma predeterminada. Los remitentes desconocidos reciben un código de emparejamiento; apruébalo con:
   - `openclaw pairing approve googlechat <code>`
5. Los espacios de grupo requieren una mención con @ de forma predeterminada. Usa `botUser` si la detección de menciones necesita el nombre de usuario de la app.

## Destinos

Usa estos identificadores para la entrega y las listas de permitidos:

- Mensajes directos: `users/<userId>` (recomendado).
- El correo sin formato `name@example.com` es mutable y solo se usa para coincidencias directas en listas de permitidos cuando `channels.googlechat.dangerouslyAllowNameMatching: true`.
- Obsoleto: `users/<email>` se trata como un id de usuario, no como una lista de permitidos de correo.
- Espacios: `spaces/<spaceId>`.

## Aspectos destacados de la configuración

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // or serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // optional; helps mention detection
      dm: {
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          allow: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Short answers only.",
        },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

Notas:

- Las credenciales de la cuenta de servicio también pueden pasarse en línea con `serviceAccount` (cadena JSON).
- `serviceAccountRef` también es compatible (env/file SecretRef), incluidas las referencias por cuenta en `channels.googlechat.accounts.<id>.serviceAccountRef`.
- La ruta predeterminada del webhook es `/googlechat` si no se configura `webhookPath`.
- `dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia de principales por correo mutable para listas de permitidos (modo de compatibilidad de emergencia).
- Las reacciones están disponibles mediante la herramienta `reactions` y `channels action` cuando `actions.reactions` está habilitado.
- Las acciones de mensajes exponen `send` para texto y `upload-file` para envíos explícitos de archivos adjuntos. `upload-file` acepta `media` / `filePath` / `path` además de `message`, `filename` y destino de hilo opcionales.
- `typingIndicator` admite `none`, `message` (predeterminado) y `reaction` (la reacción requiere OAuth de usuario).
- Los archivos adjuntos se descargan mediante Chat API y se almacenan en la canalización de medios (el tamaño está limitado por `mediaMaxMb`).

Detalles sobre la referencia de secretos: [Secrets Management](/gateway/secrets).

## Solución de problemas

### 405 Method Not Allowed

Si Google Cloud Logs Explorer muestra errores como:

```
status code: 405, reason phrase: HTTP error response: HTTP/1.1 405 Method Not Allowed
```

Esto significa que el controlador del webhook no está registrado. Causas comunes:

1. **Canal no configurado**: falta la sección `channels.googlechat` en tu configuración. Verifícalo con:

   ```bash
   openclaw config get channels.googlechat
   ```

   Si devuelve "Config path not found", añade la configuración (consulta [Aspectos destacados de la configuración](#aspectos-destacados-de-la-configuración)).

2. **Plugin no habilitado**: comprueba el estado del plugin:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   Si muestra "disabled", añade `plugins.entries.googlechat.enabled: true` a tu configuración.

3. **Gateway no reiniciado**: después de añadir la configuración, reinicia el gateway:

   ```bash
   openclaw gateway restart
   ```

Verifica que el canal esté en ejecución:

```bash
openclaw channels status
# Should show: Google Chat default: enabled, configured, ...
```

### Otros problemas

- Comprueba `openclaw channels status --probe` para detectar errores de autenticación o falta de configuración de audiencia.
- Si no llegan mensajes, confirma la URL del webhook de la app de Chat y las suscripciones a eventos.
- Si la restricción por mención bloquea las respuestas, configura `botUser` con el nombre de recurso de usuario de la app y verifica `requireMention`.
- Usa `openclaw logs --follow` mientras envías un mensaje de prueba para ver si las solicitudes llegan al gateway.

Documentación relacionada:

- [Configuración del gateway](/gateway/configuration)
- [Seguridad](/gateway/security)
- [Reacciones](/tools/reactions)

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y restricción por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
