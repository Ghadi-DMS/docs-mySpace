---
read_when:
    - Chcesz skonfigurować dostawców wyszukiwania pamięci lub modele embeddingów
    - Chcesz skonfigurować backend QMD
    - Chcesz dostroić wyszukiwanie hybrydowe, MMR lub zanik czasowy
    - Chcesz włączyć multimodalne indeksowanie pamięci
summary: Wszystkie opcje konfiguracji dotyczące wyszukiwania pamięci, dostawców embeddingów, QMD, wyszukiwania hybrydowego i indeksowania multimodalnego
title: Dokumentacja konfiguracji pamięci
x-i18n:
    generated_at: "2026-04-12T23:33:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299ca9b69eea292ea557a2841232c637f5c1daf2bc0f73c0a42f7c0d8d566ce2
    source_path: reference/memory-config.md
    workflow: 15
---

# Dokumentacja konfiguracji pamięci

Ta strona zawiera wszystkie opcje konfiguracji wyszukiwania pamięci OpenClaw. Aby zapoznać się z omówieniami koncepcyjnymi, zobacz:

- [Przegląd pamięci](/pl/concepts/memory) -- jak działa pamięć
- [Wbudowany silnik](/pl/concepts/memory-builtin) -- domyślny backend SQLite
- [Silnik QMD](/pl/concepts/memory-qmd) -- lokalny sidecar działający lokalnie w pierwszej kolejności
- [Wyszukiwanie pamięci](/pl/concepts/memory-search) -- pipeline wyszukiwania i dostrajanie
- [Active Memory](/pl/concepts/active-memory) -- włączanie sub-agenta pamięci dla interaktywnych sesji

Wszystkie ustawienia wyszukiwania pamięci znajdują się w `agents.defaults.memorySearch` w
`openclaw.json`, o ile nie zaznaczono inaczej.

Jeśli szukasz przełącznika funkcji **Active Memory** i konfiguracji sub-agenta,
znajdują się one w `plugins.entries.active-memory`, a nie w `memorySearch`.

Active Memory używa modelu dwóch bramek:

1. Plugin musi być włączony i wskazywać bieżący identyfikator agenta
2. Żądanie musi być kwalifikującą się interaktywną trwałą sesją czatu

Zobacz [Active Memory](/pl/concepts/active-memory), aby poznać model aktywacji,
konfigurację należącą do Pluginu, trwałość transkryptów i bezpieczny wzorzec wdrażania.

---

## Wybór dostawcy

| Klucz     | Typ       | Domyślnie       | Opis                                                                                       |
| --------- | --------- | --------------- | ------------------------------------------------------------------------------------------ |
| `provider` | `string` | wykrywane automatycznie | Id adaptera embeddingów: `openai`, `gemini`, `voyage`, `mistral`, `bedrock`, `ollama`, `local` |
| `model`   | `string`  | domyślny dla dostawcy | Nazwa modelu embeddingów                                                              |
| `fallback` | `string` | `"none"`        | Id adaptera zapasowego, gdy główny zawiedzie                                              |
| `enabled` | `boolean` | `true`          | Włącza lub wyłącza wyszukiwanie pamięci                                                    |

### Kolejność automatycznego wykrywania

Gdy `provider` nie jest ustawiony, OpenClaw wybiera pierwszy dostępny:

1. `local` -- jeśli skonfigurowano `memorySearch.local.modelPath` i plik istnieje.
2. `openai` -- jeśli można rozwiązać klucz OpenAI.
3. `gemini` -- jeśli można rozwiązać klucz Gemini.
4. `voyage` -- jeśli można rozwiązać klucz Voyage.
5. `mistral` -- jeśli można rozwiązać klucz Mistral.
6. `bedrock` -- jeśli łańcuch poświadczeń AWS SDK zostanie rozwiązany (rola instancji, klucze dostępu, profil, SSO, tożsamość web lub współdzielona konfiguracja).

`ollama` jest obsługiwane, ale nie jest wykrywane automatycznie (ustaw je jawnie).

### Rozwiązywanie klucza API

Zdalne embeddingi wymagają klucza API. Bedrock używa zamiast tego domyślnego
łańcucha poświadczeń AWS SDK (role instancji, SSO, klucze dostępu).

