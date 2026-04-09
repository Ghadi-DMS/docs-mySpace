---
read_when:
    - doctor geçişleri eklenirken veya değiştirilirken
    - geriye dönük uyumluluğu bozan yapılandırma değişiklikleri sunulurken
summary: 'Doctor komutu: sağlık kontrolleri, yapılandırma geçişleri ve onarım adımları'
title: Doctor
x-i18n:
    generated_at: "2026-04-09T01:29:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 75d321bd1ad0e16c29f2382e249c51edfc3a8d33b55bdceea39e7dbcd4901fce
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor`, OpenClaw için onarım + geçiş aracıdır. Eski
yapılandırma/durumu düzeltir, sağlığı kontrol eder ve uygulanabilir onarım adımları sağlar.

## Hızlı başlangıç

```bash
openclaw doctor
```

### Başsız / otomasyon

```bash
openclaw doctor --yes
```

İstem olmadan varsayılanları kabul eder (uygulanabildiğinde yeniden başlatma/hizmet/sandbox onarım adımları dahil).

```bash
openclaw doctor --repair
```

Önerilen onarımları istem olmadan uygular (güvenli olduğunda onarımlar + yeniden başlatmalar).

```bash
openclaw doctor --repair --force
```

Agresif onarımları da uygular (özel supervisor yapılandırmalarının üzerine yazar).

```bash
openclaw doctor --non-interactive
```

İstemler olmadan çalışır ve yalnızca güvenli geçişleri uygular (yapılandırma normalleştirme + disk üzerindeki durum taşımaları). İnsan onayı gerektiren yeniden başlatma/hizmet/sandbox eylemlerini atlar.
Eski durum geçişleri algılandığında otomatik olarak çalışır.

```bash
openclaw doctor --deep
```

Ek gateway kurulumları için sistem hizmetlerini tarar (launchd/systemd/schtasks).

Yazmadan önce değişiklikleri gözden geçirmek istiyorsanız önce yapılandırma dosyasını açın:

```bash
cat ~/.openclaw/openclaw.json
```

## Ne yapar (özet)

- Git kurulumları için isteğe bağlı ön kontrol güncellemesi (yalnızca etkileşimli).
- UI protokol güncelliği kontrolü (protokol şeması daha yeniyse Control UI’ı yeniden derler).
- Sağlık kontrolü + yeniden başlatma istemi.
- Skills durum özeti (uygun/eksik/engelli) ve plugin durumu.
- Eski değerler için yapılandırma normalleştirme.
- Eski düz `talk.*` alanlarından `talk.provider` + `talk.providers.<provider>` yapısına Talk yapılandırma geçişi.
- Eski Chrome uzantısı yapılandırmaları ve Chrome MCP hazır olma durumu için tarayıcı geçiş kontrolleri.
- OpenCode sağlayıcı geçersiz kılma uyarıları (`models.providers.opencode` / `models.providers.opencode-go`).
- Codex OAuth gölgeleme uyarıları (`models.providers.openai-codex`).
- OpenAI Codex OAuth profilleri için OAuth TLS önkoşul kontrolü.
- Disk üzerindeki eski durum geçişi (sessions/agent dizini/WhatsApp kimlik doğrulaması).
- Eski plugin manifest sözleşme anahtarı geçişi (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Eski cron deposu geçişi (`jobId`, `schedule.cron`, üst düzey delivery/payload alanları, payload `provider`, basit `notify: true` webhook fallback işleri).
- Oturum kilit dosyası incelemesi ve eski kilit temizliği.
- Durum bütünlüğü ve izin kontrolleri (oturumlar, dökümler, durum dizini).
- Yerelde çalışırken yapılandırma dosyası izin kontrolleri (`chmod 600`).
- Model kimlik doğrulama sağlığı: OAuth sona ermesini kontrol eder, süresi dolmak üzere olan belirteçleri yenileyebilir ve auth-profile bekleme süresi/devre dışı bırakılmış durumlarını raporlar.
- Ek çalışma alanı dizini algılama (`~/openclaw`).
- Sandbox etkin olduğunda sandbox imajı onarımı.
- Eski hizmet geçişi ve ek gateway algılama.
- Matrix kanal eski durum geçişi (`--fix` / `--repair` kipinde).
- Gateway çalışma zamanı kontrolleri (hizmet kurulu ama çalışmıyor; önbelleğe alınmış launchd etiketi).
- Kanal durum uyarıları (çalışan gateway üzerinden yoklanır).
- İsteğe bağlı onarımla birlikte supervisor yapılandırma denetimi (launchd/systemd/schtasks).
- Gateway çalışma zamanı en iyi uygulama kontrolleri (Node ve Bun, sürüm yöneticisi yolları).
- Gateway bağlantı noktası çakışma tanılaması (varsayılan `18789`).
- Açık DM ilkeleri için güvenlik uyarıları.
- Yerel belirteç kipi için gateway kimlik doğrulama kontrolleri (belirteç kaynağı yoksa belirteç üretmeyi önerir; belirteç SecretRef yapılandırmalarının üzerine yazmaz).
- Linux üzerinde systemd linger kontrolü.
- Çalışma alanı bootstrap dosya boyutu kontrolü (bağlam dosyaları için kesme/sınıra yakın uyarıları).
- Kabuk tamamlama durumu kontrolü ve otomatik kurulum/yükseltme.
- Bellek araması gömme sağlayıcısı hazır olma durumu kontrolü (yerel model, uzak API anahtarı veya QMD ikilisi).
- Kaynak kurulum kontrolleri (pnpm çalışma alanı uyumsuzluğu, eksik UI varlıkları, eksik tsx ikilisi).
- Güncellenmiş yapılandırmayı + sihirbaz metadatasını yazar.

## Dreams UI geri doldurma ve sıfırlama

Control UI Dreams sahnesi, grounded dreaming iş akışı için **Backfill**, **Reset** ve **Clear Grounded**
eylemlerini içerir. Bu eylemler gateway
doctor tarzı RPC yöntemlerini kullanır, ancak `openclaw doctor` CLI
onarım/geçişinin bir parçası **değildir**.

Yaptıkları:

- **Backfill**, etkin
  çalışma alanındaki geçmiş `memory/YYYY-MM-DD.md` dosyalarını tarar, grounded REM diary geçişini çalıştırır ve geri alınabilir geri doldurma
  girdilerini `DREAMS.md` içine yazar.
- **Reset**, yalnızca işaretlenmiş geri doldurma günlük girdilerini `DREAMS.md` dosyasından kaldırır.
- **Clear Grounded**, yalnızca geçmiş yeniden oynatmadan gelen ve henüz canlı geri çağırma veya günlük
  destek biriktirmemiş, aşamalanmış yalnızca grounded kısa süreli girdileri kaldırır.

Kendi başlarına **yapmadıkları**:

- `MEMORY.md` dosyasını düzenlemezler
- tam doctor geçişlerini çalıştırmazlar
- önce aşamalanmış CLI yolunu açıkça çalıştırmadığınız sürece grounded adayları otomatik olarak canlı kısa süreli
  yükseltme deposuna aşamalamazlar

Grounded geçmiş yeniden oynatımının normal derin yükseltme
hattını etkilemesini istiyorsanız, bunun yerine CLI akışını kullanın:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

Bu, grounded dayanıklı adayları kısa süreli dreaming deposuna aşamalar;
`DREAMS.md` dosyasını gözden geçirme yüzeyi olarak korur.

## Ayrıntılı davranış ve gerekçe

### 0) İsteğe bağlı güncelleme (git kurulumları)

Bu bir git checkout ise ve doctor etkileşimli olarak çalışıyorsa,
doctor çalıştırılmadan önce güncelleme (fetch/rebase/build) sunar.

### 1) Yapılandırma normalleştirme

Yapılandırma eski değer biçimlerini içeriyorsa (örneğin kanala özgü bir geçersiz kılma olmadan `messages.ackReaction`),
doctor bunları geçerli
şemaya normalleştirir.

Buna eski Talk düz alanları da dahildir. Geçerli herkese açık Talk yapılandırması
`talk.provider` + `talk.providers.<provider>` şeklindedir. Doctor, eski
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` biçimlerini sağlayıcı haritasına yeniden yazar.

