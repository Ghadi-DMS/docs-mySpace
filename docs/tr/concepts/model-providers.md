---
read_when:
    - Sağlayıcı bazında model kurulumu için bir başvuruya ihtiyacınız var
    - Model sağlayıcıları için örnek yapılandırmalar veya CLI onboarding komutları istiyorsunuz
summary: Örnek yapılandırmalar ve CLI akışlarıyla model sağlayıcısı genel bakışı
title: Model Sağlayıcıları
x-i18n:
    generated_at: "2026-04-09T01:29:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 53e3141256781002bbe1d7e7b78724a18d061fcf36a203baae04a091b8c9ea1b
    source_path: concepts/model-providers.md
    workflow: 15
---

# Model sağlayıcıları

Bu sayfa **LLM/model sağlayıcılarını** kapsar (WhatsApp/Telegram gibi sohbet kanallarını değil).
Model seçim kuralları için bkz. [/concepts/models](/tr/concepts/models).

## Hızlı kurallar

- Model başvuruları `provider/model` kullanır (örnek: `opencode/claude-opus-4-6`).
- `agents.defaults.models` ayarlarsanız, bu izin listesi olur.
- CLI yardımcıları: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Yedek çalışma zamanı kuralları, bekleme süresi probları ve oturum geçersiz kılma kalıcılığı
  [/concepts/model-failover](/tr/concepts/model-failover) içinde belgelenmiştir.
- `models.providers.*.models[].contextWindow` yerel model meta verisidir;
  `models.providers.*.models[].contextTokens` ise etkili çalışma zamanı sınırıdır.
- Sağlayıcı eklentileri `registerProvider({ catalog })` aracılığıyla model katalogları enjekte edebilir;
  OpenClaw bu çıktıyı `models.providers` içine birleştirir ve sonra
  `models.json` yazar.
- Sağlayıcı manifestleri `providerAuthEnvVars` ve
  `providerAuthAliases` bildirebilir; böylece genel env tabanlı auth probları ve sağlayıcı varyantlarının
  eklenti çalışma zamanını yüklemesi gerekmez. Kalan çekirdek env değişkeni haritası artık
  sadece eklenti olmayan/çekirdek sağlayıcılar ve Anthropic için API anahtarı öncelikli onboarding gibi
  birkaç genel öncelik durumu içindir.
