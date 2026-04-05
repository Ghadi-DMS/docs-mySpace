---
read_when:
    - Quieres entender OAuth de extremo a extremo en OpenClaw
    - Tuviste problemas de invalidación de tokens o cierre de sesión
    - Quieres flujos de autenticación de Claude CLI u OAuth
    - Quieres múltiples cuentas o enrutamiento de perfiles
summary: 'OAuth en OpenClaw: intercambio de tokens, almacenamiento y patrones de múltiples cuentas'
title: OAuth
x-i18n:
    generated_at: "2026-04-05T12:40:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0b364be2182fcf9082834450f39aecc0913c85fb03237eec1228a589d4851dcd
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

OpenClaw admite “autenticación por suscripción” mediante OAuth para proveedores que la ofrecen
(en particular **OpenAI Codex (ChatGPT OAuth)**). Para suscripciones de Anthropic, la nueva
configuración debe usar la ruta de inicio de sesión local de **Claude CLI** en el host del gateway, pero
Anthropic distingue entre el uso directo de Claude Code y la ruta de reutilización de OpenClaw.
La documentación pública de Claude Code de Anthropic dice que el uso directo de Claude Code se mantiene
dentro de los límites de la suscripción de Claude. Por separado, Anthropic notificó a los usuarios
de OpenClaw el **4 de abril de 2026 a las 12:00 p. m. PT / 8:00 p. m. BST** que OpenClaw cuenta como
un arnés de terceros y ahora requiere **Extra Usage** para ese tráfico.
OpenAI Codex OAuth es compatible explícitamente para su uso en herramientas externas como
OpenClaw. Esta página explica:

Para Anthropic en producción, la autenticación con clave de API es la opción recomendada más segura.

- cómo funciona el **intercambio** de tokens OAuth (PKCE)
- dónde se **almacenan** los tokens (y por qué)
- cómo manejar **múltiples cuentas** (perfiles + anulaciones por sesión)

OpenClaw también admite **plugins de proveedor** que incorporan sus propios flujos de OAuth o clave de API.
Ejecútalos mediante:

```bash
openclaw models auth login --provider <id>
```

## El sumidero de tokens (por qué existe)

Los proveedores de OAuth suelen emitir un **nuevo token de actualización** durante los flujos de inicio de sesión o actualización. Algunos proveedores (o clientes OAuth) pueden invalidar tokens de actualización antiguos cuando se emite uno nuevo para el mismo usuario/aplicación.

Síntoma práctico:

- inicias sesión mediante OpenClaw _y_ mediante Claude Code / Codex CLI → uno de ellos termina “cerrando sesión” aleatoriamente más tarde

Para reducir eso, OpenClaw trata `auth-profiles.json` como un **sumidero de tokens**:

- el entorno de ejecución lee las credenciales desde **un solo lugar**
- podemos mantener múltiples perfiles y enrutarlos de forma determinista
- cuando las credenciales se reutilizan desde una CLI externa como Codex CLI, OpenClaw
  las refleja con procedencia y vuelve a leer esa fuente externa en lugar de
  rotar él mismo el token de actualización

## Almacenamiento (dónde viven los tokens)

Los secretos se almacenan **por agente**:

- Perfiles de autenticación (OAuth + claves de API + referencias opcionales a nivel de valor): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Archivo heredado de compatibilidad: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (las entradas estáticas `api_key` se limpian cuando se descubren)

Archivo heredado solo para importación (todavía compatible, pero no es el almacén principal):

- `~/.openclaw/credentials/oauth.json` (se importa a `auth-profiles.json` en el primer uso)

