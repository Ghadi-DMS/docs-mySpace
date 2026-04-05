---
read_when:
    - Configurando grupos de difusión
    - Depurando respuestas de múltiples agentes en WhatsApp
status: experimental
summary: Difundir un mensaje de WhatsApp a múltiples agentes
title: Grupos de difusión
x-i18n:
    generated_at: "2026-04-05T12:35:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1d117ae65ec3b63c2bd4b3c215d96f32d7eafa0f99a9cd7378e502c15e56ca56
    source_path: channels/broadcast-groups.md
    workflow: 15
---

# Grupos de difusión

**Estado:** Experimental  
**Versión:** Añadido en 2026.1.9

## Descripción general

Los grupos de difusión permiten que varios agentes procesen y respondan al mismo mensaje simultáneamente. Esto te permite crear equipos especializados de agentes que trabajan juntos en un solo grupo o MD de WhatsApp, todo usando un único número de teléfono.

Alcance actual: **solo WhatsApp** (canal web).

Los grupos de difusión se evalúan después de las listas de permitidos del canal y las reglas de activación del grupo. En los grupos de WhatsApp, esto significa que las difusiones ocurren cuando OpenClaw normalmente respondería (por ejemplo, al recibir una mención, según la configuración de tu grupo).

## Casos de uso

### 1. Equipos especializados de agentes

Implementa múltiples agentes con responsabilidades atómicas y centradas:

```
Grupo: "Equipo de desarrollo"
Agentes:
  - CodeReviewer (revisa fragmentos de código)
  - DocumentationBot (genera documentación)
  - SecurityAuditor (comprueba vulnerabilidades)
  - TestGenerator (sugiere casos de prueba)
```

Cada agente procesa el mismo mensaje y aporta su perspectiva especializada.

### 2. Soporte multilingüe

```
Grupo: "Soporte internacional"
Agentes:
  - Agent_EN (responde en inglés)
  - Agent_DE (responde en alemán)
  - Agent_ES (responde en español)
```

### 3. Flujos de trabajo de aseguramiento de calidad

```
Grupo: "Atención al cliente"
Agentes:
  - SupportAgent (proporciona una respuesta)
  - QAAgent (revisa la calidad, solo responde si encuentra problemas)
```

### 4. Automatización de tareas

```
Grupo: "Gestión de proyectos"
Agentes:
  - TaskTracker (actualiza la base de datos de tareas)
  - TimeLogger (registra el tiempo empleado)
  - ReportGenerator (crea resúmenes)
```

## Configuración

### Configuración básica

Añade una sección `broadcast` de nivel superior (junto a `bindings`). Las claves son los ID de pares de WhatsApp:

- chats de grupo: JID del grupo (por ejemplo, `120363403215116621@g.us`)
- MD: número de teléfono en formato E.164 (por ejemplo, `+15551234567`)

```json
{
  "broadcast": {
    "120363403215116621@g.us": ["alfred", "baerbel", "assistant3"]
  }
}
```

**Resultado:** Cuando OpenClaw respondería en este chat, ejecutará los tres agentes.

### Estrategia de procesamiento

Controla cómo los agentes procesan los mensajes:

#### Paralelo (predeterminado)

Todos los agentes procesan simultáneamente:

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

#### Secuencial

Los agentes procesan en orden (uno espera a que termine el anterior):

