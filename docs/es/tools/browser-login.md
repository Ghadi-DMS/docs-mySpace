---
read_when:
    - Necesitas iniciar sesión en sitios para automatización del navegador
    - Quieres publicar actualizaciones en X/Twitter
summary: Inicios de sesión manuales para automatización del navegador + publicación en X/Twitter
title: Inicio de sesión en el navegador
x-i18n:
    generated_at: "2026-04-05T12:54:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: de40685c70f1c141dba98e6dadc2c6f3a2b3b6d98c89ef8404144c9d178bb763
    source_path: tools/browser-login.md
    workflow: 15
---

# Inicio de sesión en el navegador + publicación en X/Twitter

## Inicio de sesión manual (recomendado)

Cuando un sitio requiera inicio de sesión, **inicia sesión manualmente** en el perfil del navegador del **host** (el navegador de openclaw).

**No** le des tus credenciales al modelo. Los inicios de sesión automatizados suelen activar defensas anti-bot y pueden bloquear la cuenta.

Volver a la documentación principal del navegador: [Browser](/tools/browser).

## ¿Qué perfil de Chrome se usa?

OpenClaw controla un **perfil de Chrome dedicado** (llamado `openclaw`, con una interfaz teñida de naranja). Este perfil está separado de tu perfil de navegador diario.

Para las llamadas de la herramienta de navegador del agente:

- Opción predeterminada: el agente debe usar su navegador `openclaw` aislado.
- Usa `profile="user"` solo cuando importen las sesiones existentes con sesión iniciada y el usuario esté en la computadora para hacer clic o aprobar cualquier aviso de conexión.
- Si tienes varios perfiles de navegador de usuario, especifica el perfil explícitamente en lugar de adivinar.

Dos formas sencillas de acceder a él:

1. **Pídele al agente que abra el navegador** y luego inicia sesión tú mismo.
2. **Ábrelo mediante la CLI**:

```bash
openclaw browser start
openclaw browser open https://x.com
```

Si tienes varios perfiles, pasa `--browser-profile <name>` (el valor predeterminado es `openclaw`).

## X/Twitter: flujo recomendado

- **Leer/buscar/hilos:** usa el navegador del **host** (inicio de sesión manual).
- **Publicar actualizaciones:** usa el navegador del **host** (inicio de sesión manual).

## Sandboxing + acceso al navegador del host

Las sesiones de navegador en sandbox son **más propensas** a activar la detección de bots. Para X/Twitter (y otros sitios estrictos), usa preferentemente el navegador del **host**.

Si el agente está en sandbox, la herramienta de navegador usa el sandbox de forma predeterminada. Para permitir el control del host:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

Luego apunta al navegador del host:

```bash
openclaw browser open https://x.com --browser-profile openclaw --target host
```

O desactiva el sandboxing para el agente que publica actualizaciones.
