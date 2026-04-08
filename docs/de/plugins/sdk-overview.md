---
read_when:
    - Sie müssen wissen, aus welchem SDK-Unterpfad Sie importieren sollen
    - Sie möchten eine Referenz für alle Registrierungsmethoden in OpenClawPluginApi
    - Sie schlagen einen bestimmten SDK-Export nach
sidebarTitle: SDK Overview
summary: Import-Map, Referenz der Registrierungs-API und SDK-Architektur
title: Überblick über das Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:18:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: c5a41bd82d165dfbb7fbd6e4528cf322e9133a51efe55fa8518a7a0a626d9d30
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Überblick über das Plugin SDK

Das Plugin SDK ist der typisierte Vertrag zwischen Plugins und dem Core. Diese Seite ist die
Referenz für **was importiert werden soll** und **was Sie registrieren können**.

<Tip>
  **Suchen Sie nach einer How-to-Anleitung?**
  - Ihr erstes Plugin? Beginnen Sie mit [Erste Schritte](/de/plugins/building-plugins)
  - Channel-Plugin? Siehe [Channel Plugins](/de/plugins/sdk-channel-plugins)
  - Provider-Plugin? Siehe [Provider Plugins](/de/plugins/sdk-provider-plugins)
</Tip>

## Importkonvention

Importieren Sie immer aus einem spezifischen Unterpfad:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Jeder Unterpfad ist ein kleines, in sich abgeschlossenes Modul. Das hält den Start schnell und
verhindert Probleme mit zirkulären Abhängigkeiten. Für channelspezifische Entry-/Build-Helper
bevorzugen Sie `openclaw/plugin-sdk/channel-core`; verwenden Sie `openclaw/plugin-sdk/core` für
die breitere Dachoberfläche und gemeinsame Helper wie
`buildChannelConfigSchema`.

