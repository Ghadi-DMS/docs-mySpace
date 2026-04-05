---
read_when:
    - Agregar o modificar la CLI de modelos (models list/set/scan/aliases/fallbacks)
    - Cambiar el comportamiento de respaldo de modelos o la experiencia de selección
    - Actualizar sondeos de escaneo de modelos (tools/images)
summary: 'CLI de modelos: listar, establecer, alias, respaldos, escaneo, estado'
title: CLI de modelos
x-i18n:
    generated_at: "2026-04-05T12:40:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: e08f7e50da263895dae2bd2b8dc327972ea322615f8d1918ddbd26bb0fb24840
    source_path: concepts/models.md
    workflow: 15
---

# CLI de modelos

Consulta [/concepts/model-failover](/concepts/model-failover) para la
rotación de perfiles de autenticación, los tiempos de espera y cómo interactúan con los respaldos.
Resumen rápido de proveedores + ejemplos: [/concepts/model-providers](/concepts/model-providers).

## Cómo funciona la selección de modelos

OpenClaw selecciona modelos en este orden:

1. Modelo **primario** (`agents.defaults.model.primary` o `agents.defaults.model`).
2. **Respaldos** en `agents.defaults.model.fallbacks` (en orden).
3. El **failover de autenticación del proveedor** ocurre dentro de un proveedor antes de pasar al
   siguiente modelo.

Relacionado:

- `agents.defaults.models` es la lista de permitidos/catálogo de modelos que OpenClaw puede usar (más alias).
- `agents.defaults.imageModel` se usa **solo cuando** el modelo primario no puede aceptar imágenes.
- `agents.defaults.pdfModel` es usado por la herramienta `pdf`. Si se omite, la herramienta
  recurre a `agents.defaults.imageModel`, y luego al modelo de sesión/predeterminado resuelto.
- `agents.defaults.imageGenerationModel` es usado por la capacidad compartida de generación de imágenes. Si se omite, `image_generate` aún puede inferir un proveedor predeterminado respaldado por autenticación. Primero intenta el proveedor predeterminado actual y luego los proveedores de generación de imágenes registrados restantes en orden por id de proveedor. Si estableces un proveedor/modelo específico, configura también la autenticación/clave API de ese proveedor.
- `agents.defaults.videoGenerationModel` es usado por la capacidad compartida de generación de video. A diferencia de la generación de imágenes, hoy esto no infiere un proveedor predeterminado. Establece un `provider/model` explícito como `qwen/wan2.6-t2v` y configura también la autenticación/clave API de ese proveedor.
- Los valores predeterminados por agente pueden anular `agents.defaults.model` mediante `agents.list[].model` más asociaciones (consulta [/concepts/multi-agent](/concepts/multi-agent)).

## Política rápida de modelos

- Establece tu modelo primario al modelo más potente de última generación que tengas disponible.
- Usa respaldos para tareas sensibles a costo/latencia y chats de menor importancia.
- Para agentes con herramientas habilitadas o entradas no confiables, evita niveles de modelos antiguos/más débiles.

## Onboarding (recomendado)

Si no quieres editar manualmente la configuración, ejecuta el onboarding:

```bash
openclaw onboard
```

Puede configurar modelo + autenticación para proveedores comunes, incluidas suscripciones a **OpenAI Code (Codex)**
(OAuth) y **Anthropic** (clave API o Claude CLI).

## Claves de configuración (resumen)

- `agents.defaults.model.primary` y `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` y `agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` y `agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` y `agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` y `agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models` (lista de permitidos + alias + parámetros del proveedor)
- `models.providers` (proveedores personalizados escritos en `models.json`)

Las referencias de modelo se normalizan a minúsculas. Los alias de proveedor como `z.ai/*` se normalizan
a `zai/*`.

Los ejemplos de configuración de proveedor (incluido OpenCode) están en
[/providers/opencode](/providers/opencode).

## "Model is not allowed" (y por qué se detienen las respuestas)

Si `agents.defaults.models` está establecido, se convierte en la **lista de permitidos** para `/model` y para
las anulaciones de sesión. Cuando una persona usuaria selecciona un modelo que no está en esa lista de permitidos,
OpenClaw devuelve:

