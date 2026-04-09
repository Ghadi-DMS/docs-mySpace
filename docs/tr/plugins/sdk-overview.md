---
read_when:
    - Hangi SDK alt yolundan içe aktarma yapmanız gerektiğini bilmeniz gerekiyor
    - OpenClawPluginApi üzerindeki tüm kayıt yöntemleri için bir başvuru istiyorsunuz
    - Belirli bir SDK dışa aktarımını arıyorsunuz
sidebarTitle: SDK Overview
summary: İçe aktarma eşlemesi, kayıt API başvurusu ve SDK mimarisi
title: Plugin SDK Genel Bakış
x-i18n:
    generated_at: "2026-04-09T01:31:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf205af060971931df97dca4af5110ce173d2b7c12f56ad7c62d664a402f2381
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK Genel Bakış

Plugin SDK, plugin'ler ile çekirdek arasındaki türlendirilmiş sözleşmedir. Bu sayfa,
**neyi içe aktaracağınızın** ve **neleri kaydedebileceğinizin** başvurusudur.

<Tip>
  **Nasıl yapılır kılavuzu mu arıyorsunuz?**
  - İlk plugin mi? [Başlangıç](/tr/plugins/building-plugins) ile başlayın
  - Kanal plugin'i mi? Bkz. [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins)
  - Sağlayıcı plugin'i mi? Bkz. [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins)
</Tip>

## İçe aktarma kuralı

Her zaman belirli bir alt yoldan içe aktarın:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Her alt yol küçük ve kendi içinde yeterli bir modüldür. Bu, başlangıcı hızlı tutar
ve dairesel bağımlılık sorunlarını önler. Kanala özgü giriş/derleme yardımcıları için
`openclaw/plugin-sdk/channel-core` yolunu tercih edin; daha geniş şemsiye yüzey ve
`buildChannelConfigSchema` gibi paylaşılan yardımcılar için
`openclaw/plugin-sdk/core` yolunu kullanın.

`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` gibi
sağlayıcı adlı kolaylık yüzeyleri veya kanal markalı yardımcı yüzeyler
eklemeyin ya da bunlara bağımlı olmayın. Paketlenmiş plugin'ler genel
SDK alt yollarını kendi `api.ts` veya `runtime-api.ts` barrel dosyaları içinde
birleştirmelidir; çekirdek ise ya bu plugin'e özgü yerel barrel dosyalarını
kullanmalı ya da ihtiyaç gerçekten kanallar arasıysa dar kapsamlı genel bir SDK
sözleşmesi eklemelidir.

Oluşturulmuş dışa aktarma eşlemesi hâlâ `plugin-sdk/feishu`,
`plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup` ve `plugin-sdk/matrix*`
gibi küçük bir paketlenmiş-plugin yardımcı yüzey kümesi içerir. Bu
alt yollar yalnızca paketlenmiş-plugin bakımı ve uyumluluğu için vardır;
aşağıdaki ortak tabloda bilerek atlanmıştır ve yeni üçüncü taraf plugin'ler için
önerilen içe aktarma yolu değildir.

## Alt yol başvurusu

Amaca göre gruplanmış en yaygın kullanılan alt yollar. 200'den fazla alt yolun
oluşturulmuş tam listesi `scripts/lib/plugin-sdk-entrypoints.json` içinde bulunur.

Ayrılmış paketlenmiş-plugin yardımcı alt yolları bu oluşturulmuş listede yine de görünür.
Bir belge sayfası bunlardan birini açıkça herkese açık olarak tanıtmadıkça,
bunları uygulama ayrıntısı/uyumluluk yüzeyleri olarak değerlendirin.

### Plugin girişi

| Alt yol                     | Temel dışa aktarımlar                                                                                                               |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                 |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                    |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                   |