Fügen Sie keine nach Providern benannten Convenience-Seams hinzu und hängen Sie nicht von ihnen ab, wie
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` oder
channelgebrandete Helper-Seams. Gebündelte Plugins sollten generische
SDK-Unterpfade innerhalb ihrer eigenen `api.ts`- oder `runtime-api.ts`-Barrels zusammensetzen, und der Core
sollte entweder diese pluginlokalen Barrels verwenden oder einen schmalen generischen SDK-
Vertrag hinzufügen, wenn der Bedarf wirklich channelübergreifend ist.

Die generierte Export-Map enthält weiterhin eine kleine Menge an Helper-
Seams für gebündelte Plugins wie `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` und `plugin-sdk/matrix*`. Diese
Unterpfade existieren nur für Wartung und Kompatibilität gebündelter Plugins; sie werden
absichtlich aus der unten stehenden allgemeinen Tabelle weggelassen und sind nicht der empfohlene
Importpfad für neue Drittanbieter-Plugins.

## Unterpfad-Referenz

Die am häufigsten verwendeten Unterpfade, nach Zweck gruppiert. Die generierte vollständige Liste von
mehr als 200 Unterpfaden befindet sich in `scripts/lib/plugin-sdk-entrypoints.json`.

Reservierte Helper-Unterpfade für gebündelte Plugins erscheinen weiterhin in dieser generierten Liste.
Behandeln Sie diese als Implementierungsdetail-/Kompatibilitätsoberflächen, sofern eine Dokumentationsseite
nicht ausdrücklich einen davon als öffentlich empfiehlt.

### Plugin-Einstiegspunkt

| Subpath                     | Key exports                                                                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Channel-Unterpfade">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Root-`openclaw.json`-Zod-Schema-Export (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard` sowie `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Gemeinsame Helper für Setup-Wizards, Allowlist-Prompts und Setup-Status-Builder |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helper für Multi-Account-Konfiguration/Aktions-Gates und Fallbacks für Standardkonten |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, Helper zur Normalisierung von Account-IDs |
    | `plugin-sdk/account-resolution` | Account-Lookup- und Standard-Fallback-Helper |
    | `plugin-sdk/account-helpers` | Schmale Helper für Account-Listen/Account-Aktionen |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Channel-Konfigurationsschematypen |
    | `plugin-sdk/telegram-command-config` | Helper zur Normalisierung/Validierung benutzerdefinierter Telegram-Befehle mit Fallback für gebündelte Verträge |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Gemeinsame Helper für eingehende Routen und Envelope-Builder |
    | `plugin-sdk/inbound-reply-dispatch` | Gemeinsame Helper zum Erfassen und Dispatchen eingehender Nachrichten |
    | `plugin-sdk/messaging-targets` | Helper zum Parsen/Abgleichen von Zielen |
    | `plugin-sdk/outbound-media` | Gemeinsame Helper zum Laden ausgehender Medien |
    | `plugin-sdk/outbound-runtime` | Helper für ausgehende Identität/Sende-Delegates |
    | `plugin-sdk/thread-bindings-runtime` | Lifecycle- und Adapter-Helper für Thread-Bindings |
    | `plugin-sdk/agent-media-payload` | Legacy-Builder für Agent-Media-Payloads |
    | `plugin-sdk/conversation-runtime` | Helper für Konversations-/Thread-Bindings, Pairing und konfigurierte Bindings |
    | `plugin-sdk/runtime-config-snapshot` | Helper für Runtime-Konfigurations-Snapshots |
    | `plugin-sdk/runtime-group-policy` | Helper zur Auflösung von Runtime-Gruppenrichtlinien |
    | `plugin-sdk/channel-status` | Gemeinsame Helper für Snapshots/Zusammenfassungen des Channel-Status |
    | `plugin-sdk/channel-config-primitives` | Schmale Primitive für Channel-Konfigurationsschemas |
    | `plugin-sdk/channel-config-writes` | Helper zur Autorisierung von Channel-Konfigurationsschreibvorgängen |
    | `plugin-sdk/channel-plugin-common` | Gemeinsame Prelude-Exporte für Channel-Plugins |
    | `plugin-sdk/allowlist-config-edit` | Helper zum Lesen/Bearbeiten der Allowlist-Konfiguration |
    | `plugin-sdk/group-access` | Gemeinsame Helper für Gruppen-Zugriffsentscheidungen |
    | `plugin-sdk/direct-dm` | Gemeinsame Helper für direkte-DM-Auth/Guards |
    | `plugin-sdk/interactive-runtime` | Helper zur Normalisierung/Reduktion interaktiver Antwort-Payloads |
    | `plugin-sdk/channel-inbound` | Helper für Inbound-Debounce, Mention-Matching, Mention-Policy und Envelopes |
    | `plugin-sdk/channel-send-result` | Antwort-Ergebnistypen |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helper zum Parsen/Abgleichen von Zielen |
    | `plugin-sdk/channel-contract` | Channel-Vertragstypen |
    | `plugin-sdk/channel-feedback` | Feedback-/Reaktions-Verdrahtung |
    | `plugin-sdk/channel-secret-runtime` | Schmale Helper für Secret-Verträge wie `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` und Secret-Zieltypen |
  </Accordion>

  <Accordion title="Provider-Unterpfade">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Kuratierte Setup-Helper für lokale/selbstgehostete Provider |
    | `plugin-sdk/self-hosted-provider-setup` | Fokussierte Setup-Helper für OpenAI-kompatible selbstgehostete Provider |
    | `plugin-sdk/cli-backend` | Standardwerte für CLI-Backends + Watchdog-Konstanten |
    | `plugin-sdk/provider-auth-runtime` | Helper für die Auflösung von Runtime-API-Schlüsseln für Provider-Plugins |
    | `plugin-sdk/provider-auth-api-key` | Onboarding-/Profile-Write-Helper für API-Schlüssel wie `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Standard-Builder für OAuth-Authentifizierungsergebnisse |
    | `plugin-sdk/provider-auth-login` | Gemeinsame interaktive Login-Helper für Provider-Plugins |
    | `plugin-sdk/provider-env-vars` | Helper zum Lookup von Umgebungsvariablen für Provider-Auth |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, gemeinsame Replay-Policy-Builder, Provider-Endpoint-Helper und Helper zur Normalisierung von Modell-IDs wie `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Generische Helper für Provider-HTTP/Endpoint-Fähigkeiten |
    | `plugin-sdk/provider-web-fetch-contract` | Schmale Helper für Web-Fetch-Konfigurations-/Auswahlverträge wie `enablePluginInConfig` und `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helper für Registrierung/Cache von Web-Fetch-Providern |
    | `plugin-sdk/provider-web-search-contract` | Schmale Helper für Web-Search-Konfigurations-/Credential-Verträge wie `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` und Scoped-Setter/Getter für Credentials |
    | `plugin-sdk/provider-web-search` | Helper für Registrierung/Cache/Runtime von Web-Search-Providern |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini-Schema-Bereinigung + Diagnostik sowie xAI-Kompatibilitäts-Helper wie `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` und Ähnliches |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, Stream-Wrapper-Typen und gemeinsame Wrapper-Helper für Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helper zum Patchen von Onboarding-Konfigurationen |
    | `plugin-sdk/global-singleton` | Prozesslokale Singleton-/Map-/Cache-Helper |
  </Accordion>

  <Accordion title="Auth- und Sicherheits-Unterpfade">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, Helper für Befehlsregistrierung und Sender-Autorisierung |
    | `plugin-sdk/approval-auth-runtime` | Approver-Auflösung und Helper für Same-Chat-Action-Auth |
    | `plugin-sdk/approval-client-runtime` | Native Helper für Exec-Approval-Profile/-Filter |
    | `plugin-sdk/approval-delivery-runtime` | Native Adapter für Approval-Fähigkeiten/Zustellung |
    | `plugin-sdk/approval-gateway-runtime` | Gemeinsamer Helper zur Auflösung des Approval-Gateways |
    | `plugin-sdk/approval-handler-adapter-runtime` | Leichtgewichtige Lade-Helper für native Approval-Adapter für Hot-Channel-Entrypoints |
    | `plugin-sdk/approval-handler-runtime` | Breitere Runtime-Helper für Approval-Handler; bevorzugen Sie die schmaleren Adapter-/Gateway-Seams, wenn sie ausreichen |
    | `plugin-sdk/approval-native-runtime` | Native Helper für Approval-Ziele und Account-Bindings |
    | `plugin-sdk/approval-reply-runtime` | Helper für Antwort-Payloads zu Exec-/Plugin-Approvals |
    | `plugin-sdk/command-auth-native` | Native Command-Auth und native Session-Target-Helper |
    | `plugin-sdk/command-detection` | Gemeinsame Helper zur Befehlserkennung |
    | `plugin-sdk/command-surface` | Helper zur Normalisierung von Befehls-Bodies und Befehlsoberflächen |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Schmale Collection-Helper für Secret-Verträge von Channel-/Plugin-Secret-Oberflächen |
    | `plugin-sdk/secret-ref-runtime` | Schmale `coerceSecretRef`- und SecretRef-Typing-Helper für Secret-Vertrags-/Konfigurations-Parsing |
    | `plugin-sdk/security-runtime` | Gemeinsame Helper für Vertrauen, DM-Gating, externe Inhalte und Secret-Sammlung |
    | `plugin-sdk/ssrf-policy` | Helper für Host-Allowlists und SSRF-Richtlinien für private Netzwerke |
    | `plugin-sdk/ssrf-runtime` | Helper für Pinned-Dispatcher, SSRF-geschütztes Fetch und SSRF-Richtlinien |
    | `plugin-sdk/secret-input` | Helper zum Parsen geheimer Eingaben |
    | `plugin-sdk/webhook-ingress` | Helper für Webhook-Requests/-Ziele |
    | `plugin-sdk/webhook-request-guards` | Helper für Größe/Timeout des Request-Bodys |
  </Accordion>

  <Accordion title="Runtime- und Speicher-Unterpfade">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/runtime` | Breite Helper für Runtime/Logging/Backups/Plugin-Installation |
    | `plugin-sdk/runtime-env` | Schmale Helper für Runtime-Umgebung, Logger, Timeout, Retry und Backoff |
    | `plugin-sdk/channel-runtime-context` | Generische Helper für Registrierung und Lookup von Channel-Runtime-Kontexten |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Gemeinsame Helper für Plugin-Befehle/Hooks/HTTP/Interaktivität |
    | `plugin-sdk/hook-runtime` | Gemeinsame Helper für Webhook-/interne Hook-Pipelines |
    | `plugin-sdk/lazy-runtime` | Helper für Lazy-Runtime-Importe/-Bindings wie `createLazyRuntimeModule`, `createLazyRuntimeMethod` und `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helper für Prozessausführung |
    | `plugin-sdk/cli-runtime` | Helper für CLI-Formatierung, Warten und Version |
    | `plugin-sdk/gateway-runtime` | Gateway-Client- und Helper zum Patchen des Channel-Status |
    | `plugin-sdk/config-runtime` | Helper zum Laden/Schreiben von Konfigurationen |
    | `plugin-sdk/telegram-command-config` | Normalisierung von Telegram-Befehlsnamen/-beschreibungen und Prüfungen auf Duplikate/Konflikte, auch wenn die gebündelte Telegram-Vertragsoberfläche nicht verfügbar ist |
    | `plugin-sdk/approval-runtime` | Helper für Exec-/Plugin-Approvals, Approval-Capability-Builder, Auth-/Profil-Helper und native Routing-/Runtime-Helper |
    | `plugin-sdk/reply-runtime` | Gemeinsame Inbound-/Reply-Runtime-Helper, Chunking, Dispatch, Heartbeat und Reply-Planer |
    | `plugin-sdk/reply-dispatch-runtime` | Schmale Helper für Antwort-Dispatch/-Finalisierung |
    | `plugin-sdk/reply-history` | Gemeinsame Helper für Reply-Historie in kurzen Fenstern wie `buildHistoryContext`, `recordPendingHistoryEntry` und `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Schmale Helper für Text-/Markdown-Chunking |
    | `plugin-sdk/session-store-runtime` | Helper für Pfade und `updated-at` im Session-Store |
    | `plugin-sdk/state-paths` | Helper für Pfade zu State-/OAuth-Verzeichnissen |
    | `plugin-sdk/routing` | Helper für Route-/Session-Key-/Account-Bindings wie `resolveAgentRoute`, `buildAgentSessionKey` und `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Gemeinsame Helper für Zusammenfassungen des Channel-/Account-Status, Runtime-State-Standards und Metadaten zu Problemen |
    | `plugin-sdk/target-resolver-runtime` | Gemeinsame Helper für Target-Resolver |
    | `plugin-sdk/string-normalization-runtime` | Helper zur Slug-/String-Normalisierung |
    | `plugin-sdk/request-url` | String-URLs aus Fetch-/Request-ähnlichen Eingaben extrahieren |
    | `plugin-sdk/run-command` | Zeitgesteuerter Befehls-Runner mit normalisierten stdout-/stderr-Ergebnissen |
    | `plugin-sdk/param-readers` | Allgemeine Tool-/CLI-Parameterleser |
    | `plugin-sdk/tool-send` | Kanonische Send-Zielfelder aus Tool-Argumenten extrahieren |
    | `plugin-sdk/temp-path` | Gemeinsame Helper für temporäre Download-Pfade |
    | `plugin-sdk/logging-core` | Helper für Subsystem-Logger und Redaction |
    | `plugin-sdk/markdown-table-runtime` | Helper für Markdown-Tabellenmodi |
    | `plugin-sdk/json-store` | Kleine Helper zum Lesen/Schreiben von JSON-State |
    | `plugin-sdk/file-lock` | Reentrante File-Lock-Helper |
    | `plugin-sdk/persistent-dedupe` | Helper für diskgestützten Dedupe-Cache |
    | `plugin-sdk/acp-runtime` | Helper für ACP-Runtime/Session und Reply-Dispatch |
    | `plugin-sdk/agent-config-primitives` | Schmale Primitive für Agent-Runtime-Konfigurationsschemas |
    | `plugin-sdk/boolean-param` | Leser für lockere Boolesche Parameter |
    | `plugin-sdk/dangerous-name-runtime` | Helper zur Auflösung von Dangerous-Name-Matching |
    | `plugin-sdk/device-bootstrap` | Helper für Device-Bootstrap und Pairing-Token |
    | `plugin-sdk/extension-shared` | Gemeinsame Primitive für passive Channels, Status und Ambient-Proxy-Helper |
    | `plugin-sdk/models-provider-runtime` | Helper für `/models`-Befehle/Provider-Antworten |
    | `plugin-sdk/skill-commands-runtime` | Helper zum Auflisten von Skill-Befehlen |
    | `plugin-sdk/native-command-registry` | Helper zum Registrieren/Bauen/Serialisieren nativer Befehle |
    | `plugin-sdk/provider-zai-endpoint` | Helper zur Erkennung von Z.AI-Endpoints |
    | `plugin-sdk/infra-runtime` | Helper für Systemereignisse/Heartbeat |
    | `plugin-sdk/collection-runtime` | Kleine Helper für begrenzte Caches |
    | `plugin-sdk/diagnostic-runtime` | Helper für Diagnose-Flags und -Ereignisse |
    | `plugin-sdk/error-runtime` | Helper für Fehlergraphen, Formatierung, gemeinsame Fehlerklassifikation, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helper für Wrapped Fetch, Proxy und Pinned-Lookup |
    | `plugin-sdk/host-runtime` | Helper zur Normalisierung von Hostnamen und SCP-Hosts |
    | `plugin-sdk/retry-runtime` | Helper für Retry-Konfiguration und Retry-Runner |
    | `plugin-sdk/agent-runtime` | Helper für Agent-Verzeichnis/Identität/Workspace |
    | `plugin-sdk/directory-runtime` | Konfigurationsgestützte Verzeichnisabfrage/-Deduplizierung |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Unterpfade für Fähigkeiten und Tests">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Gemeinsame Helper für Fetch/Transformation/Speicherung von Medien sowie Builder für Medien-Payloads |
    | `plugin-sdk/media-generation-runtime` | Gemeinsame Helper für Media-Generation-Failover, Kandidatenauswahl und Meldungen zu fehlenden Modellen |
    | `plugin-sdk/media-understanding` | Provider-Typen für Media Understanding sowie providerseitige Exporte für Bild-/Audio-Helper |
    | `plugin-sdk/text-runtime` | Gemeinsame Text-/Markdown-/Logging-Helper wie das Entfernen von für Assistenten sichtbarem Text, Markdown-Render-/Chunking-/Tabellen-Helper, Redaction-Helper, Directive-Tag-Helper und Safe-Text-Utilities |
    | `plugin-sdk/text-chunking` | Helper für Chunking ausgehenden Texts |
    | `plugin-sdk/speech` | Typen für Sprachprovider sowie providerseitige Helper für Directive, Registry und Validierung |
    | `plugin-sdk/speech-core` | Gemeinsame Typen für Sprachprovider, Registry, Directive und Normalisierungs-Helper |
    | `plugin-sdk/realtime-transcription` | Typen für Realtime-Transkriptionsprovider und Registry-Helper |
    | `plugin-sdk/realtime-voice` | Typen für Realtime-Voice-Provider und Registry-Helper |
    | `plugin-sdk/image-generation` | Provider-Typen für Bildgenerierung |
    | `plugin-sdk/image-generation-core` | Gemeinsame Typen, Failover-, Auth- und Registry-Helper für Bildgenerierung |
    | `plugin-sdk/music-generation` | Typen für Music-Generation-Provider/Requests/Ergebnisse |
    | `plugin-sdk/music-generation-core` | Gemeinsame Typen, Failover-Helper, Provider-Lookup und Modellref-Parsing für Music Generation |
    | `plugin-sdk/video-generation` | Typen für Video-Generation-Provider/Requests/Ergebnisse |
    | `plugin-sdk/video-generation-core` | Gemeinsame Typen, Failover-Helper, Provider-Lookup und Modellref-Parsing für Video Generation |
    | `plugin-sdk/webhook-targets` | Registry für Webhook-Ziele und Helper zur Routeninstallation |
    | `plugin-sdk/webhook-path` | Helper zur Normalisierung von Webhook-Pfaden |
    | `plugin-sdk/web-media` | Gemeinsame Helper zum Laden entfernter/lokaler Medien |
    | `plugin-sdk/zod` | Re-exportiertes `zod` für Plugin-SDK-Konsumenten |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Memory-Unterpfade">
    | Subpath | Key exports |
    | --- | --- |
    | `plugin-sdk/memory-core` | Helper-Oberfläche des gebündelten memory-core für Manager-/Konfigurations-/Datei-/CLI-Helper |
    | `plugin-sdk/memory-core-engine-runtime` | Runtime-Fassade für Memory-Index/Search |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exporte der Foundation-Engine des Memory-Hosts |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exporte der Embedding-Engine des Memory-Hosts |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exporte der QMD-Engine des Memory-Hosts |
    | `plugin-sdk/memory-core-host-engine-storage` | Exporte der Storage-Engine des Memory-Hosts |
    | `plugin-sdk/memory-core-host-multimodal` | Multimodale Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-query` | Query-Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-secret` | Secret-Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-events` | Helper für das Event-Journal des Memory-Hosts |
    | `plugin-sdk/memory-core-host-status` | Status-Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-runtime-cli` | CLI-Runtime-Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-runtime-core` | Core-Runtime-Helper des Memory-Hosts |
    | `plugin-sdk/memory-core-host-runtime-files` | Datei-/Runtime-Helper des Memory-Hosts |
    | `plugin-sdk/memory-host-core` | Herstellerneutraler Alias für Core-Runtime-Helper des Memory-Hosts |
    | `plugin-sdk/memory-host-events` | Herstellerneutraler Alias für Helper für das Event-Journal des Memory-Hosts |
    | `plugin-sdk/memory-host-files` | Herstellerneutraler Alias für Datei-/Runtime-Helper des Memory-Hosts |
    | `plugin-sdk/memory-host-markdown` | Gemeinsame Helper für verwaltetes Markdown für Memory-nahe Plugins |
    | `plugin-sdk/memory-host-search` | Aktive Runtime-Fassade für Memory für den Zugriff auf Search-Manager |
    | `plugin-sdk/memory-host-status` | Herstellerneutraler Alias für Status-Helper des Memory-Hosts |
    | `plugin-sdk/memory-lancedb` | Helper-Oberfläche des gebündelten memory-lancedb |
  </Accordion>

  <Accordion title="Reservierte Unterpfade für gebündelte Helper">
    | Family | Current subpaths | Intended use |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Support-Helper für das gebündelte Browser-Plugin (`browser-support` bleibt das Kompatibilitäts-Barrel) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Helper-/Runtime-Oberfläche des gebündelten Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Helper-/Runtime-Oberfläche des gebündelten LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Helper-Oberfläche des gebündelten IRC |
    | Channelspezifische Helper | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Gebündelte Kompatibilitäts-/Helper-Seams für Channels |
    | Auth-/pluginspezifische Helper | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Gebündelte Feature-/Plugin-Helper-Seams; `plugin-sdk/github-copilot-token` exportiert derzeit `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` und `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## Registrierungs-API

