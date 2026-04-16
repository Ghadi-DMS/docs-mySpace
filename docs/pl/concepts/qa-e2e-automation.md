---
read_when:
    - Rozszerzanie qa-lab lub qa-channel
    - Dodawanie scenariuszy QA opartych na repozytorium
    - Tworzenie bardziej realistycznej automatyzacji QA wokół panelu Gateway
summary: Prywatny kształt automatyzacji QA dla qa-lab, qa-channel, scenariuszy seedowanych i raportów protokołu
title: Automatyzacja QA E2E
x-i18n:
    generated_at: "2026-04-16T21:51:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7deefda1c90a0d2e21e2155ffd8b585fb999e7416bdbaf0ff57eb33ccc063afc
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatyzacja QA E2E

Prywatny stos QA ma na celu testowanie OpenClaw w bardziej realistyczny,
zorientowany na kanały sposób niż pojedynczy test jednostkowy.

Obecne elementy:

- `extensions/qa-channel`: syntetyczny kanał wiadomości z powierzchniami DM, kanału, wątku,
  reakcji, edycji i usuwania.
- `extensions/qa-lab`: interfejs debugowania i magistrala QA do obserwowania transkryptu,
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
stronę QA Lab, gdzie operator lub pętla automatyzacji może dać agentowi misję QA,
obserwować rzeczywiste zachowanie kanału i zapisywać, co zadziałało, co się nie
powiodło lub co pozostało zablokowane.

Aby szybciej iterować nad interfejsem QA Lab bez przebudowywania obrazu Dockera za każdym razem,
uruchom stos z podmontowanym pakietem QA Lab:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` utrzymuje usługi Dockera na wcześniej zbudowanym obrazie i podmontowuje
`extensions/qa-lab/web/dist` do kontenera `qa-lab`. `qa:lab:watch`
przebudowuje ten pakiet po zmianach, a przeglądarka automatycznie odświeża się, gdy hash
zasobów QA Lab ulegnie zmianie.

Aby uruchomić ścieżkę smoke z rzeczywistym transportem Matrix, wykonaj:

```bash
pnpm openclaw qa matrix
```

Ta ścieżka provisionuje jednorazowy homeserver Tuwunel w Dockerze, rejestruje
tymczasowych użytkowników drivera, SUT i obserwatora, tworzy jeden prywatny pokój, a następnie
uruchamia rzeczywistą wtyczkę Matrix w podrzędnym procesie gateway QA. Ścieżka z żywym transportem
utrzymuje konfigurację podrzędną ograniczoną do testowanego transportu, więc Matrix działa bez
`qa-channel` w konfiguracji podrzędnej. Zapisuje ustrukturyzowane artefakty raportu oraz
połączony log stdout/stderr do wybranego katalogu wyjściowego Matrix QA. Aby
przechwycić także zewnętrzne dane wyjściowe budowania/uruchamiania z `scripts/run-node.mjs`,
ustaw `OPENCLAW_RUN_NODE_OUTPUT_LOG=<path>` na plik logu lokalny względem repozytorium.

Aby uruchomić ścieżkę smoke z rzeczywistym transportem Telegram, wykonaj:

```bash
pnpm openclaw qa telegram
```

Ta ścieżka kieruje ruch do jednej rzeczywistej prywatnej grupy Telegram zamiast provisionować
jednorazowy serwer. Wymaga `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` oraz
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`, a także dwóch różnych botów w tej samej
prywatnej grupie. Bot SUT musi mieć nazwę użytkownika Telegram, a obserwacja bot-do-bota
działa najlepiej, gdy oba boty mają włączony tryb Bot-to-Bot Communication Mode
w `@BotFather`.

Ścieżki z żywym transportem współdzielą teraz jeden mniejszy kontrakt zamiast tego, by każda
wymyślała własny kształt listy scenariuszy:

`qa-channel` pozostaje szerokim, syntetycznym zestawem testów zachowania produktu i nie jest częścią
macierzy pokrycia żywego transportu.

