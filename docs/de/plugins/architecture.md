---
read_when:
    - Erstellen oder Debuggen nativer OpenClaw-Plugins
    - Das Fähigkeitsmodell von Plugins oder Eigentumsgrenzen verstehen
    - An der Plugin-Ladepipeline oder Registry arbeiten
    - Provider-Laufzeit-Hooks oder Kanal-Plugins implementieren
sidebarTitle: Internals
summary: 'Plugin-Interna: Fähigkeitsmodell, Eigentümerschaft, Verträge, Ladepipeline und Laufzeit-Hilfsfunktionen'
title: Plugin-Interna
x-i18n:
    generated_at: "2026-04-08T02:20:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: c40ecf14e2a0b2b8d332027aed939cd61fb4289a489f4cd4c076c96d707d1138
    source_path: plugins/architecture.md
    workflow: 15
---

# Plugin-Interna

<Info>
  Dies ist die **ausführliche Architekturreferenz**. Praktische Anleitungen finden Sie unter:
  - [Plugins installieren und verwenden](/de/tools/plugin) — Benutzerhandbuch
  - [Erste Schritte](/de/plugins/building-plugins) — erstes Plugin-Tutorial
  - [Kanal-Plugins](/de/plugins/sdk-channel-plugins) — einen Messaging-Kanal erstellen
  - [Provider-Plugins](/de/plugins/sdk-provider-plugins) — einen Modell-Provider erstellen
  - [SDK-Überblick](/de/plugins/sdk-overview) — Import-Map und Registrierungs-API
</Info>

Diese Seite behandelt die interne Architektur des OpenClaw-Plugin-Systems.

## Öffentliches Fähigkeitsmodell

Fähigkeiten sind das öffentliche Modell für **native Plugins** innerhalb von OpenClaw. Jedes
native OpenClaw-Plugin registriert sich für einen oder mehrere Fähigkeitstypen:

| Fähigkeit             | Registrierungsmethode                           | Beispiel-Plugins                    |
| --------------------- | ----------------------------------------------- | ----------------------------------- |
| Textinferenz          | `api.registerProvider(...)`                     | `openai`, `anthropic`               |
| CLI-Inferenz-Backend  | `api.registerCliBackend(...)`                   | `openai`, `anthropic`               |
| Sprache               | `api.registerSpeechProvider(...)`               | `elevenlabs`, `microsoft`           |
| Echtzeit-Transkription | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                            |
| Echtzeit-Stimme       | `api.registerRealtimeVoiceProvider(...)`        | `openai`                            |
| Medienverständnis     | `api.registerMediaUnderstandingProvider(...)`   | `openai`, `google`                  |
| Bildgenerierung       | `api.registerImageGenerationProvider(...)`      | `openai`, `google`, `fal`, `minimax` |
| Musikgenerierung      | `api.registerMusicGenerationProvider(...)`      | `google`, `minimax`                 |
| Videogenerierung      | `api.registerVideoGenerationProvider(...)`      | `qwen`                              |
| Web-Abruf             | `api.registerWebFetchProvider(...)`             | `firecrawl`                         |
| Websuche              | `api.registerWebSearchProvider(...)`            | `google`                            |
| Kanal / Messaging     | `api.registerChannel(...)`                      | `msteams`, `matrix`                 |

Ein Plugin, das null Fähigkeiten registriert, aber Hooks, Tools oder
Dienste bereitstellt, ist ein **Legacy-Hook-only**-Plugin. Dieses Muster wird weiterhin vollständig unterstützt.

### Haltung zur externen Kompatibilität

Das Fähigkeitsmodell ist im Core angekommen und wird heute von gebündelten/nativen Plugins
verwendet, aber die Kompatibilität für externe Plugins braucht weiterhin einen strengeren Maßstab als „es wird
exportiert, also ist es eingefroren“.

Aktuelle Richtlinien:

- **bestehende externe Plugins:** Hook-basierte Integrationen funktionsfähig halten; behandeln Sie
  dies als Kompatibilitäts-Basislinie
- **neue gebündelte/native Plugins:** explizite Fähigkeitsregistrierung gegenüber
  herstellerspezifischen Eingriffen oder neuen Hook-only-Designs bevorzugen
- **externe Plugins, die Fähigkeitsregistrierung übernehmen:** erlaubt, aber die
  fähigkeitsspezifischen Hilfsoberflächen als in Entwicklung betrachten, sofern die Dokumentation einen
  Vertrag nicht ausdrücklich als stabil kennzeichnet

Praktische Regel:

- APIs zur Fähigkeitsregistrierung sind die beabsichtigte Richtung
- Legacy-Hooks bleiben während
  des Übergangs der sicherste Weg ohne Breaking Changes für externe Plugins
- exportierte Hilfs-Subpaths sind nicht alle gleich; bevorzugen Sie den eng dokumentierten
  Vertrag, nicht zufällige Hilfsexporte

### Plugin-Formen

OpenClaw klassifiziert jedes geladene Plugin anhand seines tatsächlichen
Registrierungsverhaltens in eine Form (nicht nur anhand statischer Metadaten):

- **plain-capability** -- registriert genau einen Fähigkeitstyp (zum Beispiel ein
  reines Provider-Plugin wie `mistral`)
- **hybrid-capability** -- registriert mehrere Fähigkeitstypen (zum Beispiel
  besitzt `openai` Textinferenz, Sprache, Medienverständnis und Bild-
  generierung)
- **hook-only** -- registriert nur Hooks (typisiert oder benutzerdefiniert), keine Fähigkeiten,
  Tools, Befehle oder Dienste
- **non-capability** -- registriert Tools, Befehle, Dienste oder Routen, aber keine
  Fähigkeiten

