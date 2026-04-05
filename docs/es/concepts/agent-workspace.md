---
read_when:
    - Necesitas explicar el espacio de trabajo del agente o su diseño de archivos
    - Quieres hacer una copia de seguridad o migrar un espacio de trabajo del agente
summary: 'Espacio de trabajo del agente: ubicación, diseño y estrategia de copia de seguridad'
title: Espacio de trabajo del agente
x-i18n:
    generated_at: "2026-04-05T12:39:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3735633f1098c733415369f9836fdbbc0bf869636a24ed42e95e6784610d964a
    source_path: concepts/agent-workspace.md
    workflow: 15
---

# Espacio de trabajo del agente

El espacio de trabajo es el hogar del agente. Es el único directorio de trabajo usado para
herramientas de archivos y para el contexto del espacio de trabajo. Mantenlo privado y trátalo como memoria.

Esto es independiente de `~/.openclaw/`, que almacena configuración, credenciales y
sesiones.

**Importante:** el espacio de trabajo es el **cwd predeterminado**, no un sandbox estricto. Las herramientas
resuelven rutas relativas con respecto al espacio de trabajo, pero las rutas absolutas aún pueden llegar
a otras ubicaciones del host a menos que el sandboxing esté habilitado. Si necesitas aislamiento, usa
[`agents.defaults.sandbox`](/gateway/sandboxing) (y/o configuración de sandbox por agente).
Cuando el sandboxing está habilitado y `workspaceAccess` no es `"rw"`, las herramientas operan
dentro de un espacio de trabajo aislado bajo `~/.openclaw/sandboxes`, no en el espacio de trabajo del host.

## Ubicación predeterminada

- Predeterminado: `~/.openclaw/workspace`
- Si `OPENCLAW_PROFILE` está establecido y no es `"default"`, el valor predeterminado pasa a ser
  `~/.openclaw/workspace-<profile>`.
- Sobrescríbelo en `~/.openclaw/openclaw.json`:

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

`openclaw onboard`, `openclaw configure` o `openclaw setup` crearán el
espacio de trabajo y sembrarán los archivos bootstrap si faltan.
Las copias semilla del sandbox solo aceptan archivos normales dentro del espacio de trabajo; los
alias de symlink/hardlink que se resuelven fuera del espacio de trabajo de origen se ignoran.

Si ya gestionas tú mismo los archivos del espacio de trabajo, puedes desactivar la
creación de archivos bootstrap:

```json5
{ agent: { skipBootstrap: true } }
```

## Carpetas adicionales del espacio de trabajo

Las instalaciones antiguas pueden haber creado `~/openclaw`. Mantener varios directorios de espacio de trabajo
puede causar confusión por desajustes de autenticación o de estado, porque solo un
espacio de trabajo está activo a la vez.

**Recomendación:** mantén un único espacio de trabajo activo. Si ya no usas las
carpetas extra, archívalas o muévelas a la Papelera (por ejemplo `trash ~/openclaw`).
Si intencionadamente mantienes varios espacios de trabajo, asegúrate de que
`agents.defaults.workspace` apunte al activo.

`openclaw doctor` avisa cuando detecta directorios adicionales de espacio de trabajo.

## Mapa de archivos del espacio de trabajo (qué significa cada archivo)

Estos son los archivos estándar que OpenClaw espera dentro del espacio de trabajo:

- `AGENTS.md`
  - Instrucciones operativas para el agente y cómo debe usar la memoria.
  - Se carga al inicio de cada sesión.
  - Buen lugar para reglas, prioridades y detalles de "cómo comportarse".

- `SOUL.md`
  - Personalidad, tono y límites.
  - Se carga en cada sesión.
  - Guía: [Guía de personalidad de SOUL.md](/concepts/soul)

- `USER.md`
  - Quién es el usuario y cómo dirigirse a él.
  - Se carga en cada sesión.

- `IDENTITY.md`
  - El nombre, estilo y emoji del agente.
  - Se crea/actualiza durante el ritual de bootstrap.

- `TOOLS.md`
  - Notas sobre tus herramientas y convenciones locales.
  - No controla la disponibilidad de herramientas; es solo orientación.

- `HEARTBEAT.md`
  - Lista de comprobación opcional y pequeña para ejecuciones de heartbeat.
  - Mantenla breve para evitar gastar tokens.

- `BOOT.md`
  - Lista de comprobación opcional de arranque ejecutada al reiniciar la gateway cuando los hooks internos están habilitados.
  - Mantenla breve; usa la herramienta de mensajes para envíos salientes.

- `BOOTSTRAP.md`
  - Ritual único de la primera ejecución.
  - Solo se crea para un espacio de trabajo completamente nuevo.
  - Elimínalo cuando el ritual haya terminado.

