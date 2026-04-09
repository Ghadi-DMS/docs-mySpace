---
read_when:
    - Konfigurowanie Matrix w OpenClaw
    - Konfigurowanie E2EE i weryfikacji Matrix
summary: Stan obsługi Matrix, konfiguracja i przykłady ustawień
title: Matrix
x-i18n:
    generated_at: "2026-04-09T01:32:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28fc13c7620c1152200315ae69c94205da6de3180c53c814dd8ce03b5cb1758f
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix jest dołączoną wtyczką kanału dla OpenClaw.
Używa oficjalnego `matrix-js-sdk` i obsługuje wiadomości prywatne, pokoje, wątki, multimedia, reakcje, ankiety, lokalizację oraz E2EE.

## Dołączona wtyczka

Matrix jest dostarczany jako dołączona wtyczka w bieżących wydaniach OpenClaw, więc zwykłe
spakowane kompilacje nie wymagają osobnej instalacji.

Jeśli używasz starszej kompilacji lub niestandardowej instalacji, która nie zawiera Matrix, zainstaluj
go ręcznie:

Zainstaluj z npm:

```bash
openclaw plugins install @openclaw/matrix
```

Zainstaluj z lokalnego checkoutu:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Zobacz [Wtyczki](/pl/tools/plugin), aby poznać zachowanie wtyczek i zasady instalacji.

## Konfiguracja

1. Upewnij się, że wtyczka Matrix jest dostępna.
   - Bieżące spakowane wydania OpenClaw już ją zawierają.
   - Starsze/niestandardowe instalacje mogą dodać ją ręcznie za pomocą powyższych poleceń.
2. Utwórz konto Matrix na swoim homeserverze.
3. Skonfiguruj `channels.matrix` z użyciem:
   - `homeserver` + `accessToken`, lub
   - `homeserver` + `userId` + `password`.
4. Uruchom ponownie gateway.
5. Rozpocznij wiadomość prywatną z botem lub zaproś go do pokoju.
   - Nowe zaproszenia Matrix działają tylko wtedy, gdy pozwala na to `channels.matrix.autoJoin`.

Ścieżki interaktywnej konfiguracji:

```bash
openclaw channels add
openclaw configure --section channels
```

Kreator Matrix pyta o:

- URL homeservera
- metodę uwierzytelniania: access token lub hasło
- ID użytkownika (tylko przy uwierzytelnianiu hasłem)
- opcjonalną nazwę urządzenia
- czy włączyć E2EE
- czy skonfigurować dostęp do pokoi i automatyczne dołączanie do zaproszeń

Kluczowe zachowania kreatora:

- Jeśli zmienne środowiskowe uwierzytelniania Matrix już istnieją, a dla tego konta nie zapisano jeszcze uwierzytelniania w konfiguracji, kreator zaproponuje skrót do użycia zmiennych środowiskowych, aby zachować uwierzytelnianie w env vars.
- Nazwy kont są normalizowane do ID konta. Na przykład `Ops Bot` staje się `ops-bot`.
- Wpisy allowlist dla wiadomości prywatnych akceptują bezpośrednio `@user:server`; nazwy wyświetlane działają tylko wtedy, gdy wyszukiwanie w katalogu na żywo znajdzie dokładnie jedno dopasowanie.
- Wpisy allowlist dla pokoi akceptują bezpośrednio ID pokoi i aliasy. Preferuj `!room:server` lub `#alias:server`; nierozwiązane nazwy są ignorowane w czasie działania podczas rozwiązywania allowlist.
- W trybie allowlist dla automatycznego dołączania do zaproszeń używaj wyłącznie stabilnych celów zaproszeń: `!roomId:server`, `#alias:server` lub `*`. Zwykłe nazwy pokoi są odrzucane.
- Aby rozwiązać nazwy pokoi przed zapisaniem, użyj `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` ma domyślnie wartość `off`.

Jeśli pozostawisz tę opcję nieustawioną, bot nie będzie dołączać do zaproszonych pokoi ani nowych zaproszeń w stylu wiadomości prywatnych, więc nie pojawi się w nowych grupach ani zaproszonych wiadomościach prywatnych, chyba że najpierw dołączysz ręcznie.

Ustaw `autoJoin: "allowlist"` razem z `autoJoinAllowlist`, aby ograniczyć, które zaproszenia są akceptowane, albo ustaw `autoJoin: "always"`, jeśli ma dołączać do każdego zaproszenia.

W trybie `allowlist` `autoJoinAllowlist` akceptuje tylko `!roomId:server`, `#alias:server` lub `*`.
</Warning>

Przykład allowlist:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Dołączaj do każdego zaproszenia:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

Minimalna konfiguracja oparta na tokenie:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Konfiguracja oparta na haśle (token jest buforowany po zalogowaniu):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix przechowuje zbuforowane poświadczenia w `~/.openclaw/credentials/matrix/`.
Konto domyślne używa `credentials.json`; konta nazwane używają `credentials-<account>.json`.
Jeśli w tej lokalizacji istnieją zbuforowane poświadczenia, OpenClaw traktuje Matrix jako skonfigurowany na potrzeby konfiguracji, doctor i wykrywania statusu kanału, nawet jeśli bieżące uwierzytelnianie nie jest ustawione bezpośrednio w konfiguracji.

Odpowiedniki zmiennych środowiskowych (używane, gdy klucz konfiguracji nie jest ustawiony):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Dla kont innych niż domyślne użyj zmiennych środowiskowych z zakresem konta:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Przykład dla konta `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Dla znormalizowanego ID konta `ops-bot` użyj:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix escapuje znaki interpunkcyjne w ID kont, aby zmienne środowiskowe z zakresem konta nie kolidowały ze sobą.
Na przykład `-` staje się `_X2D_`, więc `ops-prod` mapuje się na `MATRIX_OPS_X2D_PROD_*`.

Interaktywny kreator oferuje skrót do zmiennych środowiskowych tylko wtedy, gdy te zmienne uwierzytelniania są już obecne, a wybrane konto nie ma jeszcze zapisanego uwierzytelniania Matrix w konfiguracji.

## Przykład konfiguracji

To praktyczna bazowa konfiguracja z parowaniem wiadomości prywatnych, allowlistą pokoi i włączonym E2EE:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin` dotyczy wszystkich zaproszeń Matrix, w tym zaproszeń w stylu wiadomości prywatnych. OpenClaw nie może niezawodnie
określić, czy zaproszony pokój jest wiadomością prywatną czy grupą w momencie zaproszenia, więc wszystkie zaproszenia przechodzą najpierw przez `autoJoin`.
`dm.policy` ma zastosowanie po dołączeniu bota i sklasyfikowaniu pokoju jako wiadomości prywatnej.

## Podglądy strumieniowe

Strumieniowanie odpowiedzi Matrix jest opcjonalne.

