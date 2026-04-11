---
read_when:
    - Membuat video melalui agen
    - Mengonfigurasi penyedia dan model pembuatan video
    - Memahami parameter alat `video_generate`
summary: Buat video dari teks, gambar, atau video yang sudah ada menggunakan 14 backend penyedia
title: Pembuatan Video
x-i18n:
    generated_at: "2026-04-11T15:15:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0ec159a0bbb6b8a030e68828c0a8bcaf40c8538ecf98bc8ff609dab9d0068263
    source_path: tools/video-generation.md
    workflow: 15
---

# Pembuatan Video

Agen OpenClaw dapat membuat video dari prompt teks, gambar referensi, atau video yang sudah ada. Empat belas backend penyedia didukung, masing-masing dengan opsi model, mode input, dan kumpulan fitur yang berbeda. Agen memilih penyedia yang tepat secara otomatis berdasarkan konfigurasi Anda dan API key yang tersedia.

<Note>
Alat `video_generate` hanya muncul ketika setidaknya satu penyedia pembuatan video tersedia. Jika Anda tidak melihatnya di alat agen Anda, atur API key penyedia atau konfigurasikan `agents.defaults.videoGenerationModel`.
</Note>

OpenClaw memperlakukan pembuatan video sebagai tiga mode runtime:

- `generate` untuk permintaan teks-ke-video tanpa media referensi
- `imageToVideo` ketika permintaan menyertakan satu atau lebih gambar referensi
- `videoToVideo` ketika permintaan menyertakan satu atau lebih video referensi

Penyedia dapat mendukung subset apa pun dari mode tersebut. Alat ini memvalidasi mode aktif sebelum pengiriman dan melaporkan mode yang didukung dalam `action=list`.

## Mulai cepat

1. Atur API key untuk penyedia yang didukung:

```bash
export GEMINI_API_KEY="your-key"
```

2. Secara opsional, tetapkan model default:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. Minta agen:

> Buat video sinematik berdurasi 5 detik tentang seekor lobster ramah yang berselancar saat matahari terbenam.

Agen memanggil `video_generate` secara otomatis. Tidak perlu allowlist alat.

## Apa yang terjadi saat Anda membuat video

Pembuatan video bersifat asinkron. Saat agen memanggil `video_generate` dalam sebuah sesi:

1. OpenClaw mengirim permintaan ke penyedia dan langsung mengembalikan ID tugas.
2. Penyedia memproses pekerjaan di latar belakang (biasanya 30 detik hingga 5 menit tergantung penyedia dan resolusi).
3. Saat video siap, OpenClaw membangunkan sesi yang sama dengan peristiwa penyelesaian internal.
4. Agen memposting video yang sudah selesai kembali ke percakapan asli.

Saat sebuah pekerjaan sedang berjalan, panggilan `video_generate` duplikat dalam sesi yang sama mengembalikan status tugas saat ini alih-alih memulai pembuatan baru. Gunakan `openclaw tasks list` atau `openclaw tasks show <taskId>` untuk memeriksa progres dari CLI.

Di luar eksekusi agen berbasis sesi (misalnya, pemanggilan alat langsung), alat ini kembali ke pembuatan inline dan mengembalikan path media final pada giliran yang sama.

### Siklus hidup tugas

Setiap permintaan `video_generate` bergerak melalui empat status:

1. **queued** -- tugas dibuat, menunggu penyedia menerimanya.
2. **running** -- penyedia sedang memproses (biasanya 30 detik hingga 5 menit tergantung penyedia dan resolusi).
3. **succeeded** -- video siap; agen bangun dan mempostingnya ke percakapan.
4. **failed** -- kesalahan penyedia atau timeout; agen bangun dengan detail kesalahan.

Periksa status dari CLI:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

Pencegahan duplikasi: jika tugas video sudah `queued` atau `running` untuk sesi saat ini, `video_generate` mengembalikan status tugas yang ada alih-alih memulai tugas baru. Gunakan `action: "status"` untuk memeriksa secara eksplisit tanpa memicu pembuatan baru.

