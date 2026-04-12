---
read_when:
    - Du möchtest OpenAI-Modelle in OpenClaw verwenden
    - Du möchtest statt API-Schlüsseln die Authentifizierung per Codex-Abonnement verwenden
    - Du benötigst ein strengeres Ausführungsverhalten des GPT-5-Agenten
summary: Verwende OpenAI über API-Schlüssel oder ein Codex-Abonnement in OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-12T00:18:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7aa06fba9ac901e663685a6b26443a2f6aeb6ec3589d939522dc87cbb43497b4
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI bietet Entwickler-APIs für GPT-Modelle. Codex unterstützt die **ChatGPT-Anmeldung** für den Zugriff über ein Abonnement oder die Anmeldung per **API-Schlüssel** für nutzungsbasierten Zugriff. Codex cloud erfordert eine ChatGPT-Anmeldung.
OpenAI unterstützt ausdrücklich die Nutzung von Abonnement-OAuth in externen Tools/Workflows wie OpenClaw.

## Standard-Interaktionsstil

OpenClaw kann für `openai/*`- und `openai-codex/*`-Ausführungen ein kleines OpenAI-spezifisches Prompt-Overlay hinzufügen. Standardmäßig hält das Overlay den Assistenten warm, kooperativ, prägnant, direkt und etwas emotional ausdrucksstärker, ohne den grundlegenden OpenClaw-System-Prompt zu ersetzen. Das freundliche Overlay erlaubt außerdem gelegentlich ein Emoji, wenn es natürlich passt, während die Ausgabe insgesamt prägnant bleibt.

Konfigurationsschlüssel:

`plugins.entries.openai.config.personality`

Zulässige Werte:

- `"friendly"`: Standard; aktiviert das OpenAI-spezifische Overlay.
- `"on"`: Alias für `"friendly"`.
- `"off"`: deaktiviert das Overlay und verwendet nur den grundlegenden OpenClaw-Prompt.

Geltungsbereich:

- Gilt für `openai/*`-Modelle.
- Gilt für `openai-codex/*`-Modelle.
- Beeinflusst keine anderen Anbieter.

Dieses Verhalten ist standardmäßig aktiviert. Behalte `"friendly"` explizit bei, wenn du möchtest, dass dies zukünftige lokale Konfigurationsänderungen übersteht:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### Das OpenAI-Prompt-Overlay deaktivieren

Wenn du den unveränderten grundlegenden OpenClaw-Prompt möchtest, setze das Overlay auf `"off"`:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Du kannst es auch direkt mit der Konfigurations-CLI setzen:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw normalisiert diese Einstellung zur Laufzeit ohne Berücksichtigung der Groß-/Kleinschreibung, daher deaktivieren Werte wie `"Off"` das freundliche Overlay ebenfalls.

## Option A: OpenAI API-Schlüssel (OpenAI Platform)

**Am besten für:** direkten API-Zugriff und nutzungsbasierte Abrechnung.
Hole dir deinen API-Schlüssel aus dem OpenAI-Dashboard.

Routenübersicht:

- `openai/gpt-5.4` = direkte OpenAI Platform API-Route
- Erfordert `OPENAI_API_KEY` (oder eine entsprechende OpenAI-Anbieterkonfiguration)
- In OpenClaw wird die ChatGPT/Codex-Anmeldung über `openai-codex/*` geleitet, nicht über `openai/*`

### CLI-Einrichtung