Ustaw `channels.matrix.streaming` na `"partial"`, jeśli chcesz, aby OpenClaw wysłał pojedynczą odpowiedź
z podglądem na żywo, edytował ten podgląd na miejscu podczas generowania tekstu przez model, a następnie sfinalizował go po
zakończeniu odpowiedzi:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` to wartość domyślna. OpenClaw czeka na końcową odpowiedź i wysyła ją tylko raz.
- `streaming: "partial"` tworzy jedną edytowalną wiadomość podglądu dla bieżącego bloku odpowiedzi asystenta z użyciem zwykłych wiadomości tekstowych Matrix. Zachowuje to starsze zachowanie Matrix polegające na powiadamianiu najpierw o podglądzie, więc standardowe klienty mogą powiadamiać o pierwszym strumieniowanym tekście podglądu zamiast o ukończonym bloku.
- `streaming: "quiet"` tworzy jeden edytowalny cichy komunikat podglądu dla bieżącego bloku odpowiedzi asystenta. Używaj tego tylko wtedy, gdy skonfigurujesz także reguły push odbiorców dla sfinalizowanych edycji podglądu.
- `blockStreaming: true` włącza osobne komunikaty postępu Matrix. Gdy strumieniowanie podglądu jest włączone, Matrix zachowuje wersję roboczą na żywo dla bieżącego bloku i pozostawia ukończone bloki jako osobne wiadomości.
- Gdy podgląd strumieniowy jest włączony, a `blockStreaming` jest wyłączone, Matrix edytuje wersję roboczą na żywo na miejscu i finalizuje to samo zdarzenie po zakończeniu bloku lub tury.
- Jeśli podgląd nie mieści się już w jednym zdarzeniu Matrix, OpenClaw zatrzymuje strumieniowanie podglądu i wraca do zwykłego końcowego dostarczania.
- Odpowiedzi multimedialne nadal wysyłają załączniki normalnie. Jeśli nieaktualnego podglądu nie da się już bezpiecznie ponownie użyć, OpenClaw usuwa go przed wysłaniem końcowej odpowiedzi multimedialnej.
- Edycje podglądu generują dodatkowe wywołania API Matrix. Pozostaw strumieniowanie wyłączone, jeśli chcesz zachować najbardziej konserwatywne zachowanie względem limitów szybkości.

`blockStreaming` samo w sobie nie włącza podglądów wersji roboczych.
Użyj `streaming: "partial"` lub `streaming: "quiet"` dla edycji podglądu; następnie dodaj `blockStreaming: true` tylko wtedy, gdy chcesz również, aby ukończone bloki asystenta pozostawały widoczne jako osobne komunikaty postępu.

Jeśli potrzebujesz standardowych powiadomień Matrix bez niestandardowych reguł push, użyj `streaming: "partial"` dla zachowania „najpierw podgląd” albo pozostaw `streaming` wyłączone dla dostarczania wyłącznie finalnej odpowiedzi. Przy `streaming: "off"`:

- `blockStreaming: true` wysyła każdy ukończony blok jako zwykłą powiadamiającą wiadomość Matrix.
- `blockStreaming: false` wysyła tylko końcową ukończoną odpowiedź jako zwykłą powiadamiającą wiadomość Matrix.

### Samohostowane reguły push dla cichych sfinalizowanych podglądów

Jeśli uruchamiasz własną infrastrukturę Matrix i chcesz, aby ciche podglądy wysyłały powiadomienie tylko po zakończeniu bloku lub
końcowej odpowiedzi, ustaw `streaming: "quiet"` i dodaj regułę push per użytkownik dla sfinalizowanych edycji podglądu.

Zwykle jest to konfiguracja po stronie użytkownika-odbiorcy, a nie globalna zmiana konfiguracji homeservera:

Szybka mapa przed rozpoczęciem:

- użytkownik odbiorca = osoba, która ma otrzymać powiadomienie
- użytkownik bota = konto Matrix OpenClaw, które wysyła odpowiedź
- do poniższych wywołań API użyj access tokena użytkownika odbiorcy
- w regule push dopasuj `sender` do pełnego MXID użytkownika bota

1. Skonfiguruj OpenClaw do używania cichych podglądów:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Upewnij się, że konto odbiorcy już otrzymuje zwykłe powiadomienia push Matrix. Reguły
   cichych podglądów działają tylko wtedy, gdy ten użytkownik ma już działające pushery/urządzenia.

3. Pobierz access token użytkownika odbiorcy.
   - Użyj tokena użytkownika odbierającego, a nie tokena bota.
   - Najłatwiej zwykle ponownie wykorzystać istniejący token sesji klienta.
   - Jeśli musisz wygenerować nowy token, możesz zalogować się przez standardowe API Matrix Client-Server:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Zweryfikuj, że konto odbiorcy ma już pushery:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Jeśli to zwraca brak aktywnych pusherów/urządzeń, najpierw napraw zwykłe powiadomienia Matrix przed dodaniem
poniższej reguły OpenClaw.

OpenClaw oznacza sfinalizowane edycje tekstowych podglądów za pomocą:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Utwórz regułę push typu override dla każdego konta odbiorcy, które ma otrzymywać te powiadomienia:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Przed uruchomieniem polecenia zastąp te wartości:

- `https://matrix.example.org`: podstawowy URL Twojego homeservera
- `$USER_ACCESS_TOKEN`: access token użytkownika odbierającego
- `openclaw-finalized-preview-botname`: ID reguły unikalne dla tego bota i tego użytkownika odbierającego
- `@bot:example.org`: MXID bota Matrix OpenClaw, a nie MXID użytkownika odbierającego

Ważne dla konfiguracji z wieloma botami:

- Reguły push są kluczowane według `ruleId`. Ponowne uruchomienie `PUT` względem tego samego ID reguły aktualizuje tę jedną regułę.
- Jeśli jeden użytkownik odbierający ma otrzymywać powiadomienia dla wielu kont botów Matrix OpenClaw, utwórz jedną regułę na bota z unikalnym ID reguły dla każdego dopasowania nadawcy.
- Prostym wzorcem jest `openclaw-finalized-preview-<botname>`, na przykład `openclaw-finalized-preview-ops` lub `openclaw-finalized-preview-support`.

Reguła jest oceniana względem nadawcy zdarzenia:

- uwierzytelnij się tokenem użytkownika odbierającego
- dopasuj `sender` do MXID bota OpenClaw

6. Zweryfikuj, że reguła istnieje:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Przetestuj odpowiedź strumieniowaną. W trybie cichym pokój powinien pokazać cichy podgląd wersji roboczej, a końcowa
   edycja na miejscu powinna wysłać powiadomienie po zakończeniu bloku lub tury.

Jeśli później będziesz potrzebować usunąć regułę, usuń to samo ID reguły tokenem użytkownika odbierającego:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Uwagi:

