---
read_when:
    - Sie benötigen eine Referenz zur Modelleinrichtung pro Anbieter
    - Sie möchten Beispielkonfigurationen oder CLI-Onboarding-Befehle für Modellanbieter
summary: Überblick über Modellanbieter mit Beispielkonfigurationen + CLI-Abläufen
title: Modellanbieter
x-i18n:
    generated_at: "2026-04-08T02:15:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26b36a2bc19a28a7ef39aa8e81a0050fea1d452ac4969122e5cdf8755e690258
    source_path: concepts/model-providers.md
    workflow: 15
---

# Modellanbieter

Diese Seite behandelt **LLM-/Modellanbieter** (keine Chat-Kanäle wie WhatsApp/Telegram).
Informationen zu Regeln für die Modellauswahl finden Sie unter [/concepts/models](/de/concepts/models).

## Schnellregeln

- Modell-Refs verwenden `provider/model` (Beispiel: `opencode/claude-opus-4-6`).
- Wenn Sie `agents.defaults.models` festlegen, wird es zur Allowlist.
- CLI-Helfer: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Fallback-Laufzeitregeln, Cooldown-Sonden und Persistenz von Sitzungsüberschreibungen
  sind unter [/concepts/model-failover](/de/concepts/model-failover)
  dokumentiert.
- `models.providers.*.models[].contextWindow` sind native Modellmetadaten;
  `models.providers.*.models[].contextTokens` ist die effektive Laufzeitobergrenze.
- Anbieter-Plugins können Modellkataloge über `registerProvider({ catalog })` einspeisen;
  OpenClaw führt diese Ausgabe in `models.providers` zusammen, bevor
  `models.json` geschrieben wird.
- Anbieter-Manifeste können `providerAuthEnvVars` deklarieren, damit generische
  umgebungsvariablenbasierte Auth-Sonden die Plugin-Laufzeit nicht laden müssen. Die verbleibende Core-Env-Var-Zuordnung
  ist jetzt nur noch für Nicht-Plugin-/Core-Anbieter und einige Fälle mit generischer Priorität
  gedacht, etwa Anthropic-Onboarding mit API-Schlüssel zuerst.
