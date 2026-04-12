---
read_when:
    - Włączanie syntezy mowy dla odpowiedzi
    - Konfigurowanie dostawców TTS lub limitów
    - Używanie komend `/tts`
summary: Synteza mowy (TTS) dla odpowiedzi wychodzących
title: Synteza mowy tekstu na mowę
x-i18n:
    generated_at: "2026-04-12T23:34:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: ad79a6be34879347dc73fdab1bd219823cd7c6aa8504e3e4c73e1a0554c837c5
    source_path: tools/tts.md
    workflow: 15
---

# Synteza mowy (TTS)

OpenClaw może konwertować odpowiedzi wychodzące na audio za pomocą ElevenLabs, Microsoft, MiniMax lub OpenAI.
Działa wszędzie tam, gdzie OpenClaw może wysyłać audio.

## Obsługiwane usługi

- **ElevenLabs** (dostawca główny lub fallback)
- **Microsoft** (dostawca główny lub fallback; obecna dołączona implementacja używa `node-edge-tts`)
- **MiniMax** (dostawca główny lub fallback; używa API T2A v2)
- **OpenAI** (dostawca główny lub fallback; używany także do podsumowań)

### Uwagi dotyczące mowy Microsoft

Dołączony dostawca mowy Microsoft obecnie używa usługi online neural TTS Microsoft Edge
przez bibliotekę `node-edge-tts`. Jest to usługa hostowana (nie lokalna),
używa endpointów Microsoft i nie wymaga klucza API.
`node-edge-tts` udostępnia opcje konfiguracji mowy i formaty wyjściowe, ale
nie wszystkie opcje są obsługiwane przez usługę. Legacy konfiguracja i wejście dyrektywy
używające `edge` nadal działają i są normalizowane do `microsoft`.

Ponieważ ta ścieżka to publiczna usługa web bez opublikowanego SLA ani limitu quota,
traktuj ją jako best-effort. Jeśli potrzebujesz gwarantowanych limitów i wsparcia, użyj OpenAI
lub ElevenLabs.

## Opcjonalne klucze

Jeśli chcesz używać OpenAI, ElevenLabs lub MiniMax:

- `ELEVENLABS_API_KEY` (lub `XI_API_KEY`)
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

Mowa Microsoft **nie** wymaga klucza API.

Jeśli skonfigurowano wielu dostawców, najpierw używany jest wybrany dostawca, a pozostali pełnią rolę opcji fallback.
Auto-summary używa skonfigurowanego `summaryModel` (lub `agents.defaults.model.primary`),
więc ten dostawca także musi być uwierzytelniony, jeśli włączysz podsumowania.

## Linki do usług

- [OpenAI Text-to-Speech guide](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API reference](https://platform.openai.com/docs/api-reference/audio)
- [ElevenLabs Text to Speech](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs Authentication](https://elevenlabs.io/docs/api-reference/authentication)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech output formats](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## Czy jest włączone domyślnie?

Nie. Auto‑TTS jest domyślnie **wyłączone**. Włącz je w konfiguracji za pomocą
`messages.tts.auto` lub lokalnie przez `/tts on`.

Gdy `messages.tts.provider` nie jest ustawione, OpenClaw wybiera pierwszego skonfigurowanego
dostawcę mowy według kolejności automatycznego wyboru w rejestrze.

## Konfiguracja

Konfiguracja TTS znajduje się w `messages.tts` w `openclaw.json`.
Pełny schemat znajduje się w [Gateway configuration](/pl/gateway/configuration).

### Minimalna konfiguracja (włączenie + dostawca)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
    },
  },
}
```

### OpenAI jako główny z fallbackiem ElevenLabs

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true,
      },
      providers: {
        openai: {
          apiKey: "openai_api_key",
          baseUrl: "https://api.openai.com/v1",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "elevenlabs_api_key",
          baseUrl: "https://api.elevenlabs.io",
          voiceId: "voice_id",
          modelId: "eleven_multilingual_v2",
          seed: 42,
          applyTextNormalization: "auto",
          languageCode: "en",
          voiceSettings: {
            stability: 0.5,
            similarityBoost: 0.75,
            style: 0.0,
            useSpeakerBoost: true,
            speed: 1.0,
          },
        },
      },
    },
  },
}
```