## Penyedia yang didukung

| Penyedia              | Model default                   | Teks | Referensi gambar                                     | Referensi video | API key                                  |
| --------------------- | ------------------------------- | ---- | ---------------------------------------------------- | ---------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    | Ya   | Ya (URL jarak jauh)                                  | Ya (URL jarak jauh) | `MODELSTUDIO_API_KEY`                 |
| BytePlus (1.0)        | `seedance-1-0-pro-250528`       | Ya   | Hingga 2 gambar (hanya model I2V; frame pertama + terakhir) | Tidak      | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 1.5 | `seedance-1-5-pro-251215`       | Ya   | Hingga 2 gambar (frame pertama + terakhir melalui peran) | Tidak        | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  | Ya   | Hingga 9 gambar referensi                            | Hingga 3 video   | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      | Ya   | 1 gambar                                             | Tidak            | `COMFY_API_KEY` atau `COMFY_CLOUD_API_KEY` |
| fal                   | `fal-ai/minimax/video-01-live`  | Ya   | 1 gambar                                             | Tidak            | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` | Ya   | 1 gambar                                             | 1 video          | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            | Ya   | 1 gambar                                             | Tidak            | `MINIMAX_API_KEY`                        |
| OpenAI                | `sora-2`                        | Ya   | 1 gambar                                             | 1 video          | `OPENAI_API_KEY`                         |
| Qwen                  | `wan2.6-t2v`                    | Ya   | Ya (URL jarak jauh)                                  | Ya (URL jarak jauh) | `QWEN_API_KEY`                        |
| Runway                | `gen4.5`                        | Ya   | 1 gambar                                             | 1 video          | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        | Ya   | 1 gambar                                             | Tidak            | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          | Ya   | 1 gambar (`kling`)                                   | Tidak            | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            | Ya   | 1 gambar                                             | 1 video          | `XAI_API_KEY`                            |

Beberapa penyedia menerima env var API key tambahan atau alternatif. Lihat [halaman penyedia](#related) masing-masing untuk detail.

Jalankan `video_generate action=list` untuk memeriksa penyedia, model, dan
mode runtime yang tersedia saat runtime.

### Matriks kapabilitas yang dideklarasikan

Ini adalah kontrak mode eksplisit yang digunakan oleh `video_generate`, uji kontrak,
dan shared live sweep.

| Penyedia | `generate` | `imageToVideo` | `videoToVideo` | Shared live lane saat ini                                                                                                                |
| -------- | ---------- | -------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba  | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; `videoToVideo` dilewati karena penyedia ini memerlukan URL video `http(s)` jarak jauh                        |
| BytePlus | Ya         | Ya             | Tidak          | `generate`, `imageToVideo`                                                                                                               |
| ComfyUI  | Ya         | Ya             | Tidak          | Tidak ada dalam shared sweep; cakupan khusus workflow ada bersama pengujian Comfy                                                        |
| fal      | Ya         | Ya             | Tidak          | `generate`, `imageToVideo`                                                                                                               |
| Google   | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; shared `videoToVideo` dilewati karena sweep Gemini/Veo berbasis buffer saat ini tidak menerima input itu     |
| MiniMax  | Ya         | Ya             | Tidak          | `generate`, `imageToVideo`                                                                                                               |
| OpenAI   | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; shared `videoToVideo` dilewati karena jalur org/input ini saat ini memerlukan akses inpaint/remix sisi penyedia |
| Qwen     | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; `videoToVideo` dilewati karena penyedia ini memerlukan URL video `http(s)` jarak jauh                        |
| Runway   | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; `videoToVideo` hanya berjalan ketika model yang dipilih adalah `runway/gen4_aleph`                           |
| Together | Ya         | Ya             | Tidak          | `generate`, `imageToVideo`                                                                                                               |
| Vydra    | Ya         | Ya             | Tidak          | `generate`; shared `imageToVideo` dilewati karena `veo3` bawaan hanya teks dan `kling` bawaan memerlukan URL gambar jarak jauh          |
| xAI      | Ya         | Ya             | Ya             | `generate`, `imageToVideo`; `videoToVideo` dilewati karena penyedia ini saat ini memerlukan URL MP4 jarak jauh                           |

## Parameter alat

### Wajib

| Parameter | Tipe   | Deskripsi                                                                        |
| --------- | ------ | -------------------------------------------------------------------------------- |
| `prompt`  | string | Deskripsi teks dari video yang akan dibuat (wajib untuk `action: "generate"`)    |

### Input konten

| Parameter    | Tipe     | Deskripsi                                                                                                                              |
| ------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `image`      | string   | Gambar referensi tunggal (path atau URL)                                                                                               |
| `images`     | string[] | Beberapa gambar referensi (hingga 9)                                                                                                   |
| `imageRoles` | string[] | Petunjuk peran opsional per posisi yang paralel dengan daftar gambar gabungan. Nilai kanonis: `first_frame`, `last_frame`, `reference_image` |
| `video`      | string   | Video referensi tunggal (path atau URL)                                                                                                |
| `videos`     | string[] | Beberapa video referensi (hingga 4)                                                                                                    |
| `videoRoles` | string[] | Petunjuk peran opsional per posisi yang paralel dengan daftar video gabungan. Nilai kanonis: `reference_video`                        |
| `audioRef`   | string   | Audio referensi tunggal (path atau URL). Digunakan misalnya untuk musik latar atau referensi suara saat penyedia mendukung input audio |
| `audioRefs`  | string[] | Beberapa audio referensi (hingga 3)                                                                                                    |
| `audioRoles` | string[] | Petunjuk peran opsional per posisi yang paralel dengan daftar audio gabungan. Nilai kanonis: `reference_audio`                       |

Petunjuk peran diteruskan ke penyedia apa adanya. Nilai kanonis berasal dari
union `VideoGenerationAssetRole`, tetapi penyedia mungkin menerima string
peran tambahan. Array `*Roles` tidak boleh memiliki lebih banyak entri daripada
daftar referensi yang sesuai; kesalahan off-by-one gagal dengan pesan kesalahan yang jelas.
Gunakan string kosong untuk membiarkan slot tidak diatur.

### Kontrol gaya

| Parameter         | Tipe   | Deskripsi                                                                                 |
| ----------------- | ------ | ----------------------------------------------------------------------------------------- |
| `aspectRatio`     | string | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, atau `adaptive` |
| `resolution`      | string | `480P`, `720P`, `768P`, atau `1080P`                                                      |
| `durationSeconds` | number | Durasi target dalam detik (dibulatkan ke nilai terdekat yang didukung penyedia)          |
| `size`            | string | Petunjuk ukuran saat penyedia mendukungnya                                                |
| `audio`           | boolean | Aktifkan audio yang dihasilkan dalam output saat didukung. Berbeda dari `audioRef*` (input) |
| `watermark`       | boolean | Aktifkan/nonaktifkan watermark penyedia saat didukung                                    |

`adaptive` adalah sentinel khusus penyedia: nilai ini diteruskan apa adanya ke
penyedia yang mendeklarasikan `adaptive` dalam kapabilitasnya (misalnya BytePlus
Seedance menggunakannya untuk mendeteksi rasio secara otomatis dari dimensi
gambar input). Penyedia yang tidak mendeklarasikannya menampilkan nilai tersebut melalui
`details.ignoredOverrides` dalam hasil alat agar pengabaian itu terlihat.

### Lanjutan

| Parameter         | Tipe   | Deskripsi                                                                                                                                                                                                                                                                                                                                           |
| ----------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action`          | string | `"generate"` (default), `"status"`, atau `"list"`                                                                                                                                                                                                                                                                                                   |
| `model`           | string | Override penyedia/model (misalnya `runway/gen4.5`)                                                                                                                                                                                                                                                                                                  |
| `filename`        | string | Petunjuk nama file output                                                                                                                                                                                                                                                                                                                            |
| `providerOptions` | object | Opsi khusus penyedia sebagai objek JSON (misalnya `{"seed": 42, "draft": true}`). Penyedia yang mendeklarasikan skema bertipe memvalidasi kunci dan tipenya; kunci yang tidak dikenal atau ketidaksesuaian tipe akan melewati kandidat selama fallback. Penyedia tanpa skema yang dideklarasikan menerima opsi apa adanya. Jalankan `video_generate action=list` untuk melihat apa yang diterima masing-masing penyedia |

