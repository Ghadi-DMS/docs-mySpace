---
read_when:
    - Quieres contribuir con hallazgos de seguridad o escenarios de amenazas
    - Revisar o actualizar el modelo de amenazas
summary: Cómo contribuir al modelo de amenazas de OpenClaw
title: Contribuir al modelo de amenazas
x-i18n:
    generated_at: "2026-04-05T12:53:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9cd212d456571a25da63031588d3b584bdfc119e2096b528b97a3f7ec5e4b3db
    source_path: security/CONTRIBUTING-THREAT-MODEL.md
    workflow: 15
---

# Contribuir al modelo de amenazas de OpenClaw

Gracias por ayudar a que OpenClaw sea más seguro. Este modelo de amenazas es un documento vivo y agradecemos contribuciones de cualquier persona; no necesitas ser un experto en seguridad.

## Formas de contribuir

### Añadir una amenaza

¿Has detectado un vector de ataque o un riesgo que no hemos cubierto? Abre un issue en [openclaw/trust](https://github.com/openclaw/trust/issues) y descríbelo con tus propias palabras. No necesitas conocer ningún framework ni completar todos los campos; solo describe el escenario.

**Conviene incluirlo (pero no es obligatorio):**

- El escenario de ataque y cómo podría explotarse
- Qué partes de OpenClaw se ven afectadas (CLI, gateway, canales, ClawHub, servidores MCP, etc.)
- Qué gravedad crees que tiene (baja / media / alta / crítica)
- Cualquier enlace a investigaciones relacionadas, CVE o ejemplos del mundo real

Nos ocuparemos del mapeo de ATLAS, los ID de amenazas y la evaluación de riesgos durante la revisión. Si quieres incluir esos detalles, genial, pero no se espera que lo hagas.

> **Esto es para añadir elementos al modelo de amenazas, no para informar de vulnerabilidades activas.** Si has encontrado una vulnerabilidad explotable, consulta nuestra [página de Trust](https://trust.openclaw.ai) para ver las instrucciones de divulgación responsable.

### Sugerir una mitigación

¿Tienes una idea sobre cómo abordar una amenaza existente? Abre un issue o PR haciendo referencia a la amenaza. Las mitigaciones útiles son específicas y aplicables; por ejemplo, "limitación de tasa por remitente de 10 mensajes/minuto en el gateway" es mejor que "implementar limitación de tasa".

### Proponer una cadena de ataque

Las cadenas de ataque muestran cómo varias amenazas se combinan en un escenario de ataque realista. Si detectas una combinación peligrosa, describe los pasos y cómo un atacante los encadenaría. Una narración breve de cómo se desarrolla el ataque en la práctica es más valiosa que una plantilla formal.

### Corregir o mejorar contenido existente

Erratas, aclaraciones, información desactualizada, mejores ejemplos: los PR son bienvenidos, no hace falta abrir un issue.

## Qué usamos

### MITRE ATLAS

Este modelo de amenazas se basa en [MITRE ATLAS](https://atlas.mitre.org/) (Adversarial Threat Landscape for AI Systems), un framework diseñado específicamente para amenazas de IA/ML como la inyección de prompts, el uso indebido de herramientas y la explotación de agentes. No necesitas conocer ATLAS para contribuir; durante la revisión asignamos las contribuciones al framework.

### ID de amenazas

Cada amenaza recibe un ID como `T-EXEC-003`. Las categorías son:

| Code    | Category                                   |
| ------- | ------------------------------------------ |
| RECON   | Reconocimiento - recopilación de información |
| ACCESS  | Acceso inicial - obtener entrada             |
| EXEC    | Ejecución - realizar acciones maliciosas     |
| PERSIST | Persistencia - mantener el acceso            |
| EVADE   | Evasión de defensas - evitar la detección    |
| DISC    | Descubrimiento - conocer el entorno          |
| EXFIL   | Exfiltración - robo de datos                 |
| IMPACT  | Impacto - daño o interrupción                |

Los ID son asignados por los mantenedores durante la revisión. No necesitas elegir uno.

### Niveles de riesgo

| Level        | Meaning                                                           |
| ------------ | ----------------------------------------------------------------- |
| **Critical** | Compromiso total del sistema, o alta probabilidad + impacto crítico |
| **High**     | Daño significativo probable, o probabilidad media + impacto crítico |
| **Medium**   | Riesgo moderado, o baja probabilidad + alto impacto                |
| **Low**      | Poco probable e impacto limitado                                   |

Si no estás seguro del nivel de riesgo, simplemente describe el impacto y nosotros lo evaluaremos.

## Proceso de revisión

1. **Triaje** - Revisamos las nuevas contribuciones en un plazo de 48 horas
2. **Evaluación** - Verificamos la viabilidad, asignamos el mapeo de ATLAS y el ID de amenaza, y validamos el nivel de riesgo
3. **Documentación** - Nos aseguramos de que todo tenga el formato correcto y esté completo
4. **Fusión** - Se añade al modelo de amenazas y a la visualización

## Recursos

- [Sitio web de ATLAS](https://atlas.mitre.org/)
- [Técnicas de ATLAS](https://atlas.mitre.org/techniques/)
- [Casos de estudio de ATLAS](https://atlas.mitre.org/studies/)
- [Modelo de amenazas de OpenClaw](/security/THREAT-MODEL-ATLAS)

## Contacto

- **Vulnerabilidades de seguridad:** Consulta nuestra [página de Trust](https://trust.openclaw.ai) para ver las instrucciones de reporte
- **Preguntas sobre el modelo de amenazas:** Abre un issue en [openclaw/trust](https://github.com/openclaw/trust/issues)
- **Chat general:** Canal #security de Discord

## Reconocimiento

Los colaboradores del modelo de amenazas son reconocidos en los agradecimientos del modelo de amenazas, las notas de la versión y el salón de la fama de seguridad de OpenClaw por contribuciones significativas.
