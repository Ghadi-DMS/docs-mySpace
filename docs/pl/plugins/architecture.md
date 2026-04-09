---
read_when:
    - Tworzysz lub debugujesz natywne pluginy OpenClaw
    - Chcesz zrozumieć model capability pluginów lub granice własności
    - Pracujesz nad pipeline ładowania pluginów lub rejestrem
    - Implementujesz hooki runtime dostawcy lub pluginy kanałów
sidebarTitle: Internals
summary: 'Wnętrze pluginów: model capability, własność, kontrakty, pipeline ładowania i helpery runtime'
title: Wnętrze pluginów
x-i18n:
    generated_at: "2026-04-09T01:32:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2575791f835990589219bb06d8ca92e16a8c38b317f0bfe50b421682f253ef18
    source_path: plugins/architecture.md
    workflow: 15
---

# Wnętrze pluginów

<Info>
  To jest **dogłębna dokumentacja architektury**. Praktyczne przewodniki znajdziesz tutaj:
  - [Install and use plugins](/pl/tools/plugin) — przewodnik użytkownika
  - [Getting Started](/pl/plugins/building-plugins) — samouczek tworzenia pierwszego pluginu
  - [Channel Plugins](/pl/plugins/sdk-channel-plugins) — zbuduj kanał komunikacyjny
  - [Provider Plugins](/pl/plugins/sdk-provider-plugins) — zbuduj dostawcę modeli
  - [SDK Overview](/pl/plugins/sdk-overview) — mapa importów i API rejestracji
</Info>

Ta strona opisuje wewnętrzną architekturę systemu pluginów OpenClaw.

## Publiczny model capability

Capabilities to publiczny model **natywnych pluginów** w OpenClaw. Każdy
natywny plugin OpenClaw rejestruje się względem co najmniej jednego typu capability:

| Capability             | Metoda rejestracji                            | Przykładowe pluginy                |
| ---------------------- | --------------------------------------------- | ---------------------------------- |
| Wnioskowanie tekstowe  | `api.registerProvider(...)`                   | `openai`, `anthropic`              |
| Backend wnioskowania CLI | `api.registerCliBackend(...)`               | `openai`, `anthropic`              |
| Mowa                   | `api.registerSpeechProvider(...)`             | `elevenlabs`, `microsoft`          |
| Transkrypcja realtime  | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                        |
| Głos realtime          | `api.registerRealtimeVoiceProvider(...)`      | `openai`                           |
| Rozumienie mediów      | `api.registerMediaUnderstandingProvider(...)` | `openai`, `google`                 |
| Generowanie obrazów    | `api.registerImageGenerationProvider(...)`    | `openai`, `google`, `fal`, `minimax` |
| Generowanie muzyki     | `api.registerMusicGenerationProvider(...)`    | `google`, `minimax`                |
| Generowanie wideo      | `api.registerVideoGenerationProvider(...)`    | `qwen`                             |
| Pobieranie z sieci     | `api.registerWebFetchProvider(...)`           | `firecrawl`                        |
| Wyszukiwanie w sieci   | `api.registerWebSearchProvider(...)`          | `google`                           |
| Kanał / komunikacja    | `api.registerChannel(...)`                    | `msteams`, `matrix`                |

Plugin, który rejestruje zero capabilities, ale dostarcza hooki, narzędzia lub
usługi, jest pluginem **legacy hook-only**. Ten wzorzec jest nadal w pełni wspierany.

### Stan kompatybilności zewnętrznej

Model capability jest wdrożony w core i używany dziś przez dołączone/natywne
pluginy, ale kompatybilność z pluginami zewnętrznymi nadal wymaga wyższego progu
niż „jest eksportowane, więc jest zamrożone”.

Aktualne wytyczne:

- **istniejące pluginy zewnętrzne:** zachowuj działanie integracji opartych na hookach; traktuj
  to jako bazowy poziom kompatybilności
- **nowe dołączone/natywne pluginy:** preferuj jawną rejestrację capability zamiast
  zależności od konkretnego dostawcy lub nowych projektów hook-only
- **pluginy zewnętrzne przyjmujące rejestrację capability:** dozwolone, ale traktuj
  helpery specyficzne dla capability jako rozwijające się, chyba że dokumentacja
  wyraźnie oznacza kontrakt jako stabilny

Praktyczna zasada:

- API rejestracji capability jest docelowym kierunkiem
- legacy hooks pozostają najbezpieczniejszą ścieżką bez ryzyka złamania dla pluginów zewnętrznych podczas
  przejścia
- eksportowane subścieżki helperów nie są równoważne; preferuj wąski, udokumentowany
  kontrakt, a nie przypadkowe eksporty helperów

### Kształty pluginów

OpenClaw klasyfikuje każdy załadowany plugin do jednego z kształtów na podstawie
jego rzeczywistego zachowania rejestracyjnego (a nie tylko statycznych metadanych):

- **plain-capability** -- rejestruje dokładnie jeden typ capability (na przykład
  plugin tylko-dostawca, taki jak `mistral`)
- **hybrid-capability** -- rejestruje wiele typów capability (na przykład
  `openai` obsługuje wnioskowanie tekstowe, mowę, rozumienie mediów i generowanie
  obrazów)
- **hook-only** -- rejestruje wyłącznie hooki (typowane lub niestandardowe), bez capabilities,
  narzędzi, komend ani usług
- **non-capability** -- rejestruje narzędzia, komendy, usługi lub trasy, ale bez
  capabilities

