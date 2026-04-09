---
read_when:
    - Chcesz używać Qwen z OpenClaw
    - Wcześniej używałeś OAuth Qwen
summary: Używaj Qwen Cloud przez dołączonego dostawcę qwen w OpenClaw
title: Qwen
x-i18n:
    generated_at: "2026-04-09T01:30:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4786df2cb6ec1ab29d191d012c61dcb0e5468bf0f8561fbbb50eed741efad325
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**OAuth Qwen został usunięty.** Integracja OAuth w bezpłatnym planie
(`qwen-portal`), która używała endpointów `portal.qwen.ai`, nie jest już dostępna.
Kontekst znajdziesz w [Issue #49557](https://github.com/openclaw/openclaw/issues/49557).

</Warning>

## Zalecane: Qwen Cloud

OpenClaw traktuje teraz Qwen jako pełnoprawnego dołączonego dostawcę z kanonicznym identyfikatorem
`qwen`. Dołączony dostawca obsługuje endpointy Qwen Cloud / Alibaba DashScope oraz
Coding Plan i zachowuje działanie starszych identyfikatorów `modelstudio` jako
aliasu zgodności.

- Dostawca: `qwen`
- Preferowana zmienna env: `QWEN_API_KEY`
- Akceptowane także dla zgodności: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- Styl API: zgodny z OpenAI

Jeśli chcesz używać `qwen3.6-plus`, wybierz endpoint **Standard (pay-as-you-go)**.
Obsługa Coding Plan może być opóźniona względem publicznego katalogu.

```bash
# Globalny endpoint Coding Plan
openclaw onboard --auth-choice qwen-api-key

# Chiński endpoint Coding Plan
openclaw onboard --auth-choice qwen-api-key-cn

# Globalny endpoint Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key

# Chiński endpoint Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

Starsze identyfikatory `auth-choice` `modelstudio-*` oraz referencje modeli `modelstudio/...`
nadal działają jako aliasy zgodności, ale nowe przepływy konfiguracji powinny preferować kanoniczne
identyfikatory `auth-choice` z rodziny `qwen-*` oraz referencje modeli `qwen/...`.

Po onboardingu ustaw model domyślny:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## Typy planów i endpointy

| Plan                       | Region | Wybór auth                 | Endpoint                                         |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| Standard (pay-as-you-go)   | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)   | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (subscription) | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (subscription) | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

Dostawca automatycznie wybiera endpoint na podstawie wybranego auth. Kanoniczne
opcje używają rodziny `qwen-*`; `modelstudio-*` pozostaje wyłącznie dla zgodności.
Możesz nadpisać to własnym `baseUrl` w konfiguracji.

Natywne endpointy Model Studio deklarują zgodność użycia strumieniowego na
współdzielonym transporcie `openai-completions`. OpenClaw opiera to teraz na możliwościach endpointu,
więc niestandardowe identyfikatory dostawców zgodnych z DashScope, kierujące na te same
natywne hosty, dziedziczą to samo zachowanie użycia strumieniowego zamiast
wymagać konkretnie wbudowanego identyfikatora dostawcy `qwen`.

## Pobierz swój klucz API

- **Zarządzanie kluczami**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **Dokumentacja**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Wbudowany katalog

OpenClaw obecnie dostarcza ten dołączony katalog Qwen. Skonfigurowany katalog jest
świadomy endpointu: konfiguracje Coding Plan pomijają modele, o których wiadomo, że działają
tylko na endpointach Standard.

| Referencja modelu          | Wejście     | Kontekst  | Uwagi                                              |
| -------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`        | text, image | 1,000,000 | Model domyślny                                     |
| `qwen/qwen3.6-plus`        | text, image | 1,000,000 | Przy tym modelu preferuj endpointy Standard        |
| `qwen/qwen3-max-2026-01-23`| text        | 262,144   | Linia Qwen Max                                     |
| `qwen/qwen3-coder-next`    | text        | 262,144   | Coding                                             |
| `qwen/qwen3-coder-plus`    | text        | 1,000,000 | Coding                                             |
| `qwen/MiniMax-M2.5`        | text        | 1,000,000 | Reasoning włączony                                 |
| `qwen/glm-5`               | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`             | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`           | text, image | 262,144   | Moonshot AI przez Alibaba                          |

Dostępność może nadal zależeć od endpointu i planu rozliczeniowego, nawet jeśli model
jest obecny w dołączonym katalogu.

Zgodność użycia natywnego strumieniowania dotyczy zarówno hostów Coding Plan, jak i
hostów Standard zgodnych z DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Dostępność Qwen 3.6 Plus

`qwen3.6-plus` jest dostępny na endpointach Model Studio Standard (pay-as-you-go):

- China: `dashscope.aliyuncs.com/compatible-mode/v1`
- Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Jeśli endpointy Coding Plan zwracają błąd „unsupported model” dla
`qwen3.6-plus`, przełącz się na Standard (pay-as-you-go) zamiast używać pary
endpoint/klucz Coding Plan.

## Plan możliwości

Rozszerzenie `qwen` jest pozycjonowane jako dom producenta dla pełnej powierzchni Qwen
Cloud, a nie tylko modeli coding/text.

- Modele text/chat: dołączone już teraz
- Wywoływanie narzędzi, output strukturalny, thinking: dziedziczone z transportu zgodnego z OpenAI
- Generowanie obrazów: planowane na warstwie wtyczki dostawcy
- Rozumienie obrazów/wideo: dołączone już teraz na endpointach Standard
- Speech/audio: planowane na warstwie wtyczki dostawcy
- Embeddingi pamięci/reranking: planowane przez powierzchnię adaptera embeddingów
- Generowanie wideo: dołączone już teraz przez współdzieloną możliwość generowania wideo

## Dodatki multimodalne

Rozszerzenie `qwen` udostępnia teraz także:

- Rozumienie wideo przez `qwen-vl-max-latest`
- Generowanie wideo Wan przez:
  - `wan2.6-t2v` (domyślnie)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

Te powierzchnie multimodalne używają endpointów DashScope **Standard**, a nie
endpointów Coding Plan.

- Global/Intl Standard base URL: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- China Standard base URL: `https://dashscope.aliyuncs.com/compatible-mode/v1`

Dla generowania wideo OpenClaw mapuje skonfigurowany region Qwen na odpowiadający
mu host DashScope AIGC przed wysłaniem zadania:

- Global/Intl: `https://dashscope-intl.aliyuncs.com`
- China: `https://dashscope.aliyuncs.com`

Oznacza to, że zwykłe `models.providers.qwen.baseUrl` wskazujące na hosty Qwen
Coding Plan lub Standard nadal utrzymuje generowanie wideo na poprawnym
regionalnym endpointcie wideo DashScope.

Dla generowania wideo ustaw jawnie model domyślny:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Obecne limity generowania wideo dla dołączonego Qwen:

- Maksymalnie **1** wyjściowe wideo na żądanie
- Maksymalnie **1** obraz wejściowy
- Maksymalnie **4** wejściowe wideo
- Maksymalny czas trwania **10 sekund**
- Obsługuje `size`, `aspectRatio`, `resolution`, `audio` i `watermark`
- Tryb obrazu/wideo referencyjnego obecnie wymaga **zdalnych adresów URL http(s)**. Lokalne
  ścieżki plików są odrzucane od razu, ponieważ endpoint wideo DashScope nie
  akceptuje przesyłanych lokalnych buforów dla takich referencji.

Zobacz [Video Generation](/pl/tools/video-generation), aby poznać współdzielone parametry
narzędzia, wybór dostawcy i zachowanie failover.

## Uwaga dotycząca środowiska

Jeśli Gateway działa jako demon (launchd/systemd), upewnij się, że `QWEN_API_KEY` jest
dostępne dla tego procesu (na przykład w `~/.openclaw/.env` lub przez
`env.shellEnv`).
