---
read_when:
    - Integrar herramientas que esperan OpenAI Chat Completions
summary: Expone un endpoint HTTP compatible con OpenAI `/v1/chat/completions` desde el Gateway
title: OpenAI Chat Completions
x-i18n:
    generated_at: "2026-04-05T12:42:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: c374b2f32ce693a8c752e2b0a2532c5f0299ed280f9a0e97b1a9d73bcec37b95
    source_path: gateway/openai-http-api.md
    workflow: 15
---

# OpenAI Chat Completions (HTTP)

El Gateway de OpenClaw puede servir un pequeño endpoint Chat Completions compatible con OpenAI.

Este endpoint está **deshabilitado de forma predeterminada**. Habilítalo primero en la configuración.

- `POST /v1/chat/completions`
- Mismo puerto que el Gateway (multiplexado WS + HTTP): `http://<gateway-host>:<port>/v1/chat/completions`

Cuando la superficie HTTP compatible con OpenAI del Gateway está habilitada, también sirve:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/responses`

Internamente, las solicitudes se ejecutan como una ejecución normal de agente del Gateway (la misma ruta de código que `openclaw agent`), por lo que el enrutamiento, los permisos y la configuración coinciden con tu Gateway.

## Autenticación

Usa la configuración de autenticación del Gateway.

Rutas comunes de autenticación HTTP:

- autenticación con secreto compartido (`gateway.auth.mode="token"` o `"password"`):
  `Authorization: Bearer <token-or-password>`
- autenticación HTTP de confianza con identidad (`gateway.auth.mode="trusted-proxy"`):
  enruta mediante el proxy con reconocimiento de identidad configurado y deja que inyecte
  los encabezados de identidad requeridos
- autenticación abierta en ingreso privado (`gateway.auth.mode="none"`):
  no se requiere encabezado de autenticación

Notas:

- Cuando `gateway.auth.mode="token"`, usa `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
- Cuando `gateway.auth.mode="password"`, usa `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
- Cuando `gateway.auth.mode="trusted-proxy"`, la solicitud HTTP debe provenir de una
  fuente de proxy confiable configurada que no sea loopback; los proxies loopback en el mismo host no
  cumplen este modo.
- Si `gateway.auth.rateLimit` está configurado y ocurren demasiados fallos de autenticación, el endpoint devuelve `429` con `Retry-After`.

## Límite de seguridad (importante)

Trata este endpoint como una superficie de **acceso completo de operador** para la instancia del gateway.

- La autenticación bearer HTTP aquí no es un modelo estrecho de alcance por usuario.
- Un token/contraseña válidos del Gateway para este endpoint deben tratarse como una credencial de propietario/operador.
- Las solicitudes se ejecutan a través de la misma ruta de agente del plano de control que las acciones confiables del operador.
- No hay un límite separado de herramientas para no propietarios/por usuario en este endpoint; una vez que quien llama supera la autenticación del Gateway aquí, OpenClaw trata a quien llama como un operador confiable para este gateway.
- Si la política del agente de destino permite herramientas sensibles, este endpoint puede usarlas.
- Mantén este endpoint solo en loopback/tailnet/ingreso privado; no lo expongas directamente a internet público.

Matriz de autenticación:

- `gateway.auth.mode="token"` o `"password"` + `Authorization: Bearer ...`
  - demuestra posesión del secreto compartido de operador del gateway
  - ignora `x-openclaw-scopes` más restringido
  - restablece el conjunto completo predeterminado de alcances del operador:
    `operator.admin`, `operator.approvals`, `operator.pairing`,
    `operator.read`, `operator.talk.secrets`, `operator.write`
  - trata los turnos de chat en este endpoint como turnos de remitente propietario
- modos HTTP de confianza con identidad (por ejemplo, autenticación por proxy confiable o `gateway.auth.mode="none"` en ingreso privado)
  - autentican una identidad externa confiable o un límite de implementación
  - respetan `x-openclaw-scopes` cuando el encabezado está presente
  - recurren al conjunto predeterminado normal de alcances del operador cuando el encabezado está ausente
  - solo pierden la semántica de propietario cuando quien llama restringe explícitamente los alcances y omite `operator.admin`

Consulta [Seguridad](/gateway/security) y [Acceso remoto](/gateway/remote).

## Contrato de modelo centrado en el agente

OpenClaw trata el campo OpenAI `model` como un **destino de agente**, no como un id sin procesar de modelo de proveedor.

- `model: "openclaw"` enruta al agente predeterminado configurado.
- `model: "openclaw/default"` también enruta al agente predeterminado configurado.
- `model: "openclaw/<agentId>"` enruta a un agente específico.

Encabezados de solicitud opcionales:

- `x-openclaw-model: <provider/model-or-bare-id>` reemplaza el modelo de backend para el agente seleccionado.
- `x-openclaw-agent-id: <agentId>` sigue siendo compatible como reemplazo de compatibilidad.
- `x-openclaw-session-key: <sessionKey>` controla completamente el enrutamiento de sesión.
- `x-openclaw-message-channel: <channel>` establece el contexto sintético del canal de entrada para prompts y políticas con reconocimiento de canal.

También se aceptan alias de compatibilidad:

- `model: "openclaw:<agentId>"`
- `model: "agent:<agentId>"`

## Habilitar el endpoint

Establece `gateway.http.endpoints.chatCompletions.enabled` en `true`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

## Deshabilitar el endpoint

Establece `gateway.http.endpoints.chatCompletions.enabled` en `false`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: false },
      },
    },
  },
}
```

