---
read_when:
    - Tworzenie lub debugowanie natywnych Pluginów OpenClaw
    - Zrozumienie modelu możliwości Pluginów lub granic własności
    - Praca nad potokiem ładowania Pluginów lub rejestrem
    - Implementowanie hooków środowiska uruchomieniowego dostawców lub Pluginów kanałów
sidebarTitle: Internals
summary: 'Wnętrze Pluginów: model możliwości, własność, kontrakty, potok ładowania i pomocniki środowiska uruchomieniowego'
title: Wnętrze Pluginów
x-i18n:
    generated_at: "2026-04-12T23:28:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 37361c1e9d2da57c77358396f19dfc7f749708b66ff68f1bf737d051b5d7675d
    source_path: plugins/architecture.md
    workflow: 15
---

# Wnętrze Pluginów

<Info>
  To jest **szczegółowe odniesienie architektoniczne**. Praktyczne przewodniki znajdziesz tutaj:
  - [Instalowanie i używanie pluginów](/pl/tools/plugin) — przewodnik użytkownika
  - [Pierwsze kroki](/pl/plugins/building-plugins) — pierwszy samouczek Pluginu
  - [Pluginy kanałów](/pl/plugins/sdk-channel-plugins) — tworzenie kanału wiadomości
  - [Pluginy dostawców](/pl/plugins/sdk-provider-plugins) — tworzenie dostawcy modeli
  - [Przegląd SDK](/pl/plugins/sdk-overview) — mapa importów i API rejestracji
</Info>

Ta strona opisuje wewnętrzną architekturę systemu Pluginów OpenClaw.

## Publiczny model możliwości

Możliwości to publiczny model **natywnych Pluginów** w OpenClaw. Każdy
natywny Plugin OpenClaw rejestruje się względem jednego lub większej liczby typów możliwości:

| Możliwość             | Metoda rejestracji                              | Przykładowe Pluginy                  |
| --------------------- | ----------------------------------------------- | ------------------------------------ |
| Wnioskowanie tekstowe | `api.registerProvider(...)`                     | `openai`, `anthropic`                |
| Backend wnioskowania CLI | `api.registerCliBackend(...)`                | `openai`, `anthropic`                |
| Mowa                  | `api.registerSpeechProvider(...)`               | `elevenlabs`, `microsoft`            |
| Transkrypcja w czasie rzeczywistym | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                 |
| Głos w czasie rzeczywistym | `api.registerRealtimeVoiceProvider(...)`   | `openai`                             |
| Rozumienie mediów     | `api.registerMediaUnderstandingProvider(...)`   | `openai`, `google`                   |
| Generowanie obrazów   | `api.registerImageGenerationProvider(...)`      | `openai`, `google`, `fal`, `minimax` |
| Generowanie muzyki    | `api.registerMusicGenerationProvider(...)`      | `google`, `minimax`                  |
| Generowanie wideo     | `api.registerVideoGenerationProvider(...)`      | `qwen`                               |
| Pobieranie z sieci    | `api.registerWebFetchProvider(...)`             | `firecrawl`                          |
| Wyszukiwanie w sieci  | `api.registerWebSearchProvider(...)`            | `google`                             |
| Kanał / wiadomości    | `api.registerChannel(...)`                      | `msteams`, `matrix`                  |

Plugin, który rejestruje zero możliwości, ale udostępnia hooki, narzędzia lub
usługi, to **starszy Plugin wyłącznie z hookami**. Ten wzorzec nadal jest w pełni wspierany.

### Podejście do kompatybilności zewnętrznej

Model możliwości jest wdrożony w core i używany dziś przez bundlowane/natywne Pluginy,
ale kompatybilność zewnętrznych Pluginów nadal wymaga wyższego progu niż „jest
eksportowane, więc jest zamrożone”.

Obecne wytyczne:

- **istniejące zewnętrzne Pluginy:** zachowuj działanie integracji opartych na hookach; traktuj
  to jako bazowy poziom kompatybilności
- **nowe bundlowane/natywne Pluginy:** preferuj jawną rejestrację możliwości zamiast
  podejść specyficznych dla dostawcy lub nowych projektów wyłącznie opartych na hookach
- **zewnętrzne Pluginy przyjmujące rejestrację możliwości:** dozwolone, ale traktuj
  powierzchnie pomocnicze specyficzne dla możliwości jako rozwijające się, chyba że dokumentacja wyraźnie oznacza
  dany kontrakt jako stabilny

Praktyczna zasada:

- API rejestracji możliwości są zamierzonym kierunkiem
- starsze hooki pozostają najbezpieczniejszą ścieżką bez ryzyka zepsucia dla zewnętrznych Pluginów w czasie
  przejścia
- eksportowane podścieżki pomocnicze nie są równoważne; preferuj wąski, udokumentowany
  kontrakt, a nie przypadkowo wyeksportowane pomocniki

### Kształty Pluginów

OpenClaw klasyfikuje każdy załadowany Plugin do określonego kształtu na podstawie jego rzeczywistego
zachowania rejestracyjnego (a nie tylko statycznych metadanych):

- **plain-capability** -- rejestruje dokładnie jeden typ możliwości (na przykład
  Plugin tylko dostawcy, taki jak `mistral`)
- **hybrid-capability** -- rejestruje wiele typów możliwości (na przykład
  `openai` odpowiada za wnioskowanie tekstowe, mowę, rozumienie mediów i
  generowanie obrazów)
- **hook-only** -- rejestruje tylko hooki (typowane lub niestandardowe), bez możliwości,
  narzędzi, poleceń ani usług
- **non-capability** -- rejestruje narzędzia, polecenia, usługi lub trasy, ale bez
  możliwości

