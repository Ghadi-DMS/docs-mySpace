---
read_when:
    - Chcesz udostępniać modele z własnej maszyny z GPU
    - Konfigurujesz LM Studio lub proxy zgodne z OpenAI
    - Potrzebujesz najbezpieczniejszych wskazówek dotyczących modeli lokalnych
summary: Uruchamiaj OpenClaw na lokalnych LLM-ach (LM Studio, vLLM, LiteLLM, niestandardowe endpointy OpenAI)
title: Modele lokalne
x-i18n:
    generated_at: "2026-04-14T09:50:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1544c522357ba4b18dfa6d05ea8d60c7c6262281b53863d9aee7002464703ca7
    source_path: gateway/local-models.md
    workflow: 15
---

# Modele lokalne

Lokalnie da się to zrobić, ale OpenClaw oczekuje dużego kontekstu + silnych zabezpieczeń przed prompt injection. Małe karty obcinają kontekst i osłabiają bezpieczeństwo. Celuj wysoko: **≥2 w pełni wyposażone Mac Studio lub równoważny zestaw GPU (~30 tys. USD+)**. Pojedyncze GPU **24 GB** działa tylko przy lżejszych promptach i z większymi opóźnieniami. Używaj **największego / pełnowymiarowego wariantu modelu, jaki jesteś w stanie uruchomić**; agresywnie kwantyzowane lub „małe” checkpointy zwiększają ryzyko prompt injection (zobacz [Bezpieczeństwo](/pl/gateway/security)).

Jeśli chcesz najprostszej konfiguracji lokalnej, zacznij od [LM Studio](/pl/providers/lmstudio) lub [Ollama](/pl/providers/ollama) i `openclaw onboard`. Ta strona to praktyczny przewodnik po bardziej zaawansowanych stosach lokalnych i niestandardowych lokalnych serwerach zgodnych z OpenAI.

## Zalecane: LM Studio + duży model lokalny (Responses API)

Obecnie najlepszy lokalny stos. Załaduj duży model w LM Studio (na przykład pełnowymiarową wersję Qwen, DeepSeek lub Llama), włącz serwer lokalny (domyślnie `http://127.0.0.1:1234`), i użyj Responses API, aby oddzielić rozumowanie od końcowego tekstu.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Lista kontrolna konfiguracji**

- Zainstaluj LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- W LM Studio pobierz **największą dostępną wersję modelu** (unikaj wariantów „small” / mocno kwantyzowanych), uruchom serwer i potwierdź, że `http://127.0.0.1:1234/v1/models` go wyświetla.
- Zastąp `my-local-model` rzeczywistym identyfikatorem modelu widocznym w LM Studio.
- Utrzymuj model załadowany; zimne ładowanie zwiększa opóźnienie startu.
- Dostosuj `contextWindow` / `maxTokens`, jeśli Twoja wersja LM Studio różni się od tej tutaj.
- W przypadku WhatsApp trzymaj się Responses API, aby wysyłany był tylko końcowy tekst.

Zachowaj też konfigurację modeli hostowanych, nawet jeśli działasz lokalnie; użyj `models.mode: "merge"`, aby fallbacki pozostały dostępne.

### Konfiguracja hybrydowa: hostowany model główny, lokalny fallback

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Podejście local-first z hostowaną siatką bezpieczeństwa

Zamień miejscami kolejność modelu głównego i fallbacków; zachowaj ten sam blok providerów oraz `models.mode: "merge"`, aby móc przełączyć się na Sonnet lub Opus, gdy lokalna maszyna będzie niedostępna.

### Hosting regionalny / trasowanie danych

- Hostowane warianty MiniMax / Kimi / GLM są też dostępne w OpenRouter z endpointami przypiętymi do regionu (np. hostowanymi w USA). Wybierz tam wariant regionalny, aby utrzymać ruch w wybranej jurysdykcji, jednocześnie nadal używając `models.mode: "merge"` dla fallbacków Anthropic / OpenAI.
- Podejście wyłącznie lokalne pozostaje najlepszą ścieżką prywatności; hostowane trasowanie regionalne to rozwiązanie pośrednie, gdy potrzebujesz funkcji dostawcy, ale chcesz zachować kontrolę nad przepływem danych.

