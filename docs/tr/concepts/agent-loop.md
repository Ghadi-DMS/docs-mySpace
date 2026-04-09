---
read_when:
    - Aracı döngüsü veya yaşam döngüsü olayları için tam ve adım adım bir açıklamaya ihtiyacınız var
summary: Aracı döngüsü yaşam döngüsü, akışlar ve bekleme semantiği
title: Aracı Döngüsü
x-i18n:
    generated_at: "2026-04-09T01:28:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 32d3a73df8dabf449211a6183a70dcfd2a9b6f584dc76d0c4c9147582b2ca6a1
    source_path: concepts/agent-loop.md
    workflow: 15
---

# Aracı Döngüsü (OpenClaw)

Aracı odaklı bir döngü, bir aracının tam “gerçek” çalıştırmasıdır: alım → bağlam birleştirme → model çıkarımı →
araç yürütme → akış halinde yanıtlar → kalıcılık. Bu, bir mesajı eylemlere ve son bir yanıta dönüştüren, aynı zamanda oturum durumunu tutarlı tutan yetkili yoldur.

OpenClaw içinde bir döngü, model düşünürken, araç çağırırken ve çıktı akışı sağlarken yaşam döngüsü ve akış olayları yayan, oturum başına tek ve serileştirilmiş bir çalıştırmadır. Bu belge, bu özgün döngünün uçtan uca nasıl bağlandığını açıklar.

## Giriş noktaları

- Gateway RPC: `agent` ve `agent.wait`.
- CLI: `agent` komutu.

## Nasıl çalışır (üst düzey)

1. `agent` RPC parametreleri doğrular, oturumu çözümler (sessionKey/sessionId), oturum meta verilerini kalıcı hale getirir ve hemen `{ runId, acceptedAt }` döndürür.
2. `agentCommand` aracıyı çalıştırır:
   - model + thinking/verbose varsayılanlarını çözümler
   - Skills anlık görüntüsünü yükler
   - `runEmbeddedPiAgent` çağırır (pi-agent-core çalışma zamanı)
   - gömülü döngü bir tane yaymazsa **yaşam döngüsü end/error** yayar
3. `runEmbeddedPiAgent`:
   - çalıştırmaları oturum başına ve genel kuyruklar üzerinden serileştirir
   - modeli + auth profilini çözümler ve pi oturumunu oluşturur
   - pi olaylarına abone olur ve assistant/tool deltalarını akış halinde iletir
   - zaman aşımını zorunlu kılar -> aşılırsa çalıştırmayı iptal eder
   - yükleri + kullanım meta verilerini döndürür
