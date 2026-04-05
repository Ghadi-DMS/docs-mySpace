---
read_when:
    - Ejecutar pruebas localmente o en CI
    - Agregar regresiones para errores de modelos/proveedores
    - Depurar el comportamiento de gateway + agente
summary: 'Kit de pruebas: suites unit/e2e/live, ejecutores Docker y qué cubre cada prueba'
title: Pruebas
x-i18n:
    generated_at: "2026-04-05T12:46:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 854a39ae261d8749b8d8d82097b97a7c52cf2216d1fe622e302d830a888866ab
    source_path: help/testing.md
    workflow: 15
---

# Pruebas

OpenClaw tiene tres suites de Vitest (unit/integration, e2e, live) y un pequeño conjunto de ejecutores Docker.

Este documento es una guía de “cómo probamos”:

- Qué cubre cada suite (y qué deliberadamente _no_ cubre)
- Qué comandos ejecutar para flujos de trabajo habituales (local, antes de hacer push, depuración)
- Cómo las pruebas live descubren credenciales y seleccionan modelos/proveedores
- Cómo agregar regresiones para problemas reales de modelos/proveedores

## Inicio rápido

La mayoría de los días:

- Barrera completa (esperada antes de hacer push): `pnpm build && pnpm check && pnpm test`
- Ejecución local más rápida de la suite completa en una máquina con buenos recursos: `pnpm test:max`
- Bucle directo de Vitest watch (configuración moderna de proyectos): `pnpm test:watch`
- El direccionamiento directo de archivos ahora también enruta rutas de extensiones/canales: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`

Cuando tocas pruebas o quieres más confianza:

- Barrera de cobertura: `pnpm test:coverage`
- Suite E2E: `pnpm test:e2e`

Cuando depuras proveedores/modelos reales (requiere credenciales reales):

- Suite live (modelos + sondas de herramientas/imágenes del gateway): `pnpm test:live`
- Dirigir una sola prueba live en modo silencioso: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Consejo: cuando solo necesites un caso que falla, prefiere acotar las pruebas live mediante las variables de entorno de allowlist descritas más abajo.

## Suites de pruebas (qué se ejecuta dónde)

Piensa en las suites como “realismo creciente” (y también mayor inestabilidad/costo):

### Unit / integration (predeterminada)

- Comando: `pnpm test`
- Configuración: `projects` nativos de Vitest mediante `vitest.config.ts`
- Archivos: inventarios core/unit en `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` y las pruebas node de `ui` en allowlist cubiertas por `vitest.unit.config.ts`
- Alcance:
  - Pruebas unitarias puras
  - Pruebas de integración en proceso (autenticación del gateway, enrutamiento, herramientas, parsing, configuración)
  - Regresiones deterministas para errores conocidos
- Expectativas:
  - Se ejecuta en CI
  - No requiere claves reales
  - Debe ser rápida y estable
- Nota sobre proyectos:
  - `pnpm test`, `pnpm test:watch` y `pnpm test:changed` ahora usan la misma configuración raíz nativa de `projects` de Vitest.
  - Los filtros directos por archivo se enrutan de forma nativa por el grafo raíz del proyecto, así que `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` funciona sin un contenedor personalizado.
- Nota sobre el ejecutor integrado:
  - Cuando cambies entradas de descubrimiento de herramientas de mensajes o el contexto de tiempo de ejecución de compactación,
    mantén ambos niveles de cobertura.
  - Agrega regresiones centradas en helpers para límites puros de enrutamiento/normalización.
  - Mantén también sanas las suites de integración del ejecutor integrado:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` y
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Esas suites verifican que los ids con alcance y el comportamiento de compactación sigan fluyendo
    por las rutas reales de `run.ts` / `compact.ts`; las pruebas solo de helper no son un
    sustituto suficiente para esas rutas de integración.
- Nota sobre el pool:
  - La configuración base de Vitest ahora usa `threads` de forma predeterminada.
  - La configuración compartida de Vitest también fija `isolate: false` y usa el ejecutor no aislado en los proyectos raíz, e2e y live.
  - La ruta raíz de UI mantiene su configuración `jsdom` y optimizador, pero ahora también se ejecuta en el ejecutor compartido no aislado.
  - `pnpm test` hereda los mismos valores predeterminados `threads` + `isolate: false` desde la configuración de `projects` en `vitest.config.ts`.
  - El lanzador compartido `scripts/run-vitest.mjs` ahora también agrega `--no-maglev` de forma predeterminada para procesos hijos Node de Vitest para reducir la sobrecarga de compilación de V8 durante ejecuciones locales grandes. Define `OPENCLAW_VITEST_ENABLE_MAGLEV=1` si necesitas comparar con el comportamiento estándar de V8.