| Ścieżka  | Canary | Bramka wzmianek | Blokada allowlisty | Odpowiedź najwyższego poziomu | Wznowienie po restarcie | Dalszy ciąg wątku | Izolacja wątku | Obserwacja reakcji | Komenda help |
| -------- | ------ | --------------- | ------------------ | ----------------------------- | ----------------------- | ----------------- | -------------- | ------------------ | ------------ |
| Matrix   | x      | x               | x                  | x                             | x                       | x                 | x              | x                  |              |
| Telegram | x      |                 |                    |                               |                         |                   |                |                    | x            |

Dzięki temu `qa-channel` pozostaje szerokim zestawem testów zachowania produktu, podczas gdy Matrix,
Telegram i przyszłe żywe transporty współdzielą jedną jawną listę kontrolną kontraktu transportowego.

Aby uruchomić ścieżkę w jednorazowej maszynie wirtualnej Linux bez wprowadzania Dockera do ścieżki QA, wykonaj:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

To uruchamia świeżego gościa Multipass, instaluje zależności, buduje OpenClaw
wewnątrz gościa, uruchamia `qa suite`, a następnie kopiuje zwykły raport QA i
podsumowanie z powrotem do `.artifacts/qa-e2e/...` na hoście.
Ponownie wykorzystuje to samo zachowanie wyboru scenariuszy co `qa suite` na hoście.
Uruchomienia hosta i Multipass suite domyślnie wykonują wiele wybranych scenariuszy równolegle
z izolowanymi workerami gateway, do 64 workerów lub liczby wybranych
scenariuszy. Użyj `--concurrency <count>`, aby dostroić liczbę workerów, lub
`--concurrency 1` do wykonania szeregowego.
Uruchomienia live przekazują obsługiwane wejścia uwierzytelniania QA, które są praktyczne dla
gościa: klucze dostawców oparte na zmiennych środowiskowych, ścieżkę konfiguracji dostawcy QA live oraz
`CODEX_HOME`, jeśli jest obecne. Utrzymuj `--output-dir` w obrębie repozytorium, aby gość
mógł zapisywać z powrotem przez podmontowany workspace.

## Seedy oparte na repozytorium