Todo lo anterior también respeta `$OPENCLAW_STATE_DIR` (anulación del directorio de estado). Referencia completa: [/gateway/configuration](/gateway/configuration-reference#auth-storage)

Para referencias de secretos estáticos y el comportamiento de activación de instantáneas en tiempo de ejecución, consulta [Gestión de secretos](/gateway/secrets).

## Compatibilidad heredada de tokens de Anthropic

<Warning>
La documentación pública de Claude Code de Anthropic dice que el uso directo de Claude Code se mantiene dentro
de los límites de la suscripción de Claude. Por separado, Anthropic informó a los usuarios de OpenClaw el
**4 de abril de 2026 a las 12:00 p. m. PT / 8:00 p. m. BST** que **OpenClaw cuenta como un
arnés de terceros**. Los perfiles de tokens existentes de Anthropic siguen siendo técnicamente
utilizables en OpenClaw, pero Anthropic dice que la ruta de OpenClaw ahora requiere **Extra
Usage** (pago por uso facturado por separado de la suscripción) para ese tráfico.

Para la documentación actual de Anthropic sobre planes directos de Claude Code, consulta [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
y [Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

Si quieres otras opciones de estilo suscripción en OpenClaw, consulta [OpenAI
Codex](/providers/openai), [Qwen Cloud Coding
Plan](/providers/qwen), [MiniMax Coding Plan](/providers/minimax),
y [Z.AI / GLM Coding Plan](/providers/glm).
</Warning>

OpenClaw ahora vuelve a exponer el token de configuración de Anthropic como una ruta heredada/manual.
El aviso de facturación específico de Anthropic para OpenClaw sigue aplicándose a esa ruta, así que
úsala esperando que Anthropic requiera **Extra Usage** para
el tráfico de inicio de sesión de Claude impulsado por OpenClaw.

## Migración a Anthropic Claude CLI

Si Claude CLI ya está instalado y con sesión iniciada en el host del gateway, puedes
cambiar la selección de modelos de Anthropic al backend local de CLI. Esta es una
ruta compatible de OpenClaw cuando quieres reutilizar un inicio de sesión local de Claude CLI en el
mismo host.

Requisitos previos:

- el binario `claude` está instalado en el host del gateway
- Claude CLI ya está autenticado allí mediante `claude auth login`

Comando de migración:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

Atajo de onboarding:

```bash
openclaw onboard --auth-choice anthropic-cli
```

Esto conserva los perfiles de autenticación de Anthropic existentes para una posible reversión, pero reescribe la
ruta principal del modelo predeterminado de `anthropic/...` a `claude-cli/...`, reescribe los
fallbacks coincidentes de Anthropic Claude y añade entradas coincidentes de lista de permitidos `claude-cli/...` en `agents.defaults.models`.

Verifica con:

```bash
openclaw models status
```

## Intercambio OAuth (cómo funciona el inicio de sesión)

Los flujos interactivos de inicio de sesión de OpenClaw están implementados en `@mariozechner/pi-ai` y conectados a los asistentes/comandos.

### Anthropic Claude CLI

Forma del flujo:

Ruta de Claude CLI:

1. inicia sesión con `claude auth login` en el host del gateway
2. ejecuta `openclaw models auth login --provider anthropic --method cli --set-default`
3. no almacena ningún perfil de autenticación nuevo; cambia la selección del modelo a `claude-cli/...`
4. conserva los perfiles de autenticación de Anthropic existentes para una posible reversión

La documentación pública de Claude Code de Anthropic describe este flujo de
inicio de sesión directo de suscripción de Claude para `claude`. OpenClaw puede reutilizar ese inicio de sesión local, pero
Anthropic clasifica por separado la ruta controlada por OpenClaw como uso de arnés de terceros a efectos de facturación.

Ruta del asistente interactivo:

- `openclaw onboard` / `openclaw configure` → opción de autenticación `anthropic-cli`

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth es compatible explícitamente para usarse fuera de Codex CLI, incluidos los flujos de OpenClaw.

Forma del flujo (PKCE):

1. genera el verificador/desafío PKCE + `state` aleatorio
2. abre `https://auth.openai.com/oauth/authorize?...`
3. intenta capturar el callback en `http://127.0.0.1:1455/auth/callback`
4. si el callback no puede enlazarse (o estás en remoto/sin interfaz), pega la URL/código de redirección
5. intercambia en `https://auth.openai.com/oauth/token`
6. extrae `accountId` del token de acceso y almacena `{ access, refresh, expires, accountId }`

La ruta del asistente es `openclaw onboard` → opción de autenticación `openai-codex`.

## Actualización + caducidad

Los perfiles almacenan una marca de tiempo `expires`.

En tiempo de ejecución:

- si `expires` está en el futuro → usa el token de acceso almacenado
- si está caducado → actualiza (bajo un bloqueo de archivo) y sobrescribe las credenciales almacenadas
- excepción: las credenciales reutilizadas de una CLI externa siguen gestionadas externamente; OpenClaw
  vuelve a leer el almacén de autenticación de la CLI y nunca consume por sí mismo el token de actualización copiado

El flujo de actualización es automático; por lo general no necesitas gestionar los tokens manualmente.

## Múltiples cuentas (perfiles) + enrutamiento

Dos patrones:

### 1) Preferido: agentes separados

Si quieres que “personal” y “trabajo” nunca interactúen, usa agentes aislados (sesiones + credenciales + espacio de trabajo separados):

```bash
openclaw agents add work
openclaw agents add personal
```

Luego configura la autenticación por agente (asistente) y enruta los chats al agente correcto.

### 2) Avanzado: múltiples perfiles en un solo agente

`auth-profiles.json` admite múltiples ID de perfil para el mismo proveedor.

Elige qué perfil se usa:

- globalmente mediante el orden de configuración (`auth.order`)
- por sesión mediante `/model ...@<profileId>`

Ejemplo (anulación por sesión):

- `/model Opus@anthropic:work`

Cómo ver qué ID de perfil existen:

- `openclaw channels list --json` (muestra `auth[]`)

Documentación relacionada:

- [/concepts/model-failover](/concepts/model-failover) (reglas de rotación + enfriamiento)
- [/tools/slash-commands](/tools/slash-commands) (superficie de comandos)

## Relacionado

- [Autenticación](/gateway/authentication) — resumen de autenticación de proveedores de modelos
- [Secretos](/gateway/secrets) — almacenamiento de credenciales y SecretRef
- [Referencia de configuración](/gateway/configuration-reference#auth-storage) — claves de configuración de autenticación
