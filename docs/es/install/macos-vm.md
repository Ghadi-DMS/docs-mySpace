---
read_when:
    - Quieres OpenClaw aislado de tu entorno principal de macOS
    - Quieres integración con iMessage (BlueBubbles) en un sandbox
    - Quieres un entorno de macOS reiniciable que puedas clonar
    - Quieres comparar opciones de VM de macOS local frente a alojada
summary: Ejecuta OpenClaw en una VM de macOS aislada (local o alojada) cuando necesites aislamiento o iMessage
title: VM de macOS
x-i18n:
    generated_at: "2026-04-05T12:46:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: b1f7c5691fd2686418ee25f2c38b1f9badd511daeef2906d21ad30fb523b013f
    source_path: install/macos-vm.md
    workflow: 15
---

# OpenClaw en VM de macOS (Sandboxing)

## Opción predeterminada recomendada (para la mayoría de los usuarios)

- **VPS pequeño con Linux** para un Gateway siempre activo y de bajo costo. Consulta [Alojamiento VPS](/vps).
- **Hardware dedicado** (Mac mini o equipo Linux) si quieres control total y una **IP residencial** para automatización del navegador. Muchos sitios bloquean las IP de centros de datos, así que la navegación local suele funcionar mejor.
- **Híbrido:** mantén el Gateway en un VPS económico y conecta tu Mac como **node** cuando necesites automatización del navegador/UI. Consulta [Nodes](/nodes) y [Gateway remote](/gateway/remote).

Usa una VM de macOS cuando necesites específicamente capacidades exclusivas de macOS (iMessage/BlueBubbles) o quieras un aislamiento estricto de tu Mac de uso diario.

## Opciones de VM de macOS

### VM local en tu Mac Apple Silicon (Lume)

Ejecuta OpenClaw en una VM de macOS aislada en tu Mac Apple Silicon existente usando [Lume](https://cua.ai/docs/lume).

Esto te da:

- Entorno macOS completo en aislamiento (tu host permanece limpio)
- Compatibilidad con iMessage mediante BlueBubbles (imposible en Linux/Windows)
- Restablecimiento instantáneo clonando VM
- Sin costos adicionales de hardware o nube

### Proveedores de Mac alojado (nube)

Si quieres macOS en la nube, los proveedores de Mac alojado también funcionan:

- [MacStadium](https://www.macstadium.com/) (Mac alojados)
- Otros proveedores de Mac alojado también funcionan; sigue su documentación de VM + SSH

Una vez que tengas acceso SSH a una VM de macOS, continúa en el paso 6 más abajo.

---

## Ruta rápida (Lume, usuarios con experiencia)

1. Instala Lume
2. `lume create openclaw --os macos --ipsw latest`
3. Completa el Asistente de configuración, habilita Remote Login (SSH)
4. `lume run openclaw --no-display`
5. Accede por SSH, instala OpenClaw, configura los canales
6. Listo

---

## Qué necesitas (Lume)

- Mac Apple Silicon (M1/M2/M3/M4)
- macOS Sequoia o posterior en el host
- ~60 GB de espacio libre por VM
- ~20 minutos

---

## 1) Instalar Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

Si `~/.local/bin` no está en tu PATH:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

Verifica:

```bash
lume --version
```

Documentación: [Instalación de Lume](https://cua.ai/docs/lume/guide/getting-started/installation)

---

## 2) Crear la VM de macOS

```bash
lume create openclaw --os macos --ipsw latest
```

Esto descarga macOS y crea la VM. Se abre automáticamente una ventana VNC.

Nota: La descarga puede tardar un rato según tu conexión.

---

## 3) Completar el Asistente de configuración

En la ventana VNC:

1. Selecciona idioma y región
2. Omite el Apple ID (o inicia sesión si quieres iMessage más adelante)
3. Crea una cuenta de usuario (recuerda el nombre de usuario y la contraseña)
4. Omite todas las funciones opcionales

Cuando termine la configuración, habilita SSH:

1. Abre Configuración del sistema → General → Compartir
2. Habilita "Remote Login"

---

## 4) Obtener la dirección IP de la VM

```bash
lume get openclaw
```

Busca la dirección IP (normalmente `192.168.64.x`).

---

## 5) Acceder por SSH a la VM

```bash
ssh youruser@192.168.64.X
```

Sustituye `youruser` por la cuenta que creaste y la IP por la IP de tu VM.

---

## 6) Instalar OpenClaw

Dentro de la VM:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

Sigue las indicaciones del onboarding para configurar tu proveedor de modelos (Anthropic, OpenAI, etc.).

---

## 7) Configurar canales

Edita el archivo de configuración:

```bash
nano ~/.openclaw/openclaw.json
```

Añade tus canales:

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
  },
}
```

Después inicia sesión en WhatsApp (escanea el QR):

```bash
openclaw channels login
```

---

## 8) Ejecutar la VM sin interfaz

Detén la VM y reiníciala sin pantalla:

```bash
lume stop openclaw
lume run openclaw --no-display
```

La VM se ejecuta en segundo plano. El daemon de OpenClaw mantiene el gateway en funcionamiento.

Para comprobar el estado:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

---

## Extra: integración con iMessage

Esta es la gran ventaja de ejecutarlo en macOS. Usa [BlueBubbles](https://bluebubbles.app) para añadir iMessage a OpenClaw.

Dentro de la VM:

1. Descarga BlueBubbles desde bluebubbles.app
2. Inicia sesión con tu Apple ID
3. Habilita la API web y establece una contraseña
4. Apunta los webhooks de BlueBubbles a tu gateway (ejemplo: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`)

Añade esto a tu configuración de OpenClaw:

```json5
{
  channels: {
    bluebubbles: {
      serverUrl: "http://localhost:1234",
      password: "your-api-password",
      webhookPath: "/bluebubbles-webhook",
    },
  },
}
```

Reinicia el gateway. Ahora tu agente puede enviar y recibir iMessages.

Detalles completos de configuración: [Canal BlueBubbles](/channels/bluebubbles)

---

## Guardar una imagen dorada

Antes de personalizar más, crea una instantánea de tu estado limpio:

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

Restablece en cualquier momento:

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

---

## Ejecutarlo 24/7

Mantén la VM en ejecución:

- Manteniendo tu Mac conectado a la corriente
- Deshabilitando el reposo en Configuración del sistema → Ahorro de energía
- Usando `caffeinate` si es necesario

Para un funcionamiento realmente permanente, considera un Mac mini dedicado o un VPS pequeño. Consulta [Alojamiento VPS](/vps).

---

## Solución de problemas

| Problema                  | Solución                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------- |
| No puedes acceder por SSH a la VM        | Comprueba que "Remote Login" esté habilitado en la Configuración del sistema de la VM                            |
| La IP de la VM no aparece        | Espera a que la VM termine de arrancar y ejecuta `lume get openclaw` otra vez                           |
| No se encuentra el comando Lume   | Añade `~/.local/bin` a tu PATH                                                    |
| El QR de WhatsApp no se escanea | Asegúrate de haber iniciado sesión en la VM (no en el host) cuando ejecutes `openclaw channels login` |

---

## Documentación relacionada

- [Alojamiento VPS](/vps)
- [Nodes](/nodes)
- [Gateway remote](/gateway/remote)
- [Canal BlueBubbles](/channels/bluebubbles)
- [Inicio rápido de Lume](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [Referencia de la CLI de Lume](https://cua.ai/docs/lume/reference/cli-reference)
- [Configuración desatendida de VM](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup) (avanzado)
- [Sandboxing con Docker](/install/docker) (enfoque alternativo de aislamiento)
