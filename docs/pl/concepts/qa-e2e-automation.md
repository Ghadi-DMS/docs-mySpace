---
read_when:
    - Rozszerzanie qa-lab lub qa-channel
    - Dodawanie scenariuszy QA opartych na repozytorium
    - Tworzenie bardziej realistycznej automatyzacji QA wokół panelu Gateway
summary: Prywatny kształt automatyzacji QA dla qa-lab, qa-channel, scenariuszy seedowanych i raportów protokołu
title: Automatyzacja QA E2E
x-i18n:
    generated_at: "2026-04-12T23:28:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: b9fe27dc049823d5e3eb7ae1eac6aad21ed9e917425611fb1dbcb28ab9210d5e
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatyzacja QA E2E

Prywatny stos QA ma na celu testowanie OpenClaw w bardziej realistyczny,
kanałowy sposób niż pojedynczy test jednostkowy.

Obecne elementy:

- `extensions/qa-channel`: syntetyczny kanał wiadomości z powierzchniami DM, kanału, wątku,
  reakcji, edycji i usuwania.
- `extensions/qa-lab`: interfejs debuggera i magistrala QA do obserwowania transkryptu,
  wstrzykiwania wiadomości przychodzących i eksportowania raportu Markdown.
- `qa/`: zasoby seedowane oparte na repozytorium dla zadania startowego i bazowych scenariuszy QA.

Obecny przepływ pracy operatora QA to dwupanelowa witryna QA:

- Po lewej: panel Gateway (Control UI) z agentem.
- Po prawej: QA Lab, pokazujący transkrypt w stylu Slacka i plan scenariusza.

Uruchom za pomocą:

```bash
pnpm qa:lab:up
```

To buduje witrynę QA, uruchamia opartą na Dockerze ścieżkę gateway i udostępnia
stronę QA Lab, na której operator lub pętla automatyzacji może przydzielić agentowi
misję QA, obserwować rzeczywiste zachowanie kanału oraz zapisywać, co zadziałało,
co zawiodło lub co pozostało zablokowane.

Aby szybciej iterować nad interfejsem QA Lab bez każdorazowego przebudowywania obrazu Docker,
uruchom stos z bind-mountowanym bundle QA Lab:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` utrzymuje usługi Docker na wcześniej zbudowanym obrazie i bind-mountuje
`extensions/qa-lab/web/dist` do kontenera `qa-lab`. `qa:lab:watch`
przebudowuje ten bundle przy zmianach, a przeglądarka automatycznie odświeża się,
gdy zmieni się hash zasobu QA Lab.

Aby uruchomić ścieżkę smoke Matrix z rzeczywistym transportem, wykonaj:

```bash
pnpm openclaw qa matrix
```

Ta ścieżka provisionuje jednorazowy homeserver Tuwunel w Dockerze, rejestruje
tymczasowych użytkowników driver, SUT i observer, tworzy jeden prywatny pokój,
a następnie uruchamia prawdziwy Plugin Matrix wewnątrz potomnego procesu QA gateway. Ścieżka z żywym
transportem utrzymuje konfigurację potomną ograniczoną do testowanego transportu,
dzięki czemu Matrix działa bez `qa-channel` w konfiguracji potomnej.

Aby uruchomić ścieżkę smoke Telegram z rzeczywistym transportem, wykonaj:

```bash
pnpm openclaw qa telegram
```

Ta ścieżka kieruje ruch do jednej prawdziwej prywatnej grupy Telegram zamiast provisionować
jednorazowy serwer. Wymaga `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` oraz
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`, a także dwóch odrębnych botów w tej samej
prywatnej grupie. Bot SUT musi mieć nazwę użytkownika Telegram, a obserwacja bot-bot
działa najlepiej, gdy oba boty mają włączony tryb Bot-to-Bot Communication Mode
w `@BotFather`.

