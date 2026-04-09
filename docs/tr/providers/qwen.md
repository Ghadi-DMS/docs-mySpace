---
read_when:
    - Qwen'i OpenClaw ile kullanmak istiyorsunuz
    - Daha önce Qwen OAuth kullandınız
summary: OpenClaw'ın paketlenmiş qwen sağlayıcısı üzerinden Qwen Cloud kullanın
title: Qwen
x-i18n:
    generated_at: "2026-04-09T01:30:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4786df2cb6ec1ab29d191d012c61dcb0e5468bf0f8561fbbb50eed741efad325
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth kaldırıldı.** `portal.qwen.ai` uç noktalarını kullanan ücretsiz katman OAuth entegrasyonu
(`qwen-portal`) artık kullanılamıyor.
Arka plan için [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) sayfasına bakın.

</Warning>

## Önerilen: Qwen Cloud

OpenClaw artık Qwen'i standart kimliği
`qwen` olan birinci sınıf paketlenmiş sağlayıcı olarak ele alıyor. Paketlenmiş sağlayıcı, Qwen Cloud / Alibaba DashScope ve
Coding Plan uç noktalarını hedefler ve eski `modelstudio` kimliklerini
uyumluluk takma adı olarak çalışır durumda tutar.

- Sağlayıcı: `qwen`
- Tercih edilen ortam değişkeni: `QWEN_API_KEY`
- Uyumluluk için ayrıca kabul edilir: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- API stili: OpenAI uyumlu

`qwen3.6-plus` kullanmak istiyorsanız, **Standard (kullandıkça öde)** uç noktasını tercih edin.
Coding Plan desteği, herkese açık kataloğun gerisinde kalabilir.