- Anbieter-Plugins können auch das Laufzeitverhalten des Anbieters über
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot` und
  `onModelSelected` besitzen.
- Hinweis: Laufzeit-`capabilities` des Anbieters sind gemeinsame Runner-Metadaten (Anbieterfamilie,
  Transcript-/Tooling-Besonderheiten, Transport-/Cache-Hinweise). Sie sind nicht
  dasselbe wie das [öffentliche Fähigkeitsmodell](/de/plugins/architecture#public-capability-model),
  das beschreibt, was ein Plugin registriert (Text-Inferenz, Sprache usw.).

## Plugin-eigenes Anbieterverhalten

Anbieter-Plugins können jetzt den Großteil der anbieterbezogenen Logik besitzen, während OpenClaw
die generische Inferenzschleife beibehält.

Typische Aufteilung:

- `auth[].run` / `auth[].runNonInteractive`: Der Anbieter besitzt die Onboarding-/Login-
  Abläufe für `openclaw onboard`, `openclaw models auth` und Headless-Einrichtung
- `wizard.setup` / `wizard.modelPicker`: Der Anbieter besitzt Labels für Auth-Auswahl,
  Legacy-Aliase, Hinweise zur Allowlist im Onboarding und Einrichtungseinträge in Onboarding-/Modellwählern
- `catalog`: Der Anbieter erscheint in `models.providers`
- `normalizeModelId`: Der Anbieter normalisiert Legacy-/Vorschau-Modell-IDs vor
  Lookup oder Kanonisierung
- `normalizeTransport`: Der Anbieter normalisiert für die Transportfamilie `api` / `baseUrl`
  vor der generischen Modellzusammenstellung; OpenClaw prüft zuerst den zugeordneten Anbieter,
  dann andere anbieterpluginfähige Plugins, bis eines den
  Transport tatsächlich ändert
- `normalizeConfig`: Der Anbieter normalisiert die Konfiguration `models.providers.<id>` vor
  der Nutzung zur Laufzeit; OpenClaw prüft zuerst den zugeordneten Anbieter, dann andere
  anbieterpluginfähige Plugins, bis eines die Konfiguration tatsächlich ändert. Wenn kein
  Anbieter-Hook die Konfiguration umschreibt, normalisieren gebündelte Helfer der Google-Familie weiterhin
  unterstützte Google-Anbietereinträge.
- `applyNativeStreamingUsageCompat`: Der Anbieter wendet endpunktgesteuerte Kompatibilitäts-Umschreibungen für native Streaming-Nutzung auf Konfigurationsanbieter an
- `resolveConfigApiKey`: Der Anbieter löst Auth über Umgebungsmarker für Konfigurationsanbieter auf,
  ohne das vollständige Laufzeit-Auth laden zu müssen. `amazon-bedrock` hat hier außerdem einen
  eingebauten AWS-Umgebungsmarker-Resolver, obwohl die Bedrock-Laufzeit-Auth
  die Standardkette des AWS SDK verwendet.
- `resolveSyntheticAuth`: Der Anbieter kann Verfügbarkeit für lokale/self-hosted oder andere
  konfigurationsgestützte Auth bereitstellen, ohne Klartext-Geheimnisse zu speichern
- `shouldDeferSyntheticProfileAuth`: Der Anbieter kann gespeicherte synthetische Profil-
  Platzhalter mit niedrigerer Priorität markieren als env-/konfigurationsgestützte Auth
- `resolveDynamicModel`: Der Anbieter akzeptiert Modell-IDs, die noch nicht im lokalen
  statischen Katalog vorhanden sind
- `prepareDynamicModel`: Der Anbieter benötigt vor einem erneuten Versuch der
  dynamischen Auflösung eine Metadatenaktualisierung
- `normalizeResolvedModel`: Der Anbieter benötigt Umschreibungen für Transport oder Basis-URL
- `contributeResolvedModelCompat`: Der Anbieter liefert Kompatibilitäts-Flags für seine
  Herstellermodelle, selbst wenn sie über einen anderen kompatiblen Transport ankommen
- `capabilities`: Der Anbieter veröffentlicht Besonderheiten zu Transcript/Tooling/Anbieterfamilie
- `normalizeToolSchemas`: Der Anbieter bereinigt Tool-Schemas, bevor der eingebettete
  Runner sie sieht
- `inspectToolSchemas`: Der Anbieter stellt transportbezogene Schemawarnungen
  nach der Normalisierung bereit
- `resolveReasoningOutputMode`: Der Anbieter wählt native oder getaggte
  Verträge für Reasoning-Ausgaben
- `prepareExtraParams`: Der Anbieter setzt Standardwerte oder normalisiert anfragespezifische Parameter pro Modell
- `createStreamFn`: Der Anbieter ersetzt den normalen Stream-Pfad durch einen
  vollständig benutzerdefinierten Transport
- `wrapStreamFn`: Der Anbieter wendet Wrapper für Anfrage-Header/Body/Modell-Kompatibilität an
- `resolveTransportTurnState`: Der Anbieter liefert native Transport-
  Header oder Metadaten pro Turn
- `resolveWebSocketSessionPolicy`: Der Anbieter liefert native WebSocket-Sitzungs-
  Header oder eine Cooldown-Richtlinie für Sitzungen
- `createEmbeddingProvider`: Der Anbieter besitzt das Verhalten für Memory-Embeddings, wenn es
  beim Anbieter-Plugin statt beim Core-Embedding-Switchboard liegen soll
- `formatApiKey`: Der Anbieter formatiert gespeicherte Auth-Profile in die zur Laufzeit
  vom Transport erwartete Zeichenfolge `apiKey`
- `refreshOAuth`: Der Anbieter besitzt die OAuth-Aktualisierung, wenn die gemeinsamen
  `pi-ai`-Refresher nicht ausreichen
- `buildAuthDoctorHint`: Der Anbieter hängt Reparaturhinweise an, wenn die OAuth-Aktualisierung
  fehlschlägt
- `matchesContextOverflowError`: Der Anbieter erkennt anbieterspezifische
  Fehler bei Überschreitung des Kontextfensters, die generische Heuristiken übersehen würden
- `classifyFailoverReason`: Der Anbieter ordnet anbieterspezifische rohe Transport-/API-
  Fehler Failover-Gründen wie Ratenbegrenzung oder Überlastung zu
- `isCacheTtlEligible`: Der Anbieter entscheidet, welche Upstream-Modell-IDs Prompt-Cache-TTL unterstützen
- `buildMissingAuthMessage`: Der Anbieter ersetzt den generischen Fehler des Auth-Speichers
  durch einen anbieterspezifischen Wiederherstellungshinweis
- `suppressBuiltInModel`: Der Anbieter blendet veraltete Upstream-Zeilen aus und kann einen
  herstellereigenen Fehler für direkte Auflösungsfehler zurückgeben
- `augmentModelCatalog`: Der Anbieter hängt nach
  Erkennung und Konfigurationszusammenführung synthetische/finale Katalogzeilen an
- `isBinaryThinking`: Der Anbieter besitzt die UX für binäres Denken an/aus
- `supportsXHighThinking`: Der Anbieter aktiviert für ausgewählte Modelle `xhigh`
- `resolveDefaultThinkingLevel`: Der Anbieter besitzt die Standardrichtlinie von `/think` für eine
  Modellfamilie
- `applyConfigDefaults`: Der Anbieter wendet anbieterspezifische globale Standardwerte
  bei der Materialisierung der Konfiguration auf Basis von Auth-Modus, Umgebung oder Modellfamilie an
- `isModernModelRef`: Der Anbieter besitzt die Zuordnung bevorzugter Modelle für Live-/Smoke-Tests
- `prepareRuntimeAuth`: Der Anbieter wandelt ein konfiguriertes Credential in ein kurzlebiges
  Laufzeit-Token um
- `resolveUsageAuth`: Der Anbieter löst Credentials für Nutzung/Quota für `/usage`
  und verwandte Status-/Berichtsoberflächen auf
- `fetchUsageSnapshot`: Der Anbieter besitzt das Abrufen/Parsen des Nutzungsendpunkts, während
  der Core weiterhin die Zusammenfassungs-Shell und Formatierung besitzt
- `onModelSelected`: Der Anbieter führt nach der Auswahl Nebeneffekte wie
  Telemetrie oder anbieterbezogene Sitzungsbuchhaltung aus

Aktuelle gebündelte Beispiele:

- `anthropic`: Vorwärtskompatibler Claude-4.6-Fallback, Hinweise zur Auth-Reparatur, Abruf von Nutzungsendpunkten,
  Cache-TTL-/Anbieterfamilien-Metadaten und Auth-bewusste globale
  Konfigurationsstandardwerte
- `amazon-bedrock`: Anbieter-eigene Erkennung von Kontextüberläufen und
  Klassifizierung von Failover-Gründen für Bedrock-spezifische Throttle-/Not-ready-Fehler sowie
  die gemeinsame Replay-Familie `anthropic-by-model` für Claude-exklusive Replay-Richtlinien-
  Schutzmaßnahmen auf Anthropic-Datenverkehr
- `anthropic-vertex`: Claude-exklusive Replay-Richtlinien-Schutzmaßnahmen auf Anthropic-Message-
  Datenverkehr
- `openrouter`: Durchgereichte Modell-IDs, Anfrage-Wrapper, Hinweise zu Anbieterfähigkeiten,
  Bereinigung von Gemini-Thought-Signatures bei proxy-basiertem Gemini-Datenverkehr,
  Proxy-Reasoning-Injektion über die Stream-Familie `openrouter-thinking`, Weiterleitung
  von Routing-Metadaten und Cache-TTL-Richtlinie
- `github-copilot`: Onboarding/Geräte-Login, vorwärtskompatibler Modell-Fallback,
  Hinweise zu Claude-Thinking-Transcripts, Austausch von Laufzeit-Tokens und Abruf von Nutzungsendpunkten
- `openai`: Vorwärtskompatibler GPT-5.4-Fallback, direkte OpenAI-Transport-
  Normalisierung, Codex-bewusste Hinweise bei fehlender Auth, Unterdrückung von Spark, synthetische
  OpenAI-/Codex-Katalogzeilen, Thinking-/Live-Modell-Richtlinie, Normalisierung von Aliasen für
  Usage-Token (`input` / `output` und `prompt` / `completion`-Familien), die
  gemeinsame Stream-Familie `openai-responses-defaults` für native OpenAI-/Codex-
  Wrapper, Metadaten zur Anbieterfamilie, gebündelte Registrierung von Bildgenerierungsanbietern
  für `gpt-image-1` und gebündelte Registrierung von Videogenerierungsanbietern
  für `sora-2`
- `google` und `google-gemini-cli`: Vorwärtskompatibler Gemini-3.1-Fallback,
  native Gemini-Replay-Validierung, Bereinigung von Bootstrap-Replays, getaggter
  Modus für Reasoning-Ausgaben, Modern-Model-Matching, gebündelte Registrierung von Bildgenerierungs-
  anbietern für Gemini-Bildvorschau-Modelle und gebündelte
  Registrierung von Videogenerierungsanbietern für Veo-Modelle; Gemini-CLI-OAuth besitzt außerdem
  Tokenformatierung für Auth-Profile, Parsing von Usage-Token und Abruf von Quota-Endpunkten
  für Nutzungsoberflächen
- `moonshot`: Gemeinsamer Transport, plugin-eigene Normalisierung von Thinking-Payloads
- `kilocode`: Gemeinsamer Transport, plugin-eigene Anfrage-Header, Reasoning-Payload-
  Normalisierung, Bereinigung von Thought-Signatures bei Proxy-Gemini und Cache-TTL-
  Richtlinie
- `zai`: Vorwärtskompatibler GLM-5-Fallback, Standardwerte für `tool_stream`, Cache-TTL-
  Richtlinie, Richtlinie für binäres Denken/Live-Modelle sowie Auth + Quota-Abruf für Nutzung;
  unbekannte `glm-5*`-IDs werden aus der gebündelten Vorlage `glm-4.7` synthetisiert
- `xai`: Native Responses-Transport-Normalisierung, Umschreibungen von `/fast`-Aliasen für
  schnelle Grok-Varianten, Standardwert für `tool_stream`, xAI-spezifische Bereinigung von Tool-Schemas /
  Reasoning-Payloads und gebündelte Registrierung von Videogenerierungsanbietern
  für `grok-imagine-video`
- `mistral`: Plugin-eigene Fähigkeitsmetadaten
- `opencode` und `opencode-go`: Plugin-eigene Fähigkeitsmetadaten plus
  Bereinigung von Thought-Signatures bei Proxy-Gemini
- `alibaba`: Plugin-eigener Videogenerierungskatalog für direkte Wan-Modell-Refs
  wie `alibaba/wan2.6-t2v`
- `byteplus`: Plugin-eigene Kataloge plus gebündelte Registrierung von Videogenerierungsanbietern
  für Seedance-Modelle für Text-zu-Video/Bild-zu-Video
- `fal`: gebündelte Registrierung von Videogenerierungsanbietern für gehostete Drittanbieter-
  Registrierung von Bildgenerierungsanbietern für FLUX-Bildmodelle plus gebündelte
  Registrierung von Videogenerierungsanbietern für gehostete Drittanbieter-Videomodelle
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` und `volcengine`:
  nur plugin-eigene Kataloge
