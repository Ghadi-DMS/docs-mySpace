---
read_when:
    - Chcesz uruchamiać OpenClaw z modelami chmurowymi lub lokalnymi przez Ollama
    - Potrzebujesz wskazówek dotyczących konfiguracji i ustawień Ollama
summary: Uruchamiaj OpenClaw z Ollama (modele chmurowe i lokalne)
title: Ollama
x-i18n:
    generated_at: "2026-04-12T23:32:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec796241b884ca16ec7077df4f3f1910e2850487bb3ea94f8fdb37c77e02b219
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama to lokalne środowisko uruchomieniowe LLM, które ułatwia uruchamianie modeli open source na Twojej maszynie. OpenClaw integruje się z natywnym API Ollama (`/api/chat`), obsługuje przesyłanie strumieniowe i wywoływanie narzędzi oraz może automatycznie wykrywać lokalne modele Ollama, gdy włączysz to przez `OLLAMA_API_KEY` (lub profil uwierzytelniania) i nie zdefiniujesz jawnego wpisu `models.providers.ollama`.

<Warning>
**Użytkownicy zdalnego Ollama**: Nie używaj adresu URL `/v1` zgodnego z OpenAI (`http://host:11434/v1`) z OpenClaw. Powoduje to awarie wywoływania narzędzi, a modele mogą zwracać surowy JSON narzędzi jako zwykły tekst. Zamiast tego używaj natywnego adresu URL API Ollama: `baseUrl: "http://host:11434"` (bez `/v1`).
</Warning>

## Pierwsze kroki

Wybierz preferowaną metodę konfiguracji i tryb.

<Tabs>
  <Tab title="Onboarding (zalecane)">
    **Najlepsze dla:** najszybszej ścieżki do działającej konfiguracji Ollama z automatycznym wykrywaniem modeli.

    <Steps>
      <Step title="Uruchom onboarding">
        ```bash
        openclaw onboard
        ```

        Wybierz **Ollama** z listy dostawców.
      </Step>
      <Step title="Wybierz tryb">
        - **Cloud + Local** — modele hostowane w chmurze i modele lokalne razem
        - **Local** — tylko modele lokalne

        Jeśli wybierzesz **Cloud + Local** i nie jesteś zalogowany w ollama.com, onboarding otworzy przepływ logowania w przeglądarce.
      </Step>
      <Step title="Wybierz model">
        Onboarding wykrywa dostępne modele i sugeruje ustawienia domyślne. Automatycznie pobiera wybrany model, jeśli nie jest dostępny lokalnie.
      </Step>
      <Step title="Sprawdź, czy model jest dostępny">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    ### Tryb nieinteraktywny

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --accept-risk
    ```

    Opcjonalnie podaj niestandardowy base URL lub model:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

  </Tab>

  <Tab title="Konfiguracja ręczna">
    **Najlepsze dla:** pełnej kontroli nad instalacją, pobieraniem modeli i konfiguracją.

    <Steps>
      <Step title="Zainstaluj Ollama">
        Pobierz z [ollama.com/download](https://ollama.com/download).
      </Step>
      <Step title="Pobierz model lokalny">
        ```bash
        ollama pull gemma4
        # lub
        ollama pull gpt-oss:20b
        # lub
        ollama pull llama3.3
        ```
      </Step>
      <Step title="Zaloguj się do modeli chmurowych (opcjonalnie)">
        Jeśli chcesz także modeli chmurowych:

        ```bash
        ollama signin
        ```
      </Step>
      <Step title="Włącz Ollama dla OpenClaw">
        Ustaw dowolną wartość klucza API (Ollama nie wymaga prawdziwego klucza):

        ```bash
        # Ustaw zmienną środowiskową
        export OLLAMA_API_KEY="ollama-local"

        # Albo skonfiguruj w pliku konfiguracyjnym
        openclaw config set models.providers.ollama.apiKey "ollama-local"
        ```
      </Step>
      <Step title="Sprawdź i ustaw model">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Albo ustaw model domyślny w konfiguracji:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Modele chmurowe

<Tabs>
  <Tab title="Cloud + Local">
    Modele chmurowe pozwalają uruchamiać modele hostowane w chmurze obok modeli lokalnych. Przykłady obejmują `kimi-k2.5:cloud`, `minimax-m2.7:cloud` i `glm-5.1:cloud` -- te modele **nie** wymagają lokalnego `ollama pull`.

    Wybierz tryb **Cloud + Local** podczas konfiguracji. Kreator sprawdza, czy jesteś zalogowany, i w razie potrzeby otwiera przepływ logowania w przeglądarce. Jeśli nie da się zweryfikować uwierzytelnienia, kreator wraca do domyślnych modeli lokalnych.

    Możesz też zalogować się bezpośrednio na [ollama.com/signin](https://ollama.com/signin).

    OpenClaw obecnie sugeruje następujące domyślne modele chmurowe: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`.

  </Tab>

  <Tab title="Tylko lokalne">
    W trybie tylko lokalnym OpenClaw wykrywa modele z lokalnej instancji Ollama. Logowanie do chmury nie jest potrzebne.

    OpenClaw obecnie sugeruje `gemma4` jako lokalny model domyślny.

  </Tab>
