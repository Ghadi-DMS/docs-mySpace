---
read_when:
    - Chcesz uruchomić OpenClaw z lokalnym serwerem inferrs
    - Udostępniasz Gemma lub inny model przez inferrs
    - Potrzebujesz dokładnych flag zgodności OpenClaw dla inferrs
summary: Uruchamiaj OpenClaw przez inferrs (lokalny serwer zgodny z OpenAI)
title: inferrs
x-i18n:
    generated_at: "2026-04-09T01:29:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 03b9d5a9935c75fd369068bacb7807a5308cd0bd74303b664227fb664c3a2098
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs) może udostępniać lokalne modele za
interfejsem API `/v1` zgodnym z OpenAI. OpenClaw współpracuje z `inferrs` przez ogólną
ścieżkę `openai-completions`.

`inferrs` najlepiej obecnie traktować jako niestandardowy, self-hosted
backend zgodny z OpenAI, a nie dedykowaną wtyczkę dostawcy OpenClaw.

## Szybki start

1. Uruchom `inferrs` z modelem.

Przykład:

```bash
inferrs serve google/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. Sprawdź, czy serwer jest osiągalny.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. Dodaj jawny wpis dostawcy OpenClaw i skieruj na niego swój domyślny model.

## Pełny przykład konfiguracji

Ten przykład używa Gemma 4 na lokalnym serwerze `inferrs`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## Dlaczego `requiresStringContent` ma znaczenie

Niektóre trasy Chat Completions w `inferrs` akceptują tylko ciąg znaków w
`messages[].content`, a nie ustrukturyzowane tablice części treści.

Jeśli uruchomienia OpenClaw kończą się błędem takim jak:

```text
messages[1].content: invalid type: sequence, expected a string
```

ustaw:

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw spłaszczy części treści zawierające wyłącznie tekst do zwykłych ciągów
przed wysłaniem żądania.

## Zastrzeżenie dotyczące Gemma i schematu narzędzi

Niektóre obecne kombinacje `inferrs` + Gemma akceptują małe, bezpośrednie
żądania do `/v1/chat/completions`, ale nadal kończą się błędem przy pełnych turnach
runtime agenta OpenClaw.

Jeśli tak się dzieje, najpierw spróbuj tego:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

To wyłącza powierzchnię schematu narzędzi OpenClaw dla modelu i może zmniejszyć nacisk promptu
na bardziej restrykcyjnych lokalnych backendach.

Jeśli małe bezpośrednie żądania nadal działają, ale zwykłe turny agenta OpenClaw dalej
powodują awarie wewnątrz `inferrs`, pozostały problem zwykle dotyczy zachowania
upstream modelu/serwera, a nie warstwy transportu OpenClaw.

## Ręczny test smoke

Po skonfigurowaniu przetestuj obie warstwy:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/google/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

Jeśli pierwsze polecenie działa, ale drugie kończy się błędem, skorzystaj z poniższych
wskazówek rozwiązywania problemów.

## Rozwiązywanie problemów

- `curl /v1/models` kończy się błędem: `inferrs` nie działa, nie jest osiągalny lub nie
  jest zbindowany do oczekiwanego hosta/portu.
- `messages[].content ... expected a string`: ustaw
  `compat.requiresStringContent: true`.
- Bezpośrednie małe wywołania `/v1/chat/completions` przechodzą, ale `openclaw infer model run`
  kończy się błędem: spróbuj `compat.supportsTools: false`.
- OpenClaw nie dostaje już błędów schematu, ale `inferrs` nadal ulega awarii przy większych
  turnach agenta: potraktuj to jako ograniczenie `inferrs` lub modelu upstream i zmniejsz
  nacisk promptu albo zmień lokalny backend/model.

## Zachowanie w stylu proxy

`inferrs` jest traktowany jako backend `/v1` zgodny z OpenAI w stylu proxy, a nie
natywny endpoint OpenAI.

- natywne kształtowanie żądań wyłącznie dla OpenAI nie ma tu zastosowania
- brak `service_tier`, brak `store` dla Responses, brak wskazówek cache promptów i brak
  kształtowania payload zgodności reasoning OpenAI
- ukryte nagłówki atrybucji OpenClaw (`originator`, `version`, `User-Agent`)
  nie są wstrzykiwane dla niestandardowych base URL `inferrs`

## Zobacz też

- [Local models](/pl/gateway/local-models)
- [Gateway troubleshooting](/pl/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Model providers](/pl/concepts/model-providers)
