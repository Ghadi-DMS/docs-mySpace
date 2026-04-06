---
read_when:
    - Membuat video melalui agen
    - Mengonfigurasi provider dan model pembuatan video
    - Memahami parameter tool `video_generate`
summary: Buat video dari teks, gambar, atau video yang sudah ada menggunakan 12 backend provider
title: Pembuatan Video
x-i18n:
    generated_at: "2026-04-06T03:12:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4afec87368232221db1aa5a3980254093d6a961b17271b2dcbf724e6bd455b16
    source_path: tools/video-generation.md
    workflow: 15
---

# Pembuatan Video

Agen OpenClaw dapat membuat video dari prompt teks, gambar referensi, atau video yang sudah ada. Dua belas backend provider didukung, masing-masing dengan opsi model, mode input, dan set fitur yang berbeda. Agen memilih provider yang tepat secara otomatis berdasarkan konfigurasi dan API key yang tersedia.

<Note>
Tool `video_generate` hanya muncul saat setidaknya satu provider pembuatan video tersedia. Jika Anda tidak melihatnya di tool agen Anda, tetapkan API key provider atau konfigurasikan `agents.defaults.videoGenerationModel`.
</Note>

## Memulai cepat

1. Tetapkan API key untuk provider yang didukung mana pun:

```bash
export GEMINI_API_KEY="your-key"
```

2. Secara opsional sematkan model default:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. Minta agen:

> Buat video sinematik berdurasi 5 detik tentang lobster ramah yang sedang berselancar saat matahari terbenam.

Agen memanggil `video_generate` secara otomatis. Tidak diperlukan allowlisting tool.

## Apa yang terjadi saat Anda membuat video

Pembuatan video bersifat asynchronous. Saat agen memanggil `video_generate` dalam sebuah sesi:

1. OpenClaw mengirimkan permintaan ke provider dan langsung mengembalikan ID tugas.
2. Provider memproses pekerjaan di latar belakang (biasanya 30 detik hingga 5 menit tergantung pada provider dan resolusi).
3. Saat video siap, OpenClaw membangunkan sesi yang sama dengan event penyelesaian internal.
4. Agen mengirimkan video yang telah selesai kembali ke percakapan asli.

Saat sebuah pekerjaan sedang berjalan, pemanggilan `video_generate` duplikat dalam sesi yang sama akan mengembalikan status tugas saat ini alih-alih memulai pembuatan lain. Gunakan `openclaw tasks list` atau `openclaw tasks show <taskId>` untuk memeriksa progres dari CLI.

Di luar eksekusi agen berbasis sesi (misalnya, pemanggilan tool langsung), tool ini fallback ke pembuatan inline dan mengembalikan path media final pada giliran yang sama.

## Provider yang didukung

| Provider | Model default                   | Teks | Ref gambar       | Ref video        | API key                                  |
| -------- | ------------------------------- | ---- | ---------------- | ---------------- | ---------------------------------------- |
| Alibaba  | `wan2.6-t2v`                    | Ya   | Ya (URL remote)  | Ya (URL remote)  | `MODELSTUDIO_API_KEY`                    |
| BytePlus | `seedance-1-0-lite-t2v-250428`  | Ya   | 1 gambar         | Tidak            | `BYTEPLUS_API_KEY`                       |
| ComfyUI  | `workflow`                      | Ya   | 1 gambar         | Tidak            | `COMFY_API_KEY` atau `COMFY_CLOUD_API_KEY` |
| fal      | `fal-ai/minimax/video-01-live`  | Ya   | 1 gambar         | Tidak            | `FAL_KEY`                                |
| Google   | `veo-3.1-fast-generate-preview` | Ya   | 1 gambar         | 1 video          | `GEMINI_API_KEY`                         |
| MiniMax  | `MiniMax-Hailuo-2.3`            | Ya   | 1 gambar         | Tidak            | `MINIMAX_API_KEY`                        |
| OpenAI   | `sora-2`                        | Ya   | 1 gambar         | 1 video          | `OPENAI_API_KEY`                         |
| Qwen     | `wan2.6-t2v`                    | Ya   | Ya (URL remote)  | Ya (URL remote)  | `QWEN_API_KEY`                           |
| Runway   | `gen4.5`                        | Ya   | 1 gambar         | 1 video          | `RUNWAYML_API_SECRET`                    |
| Together | `Wan-AI/Wan2.2-T2V-A14B`        | Ya   | 1 gambar         | Tidak            | `TOGETHER_API_KEY`                       |
| Vydra    | `veo3`                          | Ya   | 1 gambar (`kling`) | Tidak          | `VYDRA_API_KEY`                          |
| xAI      | `grok-imagine-video`            | Ya   | 1 gambar         | 1 video          | `XAI_API_KEY`                            |