| Dostawca | Zmienna env                  | Klucz konfiguracji               |
| -------- | ---------------------------- | -------------------------------- |
| OpenAI   | `OPENAI_API_KEY`             | `models.providers.openai.apiKey` |
| Gemini   | `GEMINI_API_KEY`             | `models.providers.google.apiKey` |
| Voyage   | `VOYAGE_API_KEY`             | `models.providers.voyage.apiKey` |
| Mistral  | `MISTRAL_API_KEY`            | `models.providers.mistral.apiKey` |
| Bedrock  | łańcuch poświadczeń AWS      | Klucz API nie jest potrzebny     |
| Ollama   | `OLLAMA_API_KEY` (placeholder) | --                             |

Codex OAuth obejmuje tylko chat/completions i nie spełnia wymagań żądań embeddingów.

---

## Konfiguracja zdalnego punktu końcowego

Dla niestandardowych punktów końcowych zgodnych z OpenAI lub nadpisywania wartości domyślnych dostawcy:

| Klucz            | Typ      | Opis                                                 |
| ---------------- | -------- | ---------------------------------------------------- |
| `remote.baseUrl` | `string` | Niestandardowy bazowy URL API                        |
| `remote.apiKey`  | `string` | Nadpisanie klucza API                                |
| `remote.headers` | `object` | Dodatkowe nagłówki HTTP (łączone z domyślnymi dostawcy) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Konfiguracja specyficzna dla Gemini

| Klucz                  | Typ      | Domyślnie              | Opis                                       |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | Obsługuje także `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | Dla Embedding 2: 768, 1536 lub 3072        |

<Warning>
Zmiana modelu lub `outputDimensionality` uruchamia automatyczne pełne ponowne indeksowanie.
</Warning>

---

## Konfiguracja embeddingów Bedrock

Bedrock używa domyślnego łańcucha poświadczeń AWS SDK -- nie są potrzebne klucze API.
Jeśli OpenClaw działa na EC2 z rolą instancji z włączonym Bedrock, po prostu ustaw
dostawcę i model:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| Klucz                  | Typ      | Domyślnie                     | Opis                              |
| ---------------------- | -------- | ----------------------------- | --------------------------------- |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | Dowolny identyfikator modelu embeddingów Bedrock |
| `outputDimensionality` | `number` | domyślna dla modelu           | Dla Titan V2: 256, 512 lub 1024   |

### Obsługiwane modele

Obsługiwane są następujące modele (z wykrywaniem rodziny i domyślnymi
wymiarami):

| Id modelu                                  | Dostawca   | Domyślne wymiary | Konfigurowalne wymiary |
| ------------------------------------------ | ---------- | ---------------- | ---------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024             | 256, 512, 1024         |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536             | --                     |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536             | --                     |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024             | --                     |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024             | 256, 384, 1024, 3072   |
| `cohere.embed-english-v3`                  | Cohere     | 1024             | --                     |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024             | --                     |
| `cohere.embed-v4:0`                        | Cohere     | 1536             | 256-1536               |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512              | --                     |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024             | --                     |

Warianty z sufiksem przepustowości (np. `amazon.titan-embed-text-v1:2:8k`) dziedziczą
konfigurację modelu bazowego.

### Uwierzytelnianie

Uwierzytelnianie Bedrock używa standardowej kolejności rozwiązywania poświadczeń AWS SDK:

1. Zmienne środowiskowe (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. Pamięć podręczna tokenów SSO
3. Poświadczenia tokenu tożsamości web
4. Współdzielone pliki poświadczeń i konfiguracji
5. Poświadczenia metadanych ECS lub EC2

Region jest rozwiązywany z `AWS_REGION`, `AWS_DEFAULT_REGION`, `baseUrl`
dostawcy `amazon-bedrock` lub domyślnie przyjmuje wartość `us-east-1`.

### Uprawnienia IAM

Rola lub użytkownik IAM potrzebuje:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

Aby zachować zasadę najmniejszych uprawnień, ogranicz `InvokeModel` do konkretnego modelu:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## Konfiguracja lokalnych embeddingów

| Klucz                 | Typ      | Domyślnie               | Opis                                |
| --------------------- | -------- | ----------------------- | ----------------------------------- |
| `local.modelPath`     | `string` | pobierany automatycznie | Ścieżka do pliku modelu GGUF        |
| `local.modelCacheDir` | `string` | domyślna dla node-llama-cpp | Katalog cache dla pobranych modeli |

Domyślny model: `embeddinggemma-300m-qat-Q8_0.gguf` (~0,6 GB, pobierany automatycznie).
Wymaga natywnego builda: `pnpm approve-builds`, a następnie `pnpm rebuild node-llama-cpp`.

---

## Konfiguracja wyszukiwania hybrydowego

Wszystko w `memorySearch.query.hybrid`:

| Klucz                 | Typ       | Domyślnie | Opis                                 |
| --------------------- | --------- | --------- | ------------------------------------ |
| `enabled`             | `boolean` | `true`    | Włącza hybrydowe wyszukiwanie BM25 + wektorowe |
| `vectorWeight`        | `number`  | `0.7`     | Waga dla wyników wektorowych (0-1)   |
| `textWeight`          | `number`  | `0.3`     | Waga dla wyników BM25 (0-1)          |
| `candidateMultiplier` | `number`  | `4`       | Mnożnik rozmiaru puli kandydatów     |

### MMR (różnorodność)

| Klucz         | Typ       | Domyślnie | Opis                                      |
| ------------- | --------- | --------- | ----------------------------------------- |
| `mmr.enabled` | `boolean` | `false`   | Włącza ponowne rangowanie MMR             |
| `mmr.lambda`  | `number`  | `0.7`     | 0 = maks. różnorodność, 1 = maks. trafność |

### Zanik czasowy (świeżość)

| Klucz                        | Typ       | Domyślnie | Opis                            |
| ---------------------------- | --------- | --------- | ------------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false`   | Włącza wzmocnienie świeżości    |
| `temporalDecay.halfLifeDays` | `number`  | `30`      | Wynik spada o połowę co N dni   |