## Inne lokalne proxy zgodne z OpenAI

vLLM, LiteLLM, OAI-proxy lub niestandardowe Gateway działają, jeśli udostępniają endpoint `/v1` w stylu OpenAI. Zastąp powyższy blok providera swoim endpointem i identyfikatorem modelu:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Zachowaj `models.mode: "merge"`, aby hostowane modele nadal były dostępne jako fallbacki.

Uwaga dotycząca działania backendów lokalnych / proxowanych `/v1`:

- OpenClaw traktuje je jako trasy zgodne z OpenAI w stylu proxy, a nie natywne endpointy OpenAI
- natywne formatowanie żądań przeznaczone wyłącznie dla OpenAI nie ma tu zastosowania: brak
  `service_tier`, brak `store` w Responses, brak formatowania payloadu zgodności rozumowania OpenAI
  i brak wskazówek dotyczących pamięci podręcznej promptów
- ukryte nagłówki atrybucji OpenClaw (`originator`, `version`, `User-Agent`)
  nie są wstrzykiwane do tych niestandardowych URL-i proxy

Uwagi dotyczące zgodności z bardziej restrykcyjnymi backendami zgodnymi z OpenAI:

- Niektóre serwery akceptują tylko ciąg znaków w `messages[].content` dla Chat Completions, a nie
  ustrukturyzowane tablice części treści. Ustaw
  `models.providers.<provider>.models[].compat.requiresStringContent: true` dla
  takich endpointów.
- Niektóre mniejsze lub bardziej restrykcyjne lokalne backendy są niestabilne przy pełnym
  kształcie promptu środowiska uruchomieniowego agenta OpenClaw, zwłaszcza gdy dołączone są schematy narzędzi. Jeśli
  backend działa dla małych bezpośrednich wywołań `/v1/chat/completions`, ale zawodzi przy zwykłych
  turach agenta OpenClaw, najpierw spróbuj
  `models.providers.<provider>.models[].compat.supportsTools: false`.
- Jeśli backend nadal zawodzi tylko przy większych uruchomieniach OpenClaw, pozostały problem
  zwykle leży po stronie pojemności modelu / serwera albo błędu backendu, a nie warstwy
  transportowej OpenClaw.

## Rozwiązywanie problemów

- Gateway może połączyć się z proxy? `curl http://127.0.0.1:1234/v1/models`.
- Model LM Studio został wyładowany? Załaduj go ponownie; zimny start to częsta przyczyna „zawieszania się”.
- OpenClaw ostrzega, gdy wykryte okno kontekstu jest mniejsze niż **32k**, i blokuje działanie poniżej **16k**. Jeśli trafisz na ten preflight, zwiększ limit kontekstu serwera / modelu albo wybierz większy model.
- Błędy kontekstu? Zmniejsz `contextWindow` albo zwiększ limit serwera.
- Serwer zgodny z OpenAI zwraca `messages[].content ... expected a string`?
  Dodaj `compat.requiresStringContent: true` do wpisu tego modelu.
- Bezpośrednie małe wywołania `/v1/chat/completions` działają, ale `openclaw infer model run`
  zawodzi na Gemma lub innym modelu lokalnym? Najpierw wyłącz schematy narzędzi przez
  `compat.supportsTools: false`, a potem przetestuj ponownie. Jeśli serwer nadal zawiesza się tylko
  przy większych promptach OpenClaw, traktuj to jako ograniczenie modelu / serwera po stronie upstream.
- Bezpieczeństwo: modele lokalne pomijają filtry po stronie dostawcy; ograniczaj zakres agentów i pozostaw Compaction włączone, aby zmniejszyć skutki prompt injection.
