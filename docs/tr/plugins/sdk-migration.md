---
read_when:
    - OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED uyarısını görüyorsunuz
    - OPENCLAW_EXTENSION_API_DEPRECATED uyarısını görüyorsunuz
    - Bir eklentiyi modern plugin mimarisine güncelliyorsunuz
    - Harici bir OpenClaw eklentisinin bakımını yapıyorsunuz
sidebarTitle: Migrate to SDK
summary: Eski geriye dönük uyumluluk katmanından modern plugin SDK'ya geçiş yapın
title: Plugin SDK Geçişi
x-i18n:
    generated_at: "2026-04-09T01:30:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60cbb6c8be30d17770887d490c14e3a4538563339a5206fb419e51e0558bbc07
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Plugin SDK Geçişi

OpenClaw, geniş bir geriye dönük uyumluluk katmanından, odaklanmış ve belgelenmiş içe aktarımlara sahip modern bir plugin
mimarısına geçti. Eklentiniz yeni mimariden önce
oluşturulduysa, bu kılavuz geçiş yapmanıza yardımcı olur.

## Neler değişiyor

Eski plugin sistemi, eklentilerin tek bir giriş noktasından ihtiyaç duydukları
her şeyi içe aktarmasına izin veren iki geniş yüzey sunuyordu:

- **`openclaw/plugin-sdk/compat`** — onlarca
  yardımcıyı yeniden dışa aktaran tek bir içe aktarma. Yeni plugin mimarisi
  oluşturulurken eski hook tabanlı eklentilerin çalışmaya devam etmesi için sunulmuştu.
- **`openclaw/extension-api`** — eklentilere gömülü agent çalıştırıcısı gibi
  ana bilgisayar tarafındaki yardımcı araçlara doğrudan erişim veren bir köprü.

Her iki yüzey de artık **kullanımdan kaldırılmıştır**. Çalışma zamanında hâlâ
çalışırlar, ancak yeni eklentiler bunları kullanmamalıdır ve mevcut eklentiler
bir sonraki büyük sürüm bunları kaldırmadan önce geçiş yapmalıdır.

<Warning>
  Geriye dönük uyumluluk katmanı gelecekteki bir büyük sürümde kaldırılacaktır.
  Hâlâ bu yüzeylerden içe aktarma yapan eklentiler bu olduğunda bozulacaktır.
</Warning>

## Bu neden değişti

Eski yaklaşım sorunlara yol açıyordu:

- **Yavaş başlangıç** — tek bir yardımcıyı içe aktarmak, birbiriyle ilgisiz onlarca modülü yüklüyordu
- **Döngüsel bağımlılıklar** — geniş yeniden dışa aktarmalar, içe aktarma döngüleri oluşturmayı kolaylaştırıyordu
- **Belirsiz API yüzeyi** — hangi dışa aktarmaların kararlı, hangilerinin dahili olduğunu ayırt etmenin bir yolu yoktu

Modern plugin SDK bunu düzeltir: her içe aktarma yolu (`openclaw/plugin-sdk/\<subpath\>`)
net bir amaca ve belgelenmiş bir sözleşmeye sahip küçük, kendi içinde yeterli bir modüldür.

