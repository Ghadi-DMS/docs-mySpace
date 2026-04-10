---
read_when:
    - Potrzebujesz dokładnego omówienia pętli agenta lub zdarzeń cyklu życia
summary: Cykl życia pętli agenta, strumienie i semantyka oczekiwania
title: Pętla agenta
x-i18n:
    generated_at: "2026-04-10T09:44:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: b6831a5b11e9100e49f650feca51ab44a2bef242ce1b5db2766d0b3b5c5ba729
    source_path: concepts/agent-loop.md
    workflow: 15
---

# Pętla agenta (OpenClaw)

Pętla agenta to pełny, „rzeczywisty” przebieg działania agenta: przyjęcie danych wejściowych → złożenie kontekstu → wnioskowanie modelu →
wykonanie narzędzi → strumieniowanie odpowiedzi → utrwalenie. To autorytatywna ścieżka, która zamienia wiadomość
w działania i końcową odpowiedź, przy zachowaniu spójnego stanu sesji.

W OpenClaw pętla to pojedynczy, serializowany przebieg na sesję, który emituje zdarzenia cyklu życia i zdarzenia strumienia,
gdy model analizuje, wywołuje narzędzia i strumieniuje wynik. Ten dokument wyjaśnia, jak ta rzeczywista pętla jest połączona od początku do końca.

## Punkty wejścia

- Gateway RPC: `agent` i `agent.wait`.
- CLI: polecenie `agent`.

## Jak to działa (wysoki poziom)

1. RPC `agent` sprawdza poprawność parametrów, rozpoznaje sesję (sessionKey/sessionId), utrwala metadane sesji i natychmiast zwraca `{ runId, acceptedAt }`.
2. `agentCommand` uruchamia agenta:
   - rozpoznaje model + domyślne wartości thinking/verbose
   - ładuje migawkę Skills
   - wywołuje `runEmbeddedPiAgent` (środowisko uruchomieniowe pi-agent-core)
   - emituje **lifecycle end/error**, jeśli osadzona pętla sama go nie emituje
3. `runEmbeddedPiAgent`:
   - serializuje przebiegi przez kolejki per sesja i globalne
   - rozpoznaje model + profil uwierzytelniania i buduje sesję Pi
   - subskrybuje zdarzenia pi i strumieniuje delty asystenta/narzędzi
   - wymusza limit czasu -> przerywa przebieg po jego przekroczeniu
   - zwraca ładunki + metadane użycia
