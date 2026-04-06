---
read_when:
    - Anda ingin menggunakan model Grok di OpenClaw
    - Anda sedang mengonfigurasi auth xAI atau ID model
summary: Gunakan model Grok xAI di OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-06T03:10:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 64bc899655427cc10bdc759171c7d1ec25ad9f1e4f9d803f1553d3d586c6d71d
    source_path: providers/xai.md
    workflow: 15
---

# xAI

OpenClaw dikirim dengan plugin provider `xai` bawaan untuk model Grok.

## Penyiapan

1. Buat API key di konsol xAI.
2. Set `XAI_API_KEY`, atau jalankan:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. Pilih model seperti:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

OpenClaw sekarang menggunakan xAI Responses API sebagai transport xAI bawaan. `XAI_API_KEY`
yang sama juga dapat digunakan untuk `web_search` berbasis Grok, `x_search` kelas satu,
dan `code_execution` jarak jauh.
Jika Anda menyimpan key xAI di bawah `plugins.entries.xai.config.webSearch.apiKey`,
provider model xAI bawaan sekarang juga menggunakan kembali key tersebut sebagai fallback.
Penyetelan `code_execution` berada di bawah `plugins.entries.xai.config.codeExecution`.

## Katalog model bawaan saat ini

OpenClaw sekarang menyertakan keluarga model xAI berikut secara bawaan:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

Plugin ini juga melakukan forward-resolve untuk ID `grok-4*` dan `grok-code-fast*` yang lebih baru saat
mengikuti bentuk API yang sama.

Catatan model cepat:

- `grok-4-fast`, `grok-4-1-fast`, dan varian `grok-4.20-beta-*` adalah
  referensi Grok berkemampuan gambar saat ini dalam katalog bawaan.
- `/fast on` atau `agents.defaults.models["xai/<model>"].params.fastMode: true`
  menulis ulang request xAI native sebagai berikut:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

Alias kompatibilitas lama masih dinormalkan ke ID bawaan kanonis. Sebagai
contoh:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## Pencarian web

Provider pencarian web `grok` bawaan juga menggunakan `XAI_API_KEY`:

```bash
openclaw config set tools.web.search.provider grok
```

## Pembuatan video

Plugin `xai` bawaan juga mendaftarkan pembuatan video melalui
tool `video_generate` bersama.

- Model video default: `xai/grok-imagine-video`
- Mode: text-to-video, image-to-video, dan alur edit/perpanjang video jarak jauh
- Mendukung `aspectRatio` dan `resolution`
- Batas saat ini: buffer video lokal tidak diterima; gunakan URL `http(s)`
  jarak jauh untuk input referensi/edit video

Untuk menggunakan xAI sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "xai/grok-imagine-video",
      },
    },
  },
}
```

Lihat [Pembuatan Video](/tools/video-generation) untuk parameter
tool bersama, pemilihan provider, dan perilaku failover.

## Batasan yang diketahui

- Auth saat ini hanya API key. Belum ada alur OAuth/device-code xAI di OpenClaw.
- `grok-4.20-multi-agent-experimental-beta-0304` tidak didukung pada jalur provider xAI normal karena memerlukan permukaan API upstream yang berbeda dari transport xAI OpenClaw standar.

## Catatan

- OpenClaw menerapkan perbaikan kompatibilitas skema tool dan tool-call khusus xAI secara otomatis pada jalur runner bersama.
- Request xAI native menggunakan default `tool_stream: true`. Set
  `agents.defaults.models["xai/<model>"].params.tool_stream` ke `false` untuk
  menonaktifkannya.
- Wrapper xAI bawaan menghapus flag strict tool-schema dan
  key payload reasoning yang tidak didukung sebelum mengirim request xAI native.
- `web_search`, `x_search`, dan `code_execution` diekspos sebagai tool OpenClaw. OpenClaw mengaktifkan built-in xAI spesifik yang dibutuhkannya di dalam setiap request tool alih-alih melampirkan semua tool native ke setiap giliran chat.
- `x_search` dan `code_execution` dimiliki oleh plugin xAI bawaan, bukan di-hardcode ke runtime model inti.
- `code_execution` adalah eksekusi sandbox xAI jarak jauh, bukan [`exec`](/id/tools/exec) lokal.
- Untuk ikhtisar provider yang lebih luas, lihat [Provider model](/id/providers/index).
