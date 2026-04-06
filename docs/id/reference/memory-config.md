---
read_when:
    - Anda ingin mengonfigurasi provider pencarian memory atau model embedding
    - Anda ingin menyiapkan backend QMD
    - Anda ingin menyetel pencarian hybrid, MMR, atau temporal decay
    - Anda ingin mengaktifkan pengindeksan memory multimodal
summary: Semua pengaturan konfigurasi untuk pencarian memory, provider embedding, QMD, pencarian hybrid, dan pengindeksan multimodal
title: Referensi konfigurasi memory
x-i18n:
    generated_at: "2026-04-06T03:11:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0de0b85125443584f4e575cf673ca8d9bd12ecd849d73c537f4a17545afa93fd
    source_path: reference/memory-config.md
    workflow: 15
---

# Referensi konfigurasi memory

Halaman ini mencantumkan setiap pengaturan konfigurasi untuk pencarian memory OpenClaw. Untuk
ikhtisar konseptual, lihat:

- [Ikhtisar Memory](/id/concepts/memory) -- cara kerja memory
- [Builtin Engine](/id/concepts/memory-builtin) -- backend SQLite default
- [QMD Engine](/id/concepts/memory-qmd) -- sidecar local-first
- [Memory Search](/id/concepts/memory-search) -- pipeline pencarian dan penyetelan

Semua pengaturan pencarian memory berada di bawah `agents.defaults.memorySearch` di
`openclaw.json` kecuali jika disebutkan lain.

---

## Pemilihan provider

| Key        | Type      | Default          | Deskripsi                                                                                     |
| ---------- | --------- | ---------------- | --------------------------------------------------------------------------------------------- |
| `provider` | `string`  | terdeteksi otomatis | ID adapter embedding: `openai`, `gemini`, `voyage`, `mistral`, `bedrock`, `ollama`, `local` |
| `model`    | `string`  | default provider | Nama model embedding                                                                          |
| `fallback` | `string`  | `"none"`         | ID adapter fallback saat yang utama gagal                                                     |
| `enabled`  | `boolean` | `true`           | Mengaktifkan atau menonaktifkan pencarian memory                                              |

### Urutan deteksi otomatis

Saat `provider` tidak diatur, OpenClaw memilih yang pertama tersedia:

1. `local` -- jika `memorySearch.local.modelPath` dikonfigurasi dan file ada.
2. `openai` -- jika key OpenAI dapat diresolusikan.
3. `gemini` -- jika key Gemini dapat diresolusikan.
4. `voyage` -- jika key Voyage dapat diresolusikan.
5. `mistral` -- jika key Mistral dapat diresolusikan.
6. `bedrock` -- jika rantai kredensial AWS SDK dapat diresolusikan (instance role, access key, profile, SSO, web identity, atau shared config).

`ollama` didukung tetapi tidak terdeteksi otomatis (atur secara eksplisit).

### Resolusi API key

Embedding remote memerlukan API key. Bedrock menggunakan rantai kredensial default AWS SDK
sebagai gantinya (instance role, SSO, access key).

| Provider | Env var                        | Key config                        |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Bedrock  | rantai kredensial AWS          | Tidak memerlukan API key          |
| Ollama   | `OLLAMA_API_KEY` (placeholder) | --                                |

OAuth Codex hanya mencakup chat/completions dan tidak memenuhi permintaan
embedding.

---

## Konfigurasi endpoint remote

Untuk endpoint kustom yang kompatibel dengan OpenAI atau meng-override default provider:

| Key              | Type     | Deskripsi                                          |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | URL dasar API kustom                               |
| `remote.apiKey`  | `string` | Override API key                                   |
| `remote.headers` | `object` | Header HTTP tambahan (digabungkan dengan default provider) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Konfigurasi khusus Gemini