Tidak semua penyedia mendukung semua parameter. OpenClaw sudah menormalkan durasi ke nilai terdekat yang didukung penyedia, dan juga memetakan ulang petunjuk geometri yang diterjemahkan seperti ukuran-ke-rasio-aspek ketika penyedia fallback mengekspos permukaan kontrol yang berbeda. Override yang benar-benar tidak didukung diabaikan dengan upaya terbaik dan dilaporkan sebagai peringatan dalam hasil alat. Batas kapabilitas yang keras (seperti terlalu banyak input referensi) gagal sebelum pengiriman.

Hasil alat melaporkan pengaturan yang diterapkan. Saat OpenClaw memetakan ulang durasi atau geometri selama fallback penyedia, nilai `durationSeconds`, `size`, `aspectRatio`, dan `resolution` yang dikembalikan mencerminkan apa yang dikirimkan, dan `details.normalization` mencatat terjemahan dari yang diminta ke yang diterapkan.

Input referensi juga memilih mode runtime:

- Tidak ada media referensi: `generate`
- Referensi gambar apa pun: `imageToVideo`
- Referensi video apa pun: `videoToVideo`
- Input audio referensi tidak mengubah mode yang ditentukan; input tersebut diterapkan di atas mode apa pun yang dipilih oleh referensi gambar/video, dan hanya berfungsi dengan penyedia yang mendeklarasikan `maxInputAudios`

