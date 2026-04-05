---
read_when:
    - Agregar automatización del navegador controlada por el agente
    - Depurar por qué openclaw está interfiriendo con tu propio Chrome
    - Implementar la configuración y el ciclo de vida del navegador en la aplicación de macOS
summary: Servicio integrado de control del navegador + comandos de acción
title: Navegador (gestionado por OpenClaw)
x-i18n:
    generated_at: "2026-04-05T12:56:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: a41162efd397ea918469e16aa67e554bcbb517b3112df1d3e7927539b6a0926a
    source_path: tools/browser.md
    workflow: 15
---

# Navegador (gestionado por openclaw)

OpenClaw puede ejecutar un **perfil dedicado de Chrome/Brave/Edge/Chromium** que el agente controla.
Está aislado de tu navegador personal y se gestiona a través de un pequeño
servicio de control local dentro del Gateway (solo loopback).

Vista para principiantes:

- Piénsalo como un **navegador independiente, solo para el agente**.
- El perfil `openclaw` **no** toca tu perfil de navegador personal.
- El agente puede **abrir pestañas, leer páginas, hacer clic y escribir** en un entorno seguro.
- El perfil integrado `user` se adjunta a tu sesión real de Chrome con sesión iniciada mediante Chrome MCP.

## Qué obtienes

- Un perfil de navegador independiente llamado **openclaw** (con acento naranja de forma predeterminada).
- Control determinista de pestañas (listar/abrir/enfocar/cerrar).
- Acciones del agente (clic/escribir/arrastrar/seleccionar), snapshots, capturas de pantalla y PDF.
- Soporte opcional para varios perfiles (`openclaw`, `work`, `remote`, ...).

Este navegador **no** es tu navegador de uso diario. Es una superficie segura y aislada para
la automatización y verificación del agente.

## Inicio rápido

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Si obtienes “Browser disabled”, actívalo en la configuración (ver más abajo) y reinicia el
Gateway.