| Key                    | Type     | Default                | Deskripsi                                 |
| ---------------------- | -------- | ---------------------- | ----------------------------------------- |
| `model`                | `string` | `gemini-embedding-001` | Juga mendukung `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | Untuk Embedding 2: 768, 1536, atau 3072   |

<Warning>
Mengubah model atau `outputDimensionality` memicu reindex penuh otomatis.
</Warning>

---

## Konfigurasi embedding Bedrock

Bedrock menggunakan rantai kredensial default AWS SDK -- tidak memerlukan API key.
Jika OpenClaw berjalan di EC2 dengan instance role yang mengaktifkan Bedrock, cukup atur
provider dan model:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| Key                    | Type     | Default                        | Deskripsi                      |
| ---------------------- | -------- | ------------------------------ | ------------------------------ |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | ID model embedding Bedrock apa pun |
| `outputDimensionality` | `number` | default model                  | Untuk Titan V2: 256, 512, atau 1024 |

### Model yang didukung

Model berikut didukung (dengan deteksi family dan default
dimension):

| Model ID                                   | Provider   | Dims Default | Dims yang Dapat Dikonfigurasi |
| ------------------------------------------ | ---------- | ------------ | ----------------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024               |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                           |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                           |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                           |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072         |
| `cohere.embed-english-v3`                  | Cohere     | 1024         | --                           |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                           |
| `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536                     |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                           |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                           |

Varian dengan sufiks throughput (misalnya, `amazon.titan-embed-text-v1:2:8k`) mewarisi
konfigurasi model dasar.

### Autentikasi

Auth Bedrock menggunakan urutan resolusi kredensial AWS SDK standar:

1. Environment variable (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. Cache token SSO
3. Kredensial token web identity
4. File shared credentials dan config
5. Kredensial metadata ECS atau EC2

Region diresolusikan dari `AWS_REGION`, `AWS_DEFAULT_REGION`, `baseUrl` provider
`amazon-bedrock`, atau default ke `us-east-1`.

### Izin IAM

Role atau pengguna IAM memerlukan:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

Untuk hak istimewa minimum, cakup `InvokeModel` ke model tertentu:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## Konfigurasi embedding lokal

| Key                   | Type     | Default                | Deskripsi                    |
| --------------------- | -------- | ---------------------- | ---------------------------- |
| `local.modelPath`     | `string` | terunduh otomatis      | Path ke file model GGUF      |
| `local.modelCacheDir` | `string` | default node-llama-cpp | Direktori cache untuk model yang diunduh |

Model default: `embeddinggemma-300m-qat-Q8_0.gguf` (~0,6 GB, terunduh otomatis).
Memerlukan build native: `pnpm approve-builds` lalu `pnpm rebuild node-llama-cpp`.

---

## Konfigurasi pencarian hybrid

Semua di bawah `memorySearch.query.hybrid`:

| Key                   | Type      | Default | Deskripsi                          |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | Aktifkan pencarian hybrid BM25 + vector |
| `vectorWeight`        | `number`  | `0.7`   | Bobot untuk skor vector (0-1)      |
| `textWeight`          | `number`  | `0.3`   | Bobot untuk skor BM25 (0-1)        |
| `candidateMultiplier` | `number`  | `4`     | Pengali ukuran kumpulan kandidat   |

### MMR (keragaman)

| Key           | Type      | Default | Deskripsi                               |
| ------------- | --------- | ------- | --------------------------------------- |
| `mmr.enabled` | `boolean` | `false` | Aktifkan re-ranking MMR                 |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = keragaman maksimum, 1 = relevansi maksimum |

### Temporal decay (kekinian)

| Key                          | Type      | Default | Deskripsi                    |
| ---------------------------- | --------- | ------- | ---------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | Aktifkan peningkatan kekinian |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | Skor menjadi setengah setiap N hari |

File evergreen (`MEMORY.md`, file tanpa tanggal di `memory/`) tidak pernah dikenai decay.

### Contoh lengkap

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## Path memory tambahan

| Key          | Type       | Deskripsi                               |
| ------------ | ---------- | --------------------------------------- |
| `extraPaths` | `string[]` | Direktori atau file tambahan untuk diindeks |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

Path dapat berupa absolut atau relatif terhadap workspace. Direktori dipindai
secara rekursif untuk file `.md`. Penanganan symlink bergantung pada backend aktif:
builtin engine mengabaikan symlink, sedangkan QMD mengikuti perilaku scanner QMD
yang mendasarinya.

Untuk pencarian transkrip lintas agen dengan cakupan agen, gunakan
`agents.list[].memorySearch.qmd.extraCollections` alih-alih `memory.qmd.paths`.
Koleksi tambahan tersebut mengikuti bentuk `{ path, name, pattern? }` yang sama, tetapi
digabungkan per agen dan dapat mempertahankan nama bersama yang eksplisit saat path
mengarah ke luar workspace saat ini.
Jika path terselesaikan yang sama muncul di `memory.qmd.paths` dan
`memorySearch.qmd.extraCollections`, QMD mempertahankan entri pertama dan melewati
duplikatnya.

---

## Memory multimodal (Gemini)

Indeks gambar dan audio bersama Markdown menggunakan Gemini Embedding 2:

| Key                       | Type       | Default    | Deskripsi                             |
| ------------------------- | ---------- | ---------- | ------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | Aktifkan pengindeksan multimodal      |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]`, atau `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | Ukuran file maksimum untuk pengindeksan |

