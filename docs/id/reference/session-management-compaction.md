---
read_when:
    - Anda perlu men-debug id sesi, JSONL transkrip, atau field sessions.json
    - Anda sedang mengubah perilaku auto-compaction atau menambahkan housekeeping “pra-compaction”
    - Anda ingin mengimplementasikan memory flush atau giliran sistem senyap
summary: 'Pendalaman: session store + transkrip, siklus hidup, dan internal (auto)compaction'
title: Pendalaman Pengelolaan Sesi
x-i18n:
    generated_at: "2026-04-06T03:11:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: e0d8c2d30be773eac0424f7a4419ab055fdd50daac8bc654e7d250c891f2c3b8
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Pengelolaan Sesi & Compaction (Pendalaman)

Dokumen ini menjelaskan bagaimana OpenClaw mengelola sesi dari ujung ke ujung:

- **Perutean sesi** (bagaimana pesan masuk dipetakan ke `sessionKey`)
- **Session store** (`sessions.json`) dan apa yang dilacaknya
- **Persistensi transkrip** (`*.jsonl`) dan strukturnya
- **Kebersihan transkrip** (perbaikan khusus provider sebelum run)
- **Batas konteks** (context window vs token yang dilacak)
- **Compaction** (compaction manual + otomatis) dan di mana mengaitkan pekerjaan pra-compaction
- **Housekeeping senyap** (misalnya penulisan memori yang tidak boleh menghasilkan output yang terlihat pengguna)

Jika Anda ingin ikhtisar tingkat tinggi terlebih dahulu, mulai dari:

- [/concepts/session](/id/concepts/session)
- [/concepts/compaction](/id/concepts/compaction)
- [/concepts/memory](/id/concepts/memory)
- [/concepts/memory-search](/id/concepts/memory-search)
- [/concepts/session-pruning](/id/concepts/session-pruning)
- [/reference/transcript-hygiene](/id/reference/transcript-hygiene)

---

## Sumber kebenaran: Gateway

OpenClaw dirancang di sekitar satu **proses Gateway** yang memiliki state sesi.

- UI (app macOS, web Control UI, TUI) harus mengkueri Gateway untuk daftar sesi dan jumlah token.
- Dalam mode remote, file sesi berada di host remote; “memeriksa file Mac lokal Anda” tidak akan mencerminkan apa yang digunakan Gateway.

---

## Dua lapisan persistensi

OpenClaw menyimpan sesi dalam dua lapisan:

1. **Session store (`sessions.json`)**
   - Peta key/value: `sessionKey -> SessionEntry`
   - Kecil, dapat berubah, aman untuk diedit (atau entri dihapus)
   - Melacak metadata sesi (id sesi saat ini, aktivitas terakhir, toggle, penghitung token, dll.)

2. **Transkrip (`<sessionId>.jsonl`)**
   - Transkrip append-only dengan struktur pohon (entri memiliki `id` + `parentId`)
   - Menyimpan percakapan sebenarnya + tool call + ringkasan compaction
   - Digunakan untuk membangun ulang konteks model untuk giliran berikutnya

---

## Lokasi di disk

Per agent, pada host Gateway:

- Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transkrip: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Sesi topik Telegram: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw menyelesaikannya melalui `src/config/sessions.ts`.

---

## Pemeliharaan store dan kontrol disk

Persistensi sesi memiliki kontrol pemeliharaan otomatis (`session.maintenance`) untuk `sessions.json` dan artefak transkrip:

- `mode`: `warn` (default) atau `enforce`
- `pruneAfter`: batas umur entri usang (default `30d`)
- `maxEntries`: batas entri di `sessions.json` (default `500`)
- `rotateBytes`: rotasi `sessions.json` saat terlalu besar (default `10mb`)
- `resetArchiveRetention`: retensi untuk arsip transkrip `*.reset.<timestamp>` (default: sama dengan `pruneAfter`; `false` menonaktifkan pembersihan)
- `maxDiskBytes`: anggaran direktori sesi opsional
- `highWaterBytes`: target opsional setelah pembersihan (default `80%` dari `maxDiskBytes`)

