---
read_when:
    - Configurar una máquina nueva
    - Quieres “lo último y lo mejor” sin romper tu configuración personal
summary: Configuración avanzada y flujos de trabajo de desarrollo para OpenClaw
title: Configuración
x-i18n:
    generated_at: "2026-04-05T12:54:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: be4e280dde7f3a224345ca557ef2fb35a9c9db8520454ff63794ac6f8d4e71e7
    source_path: start/setup.md
    workflow: 15
---

# Configuración

<Note>
Si vas a configurarlo por primera vez, empieza con [Primeros pasos](/es/start/getting-started).
Para detalles de onboarding, consulta [Onboarding (CLI)](/es/start/wizard).
</Note>

## En resumen

- **La personalización vive fuera del repositorio:** `~/.openclaw/workspace` (espacio de trabajo) + `~/.openclaw/openclaw.json` (configuración).
- **Flujo de trabajo estable:** instala la app de macOS; deja que ejecute el Gateway incluido.
- **Flujo de trabajo de vanguardia:** ejecuta el Gateway tú mismo mediante `pnpm gateway:watch`, y luego deja que la app de macOS se conecte en modo Local.

## Requisitos previos (desde el código fuente)

- Se recomienda Node 24 (Node 22 LTS, actualmente `22.14+`, sigue siendo compatible)
- Se prefiere `pnpm` (o Bun si usas intencionalmente el [flujo de trabajo con Bun](/es/install/bun))
- Docker (opcional; solo para configuración en contenedores/e2e — consulta [Docker](/es/install/docker))

## Estrategia de personalización (para que las actualizaciones no perjudiquen)

Si quieres que esté “100 % adaptado a mí” _y_ tener actualizaciones fáciles, mantén tu personalización en:

- **Configuración:** `~/.openclaw/openclaw.json` (similar a JSON/JSON5)
- **Espacio de trabajo:** `~/.openclaw/workspace` (Skills, prompts, memorias; conviértelo en un repositorio git privado)

Inicialízalo una vez:

```bash
openclaw setup
```

Desde dentro de este repositorio, usa la entrada local de la CLI:

```bash
openclaw setup
```

Si todavía no tienes una instalación global, ejecútalo mediante `pnpm openclaw setup` (o `bun run openclaw setup` si estás usando el flujo de trabajo con Bun).

## Ejecutar el Gateway desde este repositorio

Después de `pnpm build`, puedes ejecutar directamente la CLI empaquetada:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## Flujo de trabajo estable (primero la app de macOS)

1. Instala e inicia **OpenClaw.app** (barra de menús).
2. Completa la lista de comprobación de onboarding/permisos (solicitudes TCC).
3. Asegúrate de que Gateway esté en **Local** y en ejecución (la app lo gestiona).
4. Vincula superficies (ejemplo: WhatsApp):

```bash
openclaw channels login
```

5. Verificación rápida:

```bash
openclaw health
```

Si el onboarding no está disponible en tu compilación:

- Ejecuta `openclaw setup`, luego `openclaw channels login`, y después inicia el Gateway manualmente (`openclaw gateway`).

## Flujo de trabajo de vanguardia (Gateway en una terminal)

Objetivo: trabajar en el Gateway de TypeScript, obtener recarga en caliente y mantener conectada la UI de la app de macOS.

### 0) (Opcional) Ejecutar también la app de macOS desde el código fuente

Si también quieres que la app de macOS esté en la versión de vanguardia:

```bash
./scripts/restart-mac.sh
```

### 1) Iniciar el Gateway de desarrollo

```bash
pnpm install
pnpm gateway:watch
```

`gateway:watch` ejecuta el gateway en modo watch y se recarga ante cambios relevantes en el código fuente,
la configuración y los metadatos de plugins incluidos.

Si usas intencionalmente el flujo de trabajo con Bun, los comandos equivalentes son:

```bash
bun install
bun run gateway:watch
```

### 2) Apuntar la app de macOS a tu Gateway en ejecución

En **OpenClaw.app**:

- Modo de conexión: **Local**
  La app se conectará al gateway en ejecución en el puerto configurado.

### 3) Verificar

- El estado del Gateway en la app debería mostrar **“Using existing gateway …”**
- O mediante la CLI:

```bash
openclaw health
```

### Problemas habituales

- **Puerto incorrecto:** Gateway WS usa de forma predeterminada `ws://127.0.0.1:18789`; mantén la app y la CLI en el mismo puerto.
- **Dónde vive el estado:**
  - Estado de canal/proveedor: `~/.openclaw/credentials/`
  - Perfiles de autenticación del modelo: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - Sesiones: `~/.openclaw/agents/<agentId>/sessions/`
  - Registros: `/tmp/openclaw/`

## Mapa de almacenamiento de credenciales

Usa esto al depurar autenticación o decidir qué respaldar:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Token del bot de Telegram**: config/env o `channels.telegram.tokenFile` (solo archivo normal; se rechazan enlaces simbólicos)
- **Token del bot de Discord**: config/env o SecretRef (proveedores env/file/exec)
- **Tokens de Slack**: config/env (`channels.slack.*`)
- **Listas de permitidos de emparejamiento**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (cuenta predeterminada)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (cuentas no predeterminadas)
- **Perfiles de autenticación del modelo**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **Carga útil opcional de secrets respaldados por archivo**: `~/.openclaw/secrets.json`
- **Importación heredada de OAuth**: `~/.openclaw/credentials/oauth.json`
  Más detalles: [Seguridad](/es/gateway/security#credential-storage-map).

## Actualización (sin destrozar tu configuración)

- Mantén `~/.openclaw/workspace` y `~/.openclaw/` como “tus cosas”; no pongas prompts/configuración personales en el repositorio `openclaw`.
- Para actualizar el código fuente: `git pull` + el paso de instalación de tu gestor de paquetes elegido (`pnpm install` de forma predeterminada; `bun install` para el flujo de trabajo con Bun) + sigue usando el comando `gateway:watch` correspondiente.

## Linux (servicio de usuario systemd)

Las instalaciones de Linux usan un servicio de usuario de systemd. De forma predeterminada, systemd detiene los
servicios de usuario al cerrar sesión o en inactividad, lo que mata el Gateway. Onboarding intenta habilitar
lingering por ti (puede pedir sudo). Si sigue desactivado, ejecuta:

```bash
sudo loginctl enable-linger $USER
```

Para servidores siempre activos o multiusuario, considera un servicio de **sistema** en lugar de un
servicio de usuario (no requiere lingering). Consulta [Manual operativo del Gateway](/es/gateway) para ver las notas sobre systemd.

## Documentación relacionada

- [Manual operativo del Gateway](/es/gateway) (flags, supervisión, puertos)
- [Configuración del Gateway](/es/gateway/configuration) (esquema de configuración + ejemplos)
- [Discord](/es/channels/discord) y [Telegram](/es/channels/telegram) (etiquetas de respuesta + ajustes de replyToMode)
- [Configuración del asistente OpenClaw](/start/openclaw)
- [app de macOS](/es/platforms/macos) (ciclo de vida del gateway)
