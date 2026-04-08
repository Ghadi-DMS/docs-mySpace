---
read_when:
    - Responder preguntas comunes sobre configuración, instalación, onboarding o soporte en tiempo de ejecución
    - Clasificar problemas informados por usuarios antes de una depuración más profunda
summary: Preguntas frecuentes sobre la configuración, instalación y uso de OpenClaw
title: Preguntas frecuentes
x-i18n:
    generated_at: "2026-04-08T02:21:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 001b4605966b45b08108606f76ae937ec348c2179b04cf6fb34fef94833705e6
    source_path: help/faq.md
    workflow: 15
---

# Preguntas frecuentes

Respuestas rápidas más solución de problemas más profunda para configuraciones reales (desarrollo local, VPS, multiagente, OAuth/claves de API, conmutación por error de modelos). Para diagnósticos en tiempo de ejecución, consulta [Solución de problemas](/es/gateway/troubleshooting). Para la referencia completa de configuración, consulta [Configuración](/es/gateway/configuration).

## Primeros 60 segundos si algo está roto

1. **Estado rápido (primera comprobación)**

   ```bash
   openclaw status
   ```

   Resumen local rápido: SO + actualización, alcance de la gateway/servicio, agentes/sesiones, configuración del proveedor + problemas en tiempo de ejecución (cuando la gateway es accesible).

2. **Informe que se puede compartir (seguro para compartir)**

   ```bash
   openclaw status --all
   ```

   Diagnóstico de solo lectura con cola del registro (tokens redactados).

3. **Estado del daemon + puerto**

   ```bash
   openclaw gateway status
   ```

   Muestra el tiempo de ejecución del supervisor frente al alcance RPC, la URL objetivo del sondeo y qué configuración probablemente usó el servicio.

4. **Sondeos profundos**

   ```bash
   openclaw status --deep
   ```

   Ejecuta un sondeo de salud en vivo de la gateway, incluidos sondeos de canales cuando están soportados
   (requiere una gateway accesible). Consulta [Salud](/es/gateway/health).

5. **Seguir el registro más reciente**

   ```bash
   openclaw logs --follow
   ```

   Si RPC no responde, usa esto como alternativa:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Los registros de archivo están separados de los registros del servicio; consulta [Registro](/es/logging) y [Solución de problemas](/es/gateway/troubleshooting).

6. **Ejecutar Doctor (reparaciones)**

   ```bash
   openclaw doctor
   ```

   Repara/migra configuración/estado + ejecuta comprobaciones de salud. Consulta [Doctor](/es/gateway/doctor).

7. **Instantánea de la gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Pide a la gateway en ejecución una instantánea completa (solo WS). Consulta [Salud](/es/gateway/health).

## Inicio rápido y configuración de primera ejecución