- Utwórz regułę tokenem dostępu użytkownika odbierającego, a nie tokenem bota.
- Nowe reguły `override` zdefiniowane przez użytkownika są wstawiane przed domyślnymi regułami wyciszającymi, więc nie jest potrzebny dodatkowy parametr kolejności.
- Dotyczy to tylko tekstowych edycji podglądu, które OpenClaw może bezpiecznie sfinalizować na miejscu. Fallbacki multimedialne i fallbacki nieaktualnych podglądów nadal używają zwykłego dostarczania Matrix.
- Jeśli `GET /_matrix/client/v3/pushers` pokazuje brak pusherów, użytkownik nie ma jeszcze działającego dostarczania powiadomień push Matrix dla tego konta/urządzenia.

#### Synapse

W przypadku Synapse powyższa konfiguracja zwykle sama w sobie wystarcza:

- Nie jest wymagana żadna specjalna zmiana w `homeserver.yaml` dla sfinalizowanych powiadomień podglądu OpenClaw.
- Jeśli wdrożenie Synapse już wysyła zwykłe powiadomienia push Matrix, głównym krokiem konfiguracji jest token użytkownika + powyższe wywołanie `pushrules`.
- Jeśli uruchamiasz Synapse za reverse proxy lub workerami, upewnij się, że `/_matrix/client/.../pushrules/` poprawnie dociera do Synapse.
- Jeśli używasz workerów Synapse, upewnij się, że pushery działają prawidłowo. Dostarczaniem powiadomień push zajmuje się proces główny albo `synapse.app.pusher` / skonfigurowane workery pusherów.

#### Tuwunel

W przypadku Tuwunel użyj tego samego przebiegu konfiguracji i wywołania API `pushrules`, które pokazano powyżej:

- Nie jest wymagana żadna konfiguracja specyficzna dla Tuwunel dla samego znacznika sfinalizowanego podglądu.
- Jeśli zwykłe powiadomienia Matrix już działają dla tego użytkownika, głównym krokiem konfiguracji jest token użytkownika + powyższe wywołanie `pushrules`.
- Jeśli powiadomienia wydają się znikać, gdy użytkownik jest aktywny na innym urządzeniu, sprawdź, czy włączono `suppress_push_when_active`. Tuwunel dodał tę opcję w Tuwunel 1.4.2 12 września 2025 roku i może ona celowo wyciszać powiadomienia push na innych urządzeniach, gdy jedno urządzenie jest aktywne.

## Pokoje bot-do-bota

Domyślnie wiadomości Matrix z innych skonfigurowanych kont Matrix OpenClaw są ignorowane.

