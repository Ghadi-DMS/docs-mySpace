---
read_when:
    - qa-lab veya qa-channel genişletilirken
    - depo destekli QA senaryoları eklenirken
    - Gateway panosu etrafında daha yüksek gerçekçiliğe sahip QA otomasyonu oluşturulurken
summary: qa-lab, qa-channel, tohumlanmış senaryolar ve protokol raporları için özel QA otomasyon yapısı
title: QA Uçtan Uca Otomasyonu
x-i18n:
    generated_at: "2026-04-09T01:27:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: c922607d67e0f3a2489ac82bc9f510f7294ced039c1014c15b676d826441d833
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA Uçtan Uca Otomasyonu

Özel QA yığını, OpenClaw'ı tek birim testinin yapabileceğinden daha gerçekçi,
kanal biçimli bir şekilde çalıştırmayı amaçlar.

Mevcut parçalar:

- `extensions/qa-channel`: DM, kanal, ileti dizisi,
  tepki, düzenleme ve silme yüzeylerine sahip sentetik mesaj kanalı.
- `extensions/qa-lab`: dökümü gözlemlemek,
  gelen mesajları enjekte etmek ve bir Markdown raporu dışa aktarmak için hata ayıklayıcı arayüzü ve QA veri yolu.
- `qa/`: başlangıç görevi ve temel QA
  senaryoları için depo destekli tohum varlıkları.

Mevcut QA operatörü akışı iki bölmeli bir QA sitesidir:

- Sol: aracıyla birlikte Gateway panosu (Control UI).
- Sağ: Slack benzeri dökümü ve senaryo planını gösteren QA Lab.

Şununla çalıştırın:

```bash
pnpm qa:lab:up
```

Bu, QA sitesini derler, Docker destekli gateway hattını başlatır ve
bir operatörün veya otomasyon döngüsünün aracıya bir QA
görevi verebileceği, gerçek kanal davranışını gözlemleyebileceği ve nelerin işe yaradığını, başarısız olduğunu veya
engel olarak kaldığını kaydedebileceği QA Lab sayfasını açar.

Docker imajını her seferinde yeniden derlemeden daha hızlı QA Lab arayüzü yinelemesi için,
yığını bağlamalı bir QA Lab paketiyle başlatın:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast`, Docker servislerini önceden derlenmiş bir imaj üzerinde tutar ve
`extensions/qa-lab/web/dist` dizinini `qa-lab` container'ına bağlar. `qa:lab:watch`
bu paketi değişikliklerde yeniden derler ve QA Lab
varlık karması değiştiğinde tarayıcı otomatik olarak yeniden yüklenir.

## Depo destekli tohumlar

Tohum varlıkları `qa/` içinde bulunur:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Bunlar kasıtlı olarak git içindedir; böylece QA planı hem insanlar hem de
aracı tarafından görünür olur. Temel liste, şunları kapsayacak kadar geniş kalmalıdır:

- DM ve kanal sohbeti
- ileti dizisi davranışı
- mesaj eylemi yaşam döngüsü
- cron geri çağrıları
- bellekten geri çağırma
- model değiştirme
- alt aracı devri
- depoyu okuma ve belgeleri okuma
- Lobster Invaders gibi küçük bir derleme görevi

## Raporlama

`qa-lab`, gözlemlenen veri yolu zaman çizelgesinden bir Markdown protokol raporu dışa aktarır.
Rapor şunlara yanıt vermelidir:

- Neler işe yaradı
- Neler başarısız oldu
- Neler engelli kaldı
- Hangi takip senaryolarını eklemeye değer olduğu

Karakter ve stil kontrolleri için, aynı senaryoyu birden çok canlı model
referansı üzerinde çalıştırın ve değerlendirilmiş bir Markdown raporu yazın:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

Komut Docker değil, yerel QA gateway alt süreçleri çalıştırır. Karakter değerlendirme
senaryoları kişiliği `SOUL.md` aracılığıyla ayarlamalı, ardından sohbet, çalışma alanı yardımı ve küçük dosya görevleri gibi
sıradan kullanıcı dönüşlerini çalıştırmalıdır. Aday modele
değerlendirildiği söylenmemelidir. Komut her tam
dökümü korur, temel çalıştırma istatistiklerini kaydeder, ardından yargıç modellerden hızlı modda
`xhigh` akıl yürütme ile çalıştırmaları doğallık, hava ve mizaha göre sıralamalarını ister.
Sağlayıcıları karşılaştırırken `--blind-judge-models` kullanın: yargıç istemi yine de
her dökümü ve çalıştırma durumunu alır, ancak aday referansları
`candidate-01` gibi nötr etiketlerle değiştirilir; rapor sıralamaları ayrıştırmadan sonra gerçek referanslara geri eşler.
Aday çalıştırmaları varsayılan olarak `high` düşünme kullanır; bunu destekleyen OpenAI modelleri için `xhigh` kullanılır.
Belirli bir adayı satır içinde
`--model provider/model,thinking=<level>` ile geçersiz kılın. `--thinking <level>` yine de genel bir geri dönüş değeri ayarlar ve eski `--model-thinking <provider/model=level>` biçimi
uyumluluk için korunur.
OpenAI aday referansları varsayılan olarak hızlı mod kullanır; böylece sağlayıcının desteklediği yerlerde öncelikli işleme kullanılır. Tek bir
aday veya yargıç için geçersiz kılma gerektiğinde satır içinde `,fast`, `,no-fast` veya `,fast=false` ekleyin. Yalnızca
her aday model için hızlı modu zorla açmak istediğinizde `--fast` geçin. Aday ve yargıç süreleri
kıyaslama analizi için rapora kaydedilir, ancak yargıç istemleri açıkça
hıza göre sıralama yapılmamasını söyler.
Aday ve yargıç model çalıştırmalarının ikisi de varsayılan olarak 16 eşzamanlılık kullanır.
Sağlayıcı sınırları veya yerel gateway
yükü bir çalıştırmayı fazla gürültülü hale getiriyorsa `--concurrency` veya `--judge-concurrency` değerini düşürün.
Hiç aday `--model` geçirilmediğinde, karakter değerlendirmesi varsayılan olarak
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` ve
`google/gemini-3.1-pro-preview` kullanır.
Hiç `--judge-model` geçirilmediğinde, yargıçlar varsayılan olarak
`openai/gpt-5.4,thinking=xhigh,fast` ve
`anthropic/claude-opus-4-6,thinking=high` olur.

## İlgili belgeler

- [Testing](/tr/help/testing)
- [QA Channel](/tr/channels/qa-channel)
- [Dashboard](/web/dashboard)
