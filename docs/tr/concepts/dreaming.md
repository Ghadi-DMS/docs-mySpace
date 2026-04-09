---
read_when:
    - Bellek yükseltmenin otomatik olarak çalışmasını istiyorsunuz
    - Her rüya görme evresinin ne yaptığını anlamak istiyorsunuz
    - Sağlamlaştırmayı `MEMORY.md` dosyasını kirletmeden ayarlamak istiyorsunuz
summary: Hafif, derin ve REM evreleri ile arka plan bellek sağlamlaştırması ve bir Rüya Günlüğü
title: Rüya görme (deneysel)
x-i18n:
    generated_at: "2026-04-09T01:27:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26476eddb8260e1554098a6adbb069cf7f5e284cf2e09479c6d9d8f8b93280ef
    source_path: concepts/dreaming.md
    workflow: 15
---

# Rüya görme (deneysel)

Rüya görme, `memory-core` içindeki arka plan bellek sağlamlaştırma sistemidir.
OpenClaw'un güçlü kısa süreli sinyalleri kalıcı belleğe taşımasına yardımcı olurken
sürecin açıklanabilir ve incelenebilir kalmasını sağlar.

Rüya görme **isteğe bağlıdır** ve varsayılan olarak devre dışıdır.

## Rüya görmenin yazdıkları

Rüya görme iki tür çıktı tutar:

- `memory/.dreams/` içinde **makine durumu** (geri çağırma deposu, evre sinyalleri, içe aktarma denetim noktaları, kilitler).
- `DREAMS.md` içinde (veya mevcut `dreams.md` içinde) **insan tarafından okunabilir çıktı** ve `memory/dreaming/<phase>/YYYY-MM-DD.md` altında isteğe bağlı evre raporu dosyaları.

Uzun vadeli yükseltme hâlâ yalnızca `MEMORY.md` dosyasına yazar.

## Evre modeli

Rüya görme, birlikte çalışan üç evre kullanır:

| Evre | Amaç                                      | Kalıcı yazım      |
| ----- | ----------------------------------------- | ----------------- |
| Hafif | Yakın tarihli kısa süreli materyali sıralamak ve hazırlamak | Hayır             |
| Derin | Kalıcı adayları puanlamak ve yükseltmek   | Evet (`MEMORY.md`) |
| REM   | Temalar ve tekrar eden fikirler üzerine düşünmek | Hayır        |

Bu evreler, kullanıcı tarafından ayrı ayrı yapılandırılan
"modlar" değil, iç uygulama ayrıntılarıdır.

### Hafif evre

Hafif evre, yakın tarihli günlük bellek sinyallerini ve geri çağırma izlerini içe aktarır, bunları tekilleştirir
ve aday satırları hazırlar.

- Kısa süreli geri çağırma durumundan, yakın tarihli günlük bellek dosyalarından ve mevcut olduğunda sansürlenmiş oturum transkriptlerinden okur.
- Depolama satır içi çıktı içerdiğinde yönetilen bir `## Light Sleep` bloğu yazar.
- Daha sonraki derin sıralama için pekiştirme sinyalleri kaydeder.
- Asla `MEMORY.md` dosyasına yazmaz.

### Derin evre

Derin evre, neyin uzun vadeli bellek olacağına karar verir.

- Adayları ağırlıklı puanlama ve eşik geçitleri kullanarak sıralar.
- `minScore`, `minRecallCount` ve `minUniqueQueries` değerlerinin geçilmesini gerektirir.
- Yazmadan önce parçaları canlı günlük dosyalardan yeniden yükler; böylece eski/silinmiş parçalar atlanır.
- Yükseltilen girdileri `MEMORY.md` dosyasına ekler.
- `DREAMS.md` içine bir `## Deep Sleep` özeti yazar ve isteğe bağlı olarak `memory/dreaming/deep/YYYY-MM-DD.md` dosyasını yazar.

### REM evresi

REM evresi, örüntüleri ve yansıtıcı sinyalleri çıkarır.

- Yakın tarihli kısa süreli izlerden tema ve yansıma özetleri oluşturur.
- Depolama satır içi çıktı içerdiğinde yönetilen bir `## REM Sleep` bloğu yazar.
- Derin sıralamada kullanılan REM pekiştirme sinyallerini kaydeder.
- Asla `MEMORY.md` dosyasına yazmaz.

## Oturum transkripti içe aktarma

Rüya görme, sansürlenmiş oturum transkriptlerini rüya görme külliyatına içe aktarabilir. Transkriptler
mevcut olduğunda, günlük bellek sinyalleri ve geri çağırma izleriyle birlikte hafif
evreye beslenir. Kişisel ve hassas içerik içe aktarmadan önce sansürlenir.

## Rüya Günlüğü

