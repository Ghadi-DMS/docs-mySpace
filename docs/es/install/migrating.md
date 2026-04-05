---
read_when:
    - Vas a mover OpenClaw a una nueva laptop/servidor
    - Quieres conservar sesiones, autenticación e inicios de sesión de canales (WhatsApp, etc.)
summary: Mover (migrar) una instalación de OpenClaw de una máquina a otra
title: Guía de migración
x-i18n:
    generated_at: "2026-04-05T12:46:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 403f0b9677ce723c84abdbabfad20e0f70fd48392ebf23eabb7f8a111fd6a26d
    source_path: install/migrating.md
    workflow: 15
---

# Migrar OpenClaw a una nueva máquina

Esta guía mueve un gateway de OpenClaw a una nueva máquina sin volver a hacer el onboarding.

## Qué se migra

Cuando copias el **directorio de estado** (`~/.openclaw/` de forma predeterminada) y tu **espacio de trabajo**, conservas:

- **Configuración** -- `openclaw.json` y toda la configuración del gateway
- **Autenticación** -- `auth-profiles.json` por agente (claves API + OAuth), además de cualquier estado de canal/proveedor en `credentials/`
- **Sesiones** -- historial de conversaciones y estado del agente
- **Estado del canal** -- inicio de sesión de WhatsApp, sesión de Telegram, etc.
- **Archivos del espacio de trabajo** -- `MEMORY.md`, `USER.md`, Skills y prompts

<Tip>
Ejecuta `openclaw status` en la máquina antigua para confirmar la ruta de tu directorio de estado.
Los perfiles personalizados usan `~/.openclaw-<profile>/` o una ruta establecida mediante `OPENCLAW_STATE_DIR`.
</Tip>

## Pasos de migración

<Steps>
  <Step title="Detener el gateway y hacer una copia de seguridad">
    En la máquina **antigua**, detén el gateway para que los archivos no cambien durante la copia y luego archiva:

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    Si usas varios perfiles (por ejemplo, `~/.openclaw-work`), archiva cada uno por separado.

  </Step>

  <Step title="Instalar OpenClaw en la nueva máquina">
    [Instala](/install) la CLI (y Node si es necesario) en la nueva máquina.
    No pasa nada si el onboarding crea un `~/.openclaw/` nuevo; lo sobrescribirás a continuación.
  </Step>

  <Step title="Copiar el directorio de estado y el espacio de trabajo">
    Transfiere el archivo mediante `scp`, `rsync -a` o una unidad externa y luego extráelo:

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    Asegúrate de que se incluyeron los directorios ocultos y de que la propiedad de los archivos coincida con la persona usuaria que ejecutará el gateway.

  </Step>

  <Step title="Ejecutar doctor y verificar">
    En la nueva máquina, ejecuta [Doctor](/gateway/doctor) para aplicar migraciones de configuración y reparar servicios:

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

## Problemas comunes

<AccordionGroup>
  <Accordion title="Desajuste de perfil o directorio de estado">
    Si el gateway antiguo usaba `--profile` o `OPENCLAW_STATE_DIR` y el nuevo no,
    los canales aparecerán como desconectados y las sesiones estarán vacías.
    Inicia el gateway con el **mismo** perfil o directorio de estado que migraste y luego vuelve a ejecutar `openclaw doctor`.
  </Accordion>

  <Accordion title="Copiar solo openclaw.json">
    El archivo de configuración por sí solo no es suficiente. Los perfiles de autenticación del modelo viven en
    `agents/<agentId>/agent/auth-profiles.json`, y el estado del canal/proveedor sigue
    viviendo en `credentials/`. Migra siempre el **directorio de estado completo**.
  </Accordion>

  <Accordion title="Permisos y propiedad">
    Si copiaste como root o cambiaste de usuario, el gateway puede fallar al leer las credenciales.
    Asegúrate de que el directorio de estado y el espacio de trabajo pertenezcan a la persona usuaria que ejecuta el gateway.
  </Accordion>

  <Accordion title="Modo remoto">
    Si tu IU apunta a un gateway **remoto**, el host remoto es propietario de las sesiones y del espacio de trabajo.
    Migra el propio host del gateway, no tu laptop local. Consulta [FAQ](/help/faq#where-things-live-on-disk).
  </Accordion>

  <Accordion title="Secretos en las copias de seguridad">
    El directorio de estado contiene perfiles de autenticación, credenciales de canales y otro
    estado de proveedores.
    Guarda las copias de seguridad cifradas, evita canales de transferencia inseguros y rota las claves si sospechas exposición.
  </Accordion>
</AccordionGroup>

## Lista de verificación

En la nueva máquina, confirma:

- [ ] `openclaw status` muestra el gateway en ejecución
- [ ] Los canales siguen conectados (no hace falta volver a emparejar)
- [ ] El panel se abre y muestra las sesiones existentes
- [ ] Los archivos del espacio de trabajo (memoria, configuraciones) están presentes
