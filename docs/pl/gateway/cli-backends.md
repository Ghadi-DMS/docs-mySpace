---
read_when:
    - Chcesz mieć niezawodne rozwiązanie awaryjne, gdy dostawcy API zawodzą
    - Uruchamiasz Codex CLI lub inne lokalne AI CLI i chcesz używać ich ponownie
    - Chcesz zrozumieć most local loopback MCP zapewniający dostęp backendu CLI do narzędzi
summary: 'Backendy CLI: lokalne awaryjne przełączenie na AI CLI z opcjonalnym mostem narzędzi MCP'
title: Backendy CLI
x-i18n:
    generated_at: "2026-04-09T01:28:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9b458f9fe6fa64c47864c8c180f3dedfd35c5647de470a2a4d31c26165663c20
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backendy CLI (awaryjne środowisko uruchomieniowe)

OpenClaw może uruchamiać **lokalne AI CLI** jako **awaryjne rozwiązanie tylko tekstowe**, gdy dostawcy API są niedostępni,
ograniczani limitami lub tymczasowo działają nieprawidłowo. To rozwiązanie jest celowo zachowawcze:

- **Narzędzia OpenClaw nie są wstrzykiwane bezpośrednio**, ale backendy z `bundleMcp: true`
  mogą otrzymywać narzędzia bramy przez most MCP typu loopback.
- **Strumieniowanie JSONL** dla CLI, które je obsługują.
- **Sesje są obsługiwane** (dzięki temu kolejne tury pozostają spójne).
- **Obrazy mogą być przekazywane dalej**, jeśli CLI akceptuje ścieżki do obrazów.

To rozwiązanie pełni rolę **siatki bezpieczeństwa**, a nie głównej ścieżki. Używaj go, gdy
chcesz uzyskiwać tekstowe odpowiedzi typu „zawsze działa” bez polegania na zewnętrznych API.

Jeśli chcesz pełnego środowiska z kontrolą sesji ACP, zadaniami w tle,
powiązaniem wątków/konwersacji i trwałymi zewnętrznymi sesjami kodowania, użyj
[ACP Agents](/pl/tools/acp-agents). Backendy CLI nie są ACP.

## Szybki start dla początkujących

Możesz używać Codex CLI **bez żadnej konfiguracji** (dołączona wtyczka OpenAI
rejestruje domyślny backend):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Jeśli Twoja brama działa pod launchd/systemd, a `PATH` jest minimalne, dodaj tylko
ścieżkę do polecenia:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

To wszystko. Nie są potrzebne żadne klucze ani dodatkowa konfiguracja uwierzytelniania poza samym CLI.

Jeśli używasz dołączonego backendu CLI jako **głównego dostawcy wiadomości** na
hoście bramy, OpenClaw teraz automatycznie ładuje odpowiednią dołączoną wtyczkę, gdy Twoja konfiguracja
jawnie odwołuje się do tego backendu w odwołaniu do modelu albo w
`agents.defaults.cliBackends`.

## Używanie jako rozwiązania awaryjnego

Dodaj backend CLI do listy awaryjnej, aby był uruchamiany tylko wtedy, gdy główne modele zawiodą:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

Uwagi:

- Jeśli używasz `agents.defaults.models` (lista dozwolonych), musisz uwzględnić tam także modele backendu CLI.
- Jeśli główny dostawca zawiedzie (uwierzytelnianie, limity, limity czasu), OpenClaw
  spróbuje następnie użyć backendu CLI.

## Przegląd konfiguracji

Wszystkie backendy CLI znajdują się pod:

```
agents.defaults.cliBackends
```

Każdy wpis jest kluczowany przez **identyfikator dostawcy** (na przykład `codex-cli`, `my-cli`).
Identyfikator dostawcy staje się lewą stroną odwołania do modelu:

```
<provider>/<model>
```

### Przykładowa konfiguracja

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          // CLI w stylu Codex mogą zamiast tego wskazywać plik promptu:
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## Jak to działa

