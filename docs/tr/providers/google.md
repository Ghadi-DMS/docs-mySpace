---
read_when:
    - OpenClaw ile Google Gemini modellerini kullanmak istiyorsunuz
    - API anahtarı veya OAuth kimlik doğrulama akışına ihtiyacınız var
summary: Google Gemini kurulumu (API anahtarı + OAuth, görsel oluşturma, medya anlama, web arama)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T08:43:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Google eklentisi, Google AI Studio üzerinden Gemini modellerine erişim sağlar; ayrıca
Gemini Grounding aracılığıyla görsel oluşturma, medya anlama (görsel/ses/video) ve web aramayı da destekler.

- Sağlayıcı: `google`
- Kimlik doğrulama: `GEMINI_API_KEY` veya `GOOGLE_API_KEY`
- API: Google Gemini API
- Alternatif sağlayıcı: `google-gemini-cli` (OAuth)

## Hızlı başlangıç

1. API anahtarını ayarlayın:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Varsayılan bir model ayarlayın:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Etkileşimsiz örnek

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Alternatif bir sağlayıcı olan `google-gemini-cli`, API anahtarı yerine PKCE OAuth kullanır.
Bu resmi olmayan bir entegrasyondur; bazı kullanıcılar hesap
kısıtlamaları bildirmektedir. Riski size ait olacak şekilde kullanın.

- Varsayılan model: `google-gemini-cli/gemini-3-flash-preview`
- Takma ad: `gemini-cli`
- Kurulum önkoşulu: yerel Gemini CLI'ın `gemini` olarak kullanılabilir olması
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- Giriş:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Ortam değişkenleri:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(Veya `GEMINI_CLI_*` varyantları.)

Giriş yaptıktan sonra Gemini CLI OAuth istekleri başarısız olursa,
gateway host üzerinde `GOOGLE_CLOUD_PROJECT` veya `GOOGLE_CLOUD_PROJECT_ID` ayarlayın ve
yeniden deneyin.

Tarayıcı akışı başlamadan önce giriş başarısız olursa, yerel `gemini`
komutunun kurulu olduğundan ve `PATH` üzerinde bulunduğundan emin olun. OpenClaw hem Homebrew kurulumlarını
hem de genel npm kurulumlarını destekler; buna yaygın Windows/npm düzenleri de dahildir.

Gemini CLI JSON kullanım notları:

- Yanıt metni, CLI JSON içindeki `response` alanından gelir.
- CLI `usage` alanını boş bıraktığında kullanım, `stats` alanına geri döner.
- `stats.cached`, OpenClaw `cacheRead` içine normalize edilir.
- `stats.input` eksikse OpenClaw, giriş tokenlerini
  `stats.input_tokens - stats.cached` değerinden türetir.

## Yetenekler

| Yetenek                | Destekleniyor     |
| ---------------------- | ----------------- |
| Sohbet tamamlamaları   | Evet              |
| Görsel oluşturma       | Evet              |
| Müzik oluşturma        | Evet              |
| Görsel anlama          | Evet              |
| Ses dökümü             | Evet              |
| Video anlama           | Evet              |
| Web arama (Grounding)  | Evet              |
| Düşünme/muhakeme       | Evet (Gemini 3.1+) |
| Gemma 4 modelleri      | Evet              |

Gemma 4 modelleri (örneğin `gemma-4-26b-a4b-it`) düşünme modunu destekler. OpenClaw, Gemma 4 için `thinkingBudget` değerini desteklenen bir Google `thinkingLevel` değerine yeniden yazar. Düşünmeyi `off` olarak ayarlamak, bunu `MINIMAL` değerine eşlemek yerine düşünmenin devre dışı kalmasını korur.

## Doğrudan Gemini önbellek yeniden kullanımı

Doğrudan Gemini API çalıştırmaları için (`api: "google-generative-ai"`), OpenClaw artık
yapılandırılmış bir `cachedContent` tanıtıcısını Gemini isteklerine iletir.

- Model başına veya genel parametreleri
  `cachedContent` ya da eski `cached_content` ile yapılandırın
- İkisi de varsa `cachedContent` önceliklidir
- Örnek değer: `cachedContents/prebuilt-context`
- Gemini önbellek isabeti kullanımı, yukarı akıştaki `cachedContentTokenCount` değerinden
  OpenClaw `cacheRead` içine normalize edilir

Örnek:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Görsel oluşturma

Paketle gelen `google` görsel oluşturma sağlayıcısı varsayılan olarak
`google/gemini-3.1-flash-image-preview` kullanır.

- Ayrıca `google/gemini-3-pro-image-preview` da desteklenir
- Oluşturma: istek başına en fazla 4 görsel
- Düzenleme modu: etkin, en fazla 5 giriş görseli
- Geometri denetimleri: `size`, `aspectRatio` ve `resolution`

Yalnızca OAuth kullanan `google-gemini-cli` sağlayıcısı ayrı bir metin çıkarımı
yüzeyidir. Görsel oluşturma, medya anlama ve Gemini Grounding
`google` sağlayıcı kimliğinde kalır.

Google'ı varsayılan görsel sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve devralma davranışı için [Görsel Oluşturma](/tr/tools/image-generation) bölümüne bakın.

## Video oluşturma

Paketle gelen `google` eklentisi, paylaşılan
`video_generate` aracı üzerinden video oluşturmayı da kaydeder.

- Varsayılan video modeli: `google/veo-3.1-fast-generate-preview`
- Modlar: metinden videoya, görselden videoya ve tek video referans akışları
- `aspectRatio`, `resolution` ve `audio` desteklenir
- Mevcut süre sınırı: **4 ila 8 saniye**

Google'ı varsayılan video sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve devralma davranışı için [Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.

## Müzik oluşturma

Paketle gelen `google` eklentisi, paylaşılan
`music_generate` aracı üzerinden müzik oluşturmayı da kaydeder.

- Varsayılan müzik modeli: `google/lyria-3-clip-preview`
- Ayrıca `google/lyria-3-pro-preview` da desteklenir
- İstem denetimleri: `lyrics` ve `instrumental`
- Çıkış biçimi: varsayılan olarak `mp3`, ayrıca `google/lyria-3-pro-preview` üzerinde `wav`
- Referans girdileri: en fazla 10 görsel
- Oturum destekli çalıştırmalar, `action: "status"` dahil olmak üzere paylaşılan görev/durum akışı üzerinden ayrıştırılır

Google'ı varsayılan müzik sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve devralma davranışı için [Müzik Oluşturma](/tr/tools/music-generation) bölümüne bakın.

## Ortam notu

Gateway bir daemon olarak çalışıyorsa (launchd/systemd), `GEMINI_API_KEY`
değerinin bu süreç için kullanılabilir olduğundan emin olun (örneğin `~/.openclaw/.env` içinde veya
`env.shellEnv` aracılığıyla).
