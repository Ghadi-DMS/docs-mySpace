---
read_when:
    - Quieres usar Groq con OpenClaw
    - Necesitas la variable de entorno de API key o la elección de autenticación en CLI
summary: Configuración de Groq (autenticación + selección de modelo)
title: Groq
x-i18n:
    generated_at: "2026-04-05T12:51:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7e27532cafcdaf1ac336fa310e08e4e3245d2d0eb0e94e0bcf42c532c6a9a80b
    source_path: providers/groq.md
    workflow: 15
---

# Groq

[Groq](https://groq.com) proporciona inferencia ultrarrápida en modelos de código abierto
(Llama, Gemma, Mistral y más) usando hardware LPU personalizado. OpenClaw se conecta
a Groq mediante su API compatible con OpenAI.

- Proveedor: `groq`
- Autenticación: `GROQ_API_KEY`
- API: compatible con OpenAI

## Inicio rápido

1. Obtén una API key en [console.groq.com/keys](https://console.groq.com/keys).

2. Establece la API key:

```bash
export GROQ_API_KEY="gsk_..."
```

3. Establece un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## Ejemplo de archivo de configuración

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## Transcripción de audio

Groq también ofrece transcripción de audio rápida basada en Whisper. Cuando se configura como proveedor de comprensión multimeda,
OpenClaw usa el modelo `whisper-large-v3-turbo` de Groq para transcribir mensajes de voz mediante la superficie compartida `tools.media.audio`.

```json5
{
  tools: {
    media: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

## Nota sobre el entorno

Si la Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `GROQ_API_KEY` esté
disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).

## Notas sobre audio

- Ruta de configuración compartida: `tools.media.audio`
- URL base de audio predeterminada de Groq: `https://api.groq.com/openai/v1`
- Modelo de audio predeterminado de Groq: `whisper-large-v3-turbo`
- La transcripción de audio de Groq usa la ruta compatible con OpenAI `/audio/transcriptions`

## Modelos disponibles

El catálogo de modelos de Groq cambia con frecuencia. Ejecuta `openclaw models list | grep groq`
para ver los modelos disponibles actualmente, o consulta
[console.groq.com/docs/models](https://console.groq.com/docs/models).

Opciones populares:

- **Llama 3.3 70B Versatile** - propósito general, contexto amplio
- **Llama 3.1 8B Instant** - rápido, ligero
- **Gemma 2 9B** - compacto, eficiente
- **Mixtral 8x7B** - arquitectura MoE, razonamiento sólido

## Enlaces

- [Groq Console](https://console.groq.com)
- [Documentación de la API](https://console.groq.com/docs)
- [Lista de modelos](https://console.groq.com/docs/models)
- [Precios](https://groq.com/pricing)