1. **Wybiera backend** na podstawie prefiksu dostawcy (`codex-cli/...`).
2. **Buduje prompt systemowy** z użyciem tego samego promptu OpenClaw i kontekstu obszaru roboczego.
3. **Uruchamia CLI** z identyfikatorem sesji (jeśli jest obsługiwany), aby zachować spójność historii.
4. **Parsuje dane wyjściowe** (JSON lub zwykły tekst) i zwraca końcowy tekst.
5. **Utrwala identyfikatory sesji** dla każdego backendu, dzięki czemu kolejne tury używają tej samej sesji CLI.

<Note>
Dołączony backend Anthropic `claude-cli` jest ponownie obsługiwany. Pracownicy Anthropic
poinformowali nas, że użycie Claude CLI w stylu OpenClaw jest znów dozwolone, więc OpenClaw traktuje
użycie `claude -p` jako zatwierdzone dla tej integracji, chyba że Anthropic opublikuje
nową politykę.
</Note>

Dołączony backend OpenAI `codex-cli` przekazuje prompt systemowy OpenClaw przez
nadpisanie konfiguracji `model_instructions_file` w Codex (`-c
model_instructions_file="..."`). Codex nie udostępnia flagi w stylu Claude
`--append-system-prompt`, więc OpenClaw zapisuje złożony prompt do
pliku tymczasowego dla każdej nowej sesji Codex CLI.

## Sesje

- Jeśli CLI obsługuje sesje, ustaw `sessionArg` (na przykład `--session-id`) albo
  `sessionArgs` (symbol zastępczy `{sessionId}`), gdy identyfikator trzeba wstawić
  do wielu flag.
- Jeśli CLI używa **podpolecenia wznawiania** z innymi flagami, ustaw
  `resumeArgs` (zastępuje `args` przy wznawianiu) i opcjonalnie `resumeOutput`
  (dla wznowień innych niż JSON).
- `sessionMode`:
  - `always`: zawsze wysyłaj identyfikator sesji (nowy UUID, jeśli nic nie zapisano).
  - `existing`: wysyłaj identyfikator sesji tylko wtedy, gdy został wcześniej zapisany.
  - `none`: nigdy nie wysyłaj identyfikatora sesji.

Uwagi dotyczące serializacji:

- `serialize: true` utrzymuje uporządkowanie uruchomień w tym samym torze.
- Większość CLI serializuje na jednym torze dostawcy.
- OpenClaw porzuca ponowne użycie zapisanej sesji CLI, gdy stan uwierzytelnienia backendu się zmienia, w tym po ponownym logowaniu, rotacji tokena lub zmianie poświadczenia profilu uwierzytelniania.

## Obrazy (przekazywanie dalej)

Jeśli Twoje CLI akceptuje ścieżki do obrazów, ustaw `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw zapisze obrazy base64 do plików tymczasowych. Jeśli ustawiono `imageArg`, te
ścieżki są przekazywane jako argumenty CLI. Jeśli `imageArg` nie jest ustawione, OpenClaw dołącza
ścieżki plików do promptu (wstrzykiwanie ścieżek), co wystarcza dla CLI, które automatycznie
ładują pliki lokalne ze zwykłych ścieżek.

## Wejścia / wyjścia

- `output: "json"` (domyślnie) próbuje sparsować JSON i wyodrębnić tekst oraz identyfikator sesji.
- Dla wyjścia JSON Gemini CLI OpenClaw odczytuje tekst odpowiedzi z `response` oraz
  użycie ze `stats`, gdy `usage` nie istnieje lub jest puste.
- `output: "jsonl"` parsuje strumienie JSONL (na przykład Codex CLI `--json`) i wyodrębnia końcową wiadomość agenta oraz identyfikatory sesji,
  jeśli są obecne.
- `output: "text"` traktuje stdout jako końcową odpowiedź.

Tryby wejścia:

- `input: "arg"` (domyślnie) przekazuje prompt jako ostatni argument CLI.
- `input: "stdin"` wysyła prompt przez stdin.
- Jeśli prompt jest bardzo długi i ustawiono `maxPromptArgChars`, używane jest stdin.

## Domyślne ustawienia (należące do wtyczki)

Dołączona wtyczka OpenAI rejestruje również ustawienia domyślne dla `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Dołączona wtyczka Google rejestruje również ustawienia domyślne dla `google-gemini-cli`:

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Wymaganie wstępne: lokalny Gemini CLI musi być zainstalowany i dostępny jako
`gemini` w `PATH` (`brew install gemini-cli` lub
`npm install -g @google/gemini-cli`).

