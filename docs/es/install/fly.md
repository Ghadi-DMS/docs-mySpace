---
read_when:
    - Desplegar OpenClaw en Fly.io
    - Configurar volúmenes, secretos y configuración de primera ejecución en Fly
summary: Despliegue paso a paso en Fly.io para OpenClaw con almacenamiento persistente y HTTPS
title: Fly.io
x-i18n:
    generated_at: "2026-04-05T12:45:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: a5f8c2c03295d786c0d8df98f8a5ae9335fa0346a188b81aae3e07d566a2c0ef
    source_path: install/fly.md
    workflow: 15
---

# Despliegue en Fly.io

**Objetivo:** Gateway de OpenClaw ejecutándose en una máquina de [Fly.io](https://fly.io) con almacenamiento persistente, HTTPS automático y acceso a Discord/canales.

## Qué necesitas

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) instalada
- Cuenta de Fly.io (el nivel gratuito funciona)
- Autenticación del modelo: clave de API para el proveedor de modelos que elijas
- Credenciales de canal: token de bot de Discord, token de Telegram, etc.

## Ruta rápida para principiantes

1. Clonar el repositorio → personalizar `fly.toml`
2. Crear app + volumen → establecer secretos
3. Desplegar con `fly deploy`
4. Acceder por SSH para crear la configuración o usar la UI de Control

