---
read_when:
    - Implementowanie lub aktualizowanie klientów WS bramy
    - Debugowanie niezgodności protokołu lub błędów połączenia
    - Ponowne generowanie schematu/modeli protokołu
summary: 'Protokół WebSocket bramy: uzgadnianie połączenia, ramki, wersjonowanie'
title: Protokół bramy
x-i18n:
    generated_at: "2026-04-10T09:44:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83c820c46d4803d571c770468fd6782619eaa1dca253e156e8087dec735c127f
    source_path: gateway/protocol.md
    workflow: 15
---

# Protokół bramy (WebSocket)

Protokół Gateway WS to **jedyna płaszczyzna sterowania + transport węzłów** dla
OpenClaw. Wszyscy klienci (CLI, interfejs webowy, aplikacja macOS, węzły
iOS/Android, węzły bezgłowe) łączą się przez WebSocket i deklarują swoją **rolę**
+ **zakres** podczas uzgadniania połączenia.

## Transport

- WebSocket, ramki tekstowe z ładunkami JSON.
- Pierwsza ramka **musi** być żądaniem `connect`.

## Uzgadnianie połączenia (`connect`)

Brama → klient (wyzwanie przed połączeniem):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Klient → brama:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Brama → klient:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

Gdy zostanie wydany token urządzenia, `hello-ok` zawiera także:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Podczas przekazywania w zaufanym bootstrapie, `hello-ok.auth` może także zawierać
dodatkowe ograniczone wpisy ról w `deviceTokens`:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

W przypadku wbudowanego przepływu bootstrap dla węzła/operatora podstawowy token
węzła pozostaje z `scopes: []`, a każdy przekazany token operatora pozostaje
ograniczony do listy dozwolonych zakresów operatora bootstrapu
(`operator.approvals`, `operator.read`, `operator.talk.secrets`,
`operator.write`). Kontrole zakresu bootstrapu pozostają prefiksowane rolą:
wpisy operatora spełniają tylko żądania operatora, a role inne niż operator
nadal wymagają zakresów pod własnym prefiksem roli.

### Przykład węzła

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## Ramkowanie

- **Żądanie**: `{type:"req", id, method, params}`
- **Odpowiedź**: `{type:"res", id, ok, payload|error}`
- **Zdarzenie**: `{type:"event", event, payload, seq?, stateVersion?}`

Metody wywołujące skutki uboczne wymagają **kluczy idempotencji** (zobacz
schemat).

## Role + zakresy

### Role

- `operator` = klient płaszczyzny sterowania (CLI/UI/automatyzacja).
- `node` = host możliwości (camera/screen/canvas/system.run).

### Zakresy (operator)

Typowe zakresy:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` z `includeSecrets: true` wymaga `operator.talk.secrets`
(lub `operator.admin`).

Metody RPC bramy zarejestrowane przez pluginy mogą wymagać własnego zakresu
operatora, ale zarezerwowane prefiksy administracyjne rdzenia (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) zawsze są mapowane na
`operator.admin`.

Zakres metody jest tylko pierwszym progiem kontroli. Niektóre polecenia slash
osiągane przez `chat.send` nakładają dodatkowo bardziej restrykcyjne kontrole na
poziomie polecenia. Na przykład trwałe zapisy `/config set` i `/config unset`
wymagają `operator.admin`.

`node.pair.approve` ma także dodatkową kontrolę zakresu podczas zatwierdzania,
ponad bazowy zakres metody:

- żądania bez polecenia: `operator.pairing`
- żądania z poleceniami węzła innymi niż exec: `operator.pairing` + `operator.write`
- żądania zawierające `system.run`, `system.run.prepare` lub `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Węzły deklarują żądania możliwości podczas łączenia:

- `caps`: kategorie możliwości wysokiego poziomu.
- `commands`: lista dozwolonych poleceń dla invoke.
- `permissions`: szczegółowe przełączniki (np. `screen.record`, `camera.capture`).

Brama traktuje je jako **deklaracje** i egzekwuje listy dozwolonych po stronie
serwera.

## Obecność

- `system-presence` zwraca wpisy indeksowane tożsamością urządzenia.
- Wpisy obecności zawierają `deviceId`, `roles` i `scopes`, aby interfejsy UI
  mogły pokazywać jeden wiersz na urządzenie, nawet gdy łączy się ono zarówno
  jako **operator**, jak i **node**.

