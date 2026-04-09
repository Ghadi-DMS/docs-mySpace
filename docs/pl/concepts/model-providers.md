---
read_when:
    - Potrzebujesz referencji konfiguracji modeli dla poszczególnych dostawców
    - Chcesz zobaczyć przykładowe konfiguracje lub polecenia CLI do onboardingu dostawców modeli
summary: Przegląd dostawców modeli z przykładowymi konfiguracjami i przepływami CLI
title: Dostawcy modeli
x-i18n:
    generated_at: "2026-04-09T01:29:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 53e3141256781002bbe1d7e7b78724a18d061fcf36a203baae04a091b8c9ea1b
    source_path: concepts/model-providers.md
    workflow: 15
---

# Dostawcy modeli

Ta strona opisuje **dostawców LLM/modeli** (a nie kanały czatu, takie jak WhatsApp/Telegram).
Zasady wyboru modeli znajdziesz w [/concepts/models](/pl/concepts/models).

## Szybkie zasady

- Referencje modeli używają formatu `provider/model` (przykład: `opencode/claude-opus-4-6`).
- Jeśli ustawisz `agents.defaults.models`, stanie się to listą dozwolonych modeli.
- Pomocnicze polecenia CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Zasady awaryjnego działania w runtime, sondy cooldown oraz trwałość nadpisywania sesji
  są udokumentowane w [/concepts/model-failover](/pl/concepts/model-failover).
- `models.providers.*.models[].contextWindow` to natywne metadane modelu;
  `models.providers.*.models[].contextTokens` to efektywny limit runtime.
- Wtyczki dostawców mogą wstrzykiwać katalogi modeli przez `registerProvider({ catalog })`;
  OpenClaw scala ten wynik z `models.providers` przed zapisaniem
  `models.json`.
- Manifesty dostawców mogą deklarować `providerAuthEnvVars` oraz
  `providerAuthAliases`, aby ogólne sondy uwierzytelniania opartego na env i warianty dostawców
  nie musiały ładować runtime wtyczki. Pozostała mapowanie zmiennych env po stronie core
  służy teraz tylko
  dostawcom niebędącym wtyczkami/core oraz kilku przypadkom ogólnego priorytetu, takim
  jak onboarding Anthropic z priorytetem klucza API.
- Wtyczki dostawców mogą także przejmować zachowanie dostawcy w runtime przez
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
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, oraz
  `onModelSelected`.
