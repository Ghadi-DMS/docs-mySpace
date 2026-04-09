---
read_when:
    - Belleğin nasıl çalıştığını anlamak istiyorsunuz
    - Hangi bellek dosyalarına yazılması gerektiğini bilmek istiyorsunuz
summary: OpenClaw'ın oturumlar arasında şeyleri nasıl hatırladığı
title: Belleğe Genel Bakış
x-i18n:
    generated_at: "2026-04-09T01:27:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2fe47910f5bf1c44be379e971c605f1cb3a29befcf2a7ee11fb3833cbe3b9059
    source_path: concepts/memory.md
    workflow: 15
---

# Belleğe Genel Bakış

OpenClaw, ajanınızın çalışma alanında **düz Markdown dosyaları** yazarak bir şeyleri hatırlar. Model yalnızca diske kaydedilenleri "hatırlar" -- gizli bir durum yoktur.

## Nasıl çalışır

Ajanınızın bellekle ilgili üç dosyası vardır:

- **`MEMORY.md`** -- uzun vadeli bellek. Kalıcı gerçekler, tercihler ve kararlar. Her DM oturumunun başında yüklenir.
- **`memory/YYYY-MM-DD.md`** -- günlük notlar. Süreğen bağlam ve gözlemler. Bugünün ve dünün notları otomatik olarak yüklenir.
- **`DREAMS.md`** (deneysel, isteğe bağlı) -- insan incelemesi için Düş Günlüğü ve rüya taraması özetleri; buna temellendirilmiş geçmişe dönük doldurma girdileri de dahildir.

Bu dosyalar ajan çalışma alanında bulunur (varsayılan `~/.openclaw/workspace`).

<Tip>
Ajanınızın bir şeyi hatırlamasını istiyorsanız, ona söylemeniz yeterlidir: "TypeScript tercih ettiğimi hatırla." Uygun dosyaya bunu yazacaktır.
</Tip>

## Bellek araçları

Ajanın bellekle çalışmak için iki aracı vardır:

- **`memory_search`** -- ifade biçimi asıldan farklı olsa bile, anlamsal aramayı kullanarak ilgili notları bulur.
- **`memory_get`** -- belirli bir bellek dosyasını veya satır aralığını okur.

Her iki araç da etkin bellek plugin'i tarafından sağlanır (varsayılan: `memory-core`).

## Memory Wiki yardımcı plugin'i

Kalıcı belleğin yalnızca ham notlar gibi değil, bakımı yapılan bir bilgi tabanı gibi davranmasını istiyorsanız, birlikte gelen `memory-wiki` plugin'ini kullanın.

`memory-wiki`, kalıcı bilgiyi şu özelliklere sahip bir wiki kasasına derler:

- deterministik sayfa yapısı
- yapılandırılmış iddialar ve kanıtlar
- çelişki ve güncellik takibi
- oluşturulmuş panolar
- ajan/çalışma zamanı tüketicileri için derlenmiş özetler
- `wiki_search`, `wiki_get`, `wiki_apply` ve `wiki_lint` gibi wiki'ye özgü araçlar

Etkin bellek plugin'inin yerini almaz. Geri çağırma, yükseltme ve rüya görme süreçlerinin sahibi olmaya etkin bellek plugin'i devam eder. `memory-wiki`, bunun yanına kaynak geçmişi açısından zengin bir bilgi katmanı ekler.

Bkz. [Memory Wiki](/tr/plugins/memory-wiki).

## Bellek araması

Bir embedding sağlayıcısı yapılandırıldığında, `memory_search` **hibrit arama** kullanır -- vektör benzerliğini (anlamsal anlam) anahtar sözcük eşleştirmeyle (kimlikler ve kod sembolleri gibi tam terimler) birleştirir. Desteklenen herhangi bir sağlayıcı için bir API anahtarınız olduğunda bu ek yapılandırma olmadan çalışır.

<Info>
OpenClaw, mevcut API anahtarlarından embedding sağlayıcınızı otomatik olarak algılar. OpenAI, Gemini, Voyage veya Mistral anahtarınız yapılandırılmışsa bellek araması otomatik olarak etkinleştirilir.
</Info>

Aramanın nasıl çalıştığı, ayarlama seçenekleri ve sağlayıcı kurulumu hakkında ayrıntılar için bkz. [Memory Search](/tr/concepts/memory-search).

## Bellek arka uçları

<CardGroup cols={3}>
<Card title="Yerleşik (varsayılan)" icon="database" href="/tr/concepts/memory-builtin">
SQLite tabanlıdır. Anahtar sözcük araması, vektör benzerliği ve hibrit arama ile ek yapılandırma olmadan çalışır. Ek bağımlılık yoktur.
</Card>
<Card title="QMD" icon="search" href="/tr/concepts/memory-qmd">
Yeniden sıralama, sorgu genişletme ve çalışma alanı dışındaki dizinleri indeksleme yeteneğine sahip yerel öncelikli yan hizmet.
</Card>
<Card title="Honcho" icon="brain" href="/tr/concepts/memory-honcho">
Kullanıcı modelleme, anlamsal arama ve çoklu ajan farkındalığı ile AI-native oturumlar arası bellek. Plugin kurulumu gerekir.
</Card>
</CardGroup>

