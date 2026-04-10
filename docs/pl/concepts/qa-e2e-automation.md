---
read_when:
    - Rozszerzanie qa-lab lub qa-channel
    - Dodawanie scenariuszy QA opartych na repozytorium
    - Tworzenie bardziej realistycznej automatyzacji QA wokół panelu Gateway
summary: Prywatny kształt automatyzacji QA dla qa-lab, qa-channel, scenariuszy inicjalizowanych seedem i raportów protokołu
title: Automatyzacja QA E2E
x-i18n:
    generated_at: "2026-04-10T09:44:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 357d6698304ff7a8c4aa8a7be97f684d50f72b524740050aa761ac0ee68266de
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatyzacja QA E2E

Prywatny stos QA ma na celu testowanie OpenClaw w sposób bardziej realistyczny,
ukształtowany przez kanały, niż może to zrobić pojedynczy test jednostkowy.

Obecne elementy:

- `extensions/qa-channel`: syntetyczny kanał wiadomości z powierzchniami DM, kanału, wątku,
  reakcji, edycji i usuwania.
- `extensions/qa-lab`: interfejs debuggera i magistrala QA do obserwowania transkryptu,
  wstrzykiwania wiadomości przychodzących i eksportowania raportu Markdown.
- `qa/`: zasoby seed oparte na repozytorium dla zadania startowego i bazowych scenariuszy QA.

Obecny przepływ pracy operatora QA to dwupanelowa witryna QA:

- Po lewej: panel Gateway (Control UI) z agentem.
- Po prawej: QA Lab, pokazujący transkrypt w stylu Slacka i plan scenariusza.

Uruchom za pomocą:

```bash
pnpm qa:lab:up
```

To buduje witrynę QA, uruchamia ścieżkę gateway opartą na Dockerze i udostępnia
stronę QA Lab, na której operator lub pętla automatyzacji może przydzielić
agentowi misję QA, obserwować rzeczywiste zachowanie kanału oraz rejestrować,
co zadziałało, co zawiodło lub co pozostało zablokowane.

Aby szybciej iterować nad interfejsem QA Lab bez przebudowywania obrazu Dockera za każdym razem,
uruchom stos z bind-montowanym bundelem QA Lab:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` utrzymuje usługi Dockera na wcześniej zbudowanym obrazie i bind-montuje
`extensions/qa-lab/web/dist` do kontenera `qa-lab`. `qa:lab:watch`
przebudowuje ten bundle przy zmianach, a przeglądarka automatycznie przeładowuje się,
gdy zmienia się hash zasobu QA Lab.

Aby uruchomić jednorazową ścieżkę Linux VM bez włączania Dockera do ścieżki QA, użyj:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

To uruchamia świeżego gościa Multipass, instaluje zależności, buduje OpenClaw
wewnątrz gościa, uruchamia `qa suite`, a następnie kopiuje zwykły raport QA i
podsumowanie z powrotem do `.artifacts/qa-e2e/...` na hoście.
Wykorzystuje to samo zachowanie wyboru scenariusza co `qa suite` na hoście.
Uruchomienia live przekazują obsługiwane wejścia uwierzytelniania QA, które są praktyczne dla
gościa: klucze dostawców oparte na env, ścieżkę konfiguracji dostawcy live QA oraz
`CODEX_HOME`, gdy jest obecne. Utrzymuj `--output-dir` w katalogu repozytorium, aby gość
mógł zapisywać z powrotem przez zamontowany workspace.

## Seedy oparte na repozytorium

Zasoby seed znajdują się w `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Są one celowo przechowywane w git, aby plan QA był widoczny zarówno dla ludzi, jak i
agenta. Lista bazowa powinna pozostać wystarczająco szeroka, aby obejmować:

- czat DM i kanałowy
- zachowanie wątków
- cykl życia akcji na wiadomościach
- wywołania zwrotne cron
- przywoływanie pamięci
- przełączanie modeli
- przekazanie do subagenta
- odczyt repozytorium i dokumentacji
- jedno małe zadanie build, takie jak Lobster Invaders

## Raportowanie

`qa-lab` eksportuje raport protokołu w Markdown na podstawie obserwowanej osi czasu magistrali.
Raport powinien odpowiadać na pytania:

- Co zadziałało
- Co zawiodło
- Co pozostało zablokowane
- Jakie scenariusze uzupełniające warto dodać

W przypadku kontroli charakteru i stylu uruchom ten sam scenariusz dla wielu live refów modeli
i zapisz oceniony raport Markdown:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

To polecenie uruchamia lokalne podrzędne procesy gateway QA, a nie Dockera. Scenariusze
oceny charakteru powinny ustawiać personę przez `SOUL.md`, a następnie uruchamiać zwykłe
tury użytkownika, takie jak czat, pomoc dotycząca workspace i małe zadania plikowe. Kandydacki model
nie powinien być informowany, że jest oceniany. Polecenie zachowuje każdy pełny
transkrypt, rejestruje podstawowe statystyki przebiegu, a następnie prosi modele sędziujące w trybie fast z
rozumowaniem `xhigh`, aby uszeregowały przebiegi według naturalności, klimatu i humoru.
Użyj `--blind-judge-models` podczas porównywania dostawców: prompt sędziego nadal otrzymuje
każdy transkrypt i status przebiegu, ale refy kandydatów są zastępowane neutralnymi
etykietami, takimi jak `candidate-01`; raport mapuje rankingi z powrotem na rzeczywiste refy po
parsowaniu.
Przebiegi kandydatów domyślnie używają poziomu rozumowania `high`, z `xhigh` dla modeli OpenAI,
które to obsługują. Zastąp ustawienie dla konkretnego kandydata inline za pomocą
`--model provider/model,thinking=<level>`. `--thinking <level>` nadal ustawia
globalne ustawienie zapasowe, a starsza forma `--model-thinking <provider/model=level>` jest
zachowana dla zgodności.
Refy kandydatów OpenAI domyślnie używają trybu fast, aby wykorzystywane było przetwarzanie priorytetowe tam,
gdzie dostawca to obsługuje. Dodaj inline `,fast`, `,no-fast` lub `,fast=false`, gdy
pojedynczy kandydat lub sędzia wymaga nadpisania. Przekaż `--fast` tylko wtedy, gdy chcesz
wymusić tryb fast dla każdego modelu kandydującego. Czasy trwania kandydatów i sędziów są
rejestrowane w raporcie do analizy porównawczej, ale prompty sędziów wyraźnie mówią,
aby nie tworzyć rankingu według szybkości.
Zarówno przebiegi modeli kandydatów, jak i sędziów domyślnie używają współbieżności 16. Zmniejsz
`--concurrency` lub `--judge-concurrency`, gdy limity dostawcy lub obciążenie lokalnego gateway
sprawiają, że przebieg staje się zbyt zaszumiony.
Gdy nie zostanie przekazany żaden kandydacki `--model`, character eval domyślnie używa
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` oraz
`google/gemini-3.1-pro-preview`, gdy nie zostanie przekazany `--model`.
Gdy nie zostanie przekazany `--judge-model`, sędziowie domyślnie używają
`openai/gpt-5.4,thinking=xhigh,fast` oraz
`anthropic/claude-opus-4-6,thinking=high`.

## Powiązana dokumentacja

- [Testowanie](/pl/help/testing)
- [QA Channel](/pl/channels/qa-channel)
- [Panel](/web/dashboard)