Paketle gelen kanallar için eski sağlayıcı kolaylık katmanları da kaldırıldı. `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
kanal markalı yardımcı katmanları ve
`openclaw/plugin-sdk/telegram-core` gibi içe aktarmalar, kararlı
plugin sözleşmeleri değil, özel mono-repo kısayollarıydı. Bunun yerine dar kapsamlı genel SDK alt yollarını kullanın. Paketle gelen
plugin çalışma alanının içinde, sağlayıcıya ait yardımcıları ilgili eklentinin kendi
`api.ts` veya `runtime-api.ts` dosyasında tutun.

Güncel paketle gelen sağlayıcı örnekleri:

- Anthropic, Claude'a özgü akış yardımcılarını kendi `api.ts` /
  `contract-api.ts` katmanında tutar
- OpenAI, sağlayıcı oluşturucuları, varsayılan model yardımcılarını ve gerçek zamanlı sağlayıcı
  oluşturucularını kendi `api.ts` dosyasında tutar
- OpenRouter, sağlayıcı oluşturucuyu ve onboarding/yapılandırma yardımcılarını kendi
  `api.ts` dosyasında tutar

## Nasıl geçiş yapılır

<Steps>
  <Step title="Yerel onay işleyicilerini yetenek olgularına geçirin">
    Onay destekli kanal eklentileri artık yerel onay davranışını
    `approvalCapability.nativeRuntime` ve paylaşılan runtime-context kayıt defteri aracılığıyla açığa çıkarır.

    Temel değişiklikler:

    - `approvalCapability.handler.loadRuntime(...)` yerine
      `approvalCapability.nativeRuntime` kullanın
    - Onaya özgü kimlik doğrulama/teslimatı eski `plugin.auth` /
      `plugin.approvals` bağlantısından çıkarıp `approvalCapability` üzerine taşıyın
    - `ChannelPlugin.approvals`, ortak kanal eklentisi
      sözleşmesinden kaldırıldı; delivery/native/render alanlarını `approvalCapability` üzerine taşıyın
    - `plugin.auth`, yalnızca kanal giriş/çıkış akışları için kalır; oradaki
      onay kimlik doğrulama hook'ları artık core tarafından okunmaz
    - İstemciler, tokenlar veya Bolt
      uygulamaları gibi kanala ait çalışma zamanı nesnelerini `openclaw/plugin-sdk/channel-runtime-context` aracılığıyla kaydedin
    - Yerel onay işleyicilerinden eklentiye ait yeniden yönlendirme bildirimleri göndermeyin;
      core artık gerçek teslim sonuçlarından gelen başka yere yönlendirilmiş bildirimlerin sahibidir
    - `channelRuntime` değerini `createChannelManager(...)` içine geçirirken,
      gerçek bir `createPluginRuntime().channel` yüzeyi sağlayın. Kısmi stub'lar reddedilir.

    Güncel onay yeteneği
    düzeni için `/plugins/sdk-channel-plugins` bölümüne bakın.

  </Step>

  <Step title="Windows sarmalayıcı yedek davranışını denetleyin">
    Eklentiniz `openclaw/plugin-sdk/windows-spawn` kullanıyorsa, çözümlenemeyen Windows
    `.cmd`/`.bat` sarmalayıcıları artık açıkça
    `allowShellFallback: true` geçmediğiniz sürece kapalı başarısız olur.

    ```typescript
    // Önce
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Sonra
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Bunu yalnızca kasıtlı olarak
      // shell aracılı yedek çözümü kabul eden güvenilir uyumluluk çağıranları için ayarlayın.
      allowShellFallback: true,
    });
    ```

    Çağıran tarafınız bilinçli olarak shell yedeğine dayanmıyorsa, `allowShellFallback`
    ayarlamayın ve bunun yerine fırlatılan hatayı işleyin.

  </Step>

  <Step title="Kullanımdan kaldırılmış içe aktarımları bulun">
    Eklentinizde bu kullanımdan kaldırılmış yüzeylerden birinden yapılan içe aktarımları arayın:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Bunları odaklanmış içe aktarımlarla değiştirin">
    Eski yüzeydeki her dışa aktarma, belirli bir modern içe aktarma yoluna eşlenir:

    ```typescript
    // Önce (kullanımdan kaldırılmış geriye dönük uyumluluk katmanı)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Sonra (modern odaklı içe aktarımlar)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Ana bilgisayar tarafındaki yardımcılar için doğrudan içe aktarmak yerine
    enjekte edilen plugin çalışma zamanını kullanın:

    ```typescript
    // Önce (kullanımdan kaldırılmış extension-api köprüsü)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Sonra (enjekte edilen çalışma zamanı)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Aynı desen diğer eski köprü yardımcıları için de geçerlidir:

    | Eski içe aktarma | Modern eşdeğer |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | oturum deposu yardımcıları | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Derleyin ve test edin">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## İçe aktarma yolu başvurusu

<Accordion title="Yaygın içe aktarma yolu tablosu">
  | İçe aktarma yolu | Amaç | Temel dışa aktarmalar |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Kanonik plugin giriş yardımcısı | `definePluginEntry` |
  | `plugin-sdk/core` | Kanal giriş tanımları/oluşturucuları için eski şemsiye yeniden dışa aktarma | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Kök yapılandırma şeması dışa aktarması | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Tek sağlayıcılı giriş yardımcısı | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Odaklanmış kanal giriş tanımları ve oluşturucuları | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Paylaşılan kurulum sihirbazı yardımcıları | Allowlist istemleri, kurulum durumu oluşturucuları |
  | `plugin-sdk/setup-runtime` | Kurulum zamanı çalışma zamanı yardımcıları | Güvenli içe aktarma yapılabilen kurulum yama adaptörleri, lookup-note yardımcıları, `promptResolvedAllowFrom`, `splitSetupEntries`, devredilen kurulum proxy'leri |
  | `plugin-sdk/setup-adapter-runtime` | Kurulum adaptörü yardımcıları | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Kurulum araç yardımcıları | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Çok hesaplı yardımcılar | Hesap listesi/yapılandırma/eylem geçidi yardımcıları |
  | `plugin-sdk/account-id` | Hesap kimliği yardımcıları | `DEFAULT_ACCOUNT_ID`, hesap kimliği normalleştirme |
  | `plugin-sdk/account-resolution` | Hesap arama yardımcıları | Hesap arama + varsayılan yedek yardımcıları |
  | `plugin-sdk/account-helpers` | Dar kapsamlı hesap yardımcıları | Hesap listesi/hesap eylemi yardımcıları |
  | `plugin-sdk/channel-setup` | Kurulum sihirbazı adaptörleri | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, ayrıca `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | DM eşleme ilkel öğeleri | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Yanıt öneki + yazıyor durum bağlantıları | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Yapılandırma adaptörü fabrikaları | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Yapılandırma şeması oluşturucuları | Kanal yapılandırma şeması türleri |
  | `plugin-sdk/telegram-command-config` | Telegram komut yapılandırma yardımcıları | Komut adı normalleştirme, açıklama kırpma, yinelenen/çatışan doğrulama |
  | `plugin-sdk/channel-policy` | Grup/DM ilke çözümleme | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Hesap durumu izleme | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Gelen zarf yardımcıları | Paylaşılan rota + zarf oluşturucu yardımcıları |
  | `plugin-sdk/inbound-reply-dispatch` | Gelen yanıt yardımcıları | Paylaşılan kaydet ve dağıt yardımcıları |
  | `plugin-sdk/messaging-targets` | Mesajlaşma hedefi ayrıştırma | Hedef ayrıştırma/eşleştirme yardımcıları |
  | `plugin-sdk/outbound-media` | Giden medya yardımcıları | Paylaşılan giden medya yükleme |
  | `plugin-sdk/outbound-runtime` | Giden çalışma zamanı yardımcıları | Giden kimlik/gönderim temsilcisi yardımcıları |
  | `plugin-sdk/thread-bindings-runtime` | İş parçacığı bağlama yardımcıları | İş parçacığı bağlama yaşam döngüsü ve adaptör yardımcıları |
  | `plugin-sdk/agent-media-payload` | Eski medya yükü yardımcıları | Eski alan düzenleri için agent medya yükü oluşturucu |
  | `plugin-sdk/channel-runtime` | Kullanımdan kaldırılmış uyumluluk shim'i | Yalnızca eski kanal çalışma zamanı yardımcıları |
  | `plugin-sdk/channel-send-result` | Gönderim sonucu türleri | Yanıt sonucu türleri |
  | `plugin-sdk/runtime-store` | Kalıcı plugin depolaması | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Geniş çalışma zamanı yardımcıları | Çalışma zamanı/günlükleme/yedekleme/plugin yükleme yardımcıları |
  | `plugin-sdk/runtime-env` | Dar kapsamlı çalışma zamanı ortam yardımcıları | Logger/çalışma zamanı ortamı, zaman aşımı, yeniden deneme ve backoff yardımcıları |
  | `plugin-sdk/plugin-runtime` | Paylaşılan plugin çalışma zamanı yardımcıları | Plugin komutları/hook'ları/http/etkileşimli yardımcılar |
  | `plugin-sdk/hook-runtime` | Hook işlem hattı yardımcıları | Paylaşılan webhook/dahili hook işlem hattı yardımcıları |
  | `plugin-sdk/lazy-runtime` | Tembel çalışma zamanı yardımcıları | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | İşlem yardımcıları | Paylaşılan exec yardımcıları |
  | `plugin-sdk/cli-runtime` | CLI çalışma zamanı yardımcıları | Komut biçimlendirme, beklemeler, sürüm yardımcıları |
  | `plugin-sdk/gateway-runtime` | Gateway yardımcıları | Gateway istemcisi ve kanal durumu yama yardımcıları |
  | `plugin-sdk/config-runtime` | Yapılandırma yardımcıları | Yapılandırma yükleme/yazma yardımcıları |
  | `plugin-sdk/telegram-command-config` | Telegram komut yardımcıları | Paketle gelen Telegram sözleşme yüzeyi kullanılamadığında yedek olarak kararlı Telegram komut doğrulama yardımcıları |
  | `plugin-sdk/approval-runtime` | Onay istemi yardımcıları | Exec/plugin onay yükü, onay yeteneği/profil yardımcıları, yerel onay yönlendirme/çalışma zamanı yardımcıları |
  | `plugin-sdk/approval-auth-runtime` | Onay kimlik doğrulama yardımcıları | Onaylayan çözümleme, aynı sohbet eylem kimlik doğrulaması |
  | `plugin-sdk/approval-client-runtime` | Onay istemcisi yardımcıları | Yerel exec onay profili/filtre yardımcıları |
  | `plugin-sdk/approval-delivery-runtime` | Onay teslimat yardımcıları | Yerel onay yeteneği/teslimat adaptörleri |
  | `plugin-sdk/approval-gateway-runtime` | Onay gateway yardımcıları | Paylaşılan onay gateway çözümleme yardımcısı |
  | `plugin-sdk/approval-handler-adapter-runtime` | Onay adaptörü yardımcıları | Sık kullanılan kanal giriş noktaları için hafif yerel onay adaptörü yükleme yardımcıları |
  | `plugin-sdk/approval-handler-runtime` | Onay işleyici yardımcıları | Daha geniş onay işleyici çalışma zamanı yardımcıları; dar adaptör/gateway katmanları yeterliyse onları tercih edin |
  | `plugin-sdk/approval-native-runtime` | Onay hedefi yardımcıları | Yerel onay hedefi/hesap bağlama yardımcıları |
  | `plugin-sdk/approval-reply-runtime` | Onay yanıtı yardımcıları | Exec/plugin onay yanıt yükü yardımcıları |
  | `plugin-sdk/channel-runtime-context` | Kanal runtime-context yardımcıları | Genel kanal runtime-context register/get/watch yardımcıları |
  | `plugin-sdk/security-runtime` | Güvenlik yardımcıları | Paylaşılan güven, DM geçitleme, dış içerik ve gizli bilgi toplama yardımcıları |
  | `plugin-sdk/ssrf-policy` | SSRF ilke yardımcıları | Ana bilgisayar allowlist ve özel ağ ilkesi yardımcıları |
  | `plugin-sdk/ssrf-runtime` | SSRF çalışma zamanı yardımcıları | Sabitlenmiş dağıtıcı, korumalı fetch, SSRF ilke yardımcıları |
  | `plugin-sdk/collection-runtime` | Sınırlı önbellek yardımcıları | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Tanılama geçitleme yardımcıları | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Hata biçimlendirme yardımcıları | `formatUncaughtError`, `isApprovalNotFoundError`, hata grafiği yardımcıları |
  | `plugin-sdk/fetch-runtime` | Sarılmış fetch/proxy yardımcıları | `resolveFetch`, proxy yardımcıları |
  | `plugin-sdk/host-runtime` | Ana bilgisayar normalleştirme yardımcıları | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Yeniden deneme yardımcıları | `RetryConfig`, `retryAsync`, ilke çalıştırıcıları |
  | `plugin-sdk/allow-from` | Allowlist biçimlendirme | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Allowlist girdi eşleme | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Komut geçitleme ve komut yüzeyi yardımcıları | `resolveControlCommandGate`, gönderen yetkilendirme yardımcıları, komut kayıt defteri yardımcıları |
  | `plugin-sdk/command-status` | Komut durumu/yardım oluşturucuları | `buildCommandsMessage`, `buildCommandsMessagePaginated`, `buildHelpMessage` |
  | `plugin-sdk/secret-input` | Gizli girdi ayrıştırma | Gizli girdi yardımcıları |
  | `plugin-sdk/webhook-ingress` | Webhook istek yardımcıları | Webhook hedef yardımcıları |
  | `plugin-sdk/webhook-request-guards` | Webhook gövdesi koruma yardımcıları | İstek gövdesi okuma/sınır yardımcıları |
  | `plugin-sdk/reply-runtime` | Paylaşılan yanıt çalışma zamanı | Gelen dağıtım, heartbeat, yanıt planlayıcı, parçalama |
  | `plugin-sdk/reply-dispatch-runtime` | Dar kapsamlı yanıt dağıtım yardımcıları | Sonlandırma + sağlayıcı dağıtım yardımcıları |
  | `plugin-sdk/reply-history` | Yanıt geçmişi yardımcıları | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Yanıt referansı planlama | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Yanıt parça yardımcıları | Metin/markdown parça yardımcıları |
  | `plugin-sdk/session-store-runtime` | Oturum deposu yardımcıları | Depo yolu + updated-at yardımcıları |
  | `plugin-sdk/state-paths` | Durum yolu yardımcıları | Durum ve OAuth dizin yardımcıları |
  | `plugin-sdk/routing` | Yönlendirme/oturum anahtarı yardımcıları | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, oturum anahtarı normalleştirme yardımcıları |
  | `plugin-sdk/status-helpers` | Kanal durumu yardımcıları | Kanal/hesap durumu özeti oluşturucuları, çalışma zamanı durumu varsayılanları, sorun meta verisi yardımcıları |
  | `plugin-sdk/target-resolver-runtime` | Hedef çözücü yardımcıları | Paylaşılan hedef çözücü yardımcıları |
  | `plugin-sdk/string-normalization-runtime` | Dize normalleştirme yardımcıları | Slug/dize normalleştirme yardımcıları |
  | `plugin-sdk/request-url` | İstek URL yardımcıları | İstek benzeri girdilerden dize URL'leri çıkarma |
  | `plugin-sdk/run-command` | Zamanlanmış komut yardımcıları | Normalleştirilmiş stdout/stderr ile zamanlanmış komut çalıştırıcı |
  | `plugin-sdk/param-readers` | Parametre okuyucular | Yaygın araç/CLI parametre okuyucuları |
  | `plugin-sdk/tool-payload` | Araç yükü çıkarma | Araç sonuç nesnelerinden normalize edilmiş yükleri çıkarma |
  | `plugin-sdk/tool-send` | Araç gönderimi çıkarma | Araç argümanlarından kanonik gönderim hedefi alanlarını çıkarma |
  | `plugin-sdk/temp-path` | Geçici yol yardımcıları | Paylaşılan geçici indirme yolu yardımcıları |
  | `plugin-sdk/logging-core` | Günlükleme yardımcıları | Alt sistem logger ve redaksiyon yardımcıları |
  | `plugin-sdk/markdown-table-runtime` | Markdown tablosu yardımcıları | Markdown tablosu modu yardımcıları |
  | `plugin-sdk/reply-payload` | Mesaj yanıt türleri | Yanıt yükü türleri |
  | `plugin-sdk/provider-setup` | Özenle seçilmiş yerel/kendi barındırılan sağlayıcı kurulum yardımcıları | Kendi barındırılan sağlayıcı keşif/yapılandırma yardımcıları |
  | `plugin-sdk/self-hosted-provider-setup` | Odaklanmış OpenAI uyumlu kendi barındırılan sağlayıcı kurulum yardımcıları | Aynı kendi barındırılan sağlayıcı keşif/yapılandırma yardımcıları |
  | `plugin-sdk/provider-auth-runtime` | Sağlayıcı çalışma zamanı kimlik doğrulama yardımcıları | Çalışma zamanı API anahtarı çözümleme yardımcıları |
  | `plugin-sdk/provider-auth-api-key` | Sağlayıcı API anahtarı kurulum yardımcıları | API anahtarı onboarding/profil yazma yardımcıları |
  | `plugin-sdk/provider-auth-result` | Sağlayıcı auth-result yardımcıları | Standart OAuth auth-result oluşturucu |
  | `plugin-sdk/provider-auth-login` | Sağlayıcı etkileşimli giriş yardımcıları | Paylaşılan etkileşimli giriş yardımcıları |
  | `plugin-sdk/provider-env-vars` | Sağlayıcı ortam değişkeni yardımcıları | Sağlayıcı kimlik doğrulama ortam değişkeni arama yardımcıları |
  | `plugin-sdk/provider-model-shared` | Paylaşılan sağlayıcı model/replay yardımcıları | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, paylaşılan replay-policy oluşturucuları, sağlayıcı uç nokta yardımcıları ve model kimliği normalleştirme yardımcıları |
  | `plugin-sdk/provider-catalog-shared` | Paylaşılan sağlayıcı katalog yardımcıları | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Sağlayıcı onboarding yamaları | Onboarding yapılandırma yardımcıları |
  | `plugin-sdk/provider-http` | Sağlayıcı HTTP yardımcıları | Genel sağlayıcı HTTP/uç nokta yeteneği yardımcıları |
  | `plugin-sdk/provider-web-fetch` | Sağlayıcı web-fetch yardımcıları | Web-fetch sağlayıcı kayıt/önbellek yardımcıları |
  | `plugin-sdk/provider-web-search-config-contract` | Sağlayıcı web-search yapılandırma yardımcıları | Plugin etkinleştirme bağlantısına ihtiyaç duymayan sağlayıcılar için dar web-search yapılandırma/kimlik bilgisi yardımcıları |
  | `plugin-sdk/provider-web-search-contract` | Sağlayıcı web-search sözleşme yardımcıları | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` ve kapsamlı kimlik bilgisi ayarlayıcıları/getter'ları gibi dar web-search yapılandırma/kimlik bilgisi sözleşme yardımcıları |
  | `plugin-sdk/provider-web-search` | Sağlayıcı web-search yardımcıları | Web-search sağlayıcı kayıt/önbellek/çalışma zamanı yardımcıları |
  | `plugin-sdk/provider-tools` | Sağlayıcı araç/şema uyumluluk yardımcıları | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini şema temizleme + tanılama ve `resolveXaiModelCompatPatch` / `applyXaiModelCompat` gibi xAI uyumluluk yardımcıları |
  | `plugin-sdk/provider-usage` | Sağlayıcı kullanım yardımcıları | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` ve diğer sağlayıcı kullanım yardımcıları |
  | `plugin-sdk/provider-stream` | Sağlayıcı akış sarmalayıcı yardımcıları | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, akış sarmalayıcı türleri ve paylaşılan Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot sarmalayıcı yardımcıları |
  | `plugin-sdk/keyed-async-queue` | Sıralı async kuyruk | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Paylaşılan medya yardımcıları | Medya fetch/dönüştürme/depolama yardımcıları ve medya yükü oluşturucuları |
  | `plugin-sdk/media-generation-runtime` | Paylaşılan medya üretimi yardımcıları | Görüntü/video/müzik üretimi için paylaşılan failover yardımcıları, aday seçimi ve eksik model mesajları |
  | `plugin-sdk/media-understanding` | Media-understanding yardımcıları | Media-understanding sağlayıcı türleri ve sağlayıcıya dönük görüntü/ses yardımcı dışa aktarmaları |
  | `plugin-sdk/text-runtime` | Paylaşılan metin yardımcıları | Asistan tarafından görülebilen metin kaldırma, markdown oluşturma/parçalama/tablo yardımcıları, redaksiyon yardımcıları, directive-tag yardımcıları, güvenli metin araçları ve ilgili metin/günlükleme yardımcıları |
  | `plugin-sdk/text-chunking` | Metin parçalama yardımcıları | Giden metin parçalama yardımcısı |
  | `plugin-sdk/speech` | Konuşma yardımcıları | Konuşma sağlayıcı türleri ve sağlayıcıya dönük directive, kayıt defteri ve doğrulama yardımcıları |
  | `plugin-sdk/speech-core` | Paylaşılan konuşma çekirdeği | Konuşma sağlayıcı türleri, kayıt defteri, yönergeler, normalleştirme |
  | `plugin-sdk/realtime-transcription` | Gerçek zamanlı transkripsiyon yardımcıları | Sağlayıcı türleri ve kayıt defteri yardımcıları |
  | `plugin-sdk/realtime-voice` | Gerçek zamanlı ses yardımcıları | Sağlayıcı türleri ve kayıt defteri yardımcıları |
  | `plugin-sdk/image-generation-core` | Paylaşılan görüntü üretimi çekirdeği | Görüntü üretimi türleri, failover, kimlik doğrulama ve kayıt defteri yardımcıları |
  | `plugin-sdk/music-generation` | Müzik üretimi yardımcıları | Müzik üretimi sağlayıcı/istek/sonuç türleri |
  | `plugin-sdk/music-generation-core` | Paylaşılan müzik üretimi çekirdeği | Müzik üretimi türleri, failover yardımcıları, sağlayıcı arama ve model başvurusu ayrıştırma |
  | `plugin-sdk/video-generation` | Video üretimi yardımcıları | Video üretimi sağlayıcı/istek/sonuç türleri |
  | `plugin-sdk/video-generation-core` | Paylaşılan video üretimi çekirdeği | Video üretimi türleri, failover yardımcıları, sağlayıcı arama ve model başvurusu ayrıştırma |
  | `plugin-sdk/interactive-runtime` | Etkileşimli yanıt yardımcıları | Etkileşimli yanıt yükü normalleştirme/indirgeme |
  | `plugin-sdk/channel-config-primitives` | Kanal yapılandırma ilkel öğeleri | Dar kanal config-schema ilkel öğeleri |
  | `plugin-sdk/channel-config-writes` | Kanal yapılandırma yazma yardımcıları | Kanal yapılandırma yazma yetkilendirme yardımcıları |
  | `plugin-sdk/channel-plugin-common` | Paylaşılan kanal prelude | Paylaşılan kanal eklentisi prelude dışa aktarmaları |
  | `plugin-sdk/channel-status` | Kanal durumu yardımcıları | Paylaşılan kanal durumu anlık görüntü/özet yardımcıları |
  | `plugin-sdk/allowlist-config-edit` | Allowlist yapılandırma yardımcıları | Allowlist yapılandırma düzenleme/okuma yardımcıları |
  | `plugin-sdk/group-access` | Grup erişim yardımcıları | Paylaşılan grup erişim kararı yardımcıları |
  | `plugin-sdk/direct-dm` | Doğrudan DM yardımcıları | Paylaşılan doğrudan DM kimlik doğrulama/koruma yardımcıları |
  | `plugin-sdk/extension-shared` | Paylaşılan extension yardımcıları | Pasif kanal/durum ve ortam proxy yardımcısı ilkel öğeleri |
  | `plugin-sdk/webhook-targets` | Webhook hedef yardımcıları | Webhook hedef kayıt defteri ve rota kurulum yardımcıları |
  | `plugin-sdk/webhook-path` | Webhook yol yardımcıları | Webhook yolu normalleştirme yardımcıları |
  | `plugin-sdk/web-media` | Paylaşılan web medya yardımcıları | Uzak/yerel medya yükleme yardımcıları |
  | `plugin-sdk/zod` | Zod yeniden dışa aktarması | Plugin SDK kullanıcıları için yeniden dışa aktarılan `zod` |
  | `plugin-sdk/memory-core` | Paketle gelen memory-core yardımcıları | Bellek yöneticisi/yapılandırma/dosya/CLI yardımcı yüzeyi |
  | `plugin-sdk/memory-core-engine-runtime` | Bellek motoru çalışma zamanı cephesi | Bellek dizini/arama çalışma zamanı cephesi |
  | `plugin-sdk/memory-core-host-engine-foundation` | Bellek ana bilgisayar temel motoru | Bellek ana bilgisayar temel motoru dışa aktarmaları |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Bellek ana bilgisayar gömme motoru | Bellek ana bilgisayar gömme motoru dışa aktarmaları |
  | `plugin-sdk/memory-core-host-engine-qmd` | Bellek ana bilgisayar QMD motoru | Bellek ana bilgisayar QMD motoru dışa aktarmaları |
  | `plugin-sdk/memory-core-host-engine-storage` | Bellek ana bilgisayar depolama motoru | Bellek ana bilgisayar depolama motoru dışa aktarmaları |
  | `plugin-sdk/memory-core-host-multimodal` | Bellek ana bilgisayar multimodal yardımcıları | Bellek ana bilgisayar multimodal yardımcıları |
  | `plugin-sdk/memory-core-host-query` | Bellek ana bilgisayar sorgu yardımcıları | Bellek ana bilgisayar sorgu yardımcıları |
  | `plugin-sdk/memory-core-host-secret` | Bellek ana bilgisayar gizli bilgi yardımcıları | Bellek ana bilgisayar gizli bilgi yardımcıları |
  | `plugin-sdk/memory-core-host-events` | Bellek ana bilgisayar olay günlüğü yardımcıları | Bellek ana bilgisayar olay günlüğü yardımcıları |
  | `plugin-sdk/memory-core-host-status` | Bellek ana bilgisayar durum yardımcıları | Bellek ana bilgisayar durum yardımcıları |
  | `plugin-sdk/memory-core-host-runtime-cli` | Bellek ana bilgisayar CLI çalışma zamanı | Bellek ana bilgisayar CLI çalışma zamanı yardımcıları |
  | `plugin-sdk/memory-core-host-runtime-core` | Bellek ana bilgisayar çekirdek çalışma zamanı | Bellek ana bilgisayar çekirdek çalışma zamanı yardımcıları |
  | `plugin-sdk/memory-core-host-runtime-files` | Bellek ana bilgisayar dosya/çalışma zamanı yardımcıları | Bellek ana bilgisayar dosya/çalışma zamanı yardımcıları |
  | `plugin-sdk/memory-host-core` | Bellek ana bilgisayar çekirdek çalışma zamanı diğer adı | Bellek ana bilgisayar çekirdek çalışma zamanı yardımcıları için sağlayıcıdan bağımsız diğer ad |
  | `plugin-sdk/memory-host-events` | Bellek ana bilgisayar olay günlüğü diğer adı | Bellek ana bilgisayar olay günlüğü yardımcıları için sağlayıcıdan bağımsız diğer ad |
  | `plugin-sdk/memory-host-files` | Bellek ana bilgisayar dosya/çalışma zamanı diğer adı | Bellek ana bilgisayar dosya/çalışma zamanı yardımcıları için sağlayıcıdan bağımsız diğer ad |
  | `plugin-sdk/memory-host-markdown` | Yönetilen markdown yardımcıları | Belleğe bitişik eklentiler için paylaşılan yönetilen-markdown yardımcıları |
  | `plugin-sdk/memory-host-search` | Etkin bellek arama cephesi | Tembel etkin bellek search-manager çalışma zamanı cephesi |
  | `plugin-sdk/memory-host-status` | Bellek ana bilgisayar durumu diğer adı | Bellek ana bilgisayar durum yardımcıları için sağlayıcıdan bağımsız diğer ad |
  | `plugin-sdk/memory-lancedb` | Paketle gelen memory-lancedb yardımcıları | Memory-lancedb yardımcı yüzeyi |
  | `plugin-sdk/testing` | Test araçları | Test yardımcıları ve mock'lar |
