---
read_when:
    - Hangi SDK alt yolundan içe aktarma yapmanız gerektiğini bilmeniz gerekiyor
    - OpenClawPluginApi üzerindeki tüm kayıt yöntemleri için bir başvuru istiyorsunuz
    - Belirli bir SDK dışa aktarımını arıyorsunuz
sidebarTitle: SDK Overview
summary: İçe aktarma eşleme tablosu, kayıt API başvurusu ve SDK mimarisi
title: Plugin SDK Genel Bakış
x-i18n:
    generated_at: "2026-04-08T08:44:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e7b420eb0f3faa8916357d52df949f6c9a46f1c843a1e6a0c0b8bb26db6cbff
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK Genel Bakış

Plugin SDK, plugin'ler ile çekirdek arasındaki tiplenmiş sözleşmedir. Bu sayfa,
**neyin içe aktarılacağı** ve **nelerin kaydedilebileceği** için başvuru
kaynağıdır.

<Tip>
  **Nasıl yapılır kılavuzu mu arıyorsunuz?**
  - İlk plugin mi? [Başlangıç](/tr/plugins/building-plugins) ile başlayın
  - Kanal plugin'i mi? [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins) sayfasına bakın
  - Sağlayıcı plugin'i mi? [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) sayfasına bakın
</Tip>

## İçe aktarma kuralı

Her zaman belirli bir alt yoldan içe aktarın:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Her alt yol küçük, kendi içinde tamamlanmış bir modüldür. Bu, başlangıcın hızlı
kalmasını sağlar ve dairesel bağımlılık sorunlarını önler. Kanala özel
entry/build yardımcıları için `openclaw/plugin-sdk/channel-core` tercih edin;
daha geniş şemsiye yüzey ve `buildChannelConfigSchema` gibi paylaşılan
yardımcılar için `openclaw/plugin-sdk/core` kullanın.

`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` veya
kanal markalı yardımcı yüzeyler gibi sağlayıcı adlı kolaylık katmanları
eklemeyin ya da bunlara bağımlı olmayın. Paketlenmiş plugin'ler, kendi `api.ts`
veya `runtime-api.ts` barrel dosyaları içinde genel SDK alt yollarını
birleştirmelidir ve çekirdek, ihtiyaç gerçekten kanallar arası olduğunda ya bu
plugin-yerel barrel dosyalarını kullanmalı ya da dar bir genel SDK sözleşmesi
eklemelidir.

Oluşturulmuş dışa aktarma eşleme tablosu hâlâ `plugin-sdk/feishu`,
`plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup` ve
`plugin-sdk/matrix*` gibi küçük bir paketlenmiş-plugin yardımcı yüzeyi kümesi
içerir. Bu alt yollar yalnızca paketlenmiş-plugin bakımı ve uyumluluk için
vardır; aşağıdaki ortak tabloda bilerek atlanmıştır ve yeni üçüncü taraf
plugin'leri için önerilen içe aktarma yolu değildir.

## Alt yol başvurusu

Amaçlarına göre gruplandırılmış, en sık kullanılan alt yollar. Oluşturulmuş tam
200+ alt yol listesi `scripts/lib/plugin-sdk-entrypoints.json` içinde bulunur.

Ayrılmış paketlenmiş-plugin yardımcı alt yolları bu oluşturulmuş listede yine
de görünür. Bir dokümantasyon sayfası bunlardan birini açıkça herkese açık
olarak tanıtmadıkça, bunları uygulama ayrıntısı/uyumluluk yüzeyleri olarak
değerlendirin.

### Plugin entry

