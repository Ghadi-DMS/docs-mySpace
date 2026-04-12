---
read_when:
    - Chcesz używać GitHub Copilot jako dostawcy modeli
    - Potrzebujesz przepływu `openclaw models auth login-github-copilot`
summary: Zaloguj się do GitHub Copilot z OpenClaw przy użyciu device flow
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-12T23:31:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 51fee006e7d4e78e37b0c29356b0090b132de727d99b603441767d3fb642140b
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

GitHub Copilot to asystent kodowania AI od GitHub. Zapewnia dostęp do modeli
Copilot dla Twojego konta i planu GitHub. OpenClaw może używać Copilot jako
dostawcy modeli na dwa różne sposoby.

## Dwa sposoby używania Copilot w OpenClaw

<Tabs>
  <Tab title="Wbudowany dostawca (github-copilot)">
    Użyj natywnego przepływu logowania przez urządzenie, aby uzyskać token GitHub, a następnie wymieniaj go na
    tokeny API Copilot podczas działania OpenClaw. To **domyślna** i najprostsza ścieżka,
    ponieważ nie wymaga VS Code.

    <Steps>
      <Step title="Uruchom polecenie logowania">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Zostaniesz poproszony o przejście pod URL i wpisanie jednorazowego kodu. Pozostaw
        terminal otwarty do czasu zakończenia procesu.
      </Step>
      <Step title="Ustaw model domyślny">
        ```bash
        openclaw models set github-copilot/gpt-4o
        ```

        Albo w konfiguracji:

        ```json5
        {
          agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Plugin Copilot Proxy (copilot-proxy)">
    Użyj rozszerzenia VS Code **Copilot Proxy** jako lokalnego mostu. OpenClaw komunikuje się z
    endpointem `/v1` serwera proxy i używa listy modeli skonfigurowanej tam przez Ciebie.

    <Note>
    Wybierz to, jeśli już uruchamiasz Copilot Proxy w VS Code lub potrzebujesz kierować ruch
    przez niego. Musisz włączyć Plugin i utrzymywać uruchomione rozszerzenie VS Code.
    </Note>

  </Tab>
</Tabs>

## Opcjonalne flagi

| Flaga          | Opis                                                  |
| -------------- | ----------------------------------------------------- |
| `--yes`        | Pomija monit o potwierdzenie                          |
| `--set-default` | Ustawia także zalecany domyślny model tego dostawcy  |

```bash
# Pomiń potwierdzenie
openclaw models auth login-github-copilot --yes

# Zaloguj się i ustaw model domyślny w jednym kroku
openclaw models auth login --provider github-copilot --method device --set-default
```

<AccordionGroup>
  <Accordion title="Wymagany interaktywny TTY">
    Przepływ logowania przez urządzenie wymaga interaktywnego TTY. Uruchom go bezpośrednio w
    terminalu, a nie w nieinteraktywnym skrypcie ani potoku CI.
  </Accordion>

  <Accordion title="Dostępność modeli zależy od Twojego planu">
    Dostępność modeli Copilot zależy od Twojego planu GitHub. Jeśli model
    zostanie odrzucony, spróbuj innego identyfikatora (na przykład `github-copilot/gpt-4.1`).
  </Accordion>

  <Accordion title="Wybór transportu">
    Identyfikatory modeli Claude automatycznie używają transportu Anthropic Messages. Modele GPT,
    serii o i Gemini zachowują transport OpenAI Responses. OpenClaw
    wybiera właściwy transport na podstawie odwołania do modelu.
  </Accordion>

  <Accordion title="Kolejność rozpoznawania zmiennych środowiskowych">
    OpenClaw rozpoznaje uwierzytelnianie Copilot ze zmiennych środowiskowych w następującej
    kolejności priorytetów:

    | Priorytet | Zmienna              | Uwagi                                 |
    | --------- | -------------------- | ------------------------------------- |
    | 1         | `COPILOT_GITHUB_TOKEN` | Najwyższy priorytet, specyficzna dla Copilot |
    | 2         | `GH_TOKEN`           | Token GitHub CLI (awaryjnie)          |
    | 3         | `GITHUB_TOKEN`       | Standardowy token GitHub (najniższy)  |

    Gdy ustawionych jest kilka zmiennych, OpenClaw używa tej o najwyższym priorytecie.
    Przepływ logowania przez urządzenie (`openclaw models auth login-github-copilot`) zapisuje
    swój token w magazynie profili uwierzytelniania i ma pierwszeństwo przed wszystkimi
    zmiennymi środowiskowymi.

  </Accordion>

  <Accordion title="Przechowywanie tokenu">
    Logowanie zapisuje token GitHub w magazynie profili uwierzytelniania i wymienia go
    na token API Copilot podczas działania OpenClaw. Nie musisz zarządzać
    tokenem ręcznie.
  </Accordion>
</AccordionGroup>

<Warning>
Wymaga interaktywnego TTY. Uruchom polecenie logowania bezpośrednio w terminalu, a nie
wewnątrz bezobsługowego skryptu ani zadania CI.
</Warning>

## Powiązane

<CardGroup cols={2}>
  <Card title="Wybór modelu" href="/pl/concepts/model-providers" icon="layers">
    Wybór dostawców, odwołań do modeli i zachowania failover.
  </Card>
  <Card title="OAuth i uwierzytelnianie" href="/pl/gateway/authentication" icon="key">
    Szczegóły uwierzytelniania i zasady ponownego użycia poświadczeń.
  </Card>
</CardGroup>
