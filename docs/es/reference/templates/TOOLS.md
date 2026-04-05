---
read_when:
    - Inicializar manualmente un espacio de trabajo
summary: Plantilla de espacio de trabajo para TOOLS.md
title: Plantilla de TOOLS.md
x-i18n:
    generated_at: "2026-04-05T12:53:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: eed204d57e7221ae0455a87272da2b0730d6aee6ddd2446a851703276e4a96b7
    source_path: reference/templates/TOOLS.md
    workflow: 15
---

# TOOLS.md - Notas locales

Las Skills definen _cómo_ funcionan las herramientas. Este archivo es para _tus_ detalles concretos: lo que es único de tu configuración.

## Qué va aquí

Cosas como:

- Nombres y ubicaciones de cámaras
- Hosts y alias de SSH
- Voces preferidas para TTS
- Nombres de altavoces/salas
- Apodos de dispositivos
- Cualquier cosa específica del entorno

## Ejemplos

```markdown
### Cámaras

- living-room → Área principal, gran angular de 180°
- front-door → Entrada, activada por movimiento

### SSH

- home-server → 192.168.1.100, usuario: admin

### TTS

- Voz preferida: "Nova" (cálida, ligeramente británica)
- Altavoz predeterminado: HomePod de la cocina
```

## ¿Por qué separado?

Las Skills son compartidas. Tu configuración es tuya. Mantenerlas separadas significa que puedes actualizar las Skills sin perder tus notas, y compartir Skills sin filtrar tu infraestructura.

---

Añade cualquier cosa que te ayude a hacer tu trabajo. Esta es tu chuleta.
