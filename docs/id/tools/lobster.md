---
read_when:
    - Anda menginginkan alur kerja multi-langkah yang deterministik dengan persetujuan eksplisit
    - Anda perlu melanjutkan alur kerja tanpa menjalankan ulang langkah-langkah sebelumnya
summary: Runtime alur kerja bertipe untuk OpenClaw dengan gerbang persetujuan yang dapat dilanjutkan.
title: Lobster
x-i18n:
    generated_at: "2026-04-06T03:12:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: c1014945d104ef8fdca0d30be89e35136def1b274c6403b06de29e8502b8124b
    source_path: tools/lobster.md
    workflow: 15
---

# Lobster

Lobster adalah shell alur kerja yang memungkinkan OpenClaw menjalankan rangkaian alat multi-langkah sebagai satu operasi deterministik dengan checkpoint persetujuan yang eksplisit.

Lobster adalah satu lapisan authoring di atas pekerjaan latar belakang terlepas. Untuk orkestrasi alur di atas task individual, lihat [Task Flow](/id/automation/taskflow) (`openclaw tasks flow`). Untuk buku besar aktivitas task, lihat [`openclaw tasks`](/id/automation/tasks).

## Hook

Asisten Anda dapat membangun alat yang mengelola dirinya sendiri. Minta sebuah alur kerja, dan 30 menit kemudian Anda memiliki CLI plus pipeline yang berjalan sebagai satu panggilan. Lobster adalah bagian yang hilang: pipeline deterministik, persetujuan eksplisit, dan state yang dapat dilanjutkan.

## Mengapa

Saat ini, alur kerja yang kompleks memerlukan banyak pemanggilan alat bolak-balik. Setiap panggilan memakan token, dan LLM harus mengorkestrasi setiap langkah. Lobster memindahkan orkestrasi itu ke runtime bertipe:

- **Satu panggilan, bukan banyak**: OpenClaw menjalankan satu pemanggilan alat Lobster dan mendapatkan hasil terstruktur.
- **Persetujuan sudah terintegrasi**: Efek samping (kirim email, posting komentar) menghentikan alur kerja sampai disetujui secara eksplisit.
- **Dapat dilanjutkan**: Alur kerja yang dihentikan mengembalikan token; setujui dan lanjutkan tanpa menjalankan ulang semuanya.

## Mengapa DSL, bukan program biasa?

Lobster sengaja dibuat kecil. Tujuannya bukan "bahasa baru," melainkan spesifikasi pipeline yang dapat diprediksi dan ramah AI dengan persetujuan dan token resume sebagai fitur utama.

- **Setujui/lanjutkan sudah terintegrasi**: Program biasa dapat meminta manusia, tetapi tidak dapat _menjeda dan melanjutkan_ dengan token yang tahan lama tanpa Anda membangun runtime itu sendiri.
- **Determinisme + auditabilitas**: Pipeline adalah data, sehingga mudah dicatat, dibandingkan, diputar ulang, dan ditinjau.
- **Surface terbatas untuk AI**: Tata bahasa kecil + piping JSON mengurangi jalur kode yang “kreatif” dan membuat validasi menjadi realistis.
- **Kebijakan keamanan tertanam**: Timeout, batas output, pemeriksaan sandbox, dan allowlist ditegakkan oleh runtime, bukan oleh setiap skrip.
- **Tetap dapat diprogram**: Setiap langkah dapat memanggil CLI atau skrip apa pun. Jika Anda menginginkan JS/TS, hasilkan file `.lobster` dari kode.

## Cara kerjanya

OpenClaw menjalankan alur kerja Lobster **in-process** menggunakan embedded runner. Tidak ada subprocess CLI eksternal yang dihasilkan; mesin alur kerja dijalankan di dalam proses gateway dan langsung mengembalikan envelope JSON.
Jika pipeline berhenti untuk persetujuan, alat mengembalikan `resumeToken` agar Anda dapat melanjutkannya nanti.

## Pola: CLI kecil + JSON pipes + persetujuan

