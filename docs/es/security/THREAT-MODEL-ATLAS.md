---
read_when:
    - Revisando la postura de seguridad o escenarios de amenazas
    - Trabajando en funciones de seguridad o respuestas de auditoría
summary: Modelo de amenazas de OpenClaw mapeado al marco MITRE ATLAS
title: Modelo de amenazas (MITRE ATLAS)
x-i18n:
    generated_at: "2026-04-05T12:54:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 05561381c73e8efe20c8b59cd717e66447ee43988018e9670161cc63e650f2bf
    source_path: security/THREAT-MODEL-ATLAS.md
    workflow: 15
---

# Modelo de amenazas de OpenClaw v1.0

## Marco MITRE ATLAS

**Version:** 1.0-draft
**Last Updated:** 2026-02-04
**Methodology:** MITRE ATLAS + diagramas de flujo de datos
**Framework:** [MITRE ATLAS](https://atlas.mitre.org/) (Panorama de amenazas adversarias para sistemas de IA)

### Atribución del marco

Este modelo de amenazas se basa en [MITRE ATLAS](https://atlas.mitre.org/), el marco estándar de la industria para documentar amenazas adversarias a sistemas de IA/ML. ATLAS es mantenido por [MITRE](https://www.mitre.org/) en colaboración con la comunidad de seguridad de IA.

**Recursos clave de ATLAS:**

- [ATLAS Techniques](https://atlas.mitre.org/techniques/)
- [ATLAS Tactics](https://atlas.mitre.org/tactics/)
- [ATLAS Case Studies](https://atlas.mitre.org/studies/)
- [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data)
- [Contributing to ATLAS](https://atlas.mitre.org/resources/contribute)

### Cómo contribuir a este modelo de amenazas

Este es un documento vivo mantenido por la comunidad de OpenClaw. Consulta [CONTRIBUTING-THREAT-MODEL.md](/security/CONTRIBUTING-THREAT-MODEL) para ver las directrices sobre cómo contribuir:

- Informar sobre nuevas amenazas
- Actualizar amenazas existentes
- Proponer cadenas de ataque
- Sugerir mitigaciones

---

## 1. Introducción

### 1.1 Propósito

Este modelo de amenazas documenta amenazas adversarias para la plataforma de agentes de IA OpenClaw y el marketplace de Skills ClawHub, utilizando el marco MITRE ATLAS diseñado específicamente para sistemas de IA/ML.

### 1.2 Alcance

| Component              | Included | Notes                                            |
| ---------------------- | -------- | ------------------------------------------------ |
| Runtime del agente OpenClaw | Sí      | Ejecución central del agente, llamadas a herramientas, sesiones       |
| Gateway                | Sí      | Autenticación, enrutamiento, integración de canales     |
| Integraciones de canales   | Sí      | WhatsApp, Telegram, Discord, Signal, Slack, etc. |
| Marketplace ClawHub    | Sí      | Publicación, moderación y distribución de Skills       |
| Servidores MCP            | Sí      | Proveedores de herramientas externos                          |
| Dispositivos de usuario           | Parcial  | Apps móviles, clientes de escritorio                     |

### 1.3 Fuera de alcance

No hay nada explícitamente fuera del alcance de este modelo de amenazas.

---

## 2. Arquitectura del sistema

### 2.1 Límites de confianza

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZONA NO CONFIABLE                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  WhatsApp   │  │  Telegram   │  │   Discord   │  ...         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
└─────────┼────────────────┼────────────────┼──────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│               LÍMITE DE CONFIANZA 1: Acceso al canal              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      GATEWAY                              │   │
│  │  • Emparejamiento de dispositivos (1 h en DM / período de gracia de 5 min para nodo) │   │
│  │  • Validación de AllowFrom / AllowList                       │   │
│  │  • Autenticación con token/contraseña/Tailscale                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              LÍMITE DE CONFIANZA 2: Aislamiento de sesiones       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 SESIONES DEL AGENTE                        │   │
│  │  • Clave de sesión = agent:channel:peer                   │   │
│  │  • Políticas de herramientas por agente                                │   │
│  │  • Registro de transcripciones                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│             LÍMITE DE CONFIANZA 3: Ejecución de herramientas      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               SANDBOX DE EJECUCIÓN                        │   │
│  │  • Sandbox de Docker O host (exec-approvals)                │   │
│  │  • Ejecución remota de Node                                  │   │
│  │  • Protección SSRF (fijación de DNS + bloqueo de IP)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              LÍMITE DE CONFIANZA 4: Contenido externo            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           URLS / CORREOS / WEBHOOKS OBTENIDOS             │   │
│  │  • Envoltura de contenido externo (etiquetas XML)                   │   │
│  │  • Inyección de avisos de seguridad                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              LÍMITE DE CONFIANZA 5: Cadena de suministro         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLAWHUB                              │   │
│  │  • Publicación de Skills (semver, SKILL.md obligatorio)           │   │
│  │  • Marcas de moderación basadas en patrones                         │   │
│  │  • Escaneo con VirusTotal (próximamente)                      │   │
│  │  • Verificación de antigüedad de cuenta de GitHub                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Flujos de datos

| Flow | Source  | Destination | Data               | Protection           |
| ---- | ------- | ----------- | ------------------ | -------------------- |
| F1   | Canal | Gateway     | Mensajes de usuario      | TLS, AllowFrom       |
| F2   | Gateway | Agente       | Mensajes enrutados    | Aislamiento de sesiones    |
| F3   | Agente   | Herramientas       | Invocaciones de herramientas   | Aplicación de políticas   |
| F4   | Agente   | Externo    | solicitudes `web_fetch` | Bloqueo SSRF        |
| F5   | ClawHub | Agente       | Código de Skill         | Moderación, escaneo |
| F6   | Agente   | Canal     | Respuestas          | Filtrado de salida     |

---

## 3. Análisis de amenazas por táctica de ATLAS

### 3.1 Reconocimiento (AML.TA0002)

#### T-RECON-001: Descubrimiento del endpoint del agente

| Attribute               | Value                                                                |
| ----------------------- | -------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0006 - Escaneo activo                                          |
| **Description**         | El atacante escanea endpoints expuestos del gateway de OpenClaw                |
| **Attack Vector**       | Escaneo de red, consultas en shodan, enumeración DNS                    |
| **Affected Components** | Gateway, endpoints API expuestos                                       |
| **Current Mitigations** | Opción de autenticación con Tailscale, vinculación a local loopback de forma predeterminada                   |
| **Residual Risk**       | Medio - Gateways públicos detectables                                |
| **Recommendations**     | Documentar un despliegue seguro, añadir limitación de tasa en endpoints de descubrimiento |

#### T-RECON-002: Sondeo de integraciones de canales

| Attribute               | Value                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0006 - Escaneo activo                                        |
| **Description**         | El atacante sondea canales de mensajería para identificar cuentas gestionadas por IA |
| **Attack Vector**       | Envío de mensajes de prueba, observación de patrones de respuesta                 |
| **Affected Components** | Todas las integraciones de canales                                           |
| **Current Mitigations** | Ninguna específica                                                      |
| **Residual Risk**       | Bajo - Valor limitado del descubrimiento por sí solo                           |
| **Recommendations**     | Considerar aleatorización en el tiempo de respuesta                             |

---

### 3.2 Acceso inicial (AML.TA0004)

#### T-ACCESS-001: Intercepción del código de emparejamiento

| Attribute               | Value                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - Acceso a la API de inferencia del modelo de IA                                                                     |
| **Description**         | El atacante intercepta el código de emparejamiento durante el período de gracia del emparejamiento (1 h para emparejamiento de canal DM, 5 min para emparejamiento de nodos) |
| **Attack Vector**       | Observación por encima del hombro, sniffing de red, ingeniería social                                                        |
| **Affected Components** | Sistema de emparejamiento de dispositivos                                                                                         |
| **Current Mitigations** | Expiración de 1 h (emparejamiento DM) / expiración de 5 min (emparejamiento de nodos), códigos enviados a través del canal existente                            |
| **Residual Risk**       | Medio - El período de gracia puede explotarse                                                                             |
| **Recommendations**     | Reducir el período de gracia, añadir un paso de confirmación                                                                    |

#### T-ACCESS-002: Suplantación de AllowFrom

| Attribute               | Value                                                                          |
| ----------------------- | ------------------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0040 - Acceso a la API de inferencia del modelo de IA                                      |
| **Description**         | El atacante suplanta la identidad del remitente permitido en el canal                             |
| **Attack Vector**       | Depende del canal: suplantación de número de teléfono, suplantación de nombre de usuario             |
| **Affected Components** | Validación de AllowFrom por canal                                               |
| **Current Mitigations** | Verificación de identidad específica del canal                                         |
| **Residual Risk**       | Medio - Algunos canales son vulnerables a la suplantación                                  |
| **Recommendations**     | Documentar riesgos específicos por canal, añadir verificación criptográfica cuando sea posible |

#### T-ACCESS-003: Robo de tokens

| Attribute               | Value                                                       |
| ----------------------- | ----------------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - Acceso a la API de inferencia del modelo de IA                   |
| **Description**         | El atacante roba tokens de autenticación de archivos de configuración     |
| **Attack Vector**       | Malware, acceso no autorizado al dispositivo, exposición de copias de seguridad de configuración |
| **Affected Components** | `~/.openclaw/credentials/`, almacenamiento de configuración                    |
| **Current Mitigations** | Permisos de archivo                                            |
| **Residual Risk**       | Alto - Los tokens se almacenan en texto plano                           |
| **Recommendations**     | Implementar cifrado de tokens en reposo, añadir rotación de tokens      |

---

### 3.3 Ejecución (AML.TA0005)

#### T-EXEC-001: Inyección directa de prompts

| Attribute               | Value                                                                                     |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0051.000 - Inyección de prompts en LLM: directa                                              |
| **Description**         | El atacante envía prompts manipulados para alterar el comportamiento del agente                               |
| **Attack Vector**       | Mensajes del canal que contienen instrucciones adversarias                                      |
| **Affected Components** | LLM del agente, todas las superficies de entrada                                                             |
| **Current Mitigations** | Detección de patrones, envoltura de contenido externo                                              |
| **Residual Risk**       | Crítico - Solo detección, sin bloqueo; los ataques sofisticados la eluden                      |
| **Recommendations**     | Implementar defensa multicapa, validación de salida, confirmación del usuario para acciones sensibles |

#### T-EXEC-002: Inyección indirecta de prompts

| Attribute               | Value                                                       |
| ----------------------- | ----------------------------------------------------------- |
| **ATLAS ID**            | AML.T0051.001 - Inyección de prompts en LLM: indirecta              |
| **Description**         | El atacante incrusta instrucciones maliciosas en contenido obtenido   |
| **Attack Vector**       | URLs maliciosas, correos envenenados, webhooks comprometidos       |
| **Affected Components** | `web_fetch`, ingestión de correo, fuentes de datos externas           |
| **Current Mitigations** | Envoltura de contenido con etiquetas XML y aviso de seguridad          |
| **Residual Risk**       | Alto - El LLM puede ignorar las instrucciones de envoltura                  |
| **Recommendations**     | Implementar sanitización de contenido, separar contextos de ejecución |

#### T-EXEC-003: Inyección de argumentos de herramientas

| Attribute               | Value                                                        |
| ----------------------- | ------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0051.000 - Inyección de prompts en LLM: directa                 |
| **Description**         | El atacante manipula argumentos de herramientas mediante inyección de prompts |
| **Attack Vector**       | Prompts manipulados que influyen en los valores de parámetros de herramientas         |
| **Affected Components** | Todas las invocaciones de herramientas                                         |
| **Current Mitigations** | `exec approvals` para comandos peligrosos                        |
| **Residual Risk**       | Alto - Depende del juicio del usuario                               |
| **Recommendations**     | Implementar validación de argumentos, llamadas a herramientas parametrizadas      |

#### T-EXEC-004: Evasión de la aprobación de ejecución

| Attribute               | Value                                                      |
| ----------------------- | ---------------------------------------------------------- |
| **ATLAS ID**            | AML.T0043 - Crear datos adversarios                         |
| **Description**         | El atacante crea comandos que evaden la allowlist de aprobación    |
| **Attack Vector**       | Ofuscación de comandos, explotación de alias, manipulación de rutas |
| **Affected Components** | `exec-approvals.ts`, allowlist de comandos                       |
| **Current Mitigations** | Allowlist + modo de consulta                                       |
| **Residual Risk**       | Alto - Sin sanitización de comandos                             |
| **Recommendations**     | Implementar normalización de comandos, ampliar la blocklist          |

---

### 3.4 Persistencia (AML.TA0006)

#### T-PERSIST-001: Instalación de Skill maliciosa

| Attribute               | Value                                                                    |
| ----------------------- | ------------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0010.001 - Compromiso de la cadena de suministro: software de IA                     |
| **Description**         | El atacante publica una Skill maliciosa en ClawHub                            |
| **Attack Vector**       | Crear una cuenta, publicar una Skill con código malicioso oculto                 |
| **Affected Components** | ClawHub, carga de Skills, ejecución del agente                                  |
| **Current Mitigations** | Verificación de antigüedad de la cuenta de GitHub, marcas de moderación basadas en patrones          |
| **Residual Risk**       | Crítico - Sin sandboxing, revisión limitada                                 |
| **Recommendations**     | Integración con VirusTotal (en curso), sandboxing de Skills, revisión comunitaria |

#### T-PERSIST-002: Envenenamiento de actualizaciones de Skills

| Attribute               | Value                                                          |
| ----------------------- | -------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0010.001 - Compromiso de la cadena de suministro: software de IA           |
| **Description**         | El atacante compromete una Skill popular y envía una actualización maliciosa |
| **Attack Vector**       | Compromiso de cuenta, ingeniería social al propietario de la Skill          |
| **Affected Components** | Versionado de ClawHub, flujos de actualización automática                          |
| **Current Mitigations** | Huella digital de versiones                                         |
| **Residual Risk**       | Alto - Las actualizaciones automáticas pueden incorporar versiones maliciosas                |
| **Recommendations**     | Implementar firma de actualizaciones, capacidad de reversión, fijación de versiones |

#### T-PERSIST-003: Manipulación de configuración del agente

| Attribute               | Value                                                           |
| ----------------------- | --------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0010.002 - Compromiso de la cadena de suministro: datos                   |
| **Description**         | El atacante modifica la configuración del agente para mantener el acceso         |
| **Attack Vector**       | Modificación de archivos de configuración, inyección de ajustes                    |
| **Affected Components** | Configuración del agente, políticas de herramientas                                     |
| **Current Mitigations** | Permisos de archivo                                                |
| **Residual Risk**       | Medio - Requiere acceso local                                  |
| **Recommendations**     | Verificación de integridad de configuración, registro de auditoría para cambios de configuración |

---

### 3.5 Evasión de defensas (AML.TA0007)

#### T-EVADE-001: Evasión de patrones de moderación

| Attribute               | Value                                                                  |
| ----------------------- | ---------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0043 - Crear datos adversarios                                     |
| **Description**         | El atacante crea contenido de Skill para evadir patrones de moderación             |
| **Attack Vector**       | Homoglifos Unicode, trucos de codificación, carga dinámica                   |
| **Affected Components** | Moderación de ClawHub `moderation.ts`                                                  |
| **Current Mitigations** | `FLAG_RULES` basadas en patrones                                               |
| **Residual Risk**       | Alto - Regex simples se eluden con facilidad                                    |
| **Recommendations**     | Añadir análisis de comportamiento (VirusTotal Code Insight), detección basada en AST |

#### T-EVADE-002: Escape de la envoltura de contenido

| Attribute               | Value                                                     |
| ----------------------- | --------------------------------------------------------- |
| **ATLAS ID**            | AML.T0043 - Crear datos adversarios                        |
| **Description**         | El atacante crea contenido que escapa del contexto de la envoltura XML  |
| **Attack Vector**       | Manipulación de etiquetas, confusión de contexto, sobrescritura de instrucciones |
| **Affected Components** | Envoltura de contenido externo                                 |
| **Current Mitigations** | Etiquetas XML + aviso de seguridad                                |
| **Residual Risk**       | Medio - Se descubren regularmente escapes novedosos               |
| **Recommendations**     | Múltiples capas de envoltura, validación en el lado de la salida           |

---

### 3.6 Descubrimiento (AML.TA0008)

#### T-DISC-001: Enumeración de herramientas

| Attribute               | Value                                                 |
| ----------------------- | ----------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - Acceso a la API de inferencia del modelo de IA             |
| **Description**         | El atacante enumera herramientas disponibles mediante prompting |
| **Attack Vector**       | Consultas del estilo "¿Qué herramientas tienes?"               |
| **Affected Components** | Registro de herramientas del agente                                   |
| **Current Mitigations** | Ninguna específica                                         |
| **Residual Risk**       | Bajo - Las herramientas suelen estar documentadas                      |
| **Recommendations**     | Considerar controles de visibilidad de herramientas                     |

#### T-DISC-002: Extracción de datos de sesión

| Attribute               | Value                                                 |
| ----------------------- | ----------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - Acceso a la API de inferencia del modelo de IA             |
| **Description**         | El atacante extrae datos sensibles del contexto de sesión |
| **Attack Vector**       | Consultas de tipo "¿De qué hablamos?", sondeo de contexto       |
| **Affected Components** | Transcripciones de sesión, ventana de contexto                   |
| **Current Mitigations** | Aislamiento de sesiones por remitente                          |
| **Residual Risk**       | Medio - Datos accesibles dentro de la sesión               |
| **Recommendations**     | Implementar redacción de datos sensibles en el contexto         |

---

### 3.7 Recolección y exfiltración (AML.TA0009, AML.TA0010)

#### T-EXFIL-001: Robo de datos mediante web_fetch

| Attribute               | Value                                                                  |
| ----------------------- | ---------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0009 - Recolección                                                 |
| **Description**         | El atacante exfiltra datos indicando al agente que los envíe a una URL externa |
| **Attack Vector**       | Inyección de prompts que provoca que el agente haga POST de datos al servidor del atacante         |
| **Affected Components** | Herramienta `web_fetch`                                                         |
| **Current Mitigations** | Bloqueo SSRF para redes internas                                    |
| **Residual Risk**       | Alto - Se permiten URLs externas                                         |
| **Recommendations**     | Implementar allowlisting de URL, conciencia sobre clasificación de datos              |

#### T-EXFIL-002: Envío no autorizado de mensajes

| Attribute               | Value                                                            |
| ----------------------- | ---------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0009 - Recolección                                           |
| **Description**         | El atacante hace que el agente envíe mensajes que contienen datos sensibles |
| **Attack Vector**       | Inyección de prompts que provoca que el agente envíe mensajes al atacante               |
| **Affected Components** | Herramienta de mensajes, integraciones de canales                               |
| **Current Mitigations** | Restricción del envío saliente                                        |
| **Residual Risk**       | Medio - La restricción puede eludirse                                  |
| **Recommendations**     | Requerir confirmación explícita para destinatarios nuevos                 |

#### T-EXFIL-003: Recolección de credenciales

| Attribute               | Value                                                   |
| ----------------------- | ------------------------------------------------------- |
| **ATLAS ID**            | AML.T0009 - Recolección                                  |
| **Description**         | Una Skill maliciosa recolecta credenciales del contexto del agente |
| **Attack Vector**       | El código de la Skill lee variables de entorno, archivos de configuración    |
| **Affected Components** | Entorno de ejecución de Skills                             |
| **Current Mitigations** | Ninguna específica para las Skills                                 |
| **Residual Risk**       | Crítico - Las Skills se ejecutan con privilegios del agente             |
| **Recommendations**     | Sandboxing de Skills, aislamiento de credenciales                  |

---

### 3.8 Impacto (AML.TA0011)

#### T-IMPACT-001: Ejecución no autorizada de comandos

| Attribute               | Value                                               |
| ----------------------- | --------------------------------------------------- |
| **ATLAS ID**            | AML.T0031 - Erosionar la integridad del modelo de IA                |
| **Description**         | El atacante ejecuta comandos arbitrarios en el sistema del usuario |
| **Attack Vector**       | Inyección de prompts combinada con evasión de aprobación de ejecución |
| **Affected Components** | Herramienta Bash, ejecución de comandos                        |
| **Current Mitigations** | `exec approvals`, opción de sandbox de Docker               |
| **Residual Risk**       | Crítico - Ejecución en host sin sandbox           |
| **Recommendations**     | Usar sandbox de forma predeterminada, mejorar la UX de aprobación             |

#### T-IMPACT-002: Agotamiento de recursos (DoS)

| Attribute               | Value                                              |
| ----------------------- | -------------------------------------------------- |
| **ATLAS ID**            | AML.T0031 - Erosionar la integridad del modelo de IA               |
| **Description**         | El atacante agota créditos de API o recursos de cómputo |
| **Attack Vector**       | Inundación automatizada de mensajes, llamadas costosas a herramientas   |
| **Affected Components** | Gateway, sesiones del agente, proveedor de API              |
| **Current Mitigations** | Ninguna                                               |
| **Residual Risk**       | Alto - Sin limitación de tasa                            |
| **Recommendations**     | Implementar límites por remitente, presupuestos de costo     |

#### T-IMPACT-003: Daño reputacional

| Attribute               | Value                                                   |
| ----------------------- | ------------------------------------------------------- |
| **ATLAS ID**            | AML.T0031 - Erosionar la integridad del modelo de IA                    |
| **Description**         | El atacante hace que el agente envíe contenido dañino/ofensivo |
| **Attack Vector**       | Inyección de prompts que provoca respuestas inapropiadas        |
| **Affected Components** | Generación de salida, mensajería por canales                    |
| **Current Mitigations** | Políticas de contenido del proveedor de LLM                           |
| **Residual Risk**       | Medio - Los filtros del proveedor son imperfectos                     |
| **Recommendations**     | Capa de filtrado de salida, controles de usuario                   |

---

## 4. Análisis de la cadena de suministro de ClawHub

### 4.1 Controles de seguridad actuales

| Control              | Implementación              | Efectividad                                        |
| -------------------- | --------------------------- | ---------------------------------------------------- |
| Antigüedad de cuenta de GitHub   | `requireGitHubAccountAge()` | Media - Eleva el listón para atacantes nuevos                |
| Sanitización de rutas    | `sanitizePath()`            | Alta - Evita path traversal                       |
| Validación de tipo de archivo | `isTextFile()`              | Media - Solo archivos de texto, pero aún pueden ser maliciosos |
| Límites de tamaño          | Paquete total de 50 MB           | Alta - Evita agotamiento de recursos                  |
| `SKILL.md` obligatorio    | Léame obligatorio            | Bajo valor de seguridad - Solo informativo              |
| Moderación por patrones   | `FLAG_RULES` en `moderation.ts` | Baja - Fácil de eludir                                |
| Estado de moderación    | Campo `moderationStatus`    | Medio - Posible revisión manual                      |

### 4.2 Patrones de marcas de moderación

Patrones actuales en `moderation.ts`:

```javascript
// Identificadores conocidos como maliciosos
/(keepcold131\/ClawdAuthenticatorTool|ClawdAuthenticatorTool)/i

// Palabras clave sospechosas
/(malware|stealer|phish|phishing|keylogger)/i
/(api[-_ ]?key|token|password|private key|secret)/i
/(wallet|seed phrase|mnemonic|crypto)/i
/(discord\.gg|webhook|hooks\.slack)/i
/(curl[^\n]+\|\s*(sh|bash))/i
/(bit\.ly|tinyurl\.com|t\.co|goo\.gl|is\.gd)/i
```

**Limitaciones:**

- Solo comprueba `slug`, `displayName`, `summary`, frontmatter, metadatos y rutas de archivo
- No analiza el contenido real del código de la Skill
- Regex simples se eluden con facilidad mediante ofuscación
- Sin análisis de comportamiento

### 4.3 Mejoras planificadas

| Improvement            | Status                                | Impact                                                                |
| ---------------------- | ------------------------------------- | --------------------------------------------------------------------- |
| Integración con VirusTotal | En progreso                           | Alto - Análisis de comportamiento con Code Insight                               |
| Informes de la comunidad    | Parcial (existe la tabla `skillReports`) | Medio                                                                |
| Registro de auditoría          | Parcial (existe la tabla `auditLogs`)    | Medio                                                                |
| Sistema de insignias           | Implementado                           | Medio - `highlighted`, `official`, `deprecated`, `redactionApproved` |

---

## 5. Matriz de riesgo

### 5.1 Probabilidad frente a impacto

| Threat ID     | Likelihood | Impact   | Risk Level   | Priority |
| ------------- | ---------- | -------- | ------------ | -------- |
| T-EXEC-001    | Alta       | Crítico | **Crítico** | P0       |
| T-PERSIST-001 | Alta       | Crítico | **Crítico** | P0       |
| T-EXFIL-003   | Media     | Crítico | **Crítico** | P0       |
| T-IMPACT-001  | Media     | Crítico | **Alto**     | P1       |
| T-EXEC-002    | Alta       | Alto     | **Alto**     | P1       |
| T-EXEC-004    | Media     | Alto     | **Alto**     | P1       |
| T-ACCESS-003  | Media     | Alto     | **Alto**     | P1       |
| T-EXFIL-001   | Media     | Alto     | **Alto**     | P1       |
| T-IMPACT-002  | Alta       | Medio   | **Alto**     | P1       |
| T-EVADE-001   | Alta       | Medio   | **Medio**   | P2       |
| T-ACCESS-001  | Baja        | Alto     | **Medio**   | P2       |
| T-ACCESS-002  | Baja        | Alto     | **Medio**   | P2       |
| T-PERSIST-002 | Baja        | Alto     | **Medio**   | P2       |

### 5.2 Cadenas de ataque de ruta crítica

**Cadena de ataque 1: Robo de datos basado en Skills**

```
T-PERSIST-001 → T-EVADE-001 → T-EXFIL-003
(Publicar Skill maliciosa) → (Evadir moderación) → (Recolectar credenciales)
```

**Cadena de ataque 2: Inyección de prompts a RCE**

```
T-EXEC-001 → T-EXEC-004 → T-IMPACT-001
(Inyectar prompt) → (Evadir aprobación de ejecución) → (Ejecutar comandos)
```

**Cadena de ataque 3: Inyección indirecta mediante contenido obtenido**

```
T-EXEC-002 → T-EXFIL-001 → Exfiltración externa
(Envenenar contenido de URL) → (El agente lo obtiene y sigue instrucciones) → (Datos enviados al atacante)
```

---

## 6. Resumen de recomendaciones

### 6.1 Inmediatas (P0)

| ID    | Recommendation                              | Addresses                  |
| ----- | ------------------------------------------- | -------------------------- |
| R-001 | Completar la integración con VirusTotal             | T-PERSIST-001, T-EVADE-001 |
| R-002 | Implementar sandboxing de Skills                  | T-PERSIST-001, T-EXFIL-003 |
| R-003 | Añadir validación de salida para acciones sensibles | T-EXEC-001, T-EXEC-002     |

### 6.2 Corto plazo (P1)

| ID    | Recommendation                           | Addresses    |
| ----- | ---------------------------------------- | ------------ |
| R-004 | Implementar limitación de tasa                  | T-IMPACT-002 |
| R-005 | Añadir cifrado de tokens en reposo             | T-ACCESS-003 |
| R-006 | Mejorar la UX y validación de aprobación de ejecución  | T-EXEC-004   |
| R-007 | Implementar allowlisting de URL para `web_fetch` | T-EXFIL-001  |

### 6.3 Medio plazo (P2)

| ID    | Recommendation                                        | Addresses     |
| ----- | ----------------------------------------------------- | ------------- |
| R-008 | Añadir verificación criptográfica de canales cuando sea posible | T-ACCESS-002  |
| R-009 | Implementar verificación de integridad de configuración               | T-PERSIST-003 |
| R-010 | Añadir firma de actualizaciones y fijación de versiones                | T-PERSIST-002 |

---

## 7. Apéndices

### 7.1 Mapeo de técnicas de ATLAS

| ATLAS ID      | Technique Name                 | OpenClaw Threats                                                 |
| ------------- | ------------------------------ | ---------------------------------------------------------------- |
| AML.T0006     | Escaneo activo                | T-RECON-001, T-RECON-002                                         |
| AML.T0009     | Recolección                     | T-EXFIL-001, T-EXFIL-002, T-EXFIL-003                            |
| AML.T0010.001 | Cadena de suministro: software de IA      | T-PERSIST-001, T-PERSIST-002                                     |
| AML.T0010.002 | Cadena de suministro: datos             | T-PERSIST-003                                                    |
| AML.T0031     | Erosionar la integridad del modelo de IA       | T-IMPACT-001, T-IMPACT-002, T-IMPACT-003                         |
| AML.T0040     | Acceso a la API de inferencia del modelo de IA  | T-ACCESS-001, T-ACCESS-002, T-ACCESS-003, T-DISC-001, T-DISC-002 |
| AML.T0043     | Crear datos adversarios         | T-EXEC-004, T-EVADE-001, T-EVADE-002                             |
| AML.T0051.000 | Inyección de prompts en LLM: directa   | T-EXEC-001, T-EXEC-003                                           |
| AML.T0051.001 | Inyección de prompts en LLM: indirecta | T-EXEC-002                                                       |

### 7.2 Archivos clave de seguridad

| Path                                | Purpose                     | Risk Level   |
| ----------------------------------- | --------------------------- | ------------ |
| `src/infra/exec-approvals.ts`       | Lógica de aprobación de comandos      | **Crítico** |
| `src/gateway/auth.ts`               | Autenticación del Gateway      | **Crítico** |
| `src/infra/net/ssrf.ts`             | Protección SSRF             | **Crítico** |
| `src/security/external-content.ts`  | Mitigación de inyección de prompts | **Crítico** |
| `src/agents/sandbox/tool-policy.ts` | Aplicación de políticas de herramientas     | **Crítico** |
| `src/routing/resolve-route.ts`      | Aislamiento de sesiones           | **Medio**   |

### 7.3 Glosario

| Term                 | Definition                                                |
| -------------------- | --------------------------------------------------------- |
| **ATLAS**            | Panorama de amenazas adversarias para sistemas de IA de MITRE       |
| **ClawHub**          | Marketplace de Skills de OpenClaw                              |
| **Gateway**          | Capa de autenticación y enrutamiento de mensajes de OpenClaw       |
| **MCP**              | Model Context Protocol - interfaz de proveedor de herramientas          |
| **Prompt Injection** | Ataque en el que se incrustan instrucciones maliciosas en la entrada |
| **Skill**            | Extensión descargable para agentes de OpenClaw                |
| **SSRF**             | Falsificación de solicitudes del lado del servidor                               |

---

_Este modelo de amenazas es un documento vivo. Informa sobre problemas de seguridad a security@openclaw.ai_