### Microsoft jako główny (bez klucza API)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+10%",
          pitch: "-5%",
        },
      },
    },
  },
}
```

### MiniMax jako główny

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "minimax",
      providers: {
        minimax: {
          apiKey: "minimax_api_key",
          baseUrl: "https://api.minimax.io",
          model: "speech-2.8-hd",
          voiceId: "English_expressive_narrator",
          speed: 1.0,
          vol: 1.0,
          pitch: 0,
        },
      },
    },
  },
}
```

### Wyłącz mowę Microsoft

```json5
{
  messages: {
    tts: {
      providers: {
        microsoft: {
          enabled: false,
        },
      },
    },
  },
}
```

### Niestandardowe limity + ścieżka prefs

```json5
{
  messages: {
    tts: {
      auto: "always",
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
    },
  },
}
```

### Odpowiadaj audio tylko po przychodzącej wiadomości głosowej

```json5
{
  messages: {
    tts: {
      auto: "inbound",
    },
  },
}
```

### Wyłącz auto-summary dla długich odpowiedzi

```json5
{
  messages: {
    tts: {
      auto: "always",
    },
  },
}
```

Następnie uruchom:

```
/tts summary off
```

### Uwagi o polach

- `auto`: tryb auto‑TTS (`off`, `always`, `inbound`, `tagged`).
  - `inbound` wysyła audio tylko po przychodzącej wiadomości głosowej.
  - `tagged` wysyła audio tylko wtedy, gdy odpowiedź zawiera dyrektywy `[[tts:key=value]]` lub blok `[[tts:text]]...[[/tts:text]]`.
- `enabled`: przełącznik legacy (doctor migruje to do `auto`).
- `mode`: `"final"` (domyślnie) lub `"all"` (obejmuje odpowiedzi narzędzi/bloków).
- `provider`: identyfikator dostawcy mowy, taki jak `"elevenlabs"`, `"microsoft"`, `"minimax"` lub `"openai"` (fallback jest automatyczny).
- Jeśli `provider` jest **nieustawiony**, OpenClaw używa pierwszego skonfigurowanego dostawcy mowy według kolejności automatycznego wyboru w rejestrze.
- Legacy `provider: "edge"` nadal działa i jest normalizowane do `microsoft`.
- `summaryModel`: opcjonalny tani model dla auto-summary; domyślnie `agents.defaults.model.primary`.
  - Akceptuje `provider/model` lub skonfigurowany alias modelu.
- `modelOverrides`: pozwala modelowi emitować dyrektywy TTS (domyślnie włączone).
  - `allowProvider` domyślnie ma wartość `false` (przełączanie dostawcy jest opt-in).
- `providers.<id>`: ustawienia należące do dostawcy, kluczowane identyfikatorem dostawcy mowy.
- Legacy bezpośrednie bloki dostawców (`messages.tts.openai`, `messages.tts.elevenlabs`, `messages.tts.microsoft`, `messages.tts.edge`) są automatycznie migrowane przy ładowaniu do `messages.tts.providers.<id>`.
- `maxTextLength`: twardy limit wejścia TTS (znaki). `/tts audio` kończy się błędem po jego przekroczeniu.
- `timeoutMs`: timeout żądania (ms).
- `prefsPath`: nadpisuje lokalną ścieżkę JSON preferencji (dostawca/limit/podsumowanie).
- Wartości `apiKey` używają fallbacku do zmiennych środowiskowych (`ELEVENLABS_API_KEY`/`XI_API_KEY`, `MINIMAX_API_KEY`, `OPENAI_API_KEY`).
- `providers.elevenlabs.baseUrl`: nadpisuje bazowy URL API ElevenLabs.
- `providers.openai.baseUrl`: nadpisuje endpoint OpenAI TTS.
  - Kolejność rozwiązywania: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - Wartości inne niż domyślna są traktowane jako endpointy TTS zgodne z OpenAI, więc akceptowane są niestandardowe nazwy modeli i głosów.
- `providers.elevenlabs.voiceSettings`:
  - `stability`, `similarityBoost`, `style`: `0..1`
  - `useSpeakerBoost`: `true|false`
  - `speed`: `0.5..2.0` (1.0 = normalnie)
