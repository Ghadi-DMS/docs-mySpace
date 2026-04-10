---
read_when:
    - Dodawanie automatyzacji przeglądarki sterowanej przez agenta
    - Debugowanie, dlaczego openclaw zakłóca działanie Twojego Chrome
    - Implementowanie ustawień i cyklu życia przeglądarki w aplikacji macOS
summary: Zintegrowana usługa sterowania przeglądarką + polecenia akcji
title: Przeglądarka (zarządzana przez OpenClaw)
x-i18n:
    generated_at: "2026-04-10T09:45:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: cd3424f62178bbf25923b8bc8e4d9f70e330f35428d01fe153574e5fa45d7604
    source_path: tools/browser.md
    workflow: 15
---

# Przeglądarka (zarządzana przez openclaw)

OpenClaw może uruchamiać **dedykowany profil Chrome/Brave/Edge/Chromium**, który agent kontroluje.
Jest on odizolowany od Twojej osobistej przeglądarki i zarządzany przez małą lokalną
usługę sterowania wewnątrz Gateway (tylko loopback).

Ujęcie dla początkujących:

- Potraktuj to jako **oddzielną przeglądarkę tylko dla agenta**.
- Profil `openclaw` **nie** dotyka Twojego osobistego profilu przeglądarki.
- Agent może **otwierać karty, odczytywać strony, klikać i wpisywać tekst** w bezpiecznej ścieżce.
- Wbudowany profil `user` dołącza do Twojej prawdziwej zalogowanej sesji Chrome przez Chrome MCP.

## Co otrzymujesz

- Oddzielny profil przeglądarki o nazwie **openclaw** (domyślnie z pomarańczowym akcentem).
- Deterministyczne sterowanie kartami (lista/otwieranie/fokus/zamykanie).
- Akcje agenta (kliknięcie/wpisywanie/przeciąganie/zaznaczanie), snapshoty, zrzuty ekranu, pliki PDF.
- Opcjonalną obsługę wielu profili (`openclaw`, `work`, `remote`, ...).

Ta przeglądarka **nie** jest Twoją codzienną przeglądarką. To bezpieczna, odizolowana powierzchnia do
automatyzacji i weryfikacji przez agenta.

## Szybki start

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Jeśli pojawia się komunikat „Browser disabled”, włącz ją w konfiguracji (patrz niżej) i uruchom ponownie
Gateway.

