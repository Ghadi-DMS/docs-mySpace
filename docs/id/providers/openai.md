---
read_when:
    - Anda ingin menggunakan model OpenAI di OpenClaw
    - Anda ingin auth subscription Codex alih-alih API key
summary: Gunakan OpenAI melalui API key atau subscription Codex di OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-06T03:11:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e04db5787f6ed7b1eda04d965c10febae10809fc82ae4d9769e7163234471f5
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI menyediakan API pengembang untuk model GPT. Codex mendukung **login ChatGPT** untuk akses subscription
atau **login API key** untuk akses berbasis penggunaan. Codex cloud memerlukan login ChatGPT.
OpenAI secara eksplisit mendukung penggunaan OAuth subscription di alat/alur kerja eksternal seperti OpenClaw.

## Gaya interaksi default

OpenClaw dapat menambahkan prompt overlay kecil khusus OpenAI untuk run `openai/*` dan
`openai-codex/*`. Secara default, overlay ini menjaga asisten tetap hangat,
kolaboratif, ringkas, langsung, dan sedikit lebih ekspresif secara emosional
tanpa menggantikan system prompt dasar OpenClaw. Overlay yang ramah juga
mengizinkan emoji sesekali bila cocok secara alami, sambil tetap menjaga
output keseluruhan tetap ringkas.

Kunci config:

`plugins.entries.openai.config.personality`

Nilai yang diizinkan:

- `"friendly"`: default; aktifkan overlay khusus OpenAI.
- `"off"`: nonaktifkan overlay dan gunakan hanya prompt dasar OpenClaw.

Cakupan:

- Berlaku untuk model `openai/*`.
- Berlaku untuk model `openai-codex/*`.
- Tidak memengaruhi provider lain.

Perilaku ini aktif secara default. Pertahankan `"friendly"` secara eksplisit jika Anda ingin itu
tetap bertahan terhadap perubahan config lokal di masa mendatang:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### Nonaktifkan prompt overlay OpenAI

Jika Anda menginginkan prompt dasar OpenClaw yang tidak dimodifikasi, tetapkan overlay ke `"off"`:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Anda juga dapat menetapkannya langsung dengan CLI config:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

## Opsi A: API key OpenAI (OpenAI Platform)

**Paling cocok untuk:** akses API langsung dan penagihan berbasis penggunaan.
Dapatkan API key Anda dari dashboard OpenAI.

### Setup CLI