### 2) Eski yapılandırma anahtarı geçişleri

Yapılandırma kullanımdan kaldırılmış anahtarlar içerdiğinde, diğer komutlar çalışmayı reddeder ve
sizden `openclaw doctor` çalıştırmanızı ister.

Doctor şunları yapar:

- Hangi eski anahtarların bulunduğunu açıklar.
- Uyguladığı geçişi gösterir.
- `~/.openclaw/openclaw.json` dosyasını güncellenmiş şemayla yeniden yazar.

Gateway ayrıca
eski bir yapılandırma biçimi algıladığında başlangıçta doctor geçişlerini otomatik çalıştırır; böylece eski yapılandırmalar elle müdahale olmadan onarılır.
Cron iş deposu geçişleri `openclaw doctor --fix` tarafından ele alınır.

Geçerli geçişler:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → üst düzey `bindings`
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- eski `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Adlandırılmış `accounts` içeren ancak tek hesaplı üst düzey kanal değerleri kalmış kanallarda, o hesap kapsamındaki değerleri ilgili kanal için seçilen yükseltilmiş hesaba taşı (çoğu kanal için `accounts.default`; Matrix mevcut eşleşen bir adlandırılmış/varsayılan hedefi koruyabilir)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- `browser.relayBindHost` değerini kaldır (eski uzantı relay ayarı)

Doctor uyarıları ayrıca çok hesaplı kanallar için hesap varsayılanı yönlendirmesi de içerir:

- `channels.<channel>.defaultAccount` veya `accounts.default` olmadan iki veya daha fazla `channels.<channel>.accounts` girdisi yapılandırılmışsa, doctor geri dönüş yönlendirmesinin beklenmeyen bir hesap seçebileceği konusunda uyarır.
- `channels.<channel>.defaultAccount` bilinmeyen bir hesap kimliğine ayarlanmışsa, doctor uyarır ve yapılandırılmış hesap kimliklerini listeler.

### 2b) OpenCode sağlayıcı geçersiz kılmaları

`models.providers.opencode`, `opencode-zen` veya `opencode-go` değerlerini
elle eklediyseniz, bu
`@mariozechner/pi-ai` içindeki yerleşik OpenCode kataloğunu geçersiz kılar.
Bu, modelleri yanlış API’ye zorlayabilir veya maliyetleri sıfırlayabilir. Doctor, geçersiz kılmayı kaldırıp model başına API yönlendirmesini + maliyetleri geri getirebilmeniz için uyarır.

### 2c) Tarayıcı geçişi ve Chrome MCP hazır olma durumu

Tarayıcı yapılandırmanız hâlâ kaldırılmış Chrome uzantısı yoluna işaret ediyorsa, doctor
bunu geçerli host-local Chrome MCP bağlanma modeline normalleştirir:

- `browser.profiles.*.driver: "extension"` değeri `"existing-session"` olur
- `browser.relayBindHost` kaldırılır

Doctor ayrıca `defaultProfile:
"user"` veya yapılandırılmış bir `existing-session` profili kullandığınızda host-local Chrome MCP yolunu da denetler:

- varsayılan
  otomatik bağlanma profilleri için Google Chrome’un aynı host üzerinde kurulu olup olmadığını kontrol eder
- algılanan Chrome sürümünü kontrol eder ve Chrome 144’ün altında olduğunda uyarır
- tarayıcı inspect sayfasında uzak hata ayıklamayı etkinleştirmenizi hatırlatır (örneğin
  `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  veya `edge://inspect/#remote-debugging`)

