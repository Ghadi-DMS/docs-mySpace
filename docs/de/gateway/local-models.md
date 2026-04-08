---
read_when:
    - Sie möchten Modelle von Ihrer eigenen GPU-Maschine bereitstellen
    - Sie binden LM Studio oder einen OpenAI-kompatiblen Proxy an
    - Sie benötigen die sicherste Anleitung für lokale Modelle
summary: OpenClaw mit lokalen LLMs ausführen (LM Studio, vLLM, LiteLLM, benutzerdefinierte OpenAI-Endpunkte)
title: Lokale Modelle
x-i18n:
    generated_at: "2026-04-08T02:14:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: d619d72b0e06914ebacb7e9f38b746caf1b9ce8908c9c6638c3acdddbaa025e8
    source_path: gateway/local-models.md
    workflow: 15
---

# Lokale Modelle

Lokal ist machbar, aber OpenClaw erwartet großen Kontext und starke Abwehr gegen Prompt Injection. Kleine Karten kürzen den Kontext und schwächen die Sicherheit. Setzen Sie hohe Maßstäbe an: **≥2 voll ausgestattete Mac Studios oder ein gleichwertiges GPU-System (~30.000 $+)**. Eine einzelne **24-GB**-GPU funktioniert nur für leichtere Prompts mit höherer Latenz. Verwenden Sie die **größte / vollwertige Modellvariante, die Sie ausführen können**; stark quantisierte oder „kleine“ Checkpoints erhöhen das Risiko von Prompt Injection (siehe [Sicherheit](/de/gateway/security)).

Wenn Sie die lokale Einrichtung mit der geringsten Reibung möchten, beginnen Sie mit [Ollama](/de/providers/ollama) und `openclaw onboard`. Diese Seite ist der meinungsstarke Leitfaden für höherwertige lokale Stacks und benutzerdefinierte OpenAI-kompatible lokale Server.

## Empfohlen: LM Studio + großes lokales Modell (Responses API)

Der derzeit beste lokale Stack. Laden Sie ein großes Modell in LM Studio (zum Beispiel einen vollwertigen Qwen-, DeepSeek- oder Llama-Build), aktivieren Sie den lokalen Server (standardmäßig `http://127.0.0.1:1234`) und verwenden Sie Responses API, um Reasoning vom endgültigen Text getrennt zu halten.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Checkliste zur Einrichtung**

- Installieren Sie LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- Laden Sie in LM Studio den **größten verfügbaren Modell-Build** herunter (vermeiden Sie „small“-/stark quantisierte Varianten), starten Sie den Server und bestätigen Sie, dass `http://127.0.0.1:1234/v1/models` ihn auflistet.
- Ersetzen Sie `my-local-model` durch die tatsächliche Modell-ID, die in LM Studio angezeigt wird.
- Halten Sie das Modell geladen; Kaltladen erhöht die Startlatenz.
- Passen Sie `contextWindow`/`maxTokens` an, wenn sich Ihr LM Studio-Build unterscheidet.
- Für WhatsApp sollten Sie bei Responses API bleiben, damit nur der endgültige Text gesendet wird.

Lassen Sie gehostete Modelle auch beim lokalen Betrieb konfiguriert; verwenden Sie `models.mode: "merge"`, damit Fallbacks verfügbar bleiben.

### Hybride Konfiguration: gehostet als primär, lokal als Fallback

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Lokal zuerst mit gehostetem Sicherheitsnetz

Tauschen Sie die Reihenfolge von primärem Modell und Fallbacks; behalten Sie denselben Provider-Block und `models.mode: "merge"` bei, damit Sie auf Sonnet oder Opus zurückfallen können, wenn die lokale Maschine ausfällt.

### Regionales Hosting / Datenrouting