Użyj `allowBots`, jeśli celowo chcesz dopuścić ruch Matrix między agentami:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` akceptuje wiadomości z innych skonfigurowanych kont botów Matrix w dozwolonych pokojach i wiadomościach prywatnych.
- `allowBots: "mentions"` akceptuje te wiadomości tylko wtedy, gdy w pokojach wyraźnie wspominają tego bota. Wiadomości prywatne są nadal dozwolone.
- `groups.<room>.allowBots` nadpisuje ustawienie na poziomie konta dla jednego pokoju.
- OpenClaw nadal ignoruje wiadomości z tego samego ID użytkownika Matrix, aby uniknąć pętli odpowiedzi do samego siebie.
- Matrix nie udostępnia tutaj natywnej flagi bota; OpenClaw traktuje „wiadomość autorstwa bota” jako „wysłaną przez inne skonfigurowane konto Matrix na tym gatewayu OpenClaw”.

Używaj ścisłych allowlist pokoi i wymagań wzmianki podczas włączania ruchu bot-do-bota we współdzielonych pokojach.

## Szyfrowanie i weryfikacja

W szyfrowanych pokojach (E2EE) wychodzące zdarzenia obrazów używają `thumbnail_file`, dzięki czemu podglądy obrazów są szyfrowane razem z pełnym załącznikiem. Nieszyfrowane pokoje nadal używają zwykłego `thumbnail_url`. Nie jest wymagana żadna konfiguracja — wtyczka automatycznie wykrywa stan E2EE.

Włącz szyfrowanie:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Sprawdź status weryfikacji:

```bash
openclaw matrix verify status
```

Szczegółowy status (pełna diagnostyka):

```bash
openclaw matrix verify status --verbose
```

Uwzględnij zapisany klucz odzyskiwania w wyjściu do odczytu maszynowego:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Zainicjalizuj cross-signing i stan weryfikacji:

```bash
openclaw matrix verify bootstrap
```

Szczegółowa diagnostyka bootstrap:

```bash
openclaw matrix verify bootstrap --verbose
```

Wymuś reset tożsamości cross-signing przed bootstrapem:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Zweryfikuj to urządzenie za pomocą klucza odzyskiwania:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Szczegółowe informacje o weryfikacji urządzenia:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Sprawdź stan kopii zapasowej kluczy pokojów:

```bash
openclaw matrix verify backup status
```

Szczegółowa diagnostyka stanu kopii zapasowej:

```bash
openclaw matrix verify backup status --verbose
```

Przywróć klucze pokojów z kopii zapasowej serwera:

```bash
openclaw matrix verify backup restore
```

Szczegółowa diagnostyka przywracania:

```bash
openclaw matrix verify backup restore --verbose
```

Usuń bieżącą kopię zapasową z serwera i utwórz nową bazową kopię zapasową. Jeśli zapisany
klucz kopii zapasowej nie daje się poprawnie wczytać, ten reset może także odtworzyć secret storage, aby
przyszłe zimne starty mogły załadować nowy klucz kopii zapasowej:

```bash
openclaw matrix verify backup reset --yes
```

Wszystkie polecenia `verify` są domyślnie zwięzłe (w tym ciche wewnętrzne logowanie SDK) i pokazują szczegółową diagnostykę tylko z `--verbose`.
Do skryptów używaj `--json`, aby uzyskać pełne dane do odczytu maszynowego.

W konfiguracjach wielokontowych polecenia Matrix CLI używają domyślnego niejawnego konta Matrix, chyba że przekażesz `--account <id>`.
Jeśli skonfigurujesz wiele nazwanych kont, najpierw ustaw `channels.matrix.defaultAccount`, w przeciwnym razie te niejawne operacje CLI zatrzymają się i poproszą o jawny wybór konta.
Używaj `--account`, gdy chcesz, aby operacje weryfikacji lub urządzeń były jawnie kierowane do nazwanego konta:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Gdy szyfrowanie jest wyłączone lub niedostępne dla nazwanego konta, ostrzeżenia Matrix i błędy weryfikacji wskazują klucz konfiguracji tego konta, na przykład `channels.matrix.accounts.assistant.encryption`.

### Co oznacza „zweryfikowane”

OpenClaw traktuje to urządzenie Matrix jako zweryfikowane tylko wtedy, gdy jest ono zweryfikowane przez Twoją własną tożsamość cross-signing.
W praktyce `openclaw matrix verify status --verbose` pokazuje trzy sygnały zaufania:

- `Locally trusted`: to urządzenie jest zaufane tylko przez bieżącego klienta
- `Cross-signing verified`: SDK zgłasza to urządzenie jako zweryfikowane przez cross-signing
- `Signed by owner`: urządzenie jest podpisane przez Twój własny klucz self-signing

`Verified by owner` przyjmuje wartość `yes` tylko wtedy, gdy istnieje weryfikacja cross-signing lub podpis właściciela.
Samo lokalne zaufanie nie wystarcza, aby OpenClaw traktował urządzenie jako w pełni zweryfikowane.

### Co robi bootstrap

`openclaw matrix verify bootstrap` to polecenie naprawy i konfiguracji dla szyfrowanych kont Matrix.
Wykonuje kolejno wszystkie poniższe czynności:

- inicjalizuje secret storage, ponownie używając istniejącego klucza odzyskiwania, gdy to możliwe
- inicjalizuje cross-signing i przesyła brakujące publiczne klucze cross-signing
- próbuje oznaczyć i podpisać krzyżowo bieżące urządzenie
- tworzy nową kopię zapasową kluczy pokojów po stronie serwera, jeśli jeszcze nie istnieje

Jeśli homeserver wymaga interaktywnego uwierzytelniania do przesłania kluczy cross-signing, OpenClaw najpierw próbuje przesłania bez uwierzytelniania, potem z `m.login.dummy`, a następnie z `m.login.password`, gdy skonfigurowano `channels.matrix.password`.

Używaj `--force-reset-cross-signing` tylko wtedy, gdy świadomie chcesz odrzucić bieżącą tożsamość cross-signing i utworzyć nową.

Jeśli świadomie chcesz porzucić bieżącą kopię zapasową kluczy pokojów i rozpocząć nową
bazę kopii zapasowej dla przyszłych wiadomości, użyj `openclaw matrix verify backup reset --yes`.
Rób to tylko wtedy, gdy akceptujesz, że nieodwracalnie utracona stara zaszyfrowana historia pozostanie
niedostępna oraz że OpenClaw może odtworzyć secret storage, jeśli bieżącego sekretu kopii zapasowej
nie da się bezpiecznie załadować.

### Nowa baza kopii zapasowej

Jeśli chcesz zachować działanie przyszłych zaszyfrowanych wiadomości i akceptujesz utratę nieodzyskiwalnej starej historii, uruchom te polecenia po kolei:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Dodaj `--account <id>` do każdego polecenia, jeśli chcesz jawnie wskazać nazwane konto Matrix.

### Zachowanie przy uruchamianiu

Gdy `encryption: true`, Matrix domyślnie ustawia `startupVerification` na `"if-unverified"`.
Przy uruchamianiu, jeśli to urządzenie nadal nie jest zweryfikowane, Matrix wyśle żądanie samoweryfikacji w innym kliencie Matrix,
pominie duplikaty żądań, gdy jedno jest już oczekujące, i zastosuje lokalny cooldown przed ponowną próbą po restartach.
Domyślnie nieudane próby żądania są ponawiane szybciej niż skuteczne utworzenie żądania.
Ustaw `startupVerification: "off"`, aby wyłączyć automatyczne żądania przy uruchamianiu, lub dostosuj `startupVerificationCooldownHours`,
jeśli chcesz krótsze lub dłuższe okno ponawiania.

Przy uruchamianiu automatycznie wykonywany jest także konserwatywny przebieg bootstrap kryptografii.
Ten przebieg najpierw próbuje ponownie użyć bieżącego secret storage i tożsamości cross-signing oraz unika resetowania cross-signing, chyba że uruchomisz jawny przepływ naprawy bootstrap.

Jeśli przy uruchamianiu zostanie wykryty uszkodzony stan bootstrap i skonfigurowano `channels.matrix.password`, OpenClaw może spróbować bardziej rygorystycznej ścieżki naprawy.
Jeśli bieżące urządzenie jest już podpisane przez właściciela, OpenClaw zachowuje tę tożsamość zamiast resetować ją automatycznie.

Zobacz [Migracja Matrix](/pl/install/migrating-matrix), aby poznać pełny przebieg aktualizacji, ograniczenia, polecenia odzyskiwania i typowe komunikaty migracyjne.

### Komunikaty weryfikacji

Matrix publikuje komunikaty dotyczące cyklu życia weryfikacji bezpośrednio w ścisłym pokoju wiadomości prywatnych do weryfikacji jako wiadomości `m.notice`.
Obejmuje to:

- komunikaty żądania weryfikacji
- komunikaty gotowości weryfikacji (z jawną wskazówką „Zweryfikuj przez emoji”)
- komunikaty rozpoczęcia i zakończenia weryfikacji
- szczegóły SAS (emoji i liczby), gdy są dostępne

Przychodzące żądania weryfikacji z innego klienta Matrix są śledzone i automatycznie akceptowane przez OpenClaw.
W przypadku przepływów samoweryfikacji OpenClaw automatycznie uruchamia także przepływ SAS, gdy weryfikacja emoji staje się dostępna, i potwierdza swoją własną stronę.
W przypadku żądań weryfikacji z innego użytkownika/urządzenia Matrix OpenClaw automatycznie akceptuje żądanie, a następnie czeka, aż przepływ SAS będzie kontynuowany normalnie.
Aby zakończyć weryfikację, nadal musisz porównać emoji lub dziesiętny SAS w swoim kliencie Matrix i potwierdzić tam „Pasują”.

OpenClaw nie akceptuje bezrefleksyjnie samoinicjowanych zduplikowanych przepływów. Przy uruchamianiu pomijane jest tworzenie nowego żądania, jeśli żądanie samoweryfikacji jest już oczekujące.

Komunikaty protokołowe/systemowe weryfikacji nie są przekazywane do potoku czatu agenta, więc nie powodują `NO_REPLY`.

### Higiena urządzeń

Na koncie mogą gromadzić się stare urządzenia Matrix zarządzane przez OpenClaw, co utrudnia ocenę zaufania w szyfrowanych pokojach.
Wyświetl ich listę za pomocą:

```bash
openclaw matrix devices list
```

Usuń nieaktualne urządzenia zarządzane przez OpenClaw za pomocą:

```bash
openclaw matrix devices prune-stale
```

### Magazyn kryptograficzny

Matrix E2EE używa oficjalnej ścieżki kryptograficznej Rust z `matrix-js-sdk` w Node, z `fake-indexeddb` jako shimem IndexedDB. Stan kryptograficzny jest utrwalany w pliku migawki (`crypto-idb-snapshot.json`) i przywracany podczas uruchamiania. Plik migawki to wrażliwy stan czasu działania przechowywany z restrykcyjnymi uprawnieniami plików.

Zaszyfrowany stan czasu działania znajduje się w katalogach per konto i per hash tokena użytkownika w
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Ten katalog zawiera magazyn synchronizacji (`bot-storage.json`), magazyn kryptograficzny (`crypto/`),
plik klucza odzyskiwania (`recovery-key.json`), migawkę IndexedDB (`crypto-idb-snapshot.json`),
powiązania wątków (`thread-bindings.json`) oraz stan weryfikacji przy uruchamianiu (`startup-verification.json`).
Gdy token się zmienia, ale tożsamość konta pozostaje taka sama, OpenClaw ponownie używa najlepszego istniejącego
katalogu głównego dla tej krotki konto/homeserver/użytkownik, dzięki czemu wcześniejszy stan synchronizacji, stan kryptograficzny, powiązania wątków
i stan weryfikacji przy uruchamianiu pozostają dostępne.

## Zarządzanie profilem

Zaktualizuj własny profil Matrix dla wybranego konta za pomocą:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Dodaj `--account <id>`, jeśli chcesz jawnie wskazać nazwane konto Matrix.

Matrix akceptuje bezpośrednio URL-e awatarów `mxc://`. Gdy przekażesz URL awatara `http://` lub `https://`, OpenClaw najpierw prześle go do Matrix i zapisze rozwiązany URL `mxc://` z powrotem w `channels.matrix.avatarUrl` (lub w nadpisaniu dla wybranego konta).