Doctor Chrome tarafındaki ayarı sizin için etkinleştiremez. Host-local Chrome MCP
için hâlâ şunlar gerekir:

- gateway/node host üzerinde 144+ Chromium tabanlı bir tarayıcı
- tarayıcının yerelde çalışıyor olması
- o tarayıcıda uzak hata ayıklamanın etkin olması
- tarayıcıdaki ilk bağlanma izin isteminin onaylanması

Buradaki hazır olma durumu yalnızca yerel bağlanma önkoşullarıyla ilgilidir. Existing-session,
geçerli Chrome MCP rota sınırlarını korur; `responsebody`, PDF
dışa aktarma, indirme durdurma ve toplu eylemler gibi gelişmiş rotalar için hâlâ yönetilen
bir tarayıcı veya ham CDP profili gerekir.

Bu kontrol Docker, sandbox, remote-browser veya diğer
başsız akışlar için **geçerli değildir**. Bunlar ham CDP kullanmaya devam eder.

### 2d) OAuth TLS önkoşulları

Bir OpenAI Codex OAuth profili yapılandırıldığında, doctor yerel Node/OpenSSL TLS yığınının
sertifika zincirini doğrulayabildiğini doğrulamak için OpenAI
yetkilendirme uç noktasını yoklar. Yoklama bir sertifika hatasıyla başarısız olursa (örneğin
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, süresi dolmuş sertifika veya self-signed sertifika),
doctor platforma özgü düzeltme yönlendirmesi yazdırır. Homebrew Node kullanan macOS’te
düzeltme genellikle `brew postinstall ca-certificates` olur.
`--deep` ile, gateway sağlıklı olsa bile yoklama çalıştırılır.

