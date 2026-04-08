---
read_when:
    - Die Warnung OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED wird angezeigt
    - Die Warnung OPENCLAW_EXTENSION_API_DEPRECATED wird angezeigt
    - Ein Plugin wird auf die moderne Plugin-Architektur aktualisiert
    - Ein externes OpenClaw-Plugin wird verwaltet
sidebarTitle: Migrate to SDK
summary: Von der Legacy-Abwärtskompatibilitätsschicht zum modernen Plugin SDK migrieren
title: Migration des Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:17:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 155a8b14bc345319c8516ebdb8a0ccdea2c5f7fa07dad343442996daee21ecad
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migration des Plugin SDK

OpenClaw ist von einer breiten Abwärtskompatibilitätsschicht zu einer modernen Plugin-
Architektur mit fokussierten, dokumentierten Imports übergegangen. Wenn dein Plugin vor
der neuen Architektur erstellt wurde, hilft dir diese Anleitung bei der Migration.

## Was sich ändert

Das alte Plugin-System stellte zwei weit offene Oberflächen bereit, über die Plugins
alles importieren konnten, was sie über einen einzigen Einstiegspunkt benötigten:

- **`openclaw/plugin-sdk/compat`** — ein einzelner Import, der Dutzende von
  Hilfsfunktionen re-exportierte. Er wurde eingeführt, damit ältere hookbasierte Plugins weiter funktionieren,
  während die neue Plugin-Architektur aufgebaut wurde.
- **`openclaw/extension-api`** — eine Brücke, die Plugins direkten Zugriff auf
  hostseitige Hilfsfunktionen wie den eingebetteten Agent-Runner gab.

Beide Oberflächen sind jetzt **veraltet**. Sie funktionieren zur Laufzeit weiterhin, aber neue
Plugins dürfen sie nicht verwenden, und bestehende Plugins sollten migrieren, bevor die nächste
Major-Version sie entfernt.

<Warning>
  Die Abwärtskompatibilitätsschicht wird in einer zukünftigen Major-Version entfernt.
  Plugins, die weiterhin von diesen Oberflächen importieren, werden dann nicht mehr funktionieren.
</Warning>

## Warum sich das geändert hat

Der alte Ansatz verursachte Probleme:

- **Langsamer Start** — der Import einer Hilfsfunktion lud Dutzende nicht zusammenhängender Module
- **Zirkuläre Abhängigkeiten** — breite Re-Exports machten es einfach, Importzyklen zu erzeugen
- **Unklare API-Oberfläche** — es gab keine Möglichkeit zu erkennen, welche Exporte stabil und welche intern waren

Das moderne Plugin SDK behebt das: Jeder Importpfad (`openclaw/plugin-sdk/\<subpath\>`)
ist ein kleines, in sich geschlossenes Modul mit einem klaren Zweck und dokumentiertem Vertrag.