```bash
openclaw onboard --auth-choice openai-api-key
# atau non-interaktif
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Snippet config

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

Dokumentasi model API OpenAI saat ini mencantumkan `gpt-5.4` dan `gpt-5.4-pro` untuk penggunaan
API OpenAI langsung. OpenClaw meneruskan keduanya melalui path `openai/*` Responses.
OpenClaw sengaja menyembunyikan baris `openai/gpt-5.3-codex-spark` yang sudah usang,
karena panggilan API OpenAI langsung menolaknya dalam lalu lintas live.

OpenClaw **tidak** mengekspos `openai/gpt-5.3-codex-spark` pada path API OpenAI langsung.
`pi-ai` masih menyertakan baris bawaan untuk model itu, tetapi permintaan API OpenAI live
saat ini menolaknya. Spark diperlakukan sebagai khusus Codex di OpenClaw.

## Pembuatan image

Plugin `openai` bawaan juga mendaftarkan pembuatan image melalui tool bersama
`image_generate`.

- Model image default: `openai/gpt-image-1`
- Generate: hingga 4 image per permintaan
- Mode edit: diaktifkan, hingga 5 image referensi
- Mendukung `size`
- Peringatan khusus OpenAI saat ini: OpenClaw saat ini tidak meneruskan override `aspectRatio` atau
  `resolution` ke OpenAI Images API

Untuk menggunakan OpenAI sebagai provider image default:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Lihat [Pembuatan Image](/id/tools/image-generation) untuk parameter tool bersama,
pemilihan provider, dan perilaku failover.

## Pembuatan video

Plugin `openai` bawaan juga mendaftarkan pembuatan video melalui tool bersama
`video_generate`.

- Model video default: `openai/sora-2`
- Mode: text-to-video, image-to-video, dan alur referensi/edit video tunggal
- Batas saat ini: 1 input referensi image atau 1 video
- Peringatan khusus OpenAI saat ini: OpenClaw saat ini hanya meneruskan override `size`
  untuk pembuatan video OpenAI native. Override opsional yang tidak didukung
  seperti `aspectRatio`, `resolution`, `audio`, dan `watermark` diabaikan
  dan dilaporkan kembali sebagai peringatan tool.

Untuk menggunakan OpenAI sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Lihat [Pembuatan Video](/tools/video-generation) untuk parameter tool bersama,
pemilihan provider, dan perilaku failover.

## Opsi B: subscription OpenAI Code (Codex)

**Paling cocok untuk:** menggunakan akses subscription ChatGPT/Codex alih-alih API key.
Codex cloud memerlukan login ChatGPT, sedangkan Codex CLI mendukung login ChatGPT atau API key.

### Setup CLI (Codex OAuth)

```bash
# Jalankan Codex OAuth di wizard
openclaw onboard --auth-choice openai-codex

# Atau jalankan OAuth secara langsung
openclaw models auth login --provider openai-codex
```

### Snippet config (subscription Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

Dokumentasi Codex OpenAI saat ini mencantumkan `gpt-5.4` sebagai model Codex saat ini. OpenClaw
memetakannya ke `openai-codex/gpt-5.4` untuk penggunaan OAuth ChatGPT/Codex.

Jika onboarding menggunakan ulang login Codex CLI yang sudah ada, kredensial tersebut tetap
dikelola oleh Codex CLI. Saat kedaluwarsa, OpenClaw membaca ulang sumber Codex eksternal
terlebih dahulu dan, ketika provider dapat me-refresh-nya, menulis kredensial yang diperbarui
kembali ke penyimpanan Codex alih-alih mengambil alih kepemilikannya dalam salinan terpisah
khusus OpenClaw.

Jika akun Codex Anda berhak atas Codex Spark, OpenClaw juga mendukung:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw memperlakukan Codex Spark sebagai khusus Codex. OpenClaw tidak mengekspos path API-key langsung
`openai/gpt-5.3-codex-spark`.

OpenClaw juga mempertahankan `openai-codex/gpt-5.3-codex-spark` saat `pi-ai`
menemukannya. Perlakukan ini sebagai bergantung pada entitlement dan eksperimental: Codex Spark terpisah
dari GPT-5.4 `/fast`, dan ketersediaannya bergantung pada akun Codex /
ChatGPT yang sedang login.

### Batas jendela konteks Codex

OpenClaw memperlakukan metadata model Codex dan batas konteks runtime sebagai dua
nilai yang terpisah.

Untuk `openai-codex/gpt-5.4`:

- `contextWindow` native: `1050000`
- batas `contextTokens` runtime default: `272000`

Ini menjaga metadata model tetap akurat sambil mempertahankan jendela runtime
default yang lebih kecil yang dalam praktiknya memiliki karakteristik latensi dan kualitas yang lebih baik.

Jika Anda menginginkan batas efektif yang berbeda, tetapkan `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Gunakan `contextWindow` hanya saat Anda mendeklarasikan atau meng-override metadata model native.
Gunakan `contextTokens` saat Anda ingin membatasi anggaran konteks runtime.

### Transport default

OpenClaw menggunakan `pi-ai` untuk streaming model. Untuk `openai/*` dan
`openai-codex/*`, transport default adalah `"auto"` (WebSocket terlebih dahulu, lalu fallback
ke SSE).

Dalam mode `"auto"`, OpenClaw juga me-retry satu kegagalan WebSocket awal yang dapat di-retry
sebelum melakukan fallback ke SSE. Mode `"websocket"` yang dipaksakan tetap menampilkan error transport
secara langsung alih-alih menyembunyikannya di balik fallback.

Setelah kegagalan WebSocket pada koneksi atau giliran awal dalam mode `"auto"`, OpenClaw menandai
path WebSocket sesi tersebut sebagai terdegradasi selama sekitar 60 detik dan mengirim
giliran berikutnya melalui SSE selama periode cool-down alih-alih berganti-ganti
antar transport.

Untuk endpoint keluarga OpenAI native (`openai/*`, `openai-codex/*`, dan Azure
OpenAI Responses), OpenClaw juga melampirkan state identitas sesi dan giliran yang stabil
ke permintaan agar retry, reconnect, dan fallback SSE tetap selaras dengan identitas percakapan
yang sama. Pada route keluarga OpenAI native ini, hal tersebut mencakup header identitas permintaan sesi/giliran yang stabil serta metadata transport yang sesuai.

OpenClaw juga menormalkan penghitung penggunaan OpenAI di seluruh varian transport sebelum
mencapai surface session/status. Lalu lintas Responses OpenAI/Codex native dapat
melaporkan penggunaan sebagai `input_tokens` / `output_tokens` atau
`prompt_tokens` / `completion_tokens`; OpenClaw memperlakukan itu sebagai penghitung input
dan output yang sama untuk `/status`, `/usage`, dan log sesi. Saat lalu lintas WebSocket native
menghilangkan `total_tokens` (atau melaporkan `0`), OpenClaw fallback ke total input + output yang
telah dinormalisasi agar tampilan session/status tetap terisi.

Anda dapat menetapkan `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: paksa SSE
- `"websocket"`: paksa WebSocket
- `"auto"`: coba WebSocket, lalu fallback ke SSE

Untuk `openai/*` (Responses API), OpenClaw juga mengaktifkan warm-up WebSocket secara default
(`openaiWsWarmup: true`) saat transport WebSocket digunakan.

Dokumentasi OpenAI terkait:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### Warm-up WebSocket OpenAI

Dokumentasi OpenAI menjelaskan warm-up sebagai opsional. OpenClaw mengaktifkannya secara default untuk
`openai/*` guna mengurangi latensi giliran pertama saat menggunakan transport WebSocket.

### Nonaktifkan warm-up

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Aktifkan warm-up secara eksplisit

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### Pemrosesan prioritas OpenAI dan Codex

API OpenAI mengekspos pemrosesan prioritas melalui `service_tier=priority`. Di
OpenClaw, tetapkan `agents.defaults.models["<provider>/<model>"].params.serviceTier`
untuk meneruskan field tersebut pada endpoint Responses OpenAI/Codex native.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Nilai yang didukung adalah `auto`, `default`, `flex`, dan `priority`.

OpenClaw meneruskan `params.serviceTier` ke permintaan Responses `openai/*`
langsung dan permintaan Codex Responses `openai-codex/*` saat model tersebut menunjuk
ke endpoint OpenAI/Codex native.

Perilaku penting:

- `openai/*` langsung harus menargetkan `api.openai.com`
- `openai-codex/*` harus menargetkan `chatgpt.com/backend-api`
- jika Anda merutekan salah satu provider melalui base URL atau proxy lain, OpenClaw membiarkan `service_tier` tetap apa adanya

### Fast mode OpenAI

OpenClaw mengekspos toggle fast-mode bersama untuk sesi `openai/*` dan
`openai-codex/*`:

- Chat/UI: `/fast status|on|off`
- Config: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Saat fast mode diaktifkan, OpenClaw memetakannya ke pemrosesan prioritas OpenAI:

- panggilan Responses `openai/*` langsung ke `api.openai.com` mengirim `service_tier = "priority"`
- panggilan Responses `openai-codex/*` ke `chatgpt.com/backend-api` juga mengirim `service_tier = "priority"`
- nilai `service_tier` payload yang ada tetap dipertahankan
- fast mode tidak menulis ulang `reasoning` atau `text.verbosity`

Untuk GPT 5.4 secara khusus, setup yang paling umum adalah:

- kirim `/fast on` dalam sesi yang menggunakan `openai/gpt-5.4` atau `openai-codex/gpt-5.4`
- atau tetapkan `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- jika Anda juga menggunakan Codex OAuth, tetapkan `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true` juga

Contoh:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Override sesi menang atas config. Menghapus override sesi di UI Sessions
mengembalikan sesi ke default yang dikonfigurasi.

### Route OpenAI native versus route yang kompatibel dengan OpenAI

OpenClaw memperlakukan endpoint OpenAI, Codex, dan Azure OpenAI langsung secara berbeda
dari proxy `/v1` generik yang kompatibel dengan OpenAI:

- route `openai/*`, `openai-codex/*`, dan Azure OpenAI native mempertahankan
  `reasoning: { effort: "none" }` tetap utuh saat Anda secara eksplisit menonaktifkan reasoning
- route keluarga OpenAI native secara default menggunakan strict mode untuk schema tool
- header atribusi OpenClaw tersembunyi (`originator`, `version`, dan
  `User-Agent`) hanya dilampirkan pada host OpenAI native terverifikasi
  (`api.openai.com`) dan host Codex native (`chatgpt.com/backend-api`)
- route OpenAI/Codex native mempertahankan pembentukan permintaan khusus OpenAI seperti
  `service_tier`, Responses `store`, payload kompatibilitas reasoning OpenAI, dan
  petunjuk prompt-cache
- route gaya proxy yang kompatibel dengan OpenAI mempertahankan perilaku kompatibilitas yang lebih longgar dan tidak
  memaksakan strict tool schemas, pembentukan permintaan khusus native, atau header atribusi
  tersembunyi OpenAI/Codex

Azure OpenAI tetap berada dalam kelompok route native untuk perilaku transport dan kompatibilitas,
tetapi tidak menerima header atribusi tersembunyi OpenAI/Codex.

Ini mempertahankan perilaku OpenAI Responses native saat ini tanpa memaksakan shim
lama yang kompatibel dengan OpenAI ke backend `/v1` pihak ketiga.

### Compaction server-side OpenAI Responses

Untuk model OpenAI Responses langsung (`openai/*` menggunakan `api: "openai-responses"` dengan
`baseUrl` di `api.openai.com`), OpenClaw kini otomatis mengaktifkan petunjuk payload
compaction server-side OpenAI:

- Memaksa `store: true` (kecuali compat model menetapkan `supportsStore: false`)
- Menyuntikkan `context_management: [{ type: "compaction", compact_threshold: ... }]`

Secara default, `compact_threshold` adalah `70%` dari model `contextWindow` (atau `80000`
bila tidak tersedia).

### Aktifkan compaction server-side secara eksplisit

Gunakan ini saat Anda ingin memaksa injeksi `context_management` pada model
Responses yang kompatibel (misalnya Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Aktifkan dengan threshold kustom

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Nonaktifkan compaction server-side

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` hanya mengontrol injeksi `context_management`.
Model OpenAI Responses langsung tetap memaksa `store: true` kecuali compat menetapkan
`supportsStore: false`.

## Catatan

- Ref model selalu menggunakan `provider/model` (lihat [/concepts/models](/id/concepts/models)).
- Detail auth + aturan penggunaan ulang ada di [/concepts/oauth](/id/concepts/oauth).
