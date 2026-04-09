---
read_when:
    - Testleri yerelde veya CI içinde çalıştırma
    - Model/sağlayıcı hataları için regresyon ekleme
    - Gateway + ajan davranışını hata ayıklama
summary: 'Test kiti: unit/e2e/live paketleri, Docker çalıştırıcıları ve her testin neyi kapsadığı'
title: Testler
x-i18n:
    generated_at: "2026-04-09T01:30:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 01117f41d8b171a4f1da11ed78486ee700e70ae70af54eb6060c57baf64ab21b
    source_path: help/testing.md
    workflow: 15
---

# Testler

OpenClaw'ın üç Vitest paketi (unit/integration, e2e, live) ve küçük bir Docker çalıştırıcı kümesi vardır.

Bu belge bir “nasıl test ediyoruz” kılavuzudur:

- Her paketin neyi kapsadığı (ve bilerek neyi _kapsamadığı_)
- Yaygın iş akışları için hangi komutların çalıştırılacağı (yerel, push öncesi, hata ayıklama)
- Live testlerin kimlik bilgilerini nasıl keşfettiği ve modelleri/sağlayıcıları nasıl seçtiği
- Gerçek dünyadaki model/sağlayıcı sorunları için nasıl regresyon ekleneceği

## Hızlı başlangıç

Çoğu gün:

- Tam kapı (push öncesinde beklenir): `pnpm build && pnpm check && pnpm test`
- Geniş kaynaklı bir makinede daha hızlı yerel tam paket çalıştırması: `pnpm test:max`
- Doğrudan Vitest izleme döngüsü: `pnpm test:watch`
- Doğrudan dosya hedefleme artık extension/channel yollarını da yönlendiriyor: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Docker destekli QA sitesi: `pnpm qa:lab:up`

Testlere dokunduğunuzda veya daha fazla güven istediğinizde:

- Kapsam kapısı: `pnpm test:coverage`
- E2E paketi: `pnpm test:e2e`

Gerçek sağlayıcıları/modelleri hata ayıklarken (gerçek kimlik bilgileri gerekir):

- Live paketi (modeller + gateway tool/image yoklamaları): `pnpm test:live`
- Tek bir live dosyasını sessizce hedefleme: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

İpucu: yalnızca tek bir başarısız duruma ihtiyacınız olduğunda, aşağıda açıklanan izin listesi ortam değişkenleriyle live testleri daraltmayı tercih edin.

## Test paketleri (hangisi nerede çalışır)

Paketleri “artan gerçeklik” (ve artan oynaklık/maliyet) olarak düşünün:

### Unit / integration (varsayılan)

- Komut: `pnpm test`
- Yapılandırma: mevcut kapsamlı Vitest projeleri üzerinde on sıralı shard çalıştırması (`vitest.full-*.config.ts`)
- Dosyalar: `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` altındaki core/unit envanterleri ve `vitest.unit.config.ts` kapsamında yer alan izinli `ui` node testleri
- Kapsam:
  - Saf unit testleri
  - Süreç içi integration testleri (gateway auth, routing, araçlar, ayrıştırma, yapılandırma)
  - Bilinen hatalar için deterministik regresyonlar
- Beklentiler:
  - CI içinde çalışır
  - Gerçek anahtarlar gerekmez
  - Hızlı ve kararlı olmalıdır
- Projeler notu:
  - Hedeflenmemiş `pnpm test`, artık tek bir devasa yerel root-project süreci yerine on bir daha küçük shard yapılandırması (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) çalıştırır. Bu, yoğun makinelerde en yüksek RSS'yi azaltır ve auto-reply/extension işlerinin ilgisiz paketleri aç bırakmasını önler.
  - `pnpm test --watch`, çok shard'lı bir izleme döngüsü pratik olmadığı için yine de yerel root `vitest.config.ts` proje grafiğini kullanır.
  - `pnpm test`, `pnpm test:watch` ve `pnpm test:perf:imports`, açık dosya/dizin hedeflerini önce kapsamlı hatlar üzerinden yönlendirir; böylece `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` tam root proje başlangıç maliyetini ödemez.
  - `pnpm test:changed`, fark yalnızca yönlendirilebilir kaynak/test dosyalarına dokunuyorsa değişen git yollarını aynı kapsamlı hatlara genişletir; config/setup düzenlemeleri ise yine geniş root-project yeniden çalıştırmasına döner.
  - Seçili `plugin-sdk` ve `commands` testleri de `test/setup-openclaw-runtime.ts` dosyasını atlayan özel hafif hatlar üzerinden yönlendirilir; durumlu/çalışma zamanı açısından ağır dosyalar mevcut hatlarda kalır.
  - Seçili `plugin-sdk` ve `commands` yardımcı kaynak dosyaları da changed-mode çalıştırmalarını bu hafif hatlardaki açık kardeş testlere eşler; böylece yardımcı düzenlemeleri o dizin için tam ağır paketi yeniden çalıştırmaz.
  - `auto-reply` artık üç özel kovaya sahiptir: üst düzey core yardımcıları, üst düzey `reply.*` integration testleri ve `src/auto-reply/reply/**` alt ağacı. Bu, en ağır reply harness işini ucuz status/chunk/token testlerinden uzak tutar.
