---
read_when:
    - Testleri yerel olarak veya CI içinde çalıştırma
    - Model/sağlayıcı hataları için regresyonlar ekleme
    - Gateway + agent davranışında hata ayıklama
summary: 'Test kiti: birim/e2e/canlı paketleri, Docker çalıştırıcıları ve her testin neleri kapsadığı'
title: Test etme
x-i18n:
    generated_at: "2026-04-15T14:40:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec3632cafa1f38b27510372391b84af744266df96c58f7fac98aa03763465db8
    source_path: help/testing.md
    workflow: 15
---

# Test etme

OpenClaw’ın üç Vitest paketi (birim/entegrasyon, e2e, canlı) ve küçük bir Docker çalıştırıcı kümesi vardır.

Bu belge, “nasıl test ettiğimizi” anlatan bir kılavuzdur:

- Her paketin neleri kapsadığı (ve bilerek neleri _kapsamadığı_)
- Yaygın iş akışları için hangi komutların çalıştırılacağı (yerel, push öncesi, hata ayıklama)
- Canlı testlerin kimlik bilgilerini nasıl bulduğu ve model/sağlayıcıları nasıl seçtiği
- Gerçek dünya model/sağlayıcı sorunları için regresyonların nasıl ekleneceği

## Hızlı başlangıç

Çoğu gün:

- Tam geçit (push öncesinde beklenir): `pnpm build && pnpm check && pnpm test`
- Kaynakları geniş bir makinede daha hızlı yerel tam paket çalıştırması: `pnpm test:max`
- Doğrudan Vitest izleme döngüsü: `pnpm test:watch`
- Doğrudan dosya hedefleme artık eklenti/kanal yollarını da yönlendirir: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Tek bir hata üzerinde yineleme yapıyorsanız önce hedefli çalıştırmaları tercih edin.
- Docker destekli QA sitesi: `pnpm qa:lab:up`
- Linux VM destekli QA hattı: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Testlere dokunduğunuzda veya ek güven istediğinizde:

- Kapsam geçidi: `pnpm test:coverage`
- E2E paketi: `pnpm test:e2e`

Gerçek sağlayıcıları/modelleri hata ayıklarken (gerçek kimlik bilgileri gerekir):

- Canlı paket (modeller + gateway araç/görüntü yoklamaları): `pnpm test:live`
- Tek bir canlı dosyayı sessizce hedefleme: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

İpucu: yalnızca tek bir başarısız duruma ihtiyacınız olduğunda, canlı testleri aşağıda açıklanan allowlist ortam değişkenleriyle daraltmayı tercih edin.

## QA’ye özgü çalıştırıcılar

QA-lab gerçekçiliğine ihtiyaç duyduğunuzda bu komutlar ana test paketlerinin yanında yer alır:

- `pnpm openclaw qa suite`
  - Depo destekli QA senaryolarını doğrudan host üzerinde çalıştırır.
  - Varsayılan olarak seçilen birden çok senaryoyu, yalıtılmış gateway worker’larıyla paralel çalıştırır; en fazla 64 worker ya da seçilen senaryo sayısı kadar. Worker sayısını ayarlamak için `--concurrency <count>`, eski seri hat için ise `--concurrency 1` kullanın.
- `pnpm openclaw qa suite --runner multipass`
  - Aynı QA paketini tek kullanımlık bir Multipass Linux VM içinde çalıştırır.
  - Host üzerindeki `qa suite` ile aynı senaryo seçimi davranışını korur.
  - `qa suite` ile aynı sağlayıcı/model seçim bayraklarını yeniden kullanır.
  - Canlı çalıştırmalar, misafir için pratik olan desteklenen QA kimlik doğrulama girdilerini iletir:
    ortam tabanlı sağlayıcı anahtarları, QA canlı sağlayıcı yapılandırma yolu ve varsa `CODEX_HOME`.
  - Çıktı dizinleri depo kökü altında kalmalıdır; böylece misafir, bağlanan çalışma alanı üzerinden geri yazabilir.
  - Normal QA raporu + özeti ve Multipass günlüklerini `.artifacts/qa-e2e/...` altına yazar.
- `pnpm qa:lab:up`
  - Operatör tarzı QA çalışmaları için Docker destekli QA sitesini başlatır.
- `pnpm openclaw qa matrix`
  - Matrix canlı QA hattını, tek kullanımlık Docker destekli bir Tuwunel homeserver’a karşı çalıştırır.
  - Bu QA hostu bugün yalnızca repo/geliştirme içindir. Paketlenmiş OpenClaw kurulumları `qa-lab` göndermez; bu yüzden `openclaw qa` sunmazlar.
  - Depo checkout’ları paketlenmiş çalıştırıcıyı doğrudan yükler; ayrı bir eklenti kurulum adımı gerekmez.
  - Üç geçici Matrix kullanıcısı (`driver`, `sut`, `observer`) ve bir özel oda hazırlar, ardından gerçek Matrix eklentisi SUT taşıması olarak kullanılarak bir QA gateway alt süreci başlatır.
  - Varsayılan olarak sabitlenmiş kararlı Tuwunel görüntüsü `ghcr.io/matrix-construct/tuwunel:v1.5.1` kullanılır. Farklı bir görüntüyü test etmeniz gerekiyorsa `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` ile geçersiz kılın.
  - Matrix, hattın yerel olarak tek kullanımlık kullanıcılar hazırlaması nedeniyle paylaşılan kimlik bilgisi kaynağı bayraklarını sunmaz.
  - Bir Matrix QA raporu, özeti ve gözlemlenen olaylar artifact’ını `.artifacts/qa-e2e/...` altına yazar.
- `pnpm openclaw qa telegram`
  - Telegram canlı QA hattını, ortamdaki driver ve SUT bot belirteçlerini kullanarak gerçek bir özel gruba karşı çalıştırır.
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` ve `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` gerektirir. Grup kimliği sayısal Telegram sohbet kimliği olmalıdır.
  - Paylaşılan havuzlanmış kimlik bilgileri için `--credential-source convex` destekler. Varsayılan olarak env modunu kullanın ya da havuzlanmış kiralamaları etkinleştirmek için `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` ayarlayın.
  - Aynı özel grupta bulunan iki farklı bot gerektirir; SUT botunun bir Telegram kullanıcı adını açığa çıkarması gerekir.
  - Kararlı bottan-bota gözlem için `@BotFather` içinde her iki bot için de Bot-to-Bot Communication Mode’u etkinleştirin ve driver botunun grup bot trafiğini gözlemleyebildiğinden emin olun.
  - Bir Telegram QA raporu, özeti ve gözlemlenen iletiler artifact’ını `.artifacts/qa-e2e/...` altına yazar.

Canlı taşıma hatları, yeni taşıma türlerinin sapmaması için tek bir standart sözleşmeyi paylaşır:

`qa-channel`, geniş sentetik QA paketi olmaya devam eder ve canlı taşıma kapsama matrisinin parçası değildir.

| Hat      | Canary | Mention gating | Allowlist block | Üst düzey yanıt | Yeniden başlatma sonrası sürdürme | Thread takibi | Thread yalıtımı | Tepki gözlemi | Yardım komutu |
| -------- | ------ | -------------- | --------------- | --------------- | --------------------------------- | ------------- | --------------- | -------------- | ------------- |
| Matrix   | x      | x              | x               | x               | x                                 | x             | x               | x              |               |
| Telegram | x      |                |                 |                 |                                   |               |                 |                | x             |

### Convex aracılığıyla paylaşılan Telegram kimlik bilgileri (v1)

`openclaw qa telegram` için `--credential-source convex` (veya `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`) etkinleştirildiğinde, QA lab Convex destekli bir havuzdan özel bir kiralama alır, hat çalışırken bu kiralamaya Heartbeat gönderir ve kapanışta kiralamayı serbest bırakır.

Referans Convex proje iskeleti:

- `qa/convex-credential-broker/`

Gerekli ortam değişkenleri:

- `OPENCLAW_QA_CONVEX_SITE_URL` (örneğin `https://your-deployment.convex.site`)
- Seçilen rol için bir gizli anahtar:
  - `maintainer` için `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` için `OPENCLAW_QA_CONVEX_SECRET_CI`
- Kimlik bilgisi rol seçimi:
  - CLI: `--credential-role maintainer|ci`
  - Ortam varsayılanı: `OPENCLAW_QA_CREDENTIAL_ROLE` (varsayılan `maintainer`)