<AccordionGroup>
  <Accordion title="Kanal alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Kök `openclaw.json` Zod şema dışa aktarımı (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard` ve ayrıca `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Paylaşılan kurulum sihirbazı yardımcıları, allowlist istemleri, kurulum durumu oluşturucuları |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Çok hesaplı yapılandırma/eylem kapısı yardımcıları, varsayılan hesap geri dönüş yardımcıları |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, hesap kimliği normalleştirme yardımcıları |
    | `plugin-sdk/account-resolution` | Hesap arama + varsayılan geri dönüş yardımcıları |
    | `plugin-sdk/account-helpers` | Dar kapsamlı hesap listesi/hesap eylemi yardımcıları |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Kanal yapılandırma şeması türleri |
    | `plugin-sdk/telegram-command-config` | Paketlenmiş sözleşme geri dönüşüyle Telegram özel komut normalleştirme/doğrulama yardımcıları |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Paylaşılan gelen yönlendirme + zarf oluşturucu yardımcıları |
    | `plugin-sdk/inbound-reply-dispatch` | Paylaşılan gelen kaydetme ve dağıtma yardımcıları |
    | `plugin-sdk/messaging-targets` | Hedef ayrıştırma/eşleştirme yardımcıları |
    | `plugin-sdk/outbound-media` | Paylaşılan giden medya yükleme yardımcıları |
    | `plugin-sdk/outbound-runtime` | Giden kimlik/gönderim temsilci yardımcıları |
    | `plugin-sdk/thread-bindings-runtime` | İş parçacığı bağlama yaşam döngüsü ve bağdaştırıcı yardımcıları |
    | `plugin-sdk/agent-media-payload` | Eski agent medya yükü oluşturucusu |
    | `plugin-sdk/conversation-runtime` | Konuşma/iş parçacığı bağlama, eşleme ve yapılandırılmış bağlama yardımcıları |
    | `plugin-sdk/runtime-config-snapshot` | Çalışma zamanı yapılandırma anlık görüntüsü yardımcısı |
    | `plugin-sdk/runtime-group-policy` | Çalışma zamanı grup ilkesi çözümleme yardımcıları |
    | `plugin-sdk/channel-status` | Paylaşılan kanal durum anlık görüntüsü/özeti yardımcıları |
    | `plugin-sdk/channel-config-primitives` | Dar kapsamlı kanal yapılandırma şeması ilkel öğeleri |
    | `plugin-sdk/channel-config-writes` | Kanal yapılandırma yazma yetkilendirme yardımcıları |
    | `plugin-sdk/channel-plugin-common` | Paylaşılan kanal plugin başlangıç dışa aktarımları |
    | `plugin-sdk/allowlist-config-edit` | Allowlist yapılandırması düzenleme/okuma yardımcıları |
    | `plugin-sdk/group-access` | Paylaşılan grup erişimi karar yardımcıları |
    | `plugin-sdk/direct-dm` | Paylaşılan doğrudan DM auth/koruma yardımcıları |
    | `plugin-sdk/interactive-runtime` | Etkileşimli yanıt yükü normalleştirme/indirgeme yardımcıları |
    | `plugin-sdk/channel-inbound` | Gelen debounce, mention eşleştirme, mention ilkesi yardımcıları ve zarf yardımcıları |
    | `plugin-sdk/channel-send-result` | Yanıt sonuç türleri |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Hedef ayrıştırma/eşleştirme yardımcıları |
    | `plugin-sdk/channel-contract` | Kanal sözleşmesi türleri |
    | `plugin-sdk/channel-feedback` | Geri bildirim/reaksiyon bağlantıları |
    | `plugin-sdk/channel-secret-runtime` | `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` gibi dar kapsamlı gizli sözleşme yardımcıları ve gizli hedef türleri |
  </Accordion>

  <Accordion title="Sağlayıcı alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Düzenlenmiş yerel/kendi barındırılan sağlayıcı kurulum yardımcıları |
    | `plugin-sdk/self-hosted-provider-setup` | Odaklanmış OpenAI uyumlu kendi barındırılan sağlayıcı kurulum yardımcıları |
    | `plugin-sdk/cli-backend` | CLI arka uç varsayılanları + watchdog sabitleri |
    | `plugin-sdk/provider-auth-runtime` | Sağlayıcı plugin'leri için çalışma zamanı API anahtarı çözümleme yardımcıları |
    | `plugin-sdk/provider-auth-api-key` | `upsertApiKeyProfile` gibi API anahtarı onboarding/profil yazma yardımcıları |
    | `plugin-sdk/provider-auth-result` | Standart OAuth auth-result oluşturucusu |
    | `plugin-sdk/provider-auth-login` | Sağlayıcı plugin'leri için paylaşılan etkileşimli oturum açma yardımcıları |
    | `plugin-sdk/provider-env-vars` | Sağlayıcı auth env değişkeni arama yardımcıları |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, paylaşılan replay-policy oluşturucuları, sağlayıcı uç nokta yardımcıları ve `normalizeNativeXaiModelId` gibi model kimliği normalleştirme yardımcıları |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Genel sağlayıcı HTTP/uç nokta yeteneği yardımcıları |
    | `plugin-sdk/provider-web-fetch-contract` | `enablePluginInConfig` ve `WebFetchProviderPlugin` gibi dar kapsamlı web getirme yapılandırma/seçim sözleşmesi yardımcıları |
    | `plugin-sdk/provider-web-fetch` | Web getirme sağlayıcı kayıt/önbellek yardımcıları |
    | `plugin-sdk/provider-web-search-config-contract` | Plugin etkinleştirme bağlantısına ihtiyaç duymayan sağlayıcılar için dar kapsamlı web araması yapılandırma/kimlik bilgisi yardımcıları |
    | `plugin-sdk/provider-web-search-contract` | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` ve kapsamlı kimlik bilgisi ayarlayıcıları/alıcıları gibi dar kapsamlı web araması yapılandırma/kimlik bilgisi sözleşmesi yardımcıları |
    | `plugin-sdk/provider-web-search` | Web araması sağlayıcı kayıt/önbellek/çalışma zamanı yardımcıları |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini şema temizleme + tanılama ve `resolveXaiModelCompatPatch` / `applyXaiModelCompat` gibi xAI uyumluluk yardımcıları |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` ve benzerleri |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, akış sarmalayıcı türleri ve paylaşılan Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot sarmalayıcı yardımcıları |
    | `plugin-sdk/provider-onboard` | Onboarding yapılandırma yama yardımcıları |
    | `plugin-sdk/global-singleton` | Süreç yerel singleton/eşleme/önbellek yardımcıları |
  </Accordion>

  <Accordion title="Auth ve güvenlik alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, komut kayıt defteri yardımcıları, gönderici yetkilendirme yardımcıları |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated` ve `buildHelpMessage` gibi komut/yardım mesajı oluşturucuları |
    | `plugin-sdk/approval-auth-runtime` | Onaylayıcı çözümleme ve aynı sohbet eylem auth yardımcıları |
    | `plugin-sdk/approval-client-runtime` | Yerel exec onay profili/filtre yardımcıları |
    | `plugin-sdk/approval-delivery-runtime` | Yerel onay yeteneği/teslim bağdaştırıcıları |
    | `plugin-sdk/approval-gateway-runtime` | Paylaşılan onay gateway çözümleme yardımcısı |
    | `plugin-sdk/approval-handler-adapter-runtime` | Yoğun kanal giriş noktaları için hafif yerel onay bağdaştırıcısı yükleme yardımcıları |
    | `plugin-sdk/approval-handler-runtime` | Daha geniş onay işleyici çalışma zamanı yardımcıları; dar bağdaştırıcı/gateway yüzeyleri yeterliyse onları tercih edin |
    | `plugin-sdk/approval-native-runtime` | Yerel onay hedefi + hesap bağlama yardımcıları |
    | `plugin-sdk/approval-reply-runtime` | Exec/plugin onay yanıt yükü yardımcıları |
    | `plugin-sdk/command-auth-native` | Yerel komut auth + yerel oturum hedefi yardımcıları |
    | `plugin-sdk/command-detection` | Paylaşılan komut algılama yardımcıları |
    | `plugin-sdk/command-surface` | Komut gövdesi normalleştirme ve komut yüzeyi yardımcıları |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Kanal/plugin gizli yüzeyleri için dar kapsamlı gizli sözleşme toplama yardımcıları |
    | `plugin-sdk/secret-ref-runtime` | Gizli sözleşme/yapılandırma ayrıştırması için dar kapsamlı `coerceSecretRef` ve SecretRef türlendirme yardımcıları |
    | `plugin-sdk/security-runtime` | Paylaşılan güven, DM geçitleme, dış içerik ve gizli toplama yardımcıları |
    | `plugin-sdk/ssrf-policy` | Ana makine allowlist ve özel ağ SSRF ilkesi yardımcıları |
    | `plugin-sdk/ssrf-runtime` | Sabitlenmiş dağıtıcı, SSRF korumalı fetch ve SSRF ilkesi yardımcıları |
    | `plugin-sdk/secret-input` | Gizli girdi ayrıştırma yardımcıları |
    | `plugin-sdk/webhook-ingress` | Webhook istek/hedef yardımcıları |
    | `plugin-sdk/webhook-request-guards` | İstek gövdesi boyutu/zaman aşımı yardımcıları |
  </Accordion>

  <Accordion title="Çalışma zamanı ve depolama alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/runtime` | Geniş çalışma zamanı/günlükleme/yedekleme/plugin kurulum yardımcıları |
    | `plugin-sdk/runtime-env` | Dar kapsamlı çalışma zamanı env, logger, zaman aşımı, yeniden deneme ve backoff yardımcıları |
    | `plugin-sdk/channel-runtime-context` | Genel kanal çalışma zamanı bağlamı kayıt ve arama yardımcıları |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Paylaşılan plugin komutu/hook/http/etkileşimli yardımcıları |
    | `plugin-sdk/hook-runtime` | Paylaşılan webhook/dahili hook ardışık düzeni yardımcıları |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`, `createLazyRuntimeMethod` ve `createLazyRuntimeSurface` gibi tembel çalışma zamanı içe aktarma/bağlama yardımcıları |
    | `plugin-sdk/process-runtime` | Süreç exec yardımcıları |
    | `plugin-sdk/cli-runtime` | CLI biçimlendirme, bekleme ve sürüm yardımcıları |
    | `plugin-sdk/gateway-runtime` | Gateway istemcisi ve kanal durumu yama yardımcıları |
    | `plugin-sdk/config-runtime` | Yapılandırma yükleme/yazma yardımcıları |
    | `plugin-sdk/telegram-command-config` | Paketlenmiş Telegram sözleşme yüzeyi mevcut olmadığında bile Telegram komut adı/açıklaması normalleştirme ve yinelenen/çakışma kontrolleri |
    | `plugin-sdk/approval-runtime` | Exec/plugin onay yardımcıları, onay-yeteneği oluşturucuları, auth/profil yardımcıları, yerel yönlendirme/çalışma zamanı yardımcıları |
    | `plugin-sdk/reply-runtime` | Paylaşılan gelen/yanıt çalışma zamanı yardımcıları, parçalama, dağıtım, heartbeat, yanıt planlayıcı |
    | `plugin-sdk/reply-dispatch-runtime` | Dar kapsamlı yanıt dağıtma/sonlandırma yardımcıları |
    | `plugin-sdk/reply-history` | `buildHistoryContext`, `recordPendingHistoryEntry` ve `clearHistoryEntriesIfEnabled` gibi paylaşılan kısa pencere yanıt geçmişi yardımcıları |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Dar kapsamlı metin/Markdown parçalama yardımcıları |
    | `plugin-sdk/session-store-runtime` | Oturum deposu yolu + updated-at yardımcıları |
    | `plugin-sdk/state-paths` | Durum/OAuth dizin yolu yardımcıları |
    | `plugin-sdk/routing` | `resolveAgentRoute`, `buildAgentSessionKey` ve `resolveDefaultAgentBoundAccountId` gibi yönlendirme/oturum anahtarı/hesap bağlama yardımcıları |
    | `plugin-sdk/status-helpers` | Paylaşılan kanal/hesap durum özeti yardımcıları, çalışma zamanı durumu varsayılanları ve sorun meta verisi yardımcıları |
    | `plugin-sdk/target-resolver-runtime` | Paylaşılan hedef çözümleyici yardımcıları |
    | `plugin-sdk/string-normalization-runtime` | Slug/dize normalleştirme yardımcıları |
    | `plugin-sdk/request-url` | Fetch/istek benzeri girdilerden dize URL çıkarma |
    | `plugin-sdk/run-command` | Normalize edilmiş stdout/stderr sonuçlarıyla zamanlanmış komut çalıştırıcısı |
    | `plugin-sdk/param-readers` | Yaygın araç/CLI parametre okuyucuları |
    | `plugin-sdk/tool-payload` | Araç sonuç nesnelerinden normalize edilmiş yükleri çıkarma |
    | `plugin-sdk/tool-send` | Araç argümanlarından kanonik gönderim hedefi alanlarını çıkarma |
    | `plugin-sdk/temp-path` | Paylaşılan geçici indirme yolu yardımcıları |
    | `plugin-sdk/logging-core` | Alt sistem logger ve redaksiyon yardımcıları |
    | `plugin-sdk/markdown-table-runtime` | Markdown tablo modu yardımcıları |
    | `plugin-sdk/json-store` | Küçük JSON durum okuma/yazma yardımcıları |
    | `plugin-sdk/file-lock` | Yeniden girişli dosya kilidi yardımcıları |
    | `plugin-sdk/persistent-dedupe` | Disk destekli tekilleştirme önbelleği yardımcıları |
    | `plugin-sdk/acp-runtime` | ACP çalışma zamanı/oturum ve yanıt-dağıtım yardımcıları |
    | `plugin-sdk/agent-config-primitives` | Dar kapsamlı agent çalışma zamanı yapılandırma şeması ilkel öğeleri |
    | `plugin-sdk/boolean-param` | Gevşek boolean param okuyucusu |
    | `plugin-sdk/dangerous-name-runtime` | Tehlikeli ad eşleştirme çözümleme yardımcıları |
    | `plugin-sdk/device-bootstrap` | Cihaz önyükleme ve eşleme belirteci yardımcıları |
    | `plugin-sdk/extension-shared` | Paylaşılan pasif kanal, durum ve ortam proxy yardımcısı ilkel öğeleri |
    | `plugin-sdk/models-provider-runtime` | `/models` komutu/sağlayıcı yanıt yardımcıları |
    | `plugin-sdk/skill-commands-runtime` | Skill komutu listeleme yardımcıları |
    | `plugin-sdk/native-command-registry` | Yerel komut kayıt defteri/oluşturma/serileştirme yardımcıları |
    | `plugin-sdk/provider-zai-endpoint` | Z.A.I uç nokta algılama yardımcıları |
    | `plugin-sdk/infra-runtime` | Sistem olayı/heartbeat yardımcıları |
    | `plugin-sdk/collection-runtime` | Küçük sınırlı önbellek yardımcıları |
    | `plugin-sdk/diagnostic-runtime` | Tanılama bayrağı ve olay yardımcıları |
    | `plugin-sdk/error-runtime` | Hata grafiği, biçimlendirme, paylaşılan hata sınıflandırma yardımcıları, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Sarmalanmış fetch, proxy ve sabitlenmiş arama yardımcıları |
    | `plugin-sdk/host-runtime` | Ana makine adı ve SCP ana makine normalleştirme yardımcıları |
    | `plugin-sdk/retry-runtime` | Yeniden deneme yapılandırması ve yeniden deneme çalıştırıcısı yardımcıları |
    | `plugin-sdk/agent-runtime` | Agent dizini/kimliği/çalışma alanı yardımcıları |
    | `plugin-sdk/directory-runtime` | Yapılandırma destekli dizin sorgulama/tekilleştirme |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Yetenek ve test alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Paylaşılan medya fetch/dönüştürme/depolama yardımcıları ile medya yükü oluşturucuları |
    | `plugin-sdk/media-generation-runtime` | Paylaşılan medya üretimi failover yardımcıları, aday seçimi ve eksik model mesajları |
    | `plugin-sdk/media-understanding` | Medya anlama sağlayıcı türleri ve sağlayıcıya dönük görsel/ses yardımcı dışa aktarımları |
    | `plugin-sdk/text-runtime` | Assistant görünür metin temizleme, markdown işleme/parçalama/tablo yardımcıları, redaksiyon yardımcıları, yönerge etiketi yardımcıları ve güvenli metin araçları gibi paylaşılan metin/Markdown/günlükleme yardımcıları |
    | `plugin-sdk/text-chunking` | Giden metin parçalama yardımcısı |
    | `plugin-sdk/speech` | Konuşma sağlayıcı türleri ile sağlayıcıya dönük yönerge, kayıt defteri ve doğrulama yardımcıları |
    | `plugin-sdk/speech-core` | Paylaşılan konuşma sağlayıcı türleri, kayıt defteri, yönerge ve normalleştirme yardımcıları |
    | `plugin-sdk/realtime-transcription` | Gerçek zamanlı transkripsiyon sağlayıcı türleri ve kayıt yardımcıları |
    | `plugin-sdk/realtime-voice` | Gerçek zamanlı ses sağlayıcı türleri ve kayıt yardımcıları |
    | `plugin-sdk/image-generation` | Görsel üretimi sağlayıcı türleri |
    | `plugin-sdk/image-generation-core` | Paylaşılan görsel üretimi türleri, failover, auth ve kayıt yardımcıları |
    | `plugin-sdk/music-generation` | Müzik üretimi sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/music-generation-core` | Paylaşılan müzik üretimi türleri, failover yardımcıları, sağlayıcı arama ve model başvurusu ayrıştırma |
    | `plugin-sdk/video-generation` | Video üretimi sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/video-generation-core` | Paylaşılan video üretimi türleri, failover yardımcıları, sağlayıcı arama ve model başvurusu ayrıştırma |
    | `plugin-sdk/webhook-targets` | Webhook hedef kayıt defteri ve route-install yardımcıları |
    | `plugin-sdk/webhook-path` | Webhook yolu normalleştirme yardımcıları |
    | `plugin-sdk/web-media` | Paylaşılan uzak/yerel medya yükleme yardımcıları |
    | `plugin-sdk/zod` | Plugin SDK tüketicileri için yeniden dışa aktarılan `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Memory alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/memory-core` | Yönetici/yapılandırma/dosya/CLI yardımcıları için paketlenmiş memory-core yardımcı yüzeyi |
    | `plugin-sdk/memory-core-engine-runtime` | Memory dizin/arama çalışma zamanı cephesi |
    | `plugin-sdk/memory-core-host-engine-foundation` | Memory ana makine foundation engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Memory ana makine embedding engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-qmd` | Memory ana makine QMD engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-storage` | Memory ana makine depolama engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-multimodal` | Memory ana makine multimodal yardımcıları |
    | `plugin-sdk/memory-core-host-query` | Memory ana makine sorgu yardımcıları |
    | `plugin-sdk/memory-core-host-secret` | Memory ana makine gizli yardımcıları |
    | `plugin-sdk/memory-core-host-events` | Memory ana makine olay günlüğü yardımcıları |
    | `plugin-sdk/memory-core-host-status` | Memory ana makine durum yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-cli` | Memory ana makine CLI çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-core` | Memory ana makine çekirdek çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-files` | Memory ana makine dosya/çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-host-core` | Memory ana makine çekirdek çalışma zamanı yardımcıları için sağlayıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-events` | Memory ana makine olay günlüğü yardımcıları için sağlayıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-files` | Memory ana makine dosya/çalışma zamanı yardımcıları için sağlayıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-markdown` | Memory'ye bitişik plugin'ler için paylaşılan yönetilen markdown yardımcıları |
    | `plugin-sdk/memory-host-search` | Search-manager erişimi için etkin memory çalışma zamanı cephesi |
    | `plugin-sdk/memory-host-status` | Memory ana makine durum yardımcıları için sağlayıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-lancedb` | Paketlenmiş memory-lancedb yardımcı yüzeyi |
  </Accordion>

  <Accordion title="Ayrılmış paketlenmiş yardımcı alt yollar">
    | Aile | Güncel alt yollar | Amaçlanan kullanım |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Paketlenmiş browser plugin destek yardımcıları (`browser-support` uyumluluk barrel'ı olarak kalır) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Paketlenmiş Matrix yardımcı/çalışma zamanı yüzeyi |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Paketlenmiş LINE yardımcı/çalışma zamanı yüzeyi |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Paketlenmiş IRC yardımcı yüzeyi |
    | Kanala özgü yardımcılar | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Paketlenmiş kanal uyumluluk/yardımcı yüzeyleri |
    | Auth/plugin'e özgü yardımcılar | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Paketlenmiş özellik/plugin yardımcı yüzeyleri; `plugin-sdk/github-copilot-token` şu anda `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` ve `resolveCopilotApiToken` dışa aktarımlarını içerir |
  </Accordion>
</AccordionGroup>

## Kayıt API'si

`register(api)` geri çağırması, şu yöntemlere sahip bir `OpenClawPluginApi`
nesnesi alır:

### Yetenek kaydı

| Yöntem                                           | Kaydettiği şey                 |
| ------------------------------------------------ | ------------------------------ |
| `api.registerProvider(...)`                      | Metin çıkarımı (LLM)           |
| `api.registerCliBackend(...)`                    | Yerel CLI çıkarım arka ucu     |
| `api.registerChannel(...)`                       | Mesajlaşma kanalı              |
| `api.registerSpeechProvider(...)`                | Metinden konuşmaya / STT sentezi |
| `api.registerRealtimeTranscriptionProvider(...)` | Akışlı gerçek zamanlı transkripsiyon |
| `api.registerRealtimeVoiceProvider(...)`         | Çift yönlü gerçek zamanlı ses oturumları |
| `api.registerMediaUnderstandingProvider(...)`    | Görsel/ses/video analizi       |
| `api.registerImageGenerationProvider(...)`       | Görsel üretimi                 |
| `api.registerMusicGenerationProvider(...)`       | Müzik üretimi                  |
| `api.registerVideoGenerationProvider(...)`       | Video üretimi                  |
| `api.registerWebFetchProvider(...)`              | Web getirme / scrape sağlayıcısı |
| `api.registerWebSearchProvider(...)`             | Web araması                    |

### Araçlar ve komutlar

| Yöntem                          | Kaydettiği şey                              |
| ------------------------------- | ------------------------------------------- |
| `api.registerTool(tool, opts?)` | Agent aracı (zorunlu veya `{ optional: true }`) |
| `api.registerCommand(def)`      | Özel komut (LLM'yi atlar)                   |

### Altyapı

| Yöntem                                         | Kaydettiği şey                    |
| ---------------------------------------------- | --------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Olay hook'u                       |
| `api.registerHttpRoute(params)`                | Gateway HTTP uç noktası           |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC yöntemi               |
| `api.registerCli(registrar, opts?)`            | CLI alt komutu                    |
| `api.registerService(service)`                 | Arka plan hizmeti                 |
| `api.registerInteractiveHandler(registration)` | Etkileşimli işleyici              |
| `api.registerMemoryPromptSupplement(builder)`  | Eklemeli memory'ye bitişik prompt bölümü |
| `api.registerMemoryCorpusSupplement(adapter)`  | Eklemeli memory arama/okuma derlemi |

Ayrılmış çekirdek yönetici ad alanları (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`), bir plugin daha dar bir gateway yöntem kapsamı atamaya çalışsa bile
her zaman `operator.admin` olarak kalır. Plugin'e ait yöntemler için plugin'e özgü
önekleri tercih edin.

### CLI kayıt meta verileri

`api.registerCli(registrar, opts?)` iki tür üst düzey meta veri kabul eder:

- `commands`: kayıtçıya ait açık komut kökleri
- `descriptors`: kök CLI yardımı, yönlendirme ve tembel plugin CLI kaydı için
  ayrıştırma zamanında kullanılan komut tanımlayıcıları

Bir plugin komutunun normal kök CLI yolunda tembel yüklenmiş kalmasını
istiyorsanız, o kayıtçının açığa çıkardığı her üst düzey komut kökünü kapsayan
`descriptors` sağlayın.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix hesaplarını, doğrulamayı, cihazları ve profil durumunu yönet",
        hasSubcommands: true,
      },
    ],
  },
);
```

`commands` alanını tek başına yalnızca tembel kök CLI kaydına ihtiyacınız
olmadığında kullanın. Bu hevesli uyumluluk yolu desteklenmeye devam eder, ancak
ayrıştırma zamanında tembel yükleme için tanımlayıcı destekli yer tutucular
kurmaz.

### CLI arka uç kaydı

`api.registerCliBackend(...)`, bir plugin'in `codex-cli` gibi yerel bir
AI CLI arka ucu için varsayılan yapılandırmaya sahip olmasına izin verir.

- Arka uç `id` değeri, `codex-cli/gpt-5` gibi model başvurularında sağlayıcı öneki olur.
- Arka uç `config` değeri, `agents.defaults.cliBackends.<id>` ile aynı biçimi kullanır.
- Kullanıcı yapılandırması yine önceliklidir. OpenClaw, CLI'yi çalıştırmadan önce
  `agents.defaults.cliBackends.<id>` değerini plugin varsayılanı üzerine birleştirir.
- Bir arka uç birleştirmeden sonra uyumluluk yeniden yazımları gerektiriyorsa
  (örneğin eski bayrak biçimlerini normalleştirmek) `normalizeConfig` kullanın.

### Özel slot'lar

| Yöntem                                     | Kaydettiği şey                                                                                                                                          |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Bağlam engine'i (aynı anda yalnızca biri etkin). `assemble()` geri çağırması, engine'in prompt eklemelerini uyarlayabilmesi için `availableTools` ve `citationsMode` alır. |
| `api.registerMemoryCapability(capability)` | Birleşik memory yeteneği                                                                                                                                 |
| `api.registerMemoryPromptSection(builder)` | Memory prompt bölümü oluşturucusu                                                                                                                        |
| `api.registerMemoryFlushPlan(resolver)`    | Memory flush planı çözümleyicisi                                                                                                                         |
| `api.registerMemoryRuntime(runtime)`       | Memory çalışma zamanı bağdaştırıcısı                                                                                                                     |

### Memory embedding bağdaştırıcıları

| Yöntem                                         | Kaydettiği şey                                |
| ---------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Etkin plugin için memory embedding bağdaştırıcısı |

- `registerMemoryCapability`, tercih edilen özel memory-plugin API'sidir.
- `registerMemoryCapability`, eşlik eden plugin'lerin dışa aktarılmış memory yapılarını
  belirli bir memory plugin'inin özel düzenine ulaşmadan
  `openclaw/plugin-sdk/memory-host-core` üzerinden tüketebilmesi için
  `publicArtifacts.listArtifacts(...)` işlevini de açığa çıkarabilir.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` ve
  `registerMemoryRuntime`, eski uyumluluğu koruyan özel memory-plugin API'leridir.
- `registerMemoryEmbeddingProvider`, etkin memory plugin'inin bir veya daha fazla
  embedding bağdaştırıcısı kimliği kaydetmesine izin verir
  (örneğin `openai`, `gemini` veya plugin tarafından tanımlanmış özel bir kimlik).
- `agents.defaults.memorySearch.provider` ve
  `agents.defaults.memorySearch.fallback` gibi kullanıcı yapılandırmaları,
  bu kaydedilmiş bağdaştırıcı kimliklerine göre çözümlenir.

### Olaylar ve yaşam döngüsü

| Yöntem                                       | Ne yapar                    |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Türlendirilmiş yaşam döngüsü hook'u |
| `api.onConversationBindingResolved(handler)` | Konuşma bağlama geri çağırması |

### Hook karar semantiği

- `before_tool_call`: `{ block: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `before_tool_call`: `{ block: false }` döndürmek, geçersiz kılma olarak değil, karar yokmuş gibi değerlendirilir (`block` alanını atlamakla aynıdır).
- `before_install`: `{ block: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `before_install`: `{ block: false }` döndürmek, geçersiz kılma olarak değil, karar yokmuş gibi değerlendirilir (`block` alanını atlamakla aynıdır).
- `reply_dispatch`: `{ handled: true, ... }` döndürmek kesindir. Herhangi bir işleyici dağıtımı sahiplendiğinde daha düşük öncelikli işleyiciler ve varsayılan model dağıtım yolu atlanır.
- `message_sending`: `{ cancel: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `message_sending`: `{ cancel: false }` döndürmek, geçersiz kılma olarak değil, karar yokmuş gibi değerlendirilir (`cancel` alanını atlamakla aynıdır).

### API nesnesi alanları

| Alan                    | Tür                       | Açıklama                                                                                     |
| ----------------------- | ------------------------- | -------------------------------------------------------------------------------------------- |
| `api.id`                | `string`                  | Plugin kimliği                                                                               |
| `api.name`              | `string`                  | Görünen ad                                                                                   |
| `api.version`           | `string?`                 | Plugin sürümü (isteğe bağlı)                                                                 |
| `api.description`       | `string?`                 | Plugin açıklaması (isteğe bağlı)                                                             |
| `api.source`            | `string`                  | Plugin kaynak yolu                                                                           |
| `api.rootDir`           | `string?`                 | Plugin kök dizini (isteğe bağlı)                                                             |
| `api.config`            | `OpenClawConfig`          | Geçerli yapılandırma anlık görüntüsü (mevcut olduğunda etkin bellek içi çalışma zamanı anlık görüntüsü) |
| `api.pluginConfig`      | `Record<string, unknown>` | `plugins.entries.<id>.config` içinden plugin'e özgü yapılandırma                            |
| `api.runtime`           | `PluginRuntime`           | [Çalışma zamanı yardımcıları](/tr/plugins/sdk-runtime)                                          |
| `api.logger`            | `PluginLogger`            | Kapsamlı logger (`debug`, `info`, `warn`, `error`)                                           |
| `api.registrationMode`  | `PluginRegistrationMode`  | Geçerli yükleme modu; `"setup-runtime"` hafif tam giriş öncesi başlangıç/kurulum penceresidir |
| `api.resolvePath(input)` | `(string) => string`     | Plugin köküne göre göreli yolu çözümler                                                      |

## Dahili modül kuralı

Plugin'iniz içinde, dahili içe aktarımlar için yerel barrel dosyaları kullanın:

```
my-plugin/
  api.ts            # Harici tüketiciler için herkese açık dışa aktarımlar
  runtime-api.ts    # Yalnızca dahili çalışma zamanı dışa aktarımları
  index.ts          # Plugin giriş noktası
  setup-entry.ts    # Hafif yalnızca kurulum amaçlı giriş (isteğe bağlı)
```

<Warning>
  Üretim kodunda kendi plugin'inizi asla `openclaw/plugin-sdk/<sizin-plugininiz>`
  üzerinden içe aktarmayın. Dahili içe aktarımları `./api.ts` veya
  `./runtime-api.ts` üzerinden yönlendirin. SDK yolu yalnızca harici sözleşmedir.
</Warning>

Cephe üzerinden yüklenen paketlenmiş plugin herkese açık yüzeyleri (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` ve benzer herkese açık giriş dosyaları), OpenClaw zaten
çalışıyorsa artık etkin çalışma zamanı yapılandırma anlık görüntüsünü tercih eder.
Henüz çalışma zamanı anlık görüntüsü yoksa diskte çözümlenmiş yapılandırma dosyasına geri dönerler.

Sağlayıcı plugin'leri ayrıca, bir yardımcı bilinçli olarak sağlayıcıya özgüyse ve henüz
genel bir SDK alt yoluna ait değilse, dar kapsamlı plugin'e özgü bir sözleşme barrel'ı
da açığa çıkarabilir. Güncel paketlenmiş örnek: Anthropic sağlayıcısı, Anthropic beta-header
ve `service_tier` mantığını genel bir `plugin-sdk/*` sözleşmesine yükseltmek yerine
Claude akış yardımcılarını kendi herkese açık `api.ts` / `contract-api.ts`
yüzeyinde tutar.

Diğer güncel paketlenmiş örnekler:

- `@openclaw/openai-provider`: `api.ts`, sağlayıcı oluşturucuları,
  varsayılan model yardımcıları ve gerçek zamanlı sağlayıcı oluşturucularını dışa aktarır
- `@openclaw/openrouter-provider`: `api.ts`, sağlayıcı oluşturucusunu ve ayrıca
  onboarding/yapılandırma yardımcılarını dışa aktarır

<Warning>
  Uzantı üretim kodu ayrıca `openclaw/plugin-sdk/<başka-plugin>`
  içe aktarımlarından da kaçınmalıdır. Bir yardımcı gerçekten paylaşılıyorsa,
  iki plugin'i birbirine bağlamak yerine bunu `openclaw/plugin-sdk/speech`,
  `.../provider-model-shared` veya başka bir yetenek odaklı yüzey gibi tarafsız bir SDK alt yoluna yükseltin.
</Warning>

## İlgili

- [Giriş Noktaları](/tr/plugins/sdk-entrypoints) — `definePluginEntry` ve `defineChannelPluginEntry` seçenekleri
- [Çalışma Zamanı Yardımcıları](/tr/plugins/sdk-runtime) — tam `api.runtime` ad alanı başvurusu
- [Kurulum ve Yapılandırma](/tr/plugins/sdk-setup) — paketleme, manifestler, yapılandırma şemaları
- [Test](/tr/plugins/sdk-testing) — test yardımcıları ve lint kuralları
- [SDK Geçişi](/tr/plugins/sdk-migration) — kullanımdan kaldırılmış yüzeylerden geçiş
- [Plugin İç Yapısı](/tr/plugins/architecture) — derin mimari ve yetenek modeli