Legacy-Provider-Convenience-Seams für gebündelte Kanäle sind ebenfalls verschwunden. Imports
wie `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
kanalmarkierte Helper-Seams und
`openclaw/plugin-sdk/telegram-core` waren private Monorepo-Abkürzungen, keine
stabilen Plugin-Verträge. Verwende stattdessen schmale generische SDK-Subpaths. Innerhalb des
gebündelten Plugin-Workspace sollten provider-eigene Hilfsfunktionen im jeweiligen
`api.ts` oder `runtime-api.ts` des Plugins bleiben.

Aktuelle Beispiele für gebündelte Provider:

- Anthropic behält Claude-spezifische Stream-Hilfsfunktionen in seinem eigenen `api.ts` /
  `contract-api.ts`-Seam
- OpenAI behält Provider-Builder, Default-Model-Hilfsfunktionen und Realtime-Provider-
  Builder in seinem eigenen `api.ts`
- OpenRouter behält Provider-Builder sowie Onboarding-/Konfigurationshilfsfunktionen in seinem eigenen
  `api.ts`

## So migrierst du

<Steps>
  <Step title="Approval-native-Handler auf Capability-Fakten migrieren">
    Freigabefähige Kanal-Plugins stellen natives Freigabeverhalten jetzt über
    `approvalCapability.nativeRuntime` plus das gemeinsame Runtime-Context-Registry bereit.

    Wichtige Änderungen:

    - Ersetze `approvalCapability.handler.loadRuntime(...)` durch
      `approvalCapability.nativeRuntime`
    - Verschiebe Freigabe-spezifische Auth/Delivery von der Legacy-Verkabelung
      `plugin.auth` / `plugin.approvals` auf `approvalCapability`
    - `ChannelPlugin.approvals` wurde aus dem öffentlichen Vertrag für Kanal-Plugins
      entfernt; verschiebe Delivery-/native-/render-Felder auf `approvalCapability`
    - `plugin.auth` bleibt nur für Kanal-Login-/Logout-Abläufe bestehen; Freigabe-Auth-
      Hooks dort werden vom Core nicht mehr gelesen
    - Registriere kanal-eigene Runtime-Objekte wie Clients, Tokens oder Bolt-
      Apps über `openclaw/plugin-sdk/channel-runtime-context`
    - Sende keine plugin-eigenen Reroute-Hinweise aus nativen Freigabe-Handlern;
      der Core übernimmt jetzt Routed-Elsewhere-Hinweise aus tatsächlichen Delivery-Ergebnissen
    - Wenn `channelRuntime` an `createChannelManager(...)` übergeben wird, stelle eine
      echte `createPluginRuntime().channel`-Oberfläche bereit. Partielle Stubs werden abgelehnt.

    Siehe `/plugins/sdk-channel-plugins` für das aktuelle Layout der
    Freigabe-Capability.

  </Step>

  <Step title="Fallback-Verhalten des Windows-Wrappers prüfen">
    Wenn dein Plugin `openclaw/plugin-sdk/windows-spawn` verwendet,
    schlagen nicht aufgelöste Windows-Wrapper vom Typ `.cmd`/`.bat` jetzt fail-closed fehl, sofern nicht
    ausdrücklich `allowShellFallback: true` übergeben wird.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    Wenn dein Aufrufer nicht absichtlich auf Shell-Fallback angewiesen ist, setze
    `allowShellFallback` nicht und behandle stattdessen den ausgelösten Fehler.

  </Step>

  <Step title="Veraltete Imports finden">
    Durchsuche dein Plugin nach Imports von einer der beiden veralteten Oberflächen:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Durch fokussierte Imports ersetzen">
    Jeder Export der alten Oberfläche entspricht einem bestimmten modernen Importpfad:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Für hostseitige Hilfsfunktionen verwende die injizierte Plugin-Runtime, statt direkt
    zu importieren:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Dasselbe Muster gilt für andere Legacy-Brücken-Hilfsfunktionen:

    | Alter Import | Modernes Äquivalent |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | Sitzungsspeicher-Hilfsfunktionen | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Build und Tests">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Referenz für Importpfade

<Accordion title="Tabelle häufiger Importpfade">
  | Importpfad | Zweck | Wichtige Exporte |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Kanonische Plugin-Einstiegshilfe | `definePluginEntry` |
  | `plugin-sdk/core` | Legacy-Überdach-Re-Export für Kanal-Entry-Definitionen/Builder | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Export des Root-Konfigurationsschemas | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Einstiegshilfe für Single-Provider | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Fokussierte Kanal-Entry-Definitionen und Builder | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Gemeinsame Hilfsfunktionen für den Setup-Wizard | Allowlist-Prompts, Builder für Setup-Status |
  | `plugin-sdk/setup-runtime` | Runtime-Hilfsfunktionen zur Setup-Zeit | Importsichere Setup-Patch-Adapter, Hilfsfunktionen für Lookup-Notizen, `promptResolvedAllowFrom`, `splitSetupEntries`, delegierte Setup-Proxys |
  | `plugin-sdk/setup-adapter-runtime` | Hilfsfunktionen für Setup-Adapter | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Hilfsfunktionen für Setup-Tooling | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Hilfsfunktionen für mehrere Accounts | Hilfsfunktionen für Account-Liste/Konfiguration/Aktions-Gates |
  | `plugin-sdk/account-id` | Hilfsfunktionen für Account-IDs | `DEFAULT_ACCOUNT_ID`, Normalisierung von Account-IDs |
  | `plugin-sdk/account-resolution` | Hilfsfunktionen für Account-Lookups | Hilfsfunktionen für Account-Lookup + Default-Fallback |
  | `plugin-sdk/account-helpers` | Schmale Account-Hilfsfunktionen | Hilfsfunktionen für Account-Liste/Account-Aktionen |
  | `plugin-sdk/channel-setup` | Adapter für den Setup-Wizard | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitive für DM-Pairing | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Verkabelung für Antwortpräfix + Tippen | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fabriken für Konfigurationsadapter | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builder für Konfigurationsschemas | Typen für Kanal-Konfigurationsschemas |
  | `plugin-sdk/telegram-command-config` | Hilfsfunktionen für Telegram-Befehlskonfiguration | Normalisierung von Befehlsnamen, Kürzen von Beschreibungen, Validierung von Duplikaten/Konflikten |
  | `plugin-sdk/channel-policy` | Auflösung von Gruppen-/DM-Richtlinien | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Statusverfolgung für Accounts | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Hilfsfunktionen für eingehende Umschläge | Gemeinsame Hilfsfunktionen für Route- + Umschlag-Builder |
  | `plugin-sdk/inbound-reply-dispatch` | Hilfsfunktionen für eingehende Antworten | Gemeinsame Hilfsfunktionen für record-and-dispatch |
  | `plugin-sdk/messaging-targets` | Parsing von Messaging-Zielen | Hilfsfunktionen für Ziel-Parsing/-Matching |
  | `plugin-sdk/outbound-media` | Hilfsfunktionen für ausgehende Medien | Gemeinsames Laden ausgehender Medien |
  | `plugin-sdk/outbound-runtime` | Hilfsfunktionen für ausgehende Runtime | Hilfsfunktionen für ausgehende Identität/Send-Delegates |
  | `plugin-sdk/thread-bindings-runtime` | Hilfsfunktionen für Thread-Bindings | Hilfsfunktionen für Thread-Binding-Lebenszyklus und -Adapter |
  | `plugin-sdk/agent-media-payload` | Legacy-Hilfsfunktionen für Medien-Nutzlasten | Builder für Agent-Medien-Nutzlasten für Legacy-Feldlayouts |
  | `plugin-sdk/channel-runtime` | Veralteter Kompatibilitäts-Shim | Nur Legacy-Hilfsfunktionen für Kanal-Runtime |
  | `plugin-sdk/channel-send-result` | Typen für Sendeergebnisse | Typen für Antwortergebnisse |
  | `plugin-sdk/runtime-store` | Persistenter Plugin-Speicher | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Breite Runtime-Hilfsfunktionen | Hilfsfunktionen für Runtime/Logging/Backup/Plugin-Install |
  | `plugin-sdk/runtime-env` | Schmale Hilfsfunktionen für Runtime-Umgebung | Logger/Runtime-Umgebung, Timeout-, Retry- und Backoff-Hilfsfunktionen |
  | `plugin-sdk/plugin-runtime` | Gemeinsame Plugin-Runtime-Hilfsfunktionen | Hilfsfunktionen für Plugin-Befehle/Hooks/HTTP/interaktive Abläufe |
  | `plugin-sdk/hook-runtime` | Hilfsfunktionen für Hook-Pipelines | Gemeinsame Hilfsfunktionen für Webhook-/interne Hook-Pipelines |
  | `plugin-sdk/lazy-runtime` | Hilfsfunktionen für Lazy Runtime | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Hilfsfunktionen für Prozesse | Gemeinsame Exec-Hilfsfunktionen |
  | `plugin-sdk/cli-runtime` | Hilfsfunktionen für CLI-Runtime | Hilfsfunktionen für Befehlsformatierung, Waits, Versionen |
  | `plugin-sdk/gateway-runtime` | Gateway-Hilfsfunktionen | Hilfsfunktionen für Gateway-Client und Channel-Status-Patches |
  | `plugin-sdk/config-runtime` | Konfigurationshilfen | Hilfsfunktionen zum Laden/Schreiben von Konfiguration |
  | `plugin-sdk/telegram-command-config` | Hilfsfunktionen für Telegram-Befehle | Fallback-stabile Hilfsfunktionen zur Telegram-Befehlsvalidierung, wenn die gebündelte Telegram-Vertragsoberfläche nicht verfügbar ist |
  | `plugin-sdk/approval-runtime` | Hilfsfunktionen für Freigabe-Prompts | Nutzlasten für Exec-/Plugin-Freigaben, Hilfsfunktionen für Freigabe-Capability/-Profil, natives Freigabe-Routing/-Runtime |
  | `plugin-sdk/approval-auth-runtime` | Hilfsfunktionen für Freigabe-Auth | Auflösung von Approvern, Auth für Aktionen im selben Chat |
  | `plugin-sdk/approval-client-runtime` | Hilfsfunktionen für Freigabe-Clients | Hilfsfunktionen für natives Exec-Freigabeprofil/-Filter |
  | `plugin-sdk/approval-delivery-runtime` | Hilfsfunktionen für Freigabe-Delivery | Adapter für native Freigabe-Capability/-Delivery |
  | `plugin-sdk/approval-gateway-runtime` | Hilfsfunktionen für Freigabe-Gateway | Gemeinsame Hilfsfunktion für die Auflösung von Freigabe-Gateways |
  | `plugin-sdk/approval-handler-adapter-runtime` | Hilfsfunktionen für Freigabe-Adapter | Leichtgewichtige Hilfsfunktionen zum Laden nativer Freigabe-Adapter für heiße Kanal-Entrypoints |
  | `plugin-sdk/approval-handler-runtime` | Hilfsfunktionen für Freigabe-Handler | Breitere Runtime-Hilfsfunktionen für Freigabe-Handler; bevorzuge die schmaleren Adapter-/Gateway-Seams, wenn sie ausreichen |
  | `plugin-sdk/approval-native-runtime` | Hilfsfunktionen für Freigabe-Ziele | Hilfsfunktionen für native Bindings von Freigabe-Zielen/Accounts |
  | `plugin-sdk/approval-reply-runtime` | Hilfsfunktionen für Freigabe-Antworten | Hilfsfunktionen für Antwortnutzlasten bei Exec-/Plugin-Freigaben |
  | `plugin-sdk/channel-runtime-context` | Hilfsfunktionen für Kanal-Runtime-Context | Generische Hilfsfunktionen zum Registrieren/Abrufen/Beobachten von Kanal-Runtime-Contexts |
  | `plugin-sdk/security-runtime` | Hilfsfunktionen für Sicherheit | Gemeinsame Hilfsfunktionen für Vertrauen, DM-Gating, externe Inhalte und Secret-Erfassung |
  | `plugin-sdk/ssrf-policy` | Hilfsfunktionen für SSRF-Richtlinien | Hilfsfunktionen für Host-Allowlist und Private-Network-Richtlinien |
  | `plugin-sdk/ssrf-runtime` | Hilfsfunktionen für SSRF-Runtime | Pinned-Dispatcher, Guarded Fetch, SSRF-Richtlinienhilfen |
  | `plugin-sdk/collection-runtime` | Hilfsfunktionen für begrenzte Caches | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Hilfsfunktionen für Diagnostic-Gating | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Hilfsfunktionen für Fehlerformatierung | `formatUncaughtError`, `isApprovalNotFoundError`, Hilfsfunktionen für Fehlergraphen |
  | `plugin-sdk/fetch-runtime` | Hilfsfunktionen für gewrapptes Fetch/Proxy | `resolveFetch`, Proxy-Hilfsfunktionen |
  | `plugin-sdk/host-runtime` | Hilfsfunktionen für Host-Normalisierung | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Retry-Hilfsfunktionen | `RetryConfig`, `retryAsync`, Richtlinien-Runner |
  | `plugin-sdk/allow-from` | Formatierung von Zulassungslisten | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapping von Eingaben für Zulassungslisten | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Hilfsfunktionen für Befehls-Gating und Befehlsoberflächen | `resolveControlCommandGate`, Hilfsfunktionen für Sender-Autorisierung, Hilfsfunktionen für Befehlsregistry |
  | `plugin-sdk/secret-input` | Parsing von Secret-Eingaben | Hilfsfunktionen für Secret-Eingaben |
  | `plugin-sdk/webhook-ingress` | Hilfsfunktionen für Webhook-Anfragen | Hilfsfunktionen für Webhook-Ziele |
  | `plugin-sdk/webhook-request-guards` | Hilfsfunktionen für Guarding von Webhook-Bodies | Hilfsfunktionen zum Lesen/Begrenzen von Request-Bodies |
  | `plugin-sdk/reply-runtime` | Gemeinsame Runtime für Antworten | Inbound-Dispatch, Heartbeat, Antwortplanung, Chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Schmale Hilfsfunktionen für Reply-Dispatch | Hilfsfunktionen für Finalize + Provider-Dispatch |
  | `plugin-sdk/reply-history` | Hilfsfunktionen für Antwortverlauf | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planung von Antwortreferenzen | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Hilfsfunktionen für Antwort-Chunks | Hilfsfunktionen für Text-/Markdown-Chunking |
  | `plugin-sdk/session-store-runtime` | Hilfsfunktionen für Sitzungsspeicher | Hilfsfunktionen für Speicherpfade + updated-at |
  | `plugin-sdk/state-paths` | Hilfsfunktionen für Statuspfade | Hilfsfunktionen für Status- und OAuth-Verzeichnisse |
  | `plugin-sdk/routing` | Hilfsfunktionen für Routing/Sitzungsschlüssel | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, Hilfsfunktionen zur Normalisierung von Sitzungsschlüsseln |
  | `plugin-sdk/status-helpers` | Hilfsfunktionen für Kanalstatus | Builder für Kanal-/Account-Statuszusammenfassungen, Defaults für Runtime-Status, Hilfsfunktionen für Issue-Metadaten |
  | `plugin-sdk/target-resolver-runtime` | Hilfsfunktionen für Zielauflösung | Gemeinsame Hilfsfunktionen für Zielauflösung |
  | `plugin-sdk/string-normalization-runtime` | Hilfsfunktionen für String-Normalisierung | Hilfsfunktionen zur Slug-/String-Normalisierung |
  | `plugin-sdk/request-url` | Hilfsfunktionen für Request-URLs | String-URLs aus request-artigen Eingaben extrahieren |
  | `plugin-sdk/run-command` | Hilfsfunktionen für zeitgesteuerte Befehle | Zeitgesteuerter Befehls-Runner mit normalisiertem stdout/stderr |
  | `plugin-sdk/param-readers` | Param-Reader | Gemeinsame Param-Reader für Tool/CLI |
  | `plugin-sdk/tool-send` | Extraktion für Tool-Send | Kanonische Send-Zielfelder aus Tool-Argumenten extrahieren |
  | `plugin-sdk/temp-path` | Hilfsfunktionen für temporäre Pfade | Gemeinsame Hilfsfunktionen für Temp-Download-Pfade |
  | `plugin-sdk/logging-core` | Logging-Hilfsfunktionen | Subsystem-Logger und Hilfsfunktionen zur Redaction |
  | `plugin-sdk/markdown-table-runtime` | Hilfsfunktionen für Markdown-Tabellen | Hilfsfunktionen für den Modus von Markdown-Tabellen |
  | `plugin-sdk/reply-payload` | Typen für Nachrichtenantworten | Typen für Antwortnutzlasten |
  | `plugin-sdk/provider-setup` | Kuratierte Hilfsfunktionen für das Setup lokaler/self-hosted Provider | Hilfsfunktionen für Discovery/Konfiguration self-hosted Provider |
  | `plugin-sdk/self-hosted-provider-setup` | Fokussierte Hilfsfunktionen für OpenAI-kompatible self-hosted Provider-Setups | Dieselben Hilfsfunktionen für Discovery/Konfiguration self-hosted Provider |
  | `plugin-sdk/provider-auth-runtime` | Laufzeit-Hilfsfunktionen für Provider-Auth | Hilfsfunktionen zur Laufzeitauflösung von API-Keys |
  | `plugin-sdk/provider-auth-api-key` | Hilfsfunktionen für API-Key-Setups von Providern | Hilfsfunktionen für API-Key-Onboarding/Profilschreiben |
  | `plugin-sdk/provider-auth-result` | Hilfsfunktionen für Provider-Auth-Ergebnisse | Standard-Builder für OAuth-Auth-Ergebnisse |
  | `plugin-sdk/provider-auth-login` | Hilfsfunktionen für interaktiven Provider-Login | Gemeinsame Hilfsfunktionen für interaktiven Login |
  | `plugin-sdk/provider-env-vars` | Hilfsfunktionen für Provider-Umgebungsvariablen | Hilfsfunktionen für Lookup von Umgebungsvariablen für Provider-Auth |
  | `plugin-sdk/provider-model-shared` | Gemeinsame Hilfsfunktionen für Provider-Modelle/Replay | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, gemeinsame Builder für Replay-Richtlinien, Provider-Endpunkt-Hilfen und Hilfsfunktionen zur Normalisierung von Modell-IDs |
  | `plugin-sdk/provider-catalog-shared` | Gemeinsame Hilfsfunktionen für Provider-Kataloge | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patches für Provider-Onboarding | Hilfsfunktionen für Onboarding-Konfiguration |
  | `plugin-sdk/provider-http` | Hilfsfunktionen für Provider-HTTP | Generische Hilfsfunktionen für Provider-HTTP/Endpunkt-Capabilities |
  | `plugin-sdk/provider-web-fetch` | Hilfsfunktionen für Provider-Web-Fetch | Hilfsfunktionen für Registrierung/Cache von Web-Fetch-Providern |
  | `plugin-sdk/provider-web-search-contract` | Hilfsfunktionen für Verträge der Provider-Websuche | Schmale Hilfsfunktionen für Verträge von Websuche-Konfiguration/Zugangsdaten wie `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` und scoped Setter/Getter für Zugangsdaten |
  | `plugin-sdk/provider-web-search` | Hilfsfunktionen für Provider-Websuche | Hilfsfunktionen für Registrierung/Cache/Runtime von Websuche-Providern |
  | `plugin-sdk/provider-tools` | Hilfsfunktionen für Provider-Tool-/Schema-Kompatibilität | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini-Schema-Bereinigung + Diagnostics und xAI-Kompatibilitätshilfen wie `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Hilfsfunktionen für Provider-Nutzung | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` und andere Hilfsfunktionen für Provider-Nutzung |
  | `plugin-sdk/provider-stream` | Hilfsfunktionen für Wrapper von Provider-Streams | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, Typen für Stream-Wrapper und gemeinsame Wrapper-Hilfen für Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Geordnete Async-Queue | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Gemeinsame Medienhilfsfunktionen | Hilfsfunktionen für Abrufen/Transformieren/Speichern von Medien plus Builder für Medien-Nutzlasten |
  | `plugin-sdk/media-generation-runtime` | Gemeinsame Hilfsfunktionen für Mediengenerierung | Gemeinsame Hilfsfunktionen für Failover, Kandidatenauswahl und Meldungen bei fehlenden Modellen für Bild-/Video-/Musikgenerierung |
  | `plugin-sdk/media-understanding` | Hilfsfunktionen für Media Understanding | Typen für Media-Understanding-Provider plus providerseitige Exporte für Bild-/Audio-Hilfsfunktionen |
  | `plugin-sdk/text-runtime` | Gemeinsame Texthilfsfunktionen | Entfernen Assistant-sichtbaren Texts, Rendern/Chunking/Tabellen für Markdown, Hilfsfunktionen für Redaction, Direktiv-Tags, Safe-Text-Utilities und verwandte Hilfsfunktionen für Text/Logging |
  | `plugin-sdk/text-chunking` | Hilfsfunktionen für Text-Chunking | Hilfsfunktion für Chunking ausgehender Texte |
  | `plugin-sdk/speech` | Hilfsfunktionen für Sprache | Typen für Speech-Provider plus providerseitige Hilfsfunktionen für Direktiven, Registry und Validierung |
  | `plugin-sdk/speech-core` | Gemeinsamer Speech-Core | Typen für Speech-Provider, Registry, Direktiven, Normalisierung |
  | `plugin-sdk/realtime-transcription` | Hilfsfunktionen für Echtzeit-Transkription | Provider-Typen und Registry-Hilfsfunktionen |
  | `plugin-sdk/realtime-voice` | Hilfsfunktionen für Echtzeit-Sprache | Provider-Typen und Registry-Hilfsfunktionen |
  | `plugin-sdk/image-generation-core` | Gemeinsamer Core für Bildgenerierung | Typen, Failover, Auth und Registry-Hilfsfunktionen für Bildgenerierung |
  | `plugin-sdk/music-generation` | Hilfsfunktionen für Musikgenerierung | Typen für Musikgenerierungs-Provider/-Anfrage/-Ergebnis |
  | `plugin-sdk/music-generation-core` | Gemeinsamer Core für Musikgenerierung | Typen für Musikgenerierung, Hilfsfunktionen für Failover, Provider-Lookup und Modell-Ref-Parsing |
  | `plugin-sdk/video-generation` | Hilfsfunktionen für Videogenerierung | Typen für Videogenerierungs-Provider/-Anfrage/-Ergebnis |
  | `plugin-sdk/video-generation-core` | Gemeinsamer Core für Videogenerierung | Typen für Videogenerierung, Hilfsfunktionen für Failover, Provider-Lookup und Modell-Ref-Parsing |
  | `plugin-sdk/interactive-runtime` | Hilfsfunktionen für interaktive Antworten | Normalisierung/Reduktion interaktiver Antwortnutzlasten |
  | `plugin-sdk/channel-config-primitives` | Primitive für Kanal-Konfiguration | Schmale Primitive für Kanal-Konfigurationsschemas |
  | `plugin-sdk/channel-config-writes` | Hilfsfunktionen für Schreibvorgänge in Kanal-Konfiguration | Hilfsfunktionen für Autorisierung von Schreibvorgängen in Kanal-Konfiguration |
  | `plugin-sdk/channel-plugin-common` | Gemeinsames Kanal-Prelude | Exporte des gemeinsamen Kanal-Plugin-Preludes |
  | `plugin-sdk/channel-status` | Hilfsfunktionen für Kanalstatus | Gemeinsame Hilfsfunktionen für Snapshots/Zusammenfassungen des Kanalstatus |
  | `plugin-sdk/allowlist-config-edit` | Hilfsfunktionen für Allowlist-Konfiguration | Hilfsfunktionen zum Bearbeiten/Lesen von Allowlist-Konfiguration |
  | `plugin-sdk/group-access` | Hilfsfunktionen für Gruppenzugriff | Gemeinsame Hilfsfunktionen für Entscheidungen zum Gruppenzugriff |
  | `plugin-sdk/direct-dm` | Hilfsfunktionen für direkte DMs | Gemeinsame Hilfsfunktionen für Auth/Guards bei direkten DMs |
  | `plugin-sdk/extension-shared` | Gemeinsame Hilfsfunktionen für Erweiterungen | Primitive für passive Kanäle/Status und Ambient-Proxy-Hilfsfunktionen |
  | `plugin-sdk/webhook-targets` | Hilfsfunktionen für Webhook-Ziele | Hilfsfunktionen für Registry und Route-Install von Webhook-Zielen |
  | `plugin-sdk/webhook-path` | Hilfsfunktionen für Webhook-Pfade | Hilfsfunktionen zur Normalisierung von Webhook-Pfaden |
  | `plugin-sdk/web-media` | Gemeinsame Web-Medienhilfsfunktionen | Hilfsfunktionen zum Laden entfernter/lokaler Medien |
  | `plugin-sdk/zod` | Zod-Re-Export | Re-exportiertes `zod` für Nutzer des Plugin SDK |
  | `plugin-sdk/memory-core` | Gebündelte memory-core-Hilfsfunktionen | Oberfläche für Hilfsfunktionen zu Memory-Manager/Konfiguration/Datei/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Runtime-Fassade für Memory-Engine | Runtime-Fassade für Memory-Index/-Suche |
  | `plugin-sdk/memory-core-host-engine-foundation` | Foundation-Engine für Memory-Host | Exporte der Foundation-Engine für Memory-Host |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Embedding-Engine für Memory-Host | Exporte der Embedding-Engine für Memory-Host |
  | `plugin-sdk/memory-core-host-engine-qmd` | QMD-Engine für Memory-Host | Exporte der QMD-Engine für Memory-Host |
  | `plugin-sdk/memory-core-host-engine-storage` | Storage-Engine für Memory-Host | Exporte der Storage-Engine für Memory-Host |
  | `plugin-sdk/memory-core-host-multimodal` | Multimodale Hilfsfunktionen für Memory-Host | Multimodale Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-query` | Query-Hilfsfunktionen für Memory-Host | Query-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-secret` | Secret-Hilfsfunktionen für Memory-Host | Secret-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-events` | Hilfsfunktionen für Event-Journal des Memory-Hosts | Hilfsfunktionen für Event-Journal des Memory-Hosts |
  | `plugin-sdk/memory-core-host-status` | Status-Hilfsfunktionen für Memory-Host | Status-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-runtime-cli` | CLI-Runtime für Memory-Host | CLI-Runtime-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-runtime-core` | Core-Runtime für Memory-Host | Core-Runtime-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-core-host-runtime-files` | Datei-/Runtime-Hilfsfunktionen für Memory-Host | Datei-/Runtime-Hilfsfunktionen für Memory-Host |
  | `plugin-sdk/memory-host-core` | Alias für Core-Runtime des Memory-Hosts | Anbieterneutraler Alias für Core-Runtime-Hilfsfunktionen des Memory-Hosts |
  | `plugin-sdk/memory-host-events` | Alias für Event-Journal des Memory-Hosts | Anbieterneutraler Alias für Hilfsfunktionen des Event-Journals des Memory-Hosts |
  | `plugin-sdk/memory-host-files` | Alias für Datei-/Runtime des Memory-Hosts | Anbieterneutraler Alias für Datei-/Runtime-Hilfsfunktionen des Memory-Hosts |
  | `plugin-sdk/memory-host-markdown` | Hilfsfunktionen für verwaltetes Markdown | Gemeinsame Hilfsfunktionen für verwaltetes Markdown für memory-nahe Plugins |
  | `plugin-sdk/memory-host-search` | Fassade für aktive Memory-Suche | Lazy Runtime-Fassade für Search-Manager der aktiven Memory-Suche |
  | `plugin-sdk/memory-host-status` | Alias für Status des Memory-Hosts | Anbieterneutraler Alias für Status-Hilfsfunktionen des Memory-Hosts |
  | `plugin-sdk/memory-lancedb` | Gebündelte memory-lancedb-Hilfsfunktionen | Oberfläche für memory-lancedb-Hilfsfunktionen |
  | `plugin-sdk/testing` | Test-Utilities | Testhilfen und Mocks |
</Accordion>

Diese Tabelle ist absichtlich auf die häufige Migrations-Teilmenge beschränkt, nicht auf die vollständige SDK-
Oberfläche. Die vollständige Liste der über 200 Einstiegspunkte befindet sich in
`scripts/lib/plugin-sdk-entrypoints.json`.

Diese Liste enthält weiterhin einige Hilfs-Seams für gebündelte Plugins wie
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` und `plugin-sdk/matrix*`. Diese bleiben für
die Wartung gebündelter Plugins und aus Kompatibilitätsgründen exportiert, werden aber absichtlich
aus der häufigen Migrationstabelle ausgelassen und sind kein empfohlenes Ziel für
neuen Plugin-Code.

