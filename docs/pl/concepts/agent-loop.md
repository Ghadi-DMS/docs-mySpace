---
read_when:
    - Potrzebujesz dokładnego omówienia pętli agenta lub zdarzeń cyklu życia
summary: Cykl życia pętli agenta, strumienie i semantyka oczekiwania
title: Pętla agenta
x-i18n:
    generated_at: "2026-04-09T01:27:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 32d3a73df8dabf449211a6183a70dcfd2a9b6f584dc76d0c4c9147582b2ca6a1
    source_path: concepts/agent-loop.md
    workflow: 15
---

# Pętla agenta (OpenClaw)

Pętla agentowa to pełny „rzeczywisty” przebieg działania agenta: przyjęcie danych wejściowych → złożenie kontekstu → wnioskowanie modelu →
wykonanie narzędzi → strumieniowanie odpowiedzi → utrwalenie. To autorytatywna ścieżka, która przekształca wiadomość
w działania i końcową odpowiedź, jednocześnie utrzymując spójny stan sesji.

W OpenClaw pętla to pojedynczy, serializowany przebieg na sesję, który emituje zdarzenia cyklu życia i strumieni
gdy model myśli, wywołuje narzędzia i strumieniuje dane wyjściowe. Ten dokument wyjaśnia, jak ta rzeczywista pętla
jest połączona od początku do końca.

## Punkty wejścia

- Gateway RPC: `agent` i `agent.wait`.
- CLI: polecenie `agent`.

## Jak to działa (na wysokim poziomie)

1. RPC `agent` weryfikuje parametry, rozwiązuje sesję (sessionKey/sessionId), utrwala metadane sesji i natychmiast zwraca `{ runId, acceptedAt }`.
2. `agentCommand` uruchamia agenta:
   - rozwiązuje domyślne wartości modelu + thinking/verbose
   - ładuje migawkę Skills
   - wywołuje `runEmbeddedPiAgent` (środowisko uruchomieniowe pi-agent-core)
   - emituje **koniec/błąd cyklu życia**, jeśli osadzona pętla tego nie wyemituje
3. `runEmbeddedPiAgent`:
   - serializuje przebiegi przez kolejki per sesja i globalne
   - rozwiązuje profil modelu + auth i buduje sesję pi
   - subskrybuje zdarzenia pi i strumieniuje delty asystenta/narzędzi
   - wymusza limit czasu -> przerywa przebieg po jego przekroczeniu
   - zwraca ładunki + metadane użycia