Bangun perintah kecil yang berbicara JSON, lalu rangkai menjadi satu panggilan Lobster. (Nama perintah contoh di bawah ini — ganti dengan milik Anda.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

Jika pipeline meminta persetujuan, lanjutkan dengan token tersebut:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

AI memicu alur kerja; Lobster menjalankan langkah-langkahnya. Gerbang persetujuan menjaga efek samping tetap eksplisit dan dapat diaudit.

Contoh: memetakan item input menjadi pemanggilan alat:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## Langkah LLM khusus JSON (llm-task)

Untuk alur kerja yang memerlukan **langkah LLM terstruktur**, aktifkan alat plugin opsional
`llm-task` dan panggil dari Lobster. Ini menjaga alur kerja tetap
deterministik sambil tetap memungkinkan Anda melakukan klasifikasi/ringkasan/draf dengan model.

Aktifkan alat:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

Gunakan dalam pipeline:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "thinking": "low",
  "input": { "subject": "Hello", "body": "Can you help?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

Lihat [LLM Task](/id/tools/llm-task) untuk detail dan opsi konfigurasi.

## File alur kerja (.lobster)

Lobster dapat menjalankan file alur kerja YAML/JSON dengan field `name`, `args`, `steps`, `env`, `condition`, dan `approval`. Dalam pemanggilan alat OpenClaw, tetapkan `pipeline` ke path file.

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

Catatan:

- `stdin: $step.stdout` dan `stdin: $step.json` meneruskan output langkah sebelumnya.
- `condition` (atau `when`) dapat menggerbang langkah berdasarkan `$step.approved`.

## Instal Lobster

Alur kerja Lobster bawaan berjalan in-process; tidak diperlukan binary `lobster` terpisah. Embedded runner dikirim bersama plugin Lobster.

Jika Anda memerlukan CLI Lobster mandiri untuk pengembangan atau pipeline eksternal, instal dari [repo Lobster](https://github.com/openclaw/lobster) dan pastikan `lobster` ada di `PATH`.

## Aktifkan alat

Lobster adalah alat plugin **opsional** (tidak diaktifkan secara default).

Disarankan (aditif, aman):

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

Atau per-agent:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

Hindari menggunakan `tools.allow: ["lobster"]` kecuali Anda memang bermaksud menjalankan mode allowlist yang ketat.

Catatan: allowlist bersifat opt-in untuk plugin opsional. Jika allowlist Anda hanya menyebut
alat plugin (seperti `lobster`), OpenClaw tetap mengaktifkan alat core. Untuk membatasi alat core,
sertakan juga alat core atau grup yang Anda inginkan dalam allowlist.

## Contoh: triase email

Tanpa Lobster:

```
User: "Check my email and draft replies"
→ openclaw calls gmail.list
→ LLM summarizes
→ User: "draft replies to #2 and #5"
→ LLM drafts
→ User: "send #2"
→ openclaw calls gmail.send
(repeat daily, no memory of what was triaged)
```

Dengan Lobster:

```json
{
  "action": "run",
  "pipeline": "email.triage --limit 20",
  "timeoutMs": 30000
}
```

Mengembalikan envelope JSON (dipotong):

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 need replies, 2 need action" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "Send 2 draft replies?",
    "items": [],
    "resumeToken": "..."
  }
}
```

Pengguna menyetujui → lanjutkan:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

Satu alur kerja. Deterministik. Aman.

## Parameter alat

### `run`

Jalankan pipeline dalam mode alat.

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

Jalankan file alur kerja dengan argumen:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

### `resume`

Lanjutkan alur kerja yang dihentikan setelah persetujuan.

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

### Input opsional

- `cwd`: Direktori kerja relatif untuk pipeline (harus tetap berada di dalam direktori kerja gateway).
- `timeoutMs`: Batalkan alur kerja jika melebihi durasi ini (default: 20000).
- `maxStdoutBytes`: Batalkan alur kerja jika output melebihi ukuran ini (default: 512000).
- `argsJson`: String JSON yang diteruskan ke `lobster run --args-json` (khusus file alur kerja).

## Envelope output

Lobster mengembalikan envelope JSON dengan salah satu dari tiga status:

- `ok` → selesai dengan sukses
- `needs_approval` → dijeda; `requiresApproval.resumeToken` diperlukan untuk melanjutkan
- `cancelled` → ditolak atau dibatalkan secara eksplisit

Alat menampilkan envelope dalam `content` (JSON yang dirapikan) dan `details` (objek mentah).

## Persetujuan

Jika `requiresApproval` ada, periksa prompt dan tentukan:

- `approve: true` → lanjutkan dan teruskan efek samping
- `approve: false` → batalkan dan finalkan alur kerja

Gunakan `approve --preview-from-stdin --limit N` untuk melampirkan preview JSON ke permintaan persetujuan tanpa glue `jq`/heredoc kustom. Resume token kini ringkas: Lobster menyimpan state resume alur kerja di bawah state dir miliknya dan mengembalikan kunci token kecil.

## OpenProse

OpenProse cocok dipasangkan dengan Lobster: gunakan `/prose` untuk mengorkestrasi persiapan multi-agent, lalu jalankan pipeline Lobster untuk persetujuan yang deterministik. Jika program Prose memerlukan Lobster, izinkan alat `lobster` untuk sub-agent melalui `tools.subagents.tools`. Lihat [OpenProse](/id/prose).

## Keamanan

- **Hanya lokal in-process** — alur kerja dijalankan di dalam proses gateway; plugin itu sendiri tidak melakukan panggilan jaringan.
- **Tanpa secret** — Lobster tidak mengelola OAuth; Lobster memanggil alat OpenClaw yang melakukannya.
- **Sadar sandbox** — dinonaktifkan saat konteks alat berada dalam sandbox.
- **Diperkeras** — timeout dan batas output ditegakkan oleh embedded runner.

## Pemecahan masalah

- **`lobster timed out`** → tingkatkan `timeoutMs`, atau pecah pipeline yang panjang.
- **`lobster output exceeded maxStdoutBytes`** → naikkan `maxStdoutBytes` atau kurangi ukuran output.
- **`lobster returned invalid JSON`** → pastikan pipeline berjalan dalam mode alat dan hanya mencetak JSON.
- **`lobster failed`** → periksa log gateway untuk detail error embedded runner.

## Pelajari lebih lanjut

- [Plugins](/id/tools/plugin)
- [Authoring alat plugin](/id/plugins/building-plugins#registering-agent-tools)

## Studi kasus: alur kerja komunitas

Satu contoh publik: CLI “second brain” + pipeline Lobster yang mengelola tiga vault Markdown (pribadi, pasangan, bersama). CLI mengeluarkan JSON untuk statistik, daftar inbox, dan pemindaian item usang; Lobster merangkai perintah-perintah itu menjadi alur kerja seperti `weekly-review`, `inbox-triage`, `memory-consolidation`, dan `shared-task-sync`, masing-masing dengan gerbang persetujuan. AI menangani penilaian (kategorisasi) saat tersedia dan fallback ke aturan deterministik saat tidak.

- Thread: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- Repo: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## Terkait

- [Automation & Tasks](/id/automation) — menjadwalkan alur kerja Lobster
- [Ringkasan Automation](/id/automation) — semua mekanisme otomasi
- [Ringkasan Tools](/id/tools) — semua alat agen yang tersedia
