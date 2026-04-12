---
read_when:
    - Anda ingin menggunakan model OpenAI di OpenClaw
    - Anda ingin autentikasi langganan Codex alih-alih kunci API
    - Anda memerlukan perilaku eksekusi agen GPT-5 yang lebih ketat
summary: Gunakan OpenAI melalui kunci API atau langganan Codex di OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-12T00:18:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7aa06fba9ac901e663685a6b26443a2f6aeb6ec3589d939522dc87cbb43497b4
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI menyediakan API pengembang untuk model GPT. Codex mendukung **masuk dengan ChatGPT** untuk akses berbasis langganan atau **masuk dengan kunci API** untuk akses berbasis penggunaan. Codex cloud memerlukan masuk dengan ChatGPT.
OpenAI secara eksplisit mendukung penggunaan OAuth langganan di alat/alur kerja eksternal seperti OpenClaw.

## Gaya interaksi default

OpenClaw dapat menambahkan overlay prompt kecil khusus OpenAI untuk eksekusi `openai/*` dan
`openai-codex/*`. Secara default, overlay ini menjaga asisten tetap hangat,
kolaboratif, ringkas, langsung, dan sedikit lebih ekspresif secara emosional
tanpa menggantikan prompt sistem dasar OpenClaw. Overlay yang ramah ini juga
mengizinkan emoji sesekali bila terasa alami, sambil tetap menjaga keseluruhan
output tetap ringkas.

Kunci konfigurasi:

`plugins.entries.openai.config.personality`

Nilai yang diizinkan:

- `"friendly"`: default; aktifkan overlay khusus OpenAI.
- `"on"`: alias untuk `"friendly"`.
- `"off"`: nonaktifkan overlay dan gunakan hanya prompt dasar OpenClaw.

Cakupan:

- Berlaku untuk model `openai/*`.
- Berlaku untuk model `openai-codex/*`.
- Tidak memengaruhi provider lain.

Perilaku ini aktif secara default. Tetapkan `"friendly"` secara eksplisit jika Anda ingin
pengaturan ini tetap bertahan dari perubahan konfigurasi lokal di masa mendatang:

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

### Menonaktifkan overlay prompt OpenAI

Jika Anda ingin prompt dasar OpenClaw yang tidak dimodifikasi, atur overlay ke `"off"`:

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

Anda juga dapat mengaturnya langsung dengan CLI konfigurasi:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw menormalkan pengaturan ini secara case-insensitive saat runtime, sehingga nilai seperti
`"Off"` tetap menonaktifkan overlay ramah.

## Opsi A: kunci API OpenAI (OpenAI Platform)

**Terbaik untuk:** akses API langsung dan penagihan berbasis penggunaan.
Dapatkan kunci API Anda dari dashboard OpenAI.

Ringkasan rute:

- `openai/gpt-5.4` = rute API OpenAI Platform langsung
- Memerlukan `OPENAI_API_KEY` (atau konfigurasi provider OpenAI yang setara)
- Di OpenClaw, masuk ChatGPT/Codex dirutekan melalui `openai-codex/*`, bukan `openai/*`

### Penyiapan CLI

