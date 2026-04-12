---
read_when:
    - OpenClaw'da OpenAI modellerini kullanmak istiyorsunuz
    - API anahtarları yerine Codex abonelik kimlik doğrulamasını kullanmak istiyorsunuz
    - Daha sıkı GPT-5 aracı yürütme davranışına ihtiyacınız var
summary: OpenClaw'da API anahtarları veya Codex aboneliği aracılığıyla OpenAI'ı kullanın
title: OpenAI
x-i18n:
    generated_at: "2026-04-12T00:18:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7aa06fba9ac901e663685a6b26443a2f6aeb6ec3589d939522dc87cbb43497b4
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI, GPT modelleri için geliştirici API'leri sunar. Codex, abonelik
erişimi için **ChatGPT oturum açmayı** veya kullanıma dayalı erişim için **API anahtarı**
ile oturum açmayı destekler. Codex cloud, ChatGPT oturum açmayı gerektirir.
OpenAI, OpenClaw gibi harici araçlarda/iş akışlarında abonelik OAuth kullanımını açıkça destekler.

## Varsayılan etkileşim stili

OpenClaw, hem `openai/*` hem de
`openai-codex/*` çalıştırmaları için OpenAI'ya özgü küçük bir istem kaplaması ekleyebilir. Varsayılan olarak,
bu kaplama asistanı sıcak,
iş birliğine açık, kısa, doğrudan ve biraz daha duygusal olarak daha ifadeci tutar;
ancak temel OpenClaw sistem isteminin yerini almaz. Bu dostça kaplama ayrıca
uygun olduğunda ara sıra emoji kullanımına izin verirken genel
çıktıyı kısa tutar.

Yapılandırma anahtarı:

`plugins.entries.openai.config.personality`

İzin verilen değerler:

- `"friendly"`: varsayılan; OpenAI'ya özgü kaplamayı etkinleştirir.
- `"on"`: `"friendly"` için takma ad.
- `"off"`: kaplamayı devre dışı bırakır ve yalnızca temel OpenClaw istemini kullanır.

Kapsam:

- `openai/*` modellerine uygulanır.
- `openai-codex/*` modellerine uygulanır.
- Diğer sağlayıcıları etkilemez.

Bu davranış varsayılan olarak açıktır. Bunun gelecekteki yerel yapılandırma değişikliklerinden
etkilenmeden kalmasını istiyorsanız `"friendly"` değerini açıkça koruyun:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### OpenAI istem kaplamasını devre dışı bırakma

Değiştirilmemiş temel OpenClaw istemini istiyorsanız, kaplamayı `"off"` olarak ayarlayın:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Bunu yapılandırma CLI ile de doğrudan ayarlayabilirsiniz:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw bu ayarı çalışma zamanında büyük/küçük harf duyarsız şekilde normalleştirir; bu nedenle
`"Off"` gibi değerler de dostça kaplamayı devre dışı bırakır.

## Seçenek A: OpenAI API anahtarı (OpenAI Platform)

**En iyisi:** doğrudan API erişimi ve kullanıma dayalı faturalama.
API anahtarınızı OpenAI panosundan alın.

Rota özeti:

- `openai/gpt-5.4` = doğrudan OpenAI Platform API rotası
- `OPENAI_API_KEY` (veya eşdeğer OpenAI sağlayıcı yapılandırması) gerektirir
- OpenClaw'da ChatGPT/Codex oturum açma, `openai/*` yerine `openai-codex/*` üzerinden yönlendirilir

### CLI kurulumu