</Accordion>

Bu tablo, tam SDK
yüzeyi değil, bilinçli olarak yaygın geçiş alt kümesidir. 200'den fazla giriş noktasının tam listesi
`scripts/lib/plugin-sdk-entrypoints.json` içinde bulunur.

Bu liste hâlâ
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` ve `plugin-sdk/matrix*` gibi bazı paketle gelen eklenti yardımcı katmanlarını içerir. Bunlar paketle gelen eklenti bakımı ve uyumluluk için dışa aktarılmaya devam eder, ancak bilinçli olarak
yaygın geçiş tablosunda yer almaz ve yeni eklenti kodu için önerilen hedef değildir.

Aynı kural aşağıdaki diğer paketle gelen yardımcı aileleri için de geçerlidir:

- tarayıcı destek yardımcıları: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` ve `plugin-sdk/voice-call` gibi paketle gelen yardımcı/eklenti yüzeyleri

`plugin-sdk/github-copilot-token` şu anda dar kapsamlı token-yardımcı
yüzeyini `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` ve `resolveCopilotApiToken` ile açığa çıkarır.

Yapılan işe en uygun en dar içe aktarmayı kullanın. Bir dışa aktarma bulamıyorsanız,
`src/plugin-sdk/` içindeki kaynağı kontrol edin veya Discord'da sorun.

