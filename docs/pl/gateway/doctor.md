---
read_when:
    - Dodajesz lub modyfikujesz migracje Doctor
    - Wprowadzasz niekompatybilne zmiany konfiguracji
summary: 'Komenda Doctor: kontrole kondycji, migracje konfiguracji i kroki naprawcze'
title: Doctor
x-i18n:
    generated_at: "2026-04-09T01:29:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 75d321bd1ad0e16c29f2382e249c51edfc3a8d33b55bdceea39e7dbcd4901fce
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` to narzędzie naprawy i migracji dla OpenClaw. Naprawia nieaktualną
konfigurację i stan, sprawdza kondycję oraz podaje konkretne kroki naprawcze.

## Szybki start

```bash
openclaw doctor
```

### Tryb bezgłowy / automatyzacja

```bash
openclaw doctor --yes
```

Akceptuje wartości domyślne bez pytania (w tym kroki naprawy restartu/usługi/sandboxa, gdy mają zastosowanie).

```bash
openclaw doctor --repair
```

Stosuje zalecane naprawy bez pytania (naprawy + restarty tam, gdzie to bezpieczne).

```bash
openclaw doctor --repair --force
```

Stosuje także agresywne naprawy (nadpisuje niestandardowe konfiguracje supervisora).

```bash
openclaw doctor --non-interactive
```

Uruchamia bez monitów i stosuje tylko bezpieczne migracje (normalizacja konfiguracji + przenoszenie stanu na dysku). Pomija działania restartu/usługi/sandboxa wymagające potwierdzenia człowieka.
Migracje starszego stanu uruchamiają się automatycznie po ich wykryciu.

```bash
openclaw doctor --deep
```

Skanuje usługi systemowe w poszukiwaniu dodatkowych instalacji gateway (launchd/systemd/schtasks).

Jeśli chcesz przejrzeć zmiany przed zapisaniem, najpierw otwórz plik konfiguracji:

```bash
cat ~/.openclaw/openclaw.json
```

## Co robi (podsumowanie)

- Opcjonalna aktualizacja przed uruchomieniem dla instalacji git (tylko interaktywnie).
- Kontrola aktualności protokołu UI (przebudowuje Control UI, gdy schemat protokołu jest nowszy).
- Kontrola kondycji + monit o restart.
- Podsumowanie stanu Skills (kwalifikujące się/brakujące/zablokowane) i stanu pluginów.
- Normalizacja konfiguracji dla starszych wartości.
- Migracja konfiguracji Talk ze starszych płaskich pól `talk.*` do `talk.provider` + `talk.providers.<provider>`.
- Kontrole migracji przeglądarki dla starszych konfiguracji rozszerzenia Chrome i gotowości Chrome MCP.
- Ostrzeżenia o nadpisaniach dostawcy OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Ostrzeżenia o przysłanianiu OAuth Codex (`models.providers.openai-codex`).
- Kontrola wymagań wstępnych TLS dla profili OAuth OpenAI Codex.
- Migracja starszego stanu na dysku (sessions/agent dir/uwierzytelnianie WhatsApp).
- Migracja starszych kluczy kontraktów manifestu pluginów (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migracja starszego magazynu cron (`jobId`, `schedule.cron`, pola delivery/payload na najwyższym poziomie, `provider` w payload, proste zadania zapasowe webhook z `notify: true`).
- Kontrola plików blokady sesji i czyszczenie przeterminowanych blokad.
- Kontrole integralności i uprawnień stanu (sessions, transcripts, katalog stanu).
- Kontrole uprawnień pliku konfiguracji (`chmod 600`) przy uruchamianiu lokalnie.
- Kondycja uwierzytelniania modeli: sprawdza wygaśnięcie OAuth, może odświeżać tokeny bliskie wygaśnięcia i raportuje stany cooldown/wyłączenia profilów uwierzytelniania.
- Wykrywanie dodatkowego katalogu workspace (`~/openclaw`).
- Naprawa obrazu sandboxa, gdy sandboxing jest włączony.
- Migracja starszych usług i wykrywanie dodatkowych gateway.
- Migracja starszego stanu kanału Matrix (w trybie `--fix` / `--repair`).
- Kontrole runtime gateway (usługa zainstalowana, ale nieuruchomiona; zapisany w pamięci podręcznej label launchd).
- Ostrzeżenia o stanie kanałów (sprawdzane z działającego gateway).
- Audyt konfiguracji supervisora (launchd/systemd/schtasks) z opcjonalną naprawą.
- Kontrole dobrych praktyk runtime gateway (Node vs Bun, ścieżki menedżera wersji).
- Diagnostyka konfliktów portu gateway (domyślnie `18789`).
- Ostrzeżenia bezpieczeństwa dla otwartych polityk DM.
- Kontrole uwierzytelniania gateway dla lokalnego trybu tokena (oferuje wygenerowanie tokena, gdy nie istnieje jego źródło; nie nadpisuje konfiguracji tokenów SecretRef).
- Kontrola `systemd linger` w Linuxie.
- Kontrola rozmiaru plików bootstrap workspace (ostrzeżenia o obcięciu / blisko limitu dla plików kontekstu).
- Kontrola stanu shell completion oraz automatyczna instalacja/aktualizacja.
- Kontrola gotowości dostawcy embeddingów do wyszukiwania pamięci (model lokalny, zdalny klucz API lub binarka QMD).
- Kontrole instalacji ze źródeł (niezgodność workspace pnpm, brakujące zasoby UI, brakująca binarka tsx).
- Zapisuje zaktualizowaną konfigurację + metadane kreatora.

## Backfill i reset Dreams UI

Scena Dreams w Control UI zawiera działania **Backfill**, **Reset** i **Clear Grounded**
dla ugruntowanego przepływu dreaming. Te działania używają metod RPC
w stylu doctor gateway, ale **nie** są częścią naprawy/migracji w CLI `openclaw doctor`.

Co robią:

- **Backfill** skanuje historyczne pliki `memory/YYYY-MM-DD.md` w aktywnym
  workspace, uruchamia ugruntowany przebieg dziennika REM i zapisuje odwracalne wpisy
  backfill do `DREAMS.md`.
- **Reset** usuwa z `DREAMS.md` tylko te oznaczone wpisy dziennika backfill.
- **Clear Grounded** usuwa tylko przygotowane wpisy krótkoterminowe typu grounded-only,
  które pochodzą z historycznego odtworzenia i nie zgromadziły jeszcze aktywnego wsparcia
  z przypomnień ani z dziennych danych.

Czego same z siebie **nie** robią:

- nie edytują `MEMORY.md`
- nie uruchamiają pełnych migracji doctor
- nie przygotowują automatycznie ugruntowanych kandydatów w aktywnym magazynie
  promocji krótkoterminowej, chyba że najpierw jawnie uruchomisz ścieżkę CLI z przygotowaniem

Jeśli chcesz, aby ugruntowane historyczne odtworzenie wpływało na zwykłą ścieżkę
promocji deep, użyj zamiast tego przepływu CLI:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

To przygotowuje ugruntowanych trwałych kandydatów w krótkoterminowym magazynie dreaming, pozostawiając
`DREAMS.md` jako powierzchnię do przeglądu.

## Szczegółowe zachowanie i uzasadnienie

### 0) Opcjonalna aktualizacja (instalacje git)

Jeśli jest to checkout git, a doctor działa interaktywnie, oferuje
aktualizację (fetch/rebase/build) przed uruchomieniem doctor.

### 1) Normalizacja konfiguracji

Jeśli konfiguracja zawiera starsze kształty wartości (na przykład `messages.ackReaction`
bez nadpisania specyficznego dla kanału), doctor normalizuje je do bieżącego
schematu.

Obejmuje to starsze płaskie pola Talk. Bieżąca publiczna konfiguracja Talk to
`talk.provider` + `talk.providers.<provider>`. Doctor przepisuje stare
kształty `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` do mapy dostawcy.

### 2) Migracje starszych kluczy konfiguracji

Gdy konfiguracja zawiera przestarzałe klucze, inne polecenia odmawiają uruchomienia i proszą
o uruchomienie `openclaw doctor`.

Doctor:

- Wyjaśnia, które starsze klucze zostały znalezione.
- Pokazuje zastosowaną migrację.
- Przepisuje `~/.openclaw/openclaw.json` z użyciem zaktualizowanego schematu.

Gateway także automatycznie uruchamia migracje doctor przy starcie, gdy wykryje
starszy format konfiguracji, więc nieaktualne konfiguracje są naprawiane bez ręcznej interwencji.
Migracje magazynu zadań cron są obsługiwane przez `openclaw doctor --fix`.

Bieżące migracje:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → najwyższy poziom `bindings`
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- starsze `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Dla kanałów z nazwanymi `accounts`, ale z pozostawionymi na najwyższym poziomie wartościami kanału dla pojedynczego konta, przenieś te wartości ograniczone do konta do promowanego konta wybranego dla tego kanału (`accounts.default` dla większości kanałów; Matrix może zachować istniejący pasujący nazwany/domyślny cel)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- usuń `browser.relayBindHost` (starsze ustawienie relay rozszerzenia)