## Wątki

Matrix obsługuje natywne wątki Matrix zarówno dla odpowiedzi automatycznych, jak i wysyłek przez narzędzie wiadomości.

- `dm.sessionScope: "per-user"` (domyślnie) utrzymuje routowanie wiadomości prywatnych Matrix w zakresie nadawcy, dzięki czemu wiele pokoi wiadomości prywatnych może współdzielić jedną sesję, gdy odnoszą się do tego samego peer.
- `dm.sessionScope: "per-room"` izoluje każdy pokój wiadomości prywatnych Matrix do własnego klucza sesji, nadal używając zwykłego uwierzytelniania wiadomości prywatnych i kontroli allowlist.
- Jawne powiązania konwersacji Matrix nadal mają pierwszeństwo przed `dm.sessionScope`, więc powiązane pokoje i wątki zachowują wybraną sesję docelową.
- `threadReplies: "off"` utrzymuje odpowiedzi na poziomie głównym i zachowuje przychodzące wiadomości z wątku w sesji nadrzędnej.
- `threadReplies: "inbound"` odpowiada w wątku tylko wtedy, gdy przychodząca wiadomość była już w tym wątku.
- `threadReplies: "always"` utrzymuje odpowiedzi w pokojach w wątku zakorzenionym w wiadomości wyzwalającej i kieruje tę konwersację przez odpowiadającą jej sesję w zakresie wątku od pierwszej wiadomości wyzwalającej.
- `dm.threadReplies` nadpisuje ustawienie najwyższego poziomu tylko dla wiadomości prywatnych. Na przykład możesz utrzymać izolację wątków w pokojach, zachowując płaskie wiadomości prywatne.
- Przychodzące wiadomości z wątków zawierają główną wiadomość wątku jako dodatkowy kontekst agenta.
- Wysyłki przez narzędzie wiadomości automatycznie dziedziczą bieżący wątek Matrix, gdy celem jest ten sam pokój albo ten sam docelowy użytkownik wiadomości prywatnej, chyba że podano jawne `threadId`.
- Ponowne użycie celu tego samego użytkownika wiadomości prywatnej w tej samej sesji uruchamia się tylko wtedy, gdy bieżące metadane sesji potwierdzają tego samego peera wiadomości prywatnej na tym samym koncie Matrix; w przeciwnym razie OpenClaw wraca do zwykłego routowania w zakresie użytkownika.
- Gdy OpenClaw wykryje kolizję pokoju wiadomości prywatnych Matrix z innym pokojem wiadomości prywatnych w tej samej współdzielonej sesji wiadomości prywatnych Matrix, publikuje jednorazowe `m.notice` w tym pokoju z mechanizmem awaryjnym `/focus`, gdy powiązania wątków są włączone, oraz z podpowiedzią `dm.sessionScope`.
- Powiązania wątków w czasie działania są obsługiwane dla Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` i powiązane z wątkiem `/acp spawn` działają w pokojach Matrix i wiadomościach prywatnych.
- Najwyższego poziomu `/focus` w pokoju/DM Matrix tworzy nowy wątek Matrix i wiąże go z sesją docelową, gdy `threadBindings.spawnSubagentSessions=true`.
- Uruchomienie `/focus` lub `/acp spawn --thread here` wewnątrz istniejącego wątku Matrix wiąże zamiast tego bieżący wątek.

## Powiązania konwersacji ACP

Pokoje Matrix, wiadomości prywatne i istniejące wątki Matrix mogą zostać przekształcone w trwałe przestrzenie robocze ACP bez zmiany powierzchni czatu.

Szybki przepływ dla operatora:

- Uruchom `/acp spawn codex --bind here` wewnątrz wiadomości prywatnej Matrix, pokoju lub istniejącego wątku, którego chcesz nadal używać.
- W wiadomości prywatnej Matrix lub pokoju najwyższego poziomu bieżąca wiadomość prywatna/pokój pozostaje powierzchnią czatu, a przyszłe wiadomości są kierowane do utworzonej sesji ACP.
- W istniejącym wątku Matrix `--bind here` wiąże bieżący wątek na miejscu.
- `/new` i `/reset` resetują tę samą powiązaną sesję ACP na miejscu.
- `/acp close` zamyka sesję ACP i usuwa powiązanie.

Uwagi:

- `--bind here` nie tworzy podrzędnego wątku Matrix.
- `threadBindings.spawnAcpSessions` jest wymagane tylko dla `/acp spawn --thread auto|here`, gdy OpenClaw musi utworzyć lub powiązać podrzędny wątek Matrix.

### Konfiguracja powiązań wątków

Matrix dziedziczy globalne ustawienia domyślne z `session.threadBindings`, a także obsługuje nadpisania per kanał:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Flagi uruchamiania powiązane z wątkami Matrix są opt-in:

- Ustaw `threadBindings.spawnSubagentSessions: true`, aby pozwolić najwyższemu poziomowi `/focus` tworzyć i wiązać nowe wątki Matrix.
- Ustaw `threadBindings.spawnAcpSessions: true`, aby pozwolić `/acp spawn --thread auto|here` wiązać sesje ACP z wątkami Matrix.

## Reakcje

Matrix obsługuje wychodzące akcje reakcji, przychodzące powiadomienia o reakcjach oraz przychodzące reakcje potwierdzeń.

- Funkcje reakcji wychodzących są kontrolowane przez `channels["matrix"].actions.reactions`.
- `react` dodaje reakcję do konkretnego zdarzenia Matrix.
- `reactions` wyświetla bieżące podsumowanie reakcji dla konkretnego zdarzenia Matrix.
- `emoji=""` usuwa własne reakcje konta bota na tym zdarzeniu.
- `remove: true` usuwa tylko wskazaną reakcję emoji z konta bota.

Zakres wyszukiwania reakcji potwierdzeń jest rozstrzygany według standardowej kolejności OpenClaw:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- fallback do emoji tożsamości agenta

Zakres reakcji potwierdzeń jest rozstrzygany w tej kolejności:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Tryb powiadomień o reakcjach jest rozstrzygany w tej kolejności:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- domyślnie: `own`

Zachowanie:

- `reactionNotifications: "own"` przekazuje dodane zdarzenia `m.reaction`, gdy odnoszą się do wiadomości Matrix napisanych przez bota.
- `reactionNotifications: "off"` wyłącza systemowe zdarzenia reakcji.
- Usunięcia reakcji nie są syntetyzowane do zdarzeń systemowych, ponieważ Matrix pokazuje je jako redakcje, a nie jako samodzielne usunięcia `m.reaction`.

## Kontekst historii

- `channels.matrix.historyLimit` kontroluje, ile ostatnich wiadomości z pokoju jest dołączanych jako `InboundHistory`, gdy wiadomość z pokoju Matrix wyzwala agenta. Przechodzi na `messages.groupChat.historyLimit`; jeśli oba są nieustawione, efektywna wartość domyślna to `0`. Ustaw `0`, aby wyłączyć.
- Historia pokoju Matrix dotyczy tylko pokoju. Wiadomości prywatne nadal używają zwykłej historii sesji.
- Historia pokoju Matrix jest tylko dla oczekujących: OpenClaw buforuje wiadomości z pokoju, które jeszcze nie wywołały odpowiedzi, a następnie zapisuje migawkę tego okna, gdy nadejdzie wzmianka lub inny trigger.
- Bieżąca wiadomość wyzwalająca nie jest uwzględniona w `InboundHistory`; pozostaje w głównej treści przychodzącej tej tury.
- Ponowienia tego samego zdarzenia Matrix używają ponownie oryginalnej migawki historii zamiast przesuwać się do przodu wraz z nowszymi wiadomościami pokoju.

## Widoczność kontekstu

Matrix obsługuje współdzieloną kontrolę `contextVisibility` dla dodatkowego kontekstu pokoju, takiego jak pobrany tekst odpowiedzi, korzenie wątków i oczekująca historia.

- `contextVisibility: "all"` to wartość domyślna. Dodatkowy kontekst jest zachowywany bez zmian.
- `contextVisibility: "allowlist"` filtruje dodatkowy kontekst do nadawców dozwolonych przez aktywne sprawdzenia allowlist pokoju/użytkownika.
- `contextVisibility: "allowlist_quote"` działa jak `allowlist`, ale nadal zachowuje jedną jawną cytowaną odpowiedź.

To ustawienie wpływa na widoczność dodatkowego kontekstu, a nie na to, czy sama wiadomość przychodząca może wywołać odpowiedź.
Autoryzacja triggera nadal wynika z ustawień `groupPolicy`, `groups`, `groupAllowFrom` i zasad wiadomości prywatnych.

## Zasady wiadomości prywatnych i pokoi

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Zobacz [Grupy](/pl/channels/groups), aby poznać zachowanie związane z wymaganiem wzmianki i allowlistą.

Przykład parowania dla wiadomości prywatnych Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Jeśli niezatwierdzony użytkownik Matrix wciąż wysyła Ci wiadomości przed zatwierdzeniem, OpenClaw ponownie używa tego samego oczekującego kodu parowania i może po krótkim cooldownie ponownie wysłać odpowiedź-przypomnienie zamiast generować nowy kod.

Zobacz [Parowanie](/pl/channels/pairing), aby poznać współdzielony przepływ parowania wiadomości prywatnych i układ przechowywania.

## Naprawa bezpośredniego pokoju

Jeśli stan wiadomości bezpośrednich przestanie być zsynchronizowany, OpenClaw może skończyć ze starymi mapowaniami `m.direct`, które wskazują na stare pokoje solo zamiast na aktywną wiadomość prywatną. Sprawdź bieżące mapowanie dla peera za pomocą:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Napraw je za pomocą:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Przepływ naprawy:

- preferuje ścisłą wiadomość prywatną 1:1, która jest już zmapowana w `m.direct`
- w przeciwnym razie przechodzi do dowolnej aktualnie dołączonej ścisłej wiadomości prywatnej 1:1 z tym użytkownikiem
- tworzy nowy pokój direct i przepisuje `m.direct`, jeśli nie istnieje żaden zdrowy DM

Przepływ naprawy nie usuwa automatycznie starych pokoi. Wybiera tylko zdrową wiadomość prywatną i aktualizuje mapowanie, aby nowe wysyłki Matrix, komunikaty weryfikacyjne i inne przepływy wiadomości bezpośrednich znów trafiały do właściwego pokoju.

## Zatwierdzenia exec

Matrix może działać jako natywny klient zatwierdzeń dla konta Matrix. Natywne
przełączniki routingu wiadomości prywatnych/kanałów nadal znajdują się w konfiguracji zatwierdzeń exec:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (opcjonalnie; fallback do `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, domyślnie: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Zatwierdzający muszą być identyfikatorami użytkowników Matrix, takimi jak `@owner:example.org`. Matrix automatycznie włącza natywne zatwierdzenia, gdy `enabled` jest nieustawione lub ma wartość `"auto"` i można rozwiązać co najmniej jednego zatwierdzającego. Zatwierdzenia exec używają najpierw `execApprovals.approvers` i mogą przejść do `channels.matrix.dm.allowFrom`. Zatwierdzenia wtyczek autoryzują przez `channels.matrix.dm.allowFrom`. Ustaw `enabled: false`, aby jawnie wyłączyć Matrix jako natywnego klienta zatwierdzeń. W przeciwnym razie żądania zatwierdzeń wracają do innych skonfigurowanych tras zatwierdzeń lub do polityki fallback zatwierdzeń.