## Typowe rodziny metod RPC

Ta strona nie jest wygenerowanym pełnym zrzutem, ale publiczna powierzchnia WS
jest szersza niż przykłady uzgadniania połączenia/uwierzytelniania powyżej. To
główne rodziny metod, które brama udostępnia obecnie.

`hello-ok.features.methods` to zachowawcza lista wykrywania zbudowana na bazie
`src/gateway/server-methods-list.ts` oraz załadowanych eksportów metod
pluginów/kanałów. Traktuj ją jako wykrywanie funkcji, a nie jako wygenerowany
zrzut każdego wywoływalnego helpera zaimplementowanego w
`src/gateway/server-methods/*.ts`.

### System i tożsamość

- `health` zwraca buforowaną lub świeżo sprawdzoną migawkę stanu zdrowia bramy.
- `status` zwraca podsumowanie bramy w stylu `/status`; pola wrażliwe są
  dołączane tylko dla klientów operatora z zakresem administratora.
- `gateway.identity.get` zwraca tożsamość urządzenia bramy używaną przez
  przepływy relay i parowania.
- `system-presence` zwraca bieżącą migawkę obecności dla połączonych urządzeń
  operatora/węzła.
- `system-event` dopisuje zdarzenie systemowe i może aktualizować/rozgłaszać
  kontekst obecności.
- `last-heartbeat` zwraca ostatnie zapisane trwałe zdarzenie heartbeat.
- `set-heartbeats` przełącza przetwarzanie heartbeatów na bramie.

### Modele i użycie

- `models.list` zwraca katalog modeli dozwolonych w czasie działania.
- `usage.status` zwraca okna użycia dostawcy/podsumowania pozostałego limitu.
- `usage.cost` zwraca zagregowane podsumowania kosztów użycia dla zakresu dat.
- `doctor.memory.status` zwraca gotowość pamięci wektorowej / embeddingów dla
  aktywnego domyślnego workspace agenta.
- `sessions.usage` zwraca podsumowania użycia dla każdej sesji.
- `sessions.usage.timeseries` zwraca szereg czasowy użycia dla jednej sesji.
- `sessions.usage.logs` zwraca wpisy dziennika użycia dla jednej sesji.

### Kanały i helpery logowania

- `channels.status` zwraca podsumowania stanu kanałów/pluginów wbudowanych i
  dołączonych.
- `channels.logout` wylogowuje z określonego kanału/konta, jeśli dany kanał
  obsługuje wylogowanie.
- `web.login.start` uruchamia przepływ logowania QR/web dla bieżącego dostawcy
  kanału web obsługującego QR.
- `web.login.wait` czeka na zakończenie tego przepływu logowania QR/web i po
  sukcesie uruchamia kanał.
- `push.test` wysyła testowe powiadomienie push APNs do zarejestrowanego węzła iOS.
- `voicewake.get` zwraca zapisane wyzwalacze słowa aktywującego.
- `voicewake.set` aktualizuje wyzwalacze słowa aktywującego i rozgłasza zmianę.

### Wiadomości i logi

- `send` to bezpośrednia metoda RPC dostarczania wychodzącego dla wysyłek
  ukierunkowanych na kanał/konto/wątek poza runnerem czatu.
- `logs.tail` zwraca końcówkę skonfigurowanego dziennika plikowego bramy z
  kontrolą kursora/limitu i maksymalnej liczby bajtów.

### Talk i TTS

- `talk.config` zwraca efektywny ładunek konfiguracji Talk; `includeSecrets`
  wymaga `operator.talk.secrets` (lub `operator.admin`).
- `talk.mode` ustawia/rozgłasza bieżący stan trybu Talk dla klientów
  WebChat/Control UI.
- `talk.speak` syntezuje mowę przez aktywnego dostawcę mowy Talk.
- `tts.status` zwraca stan włączenia TTS, aktywnego dostawcę, dostawców
  zapasowych i stan konfiguracji dostawcy.
- `tts.providers` zwraca widoczny inwentarz dostawców TTS.
- `tts.enable` i `tts.disable` przełączają stan preferencji TTS.
- `tts.setProvider` aktualizuje preferowanego dostawcę TTS.
- `tts.convert` uruchamia jednorazową konwersję tekstu na mowę.

### Sekrety, konfiguracja, aktualizacja i kreator

