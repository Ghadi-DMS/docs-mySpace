---
read_when:
    - Quieres ajustar credenciales, dispositivos o valores predeterminados del agente de forma interactiva
summary: Referencia de la CLI para `openclaw configure` (solicitudes de configuración interactivas)
title: configure
x-i18n:
    generated_at: "2026-04-05T12:37:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 989569fdb8e1b31ce3438756b3ed9bf18e0c8baf611c5981643ba5925459c98f
    source_path: cli/configure.md
    workflow: 15
---

# `openclaw configure`

Solicitud interactiva para configurar credenciales, dispositivos y valores predeterminados del agente.

Nota: La sección **Model** ahora incluye una selección múltiple para la lista de permitidos
`agents.defaults.models` (lo que aparece en `/model` y en el selector de modelos).

Cuando configure se inicia desde una opción de autenticación del proveedor, los selectores
de modelo predeterminado y de lista de permitidos prefieren automáticamente ese proveedor. Para proveedores emparejados como
Volcengine/BytePlus, esta misma preferencia también coincide con sus variantes de
plan de código (`volcengine-plan/*`, `byteplus-plan/*`). Si el filtro del proveedor preferido
produjera una lista vacía, configure recurre al catálogo sin filtrar en lugar de mostrar un selector vacío.

Consejo: `openclaw config` sin subcomando abre el mismo asistente. Usa
`openclaw config get|set|unset` para ediciones no interactivas.

Para la búsqueda web, `openclaw configure --section web` te permite elegir un proveedor
y configurar sus credenciales. Algunos proveedores también muestran solicitudes de seguimiento
específicas del proveedor:

- **Grok** puede ofrecer una configuración opcional de `x_search` con la misma `XAI_API_KEY` y
  permitirte elegir un modelo de `x_search`.
- **Kimi** puede solicitar la región de la API de Moonshot (`api.moonshot.ai` frente a
  `api.moonshot.cn`) y el modelo predeterminado de búsqueda web de Kimi.

Relacionado:

- Referencia de configuración del gateway: [Configuración](/gateway/configuration)
- CLI de configuración: [Config](/cli/config)

## Opciones

- `--section <section>`: filtro de sección repetible

Secciones disponibles:

- `workspace`
- `model`
- `web`
- `gateway`
- `daemon`
- `channels`
- `plugins`
- `skills`
- `health`

Notas:

- Elegir dónde se ejecuta el Gateway siempre actualiza `gateway.mode`. Puedes seleccionar "Continue" sin otras secciones si eso es todo lo que necesitas.
- Los servicios orientados a canales (Slack/Discord/Matrix/Microsoft Teams) solicitan listas de permitidos de canales/salas durante la configuración. Puedes introducir nombres o IDs; el asistente resuelve nombres a IDs cuando es posible.
- Si ejecutas el paso de instalación del daemon, la autenticación por token requiere un token, y `gateway.auth.token` está gestionado por SecretRef; configure valida el SecretRef, pero no conserva valores de token en texto sin formato resueltos en los metadatos de entorno del servicio supervisor.
- Si la autenticación por token requiere un token y el SecretRef del token configurado no está resuelto, configure bloquea la instalación del daemon con una guía de corrección accionable.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está establecido, configure bloquea la instalación del daemon hasta que el modo se establezca explícitamente.

## Ejemplos

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```