Der Callback `register(api)` erhält ein `OpenClawPluginApi`-Objekt mit diesen
Methoden:

### Registrierung von Fähigkeiten

| Method                                           | What it registers                |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Textinferenz (LLM)               |
| `api.registerCliBackend(...)`                    | Lokales CLI-Inferenz-Backend     |
| `api.registerChannel(...)`                       | Nachrichten-Channel              |
| `api.registerSpeechProvider(...)`                | Text-to-Speech / STT-Synthese    |
| `api.registerRealtimeTranscriptionProvider(...)` | Streaming-Realtime-Transkription |
| `api.registerRealtimeVoiceProvider(...)`         | Duplex-Realtime-Voice-Sitzungen  |
| `api.registerMediaUnderstandingProvider(...)`    | Bild-/Audio-/Videoanalyse        |
| `api.registerImageGenerationProvider(...)`       | Bildgenerierung                  |
| `api.registerMusicGenerationProvider(...)`       | Musikgenerierung                 |
| `api.registerVideoGenerationProvider(...)`       | Videogenerierung                 |
| `api.registerWebFetchProvider(...)`              | Web-Fetch-/Scrape-Provider       |
| `api.registerWebSearchProvider(...)`             | Websuche                         |

### Tools und Befehle

| Method                          | What it registers                             |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Agent-Tool (erforderlich oder `{ optional: true }`) |
| `api.registerCommand(def)`      | Benutzerdefinierter Befehl (umgeht das LLM)   |

