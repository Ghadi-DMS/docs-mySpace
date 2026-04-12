---
read_when:
    - Tworzysz Plugin OpenClaw
    - Musisz dostarczyć schemat konfiguracji Pluginu albo debugować błędy walidacji Pluginu
summary: Manifest Pluginu + wymagania schematu JSON (ścisła walidacja konfiguracji)
title: Manifest Pluginu
x-i18n:
    generated_at: "2026-04-12T23:28:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93b57c7373e4ccd521b10945346db67991543bd2bed4cc8b6641e1f215b48579
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifest Pluginu (`openclaw.plugin.json`)

Ta strona dotyczy wyłącznie **natywnego manifestu Pluginu OpenClaw**.

Informacje o zgodnych układach bundli znajdziesz tutaj: [Bundle Pluginów](/pl/plugins/bundles).

Zgodne formaty bundli używają innych plików manifestu:

- Bundle Codex: `.codex-plugin/plugin.json`
- Bundle Claude: `.claude-plugin/plugin.json` albo domyślny układ komponentów Claude
  bez manifestu
- Bundle Cursor: `.cursor-plugin/plugin.json`

OpenClaw automatycznie wykrywa również te układy bundli, ale nie są one walidowane
względem opisanego tutaj schematu `openclaw.plugin.json`.

W przypadku zgodnych bundli OpenClaw obecnie odczytuje metadane bundla oraz zadeklarowane
korzenie Skills, korzenie poleceń Claude, domyślne wartości `settings.json` bundla Claude,
domyślne wartości LSP bundla Claude oraz obsługiwane pakiety hooków, gdy układ odpowiada
oczekiwaniom środowiska uruchomieniowego OpenClaw.

Każdy natywny Plugin OpenClaw **musi** dostarczać plik `openclaw.plugin.json` w
**katalogu głównym Pluginu**. OpenClaw używa tego manifestu do walidacji konfiguracji
**bez wykonywania kodu Pluginu**. Brakujące lub nieprawidłowe manifesty są traktowane jako
błędy Pluginu i blokują walidację konfiguracji.

