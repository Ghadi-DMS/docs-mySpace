---
read_when:
    - Dostosowywanie analizowania dyrektyw myślenia, trybu szybkiego lub szczegółowego albo ich ustawień domyślnych
summary: Składnia dyrektyw dla `/think`, `/fast`, `/verbose`, `/trace` oraz widoczności rozumowania
title: Poziomy myślenia
x-i18n:
    generated_at: "2026-04-12T23:34:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f3b1341281f07ba4e9061e3355845dca234be04cc0d358594312beeb7676e68
    source_path: tools/thinking.md
    workflow: 15
---

# Poziomy myślenia (dyrektywy `/think`)

## Co to robi

- Dyrektywa inline w dowolnej treści przychodzącej: `/t <level>`, `/think:<level>` lub `/thinking <level>`.
- Poziomy (aliasy): `off | minimal | low | medium | high | xhigh | adaptive`
  - minimal → „think”
  - low → „think hard”
  - medium → „think harder”
  - high → „ultrathink” (maksymalny budżet)
  - xhigh → „ultrathink+” (tylko modele GPT-5.2 + Codex)
  - adaptive → zarządzany przez dostawcę adaptacyjny budżet rozumowania (obsługiwany dla rodziny modeli Anthropic Claude 4.6)
  - `x-high`, `x_high`, `extra-high`, `extra high` i `extra_high` są mapowane na `xhigh`.
  - `highest`, `max` są mapowane na `high`.
- Uwagi dotyczące dostawców:
  - Modele Anthropic Claude 4.6 domyślnie używają `adaptive`, gdy nie ustawiono jawnego poziomu myślenia.
  - MiniMax (`minimax/*`) na ścieżce streamingu zgodnej z Anthropic domyślnie używa `thinking: { type: "disabled" }`, chyba że jawnie ustawisz thinking w parametrach modelu lub parametrach żądania. Zapobiega to wyciekom delt `reasoning_content` z nienatywnego formatu streamu Anthropic używanego przez MiniMax.
  - Z.AI (`zai/*`) obsługuje tylko binarne thinking (`on`/`off`). Każdy poziom inny niż `off` jest traktowany jako `on` (mapowany na `low`).
  - Moonshot (`moonshot/*`) mapuje `/think off` na `thinking: { type: "disabled" }`, a każdy poziom inny niż `off` na `thinking: { type: "enabled" }`. Gdy thinking jest włączony, Moonshot akceptuje tylko `tool_choice` `auto|none`; OpenClaw normalizuje niezgodne wartości do `auto`.

## Kolejność rozstrzygania

1. Dyrektywa inline w wiadomości (dotyczy tylko tej wiadomości).
2. Nadpisanie sesji (ustawiane przez wysłanie wiadomości zawierającej tylko dyrektywę).
3. Domyślna wartość per agent (`agents.list[].thinkingDefault` w konfiguracji).
4. Globalna wartość domyślna (`agents.defaults.thinkingDefault` w konfiguracji).
5. Fallback: `adaptive` dla modeli Anthropic Claude 4.6, `low` dla innych modeli obsługujących rozumowanie, w przeciwnym razie `off`.

## Ustawianie domyślnej wartości dla sesji

- Wyślij wiadomość, która zawiera **wyłącznie** dyrektywę (dozwolone białe znaki), np. `/think:medium` albo `/t high`.
- To ustawienie utrzymuje się dla bieżącej sesji (domyślnie per nadawca); jest czyszczone przez `/think:off` albo reset bezczynności sesji.
- Wysyłana jest odpowiedź potwierdzająca (`Thinking level set to high.` / `Thinking disabled.`). Jeśli poziom jest nieprawidłowy (np. `/thinking big`), polecenie zostaje odrzucone z podpowiedzią, a stan sesji pozostaje bez zmian.
- Wyślij `/think` (lub `/think:`) bez argumentu, aby zobaczyć bieżący poziom myślenia.

## Zastosowanie przez agenta

- **Embedded Pi**: rozstrzygnięty poziom jest przekazywany do runtime agenta Pi działającego in-process.

## Tryb szybki (`/fast`)

- Poziomy: `on|off`.
- Wiadomość zawierająca tylko dyrektywę przełącza nadpisanie trybu szybkiego dla sesji i odpowiada `Fast mode enabled.` / `Fast mode disabled.`.
- Wyślij `/fast` (lub `/fast status`) bez trybu, aby zobaczyć bieżący efektywny stan trybu szybkiego.
- OpenClaw rozstrzyga tryb szybki w tej kolejności:
  1. Inline/dyrektywa-only `/fast on|off`
  2. Nadpisanie sesji
  3. Domyślna wartość per agent (`agents.list[].fastModeDefault`)
  4. Konfiguracja per model: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Fallback: `off`
- Dla `openai/*` tryb szybki mapuje się na priorytetowe przetwarzanie OpenAI przez wysyłanie `service_tier=priority` w obsługiwanych żądaniach Responses.
- Dla `openai-codex/*` tryb szybki wysyła tę samą flagę `service_tier=priority` w Codex Responses. OpenClaw utrzymuje jeden współdzielony przełącznik `/fast` dla obu ścieżek uwierzytelniania.
- Dla bezpośrednich publicznych żądań `anthropic/*`, w tym ruchu uwierzytelnianego OAuth wysyłanego do `api.anthropic.com`, tryb szybki mapuje się na poziomy usług Anthropic: `/fast on` ustawia `service_tier=auto`, a `/fast off` ustawia `service_tier=standard_only`.
- Dla `minimax/*` na ścieżce zgodnej z Anthropic, `/fast on` (lub `params.fastMode: true`) przepisuje `MiniMax-M2.7` na `MiniMax-M2.7-highspeed`.
- Jawne parametry modelu Anthropic `serviceTier` / `service_tier` nadpisują domyślną wartość trybu szybkiego, gdy ustawione są oba. OpenClaw nadal pomija wstrzykiwanie poziomu usługi Anthropic dla bazowych URL proxy innych niż Anthropic.