Jeśli `openclaw browser` całkowicie nie istnieje albo agent mówi, że narzędzie przeglądarki
jest niedostępne, przejdź do [Brak polecenia lub narzędzia browser](/pl/tools/browser#missing-browser-command-or-tool).

## Sterowanie przez plugin

Domyślne narzędzie `browser` jest teraz dołączonym pluginem, który jest
domyślnie włączony. Oznacza to, że możesz go wyłączyć lub zastąpić bez usuwania reszty
systemu pluginów OpenClaw:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

Wyłącz dołączony plugin przed zainstalowaniem innego pluginu, który udostępnia
tę samą nazwę narzędzia `browser`. Domyślne środowisko przeglądarki wymaga obu elementów:

- `plugins.entries.browser.enabled` nie może być wyłączone
- `browser.enabled=true`

Jeśli wyłączysz tylko plugin, dołączone CLI przeglądarki (`openclaw browser`),
metoda gateway (`browser.request`), narzędzie agenta i domyślna usługa sterowania przeglądarką
znikną razem. Twoja konfiguracja `browser.*` pozostanie nienaruszona, aby mogła zostać
ponownie użyta przez plugin zastępczy.

Dołączony plugin przeglądarki jest teraz również właścicielem implementacji runtime przeglądarki.
Rdzeń zachowuje tylko współdzielone pomocniki Plugin SDK oraz re-eksporty zgodności dla
starszych wewnętrznych ścieżek importu. W praktyce usunięcie lub zastąpienie pakietu pluginu
przeglądarki usuwa zestaw funkcji przeglądarki zamiast pozostawiać drugi runtime należący do rdzenia.

Zmiany konfiguracji przeglądarki nadal wymagają ponownego uruchomienia Gateway, aby dołączony plugin
mógł ponownie zarejestrować swoją usługę przeglądarki z nowymi ustawieniami.

## Brak polecenia lub narzędzia browser

Jeśli `openclaw browser` nagle staje się nieznanym poleceniem po aktualizacji albo
agent zgłasza brak narzędzia browser, najczęstszą przyczyną jest restrykcyjna lista
`plugins.allow`, która nie zawiera `browser`.

Przykład błędnej konfiguracji:

```json5
{
  plugins: {
    allow: ["telegram"],
  },
}
```

Napraw to, dodając `browser` do allowlisty pluginów:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Ważne uwagi:

- `browser.enabled=true` samo w sobie nie wystarczy, gdy ustawione jest `plugins.allow`.
- `plugins.entries.browser.enabled=true` samo w sobie również nie wystarczy, gdy ustawione jest `plugins.allow`.
- `tools.alsoAllow: ["browser"]` **nie** ładuje dołączonego pluginu browser. To jedynie dostosowuje politykę narzędzi po tym, jak plugin został już załadowany.
- Jeśli nie potrzebujesz restrykcyjnej allowlisty pluginów, usunięcie `plugins.allow` również przywraca domyślne zachowanie dołączonej przeglądarki.

Typowe objawy:

- `openclaw browser` jest nieznanym poleceniem.
- Brakuje `browser.request`.
- Agent zgłasza narzędzie browser jako niedostępne lub brakujące.

## Profile: `openclaw` vs `user`

- `openclaw`: zarządzana, odizolowana przeglądarka (nie wymaga rozszerzenia).
- `user`: wbudowany profil dołączania Chrome MCP do Twojej **prawdziwej zalogowanej sesji Chrome**.

Dla wywołań narzędzia przeglądarki przez agenta:

- Domyślnie: używaj odizolowanej przeglądarki `openclaw`.
- Preferuj `profile="user"`, gdy mają znaczenie istniejące zalogowane sesje, a użytkownik
  jest przy komputerze, aby kliknąć/zatwierdzić ewentualny prompt dołączenia.
- `profile` jest jawnym nadpisaniem, gdy chcesz konkretny tryb przeglądarki.

Ustaw `browser.defaultProfile: "openclaw"`, jeśli chcesz, aby domyślnie używany był tryb zarządzany.

## Konfiguracja

Ustawienia przeglądarki znajdują się w `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // default: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy single-profile override
    remoteCdpTimeoutMs: 1500, // remote CDP HTTP timeout (ms)
    remoteCdpHandshakeTimeoutMs: 3000, // remote CDP WebSocket handshake timeout (ms)
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

Uwagi:

- Usługa sterowania przeglądarką wiąże się z loopback na porcie wyprowadzonym z `gateway.port`
  (domyślnie: `18791`, czyli gateway + 2).
- Jeśli nadpiszesz port Gateway (`gateway.port` lub `OPENCLAW_GATEWAY_PORT`),
  pochodne porty przeglądarki przesuną się tak, aby pozostać w tej samej „rodzinie”.
- `cdpUrl` domyślnie wskazuje zarządzany lokalny port CDP, gdy nie jest ustawione.
- `remoteCdpTimeoutMs` dotyczy kontroli osiągalności zdalnego (nie-loopbackowego) CDP.
- `remoteCdpHandshakeTimeoutMs` dotyczy kontroli osiągalności handshaku WebSocket dla zdalnego CDP.
- Nawigacja przeglądarki/otwieranie kart jest chronione przed SSRF przed nawigacją i w miarę możliwości sprawdzane ponownie dla końcowego adresu `http(s)` po nawigacji.
- W ścisłym trybie SSRF zdalne wykrywanie/sondowanie endpointów CDP (`cdpUrl`, w tym wyszukiwania `/json/version`) również są sprawdzane.
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` domyślnie ma wartość `true` (model zaufanej sieci). Ustaw `false`, aby wymusić ścisłe przeglądanie tylko publicznych zasobów.
- `browser.ssrfPolicy.allowPrivateNetwork` pozostaje obsługiwane jako starszy alias dla zgodności.
- `attachOnly: true` oznacza „nigdy nie uruchamiaj lokalnej przeglądarki; dołączaj tylko wtedy, gdy już działa”.
- `color` oraz `color` dla poszczególnych profili nadają interfejsowi przeglądarki kolor, dzięki czemu widać, który profil jest aktywny.
- Domyślny profil to `openclaw` (samodzielna przeglądarka zarządzana przez OpenClaw). Użyj `defaultProfile: "user"`, aby przejść na zalogowaną przeglądarkę użytkownika.
- Kolejność automatycznego wykrywania: domyślna przeglądarka systemowa, jeśli jest oparta na Chromium; w przeciwnym razie Chrome → Brave → Edge → Chromium → Chrome Canary.
- Lokalne profile `openclaw` automatycznie przypisują `cdpPort`/`cdpUrl` — ustawiaj je tylko dla zdalnego CDP.
- `driver: "existing-session"` używa Chrome DevTools MCP zamiast surowego CDP. Nie
  ustawiaj `cdpUrl` dla tego sterownika.
- Ustaw `browser.profiles.<name>.userDataDir`, gdy profil existing-session
  ma dołączać do niestandardowego profilu użytkownika Chromium, takiego jak Brave lub Edge.

## Używanie Brave (lub innej przeglądarki opartej na Chromium)

Jeśli Twoja **domyślna przeglądarka systemowa** jest oparta na Chromium (Chrome/Brave/Edge itd.),
OpenClaw używa jej automatycznie. Ustaw `browser.executablePath`, aby nadpisać
automatyczne wykrywanie:

Przykład CLI:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## Sterowanie lokalne vs zdalne

- **Sterowanie lokalne (domyślne):** Gateway uruchamia usługę sterowania loopback i może uruchomić lokalną przeglądarkę.
- **Sterowanie zdalne (host węzła):** uruchom host węzła na komputerze, na którym znajduje się przeglądarka; Gateway będzie proxyfikować do niego akcje przeglądarki.
- **Zdalny CDP:** ustaw `browser.profiles.<name>.cdpUrl` (lub `browser.cdpUrl`), aby
  dołączyć do zdalnej przeglądarki opartej na Chromium. W takim przypadku OpenClaw nie uruchomi lokalnej przeglądarki.

Zachowanie przy zatrzymywaniu różni się w zależności od trybu profilu:

- lokalne profile zarządzane: `openclaw browser stop` zatrzymuje proces przeglądarki, który
  został uruchomiony przez OpenClaw
- profile tylko-dołączane i zdalne profile CDP: `openclaw browser stop` zamyka aktywną
  sesję sterowania i zwalnia nadpisania emulacji Playwright/CDP (viewport,
  schemat kolorów, locale, strefę czasową, tryb offline i podobny stan), nawet
  jeśli żaden proces przeglądarki nie został uruchomiony przez OpenClaw

Zdalne adresy URL CDP mogą zawierać uwierzytelnianie:

- Tokeny w query (np. `https://provider.example?token=<token>`)
- Uwierzytelnianie HTTP Basic (np. `https://user:pass@provider.example`)

OpenClaw zachowuje uwierzytelnianie przy wywoływaniu endpointów `/json/*` oraz podczas łączenia
z WebSocketem CDP. Preferuj zmienne środowiskowe lub menedżery sekretów dla
tokenów zamiast commitowania ich do plików konfiguracji.

## Proxy przeglądarki węzła (domyślnie zero konfiguracji)

Jeśli uruchamiasz **host węzła** na komputerze, który ma Twoją przeglądarkę, OpenClaw może
automatycznie kierować wywołania narzędzia przeglądarki do tego węzła bez żadnej dodatkowej konfiguracji przeglądarki.
To domyślna ścieżka dla zdalnych gatewayów.

Uwagi:

- Host węzła udostępnia swój lokalny serwer sterowania przeglądarką przez **polecenie proxy**.
- Profile pochodzą z własnej konfiguracji `browser.profiles` węzła (takiej samej jak lokalnie).
- `nodeHost.browserProxy.allowProfiles` jest opcjonalne. Pozostaw puste dla starszego/dom yślnego zachowania: wszystkie skonfigurowane profile pozostają dostępne przez proxy, łącznie z trasami tworzenia/usuwania profili.
- Jeśli ustawisz `nodeHost.browserProxy.allowProfiles`, OpenClaw potraktuje to jako granicę minimalnych uprawnień: tylko profile z allowlisty mogą być celem, a trasy tworzenia/usuwania trwałych profili są blokowane na powierzchni proxy.
- Wyłącz to, jeśli nie chcesz z tego korzystać:
  - Na węźle: `nodeHost.browserProxy.enabled=false`
  - Na gateway: `gateway.nodes.browser.mode="off"`

## Browserless (hostowany zdalny CDP)

[Browserless](https://browserless.io) to hostowana usługa Chromium, która udostępnia
adresy URL połączeń CDP przez HTTPS i WebSocket. OpenClaw może używać obu form, ale
dla zdalnego profilu przeglądarki najprostszą opcją jest bezpośredni adres WebSocket URL
z dokumentacji połączeń Browserless.

Przykład:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

Uwagi:

- Zastąp `<BROWSERLESS_API_KEY>` swoim rzeczywistym tokenem Browserless.
- Wybierz endpoint regionu zgodny z Twoim kontem Browserless (zobacz ich dokumentację).
- Jeśli Browserless podaje bazowy adres HTTPS URL, możesz albo przekonwertować go na
  `wss://` dla bezpośredniego połączenia CDP, albo pozostawić adres HTTPS URL i pozwolić OpenClaw
  wykryć `/json/version`.

## Bezpośredni dostawcy WebSocket CDP

Niektóre hostowane usługi przeglądarki udostępniają **bezpośredni endpoint WebSocket**
zamiast standardowego wykrywania CDP opartego na HTTP (`/json/version`). OpenClaw obsługuje oba tryby:

- **Endpointy HTTP(S)** — OpenClaw wywołuje `/json/version`, aby wykryć
  adres URL debuggera WebSocket, a następnie się łączy.
- **Endpointy WebSocket** (`ws://` / `wss://`) — OpenClaw łączy się bezpośrednio,
  pomijając `/json/version`. Używaj tego w przypadku usług takich jak
  [Browserless](https://browserless.io),
  [Browserbase](https://www.browserbase.com) lub dowolnego dostawcy, który przekazuje Ci
  adres URL WebSocket.

### Browserbase

[Browserbase](https://www.browserbase.com) to platforma chmurowa do uruchamiania
przeglądarek headless z wbudowanym rozwiązywaniem CAPTCHA, trybem stealth i
rezydencyjnymi proxy.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

Uwagi:

- [Zarejestruj się](https://www.browserbase.com/sign-up) i skopiuj swój **API Key**
  z [panelu Overview](https://www.browserbase.com/overview).
- Zastąp `<BROWSERBASE_API_KEY>` swoim rzeczywistym kluczem API Browserbase.
- Browserbase automatycznie tworzy sesję przeglądarki przy połączeniu WebSocket, więc
  nie jest potrzebny ręczny krok tworzenia sesji.
- Warstwa darmowa pozwala na jedną współbieżną sesję i jedną godzinę przeglądarki miesięcznie.
  Zobacz [cennik](https://www.browserbase.com/pricing), aby poznać limity planów płatnych.
- Zobacz [dokumentację Browserbase](https://docs.browserbase.com), aby uzyskać pełne
  odniesienie do API, przewodniki po SDK i przykłady integracji.

## Bezpieczeństwo

Kluczowe założenia:

- Sterowanie przeglądarką działa tylko przez loopback; dostęp przechodzi przez uwierzytelnianie Gateway lub parowanie węzłów.
- Samodzielne loopbackowe API HTTP przeglądarki używa **wyłącznie uwierzytelniania wspólnym sekretem**:
  auth bearer tokenem gateway, `x-openclaw-password` lub HTTP Basic auth z
  skonfigurowanym hasłem gateway.
- Nagłówki tożsamości Tailscale Serve i `gateway.auth.mode: "trusted-proxy"`
  **nie** uwierzytelniają tego samodzielnego loopbackowego API przeglądarki.
- Jeśli sterowanie przeglądarką jest włączone i nie skonfigurowano uwierzytelniania wspólnym sekretem, OpenClaw
  automatycznie generuje `gateway.auth.token` przy uruchomieniu i zapisuje go w konfiguracji.
- OpenClaw **nie** generuje automatycznie tego tokenu, gdy `gateway.auth.mode` jest już ustawione na
  `password`, `none` lub `trusted-proxy`.
- Utrzymuj Gateway i wszelkie hosty węzłów w sieci prywatnej (Tailscale); unikaj publicznej ekspozycji.
- Traktuj zdalne adresy URL/tokeny CDP jako sekrety; preferuj zmienne env lub menedżer sekretów.

Wskazówki dotyczące zdalnego CDP:

- W miarę możliwości preferuj szyfrowane endpointy (HTTPS lub WSS) i tokeny krótkotrwałe.
- Unikaj osadzania długotrwałych tokenów bezpośrednio w plikach konfiguracyjnych.

## Profile (wiele przeglądarek)

OpenClaw obsługuje wiele nazwanych profili (konfiguracji routingu). Profile mogą być:

- **zarządzane przez openclaw**: dedykowana instancja przeglądarki opartej na Chromium z własnym katalogiem danych użytkownika i portem CDP
- **zdalne**: jawny adres URL CDP (przeglądarka oparta na Chromium uruchomiona gdzie indziej)
- **istniejąca sesja**: Twój istniejący profil Chrome przez auto-połączenie Chrome DevTools MCP

Domyślne ustawienia:

- Profil `openclaw` jest tworzony automatycznie, jeśli go brakuje.
- Profil `user` jest wbudowany dla dołączania do istniejącej sesji Chrome MCP.
- Profile istniejących sesji poza `user` są opcjonalne; twórz je za pomocą `--driver existing-session`.
- Lokalne porty CDP są domyślnie przydzielane z zakresu **18800–18899**.
- Usunięcie profilu przenosi jego lokalny katalog danych do Kosza.

Wszystkie endpointy sterowania akceptują `?profile=<name>`; CLI używa `--browser-profile`.

## Istniejąca sesja przez Chrome DevTools MCP

OpenClaw może również dołączyć do działającego profilu przeglądarki opartej na Chromium przez
oficjalny serwer Chrome DevTools MCP. Pozwala to ponownie wykorzystać karty i stan logowania
już otwarte w tym profilu przeglądarki.

Oficjalne materiały wprowadzające i referencje konfiguracyjne:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

Wbudowany profil:

- `user`

Opcjonalnie: utwórz własny niestandardowy profil existing-session, jeśli chcesz inną
nazwę, kolor lub katalog danych przeglądarki.

Zachowanie domyślne:

- Wbudowany profil `user` używa auto-połączenia Chrome MCP, które jest kierowane do
  domyślnego lokalnego profilu Google Chrome.

Użyj `userDataDir` dla Brave, Edge, Chromium lub niestandardowego profilu Chrome:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

Następnie w odpowiadającej przeglądarce:

1. Otwórz stronę inspekcji tej przeglądarki dla zdalnego debugowania.
2. Włącz zdalne debugowanie.
3. Pozostaw przeglądarkę uruchomioną i zatwierdź prompt połączenia, gdy OpenClaw się dołączy.

Typowe strony inspekcji:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

Test smoke dla live attach:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

Jak wygląda powodzenie:

- `status` pokazuje `driver: existing-session`
- `status` pokazuje `transport: chrome-mcp`
- `status` pokazuje `running: true`
- `tabs` wyświetla listę już otwartych kart przeglądarki
- `snapshot` zwraca refy z wybranej aktywnej karty

Co sprawdzić, jeśli dołączenie nie działa:

- docelowa przeglądarka oparta na Chromium ma wersję `144+`
- zdalne debugowanie jest włączone na stronie inspekcji tej przeglądarki
- przeglądarka wyświetliła prompt zgody na dołączenie i został on zaakceptowany
- `openclaw doctor` migruje starą konfigurację przeglądarki opartą na rozszerzeniu i sprawdza, czy
  Chrome jest zainstalowany lokalnie dla domyślnych profili auto-połączenia, ale nie może
  włączyć po Twojej stronie zdalnego debugowania w przeglądarce

Użycie przez agenta:

- Użyj `profile="user"`, gdy potrzebujesz stanu zalogowanej przeglądarki użytkownika.
- Jeśli używasz niestandardowego profilu existing-session, przekaż jawną nazwę tego profilu.
- Wybieraj ten tryb tylko wtedy, gdy użytkownik jest przy komputerze, aby zatwierdzić
  prompt dołączenia.
- Gateway lub host węzła może uruchomić `npx chrome-devtools-mcp@latest --autoConnect`

Uwagi:

- Ta ścieżka jest bardziej ryzykowna niż odizolowany profil `openclaw`, ponieważ może
  działać wewnątrz Twojej zalogowanej sesji przeglądarki.
- OpenClaw nie uruchamia przeglądarki dla tego sterownika; dołącza tylko do
  istniejącej sesji.
- OpenClaw używa tutaj oficjalnego przepływu `--autoConnect` Chrome DevTools MCP. Jeśli
  ustawiono `userDataDir`, OpenClaw przekazuje je dalej, aby kierować do tego jawnego
  katalogu danych użytkownika Chromium.
- Zrzuty ekranu existing-session obsługują przechwycenia strony i przechwycenia elementów `--ref`
  ze snapshotów, ale nie selektory CSS `--element`.
- Zrzuty ekranu strony existing-session działają bez Playwright przez Chrome MCP.
  Zrzuty ekranów elementów oparte na refach (`--ref`) także tam działają, ale `--full-page`
  nie może być łączone z `--ref` ani `--element`.
- Akcje existing-session są nadal bardziej ograniczone niż ścieżka
  zarządzanej przeglądarki:
  - `click`, `type`, `hover`, `scrollIntoView`, `drag` i `select` wymagają
    refów ze snapshotów zamiast selektorów CSS
  - `click` obsługuje tylko lewy przycisk myszy (bez nadpisywania przycisku ani modyfikatorów)
  - `type` nie obsługuje `slowly=true`; użyj `fill` lub `press`
  - `press` nie obsługuje `delayMs`
  - `hover`, `scrollIntoView`, `drag`, `select`, `fill` i `evaluate` nie
    obsługują nadpisywania timeoutów dla pojedynczego wywołania
  - `select` obecnie obsługuje tylko jedną wartość
- Existing-session `wait --url` obsługuje wzorce dokładne, podciągi i globy
  tak jak inne sterowniki przeglądarki. `wait --load networkidle` nie jest jeszcze obsługiwane.
- Hooki uploadu existing-session wymagają `ref` lub `inputRef`, obsługują po jednym pliku naraz
  i nie obsługują kierowania przez CSS `element`.
- Hooki dialogów existing-session nie obsługują nadpisywania timeoutów.
- Niektóre funkcje nadal wymagają ścieżki zarządzanej przeglądarki, w tym
  akcje wsadowe, eksport PDF, przechwytywanie pobrań i `responsebody`.
- Existing-session jest lokalne względem hosta. Jeśli Chrome działa na innej maszynie lub w
  innej przestrzeni nazw sieci, użyj zamiast tego zdalnego CDP lub hosta węzła.

## Gwarancje izolacji

- **Dedykowany katalog danych użytkownika**: nigdy nie dotyka Twojego osobistego profilu przeglądarki.
- **Dedykowane porty**: unika `9222`, aby zapobiegać kolizjom z przepływami pracy deweloperskiej.
- **Deterministyczne sterowanie kartami**: kierowanie na karty przez `targetId`, a nie „ostatnią kartę”.

## Wybór przeglądarki

Przy lokalnym uruchamianiu OpenClaw wybiera pierwszą dostępną:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

Możesz to nadpisać przez `browser.executablePath`.

Platformy:

- macOS: sprawdza `/Applications` i `~/Applications`.
- Linux: szuka `google-chrome`, `brave`, `microsoft-edge`, `chromium` itd.
- Windows: sprawdza typowe lokalizacje instalacji.

## API sterowania (opcjonalne)

Tylko dla integracji lokalnych, Gateway udostępnia niewielkie loopbackowe API HTTP:

- Status/start/stop: `GET /`, `POST /start`, `POST /stop`
- Karty: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`
- Snapshot/zrzut ekranu: `GET /snapshot`, `POST /screenshot`
- Akcje: `POST /navigate`, `POST /act`
- Hooki: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Pobieranie: `POST /download`, `POST /wait/download`
- Debugowanie: `GET /console`, `POST /pdf`
- Debugowanie: `GET /errors`, `GET /requests`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Sieć: `POST /response/body`
- Stan: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- Stan: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Ustawienia: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

Wszystkie endpointy akceptują `?profile=<name>`.

Jeśli skonfigurowano uwierzytelnianie gateway wspólnym sekretem, trasy HTTP przeglądarki również wymagają uwierzytelnienia:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` lub HTTP Basic auth z tym hasłem

Uwagi:

- To samodzielne loopbackowe API przeglądarki **nie** używa nagłówków trusted-proxy ani
  tożsamości Tailscale Serve.
- Jeśli `gateway.auth.mode` ma wartość `none` lub `trusted-proxy`, te loopbackowe trasy przeglądarki
  nie dziedziczą tych trybów opartych na tożsamości; utrzymuj je wyłącznie na loopback.

### Kontrakt błędów `/act`

`POST /act` używa ustrukturyzowanej odpowiedzi błędu dla walidacji na poziomie trasy i
błędów polityki:

```json
{ "error": "<message>", "code": "ACT_*" }
```

Obecne wartości `code`:

- `ACT_KIND_REQUIRED` (HTTP 400): brakuje `kind` lub nie został rozpoznany.
- `ACT_INVALID_REQUEST` (HTTP 400): payload akcji nie przeszedł normalizacji lub walidacji.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): użyto `selector` z nieobsługiwanym rodzajem akcji.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (lub `wait --fn`) jest wyłączone w konfiguracji.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): najwyższego poziomu lub wsadowe `targetId` koliduje z celem żądania.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): akcja nie jest obsługiwana dla profili existing-session.

Inne błędy runtime nadal mogą zwracać `{ "error": "<message>" }` bez pola
`code`.

### Wymaganie Playwright

Niektóre funkcje (navigate/act/AI snapshot/role snapshot, zrzuty ekranów elementów,
PDF) wymagają Playwright. Jeśli Playwright nie jest zainstalowany, te endpointy zwracają
czytelny błąd 501.

Co nadal działa bez Playwright:

- Snapshoty ARIA
- Zrzuty ekranu strony dla zarządzanej przeglądarki `openclaw`, gdy dostępny jest WebSocket
  CDP dla danej karty
- Zrzuty ekranu strony dla profili `existing-session` / Chrome MCP
- Zrzuty ekranów existing-session oparte na `--ref` z danych wyjściowych snapshotu

Co nadal wymaga Playwright:

- `navigate`
- `act`
- AI snapshoty / snapshoty ról
- Zrzuty ekranów elementów przez selektor CSS (`--element`)
- eksport pełnego PDF przeglądarki

Zrzuty ekranów elementów również odrzucają `--full-page`; trasa zwraca `fullPage is
not supported for element screenshots`.

Jeśli widzisz `Playwright is not available in this gateway build`, zainstaluj pełny
pakiet Playwright (nie `playwright-core`) i uruchom ponownie gateway albo zainstaluj
OpenClaw ponownie z obsługą przeglądarki.

#### Instalacja Playwright w Dockerze

Jeśli Twój Gateway działa w Dockerze, unikaj `npx playwright` (konflikty nadpisań npm).
Zamiast tego użyj dołączonego CLI:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Aby zachować pobrane przeglądarki, ustaw `PLAYWRIGHT_BROWSERS_PATH` (na przykład
`/home/node/.cache/ms-playwright`) i upewnij się, że `/home/node` jest utrwalane przez
`OPENCLAW_HOME_VOLUME` lub bind mount. Zobacz [Docker](/pl/install/docker).

## Jak to działa (wewnętrznie)

Przepływ na wysokim poziomie:

- Mały **serwer sterowania** przyjmuje żądania HTTP.
- Łączy się z przeglądarkami opartymi na Chromium (Chrome/Brave/Edge/Chromium) przez **CDP**.
- W przypadku zaawansowanych akcji (kliknięcie/wpisywanie/snapshot/PDF) używa **Playwright** na wierzchu
  CDP.
- Gdy brakuje Playwright, dostępne są tylko operacje niewymagające Playwright.

Taki projekt zapewnia agentowi stabilny, deterministyczny interfejs, a jednocześnie pozwala
zamieniać lokalne/zdalne przeglądarki i profile.

## Krótkie odniesienie do CLI

Wszystkie polecenia akceptują `--browser-profile <name>`, aby wskazać konkretny profil.
Wszystkie polecenia akceptują też `--json` dla danych wyjściowych czytelnych maszynowo (stabilne payloady).

Podstawy:

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

Inspekcja:

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`

Uwaga dotycząca cyklu życia:

- Dla profili tylko-dołączanych i zdalnych profili CDP `openclaw browser stop` nadal jest
  właściwym poleceniem czyszczenia po testach. Zamyka aktywną sesję sterowania i
  czyści tymczasowe nadpisania emulacji zamiast zamykać bazową
  przeglądarkę.
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

Akcje:

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

Stan:

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

Uwagi:

- `upload` i `dialog` to wywołania **uzbrajające**; uruchom je przed kliknięciem/naciśnięciem,
  które wywoła wybór pliku/okno dialogowe.
- Ścieżki wyjściowe pobierania i trace są ograniczone do tymczasowych katalogów głównych OpenClaw:
  - traces: `/tmp/openclaw` (fallback: `${os.tmpdir()}/openclaw`)
  - downloads: `/tmp/openclaw/downloads` (fallback: `${os.tmpdir()}/openclaw/downloads`)
- Ścieżki uploadu są ograniczone do tymczasowego katalogu głównego uploadów OpenClaw:
  - uploads: `/tmp/openclaw/uploads` (fallback: `${os.tmpdir()}/openclaw/uploads`)
- `upload` może też ustawiać wejścia plików bezpośrednio przez `--input-ref` lub `--element`.
- `snapshot`:
  - `--format ai` (domyślnie, gdy Playwright jest zainstalowany): zwraca snapshot AI z numerycznymi refami (`aria-ref="<n>"`).
  - `--format aria`: zwraca drzewo dostępności (bez refów; tylko do inspekcji).
  - `--efficient` (lub `--mode efficient`): kompaktowy preset snapshotu ról (interactive + compact + depth + niższe maxChars).
  - Domyślna konfiguracja (tylko tool/CLI): ustaw `browser.snapshotDefaults.mode: "efficient"`, aby używać wydajnych snapshotów, gdy wywołujący nie przekazuje trybu (zobacz [Konfiguracja Gateway](/pl/gateway/configuration-reference#browser)).
  - Opcje snapshotu ról (`--interactive`, `--compact`, `--depth`, `--selector`) wymuszają snapshot oparty na rolach z refami takimi jak `ref=e12`.
  - `--frame "<iframe selector>"` ogranicza snapshoty ról do iframe'a (paruje się z refami ról takimi jak `e12`).
  - `--interactive` wyświetla płaską, łatwą do wyboru listę elementów interaktywnych (najlepszą do sterowania akcjami).
  - `--labels` dodaje zrzut ekranu tylko viewportu z nałożonymi etykietami refów (wypisuje `MEDIA:<path>`).
- `click`/`type`/itd. wymagają `ref` ze `snapshot` (numerycznego `12` albo refu roli `e12`).
  Selektory CSS celowo nie są obsługiwane dla akcji.

## Snapshoty i refy

OpenClaw obsługuje dwa style „snapshotów”:

- **Snapshot AI (numeryczne refy)**: `openclaw browser snapshot` (domyślnie; `--format ai`)
  - Wynik: tekstowy snapshot zawierający numeryczne refy.
  - Akcje: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Wewnętrznie ref jest rozwiązywany przez `aria-ref` Playwright.

- **Snapshot ról (refy ról takie jak `e12`)**: `openclaw browser snapshot --interactive` (lub `--compact`, `--depth`, `--selector`, `--frame`)
  - Wynik: lista/drzewo oparte na rolach z `[ref=e12]` (i opcjonalnym `[nth=1]`).
  - Akcje: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Wewnętrznie ref jest rozwiązywany przez `getByRole(...)` (plus `nth()` dla duplikatów).
  - Dodaj `--labels`, aby dołączyć zrzut ekranu viewportu z nałożonymi etykietami `e12`.

Zachowanie refów:

- Refy **nie są stabilne między nawigacjami**; jeśli coś się nie powiedzie, uruchom ponownie `snapshot` i użyj świeżego refu.
- Jeśli snapshot ról został wykonany z `--frame`, refy ról są ograniczone do tego iframe'a aż do następnego snapshotu ról.

## Rozszerzone możliwości `wait`

Możesz czekać na więcej niż tylko czas/tekst:

- Czekanie na URL (globy obsługiwane przez Playwright):
  - `openclaw browser wait --url "**/dash"`
- Czekanie na stan ładowania:
  - `openclaw browser wait --load networkidle`
- Czekanie na predykat JS:
  - `openclaw browser wait --fn "window.ready===true"`
- Czekanie, aż selektor stanie się widoczny:
  - `openclaw browser wait "#main"`

Można je łączyć:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Przepływy debugowania

Gdy akcja się nie powiedzie (np. „not visible”, „strict mode violation”, „covered”):

1. `openclaw browser snapshot --interactive`
2. Użyj `click <ref>` / `type <ref>` (preferuj refy ról w trybie interactive)
3. Jeśli nadal się nie powiedzie: `openclaw browser highlight <ref>`, aby zobaczyć, na co kieruje Playwright
4. Jeśli strona zachowuje się dziwnie:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. Do głębokiego debugowania: nagraj trace:
   - `openclaw browser trace start`
   - odtwórz problem
   - `openclaw browser trace stop` (wypisuje `TRACE:<path>`)

## Dane wyjściowe JSON

`--json` służy do skryptów i narzędzi strukturalnych.

Przykłady:

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

Snapshoty ról w JSON zawierają `refs` oraz mały blok `stats` (lines/chars/refs/interactive), aby narzędzia mogły analizować rozmiar i gęstość payloadu.

## Pokrętła stanu i środowiska

Są przydatne w przepływach „spraw, aby witryna zachowywała się jak X”:

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Storage: `storage local|session get|set|clear`
- Offline: `set offline on|off`
- Headers: `set headers --headers-json '{"X-Debug":"1"}'` (starsze `set headers --json '{"X-Debug":"1"}'` nadal jest obsługiwane)
- HTTP Basic auth: `set credentials user pass` (lub `--clear`)
- Geolocation: `set geo <lat> <lon> --origin "https://example.com"` (lub `--clear`)
- Media: `set media dark|light|no-preference|none`
- Timezone / locale: `set timezone ...`, `set locale ...`
- Device / viewport:
  - `set device "iPhone 14"` (presety urządzeń Playwright)
  - `set viewport 1280 720`

## Bezpieczeństwo i prywatność

- Profil przeglądarki openclaw może zawierać zalogowane sesje; traktuj go jako wrażliwy.
- `browser act kind=evaluate` / `openclaw browser evaluate` oraz `wait --fn`
  wykonują dowolny JavaScript w kontekście strony. Prompt injection może tym sterować.
  Wyłącz to przez `browser.evaluateEnabled=false`, jeśli tego nie potrzebujesz.
- W przypadku logowań i uwag dotyczących anti-botów (X/Twitter itd.) zobacz [Logowanie w przeglądarce + publikowanie na X/Twitter](/pl/tools/browser-login).
- Utrzymuj Gateway/host węzła w prywatnym środowisku (tylko loopback lub tailnet).
- Zdalne endpointy CDP są potężne; tuneluj je i zabezpieczaj.

Przykład trybu ścisłego (domyślnie blokowanie prywatnych/wewnętrznych miejsc docelowych):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optional exact allow
    },
  },
}
```

## Rozwiązywanie problemów

W przypadku problemów specyficznych dla Linuxa (zwłaszcza snap Chromium) zobacz
[Rozwiązywanie problemów z przeglądarką](/pl/tools/browser-linux-troubleshooting).

W przypadku konfiguracji rozdzielonego hosta WSL2 Gateway + Windows Chrome zobacz
[WSL2 + Windows + rozwiązywanie problemów ze zdalnym Chrome CDP](/pl/tools/browser-wsl2-windows-remote-cdp-troubleshooting).

## Narzędzia agenta + jak działa sterowanie

Agent otrzymuje **jedno narzędzie** do automatyzacji przeglądarki:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

Jak to się mapuje:

- `browser snapshot` zwraca stabilne drzewo UI (AI lub ARIA).
- `browser act` używa identyfikatorów `ref` ze snapshotu do kliknięcia/wpisywania/przeciągania/zaznaczania.
- `browser screenshot` przechwytuje piksele (cała strona lub element).
- `browser` akceptuje:
  - `profile`, aby wybrać nazwany profil przeglądarki (openclaw, chrome lub zdalny CDP).
  - `target` (`sandbox` | `host` | `node`), aby wybrać, gdzie znajduje się przeglądarka.
  - W sesjach sandboxowanych `target: "host"` wymaga `agents.defaults.sandbox.browser.allowHostControl=true`.
  - Jeśli `target` zostanie pominięte: sesje sandboxowane domyślnie używają `sandbox`, a sesje niesandboxowane `host`.
  - Jeśli podłączony jest węzeł z obsługą przeglądarki, narzędzie może automatycznie kierować do niego wywołania, chyba że przypniesz `target="host"` lub `target="node"`.

Dzięki temu agent pozostaje deterministyczny i unika kruchych selektorów.

## Powiązane

- [Przegląd narzędzi](/pl/tools) — wszystkie dostępne narzędzia agenta
- [Sandboxing](/pl/gateway/sandboxing) — sterowanie przeglądarką w środowiskach sandboxowanych
- [Bezpieczeństwo](/pl/gateway/security) — ryzyka i utwardzanie sterowania przeglądarką