Zasoby seedowane znajdują się w `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Są one celowo przechowywane w git, aby plan QA był widoczny zarówno dla ludzi, jak i dla
agenta.

`qa-lab` powinien pozostać ogólnym runnerem Markdown. Każdy plik Markdown scenariusza jest
źródłem prawdy dla jednego uruchomienia testu i powinien definiować:

- metadane scenariusza
- odwołania do dokumentacji i kodu
- opcjonalne wymagania dotyczące Plugin
- opcjonalną łatkę konfiguracji gateway
- wykonywalny `qa-flow`

Powierzchnia wielokrotnego użytku środowiska uruchomieniowego, która obsługuje `qa-flow`, może pozostać
ogólna i przekrojowa. Na przykład scenariusze Markdown mogą łączyć pomocniki po stronie transportu
z pomocnikami po stronie przeglądarki, które sterują osadzonym interfejsem Control UI przez
łącze Gateway `browser.request`, bez dodawania runnera dla przypadku specjalnego.

Lista bazowa powinna pozostać wystarczająco szeroka, aby obejmować:

- czat DM i kanałowy
- zachowanie wątków
- cykl życia akcji wiadomości
- wywołania zwrotne Cron
- odwoływanie się do pamięci
- przełączanie modeli
- przekazanie do subagenta
- czytanie repozytorium i dokumentacji
- jedno małe zadanie budowania, takie jak Lobster Invaders

## Adaptery transportu

`qa-lab` jest właścicielem ogólnego łącza transportowego dla scenariuszy QA w Markdown.
`qa-channel` jest pierwszym adapterem na tym łączu, ale docelowy projekt jest szerszy:
przyszłe rzeczywiste lub syntetyczne kanały powinny podłączać się do tego samego runnera suite
zamiast dodawać runner QA specyficzny dla transportu.

Na poziomie architektury podział wygląda następująco:

- `qa-lab` jest właścicielem ogólnego wykonywania scenariuszy, współbieżności workerów, zapisu artefaktów i raportowania.
- adapter transportu jest właścicielem konfiguracji gateway, gotowości, obserwacji ruchu przychodzącego i wychodzącego, działań transportowych oraz znormalizowanego stanu transportu.
- pliki scenariuszy Markdown w `qa/scenarios/` definiują przebieg testu; `qa-lab` dostarcza wielokrotnego użytku powierzchnię środowiska uruchomieniowego, która je wykonuje.

Wskazówki wdrożeniowe dla maintainerów dotyczące nowych adapterów kanałów znajdują się w
[Testing](/pl/help/testing#adding-a-channel-to-qa).

## Raportowanie

`qa-lab` eksportuje raport protokołu Markdown z obserwowanej osi czasu magistrali.
Raport powinien odpowiadać na pytania:

- Co zadziałało
- Co się nie powiodło
- Co pozostało zablokowane
- Jakie scenariusze uzupełniające warto dodać

W celu sprawdzenia charakteru i stylu uruchom ten sam scenariusz na wielu odwołaniach do modeli live
i zapisz oceniany raport Markdown:

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

Polecenie uruchamia lokalne podrzędne procesy gateway QA, a nie Docker. Scenariusze oceny charakteru
powinny ustawiać personę przez `SOUL.md`, a następnie wykonywać zwykłe tury użytkownika,
takie jak czat, pomoc dotyczącą workspace i małe zadania na plikach. Kandydacki model
nie powinien wiedzieć, że jest oceniany. Polecenie zachowuje każdy pełny
transkrypt, rejestruje podstawowe statystyki uruchomienia, a następnie prosi modele oceniające w trybie fast z
rozumowaniem `xhigh`, aby uszeregowały uruchomienia według naturalności, klimatu i humoru.
Użyj `--blind-judge-models` przy porównywaniu dostawców: prompt oceniający nadal otrzymuje
każdy transkrypt i status uruchomienia, ale odwołania kandydatów są zastępowane neutralnymi
etykietami, takimi jak `candidate-01`; raport mapuje rankingi z powrotem na rzeczywiste odwołania po
parsowaniu.
Uruchomienia kandydatów domyślnie używają trybu myślenia `high`, z `xhigh` dla modeli OpenAI,
które go obsługują. Zastąp konkretny model kandydujący inline za pomocą
`--model provider/model,thinking=<level>`. `--thinking <level>` nadal ustawia
globalną wartość zapasową, a starsza forma `--model-thinking <provider/model=level>` jest
zachowana dla zgodności.
Odwołania kandydatów OpenAI domyślnie używają trybu fast, dzięki czemu wykorzystywane jest przetwarzanie priorytetowe tam,
gdzie dostawca je obsługuje. Dodaj inline `,fast`, `,no-fast` lub `,fast=false`, gdy
pojedynczy kandydat lub model oceniający wymaga nadpisania. Przekaż `--fast` tylko wtedy, gdy chcesz
wymusić tryb fast dla każdego modelu kandydującego. Czasy trwania kandydatów i modeli oceniających są
zapisywane w raporcie do analizy benchmarkowej, ale prompty oceniające wyraźnie mówią,
aby nie oceniać według szybkości.
Uruchomienia modeli kandydujących i oceniających domyślnie używają współbieżności 16. Zmniejsz
`--concurrency` lub `--judge-concurrency`, gdy limity dostawcy lub obciążenie lokalnego gateway
sprawiają, że uruchomienie jest zbyt zaszumione.
Gdy nie zostanie przekazany żaden kandydat `--model`, character eval domyślnie używa
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` oraz
`google/gemini-3.1-pro-preview`, gdy nie zostanie przekazany żaden `--model`.
Gdy nie zostanie przekazany żaden `--judge-model`, modele oceniające domyślnie używają
`openai/gpt-5.4,thinking=xhigh,fast` oraz
`anthropic/claude-opus-4-6,thinking=high`.

## Powiązana dokumentacja

- [Testing](/pl/help/testing)
- [QA Channel](/pl/channels/qa-channel)
- [Dashboard](/web/dashboard)
