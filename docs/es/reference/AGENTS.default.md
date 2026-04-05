---
read_when:
    - Iniciando una nueva sesión de agente de OpenClaw
    - Habilitando o auditando las Skills predeterminadas
summary: Instrucciones predeterminadas del agente de OpenClaw y repertorio de Skills para la configuración de asistente personal
title: AGENTS.md predeterminado
x-i18n:
    generated_at: "2026-04-05T12:52:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 45990bc4e6fa2e3d80e76207e62ec312c64134bee3bc832a5cae32ca2eda3b61
    source_path: reference/AGENTS.default.md
    workflow: 15
---

# AGENTS.md - Asistente personal de OpenClaw (predeterminado)

## Primera ejecución (recomendado)

OpenClaw usa un directorio de workspace dedicado para el agente. Predeterminado: `~/.openclaw/workspace` (configurable mediante `agents.defaults.workspace`).

1. Crea el workspace (si aún no existe):

```bash
mkdir -p ~/.openclaw/workspace
```

2. Copia las plantillas predeterminadas del workspace al workspace:

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. Opcional: si quieres el repertorio de Skills del asistente personal, sustituye AGENTS.md por este archivo:

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. Opcional: elige un workspace distinto configurando `agents.defaults.workspace` (admite `~`):

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## Valores predeterminados de seguridad

- No vuelques directorios ni secretos en el chat.
- No ejecutes comandos destructivos salvo que se soliciten explícitamente.
- No envíes respuestas parciales/en streaming a superficies externas de mensajería (solo respuestas finales).

## Inicio de sesión (obligatorio)

- Lee `SOUL.md`, `USER.md` y los archivos de hoy+ayer en `memory/`.
- Lee `MEMORY.md` cuando exista; usa `memory.md` en minúsculas solo como respaldo cuando `MEMORY.md` no exista.
- Hazlo antes de responder.

## Soul (obligatorio)

- `SOUL.md` define identidad, tono y límites. Mantenlo actualizado.
- Si cambias `SOUL.md`, díselo a la persona usuaria.
- Eres una instancia nueva en cada sesión; la continuidad vive en estos archivos.

## Espacios compartidos (recomendado)

- No eres la voz de la persona usuaria; ten cuidado en chats grupales o canales públicos.
- No compartas datos privados, información de contacto ni notas internas.

## Sistema de memoria (recomendado)

- Registro diario: `memory/YYYY-MM-DD.md` (crea `memory/` si es necesario).
- Memoria a largo plazo: `MEMORY.md` para hechos duraderos, preferencias y decisiones.
- `memory.md` en minúsculas es solo un respaldo heredado; no mantengas ambos archivos raíz intencionadamente.
- Al iniciar la sesión, lee hoy + ayer + `MEMORY.md` cuando exista; en caso contrario `memory.md`.
- Captura: decisiones, preferencias, restricciones, temas pendientes.
- Evita secretos salvo que se soliciten explícitamente.

## Herramientas y Skills

- Las herramientas viven en Skills; sigue el `SKILL.md` de cada Skill cuando lo necesites.
- Mantén las notas específicas del entorno en `TOOLS.md` (Notes for Skills).

## Consejo de copia de seguridad (recomendado)

Si tratas este workspace como la “memoria” de Clawd, conviértelo en un repositorio git (idealmente privado) para que `AGENTS.md` y tus archivos de memoria tengan copia de seguridad.

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add Clawd workspace"
# Opcional: agrega un remote privado + haz push
```

## Qué hace OpenClaw

- Ejecuta el gateway de WhatsApp + el agente de código Pi para que el asistente pueda leer/escribir chats, obtener contexto y ejecutar Skills mediante el Mac host.
- La app de macOS gestiona permisos (grabación de pantalla, notificaciones, micrófono) y expone la CLI `openclaw` mediante su binario integrado.
- Los chats directos se contraen por defecto en la sesión `main` del agente; los grupos se mantienen aislados como `agent:<agentId>:<channel>:group:<id>` (salas/canales: `agent:<agentId>:<channel>:channel:<id>`); los heartbeats mantienen vivas las tareas en segundo plano.

## Skills principales (habilitar en Settings → Skills)

- **mcporter** — Runtime/CLI de servidor de herramientas para gestionar backends externos de Skills.
- **Peekaboo** — Capturas de pantalla rápidas en macOS con análisis opcional de visión por IA.
- **camsnap** — Captura fotogramas, clips o alertas de movimiento desde cámaras de seguridad RTSP/ONVIF.
- **oracle** — CLI de agente preparada para OpenAI con reproducción de sesión y control del navegador.
- **eightctl** — Controla tu sueño desde la terminal.
- **imsg** — Envía, lee y transmite iMessage y SMS.
- **wacli** — CLI de WhatsApp: sincroniza, busca y envía.
- **discord** — Acciones de Discord: reaccionar, stickers, encuestas. Usa destinos `user:<id>` o `channel:<id>` (los id numéricos sin formato son ambiguos).
- **gog** — CLI de Google Suite: Gmail, Calendar, Drive, Contacts.
- **spotify-player** — Cliente de Spotify en terminal para buscar/poner en cola/controlar la reproducción.
- **sag** — Voz de ElevenLabs con UX estilo `say` de mac; reproduce por los altavoces por defecto.
- **Sonos CLI** — Controla altavoces Sonos (discover/status/playback/volume/grouping) desde scripts.
- **blucli** — Reproduce, agrupa y automatiza reproductores BluOS desde scripts.
- **OpenHue CLI** — Control de iluminación Philips Hue para escenas y automatizaciones.
- **OpenAI Whisper** — Conversión local de voz a texto para dictado rápido y transcripciones de buzón de voz.
- **Gemini CLI** — Modelos Google Gemini desde la terminal para preguntas y respuestas rápidas.
- **agent-tools** — Toolkit de utilidades para automatizaciones y scripts auxiliares.

## Notas de uso

- Prefiere la CLI `openclaw` para scripting; la app de Mac gestiona los permisos.
- Ejecuta las instalaciones desde la pestaña Skills; oculta el botón si ya hay un binario presente.
- Mantén habilitados los heartbeats para que el asistente pueda programar recordatorios, supervisar bandejas de entrada y activar capturas de cámara.
- La UI de Canvas se ejecuta a pantalla completa con superposiciones nativas. Evita colocar controles críticos en los bordes superior izquierdo/superior derecho/inferiores; añade márgenes explícitos en el diseño y no dependas de los safe-area insets.
- Para verificación guiada por navegador, usa `openclaw browser` (tabs/status/screenshot) con el perfil de Chrome gestionado por OpenClaw.
- Para inspección del DOM, usa `openclaw browser eval|query|dom|snapshot` (y `--json`/`--out` cuando necesites salida legible por máquina).
- Para interacciones, usa `openclaw browser click|type|hover|drag|select|upload|press|wait|navigate|back|evaluate|run` (click/type requieren referencias de snapshot; usa `evaluate` para selectores CSS).