Beberapa provider menerima env var API key tambahan atau alternatif. Lihat [halaman provider](#related) masing-masing untuk detail.

Jalankan `video_generate action=list` untuk memeriksa provider dan model yang tersedia saat runtime.

## Parameter tool

### Wajib

| Parameter | Tipe   | Deskripsi                                                                     |
| --------- | ------ | ----------------------------------------------------------------------------- |
| `prompt`  | string | Deskripsi teks video yang akan dibuat (wajib untuk `action: "generate"`)      |

### Input konten

| Parameter | Tipe     | Deskripsi                              |
| --------- | -------- | -------------------------------------- |
| `image`   | string   | Gambar referensi tunggal (path atau URL) |
| `images`  | string[] | Beberapa gambar referensi (hingga 5)   |
| `video`   | string   | Video referensi tunggal (path atau URL) |
| `videos`  | string[] | Beberapa video referensi (hingga 4)    |

### Kontrol gaya

| Parameter         | Tipe    | Deskripsi                                                               |
| ----------------- | ------- | ----------------------------------------------------------------------- |
| `aspectRatio`     | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`      | string  | `480P`, `720P`, atau `1080P`                                            |
| `durationSeconds` | number  | Durasi target dalam detik (dibulatkan ke nilai terdekat yang didukung provider) |
| `size`            | string  | Petunjuk ukuran saat provider mendukungnya                              |
| `audio`           | boolean | Aktifkan audio yang dihasilkan saat didukung                            |
| `watermark`       | boolean | Mengaktifkan/menonaktifkan watermark provider saat didukung             |

### Lanjutan

| Parameter  | Tipe   | Deskripsi                                      |
| ---------- | ------ | ---------------------------------------------- |
| `action`   | string | `"generate"` (default), `"status"`, atau `"list"` |
| `model`    | string | Override provider/model (misalnya `runway/gen4.5`) |
| `filename` | string | Petunjuk nama file output                      |

Tidak semua provider mendukung semua parameter. Override yang tidak didukung diabaikan berdasarkan upaya terbaik dan dilaporkan sebagai peringatan dalam hasil tool. Batas kapabilitas yang keras (seperti terlalu banyak input referensi) gagal sebelum pengiriman.

## Actions

- **generate** (default) -- membuat video dari prompt yang diberikan dan input referensi opsional.
- **status** -- memeriksa status tugas video yang sedang berjalan untuk sesi saat ini tanpa memulai pembuatan lain.
- **list** -- menampilkan provider, model, dan kapabilitasnya yang tersedia.

## Pemilihan model

Saat membuat video, OpenClaw menyelesaikan model dalam urutan berikut:

1. **Parameter tool `model`** -- jika agen menentukannya dalam pemanggilan.
2. **`videoGenerationModel.primary`** -- dari konfigurasi.
3. **`videoGenerationModel.fallbacks`** -- dicoba secara berurutan.
4. **Deteksi otomatis** -- menggunakan provider yang memiliki auth valid, dimulai dari provider default saat ini, lalu provider yang tersisa dalam urutan alfabetis.

Jika suatu provider gagal, kandidat berikutnya dicoba secara otomatis. Jika semua kandidat gagal, error akan menyertakan detail dari setiap percobaan.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

## Catatan provider

| Provider | Catatan                                                                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Alibaba  | Menggunakan endpoint asynchronous DashScope/Model Studio. Gambar dan video referensi harus berupa URL `http(s)` remote.                    |
| BytePlus | Hanya satu gambar referensi.                                                                                                               |
| ComfyUI  | Eksekusi lokal atau cloud berbasis workflow. Mendukung text-to-video dan image-to-video melalui graph yang dikonfigurasi.                 |
| fal      | Menggunakan alur berbasis antrean untuk pekerjaan berdurasi panjang. Hanya satu gambar referensi.                                         |
| Google   | Menggunakan Gemini/Veo. Mendukung satu gambar atau satu video referensi.                                                                   |
| MiniMax  | Hanya satu gambar referensi.                                                                                                               |
| OpenAI   | Hanya override `size` yang diteruskan. Override gaya lain (`aspectRatio`, `resolution`, `audio`, `watermark`) diabaikan dengan peringatan. |
| Qwen     | Backend DashScope yang sama dengan Alibaba. Input referensi harus berupa URL `http(s)` remote; file lokal ditolak di awal.                |
| Runway   | Mendukung file lokal melalui data URI. Video-to-video memerlukan `runway/gen4_aleph`. Eksekusi hanya teks mengekspos rasio aspek `16:9` dan `9:16`. |
| Together | Hanya satu gambar referensi.                                                                                                               |
| Vydra    | Menggunakan `https://www.vydra.ai/api/v1` secara langsung untuk menghindari redirect yang menghilangkan auth. `veo3` dibundel hanya sebagai text-to-video; `kling` memerlukan URL gambar remote. |
| xAI      | Mendukung alur text-to-video, image-to-video, dan edit/perpanjang video remote.                                                           |

## Konfigurasi

Tetapkan model pembuatan video default dalam konfigurasi OpenClaw Anda:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Atau melalui CLI:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## Terkait

- [Ikhtisar Tools](/id/tools)
- [Tugas Latar Belakang](/id/automation/tasks) -- pelacakan tugas untuk pembuatan video asynchronous
- [Alibaba Model Studio](/providers/alibaba)
- [BytePlus](/providers/byteplus)
- [ComfyUI](/providers/comfy)
- [fal](/providers/fal)
- [Google (Gemini)](/id/providers/google)
- [MiniMax](/id/providers/minimax)
- [OpenAI](/id/providers/openai)
- [Qwen](/id/providers/qwen)
- [Runway](/providers/runway)
- [Together AI](/id/providers/together)
- [Vydra](/providers/vydra)
- [xAI](/id/providers/xai)
- [Referensi Konfigurasi](/id/gateway/configuration-reference#agent-defaults)
- [Models](/id/concepts/models)
