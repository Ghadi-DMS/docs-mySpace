---
read_when:
    - Bellek yükseltmenin otomatik olarak çalışmasını istiyorsunuz
    - Her rüya görme aşamasının ne yaptığını anlamak istiyorsunuz
    - Birleştirmeyi `MEMORY.md` dosyasını kirletmeden ayarlamak istiyorsunuz
summary: Hafif, derin ve REM aşamalarıyla arka plan bellek birleştirmesi ve bir Rüya Günlüğü
title: Rüya Görme (deneysel)
x-i18n:
    generated_at: "2026-04-08T08:43:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0254f3b0949158264e583c12f36f2b1a83d1b44dc4da01a1b272422d38e8655d
    source_path: concepts/dreaming.md
    workflow: 15
---

# Rüya Görme (deneysel)

Rüya görme, `memory-core` içindeki arka plan bellek birleştirme sistemidir.
OpenClaw'ın güçlü kısa süreli sinyalleri kalıcı belleğe taşımasına yardımcı olurken
süreci açıklanabilir ve gözden geçirilebilir tutar.

Rüya görme **isteğe bağlıdır** ve varsayılan olarak devre dışıdır.

## Rüya görme nereye yazar

Rüya görme iki tür çıktı tutar:

- `memory/.dreams/` içinde **Makine durumu** (anımsama deposu, aşama sinyalleri, içe alma denetim noktaları, kilitler).
- `DREAMS.md` içinde (veya mevcut `dreams.md`) **İnsan tarafından okunabilir çıktı** ve `memory/dreaming/<phase>/YYYY-MM-DD.md` altında isteğe bağlı aşama raporu dosyaları.

Uzun vadeli yükseltme yine yalnızca `MEMORY.md` dosyasına yazar.

## Aşama modeli

Rüya görme üç iş birliği yapan aşama kullanır:

| Aşama | Amaç                                      | Kalıcı yazım       |
| ----- | ----------------------------------------- | ------------------ |
| Hafif | Son kısa süreli materyali sıralamak ve hazırlamak | Hayır              |
| Derin | Kalıcı adayları puanlamak ve yükseltmek   | Evet (`MEMORY.md`) |
| REM   | Temalar ve tekrar eden fikirler üzerine düşünmek | Hayır              |

Bu aşamalar ayrı kullanıcı tarafından yapılandırılan
"modlar" değil, iç uygulama ayrıntılarıdır.

### Hafif aşama

Hafif aşama son günlük bellek sinyallerini ve anımsama izlerini içe alır, bunların
yinelenenlerini kaldırır ve aday satırları hazırlar.

- Kısa süreli anımsama durumundan, son günlük bellek dosyalarından ve mevcut olduğunda sansürlenmiş oturum dökümlerinden okur.
- Depolama satır içi çıktı içerdiğinde yönetilen bir `## Light Sleep` bloğu yazar.
- Daha sonra derin sıralama için pekiştirme sinyalleri kaydeder.
- Asla `MEMORY.md` dosyasına yazmaz.

### Derin aşama

Derin aşama neyin uzun vadeli bellek olacağına karar verir.

- Adayları ağırlıklı puanlama ve eşik geçitleri kullanarak sıralar.
- Geçmek için `minScore`, `minRecallCount` ve `minUniqueQueries` gerektirir.
- Yazmadan önce parçaları canlı günlük dosyalardan yeniden alır; böylece eski/silinmiş parçalar atlanır.
- Yükseltilen girdileri `MEMORY.md` dosyasına ekler.
- `DREAMS.md` içine bir `## Deep Sleep` özeti yazar ve isteğe bağlı olarak `memory/dreaming/deep/YYYY-MM-DD.md` dosyasına yazar.

### REM aşaması

REM aşaması örüntüleri ve yansıtıcı sinyalleri çıkarır.

- Son kısa süreli izlerden tema ve düşünüm özetleri oluşturur.
- Depolama satır içi çıktı içerdiğinde yönetilen bir `## REM Sleep` bloğu yazar.
- Derin sıralamada kullanılan REM pekiştirme sinyallerini kaydeder.
- Asla `MEMORY.md` dosyasına yazmaz.

## Oturum dökümü içe alma