İsteğe bağlı ortam değişkenleri:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (varsayılan `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (varsayılan `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (varsayılan `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (varsayılan `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (varsayılan `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (isteğe bağlı izleme kimliği)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1`, yalnızca yerel geliştirme için loopback `http://` Convex URL’lerine izin verir.

Normal çalışmada `OPENCLAW_QA_CONVEX_SITE_URL`, `https://` kullanmalıdır.

Bakımcı yönetici komutları (havuz ekleme/kaldırma/listeleme) özellikle
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` gerektirir.

Bakımcılar için CLI yardımcıları:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Betikler ve CI yardımcı programlarında makine tarafından okunabilir çıktı için `--json` kullanın.

Varsayılan uç nokta sözleşmesi (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - İstek: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Başarı: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Tükenmiş/yeniden denenebilir: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - İstek: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Başarı: `{ status: "ok" }` (veya boş `2xx`)
- `POST /release`
  - İstek: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Başarı: `{ status: "ok" }` (veya boş `2xx`)
- `POST /admin/add` (yalnızca maintainer gizli anahtarı)
  - İstek: `{ kind, actorId, payload, note?, status? }`
  - Başarı: `{ status: "ok", credential }`
- `POST /admin/remove` (yalnızca maintainer gizli anahtarı)
  - İstek: `{ credentialId, actorId }`
  - Başarı: `{ status: "ok", changed, credential }`
  - Etkin kiralama koruması: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (yalnızca maintainer gizli anahtarı)
  - İstek: `{ kind?, status?, includePayload?, limit? }`
  - Başarı: `{ status: "ok", credentials, count }`

Telegram türü için yük şekli:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId`, sayısal bir Telegram sohbet kimliği dizgesi olmalıdır.
- `admin/add`, `kind: "telegram"` için bu şekli doğrular ve hatalı biçimlendirilmiş yükleri reddeder.

### QA’ye kanal ekleme

Markdown QA sistemine kanal eklemek tam olarak iki şey gerektirir:

1. Kanal için bir taşıma adaptörü.
2. Kanal sözleşmesini çalıştıran bir senaryo paketi.

Paylaşılan `qa-lab` host akışı sahiplenebiliyorsa yeni bir üst düzey QA komut kökü eklemeyin.

`qa-lab`, paylaşılan host mekaniklerine sahiptir:

- `openclaw qa` komut kökü
- paket başlatma ve kapatma
- worker eşzamanlılığı
- artifact yazımı
- rapor oluşturma
- senaryo yürütme
- eski `qa-channel` senaryoları için uyumluluk takma adları

Çalıştırıcı eklentileri taşıma sözleşmesine sahiptir:

- `openclaw qa <runner>` öğesinin paylaşılan `qa` kökü altına nasıl bağlandığı
- gateway’in bu taşıma için nasıl yapılandırıldığı
- hazır olma durumunun nasıl denetlendiği
- gelen olayların nasıl enjekte edildiği
- giden iletilerin nasıl gözlemlendiği
- dökümlerin ve normalize edilmiş taşıma durumunun nasıl açığa çıkarıldığı
- taşıma destekli eylemlerin nasıl yürütüldüğü
- taşıma türüne özgü sıfırlama veya temizliğin nasıl ele alındığı

Yeni bir kanal için asgari benimseme eşiği şudur:

1. Paylaşılan `qa` kökünün sahibi olarak `qa-lab`’ı koruyun.
2. Taşıma çalıştırıcısını paylaşılan `qa-lab` host dikişinde uygulayın.
3. Taşıma türüne özgü mekanikleri çalıştırıcı eklentisi veya kanal harness’i içinde tutun.
4. Çalıştırıcıyı rakip bir kök komut kaydetmek yerine `openclaw qa <runner>` olarak bağlayın.
   Çalıştırıcı eklentileri `openclaw.plugin.json` içinde `qaRunners` bildirmeli ve `runtime-api.ts` içinden eşleşen bir `qaRunnerCliRegistrations` dizisi dışa aktarmalıdır.
   `runtime-api.ts` dosyasını hafif tutun; lazy CLI ve çalıştırıcı yürütmesi ayrı giriş noktalarının arkasında kalmalıdır.
5. `qa/scenarios/` altında markdown senaryoları yazın veya uyarlayın.
6. Yeni senaryolar için genel senaryo yardımcılarını kullanın.
7. Depo kasıtlı bir geçiş yapmıyorsa mevcut uyumluluk takma adlarını çalışır durumda tutun.

Karar kuralı katıdır:

- Davranış `qa-lab` içinde bir kez ifade edilebiliyorsa, onu `qa-lab` içine koyun.
- Davranış tek bir kanal taşımasına bağlıysa, onu ilgili çalıştırıcı eklentisi veya eklenti harness’i içinde tutun.
- Bir senaryo, birden fazla kanalın kullanabileceği yeni bir yetenek gerektiriyorsa, `suite.ts` içinde kanala özgü bir dal yerine genel bir yardımcı ekleyin.
- Bir davranış yalnızca tek bir taşıma için anlamlıysa, senaryoyu taşıma türüne özgü tutun ve bunu senaryo sözleşmesinde açıkça belirtin.

Yeni senaryolar için tercih edilen genel yardımcı adları şunlardır:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Mevcut senaryolar için uyumluluk takma adları kullanılabilir olmaya devam eder; bunlar arasında şunlar bulunur:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Yeni kanal çalışmaları genel yardımcı adlarını kullanmalıdır.
Uyumluluk takma adları, tek seferlik bir geçiş gününü önlemek için vardır; yeni senaryo yazımı için model olarak değil.

## Test paketleri (ne nerede çalışır)

Paketleri “artan gerçekçilik” (ve artan oynaklık/maliyet) olarak düşünün:

### Birim / entegrasyon (varsayılan)

- Komut: `pnpm test`
- Yapılandırma: mevcut kapsamlı Vitest projeleri üzerinde on sıralı shard çalıştırması (`vitest.full-*.config.ts`)
- Dosyalar: `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` altındaki çekirdek/birim envanterleri ve `vitest.unit.config.ts` kapsamındaki allowlist’e alınmış `ui` node testleri
- Kapsam:
  - Saf birim testleri
  - Süreç içi entegrasyon testleri (gateway kimlik doğrulama, yönlendirme, araçlar, ayrıştırma, yapılandırma)
  - Bilinen hatalar için deterministik regresyonlar
- Beklentiler:
  - CI’da çalışır
  - Gerçek anahtarlar gerekmez
  - Hızlı ve kararlı olmalıdır
- Projeler notu:
  - Hedeflenmemiş `pnpm test` artık tek büyük bir doğal kök-proje süreci yerine on bir küçük shard yapılandırması (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) çalıştırır. Bu, yüklü makinelerde tepe RSS’yi azaltır ve auto-reply/extension çalışmalarının ilgisiz paketleri aç bırakmasını önler.
  - `pnpm test --watch`, çok shard’lı bir izleme döngüsü pratik olmadığından, hâlâ doğal kök `vitest.config.ts` proje grafiğini kullanır.
  - `pnpm test`, `pnpm test:watch` ve `pnpm test:perf:imports`, açık dosya/dizin hedeflerini önce kapsamlı hatlar üzerinden yönlendirir; böylece `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`, tam kök proje başlatma maliyetini ödemez.
  - `pnpm test:changed`, fark yalnızca yönlendirilebilir kaynak/test dosyalarına dokunduğunda değişen git yollarını aynı kapsamlı hatlara genişletir; yapılandırma/kurulum düzenlemeleri ise yine geniş kök-proje yeniden çalıştırmasına geri döner.
  - Agent’lar, komutlar, eklentiler, auto-reply yardımcıları, `plugin-sdk` ve benzeri saf yardımcı alanlardaki import bakımından hafif birim testleri, `test/setup-openclaw-runtime.ts` dosyasını atlayan `unit-fast` hattı üzerinden yönlendirilir; durum bilgili/çalışma zamanı ağır dosyalar mevcut hatlarda kalır.
  - Seçilmiş `plugin-sdk` ve `commands` yardımcı kaynak dosyaları da, changed modundaki çalıştırmaları bu hafif hatlardaki açık kardeş testlere eşler; böylece yardımcı düzenlemeleri, ilgili dizin için tüm ağır paketi yeniden çalıştırmaktan kaçınır.
  - `auto-reply` artık üç özel kovaya sahiptir: üst düzey çekirdek yardımcılar, üst düzey `reply.*` entegrasyon testleri ve `src/auto-reply/reply/**` alt ağacı. Bu, en ağır yanıt harness çalışmasını ucuz durum/parça/token testlerinden uzak tutar.
- Gömülü çalıştırıcı notu:
  - Mesaj-aracı keşfi girdilerini veya Compaction çalışma zamanı bağlamını değiştirdiğinizde, her iki kapsama düzeyini de koruyun.
  - Saf yönlendirme/normalleştirme sınırları için odaklı yardımcı regresyonları ekleyin.
  - Ayrıca gömülü çalıştırıcı entegrasyon paketlerini de sağlıklı tutun:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` ve
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Bu paketler, kapsamlı kimliklerin ve Compaction davranışının hâlâ gerçek `run.ts` / `compact.ts` yolları üzerinden aktığını doğrular; yalnızca yardımcı testleri bu entegrasyon yolları için yeterli bir ikame değildir.
- Havuz notu:
  - Temel Vitest yapılandırması artık varsayılan olarak `threads` kullanır.
  - Paylaşılan Vitest yapılandırması ayrıca `isolate: false` sabitler ve yalıtımsız çalıştırıcıyı kök projeler, e2e ve canlı yapılandırmalar genelinde kullanır.
  - Kök UI hattı `jsdom` kurulumunu ve optimizer’ını korur, ancak artık paylaşılan yalıtımsız çalıştırıcı üzerinde çalışır.
  - Her `pnpm test` shard’ı, paylaşılan Vitest yapılandırmasından aynı `threads` + `isolate: false` varsayılanlarını devralır.
  - Paylaşılan `scripts/run-vitest.mjs` başlatıcısı artık büyük yerel çalıştırmalar sırasında V8 derleme çalkantısını azaltmak için Vitest alt Node süreçlerine varsayılan olarak `--no-maglev` de ekler. Stok V8 davranışıyla karşılaştırma yapmanız gerekiyorsa `OPENCLAW_VITEST_ENABLE_MAGLEV=1` ayarlayın.
- Hızlı yerel yineleme notu:
  - `pnpm test:changed`, değişen yollar daha küçük bir pakete temizce eşleniyorsa kapsamlı hatlar üzerinden yönlendirme yapar.
  - `pnpm test:max` ve `pnpm test:changed:max`, yalnızca daha yüksek worker sınırıyla aynı yönlendirme davranışını korur.
  - Yerel worker otomatik ölçeklendirmesi artık kasıtlı olarak daha tutucudur ve host yük ortalaması zaten yüksekken de geri çekilir; böylece birden çok eşzamanlı Vitest çalıştırması varsayılan olarak daha az zarar verir.
  - Temel Vitest yapılandırması, test kablolaması değiştiğinde changed modundaki yeniden çalıştırmaların doğru kalması için projeleri/yapılandırma dosyalarını `forceRerunTriggers` olarak işaretler.
  - Yapılandırma, desteklenen host’larda `OPENCLAW_VITEST_FS_MODULE_CACHE` özelliğini etkin tutar; doğrudan profil oluşturma için açık bir önbellek konumu istiyorsanız `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` ayarlayın.
- Performans hata ayıklama notu:
  - `pnpm test:perf:imports`, Vitest import süresi raporlamasını ve import dökümü çıktısını etkinleştirir.
  - `pnpm test:perf:imports:changed`, aynı profil görünümünü `origin/main` sonrasındaki değişen dosyalarla sınırlar.
- `pnpm test:perf:changed:bench -- --ref <git-ref>`, yönlendirilmiş `test:changed` çalışmasını o commit edilmiş fark için doğal kök-proje yoluyla karşılaştırır ve duvar saati süresi ile macOS azami RSS’yi yazdırır.
- `pnpm test:perf:changed:bench -- --worktree`, değişen dosya listesini `scripts/test-projects.mjs` ve kök Vitest yapılandırması üzerinden yönlendirerek mevcut kirli ağacı kıyaslar.
  - `pnpm test:perf:profile:main`, Vitest/Vite başlangıcı ve dönüştürme yükü için ana iş parçacığı CPU profili yazar.
  - `pnpm test:perf:profile:runner`, dosya paralelliği devre dışıyken birim paketi için çalıştırıcı CPU+heap profilleri yazar.

### E2E (gateway smoke)

- Komut: `pnpm test:e2e`
- Yapılandırma: `vitest.e2e.config.ts`
- Dosyalar: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Çalışma zamanı varsayılanları:
  - Deponun geri kalanıyla eşleşecek şekilde Vitest `threads` ve `isolate: false` kullanır.
  - Uyarlanabilir worker’lar kullanır (CI: en fazla 2, yerel: varsayılan 1).
  - Konsol G/Ç yükünü azaltmak için varsayılan olarak sessiz modda çalışır.
- Yararlı geçersiz kılmalar:
  - Worker sayısını zorlamak için `OPENCLAW_E2E_WORKERS=<n>` (üst sınır 16).
  - Ayrıntılı konsol çıktısını yeniden etkinleştirmek için `OPENCLAW_E2E_VERBOSE=1`.
- Kapsam:
  - Çok örnekli gateway uçtan uca davranışı
  - WebSocket/HTTP yüzeyleri, Node eşleştirme ve daha ağır ağ iletişimi
- Beklentiler:
  - CI’da çalışır (iş hattında etkinleştirildiğinde)
  - Gerçek anahtarlar gerekmez
  - Birim testlerinden daha fazla hareketli parça içerir (daha yavaş olabilir)

### E2E: OpenShell backend smoke

- Komut: `pnpm test:e2e:openshell`
- Dosya: `test/openshell-sandbox.e2e.test.ts`
- Kapsam:
  - Host üzerinde Docker aracılığıyla yalıtılmış bir OpenShell gateway başlatır
  - Geçici yerel bir Dockerfile’dan sandbox oluşturur
  - OpenClaw’ın OpenShell backend’ini gerçek `sandbox ssh-config` + SSH exec üzerinden çalıştırır
  - Sandbox fs bridge üzerinden uzak-kanonik dosya sistemi davranışını doğrular
- Beklentiler:
  - Yalnızca isteğe bağlıdır; varsayılan `pnpm test:e2e` çalıştırmasının parçası değildir
  - Yerel bir `openshell` CLI ve çalışan bir Docker daemon’u gerektirir
  - Yalıtılmış `HOME` / `XDG_CONFIG_HOME` kullanır, ardından test gateway’ini ve sandbox’ı yok eder
- Yararlı geçersiz kılmalar:
  - Daha geniş e2e paketini elle çalıştırırken testi etkinleştirmek için `OPENCLAW_E2E_OPENSHELL=1`
  - Varsayılan olmayan bir CLI ikilisine veya wrapper betiğine işaret etmek için `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Canlı (gerçek sağlayıcılar + gerçek modeller)

- Komut: `pnpm test:live`
- Yapılandırma: `vitest.live.config.ts`
- Dosyalar: `src/**/*.live.test.ts`
- Varsayılan: `pnpm test:live` tarafından **etkinleştirilir** (`OPENCLAW_LIVE_TEST=1` ayarlar)
- Kapsam:
  - “Bu sağlayıcı/model bugün gerçek kimlik bilgileriyle gerçekten çalışıyor mu?”
  - Sağlayıcı biçim değişikliklerini, araç çağırma tuhaflıklarını, kimlik doğrulama sorunlarını ve rate limit davranışını yakalar
- Beklentiler:
  - Tasarım gereği CI-kararlı değildir (gerçek ağlar, gerçek sağlayıcı politikaları, kotalar, kesintiler)
  - Maliyetlidir / rate limit kullanır
  - “Her şeyi” çalıştırmak yerine daraltılmış alt kümeleri çalıştırmak tercih edilir
- Canlı çalıştırmalar, eksik API anahtarlarını almak için `~/.profile` kaynağını kullanır.
- Varsayılan olarak canlı çalıştırmalar yine de `HOME` dizinini yalıtır ve yapılandırma/kimlik doğrulama materyalini geçici bir test home dizinine kopyalar; böylece birim fixture’ları gerçek `~/.openclaw` dizininizi değiştiremez.
- Canlı testlerin gerçek home dizininizi kullanmasını özellikle istediğinizde yalnızca `OPENCLAW_LIVE_USE_REAL_HOME=1` ayarlayın.
- `pnpm test:live` artık varsayılan olarak daha sessiz bir mod kullanır: `[live] ...` ilerleme çıktısını korur, ancak ek `~/.profile` bildirimini bastırır ve gateway bootstrap günlükleri/Bonjour gürültüsünü susturur. Tüm başlangıç günlüklerini geri istiyorsanız `OPENCLAW_LIVE_TEST_QUIET=0` ayarlayın.
- API anahtarı rotasyonu (sağlayıcıya özgü): virgül/noktalı virgül biçimiyle `*_API_KEYS` veya `*_API_KEY_1`, `*_API_KEY_2` ayarlayın (örneğin `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ya da canlıya özgü geçersiz kılma olarak `OPENCLAW_LIVE_*_KEY`; testler rate limit yanıtlarında yeniden dener.
- İlerleme/Heartbeat çıktısı:
  - Canlı paketler artık ilerleme satırlarını stderr’e yazar; böylece uzun sağlayıcı çağrıları, Vitest konsol yakalaması sessizken bile görünür biçimde etkin kalır.
  - `vitest.live.config.ts`, sağlayıcı/gateway ilerleme satırlarının canlı çalıştırmalar sırasında hemen akması için Vitest konsol yakalamasını devre dışı bırakır.
  - Doğrudan model Heartbeat’lerini `OPENCLAW_LIVE_HEARTBEAT_MS` ile ayarlayın.
  - Gateway/yoklama Heartbeat’lerini `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` ile ayarlayın.

## Hangi paketi çalıştırmalıyım?

Bu karar tablosunu kullanın:

- Mantık/test düzenliyorsanız: `pnpm test` çalıştırın (çok şey değiştirdiyseniz `pnpm test:coverage` da)
- Gateway ağ iletişimi / WS protokolü / eşleştirmeye dokunuyorsanız: `pnpm test:e2e` ekleyin
- “Botum çalışmıyor” / sağlayıcıya özgü arızalar / araç çağırma sorunlarını hata ayıklıyorsanız: daraltılmış bir `pnpm test:live` çalıştırın

## Canlı: Android Node yetenek taraması

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Betik: `pnpm android:test:integration`
- Amaç: bağlı bir Android Node tarafından şu anda **reklamı yapılan her komutu** çağırmak ve komut sözleşmesi davranışını doğrulamak.
- Kapsam:
  - Önkoşullu/manuel kurulum (paket uygulamayı kurmaz/çalıştırmaz/eşleştirmez).
  - Seçilen Android Node için komut bazında gateway `node.invoke` doğrulaması.
- Gerekli ön kurulum:
  - Android uygulaması zaten bağlı ve gateway ile eşleştirilmiş olmalı.
  - Uygulama ön planda tutulmalı.
  - Geçmesini beklediğiniz yetenekler için izinler/yakalama onayı verilmiş olmalı.
- İsteğe bağlı hedef geçersiz kılmaları:
  - `OPENCLAW_ANDROID_NODE_ID` veya `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Tüm Android kurulum ayrıntıları: [Android Uygulaması](/tr/platforms/android)

## Canlı: model smoke (profil anahtarları)

Canlı testler, arızaları yalıtabilmek için iki katmana ayrılmıştır:

- “Doğrudan model”, sağlayıcının/modelin verilen anahtarla en azından yanıt verebildiğini söyler.
- “Gateway smoke”, tam gateway+agent işlem hattının bu model için çalıştığını söyler (oturumlar, geçmiş, araçlar, sandbox ilkesi vb.).

### Katman 1: Doğrudan model tamamlama (gateway yok)

- Test: `src/agents/models.profiles.live.test.ts`
- Amaç:
  - Keşfedilen modelleri numaralandırmak
  - Kimlik bilgilerinizin olduğu modelleri seçmek için `getApiKeyForModel` kullanmak
  - Model başına küçük bir tamamlama çalıştırmak (ve gerektiğinde hedefli regresyonlar)
- Nasıl etkinleştirilir:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
- Bu paketi gerçekten çalıştırmak için `OPENCLAW_LIVE_MODELS=modern` (veya `all`, `modern` takma adı) ayarlayın; aksi takdirde `pnpm test:live` odağını gateway smoke üzerinde tutmak için atlanır
- Modeller nasıl seçilir:
  - Modern allowlist’i çalıştırmak için `OPENCLAW_LIVE_MODELS=modern` (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all`, modern allowlist için bir takma addır
  - veya `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (virgülle ayrılmış allowlist)
  - Modern/tüm taramalar varsayılan olarak seçilmiş yüksek sinyalli bir üst sınır kullanır; kapsamlı bir modern tarama için `OPENCLAW_LIVE_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
- Sağlayıcılar nasıl seçilir:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (virgülle ayrılmış allowlist)
- Anahtarlar nereden gelir:
  - Varsayılan olarak: profil deposu ve env geri dönüşleri
  - Yalnızca **profil deposunu** zorunlu kılmak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` ayarlayın
- Bunun var olma nedeni:
  - “Sağlayıcı API’si bozuk / anahtar geçersiz” ile “gateway agent işlem hattı bozuk” durumlarını ayırır
  - Küçük, yalıtılmış regresyonları içerir (örnek: OpenAI Responses/Codex Responses reasoning replay + tool-call akışları)

### Katman 2: Gateway + geliştirme agent smoke (`@openclaw`’ın gerçekten yaptığı şey)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Amaç:
  - Süreç içi bir gateway başlatmak
  - Bir `agent:dev:*` oturumu oluşturmak/yamalamak (çalıştırma başına model geçersiz kılma)
  - Anahtarlı modeller üzerinde yinelemek ve şunları doğrulamak:
    - “anlamlı” yanıt (araç yok)
    - gerçek bir araç çağrısının çalışması (`read` probe)
    - isteğe bağlı ek araç probe’ları (`exec+read` probe)
    - OpenAI regresyon yollarının (yalnızca tool-call → follow-up) çalışmaya devam etmesi
- Probe ayrıntıları (böylece arızaları hızlıca açıklayabilirsiniz):
  - `read` probe: test, çalışma alanına bir nonce dosyası yazar ve agent’tan bunu `read` etmesini ve nonce’u geri yankılamasını ister.
  - `exec+read` probe: test, agent’tan nonce’u geçici bir dosyaya `exec` ile yazmasını, ardından geri `read` etmesini ister.
  - image probe: test, oluşturulmuş bir PNG’yi (kedi + rastgeleleştirilmiş kod) ekler ve modelin `cat <CODE>` döndürmesini bekler.
  - Uygulama başvurusu: `src/gateway/gateway-models.profiles.live.test.ts` ve `src/gateway/live-image-probe.ts`.
- Nasıl etkinleştirilir:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
- Modeller nasıl seçilir:
  - Varsayılan: modern allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all`, modern allowlist için bir takma addır
  - Ya da daraltmak için `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (veya virgüllü liste) ayarlayın
  - Modern/tüm gateway taramaları varsayılan olarak seçilmiş yüksek sinyalli bir üst sınır kullanır; kapsamlı bir modern tarama için `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`, daha küçük bir üst sınır için pozitif bir sayı ayarlayın.
- Sağlayıcılar nasıl seçilir (“OpenRouter her şey” yaklaşımından kaçınmak için):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (virgülle ayrılmış allowlist)
- Tool + image probe’ları bu canlı testte her zaman açıktır:
  - `read` probe + `exec+read` probe (araç stresi)
  - image probe, model image input desteğini ilan ettiğinde çalışır
  - Akış (üst düzey):
    - Test, “CAT” + rastgele kod içeren küçük bir PNG oluşturur (`src/gateway/live-image-probe.ts`)
    - Bunu `agent` üzerinden `attachments: [{ mimeType: "image/png", content: "<base64>" }]` ile gönderir
    - Gateway, ekleri `images[]` içine ayrıştırır (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Gömülü agent, modele çok modlu bir kullanıcı mesajı iletir
    - Doğrulama: yanıt `cat` + kodu içerir (OCR toleransı: küçük hatalara izin verilir)

İpucu: makinenizde neyi test edebileceğinizi (ve tam `provider/model` kimliklerini) görmek için şunu çalıştırın:

```bash
openclaw models list
openclaw models list --json
```

## Canlı: CLI backend smoke (Claude, Codex, Gemini veya diğer yerel CLI’lar)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Amaç: varsayılan yapılandırmanıza dokunmadan yerel bir CLI backend kullanarak Gateway + agent işlem hattını doğrulamak.
- Backend’e özgü smoke varsayılanları, sahibi olan eklentinin `cli-backend.ts` tanımıyla birlikte bulunur.
- Etkinleştirme:
  - `pnpm test:live` (veya Vitest doğrudan çağrılıyorsa `OPENCLAW_LIVE_TEST=1`)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Varsayılanlar:
  - Varsayılan sağlayıcı/model: `claude-cli/claude-sonnet-4-6`
  - Komut/argüman/image davranışı, sahibi olan CLI backend eklentisi meta verilerinden gelir.
- Geçersiz kılmalar (isteğe bağlı):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - Gerçek bir image eki göndermek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` (yollar prompt içine enjekte edilir).
  - Image dosya yollarını prompt enjeksiyonu yerine CLI argümanları olarak geçirmek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`.
  - `IMAGE_ARG` ayarlandığında image argümanlarının nasıl geçirileceğini kontrol etmek için `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (veya `"list"`).
  - İkinci bir tur göndermek ve resume akışını doğrulamak için `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`.
  - Varsayılan Claude Sonnet -> Opus aynı-oturum sürekliliği probe’unu devre dışı bırakmak için `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` (seçilen model bir geçiş hedefini desteklediğinde zorla açmak için `1` ayarlayın).

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
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Notlar:

- Docker çalıştırıcısı `scripts/test-live-cli-backend-docker.sh` konumundadır.
- Canlı CLI-backend smoke testini depo Docker image’ı içinde root olmayan `node` kullanıcısı olarak çalıştırır.
- CLI smoke meta verilerini sahibi olan eklentiden çözer, ardından eşleşen Linux CLI paketini (`@anthropic-ai/claude-code`, `@openai/codex` veya `@google/gemini-cli`) `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (varsayılan: `~/.cache/openclaw/docker-cli-tools`) içindeki önbelleklenmiş yazılabilir bir prefix’e kurar.
- `pnpm test:docker:live-cli-backend:claude-subscription`, taşınabilir Claude Code abonelik OAuth’u gerektirir; bunun için ya `claudeAiOauth.subscriptionType` içeren `~/.claude/.credentials.json` ya da `claude setup-token` içinden `CLAUDE_CODE_OAUTH_TOKEN` kullanılmalıdır. Önce Docker içinde doğrudan `claude -p` çalıştığını kanıtlar, ardından Anthropic API anahtarı env değişkenlerini korumadan iki Gateway CLI-backend turu çalıştırır. Bu abonelik hattı, Claude şu anda üçüncü taraf uygulama kullanımını normal abonelik planı limitleri yerine ek kullanım ücretlendirmesi üzerinden yönlendirdiği için varsayılan olarak Claude MCP/araç ve image probe’larını devre dışı bırakır.
- Canlı CLI-backend smoke artık Claude, Codex ve Gemini için aynı uçtan uca akışı uygular: metin turu, image sınıflandırma turu, ardından gateway CLI üzerinden doğrulanan MCP `cron` araç çağrısı.
- Claude’un varsayılan smoke testi ayrıca oturumu Sonnet’ten Opus’a yamalar ve sürdürülen oturumun önceki bir notu hâlâ hatırladığını doğrular.

## Canlı: ACP bind smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Amaç: gerçek ACP konuşma-bind akışını canlı bir ACP agent ile doğrulamak:
  - `/acp spawn <agent> --bind here` göndermek
  - sentetik bir message-channel konuşmasını yerinde bağlamak
  - aynı konuşma üzerinde normal bir follow-up göndermek
  - follow-up’ın bağlı ACP oturum dökümüne düştüğünü doğrulamak
- Etkinleştirme:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Varsayılanlar:
  - Docker içindeki ACP agent’ları: `claude,codex,gemini`
  - Doğrudan `pnpm test:live ...` için ACP agent’ı: `claude`
  - Sentetik kanal: Slack DM tarzı konuşma bağlamı
  - ACP backend: `acpx`
- Geçersiz kılmalar:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Notlar:
  - Bu hat, testlerin harici teslimatı taklit etmeden message-channel bağlamını ekleyebilmesi için yöneticiye özel sentetik originating-route alanlarıyla gateway `chat.send` yüzeyini kullanır.
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` ayarlanmadığında test, seçilen ACP harness agent için gömülü `acpx` eklentisinin yerleşik agent kayıt defterini kullanır.

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

Tek agent’lı Docker tarifleri:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker notları:

- Docker çalıştırıcısı `scripts/test-live-acp-bind-docker.sh` konumundadır.
- Varsayılan olarak ACP bind smoke testini tüm desteklenen canlı CLI agent’larına karşı sırayla çalıştırır: `claude`, `codex`, ardından `gemini`.
- Matrisi daraltmak için `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` veya `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` kullanın.
- `~/.profile` kaynağını alır, eşleşen CLI kimlik doğrulama materyalini container içine hazırlar, `acpx`’i yazılabilir bir npm prefix’ine kurar, ardından eksikse istenen canlı CLI’ı (`@anthropic-ai/claude-code`, `@openai/codex` veya `@google/gemini-cli`) kurar.
- Docker içinde çalıştırıcı, kaynak alınan profilden gelen sağlayıcı env değişkenlerini alt harness CLI için kullanılabilir tutmak amacıyla `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` ayarlar.

## Canlı: Codex app-server harness smoke

- Amaç: eklentiye ait Codex harness’ini normal gateway
  `agent` yöntemi üzerinden doğrulamak:
  - paketlenmiş `codex` eklentisini yüklemek
  - `OPENCLAW_AGENT_RUNTIME=codex` seçmek
  - `codex/gpt-5.4` için ilk gateway agent turunu göndermek
  - aynı OpenClaw oturumuna ikinci bir tur göndermek ve app-server
    thread’inin resume edebildiğini doğrulamak
  - aynı gateway komut
    yolu üzerinden `/codex status` ve `/codex models` çalıştırmak
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Varsayılan model: `codex/gpt-5.4`
- İsteğe bağlı image probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- İsteğe bağlı MCP/araç probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Smoke testi `OPENCLAW_AGENT_HARNESS_FALLBACK=none` ayarlar; böylece bozuk bir Codex
  harness’i sessizce PI’ya geri dönerek başarılı olamaz.
- Kimlik doğrulama: shell/profilden `OPENAI_API_KEY`, ayrıca isteğe bağlı olarak kopyalanmış
  `~/.codex/auth.json` ve `~/.codex/config.toml`

Yerel tarif:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker tarifi:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Docker notları:

- Docker çalıştırıcısı `scripts/test-live-codex-harness-docker.sh` konumundadır.
- Bağlanan `~/.profile` dosyasının kaynağını alır, `OPENAI_API_KEY` geçirir, varsa Codex CLI
  kimlik doğrulama dosyalarını kopyalar, `@openai/codex` paketini yazılabilir ve bağlanmış bir npm
  prefix’ine kurar, kaynak ağacı hazırlar, ardından yalnızca Codex-harness canlı testini çalıştırır.
- Docker varsayılan olarak image ve MCP/araç probe’larını etkinleştirir. Daha dar bir hata ayıklama çalıştırması gerektiğinde
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` veya
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` ayarlayın.
- Docker ayrıca `OPENCLAW_AGENT_HARNESS_FALLBACK=none` dışa aktarır; bu, canlı
  test yapılandırmasıyla eşleşir; böylece `openai-codex/*` veya PI geri dönüşü bir Codex harness
  regresyonunu gizleyemez.

### Önerilen canlı tarifler

Dar, açık allowlist’ler en hızlı ve en az oynak olanlardır:

- Tek model, doğrudan (gateway yok):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Tek model, gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Birkaç sağlayıcıda araç çağırma:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google odağı (Gemini API anahtarı + Antigravity):
  - Gemini (API anahtarı): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Notlar:

- `google/...`, Gemini API’yi kullanır (API anahtarı).
- `google-antigravity/...`, Antigravity OAuth köprüsünü kullanır (Cloud Code Assist tarzı agent uç noktası).
- `google-gemini-cli/...`, makinenizdeki yerel Gemini CLI’ı kullanır (ayrı kimlik doğrulama + araç davranışı farklılıkları).
- Gemini API ile Gemini CLI:
  - API: OpenClaw, Google’ın barındırılan Gemini API’sini HTTP üzerinden çağırır (API anahtarı / profil kimlik doğrulaması); çoğu kullanıcının “Gemini” derken kastettiği budur.
  - CLI: OpenClaw, yerel bir `gemini` ikilisini shell üzerinden çağırır; bunun kendi kimlik doğrulaması vardır ve davranışı farklı olabilir (streaming/araç desteği/sürüm uyumsuzluğu).

## Canlı: model matrisi (neleri kapsıyoruz)

Sabit bir “CI model listesi” yoktur (canlı testler isteğe bağlıdır), ancak bunlar anahtarlara sahip bir geliştirme makinesinde düzenli olarak kapsanması **önerilen** modellerdir.

### Modern smoke kümesi (araç çağırma + image)

Bu, çalışır durumda tutmayı beklediğimiz “yaygın modeller” çalıştırmasıdır:

- OpenAI (Codex dışı): `openai/gpt-5.4` (isteğe bağlı: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (veya `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` ve `google/gemini-3-flash-preview` (eski Gemini 2.x modellerinden kaçının)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` ve `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Araçlar + image ile gateway smoke çalıştırması:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Temel seviye: araç çağırma (Read + isteğe bağlı Exec)

Her sağlayıcı ailesinden en az birini seçin:

- OpenAI: `openai/gpt-5.4` (veya `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (veya `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (veya `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

İsteğe bağlı ek kapsama (olsa iyi olur):

- xAI: `xai/grok-4` (veya mevcut en son sürüm)
- Mistral: `mistral/`… (etkinleştirdiğiniz “tools” yetenekli bir modeli seçin)
- Cerebras: `cerebras/`… (`erişiminiz varsa`)
- LM Studio: `lmstudio/`… (yerel; araç çağırma API moduna bağlıdır)

### Vision: image gönderimi (ek → çok modlu mesaj)

Image probe’unu çalıştırmak için `OPENCLAW_LIVE_GATEWAY_MODELS` içine image yetenekli en az bir model ekleyin (Claude/Gemini/OpenAI’nin vision destekli varyantları vb.).

### Toplayıcılar / alternatif gateway’ler

Anahtarlarınız etkinse, şunlar üzerinden test etmeyi de destekliyoruz:

- OpenRouter: `openrouter/...` (yüzlerce model; tool+image yetenekli adayları bulmak için `openclaw models scan` kullanın)
- OpenCode: Zen için `opencode/...`, Go için `opencode-go/...` (`OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY` ile kimlik doğrulama)

Canlı matrise dahil edebileceğiniz daha fazla sağlayıcı (kimlik bilgileriniz/yapılandırmanız varsa):

- Yerleşik: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers` aracılığıyla (özel uç noktalar): `minimax` (bulut/API), ayrıca OpenAI/Anthropic uyumlu herhangi bir proxy (LM Studio, vLLM, LiteLLM vb.)

İpucu: belgelerde “tüm modelleri” sabit kodlamaya çalışmayın. Yetkili liste, makinenizde `discoverModels(...)` ne döndürüyorsa ve hangi anahtarlar mevcutsa odur.

## Kimlik bilgileri (asla commit etmeyin)

Canlı testler, kimlik bilgilerini CLI ile aynı şekilde keşfeder. Pratik sonuçlar:

- CLI çalışıyorsa, canlı testler de aynı anahtarları bulmalıdır.
- Bir canlı test “kimlik bilgisi yok” diyorsa, bunu `openclaw models list` / model seçimini nasıl hata ayıklıyorsanız aynı şekilde hata ayıklayın.

- Agent başına kimlik doğrulama profilleri: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (canlı testlerde “profil anahtarları” denince kastedilen budur)
- Yapılandırma: `~/.openclaw/openclaw.json` (veya `OPENCLAW_CONFIG_PATH`)
- Eski durum dizini: `~/.openclaw/credentials/` (varsa hazırlanan canlı home içine kopyalanır, ancak ana profil-anahtar deposu değildir)
- Canlı yerel çalıştırmalar, varsayılan olarak etkin yapılandırmayı, agent başına `auth-profiles.json` dosyalarını, eski `credentials/` dizinini ve desteklenen harici CLI kimlik doğrulama dizinlerini geçici bir test home dizinine kopyalar; hazırlanan canlı home’lar `workspace/` ve `sandboxes/` dizinlerini atlar, ayrıca `agents.*.workspace` / `agentDir` yol geçersiz kılmaları çıkarılır; böylece probe’lar gerçek host çalışma alanınızdan uzak kalır.

Env anahtarlarına güvenmek istiyorsanız (`~/.profile` dosyanızda dışa aktarılmış olanlar gibi), yerel testleri `source ~/.profile` sonrasında çalıştırın ya da aşağıdaki Docker çalıştırıcılarını kullanın (`~/.profile` dosyanızı container içine bağlayabilirler).

## Deepgram canlı (ses transkripsiyonu)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Etkinleştirme: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan canlı

- Test: `src/agents/byteplus.live.test.ts`
- Etkinleştirme: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- İsteğe bağlı model geçersiz kılması: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media canlı

- Test: `extensions/comfy/comfy.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Kapsam:
  - Paketlenmiş comfy image, video ve `music_generate` yollarını çalıştırır
  - `models.providers.comfy.<capability>` yapılandırılmadıkça her yeteneği atlar
  - Comfy workflow gönderimi, polling, indirmeler veya eklenti kaydı değiştirildikten sonra yararlıdır

## Image generation canlı

- Test: `src/image-generation/runtime.live.test.ts`
- Komut: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Kapsam:
  - Kaydedilmiş her image-generation sağlayıcı eklentisini numaralandırır
  - Probe yapmadan önce eksik sağlayıcı env değişkenlerini oturum açma shell’inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan kimlik doğrulama profillerinden önce canlı/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulama/profil/modeli olmayan sağlayıcıları atlar
  - Standart image-generation varyantlarını paylaşılan çalışma zamanı yeteneği üzerinden çalıştırır:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Şu anda kapsanan paketlenmiş sağlayıcılar:
  - `openai`
  - `google`
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Yalnızca profil deposu kimlik doğrulamasını zorlamak ve yalnızca env olan geçersiz kılmaları yoksaymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Music generation canlı

- Test: `extensions/music-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Kapsam:
  - Paylaşılan paketlenmiş music-generation sağlayıcı yolunu çalıştırır
  - Şu anda Google ve MiniMax’ı kapsar
  - Probe yapmadan önce sağlayıcı env değişkenlerini oturum açma shell’inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan kimlik doğrulama profillerinden önce canlı/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulama/profil/modeli olmayan sağlayıcıları atlar
  - Mevcut olduğunda bildirilen iki çalışma zamanı modunu da çalıştırır:
    - yalnızca prompt girdisiyle `generate`
    - sağlayıcı `capabilities.edit.enabled` bildirdiğinde `edit`
  - Geçerli paylaşılan hat kapsaması:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: ayrı Comfy canlı dosyası, bu paylaşılan tarama değil
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Yalnızca profil deposu kimlik doğrulamasını zorlamak ve yalnızca env olan geçersiz kılmaları yoksaymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Video generation canlı

- Test: `extensions/video-generation-providers.live.test.ts`
- Etkinleştirme: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Kapsam:
  - Paylaşılan paketlenmiş video-generation sağlayıcı yolunu çalıştırır
  - Varsayılan olarak sürüm için güvenli smoke yolunu kullanır: FAL dışı sağlayıcılar, sağlayıcı başına bir text-to-video isteği, bir saniyelik lobster prompt ve sağlayıcı başına `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (`180000` varsayılan) değerinden gelen işlem üst sınırı
  - Sağlayıcı tarafı kuyruk gecikmesi sürüm süresine baskın gelebileceği için FAL varsayılan olarak atlanır; açıkça çalıştırmak için `--video-providers fal` veya `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` verin
  - Probe yapmadan önce sağlayıcı env değişkenlerini oturum açma shell’inizden (`~/.profile`) yükler
  - Varsayılan olarak saklanan kimlik doğrulama profillerinden önce canlı/env API anahtarlarını kullanır; böylece `auth-profiles.json` içindeki eski test anahtarları gerçek shell kimlik bilgilerini maskelemez
  - Kullanılabilir kimlik doğrulama/profil/modeli olmayan sağlayıcıları atlar
  - Varsayılan olarak yalnızca `generate` çalıştırır
  - Bildirilen dönüşüm modlarını da çalıştırmak için `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` ayarlayın:
    - sağlayıcı `capabilities.imageToVideo.enabled` bildirdiğinde ve seçilen sağlayıcı/model paylaşılan taramada buffer destekli yerel image girdisini kabul ettiğinde `imageToVideo`
    - sağlayıcı `capabilities.videoToVideo.enabled` bildirdiğinde ve seçilen sağlayıcı/model paylaşılan taramada buffer destekli yerel video girdisini kabul ettiğinde `videoToVideo`
  - Paylaşılan taramada şu anda bildirilmiş ama atlanmış `imageToVideo` sağlayıcıları:
    - `vydra`, çünkü paketlenmiş `veo3` yalnızca metindir ve paketlenmiş `kling` uzak bir image URL’si gerektirir
  - Sağlayıcıya özgü Vydra kapsaması:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - bu dosya varsayılan olarak `veo3` text-to-video artı uzak bir image URL fixture’ı kullanan bir `kling` hattı çalıştırır
  - Geçerli `videoToVideo` canlı kapsaması:
    - yalnızca seçilen model `runway/gen4_aleph` olduğunda `runway`
  - Paylaşılan taramada şu anda bildirilmiş ama atlanmış `videoToVideo` sağlayıcıları:
    - `alibaba`, `qwen`, `xai`, çünkü bu yollar şu anda uzak `http(s)` / MP4 referans URL’leri gerektiriyor
    - `google`, çünkü mevcut paylaşılan Gemini/Veo hattı yerel buffer destekli girdi kullanıyor ve bu yol paylaşılan taramada kabul edilmiyor
    - `openai`, çünkü mevcut paylaşılan hat org’a özgü video inpaint/remix erişim garantilerinden yoksun
- İsteğe bağlı daraltma:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - Varsayılan taramaya FAL dahil her sağlayıcıyı eklemek için `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`
  - Daha agresif bir smoke çalıştırması için sağlayıcı başına işlem üst sınırını azaltmak üzere `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`
- İsteğe bağlı kimlik doğrulama davranışı:
  - Yalnızca profil deposu kimlik doğrulamasını zorlamak ve yalnızca env olan geçersiz kılmaları yoksaymak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Media canlı harness

- Komut: `pnpm test:live:media`
- Amaç:
  - Paylaşılan image, music ve video canlı paketlerini depo yerel tek bir giriş noktası üzerinden çalıştırır
  - Eksik sağlayıcı env değişkenlerini `~/.profile` dosyasından otomatik yükler
  - Varsayılan olarak her paketi, şu anda kullanılabilir kimlik doğrulamaya sahip sağlayıcılara otomatik olarak daraltır
  - `scripts/test-live.mjs` dosyasını yeniden kullanır; böylece Heartbeat ve sessiz mod davranışı tutarlı kalır
- Örnekler:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker çalıştırıcıları (isteğe bağlı “Linux’ta çalışıyor” kontrolleri)

Bu Docker çalıştırıcıları iki kümeye ayrılır:

- Canlı model çalıştırıcıları: `test:docker:live-models` ve `test:docker:live-gateway`, yalnızca eşleşen profil anahtarı canlı dosyalarını depo Docker image’ı içinde çalıştırır (`src/agents/models.profiles.live.test.ts` ve `src/gateway/gateway-models.profiles.live.test.ts`); yerel yapılandırma dizininizi ve çalışma alanınızı bağlar (ve bağlanmışsa `~/.profile` kaynağını alır). Eşleşen yerel giriş noktaları `test:live:models-profiles` ve `test:live:gateway-profiles`’tır.
- Docker canlı çalıştırıcıları, tam bir Docker taramasının pratik kalması için varsayılan olarak daha küçük bir smoke üst sınırı kullanır:
  `test:docker:live-models`, varsayılan olarak `OPENCLAW_LIVE_MAX_MODELS=12` kullanır ve
  `test:docker:live-gateway`, varsayılan olarak `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` ve
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` kullanır. Daha büyük kapsamlı taramayı
  özellikle istediğinizde bu env değişkenlerini geçersiz kılın.
- `test:docker:all`, önce `test:docker:live-build` aracılığıyla canlı Docker image’ını bir kez oluşturur, ardından bunu iki canlı Docker hattı için yeniden kullanır.
- Container smoke çalıştırıcıları: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` ve `test:docker:plugins`, bir veya daha fazla gerçek container başlatır ve daha üst düzey entegrasyon yollarını doğrular.

Canlı model Docker çalıştırıcıları ayrıca yalnızca gereken CLI kimlik doğrulama home dizinlerini (veya çalıştırma daraltılmamışsa desteklenenlerin tümünü) bind-mount eder, ardından çalıştırma öncesinde bunları container home dizinine kopyalar; böylece harici CLI OAuth, host kimlik doğrulama deposunu değiştirmeden token’ları yenileyebilir:

- Doğrudan modeller: `pnpm test:docker:live-models` (betik: `scripts/test-live-models-docker.sh`)
- ACP bind smoke: `pnpm test:docker:live-acp-bind` (betik: `scripts/test-live-acp-bind-docker.sh`)
- CLI backend smoke: `pnpm test:docker:live-cli-backend` (betik: `scripts/test-live-cli-backend-docker.sh`)
- Codex app-server harness smoke: `pnpm test:docker:live-codex-harness` (betik: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + geliştirme agent’ı: `pnpm test:docker:live-gateway` (betik: `scripts/test-live-gateway-models-docker.sh`)
- Open WebUI canlı smoke: `pnpm test:docker:openwebui` (betik: `scripts/e2e/openwebui-docker.sh`)
- Onboarding sihirbazı (TTY, tam iskele): `pnpm test:docker:onboard` (betik: `scripts/e2e/onboard-docker.sh`)
- Gateway ağ iletişimi (iki container, WS kimlik doğrulama + sağlık): `pnpm test:docker:gateway-network` (betik: `scripts/e2e/gateway-network-docker.sh`)
- MCP kanal köprüsü (seed edilmiş Gateway + stdio köprüsü + ham Claude bildirim-frame smoke): `pnpm test:docker:mcp-channels` (betik: `scripts/e2e/mcp-channels-docker.sh`)
- Eklentiler (kurulum smoke + `/plugin` takma adı + Claude paketi yeniden başlatma semantiği): `pnpm test:docker:plugins` (betik: `scripts/e2e/plugins-docker.sh`)

Canlı model Docker çalıştırıcıları ayrıca mevcut checkout’u salt okunur olarak bind-mount eder ve
bunu container içinde geçici bir workdir’e hazırlar. Bu, çalışma zamanı
image’ını ince tutarken yine de Vitest’i tam olarak yerel kaynak/yapılandırmanızla çalıştırır.
Hazırlama adımı, büyük yerel önbellekleri ve uygulama derleme çıktıları gibi
yalnızca makineye özgü artifact’ları kopyalayarak Docker canlı çalıştırmalarının dakikalar
harcamasını önlemek için `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` ve uygulamaya özgü `.build` veya
Gradle çıktı dizinleri gibi büyük yerel-only önbellekleri ve uygulama derleme çıktıları atlar.
Ayrıca `OPENCLAW_SKIP_CHANNELS=1` ayarlar; böylece gateway canlı probe’ları
container içinde gerçek Telegram/Discord/vb. kanal worker’larını başlatmaz.
`test:docker:live-models` yine de `pnpm test:live` çalıştırır; bu nedenle
bu Docker hattından gateway canlı kapsamını daraltmanız veya hariç tutmanız gerektiğinde
`OPENCLAW_LIVE_GATEWAY_*` değişkenlerini de iletin.
`test:docker:openwebui`, daha üst düzey bir uyumluluk smoke testidir: OpenAI uyumlu HTTP uç noktaları etkin bir
OpenClaw gateway container’ı başlatır, bu gateway’e karşı sabitlenmiş bir Open WebUI container’ı başlatır, Open WebUI üzerinden
oturum açar, `/api/models` uç noktasının `openclaw/default` öğesini sunduğunu doğrular, ardından
Open WebUI’nin `/api/chat/completions` proxy’si üzerinden gerçek bir sohbet isteği gönderir.
İlk çalıştırma belirgin biçimde daha yavaş olabilir; çünkü Docker’ın
Open WebUI image’ını çekmesi gerekebilir ve Open WebUI’nin kendi cold-start kurulumunu tamamlaması gerekebilir.
Bu hat kullanılabilir bir canlı model anahtarı bekler ve `OPENCLAW_PROFILE_FILE`
(Docker içinde çalıştırmalarda varsayılan olarak `~/.profile`) bunu sağlamak için birincil yoldur.
Başarılı çalıştırmalar `{ "ok": true, "model":
"openclaw/default", ... }` gibi küçük bir JSON yükü yazdırır.
`test:docker:mcp-channels` kasıtlı olarak deterministiktir ve gerçek bir
Telegram, Discord veya iMessage hesabı gerektirmez. Seed edilmiş bir Gateway
container’ı başlatır, `openclaw mcp serve` başlatan ikinci bir container çalıştırır, ardından
gerçek stdio MCP köprüsü üzerinden yönlendirilmiş konuşma keşfini, döküm okumalarını, ek meta verilerini,
canlı olay kuyruğu davranışını, giden gönderim yönlendirmesini ve Claude tarzı kanal +
izin bildirimlerini doğrular. Bildirim denetimi ham stdio MCP frame’lerini doğrudan
inceler; böylece smoke testi, belirli bir istemci SDK’sının tesadüfen yüzeye çıkardığını değil,
köprünün gerçekten ne yaydığını doğrular.

Elle ACP düz dil thread smoke testi (CI değil):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Bu betiği regresyon/hata ayıklama iş akışları için saklayın. ACP thread yönlendirme doğrulaması için yeniden gerekebilir, bu yüzden silmeyin.

Yararlı env değişkenleri:

- `OPENCLAW_CONFIG_DIR=...` (varsayılan: `~/.openclaw`), `/home/node/.openclaw` konumuna bağlanır
- `OPENCLAW_WORKSPACE_DIR=...` (varsayılan: `~/.openclaw/workspace`), `/home/node/.openclaw/workspace` konumuna bağlanır
- `OPENCLAW_PROFILE_FILE=...` (varsayılan: `~/.profile`), `/home/node/.profile` konumuna bağlanır ve testler çalıştırılmadan önce kaynağı alınır
- Yalnızca `OPENCLAW_PROFILE_FILE` içinden kaynak alınan env değişkenlerini doğrulamak için `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1`; geçici yapılandırma/çalışma alanı dizinleri ve harici CLI kimlik doğrulama bağlamaları kullanılmaz
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (varsayılan: `~/.cache/openclaw/docker-cli-tools`), Docker içindeki önbelleklenmiş CLI kurulumları için `/home/node/.npm-global` konumuna bağlanır
- `$HOME` altındaki harici CLI kimlik doğrulama dizinleri/dosyaları, `/host-auth...` altında salt okunur bağlanır, sonra testler başlamadan önce `/home/node/...` içine kopyalanır
  - Varsayılan dizinler: `.minimax`
  - Varsayılan dosyalar: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Daraltılmış sağlayıcı çalıştırmaları yalnızca `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` değerlerinden çıkarılan gerekli dizinleri/dosyaları bağlar
  - Elle geçersiz kılmak için `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` veya `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` gibi virgüllü bir liste kullanın
- Çalıştırmayı daraltmak için `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- Container içinde sağlayıcıları filtrelemek için `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- Yeniden derleme gerektirmeyen tekrar çalıştırmalarda mevcut `openclaw:local-live` image’ını yeniden kullanmak için `OPENCLAW_SKIP_DOCKER_BUILD=1`
- Kimlik bilgilerinin env’den değil profil deposundan geldiğinden emin olmak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI smoke için gateway tarafından sunulan modeli seçmek üzere `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI smoke tarafından kullanılan nonce denetim prompt’unu geçersiz kılmak için `OPENCLAW_OPENWEBUI_PROMPT=...`
- Sabitlenmiş Open WebUI image etiketini geçersiz kılmak için `OPENWEBUI_IMAGE=...`

## Belgeler için sağlık kontrolü

Belge düzenlemelerinden sonra belge denetimlerini çalıştırın: `pnpm check:docs`.
Sayfa içi başlık denetimlerine de ihtiyaç duyduğunuzda tam Mintlify anchor doğrulamasını çalıştırın: `pnpm docs:check-links:anchors`.

## Çevrimdışı regresyon (CI güvenli)

Bunlar gerçek sağlayıcılar olmadan “gerçek işlem hattı” regresyonlarıdır:

- Gateway araç çağırma (sahte OpenAI, gerçek gateway + agent döngüsü): `src/gateway/gateway.test.ts` (durum: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway sihirbazı (WS `wizard.start`/`wizard.next`, yapılandırma + kimlik doğrulama zorunlu olarak yazar): `src/gateway/gateway.test.ts` (durum: "runs wizard over ws and writes auth token config")

## Agent güvenilirlik değerlendirmeleri (Skills)

CI açısından güvenli ve “agent güvenilirlik değerlendirmeleri” gibi davranan birkaç testimiz zaten var:

- Gerçek gateway + agent döngüsü üzerinden sahte araç çağırma (`src/gateway/gateway.test.ts`).
- Oturum kablolamasını ve yapılandırma etkilerini doğrulayan uçtan uca sihirbaz akışları (`src/gateway/gateway.test.ts`).

Skills için hâlâ eksik olanlar (bkz. [Skills](/tr/tools/skills)):

- **Karar verme:** prompt içinde Skills listelendiğinde agent doğru Skill’i seçiyor mu (veya alakasız olanlardan kaçınıyor mu)?
- **Uyumluluk:** agent kullanımdan önce `SKILL.md` dosyasını okuyor ve gerekli adımları/argümanları takip ediyor mu?
- **İş akışı sözleşmeleri:** araç sırasını, oturum geçmişi devrini ve sandbox sınırlarını doğrulayan çok turlu senaryolar.

Gelecekteki değerlendirmeler önce deterministik kalmalıdır:

- Araç çağrılarını + sırasını, Skill dosyası okumalarını ve oturum kablolamasını doğrulamak için sahte sağlayıcılar kullanan bir senaryo çalıştırıcısı.
- Skill odaklı küçük bir senaryo paketi (kullan vs kaçın, geçitleme, prompt injection).
- Yalnızca CI güvenli paket yerleştiğinde isteğe bağlı canlı değerlendirmeler (opt-in, env ile geçitlenen).

## Sözleşme testleri (eklenti ve kanal şekli)

Sözleşme testleri, kaydedilmiş her eklenti ve kanalın kendi
arayüz sözleşmesine uyduğunu doğrular. Keşfedilen tüm eklentiler üzerinde yineleme yapar
ve bir dizi şekil ile davranış doğrulaması çalıştırır. Varsayılan `pnpm test` birim hattı
bu paylaşılan dikiş ve smoke dosyalarını bilerek atlar; paylaşılan kanal veya sağlayıcı yüzeylerine
dokunduğunuzda sözleşme komutlarını açıkça çalıştırın.

### Komutlar

- Tüm sözleşmeler: `pnpm test:contracts`
- Yalnızca kanal sözleşmeleri: `pnpm test:contracts:channels`
- Yalnızca sağlayıcı sözleşmeleri: `pnpm test:contracts:plugins`

### Kanal sözleşmeleri

`src/channels/plugins/contracts/*.contract.test.ts` konumundadır:

- **plugin** - Temel eklenti şekli (id, name, capabilities)
- **setup** - Kurulum sihirbazı sözleşmesi
- **session-binding** - Oturum bağlama davranışı
- **outbound-payload** - Mesaj yükü yapısı
- **inbound** - Gelen mesaj işleme
- **actions** - Kanal eylem işleyicileri
- **threading** - Thread kimliği işleme
- **directory** - Dizin/roster API
- **group-policy** - Grup ilkesi uygulaması

### Sağlayıcı durum sözleşmeleri

`src/plugins/contracts/*.contract.test.ts` konumundadır.

- **status** - Kanal durum yoklamaları
- **registry** - Eklenti kayıt defteri şekli

### Sağlayıcı sözleşmeleri

`src/plugins/contracts/*.contract.test.ts` konumundadır:

- **auth** - Kimlik doğrulama akışı sözleşmesi
- **auth-choice** - Kimlik doğrulama seçimi/seçim mantığı
- **catalog** - Model katalog API’si
- **discovery** - Eklenti keşfi
- **loader** - Eklenti yükleme
- **runtime** - Sağlayıcı çalışma zamanı
- **shape** - Eklenti şekli/arayüzü
- **wizard** - Kurulum sihirbazı

### Ne zaman çalıştırılmalı

- plugin-sdk dışa aktarımlarını veya alt yollarını değiştirdikten sonra
- Bir kanal veya sağlayıcı eklentisi ekledikten ya da değiştirdikten sonra
- Eklenti kaydı veya keşfini yeniden düzenledikten sonra

Sözleşme testleri CI’da çalışır ve gerçek API anahtarları gerektirmez.

## Regresyon ekleme (rehber)

Canlıda keşfedilen bir sağlayıcı/model sorununu düzelttiğinizde:

- Mümkünse CI güvenli bir regresyon ekleyin (sahte/stub sağlayıcı veya tam istek-şekli dönüşümünü yakalayın)
- Doğası gereği yalnızca canlıda yeniden üretilebiliyorsa (rate limit’ler, kimlik doğrulama ilkeleri), canlı testi dar ve env değişkenleriyle opt-in tutun
- Hatayı yakalayan en küçük katmanı hedeflemeyi tercih edin:
  - sağlayıcı istek dönüştürme/replay hatası → doğrudan modeller testi
  - gateway oturum/geçmiş/araç işlem hattı hatası → gateway canlı smoke veya CI güvenli gateway sahte testi
- SecretRef gezinme koruma kuralı:
  - `src/secrets/exec-secret-ref-id-parity.test.ts`, kayıt defteri meta verilerinden (`listSecretTargetRegistryEntries()`) SecretRef sınıfı başına örneklenmiş bir hedef türetir, ardından gezinme bölümü exec kimliklerinin reddedildiğini doğrular.
  - `src/secrets/target-registry-data.ts` içine yeni bir `includeInPlan` SecretRef hedef ailesi eklerseniz, bu testteki `classifyTargetClass` öğesini güncelleyin. Test, yeni sınıfların sessizce atlanamaması için sınıflandırılmamış hedef kimliklerinde kasıtlı olarak başarısız olur.
