---
read_when:
    - Respondes preguntas comunes sobre configuración, instalación, incorporación o soporte en tiempo de ejecución
    - Haces triaje de problemas reportados por usuarios antes de una depuración más profunda
summary: Preguntas frecuentes sobre la configuración, instalación y uso de OpenClaw
title: Preguntas frecuentes
x-i18n:
    generated_at: "2026-04-05T12:49:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0f71dc12f60aceaa1d095aaa4887d59ecf2a53e349d10a3e2f60e464ae48aff6
    source_path: help/faq.md
    workflow: 15
---

# Preguntas frecuentes

Respuestas rápidas más una solución de problemas más profunda para configuraciones del mundo real (desarrollo local, VPS, varios agentes, OAuth/claves de API, conmutación por error de modelos). Para diagnósticos de tiempo de ejecución, consulta [Troubleshooting](/gateway/troubleshooting). Para la referencia completa de configuración, consulta [Configuration](/gateway/configuration).

## Los primeros 60 segundos si algo está roto

1. **Estado rápido (primera comprobación)**

   ```bash
   openclaw status
   ```

   Resumen local rápido: SO + actualización, accesibilidad/servicio del gateway, agentes/sesiones, configuración de proveedor + problemas de tiempo de ejecución (cuando el gateway es accesible).

2. **Informe que se puede pegar (seguro para compartir)**

   ```bash
   openclaw status --all
   ```

   Diagnóstico de solo lectura con cola de logs (tokens redactados).

3. **Estado del daemon + puerto**

   ```bash
   openclaw gateway status
   ```

   Muestra el tiempo de ejecución del supervisor frente a la accesibilidad RPC, la URL objetivo de la sonda y qué configuración probablemente usó el servicio.

4. **Sondas profundas**

   ```bash
   openclaw status --deep
   ```

   Ejecuta una sonda de estado en vivo del gateway, incluidas sondas de canal cuando están disponibles
   (requiere un gateway accesible). Consulta [Health](/gateway/health).

5. **Seguir el log más reciente**

   ```bash
   openclaw logs --follow
   ```

   Si RPC no está disponible, recurre a:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Los logs de archivo están separados de los logs del servicio; consulta [Logging](/logging) y [Troubleshooting](/gateway/troubleshooting).

6. **Ejecutar doctor (reparaciones)**

   ```bash
   openclaw doctor
   ```

   Repara/migra configuración/estado + ejecuta comprobaciones de estado. Consulta [Doctor](/gateway/doctor).