<Steps>
  <Step title="Crear la app de Fly">
    ```bash
    # Clonar el repositorio
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # Crear una nueva app de Fly (elige tu propio nombre)
    fly apps create my-openclaw

    # Crear un volumen persistente (1 GB suele ser suficiente)
    fly volumes create openclaw_data --size 1 --region iad
    ```

    **Consejo:** Elige una región cercana a ti. Opciones habituales: `lhr` (Londres), `iad` (Virginia), `sjc` (San José).

  </Step>

  <Step title="Configurar fly.toml">
    Edita `fly.toml` para que coincida con el nombre de tu app y tus requisitos.

    **Nota de seguridad:** La configuración predeterminada expone una URL pública. Para un despliegue reforzado sin IP pública, consulta [Despliegue privado](#despliegue-privado-reforzado) o usa `fly.private.toml`.

    ```toml
    app = "my-openclaw"  # Nombre de tu app
    primary_region = "iad"

    [build]
      dockerfile = "Dockerfile"

    [env]
      NODE_ENV = "production"
      OPENCLAW_PREFER_PNPM = "1"
      OPENCLAW_STATE_DIR = "/data"
      NODE_OPTIONS = "--max-old-space-size=1536"

    [processes]
      app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

    [http_service]
      internal_port = 3000
      force_https = true
      auto_stop_machines = false
      auto_start_machines = true
      min_machines_running = 1
      processes = ["app"]

    [[vm]]
      size = "shared-cpu-2x"
      memory = "2048mb"

    [mounts]
      source = "openclaw_data"
      destination = "/data"
    ```

    **Ajustes clave:**

    | Ajuste                        | Motivo                                                                      |
    | ----------------------------- | --------------------------------------------------------------------------- |
    | `--bind lan`                  | Vincula a `0.0.0.0` para que el proxy de Fly pueda alcanzar el gateway      |
    | `--allow-unconfigured`        | Inicia sin archivo de configuración (lo crearás después)                    |
    | `internal_port = 3000`        | Debe coincidir con `--port 3000` (o `OPENCLAW_GATEWAY_PORT`) para las comprobaciones de estado de Fly |
    | `memory = "2048mb"`           | 512 MB es demasiado poco; se recomiendan 2 GB                               |
    | `OPENCLAW_STATE_DIR = "/data"` | Conserva el estado en el volumen                                           |

  </Step>

  <Step title="Establecer secretos">
    ```bash
    # Obligatorio: token del Gateway (para vinculación no loopback)
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # Claves de API del proveedor de modelos
    fly secrets set ANTHROPIC_API_KEY=sk-ant-...

    # Opcional: otros proveedores
    fly secrets set OPENAI_API_KEY=sk-...
    fly secrets set GOOGLE_API_KEY=...

    # Tokens de canal
    fly secrets set DISCORD_BOT_TOKEN=MTQ...
    ```

    **Notas:**

    - Las vinculaciones no loopback (`--bind lan`) requieren una ruta válida de autenticación del gateway. Este ejemplo de Fly.io usa `OPENCLAW_GATEWAY_TOKEN`, pero `gateway.auth.password` o un despliegue `trusted-proxy` no loopback correctamente configurado también cumplen este requisito.
    - Trata estos tokens como contraseñas.
    - **Prefiere variables de entorno en lugar del archivo de configuración** para todas las claves de API y tokens. Así los secretos no quedan en `openclaw.json`, donde podrían exponerse o registrarse accidentalmente.

  </Step>

  <Step title="Desplegar">
    ```bash
    fly deploy
    ```

    El primer despliegue compila la imagen de Docker (~2-3 minutos). Los despliegues posteriores son más rápidos.

    Después del despliegue, verifica:

    ```bash
    fly status
    fly logs
    ```

    Deberías ver:

    ```
    [gateway] listening on ws://0.0.0.0:3000 (PID xxx)
    [discord] logged in to discord as xxx
    ```

  </Step>

  <Step title="Crear el archivo de configuración">
    Accede por SSH a la máquina para crear una configuración adecuada:

    ```bash
    fly ssh console
    ```

    Crea el directorio y el archivo de configuración:

    ```bash
    mkdir -p /data
    cat > /data/openclaw.json << 'EOF'
    {
      "agents": {
        "defaults": {
          "model": {
            "primary": "anthropic/claude-opus-4-6",
            "fallbacks": ["anthropic/claude-sonnet-4-6", "openai/gpt-5.4"]
          },
          "maxConcurrent": 4
        },
        "list": [
          {
            "id": "main",
            "default": true
          }
        ]
      },
      "auth": {
        "profiles": {
          "anthropic:default": { "mode": "token", "provider": "anthropic" },
          "openai:default": { "mode": "token", "provider": "openai" }
        }
      },
      "bindings": [
        {
          "agentId": "main",
          "match": { "channel": "discord" }
        }
      ],
      "channels": {
        "discord": {
          "enabled": true,
          "groupPolicy": "allowlist",
          "guilds": {
            "YOUR_GUILD_ID": {
              "channels": { "general": { "allow": true } },
              "requireMention": false
            }
          }
        }
      },
      "gateway": {
        "mode": "local",
        "bind": "auto"
      },
      "meta": {}
    }
    EOF
    ```

    **Nota:** Con `OPENCLAW_STATE_DIR=/data`, la ruta de configuración es `/data/openclaw.json`.

    **Nota:** El token de Discord puede venir de:

    - Variable de entorno: `DISCORD_BOT_TOKEN` (recomendado para secretos)
    - Archivo de configuración: `channels.discord.token`

    Si usas la variable de entorno, no hace falta añadir el token a la configuración. El gateway lee `DISCORD_BOT_TOKEN` automáticamente.

    Reinicia para aplicar los cambios:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  </Step>

  <Step title="Acceder al Gateway">
    ### UI de Control

    Ábrela en el navegador:

    ```bash
    fly open
    ```

    O visita `https://my-openclaw.fly.dev/`

    Autentícate con el secreto compartido configurado. Esta guía usa el token del gateway de `OPENCLAW_GATEWAY_TOKEN`; si cambiaste a autenticación por contraseña, usa esa contraseña en su lugar.

    ### Registros

    ```bash
    fly logs              # Registros en vivo
    fly logs --no-tail    # Registros recientes
    ```

    ### Consola SSH

    ```bash
    fly ssh console
    ```

  </Step>
</Steps>

## Solución de problemas

### "App is not listening on expected address"

El gateway se está vinculando a `127.0.0.1` en lugar de `0.0.0.0`.

**Solución:** Añade `--bind lan` al comando del proceso en `fly.toml`.

### Fallos en comprobaciones de estado / conexión rechazada

Fly no puede alcanzar el gateway en el puerto configurado.

**Solución:** Asegúrate de que `internal_port` coincida con el puerto del gateway (establece `--port 3000` o `OPENCLAW_GATEWAY_PORT=3000`).

### OOM / problemas de memoria

El contenedor sigue reiniciándose o siendo terminado. Señales: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration` o reinicios silenciosos.

**Solución:** Aumenta la memoria en `fly.toml`:

```toml
[[vm]]
  memory = "2048mb"
```

O actualiza una máquina existente:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

**Nota:** 512 MB es demasiado poco. 1 GB puede funcionar, pero puede producir OOM bajo carga o con registros detallados. **Se recomiendan 2 GB.**

### Problemas de bloqueo del Gateway

El Gateway se niega a iniciar con errores de "already running".

Esto ocurre cuando el contenedor se reinicia pero el archivo de bloqueo de PID permanece en el volumen.

**Solución:** Elimina el archivo de bloqueo:

```bash
fly ssh console --command "rm -f /data/gateway.*.lock"
fly machine restart <machine-id>
```

El archivo de bloqueo está en `/data/gateway.*.lock` (no en un subdirectorio).

### La configuración no se está leyendo

`--allow-unconfigured` solo omite la protección de inicio. No crea ni repara `/data/openclaw.json`, así que asegúrate de que tu configuración real exista e incluya `gateway.mode="local"` cuando quieras un inicio local normal del gateway.

Verifica que exista la configuración:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### Escribir configuración mediante SSH

El comando `fly ssh console -C` no admite redirección del shell. Para escribir un archivo de configuración:

```bash
# Usa echo + tee (canaliza desde local a remoto)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# O usa sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

**Nota:** `fly sftp` puede fallar si el archivo ya existe. Elimínalo primero:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### El estado no persiste

Si pierdes perfiles de autenticación, estado de canales/proveedores o sesiones después de un reinicio,
el directorio de estado se está escribiendo en el sistema de archivos del contenedor.

**Solución:** Asegúrate de que `OPENCLAW_STATE_DIR=/data` esté establecido en `fly.toml` y vuelve a desplegar.

## Actualizaciones

```bash
# Obtener los últimos cambios
git pull

# Volver a desplegar
fly deploy

# Comprobar estado
fly status
fly logs
```

### Actualizar el comando de la máquina

Si necesitas cambiar el comando de inicio sin un redespliegue completo:

```bash
# Obtener el ID de la máquina
fly machines list

# Actualizar el comando
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# O con aumento de memoria
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

**Nota:** Después de `fly deploy`, el comando de la máquina puede volver a lo que esté en `fly.toml`. Si hiciste cambios manuales, vuelve a aplicarlos después del despliegue.

## Despliegue privado (reforzado)

De forma predeterminada, Fly asigna IP públicas, haciendo que tu gateway sea accesible en `https://your-app.fly.dev`. Esto es conveniente, pero significa que tu despliegue es detectable por escáneres de Internet (Shodan, Censys, etc.).

Para un despliegue reforzado con **cero exposición pública**, usa la plantilla privada.

### Cuándo usar un despliegue privado

- Solo haces llamadas/mensajes **salientes** (sin webhooks entrantes)
- Usas túneles **ngrok o Tailscale** para cualquier devolución de llamada webhook
- Accedes al gateway mediante **SSH, proxy o WireGuard** en lugar del navegador
- Quieres que el despliegue quede **oculto a escáneres de Internet**

### Configuración

Usa `fly.private.toml` en lugar de la configuración estándar:

```bash
# Desplegar con configuración privada
fly deploy -c fly.private.toml
```

O convierte un despliegue existente:

```bash
# Enumerar IP actuales
fly ips list -a my-openclaw

# Liberar IP públicas
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# Cambiar a configuración privada para que futuros despliegues no reasignen IP públicas
# (elimina [http_service] o despliega con la plantilla privada)
fly deploy -c fly.private.toml

# Asignar IPv6 solo privada
fly ips allocate-v6 --private -a my-openclaw
```

Después de esto, `fly ips list` debería mostrar solo una IP de tipo `private`:

```
VERSION  IP                   TYPE             REGION
v6       fdaa:x:x:x:x::x      private          global
```

### Acceder a un despliegue privado

Como no hay URL pública, usa uno de estos métodos:

**Opción 1: proxy local (la más sencilla)**

```bash
# Reenviar el puerto local 3000 a la app
fly proxy 3000:3000 -a my-openclaw

# Luego abre http://localhost:3000 en el navegador
```

**Opción 2: VPN WireGuard**

```bash
# Crear configuración WireGuard (una sola vez)
fly wireguard create

# Importar al cliente WireGuard y luego acceder mediante IPv6 interna
# Ejemplo: http://[fdaa:x:x:x:x::x]:3000
```

**Opción 3: solo SSH**

```bash
fly ssh console -a my-openclaw
```

### Webhooks con despliegue privado

Si necesitas devoluciones de llamada webhook (Twilio, Telnyx, etc.) sin exposición pública:

1. **Túnel ngrok** - Ejecuta ngrok dentro del contenedor o como sidecar
2. **Tailscale Funnel** - Expón rutas específicas mediante Tailscale
3. **Solo saliente** - Algunos proveedores (Twilio) funcionan bien para llamadas salientes sin webhooks

Ejemplo de configuración de voice-call con ngrok:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          tunnel: { provider: "ngrok" },
          webhookSecurity: {
            allowedHosts: ["example.ngrok.app"],
          },
        },
      },
    },
  },
}
```

El túnel ngrok se ejecuta dentro del contenedor y proporciona una URL pública de webhook sin exponer la propia app de Fly. Establece `webhookSecurity.allowedHosts` en el nombre de host público del túnel para que se acepten los encabezados de host reenviados.

### Ventajas de seguridad

| Aspecto            | Público      | Privado    |
| ------------------ | ------------ | ---------- |
| Escáneres de Internet | Detectable | Oculto     |
| Ataques directos    | Posibles     | Bloqueados |
| Acceso a la UI de Control | Navegador | Proxy/VPN  |
| Entrega de webhooks  | Directa      | Mediante túnel |

## Notas

- Fly.io usa **arquitectura x86** (no ARM)
- El Dockerfile es compatible con ambas arquitecturas
- Para el onboarding de WhatsApp/Telegram, usa `fly ssh console`
- Los datos persistentes viven en el volumen en `/data`
- Signal requiere Java + signal-cli; usa una imagen personalizada y mantén la memoria en 2 GB o más.

## Coste

Con la configuración recomendada (`shared-cpu-2x`, 2 GB de RAM):

- ~10-15 USD/mes según el uso
- El nivel gratuito incluye cierta asignación

Consulta [precios de Fly.io](https://fly.io/docs/about/pricing/) para más detalles.

## Siguientes pasos

- Configura canales de mensajería: [Channels](/channels)
- Configura el Gateway: [Configuración del gateway](/gateway/configuration)
- Mantén OpenClaw actualizado: [Updating](/install/updating)