- Sağlayıcı eklentileri sağlayıcı çalışma zamanı davranışını da
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot` ve
  `onModelSelected` aracılığıyla da yönetebilir.
- Not: sağlayıcı çalışma zamanı `capabilities` paylaşılan çalıştırıcı meta verisidir (sağlayıcı
  ailesi, transkript/araç kullanımına özgü durumlar, taşıma/önbellek ipuçları). Bu,
  bir eklentinin ne kaydettiğini açıklayan [genel yetenek modeli](/tr/plugins/architecture#public-capability-model)
  ile aynı şey değildir (metin çıkarımı, konuşma vb.).

## Eklentiye ait sağlayıcı davranışı

Sağlayıcı eklentileri artık sağlayıcıya özgü mantığın çoğunu sahiplenebilirken OpenClaw
genel çıkarım döngüsünü korur.

Tipik ayrım:

- `auth[].run` / `auth[].runNonInteractive`: sağlayıcı, `openclaw onboard`, `openclaw models auth` ve başsız kurulum
  için onboarding/giriş akışlarını sahiplenir
- `wizard.setup` / `wizard.modelPicker`: sağlayıcı auth-seçimi etiketlerini,
  eski takma adları, onboarding izin listesi ipuçlarını ve onboarding/model seçicilerindeki kurulum girdilerini sahiplenir
- `catalog`: sağlayıcı `models.providers` içinde görünür
- `normalizeModelId`: sağlayıcı, arama veya kanonikleştirme öncesinde eski/önizleme model kimliklerini
  normalize eder
- `normalizeTransport`: sağlayıcı, genel model derlemesinden önce taşıma ailesi `api` / `baseUrl`
  değerlerini normalize eder; OpenClaw önce eşleşen sağlayıcıyı,
  ardından yalnızca değişiklik yapan birini bulana kadar kanca destekli diğer sağlayıcı eklentilerini kontrol eder
- `normalizeConfig`: sağlayıcı, çalışma zamanı kullanmadan önce `models.providers.<id>` yapılandırmasını
  normalize eder; OpenClaw önce eşleşen sağlayıcıyı, ardından
  yalnızca yapılandırmayı gerçekten değiştiren birini bulana kadar diğer kanca destekli sağlayıcı eklentilerini kontrol eder. Eğer
  hiçbir sağlayıcı kancası yapılandırmayı yeniden yazmazsa, paketlenmiş Google ailesi yardımcıları yine de
  desteklenen Google sağlayıcı girdilerini normalize eder.
- `applyNativeStreamingUsageCompat`: sağlayıcı, yapılandırma sağlayıcıları için uç nokta kaynaklı yerel akış-kullanımı uyumluluk yeniden yazımlarını uygular
- `resolveConfigApiKey`: sağlayıcı, tam çalışma zamanı auth yüklemesini zorlamadan
  yapılandırma sağlayıcıları için env işaretleyici auth çözer.
  `amazon-bedrock` ayrıca burada yerleşik bir AWS env işaretleyici çözücüye de sahiptir, ancak Bedrock çalışma zamanı auth
  AWS SDK varsayılan zincirini kullanır.
- `resolveSyntheticAuth`: sağlayıcı, düz metin sırları kalıcılaştırmadan
  yerel/kendi barındırılan veya diğer yapılandırma destekli auth kullanılabilirliğini açığa çıkarabilir
- `shouldDeferSyntheticProfileAuth`: sağlayıcı, saklanan sentetik profil
  yer tutucularını env/yapılandırma destekli auth'tan daha düşük öncelikli olarak işaretleyebilir
- `resolveDynamicModel`: sağlayıcı, henüz yerel
  statik katalogda bulunmayan model kimliklerini kabul eder
- `prepareDynamicModel`: sağlayıcının, dinamik çözümlemeyi yeniden denemeden önce
  meta veri yenilemesine ihtiyacı vardır
- `normalizeResolvedModel`: sağlayıcının taşıma veya temel URL yeniden yazımlarına ihtiyacı vardır
- `contributeResolvedModelCompat`: sağlayıcı, başka bir uyumlu taşıma üzerinden gelseler bile
  kendi üretici modelleri için uyumluluk işaretleri sağlar
- `capabilities`: sağlayıcı transkript/araç kullanımı/sağlayıcı ailesiyle ilgili özel durumları yayımlar
- `normalizeToolSchemas`: sağlayıcı, gömülü çalıştırıcı görmeden önce
  araç şemalarını temizler
- `inspectToolSchemas`: sağlayıcı, normalleştirmeden sonra taşıma türüne özgü şema uyarılarını
  yüzeye çıkarır
- `resolveReasoningOutputMode`: sağlayıcı, yerel ve etiketli
  akıl yürütme çıktısı sözleşmeleri arasında seçim yapar
- `prepareExtraParams`: sağlayıcı, model başına istek parametrelerini varsayılanlar veya normalize eder
- `createStreamFn`: sağlayıcı, normal akış yolunu tamamen
  özel bir taşıma ile değiştirir
- `wrapStreamFn`: sağlayıcı, istek başlık/gövde/model uyumluluk sarmalayıcılarını uygular
- `resolveTransportTurnState`: sağlayıcı, tur başına yerel taşıma
  başlıklarını veya meta verisini sağlar
- `resolveWebSocketSessionPolicy`: sağlayıcı, yerel WebSocket oturumu
  başlıklarını veya oturum bekleme politikası sağlar
- `createEmbeddingProvider`: sağlayıcı, bellek gömme davranışını,
  çekirdek gömme anahtarlayıcısı yerine sağlayıcı eklentisine ait olduğunda sahiplenir
- `formatApiKey`: sağlayıcı, saklanan auth profillerini,
  taşımanın beklediği çalışma zamanı `apiKey` dizgesine biçimlendirir
- `refreshOAuth`: paylaşılan `pi-ai`
  yenileyicileri yeterli olmadığında OAuth yenilemeyi sağlayıcı sahiplenir
- `buildAuthDoctorHint`: OAuth yenileme
  başarısız olduğunda sağlayıcı onarım rehberliği ekler
- `matchesContextOverflowError`: sağlayıcı, genel sezgilerin kaçıracağı
  sağlayıcıya özgü bağlam penceresi taşma hatalarını tanır
- `classifyFailoverReason`: sağlayıcı, sağlayıcıya özgü ham taşıma/API
  hatalarını oran sınırı veya aşırı yük gibi yedekleme nedenlerine eşler
- `isCacheTtlEligible`: sağlayıcı, hangi yukarı akış model kimliklerinin istem önbelleği TTL'yi desteklediğine karar verir
- `buildMissingAuthMessage`: sağlayıcı, genel auth-store hatasını
  sağlayıcıya özgü bir kurtarma ipucuyla değiştirir
- `suppressBuiltInModel`: sağlayıcı, eski yukarı akış satırlarını gizler ve
  doğrudan çözümleme başarısızlıkları için üreticiye ait bir hata döndürebilir
- `augmentModelCatalog`: sağlayıcı, keşif ve yapılandırma birleştirmesinden sonra
  sentetik/nihai katalog satırları ekler
- `isBinaryThinking`: sağlayıcı, ikili açık/kapalı düşünme UX'ini sahiplenir
- `supportsXHighThinking`: sağlayıcı, seçili modelleri `xhigh` için etkinleştirir
- `resolveDefaultThinkingLevel`: sağlayıcı, bir
  model ailesi için varsayılan `/think` politikasını sahiplenir
- `applyConfigDefaults`: sağlayıcı, auth modu, env veya model ailesine göre
  yapılandırma somutlaştırma sırasında sağlayıcıya özgü genel varsayılanları uygular
- `isModernModelRef`: sağlayıcı, canlı/smoke tercih edilen model eşlemesini sahiplenir
- `prepareRuntimeAuth`: sağlayıcı, yapılandırılmış bir kimlik bilgisini kısa ömürlü
  bir çalışma zamanı belirtecine dönüştürür
- `resolveUsageAuth`: sağlayıcı, `/usage`
  ve ilgili durum/raporlama yüzeyleri için kullanım/kota kimlik bilgilerini çözer
- `fetchUsageSnapshot`: sağlayıcı, kullanım uç noktası getirme/ayrıştırma işlemini sahiplenirken
  çekirdek yine özet kabuğunu ve biçimlendirmeyi sahiplenir
- `onModelSelected`: sağlayıcı, telemetri veya sağlayıcıya ait oturum kayıt tutma gibi
  seçim sonrası yan etkileri çalıştırır

Mevcut paketlenmiş örnekler:

- `anthropic`: Claude 4.6 ileri uyumluluk yedeği, auth onarım ipuçları, kullanım
  uç noktası getirme, önbellek-TTL/sağlayıcı ailesi meta verisi ve auth farkındalıklı genel
  yapılandırma varsayılanları
- `amazon-bedrock`: Bedrock'a özgü kısma/hazır değil hataları için sağlayıcıya ait bağlam taşması eşleştirme ve
  yedekleme nedeni sınıflandırması, ayrıca
  Anthropic trafiğinde yalnızca Claude tekrar oynatma politikası korumaları için paylaşılan `anthropic-by-model` yeniden oynatma ailesi
- `anthropic-vertex`: Anthropic-messages
  trafiğinde yalnızca Claude tekrar oynatma politikası korumaları
- `openrouter`: doğrudan model kimlikleri, istek sarmalayıcıları, sağlayıcı yetenek
  ipuçları, proxy Gemini trafiğinde Gemini thought-signature temizleme,
  `openrouter-thinking` akış ailesi üzerinden proxy akıl yürütme enjeksiyonu, yönlendirme
  meta verisi iletimi ve önbellek-TTL politikası
- `github-copilot`: onboarding/cihaz girişi, ileri uyumluluk model yedeği,
  Claude düşünme transkript ipuçları, çalışma zamanı belirteci değişimi ve kullanım uç noktası
  getirme
- `openai`: GPT-5.4 ileri uyumluluk yedeği, doğrudan OpenAI taşıma
  normalleştirmesi, Codex farkındalıklı eksik auth ipuçları, Spark bastırma, sentetik
  OpenAI/Codex katalog satırları, düşünme/canlı model politikası, kullanım belirteci takma ad
  normalleştirmesi (`input` / `output` ve `prompt` / `completion` aileleri), yerel OpenAI/Codex
  sarmalayıcıları için paylaşılan `openai-responses-defaults` akış ailesi,
  sağlayıcı ailesi meta verisi, `gpt-image-1` için paketlenmiş görüntü oluşturma sağlayıcısı
  kaydı ve `sora-2` için paketlenmiş video oluşturma sağlayıcısı
  kaydı
- `google` ve `google-gemini-cli`: Gemini 3.1 ileri uyumluluk yedeği,
  yerel Gemini tekrar oynatma doğrulaması, önyükleme tekrar oynatma temizliği, etiketli
  akıl yürütme çıktısı modu, modern model eşleştirme, Gemini image-preview modelleri için paketlenmiş görüntü oluşturma
  sağlayıcısı kaydı ve Veo modelleri için paketlenmiş
  video oluşturma sağlayıcısı kaydı; Gemini CLI OAuth ayrıca
  kullanım yüzeyleri için auth profili belirteç biçimlendirmesi, kullanım belirteci ayrıştırması ve kota uç noktası
  getirmeyi de sahiplenir
- `moonshot`: paylaşılan taşıma, eklentiye ait düşünme yükü normalleştirmesi
- `kilocode`: paylaşılan taşıma, eklentiye ait istek başlıkları, akıl yürütme yükü
  normalleştirmesi, proxy-Gemini thought-signature temizliği ve önbellek-TTL
  politikası
- `zai`: GLM-5 ileri uyumluluk yedeği, `tool_stream` varsayılanları, önbellek-TTL
  politikası, ikili düşünme/canlı model politikası ve kullanım auth + kota getirme;
  bilinmeyen `glm-5*` kimlikleri paketlenmiş `glm-4.7` şablonundan sentezlenir
- `xai`: yerel Responses taşıma normalleştirmesi, Grok hızlı varyantları için
  `/fast` takma ad yeniden yazımları, varsayılan `tool_stream`, xAI'ya özgü araç şeması /
  akıl yürütme yükü temizliği ve `grok-imagine-video` için paketlenmiş video oluşturma sağlayıcısı
  kaydı
- `mistral`: eklentiye ait yetenek meta verisi
- `opencode` ve `opencode-go`: eklentiye ait yetenek meta verisi artı
  proxy-Gemini thought-signature temizliği
- `alibaba`: `alibaba/wan2.6-t2v`
  gibi doğrudan Wan model başvuruları için eklentiye ait video oluşturma kataloğu
- `byteplus`: eklentiye ait kataloglar artı Seedance metinden videoya/görüntüden videoya modelleri için
  paketlenmiş video oluşturma sağlayıcısı
  kaydı
- `fal`: barındırılan üçüncü taraf
  video modelleri için paketlenmiş video oluşturma sağlayıcısı kaydı artı FLUX görüntü modelleri için barındırılan üçüncü taraf
  görüntü oluşturma sağlayıcısı kaydı
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` ve `volcengine`:
  yalnızca eklentiye ait kataloglar
