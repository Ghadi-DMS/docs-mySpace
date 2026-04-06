---
read_when:
    - Membuat musik atau audio melalui agen
    - Mengonfigurasi provider dan model pembuatan musik
    - Memahami parameter tool `music_generate`
summary: Buat musik dengan provider bersama, termasuk plugin berbasis workflow
title: Pembuatan Musik
x-i18n:
    generated_at: "2026-04-06T03:12:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: a03de8aa75cfb7248eb0c1d969fb2a6da06117967d097e6f6e95771d0f017ae1
    source_path: tools/music-generation.md
    workflow: 15
---

# Pembuatan Musik

Tool `music_generate` memungkinkan agen membuat musik atau audio melalui
kapabilitas pembuatan musik bersama dengan provider yang dikonfigurasi seperti Google,
MiniMax, dan ComfyUI yang dikonfigurasi melalui workflow.

Untuk sesi agen berbasis provider bersama, OpenClaw memulai pembuatan musik sebagai
tugas latar belakang, melacaknya dalam ledger tugas, lalu membangunkan agen lagi saat
trek siap sehingga agen dapat mengirim audio yang sudah selesai kembali ke
channel asal.

<Note>
Tool bersama bawaan hanya muncul ketika setidaknya satu provider pembuatan musik tersedia. Jika Anda tidak melihat `music_generate` di tool agen Anda, konfigurasikan `agents.defaults.musicGenerationModel` atau siapkan API key provider.
</Note>

## Memulai cepat

### Pembuatan berbasis provider bersama

1. Tetapkan API key untuk setidaknya satu provider, misalnya `GEMINI_API_KEY` atau
   `MINIMAX_API_KEY`.
2. Secara opsional tetapkan model pilihan Anda:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

3. Minta agen: _"Generate an upbeat synthpop track about a night drive
   through a neon city."_

Agen memanggil `music_generate` secara otomatis. Tidak perlu allow-listing tool.

Untuk konteks sinkron langsung tanpa eksekusi agen berbasis sesi, tool bawaan
tetap fallback ke pembuatan inline dan mengembalikan path media final dalam
hasil tool.

Contoh prompt:

```text
Generate a cinematic piano track with soft strings and no vocals.
```

```text
Generate an energetic chiptune loop about launching a rocket at sunrise.
```

### Pembuatan Comfy berbasis workflow

Plugin `comfy` bawaan terhubung ke tool `music_generate` bersama melalui
registry provider pembuatan musik.

1. Konfigurasikan `models.providers.comfy.music` dengan workflow JSON dan
   node prompt/output.
2. Jika Anda menggunakan Comfy Cloud, set `COMFY_API_KEY` atau `COMFY_CLOUD_API_KEY`.
3. Minta agen membuat musik atau panggil tool secara langsung.

Contoh:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

## Dukungan provider bawaan bersama

