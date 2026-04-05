---
read_when:
    - Quieres instalaciones reproducibles y reversibles
    - Ya usas Nix/NixOS/Home Manager
    - Quieres todo fijado y gestionado de forma declarativa
summary: Instala OpenClaw de forma declarativa con Nix
title: Nix
x-i18n:
    generated_at: "2026-04-05T12:46:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 14e1e73533db1350d82d3a786092b4328121a082dfeeedee7c7574021dada546
    source_path: install/nix.md
    workflow: 15
---

# Instalación con Nix

Instala OpenClaw de forma declarativa con **[nix-openclaw](https://github.com/openclaw/nix-openclaw)**, un módulo de Home Manager con todo incluido.

<Info>
El repositorio [nix-openclaw](https://github.com/openclaw/nix-openclaw) es la fuente de verdad para la instalación con Nix. Esta página es un resumen rápido.
</Info>

## Qué obtienes

- Gateway + app de macOS + herramientas (whisper, spotify, cámaras), todo fijado
- Servicio launchd que sobrevive a los reinicios
- Sistema de plugins con configuración declarativa
- Reversión instantánea: `home-manager switch --rollback`

## Inicio rápido

<Steps>
  <Step title="Instala Determinate Nix">
    Si Nix aún no está instalado, sigue las instrucciones del [instalador de Determinate Nix](https://github.com/DeterminateSystems/nix-installer).
  </Step>
  <Step title="Crea un flake local">
    Usa la plantilla agent-first del repositorio nix-openclaw:
    ```bash
    mkdir -p ~/code/openclaw-local
    # Copia templates/agent-first/flake.nix del repositorio nix-openclaw
    ```
  </Step>
  <Step title="Configura secretos">
    Configura el token de tu bot de mensajería y la API key del proveedor de modelos. Los archivos sin formato en `~/.secrets/` funcionan bien.
  </Step>
  <Step title="Rellena los marcadores de posición de la plantilla y aplica los cambios">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="Verifica">
    Confirma que el servicio launchd esté en ejecución y que tu bot responda a los mensajes.
  </Step>
</Steps>

Consulta el [README de nix-openclaw](https://github.com/openclaw/nix-openclaw) para ver todas las opciones del módulo y ejemplos.

## Comportamiento del tiempo de ejecución en modo Nix

Cuando `OPENCLAW_NIX_MODE=1` está definido (automático con nix-openclaw), OpenClaw entra en un modo determinista que deshabilita los flujos de instalación automática.

También puedes definirlo manualmente:

```bash
export OPENCLAW_NIX_MODE=1
```

En macOS, la app GUI no hereda automáticamente las variables de entorno del shell. En su lugar, habilita el modo Nix mediante defaults:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### Qué cambia en modo Nix

- Se deshabilitan los flujos de instalación automática y automutación
- Las dependencias faltantes muestran mensajes de corrección específicos de Nix
- La UI muestra un banner de solo lectura para el modo Nix

### Rutas de configuración y estado

OpenClaw lee la configuración JSON5 desde `OPENCLAW_CONFIG_PATH` y almacena los datos mutables en `OPENCLAW_STATE_DIR`. Al ejecutarse bajo Nix, define estos valores explícitamente en ubicaciones gestionadas por Nix para que el estado y la configuración de tiempo de ejecución permanezcan fuera del almacén inmutable.

| Variable               | Predeterminado                          |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

## Relacionado

- [nix-openclaw](https://github.com/openclaw/nix-openclaw) -- guía completa de configuración
- [Wizard](/es/start/wizard) -- configuración de la CLI sin Nix
- [Docker](/install/docker) -- configuración en contenedor
