---
read_when:
    - Anda ingin menyajikan model dari mesin GPU Anda sendiri
    - Anda sedang menghubungkan LM Studio atau proksi yang kompatibel dengan OpenAI
    - Anda memerlukan panduan model lokal yang paling aman
summary: Jalankan OpenClaw pada LLM lokal (LM Studio, vLLM, LiteLLM, endpoint OpenAI kustom)
title: Model Lokal
x-i18n:
    generated_at: "2026-04-15T09:14:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8778cc1c623a356ff3cf306c494c046887f9417a70ec71e659e4a8aae912a780
    source_path: gateway/local-models.md
    workflow: 15
---

# Model lokal

Penggunaan lokal memungkinkan, tetapi OpenClaw mengharapkan konteks besar + pertahanan kuat terhadap injeksi prompt. Kartu kecil akan memangkas konteks dan melemahkan keamanan. Targetkan spesifikasi tinggi: **≥2 Mac Studio dengan konfigurasi maksimal atau rig GPU setara (~$30k+)**. Satu GPU **24 GB** hanya cocok untuk prompt yang lebih ringan dengan latensi lebih tinggi. Gunakan **varian model terbesar / ukuran penuh yang dapat Anda jalankan**; checkpoint yang sangat dikuantisasi atau “kecil” meningkatkan risiko injeksi prompt (lihat [Keamanan](/id/gateway/security)).

Jika Anda menginginkan penyiapan lokal dengan hambatan paling rendah, mulai dengan [LM Studio](/id/providers/lmstudio) atau [Ollama](/id/providers/ollama) dan `openclaw onboard`. Halaman ini adalah panduan beropini untuk stack lokal kelas atas dan server lokal kustom yang kompatibel dengan OpenAI.

## Direkomendasikan: LM Studio + model lokal besar (Responses API)

Stack lokal terbaik saat ini. Muat model besar di LM Studio (misalnya, build Qwen, DeepSeek, atau Llama ukuran penuh), aktifkan server lokal (default `http://127.0.0.1:1234`), dan gunakan Responses API untuk memisahkan penalaran dari teks akhir.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Daftar periksa penyiapan**

- Instal LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- Di LM Studio, unduh **build model terbesar yang tersedia** (hindari varian “small”/yang sangat dikuantisasi), jalankan server, pastikan `http://127.0.0.1:1234/v1/models` menampilkannya.
- Ganti `my-local-model` dengan ID model sebenarnya yang ditampilkan di LM Studio.
- Pastikan model tetap dimuat; cold-load menambah latensi startup.
- Sesuaikan `contextWindow`/`maxTokens` jika build LM Studio Anda berbeda.
- Untuk WhatsApp, tetap gunakan Responses API agar hanya teks akhir yang dikirim.

Tetap konfigurasikan model ter-hosting meskipun berjalan secara lokal; gunakan `models.mode: "merge"` agar fallback tetap tersedia.

### Konfigurasi hibrida: primer ter-hosting, fallback lokal

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Lokal sebagai prioritas dengan jaring pengaman ter-hosting

Tukar urutan primer dan fallback; pertahankan blok provider yang sama dan `models.mode: "merge"` agar Anda dapat fallback ke Sonnet atau Opus saat mesin lokal tidak aktif.

### Hosting regional / perutean data

- Varian MiniMax/Kimi/GLM ter-hosting juga tersedia di OpenRouter dengan endpoint yang dipatok ke wilayah tertentu (misalnya, di-host di AS). Pilih varian regional di sana agar lalu lintas tetap berada dalam yurisdiksi pilihan Anda sambil tetap menggunakan `models.mode: "merge"` untuk fallback Anthropic/OpenAI.
- Hanya lokal tetap menjadi jalur privasi terkuat; perutean regional ter-hosting adalah jalan tengah saat Anda membutuhkan fitur provider tetapi tetap ingin mengendalikan aliran data.

## Proksi lokal lain yang kompatibel dengan OpenAI

vLLM, LiteLLM, OAI-proxy, atau gateway kustom dapat digunakan jika mereka mengekspos endpoint `/v1` bergaya OpenAI. Ganti blok provider di atas dengan endpoint dan ID model Anda:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Pertahankan `models.mode: "merge"` agar model ter-hosting tetap tersedia sebagai fallback.

Catatan perilaku untuk backend `/v1` lokal/proksi:

- OpenClaw memperlakukan ini sebagai rute kompatibel OpenAI bergaya proksi, bukan endpoint OpenAI native
- pembentukan permintaan khusus OpenAI native tidak berlaku di sini: tidak ada `service_tier`, tidak ada Responses `store`, tidak ada pembentukan payload kompatibilitas penalaran OpenAI, dan tidak ada petunjuk cache prompt
- header atribusi OpenClaw tersembunyi (`originator`, `version`, `User-Agent`) tidak disisipkan pada URL proksi kustom ini

Catatan kompatibilitas untuk backend yang kompatibel dengan OpenAI tetapi lebih ketat:

- Beberapa server hanya menerima `messages[].content` berbentuk string pada Chat Completions, bukan array bagian konten terstruktur. Atur `models.providers.<provider>.models[].compat.requiresStringContent: true` untuk endpoint tersebut.
- Beberapa backend lokal yang lebih kecil atau lebih ketat tidak stabil dengan bentuk prompt runtime agen penuh milik OpenClaw, terutama saat skema tool disertakan. Jika backend berfungsi untuk panggilan `/v1/chat/completions` langsung yang kecil tetapi gagal pada giliran agen OpenClaw normal, pertama coba `agents.defaults.localModelMode: "lean"` untuk menghapus tool default yang berat seperti `browser`, `cron`, dan `message`; jika masih gagal, coba `models.providers.<provider>.models[].compat.supportsTools: false`.
- Jika backend masih gagal hanya pada eksekusi OpenClaw yang lebih besar, masalah yang tersisa biasanya adalah kapasitas model/server upstream atau bug backend, bukan lapisan transport OpenClaw.

## Pemecahan masalah

- Gateway dapat menjangkau proksi? `curl http://127.0.0.1:1234/v1/models`.
- Model LM Studio tidak dimuat? Muat ulang; cold start adalah penyebab umum “macet”.
- OpenClaw memperingatkan saat jendela konteks yang terdeteksi berada di bawah **32k** dan memblokir di bawah **16k**. Jika Anda menemui preflight itu, tingkatkan batas konteks server/model atau pilih model yang lebih besar.
- Error konteks? Turunkan `contextWindow` atau naikkan batas server Anda.
- Server yang kompatibel dengan OpenAI mengembalikan `messages[].content ... expected a string`?
  Tambahkan `compat.requiresStringContent: true` pada entri model tersebut.
- Panggilan `/v1/chat/completions` langsung yang kecil berfungsi, tetapi `openclaw infer model run`
  gagal pada Gemma atau model lokal lain? Nonaktifkan skema tool terlebih dahulu dengan
  `compat.supportsTools: false`, lalu uji lagi. Jika server masih crash hanya
  pada prompt OpenClaw yang lebih besar, anggap ini sebagai keterbatasan model/server upstream.
- Keamanan: model lokal melewati filter sisi provider; pertahankan agen tetap sempit dan Compaction aktif untuk membatasi radius dampak injeksi prompt.
