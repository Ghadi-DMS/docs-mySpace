---
read_when:
    - OpenClaw'ı yerel bir inferrs sunucusuna karşı çalıştırmak istiyorsunuz
    - Gemma veya başka bir modeli inferrs üzerinden sunuyorsunuz
    - inferrs için tam OpenClaw uyumluluk bayraklarına ihtiyacınız var
summary: OpenClaw'ı inferrs (OpenAI uyumlu yerel sunucu) üzerinden çalıştırın
title: inferrs
x-i18n:
    generated_at: "2026-04-09T01:29:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 03b9d5a9935c75fd369068bacb7807a5308cd0bd74303b664227fb664c3a2098
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs), yerel modelleri
OpenAI uyumlu bir `/v1` API arkasında sunabilir. OpenClaw, `inferrs` ile genel
`openai-completions` yolu üzerinden çalışır.

`inferrs`, şu anda özel olarak self-hosted bir OpenAI uyumlu
backend olarak ele alınmalıdır; özel bir OpenClaw sağlayıcı plugin'i olarak değil.

## Hızlı başlangıç

1. `inferrs`'ü bir modelle başlatın.

Örnek:

```bash
inferrs serve google/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. Sunucuya erişilebildiğini doğrulayın.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. Açık bir OpenClaw sağlayıcı girdisi ekleyin ve varsayılan modelinizi buna yönlendirin.

## Tam yapılandırma örneği

Bu örnek, yerel bir `inferrs` sunucusunda Gemma 4 kullanır.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## `requiresStringContent` neden önemlidir

Bazı `inferrs` Chat Completions rotaları yalnızca dize
`messages[].content` kabul eder, yapılandırılmış içerik parçası dizilerini değil.

OpenClaw çalıştırmaları aşağıdaki gibi bir hatayla başarısız olursa:

```text
messages[1].content: invalid type: sequence, expected a string
```

şunu ayarlayın:

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw, isteği göndermeden önce yalnızca metin içeren içerik parçalarını düz dizelere indirger.

## Gemma ve araç şeması kısıtı

Bazı güncel `inferrs` + Gemma birleşimleri, küçük doğrudan
`/v1/chat/completions` isteklerini kabul eder ancak yine de tam OpenClaw agent çalışma zamanı
dönüşlerinde başarısız olur.

Böyle olursa, önce şunu deneyin:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

Bu, model için OpenClaw'ın araç şeması yüzeyini devre dışı bırakır ve daha katı yerel backend'lerde istem
yükünü azaltabilir.

Küçük doğrudan istekler yine de çalışıyor ancak normal OpenClaw agent dönüşleri
`inferrs` içinde çökmeye devam ediyorsa, kalan sorun genellikle OpenClaw'ın taşıma katmanından ziyade yukarı akış model/sunucu davranışıdır.

## Elle smoke test

Yapılandırıldıktan sonra her iki katmanı da test edin:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/google/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

İlk komut çalışıyor ancak ikincisi başarısız oluyorsa, aşağıdaki sorun giderme notlarını
kullanın.

## Sorun giderme

- `curl /v1/models` başarısız oluyor: `inferrs` çalışmıyor, erişilemiyor veya
  beklenen host/port'a bağlanmamış.
- `messages[].content ... expected a string`: şunu ayarlayın:
  `compat.requiresStringContent: true`.
- Doğrudan küçük `/v1/chat/completions` çağrıları başarılı, ancak `openclaw infer model run`
  başarısız oluyor: `compat.supportsTools: false` deneyin.
- OpenClaw artık şema hataları almıyor, ancak `inferrs` daha büyük
  agent dönüşlerinde hâlâ çöküyor: bunu yukarı akış `inferrs` veya model kısıtı olarak değerlendirin ve
  istem yükünü azaltın ya da yerel backend/model değiştirin.

## Proxy benzeri davranış

`inferrs`, yerel bir
OpenAI uç noktası değil, proxy benzeri bir OpenAI uyumlu `/v1` backend olarak ele alınır.

- yerel OpenAI'ye özgü istek şekillendirme burada uygulanmaz
- `service_tier`, Responses `store`, prompt-cache ipuçları ve
  OpenAI reasoning uyumluluk payload şekillendirmesi yoktur
- gizli OpenClaw atıf üstbilgileri (`originator`, `version`, `User-Agent`)
  özel `inferrs` base URL'lerine enjekte edilmez

## Ayrıca bakın

- [Yerel modeller](/tr/gateway/local-models)
- [Gateway sorun giderme](/tr/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Model sağlayıcıları](/tr/concepts/model-providers)