```bash
openclaw onboard --auth-choice openai-api-key
# veya etkileşimsiz
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Yapılandırma örneği

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

OpenAI'nin güncel API model belgelerinde, doğrudan
OpenAI API kullanımı için `gpt-5.4` ve `gpt-5.4-pro` listelenir. OpenClaw, her ikisini de
`openai/*` Responses yolu üzerinden iletir.
OpenClaw, eski `openai/gpt-5.3-codex-spark` satırını kasıtlı olarak gizler,
çünkü doğrudan OpenAI API çağrıları canlı trafikte bunu reddeder.

OpenClaw, doğrudan OpenAI
API yolunda `openai/gpt-5.3-codex-spark` sunmaz. `pi-ai` hâlâ bu model için
yerleşik bir satır sunar, ancak canlı OpenAI API
istekleri şu anda bunu reddeder. Spark, OpenClaw'da yalnızca Codex olarak değerlendirilir.

## Görsel oluşturma

Paketlenmiş `openai` eklentisi, paylaşılan
`image_generate` aracı aracılığıyla görsel oluşturmayı da kaydeder.

- Varsayılan görsel modeli: `openai/gpt-image-1`
- Oluşturma: istek başına en fazla 4 görsel
- Düzenleme modu: etkin, en fazla 5 referans görsel
- `size` desteklenir
- Mevcut OpenAI'ya özgü sınırlama: OpenClaw bugün `aspectRatio` veya
  `resolution` geçersiz kılmalarını OpenAI Images API'sine iletmez

OpenAI'ı varsayılan görsel sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve devretme davranışı için [Görsel Oluşturma](/tr/tools/image-generation) bölümüne bakın.

## Video oluşturma

Paketlenmiş `openai` eklentisi, paylaşılan
`video_generate` aracı aracılığıyla video oluşturmayı da kaydeder.

- Varsayılan video modeli: `openai/sora-2`
- Modlar: metinden videoya, görselden videoya ve tek video referansı/düzenleme akışları
- Mevcut sınırlar: 1 görsel veya 1 video referans girdisi
- Mevcut OpenAI'ya özgü sınırlama: OpenClaw şu anda yerel OpenAI video oluşturma için yalnızca `size`
  geçersiz kılmalarını iletir. `aspectRatio`, `resolution`, `audio` ve `watermark` gibi
  desteklenmeyen isteğe bağlı geçersiz kılmalar yok sayılır
  ve araç uyarısı olarak geri bildirilir.

OpenAI'ı varsayılan video sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Paylaşılan araç
parametreleri, sağlayıcı seçimi ve devretme davranışı için [Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.

## Seçenek B: OpenAI Code (Codex) aboneliği

**En iyisi:** API anahtarı yerine ChatGPT/Codex abonelik erişimini kullanmak.
Codex cloud, ChatGPT oturum açmayı gerektirirken Codex CLI, ChatGPT veya API anahtarıyla oturum açmayı destekler.

Rota özeti:

- `openai-codex/gpt-5.4` = ChatGPT/Codex OAuth rotası
- Doğrudan OpenAI Platform API anahtarı değil, ChatGPT/Codex oturum açmayı kullanır
- `openai-codex/*` için sağlayıcı tarafı sınırlar, ChatGPT web/uygulama deneyiminden farklı olabilir

### CLI kurulumu (Codex OAuth)

```bash
# Sihirbazda Codex OAuth çalıştırın
openclaw onboard --auth-choice openai-codex

# Veya OAuth'u doğrudan çalıştırın
openclaw models auth login --provider openai-codex
```

### Yapılandırma örneği (Codex aboneliği)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

OpenAI'nin güncel Codex belgelerinde `gpt-5.4`, mevcut Codex modeli olarak listelenir. OpenClaw
bunu ChatGPT/Codex OAuth kullanımı için `openai-codex/gpt-5.4` ile eşler.

Bu rota kasıtlı olarak `openai/gpt-5.4` rotasından ayrıdır. Doğrudan
OpenAI Platform API yolunu istiyorsanız, API anahtarıyla `openai/*` kullanın. ChatGPT/Codex
oturum açmayı istiyorsanız `openai-codex/*` kullanın.

Onboarding mevcut bir Codex CLI oturum açmasını yeniden kullanırsa, bu kimlik bilgileri
Codex CLI tarafından yönetilmeye devam eder. Süresi dolduğunda, OpenClaw önce harici Codex kaynağını yeniden okur
ve sağlayıcı bunu yenileyebiliyorsa, ayrı bir yalnızca-OpenClaw kopyasının sahipliğini almak yerine
yenilenmiş kimlik bilgisini tekrar Codex depolamasına yazar.

Codex hesabınızın Codex Spark yetkisi varsa, OpenClaw ayrıca şunu da destekler:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw, Codex Spark'ı yalnızca Codex olarak değerlendirir. Doğrudan
`openai/gpt-5.3-codex-spark` API anahtarı yolunu sunmaz.

OpenClaw ayrıca `pi-ai`
bunu keşfettiğinde `openai-codex/gpt-5.3-codex-spark` değerini korur. Bunu yetkiye bağlı ve deneysel olarak değerlendirin: Codex Spark,
GPT-5.4 `/fast`'tan ayrıdır ve kullanılabilirlik, oturum açılmış Codex /
ChatGPT hesabına bağlıdır.

### Codex bağlam penceresi sınırı

OpenClaw, Codex model meta verilerini ve çalışma zamanı bağlam sınırını ayrı
değerler olarak ele alır.

`openai-codex/gpt-5.4` için:

- yerel `contextWindow`: `1050000`
- varsayılan çalışma zamanı `contextTokens` sınırı: `272000`

Bu, model meta verilerini doğru tutarken pratikte daha iyi gecikme ve kalite
özelliklerine sahip olan daha küçük varsayılan çalışma zamanı penceresini korur.

Farklı bir etkin sınır istiyorsanız `models.providers.<provider>.models[].contextTokens` ayarlayın:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Yalnızca yerel model
meta verilerini tanımlarken veya geçersiz kılarken `contextWindow` kullanın. Çalışma zamanı bağlam bütçesini sınırlamak istediğinizde `contextTokens` kullanın.

### Aktarım varsayılanı

OpenClaw, model akışı için `pi-ai` kullanır. Hem `openai/*` hem de
`openai-codex/*` için varsayılan aktarım `"auto"` şeklindedir (önce WebSocket, ardından SSE
yedeklemesi).

`"auto"` modunda OpenClaw, SSE'ye dönmeden önce
erken ve yeniden denenebilir bir WebSocket hatasını da bir kez yeniden dener. Zorunlu `"websocket"` modu ise aktarım
hatalarını yedekleme arkasına gizlemek yerine doğrudan gösterir.

`"auto"` modunda bağlantı veya erken dönüş WebSocket hatasından sonra OpenClaw,
o oturumun WebSocket yolunu yaklaşık 60 saniye boyunca bozulmuş olarak işaretler ve
aktarımlar arasında gidip gelmek yerine sonraki dönüşleri
soğuma süresi boyunca SSE üzerinden gönderir.

Yerel OpenAI ailesi uç noktaları için (`openai/*`, `openai-codex/*` ve Azure
OpenAI Responses), OpenClaw ayrıca isteklerle birlikte kararlı oturum ve dönüş kimliği durumunu da ekler;
böylece yeniden denemeler, yeniden bağlanmalar ve SSE yedeklemesi aynı
konuşma kimliğiyle hizalı kalır. Yerel OpenAI ailesi rotalarında buna kararlı
oturum/dönüş istek kimliği üst bilgileri ve eşleşen aktarım meta verileri dahildir.

OpenClaw ayrıca OpenAI kullanım sayaçlarını oturum/durum yüzeylerine ulaşmadan önce
aktarım varyantları arasında normalleştirir. Yerel OpenAI/Codex Responses trafiği
kullanımı `input_tokens` / `output_tokens` veya
`prompt_tokens` / `completion_tokens` olarak bildirebilir; OpenClaw bunları `/status`, `/usage` ve oturum günlükleri için aynı girdi
ve çıktı sayaçları olarak değerlendirir. Yerel
WebSocket trafiği `total_tokens` değerini atladığında (veya `0` bildirdiğinde), OpenClaw
oturum/durum görüntülerinin dolu kalması için normalleştirilmiş girdi + çıktı toplamına geri döner.

`agents.defaults.models.<provider/model>.params.transport` ayarlayabilirsiniz:

- `"sse"`: SSE'yi zorla
- `"websocket"`: WebSocket'i zorla
- `"auto"`: WebSocket'i dene, ardından SSE'ye geri dön

`openai/*` için (Responses API), OpenClaw ayrıca WebSocket aktarımı kullanıldığında
varsayılan olarak WebSocket ısınmasını etkinleştirir (`openaiWsWarmup: true`).

İlgili OpenAI belgeleri:

- [WebSocket ile Realtime API](https://platform.openai.com/docs/guides/realtime-websocket)
- [Akış API yanıtları (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### OpenAI WebSocket ısınması

OpenAI belgeleri, ısınmayı isteğe bağlı olarak tanımlar. OpenClaw,
WebSocket aktarımı kullanılırken ilk dönüş gecikmesini azaltmak için
`openai/*` için bunu varsayılan olarak etkinleştirir.

### Isınmayı devre dışı bırak

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Isınmayı açıkça etkinleştir

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### OpenAI ve Codex öncelikli işleme

OpenAI'nin API'si, `service_tier=priority` üzerinden öncelikli işlemeyi sunar. OpenClaw'da,
yerel OpenAI/Codex Responses uç noktalarında bu alanı iletmek için `agents.defaults.models["<provider>/<model>"].params.serviceTier`
değerini ayarlayın.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Desteklenen değerler `auto`, `default`, `flex` ve `priority` şeklindedir.

OpenClaw, `params.serviceTier` değerini hem doğrudan `openai/*` Responses
isteklerine hem de `openai-codex/*` Codex Responses isteklerine, bu modeller
yerel OpenAI/Codex uç noktalarını işaret ettiğinde iletir.

Önemli davranış:

- doğrudan `openai/*`, `api.openai.com` hedeflemelidir
- `openai-codex/*`, `chatgpt.com/backend-api` hedeflemelidir
- bu sağlayıcılardan herhangi birini başka bir temel URL veya proxy üzerinden yönlendirirseniz, OpenClaw `service_tier` değerine dokunmaz

### OpenAI hızlı modu

OpenClaw, hem `openai/*` hem de
`openai-codex/*` oturumları için paylaşılan bir hızlı mod geçişi sunar:

- Sohbet/UI: `/fast status|on|off`
- Yapılandırma: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Hızlı mod etkin olduğunda, OpenClaw bunu OpenAI öncelikli işlemeye eşler:

- `api.openai.com` adresine yapılan doğrudan `openai/*` Responses çağrıları `service_tier = "priority"` gönderir
- `chatgpt.com/backend-api` adresine yapılan `openai-codex/*` Responses çağrıları da `service_tier = "priority"` gönderir
- mevcut yük `service_tier` değerleri korunur
- hızlı mod `reasoning` veya `text.verbosity` değerlerini yeniden yazmaz

GPT 5.4 özelinde en yaygın kurulum şudur:

- `openai/gpt-5.4` veya `openai-codex/gpt-5.4` kullanan bir oturumda `/fast on` gönderin
- veya `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true` ayarlayın
- Codex OAuth da kullanıyorsanız `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true` değerini de ayarlayın

Örnek:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Oturum geçersiz kılmaları yapılandırmaya göre önceliklidir. Oturumlar UI'da oturum geçersiz kılmasını temizlemek,
oturumu yapılandırılmış varsayılana döndürür.

### Yerel OpenAI ile OpenAI uyumlu rotalar karşılaştırması

OpenClaw, doğrudan OpenAI, Codex ve Azure OpenAI uç noktalarını,
genel OpenAI uyumlu `/v1` proxy'lerinden farklı şekilde ele alır:

- yerel `openai/*`, `openai-codex/*` ve Azure OpenAI rotaları,
  muhakemeyi açıkça devre dışı bıraktığınızda `reasoning: { effort: "none" }` değerini
  olduğu gibi korur
- yerel OpenAI ailesi rotalarında araç şemaları varsayılan olarak strict modda olur
- gizli OpenClaw atıf üst bilgileri (`originator`, `version` ve
  `User-Agent`) yalnızca doğrulanmış yerel OpenAI ana bilgisayarlarında
  (`api.openai.com`) ve yerel Codex ana bilgisayarlarında (`chatgpt.com/backend-api`) eklenir
- yerel OpenAI/Codex rotaları,
  `service_tier`, Responses `store`, OpenAI reasoning uyumluluk yükleri ve
  istem önbelleği ipuçları gibi yalnızca OpenAI'ya özgü istek şekillendirmelerini korur
- proxy tarzı OpenAI uyumlu rotalar daha gevşek uyumluluk davranışını korur ve
  strict araç şemalarını, yalnızca yerel ortama özgü istek şekillendirmelerini veya gizli
  OpenAI/Codex atıf üst bilgilerini zorlamaz

Azure OpenAI, aktarım ve uyumluluk
davranışı açısından yerel yönlendirme grubunda kalır, ancak gizli OpenAI/Codex atıf üst bilgilerini almaz.

Bu, üçüncü taraf `/v1` arka uçlarına eski
OpenAI uyumlu katmanları zorlamadan mevcut yerel OpenAI Responses davranışını korur.

### Strict-agentic GPT modu

`openai/*` ve `openai-codex/*` GPT-5 ailesi çalıştırmaları için OpenClaw,
daha sıkı bir gömülü Pi yürütme sözleşmesi kullanabilir:

```json5
{
  agents: {
    defaults: {
      embeddedPi: {
        executionContract: "strict-agentic",
      },
    },
  },
}
```

`strict-agentic` ile OpenClaw, somut bir araç eylemi mevcut olduğunda artık
yalnızca plan içeren bir asistan dönüşünü başarılı ilerleme olarak değerlendirmez. Dönüşü
hemen eyleme geçme yönlendirmesiyle yeniden dener, önemli işler için yapılandırılmış `update_plan` aracını otomatik olarak etkinleştirir
ve model eyleme geçmeden plan yapmaya devam ederse açık bir engellendi durumunu gösterir.

Bu mod, OpenAI ve OpenAI Codex GPT-5 ailesi çalıştırmalarıyla sınırlıdır. Diğer sağlayıcılar
ve daha eski model aileleri, siz onları başka çalışma zamanı ayarlarına dahil etmedikçe
varsayılan gömülü Pi davranışını korur.

### OpenAI Responses sunucu tarafı sıkıştırma

Doğrudan OpenAI Responses modelleri için (`api.openai.com` üzerinde `baseUrl`
ve `api: "openai-responses"` kullanan `openai/*`), OpenClaw artık OpenAI sunucu tarafı
sıkıştırma yükü ipuçlarını otomatik olarak etkinleştirir:

- `store: true` değerini zorlar (`supportsStore: false` ayarlayan model uyumluluğu yoksa)
- `context_management: [{ type: "compaction", compact_threshold: ... }]` ekler

Varsayılan olarak `compact_threshold`, model `contextWindow` değerinin `%70`'idir
(yoksa `80000` kullanılır).

### Sunucu tarafı sıkıştırmayı açıkça etkinleştirme

Uyumlu Responses modellerinde `context_management` eklemeyi zorlamak istediğinizde bunu kullanın
(örneğin Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Özel bir eşikle etkinleştirme

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Sunucu tarafı sıkıştırmayı devre dışı bırakma

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` yalnızca `context_management` eklemeyi kontrol eder.
Doğrudan OpenAI Responses modelleri, uyumluluk `supportsStore: false`
ayarlamadıkça yine de `store: true` değerini zorlar.

## Notlar

- Model başvuruları her zaman `provider/model` biçimini kullanır (bkz. [/concepts/models](/tr/concepts/models)).
- Kimlik doğrulama ayrıntıları ve yeniden kullanım kuralları [/concepts/oauth](/tr/concepts/oauth) bölümündedir.