</Tabs>

## Wykrywanie modeli (dostawca niejawny)

Gdy ustawisz `OLLAMA_API_KEY` (lub profil uwierzytelniania) i **nie** zdefiniujesz `models.providers.ollama`, OpenClaw wykrywa modele z lokalnej instancji Ollama pod adresem `http://127.0.0.1:11434`.

| Zachowanie           | Szczegóły                                                                                                                                                            |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zapytanie do katalogu | Wysyła zapytania do `/api/tags`                                                                                                                                     |
| Wykrywanie możliwości | Używa możliwie najlepszych odczytów `/api/show`, aby odczytać `contextWindow` i wykryć możliwości (w tym vision)                                                 |
| Modele vision        | Modele z możliwością `vision` zgłoszoną przez `/api/show` są oznaczane jako obsługujące obrazy (`input: ["text", "image"]`), więc OpenClaw automatycznie wstrzykuje obrazy do promptu |
| Wykrywanie rozumowania | Oznacza `reasoning` za pomocą heurystyki nazwy modelu (`r1`, `reasoning`, `think`)                                                                                |
| Limity tokenów       | Ustawia `maxTokens` na domyślny limit maksymalnej liczby tokenów Ollama używany przez OpenClaw                                                                     |
| Koszty               | Ustawia wszystkie koszty na `0`                                                                                                                                      |

Pozwala to uniknąć ręcznych wpisów modeli, a jednocześnie utrzymać katalog zgodny z lokalną instancją Ollama.

```bash
# Zobacz, jakie modele są dostępne
ollama list
openclaw models list
```

Aby dodać nowy model, po prostu pobierz go przez Ollama:

```bash
ollama pull mistral
```

Nowy model zostanie automatycznie wykryty i będzie dostępny do użycia.

<Note>
Jeśli jawnie ustawisz `models.providers.ollama`, automatyczne wykrywanie zostanie pominięte i modele trzeba będzie definiować ręcznie. Zobacz sekcję jawnej konfiguracji poniżej.
</Note>

## Konfiguracja