## Kaldırma zaman çizelgesi

| Ne zaman               | Ne olur                                                                |
| ---------------------- | ----------------------------------------------------------------------- |
| **Şimdi**              | Kullanımdan kaldırılmış yüzeyler çalışma zamanında uyarılar üretir      |
| **Bir sonraki büyük sürüm** | Kullanımdan kaldırılmış yüzeyler kaldırılır; hâlâ onları kullanan eklentiler başarısız olur |

Tüm core eklentileri zaten geçirildi. Harici eklentiler
bir sonraki büyük sürümden önce geçiş yapmalıdır.

## Uyarıları geçici olarak bastırma

Geçiş üzerinde çalışırken bu ortam değişkenlerini ayarlayın:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Bu geçici bir kaçış kapağıdır, kalıcı bir çözüm değildir.

## İlgili

- [Getting Started](/tr/plugins/building-plugins) — ilk eklentinizi oluşturun
- [SDK Overview](/tr/plugins/sdk-overview) — tam alt yol içe aktarma başvurusu
- [Channel Plugins](/tr/plugins/sdk-channel-plugins) — kanal eklentileri oluşturma
- [Provider Plugins](/tr/plugins/sdk-provider-plugins) — sağlayıcı eklentileri oluşturma
- [Plugin Internals](/tr/plugins/architecture) — mimariye derinlemesine bakış
- [Plugin Manifest](/tr/plugins/manifest) — manifest şeması başvurusu