Referensi gambar dan video campuran bukanlah permukaan kapabilitas bersama yang stabil.
Sebaiknya gunakan satu jenis referensi per permintaan.

#### Fallback dan opsi bertipe

Beberapa pemeriksaan kapabilitas diterapkan di lapisan fallback, bukan di batas
alat, sehingga permintaan yang melampaui batas penyedia utama
tetap dapat berjalan pada fallback yang mampu:

- Jika kandidat aktif tidak mendeklarasikan `maxInputAudios` (atau mendeklarasikannya sebagai
  `0`), kandidat itu dilewati saat permintaan berisi referensi audio, dan
  kandidat berikutnya dicoba.
- Jika `maxDurationSeconds` kandidat aktif berada di bawah
  `durationSeconds` yang diminta dan kandidat tersebut tidak mendeklarasikan daftar
  `supportedDurationSeconds`, kandidat itu dilewati.
- Jika permintaan berisi `providerOptions` dan kandidat aktif
  secara eksplisit mendeklarasikan skema `providerOptions` bertipe, kandidat tersebut
  dilewati ketika kunci yang diberikan tidak ada dalam skema atau tipe nilainya
  tidak cocok. Penyedia yang belum mendeklarasikan skema menerima
  opsi apa adanya (pass-through yang kompatibel ke belakang). Penyedia dapat
  secara eksplisit menolak semua opsi penyedia dengan mendeklarasikan skema kosong
  (`capabilities.providerOptions: {}`), yang menyebabkan pelewatan yang sama seperti
  ketidakcocokan tipe.

Alasan pelewatan pertama dalam sebuah permintaan dicatat pada `warn` agar operator melihat
kapan penyedia utama mereka dilewati; pelewatan berikutnya dicatat pada
`debug` agar rantai fallback panjang tetap tenang. Jika setiap kandidat dilewati,
kesalahan gabungan mencakup alasan pelewatan untuk masing-masing kandidat.