- Nota sobre iteración local rápida:
  - `pnpm test:changed` ejecuta la configuración nativa de proyectos con `--changed origin/main`.
  - `pnpm test:max` y `pnpm test:changed:max` mantienen la misma configuración nativa de proyectos, solo con un límite superior de workers.
  - El autoescalado local de workers ahora es intencionalmente conservador y también reduce el ritmo cuando la carga media del host ya es alta, así que múltiples ejecuciones concurrentes de Vitest hacen menos daño de forma predeterminada.
  - La configuración base de Vitest marca los archivos de proyectos/configuración como `forceRerunTriggers` para que las repeticiones en modo changed sigan siendo correctas cuando cambia el cableado de pruebas.
  - La configuración mantiene `OPENCLAW_VITEST_FS_MODULE_CACHE` habilitado en hosts compatibles; define `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` si quieres una ubicación explícita de caché para perfilado directo.
- Nota de depuración de rendimiento:
  - `pnpm test:perf:imports` habilita informes de duración de importación de Vitest además de salida de desglose de importaciones.
  - `pnpm test:perf:imports:changed` limita la misma vista de perfilado a archivos cambiados desde `origin/main`.
  - `pnpm test:perf:profile:main` escribe un perfil de CPU del hilo principal para la sobrecarga de arranque y transformación de Vitest/Vite.
  - `pnpm test:perf:profile:runner` escribe perfiles de CPU+heap del ejecutor para la suite unit con el paralelismo por archivo deshabilitado.

### E2E (smoke del gateway)

- Comando: `pnpm test:e2e`
- Configuración: `vitest.e2e.config.ts`
- Archivos: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Valores predeterminados de tiempo de ejecución:
  - Usa Vitest `threads` con `isolate: false`, igual que el resto del repositorio.
  - Usa workers adaptativos (CI: hasta 2, local: 1 de forma predeterminada).
  - Se ejecuta en modo silencioso de forma predeterminada para reducir la sobrecarga de E/S de consola.
- Reemplazos útiles:
  - `OPENCLAW_E2E_WORKERS=<n>` para forzar el número de workers (limitado a 16).
  - `OPENCLAW_E2E_VERBOSE=1` para volver a habilitar salida detallada en consola.
- Alcance:
  - Comportamiento end-to-end del gateway con varias instancias
  - Superficies WebSocket/HTTP, emparejamiento de nodos y redes más pesadas
- Expectativas:
  - Se ejecuta en CI (cuando está habilitada en la canalización)
  - No requiere claves reales
  - Tiene más piezas móviles que las pruebas unitarias (puede ser más lenta)

### E2E: smoke del backend OpenShell

- Comando: `pnpm test:e2e:openshell`
- Archivo: `test/openshell-sandbox.e2e.test.ts`
- Alcance:
  - Inicia un gateway OpenShell aislado en el host mediante Docker
  - Crea un sandbox desde un Dockerfile local temporal
  - Ejercita el backend OpenShell de OpenClaw sobre `sandbox ssh-config` real + exec por SSH
  - Verifica el comportamiento canónico del sistema de archivos remoto a través del puente fs del sandbox
- Expectativas:
  - Solo opt-in; no forma parte de la ejecución predeterminada `pnpm test:e2e`
  - Requiere una CLI local `openshell` y un daemon Docker funcional
  - Usa `HOME` / `XDG_CONFIG_HOME` aislados y luego destruye el gateway de prueba y el sandbox
- Reemplazos útiles:
  - `OPENCLAW_E2E_OPENSHELL=1` para habilitar la prueba al ejecutar manualmente la suite e2e más amplia
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` para apuntar a un binario de CLI no predeterminado o a un script contenedor

### Live (proveedores reales + modelos reales)

- Comando: `pnpm test:live`
- Configuración: `vitest.live.config.ts`
- Archivos: `src/**/*.live.test.ts`
- Predeterminado: **habilitada** por `pnpm test:live` (define `OPENCLAW_LIVE_TEST=1`)
- Alcance:
  - “¿Este proveedor/modelo realmente funciona _hoy_ con credenciales reales?”
  - Detectar cambios en formatos de proveedor, particularidades de llamadas a herramientas, problemas de autenticación y comportamiento de límites de tasa
- Expectativas:
  - No es estable en CI por diseño (redes reales, políticas reales de proveedores, cuotas, caídas)
  - Cuesta dinero / consume límites de tasa
  - Es preferible ejecutar subconjuntos acotados en lugar de “todo”
- Las ejecuciones live cargan `~/.profile` para recoger API keys faltantes.
- De forma predeterminada, las ejecuciones live siguen aislando `HOME` y copian configuración/material de autenticación a un home de prueba temporal para que los fixtures unit no muten tu `~/.openclaw` real.
- Define `OPENCLAW_LIVE_USE_REAL_HOME=1` solo cuando necesites intencionalmente que las pruebas live usen tu directorio personal real.
- `pnpm test:live` ahora usa por defecto un modo más silencioso: mantiene la salida de progreso `[live] ...`, pero suprime el aviso adicional de `~/.profile` y silencia registros de arranque del gateway/ruido de Bonjour. Define `OPENCLAW_LIVE_TEST_QUIET=0` si quieres recuperar los registros completos de inicio.
- Rotación de API keys (específica por proveedor): define `*_API_KEYS` con formato de comas/punto y coma o `*_API_KEY_1`, `*_API_KEY_2` (por ejemplo `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) o reemplazo por live mediante `OPENCLAW_LIVE_*_KEY`; las pruebas reintentan ante respuestas de límite de tasa.
- Salida de progreso/heartbeat:
  - Las suites live ahora emiten líneas de progreso a stderr para que las llamadas largas a proveedores se vean activas incluso cuando la captura de consola de Vitest está en modo silencioso.
  - `vitest.live.config.ts` deshabilita la intercepción de consola de Vitest para que las líneas de progreso de proveedor/gateway fluyan inmediatamente durante las ejecuciones live.
  - Ajusta los heartbeats de modelo directo con `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Ajusta los heartbeats de gateway/sonda con `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## ¿Qué suite debería ejecutar?

