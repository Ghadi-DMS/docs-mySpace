---
read_when:
    - Anda ingin menggunakan model MiniMax di OpenClaw
    - Anda memerlukan panduan penyiapan MiniMax
summary: Gunakan model MiniMax di OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-04-06T03:10:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ca35c43cdde53f6f09d9e12d48ce09e4c099cf8cbe1407ac6dbb45b1422507e
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

Provider MiniMax OpenClaw secara default menggunakan **MiniMax M2.7**.

MiniMax juga menyediakan:

- sintesis suara bawaan melalui T2A v2
- pemahaman gambar bawaan melalui `MiniMax-VL-01`
- pembuatan musik bawaan melalui `music-2.5+`
- `web_search` bawaan melalui API pencarian MiniMax Coding Plan

Pembagian provider:

- `minimax`: provider teks berbasis API key, ditambah pembuatan gambar, pemahaman gambar, suara, dan pencarian web bawaan
- `minimax-portal`: provider teks OAuth, ditambah pembuatan gambar dan pemahaman gambar bawaan

## Jajaran model

- `MiniMax-M2.7`: model reasoning terhosting default.
- `MiniMax-M2.7-highspeed`: tingkat reasoning M2.7 yang lebih cepat.
- `image-01`: model pembuatan gambar (generate dan pengeditan image-to-image).

## Pembuatan gambar

Plugin MiniMax mendaftarkan model `image-01` untuk alat `image_generate`. Model ini mendukung:

- **Pembuatan text-to-image** dengan kontrol aspect ratio.
- **Pengeditan image-to-image** (referensi subjek) dengan kontrol aspect ratio.
- Hingga **9 gambar keluaran** per permintaan.
- Hingga **1 gambar referensi** per permintaan edit.
- Aspect ratio yang didukung: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`.

Untuk menggunakan MiniMax untuk pembuatan gambar, setel sebagai provider pembuatan gambar:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

Plugin ini menggunakan `MINIMAX_API_KEY` atau auth OAuth yang sama seperti model teks. Tidak diperlukan konfigurasi tambahan jika MiniMax sudah disiapkan.

Baik `minimax` maupun `minimax-portal` mendaftarkan `image_generate` dengan model
`image-01` yang sama. Penyiapan API key menggunakan `MINIMAX_API_KEY`; penyiapan OAuth dapat menggunakan
jalur auth `minimax-portal` bawaan sebagai gantinya.

Saat onboarding atau penyiapan API key menulis entri `models.providers.minimax`
yang eksplisit, OpenClaw mematerialisasikan `MiniMax-M2.7` dan
`MiniMax-M2.7-highspeed` dengan `input: ["text", "image"]`.

Katalog teks MiniMax bawaan yang terpasang sendiri tetap berupa metadata khusus teks sampai
config provider eksplisit tersebut ada. Pemahaman gambar diekspos secara terpisah
melalui provider media `MiniMax-VL-01` milik plugin.

Lihat [Image Generation](/id/tools/image-generation) untuk parameter alat bersama,
pemilihan provider, dan perilaku failover.

## Pembuatan musik

Plugin `minimax` bawaan juga mendaftarkan pembuatan musik melalui alat bersama
`music_generate`.

- Model musik default: `minimax/music-2.5+`
- Juga mendukung `minimax/music-2.5` dan `minimax/music-2.0`
- Kontrol prompt: `lyrics`, `instrumental`, `durationSeconds`
- Format keluaran: `mp3`
- Proses berbasis sesi dipisahkan melalui alur task/status bersama, termasuk `action: "status"`

Untuk menggunakan MiniMax sebagai provider musik default:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "minimax/music-2.5+",
      },
    },
  },
}
```

Lihat [Music Generation](/tools/music-generation) untuk parameter alat bersama,
pemilihan provider, dan perilaku failover.

## Pembuatan video

Plugin `minimax` bawaan juga mendaftarkan pembuatan video melalui alat bersama
`video_generate`.

- Model video default: `minimax/MiniMax-Hailuo-2.3`
- Mode: text-to-video dan alur referensi satu gambar
- Mendukung `aspectRatio` dan `resolution`

