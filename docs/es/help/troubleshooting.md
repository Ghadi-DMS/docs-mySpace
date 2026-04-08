---
read_when:
    - OpenClaw no funciona y necesita la vía más rápida para corregirlo
    - Quiere un flujo de triaje antes de profundizar en guías detalladas
summary: Centro de solución de problemas de OpenClaw orientado primero a los síntomas
title: Solución general de problemas
x-i18n:
    generated_at: "2026-04-08T02:16:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8abda90ef80234c2f91a51c5e1f2c004d4a4da12a5d5631b5927762550c6d5e3
    source_path: help/troubleshooting.md
    workflow: 15
---

# Solución de problemas

Si solo tiene 2 minutos, use esta página como punto de entrada para el triaje.

## Primeros 60 segundos

Ejecute esta secuencia exacta en orden:

```bash
openclaw status
openclaw status --all
openclaw gateway probe
openclaw gateway status
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

Buen resultado en una línea:

- `openclaw status` → muestra los canales configurados y ningún error de autenticación evidente.
- `openclaw status --all` → el informe completo está presente y se puede compartir.
- `openclaw gateway probe` → se puede acceder al destino esperado del gateway (`Reachable: yes`). `RPC: limited - missing scope: operator.read` indica diagnósticos degradados, no un fallo de conexión.
- `openclaw gateway status` → `Runtime: running` y `RPC probe: ok`.
- `openclaw doctor` → no hay errores de configuración/servicio que bloqueen.
- `openclaw channels status --probe` → si se puede acceder al gateway, devuelve el estado de transporte en vivo por cuenta más resultados de sondeo/auditoría como `works` o `audit ok`; si no se puede acceder al gateway, el comando usa como alternativa resúmenes basados solo en la configuración.
- `openclaw logs --follow` → actividad constante, sin errores fatales repetidos.

## 429 de Anthropic para contexto largo

Si ve:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`,
vaya a [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/es/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

## El backend local compatible con OpenAI funciona directamente pero falla en OpenClaw

Si su backend local o autoalojado `/v1` responde a sondeos directos pequeños de
`/v1/chat/completions` pero falla en `openclaw infer model run` o en los turnos normales
del agente:

1. Si el error menciona que `messages[].content` espera una cadena, configure
   `models.providers.<provider>.models[].compat.requiresStringContent: true`.
2. Si el backend sigue fallando solo en los turnos del agente de OpenClaw, configure
   `models.providers.<provider>.models[].compat.supportsTools: false` y vuelva a intentarlo.
3. Si las llamadas directas mínimas siguen funcionando pero los prompts más grandes de OpenClaw hacen fallar el
   backend, trate el problema restante como una limitación ascendente del modelo/servidor y
   continúe en la guía detallada:
   [/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail](/es/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)

## La instalación del plugin falla por falta de extensiones de openclaw

Si la instalación falla con `package.json missing openclaw.extensions`, el paquete del plugin
está usando una forma antigua que OpenClaw ya no acepta.

Corríjalo en el paquete del plugin:

1. Agregue `openclaw.extensions` a `package.json`.
2. Apunte las entradas a archivos de tiempo de ejecución compilados (normalmente `./dist/index.js`).
3. Vuelva a publicar el plugin y ejecute `openclaw plugins install <package>` otra vez.

Ejemplo:

```json
{
  "name": "@openclaw/my-plugin",
  "version": "1.2.3",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

Referencia: [Arquitectura de plugins](/es/plugins/architecture)

## Árbol de decisiones

```mermaid
flowchart TD
  A[OpenClaw no funciona] --> B{Qué es lo primero que falla}
  B --> C[No hay respuestas]
  B --> D[Dashboard o Control UI no se conectan]
  B --> E[El gateway no inicia o el servicio no está en ejecución]
  B --> F[El canal se conecta pero los mensajes no fluyen]
  B --> G[Cron o heartbeat no se ejecutó o no entregó]
  B --> H[El nodo está emparejado pero falla la herramienta camera canvas screen exec]
  B --> I[Falla la herramienta del navegador]

  C --> C1[/Sección Sin respuestas/]
  D --> D1[/Sección Control UI/]
  E --> E1[/Sección Gateway/]
  F --> F1[/Sección Flujo del canal/]
  G --> G1[/Sección Automatización/]
  H --> H1[/Sección Herramientas del nodo/]
  I --> I1[/Sección Navegador/]
```

<AccordionGroup>
  <Accordion title="Sin respuestas">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw channels status --probe
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    ```

    Un buen resultado se ve así:

    - `Runtime: running`
    - `RPC probe: ok`
    - Su canal muestra el transporte conectado y, donde se admite, `works` o `audit ok` en `channels status --probe`
    - El remitente aparece como aprobado (o la política de DM es open/allowlist)

    Firmas comunes en los registros:

    - `drop guild message (mention required` → la restricción por mención bloqueó el mensaje en Discord.
    - `pairing request` → el remitente no está aprobado y está esperando la aprobación del emparejamiento por DM.
    - `blocked` / `allowlist` en los registros del canal → el remitente, la sala o el grupo está filtrado.

    Páginas detalladas:

    - [/gateway/troubleshooting#no-replies](/es/gateway/troubleshooting#no-replies)
    - [/channels/troubleshooting](/es/channels/troubleshooting)
    - [/channels/pairing](/es/channels/pairing)

  </Accordion>

  <Accordion title="Dashboard o Control UI no se conectan">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un buen resultado se ve así:

    - `Dashboard: http://...` aparece en `openclaw gateway status`
    - `RPC probe: ok`
    - No hay bucle de autenticación en los registros

    Firmas comunes en los registros:

    - `device identity required` → el contexto HTTP/no seguro no puede completar la autenticación del dispositivo.
    - `origin not allowed` → el `Origin` del navegador no está permitido para el destino del gateway de Control UI.
    - `AUTH_TOKEN_MISMATCH` con sugerencias de reintento (`canRetryWithDeviceToken=true`) → puede producirse automáticamente un reintento con token de dispositivo de confianza.
    - Ese reintento con token en caché reutiliza el conjunto de alcances en caché almacenado con el token del dispositivo emparejado. Los llamadores con `deviceToken` explícito o `scopes` explícitos conservan en su lugar el conjunto de alcances solicitado.
    - En la ruta asíncrona de Tailscale Serve para Control UI, los intentos fallidos para el mismo `{scope, ip}` se serializan antes de que el limitador registre el fallo, por lo que un segundo reintento incorrecto concurrente ya puede mostrar `retry later`.
    - `too many failed authentication attempts (retry later)` desde un origen de navegador localhost → los fallos repetidos desde ese mismo `Origin` quedan bloqueados temporalmente; otro origen localhost usa un depósito separado.
    - `unauthorized` repetido después de ese reintento → token/contraseña incorrectos, discrepancia en el modo de autenticación o token de dispositivo emparejado obsoleto.
    - `gateway connect failed:` → la UI apunta a la URL/puerto equivocado o a un gateway inaccesible.

    Páginas detalladas:

    - [/gateway/troubleshooting#dashboard-control-ui-connectivity](/es/gateway/troubleshooting#dashboard-control-ui-connectivity)
    - [/web/control-ui](/web/control-ui)
    - [/gateway/authentication](/es/gateway/authentication)

  </Accordion>

  <Accordion title="El gateway no inicia o el servicio está instalado pero no está en ejecución">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un buen resultado se ve así:

    - `Service: ... (loaded)`
    - `Runtime: running`
    - `RPC probe: ok`

    Firmas comunes en los registros:

    - `Gateway start blocked: set gateway.mode=local` o `existing config is missing gateway.mode` → el modo del gateway es remoto, o al archivo de configuración le falta la marca de modo local y debe repararse.
    - `refusing to bind gateway ... without auth` → enlace sin loopback sin una ruta de autenticación válida del gateway (token/contraseña, o trusted-proxy cuando esté configurado).
    - `another gateway instance is already listening` o `EADDRINUSE` → el puerto ya está en uso.

    Páginas detalladas:

    - [/gateway/troubleshooting#gateway-service-not-running](/es/gateway/troubleshooting#gateway-service-not-running)
    - [/gateway/background-process](/es/gateway/background-process)
    - [/gateway/configuration](/es/gateway/configuration)

  </Accordion>

  <Accordion title="El canal se conecta pero los mensajes no fluyen">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un buen resultado se ve así:

    - El transporte del canal está conectado.
    - Las comprobaciones de emparejamiento/lista de permitidos pasan.
    - Las menciones se detectan cuando son necesarias.

    Firmas comunes en los registros:

    - `mention required` → la restricción por mención bloqueó el procesamiento del mensaje.
    - `pairing` / `pending` → el remitente del DM aún no está aprobado.
    - `not_in_channel`, `missing_scope`, `Forbidden`, `401/403` → problema de token de permisos del canal.

    Páginas detalladas:

    - [/gateway/troubleshooting#channel-connected-messages-not-flowing](/es/gateway/troubleshooting#channel-connected-messages-not-flowing)
    - [/channels/troubleshooting](/es/channels/troubleshooting)

  </Accordion>

  <Accordion title="Cron o heartbeat no se ejecutó o no entregó">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw cron status
    openclaw cron list
    openclaw cron runs --id <jobId> --limit 20
    openclaw logs --follow
    ```

    Un buen resultado se ve así:

    - `cron.status` aparece como habilitado con un próximo despertar.
    - `cron runs` muestra entradas recientes `ok`.
    - Heartbeat está habilitado y no está fuera de las horas activas.

    Firmas comunes en los registros:

- `cron: scheduler disabled; jobs will not run automatically` → cron está deshabilitado.
- `heartbeat skipped` con `reason=quiet-hours` → fuera de las horas activas configuradas.
- `heartbeat skipped` con `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe pero solo contiene estructura vacía o solo encabezados.
- `heartbeat skipped` con `reason=no-tasks-due` → el modo de tareas de `HEARTBEAT.md` está activo, pero aún no venció ninguno de los intervalos de tareas.
- `heartbeat skipped` con `reason=alerts-disabled` → toda la visibilidad de heartbeat está deshabilitada (`showOk`, `showAlerts` y `useIndicator` están todos apagados).
- `requests-in-flight` → el carril principal está ocupado; el despertar de heartbeat se aplazó. - `unknown accountId` → el destino de entrega de la cuenta de heartbeat no existe.

      Páginas detalladas:

      - [/gateway/troubleshooting#cron-and-heartbeat-delivery](/es/gateway/troubleshooting#cron-and-heartbeat-delivery)
      - [/automation/cron-jobs#troubleshooting](/es/automation/cron-jobs#troubleshooting)
      - [/gateway/heartbeat](/es/gateway/heartbeat)

    </Accordion>

    <Accordion title="El nodo está emparejado pero falla la herramienta camera canvas screen exec">
      ```bash
      openclaw status
      openclaw gateway status
      openclaw nodes status
      openclaw nodes describe --node <idOrNameOrIp>
      openclaw logs --follow
      ```

      Un buen resultado se ve así:

      - El nodo aparece como conectado y emparejado para el rol `node`.
      - La capacidad existe para el comando que está invocando.
      - El estado de permisos está concedido para la herramienta.

      Firmas comunes en los registros:

      - `NODE_BACKGROUND_UNAVAILABLE` → lleve la app del nodo al primer plano.
      - `*_PERMISSION_REQUIRED` → el permiso del sistema operativo fue denegado o falta.
      - `SYSTEM_RUN_DENIED: approval required` → la aprobación de exec está pendiente.
      - `SYSTEM_RUN_DENIED: allowlist miss` → el comando no está en la lista de permitidos de exec.

      Páginas detalladas:

      - [/gateway/troubleshooting#node-paired-tool-fails](/es/gateway/troubleshooting#node-paired-tool-fails)
      - [/nodes/troubleshooting](/es/nodes/troubleshooting)
      - [/tools/exec-approvals](/es/tools/exec-approvals)

    </Accordion>

    <Accordion title="Exec de repente pide aprobación">
      ```bash
      openclaw config get tools.exec.host
      openclaw config get tools.exec.security
      openclaw config get tools.exec.ask
      openclaw gateway restart
      ```

      Qué cambió:

      - Si `tools.exec.host` no está configurado, el valor predeterminado es `auto`.
      - `host=auto` se resuelve como `sandbox` cuando hay un tiempo de ejecución de sandbox activo, `gateway` en caso contrario.
      - `host=auto` es solo enrutamiento; el comportamiento “YOLO” sin solicitud proviene de `security=full` más `ask=off` en gateway/node.
      - En `gateway` y `node`, si `tools.exec.security` no está configurado, el valor predeterminado es `full`.
      - Si `tools.exec.ask` no está configurado, el valor predeterminado es `off`.
      - Resultado: si está viendo aprobaciones, alguna política local del host o por sesión endureció exec con respecto a los valores predeterminados actuales.

      Restaure el comportamiento actual predeterminado sin aprobación:

      ```bash
      openclaw config set tools.exec.host gateway
      openclaw config set tools.exec.security full
      openclaw config set tools.exec.ask off
      openclaw gateway restart
      ```

      Alternativas más seguras:

      - Configure solo `tools.exec.host=gateway` si solo quiere un enrutamiento estable del host.
      - Use `security=allowlist` con `ask=on-miss` si quiere exec del host pero aún desea revisión en los fallos de lista de permitidos.
      - Habilite el modo sandbox si quiere que `host=auto` vuelva a resolverse como `sandbox`.

      Firmas comunes en los registros:

      - `Approval required.` → el comando está esperando `/approve ...`.
      - `SYSTEM_RUN_DENIED: approval required` → la aprobación de exec en el host del nodo está pendiente.
      - `exec host=sandbox requires a sandbox runtime for this session` → selección implícita/explícita de sandbox, pero el modo sandbox está desactivado.

      Páginas detalladas:

      - [/tools/exec](/es/tools/exec)
      - [/tools/exec-approvals](/es/tools/exec-approvals)
      - [/gateway/security#runtime-expectation-drift](/es/gateway/security#runtime-expectation-drift)

    </Accordion>

    <Accordion title="Falla la herramienta del navegador">
      ```bash
      openclaw status
      openclaw gateway status
      openclaw browser status
      openclaw logs --follow
      openclaw doctor
      ```

      Un buen resultado se ve así:

      - El estado del navegador muestra `running: true` y un navegador/perfil seleccionado.
      - `openclaw` se inicia, o `user` puede ver pestañas locales de Chrome.

      Firmas comunes en los registros:

      - `unknown command "browser"` o `unknown command 'browser'` → `plugins.allow` está configurado y no incluye `browser`.
      - `Failed to start Chrome CDP on port` → falló el inicio del navegador local.
      - `browser.executablePath not found` → la ruta binaria configurada es incorrecta.
      - `browser.cdpUrl must be http(s) or ws(s)` → la URL de CDP configurada usa un esquema no compatible.
      - `browser.cdpUrl has invalid port` → la URL de CDP configurada tiene un puerto incorrecto o fuera de rango.
      - `No Chrome tabs found for profile="user"` → el perfil de conexión de Chrome MCP no tiene pestañas locales de Chrome abiertas.
      - `Remote CDP for profile "<name>" is not reachable` → no se puede acceder desde este host al endpoint remoto de CDP configurado.
      - `Browser attachOnly is enabled ... not reachable` o `Browser attachOnly is enabled and CDP websocket ... is not reachable` → el perfil solo de conexión no tiene un destino CDP activo.
      - sobrescrituras obsoletas de viewport / modo oscuro / configuración regional / sin conexión en perfiles solo de conexión o CDP remotos → ejecute `openclaw browser stop --browser-profile <name>` para cerrar la sesión de control activa y liberar el estado de emulación sin reiniciar el gateway.

      Páginas detalladas:

      - [/gateway/troubleshooting#browser-tool-fails](/es/gateway/troubleshooting#browser-tool-fails)
      - [/tools/browser#missing-browser-command-or-tool](/es/tools/browser#missing-browser-command-or-tool)
      - [/tools/browser-linux-troubleshooting](/es/tools/browser-linux-troubleshooting)
      - [/tools/browser-wsl2-windows-remote-cdp-troubleshooting](/es/tools/browser-wsl2-windows-remote-cdp-troubleshooting)

    </Accordion>
  </AccordionGroup>

## Relacionado

- [Preguntas frecuentes](/es/help/faq) — preguntas frecuentes
- [Solución de problemas del gateway](/es/gateway/troubleshooting) — problemas específicos del gateway
- [Doctor](/es/gateway/doctor) — comprobaciones automáticas de estado y reparaciones
- [Solución de problemas de canales](/es/channels/troubleshooting) — problemas de conectividad de canales
- [Solución de problemas de automatización](/es/automation/cron-jobs#troubleshooting) — problemas de cron y heartbeat