```bash
openclaw onboard --auth-choice openai-api-key
# oder nicht interaktiv
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Konfigurationsbeispiel

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

In der aktuellen API-Modelldokumentation von OpenAI werden `gpt-5.4` und `gpt-5.4-pro` für die direkte OpenAI-API-Nutzung aufgeführt. OpenClaw leitet beide über den `openai/*`-Responses-Pfad weiter.
OpenClaw unterdrückt absichtlich den veralteten Eintrag `openai/gpt-5.3-codex-spark`, da direkte OpenAI-API-Aufrufe ihn im Live-Verkehr ablehnen.

OpenClaw stellt `openai/gpt-5.3-codex-spark` **nicht** auf dem direkten OpenAI-API-Pfad bereit. `pi-ai` liefert weiterhin einen integrierten Eintrag für dieses Modell aus, aber Live-Anfragen an die OpenAI-API lehnen es derzeit ab. Spark wird in OpenClaw als ausschließlich Codex behandelt.

## Bildgenerierung

Das gebündelte `openai`-Plugin registriert außerdem die Bildgenerierung über das gemeinsame Tool `image_generate`.

- Standard-Bildmodell: `openai/gpt-image-1`
- Generierung: bis zu 4 Bilder pro Anfrage
- Bearbeitungsmodus: aktiviert, bis zu 5 Referenzbilder
- Unterstützt `size`
- Aktuelle OpenAI-spezifische Einschränkung: OpenClaw leitet `aspectRatio`- oder `resolution`-Überschreibungen derzeit nicht an die OpenAI Images API weiter

Um OpenAI als Standard-Bildanbieter zu verwenden:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Siehe [Bildgenerierung](/de/tools/image-generation) für die gemeinsamen Tool-Parameter, die Anbieterauswahl und das Failover-Verhalten.

## Videogenerierung

Das gebündelte `openai`-Plugin registriert außerdem die Videogenerierung über das gemeinsame Tool `video_generate`.

- Standard-Videomodell: `openai/sora-2`
- Modi: Text-zu-Video, Bild-zu-Video und Abläufe mit einer einzelnen Videoreferenz bzw. Bearbeitung
- Aktuelle Limits: 1 Bild- oder 1 Videoreferenzeingabe
- Aktuelle OpenAI-spezifische Einschränkung: OpenClaw leitet derzeit nur `size`-Überschreibungen für die native OpenAI-Videogenerierung weiter. Nicht unterstützte optionale Überschreibungen wie `aspectRatio`, `resolution`, `audio` und `watermark` werden ignoriert und als Tool-Warnung zurückgemeldet.

Um OpenAI als Standard-Videoanbieter zu verwenden:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Siehe [Videogenerierung](/de/tools/video-generation) für die gemeinsamen Tool-Parameter, die Anbieterauswahl und das Failover-Verhalten.

## Option B: OpenAI Code (Codex)-Abonnement

**Am besten für:** die Nutzung des ChatGPT/Codex-Abonnementzugangs statt eines API-Schlüssels.
Codex cloud erfordert eine ChatGPT-Anmeldung, während die Codex CLI die Anmeldung per ChatGPT oder API-Schlüssel unterstützt.

Routenübersicht:

- `openai-codex/gpt-5.4` = ChatGPT/Codex-OAuth-Route
- Verwendet die ChatGPT/Codex-Anmeldung, keinen direkten OpenAI Platform API-Schlüssel
- Anbieterseitige Limits für `openai-codex/*` können sich von der Erfahrung in der ChatGPT-Web-/App unterscheiden

### CLI-Einrichtung (Codex OAuth)

```bash
# Codex OAuth im Assistenten ausführen
openclaw onboard --auth-choice openai-codex

# Oder OAuth direkt ausführen
openclaw models auth login --provider openai-codex
```

### Konfigurationsbeispiel (Codex-Abonnement)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

In der aktuellen Codex-Dokumentation von OpenAI wird `gpt-5.4` als aktuelles Codex-Modell aufgeführt. OpenClaw ordnet dies `openai-codex/gpt-5.4` für die Nutzung mit ChatGPT/Codex-OAuth zu.

Diese Route ist absichtlich getrennt von `openai/gpt-5.4`. Wenn du den direkten OpenAI Platform API-Pfad möchtest, verwende `openai/*` mit einem API-Schlüssel. Wenn du die ChatGPT/Codex-Anmeldung möchtest, verwende `openai-codex/*`.

Wenn das Onboarding eine bestehende Codex CLI-Anmeldung wiederverwendet, bleiben diese Anmeldedaten von der Codex CLI verwaltet. Bei Ablauf liest OpenClaw zuerst erneut die externe Codex-Quelle und schreibt die aktualisierten Anmeldedaten, wenn der Anbieter sie aktualisieren kann, zurück in den Codex-Speicher, statt die Verwaltung in einer separaten, nur von OpenClaw verwendeten Kopie zu übernehmen.

Wenn dein Codex-Konto Anspruch auf Codex Spark hat, unterstützt OpenClaw außerdem:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw behandelt Codex Spark als ausschließlich Codex. Es stellt keinen direkten API-Schlüssel-Pfad `openai/gpt-5.3-codex-spark` bereit.

OpenClaw behält außerdem `openai-codex/gpt-5.3-codex-spark` bei, wenn `pi-ai` es erkennt. Betrachte es als anspruchsabhängig und experimentell: Codex Spark ist getrennt von GPT-5.4 `/fast`, und die Verfügbarkeit hängt vom angemeldeten Codex-/ChatGPT-Konto ab.

### Codex-Kontextfensterbegrenzung

OpenClaw behandelt die Codex-Modellmetadaten und die Laufzeit-Kontextbegrenzung als getrennte Werte.

Für `openai-codex/gpt-5.4`:

- natives `contextWindow`: `1050000`
- Standardbegrenzung für `contextTokens` zur Laufzeit: `272000`

So bleiben die Modellmetadaten wahrheitsgetreu, während das kleinere Standard-Laufzeitfenster erhalten bleibt, das in der Praxis bessere Latenz- und Qualitätsmerkmale aufweist.

Wenn du eine andere effektive Begrenzung möchtest, setze `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Verwende `contextWindow` nur, wenn du native Modellmetadaten deklarierst oder überschreibst. Verwende `contextTokens`, wenn du das Laufzeit-Kontextbudget begrenzen möchtest.

### Standard für den Transport

OpenClaw verwendet `pi-ai` für Modell-Streaming. Für sowohl `openai/*` als auch `openai-codex/*` ist der Standardtransport `"auto"` (zuerst WebSocket, dann SSE-Fallback).

Im Modus `"auto"` versucht OpenClaw außerdem bei einem frühen, wiederholbaren WebSocket-Fehler einen erneuten Versuch, bevor auf SSE zurückgegriffen wird. Der erzwungene Modus `"websocket"` gibt Transportfehler weiterhin direkt aus, statt sie hinter einem Fallback zu verbergen.

Nach einem Verbindungsfehler oder einem frühen WebSocket-Fehler in einem Turn markiert OpenClaw im Modus `"auto"` den WebSocket-Pfad dieser Sitzung für etwa 60 Sekunden als beeinträchtigt und sendet nachfolgende Turns während der Abkühlphase über SSE, statt zwischen den Transporten hin- und herzuschwanken.

Für native Endpunkte der OpenAI-Familie (`openai/*`, `openai-codex/*` und Azure OpenAI Responses) hängt OpenClaw außerdem stabilen Sitzungs- und Turn-Identitätsstatus an Anfragen an, damit Wiederholungen, Neuverbindungen und SSE-Fallbacks auf dieselbe Konversationsidentität abgestimmt bleiben. Auf nativen Routen der OpenAI-Familie umfasst dies stabile Identitäts-Header für Sitzungs-/Turn-Anfragen sowie passende Transportmetadaten.

OpenClaw normalisiert außerdem OpenAI-Nutzungszähler transportübergreifend, bevor sie Oberflächen für Sitzung/Status erreichen. Nativer OpenAI/Codex-Responses-Verkehr kann die Nutzung entweder als `input_tokens` / `output_tokens` oder als `prompt_tokens` / `completion_tokens` melden; OpenClaw behandelt diese für `/status`, `/usage` und Sitzungsprotokolle als dieselben Eingabe- und Ausgabezähler. Wenn nativer WebSocket-Verkehr `total_tokens` auslässt (oder `0` meldet), greift OpenClaw auf die normalisierte Summe aus Eingabe und Ausgabe zurück, damit die Anzeige in Sitzung/Status ausgefüllt bleibt.

Du kannst `agents.defaults.models.<provider/model>.params.transport` setzen:

- `"sse"`: SSE erzwingen
- `"websocket"`: WebSocket erzwingen
- `"auto"`: WebSocket versuchen, dann auf SSE zurückfallen

Für `openai/*` (Responses API) aktiviert OpenClaw standardmäßig außerdem das WebSocket-Warm-up (`openaiWsWarmup: true`), wenn WebSocket-Transport verwendet wird.

Zugehörige OpenAI-Dokumente:

- [Realtime API mit WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming-API-Antworten (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### OpenAI-WebSocket-Warm-up

Die OpenAI-Dokumentation beschreibt Warm-up als optional. OpenClaw aktiviert es standardmäßig für `openai/*`, um die Latenz beim ersten Turn zu verringern, wenn WebSocket-Transport verwendet wird.

### Warm-up deaktivieren

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Warm-up explizit aktivieren

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### OpenAI- und Codex-Prioritätsverarbeitung

Die API von OpenAI stellt Prioritätsverarbeitung über `service_tier=priority` bereit. In OpenClaw setze `agents.defaults.models["<provider>/<model>"].params.serviceTier`, um dieses Feld an native OpenAI/Codex-Responses-Endpunkte weiterzugeben.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Unterstützte Werte sind `auto`, `default`, `flex` und `priority`.

OpenClaw leitet `params.serviceTier` sowohl an direkte `openai/*`-Responses-Anfragen als auch an `openai-codex/*`-Codex-Responses-Anfragen weiter, wenn diese Modelle auf die nativen OpenAI/Codex-Endpunkte zeigen.

Wichtiges Verhalten:

- direktes `openai/*` muss auf `api.openai.com` zielen
- `openai-codex/*` muss auf `chatgpt.com/backend-api` zielen
- wenn du einen der beiden Anbieter über eine andere Basis-URL oder einen Proxy leitest, lässt OpenClaw `service_tier` unverändert

### OpenAI-Schnellmodus

OpenClaw stellt einen gemeinsamen Schnellmodus-Schalter für sowohl `openai/*`- als auch `openai-codex/*`-Sitzungen bereit:

- Chat/UI: `/fast status|on|off`
- Konfiguration: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Wenn der Schnellmodus aktiviert ist, ordnet OpenClaw ihn der OpenAI-Prioritätsverarbeitung zu:

- direkte `openai/*`-Responses-Aufrufe an `api.openai.com` senden `service_tier = "priority"`
- `openai-codex/*`-Responses-Aufrufe an `chatgpt.com/backend-api` senden ebenfalls `service_tier = "priority"`
- vorhandene `service_tier`-Werte in der Nutzlast bleiben erhalten
- der Schnellmodus überschreibt weder `reasoning` noch `text.verbosity`

Für GPT 5.4 ist die gängigste Einrichtung konkret:

- sende `/fast on` in einer Sitzung mit `openai/gpt-5.4` oder `openai-codex/gpt-5.4`
- oder setze `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- wenn du außerdem Codex OAuth verwendest, setze auch `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true`

Beispiel:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Sitzungsüberschreibungen haben Vorrang vor der Konfiguration. Wenn du die Sitzungsüberschreibung in der Sitzungs-UI löschst, kehrt die Sitzung zum konfigurierten Standard zurück.

### Native OpenAI- gegenüber OpenAI-kompatiblen Routen

OpenClaw behandelt direkte OpenAI-, Codex- und Azure OpenAI-Endpunkte anders als generische OpenAI-kompatible `/v1`-Proxys:

- native `openai/*`-, `openai-codex/*`- und Azure OpenAI-Routen behalten `reasoning: { effort: "none" }` unverändert bei, wenn du Reasoning explizit deaktivierst
- native Routen der OpenAI-Familie setzen Tool-Schemas standardmäßig in den Strict-Modus
- verborgene OpenClaw-Attributions-Header (`originator`, `version` und `User-Agent`) werden nur an verifizierten nativen OpenAI-Hosts (`api.openai.com`) und nativen Codex-Hosts (`chatgpt.com/backend-api`) angehängt
- native OpenAI/Codex-Routen behalten OpenAI-spezifisches Request-Shaping wie `service_tier`, Responses-`store`, OpenAI-Reasoning-Kompatibilitäts-Payloads und Prompt-Cache-Hinweise bei
- OpenAI-kompatible Routen im Proxy-Stil behalten das lockerere Kompatibilitätsverhalten bei und erzwingen keine Strict-Tool-Schemas, kein natives Request-Shaping und keine verborgenen OpenAI/Codex-Attributions-Header

Azure OpenAI bleibt für Transport- und Kompatibilitätsverhalten in der Kategorie der nativen Routen, erhält aber nicht die verborgenen OpenAI/Codex-Attributions-Header.

Dadurch bleibt das aktuelle Verhalten nativer OpenAI-Responses erhalten, ohne ältere OpenAI-kompatible Shims auf Drittanbieter-Backends mit `/v1` zu erzwingen.

### Strikter agentischer GPT-Modus

Für GPT-5-Familienausführungen mit `openai/*` und `openai-codex/*` kann OpenClaw einen strengeren eingebetteten Pi-Ausführungsvertrag verwenden:

```json5
{
  agents: {
    defaults: {
      embeddedPi: {
        executionContract: "strict-agentic",
      },
    },
  },
}
```

Mit `strict-agentic` behandelt OpenClaw einen reinen Plan-Assistenten-Turn nicht mehr als erfolgreichen Fortschritt, wenn eine konkrete Tool-Aktion verfügbar ist. Es versucht den Turn mit einer Jetzt-handeln-Steuerung erneut, aktiviert das strukturierte Tool `update_plan` automatisch für umfangreichere Arbeit und zeigt einen expliziten blockierten Zustand an, wenn das Modell weiter plant, ohne zu handeln.

Der Modus ist auf GPT-5-Familienausführungen von OpenAI und OpenAI Codex begrenzt. Andere Anbieter und ältere Modellfamilien behalten das Standardverhalten des eingebetteten Pi bei, sofern du sie nicht explizit in andere Laufzeiteinstellungen aufnimmst.

### OpenAI Responses serverseitige Komprimierung

Für direkte OpenAI-Responses-Modelle (`openai/*` mit `api: "openai-responses"` und `baseUrl` auf `api.openai.com`) aktiviert OpenClaw jetzt automatisch Payload-Hinweise für die serverseitige Komprimierung von OpenAI:

- Erzwingt `store: true` (es sei denn, die Modellkompatibilität setzt `supportsStore: false`)
- Injiziert `context_management: [{ type: "compaction", compact_threshold: ... }]`

Standardmäßig ist `compact_threshold` auf `70%` des Modell-`contextWindow` gesetzt (oder auf `80000`, wenn nicht verfügbar).

### Serverseitige Komprimierung explizit aktivieren

Verwende dies, wenn du die Injektion von `context_management` bei kompatiblen Responses-Modellen erzwingen möchtest (zum Beispiel Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Mit einem benutzerdefinierten Schwellenwert aktivieren

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Serverseitige Komprimierung deaktivieren

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` steuert nur die Injektion von `context_management`.
Direkte OpenAI-Responses-Modelle erzwingen weiterhin `store: true`, es sei denn, die Kompatibilität setzt `supportsStore: false`.

## Hinweise

- Modellreferenzen verwenden immer `provider/model` (siehe [/concepts/models](/de/concepts/models)).
- Authentifizierungsdetails und Regeln zur Wiederverwendung stehen in [/concepts/oauth](/de/concepts/oauth).