Urutan penegakan untuk pembersihan anggaran disk (`mode: "enforce"`):

1. Hapus artefak transkrip arsip atau yatim tertua terlebih dahulu.
2. Jika masih di atas target, keluarkan entri sesi tertua beserta file transkripnya.
3. Lanjutkan sampai penggunaan berada pada atau di bawah `highWaterBytes`.

Dalam `mode: "warn"`, OpenClaw melaporkan potensi pengeluaran tetapi tidak mengubah store/file.

Jalankan pemeliharaan sesuai permintaan:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Sesi cron dan log run

Run cron terisolasi juga membuat entri/transkrip sesi, dan memiliki kontrol retensi khusus:

- `cron.sessionRetention` (default `24h`) memangkas sesi run cron terisolasi lama dari session store (`false` menonaktifkan).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` memangkas file `~/.openclaw/cron/runs/<jobId>.jsonl` (default: `2_000_000` byte dan `2000` baris).

---

## Kunci sesi (`sessionKey`)

`sessionKey` mengidentifikasi _bucket percakapan mana_ yang sedang Anda masuki (perutean + isolasi).

Pola umum:

- Chat utama/langsung (per agent): `agent:<agentId>:<mainKey>` (default `main`)
- Grup: `agent:<agentId>:<channel>:group:<id>`
- Room/channel (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` atau `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (kecuali ditimpa)

Aturan kanonis didokumentasikan di [/concepts/session](/id/concepts/session).

---

## Id sesi (`sessionId`)

Setiap `sessionKey` menunjuk ke `sessionId` saat ini (file transkrip yang melanjutkan percakapan).

Aturan praktis:

- **Reset** (`/new`, `/reset`) membuat `sessionId` baru untuk `sessionKey` tersebut.
- **Reset harian** (default pukul 4:00 pagi waktu lokal di host gateway) membuat `sessionId` baru pada pesan berikutnya setelah batas reset.
- **Kedaluwarsa idle** (`session.reset.idleMinutes` atau `session.idleMinutes` lama) membuat `sessionId` baru ketika sebuah pesan tiba setelah jendela idle. Saat harian + idle sama-sama dikonfigurasi, mana yang kedaluwarsa lebih dulu akan menang.
- **Thread parent fork guard** (`session.parentForkMaxTokens`, default `100000`) melewati forking transkrip induk saat sesi induk sudah terlalu besar; thread baru mulai dari awal. Setel `0` untuk menonaktifkan.

Detail implementasi: keputusan ini terjadi di `initSessionState()` dalam `src/auto-reply/reply/session.ts`.

---

## Skema session store (`sessions.json`)

Tipe nilai store adalah `SessionEntry` di `src/config/sessions.ts`.

Field utama (tidak lengkap):

- `sessionId`: id transkrip saat ini (nama file diturunkan dari sini kecuali `sessionFile` disetel)
- `updatedAt`: stempel waktu aktivitas terakhir
- `sessionFile`: override path transkrip eksplisit opsional
- `chatType`: `direct | group | room` (membantu UI dan kebijakan pengiriman)
- `provider`, `subject`, `room`, `space`, `displayName`: metadata untuk pelabelan grup/channel
- Toggle:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (override per sesi)
- Pemilihan model:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Penghitung token (best-effort / bergantung pada provider):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: seberapa sering auto-compaction selesai untuk kunci sesi ini
- `memoryFlushAt`: stempel waktu untuk memory flush pra-compaction terakhir
- `memoryFlushCompactionCount`: jumlah compaction saat flush terakhir dijalankan

Store aman untuk diedit, tetapi Gateway adalah otoritasnya: Gateway dapat menulis ulang atau menghidrasi ulang entri saat sesi berjalan.

---

## Struktur transkrip (`*.jsonl`)

Transkrip dikelola oleh `@mariozechner/pi-coding-agent` `SessionManager`.

File ini berformat JSONL:

- Baris pertama: header sesi (`type: "session"`, mencakup `id`, `cwd`, `timestamp`, `parentSession` opsional)
- Lalu: entri sesi dengan `id` + `parentId` (pohon)

Jenis entri yang penting:

- `message`: pesan user/assistant/toolResult
- `custom_message`: pesan yang diinjeksi extension yang _memang_ masuk ke konteks model (dapat disembunyikan dari UI)
- `custom`: state extension yang _tidak_ masuk ke konteks model
- `compaction`: ringkasan compaction yang dipersistenkan dengan `firstKeptEntryId` dan `tokensBefore`
- `branch_summary`: ringkasan yang dipersistenkan saat menavigasi cabang pohon

OpenClaw sengaja **tidak** “memperbaiki” transkrip; Gateway menggunakan `SessionManager` untuk membacanya/menulisnya.

---

## Context window vs token yang dilacak

Dua konsep berbeda penting di sini:

1. **Context window model**: batas keras per model (token yang terlihat oleh model)
2. **Penghitung session store**: statistik bergulir yang ditulis ke `sessions.json` (digunakan untuk /status dan dasbor)

Jika Anda sedang menyetel batas:

- Context window berasal dari katalog model (dan dapat ditimpa melalui config).
- `contextTokens` di store adalah nilai estimasi/pelaporan runtime; jangan perlakukan sebagai jaminan ketat.

Untuk informasi lebih lanjut, lihat [/token-use](/id/reference/token-use).

---

## Compaction: apa itu

Compaction merangkum percakapan lama menjadi entri `compaction` yang dipersistenkan dalam transkrip dan menjaga pesan terbaru tetap utuh.

Setelah compaction, giliran berikutnya akan melihat:

- Ringkasan compaction
- Pesan setelah `firstKeptEntryId`

Compaction bersifat **persisten** (tidak seperti session pruning). Lihat [/concepts/session-pruning](/id/concepts/session-pruning).

## Batas chunk compaction dan pemasangan tool

Saat OpenClaw membagi transkrip panjang menjadi chunk compaction, OpenClaw menjaga
tool call assistant tetap dipasangkan dengan entri `toolResult` yang cocok.

- Jika pembagian berbagi-token jatuh di antara tool call dan hasilnya, OpenClaw
  menggeser batas ke pesan tool-call assistant alih-alih memisahkan
  pasangan tersebut.
- Jika blok tool-result di ekor sebaliknya akan mendorong chunk melewati target,
  OpenClaw mempertahankan blok tool yang masih tertunda dan menjaga ekor yang belum diringkas tetap utuh.
- Blok tool-call yang dibatalkan/error tidak menahan pembagian tertunda tetap terbuka.

---

## Kapan auto-compaction terjadi (runtime Pi)

Dalam embedded Pi agent, auto-compaction dipicu dalam dua kasus:

1. **Pemulihan overflow**: model mengembalikan error overflow konteks
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded`, dan varian serupa berbentuk provider) → compact → coba lagi.
2. **Pemeliharaan ambang**: setelah giliran berhasil, ketika:

`contextTokens > contextWindow - reserveTokens`

Di mana:

- `contextWindow` adalah context window model
- `reserveTokens` adalah ruang cadangan yang disisihkan untuk prompt + output model berikutnya

Ini adalah semantik runtime Pi (OpenClaw mengonsumsi eventnya, tetapi Pi yang memutuskan kapan harus compact).

---

## Setelan compaction (`reserveTokens`, `keepRecentTokens`)

Setelan compaction Pi berada di setelan Pi:

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw juga menegakkan batas bawah keamanan untuk embedded run:

- Jika `compaction.reserveTokens < reserveTokensFloor`, OpenClaw menaikkannya.
- Batas bawah default adalah `20000` token.
- Setel `agents.defaults.compaction.reserveTokensFloor: 0` untuk menonaktifkan batas bawah.
- Jika nilainya sudah lebih tinggi, OpenClaw membiarkannya.

Mengapa: sisakan cukup ruang untuk “housekeeping” multi-giliran (seperti penulisan memori) sebelum compaction menjadi tak terhindarkan.

Implementasi: `ensurePiCompactionReserveTokens()` dalam `src/agents/pi-settings.ts`
(dipanggil dari `src/agents/pi-embedded-runner.ts`).

---

## Permukaan yang terlihat pengguna