Dieselbe Regel gilt für andere Familien gebündelter Hilfsfunktionen wie:

- Browser-Support-Hilfsfunktionen: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- gebündelte Hilfs-/Plugin-Oberflächen wie `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` und `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` stellt derzeit die schmale Token-Hilfsoberfläche
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` und `resolveCopilotApiToken` bereit.

Verwende den schmalsten Import, der zur Aufgabe passt. Wenn du keinen Export finden kannst,
prüfe die Quelle unter `src/plugin-sdk/` oder frage auf Discord nach.

## Zeitplan für die Entfernung

| Wann                   | Was passiert                                                            |
| ---------------------- | ----------------------------------------------------------------------- |
| **Jetzt**              | Veraltete Oberflächen geben Laufzeitwarnungen aus                       |
| **Nächste Major-Version** | Veraltete Oberflächen werden entfernt; Plugins, die sie noch verwenden, schlagen fehl |

Alle Core-Plugins wurden bereits migriert. Externe Plugins sollten vor der
nächsten Major-Version migrieren.

## Warnungen vorübergehend unterdrücken

Setze diese Umgebungsvariablen, während du an der Migration arbeitest:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Dies ist ein temporärer Escape Hatch, keine dauerhafte Lösung.

## Verwandt

- [Erste Schritte](/de/plugins/building-plugins) — dein erstes Plugin erstellen
- [SDK Overview](/de/plugins/sdk-overview) — vollständige Referenz für Subpath-Imports
- [Channel Plugins](/de/plugins/sdk-channel-plugins) — Kanal-Plugins erstellen
- [Provider Plugins](/de/plugins/sdk-provider-plugins) — Provider-Plugins erstellen
- [Plugin Internals](/de/plugins/architecture) — Architektur-Deep-Dive
- [Plugin Manifest](/de/plugins/manifest) — Referenz für das Manifest-Schema