Rüya görme, sansürlenmiş oturum dökümlerini rüya görme bütüncesine içe alabilir. Dökümler
mevcut olduğunda, bunlar günlük bellek sinyalleri ve anımsama izleriyle birlikte hafif
aşamaya beslenir. Kişisel ve hassas içerik içe almadan önce sansürlenir.

## Rüya Günlüğü

Rüya görme ayrıca `DREAMS.md` içinde anlatı tarzında bir **Rüya Günlüğü** tutar.
Her aşama yeterli materyale sahip olduktan sonra, `memory-core` en iyi çaba ile bir arka plan
alt ajan dönüşü çalıştırır (varsayılan çalışma zamanı modeli kullanılarak) ve kısa bir günlük girdisi ekler.

Bu günlük, yükseltme kaynağı değil, Dreams kullanıcı arayüzünde insanlar tarafından okunmak içindir.

## Derin sıralama sinyalleri

Derin sıralama altı ağırlıklı temel sinyal ve aşama pekiştirmesi kullanır:

| Sinyal              | Ağırlık | Açıklama                                        |
| ------------------- | ------ | ------------------------------------------------ |
| Sıklık              | 0.24   | Girdinin kaç kısa süreli sinyal biriktirdiği    |
| İlgililik           | 0.30   | Girdi için ortalama geri getirme kalitesi       |
| Sorgu çeşitliliği   | 0.15   | Onu ortaya çıkaran farklı sorgu/gün bağlamları  |
| Güncellik           | 0.15   | Zamana göre azalan tazelik puanı                |
| Birleştirme         | 0.10   | Çok günlük tekrar gücü                          |
| Kavramsal zenginlik | 0.06   | Parça/yoldan gelen kavram etiketi yoğunluğu     |

Hafif ve REM aşaması isabetleri,
`memory/.dreams/phase-signals.json` içinden güncelliğe göre azalan küçük bir artış ekler.

## Zamanlama

Etkinleştirildiğinde, `memory-core` tam bir rüya görme
taraması için bir cron işini otomatik olarak yönetir. Her tarama aşamaları sırayla çalıştırır: hafif -> REM -> derin.

Varsayılan sıklık davranışı:

| Ayar                 | Varsayılan |
| -------------------- | ---------- |
| `dreaming.frequency` | `0 3 * * *` |

## Hızlı başlangıç

Rüya görmeyi etkinleştirin:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Özel bir tarama sıklığıyla rüya görmeyi etkinleştirin:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Eğik çizgi komutu

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## CLI iş akışı

Önizleme veya elle uygulama için CLI yükseltmesini kullanın:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

Elle `memory promote`, CLI bayraklarıyla geçersiz kılınmadıkça varsayılan olarak derin aşama eşiklerini kullanır.

Belirli bir adayın neden yükseleceğini veya yükselmeyeceğini açıklayın:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Hiçbir şey yazmadan REM düşüncelerini, aday doğruları ve derin yükseltme çıktısını önizleyin:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Temel varsayılanlar

Tüm ayarlar `plugins.entries.memory-core.config.dreaming` altında bulunur.

| Anahtar     | Varsayılan |
| ----------- | ---------- |
| `enabled`   | `false`    |
| `frequency` | `0 3 * * *` |

Aşama ilkesi, eşikler ve depolama davranışı iç uygulama
ayrıntılarıdır (kullanıcıya yönelik yapılandırma değildir).

Tam anahtar listesi için [Bellek yapılandırma başvurusu](/tr/reference/memory-config#dreaming-experimental) sayfasına bakın.

## Dreams kullanıcı arayüzü

Etkinleştirildiğinde, Gateway içindeki **Dreams** sekmesi şunları gösterir:

- mevcut rüya görme etkin durumu
- aşama düzeyinde durum ve yönetilen tarama varlığı
- kısa süreli, uzun süreli ve bugün yükseltilen sayıları
- bir sonraki zamanlanmış çalıştırmanın zamanı
- `doctor.memory.dreamDiary` tarafından desteklenen genişletilebilir bir Rüya Günlüğü okuyucusu

## İlgili

- [Bellek](/tr/concepts/memory)
- [Bellek Arama](/tr/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Bellek yapılandırma başvurusu](/tr/reference/memory-config)