## Aksi

- **generate** (default) -- buat video dari prompt yang diberikan dan input referensi opsional.
- **status** -- periksa status tugas video yang sedang berjalan untuk sesi saat ini tanpa memulai pembuatan lain.
- **list** -- tampilkan penyedia, model, dan kapabilitasnya yang tersedia.

## Pemilihan model

Saat membuat video, OpenClaw menentukan model dalam urutan ini:

1. **Parameter alat `model`** -- jika agen menentukannya dalam pemanggilan.
2. **`videoGenerationModel.primary`** -- dari konfigurasi.
3. **`videoGenerationModel.fallbacks`** -- dicoba sesuai urutan.
4. **Deteksi otomatis** -- menggunakan penyedia yang memiliki autentikasi valid, dimulai dari penyedia default saat ini, lalu penyedia lainnya dalam urutan alfabet.

Jika penyedia gagal, kandidat berikutnya dicoba secara otomatis. Jika semua kandidat gagal, kesalahan mencakup detail dari setiap percobaan.

Atur `agents.defaults.mediaGenerationAutoProviderFallback: false` jika Anda ingin
pembuatan video hanya menggunakan entri `model`, `primary`, dan `fallbacks`
yang eksplisit.

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

## Catatan penyedia

| Penyedia              | Catatan                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba               | Menggunakan endpoint asinkron DashScope/Model Studio. Gambar dan video referensi harus berupa URL `http(s)` jarak jauh.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| BytePlus (1.0)        | ID penyedia `byteplus`. Model: `seedance-1-0-pro-250528` (default), `seedance-1-0-pro-t2v-250528`, `seedance-1-0-pro-fast-251015`, `seedance-1-0-lite-t2v-250428`, `seedance-1-0-lite-i2v-250428`. Model T2V (`*-t2v-*`) tidak menerima input gambar; model I2V dan model umum `*-pro-*` mendukung satu gambar referensi (frame pertama). Berikan gambar secara posisional atau tetapkan `role: "first_frame"`. ID model T2V secara otomatis dialihkan ke varian I2V yang sesuai ketika gambar diberikan. Kunci `providerOptions` yang didukung: `seed` (number), `draft` (boolean, memaksa 480p), `camera_fixed` (boolean).                                                                            |
| BytePlus Seedance 1.5 | Memerlukan plugin [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark). ID penyedia `byteplus-seedance15`. Model: `seedance-1-5-pro-251215`. Menggunakan API `content[]` terpadu. Mendukung maksimal 2 gambar input (first_frame + last_frame). Semua input harus berupa URL `https://` jarak jauh. Tetapkan `role: "first_frame"` / `"last_frame"` pada setiap gambar, atau berikan gambar secara posisional. `aspectRatio: "adaptive"` mendeteksi rasio secara otomatis dari gambar input. `audio: true` dipetakan ke `generate_audio`. `providerOptions.seed` (number) diteruskan.                                                                                                                     |
| BytePlus Seedance 2.0 | Memerlukan plugin [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark). ID penyedia `byteplus-seedance2`. Model: `dreamina-seedance-2-0-260128`, `dreamina-seedance-2-0-fast-260128`. Menggunakan API `content[]` terpadu. Mendukung hingga 9 gambar referensi, 3 video referensi, dan 3 audio referensi. Semua input harus berupa URL `https://` jarak jauh. Tetapkan `role` pada setiap aset — nilai yang didukung: `"first_frame"`, `"last_frame"`, `"reference_image"`, `"reference_video"`, `"reference_audio"`. `aspectRatio: "adaptive"` mendeteksi rasio secara otomatis dari gambar input. `audio: true` dipetakan ke `generate_audio`. `providerOptions.seed` (number) diteruskan. |
| ComfyUI               | Eksekusi lokal atau cloud berbasis workflow. Mendukung teks-ke-video dan gambar-ke-video melalui grafik yang dikonfigurasi.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| fal                   | Menggunakan alur berbasis antrean untuk pekerjaan berdurasi panjang. Hanya satu gambar referensi.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Google                | Menggunakan Gemini/Veo. Mendukung satu gambar atau satu video referensi.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| MiniMax               | Hanya satu gambar referensi.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| OpenAI                | Hanya override `size` yang diteruskan. Override gaya lainnya (`aspectRatio`, `resolution`, `audio`, `watermark`) diabaikan dengan peringatan.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Qwen                  | Backend DashScope yang sama dengan Alibaba. Input referensi harus berupa URL `http(s)` jarak jauh; file lokal ditolak sejak awal.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Runway                | Mendukung file lokal melalui URI data. Video-ke-video memerlukan `runway/gen4_aleph`. Eksekusi hanya teks mengekspos rasio aspek `16:9` dan `9:16`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Together              | Hanya satu gambar referensi.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Vydra                 | Menggunakan `https://www.vydra.ai/api/v1` secara langsung untuk menghindari pengalihan yang menghapus autentikasi. `veo3` dibundel hanya sebagai teks-ke-video; `kling` memerlukan URL gambar jarak jauh.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| xAI                   | Mendukung teks-ke-video, gambar-ke-video, dan alur edit/perluas video jarak jauh.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

