---
read_when:
    - OpenClaw soll gegen einen lokalen inferrs-Server ausgeführt werden
    - Gemma oder ein anderes Modell wird über inferrs bereitgestellt
    - Die exakten OpenClaw-Kompatibilitäts-Flags für inferrs werden benötigt
summary: OpenClaw über inferrs ausführen (OpenAI-kompatibler lokaler Server)
title: inferrs
x-i18n:
    generated_at: "2026-04-08T02:17:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: d84f660d49a682d0c0878707eebe1bc1e83dd115850687076ea3938b9f9c86c6
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs) kann lokale Modelle hinter einer
OpenAI-kompatiblen `/v1`-API bereitstellen. OpenClaw funktioniert mit `inferrs` über den generischen
Pfad `openai-completions`.

`inferrs` sollte derzeit am besten als benutzerdefiniertes self-hosted
OpenAI-kompatibles Backend behandelt werden, nicht als dediziertes OpenClaw-Provider-Plugin.

## Schnellstart

1. `inferrs` mit einem Modell starten.

Beispiel:

```bash
inferrs serve gg-hf-gg/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. Prüfen, ob der Server erreichbar ist.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. Einen expliziten OpenClaw-Provider-Eintrag hinzufügen und das Standardmodell darauf verweisen lassen.

## Vollständiges Konfigurationsbeispiel

Dieses Beispiel verwendet Gemma 4 auf einem lokalen `inferrs`-Server.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/gg-hf-gg/gemma-4-E2B-it" },
      models: {
        "inferrs/gg-hf-gg/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "gg-hf-gg/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## Warum `requiresStringContent` wichtig ist

Einige Chat-Completions-Routen von `inferrs` akzeptieren nur String-
`messages[].content` und keine strukturierten Content-Part-Arrays.

Wenn OpenClaw-Läufe mit einem Fehler wie diesem fehlschlagen:

```text
messages[1].content: invalid type: sequence, expected a string
```

setze:

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw reduziert dann reine Text-Content-Parts vor dem Senden der
Anfrage auf einfache Strings.

## Vorbehalt zu Gemma und Tool-Schema

Einige aktuelle Kombinationen aus `inferrs` + Gemma akzeptieren kleine direkte
`/v1/chat/completions`-Anfragen, schlagen aber bei vollständigen OpenClaw-Agent-Runtime-
Turns weiterhin fehl.

Wenn das passiert, versuche zuerst Folgendes:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

Dadurch wird die Tool-Schema-Oberfläche von OpenClaw für das Modell deaktiviert und der Prompt-
Druck auf strengeren lokalen Backends kann reduziert werden.

Wenn winzige direkte Anfragen weiterhin funktionieren, normale OpenClaw-Agent-Turns aber in `inferrs`
weiterhin abstürzen, liegt das verbleibende Problem normalerweise eher am Verhalten des Upstream-
Modells/Servers als an der Transportschicht von OpenClaw.

## Manueller Smoke-Test

Nach der Konfiguration beide Ebenen testen:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"gg-hf-gg/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/gg-hf-gg/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

Wenn der erste Befehl funktioniert, der zweite aber fehlschlägt, verwende die Hinweise
zur Fehlerbehebung unten.

## Fehlerbehebung

- `curl /v1/models` schlägt fehl: `inferrs` läuft nicht, ist nicht erreichbar oder
  nicht an den erwarteten Host/Port gebunden.
- `messages[].content ... expected a string`: setze
  `compat.requiresStringContent: true`.
- Direkte kleine `/v1/chat/completions`-Aufrufe funktionieren, aber `openclaw infer model run`
  schlägt fehl: versuche `compat.supportsTools: false`.
- OpenClaw bekommt keine Schemafehler mehr, aber `inferrs` stürzt bei größeren
  Agent-Turns weiterhin ab: behandle dies als Einschränkung von Upstream-`inferrs` oder des Modells und reduziere
  den Prompt-Druck oder wechsle Backend/Modell lokal.

## Proxy-ähnliches Verhalten

`inferrs` wird als proxyähnliches OpenAI-kompatibles `/v1`-Backend behandelt, nicht als
nativer OpenAI-Endpunkt.

- natives request shaping nur für OpenAI gilt hier nicht
- kein `service_tier`, kein Responses-`store`, keine Prompt-Cache-Hinweise und kein
  OpenAI-Reasoning-Compat-Payload-Shaping
- versteckte OpenClaw-Attribution-Header (`originator`, `version`, `User-Agent`)
  werden bei benutzerdefinierten `inferrs`-Base-URLs nicht injiziert

## Siehe auch

- [Lokale Modelle](/de/gateway/local-models)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Modell-Provider](/de/concepts/model-providers)