Natywne routowanie Matrix obsługuje oba typy zatwierdzeń:

- `channels.matrix.execApprovals.*` kontroluje natywny tryb rozsyłania do wiadomości prywatnych/kanałów dla promptów zatwierdzeń Matrix.
- Zatwierdzenia exec używają zbioru zatwierdzających exec z `execApprovals.approvers` lub `channels.matrix.dm.allowFrom`.
- Zatwierdzenia wtyczek używają allowlist wiadomości prywatnych Matrix z `channels.matrix.dm.allowFrom`.
- Skróty reakcji Matrix i aktualizacje wiadomości mają zastosowanie zarówno do zatwierdzeń exec, jak i zatwierdzeń wtyczek.

Zasady dostarczania:

- `target: "dm"` wysyła prompty zatwierdzeń do wiadomości prywatnych zatwierdzających
- `target: "channel"` wysyła prompt z powrotem do źródłowego pokoju Matrix lub wiadomości prywatnej
- `target: "both"` wysyła do wiadomości prywatnych zatwierdzających oraz do źródłowego pokoju Matrix lub wiadomości prywatnej

Prompty zatwierdzeń Matrix inicjalizują skróty reakcji na głównej wiadomości zatwierdzenia:

- `✅` = zezwól raz
- `❌` = odrzuć
- `♾️` = zezwól zawsze, gdy taka decyzja jest dozwolona przez efektywną politykę exec