- Gehostete MiniMax-/Kimi-/GLM-Varianten gibt es auch auf OpenRouter mit regional gebundenen Endpunkten (z. B. in den USA gehostet). Wählen Sie dort die regionale Variante, um den Datenverkehr in Ihrer gewünschten Gerichtsbarkeit zu halten und dennoch `models.mode: "merge"` für Anthropic-/OpenAI-Fallbacks zu verwenden.
- Nur lokal bleibt der stärkste Datenschutzpfad; gehostetes regionales Routing ist der Mittelweg, wenn Sie Provider-Funktionen benötigen, aber Kontrolle über den Datenfluss wünschen.

## Andere OpenAI-kompatible lokale Proxys

vLLM, LiteLLM, OAI-proxy oder benutzerdefinierte Gateways funktionieren, wenn sie einen OpenAI-artigen `/v1`-Endpunkt bereitstellen. Ersetzen Sie den obigen Provider-Block durch Ihren Endpunkt und Ihre Modell-ID:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Behalten Sie `models.mode: "merge"` bei, damit gehostete Modelle als Fallbacks verfügbar bleiben.

Verhaltenshinweis für lokale/proxied `/v1`-Backends:

- OpenClaw behandelt diese als Proxy-artige OpenAI-kompatible Routen, nicht als native
  OpenAI-Endpunkte
- rein native OpenAI-Anpassungen für Requests gelten hier nicht: kein
  `service_tier`, kein Responses-`store`, keine OpenAI-Reasoning-Kompatibilitäts-
  Anpassung der Payload und keine Prompt-Cache-Hinweise
- versteckte OpenClaw-Attribution-Header (`originator`, `version`, `User-Agent`)
  werden auf diesen benutzerdefinierten Proxy-URLs nicht eingefügt

Kompatibilitätshinweise für strengere OpenAI-kompatible Backends:

- Einige Server akzeptieren bei Chat Completions nur String-`messages[].content`, nicht
  strukturierte Content-Part-Arrays. Setzen Sie
  `models.providers.<provider>.models[].compat.requiresStringContent: true` für
  diese Endpunkte.
- Einige kleinere oder strengere lokale Backends sind mit der vollständigen
  Prompt-Form der OpenClaw-Agent-Laufzeit instabil, insbesondere wenn Tool-Schemas enthalten sind. Wenn das
  Backend für kleine direkte `/v1/chat/completions`-Aufrufe funktioniert, aber bei normalen
  OpenClaw-Agent-Turns fehlschlägt, versuchen Sie zuerst
  `models.providers.<provider>.models[].compat.supportsTools: false`.
- Wenn das Backend weiterhin nur bei größeren OpenClaw-Läufen fehlschlägt, liegt das verbleibende Problem
  in der Regel an der Kapazität des Upstream-Modells/Servers oder an einem Backend-Fehler, nicht an der
  Transportebene von OpenClaw.

## Fehlerbehebung

- Kann das Gateway den Proxy erreichen? `curl http://127.0.0.1:1234/v1/models`.
- LM Studio-Modell entladen? Laden Sie es erneut; Kaltstart ist eine häufige Ursache für „Hängenbleiben“.
- Kontextfehler? Verringern Sie `contextWindow` oder erhöhen Sie das Limit Ihres Servers.
- Der OpenAI-kompatible Server gibt `messages[].content ... expected a string` zurück?
  Fügen Sie `compat.requiresStringContent: true` bei diesem Modelleintrag hinzu.
- Kleine direkte `/v1/chat/completions`-Aufrufe funktionieren, aber `openclaw infer model run`
  schlägt bei Gemma oder einem anderen lokalen Modell fehl? Deaktivieren Sie zuerst Tool-Schemas mit
  `compat.supportsTools: false` und testen Sie dann erneut. Wenn der Server weiterhin nur
  bei größeren OpenClaw-Prompts abstürzt, behandeln Sie dies als Einschränkung des Upstream-Servers/-Modells.
- Sicherheit: Lokale Modelle überspringen providerseitige Filter; halten Sie Agents eng gefasst und die Kompaktierung aktiviert, um den Wirkungsbereich von Prompt Injection zu begrenzen.