```
Model "provider/model" is not allowed. Use /model to list available models.
```

Esto ocurre **antes** de que se genere una respuesta normal, por lo que el mensaje puede dar la impresión
de que “no respondió”. La solución es:

- Añadir el modelo a `agents.defaults.models`, o
- Vaciar la lista de permitidos (eliminar `agents.defaults.models`), o
- Elegir un modelo de `/model list`.

Ejemplo de configuración de lista de permitidos:

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-6" },
    models: {
      "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## Cambiar modelos en el chat (`/model`)

Puedes cambiar modelos para la sesión actual sin reiniciar:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

Notas:

- `/model` (y `/model list`) es un selector compacto numerado (familia de modelos + proveedores disponibles).
- En Discord, `/model` y `/models` abren un selector interactivo con menús desplegables de proveedor y modelo, más un paso de envío.
- `/model <#>` selecciona desde ese selector.
- `/model` conserva inmediatamente la nueva selección de sesión.
- Si el agente está inactivo, la siguiente ejecución usa de inmediato el nuevo modelo.
- Si ya hay una ejecución activa, OpenClaw marca un cambio en vivo como pendiente y solo reinicia con el nuevo modelo en un punto de reintento limpio.
- Si ya ha comenzado la actividad de herramientas o la salida de la respuesta, el cambio pendiente puede quedar en cola hasta una oportunidad posterior de reintento o el siguiente turno de la persona usuaria.
- `/model status` es la vista detallada (candidatos de autenticación y, cuando está configurado, `baseUrl` del endpoint del proveedor + modo `api`).
- Las referencias de modelo se analizan dividiéndolas en la **primera** `/`. Usa `provider/model` al escribir `/model <ref>`.
- Si el propio ID del modelo contiene `/` (estilo OpenRouter), debes incluir el prefijo del proveedor (ejemplo: `/model openrouter/moonshotai/kimi-k2`).
- Si omites el proveedor, OpenClaw resuelve la entrada en este orden:
  1. coincidencia de alias
  2. coincidencia única de proveedor configurado para ese id exacto de modelo sin prefijo
  3. respaldo obsoleto al proveedor predeterminado configurado
     Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw
     recurre al primer proveedor/modelo configurado para evitar
     mostrar un valor predeterminado obsoleto de un proveedor eliminado.

Comportamiento/configuración completa del comando: [Comandos slash](/tools/slash-commands).

## Comandos de CLI

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models` (sin subcomando) es un atajo de `models status`.

### `models list`

Muestra los modelos configurados de forma predeterminada. Indicadores útiles:

- `--all`: catálogo completo
- `--local`: solo proveedores locales
- `--provider <name>`: filtrar por proveedor
- `--plain`: un modelo por línea
- `--json`: salida legible por máquinas

### `models status`

Muestra el modelo primario resuelto, los respaldos, el modelo de imagen y una vista general de autenticación
de los proveedores configurados. También muestra el estado de expiración de OAuth para perfiles encontrados
en el almacén de autenticación (advierte dentro de las 24 h de forma predeterminada). `--plain` imprime solo el
modelo primario resuelto.
El estado de OAuth siempre se muestra (y se incluye en la salida de `--json`). Si un proveedor configurado
no tiene credenciales, `models status` imprime una sección de **Missing auth**.
JSON incluye `auth.oauth` (ventana de advertencia + perfiles) y `auth.providers`
(autenticación efectiva por proveedor).
Usa `--check` para automatización (salida `1` cuando falta/expiró, `2` cuando está por expirar).
Usa `--probe` para comprobaciones de autenticación en vivo; las filas de sondeo pueden venir de perfiles de autenticación, credenciales de entorno o `models.json`.
Si un `auth.order.<provider>` explícito omite un perfil almacenado, el sondeo informa
`excluded_by_auth_order` en lugar de probarlo. Si existe autenticación pero no se puede resolver
ningún modelo sondeable para ese proveedor, el sondeo informa `status: no_model`.

La elección de autenticación depende del proveedor/cuenta. Para hosts de gateway siempre activos, las
claves API suelen ser las más predecibles; también se admite la reutilización de Claude CLI y perfiles OAuth/token existentes de Anthropic.

Ejemplo (Claude CLI):

```bash
claude auth login
openclaw models status
```

## Escaneo (modelos gratuitos de OpenRouter)

`openclaw models scan` inspecciona el **catálogo de modelos gratuitos** de OpenRouter y puede
sondear opcionalmente los modelos para comprobar compatibilidad con herramientas e imágenes.

Indicadores clave:

- `--no-probe`: omitir sondeos en vivo (solo metadatos)
- `--min-params <b>`: tamaño mínimo de parámetros (miles de millones)
- `--max-age-days <days>`: omitir modelos antiguos
- `--provider <name>`: filtro de prefijo de proveedor
- `--max-candidates <n>`: tamaño de la lista de respaldos
- `--set-default`: establecer `agents.defaults.model.primary` en la primera selección
- `--set-image`: establecer `agents.defaults.imageModel.primary` en la primera selección de imagen

El sondeo requiere una clave API de OpenRouter (de perfiles de autenticación o
`OPENROUTER_API_KEY`). Sin clave, usa `--no-probe` para enumerar solo candidatos.

Los resultados del escaneo se clasifican por:

1. Compatibilidad con imágenes
2. Latencia de herramientas
3. Tamaño de contexto
4. Cantidad de parámetros

Entrada

- Lista `/models` de OpenRouter (filtro `:free`)
- Requiere clave API de OpenRouter de perfiles de autenticación o `OPENROUTER_API_KEY` (consulta [/environment](/help/environment))
- Filtros opcionales: `--max-age-days`, `--min-params`, `--provider`, `--max-candidates`
- Controles de sondeo: `--timeout`, `--concurrency`

Cuando se ejecuta en un TTY, puedes seleccionar respaldos de forma interactiva. En modo no interactivo,
pasa `--yes` para aceptar los valores predeterminados.

## Registro de modelos (`models.json`)

Los proveedores personalizados en `models.providers` se escriben en `models.json` dentro del
directorio del agente (predeterminado `~/.openclaw/agents/<agentId>/agent/models.json`). Este archivo
se fusiona de forma predeterminada a menos que `models.mode` se establezca en `replace`.

Precedencia del modo de fusión para id de proveedor coincidentes:

- Un `baseUrl` no vacío ya presente en `models.json` del agente tiene prioridad.
- Un `apiKey` no vacío en `models.json` del agente tiene prioridad solo cuando ese proveedor no está administrado por SecretRef en el contexto actual de configuración/perfil de autenticación.
- Los valores `apiKey` del proveedor administrados por SecretRef se actualizan a partir de marcadores de origen (`ENV_VAR_NAME` para referencias env, `secretref-managed` para referencias file/exec) en lugar de conservar secretos resueltos.
- Los valores de encabezado del proveedor administrados por SecretRef se actualizan a partir de marcadores de origen (`secretref-env:ENV_VAR_NAME` para referencias env, `secretref-managed` para referencias file/exec).
- Si `apiKey`/`baseUrl` del agente están vacíos o ausentes, se recurre a la configuración `models.providers`.
- Los demás campos del proveedor se actualizan a partir de la configuración y los datos normalizados del catálogo.

La persistencia de marcadores es autoritativa desde el origen: OpenClaw escribe los marcadores a partir de la instantánea activa de configuración de origen (antes de la resolución), no a partir de los valores secretos resueltos en tiempo de ejecución.
Esto se aplica siempre que OpenClaw regenera `models.json`, incluidas rutas controladas por comandos como `openclaw agent`.

## Relacionado

- [Proveedores de modelos](/concepts/model-providers) — enrutamiento y autenticación de proveedores
- [Failover de modelos](/concepts/model-failover) — cadenas de respaldo
- [Generación de imágenes](/tools/image-generation) — configuración de modelos de imagen
- [Referencia de configuración](/gateway/configuration-reference#agent-defaults) — claves de configuración de modelos