Zatwierdzający mogą zareagować na tę wiadomość lub użyć zapasowych poleceń ukośnikowych: `/approve <id> allow-once`, `/approve <id> allow-always` albo `/approve <id> deny`.

Tylko rozwiązani zatwierdzający mogą zatwierdzać lub odrzucać. W przypadku zatwierdzeń exec dostarczanie kanałowe zawiera tekst polecenia, więc włączaj `channel` lub `both` tylko w zaufanych pokojach.

Nadpisanie per konto:

- `channels.matrix.accounts.<account>.execApprovals`

Powiązana dokumentacja: [Zatwierdzenia exec](/pl/tools/exec-approvals)

## Wiele kont

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Wartości najwyższego poziomu `channels.matrix` działają jako domyślne dla nazwanych kont, chyba że konto je nadpisze.
Możesz zawęzić dziedziczone wpisy pokoi do jednego konta Matrix za pomocą `groups.<room>.account`.
Wpisy bez `account` pozostają współdzielone między wszystkimi kontami Matrix, a wpisy z `account: "default"` nadal działają, gdy konto domyślne jest skonfigurowane bezpośrednio na najwyższym poziomie `channels.matrix.*`.
Częściowe współdzielone domyślne ustawienia uwierzytelniania same w sobie nie tworzą osobnego niejawnego konta domyślnego. OpenClaw syntetyzuje konto najwyższego poziomu `default` tylko wtedy, gdy to domyślne konto ma świeże uwierzytelnianie (`homeserver` plus `accessToken` lub `homeserver` plus `userId` i `password`); nazwane konta mogą nadal pozostawać wykrywalne z `homeserver` plus `userId`, gdy zbuforowane poświadczenia później spełnią wymagania uwierzytelniania.
Jeśli Matrix ma już dokładnie jedno nazwane konto albo `defaultAccount` wskazuje istniejący klucz nazwanego konta, promocja naprawy/konfiguracji z jednego konta do wielu zachowuje to konto zamiast tworzyć nowy wpis `accounts.default`. Tylko klucze uwierzytelniania/bootstrap Matrix są przenoszone do promowanego konta; współdzielone klucze polityki dostarczania pozostają na najwyższym poziomie.
Ustaw `defaultAccount`, jeśli chcesz, aby OpenClaw preferował jedno nazwane konto Matrix do niejawnego routingu, sondowania i operacji CLI.
Jeśli skonfigurujesz wiele nazwanych kont, ustaw `defaultAccount` lub przekazuj `--account <id>` dla poleceń CLI zależnych od niejawnego wyboru konta.
Przekaż `--account <id>` do `openclaw matrix verify ...` i `openclaw matrix devices ...`, jeśli chcesz nadpisać ten niejawny wybór dla jednego polecenia.

