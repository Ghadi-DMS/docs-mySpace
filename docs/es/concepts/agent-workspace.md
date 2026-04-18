---
read_when:
    - Necesitas explicar el espacio de trabajo del agente o su diseño de archivos
    - Quieres hacer una copia de seguridad o migrar un espacio de trabajo del agente
summary: 'Espacio de trabajo del agente: ubicación, diseño y estrategia de copia de seguridad'
title: Espacio de trabajo del agente
x-i18n:
    generated_at: "2026-04-18T04:59:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: dd2e74614d8d45df04b1bbda48e2224e778b621803d774d38e4b544195eb234e
    source_path: concepts/agent-workspace.md
    workflow: 15
---

# Espacio de trabajo del agente

El espacio de trabajo es el hogar del agente. Es el único directorio de trabajo usado para
las herramientas de archivos y para el contexto del espacio de trabajo. Mantenlo privado y trátalo como memoria.

Esto es independiente de `~/.openclaw/`, que almacena configuración, credenciales y
sesiones.

**Importante:** el espacio de trabajo es el **cwd predeterminado**, no un sandbox rígido. Las herramientas
resuelven las rutas relativas contra el espacio de trabajo, pero las rutas absolutas aún pueden llegar
a otras ubicaciones del host a menos que el sandbox esté habilitado. Si necesitas aislamiento, usa
[`agents.defaults.sandbox`](/es/gateway/sandboxing) (y/o la configuración de sandbox por agente).
Cuando el sandbox está habilitado y `workspaceAccess` no es `"rw"`, las herramientas operan
dentro de un espacio de trabajo aislado bajo `~/.openclaw/sandboxes`, no en tu espacio de trabajo del host.

## Ubicación predeterminada

- Predeterminado: `~/.openclaw/workspace`
- Si `OPENCLAW_PROFILE` está establecido y no es `"default"`, el valor predeterminado pasa a ser
  `~/.openclaw/workspace-<profile>`.
- Puedes sobrescribirlo en `~/.openclaw/openclaw.json`:

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

`openclaw onboard`, `openclaw configure` o `openclaw setup` crearán el
espacio de trabajo y sembrarán los archivos de arranque si faltan.
Las copias de siembra del sandbox solo aceptan archivos regulares dentro del espacio de trabajo;
los alias de enlaces simbólicos/enlaces físicos que se resuelven fuera del espacio de trabajo de origen se ignoran.

Si ya administras tú mismo los archivos del espacio de trabajo, puedes deshabilitar la creación de archivos de arranque:

```json5
{ agent: { skipBootstrap: true } }
```

## Carpetas adicionales del espacio de trabajo

Las instalaciones antiguas pueden haber creado `~/openclaw`. Mantener varios directorios de espacio de trabajo
puede causar deriva confusa de autenticación o estado, porque solo un
espacio de trabajo está activo a la vez.

**Recomendación:** mantén un único espacio de trabajo activo. Si ya no usas las
carpetas adicionales, archívalas o muévelas a la Papelera (por ejemplo `trash ~/openclaw`).
Si intencionalmente mantienes varios espacios de trabajo, asegúrate de que
`agents.defaults.workspace` apunte al activo.

`openclaw doctor` advierte cuando detecta directorios adicionales del espacio de trabajo.

## Mapa de archivos del espacio de trabajo (qué significa cada archivo)

Estos son los archivos estándar que OpenClaw espera dentro del espacio de trabajo:

- `AGENTS.md`
  - Instrucciones operativas para el agente y cómo debe usar la memoria.
  - Se carga al inicio de cada sesión.
  - Buen lugar para reglas, prioridades y detalles sobre "cómo comportarse".

- `SOUL.md`
  - Personalidad, tono y límites.
  - Se carga en cada sesión.
  - Guía: [Guía de personalidad de SOUL.md](/es/concepts/soul)

- `USER.md`
  - Quién es el usuario y cómo dirigirse a él.
  - Se carga en cada sesión.

- `IDENTITY.md`
  - El nombre, la vibra y el emoji del agente.
  - Se crea/actualiza durante el ritual de arranque.

- `TOOLS.md`
  - Notas sobre tus herramientas locales y convenciones.
  - No controla la disponibilidad de herramientas; es solo orientación.

- `HEARTBEAT.md`
  - Lista de verificación opcional y pequeña para ejecuciones de Heartbeat.
  - Mantenla breve para evitar gasto de tokens.

- `BOOT.md`
  - Lista de verificación opcional de inicio ejecutada al reiniciar el Gateway cuando los hooks internos están habilitados.
  - Mantenla breve; usa la herramienta de mensajes para envíos salientes.

- `BOOTSTRAP.md`
  - Ritual único de primera ejecución.
  - Solo se crea para un espacio de trabajo completamente nuevo.
  - Elimínalo una vez que el ritual esté completo.

- `memory/YYYY-MM-DD.md`
  - Registro diario de memoria (un archivo por día).
  - Se recomienda leer el de hoy y el de ayer al iniciar la sesión.

