---
read_when:
    - Tworzysz plugin OpenClaw
    - Musisz dostarczyć schemat konfiguracji pluginu lub debugować błędy walidacji pluginu
summary: Manifest pluginu + wymagania schematu JSON (ścisła walidacja konfiguracji)
title: Manifest pluginu
x-i18n:
    generated_at: "2026-04-09T01:29:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9a7ee4b621a801d2a8f32f8976b0e1d9433c7810eb360aca466031fc0ffb286a
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifest pluginu (openclaw.plugin.json)

Ta strona dotyczy wyłącznie **natywnego manifestu pluginu OpenClaw**.

Informacje o zgodnych układach bundli znajdziesz w [Bundlach pluginów](/pl/plugins/bundles).

Zgodne formaty bundli używają innych plików manifestu:

- Bundel Codex: `.codex-plugin/plugin.json`
- Bundel Claude: `.claude-plugin/plugin.json` lub domyślny układ komponentu Claude
  bez manifestu
- Bundel Cursor: `.cursor-plugin/plugin.json`

OpenClaw automatycznie wykrywa również te układy bundli, ale nie są one walidowane
względem schematu `openclaw.plugin.json` opisanego tutaj.

Dla zgodnych bundli OpenClaw obecnie odczytuje metadane bundla oraz zadeklarowane
korzenie skillów, korzenie poleceń Claude, domyślne wartości `settings.json` bundla Claude,
domyślne wartości LSP bundla Claude oraz obsługiwane zestawy hooków, gdy układ odpowiada
oczekiwaniom środowiska uruchomieniowego OpenClaw.

Każdy natywny plugin OpenClaw **musi** dostarczać plik `openclaw.plugin.json` w
**katalogu głównym pluginu**. OpenClaw używa tego manifestu do walidacji konfiguracji
**bez wykonywania kodu pluginu**. Brakujące lub nieprawidłowe manifesty są traktowane jako
błędy pluginu i blokują walidację konfiguracji.