Zobacz [Dokumentacja referencyjna konfiguracji](/pl/gateway/configuration-reference#multi-account-all-channels), aby poznać współdzielony wzorzec wielu kont.

## Prywatne/LAN homeservery

Domyślnie OpenClaw blokuje prywatne/wewnętrzne homeservery Matrix w celu ochrony przed SSRF, chyba że
jawnie włączysz je per konto.

Jeśli Twój homeserver działa na localhost, adresie LAN/Tailscale albo wewnętrznej nazwie hosta, włącz
`network.dangerouslyAllowPrivateNetwork` dla tego konta Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Przykład konfiguracji przez CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

To ustawienie opt-in pozwala tylko na zaufane prywatne/wewnętrzne cele. Publiczne homeservery cleartext takie jak
`http://matrix.example.org:8008` pozostają blokowane. Gdy to możliwe, preferuj `https://`.

## Proxy dla ruchu Matrix

Jeśli wdrożenie Matrix wymaga jawnego wychodzącego proxy HTTP(S), ustaw `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Nazwane konta mogą nadpisywać domyślne ustawienie najwyższego poziomu za pomocą `channels.matrix.accounts.<id>.proxy`.
OpenClaw używa tego samego ustawienia proxy zarówno dla ruchu Matrix w czasie działania, jak i dla sond stanu konta.

## Rozwiązywanie celów

Matrix akceptuje następujące formy celu wszędzie tam, gdzie OpenClaw prosi o pokój lub cel użytkownika:

- Użytkownicy: `@user:server`, `user:@user:server` lub `matrix:user:@user:server`
- Pokoje: `!room:server`, `room:!room:server` lub `matrix:room:!room:server`
- Aliasy: `#alias:server`, `channel:#alias:server` lub `matrix:channel:#alias:server`

Wyszukiwanie w katalogu na żywo używa zalogowanego konta Matrix:

- Wyszukiwanie użytkowników odpytuje katalog użytkowników Matrix na tym homeserverze.
- Wyszukiwanie pokoi akceptuje bezpośrednio jawne ID pokoi i aliasy, a następnie przechodzi do wyszukiwania nazw dołączonych pokoi dla tego konta.
- Wyszukiwanie nazw dołączonych pokoi jest best-effort. Jeśli nazwy pokoju nie da się rozwiązać do ID ani aliasu, jest ignorowana podczas rozwiązywania allowlist w czasie działania.

## Dokumentacja referencyjna konfiguracji

- `enabled`: włącza lub wyłącza kanał.
- `name`: opcjonalna etykieta konta.
- `defaultAccount`: preferowane ID konta, gdy skonfigurowano wiele kont Matrix.
- `homeserver`: URL homeservera, na przykład `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: pozwala temu kontu Matrix łączyć się z prywatnymi/wewnętrznymi homeserverami. Włącz to, gdy homeserver rozwiązuje się do `localhost`, adresu LAN/Tailscale albo wewnętrznego hosta, takiego jak `matrix-synapse`.
- `proxy`: opcjonalny URL proxy HTTP(S) dla ruchu Matrix. Nazwane konta mogą nadpisywać domyślne ustawienie najwyższego poziomu własnym `proxy`.
- `userId`: pełne ID użytkownika Matrix, na przykład `@bot:example.org`.
- `accessToken`: access token do uwierzytelniania opartego na tokenie. Zwykłe wartości tekstowe i wartości SecretRef są obsługiwane dla `channels.matrix.accessToken` i `channels.matrix.accounts.<id>.accessToken` w providerach env/file/exec. Zobacz [Zarządzanie sekretami](/pl/gateway/secrets).
- `password`: hasło do logowania opartego na haśle. Zwykłe wartości tekstowe i wartości SecretRef są obsługiwane.
- `deviceId`: jawne ID urządzenia Matrix.
- `deviceName`: nazwa wyświetlana urządzenia przy logowaniu hasłem.
- `avatarUrl`: zapisany URL własnego awatara do synchronizacji profilu i aktualizacji `profile set`.
- `initialSyncLimit`: maksymalna liczba zdarzeń pobieranych podczas synchronizacji przy uruchamianiu.
- `encryption`: włącza E2EE.
- `allowlistOnly`: gdy ma wartość `true`, podnosi otwartą politykę pokoju `open` do `allowlist` i wymusza `allowlist` dla wszystkich aktywnych zasad wiadomości prywatnych poza `disabled` (w tym `pairing` i `open`). Nie wpływa na zasady `disabled`.
- `allowBots`: pozwala na wiadomości z innych skonfigurowanych kont Matrix OpenClaw (`true` lub `"mentions"`).
- `groupPolicy`: `open`, `allowlist` lub `disabled`.
- `contextVisibility`: tryb widoczności dodatkowego kontekstu pokoju (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: allowlista ID użytkowników dla ruchu w pokojach. Wpisy powinny być pełnymi ID użytkowników Matrix; nierozwiązane nazwy są ignorowane w czasie działania.
- `historyLimit`: maksymalna liczba wiadomości z pokoju uwzględnianych jako kontekst historii grupy. Fallback do `messages.groupChat.historyLimit`; jeśli oba są nieustawione, efektywna wartość domyślna to `0`. Ustaw `0`, aby wyłączyć.
- `replyToMode`: `off`, `first`, `all` lub `batched`.
- `markdown`: opcjonalna konfiguracja renderowania Markdown dla wychodzącego tekstu Matrix.
- `streaming`: `off` (domyślnie), `"partial"`, `"quiet"`, `true` lub `false`. `"partial"` i `true` włączają aktualizacje wersji roboczej „najpierw podgląd” przy użyciu zwykłych wiadomości tekstowych Matrix. `"quiet"` używa niepowiadamiających komunikatów podglądu dla samohostowanych konfiguracji reguł push. `false` jest równoważne `"off"`.
- `blockStreaming`: `true` włącza osobne komunikaty postępu dla ukończonych bloków asystenta, gdy aktywne jest strumieniowanie podglądu wersji roboczej.
- `threadReplies`: `off`, `inbound` lub `always`.
- `threadBindings`: nadpisania per kanał dla routingu i cyklu życia sesji powiązanych z wątkami.
- `startupVerification`: tryb automatycznych żądań samoweryfikacji przy uruchamianiu (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: cooldown przed ponowieniem automatycznych żądań weryfikacji przy uruchamianiu.
- `textChunkLimit`: rozmiar chunków wiadomości wychodzących w znakach (ma zastosowanie, gdy `chunkMode` ma wartość `length`).
- `chunkMode`: `length` dzieli wiadomości według liczby znaków; `newline` dzieli na granicach linii.
- `responsePrefix`: opcjonalny ciąg znaków dodawany na początku wszystkich odpowiedzi wychodzących dla tego kanału.
- `ackReaction`: opcjonalne nadpisanie reakcji potwierdzenia dla tego kanału/konta.
- `ackReactionScope`: opcjonalne nadpisanie zakresu reakcji potwierdzenia (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: tryb przychodzących powiadomień o reakcjach (`own`, `off`).
- `mediaMaxMb`: limit rozmiaru multimediów w MB dla wysyłek wychodzących i przetwarzania multimediów przychodzących.
- `autoJoin`: polityka automatycznego dołączania do zaproszeń (`always`, `allowlist`, `off`). Domyślnie: `off`. Dotyczy wszystkich zaproszeń Matrix, w tym zaproszeń w stylu wiadomości prywatnych.
- `autoJoinAllowlist`: pokoje/aliasy dozwolone, gdy `autoJoin` ma wartość `allowlist`. Wpisy aliasów są rozwiązywane do ID pokoi podczas obsługi zaproszeń; OpenClaw nie ufa stanowi aliasu deklarowanemu przez zaproszony pokój.
- `dm`: blok polityki wiadomości prywatnych (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: kontroluje dostęp do wiadomości prywatnych po dołączeniu OpenClaw do pokoju i sklasyfikowaniu go jako wiadomości prywatnej. Nie zmienia tego, czy zaproszenie zostanie automatycznie zaakceptowane przez dołączenie.
- `dm.allowFrom`: wpisy powinny być pełnymi ID użytkowników Matrix, chyba że zostały już rozwiązane przez wyszukiwanie katalogu na żywo.
- `dm.sessionScope`: `per-user` (domyślnie) lub `per-room`. Użyj `per-room`, gdy chcesz, aby każdy pokój wiadomości prywatnych Matrix zachowywał oddzielny kontekst, nawet jeśli peer jest ten sam.
- `dm.threadReplies`: nadpisanie polityki wątków tylko dla wiadomości prywatnych (`off`, `inbound`, `always`). Nadpisuje ustawienie najwyższego poziomu `threadReplies` zarówno dla umieszczania odpowiedzi, jak i izolacji sesji w wiadomościach prywatnych.
- `execApprovals`: natywne dostarczanie zatwierdzeń exec w Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID użytkowników Matrix, którzy mogą zatwierdzać żądania exec. Opcjonalne, gdy `dm.allowFrom` już identyfikuje zatwierdzających.
- `execApprovals.target`: `dm | channel | both` (domyślnie: `dm`).
- `accounts`: nazwane nadpisania per konto. Wartości najwyższego poziomu `channels.matrix` działają jako wartości domyślne dla tych wpisów.
- `groups`: mapa polityk per pokój. Preferuj ID pokoi lub aliasy; nierozwiązane nazwy pokoi są ignorowane w czasie działania. Tożsamość sesji/grupy używa stabilnego ID pokoju po rozwiązaniu.
- `groups.<room>.account`: ogranicza jeden dziedziczony wpis pokoju do konkretnego konta Matrix w konfiguracjach wielokontowych.
- `groups.<room>.allowBots`: nadpisanie na poziomie pokoju dla nadawców będących skonfigurowanymi botami (`true` lub `"mentions"`).
- `groups.<room>.users`: allowlista nadawców per pokój.
- `groups.<room>.tools`: nadpisania per pokój dla dozwalania/odmawiania narzędzi.
- `groups.<room>.autoReply`: nadpisanie wymagania wzmianki na poziomie pokoju. `true` wyłącza wymaganie wzmianki dla tego pokoju; `false` wymusza je ponownie.
- `groups.<room>.skills`: opcjonalny filtr Skills na poziomie pokoju.
- `groups.<room>.systemPrompt`: opcjonalny fragment system promptu na poziomie pokoju.
- `rooms`: starszy alias dla `groups`.
- `actions`: kontrola użycia narzędzi per akcja (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Powiązane

- [Przegląd kanałów](/pl/channels) — wszystkie obsługiwane kanały
- [Parowanie](/pl/channels/pairing) — uwierzytelnianie wiadomości prywatnych i przepływ parowania
- [Grupy](/pl/channels/groups) — zachowanie czatu grupowego i wymaganie wzmianki
- [Routing kanałów](/pl/channels/channel-routing) — routing sesji dla wiadomości
- [Bezpieczeństwo](/pl/gateway/security) — model dostępu i utwardzanie