| Alt yol                     | Temel dışa aktarımlar                                                                                                                 |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Kanal alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Kök `openclaw.json` Zod şema dışa aktarımı (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, ayrıca `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Paylaşılan kurulum sihirbazı yardımcıları, izin listesi istemleri, kurulum durumu oluşturucuları |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Çoklu hesap yapılandırma/eylem geçidi yardımcıları, varsayılan hesap geri dönüş yardımcıları |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, hesap kimliği normalizasyon yardımcıları |
    | `plugin-sdk/account-resolution` | Hesap arama + varsayılan geri dönüş yardımcıları |
    | `plugin-sdk/account-helpers` | Dar hesap listesi/hesap eylemi yardımcıları |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Kanal yapılandırma şema türleri |
    | `plugin-sdk/telegram-command-config` | Paketlenmiş sözleşme geri dönüşü ile Telegram özel komut normalizasyonu/doğrulama yardımcıları |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Paylaşılan gelen rota + zarf oluşturucu yardımcıları |
    | `plugin-sdk/inbound-reply-dispatch` | Paylaşılan gelen kayıt ve dağıtım yardımcıları |
    | `plugin-sdk/messaging-targets` | Hedef ayrıştırma/eşleme yardımcıları |
    | `plugin-sdk/outbound-media` | Paylaşılan giden medya yükleme yardımcıları |
    | `plugin-sdk/outbound-runtime` | Giden kimlik/gönderim delege yardımcıları |
    | `plugin-sdk/thread-bindings-runtime` | İş parçacığı bağlama yaşam döngüsü ve bağdaştırıcı yardımcıları |
    | `plugin-sdk/agent-media-payload` | Eski agent medya payload oluşturucusu |
    | `plugin-sdk/conversation-runtime` | Konuşma/iş parçacığı bağlama, eşleştirme ve yapılandırılmış bağlama yardımcıları |
    | `plugin-sdk/runtime-config-snapshot` | Çalışma zamanı yapılandırma anlık görüntü yardımcısı |
    | `plugin-sdk/runtime-group-policy` | Çalışma zamanı grup ilkesi çözümleme yardımcıları |
    | `plugin-sdk/channel-status` | Paylaşılan kanal durum anlık görüntüsü/özet yardımcıları |
    | `plugin-sdk/channel-config-primitives` | Dar kanal yapılandırma-şeması ilkelleri |
    | `plugin-sdk/channel-config-writes` | Kanal yapılandırma yazma yetkilendirme yardımcıları |
    | `plugin-sdk/channel-plugin-common` | Paylaşılan kanal plugin prelude dışa aktarımları |
    | `plugin-sdk/allowlist-config-edit` | İzin listesi yapılandırma düzenleme/okuma yardımcıları |
    | `plugin-sdk/group-access` | Paylaşılan grup erişim kararı yardımcıları |
    | `plugin-sdk/direct-dm` | Paylaşılan doğrudan DM kimlik doğrulama/koruma yardımcıları |
    | `plugin-sdk/interactive-runtime` | Etkileşimli yanıt payload normalizasyonu/indirgeme yardımcıları |
    | `plugin-sdk/channel-inbound` | Gelen debounce, mention eşleme, mention ilkesi yardımcıları ve zarf yardımcıları |
    | `plugin-sdk/channel-send-result` | Yanıt sonuç türleri |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Hedef ayrıştırma/eşleme yardımcıları |
    | `plugin-sdk/channel-contract` | Kanal sözleşme türleri |
    | `plugin-sdk/channel-feedback` | Geri bildirim/reaksiyon bağlantısı |
    | `plugin-sdk/channel-secret-runtime` | `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` ve secret hedef türleri gibi dar secret-sözleşme yardımcıları |
  </Accordion>

  <Accordion title="Sağlayıcı alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Düzenlenmiş yerel/self-hosted sağlayıcı kurulum yardımcıları |
    | `plugin-sdk/self-hosted-provider-setup` | Odaklanmış OpenAI uyumlu self-hosted sağlayıcı kurulum yardımcıları |
    | `plugin-sdk/cli-backend` | CLI arka uç varsayılanları + watchdog sabitleri |
    | `plugin-sdk/provider-auth-runtime` | Sağlayıcı plugin'leri için çalışma zamanı API anahtarı çözümleme yardımcıları |
    | `plugin-sdk/provider-auth-api-key` | `upsertApiKeyProfile` gibi API anahtarı onboarding/profil yazma yardımcıları |
    | `plugin-sdk/provider-auth-result` | Standart OAuth kimlik doğrulama sonuç oluşturucusu |
    | `plugin-sdk/provider-auth-login` | Sağlayıcı plugin'leri için paylaşılan etkileşimli oturum açma yardımcıları |
    | `plugin-sdk/provider-env-vars` | Sağlayıcı kimlik doğrulama ortam değişkeni arama yardımcıları |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, paylaşılan replay-policy oluşturucuları, sağlayıcı uç nokta yardımcıları ve `normalizeNativeXaiModelId` gibi model-kimliği normalizasyon yardımcıları |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Genel sağlayıcı HTTP/uç nokta yetenek yardımcıları |
    | `plugin-sdk/provider-web-fetch-contract` | `enablePluginInConfig` ve `WebFetchProviderPlugin` gibi dar web-fetch yapılandırma/seçim sözleşmesi yardımcıları |
    | `plugin-sdk/provider-web-fetch` | Web-fetch sağlayıcı kaydı/önbellek yardımcıları |
    | `plugin-sdk/provider-web-search-config-contract` | Plugin-etkinleştirme bağlantısına ihtiyaç duymayan sağlayıcılar için dar web-search yapılandırma/kimlik bilgisi yardımcıları |
    | `plugin-sdk/provider-web-search-contract` | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` ve kapsamlı kimlik bilgisi ayarlayıcıları/alıcıları gibi dar web-search yapılandırma/kimlik bilgisi sözleşmesi yardımcıları |
    | `plugin-sdk/provider-web-search` | Web-search sağlayıcı kaydı/önbellek/çalışma zamanı yardımcıları |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini şema temizliği + tanılama ve `resolveXaiModelCompatPatch` / `applyXaiModelCompat` gibi xAI uyumluluk yardımcıları |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` ve benzerleri |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, akış sarmalayıcı türleri ve paylaşılan Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot sarmalayıcı yardımcıları |
    | `plugin-sdk/provider-onboard` | Onboarding yapılandırma yama yardımcıları |
    | `plugin-sdk/global-singleton` | Süreç-yerel singleton/map/önbellek yardımcıları |
  </Accordion>

  <Accordion title="Kimlik doğrulama ve güvenlik alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, komut kayıt yardımcıları, gönderen yetkilendirme yardımcıları |
    | `plugin-sdk/approval-auth-runtime` | Onaylayıcı çözümleme ve aynı sohbet eylem-kimlik doğrulama yardımcıları |
    | `plugin-sdk/approval-client-runtime` | Yerel exec onay profili/filtre yardımcıları |
    | `plugin-sdk/approval-delivery-runtime` | Yerel onay yeteneği/teslim bağdaştırıcıları |
    | `plugin-sdk/approval-gateway-runtime` | Paylaşılan onay gateway çözümleme yardımcısı |
    | `plugin-sdk/approval-handler-adapter-runtime` | Sıcak kanal entrypoint'leri için hafif yerel onay bağdaştırıcısı yükleme yardımcıları |
    | `plugin-sdk/approval-handler-runtime` | Daha geniş onay işleyici çalışma zamanı yardımcıları; dar bağdaştırıcı/gateway yüzeyleri yeterliyse onları tercih edin |
    | `plugin-sdk/approval-native-runtime` | Yerel onay hedefi + hesap bağlama yardımcıları |
    | `plugin-sdk/approval-reply-runtime` | Exec/plugin onay yanıt payload yardımcıları |
    | `plugin-sdk/command-auth-native` | Yerel komut kimlik doğrulama + yerel oturum hedefi yardımcıları |
    | `plugin-sdk/command-detection` | Paylaşılan komut algılama yardımcıları |
    | `plugin-sdk/command-surface` | Komut gövdesi normalizasyonu ve komut yüzeyi yardımcıları |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Kanal/plugin secret yüzeyleri için dar secret-sözleşmesi toplama yardımcıları |
    | `plugin-sdk/secret-ref-runtime` | Secret-sözleşmesi/yapılandırma ayrıştırması için dar `coerceSecretRef` ve SecretRef türlendirme yardımcıları |
    | `plugin-sdk/security-runtime` | Paylaşılan güven, DM geçidi, harici içerik ve secret toplama yardımcıları |
    | `plugin-sdk/ssrf-policy` | Ana makine izin listesi ve özel ağ SSRF ilkesi yardımcıları |
    | `plugin-sdk/ssrf-runtime` | Sabitlenmiş dispatcher, SSRF korumalı fetch ve SSRF ilkesi yardımcıları |
    | `plugin-sdk/secret-input` | Secret girdi ayrıştırma yardımcıları |
    | `plugin-sdk/webhook-ingress` | Webhook istek/hedef yardımcıları |
    | `plugin-sdk/webhook-request-guards` | İstek gövdesi boyutu/zaman aşımı yardımcıları |
  </Accordion>

  <Accordion title="Çalışma zamanı ve depolama alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/runtime` | Geniş çalışma zamanı/günlükleme/yedekleme/plugin-kurulum yardımcıları |
    | `plugin-sdk/runtime-env` | Dar çalışma zamanı env, logger, zaman aşımı, yeniden deneme ve backoff yardımcıları |
    | `plugin-sdk/channel-runtime-context` | Genel kanal çalışma zamanı bağlamı kaydı ve arama yardımcıları |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Paylaşılan plugin komutu/hook/http/etkileşimli yardımcıları |
    | `plugin-sdk/hook-runtime` | Paylaşılan webhook/dahili hook işlem hattı yardımcıları |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`, `createLazyRuntimeMethod` ve `createLazyRuntimeSurface` gibi tembel çalışma zamanı içe aktarma/bağlama yardımcıları |
    | `plugin-sdk/process-runtime` | Süreç exec yardımcıları |
    | `plugin-sdk/cli-runtime` | CLI biçimlendirme, bekleme ve sürüm yardımcıları |
    | `plugin-sdk/gateway-runtime` | Gateway istemcisi ve kanal durum yaması yardımcıları |
    | `plugin-sdk/config-runtime` | Yapılandırma yükleme/yazma yardımcıları |
    | `plugin-sdk/telegram-command-config` | Paketlenmiş Telegram sözleşme yüzeyi kullanılamadığında bile Telegram komut adı/açıklama normalizasyonu ve yinelenen/çakışma denetimleri |
    | `plugin-sdk/approval-runtime` | Exec/plugin onay yardımcıları, onay-yetenek oluşturucuları, kimlik doğrulama/profil yardımcıları, yerel yönlendirme/çalışma zamanı yardımcıları |
    | `plugin-sdk/reply-runtime` | Paylaşılan gelen/yanıt çalışma zamanı yardımcıları, parçalama, dağıtım, heartbeat, yanıt planlayıcı |
    | `plugin-sdk/reply-dispatch-runtime` | Dar yanıt dağıtımı/tamamlama yardımcıları |
    | `plugin-sdk/reply-history` | `buildHistoryContext`, `recordPendingHistoryEntry` ve `clearHistoryEntriesIfEnabled` gibi paylaşılan kısa pencere yanıt geçmişi yardımcıları |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Dar metin/markdown parçalama yardımcıları |
    | `plugin-sdk/session-store-runtime` | Oturum deposu yolu + updated-at yardımcıları |
    | `plugin-sdk/state-paths` | Durum/OAuth dizin yolu yardımcıları |
    | `plugin-sdk/routing` | `resolveAgentRoute`, `buildAgentSessionKey` ve `resolveDefaultAgentBoundAccountId` gibi rota/oturum anahtarı/hesap bağlama yardımcıları |
    | `plugin-sdk/status-helpers` | Paylaşılan kanal/hesap durum özeti yardımcıları, çalışma zamanı durum varsayılanları ve sorun meta verisi yardımcıları |
    | `plugin-sdk/target-resolver-runtime` | Paylaşılan hedef çözümleyici yardımcıları |
    | `plugin-sdk/string-normalization-runtime` | Slug/dize normalizasyon yardımcıları |
    | `plugin-sdk/request-url` | Fetch/istek benzeri girdilerden dize URL'leri çıkarın |
    | `plugin-sdk/run-command` | Normalleştirilmiş stdout/stderr sonuçlarıyla zamanlanmış komut çalıştırıcı |
    | `plugin-sdk/param-readers` | Ortak araç/CLI parametre okuyucuları |
    | `plugin-sdk/tool-payload` | Araç sonuç nesnelerinden normalleştirilmiş payload'ları çıkarın |
    | `plugin-sdk/tool-send` | Araç bağımsız değişkenlerinden kanonik gönderim hedef alanlarını çıkarın |
    | `plugin-sdk/temp-path` | Paylaşılan geçici indirme yolu yardımcıları |
    | `plugin-sdk/logging-core` | Alt sistem logger ve redaksiyon yardımcıları |
    | `plugin-sdk/markdown-table-runtime` | Markdown tablo modu yardımcıları |
    | `plugin-sdk/json-store` | Küçük JSON durum okuma/yazma yardımcıları |
    | `plugin-sdk/file-lock` | Yeniden girişli dosya kilidi yardımcıları |
    | `plugin-sdk/persistent-dedupe` | Disk destekli yinelenenleri kaldırma önbelleği yardımcıları |
    | `plugin-sdk/acp-runtime` | ACP çalışma zamanı/oturum ve yanıt dağıtımı yardımcıları |
    | `plugin-sdk/agent-config-primitives` | Dar agent çalışma zamanı yapılandırma-şeması ilkelleri |
    | `plugin-sdk/boolean-param` | Gevşek boolean parametre okuyucusu |
    | `plugin-sdk/dangerous-name-runtime` | Tehlikeli ad eşleme çözümleme yardımcıları |
    | `plugin-sdk/device-bootstrap` | Cihaz bootstrap ve eşleştirme token yardımcıları |
    | `plugin-sdk/extension-shared` | Paylaşılan pasif kanal, durum ve ambient proxy yardımcı ilkelleri |
    | `plugin-sdk/models-provider-runtime` | `/models` komutu/sağlayıcı yanıt yardımcıları |
    | `plugin-sdk/skill-commands-runtime` | Skills komut listeleme yardımcıları |
    | `plugin-sdk/native-command-registry` | Yerel komut kaydı/oluşturma/serileştirme yardımcıları |
    | `plugin-sdk/provider-zai-endpoint` | Z.AI uç nokta algılama yardımcıları |
    | `plugin-sdk/infra-runtime` | Sistem olayı/heartbeat yardımcıları |
    | `plugin-sdk/collection-runtime` | Küçük sınırlı önbellek yardımcıları |
    | `plugin-sdk/diagnostic-runtime` | Tanılama bayrağı ve olay yardımcıları |
    | `plugin-sdk/error-runtime` | Hata grafiği, biçimlendirme, paylaşılan hata sınıflandırma yardımcıları, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Sarılmış fetch, proxy ve sabitlenmiş arama yardımcıları |
    | `plugin-sdk/host-runtime` | Ana makine adı ve SCP ana makinesi normalizasyon yardımcıları |
    | `plugin-sdk/retry-runtime` | Yeniden deneme yapılandırması ve yeniden deneme çalıştırıcı yardımcıları |
    | `plugin-sdk/agent-runtime` | Agent dizini/kimliği/çalışma alanı yardımcıları |
    | `plugin-sdk/directory-runtime` | Yapılandırma destekli dizin sorgusu/yinelenen kaldırma |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Yetenek ve test alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Medya payload oluşturucularına ek olarak paylaşılan medya getirme/dönüştürme/depolama yardımcıları |
    | `plugin-sdk/media-generation-runtime` | Paylaşılan medya üretimi failover yardımcıları, aday seçimi ve eksik model mesajları |
    | `plugin-sdk/media-understanding` | Medya anlama sağlayıcı türleri ve sağlayıcıya dönük görsel/ses yardımcı dışa aktarımları |
    | `plugin-sdk/text-runtime` | Yardımcıya görünür metin temizleme, markdown işleme/parçalama/tablo yardımcıları, redaksiyon yardımcıları, directive-tag yardımcıları ve güvenli metin araçları gibi paylaşılan metin/markdown/günlükleme yardımcıları |
    | `plugin-sdk/text-chunking` | Giden metin parçalama yardımcısı |
    | `plugin-sdk/speech` | Konuşma sağlayıcı türleri ile sağlayıcıya dönük directive, kayıt ve doğrulama yardımcıları |
    | `plugin-sdk/speech-core` | Paylaşılan konuşma sağlayıcı türleri, kayıt, directive ve normalizasyon yardımcıları |
    | `plugin-sdk/realtime-transcription` | Gerçek zamanlı transcription sağlayıcı türleri ve kayıt yardımcıları |
    | `plugin-sdk/realtime-voice` | Gerçek zamanlı ses sağlayıcı türleri ve kayıt yardımcıları |
    | `plugin-sdk/image-generation` | Görsel üretimi sağlayıcı türleri |
    | `plugin-sdk/image-generation-core` | Paylaşılan görsel üretimi türleri, failover, kimlik doğrulama ve kayıt yardımcıları |
    | `plugin-sdk/music-generation` | Müzik üretimi sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/music-generation-core` | Paylaşılan müzik üretimi türleri, failover yardımcıları, sağlayıcı arama ve model-ref ayrıştırma |
    | `plugin-sdk/video-generation` | Video üretimi sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/video-generation-core` | Paylaşılan video üretimi türleri, failover yardımcıları, sağlayıcı arama ve model-ref ayrıştırma |
    | `plugin-sdk/webhook-targets` | Webhook hedef kaydı ve rota kurulum yardımcıları |
    | `plugin-sdk/webhook-path` | Webhook yolu normalizasyon yardımcıları |
    | `plugin-sdk/web-media` | Paylaşılan uzak/yerel medya yükleme yardımcıları |
    | `plugin-sdk/zod` | Plugin SDK tüketicileri için yeniden dışa aktarılan `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Memory alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/memory-core` | Yönetici/yapılandırma/dosya/CLI yardımcıları için paketlenmiş memory-core yardımcı yüzeyi |
    | `plugin-sdk/memory-core-engine-runtime` | Memory dizin/arama çalışma zamanı cephesi |
    | `plugin-sdk/memory-core-host-engine-foundation` | Memory host foundation engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Memory host embedding engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-qmd` | Memory host QMD engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-storage` | Memory host storage engine dışa aktarımları |
    | `plugin-sdk/memory-core-host-multimodal` | Memory host multimodal yardımcıları |
    | `plugin-sdk/memory-core-host-query` | Memory host sorgu yardımcıları |
    | `plugin-sdk/memory-core-host-secret` | Memory host secret yardımcıları |
    | `plugin-sdk/memory-core-host-events` | Memory host olay günlüğü yardımcıları |
    | `plugin-sdk/memory-core-host-status` | Memory host durum yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-cli` | Memory host CLI çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-core` | Memory host çekirdek çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-files` | Memory host dosya/çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-host-core` | Memory host çekirdek çalışma zamanı yardımcıları için satıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-events` | Memory host olay günlüğü yardımcıları için satıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-files` | Memory host dosya/çalışma zamanı yardımcıları için satıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-markdown` | Memory'e bitişik plugin'ler için paylaşılan yönetilen-markdown yardımcıları |
    | `plugin-sdk/memory-host-search` | Search-manager erişimi için etkin memory çalışma zamanı cephesi |
    | `plugin-sdk/memory-host-status` | Memory host durum yardımcıları için satıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-lancedb` | Paketlenmiş memory-lancedb yardımcı yüzeyi |
  </Accordion>

  <Accordion title="Ayrılmış paketlenmiş-yardımcı alt yolları">
    | Aile | Mevcut alt yollar | Amaçlanan kullanım |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Paketlenmiş browser plugin destek yardımcıları (`browser-support` uyumluluk barrel'i olarak kalır) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Paketlenmiş Matrix yardımcı/çalışma zamanı yüzeyi |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Paketlenmiş LINE yardımcı/çalışma zamanı yüzeyi |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Paketlenmiş IRC yardımcı yüzeyi |
    | Kanala özel yardımcılar | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Paketlenmiş kanal uyumluluk/yardımcı yüzeyleri |
    | Kimlik doğrulama/plugin'e özel yardımcılar | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Paketlenmiş özellik/plugin yardımcı yüzeyleri; `plugin-sdk/github-copilot-token` şu anda `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` ve `resolveCopilotApiToken` dışa aktarır |
  </Accordion>
</AccordionGroup>

## Kayıt API'si

`register(api)` callback'i şu yöntemlere sahip bir `OpenClawPluginApi` nesnesi
alır:

### Yetenek kaydı

| Yöntem                                           | Kaydettiği şey                  |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Metin çıkarımı (LLM)             |
| `api.registerCliBackend(...)`                    | Yerel CLI çıkarım arka ucu       |
| `api.registerChannel(...)`                       | Mesajlaşma kanalı                |
| `api.registerSpeechProvider(...)`                | Metinden sese / STT sentezi      |
| `api.registerRealtimeTranscriptionProvider(...)` | Akışlı gerçek zamanlı transcription |
| `api.registerRealtimeVoiceProvider(...)`         | İki yönlü gerçek zamanlı ses oturumları |
| `api.registerMediaUnderstandingProvider(...)`    | Görsel/ses/video analizi         |
| `api.registerImageGenerationProvider(...)`       | Görsel üretimi                   |
| `api.registerMusicGenerationProvider(...)`       | Müzik üretimi                    |
| `api.registerVideoGenerationProvider(...)`       | Video üretimi                    |
| `api.registerWebFetchProvider(...)`              | Web fetch / scrape sağlayıcısı   |
| `api.registerWebSearchProvider(...)`             | Web arama                        |

### Araçlar ve komutlar

| Yöntem                          | Kaydettiği şey                                  |
| ------------------------------- | ----------------------------------------------- |
| `api.registerTool(tool, opts?)` | Agent aracı (zorunlu veya `{ optional: true }`) |
| `api.registerCommand(def)`      | Özel komut (LLM'yi atlar)                       |

### Altyapı

| Yöntem                                         | Kaydettiği şey                      |
| ---------------------------------------------- | ----------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Olay hook'u                         |
| `api.registerHttpRoute(params)`                | Gateway HTTP uç noktası             |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC yöntemi                 |
| `api.registerCli(registrar, opts?)`            | CLI alt komutu                      |
| `api.registerService(service)`                 | Arka plan hizmeti                   |
| `api.registerInteractiveHandler(registration)` | Etkileşimli işleyici                |
| `api.registerMemoryPromptSupplement(builder)`  | Ekleyici memory'e bitişik prompt bölümü |
| `api.registerMemoryCorpusSupplement(adapter)`  | Ekleyici memory arama/okuma derlemi |

Ayrılmış çekirdek yönetici ad alanları (`config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`), bir plugin daha dar bir gateway yöntem kapsamı
atamaya çalışsa bile her zaman `operator.admin` olarak kalır. Plugin'e ait
yöntemler için plugin'e özgü önekleri tercih edin.

### CLI kayıt meta verisi

`api.registerCli(registrar, opts?)` iki tür üst düzey meta veri kabul eder:

- `commands`: registrar'ın sahip olduğu açık komut kökleri
- `descriptors`: kök CLI yardımı, yönlendirme ve tembel plugin CLI kaydı için
  ayrıştırma zamanı komut tanımlayıcıları

Bir plugin komutunun normal kök CLI yolunda tembel yüklenmiş kalmasını
istiyorsanız, bu registrar tarafından açığa çıkarılan her üst düzey komut
kökünü kapsayan `descriptors` sağlayın.

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
        description: "Matrix hesaplarını, doğrulamayı, cihazları ve profil durumunu yönetin",
        hasSubcommands: true,
      },
    ],
  },
);
```

Normal kök CLI kaydı için tembel yükleme gerekmiyorsa yalnızca `commands`
kullanın. Bu istekli uyumluluk yolu desteklenmeye devam eder, ancak ayrıştırma
zamanı tembel yükleme için descriptor destekli yer tutucular kurmaz.

### CLI arka uç kaydı

`api.registerCliBackend(...)`, bir plugin'in `codex-cli` gibi yerel bir
AI CLI arka ucunun varsayılan yapılandırmasına sahip olmasına izin verir.

- Arka uç `id`, `codex-cli/gpt-5` gibi model ref'lerinde sağlayıcı öneki olur.
- Arka uç `config`, `agents.defaults.cliBackends.<id>` ile aynı şekli kullanır.
- Kullanıcı yapılandırması yine önceliklidir. OpenClaw, CLI'yi çalıştırmadan
  önce plugin varsayılanı üzerine `agents.defaults.cliBackends.<id>` değerini
  birleştirir.
- Bir arka uç birleştirmeden sonra uyumluluk yeniden yazımları gerektiriyorsa
  `normalizeConfig` kullanın (örneğin eski bayrak şekillerini normalize etmek
  için).

### Münhasır yuvalar

| Yöntem                                     | Kaydettiği şey                                                                                                                                         |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `api.registerContextEngine(id, factory)`   | Bağlam motoru (aynı anda yalnızca biri etkin). `assemble()` callback'i, motorun prompt eklemelerini uyarlayabilmesi için `availableTools` ve `citationsMode` alır. |
| `api.registerMemoryCapability(capability)` | Birleşik memory yeteneği                                                                                                                               |
| `api.registerMemoryPromptSection(builder)` | Memory prompt bölümü oluşturucusu                                                                                                                      |
| `api.registerMemoryFlushPlan(resolver)`    | Memory flush planı çözümleyicisi                                                                                                                       |
| `api.registerMemoryRuntime(runtime)`       | Memory çalışma zamanı bağdaştırıcısı                                                                                                                   |

### Memory embedding bağdaştırıcıları

| Yöntem                                         | Kaydettiği şey                                 |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Etkin plugin için memory embedding bağdaştırıcısı |

- `registerMemoryCapability`, tercih edilen münhasır memory-plugin API'sidir.
- `registerMemoryCapability`, eşlik eden plugin'lerin dışa aktarılan memory
  yapılarını belirli bir memory plugin'inin özel düzenine ulaşmadan
  `openclaw/plugin-sdk/memory-host-core` üzerinden tüketebilmesi için
  `publicArtifacts.listArtifacts(...)` da sunabilir.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` ve
  `registerMemoryRuntime`, eskiyle uyumlu münhasır memory-plugin API'leridir.
- `registerMemoryEmbeddingProvider`, etkin memory plugin'inin bir veya daha
  fazla embedding bağdaştırıcısı kimliği kaydetmesine izin verir (örneğin
  `openai`, `gemini` veya özel plugin tanımlı bir kimlik).
- `agents.defaults.memorySearch.provider` ve
  `agents.defaults.memorySearch.fallback` gibi kullanıcı yapılandırmaları, bu
  kaydedilmiş bağdaştırıcı kimliklerine göre çözülür.

### Olaylar ve yaşam döngüsü

| Yöntem                                       | Yaptığı şey                   |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Tiplenmiş yaşam döngüsü hook'u |
| `api.onConversationBindingResolved(handler)` | Konuşma bağlama callback'i     |

### Hook karar semantiği

- `before_tool_call`: `{ block: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında, daha düşük öncelikli işleyiciler atlanır.
- `before_tool_call`: `{ block: false }` döndürmek karar yok olarak değerlendirilir (`block` alanını hiç vermemekle aynıdır), geçersiz kılma olarak değil.
- `before_install`: `{ block: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında, daha düşük öncelikli işleyiciler atlanır.
- `before_install`: `{ block: false }` döndürmek karar yok olarak değerlendirilir (`block` alanını hiç vermemekle aynıdır), geçersiz kılma olarak değil.
- `reply_dispatch`: `{ handled: true, ... }` döndürmek kesindir. Herhangi bir işleyici dağıtımı sahiplendiğinde, daha düşük öncelikli işleyiciler ve varsayılan model dağıtım yolu atlanır.
- `message_sending`: `{ cancel: true }` döndürmek kesindir. Herhangi bir işleyici bunu ayarladığında, daha düşük öncelikli işleyiciler atlanır.
- `message_sending`: `{ cancel: false }` döndürmek karar yok olarak değerlendirilir (`cancel` alanını hiç vermemekle aynıdır), geçersiz kılma olarak değil.

### API nesnesi alanları

| Alan                    | Tür                       | Açıklama                                                                                  |
| ----------------------- | ------------------------- | ----------------------------------------------------------------------------------------- |
| `api.id`                | `string`                  | Plugin kimliği                                                                            |
| `api.name`              | `string`                  | Görünen ad                                                                                |
| `api.version`           | `string?`                 | Plugin sürümü (isteğe bağlı)                                                              |
| `api.description`       | `string?`                 | Plugin açıklaması (isteğe bağlı)                                                          |
| `api.source`            | `string`                  | Plugin kaynak yolu                                                                        |
| `api.rootDir`           | `string?`                 | Plugin kök dizini (isteğe bağlı)                                                          |
| `api.config`            | `OpenClawConfig`          | Geçerli yapılandırma anlık görüntüsü (mevcutsa etkin bellek içi çalışma zamanı anlık görüntüsü) |
| `api.pluginConfig`      | `Record<string, unknown>` | `plugins.entries.<id>.config` içinden plugin'e özgü yapılandırma                         |
| `api.runtime`           | `PluginRuntime`           | [Çalışma zamanı yardımcıları](/tr/plugins/sdk-runtime)                                       |
| `api.logger`            | `PluginLogger`            | Kapsamlı logger (`debug`, `info`, `warn`, `error`)                                        |
| `api.registrationMode`  | `PluginRegistrationMode`  | Geçerli yükleme modu; `"setup-runtime"` hafif, tam entry öncesi başlatma/kurulum penceresidir |
| `api.resolvePath(input)` | `(string) => string`     | Yolu plugin köküne göre çözümleyin                                                        |

## Dahili modül kuralı

Plugin'iniz içinde, dahili içe aktarmalar için yerel barrel dosyaları kullanın:

```
my-plugin/
  api.ts            # Harici tüketiciler için herkese açık dışa aktarımlar
  runtime-api.ts    # Yalnızca dahili çalışma zamanı dışa aktarımları
  index.ts          # Plugin entry point
  setup-entry.ts    # Yalnızca hafif kurulum entry'si (isteğe bağlı)
```

<Warning>
  Üretim kodunda kendi plugin'inizi asla `openclaw/plugin-sdk/<your-plugin>`
  üzerinden içe aktarmayın. Dahili içe aktarmaları `./api.ts` veya
  `./runtime-api.ts` üzerinden yönlendirin. SDK yolu yalnızca harici
  sözleşmedir.
</Warning>

Cephe üzerinden yüklenen paketlenmiş plugin herkese açık yüzeyleri (`api.ts`,
`runtime-api.ts`, `index.ts`, `setup-entry.ts` ve benzer herkese açık entry
dosyaları), OpenClaw zaten çalışıyorsa artık etkin çalışma zamanı yapılandırma
anlık görüntüsünü tercih eder. Henüz çalışma zamanı anlık görüntüsü yoksa,
diskteki çözülmüş yapılandırma dosyasına geri dönerler.

Sağlayıcı plugin'leri, bir yardımcı bilinçli olarak sağlayıcıya özelse ve henüz
genel bir SDK alt yoluna ait değilse dar bir plugin-yerel sözleşme barrel'i de
sunabilir. Mevcut paketlenmiş örnek: Anthropic sağlayıcısı, Anthropic beta
başlığı ve `service_tier` mantığını genel bir `plugin-sdk/*` sözleşmesine
taşımak yerine Claude akış yardımcılarını kendi herkese açık `api.ts` /
`contract-api.ts` yüzeyinde tutar.

Diğer mevcut paketlenmiş örnekler:

- `@openclaw/openai-provider`: `api.ts`, sağlayıcı oluşturucularını,
  varsayılan model yardımcılarını ve gerçek zamanlı sağlayıcı oluşturucularını
  dışa aktarır
- `@openclaw/openrouter-provider`: `api.ts`, sağlayıcı oluşturucuyu ve
  onboarding/yapılandırma yardımcılarını dışa aktarır

<Warning>
  Extension üretim kodu ayrıca `openclaw/plugin-sdk/<other-plugin>` içe
  aktarmalarından da kaçınmalıdır. Bir yardımcı gerçekten paylaşılıyorsa, iki
  plugin'i birbirine bağlamak yerine onu `openclaw/plugin-sdk/speech`,
  `.../provider-model-shared` veya başka bir yetenek odaklı yüzey gibi tarafsız
  bir SDK alt yoluna taşıyın.
</Warning>

## İlgili

- [Entry Points](/tr/plugins/sdk-entrypoints) — `definePluginEntry` ve `defineChannelPluginEntry` seçenekleri
- [Çalışma Zamanı Yardımcıları](/tr/plugins/sdk-runtime) — tam `api.runtime` ad alanı başvurusu
- [Kurulum ve Yapılandırma](/tr/plugins/sdk-setup) — paketleme, manifestler, yapılandırma şemaları
- [Test](/tr/plugins/sdk-testing) — test araçları ve lint kuralları
- [SDK Geçişi](/tr/plugins/sdk-migration) — kullanımdan kaldırılmış yüzeylerden geçiş
- [Plugin İç Yapısı](/tr/plugins/architecture) — derin mimari ve yetenek modeli