- Gömülü çalıştırıcı notu:
  - Mesaj-tool keşif girdilerini veya compaction çalışma zamanı bağlamını değiştirdiğinizde, her iki kapsama düzeyini de koruyun.
  - Saf routing/normalization sınırları için odaklı yardımcı regresyonlar ekleyin.
  - Ayrıca gömülü çalıştırıcı integration paketlerini de sağlıklı tutun:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` ve
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Bu paketler, kapsamlı kimliklerin ve compaction davranışının gerçek `run.ts` / `compact.ts` yollarından geçmeye devam ettiğini doğrular; yalnızca yardımcı testleri bu integration yolları için yeterli bir ikame değildir.
- Havuz notu:
  - Temel Vitest yapılandırması artık varsayılan olarak `threads` kullanır.
  - Paylaşılan Vitest yapılandırması ayrıca `isolate: false` ayarlar ve yalıtılmamış çalıştırıcıyı root projeleri, e2e ve live yapılandırmaları genelinde kullanır.
  - Root UI hattı kendi `jsdom` kurulumunu ve optimizer'ını korur, ancak artık paylaşılan yalıtılmamış çalıştırıcıda çalışır.
  - Her `pnpm test` shard'ı, paylaşılan Vitest yapılandırmasından aynı `threads` + `isolate: false` varsayılanlarını devralır.
  - Paylaşılan `scripts/run-vitest.mjs` başlatıcısı artık büyük yerel çalıştırmalar sırasında V8 derleme çalkantısını azaltmak için varsayılan olarak Vitest alt Node süreçlerine `--no-maglev` de ekler. Stok V8 davranışıyla karşılaştırmanız gerekiyorsa `OPENCLAW_VITEST_ENABLE_MAGLEV=1` ayarlayın.
- Hızlı yerel yineleme notu:
  - `pnpm test:changed`, değişen yollar daha küçük bir pakete temiz şekilde eşleniyorsa kapsamlı hatlar üzerinden yönlendirilir.
  - `pnpm test:max` ve `pnpm test:changed:max` aynı yönlendirme davranışını korur; yalnızca daha yüksek bir worker sınırıyla.
  - Yerel worker otomatik ölçeklendirmesi artık bilerek daha tutucudur ve ana makinenin yük ortalaması zaten yüksek olduğunda da geri çekilir; böylece aynı anda çalışan birden çok Vitest süreci varsayılan olarak daha az zarar verir.
  - Temel Vitest yapılandırması, test kablolaması değiştiğinde changed-mode yeniden çalıştırmaların doğru kalması için projeleri/yapılandırma dosyalarını `forceRerunTriggers` olarak işaretler.
  - Yapılandırma, desteklenen ana makinelerde `OPENCLAW_VITEST_FS_MODULE_CACHE` özelliğini etkin tutar; doğrudan profiling için tek bir açık önbellek konumu istiyorsanız `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` ayarlayın.
- Performans hata ayıklama notu:
  - `pnpm test:perf:imports`, Vitest import-duration raporlamasını ve import-breakdown çıktısını etkinleştirir.
  - `pnpm test:perf:imports:changed`, aynı profiling görünümünü `origin/main` sonrası değişen dosyalarla sınırlar.
- `pnpm test:perf:changed:bench -- --ref <git-ref>`, yönlendirilmiş `test:changed` ile o commit edilmiş fark için yerel root-project yolunu karşılaştırır ve duvar saati süresi ile macOS maksimum RSS'yi yazdırır.
- `pnpm test:perf:changed:bench -- --worktree`, mevcut kirli ağacı değişen dosya listesini `scripts/test-projects.mjs` ve root Vitest yapılandırması üzerinden yönlendirerek kıyaslar.
  - `pnpm test:perf:profile:main`, Vitest/Vite başlangıcı ve transform yükü için bir ana iş parçacığı CPU profili yazar.
  - `pnpm test:perf:profile:runner`, unit paketi için dosya paralelliği devre dışıyken çalıştırıcı CPU+heap profilleri yazar.

### E2E (gateway smoke)

- Komut: `pnpm test:e2e`
- Yapılandırma: `vitest.e2e.config.ts`
- Dosyalar: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Çalışma zamanı varsayılanları:
  - Deponun geri kalanıyla eşleşecek şekilde Vitest `threads` ile `isolate: false` kullanır.
  - Uyarlanabilir worker'lar kullanır (CI: en fazla 2, yerel: varsayılan 1).
  - Konsol G/Ç yükünü azaltmak için varsayılan olarak sessiz modda çalışır.
- Yararlı geçersiz kılmalar:
  - Worker sayısını zorlamak için `OPENCLAW_E2E_WORKERS=<n>` (üst sınır 16).
  - Ayrıntılı konsol çıktısını yeniden etkinleştirmek için `OPENCLAW_E2E_VERBOSE=1`.
- Kapsam:
  - Çok örnekli gateway uçtan uca davranışı
  - WebSocket/HTTP yüzeyleri, node eşleme ve daha ağır ağ işlemleri
- Beklentiler:
  - CI içinde çalışır (boru hattında etkinleştirildiğinde)
  - Gerçek anahtarlar gerekmez
  - Unit testlerinden daha fazla hareketli parça vardır (daha yavaş olabilir)

### E2E: OpenShell arka uç smoke

- Komut: `pnpm test:e2e:openshell`
- Dosya: `test/openshell-sandbox.e2e.test.ts`
- Kapsam:
  - Docker aracılığıyla ana makinede yalıtılmış bir OpenShell gateway başlatır
  - Geçici bir yerel Dockerfile'dan bir sandbox oluşturur
  - OpenClaw'ın OpenShell arka ucunu gerçek `sandbox ssh-config` + SSH exec üzerinden çalıştırır
  - Uzak-kanonik dosya sistemi davranışını sandbox fs bridge üzerinden doğrular
- Beklentiler:
  - Yalnızca isteğe bağlıdır; varsayılan `pnpm test:e2e` çalıştırmasının parçası değildir
  - Yerel bir `openshell` CLI ve çalışan bir Docker daemon gerektirir
  - Yalıtılmış `HOME` / `XDG_CONFIG_HOME` kullanır, ardından test gateway'ini ve sandbox'ı yok eder
- Yararlı geçersiz kılmalar:
  - Daha geniş e2e paketini elle çalıştırırken testi etkinleştirmek için `OPENCLAW_E2E_OPENSHELL=1`
  - Varsayılan olmayan bir CLI binary'si veya wrapper script'ine işaret etmek için `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live (gerçek sağlayıcılar + gerçek modeller)

- Komut: `pnpm test:live`
- Yapılandırma: `vitest.live.config.ts`
- Dosyalar: `src/**/*.live.test.ts`
- Varsayılan: `pnpm test:live` tarafından **etkinleştirilir** (`OPENCLAW_LIVE_TEST=1` ayarlar)
- Kapsam:
  - “Bu sağlayıcı/model _bugün_ gerçek kimlik bilgileriyle gerçekten çalışıyor mu?”
  - Sağlayıcı biçim değişikliklerini, tool-calling tuhaflıklarını, auth sorunlarını ve rate limit davranışını yakalar
- Beklentiler:
  - Tasarım gereği CI'da kararlı değildir (gerçek ağlar, gerçek sağlayıcı politikaları, kotalar, kesintiler)
  - Para harcar / rate limit kullanır
  - “Her şeyi” yerine daraltılmış alt kümeleri çalıştırmak tercih edilmelidir