## Mode kapabilitas penyedia

Kontrak pembuatan video bersama kini memungkinkan penyedia mendeklarasikan
kapabilitas khusus mode alih-alih hanya batas agregat datar. Implementasi
penyedia baru sebaiknya menggunakan blok mode yang eksplisit:

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

Bidang agregat datar seperti `maxInputImages` dan `maxInputVideos` tidak
cukup untuk mengiklankan dukungan mode transformasi. Penyedia harus mendeklarasikan
`generate`, `imageToVideo`, dan `videoToVideo` secara eksplisit agar pengujian live,
uji kontrak, dan alat bersama `video_generate` dapat memvalidasi dukungan mode
secara deterministik.

## Pengujian live

Cakupan live opt-in untuk penyedia bundel bersama:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

Wrapper repo:

```bash
pnpm test:live:media video
```

File live ini memuat env var penyedia yang belum ada dari `~/.profile`, memprioritaskan
API key live/env dibanding profil autentikasi yang tersimpan secara default, dan menjalankan
mode yang dideklarasikan yang dapat diuji dengan aman menggunakan media lokal:

- `generate` untuk setiap penyedia dalam sweep
- `imageToVideo` ketika `capabilities.imageToVideo.enabled`
- `videoToVideo` ketika `capabilities.videoToVideo.enabled` dan penyedia/model
  menerima input video lokal berbasis buffer dalam shared sweep

Saat ini shared live lane `videoToVideo` mencakup:

- `runway` hanya saat Anda memilih `runway/gen4_aleph`

## Konfigurasi

Tetapkan model pembuatan video default di konfigurasi OpenClaw Anda:

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

- [Ikhtisar Alat](/id/tools)
- [Tugas Latar Belakang](/id/automation/tasks) -- pelacakan tugas untuk pembuatan video asinkron
- [Alibaba Model Studio](/id/providers/alibaba)
- [BytePlus](/id/concepts/model-providers#byteplus-international)
- [ComfyUI](/id/providers/comfy)
- [fal](/id/providers/fal)
- [Google (Gemini)](/id/providers/google)
- [MiniMax](/id/providers/minimax)
- [OpenAI](/id/providers/openai)
- [Qwen](/id/providers/qwen)
- [Runway](/id/providers/runway)
- [Together AI](/id/providers/together)
- [Vydra](/id/providers/vydra)
- [xAI](/id/providers/xai)
- [Referensi Konfigurasi](/id/gateway/configuration-reference#agent-defaults)
- [Model](/id/concepts/models)
