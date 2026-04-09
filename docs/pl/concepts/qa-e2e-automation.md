---
read_when:
    - Rozszerzanie qa-lab lub qa-channel
    - Dodawanie scenariuszy QA opartych na repozytorium
    - Tworzenie bardziej realistycznej automatyzacji QA wokół panelu Gateway
summary: Prywatna struktura automatyzacji QA dla qa-lab, qa-channel, scenariuszy seed i raportów protokołu
title: Automatyzacja QA E2E
x-i18n:
    generated_at: "2026-04-09T01:27:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: c922607d67e0f3a2489ac82bc9f510f7294ced039c1014c15b676d826441d833
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatyzacja QA E2E

Prywatny stos QA ma na celu testowanie OpenClaw w bardziej realistyczny,
zbliżony do kanałów sposób, niż może to zapewnić pojedynczy test jednostkowy.

Obecne elementy:

- `extensions/qa-channel`: syntetyczny kanał wiadomości z obsługą DM, kanałów, wątków,
  reakcji, edycji i usuwania.
- `extensions/qa-lab`: interfejs debuggera i magistrala QA do obserwowania transkryptu,
  wstrzykiwania wiadomości przychodzących i eksportowania raportu Markdown.
- `qa/`: zasoby seed oparte na repozytorium dla zadania startowego i bazowych
  scenariuszy QA.

Obecny przepływ pracy operatora QA to dwupanelowa witryna QA:

- Lewy panel: panel Gateway (Control UI) z agentem.
- Prawy panel: QA Lab, pokazujący transkrypt w stylu Slacka i plan scenariusza.

Uruchom za pomocą:

```bash
pnpm qa:lab:up
```

To buduje witrynę QA, uruchamia ścieżkę Gateway opartą na Dockerze i udostępnia
stronę QA Lab, na której operator lub pętla automatyzacji może przekazać agentowi
misję QA, obserwować rzeczywiste zachowanie kanału oraz zapisywać, co działało,
co zawiodło lub co pozostało zablokowane.

Aby szybciej iterować nad interfejsem QA Lab bez przebudowywania obrazu Dockera za każdym razem,
uruchom stos z bind-mountowanym bundlem QA Lab:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` utrzymuje usługi Dockera na wcześniej zbudowanym obrazie i bind-mountuje
`extensions/qa-lab/web/dist` do kontenera `qa-lab`. `qa:lab:watch`
przebudowuje ten bundle po zmianach, a przeglądarka automatycznie przeładowuje się,
gdy hash zasobu QA Lab ulegnie zmianie.

## Seedy oparte na repozytorium

Zasoby seed znajdują się w `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Są one celowo przechowywane w git, aby plan QA był widoczny zarówno dla ludzi, jak i dla
agenta. Lista bazowa powinna pozostać na tyle szeroka, aby obejmować:

- czat w DM i na kanałach
- zachowanie wątków
- cykl życia akcji na wiadomościach
- wywołania cron
- przywoływanie pamięci
- przełączanie modeli
- przekazanie do subagenta
- odczytywanie repozytorium i dokumentacji
- jedno małe zadanie build, takie jak Lobster Invaders

## Raportowanie

`qa-lab` eksportuje raport protokołu w Markdown na podstawie obserwowanej osi czasu magistrali.
Raport powinien odpowiadać na pytania:

- Co działało
- Co zawiodło
- Co pozostało zablokowane
- Jakie scenariusze uzupełniające warto dodać

W przypadku kontroli charakteru i stylu uruchom ten sam scenariusz dla wielu aktywnych
referencji modeli i zapisz oceniany raport Markdown:

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

Polecenie uruchamia lokalne podrzędne procesy QA Gateway, a nie Docker. Scenariusze
oceny charakteru powinny ustawiać personę przez `SOUL.md`, a następnie wykonywać zwykłe
tury użytkownika, takie jak czat, pomoc dotyczącą workspace i małe zadania na plikach. Modelowi
kandydującemu nie należy mówić, że jest oceniany. Polecenie zachowuje każdy pełny
transkrypt, rejestruje podstawowe statystyki uruchomienia, a następnie prosi modele oceniające
w trybie fast z rozumowaniem `xhigh` o uszeregowanie uruchomień według naturalności, klimatu i humoru.
Użyj `--blind-judge-models` podczas porównywania providerów: prompt oceniający nadal otrzymuje
każdy transkrypt i status uruchomienia, ale referencje kandydatów są zastępowane neutralnymi
etykietami, takimi jak `candidate-01`; raport mapuje rankingi z powrotem na rzeczywiste referencje po
parsowaniu.
Uruchomienia kandydatów domyślnie używają poziomu myślenia `high`, a dla modeli OpenAI,
które go obsługują, `xhigh`. Zastąp ustawienie dla konkretnego kandydata inline za pomocą
`--model provider/model,thinking=<level>`. `--thinking <level>` nadal ustawia
globalne ustawienie zapasowe, a starsza forma `--model-thinking <provider/model=level>` jest
zachowana dla zgodności.
Referencje kandydatów OpenAI domyślnie używają trybu fast, aby tam, gdzie provider to obsługuje,
korzystać z przetwarzania priorytetowego. Dodaj inline `,fast`, `,no-fast` lub `,fast=false`, gdy
pojedynczy kandydat lub sędzia wymaga nadpisania. Przekaż `--fast` tylko wtedy, gdy chcesz
wymusić tryb fast dla każdego modelu kandydującego. Czasy trwania dla kandydatów i modeli
oceniających są zapisywane w raporcie do analizy benchmarków, ale prompty oceniające wyraźnie
mówią, aby nie tworzyć rankingu na podstawie szybkości.
Zarówno uruchomienia modeli kandydujących, jak i oceniających domyślnie używają współbieżności 16. Zmniejsz
`--concurrency` lub `--judge-concurrency`, gdy limity providera lub obciążenie lokalnego Gateway
powodują, że uruchomienie staje się zbyt zaszumione.
Gdy nie zostanie przekazany żaden kandydat `--model`, ocena charakteru domyślnie używa
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` oraz
`google/gemini-3.1-pro-preview`, gdy nie zostanie przekazany parametr `--model`.
Gdy nie zostanie przekazany żaden `--judge-model`, modele oceniające domyślnie używają
`openai/gpt-5.4,thinking=xhigh,fast` oraz
`anthropic/claude-opus-4-6,thinking=high`.

## Powiązana dokumentacja

- [Testing](/pl/help/testing)
- [QA Channel](/pl/channels/qa-channel)
- [Dashboard](/web/dashboard)