- `providers.elevenlabs.applyTextNormalization`: `auto|on|off`
- `providers.elevenlabs.languageCode`: 2-literowy kod ISO 639-1 (np. `en`, `de`)
- `providers.elevenlabs.seed`: liczba całkowita `0..4294967295` (best-effort determinism)
- `providers.minimax.baseUrl`: nadpisuje bazowy URL API MiniMax (domyślnie `https://api.minimax.io`, env: `MINIMAX_API_HOST`).
- `providers.minimax.model`: model TTS (domyślnie `speech-2.8-hd`, env: `MINIMAX_TTS_MODEL`).
- `providers.minimax.voiceId`: identyfikator głosu (domyślnie `English_expressive_narrator`, env: `MINIMAX_TTS_VOICE_ID`).
- `providers.minimax.speed`: szybkość odtwarzania `0.5..2.0` (domyślnie 1.0).
- `providers.minimax.vol`: głośność `(0, 10]` (domyślnie 1.0; musi być większa od 0).
- `providers.minimax.pitch`: przesunięcie tonu `-12..12` (domyślnie 0).
- `providers.microsoft.enabled`: zezwala na użycie mowy Microsoft (domyślnie `true`; bez klucza API).
- `providers.microsoft.voice`: nazwa neural głosu Microsoft (np. `en-US-MichelleNeural`).
- `providers.microsoft.lang`: kod języka (np. `en-US`).
- `providers.microsoft.outputFormat`: format wyjściowy Microsoft (np. `audio-24khz-48kbitrate-mono-mp3`).
  - Prawidłowe wartości znajdziesz w Microsoft Speech output formats; nie wszystkie formaty są obsługiwane przez dołączony transport oparty na Edge.
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: ciągi procentowe (np. `+10%`, `-5%`).
- `providers.microsoft.saveSubtitles`: zapisuje napisy JSON obok pliku audio.
- `providers.microsoft.proxy`: URL proxy dla żądań mowy Microsoft.
- `providers.microsoft.timeoutMs`: nadpisanie timeoutu żądania (ms).
- `edge.*`: alias legacy dla tych samych ustawień Microsoft.

## Nadpisania sterowane przez model (domyślnie włączone)

Domyślnie model **może** emitować dyrektywy TTS dla pojedynczej odpowiedzi.
Gdy `messages.tts.auto` ma wartość `tagged`, te dyrektywy są wymagane do wyzwolenia audio.

Po włączeniu model może emitować dyrektywy `[[tts:...]]`, aby nadpisać głos
dla pojedynczej odpowiedzi, plus opcjonalny blok `[[tts:text]]...[[/tts:text]]`,
aby przekazać ekspresyjne znaczniki (śmiech, wskazówki śpiewu itp.), które powinny pojawić się tylko w
audio.

Dyrektywy `provider=...` są ignorowane, chyba że `modelOverrides.allowProvider: true`.

Przykładowy payload odpowiedzi:

```
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

Dostępne klucze dyrektyw (gdy włączone):

- `provider` (zarejestrowany identyfikator dostawcy mowy, na przykład `openai`, `elevenlabs`, `minimax` lub `microsoft`; wymaga `allowProvider: true`)
- `voice` (głos OpenAI) lub `voiceId` (ElevenLabs / MiniMax)
- `model` (model OpenAI TTS, identyfikator modelu ElevenLabs lub model MiniMax)
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (głośność MiniMax, 0-10)
- `pitch` (ton MiniMax, -12 do 12)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

Wyłącz wszystkie nadpisania modelu:

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: false,
      },
    },
  },
}
```