Hanya berlaku untuk file di `extraPaths`. Root memory default tetap hanya Markdown.
Memerlukan `gemini-embedding-2-preview`. `fallback` harus `"none"`.

Format yang didukung: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(gambar); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (audio).

---

## Cache embedding

| Key                | Type      | Default | Deskripsi                         |
| ------------------ | --------- | ------- | --------------------------------- |
| `cache.enabled`    | `boolean` | `false` | Cache embedding chunk di SQLite   |
| `cache.maxEntries` | `number`  | `50000` | Embedding yang di-cache maksimum  |

Mencegah embedding ulang teks yang tidak berubah selama reindex atau pembaruan transkrip.

---

## Batch indexing

| Key                           | Type      | Default | Deskripsi                  |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | Aktifkan API embedding batch |
| `remote.batch.concurrency`    | `number`  | `2`     | Job batch paralel          |
| `remote.batch.wait`           | `boolean` | `true`  | Tunggu penyelesaian batch  |
| `remote.batch.pollIntervalMs` | `number`  | --      | Interval polling           |
| `remote.batch.timeoutMinutes` | `number`  | --      | Timeout batch              |

Tersedia untuk `openai`, `gemini`, dan `voyage`. Batch OpenAI biasanya
paling cepat dan paling murah untuk backfill besar.

---

## Pencarian memory sesi (eksperimental)

Indeks transkrip sesi dan tampilkan melalui `memory_search`:

| Key                           | Type       | Default      | Deskripsi                               |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | Aktifkan pengindeksan sesi              |
| `sources`                     | `string[]` | `["memory"]` | Tambahkan `"sessions"` untuk menyertakan transkrip |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | Ambang byte untuk reindex               |
| `sync.sessions.deltaMessages` | `number`   | `50`         | Ambang pesan untuk reindex              |

Pengindeksan sesi bersifat opt-in dan berjalan secara asynchronous. Hasil dapat sedikit
usang. Log sesi berada di disk, jadi perlakukan akses filesystem sebagai batas
kepercayaan.

---

## Akselerasi vector SQLite (sqlite-vec)

| Key                          | Type      | Default | Deskripsi                        |
| ---------------------------- | --------- | ------- | -------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | Gunakan sqlite-vec untuk query vector |
| `store.vector.extensionPath` | `string`  | bundled | Override path sqlite-vec         |

Saat sqlite-vec tidak tersedia, OpenClaw otomatis fallback ke cosine
similarity dalam proses.

---

## Penyimpanan indeks

