---
read_when:
    - Quieres memoria persistente que funcione entre sesiones y canales
    - Quieres recuperación impulsada por IA y modelado de usuarios
summary: Memoria nativa de IA entre sesiones mediante el plugin Honcho
title: Memoria Honcho
x-i18n:
    generated_at: "2026-04-05T12:39:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83ae3561152519a23589f754e0625f1e49c43e38f85de07686b963170a6cf229
    source_path: concepts/memory-honcho.md
    workflow: 15
---

# Memoria Honcho

[Honcho](https://honcho.dev) añade memoria nativa de IA a OpenClaw. Persiste
las conversaciones en un servicio dedicado y construye modelos de usuario y de agente con el tiempo,
dando a tu agente un contexto entre sesiones que va más allá de los archivos
Markdown del espacio de trabajo.

## Qué proporciona

- **Memoria entre sesiones** -- las conversaciones se conservan después de cada turno, por lo que
  el contexto se mantiene entre reinicios de sesión, compactación y cambios de canal.
- **Modelado de usuarios** -- Honcho mantiene un perfil para cada usuario (preferencias,
  hechos, estilo de comunicación) y para el agente (personalidad, comportamientos
  aprendidos).
- **Búsqueda semántica** -- búsqueda sobre observaciones de conversaciones pasadas, no
  solo de la sesión actual.
- **Conciencia de varios agentes** -- los agentes padre rastrean automáticamente a los
  subagentes generados, y los padres se añaden como observadores en las sesiones hijas.

## Herramientas disponibles

Honcho registra herramientas que el agente puede usar durante la conversación:

**Recuperación de datos (rápida, sin llamada al LLM):**

| Herramienta                 | Qué hace                                             |
| --------------------------- | ---------------------------------------------------- |
| `honcho_context`            | Representación completa del usuario entre sesiones   |
| `honcho_search_conclusions` | Búsqueda semántica sobre conclusiones almacenadas    |
| `honcho_search_messages`    | Busca mensajes entre sesiones (filtra por remitente, fecha) |
| `honcho_session`            | Historial y resumen de la sesión actual              |

**Preguntas y respuestas (impulsado por LLM):**

| Herramienta   | Qué hace                                                                 |
| ------------- | ------------------------------------------------------------------------ |
| `honcho_ask` | Hace preguntas sobre el usuario. `depth='quick'` para hechos, `'thorough'` para síntesis |

## Primeros pasos

Instala el plugin y ejecuta la configuración:

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

El comando de configuración solicita tus credenciales de API, escribe la configuración y
opcionalmente migra archivos de memoria existentes del espacio de trabajo.

<Info>
Honcho puede ejecutarse completamente en local (autoalojado) o mediante la API gestionada en
`api.honcho.dev`. No se requieren dependencias externas para la opción
autoalojada.
</Info>

## Configuración

La configuración se encuentra en `plugins.entries["openclaw-honcho"].config`:

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // omit for self-hosted
          workspaceId: "openclaw", // memory isolation
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

Para instancias autoalojadas, apunta `baseUrl` a tu servidor local (por ejemplo
`http://localhost:8000`) y omite la clave de API.

## Migración de memoria existente

Si tienes archivos de memoria existentes del espacio de trabajo (`USER.md`, `MEMORY.md`,
`IDENTITY.md`, `memory/`, `canvas/`), `openclaw honcho setup` los detecta y
ofrece migrarlos.

<Info>
La migración no es destructiva -- los archivos se cargan en Honcho. Los originales
nunca se eliminan ni se mueven.
</Info>

## Cómo funciona

Después de cada turno de IA, la conversación se conserva en Honcho. Tanto los mensajes del usuario como los
del agente se observan, lo que permite a Honcho construir y refinar sus modelos con el paso del
tiempo.

Durante la conversación, las herramientas de Honcho consultan el servicio en la fase `before_prompt_build`,
inyectando contexto relevante antes de que el modelo vea el prompt. Esto garantiza
límites de turno precisos y recuperación relevante.

## Honcho frente a la memoria integrada

|                   | Integrada / QMD               | Honcho                              |
| ----------------- | ----------------------------- | ----------------------------------- |
| **Almacenamiento** | Archivos Markdown del espacio de trabajo | Servicio dedicado (local o alojado) |
| **Entre sesiones** | Mediante archivos de memoria  | Automático, integrado               |
| **Modelado de usuarios** | Manual (escribir en `MEMORY.md`) | Perfiles automáticos                |
| **Búsqueda**      | Vectorial + palabra clave (híbrida) | Semántica sobre observaciones       |
| **Varios agentes** | Sin seguimiento               | Conciencia de padre/hijo            |
| **Dependencias**  | Ninguna (integrada) o binario QMD | Instalación de plugin               |

Honcho y el sistema de memoria integrado pueden trabajar juntos. Cuando QMD está configurado,
hay herramientas adicionales disponibles para buscar en archivos Markdown locales junto con
la memoria entre sesiones de Honcho.

## Comandos de CLI

```bash
openclaw honcho setup                        # Configura la clave de API y migra archivos
openclaw honcho status                       # Comprueba el estado de la conexión
openclaw honcho ask <question>               # Consulta a Honcho sobre el usuario
openclaw honcho search <query> [-k N] [-d D] # Búsqueda semántica sobre la memoria
```

## Lecturas adicionales

- [Código fuente del plugin](https://github.com/plastic-labs/openclaw-honcho)
- [Documentación de Honcho](https://docs.honcho.dev)
- [Guía de integración de Honcho con OpenClaw](https://docs.honcho.dev/v3/guides/integrations/openclaw)
- [Memory](/concepts/memory) -- resumen de memoria de OpenClaw
- [Context Engines](/concepts/context-engine) -- cómo funcionan los motores de contexto de los plugins