### Infrastruktur

| Method                                         | What it registers                       |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Event-Hook                              |
| `api.registerHttpRoute(params)`                | Gateway-HTTP-Endpoint                   |
| `api.registerGatewayMethod(name, handler)`     | Gateway-RPC-Methode                     |
| `api.registerCli(registrar, opts?)`            | CLI-Unterbefehl                         |
| `api.registerService(service)`                 | Hintergrunddienst                       |
| `api.registerInteractiveHandler(registration)` | Interaktiver Handler                    |
| `api.registerMemoryPromptSupplement(builder)`  | Additiver promptnaher Memory-Abschnitt  |
| `api.registerMemoryCorpusSupplement(adapter)`  | Additiver Memory-Such-/Lese-Korpus      |

Reservierte Core-Admin-Namespaces (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) bleiben immer `operator.admin`, auch wenn ein Plugin versucht, einer
Gateway-Methode einen engeren Scope zuzuweisen. Bevorzugen Sie pluginspezifische Präfixe für
plugin-eigene Methoden.

### Metadaten zur CLI-Registrierung

`api.registerCli(registrar, opts?)` akzeptiert zwei Arten von Metadaten auf oberster Ebene:

- `commands`: explizite Befehlswurzeln, die dem Registrar gehören
- `descriptors`: Parse-Time-Befehlsdeskriptoren für Root-CLI-Hilfe,
  Routing und Lazy-CLI-Registrierung von Plugins

