---
read_when:
    - Ejecutas OpenClaw Gateway en WSL2 mientras Chrome vive en Windows
    - Ves errores superpuestos del navegador o de la UI de control entre WSL2 y Windows
    - Estás decidiendo entre Chrome MCP local al host y CDP remoto sin procesar en configuraciones divididas entre hosts
summary: Soluciona problemas de Gateway en WSL2 + Chrome remoto en Windows con CDP por capas
title: Solución de problemas de WSL2 + Windows + Chrome remoto con CDP
x-i18n:
    generated_at: "2026-04-05T12:55:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 99df2988d3c6cf36a8c2124d5b724228d095a60b2d2b552f3810709b5086127d
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 15
---

# Solución de problemas de WSL2 + Windows + Chrome remoto con CDP

Esta guía cubre la configuración dividida entre hosts más común, donde:

- OpenClaw Gateway se ejecuta dentro de WSL2
- Chrome se ejecuta en Windows
- el control del navegador debe cruzar el límite entre WSL2 y Windows

También cubre el patrón de fallos por capas del [issue #39369](https://github.com/openclaw/openclaw/issues/39369): pueden aparecer varios problemas independientes al mismo tiempo, lo que hace que primero parezca rota la capa equivocada.

## Primero elige el modo de navegador correcto

Tienes dos patrones válidos:

### Opción 1: CDP remoto sin procesar desde WSL2 a Windows

Usa un perfil de navegador remoto que apunte desde WSL2 a un endpoint CDP de Chrome en Windows.

Elige esta opción cuando:

- el Gateway permanece dentro de WSL2
- Chrome se ejecuta en Windows
- necesitas que el control del navegador cruce el límite entre WSL2 y Windows

### Opción 2: Chrome MCP local al host

Usa `existing-session` / `user` solo cuando el propio Gateway se ejecute en el mismo host que Chrome.

Elige esta opción cuando:

- OpenClaw y Chrome están en la misma máquina
- quieres el estado local del navegador con sesión iniciada
- no necesitas transporte del navegador entre hosts
- no necesitas rutas avanzadas gestionadas o exclusivas de raw CDP, como `responsebody`, exportación de PDF,
  interceptación de descargas o acciones por lotes

Para Gateway en WSL2 + Chrome en Windows, prefiere CDP remoto sin procesar. Chrome MCP es local al host, no un puente de WSL2 a Windows.

## Arquitectura funcional

Forma de referencia:

- WSL2 ejecuta el Gateway en `127.0.0.1:18789`
- Windows abre la UI de control en un navegador normal en `http://127.0.0.1:18789/`
- Chrome en Windows expone un endpoint CDP en el puerto `9222`
- WSL2 puede alcanzar ese endpoint CDP de Windows
- OpenClaw apunta un perfil de navegador a la dirección accesible desde WSL2

## Por qué esta configuración es confusa

Pueden superponerse varios fallos:

- WSL2 no puede alcanzar el endpoint CDP de Windows
- la UI de control se abre desde un origen no seguro
- `gateway.controlUi.allowedOrigins` no coincide con el origen de la página
- falta el token o el emparejamiento
- el perfil del navegador apunta a la dirección equivocada

Por eso, corregir una capa aún puede dejar visible un error diferente.

## Regla crítica para la UI de control

Cuando la UI se abra desde Windows, usa localhost de Windows, a menos que tengas una configuración HTTPS deliberada.

Usa:

`http://127.0.0.1:18789/`

No uses por defecto una IP de LAN para la UI de control. HTTP sin cifrar sobre una dirección LAN o de tailnet puede activar comportamientos de origen no seguro o autenticación de dispositivo que no están relacionados con CDP en sí. Consulta [Control UI](/web/control-ui).

## Valida por capas

Trabaja de arriba abajo. No te saltes pasos.

### Capa 1: Verifica que Chrome esté sirviendo CDP en Windows

Inicia Chrome en Windows con depuración remota habilitada:

```powershell
chrome.exe --remote-debugging-port=9222
```

Desde Windows, verifica primero el propio Chrome:

```powershell
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:9222/json/list
```

Si esto falla en Windows, OpenClaw todavía no es el problema.

### Capa 2: Verifica que WSL2 pueda alcanzar ese endpoint de Windows

Desde WSL2, prueba la dirección exacta que planeas usar en `cdpUrl`:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Buen resultado:

- `/json/version` devuelve JSON con metadatos de Browser / Protocol-Version
- `/json/list` devuelve JSON (un arreglo vacío está bien si no hay páginas abiertas)

Si esto falla:

- Windows aún no está exponiendo el puerto a WSL2
- la dirección es incorrecta para el lado de WSL2
- todavía falta firewall / reenvío de puertos / proxy local

Corrige eso antes de tocar la configuración de OpenClaw.

### Capa 3: Configura el perfil de navegador correcto

Para CDP remoto sin procesar, apunta OpenClaw a la dirección accesible desde WSL2:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Notas:

- usa la dirección accesible desde WSL2, no la que solo funciona en Windows
- mantén `attachOnly: true` para navegadores gestionados externamente
- `cdpUrl` puede ser `http://`, `https://`, `ws://` o `wss://`
- usa HTTP(S) cuando quieras que OpenClaw detecte `/json/version`
- usa WS(S) solo cuando el proveedor del navegador te dé una URL directa del socket DevTools
- prueba la misma URL con `curl` antes de esperar que OpenClaw funcione

### Capa 4: Verifica por separado la capa de la UI de control

Abre la UI desde Windows:

`http://127.0.0.1:18789/`

Luego verifica:

- que el origen de la página coincida con lo que espera `gateway.controlUi.allowedOrigins`
- que la autenticación por token o el emparejamiento estén configurados correctamente
- que no estés depurando un problema de autenticación de la UI de control como si fuera un problema del navegador

Página útil:

- [Control UI](/web/control-ui)

### Capa 5: Verifica el control del navegador de extremo a extremo

Desde WSL2:

```bash
openclaw browser open https://example.com --browser-profile remote
openclaw browser tabs --browser-profile remote
```

Buen resultado:

- la pestaña se abre en Chrome de Windows
- `openclaw browser tabs` devuelve el destino
- acciones posteriores (`snapshot`, `screenshot`, `navigate`) funcionan desde el mismo perfil

## Errores engañosos comunes

Trata cada mensaje como una pista específica de una capa:

- `control-ui-insecure-auth`
  - problema de origen de la UI / contexto seguro, no un problema de transporte de CDP
- `token_missing`
  - problema de configuración de autenticación
- `pairing required`
  - problema de aprobación del dispositivo
- `Remote CDP for profile "remote" is not reachable`
  - WSL2 no puede alcanzar el `cdpUrl` configurado
- `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable`
  - el endpoint HTTP respondió, pero aún no se pudo abrir el WebSocket de DevTools
- anulaciones obsoletas de viewport / modo oscuro / configuración regional / modo sin conexión después de una sesión remota
  - ejecuta `openclaw browser stop --browser-profile remote`
  - esto cierra la sesión de control activa y libera el estado de emulación de Playwright/CDP sin reiniciar el gateway ni el navegador externo
- `gateway timeout after 1500ms`
  - a menudo sigue siendo un problema de alcance de CDP o de un endpoint remoto lento o inaccesible
- `No Chrome tabs found for profile="user"`
  - se seleccionó un perfil local de Chrome MCP donde no hay pestañas locales del host disponibles

## Lista rápida de comprobación para triage

1. Windows: ¿funciona `curl http://127.0.0.1:9222/json/version`?
2. WSL2: ¿funciona `curl http://WINDOWS_HOST_OR_IP:9222/json/version`?
3. Configuración de OpenClaw: ¿`browser.profiles.<name>.cdpUrl` usa exactamente esa dirección accesible desde WSL2?
4. UI de control: ¿estás abriendo `http://127.0.0.1:18789/` en lugar de una IP de LAN?
5. ¿Estás intentando usar `existing-session` entre WSL2 y Windows en vez de CDP remoto sin procesar?

## Conclusión práctica

Esta configuración normalmente es viable. La parte difícil es que el transporte del navegador, la seguridad del origen de la UI de control y el token/emparejamiento pueden fallar de forma independiente mientras desde el lado del usuario parecen similares.

En caso de duda:

- primero verifica localmente el endpoint de Chrome en Windows
- después verifica ese mismo endpoint desde WSL2
- solo entonces depura la configuración de OpenClaw o la autenticación de la UI de control