<Tabs>
  <Tab title="Podstawowa (wykrywanie niejawne)">
    Najprostszy sposób włączenia Ollama to zmienna środowiskowa:

    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Jeśli ustawiono `OLLAMA_API_KEY`, możesz pominąć `apiKey` we wpisie dostawcy, a OpenClaw uzupełni je na potrzeby kontroli dostępności.
    </Tip>

  </Tab>

  <Tab title="Jawna (modele ręczne)">
    Użyj jawnej konfiguracji, gdy Ollama działa na innym hoście/porcie, chcesz wymusić określone okna kontekstu lub listy modeli albo chcesz mieć w pełni ręczne definicje modeli.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            apiKey: "ollama-local",
            api: "ollama",
            models: [
              {
                id: "gpt-oss:20b",
                name: "GPT-OSS 20B",
                reasoning: false,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 8192,
                maxTokens: 8192 * 10
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="Niestandardowy base URL">
    Jeśli Ollama działa na innym hoście lub porcie (jawna konfiguracja wyłącza automatyczne wykrywanie, więc modele trzeba definiować ręcznie):

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Bez /v1 — użyj natywnego adresu URL API Ollama
            api: "ollama", // Ustaw jawnie, aby zagwarantować natywne zachowanie wywoływania narzędzi
          },
        },
      },
    }
    ```

    <Warning>
    Nie dodawaj `/v1` do adresu URL. Ścieżka `/v1` używa trybu zgodnego z OpenAI, w którym wywoływanie narzędzi nie jest niezawodne. Używaj bazowego adresu URL Ollama bez sufiksu ścieżki.
    </Warning>

  </Tab>
</Tabs>

### Wybór modelu

Po skonfigurowaniu wszystkie Twoje modele Ollama są dostępne:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Ollama Web Search

OpenClaw obsługuje **Ollama Web Search** jako dołączonego dostawcę `web_search`.

| Właściwość | Szczegóły                                                                                                            |
| ---------- | -------------------------------------------------------------------------------------------------------------------- |
| Host       | Używa skonfigurowanego hosta Ollama (`models.providers.ollama.baseUrl`, jeśli ustawiono, w przeciwnym razie `http://127.0.0.1:11434`) |
| Uwierzytelnianie | Bez klucza                                                                                                  |
| Wymaganie  | Ollama musi działać i być zalogowane przez `ollama signin`                                                          |

Wybierz **Ollama Web Search** podczas `openclaw onboard` lub `openclaw configure --section web`, albo ustaw:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

<Note>
Pełne szczegóły konfiguracji i działania znajdziesz w [Ollama Web Search](/pl/tools/ollama-search).
</Note>

## Konfiguracja zaawansowana