Untuk menggunakan MiniMax sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "minimax/MiniMax-Hailuo-2.3",
      },
    },
  },
}
```

Lihat [Video Generation](/tools/video-generation) untuk parameter alat bersama,
pemilihan provider, dan perilaku failover.

## Pemahaman gambar

Plugin MiniMax mendaftarkan pemahaman gambar secara terpisah dari katalog
teks:

- `minimax`: model gambar default `MiniMax-VL-01`
- `minimax-portal`: model gambar default `MiniMax-VL-01`

Itulah sebabnya perutean media otomatis dapat menggunakan pemahaman gambar MiniMax bahkan
ketika katalog provider teks bawaan masih menampilkan referensi chat M2.7 yang hanya teks.

## Pencarian web

Plugin MiniMax juga mendaftarkan `web_search` melalui API pencarian MiniMax Coding Plan.

- ID provider: `minimax`
- Hasil terstruktur: judul, URL, cuplikan, kueri terkait
- Env var yang direkomendasikan: `MINIMAX_CODE_PLAN_KEY`
- Alias env yang diterima: `MINIMAX_CODING_API_KEY`
- Fallback kompatibilitas: `MINIMAX_API_KEY` ketika sudah mengarah ke token coding-plan
- Penggunaan ulang region: `plugins.entries.minimax.config.webSearch.region`, lalu `MINIMAX_API_HOST`, lalu base URL provider MiniMax
- Pencarian tetap berada pada ID provider `minimax`; penyiapan OAuth CN/global tetap dapat mengarahkan region secara tidak langsung melalui `models.providers.minimax-portal.baseUrl`

Config berada di bawah `plugins.entries.minimax.config.webSearch.*`.
Lihat [MiniMax Search](/id/tools/minimax-search).

## Pilih penyiapan

### OAuth MiniMax (Coding Plan) - direkomendasikan

**Terbaik untuk:** penyiapan cepat dengan MiniMax Coding Plan melalui OAuth, tanpa memerlukan API key.

Lakukan autentikasi dengan pilihan OAuth regional eksplisit:

```bash
openclaw onboard --auth-choice minimax-global-oauth
# or
openclaw onboard --auth-choice minimax-cn-oauth
```

Pemetaan pilihan:

- `minimax-global-oauth`: pengguna internasional (`api.minimax.io`)
- `minimax-cn-oauth`: pengguna di China (`api.minimaxi.com`)

Lihat README paket plugin MiniMax di repo OpenClaw untuk detailnya.

### MiniMax M2.7 (API key)

**Terbaik untuk:** MiniMax terhosting dengan API yang kompatibel Anthropic.

Konfigurasikan melalui CLI:

- Onboarding interaktif:

```bash
openclaw onboard --auth-choice minimax-global-api
# or
openclaw onboard --auth-choice minimax-cn-api
```

- `minimax-global-api`: pengguna internasional (`api.minimax.io`)
- `minimax-cn-api`: pengguna di China (`api.minimaxi.com`)

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
          {
            id: "MiniMax-M2.7-highspeed",
            name: "MiniMax M2.7 Highspeed",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Pada jalur streaming yang kompatibel Anthropic, OpenClaw sekarang menonaktifkan
thinking MiniMax secara default kecuali Anda secara eksplisit menetapkan `thinking` sendiri. Endpoint
streaming MiniMax mengeluarkan `reasoning_content` dalam potongan delta bergaya OpenAI
alih-alih blok thinking Anthropic native, yang dapat membocorkan reasoning internal
ke keluaran yang terlihat jika dibiarkan aktif secara implisit.

### MiniMax M2.7 sebagai fallback (contoh)

**Terbaik untuk:** mempertahankan model generasi terbaru terkuat Anda sebagai primary, lalu fail over ke MiniMax M2.7.
Contoh di bawah menggunakan Opus sebagai primary konkret; ganti dengan model primary generasi terbaru pilihan Anda.

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "primary" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

## Konfigurasi melalui `openclaw configure`

Gunakan wizard config interaktif untuk menyiapkan MiniMax tanpa mengedit JSON:

1. Jalankan `openclaw configure`.
2. Pilih **Model/auth**.
3. Pilih opsi auth **MiniMax**.
4. Pilih model default Anda saat diminta.

Pilihan auth MiniMax saat ini di wizard/CLI:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## Opsi konfigurasi

- `models.providers.minimax.baseUrl`: sebaiknya `https://api.minimax.io/anthropic` (kompatibel Anthropic); `https://api.minimax.io/v1` bersifat opsional untuk payload yang kompatibel OpenAI.
- `models.providers.minimax.api`: sebaiknya `anthropic-messages`; `openai-completions` bersifat opsional untuk payload yang kompatibel OpenAI.
- `models.providers.minimax.apiKey`: API key MiniMax (`MINIMAX_API_KEY`).
- `models.providers.minimax.models`: definisikan `id`, `name`, `reasoning`, `contextWindow`, `maxTokens`, `cost`.
- `agents.defaults.models`: alias model yang Anda inginkan di allowlist.
- `models.mode`: pertahankan `merge` jika Anda ingin menambahkan MiniMax di samping yang bawaan.

