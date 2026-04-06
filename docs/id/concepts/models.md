---
read_when:
    - Menambahkan atau mengubah CLI model (models list/set/scan/aliases/fallbacks)
    - Mengubah perilaku fallback model atau UX pemilihan
    - Memperbarui probe pemindaian model (alat/gambar)
summary: 'CLI model: daftar, setel, alias, fallback, pemindaian, status'
title: Models CLI
x-i18n:
    generated_at: "2026-04-06T03:07:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299602ccbe0c3d6bbdb2deab22bc60e1300ef6843ed0b8b36be574cc0213c155
    source_path: concepts/models.md
    workflow: 15
---

# Models CLI

Lihat [/concepts/model-failover](/id/concepts/model-failover) untuk rotasi profil auth,
cooldown, dan bagaimana hal itu berinteraksi dengan fallback.
Ringkasan cepat provider + contoh: [/concepts/model-providers](/id/concepts/model-providers).

## Cara kerja pemilihan model

OpenClaw memilih model dalam urutan ini:

1. Model **utama** (`agents.defaults.model.primary` atau `agents.defaults.model`).
2. **Fallback** di `agents.defaults.model.fallbacks` (sesuai urutan).
3. **Failover auth provider** terjadi di dalam sebuah provider sebelum berpindah ke
   model berikutnya.

Terkait:

- `agents.defaults.models` adalah allowlist/katalog model yang dapat digunakan OpenClaw (termasuk alias).
- `agents.defaults.imageModel` digunakan **hanya ketika** model utama tidak dapat menerima gambar.
- `agents.defaults.pdfModel` digunakan oleh alat `pdf`. Jika dihilangkan, alat
  akan fallback ke `agents.defaults.imageModel`, lalu model sesi/default yang
  telah diselesaikan.
- `agents.defaults.imageGenerationModel` digunakan oleh kapabilitas pembuatan gambar bersama. Jika dihilangkan, `image_generate` masih dapat menyimpulkan default provider yang didukung auth. Ia mencoba provider default saat ini terlebih dahulu, lalu provider pembuatan gambar terdaftar lainnya dalam urutan id provider. Jika Anda menetapkan provider/model tertentu, konfigurasi juga auth/API key provider tersebut.
- `agents.defaults.musicGenerationModel` digunakan oleh kapabilitas pembuatan musik bersama. Jika dihilangkan, `music_generate` masih dapat menyimpulkan default provider yang didukung auth. Ia mencoba provider default saat ini terlebih dahulu, lalu provider pembuatan musik terdaftar lainnya dalam urutan id provider. Jika Anda menetapkan provider/model tertentu, konfigurasi juga auth/API key provider tersebut.
- `agents.defaults.videoGenerationModel` digunakan oleh kapabilitas pembuatan video bersama. Jika dihilangkan, `video_generate` masih dapat menyimpulkan default provider yang didukung auth. Ia mencoba provider default saat ini terlebih dahulu, lalu provider pembuatan video terdaftar lainnya dalam urutan id provider. Jika Anda menetapkan provider/model tertentu, konfigurasi juga auth/API key provider tersebut.
- Default per agen dapat menimpa `agents.defaults.model` melalui `agents.list[].model` plus binding (lihat [/concepts/multi-agent](/id/concepts/multi-agent)).

## Kebijakan model cepat

- Setel model utama Anda ke model generasi terbaru terkuat yang tersedia untuk Anda.
- Gunakan fallback untuk tugas yang sensitif terhadap biaya/latensi dan obrolan dengan risiko lebih rendah.
- Untuk agen dengan alat aktif atau input yang tidak tepercaya, hindari model tingkat lama/lebih lemah.

## Onboarding (disarankan)

Jika Anda tidak ingin mengedit konfigurasi secara manual, jalankan onboarding:

```bash
openclaw onboard
```

Ini dapat menyiapkan model + auth untuk provider umum, termasuk **langganan
OpenAI Code (Codex)** (OAuth) dan **Anthropic** (API key atau Claude CLI).

## Kunci konfigurasi (ringkasan)

- `agents.defaults.model.primary` dan `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` dan `agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` dan `agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` dan `agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` dan `agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models` (allowlist + alias + parameter provider)
- `models.providers` (provider kustom yang ditulis ke `models.json`)

Referensi model dinormalisasi ke huruf kecil. Alias provider seperti `z.ai/*` dinormalisasi
menjadi `zai/*`.