<AccordionGroup>
  <Accordion title="Starszy tryb zgodny z OpenAI">
    <Warning>
    **Wywoływanie narzędzi nie jest niezawodne w trybie zgodnym z OpenAI.** Używaj tego trybu tylko wtedy, gdy potrzebujesz formatu OpenAI dla serwera proxy i nie zależy Ci na natywnym zachowaniu wywoływania narzędzi.
    </Warning>

    Jeśli zamiast tego musisz używać endpointu zgodnego z OpenAI (na przykład za serwerem proxy obsługującym tylko format OpenAI), ustaw jawnie `api: "openai-completions"`:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // domyślnie: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    Ten tryb może nie obsługiwać jednocześnie przesyłania strumieniowego i wywoływania narzędzi. Może być konieczne wyłączenie przesyłania strumieniowego za pomocą `params: { streaming: false }` w konfiguracji modelu.

    Gdy `api: "openai-completions"` jest używane z Ollama, OpenClaw domyślnie wstrzykuje `options.num_ctx`, aby Ollama nie przechodziło po cichu do okna kontekstu 4096. Jeśli Twój serwer proxy/upstream odrzuca nieznane pola `options`, wyłącz to zachowanie:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Okna kontekstu">
    W przypadku modeli wykrytych automatycznie OpenClaw używa okna kontekstu zgłoszonego przez Ollama, jeśli jest dostępne, w przeciwnym razie wraca do domyślnego okna kontekstu Ollama używanego przez OpenClaw.

    Możesz nadpisać `contextWindow` i `maxTokens` w jawnej konfiguracji dostawcy:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
              }
            ]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Modele rozumujące">
    OpenClaw domyślnie traktuje modele o nazwach takich jak `deepseek-r1`, `reasoning` lub `think` jako zdolne do rozumowania.

    ```bash
    ollama pull deepseek-r1:32b
    ```

    Nie jest potrzebna żadna dodatkowa konfiguracja -- OpenClaw oznacza je automatycznie.

  </Accordion>

  <Accordion title="Koszty modeli">
    Ollama jest darmowe i działa lokalnie, więc wszystkie koszty modeli są ustawione na 0 USD. Dotyczy to zarówno modeli wykrytych automatycznie, jak i zdefiniowanych ręcznie.
  </Accordion>

  <Accordion title="Osadzania pamięci">
    Dołączony Plugin Ollama rejestruje dostawcę osadzań pamięci dla
    [wyszukiwania pamięci](/pl/concepts/memory). Używa skonfigurowanego base URL
    oraz klucza API Ollama.

    | Właściwość    | Wartość             |
    | ------------- | ------------------- |
    | Model domyślny | `nomic-embed-text` |
    | Auto-pull     | Tak — model osadzania jest pobierany automatycznie, jeśli nie jest obecny lokalnie |

    Aby wybrać Ollama jako dostawcę osadzań dla wyszukiwania pamięci:

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: { provider: "ollama" },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Konfiguracja przesyłania strumieniowego">
    Integracja Ollama w OpenClaw domyślnie używa **natywnego API Ollama** (`/api/chat`), które w pełni obsługuje jednocześnie przesyłanie strumieniowe i wywoływanie narzędzi. Nie jest potrzebna żadna specjalna konfiguracja.

    <Tip>
    Jeśli musisz używać endpointu zgodnego z OpenAI, zobacz sekcję „Starszy tryb zgodny z OpenAI” powyżej. W tym trybie przesyłanie strumieniowe i wywoływanie narzędzi mogą nie działać jednocześnie.
    </Tip>

  </Accordion>
</AccordionGroup>

## Rozwiązywanie problemów

<AccordionGroup>
  <Accordion title="Nie wykryto Ollama">
    Upewnij się, że Ollama działa, że ustawiono `OLLAMA_API_KEY` (lub profil uwierzytelniania) oraz że **nie** zdefiniowano jawnego wpisu `models.providers.ollama`:

    ```bash
    ollama serve
    ```

    Sprawdź, czy API jest dostępne:

    ```bash
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Brak dostępnych modeli">
    Jeśli Twojego modelu nie ma na liście, pobierz go lokalnie albo zdefiniuj jawnie w `models.providers.ollama`.

    ```bash
    ollama list  # Zobacz, co jest zainstalowane
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Albo inny model
    ```

  </Accordion>

  <Accordion title="Połączenie odrzucone">
    Sprawdź, czy Ollama działa na właściwym porcie:

    ```bash
    # Sprawdź, czy Ollama działa
    ps aux | grep ollama

    # Albo uruchom Ollama ponownie
    ollama serve
    ```

  </Accordion>
</AccordionGroup>

<Note>
Więcej pomocy: [Rozwiązywanie problemów](/pl/help/troubleshooting) i [FAQ](/pl/help/faq).
</Note>

## Powiązane

<CardGroup cols={2}>
  <Card title="Dostawcy modeli" href="/pl/concepts/model-providers" icon="layers">
    Omówienie wszystkich dostawców, odwołań do modeli i zachowania failover.
  </Card>
  <Card title="Wybór modelu" href="/pl/concepts/models" icon="brain">
    Jak wybierać i konfigurować modele.
  </Card>
  <Card title="Ollama Web Search" href="/pl/tools/ollama-search" icon="magnifying-glass">
    Pełne szczegóły konfiguracji i działania dla wyszukiwania w sieci opartego na Ollama.
  </Card>
  <Card title="Configuration" href="/pl/gateway/configuration" icon="gear">
    Pełna dokumentacja konfiguracji.
  </Card>
</CardGroup>