Wenn Sie möchten, dass ein Plugin-Befehl im normalen Root-CLI-Pfad lazy geladen bleibt,
stellen Sie `descriptors` bereit, die jede oberste Befehlswurzel abdecken, die durch diesen
Registrar verfügbar gemacht wird.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix-Konten, Verifizierung, Geräte und Profilstatus verwalten",
        hasSubcommands: true,
      },
    ],
  },
);
```

Verwenden Sie `commands` allein nur dann, wenn Sie keine lazy Root-CLI-Registrierung benötigen.
Dieser eager-Kompatibilitätspfad wird weiterhin unterstützt, installiert jedoch keine
deskriptorbasierten Platzhalter für parsezeitiges Lazy Loading.

### Registrierung von CLI-Backends

`api.registerCliBackend(...)` erlaubt es einem Plugin, die Standardkonfiguration für ein lokales
AI-CLI-Backend wie `codex-cli` zu besitzen.

- Die `id` des Backends wird zum Provider-Präfix in Modellreferenzen wie `codex-cli/gpt-5`.
- Die `config` des Backends verwendet dieselbe Form wie `agents.defaults.cliBackends.<id>`.
- Benutzerkonfiguration hat weiterhin Vorrang. OpenClaw führt `agents.defaults.cliBackends.<id>` über dem
  Plugin-Standard zusammen, bevor die CLI ausgeführt wird.
- Verwenden Sie `normalizeConfig`, wenn ein Backend nach dem Zusammenführen Kompatibilitäts-Umschreibungen benötigt
  (zum Beispiel zur Normalisierung alter Flag-Formen).

### Exklusive Slots

| Method                                     | What it registers                     |
| ------------------------------------------ | ------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Kontext-Engine (jeweils nur eine aktiv) |
| `api.registerMemoryCapability(capability)` | Vereinheitlichte Memory-Fähigkeit     |
| `api.registerMemoryPromptSection(builder)` | Builder für Memory-Prompt-Abschnitte  |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver für Memory-Flush-Pläne       |
| `api.registerMemoryRuntime(runtime)`       | Memory-Runtime-Adapter                |

### Memory-Embedding-Adapter

| Method                                         | What it registers                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Memory-Embedding-Adapter für das aktive Plugin |

- `registerMemoryCapability` ist die bevorzugte exklusive API für Memory-Plugins.
- `registerMemoryCapability` kann auch `publicArtifacts.listArtifacts(...)`
  bereitstellen, damit Begleit-Plugins exportierte Memory-Artefakte über
  `openclaw/plugin-sdk/memory-host-core` verwenden können, statt in das private
  Layout eines bestimmten Memory-Plugins zu greifen.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` und
  `registerMemoryRuntime` sind Legacy-kompatible exklusive APIs für Memory-Plugins.