## Comportamiento de la sesión

De forma predeterminada, el endpoint es **sin estado por solicitud** (se genera una nueva clave de sesión en cada llamada).

Si la solicitud incluye una cadena OpenAI `user`, el Gateway deriva una clave de sesión estable a partir de ella, de modo que las llamadas repetidas puedan compartir una sesión del agente.

## Por qué importa esta superficie

Este es el conjunto de compatibilidad de mayor impacto para frontends y herramientas autoalojados:

- La mayoría de las configuraciones de Open WebUI, LobeChat y LibreChat esperan `/v1/models`.
- Muchos sistemas RAG esperan `/v1/embeddings`.
- Los clientes de chat OpenAI existentes normalmente pueden comenzar con `/v1/chat/completions`.
- Los clientes más orientados a agentes prefieren cada vez más `/v1/responses`.

## Lista de modelos y enrutamiento de agentes

<AccordionGroup>
  <Accordion title="¿Qué devuelve `/v1/models`?">
    Una lista de destinos de agentes de OpenClaw.

    Los ids devueltos son `openclaw`, `openclaw/default` y entradas `openclaw/<agentId>`.
    Úsalos directamente como valores OpenAI `model`.

  </Accordion>
  <Accordion title="¿`/v1/models` lista agentes o subagentes?">
    Lista destinos de agentes de nivel superior, no modelos de proveedor de backend ni subagentes.

    Los subagentes siguen siendo una topología de ejecución interna. No aparecen como pseudomodelos.

  </Accordion>
  <Accordion title="¿Por qué se incluye `openclaw/default`?">
    `openclaw/default` es el alias estable para el agente predeterminado configurado.

    Eso significa que los clientes pueden seguir usando un id predecible aunque el id real del agente predeterminado cambie entre entornos.

  </Accordion>
  <Accordion title="¿Cómo reemplazo el modelo de backend?">
    Usa `x-openclaw-model`.

    Ejemplos:
    `x-openclaw-model: openai/gpt-5.4`
    `x-openclaw-model: gpt-5.4`

    Si lo omites, el agente seleccionado se ejecuta con su elección normal de modelo configurado.

  </Accordion>
  <Accordion title="¿Cómo encajan los embeddings en este contrato?">
    `/v1/embeddings` usa los mismos ids `model` de destino de agente.

    Usa `model: "openclaw/default"` o `model: "openclaw/<agentId>"`.
    Cuando necesites un modelo específico de embedding, envíalo en `x-openclaw-model`.
    Sin ese encabezado, la solicitud se transmite a la configuración normal de embeddings del agente seleccionado.

  </Accordion>
</AccordionGroup>

## Streaming (SSE)

Establece `stream: true` para recibir Server-Sent Events (SSE):

- `Content-Type: text/event-stream`
- Cada línea de evento es `data: <json>`
- El stream termina con `data: [DONE]`

## Configuración rápida de Open WebUI

Para una conexión básica de Open WebUI:

- URL base: `http://127.0.0.1:18789/v1`
- URL base de Docker en macOS: `http://host.docker.internal:18789/v1`
- API key: tu token bearer del Gateway
- Modelo: `openclaw/default`

Comportamiento esperado:

- `GET /v1/models` debe listar `openclaw/default`
- Open WebUI debe usar `openclaw/default` como id de modelo de chat
- Si quieres un proveedor/modelo de backend específico para ese agente, define el modelo predeterminado normal del agente o envía `x-openclaw-model`

Prueba rápida:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Si eso devuelve `openclaw/default`, la mayoría de las configuraciones de Open WebUI pueden conectarse con la misma URL base y token.

## Ejemplos

Sin streaming:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"hi"}]
  }'
```

Con streaming:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"hi"}]
  }'
```

Listar modelos:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Obtener un modelo:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Crear embeddings:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

Notas:

- `/v1/models` devuelve destinos de agentes de OpenClaw, no catálogos sin procesar de proveedores.
- `openclaw/default` siempre está presente para que un id estable funcione en todos los entornos.
- Los reemplazos de proveedor/modelo de backend pertenecen a `x-openclaw-model`, no al campo OpenAI `model`.
- `/v1/embeddings` admite `input` como una cadena o una matriz de cadenas.