<AccordionGroup>
  <Accordion title="Estoy atascado, ¿cuál es la forma más rápida de salir del atasco?">
    Usa un agente de IA local que pueda **ver tu máquina**. Eso es mucho más efectivo que preguntar
    en Discord, porque la mayoría de los casos de “estoy atascado” son **problemas de configuración local o de entorno**
    que los ayudantes remotos no pueden inspeccionar.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Estas herramientas pueden leer el repositorio, ejecutar comandos, inspeccionar registros y ayudarte a corregir la configuración
    de tu máquina (PATH, servicios, permisos, archivos de autenticación). Dales el **checkout completo del código fuente** mediante
    la instalación modificable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Esto instala OpenClaw **desde un checkout de git**, para que el agente pueda leer el código + la documentación y
    razonar sobre la versión exacta que estás ejecutando. Siempre puedes volver a la versión estable más tarde
    ejecutando de nuevo el instalador sin `--install-method git`.

    Consejo: pídele al agente que **planifique y supervise** la corrección (paso a paso), y luego que ejecute solo los
    comandos necesarios. Eso mantiene los cambios pequeños y más fáciles de auditar.

    Si descubres un error real o una corrección, por favor crea un issue en GitHub o envía un PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Empieza con estos comandos (comparte las salidas cuando pidas ayuda):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Qué hacen:

    - `openclaw status`: instantánea rápida de la salud de la gateway/agente + configuración básica.
    - `openclaw models status`: comprueba la autenticación del proveedor + disponibilidad del modelo.
    - `openclaw doctor`: valida y repara problemas comunes de configuración/estado.

    Otras comprobaciones útiles de la CLI: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Bucle rápido de depuración: [Primeros 60 segundos si algo está roto](#primeros-60-segundos-si-algo-está-roto).
    Documentación de instalación: [Instalación](/es/install), [Flags del instalador](/es/install/installer), [Actualización](/es/install/updating).

  </Accordion>

  <Accordion title="Heartbeat sigue omitiéndose. ¿Qué significan los motivos de omisión?">
    Motivos comunes de omisión de heartbeat:

    - `quiet-hours`: fuera de la ventana configurada de horas activas
    - `empty-heartbeat-file`: `HEARTBEAT.md` existe pero solo contiene andamiaje vacío o solo encabezados
    - `no-tasks-due`: el modo de tareas de `HEARTBEAT.md` está activo pero aún no vence ninguno de los intervalos de tareas
    - `alerts-disabled`: toda la visibilidad de heartbeat está desactivada (`showOk`, `showAlerts` y `useIndicator` están todos apagados)

    En modo de tareas, las marcas de tiempo de vencimiento solo se adelantan después de que se complete
    una ejecución real de heartbeat. Las ejecuciones omitidas no marcan tareas como completadas.

    Documentación: [Heartbeat](/es/gateway/heartbeat), [Automatización y tareas](/es/automation).

  </Accordion>

  <Accordion title="Forma recomendada de instalar y configurar OpenClaw">
    El repositorio recomienda ejecutar desde el código fuente y usar onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    El asistente también puede compilar recursos de la UI automáticamente. Después del onboarding, normalmente ejecutas la Gateway en el puerto **18789**.

    Desde el código fuente (colaboradores/desarrollo):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    Si todavía no tienes una instalación global, ejecútalo mediante `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="¿Cómo abro el panel después del onboarding?">
    El asistente abre tu navegador con una URL limpia del panel (sin token) justo después del onboarding y también imprime el enlace en el resumen. Mantén esa pestaña abierta; si no se abrió, copia y pega la URL impresa en la misma máquina.
  </Accordion>

  <Accordion title="¿Cómo autentico el panel en localhost frente a remoto?">
    **Localhost (misma máquina):**

    - Abre `http://127.0.0.1:18789/`.
    - Si pide autenticación con secreto compartido, pega el token o la contraseña configurados en la configuración de Control UI.
    - Origen del token: `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
    - Origen de la contraseña: `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
    - Si todavía no hay un secreto compartido configurado, genera un token con `openclaw doctor --generate-gateway-token`.

    **No en localhost:**

    - **Tailscale Serve** (recomendado): mantén bind loopback, ejecuta `openclaw gateway --tailscale serve`, abre `https://<magicdns>/`. Si `gateway.auth.allowTailscale` es `true`, los encabezados de identidad satisfacen la autenticación de Control UI/WebSocket (sin pegar secreto compartido, asume un host de gateway de confianza); las API HTTP siguen requiriendo autenticación con secreto compartido salvo que uses deliberadamente `none` para ingreso privado o autenticación HTTP de proxy de confianza.
      Los intentos simultáneos fallidos de autenticación de Serve desde el mismo cliente se serializan antes de que el limitador de autenticación fallida los registre, por lo que el segundo reintento fallido ya puede mostrar `retry later`.
    - **Bind de tailnet**: ejecuta `openclaw gateway --bind tailnet --token "<token>"` (o configura autenticación por contraseña), abre `http://<tailscale-ip>:18789/`, y luego pega el secreto compartido correspondiente en la configuración del panel.
    - **Proxy inverso con reconocimiento de identidad**: mantén la Gateway detrás de un proxy de confianza no loopback, configura `gateway.auth.mode: "trusted-proxy"`, y luego abre la URL del proxy.
    - **Túnel SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@host` y luego abre `http://127.0.0.1:18789/`. La autenticación por secreto compartido sigue aplicándose a través del túnel; pega el token o la contraseña configurados si se solicita.

    Consulta [Panel](/web/dashboard) y [Superficies web](/web) para detalles sobre modos de bind y autenticación.

  </Accordion>

  <Accordion title="¿Por qué hay dos configuraciones de aprobación de exec para aprobaciones por chat?">
    Controlan capas distintas:

    - `approvals.exec`: reenvía solicitudes de aprobación a destinos de chat
    - `channels.<channel>.execApprovals`: hace que ese canal actúe como cliente nativo de aprobación para aprobaciones de exec

    La política de exec del host sigue siendo la puerta real de aprobación. La configuración del chat solo controla dónde aparecen
    las solicitudes de aprobación y cómo puede responder la gente.

    En la mayoría de las configuraciones **no** necesitas ambas:

    - Si el chat ya soporta comandos y respuestas, `/approve` en el mismo chat funciona a través de la ruta compartida.
    - Si un canal nativo compatible puede inferir aprobadores de forma segura, OpenClaw ahora habilita automáticamente aprobaciones nativas con prioridad a DM cuando `channels.<channel>.execApprovals.enabled` no está configurado o es `"auto"`.
    - Cuando hay tarjetas/botones nativos de aprobación disponibles, esa UI nativa es la ruta principal; el agente solo debe incluir un comando manual `/approve` si el resultado de la herramienta dice que las aprobaciones por chat no están disponibles o que la aprobación manual es la única vía.
    - Usa `approvals.exec` solo cuando las solicitudes también deban reenviarse a otros chats o salas operativas explícitas.
    - Usa `channels.<channel>.execApprovals.target: "channel"` o `"both"` solo cuando quieras explícitamente que las solicitudes de aprobación se publiquen de vuelta en la sala/tema de origen.
    - Las aprobaciones de plugins se separan otra vez: usan `/approve` en el mismo chat por defecto, reenvío opcional de `approvals.plugin`, y solo algunos canales nativos mantienen además un manejo nativo de aprobación de plugins.

    En resumen: el reenvío es para el enrutamiento, la configuración del cliente nativo es para una UX más rica y específica del canal.
    Consulta [Aprobaciones de exec](/es/tools/exec-approvals).

  </Accordion>

  <Accordion title="¿Qué runtime necesito?">
    Se requiere Node **>= 22**. Se recomienda `pnpm`. Bun **no se recomienda** para la Gateway.
  </Accordion>

  <Accordion title="¿Funciona en Raspberry Pi?">
    Sí. La Gateway es ligera: la documentación indica que **512MB-1GB de RAM**, **1 núcleo** y alrededor de **500MB**
    de disco son suficientes para uso personal, y señala que una **Raspberry Pi 4 puede ejecutarla**.

    Si quieres margen adicional (registros, medios, otros servicios), se recomiendan **2GB**,
    pero no es un mínimo estricto.

    Consejo: una Pi/VPS pequeña puede alojar la Gateway, y puedes emparejar **nodos** en tu portátil/teléfono para
    pantalla/cámara/canvas locales o ejecución de comandos. Consulta [Nodos](/es/nodes).

  </Accordion>

  <Accordion title="¿Algún consejo para instalaciones en Raspberry Pi?">
    Versión corta: funciona, pero espera algunas asperezas.

    - Usa un SO de **64 bits** y mantén Node >= 22.
    - Prefiere la **instalación modificable (git)** para poder ver registros y actualizar rápido.
    - Empieza sin canales/Skills y luego añádelos uno por uno.
    - Si encuentras problemas binarios raros, normalmente es un problema de **compatibilidad ARM**.

    Documentación: [Linux](/es/platforms/linux), [Instalación](/es/install).

  </Accordion>

  <Accordion title="Se queda atascado en wake up my friend / el onboarding no termina de salir. ¿Y ahora qué?">
    Esa pantalla depende de que la Gateway sea accesible y esté autenticada. La TUI también envía
    “Wake up, my friend!” automáticamente en el primer inicio. Si ves esa línea **sin respuesta**
    y los tokens se quedan en 0, el agente nunca se ejecutó.

    1. Reinicia la Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. Comprueba estado + autenticación:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Si sigue colgado, ejecuta:

    ```bash
    openclaw doctor
    ```

    Si la Gateway es remota, asegúrate de que el túnel/la conexión Tailscale esté activa y de que la UI
    apunte a la Gateway correcta. Consulta [Acceso remoto](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Puedo migrar mi configuración a una máquina nueva (Mac mini) sin rehacer el onboarding?">
    Sí. Copia el **directorio de estado** y el **workspace**, luego ejecuta Doctor una vez. Esto
    mantiene tu bot “exactamente igual” (memoria, historial de sesiones, autenticación y
    estado de canales) siempre que copies **ambas** ubicaciones:

    1. Instala OpenClaw en la nueva máquina.
    2. Copia `$OPENCLAW_STATE_DIR` (predeterminado: `~/.openclaw`) desde la máquina anterior.
    3. Copia tu workspace (predeterminado: `~/.openclaw/workspace`).
    4. Ejecuta `openclaw doctor` y reinicia el servicio Gateway.

    Eso conserva configuración, perfiles de autenticación, credenciales de WhatsApp, sesiones y memoria. Si estás en
    modo remoto, recuerda que el host de la gateway es el propietario del almacén de sesiones y del workspace.

    **Importante:** si solo haces commit/push de tu workspace a GitHub, estás respaldando
    **memoria + archivos de arranque**, pero **no** el historial de sesiones ni la autenticación. Esos viven
    en `~/.openclaw/` (por ejemplo `~/.openclaw/agents/<agentId>/sessions/`).

    Relacionado: [Migración](/es/install/migrating), [Dónde vive cada cosa en disco](#donde-vive-cada-cosa-en-disco),
    [Workspace del agente](/es/concepts/agent-workspace), [Doctor](/es/gateway/doctor),
    [Modo remoto](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde veo qué hay de nuevo en la última versión?">
    Consulta el changelog de GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Las entradas más nuevas están arriba. Si la sección superior está marcada como **Unreleased**, la siguiente
    sección con fecha es la última versión publicada. Las entradas se agrupan por **Highlights**, **Changes** y
    **Fixes** (más secciones de docs/otras cuando hace falta).

  </Accordion>

  <Accordion title="No puedo acceder a docs.openclaw.ai (error SSL)">
    Algunas conexiones de Comcast/Xfinity bloquean incorrectamente `docs.openclaw.ai` mediante Xfinity
    Advanced Security. Desactívalo o añade `docs.openclaw.ai` a la lista de permitidos, y vuelve a intentarlo.
    Ayúdanos a desbloquearlo informándolo aquí: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Si aun así no puedes acceder al sitio, la documentación está reflejada en GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Diferencia entre estable y beta">
    **Stable** y **beta** son **dist-tags de npm**, no líneas de código separadas:

    - `latest` = estable
    - `beta` = compilación temprana para pruebas

    Normalmente, una versión estable llega primero a **beta**, y luego un paso explícito
    de promoción mueve esa misma versión a `latest`. Los mantenedores también pueden
    publicar directamente en `latest` cuando sea necesario. Por eso beta y estable pueden
    apuntar a la **misma versión** después de la promoción.

    Consulta qué cambió:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Para los one-liners de instalación y la diferencia entre beta y dev, consulta el acordeón de abajo.

  </Accordion>

  <Accordion title="¿Cómo instalo la versión beta y cuál es la diferencia entre beta y dev?">
    **Beta** es el dist-tag de npm `beta` (puede coincidir con `latest` después de la promoción).
    **Dev** es la cabecera móvil de `main` (git); cuando se publica, usa el dist-tag de npm `dev`.

    One-liners (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Instalador de Windows (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Más detalle: [Canales de desarrollo](/es/install/development-channels) y [Flags del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="¿Cómo pruebo lo más reciente?">
    Tienes dos opciones:

    1. **Canal dev (checkout de git):**

    ```bash
    openclaw update --channel dev
    ```

    Esto cambia a la rama `main` y actualiza desde el código fuente.

    2. **Instalación modificable (desde el sitio del instalador):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Eso te da un repositorio local que puedes editar y luego actualizar mediante git.

    Si prefieres un clon limpio manualmente, usa:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Documentación: [Update](/cli/update), [Canales de desarrollo](/es/install/development-channels),
    [Instalación](/es/install).

  </Accordion>

  <Accordion title="¿Cuánto suelen tardar la instalación y el onboarding?">
    Guía aproximada:

    - **Instalación:** 2-5 minutos
    - **Onboarding:** 5-15 minutos según cuántos canales/modelos configures

    Si se cuelga, usa [¿Instalador atascado?](#inicio-rapido-y-configuracion-de-primera-ejecucion)
    y el bucle rápido de depuración en [Estoy atascado](#inicio-rapido-y-configuracion-de-primera-ejecucion).

  </Accordion>

  <Accordion title="¿Instalador atascado? ¿Cómo obtengo más información?">
    Vuelve a ejecutar el instalador con **salida detallada**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Instalación beta con detalle:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Para una instalación modificable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Equivalente en Windows (PowerShell):

    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Más opciones: [Flags del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="La instalación en Windows dice git not found o openclaw not recognized">
    Dos problemas comunes en Windows:

    **1) error npm spawn git / git not found**

    - Instala **Git for Windows** y asegúrate de que `git` esté en tu PATH.
    - Cierra y vuelve a abrir PowerShell, luego vuelve a ejecutar el instalador.

    **2) openclaw no se reconoce después de la instalación**

    - Tu carpeta global bin de npm no está en PATH.
    - Comprueba la ruta:

      ```powershell
      npm config get prefix
      ```

    - Añade ese directorio a tu PATH de usuario (en Windows no hace falta el sufijo `\bin`; en la mayoría de los sistemas es `%AppData%\npm`).
    - Cierra y vuelve a abrir PowerShell después de actualizar PATH.

    Si quieres la configuración más fluida en Windows, usa **WSL2** en lugar de Windows nativo.
    Documentación: [Windows](/es/platforms/windows).

  </Accordion>

  <Accordion title="La salida de exec en Windows muestra texto chino corrupto. ¿Qué debo hacer?">
    Esto suele ser un desajuste del code page de la consola en shells nativos de Windows.

    Síntomas:

    - La salida de `system.run`/`exec` muestra caracteres chinos corruptos
    - El mismo comando se ve bien en otro perfil de terminal

    Solución rápida en PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Luego reinicia la Gateway y vuelve a intentar el comando:

    ```powershell
    openclaw gateway restart
    ```

    Si sigues reproduciendo esto en la última versión de OpenClaw, haz seguimiento/informa en:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="La documentación no respondió mi pregunta. ¿Cómo obtengo una mejor respuesta?">
    Usa la **instalación modificable (git)** para tener el código fuente completo y la documentación localmente, y luego pregunta
    a tu bot (o a Claude/Codex) _desde esa carpeta_ para que pueda leer el repositorio y responder con precisión.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Más detalle: [Instalación](/es/install) y [Flags del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en Linux?">
    Respuesta corta: sigue la guía de Linux y luego ejecuta onboarding.

    - Ruta rápida de Linux + instalación del servicio: [Linux](/es/platforms/linux).
    - Guía completa: [Primeros pasos](/es/start/getting-started).
    - Instalador + actualizaciones: [Instalación y actualizaciones](/es/install/updating).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en un VPS?">
    Cualquier VPS Linux funciona. Instala en el servidor y luego usa SSH/Tailscale para acceder a la Gateway.

    Guías: [exe.dev](/es/install/exe-dev), [Hetzner](/es/install/hetzner), [Fly.io](/es/install/fly).
    Acceso remoto: [Gateway remota](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde están las guías de instalación en la nube/VPS?">
    Mantenemos un **hub de alojamiento** con los proveedores habituales. Elige uno y sigue la guía:

    - [Alojamiento VPS](/es/vps) (todos los proveedores en un solo lugar)
    - [Fly.io](/es/install/fly)
    - [Hetzner](/es/install/hetzner)
    - [exe.dev](/es/install/exe-dev)

    Cómo funciona en la nube: la **Gateway se ejecuta en el servidor**, y accedes a ella
    desde tu portátil/teléfono mediante Control UI (o Tailscale/SSH). Tu estado + workspace
    viven en el servidor, así que trata al host como la fuente de verdad y haz copias de seguridad.

    Puedes emparejar **nodos** (Mac/iOS/Android/headless) con esa Gateway en la nube para acceder a
    pantalla/cámara/canvas locales o ejecutar comandos en tu portátil mientras mantienes la
    Gateway en la nube.

    Hub: [Plataformas](/es/platforms). Acceso remoto: [Gateway remota](/es/gateway/remote).
    Nodos: [Nodos](/es/nodes), [CLI de nodos](/cli/nodes).

  </Accordion>

  <Accordion title="¿Puedo pedirle a OpenClaw que se actualice a sí mismo?">
    Respuesta corta: **posible, no recomendado**. El flujo de actualización puede reiniciar la
    Gateway (lo que corta la sesión activa), puede necesitar un checkout limpio de git y
    puede pedir confirmación. Más seguro: ejecutar las actualizaciones desde un shell como operador.

    Usa la CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Si debes automatizar desde un agente:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Documentación: [Update](/cli/update), [Actualización](/es/install/updating).

  </Accordion>

  <Accordion title="¿Qué hace realmente onboarding?">
    `openclaw onboard` es la ruta de configuración recomendada. En **modo local** te guía por:

    - **Configuración de modelo/autenticación** (OAuth del proveedor, claves API, token de configuración de Anthropic, además de opciones de modelo local como LM Studio)
    - Ubicación del **workspace** + archivos de arranque
    - **Configuración de la Gateway** (bind/puerto/autenticación/tailscale)
    - **Canales** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, más plugins de canal incluidos como QQ Bot)
    - **Instalación del daemon** (LaunchAgent en macOS; unidad systemd de usuario en Linux/WSL2)
    - **Comprobaciones de salud** y selección de **Skills**

    También avisa si tu modelo configurado es desconocido o le falta autenticación.

  </Accordion>

  <Accordion title="¿Necesito una suscripción a Claude u OpenAI para ejecutar esto?">
    No. Puedes ejecutar OpenClaw con **claves API** (Anthropic/OpenAI/otros) o con
    **modelos solo locales** para que tus datos permanezcan en tu dispositivo. Las suscripciones (Claude
    Pro/Max u OpenAI Codex) son formas opcionales de autenticar esos proveedores.

    Para Anthropic en OpenClaw, la división práctica es:

    - **Clave API de Anthropic**: facturación normal de la API de Anthropic
    - **Autenticación de Claude CLI / suscripción Claude en OpenClaw**: el personal de Anthropic
      nos dijo que este uso vuelve a estar permitido, y OpenClaw está tratando el uso de `claude -p`
      como autorizado para esta integración salvo que Anthropic publique una política nueva

    Para hosts de gateway de larga duración, las claves API de Anthropic siguen siendo la
    configuración más predecible. OpenAI Codex OAuth está soportado explícitamente para herramientas externas
    como OpenClaw.

    OpenClaw también soporta otras opciones alojadas de estilo suscripción, incluidas
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan** y
    **Z.AI / GLM Coding Plan**.

    Documentación: [Anthropic](/es/providers/anthropic), [OpenAI](/es/providers/openai),
    [Qwen Cloud](/es/providers/qwen),
    [MiniMax](/es/providers/minimax), [Modelos GLM](/es/providers/glm),
    [Modelos locales](/es/gateway/local-models), [Modelos](/es/concepts/models).

  </Accordion>

  <Accordion title="¿Puedo usar la suscripción Claude Max sin una clave API?">
    Sí.

    El personal de Anthropic nos dijo que el uso de Claude CLI al estilo OpenClaw vuelve a estar permitido, así que
    OpenClaw trata la autenticación por suscripción Claude y el uso de `claude -p` como autorizados
    para esta integración salvo que Anthropic publique una nueva política. Si quieres
    la configuración del lado del servidor más predecible, usa una clave API de Anthropic en su lugar.

  </Accordion>

  <Accordion title="¿Soportan autenticación por suscripción Claude (Claude Pro o Max)?">
    Sí.

    El personal de Anthropic nos dijo que este uso vuelve a estar permitido, así que OpenClaw trata
    la reutilización de Claude CLI y el uso de `claude -p` como autorizados para esta integración
    salvo que Anthropic publique una nueva política.

    El token de configuración de Anthropic sigue disponible como ruta de token soportada por OpenClaw, pero OpenClaw ahora prefiere la reutilización de Claude CLI y `claude -p` cuando están disponibles.
    Para cargas de trabajo de producción o multiusuario, la autenticación con clave API de Anthropic sigue siendo la
    opción más segura y predecible. Si quieres otras opciones alojadas de estilo suscripción
    en OpenClaw, consulta [OpenAI](/es/providers/openai), [Qwen / Model
    Cloud](/es/providers/qwen), [MiniMax](/es/providers/minimax) y [Modelos
    GLM](/es/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="¿Por qué veo HTTP 429 rate_limit_error de Anthropic?">
Eso significa que tu **cuota/límite de tasa de Anthropic** está agotado para la ventana actual. Si
usas **Claude CLI**, espera a que se reinicie la ventana o mejora tu plan. Si
usas una **clave API de Anthropic**, revisa la consola de Anthropic
para ver uso/facturación y aumenta los límites según sea necesario.

    Si el mensaje es específicamente:
    `Extra usage is required for long context requests`, la solicitud está intentando usar
    la beta de contexto 1M de Anthropic (`context1m: true`). Eso solo funciona cuando tu
    credencial es apta para facturación de contexto largo (facturación con clave API o la
    ruta de inicio de sesión Claude de OpenClaw con Extra Usage habilitado).

    Consejo: configura un **modelo alternativo** para que OpenClaw pueda seguir respondiendo mientras un proveedor está limitado por tasa.
    Consulta [Models](/cli/models), [OAuth](/es/concepts/oauth) y
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/es/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="¿Está soportado AWS Bedrock?">
    Sí. OpenClaw tiene un proveedor incluido de **Amazon Bedrock (Converse)**. Con marcadores de entorno de AWS presentes, OpenClaw puede descubrir automáticamente el catálogo Bedrock de streaming/texto y fusionarlo como un proveedor implícito `amazon-bedrock`; de lo contrario, puedes habilitar explícitamente `plugins.entries.amazon-bedrock.config.discovery.enabled` o añadir una entrada manual de proveedor. Consulta [Amazon Bedrock](/es/providers/bedrock) y [Proveedores de modelos](/es/providers/models). Si prefieres un flujo de claves administrado, un proxy compatible con OpenAI delante de Bedrock sigue siendo una opción válida.
  </Accordion>

  <Accordion title="¿Cómo funciona la autenticación de Codex?">
    OpenClaw soporta **OpenAI Code (Codex)** mediante OAuth (inicio de sesión de ChatGPT). Onboarding puede ejecutar el flujo OAuth y configurará el modelo predeterminado en `openai-codex/gpt-5.4` cuando corresponda. Consulta [Proveedores de modelos](/es/concepts/model-providers) y [Onboarding (CLI)](/es/start/wizard).
  </Accordion>

  <Accordion title="¿Por qué ChatGPT GPT-5.4 no desbloquea openai/gpt-5.4 en OpenClaw?">
    OpenClaw trata las dos rutas por separado:

    - `openai-codex/gpt-5.4` = OAuth de ChatGPT/Codex
    - `openai/gpt-5.4` = API directa de OpenAI Platform

    En OpenClaw, el inicio de sesión de ChatGPT/Codex está conectado a la ruta `openai-codex/*`,
    no a la ruta directa `openai/*`. Si quieres la ruta de API directa en
    OpenClaw, configura `OPENAI_API_KEY` (o la configuración equivalente del proveedor OpenAI).
    Si quieres el inicio de sesión de ChatGPT/Codex en OpenClaw, usa `openai-codex/*`.

  </Accordion>

  <Accordion title="¿Por qué los límites de Codex OAuth pueden diferir de ChatGPT web?">
    `openai-codex/*` usa la ruta OAuth de Codex, y sus ventanas de cuota utilizables están
    gestionadas por OpenAI y dependen del plan. En la práctica, esos límites pueden diferir de
    la experiencia del sitio/app de ChatGPT, aunque ambos estén vinculados a la misma cuenta.

    OpenClaw puede mostrar las ventanas de uso/cuota visibles del proveedor en
    `openclaw models status`, pero no inventa ni normaliza derechos de ChatGPT web
    como acceso directo a la API. Si quieres la ruta directa de facturación/límites de OpenAI Platform,
    usa `openai/*` con una clave API.

  </Accordion>

  <Accordion title="¿Soportan autenticación por suscripción OpenAI (Codex OAuth)?">
    Sí. OpenClaw soporta completamente **OAuth por suscripción de OpenAI Code (Codex)**.
    OpenAI permite explícitamente el uso de OAuth por suscripción en herramientas/flujos externos
    como OpenClaw. Onboarding puede ejecutar el flujo OAuth por ti.

    Consulta [OAuth](/es/concepts/oauth), [Proveedores de modelos](/es/concepts/model-providers) y [Onboarding (CLI)](/es/start/wizard).

  </Accordion>

  <Accordion title="¿Cómo configuro Gemini CLI OAuth?">
    Gemini CLI usa un **flujo de autenticación de plugin**, no un client id ni un secret en `openclaw.json`.

    Pasos:

    1. Instala Gemini CLI localmente para que `gemini` esté en `PATH`
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Habilita el plugin: `openclaw plugins enable google`
    3. Inicia sesión: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Modelo predeterminado tras iniciar sesión: `google-gemini-cli/gemini-3-flash-preview`
    5. Si las solicitudes fallan, configura `GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host de la gateway

    Esto almacena tokens OAuth en perfiles de autenticación en el host de la gateway. Detalles: [Proveedores de modelos](/es/concepts/model-providers).

  </Accordion>

  <Accordion title="¿Un modelo local sirve para chats casuales?">
    Normalmente no. OpenClaw necesita contexto amplio + seguridad sólida; las tarjetas pequeñas truncan y filtran. Si debes hacerlo, ejecuta la compilación de modelo **más grande** que puedas localmente (LM Studio) y consulta [/gateway/local-models](/es/gateway/local-models). Los modelos más pequeños/cuantizados aumentan el riesgo de inyección de prompts; consulta [Security](/es/gateway/security).
  </Accordion>

  <Accordion title="¿Cómo mantengo el tráfico de modelos alojados en una región concreta?">
    Elige endpoints fijados por región. OpenRouter expone opciones alojadas en EE. UU. para MiniMax, Kimi y GLM; elige la variante alojada en EE. UU. para mantener los datos dentro de la región. Aun así puedes listar Anthropic/OpenAI junto a estos usando `models.mode: "merge"` para que las alternativas sigan disponibles mientras respetas el proveedor regional que selecciones.
  </Accordion>

  <Accordion title="¿Tengo que comprar un Mac Mini para instalar esto?">
    No. OpenClaw funciona en macOS o Linux (Windows mediante WSL2). Un Mac mini es opcional: algunas personas
    compran uno como host siempre activo, pero también sirve un VPS pequeño, un servidor doméstico o una máquina de la clase Raspberry Pi.

    Solo necesitas un Mac **para herramientas exclusivas de macOS**. Para iMessage, usa [BlueBubbles](/es/channels/bluebubbles) (recomendado): el servidor BlueBubbles se ejecuta en cualquier Mac, y la Gateway puede ejecutarse en Linux o en otro sitio. Si quieres otras herramientas exclusivas de macOS, ejecuta la Gateway en un Mac o empareja un nodo macOS.

    Documentación: [BlueBubbles](/es/channels/bluebubbles), [Nodos](/es/nodes), [Modo remoto en Mac](/es/platforms/mac/remote).

  </Accordion>

  <Accordion title="¿Necesito un Mac mini para el soporte de iMessage?">
    Necesitas **algún dispositivo macOS** con sesión iniciada en Messages. **No** tiene que ser un Mac mini:
    cualquier Mac sirve. **Usa [BlueBubbles](/es/channels/bluebubbles)** (recomendado) para iMessage: el servidor BlueBubbles se ejecuta en macOS, mientras que la Gateway puede ejecutarse en Linux o en otro sitio.

    Configuraciones comunes:

    - Ejecuta la Gateway en Linux/VPS, y ejecuta el servidor BlueBubbles en cualquier Mac con sesión iniciada en Messages.
    - Ejecuta todo en el Mac si quieres la configuración de una sola máquina más simple.

    Documentación: [BlueBubbles](/es/channels/bluebubbles), [Nodos](/es/nodes),
    [Modo remoto en Mac](/es/platforms/mac/remote).

  </Accordion>

  <Accordion title="Si compro un Mac mini para ejecutar OpenClaw, ¿puedo conectarlo a mi MacBook Pro?">
    Sí. El **Mac mini puede ejecutar la Gateway**, y tu MacBook Pro puede conectarse como
    **nodo** (dispositivo complementario). Los nodos no ejecutan la Gateway; proporcionan capacidades
    extra como pantalla/cámara/canvas y `system.run` en ese dispositivo.

    Patrón común:

    - Gateway en el Mac mini (siempre activa).
    - El MacBook Pro ejecuta la app macOS o un host de nodo y se empareja con la Gateway.
    - Usa `openclaw nodes status` / `openclaw nodes list` para verlo.

    Documentación: [Nodos](/es/nodes), [CLI de nodos](/cli/nodes).

  </Accordion>

  <Accordion title="¿Puedo usar Bun?">
    Bun **no se recomienda**. Vemos errores en tiempo de ejecución, especialmente con WhatsApp y Telegram.
    Usa **Node** para gateways estables.

    Si aun así quieres experimentar con Bun, hazlo en una gateway no productiva
    sin WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: ¿qué va en allowFrom?">
    `channels.telegram.allowFrom` es **el ID de usuario de Telegram del remitente humano** (numérico). No es el nombre de usuario del bot.

    Onboarding acepta una entrada `@username` y la resuelve a un ID numérico, pero la autorización de OpenClaw usa solo IDs numéricos.

    Más seguro (sin bot de terceros):

    - Envía un DM a tu bot, luego ejecuta `openclaw logs --follow` y lee `from.id`.

    API oficial de Bot:

    - Envía un DM a tu bot y luego llama a `https://api.telegram.org/bot<bot_token>/getUpdates` y lee `message.from.id`.

    Terceros (menos privado):

    - Envía un DM a `@userinfobot` o `@getidsbot`.

    Consulta [/channels/telegram](/es/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="¿Pueden varias personas usar un número de WhatsApp con distintas instancias de OpenClaw?">
    Sí, mediante **enrutamiento multiagente**. Vincula el **DM** de WhatsApp de cada remitente (peer `kind: "direct"`, remitente E.164 como `+15551234567`) a un `agentId` diferente, para que cada persona tenga su propio workspace y almacén de sesiones. Las respuestas seguirán saliendo de la **misma cuenta de WhatsApp**, y el control de acceso por DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) es global por cuenta de WhatsApp. Consulta [Enrutamiento multiagente](/es/concepts/multi-agent) y [WhatsApp](/es/channels/whatsapp).
  </Accordion>

  <Accordion title='¿Puedo ejecutar un agente de "chat rápido" y otro de "Opus para programación"?'>
    Sí. Usa enrutamiento multiagente: da a cada agente su propio modelo predeterminado y luego vincula las rutas entrantes (cuenta del proveedor o peers específicos) a cada agente. Hay una configuración de ejemplo en [Enrutamiento multiagente](/es/concepts/multi-agent). Consulta también [Modelos](/es/concepts/models) y [Configuración](/es/gateway/configuration).
  </Accordion>

  <Accordion title="¿Homebrew funciona en Linux?">
    Sí. Homebrew soporta Linux (Linuxbrew). Configuración rápida:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Si ejecutas OpenClaw mediante systemd, asegúrate de que el PATH del servicio incluya `/home/linuxbrew/.linuxbrew/bin` (o tu prefijo de brew) para que las herramientas instaladas con `brew` se resuelvan en shells no interactivos.
    Las compilaciones recientes también anteponen directorios bin comunes de usuario en servicios systemd de Linux (por ejemplo `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) y respetan `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` y `FNM_DIR` cuando están definidos.

  </Accordion>

  <Accordion title="Diferencia entre la instalación modificable con git y npm install">
    - **Instalación modificable (git):** checkout completo del código fuente, editable, ideal para colaboradores.
      Ejecutas las compilaciones localmente y puedes parchear código/documentación.
    - **npm install:** instalación global de la CLI, sin repositorio, ideal para “simplemente ejecutarlo”.
      Las actualizaciones vienen de dist-tags de npm.

    Documentación: [Primeros pasos](/es/start/getting-started), [Actualización](/es/install/updating).

  </Accordion>

  <Accordion title="¿Puedo cambiar entre instalaciones npm y git más adelante?">
    Sí. Instala la otra modalidad y luego ejecuta Doctor para que el servicio gateway apunte al nuevo entrypoint.
    Esto **no elimina tus datos**: solo cambia la instalación de código de OpenClaw. Tu estado
    (`~/.openclaw`) y tu workspace (`~/.openclaw/workspace`) permanecen intactos.

    De npm a git:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    De git a npm:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor detecta una discrepancia entre el entrypoint del servicio gateway y ofrece reescribir la configuración del servicio para que coincida con la instalación actual (usa `--repair` en automatización).

    Consejos de copia de seguridad: consulta [Estrategia de copia de seguridad](#donde-vive-cada-cosa-en-disco).

  </Accordion>

  <Accordion title="¿Debo ejecutar la Gateway en mi portátil o en un VPS?">
    Respuesta corta: **si quieres fiabilidad 24/7, usa un VPS**. Si quieres la
    menor fricción y te conformas con suspensión/reinicios, ejecútala localmente.

    **Portátil (Gateway local)**

    - **Ventajas:** sin coste de servidor, acceso directo a archivos locales, ventana del navegador visible.
    - **Desventajas:** suspensión/caídas de red = desconexiones, actualizaciones/reinicios del SO interrumpen, debe permanecer despierto.

    **VPS / nube**

    - **Ventajas:** siempre activo, red estable, sin problemas de suspensión del portátil, más fácil de mantener en ejecución.
    - **Desventajas:** a menudo se ejecuta sin interfaz (usa capturas), acceso solo remoto a archivos, debes usar SSH para actualizar.

    **Nota específica de OpenClaw:** WhatsApp/Telegram/Slack/Mattermost/Discord funcionan bien desde un VPS. La única diferencia real es **navegador headless** frente a ventana visible. Consulta [Browser](/es/tools/browser).

    **Recomendación por defecto:** VPS si ya sufriste desconexiones de la gateway. Local es ideal cuando estás usando activamente el Mac y quieres acceso a archivos locales o automatización de UI con un navegador visible.

  </Accordion>

  <Accordion title="¿Qué tan importante es ejecutar OpenClaw en una máquina dedicada?">
    No es obligatorio, pero **se recomienda para fiabilidad y aislamiento**.

    - **Host dedicado (VPS/Mac mini/Pi):** siempre activo, menos interrupciones por suspensión/reinicio, permisos más limpios, más fácil de mantener en ejecución.
    - **Portátil/escritorio compartido:** totalmente válido para pruebas y uso activo, pero espera pausas cuando la máquina suspenda o se actualice.

    Si quieres lo mejor de ambos mundos, mantén la Gateway en un host dedicado y empareja tu portátil como **nodo** para herramientas locales de pantalla/cámara/exec. Consulta [Nodos](/es/nodes).
    Para orientación de seguridad, lee [Security](/es/gateway/security).

  </Accordion>

  <Accordion title="¿Cuáles son los requisitos mínimos de VPS y el SO recomendado?">
    OpenClaw es ligero. Para una Gateway básica + un canal de chat:

    - **Mínimo absoluto:** 1 vCPU, 1GB de RAM, ~500MB de disco.
    - **Recomendado:** 1-2 vCPU, 2GB de RAM o más para margen (registros, medios, múltiples canales). Las herramientas de nodo y la automatización del navegador pueden consumir muchos recursos.

    SO: usa **Ubuntu LTS** (o cualquier Debian/Ubuntu moderno). La ruta de instalación en Linux está mejor probada allí.

    Documentación: [Linux](/es/platforms/linux), [Alojamiento VPS](/es/vps).

  </Accordion>

  <Accordion title="¿Puedo ejecutar OpenClaw en una VM y cuáles son los requisitos?">
    Sí. Trata una VM igual que un VPS: debe estar siempre activa, accesible y tener suficiente
    RAM para la Gateway y cualquier canal que habilites.

    Orientación básica:

    - **Mínimo absoluto:** 1 vCPU, 1GB de RAM.
    - **Recomendado:** 2GB de RAM o más si ejecutas múltiples canales, automatización del navegador o herramientas multimedia.
    - **SO:** Ubuntu LTS u otro Debian/Ubuntu moderno.

    Si estás en Windows, **WSL2 es la configuración tipo VM más fácil** y la que tiene mejor
    compatibilidad de herramientas. Consulta [Windows](/es/platforms/windows), [Alojamiento VPS](/es/vps).
    Si ejecutas macOS en una VM, consulta [macOS VM](/es/install/macos-vm).

  </Accordion>
</AccordionGroup>

## ¿Qué es OpenClaw?

<AccordionGroup>
  <Accordion title="¿Qué es OpenClaw, en un párrafo?">
    OpenClaw es un asistente de IA personal que ejecutas en tus propios dispositivos. Responde en las superficies de mensajería que ya usas (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat y plugins de canal incluidos como QQ Bot) y también puede hacer voz + un Canvas en vivo en plataformas compatibles. La **Gateway** es el plano de control siempre activo; el asistente es el producto.
  </Accordion>

  <Accordion title="Propuesta de valor">
    OpenClaw no es “solo un envoltorio de Claude”. Es un **plano de control local-first** que te permite ejecutar un
    asistente capaz en **tu propio hardware**, accesible desde las aplicaciones de chat que ya usas, con
    sesiones con estado, memoria y herramientas, sin ceder el control de tus flujos de trabajo a un
    SaaS alojado.

    Puntos destacados:

    - **Tus dispositivos, tus datos:** ejecuta la Gateway donde quieras (Mac, Linux, VPS) y mantén el
      workspace + historial de sesiones en local.
    - **Canales reales, no un sandbox web:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc.,
      además de voz móvil y Canvas en plataformas compatibles.
    - **Independiente del modelo:** usa Anthropic, OpenAI, MiniMax, OpenRouter, etc., con enrutamiento
      por agente y conmutación por error.
    - **Opción solo local:** ejecuta modelos locales para que **todos los datos puedan permanecer en tu dispositivo** si quieres.
    - **Enrutamiento multiagente:** agentes separados por canal, cuenta o tarea, cada uno con su propio
      workspace y valores predeterminados.
    - **Código abierto y modificable:** inspecciona, amplía y autoaloja sin dependencia de proveedor.

    Documentación: [Gateway](/es/gateway), [Canales](/es/channels), [Multiagente](/es/concepts/multi-agent),
    [Memoria](/es/concepts/memory).

  </Accordion>

  <Accordion title="Acabo de configurarlo. ¿Qué debería hacer primero?">
    Buenos primeros proyectos:

    - Crear un sitio web (WordPress, Shopify o un sitio estático simple).
    - Prototipar una app móvil (esquema, pantallas, plan de API).
    - Organizar archivos y carpetas (limpieza, nombres, etiquetado).
    - Conectar Gmail y automatizar resúmenes o seguimientos.

    Puede manejar tareas grandes, pero funciona mejor cuando las divides en fases y
    usas subagentes para trabajo en paralelo.

  </Accordion>

  <Accordion title="¿Cuáles son los cinco casos de uso cotidianos principales de OpenClaw?">
    Las victorias cotidianas suelen ser:

    - **Informes personales:** resúmenes de tu bandeja de entrada, calendario y noticias que te importan.
    - **Investigación y borradores:** investigación rápida, resúmenes y primeros borradores para correos o documentos.
    - **Recordatorios y seguimientos:** avisos y listas de verificación guiados por cron o heartbeat.
    - **Automatización del navegador:** rellenar formularios, recopilar datos y repetir tareas web.
    - **Coordinación entre dispositivos:** envía una tarea desde tu teléfono, deja que la Gateway la ejecute en un servidor y recibe el resultado de vuelta en el chat.

  </Accordion>

  <Accordion title="¿Puede OpenClaw ayudar con lead gen, outreach, anuncios y blogs para un SaaS?">
    Sí para **investigación, cualificación y redacción**. Puede escanear sitios, crear listas cortas,
    resumir prospectos y redactar borradores de outreach o copy para anuncios.

    Para **outreach o campañas de anuncios**, mantén a una persona en el circuito. Evita el spam, sigue las leyes locales y
    las políticas de la plataforma, y revisa todo antes de enviarlo. El patrón más seguro es dejar que
    OpenClaw redacte y que tú apruebes.

    Documentación: [Security](/es/gateway/security).

  </Accordion>

  <Accordion title="¿Qué ventajas tiene frente a Claude Code para desarrollo web?">
    OpenClaw es un **asistente personal** y una capa de coordinación, no un sustituto del IDE. Usa
    Claude Code o Codex para el bucle de programación directa más rápido dentro de un repositorio. Usa OpenClaw cuando
    quieras memoria duradera, acceso entre dispositivos y orquestación de herramientas.

    Ventajas:

    - **Memoria + workspace persistentes** entre sesiones
    - **Acceso multiplataforma** (WhatsApp, Telegram, TUI, WebChat)
    - **Orquestación de herramientas** (navegador, archivos, programación, hooks)
    - **Gateway siempre activa** (ejecútala en un VPS, interactúa desde cualquier lugar)
    - **Nodos** para navegador/pantalla/cámara/exec locales

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills y automatización

<AccordionGroup>
  <Accordion title="¿Cómo personalizo Skills sin ensuciar el repositorio?">
    Usa sobrescrituras gestionadas en lugar de editar la copia del repositorio. Pon tus cambios en `~/.openclaw/skills/<name>/SKILL.md` (o añade una carpeta mediante `skills.load.extraDirs` en `~/.openclaw/openclaw.json`). La precedencia es `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → incluidas → `skills.load.extraDirs`, así que las sobrescrituras gestionadas siguen prevaleciendo sobre las Skills incluidas sin tocar git. Si necesitas la Skill instalada globalmente pero visible solo para algunos agentes, mantén la copia compartida en `~/.openclaw/skills` y controla la visibilidad con `agents.defaults.skills` y `agents.list[].skills`. Solo las ediciones dignas de upstream deberían vivir en el repositorio y salir como PRs.
  </Accordion>

  <Accordion title="¿Puedo cargar Skills desde una carpeta personalizada?">
    Sí. Añade directorios extra mediante `skills.load.extraDirs` en `~/.openclaw/openclaw.json` (precedencia más baja). La precedencia predeterminada es `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → incluidas → `skills.load.extraDirs`. `clawhub` instala en `./skills` por defecto, lo que OpenClaw trata como `<workspace>/skills` en la siguiente sesión. Si la Skill solo debe ser visible para ciertos agentes, combínalo con `agents.defaults.skills` o `agents.list[].skills`.
  </Accordion>

  <Accordion title="¿Cómo puedo usar distintos modelos para distintas tareas?">
    Hoy los patrones soportados son:

    - **Trabajos cron**: los trabajos aislados pueden establecer un override `model` por trabajo.
    - **Subagentes**: enruta tareas a agentes separados con distintos modelos predeterminados.
    - **Cambio bajo demanda**: usa `/model` para cambiar el modelo de la sesión actual en cualquier momento.

    Consulta [Trabajos cron](/es/automation/cron-jobs), [Enrutamiento multiagente](/es/concepts/multi-agent) y [Comandos slash](/es/tools/slash-commands).

  </Accordion>

  <Accordion title="El bot se congela mientras hace trabajo pesado. ¿Cómo descargo eso?">
    Usa **subagentes** para tareas largas o paralelas. Los subagentes se ejecutan en su propia sesión,
    devuelven un resumen y mantienen tu chat principal con capacidad de respuesta.

    Pide a tu bot que “cree un subagente para esta tarea” o usa `/subagents`.
    Usa `/status` en el chat para ver qué está haciendo la Gateway ahora mismo (y si está ocupada).

    Consejo sobre tokens: las tareas largas y los subagentes consumen tokens. Si el coste te preocupa, configura un
    modelo más barato para subagentes mediante `agents.defaults.subagents.model`.

    Documentación: [Subagentes](/es/tools/subagents), [Tareas en segundo plano](/es/automation/tasks).

  </Accordion>

  <Accordion title="¿Cómo funcionan las sesiones de subagente vinculadas a hilos en Discord?">
    Usa vinculaciones de hilos. Puedes vincular un hilo de Discord a un subagente o destino de sesión para que los mensajes de seguimiento en ese hilo permanezcan en esa sesión vinculada.

    Flujo básico:

    - Crea con `sessions_spawn` usando `thread: true` (y opcionalmente `mode: "session"` para seguimiento persistente).
    - O vincula manualmente con `/focus <target>`.
    - Usa `/agents` para inspeccionar el estado de la vinculación.
    - Usa `/session idle <duration|off>` y `/session max-age <duration|off>` para controlar el desenfoque automático.
    - Usa `/unfocus` para separar el hilo.

    Configuración necesaria:

    - Valores predeterminados globales: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Overrides de Discord: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Vinculación automática al crear: establece `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Documentación: [Subagentes](/es/tools/subagents), [Discord](/es/channels/discord), [Referencia de configuración](/es/gateway/configuration-reference), [Comandos slash](/es/tools/slash-commands).

  </Accordion>

  <Accordion title="Un subagente terminó, pero la actualización de finalización fue al lugar equivocado o nunca se publicó. ¿Qué debo revisar?">
    Revisa primero la ruta del solicitante resuelta:

    - La entrega del subagente en modo de finalización prefiere cualquier hilo vinculado o ruta de conversación cuando existe.
    - Si el origen de finalización solo incluye un canal, OpenClaw usa como alternativa la ruta almacenada de la sesión solicitante (`lastChannel` / `lastTo` / `lastAccountId`) para que la entrega directa aún pueda funcionar.
    - Si no existe ni una ruta vinculada ni una ruta almacenada utilizable, la entrega directa puede fallar y el resultado vuelve a la entrega en cola de la sesión en lugar de publicarse inmediatamente en el chat.
    - Los destinos inválidos o desactualizados aún pueden forzar la vuelta a la cola o el fallo final de entrega.
    - Si la última respuesta visible del asistente hijo es exactamente el token silencioso `NO_REPLY` / `no_reply`, o exactamente `ANNOUNCE_SKIP`, OpenClaw suprime intencionadamente el anuncio en lugar de publicar progreso anterior obsoleto.
    - Si el hijo agotó el tiempo después de solo llamadas a herramientas, el anuncio puede colapsar eso en un breve resumen de progreso parcial en lugar de reproducir la salida bruta de herramientas.

    Depuración:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Subagentes](/es/tools/subagents), [Tareas en segundo plano](/es/automation/tasks), [Herramienta de sesión](/es/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron o los recordatorios no se disparan. ¿Qué debo revisar?">
    Cron se ejecuta dentro del proceso Gateway. Si la Gateway no está ejecutándose de forma continua,
    los trabajos programados no se ejecutarán.

    Lista de comprobación:

    - Confirma que cron está habilitado (`cron.enabled`) y que `OPENCLAW_SKIP_CRON` no está definido.
    - Comprueba que la Gateway se ejecuta 24/7 (sin suspensión/reinicios).
    - Verifica la configuración de zona horaria del trabajo (`--tz` frente a la zona horaria del host).

    Depuración:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentación: [Trabajos cron](/es/automation/cron-jobs), [Automatización y tareas](/es/automation).

  </Accordion>

  <Accordion title="Cron se disparó, pero no se envió nada al canal. ¿Por qué?">
    Revisa primero el modo de entrega:

    - `--no-deliver` / `delivery.mode: "none"` significa que no se espera ningún mensaje externo.
    - Un destino de anuncio faltante o inválido (`channel` / `to`) significa que el ejecutor omitió la entrega saliente.
    - Los fallos de autenticación del canal (`unauthorized`, `Forbidden`) significan que el ejecutor intentó entregar pero las credenciales lo bloquearon.
    - Un resultado aislado silencioso (`NO_REPLY` / `no_reply` solamente) se trata como intencionadamente no entregable, por lo que el ejecutor también suprime la entrega alternativa en cola.

    Para trabajos cron aislados, el ejecutor es responsable de la entrega final. Se espera
    que el agente devuelva un resumen en texto plano para que el ejecutor lo envíe. `--no-deliver` mantiene
    ese resultado interno; no permite que el agente envíe directamente con la
    herramienta de mensajes.

    Depuración:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Trabajos cron](/es/automation/cron-jobs), [Tareas en segundo plano](/es/automation/tasks).

  </Accordion>

  <Accordion title="¿Por qué una ejecución cron aislada cambió de modelo o reintentó una vez?">
    Suele ser la ruta de cambio de modelo en vivo, no una programación duplicada.

    Cron aislado puede persistir un traspaso de modelo en tiempo de ejecución y reintentar cuando la
    ejecución activa lanza `LiveSessionModelSwitchError`. El reintento conserva el
    proveedor/modelo cambiado, y si el cambio llevaba un nuevo override de perfil de autenticación, cron
    también lo persiste antes de reintentar.

    Reglas de selección relacionadas:

    - El override de modelo del hook de Gmail gana primero cuando aplica.
    - Luego el `model` por trabajo.
    - Luego cualquier override de modelo almacenado de la sesión cron.
    - Luego la selección normal de modelo predeterminado/agente.

    El bucle de reintento está acotado. Después del intento inicial más 2 reintentos por cambio,
    cron aborta en lugar de entrar en un bucle infinito.

    Depuración:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Trabajos cron](/es/automation/cron-jobs), [CLI de cron](/cli/cron).

  </Accordion>

  <Accordion title="¿Cómo instalo Skills en Linux?">
    Usa los comandos nativos `openclaw skills` o coloca Skills en tu workspace. La UI de Skills de macOS no está disponible en Linux.
    Explora Skills en [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    La instalación nativa `openclaw skills install` escribe en el directorio `skills/`
    del workspace activo. Instala la CLI separada `clawhub` solo si quieres publicar o
    sincronizar tus propias Skills. Para instalaciones compartidas entre agentes, coloca la Skill bajo
    `~/.openclaw/skills` y usa `agents.defaults.skills` o
    `agents.list[].skills` si quieres limitar qué agentes pueden verla.

  </Accordion>

  <Accordion title="¿Puede OpenClaw ejecutar tareas según un horario o continuamente en segundo plano?">
    Sí. Usa el programador de la Gateway:

    - **Trabajos cron** para tareas programadas o recurrentes (persisten tras reinicios).
    - **Heartbeat** para comprobaciones periódicas de la “sesión principal”.
    - **Trabajos aislados** para agentes autónomos que publican resúmenes o entregan a chats.

    Documentación: [Trabajos cron](/es/automation/cron-jobs), [Automatización y tareas](/es/automation),
    [Heartbeat](/es/gateway/heartbeat).

  </Accordion>

  <Accordion title="¿Puedo ejecutar Skills exclusivas de macOS desde Linux?">
    No directamente. Las Skills de macOS están restringidas por `metadata.openclaw.os` más los binarios requeridos, y las Skills solo aparecen en la indicación del sistema cuando son elegibles en el **host de la Gateway**. En Linux, las Skills solo de `darwin` (como `apple-notes`, `apple-reminders`, `things-mac`) no se cargarán salvo que sobrescribas la restricción.

    Tienes tres patrones compatibles:

    **Opción A - ejecutar la Gateway en un Mac (más simple).**
    Ejecuta la Gateway donde existan los binarios de macOS, y luego conéctate desde Linux en [modo remoto](#gateway-puertos-ya-en-ejecucion-y-modo-remoto) o por Tailscale. Las Skills se cargan normalmente porque el host de la Gateway es macOS.

    **Opción B - usar un nodo macOS (sin SSH).**
    Ejecuta la Gateway en Linux, empareja un nodo macOS (app de barra de menús) y establece **Node Run Commands** en “Always Ask” o “Always Allow” en el Mac. OpenClaw puede tratar las Skills exclusivas de macOS como elegibles cuando los binarios necesarios existen en el nodo. El agente ejecuta esas Skills mediante la herramienta `nodes`. Si eliges “Always Ask”, aprobar “Always Allow” en el aviso añade ese comando a la lista de permitidos.

    **Opción C - hacer proxy de binarios macOS por SSH (avanzado).**
    Mantén la Gateway en Linux, pero haz que los binarios CLI necesarios se resuelvan a wrappers SSH que se ejecuten en un Mac. Luego sobrescribe la Skill para permitir Linux y que siga siendo elegible.

    1. Crea un wrapper SSH para el binario (ejemplo: `memo` para Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Pon el wrapper en `PATH` en el host Linux (por ejemplo `~/bin/memo`).
    3. Sobrescribe los metadatos de la Skill (workspace o `~/.openclaw/skills`) para permitir Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Inicia una nueva sesión para que se actualice la instantánea de Skills.

  </Accordion>

  <Accordion title="¿Tienen integración con Notion o HeyGen?">
    No integrada hoy.

    Opciones:

    - **Skill / plugin personalizado:** mejor para un acceso fiable a la API (Notion/HeyGen tienen API).
    - **Automatización del navegador:** funciona sin código, pero es más lenta y frágil.

    Si quieres mantener el contexto por cliente (flujos de agencia), un patrón simple es:

    - Una página de Notion por cliente (contexto + preferencias + trabajo activo).
    - Pedir al agente que obtenga esa página al inicio de una sesión.

    Si quieres una integración nativa, abre una solicitud de función o crea una Skill
    orientada a esas API.

    Instalar Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Las instalaciones nativas aterrizan en el directorio `skills/` del workspace activo. Para Skills compartidas entre agentes, colócalas en `~/.openclaw/skills/<name>/SKILL.md`. Si solo algunos agentes deben ver una instalación compartida, configura `agents.defaults.skills` o `agents.list[].skills`. Algunas Skills esperan binarios instalados con Homebrew; en Linux eso significa Linuxbrew (consulta la entrada de preguntas frecuentes de Homebrew en Linux más arriba). Consulta [Skills](/es/tools/skills), [Configuración de Skills](/es/tools/skills-config) y [ClawHub](/es/tools/clawhub).

  </Accordion>

  <Accordion title="¿Cómo uso mi Chrome ya autenticado con OpenClaw?">
    Usa el perfil de navegador integrado `user`, que se conecta mediante Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Si quieres un nombre personalizado, crea un perfil MCP explícito:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Esta ruta es local al host. Si la Gateway se ejecuta en otro sitio, ejecuta un host de nodo en la máquina del navegador o usa CDP remoto.

    Límites actuales de `existing-session` / `user`:

    - las acciones se basan en `ref`, no en selectores CSS
    - las subidas requieren `ref` / `inputRef` y actualmente soportan un archivo cada vez
    - `responsebody`, exportación PDF, interceptación de descargas y acciones por lotes siguen necesitando un navegador gestionado o un perfil CDP sin procesar

  </Accordion>
</AccordionGroup>

## Sandbox y memoria

<AccordionGroup>
  <Accordion title="¿Hay una documentación dedicada al sandboxing?">
    Sí. Consulta [Sandboxing](/es/gateway/sandboxing). Para configuración específica de Docker (gateway completa en Docker o imágenes sandbox), consulta [Docker](/es/install/docker).
  </Accordion>

  <Accordion title="Docker se siente limitado. ¿Cómo habilito funciones completas?">
    La imagen predeterminada prioriza la seguridad y se ejecuta como el usuario `node`, así que no
    incluye paquetes del sistema, Homebrew ni navegadores incluidos. Para una configuración más completa:

    - Persiste `/home/node` con `OPENCLAW_HOME_VOLUME` para que las cachés sobrevivan.
    - Incorpora dependencias del sistema en la imagen con `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Instala navegadores Playwright mediante la CLI incluida:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Define `PLAYWRIGHT_BROWSERS_PATH` y asegúrate de que la ruta persista.

    Documentación: [Docker](/es/install/docker), [Browser](/es/tools/browser).

  </Accordion>

  <Accordion title="¿Puedo mantener los DM privados pero hacer públicos/con sandbox los grupos con un solo agente?">
    Sí, **si** tu tráfico privado son **DM** y tu tráfico público son **grupos**.

    Usa `agents.defaults.sandbox.mode: "non-main"` para que las sesiones de grupo/canal (claves no principales) se ejecuten en Docker, mientras que la sesión principal DM permanezca en el host. Luego restringe qué herramientas están disponibles en las sesiones con sandbox mediante `tools.sandbox.tools`.

    Guía de configuración + ejemplo: [Grupos: DM personales + grupos públicos](/es/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Referencia clave de configuración: [Configuración de Gateway](/es/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="¿Cómo vinculo una carpeta del host en el sandbox?">
    Define `agents.defaults.sandbox.docker.binds` como `["host:path:mode"]` (por ejemplo `"/home/user/src:/src:ro"`). Los binds globales + por agente se fusionan; los binds por agente se ignoran cuando `scope: "shared"`. Usa `:ro` para cualquier cosa sensible y recuerda que los binds eluden las barreras del sistema de archivos del sandbox.

    OpenClaw valida los orígenes de bind tanto contra la ruta normalizada como contra la ruta canónica resuelta a través del ancestro existente más profundo. Eso significa que las fugas por symlink en el directorio padre siguen fallando de forma segura incluso cuando el último segmento de la ruta aún no existe, y las comprobaciones de raíces permitidas siguen aplicándose tras resolver symlinks.

    Consulta [Sandboxing](/es/gateway/sandboxing#custom-bind-mounts) y [Sandbox vs Tool Policy vs Elevated](/es/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) para ejemplos y notas de seguridad.

  </Accordion>

  <Accordion title="¿Cómo funciona la memoria?">
    La memoria de OpenClaw son simplemente archivos Markdown en el workspace del agente:

    - Notas diarias en `memory/YYYY-MM-DD.md`
    - Notas curadas de largo plazo en `MEMORY.md` (solo sesiones principales/privadas)

    OpenClaw también ejecuta un **volcado silencioso de memoria antes de compactar** para recordarle al modelo
    que escriba notas duraderas antes de la autocompactación. Esto solo se ejecuta cuando el workspace
    se puede escribir (los sandboxes de solo lectura lo omiten). Consulta [Memoria](/es/concepts/memory).

  </Accordion>

  <Accordion title="La memoria sigue olvidando cosas. ¿Cómo hago que se mantengan?">
    Pide al bot que **escriba el dato en memoria**. Las notas de largo plazo pertenecen a `MEMORY.md`,
    el contexto de corto plazo va en `memory/YYYY-MM-DD.md`.

    Esta sigue siendo un área que estamos mejorando. Ayuda recordar al modelo que almacene recuerdos;
    sabrá qué hacer. Si sigue olvidando, verifica que la Gateway esté usando el mismo
    workspace en cada ejecución.

    Documentación: [Memoria](/es/concepts/memory), [Workspace del agente](/es/concepts/agent-workspace).

  </Accordion>

  <Accordion title="¿La memoria persiste para siempre? ¿Cuáles son los límites?">
    Los archivos de memoria viven en disco y persisten hasta que los eliminas. El límite es tu
    almacenamiento, no el modelo. El **contexto de sesión** sigue limitado por la ventana de contexto
    del modelo, así que las conversaciones largas pueden compactarse o truncarse. Por eso
    existe la búsqueda en memoria: recupera solo las partes relevantes al contexto.

    Documentación: [Memoria](/es/concepts/memory), [Contexto](/es/concepts/context).

  </Accordion>

  <Accordion title="¿La búsqueda semántica en memoria requiere una clave API de OpenAI?">
    Solo si usas **embeddings de OpenAI**. Codex OAuth cubre chat/completions y
    **no** concede acceso a embeddings, así que **iniciar sesión con Codex (OAuth o el
    login de Codex CLI)** no ayuda para la búsqueda semántica en memoria. Los embeddings de OpenAI
    siguen necesitando una clave API real (`OPENAI_API_KEY` o `models.providers.openai.apiKey`).

    Si no defines explícitamente un proveedor, OpenClaw selecciona automáticamente uno cuando
    puede resolver una clave API (perfiles de autenticación, `models.providers.*.apiKey` o variables de entorno).
    Prefiere OpenAI si puede resolver una clave de OpenAI, en caso contrario Gemini si puede resolver una clave de Gemini,
    luego Voyage, luego Mistral. Si no hay ninguna clave remota disponible, la búsqueda en
    memoria permanece deshabilitada hasta que la configures. Si tienes una ruta de modelo local
    configurada y presente, OpenClaw
    prefiere `local`. Ollama está soportado cuando estableces explícitamente
    `memorySearch.provider = "ollama"`.

    Si prefieres mantenerlo local, define `memorySearch.provider = "local"` (y opcionalmente
    `memorySearch.fallback = "none"`). Si quieres embeddings de Gemini, define
    `memorySearch.provider = "gemini"` y proporciona `GEMINI_API_KEY` (o
    `memorySearch.remote.apiKey`). Soportamos modelos de embeddings de **OpenAI, Gemini, Voyage, Mistral, Ollama o local**;
    consulta [Memoria](/es/concepts/memory) para los detalles de configuración.

  </Accordion>
</AccordionGroup>

## Dónde vive cada cosa en disco

<AccordionGroup>
  <Accordion title="¿Se guardan localmente todos los datos usados con OpenClaw?">
    No: **el estado de OpenClaw es local**, pero **los servicios externos siguen viendo lo que les envías**.

    - **Local por defecto:** sesiones, archivos de memoria, configuración y workspace viven en el host de la Gateway
      (`~/.openclaw` + tu directorio de workspace).
    - **Remoto por necesidad:** los mensajes que envías a proveedores de modelos (Anthropic/OpenAI/etc.) van a
      sus API, y las plataformas de chat (WhatsApp/Telegram/Slack/etc.) almacenan los datos de mensajes en
      sus servidores.
    - **Tú controlas la huella:** usar modelos locales mantiene los prompts en tu máquina, pero el
      tráfico de canales sigue pasando por los servidores del canal.

    Relacionado: [Workspace del agente](/es/concepts/agent-workspace), [Memoria](/es/concepts/memory).

  </Accordion>

  <Accordion title="¿Dónde almacena OpenClaw sus datos?">
    Todo vive bajo `$OPENCLAW_STATE_DIR` (predeterminado: `~/.openclaw`):

    | Path                                                            | Purpose                                                            |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Configuración principal (JSON5)                                    |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Importación OAuth heredada (copiada a perfiles de autenticación en el primer uso)       |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Perfiles de autenticación (OAuth, claves API y `keyRef`/`tokenRef` opcionales)  |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Carga útil opcional de secretos respaldada por archivo para proveedores SecretRef `file` |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | Archivo heredado de compatibilidad (entradas estáticas `api_key` depuradas)      |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | Estado del proveedor (p. ej. `whatsapp/<accountId>/creds.json`)            |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | Estado por agente (agentDir + sesiones)                              |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Historial y estado de conversaciones (por agente)                           |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Metadatos de sesión (por agente)                                       |

    Ruta heredada de agente único: `~/.openclaw/agent/*` (migrada por `openclaw doctor`).

    Tu **workspace** (`AGENTS.md`, archivos de memoria, Skills, etc.) es independiente y se configura mediante `agents.defaults.workspace` (predeterminado: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="¿Dónde deben vivir AGENTS.md / SOUL.md / USER.md / MEMORY.md?">
    Estos archivos viven en el **workspace del agente**, no en `~/.openclaw`.

    - **Workspace (por agente)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (o el respaldo heredado `memory.md` cuando `MEMORY.md` no existe),
      `memory/YYYY-MM-DD.md`, `HEARTBEAT.md` opcional.
    - **Directorio de estado (`~/.openclaw`)**: configuración, estado de canal/proveedor, perfiles de autenticación, sesiones, registros,
      y Skills compartidas (`~/.openclaw/skills`).

    El workspace predeterminado es `~/.openclaw/workspace`, configurable mediante:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Si el bot “olvida” después de un reinicio, confirma que la Gateway esté usando el mismo
    workspace en cada arranque (y recuerda: el modo remoto usa el workspace del **host de la gateway**,
    no el de tu portátil local).

    Consejo: si quieres un comportamiento o preferencia duradera, pide al bot que **lo escriba en
    AGENTS.md o MEMORY.md** en lugar de depender del historial del chat.

    Consulta [Workspace del agente](/es/concepts/agent-workspace) y [Memoria](/es/concepts/memory).

  </Accordion>

  <Accordion title="Estrategia de copia de seguridad recomendada">
    Pon tu **workspace del agente** en un repositorio git **privado** y haz copia de seguridad en algún
    lugar privado (por ejemplo GitHub privado). Esto captura memoria + archivos AGENTS/SOUL/USER
    y te permite restaurar la “mente” del asistente más adelante.

    **No** hagas commit de nada bajo `~/.openclaw` (credenciales, sesiones, tokens o cargas útiles de secretos cifradas).
    Si necesitas una restauración completa, haz copia de seguridad tanto del workspace como del directorio de estado
    por separado (consulta la pregunta de migración más arriba).

    Documentación: [Workspace del agente](/es/concepts/agent-workspace).

  </Accordion>

  <Accordion title="¿Cómo desinstalo completamente OpenClaw?">
    Consulta la guía dedicada: [Desinstalación](/es/install/uninstall).
  </Accordion>

  <Accordion title="¿Pueden los agentes trabajar fuera del workspace?">
    Sí. El workspace es el **cwd predeterminado** y ancla de memoria, no un sandbox rígido.
    Las rutas relativas se resuelven dentro del workspace, pero las rutas absolutas pueden acceder a otras
    ubicaciones del host a menos que el sandboxing esté habilitado. Si necesitas aislamiento, usa
    [`agents.defaults.sandbox`](/es/gateway/sandboxing) o configuraciones de sandbox por agente. Si
    quieres que un repositorio sea el directorio de trabajo predeterminado, apunta el
    `workspace` de ese agente a la raíz del repositorio. El repositorio de OpenClaw es solo código fuente; mantén el
    workspace separado salvo que quieras intencionadamente que el agente trabaje dentro de él.

    Ejemplo (repositorio como cwd predeterminado):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Modo remoto: ¿dónde está el almacén de sesiones?">
    El estado de sesión pertenece al **host de la gateway**. Si estás en modo remoto, el almacén de sesiones que importa está en la máquina remota, no en tu portátil local. Consulta [Gestión de sesiones](/es/concepts/session).
  </Accordion>
</AccordionGroup>

## Conceptos básicos de configuración

<AccordionGroup>
  <Accordion title="¿Qué formato tiene la configuración? ¿Dónde está?">
    OpenClaw lee una configuración opcional **JSON5** desde `$OPENCLAW_CONFIG_PATH` (predeterminado: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Si falta el archivo, usa valores predeterminados razonablemente seguros (incluido un workspace predeterminado de `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='Definí gateway.bind: "lan" (o "tailnet") y ahora no escucha nada / la UI dice unauthorized'>
    Los bind no loopback **requieren una ruta válida de autenticación de gateway**. En la práctica eso significa:

    - autenticación por secreto compartido: token o contraseña
    - `gateway.auth.mode: "trusted-proxy"` detrás de un proxy inverso con reconocimiento de identidad no loopback correctamente configurado

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    Notas:

    - `gateway.remote.token` / `.password` **no** habilitan por sí solos la autenticación de gateway local.
    - Las rutas de llamada locales pueden usar `gateway.remote.*` como alternativa solo cuando `gateway.auth.*` no está configurado.
    - Para autenticación por contraseña, establece `gateway.auth.mode: "password"` más `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
    - Si `gateway.auth.token` / `gateway.auth.password` se configuran explícitamente mediante SecretRef y no se resuelven, la resolución falla de forma segura (sin que el respaldo remoto lo enmascare).
    - Las configuraciones de Control UI con secreto compartido se autentican mediante `connect.params.auth.token` o `connect.params.auth.password` (almacenados en la configuración de app/UI). Los modos con identidad, como Tailscale Serve o `trusted-proxy`, usan en cambio encabezados de solicitud. Evita poner secretos compartidos en las URL.
    - Con `gateway.auth.mode: "trusted-proxy"`, los proxies inversos loopback en el mismo host siguen **sin** satisfacer la autenticación trusted-proxy. El proxy de confianza debe ser una fuente no loopback configurada.

  </Accordion>

  <Accordion title="¿Por qué ahora necesito un token en localhost?">
    OpenClaw aplica autenticación de gateway por defecto, incluido loopback. En la ruta predeterminada normal eso significa autenticación por token: si no se configura una ruta explícita de autenticación, el arranque de la gateway se resuelve al modo token y autogenera uno, guardándolo en `gateway.auth.token`, así que **los clientes WS locales deben autenticarse**. Esto bloquea que otros procesos locales llamen a la Gateway.

    Si prefieres una ruta de autenticación distinta, puedes elegir explícitamente el modo contraseña (o, para proxies inversos con reconocimiento de identidad no loopback, `trusted-proxy`). Si **de verdad** quieres loopback abierto, establece `gateway.auth.mode: "none"` explícitamente en tu configuración. Doctor puede generarte un token en cualquier momento: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="¿Tengo que reiniciar después de cambiar la configuración?">
    La Gateway observa la configuración y soporta hot-reload:

    - `gateway.reload.mode: "hybrid"` (predeterminado): aplica en caliente cambios seguros, reinicia para cambios críticos
    - también se soportan `hot`, `restart` y `off`

  </Accordion>

  <Accordion title="¿Cómo desactivo los eslóganes graciosos de la CLI?">
    Define `cli.banner.taglineMode` en la configuración:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: oculta el texto del eslogan pero mantiene la línea de título/versión del banner.
    - `default`: usa `All your chats, one OpenClaw.` siempre.
    - `random`: eslóganes graciosos/de temporada rotativos (comportamiento predeterminado).
    - Si no quieres ningún banner, define la variable de entorno `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="¿Cómo habilito la búsqueda web (y la obtención web)?">
    `web_fetch` funciona sin una clave API. `web_search` depende del
    proveedor seleccionado:

    - Los proveedores respaldados por API como Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity y Tavily requieren su configuración normal de clave API.
    - Ollama Web Search no usa clave, pero utiliza el host Ollama configurado y requiere `ollama signin`.
    - DuckDuckGo no usa clave, pero es una integración no oficial basada en HTML.
    - SearXNG no usa clave y es autoalojado; configura `SEARXNG_BASE_URL` o `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Recomendado:** ejecuta `openclaw configure --section web` y elige un proveedor.
    Alternativas por variable de entorno:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` o `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` o `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // optional; omit for auto-detect
            },
          },
        },
    }
    ```

    La configuración específica del proveedor para búsqueda web ahora vive bajo `plugins.entries.<plugin>.config.webSearch.*`.
    Las rutas heredadas `tools.web.search.*` del proveedor siguen cargándose temporalmente por compatibilidad, pero no deben usarse en configuraciones nuevas.
    La configuración de respaldo de obtención web de Firecrawl vive bajo `plugins.entries.firecrawl.config.webFetch.*`.

    Notas:

    - Si usas listas de permitidos, añade `web_search`/`web_fetch`/`x_search` o `group:web`.
    - `web_fetch` está habilitado por defecto (a menos que se deshabilite explícitamente).
    - Si se omite `tools.web.fetch.provider`, OpenClaw detecta automáticamente el primer proveedor de respaldo listo para fetch a partir de las credenciales disponibles. Hoy el proveedor incluido es Firecrawl.
    - Los daemons leen variables de entorno desde `~/.openclaw/.env` (o el entorno del servicio).

    Documentación: [Herramientas web](/es/tools/web).

  </Accordion>

  <Accordion title="config.apply borró mi configuración. ¿Cómo la recupero y evito esto?">
    `config.apply` sustituye la **configuración completa**. Si envías un objeto parcial, se elimina todo
    lo demás.

    Recuperación:

    - Restaura desde una copia de seguridad (git o una copia de `~/.openclaw/openclaw.json`).
    - Si no tienes copia de seguridad, vuelve a ejecutar `openclaw doctor` y reconfigura canales/modelos.
    - Si esto fue inesperado, informa de un error e incluye tu última configuración conocida o cualquier copia de seguridad.
    - Un agente local de programación a menudo puede reconstruir una configuración funcional a partir de registros o historial.

    Para evitarlo:

    - Usa `openclaw config set` para cambios pequeños.
    - Usa `openclaw configure` para ediciones interactivas.
    - Usa `config.schema.lookup` primero cuando no tengas claro una ruta exacta o la forma de un campo; devuelve un nodo de esquema superficial más resúmenes inmediatos de hijos para profundizar.
    - Usa `config.patch` para ediciones RPC parciales; reserva `config.apply` solo para reemplazo completo de configuración.
    - Si estás usando la herramienta `gateway` solo para el propietario desde una ejecución del agente, seguirá rechazando escrituras en `tools.exec.ask` / `tools.exec.security` (incluidos los alias heredados `tools.bash.*` que se normalizan a las mismas rutas protegidas de exec).

    Documentación: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/es/gateway/doctor).

  </Accordion>

  <Accordion title="¿Cómo ejecuto una Gateway central con workers especializados en distintos dispositivos?">
    El patrón habitual es **una Gateway** (p. ej. Raspberry Pi) más **nodos** y **agentes**:

    - **Gateway (central):** posee canales (Signal/WhatsApp), enrutamiento y sesiones.
    - **Nodos (dispositivos):** Macs/iOS/Android se conectan como periféricos y exponen herramientas locales (`system.run`, `canvas`, `camera`).
    - **Agentes (workers):** cerebros/workspaces separados para roles especiales (p. ej. “Hetzner ops”, “Personal data”).
    - **Subagentes:** crean trabajo en segundo plano desde un agente principal cuando quieres paralelismo.
    - **TUI:** se conecta a la Gateway y cambia entre agentes/sesiones.

    Documentación: [Nodos](/es/nodes), [Acceso remoto](/es/gateway/remote), [Enrutamiento multiagente](/es/concepts/multi-agent), [Subagentes](/es/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="¿Puede el navegador de OpenClaw ejecutarse en modo headless?">
    Sí. Es una opción de configuración:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    El valor predeterminado es `false` (con interfaz). El modo headless tiene más probabilidades de activar comprobaciones anti-bot en algunos sitios. Consulta [Browser](/es/tools/browser).

    Headless usa el **mismo motor Chromium** y funciona para la mayoría de automatizaciones (formularios, clics, scraping, inicios de sesión). Las diferencias principales:

    - No hay ventana visible del navegador (usa capturas si necesitas elementos visuales).
    - Algunos sitios son más estrictos con la automatización en modo headless (CAPTCHA, anti-bot).
      Por ejemplo, X/Twitter suele bloquear sesiones headless.

  </Accordion>

  <Accordion title="¿Cómo uso Brave para controlar el navegador?">
    Establece `browser.executablePath` en el binario de Brave (o cualquier navegador basado en Chromium) y reinicia la Gateway.
    Consulta los ejemplos completos de configuración en [Browser](/es/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateways remotas y nodos

<AccordionGroup>
  <Accordion title="¿Cómo se propagan los comandos entre Telegram, la gateway y los nodos?">
    Los mensajes de Telegram son gestionados por la **gateway**. La gateway ejecuta el agente y
    solo entonces llama a nodos mediante el **WebSocket de la Gateway** cuando se necesita una herramienta de nodo:

    Telegram → Gateway → Agente → `node.*` → Nodo → Gateway → Telegram

    Los nodos no ven tráfico entrante del proveedor; solo reciben llamadas RPC de nodo.

  </Accordion>

  <Accordion title="¿Cómo puede mi agente acceder a mi ordenador si la Gateway está alojada en remoto?">
    Respuesta corta: **empareja tu ordenador como nodo**. La Gateway se ejecuta en otro sitio, pero puede
    llamar herramientas `node.*` (pantalla, cámara, sistema) en tu máquina local mediante el WebSocket de la Gateway.

    Configuración típica:

    1. Ejecuta la Gateway en el host siempre activo (VPS/servidor doméstico).
    2. Pon el host de la Gateway + tu ordenador en la misma tailnet.
    3. Asegúrate de que la WS de la Gateway sea accesible (bind de tailnet o túnel SSH).
    4. Abre la app macOS localmente y conéctate en modo **Remote over SSH** (o tailnet directo)
       para que pueda registrarse como nodo.
    5. Aprueba el nodo en la Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    No se requiere ningún puente TCP aparte; los nodos se conectan mediante el WebSocket de la Gateway.

    Recordatorio de seguridad: emparejar un nodo macOS permite `system.run` en esa máquina. Solo
    empareja dispositivos en los que confíes y revisa [Security](/es/gateway/security).

    Documentación: [Nodos](/es/nodes), [Protocolo de Gateway](/es/gateway/protocol), [Modo remoto macOS](/es/platforms/mac/remote), [Security](/es/gateway/security).

  </Accordion>

  <Accordion title="Tailscale está conectado pero no obtengo respuestas. ¿Y ahora qué?">
    Revisa lo básico:

    - La Gateway está ejecutándose: `openclaw gateway status`
    - Salud de la Gateway: `openclaw status`
    - Salud del canal: `openclaw channels status`

    Luego verifica autenticación y enrutamiento:

    - Si usas Tailscale Serve, asegúrate de que `gateway.auth.allowTailscale` esté configurado correctamente.
    - Si te conectas mediante túnel SSH, confirma que el túnel local esté activo y apunte al puerto correcto.
    - Confirma que tus listas de permitidos (DM o grupo) incluyan tu cuenta.

    Documentación: [Tailscale](/es/gateway/tailscale), [Acceso remoto](/es/gateway/remote), [Canales](/es/channels).

  </Accordion>

  <Accordion title="¿Pueden dos instancias de OpenClaw hablar entre sí (local + VPS)?">
    Sí. No hay un puente “bot a bot” integrado, pero puedes conectarlo de varias
    formas fiables:

    **Lo más simple:** usa un canal de chat normal al que ambos bots puedan acceder (Telegram/Slack/WhatsApp).
    Haz que el Bot A envíe un mensaje al Bot B y luego deja que el Bot B responda como siempre.

    **Puente CLI (genérico):** ejecuta un script que llame a la otra Gateway con
    `openclaw agent --message ... --deliver`, apuntando a un chat donde el otro bot
    escuche. Si uno de los bots está en un VPS remoto, apunta tu CLI a esa Gateway remota
    mediante SSH/Tailscale (consulta [Acceso remoto](/es/gateway/remote)).

    Patrón de ejemplo (ejecutar desde una máquina que pueda llegar a la Gateway objetivo):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Consejo: añade una medida de protección para que los dos bots no entren en un bucle infinito (solo menciones,
    listas de permitidos del canal o una regla de “no responder a mensajes de bots”).

    Documentación: [Acceso remoto](/es/gateway/remote), [CLI de agente](/cli/agent), [Agent send](/es/tools/agent-send).

  </Accordion>

  <Accordion title="¿Necesito VPS separados para varios agentes?">
    No. Una Gateway puede alojar varios agentes, cada uno con su propio workspace, modelos predeterminados
    y enrutamiento. Esa es la configuración normal y es mucho más barata y simple que ejecutar
    un VPS por agente.

    Usa VPS separados solo cuando necesites aislamiento rígido (límites de seguridad) o configuraciones muy
    distintas que no quieras compartir. En caso contrario, mantén una sola Gateway y
    usa múltiples agentes o subagentes.

  </Accordion>

  <Accordion title="¿Hay ventaja en usar un nodo en mi portátil personal en vez de SSH desde un VPS?">
    Sí: los nodos son la forma de primera clase de acceder a tu portátil desde una Gateway remota, y
    desbloquean más que acceso shell. La Gateway se ejecuta en macOS/Linux (Windows mediante WSL2) y es
    ligera (un VPS pequeño o una máquina de la clase Raspberry Pi sirve; 4 GB de RAM es más que suficiente), así que una
    configuración habitual es un host siempre activo más tu portátil como nodo.

    - **No requiere SSH entrante.** Los nodos se conectan hacia fuera al WebSocket de la Gateway y usan emparejamiento de dispositivos.
    - **Controles de ejecución más seguros.** `system.run` queda protegido por listas de permitidos/aprobaciones del nodo en ese portátil.
    - **Más herramientas del dispositivo.** Los nodos exponen `canvas`, `camera` y `screen` además de `system.run`.
    - **Automatización local del navegador.** Mantén la Gateway en un VPS, pero ejecuta Chrome localmente mediante un host de nodo en el portátil, o adjúntate al Chrome local del host mediante Chrome MCP.

    SSH sirve para acceso shell puntual, pero los nodos son más simples para flujos continuos de agentes y
    automatización del dispositivo.

    Documentación: [Nodos](/es/nodes), [CLI de nodos](/cli/nodes), [Browser](/es/tools/browser).

  </Accordion>

  <Accordion title="¿Los nodos ejecutan un servicio gateway?">
    No. Solo debe ejecutarse **una gateway** por host salvo que intencionalmente ejecutes perfiles aislados (consulta [Múltiples gateways](/es/gateway/multiple-gateways)). Los nodos son periféricos que se conectan
    a la gateway (nodos iOS/Android, o “modo nodo” macOS en la app de barra de menús). Para hosts de nodo sin interfaz
    y control por CLI, consulta [CLI de host de nodo](/cli/node).

    Se requiere un reinicio completo para cambios en `gateway`, `discovery` y `canvasHost`.

  </Accordion>

  <Accordion title="¿Hay una forma API / RPC de aplicar configuración?">
    Sí.

    - `config.schema.lookup`: inspecciona un subárbol de configuración con su nodo de esquema superficial, pista de UI coincidente y resúmenes inmediatos de hijos antes de escribir
    - `config.get`: obtiene la instantánea actual + hash
    - `config.patch`: actualización parcial segura (preferida para la mayoría de las ediciones RPC); hace hot-reload cuando es posible y reinicia cuando es necesario
    - `config.apply`: valida + reemplaza la configuración completa; hace hot-reload cuando es posible y reinicia cuando es necesario
    - La herramienta de tiempo de ejecución `gateway`, solo para el propietario, sigue negándose a reescribir `tools.exec.ask` / `tools.exec.security`; los alias heredados `tools.bash.*` se normalizan a las mismas rutas protegidas de exec

  </Accordion>

  <Accordion title="Configuración mínima razonable para una primera instalación">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Esto establece tu workspace y restringe quién puede activar el bot.

  </Accordion>

  <Accordion title="¿Cómo configuro Tailscale en un VPS y me conecto desde mi Mac?">
    Pasos mínimos:

    1. **Instala + inicia sesión en el VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Instala + inicia sesión en tu Mac**
       - Usa la app de Tailscale e inicia sesión en la misma tailnet.
    3. **Habilita MagicDNS (recomendado)**
       - En la consola de administración de Tailscale, habilita MagicDNS para que el VPS tenga un nombre estable.
    4. **Usa el hostname de la tailnet**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Si quieres la Control UI sin SSH, usa Tailscale Serve en el VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Esto mantiene la gateway enlazada a loopback y expone HTTPS a través de Tailscale. Consulta [Tailscale](/es/gateway/tailscale).

  </Accordion>

  <Accordion title="¿Cómo conecto un nodo Mac a una Gateway remota (Tailscale Serve)?">
    Serve expone la **Control UI + WS** de la Gateway. Los nodos se conectan mediante el mismo endpoint WS de la Gateway.

    Configuración recomendada:

    1. **Asegúrate de que el VPS + Mac estén en la misma tailnet**.
    2. **Usa la app macOS en modo Remote** (el destino SSH puede ser el hostname de la tailnet).
       La app tunelizará el puerto de la Gateway y se conectará como nodo.
    3. **Aprueba el nodo** en la gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentación: [Protocolo de Gateway](/es/gateway/protocol), [Discovery](/es/gateway/discovery), [Modo remoto macOS](/es/platforms/mac/remote).

  </Accordion>

  <Accordion title="¿Debo instalar en un segundo portátil o simplemente añadir un nodo?">
    Si solo necesitas **herramientas locales** (pantalla/cámara/exec) en el segundo portátil, añádelo como
    **nodo**. Eso mantiene una sola Gateway y evita configuración duplicada. Las herramientas de nodo local
    actualmente solo están disponibles en macOS, pero planeamos ampliarlas a otros SO.

    Instala una segunda Gateway solo cuando necesites **aislamiento rígido** o dos bots completamente separados.

    Documentación: [Nodos](/es/nodes), [CLI de nodos](/cli/nodes), [Múltiples gateways](/es/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Variables de entorno y carga de .env

<AccordionGroup>
  <Accordion title="¿Cómo carga OpenClaw las variables de entorno?">
    OpenClaw lee variables de entorno del proceso padre (shell, launchd/systemd, CI, etc.) y además carga:

    - `.env` desde el directorio de trabajo actual
    - un `.env` global de respaldo desde `~/.openclaw/.env` (también conocido como `$OPENCLAW_STATE_DIR/.env`)

    Ninguno de los archivos `.env` sobrescribe variables de entorno ya existentes.

    También puedes definir variables de entorno en línea en la configuración (solo se aplican si faltan en el entorno del proceso):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Consulta [/environment](/es/help/environment) para la precedencia completa y las fuentes.

  </Accordion>

  <Accordion title="Inicié la Gateway mediante el servicio y mis variables de entorno desaparecieron. ¿Y ahora qué?">
    Dos soluciones habituales:

    1. Pon las claves que faltan en `~/.openclaw/.env` para que se recojan incluso cuando el servicio no hereda el entorno de tu shell.
    2. Habilita la importación de shell (comodidad opcional):

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    Esto ejecuta tu shell de inicio de sesión e importa solo las claves esperadas que falten (nunca sobrescribe). Equivalentes por variable de entorno:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Definí COPILOT_GITHUB_TOKEN, pero models status muestra "Shell env: off." ¿Por qué?'>
    `openclaw models status` informa si la **importación de variables de entorno del shell** está habilitada. “Shell env: off”
    **no** significa que tus variables de entorno falten; solo significa que OpenClaw no cargará
    automáticamente tu shell de inicio de sesión.

    Si la Gateway se ejecuta como servicio (launchd/systemd), no heredará tu
    entorno del shell. Corrígelo haciendo una de estas cosas:

    1. Pon el token en `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. O habilita la importación de shell (`env.shellEnv.enabled: true`).
    3. O añádelo al bloque `env` de tu configuración (solo se aplica si falta).

    Luego reinicia la gateway y vuelve a comprobar:

    ```bash
    openclaw models status
    ```

    Los tokens de Copilot se leen desde `COPILOT_GITHUB_TOKEN` (también `GH_TOKEN` / `GITHUB_TOKEN`).
    Consulta [/concepts/model-providers](/es/concepts/model-providers) y [/environment](/es/help/environment).

  </Accordion>
</AccordionGroup>

## Sesiones y múltiples chats

<AccordionGroup>
  <Accordion title="¿Cómo inicio una conversación nueva?">
    Envía `/new` o `/reset` como mensaje independiente. Consulta [Gestión de sesiones](/es/concepts/session).
  </Accordion>

  <Accordion title="¿Las sesiones se reinician automáticamente si nunca envío /new?">
    Las sesiones pueden expirar después de `session.idleMinutes`, pero esto está **deshabilitado por defecto** (valor predeterminado **0**).
    Establécelo a un valor positivo para habilitar el vencimiento por inactividad. Cuando está habilitado, el **siguiente**
    mensaje tras el período de inactividad inicia un id de sesión nuevo para esa clave de chat.
    Esto no elimina transcripciones; solo inicia una sesión nueva.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="¿Hay alguna forma de crear un equipo de instancias de OpenClaw (un CEO y muchos agentes)?">
    Sí, mediante **enrutamiento multiagente** y **subagentes**. Puedes crear un agente coordinador
    y varios agentes worker con sus propios workspaces y modelos.

    Dicho esto, esto se ve mejor como un **experimento divertido**. Consume muchos tokens y a menudo
    es menos eficiente que usar un solo bot con sesiones separadas. El modelo típico que
    imaginamos es un solo bot con el que hablas, con distintas sesiones para trabajo en paralelo. Ese
    bot también puede crear subagentes cuando hace falta.

    Documentación: [Enrutamiento multiagente](/es/concepts/multi-agent), [Subagentes](/es/tools/subagents), [CLI de agentes](/cli/agents).

  </Accordion>

  <Accordion title="¿Por qué el contexto se truncó a mitad de una tarea? ¿Cómo lo evito?">
    El contexto de sesión está limitado por la ventana del modelo. Chats largos, salidas grandes de herramientas o muchos
    archivos pueden activar compactación o truncado.

    Qué ayuda:

    - Pide al bot que resuma el estado actual y lo escriba en un archivo.
    - Usa `/compact` antes de tareas largas, y `/new` al cambiar de tema.
    - Mantén el contexto importante en el workspace y pide al bot que lo lea de nuevo.
    - Usa subagentes para trabajo largo o paralelo para que el chat principal siga siendo más pequeño.
    - Elige un modelo con una ventana de contexto más grande si esto ocurre a menudo.

  </Accordion>

  <Accordion title="¿Cómo reinicio completamente OpenClaw pero lo mantengo instalado?">
    Usa el comando de reinicio:

    ```bash
    openclaw reset
    ```

    Reinicio completo no interactivo:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Luego vuelve a ejecutar la configuración:

    ```bash
    openclaw onboard --install-daemon
    ```

    Notas:

    - Onboarding también ofrece **Reset** si detecta una configuración existente. Consulta [Onboarding (CLI)](/es/start/wizard).
    - Si usaste perfiles (`--profile` / `OPENCLAW_PROFILE`), reinicia cada directorio de estado (los predeterminados son `~/.openclaw-<profile>`).
    - Reinicio de desarrollo: `openclaw gateway --dev --reset` (solo desarrollo; borra config + credenciales + sesiones + workspace de desarrollo).

  </Accordion>

  <Accordion title='Recibo errores de "context too large". ¿Cómo reinicio o compacto?'>
    Usa una de estas opciones:

    - **Compactar** (mantiene la conversación pero resume los turnos anteriores):

      ```
      /compact
      ```

      o `/compact <instructions>` para guiar el resumen.

    - **Reiniciar** (id de sesión nueva para la misma clave de chat):

      ```
      /new
      /reset
      ```

    Si sigue ocurriendo:

    - Habilita o ajusta la **depuración de sesión** (`agents.defaults.contextPruning`) para recortar salidas antiguas de herramientas.
    - Usa un modelo con una ventana de contexto más grande.

    Documentación: [Compactación](/es/concepts/compaction), [Depuración de sesión](/es/concepts/session-pruning), [Gestión de sesiones](/es/concepts/session).

  </Accordion>

  <Accordion title='¿Por qué veo "LLM request rejected: messages.content.tool_use.input field required"?'>
    Este es un error de validación del proveedor: el modelo emitió un bloque `tool_use` sin el
    `input` requerido. Normalmente significa que el historial de la sesión está obsoleto o corrupto (a menudo después de hilos largos
    o un cambio de herramienta/esquema).

    Solución: inicia una sesión nueva con `/new` (mensaje independiente).

  </Accordion>

  <Accordion title="¿Por qué recibo mensajes de heartbeat cada 30 minutos?">
    Los heartbeats se ejecutan cada **30m** por defecto (**1h** cuando se usa autenticación OAuth). Ajústalos o desactívalos:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    Si `HEARTBEAT.md` existe pero está efectivamente vacío (solo líneas en blanco y
    encabezados markdown como `# Heading`), OpenClaw omite la ejecución de heartbeat para ahorrar llamadas a la API.
    Si falta el archivo, el heartbeat sigue ejecutándose y el modelo decide qué hacer.

    Los overrides por agente usan `agents.list[].heartbeat`. Documentación: [Heartbeat](/es/gateway/heartbeat).

  </Accordion>

  <Accordion title='¿Necesito añadir una "cuenta de bot" a un grupo de WhatsApp?'>
    No. OpenClaw se ejecuta en **tu propia cuenta**, así que si estás en el grupo, OpenClaw puede verlo.
    Por defecto, las respuestas en grupo están bloqueadas hasta que permites remitentes (`groupPolicy: "allowlist"`).

    Si quieres que solo **tú** puedas activar respuestas en grupo:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="¿Cómo obtengo el JID de un grupo de WhatsApp?">
    Opción 1 (más rápida): sigue los registros y envía un mensaje de prueba en el grupo:

    ```bash
    openclaw logs --follow --json
    ```

    Busca `chatId` (o `from`) terminado en `@g.us`, por ejemplo:
    `1234567890-1234567890@g.us`.

    Opción 2 (si ya está configurado/en lista de permitidos): lista grupos desde la configuración:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentación: [WhatsApp](/es/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="¿Por qué OpenClaw no responde en un grupo?">
    Dos causas comunes:

    - La restricción por mención está activada (predeterminado). Debes mencionar con @ al bot (o coincidir con `mentionPatterns`).
    - Configuraste `channels.whatsapp.groups` sin `"*"` y el grupo no está en la lista de permitidos.

    Consulta [Grupos](/es/channels/groups) y [Mensajes de grupo](/es/channels/group-messages).

  </Accordion>

  <Accordion title="¿Los grupos/hilos comparten contexto con los DM?">
    Los chats directos colapsan en la sesión principal por defecto. Los grupos/canales tienen sus propias claves de sesión, y los temas de Telegram / hilos de Discord son sesiones separadas. Consulta [Grupos](/es/channels/groups) y [Mensajes de grupo](/es/channels/group-messages).
  </Accordion>

  <Accordion title="¿Cuántos workspaces y agentes puedo crear?">
    No hay límites estrictos. Decenas (incluso cientos) están bien, pero vigila:

    - **Crecimiento de disco:** sesiones + transcripciones viven bajo `~/.openclaw/agents/<agentId>/sessions/`.
    - **Coste en tokens:** más agentes significa más uso concurrente de modelos.
    - **Carga operativa:** perfiles de autenticación por agente, workspaces y enrutamiento de canales.

    Consejos:

    - Mantén un workspace **activo** por agente (`agents.defaults.workspace`).
    - Depura sesiones antiguas (elimina JSONL o entradas de almacén) si el disco crece.
    - Usa `openclaw doctor` para detectar workspaces huérfanos y desajustes de perfiles.

  </Accordion>

  <Accordion title="¿Puedo ejecutar varios bots o chats al mismo tiempo (Slack), y cómo debería configurarlo?">
    Sí. Usa **Enrutamiento multiagente** para ejecutar varios agentes aislados y enrutar mensajes entrantes por
    canal/cuenta/peer. Slack está soportado como canal y puede vincularse a agentes específicos.

    El acceso al navegador es potente, pero no “puede hacer cualquier cosa que haría un humano”: anti-bot, CAPTCHA y MFA
    pueden seguir bloqueando la automatización. Para el control de navegador más fiable, usa Chrome MCP local en el host,
    o usa CDP en la máquina que realmente ejecuta el navegador.

    Configuración recomendada:

    - Host Gateway siempre activo (VPS/Mac mini).
    - Un agente por rol (bindings).
    - Canales de Slack vinculados a esos agentes.
    - Navegador local mediante Chrome MCP o un nodo cuando haga falta.

    Documentación: [Enrutamiento multiagente](/es/concepts/multi-agent), [Slack](/es/channels/slack),
    [Browser](/es/tools/browser), [Nodos](/es/nodes).

  </Accordion>
</AccordionGroup>

## Modelos: valores predeterminados, selección, alias, cambio

<AccordionGroup>
  <Accordion title='¿Qué es el "modelo predeterminado"?'>
    El modelo predeterminado de OpenClaw es el que establezcas como:

    ```
    agents.defaults.model.primary
    ```

    Los modelos se referencian como `provider/model` (ejemplo: `openai/gpt-5.4`). Si omites el proveedor, OpenClaw primero intenta un alias, luego una coincidencia única de proveedor configurado para ese id exacto de modelo, y solo después recurre al proveedor predeterminado configurado como ruta heredada de compatibilidad. Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw recurre al primer proveedor/modelo configurado en lugar de mostrar un valor predeterminado obsoleto de un proveedor eliminado. Aun así deberías **establecer explícitamente** `provider/model`.

  </Accordion>

  <Accordion title="¿Qué modelo recomiendan?">
    **Predeterminado recomendado:** usa el modelo más potente y de última generación disponible en tu pila de proveedores.
    **Para agentes con herramientas o con entradas no confiables:** prioriza la calidad del modelo sobre el coste.
    **Para chat rutinario/de bajo riesgo:** usa modelos alternativos más baratos y enruta por rol de agente.

    MiniMax tiene su propia documentación: [MiniMax](/es/providers/minimax) y
    [Modelos locales](/es/gateway/local-models).

    Regla general: usa el **mejor modelo que puedas pagar** para trabajo de alto impacto, y un modelo más barato
    para chat rutinario o resúmenes. Puedes enrutar modelos por agente y usar subagentes para
    paralelizar tareas largas (cada subagente consume tokens). Consulta [Modelos](/es/concepts/models) y
    [Subagentes](/es/tools/subagents).

    Advertencia importante: los modelos más débiles o excesivamente cuantizados son más vulnerables a la inyección de prompts
    y a comportamientos inseguros. Consulta [Security](/es/gateway/security).

    Más contexto: [Modelos](/es/concepts/models).

  </Accordion>

  <Accordion title="¿Cómo cambio de modelo sin borrar mi configuración?">
    Usa **comandos de modelo** o edita solo los campos de **modelo**. Evita reemplazos completos de configuración.

    Opciones seguras:

    - `/model` en el chat (rápido, por sesión)
    - `openclaw models set ...` (actualiza solo la configuración del modelo)
    - `openclaw configure --section model` (interactivo)
    - editar `agents.defaults.model` en `~/.openclaw/openclaw.json`

    Evita `config.apply` con un objeto parcial a menos que quieras reemplazar toda la configuración.
    Para ediciones RPC, inspecciona primero con `config.schema.lookup` y prefiere `config.patch`. La carga de lookup te da la ruta normalizada, documentación/restricciones superficiales del esquema y resúmenes inmediatos de hijos
    para actualizaciones parciales.
    Si sobrescribiste la configuración, restaúrala desde una copia de seguridad o vuelve a ejecutar `openclaw doctor` para repararla.

    Documentación: [Modelos](/es/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/es/gateway/doctor).

  </Accordion>

  <Accordion title="¿Puedo usar modelos autoalojados (llama.cpp, vLLM, Ollama)?">
    Sí. Ollama es la ruta más fácil para modelos locales.

    Configuración más rápida:

    1. Instala Ollama desde `https://ollama.com/download`
    2. Descarga un modelo local como `ollama pull gemma4`
    3. Si también quieres modelos en la nube, ejecuta `ollama signin`
    4. Ejecuta `openclaw onboard` y elige `Ollama`
    5. Elige `Local` o `Cloud + Local`

    Notas:

    - `Cloud + Local` te da modelos en la nube más tus modelos locales de Ollama
    - los modelos en la nube como `kimi-k2.5:cloud` no necesitan descarga local
    - para cambiar manualmente, usa `openclaw models list` y `openclaw models set ollama/<model>`

    Nota de seguridad: los modelos más pequeños o muy cuantizados son más vulnerables a la inyección de prompts.
    Recomendamos firmemente **modelos grandes** para cualquier bot que pueda usar herramientas.
    Si aun así quieres modelos pequeños, habilita sandboxing y listas estrictas de herramientas permitidas.

    Documentación: [Ollama](/es/providers/ollama), [Modelos locales](/es/gateway/local-models),
    [Proveedores de modelos](/es/concepts/model-providers), [Security](/es/gateway/security),
    [Sandboxing](/es/gateway/sandboxing).

  </Accordion>

  <Accordion title="¿Qué usan OpenClaw, Flawd y Krill para los modelos?">
    - Estos despliegues pueden diferir y cambiar con el tiempo; no hay una recomendación fija de proveedor.
    - Comprueba la configuración actual en tiempo de ejecución en cada gateway con `openclaw models status`.
    - Para agentes sensibles a la seguridad/con herramientas, usa el modelo más potente y de última generación disponible.
  </Accordion>

  <Accordion title="¿Cómo cambio modelos sobre la marcha (sin reiniciar)?">
    Usa el comando `/model` como mensaje independiente:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    Estos son los alias integrados. Los alias personalizados pueden añadirse mediante `agents.defaults.models`.

    Puedes listar los modelos disponibles con `/model`, `/model list` o `/model status`.

    `/model` (y `/model list`) muestra un selector compacto y numerado. Selecciona por número:

    ```
    /model 3
    ```

    También puedes forzar un perfil de autenticación específico para el proveedor (por sesión):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Consejo: `/model status` muestra qué agente está activo, qué archivo `auth-profiles.json` se está usando y qué perfil de autenticación se intentará a continuación.
    También muestra el endpoint configurado del proveedor (`baseUrl`) y el modo API (`api`) cuando están disponibles.

    **¿Cómo quito la fijación de un perfil que configuré con @profile?**

    Vuelve a ejecutar `/model` **sin** el sufijo `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    Si quieres volver al valor predeterminado, elígelo desde `/model` (o envía `/model <default provider/model>`).
    Usa `/model status` para confirmar qué perfil de autenticación está activo.

  </Accordion>

  <Accordion title="¿Puedo usar GPT 5.2 para tareas diarias y Codex 5.3 para programar?">
    Sí. Establece uno como predeterminado y cambia según sea necesario:

    - **Cambio rápido (por sesión):** `/model gpt-5.4` para tareas diarias, `/model openai-codex/gpt-5.4` para programar con Codex OAuth.
    - **Predeterminado + cambio:** establece `agents.defaults.model.primary` en `openai/gpt-5.4`, luego cambia a `openai-codex/gpt-5.4` al programar (o al revés).
    - **Subagentes:** enruta tareas de programación a subagentes con un modelo predeterminado distinto.

    Consulta [Modelos](/es/concepts/models) y [Comandos slash](/es/tools/slash-commands).

  </Accordion>

  <Accordion title="¿Cómo configuro el modo rápido para GPT 5.4?">
    Usa un interruptor por sesión o un valor predeterminado de configuración:

    - **Por sesión:** envía `/fast on` mientras la sesión usa `openai/gpt-5.4` o `openai-codex/gpt-5.4`.
    - **Predeterminado por modelo:** establece `agents.defaults.models["openai/gpt-5.4"].params.fastMode` en `true`.
    - **También para Codex OAuth:** si también usas `openai-codex/gpt-5.4`, establece la misma bandera allí.

    Ejemplo:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    Para OpenAI, el modo rápido se asigna a `service_tier = "priority"` en solicitudes nativas Responses soportadas. Los overrides de sesión `/fast` prevalecen sobre los valores predeterminados de configuración.

    Consulta [Thinking y modo rápido](/es/tools/thinking) y [Modo rápido de OpenAI](/es/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='¿Por qué veo "Model ... is not allowed" y luego no hay respuesta?'>
    Si `agents.defaults.models` está configurado, se convierte en la **lista de permitidos** para `/model` y cualquier
    override de sesión. Elegir un modelo que no esté en esa lista devuelve:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Ese error se devuelve **en lugar de** una respuesta normal. Solución: añade el modelo a
    `agents.defaults.models`, elimina la lista de permitidos o elige un modelo de `/model list`.

  </Accordion>

  <Accordion title='¿Por qué veo "Unknown model: minimax/MiniMax-M2.7"?'>
    Esto significa que el **proveedor no está configurado** (no se encontró configuración de proveedor MiniMax ni perfil de
    autenticación), así que el modelo no puede resolverse.

    Lista de comprobación para solucionarlo:

    1. Actualiza a una versión actual de OpenClaw (o ejecuta desde el código fuente `main`) y reinicia la gateway.
    2. Asegúrate de que MiniMax está configurado (asistente o JSON), o de que exista autenticación de MiniMax
       en variables de entorno/perfiles de autenticación para que pueda inyectarse el proveedor correspondiente
       (`MINIMAX_API_KEY` para `minimax`, `MINIMAX_OAUTH_TOKEN` o OAuth MiniMax almacenado
       para `minimax-portal`).
    3. Usa el id exacto del modelo (distingue mayúsculas/minúsculas) para tu ruta de autenticación:
       `minimax/MiniMax-M2.7` o `minimax/MiniMax-M2.7-highspeed` para configuración con clave API,
       o `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` para configuración con OAuth.
    4. Ejecuta:

       ```bash
       openclaw models list
       ```

       y elige de la lista (o `/model list` en el chat).

    Consulta [MiniMax](/es/providers/minimax) y [Modelos](/es/concepts/models).

  </Accordion>

  <Accordion title="¿Puedo usar MiniMax como predeterminado y OpenAI para tareas complejas?">
    Sí. Usa **MiniMax como predeterminado** y cambia modelos **por sesión** cuando sea necesario.
    Los fallbacks son para **errores**, no para “tareas difíciles”, así que usa `/model` o un agente separado.

    **Opción A: cambiar por sesión**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Luego:

    ```
    /model gpt
    ```

    **Opción B: agentes separados**

    - Agente A predeterminado: MiniMax
    - Agente B predeterminado: OpenAI
    - Enruta por agente o usa `/agent` para cambiar

    Documentación: [Modelos](/es/concepts/models), [Enrutamiento multiagente](/es/concepts/multi-agent), [MiniMax](/es/providers/minimax), [OpenAI](/es/providers/openai).

  </Accordion>

  <Accordion title="¿opus / sonnet / gpt son atajos integrados?">
    Sí. OpenClaw incluye algunos atajos predeterminados (solo se aplican cuando el modelo existe en `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Si defines tu propio alias con el mismo nombre, gana tu valor.

  </Accordion>

  <Accordion title="¿Cómo defino/sobrescribo atajos de modelo (aliases)?">
    Los alias provienen de `agents.defaults.models.<modelId>.alias`. Ejemplo:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    Luego `/model sonnet` (o `/<alias>` cuando esté soportado) se resuelve a ese id de modelo.

  </Accordion>

  <Accordion title="¿Cómo añado modelos de otros proveedores como OpenRouter o Z.AI?">
    OpenRouter (pago por token; muchos modelos):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (modelos GLM):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Si haces referencia a un proveedor/modelo pero falta la clave requerida del proveedor, recibirás un error de autenticación en tiempo de ejecución (por ejemplo `No API key found for provider "zai"`).

    **No API key found for provider after adding a new agent**

    Esto suele significar que el **nuevo agente** tiene un almacén de autenticación vacío. La autenticación es por agente y
    se almacena en:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Opciones para corregirlo:

    - Ejecuta `openclaw agents add <id>` y configura la autenticación durante el asistente.
    - O copia `auth-profiles.json` desde el `agentDir` del agente principal al `agentDir` del nuevo agente.

    **No** reutilices `agentDir` entre agentes; provoca colisiones de autenticación/sesión.

  </Accordion>
</AccordionGroup>

## Conmutación por error de modelos y "All models failed"

<AccordionGroup>
  <Accordion title="¿Cómo funciona la conmutación por error?">
    La conmutación por error ocurre en dos etapas:

    1. **Rotación de perfil de autenticación** dentro del mismo proveedor.
    2. **Fallback de modelo** al siguiente modelo en `agents.defaults.model.fallbacks`.

    Se aplican periodos de espera a los perfiles que fallan (retroceso exponencial), para que OpenClaw pueda seguir respondiendo incluso cuando un proveedor está limitado por tasa o falla temporalmente.

    El bucket de límite de tasa incluye algo más que respuestas `429` simples. OpenClaw
    también trata mensajes como `Too many concurrent requests`,
    `ThrottlingException`, `concurrency limit reached`,
    `workers_ai ... quota limit exceeded`, `resource exhausted` y límites
    periódicos de ventana de uso (`weekly/monthly limit reached`) como
    límites de tasa aptos para conmutación por error.

    Algunas respuestas que parecen de facturación no son `402`, y algunas respuestas HTTP `402`
    también permanecen en ese bucket transitorio. Si un proveedor devuelve
    texto explícito de facturación en `401` o `403`, OpenClaw aún puede mantener eso en
    la vía de facturación, pero los comparadores de texto específicos del proveedor siguen limitados al
    proveedor que les corresponde (por ejemplo OpenRouter `Key limit exceeded`). Si un mensaje `402`
    en cambio parece un límite de ventana de uso reintentable o de gasto de organización/workspace (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`), OpenClaw lo trata como
    `rate_limit`, no como una desactivación larga por facturación.

    Los errores de desbordamiento de contexto son distintos: firmas como
    `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`,
    `input is too long for the model`, o `ollama error: context length
    exceeded` se mantienen en la ruta de compactación/reintento en lugar de avanzar
    al fallback de modelo.

    El texto genérico de error de servidor es intencionadamente más estricto que “cualquier cosa con
    unknown/error dentro”. OpenClaw sí trata formas transitorias con contexto de proveedor
    como Anthropic bare `An unknown error occurred`, OpenRouter bare
    `Provider returned error`, errores de stop-reason como `Unhandled stop reason:
    error`, cargas JSON `api_error` con texto transitorio de servidor
    (`internal server error`, `unknown error, 520`, `upstream error`, `backend
    error`), y errores de proveedor ocupado como `ModelNotReadyException` como
    señales de timeout/sobrecarga aptas para conmutación por error cuando el contexto del proveedor
    coincide.
    El texto genérico interno de fallback como `LLM request failed with an unknown
    error.` se mantiene conservador y no activa por sí solo fallback de modelo.

  </Accordion>

  <Accordion title='¿Qué significa "No credentials found for profile anthropic:default"?'>
    Significa que el sistema intentó usar el id de perfil de autenticación `anthropic:default`, pero no pudo encontrar credenciales para él en el almacén de autenticación esperado.

    **Lista de comprobación para corregirlo:**

    - **Confirma dónde viven los perfiles de autenticación** (rutas nuevas frente a heredadas)
      - Actual: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
      - Heredada: `~/.openclaw/agent/*` (migrada por `openclaw doctor`)
    - **Confirma que la variable de entorno está cargada por la Gateway**
      - Si definiste `ANTHROPIC_API_KEY` en tu shell pero ejecutas la Gateway mediante systemd/launchd, puede que no la herede. Ponla en `~/.openclaw/.env` o habilita `env.shellEnv`.
    - **Asegúrate de estar editando el agente correcto**
      - Las configuraciones multiagente implican que puede haber varios archivos `auth-profiles.json`.
    - **Comprueba el estado de modelo/autenticación**
      - Usa `openclaw models status` para ver los modelos configurados y si los proveedores están autenticados.

    **Lista de comprobación para "No credentials found for profile anthropic"**

    Esto significa que la ejecución está fijada a un perfil de autenticación de Anthropic, pero la Gateway
    no puede encontrarlo en su almacén de autenticación.

    - **Usa Claude CLI**
      - Ejecuta `openclaw models auth login --provider anthropic --method cli --set-default` en el host de la gateway.
    - **Si quieres usar una clave API en su lugar**
      - Pon `ANTHROPIC_API_KEY` en `~/.openclaw/.env` en el **host de la gateway**.
      - Elimina cualquier orden fijado que fuerce un perfil inexistente:

        ```bash
        openclaw models auth order clear --provider anthropic
        ```

    - **Confirma que estás ejecutando los comandos en el host de la gateway**
      - En modo remoto, los perfiles de autenticación viven en la máquina gateway, no en tu portátil.

  </Accordion>

  <Accordion title="¿Por qué también intentó Google Gemini y falló