Ścieżki z żywym transportem współdzielą teraz jeden mniejszy kontrakt zamiast tego,
by każda wymyślała własny kształt listy scenariuszy:

`qa-channel` pozostaje szerokim syntetycznym zestawem zachowań produktu i nie jest częścią
macierzy pokrycia żywego transportu.

| Ścieżka  | Canary | Bramka wzmianek | Blokada allowlisty | Odpowiedź najwyższego poziomu | Wznowienie po restarcie | Dalszy ciąg wątku | Izolacja wątku | Obserwacja reakcji | Komenda pomocy |
| -------- | ------ | --------------- | ------------------ | ----------------------------- | ----------------------- | ----------------- | -------------- | ------------------ | -------------- |
| Matrix   | x      | x               | x                  | x                             | x                       | x                 | x              | x                  |                |
| Telegram | x      |                 |                    |                               |                         |                   |                |                    | x              |

Dzięki temu `qa-channel` pozostaje szerokim zestawem zachowań produktu, podczas gdy Matrix,
Telegram i przyszłe żywe transporty współdzielą jedną jawną listę kontrolną kontraktu transportu.

Aby uruchomić jednorazową ścieżkę Linux VM bez wprowadzania Dockera do ścieżki QA, wykonaj:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

To uruchamia świeżego gościa Multipass, instaluje zależności, buduje OpenClaw
wewnątrz gościa, uruchamia `qa suite`, a następnie kopiuje zwykły raport QA i
podsumowanie z powrotem do `.artifacts/qa-e2e/...` na hoście.
Ponownie wykorzystuje to samo zachowanie wyboru scenariuszy co `qa suite` na hoście.
Uruchomienia hosta i Multipass suite wykonują domyślnie wiele wybranych scenariuszy równolegle
z izolowanymi workerami gateway, maksymalnie do 64 workerów lub liczby wybranych
scenariuszy. Użyj `--concurrency <count>`, aby dostroić liczbę workerów, lub
`--concurrency 1` dla wykonania seryjnego.
Uruchomienia live przekazują obsługiwane wejścia uwierzytelniania QA, które są praktyczne dla
gościa: klucze dostawcy oparte na env, ścieżkę konfiguracji dostawcy QA live oraz
`CODEX_HOME`, gdy jest obecne. Zachowaj `--output-dir` pod katalogiem głównym repozytorium, aby gość
mógł zapisywać z powrotem przez zamontowany workspace.

## Seedy oparte na repozytorium