Pełny przewodnik po systemie pluginów znajdziesz tutaj: [Pluginy](/pl/tools/plugin).
Informacje o natywnym modelu możliwości i aktualnych wskazówkach dotyczących zgodności zewnętrznej:
[Model możliwości](/pl/plugins/architecture#public-capability-model).

## Do czego służy ten plik

`openclaw.plugin.json` to metadane, które OpenClaw odczytuje przed załadowaniem
kodu Twojego pluginu.

Używaj go do:

- tożsamości pluginu
- walidacji konfiguracji
- metadanych auth i onboardingu, które powinny być dostępne bez uruchamiania
  środowiska uruchomieniowego pluginu
- metadanych aliasów i automatycznego włączania, które powinny być rozstrzygane przed załadowaniem środowiska uruchomieniowego pluginu
- skróconych metadanych własności rodzin modeli, które powinny automatycznie aktywować
  plugin przed załadowaniem środowiska uruchomieniowego
- statycznych migawek własności możliwości używanych do zgodnego okablowania bundli i
  pokrycia kontraktów
- metadanych konfiguracji specyficznych dla kanału, które powinny być scalane z powierzchniami katalogu i walidacji
  bez ładowania środowiska uruchomieniowego
- wskazówek UI konfiguracji

Nie używaj go do:

- rejestrowania zachowań środowiska uruchomieniowego
- deklarowania punktów wejścia kodu
- metadanych instalacji npm

To należy do kodu pluginu i `package.json`.

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

## Dokumentacja pól najwyższego poziomu

| Field                               | Required | Type                             | What it means                                                                                                                                                                                                |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Tak      | `string`                         | Kanoniczny identyfikator pluginu. To identyfikator używany w `plugins.entries.<id>`.                                                                                                                        |
| `configSchema`                      | Tak      | `object`                         | Wbudowany schemat JSON dla konfiguracji tego pluginu.                                                                                                                                                        |
| `enabledByDefault`                  | Nie      | `true`                           | Oznacza bundlowany plugin jako domyślnie włączony. Pomiń to pole albo ustaw dowolną wartość inną niż `true`, aby pozostawić plugin domyślnie wyłączony.                                                   |
| `legacyPluginIds`                   | Nie      | `string[]`                       | Starsze identyfikatory normalizowane do tego kanonicznego identyfikatora pluginu.                                                                                                                           |
| `autoEnableWhenConfiguredProviders` | Nie      | `string[]`                       | Identyfikatory dostawców, które powinny automatycznie włączać ten plugin, gdy auth, konfiguracja lub odwołania do modeli o nich wspominają.                                                               |
| `kind`                              | Nie      | `"memory"` \| `"context-engine"` | Deklaruje wyłączny rodzaj pluginu używany przez `plugins.slots.*`.                                                                                                                                          |
| `channels`                          | Nie      | `string[]`                       | Identyfikatory kanałów należących do tego pluginu. Używane do wykrywania i walidacji konfiguracji.                                                                                                          |
| `providers`                         | Nie      | `string[]`                       | Identyfikatory dostawców należących do tego pluginu.                                                                                                                                                         |
| `modelSupport`                      | Nie      | `object`                         | Należące do manifestu skrócone metadane rodzin modeli używane do automatycznego ładowania pluginu przed środowiskiem uruchomieniowym.                                                                      |
| `cliBackends`                       | Nie      | `string[]`                       | Identyfikatory backendów inferencji CLI należących do tego pluginu. Używane do automatycznej aktywacji przy uruchomieniu na podstawie jawnych odwołań w konfiguracji.                                      |
| `providerAuthEnvVars`               | Nie      | `Record<string, string[]>`       | Tanie metadane env dla auth dostawcy, które OpenClaw może sprawdzać bez ładowania kodu pluginu.                                                                                                             |
| `providerAuthAliases`               | Nie      | `Record<string, string>`         | Identyfikatory dostawców, które powinny ponownie używać innego identyfikatora dostawcy do wyszukiwania auth, na przykład dostawca do kodowania współdzielący bazowy klucz API dostawcy i profile auth.   |
| `channelEnvVars`                    | Nie      | `Record<string, string[]>`       | Tanie metadane env kanału, które OpenClaw może sprawdzać bez ładowania kodu pluginu. Używaj tego dla konfiguracji kanału opartej na env lub powierzchni auth, które powinny być widoczne dla ogólnych pomocników uruchamiania/konfiguracji. |
| `providerAuthChoices`               | Nie      | `object[]`                       | Tanie metadane wyboru auth dla selektorów onboardingu, rozstrzygania preferowanych dostawców i prostego mapowania flag CLI.                                                                                |
| `contracts`                         | Nie      | `object`                         | Statyczna migawka możliwości bundli dla mowy, transkrypcji czasu rzeczywistego, głosu czasu rzeczywistego, rozumienia mediów, generowania obrazów, generowania muzyki, generowania wideo, web-fetch, wyszukiwania w sieci i własności narzędzi. |
| `channelConfigs`                    | Nie      | `Record<string, object>`         | Należące do manifestu metadane konfiguracji kanałów scalane z powierzchniami wykrywania i walidacji przed załadowaniem środowiska uruchomieniowego.                                                        |
| `skills`                            | Nie      | `string[]`                       | Katalogi Skills do załadowania, względem katalogu głównego pluginu.                                                                                                                                         |
| `name`                              | Nie      | `string`                         | Czytelna dla człowieka nazwa pluginu.                                                                                                                                                                        |
| `description`                       | Nie      | `string`                         | Krótkie podsumowanie wyświetlane na powierzchniach pluginu.                                                                                                                                                  |
| `version`                           | Nie      | `string`                         | Informacyjna wersja pluginu.                                                                                                                                                                                 |
| `uiHints`                           | Nie      | `Record<string, object>`         | Etykiety UI, placeholdery i wskazówki dotyczące wrażliwości dla pól konfiguracji.                                                                                                                           |

## Dokumentacja `providerAuthChoices`

Każdy wpis `providerAuthChoices` opisuje jeden wybór onboardingu lub auth.
OpenClaw odczytuje to przed załadowaniem środowiska uruchomieniowego dostawcy.

| Field                 | Required | Type                                            | What it means                                                                                            |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider`            | Tak      | `string`                                        | Identyfikator dostawcy, do którego należy ten wybór.                                                     |
| `method`              | Tak      | `string`                                        | Identyfikator metody auth, do której należy przekierować.                                                |
| `choiceId`            | Tak      | `string`                                        | Stabilny identyfikator wyboru auth używany przez onboarding i przepływy CLI.                            |
| `choiceLabel`         | Nie      | `string`                                        | Etykieta widoczna dla użytkownika. Jeśli zostanie pominięta, OpenClaw użyje `choiceId`.                |
| `choiceHint`          | Nie      | `string`                                        | Krótki tekst pomocniczy dla selektora.                                                                   |
| `assistantPriority`   | Nie      | `number`                                        | Niższe wartości są sortowane wcześniej w interaktywnych selektorach sterowanych przez asystenta.        |
| `assistantVisibility` | Nie      | `"visible"` \| `"manual-only"`                  | Ukrywa wybór w selektorach asystenta, nadal pozwalając na ręczny wybór w CLI.                           |
| `deprecatedChoiceIds` | Nie      | `string[]`                                      | Starsze identyfikatory wyborów, które powinny przekierowywać użytkowników do tego wyboru zastępczego.   |
| `groupId`             | Nie      | `string`                                        | Opcjonalny identyfikator grupy do grupowania powiązanych wyborów.                                       |
| `groupLabel`          | Nie      | `string`                                        | Etykieta tej grupy widoczna dla użytkownika.                                                             |
| `groupHint`           | Nie      | `string`                                        | Krótki tekst pomocniczy dla grupy.                                                                       |
| `optionKey`           | Nie      | `string`                                        | Wewnętrzny klucz opcji dla prostych przepływów auth opartych na jednej fladze.                          |
| `cliFlag`             | Nie      | `string`                                        | Nazwa flagi CLI, np. `--openrouter-api-key`.                                                             |
| `cliOption`           | Nie      | `string`                                        | Pełny kształt opcji CLI, np. `--openrouter-api-key <key>`.                                               |
| `cliDescription`      | Nie      | `string`                                        | Opis używany w pomocy CLI.                                                                               |
| `onboardingScopes`    | Nie      | `Array<"text-inference" \| "image-generation">` | Na których powierzchniach onboardingu ten wybór powinien się pojawiać. Jeśli pole zostanie pominięte, domyślnie używane jest `["text-inference"]`. |

## Dokumentacja `uiHints`

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

Wskazówka dla każdego pola może zawierać:

| Field         | Type       | What it means                           |
| ------------- | ---------- | --------------------------------------- |
| `label`       | `string`   | Etykieta pola widoczna dla użytkownika. |
| `help`        | `string`   | Krótki tekst pomocniczy.                |
| `tags`        | `string[]` | Opcjonalne tagi UI.                     |
| `advanced`    | `boolean`  | Oznacza pole jako zaawansowane.         |
| `sensitive`   | `boolean`  | Oznacza pole jako sekretne lub wrażliwe. |
| `placeholder` | `string`   | Tekst placeholdera dla pól formularza.  |

## Dokumentacja `contracts`

Używaj `contracts` tylko dla statycznych metadanych własności możliwości, które OpenClaw może
odczytać bez importowania środowiska uruchomieniowego pluginu.

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

| Field                            | Type       | What it means                                                  |
| -------------------------------- | ---------- | -------------------------------------------------------------- |
| `speechProviders`                | `string[]` | Identyfikatory dostawców mowy należących do tego pluginu.      |
| `realtimeTranscriptionProviders` | `string[]` | Identyfikatory dostawców transkrypcji czasu rzeczywistego należących do tego pluginu. |
| `realtimeVoiceProviders`         | `string[]` | Identyfikatory dostawców głosu czasu rzeczywistego należących do tego pluginu. |
| `mediaUnderstandingProviders`    | `string[]` | Identyfikatory dostawców rozumienia mediów należących do tego pluginu. |
| `imageGenerationProviders`       | `string[]` | Identyfikatory dostawców generowania obrazów należących do tego pluginu. |
| `videoGenerationProviders`       | `string[]` | Identyfikatory dostawców generowania wideo należących do tego pluginu. |
| `webFetchProviders`              | `string[]` | Identyfikatory dostawców web-fetch należących do tego pluginu. |
| `webSearchProviders`             | `string[]` | Identyfikatory dostawców wyszukiwania w sieci należących do tego pluginu. |
| `tools`                          | `string[]` | Nazwy narzędzi agenta należących do tego pluginu do sprawdzania kontraktów bundli. |

## Dokumentacja `channelConfigs`

Używaj `channelConfigs`, gdy plugin kanału potrzebuje tanich metadanych konfiguracji przed
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
          "label": "URL homeservera",
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

| Field         | Type                     | What it means                                                                             |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | Schemat JSON dla `channels.<id>`. Wymagany dla każdego zadeklarowanego wpisu konfiguracji kanału. |
| `uiHints`     | `Record<string, object>` | Opcjonalne etykiety UI/placeholdery/wskazówki dotyczące wrażliwości dla tej sekcji konfiguracji kanału. |
| `label`       | `string`                 | Etykieta kanału scalana z powierzchniami selektora i inspekcji, gdy metadane środowiska uruchomieniowego nie są gotowe. |
| `description` | `string`                 | Krótki opis kanału dla powierzchni inspekcji i katalogu.                                 |
| `preferOver`  | `string[]`               | Starsze lub niższe priorytetowo identyfikatory pluginów, które ten kanał powinien wyprzedzać na powierzchniach wyboru. |

## Dokumentacja `modelSupport`

Używaj `modelSupport`, gdy OpenClaw powinien wywnioskować plugin Twojego dostawcy na podstawie
skrótowych identyfikatorów modeli, takich jak `gpt-5.4` lub `claude-sonnet-4.6`, przed załadowaniem środowiska uruchomieniowego pluginu.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw stosuje następujący priorytet:

- jawne odwołania `provider/model` używają należących do manifestu metadanych `providers`
- `modelPatterns` mają pierwszeństwo przed `modelPrefixes`
- jeśli jeden plugin niebundlowany i jeden plugin bundlowany pasują jednocześnie, wygrywa plugin niebundlowany
- pozostała niejednoznaczność jest ignorowana, dopóki użytkownik lub konfiguracja nie określi dostawcy

Pola:

| Field           | Type       | What it means                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefiksy dopasowywane przez `startsWith` do skrótowych identyfikatorów modeli.  |
| `modelPatterns` | `string[]` | Źródła regex dopasowywane do skrótowych identyfikatorów modeli po usunięciu sufiksu profilu. |

Starsze klucze możliwości na najwyższym poziomie są przestarzałe. Użyj `openclaw doctor --fix`, aby
przenieść `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` i `webSearchProviders` do `contracts`; standardowe
ładowanie manifestu nie traktuje już tych pól najwyższego poziomu jako
własności możliwości.

## Manifest a package.json

Te dwa pliki pełnią różne role:

| File                   | Use it for                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Wykrywanie, walidacja konfiguracji, metadane wyboru auth i wskazówki UI, które muszą istnieć przed uruchomieniem kodu pluginu |
| `package.json`         | Metadane npm, instalacja zależności i blok `openclaw` używany dla punktów wejścia, bramkowania instalacji, konfiguracji lub metadanych katalogu |

Jeśli nie masz pewności, gdzie powinien trafić dany element metadanych, użyj tej zasady:

- jeśli OpenClaw musi znać tę informację przed załadowaniem kodu pluginu, umieść ją w `openclaw.plugin.json`
- jeśli dotyczy pakowania, plików wejściowych lub zachowania instalacji npm, umieść ją w `package.json`

### Pola package.json wpływające na wykrywanie

Niektóre metadane pluginu sprzed uruchomienia celowo znajdują się w `package.json` w bloku
`openclaw`, a nie w `openclaw.plugin.json`.

Ważne przykłady:

| Field                                                             | What it means                                                                                                                                |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Deklaruje punkty wejścia natywnych pluginów.                                                                                                 |
| `openclaw.setupEntry`                                             | Lekki punkt wejścia tylko do konfiguracji używany podczas onboardingu i odroczonego uruchamiania kanału.                                    |
| `openclaw.channel`                                                | Tanie metadane katalogu kanałów, takie jak etykiety, ścieżki dokumentacji, aliasy i teksty wyboru.                                         |
| `openclaw.channel.configuredState`                                | Lekkie metadane sprawdzania stanu konfiguracji, które mogą odpowiedzieć na pytanie „czy konfiguracja oparta tylko na env już istnieje?” bez ładowania pełnego środowiska uruchomieniowego kanału. |
| `openclaw.channel.persistedAuthState`                             | Lekkie metadane sprawdzania utrwalonego stanu auth, które mogą odpowiedzieć na pytanie „czy coś jest już zalogowane?” bez ładowania pełnego środowiska uruchomieniowego kanału. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Wskazówki instalacji/aktualizacji dla bundlowanych i zewnętrznie publikowanych pluginów.                                                    |
| `openclaw.install.defaultChoice`                                  | Preferowana ścieżka instalacji, gdy dostępnych jest wiele źródeł instalacji.                                                                |
| `openclaw.install.minHostVersion`                                 | Minimalna obsługiwana wersja hosta OpenClaw, z użyciem dolnej granicy semver, np. `>=2026.3.22`.                                           |
| `openclaw.install.allowInvalidConfigRecovery`                     | Pozwala na wąską ścieżkę odzyskiwania przez ponowną instalację bundlowanego pluginu, gdy konfiguracja jest nieprawidłowa.                  |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Pozwala na ładowanie powierzchni kanału tylko do konfiguracji przed pełnym pluginem kanału podczas uruchamiania.                           |

`openclaw.install.minHostVersion` jest wymuszane podczas instalacji i ładowania rejestru
manifestów. Nieprawidłowe wartości są odrzucane; nowsze, ale poprawne wartości powodują pominięcie
pluginu na starszych hostach.

`openclaw.install.allowInvalidConfigRecovery` jest celowo wąskie. Nie
sprawia, że dowolnie uszkodzone konfiguracje stają się możliwe do zainstalowania. Dziś pozwala tylko przepływom instalacji
odzyskać sprawność po określonych nieaktualnych błędach aktualizacji bundlowanego pluginu, takich jak
brakująca ścieżka bundlowanego pluginu lub nieaktualny wpis `channels.<id>` dla tego samego
bundlowanego pluginu. Niezwiązane błędy konfiguracji nadal blokują instalację i kierują operatorów
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

Używaj tego, gdy konfiguracja, doctor lub przepływy stanu skonfigurowania potrzebują taniego sprawdzenia auth typu tak/nie
przed załadowaniem pełnego pluginu kanału. Eksport docelowy powinien być małą
funkcją, która odczytuje wyłącznie stan utrwalony; nie kieruj tego przez pełny
barrel środowiska uruchomieniowego kanału.

`openclaw.channel.configuredState` ma ten sam kształt dla tanich sprawdzeń
stanu skonfigurowania opartych wyłącznie na env:

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

Używaj tego, gdy kanał może odpowiedzieć na pytanie o stan skonfigurowania na podstawie env lub innych małych
wejść niezwiązanych ze środowiskiem uruchomieniowym. Jeśli sprawdzenie wymaga pełnego rozstrzygania konfiguracji lub rzeczywistego
środowiska uruchomieniowego kanału, pozostaw tę logikę w hooku pluginu `config.hasConfiguredState`.

## Wymagania schematu JSON

- **Każdy plugin musi dostarczać schemat JSON**, nawet jeśli nie akceptuje żadnej konfiguracji.
- Pusty schemat jest akceptowalny (na przykład `{ "type": "object", "additionalProperties": false }`).
- Schematy są walidowane podczas odczytu/zapisu konfiguracji, a nie w czasie działania.

## Zachowanie walidacji

- Nieznane klucze `channels.*` są **błędami**, chyba że identyfikator kanału jest deklarowany przez
  manifest pluginu.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` i `plugins.slots.*`
  muszą odwoływać się do **wykrywalnych** identyfikatorów pluginów. Nieznane identyfikatory są **błędami**.
- Jeśli plugin jest zainstalowany, ale ma uszkodzony lub brakujący manifest albo schemat,
  walidacja kończy się niepowodzeniem, a Doctor zgłasza błąd pluginu.
- Jeśli konfiguracja pluginu istnieje, ale plugin jest **wyłączony**, konfiguracja zostaje zachowana i
  w Doctor + logach pojawia się **ostrzeżenie**.

Pełny schemat `plugins.*` znajdziesz w [Dokumentacji konfiguracji](/pl/gateway/configuration).

## Uwagi

- Manifest jest **wymagany dla natywnych pluginów OpenClaw**, także przy ładowaniu z lokalnego systemu plików.
- Środowisko uruchomieniowe nadal ładuje moduł pluginu osobno; manifest służy wyłącznie do
  wykrywania + walidacji.
- Natywne manifesty są parsowane przy użyciu JSON5, więc komentarze, końcowe przecinki i
  klucze bez cudzysłowów są akceptowane, o ile końcowa wartość nadal jest obiektem.
- Loader manifestu odczytuje tylko udokumentowane pola manifestu. Unikaj dodawania
  tutaj niestandardowych kluczy najwyższego poziomu.
- `providerAuthEnvVars` to tania ścieżka metadanych dla sond auth, walidacji znaczników env
  i podobnych powierzchni auth dostawców, które nie powinny uruchamiać środowiska uruchomieniowego pluginu tylko po to, aby sprawdzić nazwy env.
- `providerAuthAliases` pozwala wariantom dostawców ponownie używać zmiennych env auth
  innego dostawcy, profili auth, auth opartych na konfiguracji oraz wyboru onboardingu klucza API
  bez kodowania tej relacji na sztywno w rdzeniu.
- `channelEnvVars` to tania ścieżka metadanych dla fallbacku shell-env, promptów konfiguracji
  i podobnych powierzchni kanałów, które nie powinny uruchamiać środowiska uruchomieniowego pluginu
  tylko po to, aby sprawdzić nazwy env.
- `providerAuthChoices` to tania ścieżka metadanych dla selektorów wyboru auth,
  rozstrzygania `--auth-choice`, mapowania preferowanych dostawców i prostej rejestracji
  flag CLI onboardingu przed załadowaniem środowiska uruchomieniowego dostawcy. Dla metadanych
  kreatora środowiska uruchomieniowego, które wymagają kodu dostawcy, zobacz
  [Hooki środowiska uruchomieniowego dostawcy](/pl/plugins/architecture#provider-runtime-hooks).
- Wyłączne rodzaje pluginów są wybierane przez `plugins.slots.*`.
  - `kind: "memory"` jest wybierane przez `plugins.slots.memory`.
  - `kind: "context-engine"` jest wybierane przez `plugins.slots.contextEngine`
    (domyślnie: wbudowane `legacy`).
- `channels`, `providers`, `cliBackends` i `skills` można pominąć, gdy
  plugin ich nie potrzebuje.
- Jeśli Twój plugin zależy od modułów natywnych, udokumentuj kroki budowania i wszelkie
  wymagania dotyczące listy dozwolonych menedżera pakietów (na przykład pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Powiązane

- [Tworzenie pluginów](/pl/plugins/building-plugins) — rozpoczęcie pracy z pluginami
- [Architektura pluginów](/pl/plugins/architecture) — architektura wewnętrzna
- [Przegląd SDK](/pl/plugins/sdk-overview) — dokumentacja Plugin SDK