Anda dapat mengamati compaction dan state sesi melalui:

- `/status` (di sesi chat mana pun)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Mode verbose: `🧹 Auto-compaction complete` + jumlah compaction

---

## Housekeeping senyap (`NO_REPLY`)

OpenClaw mendukung giliran “senyap” untuk tugas latar belakang di mana pengguna tidak boleh melihat output perantara.

Konvensi:

- Assistant memulai outputnya dengan token senyap persis `NO_REPLY` /
  `no_reply` untuk menunjukkan “jangan kirim balasan ke pengguna”.
- OpenClaw menghapus/menekannya di lapisan pengiriman.
- Penekanan token senyap persis tidak peka huruf besar-kecil, sehingga `NO_REPLY` dan
  `no_reply` keduanya dihitung ketika seluruh payload hanyalah token senyap tersebut.
- Ini hanya untuk giliran latar belakang/tanpa pengiriman yang benar-benar senyap; ini bukan jalan pintas untuk
  permintaan pengguna biasa yang dapat ditindaklanjuti.

Mulai `2026.1.10`, OpenClaw juga menekan **streaming draf/typing** ketika sebuah
chunk parsial dimulai dengan `NO_REPLY`, sehingga operasi senyap tidak membocorkan output parsial di tengah giliran.

---

## "Memory flush" pra-compaction (sudah diimplementasikan)

Tujuan: sebelum auto-compaction terjadi, jalankan giliran agentic senyap yang menulis state
tahan lama ke disk (misalnya `memory/YYYY-MM-DD.md` di workspace agent) sehingga compaction tidak dapat
menghapus konteks penting.

OpenClaw menggunakan pendekatan **flush pra-ambang**:

1. Pantau penggunaan konteks sesi.
2. Saat penggunaan melewati “ambang lunak” (di bawah ambang compaction Pi), jalankan direktif
   senyap “tulis memori sekarang” ke agent.
3. Gunakan token senyap persis `NO_REPLY` / `no_reply` agar pengguna tidak melihat
   apa pun.

Config (`agents.defaults.compaction.memoryFlush`):

- `enabled` (default: `true`)
- `softThresholdTokens` (default: `4000`)
- `prompt` (pesan user untuk giliran flush)
- `systemPrompt` (system prompt tambahan yang ditambahkan untuk giliran flush)

Catatan:

- Prompt/system prompt default menyertakan petunjuk `NO_REPLY` untuk menekan
  pengiriman.
- Flush berjalan sekali per siklus compaction (dilacak di `sessions.json`).
- Flush hanya berjalan untuk sesi embedded Pi.
- Flush dilewati saat workspace sesi bersifat read-only (`workspaceAccess: "ro"` atau `"none"`).
- Lihat [Memory](/id/concepts/memory) untuk tata letak file workspace dan pola penulisan.

Pi juga mengekspos hook `session_before_compact` di extension API, tetapi logika
flush OpenClaw saat ini berada di sisi Gateway.

---

## Checklist pemecahan masalah

- Session key salah? Mulai dari [/concepts/session](/id/concepts/session) dan konfirmasi `sessionKey` di `/status`.
- Store vs transkrip tidak cocok? Konfirmasi host Gateway dan path store dari `openclaw status`.
- Spam compaction? Periksa:
  - context window model (terlalu kecil)
  - setelan compaction (`reserveTokens` yang terlalu tinggi untuk window model dapat menyebabkan compaction lebih awal)
  - pembengkakan tool-result: aktifkan/sesuaikan session pruning
- Giliran senyap bocor? Pastikan balasan dimulai dengan `NO_REPLY` (token persis tidak peka huruf besar-kecil) dan Anda menggunakan build yang menyertakan perbaikan penekanan streaming.