- `qwen`: Plugin-eigene Kataloge für Textmodelle plus gemeinsame
  Registrierungen von Media-Understanding- und Videogenerierungsanbietern für seine
  multimodalen Oberflächen; Qwen-Videogenerierung verwendet die Standard-DashScope-Video-
  Endpunkte mit gebündelten Wan-Modellen wie `wan2.6-t2v` und `wan2.7-r2v`
- `runway`: Plugin-eigene Registrierung von Videogenerierungsanbietern für native,
  auf Aufgaben basierende Runway-Modelle wie `gen4.5`
- `minimax`: Plugin-eigene Kataloge, gebündelte Registrierung von Videogenerierungsanbietern
  für Hailuo-Videomodelle, gebündelte Registrierung von Bildgenerierungsanbietern
  für `image-01`, hybride Auswahl von Anthropic-/OpenAI-Replay-Richtlinien
  sowie Auth-/Snapshot-Logik für Nutzung
- `together`: Plugin-eigene Kataloge plus gebündelte Registrierung von Videogenerierungsanbietern
  für Wan-Videomodelle
- `xiaomi`: Plugin-eigene Kataloge plus Auth-/Snapshot-Logik für Nutzung

Das gebündelte Plugin `openai` besitzt jetzt beide Anbieter-IDs: `openai` und
`openai-codex`.