- `memory/YYYY-MM-DD.md`
  - Registro diario de memoria (un archivo por día).
  - Se recomienda leer el de hoy y el de ayer al inicio de la sesión.

- `MEMORY.md` (opcional)
  - Memoria de largo plazo curada.
  - Cárgala solo en la sesión principal y privada (no en contextos compartidos/de grupo).

Consulta [Memory](/concepts/memory) para ver el flujo de trabajo y el vaciado automático de memoria.

- `skills/` (opcional)
  - Skills específicas del espacio de trabajo.
  - Ubicación de Skills de mayor precedencia para ese espacio de trabajo.
  - Sobrescribe Skills de agentes del proyecto, Skills de agentes personales, Skills gestionadas, Skills integradas y `skills.load.extraDirs` cuando los nombres colisionan.

- `canvas/` (opcional)
  - Archivos de UI de canvas para pantallas de nodo (por ejemplo `canvas/index.html`).

Si falta cualquier archivo bootstrap, OpenClaw inyecta un marcador de "archivo faltante" en
la sesión y continúa. Los archivos bootstrap grandes se truncan al inyectarse;
ajusta los límites con `agents.defaults.bootstrapMaxChars` (predeterminado: 20000) y
`agents.defaults.bootstrapTotalMaxChars` (predeterminado: 150000).
`openclaw setup` puede recrear los valores predeterminados que falten sin sobrescribir
archivos existentes.

## Lo que NO está en el espacio de trabajo

Esto vive bajo `~/.openclaw/` y NO debe confirmarse en el repositorio del espacio de trabajo:

- `~/.openclaw/openclaw.json` (configuración)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (perfiles de autenticación de modelos: OAuth + API keys)
- `~/.openclaw/credentials/` (estado de canal/proveedor más datos heredados de importación OAuth)
- `~/.openclaw/agents/<agentId>/sessions/` (transcripciones de sesión + metadatos)
- `~/.openclaw/skills/` (Skills gestionadas)

Si necesitas migrar sesiones o configuración, cópialas por separado y mantenlas
fuera del control de versiones.

## Copia de seguridad con Git (recomendada, privada)

Trata el espacio de trabajo como memoria privada. Ponlo en un repositorio git **privado** para que esté
respaldado y sea recuperable.

Ejecuta estos pasos en la máquina donde se ejecuta la Gateway (ahí es donde vive el
espacio de trabajo).

### 1) Inicializar el repositorio

Si git está instalado, los espacios de trabajo completamente nuevos se inicializan automáticamente. Si este
espacio de trabajo todavía no es un repositorio, ejecuta:

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

### 2) Añadir un remoto privado (opciones sencillas para principiantes)

Opción A: interfaz web de GitHub

1. Crea un nuevo repositorio **privado** en GitHub.
2. No lo inicialices con un README (evita conflictos de merge).
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

1. Crea un nuevo repositorio **privado** en GitLab.
2. No lo inicialices con un README (evita conflictos de merge).
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

- API keys, tokens OAuth, contraseñas o credenciales privadas.
- Cualquier cosa bajo `~/.openclaw/`.
- Volcados sin procesar de chats o adjuntos sensibles.

Si debes almacenar referencias sensibles, usa placeholders y guarda el secreto
real en otro lugar (gestor de contraseñas, variables de entorno o `~/.openclaw/`).

Inicio sugerido para `.gitignore`:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## Mover el espacio de trabajo a una nueva máquina

1. Clona el repositorio en la ruta deseada (predeterminada `~/.openclaw/workspace`).
2. Establece `agents.defaults.workspace` en esa ruta dentro de `~/.openclaw/openclaw.json`.
3. Ejecuta `openclaw setup --workspace <path>` para sembrar cualquier archivo faltante.
4. Si necesitas sesiones, copia `~/.openclaw/agents/<agentId>/sessions/` desde la
   máquina antigua por separado.

## Notas avanzadas

- El enrutamiento multiagente puede usar distintos espacios de trabajo por agente. Consulta
  [Enrutamiento de canales](/channels/channel-routing) para la configuración de enrutamiento.
- Si `agents.defaults.sandbox` está habilitado, las sesiones no principales pueden usar
  espacios de trabajo aislados por sesión bajo `agents.defaults.sandbox.workspaceRoot`.

## Relacionado

- [Órdenes permanentes](/automation/standing-orders) — instrucciones persistentes en archivos del espacio de trabajo
- [Heartbeat](/gateway/heartbeat) — archivo de espacio de trabajo HEARTBEAT.md
- [Sesión](/concepts/session) — rutas de almacenamiento de sesiones
- [Sandboxing](/gateway/sandboxing) — acceso al espacio de trabajo en entornos aislados