## Bilgi wiki katmanı

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/tr/plugins/memory-wiki">
Kalıcı belleği iddialar, panolar, köprü modu ve Obsidian dostu iş akışlarıyla kaynak geçmişi açısından zengin bir wiki kasasına derler.
</Card>
</CardGroup>

## Otomatik bellek boşaltma

[Compaction](/tr/concepts/compaction) konuşmanızı özetlemeden önce OpenClaw, ajana önemli bağlamı bellek dosyalarına kaydetmesini hatırlatan sessiz bir tur çalıştırır. Bu varsayılan olarak açıktır -- herhangi bir şeyi yapılandırmanız gerekmez.

<Tip>
Bellek boşaltma, compaction sırasında bağlam kaybını önler. Ajanınızın konuşmada bulunan ancak henüz bir dosyaya yazılmamış önemli bilgileri varsa, özetleme gerçekleşmeden önce bunlar otomatik olarak kaydedilir.
</Tip>

## Rüya görme (deneysel)

Rüya görme, bellek için isteğe bağlı bir arka plan pekiştirme geçişidir. Kısa vadeli sinyalleri toplar, adayları puanlar ve yalnızca uygun öğeleri uzun vadeli belleğe (`MEMORY.md`) yükseltir.

Uzun vadeli belleği yüksek sinyalli tutacak şekilde tasarlanmıştır:

- **İsteğe bağlı**: varsayılan olarak devre dışıdır.
- **Zamanlanmış**: etkinleştirildiğinde `memory-core`, tam bir rüya taraması için yinelenen tek bir cron işini otomatik olarak yönetir.
- **Eşikli**: yükseltmeler puan, geri çağırma sıklığı ve sorgu çeşitliliği geçitlerini geçmelidir.
- **İncelenebilir**: aşama özetleri ve günlük girdileri, insan incelemesi için `DREAMS.md` dosyasına yazılır.

Aşama davranışı, puanlama sinyalleri ve Düş Günlüğü ayrıntıları için bkz. [Dreaming (experimental)](/tr/concepts/dreaming).

## Temellendirilmiş geçmişe dönük doldurma ve canlı yükseltme

Rüya görme sisteminin artık birbirine yakın iki inceleme hattı vardır:

- **Canlı rüya görme**, `memory/.dreams/` altındaki kısa vadeli rüya görme deposundan çalışır ve bir şeyin `MEMORY.md` içine yükselip yükselemeyeceğine karar verirken normal derin aşamanın kullandığı şeydir.
- **Temellendirilmiş geçmişe dönük doldurma**, geçmiş `memory/YYYY-MM-DD.md` notlarını bağımsız günlük dosyaları olarak okur ve yapılandırılmış inceleme çıktısını `DREAMS.md` dosyasına yazar.

Temellendirilmiş geçmişe dönük doldurma, eski notları yeniden oynatmak ve sistemin `MEMORY.md` dosyasını elle düzenlemeden neyin kalıcı olduğunu düşündüğünü incelemek istediğinizde kullanışlıdır.

Şunu kullandığınızda:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

temellendirilmiş kalıcı adaylar doğrudan yükseltilmez. Bunun yerine, normal derin aşamanın zaten kullandığı aynı kısa vadeli rüya görme deposuna hazırlanırlar. Bu şu anlama gelir:

- `DREAMS.md`, insan inceleme yüzeyi olarak kalır.
- kısa vadeli depo, makineye dönük sıralama yüzeyi olarak kalır.
- `MEMORY.md` hâlâ yalnızca derin yükseltme tarafından yazılır.

Yeniden oynatmanın yararlı olmadığına karar verirseniz, hazırlanan yapıtları sıradan günlük girdilerine veya normal geri çağırma durumuna dokunmadan kaldırabilirsiniz:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Dizin durumunu ve sağlayıcıyı kontrol et
openclaw memory search "query"  # Komut satırından ara
openclaw memory index --force   # Dizini yeniden oluştur
```

## Daha fazla okuma

- [Builtin Memory Engine](/tr/concepts/memory-builtin) -- varsayılan SQLite arka ucu
- [QMD Memory Engine](/tr/concepts/memory-qmd) -- gelişmiş yerel öncelikli yan hizmet
- [Honcho Memory](/tr/concepts/memory-honcho) -- AI-native oturumlar arası bellek
- [Memory Wiki](/tr/plugins/memory-wiki) -- derlenmiş bilgi kasası ve wiki'ye özgü araçlar
- [Memory Search](/tr/concepts/memory-search) -- arama hattı, sağlayıcılar ve ayarlama
- [Dreaming (experimental)](/tr/concepts/dreaming) -- kısa vadeli geri çağırmadan uzun vadeli belleğe arka plan yükseltmesi
- [Memory configuration reference](/tr/reference/memory-config) -- tüm yapılandırma ayarları
- [Compaction](/tr/concepts/compaction) -- compaction'ın bellekle nasıl etkileştiği
