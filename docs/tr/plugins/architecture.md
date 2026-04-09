---
read_when:
    - Yerel OpenClaw plugin'leri oluşturuyor veya hata ayıklıyorsunuz
    - Plugin yetenek modelini veya sahiplik sınırlarını anlamak istiyorsunuz
    - Plugin yükleme işlem hattı veya kayıt sistemi üzerinde çalışıyorsunuz
    - Sağlayıcı çalışma zamanı kancalarını veya kanal plugin'lerini uyguluyorsunuz
sidebarTitle: Internals
summary: 'Plugin iç yapısı: yetenek modeli, sahiplik, sözleşmeler, yükleme işlem hattı ve çalışma zamanı yardımcıları'
title: Plugin İç Yapısı
x-i18n:
    generated_at: "2026-04-09T01:32:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2575791f835990589219bb06d8ca92e16a8c38b317f0bfe50b421682f253ef18
    source_path: plugins/architecture.md
    workflow: 15
---

# Plugin İç Yapısı

<Info>
  Bu, **derin mimari başvurusudur**. Pratik kılavuzlar için şunlara bakın:
  - [Plugin yükleme ve kullanma](/tr/tools/plugin) — kullanıcı kılavuzu
  - [Başlangıç](/tr/plugins/building-plugins) — ilk plugin öğreticisi
  - [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins) — bir mesajlaşma kanalı oluşturun
  - [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) — bir model sağlayıcısı oluşturun
  - [SDK Genel Bakış](/tr/plugins/sdk-overview) — içe aktarma haritası ve kayıt API'si
</Info>

Bu sayfa, OpenClaw plugin sisteminin iç mimarisini kapsar.

## Genel yetenek modeli

Yetenekler, OpenClaw içindeki genel **yerel plugin** modelidir. Her
yerel OpenClaw plugin'i bir veya daha fazla yetenek türüne kayıt olur:

| Yetenek               | Kayıt yöntemi                                   | Örnek plugin'ler                    |
| --------------------- | ------------------------------------------------ | ----------------------------------- |
| Metin çıkarımı        | `api.registerProvider(...)`                      | `openai`, `anthropic`               |
| CLI çıkarım arka ucu  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`               |
| Konuşma               | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`           |
| Gerçek zamanlı yazıya dökme | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                      |
| Gerçek zamanlı ses    | `api.registerRealtimeVoiceProvider(...)`         | `openai`                            |
| Medya anlama          | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                  |
| Görsel oluşturma      | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Müzik oluşturma       | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                 |
| Video oluşturma       | `api.registerVideoGenerationProvider(...)`       | `qwen`                              |
| Web getirme           | `api.registerWebFetchProvider(...)`              | `firecrawl`                         |
| Web arama             | `api.registerWebSearchProvider(...)`             | `google`                            |
| Kanal / mesajlaşma    | `api.registerChannel(...)`                       | `msteams`, `matrix`                 |

Sıfır yetenek kaydeden ancak kancalar, araçlar veya
hizmetler sağlayan bir plugin, **eski yalnızca kanca** plugin'idir. Bu desen hâlâ tamamen desteklenir.

### Dış uyumluluk duruşu

Yetenek modeli çekirdeğe yerleşti ve bugün paketlenmiş/yerel plugin'ler
tarafından kullanılıyor, ancak dış plugin uyumluluğu için hâlâ
"aktarılıyor, dolayısıyla sabitlenmiştir" ifadesinden daha sıkı bir eşik gerekir.

Güncel rehberlik:

- **mevcut dış plugin'ler:** kanca tabanlı entegrasyonları çalışır durumda tutun; bunu
  uyumluluk temeli olarak değerlendirin
- **yeni paketlenmiş/yerel plugin'ler:** satıcıya özgü iç erişimler veya yeni yalnızca kanca tasarımları yerine
  açık yetenek kaydını tercih edin
- **yetenek kaydını benimseyen dış plugin'ler:** izin verilir, ancak belgelerde açıkça kararlı bir sözleşme olarak işaretlenmedikçe
  yeteneğe özgü yardımcı yüzeyleri gelişmekte olan yüzeyler olarak değerlendirin

Pratik kural:

- yetenek kayıt API'leri amaçlanan yöndür
- geçiş sırasında eski kancalar dış plugin'ler için en güvenli, kırılmasız yol olmaya devam eder
- dışa aktarılan yardımcı alt yolların hepsi eşdeğer değildir; rastlantısal yardımcı aktarımlarını değil,
  belgelenmiş dar sözleşmeyi tercih edin

### Plugin biçimleri

OpenClaw, yüklenen her plugin'i gerçek kayıt
davranışına göre bir biçime sınıflandırır (yalnızca statik meta veriye göre değil):

- **plain-capability** -- tam olarak bir yetenek türü kaydeder (örneğin
  `mistral` gibi yalnızca sağlayıcı olan bir plugin)
- **hybrid-capability** -- birden fazla yetenek türü kaydeder (örneğin
  `openai`, metin çıkarımı, konuşma, medya anlama ve görsel
  oluşturma sahipliğine sahiptir)
- **hook-only** -- yalnızca kanca kaydeder (yazımlı veya özel), yetenek,
  araç, komut veya hizmet kaydetmez
- **non-capability** -- yetenek olmadan araç, komut, hizmet veya rota kaydeder