```bash
openclaw onboard --auth-choice openai-api-key
# atau non-interaktif
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Potongan konfigurasi

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

Dokumentasi model API OpenAI saat ini mencantumkan `gpt-5.4` dan `gpt-5.4-pro` untuk penggunaan API OpenAI
langsung. OpenClaw meneruskan keduanya melalui jalur Responses `openai/*`.
OpenClaw sengaja menyembunyikan baris lama `openai/gpt-5.3-codex-spark`,
karena panggilan API OpenAI langsung menolaknya pada lalu lintas live.

OpenClaw **tidak** mengekspos `openai/gpt-5.3-codex-spark` pada jalur OpenAI
API langsung. `pi-ai` masih mengirimkan baris bawaan untuk model itu, tetapi permintaan OpenAI API live
saat ini menolaknya. Spark diperlakukan sebagai khusus Codex di OpenClaw.

## Pembuatan gambar

Plugin `openai` bawaan juga mendaftarkan pembuatan gambar melalui
tool `image_generate` bersama.

- Model gambar default: `openai/gpt-image-1`
- Hasilkan: hingga 4 gambar per permintaan
- Mode edit: diaktifkan, hingga 5 gambar referensi
- Mendukung `size`
- Peringatan khusus OpenAI saat ini: OpenClaw tidak meneruskan override `aspectRatio` atau
  `resolution` ke OpenAI Images API saat ini

Untuk menggunakan OpenAI sebagai provider gambar default:

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

Lihat [Pembuatan Gambar](/id/tools/image-generation) untuk parameter
tool bersama, pemilihan provider, dan perilaku failover.

## Pembuatan video

Plugin `openai` bawaan juga mendaftarkan pembuatan video melalui tool
`video_generate` bersama.

- Model video default: `openai/sora-2`
- Mode: text-to-video, image-to-video, dan alur referensi/edit satu video
- Batas saat ini: 1 input referensi gambar atau 1 video
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

Lihat [Pembuatan Video](/id/tools/video-generation) untuk parameter
tool bersama, pemilihan provider, dan perilaku failover.

## Opsi B: langganan OpenAI Code (Codex)

**Terbaik untuk:** menggunakan akses langganan ChatGPT/Codex alih-alih kunci API.
Codex cloud memerlukan masuk dengan ChatGPT, sedangkan CLI Codex mendukung masuk dengan ChatGPT atau kunci API.

Ringkasan rute:

- `openai-codex/gpt-5.4` = rute OAuth ChatGPT/Codex
- Menggunakan masuk ChatGPT/Codex, bukan kunci API OpenAI Platform langsung
- Batas di sisi provider untuk `openai-codex/*` dapat berbeda dari pengalaman web/aplikasi ChatGPT

### Penyiapan CLI (Codex OAuth)

```bash
# Jalankan Codex OAuth di wizard
openclaw onboard --auth-choice openai-codex

# Atau jalankan OAuth secara langsung
openclaw models auth login --provider openai-codex
```

### Potongan konfigurasi (langganan Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

Dokumentasi Codex OpenAI saat ini mencantumkan `gpt-5.4` sebagai model Codex saat ini. OpenClaw
memetakannya ke `openai-codex/gpt-5.4` untuk penggunaan OAuth ChatGPT/Codex.

Rute ini sengaja dipisahkan dari `openai/gpt-5.4`. Jika Anda menginginkan
jalur API OpenAI Platform langsung, gunakan `openai/*` dengan kunci API. Jika Anda menginginkan
masuk ChatGPT/Codex, gunakan `openai-codex/*`.

Jika onboarding menggunakan kembali login CLI Codex yang sudah ada, kredensial tersebut tetap
dikelola oleh CLI Codex. Saat kedaluwarsa, OpenClaw membaca ulang sumber Codex eksternal
terlebih dahulu dan, ketika provider dapat me-refresh-nya, menulis kredensial yang telah diperbarui
kembali ke penyimpanan Codex alih-alih mengambil alih kepemilikan dalam salinan terpisah khusus OpenClaw.

Jika akun Codex Anda berhak atas Codex Spark, OpenClaw juga mendukung:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw memperlakukan Codex Spark sebagai khusus Codex. OpenClaw tidak mengekspos
jalur kunci API `openai/gpt-5.3-codex-spark` langsung.

OpenClaw juga mempertahankan `openai-codex/gpt-5.3-codex-spark` saat `pi-ai`
menemukannya. Perlakukan ini sebagai bergantung pada entitlement dan eksperimental: Codex Spark
terpisah dari GPT-5.4 `/fast`, dan ketersediaannya bergantung pada akun Codex /
ChatGPT yang sedang masuk.

### Batas jendela konteks Codex

OpenClaw memperlakukan metadata model Codex dan batas konteks runtime sebagai nilai yang terpisah.

Untuk `openai-codex/gpt-5.4`:

- `contextWindow` native: `1050000`
- batas `contextTokens` runtime default: `272000`

Ini menjaga metadata model tetap akurat sambil mempertahankan jendela runtime default yang lebih kecil
yang dalam praktiknya memiliki karakteristik latensi dan kualitas yang lebih baik.

Jika Anda menginginkan batas efektif yang berbeda, atur `models.providers.<provider>.models[].contextTokens`:

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

Gunakan `contextWindow` hanya saat Anda mendeklarasikan atau menimpa metadata model
native. Gunakan `contextTokens` saat Anda ingin membatasi anggaran konteks runtime.

### Default transport

OpenClaw menggunakan `pi-ai` untuk streaming model. Untuk `openai/*` dan
`openai-codex/*`, transport default adalah `"auto"` (WebSocket terlebih dahulu, lalu fallback SSE).

Dalam mode `"auto"`, OpenClaw juga mencoba ulang satu kegagalan WebSocket awal yang dapat dicoba ulang
sebelum beralih ke SSE. Mode `"websocket"` yang dipaksakan tetap menampilkan error transport secara langsung
alih-alih menyembunyikannya di balik fallback.

Setelah kegagalan koneksi atau kegagalan WebSocket di awal giliran dalam mode `"auto"`, OpenClaw menandai
jalur WebSocket sesi tersebut sebagai menurun selama sekitar 60 detik dan mengirim
giliran berikutnya melalui SSE selama masa pendinginan alih-alih terus berpindah-pindah
antartransport.

Untuk endpoint keluarga OpenAI native (`openai/*`, `openai-codex/*`, dan Azure
OpenAI Responses), OpenClaw juga melampirkan status identitas sesi dan giliran yang stabil
ke permintaan agar retry, reconnect, dan fallback SSE tetap selaras dengan
identitas percakapan yang sama. Pada rute keluarga OpenAI native ini, hal tersebut mencakup header identitas permintaan sesi/giliran yang stabil serta metadata transport yang sesuai.

OpenClaw juga menormalkan penghitung penggunaan OpenAI di seluruh varian transport sebelum
mencapai permukaan sesi/status. Lalu lintas Responses OpenAI/Codex native dapat
melaporkan penggunaan sebagai `input_tokens` / `output_tokens` atau
`prompt_tokens` / `completion_tokens`; OpenClaw memperlakukan keduanya sebagai penghitung input
dan output yang sama untuk `/status`, `/usage`, dan log sesi. Saat lalu lintas WebSocket native
menghilangkan `total_tokens` (atau melaporkan `0`), OpenClaw menggunakan fallback ke total input + output yang telah dinormalisasi agar tampilan sesi/status tetap terisi.

Anda dapat mengatur `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: paksa SSE
- `"websocket"`: paksa WebSocket
- `"auto"`: coba WebSocket, lalu fallback ke SSE

Untuk `openai/*` (Responses API), OpenClaw juga mengaktifkan warm-up WebSocket secara default
(`openaiWsWarmup: true`) saat transport WebSocket digunakan.

Dokumen OpenAI terkait:

- [Realtime API dengan WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming respons API (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

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

### Menonaktifkan warm-up

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

### Mengaktifkan warm-up secara eksplisit

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
OpenClaw, atur `agents.defaults.models["<provider>/<model>"].params.serviceTier`
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
langsung dan ke permintaan Codex Responses `openai-codex/*` saat model tersebut menunjuk
ke endpoint OpenAI/Codex native.

Perilaku penting:

- `openai/*` langsung harus menargetkan `api.openai.com`
- `openai-codex/*` harus menargetkan `chatgpt.com/backend-api`
- jika Anda merutekan salah satu provider melalui base URL atau proxy lain, OpenClaw membiarkan `service_tier` apa adanya

### Mode cepat OpenAI

OpenClaw mengekspos toggle mode cepat bersama untuk sesi `openai/*` dan
`openai-codex/*`:

- Chat/UI: `/fast status|on|off`
- Konfigurasi: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Saat mode cepat diaktifkan, OpenClaw memetakannya ke pemrosesan prioritas OpenAI:

- panggilan Responses `openai/*` langsung ke `api.openai.com` mengirim `service_tier = "priority"`
- panggilan Responses `openai-codex/*` ke `chatgpt.com/backend-api` juga mengirim `service_tier = "priority"`
- nilai `service_tier` payload yang sudah ada dipertahankan
- mode cepat tidak menulis ulang `reasoning` atau `text.verbosity`

Khusus untuk GPT 5.4, penyiapan yang paling umum adalah:

- kirim `/fast on` dalam sesi yang menggunakan `openai/gpt-5.4` atau `openai-codex/gpt-5.4`
- atau atur `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- jika Anda juga menggunakan OAuth Codex, atur `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true` juga

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

Override sesi lebih diprioritaskan daripada konfigurasi. Menghapus override sesi di UI Sessions
akan mengembalikan sesi ke default yang dikonfigurasi.

### Rute OpenAI native versus OpenAI-compatible

OpenClaw memperlakukan endpoint OpenAI, Codex, dan Azure OpenAI langsung secara berbeda
dari proxy `/v1` OpenAI-compatible generik:

- rute native `openai/*`, `openai-codex/*`, dan Azure OpenAI mempertahankan
  `reasoning: { effort: "none" }` apa adanya saat Anda secara eksplisit menonaktifkan reasoning
- rute keluarga OpenAI native menggunakan mode strict sebagai default untuk schema tool
- header atribusi OpenClaw tersembunyi (`originator`, `version`, dan
  `User-Agent`) hanya dilampirkan pada host OpenAI native terverifikasi
  (`api.openai.com`) dan host Codex native (`chatgpt.com/backend-api`)
- rute OpenAI/Codex native mempertahankan pembentukan permintaan khusus OpenAI seperti
  `service_tier`, Responses `store`, payload kompatibilitas reasoning OpenAI, dan
  petunjuk prompt-cache
- rute OpenAI-compatible bergaya proxy mempertahankan perilaku kompatibilitas yang lebih longgar dan
  tidak memaksakan schema tool strict, pembentukan permintaan khusus native, atau header
  atribusi OpenAI/Codex tersembunyi

Azure OpenAI tetap berada dalam kelompok rute native untuk perilaku transport dan kompatibilitas,
tetapi tidak menerima header atribusi OpenAI/Codex tersembunyi.

Ini mempertahankan perilaku OpenAI Responses native saat ini tanpa memaksakan shim
OpenAI-compatible lama ke backend `/v1` pihak ketiga.

### Mode GPT agentik ketat

Untuk eksekusi GPT keluarga-5 `openai/*` dan `openai-codex/*`, OpenClaw dapat menggunakan
kontrak eksekusi Pi tertanam yang lebih ketat:

```json5
{
  agents: {
    defaults: {
      embeddedPi: {
        executionContract: "strict-agentic",
      },
    },
  },
}
```

Dengan `strict-agentic`, OpenClaw tidak lagi memperlakukan giliran asisten yang hanya berisi rencana sebagai
kemajuan yang berhasil ketika tindakan tool konkret tersedia. OpenClaw akan mencoba ulang
giliran tersebut dengan arahan bertindak sekarang, mengaktifkan otomatis tool `update_plan` terstruktur untuk
pekerjaan yang substansial, dan menampilkan status terblokir yang eksplisit jika model terus
membuat rencana tanpa bertindak.

Mode ini dibatasi untuk eksekusi GPT keluarga-5 OpenAI dan OpenAI Codex. Provider lain
dan keluarga model yang lebih lama tetap menggunakan perilaku Pi tertanam default kecuali Anda memilih
pengaturan runtime lain untuk mereka.

### Kompaksi sisi server OpenAI Responses

Untuk model OpenAI Responses langsung (`openai/*` yang menggunakan `api: "openai-responses"` dengan
`baseUrl` pada `api.openai.com`), OpenClaw sekarang otomatis mengaktifkan petunjuk payload kompaksi sisi server OpenAI:

- Memaksa `store: true` (kecuali kompatibilitas model menetapkan `supportsStore: false`)
- Menyisipkan `context_management: [{ type: "compaction", compact_threshold: ... }]`

Secara default, `compact_threshold` adalah `70%` dari `contextWindow` model (atau `80000`
jika tidak tersedia).

### Mengaktifkan kompaksi sisi server secara eksplisit

Gunakan ini saat Anda ingin memaksa penyisipan `context_management` pada model
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

### Mengaktifkan dengan ambang khusus

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

### Menonaktifkan kompaksi sisi server

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

`responsesServerCompaction` hanya mengontrol penyisipan `context_management`.
Model OpenAI Responses langsung tetap memaksa `store: true` kecuali kompatibilitas menetapkan
`supportsStore: false`.

## Catatan

- Referensi model selalu menggunakan `provider/model` (lihat [/concepts/models](/id/concepts/models)).
- Detail autentikasi + aturan penggunaan ulang ada di [/concepts/oauth](/id/concepts/oauth).
