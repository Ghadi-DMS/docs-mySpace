---
read_when:
    - Estás creando una nueva Skill personalizada en tu workspace
    - Necesitas un flujo de trabajo inicial rápido para Skills basadas en SKILL.md
summary: Compila y prueba Skills personalizados del workspace con SKILL.md
title: Creación de Skills
x-i18n:
    generated_at: "2026-04-05T12:55:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 747cebc5191b96311d1d6760bede1785a099acd7633a0b88de6b7882b57e1db6
    source_path: tools/creating-skills.md
    workflow: 15
---

# Creación de Skills

Las Skills enseñan al agente cómo y cuándo usar herramientas. Cada Skill es un directorio
que contiene un archivo `SKILL.md` con frontmatter YAML e instrucciones en markdown.

Para saber cómo se cargan y priorizan las Skills, consulta [Skills](/tools/skills).

## Crea tu primera Skill

<Steps>
  <Step title="Crea el directorio de la Skill">
    Las Skills viven en tu workspace. Crea una carpeta nueva:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

  </Step>

  <Step title="Escribe SKILL.md">
    Crea `SKILL.md` dentro de ese directorio. El frontmatter define los metadatos,
    y el cuerpo en markdown contiene instrucciones para el agente.

    ```markdown
    ---
    name: hello_world
    description: Una Skill sencilla que dice hola.
    ---

    # Hello World Skill

    Cuando el usuario pida un saludo, usa la herramienta `echo` para decir
    "Hello from your custom skill!".
    ```

  </Step>

  <Step title="Añade herramientas (opcional)">
    Puedes definir esquemas de herramientas personalizadas en el frontmatter o indicar al agente
    que use herramientas del sistema existentes (como `exec` o `browser`). Las Skills también pueden
    incluirse dentro de plugins junto con las herramientas que documentan.

  </Step>

  <Step title="Carga la Skill">
    Inicia una nueva sesión para que OpenClaw detecte la Skill:

    ```bash
    # Desde el chat
    /new

    # O reinicia el gateway
    openclaw gateway restart
    ```

    Verifica que la Skill se haya cargado:

    ```bash
    openclaw skills list
    ```

  </Step>

  <Step title="Pruébala">
    Envía un mensaje que deba activar la Skill:

    ```bash
    openclaw agent --message "give me a greeting"
    ```

    O simplemente chatea con el agente y pídele un saludo.

  </Step>
</Steps>

## Referencia de metadatos de Skills

El frontmatter YAML admite estos campos:

| Campo                               | Obligatorio | Descripción                                 |
| ----------------------------------- | ----------- | ------------------------------------------- |
| `name`                              | Sí          | Identificador único (`snake_case`)          |
| `description`                       | Sí          | Descripción de una línea que se muestra al agente |
| `metadata.openclaw.os`              | No          | Filtro de SO (`["darwin"]`, `["linux"]`, etc.) |
| `metadata.openclaw.requires.bins`   | No          | Binarios requeridos en `PATH`               |
| `metadata.openclaw.requires.config` | No          | Claves de configuración requeridas          |

## Prácticas recomendadas

- **Sé conciso** — indica al modelo _qué_ debe hacer, no cómo ser una IA
- **La seguridad primero** — si tu Skill usa `exec`, asegúrate de que los prompts no permitan inyección arbitraria de comandos desde entradas no confiables
- **Prueba localmente** — usa `openclaw agent --message "..."` para probar antes de compartir
- **Usa ClawHub** — explora y contribuye Skills en [ClawHub](https://clawhub.ai)

## Dónde viven las Skills

| Ubicación                        | Precedencia | Alcance                |
| -------------------------------- | ----------- | ---------------------- |
| `\<workspace\>/skills/`         | La más alta | Por agente             |
| `\<workspace\>/.agents/skills/` | Alta        | Por agente del workspace   |
| `~/.agents/skills/`             | Media       | Perfil de agente compartido  |
| `~/.openclaw/skills/`           | Media       | Compartido (todos los agentes)   |
| Agrupadas (incluidas con OpenClaw) | Baja        | Global                |
| `skills.load.extraDirs`         | La más baja | Carpetas compartidas personalizadas |

## Relacionado

- [Referencia de Skills](/tools/skills) — reglas de carga, precedencia y restricción
- [Configuración de Skills](/tools/skills-config) — esquema de configuración `skills.*`
- [ClawHub](/tools/clawhub) — registro público de Skills
- [Creación de plugins](/es/plugins/building-plugins) — los plugins pueden incluir Skills