Bir plugin'in biçimini ve yetenek
dökümünü görmek için `openclaw plugins inspect <id>` kullanın. Ayrıntılar için [CLI başvurusu](/cli/plugins#inspect) sayfasına bakın.

### Eski kancalar

`before_agent_start` kancası, yalnızca kanca kullanan plugin'ler için
bir uyumluluk yolu olarak desteklenmeye devam eder. Eski gerçek dünya plugin'leri hâlâ buna bağımlıdır.

Yön:

- çalışır durumda tutun
- bunu eski olarak belgelendirin
- model/sağlayıcı geçersiz kılma işleri için `before_model_resolve` tercih edin
- istem değiştirme işleri için `before_prompt_build` tercih edin
- yalnızca gerçek kullanım düştüğünde ve fikstür kapsamı geçiş güvenliğini kanıtladığında kaldırın

### Uyumluluk sinyalleri

`openclaw doctor` veya `openclaw plugins inspect <id>` çalıştırdığınızda,
şu etiketlerden birini görebilirsiniz:

| Sinyal                     | Anlamı                                                     |
| -------------------------- | ---------------------------------------------------------- |
| **config valid**           | Yapılandırma sorunsuz ayrıştırılır ve plugin'ler çözülür   |
| **compatibility advisory** | Plugin desteklenen ama daha eski bir desen kullanıyor (ör. `hook-only`) |
| **legacy warning**         | Plugin, kullanımdan kaldırılmış olan `before_agent_start` kullanıyor |
| **hard error**             | Yapılandırma geçersiz veya plugin yüklenemedi              |

Ne `hook-only` ne de `before_agent_start` plugin'inizi bugün bozmaz --
`hook-only` bir tavsiye niteliğindedir ve `before_agent_start` yalnızca bir uyarı tetikler. Bu
sinyaller ayrıca `openclaw status --all` ve `openclaw plugins doctor` içinde de görünür.

## Mimariye genel bakış

OpenClaw'un plugin sistemi dört katmandan oluşur:

1. **Manifest + keşif**
   OpenClaw, yapılandırılmış yollardan, çalışma alanı köklerinden,
   genel uzantı köklerinden ve paketlenmiş uzantılardan aday plugin'leri bulur.
   Keşif önce yerel `openclaw.plugin.json` manifestlerini ve desteklenen paket manifestlerini okur.
2. **Etkinleştirme + doğrulama**
   Çekirdek, keşfedilen bir plugin'in etkin, devre dışı, engellenmiş veya
   bellek gibi özel bir yuva için seçilmiş olup olmadığına karar verir.
3. **Çalışma zamanı yükleme**
   Yerel OpenClaw plugin'leri jiti aracılığıyla işlem içi yüklenir ve
   yetenekleri merkezi bir kayıt sistemine kaydeder. Uyumlu paketler, çalışma zamanı kodu içe aktarılmadan
   kayıt kayıtlarına normalize edilir.
4. **Yüzey tüketimi**
   OpenClaw'un geri kalanı araçları, kanalları, sağlayıcı
   kurulumunu, kancaları, HTTP rotalarını, CLI komutlarını ve hizmetleri açığa çıkarmak için kayıt sistemini okur.

Özellikle plugin CLI için, kök komut keşfi iki aşamaya bölünür:

- ayrıştırma zamanı meta verisi `registerCli(..., { descriptors: [...] })` içinden gelir
- gerçek plugin CLI modülü tembel kalabilir ve ilk çağrıda kaydolabilir

Bu, plugin'e ait CLI kodunu plugin içinde tutarken OpenClaw'un
ayrıştırmadan önce kök komut adlarını ayırmasına olanak verir.

Önemli tasarım sınırı:

- keşif + yapılandırma doğrulaması, plugin kodu yürütülmeden
  **manifest/şema meta verisinden** çalışabilmelidir
- yerel çalışma zamanı davranışı, plugin modülünün `register(api)` yolundan gelir

Bu ayrım, tam çalışma zamanı etkin olmadan önce OpenClaw'un yapılandırmayı doğrulamasına,
eksik/devre dışı plugin'leri açıklamasına ve UI/şema ipuçları oluşturmasına olanak tanır.

### Kanal plugin'leri ve paylaşılan message aracı

Kanal plugin'lerinin normal sohbet eylemleri için ayrı bir gönder/düzenle/tepki verme aracı kaydetmesi gerekmez.
OpenClaw çekirdekte tek bir paylaşılan `message` aracını tutar ve
kanala özgü keşif ile yürütme bunun arkasında kanal plugin'lerine aittir.

Mevcut sınır şöyledir:

- çekirdek, paylaşılan `message` aracı ana makinesine, istem bağlamasına, oturum/iş parçacığı
  muhasebesine ve yürütme sevkine sahiptir
- kanal plugin'leri kapsamlı eylem keşfine, yetenek keşfine ve
  kanala özgü şema parçalarına sahiptir
- kanal plugin'leri, konuşma kimliklerinin iş parçacığı kimliklerini nasıl kodladığı veya üst konuşmalardan
  nasıl miras aldığı gibi, sağlayıcıya özgü oturum konuşma dil bilgisine sahiptir
- kanal plugin'leri son eylemi kendi eylem bağdaştırıcıları üzerinden yürütür

Kanal plugin'leri için SDK yüzeyi
`ChannelMessageActionAdapter.describeMessageTool(...)` şeklindedir. Bu birleşik keşif
çağrısı, bir plugin'in görünür eylemlerini, yeteneklerini ve şema
katkılarını birlikte döndürmesine olanak verir; böylece bu parçalar birbirinden ayrışmaz.

Çekirdek, bu keşif adımına çalışma zamanı kapsamını geçirir. Önemli alanlar şunlardır:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- güvenilir gelen `requesterSenderId`

Bu, bağlama duyarlı plugin'ler için önemlidir. Bir kanal; etkin hesaba,
geçerli odaya/iş parçacığına/mesaja veya güvenilir istek sahibi kimliğine göre
çekirdekte kanala özgü dallar sabitlemeden mesaj eylemlerini gizleyebilir veya açığa çıkarabilir.

Bu nedenle gömülü çalıştırıcı yönlendirme değişiklikleri hâlâ plugin işidir: çalıştırıcı,
paylaşılan `message` aracının mevcut tur için doğru kanala ait
yüzeyi açığa çıkarması amacıyla geçerli sohbet/oturum kimliğini plugin
keşif sınırına iletmekten sorumludur.

Kanala ait yürütme yardımcıları için paketlenmiş plugin'ler, yürütme
çalışma zamanını kendi uzantı modülleri içinde tutmalıdır. Çekirdek artık
`src/agents/tools` altında Discord, Slack, Telegram veya WhatsApp mesaj-eğlemi çalışma zamanlarına sahip değildir.
Ayrı `plugin-sdk/*-action-runtime` alt yolları yayımlamıyoruz ve paketlenmiş
plugin'ler kendi yerel çalışma zamanı kodlarını doğrudan kendi
uzantı modüllerinden içe aktarmalıdır.

Aynı sınır genel olarak sağlayıcı adlı SDK geçişleri için de geçerlidir: çekirdek,
Slack, Discord, Signal,
WhatsApp veya benzeri uzantılar için kanala özgü kolaylık varillerini içe aktarmamalıdır. Çekirdeğin bir davranışa ihtiyacı varsa ya
paketlenmiş plugin'in kendi `api.ts` / `runtime-api.ts` varilini tüketmeli ya da bu ihtiyacı
paylaşılan SDK içinde dar ve genel bir yeteneğe yükseltmelidir.

Özellikle anketler için iki yürütme yolu vardır:

- `outbound.sendPoll`, ortak
  anket modeline uyan kanallar için paylaşılan temel yoldur
- `actions.handleAction("poll")`, kanala özgü
  anket anlamları veya ek anket parametreleri için tercih edilen yoldur

Çekirdek artık paylaşılan anket ayrıştırmasını, plugin anket sevki eylemi reddettikten
sonraya erteler; böylece plugin'e ait anket işleyicileri, önce genel anket ayrıştırıcısı
tarafından engellenmeden kanala özgü anket alanlarını kabul edebilir.

Tam başlangıç sırası için [Yükleme işlem hattı](#load-pipeline) bölümüne bakın.

## Yetenek sahipliği modeli

OpenClaw, yerel bir plugin'i ilgisiz entegrasyonlardan oluşan bir torba olarak değil,
bir **şirketin** veya bir **özelliğin** sahiplik sınırı olarak ele alır.

Bu şu anlama gelir:

- bir şirket plugin'i genellikle o şirketin OpenClaw'a dönük tüm
  yüzeylerine sahip olmalıdır
- bir özellik plugin'i genellikle tanıttığı özelliğin tam yüzeyine sahip olmalıdır
- kanallar, sağlayıcı davranışını geçici olarak yeniden uygulamak yerine paylaşılan çekirdek yetenekleri tüketmelidir

Örnekler:

- paketlenmiş `openai` plugin'i OpenAI model-sağlayıcı davranışına ve OpenAI
  konuşma + gerçek zamanlı ses + medya anlama + görsel oluşturma davranışına sahiptir
- paketlenmiş `elevenlabs` plugin'i ElevenLabs konuşma davranışına sahiptir
- paketlenmiş `microsoft` plugin'i Microsoft konuşma davranışına sahiptir
- paketlenmiş `google` plugin'i Google model-sağlayıcı davranışına artı Google
  medya anlama + görsel oluşturma + web arama davranışına sahiptir
- paketlenmiş `firecrawl` plugin'i Firecrawl web-getirme davranışına sahiptir
- paketlenmiş `minimax`, `mistral`, `moonshot` ve `zai` plugin'leri
  medya anlama arka uçlarına sahiptir
- `voice-call` plugin'i bir özellik plugin'idir: çağrı aktarımı, araçlar,
  CLI, rotalar ve Twilio medya akışı köprülemesine sahiptir, ancak satıcı plugin'lerini doğrudan
  içe aktarmak yerine paylaşılan konuşma ile gerçek zamanlı yazıya dökme ve gerçek zamanlı ses yeteneklerini tüketir

Amaçlanan son durum şudur:

- metin modelleri, konuşma, görseller ve
  gelecekteki video gibi alanları kapsasa bile OpenAI tek bir plugin'de yaşar
- başka bir satıcı da kendi yüzey alanı için aynı şeyi yapabilir
- kanallar, sağlayıcının hangi satıcı plugin'ine ait olduğunu önemsemez; çekirdek tarafından açığa çıkarılan
  paylaşılan yetenek sözleşmesini tüketir

Temel ayrım şudur:

- **plugin** = sahiplik sınırı
- **capability** = birden çok plugin'in uygulayabileceği veya tüketebileceği çekirdek sözleşme

Dolayısıyla OpenClaw video gibi yeni bir alan eklerse ilk soru
"hangi sağlayıcı video işlemeyi sabit kodlamalı?" değildir. İlk soru
"çekirdek video yetenek sözleşmesi nedir?" olmalıdır. Bu sözleşme var olduğunda,
satıcı plugin'leri buna kayıt olabilir ve kanal/özellik plugin'leri bunu tüketebilir.

Yetenek henüz yoksa, genellikle doğru adım şudur:

1. eksik yeteneği çekirdekte tanımlayın
2. bunu plugin API'si/çalışma zamanı üzerinden yazımlı biçimde açığa çıkarın
3. kanalları/özellikleri bu yeteneğe bağlayın
4. satıcı plugin'lerinin uygulamaları kaydetmesine izin verin

Bu, sahipliği açık tutarken tek bir satıcıya veya
tek seferlik plugin'e özgü bir kod yoluna bağımlı çekirdek davranışını önler.

### Yetenek katmanları

Kodun nereye ait olduğuna karar verirken şu zihinsel modeli kullanın:

- **çekirdek yetenek katmanı**: paylaşılan orkestrasyon, politika, geri dönüş,
  yapılandırma birleştirme kuralları, teslimat anlamları ve yazımlı sözleşmeler
- **satıcı plugin katmanı**: satıcıya özgü API'ler, kimlik doğrulama, model katalogları, konuşma
  sentezi, görsel oluşturma, gelecekteki video arka uçları, kullanım uç noktaları
- **kanal/özellik plugin katmanı**: paylaşılan çekirdek yetenekleri tüketen
  ve bunları bir yüzeyde sunan Slack/Discord/voice-call/vb. entegrasyonlar

Örneğin TTS şu yapıyı izler:

- çekirdek, yanıt zamanı TTS politikasına, geri dönüş sırasına, tercihlere ve kanal teslimatına sahiptir
- `openai`, `elevenlabs` ve `microsoft` sentez uygulamalarına sahiptir
- `voice-call` telefon TTS çalışma zamanı yardımcısını tüketir

Aynı desen gelecekteki yetenekler için de tercih edilmelidir.

### Çok yetenekli şirket plugin'i örneği

Bir şirket plugin'i dışarıdan bakıldığında tutarlı hissettirmelidir. OpenClaw;
modeller, konuşma, gerçek zamanlı yazıya dökme, gerçek zamanlı ses, medya
anlama, görsel oluşturma, video oluşturma, web getirme ve web arama için paylaşılan
sözleşmelere sahipse, bir satıcı tüm yüzeylerine tek bir yerde sahip olabilir:

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

Önemli olan yardımcı adlarının tam olarak ne olduğu değildir. Önemli olan yapıdır:

- tek bir plugin satıcı yüzeyine sahiptir
- çekirdek yine de yetenek sözleşmelerine sahiptir
- kanallar ve özellik plugin'leri satıcı kodunu değil, `api.runtime.*` yardımcılarını tüketir
- sözleşme testleri, plugin'in sahip olduğunu iddia ettiği yetenekleri kaydettiğini doğrulayabilir

### Yetenek örneği: video anlama

OpenClaw, görsel/ses/video anlamayı zaten tek bir paylaşılan
yetenek olarak ele alır. Aynı sahiplik modeli burada da geçerlidir:

1. çekirdek medya-anlama sözleşmesini tanımlar
2. satıcı plugin'leri uygun olduğunda `describeImage`, `transcribeAudio` ve
   `describeVideo` kaydı yapar
3. kanal ve özellik plugin'leri satıcı koduna doğrudan bağlanmak yerine paylaşılan çekirdek davranışını tüketir

Bu, tek bir sağlayıcının video varsayımlarının çekirdeğe gömülmesini önler. Plugin satıcı
yüzeyine sahiptir; çekirdek yetenek sözleşmesine ve geri dönüş davranışına sahiptir.

Video oluşturma zaten aynı sırayı kullanır: çekirdek yazımlı
yetenek sözleşmesine ve çalışma zamanı yardımcısına sahiptir, satıcı plugin'leri ise
`api.registerVideoGenerationProvider(...)` uygulamaları kaydeder.

Somut bir dağıtım kontrol listesi mi gerekiyor? Bkz.
[Capability Cookbook](/tr/plugins/architecture).

## Sözleşmeler ve zorunlu uygulama

Plugin API yüzeyi bilerek yazımlı ve
`OpenClawPluginApi` içinde merkezidir. Bu sözleşme desteklenen kayıt noktalarını ve
bir plugin'in güvenebileceği çalışma zamanı yardımcılarını tanımlar.

Bunun neden önemli olduğu:

- plugin yazarları tek bir kararlı iç standart elde eder
- çekirdek, aynı sağlayıcı kimliğini kaydeden iki plugin gibi yinelenen sahipliği reddedebilir
- başlangıç, bozuk kayıtlar için uygulanabilir tanılamaları yüzeye çıkarabilir
- sözleşme testleri paketlenmiş plugin sahipliğini zorunlu kılabilir ve sessiz sürüklenmeyi önleyebilir

İki katmanlı bir zorunlu uygulama vardır:

1. **çalışma zamanı kayıt zorlaması**
   Plugin kayıt sistemi, plugin'ler yüklenirken kayıtları doğrular. Örnekler:
   yinelenen sağlayıcı kimlikleri, yinelenen konuşma sağlayıcı kimlikleri ve bozuk
   kayıtlar, tanımsız davranış yerine plugin tanılamaları üretir.
2. **sözleşme testleri**
   Paketlenmiş plugin'ler, test çalıştırmaları sırasında sözleşme kayıt sistemlerinde yakalanır; böylece
   OpenClaw sahipliği açıkça doğrulayabilir. Bugün bu, model
   sağlayıcıları, konuşma sağlayıcıları, web arama sağlayıcıları ve paketlenmiş kayıt
   sahipliği için kullanılır.

Pratik etki şudur: OpenClaw hangi plugin'in hangi
yüzeye sahip olduğunu en baştan bilir. Bu, çekirdek ve kanalların sorunsuz biçimde bileşim kurmasına olanak tanır çünkü
sahiplik örtük değil, bildirilmiş, yazımlı ve test edilebilirdir.

### Bir sözleşmede ne bulunmalı

İyi plugin sözleşmeleri:

- yazımlıdır
- küçüktür
- yeteneğe özgüdür
- çekirdeğe aittir
- birden çok plugin tarafından yeniden kullanılabilir
- satıcı bilgisi olmadan kanallar/özellikler tarafından tüketilebilir

Kötü plugin sözleşmeleri:

- çekirdekte gizlenmiş satıcıya özgü politika
- kayıt sistemini baypas eden tek seferlik plugin kaçış kapıları
- doğrudan satıcı uygulamasına erişen kanal kodu
- `OpenClawPluginApi` veya
  `api.runtime` parçası olmayan geçici çalışma zamanı nesneleri

Şüphe duyduğunuzda soyutlama düzeyini yükseltin: önce yeteneği tanımlayın, sonra
plugin'lerin buna bağlanmasına izin verin.

## Yürütme modeli

Yerel OpenClaw plugin'leri Gateway ile **işlem içinde** çalışır. Kum havuzuna
alınmazlar. Yüklenmiş bir yerel plugin, çekirdek kodla aynı süreç düzeyindeki güven sınırına sahiptir.

Sonuçlar:

- yerel bir plugin araçlar, ağ işleyicileri, kancalar ve hizmetler kaydedebilir
- yerel bir plugin hatası gateway'i çökertebilir veya dengesizleştirebilir
- kötü amaçlı bir yerel plugin, OpenClaw süreci içinde keyfi kod yürütmeye denktir

Uyumlu paketler varsayılan olarak daha güvenlidir çünkü OpenClaw şu anda onları
meta veri/içerik paketleri olarak ele alır. Mevcut sürümlerde bu çoğunlukla paketlenmiş
Skills anlamına gelir.

Paketlenmemiş plugin'ler için allowlist'ler ve açık kurulum/yükleme yolları kullanın. Çalışma alanı plugin'lerini
üretim varsayılanları olarak değil, geliştirme zamanı kodu olarak değerlendirin.

Paketlenmiş çalışma alanı paket adları için plugin kimliğini npm
adına bağlı tutun: varsayılan olarak `@openclaw/<id>` veya
paket bilerek daha dar bir plugin rolü açığa çıkarıyorsa
`-provider`, `-plugin`, `-speech`, `-sandbox` veya `-media-understanding` gibi onaylı yazımlı son ekler kullanın.

Önemli güven notu:

- `plugins.allow`, **kaynak kökenine** değil **plugin kimliklerine** güvenir.
- Paketlenmiş bir plugin ile aynı kimliğe sahip bir çalışma alanı plugin'i etkinleştirildiğinde/allowlist'e alındığında
  kasıtlı olarak paketlenmiş kopyayı gölgeler.
- Bu normaldir ve yerel geliştirme, yama testi ve hotfix'ler için kullanışlıdır.

## Dışa aktarma sınırı

OpenClaw uygulama kolaylıklarını değil, yetenekleri dışa aktarır.

Yetenek kaydını genel tutun. Sözleşme olmayan yardımcı dışa aktarmaları azaltın:

- paketlenmiş plugin'e özgü yardımcı alt yollar
- genel API olması amaçlanmayan çalışma zamanı tesisat alt yolları
- satıcıya özgü kolaylık yardımcıları
- uygulama ayrıntısı olan kurulum/katılım yardımcıları

Bazı paketlenmiş plugin yardımcı alt yolları uyumluluk ve paketlenmiş plugin bakımı için
oluşturulmuş SDK dışa aktarma haritasında hâlâ kalmaktadır. Güncel örnekler
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` ve birkaç `plugin-sdk/matrix*` geçişidir. Bunları yeni üçüncü taraf plugin'ler için
önerilen SDK deseni olarak değil, ayrılmış uygulama ayrıntısı dışa aktarmaları olarak değerlendirin.

## Yükleme işlem hattı

Başlangıçta OpenClaw kabaca şunları yapar:

1. aday plugin köklerini keşfeder
2. yerel veya uyumlu paket manifestlerini ve paket meta verilerini okur
3. güvenli olmayan adayları reddeder
4. plugin yapılandırmasını normalize eder (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. her aday için etkinleştirmeye karar verir
6. etkin yerel modülleri jiti aracılığıyla yükler
7. yerel `register(api)` (veya eski bir takma ad olan `activate(api)`) kancalarını çağırır ve kayıtları plugin kayıt sisteminde toplar
8. kayıt sistemini komutlara/çalışma zamanı yüzeylerine açığa çıkarır

<Note>
`activate`, `register` için eski bir takma addır — yükleyici mevcut olanı çözer (`def.register ?? def.activate`) ve aynı noktada çağırır. Tüm paketlenmiş plugin'ler `register` kullanır; yeni plugin'ler için `register` tercih edin.
</Note>

Güvenlik geçitleri çalışma zamanı yürütümünden **önce** gerçekleşir. Giriş plugin kökünün dışına çıktığında,
yol herkes tarafından yazılabilir olduğunda veya paketlenmemiş plugin'ler için
yol sahipliği şüpheli göründüğünde adaylar engellenir.

### Manifest-öncelikli davranış

Manifest denetim düzlemi için doğruluk kaynağıdır. OpenClaw bunu şunlar için kullanır:

- plugin'i tanımlamak
- bildirilen kanalları/Skills/yapılandırma şemasını veya paket yeteneklerini keşfetmek
- `plugins.entries.<id>.config` değerini doğrulamak
- Control UI etiketlerini/yer tutucularını zenginleştirmek
- kurulum/katalog meta verisini göstermek

Yerel plugin'ler için çalışma zamanı modülü veri düzlemi kısmıdır. Kancalar,
araçlar, komutlar veya sağlayıcı akışları gibi gerçek davranışları kaydeder.

### Yükleyicinin önbelleğe aldıkları

OpenClaw şu öğeler için kısa süreli işlem içi önbellekler tutar:

- keşif sonuçları
- manifest kayıt sistemi verisi
- yüklenmiş plugin kayıt sistemleri

Bu önbellekler ani başlangıçları ve tekrarlanan komut yükünü azaltır. Onları
kalıcılık değil, kısa ömürlü performans önbellekleri olarak düşünmek güvenlidir.

Performans notu:

- Bu önbellekleri devre dışı bırakmak için `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` veya
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` ayarlayın.
- Önbellek pencerelerini `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` ve
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS` ile ayarlayın.

## Kayıt sistemi modeli

Yüklenmiş plugin'ler rastgele çekirdek küresellerini doğrudan değiştirmez. Bunun yerine
merkezi bir plugin kayıt sistemine kayıt olurlar.

Kayıt sistemi şunları izler:

- plugin kayıtları (kimlik, kaynak, köken, durum, tanılamalar)
- araçlar
- eski kancalar ve yazımlı kancalar
- kanallar
- sağlayıcılar
- gateway RPC işleyicileri
- HTTP rotaları
- CLI kayıtçıları
- arka plan hizmetleri
- plugin'e ait komutlar

Ardından çekirdek özellikler plugin modülleriyle doğrudan konuşmak yerine bu kayıt sisteminden okur.
Bu, yüklemeyi tek yönlü tutar:

- plugin modülü -> kayıt sistemi kaydı
- çekirdek çalışma zamanı -> kayıt sistemi tüketimi

Bu ayrım bakım kolaylığı için önemlidir. Şu anlama gelir: çoğu çekirdek yüzeyin yalnızca
tek bir entegrasyon noktasına ihtiyacı vardır: "kayıt sistemini oku", yoksa "her plugin
modülünü özel durum olarak işle" değil.

## Konuşma bağlama geri çağrıları

Bir konuşmayı bağlayan plugin'ler, bir onay çözümlendiğinde tepki verebilir.

Bir bağlama isteği onaylandıktan veya reddedildikten sonra geri çağrı almak için
`api.onConversationBindingResolved(...)` kullanın:

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

Geri çağrı yük alanları:

- `status`: `"approved"` veya `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` veya `"deny"`
- `binding`: onaylanan istekler için çözülmüş bağ
- `request`: özgün istek özeti, ayırma ipucu, gönderen kimliği ve
  konuşma meta verisi

Bu geri çağrı yalnızca bildirim amaçlıdır. Kimin bir konuşmayı bağlamasına izin verildiğini değiştirmez
ve çekirdek onay işleme bittikten sonra çalışır.

## Sağlayıcı çalışma zamanı kancaları

Sağlayıcı plugin'lerinin artık iki katmanı vardır:

- manifest meta verisi: çalışma zamanı yüklemesinden
  önce ucuz sağlayıcı ortam kimlik doğrulama araması için `providerAuthEnvVars`, kimlik doğrulamayı paylaşan sağlayıcı varyantları için `providerAuthAliases`,
  çalışma zamanı yüklemesinden önce ucuz kanal ortamı/kurulum araması için `channelEnvVars`,
  ayrıca çalışma zamanı yüklemesinden önce ucuz katılım/kimlik doğrulama seçimi etiketleri ve
  CLI bayrağı meta verisi için `providerAuthChoices`
- yapılandırma zamanı kancaları: `catalog` / eski `discovery` ile `applyConfigDefaults`
- çalışma zamanı kancaları: `normalizeModelId`, `normalizeTransport`,
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

OpenClaw hâlâ genel ajan döngüsüne, failover'a, transcript işleme ve
araç politikasına sahiptir. Bu kancalar, sağlayıcıya özgü davranış için tümüyle özel bir çıkarım aktarımına
gerek kalmadan uzantı yüzeyidir.

Sağlayıcının, genel kimlik doğrulama/durum/model seçici yollarının
plugin çalışma zamanını yüklemeden görebileceği ortam tabanlı kimlik bilgileri varsa manifest `providerAuthEnvVars` kullanın.
Bir sağlayıcı kimliği başka bir sağlayıcı kimliğinin ortam değişkenlerini, kimlik doğrulama profillerini, yapılandırma tabanlı kimlik doğrulamayı ve
API anahtarı katılım seçimini yeniden kullanacaksa manifest `providerAuthAliases` kullanın. Katılım/kimlik doğrulama seçimi
CLI yüzeylerinin sağlayıcının seçim kimliğini, grup etiketlerini ve basit
tek bayraklı kimlik doğrulama bağlamasını sağlayıcı çalışma zamanını yüklemeden bilmesi gerektiğinde manifest `providerAuthChoices` kullanın.
Sağlayıcı çalışma zamanındaki `envVars` alanını ise katılım etiketleri veya OAuth
client-id/client-secret kurulum değişkenleri gibi operatöre dönük ipuçları için tutun.

Bir kanal, genel kabuk ortamı geri dönüşünün, yapılandırma/durum denetimlerinin veya kurulum istemlerinin
kanal çalışma zamanını yüklemeden görebileceği ortam güdümlü kimlik doğrulama ya da kurulum kullanıyorsa manifest `channelEnvVars` kullanın.

### Kanca sırası ve kullanım

Model/sağlayıcı plugin'leri için OpenClaw kancaları kabaca şu sırayla çağırır.
"Ne zaman kullanılmalı" sütunu hızlı karar rehberidir.

| #   | Kanca                             | Ne yapar                                                                                                        | Ne zaman kullanılmalı                                                                                                                       |
| --- | --------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | `models.json` üretimi sırasında sağlayıcı yapılandırmasını `models.providers` içine yayımlar                   | Sağlayıcı bir kataloğa veya temel URL varsayılanlarına sahipse                                                                              |
| 2   | `applyConfigDefaults`             | Yapılandırma somutlaştırması sırasında sağlayıcıya ait genel yapılandırma varsayılanlarını uygular            | Varsayılanlar kimlik doğrulama moduna, ortama veya sağlayıcı model ailesi anlamlarına bağlıysa                                             |
| --  | _(yerleşik model araması)_        | OpenClaw önce normal kayıt sistemi/katalog yolunu dener                                                        | _(plugin kancası değildir)_                                                                                                                 |
| 3   | `normalizeModelId`                | Aramadan önce eski veya önizleme model kimliği takma adlarını normalize eder                                   | Sağlayıcı, kanonik model çözümünden önce takma ad temizliğine sahipse                                                                       |
| 4   | `normalizeTransport`              | Genel model birleştirmesinden önce sağlayıcı ailesine ait `api` / `baseUrl` değerlerini normalize eder        | Sağlayıcı, aynı aktarım ailesindeki özel sağlayıcı kimlikleri için aktarım temizliğine sahipse                                             |
| 5   | `normalizeConfig`                 | Çalışma zamanı/sağlayıcı çözümünden önce `models.providers.<id>` değerini normalize eder                       | Sağlayıcının plugin ile yaşaması gereken yapılandırma temizliğine ihtiyacı varsa; paketlenmiş Google ailesi yardımcıları da desteklenen Google yapılandırma girdilerini geriden destekler |
| 6   | `applyNativeStreamingUsageCompat` | Yapılandırma sağlayıcılarına doğal akış kullanımı uyumluluk yeniden yazımlarını uygular                        | Sağlayıcının uç nokta güdümlü doğal akış kullanımı meta veri düzeltmelerine ihtiyacı varsa                                                 |
| 7   | `resolveConfigApiKey`             | Çalışma zamanı kimlik doğrulaması yüklenmeden önce yapılandırma sağlayıcıları için ortam işaretleyici kimlik doğrulamasını çözer | Sağlayıcının kendine ait ortam işaretleyici API anahtarı çözümlemesi varsa; `amazon-bedrock` burada ayrıca yerleşik bir AWS ortam işaretleyici çözümleyiciye sahiptir |
| 8   | `resolveSyntheticAuth`            | Düz metin kalıcı olmadan yerel/kendi kendine barındırılan veya yapılandırma tabanlı kimlik doğrulamayı yüzeye çıkarır | Sağlayıcı sentetik/yerel bir kimlik bilgisi işaretleyicisi ile çalışabiliyorsa                                                             |
| 9   | `resolveExternalAuthProfiles`     | Sağlayıcıya ait harici kimlik doğrulama profillerini üstüne bindirir; CLI/uygulama sahipli kimlik bilgileri için varsayılan `persistence`, `runtime-only` olur | Sağlayıcı kopyalanmış yenileme belirteçlerini kalıcı yapmadan harici kimlik doğrulama bilgilerini yeniden kullanıyorsa                     |
| 10  | `shouldDeferSyntheticProfileAuth` | Saklanan sentetik profil yer tutucularını ortam/yapılandırma tabanlı kimlik doğrulamanın altına düşürür       | Sağlayıcı, öncelik kazanmaması gereken sentetik yer tutucu profiller saklıyorsa                                                            |
| 11  | `resolveDynamicModel`             | Yerel kayıt sisteminde henüz bulunmayan sağlayıcıya ait model kimlikleri için eşzamanlı geri dönüş            | Sağlayıcı herhangi bir üst akış model kimliğini kabul ediyorsa                                                                              |
| 12  | `prepareDynamicModel`             | Eşzamansız ısınma yapar, sonra `resolveDynamicModel` yeniden çalışır                                            | Sağlayıcı bilinmeyen kimlikleri çözmeden önce ağ meta verisine ihtiyaç duyuyorsa                                                            |
| 13  | `normalizeResolvedModel`          | Gömülü çalıştırıcı çözülmüş modeli kullanmadan önce son yeniden yazımı yapar                                   | Sağlayıcının aktarım yeniden yazımlarına ihtiyacı varsa ancak yine de bir çekirdek aktarımı kullanıyorsa                                   |
| 14  | `contributeResolvedModelCompat`   | Başka bir uyumlu aktarım arkasındaki satıcı modelleri için uyumluluk bayrakları katkısı yapar                 | Sağlayıcı, sağlayıcıyı devralmadan kendi modellerini vekil aktarımlarda tanıyorsa                                                          |
| 15  | `capabilities`                    | Paylaşılan çekirdek mantık tarafından kullanılan sağlayıcıya ait transcript/araç meta verisi                   | Sağlayıcı transcript/sağlayıcı ailesi tuhaflıklarına ihtiyaç duyuyorsa                                                                      |
| 16  | `normalizeToolSchemas`            | Gömülü çalıştırıcı görmeden önce araç şemalarını normalize eder                                                | Sağlayıcının aktarım ailesine ait şema temizliğine ihtiyacı varsa                                                                           |
| 17  | `inspectToolSchemas`              | Normalizasyondan sonra sağlayıcıya ait şema tanılamalarını yüzeye çıkarır                                      | Sağlayıcı çekirdeğe sağlayıcıya özgü kurallar öğretmeden anahtar sözcük uyarıları istiyorsa                                                |
| 18  | `resolveReasoningOutputMode`      | Doğal ile etiketli gerekçelendirme-çıktısı sözleşmesi arasında seçim yapar                                     | Sağlayıcı doğal alanlar yerine etiketli gerekçelendirme/nihai çıktı istiyorsa                                                               |
| 19  | `prepareExtraParams`              | Genel akış seçeneği sarmalayıcılarından önce istek parametresi normalizasyonu yapar                            | Sağlayıcının varsayılan istek parametrelerine veya sağlayıcı başına parametre temizliğine ihtiyacı varsa                                   |
| 20  | `createStreamFn`                  | Normal akış yolunu tamamen özel bir aktarım ile değiştirir                                                     | Sağlayıcının yalnızca bir sarmalayıcıya değil, özel bir kablo protokolüne ihtiyacı varsa                                                   |
| 21  | `wrapStreamFn`                    | Genel sarmalayıcılar uygulandıktan sonra akışı sarar                                                           | Sağlayıcının özel bir aktarım olmadan istek üstbilgisi/gövdesi/model uyumluluk sarmalayıcılarına ihtiyacı varsa                           |
| 22  | `resolveTransportTurnState`       | Sağlayıcıya özgü tur başına aktarım üstbilgileri veya meta veri ekler                                          | Sağlayıcı, genel aktarımların sağlayıcıya özgü doğal tur kimliğini göndermesini istiyorsa                                                  |
| 23  | `resolveWebSocketSessionPolicy`   | Doğal WebSocket üstbilgileri veya oturum soğuma politikası ekler                                               | Sağlayıcı, genel WS aktarımlarının oturum üstbilgilerini veya geri dönüş politikasını ayarlamasını istiyorsa                              |
| 24  | `formatApiKey`                    | Kimlik doğrulama profili biçimleyicisi: saklanan profil çalışma zamanı `apiKey` dizgesi olur                  | Sağlayıcı ek kimlik doğrulama meta verisi saklıyorsa ve özel bir çalışma zamanı belirteç biçimine ihtiyaç duyuyorsa                        |
| 25  | `refreshOAuth`                    | Özel yenileme uç noktaları veya yenileme hatası politikası için OAuth yenileme geçersiz kılması               | Sağlayıcı paylaşılan `pi-ai` yenileyicilerine uymuyorsa                                                                                     |
| 26  | `buildAuthDoctorHint`             | OAuth yenilemesi başarısız olduğunda eklenen onarım ipucu                                                      | Sağlayıcının yenileme hatasından sonra sağlayıcıya ait kimlik doğrulama onarım rehberliğine ihtiyacı varsa                                 |
| 27  | `matchesContextOverflowError`     | Sağlayıcıya ait bağlam penceresi taşması eşleştiricisi                                                         | Sağlayıcının, genel sezgilerin kaçıracağı ham taşma hataları varsa                                                                          |
| 28  | `classifyFailoverReason`          | Sağlayıcıya ait failover nedeni sınıflandırması                                                                | Sağlayıcı ham API/aktarım hatalarını hız sınırı/aşırı yük/vb. olarak eşleyebiliyorsa                                                       |
| 29  | `isCacheTtlEligible`              | Proxy/arka taşıma sağlayıcıları için istem önbelleği politikası                                                | Sağlayıcının vekile özgü önbellek TTL geçitlemesine ihtiyacı varsa                                                                          |
| 30  | `buildMissingAuthMessage`         | Genel eksik kimlik doğrulama kurtarma mesajının yerine geçer                                                   | Sağlayıcının sağlayıcıya özgü eksik kimlik doğrulama kurtarma ipucuna ihtiyacı varsa                                                       |
| 31  | `suppressBuiltInModel`            | Eski üst akış model bastırması ve isteğe bağlı kullanıcıya dönük hata ipucu                                    | Sağlayıcının eski üst akış satırlarını gizlemesi veya bunları bir satıcı ipucuyla değiştirmesi gerekiyorsa                                 |
| 32  | `augmentModelCatalog`             | Keşiften sonra sentetik/nihai katalog satırları ekler                                                          | Sağlayıcının `models list` ve seçicilerde sentetik ileri uyumluluk satırlarına ihtiyacı varsa                                              |
| 33  | `isBinaryThinking`                | İkili düşünme sağlayıcıları için açık/kapalı gerekçelendirme anahtarı                                          | Sağlayıcı yalnızca ikili düşünme açık/kapalı sunuyorsa                                                                                      |
| 34  | `supportsXHighThinking`           | Seçili modeller için `xhigh` gerekçelendirme desteği                                                           | Sağlayıcı `xhigh` değerini yalnızca modellerin bir alt kümesinde istiyorsa                                                                  |
| 35  | `resolveDefaultThinkingLevel`     | Belirli bir model ailesi için varsayılan `/think` düzeyi                                                       | Sağlayıcı bir model ailesi için varsayılan `/think` politikasına sahipse                                                                    |
| 36  | `isModernModelRef`                | Canlı profil filtreleri ve smoke seçimi için modern model eşleştiricisi                                        | Sağlayıcı canlı/smoke tercih edilen model eşleştirmesine sahipse                                                                            |
| 37  | `prepareRuntimeAuth`              | Çıkarımdan hemen önce yapılandırılmış kimlik bilgisini gerçek çalışma zamanı belirtecine/anahtarına dönüştürür | Sağlayıcı belirteç değişimine veya kısa ömürlü istek kimlik bilgisine ihtiyaç duyuyorsa                                                    |
| 38  | `resolveUsageAuth`                | `/usage` ve ilgili durum yüzeyleri için kullanım/faturalama kimlik bilgilerini çözer                           | Sağlayıcının özel kullanım/kota belirteci ayrıştırmasına veya farklı bir kullanım kimlik bilgisine ihtiyacı varsa                          |
| 39  | `fetchUsageSnapshot`              | Kimlik doğrulama çözüldükten sonra sağlayıcıya özgü kullanım/kota anlık görüntülerini getirir ve normalize eder | Sağlayıcının sağlayıcıya özgü kullanım uç noktasına veya yük ayrıştırıcısına ihtiyacı varsa                                                |
| 40  | `createEmbeddingProvider`         | Bellek/arama için sağlayıcıya ait bir embedding bağdaştırıcısı oluşturur                                       | Bellek embedding davranışı sağlayıcı plugin'i ile birlikte olmalıdır                                                                        |
| 41  | `buildReplayPolicy`               | Sağlayıcı için transcript işlemeyi kontrol eden bir yeniden oynatma politikası döndürür                        | Sağlayıcının özel transcript politikasına ihtiyacı varsa (örneğin düşünme bloklarını çıkarma)                                              |
| 42  | `sanitizeReplayHistory`           | Genel transcript temizliğinden sonra yeniden oynatma geçmişini yeniden yazar                                   | Sağlayıcı, paylaşılan sıkıştırma yardımcılarının ötesinde sağlayıcıya özgü yeniden oynatma yeniden yazımlarına ihtiyaç duyuyorsa           |
| 43  | `validateReplayTurns`             | Gömülü çalıştırıcıdan önce yeniden oynatma turlarını son kez doğrular veya yeniden şekillendirir              | Sağlayıcı aktarımı, genel temizlikten sonra daha sıkı tur doğrulaması gerektiriyorsa                                                       |
| 44  | `onModelSelected`                 | Sağlayıcıya ait seçim sonrası yan etkileri çalıştırır                                                          | Sağlayıcının bir model etkin olduğunda telemetriye veya sağlayıcıya ait duruma ihtiyacı varsa                                              |

`normalizeModelId`, `normalizeTransport` ve `normalizeConfig` önce
eşleşen sağlayıcı plugin'ini kontrol eder, ardından model
kimliğini veya aktarımı/yapılandırmayı gerçekten değiştiren biri bulunana kadar kanca yetenekli diğer sağlayıcı plugin'lerine düşer. Bu,
çağıranın hangi paketlenmiş plugin'in yeniden yazıma sahip olduğunu bilmesini gerektirmeden
takma ad/uyumluluk sağlayıcı geçişlerinin çalışmasını sağlar. Hiçbir sağlayıcı kancası desteklenen
bir Google ailesi yapılandırma girdisini yeniden yazmazsa, paketlenmiş Google yapılandırma normalleştiricisi yine de
bu uyumluluk temizliğini uygular.

Sağlayıcının tamamen özel bir kablo protokolüne veya özel istek yürütücüsüne ihtiyacı varsa,
bu farklı bir uzantı sınıfıdır. Bu kancalar,
OpenClaw'un normal çıkarım döngüsü üzerinde çalışan sağlayıcı davranışı içindir.

### Sağlayıcı örneği

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

### Yerleşik örnekler

- Anthropic; `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`
  ve `wrapStreamFn` kullanır çünkü Claude 4.6 ileri uyumluluğuna,
  sağlayıcı ailesi ipuçlarına, kimlik doğrulama onarım rehberliğine, kullanım uç noktası entegrasyonuna,
  istem önbelleği uygunluğuna, kimlik doğrulama farkındalıklı yapılandırma varsayılanlarına, Claude
  varsayılan/uyarlanabilir düşünme politikasına ve beta üstbilgileri, `/fast` / `serviceTier` ile `context1m` için
  Anthropic'e özgü akış şekillendirmesine sahiptir.
- Anthropic'in Claude'a özgü akış yardımcıları şimdilik paketlenmiş plugin'in kendi
  genel `api.ts` / `contract-api.ts` geçişinde kalır. Bu paket yüzeyi,
  genel SDK'yi tek bir sağlayıcının beta-header kuralları çevresinde genişletmek yerine
  `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` ve daha alt düzey
  Anthropic sarmalayıcı oluşturucularını dışa aktarır.
- OpenAI; `resolveDynamicModel`, `normalizeResolvedModel` ve
  `capabilities` ile birlikte `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` ve `isModernModelRef`
  kullanır çünkü GPT-5.4 ileri uyumluluğuna, doğrudan OpenAI
  `openai-completions` -> `openai-responses` normalizasyonuna, Codex farkındalıklı kimlik doğrulama
  ipuçlarına, Spark bastırmasına, sentetik OpenAI liste satırlarına ve GPT-5 düşünme /
  canlı model politikasına sahiptir; `openai-responses-defaults` akış ailesi ise atıf üstbilgileri,
  `/fast`/`serviceTier`, metin ayrıntı düzeyi, doğal Codex web arama,
  gerekçelendirme uyumluluğu yük şekillendirmesi ve Responses bağlam yönetimi için
  paylaşılan doğal OpenAI Responses sarmalayıcılarına sahiptir.
- OpenRouter; sağlayıcı bir geçiş sağlayıcısı olduğu ve OpenClaw'un durağan
  kataloğu güncellenmeden önce yeni model kimlikleri açığa çıkarabileceği için `catalog` ile birlikte `resolveDynamicModel` ve
  `prepareDynamicModel` kullanır; ayrıca
  sağlayıcıya özgü istek üstbilgilerini, yönlendirme meta verisini, gerekçelendirme yamalarını ve
  istem önbelleği politikasını çekirdekten uzak tutmak için `capabilities`, `wrapStreamFn` ve `isCacheTtlEligible` de kullanır. Yeniden oynatma politikası
  `passthrough-gemini` ailesinden gelirken, `openrouter-thinking` akış ailesi ise
  proxy gerekçelendirme ekleme ile desteklenmeyen model / `auto` atlamalarına sahiptir.
- GitHub Copilot; sağlayıcıya ait cihaz oturum açma, model geri dönüş davranışı, Claude transcript tuhaflıkları,
  GitHub belirteci -> Copilot belirteci değişimi ve sağlayıcıya ait kullanım uç noktası gerektiği için
  `catalog`, `auth`, `resolveDynamicModel` ve `capabilities` ile birlikte `prepareRuntimeAuth` ve `fetchUsageSnapshot`
  kullanır.
- OpenAI Codex; hâlâ çekirdek OpenAI aktarımları üzerinde çalışmasına rağmen
  kendi aktarım/temel URL normalizasyonuna, OAuth yenileme geri dönüş politikasına, varsayılan aktarım seçimine,
  sentetik Codex katalog satırlarına ve ChatGPT kullanım uç noktası entegrasyonuna sahip olduğu için
  `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` ve `augmentModelCatalog` ile birlikte
  `prepareExtraParams`, `resolveUsageAuth` ve `fetchUsageSnapshot` kullanır; doğrudan OpenAI ile aynı `openai-responses-defaults` akış ailesini paylaşır.
- Google AI Studio ve Gemini CLI OAuth; `google-gemini` yeniden oynatma ailesi
  Gemini 3.1 ileri uyumluluk geri dönüşüne,
  doğal Gemini yeniden oynatma doğrulamasına, önyükleme yeniden oynatma temizliğine,
  etiketli gerekçelendirme-çıktı moduna ve modern model eşleştirmesine sahip olduğu için
  `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` ve `isModernModelRef` kullanır; `google-thinking` akış ailesi ise Gemini düşünme yükü normalizasyonuna sahiptir;
  Gemini CLI OAuth ayrıca belirteç biçimlendirme, belirteç ayrıştırma ve kota uç noktası bağlantısı için
  `formatApiKey`, `resolveUsageAuth` ve
  `fetchUsageSnapshot` kullanır.
- Anthropic Vertex, `anthropic-by-model` yeniden oynatma ailesi aracılığıyla `buildReplayPolicy`
  kullanır; böylece Claude'a özgü yeniden oynatma temizliği tüm `anthropic-messages`
  aktarımı yerine Claude kimlikleriyle sınırlı kalır.
- Amazon Bedrock; Anthropic-on-Bedrock trafiği için Bedrock'a özgü daraltma/hazır değil/bağlam taşması hata sınıflandırmasına sahip olduğu için
  `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` ve `resolveDefaultThinkingLevel` kullanır; yeniden oynatma politikası ise
  yine aynı yalnızca-Claude `anthropic-by-model` korumasını paylaşır.
- OpenRouter, Kilocode, Opencode ve Opencode Go;
  Gemini modellerini OpenAI uyumlu aktarımlar üzerinden vekil sundukları ve doğal Gemini yeniden oynatma doğrulaması veya
  önyükleme yeniden yazımları olmadan Gemini düşünce-imzası temizliğine ihtiyaç duydukları için
  `passthrough-gemini` yeniden oynatma ailesi aracılığıyla `buildReplayPolicy`
  kullanır.
- MiniMax, `hybrid-anthropic-openai` yeniden oynatma ailesi aracılığıyla `buildReplayPolicy`
  kullanır çünkü tek bir sağlayıcı hem Anthropic-message hem de OpenAI uyumlu anlamlara sahiptir;
  Anthropic tarafında yalnızca Claude'a özgü düşünme bloğu düşürmeyi sürdürürken gerekçelendirme
  çıktı modunu tekrar doğala geçersiz kılar ve `minimax-fast-mode` akış ailesi
  paylaşılan akış yolunda hızlı mod model yeniden yazımlarına sahiptir.
- Moonshot; hâlâ paylaşılan OpenAI aktarımını kullandığı ancak sağlayıcıya ait düşünme yükü normalizasyonuna ihtiyaç duyduğu için
  `catalog` ile birlikte `wrapStreamFn` kullanır; `moonshot-thinking` akış ailesi ise yapılandırma ile `/think` durumunu
  kendi doğal ikili düşünme yüküne eşler.
- Kilocode; sağlayıcıya ait istek üstbilgilerine,
  gerekçelendirme yükü normalizasyonuna, Gemini transcript ipuçlarına ve Anthropic
  önbellek-TTL geçitlemesine ihtiyaç duyduğu için `catalog`, `capabilities`, `wrapStreamFn` ve
  `isCacheTtlEligible` kullanır; `kilocode-thinking` akış ailesi ise
  paylaşılan proxy akış yolunda Kilo düşünme eklemeyi tutarken `kilo/auto` ve açık gerekçelendirme yüklerini desteklemeyen
  diğer proxy model kimliklerini atlar.
- Z.AI; GLM-5 geri dönüşüne,
  `tool_stream` varsayılanlarına, ikili düşünme UX'ine, modern model eşleştirmesine ve hem kullanım kimlik doğrulaması hem de kota getirmeye sahip olduğu için
  `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` ve `fetchUsageSnapshot` kullanır; `tool-stream-default-on` akış ailesi ise varsayılan açık `tool_stream` sarmalayıcısını
  sağlayıcı başına elle yazılmış yapıştırma kodundan uzak tutar.
- xAI; doğal xAI Responses aktarımı normalizasyonuna, Grok hızlı mod
  takma ad yeniden yazımlarına, varsayılan `tool_stream`e, katı araç / gerekçelendirme yükü
  temizliğine, plugin'e ait araçlar için geri dönüş kimlik doğrulaması yeniden kullanımına, ileri uyumluluk Grok
  model çözümlemesine ve xAI araç-şema
  profili, desteklenmeyen şema anahtar sözcükleri, doğal `web_search` ve HTML varlık
  araç çağrısı argüman çözümlemesi gibi sağlayıcıya ait uyumluluk yamalarına sahip olduğu için
  `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` ve `isModernModelRef`
  kullanır.
- Mistral, OpenCode Zen ve OpenCode Go; transcript/araç
  tuhaflıklarını çekirdekten uzak tutmak için yalnızca `capabilities` kullanır.
- `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` ve `volcengine` gibi
  yalnızca katalog kullanan paketlenmiş sağlayıcılar yalnızca `catalog` kullanır.
- Qwen, metin sağlayıcısı için `catalog` ile birlikte çok modlu yüzeyleri için
  paylaşılan medya-anlama ve video-oluşturma kayıtlarını kullanır.
- MiniMax ve Xiaomi, çıkarım hâlâ paylaşılan aktarımlardan geçse de
  `/usage` davranışları plugin'e ait olduğu için `catalog` ile birlikte kullanım kancalarını kullanır.

## Çalışma zamanı yardımcıları

Plugin'ler seçilmiş çekirdek yardımcılarına `api.runtime` üzerinden erişebilir. TTS için:

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

Notlar:

- `textToSpeech`, dosya/sesli not yüzeyleri için normal çekirdek TTS çıktı yükünü döndürür.
- Çekirdek `messages.tts` yapılandırmasını ve sağlayıcı seçimini kullanır.
- PCM ses arabelleği + örnekleme hızı döndürür. Plugin'ler sağlayıcılar için yeniden örnekleme/kodlama yapmalıdır.
- `listVoices`, sağlayıcı başına isteğe bağlıdır. Satıcıya ait ses seçiciler veya kurulum akışları için kullanın.
- Ses listeleri yerel ayar, cinsiyet ve kişilik etiketleri gibi daha zengin meta veriler içerebilir.
- OpenAI ve ElevenLabs bugün telefon desteği sunuyor. Microsoft sunmuyor.

Plugin'ler ayrıca `api.registerSpeechProvider(...)` üzerinden konuşma sağlayıcıları kaydedebilir.

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

Notlar:

- TTS politikasını, geri dönüşü ve yanıt teslimatını çekirdekte tutun.
- Satıcıya ait sentez davranışı için konuşma sağlayıcıları kullanın.
- Eski Microsoft `edge` girdisi `microsoft` sağlayıcı kimliğine normalize edilir.
- Tercih edilen sahiplik modeli şirket odaklıdır: tek bir satıcı plugin'i
  OpenClaw bu yetenek sözleşmelerini ekledikçe metin, konuşma, görsel ve gelecekteki medya sağlayıcılarına sahip olabilir.

Görsel/ses/video anlama için plugin'ler genel bir anahtar/değer torbası yerine
yazımlı bir medya-anlama sağlayıcısı kaydeder:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Notlar:

- Orkestrasyonu, geri dönüşü, yapılandırmayı ve kanal bağlamasını çekirdekte tutun.
- Satıcı davranışını sağlayıcı plugin'inde tutun.
- Artımlı genişleme yazımlı kalmalıdır: yeni isteğe bağlı yöntemler, yeni isteğe bağlı
  sonuç alanları, yeni isteğe bağlı yetenekler.
- Video oluşturma zaten aynı deseni izler:
  - çekirdek yetenek sözleşmesine ve çalışma zamanı yardımcısına sahiptir
  - satıcı plugin'leri `api.registerVideoGenerationProvider(...)` kaydeder
  - özellik/kanal plugin'leri `api.runtime.videoGeneration.*` tüketir

Medya-anlama çalışma zamanı yardımcıları için plugin'ler şunları çağırabilir:

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

Ses yazıya dökme için plugin'ler medya-anlama çalışma zamanını
veya eski STT takma adını kullanabilir:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Notlar:

- `api.runtime.mediaUnderstanding.*`, görsel/ses/video anlama için
  tercih edilen paylaşılan yüzeydir.
- Çekirdek medya-anlama ses yapılandırmasını (`tools.media.audio`) ve sağlayıcı geri dönüş sırasını kullanır.
- Hiçbir yazıya dökme çıktısı üretilmediğinde `{ text: undefined }` döndürür (örneğin atlanan/desteklenmeyen girdi).
- `api.runtime.stt.transcribeAudioFile(...)` uyumluluk takma adı olarak kalır.

Plugin'ler ayrıca `api.runtime.subagent` üzerinden arka plan alt ajan çalıştırmaları başlatabilir:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Notlar:

- `provider` ve `model`, kalıcı oturum değişiklikleri değil, çalışma başına isteğe bağlı geçersiz kılmalardır.
- OpenClaw bu geçersiz kılma alanlarını yalnızca güvenilir çağıranlar için dikkate alır.
- Plugin'e ait geri dönüş çalıştırmaları için operatörler `plugins.entries.<id>.subagent.allowModelOverride: true` ile açıkça izin vermelidir.
- Güvenilir plugin'leri belirli kanonik `provider/model` hedefleriyle sınırlamak için `plugins.entries.<id>.subagent.allowedModels` kullanın veya herhangi bir hedefe açıkça izin vermek için `"*"` kullanın.
- Güvenilmeyen plugin alt ajan çalıştırmaları yine de çalışır, ancak geçersiz kılma istekleri sessizce geri dönmek yerine reddedilir.

Web arama için plugin'ler ajan araç bağlamasına
erişmek yerine paylaşılan çalışma zamanı yardımcısını tüketebilir:

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

Plugin'ler ayrıca
`api.registerWebSearchProvider(...)` üzerinden web arama sağlayıcıları da kaydedebilir.

Notlar:

- Sağlayıcı seçimini, kimlik bilgisi çözümlemesini ve paylaşılan istek anlamlarını çekirdekte tutun.
- Satıcıya özgü arama aktarımları için web arama sağlayıcıları kullanın.
- `api.runtime.webSearch.*`, ajan araç sarmalayıcısına bağımlı olmadan arama davranışına ihtiyaç duyan özellik/kanal plugin'leri için
  tercih edilen paylaşılan yüzeydir.

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

- `generate(...)`: yapılandırılmış görsel oluşturma sağlayıcı zincirini kullanarak görsel oluşturur.
- `listProviders(...)`: kullanılabilir görsel oluşturma sağlayıcılarını ve yeteneklerini listeler.

## Gateway HTTP rotaları

Plugin'ler `api.registerHttpRoute(...)` ile HTTP uç noktaları açığa çıkarabilir.

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

Rota alanları:

- `path`: gateway HTTP sunucusu altındaki rota yolu.
- `auth`: gerekli. Normal gateway kimlik doğrulaması gerektirmek için `"gateway"`, plugin tarafından yönetilen kimlik doğrulama/webhook doğrulaması için `"plugin"` kullanın.
- `match`: isteğe bağlı. `"exact"` (varsayılan) veya `"prefix"`.
- `replaceExisting`: isteğe bağlı. Aynı plugin'in kendi mevcut rota kaydını değiştirmesine izin verir.
- `handler`: rota isteği işlediğinde `true` döndürmelidir.

Notlar:

- `api.registerHttpHandler(...)` kaldırıldı ve plugin yükleme hatasına neden olur. Bunun yerine `api.registerHttpRoute(...)` kullanın.
- Plugin rotaları `auth` değerini açıkça bildirmelidir.
- Tam `path + match` çakışmaları, `replaceExisting: true` olmadığı sürece reddedilir ve bir plugin başka bir plugin'in rotasını değiştiremez.
- Farklı `auth` düzeylerine sahip çakışan rotalar reddedilir. `exact`/`prefix` ardışık geçiş zincirlerini yalnızca aynı kimlik doğrulama düzeyinde tutun.
- `auth: "plugin"` rotaları otomatik olarak operatör çalışma zamanı kapsamları almaz. Bunlar ayrıcalıklı Gateway yardımcı çağrıları için değil, plugin tarafından yönetilen webhook'lar/imza doğrulaması içindir.
- `auth: "gateway"` rotaları bir Gateway istek çalışma zamanı kapsamında çalışır, ancak bu kapsam bilerek muhafazakârdır:
  - paylaşılan gizli bearer kimlik doğrulaması (`gateway.auth.mode = "token"` / `"password"`) kullanıldığında, çağıran `x-openclaw-scopes` gönderse bile plugin-rotası çalışma zamanı kapsamları `operator.write` olarak sabitlenir
  - güvenilir kimlik taşıyan HTTP modları (örneğin `trusted-proxy` veya özel girişte `gateway.auth.mode = "none"`) `x-openclaw-scopes` değerini yalnızca üstbilgi açıkça mevcutsa dikkate alır
  - bu kimlik taşıyan plugin-rotası isteklerinde `x-openclaw-scopes` yoksa, çalışma zamanı kapsamı `operator.write` değerine geri düşer
- Pratik kural: gateway kimlik doğrulamalı bir plugin rotasının örtük bir yönetici yüzeyi olduğunu varsaymayın. Rotanız yöneticiye özel davranış gerektiriyorsa, kimlik taşıyan bir kimlik doğrulama modu isteyin ve açık `x-openclaw-scopes` üstbilgisi sözleşmesini belgelendirin.

## Plugin SDK içe aktarma yolları

Plugin yazarken tek parça `openclaw/plugin-sdk` içe aktarması yerine
SDK alt yollarını kullanın:

- plugin kayıt ilkelleri için `openclaw/plugin-sdk/plugin-entry`.
- genel paylaşılan plugin'e dönük sözleşme için `openclaw/plugin-sdk/core`.
- kök `openclaw.json` Zod şema dışa aktarımı (`OpenClawSchema`) için `openclaw/plugin-sdk/config-schema`.
- `openclaw/plugin-sdk/channel-setup`,
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
  `openclaw/plugin-sdk/secret-input` ve
  `openclaw/plugin-sdk/webhook-ingress` gibi kararlı kanal ilkelleri, paylaşılan kurulum/kimlik doğrulama/yanıt/webhook
  bağlaması içindir. `channel-inbound`; debounce, mention eşleştirme,
  gelen mention-policy yardımcıları, zarf biçimlendirme ve gelen zarf
  bağlam yardımcıları için paylaşılan yuvadır.
  `channel-setup`, dar isteğe bağlı kurulum geçişidir.
  `setup-runtime`, `setupEntry` /
  ertelenmiş başlangıç tarafından kullanılan, içe aktarma açısından güvenli kurulum yama bağdaştırıcıları dahil çalışma zamanı güvenli kurulum yüzeyidir.
  `setup-adapter-runtime`, ortama duyarlı hesap-kurulum bağdaştırıcısı geçişidir.
  `setup-tools`, küçük CLI/arşiv/belge yardımcı geçişidir (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- `openclaw/plugin-sdk/channel-config-helpers`,
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
  `openclaw/plugin-sdk/runtime-store` ve
  `openclaw/plugin-sdk/directory-runtime` gibi alan alt yolları, paylaşılan çalışma zamanı/yapılandırma yardımcıları içindir.
  `telegram-command-config`, Telegram özel
  komut normalizasyonu/doğrulaması için dar genel geçiştir ve paketlenmiş
  Telegram sözleşme yüzeyi geçici olarak kullanılamasa bile kullanılabilir kalır.
  `text-runtime`, yardımcıya görünür metin çıkarma, markdown render/parçalama yardımcıları, sansürleme
  yardımcıları, yönerge etiketi yardımcıları ve güvenli metin yardımcıları dahil
  paylaşılan metin/markdown/günlükleme geçişidir.
- Onaya özgü kanal geçişleri, plugin üzerinde tek bir `approvalCapability`
  sözleşmesini tercih etmelidir. Çekirdek daha sonra onay kimlik doğrulaması, teslimat, render,
  doğal yönlendirme ve tembel doğal işleyici davranışını alakasız plugin alanlarına karıştırmak yerine
  bu tek yetenek üzerinden okur.
- `openclaw/plugin-sdk/channel-runtime` kullanımdan kaldırılmıştır ve yalnızca eski plugin'ler için
  uyumluluk geçişi olarak kalır. Yeni kod bunun yerine daha dar genel ilkelleri içe aktarmalıdır ve repo kodu bu
  geçiş için yeni içe aktarmalar eklememelidir.
- Paketlenmiş uzantı iç yapıları özel kalır. Dış plugin'ler yalnızca
  `openclaw/plugin-sdk/*` alt yollarını kullanmalıdır. OpenClaw çekirdeği/test kodu,
  `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` ve `login-qr-api.js` gibi dar kapsamlı dosyalar gibi
  bir plugin paket kökü altındaki repo genel giriş noktalarını kullanabilir. Çekirdekten veya
  başka bir uzantıdan asla bir plugin paketinin `src/*` yolunu içe aktarmayın.
- Repo giriş noktası ayrımı:
  `<plugin-package-root>/api.js` yardımcı/türler varilidir,
  `<plugin-package-root>/runtime-api.js` yalnızca çalışma zamanı varilidir,
  `<plugin-package-root>/index.js` paketlenmiş plugin girişidir,
  ve `<plugin-package-root>/setup-entry.js` kurulum plugin girişidir.
- Güncel paketlenmiş sağlayıcı örnekleri:
  - Anthropic, `wrapAnthropicProviderStream`, beta-header yardımcıları ve `service_tier`
    ayrıştırması gibi Claude akış yardımcıları için `api.js` / `contract-api.js` kullanır.
  - OpenAI, sağlayıcı oluşturucuları, varsayılan model yardımcıları ve
    gerçek zamanlı sağlayıcı oluşturucuları için `api.js` kullanır.
  - OpenRouter, kendi sağlayıcı oluşturucusu ve katılım/yapılandırma
    yardımcıları için `api.js` kullanırken `register.runtime.js`, repo içi kullanım için genel
    `plugin-sdk/provider-stream` yardımcılarını yeniden dışa aktarabilir.
- Yüklenen yüzler üzerinden gelen genel giriş noktaları, mevcutsa etkin çalışma zamanı yapılandırma anlık görüntüsünü tercih eder;
  OpenClaw henüz çalışma zamanı anlık görüntüsü sunmuyorsa diskteki çözülmüş yapılandırma dosyasına geri döner.
- Genel paylaşılan ilkeller tercih edilen genel SDK sözleşmesi olmaya devam eder. Kanal markalı küçük,
  ayrılmış bir uyumluluk kümesi hâlâ mevcuttur. Bunları yeni
  üçüncü taraf içe aktarma hedefleri olarak değil, paket bakımı/uyumluluk geçişleri olarak değerlendirin; yeni çapraz kanal sözleşmeleri yine genel
  `plugin-sdk/*` alt yollarında veya plugin'in yerel `api.js` /
  `runtime-api.js` varillerinde yer almalıdır.

Uyumluluk notu:

- Yeni kod için kök `openclaw/plugin-sdk` varilinden kaçının.
- Önce dar kararlı ilkelleri tercih edin. Daha yeni setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool alt yolları, yeni
  paketlenmiş ve dış plugin çalışmaları için amaçlanan sözleşmedir.
  Hedef ayrıştırma/eşleştirme `openclaw/plugin-sdk/channel-targets` üzerinde olmalıdır.
  Mesaj eylemi geçitleri ve tepki mesaj kimliği yardımcıları
  `openclaw/plugin-sdk/channel-actions` üzerinde olmalıdır.
- Paketlenmiş uzantıya özgü yardımcı varilleri varsayılan olarak kararlı değildir. Eğer bir
  yardımcı yalnızca paketlenmiş bir uzantı tarafından gerekiyorsa, bunu
  `openclaw/plugin-sdk/<extension>` içine yükseltmek yerine uzantının yerel `api.js` veya `runtime-api.js` geçişi arkasında tutun.
- Yeni paylaşılan yardımcı geçişleri kanal markalı değil, genel olmalıdır. Paylaşılan hedef
  ayrıştırma `openclaw/plugin-sdk/channel-targets` üzerinde olmalıdır; kanala özgü
  iç yapılar ise sahip plugin'in yerel `api.js` veya `runtime-api.js`
  geçişi arkasında kalmalıdır.
- `image-generation`,
  `media-understanding` ve `speech` gibi yeteneğe özgü alt yollar bugün paketlenmiş/yerel plugin'ler bunları kullandığı için vardır.
  Bunların varlığı tek başına, dışa aktarılan her yardımcının
  uzun vadede sabitlenmiş bir dış sözleşme olduğu anlamına gelmez.

## Message aracı şemaları

Plugin'ler kanala özgü `describeMessageTool(...)` şema
katkılarına sahip olmalıdır. Sağlayıcıya özgü alanları paylaşılan çekirdekte değil plugin içinde tutun.

Paylaşılan taşınabilir şema parçaları için
`openclaw/plugin-sdk/channel-actions` üzerinden dışa aktarılan genel yardımcıları yeniden kullanın:

- düğme ızgarası tarzı yükler için `createMessageToolButtonsSchema()`
- yapılandırılmış kart yükleri için `createMessageToolCardSchema()`

Bir şema biçimi yalnızca bir sağlayıcı için anlamlıysa, bunu
paylaşılan SDK'ye yükseltmek yerine o plugin'in kendi kaynağında tanımlayın.

## Kanal hedef çözümleme

Kanal plugin'leri kanala özgü hedef anlamlarına sahip olmalıdır. Paylaşılan
giden ana makineyi genel tutun ve sağlayıcı kuralları için mesajlaşma bağdaştırıcı yüzeyini kullanın:

- `messaging.inferTargetChatType({ to })`, normalize edilmiş bir hedefin
  dizin aramasından önce `direct`, `group` veya `channel` olarak ele alınıp alınmayacağına karar verir.
- `messaging.targetResolver.looksLikeId(raw, normalized)`, bir girdinin
  dizin araması yerine doğrudan kimlik benzeri çözümlemeye atlayıp atlamayacağını çekirdeğe söyler.
- `messaging.targetResolver.resolveTarget(...)`, çekirdeğin normalizasyondan veya
  dizin kaçırmasından sonra son sağlayıcıya ait çözümlemeye ihtiyaç duyduğunda plugin geri dönüş yoludur.
- `messaging.resolveOutboundSessionRoute(...)`, hedef çözüldükten sonra
  sağlayıcıya özgü oturum rota kurulumuna sahiptir.

Önerilen ayrım:

- eşler/gruplar aranmadan önce gerçekleşmesi gereken kategori kararları için `inferTargetChatType` kullanın.
- "bunu açık/doğal hedef kimliği olarak işle" denetimleri için `looksLikeId` kullanın.
- geniş dizin araması için değil, sağlayıcıya özgü normalizasyon geri dönüşü için `resolveTarget` kullanın.
- sohbet kimlikleri, iş parçacığı kimlikleri, JID'ler, tanıtıcılar ve oda
  kimlikleri gibi sağlayıcıya özgü doğal kimlikleri genel SDK alanlarında değil
  `target` değerlerinde veya sağlayıcıya özgü parametrelerde tutun.

## Yapılandırma destekli dizinler

Yapılandırmadan dizin girdileri türeten plugin'ler, bu mantığı plugin içinde tutmalı
ve paylaşılan yardımcıları
`openclaw/plugin-sdk/directory-runtime` üzerinden yeniden kullanmalıdır.

Bir kanal aşağıdakiler gibi yapılandırma destekli eşlere/gruplara ihtiyaç duyduğunda bunu kullanın:

- allowlist güdümlü DM eşleri
- yapılandırılmış kanal/grup haritaları
- hesap kapsamlı durağan dizin geri dönüşleri

`directory-runtime` içindeki paylaşılan yardımcılar yalnızca genel işlemleri yönetir:

- sorgu filtreleme
- limit uygulama
- tekilleştirme/normalizasyon yardımcıları
- `ChannelDirectoryEntry[]` oluşturma

Kanala özgü hesap inceleme ve kimlik normalizasyonu
plugin uygulamasında kalmalıdır.

## Sağlayıcı katalogları

Sağlayıcı plugin'leri, çıkarım için model kataloglarını
`registerProvider({ catalog: { run(...) { ... } } })` ile tanımlayabilir.

`catalog.run(...)`, OpenClaw'un
`models.providers` içine yazdığıyla aynı biçimi döndürür:

- tek bir sağlayıcı girdisi için `{ provider }`
- birden çok sağlayıcı girdisi için `{ providers }`

Sağlayıcıya özgü model kimlikleri, temel URL varsayılanları veya kimlik doğrulama kapılı model meta verileri
plugin'e ait olduğunda `catalog` kullanın.

`catalog.order`, bir plugin'in kataloğunun OpenClaw'un
yerleşik örtük sağlayıcılarına göre ne zaman birleşeceğini denetler:

- `simple`: düz API anahtarı veya ortam güdümlü sağlayıcılar
- `profile`: kimlik doğrulama profilleri olduğunda görünen sağlayıcılar
- `paired`: birden çok ilişkili sağlayıcı girdisi sentezleyen sağlayıcılar
- `late`: diğer örtük sağlayıcılardan sonra son geçiş

Daha sonraki sağlayıcılar anahtar çakışmasında kazanır; böylece plugin'ler aynı sağlayıcı kimliğine sahip
yerleşik bir sağlayıcı girdisini kasıtlı olarak geçersiz kılabilir.

Uyumluluk:

- `discovery` eski bir takma ad olarak hâlâ çalışır
- hem `catalog` hem `discovery` kayıtlıysa OpenClaw `catalog` kullanır

## Salt okunur kanal inceleme

Plugin'iniz bir kanal kaydediyorsa, `resolveAccount(...)` ile birlikte
`plugin.config.inspectAccount(cfg, accountId)` uygulamayı tercih edin.

Neden:

- `resolveAccount(...)` çalışma zamanı yoludur. Kimlik bilgilerinin
  tamamen somutlaştığını varsayabilir ve gerekli sırlar eksik olduğunda hızlıca hata verebilir.
- `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` ve doctor/config
  onarım akışları gibi salt okunur komut yollarının, yapılandırmayı açıklamak için çalışma zamanı kimlik bilgilerini somutlaştırması gerekmez.

Önerilen `inspectAccount(...)` davranışı:

- Yalnızca açıklayıcı hesap durumu döndürün.
- `enabled` ve `configured` değerlerini koruyun.
- Gerektiğinde şu gibi kimlik bilgisi kaynağı/durum alanları ekleyin:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Salt okunur kullanılabilirliği bildirmek için ham belirteç değerlerini döndürmeniz gerekmez.
  Durum tarzı komutlar için `tokenStatus: "available"` (ve eşleşen kaynak alanı) döndürmek yeterlidir.
- Bir kimlik bilgisi SecretRef üzerinden yapılandırılmış ancak mevcut komut yolunda kullanılamıyorsa
  `configured_unavailable` kullanın.

Bu, salt okunur komutların çökmesi veya hesabı yanlışlıkla yapılandırılmamış diye bildirmesi yerine
"yapılandırılmış ama bu komut yolunda kullanılamıyor" şeklinde rapor vermesini sağlar.

## Paket paketleri

Bir plugin dizini, `openclaw.extensions` içeren bir `package.json` dosyası içerebilir:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Her girdi bir plugin olur. Paket birden fazla uzantı listeliyorsa plugin kimliği
`name/<fileBase>` olur.

Plugin'iniz npm bağımlılıklarını içe aktarıyorsa, `node_modules`
mevcut olsun diye bunları o dizine kurun (`npm install` / `pnpm install`).

Güvenlik koruması: her `openclaw.extensions` girdisi, sembolik bağ çözümlemesinden sonra plugin
dizini içinde kalmalıdır. Paket dizininden çıkan girdiler
reddedilir.

Güvenlik notu: `openclaw plugins install`, plugin bağımlılıklarını
`npm install --omit=dev --ignore-scripts` ile kurar (yaşam döngüsü betikleri yok, çalışma zamanında geliştirme bağımlılıkları yok). Plugin bağımlılık
ağaçlarını "salt JS/TS" tutun ve `postinstall` derlemeleri gerektiren paketlerden kaçının.

İsteğe bağlı: `openclaw.setupEntry` hafif bir yalnızca kurulum modülünü işaret edebilir.
OpenClaw, devre dışı bir kanal plugin'i için kurulum yüzeylerine ihtiyaç duyduğunda veya
bir kanal plugin'i etkin ancak hâlâ yapılandırılmamış olduğunda tam plugin girişi yerine `setupEntry`
yükler. Bu, ana plugin girişiniz araçlar, kancalar veya diğer yalnızca çalışma zamanı
kodlarını da bağlıyorsa başlangıç ve kurulumu daha hafif tutar.

İsteğe bağlı: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`,
bir kanal plugin'ini, kanal zaten yapılandırılmış olsa bile gateway'nin
dinleme öncesi başlangıç aşamasında aynı `setupEntry` yoluna almayı seçebilir.

Bunu yalnızca `setupEntry`, gateway dinlemeye başlamadan önce
var olması gereken başlangıç yüzeyini tamamen kapsıyorsa kullanın. Pratikte bu, kurulum girdisinin
başlangıcın bağlı olduğu aşağıdaki gibi tüm kanala ait yetenekleri kaydetmesi gerektiği anlamına gelir:

- kanal kaydının kendisi
- gateway dinlemeye başlamadan önce kullanılabilir olması gereken tüm HTTP rotaları
- aynı pencere sırasında var olması gereken tüm gateway yöntemleri, araçlar veya hizmetleri

Tam girişiniz hâlâ gerekli herhangi bir başlangıç yeteneğine sahipse bu bayrağı etkinleştirmeyin.
Plugin'i varsayılan davranışta bırakın ve OpenClaw'un başlangıç sırasında
tam girişi yüklemesine izin verin.

Paketlenmiş kanallar ayrıca, tam kanal çalışma zamanı yüklenmeden önce çekirdeğin danışabileceği
yalnızca kurulum sözleşme yüzeyi yardımcıları yayımlayabilir. Mevcut kurulum
yükseltme yüzeyi şöyledir:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Çekirdek bu yüzeyi, tam plugin girişini yüklemeden eski bir tek hesaplı kanal
yapılandırmasını `channels.<id>.accounts.*` içine yükseltmesi gerektiğinde kullanır.
Matrix mevcut paketlenmiş örnektir: adlandırılmış hesaplar zaten varsa yalnızca auth/bootstrap anahtarlarını
adlandırılmış yükseltilmiş bir hesaba taşır ve her zaman
`accounts.default` oluşturmadan yapılandırılmış kanonik olmayan bir varsayılan hesap anahtarını koruyabilir.

Bu kurulum yama bağdaştırıcıları paketlenmiş sözleşme yüzeyi keşfini tembel tutar.
İçe aktarma zamanı hafif kalır; yükseltme yüzeyi modül içe aktarımında paketlenmiş kanal başlangıcına yeniden girmek yerine
yalnızca ilk kullanımda yüklenir.

Bu başlangıç yüzeyleri gateway RPC yöntemleri içerdiğinde, bunları
plugin'e özgü bir ön ek üzerinde tutun. Çekirdek yönetici ad alanları (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) ayrılmış kalır ve bir plugin daha dar kapsam istese bile
her zaman `operator.admin` olarak çözülür.

Örnek:

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

### Kanal katalog meta verisi

Kanal plugin'leri `openclaw.channel` üzerinden kurulum/keşif meta verisi ve
`openclaw.install` üzerinden kurulum ipuçları duyurabilir. Bu, çekirdek katalog verisini verisiz tutar.

Örnek:

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

Asgari örneğin ötesindeki yararlı `openclaw.channel` alanları:

- daha zengin katalog/durum yüzeyleri için `detailLabel`: ikincil etiket
- belge bağlantısı metnini geçersiz kılmak için `docsLabel`
- bu katalog girdisinin geride bırakması gereken daha düşük öncelikli plugin/kanal kimlikleri için `preferOver`
- seçim yüzeyi metin denetimleri için `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`
- giden biçimlendirme kararları için kanalın markdown yeteneğine sahip olduğunu işaretleyen `markdownCapable`
- `false` olarak ayarlandığında kanalı yapılandırılmış kanal listeleme yüzeylerinden gizleyen `exposure.configured`
- `false` olarak ayarlandığında kanalı etkileşimli kurulum/yapılandırma seçicilerinden gizleyen `exposure.setup`
- belge gezinme yüzeyleri için kanalı dahili/özel olarak işaretleyen `exposure.docs`
- uyumluluk için eski takma adlar hâlâ kabul edilir: `showConfigured` / `showInSetup`; tercih edilen alan `exposure`dur
- kanalı standart hızlı başlangıç `allowFrom` akışına dâhil eden `quickstartAllowFrom`
- yalnızca tek hesap olsa bile açık hesap bağlamasını zorunlu kılan `forceAccountBinding`
- duyuru hedeflerini çözerken oturum aramasını tercih eden `preferSessionLookupForAnnounceTarget`

OpenClaw ayrıca **harici kanal kataloglarını** da (örneğin bir MPM
kayıt dışa aktarımı) birleştirebilir. Şu konumlardan birine bir JSON dosyası bırakın:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Ya da `OPENCLAW_PLUGIN_CATALOG_PATHS` (veya `OPENCLAW_MPM_CATALOG_PATHS`) değişkenini
bir veya daha fazla JSON dosyasına yönlendirin (virgül/noktalı virgül/`PATH` ayırımlı). Her dosya
`{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` içermelidir. Ayrıştırıcı, `"entries"` anahtarı için
eski takma adlar olarak `"packages"` veya `"plugins"` anahtarlarını da kabul eder.

## Bağlam motoru plugin'leri

Bağlam motoru plugin'leri, alma, birleştirme
ve sıkıştırma için oturum bağlamı orkestrasyonuna sahiptir. Bunları plugin'inizden
`api.registerContextEngine(id, factory)` ile kaydedin, ardından etkin motoru
`plugins.slots.contextEngine` ile seçin.

Bunu, plugin'inizin varsayılan bağlam
işlem hattısını sadece bellek araması veya kanca eklemek yerine değiştirmesi ya da genişletmesi gerektiğinde kullanın.

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

Motorunuz sıkıştırma algoritmasına **sahip değilse**, `compact()`
uygulamasını koruyun ve açıkça ona devredin:

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

## Yeni bir yetenek ekleme

Bir plugin mevcut API'ye uymayan davranışa ihtiyaç duyduğunda, plugin sistemini özel bir iç erişimle baypas etmeyin.
Eksik yeteneği ekleyin.

Önerilen sıra:

1. çekirdek sözleşmesini tanımlayın
   Çekirdeğin hangi paylaşılan davranışa sahip olması gerektiğine karar verin: politika, geri dönüş, yapılandırma birleştirme,
   yaşam döngüsü, kanala dönük anlamlar ve çalışma zamanı yardımcı biçimi.
2. yazımlı plugin kayıt/çalışma zamanı yüzeyleri ekleyin
   En küçük yararlı
   yazımlı yetenek yüzeyiyle `OpenClawPluginApi` ve/veya `api.runtime` öğesini genişletin.
3. çekirdek + kanal/özellik tüketicilerini bağlayın
   Kanal ve özellik plugin'leri yeni yeteneği bir satıcı uygulamasını doğrudan içe aktararak değil,
   çekirdek üzerinden tüketmelidir.
4. satıcı uygulamalarını kaydedin
   Satıcı plugin'leri ardından arka uçlarını bu yeteneğe kaydeder.
5. sözleşme kapsamı ekleyin
   Sahiplik ve kayıt biçimi zaman içinde açık kalsın diye testler ekleyin.

OpenClaw bu şekilde tek bir
sağlayıcının dünya görüşüne sabitlenmeden görüş sahibi kalır. Somut bir dosya kontrol listesi ve
işlenmiş örnek için [Capability Cookbook](/tr/plugins/architecture) sayfasına bakın.

### Yetenek kontrol listesi

Yeni bir yetenek eklediğinizde uygulama genellikle bu
yüzeylere birlikte dokunmalıdır:

- `src/<capability>/types.ts` içindeki çekirdek sözleşme türleri
- `src/<capability>/runtime.ts` içindeki çekirdek çalıştırıcı/çalışma zamanı yardımcısı
- `src/plugins/types.ts` içindeki plugin API kayıt yüzeyi
- `src/plugins/registry.ts` içindeki plugin kayıt sistemi bağlaması
- özellik/kanal plugin'lerinin bunu tüketmesi gerektiğinde `src/plugins/runtime/*` içindeki plugin çalışma zamanı açığa çıkarımı
- `src/test-utils/plugin-registration.ts` içindeki yakalama/test yardımcıları
- `src/plugins/contracts/registry.ts` içindeki sahiplik/sözleşme doğrulamaları
- `docs/` içindeki operatör/plugin belgeleri

Bu yüzeylerden biri eksikse, bu genellikle yeteneğin henüz tam
entegre edilmediğinin işaretidir.

### Yetenek şablonu

Asgari desen:

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

Sözleşme test deseni:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Bu, kuralı basit tutar:

- çekirdek yetenek sözleşmesine + orkestrasyona sahiptir
- satıcı plugin'leri satıcı uygulamalarına sahiptir
- özellik/kanal plugin'leri çalışma zamanı yardımcılarını tüketir
- sözleşme testleri sahipliği açık tutar