| Provider | Model default          | Input referensi  | Kontrol yang didukung                                   | API key                                |
| -------- | ---------------------- | ---------------- | ------------------------------------------------------- | -------------------------------------- |
| ComfyUI  | `workflow`             | Hingga 1 gambar  | Musik atau audio yang ditentukan workflow               | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google   | `lyria-3-clip-preview` | Hingga 10 gambar | `lyrics`, `instrumental`, `format`                      | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax  | `music-2.5+`           | Tidak ada        | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY`                    |

Gunakan `action: "list"` untuk memeriksa provider dan model bersama yang tersedia saat
runtime:

```text
/tool music_generate action=list
```

Gunakan `action: "status"` untuk memeriksa tugas musik aktif berbasis sesi:

```text
/tool music_generate action=status
```

Contoh pembuatan langsung:

```text
/tool music_generate prompt="Dreamy lo-fi hip hop with vinyl texture and gentle rain" instrumental=true
```

## Parameter tool bawaan

| Parameter         | Tipe     | Deskripsi                                                                                       |
| ----------------- | -------- | ------------------------------------------------------------------------------------------------ |
| `prompt`          | string   | Prompt pembuatan musik (wajib untuk `action: "generate"`)                                        |
| `action`          | string   | `"generate"` (default), `"status"` untuk tugas sesi saat ini, atau `"list"` untuk memeriksa provider |
| `model`           | string   | Override provider/model, misalnya `google/lyria-3-pro-preview` atau `comfy/workflow`             |
| `lyrics`          | string   | Lirik opsional saat provider mendukung input lirik eksplisit                                     |
| `instrumental`    | boolean  | Meminta output instrumental saja saat provider mendukungnya                                      |
| `image`           | string   | Path atau URL gambar referensi tunggal                                                           |
| `images`          | string[] | Beberapa gambar referensi (hingga 10)                                                            |
| `durationSeconds` | number   | Durasi target dalam detik saat provider mendukung petunjuk durasi                                |
| `format`          | string   | Petunjuk format output (`mp3` atau `wav`) saat provider mendukungnya                             |
| `filename`        | string   | Petunjuk nama file output                                                                        |

Tidak semua provider mendukung semua parameter. OpenClaw tetap memvalidasi batas keras
seperti jumlah input sebelum pengiriman, tetapi petunjuk opsional yang tidak didukung akan
diabaikan dengan peringatan saat provider atau model yang dipilih tidak dapat memenuhinya.

## Perilaku async untuk jalur berbasis provider bersama

- Eksekusi agen berbasis sesi: `music_generate` membuat tugas latar belakang, segera mengembalikan respons started/task, dan mengirim trek yang sudah selesai nanti dalam pesan agen tindak lanjut.
- Pencegahan duplikasi: selama tugas latar belakang tersebut masih `queued` atau `running`, pemanggilan `music_generate` berikutnya dalam sesi yang sama akan mengembalikan status tugas alih-alih memulai pembuatan lain.
- Pencarian status: gunakan `action: "status"` untuk memeriksa tugas musik aktif berbasis sesi tanpa memulai yang baru.
- Pelacakan tugas: gunakan `openclaw tasks list` atau `openclaw tasks show <taskId>` untuk memeriksa status queued, running, dan terminal untuk pembuatan tersebut.
- Bangun saat selesai: OpenClaw menyuntikkan event penyelesaian internal kembali ke sesi yang sama sehingga model dapat menulis sendiri tindak lanjut yang ditujukan kepada pengguna.
- Petunjuk prompt: giliran pengguna/manual berikutnya dalam sesi yang sama mendapat petunjuk runtime kecil saat tugas musik sudah berjalan sehingga model tidak secara membabi buta memanggil `music_generate` lagi.
- Fallback tanpa sesi: konteks langsung/lokal tanpa sesi agen nyata tetap berjalan inline dan mengembalikan hasil audio final pada giliran yang sama.

## Konfigurasi

### Pemilihan model

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["minimax/music-2.5+"],
      },
    },
  },
}
```

### Urutan pemilihan provider

Saat membuat musik, OpenClaw mencoba provider dalam urutan berikut:

1. Parameter `model` dari pemanggilan tool, jika agen menentukannya
2. `musicGenerationModel.primary` dari konfigurasi
3. `musicGenerationModel.fallbacks` secara berurutan
4. Deteksi otomatis yang hanya menggunakan default provider yang didukung auth:
   - provider default saat ini terlebih dahulu
   - provider pembuatan musik terdaftar yang tersisa dalam urutan id provider

Jika suatu provider gagal, kandidat berikutnya dicoba secara otomatis. Jika semuanya gagal,
error akan menyertakan detail dari setiap percobaan.

## Catatan provider

- Google menggunakan pembuatan batch Lyria 3. Alur bawaan saat ini mendukung
  prompt, teks lirik opsional, dan gambar referensi opsional.
- MiniMax menggunakan endpoint batch `music_generation`. Alur bawaan saat ini
  mendukung prompt, lirik opsional, mode instrumental, pengaturan durasi, dan
  output mp3.
- Dukungan ComfyUI berbasis workflow dan bergantung pada graph yang dikonfigurasi plus
  pemetaan node untuk field prompt/output.

## Memilih jalur yang tepat

- Gunakan jalur berbasis provider bersama saat Anda menginginkan pemilihan model, failover provider, dan alur tugas/status async bawaan.
- Gunakan jalur plugin seperti ComfyUI saat Anda memerlukan graph workflow kustom atau provider yang bukan bagian dari kapabilitas musik bawaan bersama.
- Jika Anda sedang men-debug perilaku spesifik ComfyUI, lihat [ComfyUI](/providers/comfy). Jika Anda sedang men-debug perilaku provider bersama, mulai dari [Google (Gemini)](/id/providers/google) atau [MiniMax](/id/providers/minimax).

## Live test

Cakupan live opsional untuk provider bawaan bersama:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Cakupan live opsional untuk jalur musik ComfyUI bawaan:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

File live Comfy juga mencakup workflow gambar dan video comfy saat bagian tersebut
dikonfigurasi.

## Terkait

- [Tugas Latar Belakang](/id/automation/tasks) - pelacakan tugas untuk eksekusi `music_generate` terlepas
- [Referensi Konfigurasi](/id/gateway/configuration-reference#agent-defaults) - konfigurasi `musicGenerationModel`
- [ComfyUI](/providers/comfy)
- [Google (Gemini)](/id/providers/google)
- [MiniMax](/id/providers/minimax)
- [Models](/id/concepts/models) - konfigurasi model dan failover
- [Ikhtisar Tools](/id/tools)