Damit sind Anbieter abgedeckt, die weiterhin in die normalen Transporte von OpenClaw passen. Ein Anbieter,
der einen vollständig benutzerdefinierten Anfrage-Executor benötigt, ist eine separate, tiefergehende
Erweiterungsoberfläche.

## API-Schlüsselrotation

- Unterstützt generische Rotation für ausgewählte Anbieter.
- Konfigurieren Sie mehrere Schlüssel über:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (einzelne Live-Überschreibung, höchste Priorität)
  - `<PROVIDER>_API_KEYS` (durch Komma oder Semikolon getrennte Liste)
  - `<PROVIDER>_API_KEY` (primärer Schlüssel)
  - `<PROVIDER>_API_KEY_*` (nummerierte Liste, z. B. `<PROVIDER>_API_KEY_1`)
- Für Google-Anbieter wird `GOOGLE_API_KEY` zusätzlich als Fallback einbezogen.
- Die Schlüsselreihenfolge bewahrt die Priorität und entfernt doppelte Werte.
- Anfragen werden nur bei Antworten mit Ratenbegrenzung mit dem nächsten Schlüssel erneut versucht (zum
  Beispiel `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` oder periodische Meldungen über Nutzungslimits).
- Fehler ohne Ratenbegrenzung schlagen sofort fehl; es wird keine Schlüsselrotation versucht.
- Wenn alle Kandidatenschlüssel fehlschlagen, wird der letzte Fehler aus dem letzten Versuch zurückgegeben.

## Integrierte Anbieter (pi-ai-Katalog)

OpenClaw wird mit dem pi-ai-Katalog ausgeliefert. Diese Anbieter benötigen **keine**
`models.providers`-Konfiguration; setzen Sie einfach Auth und wählen Sie ein Modell aus.

### OpenAI

- Anbieter: `openai`
- Auth: `OPENAI_API_KEY`
- Optionale Rotation: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, plus `OPENCLAW_LIVE_OPENAI_KEY` (einzelne Überschreibung)
- Beispielmodelle: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Standardtransport ist `auto` (WebSocket zuerst, SSE-Fallback)
- Pro Modell überschreiben über `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` oder `"auto"`)
- Aufwärmen für OpenAI Responses WebSocket ist standardmäßig über `params.openaiWsWarmup` (`true`/`false`) aktiviert
- Priority Processing von OpenAI kann über `agents.defaults.models["openai/<model>"].params.serviceTier` aktiviert werden
- `/fast` und `params.fastMode` ordnen direkte `openai/*`-Responses-Anfragen auf `service_tier=priority` bei `api.openai.com` zu
- Verwenden Sie `params.serviceTier`, wenn Sie statt des gemeinsamen Schalters `/fast` einen expliziten Tier-Wert möchten
- Versteckte OpenClaw-Attributionsheader (`originator`, `version`,
  `User-Agent`) gelten nur für nativen OpenAI-Datenverkehr zu `api.openai.com`, nicht
  für generische OpenAI-kompatible Proxys
