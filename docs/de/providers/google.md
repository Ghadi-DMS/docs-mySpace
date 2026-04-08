---
read_when:
    - Sie möchten Google-Gemini-Modelle mit OpenClaw verwenden
    - Sie benötigen den Auth-Flow für API-Schlüssel oder OAuth
summary: Einrichtung von Google Gemini (API-Schlüssel + OAuth, Bildgenerierung, Medienverständnis, Websuche)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T02:17:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: e9e558f5ce35c853e0240350be9a1890460c5f7f7fd30b05813a656497dee516
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Das Google-Plugin bietet Zugriff auf Gemini-Modelle über Google AI Studio sowie
Bildgenerierung, Medienverständnis (Bild/Audio/Video) und Websuche über
Gemini Grounding.

- Anbieter: `google`
- Auth: `GEMINI_API_KEY` oder `GOOGLE_API_KEY`
- API: Google Gemini API
- Alternativer Anbieter: `google-gemini-cli` (OAuth)

## Schnellstart

1. API-Schlüssel festlegen:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Ein Standardmodell festlegen:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Nicht interaktives Beispiel

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Ein alternativer Anbieter `google-gemini-cli` verwendet PKCE-OAuth statt eines API-
Schlüssels. Dies ist eine inoffizielle Integration; einige Nutzer berichten von Konto-
einschränkungen. Verwendung auf eigenes Risiko.

- Standardmodell: `google-gemini-cli/gemini-3-flash-preview`
- Alias: `gemini-cli`
- Installationsvoraussetzung: lokale Gemini CLI als `gemini` verfügbar
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- Anmeldung:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Umgebungsvariablen:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(Oder die Varianten `GEMINI_CLI_*`.)

Wenn Gemini-CLI-OAuth-Anfragen nach der Anmeldung fehlschlagen, setzen Sie
`GOOGLE_CLOUD_PROJECT` oder `GOOGLE_CLOUD_PROJECT_ID` auf dem Gateway-Host und
versuchen Sie es erneut.

Wenn die Anmeldung fehlschlägt, bevor der Browser-Flow startet, stellen Sie sicher, dass der lokale Befehl `gemini`
installiert und auf `PATH` verfügbar ist. OpenClaw unterstützt sowohl Homebrew-Installationen
als auch globale npm-Installationen, einschließlich gängiger Windows-/npm-Layouts.

Hinweise zur JSON-Nutzung von Gemini CLI:

- Antworttext stammt aus dem JSON-Feld `response` der CLI.
- Die Nutzung fällt auf `stats` zurück, wenn die CLI `usage` leer lässt.
- `stats.cached` wird zu OpenClaw-`cacheRead` normalisiert.
- Wenn `stats.input` fehlt, leitet OpenClaw die Eingabetokens aus
  `stats.input_tokens - stats.cached` ab.

## Fähigkeiten

| Fähigkeit              | Unterstützt      |
| ---------------------- | ---------------- |
| Chat Completions       | Ja               |
| Bildgenerierung        | Ja               |
| Musikgenerierung       | Ja               |
| Bildverständnis        | Ja               |
| Audiotranskription     | Ja               |
| Videoverständnis       | Ja               |
| Websuche (Grounding)   | Ja               |
| Thinking/Reasoning     | Ja (Gemini 3.1+) |

## Direkte Wiederverwendung des Gemini-Cache

Bei direkten Gemini-API-Ausführungen (`api: "google-generative-ai"`) reicht OpenClaw jetzt
einen konfigurierten `cachedContent`-Handle an Gemini-Anfragen durch.

- Konfigurieren Sie pro Modell oder globalen Parametern entweder
  `cachedContent` oder das Legacy-`cached_content`
- Wenn beide vorhanden sind, hat `cachedContent` Vorrang
- Beispielwert: `cachedContents/prebuilt-context`
- Die Nutzung bei Gemini-Cache-Treffern wird in OpenClaw als `cacheRead` aus
  Upstream-`cachedContentTokenCount` normalisiert

Beispiel:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Bildgenerierung

Der gebündelte Bildgenerierungsanbieter `google` verwendet standardmäßig
`google/gemini-3.1-flash-image-preview`.

- Unterstützt außerdem `google/gemini-3-pro-image-preview`
- Generieren: bis zu 4 Bilder pro Anfrage
- Bearbeitungsmodus: aktiviert, bis zu 5 Eingabebilder
- Geometriesteuerungen: `size`, `aspectRatio` und `resolution`

Der OAuth-only-Anbieter `google-gemini-cli` ist eine separate Oberfläche
für Textinferenz. Bildgenerierung, Medienverständnis und Gemini Grounding bleiben auf
der Anbieter-ID `google`.

So verwenden Sie Google als Standardanbieter für Bilder:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Unter [Image Generation](/de/tools/image-generation) finden Sie die gemeinsam genutzten Tool-
Parameter, die Anbieterauswahl und das Failover-Verhalten.

## Videogenerierung

Das gebündelte Plugin `google` registriert auch Videogenerierung über das gemeinsame
Tool `video_generate`.

- Standard-Videomodell: `google/veo-3.1-fast-generate-preview`
- Modi: Text-zu-Video, Bild-zu-Video und Abläufe mit einzelner Video-Referenz
- Unterstützt `aspectRatio`, `resolution` und `audio`
- Aktuelle Begrenzung der Dauer: **4 bis 8 Sekunden**

So verwenden Sie Google als Standardanbieter für Videos:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Unter [Video Generation](/de/tools/video-generation) finden Sie die gemeinsam genutzten Tool-
Parameter, die Anbieterauswahl und das Failover-Verhalten.

## Musikgenerierung

Das gebündelte Plugin `google` registriert auch Musikgenerierung über das gemeinsame
Tool `music_generate`.

- Standard-Musikmodell: `google/lyria-3-clip-preview`
- Unterstützt außerdem `google/lyria-3-pro-preview`
- Prompt-Steuerungen: `lyrics` und `instrumental`
- Ausgabeformat: standardmäßig `mp3`, zusätzlich `wav` bei `google/lyria-3-pro-preview`
- Referenzeingaben: bis zu 10 Bilder
- Sitzungsgebundene Ausführungen werden über den gemeinsamen Task-/Status-Ablauf entkoppelt, einschließlich `action: "status"`

So verwenden Sie Google als Standardanbieter für Musik:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Unter [Music Generation](/de/tools/music-generation) finden Sie die gemeinsam genutzten Tool-
Parameter, die Anbieterauswahl und das Failover-Verhalten.

## Hinweis zur Umgebung

Wenn das Gateway als Daemon läuft (launchd/systemd), stellen Sie sicher, dass `GEMINI_API_KEY`
für diesen Prozess verfügbar ist (zum Beispiel in `~/.openclaw/.env` oder über
`env.shellEnv`).
