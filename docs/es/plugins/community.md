---
read_when:
    - Quieres encontrar plugins de OpenClaw de terceros
    - Quieres publicar o listar tu propio plugin
summary: 'Plugins de OpenClaw mantenidos por la comunidad: explorar, instalar y enviar el tuyo'
title: Plugins de la comunidad
x-i18n:
    generated_at: "2026-04-05T12:49:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 01804563a63399fe564b0cd9b9aadef32e5211b63d8467fdbbd1f988200728de
    source_path: plugins/community.md
    workflow: 15
---

# Plugins de la comunidad

Los plugins de la comunidad son paquetes de terceros que amplían OpenClaw con nuevos
canales, herramientas, proveedores u otras capacidades. Son creados y mantenidos
por la comunidad, se publican en [ClawHub](/tools/clawhub) o npm, y
se pueden instalar con un solo comando.

ClawHub es la superficie canónica de descubrimiento para plugins de la comunidad. No abras
PR solo de documentación solo para agregar tu plugin aquí por descubribilidad; publícalo en
ClawHub en su lugar.

```bash
openclaw plugins install <package-name>
```

OpenClaw primero comprueba ClawHub y luego usa npm automáticamente como respaldo.

## Plugins listados

### Codex App Server Bridge

Puente independiente de OpenClaw para conversaciones de Codex App Server. Vincula un chat a
un hilo de Codex, habla con él en texto plano y contrólalo con comandos nativos
del chat para reanudar, planificar, revisar, seleccionar modelo, compactar y más.

- **npm:** `openclaw-codex-app-server`
- **repo:** [github.com/pwrdrvr/openclaw-codex-app-server](https://github.com/pwrdrvr/openclaw-codex-app-server)

```bash
openclaw plugins install openclaw-codex-app-server
```

### DingTalk

Integración de robot empresarial mediante modo Stream. Admite texto, imágenes y
mensajes de archivo desde cualquier cliente de DingTalk.

- **npm:** `@largezhou/ddingtalk`
- **repo:** [github.com/largezhou/openclaw-dingtalk](https://github.com/largezhou/openclaw-dingtalk)

```bash
openclaw plugins install @largezhou/ddingtalk
```

### Lossless Claw (LCM)

Plugin Lossless Context Management para OpenClaw. Resumen de conversaciones
basado en DAG con compactación incremental: preserva toda la fidelidad del contexto
mientras reduce el uso de tokens.

- **npm:** `@martian-engineering/lossless-claw`
- **repo:** [github.com/Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw)

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

### Opik

Plugin oficial que exporta trazas de agentes a Opik. Supervisa el comportamiento del agente,
coste, tokens, errores y más.

- **npm:** `@opik/opik-openclaw`
- **repo:** [github.com/comet-ml/opik-openclaw](https://github.com/comet-ml/opik-openclaw)

```bash
openclaw plugins install @opik/opik-openclaw
```

### QQbot

Conecta OpenClaw con QQ mediante la API de QQ Bot. Admite chats privados, menciones
en grupos, mensajes de canal y contenido multimedia enriquecido, incluidos voz, imágenes, vídeos
y archivos.

- **npm:** `@tencent-connect/openclaw-qqbot`
- **repo:** [github.com/tencent-connect/openclaw-qqbot](https://github.com/tencent-connect/openclaw-qqbot)

```bash
openclaw plugins install @tencent-connect/openclaw-qqbot
```

### wecom

Plugin del canal WeCom para OpenClaw del equipo Tencent WeCom. Impulsado por
conexiones persistentes WebSocket de WeCom Bot, admite mensajes directos y chats de grupo,
respuestas en streaming, mensajería proactiva, procesamiento de imágenes/archivos, formato
Markdown, control de acceso integrado y Skills de documentos/reuniones/mensajería.

- **npm:** `@wecom/wecom-openclaw-plugin`
- **repo:** [github.com/WecomTeam/wecom-openclaw-plugin](https://github.com/WecomTeam/wecom-openclaw-plugin)

```bash
openclaw plugins install @wecom/wecom-openclaw-plugin
```

## Envía tu plugin

Damos la bienvenida a plugins de la comunidad que sean útiles, estén documentados y sean seguros de operar.

<Steps>
  <Step title="Publica en ClawHub o npm">
    Tu plugin debe poder instalarse mediante `openclaw plugins install \<package-name\>`.
    Publícalo en [ClawHub](/tools/clawhub) (preferido) o en npm.
    Consulta [Building Plugins](/plugins/building-plugins) para ver la guía completa.

  </Step>

  <Step title="Aloja el código en GitHub">
    El código fuente debe estar en un repositorio público con documentación de configuración y un
    gestor de incidencias.

  </Step>

  <Step title="Usa PR de documentación solo para cambios en la documentación fuente">
    No necesitas un PR de documentación solo para que tu plugin sea descubrible. Publícalo
    en ClawHub en su lugar.

    Abre un PR de documentación solo cuando la documentación fuente de OpenClaw necesite un cambio
    real de contenido, como corregir instrucciones de instalación o agregar documentación
    entre repositorios que pertenezca al conjunto principal de documentación.

  </Step>
</Steps>

## Nivel de calidad

| Requisito                  | Por qué                                          |
| -------------------------- | ------------------------------------------------ |
| Publicado en ClawHub o npm | Las personas usuarias necesitan que `openclaw plugins install` funcione |
| Repositorio público en GitHub | Revisión del código, seguimiento de incidencias, transparencia |
| Documentación de configuración y uso | Las personas usuarias necesitan saber cómo configurarlo |
| Mantenimiento activo       | Actualizaciones recientes o gestión receptiva de incidencias |

Los wrappers de bajo esfuerzo, propiedad poco clara o paquetes sin mantenimiento pueden ser rechazados.

## Relacionado

- [Install and Configure Plugins](/tools/plugin) — cómo instalar cualquier plugin
- [Building Plugins](/plugins/building-plugins) — crea el tuyo propio
- [Plugin Manifest](/plugins/manifest) — esquema del manifiesto