```json
{
  "broadcast": {
    "strategy": "sequential",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

### Ejemplo completo

```json
{
  "agents": {
    "list": [
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "workspace": "/path/to/code-reviewer",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "security-auditor",
        "name": "Security Auditor",
        "workspace": "/path/to/security-auditor",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "docs-generator",
        "name": "Documentation Generator",
        "workspace": "/path/to/docs-generator",
        "sandbox": { "mode": "all" }
      }
    ]
  },
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["code-reviewer", "security-auditor", "docs-generator"],
    "120363424282127706@g.us": ["support-en", "support-de"],
    "+15555550123": ["assistant", "logger"]
  }
}
```

## Cómo funciona

### Flujo de mensajes

1. **Llega un mensaje entrante** a un grupo de WhatsApp
2. **Comprobación de difusión**: el sistema comprueba si el ID del par está en `broadcast`
3. **Si está en la lista de difusión**:
   - Todos los agentes listados procesan el mensaje
   - Cada agente tiene su propia clave de sesión y contexto aislado
   - Los agentes procesan en paralelo (predeterminado) o secuencialmente
4. **Si no está en la lista de difusión**:
   - Se aplica el enrutamiento normal (primer `binding` coincidente)

Nota: los grupos de difusión no omiten las listas de permitidos del canal ni las reglas de activación del grupo (menciones/comandos/etc.). Solo cambian _qué agentes se ejecutan_ cuando un mensaje es elegible para su procesamiento.

### Aislamiento de sesiones

Cada agente de un grupo de difusión mantiene completamente separados:

- **Claves de sesión** (`agent:alfred:whatsapp:group:120363...` frente a `agent:baerbel:whatsapp:group:120363...`)
- **Historial de conversación** (el agente no ve los mensajes de otros agentes)
- **Espacio de trabajo** (sandboxes separados si están configurados)
- **Acceso a herramientas** (distintas listas de permitidos/denegados)
- **Memoria/contexto** (`IDENTITY.md`, `SOUL.md`, etc. por separado)
- **Búfer de contexto del grupo** (mensajes recientes del grupo usados como contexto), que se comparte por par, por lo que todos los agentes de difusión ven el mismo contexto cuando se activan

Esto permite que cada agente tenga:

- Personalidades diferentes
- Acceso a herramientas diferente (por ejemplo, solo lectura frente a lectura-escritura)
- Modelos diferentes (por ejemplo, opus frente a sonnet)
- Skills diferentes instaladas

### Ejemplo: sesiones aisladas

En el grupo `120363403215116621@g.us` con agentes `["alfred", "baerbel"]`:

**Contexto de Alfred:**

```
Session: agent:alfred:whatsapp:group:120363403215116621@g.us
History: [user message, alfred's previous responses]
Workspace: /Users/user/openclaw-alfred/
Tools: read, write, exec
```

**Contexto de Bärbel:**

```
Session: agent:baerbel:whatsapp:group:120363403215116621@g.us
History: [user message, baerbel's previous responses]
Workspace: /Users/user/openclaw-baerbel/
Tools: read only
```

## Buenas prácticas

### 1. Mantén a los agentes centrados

Diseña cada agente con una responsabilidad única y clara:

```json
{
  "broadcast": {
    "DEV_GROUP": ["formatter", "linter", "tester"]
  }
}
```

✅ **Bueno:** Cada agente tiene un trabajo  
❌ **Malo:** Un agente genérico "dev-helper"

### 2. Usa nombres descriptivos

Haz que quede claro qué hace cada agente:

```json
{
  "agents": {
    "security-scanner": { "name": "Security Scanner" },
    "code-formatter": { "name": "Code Formatter" },
    "test-generator": { "name": "Test Generator" }
  }
}
```

### 3. Configura distinto acceso a herramientas

Da a los agentes solo las herramientas que necesitan:

```json
{
  "agents": {
    "reviewer": {
      "tools": { "allow": ["read", "exec"] } // Solo lectura
    },
    "fixer": {
      "tools": { "allow": ["read", "write", "edit", "exec"] } // Lectura-escritura
    }
  }
}
```

### 4. Supervisa el rendimiento

Con muchos agentes, considera lo siguiente:

- Usar `"strategy": "parallel"` (predeterminado) para mayor velocidad
- Limitar los grupos de difusión a 5-10 agentes
- Usar modelos más rápidos para agentes más simples

### 5. Gestiona los fallos correctamente

Los agentes fallan de forma independiente. El error de un agente no bloquea a los demás:

```
Mensaje → [Agente A ✓, Agente B ✗ error, Agente C ✓]
Resultado: los agentes A y C responden, el agente B registra un error
```

## Compatibilidad

### Proveedores

Los grupos de difusión actualmente funcionan con:

- ✅ WhatsApp (implementado)
- 🚧 Telegram (planificado)
- 🚧 Discord (planificado)
- 🚧 Slack (planificado)

### Enrutamiento

Los grupos de difusión funcionan junto con el enrutamiento existente:

```json
{
  "bindings": [
    {
      "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "GROUP_A" } },
      "agentId": "alfred"
    }
  ],
  "broadcast": {
    "GROUP_B": ["agent1", "agent2"]
  }
}
```

- `GROUP_A`: solo responde alfred (enrutamiento normal)
- `GROUP_B`: responden agent1 Y agent2 (difusión)

**Precedencia:** `broadcast` tiene prioridad sobre `bindings`.

## Solución de problemas

### Los agentes no responden

**Comprueba lo siguiente:**

1. Los ID de agentes existen en `agents.list`
2. El formato del ID del par es correcto (por ejemplo, `120363403215116621@g.us`)
3. Los agentes no están en listas de denegación

**Depuración:**

```bash
tail -f ~/.openclaw/logs/gateway.log | grep broadcast
```

### Solo responde un agente

**Causa:** Es posible que el ID del par esté en `bindings`, pero no en `broadcast`.

**Solución:** Añádelo a la configuración de difusión o elimínalo de `bindings`.

### Problemas de rendimiento

**Si va lento con muchos agentes:**

- Reduce la cantidad de agentes por grupo
- Usa modelos más ligeros (`sonnet` en lugar de `opus`)
- Comprueba el tiempo de inicio del sandbox

## Ejemplos

### Ejemplo 1: Equipo de revisión de código

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": [
      "code-formatter",
      "security-scanner",
      "test-coverage",
      "docs-checker"
    ]
  },
  "agents": {
    "list": [
      {
        "id": "code-formatter",
        "workspace": "~/agents/formatter",
        "tools": { "allow": ["read", "write"] }
      },
      {
        "id": "security-scanner",
        "workspace": "~/agents/security",
        "tools": { "allow": ["read", "exec"] }
      },
      {
        "id": "test-coverage",
        "workspace": "~/agents/testing",
        "tools": { "allow": ["read", "exec"] }
      },
      { "id": "docs-checker", "workspace": "~/agents/docs", "tools": { "allow": ["read"] } }
    ]
  }
}
```

**El usuario envía:** Fragmento de código  
**Respuestas:**

- code-formatter: "Corregí la indentación y añadí sugerencias de tipo"
- security-scanner: "⚠️ Vulnerabilidad de inyección SQL en la línea 12"
- test-coverage: "La cobertura es del 45 %, faltan pruebas para casos de error"
- docs-checker: "Falta el docstring para la función `process_data`"

### Ejemplo 2: Soporte multilingüe

```json
{
  "broadcast": {
    "strategy": "sequential",
    "+15555550123": ["detect-language", "translator-en", "translator-de"]
  },
  "agents": {
    "list": [
      { "id": "detect-language", "workspace": "~/agents/lang-detect" },
      { "id": "translator-en", "workspace": "~/agents/translate-en" },
      { "id": "translator-de", "workspace": "~/agents/translate-de" }
    ]
  }
}
```

## Referencia de API

### Esquema de configuración

```typescript
interface OpenClawConfig {
  broadcast?: {
    strategy?: "parallel" | "sequential";
    [peerId: string]: string[];
  };
}
```

### Campos

- `strategy` (opcional): cómo procesar agentes
  - `"parallel"` (predeterminado): todos los agentes procesan simultáneamente
  - `"sequential"`: los agentes procesan en el orden del array
- `[peerId]`: JID de grupo de WhatsApp, número E.164 u otro ID de par
  - Valor: array de ID de agentes que deben procesar mensajes

## Limitaciones

1. **Máximo de agentes:** No hay un límite estricto, pero 10 o más agentes pueden ir lentos
2. **Contexto compartido:** Los agentes no ven las respuestas de los demás (por diseño)
3. **Orden de mensajes:** Las respuestas en paralelo pueden llegar en cualquier orden
4. **Límites de frecuencia:** Todos los agentes cuentan para los límites de frecuencia de WhatsApp

## Mejoras futuras

Funciones planificadas:

- [ ] Modo de contexto compartido (los agentes ven las respuestas de los demás)
- [ ] Coordinación entre agentes (los agentes pueden enviarse señales entre sí)
- [ ] Selección dinámica de agentes (elegir agentes según el contenido del mensaje)
- [ ] Prioridades de agentes (algunos agentes responden antes que otros)

## Consulta también

- [Configuración de múltiples agentes](/tools/multi-agent-sandbox-tools)
- [Configuración de enrutamiento](/channels/channel-routing)
- [Gestión de sesiones](/concepts/session)