### 2c) Codex OAuth sağlayıcı geçersiz kılmaları

Daha önce
`models.providers.openai-codex` altında eski OpenAI taşıma ayarları eklediyseniz, bunlar yeni sürümlerin otomatik kullandığı yerleşik Codex OAuth
sağlayıcı yolunu gölgeleyebilir. Doctor, bu eski taşıma ayarlarını Codex OAuth ile birlikte gördüğünde
uyarır; böylece eski taşıma geçersiz kılmasını kaldırabilir veya yeniden yazabilir ve yerleşik yönlendirme/geri dönüş davranışını
geri alabilirsiniz. Özel proxy’ler ve yalnızca başlık geçersiz kılmaları hâlâ desteklenir ve bu uyarıyı
tetiklemez.

### 3) Eski durum geçişleri (disk düzeni)

Doctor, eski disk üstü düzenleri geçerli yapıya taşıyabilir:

- Oturum deposu + dökümler:
  - `~/.openclaw/sessions/` konumundan `~/.openclaw/agents/<agentId>/sessions/` konumuna
- Agent dizini:
  - `~/.openclaw/agent/` konumundan `~/.openclaw/agents/<agentId>/agent/` konumuna
- WhatsApp kimlik doğrulama durumu (Baileys):
  - eski `~/.openclaw/credentials/*.json` konumundan (`oauth.json` hariç)
  - `~/.openclaw/credentials/whatsapp/<accountId>/...` konumuna (varsayılan hesap kimliği: `default`)

Bu geçişler en iyi çabayla ve idempotent olarak yapılır; doctor
yedek olarak bıraktığı eski klasörler varsa uyarılar verir. Gateway/CLI ayrıca
başlangıçta eski oturumlar + agent dizinini otomatik taşır; böylece geçmiş/kimlik doğrulama/modeller elle doctor çalıştırmadan agent başına yol altında konumlanır. WhatsApp kimlik doğrulaması özellikle yalnızca
`openclaw doctor` üzerinden taşınır. Talk sağlayıcı/sağlayıcı-haritası normalleştirmesi artık
yapısal eşitliğe göre karşılaştırılır; böylece yalnızca anahtar sırası farkı olan değişiklikler
tekrarlayan, etkisiz `doctor --fix` değişikliklerini artık tetiklemez.

### 3a) Eski plugin manifest geçişleri