Zobacz pełny przewodnik po systemie Pluginów: [Pluginy](/pl/tools/plugin).
Informacje o natywnym modelu capabilities i aktualnych wskazówkach dotyczących zgodności zewnętrznej:
[Model capabilities](/pl/plugins/architecture#public-capability-model).

## Do czego służy ten plik

`openclaw.plugin.json` to metadane, które OpenClaw odczytuje, zanim załaduje kod
Twojego Pluginu.

Używaj go do:

- tożsamości Pluginu
- walidacji konfiguracji
- metadanych auth i onboardingu, które powinny być dostępne bez uruchamiania środowiska uruchomieniowego Pluginu
- lekkich wskazówek aktywacji, które powierzchnie płaszczyzny sterowania mogą sprawdzać przed załadowaniem środowiska uruchomieniowego
- lekkich deskryptorów konfiguracji, które powierzchnie konfiguracji/onboardingu mogą sprawdzać przed załadowaniem środowiska uruchomieniowego
- metadanych aliasów i auto-enable, które powinny zostać rozstrzygnięte przed załadowaniem środowiska uruchomieniowego Pluginu
- skróconych metadanych własności rodzin modeli, które powinny auto-activate
  Plugin przed załadowaniem środowiska uruchomieniowego
- statycznych snapshotów własności capabilities używanych do zgodnego okablowania bundli i pokrycia kontraktów
- metadanych konfiguracji specyficznych dla kanału, które powinny zostać scalone z powierzchniami katalogu i walidacji
  bez ładowania środowiska uruchomieniowego
- wskazówek UI konfiguracji

Nie używaj go do:

- rejestrowania zachowania środowiska uruchomieniowego
- deklarowania entrypointów kodu
- metadanych instalacji npm

To należy do kodu Pluginu i `package.json`.

## Minimalny przykład

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Rozbudowany przykład

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "Plugin dostawcy OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Klucz API OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Klucz API OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Klucz API",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Informacje o polach najwyższego poziomu

| Pole                                | Wymagane | Typ                              | Co oznacza                                                                                                                                                                                                   |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Tak      | `string`                         | Kanoniczny identyfikator Pluginu. To identyfikator używany w `plugins.entries.<id>`.                                                                                                                        |
| `configSchema`                      | Tak      | `object`                         | Wbudowany schemat JSON dla konfiguracji tego Pluginu.                                                                                                                                                        |
| `enabledByDefault`                  | Nie      | `true`                           | Oznacza bundlowy Plugin jako domyślnie włączony. Pomiń to pole albo ustaw dowolną wartość inną niż `true`, aby pozostawić Plugin domyślnie wyłączony.                                                     |
| `legacyPluginIds`                   | Nie      | `string[]`                       | Starsze identyfikatory, które są normalizowane do tego kanonicznego identyfikatora Pluginu.                                                                                                                 |
| `autoEnableWhenConfiguredProviders` | Nie      | `string[]`                       | Identyfikatory dostawców, które powinny automatycznie włączać ten Plugin, gdy auth, konfiguracja lub odwołania do modeli na nie wskazują.                                                                  |
| `kind`                              | Nie      | `"memory"` \| `"context-engine"` | Deklaruje wyłączny rodzaj Pluginu używany przez `plugins.slots.*`.                                                                                                                                          |
| `channels`                          | Nie      | `string[]`                       | Identyfikatory kanałów należących do tego Pluginu. Używane do wykrywania i walidacji konfiguracji.                                                                                                          |
| `providers`                         | Nie      | `string[]`                       | Identyfikatory dostawców należących do tego Pluginu.                                                                                                                                                         |
| `modelSupport`                      | Nie      | `object`                         | Należące do manifestu skrócone metadane rodzin modeli używane do automatycznego załadowania Pluginu przed środowiskiem uruchomieniowym.                                                                    |
| `cliBackends`                       | Nie      | `string[]`                       | Identyfikatory backendów inferencji CLI należących do tego Pluginu. Używane do automatycznej aktywacji przy uruchomieniu na podstawie jawnych odwołań w konfiguracji.                                     |
| `commandAliases`                    | Nie      | `object[]`                       | Nazwy poleceń należące do tego Pluginu, które powinny generować diagnostykę konfiguracji i CLI uwzględniającą Plugin jeszcze przed załadowaniem środowiska uruchomieniowego.                              |
| `providerAuthEnvVars`               | Nie      | `Record<string, string[]>`       | Lekkie metadane env dla auth dostawców, które OpenClaw może sprawdzać bez ładowania kodu Pluginu.                                                                                                          |
| `providerAuthAliases`               | Nie      | `Record<string, string>`         | Identyfikatory dostawców, które powinny ponownie używać innego identyfikatora dostawcy do wyszukiwania auth, na przykład dostawca kodowania, który współdzieli bazowy klucz API dostawcy i profile auth. |
| `channelEnvVars`                    | Nie      | `Record<string, string[]>`       | Lekkie metadane env dla kanałów, które OpenClaw może sprawdzać bez ładowania kodu Pluginu. Używaj tego dla konfiguracji kanałów sterowanej przez env albo powierzchni auth, które powinny widzieć ogólne pomocniki uruchamiania/konfiguracji. |
| `providerAuthChoices`               | Nie      | `object[]`                       | Lekkie metadane wyboru auth dla selektorów onboardingu, rozstrzygania preferowanego dostawcy i prostego okablowania flag CLI.                                                                             |
| `activation`                        | Nie      | `object`                         | Lekkie wskazówki aktywacji dla ładowania wyzwalanego przez dostawcę, polecenie, kanał, trasę i capability. Tylko metadane; rzeczywiste zachowanie nadal należy do środowiska uruchomieniowego Pluginu.   |
| `setup`                             | Nie      | `object`                         | Lekkie deskryptory konfiguracji/onboardingu, które powierzchnie wykrywania i konfiguracji mogą sprawdzać bez ładowania środowiska uruchomieniowego Pluginu.                                               |
| `contracts`                         | Nie      | `object`                         | Statyczny snapshot bundlowych capabilities dla speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search i własności narzędzi. |
| `channelConfigs`                    | Nie      | `Record<string, object>`         | Metadane konfiguracji kanału należące do manifestu, scalane z powierzchniami wykrywania i walidacji przed załadowaniem środowiska uruchomieniowego.                                                       |
| `skills`                            | Nie      | `string[]`                       | Katalogi Skills do załadowania, względne względem katalogu głównego Pluginu.                                                                                                                                |
| `name`                              | Nie      | `string`                         | Przyjazna dla człowieka nazwa Pluginu.                                                                                                                                                                       |
| `description`                       | Nie      | `string`                         | Krótkie podsumowanie wyświetlane na powierzchniach Pluginu.                                                                                                                                                  |
| `version`                           | Nie      | `string`                         | Informacyjna wersja Pluginu.                                                                                                                                                                                 |
| `uiHints`                           | Nie      | `Record<string, object>`         | Etykiety UI, placeholdery i wskazówki dotyczące wrażliwości dla pól konfiguracji.                                                                                                                           |

## Informacje o `providerAuthChoices`

Każdy wpis `providerAuthChoices` opisuje jeden wybór onboardingu lub auth.
OpenClaw odczytuje to, zanim załaduje się środowisko uruchomieniowe dostawcy.

| Pole                  | Wymagane | Typ                                             | Co oznacza                                                                                               |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider`            | Tak      | `string`                                        | Identyfikator dostawcy, do którego należy ten wybór.                                                     |
| `method`              | Tak      | `string`                                        | Identyfikator metody auth, do której należy przekazać sterowanie.                                        |
| `choiceId`            | Tak      | `string`                                        | Stabilny identyfikator wyboru auth używany przez onboarding i przepływy CLI.                             |
| `choiceLabel`         | Nie      | `string`                                        | Etykieta widoczna dla użytkownika. Jeśli zostanie pominięta, OpenClaw użyje `choiceId`.                 |
| `choiceHint`          | Nie      | `string`                                        | Krótki tekst pomocniczy dla selektora.                                                                   |
| `assistantPriority`   | Nie      | `number`                                        | Niższe wartości są sortowane wcześniej w interaktywnych selektorach sterowanych przez asystenta.        |
| `assistantVisibility` | Nie      | `"visible"` \| `"manual-only"`                  | Ukrywa wybór w selektorach asystenta, nadal pozwalając na ręczny wybór w CLI.                            |
| `deprecatedChoiceIds` | Nie      | `string[]`                                      | Starsze identyfikatory wyboru, które powinny przekierowywać użytkowników do tego wyboru zastępczego.    |
| `groupId`             | Nie      | `string`                                        | Opcjonalny identyfikator grupy do grupowania powiązanych wyborów.                                        |
| `groupLabel`          | Nie      | `string`                                        | Etykieta tej grupy widoczna dla użytkownika.                                                             |
| `groupHint`           | Nie      | `string`                                        | Krótki tekst pomocniczy dla grupy.                                                                       |
| `optionKey`           | Nie      | `string`                                        | Wewnętrzny klucz opcji dla prostych przepływów auth z jedną flagą.                                       |
| `cliFlag`             | Nie      | `string`                                        | Nazwa flagi CLI, taka jak `--openrouter-api-key`.                                                        |
| `cliOption`           | Nie      | `string`                                        | Pełna postać opcji CLI, taka jak `--openrouter-api-key <key>`.                                           |
| `cliDescription`      | Nie      | `string`                                        | Opis używany w pomocy CLI.                                                                               |
| `onboardingScopes`    | Nie      | `Array<"text-inference" \| "image-generation">` | Na których powierzchniach onboardingu ten wybór powinien się pojawiać. Jeśli zostanie pominięte, domyślnie używane jest `["text-inference"]`. |

## Informacje o `commandAliases`

Używaj `commandAliases`, gdy Plugin posiada nazwę polecenia środowiska uruchomieniowego, którą użytkownicy mogą
omyłkowo umieścić w `plugins.allow` albo próbować uruchomić jako główne polecenie CLI. OpenClaw
używa tych metadanych do diagnostyki bez importowania kodu środowiska uruchomieniowego Pluginu.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Pole         | Wymagane | Typ               | Co oznacza                                                                          |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------------------- |
| `name`       | Tak      | `string`          | Nazwa polecenia należąca do tego Pluginu.                                           |
| `kind`       | Nie      | `"runtime-slash"` | Oznacza alias jako polecenie ukośnikowe czatu, a nie główne polecenie CLI.         |
| `cliCommand` | Nie      | `string`          | Powiązane główne polecenie CLI, które należy zasugerować dla operacji CLI, jeśli istnieje. |

## Informacje o `activation`

Używaj `activation`, gdy Plugin może w prosty sposób zadeklarować, które zdarzenia płaszczyzny sterowania
powinny aktywować go później.

Ten blok zawiera wyłącznie metadane. Nie rejestruje zachowania środowiska uruchomieniowego i nie
zastępuje `register(...)`, `setupEntry` ani innych entrypointów środowiska uruchomieniowego/Pluginu.
Obecni konsumenci używają go jako wskazówki zawężającej przed szerszym ładowaniem Pluginów, więc
brak metadanych aktywacji zwykle wpływa tylko na wydajność; nie powinien zmieniać
poprawności, dopóki nadal istnieją starsze fallbacki własności manifestu.

```json
{
  "activation": {
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Pole             | Wymagane | Typ                                                  | Co oznacza                                                          |
| ---------------- | -------- | ---------------------------------------------------- | ------------------------------------------------------------------- |
| `onProviders`    | Nie      | `string[]`                                           | Identyfikatory dostawców, które powinny aktywować ten Plugin po zażądaniu. |
| `onCommands`     | Nie      | `string[]`                                           | Identyfikatory poleceń, które powinny aktywować ten Plugin.         |
| `onChannels`     | Nie      | `string[]`                                           | Identyfikatory kanałów, które powinny aktywować ten Plugin.         |
| `onRoutes`       | Nie      | `string[]`                                           | Rodzaje tras, które powinny aktywować ten Plugin.                   |
| `onCapabilities` | Nie      | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Szerokie wskazówki capabilities używane przez planowanie aktywacji płaszczyzny sterowania. |

Obecni aktywni konsumenci:

- planowanie CLI wyzwalane poleceniem wraca awaryjnie do starszych
  `commandAliases[].cliCommand` albo `commandAliases[].name`
- planowanie konfiguracji/kanałów wyzwalane kanałem wraca awaryjnie do starszej własności
  `channels[]`, gdy brakuje jawnych metadanych aktywacji kanału
- planowanie konfiguracji/środowiska uruchomieniowego wyzwalane dostawcą wraca awaryjnie do starszej
  własności `providers[]` i `cliBackends[]` najwyższego poziomu, gdy brakuje jawnych metadanych
  aktywacji dostawcy

## Informacje o `setup`

Używaj `setup`, gdy powierzchnie konfiguracji i onboardingu potrzebują lekkich metadanych należących do Pluginu
przed załadowaniem środowiska uruchomieniowego.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

`cliBackends` najwyższego poziomu pozostaje prawidłowe i nadal opisuje backendy
inferencji CLI. `setup.cliBackends` to powierzchnia deskryptorów specyficzna dla konfiguracji
dla przepływów płaszczyzny sterowania/konfiguracji, które powinny pozostać wyłącznie metadanymi.

Jeśli są obecne, `setup.providers` i `setup.cliBackends` są preferowaną
powierzchnią wyszukiwania najpierw po deskryptorach dla wykrywania konfiguracji. Jeśli deskryptor tylko
zawęża kandydujący Plugin, a konfiguracja nadal potrzebuje bogatszych hooków środowiska uruchomieniowego w czasie konfiguracji,
ustaw `requiresRuntime: true` i pozostaw `setup-api` jako
awaryjną ścieżkę wykonania.

Ponieważ wyszukiwanie konfiguracji może wykonywać należący do Pluginu kod `setup-api`,
znormalizowane wartości `setup.providers[].id` i `setup.cliBackends[]` muszą pozostać unikalne globalnie
wśród wykrytych Pluginów. Niejednoznaczna własność kończy się bez wyboru zwycięzcy
według kolejności wykrycia.

### Informacje o `setup.providers`

| Pole          | Wymagane | Typ        | Co oznacza                                                                                  |
| ------------- | -------- | ---------- | -------------------------------------------------------------------------------------------- |
| `id`          | Tak      | `string`   | Identyfikator dostawcy udostępniany podczas konfiguracji lub onboardingu. Znormalizowane identyfikatory muszą być globalnie unikalne. |
| `authMethods` | Nie      | `string[]` | Identyfikatory metod konfiguracji/auth obsługiwanych przez tego dostawcę bez ładowania pełnego środowiska uruchomieniowego. |
| `envVars`     | Nie      | `string[]` | Zmienne env, które ogólne powierzchnie konfiguracji/statusu mogą sprawdzać przed załadowaniem środowiska uruchomieniowego Pluginu. |

### Pola `setup`

| Pole               | Wymagane | Typ        | Co oznacza                                                                                          |
| ------------------ | -------- | ---------- | ---------------------------------------------------------------------------------------------------- |
| `providers`        | Nie      | `object[]` | Deskryptory konfiguracji dostawców udostępniane podczas konfiguracji i onboardingu.                 |
| `cliBackends`      | Nie      | `string[]` | Identyfikatory backendów czasu konfiguracji używane do wyszukiwania konfiguracji najpierw po deskryptorach. Znormalizowane identyfikatory muszą być globalnie unikalne. |
| `configMigrations` | Nie      | `string[]` | Identyfikatory migracji konfiguracji należące do powierzchni konfiguracji tego Pluginu.             |
| `requiresRuntime`  | Nie      | `boolean`  | Czy konfiguracja nadal wymaga wykonania `setup-api` po wyszukaniu deskryptora.                      |

## Informacje o `uiHints`

`uiHints` to mapa od nazw pól konfiguracji do małych wskazówek renderowania.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Klucz API",
      "help": "Używany do żądań OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Każda wskazówka pola może zawierać:

| Pole          | Typ        | Co oznacza                               |
| ------------- | ---------- | ---------------------------------------- |
| `label`       | `string`   | Etykieta pola widoczna dla użytkownika.  |
| `help`        | `string`   | Krótki tekst pomocniczy.                 |
| `tags`        | `string[]` | Opcjonalne tagi UI.                      |
| `advanced`    | `boolean`  | Oznacza pole jako zaawansowane.          |
| `sensitive`   | `boolean`  | Oznacza pole jako sekretne lub wrażliwe. |
| `placeholder` | `string`   | Tekst placeholdera dla pól formularza.   |

## Informacje o `contracts`

Używaj `contracts` wyłącznie dla statycznych metadanych własności capabilities, które OpenClaw może
odczytać bez importowania środowiska uruchomieniowego Pluginu.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Każda lista jest opcjonalna:

| Pole                             | Typ        | Co oznacza                                                   |
| -------------------------------- | ---------- | ------------------------------------------------------------ |
| `speechProviders`                | `string[]` | Identyfikatory dostawców speech należących do tego Pluginu.  |
| `realtimeTranscriptionProviders` | `string[]` | Identyfikatory dostawców realtime transcription należących do tego Pluginu. |
| `realtimeVoiceProviders`         | `string[]` | Identyfikatory dostawców realtime voice należących do tego Pluginu. |
| `mediaUnderstandingProviders`    | `string[]` | Identyfikatory dostawców media-understanding należących do tego Pluginu. |
| `imageGenerationProviders`       | `string[]` | Identyfikatory dostawców image-generation należących do tego Pluginu. |
| `videoGenerationProviders`       | `string[]` | Identyfikatory dostawców video-generation należących do tego Pluginu. |
| `webFetchProviders`              | `string[]` | Identyfikatory dostawców web-fetch należących do tego Pluginu. |
| `webSearchProviders`             | `string[]` | Identyfikatory dostawców web search należących do tego Pluginu. |
| `tools`                          | `string[]` | Nazwy narzędzi agenta należących do tego Pluginu dla bundlowych kontroli kontraktów. |

## Informacje o `channelConfigs`

Używaj `channelConfigs`, gdy Plugin kanału potrzebuje lekkich metadanych konfiguracji przed
załadowaniem środowiska uruchomieniowego.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "URL Homeservera",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Połączenie z homeserverem Matrix",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Każdy wpis kanału może zawierać:

| Pole          | Typ                      | Co oznacza                                                                                  |
| ------------- | ------------------------ | -------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | Schemat JSON dla `channels.<id>`. Wymagany dla każdego zadeklarowanego wpisu konfiguracji kanału. |
| `uiHints`     | `Record<string, object>` | Opcjonalne etykiety UI/placeholdery/wskazówki dotyczące wrażliwości dla tej sekcji konfiguracji kanału. |
| `label`       | `string`                 | Etykieta kanału scalana z powierzchniami wyboru i inspekcji, gdy metadane środowiska uruchomieniowego nie są jeszcze gotowe. |
| `description` | `string`                 | Krótki opis kanału dla powierzchni inspekcji i katalogu.                                     |
| `preferOver`  | `string[]`               | Starsze lub mniej priorytetowe identyfikatory Pluginów, które ten kanał powinien wyprzedzać na powierzchniach wyboru. |

## Informacje o `modelSupport`

Używaj `modelSupport`, gdy OpenClaw powinien wywnioskować Plugin dostawcy na podstawie
skróconych identyfikatorów modeli, takich jak `gpt-5.4` albo `claude-sonnet-4.6`, zanim załaduje się środowisko uruchomieniowe Pluginu.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw stosuje następujący priorytet:

- jawne odwołania `provider/model` używają należących do właściciela metadanych manifestu `providers`
- `modelPatterns` mają wyższy priorytet niż `modelPrefixes`
- jeśli pasuje jeden Plugin zewnętrzny i jeden Plugin bundlowy, wygrywa Plugin zewnętrzny
- pozostała niejednoznaczność jest ignorowana, dopóki użytkownik lub konfiguracja nie wskaże dostawcy

Pola:

| Pole            | Typ        | Co oznacza                                                                     |
| --------------- | ---------- | ------------------------------------------------------------------------------ |
| `modelPrefixes` | `string[]` | Prefiksy dopasowywane przez `startsWith` do skróconych identyfikatorów modeli. |
| `modelPatterns` | `string[]` | Źródła regex dopasowywane do skróconych identyfikatorów modeli po usunięciu sufiksu profilu. |

Starsze klucze capabilities najwyższego poziomu są przestarzałe. Użyj `openclaw doctor --fix`, aby
przenieść `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` i `webSearchProviders` do `contracts`; zwykłe
ładowanie manifestu nie traktuje już tych pól najwyższego poziomu jako
własności capabilities.

## Manifest a `package.json`

Te dwa pliki służą do różnych zadań:

| Plik                   | Używaj go do                                                                                                                       |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Wykrywania, walidacji konfiguracji, metadanych wyboru auth i wskazówek UI, które muszą istnieć przed uruchomieniem kodu Pluginu |
| `package.json`         | Metadanych npm, instalacji zależności oraz bloku `openclaw` używanego do entrypointów, bramek instalacji, konfiguracji lub metadanych katalogu |

Jeśli nie masz pewności, gdzie powinien trafić dany fragment metadanych, stosuj tę zasadę:

- jeśli OpenClaw musi znać go przed załadowaniem kodu Pluginu, umieść go w `openclaw.plugin.json`
- jeśli dotyczy pakowania, plików wejściowych albo zachowania instalacji npm, umieść go w `package.json`

### Pola `package.json`, które wpływają na wykrywanie

Niektóre metadane Pluginów przed uruchomieniem celowo znajdują się w `package.json` w bloku
`openclaw`, a nie w `openclaw.plugin.json`.

Ważne przykłady:

| Pole                                                              | Co oznacza                                                                                                                                  |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Deklaruje natywne entrypointy Pluginów.                                                                                                      |
| `openclaw.setupEntry`                                             | Lekki entrypoint tylko do konfiguracji, używany podczas onboardingu i odroczonego uruchamiania kanału.                                     |
| `openclaw.channel`                                                | Lekkie metadane katalogu kanałów, takie jak etykiety, ścieżki do dokumentacji, aliasy i tekst wyboru.                                      |
| `openclaw.channel.configuredState`                                | Lekkie metadane sprawdzania stanu konfiguracji, które mogą odpowiedzieć na pytanie „czy konfiguracja oparta wyłącznie na env już istnieje?” bez ładowania pełnego środowiska uruchomieniowego kanału. |
| `openclaw.channel.persistedAuthState`                             | Lekkie metadane sprawdzania trwałego stanu auth, które mogą odpowiedzieć na pytanie „czy cokolwiek jest już zalogowane?” bez ładowania pełnego środowiska uruchomieniowego kanału. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Wskazówki instalacji/aktualizacji dla Pluginów bundlowych i publikowanych zewnętrznie.                                                      |
| `openclaw.install.defaultChoice`                                  | Preferowana ścieżka instalacji, gdy dostępnych jest wiele źródeł instalacji.                                                               |
| `openclaw.install.minHostVersion`                                 | Minimalna obsługiwana wersja hosta OpenClaw, z użyciem dolnego ograniczenia semver, takiego jak `>=2026.3.22`.                             |
| `openclaw.install.allowInvalidConfigRecovery`                     | Umożliwia wąską ścieżkę odzyskiwania po ponownej instalacji bundlowego Pluginu, gdy konfiguracja jest nieprawidłowa.                      |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Pozwala powierzchniom kanałów tylko do konfiguracji załadować się przed pełnym Pluginem kanału podczas uruchamiania.                      |

`openclaw.install.minHostVersion` jest wymuszane podczas instalacji i
ładowania rejestru manifestów. Nieprawidłowe wartości są odrzucane; nowsze, ale poprawne wartości powodują pominięcie
Pluginu na starszych hostach.

`openclaw.install.allowInvalidConfigRecovery` jest celowo wąskie. Nie
sprawia, że dowolnie uszkodzone konfiguracje stają się możliwe do zainstalowania. Obecnie pozwala tylko
przepływom instalacji odzyskać sprawność po określonych niepowodzeniach aktualizacji bundlowych Pluginów, takich jak
brak ścieżki bundlowego Pluginu albo nieaktualny wpis `channels.<id>` dla tego samego
bundlowego Pluginu. Niezwiązane błędy konfiguracji nadal blokują instalację i kierują operatorów
do `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` to metadane pakietu dla małego modułu
sprawdzającego:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Używaj tego, gdy przepływy konfiguracji, doctor albo configured-state potrzebują prostego
sprawdzenia auth typu tak/nie, zanim załaduje się pełny Plugin kanału. Docelowy eksport powinien być małą
funkcją, która odczytuje wyłącznie stan trwały; nie kieruj tego przez pełny barrel środowiska uruchomieniowego kanału.

`openclaw.channel.configuredState` ma taki sam kształt dla lekkich sprawdzeń
konfiguracji opartych wyłącznie na env:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Używaj tego, gdy kanał może odpowiedzieć o stanie konfiguracji na podstawie env lub innych małych
wejść niezwiązanych ze środowiskiem uruchomieniowym. Jeśli sprawdzenie wymaga pełnego rozstrzygnięcia konfiguracji albo rzeczywistego
środowiska uruchomieniowego kanału, pozostaw tę logikę w hooku Pluginu `config.hasConfiguredState`.

## Wymagania schematu JSON

- **Każdy Plugin musi dostarczać schemat JSON**, nawet jeśli nie przyjmuje żadnej konfiguracji.
- Pusty schemat jest akceptowalny (na przykład `{ "type": "object", "additionalProperties": false }`).
- Schematy są walidowane podczas odczytu/zapisu konfiguracji, a nie w czasie działania.

## Zachowanie walidacji

- Nieznane klucze `channels.*` są **błędami**, chyba że identyfikator kanału jest zadeklarowany przez
  manifest Pluginu.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` i `plugins.slots.*`
  muszą odwoływać się do **wykrywalnych** identyfikatorów Pluginów. Nieznane identyfikatory są **błędami**.
- Jeśli Plugin jest zainstalowany, ale ma uszkodzony lub brakujący manifest albo schemat,
  walidacja kończy się niepowodzeniem, a Doctor zgłasza błąd Pluginu.
- Jeśli konfiguracja Pluginu istnieje, ale Plugin jest **wyłączony**, konfiguracja jest zachowywana i
  w Doctor + logach wyświetlane jest **ostrzeżenie**.

Pełny schemat `plugins.*` znajdziesz tutaj: [Informacje o konfiguracji](/pl/gateway/configuration).

## Uwagi

- Manifest jest **wymagany dla natywnych Pluginów OpenClaw**, w tym dla ładowania z lokalnego systemu plików.
- Środowisko uruchomieniowe nadal ładuje moduł Pluginu osobno; manifest służy tylko do
  wykrywania + walidacji.
- Natywne manifesty są parsowane przy użyciu JSON5, więc komentarze, końcowe przecinki i
  klucze bez cudzysłowów są akceptowane, o ile wartość końcowa nadal jest obiektem.
- Loader manifestu odczytuje tylko udokumentowane pola manifestu. Unikaj dodawania
  tutaj własnych kluczy najwyższego poziomu.
- `providerAuthEnvVars` to lekka ścieżka metadanych dla sond auth, walidacji
  znaczników env i podobnych powierzchni auth dostawców, które nie powinny uruchamiać środowiska uruchomieniowego Pluginu tylko po to, by sprawdzić nazwy env.
- `providerAuthAliases` pozwala wariantom dostawców ponownie używać zmiennych env auth,
  profili auth, auth opartych na konfiguracji oraz wyboru onboardingu klucza API innego dostawcy
  bez kodowania tej relacji na sztywno w core.
- `channelEnvVars` to lekka ścieżka metadanych dla fallbacku shell-env, promptów konfiguracji
  i podobnych powierzchni kanałów, które nie powinny uruchamiać środowiska uruchomieniowego Pluginu
  tylko po to, by sprawdzić nazwy env.
- `providerAuthChoices` to lekka ścieżka metadanych dla selektorów wyboru auth,
  rozstrzygania `--auth-choice`, mapowania preferowanego dostawcy i prostej rejestracji flag CLI
  onboardingu przed załadowaniem środowiska uruchomieniowego dostawcy. W przypadku metadanych kreatora środowiska uruchomieniowego,
  które wymagają kodu dostawcy, zobacz
  [Hooki środowiska uruchomieniowego dostawcy](/pl/plugins/architecture#provider-runtime-hooks).
- Wyłączne rodzaje Pluginów są wybierane przez `plugins.slots.*`.
  - `kind: "memory"` jest wybierany przez `plugins.slots.memory`.
  - `kind: "context-engine"` jest wybierany przez `plugins.slots.contextEngine`
    (domyślnie: wbudowany `legacy`).
- `channels`, `providers`, `cliBackends` i `skills` można pominąć, jeśli
  Plugin ich nie potrzebuje.
- Jeśli Twój Plugin zależy od modułów natywnych, udokumentuj kroki budowania oraz wszelkie
  wymagania dotyczące allowlist menedżera pakietów (na przykład pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Powiązane

- [Tworzenie Pluginów](/pl/plugins/building-plugins) — pierwsze kroki z Pluginami
- [Architektura Pluginów](/pl/plugins/architecture) — architektura wewnętrzna
- [Przegląd SDK](/pl/plugins/sdk-overview) — dokumentacja Plugin SDK