Użyj `openclaw plugins inspect <id>`, aby zobaczyć kształt pluginu i podział capability.
Szczegóły znajdziesz w [CLI reference](/cli/plugins#inspect).

### Legacy hooks

Hook `before_agent_start` pozostaje wspierany jako ścieżka kompatybilności dla
pluginów hook-only. Legacy pluginy używane w praktyce nadal od niego zależą.

Kierunek:

- zachowaj jego działanie
- dokumentuj go jako legacy
- dla pracy z nadpisywaniem modelu/dostawcy preferuj `before_model_resolve`
- dla modyfikacji promptów preferuj `before_prompt_build`
- usuń go dopiero wtedy, gdy realne użycie spadnie, a pokrycie przez fixture potwierdzi
  bezpieczeństwo migracji

### Sygnały kompatybilności

Po uruchomieniu `openclaw doctor` lub `openclaw plugins inspect <id>` możesz zobaczyć
jedną z tych etykiet:

| Sygnał                     | Znaczenie                                                    |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | Konfiguracja parsuje się poprawnie, a pluginy są rozwiązywane |
| **compatibility advisory** | Plugin używa wspieranego, ale starszego wzorca (np. `hook-only`) |
| **legacy warning**         | Plugin używa `before_agent_start`, który jest przestarzały   |
| **hard error**             | Konfiguracja jest nieprawidłowa lub plugin nie załadował się |

Ani `hook-only`, ani `before_agent_start` nie zepsują dziś Twojego pluginu --
`hook-only` ma charakter informacyjny, a `before_agent_start` wywołuje tylko ostrzeżenie. Te
sygnały pojawiają się również w `openclaw status --all` i `openclaw plugins doctor`.

## Przegląd architektury

System pluginów OpenClaw ma cztery warstwy:

1. **Manifest + wykrywanie**
   OpenClaw znajduje kandydackie pluginy na podstawie skonfigurowanych ścieżek, katalogów
   workspace, globalnych katalogów rozszerzeń i dołączonych rozszerzeń. Wykrywanie najpierw odczytuje natywne
   manifesty `openclaw.plugin.json` oraz wspierane manifesty bundle.
2. **Włączanie + walidacja**
   Core decyduje, czy wykryty plugin jest włączony, wyłączony, zablokowany czy
   wybrany do ekskluzywnego slotu, takiego jak pamięć.
3. **Ładowanie runtime**
   Natywne pluginy OpenClaw są ładowane wewnątrz procesu przez jiti i rejestrują
   capabilities w centralnym rejestrze. Kompatybilne bundle są normalizowane do
   rekordów rejestru bez importowania kodu runtime.
4. **Korzystanie z powierzchni**
   Reszta OpenClaw odczytuje rejestr, aby udostępniać narzędzia, kanały, konfigurację
   dostawców, hooki, trasy HTTP, komendy CLI i usługi.

W szczególności dla pluginowego CLI wykrywanie komend głównych jest podzielone na dwa etapy:

- metadane czasu parsowania pochodzą z `registerCli(..., { descriptors: [...] })`
- właściwy moduł CLI pluginu może pozostać lazy i zarejestrować się przy pierwszym wywołaniu

Dzięki temu kod CLI należący do pluginu pozostaje w pluginie, a jednocześnie OpenClaw może
zarezerwować nazwy komend głównych przed parsowaniem.

Ważna granica projektowa:

- wykrywanie + walidacja konfiguracji powinny działać na podstawie **metadanych manifestu/schematu**
  bez wykonywania kodu pluginu
- natywne zachowanie runtime pochodzi ze ścieżki `register(api)` modułu pluginu

Ten podział pozwala OpenClaw walidować konfigurację, wyjaśniać brakujące/wyłączone pluginy i
budować wskazówki dla UI/schematu, zanim pełny runtime stanie się aktywny.

### Pluginy kanałów i współdzielone narzędzie wiadomości

Pluginy kanałów nie muszą rejestrować osobnego narzędzia send/edit/react dla
zwykłych działań czatu. OpenClaw utrzymuje jedno współdzielone narzędzie `message` w core, a
pluginy kanałów obsługują wykrywanie i wykonanie specyficzne dla kanału za nim.

Obecna granica wygląda tak:

- core zarządza hostem współdzielonego narzędzia `message`, podłączaniem promptów, prowadzeniem
  sesji/wątków i dyspozycją wykonania
- pluginy kanałów obsługują wykrywanie działań z zakresem, wykrywanie capabilities i wszelkie
  fragmenty schematu specyficzne dla kanału
- pluginy kanałów obsługują gramatykę konwersacji sesji specyficzną dla dostawcy, np.
  sposób, w jaki identyfikatory konwersacji kodują identyfikatory wątków lub dziedziczą po
  konwersacjach nadrzędnych
- pluginy kanałów wykonują końcowe działanie przez swój adapter działań

Dla pluginów kanałów powierzchnią SDK jest
`ChannelMessageActionAdapter.describeMessageTool(...)`. To ujednolicone wywołanie wykrywania
pozwala pluginowi zwrócić widoczne działania, capabilities i wkłady do schematu
razem, tak aby te elementy nie rozjeżdżały się z czasem.

Core przekazuje zakres runtime do tego kroku wykrywania. Ważne pola obejmują:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- zaufane przychodzące `requesterSenderId`

To ma znaczenie dla pluginów zależnych od kontekstu. Kanał może ukrywać lub ujawniać
działania wiadomości na podstawie aktywnego konta, bieżącego pokoju/wątku/wiadomości lub
zaufanej tożsamości nadawcy, bez hardcodowania gałęzi specyficznych dla kanału w
core'owym narzędziu `message`.

To dlatego zmiany routingu embedded-runner nadal są pracą po stronie pluginu: runner
odpowiada za przekazanie bieżącej tożsamości czatu/sesji do granicy wykrywania pluginu,
tak aby współdzielone narzędzie `message` ujawniało właściwą powierzchnię należącą do kanału
dla bieżącej tury.

W przypadku helperów wykonania należących do kanału dołączone pluginy powinny trzymać runtime
wykonania we własnych modułach rozszerzeń. Core nie zarządza już runtime'ami działań wiadomości dla
Discord, Slack, Telegram ani WhatsApp w `src/agents/tools`.
Nie publikujemy osobnych subścieżek `plugin-sdk/*-action-runtime`, a dołączone
pluginy powinny importować własny lokalny kod runtime bezpośrednio ze swoich
modułów należących do rozszerzenia.

Ta sama granica dotyczy ogólnie nazwanych według dostawców granic SDK: core nie powinien
importować wygodnych barrelów specyficznych dla kanałów takich jak Slack, Discord, Signal,
WhatsApp ani podobnych rozszerzeń. Jeśli core potrzebuje danego zachowania, powinien albo
korzystać z własnego `api.ts` / `runtime-api.ts` dołączonego pluginu, albo promować tę potrzebę
do wąskiej, ogólnej capability w współdzielonym SDK.

W przypadku ankiet istnieją obecnie dwie ścieżki wykonania:

- `outbound.sendPoll` jest współdzieloną bazą dla kanałów pasujących do wspólnego
  modelu ankiet
- `actions.handleAction("poll")` jest preferowaną ścieżką dla semantyki ankiet
  specyficznej dla kanału lub dodatkowych parametrów ankiet

Core odracza teraz współdzielone parsowanie ankiet do momentu, aż pluginowa dyspozycja ankiety
odrzuci działanie, tak aby należące do pluginu handlery ankiet mogły akceptować pola ankiet
specyficzne dla kanału bez wcześniejszego zablokowania przez ogólny parser ankiet.

Pełną sekwencję uruchamiania znajdziesz w [Load pipeline](#load-pipeline).

## Model własności capability

OpenClaw traktuje natywny plugin jako granicę własności dla **firmy** albo
**funkcji**, a nie jako worek na niepowiązane integracje.

To oznacza, że:

- plugin firmowy powinien zwykle obsługiwać wszystkie powierzchnie OpenClaw skierowane do tej firmy
- plugin funkcjonalny powinien zwykle obsługiwać pełną powierzchnię funkcji, którą wprowadza
- kanały powinny korzystać ze współdzielonych capabilities core zamiast implementować
  zachowanie dostawców ad hoc

Przykłady:

- dołączony plugin `openai` obsługuje zachowanie dostawcy modeli OpenAI oraz
  zachowanie OpenAI dla mowy + realtime voice + rozumienia mediów + generowania obrazów
- dołączony plugin `elevenlabs` obsługuje zachowanie mowy ElevenLabs
- dołączony plugin `microsoft` obsługuje zachowanie mowy Microsoft
- dołączony plugin `google` obsługuje zachowanie dostawcy modeli Google oraz Google
  media-understanding + image-generation + web-search
- dołączony plugin `firecrawl` obsługuje zachowanie web-fetch Firecrawl
- dołączone pluginy `minimax`, `mistral`, `moonshot` i `zai` obsługują własne
  backendy media-understanding
- dołączony plugin `qwen` obsługuje zachowanie dostawcy tekstu Qwen oraz
  media-understanding i video-generation
- plugin `voice-call` jest pluginem funkcjonalnym: obsługuje transport połączeń, narzędzia,
  CLI, trasy i mostkowanie strumieni mediów Twilio, ale korzysta ze współdzielonych capabilities speech
  oraz realtime-transcription i realtime-voice zamiast importować pluginy dostawców bezpośrednio

Docelowy stan to:

- OpenAI znajduje się w jednym pluginie nawet wtedy, gdy obejmuje modele tekstowe, mowę, obrazy i
  przyszłe wideo
- inny dostawca może zrobić to samo dla własnej powierzchni
- kanały nie obchodzą, który plugin dostawcy obsługuje provider; korzystają ze
  współdzielonego kontraktu capability udostępnionego przez core

To jest kluczowe rozróżnienie:

- **plugin** = granica własności
- **capability** = kontrakt core, który może być implementowany lub używany przez wiele pluginów

Jeśli więc OpenClaw dodaje nową domenę, taką jak wideo, pierwsze pytanie nie brzmi
„który dostawca powinien na sztywno obsłużyć wideo?”. Pierwsze pytanie brzmi „jaki jest
kontrakt core capability dla wideo?”. Gdy taki kontrakt istnieje, pluginy dostawców
mogą się względem niego rejestrować, a pluginy kanałów/funkcji mogą z niego korzystać.

Jeśli capability jeszcze nie istnieje, właściwym krokiem jest zwykle:

1. zdefiniować brakującą capability w core
2. udostępnić ją w typowany sposób przez API/runtime pluginu
3. podłączyć kanały/funkcje do tej capability
4. pozwolić pluginom dostawców zarejestrować implementacje

To utrzymuje jawną własność, jednocześnie unikając zachowania core zależnego od
jednego dostawcy lub jednorazowej ścieżki kodu specyficznej dla pluginu.

### Warstwowanie capability

Używaj tego modelu mentalnego przy podejmowaniu decyzji, gdzie powinien znajdować się kod:

- **warstwa capability core**: współdzielona orkiestracja, polityka, fallback, zasady
  łączenia konfiguracji, semantyka dostarczania i typowane kontrakty
- **warstwa pluginu dostawcy**: API specyficzne dla dostawcy, auth, katalogi modeli, synteza mowy,
  generowanie obrazów, przyszłe backendy wideo, endpointy użycia
- **warstwa pluginu kanału/funkcji**: integracja Slack/Discord/voice-call/etc.,
  która korzysta z capabilities core i wystawia je na zewnątrz

Na przykład TTS ma taki kształt:

- core zarządza polityką TTS przy odpowiedzi, kolejnością fallbacków, preferencjami i dostarczaniem do kanału
- `openai`, `elevenlabs` i `microsoft` obsługują implementacje syntezy
- `voice-call` korzysta z helpera runtime TTS dla telefonii

Ten sam wzorzec powinien być preferowany dla przyszłych capabilities.

### Przykład firmowego pluginu z wieloma capabilities

Plugin firmowy powinien z zewnątrz sprawiać spójne wrażenie. Jeśli OpenClaw ma współdzielone
kontrakty dla modeli, mowy, transkrypcji realtime, głosu realtime, rozumienia mediów,
generowania obrazów, generowania wideo, web fetch i web search, dostawca może
obsługiwać wszystkie swoje powierzchnie w jednym miejscu:

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

Liczą się nie tyle dokładne nazwy helperów, co sam kształt:

- jeden plugin obsługuje powierzchnię dostawcy
- core nadal obsługuje kontrakty capability
- kanały i pluginy funkcjonalne korzystają z helperów `api.runtime.*`, a nie z kodu dostawcy
- testy kontraktów mogą potwierdzić, że plugin zarejestrował capabilities, których
  twierdzi, że jest właścicielem

### Przykład capability: rozumienie wideo

OpenClaw już traktuje rozumienie obrazów/audio/wideo jako jedną współdzieloną
capability. Ten sam model własności ma tu zastosowanie:

1. core definiuje kontrakt media-understanding
2. pluginy dostawców rejestrują `describeImage`, `transcribeAudio` i
   `describeVideo`, stosownie do możliwości
3. kanały i pluginy funkcjonalne korzystają ze współdzielonego zachowania core zamiast
   bezpośrednio podłączać kod dostawcy

To zapobiega wpisaniu założeń jednego dostawcy o wideo na stałe do core. Plugin obsługuje
powierzchnię dostawcy; core obsługuje kontrakt capability i zachowanie fallback.

Generowanie wideo używa już tej samej sekwencji: core obsługuje typowany
kontrakt capability i helper runtime, a pluginy dostawców rejestrują
implementacje `api.registerVideoGenerationProvider(...)`.

Potrzebujesz konkretnej listy wdrożeniowej? Zobacz
[Capability Cookbook](/pl/plugins/architecture).

## Kontrakty i egzekwowanie

Powierzchnia API pluginów jest celowo typowana i scentralizowana w
`OpenClawPluginApi`. Ten kontrakt definiuje wspierane punkty rejestracji oraz
helpery runtime, na których plugin może polegać.

Dlaczego to ważne:

- autorzy pluginów dostają jeden stabilny wewnętrzny standard
- core może odrzucać zduplikowaną własność, na przykład dwa pluginy rejestrujące ten sam
  identyfikator dostawcy
- podczas uruchamiania można pokazać użyteczne diagnostyki dla nieprawidłowej rejestracji
- testy kontraktów mogą egzekwować własność dołączonych pluginów i zapobiegać cichemu driftowi

Istnieją dwie warstwy egzekwowania:

1. **egzekwowanie rejestracji w runtime**
   Rejestr pluginów waliduje rejestracje podczas ładowania pluginów. Przykłady:
   zduplikowane identyfikatory providerów, zduplikowane identyfikatory dostawców mowy i nieprawidłowe
   rejestracje powodują diagnostyki pluginów zamiast niezdefiniowanego zachowania.
2. **testy kontraktów**
   Dołączone pluginy są przechwytywane w rejestrach kontraktów podczas uruchamiania testów, aby
   OpenClaw mógł jawnie potwierdzać własność. Obecnie używa się tego dla
   providerów modeli, dostawców mowy, dostawców web search i własności rejestracji dołączonych pluginów.

Praktyczny efekt jest taki, że OpenClaw wie z góry, który plugin obsługuje którą
powierzchnię. Dzięki temu core i kanały mogą składać się bezproblemowo, ponieważ własność jest
zadeklarowana, typowana i testowalna, a nie dorozumiana.

### Co należy do kontraktu

Dobre kontrakty pluginów są:

- typowane
- małe
- specyficzne dla capability
- należące do core
- możliwe do ponownego użycia przez wiele pluginów
- możliwe do użycia przez kanały/funkcje bez wiedzy o dostawcy

Złe kontrakty pluginów to:

- polityka specyficzna dla dostawcy ukryta w core
- jednorazowe furtki dla pluginów omijające rejestr
- kod kanału sięgający bezpośrednio do implementacji dostawcy
- ad hoc obiekty runtime, które nie są częścią `OpenClawPluginApi` ani
  `api.runtime`

W razie wątpliwości podnieś poziom abstrakcji: najpierw zdefiniuj capability, a potem
pozwól pluginom się do niej podłączyć.

## Model wykonania

Natywne pluginy OpenClaw działają **wewnątrz procesu** razem z Gateway. Nie są
sandboxowane. Załadowany natywny plugin ma tę samą granicę zaufania na poziomie procesu co
kod core.

Implikacje:

- natywny plugin może rejestrować narzędzia, handlery sieciowe, hooki i usługi
- błąd natywnego pluginu może zawiesić lub zdestabilizować gateway
- złośliwy natywny plugin jest równoważny dowolnemu wykonaniu kodu wewnątrz
  procesu OpenClaw

Kompatybilne bundle są domyślnie bezpieczniejsze, ponieważ OpenClaw obecnie traktuje je
jako paczki metadanych/treści. W obecnych wydaniach oznacza to głównie dołączone
Skills.

Dla pluginów niedołączonych używaj allowlist i jawnych ścieżek instalacji/ładowania. Traktuj
pluginy workspace jako kod deweloperski, a nie domyślne ustawienie produkcyjne.

Dla nazw pakietów dołączonych pluginów workspace utrzymuj identyfikator pluginu zakotwiczony w nazwie npm:
domyślnie `@openclaw/<id>` albo zatwierdzony sufiks typowany, taki jak
`-provider`, `-plugin`, `-speech`, `-sandbox` lub `-media-understanding`, gdy
pakiet celowo wystawia węższą rolę pluginu.

Ważna uwaga dotycząca zaufania:

- `plugins.allow` ufa **identyfikatorom pluginów**, a nie pochodzeniu źródła.
- Plugin workspace o tym samym identyfikatorze co dołączony plugin celowo przesłania
  dołączoną kopię, gdy ten plugin workspace jest włączony/na allowliście.
- To normalne i przydatne przy lokalnym rozwoju, testowaniu poprawek i hotfixach.

## Granica eksportu

OpenClaw eksportuje capabilities, a nie implementacyjne ułatwienia.

Utrzymuj publiczną rejestrację capability. Ograniczaj eksport helperów, które nie są częścią kontraktu:

- subścieżki helperów specyficznych dla dołączonych pluginów
- subścieżki infrastruktury runtime, które nie są przeznaczone jako publiczne API
- helpery wygodne specyficzne dla dostawców
- helpery konfiguracji/onboardingu będące szczegółami implementacji

Niektóre subścieżki helperów dołączonych pluginów nadal pozostają w wygenerowanej mapie eksportów SDK
ze względów kompatybilności i utrzymania dołączonych pluginów. Aktualne przykłady to
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` oraz kilka granic `plugin-sdk/matrix*`. Traktuj je jako
zastrzeżone eksporty będące szczegółami implementacji, a nie jako zalecany wzorzec SDK dla
nowych pluginów zewnętrznych.

## Pipeline ładowania

Podczas uruchamiania OpenClaw robi w przybliżeniu to:

1. wykrywa katalogi główne kandydackich pluginów
2. odczytuje natywne lub kompatybilne manifesty bundle i metadane pakietów
3. odrzuca niebezpiecznych kandydatów
4. normalizuje konfigurację pluginów (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decyduje o włączeniu dla każdego kandydata
6. ładuje włączone natywne moduły przez jiti
7. wywołuje hooki natywnego `register(api)` (lub `activate(api)` — legacy alias) i zbiera rejestracje do rejestru pluginów
8. udostępnia rejestr komendom/powierzchniom runtime

<Note>
`activate` jest legacy aliasem `register` — loader rozwiązuje to, co jest obecne (`def.register ?? def.activate`) i wywołuje je w tym samym miejscu. Wszystkie dołączone pluginy używają `register`; dla nowych pluginów preferuj `register`.
</Note>

Bramki bezpieczeństwa działają **przed** wykonaniem runtime. Kandydaci są blokowani,
gdy entry wychodzi poza katalog główny pluginu, ścieżka ma uprawnienia world-writable albo
własność ścieżki wygląda podejrzanie dla pluginów niedołączonych.

### Zachowanie manifest-first

Manifest jest źródłem prawdy dla control plane. OpenClaw używa go do:

- identyfikacji pluginu
- wykrywania deklarowanych kanałów/Skills/schematu konfiguracji lub capabilities bundle
- walidacji `plugins.entries.<id>.config`
- rozszerzania etykiet/placeholderów Control UI
- wyświetlania metadanych instalacji/katalogu

Dla natywnych pluginów moduł runtime jest częścią data plane. Rejestruje
rzeczywiste zachowania, takie jak hooki, narzędzia, komendy lub przepływy providerów.

### Co loader przechowuje w cache

OpenClaw utrzymuje krótkie cache'e wewnątrz procesu dla:

- wyników wykrywania
- danych rejestru manifestów
- załadowanych rejestrów pluginów

Te cache'e zmniejszają koszt gwałtownych startów i powtarzanych komend. Można o nich myśleć
jako o krótkotrwałych cache'ach wydajnościowych, a nie o trwałym stanie.

Uwaga dotycząca wydajności:

- Ustaw `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` albo
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1`, aby wyłączyć te cache'e.
- Okna cache można stroić przez `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` i
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Model rejestru

Załadowane pluginy nie mutują bezpośrednio losowych globali core. Rejestrują się w
centralnym rejestrze pluginów.

Rejestr śledzi:

- rekordy pluginów (tożsamość, źródło, pochodzenie, status, diagnostyka)
- narzędzia
- legacy hooks i hooki typowane
- kanały
- providerów
- handlery Gateway RPC
- trasy HTTP
- rejestratory CLI
- usługi w tle
- komendy należące do pluginów

Funkcje core odczytują następnie z tego rejestru zamiast komunikować się bezpośrednio z modułami pluginów.
Dzięki temu ładowanie pozostaje jednokierunkowe:

- moduł pluginu -> rejestracja w rejestrze
- runtime core -> korzystanie z rejestru

To rozdzielenie ma znaczenie dla utrzymywalności. Oznacza, że większość powierzchni core
potrzebuje tylko jednego punktu integracji: „odczytaj rejestr”, a nie „obsługuj osobno każdy moduł pluginu”.

## Callbacki powiązania konwersacji

Pluginy, które wiążą konwersację, mogą reagować po rozstrzygnięciu zatwierdzenia.

Użyj `api.onConversationBindingResolved(...)`, aby otrzymać callback po zatwierdzeniu lub odrzuceniu
żądania powiązania:

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

Pola ładunku callbacka:

- `status`: `"approved"` lub `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` lub `"deny"`
- `binding`: rozstrzygnięte powiązanie dla zatwierdzonych żądań
- `request`: oryginalne podsumowanie żądania, wskazówka odłączenia, identyfikator nadawcy oraz
  metadane konwersacji

Ten callback służy wyłącznie do powiadamiania. Nie zmienia tego, kto może powiązać
konwersację, i uruchamia się po zakończeniu obsługi zatwierdzenia przez core.

## Hooki runtime dostawcy

Pluginy dostawców mają teraz dwie warstwy:

- metadane manifestu: `providerAuthEnvVars` dla taniego sprawdzania auth dostawcy z env
  przed załadowaniem runtime, `providerAuthAliases` dla wariantów dostawców współdzielących
  auth, `channelEnvVars` dla taniego sprawdzania env/konfiguracji kanału przed
  załadowaniem runtime oraz `providerAuthChoices` dla tanich etykiet onboardingu/wyboru auth i
  metadanych flag CLI przed załadowaniem runtime
- hooki czasu konfiguracji: `catalog` / legacy `discovery` oraz `applyConfigDefaults`
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

OpenClaw nadal obsługuje ogólną pętlę agenta, failover, obsługę transkryptów i
politykę narzędzi. Te hooki są powierzchnią rozszerzeń dla zachowań specyficznych dla dostawcy bez
potrzeby budowania całkowicie niestandardowego transportu inferencyjnego.

Używaj manifestowego `providerAuthEnvVars`, gdy dostawca ma poświadczenia oparte na env,
które ogólne ścieżki auth/status/wyboru modelu powinny widzieć bez ładowania runtime pluginu.
Używaj manifestowego `providerAuthAliases`, gdy jeden identyfikator dostawcy ma ponownie używać
zmiennych env, profili auth, auth opartego na konfiguracji oraz wyboru onboardingu klucza API
innego identyfikatora dostawcy. Używaj manifestowego `providerAuthChoices`, gdy powierzchnie
CLI onboardingu/wyboru auth powinny znać identyfikator wyboru dostawcy, etykiety grup i prostą
obsługę auth przez pojedynczą flagę bez ładowania runtime dostawcy. Zachowaj runtime'owe
`envVars` dostawcy dla wskazówek operatora, takich jak etykiety onboardingowe lub zmienne
konfiguracji OAuth client-id/client-secret.

Używaj manifestowego `channelEnvVars`, gdy kanał ma auth lub konfigurację opartą na env, które
ogólny fallback z shell-env, sprawdzania config/status albo prompty konfiguracji powinny widzieć
bez ładowania runtime kanału.

### Kolejność hooków i ich użycie

Dla pluginów model/provider OpenClaw wywołuje hooki mniej więcej w tej kolejności.
Kolumna „Kiedy używać” to szybki przewodnik decyzyjny.

| #   | Hook                              | Co robi                                                                                                        | Kiedy używać                                                                                                                               |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `catalog`                         | Publikuje konfigurację dostawcy do `models.providers` podczas generowania `models.json`                       | Dostawca posiada katalog albo domyślne wartości `baseUrl`                                                                                  |
| 2   | `applyConfigDefaults`             | Stosuje globalne domyślne ustawienia konfiguracji należące do dostawcy podczas materializacji konfiguracji    | Ustawienia domyślne zależą od trybu auth, env lub semantyki rodziny modeli dostawcy                                                       |
| --  | _(wbudowane wyszukiwanie modelu)_ | OpenClaw najpierw próbuje normalnej ścieżki rejestru/katalogu                                                  | _(to nie jest hook pluginu)_                                                                                                               |
| 3   | `normalizeModelId`                | Normalizuje legacy aliasy identyfikatorów modeli lub wersji preview przed wyszukaniem                         | Dostawca zarządza porządkowaniem aliasów przed kanonicznym rozstrzygnięciem modelu                                                        |
| 4   | `normalizeTransport`              | Normalizuje `api` / `baseUrl` dla rodziny dostawców przed ogólnym składaniem modelu                           | Dostawca zarządza porządkowaniem transportu dla niestandardowych identyfikatorów dostawców w tej samej rodzinie transportu               |
| 5   | `normalizeConfig`                 | Normalizuje `models.providers.<id>` przed rozstrzygnięciem runtime/dostawcy                                   | Dostawca potrzebuje porządkowania konfiguracji, które powinno znajdować się w pluginie; dołączone helpery rodziny Google również wspierają tu wspierane wpisy konfiguracji Google |
| 6   | `applyNativeStreamingUsageCompat` | Stosuje kompatybilnościowe przepisywanie natywnego użycia streaming do providerów konfiguracji                | Dostawca potrzebuje poprawek metadanych natywnego użycia streaming zależnych od endpointu                                                 |
| 7   | `resolveConfigApiKey`             | Rozstrzyga auth typu env-marker dla providerów konfiguracji przed ładowaniem auth runtime                     | Dostawca ma własne rozstrzyganie klucza API przez env-marker; `amazon-bedrock` ma tu także wbudowany resolver env-marker AWS             |
| 8   | `resolveSyntheticAuth`            | Ujawnia auth lokalny/self-hosted albo oparty na konfiguracji bez utrwalania jawnego tekstu                    | Dostawca może działać z syntetycznym/lokalnym markerem poświadczeń                                                                         |
| 9   | `resolveExternalAuthProfiles`     | Nakłada profile zewnętrznego auth należące do dostawcy; domyślne `persistence` to `runtime-only` dla poświadczeń należących do CLI/aplikacji | Dostawca ponownie używa poświadczeń zewnętrznego auth bez utrwalania skopiowanych refresh tokenów                                 |
| 10  | `shouldDeferSyntheticProfileAuth` | Obniża priorytet zapisanych syntetycznych placeholderów profili względem auth opartego na env/konfiguracji    | Dostawca przechowuje syntetyczne placeholdery profili, które nie powinny mieć pierwszeństwa                                               |
| 11  | `resolveDynamicModel`             | Synchroniczny fallback dla identyfikatorów modeli należących do dostawcy, których nie ma jeszcze w lokalnym rejestrze | Dostawca akceptuje dowolne identyfikatory modeli upstream                                                                            |
| 12  | `prepareDynamicModel`             | Asynchroniczne rozgrzanie, po czym `resolveDynamicModel` uruchamia się ponownie                               | Dostawca potrzebuje metadanych z sieci przed rozstrzyganiem nieznanych identyfikatorów                                                    |
| 13  | `normalizeResolvedModel`          | Ostateczne przepisanie przed użyciem rozstrzygniętego modelu przez embedded runner                            | Dostawca potrzebuje przepisań transportu, ale nadal używa transportu core                                                                 |
| 14  | `contributeResolvedModelCompat`   | Dostarcza flagi kompatybilności dla modeli dostawcy działających przez inny kompatybilny transport            | Dostawca rozpoznaje własne modele na transportach proxy bez przejmowania roli dostawcy                                                    |
| 15  | `capabilities`                    | Metadane transkryptów/narzędzi należące do dostawcy, używane przez współdzieloną logikę core                  | Dostawca potrzebuje niuansów związanych z transkryptem/rodziną dostawców                                                                  |
| 16  | `normalizeToolSchemas`            | Normalizuje schematy narzędzi, zanim zobaczy je embedded runner                                                | Dostawca potrzebuje porządkowania schematów dla rodziny transportu                                                                        |
| 17  | `inspectToolSchemas`              | Ujawnia diagnostykę schematów należącą do dostawcy po normalizacji                                             | Dostawca chce ostrzeżeń o słowach kluczowych bez uczenia core zasad specyficznych dla dostawcy                                           |
| 18  | `resolveReasoningOutputMode`      | Wybiera natywny albo tagowany kontrakt wyjścia reasoning                                                       | Dostawca potrzebuje tagowanego reasoning/final output zamiast natywnych pól                                                               |
| 19  | `prepareExtraParams`              | Normalizacja parametrów żądania przed ogólnymi wrapperami opcji stream                                         | Dostawca potrzebuje domyślnych parametrów żądania lub porządkowania parametrów dla konkretnego dostawcy                                  |
| 20  | `createStreamFn`                  | W pełni zastępuje normalną ścieżkę stream własnym transportem                                                  | Dostawca potrzebuje własnego protokołu wire, a nie tylko wrappera                                                                         |
| 21  | `wrapStreamFn`                    | Wrapper stream po zastosowaniu ogólnych wrapperów                                                              | Dostawca potrzebuje wrapperów zgodności nagłówków/ciała/modelu bez własnego transportu                                                   |
| 22  | `resolveTransportTurnState`       | Dołącza natywne nagłówki per-turn transportu lub metadane                                                      | Dostawca chce, aby ogólne transporty wysyłały natywną tożsamość tury dostawcy                                                            |
| 23  | `resolveWebSocketSessionPolicy`   | Dołącza natywne nagłówki WebSocket albo politykę cool-down sesji                                               | Dostawca chce, by ogólne transporty WS stroiły nagłówki sesji lub politykę fallback                                                      |
| 24  | `formatApiKey`                    | Formatter profilu auth: zapisany profil staje się runtime'owym ciągiem `apiKey`                               | Dostawca przechowuje dodatkowe metadane auth i potrzebuje niestandardowego kształtu tokenu runtime                                      |
| 25  | `refreshOAuth`                    | Nadpisanie odświeżania OAuth dla niestandardowych endpointów odświeżania lub polityki błędów odświeżania      | Dostawca nie pasuje do współdzielonych odświeżaczy `pi-ai`                                                                                |
| 26  | `buildAuthDoctorHint`             | Wskazówka naprawcza dołączana po niepowodzeniu odświeżenia OAuth                                               | Dostawca potrzebuje własnych wskazówek naprawy auth po błędzie odświeżenia                                                                |
| 27  | `matchesContextOverflowError`     | Matcher przepełnienia okna kontekstu należący do dostawcy                                                      | Dostawca ma surowe błędy przepełnienia, których ogólne heurystyki nie wychwycą                                                           |
| 28  | `classifyFailoverReason`          | Klasyfikacja przyczyn failover należąca do dostawcy                                                            | Dostawca potrafi mapować surowe błędy API/transportu na rate-limit/przeciążenie itp.                                                      |
| 29  | `isCacheTtlEligible`              | Polityka prompt-cache dla dostawców proxy/backhaul                                                             | Dostawca potrzebuje bramkowania TTL cache specyficznego dla proxy                                                                         |
| 30  | `buildMissingAuthMessage`         | Zastępstwo dla ogólnego komunikatu odzyskiwania po braku auth                                                  | Dostawca potrzebuje specyficznej dla siebie wskazówki odzyskiwania po braku auth                                                          |
| 31  | `suppressBuiltInModel`            | Tłumienie przestarzałych modeli upstream plus opcjonalna wskazówka błędu dla użytkownika                      | Dostawca musi ukrywać przestarzałe wiersze upstream lub zastąpić je wskazówką dostawcy                                                    |
| 32  | `augmentModelCatalog`             | Syntetyczne/końcowe wiersze katalogu dodawane po wykryciu                                                      | Dostawca potrzebuje syntetycznych wierszy zgodnych z przyszłością w `models list` i selektorach                                         |
| 33  | `isBinaryThinking`                | Przełącznik on/off reasoning dla providerów binary-thinking                                                    | Dostawca udostępnia tylko binarne włączenie/wyłączenie thinking                                                                           |
| 34  | `supportsXHighThinking`           | Obsługa reasoning `xhigh` dla wybranych modeli                                                                 | Dostawca chce `xhigh` tylko dla podzbioru modeli                                                                                           |
| 35  | `resolveDefaultThinkingLevel`     | Domyślny poziom `/think` dla konkretnej rodziny modeli                                                         | Dostawca zarządza domyślną polityką `/think` dla rodziny modeli                                                                            |
| 36  | `isModernModelRef`                | Matcher nowoczesnych modeli do filtrów live profile i wyboru smoke                                             | Dostawca zarządza dopasowaniem preferowanych modeli live/smoke                                                                             |
| 37  | `prepareRuntimeAuth`              | Wymienia skonfigurowane poświadczenie na właściwy token/klucz runtime tuż przed inferencją                    | Dostawca potrzebuje wymiany tokenu albo krótkotrwałego poświadczenia dla żądania                                                          |
| 38  | `resolveUsageAuth`                | Rozstrzyga poświadczenia użycia/rozliczeń dla `/usage` i powiązanych powierzchni statusu                      | Dostawca potrzebuje własnego parsowania tokenów użycia/kwot lub innego poświadczenia użycia                                               |
| 39  | `fetchUsageSnapshot`              | Pobiera i normalizuje snapshoty użycia/kwot specyficzne dla dostawcy po rozstrzygnięciu auth                  | Dostawca potrzebuje endpointu użycia lub parsera ładunku specyficznego dla dostawcy                                                       |
| 40  | `createEmbeddingProvider`         | Buduje adapter embeddingów należący do dostawcy dla pamięci/wyszukiwania                                       | Zachowanie embeddingów pamięci należy do pluginu dostawcy                                                                                  |
| 41  | `buildReplayPolicy`               | Zwraca politykę replay kontrolującą obsługę transkryptu dla dostawcy                                           | Dostawca potrzebuje własnej polityki transkryptów (na przykład usuwania bloków thinking)                                                  |
| 42  | `sanitizeReplayHistory`           | Przepisuje historię replay po ogólnym czyszczeniu transkryptu                                                  | Dostawca potrzebuje własnych przepisań replay wykraczających poza współdzielone helpery kompaktowania                                     |
| 43  | `validateReplayTurns`             | Ostateczna walidacja lub przekształcenie tur replay przed embedded runnerem                                    | Transport dostawcy wymaga surowszej walidacji tur po ogólnej sanitacji                                                                     |
| 44  | `onModelSelected`                 | Uruchamia skutki uboczne należące do dostawcy po wybraniu modelu                                               | Dostawca potrzebuje telemetrii lub stanu należącego do dostawcy, gdy model staje się aktywny                                              |

`normalizeModelId`, `normalizeTransport` i `normalizeConfig` najpierw sprawdzają
dopasowany plugin dostawcy, a następnie przechodzą do innych pluginów dostawców zdolnych do hooków,
dopóki któryś faktycznie nie zmieni identyfikatora modelu albo transportu/konfiguracji. Dzięki temu
shimy aliasów/kompatybilności providerów nadal działają bez wymagania, aby wywołujący wiedział,
który dołączony plugin obsługuje przepisanie. Jeśli żaden hook dostawcy nie przepisze
wspieranego wpisu konfiguracji rodziny Google, dołączony normalizator konfiguracji Google nadal
stosuje tę poprawkę kompatybilności.

Jeśli dostawca potrzebuje w pełni niestandardowego protokołu wire albo niestandardowego wykonawcy żądań,
to jest inna klasa rozszerzenia. Te hooki są przeznaczone dla zachowań dostawców,
które nadal działają na normalnej pętli inferencyjnej OpenClaw.

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
  i `wrapStreamFn`, ponieważ obsługuje zgodność z przyszłością Claude 4.6,
  wskazówki dla rodziny providerów, wskazówki naprawy auth, integrację
  endpointu użycia, kwalifikowalność prompt-cache, domyślne ustawienia konfiguracji zależne od auth,
  politykę domyślnego/adaptacyjnego thinking dla Claude oraz kształtowanie streamu specyficzne dla Anthropic
  dla nagłówków beta, `/fast` / `serviceTier` i `context1m`.
- Helpery streamów specyficznych dla Claude w Anthropic pozostają na razie
  w publicznej granicy `api.ts` / `contract-api.ts` należącej do dołączonego pluginu. Ta powierzchnia
  pakietu eksportuje `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` oraz niższopoziomowe
  buildery wrapperów Anthropic zamiast poszerzać ogólne SDK o reguły nagłówków beta jednego
  dostawcy.
- OpenAI używa `resolveDynamicModel`, `normalizeResolvedModel` i
  `capabilities` oraz `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` i `isModernModelRef`,
  ponieważ obsługuje zgodność z przyszłością GPT-5.4, bezpośrednią normalizację OpenAI
  `openai-completions` -> `openai-responses`, wskazówki auth uwzględniające Codex,
  tłumienie Spark, syntetyczne wiersze list OpenAI oraz politykę thinking /
  modeli live dla GPT-5; rodzina streamów `openai-responses-defaults` obsługuje
  współdzielone natywne wrappery OpenAI Responses dla nagłówków atrybucji,
  `/fast`/`serviceTier`, szczegółowości tekstu, natywnego wyszukiwania w sieci Codex,
  kształtowania ładunku reasoning-compat oraz zarządzania kontekstem Responses.
- OpenRouter używa `catalog` oraz `resolveDynamicModel` i
  `prepareDynamicModel`, ponieważ dostawca jest pass-through i może wystawiać nowe
  identyfikatory modeli przed aktualizacją statycznego katalogu OpenClaw; używa też
  `capabilities`, `wrapStreamFn` i `isCacheTtlEligible`, aby utrzymać
  nagłówki żądań specyficzne dla dostawcy, metadane routingu, łatki reasoning i politykę prompt-cache
  poza core. Jego polityka replay pochodzi z rodziny
  `passthrough-gemini`, a rodzina streamów `openrouter-thinking` obsługuje
  wstrzykiwanie reasoning proxy oraz pomijanie nieobsługiwanych modeli i `auto`.
- GitHub Copilot używa `catalog`, `auth`, `resolveDynamicModel` i
  `capabilities` oraz `prepareRuntimeAuth` i `fetchUsageSnapshot`, ponieważ
  potrzebuje własnego logowania urządzenia, zachowania fallback modeli, niuansów transkryptów Claude,
  wymiany tokenu GitHub -> token Copilot oraz własnego endpointu użycia.
- OpenAI Codex używa `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` i `augmentModelCatalog` oraz
  `prepareExtraParams`, `resolveUsageAuth` i `fetchUsageSnapshot`, ponieważ
  nadal działa na transportach OpenAI core, ale obsługuje własną normalizację transportu/base URL,
  politykę fallback odświeżania OAuth, domyślny wybór transportu,
  syntetyczne wiersze katalogu Codex oraz integrację z endpointem użycia ChatGPT; współdzieli
  tę samą rodzinę streamów `openai-responses-defaults` co bezpośredni OpenAI.
- Google AI Studio i Gemini CLI OAuth używają `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` i `isModernModelRef`, ponieważ rodzina replay
  `google-gemini` obsługuje fallback zgodności z przyszłością Gemini 3.1,
  natywną walidację replay Gemini, sanitację replay bootstrap, tagowany tryb
  reasoning-output i dopasowywanie nowoczesnych modeli, natomiast rodzina streamów
  `google-thinking` obsługuje normalizację ładunku thinking Gemini;
  Gemini CLI OAuth używa również `formatApiKey`, `resolveUsageAuth` i
  `fetchUsageSnapshot` do formatowania tokenu, parsowania tokenu i podłączenia
  endpointu kwot.
- Anthropic Vertex używa `buildReplayPolicy` przez rodzinę replay
  `anthropic-by-model`, aby czyszczenie replay specyficzne dla Claude pozostawało
  ograniczone do identyfikatorów Claude zamiast każdego transportu `anthropic-messages`.
- Amazon Bedrock używa `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` i `resolveDefaultThinkingLevel`, ponieważ obsługuje
  klasyfikację błędów throttle/not-ready/context-overflow specyficzną dla Bedrock
  dla ruchu Anthropic-on-Bedrock; jego polityka replay nadal współdzieli to samo
  zabezpieczenie `anthropic-by-model` tylko dla Claude.
- OpenRouter, Kilocode, Opencode i Opencode Go używają `buildReplayPolicy`
  przez rodzinę replay `passthrough-gemini`, ponieważ proxy'ują modele Gemini
  przez transporty kompatybilne z OpenAI i potrzebują sanitacji
  thought-signature Gemini bez natywnej walidacji replay Gemini lub
  przepisań bootstrap.
- MiniMax używa `buildReplayPolicy` przez rodzinę replay
  `hybrid-anthropic-openai`, ponieważ jeden dostawca obsługuje semantykę zarówno
  Anthropic-message, jak i kompatybilną z OpenAI; utrzymuje usuwanie bloków
  thinking tylko dla Claude po stronie Anthropic, jednocześnie nadpisując tryb reasoning
  output z powrotem na natywny, a rodzina streamów `minimax-fast-mode` obsługuje
  przepisywanie modeli fast-mode na współdzielonej ścieżce stream.
- Moonshot używa `catalog` oraz `wrapStreamFn`, ponieważ nadal korzysta ze współdzielonego
  transportu OpenAI, ale potrzebuje normalizacji ładunku thinking należącej do dostawcy; rodzina
  streamów `moonshot-thinking` mapuje konfigurację oraz stan `/think` na
  natywny binarny ładunek thinking.
- Kilocode używa `catalog`, `capabilities`, `wrapStreamFn` i
  `isCacheTtlEligible`, ponieważ potrzebuje nagłówków żądań należących do dostawcy,
  normalizacji ładunku reasoning, wskazówek dla transkryptów Gemini oraz bramkowania
  cache-TTL dla Anthropic; rodzina streamów `kilocode-thinking` utrzymuje
  wstrzykiwanie Kilo thinking na współdzielonej ścieżce stream proxy, pomijając `kilo/auto`
  i inne identyfikatory modeli proxy, które nie wspierają jawnych ładunków reasoning.
- Z.AI używa `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` i `fetchUsageSnapshot`, ponieważ obsługuje fallback GLM-5,
  domyślne `tool_stream`, UX binarnego thinking, dopasowywanie nowoczesnych modeli oraz
  zarówno auth użycia, jak i pobieranie kwot; rodzina streamów `tool-stream-default-on`
  utrzymuje wrapper `tool_stream` włączony domyślnie poza ręcznie pisanym klejem dla poszczególnych dostawców.
- xAI używa `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` i `isModernModelRef`,
  ponieważ obsługuje natywną normalizację transportu xAI Responses, przepisywanie aliasów
  Grok fast-mode, domyślne `tool_stream`, porządkowanie strict-tool / reasoning-payload,
  ponowne użycie fallback auth dla narzędzi należących do pluginu, zgodne z przyszłością rozstrzyganie modeli Grok
  oraz łatki kompatybilności należące do dostawcy, takie jak profil schematu narzędzi xAI,
  nieobsługiwane słowa kluczowe schematu, natywne `web_search` i dekodowanie argumentów
  wywołań narzędzi z encji HTML.
- Mistral, OpenCode Zen i OpenCode Go używają wyłącznie `capabilities`, aby
  utrzymać niuanse transkryptów/narzędzi poza core.
- Dołączone providery tylko-katalogowe, takie jak `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` i `volcengine`, używają
  wyłącznie `catalog`.
- Qwen używa `catalog` dla swojego dostawcy tekstu oraz współdzielonych rejestracji
  media-understanding i video-generation dla swoich multimodalnych powierzchni.
- MiniMax i Xiaomi używają `catalog` oraz hooków użycia, ponieważ ich zachowanie `/usage`
  należy do pluginu, mimo że inferencja nadal działa przez współdzielone transporty.

## Helpery runtime

Pluginy mogą uzyskiwać dostęp do wybranych helperów core przez `api.runtime`. Dla TTS:

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

- `textToSpeech` zwraca standardowy ładunek wyjściowy TTS core dla powierzchni plików/notatek głosowych.
- Używa konfiguracji core `messages.tts` i wyboru dostawcy.
- Zwraca bufor audio PCM + częstotliwość próbkowania. Pluginy muszą wykonać resampling/kodowanie dla dostawców.
- `listVoices` jest opcjonalne per dostawca. Używaj tego dla selektorów głosów lub przepływów konfiguracji należących do dostawcy.
- Listy głosów mogą zawierać bogatsze metadane, takie jak locale, gender i tagi personality dla selektorów świadomych dostawcy.
- OpenAI i ElevenLabs wspierają dziś telefonię. Microsoft nie.

Pluginy mogą również rejestrować dostawców mowy przez `api.registerSpeechProvider(...)`.

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

- Utrzymuj politykę TTS, fallback i dostarczanie odpowiedzi w core.
- Używaj dostawców mowy dla zachowań syntezy należących do dostawcy.
- Legacy wartość wejściowa Microsoft `edge` jest normalizowana do identyfikatora dostawcy `microsoft`.
- Preferowany model własności jest zorientowany firmowo: jeden plugin dostawcy może obsługiwać
  tekst, mowę, obraz i przyszłych dostawców mediów, gdy OpenClaw doda ich
  kontrakty capability.

Dla rozumienia obrazów/audio/wideo pluginy rejestrują jednego typowanego
dostawcę media-understanding zamiast ogólnej torby klucz/wartość:

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

- Utrzymuj orkiestrację, fallback, konfigurację i podłączenie kanałów w core.
- Utrzymuj zachowanie dostawcy w pluginie dostawcy.
- Rozszerzanie addytywne powinno pozostać typowane: nowe opcjonalne metody, nowe opcjonalne
  pola wyników, nowe opcjonalne capabilities.
- Generowanie wideo już podąża tym samym wzorcem:
  - core obsługuje kontrakt capability i helper runtime
  - pluginy dostawców rejestrują `api.registerVideoGenerationProvider(...)`
  - pluginy funkcjonalne/kanałów korzystają z `api.runtime.videoGeneration.*`

Dla helperów runtime media-understanding pluginy mogą wywoływać:

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

Dla transkrypcji audio pluginy mogą używać albo runtime media-understanding,
albo starszego aliasu STT:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Uwagi:

- `api.runtime.mediaUnderstanding.*` jest preferowaną współdzieloną powierzchnią dla
  rozumienia obrazów/audio/wideo.
- Używa konfiguracji audio media-understanding core (`tools.media.audio`) oraz kolejności fallback dostawców.
- Zwraca `{ text: undefined }`, gdy nie zostanie wygenerowany wynik transkrypcji (na przykład dla pominiętego/nieobsługiwanego wejścia).
- `api.runtime.stt.transcribeAudioFile(...)` pozostaje aliasem kompatybilności.

Pluginy mogą również uruchamiać podrzędne przebiegi subagentów w tle przez `api.runtime.subagent`:

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

- `provider` i `model` są opcjonalnymi nadpisaniami dla pojedynczego uruchomienia, a nie trwałymi zmianami sesji.
- OpenClaw honoruje te pola nadpisania wyłącznie dla zaufanych wywołujących.
- Dla przebiegów fallback należących do pluginu operatorzy muszą jawnie wyrazić zgodę przez `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Użyj `plugins.entries.<id>.subagent.allowedModels`, aby ograniczyć zaufane pluginy do konkretnych kanonicznych celów `provider/model`, albo `"*"`, aby jawnie dopuścić dowolny cel.
- Niezaufane przebiegi subagentów nadal działają, ale żądania nadpisania są odrzucane zamiast cicho przechodzić do fallback.

Dla web search pluginy mogą korzystać ze współdzielonego helpera runtime zamiast
sięgać do podłączenia narzędzia agenta:

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

Pluginy mogą również rejestrować dostawców web-search przez
`api.registerWebSearchProvider(...)`.

Uwagi:

- Utrzymuj wybór dostawcy, rozstrzyganie poświadczeń i współdzieloną semantykę żądań w core.
- Używaj dostawców web-search dla transportów wyszukiwania specyficznych dla dostawcy.
- `api.runtime.webSearch.*` jest preferowaną współdzieloną powierzchnią dla pluginów funkcjonalnych/kanałów, które potrzebują zachowania wyszukiwania bez zależności od wrappera narzędzia agenta.

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

- `generate(...)`: wygeneruj obraz przy użyciu skonfigurowanego łańcucha dostawców generowania obrazów.
- `listProviders(...)`: wyświetl dostępnych dostawców generowania obrazów i ich capabilities.

## Trasy HTTP Gateway

Pluginy mogą wystawiać endpointy HTTP przez `api.registerHttpRoute(...)`.

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

- `path`: ścieżka trasy pod serwerem HTTP gateway.
- `auth`: wymagane. Użyj `"gateway"`, aby wymagać zwykłego auth gateway, albo `"plugin"` dla auth zarządzanego przez plugin / weryfikacji webhooka.
- `match`: opcjonalne. `"exact"` (domyślnie) albo `"prefix"`.
- `replaceExisting`: opcjonalne. Pozwala temu samemu pluginowi zastąpić własną istniejącą rejestrację trasy.
- `handler`: zwróć `true`, gdy trasa obsłużyła żądanie.

Uwagi:

- `api.registerHttpHandler(...)` zostało usunięte i spowoduje błąd ładowania pluginu. Używaj `api.registerHttpRoute(...)`.
- Trasy pluginów muszą jawnie deklarować `auth`.
- Konflikty dokładnego `path + match` są odrzucane, chyba że ustawiono `replaceExisting: true`, i jeden plugin nie może zastąpić trasy innego pluginu.
- Nakładające się trasy z różnymi poziomami `auth` są odrzucane. Utrzymuj łańcuchy przejścia `exact`/`prefix` tylko na tym samym poziomie auth.
- Trasy `auth: "plugin"` **nie** otrzymują automatycznie zakresów runtime operatora. Służą do webhooków / weryfikacji podpisów zarządzanych przez plugin, a nie do uprzywilejowanych wywołań helperów Gateway.
- Trasy `auth: "gateway"` działają wewnątrz zakresu runtime żądania Gateway, ale ten zakres jest celowo zachowawczy:
  - bearer auth oparty na współdzielonym sekrecie (`gateway.auth.mode = "token"` / `"password"`) utrzymuje zakresy runtime tras pluginów przypięte do `operator.write`, nawet jeśli wywołujący wysyła `x-openclaw-scopes`
  - zaufane tryby HTTP z tożsamością (na przykład `trusted-proxy` albo `gateway.auth.mode = "none"` na prywatnym ingressie) honorują `x-openclaw-scopes` tylko wtedy, gdy nagłówek jest jawnie obecny
  - jeśli w takich żądaniach tras pluginów z tożsamością brakuje `x-openclaw-scopes`, zakres runtime wraca do `operator.write`
- Praktyczna zasada: nie zakładaj, że trasa pluginu z auth gateway jest domyślnie powierzchnią administracyjną. Jeśli trasa wymaga zachowania tylko dla administratora, wymagaj trybu auth z tożsamością i udokumentuj jawny kontrakt nagłówka `x-openclaw-scopes`.

## Ścieżki importu Plugin SDK

Przy tworzeniu pluginów używaj subścieżek SDK zamiast monolitycznego importu `openclaw/plugin-sdk`:

- `openclaw/plugin-sdk/plugin-entry` dla prymitywów rejestracji pluginów.
- `openclaw/plugin-sdk/core` dla ogólnego współdzielonego kontraktu skierowanego do pluginów.
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
  `openclaw/plugin-sdk/secret-input` i
  `openclaw/plugin-sdk/webhook-ingress` dla współdzielonego podłączania
  konfiguracji/auth/odpowiedzi/webhooków. `channel-inbound` jest współdzielonym domem dla debounce, dopasowywania mention,
  helperów polityki mention przychodzących, formatowania kopert oraz helperów kontekstu kopert przychodzących.
  `channel-setup` to wąska granica konfiguracji opcjonalnej instalacji.
  `setup-runtime` to bezpieczna w runtime powierzchnia konfiguracji używana przez `setupEntry` /
  odroczony start, w tym bezpieczne importowo adaptery łatek konfiguracji.
  `setup-adapter-runtime` to granica adaptera konfiguracji kont świadoma env.
  `setup-tools` to mała granica helperów CLI/archiwów/dokumentacji (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Subścieżki domenowe, takie jak `openclaw/plugin-sdk/channel-config-helpers`,
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
  `openclaw/plugin-sdk/runtime-store` i
  `openclaw/plugin-sdk/directory-runtime` dla współdzielonych helperów runtime/konfiguracji.
  `telegram-command-config` to wąska publiczna granica dla normalizacji/walidacji niestandardowych komend Telegram
  i pozostaje dostępna nawet wtedy, gdy dołączona powierzchnia kontraktu Telegram jest tymczasowo niedostępna.
  `text-runtime` to współdzielona granica tekst/markdown/logowanie, obejmująca
  usuwanie tekstu widocznego dla asystenta, helpery renderowania/dzielenia markdown,
  helpery redakcji, helpery tagów dyrektyw i bezpieczne narzędzia tekstowe.
- Granice kanałów specyficzne dla zatwierdzeń powinny preferować jeden kontrakt
  `approvalCapability` na pluginie. Core odczytuje wtedy auth zatwierdzeń, dostarczanie, renderowanie,
  routing natywny i leniwe zachowanie natywnych handlerów przez tę jedną capability
  zamiast mieszać zachowanie zatwierdzeń z niepowiązanymi polami pluginu.
- `openclaw/plugin-sdk/channel-runtime` jest przestarzałe i pozostaje jedynie jako
  shim kompatybilności dla starszych pluginów. Nowy kod powinien importować węższe
  ogólne prymitywy, a kod repo nie powinien dodawać nowych importów tego shima.
- Wnętrza dołączonych rozszerzeń pozostają prywatne. Pluginy zewnętrzne powinny używać wyłącznie
  subścieżek `openclaw/plugin-sdk/*`. Kod/testy core OpenClaw mogą używać publicznych punktów wejścia repo
  pod katalogiem głównym pakietu pluginu, takich jak `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` oraz wąsko wyspecjalizowanych plików takich jak
  `login-qr-api.js`. Nigdy nie importuj `src/*` pakietu pluginu z core ani z innego rozszerzenia.
- Podział punktów wejścia repo:
  `<plugin-package-root>/api.js` jest barrelem helperów/typów,
  `<plugin-package-root>/runtime-api.js` jest barrelem tylko runtime,
  `<plugin-package-root>/index.js` jest punktem wejścia dołączonego pluginu,
  a `<plugin-package-root>/setup-entry.js` jest punktem wejścia pluginu konfiguracji.
- Aktualne przykłady dołączonych dostawców:
  - Anthropic używa `api.js` / `contract-api.js` dla helperów strumieni Claude, takich
    jak `wrapAnthropicProviderStream`, helpery nagłówków beta i parsowanie `service_tier`.
  - OpenAI używa `api.js` dla builderów providerów, helperów modeli domyślnych i
    builderów providerów realtime.
  - OpenRouter używa `api.js` dla swojego buildera providera oraz helperów onboardingu/konfiguracji,
    podczas gdy `register.runtime.js` może nadal re-eksportować ogólne
    helpery `plugin-sdk/provider-stream` do użytku lokalnego w repo.
- Publiczne punkty wejścia ładowane przez fasadę preferują aktywny snapshot konfiguracji runtime,
  gdy taki istnieje, a następnie wracają do rozstrzygniętego pliku konfiguracji na dysku, gdy
  OpenClaw nie udostępnia jeszcze snapshotu runtime.
- Ogólne współdzielone prymitywy pozostają preferowanym publicznym kontraktem SDK. Nadal istnieje mały
  zastrzeżony zestaw granic helperów dołączonych kanałów oznaczonych marką kanału. Traktuj je jako
  granice utrzymaniowe/kompatybilnościowe dołączonych pluginów, a nie nowe cele importu dla stron trzecich; nowe kontrakty międzykanałowe nadal powinny trafiać do
  ogólnych subścieżek `plugin-sdk/*` albo do lokalnych barrelów `api.js` /
  `runtime-api.js` pluginu.

Uwaga dotycząca kompatybilności:

- Dla nowego kodu unikaj głównego barrelu `openclaw/plugin-sdk`.
- Najpierw preferuj wąskie stabilne prymitywy. Nowsze subścieżki setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool są docelowym kontraktem dla nowych
  prac nad dołączonymi i zewnętrznymi pluginami.
  Parsowanie/dopasowywanie targetów należy do `openclaw/plugin-sdk/channel-targets`.
  Bramki działań wiadomości i helpery identyfikatorów wiadomości reakcji należą do
  `openclaw/plugin-sdk/channel-actions`.
- Barrele helperów specyficznych dla dołączonych rozszerzeń nie są domyślnie stabilne. Jeśli
  helper jest potrzebny tylko dołączonemu rozszerzeniu, trzymaj go za lokalną granicą
  `api.js` lub `runtime-api.js` rozszerzenia zamiast promować go do
  `openclaw/plugin-sdk/<extension>`.
- Nowe współdzielone granice helperów powinny być ogólne, a nie oznaczone marką kanału. Współdzielone parsowanie
  targetów należy do `openclaw/plugin-sdk/channel-targets`; wnętrza specyficzne dla kanału
  pozostają za lokalną granicą `api.js` lub `runtime-api.js` należącego pluginu.
- Subścieżki specyficzne dla capability, takie jak `image-generation`,
  `media-understanding` i `speech`, istnieją, ponieważ dołączone/natywne pluginy używają
  ich dziś. Ich obecność sama w sobie nie oznacza, że każdy eksportowany helper jest
  długoterminowym zamrożonym kontraktem zewnętrznym.

## Schematy narzędzia wiadomości

Pluginy powinny obsługiwać wkłady do schematu `describeMessageTool(...)`
specyficzne dla kanału. Pola specyficzne dla dostawcy trzymaj w pluginie, a nie we współdzielonym core.

Dla współdzielonych, przenośnych fragmentów schematu używaj ponownie ogólnych helperów eksportowanych przez
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` dla ładunków w stylu siatki przycisków
- `createMessageToolCardSchema()` dla ustrukturyzowanych ładunków kart

Jeśli dany kształt schematu ma sens tylko dla jednego dostawcy, zdefiniuj go w
kodzie źródłowym tego pluginu zamiast promować go do współdzielonego SDK.

## Rozstrzyganie targetów kanałów

Pluginy kanałów powinny obsługiwać semantykę targetów specyficzną dla kanału. Utrzymuj
wspólny host outbound jako ogólny i używaj powierzchni adaptera komunikacji dla reguł dostawcy:

- `messaging.inferTargetChatType({ to })` decyduje, czy znormalizowany target
  ma być traktowany jako `direct`, `group` czy `channel` przed wyszukaniem w katalogu.
- `messaging.targetResolver.looksLikeId(raw, normalized)` informuje core, czy dane
  wejście powinno od razu przejść do rozstrzygania podobnego do identyfikatora zamiast do wyszukiwania w katalogu.
- `messaging.targetResolver.resolveTarget(...)` jest fallbackiem pluginu, gdy
  core potrzebuje końcowego rozstrzygnięcia należącego do dostawcy po normalizacji albo po
  nieudanym wyszukaniu w katalogu.
- `messaging.resolveOutboundSessionRoute(...)` obsługuje budowę trasy sesji specyficznej dla dostawcy po rozstrzygnięciu targetu.

Zalecany podział:

- Używaj `inferTargetChatType` dla decyzji kategorialnych, które powinny zapaść przed
  wyszukiwaniem peerów/grup.
- Używaj `looksLikeId` dla sprawdzeń typu „traktuj to jako jawny/natywny identyfikator targetu”.
- Używaj `resolveTarget` jako fallbacku normalizacji specyficznego dla dostawcy, a nie do
  szerokiego wyszukiwania katalogowego.
- Natywne identyfikatory dostawcy, takie jak chat ids, thread ids, JIDs, handles i room
  ids, trzymaj w wartościach `target` albo parametrach specyficznych dla dostawcy, a nie w ogólnych polach SDK.

## Katalogi oparte na konfiguracji

Pluginy, które wyprowadzają wpisy katalogowe z konfiguracji, powinny utrzymywać tę logikę w
pluginie i używać ponownie współdzielonych helperów z
`openclaw/plugin-sdk/directory-runtime`.

Używaj tego, gdy kanał potrzebuje peerów/grup opartych na konfiguracji, takich jak:

- peery DM oparte na allowliście
- skonfigurowane mapy kanałów/grup
- statyczne fallbacki katalogowe zależne od konta

Współdzielone helpery w `directory-runtime` obsługują tylko ogólne operacje:

- filtrowanie zapytań
- stosowanie limitów
- helpery deduplikacji/normalizacji
- budowanie `ChannelDirectoryEntry[]`

Inspekcja kont i normalizacja identyfikatorów specyficzna dla kanału powinna pozostać w implementacji pluginu.

## Katalogi dostawców

Pluginy dostawców mogą definiować katalogi modeli do inferencji przez
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` zwraca ten sam kształt, który OpenClaw zapisuje do
`models.providers`:

- `{ provider }` dla jednego wpisu dostawcy
- `{ providers }` dla wielu wpisów dostawców

Używaj `catalog`, gdy plugin obsługuje identyfikatory modeli specyficzne dla dostawcy, domyślne wartości base URL
lub metadane modeli zależne od auth.

`catalog.order` kontroluje, kiedy katalog pluginu jest scalany względem wbudowanych
niejawnych providerów OpenClaw:

- `simple`: zwykli dostawcy opierający się na kluczu API lub env
- `profile`: dostawcy pojawiający się, gdy istnieją profile auth
- `paired`: dostawcy syntetyzujący wiele powiązanych wpisów providerów
- `late`: ostatni przebieg, po innych niejawnych dostawcach

Późniejsze providery wygrywają przy kolizji kluczy, więc pluginy mogą celowo nadpisywać
wbudowany wpis providera o tym samym identyfikatorze.

Kompatybilność:

- `discovery` nadal działa jako legacy alias
- jeśli zarejestrowano zarówno `catalog`, jak i `discovery`, OpenClaw używa `catalog`

## Odczytowa inspekcja kanałów

Jeśli Twój plugin rejestruje kanał, preferuj implementację
`plugin.config.inspectAccount(cfg, accountId)` obok `resolveAccount(...)`.

Dlaczego:

- `resolveAccount(...)` jest ścieżką runtime. Może zakładać, że poświadczenia
  są w pełni zmaterializowane, i może szybko zwracać błąd, gdy brakuje wymaganych sekretów.
- Ścieżki komend tylko do odczytu, takie jak `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` oraz przepływy napraw doctor/config
  nie powinny wymagać materializacji poświadczeń runtime tylko po to, by opisać konfigurację.

Zalecane zachowanie `inspectAccount(...)`:

- Zwracaj tylko opisowy stan konta.
- Zachowuj `enabled` i `configured`.
- Dołączaj pola źródła/statusu poświadczeń, gdy to istotne, takie jak:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Nie musisz zwracać surowych wartości tokenów tylko po to, aby zgłaszać dostępność
  w trybie tylko do odczytu. Wystarczy zwrócić `tokenStatus: "available"` (oraz odpowiadające pole źródła).
- Używaj `configured_unavailable`, gdy poświadczenie jest skonfigurowane przez SecretRef, ale
  niedostępne w bieżącej ścieżce komendy.

Dzięki temu komendy tylko do odczytu mogą zgłaszać „skonfigurowane, ale niedostępne w tej ścieżce komendy”
zamiast powodować awarię albo błędnie raportować konto jako nieskonfigurowane.

## Package packs

Katalog pluginu może zawierać `package.json` z `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Każdy wpis staje się pluginem. Jeśli paczka zawiera wiele rozszerzeń, identyfikator pluginu
przyjmuje postać `name/<fileBase>`.

Jeśli Twój plugin importuje zależności npm, zainstaluj je w tym katalogu, aby
`node_modules` było dostępne (`npm install` / `pnpm install`).

Zabezpieczenie bezpieczeństwa: każdy wpis `openclaw.extensions` musi pozostać wewnątrz katalogu pluginu
po rozstrzygnięciu symlinków. Wpisy wychodzące poza katalog pakietu są
odrzucane.

Uwaga bezpieczeństwa: `openclaw plugins install` instaluje zależności pluginu przez
`npm install --omit=dev --ignore-scripts` (bez skryptów lifecycle i bez zależności dev w runtime). Utrzymuj drzewa zależności pluginów jako „czyste JS/TS” i unikaj pakietów, które wymagają buildów `postinstall`.

Opcjonalnie: `openclaw.setupEntry` może wskazywać lekki moduł tylko-konfiguracyjny.
Gdy OpenClaw potrzebuje powierzchni konfiguracji dla wyłączonego pluginu kanału albo
gdy plugin kanału jest włączony, ale nadal nieskonfigurowany, ładuje `setupEntry`
zamiast pełnego entry pluginu. Dzięki temu uruchamianie i konfiguracja są lżejsze,
gdy główne entry pluginu podłącza też narzędzia, hooki lub inny kod wyłącznie runtime.

Opcjonalnie: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
pozwala pluginowi kanału skorzystać z tej samej ścieżki `setupEntry` podczas
fazy uruchamiania gateway przed nasłuchem, nawet gdy kanał jest już skonfigurowany.

Używaj tego tylko wtedy, gdy `setupEntry` w pełni pokrywa powierzchnię startową, która musi istnieć
przed rozpoczęciem nasłuchiwania przez gateway. W praktyce oznacza to, że entry konfiguracji
musi rejestrować każdą capability należącą do kanału, od której zależy start, taką jak:

- sama rejestracja kanału
- wszelkie trasy HTTP, które muszą być dostępne przed rozpoczęciem nasłuchiwania przez gateway
- wszelkie metody gateway, narzędzia lub usługi, które muszą istnieć w tym samym oknie czasu

Jeśli Twoje pełne entry nadal obsługuje jakąkolwiek wymaganą capability startową, nie włączaj
tej flagi. Zachowaj domyślne zachowanie pluginu i pozwól OpenClaw ładować
pełne entry podczas startu.

Dołączone kanały mogą także publikować helpery powierzchni kontraktowej tylko-konfiguracyjnej, z których core
może korzystać przed załadowaniem pełnego runtime kanału. Aktualna powierzchnia promowania konfiguracji to:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Core używa tej powierzchni, gdy musi wypromować legacy konfigurację kanału z jednym kontem do
`channels.<id>.accounts.*` bez ładowania pełnego entry pluginu.
Matrix jest aktualnym dołączonym przykładem: przenosi tylko klucze auth/bootstrap do
nazwanego wypromowanego konta, gdy nazwane konta już istnieją, i może zachować
skonfigurowany niekanoniczny klucz konta domyślnego zamiast zawsze tworzyć
`accounts.default`.

Te adaptery łatek konfiguracji utrzymują leniwe wykrywanie powierzchni kontraktowej dołączonych pluginów.
Czas importu pozostaje niski; powierzchnia promowania jest ładowana dopiero przy pierwszym użyciu, zamiast
ponownie wchodzić w start dołączonego kanału przy imporcie modułu.

Gdy te powierzchnie startowe zawierają metody Gateway RPC, trzymaj je pod
prefiksem specyficznym dla pluginu. Przestrzenie nazw administracyjnych core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) pozostają zastrzeżone i zawsze są rozstrzygane
do `operator.admin`, nawet jeśli plugin żąda węższego zakresu.

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

### Metadane katalogu kanałów

Pluginy kanałów mogą reklamować metadane konfiguracji/wykrywania przez `openclaw.channel` oraz
wskazówki instalacyjne przez `openclaw.install`. Dzięki temu dane katalogu core pozostają puste.

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
- `preferOver`: identyfikatory pluginów/kanałów o niższym priorytecie, które ten wpis katalogu powinien wyprzedzać
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: kontrolki treści dla powierzchni wyboru
- `markdownCapable`: oznacza kanał jako obsługujący markdown dla decyzji o formatowaniu outbound
- `exposure.configured`: ukrywa kanał z powierzchni listy skonfigurowanych kanałów po ustawieniu na `false`
- `exposure.setup`: ukrywa kanał z interaktywnych selektorów konfiguracji/ustawiania po ustawieniu na `false`
- `exposure.docs`: oznacza kanał jako wewnętrzny/prywatny dla powierzchni nawigacji dokumentacji
- `showConfigured` / `showInSetup`: legacy aliasy nadal akceptowane ze względów kompatybilności; preferuj `exposure`
- `quickstartAllowFrom`: włącza kanał do standardowego przepływu quickstart `allowFrom`
- `forceAccountBinding`: wymaga jawnego powiązania konta nawet wtedy, gdy istnieje tylko jedno konto
- `preferSessionLookupForAnnounceTarget`: preferuje wyszukiwanie sesji przy rozstrzyganiu targetów ogłoszeń

OpenClaw może także scalać **zewnętrzne katalogi kanałów** (na przykład eksport rejestru MPM).
Umieść plik JSON w jednej z lokalizacji:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Albo wskaż `OPENCLAW_PLUGIN_CATALOG_PATHS` (lub `OPENCLAW_MPM_CATALOG_PATHS`) na
jeden lub więcej plików JSON (rozdzielonych przecinkiem/średnikiem/formatem `PATH`). Każdy plik powinien
zawierać `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. Parser akceptuje także `"packages"` lub `"plugins"` jako legacy aliasy klucza `"entries"`.

## Pluginy silnika kontekstu

Pluginy silnika kontekstu obsługują orkiestrację kontekstu sesji dla ingest, składania
i kompaktowania. Rejestruj je ze swojego pluginu przez
`api.registerContextEngine(id, factory)`, a następnie wybierz aktywny silnik przez
`plugins.slots.contextEngine`.

Używaj tego, gdy Twój plugin musi zastąpić lub rozszerzyć domyślny pipeline kontekstu,
a nie tylko dodać wyszukiwanie pamięci lub hooki.

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

Jeśli Twój silnik **nie** obsługuje algorytmu kompaktowania, pozostaw `compact()`
zaimplementowane i jawnie je deleguj:

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

## Dodawanie nowej capability

Gdy plugin potrzebuje zachowania, które nie mieści się w obecnym API, nie omijaj
systemu pluginów prywatnym reach-in. Dodaj brakującą capability.

Zalecana sekwencja:

1. zdefiniuj kontrakt core
   Ustal, jakie współdzielone zachowanie powinien obsługiwać core: politykę, fallback, łączenie konfiguracji,
   cykl życia, semantykę skierowaną do kanałów i kształt helperów runtime.
2. dodaj typowane powierzchnie rejestracji/runtime pluginu
   Rozszerz `OpenClawPluginApi` i/lub `api.runtime` o najmniejszą użyteczną
   typowaną powierzchnię capability.
3. podłącz konsumentów core + kanałów/funkcji
   Kanały i pluginy funkcjonalne powinny korzystać z nowej capability przez core,
   a nie importować bezpośrednio implementacji dostawcy.
4. zarejestruj implementacje dostawców
   Pluginy dostawców rejestrują następnie swoje backendy względem tej capability.
5. dodaj pokrycie kontraktowe
   Dodaj testy, aby własność i kształt rejestracji pozostawały z czasem jawne.

Tak OpenClaw zachowuje opiniotwórczość bez twardego zakodowania światopoglądu jednego
dostawcy. Zobacz [Capability Cookbook](/pl/plugins/architecture),
aby znaleźć konkretną listę plików i opracowany przykład.

### Lista kontrolna capability

Gdy dodajesz nową capability, implementacja zwykle powinna dotykać tych
powierzchni jednocześnie:

- typy kontraktu core w `src/<capability>/types.ts`
- helper runner/runtime core w `src/<capability>/runtime.ts`
- powierzchnia rejestracji API pluginu w `src/plugins/types.ts`
- podłączenie rejestru pluginów w `src/plugins/registry.ts`
- udostępnienie runtime pluginów w `src/plugins/runtime/*`, gdy pluginy funkcjonalne/kanałów
  muszą z niego korzystać
- helpery przechwytywania/testów w `src/test-utils/plugin-registration.ts`
- asercje własności/kontraktu w `src/plugins/contracts/registry.ts`
- dokumentacja operatora/pluginów w `docs/`

Jeśli którejś z tych powierzchni brakuje, zwykle jest to znak, że capability nie jest
jeszcze w pełni zintegrowana.

### Szablon capability

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

Wzorzec testu kontraktu:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

To utrzymuje prostą zasadę:

- core obsługuje kontrakt capability + orkiestrację
- pluginy dostawców obsługują implementacje dostawców
- pluginy funkcjonalne/kanałów korzystają z helperów runtime
- testy kontraktów utrzymują jawną własność