## Dyrektywy szczegółowości (`/verbose` lub `/v`)

- Poziomy: `on` (minimalny) | `full` | `off` (domyślnie).
- Wiadomość zawierająca tylko dyrektywę przełącza szczegółowość sesji i odpowiada `Verbose logging enabled.` / `Verbose logging disabled.`; nieprawidłowe poziomy zwracają podpowiedź bez zmiany stanu.
- `/verbose off` zapisuje jawne nadpisanie sesji; wyczyść je w UI Sessions, wybierając `inherit`.
- Dyrektywa inline dotyczy tylko tej wiadomości; w pozostałych przypadkach stosowane są domyślne wartości sesji/globalne.
- Wyślij `/verbose` (lub `/verbose:`) bez argumentu, aby zobaczyć bieżący poziom szczegółowości.
- Gdy szczegółowość jest włączona, agenci emitujący ustrukturyzowane wyniki narzędzi (Pi, inne agenty JSON) odsyłają każde wywołanie narzędzia jako osobną wiadomość zawierającą tylko metadane, poprzedzoną `<emoji> <tool-name>: <arg>`, jeśli to możliwe (ścieżka/polecenie). Te podsumowania narzędzi są wysyłane natychmiast po starcie każdego narzędzia (osobne dymki), a nie jako delty streamingu.
- Podsumowania błędów narzędzi pozostają widoczne w trybie normalnym, ale surowe sufiksy szczegółów błędów są ukryte, chyba że `verbose` ma wartość `on` lub `full`.
- Gdy `verbose` ma wartość `full`, po zakończeniu przekazywane są także wyjścia narzędzi (osobny dymek, obcięty do bezpiecznej długości). Jeśli przełączysz `/verbose on|full|off` podczas trwającego uruchomienia, kolejne dymki narzędzi będą respektować nowe ustawienie.

## Dyrektywy śledzenia Plugin (`/trace`)

- Poziomy: `on` | `off` (domyślnie).
- Wiadomość zawierająca tylko dyrektywę przełącza wyjście śledzenia pluginów dla sesji i odpowiada `Plugin trace enabled.` / `Plugin trace disabled.`.
- Dyrektywa inline dotyczy tylko tej wiadomości; w pozostałych przypadkach stosowane są domyślne wartości sesji/globalne.
- Wyślij `/trace` (lub `/trace:`) bez argumentu, aby zobaczyć bieżący poziom śledzenia.
- `/trace` jest węższe niż `/verbose`: ujawnia tylko linie trace/debug należące do pluginu, takie jak podsumowania debug Active Memory.
- Linie trace mogą pojawiać się w `/status` oraz jako dodatkowa wiadomość diagnostyczna po zwykłej odpowiedzi asystenta.

## Widoczność rozumowania (`/reasoning`)

- Poziomy: `on|off|stream`.
- Wiadomość zawierająca tylko dyrektywę przełącza pokazywanie bloków myślenia w odpowiedziach.
- Gdy jest włączone, rozumowanie jest wysyłane jako **osobna wiadomość** poprzedzona `Reasoning:`.
- `stream` (tylko Telegram): streamuje rozumowanie do roboczego dymka Telegram podczas generowania odpowiedzi, a następnie wysyła końcową odpowiedź bez rozumowania.
- Alias: `/reason`.
- Wyślij `/reasoning` (lub `/reasoning:`) bez argumentu, aby zobaczyć bieżący poziom rozumowania.
- Kolejność rozstrzygania: dyrektywa inline, następnie nadpisanie sesji, następnie wartość domyślna per agent (`agents.list[].reasoningDefault`), a na końcu fallback (`off`).

## Powiązane

- Dokumentacja trybu Elevated znajduje się w [trybie Elevated](/pl/tools/elevated).

## Heartbeat

- Treścią sondy Heartbeat jest skonfigurowany prompt heartbeat (domyślnie: `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Dyrektywy inline w wiadomości heartbeat są stosowane normalnie (ale unikaj zmieniania domyślnych ustawień sesji przez heartbeat).
- Dostarczanie Heartbeat domyślnie wysyła tylko końcowy ładunek. Aby wysyłać także osobną wiadomość `Reasoning:` (gdy jest dostępna), ustaw `agents.defaults.heartbeat.includeReasoning: true` albo per agent `agents.list[].heartbeat.includeReasoning: true`.

## UI czatu webowego

- Selektor myślenia w UI czatu webowego odzwierciedla zapisany poziom sesji z magazynu/config sesji przychodzącej podczas ładowania strony.
- Wybranie innego poziomu od razu zapisuje nadpisanie sesji przez `sessions.patch`; nie czeka na kolejne wysłanie i nie jest jednorazowym nadpisaniem `thinkingOnce`.
- Pierwsza opcja to zawsze `Default (<resolved level>)`, gdzie rozstrzygnięta wartość domyślna pochodzi z aktywnego modelu sesji: `adaptive` dla Claude 4.6 w Anthropic/Bedrock, `low` dla innych modeli obsługujących rozumowanie, w przeciwnym razie `off`.
- Selektor pozostaje świadomy dostawcy:
  - większość dostawców pokazuje `off | minimal | low | medium | high | adaptive`
  - Z.AI pokazuje binarne `off | on`
- `/think:<level>` nadal działa i aktualizuje ten sam zapisany poziom sesji, dzięki czemu dyrektywy czatu i selektor pozostają zsynchronizowane.
