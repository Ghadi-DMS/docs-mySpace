---
read_when:
    - Instalar OpenClaw en Windows
    - Elegir entre Windows nativo y WSL2
    - Buscar el estado de la app complementaria de Windows
summary: 'Compatibilidad con Windows: rutas de instalación nativa y WSL2, daemon y advertencias actuales'
title: Windows
x-i18n:
    generated_at: "2026-04-05T12:49:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7d9819206bdd65cf03519c1bc73ed0c7889b0ab842215ea94343262300adfd14
    source_path: platforms/windows.md
    workflow: 15
---

# Windows

OpenClaw es compatible tanto con **Windows nativo** como con **WSL2**. WSL2 es la
ruta más estable y la recomendada para la experiencia completa: la CLI, el Gateway y las
herramientas se ejecutan dentro de Linux con compatibilidad total. Windows nativo funciona para
el uso principal de CLI y Gateway, con algunas advertencias indicadas más abajo.

Están previstas apps complementarias nativas para Windows.

## WSL2 (recomendado)

- [Primeros pasos](/es/start/getting-started) (úsalo dentro de WSL)
- [Instalación y actualizaciones](/install/updating)
- Guía oficial de WSL2 (Microsoft): [https://learn.microsoft.com/windows/wsl/install](https://learn.microsoft.com/windows/wsl/install)

## Estado de Windows nativo

Los flujos de CLI nativos de Windows están mejorando, pero WSL2 sigue siendo la ruta recomendada.

Qué funciona bien hoy en Windows nativo:

- instalador del sitio web mediante `install.ps1`
- uso local de CLI como `openclaw --version`, `openclaw doctor` y `openclaw plugins list --json`
- prueba smoke integrada de agente/proveedor local como:

```powershell
openclaw agent --local --agent main --thinking low -m "Reply with exactly WINDOWS-HATCH-OK."
```

Advertencias actuales:

- `openclaw onboard --non-interactive` todavía espera un gateway local accesible a menos que pases `--skip-health`
- `openclaw onboard --non-interactive --install-daemon` y `openclaw gateway install` intentan primero usar Tareas programadas de Windows
- si se deniega la creación de la tarea programada, OpenClaw recurre a un elemento de inicio de sesión por usuario en la carpeta Startup y arranca el gateway inmediatamente
- si `schtasks` se cuelga o deja de responder, OpenClaw ahora aborta rápidamente esa ruta y recurre al respaldo en lugar de quedarse bloqueado para siempre
- las Tareas programadas siguen siendo la opción preferida cuando están disponibles porque proporcionan mejor estado del supervisor

Si solo quieres la CLI nativa, sin instalación del servicio gateway, usa una de estas opciones:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

Si sí quieres un inicio gestionado en Windows nativo:

```powershell
openclaw gateway install
openclaw gateway status --json
```

Si la creación de la tarea programada está bloqueada, el modo de servicio de respaldo seguirá iniciándose automáticamente después del inicio de sesión mediante la carpeta Startup del usuario actual.

## Gateway

- [Runbook del Gateway](/gateway)
- [Configuración](/gateway/configuration)

## Instalación del servicio Gateway (CLI)

Dentro de WSL2:

```
openclaw onboard --install-daemon
```

O bien:

```
openclaw gateway install
```

O bien:

```
openclaw configure
```

Selecciona **Gateway service** cuando se te pida.

Reparar/migrar:

```
openclaw doctor
```

## Inicio automático del Gateway antes del inicio de sesión en Windows

Para configuraciones sin interfaz, asegúrate de que toda la cadena de arranque funcione incluso cuando nadie haya iniciado sesión en
Windows.

### 1) Mantener los servicios de usuario activos sin iniciar sesión

Dentro de WSL:

```bash
sudo loginctl enable-linger "$(whoami)"
```

### 2) Instalar el servicio de usuario del gateway de OpenClaw

Dentro de WSL:

```bash
openclaw gateway install
```

### 3) Iniciar WSL automáticamente al arrancar Windows

En PowerShell como administrador:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec /bin/true" /sc onstart /ru SYSTEM
```

Sustituye `Ubuntu` por el nombre de tu distribución obtenido con:

```powershell
wsl --list --verbose
```

### Verificar la cadena de inicio

Después de un reinicio (antes de iniciar sesión en Windows), comprueba desde WSL:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## Avanzado: exponer servicios WSL por LAN (portproxy)

WSL tiene su propia red virtual. Si otra máquina necesita llegar a un servicio
que se ejecuta **dentro de WSL** (SSH, un servidor local de TTS o el Gateway), debes
reenviar un puerto de Windows a la IP actual de WSL. La IP de WSL cambia después de reinicios,
por lo que puede que necesites actualizar la regla de reenvío.

Ejemplo (PowerShell **como administrador**):

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP not found." }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort
```

Permite el puerto en el Firewall de Windows (una sola vez):

```powershell
New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

Actualiza el portproxy después de reiniciar WSL:

```powershell
netsh interface portproxy delete v4tov4 listenport=$ListenPort listenaddress=0.0.0.0 | Out-Null
netsh interface portproxy add v4tov4 listenport=$ListenPort listenaddress=0.0.0.0 `
  connectaddress=$WslIp connectport=$TargetPort | Out-Null
```

Notas:

- El SSH desde otra máquina apunta a la **IP del host de Windows** (ejemplo: `ssh user@windows-host -p 2222`).
- Los nodos remotos deben apuntar a una URL de Gateway **alcanzable** (no `127.0.0.1`); usa
  `openclaw status --all` para confirmarlo.
- Usa `listenaddress=0.0.0.0` para acceso por LAN; `127.0.0.1` lo mantiene solo local.
- Si quieres esto automático, registra una Tarea programada para ejecutar el paso de
  actualización al iniciar sesión.

## Instalación paso a paso de WSL2

### 1) Instalar WSL2 + Ubuntu

Abre PowerShell (administrador):

```powershell
wsl --install
# Or pick a distro explicitly:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

Reinicia si Windows lo solicita.

### 2) Habilitar systemd (obligatorio para instalar el gateway)

En tu terminal WSL:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Luego, desde PowerShell:

```powershell
wsl --shutdown
```

Vuelve a abrir Ubuntu y luego verifica:

```bash
systemctl --user status
```

### 3) Instalar OpenClaw (dentro de WSL)

Sigue el flujo de Linux de Primeros pasos dentro de WSL:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
openclaw onboard
```

Guía completa: [Primeros pasos](/es/start/getting-started)

## App complementaria de Windows

Todavía no tenemos una app complementaria para Windows. Las contribuciones son bienvenidas si quieres
hacerla realidad.
