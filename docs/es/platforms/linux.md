---
read_when:
    - Buscando el estado de la app complementaria para Linux
    - Planificando cobertura de plataformas o contribuciones
summary: Compatibilidad con Linux + estado de la app complementaria
title: App de Linux
x-i18n:
    generated_at: "2026-04-05T12:48:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5dbfc89eb65e04347479fc6c9a025edec902fb0c544fb8d5bd09c24558ea03b1
    source_path: platforms/linux.md
    workflow: 15
---

# App de Linux

El Gateway tiene compatibilidad completa en Linux. **Node es el tiempo de ejecución recomendado**.
Bun no se recomienda para el Gateway (errores de WhatsApp/Telegram).

Las apps complementarias nativas para Linux están planificadas. Las contribuciones son bienvenidas si quieres ayudar a crear una.

## Ruta rápida para principiantes (VPS)

1. Instala Node 24 (recomendado; Node 22 LTS, actualmente `22.14+`, sigue funcionando por compatibilidad)
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. Desde tu portátil: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. Abre `http://127.0.0.1:18789/` y autentícate con el secreto compartido configurado (token por defecto; contraseña si configuras `gateway.auth.mode: "password"`)

Guía completa del servidor Linux: [Linux Server](/vps). Ejemplo paso a paso de VPS: [exe.dev](/install/exe-dev)

## Instalación

- [Getting Started](/es/start/getting-started)
- [Install & updates](/install/updating)
- Flujos opcionales: [Bun (experimental)](/install/bun), [Nix](/install/nix), [Docker](/install/docker)

## Gateway

- [Gateway runbook](/gateway)
- [Configuration](/gateway/configuration)

## Instalación del servicio Gateway (CLI)

Usa una de estas opciones:

```
openclaw onboard --install-daemon
```

O:

```
openclaw gateway install
```

O:

```
openclaw configure
```

Selecciona **Gateway service** cuando se te solicite.

Reparar/migrar:

```
openclaw doctor
```

## Control del sistema (unidad de usuario de systemd)

OpenClaw instala por defecto un servicio de usuario de systemd. Usa un servicio del **sistema** para servidores compartidos o siempre activos. `openclaw gateway install` y
`openclaw onboard --install-daemon` ya generan por ti la unidad canónica actual;
escríbela a mano solo cuando necesites una configuración personalizada del sistema/gestor de servicios. Toda la guía del servicio está en el [Gateway runbook](/gateway).

Configuración mínima:

Crea `~/.config/systemd/user/openclaw-gateway[-<profile>].service`:

```
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

Habilítalo:

```
systemctl --user enable --now openclaw-gateway[-<profile>].service
```
