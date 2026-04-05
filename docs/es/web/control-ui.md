---
read_when:
    - Quieres operar el Gateway desde un navegador
    - Quieres acceso por Tailnet sin túneles SSH
summary: Interfaz de control basada en navegador para el Gateway (chat, nodos, configuración)
title: Interfaz de control
x-i18n:
    generated_at: "2026-04-05T12:57:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1568680a07907343352dbb3a2e6a1b896826404a7d8baba62512f03eac28e3d7
    source_path: web/control-ui.md
    workflow: 15
---

# Interfaz de control (navegador)

La interfaz de control es una pequeña aplicación de una sola página **Vite + Lit** servida por el Gateway:

- predeterminado: `http://<host>:18789/`
- prefijo opcional: configura `gateway.controlUi.basePath` (por ejemplo `/openclaw`)

Se comunica **directamente con el WebSocket del Gateway** en el mismo puerto.

## Apertura rápida (local)

Si el Gateway se está ejecutando en el mismo equipo, abre:

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (o [http://localhost:18789/](http://localhost:18789/))

Si la página no carga, inicia primero el Gateway: `openclaw gateway`.

La autenticación se proporciona durante el handshake del WebSocket mediante:

- `connect.params.auth.token`
- `connect.params.auth.password`
- encabezados de identidad de Tailscale Serve cuando `gateway.auth.allowTailscale: true`
- encabezados de identidad de proxy de confianza cuando `gateway.auth.mode: "trusted-proxy"`

El panel de configuración del dashboard conserva un token para la sesión actual
de la pestaña del navegador y la URL del gateway seleccionada; las contraseñas no se conservan. El onboarding suele
generar un token del gateway para autenticación con secreto compartido en la primera conexión, pero la
autenticación por contraseña también funciona cuando `gateway.auth.mode` es `"password"`.

## Emparejamiento de dispositivos (primera conexión)

Cuando te conectas a la interfaz de control desde un navegador o dispositivo nuevo, el Gateway
requiere una **aprobación de emparejamiento de una sola vez**, incluso si estás en la misma Tailnet
con `gateway.auth.allowTailscale: true`. Esta es una medida de seguridad para evitar
accesos no autorizados.

**Lo que verás:** "disconnected (1008): pairing required"

**Para aprobar el dispositivo:**

```bash
# Listar solicitudes pendientes
openclaw devices list

# Aprobar por ID de solicitud
openclaw devices approve <requestId>
```

Si el navegador vuelve a intentar el emparejamiento con detalles de autenticación cambiados (rol/ámbitos/clave
pública), la solicitud pendiente anterior queda reemplazada y se crea un nuevo `requestId`.
Vuelve a ejecutar `openclaw devices list` antes de aprobar.

Una vez aprobado, el dispositivo se recuerda y no requerirá una nueva aprobación a menos
que lo revoques con `openclaw devices revoke --device <id> --role <role>`. Consulta
[CLI de Devices](/cli/devices) para rotación de tokens y revocación.

**Notas:**

- Las conexiones directas del navegador local por loopback (`127.0.0.1` / `localhost`) se
  aprueban automáticamente.
- Las conexiones del navegador por Tailnet y LAN siguen requiriendo aprobación explícita, incluso cuando
  se originan desde la misma máquina.
- Cada perfil del navegador genera un ID de dispositivo único, por lo que cambiar de navegador o
  borrar los datos del navegador requerirá volver a emparejar.

## Compatibilidad de idiomas

La interfaz de control puede localizarse en la primera carga según la configuración regional de tu navegador, y puedes sobrescribirla más tarde desde el selector de idioma de la tarjeta Access.

- Configuraciones regionales admitidas: `en`, `zh-CN`, `zh-TW`, `pt-BR`, `de`, `es`
- Las traducciones que no están en inglés se cargan de forma diferida en el navegador.
- La configuración regional seleccionada se guarda en el almacenamiento del navegador y se reutiliza en futuras visitas.
- Las claves de traducción faltantes vuelven al inglés.

## Qué puede hacer (hoy)

- Chatear con el modelo mediante el WS del Gateway (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`)
- Transmitir llamadas a herramientas + tarjetas de salida de herramientas en vivo en el chat (eventos del agente)
- Canales: estado, inicio de sesión por QR y configuración por canal de canales integrados y de plugins incluidos/externos (`channels.status`, `web.login.*`, `config.patch`)
- Instancias: lista de presencia + actualización (`system-presence`)
- Sesiones: lista + sobrescrituras por sesión de modelo/razonamiento/rápido/detallado/reasoning (`sessions.list`, `sessions.patch`)
- Trabajos cron: listar/agregar/editar/ejecutar/habilitar/deshabilitar + historial de ejecuciones (`cron.*`)
- Skills: estado, habilitar/deshabilitar, instalar, actualizaciones de claves de API (`skills.*`)
- Nodos: lista + capacidades (`node.list`)
- Aprobaciones de Exec: editar listas de permitidos del gateway o nodo + política ask para `exec host=gateway/node` (`exec.approvals.*`)
- Configuración: ver/editar `~/.openclaw/openclaw.json` (`config.get`, `config.set`)
- Configuración: aplicar + reiniciar con validación (`config.apply`) y activar la última sesión activa
- Las escrituras de configuración incluyen una protección de hash base para evitar sobrescribir ediciones concurrentes
- Las escrituras de configuración (`config.set`/`config.apply`/`config.patch`) también validan previamente la resolución activa de SecretRef para referencias en la carga útil de configuración enviada; las referencias activas enviadas no resueltas se rechazan antes de la escritura
- Esquema de configuración + renderizado de formularios (`config.schema` / `config.schema.lookup`,
  incluidos `title` / `description` de campos, sugerencias de UI coincidentes, resúmenes
  inmediatos de elementos secundarios, metadatos de documentación en nodos anidados de objeto/comodín/arreglo/composición,
  además de esquemas de plugins y canales cuando están disponibles); el editor JSON sin formato
  está disponible solo cuando la instantánea tiene un round-trip seguro del texto sin formato
- Si una instantánea no puede hacer round-trip seguro del texto sin formato, la interfaz de control fuerza el modo Form y desactiva el modo Raw para esa instantánea
- Los valores de objeto SecretRef estructurados se renderizan como solo lectura en las entradas de texto del formulario para evitar corrupción accidental de objeto a cadena
- Depuración: instantáneas de estado/health/models + registro de eventos + llamadas RPC manuales (`status`, `health`, `models.list`)
- Registros: tail en vivo de los archivos de registro del gateway con filtro/exportación (`logs.tail`)
- Actualización: ejecutar una actualización de paquete/git + reinicio (`update.run`) con un informe de reinicio

Notas del panel de trabajos cron:

- Para trabajos aislados, la entrega usa de forma predeterminada resumen anunciado. Puedes cambiar a none si quieres ejecuciones solo internas.
- Los campos de canal/destino aparecen cuando se selecciona announce.
- El modo webhook usa `delivery.mode = "webhook"` con `delivery.to` establecido en una URL de webhook HTTP(S) válida.
- Para trabajos de sesión principal, están disponibles los modos de entrega webhook y none.
- Los controles de edición avanzada incluyen delete-after-run, borrar la sobrescritura del agente, opciones de exact/stagger de cron,
  sobrescrituras de modelo/razonamiento del agente y alternancias de entrega best-effort.
- La validación del formulario es en línea con errores a nivel de campo; los valores no válidos desactivan el botón de guardar hasta corregirlos.
- Configura `cron.webhookToken` para enviar un token bearer dedicado; si se omite, el webhook se envía sin encabezado de autenticación.
- Alternativa obsoleta: los trabajos heredados almacenados con `notify: true` todavía pueden usar `cron.webhook` hasta que se migren.

## Comportamiento del chat

- `chat.send` es **no bloqueante**: reconoce inmediatamente con `{ runId, status: "started" }` y la respuesta se transmite mediante eventos `chat`.
- Volver a enviar con la misma `idempotencyKey` devuelve `{ status: "in_flight" }` mientras se está ejecutando, y `{ status: "ok" }` tras completarse.
- Las respuestas de `chat.history` tienen tamaño limitado por seguridad de la UI. Cuando las entradas de la transcripción son demasiado grandes, Gateway puede truncar campos de texto largos, omitir bloques pesados de metadatos y reemplazar mensajes sobredimensionados por un marcador (`[chat.history omitted: message too large]`).
- `chat.history` también elimina de la vista el texto visible del asistente de etiquetas inline de directivas solo para visualización (por ejemplo `[[reply_to_*]]` y `[[audio_as_voice]]`), cargas XML de llamadas a herramientas en texto plano (incluidas `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y bloques truncados de llamadas a herramientas), y tokens de control del modelo filtrados en ASCII/ancho completo, y omite entradas del asistente cuyo texto visible completo sea solo el token silencioso exacto `NO_REPLY` / `no_reply`.
- `chat.inject` agrega una nota del asistente a la transcripción de la sesión y emite un evento `chat` para actualizaciones solo de la UI (sin ejecución del agente, sin entrega al canal).
- Los selectores de modelo y razonamiento del encabezado del chat aplican parches inmediatamente a la sesión activa mediante `sessions.patch`; son sobrescrituras persistentes de la sesión, no opciones de envío para un solo turno.
- Detener:
  - Haz clic en **Stop** (llama a `chat.abort`)
  - Escribe `/stop` (o frases de aborto independientes como `stop`, `stop action`, `stop run`, `stop openclaw`, `please stop`) para abortar fuera de banda
  - `chat.abort` admite `{ sessionKey }` (sin `runId`) para abortar todas las ejecuciones activas de esa sesión
- Conservación parcial de abortos:
  - Cuando se aborta una ejecución, el texto parcial del asistente todavía puede mostrarse en la UI
  - Gateway conserva el texto parcial abortado del asistente en el historial de la transcripción cuando existe salida en buffer
  - Las entradas conservadas incluyen metadatos de aborto para que los consumidores de la transcripción puedan distinguir las partes abortadas de la salida completada normalmente

## Acceso por Tailnet (recomendado)

### Tailscale Serve integrado (preferido)

Mantén el Gateway en loopback y deja que Tailscale Serve lo proxifique con HTTPS:

```bash
openclaw gateway --tailscale serve
```

Abre:

- `https://<magicdns>/` (o tu `gateway.controlUi.basePath` configurado)

De forma predeterminada, las solicitudes de Control UI/WebSocket Serve pueden autenticarse mediante encabezados de identidad de Tailscale
(`tailscale-user-login`) cuando `gateway.auth.allowTailscale` es `true`. OpenClaw
verifica la identidad resolviendo la dirección `x-forwarded-for` con
`tailscale whois` y comparándola con el encabezado, y solo acepta esto cuando la
solicitud llega a loopback con los encabezados `x-forwarded-*` de Tailscale. Configura
`gateway.auth.allowTailscale: false` si quieres exigir credenciales explícitas de secreto compartido
incluso para tráfico de Serve. Entonces usa `gateway.auth.mode: "token"` o
`"password"`.
Para esa ruta asíncrona de identidad de Serve, los intentos fallidos de autenticación del mismo IP
de cliente y ámbito de autenticación se serializan antes de las escrituras del límite de tasa. Por lo tanto,
los reintentos malos concurrentes desde el mismo navegador pueden mostrar `retry later` en la segunda solicitud
en lugar de dos discrepancias simples compitiendo en paralelo.
La autenticación de Serve sin token asume que el host del gateway es de confianza. Si puede ejecutarse
código local no confiable en ese host, exige autenticación por token/contraseña.

### Bind a tailnet + token

```bash
openclaw gateway --bind tailnet --token "$(openssl rand -hex 32)"
```

Luego abre:

- `http://<tailscale-ip>:18789/` (o tu `gateway.controlUi.basePath` configurado)

Pega el secreto compartido correspondiente en la configuración de la UI (se envía como
`connect.params.auth.token` o `connect.params.auth.password`).

## HTTP inseguro

Si abres el dashboard mediante HTTP simple (`http://<lan-ip>` o `http://<tailscale-ip>`),
el navegador se ejecuta en un **contexto no seguro** y bloquea WebCrypto. De forma predeterminada,
OpenClaw **bloquea** las conexiones de la interfaz de control sin identidad de dispositivo.

Excepciones documentadas:

- compatibilidad con HTTP inseguro solo localhost con `gateway.controlUi.allowInsecureAuth=true`
- autenticación correcta del operador de la interfaz de control mediante `gateway.auth.mode: "trusted-proxy"`
- solución de emergencia `gateway.controlUi.dangerouslyDisableDeviceAuth=true`

**Corrección recomendada:** usa HTTPS (Tailscale Serve) o abre la UI localmente:

- `https://<magicdns>/` (Serve)
- `http://127.0.0.1:18789/` (en el host del gateway)

**Comportamiento de la alternancia insecure-auth:**

```json5
{
  gateway: {
    controlUi: { allowInsecureAuth: true },
    bind: "tailnet",
    auth: { mode: "token", token: "replace-me" },
  },
}
```

`allowInsecureAuth` es solo una alternancia local de compatibilidad:

- Permite que las sesiones de la interfaz de control de localhost continúen sin identidad de dispositivo en
  contextos HTTP no seguros.
- No omite las comprobaciones de emparejamiento.
- No relaja los requisitos remotos (no localhost) de identidad del dispositivo.

**Solo como solución de emergencia:**

```json5
{
  gateway: {
    controlUi: { dangerouslyDisableDeviceAuth: true },
    bind: "tailnet",
    auth: { mode: "token", token: "replace-me" },
  },
}
```

`dangerouslyDisableDeviceAuth` desactiva las comprobaciones de identidad de dispositivo de la interfaz de control y supone una
grave degradación de seguridad. Reviértelo rápidamente después del uso de emergencia.

Nota sobre proxy de confianza:

- una autenticación correcta mediante trusted-proxy puede admitir sesiones **de operador** de la interfaz de control sin
  identidad de dispositivo
- esto **no** se extiende a las sesiones de la interfaz de control con rol de nodo
- los proxies inversos loopback del mismo host siguen sin satisfacer la autenticación trusted-proxy; consulta
  [Autenticación Trusted Proxy](/es/gateway/trusted-proxy-auth)

Consulta [Tailscale](/es/gateway/tailscale) para obtener orientación de configuración de HTTPS.

## Compilación de la UI

El Gateway sirve archivos estáticos desde `dist/control-ui`. Compílalos con:

```bash
pnpm ui:build # instala automáticamente las dependencias de la UI en la primera ejecución
```

Base absoluta opcional (cuando quieres URLs de recursos fijas):

```bash
OPENCLAW_CONTROL_UI_BASE_PATH=/openclaw/ pnpm ui:build
```

Para desarrollo local (servidor de desarrollo independiente):

```bash
pnpm ui:dev # instala automáticamente las dependencias de la UI en la primera ejecución
```

Luego apunta la UI a la URL WS de tu Gateway (por ejemplo `ws://127.0.0.1:18789`).

## Depuración/pruebas: servidor de desarrollo + Gateway remoto

La interfaz de control son archivos estáticos; el objetivo del WebSocket es configurable y puede ser
distinto del origen HTTP. Esto es útil cuando quieres el servidor de desarrollo de Vite
localmente pero el Gateway se ejecuta en otro lugar.

1. Inicia el servidor de desarrollo de la UI: `pnpm ui:dev`
2. Abre una URL como:

```text
http://localhost:5173/?gatewayUrl=ws://<gateway-host>:18789
```

Autenticación opcional de una sola vez (si es necesaria):

```text
http://localhost:5173/?gatewayUrl=wss://<gateway-host>:18789#token=<gateway-token>
```

Notas:

- `gatewayUrl` se almacena en `localStorage` después de la carga y se elimina de la URL.
- `token` debe pasarse mediante el fragmento de la URL (`#token=...`) siempre que sea posible. Los fragmentos no se envían al servidor, lo que evita filtraciones en registros de solicitudes y Referer. Los parámetros heredados de consulta `?token=` todavía se importan una vez por compatibilidad, pero solo como alternativa, y se eliminan inmediatamente después del bootstrap.
- `password` se conserva solo en memoria.
- Cuando `gatewayUrl` está configurado, la UI no recurre a credenciales de configuración ni de entorno.
  Proporciona `token` (o `password`) explícitamente. La ausencia de credenciales explícitas es un error.
- Usa `wss://` cuando el Gateway esté detrás de TLS (Tailscale Serve, proxy HTTPS, etc.).
- `gatewayUrl` solo se acepta en una ventana de nivel superior (no incrustada) para evitar clickjacking.
- Las implementaciones de la interfaz de control que no son loopback deben configurar `gateway.controlUi.allowedOrigins`
  explícitamente (orígenes completos). Esto incluye configuraciones remotas de desarrollo.
- No uses `gateway.controlUi.allowedOrigins: ["*"]` salvo en pruebas locales
  estrictamente controladas. Significa permitir cualquier origen del navegador, no “hacer coincidir cualquier host que esté
  usando”.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` habilita
  el modo alternativo de origen basado en encabezado Host, pero es un modo de seguridad peligroso.

Ejemplo:

```json5
{
  gateway: {
    controlUi: {
      allowedOrigins: ["http://localhost:5173"],
    },
  },
}
```

Detalles de configuración de acceso remoto: [Acceso remoto](/es/gateway/remote).

## Relacionado

- [Dashboard](/web/dashboard) — dashboard del gateway
- [WebChat](/web/webchat) — interfaz de chat basada en navegador
- [TUI](/web/tui) — interfaz de usuario de terminal
- [Health Checks](/es/gateway/health) — monitorización del estado del gateway