- `registerMemoryEmbeddingProvider` erlaubt dem aktiven Memory-Plugin, einen
  oder mehrere Embedding-Adapter-IDs zu registrieren (zum Beispiel `openai`, `gemini` oder eine benutzerdefinierte plugindefinierte ID).
- Benutzerkonfigurationen wie `agents.defaults.memorySearch.provider` und
  `agents.defaults.memorySearch.fallback` werden gegen diese registrierten
  Adapter-IDs aufgelöst.

### Ereignisse und Lifecycle

| Method                                       | What it does                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Typisierter Lifecycle-Hook    |
| `api.onConversationBindingResolved(handler)` | Callback für Konversations-Binding |

### Entscheidungssemantik von Hooks

- `before_tool_call`: Die Rückgabe von `{ block: true }` ist final. Sobald ein Handler dies setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `before_tool_call`: Die Rückgabe von `{ block: false }` wird als keine Entscheidung behandelt (wie das Weglassen von `block`), nicht als Override.
- `before_install`: Die Rückgabe von `{ block: true }` ist final. Sobald ein Handler dies setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `before_install`: Die Rückgabe von `{ block: false }` wird als keine Entscheidung behandelt (wie das Weglassen von `block`), nicht als Override.
- `reply_dispatch`: Die Rückgabe von `{ handled: true, ... }` ist final. Sobald ein Handler den Dispatch beansprucht, werden Handler mit niedrigerer Priorität und der Standardpfad für den Modell-Dispatch übersprungen.
- `message_sending`: Die Rückgabe von `{ cancel: true }` ist final. Sobald ein Handler dies setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `message_sending`: Die Rückgabe von `{ cancel: false }` wird als keine Entscheidung behandelt (wie das Weglassen von `cancel`), nicht als Override.