- Native OpenAI-Routen behalten außerdem Responses-`store`, Prompt-Cache-Hinweise und
  OpenAI-Reasoning-kompatible Payload-Anpassung bei; Proxy-Routen nicht
- `openai/gpt-5.3-codex-spark` wird in OpenClaw absichtlich unterdrückt, weil die Live-OpenAI-API es ablehnt; Spark wird als nur für Codex behandelt

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Anbieter: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- Optionale Rotation: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, plus `OPENCLAW_LIVE_ANTHROPIC_KEY` (einzelne Überschreibung)
- Beispielmodell: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Direkte öffentliche Anthropic-Anfragen unterstützen den gemeinsamen Schalter `/fast` und `params.fastMode`, einschließlich per API-Schlüssel und OAuth authentifiziertem Datenverkehr an `api.anthropic.com`; OpenClaw ordnet dies Anthropic-`service_tier` zu (`auto` vs `standard_only`)
- Hinweis zu Anthropic: Mitarbeiter von Anthropic haben uns mitgeteilt, dass die Nutzung im Stil von OpenClaw Claude CLI wieder erlaubt ist, daher behandelt OpenClaw die Wiederverwendung von Claude CLI und die Nutzung von `claude -p` für diese Integration als zulässig, sofern Anthropic keine neue Richtlinie veröffentlicht.
- Das Anthropic-Setup-Token bleibt als unterstützter OpenClaw-Tokenpfad verfügbar, OpenClaw bevorzugt nun jedoch die Wiederverwendung von Claude CLI und `claude -p`, wenn verfügbar.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Anbieter: `openai-codex`
- Auth: OAuth (ChatGPT)
- Beispielmodell: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` oder `openclaw models auth login --provider openai-codex`
- Standardtransport ist `auto` (WebSocket zuerst, SSE-Fallback)
- Pro Modell überschreiben über `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` oder `"auto"`)
- `params.serviceTier` wird ebenfalls bei nativen Codex-Responses-Anfragen (`chatgpt.com/backend-api`) weitergeleitet
- Versteckte OpenClaw-Attributionsheader (`originator`, `version`,
  `User-Agent`) werden nur an nativen Codex-Datenverkehr zu
  `chatgpt.com/backend-api` angehängt, nicht an generische OpenAI-kompatible Proxys
- Verwendet denselben Schalter `/fast` und dieselbe Konfiguration `params.fastMode` wie direktes `openai/*`; OpenClaw ordnet dies `service_tier=priority` zu
- `openai-codex/gpt-5.3-codex-spark` bleibt verfügbar, wenn der Codex-OAuth-Katalog es anbietet; abhängig von Berechtigungen
- `openai-codex/gpt-5.4` behält das native `contextWindow = 1050000` und ein standardmäßiges Laufzeit-`contextTokens = 272000`; überschreiben Sie die Laufzeitobergrenze mit `models.providers.openai-codex.models[].contextTokens`
- Richtlinienhinweis: OpenAI-Codex-OAuth wird ausdrücklich für externe Tools/Workflows wie OpenClaw unterstützt.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Andere gehostete Optionen im Abonnementstil

- [Qwen Cloud](/de/providers/qwen): Oberfläche des Qwen-Cloud-Anbieters plus Zuordnung von Endpunkten für Alibaba DashScope und Coding Plan
- [MiniMax](/de/providers/minimax): OAuth- oder API-Schlüsselzugriff für MiniMax Coding Plan
- [GLM Models](/de/providers/glm): Z.AI Coding Plan oder allgemeine API-Endpunkte

### OpenCode

- Auth: `OPENCODE_API_KEY` (oder `OPENCODE_ZEN_API_KEY`)
- Zen-Laufzeitanbieter: `opencode`
- Go-Laufzeitanbieter: `opencode-go`
- Beispielmodelle: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` oder `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API-Schlüssel)

- Anbieter: `google`
- Auth: `GEMINI_API_KEY`
- Optionale Rotation: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY` als Fallback und `OPENCLAW_LIVE_GEMINI_KEY` (einzelne Überschreibung)
- Beispielmodelle: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Kompatibilität: Legacy-OpenClaw-Konfiguration mit `google/gemini-3.1-flash-preview` wird zu `google/gemini-3-flash-preview` normalisiert
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Direkte Gemini-Ausführungen akzeptieren außerdem `agents.defaults.models["google/<model>"].params.cachedContent`
  (oder das Legacy-`cached_content`), um einen anbieter-nativen
  Handle `cachedContents/...` weiterzuleiten; Gemini-Cache-Treffer erscheinen in OpenClaw als `cacheRead`

### Google Vertex und Gemini CLI

- Anbieter: `google-vertex`, `google-gemini-cli`
- Auth: Vertex verwendet gcloud ADC; Gemini CLI verwendet seinen OAuth-Flow
- Achtung: Gemini-CLI-OAuth in OpenClaw ist eine inoffizielle Integration. Einige Nutzer haben nach der Verwendung von Drittanbieter-Clients von Einschränkungen ihres Google-Kontos berichtet. Prüfen Sie die Google-Bedingungen und verwenden Sie ein nicht kritisches Konto, wenn Sie sich dafür entscheiden.
- Gemini-CLI-OAuth wird als Teil des gebündelten Plugins `google` ausgeliefert.
  - Installieren Sie zuerst Gemini CLI:
    - `brew install gemini-cli`
    - oder `npm install -g @google/gemini-cli`
  - Aktivieren: `openclaw plugins enable google`
  - Anmelden: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Standardmodell: `google-gemini-cli/gemini-3-flash-preview`
  - Hinweis: Sie fügen **keine** Client-ID oder kein Secret in `openclaw.json` ein. Der CLI-Login-Flow speichert
    Tokens in Auth-Profilen auf dem Gateway-Host.
  - Wenn Anfragen nach der Anmeldung fehlschlagen, setzen Sie `GOOGLE_CLOUD_PROJECT` oder `GOOGLE_CLOUD_PROJECT_ID` auf dem Gateway-Host.
  - Gemini-CLI-JSON-Antworten werden aus `response` geparst; die Nutzung fällt auf
    `stats` zurück, wobei `stats.cached` zu OpenClaw-`cacheRead` normalisiert wird.

### Z.AI (GLM)

- Anbieter: `zai`
- Auth: `ZAI_API_KEY`
- Beispielmodell: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Aliase: `z.ai/*` und `z-ai/*` werden zu `zai/*` normalisiert
  - `zai-api-key` erkennt automatisch den passenden Z.AI-Endpunkt; `zai-coding-global`, `zai-coding-cn`, `zai-global` und `zai-cn` erzwingen eine bestimmte Oberfläche

### Vercel AI Gateway

- Anbieter: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- Beispielmodell: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Anbieter: `kilocode`
- Auth: `KILOCODE_API_KEY`
- Beispielmodell: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Basis-URL: `https://api.kilo.ai/api/gateway/`
- Der statische Fallback-Katalog enthält `kilocode/kilo/auto`; die Live-
  Erkennung über `https://api.kilo.ai/api/gateway/models` kann den Laufzeit-
  Katalog weiter erweitern.
- Das genaue Upstream-Routing hinter `kilocode/kilo/auto` gehört zu Kilo Gateway
  und ist nicht fest in OpenClaw codiert.

Einzelheiten zur Einrichtung finden Sie unter [/providers/kilocode](/de/providers/kilocode).

### Andere gebündelte Anbieter-Plugins

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Beispielmodell: `openrouter/auto`
- OpenClaw wendet die von OpenRouter dokumentierten App-Attributionsheader nur an,
  wenn die Anfrage tatsächlich an `openrouter.ai` geht
- OpenRouter-spezifische Anthropic-`cache_control`-Marker sind ebenso auf
  verifizierte OpenRouter-Routen beschränkt, nicht auf beliebige Proxy-URLs
- OpenRouter bleibt auf dem Proxy-Stil-Pfad für OpenAI-Kompatibilität, daher wird keine
  native, nur für OpenAI bestimmte Anfrageanpassung (`serviceTier`, Responses-`store`,
  Prompt-Cache-Hinweise, OpenAI-Reasoning-kompatible Payloads) weitergeleitet
- Gemini-basierte OpenRouter-Refs behalten nur die Bereinigung von Thought-Signatures bei Proxy-Gemini
  bei; native Gemini-Replay-Validierung und Bootstrap-Umschreibungen bleiben deaktiviert
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Beispielmodell: `kilocode/kilo/auto`
- Gemini-basierte Kilo-Refs behalten denselben Pfad zur Bereinigung von Thought-Signatures bei Proxy-Gemini;
  `kilocode/kilo/auto` und andere Hinweise ohne Unterstützung für Proxy-Reasoning
  überspringen die Proxy-Reasoning-Injektion
- MiniMax: `minimax` (API-Schlüssel) und `minimax-portal` (OAuth)
- Auth: `MINIMAX_API_KEY` für `minimax`; `MINIMAX_OAUTH_TOKEN` oder `MINIMAX_API_KEY` für `minimax-portal`
- Beispielmodell: `minimax/MiniMax-M2.7` oder `minimax-portal/MiniMax-M2.7`
- Onboarding/Einrichtung per API-Schlüssel für MiniMax schreibt explizite M2.7-Modelldefinitionen mit
  `input: ["text", "image"]`; der gebündelte Anbieterkatalog hält die Chat-Refs
  text-only, bis diese Anbieterkonfiguration materialisiert wird
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Beispielmodell: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` oder `KIMICODE_API_KEY`)
- Beispielmodell: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Beispielmodell: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` oder `DASHSCOPE_API_KEY`)
- Beispielmodell: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Beispielmodell: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Beispielmodelle: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Beispielmodell: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Beispielmodell: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` oder `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Beispielmodell: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Beispielmodell: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Native gebündelte xAI-Anfragen verwenden den xAI-Responses-Pfad
  - `/fast` oder `params.fastMode: true` schreiben `grok-3`, `grok-3-mini`,
    `grok-4` und `grok-4-0709` auf ihre `*-fast`-Varianten um
  - `tool_stream` ist standardmäßig aktiviert; setzen Sie
    `agents.defaults.models["xai/<model>"].params.tool_stream` auf `false`, um
    es zu deaktivieren
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Beispielmodell: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - GLM-Modelle auf Cerebras verwenden die IDs `zai-glm-4.7` und `zai-glm-4.6`.
  - OpenAI-kompatible Basis-URL: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Beispielmodell für Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Siehe [Hugging Face (Inference)](/de/providers/huggingface).

## Anbieter über `models.providers` (benutzerdefiniert/Basis-URL)

Verwenden Sie `models.providers` (oder `models.json`), um **benutzerdefinierte** Anbieter oder
OpenAI-/Anthropic-kompatible Proxys hinzuzufügen.

Viele der unten aufgeführten gebündelten Anbieter-Plugins veröffentlichen bereits einen Standardkatalog.
Verwenden Sie explizite Einträge `models.providers.<id>` nur dann, wenn Sie die
standardmäßige Basis-URL, Header oder Modellliste überschreiben möchten.

### Moonshot AI (Kimi)

Moonshot wird als gebündeltes Anbieter-Plugin ausgeliefert. Verwenden Sie standardmäßig den integrierten Anbieter
und fügen Sie nur dann einen expliziten Eintrag `models.providers.moonshot` hinzu, wenn Sie die Basis-URL oder Modellmetadaten
überschreiben müssen:

- Anbieter: `moonshot`
- Auth: `MOONSHOT_API_KEY`
- Beispielmodell: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` oder `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi-K2-Modell-IDs:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding verwendet den Anthropic-kompatiblen Endpunkt von Moonshot AI:

- Anbieter: `kimi`
- Auth: `KIMI_API_KEY`
- Beispielmodell: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

Das Legacy-`kimi/k2p5` bleibt als Kompatibilitäts-Modell-ID akzeptiert.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) bietet Zugriff auf Doubao und andere Modelle in China.

- Anbieter: `volcengine` (Coding: `volcengine-plan`)
- Auth: `VOLCANO_ENGINE_API_KEY`
- Beispielmodell: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

Im Onboarding wird standardmäßig die Coding-Oberfläche verwendet, aber der allgemeine `volcengine/*`-
Katalog wird gleichzeitig registriert.

In Modellwählern für Onboarding/Konfiguration bevorzugt die Volcengine-Auth-Auswahl sowohl
Zeilen `volcengine/*` als auch `volcengine-plan/*`. Wenn diese Modelle noch nicht geladen sind,
fällt OpenClaw auf den ungefilterten Katalog zurück, statt einen leeren
anbieterspezifischen Wähler anzuzeigen.

Verfügbare Modelle:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Coding-Modelle (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (International)

BytePlus ARK bietet internationalen Nutzern Zugriff auf dieselben Modelle wie Volcano Engine.

- Anbieter: `byteplus` (Coding: `byteplus-plan`)
- Auth: `BYTEPLUS_API_KEY`
- Beispielmodell: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

Im Onboarding wird standardmäßig die Coding-Oberfläche verwendet, aber der allgemeine `byteplus/*`-
Katalog wird gleichzeitig registriert.

In Modellwählern für Onboarding/Konfiguration bevorzugt die BytePlus-Auth-Auswahl sowohl
Zeilen `byteplus/*` als auch `byteplus-plan/*`. Wenn diese Modelle noch nicht geladen sind,
fällt OpenClaw auf den ungefilterten Katalog zurück, statt einen leeren
anbieterspezifischen Wähler anzuzeigen.

Verfügbare Modelle:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Coding-Modelle (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic stellt Anthropic-kompatible Modelle hinter dem Anbieter `synthetic` bereit:

- Anbieter: `synthetic`
- Auth: `SYNTHETIC_API_KEY`
- Beispielmodell: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax wird über `models.providers` konfiguriert, da es benutzerdefinierte Endpunkte verwendet:

- MiniMax OAuth (global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax API-Schlüssel (global): `--auth-choice minimax-global-api`
- MiniMax API-Schlüssel (CN): `--auth-choice minimax-cn-api`
- Auth: `MINIMAX_API_KEY` für `minimax`; `MINIMAX_OAUTH_TOKEN` oder
  `MINIMAX_API_KEY` für `minimax-portal`

Einzelheiten zur Einrichtung, zu Modelloptionen und Konfigurations-Snippets finden Sie unter [/providers/minimax](/de/providers/minimax).

Auf dem Anthropic-kompatiblen Streaming-Pfad von MiniMax deaktiviert OpenClaw Thinking
standardmäßig, sofern Sie es nicht explizit festlegen, und `/fast on` schreibt
`MiniMax-M2.7` auf `MiniMax-M2.7-highspeed` um.

Plugin-eigene Aufteilung der Fähigkeiten:

- Standardwerte für Text/Chat bleiben auf `minimax/MiniMax-M2.7`
- Bildgenerierung ist `minimax/image-01` oder `minimax-portal/image-01`
- Bildverständnis ist das plugin-eigene `MiniMax-VL-01` auf beiden MiniMax-Auth-Pfaden
- Websuche bleibt auf der Anbieter-ID `minimax`

### Ollama

Ollama wird als gebündeltes Anbieter-Plugin ausgeliefert und verwendet die native API von Ollama:

- Anbieter: `ollama`
- Auth: Nicht erforderlich (lokaler Server)
- Beispielmodell: `ollama/llama3.3`
- Installation: [https://ollama.com/download](https://ollama.com/download)

```bash
# Installieren Sie Ollama und laden Sie dann ein Modell:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama wird lokal unter `http://127.0.0.1:11434` erkannt, wenn Sie sich mit
`OLLAMA_API_KEY` dafür anmelden, und das gebündelte Anbieter-Plugin fügt Ollama direkt zu
`openclaw onboard` und dem Modellwähler hinzu. Informationen zu Onboarding, Cloud-/Lokalmodus und benutzerdefinierter Konfiguration finden Sie unter [/providers/ollama](/de/providers/ollama).

### vLLM

vLLM wird als gebündeltes Anbieter-Plugin für lokale/self-hosted OpenAI-kompatible
Server ausgeliefert:

- Anbieter: `vllm`
- Auth: Optional (abhängig von Ihrem Server)
- Standard-Basis-URL: `http://127.0.0.1:8000/v1`

So aktivieren Sie die automatische lokale Erkennung (jeder Wert funktioniert, wenn Ihr Server keine Auth erzwingt):

```bash
export VLLM_API_KEY="vllm-local"
```

Legen Sie dann ein Modell fest (ersetzen Sie es durch eine der IDs, die von `/v1/models` zurückgegeben werden):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Einzelheiten finden Sie unter [/providers/vllm](/de/providers/vllm).

### SGLang

SGLang wird als gebündeltes Anbieter-Plugin für schnelle self-hosted
OpenAI-kompatible Server ausgeliefert:

- Anbieter: `sglang`
- Auth: Optional (abhängig von Ihrem Server)
- Standard-Basis-URL: `http://127.0.0.1:30000/v1`

So aktivieren Sie die automatische lokale Erkennung (jeder Wert funktioniert, wenn Ihr Server keine
Auth erzwingt):

```bash
export SGLANG_API_KEY="sglang-local"
```

Legen Sie dann ein Modell fest (ersetzen Sie es durch eine der IDs, die von `/v1/models` zurückgegeben werden):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Einzelheiten finden Sie unter [/providers/sglang](/de/providers/sglang).

### Lokale Proxys (LM Studio, vLLM, LiteLLM usw.)

Beispiel (OpenAI-kompatibel):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Hinweise:

- Für benutzerdefinierte Anbieter sind `reasoning`, `input`, `cost`, `contextWindow` und `maxTokens` optional.
  Wenn sie weggelassen werden, verwendet OpenClaw standardmäßig:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Empfohlen: Legen Sie explizite Werte fest, die zu den Grenzen Ihres Proxys/Modells passen.
- Für `api: "openai-completions"` auf nicht nativen Endpunkten (jede nicht leere `baseUrl`, deren Host nicht `api.openai.com` ist), erzwingt OpenClaw `compat.supportsDeveloperRole: false`, um Provider-400-Fehler für nicht unterstützte `developer`-Rollen zu vermeiden.
- Proxyartige OpenAI-kompatible Routen überspringen außerdem native, nur für OpenAI bestimmte Anfrage-
  Anpassung: kein `service_tier`, kein Responses-`store`, keine Prompt-Cache-Hinweise, keine
  OpenAI-Reasoning-kompatible Payload-Anpassung und keine versteckten OpenClaw-Attributions-
  Header.
- Wenn `baseUrl` leer ist oder weggelassen wird, behält OpenClaw das Standardverhalten von OpenAI bei (das zu `api.openai.com` aufgelöst wird).
- Aus Sicherheitsgründen wird ein explizites `compat.supportsDeveloperRole: true` auf nicht nativen `openai-completions`-Endpunkten dennoch überschrieben.

## CLI-Beispiele

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Siehe auch: [/gateway/configuration](/de/gateway/configuration) für vollständige Konfigurationsbeispiele.

## Verwandt

- [Models](/de/concepts/models) — Modellkonfiguration und Aliase
- [Model Failover](/de/concepts/model-failover) — Fallback-Ketten und Wiederholungsverhalten
- [Configuration Reference](/de/gateway/configuration-reference#agent-defaults) — Konfigurationsschlüssel für Modelle
- [Providers](/de/providers) — Einrichtungsanleitungen pro Anbieter