- `qwen`: metin modelleri için eklentiye ait kataloglar artı
  çok modlu yüzeyleri için paylaşılan
  medya anlama ve video oluşturma sağlayıcısı kayıtları; Qwen video oluşturma Standard DashScope video
  uç noktalarını ve `wan2.6-t2v` ile `wan2.7-r2v` gibi paketlenmiş Wan modellerini kullanır
- `runway`: `gen4.5` gibi yerel
  Runway görev tabanlı modeller için eklentiye ait video oluşturma sağlayıcısı kaydı
- `minimax`: eklentiye ait kataloglar, Hailuo video modelleri için paketlenmiş video oluşturma sağlayıcısı
  kaydı, `image-01` için paketlenmiş görüntü oluşturma sağlayıcısı
  kaydı, hibrit Anthropic/OpenAI tekrar oynatma politikası
  seçimi ve kullanım auth/anlık görüntü mantığı
- `together`: eklentiye ait kataloglar artı Wan video modelleri için paketlenmiş video oluşturma sağlayıcısı
  kaydı
- `xiaomi`: eklentiye ait kataloglar artı kullanım auth/anlık görüntü mantığı

Paketlenmiş `openai` eklentisi artık her iki sağlayıcı kimliğini de sahipleniyor: `openai` ve
`openai-codex`.