### Felder des API-Objekts

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin-ID                                                                                   |
| `api.name`               | `string`                  | Anzeigename                                                                                 |
| `api.version`            | `string?`                 | Plugin-Version (optional)                                                                   |
| `api.description`        | `string?`                 | Plugin-Beschreibung (optional)                                                              |
| `api.source`             | `string`                  | Plugin-Quellpfad                                                                            |
| `api.rootDir`            | `string?`                 | Plugin-Root-Verzeichnis (optional)                                                          |
| `api.config`             | `OpenClawConfig`          | Aktueller Konfigurations-Snapshot (aktive In-Memory-Runtime-Snapshot, wenn verfügbar)      |
| `api.pluginConfig`       | `Record<string, unknown>` | Pluginspezifische Konfiguration aus `plugins.entries.<id>.config`                           |
| `api.runtime`            | `PluginRuntime`           | [Runtime-Helper](/de/plugins/sdk-runtime)                                                      |
| `api.logger`             | `PluginLogger`            | Scoped Logger (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Aktueller Lademodus; `"setup-runtime"` ist das leichtgewichtige Fenster vor dem vollständigen Entry für Start/Setup |
| `api.resolvePath(input)` | `(string) => string`      | Pfad relativ zum Plugin-Root auflösen                                                       |

## Interne Modulkonvention

Verwenden Sie innerhalb Ihres Plugins lokale Barrel-Dateien für interne Importe:

```
my-plugin/
  api.ts            # Öffentliche Exporte für externe Konsumenten
  runtime-api.ts    # Interne Runtime-Exporte
  index.ts          # Plugin-Einstiegspunkt
  setup-entry.ts    # Leichtgewichtiger setup-only-Einstiegspunkt (optional)
```

<Warning>
  Importieren Sie Ihr eigenes Plugin in Produktionscode niemals über `openclaw/plugin-sdk/<your-plugin>`.
  Leiten Sie interne Importe über `./api.ts` oder
  `./runtime-api.ts`. Der SDK-Pfad ist nur der externe Vertrag.
</Warning>

Facadengeladene öffentliche Oberflächen gebündelter Plugins (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` und ähnliche öffentliche Entry-Dateien) bevorzugen jetzt den
aktiven Runtime-Konfigurations-Snapshot, wenn OpenClaw bereits läuft. Wenn noch kein Runtime-
Snapshot existiert, greifen sie auf die auf der Festplatte aufgelöste Konfigurationsdatei zurück.

Provider-Plugins können auch ein schmales pluginlokales Vertrags-Barrel bereitstellen, wenn ein
Helper absichtlich providerspezifisch ist und noch nicht in einen generischen SDK-Unterpfad gehört. Aktuelles gebündeltes Beispiel: Der Anthropic-Provider hält seine Claude-
Stream-Helper in seiner eigenen öffentlichen `api.ts`-/`contract-api.ts`-Seam, statt Anthropic-Beta-Header und `service_tier`-Logik in einen generischen
`plugin-sdk/*`-Vertrag zu überführen.

Weitere aktuelle gebündelte Beispiele:

- `@openclaw/openai-provider`: `api.ts` exportiert Provider-Builder,
  Standardmodell-Helper und Realtime-Provider-Builder
- `@openclaw/openrouter-provider`: `api.ts` exportiert den Provider-Builder sowie
  Helper für Onboarding/Konfiguration

<Warning>
  Produktionscode von Extensions sollte außerdem Importe von `openclaw/plugin-sdk/<other-plugin>`
  vermeiden. Wenn ein Helper wirklich gemeinsam genutzt wird, überführen Sie ihn in einen neutralen SDK-Unterpfad
  wie `openclaw/plugin-sdk/speech`, `.../provider-model-shared` oder eine andere
  fähigkeitsorientierte Oberfläche, statt zwei Plugins miteinander zu koppeln.
</Warning>

## Zugehörig

- [Einstiegspunkte](/de/plugins/sdk-entrypoints) — Optionen für `definePluginEntry` und `defineChannelPluginEntry`
- [Runtime-Helper](/de/plugins/sdk-runtime) — vollständige Referenz des Namespace `api.runtime`
- [Setup und Konfiguration](/de/plugins/sdk-setup) — Paketierung, Manifeste, Konfigurationsschemas
- [Testing](/de/plugins/sdk-testing) — Test-Utilities und Lint-Regeln
- [SDK-Migration](/de/plugins/sdk-migration) — Migration von veralteten Oberflächen
- [Plugin-Interna](/de/plugins/architecture) — detaillierte Architektur und Fähigkeitsmodell