- Live çalıştırmalar, eksik API anahtarlarını almak için `~/.profile` dosyasını source eder.
- Varsayılan olarak live çalıştırmalar yine de `HOME` dizinini yalıtır ve config/auth materyallerini geçici bir test home'una kopyalar; böylece unit fixture'ları gerçek `~/.openclaw` dizininizi değiştiremez.
- Live testlerin gerçek home dizininizi kullanmasını kasıtlı olarak istediğiniz durumlarda yalnızca `OPENCLAW_LIVE_USE_REAL_HOME=1` ayarlayın.
- `pnpm test:live` artık daha sessiz bir modda varsayılandır: `[live] ...` ilerleme çıktısını korur, ancak ek `~/.profile` bildirimini bastırır ve gateway bootstrap loglarını/Bonjour gürültüsünü susturur. Tam başlangıç loglarını geri istiyorsanız `OPENCLAW_LIVE_TEST_QUIET=0` ayarlayın.
- API anahtarı rotasyonu (sağlayıcıya özgü): virgül/noktalı virgül biçimiyle `*_API_KEYS` veya `*_API_KEY_1`, `*_API_KEY_2` ayarlayın (örneğin `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ya da live başına geçersiz kılma için `OPENCLAW_LIVE_*_KEY`; testler rate limit yanıtlarında yeniden dener.
- İlerleme/heartbeat çıktısı:
  - Live paketleri artık uzun sağlayıcı çağrılarının Vitest konsol yakalaması sessiz olsa bile görünür şekilde etkin kalması için stderr'e ilerleme satırları yazar.
  - `vitest.live.config.ts`, sağlayıcı/gateway ilerleme satırlarının live çalıştırmalar sırasında hemen akması için Vitest konsol yakalamasını devre dışı bırakır.
  - Doğrudan model heartbeat'lerini `OPENCLAW_LIVE_HEARTBEAT_MS` ile ayarlayın.
  - Gateway/probe heartbeat'lerini `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` ile ayarlayın.

## Hangi paketi çalıştırmalıyım?

Bu karar tablosunu kullanın:

- Mantığı/testleri düzenliyorsanız: `pnpm test` çalıştırın (ve çok şey değiştirdiyseniz `pnpm test:coverage`)
- Gateway ağı / WS protokolü / eşleme yüzeyine dokunuyorsanız: `pnpm test:e2e` ekleyin
- “Botum çalışmıyor” / sağlayıcıya özgü hatalar / tool calling hata ayıklaması yapıyorsanız: daraltılmış bir `pnpm test:live` çalıştırın

## Live: Android node capability sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Script: `pnpm android:test:integration`
- Amaç: bağlı bir Android node tarafından şu anda reklamı yapılan **her komutu** çağırmak ve komut sözleşmesi davranışını doğrulamak.
- Kapsam:
  - Önkoşullu/manuel kurulum (paket uygulamayı kurmaz/çalıştırmaz/eşlemez).
  - Seçilen Android node için komut bazında gateway `node.invoke` doğrulaması.
- Gerekli ön kurulum:
  - Android uygulaması zaten bağlanmış ve gateway ile eşlenmiş olmalı.
  - Uygulama ön planda tutulmalı.
  - Geçmesini beklediğiniz yetenekler için izinler/yakalama onayı verilmiş olmalı.
- İsteğe bağlı hedef geçersiz kılmaları:
  - `OPENCLAW_ANDROID_NODE_ID` veya `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Tüm Android kurulum ayrıntıları: [Android App](/tr/platforms/android)

## Live: model smoke (profil anahtarları)

Live testler, hataları izole edebilmek için iki katmana ayrılmıştır:

- “Doğrudan model”, sağlayıcının/modelin verilen anahtarla gerçekten yanıt verebildiğini gösterir.
- “Gateway smoke”, tam gateway+ajan hattının o model için çalıştığını gösterir (oturumlar, geçmiş, araçlar, sandbox politikası vb.).

### Katman 1: Doğrudan model tamamlama (gateway yok)

- Test: `src/agents/models.profiles.live.test.ts`
- Amaç:
  - Keşfedilen modelleri listelemek
  - Kimlik bilgilerinizin olduğu modelleri seçmek için `getApiKeyForModel` kullanmak
  - Model başına küçük bir tamamlama çalıştırmak (ve gerektiğinde hedefli regresyonlar)
- Nasıl etkinleştirilir:
  - `pnpm test:live` (veya Vitest'i doğrudan çağırıyorsanız `OPENCLAW_LIVE_TEST=1`)
- Bu paketi gerçekten çalıştırmak için `OPENCLAW_LIVE_MODELS=modern` (veya `all`, modern için takma ad) ayarlayın; aksi halde `pnpm test:live` odağını gateway smoke üzerinde tutmak için atlanır
- Modeller nasıl seçilir:
  - Modern izin listesini çalıştırmak için `OPENCLAW_LIVE_MODELS=modern` (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all`, modern izin listesi için bir takma addır
  - veya `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (virgülle ayrılmış izin listesi)
  - Modern/all taramaları varsayılan olarak özenle seçilmiş yüksek sinyalli bir üst sınır kullanır; kapsamlı bir modern tarama için `OPENCLAW_LIVE_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
- Sağlayıcılar nasıl seçilir:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (virgülle ayrılmış izin listesi)
- Anahtarlar nereden gelir:
  - Varsayılan olarak: profil deposu ve env geri dönüşleri
  - Yalnızca **profil deposunu** zorunlu kılmak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` ayarlayın
- Bunun varlık nedeni:
  - “Sağlayıcı API'si bozuk / anahtar geçersiz” ile “gateway ajan hattı bozuk” durumlarını ayırır
  - Küçük, yalıtılmış regresyonlar içerir (örnek: OpenAI Responses/Codex Responses reasoning replay + tool-call akışları)

### Katman 2: Gateway + dev agent smoke (`@openclaw` gerçekten ne yapıyor)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Amaç:
  - Süreç içi bir gateway başlatmak
  - Bir `agent:dev:*` oturumu oluşturmak/yamalamak (çalıştırma başına model geçersiz kılma)
  - Anahtarı olan modeller arasında dolaşmak ve şunları doğrulamak:
    - “anlamlı” yanıt (araç yok)
    - gerçek bir tool invocation çalışır (`read` probe)
    - isteğe bağlı ek tool probe'ları (`exec+read` probe)
    - OpenAI regresyon yolları (yalnızca tool-call → devam) çalışmaya devam eder
- Probe ayrıntıları (böylece hataları hızlıca açıklayabilirsiniz):
  - `read` probe: test çalışma alanına nonce içeren bir dosya yazar ve ajandan onu `read` ile okuyup nonce'u geri yankılamasını ister.
  - `exec+read` probe: test ajandan nonce'u geçici bir dosyaya `exec` ile yazmasını, ardından `read` ile geri okumasını ister.
  - image probe: test oluşturulmuş bir PNG (kedi + rastgele kod) ekler ve modelin `cat <CODE>` döndürmesini bekler.
  - Uygulama başvurusu: `src/gateway/gateway-models.profiles.live.test.ts` ve `src/gateway/live-image-probe.ts`.
- Nasıl etkinleştirilir:
  - `pnpm test:live` (veya Vitest'i doğrudan çağırıyorsanız `OPENCLAW_LIVE_TEST=1`)
- Modeller nasıl seçilir:
  - Varsayılan: modern izin listesi (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all`, modern izin listesi için bir takma addır
  - Ya da daraltmak için `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (veya virgüllü liste) ayarlayın
  - Modern/all gateway taramaları varsayılan olarak özenle seçilmiş yüksek sinyalli bir üst sınır kullanır; kapsamlı bir modern tarama için `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
- Sağlayıcılar nasıl seçilir (“OpenRouter her şey” yaklaşımından kaçının):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (virgülle ayrılmış izin listesi)
- Tool + image probe'ları bu live testte her zaman açıktır:
  - `read` probe + `exec+read` probe (tool stress)
  - model image input desteği bildiriyorsa image probe çalışır
  - Akış (yüksek seviye):
    - Test, “CAT” + rastgele kod içeren küçük bir PNG üretir (`src/gateway/live-image-probe.ts`)
    - Bunu `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` üzerinden gönderir
    - Gateway ekleri `images[]` içine ayrıştırır (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Gömülü ajan modele çok kipli bir kullanıcı mesajı iletir
    - Doğrulama: yanıt `cat` + kodu içerir (OCR toleransı: küçük hatalara izin verilir)

İpucu: makinenizde neyi test edebileceğinizi (ve tam `provider/model` kimliklerini) görmek için şunu çalıştırın:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI arka uç smoke (Claude, Codex, Gemini veya diğer yerel CLI'lar)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Amaç: varsayılan yapılandırmanıza dokunmadan, Gateway + ajan hattını yerel bir CLI arka ucu kullanarak doğrulamak.
- Arka uca özgü smoke varsayılanları, sahibi olan extension'ın `cli-backend.ts` tanımı içinde bulunur.
- Etkinleştirme:
  - `pnpm test:live` (veya Vitest'i doğrudan çağırıyorsanız `OPENCLAW_LIVE_TEST=1`)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Varsayılanlar:
  - Varsayılan sağlayıcı/model: `claude-cli/claude-sonnet-4-6`
  - Komut/argüman/image davranışı sahibi olan CLI arka uç plugin metadata'sından gelir.
- Geçersiz kılmalar (isteğe bağlı):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - Gerçek bir image attachment göndermek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` (yollar prompt içine enjekte edilir).
  - Görüntü dosya yollarını prompt enjeksiyonu yerine CLI argümanları olarak geçirmek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`.
  - `IMAGE_ARG` ayarlandığında görüntü argümanlarının nasıl geçirileceğini denetlemek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (veya `"list"`).
  - İkinci bir tur göndermek ve resume akışını doğrulamak için `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`.
  - Varsayılan Claude Sonnet -> Opus aynı oturum sürekliliği probe'unu devre dışı bırakmak için `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` (seçilen model bir geçiş hedefini desteklediğinde bunu açıkça zorlamak için `1`).

Örnek:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Docker tarifi:

```bash
pnpm test:docker:live-cli-backend
```

Tek sağlayıcılı Docker tarifleri:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Notlar:

- Docker çalıştırıcısı `scripts/test-live-cli-backend-docker.sh` konumundadır.
- Live CLI-backend smoke'u repo Docker imajı içinde root olmayan `node` kullanıcısı olarak çalıştırır.
- CLI smoke metadata'sını sahibi olan extension'dan çözer, ardından eşleşen Linux CLI paketini (`@anthropic-ai/claude-code`, `@openai/codex` veya `@google/gemini-cli`) `OPENCLAW_DOCKER_CLI_TOOLS_DIR` içindeki önbelleğe alınmış yazılabilir bir öneke kurar (varsayılan: `~/.cache/openclaw/docker-cli-tools`).
- Live CLI-backend smoke artık Claude, Codex ve Gemini için aynı uçtan uca akışı çalıştırır: metin turu, görüntü sınıflandırma turu, sonra gateway CLI üzerinden doğrulanan MCP `cron` tool call.
- Claude'un varsayılan smoke'u ayrıca oturumu Sonnet'ten Opus'a yamalar ve sürdürülen oturumun daha önceki bir notu hâlâ hatırladığını doğrular.

## Live: ACP bind smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Amaç: canlı bir ACP ajanıyla gerçek ACP conversation-bind akışını doğrulamak:
  - `/acp spawn <agent> --bind here` gönder
  - sentetik bir message-channel konuşmasını yerinde bağla
  - aynı konuşma üzerinde normal bir devam mesajı gönder
  - devamın bağlı ACP oturum dökümüne indiğini doğrula
- Etkinleştirme:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Varsayılanlar:
  - Docker içindeki ACP ajanları: `claude,codex,gemini`
  - Doğrudan `pnpm test:live ...` için ACP ajanı: `claude`
  - Sentetik channel: Slack DM tarzı konuşma bağlamı
  - ACP arka ucu: `acpx`
- Geçersiz kılmalar:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Notlar:
  - Bu hat, testlerin haricen teslim ediliyormuş gibi yapmadan message-channel bağlamı ekleyebilmesi için yöneticiye özel sentetik originating-route alanlarıyla gateway `chat.send` yüzeyini kullanır.
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` ayarlı değilse test, seçilen ACP harness ajanı için gömülü `acpx` plugin'inin yerleşik ajan kaydını kullanır.

Örnek:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker tarifi:

```bash
pnpm test:docker:live-acp-bind
```

Tek ajanlı Docker tarifleri:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker notları:

- Docker çalıştırıcısı `scripts/test-live-acp-bind-docker.sh` konumundadır.
- Varsayılan olarak ACP bind smoke'u desteklenen tüm live CLI ajanlarına sırayla karşı çalıştırır: `claude`, `codex`, ardından `gemini`.
- Matrisi daraltmak için `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` veya `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` kullanın.
- `~/.profile` dosyasını source eder, eşleşen CLI auth materyalini konteynere hazırlar, `acpx`'i yazılabilir bir npm önekine kurar, ardından eksikse istenen live CLI'ı (`@anthropic-ai/claude-code`, `@openai/codex` veya `@google/gemini-cli`) kurar.
- Docker içinde çalıştırıcı, source edilen profilden sağlayıcı env değişkenlerinin alt harness CLI için kullanılabilir kalması amacıyla `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` ayarlar.

### Önerilen live tarifleri

Dar, açık izin listeleri en hızlısı ve en az oynak olanıdır:

- Tek model, doğrudan (gateway yok):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Tek model, gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Birden çok sağlayıcıda tool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google odağı (Gemini API anahtarı + Antigravity):
  - Gemini (API anahtarı): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Notlar:

- `google/...`, Gemini API'sini kullanır (API anahtarı).
- `google-antigravity/...`, Antigravity OAuth köprüsünü kullanır (Cloud Code Assist tarzı ajan uç noktası).
- `google-gemini-cli/...`, makinenizdeki yerel Gemini CLI'ı kullanır (ayrı auth + tooling tuhaflıkları).
- Gemini API ile Gemini CLI:
  - API: OpenClaw, Google'ın barındırılan Gemini API'sini HTTP üzerinden çağırır (API anahtarı / profil auth); çoğu kullanıcının “Gemini” derken kastettiği budur.
  - CLI: OpenClaw yerel `gemini` binary'sini shell üzerinden çalıştırır; kendi auth'una sahiptir ve farklı davranabilir (streaming/tool desteği/sürüm kayması).

## Live: model matrisi (neleri kapsıyoruz)

Sabit bir “CI model listesi” yoktur (live isteğe bağlıdır), ancak anahtarları olan bir geliştirici makinesinde düzenli olarak kapsanması **önerilen** modeller bunlardır.

### Modern smoke kümesi (tool calling + image)

Bu, çalışır durumda kalmasını beklediğimiz “yaygın modeller” çalıştırmasıdır:

- OpenAI (Codex olmayan): `openai/gpt-5.4` (isteğe bağlı: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (veya `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` ve `google/gemini-3-flash-preview` (eski Gemini 2.x modellerinden kaçının)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` ve `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Araçlar + image ile gateway smoke çalıştırın:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Temel çizgi: tool calling (Read + isteğe bağlı Exec)

Sağlayıcı ailesi başına en az birini seçin:

- OpenAI: `openai/gpt-5.4` (veya `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (veya `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (veya `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

İsteğe bağlı ek kapsam (olursa iyi olur):

- xAI: `xai/grok-4` (veya mevcut en yeni sürüm)
- Mistral: `mistral/`… (etkinleştirdiğiniz “tools” yetenekli bir model seçin)
- Cerebras: `cerebras/`… (erişiminiz varsa)
- LM Studio: `lmstudio/`… (yerel; tool calling API moduna bağlıdır)

### Vision: görüntü gönderme (ek → çok kipli mesaj)

Image probe'u çalıştırmak için `OPENCLAW_LIVE_GATEWAY_MODELS` içine en az bir image-capable model ekleyin (Claude/Gemini/OpenAI'nin image destekli varyantları vb.).

### Aggregator'lar / alternatif gateway'ler

Anahtarlarınız etkinse şunlar üzerinden test etmeyi de destekliyoruz:

- OpenRouter: `openrouter/...` (yüzlerce model; tool+image destekli adayları bulmak için `openclaw models scan` kullanın)
- OpenCode: Zen için `opencode/...` ve Go için `opencode-go/...` (`OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY` ile auth)

Live matrisine dahil edebileceğiniz daha fazla sağlayıcı (kimlik bilgileriniz/yapılandırmanız varsa):

- Yerleşik: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers` aracılığıyla (özel uç noktalar): `minimax` (bulut/API), ayrıca OpenAI/Anthropic uyumlu tüm proxy'ler (LM Studio, vLLM, LiteLLM vb.)

İpucu: belgelerde “tüm modelleri” sabit kodlamaya çalışmayın. Yetkili liste, makinenizde `discoverModels(...)` ne döndürüyorsa ve hangi anahtarlar mevcutsa odur.

## Kimlik bilgileri (asla commit etmeyin)

Live testler kimlik bilgilerini CLI ile aynı şekilde keşfeder. Pratik sonuçları:

- CLI çalışıyorsa, live testler aynı anahtarları bulmalıdır.
- Bir live test “kimlik bilgisi yok” diyorsa, bunu `openclaw models list` / model seçimi hata ayıklamasıyla aynı şekilde ayıklayın.

- Ajan başına auth profilleri: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (live testlerde “profil anahtarları” ile kastedilen budur)
- Yapılandırma: `~/.openclaw/openclaw.json` (veya `OPENCLAW_CONFIG_PATH`)
- Eski durum dizini: `~/.openclaw/credentials/` (varsa hazırlanan live home içine kopyalanır, ancak ana profil-anahtarı deposu değildir)
- Yerel live çalıştırmalar varsayılan olarak etkin config'i, ajan başına `auth-profiles.json` dosyalarını, eski `credentials/` dizinini ve desteklenen harici CLI auth dizinlerini geçici bir test home'una kopyalar; hazırlanan live home'lar `workspace/` ve `sandboxes/` dizinlerini atlar ve probe'ların gerçek ana makine çalışma alanınızdan uzak kalması için `agents.*.workspace` / `agentDir` yol geçersiz kılmaları çıkarılır.

Env anahtarlarına güvenmek istiyorsanız (örneğin `~/.profile` dosyanızda export edilmişlerse), yerel testleri `source ~/.profile` sonrasında çalıştırın veya aşağıdaki Docker çalıştırıcılarını kullanın (bunlar `~/.profile` dosyasını konteynere bağlayabilir).

## Deepgram live (ses transkripsiyonu)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Etkinleştirme: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- Test: `src/agents/byteplus.live.test.ts`
- Etkinleştirme: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- İsteğe bağlı model geçersiz kılması: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- Test: `extensions/comfy/comfy.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Kapsam:
  - Birlikte gelen comfy image, video ve `music_generate` yollarını çalıştırır
  - `models.providers.comfy.<capability>` yapılandırılmadıkça her yeteneği atlar
  - Comfy workflow gönderimi, polling, indirmeler veya plugin kaydı değiştirildikten sonra kullanışlıdır

## Görüntü oluşturma live

- Test: `src/image-generation/runtime.live.test.ts`
- Komut: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Kapsam:
  - Kayıtlı her image-generation sağlayıcı plugin'ini listeler
  - Yoklamadan önce eksik sağlayıcı env değişkenlerini login shell'inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan auth profillerinden önce live/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir auth/profil/model olmayan sağlayıcıları atlar
  - Stok image-generation varyantlarını paylaşılan runtime capability üzerinden çalıştırır:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Şu anda kapsanan birlikte gelen sağlayıcılar:
  - `openai`
  - `google`
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- İsteğe bağlı auth davranışı:
  - Yalnızca profil-deposu auth'unu zorlamak ve yalnızca env geçersiz kılmalarını yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Müzik oluşturma live

- Test: `extensions/music-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Kapsam:
  - Paylaşılan birlikte gelen music-generation sağlayıcı yolunu çalıştırır
  - Şu anda Google ve MiniMax'ı kapsar
  - Yoklamadan önce sağlayıcı env değişkenlerini login shell'inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan auth profillerinden önce live/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir auth/profil/model olmayan sağlayıcıları atlar
  - Mevcut olduğunda bildirilen her iki runtime modunu da çalıştırır:
    - prompt-only girdi ile `generate`
    - sağlayıcı `capabilities.edit.enabled` bildiriyorsa `edit`
  - Şu anki paylaşılan hat kapsamı:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: ayrı Comfy live dosyası, bu paylaşılan tarama değil
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- İsteğe bağlı auth davranışı:
  - Yalnızca profil-deposu auth'unu zorlamak ve yalnızca env geçersiz kılmalarını yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Video oluşturma live

- Test: `extensions/video-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Kapsam:
  - Paylaşılan birlikte gelen video-generation sağlayıcı yolunu çalıştırır
  - Yoklamadan önce sağlayıcı env değişkenlerini login shell'inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan auth profillerinden önce live/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir auth/profil/model olmayan sağlayıcıları atlar
  - Mevcut olduğunda bildirilen her iki runtime modunu da çalıştırır:
    - prompt-only girdi ile `generate`
    - sağlayıcı `capabilities.imageToVideo.enabled` bildiriyorsa ve seçilen sağlayıcı/model paylaşılan taramada buffer destekli yerel image input kabul ediyorsa `imageToVideo`
    - sağlayıcı `capabilities.videoToVideo.enabled` bildiriyorsa ve seçilen sağlayıcı/model paylaşılan taramada buffer destekli yerel video input kabul ediyorsa `videoToVideo`
  - Paylaşılan taramada şu anda bildirilen ancak atlanan `imageToVideo` sağlayıcıları:
    - `vydra`, çünkü birlikte gelen `veo3` yalnızca metindir ve birlikte gelen `kling` uzak bir image URL'si gerektirir
  - Sağlayıcıya özgü Vydra kapsamı:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - bu dosya varsayılan olarak uzak image URL fixture'ı kullanan `kling` hattıyla birlikte `veo3` text-to-video çalıştırır
  - Mevcut `videoToVideo` live kapsamı:
    - yalnızca seçilen model `runway/gen4_aleph` olduğunda `runway`
  - Paylaşılan taramada şu anda bildirilen ancak atlanan `videoToVideo` sağlayıcıları:
    - `alibaba`, `qwen`, `xai`; çünkü bu yollar şu anda uzak `http(s)` / MP4 referans URL'leri gerektiriyor
    - `google`; çünkü mevcut paylaşılan Gemini/Veo hattı yerel buffer destekli girdi kullanıyor ve bu yol paylaşılan taramada kabul edilmiyor
    - `openai`; çünkü mevcut paylaşılan hattın org'a özgü video inpaint/remix erişim garantileri yok
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- İsteğe bağlı auth davranışı:
  - Yalnızca profil-deposu auth'unu zorlamak ve yalnızca env geçersiz kılmalarını yok saymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Medya live harness

- Komut: `pnpm test:live:media`
- Amaç:
  - Paylaşılan image, music ve video live paketlerini depo-yerel tek bir giriş noktası üzerinden çalıştırır
  - Eksik sağlayıcı env değişkenlerini `~/.profile` dosyasından otomatik yükler
  - Varsayılan olarak her paketi şu anda kullanılabilir auth'a sahip sağlayıcılara otomatik daraltır
  - `scripts/test-live.mjs` dosyasını yeniden kullanır; böylece heartbeat ve sessiz mod davranışı tutarlı kalır
- Örnekler:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker çalıştırıcıları (isteğe bağlı “Linux'ta çalışıyor” kontrolleri)

Bu Docker çalıştırıcıları iki kovaya ayrılır:

- Live-model çalıştırıcıları: `test:docker:live-models` ve `test:docker:live-gateway`, yalnızca eşleşen profil-anahtarı live dosyalarını repo Docker imajı içinde çalıştırır (`src/agents/models.profiles.live.test.ts` ve `src/gateway/gateway-models.profiles.live.test.ts`), yerel config dizininizi ve çalışma alanınızı bağlar (ve bağlandıysa `~/.profile` dosyasını source eder). Eşleşen yerel giriş noktaları `test:live:models-profiles` ve `test:live:gateway-profiles`'tır.
- Docker live çalıştırıcıları, tam bir Docker taramasının pratik kalması için varsayılan olarak daha küçük bir smoke sınırı kullanır:
  `test:docker:live-models` varsayılan olarak `OPENCLAW_LIVE_MAX_MODELS=12`, ve
  `test:docker:live-gateway` varsayılan olarak `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` ve
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` kullanır. Daha büyük kapsamlı taramayı açıkça istediğinizde bu env değişkenlerini geçersiz kılın.
- `test:docker:all`, canlı Docker imajını bir kez `test:docker:live-build` ile oluşturur, ardından iki live Docker hattı için onu yeniden kullanır.
- Konteyner smoke çalıştırıcıları: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` ve `test:docker:plugins`; bunlar bir veya daha fazla gerçek konteyner başlatır ve daha üst düzey integration yollarını doğrular.

Live-model Docker çalıştırıcıları ayrıca yalnızca gerekli CLI auth home'larını bağlayarak bağlar (veya çalıştırma daraltılmadıysa desteklenenlerin tümünü), ardından harici CLI OAuth'un host auth deposunu değiştirmeden token yenileyebilmesi için çalıştırma öncesi bunları konteyner home'una kopyalar:

- Doğrudan modeller: `pnpm test:docker:live-models` (script: `scripts/test-live-models-docker.sh`)
- ACP bind smoke: `pnpm test:docker:live-acp-bind` (script: `scripts/test-live-acp-bind-docker.sh`)
- CLI arka uç smoke: `pnpm test:docker:live-cli-backend` (script: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + dev agent: `pnpm test:docker:live-gateway` (script: `scripts/test-live-gateway-models-docker.sh`)
- Open WebUI live smoke: `pnpm test:docker:openwebui` (script: `scripts/e2e/openwebui-docker.sh`)
- Onboarding wizard (TTY, tam iskelet kurulum): `pnpm test:docker:onboard` (script: `scripts/e2e/onboard-docker.sh`)
- Gateway ağı (iki konteyner, WS auth + health): `pnpm test:docker:gateway-network` (script: `scripts/e2e/gateway-network-docker.sh`)
- MCP channel bridge (tohumlanmış Gateway + stdio bridge + ham Claude notification-frame smoke): `pnpm test:docker:mcp-channels` (script: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (kurulum smoke + `/plugin` takma adı + Claude-bundle yeniden başlatma semantiği): `pnpm test:docker:plugins` (script: `scripts/e2e/plugins-docker.sh`)

Live-model Docker çalıştırıcıları ayrıca mevcut checkout'u salt okunur olarak bağlar ve bunu konteyner içinde geçici bir workdir'e hazırlar. Bu, runtime imajını ince tutarken yine de Vitest'i tam olarak yerel kaynak/yapılandırmanız üzerinde çalıştırır.
Hazırlama adımı; `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` ve uygulamaya özgü `.build` veya Gradle çıktı dizinleri gibi büyük yerel önbellekleri ve uygulama derleme çıktılarını atlar; böylece Docker live çalıştırmaları makineye özgü yapıtları kopyalamak için dakikalar harcamaz.
Ayrıca `OPENCLAW_SKIP_CHANNELS=1` ayarlarlar; böylece gateway live probe'ları konteyner içinde gerçek Telegram/Discord/vb. channel worker'ları başlatmaz.
`test:docker:live-models` yine de `pnpm test:live` çalıştırır, bu yüzden bu Docker hattında gateway live kapsamını daraltmanız veya hariç tutmanız gerektiğinde `OPENCLAW_LIVE_GATEWAY_*` değişkenlerini de geçirin.
`test:docker:openwebui`, daha üst düzey bir uyumluluk smoke testidir: OpenAI uyumlu HTTP uç noktaları etkinleştirilmiş bir OpenClaw gateway konteyneri başlatır, bu gateway'e karşı sabitlenmiş bir Open WebUI konteyneri başlatır, Open WebUI üzerinden oturum açar, `/api/models` içinde `openclaw/default` modelinin gösterildiğini doğrular, ardından Open WebUI'nin `/api/chat/completions` proxy'si üzerinden gerçek bir chat isteği gönderir.
İlk çalıştırma belirgin biçimde daha yavaş olabilir; çünkü Docker'ın Open WebUI imajını çekmesi gerekebilir ve Open WebUI'nin kendi soğuk başlangıç kurulumunu tamamlaması gerekebilir.
Bu hat kullanılabilir bir live model anahtarı bekler ve Docker'lı çalıştırmalarda bunu sağlamak için birincil yöntem `OPENCLAW_PROFILE_FILE` (`~/.profile` varsayılan) değişkenidir.
Başarılı çalıştırmalar `{ "ok": true, "model": "openclaw/default", ... }` gibi küçük bir JSON yükü yazdırır.
`test:docker:mcp-channels` bilerek deterministiktir ve gerçek bir Telegram, Discord veya iMessage hesabına ihtiyaç duymaz. Tohumlanmış bir Gateway konteyneri başlatır, `openclaw mcp serve` başlatan ikinci bir konteyner çalıştırır, ardından gerçek stdio MCP bridge üzerinden yönlendirilmiş konuşma keşfini, döküm okumalarını, ek metadata'sını, canlı olay kuyruğu davranışını, giden gönderim yönlendirmesini ve Claude tarzı channel + izin bildirimlerini doğrular. Bildirim kontrolü ham stdio MCP frame'lerini doğrudan inceler; böylece smoke testi yalnızca belirli bir istemci SDK'sının yüzeye çıkardığını değil, bridge'in gerçekten ne yaydığını doğrular.

Manuel ACP düz dil thread smoke (CI değil):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Bu script'i regresyon/hata ayıklama iş akışları için koruyun. ACP thread yönlendirme doğrulaması için yeniden gerekebilir, bu nedenle silmeyin.

Yararlı env değişkenleri:

- `OPENCLAW_CONFIG_DIR=...` (varsayılan: `~/.openclaw`) `/home/node/.openclaw` konumuna bağlanır
- `OPENCLAW_WORKSPACE_DIR=...` (varsayılan: `~/.openclaw/workspace`) `/home/node/.openclaw/workspace` konumuna bağlanır
- `OPENCLAW_PROFILE_FILE=...` (varsayılan: `~/.profile`) `/home/node/.profile` konumuna bağlanır ve testler çalıştırılmadan önce source edilir
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (varsayılan: `~/.cache/openclaw/docker-cli-tools`) Docker içinde önbelleğe alınmış CLI kurulumları için `/home/node/.npm-global` konumuna bağlanır
- `$HOME` altındaki harici CLI auth dizinleri/dosyaları salt okunur olarak `/host-auth...` altına bağlanır, ardından testler başlamadan önce `/home/node/...` altına kopyalanır
  - Varsayılan dizinler: `.minimax`
  - Varsayılan dosyalar: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Daraltılmış sağlayıcı çalıştırmaları yalnızca `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` üzerinden çıkarılan gerekli dizinleri/dosyaları bağlar
  - Elle geçersiz kılmak için `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` veya `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` gibi virgüllü bir liste kullanın
- Çalıştırmayı daraltmak için `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- Konteyner içindeki sağlayıcıları filtrelemek için `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- Kimlik bilgilerinin env'den değil profil deposundan gelmesini zorlamak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI smoke testinde gateway tarafından sunulan modeli seçmek için `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI smoke testinde kullanılan nonce-kontrol prompt'unu geçersiz kılmak için `OPENCLAW_OPENWEBUI_PROMPT=...`
- Sabitlenmiş Open WebUI imaj etiketini geçersiz kılmak için `OPENWEBUI_IMAGE=...`

## Belge sağlıklılık kontrolü

Belge düzenlemelerinden sonra belge kontrollerini çalıştırın: `pnpm check:docs`.
Sayfa içi başlık kontrollerine de ihtiyacınız olduğunda tam Mintlify anchor doğrulamasını çalıştırın: `pnpm docs:check-links:anchors`.

## Çevrimdışı regresyon (CI-güvenli)

Bunlar gerçek sağlayıcılar olmadan “gerçek hat” regresyonlarıdır:

- Gateway tool calling (sahte OpenAI, gerçek gateway + ajan döngüsü): `src/gateway/gateway.test.ts` (durum: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway wizard (WS `wizard.start`/`wizard.next`, config + auth enforced yazımı): `src/gateway/gateway.test.ts` (durum: "runs wizard over ws and writes auth token config")

## Ajan güvenilirliği değerlendirmeleri (Skills)

Zaten “ajan güvenilirliği değerlendirmeleri” gibi davranan birkaç CI-güvenli testimiz var:

- Gerçek gateway + ajan döngüsü üzerinden sahte tool-calling (`src/gateway/gateway.test.ts`).
- Oturum kablolamasını ve config etkilerini doğrulayan uçtan uca wizard akışları (`src/gateway/gateway.test.ts`).

Skills için hâlâ eksik olanlar (bkz. [Skills](/tr/tools/skills)):

- **Karar verme:** prompt içinde skills listelendiğinde, ajan doğru skill'i seçiyor mu (ya da ilgisiz olanlardan kaçınıyor mu)?
- **Uyumluluk:** ajan kullanmadan önce `SKILL.md` dosyasını okuyor ve gerekli adımları/argümanları izliyor mu?
- **İş akışı sözleşmeleri:** tool sırasını, oturum geçmişi devrini ve sandbox sınırlarını doğrulayan çok turlu senaryolar.

Gelecekteki değerlendirmeler önce deterministik kalmalıdır:

- Tool çağrılarını + sırasını, skill dosyası okumalarını ve oturum kablolamasını doğrulamak için sahte sağlayıcılar kullanan bir senaryo çalıştırıcısı.
- Skill odaklı küçük bir senaryo paketi (kullan vs kaçın, geçitleme, prompt injection).
- İsteğe bağlı live değerlendirmeler (opt-in, env kontrollü), ancak CI-güvenli paket hazır olduktan sonra.

## Sözleşme testleri (plugin ve channel yapısı)

Sözleşme testleri, kayıtlı her plugin ve channel'ın arayüz sözleşmesine uyduğunu doğrular. Keşfedilen tüm plugin'ler üzerinde dolaşır ve bir dizi yapı ve davranış doğrulaması çalıştırır. Varsayılan `pnpm test` unit hattı bu paylaşılan seam ve smoke dosyalarını bilerek atlar; paylaşılan channel veya sağlayıcı yüzeylerine dokunduğunuzda sözleşme komutlarını açıkça çalıştırın.

### Komutlar

- Tüm sözleşmeler: `pnpm test:contracts`
- Yalnızca channel sözleşmeleri: `pnpm test:contracts:channels`
- Yalnızca sağlayıcı sözleşmeleri: `pnpm test:contracts:plugins`

### Channel sözleşmeleri

`src/channels/plugins/contracts/*.contract.test.ts` konumundadır:

- **plugin** - Temel plugin yapısı (id, name, capabilities)
- **setup** - Kurulum wizard sözleşmesi
- **session-binding** - Oturum bağlama davranışı
- **outbound-payload** - Mesaj payload yapısı
- **inbound** - Gelen mesaj işleme
- **actions** - Channel action handler'ları
- **threading** - Thread ID işleme
- **directory** - Dizin/roster API
- **group-policy** - Grup politikası uygulaması

### Sağlayıcı durum sözleşmeleri

`src/plugins/contracts/*.contract.test.ts` konumundadır.

- **status** - Channel durum probe'ları
- **registry** - Plugin registry yapısı

### Sağlayıcı sözleşmeleri

`src/plugins/contracts/*.contract.test.ts` konumundadır:

- **auth** - Auth akışı sözleşmesi
- **auth-choice** - Auth seçimi/seçiciliği
- **catalog** - Model katalog API'si
- **discovery** - Plugin keşfi
- **loader** - Plugin yükleme
- **runtime** - Sağlayıcı çalışma zamanı
- **shape** - Plugin yapısı/arayüzü
- **wizard** - Kurulum wizard'ı

### Ne zaman çalıştırılmalı

- plugin-sdk export'larını veya alt yollarını değiştirdikten sonra
- Bir channel veya sağlayıcı plugin'i ekledikten ya da değiştirdikten sonra
- Plugin kaydını veya keşfini yeniden düzenledikten sonra

Sözleşme testleri CI içinde çalışır ve gerçek API anahtarları gerektirmez.

## Regresyon ekleme (rehberlik)

Live içinde keşfedilen bir sağlayıcı/model sorununu düzelttiğinizde:

- Mümkünse CI-güvenli bir regresyon ekleyin (sağlayıcıyı mock/stub yapın veya tam request-shape dönüşümünü yakalayın)
- Doğası gereği yalnızca live ise (rate limit'ler, auth politikaları), live testi dar ve env değişkenleriyle opt-in tutun
- Hatayı yakalayan en küçük katmanı hedeflemeyi tercih edin:
  - sağlayıcı request dönüşümü/replay hatası → doğrudan models testi
  - gateway session/history/tool pipeline hatası → gateway live smoke veya CI-güvenli gateway mock testi
- SecretRef traversal guardrail:
  - `src/secrets/exec-secret-ref-id-parity.test.ts`, kayıt metadata'sından (`listSecretTargetRegistryEntries()`) SecretRef sınıfı başına örneklenmiş bir hedef türetir, ardından traversal-segment exec kimliklerinin reddedildiğini doğrular.
  - `src/secrets/target-registry-data.ts` içinde yeni bir `includeInPlan` SecretRef hedef ailesi eklerseniz, o testteki `classifyTargetClass` işlevini güncelleyin. Test, yeni sınıfların sessizce atlanamaması için sınıflandırılmamış hedef kimliklerinde bilerek başarısız olur.
