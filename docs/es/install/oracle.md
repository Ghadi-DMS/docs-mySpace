---
read_when:
    - Configurar OpenClaw en Oracle Cloud
    - Buscar alojamiento VPS gratuito para OpenClaw
    - Quieres OpenClaw 24/7 en un servidor pequeño
summary: Aloja OpenClaw en la capa ARM Always Free de Oracle Cloud
title: Oracle Cloud
x-i18n:
    generated_at: "2026-04-05T12:46:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6915f8c428cfcbc215ba6547273df6e7b93212af6590827a3853f15617ba245e
    source_path: install/oracle.md
    workflow: 15
---

# Oracle Cloud

Ejecuta un Gateway persistente de OpenClaw en la capa ARM **Always Free** de Oracle Cloud (hasta 4 OCPU, 24 GB de RAM y 200 GB de almacenamiento) sin costo.

## Requisitos previos

- Cuenta de Oracle Cloud ([registro](https://www.oracle.com/cloud/free/)) -- consulta la [guía de registro de la comunidad](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) si encuentras problemas
- Cuenta de Tailscale (gratis en [tailscale.com](https://tailscale.com))
- Un par de claves SSH
- Aproximadamente 30 minutos

## Configuración

<Steps>
  <Step title="Crear una instancia de OCI">
    1. Inicia sesión en la [Consola de Oracle Cloud](https://cloud.oracle.com/).
    2. Ve a **Compute > Instances > Create Instance**.
    3. Configura:
       - **Name:** `openclaw`
       - **Image:** Ubuntu 24.04 (aarch64)
       - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
       - **OCPUs:** 2 (o hasta 4)
       - **Memory:** 12 GB (o hasta 24 GB)
       - **Boot volume:** 50 GB (hasta 200 GB gratis)
       - **SSH key:** agrega tu clave pública
    4. Haz clic en **Create** y anota la dirección IP pública.

    <Tip>
    Si la creación de la instancia falla con "Out of capacity", prueba con un dominio de disponibilidad diferente o inténtalo más tarde. La capacidad de la capa gratuita es limitada.
    </Tip>

  </Step>

  <Step title="Conectarte y actualizar el sistema">
    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    `build-essential` es necesario para la compilación en ARM de algunas dependencias.

  </Step>

  <Step title="Configurar el usuario y el hostname">
    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    Habilitar linger mantiene los servicios de usuario en ejecución después de cerrar sesión.

  </Step>

  <Step title="Instalar Tailscale">
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    A partir de ahora, conéctate mediante Tailscale: `ssh ubuntu@openclaw`.

  </Step>

  <Step title="Instalar OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    Cuando aparezca el mensaje "How do you want to hatch your bot?", selecciona **Do this later**.

  </Step>

  <Step title="Configurar el gateway">
    Usa autenticación por token con Tailscale Serve para un acceso remoto seguro.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw config set gateway.auth.mode token
    openclaw doctor --generate-gateway-token
    openclaw config set gateway.tailscale.mode serve
    openclaw config set gateway.trustedProxies '["127.0.0.1"]'

    systemctl --user restart openclaw-gateway.service
    ```

    `gateway.trustedProxies=["127.0.0.1"]` aquí es solo para el manejo de IP reenviada/cliente local del proxy local de Tailscale Serve. **No** es `gateway.auth.mode: "trusted-proxy"`. Las rutas del visor de diff mantienen el comportamiento de cierre por fallo en esta configuración: las solicitudes del visor crudas a `127.0.0.1` sin encabezados de proxy reenviado pueden devolver `Diff not found`. Usa `mode=file` / `mode=both` para adjuntos, o habilita intencionalmente visores remotos y configura `plugins.entries.diffs.config.viewerBaseUrl` (o pasa un `baseUrl` de proxy) si necesitas enlaces compartibles del visor.

  </Step>

  <Step title="Restringir la seguridad de la VCN">
    Bloquea todo el tráfico excepto Tailscale en el borde de la red:

    1. Ve a **Networking > Virtual Cloud Networks** en la Consola de OCI.
    2. Haz clic en tu VCN y luego en **Security Lists > Default Security List**.
    3. **Elimina** todas las reglas de entrada excepto `0.0.0.0/0 UDP 41641` (Tailscale).
    4. Mantén las reglas de salida predeterminadas (permitir todo el tráfico saliente).

    Esto bloquea SSH en el puerto 22, HTTP, HTTPS y todo lo demás en el borde de la red. A partir de este punto, solo podrás conectarte mediante Tailscale.

  </Step>

  <Step title="Verificar">
    ```bash
    openclaw --version
    systemctl --user status openclaw-gateway.service
    tailscale serve status
    curl http://localhost:18789
    ```

    Accede a la UI de control desde cualquier dispositivo de tu tailnet:

    ```
    https://openclaw.<tailnet-name>.ts.net/
    ```

    Reemplaza `<tailnet-name>` por el nombre de tu tailnet (visible en `tailscale status`).

  </Step>
</Steps>

## Respaldo: túnel SSH

Si Tailscale Serve no funciona, usa un túnel SSH desde tu máquina local:

```bash
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

Luego abre `http://localhost:18789`.

## Solución de problemas

**La creación de la instancia falla ("Out of capacity")** -- Las instancias ARM de la capa gratuita son populares. Prueba con un dominio de disponibilidad diferente o vuelve a intentarlo en horas de menor demanda.

**Tailscale no se conecta** -- Ejecuta `sudo tailscale up --ssh --hostname=openclaw --reset` para volver a autenticarte.

**El gateway no se inicia** -- Ejecuta `openclaw doctor --non-interactive` y revisa los registros con `journalctl --user -u openclaw-gateway.service -n 50`.

**Problemas con binarios ARM** -- La mayoría de los paquetes npm funcionan en ARM64. Para binarios nativos, busca versiones `linux-arm64` o `aarch64`. Verifica la arquitectura con `uname -m`.

## Próximos pasos

- [Canales](/channels) -- conecta Telegram, WhatsApp, Discord y más
- [Configuración del gateway](/gateway/configuration) -- todas las opciones de configuración
- [Actualización](/install/updating) -- mantén OpenClaw actualizado
