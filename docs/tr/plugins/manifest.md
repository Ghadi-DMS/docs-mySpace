---
read_when:
    - Bir OpenClaw plugin'i geliştiriyorsunuz
    - Bir plugin yapılandırma şeması yayımlamanız veya plugin doğrulama hatalarında hata ayıklamanız gerekiyor
summary: Plugin manifesti + JSON Schema gereksinimleri (katı yapılandırma doğrulaması)
title: Plugin Manifesti
x-i18n:
    generated_at: "2026-04-09T01:29:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9a7ee4b621a801d2a8f32f8976b0e1d9433c7810eb360aca466031fc0ffb286a
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin manifesti (openclaw.plugin.json)

Bu sayfa yalnızca **yerel OpenClaw plugin manifesti** içindir.

Uyumlu paket düzenleri için bkz. [Plugin paketleri](/tr/plugins/bundles).

Uyumlu paket biçimleri farklı manifest dosyaları kullanır:

- Codex paketi: `.codex-plugin/plugin.json`
- Claude paketi: `.claude-plugin/plugin.json` veya manifestsiz varsayılan Claude bileşen
  düzeni
- Cursor paketi: `.cursor-plugin/plugin.json`

OpenClaw bu paket düzenlerini de otomatik algılar, ancak bunlar burada
açıklanan `openclaw.plugin.json` şemasına göre doğrulanmaz.

Uyumlu paketler için OpenClaw şu anda düzen OpenClaw çalışma zamanı
beklentileriyle eşleştiğinde paket meta verilerini, bildirilmiş skill
köklerini, Claude komut köklerini, Claude paketi `settings.json`
varsayılanlarını, Claude paketi LSP varsayılanlarını ve desteklenen hook
paketlerini okur.

Her yerel OpenClaw plugin'i, **plugin kökünde** bir `openclaw.plugin.json`
dosyası **göndermelidir**. OpenClaw bu manifesti, **plugin kodunu çalıştırmadan**
yapılandırmayı doğrulamak için kullanır. Eksik veya geçersiz manifestler plugin
hatası olarak değerlendirilir ve yapılandırma doğrulamasını engeller.