4. `subscribeEmbeddedPiSession` mostkuje zdarzenia pi-agent-core do strumienia OpenClaw `agent`:
   - zdarzenia narzędzi => `stream: "tool"`
   - delty asystenta => `stream: "assistant"`
   - zdarzenia cyklu życia => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait` używa `waitForAgentRun`:
   - czeka na **koniec/błąd cyklu życia** dla `runId`
   - zwraca `{ status: ok|error|timeout, startedAt, endedAt, error? }`

## Kolejkowanie + współbieżność

- Przebiegi są serializowane dla każdego klucza sesji (pas sesji) i opcjonalnie przez pas globalny.
- Zapobiega to wyścigom narzędzi/sesji i utrzymuje spójną historię sesji.
- Kanały wiadomości mogą wybierać tryby kolejki (collect/steer/followup), które zasilają ten system pasów.
  Zobacz [Kolejka poleceń](/pl/concepts/queue).

## Przygotowanie sesji + przestrzeni roboczej

- Przestrzeń robocza jest rozwiązywana i tworzona; przebiegi sandboxowane mogą przekierowywać do katalogu głównego przestrzeni roboczej sandboxa.
- Skills są ładowane (lub ponownie używane z migawki) i wstrzykiwane do env oraz promptu.
- Pliki bootstrap/kontekstu są rozwiązywane i wstrzykiwane do raportu promptu systemowego.
- Uzyskiwana jest blokada zapisu sesji; `SessionManager` jest otwierany i przygotowywany przed strumieniowaniem.

## Składanie promptu + prompt systemowy

- Prompt systemowy jest budowany z bazowego promptu OpenClaw, promptu Skills, kontekstu bootstrap i nadpisań dla przebiegu.
- Wymuszane są limity specyficzne dla modelu i tokeny rezerwowe dla kompaktowania.
- Zobacz [Prompt systemowy](/pl/concepts/system-prompt), aby sprawdzić, co widzi model.

## Punkty hooków (gdzie można przechwycić)

OpenClaw ma dwa systemy hooków:

- **Hooki wewnętrzne** (hooki Gateway): skrypty sterowane zdarzeniami dla poleceń i zdarzeń cyklu życia.
- **Hooki pluginów**: punkty rozszerzeń wewnątrz cyklu życia agenta/narzędzi i potoku gateway.

### Hooki wewnętrzne (hooki Gateway)

- **`agent:bootstrap`**: uruchamia się podczas budowania plików bootstrap przed finalizacją promptu systemowego.
  Użyj tego, aby dodawać/usuwać pliki kontekstu bootstrap.
- **Hooki poleceń**: `/new`, `/reset`, `/stop` i inne zdarzenia poleceń (zobacz dokumentację hooków).

Zobacz [Hooki](/pl/automation/hooks), aby sprawdzić konfigurację i przykłady.

### Hooki pluginów (cykl życia agenta + gateway)

Uruchamiają się wewnątrz pętli agenta lub potoku gateway:

- **`before_model_resolve`**: uruchamia się przed sesją (bez `messages`), aby deterministycznie nadpisać dostawcę/model przed rozwiązywaniem modelu.
- **`before_prompt_build`**: uruchamia się po załadowaniu sesji (z `messages`), aby wstrzyknąć `prependContext`, `systemPrompt`, `prependSystemContext` lub `appendSystemContext` przed wysłaniem promptu. Używaj `prependContext` dla dynamicznego tekstu per tura, a pól kontekstu systemowego dla stabilnych wskazówek, które powinny znajdować się w przestrzeni promptu systemowego.
- **`before_agent_start`**: hook zgodności wstecznej; może uruchamiać się w obu fazach; preferuj powyższe jawne hooki.
- **`before_agent_reply`**: uruchamia się po działaniach inline i przed wywołaniem LLM, pozwalając pluginowi przejąć turę i zwrócić syntetyczną odpowiedź lub całkowicie wyciszyć turę.
- **`agent_end`**: sprawdza końcową listę wiadomości i metadane przebiegu po zakończeniu.
- **`before_compaction` / `after_compaction`**: obserwują lub opisują cykle kompaktowania.
- **`before_tool_call` / `after_tool_call`**: przechwytują parametry/wyniki narzędzi.
- **`before_install`**: sprawdza wbudowane wyniki skanowania i opcjonalnie blokuje instalacje skillów lub pluginów.
- **`tool_result_persist`**: synchronicznie przekształca wyniki narzędzi przed zapisaniem ich do transkryptu sesji.
- **`message_received` / `message_sending` / `message_sent`**: hooki wiadomości przychodzących i wychodzących.
- **`session_start` / `session_end`**: granice cyklu życia sesji.
- **`gateway_start` / `gateway_stop`**: zdarzenia cyklu życia gateway.

Zasady decyzji hooków dla zabezpieczeń wychodzących/narzędzi:

- `before_tool_call`: `{ block: true }` jest końcowe i zatrzymuje handlery o niższym priorytecie.
- `before_tool_call`: `{ block: false }` nic nie robi i nie usuwa wcześniejszej blokady.
- `before_install`: `{ block: true }` jest końcowe i zatrzymuje handlery o niższym priorytecie.
- `before_install`: `{ block: false }` nic nie robi i nie usuwa wcześniejszej blokady.
- `message_sending`: `{ cancel: true }` jest końcowe i zatrzymuje handlery o niższym priorytecie.
- `message_sending`: `{ cancel: false }` nic nie robi i nie usuwa wcześniejszego anulowania.

Zobacz [Hooki pluginów](/pl/plugins/architecture#provider-runtime-hooks), aby sprawdzić API hooków i szczegóły rejestracji.

## Strumieniowanie + odpowiedzi częściowe

- Delty asystenta są strumieniowane z pi-agent-core i emitowane jako zdarzenia `assistant`.
- Strumieniowanie bloków może emitować odpowiedzi częściowe na `text_end` lub `message_end`.
- Strumieniowanie rozumowania może być emitowane jako osobny strumień lub jako odpowiedzi blokowe.
- Zobacz [Strumieniowanie](/pl/concepts/streaming), aby sprawdzić dzielenie na fragmenty i zachowanie odpowiedzi blokowych.

## Wykonanie narzędzi + narzędzia do wiadomości

- Zdarzenia start/aktualizacja/koniec narzędzi są emitowane w strumieniu `tool`.
- Wyniki narzędzi są sanitizowane pod kątem rozmiaru i ładunków obrazów przed logowaniem/emisją.
- Wysłania narzędzi do wiadomości są śledzone, aby wyeliminować zduplikowane potwierdzenia asystenta.

## Kształtowanie odpowiedzi + wyciszanie

- Końcowe ładunki są składane z:
  - tekstu asystenta (i opcjonalnie rozumowania)
  - podsumowań narzędzi inline (gdy verbose + dozwolone)
  - tekstu błędu asystenta, gdy model zwraca błąd
- Dokładny token wyciszenia `NO_REPLY` / `no_reply` jest odfiltrowywany z wychodzących
  ładunków.
- Duplikaty narzędzi do wiadomości są usuwane z końcowej listy ładunków.
- Jeśli nie pozostaną żadne renderowalne ładunki, a narzędzie zwróciło błąd, emitowana jest
  zapasowa odpowiedź błędu narzędzia
  (chyba że narzędzie do wiadomości już wysłało widoczną dla użytkownika odpowiedź).

## Kompaktowanie + ponowienia

- Automatyczne kompaktowanie emituje zdarzenia strumienia `compaction` i może wywołać ponowienie.
- Przy ponowieniu bufory w pamięci i podsumowania narzędzi są resetowane, aby uniknąć zduplikowanego wyjścia.
- Zobacz [Kompaktowanie](/pl/concepts/compaction), aby sprawdzić potok kompaktowania.

## Strumienie zdarzeń (obecnie)

- `lifecycle`: emitowany przez `subscribeEmbeddedPiSession` (oraz awaryjnie przez `agentCommand`)
- `assistant`: strumieniowane delty z pi-agent-core
- `tool`: strumieniowane zdarzenia narzędzi z pi-agent-core

## Obsługa kanału czatu

- Delty asystenta są buforowane do wiadomości czatu `delta`.
- Końcowa wiadomość czatu `final` jest emitowana przy **końcu/błędzie cyklu życia**.

## Limity czasu

- Domyślnie `agent.wait`: 30 s (tylko oczekiwanie). Parametr `timeoutMs` nadpisuje tę wartość.
- Czas działania agenta: domyślnie `agents.defaults.timeoutSeconds` to 172800 s (48 godzin); wymuszany przez licznik przerwania w `runEmbeddedPiAgent`.
- Limit bezczynności LLM: `agents.defaults.llm.idleTimeoutSeconds` przerywa żądanie modelu, gdy przed upływem okna bezczynności nie nadejdą żadne fragmenty odpowiedzi. Ustaw tę wartość jawnie dla wolnych modeli lokalnych lub dostawców rozumowania/wywołań narzędzi; ustaw `0`, aby wyłączyć. Jeśli nie jest ustawiona, OpenClaw używa `agents.defaults.timeoutSeconds`, gdy jest skonfigurowane, w przeciwnym razie 60 s. Przebiegi wyzwalane przez cron bez jawnego limitu czasu LLM lub agenta wyłączają mechanizm bezczynności i polegają na zewnętrznym limicie czasu crona.

## Gdzie wszystko może zakończyć się wcześniej

- Limit czasu agenta (przerwanie)
- AbortSignal (anulowanie)
- Rozłączenie gateway lub limit czasu RPC
- Limit czasu `agent.wait` (tylko oczekiwanie, nie zatrzymuje agenta)

## Powiązane

- [Narzędzia](/pl/tools) — dostępne narzędzia agenta
- [Hooki](/pl/automation/hooks) — skrypty sterowane zdarzeniami wyzwalane przez zdarzenia cyklu życia agenta
- [Kompaktowanie](/pl/concepts/compaction) — jak podsumowywane są długie rozmowy
- [Zgody exec](/pl/tools/exec-approvals) — bramki zatwierdzania dla poleceń powłoki
- [Thinking](/pl/tools/thinking) — konfiguracja poziomu myślenia/rozumowania