Pliki evergreen (`MEMORY.md`, pliki bez daty w `memory/`) nigdy nie podlegają zanikowi.

### Pełny przykład

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## Dodatkowe ścieżki pamięci

| Klucz       | Typ        | Opis                                      |
| ----------- | ---------- | ----------------------------------------- |
| `extraPaths` | `string[]` | Dodatkowe katalogi lub pliki do indeksowania |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

Ścieżki mogą być bezwzględne lub względne względem obszaru roboczego. Katalogi są skanowane
rekurencyjnie w poszukiwaniu plików `.md`. Obsługa symlinków zależy od aktywnego backendu:
wbudowany silnik ignoruje symlinki, natomiast QMD stosuje zachowanie skanera QMD.

W przypadku wyszukiwania transkryptów między agentami o zakresie agenta użyj
`agents.list[].memorySearch.qmd.extraCollections` zamiast `memory.qmd.paths`.
Te dodatkowe kolekcje używają tego samego kształtu `{ path, name, pattern? }`, ale
są scalane per agent i mogą zachować jawne współdzielone nazwy, gdy ścieżka
wskazuje poza bieżący obszar roboczy.
Jeśli ta sama rozwiązana ścieżka pojawia się zarówno w `memory.qmd.paths`, jak i
`memorySearch.qmd.extraCollections`, QMD zachowuje pierwszy wpis i pomija duplikat.

---

## Pamięć multimodalna (Gemini)

Indeksuj obrazy i audio obok Markdown przy użyciu Gemini Embedding 2:

| Klucz                     | Typ        | Domyślnie  | Opis                                     |
| ------------------------- | ---------- | ---------- | ---------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | Włącza indeksowanie multimodalne         |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]` lub `["all"]`   |
| `multimodal.maxFileBytes` | `number`   | `10000000` | Maksymalny rozmiar pliku do indeksowania |

Dotyczy tylko plików w `extraPaths`. Domyślne katalogi główne pamięci pozostają tylko dla Markdown.
Wymaga `gemini-embedding-2-preview`. `fallback` musi mieć wartość `"none"`.

Obsługiwane formaty: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(obrazy); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (audio).

---

## Cache embeddingów

| Klucz              | Typ       | Domyślnie | Opis                                |
| ------------------ | --------- | --------- | ----------------------------------- |
| `cache.enabled`    | `boolean` | `false`   | Buforuje embeddingi chunków w SQLite |
| `cache.maxEntries` | `number`  | `50000`   | Maksymalna liczba buforowanych embeddingów |

Zapobiega ponownemu tworzeniu embeddingów dla niezmienionego tekstu podczas ponownego indeksowania lub aktualizacji transkryptów.

---

## Indeksowanie wsadowe

| Klucz                         | Typ       | Domyślnie | Opis                         |
| ----------------------------- | --------- | --------- | ---------------------------- |
| `remote.batch.enabled`        | `boolean` | `false`   | Włącza API embeddingów wsadowych |
| `remote.batch.concurrency`    | `number`  | `2`       | Równoległe zadania wsadowe   |
| `remote.batch.wait`           | `boolean` | `true`    | Czeka na zakończenie wsadu   |
| `remote.batch.pollIntervalMs` | `number`  | --        | Interwał odpytywania         |
| `remote.batch.timeoutMinutes` | `number`  | --        | Limit czasu wsadu            |

Dostępne dla `openai`, `gemini` i `voyage`. Wsad OpenAI jest zazwyczaj
najszybszy i najtańszy przy dużych uzupełnieniach danych.

---

## Wyszukiwanie pamięci sesji (eksperymentalne)

Indeksuj transkrypcje sesji i udostępniaj je przez `memory_search`:

| Klucz                       | Typ        | Domyślnie    | Opis                                       |
| --------------------------- | ---------- | ------------ | ------------------------------------------ |
| `experimental.sessionMemory` | `boolean` | `false`      | Włącza indeksowanie sesji                  |
| `sources`                   | `string[]` | `["memory"]` | Dodaj `"sessions"`, aby uwzględnić transkrypty |
| `sync.sessions.deltaBytes`  | `number`   | `100000`     | Próg bajtów dla ponownego indeksowania     |
| `sync.sessions.deltaMessages` | `number` | `50`         | Próg liczby wiadomości dla ponownego indeksowania |

Indeksowanie sesji jest opt-in i działa asynchronicznie. Wyniki mogą być nieco
nieaktualne. Logi sesji są przechowywane na dysku, więc traktuj dostęp do systemu plików jako granicę zaufania.

---

## Przyspieszenie wektorowe SQLite (`sqlite-vec`)

| Klucz                    | Typ       | Domyślnie | Opis                                  |
| ------------------------ | --------- | --------- | ------------------------------------- |
| `store.vector.enabled`   | `boolean` | `true`    | Używa `sqlite-vec` do zapytań wektorowych |
| `store.vector.extensionPath` | `string` | bundlowane | Nadpisuje ścieżkę `sqlite-vec`      |

Gdy `sqlite-vec` nie jest dostępne, OpenClaw automatycznie przechodzi na obliczanie
podobieństwa cosinusowego w procesie.

---

## Przechowywanie indeksu

| Klucz                | Typ      | Domyślnie                             | Opis                                         |
| -------------------- | -------- | ------------------------------------- | -------------------------------------------- |
| `store.path`         | `string` | `~/.openclaw/memory/{agentId}.sqlite` | Lokalizacja indeksu (obsługuje token `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                          | Tokenizer FTS5 (`unicode61` lub `trigram`)   |

---

## Konfiguracja backendu QMD

Ustaw `memory.backend = "qmd"`, aby włączyć. Wszystkie ustawienia QMD znajdują się w
`memory.qmd`:

| Klucz                   | Typ       | Domyślnie | Opis                                          |
| ----------------------- | --------- | --------- | --------------------------------------------- |
| `command`               | `string`  | `qmd`     | Ścieżka wykonywalna QMD                       |
| `searchMode`            | `string`  | `search`  | Polecenie wyszukiwania: `search`, `vsearch`, `query` |
| `includeDefaultMemory`  | `boolean` | `true`    | Automatycznie indeksuje `MEMORY.md` + `memory/**/*.md` |
| `paths[]`               | `array`   | --        | Dodatkowe ścieżki: `{ name, path, pattern? }` |
| `sessions.enabled`      | `boolean` | `false`   | Indeksuje transkrypty sesji                   |
| `sessions.retentionDays` | `number` | --        | Retencja transkryptów                         |
| `sessions.exportDir`    | `string`  | --        | Katalog eksportu                              |

OpenClaw preferuje bieżące kształty kolekcji QMD i zapytań MCP, ale utrzymuje
zgodność ze starszymi wydaniami QMD, przechodząc awaryjnie na starsze flagi kolekcji `--mask`
i starsze nazwy narzędzi MCP, gdy jest to potrzebne.