- `secrets.reload` ponownie rozwiązuje aktywne SecretRef i podmienia stan
  sekretów w czasie działania tylko przy pełnym sukcesie.
- `secrets.resolve` rozwiązuje przypisania sekretów docelowych poleceń dla
  określonego zestawu poleceń/celów.
- `config.get` zwraca bieżącą migawkę konfiguracji i hash.
- `config.set` zapisuje zweryfikowany ładunek konfiguracji.
- `config.patch` scala częściową aktualizację konfiguracji.
- `config.apply` weryfikuje i zastępuje pełny ładunek konfiguracji.
- `config.schema` zwraca ładunek aktywnego schematu konfiguracji używany przez
  Control UI i narzędzia CLI: schemat, `uiHints`, wersję i metadane generowania,
  w tym metadane schematów pluginów + kanałów, gdy środowisko wykonawcze może je
  załadować. Schemat zawiera metadane pól `title` / `description` pochodzące z
  tych samych etykiet i tekstów pomocy używanych przez UI, w tym dla zagnieżdżonych
  obiektów, wildcard, elementów tablic i gałęzi kompozycji `anyOf` / `oneOf` / `allOf`,
  gdy istnieje odpowiadająca dokumentacja pola.
- `config.schema.lookup` zwraca ładunek wyszukiwania ograniczony do ścieżki dla
  jednej ścieżki konfiguracji: znormalizowaną ścieżkę, płytki węzeł schematu,
  dopasowaną wskazówkę + `hintPath` oraz podsumowania bezpośrednich elementów
  podrzędnych do drążenia w UI/CLI.
  - Węzły schematu wyszukiwania zachowują dokumentację widoczną dla użytkownika
    i typowe pola walidacji: `title`, `description`, `type`, `enum`, `const`,
    `format`, `pattern`, ograniczenia liczbowe/tekstowe/tablicowe/obiektowe oraz
    flagi logiczne, takie jak `additionalProperties`, `deprecated`, `readOnly`,
    `writeOnly`.
  - Podsumowania elementów podrzędnych udostępniają `key`, znormalizowaną `path`,
    `type`, `required`, `hasChildren`, a także dopasowane `hint` / `hintPath`.
- `update.run` uruchamia przepływ aktualizacji bramy i planuje restart tylko
  wtedy, gdy sama aktualizacja zakończyła się sukcesem.
- `wizard.start`, `wizard.next`, `wizard.status` i `wizard.cancel` udostępniają
  kreator onboardingu przez WS RPC.

### Istniejące główne rodziny

#### Helpery agentów i workspace

- `agents.list` zwraca skonfigurowane wpisy agentów.
- `agents.create`, `agents.update` i `agents.delete` zarządzają rekordami
  agentów i okablowaniem workspace.
- `agents.files.list`, `agents.files.get` i `agents.files.set` zarządzają
  plikami bootstrap workspace udostępnianymi dla agenta.
- `agent.identity.get` zwraca efektywną tożsamość asystenta dla agenta lub sesji.
- `agent.wait` czeka na zakończenie uruchomienia i zwraca terminalną migawkę,
  jeśli jest dostępna.

#### Sterowanie sesją

- `sessions.list` zwraca bieżący indeks sesji.
- `sessions.subscribe` i `sessions.unsubscribe` przełączają subskrypcje zdarzeń
  zmian sesji dla bieżącego klienta WS.
- `sessions.messages.subscribe` i `sessions.messages.unsubscribe` przełączają
  subskrypcje zdarzeń transkryptu/wiadomości dla jednej sesji.
- `sessions.preview` zwraca ograniczone podglądy transkryptu dla określonych
  kluczy sesji.
- `sessions.resolve` rozwiązuje lub kanonizuje cel sesji.
- `sessions.create` tworzy nowy wpis sesji.
- `sessions.send` wysyła wiadomość do istniejącej sesji.
- `sessions.steer` to wariant przerwania i sterowania dla aktywnej sesji.
- `sessions.abort` przerywa aktywną pracę dla sesji.
- `sessions.patch` aktualizuje metadane/nadpisania sesji.
- `sessions.reset`, `sessions.delete` i `sessions.compact` wykonują działania
  konserwacyjne na sesji.
- `sessions.get` zwraca pełny zapisany wiersz sesji.
- wykonanie czatu nadal używa `chat.history`, `chat.send`, `chat.abort` i
  `chat.inject`.