Ostrzeżenia doctor obejmują także wskazówki dotyczące domyślnego konta dla kanałów wielokontowych:

- Jeśli skonfigurowano co najmniej dwa wpisy `channels.<channel>.accounts` bez `channels.<channel>.defaultAccount` lub `accounts.default`, doctor ostrzega, że routowanie zapasowe może wybrać nieoczekiwane konto.
- Jeśli `channels.<channel>.defaultAccount` jest ustawione na nieznany identyfikator konta, doctor ostrzega i wypisuje skonfigurowane identyfikatory kont.

### 2b) Nadpisania dostawcy OpenCode

Jeśli ręcznie dodano `models.providers.opencode`, `opencode-zen` lub `opencode-go`,
nadpisuje to wbudowany katalog OpenCode z `@mariozechner/pi-ai`.
Może to wymusić przypisanie modeli do niewłaściwego API albo wyzerować koszty. Doctor ostrzega, aby można było
usunąć nadpisanie i przywrócić routowanie API oraz koszty per model.

### 2c) Migracja przeglądarki i gotowość Chrome MCP

Jeśli konfiguracja przeglądarki nadal wskazuje usuniętą ścieżkę rozszerzenia Chrome, doctor
normalizuje ją do obecnego modelu dołączania Chrome MCP host-local:

- `browser.profiles.*.driver: "extension"` zmienia się na `"existing-session"`
- `browser.relayBindHost` jest usuwane

Doctor audytuje też ścieżkę Chrome MCP host-local, gdy używasz `defaultProfile:
"user"` lub skonfigurowanego profilu `existing-session`:

- sprawdza, czy Google Chrome jest zainstalowane na tym samym hoście dla domyślnych
  profili auto-connect
- sprawdza wykrytą wersję Chrome i ostrzega, gdy jest niższa niż Chrome 144
- przypomina o włączeniu zdalnego debugowania na stronie inspect przeglądarki (na
  przykład `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  lub `edge://inspect/#remote-debugging`)

Doctor nie może włączyć za Ciebie ustawienia po stronie Chrome. Chrome MCP host-local
nadal wymaga:

- przeglądarki opartej na Chromium 144+ na hoście gateway/node
- lokalnie uruchomionej przeglądarki
- włączonego zdalnego debugowania w tej przeglądarce
- zatwierdzenia pierwszego monitu o zgodę na połączenie w przeglądarce

Gotowość tutaj dotyczy wyłącznie lokalnych wymagań wstępnych dołączenia. Existing-session zachowuje
bieżące ograniczenia tras Chrome MCP; zaawansowane trasy, takie jak `responsebody`, eksport PDF,
przechwytywanie pobierania i działania wsadowe, nadal wymagają zarządzanej
przeglądarki lub surowego profilu CDP.

Ta kontrola **nie** dotyczy Docker, sandbox, remote-browser ani innych
przepływów bezgłowych. Nadal używają one surowego CDP.

### 2d) Wymagania wstępne OAuth TLS

Gdy skonfigurowany jest profil OAuth OpenAI Codex, doctor sonduje endpoint
autoryzacji OpenAI, aby sprawdzić, czy lokalny stos TLS Node/OpenSSL potrafi
zweryfikować łańcuch certyfikatów. Jeśli sonda kończy się błędem certyfikatu (na
przykład `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, wygasły certyfikat lub certyfikat samopodpisany),
doctor wypisuje wskazówki naprawy specyficzne dla platformy. W macOS z Node z Homebrew
naprawą jest zwykle `brew postinstall ca-certificates`. Z `--deep` sonda uruchamia się
nawet wtedy, gdy gateway jest w dobrym stanie.

### 2c) Nadpisania dostawcy OAuth Codex

Jeśli wcześniej dodano starsze ustawienia transportu OpenAI w
`models.providers.openai-codex`, mogą one przysłaniać wbudowaną ścieżkę dostawcy
Codex OAuth, której nowsze wydania używają automatycznie. Doctor ostrzega, gdy widzi
te stare ustawienia transportu obok Codex OAuth, aby można było usunąć lub przepisać
nieaktualne nadpisanie transportu i odzyskać wbudowane zachowanie routingu/fallbacków.
Niestandardowe proxy i nadpisania tylko nagłówków są nadal wspierane i nie
wywołują tego ostrzeżenia.

### 3) Migracje starszego stanu (układ na dysku)

Doctor może migrować starsze układy na dysku do bieżącej struktury:

- Magazyn sessions + transcripts:
  - z `~/.openclaw/sessions/` do `~/.openclaw/agents/<agentId>/sessions/`
- Katalog agenta:
  - z `~/.openclaw/agent/` do `~/.openclaw/agents/<agentId>/agent/`
- Stan uwierzytelniania WhatsApp (Baileys):
  - ze starszego `~/.openclaw/credentials/*.json` (z wyjątkiem `oauth.json`)
  - do `~/.openclaw/credentials/whatsapp/<accountId>/...` (domyślny identyfikator konta: `default`)

Te migracje są wykonywane w trybie best-effort i są idempotentne; doctor będzie emitować ostrzeżenia, gdy
pozostawi jakiekolwiek starsze foldery jako kopie zapasowe. Gateway/CLI także automatycznie migruje
starsze sessions + katalog agenta przy starcie, dzięki czemu historia/uwierzytelnianie/modele trafiają do
ścieżki per agent bez ręcznego uruchamiania doctor. Uwierzytelnianie WhatsApp jest celowo migrowane wyłącznie
przez `openclaw doctor`. Normalizacja Talk provider/provider-map porównuje teraz
według równości strukturalnej, więc różnice wyłącznie w kolejności kluczy nie wywołują już
powtarzających się, pustych zmian `doctor --fix`.

### 3a) Migracje starszych manifestów pluginów

Doctor skanuje wszystkie zainstalowane manifesty pluginów pod kątem przestarzałych kluczy
możliwości na najwyższym poziomie (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Gdy je znajdzie, oferuje przeniesienie ich do obiektu `contracts`
i przepisanie pliku manifestu na miejscu. Ta migracja jest idempotentna;
jeśli klucz `contracts` zawiera już te same wartości, starszy klucz jest usuwany
bez duplikowania danych.

### 3b) Migracje starszego magazynu cron

Doctor sprawdza też magazyn zadań cron (`~/.openclaw/cron/jobs.json` domyślnie,
lub `cron.store`, jeśli został nadpisany) pod kątem starych kształtów zadań, które scheduler nadal
akceptuje dla zgodności.

Bieżące porządki cron obejmują:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- pola payload na najwyższym poziomie (`message`, `model`, `thinking`, ...) → `payload`
- pola delivery na najwyższym poziomie (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- aliasy delivery `provider` w payload → jawne `delivery.channel`
- proste starsze zadania zapasowe webhook z `notify: true` → jawne `delivery.mode="webhook"` z `delivery.to=cron.webhook`

Doctor automatycznie migruje zadania `notify: true` tylko wtedy, gdy może to zrobić bez
zmiany zachowania. Jeśli zadanie łączy starszy fallback notify z istniejącym
trybem delivery innym niż webhook, doctor ostrzega i pozostawia to zadanie do ręcznego przeglądu.

### 3c) Czyszczenie blokad sesji

Doctor skanuje każdy katalog sesji agenta w poszukiwaniu przeterminowanych plików blokad zapisu — plików pozostawionych
po nienormalnym zakończeniu sesji. Dla każdego znalezionego pliku blokady raportuje:
ścieżkę, PID, czy PID nadal żyje, wiek blokady oraz czy jest
uznawana za przeterminowaną (martwy PID lub starsza niż 30 minut). W trybie `--fix` / `--repair`
automatycznie usuwa przeterminowane pliki blokad; w przeciwnym razie wypisuje uwagę i
poleca ponowne uruchomienie z `--fix`.

### 4) Kontrole integralności stanu (trwałość sesji, routing i bezpieczeństwo)

Katalog stanu to operacyjny pień mózgu. Jeśli zniknie, tracisz
sessions, credentials, logi i konfigurację (chyba że masz kopie zapasowe gdzie indziej).

Doctor sprawdza:

- **Brak katalogu stanu**: ostrzega o katastrofalnej utracie stanu, prosi o odtworzenie
  katalogu i przypomina, że nie może odzyskać brakujących danych.
- **Uprawnienia katalogu stanu**: weryfikuje możliwość zapisu; oferuje naprawę uprawnień
  (i emituje wskazówkę `chown`, gdy wykryje niezgodność właściciela/grupy).
- **Katalog stanu synchronizowany przez chmurę w macOS**: ostrzega, gdy stan rozwiązuje się do iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) lub
  `~/Library/CloudStorage/...`, ponieważ ścieżki oparte na synchronizacji mogą powodować wolniejsze I/O
  i wyścigi blokad/synchronizacji.
- **Katalog stanu na SD lub eMMC w Linuxie**: ostrzega, gdy stan rozwiązuje się do źródła montowania `mmcblk*`,
  ponieważ losowe I/O na nośnikach SD lub eMMC może być wolniejsze i szybciej zużywać nośnik
  przy zapisach sesji i credentials.
- **Brak katalogów sesji**: `sessions/` i katalog magazynu sesji są
  wymagane do utrwalenia historii i unikania awarii `ENOENT`.
- **Niezgodność transcript**: ostrzega, gdy ostatnie wpisy sesji mają brakujące
  pliki transcript.
- **Główna sesja „1-line JSONL”**: oznacza sytuację, gdy główny transcript ma tylko jedną
  linię (historia się nie kumuluje).
- **Wiele katalogów stanu**: ostrzega, gdy istnieje wiele folderów `~/.openclaw` w różnych
  katalogach domowych lub gdy `OPENCLAW_STATE_DIR` wskazuje inne miejsce (historia może
  zostać rozdzielona między instalacje).
- **Przypomnienie o trybie zdalnym**: jeśli `gateway.mode=remote`, doctor przypomina o jego uruchomieniu
  na zdalnym hoście (tam znajduje się stan).
- **Uprawnienia pliku konfiguracji**: ostrzega, jeśli `~/.openclaw/openclaw.json` jest
  czytelny dla grupy/świata, i oferuje zaostrzenie do `600`.

### 5) Kondycja uwierzytelniania modeli (wygaśnięcie OAuth)

Doctor sprawdza profile OAuth w magazynie uwierzytelniania, ostrzega, gdy tokeny
wygasają lub wygasły, i może je odświeżyć, gdy jest to bezpieczne. Jeśli profil
OAuth/token Anthropic jest nieaktualny, sugeruje klucz API Anthropic lub
ścieżkę setup-token Anthropic.
Monity o odświeżenie pojawiają się tylko w trybie interaktywnym (TTY); `--non-interactive`
pomija próby odświeżania.

Gdy odświeżenie OAuth kończy się trwałym niepowodzeniem (na przykład `refresh_token_reused`,
`invalid_grant` lub dostawca informuje, że trzeba zalogować się ponownie), doctor zgłasza,
że wymagana jest ponowna autoryzacja, i wypisuje dokładną komendę `openclaw models auth login --provider ...`,
którą należy uruchomić.

Doctor raportuje także profile uwierzytelniania, które są tymczasowo nieużywalne z powodu:

- krótkich cooldownów (limity szybkości/timeouty/błędy uwierzytelniania)
- dłuższych wyłączeń (problemy z rozliczeniami/kredytami)

### 6) Walidacja modelu hooks

Jeśli ustawiono `hooks.gmail.model`, doctor waliduje odwołanie do modelu względem
katalogu i listy dozwolonych modeli oraz ostrzega, gdy nie będzie się rozwiązywać lub jest niedozwolone.

### 7) Naprawa obrazu sandboxa

Gdy sandboxing jest włączony, doctor sprawdza obrazy Docker i oferuje ich zbudowanie lub
przełączenie na starsze nazwy, jeśli bieżący obraz jest niedostępny.

### 7b) Zależności runtime bundlowanych pluginów

Doctor weryfikuje, czy zależności runtime bundlowanych pluginów (na przykład
pakiety runtime pluginu Discord) są obecne w katalogu głównym instalacji OpenClaw.
Jeśli którychkolwiek brakuje, doctor raportuje pakiety i instaluje je w
trybie `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migracje usług gateway i wskazówki czyszczenia

Doctor wykrywa starsze usługi gateway (launchd/systemd/schtasks) i
oferuje ich usunięcie oraz instalację usługi OpenClaw z użyciem bieżącego portu gateway.
Może też skanować w poszukiwaniu dodatkowych usług podobnych do gateway i wypisywać wskazówki czyszczenia.
Usługi gateway OpenClaw nazwane profilem są traktowane jako pełnoprawne i nie są
oznaczane jako „dodatkowe”.

### 8b) Migracja Matrix przy uruchamianiu

Gdy konto kanału Matrix ma oczekującą lub możliwą do wykonania migrację starszego stanu,
doctor (w trybie `--fix` / `--repair`) tworzy snapshot przed migracją, a następnie
uruchamia kroki migracji w trybie best-effort: migrację starszego stanu Matrix i przygotowanie starszego stanu szyfrowanego. Oba kroki nie są krytyczne; błędy są logowane i
uruchamianie trwa dalej. W trybie tylko do odczytu (`openclaw doctor` bez `--fix`) ta kontrola
jest całkowicie pomijana.

### 9) Ostrzeżenia bezpieczeństwa

Doctor emituje ostrzeżenia, gdy dostawca jest otwarty na DM bez listy dozwolonych,
lub gdy polityka jest skonfigurowana w niebezpieczny sposób.

### 10) systemd linger (Linux)

Jeśli działa jako usługa użytkownika systemd, doctor upewnia się, że lingering jest włączony, aby
gateway pozostał aktywny po wylogowaniu.

### 11) Stan workspace (Skills, pluginy i starsze katalogi)

Doctor wypisuje podsumowanie stanu workspace dla domyślnego agenta:

- **Stan Skills**: liczba Skills kwalifikujących się, z brakującymi wymaganiami i zablokowanych przez allowlist.
- **Starsze katalogi workspace**: ostrzega, gdy `~/openclaw` lub inne starsze katalogi workspace
  istnieją obok bieżącego workspace.
- **Stan pluginów**: liczby pluginów załadowanych/wyłączonych/z błędami; wypisuje identyfikatory pluginów dla
  błędów; raportuje możliwości bundle plugin.
- **Ostrzeżenia o zgodności pluginów**: oznacza pluginy, które mają problemy zgodności z
  bieżącym runtime.
- **Diagnostyka pluginów**: pokazuje wszelkie ostrzeżenia lub błędy z czasu ładowania emitowane przez
  rejestr pluginów.

### 11b) Rozmiar pliku bootstrap

Doctor sprawdza, czy pliki bootstrap workspace (na przykład `AGENTS.md`,
`CLAUDE.md` lub inne wstrzykiwane pliki kontekstowe) są blisko lub powyżej skonfigurowanego
budżetu znaków. Raportuje dla każdego pliku liczbę znaków surowych vs. wstrzykniętych, procent
obcięcia, przyczynę obcięcia (`max/file` lub `max/total`) oraz łączną liczbę wstrzykniętych
znaków jako ułamek całkowitego budżetu. Gdy pliki są obcięte lub blisko limitu, doctor wypisuje wskazówki
dotyczące dostrajania `agents.defaults.bootstrapMaxChars`
i `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Shell completion

Doctor sprawdza, czy autouzupełnianie jest zainstalowane dla bieżącej powłoki
(zsh, bash, fish lub PowerShell):

- Jeśli profil powłoki używa wolnego wzorca dynamic completion
  (`source <(openclaw completion ...)`), doctor aktualizuje go do szybszego
  wariantu z buforowanym plikiem.
- Jeśli completion jest skonfigurowane w profilu, ale brakuje pliku cache,
  doctor automatycznie regeneruje cache.
- Jeśli completion w ogóle nie jest skonfigurowane, doctor pyta o instalację
  (tylko w trybie interaktywnym; pomijane przy `--non-interactive`).

Uruchom `openclaw completion --write-state`, aby ręcznie zregenerować cache.

### 12) Kontrole uwierzytelniania gateway (lokalny token)

Doctor sprawdza gotowość uwierzytelniania lokalnego tokena gateway.

- Jeśli tryb tokena wymaga tokena, a nie istnieje jego źródło, doctor oferuje jego wygenerowanie.
- Jeśli `gateway.auth.token` jest zarządzany przez SecretRef, ale niedostępny, doctor ostrzega i nie nadpisuje go jawnym tekstem.
- `openclaw doctor --generate-gateway-token` wymusza generowanie tylko wtedy, gdy nie skonfigurowano SecretRef tokena.

### 12b) Naprawy tylko do odczytu uwzględniające SecretRef

Niektóre przepływy naprawcze muszą sprawdzać skonfigurowane credentials bez osłabiania zachowania fail-fast w runtime.

- `openclaw doctor --fix` używa teraz tego samego modelu podsumowania SecretRef tylko do odczytu, co polecenia z rodziny status, do ukierunkowanych napraw konfiguracji.
- Przykład: naprawa `allowFrom` / `groupAllowFrom` Telegram z `@username` próbuje użyć skonfigurowanych credentials bota, gdy są dostępne.
- Jeśli token bota Telegram jest skonfigurowany przez SecretRef, ale niedostępny w bieżącej ścieżce polecenia, doctor zgłasza, że credential jest skonfigurowany, ale niedostępny, i pomija automatyczne rozwiązywanie zamiast kończyć się awarią lub błędnie raportować brak tokena.

### 13) Kontrola kondycji gateway + restart

Doctor uruchamia kontrolę kondycji i oferuje restart gateway, gdy wygląda
na niezdrowy.

### 13b) Gotowość wyszukiwania pamięci

Doctor sprawdza, czy skonfigurowany dostawca embeddingów do wyszukiwania pamięci jest gotowy
dla domyślnego agenta. Zachowanie zależy od skonfigurowanego backendu i dostawcy:

- **Backend QMD**: sonduje, czy binarka `qmd` jest dostępna i da się ją uruchomić.
  Jeśli nie, wypisuje wskazówki naprawy, w tym pakiet npm i opcję ręcznej ścieżki do binarki.
- **Jawny dostawca lokalny**: sprawdza obecność lokalnego pliku modelu lub rozpoznawanego
  zdalnego/pobieralnego URL modelu. Jeśli go brakuje, sugeruje przełączenie na zdalnego dostawcę.
- **Jawny dostawca zdalny** (`openai`, `voyage` itp.): sprawdza, czy klucz API jest
  obecny w środowisku lub magazynie uwierzytelniania. W razie braku wypisuje konkretne wskazówki naprawy.
- **Dostawca auto**: najpierw sprawdza dostępność modelu lokalnego, a następnie próbuje każdego zdalnego
  dostawcy w kolejności automatycznego wyboru.

Gdy wynik sondy gateway jest dostępny (gateway był zdrowy w momencie
kontroli), doctor porównuje go z konfiguracją widoczną dla CLI i odnotowuje
wszelkie rozbieżności.

Użyj `openclaw memory status --deep`, aby zweryfikować gotowość embeddingów w runtime.

### 14) Ostrzeżenia o stanie kanałów

Jeśli gateway jest zdrowy, doctor uruchamia sondę stanu kanałów i zgłasza
ostrzeżenia wraz z proponowanymi poprawkami.

### 15) Audyt konfiguracji supervisora + naprawa

Doctor sprawdza zainstalowaną konfigurację supervisora (launchd/systemd/schtasks) pod kątem
brakujących lub nieaktualnych wartości domyślnych (np. zależności systemd od network-online oraz
opóźnienia restartu). Gdy znajdzie niezgodność, zaleca aktualizację i może
przepisać plik usługi/zadanie do bieżących wartości domyślnych.

Uwagi:

- `openclaw doctor` pyta przed przepisaniem konfiguracji supervisora.
- `openclaw doctor --yes` akceptuje domyślne monity naprawy.
- `openclaw doctor --repair` stosuje zalecane poprawki bez monitów.
- `openclaw doctor --repair --force` nadpisuje niestandardowe konfiguracje supervisora.
- Jeśli uwierzytelnianie tokenem wymaga tokena, a `gateway.auth.token` jest zarządzany przez SecretRef, doctor przy instalacji/naprawie usługi waliduje SecretRef, ale nie zapisuje rozwiązanych wartości tokena w jawnym tekście w metadanych środowiska usługi supervisora.
- Jeśli uwierzytelnianie tokenem wymaga tokena, a skonfigurowany SecretRef tokena nie jest rozwiązany, doctor blokuje ścieżkę instalacji/naprawy i podaje konkretne wskazówki.
- Jeśli skonfigurowano zarówno `gateway.auth.token`, jak i `gateway.auth.password`, a `gateway.auth.mode` nie jest ustawione, doctor blokuje instalację/naprawę do czasu jawnego ustawienia trybu.
- W przypadku jednostek user-systemd w Linuxie kontrole dryfu tokena w doctor obejmują teraz zarówno źródła `Environment=`, jak i `EnvironmentFile=` przy porównywaniu metadanych uwierzytelniania usługi.
- Zawsze możesz wymusić pełne przepisanie przez `openclaw gateway install --force`.

### 16) Diagnostyka runtime gateway i portu

Doctor sprawdza runtime usługi (PID, ostatni status wyjścia) i ostrzega, gdy
usługa jest zainstalowana, ale faktycznie nie działa. Sprawdza też konflikty portu
na porcie gateway (domyślnie `18789`) i raportuje prawdopodobne przyczyny (gateway już
działa, tunel SSH).

### 17) Dobre praktyki runtime gateway

Doctor ostrzega, gdy usługa gateway działa na Bun lub na ścieżce Node zarządzanej przez menedżer wersji
(`nvm`, `fnm`, `volta`, `asdf` itd.). Kanały WhatsApp i Telegram wymagają Node,
a ścieżki menedżera wersji mogą przestać działać po aktualizacjach, ponieważ usługa nie
ładuje inicjalizacji powłoki. Doctor oferuje migrację do systemowej instalacji Node, gdy
jest dostępna (Homebrew/apt/choco).

### 18) Zapis konfiguracji + metadane kreatora

Doctor zapisuje wszelkie zmiany konfiguracji i oznacza metadane kreatora, aby zarejestrować
uruchomienie doctor.

### 19) Wskazówki dotyczące workspace (backup + system pamięci)

Doctor sugeruje system pamięci workspace, jeśli go brakuje, i wypisuje wskazówkę dotyczącą backupu,
jeśli workspace nie jest już pod kontrolą git.

Zobacz [/concepts/agent-workspace](/pl/concepts/agent-workspace), aby przeczytać pełny przewodnik po
strukturze workspace i backupie git (zalecany prywatny GitHub lub GitLab).