4. `subscribeEmbeddedPiSession`, pi-agent-core olaylarını OpenClaw `agent` akışına köprüler:
   - araç olayları => `stream: "tool"`
   - assistant deltaları => `stream: "assistant"`
   - yaşam döngüsü olayları => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait`, `waitForAgentRun` kullanır:
   - `runId` için **yaşam döngüsü end/error** durumunu bekler
   - `{ status: ok|error|timeout, startedAt, endedAt, error? }` döndürür

## Kuyruklama + eşzamanlılık

- Çalıştırmalar, oturum anahtarı başına (oturum şeridi) ve isteğe bağlı olarak genel bir şerit üzerinden serileştirilir.
- Bu, araç/oturum yarışlarını önler ve oturum geçmişini tutarlı tutar.
- Mesajlaşma kanalları, bu şerit sistemine beslenen kuyruk modlarını (collect/steer/followup) seçebilir.
  Bkz. [Komut Kuyruğu](/tr/concepts/queue).

## Oturum + çalışma alanı hazırlığı

- Çalışma alanı çözümlenir ve oluşturulur; korumalı alan çalıştırmaları bir korumalı alan çalışma alanı köküne yönlendirilebilir.
- Skills yüklenir (veya bir anlık görüntüden yeniden kullanılır) ve env ile prompt içine enjekte edilir.
- Bootstrap/context dosyaları çözümlenir ve sistem prompt raporuna enjekte edilir.
- Bir oturum yazma kilidi alınır; `SessionManager`, akış başlamadan önce açılır ve hazırlanır.

## Prompt birleştirme + sistem promptu

- Sistem promptu, OpenClaw’ın temel promptu, skills promptu, bootstrap bağlamı ve çalıştırma başına geçersiz kılmalar kullanılarak oluşturulur.
- Modele özgü sınırlar ve sıkıştırma için ayrılan token sayıları uygulanır.
- Modelin ne gördüğü için bkz. [Sistem promptu](/tr/concepts/system-prompt).

## Hook noktaları (nerede araya girebilirsiniz)

OpenClaw iki hook sistemine sahiptir:

- **Dahili hook'lar** (Gateway hook'ları): komutlar ve yaşam döngüsü olayları için olay güdümlü komut dosyaları.
- **Plugin hook'ları**: aracı/araç yaşam döngüsü ve gateway ardışık düzeni içindeki uzatma noktaları.

### Dahili hook'lar (Gateway hook'ları)

- **`agent:bootstrap`**: bootstrap dosyaları oluşturulurken, sistem promptu son halini almadan önce çalışır.
  Bunu bootstrap bağlam dosyaları eklemek/kaldırmak için kullanın.
- **Komut hook'ları**: `/new`, `/reset`, `/stop` ve diğer komut olayları (bkz. Hooks belgesi).

Kurulum ve örnekler için bkz. [Hooks](/tr/automation/hooks).

### Plugin hook'ları (aracı + gateway yaşam döngüsü)

Bunlar aracı döngüsü veya gateway ardışık düzeni içinde çalışır:

- **`before_model_resolve`**: model çözümlemesinden önce provider/modeli belirlenimci biçimde geçersiz kılmak için oturum öncesinde (`messages` olmadan) çalışır.
- **`before_prompt_build`**: oturum yüklendikten sonra (`messages` ile) prompt gönderiminden önce `prependContext`, `systemPrompt`, `prependSystemContext` veya `appendSystemContext` enjekte etmek için çalışır. Tur başına dinamik metin için `prependContext`, sistem promptu alanında yer alması gereken kararlı yönlendirme için sistem bağlamı alanlarını kullanın.
- **`before_agent_start`**: eski uyumluluk hook'udur; her iki aşamada da çalışabilir; bunun yerine yukarıdaki açık hook'ları tercih edin.
- **`before_agent_reply`**: satır içi eylemlerden sonra ve LLM çağrısından önce çalışır; bir plugin'in turu sahiplenmesine ve sentetik bir yanıt döndürmesine veya turu tamamen sessize almasına izin verir.
- **`agent_end`**: tamamlandıktan sonra son mesaj listesini ve çalışma meta verilerini inceler.
- **`before_compaction` / `after_compaction`**: sıkıştırma döngülerini gözlemler veya not ekler.
- **`before_tool_call` / `after_tool_call`**: araç parametrelerine/sonuçlarına araya girer.
- **`before_install`**: yerleşik tarama bulgularını inceler ve isteğe bağlı olarak skill veya plugin kurulumlarını engeller.
- **`tool_result_persist`**: araç sonuçları oturum dökümüne yazılmadan önce bunları eşzamanlı olarak dönüştürür.
- **`message_received` / `message_sending` / `message_sent`**: gelen + giden mesaj hook'ları.
- **`session_start` / `session_end`**: oturum yaşam döngüsü sınırları.
- **`gateway_start` / `gateway_stop`**: gateway yaşam döngüsü olayları.

Giden/araç korumaları için hook karar kuralları:

- `before_tool_call`: `{ block: true }` kesindir ve daha düşük öncelikli işleyicileri durdurur.
- `before_tool_call`: `{ block: false }` bir no-op'tur ve önceki bir engeli temizlemez.
- `before_install`: `{ block: true }` kesindir ve daha düşük öncelikli işleyicileri durdurur.
- `before_install`: `{ block: false }` bir no-op'tur ve önceki bir engeli temizlemez.
- `message_sending`: `{ cancel: true }` kesindir ve daha düşük öncelikli işleyicileri durdurur.
- `message_sending`: `{ cancel: false }` bir no-op'tur ve önceki bir iptali temizlemez.

Hook API'si ve kayıt ayrıntıları için bkz. [Plugin hook'ları](/tr/plugins/architecture#provider-runtime-hooks).

## Akış + kısmi yanıtlar

- Assistant deltaları pi-agent-core'dan akış halinde iletilir ve `assistant` olayları olarak yayılır.
- Blok akışı, kısmi yanıtları `text_end` veya `message_end` anında yayabilir.
- Akıl yürütme akışı ayrı bir akış olarak veya blok yanıtları olarak yayılabilir.
- Parçalama ve blok yanıt davranışı için bkz. [Akış](/tr/concepts/streaming).

## Araç yürütme + mesajlaşma araçları

- Araç start/update/end olayları `tool` akışında yayılır.
- Araç sonuçları, günlüğe kaydedilmeden/yayılmadan önce boyut ve görsel yükleri açısından temizlenir.
- Mesajlaşma aracı gönderimleri, yinelenen assistant onaylarını bastırmak için izlenir.

## Yanıt şekillendirme + bastırma

- Son yükler şunlardan birleştirilir:
  - assistant metni (ve isteğe bağlı akıl yürütme)
  - satır içi araç özetleri (verbose + izinliyse)
  - model hata verdiğinde assistant hata metni
- Tam sessizlik belirteci `NO_REPLY` / `no_reply`, giden
  yüklerden filtrelenir.
- Mesajlaşma aracı yinelenmeleri son yük listesinden kaldırılır.
- Görüntülenebilir hiçbir yük kalmazsa ve bir araç hata verdiyse, yedek bir araç hata yanıtı yayılır
  (bir mesajlaşma aracı zaten kullanıcıya görünür bir yanıt göndermediyse).

## Sıkıştırma + yeniden denemeler

- Otomatik sıkıştırma `compaction` akış olayları yayar ve bir yeniden denemeyi tetikleyebilir.
- Yeniden denemede, yinelenen çıktıyı önlemek için bellek içi arabellekler ve araç özetleri sıfırlanır.
- Sıkıştırma ardışık düzeni için bkz. [Sıkıştırma](/tr/concepts/compaction).

## Olay akışları (bugün)

- `lifecycle`: `subscribeEmbeddedPiSession` tarafından yayılır (ve yedek olarak `agentCommand` tarafından)
- `assistant`: pi-agent-core'dan akış halinde gelen deltalar
- `tool`: pi-agent-core'dan akış halinde gelen araç olayları

## Sohbet kanalı işleme

- Assistant deltaları, sohbet `delta` mesajları içinde arabelleğe alınır.
- Bir sohbet `final`, **yaşam döngüsü end/error** anında yayılır.

## Zaman aşımları

- `agent.wait` varsayılanı: 30 sn (yalnızca bekleme). `timeoutMs` parametresi geçersiz kılar.
- Aracı çalışma zamanı: `agents.defaults.timeoutSeconds` varsayılanı 172800 sn (48 saat); `runEmbeddedPiAgent` içindeki iptal zamanlayıcısında uygulanır.
- LLM boşta kalma zaman aşımı: `agents.defaults.llm.idleTimeoutSeconds`, boşta kalma penceresi dolmadan önce hiçbir yanıt parçası gelmezse model isteğini iptal eder. Bunu yavaş yerel modeller veya akıl yürütme/araç çağrısı provider'ları için açıkça ayarlayın; devre dışı bırakmak için 0 yapın. Ayarlanmamışsa OpenClaw, yapılandırılmışsa `agents.defaults.timeoutSeconds`, aksi halde 60 sn kullanır. Açık bir LLM veya aracı zaman aşımı olmayan cron tetiklemeli çalıştırmalar, boşta izleme mekanizmasını devre dışı bırakır ve cron dış zaman aşımına güvenir.

## İşlerin erken bitebileceği yerler

- Aracı zaman aşımı (iptal)
- AbortSignal (iptal)
- Gateway bağlantısının kesilmesi veya RPC zaman aşımı
- `agent.wait` zaman aşımı (yalnızca bekleme, aracıyı durdurmaz)

## İlgili

- [Araçlar](/tr/tools) — kullanılabilir aracı araçları
- [Hooks](/tr/automation/hooks) — aracı yaşam döngüsü olayları tarafından tetiklenen olay güdümlü komut dosyaları
- [Sıkıştırma](/tr/concepts/compaction) — uzun konuşmaların nasıl özetlendiği
- [Exec Onayları](/tr/tools/exec-approvals) — kabuk komutları için onay geçitleri
- [Thinking](/tr/tools/thinking) — thinking/akıl yürütme seviyesi yapılandırması