- Uwaga: `capabilities` runtime dostawcy to współdzielone metadane runnera (rodzina dostawcy,
  specyfika transkryptu/narzędzi, wskazówki dotyczące transportu/cache). To nie jest
  to samo co [publiczny model możliwości](/pl/plugins/architecture#public-capability-model),
  który opisuje, co rejestruje wtyczka (wnioskowanie tekstowe, mowa itd.).

## Zachowanie dostawcy należące do wtyczki

Wtyczki dostawców mogą teraz przejmować większość logiki specyficznej dla dostawcy, podczas gdy OpenClaw zachowuje
ogólną pętlę wnioskowania.

Typowy podział:

- `auth[].run` / `auth[].runNonInteractive`: dostawca obsługuje onboarding/logowanie
  dla `openclaw onboard`, `openclaw models auth` i konfiguracji bez interakcji
- `wizard.setup` / `wizard.modelPicker`: dostawca obsługuje etykiety wyboru uwierzytelniania,
  stare aliasy, wskazówki dotyczące listy dozwolonych modeli podczas onboardingu oraz wpisy konfiguracji w selektorach onboardingu/modeli
- `catalog`: dostawca pojawia się w `models.providers`
- `normalizeModelId`: dostawca normalizuje stare/podglądowe identyfikatory modeli przed
  wyszukiwaniem lub kanonizacją
- `normalizeTransport`: dostawca normalizuje `api` / `baseUrl` rodziny transportu
  przed ogólnym składaniem modelu; OpenClaw najpierw sprawdza dopasowanego dostawcę,
  a następnie inne wtyczki dostawców obsługujące hooki, dopóki jedna z nich rzeczywiście nie zmieni
  transportu
- `normalizeConfig`: dostawca normalizuje konfigurację `models.providers.<id>` przed
  użyciem jej przez runtime; OpenClaw najpierw sprawdza dopasowanego dostawcę, a potem inne
  wtyczki dostawców obsługujące hooki, dopóki jedna z nich rzeczywiście nie zmieni konfiguracji. Jeśli żaden
  hook dostawcy nie przepisze konfiguracji, dołączone helpery rodziny Google nadal
  normalizują obsługiwane wpisy dostawców Google.
- `applyNativeStreamingUsageCompat`: dostawca stosuje zgodnościowe przepisania natywnego zużycia strumieniowego oparte na endpointach dla dostawców konfiguracyjnych
- `resolveConfigApiKey`: dostawca rozwiązuje uwierzytelnianie oparte na markerach env dla dostawców konfiguracyjnych
  bez wymuszania pełnego ładowania auth runtime. `amazon-bedrock` ma tu także
  wbudowany resolver markerów env AWS, mimo że uwierzytelnianie runtime Bedrock używa
  domyślnego łańcucha AWS SDK.
- `resolveSyntheticAuth`: dostawca może ujawniać dostępność lokalnego/self-hosted lub innego
  uwierzytelniania opartego na konfiguracji bez utrwalania sekretów w postaci jawnego tekstu
- `shouldDeferSyntheticProfileAuth`: dostawca może oznaczyć zapisane syntetyczne placeholdery profili
  jako mające niższy priorytet niż uwierzytelnianie oparte na env/konfiguracji
- `resolveDynamicModel`: dostawca akceptuje identyfikatory modeli, których nie ma jeszcze
  w lokalnym statycznym katalogu
- `prepareDynamicModel`: dostawca wymaga odświeżenia metadanych przed ponowną próbą
  dynamicznego rozpoznania
- `normalizeResolvedModel`: dostawca wymaga przepisania transportu lub base URL
- `contributeResolvedModelCompat`: dostawca dodaje flagi zgodności dla swoich
  modeli producenta nawet wtedy, gdy docierają przez inny zgodny transport
- `capabilities`: dostawca publikuje specyfikę transkryptu/narzędzi/rodziny dostawców
- `normalizeToolSchemas`: dostawca czyści schematy narzędzi, zanim zobaczy je osadzony
  runner
- `inspectToolSchemas`: dostawca ujawnia ostrzeżenia dotyczące schematów specyficznych dla transportu
  po normalizacji
- `resolveReasoningOutputMode`: dostawca wybiera natywne lub otagowane
  kontrakty wyjścia reasoning
- `prepareExtraParams`: dostawca ustawia wartości domyślne lub normalizuje parametry żądań dla danego modelu
- `createStreamFn`: dostawca zastępuje zwykłą ścieżkę strumieniowania całkowicie
  niestandardowym transportem
- `wrapStreamFn`: dostawca stosuje opakowania zgodności żądania/nagłówków/body/modelu
- `resolveTransportTurnState`: dostawca dostarcza natywne nagłówki transportu
  lub metadane dla pojedynczego turnu
- `resolveWebSocketSessionPolicy`: dostawca dostarcza natywne nagłówki sesji WebSocket
  lub politykę cooldown sesji
- `createEmbeddingProvider`: dostawca przejmuje zachowanie embeddingów pamięci, gdy
  powinno ono należeć do wtyczki dostawcy zamiast do przełącznika embeddingów w core
- `formatApiKey`: dostawca formatuje zapisane profile auth do postaci ciągu
  `apiKey`, której oczekuje transport w runtime
- `refreshOAuth`: dostawca obsługuje odświeżanie OAuth, gdy współdzielone
  refreshery `pi-ai` nie wystarczają
- `buildAuthDoctorHint`: dostawca dopisuje wskazówki naprawcze, gdy odświeżenie OAuth
  się nie powiedzie
- `matchesContextOverflowError`: dostawca rozpoznaje błędy przepełnienia okna kontekstu
  specyficzne dla dostawcy, które umknęłyby ogólnym heurystykom
- `classifyFailoverReason`: dostawca mapuje surowe błędy transportu/API specyficzne dla dostawcy
  na przyczyny failover, takie jak limit szybkości lub przeciążenie
- `isCacheTtlEligible`: dostawca decyduje, które identyfikatory modeli upstream obsługują TTL cache promptów
- `buildMissingAuthMessage`: dostawca zastępuje ogólny błąd magazynu auth
  wskazówką odzyskiwania specyficzną dla dostawcy
- `suppressBuiltInModel`: dostawca ukrywa nieaktualne wiersze upstream i może zwrócić
  błąd należący do producenta przy bezpośrednich niepowodzeniach rozpoznania
- `augmentModelCatalog`: dostawca dopisuje syntetyczne/końcowe wiersze katalogu po
  odkryciu i scaleniu konfiguracji
- `isBinaryThinking`: dostawca obsługuje UX binarnego thinking w trybie włącz/wyłącz
- `supportsXHighThinking`: dostawca włącza `xhigh` dla wybranych modeli
- `resolveDefaultThinkingLevel`: dostawca ustala domyślną politykę `/think` dla
  rodziny modeli
- `applyConfigDefaults`: dostawca stosuje globalne wartości domyślne specyficzne dla dostawcy
  podczas materializacji konfiguracji na podstawie trybu auth, env lub rodziny modeli
- `isModernModelRef`: dostawca obsługuje dopasowanie preferowanych modeli dla live/smoke
- `prepareRuntimeAuth`: dostawca zamienia skonfigurowane poświadczenie na krótkożyjący
  token runtime
- `resolveUsageAuth`: dostawca rozwiązuje poświadczenia użycia/kwoty dla `/usage`
  i powiązanych powierzchni statusu/raportowania
- `fetchUsageSnapshot`: dostawca obsługuje pobieranie/parsing endpointu użycia, podczas gdy
  core nadal obsługuje powłokę podsumowania i formatowanie
- `onModelSelected`: dostawca uruchamia efekty uboczne po wyborze modelu, takie jak
  telemetria lub bookkeeping sesji należący do dostawcy

Obecne dołączone przykłady:

- `anthropic`: fallback zgodności przyszłej dla Claude 4.6, wskazówki naprawy auth, pobieranie
  endpointu użycia, metadane cache-TTL/rodziny dostawców oraz globalne
  wartości domyślne konfiguracji zależne od auth
- `amazon-bedrock`: należące do dostawcy dopasowanie przepełnienia kontekstu i klasyfikacja
  przyczyn failover dla błędów throttle/not-ready specyficznych dla Bedrock, a także
  współdzielona rodzina replay `anthropic-by-model` dla ochrony polityki replay wyłącznie dla Claude
  na ruchu Anthropic
- `anthropic-vertex`: zabezpieczenia polityki replay tylko dla Claude na ruchu
  `anthropic-message`
- `openrouter`: przekazywane identyfikatory modeli, opakowania żądań, wskazówki dotyczące możliwości dostawcy,
  sanityzacja podpisów thought Gemini na ruchu proxy Gemini, proxy
  wstrzykiwanie reasoning przez rodzinę strumienia `openrouter-thinking`, przekazywanie
  metadanych routingu oraz polityka cache-TTL
- `github-copilot`: onboarding/logowanie urządzenia, fallback zgodności przyszłej modeli,
  wskazówki transkryptu Claude-thinking, wymiana tokena runtime oraz pobieranie endpointu
  użycia
- `openai`: fallback zgodności przyszłej dla GPT-5.4, bezpośrednia normalizacja
  transportu OpenAI, wskazówki brakującego auth świadome Codex, tłumienie Spark, syntetyczne
  wiersze katalogu OpenAI/Codex, polityka thinking/live-model, normalizacja aliasów tokenów użycia
  (`input` / `output` oraz rodziny `prompt` / `completion`), współdzielona
  rodzina strumienia `openai-responses-defaults` dla natywnych wrapperów OpenAI/Codex,
  metadane rodziny dostawcy, dołączona rejestracja dostawcy generowania obrazów
  dla `gpt-image-1` oraz dołączona rejestracja dostawcy generowania wideo
  dla `sora-2`
- `google` i `google-gemini-cli`: fallback zgodności przyszłej dla Gemini 3.1,
  natywna walidacja replay Gemini, sanityzacja replay podczas bootstrapu, tryb
  otagowanego wyjścia reasoning, dopasowanie nowoczesnych modeli, dołączona rejestracja dostawcy generowania obrazów
  dla modeli Gemini image-preview oraz dołączona
  rejestracja dostawcy generowania wideo dla modeli Veo; Gemini CLI OAuth także
  obsługuje formatowanie tokenów profilu auth, parsowanie tokenów użycia oraz pobieranie endpointu kwot
  dla powierzchni użycia
- `moonshot`: współdzielony transport, normalizacja payload thinking należąca do wtyczki
- `kilocode`: współdzielony transport, nagłówki żądań należące do wtyczki, normalizacja payload
  reasoning, sanityzacja podpisów thought proxy-Gemini oraz polityka cache-TTL
- `zai`: fallback zgodności przyszłej dla GLM-5, domyślne `tool_stream`, polityka cache-TTL,
  polityka binarnego thinking/live-model oraz auth użycia + pobieranie kwot;
  nieznane identyfikatory `glm-5*` są syntetyzowane na podstawie dołączonego szablonu `glm-4.7`
- `xai`: natywna normalizacja transportu Responses, przepisania aliasów `/fast` dla
  szybkich wariantów Grok, domyślne `tool_stream`, czyszczenie schematów narzędzi /
  payload reasoning specyficzne dla xAI oraz dołączona rejestracja dostawcy generowania wideo
  dla `grok-imagine-video`
- `mistral`: metadane możliwości należące do wtyczki
- `opencode` i `opencode-go`: metadane możliwości należące do wtyczki oraz
  sanityzacja podpisów thought proxy-Gemini
- `alibaba`: katalog wideo należący do wtyczki dla bezpośrednich referencji modeli Wan
  takich jak `alibaba/wan2.6-t2v`
- `byteplus`: katalogi należące do wtyczki oraz dołączona rejestracja dostawcy generowania wideo
  dla modeli text-to-video/image-to-video Seedance
- `fal`: dołączona rejestracja dostawcy generowania wideo dla hostowanych modeli wideo innych firm
  oraz rejestracja dostawcy generowania obrazów dla modeli obrazów FLUX, a także dołączona
  rejestracja dostawcy generowania wideo dla hostowanych modeli wideo innych firm
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` i `volcengine`:
  tylko katalogi należące do wtyczki
- `qwen`: katalogi należące do wtyczki dla modeli tekstowych oraz współdzielone
  rejestracje dostawców media-understanding i generowania wideo dla ich
  powierzchni multimodalnych; generowanie wideo Qwen używa standardowych endpointów DashScope video
  z dołączonymi modelami Wan, takimi jak `wan2.6-t2v` i `wan2.7-r2v`
- `runway`: rejestracja dostawcy generowania wideo należąca do wtyczki dla natywnych
  modeli opartych na zadaniach Runway, takich jak `gen4.5`
- `minimax`: katalogi należące do wtyczki, dołączona rejestracja dostawcy generowania wideo
  dla modeli wideo Hailuo, dołączona rejestracja dostawcy generowania obrazów
  dla `image-01`, hybrydowy wybór polityki replay Anthropic/OpenAI
  oraz logika auth/snapshot użycia
- `together`: katalogi należące do wtyczki oraz dołączona rejestracja dostawcy generowania wideo
  dla modeli wideo Wan
- `xiaomi`: katalogi należące do wtyczki oraz logika auth/snapshot użycia

Dołączona wtyczka `openai` obsługuje teraz oba identyfikatory dostawcy: `openai` i
`openai-codex`.

To obejmuje dostawców, którzy nadal mieszczą się w normalnych transportach OpenClaw. Dostawca,
który potrzebuje całkowicie niestandardowego wykonawcy żądań, to osobna, głębsza
powierzchnia rozszerzeń.

## Rotacja kluczy API

- Obsługuje ogólną rotację dla wybranych dostawców.
- Skonfiguruj wiele kluczy przez:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (pojedyncze nadpisanie live, najwyższy priorytet)
  - `<PROVIDER>_API_KEYS` (lista rozdzielana przecinkami lub średnikami)
  - `<PROVIDER>_API_KEY` (główny klucz)
  - `<PROVIDER>_API_KEY_*` (lista numerowana, np. `<PROVIDER>_API_KEY_1`)
- Dla dostawców Google jako fallback uwzględniany jest także `GOOGLE_API_KEY`.
- Kolejność wyboru kluczy zachowuje priorytet i deduplikuje wartości.
- Żądania są ponawiane z następnym kluczem tylko przy odpowiedziach z limitem szybkości (na
  przykład `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` lub okresowych komunikatach o limicie użycia).
- Błędy inne niż limit szybkości kończą się natychmiast; nie podejmuje się rotacji kluczy.
- Gdy wszystkie kandydackie klucze zawiodą, zwracany jest końcowy błąd z ostatniej próby.

## Wbudowani dostawcy (katalog pi-ai)

OpenClaw jest dostarczany z katalogiem pi‑ai. Ci dostawcy nie wymagają
konfiguracji `models.providers`; wystarczy ustawić auth i wybrać model.

### OpenAI

- Dostawca: `openai`
- Auth: `OPENAI_API_KEY`
- Opcjonalna rotacja: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, oraz `OPENCLAW_LIVE_OPENAI_KEY` (pojedyncze nadpisanie)
- Przykładowe modele: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Domyślny transport to `auto` (najpierw WebSocket, fallback do SSE)
- Nadpisanie dla modelu przez `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` lub `"auto"`)
- Rozgrzewanie OpenAI Responses WebSocket jest domyślnie włączone przez `params.openaiWsWarmup` (`true`/`false`)
- Przetwarzanie priorytetowe OpenAI można włączyć przez `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` i `params.fastMode` mapują bezpośrednie żądania Responses `openai/*` do `service_tier=priority` na `api.openai.com`
- Użyj `params.serviceTier`, jeśli chcesz jawnie ustawić poziom zamiast używać współdzielonego przełącznika `/fast`
- Ukryte nagłówki atrybucji OpenClaw (`originator`, `version`,
  `User-Agent`) są stosowane tylko do natywnego ruchu OpenAI do `api.openai.com`, a nie
  do ogólnych proxy zgodnych z OpenAI
- Natywne trasy OpenAI zachowują też `store` dla Responses, wskazówki cache promptów oraz
  kształtowanie payload zgodności reasoning OpenAI; trasy proxy tego nie robią
- `openai/gpt-5.3-codex-spark` jest celowo ukryty w OpenClaw, ponieważ live API OpenAI go odrzuca; Spark jest traktowany wyłącznie jako Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Dostawca: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- Opcjonalna rotacja: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, oraz `OPENCLAW_LIVE_ANTHROPIC_KEY` (pojedyncze nadpisanie)
- Przykładowy model: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Bezpośrednie publiczne żądania Anthropic obsługują współdzielony przełącznik `/fast` i `params.fastMode`, zarówno dla ruchu uwierzytelnionego kluczem API, jak i OAuth wysyłanego do `api.anthropic.com`; OpenClaw mapuje to na `service_tier` Anthropic (`auto` vs `standard_only`)
- Uwaga dotycząca Anthropic: pracownicy Anthropic poinformowali nas, że użycie Claude CLI w stylu OpenClaw jest ponownie dozwolone, więc OpenClaw traktuje ponowne użycie Claude CLI i użycie `claude -p` jako dozwolone dla tej integracji, chyba że Anthropic opublikuje nową politykę.
- Token konfiguracyjny Anthropic pozostaje dostępną i wspieraną ścieżką tokenu OpenClaw, ale OpenClaw teraz preferuje ponowne użycie Claude CLI i `claude -p`, gdy są dostępne.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Dostawca: `openai-codex`
- Auth: OAuth (ChatGPT)
- Przykładowy model: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` lub `openclaw models auth login --provider openai-codex`
- Domyślny transport to `auto` (najpierw WebSocket, fallback do SSE)
- Nadpisanie dla modelu przez `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` lub `"auto"`)
- `params.serviceTier` jest także przekazywane w natywnych żądaniach Codex Responses (`chatgpt.com/backend-api`)
- Ukryte nagłówki atrybucji OpenClaw (`originator`, `version`,
  `User-Agent`) są dołączane tylko do natywnego ruchu Codex do
  `chatgpt.com/backend-api`, a nie do ogólnych proxy zgodnych z OpenAI
- Współdzieli ten sam przełącznik `/fast` i konfigurację `params.fastMode` co bezpośrednie `openai/*`; OpenClaw mapuje to na `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` pozostaje dostępny, gdy katalog OAuth Codex go udostępnia; zależy od uprawnień
- `openai-codex/gpt-5.4` zachowuje natywne `contextWindow = 1050000` i domyślne runtime `contextTokens = 272000`; nadpisz limit runtime przez `models.providers.openai-codex.models[].contextTokens`
- Uwaga dotycząca polityki: OAuth OpenAI Codex jest jawnie wspierany dla zewnętrznych narzędzi/przepływów pracy, takich jak OpenClaw.

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

### Inne hostowane opcje w stylu subskrypcyjnym

- [Qwen Cloud](/pl/providers/qwen): powierzchnia dostawcy Qwen Cloud oraz mapowanie endpointów Alibaba DashScope i Coding Plan
- [MiniMax](/pl/providers/minimax): dostęp przez OAuth MiniMax Coding Plan lub klucz API
- [GLM Models](/pl/providers/glm): Z.AI Coding Plan lub ogólne endpointy API

### OpenCode

- Auth: `OPENCODE_API_KEY` (lub `OPENCODE_ZEN_API_KEY`)
- Dostawca runtime Zen: `opencode`
- Dostawca runtime Go: `opencode-go`
- Przykładowe modele: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` lub `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (klucz API)

- Dostawca: `google`
- Auth: `GEMINI_API_KEY`
- Opcjonalna rotacja: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, fallback `GOOGLE_API_KEY` oraz `OPENCLAW_LIVE_GEMINI_KEY` (pojedyncze nadpisanie)
- Przykładowe modele: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Zgodność: starsza konfiguracja OpenClaw używająca `google/gemini-3.1-flash-preview` jest normalizowana do `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Bezpośrednie uruchomienia Gemini akceptują także `agents.defaults.models["google/<model>"].params.cachedContent`
  (lub starsze `cached_content`) do przekazania natywnego dla dostawcy
  uchwytu `cachedContents/...`; trafienia cache Gemini są ujawniane jako OpenClaw `cacheRead`

### Google Vertex i Gemini CLI

- Dostawcy: `google-vertex`, `google-gemini-cli`
- Auth: Vertex używa gcloud ADC; Gemini CLI używa własnego przepływu OAuth
- Ostrzeżenie: OAuth Gemini CLI w OpenClaw to nieoficjalna integracja. Niektórzy użytkownicy zgłaszali ograniczenia kont Google po używaniu klientów zewnętrznych. Zapoznaj się z warunkami Google i używaj konta niekrytycznego, jeśli zdecydujesz się kontynuować.
- OAuth Gemini CLI jest dostarczane jako część dołączonej wtyczki `google`.
  - Najpierw zainstaluj Gemini CLI:
    - `brew install gemini-cli`
    - lub `npm install -g @google/gemini-cli`
  - Włącz: `openclaw plugins enable google`
  - Zaloguj: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Domyślny model: `google-gemini-cli/gemini-3-flash-preview`
  - Uwaga: **nie** wklejasz client id ani secret do `openclaw.json`. Przepływ logowania CLI zapisuje
    tokeny w profilach auth na hoście gateway.
  - Jeśli żądania nie działają po zalogowaniu, ustaw `GOOGLE_CLOUD_PROJECT` lub `GOOGLE_CLOUD_PROJECT_ID` na hoście gateway.
  - Odpowiedzi JSON Gemini CLI są parsowane z `response`; użycie korzysta z fallback do
    `stats`, a `stats.cached` jest normalizowane do OpenClaw `cacheRead`.

### Z.AI (GLM)

- Dostawca: `zai`
- Auth: `ZAI_API_KEY`
- Przykładowy model: `zai/glm-5.1`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Aliasy: `z.ai/*` i `z-ai/*` są normalizowane do `zai/*`
  - `zai-api-key` automatycznie wykrywa pasujący endpoint Z.AI; `zai-coding-global`, `zai-coding-cn`, `zai-global` i `zai-cn` wymuszają konkretną powierzchnię

### Vercel AI Gateway

- Dostawca: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- Przykładowy model: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Dostawca: `kilocode`
- Auth: `KILOCODE_API_KEY`
- Przykładowy model: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Base URL: `https://api.kilo.ai/api/gateway/`
- Statyczny katalog fallback zawiera `kilocode/kilo/auto`; live
  odkrywanie pod `https://api.kilo.ai/api/gateway/models` może dalej rozszerzać katalog runtime.
- Dokładny routing upstream za `kilocode/kilo/auto` należy do Kilo Gateway,
  a nie jest zakodowany na sztywno w OpenClaw.

Szczegóły konfiguracji znajdziesz w [/providers/kilocode](/pl/providers/kilocode).

### Inne dołączone wtyczki dostawców

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Przykładowy model: `openrouter/auto`
- OpenClaw stosuje udokumentowane nagłówki atrybucji aplikacji OpenRouter tylko wtedy, gdy
  żądanie faktycznie trafia do `openrouter.ai`
- Specyficzne dla OpenRouter znaczniki Anthropic `cache_control` są podobnie ograniczone do
  zweryfikowanych tras OpenRouter, a nie dowolnych adresów proxy
- OpenRouter pozostaje na ścieżce proxy zgodnej z OpenAI, więc natywne
  kształtowanie żądań tylko dla OpenAI (`serviceTier`, `store` dla Responses,
  wskazówki cache promptów, payloady zgodności reasoning OpenAI) nie jest przekazywane
- Referencje OpenRouter oparte na Gemini zachowują tylko sanityzację podpisów thought proxy-Gemini;
  natywna walidacja replay Gemini i przepisania bootstrap pozostają wyłączone
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Przykładowy model: `kilocode/kilo/auto`
- Referencje Kilo oparte na Gemini zachowują tę samą ścieżkę sanityzacji podpisów thought
  proxy-Gemini; `kilocode/kilo/auto` i inne wskazówki proxy bez obsługi reasoning
  pomijają wstrzykiwanie reasoning do proxy
- MiniMax: `minimax` (klucz API) i `minimax-portal` (OAuth)
- Auth: `MINIMAX_API_KEY` dla `minimax`; `MINIMAX_OAUTH_TOKEN` lub `MINIMAX_API_KEY` dla `minimax-portal`
- Przykładowy model: `minimax/MiniMax-M2.7` lub `minimax-portal/MiniMax-M2.7`
- Onboarding MiniMax/konfiguracja klucza API zapisuje jawne definicje modeli M2.7 z
  `input: ["text", "image"]`; dołączony katalog dostawcy utrzymuje referencje czatu
  jako tylko tekstowe, dopóki konfiguracja tego dostawcy nie zostanie zmaterializowana
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Przykładowy model: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` lub `KIMICODE_API_KEY`)
- Przykładowy model: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Przykładowy model: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` lub `DASHSCOPE_API_KEY`)
- Przykładowy model: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Przykładowy model: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Przykładowe modele: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Przykładowy model: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Przykładowy model: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` lub `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Przykładowy model: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Przykładowy model: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Natywne dołączone żądania xAI używają ścieżki xAI Responses
  - `/fast` lub `params.fastMode: true` przepisuje `grok-3`, `grok-3-mini`,
    `grok-4` i `grok-4-0709` do ich wariantów `*-fast`
  - `tool_stream` jest domyślnie włączone; ustaw
    `agents.defaults.models["xai/<model>"].params.tool_stream` na `false`, aby
    je wyłączyć
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Przykładowy model: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - Modele GLM w Cerebras używają identyfikatorów `zai-glm-4.7` i `zai-glm-4.6`.
  - Base URL zgodny z OpenAI: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Przykładowy model Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Zobacz [Hugging Face (Inference)](/pl/providers/huggingface).

## Dostawcy przez `models.providers` (własny/base URL)

Użyj `models.providers` (lub `models.json`), aby dodać **własnych** dostawców lub
proxy zgodne z OpenAI/Anthropic.

Wiele dołączonych poniżej wtyczek dostawców publikuje już domyślny katalog.
Używaj jawnych wpisów `models.providers.<id>` tylko wtedy, gdy chcesz nadpisać
domyślny base URL, nagłówki lub listę modeli.

### Moonshot AI (Kimi)

Moonshot jest dostarczany jako dołączona wtyczka dostawcy. Używaj wbudowanego dostawcy
domyślnie i dodawaj jawny wpis `models.providers.moonshot` tylko wtedy, gdy
musisz nadpisać base URL lub metadane modelu:

- Dostawca: `moonshot`
- Auth: `MOONSHOT_API_KEY`
- Przykładowy model: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` lub `openclaw onboard --auth-choice moonshot-api-key-cn`

Identyfikatory modeli Kimi K2:

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

Kimi Coding używa endpointu Moonshot AI zgodnego z Anthropic:

- Dostawca: `kimi`
- Auth: `KIMI_API_KEY`
- Przykładowy model: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

Starszy `kimi/k2p5` pozostaje akceptowanym identyfikatorem modelu dla zgodności.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) zapewnia dostęp do Doubao i innych modeli w Chinach.

- Dostawca: `volcengine` (coding: `volcengine-plan`)
- Auth: `VOLCANO_ENGINE_API_KEY`
- Przykładowy model: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

Onboarding domyślnie używa powierzchni coding, ale ogólny katalog `volcengine/*`
jest rejestrowany jednocześnie.

W selektorach modeli onboarding/configure model wybór auth Volcengine preferuje zarówno
wiersze `volcengine/*`, jak i `volcengine-plan/*`. Jeśli te modele nie są jeszcze załadowane,
OpenClaw wraca do niefiltrowanego katalogu zamiast pokazywać pusty
selektor ograniczony do dostawcy.

Dostępne modele:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Modele coding (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (International)

BytePlus ARK zapewnia międzynarodowym użytkownikom dostęp do tych samych modeli co Volcano Engine.

- Dostawca: `byteplus` (coding: `byteplus-plan`)
- Auth: `BYTEPLUS_API_KEY`
- Przykładowy model: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

Onboarding domyślnie używa powierzchni coding, ale ogólny katalog `byteplus/*`
jest rejestrowany jednocześnie.

W selektorach modeli onboarding/configure model wybór auth BytePlus preferuje zarówno
wiersze `byteplus/*`, jak i `byteplus-plan/*`. Jeśli te modele nie są jeszcze załadowane,
OpenClaw wraca do niefiltrowanego katalogu zamiast pokazywać pusty
selektor ograniczony do dostawcy.

Dostępne modele:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Modele coding (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic udostępnia modele zgodne z Anthropic za dostawcą `synthetic`:

- Dostawca: `synthetic`
- Auth: `SYNTHETIC_API_KEY`
- Przykładowy model: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
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

MiniMax jest konfigurowany przez `models.providers`, ponieważ używa niestandardowych endpointów:

- MiniMax OAuth (Global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax klucz API (Global): `--auth-choice minimax-global-api`
- MiniMax klucz API (CN): `--auth-choice minimax-cn-api`
- Auth: `MINIMAX_API_KEY` dla `minimax`; `MINIMAX_OAUTH_TOKEN` lub
  `MINIMAX_API_KEY` dla `minimax-portal`

Szczegóły konfiguracji, opcje modeli i fragmenty konfiguracji znajdziesz w [/providers/minimax](/pl/providers/minimax).

Na ścieżce strumieniowania MiniMax zgodnej z Anthropic OpenClaw domyślnie wyłącza thinking,
chyba że jawnie go ustawisz, a `/fast on` przepisuje
`MiniMax-M2.7` na `MiniMax-M2.7-highspeed`.

Podział możliwości należących do wtyczki:

- Domyślne ustawienia tekst/czat pozostają na `minimax/MiniMax-M2.7`
- Generowanie obrazów to `minimax/image-01` lub `minimax-portal/image-01`
- Rozumienie obrazów to należący do wtyczki `MiniMax-VL-01` na obu ścieżkach auth MiniMax
- Wyszukiwanie w sieci pozostaje przy identyfikatorze dostawcy `minimax`

### Ollama

Ollama jest dostarczany jako dołączona wtyczka dostawcy i używa natywnego API Ollama:

- Dostawca: `ollama`
- Auth: niewymagane (serwer lokalny)
- Przykładowy model: `ollama/llama3.3`
- Instalacja: [https://ollama.com/download](https://ollama.com/download)

```bash
# Zainstaluj Ollama, a następnie pobierz model:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama jest wykrywany lokalnie pod `http://127.0.0.1:11434`, gdy włączysz go przez
`OLLAMA_API_KEY`, a dołączona wtyczka dostawcy dodaje Ollama bezpośrednio do
`openclaw onboard` i selektora modeli. Zobacz [/providers/ollama](/pl/providers/ollama),
aby poznać onboarding, tryb chmura/lokalny i konfigurację niestandardową.

### vLLM

vLLM jest dostarczany jako dołączona wtyczka dostawcy dla lokalnych/self-hosted
serwerów zgodnych z OpenAI:

- Dostawca: `vllm`
- Auth: opcjonalne (zależy od serwera)
- Domyślny base URL: `http://127.0.0.1:8000/v1`

Aby włączyć lokalne auto-discovery (dowolna wartość działa, jeśli serwer nie wymusza auth):

```bash
export VLLM_API_KEY="vllm-local"
```

Następnie ustaw model (zamień na jeden z identyfikatorów zwracanych przez `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Szczegóły znajdziesz w [/providers/vllm](/pl/providers/vllm).

### SGLang

SGLang jest dostarczany jako dołączona wtyczka dostawcy dla szybkich serwerów self-hosted
zgodnych z OpenAI:

- Dostawca: `sglang`
- Auth: opcjonalne (zależy od serwera)
- Domyślny base URL: `http://127.0.0.1:30000/v1`

Aby włączyć lokalne auto-discovery (dowolna wartość działa, jeśli serwer nie
wymusza auth):

```bash
export SGLANG_API_KEY="sglang-local"
```

Następnie ustaw model (zamień na jeden z identyfikatorów zwracanych przez `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Szczegóły znajdziesz w [/providers/sglang](/pl/providers/sglang).

### Lokalne proxy (LM Studio, vLLM, LiteLLM itd.)

Przykład (zgodny z OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Lokalny" } },
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

Uwagi:

- Dla niestandardowych dostawców `reasoning`, `input`, `cost`, `contextWindow` i `maxTokens` są opcjonalne.
  Po pominięciu OpenClaw przyjmuje domyślnie:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Zalecane: ustaw jawne wartości zgodne z limitami Twojego proxy/modelu.
- Dla `api: "openai-completions"` na nienatywnych endpointach (dowolny niepusty `baseUrl`, którego host nie jest `api.openai.com`), OpenClaw wymusza `compat.supportsDeveloperRole: false`, aby uniknąć błędów dostawcy 400 dla nieobsługiwanych ról `developer`.
- Trasy proxy zgodne z OpenAI również pomijają natywne kształtowanie żądań tylko dla OpenAI:
  brak `service_tier`, brak `store` dla Responses, brak wskazówek cache promptów, brak
  kształtowania payload zgodności reasoning OpenAI i brak ukrytych nagłówków
  atrybucji OpenClaw.
- Jeśli `baseUrl` jest pusty/pominięty, OpenClaw zachowuje domyślne zachowanie OpenAI (które jest rozwiązywane do `api.openai.com`).
- Dla bezpieczeństwa jawne `compat.supportsDeveloperRole: true` nadal jest nadpisywane na nienatywnych endpointach `openai-completions`.

## Przykłady CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Zobacz też: [/gateway/configuration](/pl/gateway/configuration), aby poznać pełne przykłady konfiguracji.

## Powiązane

- [Models](/pl/concepts/models) — konfiguracja modeli i aliasy
- [Model Failover](/pl/concepts/model-failover) — łańcuchy fallback i zachowanie ponownych prób
- [Configuration Reference](/pl/gateway/configuration-reference#agent-defaults) — klucze konfiguracji modeli
- [Providers](/pl/providers) — przewodniki konfiguracji dla poszczególnych dostawców