Usa esta tabla de decisión:

- Editar lógica/pruebas: ejecuta `pnpm test` (y `pnpm test:coverage` si cambiaste bastante)
- Tocar redes del gateway / protocolo WS / emparejamiento: añade `pnpm test:e2e`
- Depurar “mi bot está caído” / fallos específicos de proveedor / llamadas a herramientas: ejecuta un `pnpm test:live` acotado

## Live: barrido de capacidades de nodo Android

- Prueba: `src/gateway/android-node.capabilities.live.test.ts`
- Script: `pnpm android:test:integration`
- Objetivo: invocar **cada comando anunciado actualmente** por un nodo Android conectado y afirmar el comportamiento del contrato del comando.
- Alcance:
  - Configuración previa/manual (la suite no instala/ejecuta/empareja la app).
  - Validación comando por comando de gateway `node.invoke` para el nodo Android seleccionado.
- Configuración previa obligatoria:
  - App Android ya conectada y emparejada con el gateway.
  - App mantenida en primer plano.
  - Permisos/consentimiento de captura concedidos para las capacidades que esperas que pasen.
- Reemplazos opcionales de objetivo:
  - `OPENCLAW_ANDROID_NODE_ID` o `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Detalles completos de configuración Android: [App Android](/platforms/android)

## Live: smoke de modelo (claves de perfil)

Las pruebas live se dividen en dos capas para poder aislar fallos:

- “Modelo directo” nos dice si el proveedor/modelo puede responder en absoluto con la clave dada.
- “Gateway smoke” nos dice si funciona la canalización completa gateway+agente para ese modelo (sesiones, historial, herramientas, política de sandbox, etc.).

### Capa 1: finalización directa de modelo (sin gateway)

- Prueba: `src/agents/models.profiles.live.test.ts`
- Objetivo:
  - Enumerar modelos detectados
  - Usar `getApiKeyForModel` para seleccionar modelos para los que tienes credenciales
  - Ejecutar una pequeña finalización por modelo (y regresiones dirigidas cuando sea necesario)
- Cómo habilitar:
  - `pnpm test:live` (o `OPENCLAW_LIVE_TEST=1` si invocas Vitest directamente)
- Define `OPENCLAW_LIVE_MODELS=modern` (o `all`, alias de modern) para ejecutar realmente esta suite; en caso contrario se omite para mantener `pnpm test:live` centrado en el gateway smoke
- Cómo seleccionar modelos:
  - `OPENCLAW_LIVE_MODELS=modern` para ejecutar la allowlist moderna (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` es un alias de la allowlist moderna
  - o `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist separada por comas)
- Cómo seleccionar proveedores:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (allowlist separada por comas)
- De dónde vienen las claves:
  - De forma predeterminada: almacén de perfiles y respaldos por entorno
  - Define `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` para forzar **solo el almacén de perfiles**
- Por qué existe esto:
  - Separa “la API del proveedor está rota / la clave no es válida” de “la canalización del agente del gateway está rota”
  - Contiene regresiones pequeñas y aisladas (ejemplo: reproducción de razonamiento + flujos de llamadas a herramientas de OpenAI Responses/Codex Responses)

### Capa 2: smoke del gateway + agente dev (lo que realmente hace "@openclaw")

- Prueba: `src/gateway/gateway-models.profiles.live.test.ts`
- Objetivo:
  - Iniciar un gateway en proceso
  - Crear/parchear una sesión `agent:dev:*` (reemplazo de modelo por ejecución)
  - Iterar modelos-con-claves y afirmar:
    - respuesta “significativa” (sin herramientas)
    - que una invocación real de herramienta funciona (sonda de lectura)
    - sondas opcionales adicionales de herramientas (sonda exec+read)
    - que las rutas de regresión de OpenAI (solo llamada a herramienta → seguimiento) siguen funcionando
- Detalles de las sondas (para que puedas explicar fallos rápidamente):
  - Sonda `read`: la prueba escribe un archivo nonce en el espacio de trabajo y pide al agente que lo `read` y devuelva el nonce.
  - Sonda `exec+read`: la prueba pide al agente que escriba un nonce con `exec` en un archivo temporal y luego lo `read`.
  - Sonda de imagen: la prueba adjunta un PNG generado (cat + código aleatorio) y espera que el modelo devuelva `cat <CODE>`.
  - Referencia de implementación: `src/gateway/gateway-models.profiles.live.test.ts` y `src/gateway/live-image-probe.ts`.
- Cómo habilitar:
  - `pnpm test:live` (o `OPENCLAW_LIVE_TEST=1` si invocas Vitest directamente)
- Cómo seleccionar modelos:
  - Predeterminado: allowlist moderna (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` es un alias de la allowlist moderna
  - O define `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (o lista separada por comas) para acotar
- Cómo seleccionar proveedores (evitar “todo OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (allowlist separada por comas)
- Las sondas de herramientas + imagen siempre están activadas en esta prueba live:
  - sonda `read` + sonda `exec+read` (estrés de herramientas)
  - la sonda de imagen se ejecuta cuando el modelo anuncia compatibilidad con entrada de imagen
  - Flujo (alto nivel):
    - La prueba genera un PNG diminuto con “CAT” + código aleatorio (`src/gateway/live-image-probe.ts`)
    - Lo envía mediante `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway analiza los adjuntos en `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - El agente integrado reenvía un mensaje de usuario multimodal al modelo
    - Afirmación: la respuesta contiene `cat` + el código (tolerancia OCR: se permiten errores menores)

Consejo: para ver qué puedes probar en tu máquina (y los ids exactos `provider/model`), ejecuta:

```bash
openclaw models list
openclaw models list --json
```

## Live: smoke de backend CLI (Claude CLI u otras CLI locales)

- Prueba: `src/gateway/gateway-cli-backend.live.test.ts`
- Objetivo: validar la canalización Gateway + agente usando un backend CLI local, sin tocar tu configuración predeterminada.
- Habilitar:
  - `pnpm test:live` (o `OPENCLAW_LIVE_TEST=1` si invocas Vitest directamente)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Valores predeterminados:
  - Modelo: `claude-cli/claude-sonnet-4-6`
  - Comando: `claude`
  - Args: `["-p","--output-format","stream-json","--include-partial-messages","--verbose","--permission-mode","bypassPermissions"]`
- Reemplazos (opcionales):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-opus-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","stream-json","--include-partial-messages","--verbose","--permission-mode","bypassPermissions"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_CLEAR_ENV='["ANTHROPIC_API_KEY","ANTHROPIC_API_KEY_OLD"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` para enviar un adjunto de imagen real (las rutas se inyectan en el prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` para pasar rutas de archivos de imagen como args de CLI en lugar de inyección en el prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (o `"list"`) para controlar cómo se pasan los args de imagen cuando `IMAGE_ARG` está definido.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` para enviar un segundo turno y validar el flujo de reanudación.
- `OPENCLAW_LIVE_CLI_BACKEND_DISABLE_MCP_CONFIG=0` para mantener habilitada la configuración MCP de Claude CLI (de forma predeterminada inyecta una configuración estricta temporal vacía `--mcp-config` para que los servidores MCP ambientales/globales sigan deshabilitados durante el smoke).

Ejemplo:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Receta Docker:

```bash
pnpm test:docker:live-cli-backend
```

Notas:

- El ejecutor Docker está en `scripts/test-live-cli-backend-docker.sh`.
- Ejecuta el smoke live de backend CLI dentro de la imagen Docker del repositorio como usuario no root `node`, porque Claude CLI rechaza `bypassPermissions` cuando se invoca como root.
- Para `claude-cli`, instala el paquete Linux `@anthropic-ai/claude-code` en un prefijo escribible en caché en `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (predeterminado: `~/.cache/openclaw/docker-cli-tools`).
- Para `claude-cli`, el smoke live inyecta una configuración MCP estricta vacía a menos que definas `OPENCLAW_LIVE_CLI_BACKEND_DISABLE_MCP_CONFIG=0`.
- Copia `~/.claude` al contenedor cuando está disponible, pero en máquinas donde la autenticación de Claude depende de `ANTHROPIC_API_KEY`, también conserva `ANTHROPIC_API_KEY` / `ANTHROPIC_API_KEY_OLD` para la CLI hija de Claude mediante `OPENCLAW_LIVE_CLI_BACKEND_PRESERVE_ENV`.

## Live: smoke de binding ACP (`/acp spawn ... --bind here`)

- Prueba: `src/gateway/gateway-acp-bind.live.test.ts`
- Objetivo: validar el flujo real de binding de conversación ACP con un agente ACP live:
  - enviar `/acp spawn <agent> --bind here`
  - vincular una conversación sintética de canal de mensajes en el mismo lugar
  - enviar un seguimiento normal en esa misma conversación
  - verificar que el seguimiento llegue a la transcripción de la sesión ACP vinculada
- Habilitar:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Valores predeterminados:
  - Agente ACP: `claude`
  - Canal sintético: contexto de conversación estilo DM de Slack
  - Backend ACP: `acpx`
- Reemplazos:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=/full/path/to/acpx`
- Notas:
  - Esta ruta usa la superficie `chat.send` del gateway con campos administrativos de ruta de origen sintética para que las pruebas puedan adjuntar contexto de canal de mensajes sin fingir entrega externa.
  - Cuando `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND` no está definido, la prueba usa el comando acpx configurado/incluido. Si la autenticación de tu harness depende de variables de entorno de `~/.profile`, prefiere un comando `acpx` personalizado que preserve el entorno del proveedor.

Ejemplo:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Receta Docker:

```bash
pnpm test:docker:live-acp-bind
```

Notas de Docker:

- El ejecutor Docker está en `scripts/test-live-acp-bind-docker.sh`.
- Carga `~/.profile`, copia el home de autenticación CLI correspondiente (`~/.claude` o `~/.codex`) al contenedor, instala `acpx` en un prefijo npm escribible y luego instala la CLI live solicitada (`@anthropic-ai/claude-code` o `@openai/codex`) si falta.
- Dentro de Docker, el ejecutor define `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` para que acpx mantenga disponibles para la CLI hija del harness las variables de entorno del proveedor cargadas desde el profile.

### Recetas live recomendadas

Las allowlists acotadas y explícitas son las más rápidas y menos inestables:

- Modelo único, directo (sin gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Modelo único, gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Llamadas a herramientas en varios proveedores:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Enfoque en Google (Gemini API key + Antigravity):
  - Gemini (API key): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Notas:

- `google/...` usa la API de Gemini (API key).
- `google-antigravity/...` usa el puente OAuth de Antigravity (endpoint de agente estilo Cloud Code Assist).
- `google-gemini-cli/...` usa la Gemini CLI local de tu máquina (autenticación aparte + particularidades de herramientas).
- Gemini API frente a Gemini CLI:
  - API: OpenClaw llama a la API alojada de Gemini de Google por HTTP (autenticación con API key / perfil); esto es lo que la mayoría de los usuarios quieren decir con “Gemini”.
  - CLI: OpenClaw invoca un binario local `gemini`; tiene su propia autenticación y puede comportarse de forma diferente (streaming/soporte de herramientas/desfase de versión).

## Live: matriz de modelos (qué cubrimos)

No hay una “lista fija de modelos de CI” (live es opt-in), pero estos son los modelos **recomendados** para cubrir regularmente en una máquina de desarrollo con claves.

### Conjunto smoke moderno (llamadas a herramientas + imagen)

Esta es la ejecución de “modelos comunes” que esperamos mantener funcional:

- OpenAI (no Codex): `openai/gpt-5.4` (opcional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (o `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` y `google/gemini-3-flash-preview` (evita modelos Gemini 2.x más antiguos)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` y `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Ejecuta gateway smoke con herramientas + imagen:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Línea base: llamadas a herramientas (Read + Exec opcional)

Elige al menos una por familia de proveedor:

- OpenAI: `openai/gpt-5.4` (o `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (o `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (o `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Cobertura adicional opcional (deseable):

- xAI: `xai/grok-4` (o la última disponible)
- Mistral: `mistral/`… (elige un modelo compatible con “tools” que tengas habilitado)
- Cerebras: `cerebras/`… (si tienes acceso)
- LM Studio: `lmstudio/`… (local; las llamadas a herramientas dependen del modo de API)

### Visión: envío de imagen (adjunto → mensaje multimodal)

Incluye al menos un modelo compatible con imagen en `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/variantes de OpenAI compatibles con visión, etc.) para ejercitar la sonda de imagen.

### Agregadores / gateways alternativos

Si tienes claves habilitadas, también admitimos pruebas mediante:

- OpenRouter: `openrouter/...` (cientos de modelos; usa `openclaw models scan` para encontrar candidatos compatibles con herramientas+imagen)
- OpenCode: `opencode/...` para Zen y `opencode-go/...` para Go (autenticación mediante `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Más proveedores que puedes incluir en la matriz live (si tienes credenciales/configuración):

- Integrados: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Mediante `models.providers` (endpoints personalizados): `minimax` (cloud/API), además de cualquier proxy compatible con OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, etc.)

Consejo: no intentes fijar en la documentación “todos los modelos”. La lista autoritativa es lo que `discoverModels(...)` devuelve en tu máquina + las claves que estén disponibles.

## Credenciales (nunca las confirmes en git)

Las pruebas live descubren credenciales igual que la CLI. Implicaciones prácticas:

- Si la CLI funciona, las pruebas live deberían encontrar las mismas claves.
- Si una prueba live dice “no creds”, depúrala igual que depurarías `openclaw models list` / selección de modelo.

- Perfiles de autenticación por agente: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (esto es lo que significan las “profile keys” en las pruebas live)
- Configuración: `~/.openclaw/openclaw.json` (o `OPENCLAW_CONFIG_PATH`)
- Directorio de estado heredado: `~/.openclaw/credentials/` (se copia al home live preparado cuando existe, pero no es el almacén principal de profile-key)
- Las ejecuciones locales live copian de forma predeterminada la configuración activa, los archivos `auth-profiles.json` por agente, `credentials/` heredado y los directorios compatibles de autenticación CLI externa a un home de prueba temporal; en esa configuración preparada se eliminan los reemplazos de ruta `agents.*.workspace` / `agentDir` para que las sondas no toquen tu espacio de trabajo real del host.

Si quieres depender de claves de entorno (por ejemplo exportadas en tu `~/.profile`), ejecuta las pruebas locales después de `source ~/.profile`, o usa los ejecutores Docker de abajo (pueden montar `~/.profile` dentro del contenedor).

## Deepgram live (transcripción de audio)

- Prueba: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Habilitar: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- Prueba: `src/agents/byteplus.live.test.ts`
- Habilitar: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Reemplazo opcional de modelo: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Generación de imágenes live

- Prueba: `src/image-generation/runtime.live.test.ts`
- Comando: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Alcance:
  - Enumera todos los plugins registrados de proveedores de generación de imágenes
  - Carga variables de entorno faltantes de proveedores desde tu shell de inicio de sesión (`~/.profile`) antes de sondear
  - Usa por defecto API keys live/de entorno antes que perfiles de autenticación almacenados, para que claves de prueba obsoletas en `auth-profiles.json` no oculten credenciales reales del shell
  - Omite proveedores sin autenticación/perfil/modelo utilizable
  - Ejecuta las variantes estándar de generación de imágenes a través de la capacidad compartida del runtime:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Proveedores incluidos actualmente cubiertos:
  - `openai`
  - `google`
- Acotación opcional:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Comportamiento opcional de autenticación:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` para forzar autenticación desde el almacén de perfiles e ignorar reemplazos solo de entorno

## Ejecutores Docker (comprobaciones opcionales de “funciona en Linux”)

Estos ejecutores Docker se dividen en dos grupos:

- Ejecutores live de modelos: `test:docker:live-models` y `test:docker:live-gateway` ejecutan solo su archivo live correspondiente con profile-key dentro de la imagen Docker del repositorio (`src/agents/models.profiles.live.test.ts` y `src/gateway/gateway-models.profiles.live.test.ts`), montando tu directorio local de configuración y espacio de trabajo (y cargando `~/.profile` si está montado). Los entrypoints locales correspondientes son `test:live:models-profiles` y `test:live:gateway-profiles`.
- Los ejecutores Docker live usan por defecto un límite smoke menor para que un barrido Docker completo siga siendo práctico:
  `test:docker:live-models` usa por defecto `OPENCLAW_LIVE_MAX_MODELS=12`, y
  `test:docker:live-gateway` usa por defecto `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` y
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Reemplaza esas variables de entorno cuando
  quieras explícitamente el análisis exhaustivo más grande.
- `test:docker:all` construye una vez la imagen Docker live mediante `test:docker:live-build`, y luego la reutiliza para las dos rutas Docker live.
- Ejecutores smoke de contenedor: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` y `test:docker:plugins` levantan uno o más contenedores reales y verifican rutas de integración de mayor nivel.

Los ejecutores Docker live de modelos también montan solo los homes de autenticación CLI necesarios (o todos los compatibles cuando la ejecución no está acotada), y luego los copian al home del contenedor antes de la ejecución para que OAuth de CLI externas pueda refrescar tokens sin mutar el almacén de autenticación del host:

- Modelos directos: `pnpm test:docker:live-models` (script: `scripts/test-live-models-docker.sh`)
- Smoke de binding ACP: `pnpm test:docker:live-acp-bind` (script: `scripts/test-live-acp-bind-docker.sh`)
- Smoke de backend CLI: `pnpm test:docker:live-cli-backend` (script: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + agente dev: `pnpm test:docker:live-gateway` (script: `scripts/test-live-gateway-models-docker.sh`)
- Open WebUI live smoke: `pnpm test:docker:openwebui` (script: `scripts/e2e/openwebui-docker.sh`)
- Asistente de incorporación (TTY, scaffolding completo): `pnpm test:docker:onboard` (script: `scripts/e2e/onboard-docker.sh`)
- Redes del gateway (dos contenedores, autenticación WS + health): `pnpm test:docker:gateway-network` (script: `scripts/e2e/gateway-network-docker.sh`)
- Puente de canal MCP (Gateway inicializado + puente stdio + smoke sin procesar de frame de notificación de Claude): `pnpm test:docker:mcp-channels` (script: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke de instalación + alias `/plugin` + semántica de reinicio del bundle de Claude): `pnpm test:docker:plugins` (script: `scripts/e2e/plugins-docker.sh`)

Los ejecutores Docker live de modelos también montan el checkout actual en solo lectura y
lo preparan en un directorio de trabajo temporal dentro del contenedor. Esto mantiene la imagen
de tiempo de ejecución ligera mientras sigue ejecutando Vitest contra tu fuente/configuración local exacta.
También definen `OPENCLAW_SKIP_CHANNELS=1` para que las sondas live del gateway no inicien
workers reales de canales Telegram/Discord/etc. dentro del contenedor.
`test:docker:live-models` sigue ejecutando `pnpm test:live`, así que pasa también
`OPENCLAW_LIVE_GATEWAY_*` cuando necesites acotar o excluir la cobertura gateway
live de esa ruta Docker.
`test:docker:openwebui` es un smoke de compatibilidad de mayor nivel: inicia un
contenedor del gateway OpenClaw con los endpoints HTTP compatibles con OpenAI habilitados,
inicia un contenedor fijado de Open WebUI contra ese gateway, inicia sesión a través de
Open WebUI, verifica que `/api/models` exponga `openclaw/default` y luego envía una
solicitud de chat real a través del proxy `/api/chat/completions` de Open WebUI.
La primera ejecución puede ser notablemente más lenta porque Docker puede necesitar descargar la
imagen de Open WebUI y Open WebUI puede necesitar completar su propia configuración en frío.
Esta ruta espera una clave de modelo live utilizable, y `OPENCLAW_PROFILE_FILE`
(`~/.profile` por defecto) es la forma principal de proporcionarla en ejecuciones Dockerizadas.
Las ejecuciones correctas imprimen una pequeña carga JSON como `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` es intencionalmente determinista y no necesita una
cuenta real de Telegram, Discord ni iMessage. Inicia un Gateway
inicializado, arranca un segundo contenedor que ejecuta `openclaw mcp serve`, y luego
verifica detección de conversaciones enrutadas, lecturas de transcripción, metadatos de adjuntos,
comportamiento de cola de eventos live, enrutamiento de envíos salientes y notificaciones
de canal + permisos estilo Claude sobre el puente MCP stdio real. La comprobación de notificación
inspecciona directamente los frames MCP stdio sin procesar para que el smoke valide lo que
realmente emite el puente, no solo lo que un SDK específico de cliente exponga.

Smoke manual de hilos ACP en lenguaje natural (no CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Conserva este script para flujos de trabajo de regresión/depuración. Puede volver a ser necesario para validar el enrutamiento de hilos ACP, así que no lo elimines.

Variables de entorno útiles:

- `OPENCLAW_CONFIG_DIR=...` (predeterminado: `~/.openclaw`) montado en `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (predeterminado: `~/.openclaw/workspace`) montado en `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (predeterminado: `~/.profile`) montado en `/home/node/.profile` y cargado antes de ejecutar las pruebas
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (predeterminado: `~/.cache/openclaw/docker-cli-tools`) montado en `/home/node/.npm-global` para instalaciones CLI en caché dentro de Docker
- Los directorios de autenticación de CLI externas bajo `$HOME` se montan en solo lectura en `/host-auth/...`, y luego se copian a `/home/node/...` antes de iniciar las pruebas
  - Predeterminado: monta todos los directorios compatibles (`.codex`, `.claude`, `.minimax`)
  - Las ejecuciones acotadas por proveedor montan solo los directorios necesarios inferidos de `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Reemplaza manualmente con `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` o una lista separada por comas como `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` para acotar la ejecución
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` para filtrar proveedores dentro del contenedor
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` para asegurar que las credenciales vengan del almacén de perfiles (no del entorno)
- `OPENCLAW_OPENWEBUI_MODEL=...` para elegir el modelo expuesto por el gateway para el smoke de Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` para reemplazar el prompt de verificación con nonce usado por el smoke de Open WebUI
- `OPENWEBUI_IMAGE=...` para reemplazar la etiqueta fijada de imagen de Open WebUI

## Comprobación básica de documentación

Ejecuta comprobaciones de documentación después de editar docs: `pnpm check:docs`.
Ejecuta la validación completa de anclas de Mintlify cuando también necesites comprobaciones de encabezados dentro de la página: `pnpm docs:check-links:anchors`.

## Regresión offline (segura para CI)

Estas son regresiones de “canalización real” sin proveedores reales:

- Llamadas a herramientas del gateway (OpenAI simulado, gateway real + bucle de agente): `src/gateway/gateway.test.ts` (caso: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Asistente del gateway (WS `wizard.start`/`wizard.next`, escritura de configuración + autenticación forzada): `src/gateway/gateway.test.ts` (caso: "runs wizard over ws and writes auth token config")

## Evaluaciones de fiabilidad del agente (Skills)

Ya tenemos algunas pruebas seguras para CI que se comportan como “evaluaciones de fiabilidad del agente”:

- Llamadas simuladas a herramientas a través del gateway real + bucle de agente (`src/gateway/gateway.test.ts`).
- Flujos end-to-end del asistente que validan el cableado de sesiones y los efectos en la configuración (`src/gateway/gateway.test.ts`).

Lo que aún falta para Skills (consulta [Skills](/tools/skills)):

- **Toma de decisiones:** cuando los Skills aparecen en el prompt, ¿el agente elige el Skill correcto (o evita los irrelevantes)?
- **Cumplimiento:** ¿el agente lee `SKILL.md` antes de usarlo y sigue los pasos/args requeridos?
- **Contratos de flujo de trabajo:** escenarios de varios turnos que afirmen el orden de herramientas, el arrastre del historial de sesiones y los límites del sandbox.

Las evaluaciones futuras deberían seguir siendo deterministas primero:

- Un ejecutor de escenarios con proveedores simulados para afirmar llamadas a herramientas + orden, lecturas de archivos de Skill y cableado de sesiones.
- Un pequeño conjunto de escenarios centrados en Skills (usar vs evitar, control de acceso, inyección de prompts).
- Evaluaciones live opcionales (opt-in, limitadas por variables de entorno) solo después de que la suite segura para CI esté lista.

## Pruebas de contrato (forma de plugins y canales)

Las pruebas de contrato verifican que cada plugin y canal registrado cumpla su
contrato de interfaz. Iteran sobre todos los plugins descubiertos y ejecutan una suite de
afirmaciones de forma y comportamiento. La ruta unit predeterminada `pnpm test`
omite intencionalmente estos archivos compartidos de seam y smoke; ejecuta los comandos de contrato explícitamente
cuando toques superficies compartidas de canales o proveedores.

### Comandos

- Todos los contratos: `pnpm test:contracts`
- Solo contratos de canales: `pnpm test:contracts:channels`
- Solo contratos de proveedores: `pnpm test:contracts:plugins`

### Contratos de canales

Ubicados en `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Forma básica del plugin (id, nombre, capacidades)
- **setup** - Contrato del asistente de configuración
- **session-binding** - Comportamiento del binding de sesión
- **outbound-payload** - Estructura de la carga de mensajes
- **inbound** - Manejo de mensajes entrantes
- **actions** - Manejadores de acciones de canal
- **threading** - Manejo de ids de hilo
- **directory** - API de directorio/listado
- **group-policy** - Aplicación de la política de grupos

### Contratos de estado de proveedores

Ubicados en `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondas de estado del canal
- **registry** - Forma del registro de plugins

### Contratos de proveedores

Ubicados en `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Contrato del flujo de autenticación
- **auth-choice** - Elección/selección de autenticación
- **catalog** - API del catálogo de modelos
- **discovery** - Descubrimiento de plugins
- **loader** - Carga de plugins
- **runtime** - Runtime del proveedor
- **shape** - Forma/interfaz del plugin
- **wizard** - Asistente de configuración

### Cuándo ejecutarlas

- Después de cambiar exportaciones o subrutas de plugin-sdk
- Después de agregar o modificar un plugin de canal o proveedor
- Después de refactorizar el registro o descubrimiento de plugins

Las pruebas de contrato se ejecutan en CI y no requieren API keys reales.

## Agregar regresiones (guía)

Cuando corrijas un problema de proveedor/modelo descubierto en live:

- Agrega una regresión segura para CI si es posible (proveedor simulado/stub o captura de la transformación exacta de la forma de la solicitud)
- Si el problema es inherentemente solo live (límites de tasa, políticas de autenticación), mantén la prueba live acotada y opt-in mediante variables de entorno
- Prefiere apuntar a la capa más pequeña que detecte el error:
  - error de conversión/reproducción de solicitud del proveedor → prueba de modelos directos
  - error de canalización de sesión/historial/herramientas del gateway → gateway live smoke o prueba simulada del gateway segura para CI
- Medida de protección contra traversal de SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` deriva un objetivo de muestra por clase de SecretRef a partir de metadatos del registro (`listSecretTargetRegistryEntries()`), y luego afirma que los ids exec con segmentos traversal se rechazan.
  - Si agregas una nueva familia de objetivos SecretRef `includeInPlan` en `src/secrets/target-registry-data.ts`, actualiza `classifyTargetClass` en esa prueba. La prueba falla intencionalmente con ids de objetivo no clasificados para que no se puedan omitir silenciosamente nuevas clases.