Contoh konfigurasi provider (termasuk OpenCode) ada di
[/providers/opencode](/id/providers/opencode).

## "Model is not allowed" (dan mengapa balasan berhenti)

Jika `agents.defaults.models` disetel, itu menjadi **allowlist** untuk `/model` dan untuk
override sesi. Ketika pengguna memilih model yang tidak ada dalam allowlist itu,
OpenClaw mengembalikan:

```
Model "provider/model" is not allowed. Use /model to list available models.
```

Ini terjadi **sebelum** balasan normal dibuat, sehingga pesan dapat terasa
seperti “tidak merespons.” Solusinya adalah:

- Tambahkan model ke `agents.defaults.models`, atau
- Hapus allowlist (hapus `agents.defaults.models`), atau
- Pilih model dari `/model list`.

Contoh konfigurasi allowlist:

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-6" },
    models: {
      "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## Mengganti model di obrolan (`/model`)

Anda dapat mengganti model untuk sesi saat ini tanpa memulai ulang:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

Catatan:

- `/model` (dan `/model list`) adalah pemilih ringkas bernomor (keluarga model + provider yang tersedia).
- Di Discord, `/model` dan `/models` membuka pemilih interaktif dengan dropdown provider dan model plus langkah Submit.
- `/model <#>` memilih dari pemilih itu.
- `/model` menyimpan pilihan sesi baru segera.
- Jika agen sedang idle, proses berikutnya langsung menggunakan model baru.
- Jika sebuah proses sudah aktif, OpenClaw menandai peralihan langsung sebagai tertunda dan hanya memulai ulang ke model baru pada titik percobaan ulang yang aman.
- Jika aktivitas alat atau keluaran balasan sudah dimulai, peralihan tertunda dapat tetap antre sampai ada kesempatan percobaan ulang berikutnya atau giliran pengguna berikutnya.
- `/model status` adalah tampilan terperinci (kandidat auth dan, jika dikonfigurasi, `baseUrl` endpoint provider + mode `api`).
- Referensi model diurai dengan membagi pada `/` **pertama**. Gunakan `provider/model` saat mengetik `/model <ref>`.
- Jika id model itu sendiri mengandung `/` (gaya OpenRouter), Anda harus menyertakan prefiks provider (contoh: `/model openrouter/moonshotai/kimi-k2`).
- Jika Anda menghilangkan provider, OpenClaw menyelesaikan input dalam urutan ini:
  1. kecocokan alias
  2. kecocokan provider-terkonfigurasi yang unik untuk id model tanpa prefiks yang persis sama
  3. fallback usang ke provider default yang dikonfigurasi
     Jika provider tersebut tidak lagi mengekspos model default terkonfigurasi, OpenClaw
     akan fallback ke provider/model terkonfigurasi pertama untuk menghindari
     menampilkan default provider lama yang sudah dihapus.

Perilaku/perintah konfigurasi lengkap: [Slash commands](/id/tools/slash-commands).

## Perintah CLI

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models` (tanpa subperintah) adalah pintasan untuk `models status`.

### `models list`

Secara default menampilkan model yang dikonfigurasi. Flag yang berguna:

- `--all`: katalog lengkap
- `--local`: hanya provider lokal
- `--provider <name>`: filter berdasarkan provider
- `--plain`: satu model per baris
- `--json`: keluaran yang dapat dibaca mesin

### `models status`

Menampilkan model utama yang telah diselesaikan, fallback, model gambar, dan ringkasan auth
dari provider yang dikonfigurasi. Ini juga menampilkan status kedaluwarsa OAuth untuk profil yang ditemukan
di penyimpanan auth (memperingatkan dalam 24 jam secara default). `--plain` hanya mencetak
model utama yang telah diselesaikan.
Status OAuth selalu ditampilkan (dan disertakan dalam keluaran `--json`). Jika sebuah provider yang dikonfigurasi
tidak memiliki kredensial, `models status` mencetak bagian **Missing auth**.
JSON mencakup `auth.oauth` (jendela peringatan + profil) dan `auth.providers`
(auth efektif per provider, termasuk kredensial berbasis env). `auth.oauth`
hanya kesehatan profil penyimpanan auth; provider yang hanya berbasis env tidak muncul di sana.
Gunakan `--check` untuk otomatisasi (exit `1` saat hilang/kedaluwarsa, `2` saat akan kedaluwarsa).
Gunakan `--probe` untuk pemeriksaan auth langsung; baris probe dapat berasal dari profil auth, kredensial env,
atau `models.json`.
Jika `auth.order.<provider>` eksplisit menghilangkan sebuah profil tersimpan, probe melaporkan
`excluded_by_auth_order` alih-alih mencobanya. Jika auth ada tetapi tidak ada model yang bisa diprobe
yang dapat diselesaikan untuk provider tersebut, probe melaporkan `status: no_model`.

Pilihan auth bergantung pada provider/akun. Untuk host gateway yang selalu aktif, API
key biasanya paling mudah diprediksi; penggunaan ulang Claude CLI dan profil
OAuth/token Anthropic yang sudah ada juga didukung.

Contoh (Claude CLI):

```bash
claude auth login
openclaw models status
```

## Pemindaian (model gratis OpenRouter)

`openclaw models scan` memeriksa **katalog model gratis** OpenRouter dan secara
opsional dapat memprobe model untuk dukungan alat dan gambar.

Flag utama:

- `--no-probe`: lewati probe langsung (hanya metadata)
- `--min-params <b>`: ukuran parameter minimum (miliar)
- `--max-age-days <days>`: lewati model yang lebih lama
- `--provider <name>`: filter prefiks provider
- `--max-candidates <n>`: ukuran daftar fallback
- `--set-default`: setel `agents.defaults.model.primary` ke pilihan pertama
- `--set-image`: setel `agents.defaults.imageModel.primary` ke pilihan gambar pertama

Probing memerlukan OpenRouter API key (dari profil auth atau
`OPENROUTER_API_KEY`). Tanpa key, gunakan `--no-probe` untuk hanya menampilkan kandidat.

Hasil pemindaian diberi peringkat berdasarkan:

1. Dukungan gambar
2. Latensi alat
3. Ukuran konteks
4. Jumlah parameter

Input

- Daftar OpenRouter `/models` (filter `:free`)
- Memerlukan OpenRouter API key dari profil auth atau `OPENROUTER_API_KEY` (lihat [/environment](/id/help/environment))
- Filter opsional: `--max-age-days`, `--min-params`, `--provider`, `--max-candidates`
- Kontrol probe: `--timeout`, `--concurrency`

Saat dijalankan di TTY, Anda dapat memilih fallback secara interaktif. Dalam mode non-interaktif,
berikan `--yes` untuk menerima default.

## Registry model (`models.json`)

Provider kustom di `models.providers` ditulis ke `models.json` di bawah
direktori agen (default `~/.openclaw/agents/<agentId>/agent/models.json`). File ini
digabungkan secara default kecuali `models.mode` disetel ke `replace`.

Prioritas mode penggabungan untuk id provider yang cocok:

- `baseUrl` non-kosong yang sudah ada di `models.json` agen akan diprioritaskan.
- `apiKey` non-kosong di `models.json` agen diprioritaskan hanya ketika provider tersebut tidak dikelola SecretRef dalam konteks konfigurasi/profil-auth saat ini.
- Nilai `apiKey` provider yang dikelola SecretRef disegarkan dari marker sumber (`ENV_VAR_NAME` untuk referensi env, `secretref-managed` untuk referensi file/exec) alih-alih menyimpan secret yang telah diselesaikan.
- Nilai header provider yang dikelola SecretRef disegarkan dari marker sumber (`secretref-env:ENV_VAR_NAME` untuk referensi env, `secretref-managed` untuk referensi file/exec).
- `apiKey`/`baseUrl` agen yang kosong atau tidak ada akan fallback ke konfigurasi `models.providers`.
- Bidang provider lain disegarkan dari konfigurasi dan data katalog yang dinormalisasi.

Persistensi marker bersifat source-authoritative: OpenClaw menulis marker dari snapshot konfigurasi sumber aktif (pra-resolusi), bukan dari nilai secret runtime yang telah diselesaikan.
Ini berlaku setiap kali OpenClaw meregenerasi `models.json`, termasuk jalur yang dipicu perintah seperti `openclaw agent`.

## Terkait

- [Model Providers](/id/concepts/model-providers) — perutean provider dan auth
- [Model Failover](/id/concepts/model-failover) — rantai fallback
- [Image Generation](/id/tools/image-generation) — konfigurasi model gambar
- [Music Generation](/tools/music-generation) — konfigurasi model musik
- [Video Generation](/tools/video-generation) — konfigurasi model video
- [Configuration Reference](/id/gateway/configuration-reference#agent-defaults) — kunci konfigurasi model