- `MEMORY.md` (opcional)
  - Memoria de largo plazo curada.
  - Cárgala solo en la sesión principal y privada (no en contextos compartidos/de grupo).

Consulta [Memory](/es/concepts/memory) para el flujo de trabajo y el volcado automático de memoria.

- `skills/` (opcional)
  - Skills específicos del espacio de trabajo.
  - Ubicación de Skills de mayor precedencia para ese espacio de trabajo.
  - Sobrescribe Skills del agente del proyecto, Skills personales del agente, Skills administrados, Skills incluidos y `skills.load.extraDirs` cuando los nombres coinciden.

- `canvas/` (opcional)
  - Archivos de la interfaz Canvas para visualizaciones de nodos (por ejemplo `canvas/index.html`).

Si falta algún archivo de arranque, OpenClaw inserta un marcador de "archivo faltante" en
la sesión y continúa. Los archivos de arranque grandes se truncan al insertarse;
ajusta los límites con `agents.defaults.bootstrapMaxChars` (predeterminado: 12000) y
`agents.defaults.bootstrapTotalMaxChars` (predeterminado: 60000).
`openclaw setup` puede recrear los valores predeterminados faltantes sin sobrescribir los
archivos existentes.

## Lo que NO está en el espacio de trabajo

Estos elementos viven bajo `~/.openclaw/` y NO deben confirmarse en el repositorio del espacio de trabajo:

- `~/.openclaw/openclaw.json` (configuración)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (perfiles de autenticación del modelo: OAuth + claves API)
- `~/.openclaw/credentials/` (estado de canal/proveedor más datos de importación heredados de OAuth)
- `~/.openclaw/agents/<agentId>/sessions/` (transcripciones de sesiones + metadatos)
- `~/.openclaw/skills/` (Skills administrados)

Si necesitas migrar sesiones o configuración, cópialas por separado y mantenlas
fuera del control de versiones.

## Copia de seguridad con Git (recomendada, privada)

Trata el espacio de trabajo como memoria privada. Ponlo en un repositorio de git **privado** para que esté
respaldado y sea recuperable.

Ejecuta estos pasos en la máquina donde se ejecuta el Gateway (ahí es donde vive el
espacio de trabajo).

### 1) Inicializa el repositorio

Si git está instalado, los espacios de trabajo completamente nuevos se inicializan automáticamente. Si este
espacio de trabajo aún no es un repositorio, ejecuta:

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

### 2) Añade un remoto privado (opciones amigables para principiantes)

Opción A: interfaz web de GitHub

1. Crea un repositorio **privado** nuevo en GitHub.
2. No lo inicialices con un README (evita conflictos de fusión).
3. Copia la URL remota HTTPS.
4. Añade el remoto y haz push:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

Opción B: GitHub CLI (`gh`)

```bash
gh auth login
gh repo create openclaw-workspace --private --source . --remote origin --push
```

Opción C: interfaz web de GitLab

1. Crea un repositorio **privado** nuevo en GitLab.
2. No lo inicialices con un README (evita conflictos de fusión).
3. Copia la URL remota HTTPS.
4. Añade el remoto y haz push:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

### 3) Actualizaciones continuas

```bash
git status
git add .
git commit -m "Update memory"
git push
```

## No confirmes secretos

Incluso en un repositorio privado, evita almacenar secretos en el espacio de trabajo:

- Claves API, tokens OAuth, contraseñas o credenciales privadas.
- Cualquier cosa bajo `~/.openclaw/`.
- Volcados sin procesar de chats o archivos adjuntos sensibles.

Si debes almacenar referencias sensibles, usa marcadores de posición y guarda el secreto real en otra parte
(administrador de contraseñas, variables de entorno o `~/.openclaw/`).

Plantilla sugerida para `.gitignore`:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## Mover el espacio de trabajo a una máquina nueva

1. Clona el repositorio en la ruta deseada (predeterminada `~/.openclaw/workspace`).
2. Establece `agents.defaults.workspace` en esa ruta dentro de `~/.openclaw/openclaw.json`.
3. Ejecuta `openclaw setup --workspace <path>` para sembrar cualquier archivo faltante.
4. Si necesitas las sesiones, copia `~/.openclaw/agents/<agentId>/sessions/` desde la
   máquina anterior por separado.

## Notas avanzadas

- El enrutamiento de múltiples agentes puede usar diferentes espacios de trabajo por agente. Consulta
  [Enrutamiento de canales](/es/channels/channel-routing) para la configuración de enrutamiento.
- Si `agents.defaults.sandbox` está habilitado, las sesiones no principales pueden usar espacios de trabajo
  aislados por sesión bajo `agents.defaults.sandbox.workspaceRoot`.

## Relacionado

- [Standing Orders](/es/automation/standing-orders) — instrucciones persistentes en archivos del espacio de trabajo
- [Heartbeat](/es/gateway/heartbeat) — archivo `HEARTBEAT.md` del espacio de trabajo
- [Session](/es/concepts/session) — rutas de almacenamiento de sesiones
- [Sandboxing](/es/gateway/sandboxing) — acceso al espacio de trabajo en entornos con sandbox