Rüya görme ayrıca `DREAMS.md` içinde anlatı biçiminde bir **Rüya Günlüğü** tutar.
Her evre yeterli miktarda materyale sahip olduktan sonra, `memory-core` en iyi çabayla arka planda
bir alt ajan dönüşü çalıştırır (varsayılan çalışma zamanı modeli kullanılarak) ve kısa bir günlük girdisi ekler.

Bu günlük, yükseltme kaynağı değil, Dreams UI içinde insanlar tarafından okunmak içindir.

İnceleme ve kurtarma çalışmaları için ayrıca temellendirilmiş bir geçmiş doldurma hattı da vardır:

- `memory rem-harness --path ... --grounded`, geçmiş `YYYY-MM-DD.md` notlarından temellendirilmiş günlük çıktısını önizler.
- `memory rem-backfill --path ...`, geri alınabilir temellendirilmiş günlük girdilerini `DREAMS.md` içine yazar.
- `memory rem-backfill --path ... --stage-short-term`, temellendirilmiş kalıcı adayları, normal derin evrenin zaten kullandığı aynı kısa süreli kanıt deposuna hazırlar.
- `memory rem-backfill --rollback` ve `--rollback-short-term`, bu hazırlanmış geçmiş doldurma yapıtlarını, sıradan günlük girdilerine veya canlı kısa süreli geri çağırmaya dokunmadan kaldırır.

Control UI, aynı günlük geri doldurma/sıfırlama akışını sunar; böylece
temellendirilmiş adayların yükseltmeyi hak edip etmediğine karar vermeden önce sonuçları Dreams sahnesinde
inceleyebilirsiniz. Sahne ayrıca ayrı bir temellendirilmiş hat gösterir; böylece
hangi hazırlanmış kısa süreli girdilerin geçmiş yeniden oynatmadan geldiğini, hangi yükseltilmiş
öğelerin temellendirme odaklı olduğunu görebilir ve yalnızca temellendirilmiş hazırlanmış girdileri
sıradan canlı kısa süreli duruma dokunmadan temizleyebilirsiniz.

## Derin sıralama sinyalleri

Derin sıralama, evre pekiştirmesine ek olarak altı ağırlıklı temel sinyal kullanır:

| Sinyal              | Ağırlık | Açıklama                                          |
| ------------------- | ------ | ------------------------------------------------- |
| Sıklık              | 0.24   | Girdinin biriktirdiği kısa süreli sinyal sayısı   |
| İlgililik           | 0.30   | Girdi için ortalama getirme kalitesi              |
| Sorgu çeşitliliği   | 0.15   | Onu ortaya çıkaran farklı sorgu/gün bağlamları    |
| Yakınlık            | 0.15   | Zamanla azalan tazelik puanı                      |
| Sağlamlaştırma      | 0.10   | Çok günlük tekrar gücü                            |
| Kavramsal zenginlik | 0.06   | Parça/yol kaynağından kavram etiketi yoğunluğu    |

Hafif ve REM evresi isabetleri,
`memory/.dreams/phase-signals.json` içinden zamanla azalan küçük bir ek artış sağlar.

## Zamanlama

Etkinleştirildiğinde `memory-core`, tam bir rüya görme
taraması için tek bir cron işi otomatik olarak yönetir. Her tarama evreleri sırayla çalıştırır: hafif -> REM -> derin.

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

Özel bir tarama sıklığı ile rüya görmeyi etkinleştirin:

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

Elle `memory promote`, CLI bayraklarıyla geçersiz kılınmadığı sürece varsayılan olarak
derin evre eşiklerini kullanır.

Belirli bir adayın neden yükseltileceğini veya yükseltilmeyeceğini açıklayın:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Hiçbir şey yazmadan REM yansımalarını, aday gerçekleri ve derin yükseltme çıktısını
önizleyin:

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

Evre politikası, eşikler ve depolama davranışı iç uygulama
ayrıntılarıdır (kullanıcıya açık yapılandırma değildir).

Tam anahtar listesi için [Bellek yapılandırma başvurusu](/tr/reference/memory-config#dreaming-experimental) sayfasına bakın.

## Dreams UI

Etkinleştirildiğinde Gateway içindeki **Dreams** sekmesi şunları gösterir:

- mevcut rüya görme etkin durumu
- evre düzeyi durum ve yönetilen tarama varlığı
- kısa süreli, temellendirilmiş, sinyal ve bugün yükseltilen sayılarını
- bir sonraki zamanlanmış çalıştırmanın zamanlamasını
- hazırlanmış geçmiş yeniden oynatma girdileri için ayrı bir temellendirilmiş Sahne hattını
- `doctor.memory.dreamDiary` tarafından desteklenen genişletilebilir bir Rüya Günlüğü okuyucusunu

## İlgili

- [Bellek](/tr/concepts/memory)
- [Bellek Arama](/tr/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Bellek yapılandırma başvurusu](/tr/reference/memory-config)