Doctor tüm kurulu plugin manifest dosyalarını, kullanımdan kaldırılmış üst düzey yetenek
anahtarları (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`) için tarar. Bulunduğunda, bunları `contracts`
nesnesine taşımayı ve manifest dosyasını yerinde yeniden yazmayı önerir. Bu geçiş idempotenttir;
`contracts` anahtarı zaten aynı değerlere sahipse, eski anahtar
veri çoğaltılmadan kaldırılır.

### 3b) Eski cron deposu geçişleri

Doctor ayrıca cron iş deposunu (`varsayılan olarak ~/.openclaw/cron/jobs.json`,
veya geçersiz kılındığında `cron.store`) zamanlayıcının uyumluluk için hâlâ kabul ettiği
eski iş biçimleri açısından kontrol eder.

Geçerli cron temizlemeleri şunları içerir:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- üst düzey payload alanları (`message`, `model`, `thinking`, ...) → `payload`
- üst düzey delivery alanları (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- payload `provider` delivery takma adları → açık `delivery.channel`
- basit eski `notify: true` webhook fallback işleri → açık `delivery.mode="webhook"` ve `delivery.to=cron.webhook`

Doctor, `notify: true` işlerini yalnızca
davranışı değiştirmeden yapabildiğinde otomatik taşır. Bir iş eski notify fallback’i var olan
webhook olmayan bir delivery kipiyle birleştiriyorsa, doctor uyarır ve
o işi elle inceleme için bırakır.

### 3c) Oturum kilidi temizliği

Doctor her agent oturum dizinini eski yazma kilidi dosyaları için tarar — oturum anormal şekilde sonlandığında
geride kalan dosyalar. Bulunan her kilit dosyası için şunları raporlar:
yol, PID, PID’nin hâlâ canlı olup olmadığı, kilit yaşı ve dosyanın
eski kabul edilip edilmediği (ölü PID veya 30 dakikadan eski). `--fix` / `--repair`
kipinde eski kilit dosyalarını otomatik kaldırır; aksi takdirde bir not yazdırır ve
sizi `--fix` ile yeniden çalıştırmanız için yönlendirir.

### 4) Durum bütünlüğü kontrolleri (oturum kalıcılığı, yönlendirme ve güvenlik)

Durum dizini işlemsel beyin sapıdır. Ortadan kaybolursa,
oturumları, kimlik bilgilerini, günlükleri ve yapılandırmayı kaybedersiniz (başka yerde yedekleriniz yoksa).

Doctor şunları kontrol eder:

- **Durum dizini eksik**: yıkıcı durum kaybı konusunda uyarır, dizini yeniden oluşturmayı önerir
  ve eksik verileri kurtaramayacağını hatırlatır.
- **Durum dizini izinleri**: yazılabilirliği doğrular; izinleri onarmayı önerir
  (sahip/grup uyumsuzluğu saptanırsa `chown` ipucu da verir).
- **macOS bulut eşzamanlı durum dizini**: durum iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) veya
  `~/Library/CloudStorage/...` altında çözülüyorsa uyarır; çünkü eşzamanlama destekli yollar daha yavaş G/Ç
  ve kilit/eşzamanlama yarışlarına yol açabilir.
- **Linux SD veya eMMC durum dizini**: durum bir `mmcblk*`
  bağlama kaynağına çözülüyorsa uyarır; çünkü SD veya eMMC tabanlı rastgele G/Ç,
  oturum ve kimlik bilgisi yazımları altında daha yavaş olabilir ve daha hızlı aşınabilir.
- **Oturum dizinleri eksik**: `sessions/` ve oturum deposu dizini,
  geçmişi kalıcı tutmak ve `ENOENT` çökmelerini önlemek için gereklidir.
- **Döküm uyumsuzluğu**: son oturum girdilerinde eksik
  döküm dosyaları olduğunda uyarır.
- **Ana oturum “1 satırlık JSONL”**: ana döküm yalnızca bir
  satıra sahipse işaretler (geçmiş birikmiyor demektir).
- **Birden çok durum dizini**: birden fazla `~/.openclaw` klasörü
  farklı ana dizinlerde varsa veya `OPENCLAW_STATE_DIR` başka bir yeri gösteriyorsa uyarır
  (geçmiş kurulumlar arasında bölünebilir).
- **Uzak kip hatırlatması**: `gateway.mode=remote` ise, doctor bunu
  uzak host üzerinde çalıştırmanızı hatırlatır (durum orada bulunur).
- **Yapılandırma dosyası izinleri**: `~/.openclaw/openclaw.json`
  grup/herkes tarafından okunabiliyorsa uyarır ve bunu `600` olacak şekilde sıkılaştırmayı önerir.

### 5) Model kimlik doğrulama sağlığı (OAuth sona ermesi)

Doctor auth deposundaki OAuth profillerini inceler, belirteçlerin
süresi dolmak üzereyse/dolmuşsa uyarır ve güvenli olduğunda bunları yenileyebilir. Anthropic
OAuth/token profili eskiyse, Anthropic API anahtarı veya
Anthropic setup-token yolunu önerir.
Yenileme istemleri yalnızca etkileşimli (TTY) çalıştırıldığında görünür; `--non-interactive`
yenileme denemelerini atlar.

Bir OAuth yenilemesi kalıcı olarak başarısız olduğunda (örneğin `refresh_token_reused`,
`invalid_grant` veya sağlayıcının yeniden oturum açmanızı söylemesi gibi), doctor
yeniden kimlik doğrulamanın gerekli olduğunu bildirir ve çalıştırmanız gereken tam `openclaw models auth login --provider ...`
komutunu yazdırır.

Doctor ayrıca aşağıdaki nedenlerle geçici olarak kullanılamayan auth profillerini de raporlar:

- kısa bekleme süreleri (oran sınırları/zaman aşımları/kimlik doğrulama hataları)
- daha uzun devre dışı bırakmalar (faturalama/kredi hataları)

### 6) Hooks model doğrulaması

`hooks.gmail.model` ayarlanmışsa, doctor model referansını
katalog ve allowlist’e karşı doğrular ve çözümlenmeyeceği veya izin verilmediği durumlarda uyarır.

### 7) Sandbox imajı onarımı

Sandbox etkin olduğunda, doctor Docker imajlarını kontrol eder ve geçerli imaj eksikse
oluşturmayı veya eski adlara geçmeyi önerir.

### 7b) Paketlenmiş plugin çalışma zamanı bağımlılıkları

Doctor, paketlenmiş plugin çalışma zamanı bağımlılıklarının (örneğin
Discord plugin çalışma zamanı paketleri) OpenClaw kurulum kökünde
bulunduğunu doğrular.
Eksik olanlar varsa, doctor paketleri raporlar ve
`openclaw doctor --fix` / `openclaw doctor --repair` kipinde bunları kurar.

### 8) Gateway hizmet geçişleri ve temizlik ipuçları

Doctor eski gateway hizmetlerini (launchd/systemd/schtasks) algılar ve
bunları kaldırmayı, ardından geçerli gateway
bağlantı noktasıyla OpenClaw hizmetini kurmayı önerir. Ayrıca ek gateway benzeri hizmetleri tarayabilir ve temizlik ipuçları yazdırabilir.
Profil adlı OpenClaw gateway hizmetleri birinci sınıf kabul edilir ve “ek” olarak
işaretlenmez.

### 8b) Başlangıç Matrix geçişi

Bir Matrix kanal hesabında bekleyen veya eyleme geçirilebilir eski durum geçişi varsa,
doctor (`--fix` / `--repair` kipinde) önce geçiş öncesi bir anlık görüntü oluşturur ve sonra
en iyi çabayla geçiş adımlarını çalıştırır: eski Matrix durum geçişi ve eski
şifreli durum hazırlığı. Her iki adım da ölümcül değildir; hatalar günlüğe yazılır ve
başlangıç devam eder. Salt okunur kipte (`--fix` olmadan `openclaw doctor`) bu kontrol
tamamen atlanır.

### 9) Güvenlik uyarıları

Doctor, bir sağlayıcı allowlist olmadan DM’lere açıksa veya
bir ilke tehlikeli şekilde yapılandırılmışsa uyarılar verir.

### 10) systemd linger (Linux)

Bir systemd kullanıcı hizmeti olarak çalışıyorsa, doctor logout sonrasında gateway’in
çalışmaya devam etmesi için lingering’in etkin olduğundan emin olur.

### 11) Çalışma alanı durumu (Skills, pluginler ve eski dizinler)

Doctor varsayılan agent için çalışma alanı durumunun bir özetini yazdırır:

- **Skills durumu**: uygun, gereksinimi eksik ve allowlist tarafından engellenmiş skill sayılarını sayar.
- **Eski çalışma alanı dizinleri**: `~/openclaw` veya diğer eski çalışma alanı dizinleri
  geçerli çalışma alanının yanında varsa uyarır.
- **Plugin durumu**: yüklenmiş/devre dışı/hatalı plugin sayılarını sayar; herhangi bir
  hata için plugin kimliklerini listeler; bundle plugin yeteneklerini raporlar.
- **Plugin uyumluluk uyarıları**: geçerli çalışma zamanı ile uyumluluk sorunu olan pluginleri işaretler.
- **Plugin tanılamaları**: plugin kayıt sistemi tarafından yayılan tüm yükleme zamanı uyarılarını veya hatalarını
  gösterir.

### 11b) Bootstrap dosya boyutu

Doctor, çalışma alanı bootstrap dosyalarının (örneğin `AGENTS.md`,
`CLAUDE.md` veya diğer enjekte edilen bağlam dosyaları) yapılandırılmış
karakter bütçesine yakın ya da bu bütçenin üzerinde olup olmadığını kontrol eder. Dosya başına ham ve enjekte edilmiş karakter sayıları, kesme
yüzdesi, kesme nedeni (`max/file` veya `max/total`) ve toplam bütçenin bir oranı olarak toplam enjekte edilmiş
karakter sayısını raporlar. Dosyalar kesilmişse veya sınıra yakınsa,
doctor `agents.defaults.bootstrapMaxChars`
ve `agents.defaults.bootstrapTotalMaxChars` ayarlarını düzenlemek için ipuçları yazdırır.

### 11c) Kabuk tamamlama

Doctor, geçerli kabuk için
(zsh, bash, fish veya PowerShell) sekme tamamlamasının kurulu olup olmadığını kontrol eder:

- Kabuk profili yavaş dinamik tamamlama deseni kullanıyorsa
  (`source <(openclaw completion ...)`), doctor bunu daha hızlı
  önbelleğe alınmış dosya varyantına yükseltir.
- Tamamlama profilde yapılandırılmış ama önbellek dosyası eksikse,
  doctor önbelleği otomatik olarak yeniden üretir.
- Hiç tamamlama yapılandırılmamışsa, doctor bunu kurmayı önerir
  (yalnızca etkileşimli kipte; `--non-interactive` ile atlanır).

Önbelleği elle yeniden üretmek için `openclaw completion --write-state` çalıştırın.

### 12) Gateway kimlik doğrulama kontrolleri (yerel belirteç)

Doctor, yerel gateway belirteç kimlik doğrulama hazır olma durumunu kontrol eder.

- Belirteç kipi bir belirteç gerektiriyor ve belirteç kaynağı yoksa, doctor bir tane üretmeyi önerir.
- `gateway.auth.token` SecretRef tarafından yönetiliyor ancak kullanılamıyorsa, doctor uyarır ve bunu düz metinle üzerine yazmaz.
- `openclaw doctor --generate-gateway-token`, yalnızca belirteç SecretRef yapılandırılmadığında üretimi zorlar.

### 12b) Salt okunur SecretRef farkındalıklı onarımlar

Bazı onarım akışlarının, çalışma zamanındaki hızlı başarısızlık davranışını zayıflatmadan yapılandırılmış kimlik bilgilerini incelemesi gerekir.

- `openclaw doctor --fix`, hedeflenmiş yapılandırma onarımları için artık durum ailesi komutlarıyla aynı salt okunur SecretRef özet modelini kullanır.
- Örnek: Telegram `allowFrom` / `groupAllowFrom` `@username` onarımı, yapılandırılmış bot kimlik bilgileri mevcutsa bunları kullanmayı dener.
- Telegram bot belirteci SecretRef üzerinden yapılandırılmış ancak geçerli komut yolunda kullanılamıyorsa, doctor kimlik bilgisinin yapılandırılmış ama kullanılamaz olduğunu bildirir ve belirteci eksik olarak çökmeden veya yanlış raporlamadan otomatik çözümlemeyi atlar.

### 13) Gateway sağlık kontrolü + yeniden başlatma

Doctor bir sağlık kontrolü çalıştırır ve gateway sağlıksız görünüyorsa
yeniden başlatmayı önerir.

### 13b) Bellek araması hazır olma durumu

Doctor, varsayılan agent için yapılandırılmış bellek araması gömme sağlayıcısının hazır olup olmadığını kontrol eder.
Davranış, yapılandırılmış backend ve sağlayıcıya bağlıdır:

- **QMD backend**: `qmd` ikilisinin kullanılabilir ve başlatılabilir olup olmadığını yoklar.
  Değilse, npm paketi ve elle ikili yol seçeneği dahil
  düzeltme yönlendirmesi yazdırır.
- **Açık yerel sağlayıcı**: yerel bir model dosyası veya tanınan
  uzak/indirilebilir model URL’si olup olmadığını kontrol eder. Eksikse, uzak sağlayıcıya geçmeyi önerir.
- **Açık uzak sağlayıcı** (`openai`, `voyage` vb.): ortamda veya auth deposunda API anahtarının
  mevcut olduğunu doğrular. Eksikse uygulanabilir düzeltme ipuçları yazdırır.
- **Otomatik sağlayıcı**: önce yerel model kullanılabilirliğini kontrol eder, ardından
  otomatik seçim sırasındaki her uzak sağlayıcıyı dener.

Bir gateway yoklama sonucu mevcut olduğunda (kontrol sırasında gateway sağlıklıysa),
doctor bunu CLI tarafından görülebilen yapılandırmayla çapraz doğrular ve
olası tüm uyumsuzlukları not eder.

Gömme hazır olma durumunu çalışma zamanında doğrulamak için `openclaw memory status --deep` kullanın.

### 14) Kanal durum uyarıları

Gateway sağlıklıysa, doctor kanal durum yoklaması çalıştırır ve
önerilen düzeltmelerle birlikte uyarıları raporlar.

### 15) Supervisor yapılandırma denetimi + onarım

Doctor kurulu supervisor yapılandırmasını (launchd/systemd/schtasks)
eksik veya güncel olmayan varsayılanlar açısından kontrol eder (ör. systemd network-online bağımlılıkları ve
yeniden başlatma gecikmesi). Uyuşmazlık bulduğunda,
bir güncelleme önerir ve hizmet dosyasını/görevi geçerli varsayılanlara göre
yeniden yazabilir.

Notlar:

- `openclaw doctor`, supervisor yapılandırmasını yeniden yazmadan önce ister.
- `openclaw doctor --yes`, varsayılan onarım istemlerini kabul eder.
- `openclaw doctor --repair`, önerilen düzeltmeleri istem olmadan uygular.
- `openclaw doctor --repair --force`, özel supervisor yapılandırmalarının üzerine yazar.
- Belirteç kimlik doğrulaması belirteç gerektiriyorsa ve `gateway.auth.token` SecretRef tarafından yönetiliyorsa, doctor hizmet kurulum/onarımları SecretRef’i doğrular ama çözümlenmiş düz metin belirteç değerlerini supervisor hizmet ortamı metadatasına kalıcı olarak yazmaz.
- Belirteç kimlik doğrulaması belirteç gerektiriyorsa ve yapılandırılmış belirteç SecretRef’i çözümlenmemişse, doctor kurulum/onarma yolunu uygulanabilir yönlendirmeyle engeller.
- Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmış ama `gateway.auth.mode` ayarlanmamışsa, doctor kurulum/onarma işlemini kip açıkça ayarlanana kadar engeller.
- Linux user-systemd birimleri için, doctor belirteç kayması kontrolleri artık hizmet auth metadatasını karşılaştırırken hem `Environment=` hem de `EnvironmentFile=` kaynaklarını içerir.
- Tam yeniden yazımı her zaman `openclaw gateway install --force` ile zorlayabilirsiniz.

### 16) Gateway çalışma zamanı + bağlantı noktası tanılamaları

Doctor hizmet çalışma zamanını (PID, son çıkış durumu) inceler ve hizmetin
kurulu olup gerçekte çalışmadığı durumlarda uyarır. Ayrıca gateway bağlantı noktasında
(varsayılan `18789`) çakışma olup olmadığını kontrol eder ve olası nedenleri raporlar (gateway zaten çalışıyor,
SSH tüneli).

### 17) Gateway çalışma zamanı en iyi uygulamaları

Doctor, gateway hizmeti Bun üzerinde veya sürüm yöneticili bir Node yolu üzerinde
(`nvm`, `fnm`, `volta`, `asdf` vb.) çalışıyorsa uyarır. WhatsApp + Telegram kanalları Node gerektirir
ve sürüm yöneticisi yolları yükseltmelerden sonra bozulabilir; çünkü hizmet
kabuk başlatma dosyalarınızı yüklemez. Doctor, kullanılabiliyorsa sistem Node kurulumuna
(Homebrew/apt/choco) geçmeyi önerir.

### 18) Yapılandırma yazımı + sihirbaz metadatası

Doctor tüm yapılandırma değişikliklerini kalıcı yazar ve doctor çalıştırmasını kaydetmek için
sihirbaz metadatasını damgalar.

### 19) Çalışma alanı ipuçları (yedekleme + bellek sistemi)

Doctor, çalışma alanı bellek sistemi eksikse bunu önerir ve çalışma alanı
zaten git altında değilse bir yedekleme ipucu yazdırır.

Çalışma alanı yapısı ve git ile yedekleme (önerilen özel GitHub veya GitLab) için
tam kılavuz olarak [/concepts/agent-workspace](/tr/concepts/agent-workspace) sayfasına bakın.