Nadpisania modeli QMD pozostają po stronie QMD, a nie w konfiguracji OpenClaw. Jeśli chcesz
globalnie nadpisać modele QMD, ustaw zmienne środowiskowe takie jak
`QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` i `QMD_GENERATE_MODEL` w środowisku uruchomieniowym Gateway.

### Harmonogram aktualizacji

| Klucz                     | Typ       | Domyślnie | Opis                                      |
| ------------------------- | --------- | --------- | ----------------------------------------- |
| `update.interval`         | `string`  | `5m`      | Interwał odświeżania                      |
| `update.debounceMs`       | `number`  | `15000`   | Debounce zmian plików                     |
| `update.onBoot`           | `boolean` | `true`    | Odświeżanie przy starcie                  |
| `update.waitForBootSync`  | `boolean` | `false`   | Blokuje start do zakończenia odświeżenia  |
| `update.embedInterval`    | `string`  | --        | Osobny harmonogram embeddingów            |
| `update.commandTimeoutMs` | `number`  | --        | Limit czasu dla poleceń QMD               |
| `update.updateTimeoutMs`  | `number`  | --        | Limit czasu dla operacji aktualizacji QMD |
| `update.embedTimeoutMs`   | `number`  | --        | Limit czasu dla operacji embeddingów QMD  |

### Limity

| Klucz                     | Typ      | Domyślnie | Opis                            |
| ------------------------- | -------- | --------- | ------------------------------- |
| `limits.maxResults`       | `number` | `6`       | Maksymalna liczba wyników wyszukiwania |
| `limits.maxSnippetChars`  | `number` | --        | Ogranicza długość fragmentu     |
| `limits.maxInjectedChars` | `number` | --        | Ogranicza łączną liczbę wstrzykniętych znaków |
| `limits.timeoutMs`        | `number` | `4000`    | Limit czasu wyszukiwania        |

### Zakres

Kontroluje, które sesje mogą otrzymywać wyniki wyszukiwania QMD. Ten sam schemat co
[`session.sendPolicy`](/pl/gateway/configuration-reference#session):

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Dostarczana domyślna konfiguracja dopuszcza sesje bezpośrednie i kanałowe, nadal odrzucając
grupy.

Domyślnie tylko DM. `match.keyPrefix` dopasowuje znormalizowany klucz sesji;
`match.rawKeyPrefix` dopasowuje surowy klucz, włącznie z `agent:<id>:`.

### Cytowania

`memory.citations` dotyczy wszystkich backendów:

| Wartość          | Zachowanie                                         |
| ---------------- | -------------------------------------------------- |
| `auto` (domyślnie) | Dodaje stopkę `Source: <path#line>` w fragmentach |
| `on`             | Zawsze dodaje stopkę                               |
| `off`            | Pomija stopkę (ścieżka nadal jest wewnętrznie przekazywana agentowi) |

### Pełny przykład QMD

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming (eksperymentalne)

Dreaming jest konfigurowane w `plugins.entries.memory-core.config.dreaming`,
a nie w `agents.defaults.memorySearch`.

Dreaming działa jako jeden zaplanowany przebieg i wykorzystuje wewnętrzne fazy light/deep/REM jako
szczegół implementacyjny.

Aby poznać zachowanie koncepcyjne i polecenia slash, zobacz [Dreaming](/pl/concepts/dreaming).

### Ustawienia użytkownika

| Klucz      | Typ       | Domyślnie    | Opis                                               |
| ---------- | --------- | ------------ | -------------------------------------------------- |
| `enabled`  | `boolean` | `false`      | Całkowicie włącza lub wyłącza Dreaming             |
| `frequency` | `string` | `0 3 * * *`  | Opcjonalna składnia Cron dla pełnego przebiegu Dreaming |

### Przykład

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

Uwagi:

- Dreaming zapisuje stan maszyny do `memory/.dreams/`.
- Dreaming zapisuje czytelne dla człowieka wyjście narracyjne do `DREAMS.md` (lub istniejącego `dreams.md`).
- Zasady i progi faz light/deep/REM są zachowaniem wewnętrznym, a nie konfiguracją skierowaną do użytkownika.