Tam plugin sistemi kılavuzu için bkz.: [Plugins](/tr/tools/plugin).
Yerel yetenek modeli ve güncel dış uyumluluk kılavuzu için:
[Yetenek modeli](/tr/plugins/architecture#public-capability-model).

## Bu dosya ne yapar

`openclaw.plugin.json`, OpenClaw’ın plugin kodunuzu yüklemeden önce okuduğu
meta verilerdir.

Bunu şunlar için kullanın:

- plugin kimliği
- yapılandırma doğrulaması
- plugin çalışma zamanını başlatmadan erişilebilir olması gereken auth ve onboarding meta verileri
- plugin çalışma zamanı yüklenmeden önce çözümlenmesi gereken takma ad ve otomatik etkinleştirme meta verileri
- çalışma zamanı yüklenmeden önce plugin'i otomatik etkinleştirmesi gereken kısa model ailesi sahipliği meta verileri
- paketlenmiş uyumluluk bağlantıları ve sözleşme kapsamı için kullanılan statik yetenek sahipliği anlık görüntüleri
- çalışma zamanı yüklenmeden katalog ve doğrulama yüzeylerine birleştirilmesi gereken kanala özgü yapılandırma meta verileri
- yapılandırma arayüzü ipuçları

Bunu şunlar için kullanmayın:

- çalışma zamanı davranışı kaydetme
- kod giriş noktaları bildirme
- npm kurulum meta verileri

Bunlar plugin kodunuz ve `package.json` içine aittir.

## Asgari örnek

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Zengin örnek

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter sağlayıcı plugin'i",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API anahtarı",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API anahtarı",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API anahtarı",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Üst düzey alan başvurusu

| Alan                                | Gerekli | Tür                              | Anlamı                                                                                                                                                                                                      |
| ----------------------------------- | ------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | Evet    | `string`                         | Kanonik plugin kimliği. `plugins.entries.<id>` içinde kullanılan kimlik budur.                                                                                                                             |
| `configSchema`                      | Evet    | `object`                         | Bu plugin'in yapılandırması için satır içi JSON Schema.                                                                                                                                                     |
| `enabledByDefault`                  | Hayır   | `true`                           | Paketlenmiş bir plugin'in varsayılan olarak etkin olduğunu işaretler. Plugin'i varsayılan olarak devre dışı bırakmak için bunu atlayın veya `true` dışındaki herhangi bir değere ayarlayın.            |
| `legacyPluginIds`                   | Hayır   | `string[]`                       | Bu kanonik plugin kimliğine normalize edilen eski kimlikler.                                                                                                                                               |
| `autoEnableWhenConfiguredProviders` | Hayır   | `string[]`                       | Auth, yapılandırma veya model başvuruları bunlardan söz ettiğinde bu plugin'i otomatik etkinleştirmesi gereken sağlayıcı kimlikleri.                                                                     |
| `kind`                              | Hayır   | `"memory"` \| `"context-engine"` | `plugins.slots.*` tarafından kullanılan özel bir plugin türü bildirir.                                                                                                                                     |
| `channels`                          | Hayır   | `string[]`                       | Bu plugin'in sahip olduğu kanal kimlikleri. Keşif ve yapılandırma doğrulaması için kullanılır.                                                                                                            |
| `providers`                         | Hayır   | `string[]`                       | Bu plugin'in sahip olduğu sağlayıcı kimlikleri.                                                                                                                                                            |
| `modelSupport`                      | Hayır   | `object`                         | Çalışma zamanından önce plugin'i otomatik yüklemek için kullanılan, manifeste ait kısa model ailesi meta verileri.                                                                                        |
| `cliBackends`                       | Hayır   | `string[]`                       | Bu plugin'in sahip olduğu CLI çıkarım arka uç kimlikleri. Açık yapılandırma başvurularından başlangıçta otomatik etkinleştirme için kullanılır.                                                         |
| `providerAuthEnvVars`               | Hayır   | `Record<string, string[]>`       | OpenClaw’ın plugin kodunu yüklemeden inceleyebileceği düşük maliyetli sağlayıcı auth ortam değişkeni meta verileri.                                                                                      |
| `providerAuthAliases`               | Hayır   | `Record<string, string>`         | Auth araması için başka bir sağlayıcı kimliğini yeniden kullanması gereken sağlayıcı kimlikleri; örneğin temel sağlayıcının API anahtarını ve auth profillerini paylaşan bir kodlama sağlayıcısı gibi. |
| `channelEnvVars`                    | Hayır   | `Record<string, string[]>`       | OpenClaw’ın plugin kodunu yüklemeden inceleyebileceği düşük maliyetli kanal ortam değişkeni meta verileri. Bunu, genel başlangıç/yapılandırma yardımcılarının görmesi gereken env tabanlı kanal kurulumu veya auth yüzeyleri için kullanın. |
| `providerAuthChoices`               | Hayır   | `object[]`                       | Onboarding seçicileri, tercih edilen sağlayıcı çözümleme ve basit CLI bayrağı bağlantıları için düşük maliyetli auth-seçeneği meta verileri.                                                            |
| `contracts`                         | Hayır   | `object`                         | Konuşma, gerçek zamanlı transkripsiyon, gerçek zamanlı ses, medya anlama, görsel üretimi, müzik üretimi, video üretimi, web getirme, web araması ve araç sahipliği için statik paketlenmiş yetenek anlık görüntüsü. |
| `channelConfigs`                    | Hayır   | `Record<string, object>`         | Çalışma zamanı yüklenmeden önce keşif ve doğrulama yüzeylerine birleştirilen, manifeste ait kanal yapılandırma meta verileri.                                                                            |
| `skills`                            | Hayır   | `string[]`                       | Plugin köküne göre göreli yüklenecek Skills dizinleri.                                                                                                                                                     |
| `name`                              | Hayır   | `string`                         | İnsan tarafından okunabilir plugin adı.                                                                                                                                                                    |
| `description`                       | Hayır   | `string`                         | Plugin yüzeylerinde gösterilen kısa özet.                                                                                                                                                                  |
| `version`                           | Hayır   | `string`                         | Bilgilendirici plugin sürümü.                                                                                                                                                                              |
| `uiHints`                           | Hayır   | `Record<string, object>`         | Yapılandırma alanları için arayüz etiketleri, yer tutucular ve hassasiyet ipuçları.                                                                                                                       |

## providerAuthChoices başvurusu

Her `providerAuthChoices` girişi bir onboarding veya auth seçeneğini açıklar.
OpenClaw bunu sağlayıcı çalışma zamanı yüklenmeden önce okur.

| Alan                  | Gerekli | Tür                                             | Anlamı                                                                                                 |
| --------------------- | ------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `provider`            | Evet    | `string`                                        | Bu seçeneğin ait olduğu sağlayıcı kimliği.                                                             |
| `method`              | Evet    | `string`                                        | Yönlendirme yapılacak auth yöntemi kimliği.                                                            |
| `choiceId`            | Evet    | `string`                                        | Onboarding ve CLI akışlarında kullanılan kararlı auth-seçeneği kimliği.                               |
| `choiceLabel`         | Hayır   | `string`                                        | Kullanıcıya dönük etiket. Atlanırsa OpenClaw `choiceId` değerine geri döner.                          |
| `choiceHint`          | Hayır   | `string`                                        | Seçici için kısa yardımcı metin.                                                                       |
| `assistantPriority`   | Hayır   | `number`                                        | Assistant odaklı etkileşimli seçicilerde daha düşük değerler daha önce sıralanır.                     |
| `assistantVisibility` | Hayır   | `"visible"` \| `"manual-only"`                  | Seçeneği assistant seçicilerinden gizler, ancak elle CLI seçimine yine de izin verir.                 |
| `deprecatedChoiceIds` | Hayır   | `string[]`                                      | Kullanıcıları bu yedek seçeneğe yönlendirmesi gereken eski seçenek kimlikleri.                        |
| `groupId`             | Hayır   | `string`                                        | İlgili seçenekleri gruplamak için isteğe bağlı grup kimliği.                                           |
| `groupLabel`          | Hayır   | `string`                                        | Bu grup için kullanıcıya dönük etiket.                                                                 |
| `groupHint`           | Hayır   | `string`                                        | Grup için kısa yardımcı metin.                                                                          |
| `optionKey`           | Hayır   | `string`                                        | Basit tek bayraklı auth akışları için dahili seçenek anahtarı.                                        |
| `cliFlag`             | Hayır   | `string`                                        | `--openrouter-api-key` gibi CLI bayrak adı.                                                            |
| `cliOption`           | Hayır   | `string`                                        | `--openrouter-api-key <key>` gibi tam CLI seçenek biçimi.                                              |
| `cliDescription`      | Hayır   | `string`                                        | CLI yardımında kullanılan açıklama.                                                                    |
| `onboardingScopes`    | Hayır   | `Array<"text-inference" \| "image-generation">` | Bu seçeneğin hangi onboarding yüzeylerinde görünmesi gerektiği. Atlanırsa varsayılan olarak `["text-inference"]` kullanılır. |

## uiHints başvurusu

`uiHints`, yapılandırma alanı adlarından küçük işleme ipuçlarına giden bir eşlemedir.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API anahtarı",
      "help": "OpenRouter istekleri için kullanılır",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Her alan ipucu şunları içerebilir:

| Alan          | Tür        | Anlamı                                 |
| ------------- | ---------- | -------------------------------------- |
| `label`       | `string`   | Kullanıcıya dönük alan etiketi.        |
| `help`        | `string`   | Kısa yardımcı metin.                   |
| `tags`        | `string[]` | İsteğe bağlı arayüz etiketleri.        |
| `advanced`    | `boolean`  | Alanı gelişmiş olarak işaretler.       |
| `sensitive`   | `boolean`  | Alanı gizli veya hassas olarak işaretler. |
| `placeholder` | `string`   | Form girdileri için yer tutucu metin.  |

## contracts başvurusu

`contracts` alanını yalnızca OpenClaw’ın plugin çalışma zamanını içe aktarmadan
okuyabildiği statik yetenek sahipliği meta verileri için kullanın.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Her liste isteğe bağlıdır:

| Alan                             | Tür        | Anlamı                                                          |
| -------------------------------- | ---------- | --------------------------------------------------------------- |
| `speechProviders`                | `string[]` | Bu plugin'in sahip olduğu konuşma sağlayıcısı kimlikleri.       |
| `realtimeTranscriptionProviders` | `string[]` | Bu plugin'in sahip olduğu gerçek zamanlı transkripsiyon sağlayıcısı kimlikleri. |
| `realtimeVoiceProviders`         | `string[]` | Bu plugin'in sahip olduğu gerçek zamanlı ses sağlayıcısı kimlikleri. |
| `mediaUnderstandingProviders`    | `string[]` | Bu plugin'in sahip olduğu medya anlama sağlayıcısı kimlikleri.  |
| `imageGenerationProviders`       | `string[]` | Bu plugin'in sahip olduğu görsel üretimi sağlayıcısı kimlikleri. |
| `videoGenerationProviders`       | `string[]` | Bu plugin'in sahip olduğu video üretimi sağlayıcısı kimlikleri. |
| `webFetchProviders`              | `string[]` | Bu plugin'in sahip olduğu web getirme sağlayıcısı kimlikleri.   |
| `webSearchProviders`             | `string[]` | Bu plugin'in sahip olduğu web araması sağlayıcısı kimlikleri.   |
| `tools`                          | `string[]` | Paketlenmiş sözleşme kontrolleri için bu plugin'in sahip olduğu aracı araç adları. |

## channelConfigs başvurusu

Bir kanal plugin'inin çalışma zamanı yüklenmeden önce düşük maliyetli
yapılandırma meta verilerine ihtiyacı olduğunda `channelConfigs` kullanın.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver bağlantısı",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Her kanal girdisi şunları içerebilir:

| Alan          | Tür                      | Anlamı                                                                                     |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------------ |
| `schema`      | `object`                 | `channels.<id>` için JSON Schema. Bildirilmiş her kanal yapılandırma girdisi için gereklidir. |
| `uiHints`     | `Record<string, object>` | Bu kanal yapılandırma bölümü için isteğe bağlı arayüz etiketleri/yer tutucular/hassasiyet ipuçları. |
| `label`       | `string`                 | Çalışma zamanı meta verileri hazır olmadığında seçici ve inceleme yüzeylerine birleştirilen kanal etiketi. |
| `description` | `string`                 | İnceleme ve katalog yüzeyleri için kısa kanal açıklaması.                                  |
| `preferOver`  | `string[]`               | Bu kanalın seçim yüzeylerinde üstüne çıkması gereken eski veya daha düşük öncelikli plugin kimlikleri. |

## modelSupport başvurusu

OpenClaw’ın plugin çalışma zamanı yüklenmeden önce `gpt-5.4` veya
`claude-sonnet-4.6` gibi kısa model kimliklerinden sağlayıcı plugin'inizi
çıkarım yoluyla belirlemesi gerektiğinde `modelSupport` kullanın.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw şu önceliği uygular:

- açık `provider/model` başvuruları, sahip olan `providers` manifest meta verilerini kullanır
- `modelPatterns`, `modelPrefixes` üzerinde önceliklidir
- hem paketlenmemiş bir plugin hem de paketlenmiş bir plugin eşleşirse paketlenmemiş plugin kazanır
- kalan belirsizlik, kullanıcı veya yapılandırma bir sağlayıcı belirtinceye kadar yok sayılır

Alanlar:

| Alan            | Tür        | Anlamı                                                                 |
| --------------- | ---------- | ---------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Kısa model kimliklerine karşı `startsWith` ile eşleşen önekler.        |
| `modelPatterns` | `string[]` | Profil soneki kaldırıldıktan sonra kısa model kimliklerine karşı eşleşen regex kaynakları. |

Eski üst düzey yetenek anahtarları kullanımdan kaldırılmıştır. `speechProviders`,
`realtimeTranscriptionProviders`, `realtimeVoiceProviders`,
`mediaUnderstandingProviders`, `imageGenerationProviders`,
`videoGenerationProviders`, `webFetchProviders` ve `webSearchProviders`
alanlarını `contracts` altına taşımak için `openclaw doctor --fix` kullanın;
normal manifest yükleme artık bu üst düzey alanları yetenek sahipliği olarak
değerlendirmez.

## Manifest ve package.json karşılaştırması

Bu iki dosya farklı işler yapar:

| Dosya                  | Kullanım amacı                                                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Keşif, yapılandırma doğrulaması, auth-seçeneği meta verileri ve plugin kodu çalışmadan önce var olması gereken arayüz ipuçları   |
| `package.json`         | npm meta verileri, bağımlılık kurulumu ve giriş noktaları, kurulum kapıları, kurulum veya katalog meta verileri için kullanılan `openclaw` bloğu |

Bir meta veri parçasının nereye ait olduğundan emin değilseniz şu kuralı kullanın:

- OpenClaw’ın bunu plugin kodunu yüklemeden önce bilmesi gerekiyorsa `openclaw.plugin.json` içine koyun
- paketleme, giriş dosyaları veya npm kurulum davranışıyla ilgiliyse `package.json` içine koyun

### Keşfi etkileyen package.json alanları

Bazı çalışma zamanı öncesi plugin meta verileri, kasıtlı olarak
`openclaw.plugin.json` yerine `package.json` içindeki `openclaw` bloğunda bulunur.

Önemli örnekler:

| Alan                                                              | Anlamı                                                                                                                                      |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Yerel plugin giriş noktalarını bildirir.                                                                                                    |
| `openclaw.setupEntry`                                             | Onboarding ve ertelenmiş kanal başlangıcı sırasında kullanılan hafif, yalnızca kurulum amaçlı giriş noktası.                               |
| `openclaw.channel`                                                | Etiketler, belge yolları, takma adlar ve seçim metni gibi düşük maliyetli kanal katalog meta verileri.                                     |
| `openclaw.channel.configuredState`                                | Tam kanal çalışma zamanını yüklemeden "yalnızca env tabanlı kurulum zaten var mı?" sorusunu yanıtlayabilen hafif yapılandırılmış durum denetleyicisi meta verileri. |
| `openclaw.channel.persistedAuthState`                             | Tam kanal çalışma zamanını yüklemeden "zaten oturum açılmış bir şey var mı?" sorusunu yanıtlayabilen hafif kalıcı auth durumu denetleyicisi meta verileri. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Paketlenmiş ve haricen yayımlanmış plugin'ler için kurulum/güncelleme ipuçları.                                                           |
| `openclaw.install.defaultChoice`                                  | Birden çok kurulum kaynağı mevcut olduğunda tercih edilen kurulum yolu.                                                                    |
| `openclaw.install.minHostVersion`                                 | `>=2026.3.22` gibi bir semver alt sınırı kullanan, desteklenen en düşük OpenClaw ana makine sürümü.                                       |
| `openclaw.install.allowInvalidConfigRecovery`                     | Yapılandırma geçersiz olduğunda dar kapsamlı bir paketlenmiş plugin yeniden kurulum kurtarma yoluna izin verir.                           |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Başlangıç sırasında tam kanal plugin'inden önce yalnızca kurulum amaçlı kanal yüzeylerinin yüklenmesine izin verir.                       |

`openclaw.install.minHostVersion`, kurulum ve manifest kayıt defteri yükleme
sırasında zorunlu kılınır. Geçersiz değerler reddedilir; daha yeni ama geçerli
değerler, daha eski ana makinelerde plugin'i atlar.

`openclaw.install.allowInvalidConfigRecovery` kasıtlı olarak dardır. Rastgele
bozuk yapılandırmaları kurulabilir hale getirmez. Bugün yalnızca eksik
paketlenmiş plugin yolu veya aynı paketlenmiş plugin için eski bir
`channels.<id>` girdisi gibi belirli eski paketlenmiş plugin yükseltme
başarısızlıklarından kurulum akışlarının kurtulmasına izin verir. İlgisiz
yapılandırma hataları yine de kurulumu engeller ve operatörleri
`openclaw doctor --fix` komutuna yönlendirir.

`openclaw.channel.persistedAuthState`, küçük bir denetleyici modülü için paket
meta verisidir:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Bunu, kurulum, doctor veya yapılandırılmış durum akışları tam kanal plugin'i
yüklenmeden önce düşük maliyetli bir evet/hayır auth sorgusuna ihtiyaç
duyduğunda kullanın. Hedef dışa aktarma yalnızca kalıcı durumu okuyan küçük bir
işlev olmalıdır; bunu tam kanal çalışma zamanı barrel'ı üzerinden yönlendirmeyin.

`openclaw.channel.configuredState`, düşük maliyetli yalnızca env tabanlı
yapılandırılmış durum kontrolleri için aynı biçimi izler:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Bunu, bir kanal yapılandırılmış durumu env veya başka küçük çalışma zamanı dışı
girdilerden yanıtlayabildiğinde kullanın. Kontrol tam yapılandırma çözümlemesi
veya gerçek kanal çalışma zamanına ihtiyaç duyuyorsa bu mantığı bunun yerine
plugin `config.hasConfiguredState` hook'u içinde tutun.

## JSON Schema gereksinimleri

- **Her plugin bir JSON Schema göndermelidir**, hiç yapılandırma kabul etmese bile.
- Boş bir şema kabul edilebilir (örneğin `{ "type": "object", "additionalProperties": false }`).
- Şemalar çalışma zamanında değil, yapılandırma okuma/yazma sırasında doğrulanır.

## Doğrulama davranışı

- Bilinmeyen `channels.*` anahtarları, kanal kimliği bir plugin manifesti
  tarafından bildirilmedikçe **hatadır**.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` ve `plugins.slots.*`,
  **keşfedilebilir** plugin kimliklerine başvurmalıdır. Bilinmeyen kimlikler
  **hatadır**.
- Bir plugin kuruluysa ancak kırık veya eksik bir manifest ya da şemaya sahipse,
  doğrulama başarısız olur ve Doctor plugin hatasını bildirir.
- Plugin yapılandırması varsa ancak plugin **devre dışıysa**, yapılandırma korunur
  ve Doctor + günlüklerde bir **uyarı** gösterilir.

Tam `plugins.*` şeması için bkz. [Yapılandırma başvurusu](/tr/gateway/configuration).

## Notlar

- Manifest, yerel dosya sistemi yüklemeleri dahil, **yerel OpenClaw plugin'leri için zorunludur**.
- Çalışma zamanı yine de plugin modülünü ayrı olarak yükler; manifest yalnızca
  keşif + doğrulama içindir.
- Yerel manifestler JSON5 ile ayrıştırılır; son değer hâlâ bir nesne olduğu
  sürece yorumlar, sondaki virgüller ve tırnaksız anahtarlar kabul edilir.
- Manifest yükleyicisi yalnızca belgelenmiş manifest alanlarını okur. Buraya
  özel üst düzey anahtarlar eklemekten kaçının.
- `providerAuthEnvVars`, env adlarını incelemek için plugin çalışma zamanını
  başlatmaması gereken auth sorguları, env işaretleyici doğrulaması ve benzer
  sağlayıcı auth yüzeyleri için düşük maliyetli meta veri yoludur.
- `providerAuthAliases`, sağlayıcı varyantlarının başka bir sağlayıcının auth
  ortam değişkenlerini, auth profillerini, yapılandırma destekli auth'unu ve API anahtarı onboarding seçeneğini bu ilişkiyi çekirdekte sabit kodlamadan yeniden kullanmasına izin verir.
- `channelEnvVars`, env adlarını incelemek için plugin çalışma zamanını
  başlatmaması gereken kabuk env geri dönüşü, kurulum istemleri ve benzer kanal
  yüzeyleri için düşük maliyetli meta veri yoludur.
- `providerAuthChoices`, auth-seçeneği seçicileri,
  `--auth-choice` çözümlemesi, tercih edilen sağlayıcı eşlemesi ve sağlayıcı
  çalışma zamanı yüklenmeden önce basit onboarding CLI bayrağı kaydı için düşük
  maliyetli meta veri yoludur. Sağlayıcı kodu gerektiren çalışma zamanı sihirbazı
  meta verileri için bkz.
  [Sağlayıcı çalışma zamanı hook'ları](/tr/plugins/architecture#provider-runtime-hooks).
- Özel plugin türleri `plugins.slots.*` üzerinden seçilir.
  - `kind: "memory"` değeri `plugins.slots.memory` tarafından seçilir.
  - `kind: "context-engine"` değeri `plugins.slots.contextEngine`
    tarafından seçilir (varsayılan: yerleşik `legacy`).
- `channels`, `providers`, `cliBackends` ve `skills`, bir plugin bunlara
  ihtiyaç duymuyorsa atlanabilir.
- Plugin'iniz yerel modüllere bağlıysa derleme adımlarını ve paket yöneticisi
  izin listesi gereksinimlerini belgeleyin (örneğin pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## İlgili

- [Plugin Geliştirme](/tr/plugins/building-plugins) — plugin'lere başlamaya giriş
- [Plugin Mimarisi](/tr/plugins/architecture) — iç mimari
- [SDK Genel Bakış](/tr/plugins/sdk-overview) — Plugin SDK başvurusu
