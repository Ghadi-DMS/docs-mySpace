---
read_when:
    - Ejecutas OpenClaw con Docker a menudo y quieres comandos cotidianos más cortos
    - Quieres una capa auxiliar para panel, registros, configuración de token y flujos de emparejamiento
summary: Utilidades de shell de ClawDock para instalaciones de OpenClaw basadas en Docker
title: ClawDock
x-i18n:
    generated_at: "2026-04-05T12:44:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93d67d1d979450d8c9c11854d2f40977c958f1c300e75a5c42ce4c31de86735a
    source_path: install/clawdock.md
    workflow: 15
---

# ClawDock

ClawDock es una pequeña capa de utilidades de shell para instalaciones de OpenClaw basadas en Docker.

Te ofrece comandos cortos como `clawdock-start`, `clawdock-dashboard` y `clawdock-fix-token` en lugar de invocaciones más largas de `docker compose ...`.

Si todavía no has configurado Docker, empieza por [Docker](/install/docker).

## Instalación

Usa la ruta canónica del auxiliar:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

Si anteriormente instalaste ClawDock desde `scripts/shell-helpers/clawdock-helpers.sh`, vuelve a instalarlo desde la nueva ruta `scripts/clawdock/clawdock-helpers.sh`. La ruta raw antigua de GitHub se eliminó.

## Qué obtienes

### Operaciones básicas

| Comando            | Descripción            |
| ------------------ | ---------------------- |
| `clawdock-start`   | Iniciar el gateway      |
| `clawdock-stop`    | Detener el gateway       |
| `clawdock-restart` | Reiniciar el gateway    |
| `clawdock-status`  | Comprobar el estado del contenedor |
| `clawdock-logs`    | Seguir los registros del gateway    |

### Acceso al contenedor

| Comando                   | Descripción                                   |
| ------------------------- | --------------------------------------------- |
| `clawdock-shell`          | Abrir un shell dentro del contenedor del gateway     |
| `clawdock-cli <command>`  | Ejecutar comandos de la CLI de OpenClaw en Docker           |
| `clawdock-exec <command>` | Ejecutar un comando arbitrario en el contenedor |

### UI web y emparejamiento

| Comando                 | Descripción                  |
| ----------------------- | ---------------------------- |
| `clawdock-dashboard`    | Abrir la URL de la UI de Control      |
| `clawdock-devices`      | Enumerar emparejamientos de dispositivos pendientes |
| `clawdock-approve <id>` | Aprobar una solicitud de emparejamiento    |

### Configuración y mantenimiento

| Comando              | Descripción                                      |
| -------------------- | ------------------------------------------------ |
| `clawdock-fix-token` | Configurar el token del gateway dentro del contenedor |
| `clawdock-update`    | Extraer, reconstruir y reiniciar                       |
| `clawdock-rebuild`   | Reconstruir solo la imagen Docker                    |
| `clawdock-clean`     | Eliminar contenedores y volúmenes                    |

### Utilidades

| Comando                | Descripción                             |
| ---------------------- | --------------------------------------- |
| `clawdock-health`      | Ejecutar una comprobación de estado del gateway              |
| `clawdock-token`       | Imprimir el token del gateway                 |
| `clawdock-cd`          | Ir al directorio del proyecto OpenClaw  |
| `clawdock-config`      | Abrir `~/.openclaw`                      |
| `clawdock-show-config` | Imprimir archivos de configuración con valores redactados |
| `clawdock-workspace`   | Abrir el directorio del espacio de trabajo            |

## Flujo inicial

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

Si el navegador dice que se requiere emparejamiento:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## Configuración y secretos

ClawDock funciona con la misma división de configuración de Docker descrita en [Docker](/install/docker):

- `<project>/.env` para valores específicos de Docker como nombre de imagen, puertos y el token del gateway
- `~/.openclaw/.env` para claves de proveedor respaldadas por entorno y tokens de bot
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` para la autenticación OAuth/clave de API de proveedor almacenada
- `~/.openclaw/openclaw.json` para configuración de comportamiento

Usa `clawdock-show-config` cuando quieras inspeccionar rápidamente los archivos `.env` y `openclaw.json`. Redacta los valores de `.env` en su salida impresa.

## Páginas relacionadas

- [Docker](/install/docker)
- [Docker VM Runtime](/install/docker-vm-runtime)
- [Updating](/install/updating)