## Catatan

- Referensi model mengikuti jalur auth:
  - Penyiapan API key: `minimax/<model>`
  - Penyiapan OAuth: `minimax-portal/<model>`
- Model chat default: `MiniMax-M2.7`
- Model chat alternatif: `MiniMax-M2.7-highspeed`
- Pada `api: "anthropic-messages"`, OpenClaw menyuntikkan
  `thinking: { type: "disabled" }` kecuali thinking sudah ditetapkan secara eksplisit di
  params/config.
- `/fast on` atau `params.fastMode: true` menulis ulang `MiniMax-M2.7` menjadi
  `MiniMax-M2.7-highspeed` pada jalur stream yang kompatibel Anthropic.
- Onboarding dan penyiapan API key langsung menulis definisi model eksplisit dengan
  `input: ["text", "image"]` untuk kedua varian M2.7
- Katalog provider bawaan saat ini mengekspos referensi chat sebagai metadata
  khusus teks sampai config provider MiniMax eksplisit tersedia
- API penggunaan Coding Plan: `https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains` (memerlukan coding plan key).
- OpenClaw menormalkan penggunaan coding-plan MiniMax ke tampilan `% tersisa` yang sama
  seperti provider lain. Field mentah `usage_percent` / `usagePercent` MiniMax adalah
  kuota tersisa, bukan kuota yang telah digunakan, jadi OpenClaw membalikkannya.
  Field berbasis hitungan diutamakan jika ada. Saat API mengembalikan `model_remains`,
  OpenClaw mengutamakan entri model chat, menurunkan label jendela dari
  `start_time` / `end_time` bila diperlukan, dan menyertakan nama model terpilih
  dalam label paket sehingga jendela coding-plan lebih mudah dibedakan.
- Snapshot penggunaan memperlakukan `minimax`, `minimax-cn`, dan `minimax-portal` sebagai
  permukaan kuota MiniMax yang sama, dan lebih mengutamakan OAuth MiniMax yang tersimpan sebelum
  fallback ke env var key Coding Plan.
- Perbarui nilai harga di `models.json` jika Anda memerlukan pelacakan biaya yang akurat.
- Tautan referral untuk MiniMax Coding Plan (diskon 10%): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- Lihat [/concepts/model-providers](/id/concepts/model-providers) untuk aturan provider.
- Gunakan `openclaw models list` untuk mengonfirmasi ID provider saat ini, lalu ganti dengan
  `openclaw models set minimax/MiniMax-M2.7` atau
  `openclaw models set minimax-portal/MiniMax-M2.7`.

## Pemecahan masalah

### "Unknown model: minimax/MiniMax-M2.7"

Ini biasanya berarti **provider MiniMax belum dikonfigurasi** (tidak ada
entri provider yang cocok dan tidak ada profil auth/env key MiniMax yang ditemukan). Perbaikan untuk
deteksi ini ada di **2026.1.12**. Perbaiki dengan:

- Memperbarui ke **2026.1.12** (atau menjalankan dari source `main`), lalu memulai ulang gateway.
- Menjalankan `openclaw configure` dan memilih opsi auth **MiniMax**, atau
- Menambahkan blok `models.providers.minimax` atau
  `models.providers.minimax-portal` yang sesuai secara manual, atau
- Menetapkan `MINIMAX_API_KEY`, `MINIMAX_OAUTH_TOKEN`, atau profil auth MiniMax
  agar provider yang cocok dapat disuntikkan.

Pastikan ID model **peka huruf besar/kecil**:

- Jalur API key: `minimax/MiniMax-M2.7` atau `minimax/MiniMax-M2.7-highspeed`
- Jalur OAuth: `minimax-portal/MiniMax-M2.7` atau
  `minimax-portal/MiniMax-M2.7-highspeed`

Lalu periksa kembali dengan:

```bash
openclaw models list
```