Opcjonalna allowlista (włącza przełączanie dostawcy przy zachowaniu możliwości konfiguracji innych parametrów):

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: true,
        allowProvider: true,
        allowSeed: false,
      },
    },
  },
}
```

## Preferencje per użytkownik

Komendy slash zapisują lokalne nadpisania do `prefsPath` (domyślnie:
`~/.openclaw/settings/tts.json`, nadpisywane przez `OPENCLAW_TTS_PREFS` lub
`messages.tts.prefsPath`).

Przechowywane pola:

- `enabled`
- `provider`
- `maxLength` (próg podsumowania; domyślnie 1500 znaków)
- `summarize` (domyślnie `true`)

Nadpisują one `messages.tts.*` dla tego hosta.

## Formaty wyjściowe (stałe)

- **Feishu / Matrix / Telegram / WhatsApp**: wiadomość głosowa Opus (`opus_48000_64` z ElevenLabs, `opus` z OpenAI).
  - 48 kHz / 64 kb/s to dobry kompromis dla wiadomości głosowych.
- **Inne kanały**: MP3 (`mp3_44100_128` z ElevenLabs, `mp3` z OpenAI).
  - 44,1 kHz / 128 kb/s to domyślny balans dla czytelności mowy.
- **MiniMax**: MP3 (`speech-2.8-hd`, częstotliwość próbkowania 32 kHz). Format notatki głosowej nie jest natywnie obsługiwany; użyj OpenAI lub ElevenLabs, jeśli potrzebujesz gwarantowanych wiadomości głosowych Opus.
- **Microsoft**: używa `microsoft.outputFormat` (domyślnie `audio-24khz-48kbitrate-mono-mp3`).
  - Dołączony transport akceptuje `outputFormat`, ale nie wszystkie formaty są dostępne w usłudze.
  - Wartości formatu wyjściowego są zgodne z Microsoft Speech output formats (w tym Ogg/WebM Opus).
  - Telegram `sendVoice` akceptuje OGG/MP3/M4A; użyj OpenAI/ElevenLabs, jeśli potrzebujesz
    gwarantowanych wiadomości głosowych Opus.
  - Jeśli skonfigurowany format wyjściowy Microsoft zakończy się niepowodzeniem, OpenClaw ponowi próbę z MP3.

Formaty wyjściowe OpenAI/ElevenLabs są stałe dla każdego kanału (zobacz wyżej).

## Zachowanie auto-TTS

Po włączeniu OpenClaw:

- pomija TTS, jeśli odpowiedź już zawiera multimedia lub dyrektywę `MEDIA:`.
- pomija bardzo krótkie odpowiedzi (< 10 znaków).
- podsumowuje długie odpowiedzi, jeśli ta opcja jest włączona, używając `agents.defaults.model.primary` (lub `summaryModel`).
- dołącza wygenerowane audio do odpowiedzi.

Jeśli odpowiedź przekracza `maxLength`, a summary jest wyłączone (lub brak klucza API dla
modelu summary), audio
jest pomijane i wysyłana jest zwykła odpowiedź tekstowa.

## Diagram przepływu

```
Odpowiedź -> TTS włączone?
  nie -> wyślij tekst
  tak -> zawiera multimedia / MEDIA: / krótka?
          tak -> wyślij tekst
          nie -> długość > limit?
                   nie -> TTS -> dołącz audio
                   tak -> summary włączone?
                            nie -> wyślij tekst
                            tak -> podsumuj (summaryModel lub agents.defaults.model.primary)
                                      -> TTS -> dołącz audio
```

## Użycie komend slash

Istnieje jedna komenda: `/tts`.
Szczegóły włączania znajdziesz w [Slash commands](/pl/tools/slash-commands).

Uwaga dotycząca Discord: `/tts` to wbudowana komenda Discord, więc OpenClaw rejestruje tam
`/voice` jako komendę natywną. Tekstowe `/tts ...` nadal działa.

```
/tts off
/tts on
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from OpenClaw
```

Uwagi:

- Komendy wymagają autoryzowanego nadawcy (nadal obowiązują reguły allowlisty/właściciela).
- `commands.text` lub natywna rejestracja komend muszą być włączone.
- Konfiguracja `messages.tts.auto` akceptuje `off|always|inbound|tagged`.
- `/tts on` zapisuje lokalną preferencję TTS jako `always`; `/tts off` zapisuje ją jako `off`.
- Użyj konfiguracji, gdy chcesz mieć domyślne `inbound` lub `tagged`.
- `limit` i `summary` są przechowywane w lokalnych preferencjach, a nie w głównej konfiguracji.
- `/tts audio` generuje jednorazową odpowiedź audio (nie włącza TTS).
- `/tts status` zawiera widoczność fallbacku dla ostatniej próby:
  - fallback zakończony sukcesem: `Fallback: <primary> -> <used>` plus `Attempts: ...`
  - awaria: `Error: ...` plus `Attempts: ...`
  - szczegółowa diagnostyka: `Attempt details: provider:outcome(reasonCode) latency`
- Awaria API OpenAI i ElevenLabs zawiera teraz sparsowane szczegóły błędu dostawcy oraz request id (jeśli zostało zwrócone przez dostawcę), co jest pokazywane w błędach/logach TTS.

## Narzędzie agenta

Narzędzie `tts` konwertuje tekst na mowę i zwraca załącznik audio do
dostarczenia w odpowiedzi. Gdy kanałem jest Feishu, Matrix, Telegram lub WhatsApp,
audio jest dostarczane jako wiadomość głosowa, a nie jako załącznik pliku.

## Gateway RPC

Metody Gateway:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
