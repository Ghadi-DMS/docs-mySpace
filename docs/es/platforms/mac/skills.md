---
read_when:
    - Actualizar la UI de configuración de Skills de macOS
    - Cambiar la restricción o el comportamiento de instalación de Skills
summary: UI de configuración de Skills de macOS y estado respaldado por gateway
title: Skills (macOS)
x-i18n:
    generated_at: "2026-04-05T12:48:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7ffd6744646d2c8770fa12a5e511f84a40b5ece67181139250ec4cc4301b49b8
    source_path: platforms/mac/skills.md
    workflow: 15
---

# Skills (macOS)

La app de macOS muestra las Skills de OpenClaw a través del gateway; no analiza las Skills localmente.

## Origen de datos

- `skills.status` (gateway) devuelve todas las Skills más la elegibilidad y los requisitos faltantes
  (incluidos los bloqueos por lista de permitidos para las Skills incluidas).
- Los requisitos se derivan de `metadata.openclaw.requires` en cada `SKILL.md`.

## Acciones de instalación

- `metadata.openclaw.install` define opciones de instalación (brew/node/go/uv).
- La app llama a `skills.install` para ejecutar instaladores en el host del gateway.
- Los hallazgos integrados `critical` de código peligroso bloquean `skills.install` de forma predeterminada; los hallazgos sospechosos siguen mostrando solo una advertencia. La anulación peligrosa existe en la solicitud del gateway, pero el flujo predeterminado de la app sigue fallando de forma cerrada.
- Si todas las opciones de instalación son `download`, el gateway muestra todas las opciones
  de descarga.
- En caso contrario, el gateway elige un instalador preferido usando las preferencias actuales
  de instalación y los binarios del host: primero Homebrew cuando
  `skills.install.preferBrew` está habilitado y existe `brew`, luego `uv`, después el
  gestor de node configurado en `skills.install.nodeManager`, y luego
  respaldos posteriores como `go` o `download`.
- Las etiquetas de instalación de Node reflejan el gestor de node configurado,
  incluido `yarn`.

## Variables de entorno/claves API

- La app almacena las claves en `~/.openclaw/openclaw.json` bajo `skills.entries.<skillKey>`.
- `skills.update` aplica parches a `enabled`, `apiKey` y `env`.

## Modo remoto

- La instalación y las actualizaciones de configuración se realizan en el host del gateway (no en la Mac local).