Zasoby seedowane znajdują się w `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Są one celowo przechowywane w git, aby plan QA był widoczny zarówno dla ludzi, jak i dla
agenta.

`qa-lab` powinien pozostać generycznym runnerem markdown. Każdy plik markdown scenariusza jest
źródłem prawdy dla jednego uruchomienia testu i powinien definiować:

- metadane scenariusza
- odwołania do dokumentacji i kodu
- opcjonalne wymagania Pluginów
- opcjonalną łatkę konfiguracji gateway
- wykonywalny `qa-flow`

Lista bazowa powinna pozostać wystarczająco szeroka, aby obejmować:

- czat DM i kanałowy
- zachowanie wątków
- cykl życia akcji wiadomości
- wywołania zwrotne Cron
- przywoływanie pamięci
- przełączanie modeli
- przekazanie do subagenta
- czytanie repozytorium i dokumentacji
- jedno małe zadanie build, takie jak Lobster Invaders

## Adaptery transportu

`qa-lab` posiada generyczny seam transportowy dla scenariuszy QA w markdown.
`qa-channel` jest pierwszym adapterem na tym seamie, ale docelowy projekt jest szerszy:
przyszłe rzeczywiste lub syntetyczne kanały powinny podłączać się do tego samego runnera suite
zamiast dodawać runner QA specyficzny dla transportu.

Na poziomie architektury podział jest następujący:

- `qa-lab` odpowiada za generyczne wykonywanie scenariuszy, współbieżność workerów, zapisywanie artefaktów i raportowanie.
- adapter transportu odpowiada za konfigurację gateway, gotowość, obserwację wejścia i wyjścia, akcje transportu oraz znormalizowany stan transportu.
- pliki markdown scenariuszy w `qa/scenarios/` definiują przebieg testu; `qa-lab` udostępnia wielokrotnego użytku powierzchnię runtime, która je wykonuje.

Wskazówki wdrożeniowe dla maintainerów dotyczące nowych adapterów kanałów znajdują się w
[Testing](/pl/help/testing#adding-a-channel-to-qa).

## Raportowanie

`qa-lab` eksportuje raport protokołu Markdown na podstawie obserwowanej osi czasu magistrali.
Raport powinien odpowiadać na pytania:

- Co zadziałało
- Co zawiodło
- Co pozostało zablokowane
- Jakie scenariusze uzupełniające warto dodać

Aby przeprowadzić kontrole charakteru i stylu, uruchom ten sam scenariusz na wielu żywych referencjach modeli
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

Komenda uruchamia lokalne potomne procesy QA gateway, a nie Docker. Scenariusze character eval
powinny ustawiać personę przez `SOUL.md`, a następnie wykonywać zwykłe tury użytkownika,
takie jak czat, pomoc dotycząca workspace i małe zadania na plikach. Kandydacki model
nie powinien być informowany, że jest oceniany. Komenda zachowuje każdy pełny
transkrypt, rejestruje podstawowe statystyki uruchomienia, a następnie prosi modele oceniające w trybie fast z
rozumowaniem `xhigh`, aby uszeregowały uruchomienia według naturalności, klimatu i humoru.
Użyj `--blind-judge-models` podczas porównywania dostawców: prompt oceniający nadal otrzymuje
każdy transkrypt i status uruchomienia, ale referencje kandydatów są zastępowane neutralnymi
etykietami, takimi jak `candidate-01`; raport mapuje rankingi z powrotem na prawdziwe referencje po
parsowaniu.
Uruchomienia kandydatów domyślnie używają poziomu myślenia `high`, z `xhigh` dla modeli OpenAI,
które go obsługują. Zastąp określonego kandydata inline za pomocą
`--model provider/model,thinking=<level>`. `--thinking <level>` nadal ustawia
globalny fallback, a starsza forma `--model-thinking <provider/model=level>` jest
zachowana dla kompatybilności.
Referencje kandydatów OpenAI domyślnie używają trybu fast, dzięki czemu wykorzystywane jest
przetwarzanie priorytetowe tam, gdzie dostawca je obsługuje. Dodaj inline `,fast`, `,no-fast` lub `,fast=false`,
gdy pojedynczy kandydat lub oceniający wymaga nadpisania. Przekaż `--fast` tylko wtedy, gdy chcesz
wymusić tryb fast dla każdego modelu kandydata. Czasy trwania kandydatów i oceniających są
rejestrowane w raporcie do analizy benchmarkowej, ale prompty oceniające wyraźnie mówią,
aby nie ustalać rankingu według szybkości.
Zarówno uruchomienia modeli kandydatów, jak i oceniających domyślnie używają współbieżności 16. Zmniejsz
`--concurrency` lub `--judge-concurrency`, gdy limity dostawcy lub obciążenie lokalnego gateway
sprawiają, że uruchomienie jest zbyt zaszumione.
Gdy nie zostanie przekazany żaden kandydat `--model`, character eval domyślnie używa
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` oraz
`google/gemini-3.1-pro-preview`, jeśli nie przekazano `--model`.
Gdy nie zostanie przekazany żaden `--judge-model`, oceniający domyślnie używają
`openai/gpt-5.4,thinking=xhigh,fast` oraz
`anthropic/claude-opus-4-6,thinking=high`.

## Powiązana dokumentacja

- [Testing](/pl/help/testing)
- [QA Channel](/pl/channels/qa-channel)
- [Dashboard](/web/dashboard)