Bu, hâlâ OpenClaw'ın normal taşıma yollarına uyan sağlayıcıları kapsar. Tamamen
özel bir istek yürütücüsüne ihtiyaç duyan bir sağlayıcı ayrı ve daha derin bir uzantı yüzeyidir.

## API anahtarı döndürme

- Seçili sağlayıcılar için genel sağlayıcı döndürmeyi destekler.
- Birden fazla anahtarı şu yollarla yapılandırın:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (tek canlı geçersiz kılma, en yüksek öncelik)
  - `<PROVIDER>_API_KEYS` (virgül veya noktalı virgülle ayrılmış liste)
  - `<PROVIDER>_API_KEY` (birincil anahtar)
  - `<PROVIDER>_API_KEY_*` (numaralı liste, ör. `<PROVIDER>_API_KEY_1`)
- Google sağlayıcıları için `GOOGLE_API_KEY` de yedek olarak dahildir.
- Anahtar seçim sırası önceliği korur ve değerleri tekilleştirir.
- İstekler yalnızca oran sınırlama yanıtlarında sonraki anahtarla yeniden denenir (örneğin
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` veya dönemsel kullanım sınırı iletileri).
- Oran sınırı dışındaki hatalar hemen başarısız olur; anahtar döndürme denenmez.
- Tüm aday anahtarlar başarısız olduğunda, son hata son denemeden döndürülür.

## Yerleşik sağlayıcılar (pi-ai kataloğu)

OpenClaw, pi‑ai kataloğuyla gelir. Bu sağlayıcılar **hiç**
`models.providers` yapılandırması gerektirmez; yalnızca auth ayarlayın ve bir model seçin.

### OpenAI

- Sağlayıcı: `openai`
- Auth: `OPENAI_API_KEY`
- İsteğe bağlı döndürme: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, ayrıca `OPENCLAW_LIVE_OPENAI_KEY` (tek geçersiz kılma)
- Örnek modeller: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Varsayılan taşıma `auto`'dur (önce WebSocket, sonra SSE yedeği)
- Model başına geçersiz kılmak için `agents.defaults.models["openai/<model>"].params.transport` kullanın (`"sse"`, `"websocket"` veya `"auto"`)
- OpenAI Responses WebSocket ısınması varsayılan olarak `params.openaiWsWarmup` ile etkindir (`true`/`false`)
- OpenAI öncelikli işleme `agents.defaults.models["openai/<model>"].params.serviceTier` ile etkinleştirilebilir
- `/fast` ve `params.fastMode`, doğrudan `openai/*` Responses isteklerini `api.openai.com` üzerinde `service_tier=priority` olarak eşler
- Paylaşılan `/fast` geçişi yerine açık bir katman istediğinizde `params.serviceTier` kullanın
- Gizli OpenClaw atıf başlıkları (`originator`, `version`,
  `User-Agent`) yalnızca `api.openai.com` adresine giden yerel OpenAI trafiğinde uygulanır,
  genel OpenAI uyumlu proxy'lerde uygulanmaz
- Yerel OpenAI rotaları ayrıca Responses `store`, istem önbelleği ipuçları ve
  OpenAI akıl yürütme uyumluluğu yük şekillendirmesini korur; proxy rotaları bunu yapmaz
- `openai/gpt-5.3-codex-spark` OpenClaw'da kasıtlı olarak bastırılmıştır çünkü canlı OpenAI API bunu reddeder; Spark yalnızca Codex olarak değerlendirilir

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Sağlayıcı: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- İsteğe bağlı döndürme: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, ayrıca `OPENCLAW_LIVE_ANTHROPIC_KEY` (tek geçersiz kılma)
- Örnek model: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Doğrudan genel Anthropic istekleri, `api.anthropic.com` adresine gönderilen API anahtarlı ve OAuth kimlik doğrulamalı trafik dahil olmak üzere, paylaşılan `/fast` geçişini ve `params.fastMode` ayarını destekler; OpenClaw bunu Anthropic `service_tier` değerine eşler (`auto` ve `standard_only`)
- Anthropic notu: Anthropic çalışanları bize OpenClaw tarzı Claude CLI kullanımına yeniden izin verildiğini söyledi, bu nedenle Anthropic yeni bir politika yayımlamadığı sürece OpenClaw bu entegrasyon için Claude CLI yeniden kullanımını ve `claude -p` kullanımını onaylı kabul eder.
- Anthropic kurulum belirteci desteklenen bir OpenClaw belirteç yolu olarak hâlâ kullanılabilir, ancak OpenClaw artık mümkün olduğunda Claude CLI yeniden kullanımını ve `claude -p` kullanımını tercih eder.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Sağlayıcı: `openai-codex`
- Auth: OAuth (ChatGPT)
- Örnek model: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` veya `openclaw models auth login --provider openai-codex`
- Varsayılan taşıma `auto`'dur (önce WebSocket, sonra SSE yedeği)
- Model başına geçersiz kılmak için `agents.defaults.models["openai-codex/<model>"].params.transport` kullanın (`"sse"`, `"websocket"` veya `"auto"`)
- `params.serviceTier` yerel Codex Responses isteklerinde (`chatgpt.com/backend-api`) de iletilir
- Gizli OpenClaw atıf başlıkları (`originator`, `version`,
  `User-Agent`) yalnızca `chatgpt.com/backend-api` adresine giden yerel Codex trafiğine
  eklenir, genel OpenAI uyumlu proxy'lere eklenmez
- Doğrudan `openai/*` ile aynı `/fast` geçişini ve `params.fastMode` yapılandırmasını paylaşır; OpenClaw bunu `service_tier=priority` olarak eşler
- `openai-codex/gpt-5.3-codex-spark`, Codex OAuth kataloğu bunu sunduğunda kullanılabilir kalır; hakka bağlıdır
- `openai-codex/gpt-5.4`, yerel `contextWindow = 1050000` ve varsayılan çalışma zamanı `contextTokens = 272000` değerlerini korur; çalışma zamanı sınırını `models.providers.openai-codex.models[].contextTokens` ile geçersiz kılın
- Politika notu: OpenAI Codex OAuth, OpenClaw gibi harici araçlar/iş akışları için açıkça desteklenir.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Abonelik tarzı diğer barındırılan seçenekler

- [Qwen Cloud](/tr/providers/qwen): Qwen Cloud sağlayıcı yüzeyi artı Alibaba DashScope ve Coding Plan uç nokta eşlemesi
- [MiniMax](/tr/providers/minimax): MiniMax Coding Plan OAuth veya API anahtarı erişimi
- [GLM Models](/tr/providers/glm): Z.AI Coding Plan veya genel API uç noktaları

### OpenCode

- Auth: `OPENCODE_API_KEY` (veya `OPENCODE_ZEN_API_KEY`)
- Zen çalışma zamanı sağlayıcısı: `opencode`
- Go çalışma zamanı sağlayıcısı: `opencode-go`
- Örnek modeller: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` veya `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API anahtarı)

- Sağlayıcı: `google`
- Auth: `GEMINI_API_KEY`
- İsteğe bağlı döndürme: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY` yedeği ve `OPENCLAW_LIVE_GEMINI_KEY` (tek geçersiz kılma)
- Örnek modeller: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Uyumluluk: `google/gemini-3.1-flash-preview` kullanan eski OpenClaw yapılandırması `google/gemini-3-flash-preview` olarak normalize edilir
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Doğrudan Gemini çalıştırmaları ayrıca `agents.defaults.models["google/<model>"].params.cachedContent`
  (veya eski `cached_content`) değerini de kabul eder; bu değer sağlayıcıya özgü
  `cachedContents/...` tanıtıcısını iletir; Gemini önbellek isabetleri OpenClaw `cacheRead` olarak görünür

### Google Vertex ve Gemini CLI

- Sağlayıcılar: `google-vertex`, `google-gemini-cli`
- Auth: Vertex gcloud ADC kullanır; Gemini CLI kendi OAuth akışını kullanır
- Dikkat: OpenClaw içindeki Gemini CLI OAuth resmi olmayan bir entegrasyondur. Bazı kullanıcılar üçüncü taraf istemcileri kullandıktan sonra Google hesaplarında kısıtlamalar bildiriyor. Devam etmeyi seçerseniz Google şartlarını gözden geçirin ve kritik olmayan bir hesap kullanın.
- Gemini CLI OAuth, paketlenmiş `google` eklentisinin bir parçası olarak dağıtılır.
  - Önce Gemini CLI kurun:
    - `brew install gemini-cli`
    - veya `npm install -g @google/gemini-cli`
  - Etkinleştirme: `openclaw plugins enable google`
  - Giriş: `openclaw models auth login --provider google-gemini-cli --set-default`
  - Varsayılan model: `google-gemini-cli/gemini-3-flash-preview`
  - Not: `openclaw.json` içine bir istemci kimliği veya gizli anahtar yapıştırmazsınız. CLI giriş akışı
    belirteçleri ağ geçidi ana bilgisayarındaki auth profillerinde depolar.
  - Girişten sonra istekler başarısız olursa, ağ geçidi ana bilgisayarında `GOOGLE_CLOUD_PROJECT` veya `GOOGLE_CLOUD_PROJECT_ID` ayarlayın.
  - Gemini CLI JSON yanıtları `response` içinden ayrıştırılır; kullanım ise
    `stats` değerine yedeklenir, `stats.cached` ise OpenClaw `cacheRead` olarak normalize edilir.

### Z.AI (GLM)

- Sağlayıcı: `zai`
- Auth: `ZAI_API_KEY`
- Örnek model: `zai/glm-5.1`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Takma adlar: `z.ai/*` ve `z-ai/*`, `zai/*` olarak normalize edilir
  - `zai-api-key` eşleşen Z.AI uç noktasını otomatik algılar; `zai-coding-global`, `zai-coding-cn`, `zai-global` ve `zai-cn` belirli bir yüzeyi zorlar

### Vercel AI Gateway

- Sağlayıcı: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- Örnek model: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Sağlayıcı: `kilocode`
- Auth: `KILOCODE_API_KEY`
- Örnek model: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Temel URL: `https://api.kilo.ai/api/gateway/`
- Statik yedek katalog `kilocode/kilo/auto` ile gelir; canlı
  `https://api.kilo.ai/api/gateway/models` keşfi çalışma zamanı
  kataloğunu daha da genişletebilir.
- `kilocode/kilo/auto` arkasındaki tam yukarı akış yönlendirmesi Kilo Gateway'e aittir,
  OpenClaw içinde sabit kodlanmış değildir.

Kurulum ayrıntıları için bkz. [/providers/kilocode](/tr/providers/kilocode).

### Paketlenmiş diğer sağlayıcı eklentileri

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Örnek model: `openrouter/auto`
- OpenClaw, OpenRouter'ın belgelenmiş uygulama atıf başlıklarını yalnızca
  istek gerçekten `openrouter.ai` hedefine gidiyorsa uygular
- OpenRouter'a özgü Anthropic `cache_control` işaretçileri de benzer şekilde
  doğrulanmış OpenRouter rotalarıyla sınırlıdır, keyfi proxy URL'leriyle değil
- OpenRouter proxy tarzı OpenAI uyumlu yolda kalır; bu nedenle yerel
  yalnızca OpenAI'ya özgü istek şekillendirmesi (`serviceTier`, Responses `store`,
  istem önbelleği ipuçları, OpenAI akıl yürütme uyumluluğu yükleri) iletilmez
- Gemini destekli OpenRouter başvuruları yalnızca proxy-Gemini thought-signature temizliğini
  korur; yerel Gemini tekrar oynatma doğrulaması ve önyükleme yeniden yazımları devreye girmez
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Örnek model: `kilocode/kilo/auto`
- Gemini destekli Kilo başvuruları aynı proxy-Gemini thought-signature
  temizleme yolunu korur; `kilocode/kilo/auto` ve proxy akıl yürütmeyi desteklemeyen diğer
  ipuçları proxy akıl yürütme enjeksiyonunu atlar
- MiniMax: `minimax` (API anahtarı) ve `minimax-portal` (OAuth)
- Auth: `MINIMAX_API_KEY` for `minimax`; `MINIMAX_OAUTH_TOKEN` veya `MINIMAX_API_KEY` for `minimax-portal`
- Örnek model: `minimax/MiniMax-M2.7` veya `minimax-portal/MiniMax-M2.7`
- MiniMax onboarding/API anahtarı kurulumu, açık M2.7 model tanımlarını
  `input: ["text", "image"]` ile yazar; paketlenmiş sağlayıcı kataloğu sohbet başvurularını
  bu sağlayıcı yapılandırması somutlaştırılana kadar yalnızca metin olarak tutar
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Örnek model: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` veya `KIMICODE_API_KEY`)
- Örnek model: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Örnek model: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` veya `DASHSCOPE_API_KEY`)
- Örnek model: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Örnek model: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Örnek modeller: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Örnek model: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Örnek model: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` veya `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Örnek model: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Örnek model: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Yerel paketlenmiş xAI istekleri xAI Responses yolunu kullanır
  - `/fast` veya `params.fastMode: true`, `grok-3`, `grok-3-mini`,
    `grok-4` ve `grok-4-0709` modellerini `*-fast` varyantlarına yeniden yazar
  - `tool_stream` varsayılan olarak açıktır; devre dışı bırakmak için
    `agents.defaults.models["xai/<model>"].params.tool_stream` değerini `false`
    olarak ayarlayın
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Örnek model: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - Cerebras üzerindeki GLM modelleri `zai-glm-4.7` ve `zai-glm-4.6` kimliklerini kullanır.
  - OpenAI uyumlu temel URL: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Hugging Face Inference örnek modeli: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Bkz. [Hugging Face (Inference)](/tr/providers/huggingface).

## `models.providers` aracılığıyla sağlayıcılar (özel/temel URL)

**Özel** sağlayıcılar veya
OpenAI/Anthropic uyumlu proxy'ler eklemek için `models.providers` (veya `models.json`) kullanın.

Aşağıdaki paketlenmiş sağlayıcı eklentilerinin çoğu zaten varsayılan bir katalog yayımlar.
Varsayılan temel URL'yi, başlıkları veya model listesini geçersiz kılmak istediğinizde
yalnızca açık `models.providers.<id>` girdileri kullanın.

### Moonshot AI (Kimi)

Moonshot paketlenmiş bir sağlayıcı eklentisi olarak gelir. Varsayılan olarak
yerleşik sağlayıcıyı kullanın ve yalnızca temel URL'yi veya model meta verisini
geçersiz kılmanız gerektiğinde açık bir `models.providers.moonshot` girdisi ekleyin:

- Sağlayıcı: `moonshot`
- Auth: `MOONSHOT_API_KEY`
- Örnek model: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` veya `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi K2 model kimlikleri:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding, Moonshot AI'ın Anthropic uyumlu uç noktasını kullanır:

- Sağlayıcı: `kimi`
- Auth: `KIMI_API_KEY`
- Örnek model: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

Eski `kimi/k2p5`, uyumluluk modeli kimliği olarak kabul edilmeye devam eder.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎), Çin'de Doubao ve diğer modellere erişim sağlar.

- Sağlayıcı: `volcengine` (coding: `volcengine-plan`)
- Auth: `VOLCANO_ENGINE_API_KEY`
- Örnek model: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

Onboarding varsayılan olarak coding yüzeyini kullanır, ancak genel `volcengine/*`
kataloğu aynı anda kaydedilir.

Onboarding/model yapılandırma seçicilerinde Volcengine auth seçimi hem
`volcengine/*` hem de `volcengine-plan/*` satırlarını tercih eder. Bu modeller henüz yüklenmemişse,
OpenClaw boş bir sağlayıcı kapsamlı seçici göstermek yerine filtrelenmemiş kataloğa geri döner.

Kullanılabilir modeller:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Coding modelleri (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (Uluslararası)

BytePlus ARK, uluslararası kullanıcılar için Volcano Engine ile aynı modellere erişim sağlar.

- Sağlayıcı: `byteplus` (coding: `byteplus-plan`)
- Auth: `BYTEPLUS_API_KEY`
- Örnek model: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

Onboarding varsayılan olarak coding yüzeyini kullanır, ancak genel `byteplus/*`
kataloğu aynı anda kaydedilir.

Onboarding/model yapılandırma seçicilerinde BytePlus auth seçimi hem
`byteplus/*` hem de `byteplus-plan/*` satırlarını tercih eder. Bu modeller henüz yüklenmemişse,
OpenClaw boş bir sağlayıcı kapsamlı seçici göstermek yerine filtrelenmemiş kataloğa geri döner.

Kullanılabilir modeller:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Coding modelleri (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic, `synthetic` sağlayıcısının arkasında Anthropic uyumlu modeller sağlar:

- Sağlayıcı: `synthetic`
- Auth: `SYNTHETIC_API_KEY`
- Örnek model: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax, özel uç noktalar kullandığı için `models.providers` üzerinden yapılandırılır:

- MiniMax OAuth (Global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax API anahtarı (Global): `--auth-choice minimax-global-api`
- MiniMax API anahtarı (CN): `--auth-choice minimax-cn-api`
- Auth: `MINIMAX_API_KEY` için `minimax`; `MINIMAX_OAUTH_TOKEN` veya
  `MINIMAX_API_KEY` için `minimax-portal`

Kurulum ayrıntıları, model seçenekleri ve yapılandırma parçacıkları için bkz. [/providers/minimax](/tr/providers/minimax).

MiniMax'ın Anthropic uyumlu akış yolunda OpenClaw, açıkça ayarlamadığınız sürece düşünmeyi varsayılan olarak kapatır ve `/fast on`
`MiniMax-M2.7` modelini `MiniMax-M2.7-highspeed` olarak yeniden yazar.

Eklentiye ait yetenek ayrımı:

- Metin/sohbet varsayılanları `minimax/MiniMax-M2.7` üzerinde kalır
- Görüntü oluşturma `minimax/image-01` veya `minimax-portal/image-01` kullanır
- Görüntü anlama, her iki MiniMax auth yolunda da eklentiye ait `MiniMax-VL-01` modelidir
- Web araması sağlayıcı kimliği `minimax` üzerinde kalır

### Ollama

Ollama paketlenmiş bir sağlayıcı eklentisi olarak gelir ve Ollama'nın yerel API'sini kullanır:

- Sağlayıcı: `ollama`
- Auth: Gerekmez (yerel sunucu)
- Örnek model: `ollama/llama3.3`
- Kurulum: [https://ollama.com/download](https://ollama.com/download)

```bash
# Önce Ollama'yı kurun, ardından bir model çekin:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama, `OLLAMA_API_KEY` ile katılım gösterdiğinizde yerel olarak `http://127.0.0.1:11434` adresinde algılanır
ve paketlenmiş sağlayıcı eklentisi Ollama'yı doğrudan
`openclaw onboard` ve model seçiciye ekler. Onboarding, bulut/yerel mod ve özel yapılandırma için bkz. [/providers/ollama](/tr/providers/ollama).

### vLLM

vLLM, yerel/kendi barındırılan OpenAI uyumlu
sunucular için paketlenmiş bir sağlayıcı eklentisi olarak gelir:

- Sağlayıcı: `vllm`
- Auth: İsteğe bağlıdır (sunucunuza bağlıdır)
- Varsayılan temel URL: `http://127.0.0.1:8000/v1`

Yerelde otomatik keşfe katılmak için (sunucunuz auth zorlamıyorsa herhangi bir değer çalışır):

```bash
export VLLM_API_KEY="vllm-local"
```

Ardından bir model ayarlayın (`/v1/models` tarafından döndürülen kimliklerden biriyle değiştirin):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Ayrıntılar için bkz. [/providers/vllm](/tr/providers/vllm).

### SGLang

SGLang, hızlı kendi barındırılan
OpenAI uyumlu sunucular için paketlenmiş bir sağlayıcı eklentisi olarak gelir:

- Sağlayıcı: `sglang`
- Auth: İsteğe bağlıdır (sunucunuza bağlıdır)
- Varsayılan temel URL: `http://127.0.0.1:30000/v1`

Yerelde otomatik keşfe katılmak için (sunucunuz
auth zorlamıyorsa herhangi bir değer çalışır):

```bash
export SGLANG_API_KEY="sglang-local"
```

Ardından bir model ayarlayın (`/v1/models` tarafından döndürülen kimliklerden biriyle değiştirin):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Ayrıntılar için bkz. [/providers/sglang](/tr/providers/sglang).

### Yerel proxy'ler (LM Studio, vLLM, LiteLLM vb.)

Örnek (OpenAI uyumlu):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Notlar:

- Özel sağlayıcılar için `reasoning`, `input`, `cost`, `contextWindow` ve `maxTokens` isteğe bağlıdır.
  Atlandığında OpenClaw şu varsayılanları kullanır:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Önerilir: proxy/model sınırlarınızla eşleşen açık değerler ayarlayın.
- Yerel olmayan uç noktalarda `api: "openai-completions"` için (`api.openai.com` olmayan bir ana bilgisayara sahip boş olmayan her `baseUrl`), OpenClaw desteklenmeyen `developer` rolleri nedeniyle sağlayıcı 400 hatalarını önlemek için `compat.supportsDeveloperRole: false` değerini zorlar.
- Proxy tarzı OpenAI uyumlu rotalar ayrıca yerel yalnızca OpenAI'ya özgü istek
  şekillendirmesini de atlar: `service_tier` yok, Responses `store` yok, istem önbelleği ipuçları yok,
  OpenAI akıl yürütme uyumluluğu yük şekillendirmesi yok ve gizli OpenClaw atıf
  başlıkları yok.
- `baseUrl` boşsa/atlanmışsa OpenClaw varsayılan OpenAI davranışını korur (`api.openai.com` adresine çözülür).
- Güvenlik için, yerel olmayan `openai-completions` uç noktalarında açık `compat.supportsDeveloperRole: true` ayarı yine de geçersiz kılınır.

## CLI örnekleri

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Ayrıca bkz.: tam yapılandırma örnekleri için [/gateway/configuration](/tr/gateway/configuration).

## İlgili

- [Models](/tr/concepts/models) — model yapılandırması ve takma adlar
- [Model Failover](/tr/concepts/model-failover) — yedek zincirleri ve yeniden deneme davranışı
- [Configuration Reference](/tr/gateway/configuration-reference#agent-defaults) — model yapılandırma anahtarları
- [Providers](/tr/providers) — sağlayıcı başına kurulum kılavuzları
