---
read_when:
    - Anda ingin menyiapkan QMD sebagai backend memori Anda
    - Anda menginginkan fitur memori lanjutan seperti reranking atau path terindeks tambahan
summary: Sidecar pencarian local-first dengan BM25, vektor, reranking, dan ekspansi kueri
title: Mesin Memori QMD
x-i18n:
    generated_at: "2026-04-06T03:06:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 36642c7df94b88f562745dd2270334379f2aeeef4b363a8c13ef6be42dadbe5c
    source_path: concepts/memory-qmd.md
    workflow: 15
---

# Mesin Memori QMD

[QMD](https://github.com/tobi/qmd) adalah sidecar pencarian local-first yang berjalan
bersama OpenClaw. QMD menggabungkan BM25, pencarian vektor, dan reranking dalam satu
biner, dan dapat mengindeks konten di luar file memori workspace Anda.

## Apa yang ditambahkan dibandingkan bawaan

- **Reranking dan ekspansi kueri** untuk perolehan yang lebih baik.
- **Mengindeks direktori tambahan** -- dokumentasi proyek, catatan tim, apa pun di disk.
- **Mengindeks transkrip sesi** -- mengingat percakapan sebelumnya.
- **Sepenuhnya lokal** -- berjalan melalui Bun + node-llama-cpp, mengunduh model GGUF secara otomatis.
- **Fallback otomatis** -- jika QMD tidak tersedia, OpenClaw akan kembali ke
  mesin bawaan dengan mulus.

## Memulai

### Prasyarat

- Instal QMD: `npm install -g @tobilu/qmd` atau `bun install -g @tobilu/qmd`
- Build SQLite yang mengizinkan ekstensi (`brew install sqlite` di macOS).
- QMD harus ada di `PATH` gateway.
- macOS dan Linux berfungsi langsung. Windows paling didukung melalui WSL2.

### Aktifkan

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw membuat home QMD mandiri di bawah
`~/.openclaw/agents/<agentId>/qmd/` dan mengelola siklus hidup sidecar
secara otomatis -- koleksi, pembaruan, dan proses embedding ditangani untuk Anda.
OpenClaw memprioritaskan bentuk koleksi QMD saat ini dan kueri MCP, tetapi tetap kembali ke
flag koleksi `--mask` lama dan nama tool MCP yang lebih lama bila diperlukan.

## Cara kerja sidecar

- OpenClaw membuat koleksi dari file memori workspace Anda dan setiap
  `memory.qmd.paths` yang dikonfigurasi, lalu menjalankan `qmd update` + `qmd embed` saat boot
  dan secara berkala (default setiap 5 menit).
- Penyegaran saat boot berjalan di latar belakang sehingga startup chat tidak terblokir.
- Pencarian menggunakan `searchMode` yang dikonfigurasi (default: `search`; juga mendukung
  `vsearch` dan `query`). Jika suatu mode gagal, OpenClaw mencoba ulang dengan `qmd query`.
- Jika QMD gagal sepenuhnya, OpenClaw kembali ke mesin SQLite bawaan.

<Info>
Pencarian pertama mungkin lambat -- QMD mengunduh model GGUF (~2 GB) secara otomatis untuk
reranking dan ekspansi kueri pada proses `qmd query` pertama.
</Info>

## Override model

Variabel environment model QMD diteruskan tanpa perubahan dari proses
gateway, sehingga Anda dapat menyetel QMD secara global tanpa menambahkan konfigurasi OpenClaw baru:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Setelah mengubah model embedding, jalankan ulang embedding agar indeks sesuai dengan
ruang vektor yang baru.

## Mengindeks path tambahan

Arahkan QMD ke direktori tambahan agar dapat dicari:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Cuplikan dari path tambahan muncul sebagai `qmd/<collection>/<relative-path>` di
hasil pencarian. `memory_get` memahami prefiks ini dan membaca dari root koleksi
yang benar.

## Mengindeks transkrip sesi

Aktifkan pengindeksan sesi untuk mengingat percakapan sebelumnya:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Transkrip diekspor sebagai giliran Pengguna/Asisten yang telah disanitasi ke dalam koleksi QMD
khusus di bawah `~/.openclaw/agents/<id>/qmd/sessions/`.

## Cakupan pencarian

Secara default, hasil pencarian QMD hanya ditampilkan dalam sesi DM (bukan grup atau
channel). Konfigurasikan `memory.qmd.scope` untuk mengubah ini:

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

Saat cakupan menolak pencarian, OpenClaw mencatat peringatan dengan channel turunan dan
jenis chat sehingga hasil kosong lebih mudah di-debug.

## Sitasi

Saat `memory.citations` adalah `auto` atau `on`, cuplikan pencarian menyertakan
footer `Source: <path#line>`. Atur `memory.citations = "off"` untuk menghilangkan footer
sambil tetap meneruskan path ke agen secara internal.

## Kapan digunakan

Pilih QMD saat Anda membutuhkan:

- Reranking untuk hasil dengan kualitas lebih tinggi.
- Mencari dokumentasi proyek atau catatan di luar workspace.
- Mengingat percakapan sesi sebelumnya.
- Pencarian sepenuhnya lokal tanpa API key.

Untuk pengaturan yang lebih sederhana, [mesin bawaan](/id/concepts/memory-builtin) berfungsi baik
tanpa dependensi tambahan.

## Pemecahan masalah

**QMD tidak ditemukan?** Pastikan biner ada di `PATH` gateway. Jika OpenClaw
berjalan sebagai layanan, buat symlink:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

**Pencarian pertama sangat lambat?** QMD mengunduh model GGUF saat pertama digunakan. Lakukan pre-warm
dengan `qmd query "test"` menggunakan direktori XDG yang sama seperti yang digunakan OpenClaw.

**Pencarian timeout?** Tingkatkan `memory.qmd.limits.timeoutMs` (default: 4000ms).
Atur ke `120000` untuk perangkat keras yang lebih lambat.

**Hasil kosong di chat grup?** Periksa `memory.qmd.scope` -- default hanya
mengizinkan sesi DM.

**Repo sementara yang terlihat oleh workspace menyebabkan `ENAMETOOLONG` atau pengindeksan rusak?**
Traversal QMD saat ini mengikuti perilaku pemindai QMD yang mendasarinya, bukan
aturan symlink bawaan OpenClaw. Simpan checkout monorepo sementara di bawah
direktori tersembunyi seperti `.tmp/` atau di luar root QMD yang diindeks sampai QMD menyediakan
traversal yang aman terhadap siklus atau kontrol pengecualian eksplisit.

## Konfigurasi

Untuk permukaan konfigurasi lengkap (`memory.qmd.*`), mode pencarian, interval pembaruan,
aturan cakupan, dan semua pengaturan lainnya, lihat
[Referensi konfigurasi memori](/id/reference/memory-config).
