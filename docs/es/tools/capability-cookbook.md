---
read_when:
    - Añadir una nueva capacidad central y una superficie de registro de plugins
    - Decidir si el código debe pertenecer al núcleo, a un plugin del proveedor o a un plugin de funcionalidad
    - Conectar un nuevo helper de tiempo de ejecución para canales o herramientas
sidebarTitle: Adding Capabilities
summary: Guía para contribuidores para añadir una nueva capacidad compartida al sistema de plugins de OpenClaw
title: Añadir capacidades (guía para contribuidores)
x-i18n:
    generated_at: "2026-04-05T12:55:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 29604d88e6df5205b835d71f3078b6223c58b6294135c3e201756c1bcac33ea3
    source_path: tools/capability-cookbook.md
    workflow: 15
---

# Añadir capacidades

<Info>
  Esta es una **guía para contribuidores** para desarrolladores del núcleo de OpenClaw. Si estás
  creando un plugin externo, consulta [Building Plugins](/es/plugins/building-plugins)
  en su lugar.
</Info>

Usa esto cuando OpenClaw necesite un nuevo dominio, como generación de imágenes, generación de video
o alguna futura área de funcionalidad respaldada por proveedores.

La regla:

- plugin = límite de propiedad
- capability = contrato central compartido

Eso significa que no debes empezar conectando un proveedor directamente a un canal o una
herramienta. Empieza definiendo la capacidad.

## Cuándo crear una capacidad

Crea una nueva capacidad cuando todo lo siguiente sea cierto:

1. más de un proveedor podría implementarla razonablemente
2. los canales, las herramientas o los plugins de funcionalidad deberían consumirla sin preocuparse por
   el proveedor
3. el núcleo necesita ser propietario del comportamiento de respaldo, las políticas, la configuración o la entrega

Si el trabajo es solo de proveedor y todavía no existe un contrato compartido, detente y define
primero el contrato.

## La secuencia estándar

1. Define el contrato central tipado.
2. Añade el registro de plugins para ese contrato.
3. Añade un helper compartido de tiempo de ejecución.
4. Conecta un plugin real de proveedor como prueba.
5. Mueve los consumidores de funcionalidad/canal al helper de tiempo de ejecución.
6. Añade pruebas de contrato.
7. Documenta la configuración orientada al operador y el modelo de propiedad.

## Qué va dónde

Núcleo:

- tipos de solicitud/respuesta
- registro de proveedores + resolución
- comportamiento de respaldo
- esquema de configuración más metadatos de documentación `title` / `description` propagados en nodos de objetos anidados, comodines, elementos de arrays y composición
- superficie del helper de tiempo de ejecución

Plugin del proveedor:

- llamadas a la API del proveedor
- gestión de autenticación del proveedor
- normalización de solicitudes específica del proveedor
- registro de la implementación de la capacidad

Plugin de funcionalidad/canal:

- llama a `api.runtime.*` o al helper coincidente `plugin-sdk/*-runtime`
- nunca llama directamente a una implementación de proveedor

## Lista de archivos

Para una nueva capacidad, es esperable tocar estas áreas:

- `src/<capability>/types.ts`
- `src/<capability>/...registry/runtime.ts`
- `src/plugins/types.ts`
- `src/plugins/registry.ts`
- `src/plugins/captured-registration.ts`
- `src/plugins/contracts/registry.ts`
- `src/plugins/runtime/types-core.ts`
- `src/plugins/runtime/index.ts`
- `src/plugin-sdk/<capability>.ts`
- `src/plugin-sdk/<capability>-runtime.ts`
- uno o más paquetes de plugins integrados
- config/docs/tests

## Ejemplo: generación de imágenes

La generación de imágenes sigue la forma estándar:

1. el núcleo define `ImageGenerationProvider`
2. el núcleo expone `registerImageGenerationProvider(...)`
3. el núcleo expone `runtime.imageGeneration.generate(...)`
4. los plugins `openai`, `google`, `fal` y `minimax` registran implementaciones respaldadas por proveedores
5. los futuros proveedores pueden registrar el mismo contrato sin cambiar canales/herramientas

La clave de configuración es independiente del enrutamiento de análisis de visión:

- `agents.defaults.imageModel` = analizar imágenes
- `agents.defaults.imageGenerationModel` = generar imágenes

Mantén ambos separados para que el respaldo y las políticas sigan siendo explícitos.

## Lista de comprobación de revisión

Antes de publicar una nueva capacidad, verifica:

- ningún canal/herramienta importa código de proveedor directamente
- el helper de tiempo de ejecución es la ruta compartida
- al menos una prueba de contrato valida la propiedad integrada
- la documentación de configuración nombra la nueva clave de modelo/configuración
- la documentación de plugins explica el límite de propiedad

Si una PR omite la capa de capacidad y codifica de forma rígida el comportamiento del proveedor en un
canal/herramienta, devuélvela y define primero el contrato.