| Key                   | Type     | Default                               | Deskripsi                                  |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------ |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | Lokasi indeks (mendukung token `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                           | Tokenizer FTS5 (`unicode61` atau `trigram`) |

---

## Konfigurasi backend QMD

Atur `memory.backend = "qmd"` untuk mengaktifkan. Semua pengaturan QMD berada di bawah
`memory.qmd`:

| Key                      | Type      | Default  | Deskripsi                                  |
| ------------------------ | --------- | -------- | ------------------------------------------ |
| `command`                | `string`  | `qmd`    | Path executable QMD                        |
| `searchMode`             | `string`  | `search` | Perintah pencarian: `search`, `vsearch`, `query` |
| `includeDefaultMemory`   | `boolean` | `true`   | Indeks otomatis `MEMORY.md` + `memory/**/*.md` |
| `paths[]`                | `array`   | --       | Path tambahan: `{ name, path, pattern? }`  |
| `sessions.enabled`       | `boolean` | `false`  | Indeks transkrip sesi                      |
| `sessions.retentionDays` | `number`  | --       | Retensi transkrip                          |
| `sessions.exportDir`     | `string`  | --       | Direktori ekspor                           |

OpenClaw mengutamakan bentuk query koleksi dan MCP QMD saat ini, tetapi tetap
menjaga rilis QMD yang lebih lama agar tetap berfungsi dengan fallback ke flag koleksi
`--mask` lama dan nama tool MCP lama saat diperlukan.

Override model QMD tetap berada di sisi QMD, bukan config OpenClaw. Jika Anda perlu
meng-override model QMD secara global, atur environment variable seperti
`QMD_EMBED_MODEL`, `QMD_RERANK_MODEL`, dan `QMD_GENERATE_MODEL` di environment runtime
gateway.

### Jadwal pembaruan

| Key                       | Type      | Default | Deskripsi                            |
| ------------------------- | --------- | ------- | ------------------------------------ |
| `update.interval`         | `string`  | `5m`    | Interval refresh                     |
| `update.debounceMs`       | `number`  | `15000` | Debounce perubahan file              |
| `update.onBoot`           | `boolean` | `true`  | Refresh saat startup                 |
| `update.waitForBootSync`  | `boolean` | `false` | Blok startup hingga refresh selesai  |
| `update.embedInterval`    | `string`  | --      | Cadence embedding terpisah           |
| `update.commandTimeoutMs` | `number`  | --      | Timeout untuk perintah QMD           |
| `update.updateTimeoutMs`  | `number`  | --      | Timeout untuk operasi pembaruan QMD  |
| `update.embedTimeoutMs`   | `number`  | --      | Timeout untuk operasi embedding QMD  |

### Batas

| Key                       | Type     | Default | Deskripsi                    |
| ------------------------- | -------- | ------- | ---------------------------- |
| `limits.maxResults`       | `number` | `6`     | Hasil pencarian maksimum     |
| `limits.maxSnippetChars`  | `number` | --      | Batasi panjang snippet       |
| `limits.maxInjectedChars` | `number` | --      | Batasi total karakter yang disuntikkan |
| `limits.timeoutMs`        | `number` | `4000`  | Timeout pencarian            |

### Cakupan

Mengontrol sesi mana yang dapat menerima hasil pencarian QMD. Skema sama dengan
[`session.sendPolicy`](/id/gateway/configuration-reference#session):

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Default-nya hanya DM. `match.keyPrefix` mencocokkan session key yang dinormalisasi;
`match.rawKeyPrefix` mencocokkan key mentah termasuk `agent:<id>:`.

### Sitasi

`memory.citations` berlaku untuk semua backend:

| Value            | Perilaku                                              |
| ---------------- | ----------------------------------------------------- |
| `auto` (default) | Sertakan footer `Source: <path#line>` dalam snippet   |
| `on`             | Selalu sertakan footer                                |
| `off`            | Hilangkan footer (path tetap diteruskan ke agen secara internal) |

### Contoh QMD lengkap

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming (eksperimental)

Dreaming dikonfigurasi di bawah `plugins.entries.memory-core.config.dreaming`,
bukan di bawah `agents.defaults.memorySearch`.

Dreaming berjalan sebagai satu sweep terjadwal dan menggunakan fase internal light/deep/REM sebagai
detail implementasi.

Untuk perilaku konseptual dan perintah slash, lihat [Dreaming](/concepts/dreaming).

### Pengaturan pengguna

| Key         | Type      | Default     | Deskripsi                                        |
| ----------- | --------- | ----------- | ------------------------------------------------ |
| `enabled`   | `boolean` | `false`     | Mengaktifkan atau menonaktifkan dreaming sepenuhnya |
| `frequency` | `string`  | `0 3 * * *` | Cadence cron opsional untuk sweep dreaming penuh |

### Contoh

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

Catatan:

- Dreaming menulis machine state ke `memory/.dreams/`.
- Dreaming menulis output naratif yang dapat dibaca manusia ke `DREAMS.md` (atau `dreams.md` yang sudah ada).
- Kebijakan dan ambang fase light/deep/REM adalah perilaku internal, bukan config yang ditujukan untuk pengguna.