```bash
# Global Coding Plan uç noktası
openclaw onboard --auth-choice qwen-api-key

# China Coding Plan uç noktası
openclaw onboard --auth-choice qwen-api-key-cn

# Global Standard (kullandıkça öde) uç noktası
openclaw onboard --auth-choice qwen-standard-api-key

# China Standard (kullandıkça öde) uç noktası
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

Eski `modelstudio-*` auth-choice kimlikleri ve `modelstudio/...` model referansları hâlâ
uyumluluk takma adları olarak çalışır, ancak yeni kurulum akışlarında standart
`qwen-*` auth-choice kimlikleri ve `qwen/...` model referansları tercih edilmelidir.

Onboarding sonrasında varsayılan bir model ayarlayın:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## Plan türleri ve uç noktalar

| Plan                       | Bölge  | Auth choice                | Uç nokta                                         |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| Standard (kullandıkça öde) | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (kullandıkça öde) | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (abonelik)     | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (abonelik)     | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

Sağlayıcı, auth choice seçiminize göre uç noktayı otomatik seçer. Standart
seçimler `qwen-*` ailesini kullanır; `modelstudio-*` yalnızca uyumluluk içindir.
Yapılandırmada özel bir `baseUrl` ile geçersiz kılabilirsiniz.

Yerel Model Studio uç noktaları, paylaşılan
`openai-completions` taşıması üzerinde akış kullanımı uyumluluğu bildirir. OpenClaw artık bunu uç nokta
yeteneklerine göre belirliyor; bu nedenle aynı yerel host'ları hedefleyen DashScope uyumlu özel sağlayıcı kimlikleri,
özellikle yerleşik `qwen` sağlayıcı kimliğini gerektirmek yerine aynı akış kullanımı davranışını devralır.

## API anahtarınızı alın

- **Anahtarları yönetin**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **Belgeler**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Yerleşik katalog

OpenClaw şu anda bu paketlenmiş Qwen kataloğunu sunar. Yapılandırılmış katalog
uç nokta farkındalıklıdır: Coding Plan yapılandırmaları, yalnızca
Standard uç noktasında çalıştığı bilinen modelleri dışarıda bırakır.

| Model ref                   | Girdi       | Bağlam    | Notlar                                             |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | Varsayılan model                                   |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | Bu modele ihtiyacınız varsa Standard uç noktalarını tercih edin |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Qwen Max hattı                                     |
| `qwen/qwen3-coder-next`     | text        | 262,144   | Kodlama                                            |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | Kodlama                                            |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | Akıl yürütme etkin                                 |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Alibaba üzerinden Moonshot AI                      |

Bir model paketlenmiş katalogda mevcut olsa bile kullanılabilirlik yine de uç noktaya ve faturalama planına göre değişebilir.

Yerel akış kullanımı uyumluluğu hem Coding Plan host'larına hem de
Standard DashScope uyumlu host'lara uygulanır:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Qwen 3.6 Plus kullanılabilirliği

`qwen3.6-plus`, Standard (kullandıkça öde) Model Studio
uç noktalarında kullanılabilir:

- China: `dashscope.aliyuncs.com/compatible-mode/v1`
- Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Coding Plan uç noktaları
`qwen3.6-plus` için "unsupported model" hatası döndürüyorsa, Coding Plan
uç noktası/anahtar çifti yerine Standard (kullandıkça öde) kullanın.

## Yetenek planı

`qwen` uzantısı, yalnızca kodlama/metin modelleri için değil,
tam Qwen Cloud yüzeyi için üretici evi olarak konumlandırılıyor.

- Metin/sohbet modelleri: şimdi paketlenmiş
- Araç çağırma, yapılandırılmış çıktı, düşünme: OpenAI uyumlu taşımadan devralınır
- Görsel üretimi: sağlayıcı-plugin katmanında planlanıyor
- Görsel/video anlama: şimdi Standard uç noktasında paketlenmiş
- Konuşma/ses: sağlayıcı-plugin katmanında planlanıyor
- Bellek gömmeleri/yeniden sıralama: embedding adapter yüzeyi üzerinden planlanıyor
- Video üretimi: paylaşılan video-generation yeteneği üzerinden artık paketlenmiş

## Çok modlu eklentiler

`qwen` uzantısı artık ayrıca şunları da sunuyor:

- `qwen-vl-max-latest` üzerinden video anlama
- Wan video üretimi:
  - `wan2.6-t2v` (varsayılan)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

Bu çok modlu yüzeyler, Coding Plan uç noktalarını değil,
**Standard** DashScope uç noktalarını kullanır.

- Global/Intl Standard base URL: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- China Standard base URL: `https://dashscope.aliyuncs.com/compatible-mode/v1`

Video üretimi için OpenClaw, işi göndermeden önce yapılandırılmış Qwen bölgesini eşleşen
DashScope AIGC host'una eşler:

- Global/Intl: `https://dashscope-intl.aliyuncs.com`
- China: `https://dashscope.aliyuncs.com`

Bu, Coding Plan veya Standard Qwen host'larından birine işaret eden normal bir `models.providers.qwen.baseUrl`
değerinin video üretimini yine de doğru
bölgesel DashScope video uç noktasında tuttuğu anlamına gelir.

Video üretimi için varsayılan modeli açıkça ayarlayın:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Geçerli paketlenmiş Qwen video üretim sınırları:

- İstek başına en fazla **1** çıktı videosu
- En fazla **1** girdi görseli
- En fazla **4** girdi videosu
- En fazla **10 saniye** süre
- `size`, `aspectRatio`, `resolution`, `audio` ve `watermark` desteklenir
- Referans görsel/video kipi şu anda **uzak http(s) URL'leri** gerektirir. DashScope video uç noktası bu referanslar için
  yüklenen yerel tamponları kabul etmediğinden, yerel dosya yolları en başta reddedilir.

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve failover davranışı için [Video Üretimi](/tr/tools/video-generation) sayfasına bakın.

## Ortam notu

Gateway bir daemon olarak çalışıyorsa (launchd/systemd), `QWEN_API_KEY` değerinin
o süreç için kullanılabilir olduğundan emin olun (örneğin `~/.openclaw/.env` içinde veya
`env.shellEnv` aracılığıyla).