Uwagi dotyczące JSON Gemini CLI:

- Tekst odpowiedzi jest odczytywany z pola JSON `response`.
- Użycie przechodzi awaryjnie na `stats`, gdy `usage` jest nieobecne lub puste.
- `stats.cached` jest normalizowane do OpenClaw `cacheRead`.
- Jeśli `stats.input` nie istnieje, OpenClaw wyprowadza tokeny wejściowe z
  `stats.input_tokens - stats.cached`.

Nadpisuj tylko wtedy, gdy to potrzebne (najczęściej: bezwzględna ścieżka `command`).

## Ustawienia domyślne należące do wtyczki

Domyślne ustawienia backendów CLI są teraz częścią powierzchni wtyczki:

- Wtyczki rejestrują je przez `api.registerCliBackend(...)`.
- `id` backendu staje się prefiksem dostawcy w odwołaniach do modeli.
- Konfiguracja użytkownika w `agents.defaults.cliBackends.<id>` nadal nadpisuje ustawienie domyślne wtyczki.
- Czyszczenie konfiguracji specyficznej dla backendu pozostaje po stronie wtyczki dzięki opcjonalnemu
  hookowi `normalizeConfig`.

## Nakładki Bundle MCP

Backendy CLI **nie** otrzymują bezpośrednio wywołań narzędzi OpenClaw, ale backend może
włączyć wygenerowaną nakładkę konfiguracji MCP przez `bundleMcp: true`.

Bieżące dołączone zachowanie:

- `claude-cli`: wygenerowany ścisły plik konfiguracyjny MCP
- `codex-cli`: wbudowane nadpisania konfiguracji dla `mcp_servers`
- `google-gemini-cli`: wygenerowany plik ustawień systemowych Gemini

Gdy bundle MCP jest włączone, OpenClaw:

- uruchamia serwer HTTP MCP typu loopback, który udostępnia narzędzia bramy procesowi CLI
- uwierzytelnia most za pomocą tokena per sesja (`OPENCLAW_MCP_TOKEN`)
- ogranicza dostęp do narzędzi do bieżącej sesji, konta i kontekstu kanału
- ładuje włączone serwery bundle-MCP dla bieżącego obszaru roboczego
- scala je z dowolnym istniejącym kształtem konfiguracji/ustawień MCP backendu
- przepisuje konfigurację uruchamiania przy użyciu trybu integracji należącego do backendu z rozszerzenia będącego jego właścicielem

Jeśli żadne serwery MCP nie są włączone, OpenClaw nadal wstrzykuje ścisłą konfigurację, gdy
backend korzysta z bundle MCP, aby uruchomienia w tle pozostawały odizolowane.

## Ograniczenia

- **Brak bezpośrednich wywołań narzędzi OpenClaw.** OpenClaw nie wstrzykuje wywołań narzędzi do
  protokołu backendu CLI. Backendy widzą narzędzia bramy tylko wtedy, gdy wybiorą
  `bundleMcp: true`.
- **Strumieniowanie jest zależne od backendu.** Niektóre backendy strumieniują JSONL; inne buforują
  do momentu zakończenia.
- **Ustrukturyzowane dane wyjściowe** zależą od formatu JSON danego CLI.
- **Sesje Codex CLI** są wznawiane przez wyjście tekstowe (bez JSONL), co jest
  mniej ustrukturyzowane niż początkowe uruchomienie `--json`. Sesje OpenClaw nadal działają
  normalnie.

## Rozwiązywanie problemów

- **Nie znaleziono CLI**: ustaw `command` na pełną ścieżkę.
- **Nieprawidłowa nazwa modelu**: użyj `modelAliases`, aby zmapować `provider/model` → model CLI.
- **Brak ciągłości sesji**: upewnij się, że ustawiono `sessionArg`, a `sessionMode` nie ma wartości
  `none` (Codex CLI obecnie nie może wznawiać z wyjściem JSON).
- **Obrazy są ignorowane**: ustaw `imageArg` (i sprawdź, czy CLI obsługuje ścieżki do plików).