- `chat.history` jest znormalizowane do wyświetlania dla klientów UI:
  znaczniki dyrektyw inline są usuwane z widocznego tekstu, ładunki XML wywołań
  narzędzi w zwykłym tekście (w tym
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` oraz
  obcięte bloki wywołań narzędzi) i wyciekłe tokeny sterujące modelu w ASCII/pełnej
  szerokości są usuwane, czyste wiersze asystenta zawierające wyłącznie ciche tokeny,
  takie jak dokładne `NO_REPLY` / `no_reply`, są pomijane, a zbyt duże wiersze mogą
  być zastępowane placeholderami.

#### Parowanie urządzeń i tokeny urządzeń

- `device.pair.list` zwraca oczekujące i zatwierdzone sparowane urządzenia.
- `device.pair.approve`, `device.pair.reject` i `device.pair.remove` zarządzają
  rekordami parowania urządzeń.
- `device.token.rotate` obraca token sparowanego urządzenia w granicach jego
  zatwierdzonej roli i zakresów.
- `device.token.revoke` unieważnia token sparowanego urządzenia.

#### Parowanie węzłów, invoke i oczekująca praca

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` i `node.pair.verify` obejmują parowanie węzłów i
  weryfikację bootstrapu.
- `node.list` i `node.describe` zwracają stan znanych/podłączonych węzłów.
- `node.rename` aktualizuje etykietę sparowanego węzła.
- `node.invoke` przekazuje polecenie do podłączonego węzła.
- `node.invoke.result` zwraca wynik dla żądania invoke.
- `node.event` przenosi zdarzenia pochodzące z węzła z powrotem do bramy.
- `node.canvas.capability.refresh` odświeża ograniczone tokeny możliwości canvas.
- `node.pending.pull` i `node.pending.ack` to API kolejki podłączonych węzłów.
- `node.pending.enqueue` i `node.pending.drain` zarządzają trwałą oczekującą
  pracą dla węzłów offline/rozłączonych.

#### Rodziny zatwierdzeń

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` i
  `exec.approval.resolve` obejmują jednorazowe żądania zatwierdzenia exec oraz
  wyszukiwanie/odtwarzanie oczekujących zatwierdzeń.
- `exec.approval.waitDecision` czeka na jedną oczekującą decyzję zatwierdzenia
  exec i zwraca ostateczną decyzję (lub `null` przy przekroczeniu limitu czasu).
- `exec.approvals.get` i `exec.approvals.set` zarządzają migawkami polityki
  zatwierdzeń exec bramy.
- `exec.approvals.node.get` i `exec.approvals.node.set` zarządzają lokalną dla
  węzła polityką zatwierdzeń exec za pośrednictwem poleceń relay węzła.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` i `plugin.approval.resolve` obejmują przepływy
  zatwierdzeń zdefiniowane przez pluginy.

#### Inne główne rodziny

- automatyzacja:
  - `wake` planuje natychmiastowe lub przy następnym heartbeat wstrzyknięcie tekstu budzenia
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- Skills/narzędzia: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`

### Typowe rodziny zdarzeń

- `chat`: aktualizacje czatu UI, takie jak `chat.inject` i inne zdarzenia czatu
  dotyczące wyłącznie transkryptu.
- `session.message` i `session.tool`: aktualizacje transkryptu/strumienia
  zdarzeń dla subskrybowanej sesji.
- `sessions.changed`: indeks sesji lub metadane uległy zmianie.
- `presence`: aktualizacje migawki obecności systemu.
- `tick`: okresowe zdarzenie keepalive / liveness.
- `health`: aktualizacja migawki stanu zdrowia bramy.
- `heartbeat`: aktualizacja strumienia zdarzeń heartbeat.
- `cron`: zdarzenie zmiany uruchomienia/zadania cron.
- `shutdown`: powiadomienie o zamknięciu bramy.
- `node.pair.requested` / `node.pair.resolved`: cykl życia parowania węzła.
- `node.invoke.request`: rozgłoszenie żądania invoke węzła.
- `device.pair.requested` / `device.pair.resolved`: cykl życia sparowanego urządzenia.
- `voicewake.changed`: konfiguracja wyzwalaczy słowa aktywującego uległa zmianie.
- `exec.approval.requested` / `exec.approval.resolved`: cykl życia
  zatwierdzenia exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: cykl życia
  zatwierdzenia pluginu.

### Metody pomocnicze węzła

- Węzły mogą wywoływać `skills.bins`, aby pobrać bieżącą listę plików
  wykonywalnych Skills do automatycznych kontroli listy dozwolonych.

### Metody pomocnicze operatora

- Operatorzy mogą wywoływać `commands.list` (`operator.read`), aby pobrać
  inwentarz poleceń środowiska wykonawczego dla agenta.
  - `agentId` jest opcjonalne; pomiń je, aby odczytać domyślny workspace agenta.
  - `scope` określa, którą powierzchnię celuje podstawowa `name`:
    - `text` zwraca podstawowy tekstowy token polecenia bez wiodącego `/`
    - `native` oraz domyślna ścieżka `both` zwracają nazwy natywne zależne od
      dostawcy, gdy są dostępne
  - `textAliases` zawiera dokładne aliasy slash, takie jak `/model` i `/m`.
  - `nativeName` zawiera natywną nazwę zależną od dostawcy, gdy taka istnieje.
  - `provider` jest opcjonalne i wpływa tylko na nazewnictwo natywne oraz
    dostępność natywnych poleceń pluginów.
  - `includeArgs=false` pomija zserializowane metadane argumentów w odpowiedzi.
- Operatorzy mogą wywoływać `tools.catalog` (`operator.read`), aby pobrać
  katalog narzędzi środowiska wykonawczego dla agenta. Odpowiedź zawiera
  pogrupowane narzędzia i metadane pochodzenia:
  - `source`: `core` lub `plugin`
  - `pluginId`: właściciel pluginu, gdy `source="plugin"`
  - `optional`: czy narzędzie pluginu jest opcjonalne
- Operatorzy mogą wywoływać `tools.effective` (`operator.read`), aby pobrać
  efektywny inwentarz narzędzi środowiska wykonawczego dla sesji.
  - `sessionKey` jest wymagane.
  - Brama wyprowadza zaufany kontekst środowiska wykonawczego po stronie serwera
    z sesji zamiast akceptować kontekst uwierzytelniania lub dostarczenia
    dostarczony przez wywołującego.
  - Odpowiedź jest ograniczona do sesji i odzwierciedla to, z czego aktywna
    konwersacja może korzystać w danej chwili, w tym z narzędzi rdzenia,
    pluginów i kanałów.
- Operatorzy mogą wywoływać `skills.status` (`operator.read`), aby pobrać
  widoczny inwentarz Skills dla agenta.
  - `agentId` jest opcjonalne; pomiń je, aby odczytać domyślny workspace agenta.
  - Odpowiedź zawiera kwalifikowalność, brakujące wymagania, kontrole
    konfiguracji oraz oczyszczone opcje instalacji bez ujawniania surowych
    wartości sekretów.
- Operatorzy mogą wywoływać `skills.search` i `skills.detail`
  (`operator.read`) dla metadanych wykrywania ClawHub.
- Operatorzy mogą wywoływać `skills.install` (`operator.admin`) w dwóch trybach:
  - tryb ClawHub: `{ source: "clawhub", slug, version?, force? }` instaluje
    folder skill do katalogu `skills/` domyślnego workspace agenta.
  - tryb instalatora bramy: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    uruchamia zadeklarowane działanie `metadata.openclaw.install` na hoście bramy.
- Operatorzy mogą wywoływać `skills.update` (`operator.admin`) w dwóch trybach:
  - tryb ClawHub aktualizuje jeden śledzony slug lub wszystkie śledzone
    instalacje ClawHub w domyślnym workspace agenta.
  - tryb konfiguracji modyfikuje wartości `skills.entries.<skillKey>`, takie jak
    `enabled`, `apiKey` i `env`.

## Zatwierdzenia exec

- Gdy żądanie exec wymaga zatwierdzenia, brama rozgłasza `exec.approval.requested`.
- Klienci operatora rozwiązują je przez wywołanie `exec.approval.resolve`
  (wymaga zakresu `operator.approvals`).
- Dla `host=node`, `exec.approval.request` musi zawierać `systemRunPlan`
  (kanoniczne `argv`/`cwd`/`rawCommand`/metadane sesji). Żądania bez
  `systemRunPlan` są odrzucane.
- Po zatwierdzeniu przekazane dalej wywołania `node.invoke system.run` ponownie
  używają tego kanonicznego `systemRunPlan` jako autorytatywnego kontekstu
  polecenia/cwd/sesji.
- Jeśli wywołujący zmieni `command`, `rawCommand`, `cwd`, `agentId` lub
  `sessionKey` między prepare a końcowym zatwierdzonym przekazaniem `system.run`,
  brama odrzuci uruchomienie zamiast ufać zmodyfikowanemu ładunkowi.

## Fallback dostarczania agenta

- Żądania `agent` mogą zawierać `deliver=true`, aby zażądać dostarczenia wychodzącego.
- `bestEffortDeliver=false` zachowuje ścisłe działanie: nierozwiązane lub
  wyłącznie wewnętrzne cele dostarczania zwracają `INVALID_REQUEST`.
- `bestEffortDeliver=true` pozwala na fallback do wykonania tylko w sesji, gdy
  nie można rozwiązać zewnętrznej trasy dostarczania (na przykład sesje
  internal/webchat lub niejednoznaczne konfiguracje wielokanałowe).

## Wersjonowanie

- `PROTOCOL_VERSION` znajduje się w `src/gateway/protocol/schema.ts`.
- Klienci wysyłają `minProtocol` + `maxProtocol`; serwer odrzuca niezgodności.
- Schematy + modele są generowane z definicji TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Uwierzytelnianie

- Uwierzytelnianie bramy za pomocą wspólnego sekretu używa
  `connect.params.auth.token` lub `connect.params.auth.password`, zależnie od
  skonfigurowanego trybu uwierzytelniania.
- Tryby przenoszące tożsamość, takie jak Tailscale Serve
  (`gateway.auth.allowTailscale: true`) lub `gateway.auth.mode: "trusted-proxy"`
  poza loopback, spełniają kontrolę uwierzytelniania połączenia na podstawie
  nagłówków żądania zamiast `connect.params.auth.*`.
- Prywatny ingress z `gateway.auth.mode: "none"` całkowicie pomija
  uwierzytelnianie połączenia wspólnym sekretem; nie wystawiaj tego trybu na
  publiczny/niezaufany ingress.
- Po sparowaniu brama wydaje **token urządzenia** ograniczony do roli +
  zakresów połączenia. Jest on zwracany w `hello-ok.auth.deviceToken` i klient
  powinien zapisać go na potrzeby przyszłych połączeń.
- Klienci powinni zapisywać podstawowy `hello-ok.auth.deviceToken` po każdym
  udanym połączeniu.
- Ponowne połączenie z użyciem tego **zapisanego** tokenu urządzenia powinno
  również ponownie używać zapisanego zatwierdzonego zestawu zakresów dla tego
  tokenu. Zachowuje to wcześniej przyznany dostęp do odczytu/probe/status i
  zapobiega cichemu zawężeniu ponownych połączeń do węższego domyślnego zakresu
  tylko dla administratora.
- Normalna kolejność pierwszeństwa uwierzytelniania połączenia to najpierw
  jawny współdzielony token/hasło, potem jawny `deviceToken`, następnie
  zapisany token per urządzenie, a na końcu token bootstrap.
- Dodatkowe wpisy `hello-ok.auth.deviceTokens` to tokeny przekazania bootstrap.
  Zapisuj je tylko wtedy, gdy połączenie używało uwierzytelniania bootstrap na
  zaufanym transporcie, takim jak `wss://` lub loopback/lokalne parowanie.
- Jeśli klient poda **jawny** `deviceToken` lub jawne `scopes`, żądany przez
  wywołującego zestaw zakresów pozostaje autorytatywny; zakresy z pamięci
  podręcznej są używane ponownie tylko wtedy, gdy klient ponownie używa
  zapisanego tokenu per urządzenie.
- Tokeny urządzeń można obracać/unieważniać przez `device.token.rotate` i
  `device.token.revoke` (wymaga zakresu `operator.pairing`).
- Wydawanie/obracanie tokenów pozostaje ograniczone do zatwierdzonego zestawu
  ról zapisanego w wpisie parowania tego urządzenia; obrót tokenu nie może
  rozszerzyć urządzenia do roli, której zatwierdzenie parowania nigdy nie nadało.
- Dla sesji tokenów sparowanych urządzeń zarządzanie urządzeniami jest
  ograniczone do własnego zakresu, chyba że wywołujący ma także
  `operator.admin`: użytkownicy bez uprawnień administratora mogą
  usuwać/unieważniać/obracać tylko **własny** wpis urządzenia.
- `device.token.rotate` sprawdza także żądany zestaw zakresów operatora względem
  bieżących zakresów sesji wywołującego. Użytkownicy bez uprawnień administratora
  nie mogą obrócić tokenu do szerszego zestawu zakresów operatora, niż już posiadają.
- Błędy uwierzytelniania zawierają `error.details.code` oraz wskazówki odzyskiwania:
  - `error.details.canRetryWithDeviceToken` (boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Zachowanie klienta dla `AUTH_TOKEN_MISMATCH`:
  - Zaufani klienci mogą podjąć jedną ograniczoną próbę ponownego połączenia z
    użyciem tokenu per urządzenie z pamięci podręcznej.
  - Jeśli ta próba się nie powiedzie, klienci powinni zatrzymać automatyczne
    pętle ponownego łączenia i pokazać operatorowi wskazówki dotyczące dalszych działań.

## Tożsamość urządzenia + parowanie

- Węzły powinny dołączać stabilną tożsamość urządzenia (`device.id`) wyprowadzoną
  z odcisku palca pary kluczy.
- Bramy wydają tokeny dla każdego urządzenia + roli.
- Zatwierdzenia parowania są wymagane dla nowych identyfikatorów urządzeń,
  chyba że włączono lokalne automatyczne zatwierdzanie.
- Automatyczne zatwierdzanie parowania jest skoncentrowane na bezpośrednich
  lokalnych połączeniach loopback.
- OpenClaw ma także wąską ścieżkę samopołączenia backend/container-local dla
  zaufanych przepływów pomocniczych ze wspólnym sekretem.
- Połączenia tailnet lub LAN z tego samego hosta są nadal traktowane jako zdalne
  na potrzeby parowania i wymagają zatwierdzenia.
- Wszyscy klienci WS muszą dołączać tożsamość `device` podczas `connect`
  (`operator` + `node`).
  Control UI może ją pominąć tylko w tych trybach:
  - `gateway.controlUi.allowInsecureAuth=true` dla zgodności z niezabezpieczonym HTTP tylko dla localhost.
  - pomyślne uwierzytelnienie operatora Control UI przy `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (tryb awaryjny, poważne obniżenie bezpieczeństwa).
- Wszystkie połączenia muszą podpisywać `connect.challenge` nonce dostarczony przez serwer.

### Diagnostyka migracji uwierzytelniania urządzeń

Dla starszych klientów, które nadal używają zachowania podpisywania sprzed
wprowadzenia challenge, `connect` zwraca teraz kody szczegółów `DEVICE_AUTH_*`
w `error.details.code` wraz ze stabilnym `error.details.reason`.

Typowe błędy migracji:

| Komunikat                   | details.code                     | details.reason           | Znaczenie                                          |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Klient pominął `device.nonce` (lub wysłał pustą wartość). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Klient podpisał dane nieaktualnym/błędnym nonce.   |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Ładunek podpisu nie odpowiada ładunkowi v2.        |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Podpisany znacznik czasu jest poza dozwolonym odchyleniem. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` nie odpowiada odciskowi palca klucza publicznego. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Format/kanonikalizacja klucza publicznego nie powiodły się. |

Cel migracji:

- Zawsze czekaj na `connect.challenge`.
- Podpisuj ładunek v2 zawierający nonce serwera.
- Wysyłaj ten sam nonce w `connect.params.device.nonce`.
- Preferowanym ładunkiem podpisu jest `v3`, który wiąże `platform` i `deviceFamily`
  oprócz pól device/client/role/scopes/token/nonce.
- Starsze podpisy `v2` pozostają akceptowane dla zgodności, ale przypięcie
  metadanych sparowanego urządzenia nadal kontroluje politykę poleceń przy
  ponownym połączeniu.

## TLS + pinning

- TLS jest obsługiwany dla połączeń WS.
- Klienci mogą opcjonalnie przypinać odcisk palca certyfikatu bramy (zobacz
  konfigurację `gateway.tls` oraz `gateway.remote.tlsFingerprint` lub flagę CLI
  `--tls-fingerprint`).

## Zakres

Ten protokół udostępnia **pełne API bramy** (status, kanały, modele, chat,
agent, sesje, węzły, zatwierdzenia itd.). Dokładna powierzchnia jest
zdefiniowana przez schematy TypeBox w `src/gateway/protocol/schema.ts`.