7. **Instantánea del gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # muestra la URL objetivo + la ruta de configuración en los errores
   ```

   Pide al gateway en ejecución una instantánea completa (solo WS). Consulta [Health](/gateway/health).

## Inicio rápido y configuración inicial

<AccordionGroup>
  <Accordion title="Estoy atascado, ¿cuál es la forma más rápida de salir del atasco?">
    Usa un agente de IA local que pueda **ver tu máquina**. Eso es mucho más eficaz que preguntar
    en Discord, porque la mayoría de los casos de "estoy atascado" son **problemas locales de configuración o entorno** que
    los ayudantes remotos no pueden inspeccionar.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Estas herramientas pueden leer el repositorio, ejecutar comandos, inspeccionar logs y ayudar a arreglar
    la configuración de tu máquina (PATH, servicios, permisos, archivos de autenticación). Dales el **checkout completo del código fuente** mediante
    la instalación hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Esto instala OpenClaw **desde un checkout git**, para que el agente pueda leer el código y la documentación y
    razonar sobre la versión exacta que estás ejecutando. Siempre puedes volver a estable más adelante
    volviendo a ejecutar el instalador sin `--install-method git`.

    Consejo: pide al agente que **planifique y supervise** la corrección (paso a paso), y luego ejecute solo los
    comandos necesarios. Eso mantiene los cambios pequeños y más fáciles de auditar.

    Si descubres un error real o una corrección, por favor abre un issue en GitHub o envía una PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Empieza con estos comandos (comparte las salidas cuando pidas ayuda):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Qué hacen:

    - `openclaw status`: instantánea rápida del estado del gateway/agente + configuración básica.
    - `openclaw models status`: comprueba autenticación del proveedor + disponibilidad del modelo.
    - `openclaw doctor`: valida y repara problemas comunes de configuración/estado.

    Otras comprobaciones útiles de CLI: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Bucle rápido de depuración: [Los primeros 60 segundos si algo está roto](#los-primeros-60-segundos-si-algo-esta-roto).
    Documentación de instalación: [Install](/install), [Installer flags](/install/installer), [Updating](/install/updating).

  </Accordion>

  <Accordion title="Heartbeat sigue omitiendo ejecuciones. ¿Qué significan los motivos de omisión?">
    Motivos comunes de omisión de heartbeat:

    - `quiet-hours`: fuera de la ventana configurada de horas activas
    - `empty-heartbeat-file`: `HEARTBEAT.md` existe pero solo contiene estructura en blanco/solo encabezados
    - `no-tasks-due`: el modo de tareas de `HEARTBEAT.md` está activo pero todavía no vence ninguno de los intervalos de tareas
    - `alerts-disabled`: toda la visibilidad de heartbeat está deshabilitada (`showOk`, `showAlerts` y `useIndicator` están desactivados)

    En el modo de tareas, las marcas de tiempo de vencimiento solo se adelantan después de que se complete
    una ejecución real de heartbeat. Las ejecuciones omitidas no marcan las tareas como completadas.

    Documentación: [Heartbeat](/gateway/heartbeat), [Automation & Tasks](/automation).

  </Accordion>

  <Accordion title="Forma recomendada de instalar y configurar OpenClaw">
    El repositorio recomienda ejecutar desde el código fuente y usar onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    El asistente también puede compilar activos de UI automáticamente. Después de onboarding, normalmente ejecutas el Gateway en el puerto **18789**.

    Desde el código fuente (colaboradores/desarrollo):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # instala automáticamente las dependencias de UI en la primera ejecución
    openclaw onboard
    ```

    Si todavía no tienes una instalación global, ejecútalo mediante `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="¿Cómo abro el panel después de onboarding?">
    El asistente abre tu navegador con una URL limpia del panel (sin token) justo después del onboarding y también imprime el enlace en el resumen. Mantén esa pestaña abierta; si no se abrió, copia/pega la URL impresa en la misma máquina.
  </Accordion>

  <Accordion title="¿Cómo autentico el panel en localhost frente a remoto?">
    **Localhost (misma máquina):**

    - Abre `http://127.0.0.1:18789/`.
    - Si pide autenticación con secreto compartido, pega el token o la contraseña configurados en la configuración de Control UI.
    - Origen del token: `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
    - Origen de la contraseña: `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
    - Si todavía no hay ningún secreto compartido configurado, genera un token con `openclaw doctor --generate-gateway-token`.

    **No en localhost:**

    - **Tailscale Serve** (recomendado): mantén el bind en loopback, ejecuta `openclaw gateway --tailscale serve`, abre `https://<magicdns>/`. Si `gateway.auth.allowTailscale` es `true`, las cabeceras de identidad satisfacen la autenticación de Control UI/WebSocket (sin pegar secreto compartido, asume un host de gateway de confianza); las API HTTP siguen requiriendo autenticación con secreto compartido a menos que uses deliberadamente `none` con ingreso privado o autenticación HTTP `trusted-proxy`.
      Los intentos simultáneos incorrectos de autenticación Serve desde el mismo cliente se serializan antes de que el limitador de autenticación fallida los registre, por lo que el segundo reintento incorrecto ya puede mostrar `retry later`.
    - **Bind de tailnet**: ejecuta `openclaw gateway --bind tailnet --token "<token>"` (o configura autenticación por contraseña), abre `http://<tailscale-ip>:18789/`, y luego pega el secreto compartido correspondiente en la configuración del panel.
    - **Proxy inverso con reconocimiento de identidad**: mantén el Gateway detrás de un proxy de confianza no loopback, configura `gateway.auth.mode: "trusted-proxy"`, y luego abre la URL del proxy.
    - **Túnel SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@host` y luego abre `http://127.0.0.1:18789/`. La autenticación con secreto compartido sigue aplicándose a través del túnel; pega el token o la contraseña configurados si se te solicita.

    Consulta [Dashboard](/web/dashboard) y [Web surfaces](/web) para los detalles de modos de bind y autenticación.

  </Accordion>

  <Accordion title="¿Por qué hay dos configuraciones de aprobación de exec para aprobaciones en chat?">
    Controlan capas diferentes:

    - `approvals.exec`: reenvía solicitudes de aprobación a destinos de chat
    - `channels.<channel>.execApprovals`: hace que ese canal actúe como cliente nativo de aprobación para aprobaciones de exec

    La política de exec del host sigue siendo la puerta real de aprobación. La configuración del chat solo controla dónde aparecen las
    solicitudes de aprobación y cómo pueden responder las personas.

    En la mayoría de las configuraciones **no** necesitas ambas:

    - Si el chat ya admite comandos y respuestas, `/approve` en el mismo chat funciona mediante la ruta compartida.
    - Si un canal nativo compatible puede inferir aprobadores de forma segura, OpenClaw ahora habilita automáticamente aprobaciones nativas DM-first cuando `channels.<channel>.execApprovals.enabled` no está configurado o es `"auto"`.
    - Cuando hay tarjetas/botones de aprobación nativos disponibles, esa UI nativa es la ruta principal; el agente solo debería incluir un comando manual `/approve` si el resultado de la herramienta dice que las aprobaciones por chat no están disponibles o que la aprobación manual es la única ruta.
    - Usa `approvals.exec` solo cuando las solicitudes también deban reenviarse a otros chats o salas explícitas de operaciones.
    - Usa `channels.<channel>.execApprovals.target: "channel"` o `"both"` solo cuando quieras explícitamente que las solicitudes de aprobación se publiquen de vuelta en la sala/tema de origen.
    - Las aprobaciones de plugins son distintas otra vez: usan `/approve` en el mismo chat de forma predeterminada, `approvals.plugin` opcional para reenvío, y solo algunos canales nativos mantienen encima el manejo nativo de aprobación de plugins.

    En resumen: el reenvío es para enrutamiento, la configuración del cliente nativo es para una UX más rica y específica por canal.
    Consulta [Exec Approvals](/tools/exec-approvals).

  </Accordion>

  <Accordion title="¿Qué entorno de ejecución necesito?">
    Se requiere Node **>= 22**. Se recomienda `pnpm`. Bun **no se recomienda** para el Gateway.
  </Accordion>

  <Accordion title="¿Funciona en Raspberry Pi?">
    Sí. El Gateway es ligero: la documentación indica que **512 MB-1 GB de RAM**, **1 núcleo** y unos **500 MB**
    de disco son suficientes para uso personal, y señala que una **Raspberry Pi 4 puede ejecutarlo**.

    Si quieres más margen (logs, medios, otros servicios), se recomiendan **2 GB**, pero
    no es un mínimo estricto.

    Consejo: una Pi/VPS pequeña puede alojar el Gateway, y puedes emparejar **nodos** en tu portátil/teléfono para
    pantalla/cámara/canvas local o ejecución de comandos. Consulta [Nodes](/nodes).

  </Accordion>

  <Accordion title="¿Algún consejo para instalaciones en Raspberry Pi?">
    Resumen: funciona, pero espera algunas asperezas.

    - Usa un SO de **64 bits** y mantén Node >= 22.
    - Prefiere la **instalación hackable (git)** para que puedas ver logs y actualizar rápido.
    - Empieza sin canales/Skills, luego añádelos uno a uno.
    - Si encuentras problemas extraños con binarios, normalmente es un problema de **compatibilidad ARM**.

    Documentación: [Linux](/platforms/linux), [Install](/install).

  </Accordion>

  <Accordion title="Se queda atascado en wake up my friend / onboarding no termina de arrancar. ¿Y ahora qué?">
    Esa pantalla depende de que el Gateway sea accesible y esté autenticado. La TUI también envía
    "Wake up, my friend!" automáticamente en la primera activación. Si ves esa línea **sin respuesta**
    y los tokens siguen en 0, el agente nunca llegó a ejecutarse.

    1. Reinicia el Gateway:

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

    Si el Gateway es remoto, asegúrate de que la conexión de túnel/Tailscale esté activa y de que la UI
    apunte al Gateway correcto. Consulta [Remote access](/gateway/remote).

  </Accordion>

  <Accordion title="¿Puedo migrar mi configuración a una máquina nueva (Mac mini) sin rehacer onboarding?">
    Sí. Copia el **directorio de estado** y el **espacio de trabajo**, y luego ejecuta Doctor una vez. Esto
    mantiene tu bot "exactamente igual" (memoria, historial de sesiones, autenticación y
    estado del canal) siempre que copies **ambas** ubicaciones:

    1. Instala OpenClaw en la nueva máquina.
    2. Copia `$OPENCLAW_STATE_DIR` (predeterminado: `~/.openclaw`) desde la máquina antigua.
    3. Copia tu espacio de trabajo (predeterminado: `~/.openclaw/workspace`).
    4. Ejecuta `openclaw doctor` y reinicia el servicio Gateway.

    Eso conserva configuración, perfiles de autenticación, credenciales de WhatsApp, sesiones y memoria. Si estás en
    modo remoto, recuerda que el host del gateway es quien controla el almacén de sesiones y el espacio de trabajo.

    **Importante:** si solo haces commit/push de tu espacio de trabajo a GitHub, estás
    respaldando **memoria + archivos bootstrap**, pero **no** el historial de sesiones ni la autenticación. Esos viven
    bajo `~/.openclaw/` (por ejemplo `~/.openclaw/agents/<agentId>/sessions/`).

    Relacionado: [Migrating](/install/migrating), [Dónde vive todo en disco](#donde-vive-todo-en-disco),
    [Agent workspace](/concepts/agent-workspace), [Doctor](/gateway/doctor),
    [Remote mode](/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde veo qué hay de nuevo en la última versión?">
    Consulta el changelog de GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Las entradas más recientes están arriba. Si la sección superior está marcada como **Unreleased**, la siguiente
    sección con fecha es la última versión publicada. Las entradas se agrupan por **Highlights**, **Changes** y
    **Fixes** (además de secciones de documentación/u otras cuando hace falta).

  </Accordion>

  <Accordion title="No puedo acceder a docs.openclaw.ai (error SSL)">
    Algunas conexiones de Comcast/Xfinity bloquean incorrectamente `docs.openclaw.ai` mediante Xfinity
    Advanced Security. Desactívalo o añade `docs.openclaw.ai` a la lista de permitidos, y vuelve a intentarlo.
    Ayúdanos a desbloquearlo informando aquí: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Si todavía no puedes acceder al sitio, la documentación está replicada en GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Diferencia entre stable y beta">
    **Stable** y **beta** son **dist-tags de npm**, no líneas de código separadas:

    - `latest` = estable
    - `beta` = compilación temprana para pruebas

    Normalmente, una versión estable llega primero a **beta**, y luego un paso explícito
    de promoción mueve esa misma versión a `latest`. Los mantenedores también pueden
    publicar directamente en `latest` cuando sea necesario. Por eso beta y stable pueden
    apuntar a la **misma versión** después de la promoción.

    Consulta qué cambió:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Para one-liners de instalación y la diferencia entre beta y dev, consulta el acordeón inferior.

  </Accordion>

  <Accordion title="¿Cómo instalo la versión beta y cuál es la diferencia entre beta y dev?">
    **Beta** es el dist-tag de npm `beta` (puede coincidir con `latest` después de la promoción).
    **Dev** es la cabeza móvil de `main` (git); cuando se publica, usa el dist-tag de npm `dev`.

    One-liners (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Instalador para Windows (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Más detalle: [Development channels](/install/development-channels) y [Installer flags](/install/installer).

  </Accordion>

  <Accordion title="¿Cómo pruebo lo más reciente?">
    Dos opciones:

    1. **Canal dev (checkout git):**

    ```bash
    openclaw update --channel dev
    ```

    Esto cambia a la rama `main` y actualiza desde el código fuente.

    2. **Instalación hackable (desde el sitio del instalador):**

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

    Documentación: [Update](/cli/update), [Development channels](/install/development-channels),
    [Install](/install).

  </Accordion>

  <Accordion title="¿Cuánto tarda normalmente la instalación y el onboarding?">
    Guía aproximada:

    - **Instalación:** 2-5 minutos
    - **Onboarding:** 5-15 minutos dependiendo de cuántos canales/modelos configures

    Si se cuelga, usa [Instalador atascado](#quick-start-and-first-run-setup)
    y el bucle rápido de depuración de [Estoy atascado](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="¿Instalador atascado? ¿Cómo obtengo más información?">
    Vuelve a ejecutar el instalador con **salida detallada**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Instalación beta con verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Para una instalación hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Equivalente en Windows (PowerShell):

    ```powershell
    # install.ps1 todavía no tiene un indicador -Verbose dedicado.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Más opciones: [Installer flags](/install/installer).

  </Accordion>

  <Accordion title="La instalación en Windows dice git not found u openclaw not recognized">
    Dos problemas comunes en Windows:

    **1) Error de npm spawn git / git not found**

    - Instala **Git for Windows** y asegúrate de que `git` esté en tu PATH.
    - Cierra y vuelve a abrir PowerShell, luego vuelve a ejecutar el instalador.

    **2) openclaw is not recognized después de instalar**

    - Tu carpeta bin global de npm no está en PATH.
    - Comprueba la ruta:

      ```powershell
      npm config get prefix
      ```

    - Añade ese directorio a tu PATH de usuario (no hace falta el sufijo `\bin` en Windows; en la mayoría de los sistemas es `%AppData%\npm`).
    - Cierra y vuelve a abrir PowerShell después de actualizar PATH.

    Si quieres la configuración más fluida en Windows, usa **WSL2** en lugar de Windows nativo.
    Documentación: [Windows](/platforms/windows).

  </Accordion>

  <Accordion title="La salida de exec en Windows muestra texto chino corrupto. ¿Qué debo hacer?">
    Normalmente esto es una discrepancia en la página de códigos de consola en shells nativos de Windows.

    Síntomas:

    - la salida de `system.run`/`exec` muestra chino como mojibake
    - el mismo comando se ve bien en otro perfil de terminal

    Solución rápida en PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Luego reinicia el Gateway y vuelve a probar tu comando:

    ```powershell
    openclaw gateway restart
    ```

    Si sigues reproduciendo esto en la última versión de OpenClaw, haz seguimiento/repórtalo en:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="La documentación no respondió mi pregunta. ¿Cómo consigo una respuesta mejor?">
    Usa la **instalación hackable (git)** para tener el código fuente y la documentación completos localmente, y luego pregunta
    a tu bot (o a Claude/Codex) _desde esa carpeta_ para que pueda leer el repositorio y responder con precisión.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Más detalle: [Install](/install) y [Installer flags](/install/installer).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en Linux?">
    Respuesta corta: sigue la guía de Linux y luego ejecuta onboarding.

    - Ruta rápida para Linux + instalación del servicio: [Linux](/platforms/linux).
    - Guía completa: [Getting Started](/es/start/getting-started).
    - Instalador + actualizaciones: [Install & updates](/install/updating).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en un VPS?">
    Cualquier VPS Linux sirve. Instala en el servidor y luego usa SSH/Tailscale para acceder al Gateway.

    Guías: [exe.dev](/install/exe-dev), [Hetzner](/install/hetzner), [Fly.io](/install/fly).
    Acceso remoto: [Gateway remote](/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde están las guías de instalación en la nube/VPS?">
    Mantenemos un **centro de alojamiento** con los proveedores más comunes. Elige uno y sigue la guía:

    - [VPS hosting](/vps) (todos los proveedores en un mismo lugar)
    - [Fly.io](/install/fly)
    - [Hetzner](/install/hetzner)
    - [exe.dev](/install/exe-dev)

    Cómo funciona en la nube: el **Gateway se ejecuta en el servidor**, y accedes a él
    desde tu portátil/teléfono a través de Control UI (o Tailscale/SSH). Tu estado y espacio de trabajo
    viven en el servidor, así que trata el host como la fuente de verdad y haz copias de seguridad.

    Puedes emparejar **nodos** (Mac/iOS/Android/sin cabeza) a ese Gateway en la nube para acceder a
    pantalla/cámara/canvas locales o ejecutar comandos en tu portátil mientras mantienes el
    Gateway en la nube.

    Centro: [Platforms](/platforms). Acceso remoto: [Gateway remote](/gateway/remote).
    Nodos: [Nodes](/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="¿Puedo pedirle a OpenClaw que se actualice a sí mismo?">
    Respuesta corta: **posible, no recomendado**. El flujo de actualización puede reiniciar el
    Gateway (lo que corta la sesión activa), puede necesitar un checkout git limpio y
    puede pedir confirmación. Más seguro: ejecutar las actualizaciones desde un shell como operador.

    Usa la CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Si debes automatizarlo desde un agente:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Documentación: [Update](/cli/update), [Updating](/install/updating).

  </Accordion>

  <Accordion title="¿Qué hace realmente onboarding?">
    `openclaw onboard` es la ruta de configuración recomendada. En **modo local** te guía por:

    - **Configuración de modelo/autenticación** (OAuth del proveedor, reutilización de Claude CLI y claves de API, además de opciones de modelos locales como LM Studio)
    - Ubicación del **espacio de trabajo** + archivos bootstrap
    - **Ajustes del Gateway** (bind/puerto/autenticación/tailscale)
    - **Canales** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, además de plugins de canal incluidos como QQ Bot)
    - **Instalación del daemon** (LaunchAgent en macOS; unidad de usuario systemd en Linux/WSL2)
    - **Comprobaciones de estado** y selección de **Skills**

    También avisa si el modelo configurado es desconocido o carece de autenticación.

  </Accordion>

  <Accordion title="¿Necesito una suscripción de Claude o OpenAI para ejecutar esto?">
    No. Puedes ejecutar OpenClaw con **claves de API** (Anthropic/OpenAI/otros) o con
    **modelos solo locales** para que tus datos permanezcan en tu dispositivo. Las suscripciones (Claude
    Pro/Max u OpenAI Codex) son formas opcionales de autenticar esos proveedores.

    Creemos que el fallback de Claude Code CLI probablemente está permitido para automatización local
    gestionada por el usuario, según la documentación pública de Anthropic sobre su CLI. Aun así,
    la política de Anthropic sobre harnesses de terceros genera suficiente ambigüedad en torno al
    uso respaldado por suscripción en productos externos como para que no lo recomendemos
    para producción. Además, Anthropic notificó a los usuarios de OpenClaw el **4 de abril de 2026
    a las 12:00 PM PT / 8:00 PM BST** que la ruta de inicio de sesión de Claude de **OpenClaw**
    cuenta como uso mediante harness de terceros y ahora requiere **Extra Usage**
    facturado por separado de la suscripción. OpenAI Codex OAuth sí es compatible explícitamente
    para herramientas externas como OpenClaw.

    OpenClaw también admite otras opciones alojadas de tipo suscripción, incluidas
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan** y
    **Z.AI / GLM Coding Plan**.

    Documentación: [Anthropic](/providers/anthropic), [OpenAI](/providers/openai),
    [Qwen Cloud](/providers/qwen),
    [MiniMax](/providers/minimax), [GLM Models](/providers/glm),
    [Local models](/gateway/local-models), [Models](/concepts/models).

  </Accordion>

  <Accordion title="¿Puedo usar la suscripción Claude Max sin una clave de API?">
    Sí, mediante un inicio de sesión local de **Claude CLI** en el host del gateway.

    Las suscripciones Claude Pro/Max **no incluyen una clave de API**, así que la
    reutilización de Claude CLI es la ruta local de respaldo en OpenClaw. Creemos que el fallback de Claude Code CLI
    probablemente está permitido para automatización local gestionada por el usuario según
    la documentación pública de Anthropic sobre su CLI. Aun así, la política de Anthropic sobre harnesses de terceros
    genera suficiente ambigüedad en torno al uso respaldado por suscripción en productos externos
    como para que no lo recomendemos para producción. Recomendamos
    en su lugar claves de API de Anthropic.

  </Accordion>

  <Accordion title="¿Admitís autenticación con suscripción de Claude (Claude Pro o Max)?">
    Sí. Reutiliza un inicio de sesión local de **Claude CLI** en el host del gateway con `openclaw models auth login --provider anthropic --method cli --set-default`.

    Anthropic setup-token también vuelve a estar disponible como ruta heredada/manual específica de OpenClaw. El aviso de facturación de Anthropic específico de OpenClaw sigue aplicándose allí, así que úsalo asumiendo que Anthropic requiere **Extra Usage**. Consulta [Anthropic](/providers/anthropic) y [OAuth](/concepts/oauth).

    Importante: creemos que el fallback de Claude Code CLI probablemente está permitido para automatización local
    gestionada por el usuario según la documentación pública de Anthropic sobre su CLI. Aun así,
    la política de Anthropic sobre harnesses de terceros genera suficiente ambigüedad en torno al
    uso respaldado por suscripción en productos externos como para que no lo recomendemos
    para producción. Anthropic también informó a los usuarios de OpenClaw el **4 de abril de 2026 a
    las 12:00 PM PT / 8:00 PM BST** de que la ruta de inicio de sesión de Claude de **OpenClaw**
    requiere **Extra Usage** facturado por separado de la suscripción.

    Para producción o cargas multiusuario, la autenticación mediante clave de API de Anthropic es la
    opción más segura y recomendada. Si quieres otras opciones alojadas de tipo suscripción
    en OpenClaw, consulta [OpenAI](/providers/openai), [Qwen / Model
    Cloud](/providers/qwen), [MiniMax](/providers/minimax) y
    [GLM Models](/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="¿Por qué veo HTTP 429 rate_limit_error de Anthropic?">
Eso significa que tu **cuota/límite de tasa de Anthropic** está agotado para la ventana actual. Si
usas **Claude CLI**, espera a que se restablezca la ventana o mejora tu plan. Si
usas una **clave de API de Anthropic**, comprueba la consola de Anthropic
para ver uso/facturación y aumenta los límites según sea necesario.

    Si el mensaje es específicamente:
    `Extra usage is required for long context requests`, la solicitud intenta usar
    la beta de contexto de 1M de Anthropic (`context1m: true`). Eso solo funciona cuando tu
    credencial puede usar facturación de contexto largo (facturación por clave de API o la
    ruta de inicio de sesión Claude de OpenClaw con Extra Usage habilitado).

    Consejo: configura un **modelo de respaldo** para que OpenClaw pueda seguir respondiendo mientras un proveedor esté limitado por tasa.
    Consulta [Models](/cli/models), [OAuth](/concepts/oauth) y
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="¿Es compatible AWS Bedrock?">
    Sí. OpenClaw tiene un proveedor incluido de **Amazon Bedrock (Converse)**. Cuando hay marcadores de entorno AWS presentes, OpenClaw puede descubrir automáticamente el catálogo Bedrock de texto/streaming y fusionarlo como proveedor implícito `amazon-bedrock`; en caso contrario puedes habilitar explícitamente `plugins.entries.amazon-bedrock.config.discovery.enabled` o añadir una entrada de proveedor manual. Consulta [Amazon Bedrock](/providers/bedrock) y [Model providers](/providers/models). Si prefieres un flujo gestionado con clave, un proxy compatible con OpenAI delante de Bedrock sigue siendo una opción válida.
  </Accordion>

  <Accordion title="¿Cómo funciona la autenticación de Codex?">
    OpenClaw admite **OpenAI Code (Codex)** mediante OAuth (inicio de sesión con ChatGPT). Onboarding puede ejecutar el flujo OAuth y establecerá el modelo predeterminado en `openai-codex/gpt-5.4` cuando corresponda. Consulta [Model providers](/concepts/model-providers) y [Onboarding (CLI)](/es/start/wizard).
  </Accordion>

  <Accordion title="¿Admitís autenticación por suscripción de OpenAI (Codex OAuth)?">
    Sí. OpenClaw admite completamente **OpenAI Code (Codex) subscription OAuth**.
    OpenAI permite explícitamente el uso de OAuth por suscripción en herramientas/flujos de trabajo externos
    como OpenClaw. Onboarding puede ejecutar el flujo OAuth por ti.

    Consulta [OAuth](/concepts/oauth), [Model providers](/concepts/model-providers) y [Onboarding (CLI)](/es/start/wizard).

  </Accordion>

  <Accordion title="¿Cómo configuro Gemini CLI OAuth?">
    Gemini CLI usa un **flujo de autenticación de plugin**, no un client id o secret en `openclaw.json`.

    Pasos:

    1. Instala Gemini CLI localmente para que `gemini` esté en `PATH`
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Habilita el plugin: `openclaw plugins enable google`
    3. Inicia sesión: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Modelo predeterminado tras iniciar sesión: `google-gemini-cli/gemini-3.1-pro-preview`
    5. Si las solicitudes fallan, configura `GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host del gateway

    Esto almacena tokens OAuth en perfiles de autenticación en el host del gateway. Detalles: [Model providers](/concepts/model-providers).

  </Accordion>

  <Accordion title="¿Un modelo local sirve para chats informales?">
    Normalmente no. OpenClaw necesita contexto amplio + seguridad fuerte; las tarjetas pequeñas truncan y filtran. Si insistes, ejecuta la **compilación de modelo más grande** que puedas localmente (LM Studio) y consulta [/gateway/local-models](/gateway/local-models). Los modelos más pequeños/cuantizados aumentan el riesgo de inyección de prompts; consulta [Security](/gateway/security).
  </Accordion>

  <Accordion title="¿Cómo mantengo el tráfico de modelos alojados en una región concreta?">
    Elige endpoints fijados por región. OpenRouter expone opciones alojadas en EE. UU. para MiniMax, Kimi y GLM; elige la variante alojada en EE. UU. para mantener los datos en esa región. Aun así puedes listar Anthropic/OpenAI junto con estos usando `models.mode: "merge"` para que los respaldos sigan disponibles respetando el proveedor regional que selecciones.
  </Accordion>

  <Accordion title="¿Tengo que comprar un Mac Mini para instalar esto?">
    No. OpenClaw funciona en macOS o Linux (Windows mediante WSL2). Un Mac mini es opcional; algunas personas
    compran uno como host siempre encendido, pero también sirve una VPS pequeña, un servidor doméstico o una máquina tipo Raspberry Pi.

    Solo necesitas un Mac **para herramientas exclusivas de macOS**. Para iMessage, usa [BlueBubbles](/channels/bluebubbles) (recomendado): el servidor de BlueBubbles se ejecuta en cualquier Mac, y el Gateway puede ejecutarse en Linux o en otro sitio. Si quieres otras herramientas exclusivas de macOS, ejecuta el Gateway en un Mac o empareja un nodo macOS.

    Documentación: [BlueBubbles](/channels/bluebubbles), [Nodes](/nodes), [Mac remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="¿Necesito un Mac mini para compatibilidad con iMessage?">
    Necesitas **algún dispositivo macOS** con sesión iniciada en Messages. **No** tiene que ser un Mac mini;
    cualquier Mac sirve. **Usa [BlueBubbles](/channels/bluebubbles)** (recomendado) para iMessage: el servidor de BlueBubbles se ejecuta en macOS, mientras que el Gateway puede ejecutarse en Linux o en otro sitio.

    Configuraciones habituales:

    - Ejecutar el Gateway en Linux/VPS y ejecutar el servidor BlueBubbles en cualquier Mac con sesión iniciada en Messages.
    - Ejecutarlo todo en el Mac si quieres la configuración de una sola máquina más simple.

    Documentación: [BlueBubbles](/channels/bluebubbles), [Nodes](/nodes),
    [Mac remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="Si compro un Mac mini para ejecutar OpenClaw, ¿puedo conectarlo a mi MacBook Pro?">
    Sí. El **Mac mini puede ejecutar el Gateway**, y tu MacBook Pro puede conectarse como
    **nodo** (dispositivo complementario). Los nodos no ejecutan el Gateway: proporcionan
    capacidades adicionales como pantalla/cámara/canvas y `system.run` en ese dispositivo.

    Patrón habitual:

    - Gateway en el Mac mini (siempre encendido).
    - El MacBook Pro ejecuta la app de macOS o un host de nodo y se empareja con el Gateway.
    - Usa `openclaw nodes status` / `openclaw nodes list` para verlo.

    Documentación: [Nodes](/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="¿Puedo usar Bun?">
    Bun **no se recomienda**. Vemos errores de tiempo de ejecución, especialmente con WhatsApp y Telegram.
    Usa **Node** para gateways estables.

    Si aun así quieres experimentar con Bun, hazlo en un gateway no productivo
    sin WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: ¿qué va en allowFrom?">
    `channels.telegram.allowFrom` es el **ID numérico del usuario humano remitente de Telegram**. No es el nombre de usuario del bot.

    Onboarding acepta entrada `@username` y la resuelve a un ID numérico, pero la autorización de OpenClaw usa solo IDs numéricos.

    Más seguro (sin bot de terceros):

    - Envía un DM a tu bot y luego ejecuta `openclaw logs --follow` y lee `from.id`.

    Bot API oficial:

    - Envía un DM a tu bot y luego llama a `https://api.telegram.org/bot<bot_token>/getUpdates` y lee `message.from.id`.

    Terceros (menos privado):

    - Envía un DM a `@userinfobot` o `@getidsbot`.

    Consulta [/channels/telegram](/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="¿Pueden varias personas usar un solo número de WhatsApp con distintas instancias de OpenClaw?">
    Sí, mediante **enrutamiento multiagente**. Vincula el **DM** de WhatsApp de cada remitente (peer `kind: "direct"`, remitente E.164 como `+15551234567`) a un `agentId` distinto, para que cada persona tenga su propio espacio de trabajo y almacén de sesiones. Las respuestas seguirán saliendo de la **misma cuenta de WhatsApp**, y el control de acceso DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) es global por cuenta de WhatsApp. Consulta [Multi-Agent Routing](/concepts/multi-agent) y [WhatsApp](/channels/whatsapp).
  </Accordion>

  <Accordion title='¿Puedo tener un agente de "chat rápido" y otro de "Opus para programar"?'>
    Sí. Usa enrutamiento multiagente: da a cada agente su propio modelo predeterminado, luego vincula rutas entrantes (cuenta del proveedor o peers concretos) a cada agente. Hay una configuración de ejemplo en [Multi-Agent Routing](/concepts/multi-agent). Consulta también [Models](/concepts/models) y [Configuration](/gateway/configuration).
  </Accordion>

  <Accordion title="¿Homebrew funciona en Linux?">
    Sí. Homebrew es compatible con Linux (Linuxbrew). Configuración rápida:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Si ejecutas OpenClaw mediante systemd, asegúrate de que el PATH del servicio incluya `/home/linuxbrew/.linuxbrew/bin` (o tu prefijo brew) para que las herramientas instaladas con `brew` se resuelvan en shells sin login.
    Las compilaciones recientes también anteponen directorios comunes bin de usuario en servicios systemd de Linux (por ejemplo `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) y respetan `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` y `FNM_DIR` cuando están configurados.

  </Accordion>

  <Accordion title="Diferencia entre la instalación git hackable y npm install">
    - **Instalación hackable (git):** checkout completo del código fuente, editable, ideal para colaboradores.
      Ejecutas compilaciones localmente y puedes modificar código/documentación.
    - **npm install:** instalación global de CLI, sin repositorio, ideal para "simplemente ejecutarlo".
      Las actualizaciones vienen de los dist-tags de npm.

    Documentación: [Getting started](/es/start/getting-started), [Updating](/install/updating).

  </Accordion>

  <Accordion title="¿Puedo cambiar entre instalaciones npm y git más adelante?">
    Sí. Instala la otra variante y luego ejecuta Doctor para que el servicio gateway apunte al nuevo entrypoint.
    Esto **no borra tus datos**; solo cambia la instalación del código de OpenClaw. Tu estado
    (`~/.openclaw`) y tu espacio de trabajo (`~/.openclaw/workspace`) permanecen intactos.

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

    Doctor detecta un desajuste en el entrypoint del servicio gateway y ofrece reescribir la configuración del servicio para que coincida con la instalación actual (usa `--repair` en automatización).

    Consejos de copia de seguridad: consulta [Estrategia de copia de seguridad](#donde-vive-todo-en-disco).

  </Accordion>

  <Accordion title="¿Debería ejecutar el Gateway en mi portátil o en una VPS?">
    Respuesta corta: **si quieres fiabilidad 24/7, usa una VPS**. Si quieres la
    menor fricción y aceptas suspensiones/reinicios, ejecútalo localmente.

    **Portátil (Gateway local)**

    - **Ventajas:** sin coste de servidor, acceso directo a archivos locales, ventana del navegador visible.
    - **Inconvenientes:** suspensión/cortes de red = desconexiones, actualizaciones/reinicios del SO interrumpen, debe permanecer despierto.

    **VPS / nube**

    - **Ventajas:** siempre encendido, red estable, sin problemas de suspensión del portátil, más fácil de mantener en funcionamiento.
    - **Inconvenientes:** suele ejecutarse sin interfaz gráfica (usa capturas), acceso remoto solo a archivos, debes usar SSH para actualizar.

    **Nota específica de OpenClaw:** WhatsApp/Telegram/Slack/Mattermost/Discord funcionan bien desde una VPS. La única compensación real es **navegador sin cabeza** frente a una ventana visible. Consulta [Browser](/tools/browser).

    **Predeterminado recomendado:** VPS si ya tuviste desconexiones del gateway antes. Local es ideal cuando estás usando activamente el Mac y quieres acceso a archivos locales o automatización de UI con un navegador visible.

  </Accordion>

  <Accordion title="¿Qué tan importante es ejecutar OpenClaw en una máquina dedicada?">
    No es obligatorio, pero sí **recomendable para fiabilidad y aislamiento**.

    - **Host dedicado (VPS/Mac mini/Pi):** siempre encendido, menos interrupciones por suspensión/reinicio, permisos más limpios, más fácil de mantener en funcionamiento.
    - **Portátil/escritorio compartido:** perfectamente válido para pruebas y uso activo, pero espera pausas cuando la máquina entre en suspensión o se actualice.

    Si quieres lo mejor de ambos mundos, mantén el Gateway en un host dedicado y empareja tu portátil como **nodo** para herramientas locales de pantalla/cámara/exec. Consulta [Nodes](/nodes).
    Para directrices de seguridad, lee [Security](/gateway/security).

  </Accordion>

  <Accordion title="¿Cuáles son los requisitos mínimos de una VPS y el SO recomendado?">
    OpenClaw es ligero. Para un Gateway básico + un canal de chat:

    - **Mínimo absoluto:** 1 vCPU, 1 GB de RAM, ~500 MB de disco.
    - **Recomendado:** 1-2 vCPU, 2 GB de RAM o más para margen (logs, medios, varios canales). Las herramientas de nodo y la automatización de navegador pueden consumir muchos recursos.

    SO: usa **Ubuntu LTS** (o cualquier Debian/Ubuntu moderno). Esa es la ruta de instalación de Linux mejor probada.

    Documentación: [Linux](/platforms/linux), [VPS hosting](/vps).

  </Accordion>

  <Accordion title="¿Puedo ejecutar OpenClaw en una VM y cuáles son los requisitos?">
    Sí. Trata una VM igual que una VPS: debe estar siempre encendida, accesible y tener suficiente
    RAM para el Gateway y cualquier canal que habilites.

    Guía base:

    - **Mínimo absoluto:** 1 vCPU, 1 GB de RAM.
    - **Recomendado:** 2 GB de RAM o más si ejecutas varios canales, automatización de navegador o herramientas de medios.
    - **SO:** Ubuntu LTS u otro Debian/Ubuntu moderno.

    Si estás en Windows, **WSL2 es la configuración tipo VM más sencilla** y la que tiene mejor
    compatibilidad de herramientas. Consulta [Windows](/platforms/windows), [VPS hosting](/vps).
    Si estás ejecutando macOS en una VM, consulta [macOS VM](/install/macos-vm).

  </Accordion>
</AccordionGroup>

## ¿Qué es OpenClaw?

<AccordionGroup>
  <Accordion title="¿Qué es OpenClaw en un párrafo?">
    OpenClaw es un asistente de IA personal que ejecutas en tus propios dispositivos. Responde en las superficies de mensajería que ya usas (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat y plugins de canal incluidos como QQ Bot) y también puede hacer voz + un Canvas en vivo en plataformas compatibles. El **Gateway** es el plano de control siempre encendido; el asistente es el producto.
  </Accordion>

  <Accordion title="Propuesta de valor">
    OpenClaw no es "solo un wrapper de Claude". Es un **plano de control local-first** que te permite ejecutar un
    asistente potente en **tu propio hardware**, accesible desde las aplicaciones de chat que ya usas, con
    sesiones con estado, memoria y herramientas, sin entregar el control de tus flujos de trabajo a un
    SaaS alojado.

    Aspectos destacados:

    - **Tus dispositivos, tus datos:** ejecuta el Gateway donde quieras (Mac, Linux, VPS) y mantén el
      espacio de trabajo + historial de sesiones en local.
    - **Canales reales, no un sandbox web:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc.,
      además de voz móvil y Canvas en plataformas compatibles.
    - **Independiente del modelo:** usa Anthropic, OpenAI, MiniMax, OpenRouter, etc., con enrutamiento
      por agente y conmutación por error.
    - **Opción solo local:** ejecuta modelos locales para que **todos los datos puedan permanecer en tu dispositivo** si quieres.
    - **Enrutamiento multiagente:** agentes separados por canal, cuenta o tarea, cada uno con su propio
      espacio de trabajo y valores predeterminados.
    - **Código abierto y hackable:** inspecciona, amplía y autoalójalo sin dependencia de un proveedor.

    Documentación: [Gateway](/gateway), [Channels](/channels), [Multi-agent](/concepts/multi-agent),
    [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="Acabo de configurarlo. ¿Qué debería hacer primero?">
    Buenos primeros proyectos:

    - Crear un sitio web (WordPress, Shopify o un sitio estático simple).
    - Prototipar una aplicación móvil (esquema, pantallas, plan de API).
    - Organizar archivos y carpetas (limpieza, nombres, etiquetas).
    - Conectar Gmail y automatizar resúmenes o seguimientos.

    Puede manejar tareas grandes, pero funciona mejor si las divides en fases y
    usas subagentes para trabajo en paralelo.

  </Accordion>

  <Accordion title="¿Cuáles son los cinco casos de uso cotidianos principales de OpenClaw?">
    Las victorias cotidianas suelen verse así:

    - **Resúmenes personales:** resúmenes de bandeja de entrada, calendario y noticias que te importan.
    - **Investigación y redacción:** investigación rápida, resúmenes y primeros borradores para correos o documentos.
    - **Recordatorios y seguimientos:** avisos y listas de comprobación impulsados por cron o heartbeat.
    - **Automatización del navegador:** rellenar formularios, recopilar datos y repetir tareas web.
    - **Coordinación entre dispositivos:** envía una tarea desde tu teléfono, deja que el Gateway la ejecute en un servidor y recibe el resultado de vuelta en el chat.

  </Accordion>

  <Accordion title="¿Puede OpenClaw ayudar con lead gen, outreach, anuncios y blogs para un SaaS?">
    Sí, para **investigación, cualificación y redacción**. Puede escanear sitios, crear listas cortas,
    resumir prospectos y redactar borradores de outreach o copy de anuncios.

    Para **outreach o campañas de anuncios**, mantén a una persona dentro del circuito. Evita el spam, cumple la legislación local y
    las políticas de la plataforma, y revisa todo antes de enviarlo. El patrón más seguro es dejar que
    OpenClaw redacte y que tú apruebes.

    Documentación: [Security](/gateway/security).

  </Accordion>

  <Accordion title="¿Qué ventajas tiene frente a Claude Code para desarrollo web?">
    OpenClaw es un **asistente personal** y una capa de coordinación, no un sustituto del IDE. Usa
    Claude Code o Codex para el bucle de programación más rápido directamente dentro de un repositorio. Usa OpenClaw cuando
    quieras memoria duradera, acceso entre dispositivos y orquestación de herramientas.

    Ventajas:

    - **Memoria persistente + espacio de trabajo** entre sesiones
    - **Acceso multiplataforma** (WhatsApp, Telegram, TUI, WebChat)
    - **Orquestación de herramientas** (navegador, archivos, programación, hooks)
    - **Gateway siempre encendido** (ejecútalo en una VPS, interactúa desde cualquier lugar)
    - **Nodos** para navegador/pantalla/cámara/exec local

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills y automatización

<AccordionGroup>
  <Accordion title="¿Cómo personalizo Skills sin mantener el repositorio sucio?">
    Usa anulaciones gestionadas en lugar de editar la copia del repositorio. Pon tus cambios en `~/.openclaw/skills/<name>/SKILL.md` (o añade una carpeta mediante `skills.load.extraDirs` en `~/.openclaw/openclaw.json`). La precedencia es `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → incluidos → `skills.load.extraDirs`, así que las anulaciones gestionadas siguen prevaleciendo sobre los Skills incluidos sin tocar git. Si necesitas que el skill esté instalado globalmente pero solo visible para algunos agentes, mantén la copia compartida en `~/.openclaw/skills` y controla la visibilidad con `agents.defaults.skills` y `agents.list[].skills`. Solo las ediciones dignas de upstream deberían vivir en el repositorio y salir como PR.
  </Accordion>

  <Accordion title="¿Puedo cargar Skills desde una carpeta personalizada?">
    Sí. Añade directorios extra mediante `skills.load.extraDirs` en `~/.openclaw/openclaw.json` (precedencia más baja). La precedencia predeterminada es `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → incluidos → `skills.load.extraDirs`. `clawhub` instala en `./skills` por defecto, que OpenClaw trata como `<workspace>/skills` en la siguiente sesión. Si el skill solo debería ser visible para ciertos agentes, combínalo con `agents.defaults.skills` o `agents.list[].skills`.
  </Accordion>

  <Accordion title="¿Cómo puedo usar distintos modelos para distintas tareas?">
    Hoy los patrones compatibles son:

    - **Trabajos cron**: los trabajos aislados pueden establecer una anulación `model` por trabajo.
    - **Subagentes**: enruta tareas a agentes separados con distintos modelos predeterminados.
    - **Cambio bajo demanda**: usa `/model` para cambiar el modelo de la sesión actual en cualquier momento.

    Consulta [Cron jobs](/automation/cron-jobs), [Multi-Agent Routing](/concepts/multi-agent) y [Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title="El bot se congela mientras hace trabajo pesado. ¿Cómo lo descargo?">
    Usa **subagentes** para tareas largas o en paralelo. Los subagentes se ejecutan en su propia sesión,
    devuelven un resumen y mantienen tu chat principal receptivo.

    Pide a tu bot que "spawn a sub-agent for this task" o usa `/subagents`.
    Usa `/status` en el chat para ver qué está haciendo el Gateway ahora mismo (y si está ocupado).

    Consejo sobre tokens: tanto las tareas largas como los subagentes consumen tokens. Si el coste es una preocupación, configura un
    modelo más barato para subagentes mediante `agents.defaults.subagents.model`.

    Documentación: [Sub-agents](/tools/subagents), [Background Tasks](/automation/tasks).

  </Accordion>

  <Accordion title="¿Cómo funcionan las sesiones de subagente vinculadas a hilos en Discord?">
    Usa vinculaciones de hilos. Puedes vincular un hilo de Discord a un subagente o destino de sesión para que los mensajes de seguimiento en ese hilo se mantengan en esa sesión vinculada.

    Flujo básico:

    - Genera con `sessions_spawn` usando `thread: true` (y opcionalmente `mode: "session"` para seguimiento persistente).
    - O vincula manualmente con `/focus <target>`.
    - Usa `/agents` para inspeccionar el estado de la vinculación.
    - Usa `/session idle <duration|off>` y `/session max-age <duration|off>` para controlar el auto-unfocus.
    - Usa `/unfocus` para desvincular el hilo.

    Configuración necesaria:

    - Valores predeterminados globales: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Anulaciones de Discord: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Vinculación automática al generar: configura `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Documentación: [Sub-agents](/tools/subagents), [Discord](/channels/discord), [Configuration Reference](/gateway/configuration-reference), [Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title="Un subagente terminó, pero la actualización de finalización fue al lugar equivocado o nunca se publicó. ¿Qué debo revisar?">
    Comprueba primero la ruta resuelta del solicitante:

    - La entrega de subagentes en modo de finalización prefiere cualquier hilo vinculado o ruta de conversación cuando existe.
    - Si el origen de finalización solo incluye un canal, OpenClaw recurre a la ruta almacenada de la sesión solicitante (`lastChannel` / `lastTo` / `lastAccountId`) para que la entrega directa pueda seguir funcionando.
    - Si no existe ni una ruta vinculada ni una ruta almacenada utilizable, la entrega directa puede fallar y el resultado recurre a entrega en cola de sesión en lugar de publicarse inmediatamente en el chat.
    - Los destinos inválidos u obsoletos aún pueden forzar el respaldo en cola o el fallo de entrega final.
    - Si la última respuesta visible del asistente del hijo es exactamente el token silencioso `NO_REPLY` / `no_reply`, o exactamente `ANNOUNCE_SKIP`, OpenClaw suprime intencionadamente el anuncio en lugar de publicar progreso anterior obsoleto.
    - Si el hijo agotó el tiempo después de solo llamadas a herramientas, el anuncio puede colapsar eso en un breve resumen de progreso parcial en lugar de reproducir la salida sin procesar de la herramienta.

    Depuración:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Sub-agents](/tools/subagents), [Background Tasks](/automation/tasks), [Session Tools](/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron o recordatorios no se ejecutan. ¿Qué debería revisar?">
    Cron se ejecuta dentro del proceso Gateway. Si el Gateway no está en ejecución continua,
    los trabajos programados no se ejecutarán.

    Lista de comprobación:

    - Confirma que cron está habilitado (`cron.enabled`) y que `OPENCLAW_SKIP_CRON` no está configurado.
    - Comprueba que el Gateway se esté ejecutando 24/7 (sin suspensión/reinicios).
    - Verifica la configuración de zona horaria del trabajo (`--tz` frente a la zona horaria del host).

    Depuración:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentación: [Cron jobs](/automation/cron-jobs), [Automation & Tasks](/automation).

  </Accordion>

  <Accordion title="Cron se ejecutó, pero no se envió nada al canal. ¿Por qué?">
    Comprueba primero el modo de entrega:

    - `--no-deliver` / `delivery.mode: "none"` significa que no se espera ningún mensaje externo.
    - Falta o es inválido el destino de anuncio (`channel` / `to`) significa que el ejecutor omitió la entrega saliente.
    - Fallos de autenticación del canal (`unauthorized`, `Forbidden`) significan que el ejecutor intentó entregar, pero las credenciales lo bloquearon.
    - Un resultado aislado silencioso (`NO_REPLY` / `no_reply` solamente) se trata como intencionadamente no entregable, por lo que el ejecutor también suprime la entrega de respaldo en cola.

    Para trabajos cron aislados, el ejecutor controla la entrega final. Se espera
    que el agente devuelva un resumen en texto plano para que el ejecutor lo envíe. `--no-deliver` mantiene
    ese resultado interno; no permite que el agente envíe directamente con la
    herramienta de mensajes.

    Depuración:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Cron jobs](/automation/cron-jobs), [Background Tasks](/automation/tasks).

  </Accordion>

  <Accordion title="¿Por qué una ejecución cron aislada cambió de modelo o reintentó una vez?">
    Normalmente esa es la ruta de cambio de modelo en vivo, no una programación duplicada.

    Cron aislado puede conservar un traspaso de modelo en tiempo de ejecución y reintentar cuando la
    ejecución activa lanza `LiveSessionModelSwitchError`. El reintento mantiene el
    proveedor/modelo cambiado, y si el cambio incluía una nueva anulación de perfil de autenticación, cron
    también la conserva antes de reintentar.

    Reglas de selección relacionadas:

    - La anulación de modelo del hook de Gmail gana primero cuando aplica.
    - Luego `model` por trabajo.
    - Luego cualquier anulación de modelo almacenada de sesión cron.
    - Luego la selección normal de modelo predeterminado/agente.

    El bucle de reintento es acotado. Después del intento inicial más 2 reintentos de cambio,
    cron aborta en lugar de entrar en un bucle infinito.

    Depuración:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentación: [Cron jobs](/automation/cron-jobs), [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="¿Cómo instalo Skills en Linux?">
    Usa los comandos nativos `openclaw skills` o deja los Skills en tu espacio de trabajo. La UI de Skills de macOS no está disponible en Linux.
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
    del espacio de trabajo activo. Instala la CLI separada `clawhub` solo si quieres publicar o
    sincronizar tus propios Skills. Para instalaciones compartidas entre agentes, pon el skill en
    `~/.openclaw/skills` y usa `agents.defaults.skills` o
    `agents.list[].skills` si quieres limitar qué agentes pueden verlo.

  </Accordion>

  <Accordion title="¿Puede OpenClaw ejecutar tareas con una programación o continuamente en segundo plano?">
    Sí. Usa el programador del Gateway:

    - **Trabajos cron** para tareas programadas o recurrentes (persisten tras reinicios).
    - **Heartbeat** para comprobaciones periódicas de la "sesión principal".
    - **Trabajos aislados** para agentes autónomos que publican resúmenes o entregan a chats.

    Documentación: [Cron jobs](/automation/cron-jobs), [Automation & Tasks](/automation),
    [Heartbeat](/gateway/heartbeat).

  </Accordion>

  <Accordion title="¿Puedo ejecutar Skills exclusivos de macOS desde Linux?">
    No directamente. Los Skills de macOS están controlados por `metadata.openclaw.os` más los binarios requeridos, y los Skills solo aparecen en el prompt del sistema cuando son elegibles en el **host del Gateway**. En Linux, los Skills solo-`darwin` (como `apple-notes`, `apple-reminders`, `things-mac`) no se cargarán salvo que anules ese control.

    Tienes tres patrones compatibles:

    **Opción A: ejecutar el Gateway en un Mac (más simple).**
    Ejecuta el Gateway donde existan los binarios de macOS y luego conéctate desde Linux en [modo remoto](#gateway-ports-already-running-and-remote-mode) o sobre Tailscale. Los Skills se cargan normalmente porque el host del Gateway es macOS.

    **Opción B: usar un nodo macOS (sin SSH).**
    Ejecuta el Gateway en Linux, empareja un nodo macOS (app de barra de menús) y configura **Node Run Commands** en "Always Ask" o "Always Allow" en el Mac. OpenClaw puede tratar los Skills exclusivos de macOS como elegibles cuando los binarios requeridos existan en el nodo. El agente ejecuta esos Skills a través de la herramienta `nodes`. Si eliges "Always Ask", aprobar "Always Allow" en el aviso añade ese comando a la lista de permitidos.

    **Opción C: hacer proxy de binarios macOS sobre SSH (avanzado).**
    Mantén el Gateway en Linux, pero haz que los binarios CLI requeridos se resuelvan a wrappers SSH que se ejecuten en un Mac. Luego anula el skill para permitir Linux y que siga siendo elegible.

    1. Crea un wrapper SSH para el binario (ejemplo: `memo` para Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Pon el wrapper en `PATH` en el host Linux (por ejemplo `~/bin/memo`).
    3. Anula los metadatos del skill (espacio de trabajo o `~/.openclaw/skills`) para permitir Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Inicia una sesión nueva para que se actualice la instantánea de Skills.

  </Accordion>

  <Accordion title="¿Tenéis una integración con Notion o HeyGen?">
    No integrada actualmente.

    Opciones:

    - **Skill / plugin personalizado:** lo mejor para un acceso fiable a la API (Notion y HeyGen tienen APIs).
    - **Automatización del navegador:** funciona sin código, pero es más lenta y frágil.

    Si quieres mantener contexto por cliente (flujos de agencia), un patrón simple es:

    - Una página de Notion por cliente (contexto + preferencias + trabajo activo).
    - Pedir al agente que recupere esa página al inicio de una sesión.

    Si quieres una integración nativa, abre una solicitud de funcionalidad o crea un skill
    dirigido a esas APIs.

    Instalar Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Las instalaciones nativas van al directorio `skills/` del espacio de trabajo activo. Para Skills compartidos entre agentes, colócalos en `~/.openclaw/skills/<name>/SKILL.md`. Si solo algunos agentes deberían ver una instalación compartida, configura `agents.defaults.skills` o `agents.list[].skills`. Algunos Skills esperan binarios instalados mediante Homebrew; en Linux eso significa Linuxbrew (consulta la entrada de FAQ de Homebrew en Linux anterior). Consulta [Skills](/tools/skills), [Skills config](/tools/skills-config) y [ClawHub](/tools/clawhub).

  </Accordion>

  <Accordion title="¿Cómo uso mi Chrome ya autenticado con OpenClaw?">
    Usa el perfil de navegador integrado `user`, que se conecta a través de Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Si quieres un nombre personalizado, crea un perfil MCP explícito:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Esta ruta es local al host. Si el Gateway se ejecuta en otro lugar, ejecuta un host de nodo en la máquina del navegador o usa CDP remoto.

    Límites actuales de `existing-session` / `user`:

    - las acciones usan ref, no selectores CSS
    - las cargas requieren `ref` / `inputRef` y actualmente solo admiten un archivo a la vez
    - `responsebody`, exportación PDF, interceptación de descargas y acciones por lotes siguen necesitando un navegador gestionado o un perfil CDP sin procesar

  </Accordion>
</AccordionGroup>

## Sandboxing y memoria

<AccordionGroup>
  <Accordion title="¿Hay una documentación dedicada para sandboxing?">
    Sí. Consulta [Sandboxing](/gateway/sandboxing). Para configuración específica de Docker (gateway completo en Docker o imágenes de sandbox), consulta [Docker](/install/docker).
  </Accordion>

  <Accordion title="Docker parece limitado. ¿Cómo habilito funciones completas?">
    La imagen predeterminada prioriza la seguridad y se ejecuta como el usuario `node`, por lo que no
    incluye paquetes del sistema, Homebrew ni navegadores incluidos. Para una configuración más completa:

    - Haz persistente `/home/node` con `OPENCLAW_HOME_VOLUME` para que sobrevivan las cachés.
    - Incorpora dependencias del sistema en la imagen con `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Instala navegadores Playwright mediante la CLI incluida:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Configura `PLAYWRIGHT_BROWSERS_PATH` y asegúrate de que la ruta sea persistente.

    Documentación: [Docker](/install/docker), [Browser](/tools/browser).

  </Accordion>

  <Accordion title="¿Puedo mantener los DMs personales pero hacer públicos/en sandbox los grupos con un solo agente?">
    Sí, si tu tráfico privado son **DMs** y tu tráfico público son **grupos**.

    Usa `agents.defaults.sandbox.mode: "non-main"` para que las sesiones de grupo/canal (claves no principales) se ejecuten en Docker, mientras la sesión DM principal permanece en el host. Luego restringe qué herramientas están disponibles en sesiones en sandbox mediante `tools.sandbox.tools`.

    Guía de configuración + ejemplo: [Groups: personal DMs + public groups](/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Referencia de configuración clave: [Gateway configuration](/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="¿Cómo vinculo una carpeta del host al sandbox?">
    Configura `agents.defaults.sandbox.docker.binds` como `["host:path:mode"]` (por ejemplo, `"/home/user/src:/src:ro"`). Los binds globales y por agente se fusionan; los binds por agente se ignoran cuando `scope: "shared"`. Usa `:ro` para cualquier cosa sensible y recuerda que los binds eluden los límites del sistema de archivos del sandbox.

    OpenClaw valida los orígenes de los binds tanto contra la ruta normalizada como contra la ruta canónica resuelta a través del ancestro existente más profundo. Eso significa que los escapes mediante padres simbólicos siguen fallando de forma segura incluso cuando el último segmento de la ruta todavía no existe, y las comprobaciones de raíz permitida siguen aplicándose después de la resolución de enlaces simbólicos.

    Consulta [Sandboxing](/gateway/sandboxing#custom-bind-mounts) y [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) para ver ejemplos y notas de seguridad.

  </Accordion>

  <Accordion title="¿Cómo funciona la memoria?">
    La memoria de OpenClaw son simplemente archivos Markdown en el espacio de trabajo del agente:

    - Notas diarias en `memory/YYYY-MM-DD.md`
    - Notas de largo plazo seleccionadas en `MEMORY.md` (solo sesiones principales/privadas)

    OpenClaw también ejecuta un **vaciado silencioso de memoria antes de la compactación** para recordar al modelo
    que escriba notas duraderas antes de la compactación automática. Esto solo se ejecuta cuando el espacio de trabajo
    se puede escribir (los sandboxes de solo lectura lo omiten). Consulta [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="La memoria sigue olvidando cosas. ¿Cómo hago que se fijen?">
    Pide al bot que **escriba el dato en la memoria**. Las notas de largo plazo van en `MEMORY.md`,
    el contexto de corto plazo va en `memory/YYYY-MM-DD.md`.

    Esta sigue siendo un área que estamos mejorando. Ayuda recordar al modelo que almacene memorias;
    sabrá qué hacer. Si sigue olvidando, verifica que el Gateway esté usando el mismo
    espacio de trabajo en cada ejecución.

    Documentación: [Memory](/concepts/memory), [Agent workspace](/concepts/agent-workspace).

  </Accordion>

  <Accordion title="¿La memoria persiste para siempre? ¿Cuáles son los límites?">
    Los archivos de memoria viven en disco y persisten hasta que los elimines. El límite es tu
    almacenamiento, no el modelo. El **contexto de la sesión** sigue estando limitado por la ventana
    de contexto del modelo, por lo que las conversaciones largas pueden compactarse o truncarse. Por eso
    existe la búsqueda en memoria: vuelve a traer al contexto solo las partes relevantes.

    Documentación: [Memory](/concepts/memory), [Context](/concepts/context).

  </Accordion>

  <Accordion title="¿La búsqueda semántica en memoria requiere una clave de API de OpenAI?">
    Solo si usas **embeddings de OpenAI**. Codex OAuth cubre chat/completions y
    **no** concede acceso a embeddings, así que **iniciar sesión con Codex (OAuth o el
    inicio de sesión de Codex CLI)** no ayuda para la búsqueda semántica en memoria. Los embeddings de OpenAI
    siguen necesitando una clave de API real (`OPENAI_API_KEY` o `models.providers.openai.apiKey`).

    Si no configuras un proveedor explícitamente, OpenClaw selecciona automáticamente uno cuando
    puede resolver una clave de API (perfiles de autenticación, `models.providers.*.apiKey` o variables de entorno).
    Prefiere OpenAI si se resuelve una clave de OpenAI; de lo contrario Gemini si se resuelve una clave de Gemini;
    luego Voyage; luego Mistral. Si no hay ninguna clave remota disponible, la búsqueda en memoria
    permanece desactivada hasta que la configures. Si tienes una ruta de modelo local
    configurada y presente, OpenClaw
    prefiere `local`. Ollama es compatible cuando configuras explícitamente
    `memorySearch.provider = "ollama"`.

    Si prefieres seguir en local, configura `memorySearch.provider = "local"` (y opcionalmente
    `memorySearch.fallback = "none"`). Si quieres embeddings de Gemini, configura
    `memorySearch.provider = "gemini"` y proporciona `GEMINI_API_KEY` (o
    `memorySearch.remote.apiKey`). Admitimos modelos de embeddings de **OpenAI, Gemini, Voyage, Mistral, Ollama o local**;
    consulta [Memory](/concepts/memory) para los detalles de configuración.

  </Accordion>
</AccordionGroup>

## Dónde vive todo en disco

<AccordionGroup>
  <Accordion title="¿Todos los datos usados con OpenClaw se guardan localmente?">
    No: **el estado de OpenClaw es local**, pero **los servicios externos siguen viendo lo que les envías**.

    - **Local por defecto:** las sesiones, archivos de memoria, configuración y espacio de trabajo viven en el host del Gateway
      (`~/.openclaw` + tu directorio de espacio de trabajo).
    - **Remoto por necesidad:** los mensajes que envías a proveedores de modelos (Anthropic/OpenAI/etc.) van a
      sus APIs, y las plataformas de chat (WhatsApp/Telegram/Slack/etc.) almacenan los datos de mensajes en
      sus servidores.
    - **Tú controlas la huella:** usar modelos locales mantiene los prompts en tu máquina, pero el tráfico del canal
      sigue pasando por los servidores del canal.

    Relacionado: [Agent workspace](/concepts/agent-workspace), [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="¿Dónde guarda OpenClaw sus datos?">
    Todo vive bajo `$OPENCLAW_STATE_DIR` (predeterminado: `~/.openclaw`):

    | Ruta                                                            | Propósito                                                          |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Configuración principal (JSON5)                                    |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Importación heredada de OAuth (copiada a perfiles de autenticación en el primer uso) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Perfiles de autenticación (OAuth, claves de API y opcionales `keyRef`/`tokenRef`) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Carga útil opcional de secretos respaldados por archivo para proveedores `file` SecretRef |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | Archivo heredado de compatibilidad (entradas estáticas `api_key` depuradas) |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | Estado del proveedor (por ejemplo `whatsapp/<accountId>/creds.json`) |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | Estado por agente (`agentDir` + sesiones)                         |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Historial y estado de conversaciones (por agente)                 |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Metadatos de sesión (por agente)                                  |

    Ruta heredada de agente único: `~/.openclaw/agent/*` (migrada por `openclaw doctor`).

    Tu **espacio de trabajo** (`workspace`) (AGENTS.md, archivos de memoria, Skills, etc.) es independiente y se configura mediante `agents.defaults.workspace` (predeterminado: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="¿Dónde deberían vivir AGENTS.md / SOUL.md / USER.md / MEMORY.md?">
    Estos archivos viven en el **espacio de trabajo del agente**, no en `~/.openclaw`.

    - **Espacio de trabajo (por agente)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (o el respaldo heredado `memory.md` cuando `MEMORY.md` no existe),
      `memory/YYYY-MM-DD.md`, opcionalmente `HEARTBEAT.md`.
    - **Directorio de estado (`~/.openclaw`)**: configuración, estado de canales/proveedores, perfiles de autenticación, sesiones, logs
      y Skills compartidos (`~/.openclaw/skills`).

    El espacio de trabajo predeterminado es `~/.openclaw/workspace`, configurable mediante:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Si el bot "olvida" después de un reinicio, confirma que el Gateway está usando el mismo
    espacio de trabajo en cada arranque (y recuerda: el modo remoto usa el espacio de trabajo del **host del gateway**,
    no el de tu portátil local).

    Consejo: si quieres un comportamiento o preferencia duraderos, pide al bot que **lo escriba en
    AGENTS.md o MEMORY.md** en lugar de depender del historial de chat.

    Consulta [Agent workspace](/concepts/agent-workspace) y [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="Estrategia de copia de seguridad recomendada">
    Pon tu **espacio de trabajo del agente** en un repositorio git **privado** y haz copia de seguridad en algún lugar
    privado (por ejemplo GitHub privado). Esto captura memoria + archivos AGENTS/SOUL/USER
    y te permite restaurar más adelante la "mente" del asistente.

    **No** hagas commit de nada bajo `~/.openclaw` (credenciales, sesiones, tokens o cargas útiles cifradas de secretos).
    Si necesitas una restauración completa, haz copia de seguridad tanto del espacio de trabajo como del directorio de estado
    por separado (consulta la pregunta sobre migración anterior).

    Documentación: [Agent workspace](/concepts/agent-workspace).

  </Accordion>

  <Accordion title="¿Cómo desinstalo completamente OpenClaw?">
    Consulta la guía dedicada: [Uninstall](/install/uninstall).
  </Accordion>

  <Accordion title="¿Pueden los agentes trabajar fuera del espacio de trabajo?">
    Sí. El espacio de trabajo es el **cwd predeterminado** y el ancla de memoria, no un sandbox estricto.
    Las rutas relativas se resuelven dentro del espacio de trabajo, pero las rutas absolutas pueden acceder a otras
    ubicaciones del host salvo que el sandboxing esté habilitado. Si necesitas aislamiento, usa
    [`agents.defaults.sandbox`](/gateway/sandboxing) o la configuración de sandbox por agente. Si quieres
    que un repositorio sea el directorio de trabajo predeterminado, apunta el `workspace`
    de ese agente a la raíz del repositorio. El repositorio de OpenClaw es solo código fuente; mantén el
    espacio de trabajo separado salvo que quieras intencionadamente que el agente trabaje dentro de él.

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
    El estado de sesión pertenece al **host del gateway**. Si estás en modo remoto, el almacén de sesiones que te importa está en la máquina remota, no en tu portátil local. Consulta [Session management](/concepts/session).
  </Accordion>
</AccordionGroup>

## Fundamentos de configuración

<AccordionGroup>
  <Accordion title="¿Qué formato tiene la configuración? ¿Dónde está?">
    OpenClaw lee una configuración opcional en **JSON5** desde `$OPENCLAW_CONFIG_PATH` (predeterminado: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Si el archivo no existe, usa valores predeterminados razonablemente seguros (incluido un espacio de trabajo predeterminado `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='Configuré gateway.bind: "lan" (o "tailnet") y ahora no escucha nada / la UI dice unauthorized'>
    Los binds no loopback **requieren una ruta válida de autenticación del gateway**. En la práctica eso significa:

    - autenticación con secreto compartido: token o contraseña
    - `gateway.auth.mode: "trusted-proxy"` detrás de un proxy inverso con reconocimiento de identidad correctamente configurado y no loopback

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

    - `gateway.remote.token` / `.password` **no** habilitan por sí mismos la autenticación del gateway local.
    - Las rutas de llamada locales pueden usar `gateway.remote.*` como respaldo solo cuando `gateway.auth.*` no está configurado.
    - Para autenticación por contraseña, configura `gateway.auth.mode: "password"` más `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`) en su lugar.
    - Si `gateway.auth.token` / `gateway.auth.password` está configurado explícitamente mediante SecretRef y no está resuelto, la resolución falla de forma segura (sin respaldo remoto que oculte el problema).
    - Las configuraciones de Control UI con secreto compartido se autentican mediante `connect.params.auth.token` o `connect.params.auth.password` (almacenados en la configuración de app/UI). Los modos con identidad, como Tailscale Serve o `trusted-proxy`, usan en su lugar cabeceras de solicitud. Evita poner secretos compartidos en URLs.
    - Con `gateway.auth.mode: "trusted-proxy"`, los proxies inversos loopback en el mismo host **tampoco** satisfacen la autenticación trusted-proxy. El proxy de confianza debe ser una fuente no loopback configurada.

  </Accordion>

  <Accordion title="¿Por qué ahora necesito un token en localhost?">
    OpenClaw aplica autenticación del gateway por defecto, incluso en loopback. En la ruta predeterminada normal eso significa autenticación por token: si no hay ninguna ruta explícita de autenticación configurada, el arranque del gateway resuelve a modo token y genera uno automáticamente, guardándolo en `gateway.auth.token`, por lo que **los clientes WS locales deben autenticarse**. Esto bloquea a otros procesos locales para que no llamen al Gateway.

    Si prefieres otra ruta de autenticación, puedes elegir explícitamente modo contraseña (o, para proxies inversos con reconocimiento de identidad no loopback, `trusted-proxy`). Si **de verdad** quieres loopback abierto, configura `gateway.auth.mode: "none"` explícitamente en tu configuración. Doctor puede generarte un token en cualquier momento: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="¿Tengo que reiniciar después de cambiar la configuración?">
    El Gateway vigila la configuración y admite recarga en caliente:

    - `gateway.reload.mode: "hybrid"` (predeterminado): aplica en caliente cambios seguros, reinicia para los críticos
    - también se admiten `hot`, `restart`, `off`

  </Accordion>

  <Accordion title="¿Cómo desactivo los eslóganes graciosos de la CLI?">
    Configura `cli.banner.taglineMode` en la configuración:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: oculta el texto del eslogan pero mantiene la línea del título/versión del banner.
    - `default`: usa `All your chats, one OpenClaw.` siempre.
    - `random`: eslóganes rotativos, graciosos o estacionales (comportamiento predeterminado).
    - Si no quieres ningún banner, configura la variable de entorno `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="¿Cómo habilito web search (y web fetch)?">
    `web_fetch` funciona sin clave de API. `web_search` depende del proveedor
    seleccionado:

    - Los proveedores respaldados por API como Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity y Tavily requieren su configuración normal de clave de API.
    - Ollama Web Search no usa clave, pero usa tu host de Ollama configurado y requiere `ollama signin`.
    - DuckDuckGo no usa clave, pero es una integración no oficial basada en HTML.
    - SearXNG no usa clave/puede ser autoalojado; configura `SEARXNG_BASE_URL` o `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Recomendado:** ejecuta `openclaw configure --section web` y elige un proveedor.
    Alternativas por variable de entorno:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` o `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, o `MINIMAX_API_KEY`
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

    La configuración específica del proveedor para búsqueda web ahora vive en `plugins.entries.<plugin>.config.webSearch.*`.
    Las rutas heredadas de proveedor `tools.web.search.*` siguen cargándose temporalmente por compatibilidad, pero no deberían usarse en configuraciones nuevas.
    La configuración de respaldo de web-fetch de Firecrawl vive en `plugins.entries.firecrawl.config.webFetch.*`.

    Notas:

    - Si usas listas de permitidos, añade `web_search`/`web_fetch`/`x_search` o `group:web`.
    - `web_fetch` está habilitado por defecto (salvo que se deshabilite explícitamente).
    - Si se omite `tools.web.fetch.provider`, OpenClaw detecta automáticamente el primer proveedor de respaldo de fetch preparado a partir de las credenciales disponibles. Actualmente el proveedor incluido es Firecrawl.
    - Los daemons leen variables de entorno desde `~/.openclaw/.env` (o el entorno del servicio).

    Documentación: [Web tools](/tools/web).

  </Accordion>

  <Accordion title="config.apply borró mi configuración. ¿Cómo recupero y evito esto?">
    `config.apply` reemplaza la **configuración completa**. Si envías un objeto parcial, todo
    lo demás se elimina.

    Recuperar:

    - Restaura desde una copia de seguridad (git o una copia de `~/.openclaw/openclaw.json`).
    - Si no tienes copia de seguridad, vuelve a ejecutar `openclaw doctor` y reconfigura canales/modelos.
    - Si esto fue inesperado, abre un bug e incluye tu última configuración conocida o cualquier copia de seguridad.
    - Un agente de programación local a menudo puede reconstruir una configuración funcional a partir de logs o historial.

    Evitarlo:

    - Usa `openclaw config set` para cambios pequeños.
    - Usa `openclaw configure` para ediciones interactivas.
    - Usa primero `config.schema.lookup` cuando no estés seguro de la ruta exacta o la forma del campo; devuelve un nodo de esquema superficial más resúmenes de hijos inmediatos para profundizar.
    - Usa `config.patch` para ediciones RPC parciales; reserva `config.apply` solo para reemplazo completo de configuración.
    - Si estás usando la herramienta `gateway` solo para el propietario dentro de una ejecución de agente, seguirá rechazando escrituras en `tools.exec.ask` / `tools.exec.security` (incluidos alias heredados `tools.bash.*` que se normalizan a las mismas rutas protegidas de exec).

    Documentación: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/gateway/doctor).

  </Accordion>

  <Accordion title="¿Cómo ejecuto un Gateway central con trabajadores especializados en varios dispositivos?">
    El patrón común es **un Gateway** (por ejemplo Raspberry Pi) más **nodos** y **agentes**:

    - **Gateway (central):** controla canales (Signal/WhatsApp), enrutamiento y sesiones.
    - **Nodos (dispositivos):** Macs/iOS/Android se conectan como periféricos y exponen herramientas locales (`system.run`, `canvas`, `camera`).
    - **Agentes (trabajadores):** cerebros/espacios de trabajo separados para roles especiales (por ejemplo "Hetzner ops", "Personal data").
    - **Subagentes:** generan trabajo en segundo plano desde un agente principal cuando quieres paralelismo.
    - **TUI:** conéctate al Gateway y cambia de agentes/sesiones.

    Documentación: [Nodes](/nodes), [Remote access](/gateway/remote), [Multi-Agent Routing](/concepts/multi-agent), [Sub-agents](/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="¿Puede el navegador de OpenClaw ejecutarse sin cabeza?">
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

    El valor predeterminado es `false` (con interfaz). El modo sin cabeza tiene más probabilidades de activar comprobaciones anti-bot en algunos sitios. Consulta [Browser](/tools/browser).

    El modo sin cabeza usa el **mismo motor Chromium** y funciona para la mayoría de las automatizaciones (formularios, clics, scraping, inicios de sesión). Las diferencias principales:

    - No hay ventana visible del navegador (usa capturas de pantalla si necesitas visuales).
    - Algunos sitios son más estrictos con la automatización en modo sin cabeza (CAPTCHAs, anti-bot).
      Por ejemplo, X/Twitter suele bloquear sesiones sin cabeza.

  </Accordion>

  <Accordion title="¿Cómo uso Brave para control del navegador?">
    Configura `browser.executablePath` apuntando a tu binario de Brave (o cualquier navegador basado en Chromium) y reinicia el Gateway.
    Consulta los ejemplos completos de configuración en [Browser](/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateways remotos y nodos

<AccordionGroup>
  <Accordion title="¿Cómo se propagan los comandos entre Telegram, el gateway y los nodos?">
    Los mensajes de Telegram los gestiona el **gateway**. El gateway ejecuta el agente y
    solo entonces llama a nodos sobre el **WebSocket del Gateway** cuando hace falta una herramienta de nodo:

    Telegram → Gateway → Agente → `node.*` → Nodo → Gateway → Telegram

    Los nodos no ven tráfico entrante del proveedor; solo reciben llamadas RPC de nodo.

  </Accordion>

  <Accordion title="¿Cómo puede mi agente acceder a mi ordenador si el Gateway está alojado remotamente?">
    Respuesta corta: **empareja tu ordenador como nodo**. El Gateway se ejecuta en otro lugar, pero puede
    llamar a herramientas `node.*` (pantalla, cámara, sistema) en tu máquina local mediante el WebSocket del Gateway.

    Configuración típica:

    1. Ejecuta el Gateway en el host siempre encendido (VPS/servidor doméstico).
    2. Pon el host del Gateway y tu ordenador en la misma tailnet.
    3. Asegúrate de que el WS del Gateway sea accesible (bind de tailnet o túnel SSH).
    4. Abre la app de macOS localmente y conéctate en modo **Remote over SSH** (o tailnet directo)
       para que pueda registrarse como nodo.
    5. Aprueba el nodo en el Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    No se requiere un puente TCP independiente; los nodos se conectan por el WebSocket del Gateway.

    Recordatorio de seguridad: emparejar un nodo macOS permite `system.run` en esa máquina. Empareja
    solo dispositivos en los que confíes y revisa [Security](/gateway/security).

    Documentación: [Nodes](/nodes), [Gateway protocol](/gateway/protocol), [macOS remote mode](/platforms/mac/remote), [Security](/gateway/security).

  </Accordion>

  <Accordion title="Tailscale está conectado, pero no recibo respuestas. ¿Y ahora qué?">
    Comprueba lo básico:

    - El Gateway está en ejecución: `openclaw gateway status`
    - Estado del Gateway: `openclaw status`
    - Estado del canal: `openclaw channels status`

    Luego verifica autenticación y enrutamiento:

    - Si usas Tailscale Serve, asegúrate de que `gateway.auth.allowTailscale` esté configurado correctamente.
    - Si te conectas mediante túnel SSH, confirma que el túnel local esté activo y apunte al puerto correcto.
    - Confirma que tus listas de permitidos (DM o grupo) incluyan tu cuenta.

    Documentación: [Tailscale](/gateway/tailscale), [Remote access](/gateway/remote), [Channels](/channels).

  </Accordion>

  <Accordion title="¿Pueden hablar entre sí dos instancias de OpenClaw (local + VPS)?">
    Sí. No hay un puente "bot a bot" integrado, pero puedes conectarlo de varias
    formas fiables:

    **La más simple:** usa un canal de chat normal al que ambos bots puedan acceder (Telegram/Slack/WhatsApp).
    Haz que el Bot A envíe un mensaje al Bot B, y luego deja que el Bot B responda como siempre.

    **Puente CLI (genérico):** ejecuta un script que llame al otro Gateway con
    `openclaw agent --message ... --deliver`, apuntando a un chat donde escuche
    el otro bot. Si uno de los bots está en una VPS remota, apunta tu CLI a ese Gateway remoto
    mediante SSH/Tailscale (consulta [Remote access](/gateway/remote)).

    Patrón de ejemplo (ejecutar desde una máquina que pueda alcanzar el Gateway objetivo):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Consejo: añade una barrera para que los dos bots no entren en un bucle infinito (solo mención, listas de permitidos del canal o una regla de "no responder a mensajes de bots").

    Documentación: [Remote access](/gateway/remote), [Agent CLI](/cli/agent), [Agent send](/tools/agent-send).

  </Accordion>

  <Accordion title="¿Necesito VPS separadas para varios agentes?">
    No. Un Gateway puede alojar varios agentes, cada uno con su propio espacio de trabajo, modelos predeterminados
    y enrutamiento. Esa es la configuración normal y es mucho más barata y sencilla que ejecutar
    una VPS por agente.

    Usa VPS separadas solo cuando necesites aislamiento estricto (límites de seguridad) o configuraciones
    muy diferentes que no quieras compartir. En caso contrario, mantén un Gateway y
    usa varios agentes o subagentes.

  </Accordion>

  <Accordion title="¿Tiene alguna ventaja usar un nodo en mi portátil personal en lugar de SSH desde una VPS?">
    Sí: los nodos son la forma de primera clase de llegar a tu portátil desde un Gateway remoto, y
    desbloquean más cosas que el acceso por shell. El Gateway se ejecuta en macOS/Linux (Windows vía WSL2) y es
    ligero (una VPS pequeña o una máquina tipo Raspberry Pi va bien; 4 GB de RAM son suficientes), así que una configuración
    habitual es un host siempre encendido más tu portátil como nodo.

    - **No requiere SSH entrante.** Los nodos se conectan hacia fuera al WebSocket del Gateway y usan emparejamiento de dispositivos.
    - **Controles de ejecución más seguros.** `system.run` está controlado por listas de permitidos/aprobaciones del nodo en ese portátil.
    - **Más herramientas de dispositivo.** Los nodos exponen `canvas`, `camera` y `screen` además de `system.run`.
    - **Automatización local del navegador.** Mantén el Gateway en una VPS, pero ejecuta Chrome localmente mediante un host de nodo en el portátil, o conéctate al Chrome local del host vía Chrome MCP.

    SSH está bien para acceso puntual por shell, pero los nodos son más sencillos para flujos de trabajo de agentes continuos y
    automatización de dispositivos.

    Documentación: [Nodes](/nodes), [Nodes CLI](/cli/nodes), [Browser](/tools/browser).

  </Accordion>

  <Accordion title="¿Los nodos ejecutan un servicio gateway?">
    No. Solo debe ejecutarse **un gateway** por host, salvo que ejecutes intencionadamente perfiles aislados (consulta [Multiple gateways](/gateway/multiple-gateways)). Los nodos son periféricos que se conectan
    al gateway (nodos iOS/Android, o "modo nodo" de macOS en la app de barra de menús). Para hosts de nodo sin cabeza
    y control por CLI, consulta [Node host CLI](/cli/node).

    Se requiere un reinicio completo para cambios en `gateway`, `discovery` y `canvasHost`.

  </Accordion>

  <Accordion title="¿Existe una forma API / RPC de aplicar configuración?">
    Sí.

    - `config.schema.lookup`: inspecciona un subárbol de configuración con su nodo de esquema superficial, pista de UI coincidente y resúmenes de hijos inmediatos antes de escribir
    - `config.get`: obtiene la instantánea actual + hash
    - `config.patch`: actualización parcial segura (preferida para la mayoría de las ediciones RPC)
    - `config.apply`: valida + reemplaza la configuración completa, luego reinicia
    - La herramienta de tiempo de ejecución `gateway`, solo para el propietario, sigue negándose a reescribir `tools.exec.ask` / `tools.exec.security`; los alias heredados `tools.bash.*` se normalizan a las mismas rutas protegidas de exec

  </Accordion>

  <Accordion title="Configuración mínima sensata para una primera instalación">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Esto establece tu espacio de trabajo y restringe quién puede activar el bot.

  </Accordion>

  <Accordion title="¿Cómo configuro Tailscale en una VPS y me conecto desde mi Mac?">
    Pasos mínimos:

    1. **Instalar + iniciar sesión en la VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Instalar + iniciar sesión en tu Mac**
       - Usa la app de Tailscale e inicia sesión en la misma tailnet.
    3. **Habilitar MagicDNS (recomendado)**
       - En la consola de administración de Tailscale, habilita MagicDNS para que la VPS tenga un nombre estable.
    4. **Usar el nombre de host tailnet**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - WS del Gateway: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Si quieres la Control UI sin SSH, usa Tailscale Serve en la VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Esto mantiene el gateway vinculado a loopback y expone HTTPS mediante Tailscale. Consulta [Tailscale](/gateway/tailscale).

  </Accordion>

  <Accordion title="¿Cómo conecto un nodo Mac a un Gateway remoto (Tailscale Serve)?">
    Serve expone la **Control UI + WS del Gateway**. Los nodos se conectan por el mismo endpoint WS del Gateway.

    Configuración recomendada:

    1. **Asegúrate de que la VPS y el Mac están en la misma tailnet**.
    2. **Usa la app de macOS en modo Remote** (el destino SSH puede ser el nombre de host de tailnet).
       La app tunelizará el puerto del Gateway y se conectará como nodo.
    3. **Aprueba el nodo** en el gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentación: [Gateway protocol](/gateway/protocol), [Discovery](/gateway/discovery), [macOS remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="¿Debería instalarlo en un segundo portátil o simplemente añadir un nodo?">
    Si solo necesitas **herramientas locales** (pantalla/cámara/exec) en el segundo portátil, añádelo como
    **nodo**. Eso mantiene un único Gateway y evita configuración duplicada. Las herramientas locales de nodo
    actualmente son solo para macOS, pero planeamos ampliarlas a otros sistemas operativos.

    Instala un segundo Gateway solo cuando necesites **aislamiento estricto** o dos bots totalmente separados.

    Documentación: [Nodes](/nodes), [Nodes CLI](/cli/nodes), [Multiple gateways](/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Variables de entorno y carga de .env

<AccordionGroup>
  <Accordion title="¿Cómo carga OpenClaw las variables de entorno?">
    OpenClaw lee variables de entorno del proceso padre (shell, launchd/systemd, CI, etc.) y además carga:

    - `.env` desde el directorio de trabajo actual
    - un `.env` global de respaldo desde `~/.openclaw/.env` (también conocido como `$OPENCLAW_STATE_DIR/.env`)

    Ningún archivo `.env` sobrescribe variables de entorno existentes.

    También puedes definir variables de entorno en línea en la configuración (se aplican solo si faltan en el entorno del proceso):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Consulta [/environment](/help/environment) para ver la precedencia y orígenes completos.

  </Accordion>

  <Accordion title="Inicié el Gateway mediante el servicio y mis variables de entorno desaparecieron. ¿Y ahora?">
    Dos soluciones habituales:

    1. Pon las claves que faltan en `~/.openclaw/.env` para que se recojan incluso cuando el servicio no herede el entorno de tu shell.
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

    Esto ejecuta tu shell de login e importa solo las claves esperadas que falten (nunca sobrescribe). Equivalentes por variable de entorno:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Configuré COPILOT_GITHUB_TOKEN, pero models status muestra "Shell env: off." ¿Por qué?'>
    `openclaw models status` informa si la **importación del entorno del shell** está habilitada. "Shell env: off"
    **no** significa que falten tus variables de entorno; solo significa que OpenClaw no cargará
    automáticamente tu shell de login.

    Si el Gateway se ejecuta como servicio (launchd/systemd), no heredará tu entorno
    del shell. Solución: haz una de estas cosas:

    1. Pon el token en `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. O habilita la importación del shell (`env.shellEnv.enabled: true`).
    3. O añádelo a tu bloque `env` de configuración (se aplica solo si falta).

    Luego reinicia el gateway y vuelve a comprobar:

    ```bash
    openclaw models status
    ```

    Los tokens de Copilot se leen desde `COPILOT_GITHUB_TOKEN` (también `GH_TOKEN` / `GITHUB_TOKEN`).
    Consulta [/concepts/model-providers](/concepts/model-providers) y [/environment](/help/environment).

  </Accordion>
</AccordionGroup>

## Sesiones y varios chats

<AccordionGroup>
  <Accordion title="¿Cómo inicio una conversación nueva?">
    Envía `/new` o `/reset` como mensaje independiente. Consulta [Session management](/concepts/session).
  </Accordion>

  <Accordion title="¿Las sesiones se reinician automáticamente si nunca envío /new?">
    Las sesiones pueden expirar tras `session.idleMinutes`, pero esto está **deshabilitado por defecto** (valor predeterminado **0**).
    Configúralo con un valor positivo para habilitar la expiración por inactividad. Cuando está habilitado, el **siguiente**
    mensaje después del periodo de inactividad inicia un id de sesión nuevo para esa clave de chat.
    Esto no elimina transcripciones; solo inicia una nueva sesión.

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
    y varios agentes trabajadores con sus propios espacios de trabajo y modelos.

    Aun así, esto se entiende mejor como un **experimento divertido**. Consume muchos tokens y a menudo
    es menos eficiente que usar un solo bot con sesiones separadas. El modelo típico que
    imaginamos es un bot con el que hablas, con distintas sesiones para trabajo en paralelo. Ese
    bot también puede generar subagentes cuando sea necesario.

    Documentación: [Multi-agent routing](/concepts/multi-agent), [Sub-agents](/tools/subagents), [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="¿Por qué se truncó el contexto en mitad de una tarea? ¿Cómo lo evito?">
    El contexto de la sesión está limitado por la ventana del modelo. Chats largos, salidas grandes de herramientas o muchos
    archivos pueden activar compactación o truncado.

    Qué ayuda:

    - Pide al bot que resuma el estado actual y lo escriba en un archivo.
    - Usa `/compact` antes de tareas largas y `/new` al cambiar de tema.
    - Mantén el contexto importante en el espacio de trabajo y pide al bot que lo vuelva a leer.
    - Usa subagentes para trabajo largo o en paralelo para que el chat principal siga siendo más pequeño.
    - Elige un modelo con una ventana de contexto más grande si esto sucede con frecuencia.

  </Accordion>

  <Accordion title="¿Cómo restablezco completamente OpenClaw pero lo mantengo instalado?">
    Usa el comando reset:

    ```bash
    openclaw reset
    ```

    Restablecimiento completo no interactivo:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Luego vuelve a ejecutar la configuración:

    ```bash
    openclaw onboard --install-daemon
    ```

    Notas:

    - Onboarding también ofrece **Reset** si detecta una configuración existente. Consulta [Onboarding (CLI)](/es/start/wizard).
    - Si usaste perfiles (`--profile` / `OPENCLAW_PROFILE`), restablece cada directorio de estado (por defecto son `~/.openclaw-<profile>`).
    - Restablecimiento de desarrollo: `openclaw gateway --dev --reset` (solo desarrollo; borra configuración dev + credenciales + sesiones + espacio de trabajo).

  </Accordion>

  <Accordion title='Estoy recibiendo errores de "context too large". ¿Cómo reinicio o compacto?'>
    Usa una de estas opciones:

    - **Compactar** (mantiene la conversación pero resume turnos antiguos):

      ```
      /compact
      ```

      o `/compact <instructions>` para guiar el resumen.

    - **Restablecer** (id de sesión nuevo para la misma clave de chat):

      ```
      /new
      /reset
      ```

    Si sigue ocurriendo:

    - Habilita o ajusta la **poda de sesión** (`agents.defaults.contextPruning`) para recortar la salida antigua de herramientas.
    - Usa un modelo con una ventana de contexto más grande.

    Documentación: [Compaction](/concepts/compaction), [Session pruning](/concepts/session-pruning), [Session management](/concepts/session).

  </Accordion>

  <Accordion title='¿Por qué veo "LLM request rejected: messages.content.tool_use.input field required"?'>
    Esto es un error de validación del proveedor: el modelo emitió un bloque `tool_use` sin el
    `input` requerido. Suele significar que el historial de la sesión está obsoleto o corrupto (a menudo tras hilos largos
    o un cambio en la herramienta/esquema).

    Solución: inicia una sesión nueva con `/new` (mensaje independiente).

  </Accordion>

  <Accordion title="¿Por qué recibo mensajes de heartbeat cada 30 minutos?">
    Los heartbeats se ejecutan cada **30m** por defecto (**1h** cuando se usa autenticación OAuth). Ajústalos o desactívalos:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // o "0m" para desactivar
          },
        },
      },
    }
    ```

    Si `HEARTBEAT.md` existe pero está efectivamente vacío (solo líneas en blanco y
    encabezados Markdown como `# Heading`), OpenClaw omite la ejecución de heartbeat para ahorrar llamadas a la API.
    Si el archivo no existe, el heartbeat sigue ejecutándose y el modelo decide qué hacer.

    Las anulaciones por agente usan `agents.list[].heartbeat`. Documentación: [Heartbeat](/gateway/heartbeat).

  </Accordion>

  <Accordion title='¿Necesito añadir una "cuenta de bot" a un grupo de WhatsApp?'>
    No. OpenClaw se ejecuta en **tu propia cuenta**, así que si estás en el grupo, OpenClaw puede verlo.
    Por defecto, las respuestas de grupo se bloquean hasta que permitas remitentes (`groupPolicy: "allowlist"`).

    Si quieres que solo **tú** puedas activar respuestas de grupo:

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
    Opción 1 (la más rápida): sigue los logs y envía un mensaje de prueba en el grupo:

    ```bash
    openclaw logs --follow --json
    ```

    Busca `chatId` (o `from`) terminado en `@g.us`, como:
    `1234567890-1234567890@g.us`.

    Opción 2 (si ya está configurado/en la lista de permitidos): lista grupos desde la configuración:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentación: [WhatsApp](/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="¿Por qué OpenClaw no responde en un grupo?">
    Dos causas comunes:

    - El control por mención está activado (predeterminado). Debes mencionar al bot con @ (o coincidir con `mentionPatterns`).
    - Configuraste `channels.whatsapp.groups` sin `"*"` y el grupo no está en la lista de permitidos.

    Consulta [Groups](/channels/groups) y [Group messages](/channels/group-messages).

  </Accordion>

  <Accordion title="¿Los grupos/hilos comparten contexto con los DMs?">
    Los chats directos colapsan en la sesión principal por defecto. Los grupos/canales tienen sus propias claves de sesión, y los temas de Telegram / hilos de Discord son sesiones separadas. Consulta [Groups](/channels/groups) y [Group messages](/channels/group-messages).
  </Accordion>

  <Accordion title="¿Cuántos espacios de trabajo y agentes puedo crear?">
    No hay límites estrictos. Decenas (incluso cientos) van bien, pero vigila:

    - **Crecimiento de disco:** las sesiones y transcripciones viven bajo `~/.openclaw/agents/<agentId>/sessions/`.
    - **Coste de tokens:** más agentes significa más uso concurrente del modelo.
    - **Sobrecarga operativa:** perfiles de autenticación, espacios de trabajo y enrutamiento de canales por agente.

    Consejos:

    - Mantén un espacio de trabajo **activo** por agente (`agents.defaults.workspace`).
    - Poda sesiones antiguas (elimina JSONL o entradas de almacén) si el disco crece.
    - Usa `openclaw doctor` para detectar espacios de trabajo sueltos y desajustes de perfiles.

  </Accordion>

  <Accordion title="¿Puedo ejecutar varios bots o chats al mismo tiempo (Slack), y cómo debería configurarlo?">
    Sí. Usa **Multi-Agent Routing** para ejecutar varios agentes aislados y enrutar mensajes entrantes por
    canal/cuenta/peer. Slack es compatible como canal y puede vincularse a agentes concretos.

    El acceso al navegador es potente, pero no es "hacer cualquier cosa que haría un humano"; los anti-bot, CAPTCHAs y MFA
    todavía pueden bloquear la automatización. Para el control del navegador más fiable, usa Chrome MCP local en el host,
    o usa CDP en la máquina que realmente ejecuta el navegador.

    Configuración recomendada:

    - Host Gateway siempre encendido (VPS/Mac mini).
    - Un agente por rol (bindings).
    - Canal(es) de Slack vinculados a esos agentes.
    - Navegador local mediante Chrome MCP o un nodo cuando sea necesario.

    Documentación: [Multi-Agent Routing](/concepts/multi-agent), [Slack](/channels/slack),
    [Browser](/tools/browser), [Nodes](/nodes).

  </Accordion>
</AccordionGroup>

## Modelos: valores predeterminados, selección, alias, cambio

<AccordionGroup>
  <Accordion title='¿Qué es el "modelo predeterminado"?'>
    El modelo predeterminado de OpenClaw es el que configures como:

    ```
    agents.defaults.model.primary
    ```

    Los modelos se referencian como `provider/model` (ejemplo: `openai/gpt-5.4`). Si omites el proveedor, OpenClaw primero intenta un alias, luego una coincidencia única de proveedor configurado para ese model id exacto, y solo después recurre al proveedor predeterminado configurado como una ruta heredada y obsoleta. Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw recurre al primer proveedor/modelo configurado en lugar de mostrar un valor predeterminado obsoleto de un proveedor eliminado. Aun así, deberías configurar **explícitamente** `provider/model`.

  </Accordion>

  <Accordion title="¿Qué modelo recomendáis?">
    **Predeterminado recomendado:** usa el mejor modelo de última generación disponible en tu pila de proveedores.
    **Para agentes con herramientas o entrada no confiable:** prioriza la potencia del modelo sobre el coste.
    **Para chat rutinario/de bajo riesgo:** usa modelos de respaldo más baratos y enrútalos por rol del agente.

    MiniMax tiene su propia documentación: [MiniMax](/providers/minimax) y
    [Local models](/gateway/local-models).

    Regla general: usa el **mejor modelo que puedas pagar** para trabajo de alto riesgo, y un modelo más barato
    para chat rutinario o resúmenes. Puedes enrutar modelos por agente y usar subagentes para
    paralelizar tareas largas (cada subagente consume tokens). Consulta [Models](/concepts/models) y
    [Sub-agents](/tools/subagents).

    Advertencia importante: los modelos más débiles o demasiado cuantizados son más vulnerables a la inyección de prompts
    y a comportamientos inseguros. Consulta [Security](/gateway/security).

    Más contexto: [Models](/concepts/models).

  </Accordion>

  <Accordion title="¿Cómo cambio de modelo sin borrar mi configuración?">
    Usa **comandos de modelo** o edita solo los campos de **model**. Evita reemplazos completos de configuración.

    Opciones seguras:

    - `/model` en el chat (rápido, por sesión)
    - `openclaw models set ...` (actualiza solo la configuración del modelo)
    - `openclaw configure --section model` (interactivo)
    - edita `agents.defaults.model` en `~/.openclaw/openclaw.json`

    Evita `config.apply` con un objeto parcial salvo que realmente quieras reemplazar toda la configuración.
    Para ediciones RPC, inspecciona primero con `config.schema.lookup` y prefiere `config.patch`. La carga útil de lookup te da la ruta normalizada, la documentación/restricciones superficiales del esquema y resúmenes de hijos inmediatos
    para actualizaciones parciales.
    Si sobrescribiste la configuración, restaura desde una copia de seguridad o vuelve a ejecutar `openclaw doctor` para repararla.

    Documentación: [Models](/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/gateway/doctor).

  </Accordion>

  <Accordion title="¿Puedo usar modelos autoalojados (llama.cpp, vLLM, Ollama)?">
    Sí. Ollama es la ruta más sencilla para modelos locales.

    Configuración más rápida:

    1. Instala Ollama desde `https://ollama.com/download`
    2. Descarga un modelo local, por ejemplo `ollama pull glm-4.7-flash`
    3. Si también quieres modelos en la nube, ejecuta `ollama signin`
    4. Ejecuta `openclaw onboard` y elige `Ollama`
    5. Elige `Local` o `Cloud + Local`

    Notas:

    - `Cloud + Local` te da modelos en la nube además de tus modelos locales de Ollama
    - los modelos en la nube como `kimi-k2.5:cloud` no necesitan descarga local
    - para cambiar manualmente, usa `openclaw models list` y `openclaw models set ollama/<model>`

    Nota de seguridad: los modelos más pequeños o muy cuantizados son más vulnerables a la inyección de prompts.
    Recomendamos encarecidamente **modelos grandes** para cualquier bot que pueda usar herramientas.
    Si aun así quieres modelos pequeños, habilita sandboxing y listas estrictas de herramientas permitidas.

    Documentación: [Ollama](/providers/ollama), [Local models](/gateway/local-models),
    [Model providers](/concepts/model-providers), [Security](/gateway/security),
    [Sandboxing](/gateway/sandboxing).

  </Accordion>

  <Accordion title="¿Qué usan OpenClaw, Flawd y Krill para los modelos?">
    - Estos despliegues pueden variar y cambiar con el tiempo; no hay una recomendación fija de proveedor.
    - Comprueba la configuración actual de tiempo de ejecución en cada gateway con `openclaw models status`.
    - Para agentes sensibles a seguridad/herramientas, usa el mejor modelo de última generación disponible.
  </Accordion>

  <Accordion title="¿Cómo cambio de modelo sobre la marcha (sin reiniciar)?">
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

    Estos son los alias integrados. Puedes añadir alias personalizados mediante `agents.defaults.models`.

    Puedes listar los modelos disponibles con `/model`, `/model list` o `/model status`.

    `/model` (y `/model list`) muestra un selector compacto numerado. Selecciona por número:

    ```
    /model 3
    ```

    También puedes forzar un perfil de autenticación específico para el proveedor (por sesión):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Consejo: `/model status` muestra qué agente está activo, qué archivo `auth-profiles.json` se está usando y qué perfil de autenticación se intentará a continuación.
    También muestra el endpoint del proveedor configurado (`baseUrl`) y el modo de API (`api`) cuando están disponibles.

    **¿Cómo desanclo un perfil que configuré con @profile?**

    Vuelve a ejecutar `/model` **sin** el sufijo `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    Si quieres volver al predeterminado, selecciónalo desde `/model` (o envía `/model <default provider/model>`).
    Usa `/model status` para confirmar qué perfil de autenticación está activo.

  </Accordion>

  <Accordion title="¿Puedo usar GPT 5.2 para tareas diarias y Codex 5.3 para programar?">
    Sí. Configura uno como predeterminado y cambia según sea necesario:

    - **Cambio rápido (por sesión):** `/model gpt-5.4` para tareas diarias, `/model openai-codex/gpt-5.4` para programar con Codex OAuth.
    - **Predeterminado + cambio:** configura `agents.defaults.model.primary` en `openai/gpt-5.4`, luego cambia a `openai-codex/gpt-5.4` al programar (o al revés).
    - **Subagentes:** enruta tareas de programación a subagentes con un modelo predeterminado distinto.

    Consulta [Models](/concepts/models) y [Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title='¿Por qué veo "Model ... is not allowed" y luego no hay respuesta?'>
    Si `agents.defaults.models` está configurado, pasa a ser la **lista de permitidos** para `/model` y cualquier
    anulación de sesión. Elegir un modelo que no esté en esa lista devuelve:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Ese error se devuelve **en lugar de** una respuesta normal. Solución: añade el modelo a
    `agents.defaults.models`, elimina la lista de permitidos o elige un modelo de `/model list`.

  </Accordion>

  <Accordion title='¿Por qué veo "Unknown model: minimax/MiniMax-M2.7"?'>
    Esto significa que el **proveedor no está configurado** (no se encontró configuración de proveedor MiniMax ni perfil de autenticación),
    así que el modelo no puede resolverse.

    Lista de comprobación de solución:

    1. Actualiza a una versión reciente de OpenClaw (o ejecuta desde el código fuente `main`) y reinicia el gateway.
    2. Asegúrate de que MiniMax está configurado (asistente o JSON), o de que la autenticación de MiniMax
       existe en variables de entorno/perfiles de autenticación para que pueda inyectarse
       el proveedor correspondiente (`MINIMAX_API_KEY` para `minimax`, `MINIMAX_OAUTH_TOKEN` o MiniMax
       OAuth almacenado para `minimax-portal`).
    3. Usa el model id exacto (sensible a mayúsculas/minúsculas) para tu ruta de autenticación:
       `minimax/MiniMax-M2.7` o `minimax/MiniMax-M2.7-highspeed` para configuración
       con clave de API, o `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` para configuración
       con OAuth.
    4. Ejecuta:

       ```bash
       openclaw models list
       ```

       y elige de la lista (o `/model list` en el chat).

    Consulta [MiniMax](/providers/minimax) y [Models](/concepts/models).

  </Accordion>

  <Accordion title="¿Puedo usar MiniMax como predeterminado y OpenAI para tareas complejas?">
    Sí. Usa **MiniMax como predeterminado** y cambia de modelo **por sesión** cuando sea necesario.
    Los respaldos son para **errores**, no para "tareas difíciles", así que usa `/model` o un agente separado.

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

    Documentación: [Models](/concepts/models), [Multi-Agent Routing](/concepts/multi-agent), [MiniMax](/providers/minimax), [OpenAI](/providers/openai).

  </Accordion>

  <Accordion title="¿opus / sonnet / gpt son accesos directos integrados?">
    Sí. OpenClaw incluye algunas abreviaturas predeterminadas (solo se aplican cuando el modelo existe en `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Si configuras tu propio alias con el mismo nombre, prevalece tu valor.

  </Accordion>

  <Accordion title="¿Cómo defino/anulo accesos directos (alias) de modelos?">
    Los alias vienen de `agents.defaults.models.<modelId>.alias`. Ejemplo:

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

    Entonces `/model sonnet` (o `/<alias>` cuando esté permitido) se resuelve a ese model ID.

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

    Si referencias un proveedor/modelo pero falta la clave requerida del proveedor, obtendrás un error de autenticación en tiempo de ejecución (por ejemplo `No API key found for provider "zai"`).

    **No API key found for provider después de añadir un nuevo agente**

    Normalmente esto significa que el **nuevo agente** tiene un almacén de autenticación vacío. La autenticación es por agente y
    se guarda en:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Opciones de solución:

    - Ejecuta `openclaw agents add <id>` y configura la autenticación durante el asistente.
    - O copia `auth-profiles.json` del `agentDir` del agente principal al `agentDir` del nuevo agente.

    No reutilices `agentDir` entre agentes; causa colisiones de autenticación/sesión.

  </Accordion>
</AccordionGroup>

## Conmutación por error de modelo y "All models failed"

<AccordionGroup>
  <Accordion title="¿Cómo funciona la conmutación por error?">
    La conmutación por error ocurre en dos etapas:

    1. **Rotación de perfil de autenticación** dentro del mismo proveedor.
    2. **Respaldo de modelo** al siguiente modelo en `agents.defaults.model.fallbacks`.

    Los tiempos de enfriamiento se aplican a los perfiles que fallan (retroceso exponencial), para que OpenClaw pueda seguir respondiendo incluso cuando un proveedor está limitado por tasa o fallando temporalmente.

    El grupo de límites de tasa incluye más que respuestas `429` simples. OpenClaw
    también trata mensajes como `Too many concurrent requests`,
    `ThrottlingException`, `concurrency limit reached`,
    `workers_ai ... quota limit exceeded`, `resource exhausted` y límites
    periódicos de ventana de uso (`weekly/monthly limit reached`) como
    límites de tasa aptos para conmutación por error.

    Algunas respuestas con aspecto de facturación no son `402`, y algunas respuestas `402` HTTP
    también permanecen en ese grupo transitorio. Si un proveedor devuelve
    texto explícito de facturación en `401` o `403`, OpenClaw puede seguir manteniéndolo en
    la ruta de facturación, pero los comparadores de texto específicos del proveedor siguen estando limitados al
    proveedor al que pertenecen (por ejemplo OpenRouter `Key limit exceeded`). Si un mensaje `402`
    en cambio parece un límite reintentable de ventana de uso o de
    gasto de organización/espacio de trabajo (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`), OpenClaw lo trata como
    `rate_limit`, no como una deshabilitación larga por facturación.

    Los errores de desbordamiento de contexto son distintos: firmas como
    `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`,
    `input is too long for the model` o `ollama error: context length
    exceeded` permanecen en la ruta de compactación/reintento en lugar de avanzar al
    respaldo de modelo.

    El texto de error genérico del servidor es intencionadamente más estrecho que
    "cualquier cosa con unknown/error". OpenClaw sí trata formas transitorias con alcance de proveedor
    como Anthropic simple `An unknown error occurred`, OpenRouter simple
    `Provider returned error`, errores de razón de parada como `Unhandled stop reason:
    error`, cargas útiles JSON `api_error` con texto transitorio del servidor
    (`internal server error`, `unknown error, 520`, `upstream error`, `backend
    error`) y errores de proveedor ocupado como `ModelNotReadyException` como
    señales de timeout/sobrecarga aptas para conmutación por error cuando el contexto del proveedor
    coincide.
    El texto genérico interno de respaldo como `LLM request failed with an unknown
    error.` sigue siendo conservador y no activa por sí solo el respaldo de modelo.

  </Accordion>

  <Accordion title='¿Qué significa "No credentials found for profile anthropic:default"?'>
    Significa que el sistema intentó usar el ID de perfil de autenticación `anthropic:default`, pero no pudo encontrar credenciales para él en el almacén de autenticación esperado.

    **Lista de comprobación de solución:**

    - **Confirma dónde viven los perfiles de autenticación** (rutas nuevas frente a heredadas)
      - Actual: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
      - Heredado: `~/.openclaw/agent/*` (migrado por `openclaw doctor`)
    - **Confirma que tu variable de entorno está cargada por el Gateway**
      - Si configuras `ANTHROPIC_API_KEY` en tu shell pero ejecutas el Gateway mediante systemd/launchd, puede que no la herede. Ponla en `~/.openclaw/.env` o habilita `env.shellEnv`.
    - **Asegúrate de estar editando el agente correcto**
      - Las configuraciones multiagente significan que puede haber varios archivos `auth-profiles.json`.
    - **Comprueba rápidamente el estado de modelo/autenticación**
      - Usa `openclaw models status` para ver modelos configurados y si los proveedores están autenticados.

    **Lista de comprobación de solución para "No credentials found for profile anthropic"**

    Esto significa que la ejecución está fijada a un perfil de autenticación Anthropic, pero el Gateway
    no puede encontrarlo en su almacén de autenticación.

    - **Usa Claude CLI**
      - Ejecuta `openclaw models auth login --provider anthropic --method cli --set-default` en el host del gateway.
    - **Si en su lugar quieres usar una clave de API**
      - Pon `ANTHROPIC_API_KEY` en `~/.openclaw/.env` en el **host del gateway**.
      - Borra cualquier orden fijado que fuerce un perfil inexistente:

        ```bash
        openclaw models auth order clear --provider anthropic
        ```

    - **Confirma que estás ejecutando comandos en el host del gateway**
      - En modo remoto, los perfiles de autenticación viven en la máquina del gateway, no en tu portátil.

  </Accordion>

  <Accordion title="¿Por qué también intentó Google Gemini y falló?">
    Si la configuración del modelo incluye Google Gemini como respaldo (o cambiaste a una abreviatura de Gemini), OpenClaw lo intentará durante el respaldo de modelo. Si no has configurado credenciales de Google, verás `No API key found for provider "google"`.

    Solución: proporciona autenticación de Google o elimina/evita los modelos de Google en `agents.defaults.model.fallbacks` / alias para que el respaldo no se enrute allí.

    **LLM request rejected: thinking signature required (Google Antigravity)**

    Causa: el historial de la sesión contiene **bloques de thinking sin firma** (a menudo por
    un stream abortado/parcial). Google Antigravity requiere firmas para los bloques de thinking.

    Solución: OpenClaw ahora elimina los bloques de thinking sin firma para Google Antigravity Claude. Si sigue apareciendo, inicia una **sesión nueva** o configura `/thinking off` para ese agente.

  </Accordion>
</AccordionGroup>

## Perfiles de autenticación: qué son y cómo gestionarlos

Relacionado: [/concepts/oauth](/concepts/oauth) (flujos OAuth, almacenamiento de tokens, patrones multi-cuenta)

<AccordionGroup>
  <Accordion title="¿Qué es un perfil de autenticación?">
    Un perfil de autenticación es un registro de credencial con nombre (OAuth o clave de API) vinculado a un proveedor. Los perfiles viven en:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

  </Accordion>

  <Accordion title="¿Cuáles son los IDs de perfil habituales?">
    OpenClaw usa IDs con prefijo del proveedor, como:

    - `anthropic:default` (habitual cuando no existe identidad de correo electrónico)
    - `anthropic:<email>` para identidades OAuth
    - IDs personalizados que elijas (por ejemplo `anthropic:work`)

  </Accordion>

  <Accordion title="¿Puedo controlar qué perfil de autenticación se prueba primero?">
    Sí. La configuración admite metadatos opcionales para perfiles y un orden por proveedor (`auth.order.<provider>`). Esto **no** almacena secretos; asigna IDs a proveedor/modo y establece el orden de rotación.

    OpenClaw puede omitir temporalmente un perfil si está en un **tiempo de enfriamiento** corto (límites de tasa/timeouts/fallos de autenticación) o en un estado **deshabilitado** más largo (facturación/créditos insuficientes). Para inspeccionarlo,