Si `openclaw browser` falta por completo, o el agente dice que la herramienta de navegador
no está disponible, ve a [Comando o herramienta de navegador ausente](/tools/browser#missing-browser-command-or-tool).

## Control de plugins

La herramienta predeterminada `browser` ahora es un plugin incluido que se envía habilitado de
forma predeterminada. Eso significa que puedes desactivarlo o sustituirlo sin eliminar el resto del
sistema de plugins de OpenClaw:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

Desactiva el plugin incluido antes de instalar otro plugin que proporcione el
mismo nombre de herramienta `browser`. La experiencia predeterminada del navegador necesita ambos:

- `plugins.entries.browser.enabled` no deshabilitado
- `browser.enabled=true`

Si desactivas solo el plugin, la CLI del navegador incluida (`openclaw browser`),
el método del gateway (`browser.request`), la herramienta del agente y el servicio predeterminado de control del navegador
desaparecen todos juntos. Tu configuración `browser.*` permanece intacta para que un
plugin de reemplazo la reutilice.

El plugin del navegador incluido también es ahora el propietario de la implementación del runtime del navegador.
El núcleo solo conserva helpers compartidos del Plugin SDK más reexportaciones de compatibilidad para
rutas internas de importación más antiguas. En la práctica, eliminar o sustituir el paquete del plugin del navegador
elimina el conjunto de funciones del navegador en lugar de dejar detrás un segundo
runtime propiedad del núcleo.

Los cambios en la configuración del navegador siguen requiriendo un reinicio del Gateway para que el plugin incluido
pueda volver a registrar su servicio de navegador con la nueva configuración.

## Comando o herramienta de navegador ausente

Si `openclaw browser` de repente se convierte en un comando desconocido tras una actualización, o
el agente informa de que falta la herramienta de navegador, la causa más común es una
lista restrictiva `plugins.allow` que no incluye `browser`.

Ejemplo de configuración incorrecta:

```json5
{
  plugins: {
    allow: ["telegram"],
  },
}
```

Corrígelo agregando `browser` a la allowlist de plugins:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Notas importantes:

- `browser.enabled=true` no es suficiente por sí solo cuando `plugins.allow` está establecido.
- `plugins.entries.browser.enabled=true` tampoco es suficiente por sí solo cuando `plugins.allow` está establecido.
- `tools.alsoAllow: ["browser"]` **no** carga el plugin del navegador incluido. Solo ajusta la política de herramientas después de que el plugin ya esté cargado.
- Si no necesitas una allowlist de plugins restrictiva, eliminar `plugins.allow` también restaura el comportamiento predeterminado del navegador incluido.

Síntomas típicos:

- `openclaw browser` es un comando desconocido.
- Falta `browser.request`.
- El agente informa de que la herramienta de navegador no está disponible o falta.

## Perfiles: `openclaw` frente a `user`

- `openclaw`: navegador gestionado y aislado (no requiere extensión).
- `user`: perfil integrado de conexión Chrome MCP para tu **sesión real de Chrome**
  con sesión iniciada.

Para las llamadas de la herramienta de navegador del agente:

- Predeterminado: usa el navegador aislado `openclaw`.
- Prefiere `profile="user"` cuando importen las sesiones existentes con inicio de sesión y el usuario
  esté en la computadora para hacer clic o aprobar cualquier aviso de conexión.
- `profile` es la anulación explícita cuando quieres un modo de navegador específico.

Establece `browser.defaultProfile: "openclaw"` si quieres el modo gestionado de forma predeterminada.

## Configuración

La configuración del navegador vive en `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // default: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy single-profile override
    remoteCdpTimeoutMs: 1500, // remote CDP HTTP timeout (ms)
    remoteCdpHandshakeTimeoutMs: 3000, // remote CDP WebSocket handshake timeout (ms)
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

Notas:

- El servicio de control del navegador se vincula a loopback en un puerto derivado de `gateway.port`
  (predeterminado: `18791`, que es gateway + 2).
- Si anulas el puerto del Gateway (`gateway.port` o `OPENCLAW_GATEWAY_PORT`),
  los puertos del navegador derivados se desplazan para permanecer en la misma “familia”.
- `cdpUrl` usa de forma predeterminada el puerto CDP local gestionado cuando no está establecido.
- `remoteCdpTimeoutMs` se aplica a las comprobaciones de accesibilidad de CDP remotas (no loopback).
- `remoteCdpHandshakeTimeoutMs` se aplica a las comprobaciones de accesibilidad del handshake de WebSocket CDP remoto.
- La navegación/apertura de pestañas del navegador está protegida contra SSRF antes de la navegación y se vuelve a comprobar en la medida de lo posible en la URL final `http(s)` tras la navegación.
- En el modo SSRF estricto, el descubrimiento/sondeo de endpoints CDP remotos (`cdpUrl`, incluidas las búsquedas `/json/version`) también se comprueba.
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` usa `true` de forma predeterminada (modelo de red de confianza). Establécelo en `false` para una navegación estricta solo pública.
- `browser.ssrfPolicy.allowPrivateNetwork` sigue siendo compatible como alias heredado por compatibilidad.
- `attachOnly: true` significa “nunca iniciar un navegador local; solo adjuntarse si ya está en ejecución”.
- `color` + `color` por perfil tiñen la interfaz del navegador para que puedas ver qué perfil está activo.
- El perfil predeterminado es `openclaw` (navegador independiente gestionado por OpenClaw). Usa `defaultProfile: "user"` para optar por el navegador de usuario con sesión iniciada.
- Orden de detección automática: navegador predeterminado del sistema si está basado en Chromium; si no, Chrome → Brave → Edge → Chromium → Chrome Canary.
- Los perfiles `openclaw` locales asignan automáticamente `cdpPort`/`cdpUrl`; establécelos solo para CDP remoto.
- `driver: "existing-session"` usa Chrome DevTools MCP en lugar de CDP sin procesar. No
  establezcas `cdpUrl` para ese driver.
- Establece `browser.profiles.<name>.userDataDir` cuando un perfil existing-session
  deba adjuntarse a un perfil de usuario Chromium no predeterminado como Brave o Edge.

## Usar Brave (u otro navegador basado en Chromium)

Si tu navegador **predeterminado del sistema** está basado en Chromium (Chrome/Brave/Edge/etc),
OpenClaw lo usa automáticamente. Establece `browser.executablePath` para anular la
detección automática:

Ejemplo de CLI:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## Control local frente a remoto

- **Control local (predeterminado):** el Gateway inicia el servicio de control en loopback y puede lanzar un navegador local.
- **Control remoto (host node):** ejecuta un host node en la máquina que tiene el navegador; el Gateway redirige las acciones del navegador hacia él.
- **CDP remoto:** establece `browser.profiles.<name>.cdpUrl` (o `browser.cdpUrl`) para
  adjuntarte a un navegador remoto basado en Chromium. En este caso, OpenClaw no iniciará un navegador local.

El comportamiento de detención difiere según el modo de perfil:

- perfiles locales gestionados: `openclaw browser stop` detiene el proceso del navegador que
  OpenClaw lanzó
- perfiles solo adjuntos y CDP remotos: `openclaw browser stop` cierra la
  sesión de control activa y libera las anulaciones de emulación de Playwright/CDP (viewport,
  esquema de color, configuración regional, zona horaria, modo sin conexión y estado similar), incluso
  aunque OpenClaw no haya iniciado ningún proceso de navegador

Las URL de CDP remoto pueden incluir autenticación:

- Tokens de consulta (por ejemplo, `https://provider.example?token=<token>`)
- Autenticación HTTP Basic (por ejemplo, `https://user:pass@provider.example`)

OpenClaw conserva la autenticación al llamar a los endpoints `/json/*` y al conectarse
al WebSocket CDP. Prefiere variables de entorno o gestores de secretos para
los tokens en lugar de confirmarlos en archivos de configuración.

## Proxy de navegador node (predeterminado sin configuración)

Si ejecutas un **host node** en la máquina que tiene tu navegador, OpenClaw puede
redirigir automáticamente las llamadas de la herramienta de navegador a ese nodo sin ninguna configuración adicional del navegador.
Esta es la ruta predeterminada para gateways remotos.

Notas:

- El host node expone su servidor local de control del navegador mediante un **comando proxy**.
- Los perfiles provienen de la propia configuración `browser.profiles` del nodo (igual que en local).
- `nodeHost.browserProxy.allowProfiles` es opcional. Déjalo vacío para el comportamiento heredado/predeterminado: todos los perfiles configurados siguen siendo accesibles a través del proxy, incluidas las rutas de creación/eliminación de perfiles.
- Si estableces `nodeHost.browserProxy.allowProfiles`, OpenClaw lo trata como un límite de mínimo privilegio: solo se pueden dirigir perfiles de la allowlist, y las rutas persistentes de creación/eliminación de perfiles quedan bloqueadas en la superficie del proxy.
- Desactívalo si no lo quieres:
  - En el nodo: `nodeHost.browserProxy.enabled=false`
  - En el gateway: `gateway.nodes.browser.mode="off"`

## Browserless (CDP remoto alojado)

[Browserless](https://browserless.io) es un servicio alojado de Chromium que expone
URL de conexión CDP a través de HTTPS y WebSocket. OpenClaw puede usar cualquiera de las dos formas, pero
para un perfil de navegador remoto la opción más sencilla es la URL directa de WebSocket
de la documentación de conexión de Browserless.

Ejemplo:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

Notas:

- Sustituye `<BROWSERLESS_API_KEY>` por tu token real de Browserless.
- Elige el endpoint regional que coincida con tu cuenta de Browserless (consulta su documentación).
- Si Browserless te da una URL base HTTPS, puedes convertirla a
  `wss://` para una conexión CDP directa o mantener la URL HTTPS y dejar que OpenClaw
  descubra `/json/version`.

## Proveedores CDP directos por WebSocket

Algunos servicios de navegador alojados exponen un endpoint **WebSocket directo** en lugar del
descubrimiento CDP estándar basado en HTTP (`/json/version`). OpenClaw admite ambos:

- **Endpoints HTTP(S)** — OpenClaw llama a `/json/version` para descubrir la
  URL del depurador WebSocket y luego se conecta.
- **Endpoints WebSocket** (`ws://` / `wss://`) — OpenClaw se conecta directamente,
  omitiendo `/json/version`. Usa esto para servicios como
  [Browserless](https://browserless.io),
  [Browserbase](https://www.browserbase.com) o cualquier proveedor que te entregue una
  URL WebSocket.

### Browserbase

[Browserbase](https://www.browserbase.com) es una plataforma en la nube para ejecutar
navegadores headless con resolución integrada de CAPTCHA, modo sigiloso y
proxies residenciales.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

Notas:

- [Regístrate](https://www.browserbase.com/sign-up) y copia tu **API Key**
  desde el [panel general](https://www.browserbase.com/overview).
- Sustituye `<BROWSERBASE_API_KEY>` por tu clave de API real de Browserbase.
- Browserbase crea automáticamente una sesión de navegador al conectar por WebSocket, así que no
  hace falta un paso manual de creación de sesión.
- El nivel gratuito permite una sesión concurrente y una hora de navegador al mes.
  Consulta [pricing](https://www.browserbase.com/pricing) para los límites de los planes de pago.
- Consulta la [documentación de Browserbase](https://docs.browserbase.com) para la referencia completa de la API,
  guías del SDK y ejemplos de integración.

## Seguridad

Ideas clave:

- El control del navegador es solo loopback; el acceso fluye a través de la autenticación del Gateway o del emparejamiento de nodos.
- La API HTTP independiente del navegador en loopback usa **solo autenticación con secreto compartido**:
  autenticación bearer con token del gateway, `x-openclaw-password` o autenticación HTTP Basic con la
  contraseña configurada del gateway.
- Las cabeceras de identidad de Tailscale Serve y `gateway.auth.mode: "trusted-proxy"` **no**
  autentican esta API independiente del navegador en loopback.
- Si el control del navegador está habilitado y no hay autenticación con secreto compartido configurada, OpenClaw
  genera automáticamente `gateway.auth.token` al inicio y lo persiste en la configuración.
- OpenClaw **no** genera automáticamente ese token cuando `gateway.auth.mode` ya es
  `password`, `none` o `trusted-proxy`.
- Mantén el Gateway y cualquier host node en una red privada (Tailscale); evita la exposición pública.
- Trata las URL/tokens de CDP remoto como secretos; prefiere variables de entorno o un gestor de secretos.

Consejos para CDP remoto:

- Prefiere endpoints cifrados (HTTPS o WSS) y tokens de corta duración cuando sea posible.
- Evita incrustar tokens de larga duración directamente en archivos de configuración.

## Perfiles (varios navegadores)

OpenClaw admite varios perfiles con nombre (configuraciones de enrutamiento). Los perfiles pueden ser:

- **gestionados por openclaw**: una instancia dedicada de navegador basado en Chromium con su propio directorio de datos de usuario + puerto CDP
- **remotos**: una URL CDP explícita (navegador basado en Chromium ejecutándose en otro lugar)
- **sesión existente**: tu perfil de Chrome existente mediante conexión automática de Chrome DevTools MCP

Valores predeterminados:

- El perfil `openclaw` se crea automáticamente si falta.
- El perfil `user` está integrado para la conexión existing-session mediante Chrome MCP.
- Los perfiles existing-session son optativos más allá de `user`; créalos con `--driver existing-session`.
- Los puertos CDP locales se asignan desde **18800–18899** de forma predeterminada.
- Al eliminar un perfil, su directorio de datos local se mueve a la Papelera.

Todos los endpoints de control aceptan `?profile=<name>`; la CLI usa `--browser-profile`.

## Existing-session mediante Chrome DevTools MCP

OpenClaw también puede adjuntarse a un perfil de navegador basado en Chromium en ejecución mediante el
servidor oficial Chrome DevTools MCP. Esto reutiliza las pestañas y el estado de inicio de sesión
ya abiertos en ese perfil de navegador.

Referencias oficiales de contexto y configuración:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

Perfil integrado:

- `user`

Opcional: crea tu propio perfil existing-session personalizado si quieres un
nombre, color o directorio de datos de navegador diferente.

Comportamiento predeterminado:

- El perfil integrado `user` usa conexión automática de Chrome MCP, que apunta al
  perfil local predeterminado de Google Chrome.

Usa `userDataDir` para Brave, Edge, Chromium o un perfil de Chrome no predeterminado:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

Luego, en el navegador correspondiente:

1. Abre la página de inspección de ese navegador para depuración remota.
2. Habilita la depuración remota.
3. Mantén el navegador en ejecución y aprueba el aviso de conexión cuando OpenClaw se adjunte.

Páginas de inspección habituales:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

Prueba rápida de conexión en vivo:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

Cómo es un resultado correcto:

- `status` muestra `driver: existing-session`
- `status` muestra `transport: chrome-mcp`
- `status` muestra `running: true`
- `tabs` enumera las pestañas del navegador que ya tienes abiertas
- `snapshot` devuelve refs de la pestaña activa seleccionada

Qué comprobar si la conexión no funciona:

- el navegador basado en Chromium de destino es versión `144+`
- la depuración remota está habilitada en la página de inspección de ese navegador
- el navegador mostró y aceptaste el aviso de consentimiento de conexión
- `openclaw doctor` migra la configuración antigua del navegador basada en extensiones y comprueba que
  Chrome esté instalado localmente para los perfiles predeterminados con conexión automática, pero no puede
  habilitar la depuración remota en el navegador por ti

Uso por parte del agente:

- Usa `profile="user"` cuando necesites el estado del navegador del usuario con sesión iniciada.
- Si usas un perfil existing-session personalizado, pasa ese nombre de perfil explícito.
- Elige este modo solo cuando el usuario esté en la computadora para aprobar el aviso
  de conexión.
- el Gateway o el host node pueden iniciar `npx chrome-devtools-mcp@latest --autoConnect`

Notas:

- Esta ruta tiene más riesgo que el perfil aislado `openclaw` porque puede
  actuar dentro de tu sesión del navegador con inicio de sesión.
- OpenClaw no inicia el navegador para este driver; solo se adjunta a una
  sesión existente.
- OpenClaw usa aquí el flujo oficial `--autoConnect` de Chrome DevTools MCP. Si
  `userDataDir` está establecido, OpenClaw lo pasa para apuntar a ese directorio explícito
  de datos de usuario Chromium.
- Las capturas de pantalla de existing-session admiten capturas de página y capturas de elementos con `--ref`
  desde snapshots, pero no selectores CSS `--element`.
- Las capturas de pantalla de página de existing-session funcionan sin Playwright mediante Chrome MCP.
  Las capturas de elementos basadas en refs (`--ref`) también funcionan ahí, pero `--full-page`
  no puede combinarse con `--ref` ni con `--element`.
- Las acciones de existing-session siguen siendo más limitadas que la ruta de navegador gestionado:
  - `click`, `type`, `hover`, `scrollIntoView`, `drag` y `select` requieren
    refs de snapshot en lugar de selectores CSS
  - `click` es solo con el botón izquierdo (sin anulaciones de botón ni modificadores)
  - `type` no admite `slowly=true`; usa `fill` o `press`
  - `press` no admite `delayMs`
  - `hover`, `scrollIntoView`, `drag`, `select`, `fill` y `evaluate` no
    admiten anulaciones de tiempo de espera por llamada
  - `select` actualmente admite solo un valor
- `wait --url` de existing-session admite patrones exactos, subcadenas y glob
  como otros drivers de navegador. `wait --load networkidle` todavía no está admitido.
- Los hooks de carga de existing-session requieren `ref` o `inputRef`, admiten un archivo
  a la vez y no admiten direccionamiento CSS `element`.
- Los hooks de diálogo de existing-session no admiten anulaciones de tiempo de espera.
- Algunas funciones siguen requiriendo la ruta de navegador gestionado, incluidas
  acciones por lotes, exportación de PDF, interceptación de descargas y `responsebody`.
- Existing-session es local al host. Si Chrome está en otra máquina o en un
  espacio de nombres de red diferente, usa CDP remoto o un host node en su lugar.

## Garantías de aislamiento

- **Directorio de datos de usuario dedicado**: nunca toca tu perfil de navegador personal.
- **Puertos dedicados**: evita `9222` para prevenir conflictos con flujos de trabajo de desarrollo.
- **Control determinista de pestañas**: apunta a las pestañas por `targetId`, no por “última pestaña”.

## Selección de navegador

Al iniciar localmente, OpenClaw elige el primero disponible:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

Puedes anularlo con `browser.executablePath`.

Plataformas:

- macOS: comprueba `/Applications` y `~/Applications`.
- Linux: busca `google-chrome`, `brave`, `microsoft-edge`, `chromium`, etc.
- Windows: comprueba ubicaciones de instalación habituales.

## API de control (opcional)

Solo para integraciones locales, el Gateway expone una pequeña API HTTP en loopback:

- Estado/inicio/detención: `GET /`, `POST /start`, `POST /stop`
- Pestañas: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`
- Snapshot/captura de pantalla: `GET /snapshot`, `POST /screenshot`
- Acciones: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Descargas: `POST /download`, `POST /wait/download`
- Depuración: `GET /console`, `POST /pdf`
- Depuración: `GET /errors`, `GET /requests`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Red: `POST /response/body`
- Estado: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- Estado: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Configuración: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

Todos los endpoints aceptan `?profile=<name>`.

Si la autenticación del gateway con secreto compartido está configurada, las rutas HTTP del navegador también requieren autenticación:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` o autenticación HTTP Basic con esa contraseña

Notas:

- Esta API independiente del navegador en loopback **no** consume cabeceras de identidad de proxy de confianza ni
  de Tailscale Serve.
- Si `gateway.auth.mode` es `none` o `trusted-proxy`, estas rutas del navegador en loopback
  no heredan esos modos con identidad; mantenlas solo en loopback.

### Requisito de Playwright

Algunas funciones (navigate/act/AI snapshot/role snapshot, capturas de elementos,
PDF) requieren Playwright. Si Playwright no está instalado, esos endpoints devuelven
un error 501 claro.

Lo que sigue funcionando sin Playwright:

- Snapshots ARIA
- Capturas de página para el navegador gestionado `openclaw` cuando hay un WebSocket CDP
  por pestaña disponible
- Capturas de página para perfiles `existing-session` / Chrome MCP
- Capturas de pantalla de existing-session basadas en refs (`--ref`) desde la salida de snapshot

Lo que todavía necesita Playwright:

- `navigate`
- `act`
- Snapshots AI / snapshots de rol
- Capturas de elementos con selector CSS (`--element`)
- Exportación completa de PDF del navegador

Las capturas de elementos también rechazan `--full-page`; la ruta devuelve `fullPage is
not supported for element screenshots`.

Si ves `Playwright is not available in this gateway build`, instala el paquete completo
de Playwright (no `playwright-core`) y reinicia el gateway, o reinstala
OpenClaw con soporte para navegador.

#### Instalación de Playwright en Docker

Si tu Gateway se ejecuta en Docker, evita `npx playwright` (conflictos de anulación de npm).
Usa la CLI incluida en su lugar:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Para conservar las descargas del navegador, establece `PLAYWRIGHT_BROWSERS_PATH` (por ejemplo,
`/home/node/.cache/ms-playwright`) y asegúrate de que `/home/node` se conserve mediante
`OPENCLAW_HOME_VOLUME` o un bind mount. Consulta [Docker](/es/install/docker).

## Cómo funciona (interno)

Flujo de alto nivel:

- Un pequeño **servidor de control** acepta solicitudes HTTP.
- Se conecta a navegadores basados en Chromium (Chrome/Brave/Edge/Chromium) mediante **CDP**.
- Para acciones avanzadas (clic/escritura/snapshot/PDF), usa **Playwright** sobre
  CDP.
- Cuando falta Playwright, solo están disponibles las operaciones que no usan Playwright.

Este diseño mantiene al agente sobre una interfaz estable y determinista, al tiempo que te permite cambiar
entre navegadores y perfiles locales/remotos.

## Referencia rápida de la CLI

Todos los comandos aceptan `--browser-profile <name>` para apuntar a un perfil específico.
Todos los comandos también aceptan `--json` para salida legible por máquina (payloads estables).

Básicos:

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

Inspección:

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`

Nota sobre el ciclo de vida:

- Para perfiles solo adjuntos y CDP remotos, `openclaw browser stop` sigue siendo el
  comando correcto de limpieza tras las pruebas. Cierra la sesión de control activa y
  limpia las anulaciones temporales de emulación en lugar de matar el navegador
  subyacente.
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

Acciones:

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

Estado:

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

Notas:

- `upload` y `dialog` son llamadas de **preparación**; ejecútalas antes del clic/pulsación
  que activa el selector/diálogo.
- Las rutas de salida de descargas y trazas están limitadas a raíces temporales de OpenClaw:
  - trazas: `/tmp/openclaw` (respaldo: `${os.tmpdir()}/openclaw`)
  - descargas: `/tmp/openclaw/downloads` (respaldo: `${os.tmpdir()}/openclaw/downloads`)
- Las rutas de subida están limitadas a una raíz temporal de subidas de OpenClaw:
  - subidas: `/tmp/openclaw/uploads` (respaldo: `${os.tmpdir()}/openclaw/uploads`)
- `upload` también puede establecer entradas de archivo directamente mediante `--input-ref` o `--element`.
- `snapshot`:
  - `--format ai` (predeterminado cuando Playwright está instalado): devuelve un snapshot AI con refs numéricos (`aria-ref="<n>"`).
  - `--format aria`: devuelve el árbol de accesibilidad (sin refs; solo inspección).
  - `--efficient` (o `--mode efficient`): preajuste compacto de snapshot de rol (interactive + compact + depth + menor maxChars).
  - Configuración predeterminada (solo herramienta/CLI): establece `browser.snapshotDefaults.mode: "efficient"` para usar snapshots eficientes cuando quien llama no pasa un modo (consulta [Configuración del Gateway](/es/gateway/configuration-reference#browser)).
  - Las opciones de snapshot de rol (`--interactive`, `--compact`, `--depth`, `--selector`) fuerzan un snapshot basado en roles con refs como `ref=e12`.
  - `--frame "<iframe selector>"` limita los snapshots de rol a un iframe (se combina con refs de rol como `e12`).
  - `--interactive` genera una lista plana y fácil de elegir de elementos interactivos (lo mejor para ejecutar acciones).
  - `--labels` agrega una captura de pantalla solo del viewport con etiquetas de ref superpuestas (imprime `MEDIA:<path>`).
- `click`/`type`/etc requieren un `ref` de `snapshot` (ya sea numérico `12` o ref de rol `e12`).
  Los selectores CSS no son compatibles intencionalmente para acciones.

## Snapshots y refs

OpenClaw admite dos estilos de “snapshot”:

- **Snapshot AI (refs numéricos)**: `openclaw browser snapshot` (predeterminado; `--format ai`)
  - Salida: un snapshot de texto que incluye refs numéricos.
  - Acciones: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Internamente, el ref se resuelve mediante `aria-ref` de Playwright.

- **Snapshot de rol (refs de rol como `e12`)**: `openclaw browser snapshot --interactive` (o `--compact`, `--depth`, `--selector`, `--frame`)
  - Salida: una lista/árbol basado en roles con `[ref=e12]` (y opcionalmente `[nth=1]`).
  - Acciones: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Internamente, el ref se resuelve mediante `getByRole(...)` (más `nth()` para duplicados).
  - Agrega `--labels` para incluir una captura de pantalla del viewport con etiquetas `e12` superpuestas.

Comportamiento de los refs:

- Los refs **no son estables entre navegaciones**; si algo falla, vuelve a ejecutar `snapshot` y usa un ref nuevo.
- Si el snapshot de rol se tomó con `--frame`, los refs de rol quedan limitados a ese iframe hasta el siguiente snapshot de rol.

## Potenciadores de wait

Puedes esperar algo más que tiempo/texto:

- Esperar una URL (globs admitidos por Playwright):
  - `openclaw browser wait --url "**/dash"`
- Esperar un estado de carga:
  - `openclaw browser wait --load networkidle`
- Esperar un predicado JS:
  - `openclaw browser wait --fn "window.ready===true"`
- Esperar a que un selector sea visible:
  - `openclaw browser wait "#main"`

Se pueden combinar:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Flujos de depuración

Cuando una acción falla (por ejemplo, “not visible”, “strict mode violation”, “covered”):

1. `openclaw browser snapshot --interactive`
2. Usa `click <ref>` / `type <ref>` (prefiere refs de rol en modo interactivo)
3. Si sigue fallando: `openclaw browser highlight <ref>` para ver a qué apunta Playwright
4. Si la página se comporta de forma extraña:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. Para depuración profunda: graba una traza:
   - `openclaw browser trace start`
   - reproduce el problema
   - `openclaw browser trace stop` (imprime `TRACE:<path>`)

## Salida JSON

`--json` es para scripting y herramientas estructuradas.

Ejemplos:

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

Los snapshots de rol en JSON incluyen `refs` más un pequeño bloque `stats` (líneas/caracteres/refs/interactivos) para que las herramientas puedan razonar sobre el tamaño y la densidad del payload.

## Controles de estado y entorno

Son útiles para flujos de trabajo de “hacer que el sitio se comporte como X”:

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Storage: `storage local|session get|set|clear`
- Sin conexión: `set offline on|off`
- Cabeceras: `set headers --headers-json '{"X-Debug":"1"}'` (el heredado `set headers --json '{"X-Debug":"1"}'` sigue siendo compatible)
- Autenticación HTTP Basic: `set credentials user pass` (o `--clear`)
- Geolocalización: `set geo <lat> <lon> --origin "https://example.com"` (o `--clear`)
- Multimedia: `set media dark|light|no-preference|none`
- Zona horaria / configuración regional: `set timezone ...`, `set locale ...`
- Dispositivo / viewport:
  - `set device "iPhone 14"` (preajustes de dispositivos Playwright)
  - `set viewport 1280 720`

## Seguridad y privacidad

- El perfil del navegador openclaw puede contener sesiones con inicio de sesión; trátalo como información sensible.
- `browser act kind=evaluate` / `openclaw browser evaluate` y `wait --fn`
  ejecutan JavaScript arbitrario en el contexto de la página. La inyección de prompts puede dirigir
  esto. Desactívalo con `browser.evaluateEnabled=false` si no lo necesitas.
- Para notas sobre inicios de sesión y anti-bot (X/Twitter, etc.), consulta [Inicio de sesión en el navegador + publicación en X/Twitter](/tools/browser-login).
- Mantén el Gateway/host node en privado (solo loopback o tailnet).
- Los endpoints CDP remotos son potentes; túnelalos y protégelos.

Ejemplo de modo estricto (bloquear destinos privados/internos de forma predeterminada):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optional exact allow
    },
  },
}
```

## Solución de problemas

Para problemas específicos de Linux (especialmente snap Chromium), consulta
[Solución de problemas del navegador](/tools/browser-linux-troubleshooting).

Para configuraciones divididas WSL2 Gateway + Windows Chrome en hosts distintos, consulta
[Solución de problemas de WSL2 + Windows + Chrome remoto por CDP](/tools/browser-wsl2-windows-remote-cdp-troubleshooting).

## Herramientas del agente + cómo funciona el control

El agente obtiene **una herramienta** para la automatización del navegador:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

Cómo se mapea:

- `browser snapshot` devuelve un árbol de interfaz estable (AI o ARIA).
- `browser act` usa los ID `ref` del snapshot para hacer clic/escribir/arrastrar/seleccionar.
- `browser screenshot` captura píxeles (página completa o elemento).
- `browser` acepta:
  - `profile` para elegir un perfil de navegador con nombre (openclaw, chrome o CDP remoto).
  - `target` (`sandbox` | `host` | `node`) para seleccionar dónde vive el navegador.
  - En sesiones con sandbox, `target: "host"` requiere `agents.defaults.sandbox.browser.allowHostControl=true`.
  - Si se omite `target`: las sesiones con sandbox usan `sandbox` de forma predeterminada, las sesiones sin sandbox usan `host`.
  - Si hay un nodo con capacidad de navegador conectado, la herramienta puede redirigirse automáticamente a él a menos que fijes `target="host"` o `target="node"`.

Esto mantiene al agente determinista y evita selectores frágiles.

## Relacionado

- [Resumen de herramientas](/tools) — todas las herramientas de agente disponibles
- [Sandboxing](/es/gateway/sandboxing) — control del navegador en entornos con sandbox
- [Seguridad](/es/gateway/security) — riesgos y endurecimiento del control del navegador