Użyj `openclaw plugins inspect <id>`, aby zobaczyć kształt Pluginu i rozkład jego możliwości.
Szczegóły znajdziesz w [referencji CLI](/cli/plugins#inspect).

### Starsze hooki

Hook `before_agent_start` pozostaje wspierany jako ścieżka kompatybilności dla
Pluginów wyłącznie z hookami. Nadal zależą od niego starsze Pluginy używane w rzeczywistych wdrożeniach.

Kierunek:

- utrzymywać jego działanie
- dokumentować go jako starszy
- dla nadpisywania modelu/dostawcy preferować `before_model_resolve`
- dla modyfikacji promptu preferować `before_prompt_build`
- usuwać dopiero wtedy, gdy rzeczywiste użycie spadnie, a pokrycie fixture potwierdzi bezpieczeństwo migracji

### Sygnały kompatybilności

Po uruchomieniu `openclaw doctor` lub `openclaw plugins inspect <id>` możesz zobaczyć
jedną z tych etykiet:

| Sygnał                     | Znaczenie                                                    |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | Konfiguracja poprawnie się parsuje, a Pluginy są rozwiązywane |
| **compatibility advisory** | Plugin używa wspieranego, ale starszego wzorca (np. `hook-only`) |
| **legacy warning**         | Plugin używa `before_agent_start`, które jest przestarzałe   |
| **hard error**             | Konfiguracja jest nieprawidłowa lub Plugin nie załadował się |

Ani `hook-only`, ani `before_agent_start` nie zepsują dziś Twojego Pluginu --
`hook-only` ma charakter doradczy, a `before_agent_start` wywołuje jedynie ostrzeżenie. Te
sygnały pojawiają się również w `openclaw status --all` i `openclaw plugins doctor`.

## Przegląd architektury

System Pluginów OpenClaw ma cztery warstwy:

1. **Manifest + wykrywanie**
   OpenClaw znajduje kandydatów na Pluginy na podstawie skonfigurowanych ścieżek, katalogów workspace,
   globalnych katalogów rozszerzeń i bundlowanych rozszerzeń. Wykrywanie najpierw odczytuje natywne
   manifesty `openclaw.plugin.json` oraz wspierane manifesty bundli.
2. **Włączanie + walidacja**
   Core decyduje, czy wykryty Plugin jest włączony, wyłączony, zablokowany lub
   wybrany dla ekskluzywnego slotu, takiego jak pamięć.
3. **Ładowanie środowiska uruchomieniowego**
   Natywne Pluginy OpenClaw są ładowane w procesie przez jiti i rejestrują
   możliwości w centralnym rejestrze. Zgodne bundle są normalizowane do
   rekordów rejestru bez importowania kodu środowiska uruchomieniowego.
4. **Konsumpcja powierzchni**
   Reszta OpenClaw odczytuje rejestr, aby udostępniać narzędzia, kanały, konfigurację
   dostawców, hooki, trasy HTTP, polecenia CLI i usługi.

W przypadku CLI Pluginów wykrywanie poleceń głównych jest dodatkowo podzielone na dwa etapy:

- metadane na etapie parsowania pochodzą z `registerCli(..., { descriptors: [...] })`
- właściwy moduł CLI Pluginu może pozostać leniwy i rejestrować się przy pierwszym wywołaniu

Dzięki temu kod CLI należący do Pluginu pozostaje wewnątrz Pluginu, a OpenClaw nadal może
zarezerwować nazwy poleceń głównych przed parsowaniem.

Najważniejsza granica projektowa:

- wykrywanie + walidacja konfiguracji powinny działać na podstawie **metadanych manifestu/schematu**
  bez wykonywania kodu Pluginu
- natywne zachowanie środowiska uruchomieniowego pochodzi ze ścieżki modułu Pluginu `register(api)`

Ten podział pozwala OpenClaw walidować konfigurację, wyjaśniać brakujące/wyłączone Pluginy i
budować wskazówki dla UI/schematu, zanim pełne środowisko uruchomieniowe stanie się aktywne.

### Pluginy kanałów i współdzielone narzędzie message

Pluginy kanałów nie muszą rejestrować osobnego narzędzia send/edit/react dla
zwykłych działań czatu. OpenClaw utrzymuje jedno współdzielone narzędzie `message` w core,
a Pluginy kanałów odpowiadają za wykrywanie specyficzne dla kanału i wykonanie za nim.

Obecna granica wygląda tak:

- core odpowiada za host współdzielonego narzędzia `message`, połączenie z promptem, prowadzenie księgowości sesji/wątków
  oraz dyspozycję wykonania
- Pluginy kanałów odpowiadają za wykrywanie działań w ograniczonym zakresie, wykrywanie możliwości i wszelkie
  fragmenty schematu specyficzne dla kanału
- Pluginy kanałów odpowiadają za gramatykę rozmów sesji specyficzną dla dostawcy, na przykład
  za sposób, w jaki identyfikatory rozmów kodują identyfikatory wątków lub dziedziczą z rozmów nadrzędnych
- Pluginy kanałów wykonują końcowe działanie przez swój adapter działań

Dla Pluginów kanałów powierzchnią SDK jest
`ChannelMessageActionAdapter.describeMessageTool(...)`. To ujednolicone wywołanie wykrywania
pozwala Pluginowi zwrócić razem widoczne działania, możliwości i wkłady do schematu,
aby te elementy nie rozjeżdżały się względem siebie.

Core przekazuje zakres środowiska uruchomieniowego do tego kroku wykrywania. Ważne pola to:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- zaufane przychodzące `requesterSenderId`

Ma to znaczenie dla Pluginów zależnych od kontekstu. Kanał może ukrywać lub ujawniać
działania wiadomości w zależności od aktywnego konta, bieżącego pokoju/wątku/wiadomości lub
tożsamości zaufanego żądającego, bez zakodowanych na sztywno gałęzi specyficznych dla kanału w
narzędziu `message` w core.

To dlatego zmiany routingu embedded-runner nadal są pracą po stronie Pluginu: runner
odpowiada za przekazywanie bieżącej tożsamości czatu/sesji do granicy wykrywania Pluginu,
aby współdzielone narzędzie `message` udostępniało właściwą powierzchnię należącą do kanału dla bieżącej tury.

W przypadku pomocników wykonania należących do kanału, bundlowane Pluginy powinny utrzymywać środowisko uruchomieniowe wykonania
wewnątrz własnych modułów rozszerzeń. Core nie odpowiada już za środowiska uruchomieniowe działań wiadomości Discord,
Slack, Telegram czy WhatsApp w `src/agents/tools`.
Nie publikujemy oddzielnych podścieżek `plugin-sdk/*-action-runtime`, a bundlowane
Pluginy powinny importować własny lokalny kod środowiska uruchomieniowego bezpośrednio ze swoich
modułów należących do rozszerzenia.

Ta sama granica dotyczy ogólnie powierzchni SDK nazwanych od dostawców: core nie powinien
importować wygodnych barrelów specyficznych dla kanałów dla Slack, Discord, Signal,
WhatsApp ani podobnych rozszerzeń. Jeśli core potrzebuje jakiegoś zachowania, powinien albo użyć
własnego barrela `api.ts` / `runtime-api.ts` bundlowanego Pluginu, albo wynieść tę potrzebę
do wąskiej, ogólnej możliwości we współdzielonym SDK.

W przypadku ankiet istnieją konkretnie dwie ścieżki wykonania:

- `outbound.sendPoll` to współdzielona baza dla kanałów, które pasują do wspólnego
  modelu ankiet
- `actions.handleAction("poll")` to preferowana ścieżka dla semantyki ankiet specyficznej dla kanału
  lub dodatkowych parametrów ankiet

Core odkłada teraz współdzielone parsowanie ankiet do czasu, aż dyspozycja ankiety Pluginu odrzuci
działanie, dzięki czemu obsługujące ankiety należące do Pluginu mogą akceptować pola ankiet specyficzne dla kanału,
nie będąc wcześniej blokowane przez ogólny parser ankiet.

Pełną sekwencję uruchamiania znajdziesz w [Potoku ładowania](#load-pipeline).

## Model własności możliwości

OpenClaw traktuje natywny Plugin jako granicę własności dla **firmy** albo
**funkcji**, a nie jako zbiór niepowiązanych integracji.

Oznacza to, że:

- Plugin firmy powinien zwykle odpowiadać za wszystkie powierzchnie OpenClaw skierowane do tej firmy
- Plugin funkcji powinien zwykle odpowiadać za pełną powierzchnię wprowadzanej przez siebie funkcji
- kanały powinny korzystać ze współdzielonych możliwości core zamiast doraźnie ponownie implementować zachowanie dostawców

Przykłady:

- bundlowany Plugin `openai` odpowiada za zachowanie dostawcy modeli OpenAI oraz za zachowanie OpenAI dotyczące
  mowy + głosu w czasie rzeczywistym + rozumienia mediów + generowania obrazów
- bundlowany Plugin `elevenlabs` odpowiada za zachowanie mowy ElevenLabs
- bundlowany Plugin `microsoft` odpowiada za zachowanie mowy Microsoft
- bundlowany Plugin `google` odpowiada za zachowanie dostawcy modeli Google oraz za Google
  w obszarze rozumienia mediów + generowania obrazów + wyszukiwania w sieci
- bundlowany Plugin `firecrawl` odpowiada za zachowanie Firecrawl dotyczące pobierania z sieci
- bundlowane Pluginy `minimax`, `mistral`, `moonshot` i `zai` odpowiadają za swoje
  backendy rozumienia mediów
- bundlowany Plugin `qwen` odpowiada za zachowanie dostawcy tekstu Qwen oraz
  rozumienie mediów i generowanie wideo
- Plugin `voice-call` jest Pluginem funkcji: odpowiada za transport połączeń, narzędzia,
  CLI, trasy i mostkowanie strumieni mediów Twilio, ale korzysta ze współdzielonych możliwości mowy
  oraz transkrypcji i głosu w czasie rzeczywistym zamiast bezpośrednio importować Pluginy dostawców

Docelowy stan to:

- OpenAI znajduje się w jednym Pluginie, nawet jeśli obejmuje modele tekstowe, mowę, obrazy i
  przyszłe wideo
- inny dostawca może zrobić to samo dla własnego obszaru funkcjonalnego
- kanały nie muszą wiedzieć, który Plugin dostawcy jest właścicielem dostawcy; korzystają ze
  współdzielonego kontraktu możliwości udostępnianego przez core

To jest kluczowe rozróżnienie:

- **plugin** = granica własności
- **capability** = kontrakt core, który wiele Pluginów może implementować lub wykorzystywać

Jeśli więc OpenClaw dodaje nową dziedzinę, taką jak wideo, pierwsze pytanie nie brzmi
„który dostawca powinien mieć na sztywno zakodowaną obsługę wideo?”. Pierwsze pytanie brzmi: „jaki jest
kontrakt możliwości wideo w core?”. Gdy taki kontrakt istnieje, Pluginy dostawców
mogą się względem niego rejestrować, a Pluginy kanałów/funkcji mogą z niego korzystać.

Jeśli możliwość jeszcze nie istnieje, właściwym krokiem jest zwykle:

1. zdefiniowanie brakującej możliwości w core
2. udostępnienie jej przez API/runtime Pluginów w sposób typowany
3. podłączenie kanałów/funkcji do tej możliwości
4. pozwolenie Pluginom dostawców na rejestrowanie implementacji

Dzięki temu własność pozostaje jawna, a jednocześnie unika się zachowania core zależnego od
jednego dostawcy lub jednorazowej ścieżki kodu specyficznej dla jednego Pluginu.

### Warstwowanie możliwości

Używaj tego modelu mentalnego przy podejmowaniu decyzji, gdzie powinien znajdować się kod:

- **warstwa możliwości core**: współdzielona orkiestracja, polityka, fallback, reguły
  scalania konfiguracji, semantyka dostarczania i typowane kontrakty
- **warstwa Pluginu dostawcy**: API specyficzne dla dostawcy, uwierzytelnianie, katalogi modeli, synteza mowy,
  generowanie obrazów, przyszłe backendy wideo, endpointy użycia
- **warstwa Pluginu kanału/funkcji**: integracja Slack/Discord/voice-call/itd.
  korzystająca z możliwości core i prezentująca je na określonej powierzchni

Na przykład TTS ma taki układ:

- core odpowiada za politykę TTS podczas odpowiedzi, kolejność fallbacków, preferencje i dostarczanie do kanałów
- `openai`, `elevenlabs` i `microsoft` odpowiadają za implementacje syntezy
- `voice-call` korzysta z pomocnika runtime TTS dla telefonii

Ten sam wzorzec powinien być preferowany dla przyszłych możliwości.

### Przykład firmowego Pluginu z wieloma możliwościami

Plugin firmy powinien z zewnątrz sprawiać wrażenie spójnego. Jeśli OpenClaw ma współdzielone
kontrakty dla modeli, mowy, transkrypcji w czasie rzeczywistym, głosu w czasie rzeczywistym, rozumienia mediów,
generowania obrazów, generowania wideo, pobierania z sieci i wyszukiwania w sieci,
dostawca może zarządzać wszystkimi swoimi powierzchniami w jednym miejscu:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

Ważne nie są dokładne nazwy pomocników. Liczy się kształt:

- jeden Plugin zarządza powierzchnią dostawcy
- core nadal zarządza kontraktami możliwości
- kanały i Pluginy funkcji korzystają z pomocników `api.runtime.*`, a nie z kodu dostawcy
- testy kontraktowe mogą sprawdzać, że Plugin zarejestrował możliwości, które
  deklaruje jako własne

### Przykład możliwości: rozumienie wideo

OpenClaw już traktuje rozumienie obrazów/audio/wideo jako jedną współdzieloną
możliwość. Ten sam model własności obowiązuje także tutaj:

1. core definiuje kontrakt rozumienia mediów
2. Pluginy dostawców rejestrują odpowiednio `describeImage`, `transcribeAudio` i
   `describeVideo`
3. kanały i Pluginy funkcji korzystają ze współdzielonego zachowania core zamiast
   podłączać się bezpośrednio do kodu dostawcy

Dzięki temu założenia dotyczące wideo jednego dostawcy nie zostają wpisane na stałe do core. Plugin zarządza
powierzchnią dostawcy; core zarządza kontraktem możliwości i zachowaniem fallback.

Generowanie wideo już wykorzystuje tę samą sekwencję: core zarządza typowanym
kontraktem możliwości i pomocnikiem runtime, a Pluginy dostawców rejestrują
implementacje `api.registerVideoGenerationProvider(...)` względem tego kontraktu.

Potrzebujesz konkretnej listy wdrożeniowej? Zobacz
[Capability Cookbook](/pl/plugins/architecture).

## Kontrakty i egzekwowanie

Powierzchnia API Pluginów jest celowo typowana i scentralizowana w
`OpenClawPluginApi`. Ten kontrakt definiuje wspierane punkty rejestracji i
pomocniki runtime, na których Plugin może polegać.

Dlaczego to ma znaczenie:

- autorzy Pluginów otrzymują jeden stabilny wewnętrzny standard
- core może odrzucać zduplikowaną własność, na przykład dwa Pluginy rejestrujące ten sam
  identyfikator dostawcy
- podczas uruchamiania mogą być prezentowane użyteczne diagnostyki dla nieprawidłowej rejestracji
- testy kontraktowe mogą egzekwować własność bundlowanych Pluginów i zapobiegać cichemu dryfowi

Istnieją dwie warstwy egzekwowania:

1. **egzekwowanie rejestracji w runtime**
   Rejestr Pluginów waliduje rejestracje podczas ładowania Pluginów. Przykłady:
   zduplikowane identyfikatory dostawców, zduplikowane identyfikatory dostawców mowy i nieprawidłowe
   rejestracje generują diagnostyki Pluginów zamiast niezdefiniowanego zachowania.
2. **testy kontraktowe**
   Bundlowane Pluginy są przechwytywane w rejestrach kontraktowych podczas uruchamiania testów, aby
   OpenClaw mógł jawnie sprawdzać własność. Dziś jest to używane dla
   dostawców modeli, dostawców mowy, dostawców wyszukiwania w sieci i własności bundlowanej rejestracji.

Praktyczny efekt jest taki, że OpenClaw z góry wie, który Plugin jest właścicielem której
powierzchni. Dzięki temu core i kanały mogą się płynnie składać, ponieważ własność jest
zadeklarowana, typowana i testowalna, a nie domyślna.

### Co powinno należeć do kontraktu

Dobre kontrakty Pluginów są:

- typowane
- małe
- specyficzne dla możliwości
- należące do core
- możliwe do ponownego użycia przez wiele Pluginów
- możliwe do wykorzystania przez kanały/funkcje bez wiedzy o dostawcy

Złe kontrakty Pluginów to:

- polityka specyficzna dla dostawcy ukryta w core
- jednorazowe furtki dla Pluginów, które omijają rejestr
- kod kanału sięgający bezpośrednio do implementacji dostawcy
- doraźne obiekty runtime, które nie są częścią `OpenClawPluginApi` ani
  `api.runtime`

W razie wątpliwości podnieś poziom abstrakcji: najpierw zdefiniuj możliwość, a potem
pozwól Pluginom się do niej podłączać.

## Model wykonania

Natywne Pluginy OpenClaw działają **w tym samym procesie** co Gateway. Nie są
sandboxowane. Załadowany natywny Plugin ma taki sam poziom zaufania na granicy procesu jak
kod core.

Konsekwencje:

- natywny Plugin może rejestrować narzędzia, handlery sieciowe, hooki i usługi
- błąd natywnego Pluginu może spowodować awarię Gateway lub destabilizację
- złośliwy natywny Plugin jest równoważny dowolnemu wykonaniu kodu wewnątrz procesu OpenClaw

Zgodne bundle są domyślnie bezpieczniejsze, ponieważ OpenClaw obecnie traktuje je
jako pakiety metadanych/treści. W obecnych wydaniach oznacza to głównie bundlowane
Skills.

W przypadku Pluginów niebundlowanych używaj allowlist i jawnych ścieżek instalacji/ładowania. Traktuj
Pluginy workspace jako kod czasu programowania, a nie produkcyjne ustawienia domyślne.

W przypadku nazw pakietów bundlowanego workspace zachowuj identyfikator Pluginu zakotwiczony w nazwie npm:
domyślnie `@openclaw/<id>` albo zatwierdzony typowany sufiks, taki jak
`-provider`, `-plugin`, `-speech`, `-sandbox` lub `-media-understanding`, gdy
pakiet celowo udostępnia węższą rolę Pluginu.

Ważna uwaga dotycząca zaufania:

- `plugins.allow` ufa **identyfikatorom Pluginów**, a nie pochodzeniu źródła.
- Plugin workspace o tym samym identyfikatorze co bundlowany Plugin celowo przesłania
  bundlowaną kopię, gdy taki Plugin workspace jest włączony/na allowliście.
- To normalne i przydatne przy lokalnym programowaniu, testowaniu poprawek i hotfixach.

## Granica eksportu

OpenClaw eksportuje możliwości, a nie wygodne implementacje.

Rejestrację możliwości należy utrzymywać jako publiczną. Należy ograniczać eksport pomocników niebędących kontraktami:

- podścieżki pomocnicze specyficzne dla bundlowanych Pluginów
- podścieżki infrastruktury runtime, które nie są przeznaczone jako publiczne API
- wygodne pomocniki specyficzne dla dostawcy
- pomocniki konfiguracji/onboardingu, które są szczegółami implementacyjnymi

Niektóre podścieżki pomocnicze bundlowanych Pluginów nadal pozostają w wygenerowanej mapie eksportów SDK
ze względu na kompatybilność i utrzymanie bundlowanych Pluginów. Obecne przykłady obejmują
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` oraz kilka powierzchni `plugin-sdk/matrix*`. Traktuj je jako
zastrzeżone eksporty będące szczegółami implementacji, a nie jako zalecany wzorzec SDK dla
nowych Pluginów zewnętrznych.

## Potok ładowania

Podczas uruchamiania OpenClaw wykonuje w przybliżeniu to:

1. wykrywa katalogi główne kandydatów na Pluginy
2. odczytuje natywne lub zgodne manifesty bundli oraz metadane pakietów
3. odrzuca niebezpiecznych kandydatów
4. normalizuje konfigurację Pluginów (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decyduje o włączeniu dla każdego kandydata
6. ładuje włączone natywne moduły przez jiti
7. wywołuje natywne hooki `register(api)` (lub `activate(api)` — starszy alias) i zbiera rejestracje do rejestru Pluginów
8. udostępnia rejestr powierzchniom poleceń/runtime

<Note>
`activate` jest starszym aliasem dla `register` — loader rozwiązuje to, co jest obecne (`def.register ?? def.activate`), i wywołuje to w tym samym miejscu. Wszystkie bundlowane Pluginy używają `register`; dla nowych Pluginów preferuj `register`.
</Note>

Bramki bezpieczeństwa działają **przed** wykonaniem runtime. Kandydaci są blokowani,
gdy punkt wejścia wychodzi poza katalog główny Pluginu, ścieżka jest zapisywalna globalnie albo
własność ścieżki wygląda podejrzanie dla Pluginów niebundlowanych.

### Zachowanie oparte na manifeście

Manifest jest źródłem prawdy płaszczyzny sterowania. OpenClaw używa go do:

- identyfikacji Pluginu
- wykrywania zadeklarowanych kanałów/Skills/schematu konfiguracji lub możliwości bundla
- walidacji `plugins.entries.<id>.config`
- wzbogacania etykiet/placeholderów w Control UI
- wyświetlania metadanych instalacji/katalogu
- zachowania taniej aktywacji i deskryptorów konfiguracji bez ładowania runtime Pluginu

W przypadku natywnych Pluginów moduł runtime jest częścią płaszczyzny danych. Rejestruje
rzeczywiste zachowanie, takie jak hooki, narzędzia, polecenia czy przepływy dostawców.

Opcjonalne bloki manifestu `activation` i `setup` pozostają w płaszczyźnie sterowania.
Są to deskryptory wyłącznie z metadanymi dla planowania aktywacji i wykrywania konfiguracji;
nie zastępują rejestracji runtime, `register(...)` ani `setupEntry`.
Pierwsi aktywni konsumenci aktywacji używają teraz wskazówek manifestu dotyczących poleceń, kanałów i dostawców,
aby zawężać ładowanie Pluginów przed szerszą materializacją rejestru:

- ładowanie CLI zawęża się do Pluginów, które są właścicielami żądanego polecenia głównego
- rozwiązywanie konfiguracji kanału/Pluginu zawęża się do Pluginów, które są właścicielami żądanego
  identyfikatora kanału
- jawne rozwiązywanie konfiguracji/runtime dostawcy zawęża się do Pluginów, które są właścicielami
  żądanego identyfikatora dostawcy

Wykrywanie konfiguracji preferuje teraz identyfikatory należące do deskryptorów, takie jak `setup.providers` i
`setup.cliBackends`, aby zawężać kandydatów na Pluginy, zanim nastąpi fallback do
`setup-api` dla Pluginów, które nadal potrzebują hooków runtime na etapie konfiguracji. Jeśli więcej niż
jeden wykryty Plugin deklaruje ten sam znormalizowany identyfikator dostawcy konfiguracji lub backendu CLI,
wyszukiwanie konfiguracji odmawia wyboru niejednoznacznego właściciela zamiast polegać na kolejności wykrycia.

### Co loader przechowuje w cache

OpenClaw utrzymuje krótkotrwałe cache w procesie dla:

- wyników wykrywania
- danych rejestru manifestów
- załadowanych rejestrów Pluginów

Te cache zmniejszają skokowy koszt uruchamiania i powtarzanych poleceń. Można o nich bezpiecznie myśleć
jako o krótkotrwałych cache wydajnościowych, a nie trwałym przechowywaniu.

Uwaga dotycząca wydajności:

- Ustaw `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` lub
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1`, aby wyłączyć te cache.
- Dostosuj okna cache przez `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` i
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Model rejestru

Załadowane Pluginy nie mutują bezpośrednio przypadkowych globali core. Rejestrują się w
centralnym rejestrze Pluginów.

Rejestr śledzi:

- rekordy Pluginów (tożsamość, źródło, pochodzenie, status, diagnostyka)
- narzędzia
- starsze hooki i hooki typowane
- kanały
- dostawcy
- handlery Gateway RPC
- trasy HTTP
- rejestratory CLI
- usługi działające w tle
- polecenia należące do Pluginów

Funkcje core odczytują następnie z tego rejestru zamiast komunikować się
bezpośrednio z modułami Pluginów. Dzięki temu ładowanie pozostaje jednokierunkowe:

- moduł Pluginu -> rejestracja w rejestrze
- runtime core -> korzystanie z rejestru

To rozdzielenie ma znaczenie dla utrzymywalności. Oznacza, że większość powierzchni core potrzebuje
tylko jednego punktu integracji: „odczytaj rejestr”, a nie „dodaj specjalny przypadek dla każdego modułu Pluginu”.

## Callbacki wiązania rozmowy

Pluginy, które wiążą rozmowę, mogą reagować, gdy zatwierdzenie zostanie rozstrzygnięte.

Użyj `api.onConversationBindingResolved(...)`, aby otrzymać callback po zatwierdzeniu lub odrzuceniu
żądania wiązania:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Pola ładunku callbacku:

- `status`: `"approved"` lub `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` lub `"deny"`
- `binding`: rozstrzygnięte powiązanie dla zatwierdzonych żądań
- `request`: podsumowanie pierwotnego żądania, wskazówka odłączenia, identyfikator nadawcy oraz
  metadane rozmowy

Ten callback służy wyłącznie do powiadamiania. Nie zmienia tego, kto może wiązać
rozmowę, i uruchamia się po zakończeniu obsługi zatwierdzenia przez core.

## Hooki runtime dostawców

Pluginy dostawców mają teraz dwie warstwy:

- metadane manifestu: `providerAuthEnvVars` do taniego wyszukiwania uwierzytelniania dostawcy przez zmienne środowiskowe
  przed załadowaniem runtime, `providerAuthAliases` dla wariantów dostawcy współdzielących
  uwierzytelnianie, `channelEnvVars` do taniego wyszukiwania środowiska/konfiguracji kanału przed
  załadowaniem runtime, oraz `providerAuthChoices` do tanich etykiet onboardingu/wyboru uwierzytelniania i
  metadanych flag CLI przed załadowaniem runtime
- hooki czasu konfiguracji: `catalog` / starsze `discovery` oraz `applyConfigDefaults`
- hooki runtime: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw nadal odpowiada za ogólną pętlę agenta, failover, obsługę transkryptu i
politykę narzędzi. Te hooki są powierzchnią rozszerzeń dla zachowania specyficznego dla dostawcy bez
potrzeby tworzenia całego niestandardowego transportu wnioskowania.

Używaj manifestu `providerAuthEnvVars`, gdy dostawca ma poświadczenia oparte na zmiennych środowiskowych,
które ogólne ścieżki uwierzytelniania/statusu/wyboru modelu powinny widzieć bez ładowania runtime Pluginu.
Używaj manifestu `providerAuthAliases`, gdy jeden identyfikator dostawcy powinien ponownie używać
zmiennych środowiskowych, profili uwierzytelniania, uwierzytelniania opartego na konfiguracji i opcji onboardingu z kluczem API
innego identyfikatora dostawcy. Używaj manifestu `providerAuthChoices`, gdy powierzchnie CLI onboardingu/wyboru uwierzytelniania
powinny znać identyfikator opcji dostawcy, etykiety grup i prostą obsługę uwierzytelniania jedną flagą
bez ładowania runtime dostawcy. Zachowaj runtime dostawcy `envVars` dla wskazówek skierowanych do operatora,
takich jak etykiety onboardingu lub zmienne konfiguracji OAuth `client-id`/`client-secret`.

Używaj manifestu `channelEnvVars`, gdy kanał ma uwierzytelnianie lub konfigurację sterowane przez zmienne środowiskowe, które
ogólny fallback do zmiennych środowiskowych powłoki, sprawdzanie konfiguracji/statusu lub prompty konfiguracji powinny widzieć
bez ładowania runtime kanału.

### Kolejność hooków i ich użycie

Dla Pluginów modeli/dostawców OpenClaw wywołuje hooki mniej więcej w tej kolejności.
Kolumna „Kiedy używać” to szybki przewodnik decyzyjny.

| #   | Hook                              | Co robi                                                                                                        | Kiedy używać                                                                                                                               |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publikuje konfigurację dostawcy do `models.providers` podczas generowania `models.json`                       | Dostawca zarządza katalogiem lub domyślnymi wartościami `baseUrl`                                                                         |
| 2   | `applyConfigDefaults`             | Stosuje globalne domyślne ustawienia konfiguracji należące do dostawcy podczas materializacji konfiguracji    | Wartości domyślne zależą od trybu uwierzytelniania, zmiennych środowiskowych lub semantyki rodziny modeli dostawcy                      |
| --  | _(built-in model lookup)_         | OpenClaw najpierw próbuje normalnej ścieżki rejestru/katalogu                                                  | _(to nie jest hook Pluginu)_                                                                                                              |
| 3   | `normalizeModelId`                | Normalizuje starsze aliasy lub aliasy identyfikatorów modeli w wersji preview przed wyszukaniem               | Dostawca zarządza porządkowaniem aliasów przed rozwiązaniem kanonicznego modelu                                                          |
| 4   | `normalizeTransport`              | Normalizuje `api` / `baseUrl` dla rodziny dostawców przed ogólnym składaniem modelu                           | Dostawca zarządza porządkowaniem transportu dla niestandardowych identyfikatorów dostawców w tej samej rodzinie transportu              |
| 5   | `normalizeConfig`                 | Normalizuje `models.providers.<id>` przed rozwiązaniem runtime/dostawcy                                        | Dostawca potrzebuje porządkowania konfiguracji, które powinno znajdować się razem z Pluginem; bundlowane pomocniki rodziny Google dodatkowo zabezpieczają wspierane wpisy konfiguracji Google |
| 6   | `applyNativeStreamingUsageCompat` | Stosuje przepisy zgodności natywnego użycia streamingu do dostawców konfiguracji                              | Dostawca potrzebuje poprawek metadanych natywnego użycia streamingu zależnych od endpointu                                               |
| 7   | `resolveConfigApiKey`             | Rozwiązuje uwierzytelnianie markerem środowiskowym dla dostawców konfiguracji przed załadowaniem uwierzytelniania runtime | Dostawca ma własne rozwiązywanie klucza API markera środowiskowego; `amazon-bedrock` ma tu również wbudowany resolver markera środowiskowego AWS |
| 8   | `resolveSyntheticAuth`            | Ujawnia lokalne/samohostowane lub oparte na konfiguracji uwierzytelnianie bez utrwalania jawnego tekstu       | Dostawca może działać z syntetycznym/lokalnym markerem poświadczeń                                                                       |
| 9   | `resolveExternalAuthProfiles`     | Nakłada zewnętrzne profile uwierzytelniania należące do dostawcy; domyślne `persistence` to `runtime-only` dla poświadczeń należących do CLI/aplikacji | Dostawca ponownie używa poświadczeń zewnętrznego uwierzytelniania bez utrwalania skopiowanych tokenów odświeżania                       |
| 10  | `shouldDeferSyntheticProfileAuth` | Obniża priorytet zapisanych syntetycznych placeholderów profili względem uwierzytelniania opartego na zmiennych środowiskowych/konfiguracji | Dostawca przechowuje syntetyczne placeholdery profili, które nie powinny mieć pierwszeństwa                                              |
| 11  | `resolveDynamicModel`             | Synchroniczny fallback dla identyfikatorów modeli należących do dostawcy, których nie ma jeszcze w lokalnym rejestrze | Dostawca akceptuje dowolne identyfikatory modeli upstream                                                                                 |
| 12  | `prepareDynamicModel`             | Asynchroniczne rozgrzanie, po którym `resolveDynamicModel` uruchamia się ponownie                              | Dostawca potrzebuje metadanych sieciowych przed rozwiązaniem nieznanych identyfikatorów                                                  |
| 13  | `normalizeResolvedModel`          | Ostateczne przepisanie przed użyciem rozwiązanego modelu przez embedded runner                                | Dostawca potrzebuje przepisania transportu, ale nadal używa transportu core                                                              |
| 14  | `contributeResolvedModelCompat`   | Dostarcza flagi zgodności dla modeli dostawcy za innym zgodnym transportem                                     | Dostawca rozpoznaje własne modele na transportach proxy bez przejmowania roli dostawcy                                                   |
| 15  | `capabilities`                    | Metadane transkryptu/narzędzi należące do dostawcy, używane przez współdzieloną logikę core                   | Dostawca potrzebuje niestandardowych zachowań transkryptu/rodziny dostawcy                                                               |
| 16  | `normalizeToolSchemas`            | Normalizuje schematy narzędzi, zanim zobaczy je embedded runner                                                | Dostawca potrzebuje porządkowania schematów dla rodziny transportu                                                                        |
| 17  | `inspectToolSchemas`              | Ujawnia diagnostykę schematów należącą do dostawcy po normalizacji                                             | Dostawca chce ostrzeżeń o słowach kluczowych bez uczenia core reguł specyficznych dla dostawcy                                          |
| 18  | `resolveReasoningOutputMode`      | Wybiera natywny lub tagowany kontrakt wyjścia rozumowania                                                      | Dostawca potrzebuje tagowanego wyjścia rozumowania/końcowego zamiast natywnych pól                                                       |
| 19  | `prepareExtraParams`              | Normalizacja parametrów żądania przed ogólnymi wrapperami opcji strumienia                                     | Dostawca potrzebuje domyślnych parametrów żądania lub porządkowania parametrów specyficznych dla dostawcy                               |
| 20  | `createStreamFn`                  | Całkowicie zastępuje normalną ścieżkę strumienia niestandardowym transportem                                   | Dostawca potrzebuje niestandardowego protokołu przewodowego, a nie tylko wrappera                                                        |
| 21  | `wrapStreamFn`                    | Wrapper strumienia po zastosowaniu ogólnych wrapperów                                                          | Dostawca potrzebuje wrapperów zgodności nagłówków/treści żądania/modelu bez niestandardowego transportu                                 |
| 22  | `resolveTransportTurnState`       | Dołącza natywne nagłówki transportu lub metadane dla każdej tury                                               | Dostawca chce, aby ogólne transporty wysyłały natywną tożsamość tury dostawcy                                                            |
| 23  | `resolveWebSocketSessionPolicy`   | Dołącza natywne nagłówki WebSocket lub politykę schładzania sesji                                              | Dostawca chce, aby ogólne transporty WS dostrajały nagłówki sesji lub politykę fallback                                                  |
| 24  | `formatApiKey`                    | Formater profilu uwierzytelniania: zapisany profil staje się łańcuchem `apiKey` dla runtime                   | Dostawca przechowuje dodatkowe metadane uwierzytelniania i potrzebuje niestandardowego kształtu tokenu runtime                          |
| 25  | `refreshOAuth`                    | Nadpisanie odświeżania OAuth dla niestandardowych endpointów odświeżania lub polityki błędów odświeżania      | Dostawca nie pasuje do współdzielonych mechanizmów odświeżania `pi-ai`                                                                    |
| 26  | `buildAuthDoctorHint`             | Wskazówka naprawcza dołączana, gdy odświeżanie OAuth się nie powiedzie                                         | Dostawca potrzebuje własnych wskazówek naprawy uwierzytelniania po błędzie odświeżania                                                   |
| 27  | `matchesContextOverflowError`     | Matcher przepełnienia okna kontekstowego należący do dostawcy                                                  | Dostawca ma surowe błędy przepełnienia, których ogólne heurystyki nie wykryją                                                            |
| 28  | `classifyFailoverReason`          | Klasyfikacja przyczyny failover należąca do dostawcy                                                           | Dostawca może mapować surowe błędy API/transportu na rate-limit/overload/itd.                                                            |
| 29  | `isCacheTtlEligible`              | Polityka cache promptów dla dostawców proxy/backhaul                                                           | Dostawca potrzebuje bramkowania TTL cache specyficznego dla proxy                                                                         |
| 30  | `buildMissingAuthMessage`         | Zastępuje ogólny komunikat odzyskiwania po braku uwierzytelnienia                                              | Dostawca potrzebuje własnej wskazówki odzyskiwania po braku uwierzytelnienia                                                             |
| 31  | `suppressBuiltInModel`            | Tłumienie nieaktualnych modeli upstream oraz opcjonalna wskazówka błędu dla użytkownika                        | Dostawca musi ukryć nieaktualne wiersze upstream lub zastąpić je wskazówką dostawcy                                                      |
| 32  | `augmentModelCatalog`             | Syntetyczne/końcowe wiersze katalogu dołączane po wykrywaniu                                                   | Dostawca potrzebuje syntetycznych wierszy zgodności przyszłej w `models list` i selektorach                                              |
| 33  | `isBinaryThinking`                | Przełącznik rozumowania w trybie włącz/wyłącz dla dostawców z binarnym myśleniem                               | Dostawca udostępnia wyłącznie binarne myślenie włącz/wyłącz                                                                               |
| 34  | `supportsXHighThinking`           | Obsługa rozumowania `xhigh` dla wybranych modeli                                                               | Dostawca chce `xhigh` tylko dla podzbioru modeli                                                                                          |
| 35  | `resolveDefaultThinkingLevel`     | Domyślny poziom `/think` dla określonej rodziny modeli                                                         | Dostawca zarządza domyślną polityką `/think` dla rodziny modeli                                                                           |
| 36  | `isModernModelRef`                | Matcher nowoczesnych modeli dla filtrów profili live i wyboru smoke                                            | Dostawca zarządza dopasowaniem preferowanych modeli dla live/smoke                                                                        |
| 37  | `prepareRuntimeAuth`              | Zamienia skonfigurowane poświadczenie na rzeczywisty token/klucz runtime tuż przed wnioskowaniem              | Dostawca potrzebuje wymiany tokenu lub krótkotrwałego poświadczenia żądania                                                               |
| 38  | `resolveUsageAuth`                | Rozwiązuje poświadczenia użycia/rozliczeń dla `/usage` i powiązanych powierzchni statusu                      | Dostawca potrzebuje niestandardowego parsowania tokenu użycia/limitu lub innych poświadczeń użycia                                       |
| 39  | `fetchUsageSnapshot`              | Pobiera i normalizuje snapshoty użycia/limitów specyficzne dla dostawcy po rozwiązaniu uwierzytelnienia      | Dostawca potrzebuje własnego endpointu użycia lub parsera ładunku                                                                         |
| 40  | `createEmbeddingProvider`         | Buduje adapter embeddingów należący do dostawcy dla pamięci/wyszukiwania                                      | Zachowanie embeddingów pamięci powinno należeć do Pluginu dostawcy                                                                        |
| 41  | `buildReplayPolicy`               | Zwraca politykę replay sterującą obsługą transkryptu dla dostawcy                                             | Dostawca potrzebuje niestandardowej polityki transkryptu (na przykład usuwania bloków myślenia)                                          |
| 42  | `sanitizeReplayHistory`           | Przepisuje historię replay po ogólnym czyszczeniu transkryptu                                                 | Dostawca potrzebuje przepisów replay specyficznych dla dostawcy wykraczających poza współdzielone pomocniki Compaction                   |
| 43  | `validateReplayTurns`             | Ostateczna walidacja lub przekształcenie tur replay przed embedded runnerem                                   | Transport dostawcy wymaga bardziej rygorystycznej walidacji tur po ogólnej sanitacji                                                     |
| 44  | `onModelSelected`                 | Uruchamia skutki uboczne po wyborze modelu należące do dostawcy                                               | Dostawca potrzebuje telemetrii lub stanu należącego do dostawcy, gdy model staje się aktywny                                             |

`normalizeModelId`, `normalizeTransport` i `normalizeConfig` najpierw sprawdzają
dopasowany Plugin dostawcy, a następnie przechodzą do innych Pluginów dostawców obsługujących hooki,
dopóki któryś faktycznie nie zmieni identyfikatora modelu albo transportu/konfiguracji. Dzięki temu
shimy aliasów/zgodności dostawców nadal działają bez konieczności, by wywołujący wiedział,
który bundlowany Plugin jest właścicielem danego przepisania. Jeśli żaden hook dostawcy nie przepisze
wspieranego wpisu konfiguracji z rodziny Google, bundlowany normalizator konfiguracji Google nadal zastosuje
to porządkowanie zgodności.

Jeśli dostawca potrzebuje w pełni niestandardowego protokołu przewodowego lub niestandardowego wykonawcy żądań,
to jest to inna klasa rozszerzenia. Te hooki służą do zachowania dostawcy, które nadal działa
na normalnej pętli wnioskowania OpenClaw.

### Przykład dostawcy

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Wbudowane przykłady

- Anthropic używa `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`
  oraz `wrapStreamFn`, ponieważ odpowiada za zgodność w przód Claude 4.6,
  wskazówki dotyczące rodziny dostawcy, wskazówki naprawy uwierzytelniania, integrację
  endpointu użycia, kwalifikowalność cache promptów, wartości domyślne konfiguracji zależne od uwierzytelniania, domyślną/adaptacyjną
  politykę myślenia Claude oraz kształtowanie strumienia specyficzne dla Anthropic dla
  nagłówków beta, `/fast` / `serviceTier` i `context1m`.
- Pomocniki strumienia specyficzne dla Claude w Anthropic pozostają na razie we
  własnej publicznej powierzchni `api.ts` / `contract-api.ts` bundlowanego Pluginu. Ta powierzchnia pakietu
  eksportuje `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` oraz bardziej niskopoziomowe
  konstruktory wrapperów Anthropic zamiast rozszerzać ogólne SDK wokół reguł nagłówków beta jednego
  dostawcy.
- OpenAI używa `resolveDynamicModel`, `normalizeResolvedModel` i
  `capabilities` oraz `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` i `isModernModelRef`,
  ponieważ odpowiada za zgodność w przód GPT-5.4, bezpośrednią normalizację OpenAI
  `openai-completions` -> `openai-responses`, wskazówki uwierzytelniania świadome Codex,
  tłumienie Spark, syntetyczne wiersze listy OpenAI oraz politykę myślenia GPT-5 /
  modeli live; rodzina strumienia `openai-responses-defaults` odpowiada za
  współdzielone natywne wrappery OpenAI Responses dla nagłówków atrybucji,
  `/fast`/`serviceTier`, szczegółowości tekstu, natywnego web search Codex,
  kształtowania ładunku zgodności rozumowania oraz zarządzania kontekstem Responses.
- OpenRouter używa `catalog` oraz `resolveDynamicModel` i
  `prepareDynamicModel`, ponieważ dostawca jest pass-through i może udostępniać nowe
  identyfikatory modeli, zanim statyczny katalog OpenClaw zostanie zaktualizowany; używa też
  `capabilities`, `wrapStreamFn` i `isCacheTtlEligible`, aby zachować
  nagłówki żądań specyficzne dla dostawcy, metadane routingu, poprawki rozumowania i
  politykę cache promptów poza core. Jego polityka replay pochodzi z rodziny
  `passthrough-gemini`, natomiast rodzina strumienia `openrouter-thinking`
  odpowiada za wstrzykiwanie rozumowania proxy i pomijanie nieobsługiwanych modeli / `auto`.
- GitHub Copilot używa `catalog`, `auth`, `resolveDynamicModel` i
  `capabilities` oraz `prepareRuntimeAuth` i `fetchUsageSnapshot`,
  ponieważ potrzebuje logowania urządzenia należącego do dostawcy, zachowania fallback modeli,
  niestandardowych zachowań transkryptu Claude, wymiany tokenu GitHub -> token Copilot oraz
  endpointu użycia należącego do dostawcy.
- OpenAI Codex używa `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` i `augmentModelCatalog` oraz
  `prepareExtraParams`, `resolveUsageAuth` i `fetchUsageSnapshot`, ponieważ nadal
  działa na transportach OpenAI core, ale odpowiada za własną normalizację
  transportu/`baseUrl`, politykę fallback odświeżania OAuth, domyślny wybór transportu,
  syntetyczne wiersze katalogu Codex oraz integrację endpointu użycia ChatGPT; współdzieli
  tę samą rodzinę strumienia `openai-responses-defaults`, co bezpośrednie OpenAI.
- Google AI Studio i Gemini CLI OAuth używają `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` i `isModernModelRef`, ponieważ
  rodzina replay `google-gemini` odpowiada za fallback zgodności w przód Gemini 3.1,
  natywną walidację replay Gemini, sanitację bootstrap replay, tagowany
  tryb wyjścia rozumowania i dopasowywanie nowoczesnych modeli, podczas gdy
  rodzina strumienia `google-thinking` odpowiada za normalizację ładunku myślenia Gemini;
  Gemini CLI OAuth używa też `formatApiKey`, `resolveUsageAuth` i
  `fetchUsageSnapshot` do formatowania tokenu, parsowania tokenu i podłączenia
  endpointu limitu.
- Anthropic Vertex używa `buildReplayPolicy` przez rodzinę replay
  `anthropic-by-model`, dzięki czemu porządkowanie replay specyficzne dla Claude pozostaje
  ograniczone do identyfikatorów Claude, a nie każdego transportu `anthropic-messages`.
- Amazon Bedrock używa `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` i `resolveDefaultThinkingLevel`, ponieważ odpowiada za
  klasyfikację błędów throttle/not-ready/context-overflow specyficzną dla Bedrock
  dla ruchu Anthropic-on-Bedrock; jego polityka replay nadal współdzieli tę samą
  ochronę `anthropic-by-model` tylko dla Claude.
- OpenRouter, Kilocode, Opencode i Opencode Go używają `buildReplayPolicy`
  przez rodzinę replay `passthrough-gemini`, ponieważ pośredniczą dla modeli Gemini
  przez transporty zgodne z OpenAI i potrzebują sanitacji
  thought-signature Gemini bez natywnej walidacji replay Gemini ani
  przepisów bootstrap.
- MiniMax używa `buildReplayPolicy` przez rodzinę replay
  `hybrid-anthropic-openai`, ponieważ jeden dostawca odpowiada zarówno za semantykę
  wiadomości Anthropic, jak i zgodność z OpenAI; zachowuje odrzucanie bloków myślenia
  tylko dla Claude po stronie Anthropic, jednocześnie nadpisując tryb wyjścia rozumowania z powrotem na natywny,
  a rodzina strumienia `minimax-fast-mode` odpowiada za przepisywanie modeli w trybie fast
  na współdzielonej ścieżce strumienia.
- Moonshot używa `catalog` oraz `wrapStreamFn`, ponieważ nadal korzysta ze współdzielonego
  transportu OpenAI, ale potrzebuje normalizacji ładunku myślenia należącej do dostawcy; rodzina strumienia
  `moonshot-thinking` mapuje konfigurację oraz stan `/think` na swój
  natywny binarny ładunek myślenia.
- Kilocode używa `catalog`, `capabilities`, `wrapStreamFn` i
  `isCacheTtlEligible`, ponieważ potrzebuje nagłówków żądań należących do dostawcy,
  normalizacji ładunku rozumowania, wskazówek transkryptu Gemini i bramkowania
  TTL cache Anthropic; rodzina strumienia `kilocode-thinking` utrzymuje wstrzykiwanie myślenia Kilo
  na współdzielonej ścieżce strumienia proxy, jednocześnie pomijając `kilo/auto` i
  inne identyfikatory modeli proxy, które nie obsługują jawnych ładunków rozumowania.
- Z.AI używa `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` i `fetchUsageSnapshot`, ponieważ odpowiada za fallback GLM-5,
  domyślne `tool_stream`, UX binarnego myślenia, dopasowywanie nowoczesnych modeli oraz oba
  elementy: uwierzytelnianie użycia + pobieranie limitu; rodzina strumienia `tool-stream-default-on`
  utrzymuje wrapper `tool_stream` domyślnie włączony poza ręcznie pisanym klejem per dostawca.
- xAI używa `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` i `isModernModelRef`,
  ponieważ odpowiada za normalizację natywnego transportu xAI Responses, przepisywanie
  aliasów Grok w trybie fast, domyślne `tool_stream`, porządkowanie strict-tool /
  ładunku rozumowania, ponowne użycie fallback uwierzytelniania dla narzędzi należących do Pluginu,
  rozwiązywanie modeli Grok zgodne w przód oraz poprawki zgodności należące do dostawcy, takie jak profil
  schematu narzędzi xAI, nieobsługiwane słowa kluczowe schematu, natywne `web_search` oraz dekodowanie
  argumentów wywołań narzędzi z encji HTML.
- Mistral, OpenCode Zen i OpenCode Go używają wyłącznie `capabilities`, aby utrzymać
  niestandardowe zachowania transkryptu/narzędzi poza core.
- Bundlowani dostawcy tylko katalogowi, tacy jak `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` i `volcengine`, używają
  wyłącznie `catalog`.
- Qwen używa `catalog` dla swojego dostawcy tekstu oraz współdzielonych rejestracji rozumienia mediów i generowania wideo
  dla swoich powierzchni multimodalnych.
- MiniMax i Xiaomi używają `catalog` oraz hooków użycia, ponieważ ich zachowanie `/usage`
  należy do Pluginu, mimo że wnioskowanie nadal działa przez współdzielone transporty.

## Pomocniki runtime

Pluginy mogą uzyskiwać dostęp do wybranych pomocników core przez `api.runtime`. Dla TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Uwagi:

- `textToSpeech` zwraca normalny ładunek wyjściowy TTS core dla powierzchni plików/notatek głosowych.
- Używa konfiguracji core `messages.tts` i wyboru dostawcy.
- Zwraca bufor audio PCM + częstotliwość próbkowania. Pluginy muszą ponownie próbkować/kodować dla dostawców.
- `listVoices` jest opcjonalne dla każdego dostawcy. Używaj go dla selektorów głosów należących do dostawcy lub przepływów konfiguracji.
- Listy głosów mogą zawierać bogatsze metadane, takie jak lokalizacja, płeć i tagi osobowości dla selektorów świadomych dostawcy.
- OpenAI i ElevenLabs obsługują dziś telefonię. Microsoft nie.

Pluginy mogą także rejestrować dostawców mowy przez `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Uwagi:

- Politykę TTS, fallback i dostarczanie odpowiedzi należy utrzymywać w core.
- Dostawców mowy należy używać dla zachowania syntezy należącego do dostawcy.
- Starsze wejście Microsoft `edge` jest normalizowane do identyfikatora dostawcy `microsoft`.
- Preferowany model własności jest zorientowany na firmę: jeden Plugin dostawcy może zarządzać
  tekstem, mową, obrazem i przyszłymi dostawcami mediów, gdy OpenClaw dodaje te
  kontrakty możliwości.

W przypadku rozumienia obrazów/audio/wideo Pluginy rejestrują jednego typowanego
dostawcę rozumienia mediów zamiast ogólnego worka klucz/wartość:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Uwagi:

- Orkiestrację, fallback, konfigurację i połączenie z kanałami należy utrzymywać w core.
- Zachowanie dostawcy należy utrzymywać w Pluginie dostawcy.
- Rozszerzanie addytywne powinno pozostać typowane: nowe opcjonalne metody, nowe opcjonalne
  pola wyników, nowe opcjonalne możliwości.
- Generowanie wideo już stosuje ten sam wzorzec:
  - core odpowiada za kontrakt możliwości i pomocnik runtime
  - Pluginy dostawców rejestrują `api.registerVideoGenerationProvider(...)`
  - Pluginy funkcji/kanałów korzystają z `api.runtime.videoGeneration.*`

W przypadku pomocników runtime rozumienia mediów Pluginy mogą wywoływać:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

W przypadku transkrypcji audio Pluginy mogą używać albo runtime
rozumienia mediów, albo starszego aliasu STT:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Uwagi:

- `api.runtime.mediaUnderstanding.*` to preferowana współdzielona powierzchnia dla
  rozumienia obrazów/audio/wideo.
- Używa konfiguracji audio rozumienia mediów core (`tools.media.audio`) oraz kolejności fallbacków dostawców.
- Zwraca `{ text: undefined }`, gdy nie powstanie wynik transkrypcji (na przykład dla pominiętego/nieobsługiwanego wejścia).
- `api.runtime.stt.transcribeAudioFile(...)` pozostaje aliasem zgodności.

Pluginy mogą także uruchamiać subagentów działających w tle przez `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Uwagi:

- `provider` i `model` to opcjonalne nadpisania dla danego uruchomienia, a nie trwałe zmiany sesji.
- OpenClaw uwzględnia te pola nadpisania tylko dla zaufanych wywołujących.
- W przypadku uruchomień fallback należących do Pluginu operatorzy muszą wyrazić zgodę przez `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Użyj `plugins.entries.<id>.subagent.allowedModels`, aby ograniczyć zaufane Pluginy do określonych kanonicznych celów `provider/model`, albo `"*"`, aby jawnie zezwolić na dowolny cel.
- Uruchomienia subagentów z niezaufanych Pluginów nadal działają, ale żądania nadpisania są odrzucane zamiast po cichu przechodzić do fallbacku.

W przypadku web search Pluginy mogą korzystać ze współdzielonego pomocnika runtime zamiast
sięgać do połączenia narzędzia agenta:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Pluginy mogą też rejestrować dostawców web search przez
`api.registerWebSearchProvider(...)`.

Uwagi:

- Wybór dostawcy, rozwiązywanie poświadczeń i współdzieloną semantykę żądań należy utrzymywać w core.
- Dostawców web search należy używać dla transportów wyszukiwania specyficznych dla dostawcy.
- `api.runtime.webSearch.*` to preferowana współdzielona powierzchnia dla Pluginów funkcji/kanałów, które potrzebują zachowania wyszukiwania bez zależności od wrappera narzędzia agenta.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: generuje obraz przy użyciu skonfigurowanego łańcucha dostawców generowania obrazów.
- `listProviders(...)`: wyświetla dostępnych dostawców generowania obrazów i ich możliwości.

## Trasy HTTP Gateway

Pluginy mogą udostępniać endpointy HTTP przez `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Pola trasy:

- `path`: ścieżka trasy pod serwerem HTTP Gateway.
- `auth`: wymagane. Użyj `"gateway"`, aby wymagać zwykłego uwierzytelniania Gateway, albo `"plugin"` dla uwierzytelniania zarządzanego przez Plugin / weryfikacji Webhooków.
- `match`: opcjonalne. `"exact"` (domyślnie) albo `"prefix"`.
- `replaceExisting`: opcjonalne. Pozwala temu samemu Pluginowi zastąpić własną istniejącą rejestrację trasy.
- `handler`: zwraca `true`, gdy trasa obsłużyła żądanie.

Uwagi:

- `api.registerHttpHandler(...)` zostało usunięte i spowoduje błąd ładowania Pluginu. Zamiast tego używaj `api.registerHttpRoute(...)`.
- Trasy Pluginów muszą jawnie deklarować `auth`.
- Konflikty dokładnego `path + match` są odrzucane, chyba że ustawiono `replaceExisting: true`, i jeden Plugin nie może zastąpić trasy innego Pluginu.
- Nakładające się trasy z różnymi poziomami `auth` są odrzucane. Łańcuchy przejścia `exact`/`prefix` należy utrzymywać tylko na tym samym poziomie uwierzytelniania.
- Trasy `auth: "plugin"` **nie** otrzymują automatycznie zakresów runtime operatora. Są przeznaczone do Webhooków / weryfikacji podpisów zarządzanych przez Plugin, a nie do uprzywilejowanych wywołań pomocników Gateway.
- Trasy `auth: "gateway"` działają wewnątrz zakresu runtime żądania Gateway, ale ten zakres jest celowo konserwatywny:
  - uwierzytelnianie bearer oparte na współdzielonym sekrecie (`gateway.auth.mode = "token"` / `"password"`) utrzymuje zakresy runtime tras Pluginów przypięte do `operator.write`, nawet jeśli wywołujący wysyła `x-openclaw-scopes`
  - zaufane tryby HTTP przenoszące tożsamość (na przykład `trusted-proxy` albo `gateway.auth.mode = "none"` na prywatnym wejściu) honorują `x-openclaw-scopes` tylko wtedy, gdy nagłówek jest jawnie obecny
  - jeśli `x-openclaw-scopes` jest nieobecny w takich żądaniach tras Pluginów przenoszących tożsamość, zakres runtime wraca do `operator.write`
- Praktyczna zasada: nie zakładaj, że trasa Pluginu z uwierzytelnianiem Gateway jest domyślnie powierzchnią administratora. Jeśli Twoja trasa potrzebuje zachowania dostępnego wyłącznie dla administratora, wymagaj trybu uwierzytelniania przenoszącego tożsamość i udokumentuj jawny kontrakt nagłówka `x-openclaw-scopes`.

## Ścieżki importu Plugin SDK

Tworząc Pluginy, używaj podścieżek SDK zamiast monolitycznego importu `openclaw/plugin-sdk`:

- `openclaw/plugin-sdk/plugin-entry` dla prymitywów rejestracji Pluginów.
- `openclaw/plugin-sdk/core` dla ogólnego współdzielonego kontraktu skierowanego do Pluginów.
- `openclaw/plugin-sdk/config-schema` dla eksportu głównego schematu Zod `openclaw.json`
  (`OpenClawSchema`).
- Stabilne prymitywy kanałów, takie jak `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input` oraz
  `openclaw/plugin-sdk/webhook-ingress` dla współdzielonego połączenia
  konfiguracji/uwierzytelniania/odpowiedzi/Webhooków. `channel-inbound` to współdzielone miejsce dla debounce, dopasowywania wzmianek,
  pomocników polityki wzmianek przychodzących, formatowania kopert oraz pomocników kontekstu
  kopert przychodzących.
  `channel-setup` to wąska powierzchnia konfiguracji opcjonalnej instalacji.
  `setup-runtime` to bezpieczna dla runtime powierzchnia konfiguracji używana przez `setupEntry` /
  odroczone uruchamianie, w tym bezpieczne importowo adaptery łatek konfiguracji.
  `setup-adapter-runtime` to świadoma środowiska powierzchnia adaptera konfiguracji konta.
  `setup-tools` to mała powierzchnia pomocników CLI/archiwów/dokumentacji (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Podścieżki domenowe, takie jak `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store` oraz
  `openclaw/plugin-sdk/directory-runtime` dla współdzielonych pomocników runtime/konfiguracji.
  `telegram-command-config` to wąska publiczna powierzchnia dla normalizacji/walidacji niestandardowych
  poleceń Telegram i pozostaje dostępna nawet wtedy, gdy bundlowana
  powierzchnia kontraktu Telegram jest tymczasowo niedostępna.
  `text-runtime` to współdzielona powierzchnia tekst/Markdown/logowanie, w tym
  usuwanie tekstu widocznego dla asystenta, pomocniki renderowania/dzielenia Markdown, pomocniki redakcji,
  pomocniki tagów dyrektyw i narzędzia bezpiecznego tekstu.
- Powierzchnie kanałów specyficzne dla zatwierdzeń powinny preferować jeden kontrakt `approvalCapability`
  na Pluginie. Core odczytuje następnie uwierzytelnianie zatwierdzeń, dostarczanie, renderowanie,
  natywny routing i leniwe zachowanie natywnego handlera przez tę jedną możliwość
  zamiast mieszać zachowanie zatwierdzeń z niepowiązanymi polami Pluginu.
- `openclaw/plugin-sdk/channel-runtime` jest przestarzałe i pozostaje wyłącznie jako
  shim zgodności dla starszych Pluginów. Nowy kod powinien importować węższe
  ogólne prymitywy, a kod repo nie powinien dodawać nowych importów tego
  shimu.
- Wnętrze bundlowanych rozszerzeń pozostaje prywatne. Zewnętrzne Pluginy powinny używać wyłącznie podścieżek `openclaw/plugin-sdk/*`.
  Kod core/testów OpenClaw może używać publicznych punktów wejścia repo
  w katalogu głównym pakietu Pluginu, takich jak `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` oraz wąsko ograniczonych plików, takich jak
  `login-qr-api.js`. Nigdy nie importuj `src/*` pakietu Pluginu z core ani z
  innego rozszerzenia.
- Podział punktów wejścia repo:
  `<plugin-package-root>/api.js` to barrel pomocników/typów,
  `<plugin-package-root>/runtime-api.js` to barrel tylko runtime,
  `<plugin-package-root>/index.js` to punkt wejścia bundlowanego Pluginu,
  a `<plugin-package-root>/setup-entry.js` to punkt wejścia Pluginu konfiguracji.
- Obecne przykłady bundlowanych dostawców:
  - Anthropic używa `api.js` / `contract-api.js` dla pomocników strumienia Claude, takich
    jak `wrapAnthropicProviderStream`, pomocniki nagłówków beta i parsowanie
    `service_tier`.
  - OpenAI używa `api.js` dla builderów dostawców, pomocników modeli domyślnych i
    builderów dostawców realtime.
  - OpenRouter używa `api.js` dla swojego buildera dostawcy oraz pomocników onboardingu/konfiguracji,
    podczas gdy `register.runtime.js` może nadal ponownie eksportować ogólne
    pomocniki `plugin-sdk/provider-stream` do użytku lokalnego w repo.
- Publiczne punkty wejścia ładowane przez fasadę preferują aktywny snapshot konfiguracji runtime,
  jeśli istnieje, a następnie wracają do rozwiązanej konfiguracji z pliku na dysku, gdy
  OpenClaw nie udostępnia jeszcze snapshotu runtime.
- Ogólne współdzielone prymitywy pozostają preferowanym publicznym kontraktem SDK. Nadal istnieje mały
  zastrzeżony zestaw zgodności pomocniczych powierzchni bundlowanych kanałów opatrzonych marką. Traktuj je jako
  powierzchnie utrzymaniowe/zgodnościowe dla bundli, a nie nowe cele importu dla zewnętrznych Pluginów; nowe
  kontrakty międzykanałowe nadal powinny trafiać do ogólnych podścieżek `plugin-sdk/*` albo lokalnych barrelów Pluginu `api.js` /
  `runtime-api.js`.

Uwaga dotycząca zgodności:

- Unikaj głównego barrela `openclaw/plugin-sdk` w nowym kodzie.
- Najpierw preferuj wąskie stabilne prymitywy. Nowsze podścieżki setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool stanowią zamierzony kontrakt dla nowych
  bundlowanych i zewnętrznych Pluginów.
  Parsowanie/dopasowywanie celu należy umieszczać w `openclaw/plugin-sdk/channel-targets`.
  Bramki działań wiadomości i pomocniki identyfikatorów wiadomości reakcji należą do
  `openclaw/plugin-sdk/channel-actions`.
- Barrels pomocników specyficznych dla bundlowanych rozszerzeń nie są domyślnie stabilne. Jeśli
  pomocnik jest potrzebny tylko bundlowanemu rozszerzeniu, trzymaj go za lokalną powierzchnią
  `api.js` lub `runtime-api.js` tego rozszerzenia zamiast promować go do
  `openclaw/plugin-sdk/<extension>`.
- Nowe współdzielone powierzchnie pomocników powinny być ogólne, a nie oznaczone marką kanału. Współdzielone parsowanie celu
  należy umieszczać w `openclaw/plugin-sdk/channel-targets`; wnętrze specyficzne dla kanału
  powinno pozostawać za lokalną powierzchnią `api.js` lub `runtime-api.js` należącą do danego Pluginu.
- Podścieżki specyficzne dla możliwości, takie jak `image-generation`,
  `media-understanding` i `speech`, istnieją, ponieważ używają ich dziś
  bundlowane/natywne Pluginy. Ich obecność sama w sobie nie oznacza, że każdy eksportowany pomocnik jest
  długoterminowym zamrożonym kontraktem zewnętrznym.

## Schematy narzędzia message

Pluginy powinny zarządzać wkładami do schematu `describeMessageTool(...)`
specyficznymi dla kanału. Pola specyficzne dla dostawcy należy trzymać w Pluginie, a nie we współdzielonym core.

W przypadku współdzielonych przenośnych fragmentów schematu używaj ponownie ogólnych pomocników eksportowanych przez
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` dla ładunków w stylu siatki przycisków
- `createMessageToolCardSchema()` dla ładunków uporządkowanych kart

Jeśli dany kształt schematu ma sens tylko dla jednego dostawcy, zdefiniuj go we własnym
źródle tego Pluginu zamiast promować go do współdzielonego SDK.

## Rozwiązywanie celu kanału

Pluginy kanałów powinny zarządzać semantyką celu specyficzną dla kanału. Współdzielony
host outbound powinien pozostać ogólny, a dla reguł dostawcy należy używać powierzchni adaptera wiadomości:

- `messaging.inferTargetChatType({ to })` decyduje, czy znormalizowany cel
  powinien być traktowany jako `direct`, `group` czy `channel` przed wyszukiwaniem w katalogu.
- `messaging.targetResolver.looksLikeId(raw, normalized)` informuje core, czy dane
  wejście powinno od razu przejść do rozwiązywania podobnego do identyfikatora zamiast do wyszukiwania w katalogu.
- `messaging.targetResolver.resolveTarget(...)` to fallback Pluginu, gdy
  core potrzebuje końcowego rozwiązywania należącego do dostawcy po normalizacji albo po
  braku trafienia w katalogu.
- `messaging.resolveOutboundSessionRoute(...)` zarządza budowaniem trasy sesji
  specyficznej dla dostawcy po rozwiązaniu celu.

Zalecany podział:

- Używaj `inferTargetChatType` dla decyzji kategorialnych, które powinny zapaść przed
  przeszukiwaniem peerów/grup.
- Używaj `looksLikeId` dla sprawdzeń typu „traktuj to jako jawny/natywny identyfikator celu”.
- Używaj `resolveTarget` dla fallbacku normalizacji specyficznej dla dostawcy, a nie dla
  szerokiego wyszukiwania w katalogu.
- Natywne identyfikatory dostawcy, takie jak identyfikatory czatów, wątków, JID, handle i identyfikatory pokojów,
  należy trzymać wewnątrz wartości `target` albo parametrów specyficznych dla dostawcy, a nie w ogólnych polach SDK.

## Katalogi oparte na konfiguracji

Pluginy, które wyprowadzają wpisy katalogu z konfiguracji, powinny trzymać tę logikę w
Pluginie i ponownie używać współdzielonych pomocników z
`openclaw/plugin-sdk/directory-runtime`.

Używaj tego, gdy kanał potrzebuje peerów/grup opartych na konfiguracji, takich jak:

- peery DM sterowane przez allowlistę
- skonfigurowane mapy kanałów/grup
- statyczne fallbacki katalogów ograniczone do konta

Współdzielone pomocniki w `directory-runtime` obsługują tylko ogólne operacje:

- filtrowanie zapytań
- stosowanie limitów
- deduplikację/pomocniki normalizacji
- budowanie `ChannelDirectoryEntry[]`

Inspekcja kont specyficzna dla kanału i normalizacja identyfikatorów powinny pozostać w
implementacji Pluginu.

## Katalogi dostawców

Pluginy dostawców mogą definiować katalogi modeli dla wnioskowania przez
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` zwraca ten sam kształt, który OpenClaw zapisuje do
`models.providers`:

- `{ provider }` dla jednego wpisu dostawcy
- `{ providers }` dla wielu wpisów dostawców

Używaj `catalog`, gdy Plugin zarządza identyfikatorami modeli specyficznymi dla dostawcy, wartościami domyślnymi `baseUrl`
lub metadanymi modeli chronionymi uwierzytelnianiem.

`catalog.order` kontroluje, kiedy katalog Pluginu jest scalany względem
wbudowanych domyślnych dostawców OpenClaw:

- `simple`: zwykli dostawcy sterowani kluczem API lub zmiennymi środowiskowymi
- `profile`: dostawcy pojawiający się, gdy istnieją profile uwierzytelniania
- `paired`: dostawcy syntetyzujący wiele powiązanych wpisów dostawców
- `late`: ostatnie przejście, po innych domyślnych dostawcach

Późniejsi dostawcy wygrywają przy kolizji kluczy, więc Pluginy mogą celowo nadpisywać
wbudowany wpis dostawcy o tym samym identyfikatorze dostawcy.

Kompatybilność:

- `discovery` nadal działa jako starszy alias
- jeśli zarejestrowane są zarówno `catalog`, jak i `discovery`, OpenClaw używa `catalog`

## Inspekcja kanału tylko do odczytu

Jeśli Twój Plugin rejestruje kanał, preferuj implementację
`plugin.config.inspectAccount(cfg, accountId)` obok `resolveAccount(...)`.

Dlaczego:

- `resolveAccount(...)` to ścieżka runtime. Może zakładać, że poświadczenia
  są w pełni zmaterializowane, i może szybko kończyć się błędem, gdy brakuje wymaganych sekretów.
- Ścieżki poleceń tylko do odczytu, takie jak `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` oraz przepływy doctor/naprawy
  konfiguracji, nie powinny wymagać materializacji poświadczeń runtime tylko po to,
  aby opisać konfigurację.

Zalecane zachowanie `inspectAccount(...)`:

- Zwracaj tylko opisowy stan konta.
- Zachowuj `enabled` i `configured`.
- Dodawaj pola źródła/statusu poświadczeń, gdy są istotne, takie jak:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Nie musisz zwracać surowych wartości tokenów tylko po to, by raportować dostępność tylko do odczytu.
  Zwrócenie `tokenStatus: "available"` (oraz odpowiadającego mu pola źródła) wystarcza dla poleceń w stylu statusu.
- Używaj `configured_unavailable`, gdy poświadczenie jest skonfigurowane przez SecretRef, ale
  niedostępne w bieżącej ścieżce polecenia.

Dzięki temu polecenia tylko do odczytu mogą raportować „skonfigurowane, ale niedostępne w tej ścieżce polecenia”
zamiast kończyć się awarią albo błędnie raportować konto jako nieskonfigurowane.

## Pakiety zbiorcze

Katalog Pluginu może zawierać `package.json` z `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Każdy wpis staje się Pluginem. Jeśli pakiet zbiorczy zawiera wiele rozszerzeń, identyfikator Pluginu
przyjmuje postać `name/<fileBase>`.

Jeśli Twój Plugin importuje zależności npm, zainstaluj je w tym katalogu, aby
`node_modules` było dostępne (`npm install` / `pnpm install`).

Barierka bezpieczeństwa: każdy wpis `openclaw.extensions` musi pozostać wewnątrz katalogu Pluginu
po rozwiązaniu symlinków. Wpisy wychodzące poza katalog pakietu są
odrzucane.

Uwaga bezpieczeństwa: `openclaw plugins install` instaluje zależności Pluginów przez
`npm install --omit=dev --ignore-scripts` (bez skryptów cyklu życia i bez zależności deweloperskich w runtime). Drzewa zależności Pluginów powinny pozostać „czystym JS/TS” i unikać pakietów wymagających kompilacji `postinstall`.

Opcjonalnie: `openclaw.setupEntry` może wskazywać na lekki moduł tylko do konfiguracji.
Gdy OpenClaw potrzebuje powierzchni konfiguracji dla wyłączonego Pluginu kanału albo
gdy Plugin kanału jest włączony, ale nadal nieskonfigurowany, ładuje `setupEntry`
zamiast pełnego punktu wejścia Pluginu. Dzięki temu uruchamianie i konfiguracja są lżejsze,
gdy główny punkt wejścia Pluginu podłącza też narzędzia, hooki lub inny kod tylko runtime.

Opcjonalnie: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
może włączyć dla Pluginu kanału tę samą ścieżkę `setupEntry` podczas fazy
uruchamiania Gateway przed nasłuchiwaniem, nawet gdy kanał jest już skonfigurowany.

Używaj tego tylko wtedy, gdy `setupEntry` w pełni pokrywa powierzchnię uruchomieniową, która musi istnieć
przed rozpoczęciem nasłuchiwania przez Gateway. W praktyce oznacza to, że punkt wejścia konfiguracji
musi rejestrować każdą możliwość należącą do kanału, od której zależy uruchamianie, taką jak:

- sama rejestracja kanału
- wszelkie trasy HTTP, które muszą być dostępne przed rozpoczęciem nasłuchiwania przez Gateway
- wszelkie metody Gateway, narzędzia lub usługi, które muszą istnieć w tym samym oknie czasowym

Jeśli Twój pełny punkt wejścia nadal odpowiada za jakąkolwiek wymaganą możliwość uruchomieniową, nie włączaj
tej flagi. Pozostaw Plugin przy domyślnym zachowaniu i pozwól OpenClaw ładować
pełny punkt wejścia podczas uruchamiania.

Bundlowane kanały mogą też publikować pomocniki powierzchni kontraktowej tylko do konfiguracji, z których core
może korzystać przed załadowaniem pełnego runtime kanału. Obecna powierzchnia promocji konfiguracji to:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Core używa tej powierzchni, gdy musi promować starszą konfigurację kanału z jednym kontem
do `channels.<id>.accounts.*` bez ładowania pełnego punktu wejścia Pluginu.
Matrix jest obecnym bundlowanym przykładem: przenosi tylko klucze auth/bootstrap do
nazwanego promowanego konta, gdy nazwane konta już istnieją, i może zachować
skonfigurowany niekanoniczny klucz konta domyślnego zamiast zawsze tworzyć
`accounts.default`.

Te adaptery łatek konfiguracji utrzymują leniwe wykrywanie powierzchni kontraktowej bundli. Czas
importu pozostaje niski; powierzchnia promocji jest ładowana dopiero przy pierwszym użyciu zamiast
ponownego wchodzenia w uruchamianie bundlowanego kanału podczas importu modułu.

Gdy te powierzchnie uruchomieniowe obejmują metody Gateway RPC, utrzymuj je pod
prefiksem specyficznym dla Pluginu. Przestrzenie nazw administracyjnych core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) pozostają zastrzeżone i zawsze są rozwiązywane
do `operator.admin`, nawet jeśli Plugin żąda węższego zakresu.

Przykład:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadane katalogu kanału

Pluginy kanałów mogą reklamować metadane konfiguracji/wykrywania przez `openclaw.channel` oraz
wskazówki instalacji przez `openclaw.install`. Dzięki temu dane katalogu core pozostają wolne od danych.

Przykład:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Przydatne pola `openclaw.channel` poza minimalnym przykładem:

- `detailLabel`: etykieta pomocnicza dla bogatszych powierzchni katalogu/statusu
- `docsLabel`: nadpisuje tekst linku do dokumentacji
- `preferOver`: identyfikatory Pluginów/kanałów o niższym priorytecie, które ten wpis katalogu powinien wyprzedzać
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: kontrolki tekstu powierzchni wyboru
- `markdownCapable`: oznacza kanał jako obsługujący Markdown na potrzeby decyzji o formatowaniu outbound
- `exposure.configured`: ukrywa kanał na powierzchniach list skonfigurowanych kanałów, gdy ustawione na `false`
- `exposure.setup`: ukrywa kanał w interaktywnych selektorach konfiguracji, gdy ustawione na `false`
- `exposure.docs`: oznacza kanał jako wewnętrzny/prywatny dla powierzchni nawigacji dokumentacji
- `showConfigured` / `showInSetup`: starsze aliasy nadal akceptowane dla kompatybilności; preferuj `exposure`
- `quickstartAllowFrom`: włącza kanał do standardowego przepływu szybkiego startu `allowFrom`
- `forceAccountBinding`: wymaga jawnego powiązania konta nawet wtedy, gdy istnieje tylko jedno konto
- `preferSessionLookupForAnnounceTarget`: preferuje wyszukiwanie sesji podczas rozwiązywania celów ogłoszeń

OpenClaw może też scalać **zewnętrzne katalogi kanałów** (na przykład eksport rejestru MPM).
Umieść plik JSON w jednej z lokalizacji:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Albo wskaż `OPENCLAW_PLUGIN_CATALOG_PATHS` (lub `OPENCLAW_MPM_CATALOG_PATHS`) na
jeden lub więcej plików JSON (rozdzielanych przecinkiem/średnikiem/jak w `PATH`). Każdy plik powinien
zawierać `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. Parser akceptuje też `"packages"` lub `"plugins"` jako starsze aliasy dla klucza `"entries"`.

## Pluginy silnika kontekstu

Pluginy silnika kontekstu zarządzają orkiestracją kontekstu sesji dla ingestu, składania
i Compaction. Zarejestruj je ze swojego Pluginu przez
`api.registerContextEngine(id, factory)`, a następnie wybierz aktywny silnik przez
`plugins.slots.contextEngine`.

Używaj tego, gdy Twój Plugin musi zastąpić lub rozszerzyć domyślny potok
kontekstu, a nie tylko dodać wyszukiwanie pamięci lub hooki.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Jeśli Twój silnik **nie** zarządza algorytmem Compaction, pozostaw
zaimplementowane `compact()` i jawnie deleguj je dalej:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Dodawanie nowej możliwości

Gdy Plugin potrzebuje zachowania, które nie mieści się w obecnym API, nie omijaj
systemu Pluginów przez prywatne sięganie do wnętrza. Dodaj brakującą możliwość.

Zalecana sekwencja:

1. zdefiniuj kontrakt core
   Zdecyduj, jakim współdzielonym zachowaniem powinien zarządzać core: polityką, fallbackiem, scalaniem konfiguracji,
   cyklem życia, semantyką skierowaną do kanałów i kształtem pomocnika runtime.
2. dodaj typowane powierzchnie rejestracji/runtime Pluginów
   Rozszerz `OpenClawPluginApi` i/lub `api.runtime` o najmniejszą użyteczną
   typowaną powierzchnię możliwości.
3. podłącz core + konsumentów kanałów/funkcji
   Kanały i Pluginy funkcji powinny korzystać z nowej możliwości przez core,
   a nie przez bezpośredni import implementacji dostawcy.
4. zarejestruj implementacje dostawców
   Pluginy dostawców rejestrują następnie swoje backendy względem tej możliwości.
5. dodaj pokrycie kontraktowe
   Dodaj testy, aby własność i kształt rejestracji pozostawały z czasem jawne.

W ten sposób OpenClaw zachowuje wyraziste podejście, nie stając się na sztywno powiązanym
ze światopoglądem jednego dostawcy. Zobacz [Capability Cookbook](/pl/plugins/architecture),
aby uzyskać konkretną listę plików i opracowany przykład.

### Lista kontrolna możliwości

Gdy dodajesz nową możliwość, implementacja powinna zwykle dotykać tych
powierzchni razem:

- typów kontraktu core w `src/<capability>/types.ts`
- runnera/pomocnika runtime core w `src/<capability>/runtime.ts`
- powierzchni rejestracji API Pluginów w `src/plugins/types.ts`
- połączenia rejestru Pluginów w `src/plugins/registry.ts`
- udostępnienia runtime Pluginów w `src/plugins/runtime/*`, gdy Pluginy funkcji/kanałów
  muszą z tego korzystać
- pomocników przechwytywania/testów w `src/test-utils/plugin-registration.ts`
- asercji własności/kontraktów w `src/plugins/contracts/registry.ts`
- dokumentacji operatora/Pluginów w `docs/`

Jeśli którejś z tych powierzchni brakuje, zwykle jest to znak, że możliwość nie została
jeszcze w pełni zintegrowana.

### Szablon możliwości

Minimalny wzorzec:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Wzorzec testu kontraktowego:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

To utrzymuje prostą zasadę:

- core zarządza kontraktem możliwości + orkiestracją
- Pluginy dostawców zarządzają implementacjami dostawców
- Pluginy funkcji/kanałów korzystają z pomocników runtime
- testy kontraktowe utrzymują własność jako jawną