Verwenden Sie `openclaw plugins inspect <id>`, um die Form und Aufschlüsselung der Fähigkeiten
eines Plugins anzuzeigen. Details finden Sie in der [CLI-Referenz](/cli/plugins#inspect).

### Legacy-Hooks

Der Hook `before_agent_start` bleibt als Kompatibilitätspfad für
Hook-only-Plugins unterstützt. Reale ältere Plugins sind weiterhin davon abhängig.

Ausrichtung:

- funktionsfähig halten
- als veraltet dokumentieren
- `before_model_resolve` für Arbeit an Modell-/Provider-Overrides bevorzugen
- `before_prompt_build` für Prompt-Mutationen bevorzugen
- erst entfernen, wenn die reale Nutzung zurückgeht und Fixture-Abdeckung Migrationssicherheit beweist

### Kompatibilitätssignale

Wenn Sie `openclaw doctor` oder `openclaw plugins inspect <id>` ausführen, sehen Sie möglicherweise
eine dieser Kennzeichnungen:

| Signal                     | Bedeutung                                                   |
| -------------------------- | ----------------------------------------------------------- |
| **config valid**           | Konfiguration wird korrekt geparst und Plugins werden aufgelöst |
| **compatibility advisory** | Plugin verwendet ein unterstütztes, aber älteres Muster (z. B. `hook-only`) |
| **legacy warning**         | Plugin verwendet `before_agent_start`, das veraltet ist     |
| **hard error**             | Konfiguration ist ungültig oder Plugin konnte nicht geladen werden |

Weder `hook-only` noch `before_agent_start` machen Ihr Plugin heute kaputt --
`hook-only` ist ein Hinweis, und `before_agent_start` löst nur eine Warnung aus. Diese
Signale erscheinen auch in `openclaw status --all` und `openclaw plugins doctor`.

## Architekturüberblick

Das Plugin-System von OpenClaw hat vier Schichten:

1. **Manifest + Discovery**
   OpenClaw findet Kandidaten-Plugins aus konfigurierten Pfaden, Workspace-Wurzeln,
   globalen Erweiterungswurzeln und gebündelten Erweiterungen. Discovery liest zuerst native
   `openclaw.plugin.json`-Manifeste sowie unterstützte Bundle-Manifeste.
2. **Aktivierung + Validierung**
   Der Core entscheidet, ob ein entdecktes Plugin aktiviert, deaktiviert, blockiert oder
   für einen exklusiven Slot wie Memory ausgewählt ist.
3. **Laufzeitladen**
   Native OpenClaw-Plugins werden im Prozess über jiti geladen und registrieren
   Fähigkeiten in einer zentralen Registry. Kompatible Bundles werden in
   Registry-Einträge normalisiert, ohne Laufzeitcode zu importieren.
4. **Nutzung der Oberflächen**
   Der Rest von OpenClaw liest die Registry, um Tools, Kanäle, Provider-
   Einrichtung, Hooks, HTTP-Routen, CLI-Befehle und Dienste bereitzustellen.

Speziell für die Plugin-CLI ist die Discovery von Root-Befehlen in zwei Phasen aufgeteilt:

- Parse-Time-Metadaten stammen aus `registerCli(..., { descriptors: [...] })`
- das eigentliche Plugin-CLI-Modul kann lazy bleiben und sich beim ersten Aufruf registrieren

Dadurch bleibt plugin-eigener CLI-Code im Plugin, während OpenClaw dennoch
Root-Befehlsnamen vor dem Parsen reservieren kann.

Die wichtige Designgrenze:

- Discovery + Konfigurationsvalidierung sollten aus **Manifest-/Schema-Metadaten**
  funktionieren, ohne Plugin-Code auszuführen
- natives Laufzeitverhalten stammt aus dem Pfad `register(api)` des Plugin-Moduls

Diese Aufteilung ermöglicht es OpenClaw, Konfigurationen zu validieren, fehlende/deaktivierte Plugins zu erklären und
UI-/Schema-Hinweise aufzubauen, bevor die vollständige Laufzeit aktiv ist.

### Kanal-Plugins und das gemeinsame Message-Tool

Kanal-Plugins müssen für normale Chat-Aktionen kein separates Tool zum Senden/Bearbeiten/Reagieren registrieren.
OpenClaw hält ein gemeinsames `message`-Tool im Core, und
Kanal-Plugins besitzen die kanalspezifische Discovery und Ausführung dahinter.

Die aktuelle Grenze ist:

- der Core besitzt den gemeinsamen Tool-Host `message`, Prompt-Verdrahtung, Sitzungs-/Thread-
  Buchführung und Ausführungs-Dispatch
- Kanal-Plugins besitzen scoped Action-Discovery, Fähigkeits-Discovery und alle
  kanalspezifischen Schemafragmente
- Kanal-Plugins besitzen die providerspezifische Sitzungs-Gesprächsgrammatik, also
  wie Gesprächs-IDs Thread-IDs codieren oder von Elterngesprächen erben
- Kanal-Plugins führen die finale Aktion über ihren Action-Adapter aus

Für Kanal-Plugins ist die SDK-Oberfläche
`ChannelMessageActionAdapter.describeMessageTool(...)`. Dieser einheitliche Discovery-
Aufruf ermöglicht es einem Plugin, seine sichtbaren Aktionen, Fähigkeiten und Schema-
Beiträge zusammen zurückzugeben, damit diese Teile nicht auseinanderlaufen.

Der Core übergibt den Laufzeit-Scope an diesen Discovery-Schritt. Wichtige Felder sind:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- vertrauenswürdige eingehende `requesterSenderId`

Das ist wichtig für kontextsensitive Plugins. Ein Kanal kann
Message-Aktionen basierend auf dem aktiven Konto, dem aktuellen Raum/Thread/Nachricht oder der
vertrauenswürdigen Identität des Anfragenden ausblenden oder anzeigen, ohne kanalspezifische Verzweigungen
im Core-Tool `message` hart zu codieren.

Deshalb bleiben Änderungen am Embedded-Runner-Routing Plugin-Arbeit: Der Runner ist
dafür verantwortlich, die aktuelle Chat-/Sitzungsidentität an die Plugin-
Discovery-Grenze weiterzugeben, damit das gemeinsame Tool `message` die richtige
kanaleigene Oberfläche für den aktuellen Zug bereitstellt.

Für kanaleigene Ausführungs-Hilfsfunktionen sollten gebündelte Plugins die Ausführungs-
Laufzeit in ihren eigenen Erweiterungsmodulen halten. Der Core besitzt nicht mehr die
Laufzeiten für Discord-, Slack-, Telegram- oder WhatsApp-Message-Aktionen unter `src/agents/tools`.
Wir veröffentlichen keine separaten `plugin-sdk/*-action-runtime`-Subpaths, und gebündelte
Plugins sollten ihren eigenen lokalen Laufzeitcode direkt aus ihren
erweiterungseigenen Modulen importieren.

Dieselbe Grenze gilt allgemein für providerbenannte SDK-Seams: Der Core sollte
keine kanalspezifischen Convenience-Barrels für Slack, Discord, Signal,
WhatsApp oder ähnliche Erweiterungen importieren. Wenn der Core ein Verhalten braucht, dann entweder über das
eigene `api.ts`- / `runtime-api.ts`-Barrel des gebündelten Plugins konsumieren oder den Bedarf
in eine schmale generische Fähigkeit im gemeinsamen SDK überführen.

Speziell für Umfragen gibt es zwei Ausführungspfade:

- `outbound.sendPoll` ist die gemeinsame Basis für Kanäle, die zum allgemeinen
  Umfragemodell passen
- `actions.handleAction("poll")` ist der bevorzugte Pfad für kanalspezifische
  Umfragesemantik oder zusätzliche Umfrageparameter

Der Core verschiebt das gemeinsame Umfrage-Parsing nun auf den Zeitpunkt nach dem Ablehnen
des Actions durch den Plugin-Dispatch für Umfragen, sodass plugin-eigene Umfrage-Handler
kanalspezifische Umfragefelder akzeptieren können, ohne zuerst vom generischen
Umfrage-Parser blockiert zu werden.

Die vollständige Startsequenz finden Sie unter [Ladepipeline](#load-pipeline).

## Modell für Fähigkeitseigentum

OpenClaw behandelt ein natives Plugin als Eigentumsgrenze für ein **Unternehmen** oder ein
**Feature**, nicht als Sammelsurium nicht zusammenhängender Integrationen.

Das bedeutet:

- ein Unternehmens-Plugin sollte normalerweise alle OpenClaw-bezogenen
  Oberflächen dieses Unternehmens besitzen
- ein Feature-Plugin sollte normalerweise die vollständige Oberfläche des eingeführten Features besitzen
- Kanäle sollten gemeinsame Core-Fähigkeiten nutzen, statt Provider-Verhalten ad hoc neu zu implementieren

Beispiele:

- das gebündelte Plugin `openai` besitzt OpenAI-Modell-Provider-Verhalten und OpenAI-
  Verhalten für Sprache + Echtzeit-Stimme + Medienverständnis + Bildgenerierung
- das gebündelte Plugin `elevenlabs` besitzt ElevenLabs-Sprachverhalten
- das gebündelte Plugin `microsoft` besitzt Microsoft-Sprachverhalten
- das gebündelte Plugin `google` besitzt Google-Modell-Provider-Verhalten sowie Google-
  Verhalten für Medienverständnis + Bildgenerierung + Websuche
- das gebündelte Plugin `firecrawl` besitzt Firecrawl-Web-Abrufverhalten
- die gebündelten Plugins `minimax`, `mistral`, `moonshot` und `zai` besitzen ihre
  Backends für Medienverständnis
- das gebündelte Plugin `qwen` besitzt Qwen-Textprovider-Verhalten sowie
  Verhalten für Medienverständnis und Videogenerierung
- das Plugin `voice-call` ist ein Feature-Plugin: Es besitzt Anruftransport, Tools,
  CLI, Routen und Twilio-Media-Stream-Bridge, nutzt aber gemeinsame Sprach- sowie Echtzeit-
  Transkriptions- und Echtzeit-Stimmfähigkeiten, statt Hersteller-Plugins direkt
  zu importieren

Der beabsichtigte Endzustand ist:

- OpenAI lebt in einem Plugin, selbst wenn es Textmodelle, Sprache, Bilder und
  künftig Video umfasst
- ein anderer Anbieter kann dasselbe für seine eigene Oberfläche tun
- Kanäle interessiert nicht, welches Anbieter-Plugin den Provider besitzt; sie nutzen den vom Core offengelegten gemeinsamen Fähigkeitsvertrag

Das ist die entscheidende Unterscheidung:

- **plugin** = Eigentumsgrenze
- **capability** = Core-Vertrag, den mehrere Plugins implementieren oder nutzen können

Wenn OpenClaw also einen neuen Bereich wie Video hinzufügt, lautet die erste Frage nicht
„welcher Provider sollte Videoverarbeitung hart codieren?“ Die erste Frage ist:
„wie lautet der Core-Vertrag für die Videofähigkeit?“ Sobald dieser Vertrag existiert,
können Anbieter-Plugins sich dafür registrieren, und Kanal-/Feature-Plugins können ihn nutzen.

Wenn die Fähigkeit noch nicht existiert, ist der richtige Schritt normalerweise:

1. die fehlende Fähigkeit im Core definieren
2. sie typisiert über die Plugin-API/Laufzeit verfügbar machen
3. Kanäle/Features an diese Fähigkeit anbinden
4. Anbieter-Plugins ihre Implementierungen registrieren lassen

Dadurch bleibt Eigentum explizit, während vermieden wird, dass Core-Verhalten von einem
einzigen Anbieter oder einem einmaligen pluginspezifischen Codepfad abhängt.

### Fähigkeitsschichtung

Verwenden Sie dieses mentale Modell, wenn Sie entscheiden, wohin Code gehört:

- **Core-Fähigkeitsschicht**: gemeinsame Orchestrierung, Richtlinien, Fallback, Konfigurations-
  Merge-Regeln, Auslieferungssemantik und typisierte Verträge
- **Anbieter-Plugin-Schicht**: anbieterspezifische APIs, Authentifizierung, Modellkataloge, Sprach-
  synthese, Bildgenerierung, künftige Video-Backends, Usage-Endpunkte
- **Kanal-/Feature-Plugin-Schicht**: Slack-/Discord-/voice-call-/usw.-Integration,
  die Core-Fähigkeiten nutzt und sie auf einer Oberfläche präsentiert

Zum Beispiel folgt TTS dieser Form:

- der Core besitzt TTS-Richtlinie zur Antwortzeit, Fallback-Reihenfolge, Präferenzen und Kanalauslieferung
- `openai`, `elevenlabs` und `microsoft` besitzen Synthese-Implementierungen
- `voice-call` nutzt die Laufzeit-Hilfsfunktion für Telephony-TTS

Dasselbe Muster sollte für künftige Fähigkeiten bevorzugt werden.

### Beispiel für ein Unternehmens-Plugin mit mehreren Fähigkeiten

Ein Unternehmens-Plugin sollte sich von außen kohärent anfühlen. Wenn OpenClaw gemeinsame
Verträge für Modelle, Sprache, Echtzeit-Transkription, Echtzeit-Stimme, Medien-
verständnis, Bildgenerierung, Videogenerierung, Web-Abruf und Websuche hat,
kann ein Anbieter alle seine Oberflächen an einer Stelle besitzen:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

Wichtig sind nicht die exakten Hilfsnamen. Wichtig ist die Form:

- ein Plugin besitzt die Anbieteroberfläche
- der Core besitzt weiterhin die Fähigkeitsverträge
- Kanäle und Feature-Plugins nutzen `api.runtime.*`-Hilfsfunktionen, keinen Anbietercode
- Vertragstests können prüfen, dass das Plugin die Fähigkeiten registriert hat, die
  es zu besitzen beansprucht

### Fähigkeitsbeispiel: Videoverständnis

OpenClaw behandelt Bild-/Audio-/Videoverständnis bereits als eine gemeinsame
Fähigkeit. Dasselbe Eigentumsmodell gilt dort:

1. der Core definiert den Vertrag für Medienverständnis
2. Anbieter-Plugins registrieren `describeImage`, `transcribeAudio` und
   `describeVideo`, je nach Anwendbarkeit
3. Kanal- und Feature-Plugins nutzen das gemeinsame Core-Verhalten, statt direkt an Anbietercode anzubinden

Dadurch wird vermieden, dass die Videoannahmen eines Providers in den Core eingebrannt werden. Das Plugin besitzt
die Anbieteroberfläche; der Core besitzt den Fähigkeitsvertrag und das Fallback-Verhalten.

Videogenerierung verwendet bereits dieselbe Abfolge: Der Core besitzt den typisierten
Fähigkeitsvertrag und die Laufzeit-Hilfsfunktion, und Anbieter-Plugins registrieren
`api.registerVideoGenerationProvider(...)`-Implementierungen dafür.

Brauchen Sie eine konkrete Rollout-Checkliste? Siehe
[Capability Cookbook](/de/plugins/architecture).

## Verträge und Durchsetzung

Die Oberfläche der Plugin-API ist bewusst typisiert und in
`OpenClawPluginApi` zentralisiert. Dieser Vertrag definiert die unterstützten Registrierungsstellen und
die Laufzeit-Hilfsfunktionen, auf die sich ein Plugin verlassen darf.

Warum das wichtig ist:

- Plugin-Autoren erhalten einen stabilen internen Standard
- der Core kann doppelte Eigentümerschaft ablehnen, etwa wenn zwei Plugins dieselbe
  Provider-ID registrieren
- der Start kann umsetzbare Diagnosen für fehlerhafte Registrierungen anzeigen
- Vertragstests können Eigentümerschaft gebündelter Plugins durchsetzen und stilles Drift verhindern

Es gibt zwei Ebenen der Durchsetzung:

1. **Durchsetzung bei der Laufzeitregistrierung**
   Die Plugin-Registry validiert Registrierungen, während Plugins geladen werden. Beispiele:
   doppelte Provider-IDs, doppelte Sprach-Provider-IDs und fehlerhafte
   Registrierungen erzeugen Plugin-Diagnosen statt undefinierten Verhaltens.
2. **Vertragstests**
   Gebündelte Plugins werden in Vertrag-Registries während Testläufen erfasst, damit
   OpenClaw Eigentümerschaft explizit prüfen kann. Heute wird dies für Modell-
   Provider, Sprach-Provider, Websuch-Provider und Eigentümerschaft gebündelter Registrierungen verwendet.

Der praktische Effekt ist, dass OpenClaw im Voraus weiß, welches Plugin welche
Oberfläche besitzt. Dadurch können Core und Kanäle nahtlos zusammenspielen, weil Eigentum
deklariert, typisiert und testbar statt implizit ist.

### Was in einen Vertrag gehört

Gute Plugin-Verträge sind:

- typisiert
- klein
- fähigkeitsspezifisch
- im Besitz des Core
- für mehrere Plugins wiederverwendbar
- für Kanäle/Features ohne Anbieterwissen nutzbar

Schlechte Plugin-Verträge sind:

- anbieterspezifische Richtlinien, die im Core versteckt sind
- einmalige Plugin-Fluchtwege, die die Registry umgehen
- Kanalcode, der direkt in eine Anbieterimplementierung greift
- ad hoc Laufzeitobjekte, die nicht Teil von `OpenClawPluginApi` oder
  `api.runtime` sind

Im Zweifel die Abstraktionsebene anheben: zuerst die Fähigkeit definieren, dann
Plugins daran andocken lassen.

## Ausführungsmodell

Native OpenClaw-Plugins laufen **im Prozess** mit dem Gateway. Sie sind nicht
sandboxed. Ein geladenes natives Plugin hat dieselbe Vertrauensgrenze auf Prozessebene wie
Core-Code.

Auswirkungen:

- ein natives Plugin kann Tools, Netzwerk-Handler, Hooks und Dienste registrieren
- ein Fehler in einem nativen Plugin kann das Gateway abstürzen lassen oder destabilisieren
- ein bösartiges natives Plugin entspricht beliebiger Codeausführung innerhalb des
  OpenClaw-Prozesses

Kompatible Bundles sind standardmäßig sicherer, weil OpenClaw sie derzeit als
Metadaten-/Content-Pakete behandelt. In aktuellen Releases bedeutet das meist
gebündelte Skills.

Verwenden Sie Allowlists und explizite Installations-/Ladepfade für nicht gebündelte Plugins. Behandeln Sie
Workspace-Plugins als Entwicklungscode, nicht als Produktionsstandard.

Bei Namen gebündelter Workspace-Pakete sollte die Plugin-ID im npm-
Namen verankert bleiben: standardmäßig `@openclaw/<id>` oder ein genehmigtes typisiertes Suffix wie
`-provider`, `-plugin`, `-speech`, `-sandbox` oder `-media-understanding`, wenn
das Paket absichtlich eine schmalere Plugin-Rolle exponiert.

Wichtiger Hinweis zum Vertrauen:

- `plugins.allow` vertraut **Plugin-IDs**, nicht der Herkunft des Quellcodes.
- Ein Workspace-Plugin mit derselben ID wie ein gebündeltes Plugin überschattet
  absichtlich die gebündelte Kopie, wenn dieses Workspace-Plugin aktiviert/auf der Allowlist ist.
- Das ist normal und nützlich für lokale Entwicklung, Patch-Tests und Hotfixes.

## Exportgrenze

OpenClaw exportiert Fähigkeiten, nicht bequeme Implementierungsdetails.

Halten Sie die Fähigkeitsregistrierung öffentlich. Straffen Sie nichtvertragliche Hilfsexporte:

- gebündelte pluginspezifische Hilfs-Subpaths
- Laufzeit-Subpaths, die nicht als öffentliche API gedacht sind
- anbieterspezifische Convenience-Hilfsfunktionen
- Setup-/Onboarding-Hilfsfunktionen, die Implementierungsdetails sind

Einige Hilfs-Subpaths gebündelter Plugins verbleiben aus Kompatibilitäts- und Wartungsgründen
weiterhin in der generierten SDK-Export-Map. Aktuelle Beispiele sind
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` und mehrere `plugin-sdk/matrix*`-Seams. Behandeln Sie diese als
reservierte Exporte von Implementierungsdetails, nicht als empfohlenes SDK-Muster für
neue Plugins von Drittanbietern.

## Ladepipeline

Beim Start macht OpenClaw ungefähr Folgendes:

1. Kandidaten-Plugin-Wurzeln entdecken
2. native oder kompatible Bundle-Manifeste und Paketmetadaten lesen
3. unsichere Kandidaten ablehnen
4. Plugin-Konfiguration normalisieren (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. für jeden Kandidaten die Aktivierung entscheiden
6. aktivierte native Module über jiti laden
7. native Hooks `register(api)` (oder `activate(api)` — ein älterer Alias) aufrufen und Registrierungen in der Plugin-Registry sammeln
8. die Registry CLI-/Laufzeitoberflächen bereitstellen

<Note>
`activate` ist ein älterer Alias für `register` — der Loader löst die jeweils vorhandene Variante auf (`def.register ?? def.activate`) und ruft sie an derselben Stelle auf. Alle gebündelten Plugins verwenden `register`; für neue Plugins `register` bevorzugen.
</Note>

Die Sicherheitsprüfungen finden **vor** der Laufzeitausführung statt. Kandidaten werden blockiert,
wenn der Entry aus der Plugin-Wurzel herausführt, der Pfad weltbeschreibbar ist oder die Eigentümerschaft
des Pfads bei nicht gebündelten Plugins verdächtig aussieht.

### Manifest-First-Verhalten

Das Manifest ist die Quelle der Wahrheit für die Control Plane. OpenClaw verwendet es, um:

- das Plugin zu identifizieren
- deklarierte Kanäle/Skills/Konfigurationsschema oder Bundle-Fähigkeiten zu erkennen
- `plugins.entries.<id>.config` zu validieren
- Labels/Platzhalter der Control UI anzureichern
- Installations-/Katalogmetadaten anzuzeigen

Für native Plugins ist das Laufzeitmodul der Data-Plane-Teil. Es registriert
tatsächliches Verhalten wie Hooks, Tools, Befehle oder Provider-Flows.

### Was der Loader cached

OpenClaw hält kurze In-Process-Caches für:

- Discovery-Ergebnisse
- Manifest-Registry-Daten
- geladene Plugin-Registries

Diese Caches reduzieren burstige Starts und den Overhead wiederholter Befehle. Es ist sinnvoll,
sie als kurzlebige Performance-Caches und nicht als Persistenz zu betrachten.

Hinweis zur Performance:

- Setzen Sie `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` oder
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1`, um diese Caches zu deaktivieren.
- Passen Sie Cache-Fenster mit `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` und
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS` an.

## Registry-Modell

Geladene Plugins mutieren nicht direkt zufällige globale Objekte des Core. Sie registrieren sich in einer
zentralen Plugin-Registry.

Die Registry verfolgt:

- Plugin-Einträge (Identität, Quelle, Herkunft, Status, Diagnosen)
- Tools
- ältere Hooks und typisierte Hooks
- Kanäle
- Provider
- Gateway-RPC-Handler
- HTTP-Routen
- CLI-Registrare
- Hintergrunddienste
- plugin-eigene Befehle

Core-Features lesen dann aus dieser Registry, statt direkt mit Plugin-Modulen zu sprechen.
Das hält das Laden einseitig:

- Plugin-Modul -> Registry-Registrierung
- Core-Laufzeit -> Registry-Konsum

Diese Trennung ist wichtig für die Wartbarkeit. Sie bedeutet, dass die meisten Core-Oberflächen
nur einen Integrationspunkt brauchen: „die Registry lesen“, nicht „jedes Plugin-Modul gesondert behandeln“.

## Callbacks für Gesprächsbindungen

Plugins, die ein Gespräch binden, können reagieren, wenn eine Genehmigung aufgelöst wird.

Verwenden Sie `api.onConversationBindingResolved(...)`, um nach Genehmigung oder Ablehnung einer
Bindungsanfrage einen Callback zu erhalten:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Felder der Callback-Nutzlast:

- `status`: `"approved"` oder `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` oder `"deny"`
- `binding`: die aufgelöste Bindung für genehmigte Anfragen
- `request`: die ursprüngliche Anfragezusammenfassung, Detach-Hinweis, Sender-ID und
  Gesprächsmetadaten

Dieser Callback dient nur zur Benachrichtigung. Er ändert nicht, wer ein Gespräch binden darf,
und er läuft, nachdem die Core-Behandlung der Genehmigung abgeschlossen ist.

## Provider-Laufzeit-Hooks

Provider-Plugins haben jetzt zwei Ebenen:

- Manifest-Metadaten: `providerAuthEnvVars` für kostengünstige env-basierte Provider-Auth-Suche
  vor dem Laden der Laufzeit, `channelEnvVars` für kostengünstige env-/Setup-Suche für Kanäle
  vor dem Laden der Laufzeit sowie `providerAuthChoices` für kostengünstige Labels bei Onboarding/Auth-Auswahl
  und CLI-Flag-Metadaten vor dem Laden der Laufzeit
- Hooks zur Konfigurationszeit: `catalog` / älteres `discovery` plus `applyConfigDefaults`
- Laufzeit-Hooks: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw besitzt weiterhin die generische Agent-Schleife, das Failover, die Transkriptbehandlung und
die Tool-Richtlinien. Diese Hooks sind die Erweiterungsoberfläche für providerspezifisches Verhalten, ohne
einen vollständig benutzerdefinierten Inferenz-Transport zu benötigen.

Verwenden Sie Manifest-`providerAuthEnvVars`, wenn der Provider env-basierte Zugangsdaten hat,
die generische Auth-/Status-/Model-Picker-Pfade sehen sollen, ohne die Plugin-Laufzeit zu laden.
Verwenden Sie Manifest-`providerAuthChoices`, wenn Oberflächen für Onboarding/Auth-Auswahl in der CLI
die Choice-ID, Gruppenlabels und einfache
Auth-Verdrahtung per einzelner Flag des Providers kennen sollen, ohne die Provider-Laufzeit zu laden. Behalten Sie Provider-Laufzeit-
`envVars` für operatorseitige Hinweise wie Onboarding-Labels oder Variablen für die Einrichtung von OAuth-
Client-ID/Client-Secret bei.

Verwenden Sie Manifest-`channelEnvVars`, wenn ein Kanal env-gesteuerte Authentifizierung oder Einrichtung hat,
die generischer Shell-env-Fallback, Prüfungen von Konfiguration/Status oder Setup-Abfragen sehen sollen,
ohne die Kanallaufzeit zu laden.

### Hook-Reihenfolge und Verwendung

Für Modell-/Provider-Plugins ruft OpenClaw Hooks ungefähr in dieser Reihenfolge auf.
Die Spalte „Verwendung“ ist die schnelle Entscheidungshilfe.

| #   | Hook                              | Was er tut                                                                                                      | Verwendung                                                                                                                                  |
| --- | --------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Veröffentlicht Provider-Konfiguration in `models.providers` während der Generierung von `models.json`          | Der Provider besitzt einen Katalog oder Standardwerte für `baseUrl`                                                                         |
| 2   | `applyConfigDefaults`             | Wendet provider-eigene globale Konfigurationsstandards während der Konfigurationsmaterialisierung an           | Standards hängen von Auth-Modus, Umgebung oder der Semantik der Modellfamilie des Providers ab                                             |
| --  | _(built-in model lookup)_         | OpenClaw versucht zuerst den normalen Registry-/Katalogpfad                                                    | _(kein Plugin-Hook)_                                                                                                                        |
| 3   | `normalizeModelId`                | Normalisiert ältere oder Preview-Modell-ID-Aliasse vor dem Lookup                                              | Der Provider besitzt Alias-Bereinigung vor der kanonischen Modellauflösung                                                                 |
| 4   | `normalizeTransport`              | Normalisiert `api` / `baseUrl` der Provider-Familie vor der generischen Modellassemblierung                    | Der Provider besitzt Transport-Bereinigung für benutzerdefinierte Provider-IDs in derselben Transportfamilie                              |
| 5   | `normalizeConfig`                 | Normalisiert `models.providers.<id>` vor Laufzeit-/Provider-Auflösung                                          | Der Provider benötigt Konfigurationsbereinigung, die beim Plugin liegen sollte; gebündelte Hilfsfunktionen der Google-Familie stützen außerdem unterstützte Google-Konfigurationseinträge ab |
| 6   | `applyNativeStreamingUsageCompat` | Wendet native Umschreibungen für Streaming-Usage-Kompatibilität auf Konfigurations-Provider an                 | Der Provider benötigt endpointgesteuerte Korrekturen an nativen Streaming-Usage-Metadaten                                                  |
| 7   | `resolveConfigApiKey`             | Löst env-marker-Authentifizierung für Konfigurations-Provider vor dem Laden der Laufzeit-Auth auf             | Der Provider besitzt eigene API-Key-Auflösung über env-marker; `amazon-bedrock` hat hier ebenfalls einen eingebauten AWS-env-marker-Resolver |
| 8   | `resolveSyntheticAuth`            | Macht lokale/self-hosted oder konfigurationsgestützte Auth sichtbar, ohne Klartext zu persistieren            | Der Provider kann mit einem synthetischen/lokalen Zugangsdaten-Marker arbeiten                                                             |
| 9   | `resolveExternalAuthProfiles`     | Legt provider-eigene externe Auth-Profile darüber; Standard-`persistence` ist `runtime-only` für CLI-/app-eigene Zugangsdaten | Der Provider verwendet externe Auth-Zugangsdaten wieder, ohne kopierte Refresh-Tokens zu persistieren                                      |
| 10  | `shouldDeferSyntheticProfileAuth` | Ordnet gespeicherte Platzhalterprofile für synthetische Profile hinter env-/konfigurationsgestützter Auth ein | Der Provider speichert synthetische Platzhalterprofile, die nicht Vorrang haben sollten                                                     |
| 11  | `resolveDynamicModel`             | Synchroner Fallback für provider-eigene Modell-IDs, die noch nicht in der lokalen Registry sind               | Der Provider akzeptiert beliebige Upstream-Modell-IDs                                                                                       |
| 12  | `prepareDynamicModel`             | Asynchrones Warm-up, dann läuft `resolveDynamicModel` erneut                                                   | Der Provider benötigt Netzwerkmetadaten, bevor unbekannte IDs aufgelöst werden können                                                      |
| 13  | `normalizeResolvedModel`          | Letzte Umschreibung, bevor der Embedded-Runner das aufgelöste Modell verwendet                                 | Der Provider benötigt Transport-Umschreibungen, verwendet aber weiterhin einen Core-Transport                                               |
| 14  | `contributeResolvedModelCompat`   | Liefert Kompatibilitäts-Flags für Anbietermodelle hinter einem anderen kompatiblen Transport                   | Der Provider erkennt seine eigenen Modelle auf Proxy-Transporten, ohne den Provider zu übernehmen                                          |
| 15  | `capabilities`                    | Provider-eigene Transkript-/Tooling-Metadaten, die von geteilter Core-Logik genutzt werden                    | Der Provider benötigt Eigenheiten von Transkript/Provider-Familie                                                                           |
| 16  | `normalizeToolSchemas`            | Normalisiert Tool-Schemas, bevor der Embedded-Runner sie sieht                                                 | Der Provider benötigt schema-bezogene Bereinigung für die Transportfamilie                                                                  |
| 17  | `inspectToolSchemas`              | Macht provider-eigene Schema-Diagnosen nach der Normalisierung sichtbar                                        | Der Provider möchte Keyword-Warnungen anzeigen, ohne dem Core providerspezifische Regeln beizubringen                                      |
| 18  | `resolveReasoningOutputMode`      | Wählt nativen oder getaggten Vertrag für den Reasoning-Output                                                  | Der Provider benötigt getaggten Reasoning-/Final-Output statt nativer Felder                                                               |
| 19  | `prepareExtraParams`              | Normalisierung von Anfrageparametern vor generischen Stream-Option-Wrappern                                    | Der Provider benötigt Standard-Anfrageparameter oder providerspezifische Bereinigung pro Provider                                           |
| 20  | `createStreamFn`                  | Ersetzt den normalen Stream-Pfad vollständig durch einen benutzerdefinierten Transport                         | Der Provider benötigt ein benutzerdefiniertes Wire-Protokoll, nicht nur einen Wrapper                                                      |
| 21  | `wrapStreamFn`                    | Stream-Wrapper, nachdem generische Wrapper angewendet wurden                                                   | Der Provider benötigt Wrapper für Anfrage-Header/Body/Modell-Kompatibilität ohne benutzerdefinierten Transport                            |
| 22  | `resolveTransportTurnState`       | Hängt native Header oder Metadaten pro Zug an den Transport                                                    | Der Provider möchte, dass generische Transporte provider-native Zug-Identität senden                                                       |
| 23  | `resolveWebSocketSessionPolicy`   | Hängt native WebSocket-Header oder Session-Cooldown-Richtlinien an                                             | Der Provider möchte, dass generische WS-Transporte Session-Header oder Fallback-Richtlinien anpassen                                       |
| 24  | `formatApiKey`                    | Auth-Profil-Formatter: gespeichertes Profil wird zur Laufzeit-Zeichenfolge `apiKey`                           | Der Provider speichert zusätzliche Auth-Metadaten und benötigt eine benutzerdefinierte Laufzeit-Tokenform                                 |
| 25  | `refreshOAuth`                    | OAuth-Refresh-Override für benutzerdefinierte Refresh-Endpunkte oder Richtlinien bei Refresh-Fehlern          | Der Provider passt nicht zu den gemeinsamen `pi-ai`-Refreshern                                                                              |
| 26  | `buildAuthDoctorHint`             | Reparaturhinweis, der angehängt wird, wenn OAuth-Refresh fehlschlägt                                           | Der Provider benötigt provider-eigene Hinweise zur Reparatur der Auth nach Refresh-Fehler                                                  |
| 27  | `matchesContextOverflowError`     | Provider-eigener Matcher für Überläufe des Kontextfensters                                                     | Der Provider hat rohe Überlauffehler, die generische Heuristiken übersehen würden                                                          |
| 28  | `classifyFailoverReason`          | Provider-eigene Klassifikation von Failover-Gründen                                                            | Der Provider kann rohe API-/Transportfehler auf Rate-Limit/Überlastung/usw. abbilden                                                       |
| 29  | `isCacheTtlEligible`              | Prompt-Cache-Richtlinie für Proxy-/Backhaul-Provider                                                           | Der Provider benötigt proxiespezifisches Cache-TTL-Gating                                                                                  |
| 30  | `buildMissingAuthMessage`         | Ersatz für die generische Recovery-Nachricht bei fehlender Auth                                                | Der Provider benötigt einen providerspezifischen Hinweis zur Wiederherstellung bei fehlender Auth                                           |
| 31  | `suppressBuiltInModel`            | Unterdrückung veralteter Upstream-Modelle plus optionaler benutzerseitiger Fehlerhinweis                      | Der Provider muss veraltete Upstream-Zeilen ausblenden oder durch einen Anbieterhinweis ersetzen                                           |
| 32  | `augmentModelCatalog`             | Synthetische/finale Katalogzeilen, die nach der Discovery angehängt werden                                     | Der Provider benötigt synthetische Zeilen für Forward-Compatibility in `models list` und Pickern                                           |
| 33  | `isBinaryThinking`                | Ein/Aus-Toggle für Reasoning bei Providern mit binärem Thinking                                                | Der Provider bietet nur binäres Thinking an/aus an                                                                                          |
| 34  | `supportsXHighThinking`           | Unterstützung für `xhigh`-Reasoning bei ausgewählten Modellen                                                  | Der Provider möchte `xhigh` nur für eine Teilmenge von Modellen                                                                             |
| 35  | `resolveDefaultThinkingLevel`     | Standard-`/think`-Level für eine bestimmte Modellfamilie                                                       | Der Provider besitzt die Standard-`/think`-Richtlinie für eine Modellfamilie                                                               |
| 36  | `isModernModelRef`                | Matcher für moderne Modelle für Live-Profilfilter und Smoke-Auswahl                                            | Der Provider besitzt das Matching bevorzugter Live-/Smoke-Modelle                                                                           |
| 37  | `prepareRuntimeAuth`              | Tauscht konfigurierte Zugangsdaten direkt vor der Inferenz in den eigentlichen Laufzeit-Token/-Schlüssel um   | Der Provider benötigt einen Tokenaustausch oder kurzlebige Anfrage-Zugangsdaten                                                            |
| 38  | `resolveUsageAuth`                | Löst Usage-/Billing-Zugangsdaten für `/usage` und verwandte Statusoberflächen auf                              | Der Provider benötigt benutzerdefiniertes Parsing von Usage-/Quota-Tokens oder andere Usage-Zugangsdaten                                   |
| 39  | `fetchUsageSnapshot`              | Ruft providerspezifische Usage-/Quota-Snapshots ab und normalisiert sie, nachdem die Auth aufgelöst wurde     | Der Provider benötigt einen providerspezifischen Usage-Endpunkt oder Payload-Parser                                                        |
| 40  | `createEmbeddingProvider`         | Baut einen provider-eigenen Embedding-Adapter für Memory/Suche                                                 | Verhalten für Memory-Embeddings gehört in das Provider-Plugin                                                                               |
| 41  | `buildReplayPolicy`               | Gibt eine Replay-Richtlinie zurück, die die Transkriptbehandlung für den Provider steuert                      | Der Provider benötigt eine benutzerdefinierte Transkript-Richtlinie (z. B. Entfernen von Thinking-Blöcken)                                |
| 42  | `sanitizeReplayHistory`           | Schreibt den Replay-Verlauf nach der generischen Bereinigung des Transkripts um                                | Der Provider benötigt providerspezifische Replay-Umschreibungen über gemeinsame Kompaktierungs-Hilfsfunktionen hinaus                     |
| 43  | `validateReplayTurns`             | Finale Validierung oder Umformung der Replay-Züge vor dem Embedded-Runner                                      | Der Provider-Transport benötigt strengere Zugvalidierung nach generischer Bereinigung                                                      |
| 44  | `onModelSelected`                 | Führt provider-eigene Side Effects nach der Modellauswahl aus                                                  | Der Provider benötigt Telemetrie oder provider-eigenen Zustand, wenn ein Modell aktiv wird                                                 |

`normalizeModelId`, `normalizeTransport` und `normalizeConfig` prüfen zuerst das
passende Provider-Plugin und fallen dann auf andere hook-fähige Provider-Plugins
zurück, bis eines die Modell-ID oder den Transport/die Konfiguration tatsächlich ändert. Dadurch bleiben
Alias-/Kompatibilitäts-Shims für Provider funktionsfähig, ohne dass der Aufrufer wissen muss, welches
gebündelte Plugin die Umschreibung besitzt. Wenn kein Provider-Hook einen unterstützten
Konfigurationseintrag der Google-Familie umschreibt, greift weiterhin die gebündelte Google-Konfigurationsnormalisierung für diese Kompatibilitätsbereinigung.

Wenn der Provider ein vollständig benutzerdefiniertes Wire-Protokoll oder einen benutzerdefinierten Anfrage-
Executor benötigt, ist das eine andere Klasse von Erweiterung. Diese Hooks sind für Provider-Verhalten gedacht,
das weiterhin auf der normalen Inferenzschleife von OpenClaw läuft.

### Provider-Beispiel

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Eingebaute Beispiele

- Anthropic verwendet `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`
  und `wrapStreamFn`, weil es Vorwärtskompatibilität für Claude 4.6,
  Hinweise zur Provider-Familie, Richtlinien zur Reparatur der Auth,
  Integration des Usage-Endpunkts,
  Eignung für Prompt-Cache, auth-bewusste Konfigurationsstandards, Standard-/adaptive
  Thinking-Richtlinien für Claude und Anthropic-spezifische Stream-Formung für
  Beta-Header, `/fast` / `serviceTier` und `context1m` besitzt.
- Anthropic-spezifische Stream-Hilfsfunktionen für Claude bleiben vorerst in der eigenen
  öffentlichen Seam `api.ts` / `contract-api.ts` des gebündelten Plugins. Diese Paketoberfläche
  exportiert `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` und die
  niedrigstufigen Builder für Anthropic-Wrapper, statt das generische SDK um die
  Beta-Header-Regeln eines einzelnen Providers zu erweitern.
- OpenAI verwendet `resolveDynamicModel`, `normalizeResolvedModel` und
  `capabilities` sowie `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` und `isModernModelRef`,
  weil es Vorwärtskompatibilität für GPT-5.4, die direkte OpenAI-
  Normalisierung `openai-completions` -> `openai-responses`, Codex-bewusste Auth-
  Hinweise, Spark-Unterdrückung, synthetische OpenAI-Katalogzeilen und GPT-5-Thinking-/
  Live-Modell-Richtlinien besitzt; die Stream-Familie `openai-responses-defaults` besitzt die
  gemeinsamen nativen OpenAI-Responses-Wrapper für Attribution-Header,
  `/fast`/`serviceTier`, Text-Verbosity, native Codex-Websuche,
  Reasoning-kompatible Payload-Formung und Kontextverwaltung für Responses.
- OpenRouter verwendet `catalog` sowie `resolveDynamicModel` und
  `prepareDynamicModel`, weil der Provider ein Pass-through ist und neue
  Modell-IDs anzeigen kann, bevor der statische Katalog von OpenClaw aktualisiert wird; außerdem verwendet es
  `capabilities`, `wrapStreamFn` und `isCacheTtlEligible`, um
  providerspezifische Anfrage-Header, Routing-Metadaten, Reasoning-Patches und
  Prompt-Cache-Richtlinien aus dem Core herauszuhalten. Seine Replay-Richtlinie stammt aus der
  Familie `passthrough-gemini`, während die Stream-Familie `openrouter-thinking`
  das Injizieren von Proxy-Reasoning und das Überspringen nicht unterstützter Modelle bzw. von `auto` besitzt.
- GitHub Copilot verwendet `catalog`, `auth`, `resolveDynamicModel` und
  `capabilities` sowie `prepareRuntimeAuth` und `fetchUsageSnapshot`,
  weil es geräteeigenen Login des Providers, Fallback-Verhalten für Modelle, Claude-Transkript-
  Eigenheiten, einen Austausch GitHub-Token -> Copilot-Token und einen provider-eigenen
  Usage-Endpunkt benötigt.
- OpenAI Codex verwendet `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` und `augmentModelCatalog` sowie
  `prepareExtraParams`, `resolveUsageAuth` und `fetchUsageSnapshot`, weil es
  weiterhin auf Core-OpenAI-Transporten läuft, aber seine Transport-/`baseUrl`-
  Normalisierung, OAuth-Refresh-Fallback-Richtlinien, Standard-Transportwahl,
  synthetische Codex-Katalogzeilen und ChatGPT-Usage-Endpunkt-Integration besitzt; es
  teilt sich dieselbe Stream-Familie `openai-responses-defaults` wie direktes OpenAI.
- Google AI Studio und Gemini CLI OAuth verwenden `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` und `isModernModelRef`, weil die
  Replay-Familie `google-gemini` Vorwärtskompatibilitäts-Fallback für Gemini 3.1,
  native Replay-Validierung für Gemini, Bereinigung von Bootstrap-Replays, einen getaggten
  Reasoning-Output-Modus und Matching moderner Modelle besitzt, während die
  Stream-Familie `google-thinking` die Normalisierung von Thinking-Payloads für Gemini besitzt;
  Gemini CLI OAuth verwendet außerdem `formatApiKey`, `resolveUsageAuth` und
  `fetchUsageSnapshot` für Token-Formatierung, Token-Parsing und Verdrahtung
  des Quota-Endpunkts.
- Anthropic Vertex verwendet `buildReplayPolicy` über die
  Replay-Familie `anthropic-by-model`, sodass Claude-spezifische Replay-Bereinigung
  auf Claude-IDs beschränkt bleibt statt auf jeden `anthropic-messages`-Transport.
- Amazon Bedrock verwendet `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` und `resolveDefaultThinkingLevel`, weil es
  Bedrock-spezifische Klassifikation von Throttle-/Not-Ready-/Kontextüberlauf-Fehlern
  für Anthropic-on-Bedrock-Verkehr besitzt; seine Replay-Richtlinie teilt weiterhin denselben
  nur-auf-Claude-bezogenen Guard `anthropic-by-model`.
- OpenRouter, Kilocode, Opencode und Opencode Go verwenden `buildReplayPolicy`
  über die Replay-Familie `passthrough-gemini`, weil sie Gemini-
  Modelle über OpenAI-kompatible Transporte proxen und eine Bereinigung von Gemini-
  Thought-Signaturen benötigen, aber keine native Gemini-Replay-Validierung oder
  Bootstrap-Umschreibungen.
- MiniMax verwendet `buildReplayPolicy` über die
  Replay-Familie `hybrid-anthropic-openai`, weil ein Provider sowohl
  Anthropic-Message- als auch OpenAI-kompatible Semantik besitzt; es hält das
  Entfernen von Thinking-Blöcken nur für Claude auf der Anthropic-Seite bei, während es den Reasoning-
  Output-Modus wieder auf nativ zurücksetzt, und die Stream-Familie `minimax-fast-mode` besitzt
  Umschreibungen von Fast-Mode-Modellen auf dem gemeinsamen Stream-Pfad.
- Moonshot verwendet `catalog` sowie `wrapStreamFn`, weil es weiterhin den
  gemeinsamen OpenAI-Transport nutzt, aber provider-eigene Normalisierung von Thinking-Payloads benötigt; die
  Stream-Familie `moonshot-thinking` bildet Konfiguration plus `/think`-Status auf ihre
  native binäre Thinking-Payload ab.
- Kilocode verwendet `catalog`, `capabilities`, `wrapStreamFn` und
  `isCacheTtlEligible`, weil es provider-eigene Anfrage-Header,
  Normalisierung von Reasoning-Payloads, Hinweise zu Gemini-Transkripten und Anthropic-
  Cache-TTL-Gating benötigt; die Stream-Familie `kilocode-thinking` hält Kilo-Thinking-
  Injektion auf dem gemeinsamen Proxy-Stream-Pfad, während `kilo/auto` und
  andere Proxy-Modell-IDs übersprungen werden, die keine expliziten Reasoning-Payloads unterstützen.
- Z.AI verwendet `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` und `fetchUsageSnapshot`, weil es Fallbacks für GLM-5,
  Standardwerte für `tool_stream`, binäre Thinking-UX, Matching moderner Modelle und sowohl
  Usage-Auth als auch Abruf von Quoten besitzt; die Stream-Familie `tool-stream-default-on` hält
  den standardmäßig aktivierten Wrapper `tool_stream` aus handgeschriebenem Glue pro Provider heraus.
- xAI verwendet `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` und `isModernModelRef`,
  weil es Normalisierung für nativen xAI-Responses-Transport, Umschreibungen von
  Aliasen für Grok Fast-Mode, Standard-`tool_stream`, Bereinigung für Strict-Tool / Reasoning-Payload,
  Wiederverwendung von Fallback-Auth für plugin-eigene Tools, Vorwärtskompatibilität bei der Auflösung von Grok-
  Modellen und provider-eigene Kompatibilitätspatches wie xAI-Tool-Schema-
  Profil, nicht unterstützte Schema-Keywords, natives `web_search` und HTML-Entity-
  Decoding von Tool-Call-Argumenten besitzt.
- Mistral, OpenCode Zen und OpenCode Go verwenden nur `capabilities`, um
  Eigenheiten von Transkript/Tooling aus dem Core herauszuhalten.
- Nur-Katalog-Provider unter den gebündelten Providern wie `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` und `volcengine` verwenden
  nur `catalog`.
- Qwen verwendet `catalog` für seinen Text-Provider sowie gemeinsame Registrierungen für Medienverständnis und
  Videogenerierung für seine multimodalen Oberflächen.
- MiniMax und Xiaomi verwenden `catalog` plus Usage-Hooks, weil ihr `/usage`-
  Verhalten plugin-eigen ist, obwohl die Inferenz weiterhin über gemeinsame Transporte läuft.

## Laufzeit-Hilfsfunktionen

Plugins können über `api.runtime` auf ausgewählte Core-Hilfsfunktionen zugreifen. Für TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Hinweise:

- `textToSpeech` gibt die normale Core-TTS-Ausgabe-Payload für Datei-/Sprachnotiz-Oberflächen zurück.
- Verwendet die Core-Konfiguration `messages.tts` und Providerauswahl.
- Gibt PCM-Audiopuffer + Sample-Rate zurück. Plugins müssen für Provider neu sampeln/kodieren.
- `listVoices` ist pro Provider optional. Verwenden Sie es für provider-eigene Voice-Picker oder Setup-Flows.
- Stimmauflistungen können reichere Metadaten wie Locale, Geschlecht und Personality-Tags für providerbewusste Picker enthalten.
- OpenAI und ElevenLabs unterstützen heute Telephony. Microsoft nicht.

Plugins können auch Sprach-Provider über `api.registerSpeechProvider(...)` registrieren.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Hinweise:

- Behalten Sie TTS-Richtlinie, Fallback und Antwortzustellung im Core.
- Verwenden Sie Sprach-Provider für anbietereigene Synthese.
- Ältere Microsoft-Eingaben `edge` werden auf die Provider-ID `microsoft` normalisiert.
- Das bevorzugte Eigentumsmodell ist unternehmensorientiert: Ein Anbieter-Plugin kann
  Text-, Sprach-, Bild- und künftige Medien-Provider besitzen, wenn OpenClaw diese
  Fähigkeitsverträge hinzufügt.

Für Bild-/Audio-/Videoverständnis registrieren Plugins einen typisierten
Provider für Medienverständnis statt einer generischen Schlüssel/Wert-Tasche:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Hinweise:

- Behalten Sie Orchestrierung, Fallback, Konfiguration und Kanalverdrahtung im Core.
- Behalten Sie Anbieterverhalten im Provider-Plugin.
- Additive Erweiterung sollte typisiert bleiben: neue optionale Methoden, neue optionale
  Ergebnisfelder, neue optionale Fähigkeiten.
- Videogenerierung folgt bereits demselben Muster:
  - der Core besitzt den Fähigkeitsvertrag und die Laufzeit-Hilfsfunktion
  - Anbieter-Plugins registrieren `api.registerVideoGenerationProvider(...)`
  - Feature-/Kanal-Plugins verwenden `api.runtime.videoGeneration.*`

Für Laufzeit-Hilfsfunktionen des Medienverständnisses können Plugins Folgendes aufrufen:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

Für Audiotranskription können Plugins entweder die Laufzeit des Medienverständnisses
oder das ältere STT-Alias verwenden:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Hinweise:

- `api.runtime.mediaUnderstanding.*` ist die bevorzugte gemeinsame Oberfläche für
  Bild-/Audio-/Videoverständnis.
- Verwendet die Audio-Konfiguration des Core für Medienverständnis (`tools.media.audio`) und die Provider-Fallback-Reihenfolge.
- Gibt `{ text: undefined }` zurück, wenn keine Transkriptionsausgabe erzeugt wird (zum Beispiel bei übersprungenen/nicht unterstützten Eingaben).
- `api.runtime.stt.transcribeAudioFile(...)` bleibt als Kompatibilitäts-Alias erhalten.

Plugins können auch Hintergrundläufe von Subagents über `api.runtime.subagent` starten:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Hinweise:

- `provider` und `model` sind optionale Überschreibungen pro Lauf, keine persistenten Sitzungsänderungen.
- OpenClaw berücksichtigt diese Override-Felder nur für vertrauenswürdige Aufrufer.
- Für plugin-eigene Fallback-Läufe müssen Betreiber mit `plugins.entries.<id>.subagent.allowModelOverride: true` zustimmen.
- Verwenden Sie `plugins.entries.<id>.subagent.allowedModels`, um vertrauenswürdige Plugins auf bestimmte kanonische Ziele `provider/model` zu beschränken, oder `"*"`, um explizit jedes Ziel zuzulassen.
- Läufe von Subagents aus nicht vertrauenswürdigen Plugins funktionieren weiterhin, aber Override-Anfragen werden abgelehnt, statt still auf Fallback zurückzufallen.

Für Websuche können Plugins die gemeinsame Laufzeit-Hilfsfunktion nutzen, statt
in die Verdrahtung des Agent-Tools zu greifen:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Plugins können Websuch-Provider auch über
`api.registerWebSearchProvider(...)` registrieren.

Hinweise:

- Behalten Sie Providerauswahl, Auflösung von Zugangsdaten und gemeinsame Anfrage-Semantik im Core.
- Verwenden Sie Websuch-Provider für anbieterspezifische Suchtransporte.
- `api.runtime.webSearch.*` ist die bevorzugte gemeinsame Oberfläche für Feature-/Kanal-Plugins, die Suchverhalten benötigen, ohne vom Wrapper des Agent-Tools abhängig zu sein.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: Erzeugt ein Bild mit der konfigurierten Provider-Kette für Bildgenerierung.
- `listProviders(...)`: Listet verfügbare Provider für Bildgenerierung und ihre Fähigkeiten auf.

## Gateway-HTTP-Routen

Plugins können HTTP-Endpunkte mit `api.registerHttpRoute(...)` bereitstellen.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Routenfelder:

- `path`: Routenpfad unter dem HTTP-Server des Gateway.
- `auth`: erforderlich. Verwenden Sie `"gateway"`, um normale Gateway-Auth zu verlangen, oder `"plugin"` für plugin-verwaltete Auth/Webhook-Verifikation.
- `match`: optional. `"exact"` (Standard) oder `"prefix"`.
- `replaceExisting`: optional. Erlaubt demselben Plugin, seine eigene vorhandene Routenregistrierung zu ersetzen.
- `handler`: gibt `true` zurück, wenn die Route die Anfrage verarbeitet hat.

Hinweise:

- `api.registerHttpHandler(...)` wurde entfernt und führt zu einem Plugin-Ladefehler. Verwenden Sie stattdessen `api.registerHttpRoute(...)`.
- Plugin-Routen müssen `auth` explizit deklarieren.
- Exakte Konflikte bei `path + match` werden abgelehnt, es sei denn `replaceExisting: true`, und ein Plugin kann die Route eines anderen Plugins nicht ersetzen.
- Überlappende Routen mit unterschiedlichen `auth`-Ebenen werden abgelehnt. Halten Sie Fallthrough-Ketten von `exact`/`prefix` nur auf derselben Auth-Ebene.
- Routen mit `auth: "plugin"` erhalten **nicht** automatisch Runtime-Scopes des Operators. Sie sind für plugin-verwaltete Webhooks/Signaturverifikation gedacht, nicht für privilegierte Gateway-Hilfsaufrufe.
- Routen mit `auth: "gateway"` laufen innerhalb eines Runtime-Scopes für Gateway-Anfragen, aber dieser Scope ist bewusst konservativ:
  - Bearer-Auth mit gemeinsamem Secret (`gateway.auth.mode = "token"` / `"password"`) hält Runtime-Scopes für Plugin-Routen auf `operator.write` fest, auch wenn der Aufrufer `x-openclaw-scopes` sendet
  - vertrauenswürdige HTTP-Modi mit Identität (zum Beispiel `trusted-proxy` oder `gateway.auth.mode = "none"` an einem privaten Ingress) berücksichtigen `x-openclaw-scopes` nur, wenn der Header explizit vorhanden ist
  - wenn `x-openclaw-scopes` bei solchen Plugin-Routen mit Identität fehlt, fällt der Runtime-Scope auf `operator.write` zurück
- Praktische Regel: Gehen Sie nicht davon aus, dass eine Plugin-Route mit Gateway-Auth implizit eine Admin-Oberfläche ist. Wenn Ihre Route Verhalten nur für Admins benötigt, verlangen Sie einen Auth-Modus mit Identität und dokumentieren Sie den expliziten Header-Vertrag `x-openclaw-scopes`.

## Importpfade des Plugin-SDK

Verwenden Sie SDK-Subpaths statt des monolithischen Imports `openclaw/plugin-sdk`,
wenn Sie Plugins verfassen:

- `openclaw/plugin-sdk/plugin-entry` für Primitive zur Plugin-Registrierung.
- `openclaw/plugin-sdk/core` für den generischen gemeinsamen pluginseitigen Vertrag.
- `openclaw/plugin-sdk/config-schema` für den Export des Zod-Schemas der Wurzel `openclaw.json`
  (`OpenClawSchema`).
- Stabile Kanalprimitive wie `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input` und
  `openclaw/plugin-sdk/webhook-ingress` für gemeinsame Verdrahtung von Setup/Auth/Antwort/Webhook.
  `channel-inbound` ist das gemeinsame Zuhause für Debounce, Mention-Matching,
  Hilfsfunktionen für Mention-Richtlinien eingehender Nachrichten, Envelope-Formatierung und Hilfsfunktionen
  für den Kontext eingehender Envelopes.
  `channel-setup` ist die schmale Setup-Seam für optionale Installation.
  `setup-runtime` ist die laufzeitsichere Setup-Oberfläche, die von `setupEntry` /
  verzögertem Start verwendet wird, einschließlich import-sicherer Patch-Adapter für das Setup.
  `setup-adapter-runtime` ist die env-bewusste Adapter-Seam für Konto-Setup.
  `setup-tools` ist die kleine Seam für Hilfsfunktionen zu CLI/Archiven/Dokumentation (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Domänen-Subpaths wie `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store` und
  `openclaw/plugin-sdk/directory-runtime` für gemeinsame Hilfsfunktionen für Laufzeit/Konfiguration.
  `telegram-command-config` ist die schmale öffentliche Seam für Normalisierung/Validierung benutzerdefinierter Telegram-
  Befehle und bleibt verfügbar, selbst wenn die gebündelte Telegram-Vertragsoberfläche vorübergehend nicht verfügbar ist.
  `text-runtime` ist die gemeinsame Seam für Text/Markdown/Logging, einschließlich
  Entfernen für Assistenten sichtbaren Texts, Hilfsfunktionen für Rendering/Chunking von Markdown, Hilfsfunktionen
  für Redaction, Hilfsfunktionen für Direktiven-Tags und Safe-Text-Utilities.
- Genehmigungsspezifische Kanal-Seams sollten einen einzelnen Vertrag `approvalCapability`
  auf dem Plugin bevorzugen. Der Core liest dann Auth, Zustellung, Rendering,
  natives Routing und lazy native Handler-Verhalten für Genehmigungen über diese eine Fähigkeit,
  statt Genehmigungsverhalten in nicht zusammenhängende Plugin-Felder zu mischen.
- `openclaw/plugin-sdk/channel-runtime` ist veraltet und bleibt nur als
  Kompatibilitäts-Shim für ältere Plugins erhalten. Neuer Code sollte stattdessen die schmaleren
  generischen Primitive importieren, und im Repo sollte kein neuer Import des
  Shims hinzugefügt werden.
- Interne Bestandteile gebündelter Erweiterungen bleiben privat. Externe Plugins sollten nur
  `openclaw/plugin-sdk/*`-Subpaths verwenden. OpenClaw-Core-/Test-Code kann die öffentlichen
  Repo-Entry-Points unter einer Plugin-Paketwurzel verwenden, etwa `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` und eng begrenzte Dateien wie
  `login-qr-api.js`. Importieren Sie niemals `src/*` eines Plugin-Pakets aus dem Core oder aus
  einer anderen Erweiterung.
- Aufteilung der Repo-Entry-Points:
  `<plugin-package-root>/api.js` ist das Helper-/Types-Barrel,
  `<plugin-package-root>/runtime-api.js` ist das reine Laufzeit-Barrel,
  `<plugin-package-root>/index.js` ist der Entry des gebündelten Plugins
  und `<plugin-package-root>/setup-entry.js` ist der Setup-Entry des Plugins.
- Aktuelle Beispiele für gebündelte Provider:
  - Anthropic verwendet `api.js` / `contract-api.js` für Claude-Stream-Hilfsfunktionen wie
    `wrapAnthropicProviderStream`, Hilfsfunktionen für Beta-Header und Parsing von `service_tier`.
  - OpenAI verwendet `api.js` für Provider-Builder, Hilfsfunktionen für Standardmodelle und Builder für Echtzeit-Provider.
  - OpenRouter verwendet `api.js` für seinen Provider-Builder sowie Hilfsfunktionen für Onboarding/Konfiguration,
    während `register.runtime.js` weiterhin generische
    `plugin-sdk/provider-stream`-Hilfsfunktionen für repo-lokale Verwendung re-exportieren kann.
- Öffentlich zugängliche Entry-Points, die über Fassaden geladen werden, bevorzugen den aktiven Laufzeit-Snapshot der Konfiguration,
  wenn ein solcher existiert, und fallen andernfalls auf die auf dem Datenträger aufgelöste Konfigurationsdatei zurück, wenn
  OpenClaw noch keinen Laufzeit-Snapshot bereitstellt.
- Generische gemeinsame Primitive bleiben der bevorzugte öffentliche SDK-Vertrag. Ein kleiner
  reservierter Kompatibilitätssatz gebündelter, kanalgebrandeter Helper-Seams existiert weiterhin.
  Behandeln Sie diese als Seams für Wartung/Kompatibilität gebündelter Plugins, nicht als neue Importziele für Dritte; neue kanalübergreifende Verträge sollten weiterhin auf generischen `plugin-sdk/*`-Subpaths oder den plugin-lokalen Barrels `api.js` /
  `runtime-api.js` landen.

Hinweis zur Kompatibilität:

- Vermeiden Sie für neuen Code das Root-Barrel `openclaw/plugin-sdk`.
- Bevorzugen Sie zuerst die schmalen stabilen Primitive. Die neueren Setup-/Pairing-/Reply-/
  Feedback-/Contract-/Inbound-/Threading-/Command-/Secret-Input-/Webhook-/Infra-/
  Allowlist-/Status-/Message-Tool-Subpaths sind der beabsichtigte Vertrag für neue
  Arbeit an gebündelten und externen Plugins.
  Parsen/Matching von Zielen gehört zu `openclaw/plugin-sdk/channel-targets`.
  Gates für Message-Aktionen und Hilfsfunktionen für Reaktions-Message-IDs gehören zu
  `openclaw/plugin-sdk/channel-actions`.
- Erweiterungsspezifische Helper-Barrels gebündelter Erweiterungen sind standardmäßig nicht stabil. Wenn ein
  Helper nur von einer gebündelten Erweiterung benötigt wird, halten Sie ihn hinter der lokalen
  Seam `api.js` oder `runtime-api.js` dieser Erweiterung, statt ihn in
  `openclaw/plugin-sdk/<extension>` zu befördern.
- Neue gemeinsame Helper-Seams sollten generisch sein, nicht kanalgebrandet. Gemeinsames Parsen
  von Zielen gehört zu `openclaw/plugin-sdk/channel-targets`; kanalspezifische
  Interna bleiben hinter der lokalen Seam `api.js` oder `runtime-api.js` des besitzenden Plugins.
- Fähigkeitsspezifische Subpaths wie `image-generation`,
  `media-understanding` und `speech` existieren, weil gebündelte/native Plugins sie heute nutzen.
  Ihre Existenz bedeutet nicht automatisch, dass jede exportierte Hilfsfunktion ein
  langfristig eingefrorener externer Vertrag ist.

## Schemas für Message-Tools

Plugins sollten kanalspezifische Schema-Beiträge in `describeMessageTool(...)`
besitzen. Behalten Sie providerspezifische Felder im Plugin, nicht im gemeinsamen Core.

Für gemeinsame portable Schemafragmente verwenden Sie die generischen Hilfsfunktionen, die über
`openclaw/plugin-sdk/channel-actions` exportiert werden:

- `createMessageToolButtonsSchema()` für Payloads im Stil von Button-Rastern
- `createMessageToolCardSchema()` für strukturierte Card-Payloads

Wenn eine Schemaform nur für einen Provider sinnvoll ist, definieren Sie sie in den
eigenen Quellen dieses Plugins, statt sie in das gemeinsame SDK zu befördern.

## Auflösung von Kanalzielen

Kanal-Plugins sollten kanalspezifische Zielsemantik besitzen. Halten Sie den gemeinsamen
Outbound-Host generisch und verwenden Sie die Oberfläche des Messaging-Adapters für Provider-Regeln:

- `messaging.inferTargetChatType({ to })` entscheidet, ob ein normalisiertes Ziel
  vor dem Directory-Lookup als `direct`, `group` oder `channel` behandelt werden soll.
- `messaging.targetResolver.looksLikeId(raw, normalized)` teilt dem Core mit, ob eine
  Eingabe direkt zur ID-artigen Auflösung springen soll statt zur Directory-Suche.
- `messaging.targetResolver.resolveTarget(...)` ist der Plugin-Fallback, wenn
  der Core nach der Normalisierung oder nach einem Directory-Fehlschlag eine letzte provider-eigene Auflösung benötigt.
- `messaging.resolveOutboundSessionRoute(...)` besitzt die providerspezifische Konstruktion der Sitzungsroute,
  sobald ein Ziel aufgelöst ist.

Empfohlene Aufteilung:

- Verwenden Sie `inferTargetChatType` für Kategorieentscheidungen, die vor
  der Suche nach Peers/Gruppen erfolgen sollten.
- Verwenden Sie `looksLikeId` für Prüfungen wie „dies als explizite/native Ziel-ID behandeln“.
- Verwenden Sie `resolveTarget` für provider-eigenen Normalisierungs-Fallback, nicht für
  breit angelegte Directory-Suche.
- Halten Sie provider-native IDs wie Chat-IDs, Thread-IDs, JIDs, Handles und Raum-IDs
  in `target`-Werten oder providerspezifischen Parametern, nicht in generischen SDK-Feldern.

## Konfigurationsgestützte Directories

Plugins, die Directory-Einträge aus der Konfiguration ableiten, sollten diese Logik im
Plugin behalten und die gemeinsamen Hilfsfunktionen aus
`openclaw/plugin-sdk/directory-runtime` wiederverwenden.

Verwenden Sie dies, wenn ein Kanal konfigurationsgestützte Peers/Gruppen benötigt, etwa:

- Allowlist-gesteuerte DM-Peers
- konfigurierte Kanal-/Gruppen-Zuordnungen
- kontoabhängige statische Directory-Fallbacks

Die gemeinsamen Hilfsfunktionen in `directory-runtime` behandeln nur generische Operationen:

- Query-Filterung
- Anwenden von Limits
- Hilfsfunktionen für Deduplizierung/Normalisierung
- Erstellen von `ChannelDirectoryEntry[]`

Kanalspezifische Kontoinspektion und ID-Normalisierung sollten in der
Plugin-Implementierung bleiben.

## Provider-Kataloge

Provider-Plugins können Modellkataloge für Inferenz definieren mit
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` gibt dieselbe Form zurück, die OpenClaw in
`models.providers` schreibt:

- `{ provider }` für einen Provider-Eintrag
- `{ providers }` für mehrere Provider-Einträge

Verwenden Sie `catalog`, wenn das Plugin providerspezifische Modell-IDs, Standardwerte für `baseUrl`
oder auth-gesteuerte Modellmetadaten besitzt.

`catalog.order` steuert, wann der Katalog eines Plugins relativ zu den eingebauten impliziten Providern von OpenClaw zusammengeführt wird:

- `simple`: einfache API-Key- oder env-gesteuerte Provider
- `profile`: Provider, die erscheinen, wenn Auth-Profile existieren
- `paired`: Provider, die mehrere zusammengehörige Provider-Einträge synthetisieren
- `late`: letzter Durchlauf, nach anderen impliziten Providern

Spätere Provider gewinnen bei Schlüsselkollisionen, sodass Plugins absichtlich einen eingebauten Provider-
Eintrag mit derselben Provider-ID überschreiben können.

Kompatibilität:

- `discovery` funktioniert weiterhin als älterer Alias
- wenn sowohl `catalog` als auch `discovery` registriert sind, verwendet OpenClaw `catalog`

## Read-only-Kanalinspektion

Wenn Ihr Plugin einen Kanal registriert, sollten Sie
`plugin.config.inspectAccount(cfg, accountId)` parallel zu `resolveAccount(...)` implementieren.

Warum:

- `resolveAccount(...)` ist der Laufzeitpfad. Er darf davon ausgehen, dass Zugangsdaten
  vollständig materialisiert sind, und bei fehlenden erforderlichen Secrets schnell fehlschlagen.
- Read-only-Befehlspfade wie `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` und Doctor-/Konfigurations-
  Reparaturflüsse sollten Laufzeit-Zugangsdaten nicht materialisieren müssen, nur um Konfiguration zu beschreiben.

Empfohlenes Verhalten für `inspectAccount(...)`:

- Nur beschreibenden Kontostatus zurückgeben.
- `enabled` und `configured` beibehalten.
- Felder zu Quelle/Status von Zugangsdaten einbeziehen, wenn relevant, etwa:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Sie müssen keine rohen Tokenwerte zurückgeben, nur um Read-only-
  Verfügbarkeit zu melden. `tokenStatus: "available"` (und das passende Quellfeld) reicht für Statusbefehle aus.
- Verwenden Sie `configured_unavailable`, wenn Zugangsdaten über SecretRef konfiguriert sind, aber
  im aktuellen Befehlspfad nicht verfügbar sind.

Dadurch können Read-only-Befehle „konfiguriert, aber in diesem Befehlspfad nicht verfügbar“ melden,
statt abzustürzen oder das Konto fälschlich als nicht konfiguriert zu melden.

## Package-Packs

Ein Plugin-Verzeichnis kann eine `package.json` mit `openclaw.extensions` enthalten:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Jeder Eintrag wird zu einem Plugin. Wenn das Pack mehrere Erweiterungen auflistet, wird die Plugin-ID zu
`name/<fileBase>`.

Wenn Ihr Plugin npm-Abhängigkeiten importiert, installieren Sie sie in diesem Verzeichnis, damit
`node_modules` verfügbar ist (`npm install` / `pnpm install`).

Sicherheitsleitplanke: Jeder Eintrag in `openclaw.extensions` muss nach der Auflösung von Symlinks innerhalb des Plugin-
Verzeichnisses bleiben. Einträge, die aus dem Paketverzeichnis herausführen, werden
abgelehnt.

Sicherheitshinweis: `openclaw plugins install` installiert Plugin-Abhängigkeiten mit
`npm install --omit=dev --ignore-scripts` (keine Lifecycle-Skripte, keine Dev-Abhängigkeiten zur Laufzeit). Halten Sie Plugin-Abhängigkeits-
bäume „reines JS/TS“ und vermeiden Sie Pakete, die `postinstall`-Builds erfordern.

Optional: `openclaw.setupEntry` kann auf ein leichtgewichtiges, nur für Setup bestimmtes Modul verweisen.
Wenn OpenClaw Setup-Oberflächen für ein deaktiviertes Kanal-Plugin benötigt oder
wenn ein Kanal-Plugin aktiviert, aber noch nicht konfiguriert ist, lädt es `setupEntry`
statt des vollständigen Plugin-Entries. Dadurch bleiben Start und Setup leichter,
wenn Ihr Haupteintrag zusätzlich Tools, Hooks oder anderen reinen Laufzeit-
Code verdrahtet.

Optional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
kann ein Kanal-Plugin in denselben `setupEntry`-Pfad während der Pre-Listen-Startphase des Gateway aufnehmen, auch wenn der Kanal bereits konfiguriert ist.

Verwenden Sie dies nur, wenn `setupEntry` die Startoberfläche, die vor dem Lauschen
des Gateway existieren muss, vollständig abdeckt. In der Praxis bedeutet das, dass der
Setup-Entry jede kanaleigene Fähigkeit registrieren muss, von der der Start abhängt, z. B.:

- die Kanalregistrierung selbst
- alle HTTP-Routen, die verfügbar sein müssen, bevor das Gateway mit dem Lauschen beginnt
- alle Gateway-Methoden, Tools oder Dienste, die in diesem Zeitfenster existieren müssen

Wenn Ihr vollständiger Entry weiterhin eine erforderliche Startfähigkeit besitzt, aktivieren Sie
dieses Flag nicht. Behalten Sie für das Plugin das Standardverhalten bei und lassen Sie OpenClaw beim Start den
vollständigen Entry laden.

Gebündelte Kanäle können außerdem Hilfsfunktionen mit reiner Setup-Vertragsoberfläche veröffentlichen, die der Core
abfragen kann, bevor die vollständige Kanallaufzeit geladen ist. Die aktuelle Setup-
Promotion-Oberfläche ist:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Der Core verwendet diese Oberfläche, wenn er eine ältere Einzelkonto-Kanal-
Konfiguration in `channels.<id>.accounts.*` hochstufen muss, ohne den vollständigen Plugin-Entry zu laden.
Matrix ist das aktuelle gebündelte Beispiel: Es verschiebt nur Auth-/Bootstrap-Schlüssel in ein
benanntes hochgestuftes Konto, wenn bereits benannte Konten existieren, und es kann ein
konfiguriertes nicht-kanonisches Standardkonto beibehalten, statt immer
`accounts.default` zu erstellen.

Diese Setup-Patch-Adapter halten die Discovery der gebündelten Vertragsoberfläche lazy. Die Importzeit bleibt gering;
die Promotion-Oberfläche wird nur bei der ersten Verwendung geladen, statt beim Modulimport den Start des gebündelten Kanals erneut zu betreten.

Wenn diese Startoberflächen Gateway-RPC-Methoden enthalten, halten Sie sie auf einem
pluginspezifischen Präfix. Core-Admin-Namespaces (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) bleiben reserviert und werden immer
zu `operator.admin` aufgelöst, selbst wenn ein Plugin einen schmaleren Scope anfordert.

Beispiel:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Kanal-Katalogmetadaten

Kanal-Plugins können Setup-/Discovery-Metadaten über `openclaw.channel` und
Installationshinweise über `openclaw.install` bekannt machen. Dadurch bleibt der Core-Katalog frei von Daten.

Beispiel:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Nützliche Felder in `openclaw.channel` über das Minimalbeispiel hinaus:

- `detailLabel`: sekundäres Label für reichhaltigere Katalog-/Statusoberflächen
- `docsLabel`: überschreibt den Linktext für den Doku-Link
- `preferOver`: Plugin-/Kanal-IDs niedrigerer Priorität, die dieser Katalogeintrag übertreffen soll
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: Copy-Steuerung für Auswahloberflächen
- `markdownCapable`: markiert den Kanal für Entscheidungen zur Outbound-Formatierung als markdownfähig
- `exposure.configured`: blendet den Kanal aus Oberflächen mit konfigurierten Kanälen aus, wenn auf `false` gesetzt
- `exposure.setup`: blendet den Kanal aus interaktiven Setup-/Konfigurations-Pickern aus, wenn auf `false` gesetzt
- `exposure.docs`: markiert den Kanal für Dokumentationsnavigationsoberflächen als intern/privat
- `showConfigured` / `showInSetup`: ältere Aliasse werden aus Kompatibilitätsgründen weiterhin akzeptiert; `exposure` bevorzugen
- `quickstartAllowFrom`: nimmt den Kanal in den Standard-Quickstart-Flow `allowFrom` auf
- `forceAccountBinding`: verlangt explizite Kontobindung, selbst wenn nur ein Konto existiert
- `preferSessionLookupForAnnounceTarget`: bevorzugt Sitzungs-Lookup bei der Auflösung von Ankündigungszielen

OpenClaw kann auch **externe Kanal-Kataloge** zusammenführen (zum Beispiel einen MPM-
Registry-Export). Legen Sie eine JSON-Datei an einer der folgenden Stellen ab:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Oder zeigen Sie `OPENCLAW_PLUGIN_CATALOG_PATHS` (oder `OPENCLAW_MPM_CATALOG_PATHS`) auf
eine oder mehrere JSON-Dateien (durch Komma/Semikolon/`PATH` getrennt). Jede Datei sollte
`{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` enthalten. Der Parser akzeptiert außerdem `"packages"` oder `"plugins"` als ältere Aliasse für den Schlüssel `"entries"`.

## Plugins für Context Engines

Plugins für Context Engines besitzen die Orchestrierung des Sitzungs-
kontexts für Ingest, Assemblierung und Kompaktierung. Registrieren Sie sie in Ihrem Plugin mit
`api.registerContextEngine(id, factory)` und wählen Sie dann die aktive Engine mit
`plugins.slots.contextEngine`.

Verwenden Sie dies, wenn Ihr Plugin die Standard-
Kontextpipeline ersetzen oder erweitern muss, statt nur Memory-Suche oder Hooks hinzuzufügen.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Wenn Ihre Engine den Kompaktierungsalgorithmus **nicht** besitzt, halten Sie `compact()`
implementiert und delegieren Sie explizit:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Eine neue Fähigkeit hinzufügen

Wenn ein Plugin Verhalten benötigt, das nicht zur aktuellen API passt, umgehen Sie
das Plugin-System nicht mit einem privaten Eingriff. Fügen Sie die fehlende Fähigkeit hinzu.

Empfohlene Reihenfolge:

1. den Core-Vertrag definieren
   Entscheiden Sie, welches gemeinsame Verhalten der Core besitzen sollte: Richtlinie, Fallback, Konfigurations-
   Merge, Lebenszyklus, kanalorientierte Semantik und Form der Laufzeit-Hilfsfunktion.
2. typisierte Plugin-Registrierungs-/Laufzeitoberflächen hinzufügen
   Erweitern Sie `OpenClawPluginApi` und/oder `api.runtime` um die kleinste nützliche
   typisierte Fähigkeitsoberfläche.
3. Core + Kanal-/Feature-Konsumenten verdrahten
   Kanäle und Feature-Plugins sollten die neue Fähigkeit über den Core nutzen,
   nicht durch direktes Importieren einer Anbieterimplementierung.
4. Anbieterimplementierungen registrieren
   Anbieter-Plugins registrieren dann ihre Backends für diese Fähigkeit.
5. Vertragsabdeckung hinzufügen
   Fügen Sie Tests hinzu, damit Eigentum und Form der Registrierung über die Zeit explizit bleiben.

So bleibt OpenClaw meinungsstark, ohne an die Sichtweise eines
einzigen Providers hart gebunden zu werden. Siehe das [Capability Cookbook](/de/plugins/architecture)
für eine konkrete Dateicheckliste und ein ausgearbeitetes Beispiel.

### Checkliste für Fähigkeiten

Wenn Sie eine neue Fähigkeit hinzufügen, sollte die Implementierung normalerweise
diese Oberflächen gemeinsam berühren:

- Core-Vertragstypen in `src/<capability>/types.ts`
- Core-Runner/Laufzeit-Hilfsfunktion in `src/<capability>/runtime.ts`
- Plugin-API-Registrierungsoberfläche in `src/plugins/types.ts`
- Verdrahtung der Plugin-Registry in `src/plugins/registry.ts`
- Laufzeit-Exposition von Plugins in `src/plugins/runtime/*`, wenn Feature-/Kanal-
  Plugins sie nutzen müssen
- Capture-/Test-Hilfsfunktionen in `src/test-utils/plugin-registration.ts`
- Assertions zu Eigentum/Verträgen in `src/plugins/contracts/registry.ts`
- Operator-/Plugin-Dokumentation in `docs/`

Wenn eine dieser Oberflächen fehlt, ist das normalerweise ein Zeichen dafür, dass die Fähigkeit
noch nicht vollständig integriert ist.

### Vorlage für Fähigkeiten

Minimales Muster:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Muster für Vertragstests:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Damit bleibt die Regel einfach:

- der Core besitzt den Fähigkeitsvertrag + die Orchestrierung
- Anbieter-Plugins besitzen Anbieterimplementierungen
- Feature-/Kanal-Plugins nutzen Laufzeit-Hilfsfunktionen
- Vertragstests halten Eigentum explizit