4. `subscribeEmbeddedPiSession` mostkuje zdarzenia pi-agent-core do strumienia OpenClaw `agent`:
   - zdarzenia narzędzi => `stream: "tool"`
   - delty asystenta => `stream: "assistant"`
   - zdarzenia cyklu życia => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait` używa `waitForAgentRun`:
   - czeka na **lifecycle end/error** dla `runId`
   - zwraca `{ status: ok|error|timeout, startedAt, endedAt, error? }`

## Kolejkowanie + współbieżność

- Przebiegi są serializowane dla każdego klucza sesji (pas sesji) i opcjonalnie przez globalny pas.
- Zapobiega to wyścigom narzędzi/sesji i utrzymuje spójną historię sesji.
- Kanały wiadomości mogą wybierać tryby kolejki (collect/steer/followup), które zasilają ten system pasów.
  Zobacz [Command Queue](/pl/concepts/queue).

## Przygotowanie sesji + obszaru roboczego

- Obszar roboczy jest rozpoznawany i tworzony; przebiegi piaskownicowane mogą zostać przekierowane do katalogu głównego obszaru roboczego sandboxa.
- Skills są ładowane (lub ponownie używane z migawki) i wstrzykiwane do środowiska oraz promptu.
- Pliki bootstrap/context są rozpoznawane i wstrzykiwane do raportu promptu systemowego.
- Uzyskiwana jest blokada zapisu sesji; `SessionManager` jest otwierany i przygotowywany przed rozpoczęciem strumieniowania.

## Składanie promptu + prompt systemowy

- Prompt systemowy jest budowany na podstawie bazowego promptu OpenClaw, promptu Skills, kontekstu bootstrap i nadpisań dla danego przebiegu.
- Wymuszane są limity specyficzne dla modelu oraz rezerwa tokenów na kompaktowanie.
- Zobacz [System prompt](/pl/concepts/system-prompt), aby sprawdzić, co widzi model.

## Punkty hooków (gdzie można przechwycić)

OpenClaw ma dwa systemy hooków:

- **Hooki wewnętrzne** (hooki Gateway): skrypty sterowane zdarzeniami dla poleceń i zdarzeń cyklu życia.
- **Hooki pluginów**: punkty rozszerzeń wewnątrz cyklu życia agenta/narzędzi i potoku gateway.

### Hooki wewnętrzne (hooki Gateway)

- **`agent:bootstrap`**: uruchamia się podczas budowania plików bootstrap, zanim prompt systemowy zostanie ostatecznie sfinalizowany.
  Użyj tego, aby dodawać/usuwać pliki kontekstu bootstrap.
- **Hooki poleceń**: `/new`, `/reset`, `/stop` i inne zdarzenia poleceń (zobacz dokumentację Hooks).

Zobacz [Hooks](/pl/automation/hooks), aby poznać konfigurację i przykłady.

### Hooki pluginów (cykl życia agenta + gateway)

Są uruchamiane wewnątrz pętli agenta lub potoku gateway:

- **`before_model_resolve`**: uruchamia się przed sesją (bez `messages`), aby deterministycznie nadpisać dostawcę/model przed rozpoznaniem modelu.
- **`before_prompt_build`**: uruchamia się po załadowaniu sesji (z `messages`), aby wstrzyknąć `prependContext`, `systemPrompt`, `prependSystemContext` lub `appendSystemContext` przed wysłaniem promptu. Używaj `prependContext` do dynamicznego tekstu dla danego obrotu i pól kontekstu systemowego do stabilnych wskazówek, które powinny znajdować się w obszarze promptu systemowego.
- **`before_agent_start`**: hook zgodności wstecznej; może uruchamiać się w obu fazach — preferuj powyższe jawne hooki.
- **`before_agent_reply`**: uruchamia się po działaniach inline i przed wywołaniem LLM, pozwalając pluginowi przejąć obrót i zwrócić syntetyczną odpowiedź lub całkowicie wyciszyć obrót.
- **`agent_end`**: umożliwia inspekcję końcowej listy wiadomości i metadanych przebiegu po zakończeniu.
- **`before_compaction` / `after_compaction`**: obserwują lub opisują cykle kompaktowania.
- **`before_tool_call` / `after_tool_call`**: przechwytują parametry/wyniki narzędzi.
- **`before_install`**: umożliwia inspekcję wyników wbudowanego skanowania i opcjonalne blokowanie instalacji Skills lub pluginów.
- **`tool_result_persist`**: synchronicznie przekształca wyniki narzędzi przed zapisaniem ich do transkryptu sesji.
- **`message_received` / `message_sending` / `message_sent`**: hooki wiadomości przychodzących i wychodzących.
- **`session_start` / `session_end`**: granice cyklu życia sesji.
- **`gateway_start` / `gateway_stop`**: zdarzenia cyklu życia gateway.

Zasady podejmowania decyzji przez hooki dla zabezpieczeń wiadomości wychodzących/narzędzi:

- `before_tool_call`: `{ block: true }` jest terminalne i zatrzymuje handlery o niższym priorytecie.
- `before_tool_call`: `{ block: false }` nic nie robi i nie usuwa wcześniejszej blokady.
- `before_install`: `{ block: true }` jest terminalne i zatrzymuje handlery o niższym priorytecie.
- `before_install`: `{ block: false }` nic nie robi i nie usuwa wcześniejszej blokady.
- `message_sending`: `{ cancel: true }` jest terminalne i zatrzymuje handlery o niższym priorytecie.
- `message_sending`: `{ cancel: false }` nic nie robi i nie usuwa wcześniejszego anulowania.

Zobacz [Plugin hooks](/pl/plugins/architecture#provider-runtime-hooks), aby poznać API hooków i szczegóły rejestracji.

## Strumieniowanie + odpowiedzi częściowe

- Delty asystenta są strumieniowane z pi-agent-core i emitowane jako zdarzenia `assistant`.
- Strumieniowanie bloków może emitować odpowiedzi częściowe przy `text_end` lub `message_end`.
- Strumieniowanie rozumowania może być emitowane jako osobny strumień lub jako odpowiedzi blokowe.
- Zobacz [Streaming](/pl/concepts/streaming), aby poznać zachowanie fragmentacji i odpowiedzi blokowych.

## Wykonywanie narzędzi + narzędzia wiadomości

- Zdarzenia start/update/end narzędzi są emitowane w strumieniu `tool`.
- Wyniki narzędzi są oczyszczane pod kątem rozmiaru i ładunków obrazów przed logowaniem/emisją.
- Wysłania przez narzędzia wiadomości są śledzone, aby tłumić zduplikowane potwierdzenia asystenta.

## Kształtowanie odpowiedzi + tłumienie

- Końcowe ładunki są składane z:
  - tekstu asystenta (i opcjonalnie rozumowania)
  - podsumowań narzędzi inline (gdy `verbose` jest włączone i dozwolone)
  - tekstu błędu asystenta, gdy model zwróci błąd
- Dokładny cichy token `NO_REPLY` / `no_reply` jest odfiltrowywany z wychodzących
  ładunków.
- Duplikaty z narzędzi wiadomości są usuwane z końcowej listy ładunków.
- Jeśli nie pozostaną żadne renderowalne ładunki, a narzędzie zwróciło błąd, emitowana jest zapasowa odpowiedź błędu narzędzia
  (chyba że narzędzie wiadomości wysłało już odpowiedź widoczną dla użytkownika).

## Kompaktowanie + ponowienia

- Automatyczne kompaktowanie emituje zdarzenia strumienia `compaction` i może wywołać ponowienie.
- Przy ponowieniu bufory w pamięci i podsumowania narzędzi są resetowane, aby uniknąć zduplikowanego wyjścia.
- Zobacz [Compaction](/pl/concepts/compaction), aby poznać potok kompaktowania.

## Strumienie zdarzeń (obecnie)

- `lifecycle`: emitowany przez `subscribeEmbeddedPiSession` (oraz awaryjnie przez `agentCommand`)
- `assistant`: strumieniowane delty z pi-agent-core
- `tool`: strumieniowane zdarzenia narzędzi z pi-agent-core

## Obsługa kanału czatu

- Delty asystenta są buforowane do komunikatów czatu `delta`.
- Końcowy komunikat czatu `final` jest emitowany przy **lifecycle end/error**.

## Limity czasu

- Domyślny `agent.wait`: 30 s (tylko oczekiwanie). Parametr `timeoutMs` go nadpisuje.
- Czas działania agenta: domyślnie `agents.defaults.timeoutSeconds` wynosi 172800 s (48 godzin); egzekwowany przez licznik przerwania w `runEmbeddedPiAgent`.
- Limit bezczynności LLM: `agents.defaults.llm.idleTimeoutSeconds` przerywa żądanie modelu, gdy przed upływem okna bezczynności nie nadejdą żadne fragmenty odpowiedzi. Ustaw go jawnie dla wolnych modeli lokalnych lub dostawców rozumowania/wywołań narzędzi; ustaw na 0, aby wyłączyć. Jeśli nie jest ustawiony, OpenClaw używa `agents.defaults.timeoutSeconds`, jeśli skonfigurowano, w przeciwnym razie 120 s. Przebiegi wyzwalane przez cron bez jawnego limitu czasu LLM lub agenta wyłączają nadzorcę bezczynności i polegają na zewnętrznym limicie czasu crona.

## Gdzie wszystko może zakończyć się wcześniej

- Limit czasu agenta (przerwanie)
- AbortSignal (anulowanie)
- Rozłączenie gateway lub limit czasu RPC
- Limit czasu `agent.wait` (tylko oczekiwanie, nie zatrzymuje agenta)

## Powiązane

- [Tools](/pl/tools) — dostępne narzędzia agenta
- [Hooks](/pl/automation/hooks) — skrypty sterowane zdarzeniami, wyzwalane przez zdarzenia cyklu życia agenta
- [Compaction](/pl/concepts/compaction) — jak podsumowywane są długie rozmowy
- [Exec Approvals](/pl/tools/exec-approvals) — bramki zatwierdzania dla poleceń powłoki
- [Thinking](/pl/tools/thinking) — konfiguracja poziomu thinking/rozumowania
